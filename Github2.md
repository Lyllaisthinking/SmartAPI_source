# 当 model= 成为唯一变量：如何把多供应商接入压成一行配置

凌晨两点，你在本地把 gpt-4.1 换成 claude-sonnet-4，再换成 gemini-2.0-flash，只为确认同一段 Prompt 在三家上的差异。

如果每次切换都要改 base URL、换 Key、换 SDK 初始化方式、换错误码处理分支——这不是模型评测，这是在给基础设施打工。

而我们团队开发 SmartAPI（[smartapi.cc](https://www.smartapi.cc/)）想解决的就是这件事：把多供应商 AI 调用，收敛成开发者已经熟悉的那套 OpenAI 式接口，让 model 字段成为主要变量，其余配置尽量保持不变。

透明代理网关 推荐理由：可以把 SmartAPI 理解成应用与上游模型之间的透明代理。它不做模型训练，也不替代业务逻辑；职责是连接、路由、计费、稳定性，心智上接近 OpenRouter、LiteLLM 网关或自建中转，但更在意低迁移成本。

OpenAI 兼容接入 推荐理由：已有 OpenAI SDK 的项目，通常只改 `base_url` 和 `api_key` 两处即可。请求与响应体遵循上游规范，熟悉 OpenAI 或 Anthropic API 的开发者上手成本很低。

多协议分层 推荐理由：不止 Chat。平台按模型族与协议类型分层接入，覆盖 Chat Completions、Responses、Anthropic Messages，以及图像与视频生成，避免为每家供应商单独维护一套封装层。

Credits 统一计费 推荐理由：用统一账户管理多家模型的调用成本，Chat 按 token 计费，图像 / 视频按生成计费，不设最低消费，试错和 A/B 对比更可控。

零数据留存 推荐理由：SmartAPI 作为代理转发请求，不存储 prompts 与 completions，对客服 Bot、企业内部知识库、用户生成内容等场景更友好。

## 架构位置

要把这件事说清楚，得先明确 SmartAPI 在整条链路里站在什么位置。可以把它理解成应用与上游模型之间的透明代理网关：

```text
你的应用 / Agent / RAG 流水线
        ↓ （OpenAI / Anthropic 兼容请求）
    SmartAPI 网关
        · 鉴权与额度
        · 路由与健康检查
        · 失败重试 / failover
        ↓
OpenAI · Anthropic · Gemini · DeepSeek · 40+ providers
```

它不做模型训练，也不替代业务逻辑；职责是连接、路由、计费、稳定性。

心智上接近 OpenRouter、LiteLLM 网关或自建中转，但我们更在意低迁移成本——已有 OpenAI SDK 的项目，通常只改 `base_url` 和 `api_key` 两处即可。

详情请点击：[https://www.smartapi.cc/docs](https://www.smartapi.cc/docs)

## 能力：不止 Chat，协议也尽量对齐上游

在此之上，我们按模型族与协议类型做了分层接入：

| 能力类型 | 协议 / 路径 | 详细介绍 |
| --- | --- | --- |
| OpenAI 及 OpenAI 兼容模型 | OpenAI 协议，`POST /chat/completions` | 适用于 GPT 系列及遵循 OpenAI Chat Completions 规范的模型。 |
| OpenAI Responses 模型 | `openai-response` 协议，`POST /responses` | 面向采用 OpenAI Responses 规范的调用场景。 |
| Anthropic Claude 模型 | Anthropic 协议，`POST /messages` | Claude 线路可继续沿用 Anthropic SDK 或对应 HTTP 调用方式，只需将 base URL 与 API Key 指向 SmartAPI。 |
| 图像生成模型 | `POST /generate/image/generations` | 由 SmartAPI 统一封装图像生成能力。 |
| 视频生成模型 | `POST /generate/video/tasks` | 采用异步任务模式处理视频生成请求。 |

SmartAPI 按 transparent proxy 设计：请求与响应体遵循上游规范，我们不在中间改写语义。

熟悉 OpenAI 或 Anthropic API 的开发者，上手成本很低，不必再为每家供应商单独维护一套封装层。

目前平台支持 40+ providers、200+ models，OpenAI、Anthropic、Gemini 等主流线路均可通过同一入口访问。具体模型列表与定价，可在模型页面中查阅：[https://www.smartapi.cc/models](https://www.smartapi.cc/models)

## 接入：从 0 到第一条响应

SmartAPI 的接入路径刻意压短：创建 Key → 选择模型 → 发起请求。下面按实际开发流程说明。

### 第一步：在控制台创建 API Key

登录 [smartapi.cc](https://www.smartapi.cc/) 控制台，进入 API Keys 页面创建密钥。Key 以 `sk-` 为前缀，创建后完整内容仅展示一次，关闭页面后无法再次查看，请立即复制并妥善保存。

建议按密码级凭证管理：

- 通过环境变量或密钥管理工具注入，不要硬编码进源码
- 不要提交到 Git 仓库；在 `.env.example` 中只留占位符
- 不要在浏览器端、移动端或公开 Demo 中暴露 Key
- 团队协作者按人分配 Key，便于额度追踪与轮换

### 第二步：选择目标模型

在控制台的模型列表中选定本次调用的 model 标识，例如 `gpt-4.1`、`claude-sonnet-4` 等。不同模型对应不同上游供应商与计费规则，Chat 按 token 计费，图像 / 视频按生成计费，具体以模型页说明为准。

选定模型后，确认对应的 endpoint 类型：

| 模型类型 | Endpoint |
| --- | --- |
| OpenAI 兼容模型 | `POST /chat/completions` |
| Claude 模型 | `POST /messages`（Anthropic 兼容） |
| OpenAI Responses 模型 | `POST /responses` |

多数场景从 Chat Completions 开始即可。

### 第三步：发起第一条请求

建议先将 Key 写入环境变量，避免在命令历史中明文留存：

```bash
export SMARTAPI_KEY="sk-your-key-here"
```

使用 cURL 向 OpenAI 兼容 endpoint 发送请求：

```bash
curl https://api.smartapi.cc/api/openapi/v1/chat/completions \
  -H "Authorization: Bearer $SMARTAPI_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-4.1",
    "messages": [
      {"role": "user", "content": "Hello SmartAPI"}
    ]
  }'
```

请求进入 SmartAPI 网关后，会依次完成：鉴权 → 余额与限流检查 → 路由至健康上游通道（含自动 failover）→ 将响应返回客户端。响应体结构与 OpenAI Chat Completions 保持一致，可直接用于验证链路是否打通。

收到正常响应后，说明 Key、模型选择与 endpoint 均已配置正确。后续切换模型，通常只需修改 `model` 字段，无需改动其余调用逻辑。

## 可靠性：把上游抖动从业务里剥离

聚合平台的价值在于其稳定性：

- 请求路由至健康上游通道
- 短暂上游错误时自动重试
- 智能路由与连接复用，官网宣称 P99 延迟 < 200ms
- 99.9% Uptime SLA

模型可以换，业务链路尽量别跟着单点一起失效——这是网关层该做的事，而不是让每个服务自己写 retry 与 fallback。

## 计费：Credits 制，试错更可控

SmartAPI 采用 Credits 按量结算，用统一账户管理多家模型的调用成本。

| 项目 | 说明 |
| --- | --- |
| 充值换算 | $1 = 100 credits |
| Chat 类模型 | 按 token 消耗计费 |
| 图像 / 视频生成 | 按单次生成计费，具体价格随 model 不同而变化 |
| 最低消费 | 不设最低消费，用多少算多少 |

所有模型的费用汇总在同一账户下，便于在控制台查看各线路、各 model 的实际消耗，减少多平台对账成本。

目前平台提供八折优惠，更适合原型验证、灰度发布和 A/B 对比。建议先用真实流量跑一轮，再评估是否将其作为长期默认接入入口。

## 隐私设计：代理而不留存

我们团队承诺零数据留存：

- SmartAPI 作为代理转发请求，不存储 prompts 与 completions。
- 对客服 Bot、企业内部知识库、用户生成内容等场景友好。

但请注意：涉及行业合规时，仍建议结合隐私政策与服务条款做完整评估——网关降低接入摩擦，合规责任仍需按业务场景自行确认。

## 网关 vs 直连：优势何在？

| 维度 | 直连各供应商 | SmartAPI 网关 |
| --- | --- | --- |
| SDK / Client 数量 | 随供应商增加 | 尽量保持一套 |
| Key 管理 | 分散 | 统一 API Key |
| 模型切换 | 常涉及多处改动 | 主要改 model |
| 上游故障 | 业务侧自行兜底 | 内置路由与重试 |
| 账单 | 多账户对账 | 单账户 credits 结算 |
| 适用阶段 | 强合规、强绑定单家 | 多模型实验、快速迭代 |

没有“永远选网关”或“永远直连”——阶段不同，最优解不同。

多模型评测、Agent 原型、成本对比、快速换线路时，网关层往往更划算；必须 100% 直连官方、且深度依赖某家独有能力的场景，仍建议保留直连通道。

## 常见误区

误区 1：网关 = 多一层延迟，一定更慢？
多一层不等于必然慢。连接复用、路由优化与上游选路，可能抵消甚至优于业务侧裸调时的重复握手。以压测为准。

误区 2：聚合平台会锁死模型选择？
SmartAPI 覆盖 200+ models，切换通常只改 model 字段；不是换供应商就要换 SDK。

误区 3：自建 LiteLLM 更自由？
自建适合有专职运维、强定制需求的团队。SmartAPI 适合想把网关层外包、专注产品的开发者——不是替代关系，是阶段选择。

误区 4：零留存 = 零合规责任？
零留存降低数据暴露面，不等于自动满足 GDPR、HIPAA 等行业要求。请按业务场景做完整合规评估。

## 使用建议

第一，先把切换成本压下来。如果每次换模型都要改 base URL、Key、SDK 和错误处理，评测与迭代会被基础设施拖慢。更稳的做法是先统一入口，让 `model` 成为主要变量。

第二，按协议选 endpoint，而不是按品牌硬绑。Chat Completions、Responses、Anthropic Messages、图像与视频生成对应不同路径；多数场景从 Chat Completions 起步，确认链路打通后再扩展。

第三，密钥按密码级凭证管理。用环境变量注入，不写进源码，不进 Git，不在前端暴露；团队按人分配 Key，便于额度追踪与轮换。

第四，用 Credits 做可控试错。原型验证、灰度发布和 A/B 对比时，统一账户比多平台对账更省事；先用真实流量跑一轮，再决定是否作为长期默认入口。

第五，网关降低接入摩擦，但不替代合规判断。零留存降低暴露面，涉及行业合规时仍需结合隐私政策与业务场景自行确认。

最后，判断一层接入设施是否值得长期使用，不只看模型多不多，还要看它是否让你更快完成下一件事：下一次切换、下一轮评测、下一份账单核对。好的入口层应该更薄、更标准，让上层创新更快。

## 写在最后

AI 应用迭代的速度，往往快于接入层重构的速度。

当模型榜单每周都在变，入口层越薄、越标准，上层创新就越快。

SmartAPI 想提供的，就是这样一层基础设施：一把 Key · 多家模型 · 一份账单

- 站点：[https://www.smartapi.cc/](https://www.smartapi.cc/)
- 文档：[https://www.smartapi.cc/docs](https://www.smartapi.cc/docs)

我们正在搭建面向开发者的技术交流场域，围绕多模型路由、网关设计与 Agent 基础设施展开讨论。

也欢迎直接反馈产品设计与使用体验——你的真实场景，会直接影响 SmartAPI 下一版该优先补什么。

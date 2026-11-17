# 多提供商模型 API 接入调研

研究日期：2026-11-17（Asia/Shanghai）。对应 [研究票 #5](https://github.com/AlexdanerZe/PromptDeal/issues/5)，上位 [MVP 决策地图 #1](https://github.com/AlexdanerZe/PromptDeal/issues/1)。

本次只阅读官方文档、官方源码及公开只读目录；没有安装依赖、读取密钥、调用推理模型或验证付费账号。下文“事实”是资料或公开目录核验，“建议／推断”是面向本项目的判断，不代表已批准技术栈或已测通。

## 结论与最小候选组合（建议）

**可以复用成熟的统一调用层。若后续选择 TypeScript/Node，本机 MVP 的首选候选是 AI SDK Core + OpenRouter 官方适配器 + OpenRouter 托管 API。** AI SDK 负责应用内调用、结果解析和取消信号，OpenRouter 负责多厂商接入及账单；无需再启动独立模型网关。这两层已有官方集成，功能边界见下表。[AI SDK 介绍][ai-intro]、[OpenRouter 集成][or-sdk]

理由／推断：在用户尚未选定运行时 API 账号的前提下，一个聚合账号比同时处理三家账号更便于开始试用；但要接受新增数据处理方、平台费用和路由配置。应先用固定模型与提供商调试，再决定是否开启降级。10–100 条调试样本尚不足以单独证明需要网关，也不能据此承诺吞吐量。

**替代方案：AI SDK Core + OpenAI／Anthropic／Google 原生 provider，直接使用用户分别取得的 API 凭据。** 适用于已有厂商账号、希望减少聚合中转，或原生能力优先的情况；SDK 不要求部署到 Vercel。官方同时支持专用 provider 与托管 AI Gateway，二者可分别选用。[provider 选择][ai-choice]、[OpenAI][ai-openai]、[Anthropic][ai-anthropic]、[Google][ai-google]

若最终后端选 Python，LiteLLM SDK 是同一问题的自然候选；若以后需要独立进程集中管理多客户端的密钥、预算及路由，再考虑 LiteLLM Proxy。首版不建议叠加 AI SDK + LiteLLM Proxy + OpenRouter 三层，维护和故障定位成本会增加；这是项目范围推断，并非三者不兼容。

## 三者分别解决什么（官方事实）

| 维度 | OpenRouter | Vercel AI SDK | LiteLLM |
| --- | --- | --- | --- |
| 所处层次 | 托管 API 聚合服务；统一入口、认证及多提供商路由。[快速开始][or-start] | 应用内 TypeScript 库；Core 提供生成文本、结构化对象等统一接口。[介绍][ai-intro] | Python 调用库；另有可自部署的 OpenAI 格式 Proxy/Gateway。[介绍][lite-intro] |
| OpenAI／Anthropic／Google 覆盖 | 公开目录包含三家模型；模型作者与实际托管提供商需分别识别，见下节。[目录 API][or-model-doc] | 三家有专用 provider，保留各自参数和返回元数据；有适配器不等于取得模型权限。[OpenAI][ai-openai]、[Anthropic][ai-anthropic]、[Google][ai-google] | 三家有独立适配；Google AI Studio 与 Vertex 是不同接入路径。[OpenAI][lite-openai]、[Anthropic][lite-anthropic]、[Gemini][lite-google] |
| 自定义地址 | 客户端可指向 OpenRouter 的 base URL；任意自建模型地址不是普通账号的通用转发配置。Private Endpoints 为 Enterprise 附加功能，且有支持提供商、目录模型和协议条件。[快速开始][or-start]、[私有端点][or-private] | 原生 provider 可配置 `baseURL`；另有 `@ai-sdk/openai-compatible`。只支持 Chat Completions 的地址不能直接当作 Responses 端点。[OpenAI 设置][ai-openai]、[兼容 provider][ai-compatible] | 支持 `api_base`／`base_url`；可指向自建或兼容端点，仍需对应协议适配。[输入参数][lite-input] |
| 结构化输出 | 兼容端点接受 JSON Schema；Chat Completions 用 `response_format`，Responses 用 `text.format`；支持按端点核验。[结构化输出][or-json] | 当前 v7 用 `Output.object/array` 等配合 schema；最终结果有类型校验，解析／校验失败可保留原文、usage 和错误；流式中间对象不能当作完整已验证结果。[结构化数据][ai-json] | 支持 `response_format`、Pydantic／JSON Schema；可显式开启 `enable_json_schema_validation` 做客户端校验，不能假设所有模型原生支持 schema。[JSON 模式][lite-json] |
| 固定调用与降级 | 默认可动态选提供商；有 `order`、`only`、`allow_fallbacks`，也可设多模型 fallback；不是默认固定的实验环境。[提供商路由][or-route] | 显式 provider 实例确定调用路径；当前裸字符串模型默认走 **Vercel AI Gateway**，不等同于直连原厂或 OpenRouter。[provider 选择][ai-choice] | Router 可对同一别名配置多个部署并负载均衡；默认 simple-shuffle 无 rpm/tpm 时随机选部署，fallback 可配置。[Router][lite-route] |
| 重试／限流 | 有提供商故障切换；仍可能遇到上游 429、免费额度或余额／并发支出限制，应按错误来源及 `Retry-After` 处理。[限制][or-limits] | `maxRetries` 默认 2，可设 0；源码对可重试错误指数退避并读取合理的重试响应头。它不代替整个批任务的并发队列。[调用参数][ai-generate]、[重试源码][ai-retry] | Router 支持 rpm/tpm、重试、冷却和 fallback；RateLimitError 指数退避，普通错误可立即重试。[Router][lite-route] |
| 取消 | 断开**流式**连接仅在支持的提供商处停止生成和计费；当前文档列 OpenAI／Anthropic 支持，Google／Google AI Studio 不支持；非流式取消不保证停止上游。[流式取消][or-stream] | `abortSignal` 可取消调用；它是客户端能力，上游是否停止计算／计费取决于所接服务。[调用参数][ai-generate] | Proxy 的非流式 `cancel_on_disconnect` 默认 false，可显式启用并记录 499；这是取消在途调用的机制，不是跨厂商免计费承诺。[配置][lite-config]、[取消源码][lite-cancel] |
| Token／费用／耗时 | usage 有输入、输出、缓存、推理 token 和 `cost`；generation 查询另有实际 provider、latency、generation_time 等，需有账号凭据。[usage][or-usage]、[generation API][or-generation] | Core 提供 usage、raw usage、warnings、模型响应和 providerMetadata；OpenRouter 适配器将其费用放在 `providerMetadata.openrouter.usage`。不能将 Core 的 token 统计直接当成所有厂商统一账单。[返回结构][ai-generate]、[适配器源码说明][or-sdk-source] | SDK 有 `token_counter`、`completion_cost`，Proxy 可返回计算后的 `x-litellm-response-cost` 并记录 spend；这些是统计／计价功能，仍应与上游账单核对。[用量工具][lite-cost]、[Proxy spend][lite-spend] |
| 本机部署与维护 | 云端服务，应用只维护客户端；需要 OpenRouter 账号、API key 和可用额度。[快速开始][or-start]、[价格][or-price] | 随应用运行，可用专用 provider；不是必须购买的 Vercel 托管服务。[provider 选择][ai-choice] | SDK 随 Python 进程；Proxy 另起常驻服务，CLI 可先无数据库，虚拟 key／spend／Admin UI 需数据库，完整部署指南连接 Postgres。[CLI][lite-cli]、[Gateway][lite-proxy] |
| 许可／服务费用 | 托管服务按服务条款和模型价格使用；当前 Standard 购买 credits 收 5.5% 平台费，模型推理按提供商标价扣款；免费模型的配额不能当作 MVP 容量承诺。[价格][or-price] | SDK 为 Apache-2.0；模型 API／可选 Gateway 的费用另计。[许可证][ai-license]、[provider 选择][ai-choice] | `litellm/enterprise/` 外为 MIT；enterprise 目录另有生产使用许可要求。自部署不消除模型 API、运行环境及维护成本。[主许可证][lite-license]、[企业许可证][lite-enterprise] |

## 公开模型目录核验与版本边界

2026-11-17 实际无认证读取了 [OpenRouter `GET /api/v1/models`][or-models]，并读取以下端点列表；下面是目录样本，不是选定模型，更不是用户账号连通性证明：

| 模型作者 | 在目录中实际看到的 API ID | 在端点列表看到的例子 |
| --- | --- | --- |
| OpenAI | `openai/gpt-4.1-mini` | `openai`、`azure`、`azure/swedencentral`，列有 `structured_outputs`／`response_format`。[公开端点][or-ep-openai] |
| Anthropic | `anthropic/claude-sonnet-4.5` | `anthropic`、`amazon-bedrock`、`azure/global` 等；同一作者模型可有不同托管路径。[公开端点][or-ep-anthropic] |
| Google | `google/gemini-2.5-flash` | `google-ai-studio` 及 tier 变体，列有结构化输出和 reasoning 参数。[公开端点][or-ep-google] |

同次公开目录也列出了 `openai/gpt-6.1-sol`、`google/gemini-4-carbon`、`google/gemini-4-hydrogen`；此结论来自目录响应，**不来自 Orca/Codex/Antigravity CLI 可选模型**。公开列名仍不能证明用户所在地区、账号、余额、隐私过滤条件和选定端点可调用；本次未做认证查询或推理。[公开目录][or-models]

版本方面，查询 npm 官方发布元数据时，`ai` latest 为 **7.0.173**，`@openrouter/ai-sdk-provider` latest 为 **3.1.0**，均要求 Node >=22；适配器主线 README 也标明 v7／ESM，并为旧版 SDK 提供不同适配器版本。后续应锁定 SDK 与适配器组合，不能混抄旧文档；本次只核验版本声明，没有安装联测。[ai 发布元数据][ai-npm]、[OpenRouter 适配器发布元数据][or-sdk-npm]、[固定源码 README][or-sdk-source]

## 对 Prompt 调试正确性的影响（事实与建议）

1. **固定模型之外，还要固定服务路径。** OpenRouter 基础 provider slug 会匹配其多个区域／变体；精确后缀才进一步收窄。只设优先 `order` 不等于禁用 fallback，甚至 `allow_fallbacks=false` 配合 `order` 仍有私有端点例外。[路由规则][or-route] **建议：** 单模型、精确端点 `only`／`order`、关闭 fallback、`require_parameters=true`，并核对实际返回 provider；不用 `auto`、`latest` 或多模型候选做版本对比。LiteLLM 则每个实验别名只映射一个部署，不配置跨模型 fallback。可用性下降应显式记失败，不悄悄换模型。
2. **统一 API 不统一能力。** OpenRouter 的 `require_parameters` 默认 false，不支持的参数可能被忽略；LiteLLM 默认拒绝不支持的 OpenAI 参数，但 `drop_params=true` 可将其丢弃；AI SDK 会给不支持设置等 warnings。[OpenRouter][or-route]、[LiteLLM][lite-input]、[AI SDK][ai-generate] 原厂也有硬差异：OpenAI 当前 GPT-6 指南要求 reasoning 非 `none` 时移除 temperature 等参数；Claude 手动 `budget_tokens` 仅适用于支持的旧模型，4.7 及以后应使用 adaptive thinking；Google 适配器提供特定模型才支持的 `thinkingConfig`。[OpenAI][openai-model]、[Anthropic][anthropic-thinking]、[Google 适配][ai-google] **建议：** 为每个模型保存能力与有效参数，temperature、推理强度和输出上限按能力显示；保留原始厂商选项及 warnings，不承诺同名 `high` 或同值 temperature 代表相同计算量。
3. **结构正确与业务正确分开验。** schema 能约束字段与枚举；Google 官方也要求对 schema 合法但语义错误的结果进行应用校验。[Google 结构化输出][google-json] **建议：** 首个场景先约束 Context Hints 列表及每条的“方向类型（品牌宣传／竞品防守／产品功能）、素材方向、生成原因”，再评估语言、重复、事实依据和方向质量。条数、长度、品牌／竞品判定与去重规则等待用户最佳实践 Prompt 和样例，不由本次研究补造。
4. **重试和取消属于实验记录。** SDK、网关和聚合器同时重试会增加实际尝试次数；已提交但结果未知的请求再提交，费用也不能按一次成功结果推算。这是对上述重试／取消机制的推断。**建议：** 只设一个主要重试责任层，限制尝试次数与总时限；批任务停止继续派发，并把“用户已取消”“上游已确认停止”“用量待核对”分别记录。OpenRouter 流中错误仍可能出现在 HTTP 200 的 SSE 中，不能只看 HTTP 状态。[限制／流中错误][or-limits]、[流式行为][or-stream]
5. **统计保留来源和缺失状态。** OpenRouter `usage.cost` 是服务报告的扣款，LiteLLM cost 是按用量和价格计算；缓存、推理 token 和收费项需按相应口径读取。[OpenRouter usage][or-usage]、[LiteLLM 用量][lite-cost] **建议：** 每个节点／样本／attempt 记录请求及实际模型／provider、Prompt 版本、参数快照、schema 版本、状态、重试数、客户端总耗时和首 token 耗时（有流时）、input/output/cache/reasoning token、费用币种与来源。运行前估算与运行后服务报告费用分列；缺失标记未知，不能写成 0；本地估算不能重复累加已包含在 output 总数中的 reasoning token。固定价格快照时间，并将平台充值费与模型推理扣款分开。

## 数据处理与日志控制（官方事实）

- **OpenRouter 增加一跳数据处理。** 默认不开启私有输入输出日志及产品改进用数，但会保留请求元数据；当前政策还有 CSAM／法律留存以及 Safety Retention 例外。仅设置 `zdr=true` 不自动等于严格零留存：还需检查账号的 Safety Retention 允许项；严格 ZDR 会减少可用端点。[数据收集][or-data]、[ZDR][or-zdr] 上游条款仍按实际 provider 生效，而非仅按模型作者判断。[提供商政策][or-logging]
- **AI SDK 是本地软件，遥测发送到哪里取决于集成配置。** 当前 v7 注册遥测集成后默认发事件，输入输出默认可被记录；文档明确所有调用设 `telemetry.isEnabled=false` 才能同时关闭集成与 tracing-channel 事件。不要沿用旧版“实验性遥测默认关闭”的结论。[当前遥测文档][ai-telemetry]、[固定源码][ai-telemetry-source]
- **LiteLLM 自部署可控制自己的日志，不控制原厂留存。** `turn_off_message_logging` 可屏蔽发送给日志提供商的消息正文，同时保留 spend 等元数据；usage telemetry 另有 destination 及强制禁用开关。[日志][lite-logging]、[配置][lite-config] 还需逐个检查外部 callback：当前 OpenTelemetry v2 迁移说明特别指出，full span 模式下只屏蔽 generation 不一定屏蔽根请求及工具内容，不能用一个开关推断所有日志已脱敏。[OpenTelemetry v2 说明][lite-otel]
- **原厂政策要分别看。** OpenAI API 默认不用于训练，但有默认最多 30 天的滥用监测留存，以及按端点／`store` 等设置不同的状态留存；Anthropic 商业 API 默认不用于训练，主动反馈或允许共享有例外；Google Gemini Paid 与 Unpaid 的数据使用条款不同，免费服务通常可用于产品改进并有人工审核，付费服务不以输入输出改进产品但有有限安全留存，另有地区例外。[OpenAI 数据控制][openai-data]、[Anthropic 商业数据政策][anthropic-data]、[Google 条款][google-terms] 这些原厂直连条款不能自动替代 OpenRouter 的实际供应路径合同。

**面向本项目的建议：** 原始 Excel／JSON／CSV、清洗结果、Prompt 版本、评分与反馈保存在本机；模型请求只带本节点必需字段，先排除／脱敏敏感列。密钥由本机服务端使用，不下发网页；首版不主动接外部追踪平台。选择本机 Web 部署不代表云模型推理也在本机完成。

## 尚未解决的问题与后续验证边界

以下是选型前的未知项／后续验证建议，**没有在本次执行，也不是实施授权**：

- 用户希望用一个聚合账号，还是已有三家独立 API 账号；其地区、付款能力、账号权限、模型白名单、配额及所需数据政策目前均未核验。CLI 订阅、OAuth 会话和模型别名不作为运行时 API 凭据或 API ID。
- 后端语言未定；AI SDK 最新 v7 与 OpenRouter 适配器的协议、schema、usage/cost 映射，需要在最终锁定版本后做实际兼容性验证。目录状态不是连通性结果，原生 API 新能力也不保证聚合接口同日覆盖。
- 取得凭据并获准测试后，才验证：三家各一个明确 API ID 的最小请求、合法 schema／不兼容参数的明确失败、无静默 fallback、429／超时处理、流式及非流式取消后实际 usage、费用与服务端账单的对应。不得用网页文档替代此证据。
- 单数据源约 10,000 行与每轮 10–100 样本只是设计起点；文件解析／清洗、抽样、顺序工作流、任务状态、并发限额、版本对比、评分和反馈导出仍需应用自身负责。本研究没有测本机资源、网络、吞吐、延迟或模型质量，也未验证 provider Batch API；调试“批量”不预设等于厂商异步 Batch 产品。
- 未取得用户最佳实践 Prompt、真实输入／失败样例及评分准则，因此暂不选“最优模型”、不报质量分或每轮固定费用。选中字段翻译可作为同一调用层上的另一个节点；向编码 Harness 导出反馈的格式另行确定，不要求本应用复用或运行编码 CLI。

## 引用与复核说明

链接就近标注；公开目录／npm 元数据为研究日快照，文档链接会更新。涉及源码的关键引用固定到核验时提交：AI SDK `3bea041f28e65f371e640d2c70c302776a46387f`，OpenRouter 适配器 `7b07f138e005cee75d2ed8860cd59913bf8059f7`，LiteLLM `42f901ad6597d19b1661883bb53037bbc9d85bf4`；主线源码与 npm 发布包不是同一构建承诺，后续以锁定包的实际声明和行为为准。

[or-start]: https://openrouter.ai/docs/quickstart
[or-sdk]: https://openrouter.ai/docs/guides/community/vercel-ai-sdk
[or-sdk-source]: https://github.com/OpenRouterTeam/ai-sdk-provider/blob/7b07f138e005cee75d2ed8860cd59913bf8059f7/README.md
[or-sdk-npm]: https://registry.npmjs.org/@openrouter/ai-sdk-provider/latest
[or-model-doc]: https://openrouter.ai/docs/api/api-reference/models/list-all-models-and-their-properties
[or-models]: https://openrouter.ai/api/v1/models
[or-ep-openai]: https://openrouter.ai/api/v1/models/openai/gpt-4.1-mini/endpoints
[or-ep-anthropic]: https://openrouter.ai/api/v1/models/anthropic/claude-sonnet-4.5/endpoints
[or-ep-google]: https://openrouter.ai/api/v1/models/google/gemini-2.5-flash/endpoints
[or-private]: https://openrouter.ai/docs/guides/routing/private-endpoints
[or-json]: https://openrouter.ai/docs/guides/features/structured-outputs
[or-route]: https://openrouter.ai/docs/guides/routing/provider-selection
[or-limits]: https://openrouter.ai/docs/api_reference/limits
[or-stream]: https://openrouter.ai/docs/api_reference/streaming
[or-usage]: https://openrouter.ai/docs/cookbook/administration/usage-accounting
[or-generation]: https://openrouter.ai/docs/api/api-reference/generations/get-request-&-usage-metadata-for-a-generation
[or-price]: https://openrouter.ai/pricing
[or-data]: https://openrouter.ai/docs/guides/privacy/data-collection
[or-logging]: https://openrouter.ai/docs/guides/privacy/provider-logging
[or-zdr]: https://openrouter.ai/docs/guides/features/zdr
[ai-intro]: https://ai-sdk.dev/docs/introduction
[ai-choice]: https://ai-sdk.dev/docs/getting-started/choosing-a-provider
[ai-openai]: https://ai-sdk.dev/providers/ai-sdk-providers/openai
[ai-anthropic]: https://ai-sdk.dev/providers/ai-sdk-providers/anthropic
[ai-google]: https://ai-sdk.dev/providers/ai-sdk-providers/google
[ai-compatible]: https://ai-sdk.dev/providers/openai-compatible-providers
[ai-json]: https://ai-sdk.dev/docs/ai-sdk-core/generating-structured-data
[ai-generate]: https://ai-sdk.dev/docs/reference/ai-sdk-core/generate-text
[ai-retry]: https://github.com/vercel/ai/blob/3bea041f28e65f371e640d2c70c302776a46387f/packages/ai/src/util/retry-with-exponential-backoff.ts
[ai-license]: https://github.com/vercel/ai/blob/3bea041f28e65f371e640d2c70c302776a46387f/LICENSE
[ai-telemetry]: https://ai-sdk.dev/docs/ai-sdk-core/telemetry
[ai-telemetry-source]: https://github.com/vercel/ai/blob/3bea041f28e65f371e640d2c70c302776a46387f/content/docs/03-ai-sdk-core/60-telemetry.mdx
[ai-npm]: https://registry.npmjs.org/ai/latest
[lite-intro]: https://docs.litellm.ai/docs/
[lite-openai]: https://docs.litellm.ai/docs/providers/openai
[lite-anthropic]: https://docs.litellm.ai/docs/providers/anthropic
[lite-google]: https://docs.litellm.ai/docs/providers/gemini
[lite-input]: https://docs.litellm.ai/docs/completion/input
[lite-json]: https://docs.litellm.ai/docs/completion/json_mode
[lite-route]: https://docs.litellm.ai/docs/routing
[lite-config]: https://docs.litellm.ai/docs/proxy/config_settings
[lite-cancel]: https://github.com/BerriAI/litellm/blob/42f901ad6597d19b1661883bb53037bbc9d85bf4/litellm/proxy/common_request_processing.py#L2414
[lite-cost]: https://docs.litellm.ai/docs/completion/token_usage
[lite-spend]: https://docs.litellm.ai/docs/proxy/cost_tracking
[lite-cli]: https://docs.litellm.ai/docs/proxy/quick_start
[lite-proxy]: https://docs.litellm.ai/docs/simple_proxy
[lite-license]: https://github.com/BerriAI/litellm/blob/42f901ad6597d19b1661883bb53037bbc9d85bf4/LICENSE
[lite-enterprise]: https://github.com/BerriAI/litellm/blob/42f901ad6597d19b1661883bb53037bbc9d85bf4/litellm/enterprise/LICENSE.md
[lite-logging]: https://docs.litellm.ai/docs/proxy/logging
[lite-otel]: https://docs.litellm.ai/docs/observability/langfuse_to_otel_v2_migration
[openai-model]: https://developers.openai.com/api/docs/guides/latest-model
[openai-data]: https://developers.openai.com/api/docs/guides/your-data
[anthropic-thinking]: https://platform.claude.com/docs/en/build-with-claude/extended-thinking
[anthropic-data]: https://privacy.claude.com/en/articles/7996868-is-my-data-used-for-model-training
[google-json]: https://aistudio.google.com/docs/structured-output
[google-terms]: https://aistudio.google.com/docs/terms

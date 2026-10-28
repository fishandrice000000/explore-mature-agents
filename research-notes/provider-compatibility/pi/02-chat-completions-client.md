# provider-compatibility - Pi - 02 - 自研 Chat Completions 客户端的难度与复杂度

---

要实现一个 Chat Completions 客户端的难度如何? 复杂度如何?

---

## 1. 结论先说

**协议本身简单，工程复杂度中等偏低；真正的工作量在"把 Anthropic SDK 替你做掉的那三件事自己写一遍"。**

拆开看：

- 协议复杂度：低。Chat Completions 的请求体是一个 JSON，响应是"每帧一行 `data: {json}`"的 SSE，字段比 Anthropic Messages 少（没有内容块类型体系、没有签名、没有 block 生命周期事件）。
- 工程复杂度：中等。集中在 **SSE 分帧与 UTF-8 解码、工具调用的增量拼装、tool_result 与 tool_calls 的配对、错误体与流中断语义** 四处。
- 增量成本：对 JadeCode 这种"已有 `LlmClient` 抽象、已有内部消息模型、已有流式展示回调"的结构，属于**纯新增一个实现类，不改调用方**。
- 量级：核心 600–800 行 Java（含注释与错误分支），配套离线测试 300–500 行，约 1.5–3 个工作日当量；如果把重试、上下文溢出、辅助请求降级一并做完，风险调整后 3–5 天。

真正决定"要不要写"的不是难度，而是上一轮笔记第 7.1 的探测结果：**如果 OpenCode Go 对 `deepseek-v4.1-flash` 也提供 Anthropic `/v1/messages`，这个客户端可以完全不写。**

---

## 2. 为什么"看起来简单"是对的

三个客观事实：

1. **协议面窄**。请求只有 `model / messages / tools / max_tokens / stream / stream_options` 加一个供应商扩展字段（这里是 `thinking` + `reasoning_effort`）。响应只有 `choices[].delta` 的 `content`、`reasoning_content`、`tool_calls` 三类增量，加 `finish_reason` 与 `usage`。
2. **没有状态协议**。Anthropic Messages 的流是"块开始 → 块增量 → 块结束"，你要维护块生命周期；Chat Completions 只有"一个 assistant 消息的若干增量"，不需要维护块的开关状态。
3. **Pi 的实现规模可以反推**。Pi 的 Chat Completions 适配器总共 1736 行，但其中真正"协议本体"（主流程 + 消息转换 + 工具转换 + 用量与停止原因）约 830 行，其余近 900 行是跨供应商兼容、结构化思考回放、缓存标记、思考 token 预算这些东西——JadeCode 只需要其中一小部分。

---

## 3. 复杂度真正的来源：你在补 SDK 的空缺

JadeCode 现在的 Anthropic 客户端只有 349 行，其中流式解析部分之所以短，是因为它把三件重活交给了官方 SDK：

| 由 Anthropic SDK 承担 | 换到 Chat Completions 后由谁承担 |
|---|---|
| HTTP/2、连接复用、超时、代理、重试骨架 | 你自己（或 `java.net.http` / OkHttp） |
| SSE 分帧、事件类型分发、UTF-8 边界 | 你自己 |
| 把事件流拼装成完整消息（块边界、签名、工具参数拼接） | 你自己 |
| 错误体解析与 `statusCode` 映射 | 你自己（现在只把 SDK 异常翻成 `LlmException`） |
| 中止/中断时关闭连接并保证不写半份历史 | 你自己（现在靠 try-with-resources） |

所以"349 行"不是可比基数：**换协议后，客户端规模大致翻倍，而且行的"密度"更高**——每一行都在处理别人 SDK 里藏起来的边界。

JadeCode 侧已有的三块资产能显著降低工作量：

- **`LlmClient` 抽象**：调用方（Session/Agent/子代理/压缩器）完全不感知协议，新增实现零侵入；`CompleteRequest` / `ModelResponse` / `UsageRecord` 已经是协议中立的形状。
- **`ResponseStream` 回调**：`text / thinking / toolBlockStarted / toolInputDelta / toolBlockClosed / failed(stopReason)` 这套契约本来就是按"流式增量 + 预览不可执行"设计的，Chat Completions 的增量能一一映射进去（比 Anthropic 还更贴）。
- **已有的工具参数校验习惯**：现在会拒绝"非法/不闭合的工具 JSON"，这条纪律必须迁移到新客户端（服务端只给 `finish_reason=tool_calls`，不会替你校验参数完整）。

---

## 4. 工作量分解（按模块，含难度与风险）

| 模块 | 规模（行） | 难度 | 主要风险 |
|---|---|---|---|
| 传输层与 SSE 读取（分帧、`[DONE]`、注释行、空行、UTF-8） | 60–90 | 中低 | 按 chunk 读而不是按行读；多字节字符被切断 |
| 请求装配（URL 拼接、头、`max_tokens`、工具 schema、思考字段） | 60–90 | 低 | `/v1` 前缀与 Anthropic 协议语义相反，易配错 |
| 消息转换（历史 → OpenAI messages） | 120–180 | 中 | tool_result 拆条与顺序、空 assistant 消息、文本必须是字符串 |
| 工具定义转换 | 30–50 | 低 | 不要发 `strict`；schema 形状差异 |
| 流式事件循环（正文/思考/工具增量 + 展示回调） | 120–160 | 中 | 思考必须先于正文、工具增量按 index 归属 |
| 响应拼装与校验（块顺序、id、非法 JSON 拒绝） | 80–120 | 中 | 拼出的消息必须是完整可执行/可回放的历史 |
| 用量与停止原因映射 | 40–60 | 低（易漏） | `usage` 可能出现在顶层或 `choice` 内；缺 `finish_reason` |
| 错误映射与重试（状态码、`retry-after`、流中断） | 80–120 | 中 | 已显示的部分输出不能进历史；重试要发新 `requestId` |
| 非流式路径（辅助请求、压缩摘要） | 40–60 | 低 | 与流式共用转换逻辑，避免两份实现漂移 |
| 离线测试基建（本地 stub HTTP 服务 + 帧夹具） | 300–500 | 中 | 现有测试风格是不联网，需可注入传输 |

其中"离线测试基建"容易被低估：JadeCode 现有 `ToolWireContractTest` 用反射直接检查参数对象，不经过网络；新客户端如果直接 `new HttpClient()` 写死，就只能靠真实端点测试，既不可重复也不可提交。可复刻 `ModelMetadataResolver` 的 `Fetcher` 模式——注入一个 `(request) → response stream` 的传输接口，生产用 `java.net.http`，测试喂固定字节流（含故意切在 UTF-8 多字节中间、故意缺 `[DONE]`、故意 `finish_reason=length`）。

---

## 5. 会咬人的五个边界（按踩坑概率排序）

1. **SSE 分帧**。HTTP chunk 边界与 SSE 事件边界是两层，必须按行累积、空行分帧；`data:` 后可能有空格也可能没有，可能有多行 data 需要拼接；`[DONE]` 不是 JSON；服务端可能发注释保活行。常见错误是按 chunk 解码成字符串直接 `split("\n\n")`，遇到跨 chunk 的事件就丢帧。用 `java.net.http` 的 `BodyHandlers.ofLines()` 或 OkHttp 的 `readUtf8Line()` 可以把编码与分行这两件事直接拿下。
2. **工具调用的增量拼装**。首个 delta 才带 `id` 与 `name`，后续只带 `arguments` 片段；片段可能切在转义序列或数字中间；并行工具调用同时到达时必须按 `index` 归属而不是按到达顺序新建块。JadeCode 的现有纪律是"参数不闭合就拒绝执行"，这条要在新客户端保留，否则模型被截断时会拿着半个 JSON 去执行工具。
3. **tool_result 与 tool_calls 的配对**。Anthropic 侧允许一条 user 消息里放多个 `tool_result` 块；OpenAI 侧必须是多条 `role: "tool"` 消息，且**紧跟在**对应 assistant 的 `tool_calls` 之后，`tool_call_id` 必须逐字一致。这是历史转换里最容易出错、又最容易在"单工具单轮"测试里漏掉的地方（要用并行工具调用的夹具覆盖）。
4. **错误体与流中断**。非 2xx 时响应体是 JSON（`error.message` / `error.type`），要翻成带 `statusCode` 的 `LlmException`（JadeCode 的上下文溢出自救目前只看错误文本关键词，状态码另有用途，两条都不能丢）；2xx 之后流中途断掉属于"已经显示了半份输出但历史不可用"，必须走 `ResponseStream.failed(...)` 而不是静默丢弃。
5. **没有 `finish_reason` 的流**。部分兼容端点不发 `finish_reason` 就结束。Pi 对此有兼容开关（"不支持 finish_reason"时按有无工具调用推断 stop/toolUse），并在"声称支持但没发"时直接报错。JadeCode 需要自己定策略：宁可报错，也不要把"流提前结束"当成正常完成——否则会把截断的回复写进历史。

次要但要留意：`usage` 字段位置不统一（顶层 vs `choice.usage`）；`reasoning_content` 可能叫 `reasoning`；OpenCode 侧还要不要发缓存字段（`prompt_cache_key` / `prompt_cache_retention`）与 `store`，这些都属于"发了可能被拒、不发更安全"的类别。

---

## 6. 依赖选择：手写 vs 用 SDK

| 方案 | 优点 | 代价 |
|---|---|---|
| 手写（`java.net.http` 或已随 Anthropic SDK 引入的 OkHttp） | 完全控制字段与错误语义；无新依赖；`reasoning_content` 这类非标准字段随手支持 | 上面五个边界全部自己负责 |
| 引入 OpenAI 官方 Java SDK | 类型化请求体、内置流式与重试 | 强类型模型里没有 `reasoning_content` / `thinking` 这类扩展字段，需要绕到底层 JSON；错误语义被 SDK 再抽象一层；JadeCode 等于同时维护两套 SDK 风格 |
| 引入第三方轻量客户端 | 少写代码 | 多一个不受控依赖；流式增量与中止语义仍要自己接线 |

**建议手写。** 理由不是"行数更少"，而是 JadeCode 的核心诉求是"对非标准字段与失败语义有完全控制权"，而这恰好是 SDK 封装最厚的地方。此外 Anthropic Java SDK 已经把 OkHttp 带进来了，传输层零新增依赖。

---

## 7. 不要照抄 Pi 的部分

Pi 的 1736 行里，下面这些对 JadeCode 当前目标没有价值，照抄只会放大复杂度：

- **结构化思考回放**（`reasoning_details` 的解析/合并/签名序列化，约 190 行）：那是为 OpenRouter / Responses 的加密推理设计的，DeepSeek 系用不到。
- **多供应商分支**（Qwen 的 `enable_thinking`、GLM 的 `thinking.enabled`、`chat_template_kwargs`、Baseten、together、zai…）：JadeCode 只需要一条"DeepSeek 表达方式"。
- **思考 token 预算**（`thinkingBudgets`、预算与答案共享上限的钳制）：那是 Anthropic 风格预算协议的产物，effort 型协议不需要。
- **Anthropic 风格缓存标记**（`cache_control` 注入到系统提示/最后工具/最后消息）：OpenCode Go 的 DeepSeek 路线不靠这个。
- **兼容开关的自动推断引擎**（约 140 行的 `detectCompat`）：JadeCode 只需要一份静态供应商档案。

也就是说：Pi 的形态里，JadeCode 要用的大约是 35%。

---

## 8. 对 JadeCode 现有结构的最小侵入方案

1. **新增 `OpenAiCompletionsLlmClient implements LlmClient`**，与现有 Anthropic 实现并列；装配点（`JadeCodeFactory`）按配置的供应商档案选择实现。调用方一行不改。
2. **抽一个可注入的传输接口**（如 `StreamingTransport.request(...) → 行流/字节流`），生产实现走 `java.net.http`，测试实现喂夹具。这样"SSE 分帧 + 增量拼装"可以纯离线单测，符合仓库现有测试风格。
3. **把"供应商档案"做成数据**（协议、base URL 后缀语义、会话头、思考表达方式、档位表、是否发 `store`/缓存字段、输出上限），而不是继续在构造函数里按主机名分支——`AnthropicSdkLlmClient` 现在正是按主机名决定档位与 `thinking.type`，这条路每个新供应商都要改一次代码。
4. **共用一层"档位 → 请求字段"翻译**，让两个客户端都从同一份模型能力表取档位，避免"Anthropic 客户端认为有 off、Chat Completions 客户端认为没有"的不一致。

工作量对照：如果先做第 2、3 步（纯重构，行为不变，可用现有 DeepSeek 端点回归），第 1 步就只剩协议实现本身；如果不做，协议代码里会掺进大量 `if (host.equals(...))`，第二次接供应商时成本翻倍。

---

## 9. 决策建议

- **先花一小时探测**：Anthropic `/v1/messages` 可用 → 不写这个客户端（成本从 3 天降到半天：端点识别 + `reasoning_effort` + 会话头）。
- **若必须写**：按"先跑通单轮文本 + 单工具调用（含流式展示），再补并行工具/思考回放/重试/溢出"两阶段推进；第一阶段 300 行左右就能端到端跑通，第二阶段才是复杂度所在。
- **验收必须包含**：流被截断（`finish_reason=length`）、工具参数非法、并行工具调用、429 带 `retry-after`、真实上下文溢出五类失败样本。这五类过了，剩下的都是细节。

一句话概括：**协议难度低，工程复杂度中等，风险集中在"流"和"历史配对"两处；对一个已有协议中立的 `LlmClient` 抽象的项目，这是"一个下午能看到雏形、两三天才敢上生产"的量级。**

---

## 10. 证据分级

**已证实（源码直接可见）**

- Pi 的 Chat Completions 适配器 1736 行，其中主流程、消息转换、工具转换、用量与停止原因映射合计约 830 行，其余为跨供应商兼容与增强；
- Pi 在该路径上处理的边界：思考字段多候选名、按 index 归属的工具增量、无 `finish_reason` 的两种兼容策略、流结束但块不完整时报错、`usage` 顶层与 `choice` 内两处兜底、错误体重整；
- JadeCode 的 Anthropic 客户端为 349 行，流式解析依赖 SDK 的累积器与事件迭代器，自身只做工具 JSON 校验与展示回调分发；
- JadeCode 的 `LlmClient` / `CompleteRequest` / `ModelResponse` / `ResponseStream` 已是协议中立形状；`ModelMetadataResolver` 提供了"可注入传输 + 离线测试"的既有范式；
- JadeCode 依赖里已有 Jackson 与 Anthropic Java SDK（后者带入 OkHttp），无 OpenAI SDK；
- `LlmException` 携带状态码；`ResponseStream.failed(stopReason)` 已支持"失败但保留真实停止原因"；
- 现有测试风格为不联网（反射检查参数对象、脚本化替身客户端）。

**合理推断**

- 600–800 行核心实现与 1.5–3 天当量：基于 Pi 对应子系统规模按功能裁剪后的估算，非实测；
- 主要风险排序（SSE 分帧 > 工具增量 > 历史配对 > 错误语义 > 缺 `finish_reason`）：依据 Pi 代码中为这些边界单独写的分支与历史修补提交数量；
- 手写优于引入 SDK：基于"非标准字段 + 错误语义可控性"的权衡，未做同规模原型对比。

**无法确认**

- OpenCode Go 对 `deepseek-v4.1-flash` 是否也提供 Anthropic `/v1/messages`（决定是否根本不需要这个客户端）；
- 该端点是否发 `finish_reason`、`usage` 的位置、错误体形状、是否接受并行工具调用——都需要实测夹具确定，这直接决定边界处理代码的多少。

---

## 11. 源码依据

| 位置 | 内容 |
|---|---|
| `packages/ai/src/api/openai-completions.ts:304-737` | 主流程：SSE 遍历、思考/正文/工具增量、结束校验、错误收尾 |
| `packages/ai/src/api/openai-completions.ts:640-737` | 工具增量按 index 归属；缺 `finish_reason` 的两种策略；流结束但块不完整即报错 |
| `packages/ai/src/api/openai-completions.ts:560-620` | 思考字段多候选与优先级；正文增量处理 |
| `packages/ai/src/api/openai-completions.ts:1195-1483` | 历史转换：文本必须为字符串、思考回放、tool_call/tool_result 配对 |
| `packages/ai/src/api/openai-completions.ts:1484-1520` | 工具 schema 转换（`function` 形式、`strict` 由开关决定） |
| `packages/ai/src/api/openai-completions.ts:1521-1594` | `usage` 双位置兜底与停止原因映射 |
| `packages/ai/src/api/openai-completions.ts:1595-1736` | 兼容开关自动推断（JadeCode 不需要，用静态档案替代） |
| `packages/ai/src/utils/provider-retry.ts:1-60` | 可重试判定与 `retry-after` 处理 |
| `packages/ai/src/api/simple-options.ts` | 输出上限与上下文余量钳制 |
| `JadeCode src/main/java/com/jadecode/llm/AnthropicSdkLlmClient.java:110-200` | 现有流式实现：依赖 SDK 累积器，只做块校验与展示分发 |
| `JadeCode src/main/java/com/jadecode/llm/LlmClient.java`、`CompleteRequest.java`、`ModelResponse.java`、`ResponseStream.java` | 协议中立的调用契约与流式展示契约 |
| `JadeCode src/main/java/com/jadecode/llm/ModelMetadataResolver.java:40-130,250-260` | 可注入传输、离线测试范式、官方主机白名单 |
| `JadeCode src/main/java/com/jadecode/context/ContextCompactor.java:685-703` | 上下文溢出识别依赖错误文本关键词（新协议需补词） |
| `JadeCode src/test/java/com/jadecode/llm/ToolWireContractTest.java` | 现有"不联网、反射检查线上契约"的测试风格 |
| `JadeCode pom.xml:29-56` | 依赖现状：Jackson、Anthropic Java SDK（含 OkHttp），无 OpenAI SDK |

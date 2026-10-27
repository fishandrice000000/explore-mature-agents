# 研究笔记

本目录归档 Codex、Claude Code、Pi、SkillOpt、Poirot 等 Agent 相关项目的源码机制研究笔记。这里只放 Agent 架构与机制相关内容；会话规则、环境配置和工作流说明放在工作区根目录。

## 归档约定

```text
/home/fishandrice/Backend/explore-mature-agents/research-notes/<主题>/<agent>/<NN>-<slug>.md
```

- `<主题>`：英文 kebab-case，如 `memory-system`、`agent-loop`。
- `<agent>`：`codex`、`claude-code`、`pi`、`skillopt`、`poirot`；跨 Agent 对照可用 `comparison`。
- `<NN>`：在同一个 `<主题>/<agent>/` 目录内从 `01` 独立递增。
- 每轮一个文件：不追加、不覆盖；追问、计划确认与结论修正也另建新文件。
- 文件结构：一级标题 → 用户提问原文 → 完整答复。
- 回归源码核对时，结论标注“已证实 / 合理推断 / 无法确认”，并把源码位置集中放在文末。

## 索引

### tui

#### pi

| # | 文件 | 主题 |
|---|---|---|
| 01 | [01-inline-ui-mechanism-and-java-port.md](tui/pi/01-inline-ui-mechanism-and-java-port.md) | 内联终端界面的运行机制：输入与编辑、差分渲染、历史不可变边界、消息与工具卡片、选择器、主题，以及 Jade Code 的 Java 复刻分层与落地顺序 |
| 02 | [02-java-tui-architecture.md](tui/pi/02-java-tui-architecture.md) | 基于 Jade Code 现有源码的 Java TUI 架构：`SessionRuntime`/`ResponseStream`/`Hooks`/`Approval` 的真实接缝、`TaggedOutput` 身份桥接、`TuiPresentation` 与 `StreamingAdapter`、包结构、线程模型、分阶段落地与验证清单 |
| 03 | [03-runtime-and-render-pipeline.md](tui/pi/03-runtime-and-render-pipeline.md) | 运行模型、终端适配与两种差分渲染器：默认 fullscreen 与 regular 的进入/退出、16ms 调度、Kitty 协商、主屏行 diff、备用屏布局/鼠标/搜索/选择、退出 transcript 重放 |
| 04 | [04-layout-components-overlays.md](tui/pi/04-layout-components-overlays.md) | 布局、组件、Overlay 与文本几何：grapheme 宽度、ANSI 折行/截断/切列/合成、Box/Stack/Layout rect-clip、ScrollView/滚动条、Overlay 尺寸/焦点/鼠标命中 |
| 05 | [05-input-editor-keybindings.md](tui/pi/05-input-editor-keybindings.md) | 输入、键绑定、编辑器与补全：legacy/Kitty/modifyOtherKeys 解析、键绑定优先级、视觉行/粘性列/撤销/kill ring/大粘贴、slashed/@ 文件补全、Input/SelectList |
| 06 | [06-markdown-theme-images.md](tui/pi/06-markdown-theme-images.md) | Markdown、主题、图片与动效：marked+LaTeX/严格删除线、代码高亮/表格/链接、55 个主题 token 与 system/dark/light、OKLCH/256 色、Kitty/iTerm2 图片、Loader/spinner |
| 07 | [07-coding-agent-interactive-assembly.md](tui/pi/07-coding-agent-interactive-assembly.md) | Coding Agent 交互层组装：header/chat/pending/status/editor/footer 组件树、regular vs fullscreen dock、消息/thinking/工具卡/bash 卡、流式事件顺序、选择器替换 editor、退出重放 |
| 08 | [08-jadecode-porting-map.md](tui/pi/08-jadecode-porting-map.md) | JadeCode 逐项差距与移植路线：现有 `TuiApplication`/`TuiController`/`InlineRenderer`/`Editor`/`Theme` 对照 Pi、目标包结构、行表示取舍、阶段 0–7、验收矩阵与常见坑 |

### web-fetch

#### claude-code

| # | 文件 | 主题 |
|---|---|---|
| 01 | [01-webfetch-mechanism.md](web-fetch/claude-code/01-webfetch-mechanism.md) | `WebFetch` 的 URL 抓取、域名权限与风险预检、跨域重定向、HTML 转 Markdown、小模型提炼、缓存、失败路径与资源上限 |

#### codex

| # | 文件 | 主题 |
|---|---|---|
| 01 | [01-page-retrieval-mechanism.md](web-fetch/codex/01-page-retrieval-mechanism.md) | 没有独立 Fetch 工具时，Codex 如何通过托管 `web_search` 的 `open_page` 或独立 `web.run` 的 `open` 完成远端页面读取、访问模式控制、输出回传与结果旁路 |

#### pi

| # | 文件 | 主题 |
|---|---|---|
| 01 | [01-no-first-party-fetch.md](web-fetch/pi/01-no-first-party-fetch.md) | 不提供第一方网页抓取：读取工具只做本地路径解析与存在性探测（无 URL 分支）、工具枚举与模型接口均无抓取表达、提示词无抓取与引用约定；唯一取页路径是借 shell 取回原文并走通用截断，缺失 HTML 转换、正文提炼、域名策略与缓存 |

### web-search

#### claude-code

| # | 文件 | 主题 |
|---|---|---|
| 01 | [01-search-mechanism.md](web-search/claude-code/01-search-mechanism.md) | `WebSearch` 不是搜索客户端，而是“一次嵌套模型请求”：服务端检索工具的声明与次数上限、块序列折叠成摘要与链接、结果体积与落盘、权限询问、延迟加载、引用提醒与失败路径 |

#### codex

| # | 文件 | 主题 |
|---|---|---|
| 01 | [01-search-mechanism.md](web-search/codex/01-search-mechanism.md) | 托管 `web_search` 与独立 `web.run` 两条互斥检索路径：四档访问模式与默认 `cached`、每轮降级与沙箱关闭时的提升、命令集合与会话尾部裁剪、结果旁路与外部内容标记、引用与字数约束 |

#### pi

| # | 文件 | 主题 |
|---|---|---|
| 01 | [01-no-first-party-search.md](web-search/pi/01-no-first-party-search.md) | 不提供第一方联网检索：内置工具集的封闭枚举、模型接口层无法表达供应商原生工具、提示词既无联网声明也无日期注入，唯一联网途径是借 shell 调外部程序及其缺失的检索专用层 |

### context-compaction

#### codex

| # | 文件 | 主题 |
|---|---|---|
| 01 | [01-research-plan.md](context-compaction/codex/01-research-plan.md) | 研究计划：Codex 上下文压缩的触发、实现选择、远端压缩、历史替换与恢复 |
| 02 | [02-default-remote-compaction-path.md](context-compaction/codex/02-default-remote-compaction-path.md) | 默认远端压缩路径：90% 阈值、供应商能力选择 V2、请求形状、64K 保留策略、检查点与失败边界 |

#### claude-code

| # | 文件 | 主题 |
|---|---|---|
| 01 | [01-research-plan.md](context-compaction/claude-code/01-research-plan.md) | 研究计划：Claude Code 上下文压缩的触发条件、分层机制、压缩后重建、失败降级与持久化 |
| 02 | [02-context-compaction-mechanism.md](context-compaction/claude-code/02-context-compaction-mechanism.md) | 上下文压缩机制：工具结果预算、微压缩、会话记忆压缩、整段摘要、压缩后附件重建、响应式路径与阈值 |
| 03 | [03-default-behavior-and-feature-gates.md](context-compaction/claude-code/03-default-behavior-and-feature-gates.md) | 默认行为与功能开关：四类默认来源、默认开启/关闭清单、默认阈值与外部构建的实际主干路径 |

#### pi

| # | 文件 | 主题 |
|---|---|---|
| 01 | [01-research-plan.md](context-compaction/pi/01-research-plan.md) | 研究计划：Pi 上下文压缩的触发、切点、摘要、持久化、溢出恢复与分支摘要 |
| 02 | [02-compaction-and-branch-summary.md](context-compaction/pi/02-compaction-and-branch-summary.md) | 压缩与分支摘要：默认阈值、切点规则、结构化摘要、会话树投影、溢出恢复与 `pi-durable` 边界 |
| 03 | [03-auto-compaction-threshold.md](context-compaction/pi/03-auto-compaction-threshold.md) | 自动压缩触发阈值：窗口减 16384 的绝对值比较、token 口径、三处检查时机、配置覆盖与边界行为 |
| 04 | [04-context-token-accounting.md](context-compaction/pi/04-context-token-accounting.md) | 上下文 token 统计：最近的助手 usage 作锚点、尾部字符估算、系统提示重放、压缩/编辑后的失效回退与各检查点取数路径 |

### memory-system

#### claude-code

| # | 文件 | 主题 |
|---|---|---|
| 01 | [01-memories.md](memory-system/claude-code/01-memories.md) | Claude Code 自动记忆机制总览：存储布局、四条记忆类型、提取（主 agent 直写 + 后台 fork 抽取）、整理（autoDream 合并剪枝 / 团队记忆同步）、召回（索引注入 vs 相关性预取） |
| 02 | [02-memory-recall.md](memory-system/claude-code/02-memory-recall.md) | 召回详解：`startRelevantMemoryPrefetch` 三步流程（扫描 → Sonnet 选择器 → 读取注入）、零等待消费点、三层去重与三层预算 |
| 03 | [03-recall-switch.md](memory-system/claude-code/03-recall-switch.md) | `tengu_moth_copse` 开关的来源：GrowthBook 远端求值与四级缓存链、实验曝光上报、刷新间隔；“用户手动切换”与“harness 自适应”均不成立 |
| 04 | [04-prefetch-recall-limitation.md](memory-system/claude-code/04-prefetch-recall-limitation.md) | 预取的时效性评价：注入后记忆常驻上下文（至压缩为止）；无工具调用的回合不会消费预取；“鸡肋”论成立与不成立的部分，以及源码位置速查 |

#### codex

| # | 文件 | 主题 |
|---|---|---|
| 01 | [01-codex-memory-system.md](memory-system/codex/01-codex-memory-system.md) | Codex 记忆系统全链路：启动期后台流水线（Phase 1 逐 rollout 纯 sampling 提取 / Phase 2 全局 consolidation agent）、git-baseline 增量与遗忘、只读召回（2500 tokens 摘要注入 + quick memory pass）、citation→usage→留存闭环、ad-hoc note 唯一写入通道、v1/v2 差异、从 Claude Code 导入记忆的迁移规则 |
| 02 | [02-codex-memory-system-mechanism.md](memory-system/codex/02-codex-memory-system-mechanism.md) | 同主题的机制层重写版（场景驱动、正文不含源码标识符、源码依据集中在文末）：三条路径拆分、三层记忆结构、生成的两段流程、只读消费与检索纪律、引用反馈闭环、修正通道、边界与设计取舍 |
| 03 | [03-background-extraction.md](memory-system/codex/03-background-extraction.md) | 后台提取（第一阶段）细节：新回合派发与五道关卡、候选会话筛选（10 天窗口 / 6 小时空闲 / 排除自身 / 5000 条扫描）、数据库租约抢占与全局并发上限、重试退避与水位线语义、证据分层裁剪与脱敏、结构化输出契约与 no-op 删除、无心跳的超时边界 |

> `01` 为首次探索版（含较多源码标识符），`02` 为按机制讲解规范重写的版本。阅读建议以 `02` 为主、`01` 作为可追溯的细节底稿。

#### poirot

| # | 文件 | 主题 |
|---|---|---|
| 01 | [01-long-term-memory-pipeline.md](memory-system/poirot/01-long-term-memory-pipeline.md) | 被动旁路式长期记忆全链路：Markdown 单文件真相源 + 空格分词 BM25、注入隐藏用户消息、内容哈希去重编码、后台抽取与最旧 10 条合并；衰减/遗忘/关联的落地程度，以及实测的中文零召回、每次召回 1.2 秒整库重写、注入消息回流成检索词、空合并清空记忆等断点 |

### mcp-tools

#### claude-code

| # | 文件 | 主题 |
|---|---|---|
| 01 | [01-mcp-tool-lifecycle.md](mcp-tools/claude-code/01-mcp-tool-lifecycle.md) | MCP 工具全链路：多来源 Server 配置与连接、单页工具发现与变更刷新、统一工具适配与工具搜索、默认审批、并发编排、协议调用、会话恢复、结果归一化与大结果降级 |

#### codex

| # | 文件 | 主题 |
|---|---|---|
| 01 | [01-mcp-tool-lifecycle.md](mcp-tools/codex/01-mcp-tool-lifecycle.md) | MCP 工具全链路：运行时 Server 投影、分页工具目录、可见性过滤与命名去重、直接/延迟/Code Mode 暴露、目录快照绑定调用、注解驱动审批、认证与结果预算 |

#### pi

| # | 文件 | 主题 |
|---|---|---|
| 01 | [01-mcp-tool-lifecycle.md](mcp-tools/pi/01-mcp-tool-lifecycle.md) | MCP 工具全链路（三层结构：客户端包 / 内置扩展 / 工具池）：两处配置与项目覆盖、后台连接与精准等待、`mcp__server__tool` 命名与哈希去重、四档暴露（默认只能从脚本调用）、系统提示服务器清单预算、内容块转换与 20KB 中部截断、会话过期重试与隐藏式撤销 |

### multi-agent

#### claude-code

| # | 文件 | 主题 |
|---|---|---|
| 01 | [01-research-plan.md](multi-agent/claude-code/01-research-plan.md) | 研究计划：Claude Code 多 Agent 协作（子代理 / 团队队友 / 远端执行三层）、创建与继承、通信、分工、隔离、生命周期与治理；含待确认范围（是否包含远端执行体、是否与 Codex 对照） |
| 02 | [02-teammate-collaboration.md](multi-agent/claude-code/02-teammate-collaboration.md) | 常驻队友/团队机制的完整讲解：开启条件（能力开关 vs 模型判断）、任务分配的三条通道、文件邮箱与轮询通信、上下文隔离边界、并发处理（含文件级并发的缺口）与设计取舍 |

#### codex

| # | 文件 | 主题 |
|---|---|---|
| 01 | [01-research-plan.md](multi-agent/codex/01-research-plan.md) | 研究计划：Codex 常驻多 Agent（注册表 / 身份 / 角色 / 消息板 / 生命周期控制）的开启条件、任务分配、通信、上下文隔离与并发；含待确认范围（新旧两版如何取舍、是否包含角色与消息板、worktree 边界） |
| 02 | [02-persistent-teammate-collaboration.md](multi-agent/codex/02-persistent-teammate-collaboration.md) | Codex 常驻多 Agent 机制：版本与主动委派模式的开启条件、父代理显式下发（无任务池）、内存邮箱与触发型/非触发型消息、all/none 有损上下文继承、槽位准入与驻留驱逐，以及文件级并发的缺口 |

#### poirot

| # | 文件 | 主题 |
|---|---|---|
| 01 | [01-delegation-mechanism.md](multi-agent/poirot/01-delegation-mechanism.md) | 委派式多 Agent 的装配、执行与结果回流：装配期决定可委派对象、软路由（提示词 + 工具）、一次委派的六阶段时序、pi/codex/claude/自拷贝四种执行体的协议差异、产物门槛导致外部委派恒判失败、编排产物无消费者、并发与超时边界，以及自拷贝两条路径的执行断链 |
| 02 | [02-no-persistent-agents.md](multi-agent/poirot/02-no-persistent-agents.md) | 为什么没有常驻型 Agent：进程/线程/对象/状态四层盘点（三个常驻后台线程都是基础设施）、一次性委派契约如何结构上排除驻留与恢复、与 Claude Code 队友与 Codex 工作组的形态对照、四项缺失能力与由此产生的边界 |
| 03 | [03-delegation-only-no-team.md](multi-agent/poirot/03-delegation-only-no-team.md) | 只有委派层没有团队层：三层分级（单 Agent 能力 / 单层委派 / 团队）下 Poirot 的落点、外包式 specialist 与自拷贝 subagent 两种语义、team 所缺的五个要件、三处易误认为 team 的结构（注册表/待办/自演化），以及单层委派必然带来的四类边界 |

### permission-system

#### claude-code

| # | 文件 | 主题 |
|---|---|---|
| 01 | [01-permission-modes-and-approval-flow.md](permission-system/claude-code/01-permission-modes-and-approval-flow.md) | 权限档位与审批判定：外部五档与内部 `auto`、逐工具参数解释、拒绝/询问/模式/允许的优先级、规则来源与记忆语义、Bash 子命令与沙箱、计划模式和绕过模式的真实边界 |

#### codex

| # | 文件 | 主题 |
|---|---|---|
| 01 | [01-approval-and-sandbox-model.md](permission-system/codex/01-approval-and-sandbox-model.md) | Codex 的审批策略与沙箱权限双层模型：Read Only/Ask/Approve for me/Full Access、`on-request`/`never`/`untrusted`/`granular`、工作区与网络范围、execpolicy、补丁与 MCP 判定、自动审查、授权记忆与边界 |

### plugin-system

#### pi

| # | 文件 | 主题 |
|---|---|---|
| 01 | [01-extension-mechanism.md](plugin-system/pi/01-extension-mechanism.md) | 扩展程序机制：Jiti 源码直跑与三态解析、来源顺序与一层发现规则、加载期「只登记不作用」的两阶段提交、信任两趟装载、注册面四档强度与冲突策略、事件分发的快照+串行+六类合并语义、失败分级（观察类吞掉/工具前置阻断/shell 接管不回落）、工具同构包装与嵌套调用上限、reload 失效协议 |

### prompt-layering

#### codex

| # | 文件 | 主题 |
|---|---|---|
| 01 | [01-research-plan.md](prompt-layering/codex/01-research-plan.md) | 研究计划：Codex 的系统 / 开发者 / 用户三级提示词如何分层、优先级由什么机制保障。含初步核查结论（非单一优先级表，而是角色语义、位置顺序、文本内覆盖声明、配置覆盖链四类机制叠加）与待确认的范围问题 |
| 02 | [02-instruction-precedence-enforcement.md](prompt-layering/codex/02-instruction-precedence-enforcement.md) | 优先级到底靠什么保证：代码层不做冲突裁决；实际靠“放对角色槽位 + 文本里写清层级规则 + 声明失效”三件事，另含系统层主动向下层让步、开发者层自愿服从用户请求、以及一处文字与实现不一致的记录 |

### reasoning-propagation

#### comparison

| # | 文件 | 主题 |
|---|---|---|
| 01 | [01-chain-of-thought-propagation.md](reasoning-propagation/comparison/01-chain-of-thought-propagation.md) | Pi、Codex、Claude Code 的推理块回传对照：模型下发形态、本地持久化、跨轮/工具续轮回放、用户可见性、换模型与压缩边界 |

### sandbox

#### poirot

| # | 文件 | 主题 |
|---|---|---|
| 01 | [01-sandbox-isolation-and-tool-surface.md](sandbox/poirot/01-sandbox-isolation-and-tool-surface.md) | 沙箱的隔离与使用面：三组件（运行时/路径翻译/安全守卫）固定编排、本地档白名单与容器档三层缓存/预热池/空闲回收、懒获取与状态恢复、6 个工具的额外约束、产物复制与下载服务、借给 specialist 的 stdio MCP 桥 |

### session-history

#### claude-code

| # | 文件 | 主题 |
|---|---|---|
| 01 | [01-session-persistence-and-resume.md](session-history/claude-code/01-session-persistence-and-resume.md) | Claude Code 会话历史持久化与 `/resume`：每会话追加式 JSONL、消息 UUID 与 `parentUuid` 主线、会话列表轻量扫描、压缩边界/删除标记/父链修复、文件回退与已读缓存等旁路状态恢复 |
| 02 | [02-resume-command-and-jade-port.md](session-history/claude-code/02-resume-command-and-jade-port.md) | `/resume` 端到端机制：会话内与启动入口、选择器与跨目录、消息链修复、旁路状态切换，以及 Jade Code 复刻所需的会话日志、恢复修复和运行时切换边界 |

### task-system

#### claude-code

| # | 文件 | 主题 |
|---|---|---|
| 01 | [01-research-plan.md](task-system/claude-code/01-research-plan.md) | 研究计划：Claude Code 是否存在与 `learn-claude-code` s10 等价的 Task System，以及它的运行机制。含初步只读核查结论（存在，且为同一设计思路）与待确认的范围歧义（任务清单 vs 后台任务系统） |
| 02 | [02-task-list-mechanism.md](task-system/claude-code/02-task-list-mechanism.md) | 任务清单机制的完整讲解：任务记录结构、编号水位线、双向依赖、认领准入规则、读时解锁、周期性提醒、锁与并发、创建/完成钩子、协作语义，以及与 s10 的逐条对照 |

## 说明

- 被研究的源码仓库（`/home/fishandrice/Backend/codex`、`/home/fishandrice/Backend/claude-code`、`/home/fishandrice/Backend/pi`、`/home/fishandrice/Backend/SkillOpt`）始终只读。
- Claude Code 侧为 `claude-code-2.1.88-source` 源码快照；`feature('...')` 为构建期常量，GrowthBook（`tengu_*`）取值在服务端，因此“生产实际启用哪条分支”无法从本地源码判定。
- 搜索提示：`/home/fishandrice/.gitignore` 内容为 `*`，而 Claude Code 目录不是 Git 仓库，因此在该目录下使用 `rg` 需加 `--no-ignore`。

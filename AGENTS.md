# AGENTS.md

本目录用于**只读研究** Codex、Claude Code、Pi、SkillOpt 等 Agent 相关项目的源码实现，产出可复用的机制认知。这里不是这些项目的开发目录。

## 1. 角色与目标

作为源码机制研究员 + 技术讲师工作：

- 内部研究要深入到代码层（入口、调用链、数据结构、状态变化、条件分支、异常、测试）。
- 对外讲解要抽象到机制层（解决什么问题、何时触发、按什么顺序执行、哪些并行、失败怎么办、为什么这样设计）。
- 用户要的是"理解 Agent 怎么运转"，不是"学习某个仓库的写法"。

不修改被研究的源码仓库，不把研究变成开发任务。

## 2. 工作区与源码目录

### 工作区资料

| 路径 | 用途 |
|---|---|
| `navigation/` | 各源码仓库的架构导航；新会话先读，用于建立全局模型和定位模块 |
| `research-notes/` | 按主题归档的研究笔记；索引见 `research-notes/README.md` |
| `README.md`、`AGENTS.md` | 工作区总览，以及研究流程、报告风格与安全边界 |

### 被研究的源码仓库

| 项目 | 路径 | 形态 |
|---|---|---|
| Codex | `/home/fishandrice/Backend/codex` | Rust workspace（`codex-rs/`，约 180+ crate）；Bazel + Cargo；仓库根有 `docs/` |
| Claude Code | `/home/fishandrice/Backend/claude-code/claude-code-2.1.88-source` | TypeScript 源码快照（`src/`，约 1900 文件）；**没有** package.json / 构建配置，不能构建或运行，只能静态阅读 |
| Pi | `/home/fishandrice/Backend/pi` | TypeScript / npm monorepo（`packages/`：`agent`、`ai`、`coding-agent`、`codemode`、`tui`、`mcp`、`evals`、`telemetry` 等）；有完整 `package.json`、`AGENTS.md` |
| SkillOpt | `/home/fishandrice/Backend/SkillOpt` | Python 3.10+ 包（`pyproject.toml`）：`skillopt/` 研究训练引擎、`skillopt_sleep/` 独立夜间自演化、`skillopt_webui/` Gradio 面板；有 102 个 pytest 文件与 mkdocs 文档 |

参考材料（不是研究对象）：

- `/home/fishandrice/Backend/learn-claude-code`：教学式复刻项目（如 `s10_task_system`），可用于对照概念，不评价其实现质量。

### 搜索注意事项

- `/home/fishandrice/.gitignore` 内容为 `*`，而 `claude-code-2.1.88-source` 不是 git 仓库 → 在该目录用 `rg` 必须加 `--no-ignore`，否则整棵树被忽略。
- Codex 是 git 仓库，`rg` 正常；但注意跳过 `MODULE.bazel.lock`、`app-server-protocol/schema/`（生成代码）与 `third_party/`、`vendor/`。
- SkillOpt 是 git 仓库，`rg` 正常；跳过 `index.html`、`skillopt.html`、`docs/*.html`、`blog/`（渲染后的静态页）、`tests/fixtures/` 与 `__pycache__/`，源码从 `scripts/`、`skillopt/`、`skillopt_sleep/` 进入。
- 每个被研究仓库自身可能有 `AGENTS.md`（Pi 根目录有；Codex 有嵌套的），读它是为了理解项目约定，不代表可以修改。

## 3. 每个研究问题的标准流程

> **对齐优先**：在问题范围、术语和目标完成对齐之前，不做具体源码调查，不写研究报告，也不把对齐过程持久化（不建研究笔记、不改索引、不提交）。对齐只在对话中完成；用户明确确认范围后，才进入源码追踪与报告产出。

```text
对齐 → 研究计划（对话中） → 用户确认 → 源码追踪 → 结论验证 → 机制讲解与归档
```

1. **对齐**：用已有知识解释术语、可选范围与取舍，确认用户真正要解决的问题；不查源码、不预设结论。
2. **研究计划（对话中）**：说明四件事——研究目标（复述用户问题）、研究对象、拟研究内容（按机制划分，不按文件划分）、预期成果；有歧义就提出具体问题。计划只在对齐对话中呈现，不落盘。
3. **等待确认**：用户明确同意后才进入正式研究。计划本身不预设结论。未确认前不得开始源码调查，也不得生成或归档研究报告。
4. **源码追踪**：先外后内——定位入口 → 追主调用链 → 看数据结构与状态 → 关键分支与异常 → 必要处结合测试判断预期行为。不做无穷尽的调用链扩张。
5. **结论验证**：把结论分成三类，分别标注：
   - 已证实：源码/测试可直接确认；
   - 合理推断：结构与上下文支持，但没有直接证据；
   - 无法确认：现有材料不足（例如服务端开关的真实取值、模型训练层面的服从性）。
6. **机制讲解**：按第 5 节风格输出。

追问规则：

- 同一主题的追问（更细的时机、边界、失败路径）可直接继续研究，不必重新提计划。
- 范围换成另一个项目/另一套机制时，重新给计划并等待确认。
- 对齐阶段的所有讨论都只留在对话中，不写入 `research-notes/`，也不更新索引或提交。

## 4. 关键研究方法与陷阱

- 优先用 `rg` / `find` / `git grep`；只读命令辅助分析。
- 源码是**第一事实来源**：README 与公开印象可能过时。已遇到实例：Codex `memories/README.md` 声称运行时编排在 `codex-rs/core/src/memories/`，该目录实际不存在；文档中的模板路径也有缺失。
- 区分"结构上必然成立"与"仅在开关打开时成立"。Codex 和 Claude Code 都有大量构建期常量与远端开关（Codex 的 feature、Claude Code 的 GrowthBook `tengu_*`），线上默认值不可从仓库判断。
- 跨项目比较时，不要假设 A 有的机制 B 也一定有；以各自源码为准。

## 5. 报告风格（机制层）

默认结构：**场景 → 机制展开 → 条件与边界 → 设计取舍 → 证据分级**。

硬性要求：

- 正文用自然语言解释"谁在什么时候做了什么、为什么"，少用函数名、变量名、内部类型名；只在名称本身是重要概念或避免歧义时保留，并说明其职责。
- 不用"函数 A 负责启动、函数 B 负责筛选"式的逐函数罗列；按**行为阶段**组织。
- 保留影响行为的关键细节：触发条件、并行/阻塞关系、状态变化、容量上限、超时/取消、失败降级、去重策略、关键数值。
- Mermaid 节点用机制语言（"主模型""后台检索""上下文注入""重复内容过滤"），不要用文件名或函数名；并发关系必须如实画出。
- 源码位置集中放在文末「源码依据」表格（相对路径 + 关键函数/行号），正文不穿插路径与行号。
- 专业术语（异步、预取、上下文注入、Token 预算等）可直接使用；不够直观的首次出现时用一句话解释。
- 设计动机若无注释支撑，明确标注为推断。

## 6. 输出与持久化

研究性问答默认归档到本工作区：

```text
/home/fishandrice/Backend/explore-mature-agents/research-notes/<主题>/<agent>/<NN>-<slug>.md
```

- 每轮新建一个文件，不追加、不覆盖；编号在 `<主题>/<agent>/` 内独立递增。
- 文件结构：一级标题（主题 - agent - 编号 - 讨论主题）→ 用户提问原文 → 完整答复。标题与开头格式统一如下（正文内容结构不作要求）：

```markdown
# <主题> - <Agent> - <编号> - <副标题>

---

<用户提问原文>

---

<正文>
```
- 对齐完成后的正式研究轮次照常归档；范围对齐本身、研究计划的确认过程与细节澄清不归档。
- 聊天窗口只回文件路径与必要的简短提示；完整答复写在文件里。
- 笔记讨论 Agent Harness 本身，不需要写成对某个具体项目的改造建议。
- **例外**：与源码探索无关的运维/元指令轮次（例如写本 `AGENTS.md`、调整环境、修工作流）不必归档到该目录。
- 该目录**只放 Agent 架构与机制相关的研究笔记**；不要在其中创建会话规则、流程说明、环境配置之类的文件。

## 7. Git 提交与远端同步

本工作区使用 Git 记录研究资料；被研究的源码仓库仍保持只读。

### 提交单位

- 研究笔记按 `<主题>/<Agent>` 分组提交，一次只提交一个主题与一个 Agent 的内容。
- 工作区文档、导航、索引等元信息改动单独提交，不和研究笔记混在一起。
- 只提交与本次研究或文档改动相关的文件；临时脚本、构建产物和无关文件不要提交。

### 提交信息

统一格式：

```text
[<主题>@<Agent>] <内容>
```

- `<主题>`：英文 kebab-case，如 `memory-system`、`multi-agent`。
- `<Agent>`：`codex`、`claude-code`、`pi`、`skillopt`；跨 Agent 对照使用 `comparison`。
- 工作区元信息使用 `[workspace@<范围>]`，例如 `[workspace@navigation]`、`[workspace@git]`。
- `<内容>`：简短说明本次提交做了什么，使用中文。
- 示例：
  - `[memory-system@codex] 添加 Codex 记忆系统笔记`
  - `[task-system@claude-code] 添加任务清单机制笔记`
  - `[workspace@git] 补充提交规范`

### 提交与推送

- 提交前检查工作区，确保 `git status` 中没有遗漏或无关改动。
- 默认推送到 `origin main`；当前公开仓库为 `fishandrice000000/explore-mature-agents`。
- 不默认改写远端历史。只有用户明确要求覆盖远端时，才使用 force push；执行前确认目标分支与影响范围。

## 8. 安全与边界

- 所有源码仓库（Codex、Claude Code、Pi、SkillOpt）**始终只读**：不修改、不创建、不删除、不格式化，不执行 git 写操作，不运行会写缓存/构建产物或联网调用模型的构建与测试。
- 研究脚本与临时产物放 `/tmp` 或本目录，不放进源码仓库。
- 不擅自扩大研究范围；不因为某个机制"看起来相关"就顺带研究。
- 不把推断当事实；不确定就说不确定。
- 涉及删除、覆盖、迁移等破坏性操作时，先确认目标再执行。

## 9. 已有研究主题（仅导航）

历史笔记位于 `/home/fishandrice/Backend/explore-mature-agents/research-notes`，可作为背景知识，但结论需按源码重新核对：

- `memory-system/claude-code`：记忆总览、召回预取、召回开关、预取时效性。
- `memory-system/codex`：启动期记忆流水线、召回与 citation 闭环。
- `session-history/claude-code`：会话 JSONL 持久化、`/resume` 选择与恢复、压缩边界与旁路状态。
- `task-system/claude-code`：模型维护的任务清单（依赖、认领、提醒、并发）。
- `prompt-layering/codex`：三级提示词与优先级"保证"机制。
- `multi-agent/claude-code`：常驻队友/团队的开启、分配、通信、隔离与并发。

## 10. 仓库架构索引（新会话先读）

开始任何源码研究前，**先读对应仓库的架构导航**，再按其中的“问题到目录”表和源码锚点做定向检索。不要在新会话里重新遍历顶层目录或从 README 重新猜模块职责。

| 仓库 | 架构导航 | 第一入口 | 关键边界 |
|---|---|---|---|
| Codex | [`navigation/codex.md`](navigation/codex.md) | `codex-rs/cli/src/main.rs` | TUI、exec、IDE/SDK 通过 app-server 汇合；core 管 thread/turn/session；rollout 管历史 |
| Claude Code | [`navigation/claude-code.md`](navigation/claude-code.md) | `src/entrypoints/cli.tsx` → `src/main.tsx` | REPL 负责交互编排；`query.ts` 是共同 agent 循环；快照不可构建 |
| Pi | [`navigation/pi.md`](navigation/pi.md) | `packages/coding-agent/src/main.ts` | `agent-core` 是通用循环；`coding-agent` 组合会话与模式；`durable` 是独立运行时 |
| SkillOpt | [`navigation/skillopt.md`](navigation/skillopt.md) | `scripts/train.py` | 不是 Agent Harness：离线优化 Markdown 技能文档；`skillopt_sleep` 与 `skillopt/` 无代码依赖 |

使用约定：

- 架构导航只负责**定位和建立全局模型**，不替代对具体实现的源码追踪。
- 若架构文档与源码不一致，以源码为准，并在完成研究后更新对应导航文档。
- Claude Code 快照使用 `rg --no-ignore`；Codex 和 Pi 跳过生成物、依赖目录与构建产物；SkillOpt 跳过静态页与测试 fixture。
- 架构导航的维护轮次属于元指令/工作流文档，不归档到 `research-notes/`。

# explore-mature-agents

只读研究 Codex、Claude Code、Pi、SkillOpt 等成熟 Agent 相关项目的源码实现，产出可复用的机制认知。这里不是这些项目的开发目录。

## 目录结构

| 路径 | 用途 |
|---|---|
| [`navigation/`](navigation/) | 各源码仓库的架构导航：运行平面、主调用链、问题到目录、源码锚点 |
| [`research-notes/`](research-notes/) | 按研究主题归档的机制笔记；完整索引见 [`research-notes/README.md`](research-notes/README.md) |
| [`AGENTS.md`](AGENTS.md) | 研究工作流、结论分级、报告风格与安全边界 |

## 快速入口

- 开始一项新的源码研究：先读 [`navigation/README.md`](navigation/README.md)，再进入对应仓库的架构导航。
- 查找已有研究结论：读 [`research-notes/README.md`](research-notes/README.md)。
- 了解研究流程与归档规则：读 [`AGENTS.md`](AGENTS.md)。

## 被研究的源码仓库

| 项目 | 路径 | 形态 |
|---|---|---|
| Codex | `/home/fishandrice/Backend/codex` | Rust workspace（`codex-rs/`）；TUI、exec、IDE/SDK 通过 app-server 汇合 |
| Claude Code | `/home/fishandrice/Backend/claude-code/claude-code-2.1.88-source` | TypeScript 源码快照，无构建配置，只能静态阅读 |
| Pi | `/home/fishandrice/Backend/pi` | TypeScript / npm monorepo，包含 agent、coding-agent、ai、tui、durable 等包 |
| SkillOpt | `/home/fishandrice/Backend/SkillOpt` | Python 3.10+ 包（`skillopt`、`skillopt_sleep`、`skillopt_webui`），把 Markdown 技能文档当作可训练参数的离线优化器 |
| Poirot | `/home/fishandrice/Backend/poirot` | Python 3.12 + LangGraph 1.x 的单进程研究 Agent；一个 Leader ReAct 图配 20 余个横切中间件，属自研未成熟项目 |

这些仓库始终只读：不修改、不创建、不删除文件，也不在仓库内运行会写缓存、构建产物或联网的命令。研究脚本与临时产物放在 `/tmp` 或本工作区。

## 研究产出

- 研究笔记归档在 `research-notes/<主题>/<agent>/<NN>-<slug>.md`，每轮新建一个文件，不追加、不覆盖。
- 正文以机制为中心，说明触发条件、执行顺序、并发关系、状态变化、失败降级与设计取舍。
- 笔记讨论 Agent Harness 本身，不要求写成对某个具体项目的改造建议。
- 架构导航只负责定位和建立全局模型，不替代源码追踪；文档与源码不一致时，以源码和测试为准。

## 搜索提示

- Claude Code 快照不是 Git 仓库，且 `/home/fishandrice/.gitignore` 内容为 `*`，在该目录使用 `rg` 必须加 `--no-ignore`。
- Codex 研究时跳过 `MODULE.bazel.lock`、`app-server-protocol/schema/` 等生成内容，以及 `third_party/`、`vendor/`。
- Pi 研究时跳过 `dist/`、`node_modules/` 与生成的模型文件，优先从 `packages/*/src/` 进入。
- SkillOpt 是 Git 仓库；研究时跳过 `index.html`、`skillopt.html`、`docs/*.html`、`blog/` 等静态页与 `tests/fixtures/`，入口在 `scripts/` 与 `skillopt/`。
- Poirot 是 Git 仓库；`design_docs/`、`openspec/`、`skills/`、`.poirot/` 被 gitignore 因而不在仓库内，注释里的设计文档编号无法回查。入口在 `poirot/backend/app/bootstrap.py` 与 `agents/leader/factory.py`。

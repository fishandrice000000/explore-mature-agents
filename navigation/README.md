# 架构导航

本目录保存各被研究对象的总览式架构导航。它们帮助新会话快速建立全局模型，定位到正确的模块与入口，然后回到源码做定向追踪。

## 导航列表

| 项目 | 源码路径 | 导航文档 | 关键边界 |
|---|---|---|---|
| Codex | `/home/fishandrice/Backend/codex` | [`codex.md`](codex.md) | Rust workspace；TUI、exec、IDE/SDK 通过 app-server 汇合，thread/turn/session 运行在 core |
| Claude Code | `/home/fishandrice/Backend/claude-code/claude-code-2.1.88-source` | [`claude-code.md`](claude-code.md) | React/Ink 终端应用；REPL 负责交互编排，`query.ts` 是共同 agent 循环 |
| Pi | `/home/fishandrice/Backend/pi` | [`pi.md`](pi.md) | npm monorepo；agent-core 提供通用循环，coding-agent 组合 CLI 与会话，durable 是独立运行时 |
| SkillOpt | `/home/fishandrice/Backend/SkillOpt` | [`skillopt.md`](skillopt.md) | Python 包；不是 Agent Harness，而是离线优化 Markdown 技能文档的训练器，另有独立的夜间自演化包 |
| Poirot | `/home/fishandrice/Backend/poirot` | [`poirot.md`](poirot.md) | Python 3.12 + LangGraph；单 Leader ReAct 图 + 20 余个横切中间件，能力默认值分散在各自开关，属自研未成熟项目 |
| Hermes Agent | `/home/fishandrice/Backend/hermes-agent` | [`hermes-agent.md`](hermes-agent.md) | Python 主程序 + TypeScript 工作区；CLI/TUI/消息网关/dashboard/桌面/ACP 多入口共享一个 `AIAgent`，状态集中在一个 SQLite 会话库；`gateway/` 与 `tui_gateway/` 是两套不同后端 |

前三份导航在 2026-10-17 核对，SkillOpt 与 Poirot 于 2026-10-27 新增，Hermes Agent 于 2026-10-28 新增（核对 `main` @ `baa344dee7`）。导航文档不是行为结论，具体机制仍以源码和测试为准。

## 使用顺序

1. 先读本页，确认研究对象和它当前的仓库形态。
2. 读对应仓库的导航，建立“入口 → 主调用链 → 状态与持久化”的整体模型。
3. 用导航中的“问题到目录”表缩小范围，再做定向检索。
4. 进入具体实现、数据结构和测试验证结论，不把导航当作最终证据。

## 维护约定

- 导航只记录稳定的结构信息：运行平面、主调用链、模块职责、问题到目录、源码锚点。
- 源码发生变化时，完成研究后同步修正导航，并更新核对日期。
- 涉及开关的能力要区分“结构上存在”与“默认开启”；远端开关的真实取值不能从本地仓库推断。
- 所有源码仓库始终只读；不要在其中创建文件、格式化代码或运行会写缓存、产物或联网的命令。

## 相关文档

- 研究笔记索引：[`../research-notes/README.md`](../research-notes/README.md)
- 研究流程、结论分级与报告风格：[`../AGENTS.md`](../AGENTS.md)

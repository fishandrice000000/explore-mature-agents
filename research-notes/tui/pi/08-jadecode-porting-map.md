# TUI - Pi - 08 - JadeCode 移植对照与实施路线

---

1. C
2. 主要是视觉布局与交互.
3. JadeCode 基于 Java, 优先 Ubuntu.
4. 需要移植对照.

你可以顺带研究一下 JadeCode 的源码: /home/fishandrice/Backend/JadeCode/src/main/java/com/jadecode

---

## 0. 结论先说

JadeCode 已经有的不是“空白”，而是**一条正确但简化过的内联渲染链**：

```text
TuiApplication（UI 线程 + 事件队列 + 16ms 调度）
  → TuiController（输入、选择器、审批、命令）
  → ConversationView（把 TuiState 拼成完整 Frame）
  → InlineRenderer（主屏差分 + 同步输出）
  → TerminalSession（JLine raw mode）
```

这条链的线程模型、业务桥接、审批、事件队列都可以保留。缺的是 Pi 的**通用组件层、布局层、全屏渲染器、Overlay/焦点、输入协议、主题/图片/补全**。

所以不要重写 `SessionRuntime`、`AgentExecutor`、`Approval`、`SubagentRuntime`。要新增的是一层“Pi 风格 TUI 引擎”，然后把现有 `UiEvent`/`TuiState` 接到这层引擎上。

建议的目标分层：

```text
com.jadecode.tui.core         组件、容器、TUI 基类、主屏/全屏渲染器、布局、Overlay、焦点、鼠标
com.jadecode.tui.terminal     JLine 适配、输入序列缓冲、键解析、键绑定、能力探测
com.jadecode.tui.text         字素/宽度、ANSI 安全行、切列、合成
com.jadecode.tui.components   Box/Text/Input/Editor/Markdown/SelectList/ScrollView/Loader/Image/MouseRegion
com.jadecode.tui.theme        token、OKLCH/OKHSL、system/dark/light
com.jadecode.cli.tui          现有 TuiApplication/TuiController/TuiState + 新的 View 类
```

---

## 1. 现状与 Pi 的逐项对照

### 1.1 渲染与布局

| 能力 | Pi | JadeCode 现状 | 差距 | 优先级 |
|---|---|---|---|---|
| 主屏内联渲染 | `TuiMainScreen`，终端 scrollback 管理历史 | `InlineRenderer`，已有差分/同步输出/底部锚定 | 接近，但底部锚定与 Pi 不同；缺少组件树/Overlay/光标 marker | P0 |
| 备用屏全屏 | `TuiAltScreen`，固定视口、应用滚动 | 无 | 完全缺失 | P0 |
| 组件树 | `Component`/`Container`/`Box`/`VStack`/`HStack` | `ConversationView` 直接拼 `StyledLine`；`VerticalStack` 只是顺序拼接 | 缺少可复用组件、缓存、焦点、鼠标分发 | P0 |
| 布局树 | `layout.ts`，rect/clip/stack/scroll | 无 | 无法做 fullscreen dock、Overlay、滚动条 | P0 |
| 文本宽度 | `visibleWidth` + grapheme + emoji/CJK 规则 | `DisplayWidth`，`\X` + WCWidth + emoji | 规则不同；需要逐项对齐 | P0 |
| 折行/截断 | `wrapTextWithAnsi`/`truncateToWidth`/`sliceByColumn` | `StyledLine.wrap`/`DisplayWidth.take/ellipsize` | 结构不同；需要 ANSI/宽字符/边界一致 | P0 |
| Overlay | 栈、锚点/百分比/margin、焦点恢复、合成 | 无；选择器/审批通过布局顺序覆盖 | 缺少通用 Overlay | P0 |
| 滚动 | `ScrollView` + 滚动条 + follow-end | 无（主屏靠终端 scrollback） | fullscreen 必需 | P0 |
| 鼠标 | 全屏滚轮/拖拽/选择/滚动条 | 只解析左键点击；启动时关闭鼠标捕获 | 缺少滚轮/拖拽/选择 | P1 |
| 光标/IME | `CURSOR_MARKER` + 假光标 + 硬件光标 | 直接定位硬件光标；无 marker/fake cursor | IME 位置可能不准；光标视觉不同 | P1 |

### 1.2 输入与编辑

| 能力 | Pi | JadeCode 现状 | 差距 | 优先级 |
|---|---|---|---|---|
| 传统按键 | 完整 | `InputDecoder` 支持常见序列 | 缺 Home/End/Page/词移动的部分变体 | P1 |
| Kitty 键盘协议 | 协商 + CSI-u + release/repeat + alternate keys | 无 | 现代终端下 Shift+Enter/Ctrl+Shift 等行为不同 | P1 |
| modifyOtherKeys 回退 | 有 | 无 | 同上 | P2 |
| 键绑定注册表 | `KeybindingsManager` + 用户配置 | `switch` 硬编码 | 无法配置/冲突检测 | P1 |
| 编辑器 | 多行、视觉行映射、粘性列、撤销合并、kill ring、jump、补全、粘贴标记 | 多行、软换行、简单 undo、粘贴标记、历史 | 缺粘性列、kill ring、jump、补全、复杂选择 | P1 |
| 自动补全 | 斜杠/命令参数/`@` 文件/路径 + fd + 去抖/取消 | 只有简单命令候选 | 缺文件补全、参数补全、竞态处理 | P1 |
| 大粘贴 | >10 行/1000 字符折叠为标记，提交展开 | 已有类似机制 | 规则/编号重排/原子性需要对齐 | P2 |
| 鼠标点击编辑 | 全屏/主屏点击定位 | 只支持审批/列表点击 | 缺编辑器点击定位 | P2 |

### 1.3 消息与工具

| 能力 | Pi | JadeCode 现状 | 差距 | 优先级 |
|---|---|---|---|---|
| 用户消息 | Markdown + 全宽背景 + OSC133 | 有 `Entry` 卡片，标题+横线+Markdown | 视觉结构不同 | P0 |
| 助手消息 | 文本/thinking 分块、可折叠、错误/截断 | 有流式 Reply/Thinking 卡片 | 缺块级顺序/折叠/错误布局 | P0 |
| 工具卡 | 状态背景、call/result renderer、diff、图片、展开 | 统一 `Tool` 卡片，参数+预览文本 | 缺工具专属视觉 | P0 |
| bash 执行卡 | 独立边框、命令头、尾部预览、loader | 有 bash 工具卡/执行通知 | 缺独立视觉 | P1 |
| 工具结果图片 | Kitty/iTerm2 | 无 | 可选 | P2 |
| diff | 行级颜色 + 单词级反色 | 无 | edit/write 视觉差距大 | P1 |
| 自定义消息 | 扩展 renderer | 有 Custom 消息 | 接近 | P2 |
| compaction/branch summary | 专用组件 | 有 Notice/压缩通知 | 接近 | P2 |

### 1.4 底部区域与选择器

| 能力 | Pi | JadeCode 现状 | 差距 | 优先级 |
|---|---|---|---|---|
| header/logo | logo + 版本 + 快捷键 + 展开 | 简单标题 | 缺品牌视觉/帮助 | P2 |
| pending 队列 | Steering/Follow-up + 提示 | 有 `Queued` 行 | 文案/位置不同 | P1 |
| 状态 spinner | 嵌入 editor 上边框 | 独立状态行 | 视觉差异明显 | P1 |
| footer | 两行 + 右侧模型 + 扩展状态 | 有目录/model/permission/tokens/context | 排版/统计项/右对齐不同 | P1 |
| 选择器 | 替换 editor，多种选择器 | 简单 `SelectList` | 缺模型 scope/settings/session/tree | P1 |
| 全屏搜索 | Overlay + 高亮 + 跳转 | 无 | fullscreen 必需 | P1 |
| 全屏选择复制 | 鼠标拖选 + OSC52/原生 | 无 | fullscreen 交互 | P1 |

### 1.5 主题与 Markdown

| 能力 | Pi | JadeCode 现状 | 差距 | 优先级 |
|---|---|---|---|---|
| 主题 token | ~55 个、fg/bg、fallback | 16 个左右命名颜色 | 缺 token 系统 | P0 |
| 颜色模型 | indexed/sRGB/OKLCH/OKHSL | 16 色 SGR 替换 | 视觉差异大 | P0 |
| 默认主题 | `system`（从终端生成） | `dark`/`light` 固定 | 默认就不一致 | P1 |
| Markdown | marked + LaTeX + 严格删除线 + 表格算法 | CommonMark + GFM tables + 正则高亮 | 缺 LaTeX/流式围栏/链接/高亮 | P1 |
| 代码高亮 | highlight.js + token 映射 | 简单正则 | 缺语言覆盖/颜色 | P1 |
| 图片 | Kitty/iTerm2 + cell size | 无 | 可选 | P2 |

---

## 2. 关键差异的源码级解释

### 2.1 JadeCode 的“卡片”不是 Pi 的消息组件

JadeCode `ConversationView.entry()`：

```text
header(title + 状态 + 用时) + 横线
arguments（tool）
body（Markdown / 纯文本 / thinking）
```

Pi：

- 用户消息：一整块全宽背景的 Markdown，没有标题横线；
- 助手消息：Markdown 文本 + thinking 块，没有标题横线；
- 工具：Box 背景 + call renderer + result renderer，没有“标题横线”；
- bash 执行：上下边框 + 命令头 + 输出。

因此即使颜色/宽度完全一样，JadeCode 当前看起来也会像“日志列表”，而不是 Pi 的“对话 + 工具卡”。

### 2.2 JadeCode 的底部锚定与 Pi 主屏不同

`InlineRenderer.anchorBottom()` 会把输入区/页脚推到当前窗口底部，并在前面补空行。Pi 主屏模式不做这个：它把整篇文档顺序写到终端，输入框跟在内容后面，内容增长时终端自然滚动。

这带来两个行为差异：

- 内容很少时，JadeCode 的输入框在屏幕底部，Pi 的输入框紧跟在内容后面；
- 内容很多时，JadeCode 的“补空行”会改变 scrollback 位置；Pi 不会。

如果目标是 Pi 的 `regular` 模式，应让底部锚定成为可选/关闭；如果目标是 fullscreen，底部固定由 `ScrollView + dock` 实现，不需要 `anchorBottom`。

### 2.3 JadeCode 没有真正的“焦点”

当前优先级是：

```text
approval > selector > command suggestions > editor
```

这是用布局顺序和 `if` 实现的。Pi 是：

```text
Overlay 栈 + focusedComponent + preFocus + nonCapturing + 可见性回调
```

差异会在这些场景暴露：

- 选择器打开时又弹出扩展 Overlay；
- 关闭子对话框后焦点回到错误的层级；
- 点击 Overlay 外部是否应该穿透；
- 全屏搜索面板与主编辑器同时存在。

### 2.4 JadeCode 的输入协议只覆盖“旧终端”

`InputDecoder` 支持：

- `\x1b[A/B/C/D`、`\x1b[1;5D/C`、Home/End/Delete/PageUp/PageDown；
- SGR 左键点击；
- bracketed paste；
- 孤立 ESC 120ms 超时。

缺：

- Kitty CSI-u（现代终端下很多组合键根本到不了应用）；
- key release/repeat；
- Shift+Enter 的可靠区分；
- 鼠标滚轮/拖拽/选择；
- focus in/out；
- 终端回复（颜色、cell size、Kitty graphics）的识别。

这会让“交互 1:1”在 Ubuntu 的 Kitty/Ghostty/WezTerm 上明显不一致。

---

## 3. 目标架构：在 JadeCode 中复刻 Pi 的分层

### 3.1 包与职责

```text
com.jadecode.tui.core
  Component                  render(width) -> List<Line>; handleInput?; handleMouse?; invalidate()
  Container                  子组件列表 + 鼠标分发
  Tui                        addChild/removeChild/setFocus/showOverlay/start/stop/requestRender
  TuiBase                    焦点、Overlay 栈、输入监听、16ms 调度、生命周期
  TuiMainScreen              主屏差分渲染
  TuiAltScreen               备用屏 + 布局根 + 鼠标/选择/搜索/滚动
  OverlayHost                Overlay 栈、布局、合成、焦点恢复
  LayoutEngine               rect/clip/stack/scroll 布局
  MouseRouter                屏幕坐标 → 组件局部坐标

com.jadecode.tui.terminal
  TerminalSession            JLine 适配（已有，升级）
  InputSequencer             拆包、粘贴、超时、终端回复识别
  KeyParser                  legacy + Kitty + modifyOtherKeys
  KeyMatcher                 keyId 匹配、release/repeat、逻辑符号/base layout
  Keybindings                动作 → 键列表、用户覆盖、冲突检测
  CapabilityDetector         truecolor/256、图片、hyperlinks、Kitty

com.jadecode.tui.text
  GraphemeWidth              与 Pi 对齐的 grapheme/宽度规则
  AnsiLine                   一行 + 样式片段 + 可选 OSC8
  LineOps                    wrap/truncate/slice/composite/extractSegments
  CursorMarker               APC marker + fake cursor

com.jadecode.tui.components
  Box, Text, TruncatedText, Spacer
  VStack, HStack, ScrollView
  Input, Editor, SelectList, SettingsList
  Markdown, Loader, Image, MouseRegion

com.jadecode.tui.theme
  ThemeToken, Theme, Color, Oklch, Okhsl, SystemTheme

com.jadecode.cli.tui
  TuiApplication             现有生命周期/队列保留
  TuiController              现有输入/命令/审批保留
  TuiState                   现有业务显示状态保留
  views/HeaderView
  views/UserMessageView
  views/AssistantMessageView
  views/ToolCardView
  views/Footer
  views/PendingView
```

### 3.2 行表示：继续用 `StyledLine`，还是改成 ANSI 字符串？

Pi 用“带 ANSI 的字符串”作为行，因为 JS 的字符串拼接/比较很方便。Java 有两条路：

**A. 继续用 `StyledLine`（推荐）**

- 保持现有 `Span(text, style)`；
- 增加 `AnsiStyle`（fg/bg/attrs）和 `Link`；
- 在最终 `Theme.encode()` 时生成 ANSI；
- `LineOps` 直接在 span 上做宽度、切列、合成、wrap；
- 优点：不需要 ANSI 解析器；选择/复制/搜索可以直接拿明文；Overlay 合成不需要跟踪 SGR 状态；
- 缺点：要与 Pi 的 ANSI 行为逐项对齐（尤其是背景跨行、OSC8、underline reset）。

**B. 改成 ANSI 字符串**

- 更贴近 Pi 源码，可以直接移植 `wrapTextWithAnsi`/`extractSegments`；
- 但 Java 需要完整 ANSI 解析器、状态跟踪、OSC/APC 处理；
- 现有 `Markdown`/`Theme` 都要改成输出 ANSI 字符串；
- 选择/复制要重新剥 ANSI。

建议 A：JadeCode 已经有 `StyledLine` 和 `DisplayWidth`，继续用结构化行更自然；但要把 Pi 的 `AnsiCodeTracker` 语义映射成 `TextStyle` 的继承规则。

### 3.3 渲染器

**`TuiMainScreen`**

- 复用当前 `InlineRenderer` 的差分/同步输出/底部计数思路；
- 增加组件树渲染、Overlay 合成、光标 marker、`clearOnShrink`、Kitty 图片预留行、宽度检查；
- 去掉或可配置 `anchorBottom`，以匹配 Pi regular。

**`TuiAltScreen`**

- 新增 `1049h` 进入/退出、`7l/7h` 自动换行、鼠标模式；
- 用 `LayoutEngine` 渲染固定高度；
- 逐行 diff（`CSI row;1H` + `CSI 2K`）；
- 支持 ScrollView、滚动条、选择、搜索、Overlay、图片；
- 退出时默认把完整 transcript 重放到主屏。

### 3.4 输入

- 保留 `TerminalSession` 的 JLine raw mode，但不要用 `LineReader`；
- 新增 `InputSequencer`，对应 Pi 的 `StdinBuffer`：
  - 缓存不完整 CSI/OSC/DCS/APC；
  - bracketed paste 整段；
  - 孤立 ESC 超时；
  - 识别终端回复，避免把颜色/cell size 回复当按键；
- 新增 `KeyParser`：
  - legacy、Kitty CSI-u、modifyOtherKeys；
  - release/repeat；
  - 逻辑 shifted symbol/base layout；
- 新增 `Keybindings`：默认表 + 用户配置 + 冲突检测；
- 编辑器只接收“完整按键/粘贴”事件，不直接读 `System.in`。

### 3.5 主题

- 把现有 `Theme(String name, boolean colors)` 替换成 token 表；
- 实现 `Color`（indexed/rgb/oklch/okhsl）和 256 色最近邻；
- 内置 dark/light JSON；
- 可选实现 `system`：读取 OSC 10/11/4，按 Pi 的 `system-theme.ts` 生成；
- 至少先支持 truecolor/256 降级，否则 Pi 的 OKHSL 颜色在 Jade 里会变成 16 色近似。

### 3.6 Markdown

- 保留 CommonMark + GFM tables；
- 补：
  - 严格 `~~` 删除线；
  - LaTeX 块/行内；
  - 流式未闭合围栏修剪；
  - `preserveOrderedListMarkers`/`preserveBackslashEscapes`；
  - OSC 8 链接；
  - 代码高亮（先做语言映射 + 轻量词法，或接 JHighlight）；
  - 表格列宽算法按 Pi 的 `3n+1`、min word width、grow/shrink 对齐。

---

## 4. 分阶段落地顺序

### 阶段 0：Parity Harness（先做，不然无法判断“1:1”）

目标：能在没有真实终端的情况下比较 Pi 与 JadeCode 的帧。

- Java 测试侧实现 `VirtualScreen`：
  - 维护 `rows × cols` 的 cell（字符 + fg/bg/attrs）；
  - 解析 SGR、CSI H/J/K、`?1049h/l`、`?25h/l`、`?2026h/l`；
  - 忽略 OSC 8/图片序列但记录其存在；
  - 支持宽字符占两格、组合字符附着。
- 固定尺寸用例：80×24、120×40、40×10、20×6。
- 录制/断言：
  - 可见文本行；
  - 每个 cell 的样式；
  - 光标位置；
  - 帧序列（首帧、追加、单行更新、resize、overlay 开关）。
- 交互脚本：按键序列 + 粘贴 + 鼠标 SGR。

Pi 侧可以用它的 `VirtualTerminal`/xterm headless 测试作为“参考帧”的来源；不需要修改 Pi，只需要读测试/运行已有测试。

### 阶段 1：文本与组件基础

- `GraphemeWidth`：按 Pi 的规则实现，测试 CJK/emoji/regional indicator/组合字符/Tab。
- `AnsiLine`/`LineOps`：wrap/truncate/slice/composite。
- `Component`/`Container`/`Box`/`Text`/`TruncatedText`/`Spacer`。
- `VStack`/`HStack` + `LayoutEngine`（可先只做主屏顺序布局）。
- 验收：固定宽度下的行数/列宽/空白与 Pi 一致。

### 阶段 2：主屏渲染器

- 把 `InlineRenderer` 重构成 `TuiMainScreen`；
- 组件树 → 完整文档 → 差分；
- 加 Overlay 合成、光标 marker、宽度检查；
- 保留 16ms 调度与输入抢占；
- 验收：追加消息、spinner 单行更新、内容缩短、终端变宽/变窄。

### 阶段 3：编辑器与输入

- `InputSequencer` + `KeyParser` + `Keybindings`；
- 升级 `Editor`：
  - 视觉行映射、粘性列、kill ring、undo 合并、jump；
  - 自动补全列表；
  - 大粘贴标记；
  - `CURSOR_MARKER`/假光标；
- 验收：CJK/emoji 光标移动、Shift+Enter、历史、undo、粘贴、Tab 补全、方向键。

### 阶段 4：全屏渲染器与滚动

- `TuiAltScreen` + `LayoutEngine` 的 scroll 节点；
- `ScrollView` + follow-end + 滚动条；
- 鼠标滚轮/拖拽/选择/复制/搜索；
- 退出 transcript 重放；
- 验收：固定 dock、长 transcript、滚轮、PageUp/PageDown、搜索、选择复制、resize。

### 阶段 5：产品层组件

- `HeaderView`、`UserMessageView`、`AssistantMessageView`、`ToolCardView`、`BashExecutionView`、`PendingView`、`FooterView`、`StatusView`；
- 把 `TuiState` 的 entry/stream/tool/approval 映射到组件；
- 选择器替换 editor；
- 验收：流式文本、工具卡创建/更新/结束、thinking 折叠、错误、pending、footer。

### 阶段 6：主题、Markdown、图片

- token 化主题 + dark/light + truecolor/256；
- Markdown 补齐 LaTeX/表格/代码高亮/链接；
- 可选：Kitty/iTerm2 图片、system 主题、剪贴板；
- 验收：颜色序列、代码块、表格、链接、图片回退。

### 阶段 7：Parity 回归与性能

- 建立 golden frames；
- 10k 行 transcript、1M 行 scroll 内容；
- 每帧 render 时间、内存、GC；
- 终端矩阵：Ubuntu 的 GNOME Terminal、Kitty、Ghostty、WezTerm、tmux；
- 记录“已证实/合理推断/无法确认”的差异。

---

## 5. 移植时最容易踩的坑

1. **不要把 `StyledLine` 直接拼成 ANSI 再交给 `InlineRenderer`。** 现有 `Theme.encode()` 每行重置，跨行背景/链接会断；需要 `LineOps` 在合成时保留状态。
2. **不要用 `String.length()` 做列宽。** 现有 `DisplayWidth` 已经做对了一部分，但 emoji ZWJ/regional indicator/Indic mark 要与 Pi 对齐。
3. **不要在每帧重新 parse 所有 Markdown。** Pi 用 token 缓存 + 组件缓存；Jade 当前按 entry version/width 缓存，方向是对的，组件化后要继续保持。
4. **不要让 UI 线程等 fd/git/模型。** 现有 `TuiApplication` 已经把业务放别的线程；补全、git branch、主题加载也要异步，结果通过事件队列回 UI。
5. **不要在 fullscreen 里用主屏的 `anchorBottom`。** fullscreen 的 dock 由布局保证；再用 anchor 会重复补空行。
6. **不要把选择器做成 Overlay。** Pi 的选择器替换 editor，dock 高度会变化；Overlay 会浮在聊天上，视觉不同。
7. **不要忽略 `Ctrl+O` 的双重作用。** 它既展开/收起工具输出，也展开/收起 header；两个组件都要响应。
8. **不要在流式 token 上每 token 重绘。** 16ms 合并；按键才抢占。
9. **不要在 resize 后复用旧鼠标坐标。** `InlineRenderer.actionsValid` 已经处理审批；fullscreen 的布局/选择/滚动条也要在 resize 时失效。
10. **不要假设所有终端都支持图片/truecolor/Kitty。** 必须有降级路径。

---

## 6. 1:1 验收矩阵

| 维度 | 用例 | 通过标准 |
|---|---|---|
| 布局 | 80×24、120×40、40×10 | 行数、列宽、空白、dock 高度与 Pi 一致 |
| 文本 | 中文、emoji、组合字符、长单词、Tab | 折行位置、截断、光标列一致 |
| 编辑器 | 多行、软换行、历史、undo、粘贴 | 光标位置、边框、30% 高度、补全列表一致 |
| 消息 | 用户/助手/thinking/错误/截断 | 背景、间距、折叠、颜色一致 |
| 工具 | read/write/edit/bash/grep/find/ls | 调用头、预览行数、diff、展开行为一致 |
| footer | 不同宽度、长模型名、多 provider | 左右对齐、截断、颜色阈值一致 |
| 主屏 | 首帧、追加、单行更新、缩短、resize | 差分范围、scrollback、光标位置一致 |
| 全屏 | 长 transcript、滚轮、PageUp/Down、搜索、选择 | follow-end、滚动条、高亮、复制一致 |
| Overlay | 居中/锚点/百分比/margin/可见性 | 位置、尺寸、焦点、鼠标命中一致 |
| 输入 | legacy/Kitty/modifyOtherKeys、release、paste | 每个按键/组合键行为一致 |
| 主题 | dark/light/system、truecolor/256 | token → ANSI 序列一致 |
| 性能 | 10k 行、1M 行 scroll、流式 60fps | 无 O(n²)、无可感知卡顿 |

---

## 7. 建议保留 / 改造 / 新增

### 保留

- `TuiApplication` 的 UI 线程 + 队列 + 关闭顺序；
- `TuiController` 的业务命令/审批/选择器逻辑；
- `TuiState` 的 entry/stream/tool/approval/usage 模型；
- `SessionRuntime`/`AgentExecutor`/`Hooks`/`Approval` 的桥接；
- `InputDecoder` 的 bracketed paste/ESC 超时思路（升级而不是丢弃）。

### 改造

- `InlineRenderer` → `TuiMainScreen`（组件树 + Overlay + 光标 marker + 宽度检查）；
- `ConversationView` → 产品层 View 组合（消息/工具/footer/status）；
- `Editor` → Pi 风格编辑器（视觉行、粘性列、kill ring、补全、undo 合并）；
- `Markdown` → token 主题 + 表格算法 + 流式围栏 + 链接；
- `Theme` → token 表 + truecolor/256 + dark/light/system；
- `InputDecoder` → `InputSequencer` + `KeyParser`。

### 新增

- `TuiAltScreen` + `LayoutEngine` + `ScrollView`；
- `OverlayHost` + `FocusManager` + `MouseRouter`；
- `Keybindings` 注册表；
- 工具 renderer 抽象 + 各工具卡；
- 图片/搜索/选择复制；
- 虚拟终端测试 harness。

---

## 8. 粗略工作量（相对，不是承诺）

| 阶段 | 相对量 | 说明 |
|---|---|---|
| Parity harness | M | 没有它，后面只能“目测” |
| 文本/组件/布局 | L | 最底层，影响所有视觉 |
| 主屏渲染器 | M | 现有 `InlineRenderer` 可复用一半 |
| 编辑器/输入 | L | Kitty/补全/粘性列/撤销都很细 |
| 全屏/滚动/鼠标 | L | 新增最多 |
| 产品层组件 | L | 工具卡/消息/footer 各自有细节 |
| 主题/Markdown/图片 | M–L | 取决于是否做 system 主题/图片 |
| Parity/性能 | M | 终端矩阵和 golden frames |

如果只做“regular + 消息/工具/编辑器 + dark/light”，可以跳过全屏/图片/system 主题；但那就不是完整 1:1，只是“Pi 风格”。

---

## 9. 证据分级

- **已证实**：JadeCode 现有类/职责、Pi 的组件/布局/渲染/输入/主题/工具卡实现，均来自源码。
- **合理推断**：建议的分层、阶段顺序、工作量相对量，是基于两边源码结构的工程判断；没有在 JadeCode 上实现验证。
- **无法确认**：JLine 在 Ubuntu 各终端下对 Kitty/鼠标/焦点事件的原始字节行为；需要实际终端测试。Pi 的 `system` 主题在用户终端上的最终色值也取决于终端上报。

---

## 10. 源码依据

### Pi

| 相对路径 | 关键内容 |
|---|---|
| `packages/tui/src/tui.ts` | `Component` 117；`TuiBase` 506；Overlay 729；渲染调度 1001 |
| `packages/tui/src/tui-main-screen.ts` | 主屏差分 247 |
| `packages/tui/src/tui-alt-screen.ts` | 全屏 202；鼠标 901；选择 1588；渲染 1679 |
| `packages/tui/src/layout.ts` | 布局 106/379；绘制 330 |
| `packages/tui/src/components/stack.ts` | 尺寸分配 135 |
| `packages/tui/src/components/scroll-view.ts` | ScrollView 22 |
| `packages/tui/src/utils.ts` | 宽度/折行/截断 250/900/1121 |
| `packages/tui/src/keys.ts` | 键解析/匹配 602/859 |
| `packages/tui/src/components/editor.ts` | 编辑器 297/696 |
| `packages/tui/src/components/markdown.ts` | Markdown 236/852 |
| `packages/tui/src/terminal-image.ts` | 图片能力/编码 70/226/670 |
| `packages/coding-agent/src/modes/interactive/interactive-mode.ts` | 产品组装 608/943/3408/3882 |
| `packages/coding-agent/src/modes/interactive/tui-renderer.ts` | 两种渲染器 22 |
| `packages/coding-agent/src/modes/interactive/chat-viewport.ts` | fullscreen dock 20 |
| `packages/coding-agent/src/modes/interactive/components/*` | 消息/工具/状态/footer/选择器 |
| `packages/coding-agent/src/core/tools/renderers/*` | 工具视觉 |
| `packages/coding-agent/src/modes/interactive/theme/*` | token/主题/system |

### JadeCode

| 相对路径 | 关键内容 |
|---|---|
| `src/main/java/com/jadecode/cli/tui/TuiApplication.java` | UI 生命周期、事件队列、16ms 调度 67-180 |
| `src/main/java/com/jadecode/cli/tui/TuiController.java` | 输入/命令/审批 39-472 |
| `src/main/java/com/jadecode/cli/tui/ConversationView.java` | 整帧组装 27-304 |
| `src/main/java/com/jadecode/cli/tui/TuiState.java` | entry/stream/tool/approval 状态 17-226 |
| `src/main/java/com/jadecode/tui/core/InlineRenderer.java` | 主屏差分 13-125 |
| `src/main/java/com/jadecode/tui/terminal/TerminalSession.java` | JLine 适配 16-86 |
| `src/main/java/com/jadecode/tui/terminal/InputDecoder.java` | legacy 输入 9-87 |
| `src/main/java/com/jadecode/tui/components/Editor.java` | 简化编辑器 14-171 |
| `src/main/java/com/jadecode/tui/components/Markdown.java` | CommonMark + 表格 20-217 |
| `src/main/java/com/jadecode/tui/text/DisplayWidth.java` | grapheme/宽度 13-66 |
| `src/main/java/com/jadecode/tui/theme/Theme.java` | 16 色主题 7-35 |
| `src/main/java/com/jadecode/tui/components/SelectList.java` | 简单选择列表 1-57 |
| `src/main/java/com/jadecode/tui/core/Frame.java` | 行/光标/命中 1-23 |

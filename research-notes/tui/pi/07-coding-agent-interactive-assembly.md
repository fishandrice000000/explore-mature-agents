# TUI - Pi - 07 - Coding Agent 交互层的组装与视觉行为

---

1. C
2. 主要是视觉布局与交互.
3. JadeCode 基于 Java, 优先 Ubuntu.
4. 需要移植对照.

你可以顺带研究一下 JadeCode 的源码: /home/fishandrice/Backend/JadeCode/src/main/java/com/jadecode

---

## 0. 这一层是什么

`packages/tui` 提供“怎么画”，`InteractiveMode` 决定“画什么、什么时候画、用户按键后发生什么”。

它负责：

- 组件树的组装（header、chat、pending、status、editor、footer）；
- 两种布局（regular 的整篇文档 / fullscreen 的 transcript + dock）；
- Agent 事件到消息组件的映射；
- 工具调用的卡片化展示；
- 流式更新与 pending 队列；
- 选择器、对话框、Overlay；
- 主题、补全、快捷键、会话恢复；
- 退出时的 transcript 重放。

---

## 1. 初始化与组件树

### 1.1 构造

`InteractiveMode` 构造时创建：

```text
renderer = createInteractiveTui({ tuiMode, ... })
ui = createInteractiveTuiReference(() => renderer)

headerContainer
loadedResourcesContainer
chatContainer
documentContainer = header + loadedResources + chat
pendingMessagesContainer
statusContainer
widgetContainerAbove / widgetContainerBelow
editorContainer -> defaultEditor (CustomEditor)
footerContainer -> FooterComponent
```

`documentContainer` 是 transcript 的“文档”；`chatContainer` 只放消息和工具卡，后续 `renderInitialMessages()`、`renderSessionEntries()` 都往这里加。

`defaultEditor` 是 `CustomEditor extends Editor`，带：

- `app.*` 动作处理；
- `embedWorkingStatus: true`，把 working 状态嵌进上边框；
- `onCtrlD`、`onEscape`、`onPasteImage`、`onExtensionShortcut`；
- 主题 `getEditorTheme()`，边框色默认 `borderMuted`，thinking 时按 level 变色，bash 模式变 `bashMode`。

### 1.2 regular 模式

`mountInteractiveTui()` 在 regular 模式下把组件按顺序直接加到 `TuiMainScreen.children`：

```text
documentContainer
pendingMessagesContainer
statusContainer
widgetContainerAbove
editorContainer
widgetContainerBelow
footerContainer
```

主屏渲染器会把整棵树渲染成一篇文档，终端 scrollback 负责历史。输入框不在屏幕底部固定，而是跟在对话后面；对话增长时输入框自然下移，终端滚动。

### 1.3 fullscreen 模式

fullscreen 模式仍然把这些组件加到 `children`，但同时调用 `setLayoutRoot(fullscreenLayoutRoot)`：

```text
VStack
  ├─ ScrollView(documentContainer)  basis 0, grow 1, shrink 1, minSize 1
  └─ VStack(dock)                   basis auto, grow 0, shrink 1, minSize 1
       ├─ pendingMessagesContainer  shrink 1, minSize 0
       ├─ statusContainer           shrink 1, minSize 0
       ├─ widgetContainerAbove      shrink 1, minSize 0
       ├─ editorContainer           shrink 1, minSize 3
       ├─ widgetContainerBelow      shrink 1, minSize 0
       └─ footerContainer           shrink 1, minSize 0
```

`ScrollView` 配置：

- `follow: "end"`：默认贴底；
- `primary: true`：接收键盘 PageUp/PageDown、prompt 跳转、搜索；
- `overscroll: "chain"`：滚轮在 transcript 上滚到头后，继续给外层（通常没有外层，但支持嵌套）；
- scrollbar 由设置决定 `auto/always/hidden`；
- track/thumb 颜色来自 `scrollbarTrack`/`scrollbarThumb`。

fullscreen 下输入区固定在底部，transcript 自己滚动；这是 Pi 默认的视觉。

### 1.4 初始化顺序

`init()` 的关键顺序：

1. 注册信号处理；
2. 读 changelog；
3. `renderWidgets()` 初始化 widget 容器；
4. `createChatViewport()` 创建 fullscreen 布局；
5. `mountInteractiveTui()` 挂载组件；
6. 给 editor 设置启动阶段的 `onSubmit`（只提示“startup in progress”）；
7. `ui.setFocus(editor)`；
8. `ui.start()`；
9. `programStatus.report()`；
10. 应用主题，等待终端颜色查询（最多 100ms 或 DA1）；
11. 构建 header；
12. 异步确保 `fd`/`rg` 可用；
13. 启用完整键处理、提交处理；
14. 绑定扩展；
15. 渲染初始消息；
16. 启动主题文件 watcher、git branch watcher；
17. 异步加载语法高亮语言，完成后 invalidate。

为什么 header 要等终端颜色：header 的颜色在构建时就被烘焙进字符串，`system` 主题需要终端上报的 palette/background；等 DA1 或 100ms 后再构建，避免先用猜的颜色画一遍再变。

---

## 2. Header 与 loaded resources

### 2.1 内置 header

`headerContainer` 结构：

```text
Spacer(1)
BuiltInHeader
Spacer(1)
```

`BuiltInHeader` 是 `ExpandableText extends ThemedText`：

- 收起态：两行 logo + 版本 + 紧凑快捷键提示 + “Press Ctrl+O to show full startup help…” + onboarding；
- 展开态：完整快捷键列表；
- `Ctrl+O` 切换展开；
- 点击 logo 前两行的 1–4 列触发 3D logo 彩蛋；
- 不支持 logo 的终端退化为 `Pi vX` 文本。

完整提示包括：

- interrupt / clear / 两次 clear 退出 / exit / suspend；
- delete to end；
- cycle thinking level；
- cycle model；
- model select；
- expand tools / thinking；
- external editor；
- `/` commands、`!` bash、`!!` bash without context；
- follow-up / dequeue；
- paste files/images/text；
- drop files。

### 2.2 loaded resources

`loadedResourcesContainer` 在 header 和 chat 之间，展示：

- Context（AGENTS.md 等）；
- Skills；
- Prompts；
- Extensions；
- 诊断信息。

它支持 compact/expanded 两种形态；`showLoadedResources()` 会根据 `quietStartup`、是否有诊断、是否首次启动决定是否显示。这里不是核心视觉，但会影响首屏高度和“启动时看到什么”。

### 2.3 live resources 变化

`/reload` 后：

- 重新加载 keybindings/extensions/skills/prompts/themes/context；
- `rebuildChatFromMessages()` 重建聊天；
- `showLoadedResources()` 重新显示资源；
- `showStatus()` 提示 reload 结果。

---

## 3. 消息组件与间距

### 3.1 用户消息

`UserMessageComponent extends Container`：

- 只有 `Markdown` 一个子组件；
- `Markdown` 参数：`paddingX = outputPad`（默认 1）、`paddingY = 1`；
- `defaultTextStyle`：`color=userMessageText`、`bgColor=userMessageBg`；
- `preserveOrderedListMarkers: true`、`preserveBackslashEscapes: true`；
- 首行前插 OSC 133 `A`，末行前插 `B`/`C`，供全屏 prompt 跳转/语义标记；
- 因为 Markdown 自己补满整行背景，所以不需要外层 Box，否则会多一份全宽副本。

在 chat 中，如果已有内容，用户消息前加 `Spacer(1)`。这形成“每条用户消息之间一个空行”的节奏。

### 3.2 助手消息

`AssistantMessageComponent extends Container`：

- 内容容器 `contentContainer`；
- `updateContent(message, isStreaming)` 每次清空重建；
- 如果存在可见 text/thinking，先加 `Spacer(1)`；
- 按 content 顺序：
  - `text`：`Markdown(text.trim(), outputPad, 0, ...)`；
  - 连续的 `thinking` 合并成一个块：
    - 隐藏时：`Text(italic(thinkingText(hiddenThinkingLabel)))`，默认 “Thinking...”；点击可切换该块显隐；
    - 显示时：`Markdown(thinking, outputPad, 0, ..., color=thinkingText, italic=true)`；
    - 如果后面还有可见内容，加 `Spacer(1)`；
- `stopReason === "length"`：加空行 + “Response was truncated before completion.”；
- 没有 toolCall 且 `aborted`/`error`：加空行 + 错误文本；
- 如果 `hasToolCalls`，不插入 OSC 133 区域标记（因为工具卡是独立组件，区域会被切断）。

助手文本的 `paddingY=0`，避免工具卡前出现双空行；工具卡自己带 `Spacer(1)`。

### 3.3 工具卡

`ToolExecutionComponent extends Container`：

```text
Spacer(1)
Box(paddingX=outputPad, paddingY=1, bg=stateBg)
  ├─ MouseRegion(callRenderer)
  └─ MouseRegion(resultRenderer)
```

状态背景：

- `isPartial` → `toolPendingBg`；
- 结果错误 → `toolErrorBg`；
- 成功 → `toolSuccessBg`。

`renderShell`：

- `"default"`：外层 Box 提供 padding+背景；
- `"self"`：工具自己画框/背景，外层只保留 spacer 和图片。

工具卡内容：

- `renderCall(args, theme, context)` 生成调用头；
- `renderResult(result, options, theme, context)` 生成结果；
- 没有定义时用 fallback：
  - 调用：`formatToolCallWithArgs()`；
  - 结果：文本输出，收起时只显示前 10 行，追加 `... (N more lines, Ctrl+O to expand)`；
- 点击结果区域切换展开；
- 图片结果：内容后加 `Spacer(1)` + `Image`，默认 60 cells 宽；
- `hideComponent`：如果既没有内容也没有图片，整个组件隐藏。

内建工具的具体视觉：

- `bash`：`$ <command>`（powershell 是 `PS>`），输出默认只显示最后 5 个视觉行，展开显示全部；有 truncation/full output 警告；计时文本。
- `read`：`read <path>` + 行范围；对 AGENTS.md/SKILL.md/文档有 compact 形式（`read docs path`、`[skill] name`）；结果默认收起时为空，展开后显示高亮代码，最多 10 行 + “more lines”。
- `edit`：`edit <path>`；参数完整后异步计算 diff，预览阶段就用 success/error 背景；结果 diff 用行级颜色 + 单行修改的 word-level 反色高亮。
- `write`：`write <path>` + 内容预览；流式时增量高亮前 50 行，避免每次全量重排；默认 10 行。
- `grep`：`grep /pattern/ in path (glob) limit N`；结果默认 15 行。
- `find`：`find pattern in path (limit N)`；结果默认 20 行。
- `ls`：`ls path (limit N)`；结果默认 20 行。

### 3.4 bash 执行卡

`BashExecutionComponent` 是用户用 `!` 触发的 bash，不是模型工具：

```text
Spacer(1)
DynamicBorder (bashMode/dim)
  $ command
  output（默认最后 20 个逻辑行）
  loader / status
DynamicBorder
```

- `!!` 命令用 `dim` 颜色，表示输出不进入模型上下文；
- 运行中显示 `Running... (Esc to cancel)`；
- 完成后显示 `(cancelled)` / `(exit N)`；
- 输出被截断时显示 `Output truncated. Full output: path`；
- 收起时显示 `... N more lines (Ctrl+O to expand)`。

### 3.5 其他消息

- `compactionSummary`：前面 `Spacer(1)`，`CompactionSummaryMessageComponent`，可展开；
- `branchSummary`：同理；
- `custom`：扩展消息，`CustomMessageComponent`，可展开；
- `custom entry`：扩展 entry renderer；
- `system`：不显示；
- `toolResult`：不单独显示，和工具调用合并；
- `skill invocation`：技能块 + 可选用户消息；
- `cache warming`：dim 文本；
- `compaction cost`：成本提示；
- 错误/警告/状态：`Spacer(1)` + 对应颜色的 `ThemedText`，连续状态会复用上一条状态行，避免刷屏。

---

## 4. 流式事件与顺序

### 4.1 `message_start`

- `custom`：直接加消息卡；
- `user`：加用户消息卡，更新 pending 显示；
- `assistant`：
  - 创建 `AssistantMessageComponent`；
  - 记录 `streamingComponent`/`streamingMessage`；
  - 加到 `chatContainer`；
  - `updateContent(message, true)`；
  - requestRender。

### 4.2 `message_update`

- 更新 `streamingMessage`；
- `streamingComponent.updateContent(message, true)`；
- 遍历 content 中的 `toolCall`：
  - 如果没有对应 `pendingTools`，创建 `ToolExecutionComponent`，加到 `chatContainer`，记录到 map；
  - 如果已有，`updateArgs()`；
- requestRender。

这意味着工具卡在模型还没结束、参数还在流式生成时就出现了；用户能看到调用头随着参数变化。

### 4.3 `tool_execution_start`

- 如果 `parentToolCallId` 存在，跳过（嵌套工具显示在父卡里）；
- 找 `pendingTools`，没有就创建；
- `markExecutionStarted()`；
- requestRender。

### 4.4 `tool_execution_update`

- 找到组件，`updateResult(partial, true)`；
- requestRender。

### 4.5 `tool_execution_end`

- 找组件，`updateResult(final, duration)`；
- 从 `pendingTools` 删除；
- requestRender。

### 4.6 `message_end`

- 用户消息直接 break；
- assistant：
  - 更新 final content；
  - 如果 aborted/error：给所有 pendingTools 填错误结果，清空 map；
  - 否则：`setArgsComplete()` 触发 edit diff 计算、thinking drop/cache miss 提示；
  - 清空 streaming 状态；
  - footer invalidate；
- requestRender。

### 4.7 `agent_end`

- 关闭终端进度；
- 清除 working 状态；
- 如果 streamingComponent 还在，移除；
- 清空 pendingTools；
- requestRender。

### 4.8 历史渲染

`renderSessionEntries(entries)`：

- 把 session entries 转成消息；
- assistant 消息：先加 `AssistantMessageComponent`，再按 content 顺序为每个 `toolCall` 创建 `ToolExecutionComponent`；
- `toolResult`：找到对应 pending 组件，`updateResult()`；
- 其他消息走 `addMessageToChat()`；
- 最后把仍未匹配的 tool 组件放回 `pendingTools`。

这保证了恢复会话时工具卡和结果的顺序与实时流一致。

---

## 5. 底部区域

### 5.1 pending 消息

`updatePendingMessagesDisplay()`：

- 清空 `pendingMessagesContainer`；
- 取 `steering` 和 `followUp` 队列（包括 compaction 队列）；
- 如果有内容：
  ```text
  Spacer(1)
  Steering: <text>
  Follow-up: <text>
  ↳ <key> to edit all queued messages
  ```
- 每条用 `TruncatedText`，左右 padding 1。

fullscreen 下 pending 在 editor 上方；regular 下也在 editor 上方，但因为整篇文档顺序渲染，它出现在聊天末尾。

### 5.2 状态指示器

`StatusIndicator extends Loader`，kind：

- `working`：`Working` 或扩展设置的文案；颜色默认 accent/muted；如果编辑器支持嵌入，则显示在上边框；
- `retry`：`Retrying (attempt/max) in Ns... (Esc to cancel)`，带倒计时；
- `compaction`：`Compacting context...` / `Context overflow detected, Auto-compacting...`；
- `branchSummary`：`Summarizing branch...`。

`showStatusIndicator()`：

- dispose 旧 indicator；
- 清空 `statusContainer`；
- 如果当前 editor 是 `WorkingStatusEditor` 且 `embedWorkingStatus`，把 indicator 交给 editor，显示在 top border；
- 否则加到 `statusContainer`。

`clearStatusIndicator()`：

- 如果被清除的是非嵌入状态、regular 模式、`clearOnShrink` 开启，往 `statusContainer` 加 `IdleStatus`（两行空行），防止内容缩短后留下残影。

### 5.3 footer

`FooterComponent.render(width)` 输出 2–3 行：

第一行：

```text
<cwd> (<git branch>) • <session name>
```

- cwd 把 home 替换成 `~`；
- branch 来自 `FooterDataProvider` 的异步 git 查询；
- session name 可选。

第二行：

```text
↑input ↓output RcacheRead WcacheWrite CHhit% $cost <context%>/<window> (auto)     <model> • <thinking>
```

- `formatTokens`：<1000 原样；<10k 一位小数 k；<1M 整数 k；<10M 一位小数 M；否则整数 M。
- 只有非零项才显示。
- 有 cache read/write 时显示 `CH` 命中率。
- 订阅制模型显示 `$0.000 (sub)`。
- context >90% 用 error 色，>70% 用 warning 色。
- `auto` 表示自动压缩开启。
- 右侧模型：`model.id`；如果模型支持 reasoning，追加 `• thinking off/<level>`；如果有 routed model，追加 `→ <routed model> • <level>`。
- 如果可用 provider >1 且空间足够，右侧加 `(<provider>)` 前缀；放不下就退回不带 provider。
- 左右按宽度分配：放得下就右对齐；放不下先截右侧；再放不下只留左侧。
- 第三行：扩展状态，按 key 字母序拼接，单行 dim。

footer 对 `statsLeft` 和 `remainder` 分别套 dim，因为 statsLeft 里可能包含已经带 color reset 的 context 百分比；如果整体套一层 dim，reset 会把 dim 清掉。

### 5.4 editor

`editorContainer` 默认只放 `defaultEditor`。选择器打开时，`showSelector()` 会：

```text
editorContainer.clear()
editorContainer.addChild(selector)
ui.setFocus(selector)
```

关闭时恢复 `editor` 并聚焦。也就是说，模型选择器、设置、thinking、trust、session、tree 等**替换输入框**，不是盖在聊天上。这样它们的高度、滚动和 footer 位置都由 dock 布局管理。

`CustomEditor` 的视觉：

- 上/下边框 `─`，颜色 `borderMuted`；thinking 时按 level 用 `thinking*`；bash 模式用 `bashMode`；
- 上边框可嵌入 working spinner；
- 编辑器内容行没有左右竖线；
- 自动补全列表追加在下边框之后；
- `Ctrl+O` 展开工具输出也会影响 header 的展开态。

---

## 6. 选择器与 Overlay

### 6.1 替换 editor 的选择器

- `ModelSelectorComponent`：上/下边框、spacer、scope 提示、搜索 `Input`、列表、提示、下边框；列表项显示 provider/id/name；支持 fuzzy 搜索、scope 切换、设为默认。
- `SettingsSelectorComponent`：`SettingsList`，每项 label/currentValue/description/values；子菜单用于 warnings、model thinking levels 等。
- `ThinkingSelectorComponent`：标题、循环提示、搜索 Input、SelectList、确认/设为默认提示。
- `SessionSelectorComponent`、`TreeSelectorComponent`、`TrustSelectorComponent`、`OAuthSelectorComponent` 等：同样替换 editor，按各自布局渲染。
- `ExtensionSelectorComponent`、`ExtensionInputComponent`、`ExtensionEditorComponent`：扩展 UI；可以通过 `showExtensionCustom()` 选择 `overlay: true` 走 Overlay，否则也替换 editor。

这些组件的共同点：

- 都有 `Focusable`，把 `focused` 传给内部 `Input`，保证 IME 光标；
- 都用 `DynamicBorder` 画上下边框；
- 都用 `Spacer(1)` 分隔；
- 都用 `keyHint` 在底部显示操作提示；
- 关闭时 `done()` 恢复 editor 和焦点。

### 6.2 Overlay 的使用

真正使用 `showOverlay()` 的场景较少：

- 扩展自定义 UI 选择 `overlay: true`；
- 全屏搜索面板（`AltScreenSearchComponent`，右上角 40% 宽，min 32）；
- 一些对话框/通知。

Overlay 的尺寸默认 `min(80, 可用宽度)`，所以默认最大 80 列；搜索面板显式 `width: "40%"`、`minWidth: 32`、`margin: 1`。

### 6.3 全屏搜索

`ctrl+shift+f` 打开搜索 Overlay；输入查询后：

- 在 primary ScrollView 的完整内容行里找匹配；
- 匹配高亮通过 `searchMatchStyle`/`searchCurrentMatchStyle`；
- 当前匹配滚动到视口；
- `Enter`/`Ctrl+G` 下一个，`Shift+Enter`/`Ctrl+Shift+G` 上一个；
- `Esc` 关闭。

搜索高亮是在布局后的屏幕行上按列切片再套样式，必须处理 ANSI 和宽字符，不能直接 `string.replace()`。

---

## 7. 补全提供者的组装

`createBaseAutocompleteProvider()` 把四类命令合并：

1. `BUILTIN_SLASH_COMMANDS`；
2. prompt templates（`/name`）；
3. 扩展注册的命令；
4. 技能命令 `/skill:<name>`（如果启用）。

`/model`、`/thinking`、`/login` 有参数补全：

- `/model`：scoped models 或全部可用模型，fuzzy 匹配 `provider/id/name`；
- `/thinking`：可用 thinking levels；
- `/login`：provider 列表，合并 OAuth/API key 两种认证类型。

`setupAutocompleteProvider()` 还支持扩展 wrapper：

```text
base provider
  → wrapper1
  → wrapper2
  → ...
```

每个 wrapper 可以增加 `triggerCharacters`，最终去重后设置到 provider 和 editor。

---

## 8. 会话恢复与重建

`renderInitialMessages()`：

- `sessionManager.buildContextEntries()` 得到当前分支的上下文条目；
- `renderSessionEntries(entries, { updateFooter: true, populateHistory: true })`；
- `populateHistory` 把用户消息加入编辑器历史；
- 如果项目未信任，追加警告；
- 如果会话被压缩过，显示 “Session compacted N time(s)”。

`/reload` / 扩展重绑：

- `rebuildChatFromMessages()` 清空 `chatContainer`，重新 `renderSessionEntries()`；
- 清空 `pendingTools`；
- 重置 alt screen 的文本选择（因为旧选择坐标指向被替换的 transcript）。

fullscreen 退出时：

- 默认 `fullscreenExitOutput: "transcript"`；
- `stopInteractiveTui()` 如果当前是全屏且输出 transcript：
  - 关闭所有 Overlay；
  - `switchTuiMode("regular", false, false)`，把组件树迁移到主屏渲染器；
  - `renderNow()` 打印完整 transcript；
  - 再 `stop()`。
- 这样用户退出后能在 shell scrollback 里看到对话，而不是空白。

---

## 9. 视觉常量速查

| 项 | 值 |
|---|---|
| 普通渲染节流 | 16ms |
| 输入渲染 | 立即抢占 |
| 编辑器最大可视高度 | `max(5, floor(terminalRows*0.3))` |
| 大粘贴阈值 | >10 行或 >1000 字符 |
| 补全最大可见项 | 5（范围 3–20） |
| 工具 fallback 预览 | 10 行 |
| bash 工具渲染预览 | 5 个视觉行（keep end） |
| bash 执行卡预览 | 最后 20 个逻辑行 |
| grep 结果预览 | 15 行 |
| find/ls 结果预览 | 20 行 |
| read/write 结果预览 | 10 行 |
| 图片默认宽度 | 60 cells |
| 图片最大宽度 | `min(width-2, setting)` |
| Loader 帧间隔 | 80ms |
| 全屏滚动条隐藏延迟 | 1000ms |
| Overlay 默认最大宽度 | 80 |
| 全屏搜索面板 | 40% 宽，min 32，margin 1 |
| 状态行去抖 | 连续状态复用上一行 |
| 双 Esc 间隔 | 500ms |
| Ctrl+C 两次退出间隔 | 500ms |
| 扩展 widget 最大行数 | 10 |

---

## 10. 与 Java 复刻直接相关的差异点

1. **组件树必须存在。** JadeCode 当前 `ConversationView` 直接拼 `StyledLine`，没有 `Component`/`Container`/布局树；要复刻 fullscreen 的固定 dock、Overlay、ScrollView，必须先补组件树。
2. **选择器替换 editor，不是弹窗。** 这决定了 dock 高度和 footer 位置；如果做成 Overlay，视觉上会是“浮在聊天上”，与 Pi 不一致。
3. **工具卡是“调用头 + 结果”两个 renderer，不是一段固定文本。** 每个工具的背景、预览行数、展开行为、diff 高亮都不同。
4. **流式工具卡在参数生成时就出现。** 不是等 `tool_execution_start` 才创建；否则用户会看到“模型已经说完，工具卡才突然出现”。
5. **footer 是两层右对齐。** 左边统计可能带颜色，右边模型可能被截断；不能简单拼一行再 truncate。
6. **状态 spinner 在 editor 上边框里。** 不是单独一行；这是 Pi 的辨识度之一。
7. **退出全屏要重放 transcript。** 否则用户会丢失对话。
8. **主题默认是 system。** 只做 dark/light 会在默认配置下就不一致。

---

## 11. 证据分级

- **已证实**：组件树、初始化顺序、消息/工具卡/状态/footer/选择器的结构与行为、流式事件顺序、退出重放、视觉常量，均来自源码。
- **合理推断**：“必须先有组件树才能 1:1 复刻 fullscreen”是根据现有 JadeCode 架构与 Pi 布局要求推断的；没有做 Java 实现验证。
- **无法确认**：真实终端下 header/logo、图片、字体对视觉效果的影响；不同用户设置（quietStartup、hideThinkingBlock、toolOutputExpanded）导致的默认首屏差异。

---

## 12. 源码依据

| 相对路径 | 关键内容 |
|---|---|
| `packages/coding-agent/src/modes/interactive/interactive-mode.ts` | 构造 608；`init` 943；挂载 873；模式切换 890；`handleEvent` 3408；`addMessageToChat` 3882；`renderSessionItems` 4005；`renderSessionEntries` 4105；`showStatusIndicator` 2312；`updatePendingMessagesDisplay` 4737；`showSelector` 4898；`showModelSelector` 5360 |
| `packages/coding-agent/src/modes/interactive/tui-renderer.ts` | 两种渲染器 22-50 |
| `packages/coding-agent/src/modes/interactive/chat-viewport.ts` | fullscreen transcript + dock 20-53 |
| `packages/coding-agent/src/modes/interactive/components/user-message.ts` | 用户消息背景/OSC133 19-70 |
| `packages/coding-agent/src/modes/interactive/components/assistant-message.ts` | 助手消息/thinking/错误 91-200 |
| `packages/coding-agent/src/modes/interactive/components/tool-execution.ts` | 工具卡 65-410 |
| `packages/coding-agent/src/modes/interactive/components/bash-execution.ts` | bash 执行卡 35-260 |
| `packages/coding-agent/src/modes/interactive/components/status-indicator.ts` | 状态 indicator 12-125 |
| `packages/coding-agent/src/modes/interactive/components/footer.ts` | footer 157-300 |
| `packages/coding-agent/src/modes/interactive/components/custom-editor.ts` | 上边框状态 39-97 |
| `packages/coding-agent/src/modes/interactive/components/model-selector.ts` | 模型选择器 |
| `packages/coding-agent/src/modes/interactive/components/settings-selector.ts` | 设置选择器 |
| `packages/coding-agent/src/modes/interactive/components/thinking-selector.ts` | thinking 选择器 |
| `packages/coding-agent/src/modes/interactive/components/dynamic-border.ts` | 动态横线 |
| `packages/coding-agent/src/core/tools/renderers/*.ts` | 各工具调用/结果视觉 |
| `packages/coding-agent/src/core/settings-defaults.ts` | 视觉默认值 27-64 |

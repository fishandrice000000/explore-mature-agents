# TUI - Pi - 05 - 输入、键绑定、编辑器与补全

---

1. C
2. 主要是视觉布局与交互.
3. JadeCode 基于 Java, 优先 Ubuntu.
4. 需要移植对照.

你可以顺带研究一下 JadeCode 的源码: /home/fishandrice/Backend/JadeCode/src/main/java/com/jadecode

---

## 0. 输入层的五段管线

```text
终端字节
  → ProcessTerminal（raw mode / bracketed paste / Kitty 协商）
  → StdinBuffer（拆包、粘贴、超时）
  → TUI.handleTerminalInput（终端回复、全局监听器、焦点）
  → CustomEditor.handleInput（应用级快捷键优先）
  → Editor.handleInput（编辑/补全/历史）
```

任何一段顺序错了，交互都会出现“看起来像小问题、实际很难修”的偏差，例如：

- 先匹配编辑器键再匹配应用键，`Ctrl+O` 会被编辑器吃掉；
- 粘贴内容被拆成单字符，里面的 `Enter` 触发提交；
- `Esc` 等待时间太短，SSH 下方向键被拆成 Esc + `[A`；
- 过滤了 key release，但编辑器依赖 release 做按键抬起行为。

---

## 1. 键解析与匹配

### 1.1 两套编码

Pi 同时支持：

1. **传统终端序列**：`\x1b[A`、`\x7f`、`\x01` 等；
2. **Kitty keyboard protocol**：`CSI <codepoint> ; <modifier> : <event> u`，以及带 alternate key 的变体。

`ProcessTerminal` 启动时协商；`keys.ts` 的 `parseKittySequence()` 负责解析：

```text
\x1b[<codepoint>u
\x1b[<codepoint>;<mod>u
\x1b[<codepoint>;<mod>:<event>u
\x1b[<codepoint>:<shifted>;<mod>u
\x1b[<codepoint>:<shifted>:<base>;<mod>u
\x1b[<codepoint>::<base>;<mod>u
```

修饰键值是 1-based 的位掩码加 1：`modifier = value - 1`，再按位判断 shift/alt/ctrl/super。

还支持：

- 方向键带修饰：`CSI 1;<mod>A/B/C/D`；
- 功能键：`CSI <num>;<mod>~`（Insert/Delete/PageUp/PageDown/Home/End）；
- Home/End：`CSI 1;<mod>H/F`；
- xterm `modifyOtherKeys`：`CSI 27;<mod>;<keycode>~`。

### 1.2 `matchesKey(data, keyId)`

`keyId` 是一个字符串，例如 `"ctrl+c"`、`"shift+enter"`、`"alt+left"`、`"shift+ctrl+p"`。

`parseKeyId()` 会按修饰符前缀逐个剥离，支持任意顺序。然后 `matchesKey()`：

1. 先处理特殊键（escape、space、enter、tab、backspace、delete、方向键、Home/End 等）。
2. `escape`：只接受裸 `\x1b`、Kitty escape、modifyOtherKeys escape；带修饰符直接 false。
3. `space`：非 Kitty 模式下，`\x00` = Ctrl+Space，`\x1b ` = Alt+Space。
4. 字母/符号：优先精确匹配 codepoint + modifier。
5. **逻辑 shifted symbol**：Kitty 可能把 `+` 报告成 `Shift + =` 的物理键。`matchesKey` 会把这种“产生符号的隐式 Shift”归一化，所以 `shift+=` 和 `+` 都能匹配。
6. **base layout key**：非拉丁键盘布局下，Kitty 会报告 PC-101 基础布局键。`Ctrl+С`（西里尔）可以匹配 `Ctrl+c`。但只有当前 codepoint 不是已知拉丁字母/符号时才回退，避免 Dvorak/Colemak 重映射误匹配。
7. `isKeyRelease()`：检查 `:3u`、`:3~`、`:3A` 等；带 `\x1b[200~` 的粘贴内容不当作 release。
8. `isKeyRepeat()`：检查 `:2u` 等。

### 1.3 `parseKey()` 与可打印字符

`parseKey(data)` 返回人类可读的键名，`decodePrintableKey(data)` 返回按键产生的可打印字符。编辑器在最后一步用它处理 Kitty 的 CSI-u 可打印键，避免把 `\x1b[97u` 当控制序列丢掉。

### 1.4 Backspace 的平台差异

- `\x7f`：普通 Backspace。
- `\x08`：在 Windows Terminal 里可能是 Ctrl+Backspace；在旧终端/tmux 里可能是普通 Backspace。`matchesRawBackspace()` 用 `WT_SESSION` 且不在 SSH 下作为启发式，决定 `\x08` 是 Ctrl+Backspace 还是 Backspace。

---

## 2. 键绑定

### 2.1 键绑定注册表

`KeybindingsManager` 维护：

- `definitions`：每个动作的默认键；
- `userBindings`：用户覆盖；
- `keysById`：动作 → 解析后的键列表；
- `conflicts`：用户同时把同一个键绑到多个动作时的冲突列表。

`matches(data, action)` 遍历该动作的所有键，任意一个 `matchesKey` 命中即真。默认绑定集中在 `TUI_KEYBINDINGS` 和 coding-agent 的 `APP_KEYBINDINGS`。

### 2.2 TUI 默认键（编辑器/选择器/全屏）

| 动作 | 默认键 |
|---|---|
| 光标上/下 | `up` / `down` |
| 光标左/右 | `left`、`ctrl+b` / `right`、`ctrl+f` |
| 词左/右 | `alt+left`、`ctrl+left`、`alt+b` / `alt+right`、`ctrl+right`、`alt+f` |
| 行首/行尾 | `home`、`ctrl+a` / `end`、`ctrl+e` |
| 删除前/后 | `backspace` / `delete`、`ctrl+d` |
| 删词前/后 | `ctrl+w`、`alt+backspace` / `alt+d`、`alt+delete` |
| 删到行首/行尾 | `ctrl+u` / `ctrl+k` |
| yank / yank-pop | `ctrl+y` / `alt+y` |
| undo | `ctrl+-`（Windows 为 `ctrl+z`，WSL 为 `alt+z`） |
| 换行 | `shift+enter`、`ctrl+j` |
| 提交 | `enter` |
| Tab/补全 | `tab` |
| 复制 | `ctrl+c` |
| 选择器上/下/确认/取消 | `up`/`down`/`enter`/`escape`、`ctrl+c` |
| 全屏翻页 | `pageUp`/`pageDown` |
| 全屏半页/单行 | 默认空，可配置 |
| 上/下一条 prompt | `ctrl+shift+up`、`ctrl+up` / `ctrl+shift+down`、`ctrl+down` |
| 全屏搜索 | `ctrl+shift+f`（Windows 为 `ctrl+f`） |
| 搜索下一个/上一个 | `enter`、`ctrl+g` / `shift+enter`、`ctrl+shift+g` |
| 全屏顶部/底部 | `ctrl+home` / `ctrl+end` |

### 2.3 应用级默认键

| 动作 | 默认键 |
|---|---|
| interrupt / 取消 | `escape` |
| clear | `ctrl+c` |
| exit（编辑器为空） | `ctrl+d` |
| suspend | `ctrl+z`（Windows 空） |
| 循环 thinking level | `shift+tab` |
| 保存 thinking level | `ctrl+s` |
| 循环模型 | `ctrl+p` / `shift+ctrl+p`（Windows `alt+p`） |
| 模型选择器 | `ctrl+l` |
| 展开/收起工具输出 | `ctrl+o` |
| 展开/收起 thinking | `ctrl+t` |
| 外部编辑器 | `ctrl+g` |
| 复制选区/最后回复 | `ctrl+x` |
| 排队 follow-up | `alt+enter`（Windows `ctrl+q`） |
| 取回队列 | `alt+up`（Windows `alt+q`） |
| 粘贴图片/文件/文本 | `ctrl+v`（Windows `alt+v`） |

### 2.4 优先级

`CustomEditor.handleInput()` 的顺序：

1. 扩展注册的快捷键（`onExtensionShortcut`）；
2. 剪贴板粘贴键（`app.clipboard.pasteImage`）；
3. `app.interrupt`：只有自动补全未显示时才执行应用 handler；否则把 Esc 交给编辑器取消补全；
4. `app.exit`：只有编辑器为空时才退出；否则继续走编辑器的“删除后一个字符”；
5. 显式绑定的历史键（`tui.editor.historyPrevious/Next`）：优先给编辑器，避免 Ctrl+P 既循环模型又浏览历史；
6. 其他应用动作；
7. 最后交给 `Editor.handleInput()`。

这个顺序是复刻时最容易做错的：如果先把所有输入给编辑器，`Ctrl+O` 会被当成普通控制字符；如果先匹配应用动作，`Ctrl+P` 会被模型循环抢走，用户无法配置历史键。

---

## 3. `Editor` 的状态与视觉

### 3.1 状态

```ts
state = {
  lines: string[],     // 逻辑行，不含换行
  cursorLine: number,
  cursorCol: number,   // UTF-16 index within line
}
```

辅助状态：

- `lastWidth`：上次渲染的内容宽度，用于鼠标命中和垂直移动；
- `scrollOffset`：垂直滚动偏移（按视觉行）；
- `preferredVisualCol`：上下移动的“粘性列”；
- `snappedFromCursorCol`：光标被吸附到原子片段（如粘贴标记）前的原始列；
- `pastes: Map<number,string>`：大粘贴的原文；
- `history` / `historyIndex` / `historyDraft`；
- `undoStack`；
- `killRing`；
- `jumpMode`；
- `autocomplete*`。

### 3.2 渲染

```text
render(width)
  → paddingX 夹取
  → contentWidth = width - paddingX*2
  → layoutWidth = contentWidth - (paddingX ? 0 : 1)   // 无 padding 时给光标留 1 列
  → layoutText(layoutWidth)   // 逻辑行 → 视觉行
  → maxVisibleLines = max(5, floor(terminalRows * 0.3))
  → 调整 scrollOffset 让光标可见
  → 取 visibleLines
  → 上边框（有隐藏行时显示 ↑ N more）
  → 每一行：左侧 padding + 文本 + 右侧 padding
      光标所在行：
        - 有字符：把该 grapheme 包进假光标
        - 行尾：假光标包一个空格
        - focused：在假光标前插 CURSOR_MARKER
  → 下边框（有隐藏行时显示 ↓ N more）
  → 如果有补全列表：在 contentWidth 内渲染，追加在下面
```

视觉特征：

- 编辑器没有左右竖边框，只有上下两条 `─`。
- 默认 `paddingX=0`，所以内容从第 0 列开始。
- 默认最大可视高度是终端高度的 30%，至少 5 行。
- 上/下边框在内容被滚动隐藏时，会把 `↑ N more` / `↓ N more` 嵌进横线中间。
- 光标不是自己画一个块，而是用 `\x1b[7m` 反色包住当前字符；如果终端显示硬件光标，则反色假光标会被去掉，由终端画。
- `CURSOR_MARKER` 必须在假光标之前，TUI 才能把硬件光标放在正确位置，IME 候选窗才会跟着走。

### 3.3 折行 `wordWrapLine`

编辑器不能直接用 `wrapTextWithAnsi`，因为它需要知道每个视觉片段对应逻辑行的哪个字符区间，才能把光标放到正确片段。

```text
wordWrapLine(line, maxWidth, preSegmented?)
  - 如果整行放得下，返回一个 chunk
  - 按 grapheme 遍历：
      currentWidth + gWidth > maxWidth 时：
        如果有 wrap opportunity，且回退后剩余内容 + 当前字素仍放得下：
          在 opportunity 处切
        否则：
          在当前位置强制切
      gWidth > maxWidth：
        递归按 grapheme 再切（例如极窄窗口下的粘贴标记）
      记录 wrap opportunity：
        - 空白后接非空白
        - 相邻两个字符至少一个是 CJK
  - 最后 push 剩余部分
```

CJK 允许在任意相邻字符之间断行；英文尽量在空格处断；超长单词强制按 grapheme 断。粘贴标记是原子片段，不能从中间切开，除非它本身比窗口还宽。

### 3.4 垂直移动与粘性列

`buildVisualLineMap(width)` 把每个逻辑行拆成视觉行：

```ts
{ logicalLine, startCol, length }
```

上下移动时：

1. 找到当前视觉行；
2. 计算当前视觉列（`cursorCol - startCol`）；
3. 目标视觉行可能更短；
4. `computeVerticalMoveColumn()` 根据“是否有 preferred column、光标是否在行中、目标是否太短”决定落点；
5. 设置新 `cursorLine/cursorCol`；
6. 如果落到粘贴标记等多 grapheme 原子片段内部，吸附到片段开头，并记录 `snappedFromCursorCol`。

`computeVerticalMoveColumn` 的决策表在源码注释里写得很清楚：

| 有 preferred | 光标在行中 | 目标比当前短 | 目标比 preferred 短 | 行为 |
|---|---|---|---|---|
| 0 | * | 0 | - | 清除 preferred，保持当前列 |
| 0 | * | 1 | - | 设置 preferred=当前列，落到目标行尾 |
| 1 | 0 | 0 | 0 | 清除 preferred，落到 preferred |
| 1 | 0 | 0 | 1 | 保留 preferred，落到目标行尾 |
| 1 | 0 | 1 | - | 保留 preferred，落到目标行尾 |
| 1 | 1 | 0 | - | 清除 preferred，保持当前列 |
| 1 | 1 | 1 | - | 设置 preferred=当前列，落到目标行尾 |

这套行为让用户在长短行之间上下移动时不会“越移越靠左”。

### 3.5 历史

- `addToHistory(text)`：trim 后去重（只和最近一条比），最多 100 条。
- 上箭头：如果当前在第一个视觉行，且（编辑器为空 或 已在历史浏览 或 光标在列 0）→ 上一条历史；否则如果是第一视觉行 → 行首；否则上移一行。
- 下箭头：如果在最后一个视觉行且正在浏览历史 → 下一条；否则最后一行 → 行尾；否则下移。
- 第一次进入历史时保存当前草稿 `historyDraft`，回到 `-1` 时恢复。
- 历史是内存态，不持久化；提交后重置。

### 3.6 撤销

`UndoStack` 存的是深拷贝快照（编辑器还额外保存 paste registry 和 counter）。

- 输入字符时：如果是空白，或上一次动作不是 `type-word`，push 快照；连续单词字符合并成一个撤销单元。
- 删除/换行/粘贴/补全/清空都 push 快照。
- 撤销只恢复文本和光标，不恢复 autocomplete UI。
- 提交后清空 undo stack。

### 3.7 Kill ring / yank

`KillRing` 是一个字符串数组：

- 删除到行首/行尾/删词时，把被删文本 push；
- 连续 kill 会 accumulate：向后删 prepend，向前删 append；
- `yank` 插入最近一条；
- `yank-pop` 删除刚插入的文本，rotate 到上一条，再插入。

### 3.8 字符跳转

`ctrl+]` 进入 forward jump mode，`ctrl+alt+]` 进入 backward；下一次可打印字符触发 `jumpToChar()`，在**当前逻辑行**内查找目标字符并移动光标。再按一次跳转键取消。

### 3.9 换行、提交与反斜杠

换行条件：

- `tui.input.newLine`（`shift+enter` 或 `ctrl+j`）；
- `data.charCodeAt(0) === 10 && data.length > 1`；
- `\x1b\r`；
- `\x1b[13;2~`；
- 任何包含 ESC 和 CR 的多字符序列；
- 单独的 `\n`。

提交：

- `tui.input.submit` 默认 `enter`；
- 如果 `disableSubmit` 为真，忽略；
- 如果光标前是 `\`，删除它并换行（给不支持 Shift+Enter 的终端兜底）；
- 否则 `submitValue()`：展开粘贴标记、trim、清空状态、清空 undo、调用 `onSubmit`。

还有一个“反斜杠 + Enter”路径：如果配置了 `shift+enter` 绑定，且输入是裸 Enter，且光标前是 `\`，则删掉反斜杠并换行。这是为了兼容只能发裸 Enter 的终端。

### 3.10 粘贴

`Editor.handleInput` 先处理 bracketed paste 标记：

- 收到 `\x1b[200~` 进入 paste 模式；
- 累积到 `\x1b[201~`；
- 把内容交给 `handlePaste()`；
- 粘贴结束后如果还有剩余输入，递归处理。

`handlePaste()`：

1. 取消 autocomplete、退出历史、push undo；
2. 解码 tmux 在 bracketed paste 中重新编码的 Ctrl+字母：`\x1b[<code>;5u` → 对应控制字节；
3. `normalizeText()`：CRLF/CR → LF，Tab → 4 空格；
4. 过滤不可打印字符（保留 `\n`）；
5. 如果粘贴内容以 `/`、`~`、`.` 开头，且光标前是单词字符，自动补一个空格；
6. 如果 **超过 10 行或超过 1000 字符**：
   - `pasteCounter++`；
   - `pastes.set(id, text)`；
   - 插入标记 `[paste #1 +123 lines]` 或 `[paste #1 1234 chars]`；
7. 否则直接插入。

大粘贴的标记在编辑器里是**原子片段**：

- `segmentWithMarkers()` 会把整个 `[paste #n ...]` 合并成一个 grapheme 段；
- 光标移动/删除不会进入标记内部；
- Backspace 删除标记时，同时删除 `pastes` 里的原文，并把后面所有标记的 id 减 1、重写文本中的编号；
- `getText()` 返回带标记的文本；`getExpandedText()` 返回展开后的原文；提交时用展开后的文本。

这套设计解决的是“粘贴 10k 行后编辑器每帧重排全文卡死”的问题，同时保留用户可读的占位符。

---

## 4. 自动补全

### 4.1 Provider 接口

```ts
interface AutocompleteProvider {
  triggerCharacters?: string[];
  getSuggestions(lines, cursorLine, cursorCol, { signal, force }): Promise<{
    items: { value, label, description? }[];
    prefix: string;
  } | null>;
  applyCompletion(lines, cursorLine, cursorCol, item, prefix): {
    lines, cursorLine, cursorCol
  };
  shouldTriggerFileCompletion?(...): boolean;
}
```

`CombinedAutocompleteProvider` 处理四类：

1. **斜杠命令**：光标前是 `/` 开头、且还在命令名部分；fuzzy 匹配命令名，`skill:` 命令会额外按去掉前缀的名字匹配。
2. **命令参数**：`/model <prefix>`、`/thinking <prefix>`、`/login <prefix>` 等，由命令自己的 `getArgumentCompletions()` 提供。
3. **`@` 文件附件**：`@src/foo`、`@"path with space"`；用 `fd` 在 cwd 下搜索，遵守 `.gitignore`，支持 `--full-path`、`--max-depth`。
4. **普通路径**：`src/foo`、`./foo`、`~/foo`、`/abs/foo`；先 `readdirSync`，目录项加 `/`，文件项加空格。

路径 token 的边界：

- 分隔符：空白、Tab、引号、`=`、CJK 标点；
- 允许前面有 `([{<`` 等包裹符；
- 引号内的路径可以包含空格；
- 目录补全后不加空格，方便继续补全；文件补全后加空格。

### 4.2 触发

- 输入 `/` 且在消息开头 → 触发斜杠命令；
- 输入 `@`、`#` 或 provider 的 trigger character，且位于 token 边界 → 触发；
- 输入字母/数字/`.`/`-`/`_` 或 CJK，且处于 `/command ` 上下文或 symbol completion 上下文 → 更新补全；
- Tab：
  - 在斜杠命令上下文且还没输入空格 → 斜杠命令补全；
  - 否则强制文件补全；
- 强制补全时如果只有一个结果，直接应用，不显示列表。

### 4.3 请求去抖与竞态

- 普通 slash 补全立即请求；
- `@` 文件补全有 20ms 去抖（`ATTACHMENT_AUTOCOMPLETE_DEBOUNCE_MS`）；
- 每次请求有 `AbortController`；
- 请求串行化：`autocompleteRequestTask` 保证前一个请求完成后再发下一个；
- 请求返回后检查：
  - `requestId` 是否仍是最新；
  - 文本、cursorLine、cursorCol 是否与请求时快照一致；
  - 否则丢弃结果。

这避免了“用户已经继续打字，旧的 fd 结果回来把列表覆盖”的竞态。

### 4.4 列表 UI

- `SelectList`，最多 `autocompleteMaxVisible`（默认 5，范围 3–20）；
- 斜杠命令使用 `minPrimaryColumnWidth=12, maxPrimaryColumnWidth=32`；
- 每一项：`→ ` / `  ` 前缀、主列、描述列；
- 有滚动时最后一行显示 `(i/n)`；
- 编辑器渲染时，补全列表追加在下边框之后，宽度为 `contentWidth`。

### 4.5 选择与补全行为

- 上/下：`SelectList` 循环移动；
- Tab：应用当前选中项，取消补全；
- Enter：
  - 如果补全的是斜杠命令，应用后**继续走提交流程**（即直接执行命令）；
  - 否则应用并取消补全，不提交；
- Esc：取消补全。
- 最佳选中项：`getBestAutocompleteMatchIndex()` 优先精确匹配，其次第一个前缀匹配。

---

## 5. `Input`（单行输入）

`Input` 是选择器搜索框、扩展输入框等使用的单行输入组件。

视觉与行为：

- 默认 prompt `"> "`；
- 有 placeholder 时，placeholder 的第一个 grapheme 也显示为假光标；
- 内容放得下时直接显示；
- 放不下时水平滚动：
  - 预留 1 列给行尾光标；
  - 光标在左半边 → 从 0 开始；
  - 光标在右半边 → 从 `totalWidth - scrollWidth` 开始；
  - 否则让光标居中；
  - 用 `sliceByColumn` 裁切，保持宽字符完整；
- 光标位置也走 `CURSOR_MARKER` + 假光标；
- 粘贴时去掉换行（单行输入），Tab → 4 空格；
- 支持与 Editor 相同的 kill ring/undo/词移动/Home/End。

---

## 6. `SelectList`

- `items` + `filteredItems`；
- `setFilter()` 用 `startsWith` 过滤（不是 fuzzy；fuzzy 在调用方做）；
- 选中索引 0-based；
- 渲染窗口以选中项为中心：
  ```text
  start = clamp(selected - floor(maxVisible/2), 0, len - maxVisible)
  ```
- 上/下循环；
- 鼠标滚轮移动选中项；左键 press 选中，click 触发 `onSelect`；
- 有描述且 `width > 40` 时显示两列，否则单列；
- 主列宽度默认 32，可配置 min/max；
- 没有匹配项时显示 `  No matching commands`。

---

## 7. 与产品层输入处理的衔接

`InteractiveMode` 在 `CustomEditor` 之上又加了一层：

- `onSubmit`：先处理 `/settings`、`/model`、`/resume`、`/fork`、`/tree`、`/compact` 等内置命令；`!` 开头走 bash；其余交给 `session.prompt()`。
- `onEscape`：
  - 流式中 → 恢复队列并中止；
  - bash 运行中 → 中止 bash；
  - bash 模式 → 清空编辑器、退出 bash 模式；
  - 编辑器为空 → 500ms 内第二次 Esc 触发 `/tree` 或 `/fork`（由设置决定）。
- `onChange`：文本以 `!` 开头时切换 bash 模式边框颜色。
- 流式中提交：调用 `session.prompt(text, { streamingBehavior: "steer" })`，消息进入队列；UI 更新 pending 区域。
- compaction 中提交：扩展命令立即执行，普通消息进入 `compactionQueuedMessages`。

这些行为决定了“按 Enter 后消息是立刻发、排队、还是只显示在 pending 区”。复刻时不能只做 `onSubmit -> prompt`。

---

## 8. Java 复刻要点

1. **不要用 JLine 的 `LineReader`。** Pi 的编辑器是自绘的，依赖 `Editor` 的视觉行映射、粘贴标记、补全列表和光标 marker。JLine 只提供终端读写和 raw mode。
2. **按键解析要自己实现。** JLine 的 `KeyMap` 不认识 Kitty CSI-u 的全部变体；需要按 Pi 的 `keys.ts` 解析、匹配、release/repeat 过滤。
3. **输入解码器要有超时。** 孤立 ESC、拆分 CSI、SSH 延迟、tmux 重新编码，都需要和 `StdinBuffer` 类似的缓冲策略。
4. **编辑器不要按 Java `char` 移动。** 必须按 grapheme；否则 emoji、组合字符、CJK 会被切成两半。
5. **粘贴标记要原子化。** 否则大粘贴会在每次光标移动/删除时重排全文，UI 卡死。
6. **补全请求要可取消、可去重。** 用 `CompletableFuture`/`ExecutorService` 时，必须把“请求快照”与“当前文本”比较，丢弃过期结果。
7. **应用键优先于编辑键，但 Esc 例外。** 补全显示时 Esc 先取消补全；否则 Esc 触发中断/清空逻辑。
8. **双击/三击选择、鼠标拖拽、IME 光标是后续层。** 编辑器只负责光标 marker，屏幕级选择由全屏渲染器做。

---

## 9. 证据分级

- **已证实**：键解析/匹配、默认键表、`CustomEditor` 优先级、Editor 状态/折行/历史/撤销/粘贴/补全、`Input`/`SelectList` 行为，均来自源码。
- **合理推断**：Java 复刻建议基于 Pi 的输入模型和 JLine 的能力边界；没有在 Java 上实测。
- **无法确认**：不同终端/输入法对 Kitty 协议、IME、死键、组合输入的实际行为。源码只能定义协议和回退。

---

## 10. 源码依据

| 相对路径 | 关键内容 |
|---|---|
| `packages/tui/src/keys.ts` | `parseKittySequence` 602；`matchesKittySequence` 668；`matchesKey` 859；`parseKey` 1292；`decodePrintableKey` 1453 |
| `packages/tui/src/keybindings.ts` | `TUI_KEYBINDINGS` 54；`KeybindingsManager` 231 |
| `packages/coding-agent/src/core/keybindings.ts` | `APP_KEYBINDINGS` 25；合并后的 `KEYBINDINGS` 196 |
| `packages/tui/src/stdin-buffer.ts` | `StdinBuffer` 305；paste 分支 350；超时 430 |
| `packages/tui/src/components/editor.ts` | `Editor` 297；`render` 521；`handleInput` 696；`layoutText` 998；`handlePaste` 1260；`handleBackspace` 1380；`buildVisualLineMap` 1836；`computeVerticalMoveColumn` 1581；`requestAutocomplete` 2287 |
| `packages/tui/src/components/input.ts` | `Input` 32；`render` 419；`handlePaste` 403 |
| `packages/tui/src/components/select-list.ts` | `SelectList` 40；`render` 67；`handleMouse` 102；`handleInput` 137 |
| `packages/tui/src/components/editor.ts` | `wordWrapLine` 115；`segmentWithMarkers` 39 |
| `packages/tui/src/word-navigation.ts` | `findWordBackward` 15；`findWordForward` 77 |
| `packages/tui/src/kill-ring.ts` | `KillRing` |
| `packages/tui/src/undo-stack.ts` | `UndoStack` |
| `packages/tui/src/autocomplete.ts` | `CombinedAutocompleteProvider` 303；`getSuggestions` 318；`applyCompletion` 415；`walkDirectoryWithFd` 148 |
| `packages/coding-agent/src/modes/interactive/components/custom-editor.ts` | 应用快捷键优先级 26-148 |
| `packages/coding-agent/src/modes/interactive/interactive-mode.ts` | `setupKeyHandlers` 3069；`setupEditorSubmitHandler` 3201 |

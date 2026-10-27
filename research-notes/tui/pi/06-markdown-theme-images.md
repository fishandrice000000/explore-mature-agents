# TUI - Pi - 06 - Markdown、主题、图片与动效

---

1. C
2. 主要是视觉布局与交互.
3. JadeCode 基于 Java, 优先 Ubuntu.
4. 需要移植对照.

你可以顺带研究一下 JadeCode 的源码: /home/fishandrice/Backend/JadeCode/src/main/java/com/jadecode

---

## 0. 这一层的视觉影响

消息看起来“像 Pi”，主要不是靠布局，而是靠这四件事：

1. **Markdown 的排版细节**：标题、列表缩进、代码围栏、表格边框、引用竖线、行内样式、链接。
2. **主题色板**：默认不是固定 dark/light，而是从终端自身颜色生成的 `system` 主题；`dark`/`light` 是备用。
3. **代码高亮**：按语言扩展名映射到 highlight.js 的语言，再用主题 token 上色。
4. **图片与动效**：Kitty/iTerm2 内联图片、spinner、工具卡状态背景、全屏 selection/search 高亮。

---

## 1. Markdown 渲染器

### 1.1 解析器

`Markdown` 使用 `marked`，但做了三处定制：

- `StrictStrikethroughTokenizer`：只接受 `~~text~~`，且内容首尾不能是空白/`~`，避免把普通波浪线误判为删除线。
- LaTeX 扩展：
  - 块级：`$$...$$`、`\[...\]`；
  - 行内：`$...$`、`\(...\)`、`\[...\]`；
  - 流式未闭合时标记 `pending`，先按原文显示，闭合后再渲染。
- 流式代码围栏修剪：`trimPartialClosingFences()` 会删掉只输入了一半的 closing fence（例如只到了 `` ``` `` 的前两个反引号），避免代码块在流式过程中反复“长高/缩短”。

解析结果缓存：

- `cachedTokens` 是 `WeakRef<{source, tokens}>`，按源码文本复用；
- token 树大约是源码的 10 倍大小，用弱引用让长 transcript 不永久持有；
- 主题/宽度变化只重渲染，不重新 parse。

### 1.2 渲染顺序

```text
render(width)
  → contentWidth = width - paddingX*2
  → transform(markdown, contentWidth)   // 扩展/替换
  → 空文本直接返回 []
  → Tab → 3 空格
  → parse tokens（或复用缓存）
  → 逐 token renderToken()
  → 对每个渲染行 wrapTextWithAnsi(contentWidth)
  → 加左右 margin；有 bg 时 applyBackgroundToLine，否则补空格到 width
  → 上下 paddingY
```

注意：`renderToken()` 先生成“逻辑行”，再统一折行。表格/代码块内部已经按宽度处理过，外层折行通常不会改变它们，但长代码行仍可能被再次折行。

### 1.3 块级元素

#### 标题

- `#`、`##`、`###` 等；
- 所有标题都 `bold`；
- `h1` 额外 `underline`；
- `h3` 及以上会在行首保留 `### ` 前缀；`h1/h2` 不显示 `#` 前缀；
- 如果下一个 token 不是 `space`，标题后加一个空行。

#### 段落

- 渲染 inline tokens；
- 如果下一个 token 不是 `list` 或 `space`，段落末尾加空行。

#### 代码块

```text
```<lang>
  <code line>
```
```

- 第一行是 `theme.codeBlockBorder("```" + lang)`；
- 每行代码前加 `theme.codeBlockIndent`，默认两个空格；
- 如果有 `highlightCode`，按语言高亮；否则每行 `theme.codeBlock()`；
- 最后一行 ` ``` `；
- 如果下一个 token 不是 `space`，加空行。

高亮实现：

- 语言从代码围栏的 `lang` 或文件扩展名映射；
- 使用 `cli-highlight`（highlight.js）；
- 主题映射：
  - keyword → `syntaxKeyword`
  - built_in/type → `syntaxType`
  - literal/number → `syntaxNumber`
  - string/regexp → `syntaxString`
  - comment/doctag → `syntaxComment`
  - function/title → `syntaxFunction`
  - class/type → `syntaxType`
  - attr/variable/params → `syntaxVariable`
  - operator → `syntaxOperator`
  - punctuation/tag → `syntaxPunctuation`
  - addition → `toolDiffAdded`
  - deletion → `toolDiffRemoved`
- 无有效语言时不自动检测，整段用 `mdCodeBlock` 色。源码注释说明自动检测会把普通英文误判成 AppleScript 等。

#### 列表

- 每层缩进 4 个空格；
- 无序列表默认 `- `；`preserveOrderedListMarkers` 为真时保留源标记（`1)`、`*` 等）；
- 有序列表默认 `1. `、`2. `…；
- 任务列表加 `[x] ` / `[ ] `；
- 第一行用 `firstPrefix`，后续行用 `continuationPrefix`，宽度等于 marker 的 visibleWidth；
- 嵌套列表递归，深度 +1；
- loose list 的非最后一项后加空行。

#### 表格

```text
┌─<col>─┬─<col>─┐
│ header │ header │
├─<col>─┼─<col>─┤
│ cell   │ cell   │
└─<col>─┴─<col>─┘
```

- 边框开销 = `3*n + 1`（`│ ` + `(n-1)*" │ "` + ` │`）；
- 如果可用宽度连每列 1 格都放不下，退化为原始 Markdown 文本行；
- 自然列宽：表头/单元格渲染后的最大 visibleWidth；
- 最小列宽：每列最长单词宽度，上限 30；
- 如果最小列宽总和超过可用宽度，先每列 1 格，再按 `(minWordWidth-1)` 权重分配剩余；
- 否则先按最小列宽，再按 `(natural-min)` 权重把多余空间分给各列；
- 单元格按列宽 `wrapTextWithAnsi()`，每行补空格；
- 表头加粗；
- 行之间画分隔线，最后画底边框。

#### 引用

- 内容宽度 = width - 2；
- 每行前缀 `│ `；
- 引用内文本用 `theme.quote(theme.italic(...))`；
- 如果下一个 token 不是 `space`，加空行。

#### 分隔线

- `theme.hr("─".repeat(min(width, 80)))`；
- 后面加空行（除非下一个是 space）。

#### HTML

- 按纯文本输出 `token.raw.trim()`，不执行 HTML。

### 1.4 行内元素

- `text`：应用默认样式；如果带嵌套 tokens，递归。
- `strong`：`theme.bold()`，然后补回外层 stylePrefix，避免 bold 的 reset 清掉外层颜色。
- `em`：`theme.italic()` + stylePrefix。
- `codespan`：`theme.code()` + stylePrefix。
- `link`：
  - 文本 `theme.link(theme.underline(...))`；
  - 如果终端支持 OSC 8，输出 `hyperlink(styledText, href)`，不额外打印 URL；
  - 否则如果文本不等于 href，追加 ` (href)`；
  - `mailto:` 比较时去掉前缀。
- `br`：`\n`。
- `del`：`theme.strikethrough()` + stylePrefix。
- `latex`：调用 `renderLatex()` 转 Unicode 文本；失败则显示原文。
- `html`：原文。

`stylePrefix` 的作用：inline 样式函数（如 bold）会在结尾写 reset；如果不把外层的颜色前缀补回去，后面的文本会变成默认色。

### 1.5 背景与 padding

`Markdown` 的 `defaultTextStyle` 可以带 `color` 和 `bgColor`：

- `color` 在 inline 渲染时应用；
- `bgColor` 不逐字符应用，而是在最后补满整行后应用，保证背景覆盖 padding；
- 用户消息就是 `color=userMessageText, bgColor=userMessageBg`，所以整行（包括左右 padding）都是用户消息背景。

---

## 2. 主题系统

### 2.1 默认主题是 `system`，不是 `dark`

`getAvailableThemesWithPaths()` 把 `system` 排在最前，并注明“the default”。`createSystemTheme()` 从终端上报的颜色生成整套 token：

- 如果终端同时上报 background 和 16 色 palette：hue/saturation 来自 palette，lightness 由对比度曲线决定；
- 如果只上报 background：使用每个 color family 自己的 hue；
- 如果什么都没上报：使用 ANSI palette 索引和终端默认色，让终端自己渲染。

`system` 主题的生成逻辑在 `system-theme.ts`，包含：

- color families（neutral/blue/green/red/yellow/orange/violet/calamine/thinking 系列）；
- 每个 token 归属的 family；
- 对比度目标曲线（dark/light 两套多项式）；
- 在 OKHSL 中构造颜色，靠近黑/白时降低饱和度；
- 生成 `dim` token 集合。

这意味着：**同一个 Pi，在不同终端配色下看起来会不同**。要 1:1 复刻，必须复刻“system 主题生成”或至少允许用户选择固定 dark/light；只抄 dark.json 不够。

### 2.2 Token

`THEME_TOKENS` 约 55 个，分为 foreground / background 两类：

- 基础：accent、border、borderAccent、borderMuted、success、error、warning、muted、dim、text；
- 消息：thinkingText、userMessageBg/Text、customMessageBg/Text/Label；
- 工具：toolPendingBg、toolSuccessBg、toolErrorBg、toolTitle、toolOutput；
- Markdown：mdHeading、mdLink、mdLinkUrl、mdCode、mdCodeBlock、mdCodeBlockBorder、mdQuote、mdQuoteBorder、mdHr、mdListBullet；
- diff：toolDiffAdded/Removed/Context；
- 语法：syntaxComment/Keyword/Function/Variable/String/Number/Type/Operator/Punctuation；
- thinking 边框：thinkingOff/Minimal/Low/Medium/High/Xhigh/Max；
- 全屏：scrollbarTrack/Thumb、searchMatchBg/Text；
- bashMode。

可选 token 有 fallback：

- `scrollbarTrack` → `muted`
- `scrollbarThumb` → `text`
- `searchMatchBg` → `selectedBg`
- `searchMatchText` → `text`
- `thinkingMax` → `thinkingXhigh`

### 2.3 颜色模型

`Color` 有三种：

- `indexed`：0–255 ANSI 索引；
- `rgb`：sRGB 0–255；
- `oklch`：L/C/H，构造时保留，转 sRGB 时再算。

`parseColor()` 支持：

- `#RGB` / `#RRGGBB`；
- `oklch(L C H)`，L 可百分比；
- `okhsl(H S% L%)`；
- 数字 0–255；
- 变量引用（主题 `vars` 中解析）。

`okhslColor()` 会转成 sRGB，因为 OKHSL 的饱和度是相对于 sRGB gamut 的，保证不越界。

`Theme` 构造时预计算每个 token 的 ANSI 序列：

- `fgAnsi` / `bgAnsi`；
- `""` 表示终端默认前景/背景，分别用 `\x1b[39m` / `\x1b[49m`；
- `dimTokens` 会在前景色后追加 `\x1b[2m`，reset 用 `\x1b[22m`；
- `fg(token, text)` 返回 `${ansi}${text}\x1b[39m`；
- `bg(token, text)` 返回 `${ansi}${text}\x1b[49m`；
- `bold/italic/underline/inverse/strikethrough` 走 chalk。

`foregroundAnsi()`：

- indexed → `38;5;N`；
- truecolor → `38;2;R;G;B`；
- 256color → 找最近的 xterm 256 色：先找 6×6×6 cube，再比较灰阶，选择距离更近的。

### 2.4 主题文件

内置 `dark.json`、`light.json` 使用 OKHSL 变量：

```json
"vars": {
  "text": "okhsl(234 3% 89%)",
  "muted": "okhsl(229 6% 67%)",
  "violet": "okhsl(295 50% 67%)",
  "blue": "okhsl(232 54% 67%)",
  "green": "okhsl(159 59% 67%)",
  "red": "okhsl(20 72% 67%)",
  "yellow": "okhsl(83 88% 67%)",
  "blueBg": "okhsl(233 41% 24%)"
}
```

关键颜色：

| token | dark | light |
|---|---|---|
| accent | violet | violet |
| border | `okhsl(231 57% 65%)` | `okhsl(231 67% 55%)` |
| userMessageBg | `blueBg` (`233 41% 24%`) | `blueBg` (`235 19% 91%`) |
| toolPendingBg | `229 5% 24%` | `248 3% 91%` |
| toolSuccessBg | `158 46% 25%` | `156 21% 91%` |
| toolErrorBg | `19 54% 25%` | `24 23% 91%` |
| mdHeading | yellow | yellow |
| mdCode | violet | violet |
| mdCodeBlock | green | green |
| thinkingOff | `229 8% 49%` | `223 5% 80%` |
| thinkingXhigh | `337 81% 61%` | `337 74% 70%` |

主题文件可以自定义；`Theme` 会按 `appearance` 或颜色亮度自动判断 dark/light。主题切换时 `ui.invalidate()` 让所有组件重新渲染。

### 2.5 Markdown/Select/Editor 主题映射

`getMarkdownTheme()`：

- heading → `mdHeading`
- link → `mdLink`
- linkUrl → `mdLinkUrl`
- code → `mdCode`
- codeBlock → `mdCodeBlock`
- codeBlockBorder → `mdCodeBlockBorder`
- quote → `mdQuote`
- quoteBorder → `mdQuoteBorder`
- hr → `mdHr`
- listBullet → `mdListBullet`
- bold/italic/underline/strikethrough → 对应属性
- highlightCode → `highlightCode()`

`getSelectListTheme()`：

- selectedPrefix/selectedText → accent
- description/scrollInfo/noMatch → muted

`getEditorTheme()`：

- borderColor → borderMuted
- selectList → 上面的 SelectListTheme

`getThinkingBorderColor(level)` 把 thinking level 映射到 `thinkingOff`…`thinkingMax`。编辑器边框和 working spinner 都用这个颜色。

---

## 3. 图片

### 3.1 能力探测

`detectCapabilitiesFromEnvironment()`：

- tmux → 不启用图片，只根据 tmux 是否转发 OSC 8 决定 hyperlinks；
- screen → 不启用图片/超链接；
- herdr → kitty；
- Kitty（`KITTY_WINDOW_ID` / `TERM_PROGRAM=kitty`）→ kitty；
- Ghostty → kitty；
- WezTerm → kitty；
- Warp → kitty；
- iTerm2 → iterm2；
- Windows Terminal → 无图片，truecolor，hyperlinks；
- alacritty/vscode/zed → 无图片，truecolor，hyperlinks；
- JetBrains JediTerm → 无图片，truecolor，无 hyperlinks；
- 未知终端 → 保守：无图片、无 hyperlinks、只有 `COLORTERM` 提示 truecolor。

可以用环境变量覆盖：

- `PI_IMAGE_PROTOCOL=kitty|iterm2|none`
- `PI_TRUE_COLOR=1|0`
- `PI_HYPERLINKS=1|0`

### 3.2 Cell size

默认 `{widthPx: 9, heightPx: 18}`。`TuiBase.start()` 只在终端支持图片时发送 `CSI 16 t` 查询；终端回 `CSI 6 ; height ; width t`，`consumeCellSizeResponse()` 更新 cell dimensions 并 `invalidate()`，让图片重新排版。

### 3.3 图片尺寸计算

`calculateImageCellSize(imageDimensions, maxWidthCells, maxHeightCells, cellDimensions, optimizeAspectRatio)`：

```text
widthScale = maxWidth * cellWidthPx / imageWidthPx
heightScale = maxHeight * cellHeightPx / imageHeightPx
scale = min(widthScale, heightScale)
scaledWidthPx = imageWidthPx * scale
scaledHeightPx = imageHeightPx * scale
columns = clamp(ceil(scaledWidthPx / cellWidthPx), 1, maxWidth)
rows = ceil(scaledHeightPx / cellHeightPx)
rows = min(maxHeight, rows)
```

如果 `optimizeAspectRatio`（Kitty）：

- 如果宽度是限制因素，按 `columns` 反算理想 rows，再用 `chooseLessDistortedCellCount()` 在 `floor/ceil` 中选失真更小的；
- 如果高度是限制因素，按 rows 反算理想 columns。

### 3.4 编码

Kitty：

```text
ESC _ G a=T,f=100,q=2[,C=1][,c=columns][,r=rows][,i=imageId];<base64> ESC \
```

- `a=T` 传输并显示；
- `f=100` PNG；
- `q=2` 静默；
- `C=1` 不移动光标（`Image` 组件用这个，自己返回多行占位）；
- 超过 4096 字符的 base64 分成 `m=1` 多段。

iTerm2：

```text
ESC ] 1337 ; File=inline=1;width=<columns>;height=auto;preserveAspectRatio=1:<base64> BEL
```

`Image.render()`：

- `maxWidth = min(width-2, options.maxWidthCells ?? 60)`；
- 默认 maxHeight 按 cell 宽高比算；
- Kitty + 非 PNG：用注册的 `ImageTranscoder` 转 PNG（EXIF 旋转后重新读 PNG 尺寸）；
- Kitty：返回 `[sequence, "", "", ...]`，共 `rows` 行；
- iTerm2：前 `rows-1` 行空，最后一行先 `CSI <rows-1>A` 上移、画图、再下移，保证 TUI 光标记账不越界；
- 没有图片能力：回退为 `[Image: filename mime WxH]`，如果支持 OSC 8 且路径是绝对路径，文件名可点击打开。

### 3.5 全屏图片

`TuiAltScreen` 额外处理：

- Kitty 图片上传缓存，避免每帧重传；
- 图片跨行裁剪：`cropKittyImageLine()` 用 `y/h` 参数只显示可见部分；
- WezTerm 的文本写入会擦掉覆盖的图片单元格，所以先写文本、最后画图片；
- 退出备用屏时删除所有 Kitty 图片，避免残留。

---

## 4. 动效

### 4.1 Loader

`Loader` 是 `Text` 子类：

- 默认 10 帧 braille spinner：`⠋⠙⠹⠸⠼⠴⠦⠧⠇⠏`；
- 默认 80ms 一帧；
- `render()` 返回 `["", ...super.render(width)]`，即上面空一行；
- `start()` 启动 interval，`stop()` 清除；
- `setMessage()` 更新文本并 `requestRender()`；
- `setIndicator()` 可以自定义 frames/interval，空 frames 隐藏。

Working/Retry/Compaction/BranchSummary 状态都是 `StatusIndicator extends Loader`，只是颜色、文案和计时器不同。

### 4.2 状态嵌入编辑器边框

`CustomEditor` 的 `renderTopBorder()` 会把 working 状态嵌进上边框：

```text
── ⠋ Working ─────────────────────────────
```

- 状态文本经过 `StatusIndicator.renderInBorder(width-5)`，去掉边框字符并截断；
- 如果有隐藏行（`↑ N more`），会把两者排在同一个边框上，并保证不重叠；
- 空间不够时只显示 spinner；
- 再不够时把 spinner 居中，左右用 `─` 填满；
- 颜色用 thinking level 的边框色，和编辑器边框一致。

这是 Pi 很有辨识度的视觉：spinner 不是单独一行，而是在输入框上边框里。

### 4.3 工具卡背景

`ToolExecutionComponent` 的 `Box` 背景由状态决定：

- `isPartial` → `toolPendingBg`；
- 结果错误 → `toolErrorBg`；
- 成功 → `toolSuccessBg`。

工具的 `renderCall`/`renderResult` 可以自带“shell”：

- `renderShell: "default"`：外层 Box 提供 padding+背景；
- `renderShell: "self"`：工具自己画边框/背景，外层只加 spacer 和图片。

### 4.4 闪烁消息

全屏 `TuiAltScreen.flash(message, durationMs?)` 用 `AltScreenFlashContainer` 把消息画在屏幕右上/底部，默认 1s 后消失；复制成功/失败提示用它。

---

## 5. 与 Java 复刻直接相关的差异点

1. **Markdown 解析器**：Java 已有 CommonMark + GFM tables，但 Pi 用 `marked` + 自定义 LaTeX/严格删除线。要 1:1，需要补：
   - LaTeX 块/行内；
   - 流式未闭合围栏修剪；
   - `preserveOrderedListMarkers` / `preserveBackslashEscapes`；
   - 表格列宽算法（Jade 当前实现接近，但需要逐项对齐）。
2. **代码高亮**：Pi 用 highlight.js；Java 没有完全等价库。可以：
   - 用 JHighlight/Tree-sitter Java 绑定；
   - 或先用正则/轻量词法，接受与 Pi 的差异；
   - 但主题 token 映射可以完全照搬。
3. **主题**：Java 需要实现 OKLCH/OKHSL → sRGB 转换和 256 色最近邻；如果只支持 16 色，视觉差异会很大。默认 `system` 主题还需要读取终端 OSC 10/11/4 回复。
4. **图片**：Java 可以用 Kitty/iTerm2 序列，不需要 native addon；但 PNG 尺寸解析、base64 分块、cell size 查询要自己实现。
5. **字体宽度**：图片/emoji 的单元格宽度依赖终端字体；Pi 的保守规则（regional indicator=2、emoji=2）要在 Java 中复刻。
6. **动效帧率**：Pi 80ms spinner、16ms 渲染节流；Java 的 `ScheduledExecutorService` 要注意不要在每个 token 上触发重绘。

---

## 6. 证据分级

- **已证实**：Markdown token 渲染、表格列宽、主题 token/颜色模型、system 主题生成、图片能力探测/编码、Loader/状态边框，均来自源码。
- **合理推断**：Java 侧“需要 ICU/OKLCH/highlight 替代”是根据依赖差异推断的；没有做 Java 实测。
- **无法确认**：终端实际上报的 palette/背景色、用户主题文件分布、不同终端对 Kitty 图片的裁剪行为。

---

## 7. 源码依据

| 相对路径 | 关键内容 |
|---|---|
| `packages/tui/src/components/markdown.ts` | `Markdown` 236；`render` 281；`renderToken` 464；`renderInlineTokens` 644；`renderList` 766；`renderTable` 852；LaTeX/strikethrough 1-145 |
| `packages/tui/src/colors.ts` | `parseColor` 121；`okhslColor` 107；`colorToRgb` 218；`foregroundAnsi` 309；`styleTextWithAnsi` 330 |
| `packages/tui/src/oklab.ts` | OKLab/OKHSL 转换 |
| `packages/coding-agent/src/modes/interactive/theme/theme-tokens.ts` | token 列表与 fallback |
| `packages/coding-agent/src/modes/interactive/theme/theme.ts` | `Theme` 190；`getThinkingBorderColor` 348；`loadTheme` 557；`getMarkdownTheme` 1022；`highlightCode` 930 |
| `packages/coding-agent/src/modes/interactive/theme/system-theme.ts` | system 主题生成、color families、对比度曲线 |
| `packages/coding-agent/src/modes/interactive/theme/dark.json` | dark 主题色值 |
| `packages/coding-agent/src/modes/interactive/theme/light.json` | light 主题色值 |
| `packages/tui/src/terminal-image.ts` | 能力探测 70；`encodeKitty` 226；`encodeITerm2` 293；`calculateImageCellSize` 481；`renderImage` 670；`imageFallback` 750 |
| `packages/tui/src/components/image.ts` | `Image` 55；`render` 98 |
| `packages/tui/src/components/loader.ts` | `Loader` 17；默认帧/80ms 11-12 |
| `packages/coding-agent/src/modes/interactive/components/status-indicator.ts` | `StatusIndicator` 12；working/retry/compaction 39-110 |
| `packages/coding-agent/src/modes/interactive/components/custom-editor.ts` | 状态嵌入上边框 39-97 |
| `packages/coding-agent/src/modes/interactive/components/tool-execution.ts` | 背景状态/渲染壳 65-230 |
| `packages/coding-agent/src/core/tools/renderers/*.ts` | 各工具的 `renderCall`/`renderResult` |

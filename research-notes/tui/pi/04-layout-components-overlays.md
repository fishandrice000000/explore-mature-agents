# TUI - Pi - 04 - 布局、组件、Overlay 与文本几何

---

1. C
2. 主要是视觉布局与交互.
3. JadeCode 基于 Java, 优先 Ubuntu.
4. 需要移植对照.

你可以顺带研究一下 JadeCode 的源码: /home/fishandrice/Backend/JadeCode/src/main/java/com/jadecode

---

## 0. 这一层解决什么问题

Pi 的视觉不是“若干字符串拼接”，而是一套可预测的几何系统：

- 每个组件只拿到一个 `width`，自己负责把内容折行、补齐、裁切；
- 布局层把组件放进 `rect`，并用 `clip` 限制可见区域；
- 合成层把 Overlay、选择高亮、搜索高亮、flash 消息叠到同一帧；
- 所有列宽都按终端单元格计算，而不是按 JavaScript/Java 字符数。

1:1 复刻时，最常出现的偏差不是“颜色不对”，而是：

- 中文/emoji 宽度算错，导致行尾多一格或少一格；
- 折行位置差一个字符，导致后续所有行错位；
- Overlay 合成时把 base 行的 ANSI 状态截断，导致颜色泄漏；
- 滚动条/Overlay 占了一列但没有从内容宽度里扣除；
- 鼠标坐标没有从屏幕坐标转换到组件局部坐标。

这一篇按“文本 → 组件 → 布局 → 滚动 → Overlay”的顺序拆。

---

## 1. 三种坐标

复刻时必须明确区分：

| 坐标 | 含义 | 谁使用 |
|---|---|---|
| 文档行（document row） | 完整内容数组中的行号，可能远大于屏幕高度 | 主屏 `previousLines`、全屏 `ScrollView` 内容 |
| 屏幕行（screen row） | 当前终端可见区的 0-based 行 | 全屏 `previousScreen`、鼠标事件 |
| 组件局部坐标 | 相对组件 `rect` 左上角的列/行 | `handleMouse`、Editor 命中测试 |

主屏的鼠标处理很弱，因为默认不捕获鼠标；全屏则必须做完整的坐标变换：

```text
屏幕 (screenX, screenY)
  → Overlay 命中？转换到 overlay 局部坐标
  → 布局命中：getLayoutBoxesAt() 返回从深到浅的 box 链
  → 每个 box：x = screenX - rect.x, y = screenY - rect.y
  → ScrollView：内容行 = scrollTop + (screenY - rect.y)
```

`Frame`/`LayoutBox` 的 `rect` 和 `clip` 是两套矩形：

- `rect`：组件被分配到的区域；
- `clip`：父级裁剪后真正可见的区域。

绘制时必须用 `clip`，命中测试时也要先判断 `clip`，否则会点到被遮住的部分。

---

## 2. 文本测量：`visibleWidth`

### 2.1 快路径

`visibleWidth(str)` 先尝试 `asciiVisibleWidth()`：

- 可打印 ASCII（0x20–0x7e）每字符 1 列；
- Tab 固定 3 列；
- ANSI 序列跳过，不计宽；
- 遇到其他字符立即返回 -1，走慢路径。

这条快路径很重要：大多数带样式的英文行都能在 O(n) 内算完，不需要 grapheme 分割。

### 2.2 慢路径

```text
1. 如果包含 Tab，先替换为 3 个空格
2. 用 ansiCodeLength() 逐段剥离 CSI/OSC/APC
3. 用 Intl.Segmenter(granularity: "grapheme") 分字素
4. 对每个字素调用 graphemeWidth()
5. 结果写入 512 项的 LRU-ish cache
```

### 2.3 `graphemeWidth` 的规则

按源码顺序：

1. 单字符 ASCII：可打印 1，Tab 3。
2. `terminalSpacingMarkRegex`：某些 spacing mark 即使没有 base 也占列，返回码点数量。
3. `zeroWidthRegex`：Default_Ignorable、Control、Mark、Surrogate → 0。
4. 可能是 emoji 且匹配 `RGI_Emoji` → 2。
5. 去掉 leading non-printing 后取 base 码点。
6. Regional Indicator（U+1F1E6–U+1F1FF）→ 2。源码注释说明：终端通常把孤立 regional indicator 也画成 full-width，保守取 2，避免流式输出时自动折行漂移。
7. `eastAsianWidth(cp)` 取基础宽度。
8. 对 base 后面的字符再做一轮：
   - terminal spacing mark → +1；
   - 普通 mark → 标记 `followsMark`；
   - 非 printing 字符：如果跟在 mark 后或是 halfwidth/fullwidth forms，按 `eastAsianWidth` 加；Thai/Lao AM 元音 +1。

这套规则不是标准 `wcwidth`，而是“为了终端实际渲染稳定”做过修正的版本。Java 复刻时如果直接用 `WCWidth.wcwidth()`，会在 emoji ZWJ、regional indicator、Indic spacing mark、Thai/Lao AM 上产生差异。

### 2.4 ANSI 序列识别

`ansiCodeLength()` 只识别：

- CSI：`ESC [` 开始，终止于 `m`、`G`、`K`、`H`、`J`；
- OSC：`ESC ]` 开始，终止于 BEL 或 `ESC \`；
- APC：`ESC _` 开始，终止于 BEL 或 `ESC \`。

这意味着组件行里只应出现这些控制序列。`CURSOR_MARKER` 是 APC，`OSC 8` 是 OSC，SGR 是 CSI m，光标移动是 CSI G/H/J/K。其他 CSI（例如 `?25l`）属于终端控制，不应出现在组件行里。

`stripTerminalSequences()` 用同样的识别器剥掉不可见序列，保留正文。

---

## 3. 折行、截断与切片

### 3.1 `wrapTextWithAnsi(text, width)`

它**只折行，不补 padding，不加背景**。

```text
按 \r\n|\r|\n 拆逻辑行
维护一个 AnsiCodeTracker
对每一行：
  prefix = 上一行结束时仍 active 的 ANSI 状态
  wrapSingleLine(prefix + line, width)
  用本行原始文本更新 tracker
```

`wrapSingleLine`：

- 如果 `visibleWidth <= width`，直接返回原行。
- 用 `splitIntoTokensWithAnsi()` 把文本拆成“空白 token / 单词 token”，ANSI 码附着到下一个可见字符。
- CJK 字符会被当成独立 token，允许在任意 CJK 字符之间断行；英文单词保持完整，除非单词本身超过宽度。
- 如果 token 超过宽度且不是空白：先 flush 当前行，再 `breakLongWord()` 按 grapheme 强制断开。
- 如果加入 token 会超宽：当前行 `trimEnd()`，加上 `tracker.getLineEndReset()`（只关闭 underline/hyperlink，保留背景色），push；如果是空白 token，新行以 `tracker.getActiveCodes()` 开头并跳过该空白；否则新行从 active codes + token 开始。
- 最后再对每行 `trimEnd()`，因为尾部空白可能让行超宽。

关键设计：**背景色要跨行保留**，所以行尾不能简单 `\x1b[0m`；`AnsiCodeTracker` 会分别记录 bold/dim/italic/underline/blink/inverse/hidden/strikethrough/fg/bg/hyperlink，生成“只关闭需要关闭的属性”的 reset。

### 3.2 `truncateToWidth(text, maxWidth, ellipsis = "...", pad = false)`

- `maxWidth <= 0` → 空串。
- `ellipsisWidth >= maxWidth`：如果原文能放下就返回原文；否则截断 ellipsis 本身。
- 纯 ASCII 快路径。
- 慢路径逐 grapheme 保留前缀，直到 `keptWidth + w > targetWidth`；ANSI 码挂在下一个保留的 grapheme 前。
- 如果输入耗尽且没有溢出，返回原文（需要时补空格）。
- 否则 `finalizeTruncatedResult()`：关闭 OSC 8、写 reset、写 ellipsis、再 reset；需要时补到 `maxWidth`。

注意：截断时不会把宽字符切成半格；如果边界放不下一个宽字符，就停在前一个完整字素。

### 3.3 `sliceByColumn` / `sliceWithWidth`

用于 Overlay 合成、选择复制、搜索高亮。

- `sliceWithWidth(line, startCol, length, strict)`：
  - 跳过 ANSI，但把 range 之前的 ANSI 暂存到 `pendingAnsi`，在第一个进入 range 的 grapheme 前补上；
  - 按 grapheme 宽度推进 `currentCol`；
  - `strict=true` 时，只有 `currentCol + w <= endCol` 才保留，避免宽字符越过右边界。
- `sliceByColumn()` 只返回 `.text`。

### 3.4 `extractSegments`

Overlay 合成需要把一行拆成“before / after”两段，并让 after 继承 before 末尾的样式。

```text
extractSegments(line, beforeEnd, afterStart, afterLen, strictAfter)
  → { before, beforeWidth, after, afterWidth }
```

它用一个池化的 `AnsiCodeTracker` 跟踪样式；当第一个 after 字符出现时，把 `getActiveCodes()` 插到 after 前面。这样 Overlay 右侧的文本不会丢掉左侧的颜色/背景。

### 3.5 `compositeTuiLine`

```ts
compositeTuiLine(baseLine, overlayLine, startCol, overlayWidth, totalWidth)
```

- 如果 base 是图片行，直接返回 base（不能在图片上叠文字）。
- `afterStart = startCol + overlayWidth`。
- 从 base 提取 before/after；从 overlay 截取 overlayWidth。
- 补空格对齐；每段之间插 `SEGMENT_RESET = "\x1b[0m\x1b]8;;\x07"`，防止样式/超链接跨段泄漏。
- 最后如果总宽超过 `totalWidth`，再按列裁一次。

这是 Overlay 视觉正确的核心。如果只做字符串拼接，ANSI 状态会从 base 泄漏到 overlay，或从 overlay 泄漏到右侧内容。

---

## 4. 基础组件

### 4.1 `Container`

- `children: Component[]`；
- `render(width)`：按顺序渲染每个子组件，把所有行拼起来；
- `handleMouse(event)`：按子组件渲染高度做垂直命中，把 `event.y` 减去前面子组件高度，递归分发；
- `invalidate()` 递归。

`Container.handleMouse` 会在宽度变化时重新渲染子组件来得到高度；如果宽度没变，使用缓存的 `mouseLayout`。这说明鼠标命中和布局高度必须一致，不能各自估算。

### 4.2 `Box`

`Box(paddingX, paddingY, bgFn)`：

- 内容宽度 = `max(1, width - paddingX*2)`；
- 先渲染所有子组件，保留未加 padding 的 `childLines` 作为缓存 key；
- 顶部/底部各 `paddingY` 行；
- 每行加左右 padding，补到 `width`，再应用 `bgFn`；
- 缓存命中需要 width、childLines 内容、`bgFn("test")` 采样都相同；
- `setBgFn()` 不主动失效缓存，而是靠采样值变化检测。

`Box` 的鼠标命中：先把坐标减掉 padding，再按子组件高度分发。

### 4.3 `Text`

`Text(text, paddingX=1, paddingY=1, bgFn?)`：

- 空文本/全空白 → 返回空数组；
- Tab → 3 空格；
- `paddingX` 会被压缩到 `(width-1)/2`，保证内容至少 1 列；
- `wrapTextWithAnsi()` 折行；
- 每行加左右 margin；有背景时 `applyBackgroundToLine()` 补到 width 并上色；无背景时补空格到 width；
- 上下各 `paddingY` 行；
- 缓存 key 是 text + width。

### 4.4 `TruncatedText`

只取第一行（到 `\n` 为止），按 `truncateToWidth` 裁切，加 padding，补到 width；支持上下 padding。用于 footer/状态行这类必须单行的位置。

### 4.5 `Spacer`

渲染 `lines` 个空字符串。注意它**不补 width**；空行在合成时由上层补。

### 4.6 `Input` / `SelectList` / `Loader` / `Image`

这些是交互/视觉组件，分别在 05、06 篇展开。这里只记它们的几何契约：

- `Input`：单行，prompt + 值，水平滚动，光标用假光标；
- `SelectList`：每项 `→ ` / `  ` 前缀，主列宽 32 左右，描述列，滚动信息 `(i/n)`；
- `Loader`：`Text` 子类，spinner + message，默认 80ms 一帧；
- `Image`：返回 `rows` 行，第一行是图片序列，其余为空行；Kitty 用 `C=1` 禁止光标移动。

---

## 5. 布局树

### 5.1 布局节点

布局不是靠组件类型判断，而是组件可选实现一个 symbol 方法：

```ts
component[LAYOUT_NODE](): LayoutNode
```

节点类型：

- `vstack`：`entries`, `gap`, `align`；
- `hstack`：同上；
- `scroll`：`component`（唯一子节点）, `state`（ScrollView）。

普通组件没有布局节点，按“固定内容”处理：渲染一次，得到行数和宽度。

### 5.2 `layoutComponent`

```text
layoutComponent(context, component, x, y, width, height, clip)
```

- `width` 先夹到至少 1。
- 没有布局节点：
  - `lines = renderCached(component, width)`；
  - `allocatedHeight = height ?? lines.length`；
  - 如果分配高度小于行数，找到 `CURSOR_MARKER` 所在行；如果光标行在分配高度之外，设置 `lineOffset` 让光标所在区域可见；
  - 返回 `{rect, clip, lines, lineOffset}`。
- `scroll` 节点：
  - 先取 `previousScrollTop`；
  - `contentWidth = scrollView.getContentWidth(width)`（滚动条常显时减 1）；
  - 递归布局子组件，`y = y - previousScrollTop`，高度不限制；
  - `contentHeight` 来自子布局；
  - `viewportHeight = height ?? contentHeight`；
  - `scrollView.updateLayout(contentHeight, viewportHeight, requestRender)` 可能会调整 `scrollTop`（follow-end）；
  - 把子 box 平移 `previousScrollTop - newScrollTop`；
  - 返回 scroll box，记录 `scrollContentLines`。
- `vstack`：
  - 过滤 `visible` 条目；
  - 每个条目的 intrinsic height：`basis` 是数字就用数字，否则 `measureHeight()`；
  - `allocateStackSizes(entries, intrinsicHeights, height, gap)`；
  - 自然高度 = 所有 size + gap；没有高度约束时用自然高度；
  - 依次布局子组件，`childY += size + gap`。
- `hstack`：
  - 每个条目的 intrinsic width：`basis` 数字或 `measureWidth()`；
  - `allocateStackSizes(entries, intrinsicWidths, width, gap)`；
  - 每个子组件的高度由 `measureHeight()` 得到；
  - `align` 决定 `start/center/end/stretch`；
  - 宽度为 0 的子组件不渲染，直接放一个空 box。

`renderCache` 是 `Map<Component, Map<number, string[]>>`，同一组件在同一宽度只渲染一次。这一点很重要：`VStack` 测量阶段和最终绘制阶段会多次请求同一组件，缓存避免重复解析 Markdown/高亮代码。

### 5.3 `allocateStackSizes`

这是 Pi 的“简化 flex”：

```text
sizes[i] = basis 数字 ? basis : intrinsic
clamp 到 [minSize, maxSize]

contentSize = available - (n-1)*gap
total = sum(sizes)

total < contentSize → distribute(+diff, "grow")
total > contentSize → distribute(-diff, "shrink")
```

`distribute` 的权重：

- grow：`entry.grow`；
- shrink：`(entry.shrink ?? 1) * max(1, size)`。

注意 shrink 权重不是单纯的 `shrink`，而是“shrink 系数 × 当前尺寸”。越大的条目缩得越多，类似 flex 的 `flex-shrink * base-size`。

分配循环每次至少给 1 格（`proposed = max(1, floor(remaining * weight / totalWeight))`），并受 `capacity` 限制；如果一轮没有分配出去，直接停止。这样能保证确定性，不会因为浮点/取整死循环。

### 5.4 绘制 `paintBox`

```text
paintBox(box, screen, totalWidth)
  - 如果 box.lines：
      取 clip 与 rect 的交集
      对每行：
        去掉 OSC133 区域前缀
        如果是 Kitty 图片，按 clip 底部裁剪
        如果 box 从 0 开始且宽度覆盖全屏，并且目标是空行：
          直接放 source line（快路径）
        否则 compositeTuiLine()
  - 递归子 box
  - 如果 scrollView 有被滚出顶部的 Kitty 图片，裁剪后补到首行
  - 画滚动条
```

`lineOffset` 用于“组件内容比分配高度长，但必须让光标可见”的情况。Editor 在窗口高度不足时会用到。

`paintBox` 的 fast path 只适用于 `box.rect.x === 0 && box.rect.width >= totalWidth` 且目标行为空；否则必须走合成，因为要保留 base 的 ANSI 状态。

---

## 6. `ScrollView`

`ScrollView` 只有一个子组件，不能 `addChild/removeChild/clear`。

状态：

```ts
currentScrollTop;
contentHeight;
currentViewportHeight;
followingEnd;          // 是否跟随末尾
followSuppressedAtEnd; // 手动滚到末尾但不恢复 follow
transientScrollbarVisible;
scrollbarActive;
```

关键行为：

- `follow: "end"`：内容增长时自动把 `scrollTop` 设为 `maxScrollTop`。
- `scrollBy(lines)`：
  - 如果当前在 follow-end，从 `maxScrollTop` 开始算；
  - 返回“未消费的 delta”（正数表示还差多少没滚），供外层链式滚动。
- `scrollTo(scrollTop, { disableFollow: true })`：滚到末尾但不恢复 follow。
- `updateLayout(contentHeight, viewportHeight, requestRender)`：
  - 夹取 `scrollTop`；
  - 如果 follow-end，自动贴底；
  - 内容不足一屏时隐藏临时滚动条。
- 滚动条：
  - `hidden/auto/always`；
  - `auto` 只在滚动后显示，默认 1000ms 后隐藏；指针悬停时 `scrollbarActive=true` 会保持；
  - `getContentWidth(width)` 在 `always` 且 width>1 时返回 `width-1`，给滚动条留一列。
- `render(width)`：按内容宽度渲染子组件；如果内容宽度比 width 少 1，给每行补一个空格，保证滚动条列有背景。

滚动条的几何在 `layout.ts`：

```text
trackHeight = viewportHeight
thumbHeight = max(min(2, trackHeight),
                  min(trackHeight, round(trackHeight² / contentHeight)))
maxScrollTop = max(0, contentHeight - trackHeight)
maxThumbTop = trackHeight - thumbHeight
thumbTop = round(scrollTop / maxScrollTop * maxThumbTop)
column = rect.x + rect.width - 1
```

绘制时用 `replaceScrollbarCell()` 只替换目标列的 grapheme，保留该单元格的背景色（除非是 `always` 模式）。这就是为什么滚动条能覆盖在带背景的内容上而不破坏整行样式。

---

## 7. Overlay

### 7.1 数据

```ts
type OverlayStackEntry = {
  component;
  options?;
  preFocus;     // 打开前的焦点
  hidden;
  focusOrder;   // 越大越靠前
  bounds?;
};
```

### 7.2 尺寸与位置

`resolveOverlayLayout(options, overlayHeight, termWidth, termHeight)`：

1. margin 解析，负数夹到 0。
2. `availWidth = max(1, termWidth - marginLeft - marginRight)`；高度同理。
3. width：
   - `parseSizeValue(options.width, termWidth)`，数字或 `"50%"`；
   - 默认 `min(80, availWidth)`；
   - `minWidth` 取 max；
   - 夹到 `[1, availWidth]`。
4. maxHeight：
   - 数字或百分比，夹到 `[1, availHeight]`；
   - 实际高度 = `min(overlayHeight, maxHeight)`。
5. row/col：
   - 优先 `options.row/col`；
   - 百分比是“在可移动范围内的百分比”：`maxRow = availHeight - effectiveHeight`，`row = marginTop + floor(maxRow * percent)`；
   - 否则按 `anchor`：top/center/bottom、left/center/right 九宫格；
   - 再 apply `offsetX/offsetY`；
   - 最后夹到终端边界。
6. 返回 `{width, row, col, maxHeight}`。

### 7.3 合成

`compositeOverlays(lines, termWidth, termHeight)`：

1. 清空所有 `bounds`。
2. 过滤 `visible` 的条目，按 `focusOrder` 升序排序（后画的在上面）。
3. 对每个 Overlay：
   - 先用 `height=0` 解析 width/maxHeight（这两个不依赖高度）；
   - `component.render(width)`；
   - 如果超过 maxHeight，截断；
   - 用实际高度再解析 row/col；
   - 记录 `bounds`；
   - 计算 `minLinesNeeded = max(minLinesNeeded, row + overlayLines.length)`。
4. `workingHeight = max(result.length, termHeight, minLinesNeeded)`。
5. 把结果补空行到 `workingHeight`。
6. `viewportStart = max(0, workingHeight - termHeight)`。
7. 对每个 Overlay 的每一行：
   - `idx = viewportStart + row + i`；
   - 如果 overlay 行宽超过声明宽度，先 `sliceByColumn` 裁切；
   - `result[idx] = compositeLineAt(base, overlayLine, col, width, termWidth)`。

为什么要有 `workingHeight` 和 `viewportStart`：Overlay 的 `row` 是相对终端可见区的；当基础内容比终端高时，Overlay 要贴在“当前可见区”的对应位置，而不是文档顶部。`workingHeight` 至少是终端高度，保证底部锚定的 Overlay 有足够的行。

源码注释特别提到：不要用 `maxLinesRendered` 来 padding。历史上用历史最高水位会导致“终端变宽时把内容挤进 scrollback”的自我强化问题。

### 7.4 焦点

Overlay 的焦点恢复是一个小状态机：

- `showOverlay()` 记录 `preFocus = focusedComponent`，然后 `setFocus(component)`。
- `hide()`：如果当前焦点是该 Overlay，优先恢复到“最上层可见捕获 Overlay”，否则恢复到 `preFocus`。
- `setHidden(true)`：如果焦点在该 Overlay，移到下一个可见 Overlay 或 `preFocus`。
- `setHidden(false)`：如果可见且不是 `nonCapturing`，重新聚焦并把 `focusOrder` 提到最前。
- `unfocus()`：释放到下一个可见 Overlay 或 `preFocus`；可以指定 `target`。
- `overlayFocusRestore` 处理“Overlay 仍开着，但用户点击/聚焦了外部组件”的情况；当外部组件卸载或再次按键时，决定是否恢复 Overlay。

复刻时如果只做“hide 就回到 preFocus”，嵌套对话框会出现焦点跳错层。

### 7.5 鼠标

`dispatchMouseToOverlay(event)` 从 `renderedOverlayLayouts` 的末尾向前遍历（视觉最上层优先），命中后把屏幕坐标转成 Overlay 局部坐标，再 `dispatchMouseEvent()`。

如果 Overlay 没有处理，事件不会被“穿透”到基础内容；源码返回 `{hit: true}`，表示该位置被 Overlay 占据但未处理。这样点击对话框空白处不会误触下面的聊天记录。

---

## 8. 与 Java 复刻直接相关的差异点

1. **JS 的 `Intl.Segmenter` 没有完全等价的 Java 标准库。** Java 的 `BreakIterator.getCharacterInstance()` 对 emoji ZWJ/regional indicator 的处理与 ICU 不完全一致；`\X` 正则基于 JDK 的 grapheme 规则。要 1:1，建议引入 ICU4J 或自己实现与 Pi 相同的 `graphemeWidth` 规则，并用测试用例锁定。
2. **JS 字符串是 UTF-16，Java 也是 UTF-16。** 但 Pi 的 `visibleWidth` 按 code point/grapheme 计算；Java 不能直接用 `String.length()`。
3. **ANSI 解析器要保留 OSC 8 的 terminator。** Pi 的 `AnsiCodeTracker` 会保留 BEL/ST，因为有些终端只把 BEL 结尾的链接做成可点击。
4. **Overlay 合成必须做 ANSI 状态继承。** 只按列拼接字符串会让颜色泄漏；`extractSegments()` 是必须复刻的。
5. **滚动条会占用内容宽度。** `ScrollView.getContentWidth()` 在 `always` 模式下减 1；如果 Java 布局忘记扣这一列，Markdown 会在最后一列被滚动条盖住。
6. **Stack 的 shrink 权重是 `shrink * size`。** 不是纯 flexbox 的 `shrink`，移植时不要直接套 CSS flex 公式。

---

## 9. 证据分级

- **已证实**：文本宽度规则、折行/截断/切片、布局节点、stack 分配公式、ScrollView 行为、Overlay 尺寸/合成/焦点、鼠标命中，均来自源码。
- **合理推断**：“1:1 需要 ICU 级 grapheme”是根据 Pi 使用 `Intl.Segmenter` 和自定义宽度规则推断的；具体 Java 实现选择需要实测。
- **无法确认**：不同 JDK/ICU 版本、不同终端字体对 emoji 的实际单元格分配。源码只能定义“按什么规则算”，不能保证所有终端画得一样。

---

## 10. 源码依据

| 相对路径 | 关键内容 |
|---|---|
| `packages/tui/src/utils.ts` | `visibleWidth` 250；`graphemeWidth` 182；`wrapTextWithAnsi` 900；`wrapSingleLine` 925；`breakLongWord` 1022；`truncateToWidth` 1121；`sliceWithWidth` 1268；`extractSegments` 1326 |
| `packages/tui/src/tui.ts` | `Component` 117；`Container` 358；`compositeTuiLine` 426；`renderFakeCursor` 225；`CURSOR_MARKER` 219 |
| `packages/tui/src/components/box.ts` | `Box` 15；渲染/缓存 100-172 |
| `packages/tui/src/components/text.ts` | `Text` 8；`render` 55 |
| `packages/tui/src/components/truncated-text.ts` | `TruncatedText` 7 |
| `packages/tui/src/components/spacer.ts` | `Spacer` 6 |
| `packages/tui/src/components/stack.ts` | `Stack` 37；`allocateStackSizes` 135；`distribute` 84 |
| `packages/tui/src/components/v-stack.ts` | `VStack.render` 12 |
| `packages/tui/src/components/h-stack.ts` | `HStack.render` 11 |
| `packages/tui/src/components/scroll-view.ts` | `ScrollView` 22；`scrollBy` 143；`updateLayout` 188；滚动条 55-120 |
| `packages/tui/src/layout.ts` | `layoutComponent` 106；`paintBox` 330；`renderLayoutFrame` 379；`getLayoutBoxesAt` 415；`getScrollbarGeometry` 260 |
| `packages/tui/src/tui.ts` | `resolveOverlayLayout` 1190；`compositeOverlays` 1350；`showOverlay` 729 |
| `packages/tui/src/components/markdown.ts` | Markdown 的折行/表格在 06 篇 |

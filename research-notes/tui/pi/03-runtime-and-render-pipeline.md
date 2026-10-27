# TUI - Pi - 03 - 运行模型、终端适配与差分渲染

---

1. C
2. 主要是视觉布局与交互.
3. JadeCode 基于 Java, 优先 Ubuntu.
4. 需要移植对照.

你可以顺带研究一下 JadeCode 的源码: /home/fishandrice/Backend/JadeCode/src/main/java/com/jadecode

---

## 0. 先给结论

Pi 的 TUI 不是一个“把字符串打印到终端”的 CLI，而是两层独立系统：

1. **`packages/tui`**：通用终端 UI 库。它只认识 `Component`、`Terminal`、`TUI`，不认识消息、工具、模型。核心能力是行式组件树、差分渲染、备用屏视口、Overlay、键绑定、编辑器、Markdown、图片。
2. **`packages/coding-agent/src/modes/interactive`**：产品层。它把 Agent 事件流翻译成消息卡、工具卡、状态栏、选择器、主题和输入行为。

这两层的关系类似“渲染引擎 + 应用界面”。要 1:1 复刻视觉与交互，必须同时复刻；只做 `packages/tui` 会得到一个能画框的库，只做产品层会失去滚动、CJK 宽度、IME 光标、Overlay 合成这些底层行为。

另一个必须先纠正的事实：**当前版本的默认交互模式是备用屏全屏，不是主屏内联**。默认值在 `settings-defaults.ts` 里是 `tuiMode: "fullscreen"`；`regular` 才是主屏内联模式。两者共用同一套输入、组件、Overlay 和消息组件，只替换最外层渲染器。做 1:1 时不能只挑一种。

```text
CLI
  → InteractiveMode
  → createInteractiveTui()
      ├─ fullscreen → TuiAltScreen
      └─ regular    → TuiMainScreen
  → 同一个 Component 树 / 同一套输入与编辑器
```

---

## 1. 入口、对象与生命周期

### 1.1 入口路径

```text
packages/coding-agent/src/main.ts
  → InteractiveMode
  → createInteractiveTui({ tuiMode, terminal, ... })
  → TuiMainScreen | TuiAltScreen
  → InteractiveMode.init()
  → ui.start()
  → 事件循环与 editor.onSubmit
```

`createInteractiveTui()` 是唯一的组合根。它负责：

- 选择 `ProcessTerminal` 或注入的终端；
- 全屏模式传入搜索高亮、滚动条、复制、打开 URL、右键粘贴等选项；
- 主屏模式只构造 `TuiMainScreen`；
- 给 `InteractiveMode` 一个稳定的 `TUI` 代理，因为运行中可以整体替换渲染器（`switchTuiMode()`）。

产品层始终操作 `TUI` 接口，不直接依赖 `TuiMainScreen`/`TuiAltScreen` 的具体类型。只有少数能力（全屏选择、搜索、滚动）通过 `instanceof TuiAltScreen` 或 `isViewportTUI()` 判断。

### 1.2 `Component` 是最小协议

```ts
interface Component {
  render(width: number): string[];
  handleInput?(data: string): void;
  handleMouse?(event: TuiMouseEvent): TuiMouseEventResult | undefined;
  wantsKeyRelease?: boolean;
  invalidate(): void;
}
```

几个关键约束：

- `render(width)` 返回“行数组”，每行是一个可能带 ANSI/OSC/APC 的字符串。组件不直接写终端。
- 组件通常要自己把每行补齐到 `width`，否则主屏差分时会出现残留字符。`TuiMainScreen` 在差分路径里还会 `\x1b[2K` 清行，但组件宽度契约仍然重要。
- `invalidate()` 只清缓存，不触发渲染。真正触发由 `TUI.requestRender()` 完成。
- `focused` 不是单独接口要求，而是 `Focusable` 的可选字段。焦点组件在渲染时插入 `CURSOR_MARKER`，TUI 找到它并定位硬件光标，供 IME 候选窗使用。

`CURSOR_MARKER` 是一个零宽 APC 序列：

```text
\x1b_pi:c\x07
```

还有一对“假光标”标记：

```text
\x1b_pi:fc\x07 ... \x1b_pi:/fc\x07
```

组件用假光标把某个字符或空格画成反色；TUI 在合成前把假光标解析成 `\x1b[7m`/`\x1b[27m`。如果硬件光标已经显示，并且假光标紧跟在 `CURSOR_MARKER` 后，则丢弃假光标，避免“双光标”。

### 1.3 `TuiBase` 管什么

`TuiBase` 继承 `Container`，是所有渲染器的共同基类。它管理：

- 子组件数组；
- 焦点组件；
- 全局输入监听器；
- Overlay 栈；
- 渲染调度与节流；
- 终端颜色查询；
- 生命周期钩子；
- 鼠标路由的公共部分；
- `invalidate()` 传播。

它不负责“怎么把行写到终端”，那是 `TuiMainScreen`/`TuiAltScreen` 的 `doRender()`。

### 1.4 渲染调度：16ms 节流 + 输入抢占

`TuiBase` 的调度逻辑是复刻时很容易漏掉的部分：

- 普通 `requestRender()`：置 `renderRequested=true`，在 `process.nextTick()` 里安排一个 `setTimeout`。
- 定时器延迟 = `max(0, 16ms - 距上次渲染的时间)`。也就是说，普通动画/流式更新最多约 60fps。
- 如果已经有一个待执行定时器，不会重复安排。
- 键盘输入走 `requestImmediateRender()`：取消当前定时器，在 `nextTick` 里直接渲染。源码注释明确说明，Windows 上即使 `setTimeout(0)` 也可能吃掉一个 16ms tick，所以输入必须抢占。
- `renderNow(force)` 是同步立即渲染；`force` 会先 `resetRenderState()`，用于外部编辑器返回、主题切换等需要全量重绘的场景。
- `start()`：`terminal.start()` → `hideCursor()` → 可选开启终端配色通知 → 查询 cell size（仅当终端支持图片）→ 首次 `requestRender()`。
- `stop()`：取消定时器 → 关闭配色通知 → `beforeTerminalStop()` → 显示光标 → `terminal.stop()` → `afterTerminalStop()`。

这套调度的行为含义：

- 流式 token 不需要每个 token 都画一帧；16ms 合并一次。
- 按键必须立即看到回显，不能等下一帧。
- `requestRender()` 不是“立即画”，组件不能假设调用后已经上屏。

### 1.5 焦点、输入和 Overlay 的优先级

`handleTerminalInput()` 的顺序：

1. 消费终端颜色查询回复；
2. 消费终端明暗模式报告；
3. 依次调用全局 `inputListeners`，监听器可以 `consume` 或改写 `data`；
4. 消费 cell size 回复；
5. 全局调试键 `shift+ctrl+d`；
6. 如果当前焦点 Overlay 因 resize/`visible()` 变得不可见，重定向焦点；
7. 如果焦点不在 Overlay 上，但存在“应当恢复的 Overlay 焦点”，先恢复；
8. 把输入交给焦点组件的 `handleInput`；
9. 过滤 key release（除非组件声明 `wantsKeyRelease`）；
10. 请求立即渲染。

这意味着：**Overlay 的焦点恢复优先级高于普通组件输入**。复刻时如果只做一个 `focusedComponent` 字段，会遇到“对话框关闭后焦点回到错误组件”“嵌套 Overlay 关闭后焦点跳错层”的问题。

`showOverlay()` 会把条目压入栈，记录 `preFocus`、`focusOrder`、`hidden`、`bounds`。默认自动聚焦；`nonCapturing: true` 时不抢焦点。隐藏/显示/关闭时会按“最上层可见捕获 Overlay → preFocus”的顺序恢复焦点，并维护一套 `overlayFocusRestore` 状态机来处理“Overlay 还开着，但用户把焦点移到了外部组件”的情况。

这部分在 04 篇展开。

---

## 2. 终端适配层：`ProcessTerminal`

`ProcessTerminal` 是 Node 进程与真实终端之间的唯一适配器。它负责的不只是 `stdin/stdout`。

### 2.1 启动

```text
start(onInput, onResize)
  → 保存原 raw 状态
  → setRawMode(true)
  → stdin.setEncoding("utf8"); stdin.resume()
  → 写入 \x1b[?2004h（开启 bracketed paste）
  → stdout.on("resize", onResize)
  → refreshTerminalDimensions()（POSIX 发 SIGWINCH）
  → Windows 开启 ENABLE_VIRTUAL_TERMINAL_INPUT
  → queryAndEnableKittyProtocol()
```

关键点：

- **raw mode**：否则 Ctrl+C、Ctrl+Z、Ctrl+D 会被终端驱动截走，TUI 收不到。
- **bracketed paste**：终端把粘贴内容包在 `\x1b[200~ ... \x1b[201~` 里，TUI 才能把“粘贴”与“用户逐字输入”区分开。
- **resize**：Node 的 `stdout` 有 `resize` 事件；但进程被挂起/恢复后尺寸可能过期，所以主动发 `SIGWINCH` 让 Node 重新读取。
- **Windows VT input**：必须在 `setRawMode(true)` 之后设置，因为 raw mode 会重置 console mode。

### 2.2 Kitty 键盘协议协商

这是 Pi 输入层最“终端特定”的部分。

启动时写入：

```text
CSI > 7 u     请求 flags 1|2|4
CSI ? u       查询当前 flags
CSI c         DA1 哨兵
```

flags 含义：

- `1`：disambiguate escape codes（把 Esc 与 Alt+key 等歧义拆开）；
- `2`：报告 press/repeat/release；
- `4`：报告 alternate keys（shifted key、base layout key）。

终端如果支持 Kitty，会回 `CSI ? <flags> u`；如果不支持，会回 DA1。Pi 用 DA1 作为“超时哨兵”，避免启动时固定等待：

- 先收到 Kitty flags → 启用 Kitty；
- 先收到 DA1 → 启用 `modifyOtherKeys` 回退（`CSI > 4 ; 2 m`）；
- 两者都可能被拆包，所以有一个 150ms 的重组缓冲。

这套协商的结果会写入全局 `setKittyProtocolActive()`，`keys.ts` 的 `matchesKey()` 会据此选择解析路径。

### 2.3 `StdinBuffer`：把字节流切成完整序列

终端的 `data` 事件不保证一次就是一个按键。一个 SGR 鼠标序列可能分三次到达：

```text
\x1b
[<35
;20;5m
```

`StdinBuffer` 负责：

- 缓存不完整 CSI/OSC/DCS/APC/SS3 序列；
- 识别 bracketed paste 的开始/结束，整段粘贴作为 `paste` 事件发出；
- 对普通文本按单字符发出；
- 对一个孤立的 ESC 等待 `escapeTimeout`（默认 10ms；SSH 下 100ms）再判定为 Esc；
- 对其他不完整序列等待 50ms；
- 识别终端回复头（OSC 颜色、Kitty graphics 等），等待终止符而不是把回复拆成按键；
- 处理 WezTerm 把 Esc press 与 Esc release 拼成 `\x1b\x1b[27;...u` 的特殊情况。

`ProcessTerminal` 在 `StdinBuffer` 的 `paste` 事件里重新包上 `\x1b[200~...\x1b[201~` 再交给 TUI。这样上层编辑器只需要处理一种“带标记的粘贴”格式。

### 2.4 停止与清理

`stop()` 的顺序非常重要：

1. 清除进度条（OSC 9;4;0）；
2. 清除程序状态（OSC 7501）；
3. 关闭 bracketed paste（`CSI ?2004l`）；
4. 关闭 Kitty 键盘协议（`CSI < u`）或 modifyOtherKeys；
5. 销毁 `StdinBuffer`，移除 stdin/stdout 监听；
6. `stdin.pause()`；
7. 恢复 raw mode。

`stdin.pause()` 是为了防止退出后缓冲的 Ctrl+D 被 shell 重新解释；源码注释提到 SSH 下这可能直接关掉父 shell。

`drainInput(maxMs=1000, idleMs=50)` 用于退出前把迟到的 Kitty key release 排干，避免泄漏到父 shell。

### 2.5 其他终端能力

- `setTitle()`：OSC 0。
- `setProgress(active)`：OSC 9;4;3 表示不确定进度，每 1s 重发；清除用 OSC 9;4;0。
- `setProgramStatus()`：OSC 7501，向支持的终端报告 working/blocked/done/error。只有终端确认支持后才写入。
- `clearLine()`：`CSI K`；`clearFromCursor()`：`CSI J`；`clearScreen()`：`CSI 2J CSI H`。

---

## 3. 主屏渲染器 `TuiMainScreen`

主屏模式的哲学是：**终端 scrollback 是历史，TUI 只维护当前可寻址的尾部区域**。

### 3.1 状态

```ts
previousLines: string[];      // 上一次实际写出的完整文档行
previousKittyImageIds: Set;   // 上一帧的 Kitty 图片 id
previousWidth/Height: number;
cursorRow: number;            // 文档逻辑末尾行
hardwareCursorRow: number;    // 终端光标实际所在行
maxLinesRendered: number;     // 历史最高水位
previousViewportTop: number;  // 当前可见窗口在文档中的起始行
```

`previousLines` 是完整文档，不是终端可见区。终端滚出视口的行已经无法用 ANSI 光标寻址，所以主屏差分只能改“仍在可见区且是尾部”的行。

### 3.2 一帧的完整流程

```text
doRender()
  → width/height 读取
  → 计算 prevViewportTop / viewportTop
  → newLines = resolveFakeCursors(render(width))
  → 有 Overlay 时 compositeOverlays(newLines, width, height)
  → extractCursorPosition(newLines, height)
  → applyLineResets(newLines)   // 每行 normalize + reset + OSC8 close
  → 选择 full render / diff render
```

注意顺序：**假光标解析在 Overlay 合成之前**，因为合成器只认识 SGR，需要先把 APC 假光标变成 SGR；**光标提取在 line reset 之前**，因为 reset 会改变字符串。

### 3.3 什么时候全量重绘

源码中的条件：

1. 第一次渲染：`previousLines.length === 0` 且宽高没变 → `fullRender(false)`，不清屏，假设屏幕干净。这样启动时不会清掉用户的 scrollback。
2. 宽度变化：一定全量重绘，因为所有换行都会变。
3. 高度变化：通常全量重绘；Termux 例外，因为软键盘弹出会频繁改高度，全量重绘会让历史反复重放。
4. `clearOnShrink` 开启且内容行数小于历史最高水位、且没有 Overlay：全量重绘以清除空行。
5. 第一个变化行在 `prevViewportTop` 之上：该行已经滚出可寻址视口，只能全量重绘。
6. 删除行导致目标行跑到视口上方：全量重绘。

`fullRender(clear)`：

- 开始 `CSI ?2026h`（同步输出）；
- 如果 clear：删除上一帧 Kitty 图片，写 `CSI 2J CSI H CSI 3J`（清屏、归位、清 scrollback）；
- 逐行写 `\r\n` 和行内容；
- 多行 Kitty 图片要预留行并移动光标；
- 结束 `CSI ?2026l`；
- 更新 `cursorRow/hardwareCursorRow/maxLinesRendered/previousViewportTop`；
- 定位硬件光标；
- 更新 `previousLines` 和图片 id。

### 3.4 差分算法

```text
找到 firstChanged / lastChanged
  - 逐行比较 previousLines 与 newLines
  - 如果新内容更长（appended），把 firstChanged 至少设为旧长度，lastChanged 设为新末行
  - 如果变化范围涉及 Kitty 图片，扩展到整张图片占用的行

没有变化 → 只更新硬件光标位置

所有变化都在删除行 → 移动到新内容末尾，清掉多余行

firstChanged < prevViewportTop → 全量重绘

否则：
  - 开始同步输出
  - 删除变化范围内的旧 Kitty 图片
  - 如果目标行在可见区下方：
      先移动到屏幕底部，再用 "\r\n" 滚动出足够行数
  - 移动到 firstChanged（用 hardwareCursorRow 计算相对移动）
  - 如果是纯追加，先 "\r\n"；否则 "\r"
  - 只渲染 firstChanged..lastChanged
      每行先 CSI 2K 清行，再写内容
      多行图片需要预清行并移动光标
      如果行宽超过终端宽度 → 写崩溃日志、stop、抛错
  - 如果旧内容更长，清掉多余行并把光标移回新内容末尾
  - 结束同步输出
  - 更新 cursorRow/hardwareCursorRow/maxLinesRendered/previousViewportTop
  - 定位硬件光标
```

几个容易忽略的细节：

- 差分更新只重画 `firstChanged..lastChanged`，不是“从 firstChanged 到末尾”。这样 spinner 只改一行时不会重画整屏。
- 追加内容时 `appendStart` 为真，移动目标行是 `firstChanged - 1`，然后写 `\r\n`，避免在旧末行上覆盖。
- 当目标行在可见区下方，不能直接用 `CSI nB` 移动（会停在该行但不会把中间行滚入历史），必须先移动到屏幕底部再写换行，让终端自然滚动。
- `maxLinesRendered` 是历史最高水位，用于 `clearOnShrink`；Overlay 合成时不会用它 padding，否则终端变宽时会把内容挤进 scrollback（源码注释明确提到这个自我强化问题）。
- 每行写入前 `CSI 2K` 清行；行内容本身不应带换行。
- 写入用 `BoundedTerminalWriter` 按 1 MiB 分块，避免超大 Kitty 图片行触发 V8 字符串上限。分块时不会拆 surrogate pair。
- 全量/差分都包在 `CSI ?2026h/l` 中，终端支持同步输出时不会看到半帧。

### 3.5 可见窗口与滚动

主屏没有应用级滚动。终端自己负责 scrollback。`previousViewportTop` 只用于判断“某行是否还在可寻址区域”。

`previousViewportTop = max(prevViewportTop, finalCursorRow - height + 1)`：当内容增长到超过屏幕高度时，视口顶部随之下移。下一次差分如果发现 `firstChanged < prevViewportTop`，说明变化行已经滚出可见区，只能全量重绘。

`clearOnShrink` 默认关闭，因为频繁全量重绘在慢终端上很贵；但内容缩到 0 时，`TuiMainScreen` 仍会通过删除行逻辑清掉旧尾部。测试 `tui-shrink.test.ts` 覆盖了这一点。

### 3.6 光标定位

`extractCursorPosition(lines, height)` 只扫描底部 `height` 行，找到第一个 `CURSOR_MARKER`，计算它之前的 `visibleWidth` 作为列，然后从行里删掉标记。

`positionHardwareCursor(cursorPos, totalLines)`：

- 把目标行夹到 `[0, totalLines-1]`；
- 用相对 `CSI nA/B` 从 `hardwareCursorRow` 移到目标行；
- 用 `CSI nG` 移动到绝对列（1-based）；
- 根据 `showHardwareCursor` 决定 `CSI ?25h` 或 `?25l`。

`hardwareCursorRow` 和 `cursorRow` 必须分开：

- `cursorRow` 表示文档内容末尾，用于视口计算；
- `hardwareCursorRow` 表示终端光标真实位置，用于下一次相对移动。

如果把两者混用，差分时光标会逐帧漂移。

### 3.7 主屏停止

`beforeTerminalStop()` 默认会在退出时把光标移到内容末尾并写 `\r\n`，避免 shell 提示符覆盖最后一行。`preserveScreen` 为真时跳过。

---

## 4. 备用屏渲染器 `TuiAltScreen`

全屏模式的哲学是：**终端可见区是一个固定高度的画布，历史由应用自己的 ScrollView 管理**。

### 4.1 进入备用屏

```text
beforeTerminalStart()
  → 重置选择/搜索/滚动状态
  → 读取图片能力
  → 如果终端是 iTerm2，先临时关闭图片能力，避免把图片序列画进备用屏
  → 写 CSI ?1049h       进入备用屏
    CSI ?7l            关闭自动换行
    mouse sequence     开启鼠标（1000/1002/1004/1006，或 tmux 下只用 1002）
    CSI 2J CSI H       清屏
    CSI ?25l           隐藏光标
```

tmux/zellij/screen 下只开启 button-motion（1002），不开启 all-motion（1003），因为多路复用器转发每个指针移动会明显拖慢。

### 4.2 布局根

全屏模式不直接渲染 `children`，而是渲染 `layoutRoot`：

- 如果没有显式 `layoutRoot`，用 `implicitScrollView` 包住所有 `children`，形成“一个可滚动文档”。
- 产品层会调用 `setLayoutRoot()` 设置：

```text
VStack
  ├─ ScrollView(transcript)  basis 0, grow 1, minSize 1
  └─ VStack(dock)            basis auto, shrink 1, minSize 1
       ├─ pending messages
       ├─ status
       ├─ widgets above
       ├─ editor
       ├─ widgets below
       └─ footer
```

`layoutRoot` 只影响全屏渲染；`getMountedRoots()` 返回 `layoutRoot ? [layoutRoot] : children`，所以 `invalidate()`、焦点挂载判断等也会跟着变。

### 4.3 一帧的完整流程

```text
doRender()
  → width/height
  → renderLayoutFrame(root, width, height)
      → 递归布局，得到每个组件的 rect/clip/lines
      → 逐层 paint 到 screen[]
  → 如果搜索面板有查询变化，重新布局一次
  → resolveFakeCursors + 移除 OSC133 区域标记
  → 应用搜索高亮
  → 合成 “Jump to latest message” 指示器
  → compositeOverlays()
  → 截到 height 行
  → 应用文本选择高亮
  → 合成 flash 消息
  → extractCursorPosition
  → applyLineResets + 超宽裁剪
  → 与 previousScreen 逐行比较
  → 写 CSI row;1H + CSI 2K + 行内容
  → 光标定位
```

全屏的差分粒度是“整行”：只要某行字符串变了，就重写整行。它没有主屏那种“第一个变化行之前不能改”的限制，因为所有行都在固定视口内、都可寻址。

### 4.4 图片的特殊处理

- Kitty 图片在备用屏里会被缓存，按 `imageId` 和 `transmissionGeneration` 判断是否需要重新传输。
- 图片不可见时不会立刻删除；缓存有数量/传输字节/解码字节上限（16 张、32MB 传输、64MB 解码），超过才驱逐。
- 当图片锚点行变化，或 WezTerm 的图片覆盖行被文本写入触碰，会触发重绘。
- WezTerm 有特殊路径：先清文本行，再画文本，最后画图片，避免后续文本写入擦掉 Kitty 图片覆盖的单元格。
- iTerm2 图片在进入备用屏前临时关闭图片能力，避免把图片序列写进备用屏；退出时恢复。

### 4.5 退出备用屏

`beforeTerminalStop()`：

- 关闭搜索；
- 停止选择自动滚动；
- 停止滚动条拖拽；
- 写同步输出开始、删除 Kitty 图片、关闭鼠标、恢复自动换行、同步输出结束。

`afterTerminalStop()`：

- 如果 `preserveScreen`：退出备用屏、显示光标，不打印文档；
- 否则：用当前宽度重新 `render()` 完整文档，去掉光标标记和 OSC133，裁剪到宽度，逐行写到主屏，然后退出备用屏、恢复自动换行、显示光标。

产品层的默认 `fullscreenExitOutput` 是 `transcript`：退出时先 `switchTuiMode("regular")`，再用主屏渲染器把完整 transcript 打出来，然后停止。这样用户退出全屏后仍能在 shell scrollback 里看到对话。

### 4.6 全屏鼠标、选择、搜索、滚动条

这些是“视觉交互 1:1”里最容易被低估的部分：

- SGR 鼠标解析：`CSI < b ; x ; y M/m`，支持 press/release/move/drag/click/wheel，修饰键位在 button 的 bit 2/3/4。
- 滚轮：优先发给指针下的 `ScrollView`，未消费的 delta 按 `overscroll: "chain"` 继续给外层；Alt+wheel 是 5 倍；`"auto"` 模式有加速器。
- 选择：按下时记录 anchor，拖动时按字符/词/行粒度扩展；双击选词，三击选行；拖到视口边缘自动滚动（50ms）。
- 选择复制：默认 `copyOnSelect=true`，松开鼠标即复制；复制走宿主提供的 `copySelection`，否则 OSC 52。
- 搜索：`ctrl+shift+f` 打开右上角 Overlay，输入查询，`Enter`/`ctrl+g` 下一个，`Shift+Enter`/`ctrl+shift+g` 上一个，`Esc` 关闭；匹配高亮来自 ScrollView 的完整内容行，而不是屏幕行。
- 滚动条：宽度 1 列，thumb 高度按 `viewportHeight² / contentHeight` 比例，最小 2 行；“auto” 模式在滚动后显示 1s，指针悬停时保持。
- “Jump to latest message”：当 primary ScrollView 是 `follow: end` 且用户滚上去时，在最后一行居中画一个可点击标签；点击恢复 follow-end。标签不能盖住滚动条最后一列（源码测试专门覆盖了这一点）。

---

## 5. 两套渲染器的差异

| 维度 | `TuiMainScreen` | `TuiAltScreen` |
|---|---|---|
| 终端缓冲区 | 主屏 + scrollback | 备用屏 |
| 历史 | 终端管理 | 应用 `ScrollView` 管理 |
| 布局 | 组件树顺序渲染，整篇文档 | 固定高度 `layoutRoot`，clip/rect |
| 差分 | 行 diff，受可见区限制 | 整行 diff，全部行可寻址 |
| 滚动 | 终端原生 | 应用滚动、滚轮、滚动条、搜索 |
| 鼠标 | 仅组件级点击（默认不捕获） | 捕获滚轮/拖拽/选择/滚动条 |
| 退出 | 内容留在主屏 | 退出备用屏，默认重放 transcript |
| 适用 | 轻量、保留 shell 历史 | 长对话、固定输入区、复杂交互 |

两者共用：`Component`、`Container`、Overlay、焦点、输入、编辑器、Markdown、主题、图片、键绑定。

---

## 6. 复刻时最关键的实现约束

1. **不要只实现一种渲染器。** 默认是全屏；`regular` 是另一种正式模式，不是降级。
2. **同步输出必须包住整帧。** 否则滚动/重绘会闪。
3. **主屏第一次渲染不要清屏。** 清掉用户 shell 历史是明显的行为差异。
4. **主屏差分只能改可见区。** 变化行滚出视口时必须全量重绘。
5. **主屏每行写前要清行。** 否则短行覆盖长行会残留。
6. **光标行与文档行要分开记账。** 否则相对移动会漂移。
7. **所有行宽必须按终端列计算。** CJK、emoji、组合字符、Tab、ANSI 都不能按 Java `String.length()` 算。
8. **假光标/硬件光标标记要在正确阶段处理。** 先解析假光标，再合成 Overlay；先提取光标，再 apply line reset。
9. **输入必须抢占 16ms 节流。** 否则快速打字会感觉粘滞。
10. **Overlay 不是“画在最上面”这么简单。** 焦点栈、preFocus、nonCapturing、visible 回调、鼠标命中顺序都要一致。
11. **全屏退出默认要重放 transcript。** 否则用户退出后看不到对话。
12. **终端能力要探测并降级。** Kitty/modifyOtherKeys、truecolor/256、图片协议、OSC 8、OSC 52 都不是所有终端支持。

---

## 7. 证据分级

- **已证实**：默认 `fullscreen`、两套渲染器、16ms 节流、输入抢占、主屏差分条件、全屏布局与退出重放、Kitty 协商、StdinBuffer 超时、同步输出、1MiB 分块、光标 marker 顺序，均可在源码直接确认。
- **合理推断**：把“视觉 1:1”理解为需要同时复刻主屏/全屏、Overlay、鼠标选择与主题；源码没有直接写“1:1 验收标准”，这是根据产品行为归纳的。
- **无法确认**：不同终端模拟器对 Kitty/图片/OSC 8 的实际支持差异，以及 Pi 线上默认主题/配置的真实分布。仓库只能确认代码路径与默认值。

---

## 8. 源码依据

| 相对路径 | 关键内容 |
|---|---|
| `packages/tui/src/tui.ts` | `Component` 117；`Container` 358；`compositeTuiLine` 426；`TuiBase` 506；`showOverlay` 729；`requestRender` 1001；`scheduleRender` 1035；`handleTerminalInput` 1055；`compositeOverlays` 1350 |
| `packages/tui/src/tui-main-screen.ts` | `TuiMainScreen` 124；`doRender` 247；全量/差分条件 300-430；差分写行 430-590；`positionHardwareCursor` 624 |
| `packages/tui/src/tui-alt-screen.ts` | `TuiAltScreen` 202；进入备用屏 346；退出 388/403；鼠标 901；滚轮 993；选择 1588；`doRender` 1679 |
| `packages/tui/src/terminal.ts` | `Terminal` 接口 88；`ProcessTerminal` 150；Kitty 协商 288；stop 418；drain 391 |
| `packages/tui/src/stdin-buffer.ts` | `StdinBuffer` 305；`process` 318；paste 分支 350；超时 430 |
| `packages/tui/src/keys.ts` | Kitty 解析 602；`matchesKey` 859；`parseKey` 1292 |
| `packages/tui/src/keybindings.ts` | 默认绑定 54；`KeybindingsManager` 231 |
| `packages/tui/src/layout.ts` | `layoutComponent` 106；`paintBox` 330；`renderLayoutFrame` 379；命中测试 415 |
| `packages/tui/src/components/stack.ts` | `allocateStackSizes` 135 |
| `packages/tui/src/components/scroll-view.ts` | `ScrollView` 22；follow/scrollbar 逻辑 88-230 |
| `packages/coding-agent/src/modes/interactive/tui-renderer.ts` | 两种渲染器选择 22-50 |
| `packages/coding-agent/src/modes/interactive/chat-viewport.ts` | 全屏 transcript + dock 布局 20-53 |
| `packages/coding-agent/src/core/settings-defaults.ts` | `tuiMode: "fullscreen"` 62；`fullscreenScrollbar` 64 |
| `packages/coding-agent/src/modes/interactive/interactive-mode.ts` | `init` 943；渲染器挂载 873；模式切换 890；事件处理 3408 |

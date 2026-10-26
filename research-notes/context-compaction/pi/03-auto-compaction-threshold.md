# Context compaction - Pi - 03 - 自动压缩的触发阈值

---

Pi 上下文自动压缩的触发阈值是多少?

---

## 结论先行

Pi 的自动压缩**不是百分比阈值**，而是一条“窗口减预留”的绝对 token 线：

```
当前上下文 token 数 > 模型上下文窗口 - reserveTokens
```

默认 `reserveTokens = 16384`，自动压缩默认开启。所以默认阈值就是**模型窗口减去 16384 token**，并且判定是严格大于（等于不触发）：

| 模型窗口 | 默认触发线 |
|---|---|
| 128,000 | 114,688 |
| 200,000 | 183,616 |
| 400,000 | 383,616 |
| 1,000,000 | 983,616 |

三个数字容易混淆，先分清职责：

- `compaction.enabled`（默认 `true`）：总开关；
- `compaction.reserveTokens`（默认 `16384`）：**决定什么时候触发**，预留给模型输出的空间；
- `compaction.keepRecentTokens`（默认 `20000`）：不参与触发判断，只决定压缩时保留最近多少 token 的原文。

## 机制展开

### 1. 判定式

触发判断本身只有一次比较：上下文 token 数严格大于“窗口 − 预留值”就压缩，否则正常请求。窗口小于等于 0 的模型直接跳过自动压缩判断，避免除以异常值。

这里是固定 token 预留，不是按窗口比例算。换成百分比视角，默认预留占 200K 窗口约 8.2%、占 128K 窗口约 12.8% —— 窗口越小，压缩来得相对越早，因为预留量是常数。

### 2. “当前上下文 token 数”怎么来的

不是每次都重新数全部消息，而是混合口径：

1. 优先取**最近一次有效助手响应**的 usage 总量（无效的 aborted、error、全零 usage 会被跳过）；
2. 该响应之后新增的消息（工具结果、新用户消息等）用本地估算补齐；
3. 文本按约 4 字符/token 估算，图片按固定 4800 字符近似；
4. 如果最近发生过压缩或上下文编辑，旧 usage 已不能代表当前投影，则改为对重建后的完整投影做估算。

本地估算被刻意设计成偏保守（宁可高估），所以真实触发点可能略早于“按供应商计费口径正好 183,616”的位置。

### 3. 什么时候做这次比较

有三个自动检查点，时机不同但都用同一条阈值：

| 时机 | 说明 |
|---|---|
| 工具批次结束后、下一次助手响应前 | 工具结果已经追加，若越过阈值就先压缩再发下一轮请求；如果这批工具已经结束整个运行且没有排队消息，则跳过 |
| 提交新的用户提示前 | 会检查上一条助手响应的用量，即使那次响应被用户中止也会检查 |
| 运行结束后 | 专门处理上下文溢出或可恢复的截断响应，压缩后最多重试一次 |

另外还有手动 `/compact`，它不看阈值，先中止当前操作再压缩。

因此压缩发生在“轮次边界”，不会打断一次正在流式输出的助手响应；工具结果也总是和对应的工具调用一起保留，切点不会落在工具结果上。

### 4. 阈值和其他限制的关系

- **预留值与窗口独立**：模型自身声明的最大输出 `maxTokens` 不参与这条公式。若某模型最大输出远大于 16384，预留偏保守；反过来若最大输出很小，预留就偏宽松。
- **溢出是另一条路径**：供应商报上下文溢出、或响应提前以 `length` 结束时，走“可恢复溢出”分支，不依赖这条阈值，也会压缩并最多重试一次。
- **同模型护栏**：如果那条助手响应来自另一个模型（用户中途换过模型），Pi 会跳过溢出判断，避免用旧小窗口模型的错误去触发新大窗口模型的压缩。
- **压缩后的陈旧用量**：早于最近一次压缩的助手消息不会再次触发压缩，防止刚压缩完立刻被旧 usage 顶回去。

### 5. 怎么改

设置在 `~/.pi/agent/settings.json`（全局）或 `<项目>/.pi/settings.json`（项目），项目设置与全局设置合并后生效：

```json
{
  "compaction": {
    "enabled": true,
    "reserveTokens": 16384,
    "keepRecentTokens": 20000,
    "modelOverrides": {
      "anthropic/claude-sonnet-4-5": { "reserveTokens": 32000 }
    }
  }
}
```

- 每个字段独立解析：模型覆盖 → 普通设置 → 内置默认；
- `modelOverrides` 的键是精确的 `provider/modelId`，区分大小写，不支持通配；
- 取值必须是非负安全整数，负数或非整数会在加载时报错；
- 边界行为：`reserveTokens` 设为 0 时阈值等于窗口本身（几乎只在真正用满时才压缩）；设得大于等于窗口时，阈值降到 0 或负数，此后每次检查都会触发压缩；`enabled` 为 `false` 时自动压缩完全不发生，手动 `/compact` 仍可用。

## 设计取舍

- **用绝对预留而不是比例**：预留空间对应的是“一次回复要写多长”，这个量由输出上限决定，与窗口大小不成比例。对长窗口模型，固定预留不会因为窗口变大而虚耗大量压缩空间。
- **阈值和保留量分离**：`reserveTokens` 管“何时压缩”，`keepRecentTokens` 管“压缩后留多少”，两者独立可调。压缩触发后保留 20000 token 原文，意味着压缩后的上下文大约落在 2 万 token 级别，而不是恰好贴着阈值。
- **保守估算**：宁可提前触发一次压缩，也不让请求撞上供应商的硬上限；代价是压缩可能比理论值更频繁，且摘要本身要花一次模型调用。

## 条件与边界

- 以上数值来自仓库内置默认值；用户设置和项目设置可以覆盖，实际线上取值需以本机配置文件为准。
- 供应商是否真的按 `contextWindow` 拒绝请求、以及 usage 是否精确，取决于各家 API；Pi 在这里只做客户端预算。
- 虚拟模型（路由模型）下，阈值使用路由实际选中的物理模型的窗口，而不是虚拟模型条目上的数字。
- `pi-durable` 有自己独立的压缩任务，同样默认预留 16384、保留 20000，但它是另一套运行时，不能把本文结论直接套过去。

## 证据分级

- **已证实**：判定式、默认值 16384 / 20000 / enabled=true、严格大于比较、三处自动检查点、模型覆盖的解析顺序与校验、`contextWindow <= 0` 跳过、压缩后陈旧 usage 防护。
- **合理推断**：本地估算偏保守导致的实际触发点略早；固定预留对小窗口模型相对更“激进”的取舍解释（源码注释只说明预留用于模型响应）。
- **无法确认**：具体某个供应商/模型在线上被配置的窗口值与实际 usage 口径，需要通过运行时配置与供应商文档确认。

## 源码依据

| 文件 | 位置 | 内容 |
|---|---|---|
| `packages/coding-agent/src/core/settings-defaults.ts` | `SETTINGS_DEFAULTS.compaction` | 默认 enabled / reserveTokens / keepRecentTokens |
| `packages/coding-agent/src/core/compaction/compaction.ts` | `shouldCompact`、`DEFAULT_COMPACTION_SETTINGS` | 阈值比较与内置默认 |
| `packages/coding-agent/src/core/compaction/compaction.ts` | `calculateContextTokens`、`estimateContextTokens`、`estimateProjectedContextTokens`、`estimateTokens` | usage 优先 + 估算补齐的口径 |
| `packages/coding-agent/src/core/agent-session.ts` | `_exceedsCompactionThreshold` | 请求前按当前投影判断 |
| `packages/coding-agent/src/core/agent-session.ts` | `_checkCompaction` | 三处自动检查与同模型/陈旧用量护栏 |
| `packages/coding-agent/src/core/settings-manager.ts` | `getCompactionTokenSetting`、`getCompactionSettings` | 模型覆盖 → 普通设置 → 默认的解析与校验 |
| `packages/coding-agent/src/core/settings-schema.ts` | compaction schema | 非负安全整数约束 |
| `packages/coding-agent/docs/settings.md` | Compaction 表 | 配置项与默认值 |
| `packages/coding-agent/docs/compaction.md` | When It Triggers | 触发公式与检查时机 |
| `packages/coding-agent/test/settings-manager-compaction.test.ts` | 全文件 | 默认值、独立解析、精确键匹配 |

# 第 15 章 上下文治理与 Compact 设计

> 状态: 已完成初稿
> 章节目标: 让系统在长会话中仍然稳定运行。

[返回总览](/Users/magongli/Downloads/project/claude-code-sourcemap/docs/plans/2026-03-31-claude-code-runtime-reproduction/README.md)

---

这一章讨论的是 Claude Code 风格系统真正能够“长期协作”的核心原因之一：它不是假装上下文无限，而是把上下文治理做成正式 runtime 能力。

如果没有这套机制，终端 agent 很快就会出现这些症状：

- transcript 越来越长
- token 成本失控
- 模型开始遗忘当前目标
- prompt cache 命中率下降
- 用户和系统都不知道哪些历史该保留、哪些该压缩

所以 compact 在这里不是优化项，而是系统能否长期稳定运行的生命线。

```mermaid
flowchart LR
  growth[session growth<br/>messages + tool results]
  thresholds[token thresholds<br/>warning / error / auto / block]
  request[compact request<br/>manual or auto]
  summary[summary / compaction result]
  boundary[compact boundary<br/>stored summary]
  restored[restored effective context]
  next[next query continues]

  growth --> thresholds --> request --> summary --> boundary --> restored --> next
```

## 15.1 为什么要把 compact 当 runtime 核心能力

很多系统会把 compact 做成一个 slash command，但 Claude Code 风格实现的关键判断是：

> manual compact 只是入口，compact runtime 才是能力本体。

原因有三点：

- 自动 compact 必须在 query 过程中被触发。
- compact 结果必须立刻影响当前回合上下文。
- compact 之后还要继续同一会话，而不是开启一个新系统。

因此 compact 不应该只挂在 UI 或会话管理外围，而应该直接进入 Query Engine 生命周期。

## 15.2 context 治理的四层目标

一个成熟的 compact 系统不只是“缩短消息”，它至少要同时解决四件事：

- 控制 token 压力
- 保留会话事实连续性
- 维持 prompt cache 可用性
- 降低长期 transcript 的噪音和失真

这也是为什么上游不只做一种压缩，而是做了多种分层治理：

- tool result budgeting
- snip
- microcompact
- context collapse
- auto compact
- session memory compaction
- manual compact

## 15.3 `autoCompact.ts` 的核心思想

从 `autoCompact.ts` 可以看出，上游把 auto compact 设计成了一个明确阈值系统，而不是“感觉快满了就压缩”。

关键函数包括：

- `getEffectiveContextWindowSize()`
- `getAutoCompactThreshold()`
- `calculateTokenWarningState()`
- `shouldAutoCompact()`
- `autoCompactIfNeeded()`

这说明 compact 是有数学边界和状态追踪的，而不是拍脑袋触发。

## 15.4 有效上下文窗口为什么要预留 summary 输出空间

`getEffectiveContextWindowSize()` 做了一个非常合理的动作：

- 先拿模型上下文窗口
- 再预留一段给 compact summary 输出的 token

也就是说，可用于触发 compact 的窗口不是模型理论总窗口，而是“减去摘要输出保留量后的可用窗口”。

这个设计非常重要，因为如果不预留摘要输出空间，compact 请求本身就可能因为没空间而失败。

这是一种很典型的 runtime self-hosting 思维：

- compact 不是系统外的工具
- compact 本身也要消耗 context 和 output budget

## 15.5 warning / error / auto / blocking 四个阈值层

从 `calculateTokenWarningState()` 可以看到，上游实际上区分了多个阈值层次：

- warning threshold
- error threshold
- auto-compact threshold
- blocking limit

这四层的意义分别是：

- warning: 向用户和系统提示压力上升
- error: 进入更明显的风险区域
- auto-compact: 达到自动压缩触发点
- blocking: 如果自动 compact 不可用，则开始阻止继续增长

这套设计很成熟，因为它避免了只有“正常”和“爆炸”两种状态。

## 15.6 为什么 auto compact 还有 recursion guard

在 `shouldAutoCompact()` 里，上游会特意避免对某些 query source 触发自动 compact，例如：

- `compact`
- `session_memory`
- 某些 context collapse 场景

这类 guard 极其重要，因为否则会出现递归压缩和死锁：

- compact agent 自己又触发 compact
- session memory 流程为了减小上下文，结果又进入 auto compact

这说明 compact 系统不是“任何时候都压”，而是必须理解自己在 runtime 中的角色位置。

## 15.7 `AutoCompactTrackingState` 的价值

上游不仅判断要不要 compact，还会追踪 compact 链状态：

- `compacted`
- `turnCounter`
- `turnId`
- `consecutiveFailures`

这个 tracking state 的价值很大：

- 可以区分是否是连续 recompaction
- 可以做失败断路器
- 可以记录距离上一次 compact 已经过了多少 turn
- 可以让 telemetry 理解 compact 链行为

这比“压了就压了”的黑盒状态成熟得多。

## 15.8 连续失败断路器为什么重要

`autoCompactIfNeeded()` 里有一个很关键的设计：

- 当连续 autocompact 失败达到一定次数后，停止继续重试

这个断路器非常必要，因为在某些不可恢复场景下，如果每一轮都继续自动 compact，只会：

- 浪费大量 API 调用
- 让用户卡在死循环
- 让系统错误看起来像“还在努力”

这类设计非常像成熟服务端系统里的 retry circuit breaker，只不过这里发生在 agent runtime 里。

## 15.9 session memory compaction 为什么先于普通 compact

从 `autoCompactIfNeeded()` 能看到，上游会先尝试：

- `trySessionMemoryCompaction()`

然后才进入更重的 `compactConversation()`。

这说明 Claude Code 风格系统并不只有一种压缩机制，而是先尝试更轻、更贴近会话记忆管理的压缩方式，再退到完整 summary 型 compact。

这背后的设计思想值得借鉴：

- 不是所有上下文压力都要靠昂贵摘要解决
- 优先选择对会话结构破坏更小的治理路径

## 15.10 `compactConversation()` 的真正职责

`compactConversation()` 不是“生成一段摘要”这么简单。从 `compact.ts` 能看出，它至少负责：

- 预处理待压缩消息
- 运行 pre-compact hooks
- 分析上下文
- 构造 compact prompt
- 调用模型生成 summary
- 生成 boundary marker
- 计算 summaryMessages
- 确定 messagesToKeep
- 追加 attachments / hookResults
- 运行 post-compact cleanup

换句话说，它是一次正式的“上下文重写事务”。

## 15.11 `CompactionResult` 为什么不是单字段摘要

`CompactionResult` 至少包含：

- `boundaryMarker`
- `summaryMessages`
- `attachments`
- `hookResults`
- `messagesToKeep`
- token 统计
- `userDisplayMessage`

这说明 compact 结果不是一句 summary，而是一整组会话重写产物。

特别是 `messagesToKeep` 非常关键，它说明 compact 并不总是“全量历史变摘要”，而可以保留一部分近期或关键上下文。

## 15.12 `buildPostCompactMessages()` 的顺序很重要

上游显式规定了 post-compact 消息的顺序：

1. `boundaryMarker`
2. `summaryMessages`
3. `messagesToKeep`
4. `attachments`
5. `hookResults`

这个顺序不是偶然的，因为它决定：

- query 恢复时从哪里重新开始
- 哪些消息属于压缩后的新上下文
- replay 时如何解释这次 compact

这说明 compact 不是“删旧消息 + 加摘要”，而是严格的 transcript rewrite protocol。

## 15.13 compact boundary 为什么是一等消息

Claude Code 风格系统里，compact boundary 不是内部标志，而是正式系统消息。这一点特别重要。

原因是：

- Query Engine 需要知道从哪个边界后开始取有效消息。
- transcript replay 需要知道历史已经在哪被压缩。
- resume / rewind 需要明确上下文切换点。
- SDK/headless 需要收到 compact 事件。

因此复现时必须把 compact boundary 做成正式 `SystemMessage` 或等价结构，而不是只改内存数组。

## 15.14 为什么要剥离图片和会被重注入的附件

`compact.ts` 里有两个非常有启发性的函数：

- `stripImagesFromMessages()`
- `stripReinjectedAttachments()`

背后的设计理由非常清楚：

- 图片对生成会话摘要价值不高，却会极大增加 compact 请求负担。
- 某些 attachment 反正 compact 后还会重新注入，再送给 summarizer 只会浪费 token。

这说明 compact 不是无脑摘要，而是先做“面向摘要任务的输入净化”。

这点很值得在复现里照着做。

## 15.15 partial compact 与 prefix-preserving 思维

从 `compact.ts` 能看到，上游不仅支持完整 compact，还考虑了 prefix-preserving 和 partial compact 的场景。

这意味着 compact 并不总是“把前面全压掉”，而可以：

- 保留一段前缀
- 保留一段尾部
- 只总结中间的一段

这种设计非常高级，因为真实长会话里经常会有：

- 最近的几轮必须保留原文
- 更早的一大段可以总结
- 某些边界消息又必须原样保留

所以 compact 系统最好从一开始就不要假设“只有一种压缩形态”。

## 15.16 post-compact reinjection 为什么要有预算

Claude Code 风格系统 compact 后不是只留 summary，还会按预算重新注入重要事实。

从 `compact.ts` 常量可以看出，它对 reinjection 有明确控制：

- `POST_COMPACT_MAX_FILES_TO_RESTORE`
- `POST_COMPACT_TOKEN_BUDGET`
- `POST_COMPACT_MAX_TOKENS_PER_FILE`
- `POST_COMPACT_MAX_TOKENS_PER_SKILL`
- `POST_COMPACT_SKILLS_TOKEN_BUDGET`

这说明系统意识到：

- compact 后仍需要恢复部分关键文件事实
- 但 reinjection 不能反过来把上下文再次撑爆

这是一个非常成熟的“压缩后恢复”思路。

## 15.17 compact 后恢复什么

从代码和整体设计看，compact 后通常需要恢复的内容包括：

- 关键文件附件
- 技能相关上下文
- MCP instruction delta
- agent listing delta
- deferred tools delta

这说明 compact 系统本质上不是“只做摘要”，而是：

- 先做会话压缩
- 再做重要事实重注入

也正因此，它才适合作为长期上下文治理能力，而不是一次性 summary 工具。

## 15.18 compact 与 raw transcript 的关系

一个非常关键的设计点是：

- compact 会改变 effective context
- 但 raw transcript 并不应当被概念性抹除

这也是为什么系统需要：

- compact boundary
- summaryMessages
- preserved segment metadata

这样 raw history 仍可回放，而 query 只消费 compact 后视图。

## 15.19 manual compact、auto compact、reactive compact 的边界

复现时建议把 compact 至少分成三类能力：

- `manual compact`: 用户显式触发
- `auto compact`: 阈值达到后自动触发
- `reactive compact`: API 已经报 prompt too long 后的恢复路径

上游实现里这三者明确分层，这很重要，因为它们的体验和策略完全不同。

- manual 适合用户可感知
- auto 适合顺滑兜底
- reactive 是故障恢复

## 15.20 compact 设计的推荐分层

建议复现项目里拆成以下模块：

- `token-pressure.ts`
- `compact-thresholds.ts`
- `compact-tracking.ts`
- `compact-conversation.ts`
- `post-compact-reinjection.ts`
- `compact-boundary.ts`
- `reactive-compact.ts`
- `session-memory-compact.ts`

这样未来就不会把 compact 做成单个巨型文件。

## 15.21 本章结论

第 15 章最重要的结论有五个：

- compact 是 runtime 核心能力，不是一个 slash command。
- 上下文治理必须有阈值、状态追踪、失败断路器和恢复策略。
- compact 结果是一次 transcript rewrite，不是一段摘要字符串。
- post-compact reinjection 与预算控制和 compact 本身同样重要。
- 长会话系统真正的稳定性，来自“压缩 + 保真恢复 + 明确边界”三件事一起成立。

## 15.22 本章对复现工程的直接指导

如果你真的要把长会话跑稳，这一章不是“增强项”，而是核心工程项。

### 15.22.1 第一版就做 token pressure 监测

哪怕你暂时不做自动 compact，也先把下面这些量算出来：

- 当前消息 token 估算
- 预留输出预算
- warning 阈值
- hard limit 阈值

没有这些量，compact 只能在报错后被动补救，很难做成真正顺滑的 runtime 能力。

### 15.22.2 compact 结果必须回写 transcript

不要把 compact 做成“临时内存里的摘要替换”。  
正确方向是：

- compact 产出结构化 summary
- 写入 compact boundary / summary message
- 让后续 query 明确知道上下文在哪个边界之后恢复

这样 resume、rewind、debug 才能解释得通。

### 15.22.3 reinjection 必须单独建模块

图片、system context、memory、部分附件、某些 instructions 往往需要 compact 后重注入。  
第一版就把这件事单列成模块，不要把 reinjection 写死在 compact 主函数里。

### 15.22.4 manual / auto / reactive 三条路径不要混

建议最少拆成：

- 用户主动压缩
- token 阈值触发压缩
- prompt-too-long 恢复压缩

它们的 UX、日志、错误语义都不同。混在一起以后，你会非常难查“为什么这次压缩发生了”。

### 15.22.5 第一版推荐目录

```text
runtime/compact/
  token-pressure.ts
  thresholds.ts
  tracking.ts
  compact-conversation.ts
  reinjection.ts
  reactive-compact.ts
```

这一章真正帮你避免的，是把长会话问题一直拖到“产品开始能用时”才发现上下文已经完全不可控。

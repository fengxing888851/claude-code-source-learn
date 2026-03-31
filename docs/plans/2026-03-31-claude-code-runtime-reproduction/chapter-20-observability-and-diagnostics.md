# 第 20 章 可观测性与诊断设计

> 状态: 已写  
> 章节目标: 让 runtime 在真实使用中可分析、可调优、可定位、可证明。

[返回总览](/Users/magongli/Downloads/project/claude-code-sourcemap/docs/plans/2026-03-31-claude-code-runtime-reproduction/README.md)

---

## 20.0 本章结论

Claude Code 风格 runtime 的可观测性设计非常值得学，因为它不是“写几条 debug 日志”那么浅，而是从一开始就把这些问题纳入内核：

- 启动为什么慢
- 一轮 query 为什么慢
- headless 模式首 token 为什么慢
- prompt 到底长什么样
- 哪个组件在掉线
- 哪个远程步骤失败了
- 记录日志时如何避免泄露文件路径、代码和 prompt

所以 observability 在这里不是可选配件，而是 runtime 能不能长期演进的前提。

---

## 20.1 设计目标

这一层至少要解决 5 类问题：

1. 正常使用时的低成本可见性  
开发者需要知道系统在干什么，但不能让每次运行都过度打点

2. 出问题时的快速定位  
能分辨是启动慢、上下文装配慢、模型慢、工具慢、网络慢

3. 线上使用时的趋势分析  
需要采样式 telemetry，而不是每次都写巨量本地日志

4. 敏感信息保护  
日志和埋点不能随手把 prompt、代码、文件路径都送出去

5. 跨模式一致性  
REPL、headless、remote、agent background task 都要能被观测

Claude Code 的实现基本就是围绕这 5 个目标展开的。

---

## 20.2 可观测性分层

从源码可以把整个观测体系拆成 6 层：

### 20.2.1 用户可见状态层

包括：

- progress message
- tool status
- agent notification
- reconnect/disconnect 状态

这层是给最终用户看的，不追求完整，只追求当前状态可理解。

### 20.2.2 debug log 层

如 `logForDebugging()` 一类接口，主要供开发排查细节问题。它通常可以包含更多技术上下文，但仍要注意不要失控。

### 20.2.3 diagnostics 层

如 `logForDiagnosticsNoPII()`，强调的是：

- 结构化
- 可机器处理
- 严格避免 PII

### 20.2.4 analytics / telemetry 层

如 `logEvent()`，强调趋势、采样、运营和性能统计，而不是完整调试信息。

### 20.2.5 profiler 层

如：

- startup profiler
- query profiler
- headless profiler

这层用于测量耗时与瓶颈。

### 20.2.6 prompt / API dump 层

如 `dumpPrompts.ts`，用于在必要时复盘：

- 发给模型的 system / tools / messages 到底是什么
- 模型返回了什么

这层最敏感，所以只适合在受控条件下开启。

---

## 20.3 logging 分层设计

### 20.3.1 为什么不能只靠 console.log

因为 runtime 里存在不同消费方：

- 最终用户
- CLI 开发者
- SDK 调用者
- 远程问题诊断者
- 性能分析者

他们需要的粒度完全不同。  
所以正确做法是定义不同日志层，而不是把一切都写到 stdout。

### 20.3.2 diagnostics 必须无 PII

`diagLogs.ts` 里最值得注意的一点是注释写得非常硬：

- 不能带 PII
- 不能带 file path
- 不能带 project name
- 不能带 prompt

并且 `logForDiagnosticsNoPII()` 是写结构化 JSON 行日志到 `CLAUDE_CODE_DIAGNOSTICS_FILE` 指定路径。

这背后的思想极其值得复现：

> 诊断日志应该从机制上假设自己会离开本地环境，所以默认只允许无敏感信息事件。

如果你把 diagnostics 和 debug 混在一起，后面想补救数据安全会非常痛苦。

### 20.3.3 withDiagnosticsTiming

`withDiagnosticsTiming()` 又往前走了一步，把诊断日志和耗时统计结合起来：

- `event_started`
- `event_completed`
- 失败时 `event_failed`
- 自动附带 `duration_ms`

这非常适合包住：

- git status
- mcp connect
- remote bridge step
- plugin load
- session restore

复现版强烈建议你直接引入这种包装器。

---

## 20.4 Analytics / Telemetry 设计

`services/analytics/index.ts` 非常值得仔细学，因为它把 analytics 做得既轻量又安全。

### 20.4.1 零依赖入口

文件顶部明确写了：

- 这是 analytics 的公共入口
- 它尽量没有依赖
- 这样可以避免 import cycle

这是一个非常成熟的设计。因为 analytics 往往被全局很多地方调用，如果这个模块自己依赖复杂，很容易造成启动链混乱和循环依赖。

### 20.4.2 先记队列，后挂 sink

`attachAnalyticsSink()` 之前，事件会被放进内存队列；sink 挂上后再异步 drain。

这解决了启动期一个很实际的问题：

- 早期模块已经想打点
- 但 analytics backend 尚未初始化完成

Claude Code 的做法不是丢弃这些事件，也不是强行阻塞初始化，而是先排队，再异步冲刷。

这个模式特别适合复现。

### 20.4.3 类型上约束敏感数据

最值得借走的是这个 marker type：

- `AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS`

它本质上是在类型系统层面提醒开发者：

- 不要把字符串随便塞进 analytics metadata
- 除非你确认这不是代码、不是文件路径、不是其它敏感信息

这当然不是绝对安全，但它是一种很好的工程文化约束。

### 20.4.4 `_PROTO_` 字段分流

源码还处理了 `_PROTO_*` 这类特殊字段，说明系统区分：

- 一般后端可见的数据
- 只允许进入受控 proto 字段的 PII 数据

复现版未必需要完全照抄，但建议至少把：

- `general telemetry`
- `restricted telemetry`

分开设计。

---

## 20.5 Prompt Dump 与 API 请求复盘

### 20.5.1 dumpPrompts 的定位

`dumpPrompts.ts` 不是普通日志，它更像一个“API 请求取证器”。

它会把与模型请求相关的关键信息写成 JSONL，便于事后排查：

- 系统 prompt
- tools schema
- metadata
- user messages
- API response

### 20.5.2 为什么写成 JSONL

JSONL 很适合这类场景：

- 追加写简单
- 单次请求可按事件顺序记录
- 便于 grep、流式分析和后处理

### 20.5.3 dump 条目类型

从实现看，至少包含这些记录类型：

- `init`
- `system_update`
- `message`
- `response`

这个设计很精妙：

- `init` 记录首次请求的系统信息
- 如果 system/tools 变化，再写 `system_update`
- 每一轮新增 user message 单独记录
- response 单独落一条

这样不会每次把相同的大块 system/tool schema 全量重复写入。

### 20.5.4 为什么要做 fingerprint 和 hash

`DumpState` 里维护了：

- `messageCountSeen`
- `lastInitDataHash`
- `lastInitFingerprint`

而 `initFingerprint()` 会先用一个便宜的摘要判断 model/tools/system 是否大体没变，只有变了才进一步 `stringify + hash`。

这点特别值得学，因为 prompt dump 很容易变成性能黑洞。  
Claude Code 的做法是：

- 先 cheap check
- 再 expensive hash
- 避免每次都为 MB 级请求体做昂贵序列化

### 20.5.5 为什么 dump 逻辑要异步

源码里显式用 `setImmediate()` 把 request dump 延后执行，因为：

- prompt stringify 很重
- 这是 debug tooling
- 不应阻塞真实 API 请求

这个原则很重要：  
调试能力不能反过来让系统更慢。

### 20.5.6 streaming response 也会被解析

对于 SSE 响应，代码会把整条 stream 解析成 chunk 列表再写入。这意味着 dump 工具不仅能看 request，还能看 streaming response 的结构。

这对排查以下问题非常有用：

- 模型到底有没有发 tool_use
- 某个 chunk 是否格式异常
- streaming 是否中途中断

---

## 20.6 Startup Profiler

`startupProfiler.ts` 说明启动性能在 Claude Code 里是第一等问题。

### 20.6.1 两种模式

它支持：

1. sampled logging  
100% ant，0.5% external，记录阶段耗时到 telemetry

2. detailed profiling  
设置 `CLAUDE_CODE_PROFILE_STARTUP=1`，输出完整 checkpoint 报告和内存快照

### 20.6.2 启动阶段统计

源码里内置了 phase 定义：

- `import_time`
- `init_time`
- `settings_time`
- `total_time`

这说明 startup profiler 不只是打印总耗时，而是按阶段归因。

### 20.6.3 详细报告落盘

详细模式下会把报告写到：

- `~/.claude/startup-perf/{sessionId}.txt`

这个设计非常实用，因为启动慢的问题常常发生在：

- 用户环境
- CI
- 某次特定配置组合

让它落文件，比只在 terminal 输出更好追踪。

### 20.6.4 启动 profiler 为什么值得早做

CLI/agent 工程很容易随着：

- import 增多
- settings 增多
- plugin load 增多
- analytics 初始化增多

变得越来越慢。  
如果你不从早期就插入启动 profiler，到后面再补，往往已经不知道退化是从哪次开始的。

---

## 20.7 Query Profiler

`queryProfiler.ts` 是本章最值得直接借用的设计之一。

### 20.7.1 它测的是什么

它测的是从“用户输入进系统”到“首个 token 到达”的整条 query pipeline，而不是只测模型 API。

源码注释里列了大量 checkpoint，例如：

- `query_user_input_received`
- `query_context_loading_start/end`
- `query_query_start`
- `query_fn_entry`
- `query_microcompact_start/end`
- `query_autocompact_start/end`
- `query_setup_start/end`
- `query_api_loop_start`
- `query_api_streaming_start`
- `query_tool_schema_build_start/end`
- `query_message_normalization_start/end`
- `query_client_creation_start/end`
- `query_api_request_sent`
- `query_response_headers_received`
- `query_first_chunk_received`
- `query_tool_execution_start/end`
- `query_recursive_call`
- `query_end`

这意味着它把 query 延迟拆得非常细，而不是笼统说一句“模型慢”。

### 20.7.2 TTFT 被拆成了两部分

报告里会专门区分：

- `pre-request overhead`
- `network latency`

这点极其有用。因为很多人以为首 token 慢一定是模型慢，但其实经常是：

- system/context 拼装慢
- tool schema 构造慢
- message normalization 慢
- client 初始化慢

复现时你也应该把 TTFT 至少拆成“发请求前”和“发请求后”两段。

### 20.7.3 慢点告警

profiler 还会对超过阈值的 checkpoint 打 slow warning，比如：

- 超过 100ms
- 超过 1000ms
- git status / tool schema / client creation 的专用提醒

这是一种很实用的“轻诊断”机制。  
你不必等到用户报卡顿才去肉眼翻几十行时间线。

---

## 20.8 Headless Profiler

`headlessProfiler.ts` 专门针对 `-p` 这类非交互模式。这说明 Claude Code 并没有把 headless 当作 REPL 的附属模式，而是单独考虑了它的延迟指标。

### 20.8.1 关心哪些指标

它主要看：

- turn 开始时间
- 首次 system message 输出时间
- query 启动时间
- first chunk 到达时间
- query overhead

### 20.8.2 为什么要单独做

因为 headless 模式下用户关心的是另一类体验：

- 命令返回速度
- 自动化脚本等待时间
- 第一屏输出延迟

而不是 REPL 里那种持续互动体验。

### 20.8.3 采样策略

它使用：

- 100% ant
- 5% external

的 sampled logging，同时在 `CLAUDE_CODE_PROFILE_STARTUP=1` 时输出更详细 debug。

这说明 profiling 也考虑了成本，不是每个用户都完整记录。

---

## 20.9 故障定位应该怎么读这些数据

如果你复现时也把这些层做起来，排查问题大概就能按下面路径走。

### 20.9.1 启动慢

优先看：

- startup profiler
- settings load 时长
- plugin load 时长
- import phase 时长

### 20.9.2 首 token 慢

优先看：

- query profiler 的 pre-request overhead
- context loading
- tool schema build
- message normalization
- client creation

### 20.9.3 headless 首次输出慢

优先看：

- headless profiler 的 system message yielded
- query started
- first chunk

### 20.9.4 工具执行诡异

优先看：

- diagnostics timing
- tool progress
- dumpPrompts 中是否真的发出了 tool schema
- response stream 是否真的返回了 tool_use

### 20.9.5 remote/bridge 断连

优先看：

- diagnostics no-PII log
- bridge state change
- reconnect / auth recovery 事件
- control request 是否悬挂未完成

---

## 20.10 复现时最少要做哪些观测能力

如果你资源有限，我建议从 day 1 就最少做下面这些：

1. 结构化 debug log
2. diagnostics file，严格 no-PII
3. startup profiler
4. query profiler
5. prompt dump 开关
6. task / agent progress 事件

这 6 项已经足以覆盖大部分真实排查场景。

---

## 20.11 复现工程建议接口

```ts
type DiagnosticLogger = {
  log(level: 'debug' | 'info' | 'warn' | 'error', event: string, data?: Record<string, unknown>): void
  time<T>(event: string, fn: () => Promise<T>): Promise<T>
}

type QueryProfiler = {
  start(): void
  checkpoint(name: string): void
  end(): void
}

type PromptDumpWriter = {
  recordRequest(sessionId: string, request: unknown): void
  recordResponse(sessionId: string, response: unknown): void
}
```

同时给这些能力都加 feature flag：

- `PROFILE_STARTUP`
- `PROFILE_QUERY`
- `DUMP_PROMPTS`
- `DIAGNOSTICS_FILE`

不要默认全开。

---

## 20.12 你最应该借用的核心思想

这一章最值得你复用的是下面 5 点：

1. 可观测性不是“后面再补”，而是 runtime 核心设计的一部分。
2. 必须把 debug、diagnostics、telemetry、profiler、prompt dump 分层。
3. 任何可能离开本地环境的日志都应默认 no-PII。
4. profiler 要测完整 pipeline，而不是只测模型 API。
5. 调试工具本身不能严重污染主链路性能。

把这 5 点做对，你的复现工程才会是一个“可维护的系统”，而不是只能在作者电脑上勉强运行的 demo。

## 20.13 本章对复现工程的直接指导

如果你只把 observability 当成“上线以后再补”，这类 runtime 基本都会在复杂场景下失去可维护性。建议从第一版就把最小底座做起来。

### 20.13.1 第一版先做结构化日志，不要靠散落的 console

至少统一这些字段：

- `timestamp`
- `sessionId`
- `event`
- `phase`
- `durationMs`
- `metadata`

这样后面 query、tool、agent、remote 出问题时，日志才可能被自动分析和聚合。

### 20.13.2 diagnostics 与 telemetry 要先分开

第一版就建议拆成：

- 本地 debug / diagnostics
- 可上报 telemetry

前者服务排障，后者服务产品分析。  
如果不分层，后面极容易把带敏感信息的排障数据误送到遥测链路。

### 20.13.3 profiler 要围绕主链路打点

优先打这些点：

- startup begin/end
- input normalized
- model request sent
- first token
- tool start / tool end
- final output flushed

不要只测“模型请求耗时”，那样会把大量真实瓶颈都藏起来。

### 20.13.4 prompt dump 默认关闭，但接口要先留好

因为它对定位 prompt 级问题非常有价值，但同时又最容易带来敏感数据风险。  
所以第一版就应做到：

- 有独立开关
- 默认关闭
- 明确落盘位置
- 可按 session 清理

### 20.13.5 第一版推荐目录

```text
telemetry/
  logging/
  diagnostics/
  profiler/
  prompt-dump/
  analytics/
```

这一章真正帮你建立的，不只是“怎么打日志”，而是让整个 runtime 在复杂功能逐步叠上去之后，仍然能被解释、被维护、被调试。

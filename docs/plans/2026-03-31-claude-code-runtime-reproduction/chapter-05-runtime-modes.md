# 第 5 章 运行模式设计

> 状态: 已完成初稿
> 章节目标: 明确不同交互模式的定位，以及哪些能力共享、哪些能力分叉。

[返回总览](/Users/magongli/Downloads/project/claude-code-sourcemap/docs/plans/2026-03-31-claude-code-runtime-reproduction/README.md)

---

本章的任务，是把“这个系统究竟有哪些运行形态”说清楚。终端 Agent Runtime 的一个典型陷阱是：一开始只做 REPL，后面再补 headless、SDK、远程会话，结果每加一种模式都复制一套执行链路。最终看起来像同一个产品，实际上已经变成多个系统拼在一起。

因此，本章的中心思想是：

> 模式可以不同，但执行内核必须统一。

## 5.1 为什么运行模式是架构问题

很多团队会把“REPL”“CLI 命令”“SDK”“remote”理解成产品层概念，但对于 Agent Runtime 来说，它们首先是架构问题。因为不同模式会直接影响：

- 输入从哪里来。
- 输出流向哪里去。
- 权限是交互式还是预配置式。
- 状态是否需要持久化。
- 会话是否允许中断、恢复和并发观察。
- 错误应该如何暴露给调用方。

如果这些模式没有在设计上被统一约束，常见后果包括：

- REPL 的 transcript 比 headless 更完整。
- Headless 模式没有工具回写，只返回最终文本。
- SDK 直接绕过权限逻辑。
- Remote 模式维护一套单独状态，导致 resume 行为不一致。

所以我们要做的不是“给不同模式做不同功能”，而是“给不同模式做不同适配器”。

## 5.2 Interactive REPL 模式

REPL 是这个系统最重要的使用场景，因为它最接近真实协作。用户会在这里：

- 连续提出多个问题。
- 查看工具执行过程。
- 批准或拒绝权限请求。
- 使用 slash command 操作会话。
- 恢复历史会话并继续工作。

因此，REPL 模式必须具备以下能力。

### 5.2.1 用户体验目标

- 支持流式输出，而不是等整轮结束才渲染。
- 支持显示工具调用进度和工具结果摘要。
- 支持权限中断与继续执行。
- 支持 slash command 和普通 prompt 的统一输入框。
- 支持会话内状态可见性，例如当前模型、当前目录、token 压力、后台任务状态。

### 5.2.2 对运行时的要求

REPL 不应该拥有任何“只有它能做”的核心业务逻辑。它应该只是：

- 输入适配器。
- 事件消费者。
- 权限交互界面。
- 状态可视化层。

也就是说，REPL 不应直接：

- 组装系统提示词。
- 决定工具列表。
- 直接修改 transcript 结构。
- 私自处理 compact。

这些都必须留在 runtime 核心层。

### 5.2.3 REPL 生命周期

建议的 REPL 生命周期如下：

```text
launch REPL
  -> attach session state
  -> render existing transcript
  -> wait for user input
  -> normalize input
  -> handle command or enter query
  -> stream events
  -> render tool/progress/permission states
  -> finalize turn
  -> persist session
  -> wait next input
```

这里的关键点是：REPL 是一个“长驻前端”，而不是“每次输入都重建上下文”的一次性执行器。

## 5.3 Headless / SDK 模式

Headless 模式和 SDK 模式可以视为同一能力的两个入口。

- Headless 是命令行环境里的非交互执行。
- SDK 是代码环境里的编程式调用。

它们的共同点是：

- 没有 REPL UI。
- 通常更强调结构化输入和结构化输出。
- 常用于自动化流程、脚本、CI、外部程序调用。

### 5.3.1 Headless 的设计目标

- 在无 UI 的情况下完整跑通同一套 Query Engine。
- 保留工具调用、工具结果和 transcript 语义。
- 提供结构化返回值，而不是只有一段最终文本。
- 允许调用方选择是否持久化会话。

### 5.3.2 SDK 的设计目标

- 让外部程序可以嵌入同一套运行时。
- 支持事件订阅，例如流式文本、工具调用、权限请求、状态变更。
- 支持调用方注入权限回调、日志回调和存储适配器。
- 不让 SDK 绕过核心权限、消息和状态模型。

### 5.3.3 Headless/SDK 与 REPL 的关系

它们和 REPL 的区别应该只体现在边缘层：

- 输入接口不同。
- 输出接口不同。
- 权限交互方式不同。
- 生命周期宿主不同。

但以下能力必须完全共享：

- Query Engine。
- prompt 组装。
- transcript 结构。
- 工具执行链路。
- 权限判定逻辑。
- compact 策略。

### 5.3.4 无交互权限的处理方式

Headless/SDK 最大的差异点是权限确认不一定能依赖人工交互，因此设计上必须支持：

- `strict-deny`: 无法确认时直接拒绝。
- `preapproved`: 调用方通过策略预先允许。
- `callback`: 由宿主程序提供同步或异步审批回调。
- `simulation`: 只计算计划，不实际执行副作用。

这四种模式应建立在同一权限引擎之上，而不是 headless 私自跳过 ask。

## 5.4 CLI 管理子命令模式

并不是所有 CLI 入口都会进入 Query Engine。终端 Agent 产品通常还会包含一系列管理型子命令，例如：

- 配置查看和修改。
- 会话列表与恢复。
- MCP server 管理。
- 插件列表与安装。
- 认证、诊断、版本、doctor、healthcheck。

这些命令的共同特点是：

- 更偏“控制平面”，而不是“智能执行平面”。
- 很多情况下不需要进入模型推理闭环。
- 更适合快速执行和稳定返回。

因此，CLI 子命令模式应拆成两类：

### 5.4.1 Runtime-bound Commands

这类命令仍然需要 runtime 上下文，例如：

- `/compact`
- `/review`
- `/resume`
- 与当前会话密切相关的操作

它们应通过命令系统接入，并在需要时调用 Query Engine 或 Session API。

### 5.4.2 Control-plane Commands

这类命令主要操作配置和环境，例如：

- `config`
- `mcp list`
- `plugin install`
- `doctor`

它们可以不进入 Query Engine，但依然应该复用统一的配置加载、日志、权限与错误输出基础设施。

## 5.5 远程会话 / Viewer 模式

Remote / Viewer 模式是高级能力，但必须在模式设计阶段先留出位置。

它的本质不是“多一个 UI”，而是把会话的控制权和观察权从本地单终端扩展到跨进程、跨设备、甚至跨网络环境。

### 5.5.1 Remote 模式的设计目标

- 允许某个 runtime 在远端持续运行。
- 允许本地客户端连接、观察、交互和恢复会话。
- 支持权限请求、工具状态和最终输出的跨端同步。
- 尽量复用本地 transcript、事件流和任务模型。

### 5.5.2 Viewer 模式的设计目标

- 允许只读观察会话。
- 不默认拥有控制权和权限批准权。
- 与 Remote 模式共享事件协议，但权限更弱。

### 5.5.3 为什么 Remote 不是另一个 REPL

如果把 Remote 看成“REPL over network”，很容易做出一套单独的远程 UI，然后把核心逻辑也复制过去。正确的做法应该是：

- 远端 runtime 仍是会话事实来源。
- 本地 viewer/control client 只是事件和控制消息的终端。
- transcript、task、permission 状态都以远端 session 为准。

## 5.6 各模式能力矩阵

建议在设计上给每种模式定义最小能力矩阵。

| 能力 | REPL | Headless | SDK | Remote Control | Viewer |
| --- | --- | --- | --- | --- | --- |
| 流式文本输出 | 支持 | 支持 | 支持 | 支持 | 支持 |
| 工具调用 | 支持 | 支持 | 支持 | 支持 | 只观察 |
| 权限审批 | 交互式 | 策略/回调 | 回调 | 远端转发 | 默认不支持 |
| slash command | 支持 | 可选 | 不适用 | 可转发 | 不适用 |
| session 持久化 | 默认支持 | 可选 | 可选 | 必须支持 | 依赖远端 |
| compact | 支持 | 支持 | 支持 | 支持 | 只观察 |
| 背景任务 | 支持 | 支持 | 支持 | 支持 | 只观察 |

这张表的意义，不是限制模式能力，而是强迫设计者回答：

- 某个能力在不同模式下是否存在。
- 如果存在，它的交互方式如何变化。
- 如果不存在，缺失是否会破坏一致性。

## 5.7 多模式复用策略

为了避免模式分叉，建议采用三层复用策略。

### 5.7.1 输入适配层

负责把不同入口的原始输入转成统一请求对象，例如：

- REPL 文本输入。
- CLI `--print` 参数。
- SDK 方法调用。
- Remote 控制消息。

### 5.7.2 核心执行层

负责运行统一的 Query Engine、权限引擎、工具调度和 transcript 更新逻辑。这一层不应该知道自己来自 REPL 还是 SDK。

### 5.7.3 输出适配层

负责把运行时事件转成：

- 终端渲染。
- JSON 结构化输出。
- SDK 回调事件。
- 远程传输消息。

只要这三层划分稳定，系统就可以持续增加入口而不撕裂内核。

## 5.8 常见错误路径与设计约束

模式设计里最常见的问题有四类。

第一类是 REPL 私有逻辑过多。表现为 query loop 里混入大量 UI 状态判断，导致 headless 无法复用。

第二类是 headless 结果过度简化。只返回最终文本，丢掉工具调用和 transcript 语义，后面很难做 replay 和调试。

第三类是 remote 复刻本地逻辑。这样会导致远端和本地状态出现双主问题。

第四类是权限模型分模式实现。REPL 有 ask，headless 一套 if/else，SDK 一套 callback，最终行为会悄悄不一致。

因此，本章给出的硬约束是：

- 只有输入/输出适配器允许按模式分叉。
- Query Engine、权限引擎、工具执行和 transcript 结构不得按模式分叉。

## 5.9 本章结论

运行模式的正确设计不是“每种模式都造一套系统”，而是：

- 不同入口共享一套内核。
- 不同宿主消费同一套事件和状态。
- 不同交互方式复用同一权限与工具链路。

## 5.10 本章对复现工程的直接指导

如果你要避免后面模式撕裂，建议一开始就定义 4 个适配器接口。

### 5.10.1 InputAdapter

负责把 REPL/CLI/SDK/remote 输入变成统一的 `NormalizedInput`。

### 5.10.2 OutputSink

负责消费统一事件流，把它渲染到：

- 终端
- JSON stream
- 日志
- remote viewer

### 5.10.3 PermissionMediator

不同模式下权限来源不同，但接口应统一：

- REPL 弹框
- headless callback/config
- SDK hook
- remote relay

### 5.10.4 SessionHost

不同模式都应围绕同一个 session/query engine 运行，而不是自己偷偷持有会话状态。

### 5.10.5 第一版就要测试的模式一致性

至少验证：

- REPL 和 headless 的 tool_result 写回一致
- REPL 和 SDK 的权限流一致
- resume 后模式语义不乱

这样后面加新模式时，工作量会小很多。

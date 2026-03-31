# Claude Code 风格 Agent Runtime 复现设计文档

> 目标: 复现一个“Claude Code 风格”的终端 Agent Runtime，优先复现其架构分层、运行链路、核心抽象和系统设计思想，而不是逐文件 1:1 还原 Anthropic 内部工程。

**当前状态:** 文档第 1 版，先固化完整设计大纲，后续按章节逐步展开正文。  
**上游研究基础:** [Claude Code Sourcemap 架构拆解与学习指南](/Users/magongli/Downloads/project/claude-code-sourcemap/docs/plans/2026-03-31-claude-code-architecture-guide.md)

---

## 0. 文档使用说明

这份文档不是源码分析，而是“面向复现”的设计文档。

它回答的问题是：

- 如果要自己做一个 Claude Code 风格系统，应该如何分层
- 哪些能力应该先做，哪些能力应该延后
- 核心抽象如何定义
- 不同子系统之间如何协作
- 后续实现时应该如何按阶段推进

文档的组织方式采用：

1. 先给出完整大纲
2. 每章标出设计目标
3. 后续按章节逐步补全详解内容

换句话说，这份文档现在是“总设计框架”，后面会逐章生长为“完整设计说明书”。

---

## 1. 设计目标与边界

### 1.1 总目标

设计并实现一个具备以下特征的终端 Agent Runtime：

- 有统一的 CLI 入口
- 同时支持交互式 REPL 和 Headless / SDK 模式
- 有清晰的消息模型、命令模型、工具模型和会话状态模型
- 支持工具调用、权限控制、上下文压缩、会话持久化
- 支持技能、插件、MCP 等扩展机制
- 能逐步演进到多 Agent 和 Remote/Bridge 架构

### 1.2 复现原则

- 复现“架构思想”和“运行机制”，不追求 bundle 级逐行复刻
- 优先构建可维护、可理解、可扩展的 clean-room 实现
- 先做统一运行时，再叠加高级能力
- 所有高级能力都必须建立在稳定的核心抽象之上

### 1.3 非目标

当前阶段不追求：

- 完整复刻 Anthropic 的私有后端协议
- 复刻所有隐藏 feature gate
- 完整复刻所有 UI 细节和产品文案
- 一次性实现 marketplace、远程桥接、多 Agent 全部能力

---

## 2. 总设计文档大纲

下面是这份“复现设计文档”的正式章节结构。后续我们会按这些章节一步步完成详解。

### 第 1 章 引言

**目标:** 说明项目为什么做、做到什么程度、给谁看、用什么术语。

- 1.1 项目背景
- 1.2 复现目标
- 1.3 设计范围
- 1.4 非目标与约束
- 1.5 目标读者
- 1.6 术语与文档约定

### 第 2 章 问题定义与设计原则

**目标:** 明确我们到底在解决什么问题，以及设计上坚持哪些原则。

- 2.1 终端 Agent 产品的核心问题
- 2.2 Claude Code 风格系统的关键特征
- 2.3 本项目要解决的核心技术问题
- 2.4 设计原则
- 2.5 架构取舍与边界

### 第 3 章 产品分层与版本路线

**目标:** 把系统拆成可落地阶段，避免一开始范围失控。

- 3.1 MVP 范围
- 3.2 增强版范围
- 3.3 完整版范围
- 3.4 每个阶段的验收标准
- 3.5 当前实现优先级

### 第 4 章 系统总览

**目标:** 给出全局架构图，让后面每章都有锚点。

- 4.1 系统整体架构图
- 4.2 核心模块地图
- 4.3 主要数据流
- 4.4 主要控制流
- 4.5 运行边界与信任边界

### 第 5 章 运行模式设计

**目标:** 明确不同交互模式的定位，以及哪些能力共享、哪些能力分叉。

- 5.1 Interactive REPL 模式
- 5.2 Headless / SDK 模式
- 5.3 CLI 管理子命令模式
- 5.4 远程会话 / Viewer 模式
- 5.5 多模式复用策略

### 第 6 章 启动与初始化设计

**目标:** 明确从命令行启动到进入会话的全链路。

- 6.1 CLI 入口与参数解析
- 6.2 启动编排器设计
- 6.3 `init` 与 `setup` 的职责划分
- 6.4 配置加载时序
- 6.5 trust 前与 trust 后的安全边界
- 6.6 懒加载与启动性能策略

### 第 7 章 核心领域模型

**目标:** 把后续所有实现都建立在稳定的类型和概念上。

- 7.1 `Message`
- 7.2 `Command`
- 7.3 `Tool`
- 7.4 `ToolUseContext`
- 7.5 `AppState`
- 7.6 `Session`
- 7.7 `Task`
- 7.8 `AgentDefinition`
- 7.9 `PermissionRule`
- 7.10 `MCPServer`

### 第 8 章 会话与状态管理设计

**目标:** 让系统具备长期运行能力，而不是一次性脚本式调用。

- 8.1 状态分层原则
- 8.2 AppState 结构设计
- 8.3 消息历史与 transcript
- 8.4 文件状态缓存
- 8.5 file history 与 snapshot
- 8.6 session persistence
- 8.7 resume / fork / rewind 机制
- 8.8 状态变更传播机制

### 第 9 章 用户输入处理设计

**目标:** 规范“输入是如何进入系统”的。

- 9.1 prompt 输入模型
- 9.2 slash command 解析
- 9.3 pasted content / attachment
- 9.4 hooks 介入点
- 9.5 输入标准化流水线
- 9.6 输入到 query 的边界

### 第 10 章 System Prompt 与上下文拼装设计

**目标:** 定义 system prompt 不是常量，而是一个可组合系统。

- 10.1 默认 system prompt 结构
- 10.2 custom / append prompt 机制
- 10.3 agent / coordinator prompt 叠加规则
- 10.4 user context / system context 设计
- 10.5 prompt cache 边界设计
- 10.6 不同模式下的 prompt 变化

### 第 11 章 Query Engine 设计

**目标:** 定义整个 Agent Runtime 的核心执行闭环。

- 11.1 一轮 query 的生命周期
- 11.2 QueryEngine 职责边界
- 11.3 message -> model -> tool -> message 回环
- 11.4 流式输出处理
- 11.5 interrupt / retry / recovery
- 11.6 usage / cost / budget
- 11.7 REPL 与 Headless 的复用方式

### 第 12 章 Tool 系统设计

**目标:** 把“模型可调用能力”标准化。

- 12.1 Tool 抽象定义
- 12.2 tool schema 与类型系统
- 12.3 tool registry 与装配方式
- 12.4 built-in tool 分类
- 12.5 工具可见性与能力裁剪
- 12.6 并发安全与串行执行
- 12.7 tool result 记录与回写

### 第 13 章 Command / Skill 系统设计

**目标:** 定义用户入口和模型可触发能力的统一抽象。

- 13.1 `prompt` / `local` / `local-jsx` 三类命令
- 13.2 command registry
- 13.3 skill 的 markdown/frontmatter 模型
- 13.4 dynamic skill 注入
- 13.5 remote-safe / bridge-safe command 策略

### 第 14 章 权限、信任与沙箱设计

**目标:** 把安全边界做成架构的一部分，而不是补丁。

- 14.1 workspace trust 模型
- 14.2 permission mode 模型
- 14.3 allow / deny / ask 规则系统
- 14.4 权限请求生命周期
- 14.5 classifier / auto mode 的位置
- 14.6 sandbox 集成
- 14.7 安全审计与风险边界

### 第 15 章 上下文治理与 Compact 设计

**目标:** 让系统在长会话中仍然稳定运行。

- 15.1 token 估算
- 15.2 warning / blocking / auto-compact 阈值
- 15.3 auto compact
- 15.4 reactive compact
- 15.5 compact 后恢复与 reinjection
- 15.6 memory / summary / compact 的协同关系

### 第 16 章 MCP 集成设计

**目标:** 为系统引入协议级扩展平面。

- 16.1 MCP 配置来源
- 16.2 transport 设计
- 16.3 auth 与 reconnect
- 16.4 tool / command / resource / prompt 接入方式
- 16.5 去重与优先级策略
- 16.6 MCP 与主工具池的融合

### 第 17 章 Plugin 系统设计

**目标:** 引入本地扩展包能力。

- 17.1 plugin manifest 设计
- 17.2 plugin 装载流程
- 17.3 插件可贡献能力类型
- 17.4 本地插件与 marketplace 插件
- 17.5 更新、缓存与版本策略
- 17.6 插件安全与策略限制

### 第 18 章 Agent 与多 Agent 设计

**目标:** 把“子 Agent”从 feature 提升为正式运行时能力。

- 18.1 AgentDefinition 模型
- 18.2 `AgentTool` 设计
- 18.3 sync / background agent
- 18.4 task 生命周期
- 18.5 coordinator mode
- 18.6 worker 能力裁切
- 18.7 isolation: worktree / remote
- 18.8 agent 间通知与消息模型

### 第 19 章 Remote / Bridge 设计

**目标:** 支持跨端、远程和 viewer 模式会话。

- 19.1 本地会话与远程会话关系
- 19.2 transport 模型
- 19.3 control message 与 permission relay
- 19.4 reconnect / token refresh / continuity
- 19.5 viewer 模式
- 19.6 与 REPL / SDK 的集成边界

### 第 20 章 可观测性与诊断设计

**目标:** 让系统可分析、可调优、可定位问题。

- 20.1 logging 分层
- 20.2 diagnostics
- 20.3 analytics / telemetry
- 20.4 startup profiler
- 20.5 query profiler
- 20.6 故障定位思路

### 第 21 章 测试策略

**目标:** 让运行时具备可回归、可验证、可演进能力。

- 21.1 单元测试
- 21.2 集成测试
- 21.3 E2E 测试
- 21.4 transcript replay
- 21.5 tool contract 测试
- 21.6 MCP 集成测试
- 21.7 性能与稳定性测试

### 第 22 章 安全设计

**目标:** 把风险模型提前梳理清楚。

- 22.1 威胁模型
- 22.2 prompt injection 风险
- 22.3 shell / file / network 风险
- 22.4 插件与 MCP 供应链风险
- 22.5 secret 管理
- 22.6 remote 会话风险

### 第 23 章 实现路线图

**目标:** 把设计文档变成可以执行的工程阶段。

- 23.1 Phase 0: 核心骨架
- 23.2 Phase 1: MVP runtime
- 23.3 Phase 2: commands / skills / compact / persistence
- 23.4 Phase 3: MCP / plugin
- 23.5 Phase 4: multi-agent / coordinator
- 23.6 Phase 5: remote / bridge
- 23.7 每阶段交付物与验收标准

### 第 24 章 开放问题与待决策项

**目标:** 把暂未拍板的问题收口，避免实现中途反复返工。

- 24.1 平台支持优先级
- 24.2 存储后端选型
- 24.3 模型提供商抽象层深度
- 24.4 plugin marketplace 签名与信任策略
- 24.5 remote 优先做 viewer 还是完整双向控制
- 24.6 compact 与 memory 的最终职责边界

### 第 25 章 附录

**目标:** 存放实现时高频查阅的辅助内容。

- 25.1 术语表
- 25.2 模块映射表
- 25.3 核心时序图清单
- 25.4 状态机清单
- 25.5 配置项矩阵
- 25.6 风险清单

---

## 3. 写作策略与章节推进顺序

为了避免文档一开始就写散，建议按下面顺序补完。

### 第一批: 定全局边界

优先完成这些章节：

- 第 1 章 引言
- 第 2 章 问题定义与设计原则
- 第 3 章 产品分层与版本路线
- 第 4 章 系统总览
- 第 5 章 运行模式设计
- 第 6 章 启动与初始化设计

这一批的目标不是“写细节”，而是把项目边界和总架构定死。

### 第二批: 定核心抽象与主运行链

然后完成：

- 第 7 章 核心领域模型
- 第 8 章 会话与状态管理设计
- 第 9 章 用户输入处理设计
- 第 10 章 System Prompt 与上下文拼装设计
- 第 11 章 Query Engine 设计
- 第 12 章 Tool 系统设计

这一批完成之后，整个系统的内核就基本闭环了。

### 第三批: 定扩展平面

再完成：

- 第 13 章 Command / Skill 系统设计
- 第 14 章 权限、信任与沙箱设计
- 第 15 章 上下文治理与 Compact 设计
- 第 16 章 MCP 集成设计
- 第 17 章 Plugin 系统设计

这一批会决定系统是否能长期演进。

### 第四批: 定高级能力

最后完成：

- 第 18 章 Agent 与多 Agent 设计
- 第 19 章 Remote / Bridge 设计
- 第 20 章 可观测性与诊断设计
- 第 21 章 测试策略
- 第 22 章 安全设计
- 第 23 章 实现路线图
- 第 24 章 开放问题与待决策项
- 第 25 章 附录

---

## 4. 本文档后续维护方式

后续建议每一轮只扩一章或一组强相关章节，避免一次性把所有内容写满导致结构失控。

推荐的推进方式：

1. 先完成第 1 批全局章节
2. 再完成第 2 批内核章节
3. 每完成一章，就补：
   - 目标
   - 设计约束
   - 模块划分
   - 数据流
   - 错误路径
   - 测试点
4. 每一章都尽量落到“实现时可直接据此拆模块”的粒度

---

## 5. 下一步建议

按照上面的顺序，最适合先写的是：

- 第 1 章 引言
- 第 2 章 问题定义与设计原则
- 第 3 章 产品分层与版本路线
- 第 4 章 系统总览

这四章写完之后，整个复现项目的目标、范围、边界、阶段和系统地图都会非常清楚，后面再进入 QueryEngine、Tool、Permission、MCP 才不会跑偏。

---

## 6. 当前结论

这份文档当前已经完成了两件事：

1. 固化了“Claude Code 风格 Agent Runtime”的完整设计目录
2. 确定了后续按章节逐步完成详解的写作顺序

后续可以直接从第 1 章开始逐章扩写。

---

## 7. 第 1 章 引言（正文）

本章的目标，是把这份设计文档的立场、边界和读法先说清楚。因为 Claude Code 风格工程最大的迷惑性在于，它表面上像“一个终端聊天工具”，但真正复现时你会发现，它本质上是一个长期运行、可扩展、具备安全边界和状态治理能力的 Agent Runtime。如果一开始不把这个判断定死，后面很容易错误地把项目做成“命令行包一层 LLM API”的薄壳。

### 7.1 项目背景

过去几年里，终端 AI 工具大致经历了三个阶段：

- 第一阶段是“问答助手”，本质是把聊天模型放进 CLI。
- 第二阶段是“工具增强助手”，开始具备读文件、改文件、执行命令等能力。
- 第三阶段才是“运行时化的 Agent”，其重点不再是聊天，而是任务执行、状态管理、权限治理、可扩展能力和长会话稳定性。

Claude Code 风格系统属于第三阶段。它和早期 CLI 助手的根本区别不是 UI，而是运行时组织方式：

- 它有统一入口，但不只有一种交互模式。
- 它把消息、命令、工具、权限、上下文压缩、扩展协议都做成正式抽象。
- 它能够让同一个执行内核同时服务 REPL、headless、SDK、远程会话和多 Agent 场景。
- 它把“长时间协作”当成主场景，而不是只优化单轮问答。

因此，本项目不是在复刻一个终端界面，而是在设计一个“面向终端与自动化协作的通用 Agent Runtime”。

### 7.2 复现目标

本项目的复现目标可以概括成一句话：

> 用 clean-room 的方式，复现 Claude Code 风格系统的核心架构、运行链路和设计思想，得到一个可维护、可扩展、可持续演进的终端 Agent Runtime。

更具体地说，目标包括：

- 复现统一的启动与运行模型，让 CLI、REPL、Headless 共享同一执行内核。
- 复现稳定的核心抽象，包括 `Message`、`Command`、`Tool`、`ToolUseContext`、`AppState`、`Session`、`AgentDefinition` 等。
- 复现完整的一轮 Agent 闭环，也就是 `用户输入 -> 上下文拼装 -> 模型推理 -> 工具调用 -> 权限判定 -> 工具结果回写 -> 最终输出`。
- 复现长会话运行能力，包括 transcript、token 估算、compact、resume、持久化。
- 复现扩展平面，包括命令、技能、MCP、插件，并为多 Agent 和 remote 预留正式接口。
- 复现其最值得借鉴的工程思想，而不是只复刻对外可见功能。

这里强调“复现思想”而不是“复刻实现”，是因为真正能迁移复用的价值不在 bundle 内部某个具体函数，而在以下这些原则是否被吃透：

- 为什么 `Command` 和 `Tool` 必须分开。
- 为什么权限必须在工具真正执行前介入。
- 为什么 compact 不是附属 feature，而是 runtime 核心能力。
- 为什么 REPL 和 headless 必须共用一个 Query Engine。
- 为什么 MCP、插件、agent、remote 都应该建立在稳定内核之上，而不是各自直连模型。

### 7.3 设计范围

本设计文档的范围分成三个层次。

第一层是必须落地的核心内核：

- CLI 启动入口与模式分发。
- 交互式 REPL。
- Headless / SDK 模式。
- Query Engine。
- System Prompt 与上下文拼装。
- 工具注册、工具执行与工具回写。
- 权限模型。
- 会话状态与 transcript。
- compact 与会话持久化。

第二层是重要扩展层：

- Slash command 与本地命令系统。
- Skill 机制。
- MCP server 接入。
- 插件装载与能力注入。
- 基础可观测性与调试能力。

第三层是高级演进层：

- AgentTool 与多 Agent 运行时。
- coordinator 模式。
- remote / bridge / viewer。
- 更强的会话协同和远程任务编排。

这份设计文档会覆盖三层，但实现优先级会严格按照“核心内核 -> 扩展层 -> 高级层”推进。

### 7.4 非目标与约束

为了避免范围失控，这里先明确非目标。

本项目当前不追求：

- 逐文件 1:1 复刻上游工程的全部源代码结构。
- 复刻 Anthropic 私有后端协议、内部 feature flag、账户体系和云端服务。
- 先于运行时内核去追求 UI 的精细复刻。
- 在第一阶段就实现完整 marketplace、remote bridge、多 Agent 协作、企业级策略系统。
- 一开始就支持所有操作系统、所有模型供应商和所有 transport。

本项目还存在一些现实约束：

- 需要优先保证本地可运行、可调试、可读懂。
- 需要优先使用 TypeScript/Node.js 生态，降低复现门槛。
- 需要把复杂度集中在运行时抽象，而不是早期基础设施堆砌。
- 需要尽量采用 clean-room 设计，不直接把反编译产物当生产级源码使用。

这些约束并不是妥协，而是有意识的设计取舍。Claude Code 风格工程最容易让人误判的地方，就是会让人觉得“功能越多越像”。实际上，越早控制范围，越能更快复现出真正有价值的系统骨架。

### 7.5 目标读者

这份文档主要写给三类读者：

- 想复现一套终端 Agent Runtime 的工程师。
- 想从 Claude Code 风格系统里抽取架构思想的人。
- 想把这类设计迁移到自己产品中的技术负责人。

因此，文档不会把重点放在“如何使用某个命令”上，而会放在：

- 核心模块如何分层。
- 主链路如何闭环。
- 扩展点如何设计。
- 长会话为什么容易失控，以及如何治理。
- 多入口、多模式、多能力如何复用一个内核。

如果读者的目标只是“做一个能在终端里调 LLM 的工具”，那这份文档会显得过重；但如果目标是做一个可以演进到 IDE、远程协作、多 Agent 的产品，这份文档会提供必要的系统视角。

### 7.6 术语与文档约定

为了避免后续章节出现概念混淆，先约定术语。

- `Runtime`: 指整个终端 Agent 运行时系统，不等同于 UI，也不等同于模型 SDK。
- `Session`: 一段连续会话，包含 transcript、状态、权限上下文、持久化信息。
- `Turn`: 会话中的一次用户请求及其完整处理过程。
- `Query`: 一次进入模型执行闭环的运行单元，可能包含多轮工具调用。
- `Message`: 进入 transcript 的标准消息单元。
- `Command`: 用户主动触发的本地入口能力，通常通过 slash command 或 CLI 子命令进入。
- `Tool`: 暴露给模型调用的受控执行能力。
- `ToolUseContext`: 工具执行所需的上下文，包括权限、工作目录、状态、日志接口等。
- `Skill`: 基于 markdown/frontmatter 或约定目录装载的轻量扩展单元。
- `Plugin`: 以 manifest 为中心、可贡献命令、工具、资源或配置的本地扩展包。
- `MCP`: 通过 Model Context Protocol 接入的外部能力平面。
- `Agent`: 在主会话之外，以任务形式执行某个子目标的运行单元。
- `Compact`: 对长会话上下文进行压缩和恢复的运行时能力。

文档约定如下：

- 设计文档里的“建议”默认指推荐实现方式，而不是唯一实现方式。
- 当文档中出现“必须”时，表示该约束是为了保证系统骨架稳定。
- 目录结构、模块名、接口名会尽量采用接近上游抽象但不依赖上游实现的命名。
- 如果后续章节与本章表述冲突，以后续更细粒度章节为准，但必须在该章节中显式说明取舍原因。

### 7.7 本章结论

本章最重要的结论只有两个：

- 我们要复现的是一个 Agent Runtime，而不是一个聊天 CLI。
- 后续所有设计，都必须围绕“统一执行内核、稳定抽象、长会话治理、扩展优先”这四个中心来展开。

---

## 8. 第 2 章 问题定义与设计原则（正文）

本章回答两个问题：

- 我们到底在解决什么问题。
- 我们打算用什么原则解决。

如果第 1 章确定的是“做什么”，那第 2 章确定的就是“为什么这样做”。

### 8.1 终端 Agent 产品的核心问题

终端 Agent 产品之所以难，不是因为它要调用 LLM，而是因为它要同时解决下面几类问题。

第一类是执行闭环问题。

系统不能只会产出文本，还要能让模型安全地读文件、改文件、执行命令、分析结果，并把这些结果重新送回会话，让下一步推理继续发生。也就是说，它必须是一个可回环的执行系统，而不是单次推理系统。

第二类是状态问题。

真实使用里，用户不会只问一个问题，而是会连续工作几十分钟甚至几小时。系统必须记住：

- 聊天历史是什么。
- 改过哪些文件。
- 当前工作目录是什么。
- 权限状态是什么。
- 上一次 compact 发生在什么时候。
- 是否有任务在后台运行。

如果这些状态没有正式建模，产品很快就会退化成“一次次无状态调用”的不稳定工具。

第三类是安全与权限问题。

一旦模型具备 shell、文件系统、网络等能力，系统就不再只是对话产品，而是一个能执行副作用的自动化系统。此时必须回答：

- 什么工具可以被看到。
- 什么参数可以被执行。
- 什么时候需要 ask。
- 什么情况下可以自动放行。
- 工作区是否可信。
- 插件和 MCP 能否直接穿透本地权限模型。

如果权限只是 UI 层一个确认框，那么架构上已经晚了。

第四类是长会话上下文问题。

终端任务天然容易变长。随着 transcript 增长，系统会遇到：

- token 成本上涨。
- prompt cache 命中率下降。
- 关键信息被历史噪音淹没。
- 模型开始遗忘当前目标和约束。

所以 compact 不是锦上添花，而是长会话系统能否稳定运行的生命线。

第五类是多模式复用问题。

同一套能力通常需要同时服务：

- 终端交互用户。
- 批处理或脚本。
- API/SDK 调用方。
- 远程协作或 viewer 模式。

如果 REPL、headless、SDK、remote 都各自维护一套执行逻辑，系统会迅速分叉，测试与维护成本都会失控。

第六类是扩展平面问题。

一个好用的终端 Agent 不可能只依赖内置能力。它必须允许外部扩展以正式接口接入，包括：

- 本地命令。
- skills。
- plugins。
- MCP servers。
- 子 agents。

问题在于，这些扩展必须加入统一调度和安全边界，而不能各自直连模型执行。

### 8.2 Claude Code 风格系统的关键特征

从架构视角看，Claude Code 风格系统最有价值的不是“功能多”，而是它把复杂能力组织成了稳定层次。

其关键特征可以总结为以下几点。

第一，统一运行时。

用户无论从 REPL、命令行参数、headless 调用还是远程桥接进入，最终都走向同一套查询执行内核。交互层只是入口，真正的业务闭环发生在 runtime 内部。

第二，命令与工具分离。

`Command` 面向用户入口，解决“用户怎么触发能力”；`Tool` 面向模型调用，解决“模型如何受控执行能力”。二者看起来相似，但职责完全不同。这个分离是整个系统可维护性的关键。

第三，权限前置。

权限不是工具执行之后的补救，而是在工具装配、工具可见性、参数审查、执行判定这些阶段就参与决策。模型并不是默认拥有全部能力，而是在当前上下文中被裁切到“可见且可执行”的能力集合。

第四，上下文治理内建。

系统默认假设会话会变长，因此会提前准备 token 估算、compact 阈值、summary reinjection、状态恢复等机制。它不是等上下文爆炸了再补一个压缩按钮。

第五，多层扩展体系。

Claude Code 风格系统并没有只提供一种扩展方式，而是按职责拆出了 command、skill、plugin、MCP、agent 这些平面。不同扩展面解决的问题不同，因此才能避免一个“万能插件接口”变成混乱中心。

第六，任务化的 Agent 观。

子 Agent 不是一个神秘魔法，而是任务系统的一部分。它有定义、有生命周期、有资源边界、有回传机制。也正因为如此，它可以演进成 background task、coordinator、remote worker 等高级能力。

第七，性能与缓存意识。

终端 Agent 一旦进入真实项目，启动频率和会话长度都会很高，因此系统必须关心：

- 启动时加载多少内容。
- 什么能力懒加载。
- prompt 哪些部分可缓存。
- 什么消息结构对 cache 更友好。

这也是为什么“系统提示词和上下文拼装”会成为单独设计主题。

### 8.3 本项目要解决的核心技术问题

基于上面的判断，本项目至少要解决以下技术问题。

#### 8.3.1 统一入口与统一执行内核

我们需要一个系统，使得：

- CLI 子命令可以调用内核。
- REPL 可以调用内核。
- Headless/SDK 可以调用内核。
- 将来 remote/viewer 也可以调用内核。

这里的关键不是共用一套库，而是共用“同一个查询状态机”。

#### 8.3.2 稳定的消息与状态模型

我们需要正式定义：

- 什么消息可以进入 transcript。
- 工具调用和工具结果如何编码。
- UI 状态与会话状态怎么分层。
- 运行时哪些状态必须可持久化。

没有这些正式模型，后续 compact、resume、replay、调试都会变得脆弱。

#### 8.3.3 受控的工具系统

工具系统要同时满足三件事：

- 对模型来说，它是稳定、可推理、可调用的能力集合。
- 对系统来说，它是可审查、可限权、可回放、可记录的执行单元。
- 对开发者来说，它是可扩展、可注册、可测试的模块接口。

这意味着我们需要的是 `tool registry + tool schema + tool execution context + policy layer`，而不是简单的“if tool_name then run function”。

#### 8.3.4 长会话治理

系统必须从第一版就面对长会话问题。至少要具备：

- token 使用估算。
- warning 与 blocking 阈值。
- compact 触发策略。
- compact 结果回注机制。
- 会话恢复和摘要保真策略。

否则只要用户连续工作一段时间，系统就会进入高成本、低准确、低可控状态。

#### 8.3.5 安全与信任边界

系统要明确哪些输入和扩展是不可信的，至少包括：

- 用户 prompt。
- 当前工作区内容。
- 第三方插件。
- MCP server 响应。
- 网络命令和 shell 输出。

权限系统必须横切这些能力面，而不是只盯着“本地工具”。

#### 8.3.6 可演进的扩展体系

如果 command、skill、plugin、MCP、agent 没有统一接入原则，系统越做越大只会越碎。所以本项目要在第一版就定义：

- 哪些能力属于用户入口。
- 哪些能力属于模型可调用面。
- 哪些扩展会进入本地信任边界。
- 哪些扩展必须通过沙箱或额外权限审查。

### 8.4 设计原则

下面这些原则会贯穿整份设计文档。

#### 8.4.1 Runtime First

先设计运行时，再设计交互壳。UI、CLI 参数、viewer 都只是入口层，不应该反过来决定核心执行模型。

#### 8.4.2 抽象稳定优先于功能数量

宁可少做几个 feature，也要先把 `Message`、`Tool`、`Command`、`AppState`、`Session` 这些抽象做稳。Claude Code 风格系统最值得复现的地方，正是它先稳定了这些中枢接口。

#### 8.4.3 权限是执行前条件，不是执行后提示

任何具有副作用的能力，都必须在“可见性、可执行性、参数合法性”三个层面受到控制。权限框架应该是调度链路的一部分，而不是 UI 补丁。

#### 8.4.4 长会话是默认场景

系统默认会话会很长，因此 compact、summary、token 预算、持久化从一开始就是内建能力。

#### 8.4.5 多入口必须共用一套主链路

REPL、headless、SDK、remote 的区别只应存在于输入输出适配层，而不应存在于查询引擎本身。否则工具行为、权限行为和 transcript 行为会在不同入口下悄悄分叉。

#### 8.4.6 扩展必须纳入统一治理

command、skill、plugin、MCP、agent 都可以扩展系统，但不能绕过统一的注册、权限、日志、状态与调试链路。扩展能力越强，越要在统一抽象下运行。

#### 8.4.7 Prompt Cache 友好

系统提示词、上下文顺序、消息结构都要考虑缓存稳定性。一个“经常变动的 prompt 拼装器”会在成本和延迟上持续惩罚整个产品。

#### 8.4.8 渐进增强，而不是一口吃成胖子

MVP 要先证明“一轮 Agent 闭环”可以稳定工作，然后再叠加 compact、MCP、plugins、agents、remote。复杂能力应建立在可验证的内核之上。

#### 8.4.9 可诊断性内建

系统需要天然支持日志、事件、状态快照和 replay。Agent Runtime 出问题时，如果只能靠肉眼看终端输出排查，后续调试成本会迅速超过开发成本。

### 8.5 架构取舍与边界

为了让这些原则落地，本项目在架构上做出以下明确取舍。

#### 8.5.1 单进程优先，分布式延后

MVP 阶段优先做本地单进程运行时，把状态机、工具、权限和 transcript 做稳定。remote、bridge、多 worker 分布式编排留到后期。

#### 8.5.2 本地 JSON/文件持久化优先，数据库延后

在终端工具的早期阶段，JSONL 或结构化本地文件已经足够覆盖 transcript、session metadata、compact summary、task snapshot。引入数据库太早，会稀释对运行时核心问题的注意力。

#### 8.5.3 模型适配层保持薄而明确

系统可以支持多模型提供商，但第一版不要让 provider abstraction 反客为主。我们要抽象的是“query 所需能力”，而不是过度抽象所有供应商差异。

#### 8.5.4 UI 轻量，状态中枢重

REPL 应该只是状态与事件的消费层，而不是业务编排中心。尽量避免把权限弹框、工具执行逻辑、compact 逻辑直接塞进 UI 组件。

#### 8.5.5 插件与协议扩展都走统一入口

无论是本地插件还是 MCP server，都应该被装配到统一命令池、工具池、资源池或 prompt 池，而不是“各自有一条私有执行通道”。

### 8.6 本章结论

本章的核心判断可以压缩成一句话：

> 终端 Agent 难的不是“把模型接进 CLI”，而是“把副作用执行、长会话治理、多入口复用和扩展系统同时收进一个稳定运行时里”。

接下来的所有章节，都会围绕这个判断展开。

---

## 9. 第 3 章 产品分层与版本路线（正文）

本章的任务，是把一个潜在上限很高、也很容易失控的系统，拆成可以分阶段落地的工程路线。Claude Code 风格系统的复杂度很高，但它并不要求我们第一天就把所有能力做齐。相反，只有先把骨架做对，后面的高级能力才不会变成技术债。

### 9.1 MVP 范围

MVP 阶段的目标不是“做出一个看起来很像 Claude Code 的产品”，而是“做出一个真正具备 Agent 闭环的统一运行时”。

MVP 必须包含以下能力：

- 一个统一的 CLI 入口。
- 交互式 REPL。
- Headless 单次执行模式。
- 基础 Query Engine。
- 最小消息模型与 transcript。
- 最小工具集，建议至少包含 `read`、`write/edit`、`bash`、`search/find` 这类核心能力。
- 基础权限判定，支持 `allow`、`deny`、`ask`。
- 基础会话持久化。
- 基础 token 估算和手动 compact 能力。

MVP 阶段刻意不做或只做占位的能力：

- 多 Agent。
- marketplace 级插件系统。
- remote / bridge。
- 复杂企业策略。
- 丰富的可视化界面。

MVP 的核心验收问题只有两个：

- 用户是否可以在一个真实代码仓中连续完成一段小型任务。
- REPL 与 headless 是否在同一个执行内核上表现一致。

如果这两个问题都答不上来，说明内核还不稳定，不应该继续堆高级能力。

### 9.2 增强版范围

增强版阶段的目标，是让 MVP 从“可用的内核”进化成“可长期使用的工具”。

增强版建议加入：

- 更完整的 slash command 系统。
- skill 发现、加载和注入机制。
- 自动 compact 与 compact 恢复策略。
- 更完整的 session persistence。
- resume / fork / rewind 等会话能力。
- MCP 基础接入，优先支持 `stdio` transport。
- 更完整的权限规则与白名单策略。
- 基础日志、诊断与 transcript replay 能力。

增强版的重点不在“加更多工具”，而在“让系统开始具备长期演进的基本秩序”。在这个阶段，最值得投入的是：

- 命令系统是否清晰。
- transcript 是否可调试。
- compact 是否稳定。
- MCP 是否能纳入主工具池而不破坏权限模型。

### 9.3 完整版范围

完整版不是“功能全开”，而是“高级能力已经建立在稳定内核上”。

完整版建议覆盖：

- AgentTool 与正式的多 Agent 任务体系。
- background task 与 coordinator 模式。
- plugin manifest 与本地插件装载体系。
- remote / bridge / viewer。
- 更细粒度的权限自动分类与沙箱策略。
- 更完整的观测性、性能 profiling 与故障诊断。
- 更成熟的测试矩阵和回放系统。

进入完整版阶段之后，项目的重心会从“把能力做出来”切换成“如何让能力互相不打架”。因为一旦有了插件、MCP、agent、remote，系统复杂度将不再线性增长，而是进入多平面耦合状态。

### 9.4 每个阶段的验收标准

下面给出建议验收标准。

#### 9.4.1 MVP 验收

- 能启动 REPL，并连续完成至少一个真实仓库内的小任务。
- 能通过 headless 模式执行单轮或多步工具调用任务。
- 工具调用可被记录进 transcript，并能被后续 query 继续使用。
- 权限询问和放行逻辑可稳定工作。
- 会话可以保存并重新打开。
- 失败场景下不会导致 transcript 或状态完全损坏。

#### 9.4.2 增强版验收

- Slash command、skills、MCP 可以共同接入而不产生明显职责冲突。
- compact 可以自动触发，并在触发后保持关键上下文连续性。
- 用户可以恢复旧会话并继续工作。
- 基础 replay 和日志足以定位常见问题。
- 能在中等长度会话中保持可接受的延迟和成本。

#### 9.4.3 完整版验收

- 子 Agent 可以被定义、启动、跟踪和回收。
- 远程会话与本地 REPL 之间职责清晰，状态可同步。
- 插件和 MCP 不会绕过统一权限和审计机制。
- 系统在多扩展、多会话、多任务场景下仍可定位问题。
- 关键运行链路具备较完整的集成测试与回放样本。

### 9.5 当前实现优先级

如果以“尽快得到一个可复用内核”为目标，建议实现顺序如下：

1. 启动与入口分发。
2. Query Engine 与消息模型。
3. Tool registry、ToolUseContext、最小工具集。
4. 权限模型。
5. REPL 与 headless 共用执行内核。
6. transcript、session persistence、token 估算。
7. compact。
8. slash command 与 skill。
9. MCP。
10. plugin。
11. agent。
12. remote / bridge。

这个顺序的原因很简单：

- 没有 Query Engine，其他能力都只是挂件。
- 没有工具和权限，系统无法真正执行任务。
- 没有状态和 compact，系统无法支撑真实工作流。
- 没有稳定内核，MCP、plugin、agent 只会把问题放大。

### 9.6 建议的里程碑拆分

为了便于工程推进，可以把路线图再细化成以下里程碑。

#### 里程碑 A: 跑通最小闭环

- CLI 入口
- headless query
- transcript
- `read` / `bash` 两个最小工具

目标是证明“模型可调用工具并回写结果”。

#### 里程碑 B: 进入真实 REPL 使用

- REPL UI
- `write/edit`
- 权限询问
- session 保存

目标是证明“用户可在终端里持续协作完成任务”。

#### 里程碑 C: 长会话稳定化

- token 估算
- compact
- resume
- replay

目标是证明“系统能在长会话中继续工作”。

#### 里程碑 D: 扩展平面成型

- slash command
- skills
- MCP

目标是证明“系统不是一组内置功能，而是一个可扩展运行时”。

#### 里程碑 E: 高级运行时能力

- plugins
- agents
- remote

目标是把产品从单机会话工具推进到协作型平台雏形。

### 9.7 本章结论

本章最重要的收束点是：

- 我们不追求第一天做出完整 Claude Code，而是先做出对的骨架。
- 版本路线必须围绕“统一运行时闭环”来排优先级，而不是围绕“哪个功能看起来更酷”。

---

## 10. 第 4 章 系统总览（正文）

本章给出整个系统的总地图。后面所有章节都会把自己挂在这张图上，因此这里的目标不是把细节讲完，而是把层次、边界、主链路和信任关系全部标出来。

### 10.1 系统整体架构图

建议把系统理解成一个五层结构。

```text
+--------------------------------------------------------------+
| 交互与入口层                                                 |
| CLI 子命令 | REPL | Headless/SDK | Remote Viewer | Hooks     |
+--------------------------------------------------------------+
| 启动与编排层                                                 |
| init/setup | config loading | mode selection | app bootstrap |
+--------------------------------------------------------------+
| Runtime 核心层                                               |
| QueryEngine | prompt assembly | transcript | session state   |
| compact | event stream | budget/usage | task coordination    |
+--------------------------------------------------------------+
| 能力与策略层                                                 |
| Command registry | Tool registry | permissions | sandbox     |
| skills | plugins | MCP | agent definitions                   |
+--------------------------------------------------------------+
| 基础设施与外部系统层                                         |
| filesystem | shell | network | model provider | local store  |
| remote transport | external MCP servers                      |
+--------------------------------------------------------------+
```

这五层不是部署层，而是职责层。

其中最重要的判断是：

- 交互层负责把用户带进系统。
- 启动层负责把运行环境准备好。
- Runtime 核心层负责一轮 query 的真正闭环。
- 能力与策略层负责“系统能做什么，以及在什么条件下能做”。
- 基础设施层负责与本地和外部世界交互。

如果后续实现中出现职责漂移，比如 REPL 组件直接组装 prompt、工具直接修改全局状态、插件绕过权限层直接执行副作用，都说明已经偏离这张总图。

### 10.2 核心模块地图

为了让这张总图更可落地，可以先定义建议目录结构。

```text
src/
  entrypoints/
    cli/
    repl/
    headless/
    remote/
  bootstrap/
    init/
    setup/
    config/
  runtime/
    query/
    prompt/
    transcript/
    compact/
    events/
    budget/
  state/
    app-state/
    session/
    persistence/
    snapshots/
  commands/
    builtins/
    registry/
    parser/
  tools/
    builtins/
    registry/
    execution/
    schemas/
    context/
  permissions/
    policy/
    classifier/
    sandbox/
  extensions/
    skills/
    plugins/
    mcp/
  agents/
    definitions/
    tasks/
    coordinator/
  remote/
    bridge/
    transport/
    viewer/
  telemetry/
    logging/
    diagnostics/
    replay/
```

这个结构的核心思想不是“目录漂亮”，而是防止几个高频错误：

- 把运行时逻辑散落到 UI 组件里。
- 把扩展系统直接塞进工具目录。
- 把权限实现成工具内部 if/else。
- 把会话持久化写成 REPL 私有逻辑。

各模块职责建议如下。

#### 10.2.1 `entrypoints/`

负责外部入口适配，不负责核心业务闭环。它的工作是解析输入、准备环境、调用 runtime、消费输出。

#### 10.2.2 `bootstrap/`

负责初始化顺序，例如配置加载、工作目录判断、trust 检查、运行模式选择、日志与诊断启动。

#### 10.2.3 `runtime/`

整个系统的心脏。这里放 Query Engine、上下文拼装、compact、事件流、usage 统计等核心逻辑。

#### 10.2.4 `state/`

管理 AppState、Session、transcript、snapshot、resume 元数据。它要服务所有入口，而不是某个具体 UI。

#### 10.2.5 `commands/`

处理用户显式触发的入口能力，例如 slash command、本地管理命令、prompt command 等。

#### 10.2.6 `tools/`

管理模型可调用能力，包括注册、schema、执行上下文、结果回写、工具日志等。

#### 10.2.7 `permissions/`

实现统一策略层，对 command、tool、plugin、MCP 的执行资格进行治理。

#### 10.2.8 `extensions/`

负责加载 skills、plugins、MCP 等扩展源，但不直接决定它们的执行顺序。扩展装载后应进入命令池、工具池或资源池。

#### 10.2.9 `agents/`

提供子 Agent 的定义、运行、任务跟踪与回收机制，使其成为正式运行时能力，而不是“特殊工具函数”。

#### 10.2.10 `remote/`

处理远程控制、viewer、bridge 和会话跨端同步。

#### 10.2.11 `telemetry/`

提供日志、诊断、回放、分析事件。终端 Agent 的很多问题只有靠运行时事件才能看清。

### 10.3 主要数据流

本系统的主数据流建议定义为：

```text
User Input
  -> Input Normalization
  -> Command Detection / Prompt Construction
  -> Query Context Assembly
  -> System Prompt Composition
  -> Model Request
  -> Assistant Stream Events
  -> Tool Request
  -> Permission Check
  -> Tool Execution
  -> Tool Result Message
  -> Continue Query Loop
  -> Final Assistant Output
  -> Session Persist / UI Render / Replay Log
```

这里有几个关键点。

第一，用户输入不会直接进入模型，而是先经过标准化和入口判断。因为 slash command、attachments、hooks、resume 场景都可能改变输入语义。

第二，模型输出不会直接显示给用户，而是会先被解析成事件流。只有这样，系统才能在中途识别工具调用、权限请求、中断、错误和最终文本输出。

第三，工具执行结果不是临时变量，而是会回写成标准消息，再重新进入 query 循环。这一点非常关键，因为它保证了工具结果成为会话事实的一部分，而不是当前函数栈里的旁路数据。

第四，会话持久化和 UI 渲染都应该消费同一条事件或状态流，而不是各自维护一套“自认为正确”的解释。

### 10.4 主要控制流

数据流描述的是“东西如何流动”，控制流描述的是“系统在什么条件下切换状态”。

本系统至少存在以下几条关键控制流。

#### 10.4.1 启动控制流

```text
CLI start
  -> parse args
  -> init
  -> load config / environment
  -> resolve mode
  -> setup session context
  -> launch REPL or run headless or execute command
```

这里的重点是 `init` 和 `setup` 的拆分。前者更偏全局、安全和环境准备；后者更偏会话局部状态和运行上下文。

#### 10.4.2 REPL 轮次控制流

```text
user submits input
  -> normalize input
  -> handle slash command or enter query
  -> run QueryEngine
  -> stream assistant events
  -> handle tool loop
  -> update transcript/state
  -> render final output
  -> wait for next turn
```

#### 10.4.3 Headless 控制流

```text
receive prompt / task
  -> construct runtime context
  -> run QueryEngine without REPL
  -> emit structured result
  -> persist if requested
  -> exit or return to caller
```

Headless 的重点不是少一个界面，而是多一个“结构化结果出口”。因此 Query Engine 不能假设自己总在和 REPL 组件对话。

#### 10.4.4 Compact 控制流

```text
after turn / before turn / on threshold breach
  -> estimate token usage
  -> compare against warning/blocking policy
  -> decide compact strategy
  -> produce summary / preserved context
  -> rewrite or augment runtime context
  -> continue session
```

Compact 是横切控制流，它可能发生在 query 之前、之后或中间恢复阶段，所以不应该挂在某个具体命令上。

#### 10.4.5 Agent/Task 控制流

```text
parent session decides to delegate
  -> create task record
  -> spawn agent context
  -> run child query loop
  -> stream or buffer results
  -> merge summary/output back to parent
  -> close or retain task state
```

这条控制流在后期章节会详细展开。这里先强调：它和普通工具执行相似，但不是一回事，因为它有独立生命周期和状态空间。

### 10.5 运行边界与信任边界

系统总览里最不能省略的一张图，就是信任边界图。

```text
Trusted Core
  - runtime core
  - state store
  - permission engine
  - built-in command/tool registry

Conditionally Trusted
  - current workspace
  - session transcripts
  - local skills
  - approved plugins

Untrusted / Semi-trusted
  - user prompt content
  - repository files
  - MCP server responses
  - shell output
  - network resources
  - remote peers
```

这个划分会直接影响架构设计。

#### 10.5.1 为什么工作区不是天然可信的

很多终端工具默认“你本地仓库当然可信”，但这在真实场景里并不成立。仓库里可能包含：

- 恶意 prompt injection 文本。
- 误导性的 README 或脚本。
- 会诱导模型执行危险命令的内容。

因此，workspace trust 不能默认等于“本地磁盘内容可信”，而应该成为明确的权限上下文。

#### 10.5.2 为什么 MCP 和插件不能绕过权限层

MCP server 和插件虽然扩展了系统能力，但它们本身也带来了新的攻击面和失控面。如果让它们直接拥有执行能力，就会破坏整个系统的安全一致性。

因此，原则上：

- 扩展可以贡献能力。
- 但最终执行资格仍由统一权限层裁定。

#### 10.5.3 为什么 transcript 既是资产也是风险

transcript 是系统最重要的长期记忆之一，但同时也会携带历史误导、注入内容、错误策略、过期上下文。因此它既要被保留，也要被治理。compact、summary、replay 都建立在这个判断之上。

### 10.6 系统的主设计判断

基于总览图，本项目可以用一句更聚焦的话来概括：

> 这个系统的真正内核不是 REPL，也不是某个工具集合，而是一个能够在权限边界内，持续管理消息、上下文、工具调用和状态演化的 Query Runtime。

只要这句话成立，后续不论是：

- 加一个新 command
- 接一个新 MCP server
- 做一个 background agent
- 增加一个 remote viewer

都应该是在这颗内核之外叠层，而不是把内核重新打碎重做。

### 10.7 本章结论

第 4 章要固定下来的核心事实有四个：

- 系统是分层的，不能让 UI 或扩展面主导内核。
- 主链路是 `输入 -> query -> tool -> result -> transcript -> 持久化/渲染`。
- 权限与信任边界必须在系统总图里出现，而不是后补。
- 后续所有章节，都必须回答“自己在这张总图中的位置是什么”。

---

## 11. 第 5 章 运行模式设计（正文）

本章的任务，是把“这个系统究竟有哪些运行形态”说清楚。终端 Agent Runtime 的一个典型陷阱是：一开始只做 REPL，后面再补 headless、SDK、远程会话，结果每加一种模式都复制一套执行链路。最终看起来像同一个产品，实际上已经变成多个系统拼在一起。

因此，本章的中心思想是：

> 模式可以不同，但执行内核必须统一。

### 11.1 为什么运行模式是架构问题

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

### 11.2 Interactive REPL 模式

REPL 是这个系统最重要的使用场景，因为它最接近真实协作。用户会在这里：

- 连续提出多个问题。
- 查看工具执行过程。
- 批准或拒绝权限请求。
- 使用 slash command 操作会话。
- 恢复历史会话并继续工作。

因此，REPL 模式必须具备以下能力。

#### 11.2.1 用户体验目标

- 支持流式输出，而不是等整轮结束才渲染。
- 支持显示工具调用进度和工具结果摘要。
- 支持权限中断与继续执行。
- 支持 slash command 和普通 prompt 的统一输入框。
- 支持会话内状态可见性，例如当前模型、当前目录、token 压力、后台任务状态。

#### 11.2.2 对运行时的要求

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

#### 11.2.3 REPL 生命周期

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

### 11.3 Headless / SDK 模式

Headless 模式和 SDK 模式可以视为同一能力的两个入口。

- Headless 是命令行环境里的非交互执行。
- SDK 是代码环境里的编程式调用。

它们的共同点是：

- 没有 REPL UI。
- 通常更强调结构化输入和结构化输出。
- 常用于自动化流程、脚本、CI、外部程序调用。

#### 11.3.1 Headless 的设计目标

- 在无 UI 的情况下完整跑通同一套 Query Engine。
- 保留工具调用、工具结果和 transcript 语义。
- 提供结构化返回值，而不是只有一段最终文本。
- 允许调用方选择是否持久化会话。

#### 11.3.2 SDK 的设计目标

- 让外部程序可以嵌入同一套运行时。
- 支持事件订阅，例如流式文本、工具调用、权限请求、状态变更。
- 支持调用方注入权限回调、日志回调和存储适配器。
- 不让 SDK 绕过核心权限、消息和状态模型。

#### 11.3.3 Headless/SDK 与 REPL 的关系

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

#### 11.3.4 无交互权限的处理方式

Headless/SDK 最大的差异点是权限确认不一定能依赖人工交互，因此设计上必须支持：

- `strict-deny`: 无法确认时直接拒绝。
- `preapproved`: 调用方通过策略预先允许。
- `callback`: 由宿主程序提供同步或异步审批回调。
- `simulation`: 只计算计划，不实际执行副作用。

这四种模式应建立在同一权限引擎之上，而不是 headless 私自跳过 ask。

### 11.4 CLI 管理子命令模式

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

#### 11.4.1 Runtime-bound Commands

这类命令仍然需要 runtime 上下文，例如：

- `/compact`
- `/review`
- `/resume`
- 与当前会话密切相关的操作

它们应通过命令系统接入，并在需要时调用 Query Engine 或 Session API。

#### 11.4.2 Control-plane Commands

这类命令主要操作配置和环境，例如：

- `config`
- `mcp list`
- `plugin install`
- `doctor`

它们可以不进入 Query Engine，但依然应该复用统一的配置加载、日志、权限与错误输出基础设施。

### 11.5 远程会话 / Viewer 模式

Remote / Viewer 模式是高级能力，但必须在模式设计阶段先留出位置。

它的本质不是“多一个 UI”，而是把会话的控制权和观察权从本地单终端扩展到跨进程、跨设备、甚至跨网络环境。

#### 11.5.1 Remote 模式的设计目标

- 允许某个 runtime 在远端持续运行。
- 允许本地客户端连接、观察、交互和恢复会话。
- 支持权限请求、工具状态和最终输出的跨端同步。
- 尽量复用本地 transcript、事件流和任务模型。

#### 11.5.2 Viewer 模式的设计目标

- 允许只读观察会话。
- 不默认拥有控制权和权限批准权。
- 与 Remote 模式共享事件协议，但权限更弱。

#### 11.5.3 为什么 Remote 不是另一个 REPL

如果把 Remote 看成“REPL over network”，很容易做出一套单独的远程 UI，然后把核心逻辑也复制过去。正确的做法应该是：

- 远端 runtime 仍是会话事实来源。
- 本地 viewer/control client 只是事件和控制消息的终端。
- transcript、task、permission 状态都以远端 session 为准。

### 11.6 各模式能力矩阵

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

### 11.7 多模式复用策略

为了避免模式分叉，建议采用三层复用策略。

#### 11.7.1 输入适配层

负责把不同入口的原始输入转成统一请求对象，例如：

- REPL 文本输入。
- CLI `--print` 参数。
- SDK 方法调用。
- Remote 控制消息。

#### 11.7.2 核心执行层

负责运行统一的 Query Engine、权限引擎、工具调度和 transcript 更新逻辑。这一层不应该知道自己来自 REPL 还是 SDK。

#### 11.7.3 输出适配层

负责把运行时事件转成：

- 终端渲染。
- JSON 结构化输出。
- SDK 回调事件。
- 远程传输消息。

只要这三层划分稳定，系统就可以持续增加入口而不撕裂内核。

### 11.8 常见错误路径与设计约束

模式设计里最常见的问题有四类。

第一类是 REPL 私有逻辑过多。表现为 query loop 里混入大量 UI 状态判断，导致 headless 无法复用。

第二类是 headless 结果过度简化。只返回最终文本，丢掉工具调用和 transcript 语义，后面很难做 replay 和调试。

第三类是 remote 复刻本地逻辑。这样会导致远端和本地状态出现双主问题。

第四类是权限模型分模式实现。REPL 有 ask，headless 一套 if/else，SDK 一套 callback，最终行为会悄悄不一致。

因此，本章给出的硬约束是：

- 只有输入/输出适配器允许按模式分叉。
- Query Engine、权限引擎、工具执行和 transcript 结构不得按模式分叉。

### 11.9 本章结论

运行模式的正确设计不是“每种模式都造一套系统”，而是：

- 不同入口共享一套内核。
- 不同宿主消费同一套事件和状态。
- 不同交互方式复用同一权限与工具链路。

---

## 12. 第 6 章 启动与初始化设计（正文）

本章关注的是系统从“用户敲下命令”到“进入稳定运行状态”之间的全过程。对于终端 Agent Runtime 来说，启动阶段往往比看上去更重要，因为：

- 很多安全边界发生在真正进入会话之前。
- 很多性能问题发生在无意义的早期加载中。
- 很多模式分叉就从入口编排开始。

所以启动设计的目标不是“能跑起来”，而是“安全、可预测、可复用地跑起来”。

### 12.1 启动设计目标

启动流程至少要同时满足以下目标：

- 快速启动，避免每次打开终端都做大量无关加载。
- 明确区分全局初始化与会话初始化。
- 在 trust 未建立之前，避免装载高风险本地扩展和工作区内容。
- 让模式分发在一个地方完成，而不是散落到多个入口文件。
- 为日志、诊断、profile、异常处理留出统一接入点。

### 12.2 CLI 入口与参数解析

建议整个工程只有一个真正的主入口，例如 `main.tsx` 或 `cli.ts`。这个入口只负责四类事情：

- 解析 CLI 参数和子命令。
- 做最小必要的启动前准备。
- 分发到正确运行模式。
- 为后续模式创建统一 bootstrap context。

主入口不应该承担以下职责：

- 直接实现 Query Engine。
- 直接定义工具列表。
- 直接组装系统提示词。
- 直接管理会话状态。

如果入口文件过重，通常意味着整个工程正在丧失层次感。

#### 12.2.1 参数解析建议

CLI 参数建议按功能分成三类：

- 模式选择参数：例如 `--print`、`--headless`、`--resume`、`--remote`。
- 运行时覆盖参数：例如模型、工作目录、权限模式、compact 策略。
- 控制平面参数：例如 `config`、`doctor`、`version`、`mcp`、`plugin`。

这样可以在解析阶段就初步决定：

- 是否需要真正进入 runtime。
- 是否需要恢复旧 session。
- 是否允许加载工作区相关内容。

### 12.3 启动编排器设计

建议引入一个启动编排器概念，把入口和模式真正隔开。它的职责是：

- 把 CLI 输入变成标准化的启动请求。
- 运行 `init()`。
- 运行 `setup()`。
- 决定最终进入 REPL、headless、control-plane command 还是 remote client。

一个推荐的启动时序如下：

```text
process start
  -> parse argv
  -> create bootstrap context
  -> init(global)
  -> resolve command/mode
  -> setup(session-scoped)
  -> launch selected runner
```

### 12.4 `init` 与 `setup` 的职责划分

`init` 和 `setup` 的划分是整个启动设计最关键的边界之一。

#### 12.4.1 `init()` 的职责

`init()` 处理全局性、模式无关、风险较低、对后续所有模式都必要的准备工作，例如：

- 读取环境变量。
- 建立基础日志与诊断能力。
- 解析全局配置目录。
- 检查运行平台、Node 版本、必要依赖。
- 初始化基础 provider registry。
- 进行最小化的身份与认证状态读取。

`init()` 不应该做的事包括：

- 读取当前仓库里的高信任内容。
- 装载工作区 skill/plugin。
- 恢复具体某个 session。
- 创建会话级状态。

#### 12.4.2 `setup()` 的职责

`setup()` 处理会话相关、模式相关、上下文相关的工作，例如：

- 解析工作目录和项目根。
- 建立 workspace trust 上下文。
- 恢复或创建 Session。
- 装配命令池与工具池。
- 根据模式决定输出适配器和权限交互器。
- 加载 session-scoped 的 hooks、skill、MCP、plugin。

可以把二者理解为：

- `init()` 负责“让程序能安全开始”。
- `setup()` 负责“让这次会话进入可执行状态”。

### 12.5 配置加载时序

终端 Agent Runtime 的配置源通常很多，因此必须定义清晰优先级。建议的加载顺序如下。

#### 12.5.1 配置来源

- 内置默认值。
- 全局用户配置文件。
- 环境变量。
- 工作区配置文件。
- session 恢复元数据。
- CLI 参数覆盖。
- 模式级临时覆盖。

#### 12.5.2 推荐优先级

建议优先级从低到高如下：

```text
defaults
  < global config
  < env vars
  < workspace config
  < resumed session metadata
  < cli flags
  < mode-specific overrides
```

这里要特别注意两点。

第一，工作区配置应在 trust 判断后再生效，至少对会影响执行能力的部分应如此。否则相当于未审查的仓库内容可以影响本地 Agent 行为。

第二，session 恢复元数据不能无条件覆盖 CLI 参数。用户显式传入的本轮参数应拥有更高优先级。

### 12.6 trust 前与 trust 后的安全边界

启动阶段最容易被忽视的问题，就是哪些事情可以在 trust 建立之前做，哪些必须等 trust 建立之后再做。

#### 12.6.1 trust 前允许的动作

- 读取全局配置。
- 读取环境变量。
- 识别当前目录和仓库边界。
- 读取内置命令与内置工具定义。
- 启动基本日志和诊断。
- 进行只读、非执行型的环境探测。

#### 12.6.2 trust 后才允许的动作

- 加载工作区级 skills、plugins、hooks。
- 采纳工作区级 prompt 片段或配置覆盖。
- 恢复依赖仓库内容的 session 上下文。
- 启用会影响工具执行的本地策略扩展。

#### 12.6.3 设计原则

原则上，任何来自当前工作区、且可能影响执行行为的内容，都不应在 trust 建立前直接进入 runtime。

这样做有两个好处：

- 降低 prompt injection 和本地配置劫持风险。
- 让“信任工作区”成为真正有意义的状态，而不是一个摆设。

### 12.7 模式选择与启动分发

当 `init()` 和基础配置完成后，启动编排器就要选择最终模式。建议分发逻辑如下：

```text
if control-plane command:
  run command runner
else if remote viewer/client:
  run remote runner
else:
  run setup()
  if repl:
    launch REPL
  else if headless:
    run headless session
```

这里的重点是：不要让每个子命令自己偷偷做一遍 setup，也不要让 REPL 和 headless 各自组装一遍 runtime。

### 12.8 懒加载与启动性能策略

终端工具的使用频率很高，启动性能会直接影响体验。建议采用以下懒加载策略。

#### 12.8.1 启动时只加载必需能力

必需能力包括：

- 参数解析。
- 基础配置。
- 日志与错误处理。
- 模式判断。

以下内容应尽量延后：

- 大型 prompt 模板。
- 不一定会用到的命令实现。
- plugin/MCP 的深度装载。
- 重量级 UI 组件。
- Agent、remote、viewer 等高级模块。

#### 12.8.2 按模式加载

- REPL 相关 UI 组件只在 REPL 模式导入。
- Headless 输出序列化只在 headless 模式导入。
- Remote transport 只在 remote 模式导入。

#### 12.8.3 按使用路径加载

例如 command registry 可以先只注册元信息，等用户触发命令时再懒加载执行体；MCP server 可以先完成发现和元数据读取，等真正暴露工具时再建立连接。

### 12.9 启动失败与恢复策略

启动设计不能只考虑 happy path，还必须覆盖失败路径。

常见启动失败包括：

- 配置格式错误。
- 工作目录不可访问。
- 认证状态损坏。
- session 恢复数据不兼容。
- MCP/plugin 初始化失败。
- trust 状态缺失或冲突。

建议策略如下：

- 全局配置错误应尽早报错并附带修复建议。
- 工作区相关错误不应破坏 control-plane commands。
- 单个 plugin/MCP 失败应尽量降级，而不是直接拖垮整个 runtime。
- session 恢复失败时允许用户回退到新会话。

### 12.10 本章结论

启动与初始化设计的核心，不是“程序怎么启动”，而是：

- 哪些事情必须只做一次。
- 哪些事情属于具体会话。
- 哪些事情在 trust 前绝不能做。
- 哪些能力可以延后加载以保证启动快。

只要这几条边界清楚，整个系统的入口复杂度就会大幅下降。

---

## 13. 第 7 章 核心领域模型（正文）

本章的目标，是把整个 Agent Runtime 的核心对象体系定义清楚。没有稳定的领域模型，系统后面所有能力都会漂浮不定。尤其是在终端 Agent 这种多入口、多状态、多扩展、多副作用的系统里，核心类型其实比 UI 更重要。

本章不会追求把每个字段一次性定义到最终形态，但会先把必须稳定的接口边界钉住。

### 13.1 建模原则

核心领域模型设计遵循以下原则：

- 模型先服务运行时闭环，再服务外部展示。
- 可进入 transcript 的数据，必须有明确编码方式。
- 可产生副作用的能力，必须有正式上下文对象。
- UI 状态与会话事实必须分离。
- 扩展能力接入时，应落入既有模型，而不是临时造旁路结构。

### 13.2 `Message`

`Message` 是整个系统最核心的对象之一，因为它是 transcript、replay、compact、resume、调试和远程同步的共同基础。

建议定义一个统一消息基类：

```ts
type MessageRole =
  | "system"
  | "user"
  | "assistant"
  | "tool"
  | "summary"
  | "event";

type MessageKind =
  | "text"
  | "tool_use"
  | "tool_result"
  | "command_result"
  | "status"
  | "summary"
  | "attachment";

interface Message {
  id: string;
  sessionId: string;
  turnId: string;
  role: MessageRole;
  kind: MessageKind;
  createdAt: string;
  content: unknown;
  metadata?: Record<string, unknown>;
}
```

这个定义的核心思想不是字段有多少，而是以下三个原则：

- transcript 中所有关键事实都尽量落成标准消息。
- 工具调用和工具结果必须能够被标准化记录。
- 摘要、恢复信息、系统注释等也应该有正式类型，而不是散落在各处的字符串。

#### 13.2.1 为什么要区分 `role` 与 `kind`

`role` 代表消息在对话语义中的位置，`kind` 代表消息的技术形态。二者分开可以避免很多混淆，例如：

- assistant 也可能产生 `tool_use` 类型消息。
- tool 角色通常对应 `tool_result`。
- system 或 summary 也可能以结构化内容存在。

#### 13.2.2 `Message` 的不变量

- 每条消息必须归属于某个 session。
- 每条消息应尽量归属于某个 turn。
- transcript 中的顺序必须可重建。
- 所有进入模型上下文的消息，必须可序列化。

### 13.3 `Command`

`Command` 是用户入口能力，不是模型能力。它的职责是把“用户显式要做的操作”组织成可注册、可发现、可执行的单元。

建议接口如下：

```ts
type CommandType = "prompt" | "local" | "local-jsx";

interface Command {
  name: string;
  type: CommandType;
  description: string;
  aliases?: string[];
  isEnabled?(ctx: CommandContext): boolean | Promise<boolean>;
  isRemoteSafe?: boolean;
  requiresWorkspaceTrust?: boolean;
  run(ctx: CommandContext): Promise<CommandResult>;
}
```

#### 13.3.1 三类命令的职责

- `prompt`: 本质上是 prompt 模板或 prompt 入口，最终会进入 Query Engine。
- `local`: 本地执行逻辑，通常不需要模型参与。
- `local-jsx`: 带交互 UI 或终端组件渲染的本地命令。

#### 13.3.2 `Command` 的边界

命令可以调用 runtime，但不应该绕过 runtime 直接破坏会话事实。例如，一个 `/compact` 命令可以触发 compact 流程，但 compact 后的 transcript 更新仍应通过正式 session API 完成。

### 13.4 `Tool`

`Tool` 是暴露给模型调用的受控能力单元。工具的定义必须足够正式，因为它涉及：

- 能力暴露。
- schema 校验。
- 权限审查。
- 执行上下文。
- 结果回写。
- 调试与回放。

建议接口如下：

```ts
interface Tool<Input = unknown, Output = unknown> {
  name: string;
  description: string;
  inputSchema: unknown;
  isReadOnly?: boolean;
  requiresApproval?: boolean;
  isEnabled?(ctx: ToolExposureContext): boolean | Promise<boolean>;
  invoke(
    input: Input,
    ctx: ToolUseContext
  ): Promise<ToolExecutionResult<Output>>;
}
```

#### 13.4.1 `Tool` 与 `Command` 的根本区别

- `Command` 由用户显式触发。
- `Tool` 由模型在推理过程中调用。

这个区别直接决定了：

- `Tool` 必须具备更严格的 schema 和权限控制。
- `Command` 更强调入口体验和发现性。

#### 13.4.2 `Tool` 的返回值

建议工具不要只返回一段文本，而是返回结构化结果：

```ts
interface ToolExecutionResult<T = unknown> {
  output: T;
  displayText?: string;
  metadata?: Record<string, unknown>;
  artifacts?: Array<{ type: string; uri?: string; data?: unknown }>;
}
```

这样既能满足模型继续推理，也能满足 UI 展示和 replay。

### 13.5 `ToolUseContext`

`ToolUseContext` 是连接“运行时”与“工具执行”的桥梁。它的目的，是阻止工具直接依赖全局单例或隐式环境。

建议接口如下：

```ts
interface ToolUseContext {
  session: Session;
  appState: AppState;
  cwd: string;
  abortSignal: AbortSignal;
  permissions: PermissionEvaluator;
  logger: Logger;
  emitEvent(event: RuntimeEvent): void;
  appendMessage(message: Message): Promise<void>;
  resolvePath(path: string): string;
}
```

#### 13.5.1 为什么 `ToolUseContext` 重要

如果工具可以自由读取全局变量、直接操作磁盘、自己决定如何写回 transcript，那么：

- 工具不可测试。
- 权限不可统一。
- replay 不可重建。
- SDK/remote 模式无法稳定复用。

所以 `ToolUseContext` 是必须稳定的中枢接口之一。

### 13.6 `AppState`

`AppState` 代表运行时当前视角下的全局状态快照，但它不等于“所有东西都堆在一个对象里”。建议它是一个由多个 slice 组成的根状态：

```ts
interface AppState {
  session: SessionState;
  ui: UIState;
  runtime: RuntimeState;
  permissions: PermissionState;
  tools: ToolState;
  tasks: TaskState;
  telemetry: TelemetryState;
}
```

这里最重要的原则是：

- `session` 和 `runtime` 是事实状态。
- `ui` 是表现状态。
- `permissions`、`tools`、`tasks` 是横切状态。

### 13.7 `Session`

`Session` 代表一次持续交互的完整容器。建议定义如下：

```ts
interface Session {
  id: string;
  createdAt: string;
  updatedAt: string;
  cwd: string;
  mode: "repl" | "headless" | "sdk" | "remote";
  transcript: Message[];
  summary?: SessionSummary;
  configSnapshot: SessionConfigSnapshot;
  metadata: Record<string, unknown>;
}
```

`Session` 需要解决的不是“保存聊天记录”这么简单，而是：

- 让一次协作具备稳定身份。
- 让 compact 和 resume 有依托。
- 让权限、任务、工作目录、配置快照有归属。

### 13.8 `Task`

当系统引入后台任务、子 Agent、长命令执行、远程协作时，单靠 session 已经不够，需要 `Task` 来表示一个具有独立生命周期的工作单元。

```ts
type TaskStatus =
  | "pending"
  | "running"
  | "waiting_input"
  | "completed"
  | "failed"
  | "cancelled";

interface Task {
  id: string;
  sessionId: string;
  kind: "tool" | "agent" | "command" | "remote";
  status: TaskStatus;
  title: string;
  createdAt: string;
  updatedAt: string;
  parentTaskId?: string;
  result?: unknown;
  error?: SerializedError;
}
```

`Task` 让系统可以：

- 跟踪后台 agent。
- 在 UI 中显示进行中的工作。
- 对长执行工具做中断和恢复。
- 在 remote 模式中同步工作状态。

### 13.9 `AgentDefinition`

`AgentDefinition` 是“某类子 Agent 的可执行描述”，而不是一次具体运行。

```ts
interface AgentDefinition {
  name: string;
  description: string;
  prompt: string;
  tools: string[];
  model?: string;
  permissionProfile?: string;
  executionMode?: "sync" | "background";
}
```

其关键价值在于：

- 能力可裁切。
- prompt 可专门化。
- 子 Agent 可以标准化生成，而不是在主逻辑里拼 prompt。

### 13.10 `PermissionRule`

权限模型必须可序列化、可解释、可组合。建议至少包含：

```ts
type PermissionDecision = "allow" | "deny" | "ask";

interface PermissionRule {
  id: string;
  scope: "tool" | "command" | "path" | "network" | "plugin" | "mcp";
  matcher: Record<string, unknown>;
  decision: PermissionDecision;
  reason?: string;
  source: "default" | "user" | "workspace" | "session" | "policy";
}
```

这样做的好处是：

- 权限来源可追踪。
- ask/allow/deny 的覆盖关系可解释。
- headless/SDK/remote 可以复用同一规则系统。

### 13.11 `MCPServer`

MCP 不应只是“一段连接配置”，而应是正式注册对象。

```ts
interface MCPServer {
  id: string;
  name: string;
  transport: "stdio" | "http" | "sse" | "ws";
  status: "disconnected" | "connecting" | "ready" | "error";
  capabilities: {
    tools?: boolean;
    resources?: boolean;
    prompts?: boolean;
    commands?: boolean;
  };
  config: Record<string, unknown>;
}
```

这个模型让我们后续可以清晰表达：

- 哪个 server 当前已连接。
- 它贡献了哪些能力。
- 它是否在当前模式可用。
- 它的失败是否应该影响整个 runtime。

### 13.12 模型之间的关系

这些核心模型之间的关系建议如下：

```text
Session
  -> owns Message[]
  -> owns Task[]
  -> references config snapshot

AppState
  -> points to current Session
  -> tracks runtime/ui/permission/tool slices

Command
  -> entered by user
  -> may create Query or mutate Session through APIs

Tool
  -> exposed to model
  -> executes with ToolUseContext
  -> writes Message / Task / events back

AgentDefinition
  -> used to create Task + child Session/Query

PermissionRule
  -> filters Command / Tool / MCP / Plugin behavior
```

如果这个关系图不稳定，后续 compact、MCP、agent、remote 都会出现职责冲突。

### 13.13 本章结论

第 7 章真正要固定下来的，不是字段细节，而是这些判断：

- `Message` 是会话事实的基本单位。
- `Command` 和 `Tool` 必须明确分离。
- `ToolUseContext` 是受控执行的关键桥梁。
- `Session`、`Task`、`AppState` 共同构成长期运行骨架。
- `PermissionRule` 和 `AgentDefinition` 必须是正式一等对象。

---

## 14. 第 8 章 会话与状态管理设计（正文）

如果说上一章定义的是“系统里有哪些关键对象”，那本章定义的就是“这些对象如何在长时间运行中保持一致”。终端 Agent Runtime 最容易被低估的部分就是状态管理，因为早期 demo 往往只需要一个消息数组，但真实使用很快就会碰到：

- 长会话 transcript 爆炸。
- 工具执行中断。
- 多个并发任务。
- UI 状态和会话事实混在一起。
- 恢复旧会话时上下文错位。

所以本章的目标，是把状态系统从“临时内存变量”提升成正式设计。

### 14.1 状态分层原则

建议整个系统至少分成四层状态。

#### 14.1.1 会话事实状态

这类状态构成系统长期记忆，应该可以持久化和恢复，例如：

- transcript
- compact summary
- cwd
- session config snapshot
- task records
- 权限决策历史

#### 14.1.2 运行时过程状态

这类状态反映当前执行过程，不一定需要长期持久化，例如：

- 当前 query 是否正在运行
- 正在流式输出的 assistant 内容
- 当前工具调用栈
- abort / interrupt 标记
- 当前 token 估算结果

#### 14.1.3 UI 表现状态

这类状态只服务渲染，例如：

- 输入框内容
- 当前焦点
- 展开的面板
- REPL 视图滚动位置
- 权限弹窗是否打开

#### 14.1.4 派生状态

这类状态可以从其他状态计算得到，不应作为独立事实源，例如：

- 当前 turn 的消息列表
- 某个文件最近一次修改结果
- session 是否接近 token 上限
- 当前是否存在可恢复任务

这个分层的意义在于：不是所有“看起来有用的数据”都应该持久化，也不是所有状态都应该放进 AppState 根对象里。

### 14.2 AppState 结构设计

建议 `AppState` 采用分 slice 设计，而不是单一大对象无序增长。

```ts
interface AppState {
  session: SessionState;
  runtime: RuntimeState;
  ui: UIState;
  permissions: PermissionState;
  tasks: TaskState;
  integrations: IntegrationState;
}
```

可以进一步理解为：

- `session`: 当前会话的持久事实。
- `runtime`: 当前执行中的短生命周期状态。
- `ui`: REPL 或 viewer 的表现层状态。
- `permissions`: 当前 ask/allow/deny 相关状态与历史。
- `tasks`: 后台 agent、长工具执行、命令任务。
- `integrations`: MCP/plugin/remote 连接状态。

#### 14.2.1 为什么不用“只有一个 Session 对象”

因为运行时中存在大量不属于 session 长期事实的数据。例如某次流式响应已经输出到第几个 chunk、当前工具是否等待人工批准、某个 viewer 是否在线，这些都不应直接塞进 `Session` 本体。

### 14.3 消息历史与 transcript 设计

transcript 是系统最核心的持久状态之一。建议遵循以下原则。

#### 14.3.1 transcript 是事实日志，不是 UI 缓存

它应该记录：

- 用户输入。
- assistant 输出。
- 工具调用与工具结果。
- compact 产生的 summary。
- 关键系统事件。

它不应该记录：

- 每个流式 token 的临时拼接过程。
- REPL 纯展示状态。
- 仅用于本次渲染的局部标志。

#### 14.3.2 transcript 的两种视图

建议同时支持：

- `raw transcript`: 完整消息序列，用于恢复、回放、诊断。
- `effective transcript`: 经过 compact 和过滤后，实际进入模型上下文的消息视图。

如果没有这两个视图，compact 往往会把“历史事实”和“当前上下文窗口”混成一件事。

#### 14.3.3 turn 分组

建议消息在存储层保留 turn 归属。这样可以支持：

- 按轮次回放。
- 按轮次 compact。
- 分析某轮 query 的完整成本和工具调用。

### 14.4 文件状态缓存

终端 Agent 的一个独特问题，是它不仅要管理对话，还要管理“工作区变化感知”。因此建议引入文件状态缓存层。

#### 14.4.1 缓存目标

- 跟踪工具最近读写过哪些文件。
- 记录文件哈希、mtime 或内容快照摘要。
- 帮助系统判断某个文件是否在本轮外被修改。
- 为 diff 展示、compact 和恢复提供辅助信息。

#### 14.4.2 为什么需要文件缓存

如果没有文件缓存，系统很难回答：

- 用户修改过的文件是否被外部进程改写。
- 某次 edit 工具返回后，文件是否处于预期状态。
- compact 后哪些文件事实需要保留给模型。

这层缓存不必一开始就很重，但至少要把“当前会话感知到的文件状态”纳入正式模型。

### 14.5 file history 与 snapshot

仅有当前文件缓存还不够，系统还需要某种轻量历史能力。

#### 14.5.1 file history

file history 关注的是：

- 某个文件在会话内被哪些工具修改过。
- 每次修改发生在哪个 turn。
- 修改前后摘要是什么。

#### 14.5.2 snapshot

snapshot 更关注一个时间点上的整体会话状态，例如：

- transcript 截面
- 关键文件状态摘要
- 当前权限模式
- 未完成 task 列表
- compact summary

snapshot 的价值在于：

- 恢复失败时有回退点。
- rewind 时有锚点。
- 远程同步时可减少全量重传。

### 14.6 session persistence

会话持久化建议从简单方案起步，但必须设计清楚。

#### 14.6.1 持久化内容

第一版建议持久化：

- session metadata
- transcript
- compact summary
- config snapshot
- task metadata
- permission history

可选持久化：

- 文件状态缓存
- replay 事件流
- 远程同步游标

#### 14.6.2 存储形式

MVP 阶段推荐：

- `session.json` 保存元数据
- `transcript.jsonl` 保存消息流
- `tasks.json` 保存任务状态
- `summary.json` 保存 compact 结果

这样做的好处是简单、可调试、可人工检查。

#### 14.6.3 写入策略

建议采用“增量追加 + 关键点快照”的混合方式：

- transcript 追加写入。
- summary 和 metadata 覆盖更新。
- 大状态变更后触发快照。

这样可以减少单文件重写成本，也更适合 replay。

### 14.7 resume / fork / rewind 机制

这三个能力很容易被混淆，但职责不同。

#### 14.7.1 `resume`

在同一 session 上继续工作。其核心要求是：

- transcript 连续。
- compact summary 连续。
- task 状态可恢复或可解释。
- cwd 与 config snapshot 有明确继承规则。

#### 14.7.2 `fork`

从某个 session 或某个 turn 分叉出新会话。其核心价值是允许用户：

- 保留原始思路。
- 试验另一条执行路径。
- 在不污染主线 transcript 的情况下探索方案。

fork 通常应继承：

- transcript 某个截面。
- compact summary。
- 部分配置快照。

但不一定继承：

- 进行中的任务。
- UI 状态。
- 某些一次性权限批准。

#### 14.7.3 `rewind`

rewind 不是 resume，也不是 fork。它表示：

- 会话仍然是同一个 identity。
- 但上下文视角回退到某个更早节点。

rewind 的难点在于：

- 已发生的文件副作用不一定可逆。
- transcript 需要保留历史还是截断，需要明确策略。

因此设计上建议：

- 初期优先支持逻辑 rewind，而不是自动回滚文件系统。
- rewind 后通过系统消息明确标记“当前有效上下文基于哪个快照/turn”。

### 14.8 状态变更传播机制

一旦系统进入流式输出、工具回写、后台任务、多端观察场景，状态更新就不能靠“谁需要谁就直接读写”。

建议采用以下机制：

#### 14.8.1 单向更新原则

- 核心运行时通过明确 API 更新状态。
- UI 只订阅状态，不直接篡改会话事实。
- 工具通过 `ToolUseContext` 间接写入状态。

#### 14.8.2 事件与状态分离

建议同时存在：

- `state store`: 当前事实快照。
- `event stream`: 变化过程和过程性信号。

例如：

- “当前 assistant 正在输出第 10 个 chunk”更像事件。
- “当前 session 的 transcript 已追加一条 assistant 消息”是状态更新。

两者分离后，REPL、SDK、remote、replay 都会更清晰。

#### 14.8.3 订阅粒度

建议按 slice 订阅，而不是每次整个 AppState 全量刷新。否则 REPL 和 remote viewer 都容易在长会话中出现性能问题。

### 14.9 并发与一致性约束

即使第一版不做复杂并发，也必须先定义一致性约束。

建议至少满足：

- 同一 session 同时只有一个前台 query 在推进。
- 背景 task 与前台 query 通过任务系统隔离。
- transcript 追加必须有顺序保障。
- compact 期间禁止并发修改 transcript 主体。
- session 持久化写入必须避免部分成功、部分失败。

如果这些约束不先说清楚，后续一旦引入 agent 或 remote，状态系统会非常脆弱。

### 14.10 错误路径与恢复设计

状态系统必须为错误场景预留恢复能力。

常见问题包括：

- transcript 写入中断。
- compact 结果生成失败。
- task 状态与实际执行脱节。
- session 文件部分损坏。
- 恢复旧 session 时 schema 升级不兼容。

建议策略如下：

- transcript 采用追加式存储，尽量减少整体损坏概率。
- snapshot 保留版本号。
- 恢复时允许以“只读恢复模式”打开损坏会话。
- 对无法完全恢复的字段进行显式降级，而不是静默丢弃。

### 14.11 本章结论

第 8 章要固化的核心思想是：

- 长会话系统一定需要正式状态分层。
- transcript 是事实日志，不是 UI 缓存。
- 文件状态、task、snapshot、summary 都是会话治理的一部分。
- resume、fork、rewind 必须明确区分。
- 事件流和状态存储应并存，而不是互相替代。

---

## 15. 第 9 章 用户输入处理设计（正文）

到这里为止，系统的骨架已经立起来了。接下来开始进入 runtime 的主链路。第 9 章关注的不是“用户输入框怎么做”，而是“用户输入如何从原始文本演化成一次正式 query 的起点”。

这一步在 Claude Code 风格系统里非常关键，因为用户输入并不是简单的一段字符串。它可能同时包含：

- 普通文本 prompt。
- slash command。
- pasted content。
- 图片和其他附件。
- IDE 选区。
- hooks 注入的附加上下文。
- system 生成的 meta prompt。
- remote/bridge 传来的受限输入。

如果这些入口差异不在这里统一吸收，后面的 Query Engine 就会不断承担本不属于它的分歧逻辑。

### 15.1 输入处理在总架构中的位置

输入处理层位于：

```text
entrypoint / REPL / SDK
  -> input normalization
  -> attachment extraction
  -> command detection
  -> hook interception
  -> user/system message assembly
  -> query entry
```

它既不属于纯 UI，也不等于 Query Engine 本体。更准确地说，它是 Query Engine 的前置编排层。

这层的目标有四个：

- 把异构输入标准化为统一消息结构。
- 识别哪些输入需要进入 query，哪些只需本地处理。
- 在进入模型前尽可能补足有价值的上下文。
- 在安全边界内阻断不应该继续的输入。

### 15.2 输入类型矩阵

建议先把系统支持的输入类型分清楚。

#### 15.2.1 普通文本输入

这是最常见路径，通常表现为：

- REPL 输入框中的自然语言。
- Headless/SDK 传入的 prompt 字符串。

它最终通常会转成一条 `user` 消息，并进入 query。

#### 15.2.2 结构化内容块输入

这类输入在 SDK、IDE、远程端更常见，例如：

- 文本 + 图片的组合块。
- 前置 context blocks + 最后一个用户问题。

这类输入不应在一开始就粗暴降级成字符串，而应保留为 content block 数组，直到进入 API 适配层。

#### 15.2.3 slash command 输入

slash command 的关键点是：它看起来像文本，但语义上不是普通 prompt。它可能：

- 完全本地执行。
- 先修改会话状态，再进入 query。
- 生成新的 prompt 并继续提交。
- 在 REPL 模式下渲染本地 JSX 交互。

所以 slash command 必须在输入层被优先识别，而不是交给模型猜测。

#### 15.2.4 附件与 pasted content

这类输入通常不是用户显式敲出来的内容，但对任务非常重要，例如：

- 粘贴的长文本片段。
- 图片粘贴。
- IDE 当前选区。
- hooks 追加的额外上下文。
- agent mention、memory、技能发现等系统性附件。

这些内容更适合进入 attachment/message 流，而不是拼成一坨字符串埋进 prompt。

#### 15.2.5 系统生成输入

系统内部也可能注入输入，例如：

- queued command 触发的后续 prompt。
- scheduler/background task 的隐式提示。
- compact、resume、rewind 过程中的系统引导。

这类输入通常应该被标记为 `isMeta`，做到“模型可见、用户可不见”。

### 15.3 输入标准化流水线

结合上游实现，建议输入标准化至少经过以下步骤。

```text
raw input
  -> detect input shape (string / content blocks)
  -> normalize media blocks
  -> extract text tail / leading blocks
  -> collect pasted images and attachments
  -> decide slash command eligibility
  -> run prompt-submit hooks
  -> construct user/attachment/system messages
  -> return ProcessUserInputResult
```

在 Claude Code 风格实现里，这一步的代表性入口就是 `processUserInput()`。它不是“把字符串转消息”的薄包装，而是一个正式的前置管道。

### 15.4 为什么 `processUserInput` 不是薄函数

从 sourcemap 还原出来的 `processUserInput.ts` 可以看出，这一层实际上承担了很多策略职责：

- 立即显示用户输入，避免交互延迟感。
- 在 query profiler 中打点，成为运行链路的一部分。
- 调用 `processUserInputBase()` 先做基础标准化。
- 再执行 `UserPromptSubmit` hooks。
- 根据 hooks 结果阻断、附加上下文、或中止 continuation。

这说明输入处理并不是一个工具函数，而是 query 启动前的正式状态机前半段。

建议在复现项目中保留两层结构：

- `processUserInputBase()`: 负责形态标准化和本地分流。
- `processUserInput()`: 负责 hooks、阻断和更高层策略。

这样做的好处是：

- 附件、slash command、图片处理可以独立测试。
- hooks 不会污染最底层输入归一化逻辑。
- Headless/SDK 和 REPL 可以复用同一输入前置流水线。

### 15.5 图像与多模态输入处理

Claude Code 风格实现对图片输入的处理非常值得借鉴，因为它不是把图片简单透传，而是先做标准化：

- 调整尺寸和降采样，确保符合 API 限制。
- 生成图片元信息文本，例如尺寸、来源路径。
- 将 pasted image 存盘，以便后续工具链也能引用本地路径。
- 在最终消息中同时保留 image block 与必要元数据。

这背后的设计思想是：

- 多模态输入不应该只服务模型，也要服务后续工具链。
- 图像不是“附件特判”，而是正式 content block 类型。
- 一些模型不可见但对调试与工具有用的信息，可以通过 `isMeta` user message 附加。

### 15.6 attachment 提取与角色设计

在复现设计中，建议把附件提取设计成一条独立流水线，而不是散落在 REPL 或 IDE 适配层。

推荐策略：

- 原始输入只携带最直接的 prompt 内容。
- 附加上下文通过 `AttachmentMessage` 系列对象进入 transcript。
- attachment 不必全都转成字符串；允许保留类型化结构。

建议 attachment 至少覆盖这些类型：

- pasted_text
- pasted_image
- ide_selection
- hook_additional_context
- agent_mention
- memory
- skill_discovery
- structured_output

这样做的优势非常大：

- transcript 可解释。
- compact 可以按 attachment 类型决定保留策略。
- remote/replay 能保真还原“上下文是怎么被带进来的”。

### 15.7 slash command 解析与输入分流

输入层需要回答一个关键问题：

> 当前这段以 `/` 开头的内容，究竟是一个命令，还是普通文本？

上游实现里这一点处理得很谨慎，尤其在 bridge/remote 情况下：

- 默认可以禁止远端输入触发本地 slash command。
- 只对白名单 bridge-safe commands 放开执行。
- 对已知但不允许远程执行的命令，直接本地 short-circuit 返回解释信息。

这说明 slash command 解析不仅是“功能发现”，还是安全边界的一部分。

建议在复现项目中将 slash command 分流规则明确成：

```text
if skipSlashCommands:
  treat as plain text
else if startsWith('/'):
  parse command
  if command exists and is allowed in current transport:
    execute command path
  else if command exists but forbidden in this mode:
    return local error/result
  else:
    fallback to plain text or unknown-command handling
```

### 15.8 hooks 在输入处理中的位置

Claude Code 风格实现里，`UserPromptSubmit` hooks 是在基础输入处理之后、真正进入 query 之前执行的。这是一个很好的边界选择。

原因是：

- 此时用户原始输入已经被标准化，hooks 可以拿到更稳定上下文。
- 但 query 还没真正发起，hooks 仍能阻断危险或不合规输入。

hooks 结果至少可能有三类：

- `blockingError`: 直接阻止本轮继续。
- `preventContinuation`: 保留 prompt 但不继续 query。
- `additionalContexts`: 以 attachment 形式追加上下文。

这里值得借鉴的核心思想是：

- hooks 不是字符串替换器，而是运行时插槽。
- hooks 的产物也应进入正式消息流，而不是“神秘地改了 prompt”。

### 15.9 文本 prompt 如何落成正式消息

当输入最终走普通文本路径时，仍然不应该简单生成 `{ role: "user", content: string }`。

上游 `processTextPrompt()` 至少做了几件事：

- 生成 prompt id，用于 tracing 和遥测。
- 启动一次 interaction span。
- 识别 keep-going、negative 等用户意图信号。
- 处理文本与图片 blocks 的组合消息。
- 将 attachment message 一并返回。

这说明“文本 prompt”实际上是：

- 一条用户消息。
- 一组伴随 attachment。
- 一组 tracing / analytics side effects。

因此在复现设计里，建议 `processTextPrompt()` 的返回值是：

```ts
interface ProcessTextPromptResult {
  messages: Message[];
  shouldQuery: boolean;
}
```

而不是只返回字符串。

### 15.10 输入到 query 的边界

输入处理阶段的标准出口建议定义为：

```ts
interface ProcessUserInputResult {
  messages: Message[];
  shouldQuery: boolean;
  allowedTools?: string[];
  model?: string;
  effort?: EffortValue;
  resultText?: string;
  nextInput?: string;
  submitNextInput?: boolean;
}
```

这个出口很有代表性，因为它表明输入处理层不仅能产出消息，还能改变后续 query 条件，例如：

- 当前轮只允许哪些工具。
- 当前轮切换哪个模型。
- 当前轮设置怎样的 effort。
- 当前输入根本不需要 query，而只返回本地结果。

也就是说，输入层已经是 runtime policy 的一部分。

### 15.11 输入处理阶段的异常路径

这一层的错误和异常通常分成几类：

- 图片过大、尺寸不合法。
- pasted content 解析失败。
- slash command 不存在或在当前模式不可用。
- hooks 明确阻断。
- 输入为空但模式要求文本。

建议处理原则如下：

- 尽可能在输入层就转换成标准消息或结构化错误。
- 不要让 Query Engine 再去猜“为什么这轮没开始”。
- 对于命令模式不可用的情况，优先本地解释，而不是把原始 `/foo` 暴露给模型。

### 15.12 设计总结

第 9 章真正要强调的，是输入处理层的地位：

- 它不是 UI 辅助函数，而是 query 的前置编排器。
- 它既负责标准化，也负责局部分流和安全阻断。
- 它输出的不是字符串，而是“是否进入 query + 哪些消息 + 用什么约束进入”。

---

## 16. 第 10 章 System Prompt 与上下文拼装设计（正文）

这一章是整个 Claude Code 风格架构中最值得细抠的部分之一。很多人以为“系统 prompt”就是一个超长字符串，但从上游实现看，真正的设计重点从来不是“写一段神奇提示词”，而是：

- 把系统提示拆成稳定模块。
- 区分静态前缀和动态后缀。
- 明确什么属于 system prompt，什么属于 user context，什么属于 system context。
- 保证不同模式、不同 agent、不同扩展都能在同一个拼装框架下叠加。

换句话说，Claude Code 风格系统最值得学的不是 prompt 内容，而是 prompt builder 的工程化方式。

### 16.1 prompt 体系的三层结构

从 `queryContext.ts` 和 `systemPrompt.ts` 可以看出，Claude Code 风格系统实际上把前缀上下文拆成三个部分：

- `systemPrompt`
- `userContext`
- `systemContext`

在 query 发起时，大致形成：

```text
prependUserContext(messages, userContext)
+ systemPrompt
+ appendSystemContext(systemPrompt, systemContext)
```

这是一种非常重要的设计：

- `systemPrompt` 负责稳定身份、行为准则、工具使用规则。
- `userContext` 更像会话/用户侧上下文前缀。
- `systemContext` 更像环境与系统补充信息。

这种拆法比把一切都硬塞进单个 system prompt 更可控，也更利于缓存治理。

### 16.2 `getSystemPrompt()` 的真正职责

上游 `constants/prompts.ts` 里的 `getSystemPrompt()` 并不是返回一段字符串，而是返回一个 `string[]` 数组。这一点非常关键。

它意味着：

- system prompt 被视为“可组合的 section 列表”。
- 各 section 可以单独定义、单独启停、单独决定缓存边界。
- 最终 prompt 不是手拼长文本，而是由 section resolver 装配。

从 sourcemap 可以看出，它至少包含这些 section 类型：

- intro / system / doing tasks / actions / tone & style
- tool usage guidance
- output efficiency
- session-specific guidance
- memory
- env info
- language
- output style
- MCP instructions
- scratchpad
- summarize tool results
- token budget
- brief / proactive 等 feature section

这说明 Claude Code 风格系统真正采用的是“prompt registry + section composition”的思路。

### 16.3 静态前缀与动态边界

这是整个 prompt 架构里最值得借用的设计之一。

在 `constants/prompts.ts` 中，存在明确的：

```ts
SYSTEM_PROMPT_DYNAMIC_BOUNDARY
```

它的语义是：

- 该边界之前的 prompt section 尽量保持跨会话稳定，可用于全局 cache。
- 该边界之后的 section 才允许放会随 session、用户、MCP 连接、当前环境变化的内容。

这背后的思想非常强：

- prompt cache 不是模型服务层才关心的事，而是应用层 prompt builder 要主动配合的事。
- 只要把易变内容混进前缀，整个缓存命中率就会崩。

对复现项目来说，这意味着我们也应该在 prompt 设计中引入明确边界，例如：

```text
[static identity + stable policy + stable tool guidance]
=== dynamic boundary ===
[session guidance + env info + memory + mcp instructions + language/style overrides]
```

### 16.4 system prompt 不是固定文案，而是优先级系统

从 `buildEffectiveSystemPrompt()` 可以看到，真正最终生效的 system prompt 有明确优先级：

1. `overrideSystemPrompt` 完全替换一切
2. `coordinator prompt`
3. `agent prompt`
4. `custom system prompt`
5. `default system prompt`
6. `appendSystemPrompt` 总是尾部附加

这代表一个成熟系统在看待 prompt 时，不是“拼接几段字符串”，而是“做优先级决策”。

建议复现时也明确支持这几种层级：

- `default`: 产品默认行为与身份。
- `custom`: 用户显式替换默认。
- `agent`: 为特定 agent 定制行为。
- `override`: 内部强制替换。
- `append`: 在主 prompt 后附加策略或会话限定。

### 16.5 为什么 agent prompt 有时替换、有时追加

`systemPrompt.ts` 中有一个很有意思的判断：

- 正常模式下，agent prompt 可以替换 default prompt。
- 在 proactive 模式下，agent prompt 改为追加到 default prompt 后。

这说明 agent prompt 不是一个固定位置的插槽，而是取决于当前运行模式的“角色增强层”。

这个思想值得借鉴：

- 有些模式下，agent 是主身份，应替换 default。
- 有些模式下，agent 只是对主身份加一层领域能力，应追加。

所以复现设计里不应该把 `agentPrompt` 写死成一种拼法，而应把它纳入 mode-aware prompt assembly。

### 16.6 user context 与 system context 的分工

从 `fetchSystemPromptParts()` 可以看到，上游会并行拿到：

- `defaultSystemPrompt`
- `userContext`
- `systemContext`

而且当 `customSystemPrompt` 被显式设置时，`getSystemContext()` 甚至会被跳过，因为它默认是设计给 default prompt 体系使用的。

这件事说明了两个设计原则。

第一，`userContext` 和 `systemContext` 不是任意混放的附加文本，而是有依附关系的。

第二，定制 prompt 时，系统需要知道哪些默认上下文还应该存在，哪些不应该再自动注入。

建议复现时约定：

- `userContext`: 与当前用户、会话、远端控制身份、协作模式相关的上下文。
- `systemContext`: 与环境、平台、工作目录、模型、附加目录、当前 transport 相关的上下文。

### 16.7 环境信息为什么必须体系化

上游 prompt 体系里有一整块专门的环境信息 section，例如：

- 当前工作目录
- 是否 git repo
- additional working directories
- platform / shell / OS version
- 模型与知识截止信息

这不是“为了看起来详细”，而是为了让模型一开始就处于正确的运行世界观中。

原因包括：

- 工具调用强依赖工作目录和平台。
- shell 指令在不同系统上差异巨大。
- 模型能力与知识截止会影响建议。
- 多工作目录会影响文件访问边界。

因此环境信息应被视为稳定、可拼装的 prompt section，而不是某个工具需要时临时告诉模型。

### 16.8 工具指导为什么放进 system prompt

上游 prompt 里有一个非常重要的 section：指导模型优先使用专用工具，而不是一切都走 bash。

例如其设计意图大致是：

- 读文件优先走 Read tool。
- 改文件优先走 Edit/Write tool。
- 搜索优先走 Glob/Grep tool。
- Bash 应该保留给真正需要 shell 执行的场景。

这类内容之所以适合放进 system prompt，而不是仅靠工具 description，是因为它属于“全局工具使用策略”。

也就是说：

- tool description 解决“这个工具是什么”。
- system prompt 里的工具 section 解决“你在多工具环境中应该优先怎么选”。

### 16.9 session-specific guidance 为什么要后置

`constants/prompts.ts` 里专门区分了一类：

```text
session-specific guidance
```

它包含这类信息：

- 本轮有没有 AskUserQuestionTool。
- 本轮有没有 AgentTool。
- 当前会话是不是 non-interactive。
- 当前有哪些 skills。
- 当前是否开启 verification agent 等策略。

这些内容如果提前混进 static prefix，就会严重破坏缓存稳定性。

因此值得借鉴的原则是：

- 工具集的全局稳定指导可以靠前。
- 当前 session 的瞬时能力与策略，必须后置到 dynamic 区域。

### 16.10 memory、MCP、language、output style 的叠加方式

上游设计里，以下内容都不是“散落在别处”的补丁，而是正规 prompt section：

- memory prompt
- MCP server instructions
- language preference
- output style
- scratchpad instructions

这说明一个成熟系统不会把这些能力当成“额外注释”，而是纳入统一 prompt assembly 框架。

复现时建议用统一 section registry 表达：

```ts
interface PromptSection {
  id: string;
  phase: "static" | "dynamic";
  compute(ctx: PromptBuildContext): Promise<string | null>;
  cacheSensitivity?: "global" | "session" | "volatile";
}
```

这样后续接 memory、MCP、plugins、agents 时都不会破坏整体结构。

### 16.11 QueryEngine 如何消费这些 prompt parts

从 `QueryEngine.ts` 看，headless/SDK 模式不是直接调用 `getSystemPrompt()`，而是先：

1. `fetchSystemPromptParts()`
2. 合并 coordinator user context
3. 根据 custom prompt / memoryMechanics / appendSystemPrompt 拼出最终 `systemPrompt`

这说明 QueryEngine 不是 prompt 的生成源，而是 prompt parts 的消费者与最终组装者。

建议复现时把职责分清：

- `prompt registry / builder`: 负责生成各 parts。
- `query entry`: 负责根据当前会话状态、模式、agent 身份决定最终组合。

### 16.12 memory mechanics prompt 的启发

上游有一个很值得注意的小设计：当 SDK 调用方显式提供 custom system prompt，并启用 memory path override 时，会额外注入一段 memory mechanics prompt，告诉模型如何使用 memory 文件。

这个细节的启发是：

- 当你允许外部调用方替换默认 prompt 时，某些 runtime 关键机制可能会从默认 prompt 中丢失。
- 此时需要用“机制补丁 prompt”恢复模型对系统能力的理解。

因此在复现项目里，也应该考虑“能力说明补丁”这一层，而不是假设 custom prompt 永远不会破坏工具使用世界观。

### 16.13 prompt 设计的工程原则

从上游设计里可以总结出几条非常核心的工程原则。

#### 16.13.1 prompt section 化

不要维护一个巨大的字符串常量。改成 section registry。

#### 16.13.2 cache-aware

明确静态前缀与动态后缀边界。

#### 16.13.3 mode-aware

REPL、SDK、coordinator、agent、proactive 等模式都应能在同一拼装框架内表达。

#### 16.13.4 capability-aware

prompt 必须感知当前工具集、MCP 状态、memory 状态和权限语境，而不是盲目固定。

#### 16.13.5 override-safe

自定义 prompt、append prompt、agent prompt、memory mechanics 这几层要有明确优先级，而不是简单字符串拼接。

### 16.14 设计总结

第 10 章最核心的结论是：

- Claude Code 风格系统真正强的不是 prompt 内容，而是 prompt builder。
- prompt 是“分段、分层、分优先级、带缓存边界”的运行时构件。
- 一旦把 prompt 工程化，agent、MCP、memory、remote、coordinator 才能持续叠加而不失控。

---

## 17. 第 11 章 Query Engine 设计（正文）

这一章是整个 runtime 的心脏。如果前面的章节是在搭骨架，那么 Query Engine 就是让整副骨架真正动起来的循环系统。

从 sourcemap 看，`query.ts` 和 `QueryEngine.ts` 的关系非常清楚：

- `query.ts` 是底层一轮 query 的状态机与异步生成器。
- `QueryEngine.ts` 是更高层、面向 headless/SDK/会话对象的封装器。

理解这两层关系，是吃透整个工程逻辑的关键。

### 17.1 为什么 `query.ts` 才是真正的执行引擎

`query.ts` 定义的是：

```ts
export async function* query(params): AsyncGenerator<...>
```

这意味着 Query Engine 的底层模型不是“返回一个结果”，而是“产生一串事件与消息”。

这个设计特别重要，因为一轮 agent 执行并不是单步完成的，它天然会经历：

- request start
- assistant 流式输出
- tool_use block 到达
- tool 执行进度
- tool_result 回写
- compact boundary
- retry / recovery
- 最终成功或终止

如果把这一切硬压成一个 `Promise<Result>`，系统会立刻失去：

- 流式可观察性
- 细粒度中断点
- SDK/REPL/remote 的统一事件模型

所以 Query Engine 的正确抽象是异步事件流，而不是一次性 RPC。

### 17.2 `QueryEngine` 与 `query()` 的职责分工

上游设计给了一个非常清晰的两层分工。

#### 17.2.1 `query()`

负责：

- 一轮 query 的状态机。
- 上下文预处理与 compaction 入口。
- 模型调用与流式事件消费。
- tool_use 检测与工具执行。
- 各种 recovery / retry / budget / stop hook 逻辑。
- 产出 `Message | StreamEvent | Tombstone | Summary` 等流。

#### 17.2.2 `QueryEngine`

负责：

- 维持一个长期会话对象。
- 管理 `mutableMessages`、usage、permission denials、read cache。
- 在每次 `submitMessage()` 前跑输入处理和 prompt 组装。
- 把 `query()` 的产物流转成 SDK/Headless 可消费输出。
- 负责 transcript 持久化和最终 result 聚合。

可以说：

- `query()` 是单轮执行机。
- `QueryEngine` 是多轮会话容器。

### 17.3 一轮 query 的生命周期

综合 `query.ts` 和 `QueryEngine.ts`，一轮 query 的生命周期大致如下：

```text
submitMessage()
  -> fetch system prompt parts
  -> processUserInput()
  -> append messages / persist transcript
  -> call query()
      -> prepare messagesForQuery
      -> apply budgets / snip / microcompact / context collapse / autocompact
      -> assemble final prompt
      -> stream model response
      -> collect assistant messages / tool_use blocks
      -> run tools
      -> append tool results
      -> maybe continue next iteration
      -> maybe recover/retry/compact
  -> normalize output to SDK/REPL messages
  -> compute final result
```

这个生命周期里最值得注意的是：输入处理、prompt 组装、query 状态机、工具执行、最终结果整合，都是明确分层的，而不是揉成一个大函数。

### 17.4 QueryEngine 为什么要维护 `mutableMessages`

`QueryEngine` 内部维护 `mutableMessages`，不是为了偷懒，而是因为 headless/SDK 需要一个长期存活、跨 turn 的消息容器。

它承担的职责包括：

- 保留会话事实。
- 在 `processUserInput()` 前后允许 slash command 修改消息数组。
- 在 query streaming 过程中逐步追加 assistant/user/attachment/progress/system。
- 在 compact boundary 后释放旧消息，限制内存增长。

这说明 QueryEngine 不只是“包一层 query()”，而是真正的 session runtime host。

### 17.5 `query.ts` 的状态模型

在 `query.ts` 中，最值得学习的是它显式定义了 loop state：

```ts
type State = {
  messages
  toolUseContext
  autoCompactTracking
  maxOutputTokensRecoveryCount
  hasAttemptedReactiveCompact
  maxOutputTokensOverride
  pendingToolUseSummary
  stopHookActive
  turnCount
  transition
}
```

这说明它不是“while(true) 里随手改几个变量”，而是一个显式状态机。

这种写法的价值在于：

- recovery path 可以被表达为状态迁移。
- 测试可以检查 `transition.reason`，而不必只盯着消息文案。
- 后续若要抽成 reducer / step machine，也已经具备形状。

这也是我们复现时应优先借鉴的点：不要让 query loop 演化成隐式状态泥团。

### 17.6 Query 入口前的上下文预处理

真正进入模型调用之前，`query.ts` 会做大量预处理。这一点很容易被低估。

根据 sourcemap，至少包括：

- `getMessagesAfterCompactBoundary`
- `applyToolResultBudget`
- `snipCompactIfNeeded`
- `microcompact`
- `contextCollapse`
- `autocompact`

这说明在 Claude Code 风格系统中，“要发给模型的上下文”不是 transcript 的直接切片，而是一个经过多轮治理后的投影视图。

因此在复现设计里，要明确区分：

- `raw transcript`
- `messagesForQuery`

前者是事实，后者是当前回合的工作上下文。

### 17.7 为什么 compact 逻辑在 query loop 里

很多实现会把 compact 做成独立命令或后台服务，但上游把自动 compact 直接嵌进 query loop。这是一个非常关键的设计判断。

原因是：

- compact 的触发条件依赖当前 query 的 token 压力。
- compact 的结果要立刻影响当前这一轮是否继续。
- compact 后需要重新构造 `messagesForQuery` 并继续同一轮 query。

这说明 compact 不是 session 管理外围功能，而是 query continuity 的一部分。

### 17.8 模型请求为什么要走流式事件

`query.ts` 不是“拿到整条 assistant 消息后再分析”，而是在 `for await` 流式消费模型输出时处理：

- `message_start`
- `message_delta`
- `message_stop`
- assistant content blocks
- tool_use blocks

这样做有几个关键收益：

- 可以实时渲染文本。
- 可以在 tool_use block 到达时尽早启动工具执行。
- 可以捕捉 usage 和 `stop_reason` 的精确时点。
- 可以在 streaming fallback 时 tombstone 掉孤儿消息。

这其中 tombstone 机制尤其值得注意：当流式请求发生 fallback 时，旧流里已经产出的 partial assistant 消息不能继续保留，否则会污染 transcript 并破坏 thinking block 约束。

这说明 Query Engine 不只是“接流”，而是在维护一份严格一致的对话轨迹。

### 17.9 为什么有 `StreamingToolExecutor`

工具执行这块，上游明显做了两套层次：

- `runTools()`：按并发安全性分批调度。
- `StreamingToolExecutor`：在 tool_use blocks 流式到达时边收边跑。

这背后的核心思想是：

- 并非所有工具都必须等 assistant 整条消息结束后再执行。
- 工具执行本身也应该被纳入流式回合的一部分。

`StreamingToolExecutor` 做的事情包括：

- 追踪 queued/executing/completed/yielded 状态。
- 判断哪些工具可并行，哪些必须独占。
- 立即吐出 progress messages。
- 在流式 fallback、用户中断、兄弟 Bash 失败时产生 synthetic error tool_result。
- 保持结果按工具出现顺序回放。

这已经不是一个“调用函数列表”的工具框架，而是一个小型并发调度器。

### 17.10 tool_use 到 tool_result 的闭环

一轮 query 中，模型输出 `tool_use` 之后并没有结束，真正的闭环是：

```text
assistant tool_use block
  -> permission check
  -> tool execution
  -> tool_result message
  -> append to messages
  -> continue same query loop
```

上游非常强调这一点，甚至对缺失 tool_result 的情况还有专门的补偿逻辑 `yieldMissingToolResultBlocks()`。

这说明系统非常在意一条完整的不变量：

> 只要 assistant 发出过 tool_use，就必须有对应的 tool_result 返回到对话轨迹中。

这是整个 agent runtime 能稳定继续推理的基础。

### 17.11 Query 中的预算、重试与恢复

`query.ts` 里最复杂、也最体现成熟度的部分，就是各种异常与恢复路径。它处理的不是单一错误，而是一整套 runtime resilience。

至少可以看到这些治理能力：

- token blocking limit
- task budget remaining
- max output tokens escalation
- reactive compact retry
- context collapse drain retry
- stop hooks
- max turns reached
- structured output retry limit
- max USD budget stop

这说明 Query Engine 不是“发请求失败就报错”，而是一个真正会尝试自救的 runtime。

建议复现时，也把错误分层成：

- `recoverable within turn`
- `recoverable via compact/retry`
- `terminal but explainable`
- `terminal and diagnostic-heavy`

### 17.12 `QueryEngine` 如何输出最终结果

QueryEngine 的另一个重要职责，是把 query 流最终压缩成外部调用方能理解的 `result`。

从上游实现看，它会持续消费消息流，同时维护：

- `totalUsage`
- `permissionDenials`
- `turnCount`
- `structuredOutputFromTool`
- `lastStopReason`

最后再根据 terminal message 生成：

- `success`
- `error_max_turns`
- `error_max_budget_usd`
- `error_max_structured_output_retries`
- `error_during_execution`

这说明“result”不是 query 自己直接返回的，而是 QueryEngine 对流的终结性解释。

### 17.13 QueryEngine 为什么适合 SDK/Headless

上游代码里对 `QueryEngine` 的注释写得很清楚：一个 `QueryEngine` 对应一个 conversation，每次 `submitMessage()` 开启新 turn。

这恰好说明它为什么特别适合 SDK/Headless：

- 调用方可以把它当作会话对象。
- 状态跨 turn 保持。
- 输出是结构化消息流。
- 中断、恢复、usage、permission denial 都可以被包装进同一对象。

而 REPL 更接近直接围绕 `query()` 或 session state 工作。

所以复现时建议：

- 保留 `query()` 这一底层异步生成器。
- 另外提供 `QueryEngine` 这一面向编程调用的会话封装层。

### 17.14 推荐的复现接口

建议 QueryEngine 至少暴露：

```ts
interface QueryEngine {
  submitMessage(
    prompt: string | ContentBlockParam[],
    options?: { uuid?: string; isMeta?: boolean }
  ): AsyncGenerator<SDKMessage, void>;
  interrupt(): void;
  getMessages(): readonly Message[];
  getReadFileState(): FileStateCache;
  getSessionId(): string;
  setModel(model: string): void;
}
```

而底层 query 建议保持：

```ts
async function* query(params: QueryParams): AsyncGenerator<QueryEvent, Terminal>
```

### 17.15 设计总结

第 11 章最核心的结论是：

- Query Engine 不是“发模型请求”的薄层，而是整个 Agent Runtime 的执行内核。
- `query.ts` 是单轮状态机，`QueryEngine` 是多轮会话容器。
- compact、tool execution、retry、budget、stop hooks 都属于 query 生命周期，而不是外围补丁。

---

## 18. 第 12 章 Tool 系统设计（正文）

Claude Code 风格系统的另一根中轴，就是工具系统。这里最值得学的，不是“它有哪些工具”，而是“它如何把工具变成一个可治理、可扩展、可记录、可并发调度的能力平面”。

如果没有这套工具系统，Claude Code 风格工程就会退化成“模型输出命令，然后宿主去执行”的脆弱模式。真正成熟的地方，是它把工具正式化了。

### 18.1 工具系统要解决的根本问题

工具系统同时要满足四方诉求：

- 对模型：工具是可理解、可选择、可调用的能力。
- 对 runtime：工具是受权限约束、可观察、可回写的副作用单元。
- 对开发者：工具是可注册、可测试、可组合的模块。
- 对产品：工具要支持内置能力、MCP 扩展、agent、skills、plugins 的长期共存。

所以工具系统的本质不是“函数表”，而是一个带 schema、策略、上下文和回写机制的运行子系统。

### 18.2 `Tool` 抽象的关键点

从 `Tool.ts` 和相关实现看，一个成熟的 `Tool` 至少应具备：

- `name`
- `description`
- `inputSchema`
- `isEnabled()`
- `isConcurrencySafe()`
- `interruptBehavior()`
- `invoke()`

其中最关键的不是字段列表，而是这些设计选择：

- `inputSchema` 让工具调用是可验证的。
- `isEnabled()` 让工具可按环境/模式/feature gate 显隐。
- `isConcurrencySafe()` 让调度器决定是否可并行。
- `interruptBehavior()` 让 runtime 决定用户打断时的策略。

这说明工具定义本身已经包含调度语义，而不仅是业务语义。

### 18.3 `ToolUseContext` 为什么是工具系统中枢

`ToolUseContext` 是整个工具系统最重要的中枢接口之一。

从上游定义看，它不仅包含：

- `options.tools / commands / mcpClients / agentDefinitions`
- `abortController`
- `readFileState`
- `getAppState / setAppState`

还包含大量运行时接口，例如：

- `setInProgressToolUseIDs`
- `setHasInterruptibleToolInProgress`
- `setResponseLength`
- `updateFileHistoryState`
- `updateAttributionState`
- `requestPrompt`
- `handleElicitation`
- `addNotification`
- `appendSystemMessage`

这说明工具不是跑在一个“纯函数沙盒”里，而是跑在运行时框架提供的受控环境里。

值得借鉴的核心原则是：

- 工具不直接依赖全局单例。
- 工具对 runtime 的交互通过 context 暴露。
- 只把经过设计允许的能力开放给工具。

### 18.4 base tools、special tools 与 feature gating

`tools.ts` 的设计非常值得学习。它不是简单导出一个数组，而是：

- 定义 `getAllBaseTools()`
- 再经 `getTools(permissionContext)` 做模式和 deny 规则过滤
- 再经 `assembleToolPool(permissionContext, mcpTools)` 合并 MCP 工具

而且内置工具本身还会受：

- feature flags
- env flags
- 平台能力
- mode 开关
- REPL 特殊行为

影响。

这说明工具池装配是 runtime 的职责，不是每个入口自己拼。

### 18.5 为什么 `tools.ts` 是“装配中心”而不是“工具目录索引”

`tools.ts` 里做的事远不止“导出工具”：

- 简单模式下裁剪工具池。
- REPL 模式下隐藏 primitive tools。
- 通过 deny rules 过滤工具可见性。
- 保持 built-in 与 MCP tool 的排序稳定。
- 按名字去重，让 built-in 优先。

这几点背后都是系统级约束：

- 权限影响工具可见性，不只是执行时审查。
- 模式影响工具暴露，不只是文档展示。
- 工具排序影响 prompt cache。
- MCP 扩展不能随意打乱内置工具前缀。

### 18.6 prompt cache 为什么会影响工具排序

这也是一个很容易被忽略、但很体现成熟度的细节。

上游在 built-in tools 与 MCP tools 合并时，专门强调：

- built-ins 保持连续前缀。
- 各分区内部稳定排序。
- 避免平铺排序导致 MCP 工具插进 built-in 中间，从而破坏下游缓存键。

这意味着在 Claude Code 风格系统里，“工具列表”不仅是能力集合，也是 prompt 前缀的一部分。

所以复现时应明确：

- 工具排序不是随意的。
- 装配逻辑必须稳定、可重复。
- 任何扩展机制都不应破坏内置工具前缀稳定性。

### 18.7 deny rules 为什么在模型看到工具之前就生效

上游 `filterToolsByDenyRules()` 的作用，不是阻止工具执行，而是直接把 blanket-denied 工具从工具池里移除。

这说明权限有两层：

- `visibility filtering`: 模型根本看不到这项能力。
- `invocation decision`: 模型调用时再做更细粒度允许/拒绝。

这个设计非常重要，因为：

- 如果模型先看到能力，再在执行时被拦住，会浪费推理并增加摩擦。
- 某些 MCP server prefix deny 规则本来就是想彻底移除整类能力。

因此复现时应把权限整合进工具装配阶段，而不是只在 `invoke()` 前做校验。

### 18.8 `runTools()` 的调度模型

`toolOrchestration.ts` 展示了一个非常清晰的调度思路：

- 先按工具调用顺序分批。
- 每批要么是一个非并发安全工具。
- 要么是一组连续的并发安全工具。

对应逻辑大致是：

```text
tool_use blocks
  -> partitionToolCalls()
  -> run serial batches for unsafe tools
  -> run concurrent batches for safe tools
  -> apply queued context modifiers
```

这说明上游并没有采用“全部并行”或“全部串行”的粗暴方式，而是让工具自己声明并发语义，再由调度层执行。

### 18.9 `isConcurrencySafe()` 的价值

这是工具系统里一个非常关键、但很容易被忽略的设计点。

对于某些工具：

- 读文件、搜索类操作通常可以并行。
- 编辑文件、写文件、执行某些 shell 命令往往不安全。

如果工具定义层不表达这个语义，调度器就只能保守地全部串行，或者危险地全部并行。

所以复现时建议：

- 把并发安全性作为工具定义的一部分。
- 默认保守，只有明确安全的工具才允许并发。

### 18.10 `StreamingToolExecutor`：工具执行器为什么像个小 runtime

这是上游工具系统最精彩的部分之一。

`StreamingToolExecutor` 做的事情包括：

- 维护 tool queue。
- 判断当前哪些 queued 工具可以启动。
- 区分 `queued / executing / completed / yielded` 状态。
- 立即吐出 progress 消息。
- 保证结果按工具到达顺序回放。
- 在 Bash 错误时取消并行兄弟工具。
- 在用户中断或 streaming fallback 时生成 synthetic error blocks。

换句话说，它不只是一个 executor，更像是“query loop 内嵌的小型工具任务 runtime”。

### 18.11 为什么只在 Bash 失败时取消兄弟工具

上游 `StreamingToolExecutor` 有个很细的策略：

- 如果某个工具出错，不一定取消其他并行工具。
- 但如果是 Bash 出错，会触发 sibling abort。

原因也非常合理：

- Bash 工具常常代表命令链的一部分，失败后后续 shell 结果可能都失去意义。
- Read、WebFetch 这类工具更独立，一个失败不应拖垮其他并行查询。

这说明工具系统的恢复策略不应该一刀切，而要允许按工具类别定制。

### 18.12 synthetic error / synthetic tool_result 的必要性

一个成熟的 agent runtime 不仅要处理真实工具结果，还要处理“理论上应该有结果，但因为中断、fallback、并发取消没来得及正常返回”的情况。

上游为此专门做了 synthetic tool_result，例如：

- user_interrupted
- sibling_error
- streaming_fallback
- missing tool

这背后的不变量仍然是：

> 只要 assistant 发过 tool_use，系统就要尽力回填一个 tool_result，哪怕它是 synthetic 的错误结果。

这个设计很关键，因为它能保持消息轨迹闭合，让后续模型与 replay 都不出现悬空调用。

### 18.13 progress message 为什么要单独处理

从 `StreamingToolExecutor` 和 `QueryEngine` 都能看出，progress message 被特殊对待：

- 工具执行时可以随时产生 progress。
- progress 会优先即时 yield。
- 但它又不应当与最终 tool_result 混淆。

因此建议在复现中明确：

- progress 是正式消息类型。
- 它服务 UI、SDK 和 remote 观察。
- 但它不是会话事实的最终结论。

### 18.14 `runToolUse()` 的真正角色

虽然我们没有在这里逐行拆完 `toolExecution.ts`，但从外围调用关系已经很清楚：

- `runToolUse()` 是单个工具调用的细粒度执行器。
- 它内部串起 schema 校验、权限判定、hooks、实际执行、结果标准化和 telemetry。

也就是说，工具系统不是：

```text
tool block -> tool.invoke()
```

而是：

```text
tool block
  -> find tool
  -> parse/validate input
  -> permission decision
  -> run pre-tool hooks
  -> invoke tool
  -> normalize tool result / attachments / progress
  -> run post-tool hooks
  -> return message updates + context modifiers
```

这个“context modifiers”也很关键，因为它说明工具可以合法地改变后续 query 的运行上下文，但必须通过受控 modifier 回写。

### 18.15 工具结果为什么不能只是一段文本

上游工具结果可能被落成：

- `user` message 中的 `tool_result`
- `attachment` message
- `progress` message
- `tool_use_summary`

这意味着工具输出不是单一通道，而是多通道结果系统。

复现时建议至少支持：

- 面向模型继续推理的标准 `tool_result`
- 面向 UI/SDK 的 progress
- 面向结构化消费的 attachment/artifact
- 面向日志与调试的 metadata

### 18.16 MCP 工具为何能接进同一工具池

之所以 Claude Code 风格系统能把 MCP 工具接进来，却不把整体弄乱，就是因为它先有稳定的 `Tool` 抽象和装配流程。

MCP 工具进入系统后，仍然需要经过：

- 名称规范化
- deny rule 过滤
- 工具池合并
- prompt 排序稳定化
- 调用期权限判断

所以 MCP 在这里不是第二套工具系统，而是同一工具平面的外部来源。

这也是复现时必须守住的原则：扩展来源可以不同，运行时工具语义必须统一。

### 18.17 工具系统的推荐接口层次

建议复现项目中把工具系统拆成以下层次：

#### 18.17.1 Tool Definition Layer

定义工具元信息、schema、并发语义、interrupt 语义。

#### 18.17.2 Tool Registry / Pool Assembly Layer

负责 base tools、feature-gated tools、mode filter、permission visibility filter、MCP 合并。

#### 18.17.3 Tool Execution Layer

负责单个 tool_use 的权限、hooks、invoke、结果标准化。

#### 18.17.4 Tool Orchestration Layer

负责多 tool_use 的批处理、并发调度、顺序保证。

#### 18.17.5 Streaming Tool Runtime Layer

负责流式工具执行、进度回传、取消、fallback、synthetic result。

### 18.18 设计总结

第 12 章最核心的结论是：

- 工具系统不是函数表，而是能力运行时。
- `Tool` 定义层、工具池装配层、单调用执行层、多调用编排层、流式执行层必须分开。
- 权限、排序稳定性、并发安全、synthetic result、progress 消息，都是成熟工具系统不可缺的部分。

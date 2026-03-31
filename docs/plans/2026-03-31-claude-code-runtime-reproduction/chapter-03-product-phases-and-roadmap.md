# 第 3 章 产品分层与版本路线

> 状态: 已完成初稿
> 章节目标: 把系统拆成可落地阶段，避免一开始范围失控。

[返回总览](/Users/magongli/Downloads/project/claude-code-sourcemap/docs/plans/2026-03-31-claude-code-runtime-reproduction/README.md)

---

本章的任务，是把一个潜在上限很高、也很容易失控的系统，拆成可以分阶段落地的工程路线。Claude Code 风格系统的复杂度很高，但它并不要求我们第一天就把所有能力做齐。相反，只有先把骨架做对，后面的高级能力才不会变成技术债。

## 3.1 MVP 范围

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

## 3.2 增强版范围

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

## 3.3 完整版范围

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

## 3.4 每个阶段的验收标准

下面给出建议验收标准。

### 3.4.1 MVP 验收

- 能启动 REPL，并连续完成至少一个真实仓库内的小任务。
- 能通过 headless 模式执行单轮或多步工具调用任务。
- 工具调用可被记录进 transcript，并能被后续 query 继续使用。
- 权限询问和放行逻辑可稳定工作。
- 会话可以保存并重新打开。
- 失败场景下不会导致 transcript 或状态完全损坏。

### 3.4.2 增强版验收

- Slash command、skills、MCP 可以共同接入而不产生明显职责冲突。
- compact 可以自动触发，并在触发后保持关键上下文连续性。
- 用户可以恢复旧会话并继续工作。
- 基础 replay 和日志足以定位常见问题。
- 能在中等长度会话中保持可接受的延迟和成本。

### 3.4.3 完整版验收

- 子 Agent 可以被定义、启动、跟踪和回收。
- 远程会话与本地 REPL 之间职责清晰，状态可同步。
- 插件和 MCP 不会绕过统一权限和审计机制。
- 系统在多扩展、多会话、多任务场景下仍可定位问题。
- 关键运行链路具备较完整的集成测试与回放样本。

## 3.5 当前实现优先级

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

## 3.6 建议的里程碑拆分

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

## 3.7 本章结论

本章最重要的收束点是：

- 我们不追求第一天做出完整 Claude Code，而是先做出对的骨架。
- 版本路线必须围绕“统一运行时闭环”来排优先级，而不是围绕“哪个功能看起来更酷”。

## 3.8 本章对复现工程的直接指导

如果你准备正式进入实现，这一章最应该立刻转成一份“范围冻结清单”。

### 3.8.1 第一阶段明确不做什么

建议直接冻结这些“暂不实现”项：

- remote / bridge
- marketplace
- 多 agent 协调
- 复杂 UI
- 企业策略系统

### 3.8.2 第一阶段必须做什么

必须完成：

- 单轮 query 闭环
- 基础工具
- 权限 gate
- transcript/resume
- REPL 与 headless 共享内核

### 3.8.3 每个里程碑都要有退出条件

例如：

- query loop 还不稳，不进下一阶段
- compact 和 resume 还不稳，不上 MCP/plugin
- task lifecycle 还不稳，不上 multi-agent/remote

### 3.8.4 把路线图和代码目录绑定

建议阶段与模块对应：

- Phase 0-1: `bootstrap/query/tools/session`
- Phase 2: `commands/context/compact`
- Phase 3: `mcp/plugins`
- Phase 4: `agents/tasks`
- Phase 5: `remote/bridge`

### 3.8.5 路线图的核心价值

这一章不是为了“看起来规划很完整”，而是为了防止项目在第一个月就被高级功能拖穿。

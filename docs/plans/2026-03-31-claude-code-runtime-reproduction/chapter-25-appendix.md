# 第 25 章 附录

> 状态: 已写  
> 章节目标: 提供实现和复盘时高频查阅的辅助索引。

[返回总览](/Users/magongli/Downloads/project/claude-code-sourcemap/docs/plans/2026-03-31-claude-code-runtime-reproduction/README.md)

---

## 25.0 使用方式

这章不是连续阅读型正文，而是一个“查表页”。  
当你进入实际复现阶段时，可以把它当成：

- 术语速查表
- 上游模块映射表
- 核心时序清单
- 状态机清单
- 配置与风险 checklist

---

## 25.1 术语表

### AppState

整个 runtime 的主状态容器，承载会话状态、UI 状态、权限上下文、任务状态等。

### Command

面向用户的显式触发单元，通常对应 slash command。

### Skill

可复用的 prompt/command 资源，通常由 markdown/frontmatter 驱动。

### Tool

面向模型的能力单元，由 query engine 暴露给模型调用。

### ToolUseContext

工具执行时可访问的运行时上下文，负责连接工具与 session/app state。

### Query Engine

负责完成一轮模型调用、流式事件处理、工具调用循环与递归继续。

### Compact

长会话上下文压缩机制，用于在 token 预算受限时保留关键上下文继续对话。

### MCP

Model Context Protocol 集成层，把外部工具、资源、prompt 等能力接入 runtime。

### Plugin

扩展发行单元，用于分发 command、agent、hook、MCP、skill 等能力集合。

### AgentDefinition

对一个子 agent 的正式定义，包含 prompt、tools、hooks、MCP、memory、permissionMode 等运行策略。

### Task

background agent 或长时操作的生命周期对象，用于跟踪进度、结果与控制动作。

### Coordinator Mode

多 agent 协作模式。主 agent 负责分工、综合与用户沟通，worker 负责研究、实现、验证。

### Remote / Bridge

把同一 runtime 会话跨端承载的 transport/control 平面。

---

## 25.2 上游模块到复现工程的映射建议

| 上游概念 | 复现工程建议模块 | 说明 |
|---|---|---|
| `main.tsx` | `src/bootstrap/cli.ts` | CLI 入口与模式分发 |
| `setup.ts` / `init` | `src/bootstrap/setup.ts` | 环境、配置、registry 初始化 |
| `query.ts` / `QueryEngine.ts` | `src/query/engine.ts` | query 主循环 |
| `commands.ts` | `src/commands/registry.ts` | 命令注册中心 |
| `tools.ts` | `src/tools/registry.ts` | 工具注册中心 |
| `permissions/*` | `src/permissions/*` | 权限规则、解释、匹配、UI |
| `services/compact/*` | `src/compact/*` | compact 与上下文治理 |
| `services/mcp/*` | `src/mcp/*` | MCP 配置、连接、投影 |
| `utils/plugins/*` | `src/plugins/*` | plugin manifest、loader、cache |
| `tools/AgentTool/*` | `src/agents/*` | agent definition、spawn、task |
| `remote/*` + `bridge/*` | `src/remote/*` | remote session 与 bridge |
| `utils/*Profiler*` | `src/telemetry/*` | profiler 与 diagnostics |

---

## 25.3 核心时序图清单

建议你在真正实现前，把下面这些时序图单独画出来。

### 25.3.1 启动时序

`CLI -> parse args -> init/setup -> registry load -> mode dispatch -> REPL/headless start`

### 25.3.2 单轮 query 时序

`user input -> context assembly -> model stream -> tool_use -> tool execution -> tool_result -> recursive query -> assistant final`

### 25.3.3 compact 时序

`threshold reached -> compact request -> summary/result -> boundary insert -> context restore -> next query`

### 25.3.4 command 执行时序

`slash input -> command parse -> availability check -> prompt/local/local-jsx execution -> query or local action`

### 25.3.5 plugin 装载时序

`discover source -> resolve path/version -> validate manifest -> load components -> register contributions -> report active/failed`

### 25.3.6 agent spawn 时序

`AgentTool call -> resolve definition -> init context/MCP/hooks -> runAgent -> progress/task events -> completion/notification`

### 25.3.7 remote permission relay 时序

`remote tool request -> control_request -> local permission UI -> control_response -> remote continue/cancel`

---

## 25.4 状态机清单

建议至少为下面这些对象定义状态机。

### 25.4.1 Query 状态机

- `idle`
- `preparing`
- `streaming`
- `executing_tool`
- `waiting_permission`
- `compacting`
- `completed`
- `failed`
- `aborted`

### 25.4.2 Task 状态机

- `created`
- `running`
- `background`
- `waiting_permission`
- `completed`
- `failed`
- `killed`

### 25.4.3 MCP Client 状态机

- `configured`
- `connecting`
- `connected`
- `auth_required`
- `disconnected`
- `failed`

### 25.4.4 Remote Session 状态机

- `initializing`
- `ready`
- `reconnecting`
- `viewer_only`
- `failed`
- `closed`

### 25.4.5 Permission Request 状态机

- `pending`
- `approved`
- `denied`
- `cancelled`
- `timed_out`

---

## 25.5 配置项矩阵建议

复现工程至少建议定义下面这类配置：

### 25.5.1 运行模式

- `interactive`
- `print`
- `sdk`
- `remote-viewer`
- `coordinator`

### 25.5.2 权限模式

- `default`
- `acceptEdits`
- `plan`
- `bypassPermissions`

### 25.5.3 诊断开关

- `PROFILE_STARTUP`
- `PROFILE_QUERY`
- `DUMP_PROMPTS`
- `DIAGNOSTICS_FILE`

### 25.5.4 扩展开关

- `ENABLE_MCP`
- `ENABLE_PLUGINS`
- `ENABLE_MULTI_AGENT`
- `ENABLE_REMOTE`

### 25.5.5 安全开关

- `STRICT_PLUGIN_ONLY_CUSTOMIZATION`
- `ALLOWED_PLUGIN_SOURCES`
- `SANDBOX_MODE`
- `DEFAULT_PERMISSION_MODE`

---

## 25.6 复现时最容易漏掉的风险清单

### 25.6.1 架构风险

- REPL 与 headless 各写一套 query 逻辑
- tool/command/plugin 各自持有独立状态
- UI 组件直接操纵底层 runtime

### 25.6.2 运行时风险

- tool result 没有结构化回写
- transcript/resume 格式不稳定
- compact 之后丢关键上下文
- background task 无法恢复

### 25.6.3 扩展风险

- plugin 只做目录扫描，没有 manifest
- MCP 工具不做 namespace
- skill/plugin/command 边界不清

### 25.6.4 安全风险

- 写操作默认放行
- 路径规范化不完整
- dump prompt 默认开启
- remote token 泄露到普通环境

### 25.6.5 工程风险

- 没有 query profiler
- 没有 VCR / replay
- 一开始就冲 multi-agent 或 remote

---

## 25.7 推荐阅读顺序

如果你准备真的开始实现，建议按下面顺序反复看：

1. 第 4 章 系统总览
2. 第 7 章 核心领域模型
3. 第 10 章 System Prompt 与上下文拼装
4. 第 11 章 Query Engine
5. 第 12 章 Tool 系统
6. 第 14 章 权限与沙箱
7. 第 15 章 Compact
8. 第 23 章 实现路线图
9. 第 21 章 测试策略
10. 第 22 章 安全设计

---

## 25.8 最后的落地建议

你真正开工时，可以把这整套文档拆成三份随手清单：

### 实现清单

- 先做 Phase 0-2
- 先稳定 query/tool/session
- 每一阶段都补测试与 profiler

### 风险清单

- 权限
- transcript
- compact
- plugin source
- remote continuity

### 验收清单

- 单轮 query 闭环
- 长会话可持续
- 权限模型可信
- 扩展面可治理
- 多入口共内核

---

## 25.9 你最应该借用的核心思想

附录这章真正的价值，不在于“多一个收尾文件”，而在于把整个复现工程压缩成一套能反复查阅的工程索引。  
当你真正开始写代码时，这页会比长段说明更常用。

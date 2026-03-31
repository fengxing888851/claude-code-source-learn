# Claude Code Sourcemap 架构拆解与学习指南

> 面向目标: 把这个工程的整体架构、运行链路、功能模块、设计思想一次看透，并提炼出可迁移到你自己项目里的方法论。

**对象版本:** `@anthropic-ai/claude-code@2.1.88`

**分析依据:**
- 仓库顶层实际只有 npm 包产物和 sourcemap: `package/cli.js`、`package/cli.js.map`
- 不是官方内部开发仓，也不是现成可直接阅读的 `src/` 仓库
- 本文基于 `cli.js.map` 里的 `sources` 和 `sourcesContent` 反推出 TypeScript 源文件结构后分析

**先给结论:**
- 这不是一个“普通 CLI 工具”，而是一个“终端里的 agent 操作系统”
- `main.tsx` 更像启动编排器，不只是入口文件
- 真正的核心不是 UI，而是 5 层协作:
  1. 启动与环境层
  2. 会话与状态层
  3. 命令与工具层
  4. 查询与 agent 循环层
  5. 扩展与远程协作层
- 整个工程最强的地方不在“功能多”，而在“把复杂 agent 行为拆成了稳定接口”

---

## 1. 这个仓库到底是什么

先把最容易误解的点讲清楚。

### 1.1 它不是官方源码仓

顶层 `README.md` 已经说明了，这个仓库是基于公开 npm 包和 source map 还原出来的研究仓，不代表 Anthropic 的原始 monorepo 结构。

### 1.2 当前工作区里真正存在的东西

当前仓库里真正存在的核心文件其实非常少:

```text
README.md
package/
  README.md
  package.json
  cli.js
  cli.js.map
```

也就是说:

- `package/cli.js` 是真正发到 npm 的单文件打包产物
- `package/cli.js.map` 才是“可研究源码”的关键入口
- 你看到 README 里提到的 `restored-src/src/...`，在这个工作区里并没有现成落盘

### 1.3 所以分析方式必须变

这个工程不能按常规方式从 `src/` 直接开始读，而要这样看:

1. 从 `cli.js.map` 把源文件信息提出来
2. 看目录分布和入口
3. 沿着启动链路走
4. 再回头拆功能子系统

这也是你研究很多“只有 bundle、没有源码”的工程时可以复用的方法。

---

## 2. 从 sourcemap 反推出来的工程轮廓

### 2.1 总体规模

`cli.js.map` 中共有 `4756` 个 source 条目，其中:

- 约 `2378` 个来自 `node_modules`
- 约 `1902` 个来自 Claude Code 自己的 `src/`

这说明它不是一个“小而精”的 CLI，而是一个相当完整的产品级应用。

### 2.2 自身源码的高频目录

按 `src/` 一级模块粗略统计，比较大的目录有:

- `src/utils` - 564 个文件
- `src/components` - 389 个文件
- `src/commands` - 207 个文件
- `src/tools` - 184 个文件
- `src/services` - 130 个文件
- `src/hooks` - 104 个文件
- `src/ink` - 96 个文件

这组分布很说明问题:

- `utils` 很大，说明它有大量基础设施和中间层
- `commands` 和 `tools` 同时很大，说明“用户命令”和“模型工具”被明确分层
- `components + ink + hooks` 很大，说明它不是命令行脚本，而是完整 TUI 应用
- `services` 很大，说明网络、MCP、分析、策略、同步等能力都被单独服务化

### 2.3 顶层目录可以怎么理解

下面是更接近“认知地图”的解读:

| 目录 | 职责 |
|---|---|
| `main.tsx` | 总入口，负责参数解析、模式切换、启动编排 |
| `commands/` | Slash command 和 CLI 子命令能力 |
| `tools/` | 给模型调用的工具系统 |
| `query.ts` / `QueryEngine.ts` | 一轮 agent 运行的核心循环 |
| `screens/REPL.tsx` | 交互式终端主界面 |
| `cli/print.ts` | Headless/SDK 模式执行器 |
| `services/` | MCP、API、analytics、policy、compact 等服务 |
| `state/` | AppState 和状态同步 |
| `skills/` | 技能系统 |
| `plugins/` + `utils/plugins/` | 插件系统 |
| `tools/AgentTool/` | 子 agent、多 agent、teammate 体系 |
| `bridge/` + `remote/` | 远程桥接和会话同步 |
| `setup.ts` | cwd、worktree、tmux、session、hook 快照等启动准备 |

一句话总结:

> 这是一个把“终端 AI 助手”拆成了“启动器 + REPL + 查询引擎 + 工具总线 + 扩展平台”的系统。

---

## 3. 高层架构图

可以把 Claude Code 理解成下面这张分层图:

```text
User / SDK / Remote Client
        |
        v
  main.tsx / Commander
        |
        +-------------------+
        |                   |
        v                   v
 Interactive REPL      Headless / SDK
 screens/REPL.tsx      cli/print.ts
        |                   |
        +---------+---------+
                  |
                  v
          query.ts / QueryEngine.ts
                  |
     +------------+-------------+
     |                          |
     v                          v
 processUserInput         Tool orchestration
 slash commands           permissions / sandbox
 hooks / attachments      MCP / file / bash / agent
                  |
                  v
         AppState / session storage
                  |
     +------------+-------------+
     |            |             |
     v            v             v
   Skills       Plugins         MCP
   Agents       Workflows       Remote bridge
```

其中最关键的设计不是“层数多”，而是每一层的边界都很清楚:

- `main.tsx` 决定“跑哪种模式”
- `REPL` 和 `print.ts` 决定“用什么交互外壳”
- `query.ts` 决定“一轮 agent 如何推进”
- `tools.ts` 和 `commands.ts` 决定“给模型什么能力”
- `services/*` 决定“外部系统如何接进来”

---

## 4. 启动链路: 从 `claude` 命令到进入会话

这部分是全工程的主脉络。

### 4.1 最前面的几件事不是业务，而是性能和安全

`src/main.tsx` 顶部有几个非常值得学习的动作:

- 启动 profile checkpoint
- 提前启动 MDM 读取
- 提前做 keychain prefetch

这些都发生在大量 import 之前，目的只有一个:

> 把“同步慢操作”提前并行化，压缩首屏和首轮响应时间。

这说明团队把 CLI 启动时间看得非常重。

### 4.2 `preAction` 做统一初始化

`Commander` 的 `preAction` hook 里会做:

- 等待 MDM / keychain 预取结束
- 执行 `init()`
- 初始化日志 sink
- 接上 `--plugin-dir`
- 跑迁移
- 异步加载 remote managed settings / policy limits
- 启动 settings sync

这里的核心思想是:

> 把“所有模式都需要的初始化”集中在一处，而不是散落在每个命令处理器里。

### 4.3 `init()` 负责“安全可启动”

`src/entrypoints/init.ts` 不是做 UI 初始化，而是做运行时基础设施初始化:

- 启用配置系统
- 先应用“安全环境变量”
- 设置 graceful shutdown
- 初始化 analytics 的基础能力
- 预热 OAuth 信息
- 初始化 remote settings / policy limits 的 loading promise
- 配置 mTLS、代理、HTTP agent
- 预连接 Anthropic API
- 在远程环境里初始化 upstream proxy
- Windows shell 处理
- 注册清理函数

这里有个非常重要的安全分层:

- trust 之前: 只能应用 safe env
- trust 之后: 才应用完整 env

这意味着 Claude Code 把“目录信任边界”当成一等公民。

### 4.4 `setup()` 负责“当前会话落地”

`src/setup.ts` 做的是另一种初始化，它不偏全局，而偏“本次 session”:

- 设置 session id
- 启动 UDS 消息 socket
- 捕获 teammate 模式快照
- 恢复终端备份
- `setCwd()`
- 捕获 hook 配置快照
- 初始化 FileChanged watcher
- 创建 worktree / tmux
- 切换 projectRoot / originalCwd
- 初始化 session memory

你可以把它理解为:

> `init()` 让进程可运行，`setup()` 让当前会话可工作。

### 4.5 主入口同时支持很多模式

`main.tsx` 的默认 action 不是简单地“收 prompt 然后问模型”，它会根据参数进入不同分支:

- 交互式 REPL
- `-p/--print` headless 模式
- `mcp` 子命令
- `auth` 子命令
- `plugin` 子命令
- `doctor`, `update`, `install`
- `server`
- `ssh`
- `open cc://...`
- `assistant`
- `remote-control`

所以 `main.tsx` 更像一个“总调度器”。

---

## 5. 三种核心运行形态

整个系统虽然功能很多，但真正重要的是三种运行形态。

### 5.1 交互式 TUI 模式

入口链路:

`main.tsx -> createRoot() -> showSetupScreens() -> launchRepl() -> screens/REPL.tsx`

特点:

- 用 Ink 渲染完整终端 UI
- 可以显示消息列表、输入框、任务区、权限弹窗、通知
- 适合持续多轮对话和复杂 agent 行为

### 5.2 Headless / SDK 模式

入口链路:

`main.tsx -> cli/print.ts -> runHeadless() -> ask() / QueryEngine`

特点:

- 不创建 Ink root
- 支持 `text`、`json`、`stream-json`
- 支持 SDK 控制消息、权限请求、MCP server 注入、远端 transport
- 更像“agent runtime API”

### 5.3 子命令管理模式

例如:

- `claude mcp list`
- `claude auth status`
- `claude plugin install`
- `claude doctor`

特点:

- 很多命令会懒加载独立 handler
- 不一定会进入完整 REPL
- 更偏工具管理和环境操作

这个拆分很值得借鉴:

> 一套产品，三种交互层，底层能力尽量复用。

---

## 6. 关键抽象: Claude Code 是靠哪些“接口”站住的

### 6.1 `Command`

`src/types/command.ts` 里把命令分成 3 类:

- `prompt`
- `local`
- `local-jsx`

它们分别代表:

- `prompt`: 展开成一段给模型的 prompt，属于“模型可调用能力”
- `local`: 本地执行逻辑，返回文本或结构化结果
- `local-jsx`: 本地执行并渲染 UI

这意味着“slash command”并不是一种实现方式，而是一套统一抽象下的多种执行策略。

### 6.2 `Tool`

`src/Tool.ts` 里定义了工具系统的统一接口，`buildTool()` 给工具补默认行为。

工具至少会涉及这些概念:

- 名称与 schema
- `call()`
- 是否只读
- 是否可并发
- 权限检查
- 面向 auto-classifier 的输入
- 用户展示名

这层抽象让 Bash、Read、Edit、MCP、Agent、PlanMode、Task 等都能走统一管道。

### 6.3 `ToolUseContext`

这是整个系统最核心的上下文对象之一，里面装着:

- commands
- tools
- mainLoopModel
- mcpClients / resources
- appState 读写能力
- notifications / prompt / JSX / stream / fileHistory / attribution
- 当前 messages
- permission / denial / responseLength / compact progress

可以把它看成:

> agent 在执行工具时所拥有的“运行时世界”。

### 6.4 `AppState`

`src/state/AppStateStore.ts` 非常关键，它说明这个产品不只是聊天窗口。

它管理的状态包括:

- settings
- model / effort / thinking
- toolPermissionContext
- mcp clients / tools / commands / resources
- plugins
- tasks
- agents
- file history
- attribution
- todos
- notifications
- remote bridge
- companion / buddy
- tmux / browser / computer-use 等高级状态

这表明 Claude Code 的核心不是“单次问答”，而是“长期会话操作系统”。

### 6.5 `AgentDefinition`

agent 定义来源有:

- built-in
- user settings
- project settings
- managed settings
- plugin
- CLI flag

agent 可以带:

- prompt
- tools / disallowedTools
- skills
- mcpServers
- hooks
- permissionMode
- model / effort
- maxTurns
- memory
- background
- isolation

这意味着 agent 不是“硬编码角色”，而是可配置运行单元。

---

## 7. 命令系统: Claude Code 为什么不是“全靠模型自由发挥”

### 7.1 `commands.ts` 是统一注册中心

`src/commands.ts` 会把多类命令合并起来:

- built-in commands
- skill dir commands
- plugin commands
- plugin skills
- bundled skills
- workflow commands
- dynamic skills

`getCommands(cwd)` 是“当前用户在当前目录下真正可用命令”的最终结果。

### 7.2 slash command 不是花哨语法，而是“受控能力入口”

命令系统解决了两件事:

1. 给用户稳定入口
2. 给模型稳定能力边界

比如:

- `/review`
- `/plan`
- `/compact`
- `/memory`
- `/plugin`

这些能力不是靠模型临场拼 prompt，而是预制为命令对象，降低随机性。

### 7.3 命令还有来源优先级

从 `getActiveAgentsFromList()` 和命令合并逻辑可以看出，这套系统很强调覆盖顺序和来源优先级。

这类优先级治理在大规模可扩展产品里非常重要，否则插件、用户配置、项目配置会互相打架。

---

## 8. 工具系统: 真正让 agent 能“动手”的地方

### 8.1 工具池的构造逻辑

`src/tools.ts` 是工具总装配中心。

核心流程:

1. `getAllBaseTools()`
2. 根据模式过滤成 `getTools(permissionContext)`
3. 再与 MCP 工具合并成 `assembleToolPool(...)`

### 8.2 工具不是越多越好，而是按模式裁切

工具池会受这些因素影响:

- `simple` 模式
- `REPL` 模式
- `coordinator` 模式
- 权限 deny 规则
- feature gate
- env 开关

这说明 Claude Code 的工具系统不是静态白名单，而是“策略驱动的动态能力集”。

### 8.3 基础工具大致分组

你可以把内置工具分成几类:

文件与代码:

- `Read`
- `Edit`
- `Write`
- `NotebookEdit`
- `Glob`
- `Grep`
- `LSP`

执行与环境:

- `Bash`
- `PowerShell`
- `REPL`

Web 与外部:

- `WebFetch`
- `WebSearch`
- `ListMcpResourcesTool`
- `ReadMcpResourceTool`

协作与规划:

- `Agent`
- `SendMessage`
- `EnterPlanMode`
- `ExitPlanMode`
- `EnterWorktree`
- `ExitWorktree`

任务与追踪:

- `TodoWrite`
- `TaskCreate`
- `TaskGet`
- `TaskUpdate`
- `TaskList`
- `TaskStop`
- `TaskOutput`

### 8.4 工具执行是有并发控制的

`src/services/tools/toolOrchestration.ts` 会把工具调用分成两类:

- 可并发执行
- 必须串行执行

分批规则是:

- 连续的只读 / 并发安全工具可以并行
- 写操作和不安全操作串行

这背后的设计非常实用:

> 让模型“看起来会并发”，但实际执行仍有工程上的一致性保障。

### 8.5 还支持流式工具执行

`StreamingToolExecutor` 说明 Claude Code 不只是“拿到完整 assistant message 再执行工具”，还支持边流式接收边组织工具执行。

它处理了很多现实问题:

- 并发安全工具并行跑
- 非并发工具独占执行
- Bash 错误要中断兄弟任务
- 用户中断要合成错误结果
- 流式 fallback 时要丢弃未完成结果

这是非常成熟的 agent runtime 设计。

---

## 9. 一轮 agent 是怎么跑起来的

这部分是理解 Claude Code 的关键。

### 9.1 输入先走 `processUserInput`

`src/utils/processUserInput/processUserInput.ts` 会做这些事:

- 处理 prompt / pasted content / 图片
- 解析 slash command
- 生成 user message / attachment message
- 执行 `UserPromptSubmit` hooks
- 决定是否继续查询模型

也就是说，“用户输入”不是直接进模型，而是先被正规化成消息对象。

### 9.2 system prompt 不是一段字符串，而是一个组合产物

相关文件:

- `constants/prompts.ts`
- `utils/systemPrompt.ts`
- `utils/queryContext.ts`

system prompt 的来源会综合:

- 默认 Claude Code prompt
- custom system prompt
- append system prompt
- main thread agent prompt
- coordinator prompt
- output style
- hooks / 安全 / 行为约束
- MCP 指令

这说明 Claude Code 很重视“提示词构建器”，而不是把 prompt 当常量。

### 9.3 `query.ts` 才是真正的回合推进器

`src/query.ts` 是核心循环。

它负责:

- 规范化消息
- 计算 token 使用
- 判断是否需要 compact
- 向模型发请求
- 收集 stream event
- 执行工具
- 处理 tool result 回写
- 处理 stop hooks
- 控制 turnCount / budget / recovery

一句话:

> `query.ts` 是 Claude Code 的执行引擎。

### 9.4 `QueryEngine` 是 headless/SDK 对这套能力的封装

`src/QueryEngine.ts` 把 query loop 包装成“一个会话对象”:

- 一次构造，多轮 submit
- 保留 mutableMessages
- 保留 readFile cache
- 跟踪 usage
- 跟踪 permission denials

所以:

- REPL 更直接地围着 `query()` 转
- SDK / headless 更偏向围着 `QueryEngine` 转

这是非常典型的“双入口共用同一内核”做法。

---

## 10. 为什么它能长期对话而不爆上下文

### 10.1 工程里把 compact 当一级能力来做

相关模块:

- `services/compact/autoCompact.ts`
- `services/compact/compact.ts`

Claude Code 不是简单裁消息，而是有一整套上下文管理:

- warning threshold
- auto compact threshold
- blocking limit
- reactive compact
- session memory compaction
- post-compact cleanup

### 10.2 compact 不是“总结聊天”，而是“维护 agent 连续性”

它会考虑:

- 哪些消息应该保留
- 图片如何替换
- 哪些 attachment 之后会重新注入
- compact 后要恢复哪些文件上下文
- 技能、工具、MCP 指令怎样恢复

这说明 compact 不是 UX 附属功能，而是 agent runtime 的核心组成。

### 10.3 工程思想很值得借

一个成熟 agent 不该把上下文溢出当异常，而要把“长期会话下的上下文治理”当正常路径。

Claude Code 正是在这么做。

---

## 11. 权限、安全与信任模型

这是 Claude Code 和很多“随便跑 shell 的 AI 工具”最大的区别之一。

### 11.1 它把“目录信任”与“工具权限”分开

`showSetupScreens()` 的逻辑非常明确:

- trust dialog 负责“这个项目目录值不值得信任”
- permission mode 负责“工具调用时怎么批”

这两个边界不是一回事。

### 11.2 trust 前后环境变量处理不同

在 `init()` 和 `showSetupScreens()` 之间有明显的安全分界:

- trust 前只能应用 safe config env
- trust 后才应用 full env

这能防止不可信目录通过项目配置提前影响进程环境。

### 11.3 权限模式不是简单 yes/no

从 `permissionSetup.ts` 和 `permissions.ts` 看，系统支持的模式和逻辑相当丰富:

- default
- acceptEdits
- bypassPermissions
- plan
- auto
- dontAsk

还支持:

- allow / deny / ask 规则
- CLI 规则
- settings 来源规则
- tool 级、server 级、子 agent 级规则
- auto mode 下危险权限剥离

### 11.4 权限系统不是 UI 逻辑，而是 runtime 逻辑

它会参与:

- 工具是否展示给模型
- 工具是否允许执行
- 是否触发 classifier
- 是否走 sandbox override
- 是否需要 hook / approval

换句话说，权限不是“执行前的提示框”，而是“能力分发的一部分”。

---

## 12. MCP: Claude Code 的第一扩展平面

MCP 是这个工程非常核心的一层，不是附属插件。

### 12.1 MCP 配置来源很多

`services/mcp/config.ts` 会合并:

- `.mcp.json`
- 用户设置
- 项目设置
- managed settings
- plugin 提供的 MCP
- claude.ai connector
- CLI 的 `--mcp-config`

### 12.2 还会做“内容级去重”

代码里不是只按 server 名去重，而是根据:

- stdio command signature
- url signature

来判断两个 server 是否本质上是同一个。

这很聪明，因为:

- 插件和手写配置可能名字不同，但底层是同一 server
- claude.ai connector 和本地配置也可能重复

### 12.3 MCP 在 Claude Code 里不是只有 tool

MCP 能带来:

- tools
- commands
- resources
- prompts
- auth flow
- elicitation

所以它是完整扩展协议，而不是单纯“远程工具接入”。

### 12.4 MCP 客户端层也很完整

`services/mcp/client.ts` 支持多种 transport:

- stdio
- SSE
- streamable HTTP
- WebSocket
- SDK control transport

并处理:

- OAuth
- 401 刷新
- needs-auth cache
- tool result persistence
- MCP 输出截断与保存

这说明 Claude Code 基本把 MCP 当作“统一外设总线”。

---

## 13. Skills 和 Plugins: 第二、第三扩展平面

### 13.1 Skill 本质上是“Markdown 驱动的 prompt command”

`skills/loadSkillsDir.ts` 说明 skill 的核心机制是:

- 从目录加载 markdown
- 解析 frontmatter
- 生成 prompt command

它可以声明:

- description
- allowed-tools
- arguments
- when_to_use
- model
- effort
- hooks
- context=fork
- agent
- paths
- shell

这套设计非常强，因为:

- skill 不一定要改代码
- skill 可以逐步演化成更强的 agent 能力
- skill 可以被模型发现和调用

### 13.2 Plugin 比 skill 更像“能力包”

从 `utils/plugins/*` 和 schema 可以看出，plugin 不只装命令。

它大概率可贡献:

- commands
- skills
- agents
- hooks
- MCP servers
- LSP 集成
- 插件选项和持久化数据
- marketplace 分发信息

所以 plugin 是“产品扩展包”，skill 是“行为提示包”。

### 13.3 一个很强的设计点

Claude Code 没把“扩展系统”押注在单一机制上，而是分了三层:

1. skill - 最轻
2. plugin - 中等到重
3. MCP - 进程外 / 协议级

这能同时兼顾:

- 快速自定义
- 本地能力封装
- 远端系统接入

---

## 14. Agent、多 Agent、Coordinator: Claude Code 的高阶形态

这是整个工程最有代表性的部分之一。

### 14.1 `AgentTool` 不是普通工具，而是“递归执行器”

`tools/AgentTool/AgentTool.tsx` 和 `runAgent.ts` 表明:

- agent 可以由模型直接创建
- agent 可以同步跑，也可以后台跑
- agent 可以有自己的工具集、MCP、hooks、system prompt、memory、mode、隔离环境
- agent 还能继续 spawn 子 agent

这相当于把“再开一个 Claude 会话”做成了工具。

### 14.2 agent 有多种来源

除了 built-in，还有:

- 用户自定义 agent
- 项目 agent
- managed agent
- plugin agent
- CLI 注入 agent

也就是说 agent 角色本身是可配置资源。

### 14.3 built-in agents 是系统内置工作流角色

内置 agent 包括:

- general purpose
- statusline setup
- explore
- plan
- claude code guide
- verification

这说明 Anthropic 已经把“某些高频任务形态”抽象成固定 agent 角色。

### 14.4 coordinator mode 的本质

`coordinator/coordinatorMode.ts` 不是简单加个 flag，而是切换整套心智模型:

- 主 agent 变成协调者
- worker 可用工具集合被限定
- 任务通知通过内部消息回流
- 主 agent 的 system prompt 也换成 coordinator prompt

这在架构上非常干净:

> 不是“一个 prompt 教模型多代理”，而是“模式切换带来工具、提示、任务管道整体变化”。

### 14.5 任务系统是多 agent 的基础设施

从 `tasks/`、`LocalAgentTask`、`RemoteAgentTask`、`TaskOutputTool` 这些命名就能看出:

- agent 生命周期是显式建模的
- 任务进度、结果、前台/后台、输出文件、通知都有基础设施支撑

这比“临时 spawn 一次模型调用”成熟得多。

---

## 15. Remote、Bridge、Assistant: Claude Code 在做“跨端会话”

这部分说明 Claude Code 已经不是单机终端工具，而是在做跨端协作产品。

### 15.1 `RemoteSessionManager`

它负责:

- WebSocket 连接远端 session
- 收发 SDK message
- 处理 control request / permission request
- 通过 HTTP 把用户消息投递到远端 session

### 15.2 `remoteBridgeCore`

这里体现了更深的一层:

- 先创建 code session
- 再换 bridge 凭证
- 再建立 transport
- 再做 token refresh

从注释看，它已经演化到直接连 session ingress，不再依赖旧的 env dispatch 层。

### 15.3 这意味着什么

Claude Code 的终端 UI、远端 agent、Web 客户端之间，已经在往“统一会话协议”方向走。

也就是说它不只是本地 CLI，而是在建设:

> 一个可在终端、Web、远端执行环境之间迁移和继续的 agent session。

---

## 16. 性能工程: 这是一个被极致优化过的 CLI 产品

这是我认为整个工程里最容易被低估的点。

### 16.1 启动阶段优化非常激进

明显能看到的手段包括:

- import 前 side effect 并行预取
- startup profiler checkpoints
- lazy import 重度使用
- `print` 模式跳过大量子命令注册
- `setup()` 与 `getCommands()` 并行
- mcp config、file download、hooks、prefetch 交叠执行

### 16.2 prompt cache 稳定性被当成一级问题

代码里多次出现类似思路:

- 避免随机 temp path 破坏缓存
- built-in tools 与 MCP tools 合并时注意顺序稳定
- system prompt 明确划分 dynamic boundary
- forked agent 共享 cache-safe params

这说明团队非常清楚:

> 对 agent 产品来说，性能不仅是 CPU 和 IO，还包括 prompt cache 命中率。

### 16.3 headless 和 interactive 分开优化

例如:

- `--print` 跳过不必要 UI / subcommand 成本
- `--bare` 直接关闭很多预热和附加系统
- REPL 走首屏优先，headless 走流水线稳定输出优先

这不是“一个模式适配两个场景”，而是分别针对两个场景设计。

---

## 17. 我认为这套系统最核心的设计思想

如果你不是为了复刻 Claude Code，而是为了吸收其中的方法论，我建议重点抓下面 8 条。

### 17.1 把 AI 产品拆成“稳定接口”，不要把能力塞进 prompt

Claude Code 的真正能力不来自某个超长 system prompt，而来自一组稳定抽象:

- Command
- Tool
- ToolUseContext
- Query loop
- AppState
- AgentDefinition

这能让复杂行为持续演化，而不至于 prompt 越写越乱。

### 17.2 交互层和执行层解耦

- REPL 是交互层
- `query.ts` / `QueryEngine` 是执行层
- `main.tsx` 是启动编排层

这让 CLI、SDK、Remote 能复用同一个核心。

### 17.3 把“权限”前置成能力分发机制

很多工具只是“执行前弹个确认框”。

Claude Code 更进一步:

- 先决定模型能看到哪些工具
- 再决定能不能调
- 再决定要不要 classifier / ask / sandbox

这比事后拦截更稳。

### 17.4 长会话一定要内建上下文治理

不是等 context 爆了再补救，而是从一开始就设计:

- compact
- reactive compact
- memory
- file cache
- post-compact reinjection

这对做任何长期运行 agent 都非常关键。

### 17.5 扩展系统不要只有一层

Claude Code 同时拥有:

- skills
- plugins
- MCP

你自己的系统如果只做一种扩展机制，后面很容易卡死。

### 17.6 多 agent 不是“多开几个模型调用”，而是要有任务基础设施

Claude Code 的多 agent 能成立，是因为它有:

- agent 定义
- task 生命周期
- 结果通知
- permission 传播
- transcript / output / resume 机制

否则多 agent 很快就会沦为演示功能。

### 17.7 性能优化要围绕真实瓶颈

这套工程优化的不只是函数耗时，还有:

- 进程启动
- keychain / config / git 读取
- prompt cache 稳定性
- 首屏时间
- 首轮 tool-ready 时间

这说明他们是在“面向真实用户路径”优化。

### 17.8 注释里有大量“为什么”，这很值得学

这个工程的好注释很多，它们解释的是:

- 为什么要懒加载
- 为什么这里必须在 trust 之后
- 为什么这里要做 cache-safe 处理
- 为什么并发必须分区

这类注释的价值远高于“这行代码做什么”。

---

## 18. 学这套工程时最容易踩的坑

### 18.1 不要上来就读 UI

如果你先扎进 `components/` 和 `screens/REPL.tsx`，很容易觉得工程巨大而散。

正确路线应该是:

1. `main.tsx`
2. `entrypoints/init.ts`
3. `setup.ts`
4. `commands.ts`
5. `tools.ts`
6. `processUserInput.ts`
7. `query.ts`
8. `QueryEngine.ts`
9. 再回头看 MCP / plugin / agent / remote

### 18.2 不要把 slash command 和 tool 混为一谈

两者职责完全不同:

- command 负责“用户或模型如何触发能力”
- tool 负责“模型如何执行能力”

### 18.3 不要把 `main.tsx` 当成全部业务

`main.tsx` 很大，但它更多是在 orchestrate，不是在承载所有核心逻辑。

### 18.4 不要忽略注释

这个工程很多关键设计理由只写在注释里，不看注释会误判很多东西。

---

## 19. 推荐的学习路线

如果你真想“吃透逻辑并为己所用”，我建议按下面顺序读。

### 第一轮: 搭地图

读这些文件，目标是知道骨架:

1. `src/main.tsx`
2. `src/entrypoints/init.ts`
3. `src/setup.ts`
4. `src/commands.ts`
5. `src/tools.ts`
6. `src/state/AppStateStore.ts`

### 第二轮: 吃透主回路

读这些文件，目标是知道“一轮是怎么跑的”:

1. `src/utils/processUserInput/processUserInput.ts`
2. `src/query.ts`
3. `src/QueryEngine.ts`
4. `src/services/tools/toolOrchestration.ts`
5. `src/services/tools/StreamingToolExecutor.ts`
6. `src/utils/permissions/permissions.ts`

### 第三轮: 吃透扩展系统

读这些文件，目标是知道“能力如何长出来”:

1. `src/skills/loadSkillsDir.ts`
2. `src/utils/plugins/loadPluginCommands.ts`
3. `src/services/mcp/config.ts`
4. `src/services/mcp/client.ts`

### 第四轮: 吃透高阶能力

读这些文件，目标是知道“如何做复杂 agent 产品”:

1. `src/tools/AgentTool/AgentTool.tsx`
2. `src/tools/AgentTool/runAgent.ts`
3. `src/coordinator/coordinatorMode.ts`
4. `src/remote/RemoteSessionManager.ts`
5. `src/bridge/remoteBridgeCore.ts`
6. `src/services/compact/autoCompact.ts`
7. `src/services/compact/compact.ts`

---

## 20. 如果你想把里面的核心思想为己所用，最值得抄的不是功能，而是这 5 个骨架

### 20.1 一个统一的 `Tool` 抽象

先不要做一百个工具，先把工具接口定稳。

### 20.2 一个统一的 `Query / Agent Loop`

把“消息、模型、工具、权限、压缩”放在一个清晰循环里。

### 20.3 一个进程内 `AppState`

不要让每个功能自己存状态，否则 agent 产品会迅速失控。

### 20.4 一套分层扩展机制

建议至少区分:

- prompt skill
- local plugin
- protocol integration

### 20.5 把 trust / permission / sandbox 分开建模

这点非常重要。安全边界混在一起，最后一定难以维护。

---

## 21. 最后给一句总评

如果只用一句话概括这个工程，我会这样说:

> Claude Code 的本质，不是“把大模型塞进终端”，而是“把终端、会话状态、工具权限、上下文治理、扩展协议、多 agent 编排，做成了一个可以持续运行的 agent runtime”。

这也是它最值得借鉴的地方。

---

## 附录 A: 一个实用的阅读心法

阅读这种工程时，永远用下面这个问题去盯每个文件:

1. 它是“定义接口”还是“做编排”
2. 它是“运行时核心”还是“交互层外壳”
3. 它是“能力提供者”还是“能力注册表”
4. 它是在“安全边界之前”还是“安全边界之后”
5. 它是在“主线程会话”里，还是在“子 agent / 远程会话”里

一旦这五个问题答清楚，整个工程就不容易迷路。

## 附录 B: 如果你想复现本次分析

因为仓库里没有现成 `src/`，你需要自己从 `package/cli.js.map` 提取 `sourcesContent`。

核心思路就是:

1. 读取 `cli.js.map`
2. 遍历 `sources` 和 `sourcesContent`
3. 把 `../src/*` 写回本地目录
4. 再用 `rg` 和编辑器正常阅读

这套方法也适用于其他“只有 bundle + sourcemap”的项目。

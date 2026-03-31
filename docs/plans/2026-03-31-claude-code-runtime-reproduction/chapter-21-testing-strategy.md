# 第 21 章 测试策略

> 状态: 已写  
> 章节目标: 让 runtime 具备可回归、可验证、可迭代演进能力。

[返回总览](/Users/magongli/Downloads/project/claude-code-sourcemap/docs/plans/2026-03-31-claude-code-runtime-reproduction/README.md)

---

## 21.0 本章结论

Claude Code 风格 runtime 的测试，不能只靠几个单元测试。  
因为这个工程同时包含：

- CLI 入口
- 流式模型调用
- 工具调度
- 权限判定
- prompt/context 拼装
- session transcript
- background task
- MCP / plugin / remote

因此它必须是一个分层测试体系，而不是单点测试体系。

如果只测纯函数，你会漏掉：

- query loop 与 tool orchestration 的回归
- prompt 边界变化导致的行为漂移
- resume / compact / transcript 写入异常
- remote / permission relay 的异步问题

如果只测 E2E，你又会得到：

- 运行慢
- 失败难定位
- fixture 成本高
- 对模型与网络过度敏感

正确做法是构建 6 层测试网：

1. 单元测试
2. 合约测试
3. 集成测试
4. transcript / VCR replay
5. E2E 测试
6. 性能与稳定性测试

---

## 21.1 测试目标

这套 runtime 的测试至少要覆盖 5 类风险：

### 21.1.1 纯逻辑回归

例如：

- permission rule 解析错了
- compact 阈值判断错了
- tool schema 组装错了

### 21.1.2 组合行为回归

例如：

- prompt 变更后某个 command 不再生效
- tool result 没有正确写回 transcript
- plugin 注入导致命令池顺序变化

### 21.1.3 异步/生命周期回归

例如：

- background agent 无法完成
- permission request 悬挂
- remote reconnect 之后消息重复

### 21.1.4 协议与兼容性回归

例如：

- MCP server response 结构变化
- SDK stream-json 输出不兼容
- session resume 无法读取旧 transcript

### 21.1.5 性能回归

例如：

- startup 时间突然上涨
- TTFT 退化
- prompt dump 对主链路造成明显影响

---

## 21.2 测试分层模型

建议把测试体系组织成下面这个金字塔：

### 21.2.1 单元测试

覆盖纯函数与局部模块：

- message normalization
- permission rule matching
- path validation
- system prompt assembly
- compaction threshold logic
- plugin manifest schema validation

### 21.2.2 合约测试

覆盖“模块对外承诺”：

- Tool input/output schema
- Command metadata
- AgentDefinition loading
- MCP projection format
- remote control message shape

### 21.2.3 集成测试

覆盖小规模运行链闭环：

- `processUserInput -> query -> tool loop`
- `command -> prompt expansion -> query`
- `agent spawn -> task update -> transcript write`
- `permission request -> decision -> tool continue`

### 21.2.4 Replay / Fixture 测试

覆盖模型调用与 transcript 相关回归：

- API fixture replay
- transcript replay
- resume 行为验证
- compact 后上下文恢复验证

### 21.2.5 E2E 测试

覆盖用户真实入口：

- CLI interactive
- headless `--print`
- SDK stream-json
- remote/viewer

### 21.2.6 性能与稳定性测试

覆盖：

- startup
- TTFT
- 长会话 compact
- background task longevity
- reconnect/retry stability

---

## 21.3 单元测试应该测什么

单元测试的目标不是“多”，而是“卡住最容易漂移的纯逻辑”。

### 21.3.1 最值得先测的模块

建议优先覆盖这些：

- `system prompt` 拼装器
- `context` 裁切与边界插入
- `permission` 规则解析与匹配
- `filesystem` 路径规范化与危险目录识别
- `compact` 触发与结果对象生成
- `plugin manifest` / `agent frontmatter` schema
- `MCP config` merge / dedup
- `remote message` 判别与 decode

### 21.3.2 单元测试的写法建议

这类测试应尽量：

- 不依赖真实模型
- 不依赖真实网络
- 不依赖真实终端 UI
- 只验证输入输出与边界行为

举例：

- 给出 5 条 permission rule，验证某个 `Bash(git:*)` 命令匹配结果
- 给出某个 session message 列表，验证 compact 触发阈值和恢复消息
- 给出大小写混杂路径，验证危险 `.claude` / `.git` 目录识别不被绕过

---

## 21.4 Tool Contract 测试

Claude Code 风格 runtime 的工具系统很强，但也意味着工具接口很容易在迭代中悄悄破坏兼容。

### 21.4.1 为什么需要 contract test

因为 tool 对模型来说就是 API。如果你改了 schema、名字、输出结构，可能不会编译报错，但模型行为会漂。

### 21.4.2 重点校验点

每个工具至少应测试：

- `name`
- `aliases`
- `description/prompt`
- `inputSchema`
- `outputSchema`
- permission classification
- max result limits

### 21.4.3 AgentTool / BashTool / File 系列要单独重点测

这些工具对 runtime 正确性影响最大：

- `AgentTool` 关系到多 agent
- `BashTool` 关系到权限与副作用
- `Read/Edit/Write` 关系到代码修改的核心路径

建议为这些高风险工具写快照式 contract：

- schema 的稳定序列化快照
- 示例输入输出 fixture
- permission mode 下的差异行为

---

## 21.5 集成测试应该如何组织

集成测试的目标，是在不启动完整 UI 的前提下，验证一小段真实运行链。

### 21.5.1 最小 query 闭环

建议先做一个“无 UI、假模型”的 query integration harness，覆盖：

- 输入一条 user message
- 进入 query loop
- 触发一个 tool_use
- 执行 fake tool
- 回写 tool_result
- 得到 assistant 最终消息

这套 harness 是整个工程最值钱的测试基础设施之一。

### 21.5.2 权限集成测试

用假工具或假 shell 请求覆盖：

- `allow`
- `deny`
- `ask`
- `updatedInput`
- pending/cancel

### 21.5.3 session/transcript 集成测试

覆盖：

- transcript 序列化
- resume
- fork
- compact boundary
- sidechain transcript

### 21.5.4 plugin / agent / MCP 组合测试

要验证的不只是各模块自己能跑，而是组合后不打架：

- plugin agent 能否进入 activeAgents
- plugin command 能否正确出现在命令池
- agent-specific MCP 能否在启动时连上并清理
- strict plugin-only policy 是否真能阻止用户来源配置

---

## 21.6 Transcript Replay 与 VCR Fixture

这是 Claude Code 风格工程非常值得复现的一种测试资产。

### 21.6.1 为什么需要 replay

因为很多关键回归不是纯逻辑错误，而是：

- prompt/context 变化
- tool result 序列化变化
- message UUID / transcript 结构变化
- API chunk 处理变化

这类问题最适合用 replay 来兜。

### 21.6.2 `services/vcr.ts` 的启发

源码里有 `withFixture()` 与 `withVCR()`，说明测试体系支持：

- 对任意计算结果做 fixture 缓存
- 对模型 API 输出做 VCR 缓存

其中 `withVCR()` 会：

- 规范化 messages
- 把输入脱水
- 生成 fixture 文件名
- 在测试或 CI 中优先回放 fixture
- fixture 缺失时提示重新录制

这正是复现工程非常值得直接采用的模式。

### 21.6.3 VCR 的价值

它解决 4 个难题：

- 避免测试依赖实时模型网络
- 减少成本
- 提高回归可重复性
- 把模型输出行为纳入版本控制

### 21.6.4 VCR 也要注意什么

不能简单把原始 prompt 和原始响应全部裸写进去，建议：

- 脱水高波动字段
- 隐去 requestId 等瞬态标识
- 保持 message UUID 在 hydrate 后唯一
- 明确 fixture 录制入口与刷新策略

源码里甚至专门处理了“恢复时 UUID 去重导致结果被误当重复”的问题，这就是 replay 级测试非常容易踩的坑。

---

## 21.7 E2E 测试策略

E2E 不是拿来代替其它测试的，而是证明“从用户入口看，这个系统真的能用”。

### 21.7.1 应该覆盖哪些入口

至少覆盖：

- CLI 交互模式
- `--print` / headless 模式
- SDK `stream-json`
- resume / continue
- command 执行
- basic remote flow

### 21.7.2 E2E 要尽量减少外部依赖

推荐做法：

- 模型层用 fake provider 或录制 fixture
- MCP 用本地 mock server
- shell 命令用临时目录仓库
- remote transport 用本地 loopback server

### 21.7.3 终端 E2E 的关注点

终端型 agent 产品的 E2E 不应只断言退出码，还要断言：

- 首屏内容
- streaming 输出节奏
- permission prompt 出现
- 继续运行或恢复后的 transcript

---

## 21.8 MCP 与 Plugin 的专门测试

这是后期最容易烂掉的两层，所以建议单独列出测试面。

### 21.8.1 MCP 集成测试

覆盖：

- config merge
- connect / reconnect
- auth required
- tool list projection
- resource/prompt 映射
- output truncation / elicitation retry

### 21.8.2 Plugin 集成测试

覆盖：

- manifest validation
- session-only plugin 装载
- built-in plugin 启用/禁用
- versioned cache path 解析
- duplicate detection
- plugin 命令/agent/hooks/MCP 贡献注册

---

## 21.9 性能与稳定性测试

### 21.9.1 性能基线

建议至少维护这些基线：

- startup time
- first token latency
- tool execution overhead
- compact trigger latency
- remote reconnect latency

### 21.9.2 稳定性场景

建议做长跑型测试：

- 长会话连续 50-100 turn
- 多次 compact
- background agent 并发
- reconnect during permission wait
- plugin/MCP 重复加载与清理

### 21.9.3 稳定性测试要特别关注资源泄漏

例如：

- transcript 文件句柄
- MCP client 清理
- background task 未注销
- pending permission request 未释放

---

## 21.10 推荐的测试矩阵

如果你真的准备复现，我建议按下面矩阵推进：

### 21.10.1 Phase 0-1

- 单元测试
- 最小 query 集成测试
- 文件/权限工具 contract 测试

### 21.10.2 Phase 2

- transcript / compact replay
- command/skill 测试
- session persistence 测试

### 21.10.3 Phase 3

- MCP mock server 集成测试
- plugin loader 集成测试

### 21.10.4 Phase 4

- agent lifecycle 测试
- coordinator/worker 通知测试
- worktree isolation 测试

### 21.10.5 Phase 5

- remote permission relay 测试
- reconnect/token refresh 测试
- viewer mode 测试

---

## 21.11 复现工程的最低测试基线

如果你资源有限，先至少保证这 8 类测试存在：

1. permission rule unit tests
2. prompt assembly unit tests
3. tool schema contract tests
4. 最小 query loop integration
5. transcript persistence/resume integration
6. compact replay tests
7. plugin manifest loader tests
8. one headless E2E path

只要这 8 类在，系统就已经有“能持续迭代”的底盘了。

---

## 21.12 你最应该借用的核心思想

这一章最值得你拿走的是 4 点：

1. agent runtime 的测试必须分层，不能迷信某一种测试。
2. transcript replay 和 VCR fixture 是这类系统非常高价值的测试资产。
3. 工具、权限、session、remote 都要以“契约”视角测试，而不是只看 happy path。
4. 性能回归和稳定性回归必须进入测试体系，而不是等线上出问题再手测。

这样你的复现工程才不仅能搭出来，也能一直改下去。

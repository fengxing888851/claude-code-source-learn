# 第 24 章 开放问题与待决策项

> 状态: 已写  
> 章节目标: 把尚未拍板但会显著影响实现路径的问题集中收口。

[返回总览](/Users/magongli/Downloads/project/claude-code-sourcemap/docs/plans/2026-03-31-claude-code-runtime-reproduction/README.md)

---

## 24.0 本章结论

一个复现工程最怕的，不是不会做，而是边做边改目标。  
所以在真正开工前，至少要把一批“会影响目录结构、状态模型、扩展策略和安全边界”的问题提前拍板。

本章不是纯讨论，而是给出：

- 问题描述
- 备选方案
- 我的推荐
- 若暂时不拍板，会带来什么成本

---

## 24.1 平台支持优先级

### 问题

第一版是否同时支持：

- macOS
- Linux
- Windows

### 备选方案

1. 三端同时支持
2. 先 macOS/Linux
3. 先只做 macOS

### 推荐

推荐 `先 macOS/Linux，再补 Windows`。

### 原因

- shell、路径、权限、文件系统大小写、UNC path、终端能力在 Windows 上会显著增加复杂度
- Claude Code 本身也有不少专门的 Windows 路径与权限逻辑
- 如果你的目标是先复现 runtime 思想，Windows 不该成为第一阶段阻塞项

### 延后成本

后补 Windows 会要求你：

- 统一 path abstraction
- 抽象 shell tool
- 重新验证 permission/path security

但这是可以接受的延后成本。

---

## 24.2 存储后端选型

### 问题

session、transcript、task、plugin cache 元数据是：

- 先用文件
- 还是直接上 SQLite / 数据库

### 备选方案

1. 全文件化  
JSON / JSONL / 目录结构

2. SQLite 混合  
transcript 用文件，索引用 SQLite

3. 一开始就数据库化

### 推荐

推荐 `先全文件化，后期按需加索引层`。

### 原因

- 这类 CLI/runtime 工程的天然单位就是文件与会话目录
- transcript 本来就适合 JSONL
- debug、迁移、备份都更简单
- 早期需求还不稳定，不值得先设计复杂 schema

### 什么时候该引入 SQLite

当你出现下面情况时再考虑：

- 会话量明显增大
- 需要复杂检索
- 需要后台索引
- 需要更快的 session picker / analytics 聚合

---

## 24.3 模型提供商抽象层要做多深

### 问题

你是要做：

- 一个几乎完全 provider-agnostic 的通用 agent runtime
- 还是先为单一 provider 做薄适配

### 备选方案

1. 深抽象，多 provider 完全统一
2. 薄抽象，围绕当前主 provider 优化
3. 完全不抽象，直接绑死 SDK

### 推荐

推荐 `薄抽象 + capability flags`。

### 原因

Claude Code 风格系统有很多能力并不是所有 provider 都等价支持，例如：

- 工具调用格式
- 流式 thinking / reasoning
- cache 行为
- 消息 role / content block 结构

如果一开始就追求“所有 provider 完全统一”，大概率会把很多一等能力抽象掉。

更好的做法是：

- 统一上层 `ModelAdapter`
- 同时暴露 provider capability
- 让 query engine 在必要处根据 capability 分支

---

## 24.4 Plugin Marketplace 的签名与信任策略

### 问题

plugin marketplace 是否必须在第一版就引入：

- 签名
- 白名单
- 信任链

### 备选方案

1. MVP 就做完整签名链
2. 先本地插件与 session-only 插件
3. 直接开放任意远程来源

### 推荐

推荐 `MVP 不做开放 marketplace，只支持本地插件与白名单来源`。

### 原因

- marketplace 是供应链问题，不只是下载 UI
- 一旦开放任意远程插件，安全与支持成本立刻飙升
- 你前面还没把 plugin loader、source policy、cache/version 做稳

### 后续建议

进入 marketplace 阶段后，至少考虑：

- manifest 签名
- known marketplace allowlist
- publisher identity
- install/update audit log

---

## 24.5 Remote 优先做 Viewer 还是完整双向控制

### 问题

remote 第一版是：

- 先做只读 viewer
- 还是直接做完整双向会话 + permission relay

### 备选方案

1. viewer first
2. full remote control first
3. 两者同时做

### 推荐

如果你的目标是“尽快得到可展示的远程能力”，推荐 `viewer first`。  
如果你的目标是“尽量接近 Claude Code runtime 本体”，推荐 `full control first`。

### 我的整体建议

对于复现工程，我更推荐：

1. 先把 transport 与 session continuity 搭出来
2. 先支持基本 viewer
3. 再补 permission relay 与完整控制

这样难度梯度更平滑。

### 风险

如果一开始直接做 full control，但没有把：

- request_id
- cancel
- reconnect
- pending permission state

设计好，很容易返工。

---

## 24.6 Compact 与 Memory 的最终职责边界

### 问题

系统里同时存在：

- compact
- long-term memory
- snapshot / summary

它们之间的边界怎么定？

### 推荐边界

- `compact`  
负责压缩当前会话上下文，服务于 token 预算与继续对话

- `memory`  
负责跨会话、跨任务保留长期偏好、规则、经验

- `snapshot/summary`  
负责冻结某一时刻的 repo 或任务状态，供未来参考

### 为什么要尽快拍板

因为一旦边界不清，很容易出现：

- compact 顺手写长期记忆
- memory 承担当前会话总结
- snapshot 被误当成实时事实

最后三个系统互相覆盖，维护非常痛苦。

---

## 24.7 Agent 默认能力是“最小化”还是“继承父级”

### 问题

spawn 一个 agent 时，默认是：

- 继承父级全部工具和上下文
- 还是以最小能力集启动，再按 definition 加能力

### 推荐

推荐 `默认最小化，再显式扩展`。

### 原因

- 更安全
- 更容易解释 agent 职责
- 更便于做 coordinator/worker 分层

如果默认继承全部能力，后面再收回来会很痛苦。

---

## 24.8 Plugin 与 Skill 的关系要不要完全统一

### 问题

skill 是否必须总是 plugin 的组成部分？

### 推荐

推荐 `运行时抽象相互靠近，但分发形态不强行统一`。

也就是说：

- skill 可以独立存在
- 也可以由 plugin 打包提供
- 但不要为了统一而强迫所有 skill 都经过 plugin 安装

### 原因

独立 skill 对个人定制和快速试验非常友好；  
plugin 则更适合团队分发和治理。

---

## 24.9 配置是“即时生效”还是“会话快照”

### 问题

很多设置项是否应该在会话中途变化后立刻生效？

### 推荐

推荐 `区分两类配置`：

- 会话级快照配置  
例如 hooks、permission mode、agent mode

- 可动态读取配置  
例如某些 feature gate、analytics sampling

### 原因

如果所有设置都即时生效，会让会话行为难以解释；  
如果所有设置都冻结，又会让运维与诊断不方便。

---

## 24.10 推荐现在就拍板的默认答案

如果你要尽快开工，我建议直接采用下面这组默认决策：

- 平台优先级: macOS/Linux first
- 存储后端: 文件优先
- 模型抽象: 薄适配 + capability flags
- marketplace: 延后，先本地插件
- remote: 先基础 transport/viewer，再补双向控制
- compact vs memory: 严格分层
- agent 默认能力: 最小化
- 配置生效: 会话级快照为主

这组默认答案能让你避免前期大量岔路。

---

## 24.11 你最应该借用的核心思想

这一章最值得你复用的是 3 点：

1. 架构里真正难的问题要在开工前拍板，不要留到一半发现方向错了。
2. 推荐默认值比“无限讨论开放性”更重要。
3. 复现工程最该避免的是同时打开太多高复杂度前线。

把这些待决策项提前定住，后面的实现路线会顺很多。

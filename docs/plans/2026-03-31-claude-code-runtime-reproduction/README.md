# Claude Code 风格 Agent Runtime 复现设计文档总览

> 目标: 复现一个“Claude Code 风格”的终端 Agent Runtime，优先复现其架构分层、运行链路、核心抽象和系统设计思想，而不是逐文件 1:1 还原 Anthropic 内部工程。

**文档形态:** 总览索引 + 分章文档
**上游研究基础:** [Claude Code Sourcemap 架构拆解与学习指南](../2026-03-31-claude-code-architecture-guide.md)
**单文档归档:** [2026-03-31-claude-code-runtime-reproduction-design.md](../2026-03-31-claude-code-runtime-reproduction-design.md)

---

## 使用说明

这套文档现在采用“总览 + 分章”的结构，目的有三个：

- 让总览文档只承担边界、导航和推进顺序，不再承载全部细节。
- 让每一章都可以独立扩写、独立阅读、独立持续维护。
- 让后续继续写第 13 章以后时，不会把整份文档堆成难以导航的单体。

推荐阅读顺序：

1. 先读本页，明确项目目标、范围和章节关系。
2. 再按章节顺序阅读，先看第 1 到第 12 章理解 runtime 核心。
3. 然后继续进入扩展平面章节，例如 command/skill、permissions、compact、MCP。

```mermaid
flowchart LR
  A[总览与边界<br/>第 1-6 章]
  B[核心抽象与主运行链<br/>第 7-12 章]
  C[扩展平面<br/>第 13-17 章]
  D[高级运行时能力<br/>第 18-20 章]
  E[落地与治理<br/>第 21-25 章]

  A --> B --> C --> D --> E
```

## 总体系统总装图

下面这张 `SVG` 是整套文档当前最核心的总装图。  
它不是简单的分层图，而是把三条最重要的主线一起放进来了：

- 中轴主执行骨干
- 左侧入口与模式
- 右侧扩展与高级能力挂载点
- 底部横切治理与基础设施

![Claude Code 风格 Agent Runtime 总体系统架构图](../../diagrams/claude-code-runtime-overall-system-architecture.svg)

如果你觉得总装图信息密度偏高，推荐先看下面这两张拆解图，再回来看总装图：

- `执行主链图`：只看一轮输入是怎么穿过 runtime 的
- `控制平面图`：只看权限、compact、MCP、agent、remote、telemetry 怎么挂在主链上

### 执行主链图

![Claude Code 风格 Agent Runtime 执行主链图](../../diagrams/claude-code-runtime-execution-main-flow.svg)

### 控制平面图

![Claude Code 风格 Agent Runtime 控制平面图](../../diagrams/claude-code-runtime-control-plane.svg)

---

## 设计目标与边界

### 总目标

设计并实现一个具备以下特征的终端 Agent Runtime：

- 有统一的 CLI 入口。
- 同时支持交互式 REPL 和 Headless / SDK 模式。
- 有清晰的消息模型、命令模型、工具模型和会话状态模型。
- 支持工具调用、权限控制、上下文压缩、会话持久化。
- 支持技能、插件、MCP 等扩展机制。
- 能逐步演进到多 Agent 和 Remote/Bridge 架构。

### 复现原则

- 复现“架构思想”和“运行机制”，不追求 bundle 级逐行复刻。
- 优先构建可维护、可理解、可扩展的 clean-room 实现。
- 先做统一运行时，再叠加高级能力。
- 所有高级能力都必须建立在稳定的核心抽象之上。

### 非目标

当前阶段不追求：

- 完整复刻 Anthropic 的私有后端协议。
- 复刻所有隐藏 feature gate。
- 完整复刻所有 UI 细节和产品文案。
- 一次性实现 marketplace、远程桥接、多 Agent 全部能力。

---

## 章节索引

- 第 1 章 [引言](./chapter-01-introduction.md) · 已写
- 第 2 章 [问题定义与设计原则](./chapter-02-problem-definition-and-principles.md) · 已写
- 第 3 章 [产品分层与版本路线](./chapter-03-product-phases-and-roadmap.md) · 已写
- 第 4 章 [系统总览](./chapter-04-system-overview.md) · 已写
- 第 5 章 [运行模式设计](./chapter-05-runtime-modes.md) · 已写
- 第 6 章 [启动与初始化设计](./chapter-06-bootstrap-and-initialization.md) · 已写
- 第 7 章 [核心领域模型](./chapter-07-core-domain-models.md) · 已写
- 第 8 章 [会话与状态管理设计](./chapter-08-session-and-state-management.md) · 已写
- 第 9 章 [用户输入处理设计](./chapter-09-user-input-processing.md) · 已写
- 第 10 章 [System Prompt 与上下文拼装设计](./chapter-10-system-prompt-and-context-assembly.md) · 已写
- 第 11 章 [Query Engine 设计](./chapter-11-query-engine.md) · 已写
- 第 12 章 [Tool 系统设计](./chapter-12-tool-system.md) · 已写
- 第 13 章 [Command / Skill 系统设计](./chapter-13-command-and-skill-system.md) · 已写
- 第 14 章 [权限、信任与沙箱设计](./chapter-14-permissions-trust-and-sandbox.md) · 已写
- 第 15 章 [上下文治理与 Compact 设计](./chapter-15-context-governance-and-compact.md) · 已写
- 第 16 章 [MCP 集成设计](./chapter-16-mcp-integration.md) · 已写
- 第 17 章 [Plugin 系统设计](./chapter-17-plugin-system.md) · 已写
- 第 18 章 [Agent 与多 Agent 设计](./chapter-18-agent-and-multi-agent.md) · 已写
- 第 19 章 [Remote / Bridge 设计](./chapter-19-remote-and-bridge.md) · 已写
- 第 20 章 [可观测性与诊断设计](./chapter-20-observability-and-diagnostics.md) · 已写
- 第 21 章 [测试策略](./chapter-21-testing-strategy.md) · 已写
- 第 22 章 [安全设计](./chapter-22-security-design.md) · 已写
- 第 23 章 [实现路线图](./chapter-23-implementation-roadmap.md) · 已写
- 第 24 章 [开放问题与待决策项](./chapter-24-open-questions.md) · 已写
- 第 25 章 [附录](./chapter-25-appendix.md) · 已写

---

## 推进顺序

### 第一批: 定全局边界

- 第 1 章 引言
- 第 2 章 问题定义与设计原则
- 第 3 章 产品分层与版本路线
- 第 4 章 系统总览
- 第 5 章 运行模式设计
- 第 6 章 启动与初始化设计

### 第二批: 定核心抽象与主运行链

- 第 7 章 核心领域模型
- 第 8 章 会话与状态管理设计
- 第 9 章 用户输入处理设计
- 第 10 章 System Prompt 与上下文拼装设计
- 第 11 章 Query Engine 设计
- 第 12 章 Tool 系统设计

### 第三批: 定扩展平面

- 第 13 章 Command / Skill 系统设计
- 第 14 章 权限、信任与沙箱设计
- 第 15 章 上下文治理与 Compact 设计
- 第 16 章 MCP 集成设计
- 第 17 章 Plugin 系统设计

### 第四批: 定高级能力

- 第 18 章 Agent 与多 Agent 设计
- 第 19 章 Remote / Bridge 设计
- 第 20 章 可观测性与诊断设计
- 第 21 章 测试策略
- 第 22 章 安全设计
- 第 23 章 实现路线图
- 第 24 章 开放问题与待决策项
- 第 25 章 附录

---

## 当前状态

目前已经完成：

- 总览索引搭建。
- 第 1 到第 25 章拆分并写成独立文档。
- 第 1 到第 12 章已补充“本章对复现工程的直接指导”，前半段不再只是总览层说明。
- 单文档版本保留为归档快照。

下一步建议进入：

- 基于第 23 章，把复现工程拆成真正可执行的任务清单
- 基于第 25 章，生成目录结构与模块脚手架
- 基于第 21/22 章，先把测试与安全底座一起搭起来

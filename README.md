# Claude Code Source Learn

基于 `@anthropic-ai/claude-code@2.1.88` npm 包及其 sourcemap 还原出的源码研究与架构学习工程。

> **声明:** 本仓库不是 Anthropic 官方源码仓，所有内容均来自公开发布的 npm 包产物，仅用于个人学习和架构研究。

## 项目目标

1. **拆解架构** — 理解 Claude Code 作为"终端 Agent Runtime"的整体分层设计和运行链路。
2. **提炼方法论** — 从中提取可迁移到自有项目的 agent 系统设计思想。
3. **复现设计** — 基于拆解结果，产出一套完整的 Claude Code 风格 Agent Runtime 复现设计文档。

## 目录结构

```
docs/
├── diagrams/                        # 架构图 (SVG)
│   ├── claude-code-runtime-control-plane.svg
│   ├── claude-code-runtime-execution-main-flow.svg
│   └── claude-code-runtime-overall-system-architecture.svg
├── plans/
│   ├── 2026-03-31-claude-code-architecture-guide.md   # 架构拆解与学习指南
│   ├── 2026-03-31-claude-code-runtime-reproduction-design.md  # 复现设计单文档归档
│   └── 2026-03-31-claude-code-runtime-reproduction/   # 复现设计分章文档
│       ├── README.md                # 总览索引
│       ├── chapter-01 ~ chapter-25  # 分章详细设计
```

## 核心文档导读

| 文档 | 内容 |
|------|------|
| [架构拆解指南](docs/plans/2026-03-31-claude-code-architecture-guide.md) | 从 sourcemap 反推出的 5 层架构分析：启动与环境、会话与状态、命令与工具、查询与 agent 循环、扩展与远程协作 |
| [Runtime 复现设计](docs/plans/2026-03-31-claude-code-runtime-reproduction/) | 25 章分章设计文档，涵盖从引导初始化到安全设计的完整 runtime 复现方案 |
| [架构图](docs/diagrams/) | 系统总体架构、主执行流程、控制平面 三张 SVG 图 |

## 关键发现

- Claude Code 本质上是一个 **终端里的 Agent 操作系统**，而非普通 CLI 工具
- 核心竞争力在于将复杂 agent 行为拆分为稳定接口的分层设计
- 五层协作架构：启动与环境 → 会话与状态 → 命令与工具 → 查询与 Agent 循环 → 扩展与远程协作

## License

本仓库仅供学习研究使用。Claude Code 的原始代码版权归 Anthropic 所有。

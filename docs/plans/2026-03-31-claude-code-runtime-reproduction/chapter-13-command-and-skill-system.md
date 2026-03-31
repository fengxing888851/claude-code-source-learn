# 第 13 章 Command / Skill 系统设计

> 状态: 已完成初稿
> 章节目标: 定义用户入口和模型可触发能力的统一抽象。

[返回总览](/Users/magongli/Downloads/project/claude-code-sourcemap/docs/plans/2026-03-31-claude-code-runtime-reproduction/README.md)

---

这一章要讲清楚 Claude Code 风格系统里另一个特别容易被误解的点：

> `Command` 不是 `Tool` 的别名，`Skill` 也不是一段 prompt 文本而已。

如果说 `Tool` 解决的是“模型如何安全使用能力”，那 `Command / Skill` 解决的就是“用户如何进入系统、系统如何把高层能力组织成可发现、可复用、可注入的入口平面”。两者是互补关系，而不是谁替代谁。

## 13.1 Command 系统在总架构中的位置

在 Claude Code 风格架构里，`Command` 主要位于用户入口侧：

```text
user input
  -> slash command parse
  -> command lookup
  -> local / local-jsx / prompt branch
  -> mutate session or produce prompt
  -> optional query continuation
```

这意味着 command 系统的职责是：

- 帮用户显式触发能力。
- 把某些复杂操作包装成稳定入口。
- 把“高阶 prompt 能力”模块化。
- 把本地控制平面与模型执行平面隔开。

上游把 `commands.ts` 做得非常大，不是偶然，而是因为它承载了整套用户入口编排。

## 13.2 三类命令为什么必须分开

从 `types/command.ts` 可以看到，上游把命令正式拆成三类：

- `prompt`
- `local`
- `local-jsx`

这是一个非常重要的架构决定。

### 13.2.1 `prompt` 命令

`prompt` 命令的本质不是直接执行某件事，而是生成一段结构化 prompt 内容，最终仍会进入 Query Engine。

它的典型用途包括：

- skill
- review / security-review / plan 这类高阶智能任务
- workflow-backed commands
- 一些 prompt 型插件命令

这类命令最关键的属性不是“run()”，而是：

- `getPromptForCommand(args, context)`
- `allowedTools`
- `model`
- `effort`
- `hooks`
- `context`
- `agent`

也就是说，它更像“受控 prompt 程序”。

### 13.2.2 `local` 命令

`local` 命令是纯本地执行入口，不必进入模型。例如：

- 查看配置
- 打印状态
- 修改本地设置
- 输出帮助信息

它们的典型返回值是：

- 文本
- compact 结果
- skip

这说明 local command 解决的是控制平面问题，而不是智能推理问题。

### 13.2.3 `local-jsx` 命令

`local-jsx` 是一个很 Claude Code 风格的设计，它说明有些命令并不是纯文本输出，而是要在终端里渲染临时交互 UI。

例如这类命令可能会：

- 打开一个 Ink 界面
- 展示选择器
- 弹出本地交互流
- 完成后返回 `onDone()`

这个类型存在的价值非常大，因为它阻止了两个坏结果：

- 不把所有本地交互硬塞进 REPL 主组件。
- 不把本地交互命令误当成可远程、可桥接、可模型调用的能力。

## 13.3 `Command` 的关键属性设计

从上游 `CommandBase` 和 `PromptCommand` 可以看出，一个成熟命令对象不只是 `name + description`。

建议关注这些关键属性：

- `availability`
- `isEnabled`
- `isHidden`
- `aliases`
- `disableModelInvocation`
- `userInvocable`
- `loadedFrom`
- `source`
- `kind`
- `immediate`
- `isSensitive`

这些属性的设计意义如下。

### 13.3.1 `availability` 与 `isEnabled` 不是一回事

上游专门区分：

- `availability`: 从 auth/provider 角度，谁可以看见或使用这个命令。
- `isEnabled()`: 当前环境下，这个命令是否启用。

这点很成熟，因为：

- 有些命令只对某类订阅或认证环境可见。
- 有些命令在产品上可用，但当前 feature flag 没开。

如果把这两者混成一个 `enabled`，后面会很难解释“是没权限、没资格、没开关，还是暂时不可用”。

### 13.3.2 `loadedFrom` 与 `source`

上游的命令来源不止一种：

- builtin
- skills
- plugin
- bundled
- mcp
- commands_DEPRECATED

这类来源信息非常重要，因为它决定：

- 在 UI 里如何标注来源。
- 在模型可调用面里是否可见。
- 加载失败时该如何降级。
- 出问题时该去哪个系统排查。

### 13.3.3 `disableModelInvocation` 与 `userInvocable`

这两个字段说明 Claude Code 风格系统非常清楚“用户能不能用”和“模型能不能用”是两件事。

例如一个 skill 可能：

- 用户可以手动 `/skill-name` 调用。
- 但模型不应该在 SkillTool 里自动调用它。

或者反过来：

- 用户未必显式输入。
- 但模型可以在能力发现后调用。

这种区分对能力治理非常重要。

## 13.4 commands.ts：命令池装配中心

`commands.ts` 的核心价值，不是声明了很多命令，而是它定义了命令池的总装配逻辑。

从 sourcemap 可见，命令来源大致包括：

- 内置 `COMMANDS()`
- skill 目录命令
- plugin skills
- bundled skills
- built-in plugin skills
- workflow commands
- dynamic skills
- MCP skills

而且这个装配过程不是平铺的，它至少做了这些事情：

- 记忆化昂贵加载路径
- 逐次重新评估 `availability`
- 逐次重新评估 `isEnabled`
- 在 built-in commands 之前插入 dynamic skills
- 把 MCP skills 单独筛出给 skill index 等使用

这意味着 command registry 不是静态常量，而是会随会话状态和环境变化而变化的能力视图。

## 13.5 `getCommands()` 的两段式设计

上游的 `getCommands(cwd)` 很值得借鉴，因为它不是每次都做全部磁盘扫描，而是两段式：

- `loadAllCommands(cwd)`：昂贵部分做记忆化。
- `getCommands(cwd)`：每次 fresh 地做 availability / enabled 过滤与 dynamic skills 注入。

这个设计非常实用：

- 重量级 I/O 和动态 import 被缓存。
- 登录状态、provider 状态、feature 开关变化又能及时反映。

它体现了一个重要设计原则：

> 能缓存的是“发现结果”，不能缓存的是“当前会话是否可见、是否可用”。

## 13.6 Skill 为什么本质上是 Markdown 程序

从 `skills/loadSkillsDir.ts` 可以看得很清楚，Skill 不是随便一段 markdown，而是一个被 frontmatter 驱动的 prompt program。

它至少能表达这些字段：

- `description`
- `allowed-tools`
- `argument-hint`
- `arguments`
- `when_to_use`
- `version`
- `model`
- `disable-model-invocation`
- `user-invocable`
- `hooks`
- `context`
- `agent`
- `effort`
- `paths`
- `shell`

这意味着 Skill 在系统中的角色非常正式：

- 它可被发现。
- 它可被筛选。
- 它可被模型理解。
- 它可带执行策略。
- 它甚至可决定是在当前上下文 inline 执行，还是 fork 到子 agent。

这已经远远超出“可复用 prompt 模板”的范畴了。

## 13.7 frontmatter 设计为什么重要

上游对 skill frontmatter 的处理很成熟，尤其有几点特别值得借鉴。

### 13.7.1 描述不是可有可无

系统会从：

- frontmatter `description`
- markdown 第一段描述

中提取可读简介。这说明 Skill 的人机可发现性是一级能力，而不是附属文档。

### 13.7.2 `allowed-tools` 是 skill contract 的一部分

这点很关键。Skill 不是让模型无限发挥，而是显式声明“这段能力应该配什么工具集”。这让 skill 从“提示词技巧”升级成了“能力约束模块”。

### 13.7.3 `context: fork`

这一条尤其重要。它意味着 skill 可以不在当前对话里 inline 展开，而是转成 forked sub-agent 去执行。换句话说，skill 不只是 prompt 内容，也是 agent orchestration 的入口。

### 13.7.4 `paths`

上游还支持按路径模式来控制 skill 的适用性，说明 skill 发现系统会结合“你碰过哪些文件”来动态浮现相关能力。这一点是非常典型的“runtime-aware skill system”。

## 13.8 skill 的来源与作用域

从加载器实现看，skill 可能来自多个层次：

- 用户级目录
- 项目级 `.claude/skills`
- managed / enterprise 配置
- plugin 自带 skill
- bundled skill
- MCP skill

这意味着 skill 系统本身也是一个多来源装配平面。它和工具、MCP 的设计思路非常一致：

- 来源可以多样
- 进入系统后必须统一成 `Command`

这种统一非常关键，不然技能系统会很快碎成多套互不兼容的“提示词插件”。

## 13.9 plugin 命令为什么要命名空间化

从 `loadPluginCommands.ts` 能看到，上游会把插件命令命名成类似：

```text
pluginName:namespace:command
```

而 skill 目录下的 `SKILL.md` 也会根据父目录结构推导命令名。

这样设计的好处很明显：

- 避免命名冲突。
- 让来源可见。
- 让用户知道能力归属。
- 让权限与调试更容易定位。

这是一种很值得复现的设计，因为多来源命令系统迟早会遇到同名问题。

## 13.10 dynamic skills：为什么要在运行时插入

Claude Code 风格系统里很有代表性的一个设计，是 `dynamicSkills`。

它们不是静态命令表的一部分，而是在运行时根据文件操作、路径匹配、任务上下文浮现出来，然后插入命令池中。

上游甚至明确把它们插在：

- plugin / skill commands 之后
- built-in commands 之前

这说明 dynamic skills 在产品定位上处于“当前任务相关的增强层”，而不是基础命令层。

这类设计很有价值，因为它让系统能够做到：

- 不一次性把所有技能塞给模型。
- 只在相关任务时浮现相关能力。
- 把 skill system 从静态目录提升为上下文感知层。

## 13.11 `getSkillToolCommands()` 与 `getSlashCommandToolSkills()` 的区别

这是一个很容易混淆、但很体现成熟度的点。

上游并没有只做一个“技能列表”函数，而是至少分成：

- `getSkillToolCommands()`
- `getSlashCommandToolSkills()`
- `getMcpSkillCommands()`

这背后意味着系统对不同消费方有不同视图：

- SkillTool 需要“模型可以调用的 prompt 型能力”。
- slash command picker 需要“用户会输入 `/skill-name` 的技能集合”。
- MCP skill 则是额外来源，不一定直接进入普通 `getCommands()` 结果。

这说明命令系统不是一个列表，而是一组按消费场景裁切的视图函数。

## 13.12 remote-safe 与 bridge-safe 策略

从 `commands.ts` 可以清楚看到，Claude Code 风格系统对远程/桥接模式下的命令执行是非常谨慎的。

### 13.12.1 `REMOTE_SAFE_COMMANDS`

这是用于 `--remote` 模式的预过滤集合，只保留那些不会依赖本地文件系统、git、shell 或本地 Ink 状态的命令。

### 13.12.2 `BRIDGE_SAFE_COMMANDS`

这是更细的策略：

- `prompt` 命令默认安全，因为它们只是扩展成文本送给模型。
- `local` 命令需要显式进入 allowlist。
- `local-jsx` 一律不安全，因为它们依赖本地终端 UI。

这套策略很值得借鉴，因为它说明：

- “是不是 slash command”不重要。
- “这个命令依赖哪一层运行时能力”才重要。

## 13.13 命令系统的推荐分层

复现时建议把命令系统拆成以下层次。

### 13.13.1 Command Definition Layer

定义三类命令对象及其 schema。

### 13.13.2 Command Source Layer

从 builtin、skills、plugins、bundled、workflows、MCP 收集命令。

### 13.13.3 Command Registry Layer

统一命名、去重、缓存、按来源标注。

### 13.13.4 Command View Layer

针对：

- 用户 slash picker
- SkillTool
- 模型 prompt 型能力发现
- remote mode

提供不同过滤视图。

### 13.13.5 Command Execution Layer

按 `prompt` / `local` / `local-jsx` 分流执行，并与输入处理层和 Query Engine 对接。

## 13.14 本章结论

第 13 章最重要的结论有四个：

- `Command` 是用户入口系统，不是 `Tool` 的变体。
- `Skill` 本质上是 frontmatter 驱动的 prompt program，而不是 markdown 文本。
- 命令系统是多来源装配平面，必须通过统一 registry 管理。
- remote-safe / bridge-safe 这类策略说明命令系统本身就是安全边界的一部分。

## 13.15 本章对复现工程的直接指导

如果你准备真正开始实现命令层，建议按下面的顺序做，而不要一上来就把 slash command、skill、plugin command 混成一个入口。

### 13.15.1 先冻结 `Command` 的三分法

第一版就把下面三类明确成正式枚举：

- `prompt`
- `local`
- `local-jsx`

不要用一个 `Command` 然后靠 `if (type === "...")` 到处散落判断。命令分流一旦失控，后面输入处理、remote-safe、权限策略都会变得很难解释。

### 13.15.2 先做统一 registry，再做 picker 或 UI

优先顺序应该是：

1. `CommandSource`
2. `CommandRegistry`
3. `CommandExecution`
4. `CommandView`

不要反过来先做 `/help`、补全列表、命令面板，再倒逼后面拼一个 registry。那样很快会出现“UI 里能看到，runtime 里不一定能执行”的双轨问题。

### 13.15.3 Skill 先当成受限命令源，不要先当插件系统

复现时最稳的第一步是：

- skill 文件只用 markdown + frontmatter
- 只贡献 `prompt` 类命令或受控的 local 能力
- 先走本地目录发现，不急着做 marketplace

这样你能先把 `skill -> command` 这条投影链跑通，而不是一开始就陷入安装、签名、缓存和来源治理。

### 13.15.4 命令安全属性必须进模型，不要做成旁路表

至少把这些属性作为正式字段留出来：

- `source`
- `loadedFrom`
- `remoteSafe`
- `bridgeSafe`
- `disableModelInvocation`
- `userInvocable`

这些字段不是“文档注释”，而是 runtime 过滤和分流的依据。后面 remote、viewer、skill tool 和 prompt 注入都会用到。

### 13.15.5 第一版推荐目录

建议最少拆成：

```text
commands/
  types.ts
  sources/
  registry/
  execution/
  views/
skills/
  discovery/
  parsing/
  projection/
```

这里真正要避免的错误，是把 skill 解析逻辑、命令去重逻辑和命令执行逻辑堆进一个文件里。命令层只要一变大，就很容易变成第二个“看起来简单，实际上无处不在”的隐形核心模块。

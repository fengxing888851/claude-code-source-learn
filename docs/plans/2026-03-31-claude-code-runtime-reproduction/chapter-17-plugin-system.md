# 第 17 章 Plugin 系统设计

> 状态: 已写  
> 章节目标: 把“本地扩展包”设计成一等扩展平面，而不是零散脚本目录。

[返回总览](/Users/magongli/Downloads/project/claude-code-sourcemap/docs/plans/2026-03-31-claude-code-runtime-reproduction/README.md)

---

## 17.0 本章结论

在 Claude Code 风格 runtime 里，`plugin` 不是“安装几个 prompt 文件”这么简单。它本质上是一个经过校验、可缓存、可启停、可策略管控的扩展包格式，用来把一组相关能力一起交付进系统。它解决的不是“怎么多加一个命令”，而是下面这几个更工程化的问题：

- 如何把 `commands / agents / hooks / MCP servers / skills` 这类能力以一个可发布、可安装、可升级的单元分发。
- 如何让扩展具备明确的来源信息、版本信息、启用状态和错误状态。
- 如何在不信任用户输入的前提下，把插件装载进一个有权限模型、会话模型和远程模型的 agent runtime。

因此，复现时你不要把 plugin 想成“命令目录扫描器”。更准确的理解是：`plugin = 扩展包管理层 + 扩展注册层 + 策略约束层`。

---

## 17.1 为什么需要 Plugin，而不是只靠 Skill / Command

前面几章已经有 `Command`、`Skill`、`MCP`。那为什么还要 `Plugin`？

因为这几者解决的问题不同：

- `Command` 解决“用户怎样显式触发一个能力”。
- `Skill` 解决“怎样把一段可复用提示模板注入系统”。
- `MCP` 解决“怎样把外部工具和资源通过协议接入”。
- `Plugin` 解决“怎样把一组扩展作为一个带生命周期的发行单元交付和治理”。

也就是说，plugin 更像“安装包”，而 command/skill/agent/hook 更像“包里的内容”。

这也是 Claude Code 设计里很关键的一层抽象：  
上层扩展能力不直接散落在用户目录里，而是尽量被包裹进一个带 manifest 的扩展单元。这样系统才有机会处理：

- 依赖来源是谁。
- 版本号是什么。
- 是否启用。
- 是否来自官方 marketplace。
- 缓存在哪里。
- 升级时如何复用旧缓存。
- 报错时如何定位是 manifest 问题、拉取问题还是组件加载问题。

如果你复现时直接让用户随手扔几个目录进来，短期能跑，但长期一定会在治理上失控。

---

## 17.2 Plugin 的核心定位

从 `pluginLoader.ts`、`schemas.ts` 和 `builtinPlugins.ts` 可以反推出，plugin 系统承担了 3 类职责：

### 17.2.1 发行职责

把扩展打包成一个稳定单位，包含：

- manifest 元数据
- commands
- agents
- hooks
- MCP 配置
- 可选 skills

### 17.2.2 运行时接入职责

把插件内容投影进已有 runtime 抽象里，而不是另起一套系统：

- 插件命令最终仍然变成 `Command`
- 插件 agent 最终仍然变成 `AgentDefinition`
- 插件 MCP 最终仍然走统一 MCP client 管理器
- 插件 hooks 最终仍然注册到统一 hooks 体系

这点非常重要。插件只是“贡献能力”，它不应该拥有一条完全独立的执行通道。

### 17.2.3 治理职责

插件来源、启用状态、缓存路径、策略限制、错误收集都在这一层统一处理。

因此，plugin loader 的真正价值不是“读目录”，而是：

- 做 schema 验证
- 做来源约束
- 做路径合法性校验
- 做重复项处理
- 做缓存复用
- 做状态归档

---

## 17.3 Plugin Manifest 设计

### 17.3.1 Manifest 为什么必须存在

如果没有 manifest，系统无法回答这些问题：

- 这个插件叫什么
- 版本是多少
- 来自哪里
- 作者是谁
- 是否允许从这个来源安装
- 它是否声明了 hooks 或 MCP

所以复现版一定要有 manifest。哪怕你第一阶段只支持本地目录插件，也要先把 manifest 抽象建起来。

### 17.3.2 Manifest 至少应包含哪些字段

从 `PluginManifestSchema` 可以看出，manifest 不是只有 `name/version`，而是一个比较完整的包描述：

- `name`
- `version`
- `description`
- `author`
- `homepage`
- `repository`
- `license`
- `keywords`
- `dependencies`

这说明 plugin 被当作“正式扩展发行物”而不是临时目录。

对复现版来说，推荐拆成两层字段：

1. 识别层

- `name`
- `version`
- `id`
- `marketplace`

2. 元数据层

- `description`
- `author`
- `homepage`
- `repository`
- `license`
- `keywords`

3. 贡献声明层

- `commands`
- `agents`
- `hooks`
- `mcpServers`

### 17.3.3 Manifest 校验为什么要严格

`schemas.ts` 里有几个很能体现工程成熟度的点：

- 对 marketplace 名称有保留名和官方源约束。
- 对某些名称格式做反冒充限制。
- 对路径和来源做 schema 级验证，而不是等运行时报错。

这背后的思想是：  
插件系统是供应链入口，校验要尽量前移。不要把“格式不对”“来源不合法”“字段不完整”留到运行时深处才炸。

复现版建议：

- manifest 解析阶段就做完整 Zod 校验
- 报错时保留插件名、来源、字段路径
- 不要允许 silent fallback

---

## 17.4 Plugin 目录结构设计

`pluginLoader.ts` 的注释里已经给出非常清晰的目录模型：

```text
my-plugin/
├── plugin.json
├── commands/
│   ├── build.md
│   └── deploy.md
├── agents/
│   └── test-runner.md
└── hooks/
    └── hooks.json
```

结合 schema 与其它 loader，可以进一步推断出推荐结构：

```text
my-plugin/
├── plugin.json
├── commands/
├── agents/
├── hooks/
├── mcp/
└── skills/
```

复现时不必一次性支持所有目录，但建议保留这种“按能力分目录”的设计。原因很直接：

- 命令和 agent 的 frontmatter、渲染方式、注册目标不同。
- hooks 与 MCP 配置文件通常是 JSON/YAML，而不是 markdown。
- skills 更像 prompt 资源，不应该跟 command 文件混放。

目录按能力分开，loader 才能保持清晰，也更容易做按类型的启停和诊断。

---

## 17.5 Plugin 装载流程

### 17.5.1 总体流程

插件装载最好拆成 6 个阶段：

1. 收集插件声明
2. 解析来源与版本
3. 定位本地目录或缓存目录
4. 读取 manifest 并校验
5. 读取各类贡献物并转成内部抽象
6. 汇总成功结果与错误结果

从源码看，Claude Code 的 loader 已经接近这个形态。

```mermaid
flowchart LR
  discover[收集插件声明]
  resolve[解析来源与版本]
  locate[定位目录或缓存]
  manifest[读取并校验 Manifest]
  load[读取贡献物]
  project[转成内部抽象]
  register[进入各自 Registry]
  report[汇总成功与错误]

  discover --> resolve --> locate --> manifest --> load --> project --> register --> report
```

### 17.5.2 插件来源

注释里明确提到两个主要来源：

1. marketplace-based plugins  
通过 `plugin@marketplace` 这类标识出现在 settings 中

2. session-only plugins  
来自 `--plugin-dir` CLI flag 或 SDK 传入的 plugins 选项

这说明插件来源不是单一的“安装到本地目录”。至少有两类：

- 持久安装插件
- 会话级临时插件

对于复现版，这是很值得照抄的设计。因为它能同时满足：

- 用户长期安装扩展
- 测试时临时挂载本地插件
- SDK 调用时注入定制扩展

### 17.5.3 插件装载与运行时注册要分开

这是复现时特别容易犯错的一点。

正确做法是：

- `plugin loader` 只负责发现、读取、校验、转换
- `commands/agents/hooks/mcp` 各自的 registry 再负责真正接入 runtime

不要让 plugin loader 直接修改全局状态。否则后面很难测试，也很难支持按模式装载、按策略过滤、按来源调试。

建议抽象成：

- `discoverPlugins()`
- `loadPluginManifest()`
- `loadPluginComponents()`
- `materializePluginContributions()`
- `registerPluginContributions()`

---

## 17.6 缓存、版本与安装路径策略

这一节是插件系统里最容易被忽略，但最值得学的一部分。

### 17.6.1 版本化缓存路径

`pluginLoader.ts` 明确给出了缓存路径格式：

```text
~/.claude/plugins/cache/{marketplace}/{plugin}/{version}/
```

这背后有 4 个好处：

- marketplace 维度隔离来源
- plugin 维度隔离包名
- version 维度允许并存多个版本
- 可以安全复用历史缓存而不覆盖当前活跃版本

如果你复现时只用：

```text
~/.my-agent/plugins/{name}/
```

后面升级、回滚、调试和兼容都很痛苦。

### 17.6.2 legacy 路径兼容

源码里还保留了 `getLegacyCachePath()` 和 `resolvePluginPath()` 的兼容逻辑：

- 先尝试 versioned path
- 如果没有，再尝试 legacy path
- 新装插件默认仍返回 versioned path

这说明插件缓存不是一次性设计完美，而是经过迭代演进的。复现版如果打算长期维护，也最好从一开始就留好“路径版本迁移”这层抽象，不要把路径写死在几十处地方。

### 17.6.3 seed cache 与未知版本探测

`probeSeedCache()` 与 `probeSeedCacheAnyVersion()` 透露出另一个成熟设计：

- 允许从 seed dirs 预置插件缓存
- 当版本暂时无法确定时，仍可从 seed 中探测真实版本目录

这个机制适合：

- 企业内部分发预热插件
- CI 镜像预灌缓存
- 离线/弱网环境加速插件可用性

复现版前期可以不做 seed cache，但路径设计要兼容这种扩展。

### 17.6.4 zip cache

源码里还有 `zipCache` 逻辑，说明插件缓存不一定总是目录，也可能以压缩形式存在：

- 减少磁盘占用
- 便于分发或镜像预载
- 提高冷启动安装效率

如果你先做最小复现，可以只支持目录缓存；但中后期建议把“缓存存储格式”独立成接口，不要让 loader 默认所有缓存都是普通目录。

---

## 17.7 Plugin 可贡献哪些能力

从 loader、schema、built-in plugin 支持与命令/agent loader 的耦合关系看，plugin 至少能贡献以下几类能力：

### 17.7.1 Command

插件命令通常以 markdown/frontmatter 形式存在，然后被转换为 `Command` 对象进入统一命令池。

这意味着：

- 命令的最终行为仍受 `Command` 系统约束
- 插件命令不应该绕过 slash command 解析器
- 插件命令也要遵守 remote-safe、availability、model invocation 等规则

### 17.7.2 Agent

`loadPluginAgents()` 会把插件里的 agent 定义装成 `AgentDefinition`。这说明插件可以提供领域专用 agent，比如：

- reviewer
- release manager
- docs agent
- security auditor

这点非常关键。因为它说明 plugin 不只是 UI 级扩展，而是能进入多 agent runtime 的深层扩展面。

### 17.7.3 Hooks

schema 直接引入 `HooksSchema`，说明插件可以声明 hooks。这样插件不只是“给系统加能力”，还能“接入系统生命周期”。

例如可以在：

- agent 启动时
- command 执行前后
- 会话某阶段

注册附加逻辑。

### 17.7.4 MCP Servers

schema 也直接引入了 `McpServerConfigSchema`。这意味着插件可以不只提供 prompt 层能力，而是直接把外部协议能力打包交付进来。

这点非常值得复现：  
插件 + MCP 的组合，才是真正让系统形成“能力分发市场”的基础。

### 17.7.5 Skills

`builtinPlugins.ts` 里明确说明 built-in plugins 可以提供 skills，并且会通过 `getBuiltinPluginSkillCommands()` 投影为 prompt command。

这说明 `skill` 在系统里并不是非要独立于插件存在。更好的设计是：

- skill 可以独立分发
- 也可以作为 plugin 的一个组成部分被分发

这样企业团队就可以用一个插件包同时交付：

- 组织命令
- 组织 agent
- 组织 hooks
- 组织 MCP
- 组织 skills

---

## 17.8 Built-in Plugin 与普通 Plugin 的区别

`builtinPlugins.ts` 给出一个很重要的区分：

- built-in plugin 不是普通本地插件目录
- 但它们也被纳入 plugin 概念
- 它们会出现在 `/plugin` 界面里，并拥有启用/禁用状态

这里最值得借鉴的思想是：

> 把“内建扩展”和“第三方扩展”统一到同一个管理面板下，但保留不同的信任等级和来源语义。

这种设计比“内建功能硬编码，第三方功能另起一套管理逻辑”更稳定。

复现版建议：

- 核心 runtime 允许注册 built-in plugins
- built-in plugin 使用固定 source，例如 `builtin`
- 显示层统一管理启用/禁用状态
- 策略层对 built-in 和 third-party 给予不同信任等级

---

## 17.9 启用、禁用与优先级

插件系统如果只有“能不能加载”，那还不够。成熟系统一定要能回答：

- 插件装了没
- 插件启用了没
- 插件加载失败了没
- 如果贡献同名组件，谁覆盖谁

从源码里的 enable/disable state management、duplicate detection、active/all 列表模式可以推断出，Claude Code 是把“已发现”“已启用”“已失败”分开管理的。

复现版建议也这样做，至少维护三类状态：

- `discovered`
- `enabled`
- `failed`

并且为每个 plugin 记录：

- source
- resolvedPath
- version
- error list
- component list

这样 `/plugin` 或诊断页才有机会做得可用。

---

## 17.10 Plugin 安全模型

插件是供应链入口，所以安全不是附属话题。

### 17.10.1 来源约束

从 `schemas.ts`、`marketplaceHelpers.ts` 可以看出，系统对来源有明确概念：

- 官方 marketplace
- 已知 marketplace
- 被 blocklist 拦截的 marketplace
- policy 允许或禁止的来源

这意味着插件系统并不默认“任何地方都能装”。它把来源视为安全边界的一部分。

### 17.10.2 路径约束

loader 使用了路径校验工具，例如 `validatePathWithinBase()`。这类校验对插件系统至关重要，因为插件安装和缓存过程天然涉及：

- 解压
- clone
- copy
- symlink
- readlink

任何一个地方处理不好都可能引入路径逃逸问题。

### 17.10.3 admin-trusted 与 user-controlled 区分

在 agent 运行逻辑里可以看到一个重要判断：  
某些 frontmatter MCP 配置只有在“管理员信任来源”时才被允许自动加载，而普通用户控制的自定义面则会受 `plugin-only` 策略限制。

这说明系统不是粗暴地说“插件一定安全”，而是区分：

- admin-trusted surface
- user-controlled surface

这在企业版复现里非常重要。你很可能需要：

- 允许组织发布插件
- 限制普通用户只能启用白名单插件
- 禁止用户通过 markdown/frontmatter 自行注入某些高风险能力

### 17.10.4 插件错误必须结构化收集

插件加载过程会遇到很多失败模式：

- manifest 非法
- 目录不存在
- 来源不可达
- 依赖解析失败
- hooks 配置错误
- agent frontmatter 非法

这些错误不能只是 stderr 打一条日志。应该变成结构化的 `PluginError[]`，这样 UI、CLI、诊断系统才能解释“为什么插件不可用”。

---

## 17.11 复现时应该怎么做

### 17.11.1 MVP 阶段

先只支持本地目录插件，但必须保留正式抽象：

- `plugin.json`
- `commands/`
- `agents/`
- `hooks/`
- `PluginManifest`
- `LoadedPlugin`
- `PluginLoadResult`

### 17.11.2 增强版阶段

加入：

- 启用/禁用状态
- 插件错误列表
- 版本化缓存目录
- session-only 插件挂载
- built-in plugin 注册

### 17.11.3 高级版阶段

再加入：

- marketplace
- seed cache
- zip cache
- source allow/block policy
- 企业级 plugin-only customization policy

---

## 17.12 本章对复现工程的直接指导

如果你要真正把这层做出来，我建议先定义这 6 个接口：

```ts
type PluginManifest = {
  name: string
  version: string
  description?: string
  author?: string
  homepage?: string
  repository?: string
  license?: string
  keywords?: string[]
}

type PluginSource =
  | { type: 'builtin'; id: string }
  | { type: 'installed'; id: string; marketplace?: string }
  | { type: 'session'; path: string }

type LoadedPlugin = {
  id: string
  source: PluginSource
  manifest: PluginManifest
  rootDir: string
  commands: LoadedPluginCommand[]
  agents: LoadedPluginAgent[]
  hooks?: HooksSettings
  mcpServers?: ScopedMcpServerConfig[]
}

type PluginError = {
  pluginId: string
  stage: 'discover' | 'manifest' | 'components' | 'register'
  message: string
}

type PluginLoadResult = {
  discovered: LoadedPlugin[]
  active: LoadedPlugin[]
  failed: PluginError[]
}
```

然后按下面顺序落地：

1. 先做 manifest 与目录扫描
2. 再做 command/agent 贡献加载
3. 再做启用/禁用与错误收集
4. 再做缓存与安装
5. 最后做 marketplace 与策略系统

如果你反过来，一开始就做 marketplace，而内部抽象还没定稳，后面会很难收拾。

---

## 17.13 你最应该借用的核心思想

这一章最值得你复用的，不是某个缓存路径，而是这 4 个思想：

1. `plugin` 是发行单元，不是目录扫描技巧。
2. 插件贡献物最终都要投影回统一 runtime 抽象。
3. 缓存、来源、启用状态、错误状态和策略限制必须是系统级能力。
4. 内建扩展与第三方扩展应统一管理，但保留不同信任等级。

把这 4 点做对，你的复现工程才会从“能加载扩展”进化成“能长期维护的扩展平台”。

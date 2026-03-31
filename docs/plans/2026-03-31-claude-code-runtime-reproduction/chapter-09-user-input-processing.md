# 第 9 章 用户输入处理设计

> 状态: 已完成初稿
> 章节目标: 规范“输入是如何进入系统”的。

[返回总览](/Users/magongli/Downloads/project/claude-code-sourcemap/docs/plans/2026-03-31-claude-code-runtime-reproduction/README.md)

---

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

## 9.1 输入处理在总架构中的位置

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

## 9.2 输入类型矩阵

建议先把系统支持的输入类型分清楚。

### 9.2.1 普通文本输入

这是最常见路径，通常表现为：

- REPL 输入框中的自然语言。
- Headless/SDK 传入的 prompt 字符串。

它最终通常会转成一条 `user` 消息，并进入 query。

### 9.2.2 结构化内容块输入

这类输入在 SDK、IDE、远程端更常见，例如：

- 文本 + 图片的组合块。
- 前置 context blocks + 最后一个用户问题。

这类输入不应在一开始就粗暴降级成字符串，而应保留为 content block 数组，直到进入 API 适配层。

### 9.2.3 slash command 输入

slash command 的关键点是：它看起来像文本，但语义上不是普通 prompt。它可能：

- 完全本地执行。
- 先修改会话状态，再进入 query。
- 生成新的 prompt 并继续提交。
- 在 REPL 模式下渲染本地 JSX 交互。

所以 slash command 必须在输入层被优先识别，而不是交给模型猜测。

### 9.2.4 附件与 pasted content

这类输入通常不是用户显式敲出来的内容，但对任务非常重要，例如：

- 粘贴的长文本片段。
- 图片粘贴。
- IDE 当前选区。
- hooks 追加的额外上下文。
- agent mention、memory、技能发现等系统性附件。

这些内容更适合进入 attachment/message 流，而不是拼成一坨字符串埋进 prompt。

### 9.2.5 系统生成输入

系统内部也可能注入输入，例如：

- queued command 触发的后续 prompt。
- scheduler/background task 的隐式提示。
- compact、resume、rewind 过程中的系统引导。

这类输入通常应该被标记为 `isMeta`，做到“模型可见、用户可不见”。

## 9.3 输入标准化流水线

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

## 9.4 为什么 `processUserInput` 不是薄函数

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

## 9.5 图像与多模态输入处理

Claude Code 风格实现对图片输入的处理非常值得借鉴，因为它不是把图片简单透传，而是先做标准化：

- 调整尺寸和降采样，确保符合 API 限制。
- 生成图片元信息文本，例如尺寸、来源路径。
- 将 pasted image 存盘，以便后续工具链也能引用本地路径。
- 在最终消息中同时保留 image block 与必要元数据。

这背后的设计思想是：

- 多模态输入不应该只服务模型，也要服务后续工具链。
- 图像不是“附件特判”，而是正式 content block 类型。
- 一些模型不可见但对调试与工具有用的信息，可以通过 `isMeta` user message 附加。

## 9.6 attachment 提取与角色设计

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

## 9.7 slash command 解析与输入分流

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

## 9.8 hooks 在输入处理中的位置

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

## 9.9 文本 prompt 如何落成正式消息

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

## 9.10 输入到 query 的边界

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

## 9.11 输入处理阶段的异常路径

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

## 9.12 设计总结

第 9 章真正要强调的，是输入处理层的地位：

- 它不是 UI 辅助函数，而是 query 的前置编排器。
- 它既负责标准化，也负责局部分流和安全阻断。
- 它输出的不是字符串，而是“是否进入 query + 哪些消息 + 用什么约束进入”。

## 9.13 本章对复现工程的直接指导

建议你把输入处理直接实现成一条固定管线，而不是散落在 REPL 组件和命令处理器里：

```ts
raw input
  -> normalize text/attachments
  -> parse slash command
  -> run input hooks
  -> build user/meta messages
  -> decide enter-query or local-only
```

### 9.13.1 建议尽早抽出的接口

```ts
type InputProcessingResult = {
  enterQuery: boolean
  messages: Message[]
  commandResult?: LocalCommandResult
  modeOverrides?: Partial<QueryOptions>
}
```

### 9.13.2 第一版就要明确的分流

- 普通 prompt: 进入 query
- 本地 command: 不进 query
- prompt command: 先生成 prompt，再进 query
- 被 hook 阻断: 直接返回结构化错误

### 9.13.3 这一章最值钱的落地点

只要输入层先标准化好了，后面的 query engine 就可以始终假设：

- 自己拿到的是结构化消息
- slash command 已处理完
- 附件已正规化
- 当前轮是否进入 query 已决策完成

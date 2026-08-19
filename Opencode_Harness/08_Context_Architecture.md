# Context Architecture：模型每一轮究竟看见什么

上一篇：[07 Agent Loop](./07_Agent_Loop.md) ｜ 下一篇：[09 Tools 与 Permission](./09_Tools_and_Permission.md)

> 固定源码：OpenCode `0e3474509aa5ad16afcf9c439785514d6443c6af`（`dev`，2026-08-18）
>
> 本篇职责：解释当前默认 TUI 路径中，System、Messages 与 Tool definitions 怎样组成一次 Provider Request；项目规则从哪里进入；历史为何会被转换、裁剪和重排。Session 如何持久化、Compaction 如何触发留给[第 10 篇](./10_Session_and_Persistence.md)。

仍然从零基础学习者的请求开始：

> 请先查看这个项目的 Harness 教程入口和项目规则，再告诉我应该按照什么顺序学习。

这句话对人类很自然，对模型却留下了许多空白：项目位于哪个目录？“教程入口”是哪一个文件？有哪些项目规则？可以使用哪些读取能力？前一轮是否已经读过 README？这些信息不会因为存在于硬盘、配置或数据库中，就自动出现在语言模型眼前。

模型每次推理都发生在一次有限的 Provider Request 中。Harness 必须在请求发出前选择信息、改变信息的表达形式，再把它们放进模型能够接收的字段。这个过程就是上下文工程（Context Engineering）的核心工作。

本篇只回答一个问题：**一条信息怎样进入本轮模型视野，又怎样随着 Agent Loop 推进而改变身份或可见程度？**

## 一、Context Engineering 不是把所有信息塞进 Prompt

### 1.1 Prompt 是一段输入，Context 是一次判断的完整信息环境

提示词工程（Prompt Engineering）通常关注某段指令应该怎样写。上下文工程的范围更宽：它关心模型做出当前判断时，哪些信息应该进入、以什么身份进入、以什么顺序组织、哪些内容应该暂时隐藏，以及旧信息怎样在窗口有限时继续发挥作用。

```text
Prompt Engineering
    重点：一句或一组指令怎样表达得更清楚

Context Engineering
    重点：本次模型调用的完整信息环境怎样构造
          = System + Messages + Tool definitions + Provider-specific transform
```

因此，“Context 就是用户刚输入的 Prompt”是不完整的。用户问题只是其中一部分。模型还需要知道当前工作目录、适用规则、此前已经发生的 Tool Call / Tool Result，以及这一轮有哪些能力可以调用。

### 1.2 模型没有一个持续打开的项目视野

人类在编辑器里工作时，可以看到目录树、当前文件、终端和前几分钟的操作。语言模型并不天然拥有这种持续视野。对普通 Provider 调用来说，它只看见 Harness 送出的请求字段。

这带来三个基础边界：

- 文件存在于工作区，不等于模型已经看见文件内容；
- Message 和 Part 存在于 Session 存储，不等于它们会原样进入本轮请求；
- Tool executor 存在于 OpenCode 进程，不等于这一轮 Tool definition 对模型可见。

所以模型说“我读取了 README”不能只依据一句自然语言判断。系统必须能够在 Session 中找到对应 Tool Call 和已结算 Tool Result，并在后续请求中把它们转换成模型可识别的 Messages。

### 1.3 Provider Request 有三条并列输入通道

在本系列的逻辑模型中，一次普通 Provider Request 的上下文可以分为三条并列通道：

```text
Provider Request
│
├── System：这一轮应遵守什么
│   ├── Provider-family base instructions 或 Agent prompt
│   ├── Environment
│   ├── Project / MCP / Skill instructions
│   └── 本次输入附带的 system 内容
│
├── Messages：到目前为止发生了什么
│   ├── User / Assistant 会话内容
│   ├── Tool Call / Tool Result
│   └── Compaction 后仍然活跃的历史
│
└── Tool definitions：这一轮可以提议什么
    ├── 工具名称与用途
    ├── 参数 schema
    └── 经过 Agent、Session 和本轮开关过滤的工具集合
```

三条通道相互配合，却不能互相替代。“只读取，不修改项目”适合进入 System；`read` 返回的 README 正文是已经发生的观察，适合进入 Messages；`read` 需要哪些参数，属于 Tool definition。如果把它们都称为“Prompt”，就会失去规则、历史和能力之间最重要的身份差异。

### 1.4 保存、活跃与可见是三种不同状态

理解 Context Architecture 最重要的一条原则是：

> **Stored does not mean visible；visible does not mean durable。保存了不等于本轮可见，本轮可见也不等于作为 Session 历史保存。**

例如，一条旧 Tool Result 仍可保存在 Session 中，但被 Pruning 标记后，本轮模型只看见 `[Old tool result content cleared]`。反过来，工作目录和当天日期可以由 `SystemPrompt.environment()` 在每轮临时生成并进入 System，却不需要成为一条普通聊天 Message。

```text
Durable Session History
    系统保存的 Message / Part 领域事实
            │
            ▼ 选择、重排、转换
Active History
    本轮 Loop 认为仍与当前任务相关的历史
            │
            ▼ Provider 兼容转换
Provider-visible Context
    System、Messages、Tool definitions 中模型实际收到的表示
```

本篇主讲后两层怎样形成；第一层的存储、事件和恢复边界由[第 10 篇](./10_Session_and_Persistence.md)展开。

## 二、每次外层迭代都会重新组装 Context

### 2.1 Agent Loop 先重新取得活跃历史

[第 07 篇](./07_Agent_Loop.md)已经说明，当前默认路径会在 `SessionPrompt.run` 的 `while (true)` 中反复产生 Provider Turn。每次普通外层迭代开始时，Loop 都重新读取活跃历史：

```ts
let msgs = yield* MessageV2.filterCompactedEffect(sessionID).pipe(
  Effect.provideService(Database.Service, database),
)

const { user: lastUser, assistant: lastAssistant, finished: lastFinished, tasks } =
  MessageV2.latest(msgs)
```

这里不是简单地“取数据库最后一条”。`filterCompactedEffect()` 先选择 Compaction 后应继续送给模型的历史；`latest()` 再从这组可能被重排的消息里找最新 User、Assistant、已完成 Assistant 和待处理特殊任务。因此 Context 的起点是当前活跃历史，而不是全部持久化记录原样拼接。

### 2.2 Tools 先被物化为本轮候选能力

Loop 确认这一轮进入普通 Provider 路径后，会根据当前 Agent、Session、Model、Permission 和历史物化工具：

```ts
const tools = yield* SessionTools.resolve({
  agent,
  session,
  model,
  processor: handle,
  bypassAgentCheck,
  messages: msgs,
  promptOps,
})
```

“物化”意味着把 OpenCode 内部注册的 Tool 转换成当前模型适配器可接收的 AI SDK Tool：包含描述、转换后的 JSON Schema 和执行包装器。这里得到的是候选集合，还会在靠近 Provider 的边界再次按 Permission 与本轮开关过滤。

Tool definitions 与 Messages 同时依赖当前状态，但形成方式不同：Messages 由历史转换而来；Tool definitions 由注册能力按本轮策略重新生成。上一轮可以看到的 Tool，不保证下一轮仍可见。

### 2.3 五组材料并行取得，再组成 System 与 Messages

`SessionPrompt.run` 随后并行准备 Skill guidance、Environment、Project Instructions、MCP Instructions 和 Provider Messages：

```ts
const [skills, env, instructions, mcpInstructions, modelMsgs] = yield* Effect.all([
  sys.skills(agent),
  sys.environment(model),
  instruction.system().pipe(Effect.orDie),
  sys.mcp(agent, session.permission),
  MessageV2.toModelMessagesEffect(msgs, model),
])

const system = [
  ...env,
  ...instructions,
  ...(mcpInstructions ? [mcpInstructions] : []),
  ...(skills ? [skills] : []),
]
```

Environment 与 Instructions 不是从历史 Message 中恢复，而是在新的外层迭代中重新计算。工作目录、日期、配置的规则文件或 MCP Server 状况变化后，下一次外层迭代可能得到不同内容。

历史也不是直接传递。`MessageV2.toModelMessagesEffect(msgs, model)` 会根据目标 Provider / Model 的能力，把 OpenCode 的 Message / Part 领域结构转换成 Provider 能接受的消息表示。

### 2.4 靠近 Provider 的边界还会做最终组装

`SessionPrompt` 准备的三组材料还不是最终线格式。`LLM.run` 会把它们交给 `LLMRequestPrep.prepare()`：

#### 2.4.1 System 的基础层、运行时材料和本次输入在这里合并

```ts
const system = [
  [
    ...(input.agent.prompt ? [input.agent.prompt] : SystemPrompt.provider(input.model)),
    ...input.system,
    ...(input.user.system ? [input.user.system] : []),
  ]
    .filter((x) => x)
    .join("\n"),
]

yield* input.plugin.trigger(
  "experimental.chat.system.transform",
  { sessionID: input.sessionID, model: input.model },
  { system },
)
```

默认基础层是 `SystemPrompt.provider(model)`；如果当前 Agent 明确提供 `agent.prompt`，它会替代 Provider-family base instructions，而不是在后面再叠加一份。之后才追加环境与规则、本次 User Message 附带的 `user.system`，最后允许 Plugin transform 修改 System。

#### 2.4.2 Provider 适配还会改变最终线格式

不同 Provider 还可能要求不同线格式。固定源码中，OpenAI OAuth 把 System 合并到 `options.instructions`；普通非 workflow 请求则通常把它转成 `role: "system"` 的 Model Message。逻辑上仍是 System 通道，实际发送方式由 Provider 适配层决定。

### 2.5 “下一轮重组”与“同一轮 Retry”不是一回事

只有回到 `SessionPrompt.run` 的外层迭代，才会重新执行上述历史选择、Instructions 加载、Messages 转换和 Tool 物化。Provider Retry 位于同一个 `SessionProcessor.process()` 内，复用已准备好的 `streamInput` 重试，不会先回到外层重新读取一遍规则或历史。

```text
Tool Result 结算后继续
    -> 回到外层 Loop，新的 Provider Turn 重新组装 Context

Provider 暂时失败后 Retry
    -> 同一个 Provider Turn 的请求输入再次发送
```

Retry 的完整控制位置见[第 07 篇](./07_Agent_Loop.md)。对 Context 而言，只需记住：**外层 continuation 可能改变可见信息，同一轮 Retry 通常不改变。**

## 三、System：这一轮应当遵守什么

### 3.1 Provider-family base 与 Agent prompt 是二选一的基础层

`SystemPrompt.provider(model)` 会根据模型家族选择 GPT、Codex、Claude、Gemini、Kimi 或默认 prompt。它解决模型适配问题：不同模型家族对工具使用、代码任务和指令格式的偏好可能不同。

若 Agent 配置自己的 `prompt`，`LLMRequestPrep.prepare()` 使用的是：

```ts
...(input.agent.prompt
  ? [input.agent.prompt]
  : SystemPrompt.provider(input.model))
```

这说明 Agent prompt 与 Provider-family base 在这里是互斥选择。Agent 还可以携带 Model 偏好、Permission、步骤参数等配置，但那些字段不会全部以文字进入 System；它们分别影响请求参数、工具可见性或运行控制。完整层级由[第 11 篇](./11_Agent_Specialization_and_Collaboration.md)解释。

### 3.2 Environment 告诉模型“当前在哪里”，不代表已经扫描环境

`SystemPrompt.environment()` 动态产生工作环境描述：

```ts
return [
  [
    `You are powered by the model named ${model.api.id}. The exact model ID is ${model.providerID}/${model.api.id}`,
    `Here is some useful information about the environment you are running in:`,
    `<env>`,
    `  Working directory: ${ctx.directory}`,
    `  Workspace root folder: ${ctx.worktree}`,
    `  Is directory a git repo: ${ctx.project.vcs === "git" ? "yes" : "no"}`,
    `  Platform: ${process.platform}`,
    `  Today's date: ${new Date().toDateString()}`,
    `</env>`,
  ].join("\n"),
  // 可选的 project references
].filter((part): part is string => part !== undefined)
```

这一层让模型知道相对路径应从哪里理解、当前是不是 Git 项目、运行平台和日期是什么。若项目配置带描述的 references，它还会按名称排序并列出相关目录。

但 Environment 只描述元信息。看到 `Working directory: /project` 不等于看到了 `/project` 的目录树；看到 `Is directory a git repo: yes` 也不等于知道当前 diff。具体内容仍需要相应 Tool 取得。

### 3.3 Project Instructions 有 ambient 与按读取位置发现两条路径

项目规则经常被笼统理解成“读取 AGENTS.md”。固定源码实际上有两条不同路径：一条在每次普通外层迭代中作为 ambient System 加载；另一条在读取具体文件时，发现更靠近目标文件的规则并附在 Tool Result 中。

#### 3.3.1 Ambient 规则文件怎样选择

`Instruction.systemPaths()` 会寻找全局和项目级规则：

```ts
const globalFiles = [
  path.join(global.config, "AGENTS.md"),
  ...(!flags.disableClaudeCodePrompt
    ? [path.join(global.home, ".claude", "CLAUDE.md")]
    : []),
]

const instructionFiles = [
  "AGENTS.md",
  ...(!flags.disableClaudeCodePrompt ? ["CLAUDE.md"] : []),
  "CONTEXT.md", // deprecated
]
```

全局候选中找到第一个存在的文件后停止。项目级文件按照 `AGENTS.md -> CLAUDE.md -> CONTEXT.md` 的候选顺序查找；**第一个具有匹配结果的文件类型获胜**，然后把该类型沿当前目录到 worktree 范围内找到的路径加入集合。这不是把每个祖先目录里的三种文件全部叠加。

`CONTEXT.md` 在源码中明确标为 deprecated。它仍是兼容候选，不应被描述成推荐的新规则入口。

#### 3.3.2 配置可以增加本地规则和远程规则

`config.instructions` 还能指定本地 glob、绝对路径、`~/` 路径或 HTTP(S) URL。本地配置匹配的文件进入 `systemPaths()`；远程 URL 由 `Instruction.system()` 获取：

```ts
const res = yield* http.execute(HttpClientRequest.get(url)).pipe(
  Effect.timeout(5000),
  Effect.catch(() => Effect.succeed(null)),
)
```

远程读取有 5 秒超时，并把失败降为空内容。外部规则源不可用时不会让普通 Context 构建无限等待；同时也意味着“配置了远程 instructions”不保证每次都成功注入。成功内容被包装成 `Instructions from: <source>` 后进入 System。

#### 3.3.3 `read` 还会发现目标文件附近的规则

`Instruction.resolve()` 从目标文件所在目录向工作目录根部逐级向上，寻找更近的规则文件：

```ts
const target = path.resolve(filepath)
let current = path.dirname(target)

while (current.startsWith(root) && current !== root) {
  const found = yield* find(current)
  if (!found || found === target || sys.has(found) || already.has(found)) {
    current = path.dirname(current)
    continue
  }
  results.push({
    filepath: found,
    content: `Instructions from: ${found}\n${content}`,
  })
  current = path.dirname(current)
}
```

它跳过目标文件本身、已经作为 ambient System 加载的路径、历史中仍可确认已随 `read` 加载的路径，以及当前 Assistant Message 已经 claim 的路径。`read` Tool 再把新规则附到输出：

```ts
output += `\n\n<system-reminder>\n${loaded
  .map((item) => item.content)
  .join("\n\n")}\n</system-reminder>`
```

于是同为“项目规则”，在请求中可能有两种身份：

```text
Ambient Project Instructions -> System

读取目标附近新发现的 Instructions -> read Tool Result -> Messages
```

规则的语义作用可能相近，传输身份却不同。不能只搜索 System 数组，就断言模型收到的全部项目规则。

### 3.4 MCP Instructions 受相关工具可见性约束

MCP Server 可以同时提供能力和使用说明。`SystemPrompt.mcp()` 只保留至少有一个相关 Tool 未被 Permission 禁用的 Server；如果 Server 没有关联 Tool，也可以保留说明：

```ts
const instructions = (yield* mcp.instructions()).filter(
  (item) =>
    item.tools.length === 0 ||
    Permission.disabled(item.tools, ruleset).size < item.tools.length,
)
```

这避免模型收到一大段关于完全不可用能力的说明。MCP instructions 进入 System；MCP Tool 或资源读取能力仍通过 Tool definitions 暴露。一个解释“怎样使用”，一个声明“可以调用什么”。

### 3.5 Skill guidance 与 Skill 全文采用渐进披露

如果 Agent 的 Permission 没有禁用 `skill`，`SystemPrompt.skills()` 会列出可用 Skill 的名称、描述，并告诉模型在匹配任务时使用 `skill` Tool：

```ts
if (Permission.disabled(["skill"], agent.permission).has("skill")) return

const list = yield* skill.available(agent)
return [
  "Skills provide specialized instructions and workflows for specific tasks.",
  "Use the skill tool to load a skill when a task matches its description.",
  Skill.fmt(list, { verbose: true }),
].join("\n")
```

这一轮 System 中通常只是发现信息，不是把所有 Skill 正文展开。模型调用 `skill` 后，真正加载的内容以 Tool Result 进入 Messages，后续 Provider Turn 才能依据全文行动。

这就是渐进披露（Progressive Disclosure）：先用较小的名称、描述和使用条件帮助模型判断“是否相关”，只有确定相关后才把较大的正文加入历史。它既减少无关内容对当前判断的干扰，也避免每轮为所有 Skill 支付完整上下文成本。代价是 Skill 全文不会在第一次判断时自动可见，模型必须先正确选择并成功调用 `skill`。

### 3.6 本次 System、结构化输出和 Plugin 处于不同加工位置

#### 3.6.1 本次 System 与结构化输出要求属于请求级增加项

`input.user.system` 是随最新 User Message 保存的本次 System 内容，它在 `LLMRequestPrep.prepare()` 中追加到基础层之后。若本次要求 JSON Schema 结构化输出，`SessionPrompt` 还会把相应 policy 加入 System，并添加 `StructuredOutput` Tool。

两者都只影响当前 User Message 引出的运行，但承担不同职责：`user.system` 增加本次行为约束，结构化输出 policy 则与专用 Tool 一起规定最终返回形态。它们不应被误写成 ambient 项目规则，也不会改写 Agent Registry 中的永久配置。

#### 3.6.2 Plugin transform 位于完整 System 形成之后

最后，`experimental.chat.system.transform` 可以修改完整 `system` 数组。因此源码中“先拼接的顺序”描述的是 transform 之前的默认组织方式；Plugin 有能力改变最终形态。最终 System 不是仓库中某一个 `.txt` 文件的原样内容，而是多来源在运行时共同组成的结果。

## 四、Messages：到目前为止发生了什么

### 4.1 Session Message / Part 要先转换成 Provider Messages

OpenCode 内部使用 Message 与 Part 表达 User、Assistant、文本、reasoning、文件、Tool 状态、Compaction marker 等领域事实。Provider SDK 接受的是另一套消息表示。`MessageV2.toModelMessagesEffect()` 是两者之间的重要投影边界。

```text
Session Message / Part
    保留 OpenCode 运行所需的丰富状态
            │
            ▼ toModelMessagesEffect(model)
Provider Messages
    只保留目标 Provider 能接收、且本轮应看见的表示
```

“投影”不等于简单序列化。转换函数会删除、改写、补齐或重新安置信息，以满足当前模型和 Provider 的格式要求。

### 4.2 User 内容会过滤空文本并处理文件表示

```ts
if (part.type === "text" && !part.ignored && part.text !== "") {
  userMessage.parts.push({ type: "text", text: part.text })
}

if (
  part.type === "file" &&
  part.mime !== "text/plain" &&
  part.mime !== "application/x-directory"
) {
  userMessage.parts.push({
    type: "file",
    url: part.url,
    mediaType: part.mime,
    filename: part.filename,
  })
}
```

被 `ignored` 标记或为空的 User text 不会进入模型消息。`text/plain` 和目录类型 File Part 也不在这里再次作为文件发送，因为创建 User Message 时已经为它们生成相应文本内容；重复发送会让模型看到同一信息两次。

非文本附件则按 Provider 能力保留为 File Part，或在要求移除媒体时转成占位文本。这说明“Session 中有一个 File Part”不能直接推导“Provider 收到同样的文件字段”。

### 4.3 Tool Part 必须成为配对、可解释的模型历史

#### 4.3.1 未结算调用要被转换成可闭合的 Provider 表示

完成的 Tool Part 会转成 `output-available`，包含调用 ID、输入和输出；错误状态会变成 `output-error`。如果 Tool 仍是 pending 或 running，却因为中断等原因进入历史，转换层会补成 interrupted error：

```ts
if (part.state.status === "pending" || part.state.status === "running") {
  assistantMessage.parts.push({
    type: ("tool-" + part.tool) as `tool-${string}`,
    state: "output-error",
    toolCallId: part.callID,
    input: part.state.input,
    errorText: "[Tool execution was interrupted]",
  })
}
```

这不是把真实执行结果伪造为失败，而是避免向要求严格配对的 Provider 重放悬空 `tool_use`。OpenCode 的领域状态允许描述“曾经开始、后来中断”，Provider Messages 则需要一份结构闭合的 Tool Call / Tool Result 对。

#### 4.3.2 已完成调用还要根据可见性状态选择输出

对于已完成但被 Pruning 标记的 Tool Part，模型不再收到原始大输出：

```ts
const outputText = part.state.time.compacted
  ? "[Old tool result content cleared]"
  : truncateToolOutput(part.state.output, options?.toolOutputMaxChars)
```

原始 Session 事实和 Provider-visible Messages 因而可以有意不同。

### 4.4 换模型时 reasoning 与 Provider metadata 需要降级

#### 4.4.1 Reasoning 和 metadata 受原 Provider 约束

Assistant 历史可能包含某个 Provider 特有的 reasoning signature 或 metadata。若当前请求换到另一个 Provider / Model，直接重放可能不兼容：

```ts
const differentModel =
  `${model.providerID}/${model.id}` !==
  `${msg.info.providerID}/${msg.info.modelID}`
```

同模型时可以保留相关 metadata；不同模型时，非空 reasoning 会降为普通 text，Provider-specific metadata 不再附带。这是把不可移植的表示转换成更通用的消息内容。

#### 4.4.2 Tool Result 中的媒体还要适配目标模型

Tool Result 中的图片或 PDF 也有类似适配：有些 Provider 不接受媒体直接位于 Tool Result，转换层会将其抽出，注入 synthetic User Message；若目标模型根本不支持该媒体类型，则走不支持内容的处理路径。Messages 因而也是跨 Provider 兼容层。

### 4.5 Compaction marker 会被翻译成模型能理解的请求

Compaction User Message 内部包含专门的 `compaction` Part。转换时它被表达为普通文本：

```ts
if (part.type === "compaction") {
  userMessage.parts.push({
    type: "text",
    text: "What did we do so far?",
  })
}
```

模型由此知道当前 Turn 的任务是总结此前工作。内部控制 Part 与 Provider Messages 的自然语言表示不同，再次体现领域状态和模型输入之间的投影关系。

## 五、Tool definitions：这一轮可以提议什么

### 5.1 Tool definition、Tool Call、Tool Result 和 executor 是四个阶段

```text
Tool definition
    Harness 告诉模型：有一个 read，它做什么、接收哪些参数
            │
            ▼
Tool Call
    模型提出：read({ filePath: "..." })
            │
            ▼
Tool executor
    OpenCode 验证、授权并实际读取文件
            │
            ▼
Tool Result
    读取结果结算进 Tool Part，后续转成 Messages
```

只有第一项属于本轮请求的 Tool definitions 通道。Tool Call / Tool Result 会成为 Messages 的一部分；executor 运行在 Harness 的工具运行边界，不会作为源码对象直接交给模型。

### 5.2 `SessionTools.resolve()` 把注册工具转换成当前模型的定义

#### 5.2.1 Tool 对象先经过 Provider Schema 适配

```ts
for (const item of yield* registry.tools({
  modelID: ModelV2.ID.make(input.model.api.id),
  providerID: input.model.providerID,
  agent: input.agent,
  permission: input.session.permission,
})) {
  const schema = ProviderTransform.schema(
    input.model,
    ToolJsonSchema.fromTool(item),
  )

  tools[item.id] = tool({
    description: item.description,
    inputSchema: jsonSchema(schema),
    execute(args, options) {
      // 建立 Tool Context，执行 hook 与真正 executor
    },
  })
}
```

`ProviderTransform.schema()` 针对当前 Provider 调整内部 Schema，使其成为合法 function-calling 定义。这里转换的是模型能看见的参数契约，不是执行 Tool，也不会把 executor 源码送进 Context。

#### 5.2.2 执行包装器把定义连接回 Harness 运行现场

同一个 `tool({...})` 对象还包含执行包装器。模型真正生成 Tool Call 后，包装器才把 Session ID、Message ID、Call ID、AbortSignal、Agent、历史和 Permission ask 连接到真实工具。

所以 Tool definitions 通道里存在两层同时重要但职责不同的信息：描述与 Schema 帮助模型选择和构造调用；闭包中的 executor context 则留在 Harness 内，负责将来执行。完整注册、授权和结算链由[第 09 篇](./09_Tools_and_Permission.md)展开。

### 5.3 靠近请求边界还会进行最终可见性过滤

`LLMRequestPrep.resolveTools()` 合并 Agent Permission 与 Session Permission，同时读取最新 User Message 的本轮 Tool 开关：

```ts
function resolveTools(input: Pick<PrepareInput, "tools" | "agent" | "permission" | "user">) {
  const disabled = Permission.disabled(
    Object.keys(input.tools),
    Permission.merge(input.agent.permission, input.permission ?? []),
  )

  return Record.filter(
    input.tools,
    (_, k) => input.user.tools?.[k] !== false && !disabled.has(k),
  )
}
```

最终集合按工具名排序后进入 Prepared Request。工具可见性至少受四类因素影响：Registry / MCP 当前提供什么、Model 适配能否物化、Agent 与 Session Permission 是否禁用、本次 User Message 是否关闭。

Permission 既影响可见集合，也可能在执行时触发 `ask()`。但 OpenCode Permission 不等于操作系统 Sandbox；进程级文件和网络隔离属于另一安全层。详细边界见第 09 篇。

### 5.4 名称、描述和 Schema 本身就是 Context

即使 executor 完全正确，模糊的定义仍会让模型选错。例如模型要读取教程入口时，需要从 Tool definitions 判断哪个 Tool 用来读文件、哪个用来搜索，`filePath` 怎样解释，`offset` 和 `limit` 是什么，以及哪些行为可能修改环境。

这些信息是模型推理的直接输入。Tool description 决定“何时选”，Schema 决定“怎样表达参数”，System 与 Permission 决定“是否应当选”。高质量 Agent 不能只增加 Tool 数量，还要管理工具之间的语义重叠和本轮可见范围。

## 六、一条信息怎样在连续 Provider Turn 中改变身份

### 6.1 第一次判断：目标、规则和能力同时进入

用户刚提出学习请求时，第一次普通 Provider Turn 可能得到：

```text
System
├── 当前工作目录、worktree、平台和日期
├── ambient AGENTS.md 等项目规则
└── 可用 Skill / MCP guidance

Messages
└── User：请查看教程入口和项目规则，再给出学习顺序

Tool definitions
├── read：读取指定文件
├── glob / search：发现可能的入口
└── 其他经本轮策略允许的工具
```

模型此时知道目标和行为边界，也知道可以提出哪些观察动作，但还没有 README 内容。若它生成 `read(...)`，那只是本轮输出中的 Tool Call。

### 6.2 工具结算：外部信息先成为 Session 事实

`read` executor 真正读取文件后，Processor 把结果结算进 Assistant Message 的 Tool Part。此时 README 内容已经成为 Session 中发生过的观察。Agent Loop 返回外层继续，下一次迭代重新读取活跃历史、转换 Messages，并再次发起 Provider Turn。

### 6.3 第二次判断：Tool Result 进入 Messages

新的 Provider Turn 中，信息身份已经变化：

```text
第一轮之前：README 只是工作区中的外部文件
read 执行之后：README 正文是 Session 中已结算的 Tool Result
第二轮请求时：Tool Result 被转换为 Provider-visible Messages
```

如果 `read` 同时发现子目录附近的 Instructions，它们也位于这条 Tool Result 中；ambient 根规则仍由新的外层迭代重新进入 System。模型于是可以依据真实入口选择下一份材料，或已经足够时组织学习顺序。Harness 没有替代模型判断，但保证判断材料拥有可追溯来源。

### 6.4 同一来源在不同阶段可以有不同表示

| 信息 | 原始来源 | 本轮常见通道 | 后续可能变化 |
| --- | --- | --- | --- |
| 用户学习目标 | User Message | Messages | 保留在活跃历史，或被 Summary 概括 |
| 工作目录与日期 | Runtime / InstanceState | System | 下一次外层迭代重新生成 |
| 根项目规则 | ambient instruction files | System | 每轮重新加载；配置或文件变化可反映 |
| 子目录规则 | `read` 时按目标位置发现 | Tool Result in Messages | 历史中记录已加载路径；旧输出可被 Prune |
| README 正文 | 文件系统 | Tool Result in Messages | 可保留、截断、被 Summary 概括或替换 |
| `read` 参数结构 | Tool Registry / schema | Tool definitions | 按 Provider、Permission 和本轮开关重新物化 |

这张表比较同一主流程中“来源—身份—变化”三个字段，不是再造一套平铺模块。

## 七、内容过大时，截断、Compaction 与 Pruning 怎样改变可见性

### 7.1 截断先限制单次 Tool Result 的体积

Compaction 与 Pruning 处理的是已经形成的长历史；在它们发生之前，Tool Runtime 还会限制单次结果进入 Context 的体积。它解决的是更局部的问题：一次 `read`、Shell 或其他 Tool 如果直接返回数万行内容，当前 Provider Turn 后续的历史可能立刻被一个结果占满。

#### 7.1.1 `read` 用分页参数和硬上限返回局部窗口

当前 `read` 默认最多读取 2,000 行，同时限制单行最多 2,000 字符、一次读取的文本内容预算最多 50 KB。超过行数或字节上限时，结果会明确告诉模型本次显示的行区间，并给出下一次 `offset`：

```ts
const file = yield* lines(filepath, {
  limit: params.limit ?? DEFAULT_READ_LIMIT,
  offset: params.offset || 1,
})

if (file.cut) {
  output += `\n\n(Output capped at ${MAX_BYTES_LABEL}. ` +
    `Showing lines ${file.offset}-${last}. Use offset=${next} to continue.)`
}
```

这里没有把文件剩余部分伪装成“不存在”。模型得到的是一段有边界的观察，以及继续读取所需的游标信息。原始文件仍在文件系统中，下一轮可以按 `offset` 读取相关区间。

#### 7.1.2 通用 Tool 截断保留预览，并把完整输出移出 Context

没有自行声明截断状态的 Tool 还会经过通用 `Truncate.output()`。默认上限同样是 2,000 行或 50 KB，也可以由 `tool_output.max_lines` 与 `tool_output.max_bytes` 配置。超过上限时，它保留头部或尾部预览，把完整文本写入专用截断目录，并在返回中给出保存路径与继续使用 `Grep`、`Read` 或合适 Subagent 的提示。

```text
完整 Tool output
-> 超过本轮输出上限
-> Provider-visible Tool Result = 局部预览 + 截断说明 + 完整输出路径
-> 后续按需搜索或分段读取
```

这也是渐进披露，但与 Skill 的两阶段加载不同：Skill 先暴露目录再加载正文；Tool 截断先保留本次结果的局部表示，再让后续 Turn 按需要取得其余内容。两者都在减少无关信息一次性进入 Context，却没有删除真实来源。

### 7.2 Compaction 不是删除整个 Session，而是重建活跃历史

#### 7.2.1 Summary 与近期 Tail 被重新排列为模型输入

当历史接近模型窗口边界时，OpenCode 可以创建 Compaction 请求，让模型生成 Summary，并保留一段近期 Tail。完成后，`filterCompacted()` 为模型消费重组顺序：

生成 Summary 时也不会把选中历史中的每个旧 Tool output 原样塞给 Compaction Model。`SessionCompaction.serialize()` 会把单个已完成 Tool output 限制在 2,000 字符；若 Part 已被 Pruning 标记，则直接使用旧结果已清理的占位符。这个局部序列化上限只服务于 Summary 请求，不会改写持久化 Tool Part。

```ts
if (tailIndex >= 0 && tailIndex < compactionIndex && summaryIndex > compactionIndex) {
  return [
    ...result.slice(compactionIndex, summaryIndex + 1),
    ...result.slice(tailIndex, compactionIndex),
    ...result.slice(summaryIndex + 1),
  ]
}
```

```text
Compaction User marker
-> Summary Assistant Message
-> 被选中的近期 Tail
-> Summary 之后的新消息 / continue User Message
```

#### 7.2.2 重排后不能再用数组位置判断“最新”

上述数组位置不再等于严格时间顺序。因此 `MessageV2.latest()` 不靠“数组最后一项”判断最新消息，而比较创建时间，并用 ID 作为确定性 tie-breaker。

较早的逐条交互不必全部继续可见，模型依靠 Summary 获取长期任务状态，同时保留最近细节。Compaction 的触发、Tail 选择和持久化由[第 10 篇](./10_Session_and_Persistence.md)主讲。

### 7.3 Pruning 只改变旧 Tool output 的模型投影

#### 7.3.1 先选择足够旧、足够大的可清理结果

固定源码中的 Pruning 会从较旧的已完成 Tool Part 向后选择，保护近期约 40,000 tokens 和 `skill` Tool；累计可清理量超过 20,000 tokens 后，给选中的 Part 写入 `state.time.compacted` 标记。

它还跳过最近一个 User Turn，遇到已有 Summary 或已标记的 Tool Part 时停止向前寻找。这里的 40,000 与 20,000 是固定源码的默认控制常量，不是所有配置、未来版本或每种 Model 都不变的通用规律。

#### 7.3.2 标记在后续 Messages 投影时才替换正文

重要的不是记住两个数字，而是行为边界：Pruning 标记 Tool output 的可见性状态，不是把整个 Message 或原始 output 从持久化记录中物理删除。下一次 `toModelMessagesEffect()` 看见标记后，才把 Provider-visible output 替换为：

```text
[Old tool result content cleared]
```

模型仍能看到这里曾发生 Tool Call，知道调用了哪个 Tool、使用什么输入，并知道旧结果已清理。Summary 若提前吸收关键结论，也能继续承载任务状态。

### 7.4 截断、Summary、Tail 与占位符解决不同问题

- **Tool 截断** 控制一次结果立即进入 Context 的体积，并保留继续取得原始内容的路径；
- **Summary** 把较长历史中的目标、已完成工作、发现和后续方向重新表达为短文本；
- **Tail** 保留最近交互的原始细节，让模型不必只依赖概括；
- **Pruned Tool placeholder** 只替换旧 Tool output 的模型表示，同时保留调用结构。

它们共同改变未来 Context，却不等同于代码 Snapshot、文件 Revert 或系统级长期记忆。Compaction 不会自动恢复被 Tool 修改的文件；Pruning 也不会把文件系统回退到调用前。

## 八、Context Architecture 的边界与 native V2 简述

### 8.1 模型看不见的内容不会因为“系统知道”而自动生效

OpenCode 进程可以知道许多没有进入本轮请求的事实：数据库中的旧 Part、隐藏 Agent、不可见 Tool 的 executor、UI 本地状态、尚未读取的文件。它们可以影响 Harness 控制，却不一定参与模型判断。

同样，模型生成的自然语言也不自动改变 Harness 状态。它说“接下来只读”不等于 Permission rules 已修改；它说“已读取文件”不等于存在已结算 Tool Result。Context 提供判断材料，确定性控制仍由 Harness 执行。

### 8.2 Context 不是 Session，也不是模型长期记忆

Session 是持续交互的状态容器；Context 是某一次 Provider Turn 的模型输入投影。一个 Session 会产生多次不同 Context，同一份 durable history 也可能因 Model、Agent、Permission、Compaction 状态或 Tool 开关不同而形成不同请求。

模型上下文窗口只在当前调用中工作。下一次调用能够延续任务，是 Harness 重新取得 Session 信息并组织输入，而不是模型 API 内部保存了完整项目记忆。

### 8.3 当前默认路径与 native V2 不应混写

本文主线是当前默认 TUI 使用的兼容 Session API、`SessionPrompt`、`MessageV2.toModelMessagesEffect()` 和 `LLMRequestPrep.prepare()`。

固定源码中的 native V2 正在把 Context 边界表达得更显式，例如 typed System Context sources、Context Snapshot 与 Epoch。它强调某个 Turn 使用哪一份已冻结上下文、何时因为新输入或状态变化进入下一 Epoch。这些概念有助于理解未来可追溯性，但不能反写成当前默认 `SessionPrompt` 已完全采用相同结构。

两条架构的演进、运行入口与恢复差异集中在[第 12 篇](./12_Runtime_Boundary.md)。在本文只保留共同心智模型：**Provider Turn 总要消费一份有边界的输入；不同 Runtime 的区别在于这份输入何时形成、怎样冻结、怎样追踪版本。**

## 九、关键源码索引

正文已经展示理解主流程所需的关键代码。以下索引用于继续回到固定源码，不用文件清单代替前面的机制解释。

| 主题 | 源码文件 | 关键符号 |
| --- | --- | --- |
| 每轮活跃历史与 Context 组装 | `packages/opencode/src/session/prompt.ts` | `SessionPrompt.run` |
| 最终 System、Messages、Tools 与参数准备 | `packages/opencode/src/session/llm/request.ts` | `LLMRequestPrep.prepare`、`resolveTools` |
| Provider prompt、Environment、Skill 与 MCP guidance | `packages/opencode/src/session/system.ts` | `provider`、`environment`、`skills`、`mcp` |
| ambient 与按路径发现的项目规则 | `packages/opencode/src/session/instruction.ts` | `systemPaths`、`system`、`resolve` |
| `read` 附加附近 Instructions | `packages/opencode/src/tool/read.ts` | `ReadTool.execute` |
| Part 到 Provider Messages 的投影 | `packages/opencode/src/session/message-v2.ts` | `toModelMessagesEffect`、`filterCompacted`、`latest` |
| Tool definition 物化 | `packages/opencode/src/session/tools.ts` | `SessionTools.resolve` |
| `read` 的分页与输出上限 | `packages/opencode/src/tool/read.ts` | `ReadTool.lines`、`ReadTool.execute` |
| 通用 Tool output 截断 | `packages/opencode/src/tool/tool.ts`、`packages/opencode/src/tool/truncate.ts` | `Tool.define` 包装、`Truncate.output`、`limits` |
| Summary、Tail 与旧 Tool output 标记 | `packages/opencode/src/session/compaction.ts` | `serialize`、`select`、`prune`、`process` |

代表性测试入口包括：

- `packages/opencode/test/session/instruction.test.ts`：规则路径选择、配置 instructions 与按文件读取发现；
- `packages/opencode/test/session/message-v2.test.ts`：Message / Part 转换、Compaction 顺序和 Tool 状态表示；
- `packages/opencode/test/session/compaction.test.ts`：Tail 选择与 Pruning；
- `packages/opencode/test/session/prompt.test.ts`：新的外层迭代怎样利用 Tool Result 继续。

完整跨章节证据表见[源码与证据索引](./appendices/Source_Index.md)。

## 十、总结：Context 是 Harness 为一次判断搭建的信息现场

```text
Durable Session History + 当前 Runtime / 配置 / 能力
                         │
                         ▼
              选择 Compaction 后活跃历史
                         │
              ┌──────────┼──────────┐
              ▼          ▼          ▼
           System     Messages    Tool definitions
              │          │          │
              └──────────┼──────────┘
                         ▼
            Provider-specific 最终转换
                         │
                         ▼
                本次模型实际看见的 Context
```

System 说明这一轮应遵守什么，Messages 表达迄今发生了什么，Tool definitions 声明这一轮可以提议什么。Environment 与 ambient Instructions 会在新的外层迭代中重新生成；Tool Result 先结算成 Session 事实，再在下一轮转换为 Messages；Tool definition 则按 Model、Agent、Permission 和本轮开关重新物化。

内容过大时，单次 Tool 截断先用局部预览和继续读取路径限制一次 Observation；历史继续增长后，Compaction 用 Summary 与近期 Tail 重建活跃历史，Pruning 再用占位符替换旧 Tool output 的模型投影。它们都改变“未来看见什么”，却不等于删除整个 Session 或回滚外部世界。

理解这一点后，下一篇就可以继续追问：当模型依据这些信息生成 Tool Call 时，OpenCode 怎样把“调用意图”变成经过 Schema 验证、Permission 判断和 executor 执行的真实操作？

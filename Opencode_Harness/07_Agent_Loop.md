# Agent Loop：一次请求为什么可以持续多轮

上一篇：[06 Harness 总览](./06_Harness.md) ｜ 下一篇：[08 Context Architecture](./08_Context_Architecture.md)

> 固定源码：OpenCode `0e3474509aa5ad16afcf9c439785514d6443c6af`（`dev`，2026-08-18）
>
> 分析主线：当前默认 TUI 的普通消息路径，即兼容 Session API 与 `SessionPrompt` 编排。文末单独说明 native LLM adapter 与 native V2 的边界。

先想象一位刚接触 OpenCode 的学习者提出了下面的问题：

> 请先查看 Harness 教程入口和项目规则，再告诉我应该按照什么顺序学习。

如果这只是普通聊天，模型只能依据提问时已经拥有的信息直接作答。但教程入口和项目规则位于本地文件中，模型在第一次回答前并不知道它们的内容。OpenCode 因此不能只做一次“输入 → 模型 → 输出”，而要经历一个反馈过程：

```text
先判断需要什么信息
-> 读取真实文件
-> 得到新的观察
-> 根据观察重新判断
-> 信息足够后再回答
```

这个反复进行“判断、行动、观察”的控制过程，就是 **Agent Loop（Agent 循环）**。

可以把模型类比成一位只能通过窗口提出请求的分析者：它能说“我想读取 README”，却不能自己伸手操作文件系统。Harness 像窗口外的执行与管理系统，负责检查请求、执行工具、记录结果，再把结果放到下一张材料中交还给模型。只要任务还没有到达结束条件，Harness 就会再次询问模型。

Agent Loop 的核心价值不是“让模型思考更久”，而是允许模型根据真实世界反馈修正下一步：

```text
一次性生成：目标 -> 模型猜测 -> 最终答案

Agent Loop：目标 -> 模型判断 -> 真实行动 -> 新观察
                ^                           |
                |___________________________|
```

这里的判断仍然具有概率性：模型可能先读 README，也可能先搜索规则文件。Loop 提供的是可重复、可控制、可记录的执行框架，而不是保证模型每次都选择最佳路径。

## 一、Agent Loop 全景图

### 1.1 四个参与角色

OpenCode 的一次循环可以先抽象为四个角色：

```text
┌──────────────────────────────────────────────────────────────┐
│                         Agent Loop                           │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Session / History                                           │
│  保存用户目标、Assistant 输出、Tool Call 与 Tool Result        │
│             │                                                │
│             ▼                                                │
│  Harness / Orchestrator                                      │
│  重载状态、组织 Context、物化 Tools、判断继续或停止             │
│             │                                                │
│             ▼                                                │
│  Model                                                       │
│  根据本轮 Context 生成文本或提出 Tool Call                     │
│             │                                                │
│             ▼                                                │
│  Tool Runtime                                                │
│  执行读取、搜索、命令等真实操作，并产生 Tool Result             │
│             │                                                │
│             └──────── 新观察写回 Session ────────────────┐    │
│                                                         │    │
└─────────────────────────────────────────────────────────┼────┘
                                                          │
                         下一轮重新读取 ◀──────────────────┘
```

四者的职责不能互换：

- **Model** 负责提出下一步，但不直接读取文件或保存 Session。
- **Harness** 负责把模型提议放进确定性的执行边界，并控制循环方向。
- **Tool Runtime** 负责真实 I/O；操作可能成功，也可能返回错误。
- **Session / History** 保存已经发生的事实，使下一轮能够取得新的观察。

### 1.2 `Think -> Act -> Observe` 的工程含义

Agent 资料常用 `Think -> Act -> Observe` 描述反馈循环。在 OpenCode 中，可以把它翻译为：

| 抽象阶段 | OpenCode 中的对应行为 | 不能误解成什么 |
| --- | --- | --- |
| Think | Model 根据本轮 Context 生成文本或 Tool Call | 逐字暴露模型内部隐藏思维 |
| Act | Harness 校验、授权并调用 Tool executor | 模型自己操作文件系统 |
| Observe | Tool Result 被结算、保存并进入后续历史 | 工具输出自动变成正确结论 |
| Loop | Harness 根据状态再次请求模型或结束 | Provider 内部有一条永久线程 |

因此，Agent Loop 是模型外部的工程控制流。模型每次只处理当前请求；“持续完成任务”的能力来自 Harness 反复组织请求并把真实结果反馈回去。

### 1.3 模型判断与 Harness 控制的分界

Agent Loop 同时包含概率性决策和确定性控制。下面的表格不是按流程排列，而是在比较同一轮中不同问题的主要负责者：

| 问题 | 主要负责者 | 性质 |
| --- | --- | --- |
| 下一步先读 README 还是先找规则 | Model | 概率性提议 |
| 哪些 Tools 出现在当前请求中 | Harness | 根据 Agent、Session、Model 与配置物化 |
| Tool Call 参数是否符合 schema | Harness | 确定性验证 |
| 是否需要用户批准 | Harness + User | Permission 流程 |
| 文件是否真的读取成功 | Tool Runtime | 真实 I/O，可能失败 |
| Tool Result 怎样进入 Part | Harness | 状态转换与记录 |
| 是否发起下一 Provider Turn | Harness | Loop、Processor Result 与 terminal check |
| Model 是否正确理解 Tool Result | Model | 概率性判断 |

Harness 能保证的是过程边界，例如未通过参数验证的 Tool 不应进入正常 executor、需要批准的动作会等待 Permission、执行结果会按既定 Part 状态结算。它不能保证模型一定选择正确文件，也不能保证模型读完结果后形成正确结论。

这正是 Loop 需要反馈的原因：系统无法一次确定模型的完整行动序列，只能让每一步受到约束，并让后续判断基于已经发生的事实。

### 1.4 一个请求如何产生三次模型判断

仍以学习 Harness 为例，一条可能的运行轨迹是：

```text
用户请求
  “查看教程入口和项目规则，给出学习顺序”
        │
        ▼
Provider Turn 1
  Model -> read(Opencode_Harness/README.md)
  Tool Runtime -> 返回教程目录
        │
        ▼
Provider Turn 2
  Model -> read(适用的 AGENTS.md)
  Tool Runtime -> 返回项目规则
        │
        ▼
Provider Turn 3
  Model -> 综合两次观察，输出学习顺序
        │
        ▼
Harness -> 终止检查通过，Session 进入空闲
```

读取顺序不是固定脚本；稳定的是“Tool Result 成为新观察，新观察推动下一次判断”这一反馈结构。

## 二、先分清循环中的三个时间尺度

用户只发送了一条消息，不代表 OpenCode 只请求了一次模型。理解源码前，需要先区分三个嵌套尺度。

### 2.1 用户请求：完整任务目标

用户请求从一条 User Message 开始，描述希望系统完成的整体目标。它可以只产生一次模型请求，也可以跨越多个工具行动和模型请求。

### 2.2 Provider Turn：一次真实模型请求

本系列把下面这一段称为一个 **提供商轮次（Provider Turn）**：

```text
一次真实 Provider Request attempt
-> Provider 返回响应，或请求以错误结束
-> OpenCode 把结果投影到当前 Assistant / Parts
```

一次用户请求可能包含多个 Provider Turn。Retry 每重新发起一次真实 Provider Request，也构成一个新的物理 Provider Turn，只是它可能继续复用同一个 Assistant Message 和 Processor context。

### 2.3 流式事件：Provider Turn 内部的增量

一个 Provider Turn 的响应不是只能在最后一次性到达。OpenCode 会消费一系列流式事件，例如：

- `text-start / text-delta / text-end`；
- `reasoning-start / reasoning-delta / reasoning-end`；
- `tool-input-start / tool-input-delta / tool-call`；
- `tool-result / tool-error`；
- `step-start / step-finish`；
- Provider error 与最终 finish 信息。

三者的关系是包含，不是三个依次执行的平级模块：

```text
一次用户请求
├── Provider Turn 1
│   ├── text / tool-input 等流式事件
│   └── Tool Result
└── Provider Turn 2
    ├── text 等流式事件
    └── finish
```

## 三、Agent Loop 在 OpenCode 的哪一层运行

### 3.1 当前默认 TUI 的入口链

#### 3.1.1 普通消息进入兼容 Session API

普通消息从 TUI 进入兼容 Session API，再由 `SessionPrompt` 驱动：

```text
TUI submit
-> sdk.client.session.prompt(...)
-> POST /session/:sessionID/message
-> SessionHttpApi.prompt
-> SessionPrompt.prompt
-> SessionPrompt.loop
-> SessionPrompt.run
```

TUI 提交处的关键代码位于 `packages/tui/src/component/prompt/index.tsx`：

```ts
sdk.client.session
  .prompt(
    {
      sessionID,
      ...selectedModel,
      agent: agent.name,
      model: selectedModel,
      variant,
      parts: [
        ...editorParts,
        {
          type: "text",
          text: inputText,
        },
        ...nonTextParts,
      ],
    },
    { throwOnError: true },
)
```

#### 3.1.2 请求返回与实时事件是两条通道

这段代码提交的是 User Message 所需的模型、Agent 和 Parts。真正的多轮编排不在 TUI 组件中，而在服务端 Session 层。兼容 Prompt Handler 会等待 `promptSvc.prompt(...)` 得到最终 `WithParts` 后返回一个 JSON 值；TUI 运行中的增量更新来自独立事件通道，而不是这个 POST 逐 token 返回。

### 3.2 `prompt()` 先保存用户输入，再进入 Loop

`packages/opencode/src/session/prompt.ts` 中的 `SessionPrompt.prompt` 可以看到两个明确阶段：

```ts
const message = yield* createUserMessage(input)
yield* sessions.touch(input.sessionID)

// 根据本次输入更新 Tool Permission override
// ...

if (input.noReply === true) return message
return yield* loop({ sessionID: input.sessionID })
```

这意味着 User Message 先成为 Session 中的事实，然后系统才决定是否立即运行 Loop。`noReply` 可以只保存输入而不在当前调用中等待模型回复；普通 TUI 请求则继续进入 `loop()`。

### 3.3 `SessionPrompt.run` 中存在显式 `while (true)`

Loop 不是对 Provider “自动 Agent 能力”的别名。固定源码中，真正的外层循环直接写在 OpenCode 编排层：

```ts
const runLoop = Effect.fn("SessionPrompt.run")(function* (sessionID) {
  let step = 0

  while (true) {
    yield* status.set(sessionID, { type: "busy" })
    let msgs = yield* MessageV2.filterCompactedEffect(sessionID)

    // terminal check、特殊任务、Context 与 Tools 组装
    // 创建 Assistant Message，调用 Processor

    if (outcome === "break") break
    continue
  }

  yield* compaction.prune({ sessionID }).pipe(Effect.ignore, Effect.forkIn(scope))
  return yield* lastAssistant(sessionID)
})
```

Provider 只负责一次请求的生成；OpenCode 决定何时再次调用 Provider、何时压缩历史以及何时退出。

### 3.4 同一 Session 共享一个活动 Runner

#### 3.4.1 已有 Runner 时复用当前运行

`SessionPrompt.loop` 本身很短：

```ts
const loop = Effect.fn("SessionPrompt.loop")(function* (input) {
  return yield* state.ensureRunning(
    input.sessionID,
    lastAssistant(input.sessionID),
    runLoop(input.sessionID),
  )
})
```

`SessionRunState` 在进程内维护 `Map<SessionID, Runner>`。如果同一 Session 已经存在 Runner，新的调用会取得现有 Runner，而不是再创建一条互相竞争的外层循环：

```ts
const existing = data.runners.get(sessionID)
if (existing) return existing

const next = Runner.make(data.scope, {
  onIdle: Effect.gen(function* () {
    data.runners.delete(sessionID)
    yield* status.set(sessionID, { type: "idle" })
  }),
  onBusy: status.set(sessionID, { type: "busy" }),
  onInterrupt,
})
data.runners.set(sessionID, next)
```

#### 3.4.2 Runner 是进程内协调，不是持久化 Job

这个协调只说明“当前进程内，同一 Session 不启动两套独立 Loop”。Runner 本身不是数据库中的持久化 Job，也不是跨进程租约；进程退出后，Message 和 Part 可以保留，活动 Runner 不会仅凭历史记录自动复活。

## 四、一次外层循环究竟做了什么

这是理解 Agent Loop 最关键的部分。忽略少见错误后，每次 `while (true)` 迭代可以整理为：

```text
1. 将 Session 标记为 busy
2. 重新读取经过 Compaction 过滤的活跃历史
3. 找到最新 User、Assistant、已完成消息和特殊任务
4. 执行顶部终止检查
5. 根据最新 User Message 解析 Model，再优先处理 Subtask / Compaction / Context Overflow
6. 普通路径继续解析 Agent，并应用 Session reminders
7. 创建本轮 Assistant Message
8. 物化 Tools，组装 System 与 Model Messages
9. 创建 SessionProcessor，发起一个 Provider Turn
10. 把流式事件结算为 Message / Part
11. 根据 continue / compact / stop 决定方向
12. 若需继续，回到顶部重新读取状态
```

### 4.1 每轮重载活跃历史

Loop 顶部不是复用一份永不变化的内存数组，而是重新读取：

```ts
let msgs = yield* MessageV2.filterCompactedEffect(sessionID)

const {
  user: lastUser,
  assistant: lastAssistant,
  finished: lastFinished,
  tasks,
} = MessageV2.latest(msgs)
```

重新读取带来三个重要效果：

- 上一轮完成的 Tool Result 可以进入下一轮 Context；
- Loop 运行期间新到达的 User Message 可以改变后续判断；
- Compaction 改写了活跃历史后，下一轮使用新的可见结果。

这里取得的是“当前模型可用的活跃历史”，不是把数据库中的所有原始记录按时间顺序原样塞给模型。具体选择、转换和裁剪机制由[第 08 篇](./08_Context_Architecture.md)展开。

### 4.2 顶部终止检查不是只看 `finish`

#### 4.2.1 终止检查的四个条件

OpenCode 会同时检查最新 Assistant、最新 User 和 Tool Parts：

```ts
const hasToolCalls =
  lastAssistantMsg?.parts.some(
    (part) =>
      part.type === "tool" &&
      !part.metadata?.providerExecuted &&
      !isOrphanedInterruptedTool(part),
  ) ?? false

if (
  lastAssistant?.finish &&
  !["tool-calls"].includes(lastAssistant.finish) &&
  !hasToolCalls &&
  lastAssistant.parentID === lastUser.id
) {
  break
}
```

四个条件共同表达“当前最新用户目标已经有一个完成的 Assistant 响应，并且没有普通本地 Tool 结果需要再反馈给模型”。

#### 4.2.2 为什么 `finish=stop` 仍可能继续

这也解释了两个看似反常的行为：

1. Provider 返回 `stop`，但 Assistant 中仍有本地 Tool Part 时，Loop 仍可能继续，把 Tool Result 交回模型。
2. 运行期间出现新的 User Message 后，旧 Assistant 的 `parentID` 不再对应最新 User，Loop 不会把旧回复误判为新请求已经完成。

`providerExecuted` Tool 与普通本地 Tool 的执行归属不同；被 cleanup 标记的中断孤儿 Tool 也不能永远阻塞结束。因此终止检查会显式排除这些情况。

### 4.3 特殊任务先于普通 Provider Turn

#### 4.3.1 三种优先分支的控制顺序

通过顶部检查并解析 Model 后，Loop 不一定立刻请求模型。它会先查看特殊任务与上下文容量：

```ts
const task = tasks.pop()

if (task?.type === "subtask") {
  yield* handleSubtask(/* ... */)
  continue
}

if (task?.type === "compaction") {
  const result = yield* compaction.process(/* ... */)
  if (result === "stop") break
  continue
}

if (lastFinished && (yield* compaction.isOverflow({ tokens: lastFinished.tokens, model }))) {
  yield* compaction.create({ /* ... */ })
  continue
}
```

#### 4.3.2 特殊分支不一定产生 Provider Turn

因此，一次外层迭代可能只处理 Compaction，然后回到顶部，并不产生 Provider Turn。

#### 4.3.3 `SubtaskPart` 不等于 `task` Tool Call

这里的 `SubtaskPart` 属于兼容命令编排数据，不等于模型在普通 Provider Turn 中调用的 `task` Tool。Task/Subagent 的完整机制由[第 11 篇](./11_Agent_Specialization_and_Collaboration.md)说明。

### 4.4 先创建 Assistant Message，再接收流式输出

#### 4.4.1 Provider 请求前先建立输出容器

普通 Provider Turn 开始前，OpenCode 先创建一条 Assistant Message：

```ts
const msg: SessionV1.Assistant = {
  id: MessageID.ascending(),
  parentID: lastUser.id,
  role: "assistant",
  mode: agent.name,
  agent: agent.name,
  variant: lastUser.model.variant,
  path: { cwd: ctx.directory, root: ctx.worktree },
  cost: 0,
  tokens: { input: 0, output: 0, reasoning: 0, cache: { read: 0, write: 0 } },
  modelID: model.id,
  providerID: model.providerID,
  time: { created: Date.now() },
  sessionID,
}
yield* sessions.updateMessage(msg)
```

#### 4.4.2 User Message、Assistant Message 与 Provider Turn 的边界

随后到达的 Text、Reasoning、Tool、Usage、Finish 和 Error 都可以归属到这条 Assistant Message。于是三个概念有了明确边界：

```text
User Message：描述当前用户目标
Assistant Message：承载一次处理上下文产生的输出和 Parts
Provider Turn：该上下文中一次真实的 Provider Request attempt
```

一次用户请求可以产生多条 Assistant Message；Retry 又可能在同一条 Assistant Message 中产生多个物理 Provider Turn。

### 4.5 Context 与 Tools 在新的外层迭代中重新组装

#### 4.5.1 从当前状态物化两组请求材料

OpenCode 依据最新 User Message 选择 Model 和 Agent，物化本轮 Tools，并取得环境、指令、Skills 与模型消息：

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

const [skills, env, instructions, mcpInstructions, modelMsgs] =
  yield* Effect.all([
    sys.skills(agent),
    sys.environment(model),
    instruction.system().pipe(Effect.orDie),
    sys.mcp(agent, session.permission),
    MessageV2.toModelMessagesEffect(msgs, model),
  ])
```

#### 4.5.2 “下一轮”意味着重新组织，而不只是追加

这说明“下一轮”不只是往上次请求后追加一段 Tool Result。Harness 会基于当前 Session 状态创建新的 Assistant/Processor context，并重新组织请求材料。

#### 4.5.3 Retry 不经过这次重组

与此相对，Provider Retry 不回到 `while (true)` 顶部，因此不会完成这次重载和重组。这个差异将在第 7.1 节详细说明。

## 五、Provider Turn 与流式结算

### 5.1 每次普通外层迭代调用一次 `handle.process()`

Context 准备完成后，`SessionPrompt.run` 把请求交给当前 Processor：

```ts
const result = yield* handle.process({
  user: lastUser,
  agent,
  permission: session.permission,
  sessionID,
  parentSessionID: session.parentID,
  system,
  messages: [
    ...modelMsgs,
    ...(isLastStep ? [{ role: "assistant", content: MAX_STEPS_PROMPT }] : []),
  ],
  tools,
  model,
  toolChoice: format.type === "json_schema" ? "required" : undefined,
})
```

`handle.process()` 对应当前 Assistant/Processor context 的响应处理入口。一般情况下它调用一次 `llm.stream(streamInput)`；若发生可重试错误，Retry policy 会在这个入口内部重新运行同一请求材料。

### 5.2 `SessionProcessor` 把流事件转换为持久状态

#### 5.2.1 不同事件驱动不同 Part 状态

`packages/opencode/src/session/processor.ts` 中，流式事件不是只负责刷新 UI。它们会驱动 Message 和 Part 状态变化：

| 流事件 | 主要结算结果 |
| --- | --- |
| `text-start / delta / end` | 创建 Text Part、追加 delta、记录完整文本和结束时间 |
| `reasoning-*` | 创建和更新 Reasoning Part |
| `tool-input-* / tool-call` | 创建 Tool Part，并从 pending 转为 running |
| `tool-result` | 将 Tool Part 结算为 completed，并保存 output 与 attachments |
| `tool-error` | 将 Tool Part 结算为 error |
| `step-finish` | 保存 finish reason、tokens、cost、snapshot / patch 等信息 |
| Provider error | 转换为 Assistant Error，并影响 Processor 返回方向 |

#### 5.2.2 Text delta 的实时发布与完整保存

Text delta 的处理能体现“实时显示”和“完整保存”是两个相关但不同的动作：

```ts
case "text-delta":
  if (!ctx.currentText) return
  ctx.currentText.text += value.text
  yield* session.updatePartDelta({
    sessionID: ctx.currentText.sessionID,
    messageID: ctx.currentText.messageID,
    partID: ctx.currentText.id,
    field: "text",
    delta: value.text,
  })
  return
```

Processor 一边累计当前 Text Part，一边发布增量；到 `text-end` 或 cleanup 时再保存具有结束状态的 Part。Session 的 durable、process-local 与 live-only 边界由[第 10 篇](./10_Session_and_Persistence.md)展开。

### 5.3 Tool Call 在当前流中执行，下一轮利用结果

#### 5.3.1 `tool-result` 如何完成 Tool Part

默认 AI SDK 路径中，普通本地 Tool 通常在当前 `llm.stream` 的工具执行阶段完成。Processor 收到 `tool-result` 后完成 Tool Part。源码会先规范化图片附件，再结算完整输出：

```ts
case "tool-result": {
  const toolCall = yield* readToolCall(value.id)
  if (!toolCall && value.result.type === "error") return

  if (value.result.type === "error") {
    yield* failToolCall(value.id, value.result.value)
    return
  }

  const rawOutput = toolResultOutput(value)
  const normalized = yield* Effect.forEach(rawOutput.attachments ?? [], (attachment) =>
    attachment.mime.startsWith("image/")
      ? image.normalize(attachment).pipe(
          Effect.catchIf(
            (error) => error instanceof Image.ResizerUnavailableError,
            () => Effect.succeed(attachment),
          ),
          Effect.exit,
        )
      : Effect.succeed(Exit.succeed<SessionV1.FilePart>(attachment)),
  )
  const omitted = normalized.filter(Exit.isFailure).length
  const attachments = normalized.filter(Exit.isSuccess).map((item) => item.value)
  const output = {
    ...rawOutput,
    output:
      omitted === 0
        ? rawOutput.output
        : `${rawOutput.output}\n\n[${omitted} image${omitted === 1 ? "" : "s"} omitted: could not be resized below the image size limit.]`,
    attachments: attachments.length ? attachments : undefined,
  }
  yield* completeToolCall(value.id, output)
  return
}
```

#### 5.3.2 执行、结算与下一轮的时间关系

完整关系是：

```text
本轮 Provider Turn
  Model 提出 Tool Call
        │
        ▼
  Tool executor 在当前处理阶段执行
        │
        ▼
  Processor 把 Tool Part 结算为 completed / error
        │
        ▼
  Processor 返回 continue
        │
        ▼
下一轮外层迭代重载历史
  Model 现在可以利用 Tool Result
```

#### 5.3.3 下一轮负责利用结果，不是执行上轮工具

所以下一轮不是“开始执行上一轮工具”，而是“读取已经发生的 Tool 结果，再决定下一步”。

## 六、循环方向：Continuation、Compaction 与停止

### 6.1 Processor 只返回三个方向

`SessionProcessor.process` 最终把复杂的流式处理压缩成三个结果：

```ts
if (ctx.needsCompaction) return "compact"
if (ctx.blocked || ctx.assistantMessage.error) return "stop"
return "continue"
```

| Processor Result | 外层 Loop 的动作 |
| --- | --- |
| `continue` | 回到顶部，重载历史并重新判断 |
| `compact` | 创建或进入 Compaction 流程，再决定继续或停止 |
| `stop` | 将当前外层运行转为 `break` |

`continue` 并不保证一定还会调用一次模型。它只表示 Processor 没有直接要求停止；回到顶部后，terminal check 仍可能立即判断任务已经完成。

### 6.2 `continue`：回到顶部重新判断

#### 6.2.1 为什么普通最终文本也可能先返回 `continue`

当模型只返回最终文本，没有 blocked、Assistant Error 或 Context Overflow 时，Processor 仍会返回 `continue`。外层 Loop 回到顶部并重新读取历史，再使用[第 4.2 节](#42-顶部终止检查不是只看-finish)的完整条件判断当前最新用户目标是否已经得到回复。若检查通过，Loop 直接 `break`，不会产生新的 Provider Turn。

这种两阶段设计把职责分开：Processor 负责处理本次流，Loop 顶部根据完整 Session 状态判断是否已经到达空闲边界。系统不必只相信 Provider 给出的一个 finish reason。

#### 6.2.2 失败的 Tool Result 也可以成为观察

假设模型尝试读取一个不存在的规则文件：

```text
Tool Call：read("不存在的路径")
-> Tool Result：error
-> Processor 保存 error Tool Part
-> 外层 Loop continuation
-> 下一 Provider Turn 看见失败事实
-> Model 改为搜索 AGENTS.md
```

Tool Error 不必然等于 Session Error。只要错误被作为 Tool Result 正常结算，模型仍可能利用这个失败观察修正策略。相反，Provider error、blocked 状态或无法正常结算的系统错误可能让 Processor 返回 `stop`。

### 6.3 `compact`：先改变可见历史

#### 6.3.1 Context Overflow 的两条入口

Context Overflow 有两个入口：Loop 顶部可以根据上一条已完成 Assistant 的 tokens 预先创建 Compaction；流处理中也可能发现 Provider overflow，由 Processor 设置 `needsCompaction` 并返回 `compact`。后一条路径不会进入普通 Provider Retry：

```ts
if (result === "compact") {
  yield* compaction.create({
    sessionID,
    agent: lastUser.agent,
    model: lastUser.model,
    auto: true,
    overflow: !handle.message.finish,
  })
}
return "continue"
```

#### 6.3.2 下一次外层迭代处理 Compaction Part

下一次外层迭代会优先处理 Compaction Part。Compaction processor 可以继续，也可以停止。它怎样改变未来模型可见的历史属于 Context 与 Persistence 的交界，分别由[第 08 篇](./08_Context_Architecture.md)和[第 10 篇](./10_Session_and_Persistence.md)展开。

### 6.4 停止：正常空闲与直接终止是两条路径

#### 6.4.1 正常完成后进入空闲边界

最常见的结束路径是：

```text
Provider 返回最终文本
-> Processor 正常结算并返回 continue
-> Loop 回到顶部重载历史
-> 第 4.2 节的 terminal check 通过
-> break
-> 异步执行 prune
-> 返回最后一条 Assistant Message
-> Runner 进入 idle，并从进程内 Map 删除
```

这可以理解为“一次运行把当前可继续工作处理到空闲边界”，但不能把它命名为持久化 Session Job 或跨进程 Drain。

#### 6.4.2 直接结束当前分支的情况

当前默认路径还会在这些情况下结束相关分支：

- Processor 出现 blocked 状态或 Assistant Error，返回 `stop`；
- Provider Content Filter 被转换为明确错误；
- 要求 JSON Schema，但模型没有产出 Structured Output；
- Structured Output Tool 已得到目标对象；
- Compaction processor 判断应停止；
- 用户 Interrupt 取消活动 Runner；
- 无法恢复的 Provider、Tool 或存储边界错误终止处理。

这些情况并不都代表“用户目标已经成功完成”。停止只说明当前 Runner 不再继续发起下一 Provider Turn，成功、拒绝、中断和失败需要根据保存的 Assistant / Part 状态区分。

#### 6.4.3 三个不能单独证明停止的信号

**Provider finish reason** 不能单独证明停止。Provider 可能返回 `stop`，同时 Assistant 中仍存在需要把结果反馈给模型的本地 Tool Part。

**Todo 全部完成** 不能单独证明停止。Todo 是 Session 中的结构化进度，不是 Loop 调度器；pending Todo 也不会自动强制下一 Provider Turn。

**达到 `agent.steps`** 不能单独证明停止。当前兼容路径只是向模型追加提醒，没有形成硬执行闸门。

#### 6.4.4 历史可恢复不等于任务自动续跑

Message、Part 和部分事件可以持久化，下一次 Loop 也可以重新读取这些历史。但以下状态主要属于当前运行现场：

- `SessionRunState` 中的活动 Runner；
- 当前 Processor context；
- 正在累积的流；
- Retry backoff；
- 尚未完成的 Tool 协调与进程内 Deferred。

所以“重启后还能看到之前记录”与“重启后自动、安全地继续未完成工作”是两个不同结论。固定版本不能依据 durable history 推导出自动 crash continuation。

## 七、异常、等待与无效重复怎样影响 Loop

### 7.1 Retry：重新请求 Provider，不重建外层 Context

#### 7.1.1 Retry 位于 `SessionProcessor.process` 内部

某些 Provider 错误是临时性的，例如限流、服务不可用或连接中断。`SessionProcessor.process` 在同一处理上下文内应用 Retry policy。保留控制关系后的源码如下：

```ts
yield* Effect.gen(function* () {
  const stream = llm.stream(streamInput)
  yield* stream.pipe(
    Stream.tap((event) => handleEvent(event)),
    Stream.takeUntil(() => ctx.needsCompaction),
    Stream.runDrain,
  )
}).pipe(
  Effect.retry(
    SessionRetry.policy({
      provider: input.model.providerID,
      parse,
      set: (info) => status.set(ctx.sessionID, {
        type: "retry",
        attempt: info.attempt,
        message: info.message,
        next: info.next,
      }),
    }),
  ),
)
```

#### 7.1.2 Retry 与 Continuation 位于不同控制层

Retry 的位置决定了它与 continuation 的根本区别：

```text
Continuation
  返回 SessionPrompt.run 顶部
  -> 重载历史
  -> 创建新的 Assistant / Processor context
  -> 重组 Context 与 Tools

Retry
  停留在当前 SessionProcessor.process 内
  -> 复用 streamInput
  -> 不重载最新历史
  -> 不创建新的 Assistant Message
```

#### 7.1.3 错误分类、退避与次数

固定源码的 `SessionRetry` 具有以下策略：

- Context Overflow 明确不走通用 Retry；
- 优先读取 `retry-after-ms`、`retry-after` 等 Provider hint；
- 没有可用 hint 时使用指数退避，并增加 jitter；
- 无响应头时等待上限为 30 秒；
- 最多安排 5 次 Retry，也就是连同初次请求最多 6 个物理 attempt；
- 等待期间 Session Status 记录为 `retry`，包含 attempt、原因和下次时间。

#### 7.1.4 Provider Turn、Session Status 与状态边界

按本文定义，每次 Retry 都产生新的真实 Provider Request，因此是新的物理 Provider Turn；但这些 Provider Turns 共享同一 Assistant/Processor context。Retry 也不是数据库事务回滚，失败前已经投影的部分流事件可能保留。

### 7.2 Interrupt：停止未来执行，不撤销过去

#### 7.2.1 从 TUI 到活动 Runner 的取消链

当前 TUI 的中断最终进入 `SessionPrompt.cancel -> SessionRunState.cancel`。`SessionRunState` 会取消相关 Background Jobs，再取消当前 Session 的活动 Runner：

```ts
const cancel = Effect.fn("SessionRunState.cancel")(function* (sessionID) {
  yield* cancelBackgroundJobs(background, sessionID)
  const data = yield* InstanceState.get(state)
  const existing = data.runners.get(sessionID)

  if (!existing) {
    yield* status.set(sessionID, { type: "idle" })
    return
  }
  yield* existing.cancel
})
```

#### 7.2.2 Processor 如何尽力结算

Processor cleanup 会尽力保存已经累计的 Text、Reasoning、Patch 等状态，最多短暂等待当前 Tool settlement，再把仍未完成的 Tool Part 标记为中断错误：

```ts
state: {
  ...part.state,
  status: "error",
  error: "Tool execution aborted",
  metadata: { ...metadata, interrupted: true },
  time: { start, end },
}
```

#### 7.2.3 中断记录不等于副作用回滚

这里保存的是“中断发生了”这一事实，不是撤销事务。已经读完的文件不会变成没有读取过；已经启动的命令、写入或外部请求，也不保证被反向恢复。Interrupt 在这条路径中的目标是终止当前 Runner 的后续推进并尽力结算状态，而不是回滚所有副作用；外部命令或请求是否已经产生不可撤销结果，仍要按相应 Tool 和运行环境判断。

### 7.3 Doom Loop：只捕捉一种很窄的重复

#### 7.3.1 最近三个 Part 的触发条件

为了避免模型连续发出完全相同的工具调用，Processor 会查看当前 Assistant Message 最近三个 Part。下面把源码中的“不满足则提前返回”改写为等价的正向条件，以便直接看出触发规则：

```ts
const DOOM_LOOP_THRESHOLD = 3
const recentParts = parts.slice(-DOOM_LOOP_THRESHOLD)

if (
  recentParts.length === DOOM_LOOP_THRESHOLD &&
  recentParts.every(
    (part) =>
      part.type === "tool" &&
      part.tool === value.name &&
      part.state.status !== "pending" &&
      JSON.stringify(part.state.input) === JSON.stringify(input),
  )
) {
  yield* permission.ask({
    permission: "doom_loop",
    patterns: [value.name],
    metadata: { tool: value.name, input },
    ruleset: agent.permission,
  })
}
```

它要求同时满足：

- 最近三个 Part 都是 Tool Part；
- Tool 名相同；
- 状态都不是 pending；
- `JSON.stringify(input)` 后完全一致。

#### 7.3.2 为什么它不是无限循环证明器

默认处理是请求 `doom_loop` Permission，不是自动判定任务失败，也不是永久禁用该工具。参数稍有变化、对象序列化结果不同，或者 Text/Reasoning/并行 Tool 改变了最近三个 Part 的排列，都可能不命中。因此它是一条窄保护规则，不是“无限循环证明器”。

### 7.4 `agent.steps` 是提醒，不是当前路径的硬闸门

#### 7.4.1 `step` 计数发生在哪里

外层 Loop 每次通过顶部终止检查后递增 `step`：

```ts
step++
const maxSteps = agent.steps ?? Infinity
const isLastStep = step >= maxSteps
```

#### 7.4.2 达到阈值后只追加模型提示

达到阈值后，当前兼容路径只向模型消息追加 `MAX_STEPS_PROMPT`，提示模型停止使用工具并用文本收尾：

```ts
messages: [
  ...modelMsgs,
  ...(isLastStep
    ? [{ role: "assistant", content: MAX_STEPS_PROMPT }]
    : []),
],
tools,
```

注意 `tools` 仍然照常传入，代码也没有因为 `isLastStep` 直接 `break`，更没有把 `toolChoice` 强制改为 `none`。因此这里的 `steps` 是模型可见的强提示，不能表述为“最多 N 次 Provider 请求”的确定性上限。

#### 7.4.3 为什么 `step` 不等于 Provider Turn 数

此外，`step` 在 Subtask、Compaction 等特殊分支之前就会递增，所以它并不严格等于 Provider Turn 数；同一个兼容 `runLoop` 中，运行期间新增 User Message 也不会把局部变量 `step` 置零。真正结束仍依赖 terminal check、Processor Error、Permission、Compaction、Interrupt 和模型最终行为共同作用。

## 八、当前默认 Loop、native LLM adapter 与 native V2

OpenCode 源码中同时出现多个带有 `native` 的概念，容易让读者误以为默认 TUI 已完全切换到新 Session Runtime。

### 8.1 三条概念边界

| 名称 | 所处位置 | 是否改变本文的外层 Loop |
| --- | --- | --- |
| 默认 AI SDK runtime | 兼容 `SessionPrompt` 下的模型请求与流适配 | 否，本文主线 |
| 可选 native LLM adapter | 仍位于兼容 Session 编排下，替换部分请求 lowering / transport | 否，外层仍是 `SessionPrompt.run` |
| native V2 Session Runtime | 独立的 prompt admission、promotion、`SessionRunner` 与 Tool continuation | 是，属于另一套 Session 执行架构 |

“使用 native LLM adapter”不等于“进入 native V2 Session Runtime”。判断 Loop 所属架构时，应该看谁接收用户输入、谁拥有外层 continuation、谁重载历史，而不是只看模型请求适配器的名字。

### 8.2 native V2 对 Loop 心智模型的关键变化

固定源码中的 native V2 已经把若干边界做得更显式：

- Prompt 先作为 durable input admission 记录，再由执行协调器唤醒；
- Runner 在安全的 Provider Turn 边界 promotion 输入；
- 每个 Provider Turn 显式执行一次 `llm.stream(request)`；
- Tool Call、settlement 与 continuation 由 V2 SessionRunner 管理；
- V2 的 step allowance 到达阈值后会停止物化 Tools，并使用 text-only final turn，比兼容路径的提醒更具可执行性。

但不能由此推导它已经覆盖当前路径的全部保护。固定版本中，一般 Provider Retry、Doom Loop 等能力仍没有完全等价；post-crash 自动 continuation 也需要独立设计。完整 current/native V2 演进集中在[第 12 篇](./12_Runtime_Boundary.md)。

## 九、关键源码索引

以下索引用于把本文的架构图映射回固定源码。正文已经展示理解主流程所需的关键代码；这里不重复逐行解释。

| 主题 | 源码文件 | 关键符号 |
| --- | --- | --- |
| TUI 普通消息提交 | `packages/tui/src/component/prompt/index.tsx` | `submitInner` 中的 `sdk.client.session.prompt` |
| 兼容 Prompt HTTP Handler | `packages/opencode/src/server/routes/instance/httpapi/handlers/session.ts` | `SessionHttpApi.prompt`、`promptAsync` |
| User Message 与外层循环 | `packages/opencode/src/session/prompt.ts` | `SessionPrompt.prompt`、`run`、`loop` |
| 同 Session Runner 与取消 | `packages/opencode/src/session/run-state.ts` | `runner`、`ensureRunning`、`cancel` |
| 流事件与 Tool 结算 | `packages/opencode/src/session/processor.ts` | `SessionProcessor.create`、`handleEvent`、`process`、`cleanup` |
| Provider Retry | `packages/opencode/src/session/retry.ts` | `retryable`、`delay`、`policy` |
| 活跃历史与模型消息转换 | `packages/opencode/src/session/message-v2.ts` | `filterCompactedEffect`、`latest`、`toModelMessagesEffect` |
| Tool 物化 | `packages/opencode/src/session/tools.ts` | `SessionTools.resolve` |
| Context Overflow 与压缩 | `packages/opencode/src/session/compaction.ts` | `isOverflow`、`create`、`process`、`prune` |

代表性测试入口：

- `packages/opencode/test/session/prompt.test.ts`：Tool continuation、finish 与 Tool Part、同 Session 并发 Loop、Interrupt、运行中新 Prompt；
- `packages/opencode/test/session/processor-effect.test.ts`：流事件和 Tool Part 结算；
- `packages/opencode/test/session/retry.test.ts`：Retry 分类、等待、次数与状态；
- `packages/opencode/test/agent/agent.test.ts`：Agent steps 配置和默认 `doom_loop` Permission。

完整跨章节证据表见[源码与证据索引](./appendices/Source_Index.md)。

## 十、总结：一次请求怎样运行到空闲

现在可以把 OpenCode 当前默认 Agent Loop 收束为一条完整链路：

```text
User Message 先写入 Session
        │
        ▼
SessionRunState 为当前 Session 取得唯一活动 Runner
        │
        ▼
SessionPrompt.run 重载活跃历史
        │
        ├── 已完成且无待反馈 Tool -> 结束
        ├── Subtask / Compaction / Overflow -> 先处理特殊分支
        `── 普通路径 -> 创建 Assistant Message
                           │
                           ▼
                    组装 Context 与 Tools
                           │
                           ▼
                    发起 Provider Turn
                           │
                           ▼
                    Processor 结算流事件
                           │
             ┌─────────────┼─────────────┐
             ▼             ▼             ▼
          continue       compact         stop
             │             │             │
             └─────── 回到外层或结束 ─────┘
                           │
                           ▼
                    Runner 到达 idle
```

模型负责根据当前 Context 判断“下一步做什么”；Harness 负责让每一步进入可验证、可授权、可执行、可记录的边界。Tool Result 先在当前处理阶段结算为观察，下一次外层迭代再重载历史并询问模型。Retry、Interrupt、Compaction、Doom Loop 和 `agent.steps` 分别作用在不同控制位置，不能都简化成“再循环一次”或“立即停止”。

理解这条反馈链后，下一篇才能继续回答更细的问题：每次 Provider Turn 开始前，Harness 究竟怎样选择 System、Messages 与 Tool definitions，组成模型这一轮真正看见的 Context？

# Tools 与 Permission：模型的意图怎样变成真实操作

上一篇：[08 Context Architecture](./08_Context_Architecture.md) ｜ 下一篇：[10 Session 与 Persistence](./10_Session_and_Persistence.md)

> 源码基线：`0e3474509aa5ad16afcf9c439785514d6443c6af`

假设你刚开始学习 Harness，并向 OpenCode 提出：

> 请先读取这个教程的 README 和项目规则，再告诉我应该按什么顺序学习。

模型随后生成了一个 `read` 调用。此时文件已经被读取了吗？如果路径写错，谁会阻止执行？如果需要批准，谁会暂停等待？读取结果又怎样回到模型面前？

直观答案是：模型没有直接操作文件。它只生成了一个结构化的行动请求；Harness 随后负责检查参数和权限，调用真正的执行器，再把结果保存并放回下一轮上下文。整条链的中心区别是：**Tool Call 表达“希望执行什么”，不是“已经发生了什么”。**

## 一、一次 `read` 是怎样从意图走到结果的

先把一次工具调用放回完整反馈循环：模型根据 Context 选择行动，Harness 把行动变成受控操作，工具执行结果成为下一轮 Observation。

```text
候选 Tool 注册
    ↓
本轮物化 Tool definition
    ↓
模型生成 Tool Call
    ↓
参数验证
    ↓
Permission：allow / ask / deny
    ↓
Tool Executor 真实执行
    ↓
Tool Settlement
    ↓
保存终态，并在下一轮形成 Observation
```

这不是七个并列模块，而是同一次调用的七个阶段。前一阶段的输出是后一阶段的输入；特别是参数验证或 Permission 没有通过时，后面的真实操作就不应开始。

本篇会一直使用同一个低风险任务观察它：用户希望从零学习 Harness，OpenCode 先读取 `Opencode_Harness/README.md`，再根据其中的导航决定是否读取项目规则，最后推荐学习顺序。选择 `read` 是因为它不主动修改教程文件，但“只读”并不等于“不需要控制”——项目外路径同样可能包含敏感信息。

这条流程中混合了两种性质不同的工作。模型对“现在是否需要读文件、先读哪个文件”做概率性判断；schema 校验、规则求值、批准等待、executor 调用和结果落库则由 Harness 按程序规则推进。前者让 Agent 能适应开放任务，后者让一次具体操作具备可检查边界。

为了看懂这条链，先区分五个名称：

- **Tool** 是 OpenCode 向模型提供的具名能力，本地 Tool 还关联实际执行器。
- **Tool definition** 是本轮发送给模型的名称、说明和输入 schema，不包含执行器代码。
- **Tool Call** 是模型生成的工具名、调用 ID 和参数。
- **Tool Result** 是执行器或 Provider 产生的领域结果。
- **Tool Settlement** 是 Harness 把调用确定为成功、可见失败或中断，并形成持久化终态与模型输出的过程。

执行器返回的原始结果、Session 保存的结果和下一轮模型看到的工具输出可能是三种不同表示。模型输出可能经过格式转换、截断或错误归一化，不能把这几个名称混为一谈。

### 1.1 注册：系统先建立候选能力

#### 1.1.1 `ReadTool` 同时声明机器合同与执行入口

当前默认路径中，`ReadTool` 通过 `Tool.define("read", ...)` 声明工具名称、输入 schema 和执行函数。`ToolRegistry` 初始化时把它加入候选集合。

固定源码中的定义可以压缩为：

```ts
export const Parameters = Schema.Struct({
  filePath: Schema.String,
  offset: Schema.optional(NonNegativeInt),
  limit: Schema.optional(NonNegativeInt),
})

export const ReadTool = Tool.define(
  "read",
  Effect.gen(function* () {
    // 初始化依赖，最后返回 description、parameters 与 execute
  }),
)
```

`Tool.define` 保存的不是一段给模型阅读的 Prompt，而是一份宿主侧定义：它把公开说明、参数 decoder 和真实 executor 关联在同一个 Tool identity 下。之后 Registry 可以把其中的说明与 schema 发给模型，同时把 executor 留在 OpenCode 运行时。

`read` 的输入可以简化为：

```text
filePath：要读取的路径，必需
offset：从哪一行开始，可选
limit：最多读取多少行，可选
```

注册时没有模型请求，也没有文件读取。把注册与执行分开后，系统可以安装很多能力，却只在合适的 Agent、Model 和会话中公开其中一部分。

#### 1.1.2 候选来源不同，信任边界也不同

Built-in、Custom Tool、Plugin 和 MCP 的差别首先体现在候选能力从哪里来：Built-in 由 OpenCode 静态注册；Custom Tool 从配置目录发现并加载；Plugin 可以提供 Tool 和 Hook；MCP Tool 来自 Server 的 `tools/list`。其中 MCP Tool 不会先塞进旧 `ToolRegistry`，而是由 `SessionTools.resolve` 在 Registry Tool 之外取得并转换；两条来源在本轮 Tool map 中汇合。Skill 则由内置 `skill` Tool 加载说明，说明本身不会因为包含命令就自动执行；`task` Tool 会创建或恢复子 Session，它属于编排能力，不是普通叶子函数，详见 [第 11 篇](./11_Agent_Specialization_and_Collaboration.md)。安装和配置这些扩展的方法参见 [05 Enhancement](./05_Enhancement.md)。

这些入口最终都可能产生可调用能力，但信任边界不同。Custom/Plugin 代码运行在 OpenCode 进程中，MCP 的执行者可能是本地或远程 Server；“都能以 Tool 形式出现在模型面前”不等于“都由同一个受控执行器完成”。

把候选来源放在注册阶段下比较，会更容易看清这种差异：

| 候选能力 | 怎样进入体系 | 实际执行者 | 此时要审查什么 |
| --- | --- | --- | --- |
| Built-in Tool | OpenCode Registry 静态注册 | OpenCode 内部 executor | leaf 是否在副作用前正确请求 Permission |
| Custom Tool | 从配置目录发现并动态加载 | 自定义模块代码 | Host 不会自动替它补一次 `ctx.ask` |
| Plugin Tool | Plugin 加载后提供 Tool/Hook | Plugin 进程内代码 | Plugin 加载本身已获得进程能力 |
| MCP Tool | 连接 Server，取得并缓存 `tools/list` | 本地或远程 MCP Server | schema、说明、结果和远端权限都属于外部边界 |

这张表比较的是“能力从哪里进入”，不是后续执行顺序。无论来源如何，只要要交给模型选择，仍需在本轮被物化成 Tool definition。

### 1.2 物化：决定模型本轮看见什么

#### 1.2.1 `SessionTools.resolve` 把定义包装成本轮 Tool map

外层 Loop 为新的 Assistant/Processor 上下文组装首次 Provider Request 时，`SessionTools.resolve` 重新取得候选 Tool，并形成这一轮的 Tool map。一次 Retry attempt 复用同一份 `streamInput`，不会先返回这里重新物化。

```text
Registry 候选
-> 生成 JSON Schema
-> 做 Provider 兼容转换
-> 应用 Agent / Session / 本轮覆盖规则
-> 排序并形成 Tool map
-> 放入 Provider Request
```

核心包装关系如下：

```ts
for (const item of yield* registry.tools({ modelID, providerID, agent, permission })) {
  const schema = ProviderTransform.schema(input.model, ToolJsonSchema.fromTool(item))
  tools[item.id] = tool({
    description: item.description,
    inputSchema: jsonSchema(schema),
    execute(args, options) {
      return run.promise(/* Hook -> item.execute -> output */)
    },
  })
}
```

这里的“物化”不是重新安装 Tool，而是把候选定义与当前 Session、Assistant Message、Agent、Model、Abort Signal 和 Permission context 绑定，形成这一轮可执行的 Tool map。

#### 1.2.2 可见性过滤发生在最终请求边界

最终请求还会根据 Agent permission、Session permission 与本次 User Message 的 tool override 过滤：

```ts
function resolveTools(input) {
  const disabled = Permission.disabled(
    Object.keys(input.tools),
    Permission.merge(input.agent.permission, input.permission ?? []),
  )
  return Record.filter(
    input.tools,
    (_, name) => input.user.tools?.[name] !== false && !disabled.has(name),
  )
}
```

只有 whole-tool 的 `pattern: "*"` deny 才会直接隐藏对应 Tool；针对某个路径、命令或 URL 的资源级规则，通常要等 executor 调用 `ctx.ask` 时才求值。把这两种过滤混在一起，会误以为“模型看得见”就等于“具体资源已获准”。

#### 1.2.3 模型看到合同，看不到实现能力

Provider 通常只接收 `name`、`description` 和 `input schema`。模型看不到 Tool Executor 的 TypeScript/Effect 代码、`ctx.ask` 的实现、OpenCode 的文件系统对象，也不知道进程在操作系统中实际拥有多大权限。

因此，“Registry 里存在 `read`”和“模型本轮看见 `read`”是两个结论。前者描述候选能力，后者描述 Context。Agent 或本轮覆盖规则可以让整个 Tool 不进入请求；但只要 definition 被公开，模型就可能据此生成调用。

definition 的文字也会影响模型行为。名称告诉模型调用哪个入口，description 解释何时使用，schema 则限制参数结构；三者共同属于 Provider Request 的 Context。executor 却必须留在 Harness 侧，因为把实现代码写进 prompt 既不能赋予模型本地能力，也不能替代真实的参数检查与权限控制。

### 1.3 Tool Call：结构化意图还不是行动事实

模型可能输出下面这种说明性调用：

```json
{
  "tool": "read",
  "arguments": {
    "filePath": "Opencode_Harness/README.md",
    "offset": 1,
    "limit": 200
  }
}
```

不同 Provider 的线上协议可能不同，OpenCode 会把流式响应归一化成内部事件。到这里为止，教程文件仍未被 `read` executor 读取。

这是模型与 Harness 的责任分界：模型用概率性判断决定是否调用、调用哪个 Tool、先读取哪个文件；Harness 不把这份输出当成可信指令，而是把它当作等待验证的数据。

### 1.4 验证与 Hook：执行前先收窄输入

#### 1.4.1 参数验证至少跨过两层合同

普通 Built-in Tool 的输入会经过不止一层检查：AI SDK 根据发送给模型的 input schema 校验调用参数，`Tool.wrap` 再使用原始 Effect Schema 做 typed decode。缺少 `filePath`，或把 `offset` 写成非法值时，执行器不会因为参数来自模型就直接接受。

`Tool.wrap` 中的 typed decode 位于 leaf executor 之前：

```ts
const decode = Schema.decodeUnknownEffect(toolInfo.parameters)
const execute = toolInfo.execute

toolInfo.execute = (args, ctx) =>
  Effect.gen(function* () {
    const decoded = yield* decode(args).pipe(
      Effect.mapError((error) => new InvalidArgumentsError({ tool: id, detail: String(error) })),
    )
    return yield* execute(decoded, ctx)
  })
```

schema 只回答“参数的结构和类型是否合法”，不回答“这个合法路径是否允许读取”。即使 `filePath` 是完全有效的字符串，它仍可能指向项目外的敏感文件；资源授权必须在下一阶段处理。

验证失败时，真实读取不会开始，调用会沿错误路径被结算。反过来，通过验证也只说明数据满足机器合同：schema 无法判断教程路径是否符合用户意图，无法判断文件内容是否敏感，也无法判断远程 MCP Server 是否值得信任。这正是为什么验证后还需要 Permission，而 Permission 之外还需要操作系统与部署边界。

#### 1.4.2 Hook 位于验证、授权和输出处理之间

Plugin Hook 也位于这条执行链中。当前默认普通 Registry Tool 的主要顺序可以简化为：

```text
AI SDK input validation
-> tool.execute.before
-> Built-in typed decode
-> leaf executor 内调用 ctx.ask
-> 真实执行
-> 通用输出处理
-> tool.execute.after
-> 返回 LLM Runtime
```

`before` hook 可以检查或修改参数，所以受管 Built-in 应对最终实际使用的路径、命令或 URL 请求 Permission，而不能只批准修改前的值。`after` hook 又可以修改结果；固定源码中，它位于普通 Tool 的通用文本截断之后，之后没有再做同一层通用截断。因此 Hook 是需要信任的进程内扩展，而不是天然受 Tool wrapper 完整约束的脚本。

MCP Tool 的后半段顺序不同：Host 先运行 `before`，再请求 namespaced MCP Permission、调用 Server，然后让 `after` 处理 Server 返回的原始结果，最后才由 Host 转换文本/附件并做 MCP 输出限制。因此“after 位于通用截断之后”的缺口只适用于普通 Registry Tool，不能不加区分地套到 MCP adapter。

### 1.5 Permission：在副作用前决定是否继续

#### 1.5.1 规则求值采用最后匹配，缺省为 `ask`

当 `read` executor 准备访问具体路径时，它会通过 `ctx.ask` 请求资源级授权。规则求值可以理解为：

```text
permission + pattern -> allow | ask | deny
```

- `allow`：无需交互，继续执行。
- `ask`：发布批准请求并暂停，用户回复前不读取。
- `deny`：立即拒绝，文件不会被读取。

没有匹配规则时，当前规则求值默认进入 `ask`。多条规则命中时，最后一条匹配规则优先；Session rules 位于 Agent rules 之后，因此可能覆盖前面的匹配结果。

固定源码直接使用 `findLast`：

```ts
export function evaluate(permission, pattern, ...rulesets) {
  return rulesets
    .flat()
    .findLast(
      (rule) => Wildcard.match(permission, rule.permission) &&
                Wildcard.match(pattern, rule.pattern),
    ) ?? { action: "ask", permission, pattern: "*" }
}
```

因此规则顺序本身具有语义。后出现的、更具体或来自 Session 的匹配规则可以覆盖前面的结果，但只有真正同时匹配 permission 与 pattern 的规则才参与决策。

#### 1.5.2 `ask` 会建立等待点，而不是先执行后通知

`Permission.ask` 会逐个检查本次请求的 patterns：任一 pattern 命中 deny 就立即失败；全部 allow 就直接返回；至少一个 ask 才创建请求并等待：

```ts
const deferred = yield* Deferred.make()
pending.set(id, { info, deferred })
yield* events.publish(Event.Asked, info)
return yield* Effect.ensuring(
  Deferred.await(deferred),
  Effect.sync(() => pending.delete(id)),
)
```

executor 的 Effect 停在 `Deferred.await`，所以批准发生在后续文件读取、命令启动或远程调用之前。这是 Permission 能够约束受管 Tool 的关键前提。

#### 1.5.3 `once`、`always` 与 `reject` 的生命周期不同

用户面对批准请求时，`once` 只允许当前请求；`always` 把相应 pattern 加入当前 OpenCode Instance 的内存批准集合；`reject` 拒绝请求并结束相关等待。这里的 `always` 不是“永久写入配置”：固定版本当前默认兼容路径中的 pending request 和 approval 都是进程内状态，不能据此推断它们会跨进程重启保存。

内存中的 `always` approval 可能在同一 Instance 的其他 Session 中匹配，这比“只对当前对话有效”更宽；但进程退出后并不保证继续存在。批准时真正需要阅读的是具体 permission、pattern 和将要访问的资源，而不是只看模型在文本中如何描述自己的计划。

### 1.6 Executor：此时才接触真实文件系统

#### 1.6.1 `read` 先解析最终资源，再依次经过两个授权面

Permission 允许后，`read` executor 才会：

1. 按当前项目目录解析相对路径；
2. 对项目外路径处理 `external_directory` 授权；
3. 对最终资源处理 `read` 授权；
4. 读取目录、文本、图片或 PDF；
5. 对文本分页，对媒体生成附件表示；
6. 返回标题、文本、metadata 和可选 attachments。

源码中的顺序尤其值得注意：

```ts
if (!path.isAbsolute(filepath)) {
  filepath = path.resolve(instance.directory, filepath)
}

yield* assertExternalDirectoryEffect(ctx, filepath, { kind })
yield* ctx.ask({
  permission: "read",
  patterns: [path.relative(instance.worktree, filepath)],
  always: ["*"],
})

// 授权通过后才进入目录、媒体、文本等读取分支
```

`external_directory` 回答“是否允许越出工作树”，`read` 回答“是否允许读取这个最终资源”。二者都通过之后，真实内容访问才继续。

#### 1.6.2 目录、文本和媒体走不同结果分支

文本默认按 2000 行和 50 KiB 形成一页。超出范围不是错误，也不意味着 OpenCode 已经把整个文件全部发给模型；模型可以在下一轮使用新的 `offset` 请求后续内容。这个限制既控制工具工作量，也避免单次结果占满模型上下文。

目录返回经过 offset/limit 切片的 entries；文本逐行读取并限制单行长度、总字节和页大小；受支持的图片与 PDF 作为 attachments 返回；其他二进制文件会失败。`read` 还可能通过 `Instruction.resolve` 把附近规则附加为 `<system-reminder>`，因此一次读取结果可以同时包含文件正文与该路径适用的规则提醒。

这里的 50 KiB 是 `read` producer 自己的分页边界。`read` 返回的 metadata 已经说明是否截断，外层普通 Tool wrapper 不会再把它误当成一个尚未处理的无限文本；要继续读取的来源仍是原文件和新的 `offset`，不是某个自动保存了全文的模型附件。

#### 1.6.3 并非所有 Tool 都由本地 leaf executor 执行

不同能力在这一阶段由不同执行者完成。Built-in Tool 调用 OpenCode 内部 executor；MCP Tool 由 Host 对 namespaced Tool 请求 Permission 后，把调用交给本地或远程 MCP Server；Provider 托管的 `providerExecuted` Tool 则由 Provider 产生结果，OpenCode 不会再寻找同名本地 executor。后两类仍需要结果结算，但不能机械套用“本地 leaf executor 在函数内部请求资源 Permission”的流程。

### 1.7 Settlement：把执行结果变成下一轮 Observation

#### 1.7.1 Tool Part 是一条显式状态机

executor 返回并不是生命周期终点。`SessionProcessor` 还要把 Tool Part 从 `pending`、`running` 结算为 `completed` 或 `error`：

```text
tool input          -> pending
tool call           -> running
tool result success -> completed
tool result failure -> error
```

成功结算会把输入沿用到终态，并补上 output、title、metadata、时间和附件：

```ts
yield* session.updatePart({
  ...match.part,
  state: {
    status: "completed",
    input: match.part.state.input,
    output: output.output,
    metadata: output.metadata,
    title: output.title,
    time: { start: match.part.state.time.start, end: Date.now() },
    attachments: output.attachments,
  },
})
```

成功终态会保存输入、模型可见输出、标题、时间、metadata 和可选附件；失败终态会保存输入、错误及相关 metadata。参数错误、文件不存在、MCP 错误或用户拒绝因此都可以成为后续模型可观察的终态，而不是留下一个永远等待的调用。

#### 1.7.2 Raw Result、durable terminal state 与模型输出是三层表示

工具输出并非越完整越好。`read` 自己分页，`bash` 保留受限预览并可能把更完整的结果写入受管理文件，普通 Tool wrapper 会截断文本，MCP adapter 也会转换与限制结果；更早的输出还可能在 Compaction/Pruning 后不再对模型可见。文本大小限制也不能代替附件的 MIME 与大小检查。

因此需要分别问三个问题：executor 实际获得了什么，Session 终态保存了什么，下一 Provider Turn 又投影了什么。假设 `read` 遇到一个超长 README，执行器可能只按当前 `offset/limit` 返回一页；wrapper 还可能处理文本边界；未来 Compaction 又可能让旧输出从 active history 中消失。不能因为最初文件很完整，就推断模型始终拥有其全部内容。

普通 Registry Tool 的通用文本截断发生在 Tool wrapper 内，而 `tool.execute.after` 位于 wrapper 返回之后。after hook 可以修改已经截断的 output，固定路径没有再经过同一层通用截断；因此 Plugin Hook 是受信的进程内扩展，不能把通用 truncation 当作对所有扩展输出的最终安全闸门。Attachments 也有独立 MIME、大小和 Provider 兼容边界，不应只用文本字符数衡量。

#### 1.7.3 下一轮把终态投影成 Observation

结算完成后，外层 Agent Loop 重新读取会话历史，把 completed/error Tool Part 投影成 Provider 可接受的工具输出：

```text
Provider Turn 1：模型请求读取 README
-> Harness 执行并保存 Tool terminal state

Provider Turn 2：模型看到 README 的 Model Tool Output
-> 决定继续读取项目规则，或给出学习顺序
```

模型不会在调用之间自动记住 README。Observe 能够成立，是因为 Harness 保存结果并在下一轮重新注入。Tool Part 如何持久化由 [第 10 篇](./10_Session_and_Persistence.md)继续说明。

## 二、Permission 控制调用，但不是系统的最终安全边界

### 2.1 Tool 可见性、资源授权与 OS 隔离是三个控制面

理解 `read` 生命周期后，可以把三个经常混淆的控制面放到正确位置：

```text
Tool 可见性：模型本轮能否提出这种行动？
        ↓
Permission：这次具体资源访问是否继续？
        ↓
OS Sandbox：即使代码尝试越界，系统最多允许它做什么？
```

Tool 可见性和 Permission 是两层不同控制。把 `bash` 从 Tool map 中移除，模型就不能通过这份 definition 发起普通 `bash` Tool Call；但模型能看见 `read`，不表示任意路径已被允许，具体资源仍要由 executor 请求 Permission。

三层控制解决的问题不同：可见性缩小模型能够提出的受管能力集合；Permission 对某次 action/resource 求值并在必要时等待用户；OS Sandbox 即使面对有缺陷或不受管的代码，也从系统层限制进程实际能触及的文件、网络和子进程。

### 2.2 生命周期定位比反复修改 Prompt 更可靠

遇到“Tool 没有运行”时，可以沿同一生命周期定位，而不是只修改 prompt：

| 观察位置 | 要问的问题 | 未通过时的典型表现 |
| --- | --- | --- |
| Registry | OpenCode 是否发现候选 Tool | 候选集中不存在 |
| Materialization | 本轮是否公开 definition | Provider Request 中没有该 Tool |
| Model decision | 模型是否选择调用 | 只输出文本或调用别的 Tool |
| Validation | 参数是否满足 schema | executor 不启动，调用进入错误路径 |
| Permission | 具体资源是否允许 | 等待批准或直接 deny |
| Execution | 外部操作是否成功 | 形成 error Tool state |
| Settlement | 结果是否保存并投影 | 下一轮缺少可用 Observation |

这些问题有严格先后关系。例如 Permission 规则无法解释“definition 根本没有发给模型”，反复要求模型重试也无法修复一个不合法的参数 schema。

### 2.3 Permission 不会自动创建 OS Sandbox

Permission 又只是 OpenCode 应用层的策略门。操作系统沙箱（OS Sandbox）通过低权限账户、容器、虚拟机、文件系统挂载、系统调用或网络隔离限制进程的真实能力。二者不能互换：

**当前默认 OpenCode 的 Tool Permission 不会自动创建这样的 OS Sandbox。** 是否存在隔离环境，取决于 OpenCode 实际运行在哪种账户、容器、虚拟机或受限系统中。

- `bash` 获准后，会以 OpenCode 进程用户的 host 权限启动真实子进程。
- `read` 的边界来自路径解析和 Permission gate，不是受限文件系统挂载。
- Custom/Plugin Tool 可以选择不调用 `ctx.ask`；Plugin 模块及 factory 在模型调用前就可能执行代码。
- 本地 MCP Server 在连接阶段已经启动；Tool Permission 主要约束后续受管的 `tools/call`。
- 远程 MCP Server 的账号权限、网络访问和数据保留不由 OpenCode Permission 控制。

所以，Permission 对话框不能代替最小权限凭据和隔离环境。实验未知 Plugin、Shell 命令或外部服务时，应把它们当作真实代码与真实服务审查，而不是因为入口叫 Tool 就默认安全。

### 2.4 失败、拒绝与取消都要保存成可解释状态

失败与取消也要放在这个边界上理解。Permission reject 是 Harness 阻止副作用，不是模型“改变了想法”。用户取消时，OpenCode 会中断 Runner/Provider stream，并尽量把仍在 pending/running 的 Tool 标记为 interrupted error；但中断记录不会撤销已发生的外部操作。一个命令如果已经写入文件，随后才被取消，Tool Part 变为 error 也不会自动还原该文件。

参数 decode 失败、文件不存在、读取失败或 MCP 返回错误时，Processor 会尽量形成可解释的 error Tool state，让后续模型能够观察失败并换一种行动。Permission reject 在默认配置下还会阻止当前 Loop 盲目重复同一受拒行动，兼容配置可以改变部分继续行为。无论错误来自哪一层，都不应把历史中的失败调用视作一个等待重放的副作用。

`providerExecuted` 是需要就地说明的例外：它表示结果由 Provider 托管执行产生，OpenCode Processor 仍记录 Tool Part，却不会再运行同名本地 executor。Structured Output Tool 则是按单次请求直接加入 Tool map 的控制工具，用来捕获符合 JSON Schema 的最终对象；它不代表普通 Registry Tool 的注册、Hook 和 Permission 流程。

### 2.5 Native V2 只保留与本篇直接相关的边界

固定版本的 native V2 对 typed Tool、按 Location 物化、调用 identity、durable Tool events、Permission approval 和 Tool output store 做了更集中建模，但普通 TUI 尚未切换到这条完整路径，Custom/Plugin/MCP 等 parity 也未全部覆盖。本篇描述的仍是当前默认兼容 Runtime。无论哪条路径，受信 executor 主动请求授权、Permission 不等于 OS Sandbox 这两个原则都没有改变。

## 三、关键源码索引

正文已经展示理解主流程所需的控制代码。继续阅读固定源码时，可以按生命周期定位：

| 主题 | 源码文件 | 关键符号 |
| --- | --- | --- |
| Built-in、Custom 与 Plugin 注册 | `packages/opencode/src/tool/registry.ts` | `ToolRegistry.layer`、`all`、`tools` |
| Tool 定义与 typed decode | `packages/opencode/src/tool/tool.ts` | `Tool.define`、`wrap`、`init` |
| 本轮物化、Hook 与执行 Context | `packages/opencode/src/session/tools.ts` | `SessionTools.resolve`、`context` |
| 最终可见性过滤 | `packages/opencode/src/session/llm/request.ts` | `LLMRequestPrep.resolveTools` |
| Permission 规则与等待 | `packages/opencode/src/permission/index.ts` | `evaluate`、`ask`、`reply`、`disabled` |
| `read` 的真实 executor | `packages/opencode/src/tool/read.ts` | `ReadTool.execute`、`lines` |
| Tool Part 状态结算 | `packages/opencode/src/session/processor.ts` | `ensureToolCall`、`handleEvent`、`completeToolCall` |
| 下一轮模型输出转换 | `packages/opencode/src/session/message-v2.ts` | `toModelMessagesEffect` |

完整跨章节证据与代表性测试见[源码与证据索引](./appendices/Source_Index.md)。

## 四、总结：Harness 把概率性意图包进确定性边界

一次 Tool 调用的关键不在于模型会不会生成 `read`，而在于 Harness 不把生成结果直接等同于操作事实。注册与物化决定模型可选择哪些能力，验证与 Permission 决定具体调用是否合规，executor 才接触真实环境，Settlement 再把结果变成可保存、可观察的终态。

只需牢牢记住这条因果链：

```text
register -> materialize -> call -> validate -> permission
-> execute -> settle -> persist -> next-turn observation
```

正文中的关键实现可从 [Source Index 的 Tools 与 Permission](./appendices/Source_Index.md#6-tools-与-permission)继续核对，重点入口是 `tool/registry.ts`、`session/tools.ts`、`permission/`、各 Built-in executor、`session/processor.ts` 与 `session/message-v2.ts`。下一篇将沿着已经结算的 Tool Part 继续追问：OpenCode 保存了哪些状态，为什么下一轮或重新打开 Session 后还能找回它们？

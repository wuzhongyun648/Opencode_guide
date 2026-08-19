# Tools 与 Permission：模型的意图怎样变成真实操作

上一篇：[08 Context Architecture](./08_Context_Architecture.md)
下一篇：[10 Session 与 Persistence](./10_Session_and_Persistence.md)

> 源码基线：`0e3474509aa5ad16afcf9c439785514d6443c6af`

## 1. 学习问题

假设你刚开始学习 Harness，并向 OpenCode 提出：

> 请先读取这个教程的 README 和项目规则，再告诉我应该按什么顺序学习。

模型随后生成了一个 `read` 调用。此时文件已经被读取了吗？谁检查路径和参数？如果读取需要批准，谁会暂停执行？结果又怎样回到下一轮模型输入？

本篇只回答一个问题：

> **模型提出的行动意图，怎样经过 Harness 变成一次真实、可控、可记录的操作？**

## 2. 最短答案

模型不会直接读取文件。它只能根据本轮看到的工具名称、说明和参数结构，生成一个工具调用（Tool Call）。

OpenCode 随后负责找到对应实现、验证参数、执行权限判断，并在允许后调用真正的工具执行器（Tool Executor）。执行结果还要经过工具结算（Tool Settlement），成为可保存的终态和下一轮模型可见的工具输出。

因此，`Tool Call` 的准确含义是“请求执行”，不是“已经执行”。

## 3. 最小心智模型

阅读下面这条链时，重点观察三个边界：模型只提出意图，Harness 执行确定性检查，工具才接触真实环境。

```text
候选 Tool 注册
    ↓
本轮物化 Tool definition
    ↓
模型看到 name / description / input schema
    ↓
模型生成 Tool Call
    ↓
参数验证
    ↓
Permission：allow / ask / deny
    ↓
Tool Executor 真实执行
    ↓
domain/raw Tool Result
    ↓
Tool Settlement
    ↓
durable terminal state + Model Tool Output
    ↓
下一轮模型观察结果
```

这条链同时包含概率性和确定性两类行为：

- 模型是否选择 `read`、先读哪个文件，是概率性判断。
- schema 校验、规则求值、用户批准、执行器调用和结果保存，是 Harness 的控制流程。

## 4. 先分清五个容易混淆的概念

| 概念 | 本篇中的含义 | 不等于 |
| --- | --- | --- |
| Tool | 向模型声明的具名能力；本地 Tool 还关联执行器 | Tool 已经可见或已经执行 |
| Tool definition | 本轮发给模型的名称、说明和输入 schema | JavaScript/Effect 执行器本身 |
| Tool Call | 模型生成的调用名称、ID 和参数 | Tool Result |
| Tool Result | 执行器或 Provider 产生的原始领域结果 | 已完成持久化的终态 |
| Tool Settlement | Harness 把一次调用确定为成功、可见失败或中断，并形成模型输出和持久化状态的过程 | 只调用了一次 `execute()` |

为了强调结果经过了投影，本篇还使用“模型工具输出（Model Tool Output）”表示最终回放给模型的内容。它可能经过格式转换、截断或错误归一化，不一定等于执行器最初返回的完整对象。

## 5. 贯穿场景：先读取 Harness 学习入口

下面使用一个低风险场景贯穿本篇。路径仅作示例：

```text
用户目标：从零学习 OpenCode Harness
第一步：读取示例项目中的 Opencode_Harness/README.md
第二步：读取项目规则，确认学习范围
第三步：根据读取结果推荐章节顺序
```

这个场景适合观察工具生命周期，因为 `read` 主要取得信息，不主动修改教程文件。即便如此，Harness 仍然不能跳过参数和 Permission 检查：读取项目之外的路径也可能暴露敏感信息。

### 5.1 注册：OpenCode 先知道有哪些候选能力

当前默认路径中，`ReadTool` 通过 `Tool.define("read", ...)` 声明：

- 工具名称和说明；
- 输入参数 schema；
- 真正执行读取的函数。

`ToolRegistry` 初始化时把它加入 Built-in 候选集合。此时发生的只是“OpenCode 具备这种能力”，还没有形成模型请求，更没有读取文件。

`read` 的主要输入可以简化为：

```text
filePath：要读取的路径，必需
offset：从哪一行开始，可选
limit：最多读取多少行，可选
```

注册和执行分开很重要。一个工具可以存在于 Registry 中，却因为当前 Agent、Model 或配置而不出现在本轮请求里。

### 5.2 物化：本轮究竟向模型公开哪些 Tool

外层 Loop 为新的 Assistant/Processor 上下文组装首次 Provider Request 时，`SessionTools.resolve` 会重新取得候选 Tool，并把它们转换成本轮可调用的 Tool map。Retry attempt 复用同一份 `streamInput`，不会先回到这里重新物化。

```text
Registry Tool
-> 生成 JSON Schema
-> 做 Provider 兼容转换
-> 应用 Agent / Session / 本轮覆盖规则
-> 排序并形成 Tool map
-> 放入 Provider Request
```

这里需要区分两种控制：

1. **可见性控制**：决定模型本轮能不能看到整个 Tool。
2. **资源级授权**：模型给出具体路径或命令后，决定这次调用能不能执行。

例如，whole-tool deny 可以让本轮模型看不到 `bash`；但模型能看到 `read`，不表示任意路径都已被允许。具体路径仍要等执行器调用 Permission。

这也解释了为什么工具 definition 属于 Context Architecture 的一部分：名称、说明和 schema 会影响模型如何选择行动。但 Tool 的真实执行和授权属于本篇。

### 5.3 模型看到 schema，看不到 executor

Provider 通常只接收：

```text
name: read
description: 这个工具适合做什么
input schema: filePath / offset / limit 的结构与约束
```

模型看不到：

- OpenCode 的文件系统对象；
- `ctx.ask` 的实现；
- Permission 的待处理请求；
- Tool Executor 的 TypeScript/Effect 代码；
- OpenCode 进程在操作系统中的真实权限。

所以模型能够生成“读取 `Opencode_Harness/README.md`”的结构化意图，却不能从 Provider 进程直接调用本机 `fs.readFile`。

### 5.4 Tool Call：这是行动请求，不是行动事实

模型可能生成如下说明性调用：

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

不同 Provider 的线上协议格式可能不同。OpenCode 会把流式响应归一化成包含调用 ID、工具名称和输入参数的内部事件。

到这个节点为止，教程文件仍然没有被 `read` executor 读取。后面至少还有参数验证和授权。

### 5.5 验证：参数先满足机器可检查的合同

当前普通 Built-in Tool 的输入会经过不止一层检查：

1. AI SDK 根据发给模型的 input schema 检查调用参数。
2. `Tool.wrap` 使用原始 Effect Schema 再做 typed decode。

如果 `offset` 是非法值，或缺少必需的 `filePath`，执行器不会因为“这是模型生成的参数”就直接接受。验证失败会进入错误路径，真实读取不会开始。

schema 解决的是“参数形状是否有效”，不是“用户是否允许访问这个资源”。后一项由 Permission 负责。

### 5.6 Hook 与执行顺序

对当前默认路径中的普通 Registry Tool，主顺序可以简化为：

```text
AI SDK input validation
-> tool.execute.before
-> Built-in typed decode
-> leaf executor 调用 ctx.ask
-> 真实执行
-> 通用输出处理
-> tool.execute.after
-> 返回 LLM Runtime
```

`before` hook 可以在 Tool 执行前检查或修改参数。因此，受管 Built-in 应当对最终实际使用的路径、命令或 URL 求 Permission，而不能只授权模型最初生成但后来已改变的值。

`after` hook 可以修改执行结果。固定源码中，普通 Registry Tool 的 `after` 位于通用文本截断之后，之后没有再做同一层通用截断。这是一个需要信任 Plugin 代码的实现边界：不能假设 Hook 修改后的输出仍一定满足之前的截断 metadata。

### 5.7 Permission：在副作用之前做确定性决策

当前默认 Permission 规则可以理解为：

```text
permission + pattern -> allow | ask | deny
```

执行器把具体 action/resource 交给 `ctx.ask` 后，主要有三种结果：

| 结果 | Harness 的行为 | 文件是否读取 |
| --- | --- | --- |
| `allow` | 直接继续 | 随后读取 |
| `ask` | 发布批准请求并等待用户 | 批准前不读取 |
| `deny` | 立即拒绝调用 | 不读取 |

没有匹配规则时，当前规则求值默认进入 `ask`。如果多条规则匹配，最后一条匹配规则优先；Session rules 位于 Agent rules 之后，因此可能覆盖前面的匹配结果。

当用户看到批准请求时，常见回复语义是：

- `once`：只允许当前请求。
- `always`：在当前 OpenCode Instance 的内存批准集中加入相应 pattern。
- `reject`：拒绝当前请求，并结束相关等待。

当前默认旧路径中的 pending request 和 `always` approval 是进程内状态。`always` 可能在同一 Instance 的其他 Session 中生效，但不能据此认为它会跨进程重启持久保存。

### 5.8 `read` 真正接触文件系统

Permission 允许后，`read` executor 才进行真实操作：

1. 按当前项目目录解析相对路径。
2. 对项目外路径先处理 `external_directory` 授权。
3. 对最终读取资源处理 `read` 授权。
4. 读取目录、文本、图片或 PDF。
5. 对文本分页，对媒体生成相应附件表示。
6. 返回标题、文本、metadata 和可选 attachments。

当前实现对文本默认按 2000 行和 50 KiB 形成一页。超过范围时，不代表整个文件消失；下一步可以用新的 `offset` 继续读取。

回到贯穿场景，模型第一轮只读到 README 的学习地图。如果 README 指向项目规则，模型可在下一轮再生成另一个 `read` Tool Call，而不是要求一次调用吞下整个项目。

### 5.9 Tool Result 还要经过 Settlement

executor 返回的 domain/raw Tool Result 不是生命周期终点。OpenCode 还需要把调用状态从 pending、running 结算为 completed 或 error。

当前 `SessionProcessor` 处理的核心状态可以简化为：

```text
tool input -> pending
tool call  -> running
tool result success -> completed
tool result failure -> error
```

成功终态会保存输入、模型可见输出、标题、时间、metadata 和可选附件；失败终态会保存输入、错误和相关 metadata。第 10 篇会继续解释这些状态如何成为 durable Message/Part。

### 5.10 下一轮：结果重新成为 Observation

完成 Tool Settlement 后，外层 Agent Loop 不会假设模型“自动知道”文件内容。它会重新读取会话历史，把 completed/error Tool Part 投影成模型可接受的工具输出，再创建新的 Provider Request。

```text
第一轮：模型决定读取 README
-> Harness 执行 read
-> 保存 Tool terminal state

第二轮：模型看到 README 的 Model Tool Output
-> 判断还需读取项目规则
-> 生成新的 Tool Call，或给出学习路线
```

这就是 `Think -> Act -> Observe` 中的 Observe：不是模型在调用之间保留了内部记忆，而是 Harness 把工具结果放回下一轮上下文。

## 6. Tool 可见性与 Permission 是两层控制

下面这张表可以用于快速排查“为什么工具没运行”：

| 阶段 | 问题 | 失败时的表现 |
| --- | --- | --- |
| Registry | OpenCode 是否发现了这个 Tool？ | 候选集中不存在 |
| Materialization | 当前 Agent/Model 是否允许公开它？ | Provider Request 没有 definition |
| Model decision | 模型是否选择调用它？ | 只生成文本或选择别的 Tool |
| Validation | 参数是否满足 schema？ | executor 不运行 |
| Permission | 具体资源是否允许？ | ask 等待或 deny |
| Execution | 外部操作是否成功？ | error Tool state |
| Settlement | 结果是否成功投影和保存？ | 下一轮缺少可用 Observation |

“Tool 没有出现在模型面前”和“Tool 调用被 Permission 拒绝”发生在不同阶段，也有不同的调试方法。

## 7. 不同能力怎样进入 Tool 体系

| 类型 | 怎样进入当前调用体系 | 主要执行者 | 需要注意的信任边界 |
| --- | --- | --- | --- |
| Built-in Tool | OpenCode Registry 静态注册，并在外层 Loop 新一轮物化 | OpenCode 内部 executor | leaf 必须在副作用前正确调用 Permission |
| Custom Tool | 从配置目录发现并动态载入 | 自定义模块代码 | Host 不会自动替它调用一次 `ctx.ask` |
| Plugin Tool | Plugin 加载后提供 `hooks.tool` | Plugin 进程内代码 | Plugin 本身早已获得进程能力 |
| MCP Tool | 连接 Server、缓存 `tools/list`，在外层 Loop 新一轮转换 | 本地或远程 MCP Server | schema、说明和结果属于外部输入 |
| Skill | 模型调用内置 `skill` Tool 后加载说明 | `skill` executor | Skill 文本指导后续行动，不会因包含脚本就自动执行 |
| Subagent | `task` 等编排 Tool 创建或恢复子 Session | 另一个 Agent Loop | 不是普通叶子函数；详见第 11 篇 |

如何安装和配置 Custom Tool、Plugin、MCP 与 Skill，不在本篇重复，参见 [05 Enhancement](./05_Enhancement.md)。

## 8. Permission 不等于 OS Sandbox

这是本篇最重要的安全边界。

Permission 是 OpenCode 应用层中的策略门。它回答的是：

> 这次受管的 Tool 调用，是否应当继续？

操作系统沙箱（OS Sandbox）则通过低权限账户、容器、虚拟机、文件系统挂载、系统调用或网络隔离，回答：

> 即使代码尝试越界，操作系统最多允许它做到什么？

两者不能互换：

- `bash` 获准后，会以 OpenCode 进程用户的 host 权限启动真实子进程。
- `read` 的路径边界来自路径解析和 Permission gate，不是受限文件系统挂载。
- Custom/Plugin Tool 可以选择不调用 `ctx.ask`。
- Plugin 模块和 factory 在模型 Tool Call 之前就可能执行代码。
- 本地 MCP Server 在连接阶段已经启动；Tool Permission 主要约束后续受管的 `tools/call`。
- 远程 MCP Server 自身的账号权限、网络和数据保留不由 OpenCode Permission 控制。

因此，如果实验涉及未知 Plugin、Shell 命令或外部服务，仍应使用最小权限凭据和隔离环境。Permission 对话框不能替代这些边界。

## 9. 失败、拒绝和取消怎样收束

### 9.1 参数或执行错误

参数 decode 失败时，Built-in executor 不运行。文件不存在、读取失败或 MCP 返回错误时，Processor 会把调用结算为 error Tool state，使模型能够在后续轮次看到可解释的失败，而不是一直等待一个悬空调用。

### 9.2 用户拒绝

Permission reject 会使当前调用失败。默认配置下，它还会阻止当前 Loop 继续盲目尝试相同行动；兼容配置可以改变某些继续行为。

拒绝不是模型“被说服了”，而是 Harness 在执行边界阻止了副作用。

### 9.3 取消和中断

用户取消时，OpenCode 会中断当前 Runner/Provider stream，并尽量把仍 pending/running 的 Tool 标记为 interrupted error。下一次运行不应把这个历史调用当成尚待自动重放的副作用。

但中断记录不能自动撤销已经发生的外部操作。例如命令已经创建了文件，随后才被取消，单靠 Tool state 变为 error 不会还原文件。

### 9.4 `providerExecuted` 是明确例外

部分 Tool 由 Provider 托管执行。此时 OpenCode 接收 Provider 产生的调用和结果，但不会再查找并执行同名本地 executor。

它仍需要保存结果和 metadata，却不能套用“本地 Permission -> 本地 execution”的普通链路。

## 10. 输出边界：结果不是越完整越好

工具输出会占据下一轮上下文，也可能包含敏感数据。当前默认路径有多层大小控制：

- `read` 自己做分页。
- `bash` 保留受限预览，并可能把较完整输出放入受管理文件。
- 普通 Tool wrapper 对文本结果做通用截断。
- MCP adapter 对返回文本和部分媒体做转换与限制。
- 更早的 Tool output 还可能在后续 Compaction/Pruning 中失去模型可见性。

这些边界并不完全统一，文本限制也不能替代附件的 MIME 和大小检查。学习者应先记住一条原则：

> executor 得到的完整结果、Session 中保存的结果和下一轮模型看到的结果，可能是三个不同表示。

## 11. 当前默认实现与 native V2

本篇主线描述的是固定版本中普通 TUI 实际使用的兼容 Session Runtime。native V2 已引入 typed Tool、按 Location 物化、调用 identity 检查、专用 durable Tool events、持久化 `always` approval 和更集中的 Tool output store。

但 native V2 尚未完整覆盖当前默认路径中的 Custom directory Tool、Plugin Tool/Hook、MCP Tool 和 StructuredOutput 等能力，普通 TUI 也未切到这条路径。因此，本篇不把 V2 的 Permission 和 Settlement 语义写成当前使用体验。

另一个不会改变的原则是：V2 Permission 同样不等于 OS Sandbox；受信 leaf executor 仍需主动执行授权检查。

## 12. 常见误解

### 误解一：模型调用 `read`，所以模型读取了文件

模型只生成 Tool Call。OpenCode 的 executor 才读取文件。

### 误解二：模型能看到 `read`，说明任意文件都允许读取

Tool definition 可见性与具体路径 Permission 是两层控制。

### 误解三：schema 已经保证安全

schema 只验证参数结构。合法的字符串路径仍可能指向不应访问的位置。

### 误解四：选择 `always` 就永久写入权限配置

当前默认旧路径的批准记录是 Instance 内存状态，不保证跨进程重启。

### 误解五：禁止 Plugin Tool 就限制了整个 Plugin

whole-tool deny 只能影响模型是否看到该 Tool，不能撤销 Plugin 模块已有的进程权限。

### 误解六：Tool 返回成功，模型立刻知道结果

结果还要完成 Settlement、保存和历史投影，下一 Provider Turn 才把它作为 Observation 交给模型。

## 13. 本篇掌握要点

读完后，应能独立说明：

1. Registry、每轮 Tool materialization、Tool Call 和 execution 是四个不同阶段。
2. 模型只看到 name、description 和 input schema，不看到本地 executor。
3. 参数验证回答“格式是否合法”，Permission 回答“资源是否允许访问”。
4. `allow / ask / deny` 在真实副作用之前决定是否继续。
5. executor 返回的 raw result 还要经过 Tool Settlement，才形成 durable terminal state 和 Model Tool Output。
6. Tool 结果通过下一轮历史投影重新进入模型上下文。
7. Permission 是应用层策略门，不是操作系统沙箱。
8. Plugin、Custom Tool 与 MCP 扩大能力的同时，也扩大了需要审查的信任边界。

如果你能够沿着下面这条链完整复述一次 `read`，就已经掌握本篇主问题：

```text
register
-> materialize
-> schema
-> Tool Call
-> validate
-> Permission
-> execute
-> raw result
-> settle
-> persist
-> next-turn observation
```

## 14. 关键源码入口

以下入口均对应固定 commit `0e3474509aa5ad16afcf9c439785514d6443c6af`。正文不依赖易变化的行号。

| 主题 | 文件 | 关键符号 |
| --- | --- | --- |
| Built-in Tool 定义与包装 | `packages/opencode/src/tool/tool.ts` | `Tool.define`、`Tool.init`、`wrap` |
| `read` 的 schema、Permission 与执行 | `packages/opencode/src/tool/read.ts` | `ReadTool`、`Parameters`、`ReadTool.execute` |
| Tool Registry | `packages/opencode/src/tool/registry.ts` | `ToolRegistry`、`fromPlugin`、`tools` |
| 每轮 Tool 物化 | `packages/opencode/src/session/tools.ts` | `SessionTools.resolve` |
| 最终可见性过滤 | `packages/opencode/src/session/llm/request.ts` | `resolveTools`、`LLMRequestPrep.prepare` |
| Provider 调度与事件转换 | `packages/opencode/src/session/llm.ts`、`session/llm/ai-sdk.ts` | `LLM.run`、`LLMAISDK.toLLMEvents` |
| Permission | `packages/opencode/src/permission/index.ts` | `evaluate`、`ask`、`reply` |
| Tool 状态结算 | `packages/opencode/src/session/processor.ts` | `handleEvent`、`completeToolCall`、`failToolCall` |
| Session History 投影 | `packages/opencode/src/session/message-v2.ts` | `toModelMessagesEffect` |
| native V2 Tool | `packages/core/src/tool/tool.ts`、`packages/core/src/tool/registry.ts` | `Tool.make`、`materialize`、`settleWith` |

下一篇将接着追问：Tool terminal state、Message 和 Part 被保存在哪里，为什么下一轮或重新打开 Session 后还能看到它们？

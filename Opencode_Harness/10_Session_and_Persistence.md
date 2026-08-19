# Session 与 Persistence：系统保存了什么

上一篇：[09 Tools 与 Permission](./09_Tools_and_Permission.md) ｜ 下一篇：[11 Agent 专业化与协作](./11_Agent_Specialization_and_Collaboration.md)

> 源码基线：`0e3474509aa5ad16afcf9c439785514d6443c6af`

上一章中，OpenCode 读取了 Harness README，Tool 也已经结算。模型调用结束后，README 内容为什么没有立刻“消失”？如果关闭界面或重启进程，哪些记录还能找回来？屏幕上已经显示的流式文字，是否一定已经保存？

直观答案是：模型没有在调用之间自动保存记忆，是 OpenCode 把 Session、Message、Part 等运行事实写入持久化存储，并在下一轮重新读取。与此同时，Runner、流式 delta、等待批准的 Deferred 和客户端即时状态通常只存在于运行现场。因此，**数据库里有记录、下一轮模型能看见、进程重启后自动继续执行**是三个不同结论。

## 一、先用一个状态模型看懂 Session、Message、Part 与 Event

这四个名称不是同一层的并列“存储对象”。它们的关系更接近：Session 包含有序 Message，Message 包含不同类型的 Part；Event 记录这些领域对象发生了什么变化，Projector 再把变化整理成便于查询的 SQLite projection。

```text
Session：围绕同一目标持续工作的容器
└── Message：一条 User 或 Assistant 记录
    ├── Text Part
    ├── Reasoning Part
    ├── Tool Part
    └── File / Compaction / Step / Patch ...

状态变化
    ↓
Durable Event
    ↓ Projector
SQLite projection：Session / Message / Part 的当前可查询表示
```

这张图同时给出两种关系。Session、Message、Part 是包含关系：不能把一个 Tool Part 当成与 Session 平级的会话。Event 与 Projection 则是“变化事实与读取表示”的关系：事件说明某个对象发生更新，Projection 让调用者不必每次从头回放全部事件，就能查询对象当前状态。

### 1.1 Session 组织一项持续任务

会话（Session）为多轮用户输入、Assistant 输出和 Tool 状态提供共同 identity。它不是模型内部的隐藏记忆，而是 OpenCode 管理的应用状态。

以“从零学习 Harness”为例，一个 Session 可以持续包含：

```text
用户：先告诉我学习入口
Assistant：提出 read Tool Call
Tool：返回 README 内容
Assistant：总结章节顺序
用户：继续解释 Agent Loop
```

这些记录属于同一个学习目标，重载历史和客户端同步才能知道它们应被放在一起。

固定源码中的 `Session.Info` 不保存“模型脑内状态”，而是保存应用需要定位和管理会话的 metadata，例如 Session ID、Project/Workspace、目录、父 Session、标题、当前 Agent/Model、权限、token/cost 汇总、Revert marker 和时间。它提供的是共同 identity 与索引边界。

### 1.2 Message 记录总体状态，Part 承载具体内容

#### 1.2.1 Message 是容器，Part 是可独立演进的内容块

Message 保存所属 Session、role、Agent/Model 信息、parent relationship，以及 Assistant 的 finish、error、usage 等总体状态。它更像一条消息的容器和索引，不是把所有内容塞进一个字符串。

具体内容拆成 Parts，是因为它们有不同结构和生命周期。Text 可以流式增长，Reasoning 可单独标记，File 有附件信息，Tool Part 则会从 `pending` 变为 `running`，最后进入 `completed` 或 `error`。如果全部压成一个不可区分的大字段，局部更新、恢复和界面渲染都会变得困难。

schema 直接体现了这层关系：

```ts
export const Part = Schema.Union([
  TextPart,
  ReasoningPart,
  FilePart,
  ToolPart,
  StepStartPart,
  StepFinishPart,
  SnapshotPart,
  PatchPart,
  RetryPart,
  CompactionPart,
  // ...
])
```

Tool Part 内部还包含独立状态机；Text Part 有自己的文本和时间；Patch Part 保存工作树变化的索引。Part 的差异不是显示样式，而是持久化字段、状态转换和未来模型投影都不同。

一段学习过程可能形成下面的状态树：

```text
Session: ses_learning_example

├── User Message
│   └── Text Part: “请读取 Harness README，并给我学习顺序”
│
├── Assistant Message A
│   └── Tool Part: read
│       ├── input: Opencode_Harness/README.md
│       └── state: completed
│
└── Assistant Message B
    └── Text Part: “建议先读 06，再按 07-12 学习……”
```

#### 1.2.2 User Message 与 Parts 不是一次共同原子写入

当前默认路径中，`SessionPrompt.createUserMessage` 解析输入后先保存 User Message，再逐个保存 Parts。它们不是一个共同的原子写入：若中途故障，理论上可能留下 Message 和已经成功写入的前缀 Parts，而不是“全部成功或全部消失”。Persistence 不只是“有没有数据库”，还包括事务粒度和失败后能看到什么中间状态。

这条边界很重要：用户输入已经有一条 Message，不证明它的所有附件、文本和合成 Part 都已完整写入。恢复逻辑只能依据已经提交的领域事实，不能假设某个更大的应用操作天然具备数据库事务原子性。

#### 1.2.3 Assistant Message 为流式输出提供共同归属

外层 Loop 会在 Provider 请求前建立 Assistant Message，让随后出现的 Text、Reasoning、Tool、usage 和 error 都有明确父容器。Tool 的 whole Part 状态会持续更新，因此下一轮不需要依赖上一个 JavaScript 对象仍在内存中。

这种拆分还让不同内容按自己的节奏演进。模型可以先创建一个 Assistant Message，随后建立 pending Tool Part；Tool Call 到来时更新为 running，执行结束再写 completed。与此同时，同一 Message 还可能出现 Text 或 Reasoning Part。Message 提供共同归属，Part 保存各自结构，二者组合起来才是可恢复的 Assistant 输出。

### 1.3 Event 表达变化，Projection 表达当前可读状态

#### 1.3.1 Durable Event 与 Projection 在同一事务中协同

事件（Event）表示“发生了什么”，例如 Message 或 Part 已更新；Projection 表示“为了查询，现在应怎样呈现这些对象”。

```text
Event：发生过一次 PartUpdated
Projection：part 表中这个 Part 当前是什么状态
```

当前默认兼容 Session Runtime 已复用 EventV2 与 Core Projector。一次 durable event 的主要提交顺序是：在 SQLite transaction 中分配 aggregate sequence、执行 Projector、更新 sequence、插入 event row，事务提交后再通知 Listener/PubSub。Event 与 Projection 可以在同一事务中推进，但它们不是一个概念。

保留控制关系后的源码近似如下：

```ts
yield* db.transaction(() =>
  Effect.gen(function* () {
    const seq = latest + 1
    const committed = { ...event, durable: { aggregateID, seq, version } }
    for (const projector of projectors) yield* projector(committed)
    yield* db.insert(EventSequenceTable).values({ aggregate_id: aggregateID, seq })
    yield* db.insert(EventTable).values({ id: event.id, aggregate_id: aggregateID, seq, data })
  }),
)

yield* notify(event, true)
```

Projector 负责更新 `session`、`message`、`part` 等 read model，Event row 保存可按 aggregate sequence 读取的领域变化。调用者日常查询的是 Projection，不必每次手工回放整个 event log。

#### 1.3.2 带 event 名称不代表一定 durable

“durable event 已通知客户端”因此通常意味着对应事务已经提交；但不能把所有带 event 名称的通知都作此推断。whole Message/Part update 可以走 durable contract，`message.part.delta` 或部分 status 则是 live-only contract，只服务当下观察者。判断能否重放时，关键不是界面是否收到过通知，而是该通知是否进入 durable event log。

`Session.updateMessage` 与 `Session.updatePart` 发布 whole update；`Session.updatePartDelta` 则发布 `message.part.delta`。它们可以同时服务一段文本生成，却拥有不同恢复语义。

#### 1.3.3 Loop 从 Projection 重载，而不是依赖旧内存对象

这个模型能解释为什么下一轮可以继续。`SessionPrompt.run` 回到 Loop 顶部后，从 projection 重载当前有效的 Message/Part，再把已完成的 Tool Part 转换成模型可接受的 Observation：

```text
Provider Turn 1
-> read completed
-> whole Tool Part durable

Loop 回到顶部
-> 重载 active Session History
-> Tool Part 投影成 Model Tool Output
-> Provider Turn 2
```

这里的“重载历史”仍不等于把数据库全部内容逐字发给模型。哪些记录成为 active history、最终进入 Provider Request，由 [第 08 篇](./08_Context_Architecture.md)主讲；本篇只说明可选择的历史从哪里来。

把这一段与第 09 篇连接起来，就能看到一次工具结果为何跨越 Provider Turn：Tool Executor 返回 raw result，Processor 将 Tool Part 结算为 whole terminal state，Event/Projector 更新 durable projection，Loop 下一次读取 projection，最后再把选中的 Tool Part 转换为 Model Tool Output。任何一段缺失，都不能简单声称“模型已经记住工具结果”。

## 二、状态从实时生成到持久化，经历不同生命周期

### 2.1 Durable、process-local 与 live-only 不是同义词

把所有对象都称为“Session 状态”会掩盖最重要的恢复差异。更准确的分类是：

- **durable**：已经写入持久化存储，跨调用或进程仍能读取。
- **process-local**：只在当前服务进程中协调工作。
- **live-only**：只服务实时通知，不进入 durable event log。

它们在一次流式回答中同时存在：

```text
Provider text delta
-> Processor 在内存累计全文        process-local
-> message.part.delta 通知 TUI      live-only
-> TUI 立即显示
-> text-end / 受控 cleanup
-> whole Text Part 写入 SQLite      durable
```

### 2.2 Whole Part 是恢复基线，Delta 是实时体验

#### 2.2.1 Delta 负责低延迟，whole update 负责权威状态

模型生成文字时，`SessionProcessor` 在进程内累计内容，同时发布 live-only delta，TUI 因此可以边接收边渲染。Text/Reasoning 开始时会建立 whole Part，在 `text-end` 或受控 cleanup 时再把累计全文 durable 写回。

```ts
case "text-delta":
  ctx.currentText.text += value.text
  yield* session.updatePartDelta({
    partID: ctx.currentText.id,
    field: "text",
    delta: value.text,
  })
  return

case "text-end":
  ctx.currentText.time.end = Date.now()
  yield* session.updatePart(ctx.currentText)
  ctx.currentText = undefined
  return
```

这解释了一个反直觉现象：屏幕已经显示的最后几个字，未必已经落盘。如果进程在最后一次 whole update 之前硬崩溃，断开的实时连接无法补回 delta 后缀，重启后只能恢复最近一次 durable whole Part。实时传输优化响应速度，whole write 决定恢复基线。

Whole 与 Delta 不是“完整消息和残缺消息”两种业务类型，而是同一内容的两种传递粒度。Delta 适合降低首字延迟，whole update 适合成为权威恢复点。设计实时系统时经常需要同时保留两者：如果每个 token 都做 durable transaction，写放大和同步成本会很高；如果永远只发 delta，断线和重启后又缺少可靠基线。

#### 2.2.2 受控中断比硬崩溃多一次 cleanup 机会

用户主动取消与硬崩溃不等价。受控中断时，Processor cleanup 会尽量保存累计的 Text/Reasoning/Patch，把未完成 Tool 结算为 interrupted error，并完成 Assistant Message；硬崩溃没有机会执行这些收尾。即使 cleanup 成功写入 interrupted state，也只记录中断事实，不会自动撤销已经发生的 Tool side effect。

#### 2.2.3 生命周期矩阵决定能够恢复到哪里

下面的生命周期矩阵可以用来判断重启边界：

| 状态 | 生命周期 | 下一轮/重连 | 进程重启后 |
| --- | --- | --- | --- |
| Session、User/Assistant Message | durable | 可重载 | 可重载 |
| 最近一次 whole Text/Reasoning/Tool Part | durable | 可重载 | 可恢复到最近 whole write |
| durable Event 与 sequence | durable | 可作为投影、同步依据 | 记录仍可读取 |
| `message.part.delta` | live-only | 只服务当前实时连接 | 不可重放 |
| 当前 Session Runner、流式累积器 | process-local | 协调当前执行 | 清空 |
| Permission pending Deferred | process-local | 当前进程可等待 | 不恢复批准框 |
| TUI reactive store | client process-local | 可重新 hydrate | 即时状态丢失 |
| Code Snapshot 对象、Revert marker | durable | 可用于相关恢复流程 | 对象仍存在时可读取 |

Durable 的准确含义只是“跨进程可以再次读取”。它不自动保证当前 Agent Loop 重启后自行恢复、Provider 请求可以安全重放、外部 Tool 副作用可回滚，或所有在界面上闪现过的字符都已经保存。

这里还要区分服务端与客户端的恢复。服务端可以从 SQLite 重新读取 durable projection，TUI 的 reactive store 却属于客户端进程状态，需要通过 hydration 或事件同步重新构造。界面重新出现一条 Message，并不表示原来的 JavaScript store 被保存；只是新客户端根据 durable source 重建了相同领域视图。

### 2.3 Persistence 也不是模型的长期 Memory

日常语言常说“Agent 记住了 README”，实现上却至少包含三个动作：

```text
保存：哪些运行事实进入 durable store
选择：下一轮从中构造哪一段 active history
注入：哪些表示最终进入 Provider Request
```

本篇主讲保存及恢复，第 08 篇主讲选择与注入。模型本身不会因为上一轮调用结束而获得长期记忆；`AGENTS.md` 是项目指令来源，不是 Session Memory；Compaction summary 是当前 Session 的有损表示，也不是跨 Session 自动检索与复用的长期 Memory 系统。

区分这三步以后，“数据库里还有 README Tool Result”和“模型本轮仍逐字看见 README”便不再矛盾：前者描述 durable state，后者还取决于 active history、Pruning 和上下文预算。

同理，“重新打开 Session 能看见旧对话”也不能证明系统具备跨 Session 长期 Memory。前者使用明确的 Session identity 重读该会话记录；后者通常还需要抽取可复用知识、建立索引、在新任务中检索，并决定何时注入。这套固定默认路径没有因为使用 SQLite 就自动获得这些能力。

## 三、历史会变形、文件可回退，但恢复能力有明确边界

持久化不是把状态永远冻结。随着 Session 增长，OpenCode 需要改变未来使用历史的方式；当工具修改工作树后，又可能需要恢复文件并撤回对应的会话后缀。这些机制都处理“状态变化”，但作用对象不同。

### 3.1 Compaction 与 Pruning 改变模型使用的历史表示

#### 3.1.1 Compaction 保存 summary 与 tail boundary

学习过程不断积累解释、项目规则和多次 `read` 输出。原记录可以继续保存在数据库里，模型上下文窗口却不能无限逐字重放。上下文压缩（Compaction）因此创建标记，由专用 compaction Agent 生成摘要，并用 `tail_start_id` 保留近期逐字 tail。

未来模型使用的 active history 大致变为：

```text
Compaction marker
-> Assistant summary
-> retained recent tail
-> Compaction 后新增的 turns
```

摘要保留被选中的目标、约束、决定和进度，recent tail 保留近期细节；没有进入摘要且不在 tail 中的旧细节会从模型可见表示中消失。原 Message/Part、marker、summary 和 boundary 通常仍在 durable store，`filterCompacted` 选择的是 active representation，不是物理删除全部旧历史。

Compaction 因而同时具备“保存新状态”和“有损改变未来视图”两面。summary、marker 和 boundary 自身是新的持久化事实；但 summary 不可能逐字保留所有旧内容，模型后续推理只能依据摘要选中的信息与 recent tail。它解决的是有限上下文中的连续性，不是无损归档。

`filterCompacted` 会把模型消费顺序重组为 compaction marker、summary、retained tail 和后续消息。这个数组顺序不再等于数据库记录的严格时间顺序：

```ts
return [
  ...result.slice(compactionIndex, summaryIndex + 1),
  ...result.slice(tailIndex, compactionIndex),
  ...result.slice(summaryIndex + 1),
]
```

因此“原记录仍在”与“模型未来不再逐字看见旧前缀”可以同时成立。第 08 篇主讲这种可见性变化，本篇强调 marker、summary 和 boundary 本身也是可持久化的 Session state。

#### 3.1.2 Pruning 只标记旧 Tool output 不再进入模型表示

Pruning 的对象更窄：它处理较旧、较大的 completed Tool output。达到阈值后，当前实现给选中的 Tool Part 写入 `state.time.compacted`，模型投影改成类似 `[Old tool result content cleared]`，但原 `state.output` 字段不会因此被物理清空。

```ts
if (pruned > PRUNE_MINIMUM) {
  for (const part of toPrune) {
    part.state.time.compacted = Date.now()
    yield* session.updatePart(part)
  }
}
```

后续 `toModelMessagesEffect` 检查这个时间标记，再决定使用占位符并移除旧附件。这里改变的是 Provider-visible representation，不是把 `state.output` 改写为空字符串。

选择过程是从新向旧回看：至少保护最近两个 User turns，遇到已有 Assistant summary 时停止，并跳过受保护的 `skill` Tool；在保留约 40,000 tokens 的近期 Tool output 后才开始收集候选，只有预计至少回收约 20,000 tokens 才实际写入 compacted marker。这些阈值是当前实现策略，不是 Pruning 概念本身的永恒定义。

#### 3.1.3 两者都不是物理删除聊天记录

因此两者不能混称为“清理聊天记录”：Compaction 用摘要和近期 tail 替代旧前缀，Pruning 隐藏特定旧 Tool output；它们都改变未来可见性，却不等于删除 durable source。

还可以用“变换对象”快速区分：Compaction 面向会话前缀，产物是 summary 加 tail boundary；Pruning 面向已完成 Tool Part，产物是模型投影中的占位表示。前者试图保留任务脉络，后者主要回收高体积 Observation 的上下文预算。

#### 3.1.4 自动 Compaction 与手动边界使用同一持久化产物

当前默认路径既可能在本地 token 估算判断历史将超出模型预算后创建自动 Compaction，也可能在 Provider 以 Context Overflow 结束一次生成后安排带 `overflow` 标记的 Compaction。两条触发路径最终都创建 Compaction Part，并由同一 process 生成 summary 与 tail boundary；触发原因不同，持久化产物的角色相同。

当配置明确关闭自动 Compaction 时，本地 overflow 预测不会自动创建 marker，Provider overflow 也会把当前 Assistant 结束为错误，而不是偷偷生成摘要继续。因此“Session 变长”不保证一定会压缩；是否允许自动触发本身也是运行配置边界。手动请求 Compaction 则应与自动触发区分，不能只凭 Part 存在就推断它由 token threshold 触发。

### 3.2 Code Snapshot 与 Revert 处理工作树和会话后缀

#### 3.2.1 Code Snapshot 保存的是工作树基线

Code Snapshot 保存文件工作树状态，用于计算 diff、展示改动、restore 与 Revert。它不保存模型输入，也不是聊天记录的副本。

Processor 在 step 开始时可以 `Snapshot.track()`，结束后再计算 patch。`Snapshot.restore()` 与 `Snapshot.revert(patches)` 操作工作树基线或补丁；这些函数处理的是文件内容，而不是 Session Messages。

#### 3.2.2 Revert 协调文件状态和会话后缀

当前 Revert 大致协调两个状态面：先根据 boundary 之后的变化恢复工作树并在 Session 上记录 revert marker；随后 cleanup 通过 remove events 移除相应 conversation projection suffix 并清理 marker，durable event log 仍保留删除事实。`unrevert` 则恢复暂存前的工作树状态。

控制关系可以压缩为：

```text
选择 Message / Part boundary
-> 收集 boundary 之后的 Patch Parts
-> 记录当前工作树 snapshot
-> revert patches，计算 diff
-> 在 Session 上保存 revert marker
-> 后续 cleanup 发布 Message/Part remove events
-> 清除 revert marker
```

Revert 要求 Session 当前不 busy，避免在活动 Runner 同时改变 Message/Part 和工作树时执行撤回。它能协调仓库文件与对话后缀，却无法证明外部 API、远程数据库或已经发出的网络请求也被撤销。

所以 Revert 可能同时触及文件系统与 SQLite conversation projection，却不能撤销任意外部服务中的副作用。不要把它与 Compaction 混淆：Compaction 改变未来模型使用的历史表示；Revert 协调文件变化和会话后缀。

#### 3.2.3 Code Snapshot、Revert marker 与 Context Snapshot 保存不同对象

三个容易混淆的对象可以按保存内容区分：

| 对象 | 保存或标记什么 | 主要用途 | 不保证什么 |
| --- | --- | --- | --- |
| Code Snapshot | Git tree/index 形式的工作树状态 | diff、restore、Revert | 保存模型上下文 |
| Revert marker | 会话中的撤回边界与状态 | 协调工作树恢复和 conversation cleanup | 撤销任意外部服务副作用 |
| Context Snapshot（native V2） | Context Source 上次 admitted value | 比较环境或指令是否变化 | 恢复代码工作树 |

完整 native V2 架构演进集中在 [第 12 篇](./12_Runtime_Boundary.md)，这里不继续展开 Context Snapshot 的内部接线。

### 3.3 重启能够重载事实，不能自动续跑现场

#### 3.3.1 可读取的 durable state 与消失的运行现场

进程重启后，当前默认路径通常可以重读 Session metadata、已提交 Message、最近 whole Part、Tool terminal state、durable Event/projection、Compaction 状态，以及仍存在的 Revert marker 和 Code Snapshot 对象。不能直接恢复的是原 Session Runner、Provider stream、未 whole-save 的 delta、Permission Deferred、Tool promise/fiber 和 TUI 即时渲染状态。

可以把恢复分成两个问题：

```text
事实恢复：数据库和工作树中还能读取什么？
执行恢复：谁拥有未完成任务，能否安全地再次发起副作用？
```

当前默认路径主要提供第一类基础，不会仅凭一条 pending Part 自动完成第二类判断。

#### 3.3.2 副作用与 terminal state 之间存在未知区间

最棘手的边界发生在副作用与记录之间：Tool 可能已经完成文件或外部操作，但进程在 terminal state 落库前崩溃。此时 durable history 不足以证明副作用没有发生，也不能安全地决定是否重放。恢复通常需要新的 prompt 或明确 resume 入口重新驱动，并结合幂等设计、外部操作审计和副作用检查。

可以按失败时点理解可能保留的事实：

- Message 写入后、Parts 尚未全部写入：可能留下 Message 与已提交的前缀 Parts。
- Text delta 中硬崩溃：只保证最近 whole Part，不保证 UI 最后显示的后缀。
- Tool 执行后、结果落库前：外部副作用可能存在，却没有 durable terminal state。
- 受控 interrupt：cleanup 尽量保存 whole content 和 interrupted state，但不回滚外部副作用。
- Provider retry：在同一 Assistant/Processor 上下文内重试，不代表先从数据库重载最新历史。
- 进程重启：durable state 可读取，原 Runner 不会因此自动复活。

这些边界说明 Persistence 的价值是提供恢复所需的事实基础，而不是消除不确定性。可靠续跑还需要知道请求是否具备幂等性、外部系统是否接收过操作、Tool 能否查询当前结果，以及由谁决定继续或人工介入。仅凭一条 pending Tool Part，Harness 无法安全推断“上次一定没执行”。

#### 3.3.3 Native V2 把部分状态边界显式化，但没有消除恢复难题

native V2 进一步区分 durable `PromptAdmitted` 与进入模型历史的 `Prompted`，用 Started/Ended 保存 Text、Reasoning 和 Tool Input 的边界，让 Delta 继续保持 live-only，并以 completed checkpoint 和 Context Epoch 描述新的 active history 边界。但它仍不等于 durable 自动执行：Session drain、协调器和 Tool fibers 仍有 process-local 部分，普通 TUI 在固定版本中也尚未使用完整 native V2 Prompt 路径。

其中 `PromptAdmitted` 与 `Prompted` 的区分尤其重要：输入已被系统可靠接收，不代表它已经在安全执行边界进入模型历史。Checkpoint 也只是 active history 的新边界，不是“保存所有内存现场”的虚拟机快照。native V2 把状态语义描述得更明确，但没有取消外部副作用、进程协调和崩溃恢复本身的工程难题。

## 四、关键源码索引

正文展示了理解状态关系所需的关键控制代码。继续阅读固定源码时，可以按下面的入口定位：

| 主题 | 源码文件 | 关键符号 |
| --- | --- | --- |
| Session metadata 与领域更新 | `packages/opencode/src/session/session.ts` | `Session.Info`、`updateMessage`、`updatePart` |
| Message/Part schema | `packages/schema/src/v1/session.ts` | `User`、`Assistant`、`Part`、`ToolPart` |
| 历史读取和 active representation | `packages/opencode/src/session/message-v2.ts` | `stream`、`filterCompacted`、`toModelMessagesEffect` |
| whole Part 与 live delta | `packages/opencode/src/session/processor.ts` | `handleEvent`、`cleanup` |
| durable Event transaction | `packages/core/src/event.ts` | `commitDurableEvent`、`publishEvent` |
| Compaction 与 Pruning | `packages/opencode/src/session/compaction.ts` | `select`、`process`、`prune` |
| 工作树 Snapshot | `packages/opencode/src/snapshot/index.ts` | `track`、`patch`、`restore`、`revert` |
| 会话 Revert | `packages/opencode/src/session/revert.ts` | `revert`、`unrevert`、`cleanup` |

完整跨章节证据与代表性测试见[源码与证据索引](./appendices/Source_Index.md)。

## 五、总结：保存事实不等于保存运行现场

本篇的中心结论可以压缩成一句话：**系统保存的是可重载的运行事实；模型看到的是 Harness 从这些事实中为本轮选择出的有限表示；运行现场则可能随进程结束而消失。**

```text
Session
└── Messages
    └── Parts
         │
         ├── whole update -> durable Event -> Projection -> 可重载事实
         `── delta        -> live-only event              -> 实时体验

Compaction / Pruning -> 改变未来模型使用的历史表示
Snapshot / Revert    -> 协调工作树与会话后缀
进程重启             -> 可重载 durable state，不会自动复活 Runner
```

理解这些边界后，下一篇将在持久化状态之上解释：不同 Agent 如何通过指令、Tool 与 Permission 形成专业分工，Plan、Todo、Task 和 Subagent 又分别承担什么职责。

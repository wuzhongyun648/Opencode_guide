# Session 与 Persistence：系统保存了什么

上一篇：[09 Tools 与 Permission](./09_Tools_and_Permission.md)
下一篇：[11 Agent 专业化与协作](./11_Agent_Specialization_and_Collaboration.md)

> 源码基线：`0e3474509aa5ad16afcf9c439785514d6443c6af`

## 1. 学习问题

在上一章的学习场景中，OpenCode 已经读取了 Harness 的 README，并准备继续查看项目规则。模型调用结束之后，README 内容为什么没有立刻“消失”？关闭界面或重启进程后，哪些内容还能找回来？如果界面已经显示了一段流式文字，它是否一定已经保存？

本篇只回答一个问题：

> **OpenCode 为一次 Agent 学习过程保存了什么，哪些状态能够恢复，哪些只存在于当前运行现场？**

## 2. 最短答案

模型本身没有在调用之间自动保留会话记忆。OpenCode 把 Session、Message、Part 和相关事件保存到持久化存储，并在下一轮重新读取这些状态，模型才得以继续先前任务。

并非所有运行状态都会落盘。完整 Message/Part 可以持久化，流式 delta、当前 Runner、等待中的 Deferred 和客户端渲染状态通常只存在于进程或实时连接中。

因此，“数据库里有记录”“下一轮模型能看见”“进程重启后任务会自动继续”是三个不同结论。

## 3. 最小心智模型

阅读下面的图时，重点区分“发生过什么”“方便读取什么”和“此刻正在运行什么”。

```text
用户输入 / 模型输出 / Tool 状态变化
                 ↓
            Domain Event
                 ↓
      Durable Event + Projector
                 ↓
        SQLite Read Projections
       Session / Message / Part
                 ↓
       下一 Provider Turn 重载

同时存在但不一定持久化：
Runner / stream accumulator / delta / pending approval / TUI store
```

可把状态分成三类：

```text
durable       跨调用或进程仍可读取
process-local 只在当前服务进程中协调运行
live-only     只用于实时通知，不进入 durable event log
```

## 4. 先分清 Session、Message、Part 与 Event

### 4.1 Session：一次持续工作的容器

会话（Session）把围绕同一目标的多轮用户输入、Assistant 输出和 Tool 状态关联起来。它不是模型内部的隐藏记忆，而是 OpenCode 管理的应用状态。

在贯穿场景中，“从零学习 Harness”可以放在一个 Session 中持续推进：

```text
用户：先告诉我学习入口
Assistant：提出 read Tool Call
Tool：返回 README 内容
Assistant：总结章节顺序
用户：继续解释 Agent Loop
```

这些记录共享 Session identity，后续读取和界面同步才能知道它们属于同一项学习任务。

### 4.2 Message：谁在某一轮表达了什么

Message 保存一条 User 或 Assistant 记录的总体信息，例如：

- 所属 Session；
- role；
- Agent 和 Model 等运行信息；
- parent relationship；
- finish、error、usage 等 Assistant 状态。

Message 更像一个容器和索引，具体内容通常拆成 Parts。

### 4.3 Part：Message 中可以独立演进的内容块

一个 Message 可以包含不同类型的 Part：

- Text Part；
- Reasoning Part；
- File Part；
- Tool Part；
- Compaction Part；
- Step、Patch 等运行记录。

Tool Part 尤其适合说明为什么需要拆分。一次 `read` 会经历 pending、running、completed 或 error，而 Assistant 的文本可能仍在流式生成。把它们都塞进一个不可区分的大字符串，既难更新，也难恢复。

### 4.4 Event：系统中已经发生的事实

事件（Event）表示状态变化，例如 Message 已更新、Part 已更新。Durable Event 在提交时经过投影器（Projector），更新便于查询的 SQLite projection。

```text
Event：发生过一次 PartUpdated
Projection：part 表中现在应当怎样表示这个 Part
```

Event 和 projection 可以在同一事务中推进，但不是同一个概念。前者表达 domain fact，后者服务于高效读取。

## 5. 贯穿场景：一段 Harness 学习过程怎样留下记录

继续使用低风险示例，路径和 ID 均为说明性值：

```text
Session: ses_learning_example

User Message
└── Text Part: “请读取 Harness README，并给我学习顺序”

Assistant Message A
└── Tool Part: read
    ├── input: Opencode_Harness/README.md
    └── state: completed

Assistant Message B
└── Text Part: “建议先读 06，再按 07-12 学习……”
```

### 5.1 用户输入先成为 durable domain state

当前默认路径中，`SessionPrompt.createUserMessage` 会解析 Text、File、目录或 MCP Resource 等输入，之后保存 User Message，再逐个保存 Parts。

这里有一个重要边界：

> User Message 与所有 Parts 不是一个共同的原子写入。

Message 和每个 Part 分别发布 durable event。正常情况下它们都会成功；若中途发生故障，理论上可能留下 Message 和已成功写入的前缀 Parts，而不是“全部成功或全部消失”。

这类边界说明 Persistence 不只是“有没有数据库”，还包括事务粒度和失败时能看到什么中间状态。

### 5.2 Assistant Message 在 Provider 请求前建立

外层 Loop 每轮重载历史、选择 Agent/Model 后，会先创建当前 Assistant Message，再调用 Provider。这样后续 Text、Reasoning、Tool Call、usage 和 error 都有明确的父容器。

当模型决定读取 README 时，Tool Part 会沿着状态机变化：

```text
pending -> running -> completed
                    \-> error
```

这些 whole Tool Part 更新形成 durable state。下一轮不需要依赖上一次 JavaScript 对象仍在内存里，而是可以从 projection 重新读取已结算结果。

### 5.3 下一轮从持久化历史重新开始

当前 `SessionPrompt.run` 的每轮开头都会调用 `MessageV2.filterCompactedEffect(sessionID)`，从 SQLite projection 重载 Message/Part，并选择当前有效历史。

```text
Provider Turn 1
-> read completed
-> whole Tool Part durable

Loop 回到顶部
-> 重载 Session History
-> 把 Tool Part 投影成 Model Tool Output
-> Provider Turn 2
```

这解释了为什么模型第二轮能根据 README 内容给出学习顺序：系统重新注入了保存的 Tool observation。

但“重新读取历史”不等于逐字把数据库所有记录都发给模型。第 08 篇负责解释模型可见性；本篇只说明被选择的历史从哪里恢复。

## 6. 三种生命周期：durable、process-local、live-only

下面的矩阵是理解恢复边界的核心。

| 状态 | 生命周期 | 下一轮可用 | 进程重启后 |
| --- | --- | --- | --- |
| Session row | durable | 可重载 | 可重载 |
| User/Assistant Message | durable | 可重载 | 可重载 |
| whole Text/Reasoning/Tool Part | durable | 可重载 | 可恢复到最近一次 whole write |
| durable Event 与 sequence | durable | 可作为投影与同步依据 | 记录仍可读取 |
| `message.part.delta` | live-only | 不作为权威历史 | 不可重放 |
| 当前 Session Runner | process-local | 协调当前执行 | 清空 |
| 流式文本累积器 | process-local | 当前 turn 可用 | 未 whole-save 的后缀丢失 |
| Permission pending Deferred | process-local | 当前等待可用 | 不恢复等待框 |
| TUI reactive store | client process-local | 当前界面可见 | 需要重新 hydrate |
| 代码工作树 Snapshot 对象 | durable filesystem object | 可用于 diff/restore | 对象仍存在时可用 |
| Revert marker | durable | 可继续 cleanup/unrevert | 可重新读取 |

“Durable”在这里的准确含义是跨调用或跨进程可以再次读取。它不自动保证：

- 当前 Agent Loop 会在重启后自行恢复；
- 已发出的 Provider 请求可以安全重放；
- 外部 Tool 的副作用能够回滚；
- 所有实时显示过的字符都已保存。

## 7. Whole Part 与流式 Delta

### 7.1 为什么界面能边生成边显示

模型生成文本时，`SessionProcessor` 会在进程内累计内容，并把增量（delta）作为 live-only event 发布。TUI 收到后可以立即更新界面，不必等待整段响应结束。

```text
Provider text delta
-> Processor 内存累计
-> live-only message.part.delta
-> TUI 立即渲染
```

### 7.2 为什么“屏幕上看见”不等于“已经落盘”

Text/Reasoning 开始时会建立 whole Part；在 `text-end` 或受控 cleanup 时，再把累计全文 durable 写回。

如果进程在最后一次 whole update 之前硬崩溃，TUI 可能已经显示了结尾几个字，但 SQLite 中只有较早的 whole Part。重启后只能恢复 durable 版本，不能从已经断开的实时连接补回那段后缀。

这是一条通用工程原则：

> 实时传输优化响应速度，持久化边界决定恢复能力。

### 7.3 受控中断比硬崩溃保留得更多

用户主动取消时，Processor cleanup 会尽量：

- whole-save 已累积的 Text/Reasoning；
- 把未完成 Tool 标成 interrupted error；
- 结束 Assistant Message。

但“尽量保存状态”不能撤销已经发生的 Tool side effect。一个命令已经写入临时学习文件后再被取消，数据库记录和文件系统仍可能需要分别处理。

## 8. EventV2 与 SQLite Projection

当前默认旧 Session Runtime 已经复用 EventV2 和 Core Projector。一次 durable event 的主提交顺序可概括为：

```text
进入 SQLite transaction
-> 分配 aggregate sequence
-> 执行 Projector
-> 更新 event sequence
-> 插入 event row
-> transaction commit
-> 通知 Listener / PubSub
```

这带来两个有用结论：

1. Durable whole update 是先成功提交，再作为已提交状态通知观察者。
2. Live-only event 跳过 durable transaction，直接服务于实时观察。

所以“事件到达 TUI”仍需看它属于哪种 contract：whole Message/Part update 可以 durable，delta 或 status 则可能只是 live-only。

## 9. Persistence 不等于模型 Memory

日常语言里常说“Agent 记住了前面的内容”，但实现上至少要拆成三步：

```text
保存：哪些状态进入 durable store
选择：下一轮从哪些记录构造 active history
注入：哪些内容最终进入 Provider Request
```

本篇主讲第一步，并说明第二步从 projection 读取。第 08 篇主讲后两步怎样决定模型本轮可见内容。

这也意味着：

- Session History 是一个会话的持久化交互记录。
- 模型本身不会因为上一轮调用结束而拥有长期记忆。
- `AGENTS.md` 是项目指令来源，不是 Session Memory。
- Compaction summary 是当前 Session 的有损表示，不是跨 Session 的自动长期记忆。
- OpenCode 保存了会话，不代表已经实现了自动抽取、跨 Session 检索和复用的长期 Memory 系统。

## 10. Compaction：保存历史与控制模型窗口是两件事

### 10.1 为什么需要 Compaction

随着学习继续，Session 会积累：

- 多轮解释；
- 多次 `read` 的 Tool output；
- 项目规则和术语讨论；
- 可能很长的命令结果。

这些内容可以留在持久化存储里，但模型上下文窗口有限，不能无限逐字重放。上下文压缩（Compaction）因此改变“未来模型使用哪种历史表示”，而不是简单删除整个会话。

### 10.2 当前默认路径怎样形成 active history

当前 Compaction 会创建标记，并由专门的 compaction Agent 生成摘要；近期一段历史通过 `tail_start_id` 保留为逐字 tail。

完成后，模型可见历史大致变成：

```text
Compaction marker
-> Assistant summary
-> retained recent tail
-> Compaction 后新增的 turns
```

对未来模型而言：

- 摘要保留被选中的目标、约束、决定和进度；
- recent tail 保留近期逐字细节；
- 没进入摘要、又不在 tail 中的旧细节会丢失；
- 旧媒体和过大的 Tool output 也可能不再可见。

对 durable store 而言，原 Message/Part、Compaction marker、summary 和 boundary 通常仍然存在。`filterCompacted` 做的是选择 active representation，不是把旧数据库历史全部物理删除。

### 10.3 Compaction 与第 08 篇的职责边界

本篇解释 summary、marker 和 boundary 怎样保存，以及恢复时依据什么选择 active history。第 08 篇解释 Compaction 后哪些 Messages 最终进入 Provider Request。两者分别回答“系统保存什么”和“模型本轮看见什么”。

## 11. Pruning：隐藏旧 Tool Output，不等于生成摘要

工具输出裁剪（Pruning）专门处理较旧、较大的 completed Tool output。达到阈值后，当前实现会为选中的 Tool Part 写入 `state.time.compacted`；模型投影改成类似：

```text
[Old tool result content cleared]
```

但原 `state.output` 字段没有因此被物理清空。

因此：

- Compaction 用摘要和近期 tail 替代旧会话前缀。
- Pruning 隐藏部分旧 Tool output。
- 两者都会改变未来模型可见性。
- 两者都不能简单描述为“数据库把旧消息删了”。

## 12. Code Snapshot 与 Revert 不是 Context Snapshot

### 12.1 Code Snapshot 保存文件状态

OpenCode 可以在 Provider stream 前后捕获代码工作树快照（Code Snapshot），计算 patch，并把 snapshot hash 或文件信息关联到 Session Part。

它服务于：

- 展示本轮文件改动；
- 计算 diff；
- 恢复工作树；
- 支持 Revert。

它不保存模型输入，也不是聊天记录副本。

### 12.2 Revert 协调两类状态

当前 Revert 大致分成两个阶段：

1. 暂存撤回：计算 boundary 之后的文件变化，恢复工作树，并在 Session 上记录 revert marker。
2. cleanup：通过 remove events 删除相应 conversation projection suffix，并清理 marker；durable event log 仍保留删除事实。`unrevert` 则恢复暂存前工作树状态。

因此 Revert 同时碰到：

- 文件系统中的工作树状态；
- SQLite 中的 conversation projection。

它不是一次普通 Context 变换，也不保证撤销任意外部服务中的副作用。

### 12.3 不要混淆两个 Snapshot

| 名称 | 保存内容 | 主要用途 |
| --- | --- | --- |
| Code Snapshot | Git tree/index 形式的工作树状态 | diff、restore、Revert |
| Context Snapshot | native V2 中 Context Source 上次 admitted value | 比较环境/指令是否变化 |

名称相似，但它们的存储、生命周期和使用者都不同。

## 13. 进程重启后究竟能恢复什么

### 13.1 可以重新读取

当前默认路径通常可以重新读取 Session metadata、已提交的 Message、最近一次 whole Part、Tool terminal state、durable Event/projection、Compaction 状态，以及仍存在的 Revert marker 和 Code Snapshot 对象。

### 13.2 不能直接恢复

进程退出后，当前 Runner、Provider stream、尚未 whole-save 的 delta、Permission Deferred、Tool promise/fiber 和 TUI 即时渲染状态都会失效。

### 13.3 Durable history 不等于自动继续执行

重新打开 Session 时，OpenCode 可以重载历史；但它不会因此安全地猜测：

- Provider 是否已经收到上一次请求；
- Tool side effect 是否已经开始；
- 一个未完成命令是否适合重放；
- 外部服务是否已经处理过调用。

当前默认 Runner/Status 是 process-local。重启后通常需要新的 prompt 或 resume 入口重新驱动，而不是把“有历史”理解成“任务自动从断点继续”。

## 14. 典型失败边界

| 失败时点 | 可能保留的状态 | 不能保证的内容 |
| --- | --- | --- |
| Message 已写、部分 Parts 未写 | Message 和已提交的前缀 Parts | 输入整体原子性 |
| Text delta 中硬崩溃 | 最近 whole Part | UI 最后显示的 delta 后缀 |
| Tool 执行后、结果落库前 | 外部副作用可能已发生 | durable Tool terminal state |
| 受控 interrupt | cleanup 尽量保存 whole content 和 interrupted state | 外部副作用自动回滚 |
| Provider retry | 同一 Assistant/Processor 内重试 | retry 前重载最新 history |
| 进程重启 | durable Session/Message/Part/Event | 原 Runner 自动恢复 |

Persistence 的价值是给恢复提供事实基础，而不是消除所有不确定性。可靠系统还需要幂等、外部操作审计、明确的 resume 策略和副作用边界。

## 15. 当前默认实现与 native V2

本篇主线描述普通 TUI 当前使用的兼容 Session Runtime。native V2 对持久化边界做了更明确的建模：

- `PromptAdmitted` 先把输入写入 durable `session_input`。
- `Prompted` 再在安全边界把它提升为模型可见 User Message。
- Text/Reasoning/Tool Input 使用 durable Started/Ended，Delta 仍是 live-only。
- completed checkpoint 成为新的 active history boundary。
- Context Epoch 保存 baseline 与用于比较的 Context Snapshot。

这些机制把“输入已接收”和“输入已进入模型历史”分开，也让 checkpoint 与 typed Context 状态更清晰。

但 native V2 仍不等于 durable 自动执行：Session drain、协调器和 Tool fibers 依然是 process-local，已 dispatch 工作的 post-crash continuation 仍需要单独设计。普通 TUI 也没有在固定版本中使用这条 native V2 Prompt 路径。

## 16. 本篇掌握要点

读完后，应能独立说明：

1. Session 是多轮工作容器，Message 是角色记录，Part 是可独立演进的内容块。
2. Event 表示 domain fact，SQLite projection 服务于查询和重载。
3. whole Message/Part 可以 durable，delta、Runner 和 Deferred 可能只是 live-only 或 process-local。
4. 下一 Provider Turn 通过重载持久化 Message/Part 获取上一轮 Tool observation。
5. Durable history 支持恢复读取，但不保证崩溃后自动继续执行。
6. Compaction 保存摘要和 boundary，并有损改变 active history；原记录通常仍在。
7. Pruning 隐藏旧 Tool output，不等于摘要或物理删除 output。
8. Code Snapshot、Context Snapshot 和 Revert 分别处理不同状态。
9. Session Persistence 不等于模型天然长期记忆。

可以用下面这句话检查自己是否抓住了本篇边界：

> **系统保存的是可重载的运行事实；模型看到的是 Harness 从这些事实中为本轮选择出的有限表示。**

## 17. 关键源码入口

以下入口均对应固定 commit `0e3474509aa5ad16afcf9c439785514d6443c6af`。正文不依赖易变化的行号。

| 主题 | 文件 | 关键符号 |
| --- | --- | --- |
| User Message/Parts 与外层 Loop | `packages/opencode/src/session/prompt.ts` | `createUserMessage`、`SessionPrompt.run` |
| Message/Part history 与 active selection | `packages/opencode/src/session/message-v2.ts` | `stream`、`filterCompactedEffect`、`toModelMessagesEffect` |
| Stream 与 whole Part | `packages/opencode/src/session/processor.ts` | `handleEvent`、`cleanup`、`process` |
| Message/Part durable publication | `packages/opencode/src/session/session.ts` | `updateMessage`、`updatePart`、`updatePartDelta` |
| Event commit | `packages/core/src/event.ts` | `commitDurableEvent`、`publishEvent` |
| SQLite projection | `packages/core/src/session/projector.ts` | V1 `MessageUpdated`、`PartUpdated` projectors |
| Compaction 与 Pruning | `packages/opencode/src/session/compaction.ts` | `select`、`processCompaction`、`prune` |
| Code Snapshot | `packages/opencode/src/snapshot/index.ts` | `track`、`patch`、`restore` |
| Revert | `packages/opencode/src/session/revert.ts` | `revert`、`cleanup`、`unrevert` |
| native V2 admission/history | `packages/core/src/session/input.ts`、`packages/core/src/session/history.ts` | `admit`、`projectPrompted`、`entriesForRunner` |
| native V2 checkpoint/context | `packages/core/src/session/compaction.ts`、`packages/core/src/session/context-epoch.ts` | `compactIfNeeded`、`prepare`、`replace` |

下一篇将进一步解释：不同 Agent 如何通过指令、工具和 Permission 形成专业分工，以及 Plan、Todo、Task 与 Subagent 分别承担什么职责。

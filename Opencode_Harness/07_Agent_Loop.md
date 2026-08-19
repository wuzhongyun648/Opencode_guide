# Agent Loop：一次请求为什么可以持续多轮

上一篇：[06 Harness 总览](./06_Harness.md)
下一篇：[08 Context Architecture](./08_Context_Architecture.md)

> 固定源码：`0e3474509aa5ad16afcf9c439785514d6443c6af`（`dev`）
> 本篇主线：当前默认 TUI 的普通消息路径，即兼容 Session API 与 `SessionPrompt` 编排。

## 1. 学习问题

假设你第一次接触 Harness，对 OpenCode 说：

> 请带我从零学习这个项目的 Harness 架构。先查看教程入口和项目规则，告诉我应该按什么顺序学习。只做读取和解释，不要修改文件。

你只提交了一次请求，OpenCode 却可能先读 README，再寻找项目规则，随后读取相关章节，最后才整理学习路线。

问题是：**为什么一次用户请求能够产生多次模型判断和多次工具行动？是谁让这个过程继续，又是谁让它停止？**

### 最短答案

模型每次只根据当前输入，提出下一段文字或下一次工具调用。OpenCode 的 Harness 在模型外运行一个显式循环：组装本轮输入、调用模型、执行并记录工具结果，再判断是否需要下一轮。本系列把每次真实发起的 Provider Request attempt 及其响应或错误投影称为一个提供商轮次（Provider Turn），而一次用户请求可以包含多个 Provider Turn。继续、重试、中断和停止都由 Harness 的控制流落实，不是模型拥有一个会永久运行的内部线程。

## 2. 最小心智模型

先只记住这一条反馈循环：

```text
当前目标与状态
      |
      v
组织本轮输入
      |
      v
请求模型判断下一步       <--- 一个 Provider Turn
      |
      +---- 最终文本 ----------------------+
      |                                    |
      +---- Tool Call                      |
               |                           |
               v                           |
        校验、授权、执行                    |
               |                           |
               v                           |
        保存 Tool Result                   |
               |                           |
               +---- 需要继续 -> 回到顶部   |
                                           v
                                           停止
```

这张图中最关键的分工是：

- 模型提出“下一步做什么”。
- Harness 决定“这个动作怎样进入受控执行流程”。
- Tool Runtime 完成真正的读取或其他外部操作。
- Session 状态让下一轮能够重新取得已经发生的结果。

本篇只展开这条循环。工具注册与 Permission 由[第 09 篇](./09_Tools_and_Permission.md)主讲，Session 的持久化结构由[第 10 篇](./10_Session_and_Persistence.md)主讲。

## 3. 三个时间尺度不要混淆

理解循环前，需要区分用户请求、Provider Turn 和流式事件。

| 名称 | 含义 | 在学习场景中的例子 |
| --- | --- | --- |
| 用户请求 | 用户提交的一次目标 | “从零学习 Harness，先读取入口与规则” |
| Provider Turn | Harness 真实发起的一次 Provider Request attempt，以及其响应或错误投影 | 模型第一次决定读取 README |
| 流式事件 | 一个 Provider Turn 内逐步返回的文本、推理、工具调用、用量或完成信息 | 文本逐段显示、Tool Call 参数逐步形成 |

一次用户请求可能只有一个 Provider Turn：模型直接给出答案，不调用工具。

也可能包含多个 Provider Turn：

```text
Provider Turn 1：模型提出读取教程 README
-> Harness 执行 read，保存结果

Provider Turn 2：模型根据 README 提出读取 AGENTS.md
-> Harness 执行 read，保存结果

Provider Turn 3：模型综合两次观察，给出学习顺序
-> Harness 判断无需继续，结束
```

因此，“用户只说了一句话”和“模型只被调用一次”没有必然关系。

## 4. 通用 Agent 原理：反馈，而不是一次性生成

在普通聊天中，可以把过程近似理解为：

```text
输入 -> 模型 -> 输出
```

Agent 系统增加了外层反馈：

```text
目标 -> 判断 -> 行动 -> 观察 -> 新判断 -> ... -> 完成
```

这与阅读笔记中的 `Think -> Act -> Observe` 是同一个核心思想，但工程实现不应被误写成模型的隐藏思维过程。

- `Think` 对应模型基于本轮上下文生成下一步提议。
- `Act` 对应 Harness 受控地调度工具。
- `Observe` 对应工具结果被结算、保存，并进入后续模型输入。
- 外层 Loop 负责决定是否再经历一次这个过程。

模型的计划也不是第一次生成后永久固定。它读完 README 后可能发现真正入口在另一个目录，于是调整下一步。这种根据新观察改变路径的能力，正是反馈循环的价值。

## 5. OpenCode 当前默认 Loop 从哪里开始

当前默认 TUI 提交普通消息时，调用的是兼容 Session API：

```text
TUI
-> client.session.prompt(...)
-> POST /session/:id/message
-> SessionHttpApi.prompt
-> SessionPrompt.prompt
-> SessionPrompt.loop
-> SessionPrompt.run
```

`SessionPrompt.prompt` 先创建 User Message 和 Parts，再进入 Loop。`SessionPrompt.loop` 通过 `SessionRunState.ensureRunning` 协调同一个 Session 的运行，然后由 `SessionPrompt.run` 中显式的 `while (true)` 推进任务。

这里有两个容易忽略的事实：

1. Loop 写在 OpenCode 的编排代码里，不藏在 Provider 或模型内部。
2. 同一个 Session 的多个 Loop 调用会汇入同一活动 Runner，而不是各自启动一条互相竞争的模型循环。

第二点避免同一 Session 同时出现两套独立编排，但它不等于跨进程的持久任务所有权。进程退出后，活动 Runner 本身不会随历史一起恢复。

## 6. 一次循环迭代实际做什么

把实现细节压缩后，当前默认 Loop 每次迭代可以读成：

```text
1. 标记 Session 正在运行
2. 重新读取当前可用的 Session History
3. 找到最新 User、Assistant 和特殊任务状态
4. 先判断是否已经满足停止条件
5. 处理必要的特殊分支，例如 Compaction
6. 创建本轮 Assistant Message
7. 组织 Context 与可见 Tools
8. 发起 Provider Request；可重试错误可能在同一 Processor context 内再次请求
9. 消费响应并结算 Text、Reasoning、Tool 或 Error
10. 返回 continue、compact 或 stop
11. 若继续，则回到第 1 步
```

这不是源码的逐行翻译，而是帮助学习者看清职责的架构投影。

### 6.1 为什么每轮重新读取历史

OpenCode 不是只在内存中的 `messages` 数组后追加结果。当前 Loop 会调用 `MessageV2.filterCompactedEffect(sessionID)`，重新取得投影后的历史。

这让上一轮已结算的读取结果、运行中进入的新用户消息和 Compaction 结果能够成为下一轮输入。

在我们的学习场景中：

```text
第一轮前：历史里只有“请带我学习 Harness”

read README 完成后：历史里出现 Tool Call 与 Tool Result

第二轮前：Loop 重新读取历史
          模型现在才看见 README 的实际内容
```

“重新读取”不表示数据库中的所有原始记录都逐字发送给模型。历史还会经历选择、转换和裁剪；这些内容在[第 08 篇](./08_Context_Architecture.md)展开。

### 6.2 为什么先创建 Assistant Message

OpenCode 在发起 Provider Request 前创建本轮 Assistant Message。随后到达的文本、推理、工具状态、用量与错误，都可以归属到这次 Assistant 输出。

这也是为什么 Provider Turn、Assistant Message 和用户请求不能随意互换：它们处于不同层次。一次用户请求可能产生多条 Assistant Message；一次 Retry 又可能在同一条 Assistant Message 的处理上下文中发起新的物理 Provider Request。

## 7. Tool Result 怎样触发 Continuation

延续（Continuation）指当前任务在一次 Provider Turn 后仍需继续请求模型。

在默认 AI SDK 路径中，普通本地 Tool 会在当前 `llm.stream` 的工具执行阶段完成。随后：

1. LLM Runtime 将执行结果归一化为 Tool Result 或 Tool Error 事件。
2. `SessionProcessor` 把 Tool Part 从 pending/running 结算为 completed/error。
3. 完整 Tool Part 被保存，形成后续模型可见的观察结果。
4. Processor 通常向外层 Loop 返回 `continue`。
5. Loop 回到顶部，重新读取历史，再创建下一次模型请求。

换句话说，下一轮不是“现在才执行上一个工具”，而是“把已经执行并结算的结果交给模型继续判断”。

### 7.1 用学习场景走一遍

```text
用户目标：从零学习 Harness，只读取和解释

Provider Turn 1
模型提议：read Opencode_Harness/README.md
Harness：校验 -> Permission -> Tool executor -> 保存结果

Provider Turn 2
模型观察：README 给出了 06-12 的阅读顺序
模型提议：read 项目规则文件
Harness：校验 -> Permission -> Tool executor -> 保存结果

Provider Turn 3
模型观察：教程入口与规则都已获得
模型输出：建议先读总览，再观察 Loop 和 Context
Harness：顶部检查确认没有待 continuation 的本地 Tool
停止
```

工具失败也可能成为观察。例如读取一个不存在的规则文件时，模型可以在下一轮改为搜索可用的 `AGENTS.md`。失败不必然结束整个 Session；是否继续取决于错误类型、Processor 状态和后续停止判断。

## 8. 模型决定什么，Harness 控制什么

Agent Loop 同时包含概率性判断和确定性控制。

| 问题 | 主要负责者 | 性质 |
| --- | --- | --- |
| 下一步先读 README 还是先找规则 | Model | 概率性提议 |
| 哪些工具会出现在本轮请求中 | Harness | 按 Agent、Session 与本次输入规则过滤 |
| Tool Call 参数是否符合 schema | Harness | 确定性校验 |
| 是否需要向用户请求批准 | Harness + User | Permission 流程 |
| 文件是否真的被读取 | Tool Runtime | 实际 I/O，可能失败 |
| Tool Result 如何结算和保存 | Harness | 状态转换与记录 |
| 是否发起下一 Provider Turn | Harness | Loop 与 terminal check |
| 下一轮能否正确理解读取结果 | Model | 概率性判断 |

Harness 可以限制模型能够提议和执行的范围，也可以保存结果、阻止危险动作和结束循环。它不能保证模型一定选择最好的教程入口，或一定正确总结读到的规则。

## 9. `continue`、`compact` 与 `stop`

`SessionProcessor.process` 把一次响应处理的结论压缩为三个方向：

| 结果 | 外层 Loop 的动作 |
| --- | --- |
| `continue` | 回到顶部，重载历史并重新判断 |
| `compact` | 创建或进入上下文压缩流程，再按结果推进 |
| `stop` | 结束当前 Loop |

一个容易反直觉的细节是：普通最终文本通常也先得到 `continue`。Loop 回到顶部后，顶部终止检查发现最新 Assistant 已经完成、对应最新 User Message，而且没有需要继续处理的本地 Tool Part，于是不再请求模型，直接退出。

这种设计让“是否结束”不只依赖 Provider 的一个 finish reason，而能同时检查历史、Tool Part 和最新用户输入。

## 10. Retry：重新请求，不是重新开始整个 Loop

重试（Retry）处理的是一次模型请求遇到可重试错误的情况，例如某些临时 Provider 错误。

当前默认路径中，Retry 位于同一个 `SessionProcessor` 处理上下文内：

```text
同一 Assistant Message
同一份 streamInput
同一 Processor context
    |
    +-> Provider Request attempt 1：可重试失败
    +-> 等待/backoff
    +-> Provider Request attempt 2：再次请求
```

它不会先回到 `SessionPrompt.run` 顶部，不会重新加载最新 Session History，也不会创建一条新的 Assistant Message。当前策略最多安排 5 次 retry，并可依据 Provider hint 或退避策略等待。

按本系列“一次真实 Provider Request attempt 及其响应或错误投影”的定义，每次 Retry attempt 都是新的 Provider Turn；只是这些 turns 共享同一个 Assistant/Processor 上下文。

Retry 也不是数据库事务回滚。失败前已经投影的部分输出可能留下状态，因此不能把它理解成“一切从未发生过”。Context Overflow 也不走这条通用 Retry，而是进入 Compaction 或错误停止分支。

## 11. Interrupt：停止继续，不等于撤销已经发生的事

中断（Interrupt）由当前 TUI 的 interrupt 动作进入兼容 `session.abort`，最终调用 `SessionPrompt.cancel -> SessionRunState.cancel`。

取消后，Processor 会尽力：

- 保存已经累计的 Text 或 Reasoning；
- 把未完成 Tool 标记为 interrupted error；
- 为 Assistant Message 记录中止错误；
- 结束当前活动 Runner。

但中断没有时间机器。如果读取已经完成，它不会“取消已经读到的信息”；如果未来示例涉及写文件、启动 Shell 或调用外部服务，中断也不保证撤销已经发生的副作用。

这正是本系列使用低风险读取场景的原因：初学者可以观察多轮 Loop，而不需要先承担写操作和外部副作用。

## 12. OpenCode 何时停止

当前默认 Loop 的停止不是单一开关，主要边界包括：

- 顶部检查确认最新 Assistant 已完成，且没有需要 continuation 的普通本地 Tool Part。
- Processor 因 blocked 或 Assistant Error 返回 `stop`。
- Content Filter 或 Structured Output Error 结束当前分支。
- Structured Output Tool 已经得到目标对象。
- Compaction processor 决定停止。
- 用户 Interrupt 终止当前 Runner，并进行尽力结算。

下面三个值都不能单独证明 Loop 应当停止：

### 12.1 Provider finish reason

某些 Provider 可能给出 `stop`，同时响应里仍有需要把结果回传给模型的本地 Tool Part。OpenCode 会综合检查 Tool 状态，而不是机械地只看 finish reason。

### 12.2 Todo 状态

Todo 是 Session 中可观察的结构化清单，不是 Loop 调度器。Todo 全部完成不会直接触发停止，Todo 仍为 pending 也不会强制下一次 Provider Turn。它由[第 11 篇](./11_Agent_Specialization_and_Collaboration.md)详细解释。

### 12.3 当前旧路径的 `agent.steps`

达到旧 `agent.steps` 阈值时，当前默认路径会向模型追加最后步骤提醒，但不会仅凭这个值直接退出，也不会因此自动移除 Tools。它不是“最多 N 次模型请求”的硬上限。

## 13. 防止无效循环的边界

循环能够持续，不表示应该无限持续。

当前 Processor 对同一个 Assistant 最近三个 Part 做重复调用检查。如果它们是同名 Tool、输入序列化后完全相同且都不处于 pending，就请求 `doom_loop` Permission。默认行为是询问用户，而不是自动判定失败或直接永久禁用工具。

它是一个具体保护条件，不是对所有无效循环的数学证明：

- 参数稍有不同，不会命中“完全相同”。
- Text/Reasoning 或并行 Tool Part 可能改变最近 Part 的排列。
- 用户或规则仍可能允许继续。

因此，可靠停止仍依赖模型判断、Permission、错误处理、上下文窗口和 Harness 的多种控制条件共同作用。

## 14. 当前实现的几个重要边界

### 14.1 可继续读取，不等于崩溃后自动续跑

Session History 可以持久化并在下一次 Loop 重载，但活动 Runner、流式累积器和当前 Tool 协调状态属于进程内状态。进程崩溃后，历史可重新读取，不代表未完成的 Provider 或 Tool 工作会被安全地自动重试。

### 14.2 Permission 拒绝不等于 OS Sandbox

Permission 可以阻止 OpenCode 受管的工具动作，不能降低进程在操作系统中的真实权限。完整边界见[第 09 篇](./09_Tools_and_Permission.md)。

### 14.3 `Plan`、`Todo`、`Task` 不是 Agent Loop 本身

它们可以帮助组织复杂任务，但基础 Loop 即使没有 Todo 或 Subagent 也能通过 Tool Result continuation 工作。它们的职责留到[第 11 篇](./11_Agent_Specialization_and_Collaboration.md)。

## 15. 动手观察：用只读任务看见多轮 Loop

可以在一个熟悉的教程目录中启动新 Session，输入类似请求：

> 我正在从零学习 OpenCode Harness。请先读取本目录的 README 和适用的项目规则，再告诉我 06、07、08 三篇分别解决什么问题。只做读取，不修改文件，不运行 Shell 命令；如果入口不存在，请先说明再调整查找方式。

观察时不要只看最终答案，重点看执行轨迹：

1. 模型第一次提出了哪个 Tool Call？
2. Tool 是否完成，还是返回错误或 Permission 请求？
3. 结果出现后，是否产生了下一条 Assistant Message 或新的工具调用？
4. 最终文本前一共发生了多少次真实读取？
5. 最后一轮为什么没有继续调用工具？

你可能观察到不同读取顺序，因为模型决策具有概率性。学习目标不是背下一条固定轨迹，而是识别稳定结构：**每次新观察都先经过 Tool Result 结算，再由下一轮模型决定如何利用。**

如果想观察 Interrupt，可以在一个预计会读取多个文件、但仍然只读的任务中主动中断。中断后检查已完成 Tool 与未完成 Tool 的状态差异，不要把 UI 停止更新直接等同于所有底层副作用已经撤销。

## 16. 常见误解

### “Agent Loop 就是模型一直思考”

不是。Loop 是 OpenCode 在模型外运行的控制流；模型每次只处理当前 Provider Request。

### “Tool Call 出现后，下一轮才执行工具”

默认本地路径中，工具通常已在当前流处理阶段执行和结算；下一轮负责读取结果并继续判断。

### “模型返回 stop，Session 就一定停止”

不一定。Harness 还会检查本地 Tool Part、最新 User Message、错误和 Compaction 状态。

### “Retry 会重新获取所有最新上下文”

不会。当前通用 Retry 复用同一个 Processor context 和 `streamInput`，不先返回外层 Loop 重载历史。

### “Interrupt 会回滚任务”

不会。它中断当前执行并尽力结算，但不承诺撤销已发生的外部副作用。

### “设置 steps 就不会循环过多”

当前默认旧路径不是这样。`agent.steps` 主要形成模型可见提醒，真正停止仍要依赖其他边界。

## 17. 关于 native V2 的一句版本说明

固定源码中，native V2 已有独立的 durable prompt admission、`SessionRunner` 和 Tool continuation 路径，但当前默认 TUI 普通消息仍使用本文描述的兼容 `SessionPrompt` Loop。V2 对输入 promotion 和 step allowance 有不同、更显式的设计，同时一般 Provider Retry、Doom Loop 和部分能力仍未达到当前路径的等价覆盖。

本篇不展开两套循环的完整迁移矩阵；统一放在[第 12 篇](./12_Runtime_Boundary.md)的架构演进部分与附录中。


## 18. 关键源码入口

以下路径均以固定源码 `0e3474509aa5ad16afcf9c439785514d6443c6af` 为基线：

| 主题 | 文件 | 函数或导出符号 |
| --- | --- | --- |
| 默认 TUI 提交 | `packages/tui/src/component/prompt/index.tsx` | `submitInner` |
| 兼容 Prompt Handler | `packages/opencode/src/server/routes/instance/httpapi/handlers/session.ts` | `SessionHttpApi.prompt` |
| User Message 与外层 Loop | `packages/opencode/src/session/prompt.ts` | `SessionPrompt.prompt`、`run`、`loop` |
| 同 Session 运行协调与取消 | `packages/opencode/src/session/run-state.ts` | `SessionRunState.ensureRunning`、`cancel` |
| Provider 响应与 continuation | `packages/opencode/src/session/processor.ts` | `SessionProcessor.handleEvent`、`process`、`cleanup` |
| Provider Retry | `packages/opencode/src/session/retry.ts` | `SessionRetry.retryable`、`policy` |
| 最终请求与 Tool 可见性 | `packages/opencode/src/session/llm/request.ts` | `LLMRequestPrep.prepare`、`resolveTools` |
| 历史选择与模型消息转换 | `packages/opencode/src/session/message-v2.ts` | `filterCompactedEffect`、`toModelMessagesEffect` |

可用于核对行为的代表性测试：

- `packages/opencode/test/session/prompt.test.ts`：Tool continuation、finish 与 Tool Part、并发 Loop、Interrupt、运行中新 Prompt。
- `packages/opencode/test/session/processor-effect.test.ts`：Tool Part 结算与 Processor 结果。
- `packages/opencode/test/session/retry.test.ts`：重试状态、次数与停止策略。
- `packages/opencode/test/agent/agent.test.ts`：默认 `doom_loop` Permission 与 Agent steps 配置。

---

上一篇：[06 Harness 总览](./06_Harness.md)
下一篇：[08 Context Architecture：模型每一轮看见什么](./08_Context_Architecture.md)

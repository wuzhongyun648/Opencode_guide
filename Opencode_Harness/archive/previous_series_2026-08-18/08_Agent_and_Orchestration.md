# Agent 与 Orchestration：OpenCode 如何持续完成一次代码修改

> 核对日期：**2026-08-18**  
> 固定源码：`0e3474509aa5ad16afcf9c439785514d6443c6af`（下文简称“固定 SHA”）  
> **当前**：当前 TUI 普通消息实际使用的兼容 Session API、`SessionPrompt` 旧编排及其复用的新基础设施。  
> **native V2**：由 `V2Session.prompt`、`SessionExecution` 和 `SessionRunner` 组成的独立 Effect-native 流程；“V2”是迁移期架构标签，不等同于产品版本号，也不表示已接管当前 TUI。  
> **验收状态**：任务6最终交叉审计尚未完成；任务8按用户指示跳过，尚未验收。

阅读顺序：[06 总览](./06_Harness.md) -> [07 术语](./07_OpenCode_Runtime_Terminology.md) -> **08 Agent 与 Orchestration**  
上一篇：[07_OpenCode_Runtime_Terminology.md](./07_OpenCode_Runtime_Terminology.md)；06 是 Harness 总览。  
后续阅读：[09_Context_and_Persistence.md](./09_Context_and_Persistence.md)

本文用一个场景贯穿全章：

> 修复 `src/cache.ts` 的并发缓存 Bug，先调查相关调用，修改实现并运行测试。

本文涉及代理配置（Agent）、模型（Model）、提供商轮次（Provider Turn）、会话循环（Session Loop）、待办项（Todo）、任务工具（Task Tool）、子代理（Subagent）、重试（Retry）和中断（Interrupt）。这些术语描述的是 Harness 中不同的配置、控制流和状态对象，不能互换。

重点不是模型怎样“想”，而是 OpenCode 怎样选择行为配置和模型、执行模型提出的动作、保存结果，并决定继续、重试、中断或停止。

## 1. Agent、Model、Orchestration 与 Provider Turn

### 1.1 Agent 不是 Model

代理配置（Agent）定义一次执行采用的行为边界。当前 `Agent.Info` 包含身份、模式、提示、Model 偏好、Variant、采样参数、权限（Permission）和 `steps` 等字段。它不是 Model 的副本，也不只是系统提示词（System Prompt）的别名。[S1]

模型（Model）是生成下一段文本或工具调用（Tool Call）的推理服务。同一个 Model 可以供多个 Agent 使用；一个 Agent 也可以通过配置偏好某个 Model。面对缓存 Bug，`build` 和 `plan` 即使使用同一个 Model，仍会因为可见 Tool、Permission 和提醒不同而表现出不同的可执行范围。

### 1.2 Orchestration 是 Harness 的控制流

编排（Orchestration）是围绕 Model 建立的确定性流程，包括：

1. 读取会话历史（Session History）。
2. 选择 Agent 和 Model。
3. 组装 System、Messages 和 Tools。
4. 请求 Provider，并处理流式 Text、Reasoning、Tool Call 和 Usage。
5. 校验、授权并执行 Tool，将 raw/domain Tool Result 结算为 durable terminal state 和 Model Tool Output。
6. 判断 continuation、Compaction、Retry、Interrupt 或停止。

当前默认路径的主 Orchestrator 是 `SessionPrompt.run` 中显式的 `while (true)` 会话循环，而不是模型内部的“思考循环”，也不是 AI SDK 隐式代办的完整 Agent Loop。[S4]

### 1.3 Provider Turn 小于一次用户请求

提供商轮次（Provider Turn）严格指 Harness 对 Provider 的一次请求及其响应投影。一次响应可以流式产生多种事件；一次用户请求则可能包含多个 Provider Turn。

例如，修复缓存 Bug 可能经历：

```text
用户请求
-> Provider Turn 1：模型调用 grep/read
-> Harness 执行并保存终态与 Model Tool Output
-> Provider Turn 2：模型调用 edit/bash
-> Harness 执行并保存终态与 Model Tool Output
-> Provider Turn 3：模型给出最终说明
```

Retry 会在同一个 Assistant Message 和 `SessionProcessor` context 内再次发起 provider request attempt。每个 attempt 都是独立的 Provider 请求，其响应投影分别构成一个 Provider Turn；多个 retry attempts 不能合称为同一个 Provider Turn。[S5][S12]

## 2. 当前默认 Agent 与 Model 选择

### 2.1 当前 TUI 仍走兼容 `SessionPrompt`

当前 TUI 的普通消息分支调用 `sdk.client.session.prompt(...)`，并显式提交 `agent`、`model` 和 `variant`。兼容 HTTP Handler 随后进入 `SessionPrompt.prompt`，而不是 native V2 的 `V2Session.prompt -> SessionExecution -> SessionRunner`。[S2][S3]

这条主线很重要：仓库中已经存在 V2 目录、V2 Handler 和 V2 测试，不能据此推断当前 TUI 已由 V2 编排。

### 2.2 Agent 的默认规则

创建 User Message 时，有显式 `input.agent` 就按名称获取；没有则调用 `Agent.defaultInfo()`。显式名称不存在会报错，不会静默退回默认 Agent。[S3]

未显式选择时：

1. 如果配置了 `default_agent`，它必须存在、不能是纯 Subagent、不能 hidden。
2. 否则选择第一个非 Subagent 且非 hidden 的 Agent。
3. 内置 Agent 的构造顺序使无自定义配置时通常选择 `build`。[S1]

内置角色的关键差异如下：

| Agent                            | 模式           | 当前默认能力轮廓                                                                  |
| -------------------------------- | -------------- | --------------------------------------------------------------------------------- |
| `build`                          | primary        | 默认可执行工具；允许 `question` 和 `plan_enter`，具体敏感路径仍可能 ask           |
| `plan`                           | primary        | 默认拒绝一般编辑，只放行指定 plan 文件；允许 `plan_exit`，默认拒绝 `task:general` |
| `general`                        | subagent       | 通用子任务 Agent；默认禁用 `todowrite`                                            |
| `explore`                        | subagent       | 面向代码搜索与调查，采用更窄的只读式权限集合                                      |
| `compaction`、`title`、`summary` | hidden primary | 承担内部工作，不是供用户任意切换的普通人格                                        |

用户配置可以覆盖内置规则。因此“Plan 永远不能编辑或委派”并不准确；准确说法是“这是固定 SHA 下的默认权限轮廓”。[S1][T1]

在缓存 Bug 场景中，选择 `build` 意味着模型可以提出读、改、测；选择默认 `plan` 时，一般代码编辑会被 Harness 拒绝。差异来自 Agent policy，不是 Model 被确定性地切换成了“只规划模式”。

### 2.3 Model 与 Variant 的优先级

新 User Message 的 Model 选择顺序是：

```text
input.model
-> agent.model
-> Session 当前 model
-> 最近一个带 model 的 User Message
-> Provider.defaultModel()
```

`input.variant` 优先；否则只有最终 Model 与 Agent 配置 Model 相同、且该 Variant 确实存在时，才采用 `agent.variant`。最终选择写入 User Message，并在变化时更新 Session 的 Agent/Model 状态。[S3]

外层 Loop 为新的 Assistant Message/Processor context 组装首次 provider request attempt 时，会从最新 User Message 读取 Agent 和 Model；同一 context 中的 retry attempts 则复用既有 `streamInput`。因此同一 User Message 引发的 Tool continuation 通常保持原选择；活跃运行中进入的新 User Message 则可能改变后续 context 使用的选择。[S4][S5][T4]

## 3. Session Loop 与 Continuation

### 3.1 从 User Message 到第一轮

`SessionPrompt.prompt` 先创建并持久化 User Message，再进入 `SessionPrompt.loop`。同一 Session 的并发 loop 调用由 `SessionRunState.ensureRunning` 汇入同一个 Runner，不会各自启动一套 Provider 循环。[S3][S11]

每一轮顶部都会重载经过 Compaction 过滤的 Session History，而不是只维护一个不断追加的内存数组：[S4]

```text
重载 durable history
-> 找到最新 User/Assistant/特殊任务
-> terminal check
-> 处理 Subtask/Compaction，或创建 Assistant Message
-> 组装 Context 与 Tools
-> 发起一次 provider request attempt（一个 Provider Turn）
-> Retry 时可在同一 Processor context 发起后续 attempts
-> continue / compact / stop
```

这使缓存调查产生的 `grep` 结果、修改后的测试输出、Compaction 结果以及运行中新到达的 User Message，都能在下一轮重新进入模型输入。

### 3.2 已结算工具输出触发下一轮的机制

默认 AI SDK 路径在当前 `llm.stream` 内执行普通本地 Tool。`SessionProcessor` 将 raw/domain Tool Result 结算为 completed/error Tool Part，并保存下一轮可见的模型工具输出（Model Tool Output）；外层 Loop 的下一轮不是才执行 Tool，而是重载这些持久化终态和模型工具输出，再发起新的 Provider Turn。[S5]

`SessionProcessor.process` 返回三类结果：

| 返回值     | 外层行为                               |
| ---------- | -------------------------------------- |
| `continue` | 回到顶部，重载历史并重新判断           |
| `compact`  | 创建 Compaction 标记，随后继续相应流程 |
| `stop`     | 结束当前 Loop                          |

普通最终文本通常也会先得到 `continue`。Loop 回到顶部后，terminal check 看到最新 Assistant 已完成、对应最新 User Message，且没有需要 continuation 的本地 Tool Part，才无额外 Provider 请求地退出。[S4][S5]

因此不能只看 Provider 的 finish reason。某些 Provider 即使返回 `stop`，同一个 Assistant 中只要还有需要回传结果的本地 Tool Part，当前 Loop 仍会继续。[T2][T3]

### 3.3 修复 Bug 时旧 Loop 一次运行到空闲

```text
用户：修复并发缓存 Bug
-> 写入 User Message（Agent/Model 固定在该消息上）
-> Assistant 1 / Provider Turn 1：提出 read、grep
-> Harness：授权、执行、保存结果
-> Assistant 2 / Provider Turn 2：提出 edit、测试命令
-> Harness：执行修改与测试，保存结果
-> Assistant 3 / Provider Turn 3：返回总结，不再调用本地 Tool
-> 顶部 terminal check：停止
```

这里的“一次运行到空闲”只是对旧 Loop 行为的非正式描述：当前可继续的工作被处理到空闲边界。它不代表官方的 Session Drain，不会创建持久化 Job，也不等于一条 Message。

## 4. 模型决策与 Harness 控制

| 决策事项                             | 决策方                         | 性质                                     |
| ------------------------------------ | ------------------------------ | ---------------------------------------- |
| 下一步读文件、修改还是回答           | Model                          | 概率性生成 Text 或 Tool Call             |
| 哪些 Tool 出现在请求中               | Harness                        | 按 Agent、Session、User Permission 过滤  |
| Tool 参数是否合法                    | Harness                        | 按 schema 校验                           |
| 是否需要用户批准                     | Harness 依据规则发起，用户作答 | Permission 生命周期                      |
| 文件是否真的被修改、测试是否真的运行 | Tool Runtime                   | 执行副作用，可能失败                     |
| 工具终态与 Model Tool Output 怎样保存 | Harness                        | Processor、Event 和持久化投影            |
| 是否发起下一 Provider Turn           | Harness                        | Loop、Processor Result 和 terminal check |
| 下一轮是否正确利用结果               | Model                          | 概率性，Harness 不保证判断正确           |

在缓存 Bug 场景中，模型可以选择先搜索锁的调用点，但它不能直接“宣告文件已修改”。真正的读取、编辑和测试由 Tool Runtime 执行；Harness 还会决定该工具是否可见、是否允许及结果是否足以触发 continuation。[S4][S5]

## 5. Todo：结构化清单，不是调度器

`todowrite` 接收一整个 Todo 数组，通过 Permission 后替换当前 Session 的有序清单并发布更新。它不会创建 Provider Turn、启动后台任务，或强制模型完成 pending 项。[S6]

缓存 Bug 的 Todo 可以是：

```text
1. 找到缓存写入与失效路径
2. 复现并发失败
3. 修改同步策略
4. 运行定向测试和相关测试
```

它的价值是把模型报告的计划转成 Session 级可观察状态。即使所有 Todo 都标为 completed，主循环仍按 Assistant finish、Tool Part 和 Processor Result 停止；反过来，Todo 仍为 pending 也不会自动强制 Loop 继续。

必须区分：

| 名称      | 实际含义                                              |
| --------- | ----------------------------------------------------- |
| 普通 Tool | 一次受控能力调用，例如 `read`、`grep`、`edit`、`bash` |
| Todo      | Session 中的结构化清单，不执行工作                    |
| Task Tool | 一个特殊委派 Tool，用来启动或恢复子 Session           |
| Subagent  | 在子 Session 中按另一份 Agent 配置运行的执行者        |

当前旧 `TodoWriteTool.execute -> Todo.update` 未找到直接行为测试；本章不把未运行或未存在的覆盖写成通过。native V2 的 `todowrite` 则有独立测试并在任务 7 中实际运行。[T9]

## 6. Task Tool、Subagent 与父子 Session

### 6.1 Task Tool 的委派创建流程

如果父模型认为缓存调用链调查适合交给 `explore`，它生成的是一个 `task` Tool Call。`TaskTool` 随后：

1. 检查 Subagent 深度和 `task:<subagent_type>` Permission。
2. 读取指定 Subagent 配置。
3. 创建带 `parentID` 的新 Session；若给出可读取的 `task_id`，则恢复该 Session。
4. 用任务 prompt 创建子 User Message，并再次调用独立的 Session Prompt 编排。
5. 前台等待子任务完成，取子 Session 最后一个 Text Part。
6. 将文本包装为父 Assistant 中 `task` Tool Part 的 output。
7. 父 Loop continuation 后，父模型才从已结算的 Model Tool Output 看到该结果。[S7]

所以 Subagent 不是父 Session 中途“换人格”，也不是远程 A2A Agent。它有独立 Session ID 和独立 Session History。

### 6.2 子 Session 得到什么

新子 Session 不自动复制父 Session History。父模型必须在 Task prompt 中提供所需背景，例如缓存文件、怀疑的竞态位置和调查目标。父子之间主要通过以下信息连接：[S7]

- 子 Session 的 `parentID`。
- Task metadata。
- 显式任务 prompt。
- 返回父 Session 的 Tool output。

子 Model 优先使用 Subagent 自己的 `model`；没有时继承发出 Task Call 的父 Assistant Model。只有继承 Model 时才继承父 Assistant Variant。[S7]

子 Session 权限也不是简单复制父 Agent。当前实现综合 Subagent 权限与派生的子 Session 权限；父 Session 持久化的 deny 可形成硬上限，但父 `plan` Agent 自身的 edit deny 不一定自动限制 `general` Subagent。[S8][T6]

### 6.3 深度、前台与实验性后台模式

默认 `subagent_depth ?? 1` 阻止 Subagent 再启动 Subagent；提高配置才允许嵌套。这个值限制 Task 委派深度，不是 Session Loop 步数。[S7][T5]

前台 Task 等待子任务结束再返回。`background: true` 则需要 `OPENCODE_EXPERIMENTAL_BACKGROUND_SUBAGENTS=true`；它会先返回 running 结果，结束后向父 Session 注入 synthetic User Message。该能力在固定 SHA 下属于实验性，不应按默认能力介绍。[S7]

还有一个可信度边界：恢复 `task_id` 时，固定 SHA 的实现读取该 Session，但没有在相邻代码中显式验证它确实属于当前父 Session。本章只把它标为待安全审计的静态风险；任务 7 没有运行跨父所有权实验。[S7]

## 7. Retry、Interrupt、Doom Loop、Steps 与停止

### 7.1 Retry：同一 Assistant/Processor context 中的多次请求

`Effect.retry(SessionRetry.policy(...))` 位于同一个 `SessionProcessor` Handle 内。可重试错误按 Provider hint 或指数退避策略处理，最多安排 5 次 retry；Context Overflow 不走这条通用 Retry。[S5][S12][T7]

Retry 不会先回到 `SessionPrompt.run` 顶部，不会重载 Session History，也不会创建新的 Assistant Message。它在同一个 `SessionProcessor` context 中重新执行 `llm.stream(streamInput)`。这意味着：

- 从 Assistant Message 和 Processor 生命周期看，这些 provider request attempts 共享同一 context。
- 每次 attempt 都是一次独立 Provider 请求及响应投影，因此分别属于不同的 Provider Turn。
- 失败前已经投影的部分流事件未必“无痕”，所以不能称为数据库层完全回滚。

### 7.2 Interrupt：取消运行，不回滚副作用

当前 TUI interrupt 最终进入 `SessionPrompt.cancel -> SessionRunState.cancel`。Processor cleanup 会尽力保存部分 Text/Reasoning/Patch，把未完成 Tool 标记为 interrupted error，并完成 Assistant Message。[S5][S9][S10][S11][T4]

如果缓存测试命令已经启动、文件写入已经发生或外部服务已经接收请求，Interrupt 不能保证撤销这些副作用。持久化一个 interrupted error 只是结算状态，不等同于事务回滚。

### 7.3 Doom Loop：三次相同调用后询问

当前 Processor 检查同一个 Assistant 最近 3 个 Part。若它们都是同名 Tool、非 pending，且 `JSON.stringify(input)` 完全相同，就请求 `doom_loop` Permission；内置默认规则是 ask。[S1][S5]

这不是自动判定任务失败，也不是直接 `break`。规则或用户可以 allow/deny；即使允许，也没有全局永久封禁。固定 SHA 未找到直接覆盖“三次相同 Tool Call 触发 ask”的测试，任务 7 也没有运行该实验，因此本文只报告源码行为，不声称实测通过。

### 7.4 旧 `agent.steps` 不是硬上限

当前兼容 Loop 每轮先递增 `step`。当 `step >= agent.steps` 时，它只向 Model Messages 追加 `MAX_STEPS_PROMPT`；代码没有因此清空 Tools、设置 `toolChoice: none` 或直接退出。[S4]

任务 7 的隔离 Fake Provider 实验进一步验证：配置 `build.steps: 1` 后，Provider 连续返回两个参数不同的 `glob` Tool Call，再返回最终文本；实际发生 3 次 Provider 调用并正常停止，三次请求均仍携带非空 `tools`。[E1]

准确结论是：旧 `agent.steps` 是模型可见的最后步骤提醒，不是“最多 N 次 Provider 调用”的硬保证。该实验只证明至少可到 3 次调用，不证明可以无限循环。

### 7.5 停止边界

当前 Loop 主要在以下边界停止：[S4][S5]

- 顶部 terminal check 确认最终 Assistant，且没有需要 continuation 的本地 Tool Part。
- Processor 因 blocked 或 Assistant Error 返回 `stop`。
- Content Filter 或 Structured Output Error 结束当前分支。
- Structured Output Tool 已得到目标对象。
- Compaction processor 决定停止。
- Interrupt 终止当前 Runner 并执行尽力结算。

因此，停止是 Harness 综合历史、Tool Part、错误和控制状态后的结论，不是单独由 Provider finish reason、Todo 状态或 `agent.steps` 决定。

## 8. Native V2 的独立流程

### 8.1 已接线，但未接管 TUI

native V2 HTTP Handler 已调用 `V2Session.prompt`，当前 executable 也为它组合了 `SessionExecutionLocal`。这证明 V2 是可达实现，不只是类型或计划文档。[V1][V2][V3]

但当前 TUI 普通消息仍走第 2.1 节的兼容入口。准确状态是“两条流程共存，TUI 默认仍用兼容 `SessionPrompt`”，不是“V2 尚无实现”，也不是“V2 已全面上线”。

### 8.2 Admission、Promotion 与 Runner

V2 不调用旧 `SessionPrompt.loop`，其主线是：[V1][V4]

```text
native V2 HTTP prompt
-> V2Session.prompt
-> SessionInput.admit：写入 durable session_input
-> resume !== false 时 SessionExecution.wake
-> SessionRunner 在安全边界 promotion steer/queue
-> 重载 Session History
-> runTurnAttempt：一次显式 llm.stream
-> durable Tool Call + 本地 Tool settlement
-> 需要时重载历史并 continuation
```

`resume:false` 可以只 admission 而不唤醒执行。steer 在当前执行仍需 continuation 时于下一安全 Provider Turn 边界提升；queue 要等当前 continuation 将结束时再按顺序提升。新输入被 promotion 后，Agent 的步骤 allowance 重置。[V1][V4][T10]

### 8.3 V2 `steps` 更强，但不是整个 Session 的总步数

当 `currentStep >= agent.steps`，V2 不仅追加 `MAX_STEPS_PROMPT`，还不物化 Tools，并设置 `toolChoice: none`。如果 Provider 仍违规返回本地 Tool Call，Runner 会失败结算它，且不建立 Tool continuation。[V4][T11]

所以 V2 `steps` 限制的是一次输入批次引发的工具型 Provider continuation，并保留最后一个 text-only Turn。新 steer/queue 输入可以开始新的 allowance，不能简写成“整个 Session 最多 N 步”。

### 8.4 V2 已有与缺失能力

V2 已实现独立 Agent Registry、build/plan 的部分权限轮廓、基于已结算 Model Tool Output 的 continuation、steer/queue、process-local Interrupt 和 `todowrite`。[V4][V5][V6][V8][T9]

但 parity 仍不完整：

- V2 内置 Tool 列表尚未 port `task` 和 `plan_exit`；父子 Session 的 Task/Subagent 编排、继承和结果回传没有实现。[V5]
- V2 没有当前旧路径的一般 Provider Retry，也没有 Doom Loop 等价保护。[V9][P1]
- V2 Agent 配置可保存 Model/Variant/Request，但当前 Runner 的 Model resolution 只按 Session/Catalog 选择，Agent-local request policy 仍是 partial。[V4][V7][P1]
- V2 Interrupt 目前是 process-local；clustered ownership、跨进程 fencing 和 post-crash continuation recovery 仍未完成。[V8][P1]
- 一次 Context Overflow 后的受限 Compaction 重建不是通用 Retry，不能用它填补 Retry parity。[V4]

## 9. 当前与 V2 的状态对照

| 能力                       | 当前默认兼容编排                                | native V2                              | 准确含义                                           |
| -------------------------- | ----------------------------------------------- | -------------------------------------- | -------------------------------------------------- |
| TUI 普通消息               | 默认进入 `SessionPrompt`                        | 独立 HTTP 入口已接线，TUI 未使用       | 当前操作体验仍由兼容 Loop 主导                     |
| Agent Registry、build/plan | 已实现                                          | 已实现一部分工作流轮廓                 | V2 的 Plan/Build parity 不完整                     |
| Agent-local Model/Request  | 已应用                                          | partial                                | V2 Runner 当前主要按 Session/Catalog resolve Model |
| 多 Provider Turn           | `SessionPrompt.run`                             | `SessionRunner.run`                    | 两边均为显式循环，互不桥接                         |
| 已结算工具输出 continuation | 已实现                                          | 已实现                                 | 下一轮都会重载 terminal state / Model Tool Output |
| Todo                       | 已实现                                          | 已实现                                 | 都是清单，不是调度器                               |
| Task/Subagent              | 已实现；后台模式实验性                          | missing/planned                        | V2 有 subagent 数据模式不等于已能委派              |
| Provider Retry             | 同一 Assistant/Processor context 内多次 attempt | missing/planned                        | 各 attempt 分别是一个 Provider Turn                |
| Interrupt                  | process-local Runner + cleanup                  | process-local Coordinator + settlement | 两者都不能宣称跨进程恢复完备                       |
| `agent.steps`              | 仅追加提醒，不硬禁 Tool                         | final Turn 禁用 Tools                  | 两条路径语义不同                                   |
| Doom Loop                  | 三次相同调用后 Permission ask                   | missing/planned                        | 旧路径也缺直接行为测试                             |

保留这些迁移标签，是为了避免两个相反误解：不要把兼容运行时写成已消失的历史 V1，也不要把 native V2 写成已完整替代当前产品路径。

## 10. 关键源码与实测证据

### 10.1 当前默认路径源码

| ID  | 路径                                                                                                                  | 函数/符号                                                   | 行号                   | SHA                                        |
| --- | --------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------- | ---------------------- | ------------------------------------------ |
| S1  | `packages/opencode/src/agent/agent.ts`                                                                                | `Info`、Agent Layer、`defaultInfo`                          | `35-55`, `98-340`      | `0e3474509aa5ad16afcf9c439785514d6443c6af` |
| S2  | `packages/tui/src/component/prompt/index.tsx`                                                                         | `submitInner()` 普通消息分支                                | `1092-1146`            | `0e3474509aa5ad16afcf9c439785514d6443c6af` |
| S3  | `packages/opencode/src/session/prompt.ts`                                                                             | `currentModel`、`createUserMessage`、`SessionPrompt.prompt` | `614-689`, `1052-1071` | `0e3474509aa5ad16afcf9c439785514d6443c6af` |
| S4  | `packages/opencode/src/session/prompt.ts`                                                                             | `SessionPrompt.run`、`SessionPrompt.loop`                   | `1081-1347`            | `0e3474509aa5ad16afcf9c439785514d6443c6af` |
| S5  | `packages/opencode/src/session/processor.ts`                                                                          | `handleEvent`、`cleanup`、`process`                         | `315-537`, `539-683`   | `0e3474509aa5ad16afcf9c439785514d6443c6af` |
| S6  | `packages/opencode/src/tool/todo.ts`; `packages/opencode/src/session/todo.ts`                                         | `TodoWriteTool.execute`; `Todo.update/get`                  | `14-46`; `29-66`       | `0e3474509aa5ad16afcf9c439785514d6443c6af` |
| S7  | `packages/opencode/src/tool/task.ts`                                                                                  | `TaskTool.execute`、`TaskTool.runTask`                      | `81-347`               | `0e3474509aa5ad16afcf9c439785514d6443c6af` |
| S8  | `packages/opencode/src/agent/subagent-permissions.ts`                                                                 | `deriveSubagentSessionPermission`                           | `14-27`                | `0e3474509aa5ad16afcf9c439785514d6443c6af` |
| S9  | `packages/tui/src/component/prompt/index.tsx`                                                                         | 命令 `session.interrupt`                                    | `393-419`              | `0e3474509aa5ad16afcf9c439785514d6443c6af` |
| S10 | `packages/opencode/src/server/routes/instance/httpapi/handlers/session.ts`; `packages/opencode/src/session/prompt.ts` | `SessionHttpApi.abort`; `SessionPrompt.cancel`              | `232-235`; `152-155`   | `0e3474509aa5ad16afcf9c439785514d6443c6af` |
| S11 | `packages/opencode/src/session/run-state.ts`                                                                          | `cancel`、`ensureRunning`、`cancelBackgroundJobs`           | `52-94`, `111-143`     | `0e3474509aa5ad16afcf9c439785514d6443c6af` |
| S12 | `packages/opencode/src/session/retry.ts`                                                                              | `retryable`、`policy`                                       | `84-205`               | `0e3474509aa5ad16afcf9c439785514d6443c6af` |

### 10.2 Native V2 源码

| ID  | 路径                                                                                           | 函数/符号                                                  | 行号                | SHA                                        |
| --- | ---------------------------------------------------------------------------------------------- | ---------------------------------------------------------- | ------------------- | ------------------------------------------ |
| V1  | `packages/server/src/handlers/session.ts`                                                      | Handler `session.prompt`                                   | `139-171`           | `0e3474509aa5ad16afcf9c439785514d6443c6af` |
| V2  | `packages/core/src/session.ts`                                                                 | `V2Session.prompt`                                         | `360-386`           | `0e3474509aa5ad16afcf9c439785514d6443c6af` |
| V3  | `packages/opencode/src/server/routes/instance/httpapi/server.ts`                               | `createRoutes` Layer composition                           | `298-303`           | `0e3474509aa5ad16afcf9c439785514d6443c6af` |
| V4  | `packages/core/src/session/runner/llm.ts`                                                      | `runTurnAttempt`、`SessionRunner.run`                      | `173-406`           | `0e3474509aa5ad16afcf9c439785514d6443c6af` |
| V5  | `packages/core/src/tool/builtins.ts`                                                           | `BuiltInTools.node`、remaining-port TODO                   | `18-48`             | `0e3474509aa5ad16afcf9c439785514d6443c6af` |
| V6  | `packages/core/src/agent.ts`; `packages/core/src/plugin/agent.ts`                              | `AgentV2.select`; built-in Agent transforms                | `67-105`; `96-202`  | `0e3474509aa5ad16afcf9c439785514d6443c6af` |
| V7  | `packages/core/src/config/plugin/agent.ts`; `packages/core/src/session/runner/model.ts`        | Agent 配置物化；`SessionRunnerModel.locationLayer.resolve` | `80-111`; `181-215` | `0e3474509aa5ad16afcf9c439785514d6443c6af` |
| V8  | `packages/core/src/session/execution/local.ts`; `packages/core/src/session/run-coordinator.ts` | `SessionExecutionLocal`; `SessionRunCoordinator.make`      | `10-46`; `24-104`   | `0e3474509aa5ad16afcf9c439785514d6443c6af` |
| V9  | `packages/core/src/session/runner/llm.ts`                                                      | Runner 状态清单与 Retry/Doom Loop TODO                     | `43-90`             | `0e3474509aa5ad16afcf9c439785514d6443c6af` |
| P1  | `specs/v2/session.md`; `specs/v2/todo.md`                                                      | Session parity/延后项；next-slice 清单                     | `101-169`; `50-74`  | `0e3474509aa5ad16afcf9c439785514d6443c6af` |

### 10.3 关键测试定位

| ID  | 路径与测试                                                                                  | 行号                     | 证明范围                        | SHA                                        |
| --- | ------------------------------------------------------------------------------------------- | ------------------------ | ------------------------------- | ------------------------------------------ |
| T1  | `packages/opencode/test/agent/agent.test.ts`，默认 Agent 与 build/plan 权限测试             | `47-110`                 | 默认角色与用户覆盖              | `0e3474509aa5ad16afcf9c439785514d6443c6af` |
| T2  | `packages/opencode/test/session/prompt.test.ts`，`loop continues when finish is tool-calls` | `825-851`                | Tool continuation               | `0e3474509aa5ad16afcf9c439785514d6443c6af` |
| T3  | 同文件，`loop continues when finish is stop but assistant has tool parts`                   | `892-918`                | finish reason 不是唯一停止依据  | `0e3474509aa5ad16afcf9c439785514d6443c6af` |
| T4  | 同文件，cancel 与 active-run prompt tests                                                   | `1123-1169`, `1405-1469` | Interrupt 与运行中新输入        | `0e3474509aa5ad16afcf9c439785514d6443c6af` |
| T5  | `packages/opencode/test/tool/task.test.ts`，resume/create/depth tests                       | `219-469`                | 子 Session、恢复与深度          | `0e3474509aa5ad16afcf9c439785514d6443c6af` |
| T6  | `packages/opencode/test/agent/plan-mode-subagent-bypass.test.ts`                            | `29-160`                 | 父 Agent 与父 Session deny 边界 | `0e3474509aa5ad16afcf9c439785514d6443c6af` |
| T7  | `packages/opencode/test/session/retry.test.ts`，policy tests                                | `97-148`                 | retry status 与最多 5 次 retry  | `0e3474509aa5ad16afcf9c439785514d6443c6af` |
| T9  | `packages/core/test/tool-todowrite.test.ts`                                                 | `85-124`                 | V2 Todo 权限、持久化与输出      | `0e3474509aa5ad16afcf9c439785514d6443c6af` |
| T10 | `packages/core/test/session-runner.test.ts`，steer/queue tests                              | `1865-1953`              | V2 promotion 边界               | `0e3474509aa5ad16afcf9c439785514d6443c6af` |
| T11 | 同文件，final-step tests                                                                    | `3008-3106`              | V2 禁用 Tools 与 allowance 重置 | `0e3474509aa5ad16afcf9c439785514d6443c6af` |

测试文件存在只证明仓库具有相应覆盖；任务 7 的完整命令、环境、临时实验文件、耗时与复跑日志见 [`research/10_Research_Agent_and_Orchestration.md`](./research/10_Research_Agent_and_Orchestration.md)。

### 10.4 任务 7 实测结果（2026-08-18）

环境为固定 SHA 和 Linux；没有配置或调用真实、付费 Provider。详细过程不在正式教学正文重复，关键证据如下：

| 证据                         | 结果                                                             | 结论边界                                                                       |
| ---------------------------- | ---------------------------------------------------------------- | ------------------------------------------------------------------------------ |
| 当前兼容路径组跑             | `178 pass / 1 skip / 1 fail`                                     | 唯一失败是 `cancel interrupts loop queued behind shell` 超时，不能写成整组通过 |
| 唯一超时用例定向复跑         | `1 pass`                                                         | 仅说明超时未稳定复现，不能排除潜在竞态                                         |
| V2 定向组跑                  | `18 pass / 0 fail`                                               | 覆盖 run coordinator 与 `todowrite` 的选定测试，不代表 V2 parity 完整          |
| `agent.steps` 隔离实验（E1） | `1 pass`；`steps: 1` 时仍发生 3 次 Provider 调用且请求携带 Tools | 只证明至少 3 次调用，不证明无限循环，也不外推到所有真实 Provider adapter       |

**明确未运行**

- Doom Loop 三次相同调用实验。
- 跨父 Session 的 `task_id` 所有权实验。
- Retry 已产生部分输出后的重复投影实验。
- 真实 Provider / Provider adapter 实验。
- 任务 8 Teach-back。

## 11. 本章小结

1. 当前 TUI 普通消息仍走兼容 `SessionPrompt`；native V2 已接线，但尚未接管 TUI。
2. Agent 是包含 Prompt、Model 偏好、Permission、Tools 和步骤策略的行为配置；Model 负责概率性地产生文本与 Tool Call。
3. 当前 Orchestration 由显式 Session Loop 驱动。普通本地工具形成 durable terminal state 和 Model Tool Output 后，Loop 重载历史并发起新的 Provider Turn。
4. Todo 是 Session 清单；普通 Tool 是单次能力调用；Task Tool 是委派入口；Subagent 是独立子 Session 中的 Agent。四者不能互换。
5. Retry 的 provider request attempts 共享同一 Assistant Message/Processor context，但每次请求及响应投影分别构成一个 Provider Turn；Interrupt 结算状态但不回滚外部副作用；Doom Loop 是 Permission ask，不是自动失败。
6. 当前旧 `agent.steps` 不是硬上限。任务 7 实验证明 `steps: 1` 后 Tools 仍可用且至少发生 3 次 Provider 调用；它没有证明无限循环。
7. native V2 已实现 durable admission、独立 Runner、Tool continuation、Todo、steer/queue 和更强的 steps allowance；Task/Subagent、一般 Retry、Doom Loop、部分 Agent/Model policy 和跨进程恢复仍有缺口。

### 掌握要点

- 一个用户请求可以因已结算 Model Tool Output 所触发的 continuation 或 Retry 产生多个 Provider Turn。
- `build` 与 `plan` 的差异主要来自 Harness policy，而不是 Model 的确定性人格。
- Todo、普通 Tool、Task Tool、Subagent 和子 Session 各自承担不同职责。
- 旧 `agent.steps` 与 V2 `steps` 具有不同的约束强度。
- 固定源码结论、任务 7 实测证据和未决限制在证据等级上保持区分。

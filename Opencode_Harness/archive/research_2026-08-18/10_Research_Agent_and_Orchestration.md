# Agent 与 Orchestration：从一次代码修改理解 OpenCode 如何持续行动

状态：**任务 3-5 模块研究初稿；任务 7 已完成最小验证；任务 8 按用户指示跳过、未作理解验收；待任务 6 交叉审计**。

核对日期：**2026-08-18**。

源码版本：`0e3474509aa5ad16afcf9c439785514d6443c6af`（分支 `dev`）。

研究方法：固定上述 commit 的静态源码、入口接线、规格和测试交叉阅读；任务 7 使用已按 lockfile 安装的依赖运行指定测试，并以本地隔离 Fake Provider 完成一个临时实验；未调用付费或真实 Provider、未使用密钥或网络业务调用，也未把目录名、类型名或规格计划单独当成已启用证据。

引用约定：下文证据中的“完整 SHA”“版本同上”均严格指 `0e3474509aa5ad16afcf9c439785514d6443c6af`，没有混用其他 commit。

系列位置：这是任务 3-5 四份模块笔记中的第 1 篇。建议先读 `06_Current_Runtime_End_to_End_Trace.md` 建立总体链路，再读本文理解“谁驱动循环、为什么继续或停止”；之后按顺序阅读 `11_Research_Context_and_Persistence.md`、`12_Research_Tools_and_Security.md` 和 `13_Research_Runtime_Boundary.md`。

## 1. 本笔记解决什么问题

读完本笔记，读者应能按执行顺序回答五组问题：

1. OpenCode 如何确定本轮使用哪个 Agent、Model 和权限，`build` 与 `plan` 到底差在哪里？
2. 为什么一次用户输入可能触发多个提供商轮次（Provider Turn），Tool Result 又怎样触发 continuation？
3. Todo、Task Tool、Subagent 和普通 Tool 分别是什么，父子 Session 如何连接？
4. Retry、Interrupt、最大步数、停止条件和 Doom Loop 如何约束一个概率性模型？
5. 当前默认旧运行时和 native V2 各自已经做到什么，哪些仍是 partial 或 missing/planned？

### 1.1 前置知识

建议先读：

- `00_Project_Charter.md`：研究范围。
- `02_Evidence_and_V1_V2_Rules.md`：状态标签和证据等级。
- `06_Current_Runtime_End_to_End_Trace.md`：从 TUI 到 Provider、Tool 和事件通道的完整骨架。

本笔记继承并再次核对了任务 2 的结论：

> `[Current default @ 0e3474509aa5ad16afcf9c439785514d6443c6af]` 当前 TUI 普通消息走兼容 Session API 和 `SessionPrompt` 旧编排，同时复用 EventV2、Core Projector、共享 Schema、LLM Event 和新旧 Server 组合层；它没有进入 native V2 `SessionV2.prompt -> SessionExecution -> SessionRunner`。

证据：

- 文件：`packages/tui/src/component/prompt/index.tsx`；函数：`submitInner()` 普通消息分支；位置：`1092-1146`；版本：`0e3474509aa5ad16afcf9c439785514d6443c6af`。
- 文件：`packages/opencode/src/server/routes/instance/httpapi/handlers/session.ts`；函数：`SessionHttpApi.prompt`；位置：`295-309`；版本同上。
- 文件：`packages/opencode/src/session/prompt.ts`；函数：`SessionPrompt.prompt`、`SessionPrompt.loop`；位置：`1052-1071`、`1343-1347`；版本同上。
- 对照文件：`packages/server/src/handlers/session.ts`；Handler：`session.prompt`；位置：`139-171`；版本同上。
- 对照文件：`packages/core/src/session.ts`；函数：`V2Session.prompt`；位置：`360-386`；版本同上。

### 1.2 建议学习路线

先跟随第 2 节的例子建立直觉，再读第 3 节术语边界；第 4-8 节按当前默认路径的真实执行顺序展开；第 9 节单独学习 Todo 与委派；第 10 节学习失败和约束；最后用第 11 节的 V2 对照修正“看到 V2 目录就是默认运行时”的误解。

## 2. 贯穿例子：修改一个缓存 Bug

用户输入：

> 修复 `src/cache.ts` 的并发缓存 Bug，先调查相关调用，修改实现并运行测试。

下面先给出业务层故事，后文逐步映射到源码。

1. TUI 把当前选择的 `agent`、`model`、`variant` 和文本一起提交。
2. Harness 创建持久化 User Message。它确定本次输入使用 `build` 还是 `plan`，并把选定 Model 写到消息和 Session。
3. `SessionPrompt.run` 重载 Session History，创建 Assistant Message，组装 Agent 指令、Tools、权限和消息。
4. 第一个 Provider Turn 发生。模型可能概率性地选择先调用 `read`、`grep`，也可能直接解释。
5. Harness 确定性地校验 Tool 参数、检查权限、执行工具，并持久化 Tool Result。
6. 如果模型调用 `task`，Harness 不会把它当成普通文件读取；`TaskTool` 创建一个子 Session，用指定 Subagent 运行独立的 `SessionPrompt.prompt`。
7. 前台 Subagent 完成后，子 Session 最后一个文本结果成为父 Assistant Message 中 `task` Tool Part 的 output。父循环重载历史，下一次 Provider Turn 才能看到结果。
8. 如果模型连续用完全相同参数调用同一 Tool 三次，旧 Processor 会触发 `doom_loop` Permission；默认是询问，而不是静默无限执行。
9. 当模型最终返回普通文本且没有需要继续的本地 Tool Part，外层循环在下一次顶部检查停止。
10. 用户中断时，Harness 取消当前 Runner 和 Provider Stream，并尽力结算部分文本、推理和未完成 Tool。

这个例子里，模型决定“下一步想做什么”；Harness 决定“这个动作是否可见、允许、如何执行、怎样记录、是否再请求模型以及何时停下”。

## 3. 五个边界概念

### 3.1 Agent 不是模型副本

`[General concept]` Agent 是一次执行所采用的行为配置，包括身份、指令、Model 偏好、Tools/Permission、可见性和步骤预算。Model 是产生下一段文本或 Tool Call 的推理服务。一个 Model 可以被多个 Agent 使用，一个 Agent 也可以通过配置选择 Model。

`[Current default @ 0e3474509aa5ad16afcf9c439785514d6443c6af]` 当前 `Agent.Info` 明确包含 `model`、`variant`、`prompt`、`permission`、采样参数和 `steps`，所以不能把 Agent 简化为“System Prompt 的别名”。

证据：文件 `packages/opencode/src/agent/agent.ts`；导出符号 `Info`；位置 `35-55`；版本 `0e3474509aa5ad16afcf9c439785514d6443c6af`。测试：`packages/opencode/test/agent/agent.test.ts`，测试 `returns default native agents when no config`、`build agent has correct default properties`、`plan agent denies edits except .opencode/plans/*`，位置 `47-90`；版本同上。

### 3.2 Orchestration 是 Harness 的控制流

`[General concept]` 编排（Orchestration）是围绕模型建立的确定性控制流：读取状态、构造请求、执行 Tool、结算结果、判断 continuation、重试、中断和停止。它不等于模型内部推理。

`[Current default @ 0e3474509aa5ad16afcf9c439785514d6443c6af]` 当前默认路径的主 Orchestrator 是 `SessionPrompt.run` 的显式 `while (true)`，不是 AI SDK 的隐式“自动 Agent”。

证据：文件 `packages/opencode/src/session/prompt.ts`；函数 `SessionPrompt.run`（局部实现名 `runLoop`）；位置 `1081-1341`，循环入口 `1088`；版本 `0e3474509aa5ad16afcf9c439785514d6443c6af`。测试：`packages/opencode/test/session/prompt.test.ts`，测试 `loop continues when finish is tool-calls`，位置 `825-851`；版本同上。

### 3.3 Provider Turn 不是完整用户回合

`[General concept]` 一个 Provider Turn 是一次模型提供商请求及其响应投影。一次用户请求可包含多个 Provider Turn；一个 Provider Turn 内又会流式产生 Text、Reasoning、Tool Call、Usage 和 Finish 事件。

`[Current default @ 0e3474509aa5ad16afcf9c439785514d6443c6af]` 每次外层循环调用一次 `handle.process(...)`；`SessionProcessor.process` 再调用一个 `llm.stream(streamInput)`。Tool Result 后需要 continuation 时，外层循环重载历史并再次走这一边界。

证据：文件 `packages/opencode/src/session/prompt.ts`；函数 `SessionPrompt.run`；位置 `1272-1286`、`1319-1335`；文件 `packages/opencode/src/session/processor.ts`；函数 `SessionProcessor.process`；位置 `627-683`；版本均为完整基线 SHA。测试：`packages/opencode/test/session/processor-effect.test.ts`，测试 `session.processor effect tests complete AI SDK tool calls when native flag is off`，位置 `751-814`；版本同上。

### 3.4 Session Loop 比 Provider Turn 更外层

`[Interpretation]` 可以把 Provider Turn 看成“一次问模型”，把 Session Loop 看成“为了完成当前 Session 中尚未完成的工作，可能反复问模型”。Loop 的状态来自持久化 Message/Part 和少量进程内控制状态，不应称为模型的“思考循环”。

依据：`SessionPrompt.run` 每轮调用 `MessageV2.filterCompactedEffect` 重载历史，位置 `1092-1096`；创建并运行一个 Assistant Turn，位置 `1186-1286`；版本 `0e3474509aa5ad16afcf9c439785514d6443c6af`。

### 3.5 Subagent 不是普通 Tool，也不是远程 A2A Agent

`[Current default @ 0e3474509aa5ad16afcf9c439785514d6443c6af]` Subagent 是在独立子 Session 中以另一份 Agent 配置运行的 OpenCode Agent。`task` 是父模型用来请求这种委派的 Tool；它仍经过 Tool Call/Tool Result 生命周期，但它的执行体内部创建或恢复子 Session，并再次调用 Session Prompt 编排。

证据：文件 `packages/opencode/src/tool/task.ts`；导出符号 `TaskTool`、函数 `TaskTool.execute`/`TaskTool.runTask`；位置 `81-214`；版本 `0e3474509aa5ad16afcf9c439785514d6443c6af`。测试：`packages/opencode/test/tool/task.test.ts`，测试 `execute resumes an existing task session from task_id`，位置 `219-256`；版本同上。

不要混淆：

- 普通 Tool：一次受控能力调用，例如 `read`、`bash`、`edit`。
- Task Tool：Tool Registry 中一个特殊的委派工具。
- Subagent：Task Tool 启动的子 Session 执行者。
- `SubtaskPart`：旧 Slash Command 路径预先编码的内部任务 Part；它可由 Harness 直接转成 Task Tool 执行，不要求父模型先生成 `task` Tool Call。
- Agent mention：User Part 被展开成“请调用 task”的合成提示，不是立即启动 Subagent。

相关证据：`packages/opencode/src/session/prompt.ts`，`resolveUserPart` 的 Agent Part 分支 `974-989`、`SessionPrompt.handleSubtask` `255-449`、`SessionPrompt.command` 的 `SubtaskPart` 构造 `1439-1473`；版本同上。

## 4. 当前默认路径第一阶段：确定 Agent 与 Model

### 4.1 TUI 把选择显式送入消息

`[Current default @ 0e3474509aa5ad16afcf9c439785514d6443c6af]` 普通 TUI 提交把 `agent.name`、`selectedModel` 和 `variant` 放入兼容 Prompt 请求。因此常见交互路径不是让 Server 每次盲猜 Agent/Model。

证据：文件 `packages/tui/src/component/prompt/index.tsx`；函数 `submitInner()`；位置 `1092-1119`；版本完整 SHA。入口测试：`packages/opencode/test/server/httpapi-sdk.test.ts`，测试 `matches generated SDK prompt streaming through fake LLM`，位置 `774-806`；版本同上。

### 4.2 Agent 选择优先级

`[Current default @ 0e3474509aa5ad16afcf9c439785514d6443c6af]` `createUserMessage` 的 Agent 规则是：有 `input.agent` 就按名称取；否则使用 `Agent.defaultInfo()`。不存在的显式 Agent 会发布 Session Error 并失败，不会静默退回默认 Agent。

默认 Agent 的选择规则是：配置了 `default_agent` 时必须存在、不能是纯 Subagent、不能 hidden；否则取第一个非 Subagent 且非 hidden 的 Agent。内置对象插入顺序使无配置时通常选 `build`。

证据：

- 文件：`packages/opencode/src/session/prompt.ts`；函数：`SessionPrompt.createUserMessage`；位置：`635-644`；版本：完整 SHA。
- 文件：`packages/opencode/src/agent/agent.ts`；函数：`Agent.defaultInfo` 的内部实现；位置：`328-340`；版本同上。
- 测试：`packages/opencode/test/agent/agent.test.ts`；测试：`returns default native agents when no config`；位置：`47-59`；版本同上。

### 4.3 Model 与 Variant 选择优先级

`[Current default @ 0e3474509aa5ad16afcf9c439785514d6443c6af]` 新 User Message 的 Model 优先级为：

```text
input.model
-> 选中 Agent 的 agent.model
-> Session 表当前 model
-> 最近带 model 的 User Message
-> Provider.defaultModel()
```

Variant 优先使用 `input.variant`；否则只在最终 Model 与 Agent 配置 Model 相同且该 Variant 确实存在时使用 `agent.variant`。选定值写入 User Message，并在变化时通过 `Session.setAgentModel` 更新 Session。

证据：文件 `packages/opencode/src/session/prompt.ts`；函数 `currentModel`、`SessionPrompt.createUserMessage`；位置 `614-689`；文件 `packages/opencode/src/session/session.ts`；函数 `Session.setAgentModel`；位置 `767-778`；版本完整 SHA。Agent Model 配置测试：`packages/opencode/test/agent/agent.test.ts`，配置 Model 相关测试主体位置 `184-238`；版本同上。

### 4.4 Provider Turn 再按最新 User Message 取值

`[Current default @ 0e3474509aa5ad16afcf9c439785514d6443c6af]` 外层循环每轮从重载历史中的最新 User Message 读取 `lastUser.agent` 和 `lastUser.model`，解析真实 Provider Model，再创建 Assistant Message。因而同一 User Message 引发的 Tool continuation 通常保持同一选择；运行中进入的新 User Message 可以改变下一轮选择。

证据：文件 `packages/opencode/src/session/prompt.ts`；函数 `SessionPrompt.run`；位置 `1092-1098`、`1141-1145`、`1170-1201`；版本完整 SHA。测试：`packages/opencode/test/session/prompt.test.ts`，测试 `prompt submitted during an active run is included in the next LLM input`，位置 `1405-1469`；版本同上。

## 5. Build、Plan 与其他内置 Agent

### 5.1 Build

`[Current default @ 0e3474509aa5ad16afcf9c439785514d6443c6af]` `build` 是 primary、native、默认可见 Agent。默认权限基线允许工具，并把 `question` 与 `plan_enter` 设为允许；读取 `.env` 与外部目录等仍有更具体的 ask 规则。

证据：文件 `packages/opencode/src/agent/agent.ts`；Agent Layer 初始化中 `defaults` 与 `build`；位置 `98-155`；测试 `packages/opencode/test/agent/agent.test.ts`，`build agent has correct default properties`，位置 `61-70`；版本完整 SHA。

### 5.2 Plan

`[Current default @ 0e3474509aa5ad16afcf9c439785514d6443c6af]` `plan` 也是 primary Agent，但默认拒绝一般编辑，只允许特定 plan 文件路径；允许 `question` 和 `plan_exit`，并默认拒绝 `task:general`。它不是“Model 自动只思考不行动”，而是 Agent Permission、提醒文本和 Plan Tool 共同形成的约束。

证据：

- 文件：`packages/opencode/src/agent/agent.ts`；Agent Layer 中 `plan`；位置 `156-181`；版本完整 SHA。
- 文件：`packages/opencode/src/session/reminders.ts`；函数 `SessionReminders.apply`；位置 `15-90`；版本同上。
- 文件：`packages/opencode/src/tool/plan.ts`；导出符号 `PlanExitTool`、`execute`；位置 `15-79`；版本同上。
- 测试：`packages/opencode/test/agent/agent.test.ts`；测试 `plan agent denies edits except .opencode/plans/*`、`plan agent denies the general subagent by default`；位置 `72-90`；版本同上。

用户配置通过后合并，能够覆盖内置规则。例如测试明确允许从 Plan 调用 `general`。因此“Plan 永远不能编辑/委派”也是过度表述。

测试证据：`packages/opencode/test/agent/agent.test.ts`，测试 `user permission can allow the general subagent from plan mode`，位置 `93-110`；版本完整 SHA。

### 5.3 General、Explore 与隐藏 Agent

`[Current default @ 0e3474509aa5ad16afcf9c439785514d6443c6af]` `general` 和 `explore` 是 Subagent。`general` 默认禁用 Todo；`explore` 采用更窄的只读式权限集合。`compaction`、`title`、`summary` 是 hidden primary Agent，用于特定内部工作，不是用户主循环里可互换的普通人格。

证据：文件 `packages/opencode/src/agent/agent.ts`；Agent Layer 中 `general` 至 `summary`；位置 `182-265`；测试 `packages/opencode/test/agent/agent.test.ts`，位置 `112-181`；版本完整 SHA。

## 6. 当前默认路径第二阶段：外层 Session Loop

### 6.1 串行化后进入循环

`[Current default @ 0e3474509aa5ad16afcf9c439785514d6443c6af]` `SessionPrompt.loop` 使用 `SessionRunState.ensureRunning`。同一 Session 的并发 loop 调用加入同一个 Runner；不同调用者不会各自启动一套 Provider 循环。

证据：文件 `packages/opencode/src/session/prompt.ts`；函数 `SessionPrompt.loop`；位置 `1343-1347`；文件 `packages/opencode/src/session/run-state.ts`；函数 `SessionRunState.runner`、`ensureRunning`；位置 `52-69`、`88-94`；版本完整 SHA。测试：`packages/opencode/test/session/prompt.test.ts`，测试 `concurrent loop callers get same result`，位置 `1372-1385`；版本同上。

### 6.2 每轮先重载 durable history

观察重点：Loop 不是拿一个内存 messages 数组不断追加，而是每轮重新读取投影历史。

```ts
while (true) {
  yield* status.set(sessionID, { type: "busy" })
  let msgs = yield* MessageV2.filterCompactedEffect(sessionID)
  const { user: lastUser, assistant: lastAssistant, finished: lastFinished, tasks } = MessageV2.latest(msgs)
  // terminal check -> task/compaction -> one provider turn
}
```

文件：`packages/opencode/src/session/prompt.ts`；函数：`SessionPrompt.run`；位置：`1088-1096`；版本：`0e3474509aa5ad16afcf9c439785514d6443c6af`。

`[Interpretation]` 业务意义是 Tool Result、运行中进入的新 User Message、Compaction 和其他持久化变化都能在下一 Provider Turn 重新进入 Context；continuation 不是只依赖上一轮闭包中的临时数组。

### 6.3 顶部 terminal check

`[Current default @ 0e3474509aa5ad16afcf9c439785514d6443c6af]` 最新 Assistant 必须同时满足以下条件才在顶部退出：

- 已有 finish；
- finish 不是 `tool-calls`；
- 没有非 `providerExecuted`、非“中断孤儿”的 Tool Part；
- Assistant 的 `parentID` 对应最新 User Message。

这解释了为什么某些 Provider 即使返回 `stop`，只要同一 Assistant 中存在本地 Tool Part，Loop 仍应继续，把 Tool Result 送回模型。

证据：文件 `packages/opencode/src/session/prompt.ts`；函数 `SessionPrompt.run` 的 terminal check；位置 `1100-1130`；测试 `packages/opencode/test/session/prompt.test.ts`，测试 `loop continues when finish is stop but assistant has tool parts`，位置 `892-918`；版本完整 SHA。

### 6.4 特殊任务先于普通 Provider Turn

`[Current default @ 0e3474509aa5ad16afcf9c439785514d6443c6af]` 重载历史后，Loop 先处理最新 `SubtaskPart` 或 Compaction Part，再检查 token overflow；只有这些分支都不接管时才创建普通 Assistant Message 并进入 Provider Turn。

证据：文件 `packages/opencode/src/session/prompt.ts`；函数 `SessionPrompt.run`；位置 `1141-1168`；版本完整 SHA。这里的 `SubtaskPart` 是旧命令编排数据，不应与模型在普通 Provider Turn 中发出的 `task` Tool Call 混写。

### 6.5 组装一次 Provider Request

`[Current default @ 0e3474509aa5ad16afcf9c439785514d6443c6af]` 当前轮会创建并先持久化 Assistant Message，然后解析 Tools、Agent 提醒、Skills、Environment、Instructions、MCP Instructions 和 Model Messages。`LLMRequestPrep.prepare` 再合并 Agent Prompt 或 Provider base prompt、per-prompt system、Agent/Model/Variant options、插件参数与 headers，并按权限过滤最终可见 Tools。

证据：

- `packages/opencode/src/session/prompt.ts`；`SessionPrompt.run`；`1186-1286`；完整 SHA。
- `packages/opencode/src/session/llm/request.ts`；`LLMRequestPrep.prepare`、`resolveTools`；`56-214`；完整 SHA。
- `packages/opencode/src/session/llm.ts`；`LLM.run`、`LLM.stream`；`85-381`；完整 SHA。

默认 Transport 是 AI SDK `streamText`；可选 native LLM adapter 只替换旧 Session Loop 下的 LLM 请求/流适配，不等于 native V2 Session Runtime。证据为 `packages/opencode/src/session/llm.ts` 的 runtime gate 与默认分支 `224-280`、stream 适配 `357-381`；版本完整 SHA。

## 7. Tool Result 后为什么会 continuation

### 7.1 Provider Stream 中已经执行普通本地 Tool

`[Current default @ 0e3474509aa5ad16afcf9c439785514d6443c6af]` 默认 AI SDK 路径在一次 `llm.stream` 内执行本地 Tool；LLM Runtime 把结果归一化成 `tool-result`/`tool-error`，Processor 将 Tool Part 从 pending/running 结算到 completed/error。外层 Loop 的下一轮不是才开始执行 Tool，而是把已持久化结果重新发给 Provider。

证据：文件 `packages/opencode/src/session/processor.ts`；函数 `handleEvent`；位置 `315-419`；文件 `packages/opencode/src/session/llm.ts`；默认 AI SDK Tool 配置 `276-353`；版本完整 SHA。测试：`packages/opencode/test/session/processor-effect.test.ts`，位置 `751-814`；版本同上。

### 7.2 Processor 返回三值，Loop 决定下一步

`[Current default @ 0e3474509aa5ad16afcf9c439785514d6443c6af]` Processor 返回 `continue`、`stop` 或 `compact`：需要压缩则 compact；权限拒绝等导致 blocked 或 Assistant Error 则 stop；否则 continue。Loop 对 stop 直接退出，对 compact 创建 Compaction marker，对 continue 回到顶部重载历史。

证据：`packages/opencode/src/session/processor.ts`，`SessionProcessor.process`，`627-683`；`packages/opencode/src/session/prompt.ts`，`SessionPrompt.run`，`1319-1335`；版本完整 SHA。

### 7.3 普通最终文本也会先回顶部一次

`[Interpretation]` Processor 本身通常不会仅因普通 `stop` 文本直接返回 `stop`；它返回 continue，外层 Loop 重载历史，再由顶部 terminal check 无 Provider 请求地退出。这种职责拆分让 terminal check 同时考虑最新 User Message 和 Tool Part，而不是只看 Provider finish reason。

依据：`packages/opencode/src/session/processor.ts:679-681` 与 `packages/opencode/src/session/prompt.ts:1100-1130,1334-1339`，版本完整 SHA。

## 8. 模型决策与 Harness 控制的分界

| 问题 | 谁决定 | 当前实现含义 |
| --- | --- | --- |
| 下一步读文件、改文件还是回答 | 模型，概率性 | Provider 生成 Text 或 Tool Call |
| Tool 是否出现在请求中 | Harness，确定性 | Agent/Session/User Permission 与 tool override 过滤 |
| Tool 参数是否合法 | Harness，确定性 | Tool schema 校验 |
| 是否需要询问用户 | Harness 按规则确定，用户决定答复 | Permission/Question 生命周期 |
| Tool 实际副作用 | Tool Runtime，确定性但可能失败 | 模型只生成调用意图 |
| Tool Result 是否持久化 | Harness | Processor/EventV2/Projector |
| 是否发起下一 Provider Turn | Harness | `SessionPrompt.run` + Processor Result + terminal check |
| 下一轮模型会如何利用结果 | 模型，概率性 | Harness 只保证结果被投影进请求，不保证模型正确使用 |

`[Interpretation]` Harness 能约束可执行空间和生命周期，不能保证模型一定提出正确计划、一定调用 Todo、一定委派合适 Subagent，或一定在获得 Tool Result 后作出正确判断。

## 9. Todo、Task 与 Subagent

### 9.1 Todo 是 Session 的结构化清单，不是调度器

`[Current default @ 0e3474509aa5ad16afcf9c439785514d6443c6af]` `todowrite` 接收完整 Todo 数组，经过 Permission 后替换当前 Session 在 `todo` 表中的有序清单并发布更新事件。它不创建 Provider Turn、不启动后台任务，也不会自动强制模型完成 pending 项。

证据：文件 `packages/opencode/src/tool/todo.ts`；导出符号 `TodoWriteTool`；位置 `14-46`；文件 `packages/opencode/src/session/todo.ts`；函数 `Todo.update`、`Todo.get`；位置 `29-66`；版本完整 SHA。相关测试：`packages/opencode/test/session/schema-decoding.test.ts`，`Todo.Info` 测试组，位置 `255-262`；`packages/opencode/test/server/httpapi-session.test.ts`，Session todo 读取断言，位置 `325-348`；版本同上。本轮未找到当前旧 `TodoWriteTool.execute -> Todo.update` 的直接行为测试，这是明确测试缺口；native V2 对应写入行为已有第 11.9 节测试。

`[Interpretation]` Todo 的价值是把模型自报的计划从自然语言变成 Session 级可观察状态；它不是 Harness 内部 continuation 的来源。即便所有 Todo 都 completed，主循环也仍按 Assistant finish、Tool Part、Processor Result 等停止条件判断。

### 9.2 Task Tool 创建或恢复子 Session

观察重点：子 Agent 不是在父 Session History 中“换人格”，而是获得新的 Session ID。

```ts
const nextSession = session ?? (yield* sessions.create({
  parentID: ctx.sessionID,
  title: params.description + ` (@${next.name} subagent)`,
  agent: next.name,
  permission: [
    ...childPermission,
    ...childToolDenies.filter(
      (deny) => !childPermission.some(
        (rule) =>
          rule.permission === deny.permission &&
          rule.pattern === deny.pattern &&
          rule.action === deny.action,
      ),
    ),
  ],
}))

const result = yield* ops.prompt({
  sessionID: nextSession.id,
  model,
  agent: next.name,
  parts,
})
```

文件：`packages/opencode/src/tool/task.ts`；函数：`TaskTool.execute`、`TaskTool.runTask`；位置：`136-172`、`200-214`；版本：完整 SHA。

`[Current default @ 0e3474509aa5ad16afcf9c439785514d6443c6af]` 未给 `task_id` 时创建带 `parentID` 的子 Session；给出已有 `task_id` 时恢复该 Session。当前实现只尝试按 ID 取 Session，没有在 `136-138` 行显式验证该 Session 是否真属于当前父 Session，这是一个边界风险，不应把 `task_id` 描述成强所有权句柄。

测试证据：`packages/opencode/test/tool/task.test.ts`，`execute resumes an existing task session from task_id` 与 `execute creates a child when task_id does not exist`，位置 `219-256`、`354-389`；版本完整 SHA。

### 9.3 子 Session 获得什么上下文

`[Current default @ 0e3474509aa5ad16afcf9c439785514d6443c6af]` 新子 Session 不自动复制父 Session History。父模型应在 `params.prompt` 中提供完成任务所需上下文；Task Tool 解析该 prompt 为新的子 User Message。父子只通过 `parentID`、任务 metadata、显式 prompt 和返回的 Tool output 连接。

证据：`packages/opencode/src/tool/task.ts`，`BaseParameterFields.prompt` `43-52`、`runTask` `200-214`；Session 创建字段见 `packages/opencode/src/session/session.ts`，`Session.createNext` `501-540`；版本完整 SHA。该结论中的“不自动复制”是对上述创建和 prompt 输入代码的静态解释。

### 9.4 子 Model 如何选择

`[Current default @ 0e3474509aa5ad16afcf9c439785514d6443c6af]` Task 子 Model 优先使用 Subagent 自己的 `next.model`；否则继承发出 Task Call 的父 Assistant Message 的 `providerID/modelID`。只有继承 Model 时才继承父 Assistant 的 Variant；Subagent 显式 Model 时 Variant 设为 undefined，由子 Agent/Model 后续规则决定。

证据：文件 `packages/opencode/src/tool/task.ts`；函数 `TaskTool.execute`；位置 `174-212`；版本完整 SHA。

### 9.5 权限不是简单复制父 Agent

`[Current default @ 0e3474509aa5ad16afcf9c439785514d6443c6af]` 子 Session 的有效权限由“Subagent 自身权限 + 子 Session 权限”共同形成。子 Session 权限继承父 Session 的 deny 和 `external_directory` 规则，并在 Subagent 没有明确 Task/Todo 能力时添加默认 deny；它不会把父 Agent 的全部限制直接复制过去。因此 Plan Agent 的 edit deny 本身不必然限制 `general` 子 Agent，但父 Session 上持久化的 deny 会形成硬上限。

证据：文件 `packages/opencode/src/agent/subagent-permissions.ts`；函数 `deriveSubagentSessionPermission`；位置 `14-27`；调用点 `packages/opencode/src/tool/task.ts:139-172`；执行时 Permission 合并点 `packages/opencode/src/session/tools.ts` 的 `SessionTools.resolve` `59-90`；Provider 可见 Tool 过滤点 `packages/opencode/src/session/llm/request.ts` 的 `resolveTools` `208-214`；版本完整 SHA。

测试：`packages/opencode/test/agent/plan-mode-subagent-bypass.test.ts`，测试 `subagent permissions take precedence over parent agent restrictions`、`subagent inherits parent session deny rules as hard runtime ceilings`，位置 `29-55`、`141-160`；版本完整 SHA。

### 9.6 结果怎样回到父 Session

`[Current default @ 0e3474509aa5ad16afcf9c439785514d6443c6af]` 前台 Task 等待子 Prompt 完成，从子 Session 最后一个 Text Part 取文本，包装成 `<task_result>`，作为父 Assistant Message 中 `task` Tool Part 的 completed output。父 `SessionPrompt` 随后 continuation，重载父历史，模型才看到该 Tool Result。子 Session 的完整历史仍留在子 Session，不会整体拼回父 Session。

证据：`packages/opencode/src/tool/task.ts`，`runTask` 与前台等待/返回，位置 `200-214`、`310-347`；父 Tool Part 结算见 `packages/opencode/src/session/processor.ts:383-413`；版本完整 SHA。测试：`packages/opencode/test/tool/task.test.ts:219-256` 与 `packages/opencode/test/session/prompt.test.ts:825-851`；版本同上。

### 9.7 Background Subagent 是实验能力

`[Current experimental @ 0e3474509aa5ad16afcf9c439785514d6443c6af]` `background: true` 需要 `OPENCODE_EXPERIMENTAL_BACKGROUND_SUBAGENTS=true`。它立即返回 running Task Result；完成后再向父 Session 注入一条 synthetic User Message 触发父 Prompt。父 Run 取消会递归取消关联的后台子任务。

证据：`packages/opencode/src/tool/task.ts`，flag gate `96-102`、结果注入 `216-253`、后台/前台分支 `256-347`；`packages/opencode/src/session/run-state.ts`，`cancelBackgroundJobs` `111-143`；版本完整 SHA。测试：`packages/opencode/test/tool/task.test.ts`，`cancelling the parent run cancels running background tasks` 与递归取消测试，位置 `897-933`、`957-984`；版本同上。

### 9.8 Subagent 深度限制

`[Current default @ 0e3474509aa5ad16afcf9c439785514d6443c6af]` Task Tool 沿 `parentID` 计算深度；默认 `subagent_depth ?? 1` 阻止 Subagent 再启动 Subagent。配置提高后可允许嵌套。这个限制只针对 Task 委派深度，不是 Session Loop 最大步数。

证据：`packages/opencode/src/tool/task.ts`，`TaskTool.execute`，位置 `104-117`；测试 `packages/opencode/test/tool/task.test.ts`，位置 `391-469`；版本完整 SHA。

## 10. 当前默认路径的可靠性与停止控制

### 10.1 Retry：重试同一个应用层 Provider Turn

`[Current default @ 0e3474509aa5ad16afcf9c439785514d6443c6af]` `Effect.retry(SessionRetry.policy(...))` 位于同一个 Processor Handle 内部。可重试错误会使用 Provider hints 或指数退避加 jitter，最多安排 5 次 retry；Context Overflow 不走这条 retry。重试期间更新 Session Status 为 `retry`。

关键边界：重试不会先回到 `SessionPrompt.run` 顶部重载历史，也不会创建新的 Assistant Message；它重跑同一 `llm.stream(streamInput)`，复用同一 Processor Context。已经持久化的部分流事件可能存在，因此不能把它描述为数据库层“完全无痕重放”。

证据：

- `packages/opencode/src/session/processor.ts`；`SessionProcessor.process`；`627-676`；完整 SHA。
- `packages/opencode/src/session/retry.ts`；`SessionRetry.retryable`、`policy`；`84-205`；完整 SHA。
- 测试 `packages/opencode/test/session/retry.test.ts`；`policy updates retry status and increments attempts`、`policy stops after five retries`；`97-148`；完整 SHA。

### 10.2 Interrupt：中断执行并尽力结算

`[Current default @ 0e3474509aa5ad16afcf9c439785514d6443c6af]` TUI 的 interrupt 动作最终调用兼容 `session.abort`；Handler 调用 `SessionPrompt.cancel -> SessionRunState.cancel`。取消当前 Runner 也取消关联 Background Jobs。Processor cleanup 尽力持久化部分 Text/Reasoning/Patch，把未完成 Tool 标为 interrupted error，并完成 Assistant Message。

证据：

- `packages/tui/src/component/prompt/index.tsx`；命令 `session.interrupt`；`393-419`；完整 SHA。
- `packages/opencode/src/server/routes/instance/httpapi/handlers/session.ts`；`SessionHttpApi.abort`；`232-235`；完整 SHA。
- `packages/opencode/src/session/prompt.ts`；`SessionPrompt.cancel`；`152-155`；完整 SHA。
- `packages/opencode/src/session/run-state.ts`；`SessionRunState.cancel`；`77-86`；完整 SHA。
- `packages/opencode/src/session/processor.ts`；`cleanup`、`halt`、`process`；`539-683`；完整 SHA。
- 测试 `packages/opencode/test/session/prompt.test.ts`；`cancel interrupts loop and resolves with an assistant message`、`cancel records MessageAbortedError on interrupted process`；`1123-1169`；完整 SHA。

中断在 retry backoff 期间也应快速响应；相关静态测试为 `packages/opencode/test/session/compaction.test.ts` 的 `stops quickly when aborted during retry backoff`，位置 `1202-1265`，版本完整 SHA。

### 10.3 `agent.steps` 不是当前旧路径的硬循环上限

`[Current default @ 0e3474509aa5ad16afcf9c439785514d6443c6af]` 当前旧路径每次未终止循环先 `step++`，当 `step >= agent.steps` 时只向 Model Messages 追加 `MAX_STEPS_PROMPT`。它没有因此清空 Tools、设置 `toolChoice: none` 或直接 `break`。如果模型仍生成 Tool Call，当前代码仍可能 continuation。

证据：文件 `packages/opencode/src/session/prompt.ts`；`SessionPrompt.run`；位置 `1132-1139`、`1178-1179`、`1279-1282`、`1334-1335`；版本完整 SHA。配置读取测试：`packages/opencode/test/agent/agent.test.ts`，含 `steps: 50` 的 Agent 配置测试位置 `306-320`；版本同上。

`[Interpretation]` 所以本基线不能把 `agent.steps` 写成“最多 N 次 Provider 调用”的硬保证。它是模型可见的最后步骤提醒；而且旧 Loop 的计数在同一个 `runLoop` 内不会因运行中新 User Message 自动重置。

### 10.4 Doom Loop：三次相同调用后询问

`[Current default @ 0e3474509aa5ad16afcf9c439785514d6443c6af]` Processor 在 `tool-call` 时检查当前 Assistant Message 最近 3 个 Part。若三者都是同名 Tool、非 pending 且 `JSON.stringify(input)` 相同，就请求 `doom_loop` Permission。内置默认规则是 ask。

这不是自动判定“任务失败”，也不是直接 break。用户或规则仍决定 allow/deny；若继续允许，代码没有全局永久封禁该 Tool。

证据：`packages/opencode/src/session/processor.ts`，常量 `DOOM_LOOP_THRESHOLD` 与 `handleEvent` Tool Call 分支，位置 `29-30`、`331-380`；`packages/opencode/src/agent/agent.ts`，默认 `doom_loop: "ask"`，位置 `119-136`；版本完整 SHA。测试仅覆盖默认 Permission：`packages/opencode/test/agent/agent.test.ts`，`default permission includes doom_loop and external_directory as ask`，位置 `469-475`；本轮未找到直接覆盖“三次相同 Tool Call 触发 ask”的测试，记为测试缺口。

### 10.5 停止条件汇总

`[Current default @ 0e3474509aa5ad16afcf9c439785514d6443c6af]` 当前 Loop 主要在这些边界停止：

- 顶部 terminal check 确认普通最终 Assistant 且无待 continuation 的本地 Tool Part。
- Processor 因 blocked、Assistant Error 返回 `stop`。
- Content Filter 或 Structured Output Error 使当前分支 `break`。
- Structured Output Tool 已成功产生目标对象，直接 `break`。
- Compaction processor 决定 `stop`。
- Interrupt 终止 Runner，外层调用以最后 Assistant Message 完成兼容返回。

证据：`packages/opencode/src/session/prompt.ts`，`SessionPrompt.run`，`1100-1130`、`1288-1339`；`packages/opencode/src/session/processor.ts:679-681`；版本完整 SHA。

不能仅凭 Provider finish reason 判断停止：本地 Tool Part、最新 User Message、Processor Error 和 Compaction 都会改变结论。

## 11. native V2：另一条已接线但非默认 TUI 的路径

### 11.1 入口确实已接线

`[V2 implemented @ 0e3474509aa5ad16afcf9c439785514d6443c6af]` native V2 HTTP Handler 调用 `SessionV2.prompt`；Server 组合层为 `SessionExecution` 提供 `SessionExecutionLocal`，所以它不是只有类型和孤立实现的死代码。

证据：

- `packages/server/src/handlers/session.ts`；Handler `session.prompt`；`139-171`；完整 SHA。
- `packages/core/src/session.ts`；`V2Session.prompt`；`360-386`；完整 SHA。
- `packages/server/src/routes.ts`；`applicationServices`、`makeRoutes`；`26-62`；完整 SHA。
- 当前 executable 的组合接线：`packages/opencode/src/server/routes/instance/httpapi/server.ts`；`createRoutes` Layer composition；`298-303`；完整 SHA。
- 测试：`packages/core/test/session-prompt.test.ts`，`durably admits one user message before transcript promotion`，`143-163`；完整 SHA。

但当前 TUI 普通提交仍使用第 1.1 节的兼容入口，因此状态不是 `[Current default]`。

### 11.2 Admission 与 execution 分离

`[V2 implemented @ 0e3474509aa5ad16afcf9c439785514d6443c6af]` `SessionV2.prompt` 先将 Prompt 作为 durable `session_input` admission 记录；`resume !== false` 才 advisory `wake`。Runner 到安全 Provider Turn 边界才把 eligible input promotion 成模型可见 User Message。

证据：`packages/core/src/session.ts:360-386`；`packages/core/src/session/input.ts`，`SessionInput.admit`、`promoteSteers`、`promoteNextQueued`，`41-81`、`245-288`；版本完整 SHA。规格解释：`specs/v2/session.md:5-50`；版本完整 SHA。

这项改变解决的是“接受输入”与“立刻执行模型”耦合的问题：admitted Prompt 可以保持 pending，`resume:false` 可只记录不执行，steer/queue 的 promotion 时机也成为显式状态。

### 11.3 V2 Agent 与 Model

`[V2 implemented @ 0e3474509aa5ad16afcf9c439785514d6443c6af]` Agent Registry、内置 build/plan/general/explore、V2 Agent 配置 schema 和运行时选择已经实现。Runner 每个 Provider Turn 调用 `agents.select(session.agent)`；Session 无 Agent 时选择可见默认 Agent，最终 fallback ID 是 `build`。

证据：`packages/core/src/agent.ts`，`AgentV2.select` 与默认选择，`67-105`；`packages/core/src/plugin/agent.ts`，内置 Agent，`96-202`；`packages/core/src/config/agent.ts`，`ConfigAgent.Info`，`13-25`；Runner 调用点 `packages/core/src/session/runner/llm.ts:179-203`；版本完整 SHA。测试：`packages/core/test/agent.test.ts:14-38,102-130`；版本同上。

`[V2 partial @ 0e3474509aa5ad16afcf9c439785514d6443c6af]` V2 `build` 已配置默认 Agent system、primary mode、`question`/`plan_enter` 权限；`plan` 已配置 primary mode、`plan_exit` 与 plan 文件编辑例外，并拒绝一般 edit。但 V2 shipped built-ins 明确把 `task` 和 `plan_exit` 列为尚待 port，parity 表也把 plan/build switch reminder 标为 missing。因此这些 Agent 的权限轮廓已存在，不代表完整 Plan -> Build 工作流已可用。

证据：`packages/core/src/plugin/agent.ts`，内置 `build`、`plan` transform，`120-150`；`packages/core/src/tool/builtins.ts`，`BuiltInTools.node` 与 remaining leaves TODO，`18-48`；`specs/v2/session.md:137-144`；版本完整 SHA。

`[V2 partial @ 0e3474509aa5ad16afcf9c439785514d6443c6af]` V2 Agent 配置可以保存 `model`、`variant` 和 `request`，但当前 Runner 的 `models.resolve(session)` 只接收 Session，`SessionRunnerModel.locationLayer.resolve` 只看 `session.model` 或 Catalog default/首个 supported Model；没有应用 `agent.info.model` 或 `agent.info.request`。因此“Agent-local Model/Request 已完整控制 Provider Turn”在本基线不成立。

证据：`packages/core/src/config/plugin/agent.ts`，Agent 配置物化 `80-111`；`packages/core/src/session/runner/llm.ts:182-214`；`packages/core/src/session/runner/model.ts`，`SessionRunnerModel.locationLayer.resolve`，`181-215`；版本完整 SHA。规格也把 Agent request settings 标为 partial：`specs/v2/session.md:137-143`；版本同上。

### 11.4 V2 Provider Turn 与 continuation

观察重点：V2 没有调用旧 `SessionPrompt.loop`，而是自己显式执行“一次 stream + Tool settlement + history reload”。

```ts
while (shouldRun) {
  let needsContinuation = true
  let step = 1
  while (needsContinuation) {
    const result = yield* runTurn(input.sessionID, promotion, step)
    needsContinuation = result.needsContinuation
    step = result.step + 1
    promotion = "steer"
    if (!needsContinuation) needsContinuation = yield* SessionInput.hasPending(db, input.sessionID, "steer")
  }
  shouldRun = yield* SessionInput.hasPending(db, input.sessionID, "queue")
  promotion = shouldRun ? "queue" : undefined
}
```

文件：`packages/core/src/session/runner/llm.ts`；函数：`SessionRunner.run`；位置：`383-406`；版本：完整 SHA。

`[V2 implemented @ 0e3474509aa5ad16afcf9c439785514d6443c6af]` 每次 `runTurnAttempt` 只有一个显式 `llm.stream(request)`；完整本地 Tool Call 先 durable 记录，再并发执行并等待 settlement；需要 continuation 时下一次 `runTurn` 重载 Session History。

证据：`packages/core/src/session/runner/llm.ts`，`runTurnAttempt`，`173-348`；测试 `packages/core/test/session-runner.test.ts`，`continues with reloaded history after durably settling one local tool call`，`1462-1518`；版本完整 SHA。

### 11.5 V2 steer、queue 与步骤重置

`[V2 implemented @ 0e3474509aa5ad16afcf9c439785514d6443c6af]` steer 在当前 drain 仍需 continuation 时于下一安全边界 promotion；queue 要等当前 continuation 将结束时才 FIFO promotion 一个。promotion 新输入会把当前 Agent 的步骤计数重置为 1。

证据：`packages/core/src/session/runner/llm.ts:187-196,383-405`；`packages/core/src/session/input.ts:245-288`；测试 `packages/core/test/session-runner.test.ts`，`steers an active provider turn with newly recorded prompts`、`promotes queued input after continuation ends`，`1865-1953`；版本完整 SHA。

### 11.6 V2 的 `steps` 是更强的 Provider-Turn allowance

`[V2 implemented @ 0e3474509aa5ad16afcf9c439785514d6443c6af]` 当 `currentStep >= agent.steps`，V2 不只追加 `MAX_STEPS_PROMPT`，还不物化 Tools，并把 `toolChoice` 设为 `none`。即使 Provider 违规返回本地 Tool Call，也会将其失败结算而不设置 Tool continuation。新 steer promotion 会重置 allowance。

证据：`packages/core/src/session/runner/llm.ts:202-214,243-249,383-405`；测试 `packages/core/test/session-runner.test.ts`，`forces a text response on an agent's configured final step`、`resets the configured step allowance when steering input promotes`，`3008-3106`；版本完整 SHA。

准确表述应是“V2 用 Agent steps 限制一次输入批次触发的工具型 Provider continuation，并保留最后一个 text-only Provider Turn”，而不是简单说“整个 Session 最多 N 步”。新的 steer/queue 输入可开始新的 allowance；Compaction 的物理重建也不应与新的逻辑步骤混为一谈。

`[V2 implemented @ 0e3474509aa5ad16afcf9c439785514d6443c6af]` V2 的普通停止判定由 continuation 状态而不是单独的 Provider finish reason 驱动：本地 Tool Call 会把 `needsContinuation` 设为 true；没有本地 Tool continuation 时，仅有 pending steer 才继续内层循环；内层结束后一次 promotion 一个 pending queue。Provider Error 会阻止 Tool continuation，用户拒绝 Permission/Question 会中断，最后步骤中违规 Tool Call 会失败结算但不继续。

证据：`packages/core/src/session/runner/llm.ts`，`runTurnAttempt` 的 Tool/错误结算与 `SessionRunner.run`，`231-347`、`383-405`；测试 `packages/core/test/session-runner.test.ts`，Tool continuation `1462-1518`、steer/queue `1865-1953`、final step `3008-3054`、Provider Error `3108-3147`；版本完整 SHA。

### 11.7 V2 Interrupt 已实现，但只保证进程内所有权

`[V2 implemented @ 0e3474509aa5ad16afcf9c439785514d6443c6af]` `SessionExecutionLocal` 使用 process-global `SessionRunCoordinator`：同 Session resume join、wake coalesce、不同 Session 可并行；interrupt 中断本进程 owner，idle 是 no-op，并清除已有 pending wake。Runner 对活跃 Tool settlement/Assistant 做 durable failure 结算。

证据：

- `packages/core/src/session/execution/local.ts`；Layer `SessionExecutionLocal`；`10-46`；完整 SHA。
- `packages/core/src/session/run-coordinator.ts`；`SessionRunCoordinator.make`；`24-104`；完整 SHA。
- `packages/core/src/session/runner/llm.ts`；中断结算 `277-347`；完整 SHA。
- 测试 `packages/core/test/session-run-coordinator.test.ts`；`interrupts active execution and clears its pending wake`；`218-245`；完整 SHA。
- 测试 `packages/core/test/session-runner.test.ts`；`durably fails blocked local tools when interrupted while awaiting settlement`；`2971-3006`；完整 SHA。

`[V2 missing/planned]` durable/clustered interruption、跨进程 owner fencing 和 post-crash continuation recovery 尚未完成。证据：`specs/v2/session.md:101-109,153-169` 与 `specs/v2/todo.md:50-74`；版本完整 SHA。

### 11.8 V2 Retry 与 Doom Loop 仍缺失

`[V2 missing/planned]` native V2 没有当前旧路径 `SessionRetry.policy` 的一般 Provider retry/backoff/status 等价实现，也没有重复相同 Tool Call 的 Doom Loop 等价保护。Runner 源码 TODO 明确列出“Bound provider retries and repeated identical tool calls”；规格明确把 Provider timeout/retry/watchdog 延后。

证据：`packages/core/src/session/runner/llm.ts`，Runner 状态清单 `43-90`，尤其 `55`；`specs/v2/session.md:153-165`；`specs/v2/todo.md:53-74`；版本完整 SHA。

不要把 V2 的“一次 Context Overflow 后完成 Compaction 并重建一次物理请求”写成通用 Retry。它只在无 durable Assistant output/tool side effect 的特定 overflow 条件下发生一次。实现证据：`packages/core/src/session/runner/llm.ts:277-288,355-380`；规格 `specs/v2/session.md:111-121`；版本完整 SHA。

### 11.9 V2 Todo 已实现，Task/Subagent 编排未实现

`[V2 implemented @ 0e3474509aa5ad16afcf9c439785514d6443c6af]` Core V2 已有 `SessionTodo` 和 permission-checked `todowrite` Tool，使用同一 Todo 表并发布更新。

证据：`packages/core/src/session/todo.ts`，`SessionTodo.update/get`，`26-78`；`packages/core/src/tool/todowrite.ts`，Tool registration/execution，`25-62`；测试 `packages/core/test/tool-todowrite.test.ts:85-124`；版本完整 SHA。

`[V2 missing/planned]` native V2 虽然有 `mode: "subagent"` 的 Agent 数据，也保留 `Session.Info.parentID`，但 `SessionV2.create` 的 `CreateInput` 不接受 `parentID`，Core Tool Registry 没有 Task Tool，规格仍写着正在 modeling subagents，background agent dispatch 也列为下一 slice。因此 native V2 父子 Session 创建、权限/Model 继承、结果回父 Session均不能标为已实现。

证据：`packages/core/src/session.ts`，`CreateInput` 与 `V2Session.create`，`79-84`、`208-262`；`packages/schema/src/session.ts`，`Session.Info.parentID`，`18-44`；`packages/core/src/tool/builtins.ts`，`BuiltInTools.node` 的 Task port TODO，`18-48`；`specs/v2/todo.md:11-14,50-54`；版本完整 SHA。当前 V2 没有对应 Task/Subagent orchestration 测试，这是与旧 `packages/opencode/test/tool/task.test.ts` 对照后的明确测试缺口。

## 12. V1/V2 对照状态表

| 能力 | 当前默认旧编排 | native V2 | 结论 |
| --- | --- | --- | --- |
| 默认 TUI 普通消息入口 | implemented/default | 已有独立 HTTP 入口但 TUI 未使用 | 迁移共存，不是 V2 已接管 |
| Agent Registry 与 build/plan | implemented | implemented | V2 配置/Registry 已有，但工作流 parity 不完整 |
| Agent-local Model/Request | implemented | partial | V2 Runner 当前只按 Session/Catalog resolve Model |
| 外层多 Provider Turn | `SessionPrompt.run` | `SessionRunner.run` | 两边都是显式循环，V2 不桥接旧 Loop |
| Tool Result continuation | implemented | implemented | 两边下一轮都重载投影历史 |
| steer/queue 语义 | 运行中新 Prompt 被下轮看到，无独立 durable inbox vocabulary | implemented | V2 把 admission/promotion 和 delivery 显式化 |
| Todo | implemented | implemented | 都是 Session 级替换清单，不是调度器 |
| Task/Subagent | implemented；后台模式 experimental | missing/planned | 不能因 V2 Agent 有 `subagent` mode 就宣称已可委派 |
| Provider Retry | 最多 5 次 policy retry | missing/planned | Overflow 单次恢复不是通用 Retry |
| Interrupt | process-local Runner + cleanup | process-local Coordinator + durable settlement | V2 clustered/durable ownership仍缺失 |
| `agent.steps` | 只追加提醒，不硬禁 Tool | text-only final turn，Tools disabled | 语义明显不同 |
| Doom Loop | 三次同 Tool/同参数触发 Permission ask | missing/planned | V2 TODO 明确列出 |
| post-crash continuation | 没有完整 durable attempt/recovery 模型 | missing/planned | V2 明确拒绝从 advisory wake 猜测安全重试 |

### 12.1 V2 设计改变解决什么问题

`[Interpretation]` 以下不是因为“用了 Effect 所以更先进”，而是由具体状态边界支撑：

- durable admission 将“请求已接受”与“Provider 已执行”分离，可表达 pending steer/queue。
- process-global Coordinator 与 Location-scoped Runner 分离了 Session 并发所有权和实际运行环境。
- 每个 Provider Turn 显式一次 `llm.stream`，Tool Call 先 durable 记录再执行，降低 continuation 对纯内存状态的依赖。
- V2 `steps` 同时控制 Tool advertisement 与 `toolChoice`，比旧路径仅提示模型更可执行。
- V2 对不知道是否已发生副作用的 crash 场景选择明确标记 missing，而不是自动重试造成重复副作用。

这些改进目标的代价是 parity 尚未完成：Task/Subagent、一般 Retry、Doom Loop、部分 Agent/Model request policy、Plan/Build switch reminders 和多类 Tool/Plugin Context 仍缺失或 partial。

## 13. 边界、失败场景和易混淆点

### 13.1 静态确认、规格计划与解释

- `[Current default]`、`[V2 implemented]`：有入口/调用点、核心实现和测试支撑。
- `[V2 partial]`：代码覆盖一部分，但调用链或 parity 表显示缺口。
- `[V2 missing/planned]`：规格或源码 TODO 明确未完成；不能改写成用户已可用行为。
- `[Interpretation]`：用于解释业务意义，不替代实现证据。

### 13.2 典型失败场景

1. Agent 名不存在：当前旧路径发布 Error 并失败；V2 `AgentV2.select` 对显式未知 ID 返回 `{ id, info: undefined }`，Runner仍可能用 fallback Model 和空 Agent policy 继续，这一行为需要实验确认是否符合预期。
2. Model 不存在：旧路径 `getModel` 记录提示后失败；V2 返回 `ModelUnavailableError` 或 `ModelNotSelectedError`。
3. Tool 被拒绝：旧 Processor 可 blocked 并 stop；V2 用户 decline 会 interrupt continuation，而普通 ToolFailure 可作为 Tool Result 让模型继续。
4. Provider 暂时错误：旧路径按 policy retry；V2 当前一般错误 terminal，没有通用 retry。
5. 中断时 Tool 正在运行：两边都尝试错误结算，但外部进程或网络副作用可能已发生，持久化 error 不等于回滚副作用。
6. Crash 发生在 Tool 或 Provider 的未知结果区间：V2 明确没有自动 post-crash continuation recovery。
7. `task_id` 指向无关 Session：当前 Task Tool 未显式验证父子所有权，需最小实验和安全审计。

### 13.3 最容易混淆的十点

1. Provider Turn 不等于一次完整用户请求。
2. Session Drain 不等于 durable Job 或 Message。
3. Tool Call 不等于 Tool Result。
4. Task Tool 不等于 Subagent；它只是启动/恢复 Subagent 的委派入口。
5. `SubtaskPart` 不等于普通模型 Tool Call。
6. Todo 不会驱动主循环，也不是 Background Job。
7. Permission 不等于 OS Sandbox。
8. 当前旧路径的 `agent.steps` 不是硬循环上限。
9. native LLM adapter 不等于 native V2 Session Runner。
10. V2 存在 Agent `mode: subagent` 不等于 V2 已实现父子 Session 委派。

## 14. 规格与实现冲突记录

### 14.1 V2 Agent system prompt parity 表可能滞后

`[Unresolved]` `specs/v2/session.md:137` 说 V2 仍需 apply agent system prompt，但 Runner 已在 `packages/core/src/session/runner/llm.ts:208-210` 把 `agent.info?.system` 放入 request system，测试也覆盖默认与显式 Agent。可能的解释是 parity 行想表达“Agent prompt 与完整 request policy 整体仍 partial”，而不是 system 字段完全未接线。正式文档应保留 `partial`，但不要写“V2 完全没有 Agent system”。测试证据：`packages/core/test/session-runner.test.ts`，`includes the effective default agent system before durable context`、`uses an explicitly selected non-build agent system`，位置 `775-849`；版本完整 SHA。

### 14.2 `specs/v2/todo.md` 的 Tool settlement next slice 与实现进度可能滞后

`[Unresolved]` `specs/v2/todo.md:38-45` 仍把 eager Tool settlement 写为 next slice，而 Runner 注释、实现和测试已覆盖 durable call、eager execution、await settlement 与 history reload。应以基线 E1 代码/测试把该项标为 implemented，同时在任务 6 审计规格更新时间。

## 15. 关键源码索引

所有条目版本均为 `0e3474509aa5ad16afcf9c439785514d6443c6af`。

| 主题 | 文件 | 函数/符号 | 行号 |
| --- | --- | --- | --- |
| 默认 TUI 入口 | `packages/tui/src/component/prompt/index.tsx` | `submitInner()` | `1092-1146` |
| 兼容 HTTP Prompt | `packages/opencode/src/server/routes/instance/httpapi/handlers/session.ts` | `SessionHttpApi.prompt` | `295-309` |
| 旧 Agent Registry | `packages/opencode/src/agent/agent.ts` | `Info`、Agent Layer、`defaultInfo` | `35-55`, `98-351` |
| Agent/Model admission | `packages/opencode/src/session/prompt.ts` | `currentModel`、`createUserMessage` | `614-689` |
| 旧外层循环 | `packages/opencode/src/session/prompt.ts` | `SessionPrompt.run` | `1081-1341` |
| Provider request | `packages/opencode/src/session/llm/request.ts` | `LLMRequestPrep.prepare`、`resolveTools` | `56-214` |
| Provider stream | `packages/opencode/src/session/llm.ts` | `LLM.run`、`LLM.stream` | `85-381` |
| Processor/doom/stop | `packages/opencode/src/session/processor.ts` | `handleEvent`、`process` | `278-537`, `627-683` |
| Retry | `packages/opencode/src/session/retry.ts` | `retryable`、`policy` | `84-205` |
| Interrupt/serialization | `packages/opencode/src/session/run-state.ts` | `cancel`、`ensureRunning` | `52-94`, `111-143` |
| Todo | `packages/opencode/src/session/todo.ts` | `Todo.update/get` | `29-66` |
| Task/Subagent | `packages/opencode/src/tool/task.ts` | `TaskTool.execute/runTask` | `81-347` |
| 子权限 | `packages/opencode/src/agent/subagent-permissions.ts` | `deriveSubagentSessionPermission` | `14-27` |
| V2 HTTP Handler | `packages/server/src/handlers/session.ts` | `session.prompt` | `139-171` |
| V2 admission | `packages/core/src/session.ts` | `V2Session.prompt` | `360-386` |
| V2 execution routing | `packages/core/src/session/execution/local.ts` | `SessionExecutionLocal` Layer | `10-46` |
| V2 Coordinator | `packages/core/src/session/run-coordinator.ts` | `SessionRunCoordinator.make` | `24-104` |
| V2 Agent | `packages/core/src/agent.ts` | `AgentV2.select` | `67-105` |
| V2 Build/Plan 与缺失 Tool | `packages/core/src/plugin/agent.ts`、`packages/core/src/tool/builtins.ts` | `AgentPlugin.Plugin`、`BuiltInTools.node` | `120-150`, `18-48` |
| V2 Model | `packages/core/src/session/runner/model.ts` | `SessionRunnerModel.locationLayer.resolve` | `181-215` |
| V2 Runner | `packages/core/src/session/runner/llm.ts` | `runTurnAttempt`、`SessionRunner.run` | `173-406` |
| V2 Todo Tool | `packages/core/src/tool/todowrite.ts` | Tool registration/execution | `25-62` |

## 16. 关键测试证据

任务 3-5 初稿先做静态阅读；任务 7 已实际执行指定测试和一个临时隔离测试，命令与结果见第 18.1 节。所有测试版本均为完整基线 SHA。

| 测试 | 行号 | 证明内容 |
| --- | --- | --- |
| `packages/opencode/test/agent/agent.test.ts`，build/plan/general/explore tests | `47-181` | 当前内置 Agent 角色与权限差异 |
| `packages/opencode/test/session/prompt.test.ts`，`loop continues when finish is tool-calls` | `825-851` | 本地 Tool Result 后第二次 Provider Call |
| 同文件，`loop continues when finish is stop but assistant has tool parts` | `892-918` | finish reason 不是唯一停止依据 |
| 同文件，`cancel interrupts loop...` | `1123-1169` | 当前旧路径中断和 Abort Error |
| 同文件，`prompt submitted during an active run...` | `1405-1469` | 活跃 Run 中新 User Message 进入下一轮 |
| `packages/opencode/test/session/retry.test.ts`，policy tests | `97-148` | retry status 与最多 5 次 retry |
| `packages/opencode/test/tool/task.test.ts`，resume/create/depth/permission tests | `219-519` | 子 Session、恢复、深度和权限塑形 |
| `packages/opencode/test/agent/plan-mode-subagent-bypass.test.ts` | `29-160` | 父 Agent 限制、父 Session deny 与子 Agent 权限边界 |
| `packages/core/test/session-prompt.test.ts`，durable admission test | `143-163` | V2 Prompt admission 与 transcript promotion 分离 |
| `packages/core/test/session-runner.test.ts`，local Tool continuation test | `1462-1518` | V2 Tool settlement 后重载历史 continuation |
| 同文件，effective Agent system tests | `775-849` | V2 默认与显式 Agent system 已进入请求 |
| 同文件，steer/queue tests | `1865-1953` | V2 delivery 边界 |
| 同文件，interrupt settlement test | `2971-3006` | V2 中断 Tool durable failure |
| 同文件，final step tests | `3008-3106` | V2 Tool disable 与 steer 重置 allowance |
| `packages/core/test/session-run-coordinator.test.ts` | `209-245`, `351-369` | idle interrupt、active interrupt、join waiter 边界 |
| `packages/core/test/tool-todowrite.test.ts` | `85-124` | V2 Todo 权限、持久化和 typed output |

## 17. Open Questions

1. 当前 Task Tool 的 `task_id` 是否应强制验证目标 Session 是当前父 Session 的 child，还是允许跨父恢复是有意设计？
2. `[Partially answered by task 7]` 旧路径 `agent.steps` 达阈值后，真实 Provider 若继续发 Tool Call，会持续多少轮；是否有未定位到的 Provider/AI SDK stop 条件？静态代码没有硬 break。隔离 Fake Provider 已证明 `steps: 1` 后至少还能连续完成两次 Tool continuation，并发生 3 次 Provider 调用，但没有证明无限继续或所有真实 Provider adapter 的行为。
3. Doom Loop 的“三个最近 Part”在并行 Tool Call、夹杂 Text/Reasoning Part、Provider 多 step 输出下是否达到产品预期？缺直接测试。
4. V2 显式未知 Agent 得到 `info: undefined` 后继续运行是否是预期 fallback，还是应当像旧路径一样失败？
5. V2 Agent-local `model`、`variant`、`request` 何时接入 `SessionRunnerModel.resolve` 和 request assembly？
6. V2 parity 表关于 Agent system prompt 的文字是否已滞后？
7. V2 Task/Subagent 将如何定义父权限硬上限、Model 继承、结果 delivery、后台状态和 clustered cancellation？
8. 旧 Retry 在一次失败前已经 durable 写入部分 Text/Tool Event 时，下一物理 attempt 的 Provider/Projection 行为是否会形成重复 Part？需要 Recorded Provider 实验。
9. 当前和 V2 对 providerExecuted Tool 的 continuation 停止规则是否在所有 Provider adapter 上一致？

## 18. 建议最小实验

以下是任务 7 的候选实验；本轮按优先级完成第 1 项的最小隔离验证，其余仍保留为后续实验，不调用付费 Provider。

1. **旧 `agent.steps` 实验**：配置 `steps: 1`，Fake Provider 连续返回 Tool Call，确认 Tools 仍被发送、Loop 是否继续，并记录 Provider 调用数。
2. **Doom Loop 实验**：在一个 Assistant Provider stream 中生成三次同 Tool/同参数调用，监听 Permission Asked；再插入 Text Part 或改变对象 key 顺序，验证 JSON 字符串比较边界。
3. **Task 所有权实验**：父 Session A 用 `task_id` 指向父 Session B 的 child，确认当前是否允许，并评估是否构成权限/信息边界问题。
4. **Retry 部分输出实验**：第一次 Provider attempt 先产生 Text Delta/whole Part 再抛 503，第二次成功，检查同一 Assistant Message 的 Part 和请求历史。
5. **V2 Agent Model 实验**：Session 不设 Model，给 build Agent 配显式 Model，记录 `LLM.request.model`，验证静态结论“当前未应用 agent.model”。
6. **V2 unknown Agent 实验**：`switchAgent` 到不存在 ID 后 resume，记录 System、Tools、Permission 和最终错误/成功行为。
7. **V2 crash boundary 模拟**：只用已有测试 Layer 中断 provider/tool settlement，确认 pending input、promoted history 和显式 resume 的差异，不声称自动 recovery。

### 18.1 任务 7 最小验证（2026-08-18）

#### 环境与边界

- 源码仓库：`/home/wuzhongyun/projects/Intern_projects/Opencode_learn/opencode github code`；HEAD 为 `0e3474509aa5ad16afcf9c439785514d6443c6af`。
- 平台：Linux；Bun 不在 PATH，所有命令均使用 `npx --yes bun ...`；实际 Bun 版本为 `1.3.14 (0d9b296a)`。
- 测试严格从 `packages/opencode` 或 `packages/core` 对应目录运行，没有从仓库根运行。
- 没有设置真实 Provider 密钥，没有调用真实或付费 Provider，没有网络业务调用，也没有运行破坏性 Tool。隔离实验只使用进程内本地 loopback `TestLLMServer` 和无匹配文件的 `glob`。
- 隔离实验使用唯一 untracked 临时文件 `packages/opencode/test/session/task7-agent-steps-isolation-20260818.test.ts`；完成后删除。实验前后源码 `git status --short` 均为空。

#### 命令与实际结果

耗时同时记录 Bun 输出的测试耗时和 `/usr/bin/time -p` 的 wall time；二者差异包含 `npx`/Bun 启动、测试层构建和清理。

| 工作目录 | 命令 | 结果 | 耗时 | 失败原因或说明 |
| --- | --- | --- | --- | --- |
| `packages/opencode` | `npx --yes bun test test/agent/agent.test.ts test/session/retry.test.ts test/tool/task.test.ts test/agent/plan-mode-subagent-bypass.test.ts test/session/prompt.test.ts` | `178 pass, 1 skip, 1 fail`；共 180 tests/5 files，486 assertions | Bun `90.51s`；wall `100.98s` | 唯一失败为 `cancel interrupts loop queued behind shell`，达到该测试的 `30000ms` 超时；其他指定用例通过。 |
| `packages/core` | `npx --yes bun test test/session-run-coordinator.test.ts test/tool-todowrite.test.ts` | `18 pass, 0 fail`；共 18 tests/2 files，32 assertions | Bun `1.233s`；wall `11.95s` | 全部通过。 |
| `packages/opencode` | `npx --yes bun test test/session/prompt.test.ts --test-name-pattern "cancel interrupts loop queued behind shell"` | `1 pass, 0 fail, 57 filtered out`；3 assertions | Bun `2.89s`；wall `13.56s` | 定向重跑通过，说明组跑失败未稳定复现；只能将首次失败归类为本地组跑时的超时波动，不能据此证明不存在竞态。 |
| `packages/opencode` | `npx --yes bun test test/session/task7-agent-steps-isolation-20260818.test.ts` | `0 pass, 1 fail` | Bun `6.58s`；wall `19.48s` | 临时测试第一次使用框架默认 `5000ms` test timeout，测试体尚未完成即超时；这是实验 harness 超时，不是产品断言失败。 |
| `packages/opencode` | `npx --yes bun test test/session/task7-agent-steps-isolation-20260818.test.ts` | `1 pass, 0 fail`；6 assertions | Bun `9.76s`；wall `25.80s` | 仅将临时测试级 timeout 提高到 `30000ms` 后重跑；产品配置、Fake Provider 响应和断言未改变。 |

#### 结果证明什么

- 指定 opencode 测试中，Agent 配置/权限、Retry policy、Task/Subagent 生命周期与深度、Plan 到 Subagent 权限边界、Session continuation/interrupt 等既有覆盖除一个组跑超时外均通过；该超时定向重跑通过。
- 指定 core 测试确认本基线的 process-local `SessionRunCoordinator` 行为和 V2 `todowrite` 行为在本环境全部通过。
- 优先级最高的旧 `agent.steps` 隔离实验配置 `build.steps: 1`。Fake Provider 依次返回两个不同参数的 `glob` Tool Call，再返回最终文本；实际发生 3 次 Provider 调用并正常停止。
- 三次请求 body 都包含非空 `tools`；前两次请求的 messages 都包含 `MAX_STEPS_PROMPT` 对应的 maximum steps 提醒。因 `step >= 1` 从首轮即成立，这直接验证旧路径达到阈值后只是追加提醒，并未禁用 Tool 或硬停止 continuation。

#### 不能证明什么

- 实验由第三次 Fake Provider 响应主动返回文本而终止，只建立“`steps: 1` 仍可达到 3 次 Provider 调用”的下界，不能证明 Loop 可无限运行，也不能排除 Context Overflow、Permission、Doom Loop、Provider adapter 或其他停止条件在更多轮后终止。
- 本轮没有调用真实 Provider，因此不能外推所有 AI SDK/Provider adapter 的 finish reason、Tool Call 或 stop 行为。
- 定向重跑通过不能消除 `cancel interrupts loop queued behind shell` 的潜在时序风险；只说明首次 30 秒超时不是本环境中的稳定失败。
- 本轮没有执行 Doom Loop 三次同调用或跨父 `task_id` 所有权实验，也没有完成任务 8 Teach-back。

#### Open Question 状态变化

- Open Question 2 从“仅静态不确定”变为“部分回答”：旧 `agent.steps` 已实验证明不是 1 次 Provider Turn 的硬上限，阈值后 tools 仍发送，且至少可继续到 3 次调用；“真实 Provider 会持续多少轮”和是否存在 adapter-specific 上限仍未解决。
- Open Question 1（跨父 `task_id` 所有权）与 Open Question 3（Doom Loop 三次同调用及 Part 排列边界）本轮未实验，状态不变。
- 其余 Open Questions 状态不变；任务 6 交叉审计和任务 8 Teach-back 仍待完成。

## 19. 理解检查题

1. 为什么一次“修复 Bug”的用户输入会有两个或更多 Provider Turn？请指出 Tool Result 在哪一层触发 continuation。
2. 当前默认路径中，Agent 和 Model 在 User Message 创建时各自按什么优先级确定？
3. `build` 与 `plan` 的差异为什么主要是 Harness 规则，而不是换了一个必然更谨慎的 Model？
4. Task Tool、Subagent、子 Session、`SubtaskPart` 和 Todo 分别是什么？
5. 前台 Subagent 的完整历史会不会复制回父 Session？父模型实际看到什么？
6. 为什么父 Plan Agent 的 edit deny 不一定自动限制 general Subagent，而父 Session deny 可以形成硬上限？
7. 当前旧路径 `agent.steps` 与 native V2 `steps` 的关键差异是什么？
8. Doom Loop 为什么是“等价保护的一部分”而不是数学意义上的无限循环证明？
9. V2 的 `resume:false`、steer 和 queue 分别控制什么？
10. 为什么 V2 的 Context Overflow 单次重建不能称作已实现通用 Retry？
11. 哪些事实证明 native V2 已接线？哪些事实又证明它尚未接管当前 TUI 普通消息？
12. 如果进程在 Tool 已产生外部副作用、但 durable settlement 前崩溃，为什么 Harness 不能安全地盲目重试？

## 20. 本轮结论

1. `[Current default]` 当前普通 TUI 的 Agent Loop 仍由兼容 `SessionPrompt.run` 驱动；EventV2 等新基础设施的复用没有改变这一入口判断。
2. `[Current default]` 一个用户请求可包含多个显式 Provider Turn；普通本地 Tool Result durable 后，外层 Loop 重载历史并 continuation。
3. `[Current default]` Task Tool 通过独立子 Session 实现 Subagent；Model、权限、结果回传和取消都有专门规则，不能与普通 Tool 或 Todo 混同。
4. `[Current default]` 旧 `agent.steps` 只是模型提醒；Doom Loop 是三次同调用后的 Permission ask；Retry 是同一 Assistant/Provider Turn 内最多 5 次 policy retry。
5. `[V2 partial]` native V2 已实现 durable admission、独立 Runner、Tool continuation、steer/queue、process-local interrupt、Todo 和更强的 step allowance；Task/Subagent、一般 Retry、Doom Loop、clustered recovery 和部分 Agent/Model request parity 仍 missing/partial。

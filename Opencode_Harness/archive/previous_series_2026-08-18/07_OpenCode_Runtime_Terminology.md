# OpenCode Runtime 术语表

> **源码基线**：`0e3474509aa5ad16afcf9c439785514d6443c6af`（`dev`），权威术语来源为仓库根 `CONTEXT.md`。
>
> **范围说明**：本文同时标注当前默认旧 Session Runtime 与 native V2 Session Runtime。`CONTEXT.md` 中出现的 V2 术语不自动代表当前 TUI 默认实现。
>
> **验收状态**：任务 6 的最终交叉审计尚未完成；任务 8 按用户指示跳过，未作用户理解验收。本文不把这两项写成已完成或已验收。
>
> **引用约定**：除非另有说明，下文所有源码、设计文档与测试行号均对应上述完整 SHA；路径均相对于固定源码根。

系列导航：[06 Harness 总览](./06_Harness.md) · [08 Agent 与 Orchestration](./08_Agent_and_Orchestration.md) · [09 Context 与 Persistence](./09_Context_and_Persistence.md) · [10 Tools 与 Security](./10_Tools_and_Security.md) · [11 Runtime Boundary](./11_Runtime_Boundary.md) · [12 V1/V2 对照](./12_V1_V2_Comparison.md)

本文是全系列的查阅型术语表，不重复各专题的完整调用链、测试过程或配置步骤。需要理解机制时转到上面的专题；需要确认一个词能否互换时回到本文。

## 系列基础版本标签

以下标签描述固定 SHA 下的调用路径、兼容职责或 Session 架构，不是 OpenCode 产品版本：

| 标签 | 本系列简明定义 |
| --- | --- |
| **current default（当前默认）** | 当前 TUI 普通消息实际进入的默认路径：compatibility Session API 与旧 `SessionPrompt` orchestration。它是“默认调用到哪里”的运行事实。 |
| **compatibility（兼容层/兼容路径）** | 为当前 TUI、旧 API/SDK、旧 Message/Part/Event、存储投影和迁移保留的合同与桥接层；它可以复用 current 基础设施，但不因此成为 native V2。 |
| **native V2** | 由 native `/api`、`V2Session.prompt`、durable admission、`SessionExecution` 与 `SessionRunner` 组成的独立 Effect-native 路径；某个 slice 已实现不表示当前默认已切换，也不表示 parity 完成。 |
| **架构 V1/V2** | 本系列比较 Session Runtime 时使用的架构简称：V1 指当前兼容旧 Session Runtime，V2 指 replacement architecture。它们不对应产品主版本号。 |

产品版本号、Git tag、目录名、文件名、生成类名或单独出现的 `V1`/`V2` 字样都不是架构归属或默认启用的充分证据；必须沿调用者、路由、Handler 和 Core 调用链核对。完整判据见 [12 V1/V2 对照](./12_V1_V2_Comparison.md)。

## 1. 关系图

先按以下关系读图；这些说明只解释连线，第 3、4 节再按 `CONTEXT.md` 是否有正式条目，分别标注“官方定义翻译”或“本文工作定义”：

- **上下文链**：位置（Location）决定注册表（Registry）的作用域；Registry 有序组合上下文源（Context Sources）；初始化产生基线系统上下文（Baseline System Context）与上下文快照（Context Snapshot），后续比较在同一上下文纪元（Context Epoch）内形成时间序更新。
- **输入链**：提示接纳（Prompt Admission）只把输入放入 durable inbox；提示提升（Prompt Promotion）才把 User Message 加入会话历史（Session History），并发生在安全提供商轮次边界（Safe Provider-Turn Boundary）。
- **Provider 链**：一个 Provider Turn 严格等于一次物理 Provider 请求及该请求的响应投影。Retry 每次重新发出的请求各自构成新的 Provider Turn，即使它们复用同一 Assistant Message 或 Processor context。
- **工具链**：对 Core 执行的本地工具，主链统一为`工具调用（Tool Call） -> 权限（Permission） -> execution -> 原始/领域工具结果（raw/domain Tool Result） -> 工具结算（Tool Settlement） -> durable terminal state / 模型工具输出（Model Tool Output）`。终态是调用状态，Model Tool Output 是持久化到 Session History、回放给模型的有界结果投影，两者不能混称“Tool Result”。
- **压缩与持久化链**：已完成检查点（completed Checkpoint）改变 active-history boundary；durable domain facts 可重读，但会话执行排空（Session Drain）、coordinator 和 Tool fibers 仍是 process-local。

```text
Location
└─ System Context Registry
   └─ ordered Context Sources
      ├─ initialize ───────────────> Baseline System Context
      └─ compare with ─────────────> Context Snapshot
                                      │
                                      └─ all belong to one Context Epoch
                                         └─ changed source
                                            -> Mid-Conversation System Message
                                            -> enters Session History chronologically

Prompt Admission -> Admitted Prompt (durable inbox, not model-visible)
                         │
                         └─ Prompt Promotion at a Safe Provider-Turn Boundary
                            -> User Message enters Session History

Session Drain (process-local coordination span)
└─ Provider Turn N (exactly one physical provider request + response projection)
   ├─ selected Session History + Baseline System Context + Tool definitions
   ├─ Tool Call -> Permission -> execution -> raw/domain Tool Result
   │                                      -> Tool Settlement
   │                                         ├─ durable terminal state
   │                                         └─ Model Tool Output -> Session History
   ├─ Retry, if any -> another Provider Turn
   └─ settled Model Tool Output may require another Provider Turn

Compaction -> completed Checkpoint -> new active-history boundary
                                  └─ next Provider Turn starts a fresh Context Epoch baseline

Session Persistence
├─ durable: admitted input, messages, Tool terminal states/Model Tool Output, checkpoint, epoch state
├─ process-local: drain, coordinator, tool fibers
└─ live-only: streamed deltas that are not replayable
```

这张图主要表达 native V2 的正式关系。当前默认旧运行时有 Provider Turn、Tool lifecycle、Compaction 和持久化等相邻机制，但没有 typed `Context Source`、durable `Context Snapshot`、`Context Epoch` 或独立的 admission/promotion inbox；不能把两条路径拼成一套已上线流程。

## 2. 最短阅读路径

只想避免最常见误解，按下面顺序读：

1. [System Context](#31-系统上下文system-context) 与 [Session History](#32-会话历史session-history)：先分清“系统级事实”和“时间序对话”。
2. [Context Source](#33-上下文源context-source)、[Context Snapshot](#35-上下文快照context-snapshot)、[Context Epoch](#36-上下文纪元context-epoch)：理解 V2 如何观察、比较和固定 baseline。
3. [Prompt Admission](#311-提示词接纳prompt-admission)、[Prompt Promotion](#313-提示词提升prompt-promotion)、[Provider Turn](#39-提供商轮次provider-turn)、[Session Drain](#310-会话执行排空session-drain)：理解输入何时 durable、何时模型可见、一次执行为何可含多轮。
4. [Tool Call](#314-工具调用tool-call)、[Tool Result](#315-工具结果tool-result)、[Model Tool Output](#316-模型工具输出model-tool-output)、[Tool Settlement](#317-工具结果结算tool-settlement)、[Permission](#318-权限permission)：理解调用意图、原始/领域结果、持久化终态与模型可见投影的边界。
5. [Compaction](#319-上下文压缩compaction)、[Checkpoint](#320-检查点checkpoint)、[Session Persistence](#321-会话持久化session-persistence)：理解模型可见历史、持久化历史和崩溃恢复不是同一件事。

## 3. 术语词典

### 3.1 系统上下文（System Context）

**官方英文**：`System Context`

**建议中文**：系统上下文

**官方定义翻译**：呈现给模型、作为初始指令及按时间顺序更新的一组结构化上下文事实。

**通俗解释**：它回答“模型现在所处的环境和应遵循的系统级事实是什么”。在 native V2 中，它由多个 typed Context Sources 组成；首份完整内容成为 baseline，之后的变化按时间顺序补入。

**不要混淆**：不等于任意一段 `system` 字符串，不等于完整 Provider Request，也不包含 Tool schemas。当前旧路径每轮拼接的 Environment、Instructions、MCP、Skills 等 system-level 文本，不能倒推成 V2 `SystemContext` 对象。

**当前实现 / V2 状态**：当前默认旧运行时每轮重新观察并拼接 system-level 字符串，没有 typed source、snapshot 或 epoch。native V2 的 System Context algebra、Registry、baseline/update admission 已实现，但旧路径的 provider base、配置/远程/嵌套 instructions、MCP、per-prompt system 和 plugin transforms 等 parity 仍为 partial/missing。

**官方 Avoid**：`System prompt`。

**源码 / 设计引用**：`CONTEXT.md`，`Language` 与 `Relationships`，7-9、88-99；`packages/core/src/system-context/index.ts`，模块说明、`SystemContext`、`make/combine/initialize/reconcile`，5-16、31-80、131-225；`specs/v2/session.md`，`Context Epochs` 与 `V1 Runtime Context Parity`，54-109、123-151。

### 3.2 会话历史（Session History）

**官方英文**：`Session History`

**建议中文**：会话历史

**官方定义翻译**：应用当前有效的 Compaction 与 Context Epoch 截止点后，为某次 Provider Turn 选出的、按时间顺序投影的对话。

**通俗解释**：它不是数据库里的所有记录，而是“这一轮实际挑出来给模型回看的对话版本”。其中可以有 User、Assistant、Tool 记录和已接纳的 Mid-Conversation System Message。

**不要混淆**：不等于 `Session Context`，不等于 System Context，不等于完整数据库历史，也不等于 `sessions.context(...)` 暴露了完整 Provider Request。Tool schemas 也不属于 Session History。

**当前实现 / V2 状态**：当前默认旧运行时通过 `filterCompactedEffect` 选择 Message/Part projection，并在每轮重新加载；它没有 Context Epoch cutoff。native V2 按 compaction sequence 与 epoch `baselineSeq` 选择 `session_message`，该机制已实现。

**官方 Avoid**：`Session Context`。

**源码 / 设计引用**：`CONTEXT.md`，11-13、91、99-107、178-179；`packages/opencode/src/session/prompt.ts`，`SessionPrompt.run`，1081-1096、1288-1338；`packages/core/src/session/history.ts`，`messageRows/load/entriesForRunner`，13-99。

### 3.3 上下文源（Context Source）

**官方英文**：`Context Source`

**建议中文**：上下文源

**官方定义翻译**：System Context 中一个独立观察得到的 typed value；它由稳定 key、JSON codec、不会以普通错误失败的 loader、纯 baseline/update renderer，以及供动态 source 使用的可选 removal renderer 表示。

**通俗解释**：把“日期”“环境信息”“项目指令”“当前 Agent 可用 Skills”看成独立、可比较、可单独刷新的事实生产者，而不是先拼成一大段不可追踪字符串。

**不要混淆**：不等于任意 Prompt Fragment，不等于 Tool Result，也不能把旧路径中的每个字符串数组元素自动称为 Context Source。source 暂时 unavailable 与 source 已成功确认被移除也不同。

**当前实现 / V2 状态**：当前默认旧运行时无此 typed abstraction。native V2 已实现 stable namespaced key、codec、load、baseline/update/removal、组合和 unavailable 语义；已接入的 source 仅覆盖旧 runtime context 的一部分，plugin-defined source 与更多 instruction source 仍是后续项。

**官方 Avoid**：`Prompt fragment`。

**源码 / 设计引用**：`CONTEXT.md`，15-17、90-98、108-122、126；`packages/core/src/system-context/index.ts`，`Key/Source/make/combine/reconcile`，21-39、131-179、217-290；`specs/v2/session.md`，56-58、99-109、129-135。

### 3.4 系统上下文注册表（System Context Registry）

**官方英文**：`System Context Registry`

**建议中文**：系统上下文注册表

**官方定义翻译**：按 Location 划定作用域、由有序且有作用域的 producer 构成的注册表；这些 producer 共同贡献当前 System Context。

**通俗解释**：它是 Context Source 的组装入口，负责“哪些来源当前有效、按什么稳定顺序组合”，不是保存聊天记录的数据库。

**不要混淆**：不等于 Tool Registry，不等于 Context Snapshot，也不等于全进程唯一的无作用域全局表。Registry contribution key 与 source key 也是两层身份。

**当前实现 / V2 状态**：当前默认旧运行时使用独立的 system/instruction/MCP/skill 服务，没有此 registry。native V2 的 Location-scoped registration、并发加载、按 contribution key 稳定排序与 scoped removal 已实现；plugin hot reload 语义仍待设计。

**官方 Avoid**：未列出。

**源码 / 设计引用**：`CONTEXT.md`，19-21、92、109、121；`packages/core/src/system-context/registry.ts`，`register/load`，12-49；`packages/core/src/session/runner/llm.ts`，`loadSystemContext`，168-171。

### 3.5 上下文快照（Context Snapshot）

**官方英文**：`Context Snapshot`

**建议中文**：上下文快照

**官方定义翻译**：可覆盖、对模型隐藏的 JSON 状态，用于将每个 Context Source 与其最近一次被接纳进入 Provider Turn 的值进行比较。

**通俗解释**：它像一张内部“上次已经告诉模型什么”的比较表。下一轮观察到 source 变化时，runtime 用它决定是否需要产生 chronological update；模型看不到这张表本身。

**不要混淆**：不等于 Session History 的副本，不等于模型 KV cache，不等于代码工作树 Snapshot。Context Snapshot 不能 restore 文件。

**当前实现 / V2 状态**：当前默认旧运行时没有 Context Snapshot。native V2 将每个 source 的 encoded value 和可选 removal text 持久化在 active epoch row 中；普通更新与相应 Mid-Conversation System Message 在同一 durable commit 中推进 snapshot。

**官方 Avoid**：未列出。

**本系列写作约束**：不得与代码工作树 Snapshot 混用；涉及文件 diff/restore 时必须写全称“代码工作树 Snapshot”。

**源码 / 设计引用**：`CONTEXT.md`，33-35、94-96、105、111-115；`packages/core/src/system-context/index.ts`，`SourceSnapshot/Snapshot`，48-62；`packages/core/src/session/context-epoch.ts`，snapshot decode、update 与 `advance`，46-77、161-174。

### 3.6 上下文纪元（Context Epoch）

**官方英文**：`Context Epoch`

**建议中文**：上下文纪元

**官方定义翻译**：一份最初渲染的 System Context 持续作为不可变 provider-cache baseline 的时间跨度；该跨度在 Compaction 完成、Session 移动，或需要新 baseline 的不兼容 context transition 时结束。

**通俗解释**：只要 baseline 可继续安全复用，就仍处在同一 epoch；普通 source 变化写成时间序更新，而不是改写 baseline。发生会改变前缀基础的边界后，再开一个新 epoch。

**不要混淆**：不等于整个 Session 生命周期，不等于每个 Provider Turn，也不表示数据库中一定保留多个显式历史 epoch rows。

**当前实现 / V2 状态**：当前默认旧运行时没有 Context Epoch。native V2 的初始化、reconciliation 与 completed-compaction replacement 已接入 Runner；但固定 SHA 只有每 Session 一行可覆盖状态，没有独立 activation/retirement event。`reset` 函数存在，但未找到 Session move 调用接线，因此 move reset 仍为 missing/planned。

**官方 Avoid**：未列出。

**源码 / 设计引用**：`CONTEXT.md`，26-31、107、118、130-134；`packages/core/src/session/context-epoch.ts`，`initialize/prepare/reset/replace`，23-89、111-159；`specs/v2/session.md`，54-101。固定 SHA 下规格 82、96 行描述 move clear，但实现接线审计不支持把它写成已完成。

### 3.7 基线系统上下文（Baseline System Context）

**官方英文**：`Baseline System Context`

**建议中文**：基线系统上下文

**官方定义翻译**：在一个 Context Epoch 开始时渲染出的完整 System Context。

**通俗解释**：它是本 epoch 的固定“开场系统前缀”。后续日期、指令或 Skill guidance 的变化不会原地修改它，而是通过 chronological update 告知模型。

**不要混淆**：不等于每轮实时重算的 `Live system prompt`，不等于 Agent 自己的 system 文本，也不等于完整 Provider Request。V2 当前请求顺序是 Agent system 在前、durable baseline 在后。

**当前实现 / V2 状态**：当前默认旧运行时每轮重算 system-level 内容，不保存这种 epoch baseline。native V2 已将 exact joined baseline text 与 snapshot durable 保存并在同一 epoch 内原样复用；completed Compaction 后会替换为当前完整 context 的新 baseline。

**官方 Avoid**：`Live system prompt`。

**源码 / 设计引用**：`CONTEXT.md`，29-31、105-107、130-133；`packages/core/src/system-context/index.ts`，`Generation/initialize`，59-62、197-214；`packages/core/src/session/runner/llm.ts`，request `system`，197-214。

### 3.8 会话中途系统消息（Mid-Conversation System Message）

**官方英文**：`Mid-Conversation System Message`

**建议中文**：会话中途系统消息

**官方定义翻译**：一条 durable、按时间顺序排列的指令，用来告知模型某个已变化 Context Source 的新生效状态。

**通俗解释**：baseline 建立后，若日期或项目指令变化，不重写开头，而是在对话时间线上追加一条“从现在起以这个新状态为准”的系统级消息。

**不要混淆**：不等于异步推送的 notification，不等于 raw text diff，也不等于重建 baseline。多个 source 在同一安全边界的变化会合成一条消息。

**当前实现 / V2 状态**：当前默认旧运行时没有这种 durable chronological context update；system-level 内容通常每轮重算。native V2 的 `ContextUpdated` event、`session_message` projection、snapshot atomic advance 和 provider lowering 已实现；普通用户界面可以隐藏这类消息。

**官方 Avoid**：`System update`、`system notification`、`raw text diff`。

**源码 / 设计引用**：`CONTEXT.md`，22-24、91-99、127-128、186-187；`packages/core/src/session/context-epoch.ts`，`ContextUpdated` publication，56-77；`packages/core/src/session/history.ts`，system cutoff selection，24-49。

### 3.9 提供商轮次（Provider Turn）

**官方英文**：`Provider Turn`

**建议中文**：提供商轮次

**官方定义翻译**：向模型 Provider 发出的一次请求，以及从该请求投影得到的响应。

**通俗解释**：每实际请求一次 Provider，就开始一个新的 Provider Turn。用户的一句话可能因 Tool continuation 或 Retry 触发多次物理请求，因此可包含多个 Provider Turns。

**不要混淆**：不等于一次完整用户对话，不等于一个 Session Drain，也不以 Assistant Message、Processor context 或“逻辑尝试”划界。Retry 的每次物理请求及其响应投影各是一个 Provider Turn；不得因复用上述应用状态而合并这些 Turns。

**当前实现 / V2 状态**：当前默认旧运行时中，每次初始或 Retry 的 `llm.stream(streamInput)` 物理请求都分别构成一个 Provider Turn；Retry 可以复用同一 Assistant Message、Processor context 和 `streamInput`，但这不合并 Turn。Tool continuation 回到外层 loop 后再发请求。native V2 在 `runTurnAttempt` 中每轮显式调用一次 `llm.stream(request)`，并在 continuation 前等待 Tool settlement、重载 history；一般 Provider Retry parity 仍缺失。

**官方 Avoid**：未列出。

**源码 / 设计引用**：`CONTEXT.md`，48-52、123-125；`packages/opencode/src/session/prompt.ts`，`SessionPrompt.run`，1081-1338；`packages/opencode/src/session/processor.ts`，`process` 中的 Retry 包围，627-676；`packages/opencode/src/session/retry.ts`，`policy`，84-205；`packages/core/src/session/runner/llm.ts`，`runTurnAttempt`，173-345；`specs/v2/session.md`，48-52、153-165。

### 3.10 会话执行排空（Session Drain）

**官方英文**：`Session Drain`

**建议中文**：会话执行排空

**官方定义翻译**：一个进程内执行区间；它提升符合条件的输入，并运行所需 Provider Turns，直到没有可立即继续的工作。Session Drain 没有 durable identity，也不是 transcript boundary。

**通俗解释**：一次唤醒后，runtime 把“现在能做的工作”连续处理到空闲。这个运行过程本身只存在于当前进程；真正可恢复的是输入、消息、Provider/Tool 状态等 durable facts。

**不要混淆**：不等于持久化 Job、Message、Provider Turn 或 Session 生命周期。durable history 不意味着 drain 会在崩溃后自动续跑。

**当前实现 / V2 状态**：当前默认旧运行时有 process-local `SessionRunState` 与 loop，可作相邻机制理解，但没有 V2 durable inbox/drain 语义。native V2 的 coordinator、join/coalesced wake/interrupt 与 drain 已实现为 process-local；clustered ownership 和自动 post-crash continuation 仍 missing/planned。

**官方 Avoid**：未列出。

**源码 / 设计引用**：`CONTEXT.md`，51-53、102-105；`packages/core/src/session/run-coordinator.ts`，`Coordinator/make`，5-25、51-103；`packages/core/src/session/runner/llm.ts`，`SessionRunner.run`，383-405；`specs/v2/session.md`，155-169。

### 3.11 提示词接纳（Prompt Admission）

**英文术语**：`Prompt Admission`

**建议中文**：提示词接纳

**本文工作定义（依据源码与设计归纳）**：`CONTEXT.md` 没有为 `Prompt Admission` 单列正式定义；它正式定义了结果状态 `Admitted Prompt`。本文把 admission 限定为：将用户输入接受进 Session 的 durable inbox，发布 `PromptAdmitted` 并形成 `session_input`，但尚不把它追加到 model-visible Session History 的转换。

**通俗解释**：系统先给输入一张“已收件、可重试”的收据，稍后在安全边界才让模型看到它。

**不要混淆**：不等于 Prompt Promotion，不等于 Provider 已开始执行，也不等于 Prompt POST 已返回最终 Assistant 内容。`wake` 只是 advisory execution request。

**当前实现 / V2 状态**：当前默认旧运行时没有独立 inbox admission，通常创建 User Message/Parts 后直接进入旧 loop，且 Message 与所有 Parts 不是一个原子写入。native V2 的 durable `SessionInput.admit`、exact retry 与 `resume:false` admit-only 已实现。

**官方 Avoid**：未列出。

**本系列写作约束**：不要用“已执行”替代“已接纳”。

**源码 / 设计引用**：`CONTEXT.md`，42-46、100-106、180-181；`packages/core/src/session/input.ts`，`admit/projectAdmitted`，41-116；`packages/core/src/session.ts`，`V2Session.prompt`，360-386；`specs/v2/session.md`，155-165。

### 3.12 已接纳提示词（Admitted Prompt）

**官方英文**：`Admitted Prompt`

**建议中文**：已接纳提示词

**官方定义翻译**：已经 durable 接受到 Session inbox 中、但尚未包含在 Session History 里的用户输入。

**通俗解释**：输入已经不会因本次请求结束就消失，但模型还没看见；它正在等待 promotion。

**不要混淆**：不等于 User Message，不等于 pending Tool Call，也不等于模型上下文中的 prompt。它是可重放的 pending input。

**当前实现 / V2 状态**：当前默认旧运行时没有此独立状态。native V2 以 `session_input` row、`admitted_seq` 和可选 `promoted_seq` 表示；初始 System Context unavailable 时，该输入可保持 pending。

**官方 Avoid**：未列出。

**源码 / 设计引用**：`CONTEXT.md`，42-46、100、105-106；`packages/core/src/session/input.ts`，`fromRow/admit/hasPending`，21-35、41-81、170-189；`specs/v2/session.md`，56-80。

### 3.13 提示词提升（Prompt Promotion）

**官方英文**：`Prompt Promotion`

**建议中文**：提示词提升

**官方定义翻译**：一个 durable transition：从 pending input 中移除一个 Admitted Prompt，并把它的 User Message 追加到 Session History。

**通俗解释**：它是“收件箱内容正式进入模型可见时间线”的时刻。

**不要混淆**：不等于 admission，不等于 wake，不等于立即完成一个 Provider Turn。`steer` 与 `queue` 的 promotion 时机不同。

**当前实现 / V2 状态**：当前默认旧运行时没有 admission/promotion 分离；活动运行中新输入通过旧 Message 持久化和下一轮重载进入模型。native V2 在 `Prompted` durable event 的同一 transaction 中更新 `promoted_seq` 并 append User `session_message`；steer/queue 规则已实现。

**官方 Avoid**：未列出。

**源码 / 设计引用**：`CONTEXT.md`，45-46、99-103；`packages/core/src/session/input.ts`，`projectPrompted/promoteSteers/promoteNextQueued`，118-168、216-288；`specs/v2/session.md`，58-80、155-171。

### 3.14 工具调用（Tool Call）

**英文术语**：`Tool Call`

**建议中文**：工具调用

**本文工作定义（依据源码与设计归纳）**：`CONTEXT.md` 没有为 Tool Call 单列正式定义。本文把它限定为 Provider 在一个 Provider Turn 中产生的调用意图，至少关联 call ID、工具名称与输入；对本地工具而言，OpenCode 随后还要验证、授权和执行。

**通俗解释**：模型说“请用 `read` 读取这个路径”只是 Tool Call；文件在这一步还没有因为模型的文字而自动被打开。

**不要混淆**：不等于 Tool 已执行，不等于 Tool Result，也不等于 Tool Settlement。`providerExecuted: true` 的 hosted tool 则由 Provider 执行，OpenCode 不应再运行同名本地 executor。

**当前实现 / V2 状态**：当前默认旧运行时通过 AI SDK/native adapter 将 Provider call 归一化并建立 Tool Part 状态。native V2 durable 发布 `Tool.Called` 后才启动本地 settlement fiber；call 在副作用前具有 durable assistant/call identity。

**官方 Avoid**：未列出。

**源码 / 设计引用**：`packages/schema/src/session-event.ts`，`Tool.Called`，273-325；`packages/core/src/session/runner/llm.ts`，Tool Call dispatch，228-271；`specs/v2/session.md`，48-52、187-215。专题说明见 [10 Tools 与 Security](./10_Tools_and_Security.md)。

### 3.15 工具结果（Tool Result）

**英文术语**：`Tool Result`

**建议中文**：工具结果

**本文工作定义（依据源码与设计归纳）**：`CONTEXT.md` 没有为通用 Tool Result 单列正式定义。本文用 raw/domain Tool Result 指 executor 或 Provider-hosted execution 产生的原始值或领域值，包括成功值或错误值；它位于 execution 之后、Tool Settlement 之前，不保证已完成 codec、projection、bounding 或持久化。

**通俗解释**：这是工具刚执行出来的“内部结果”，可能是原始文本、结构化值或错误；模型最终看到的是后续结算形成的 Model Tool Output，不应直接把两者画等号。

**不要混淆**：不等于 Tool Call、Tool Settlement、durable terminal state 或 Model Tool Output，也不等于完整外部副作用或 Tool schema。executor 已返回 Tool Result 仍不表示调用已经 durable 封口。

**当前实现 / V2 状态**：当前默认旧运行时的 Tool wrapper、adapter 与 Processor 会把 executor 结果继续转换为 completed/error Tool Part，结果 shaping 与 bounding 分散。native V2 先通过 typed output codec 和 model projection 处理领域结果，再集中 bounding 并发布 durable `Tool.Success/Failed`；producer capture 与 hosted structured payload 仍有边界。

**官方 Avoid**：未列出。

**源码 / 设计引用**：`CONTEXT.md`，54-58、189-199（用于与正式的 Model Tool Output 区分）；`packages/schema/src/session-event.ts`，`Tool.Success/Failed`，342-372；`packages/core/src/tool/registry.ts`，`settleWith`，50-81；`specs/v2/session.md`，187-215。

### 3.16 模型工具输出（Model Tool Output）

**官方英文**：`Model Tool Output`

**建议中文**：模型工具输出

**官方定义翻译**：Core 执行的工具结果的有界投影；它持久化在 Session History 中并回放给模型。工具可以按语义塑造该投影，但 Tool Registry 强制实施最终大小限制。

**通俗解释**：这是模型下一轮真正能看到的“结果版本”。原始输出过大时，Session History 只保存有界预览；完整文本可以放入临时 managed output file，但那个文件不是模型输出本身。

**不要混淆**：不等于 raw/domain Tool Result，不等于 `Tool.Success/Failed` durable terminal state，也不等于 Tool Settlement 过程或 Managed Tool Output File。Provider-executed tool result 是 provider-native transcript fact，不属于 Core Tool Registry 的通用 bounding 合同。

**当前实现 / V2 状态**：当前默认旧运行时把 completed/error Tool Part durable 保存并在下一轮 history lowering 后回放，但通用 bounding 分散，且普通 Registry 的 after-hook 可再次放大已截断输出。native V2 已实现 model projection、集中 bounding、managed output path 与 durable replay；任意 structured-result 大小和 provider-hosted payload 仍有单独边界。

**官方 Avoid**：未列出。

**源码 / 设计引用**：`CONTEXT.md`，54-58、189-199；`packages/core/src/tool/registry.ts`，`settleWith`，50-81；`packages/core/src/tool-output-store.ts`，`bound`，112-174；`packages/schema/src/session-event.ts`，`Tool.Success/Failed`，342-372；`specs/v2/tools.md`，153-170。

### 3.17 工具结果结算（Tool Settlement）

**英文术语**：`Tool Settlement`

**建议中文**：工具结果结算

**本文工作定义（依据源码与设计归纳）**：`CONTEXT.md` 没有为 Tool Settlement 单列正式定义。本文把 settlement 限定为：Harness 接收 raw/domain Tool Result，执行必要的 codec、model-output projection 与 bounding，并把 Tool Call 发布为成功、失败或中断 durable terminal state 的过程；Core 执行结果的 Model Tool Output 随该终态持久化并供历史回放。

**通俗解释**：executor 返回只是中间点；直到结果被校验、裁界并写成 terminal fact，调用才真正“封口”。封口状态与回放给模型的内容同时产生，但不是同一个概念。

**不要混淆**：不等于只调用一次 `execute()`，不等于 raw/domain Tool Result，不等于 durable terminal state，也不等于 Model Tool Output。它不保证外部副作用与数据库记录构成一个原子事务；中断后写入 failed 只是在记录层封口，不会回滚已发生的副作用。

**当前实现 / V2 状态**：当前默认旧运行时由 Tool wrapper、LLM adapter 与 `SessionProcessor` 共同把 Tool Part 推到 completed/error。native V2 的 materialization-specific settlement、stale identity、codec、bounding、Tool fibers 与 durable Success/Failed 已实现；进程崩溃后的 ambiguous side effect 不会自动重放。

**官方 Avoid**：未列出。

**源码 / 设计引用**：`CONTEXT.md`，99、125、195；`packages/core/src/tool/registry.ts`，`settleWith/materialize`，50-121；`packages/core/src/session/runner/llm.ts`，settlement 与等待 Tool fibers，243-345；`specs/v2/session.md`，48-52、165、187-215。

### 3.18 权限（Permission）

**英文术语**：`Permission`

**建议中文**：权限；需要强调边界时写“OpenCode 应用层权限策略”

**本文工作定义（依据源码与设计归纳）**：`CONTEXT.md` 没有为 Permission 单列正式定义。本文按当前实现将其限定为：对 action/resource（旧合同为 permission/pattern）应用有序规则，得到 `allow`、`ask` 或 `deny`；`ask` 会建立 pending request 并等待用户回复。

**通俗解释**：它是 OpenCode 在执行受管动作前设置的策略门，决定直接放行、询问用户还是拒绝。

**不要混淆**：不等于 OS Sandbox，不会减少 OpenCode 进程在操作系统中的真实权限。Tool definition 被隐藏是 catalog visibility；调用到达 executor 后是否获准执行是另一道边界。Plugin/Custom Tool 也不会因注册就自动受统一 leaf authorization 包围。

**当前实现 / V2 状态**：当前默认旧 Permission 使用最后匹配规则，pending 与 `always` approval 都是 Instance 内存状态。native V2 使用 action/resource/effect；configured deny 优先于 durable saved project approval，pending Deferred 仍是 process-local。两者都不是 OS sandbox。

**官方 Avoid**：未列出。

**本系列写作约束**：避免用 `Sandbox` 代称 Permission。

**源码 / 设计引用**：`packages/opencode/src/permission/index.ts`，`evaluate/ask/reply`，23-38、67-167；`packages/core/src/permission.ts`，`evaluate/assert/reply`，76-101、131-162、190-285；`specs/v2/session.md`，204-206。专题说明见 [10 Tools 与 Security](./10_Tools_and_Security.md)。

### 3.19 上下文压缩（Compaction）

**英文术语**：`Compaction`

**建议中文**：上下文压缩

**本文工作定义（依据 `CONTEXT.md` 关系约束、源码与设计归纳）**：`CONTEXT.md` 没有在 `Language` 中为 Compaction 单列正式定义；其关系约束规定，completed Compaction 开启新的 Context Epoch，以新 Baseline System Context 取代旧 baseline，并让早期 Mid-Conversation System Messages 离开 active model history。V2 设计进一步规定：保留完整 durable transcript，同时用 completed checkpoint 替换 active model representation。

**通俗解释**：当请求快装不下时，把较早对话压成摘要和受限近期内容，让后续模型在有限窗口内继续；旧记录是否仍在存储是另一个问题。

**不要混淆**：不等于删除数据库历史，不等于 Pruning，不等于代码工作树 Snapshot，也不等于 Revert。压缩摘要是有损的，不能保证保留所有早期约束。

**当前实现 / V2 状态**：当前默认旧运行时使用 synthetic compaction marker、Assistant summary 与 retained tail；`auto=false` 时本地阈值不触发，Provider overflow 会保存错误并停止。native V2 使用 request-budget/overflow-triggered compaction 与 completed checkpoint；manual public compact 仍 unavailable，deterministic old Tool-result pruning 未实现。V2 `auto=false + Provider overflow` 固定代码显示仍可能尝试 checkpoint，但本轮无专门运行验证。

**官方 Avoid**：未列出。

**源码 / 设计引用**：`CONTEXT.md`，107、130-135；`packages/opencode/src/session/compaction.ts`，`select/processCompaction/create/prune`，203-582；`packages/core/src/session/compaction.ts`，`compactAfterOverflow/compactIfNeeded`，176-247；`specs/v2/session.md`，111-121、145。

### 3.20 检查点（Checkpoint）

**英文术语**：`Checkpoint`

**建议中文**：检查点；在 V2 Compaction 语境中可写“压缩检查点”

**本文工作定义（依据源码与设计归纳）**：`CONTEXT.md` 没有为 Checkpoint 单列正式定义。本文按 native V2 设计将其限定为：由 completed Compaction 投影出的、对普通 transcript surface 隐藏但 model-visible 的 compaction message，其中保存结构化 rolling summary 与 token-bounded serialized recent context，并成为新的 active history boundary。

**通俗解释**：它是压缩成功后的“继续工作包”。未来模型主要从摘要和近期片段继续，而不是重放 checkpoint 前的全部 provider-native turns。

**不要混淆**：不等于代码工作树 checkpoint，不等于每个 durable event，也不等于 `Compaction.Started`。只有有有效非空 summary 的 `Compaction.Ended` 才激活新 boundary；失败或中断 attempt 不激活。

**当前实现 / V2 状态**：当前默认旧运行时有 compaction marker、summary 与 retained tail，但没有 native V2 这种 `session.next.compaction.ended` checkpoint contract。native V2 的 Started/Ended、message projection、history cutoff 和下一次 epoch rebaseline 已实现；Compaction Delta schema 是 live-only，但固定 SHA 的 compactor 只在本地累积 chunks，未发布 progress delta，因此实时进度为 partial。

**官方 Avoid**：未列出。

**源码 / 设计引用**：`packages/schema/src/session-event.ts`，`Compaction.Started/Delta/Ended` 与 durable inventory，398-431、448-512；`packages/core/src/session/compaction.ts`，176-229；`packages/core/src/session/history.ts`，13-49、90-99；`specs/v2/session.md`，111-121。

### 3.21 会话持久化（Session Persistence）

**英文术语**：`Session Persistence`

**建议中文**：会话持久化

**本文工作定义（依据源码与设计归纳）**：`CONTEXT.md` 没有为 Session Persistence 单列正式定义。本文将其限定为：通过 durable events、SQLite projections、Session input/message state、Context Epoch state、Tool terminal states、Model Tool Output 与 compaction checkpoint，使 Session domain facts 能跨调用或进程重读的机制集合。

**通俗解释**：它回答“程序退出后还能从哪里读回已经确认保存的事实”，而不是“模型能否永远记住所有事情”。

**不要混淆**：不等于长期记忆（Long-term Memory），不等于 Context window，不等于 live event transport，也不等于自动 post-crash continuation。数据库可重读并不能消除 Provider dispatch 或 Tool side effect 的崩溃歧义。

**当前实现 / V2 状态**：当前默认旧运行时已有 durable Event、Message/Part projection、Compaction 和代码工作树 Snapshot/Revert 引用，但 Runner 与增量 accumulator 是 process-local。native V2 已有 durable inbox/promotion、message/event projection、epoch、Tool terminal state、Model Tool Output、checkpoint 和 per-session cursor；自动 post-crash continuation、clustered execution ownership 仍 missing/planned。

**官方 Avoid**：未列出。

**本系列写作约束**：不能仅因 Session facts 可持久化，就称为长期记忆（Long-term Memory）。

**源码 / 设计引用**：`CONTEXT.md`，54-58、104-107、127-135、165-170、189-199；`packages/core/src/event.ts`，`commitDurableEvent/publishEvent`，205-438；`packages/core/src/session/input.ts`，41-168；`packages/core/src/session/context-epoch.ts`，23-174；`specs/v2/session.md`，160-185。

## 4. 补充边界术语

### 4.1 安全的提供商轮次边界（Safe Provider-Turn Boundary）

**官方英文**：`Safe Provider-Turn Boundary`

**建议中文**：安全的提供商轮次边界

**官方定义翻译**：紧邻 Provider call 之前的时点；此时 durable input promotion 和必要 Tool settlement 已完成，context changes 可以按时间顺序接纳。

**通俗解释**：runtime 不在 source 一变化时就打断模型，而是在下一次即将请求 Provider、且前置事实已稳定的关口统一处理变化。

**不要混淆**：不等于异步 watcher callback，不等于 Provider response 结束瞬间，也不保证一定有下一轮；idle Session 不会仅因 context source 变化而被唤醒。

**当前实现 / V2 状态**：当前默认旧运行时没有该正式 admission boundary。native V2 在 promotion、context preparation、history load 与 Provider request 之间实现相应顺序；source changes 只在自然调度到的边界懒观察。

**官方 Avoid**：未列出。

**源码 / 设计引用**：`CONTEXT.md`，39-46、98-106、123-127；`packages/core/src/session/runner/llm.ts`，183-215；`specs/v2/session.md`，56-82。

### 4.2 不可用上下文（Unavailable Context）

**官方英文**：`Unavailable Context`

**建议中文**：不可用上下文；强调暂态时可写“暂时不可观测的上下文”

**官方定义翻译**：对某个 Context Source 值的预期内、暂时性观察失败；runtime 保留此前的有效状态且不发更新，或在首次成功加载前省略它。

**通俗解释**：这表示“现在读不到，先不要宣称它被删除”。初始 baseline 要求完整时会阻止 turn；已有 snapshot 时则使用 stale-while-revalidate。

**不要混淆**：不等于 source 已确认不存在，不等于 loader defect，也不等于 removal。成功观察到 absence 才可能调用 removal renderer。

**当前实现 / V2 状态**：当前默认旧路径中的某些 instruction 读取失败会降为空文本，不具有相同 stale-while-revalidate contract。native V2 的 sentinel、初始化阻塞、reconcile 保留旧值和 replacement blocking 已实现。

**官方 Avoid**：未列出。

**源码 / 设计引用**：`CONTEXT.md`，36-37、105-115；`packages/core/src/system-context/index.ts`，`unavailable/InitializationBlocked/reconcile/replace`，27-29、82-89、217-290；`specs/v2/session.md`，56-58、99。

### 4.3 位置（Location）

**英文术语**：`Location`

**建议中文**：位置；需要避免地理含义时写“运行位置”

**本文工作定义（依据源码与设计归纳）**：`CONTEXT.md` 使用 Location，但没有在 `Language` 中单列正式定义。本文按源码将其限定为运行服务的 placement identity，至少包含 directory、可选 workspace identity、resolved project，并可带 VCS 信息；System Context Registry、Session Runner、Tool Registry、Permission 和 filesystem 等服务按它划定作用域。

**通俗解释**：它回答“这个 Session 在哪个项目目录和哪组运行服务中执行”，不是经纬度。

**不要混淆**：不等于仅一个 cwd 字符串，不等于 Session ID，也不等于 OS sandbox。Session move 是 placement change，并按设计要求重建 context baseline。

**当前实现 / V2 状态**：当前默认旧运行时使用 Instance directory/worktree 等相邻作用域。native V2 已实现 Location service 与按 Location 获取 Runner/Registry/Permission/filesystem；但 fixed SHA 的 Context Epoch move reset 接线缺失，不能宣称移动后 epoch 已自动清理。

**官方 Avoid**：未列出。

**源码 / 设计引用**：`CONTEXT.md`，20、117-125、182、185；`packages/core/src/location.ts`，`Interface/Service/layer`，9-39；`specs/v2/session.md`，40-48、82、96。

### 4.4 Durable、process-local 与 live-only

**英文术语**：`durable`、`process-local`、`live-only`

**建议中文**：可持久化、进程内、仅实时

**本文工作定义（依据源码与设计归纳）**：这三个词没有作为 `CONTEXT.md Language` 的独立正式条目。本文按 Event/Session contract 将其限定为：durable facts 提交到持久化事件或 projection、可在相应保留和读取合同下重放；process-local 状态只协调当前进程；live-only event 只通知当前订阅者，不写 durable sequence/event row，也不能断线重放。

**通俗解释**：durable 是“以后能从权威存储读回”；process-local 是“进程还活着时有用”；live-only 是“在线时能看到，错过就没有”。

**不要混淆**：durable 不等于永久保留或自动续跑；live transport 可以传 durable event，但 transport 本身仍不 durable；UI 已显示 delta 也不代表 delta 已落库。

**当前实现 / V2 状态**：当前默认旧运行时的 whole Message/Part durable、`message.part.delta` live-only、Runner/accumulator process-local。native V2 的 Started/Ended/Tool settlement/checkpoint durable，Text/Reasoning/Tool Input Delta live-only，Session Drain/Tool fibers process-local。

**官方 Avoid**：未列出。

**源码 / 设计引用**：`CONTEXT.md`，104-107、126-133、165-170；`packages/schema/src/session-event.ts`，Delta 与 durable inventory，291-310、398-431、448-520；`packages/core/src/session/run-coordinator.ts`，24-103；`specs/v2/session.md`，165-183。

## 5. 高频误用速查

| 不准确说法 | 建议改写 |
| --- | --- |
| “System Prompt 里包含全部 Context” | “本轮 Provider Request 分为 system/initial instructions、selected Session History 与 Tool definitions；native V2 的 System Context 只是其中一个正式边界。” |
| “Session Context 被保存了” | 指明是 `Session History`、Context Snapshot、Context Epoch state、Message/Part，还是其他 durable projection。 |
| “Prompt 已提交，所以模型已经看到” | native V2 中先是 Prompt Admission；只有 Prompt Promotion 后才进入 Session History。 |
| “模型执行了 read” | “模型产生 Tool Call；OpenCode 经 Permission 后执行 read，得到 raw/domain Tool Result，再完成 Tool Settlement，写入 durable terminal state 与 Model Tool Output。” |
| “Permission 把进程沙箱化了” | “Permission 是应用层策略门；OS sandbox 需要额外的系统级隔离。” |
| “Compaction 删除了聊天记录” | “Compaction 替换 active model representation；durable transcript 通常仍保留。” |
| “有 durable history，所以崩溃后自动继续” | “历史可重读，但 Session Drain 与 ownership 是 process-local；自动 post-crash continuation 仍未完成。” |
| “源码里有 V2 Context Epoch，所以当前 TUI 已使用” | “native V2 API/Runner 已接线，但固定 SHA 下当前 TUI 普通消息仍走兼容 `SessionPrompt`。” |
| “Context Snapshot 能恢复代码” | “代码工作树 Snapshot 才用于 diff/restore；Context Snapshot 只比较 Context Sources。” |
| “Checkpoint 就是任何保存点” | “本文的 Checkpoint 专指 native V2 completed Compaction 形成的 active-history boundary。” |

## 6. 状态与证据限制

1. **当前默认路径**：固定 SHA 下，普通 TUI 调用兼容 Session API 并进入 `SessionPrompt.prompt -> SessionPrompt.loop`。native V2 API 与 Runner 已可达，但未接管普通 TUI。入口证据：`packages/tui/src/component/prompt/index.tsx`，`submitInner`，1092-1146；`packages/opencode/src/session/prompt.ts`，`SessionPrompt.prompt/loop`，1052-1071、1343-1347；`packages/core/src/session.ts`，`V2Session.prompt`，360-386。
2. **V2 Context/Persistence**：typed System Context、durable admission/promotion、Context Epoch 核心流程、Tool settlement、durable terminal state、Model Tool Output 和 completed checkpoint 已实现；runtime-context parity、move reset、manual compaction、clustered ownership 与 post-crash continuation 仍不完整。canonical parity 证据：`specs/v2/session.md`，101-173。
3. **实测边界**：正式文档 08-11 记录了任务 7 的代表性测试；本文不重复测试日志。与 Context/Persistence 直接相关的记录是 8 个代表文件共 241 pass、1 skip、0 fail，另有旧 runtime `auto=false` overflow 定向 1 pass；这不覆盖 hard crash、V2 `auto=false + Provider overflow`、move/revert 与 epoch interaction 或真实 Provider。

更深入的机制与限制分别见 [08 Agent 与 Orchestration](./08_Agent_and_Orchestration.md)、[09 Context 与 Persistence](./09_Context_and_Persistence.md)、[10 Tools 与 Security](./10_Tools_and_Security.md)、[11 Runtime Boundary](./11_Runtime_Boundary.md) 和 [12 V1/V2 对照](./12_V1_V2_Comparison.md)。

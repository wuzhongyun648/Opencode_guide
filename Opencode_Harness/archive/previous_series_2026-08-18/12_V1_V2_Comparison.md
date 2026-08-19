# OpenCode Session Runtime：V1/V2 分模块比较

> **验收状态**：任务 6 最终交叉审计尚未完成；任务 8 已按用户指示跳过，未进行 Teach-back 理解验收。本文不是最终审计结论，也不能视为任务 8 已通过。  
> **固定源码**：`0e3474509aa5ad16afcf9c439785514d6443c6af`（下文 `@S` 均指这个完整 SHA）。  
> **核对日期**：2026-08-18。  
> **比较对象**：当前默认旧 Session Orchestration、为迁移保留的 compatibility surface，以及独立的 native V2 Session Runtime。这里的 V1/V2 是架构和兼容性标签，不是产品版本号。

本文首次使用的关键术语统一为：**提供商轮次（Provider Turn）**严格指一次 Provider request 及其对应 response；一次 Retry 会在同一 Assistant/Processor context 内形成多个 provider request attempts，也就是多个 Provider Turns，不能把它们合并成一个 Provider Turn。**系统上下文（System Context）**是系统级 typed facts，**上下文纪元（Context Epoch）**是同一不可变 baseline 生效的跨度，**工具结算（Tool Settlement）**是把工具调用（Tool Call）确定为成功、失败或中断并形成可持久化 terminal state 的过程，**子代理（Subagent）**是由 Task 等能力启动的另一 Agent/Session 工作流，**权限（Permission）**是应用层策略门，**操作系统沙箱（OS Sandbox）**是限制进程真实系统权限的隔离边界。**上下文快照（Context Snapshot）**只记录各上下文源（Context Source）最近被接纳的模型隐藏状态；**代码工作树快照（Code Worktree Snapshot）**则完整指面向 Git 工作树与索引的文件状态基线，用于计算 patch/diff 和 restore，二者不能互换。

## 1. 先定义四个容易混淆的范围

在固定 SHA 下，这四个名称不是同义词：

| 名称                                 | 本文定义                                                                                                                                                                                                                                                     | 不能推出什么                                                                                   |
| ------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------- |
| **当前默认旧 Session Orchestration** | 默认 TUI 普通消息实际调用 `client.session.prompt(...)`，进入 `POST /session/:id/message -> SessionHttpApi.prompt -> SessionPrompt.prompt/loop`。它使用旧 Session Message/Part 合同和旧编排，但已复用 EventV2、Core Projector、共享 LLM Event 和组合 Server。 | 不能因它复用新基础设施就称为 native V2；也不能因称为“旧”就认为它已经停止工作。                 |
| **compatibility**                    | 为当前 TUI、旧 API/SDK、旧 Message/Part/Event、存储投影和迁移保留的合同与桥接层，例如 `/session/*`、`SessionV1`、`message.part.*`、`EventV2Bridge`。                                                                                                         | compatibility 不等于完全独立的旧程序；它可以调用 current 基础设施。                            |
| **native V2**                        | `V2Session.prompt -> SessionInput.admit -> SessionExecution -> SessionRunner` 组成的 Effect-native 独立路径，经 `/api/*`、Protocol、Server、Client 和 `sdk-next` 暴露。                                                                                      | “已实现”只表示该 slice 有入口、实现和测试，不表示当前 TUI 默认使用，也不表示 V1 parity 完成。  |
| **current namespace**                | 迁移目标中的无版本 canonical 命名。Schema 根导出使用 `Session`、`Permission` 等无版本名；`@opencode-ai/protocol` 和 `@opencode-ai/sdk-next` 属于 current `/api` surface。`V2` 是应逐步移除的过渡名。                                                         | current namespace 不是“当前 TUI 默认路径”的别名；无版本类名也不能单独证明某项 runtime 已启用。 |

命名规则的直接证据是 `packages/schema/AGENTS.md:12-25`，符号/章节 `Current Versus V1`、`Events` @S：current contract 无版本，兼容合同显式保留 `V1`，replacement architecture 不应永久保留 `V2`。默认入口的反证是 `packages/tui/src/component/prompt/index.tsx:1092-1146`，`submitInner()` 普通消息分支 @S；native 对照是 `packages/server/src/handlers/session.ts:139-171`，Handler `session.prompt`，以及 `packages/core/src/session.ts:360-386`，`V2Session.prompt` @S。

因此，本文使用“V1”作为当前兼容旧 Session Runtime 的比较简称，而不是把 OpenCode 产品版本 `1.18.18`、Git tag `github-v1.2.25` 或 SDK 目录名当成架构版本。

## 2. 阅读入口

- [06 Harness 总览](./06_Harness.md)
- [07 OpenCode Runtime 术语](./07_OpenCode_Runtime_Terminology.md)
- [08 Agent 与 Orchestration](./08_Agent_and_Orchestration.md)
- [09 Context 与 Persistence](./09_Context_and_Persistence.md)
- [10 Tools 与 Security](./10_Tools_and_Security.md)
- [11 Runtime Boundary](./11_Runtime_Boundary.md)

本文只做迁移对照，不重复上述专题的完整执行解释。状态词统一为：

| 状态          | 含义                                                                                 |
| ------------- | ------------------------------------------------------------------------------------ |
| `implemented` | native V2 路径在固定 SHA 下已有实际入口/调用点、实现和测试；仍需另看是否为当前默认。 |
| `partial`     | 已覆盖一部分语义，但入口接线、旧能力 parity、Provider/Tool 覆盖或故障语义仍不完整。  |
| `missing`     | 固定 SHA 的源码 TODO 或 `specs/v2/` 明确没有 native 等价实现；计划不是当前能力。     |

## 3. 当前 executable 的共存方式

图中矩形节点分别表示调用者、HTTP 合同、Session 编排器、事件/投影和消费端；实线箭头表示调用、写入或投影关系，不表示前一节点被后一节点替换。两条 Prompt 路径不互相代理：它们只在共享 EventV2/SQLite 基础设施处汇合。图中的桥接节点仅指 `EventV2Bridge + GlobalBus` 把共享事件转换为 compatibility 事件供当前 TUI 消费；它不会把 `SessionPrompt` 桥成 native `SessionRunner`，也不会让 compatibility route 获得 native API 语义。

```mermaid
flowchart TB
    TUI[当前 TUI 普通消息] --> CS[兼容 JS SDK<br/>client.session.prompt]
    CS --> CR[POST /session/:id/message]
    CR --> CH[SessionHttpApi.prompt]
    CH --> OLD[SessionPrompt.prompt / loop<br/>旧 Session Orchestration]

    NC[显式 native Client<br/>client.v2 / @opencode-ai/client / sdk-next] --> NR[POST /api/session/:id/prompt]
    NR --> NH[packages/server Session Handler]
    NH --> NEW[V2Session.prompt<br/>SessionInput + SessionExecution + SessionRunner]

    OLD --> EV[EventV2 + SQLite transaction + Projectors]
    NEW --> EV
    EV --> CP[旧 message/part projection<br/>+ compatibility events]
    EV --> VP[native session_input/session_message<br/>+ session.next.* events]
    CP --> BR[EventV2Bridge + GlobalBus]
    BR --> UI[Worker RPC 或 /global/event SSE<br/>当前 TUI reducer]
    VP --> VE[/api/event live<br/>per-session durable cursor/history]
```

这不是两个 executable 互相代理。`packages/opencode/src/server/routes/instance/httpapi/server.ts:154-181,271-312` 的 `instanceApiRoutes`、`serverRoutes`、`createRoutes` @S 把 compatibility routes 与 native `/api` routes 合并在同一个 Effect Router layer tree 中；`298-303` 同时提供 `SessionV2` 和 `SessionExecutionLocal`。当前 TUI 的调用点仍是兼容 SDK，所以 **native V2 可达** 与 **当前 TUI 默认使用 native V2** 是两个不同命题。

EventV2 也是共享基础设施，不是 Session 架构判据。旧 `Session.updateMessage/updatePart` 发布 V1 durable events，Core Projector 写旧表；native Runner 发布 `session.next.*` 并写 native projection。证据：`packages/opencode/src/session/session.ts:631-645`，`Session.updateMessage/updatePart`；`packages/core/src/event.ts:205-438`，`commitDurableEvent/publishEvent/notify`；`packages/core/src/session/projector.ts:210-393`，V1 与 native projectors @S。

## 4. 入口和执行

| 比较点                 | 当前默认 compatibility                                                                                                                         | native V2                                                                                                                                  | 状态                                          |
| ---------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------ | --------------------------------------------- |
| Prompt 入口与返回      | `client.session.prompt` 调用 `/session/:id/message`；Handler 等待整个 `SessionPrompt` Loop，返回 final `SessionV1.WithParts`。                 | `/api/session/:id/prompt` 先 durable admission，返回 `SessionInput.Admitted` receipt；默认只 advisory wake，`resume:false` 可 admit-only。 | native `implemented`；当前 TUI 接线 `missing` |
| 输入进入可见历史的时点 | 先写 visible User Message，再逐个写 Parts，然后进入 Loop；Message 与全部 Parts 不是原子写入。                                                  | `PromptAdmitted` 先写 `session_input`；Runner 在安全边界发布 `Prompted`，同一事务标记 promoted 并追加 visible User Message。               | `implemented`                                 |
| 同 Session 执行串行化  | `SessionRunState.ensureRunning` 让并发 loop callers 等待同一 Runner。                                                                          | process-global `SessionExecution` 按 Session ID 找 Location；Coordinator join resume、coalesce wake，不同 Session 可并行。                 | `implemented`，仅 process-local               |
| Provider Turn loop     | `SessionPrompt.run` 每轮重载历史，`SessionProcessor` 发出恰好一个 Provider request并处理对应 response，再按 `continue/compact/stop` 回到外层。 | `SessionRunner.run` 处理 steer/queue 与 continuation；每个 `runTurnAttempt` 恰好一次 `llm.stream(request)`，结算 Tool 后重载历史。         | `implemented`                                 |
| Provider transport     | 默认 AI SDK `streamText`；`OPENCODE_EXPERIMENTAL_NATIVE_LLM` 只切换旧 Loop 下的 Native Adapter。                                               | Runner 直接使用 canonical `@opencode-ai/llm`/`LLMClient`；只覆盖明确列出的 routes。                                                        | `partial`                                     |
| `agent.steps`          | 达到阈值只追加 `MAX_STEPS_PROMPT`，仍发送 Tools，不硬停止。                                                                                    | final allowance Turn 不物化 Tools，设置 `toolChoice:none`；违规本地 Tool Call 失败结算且不 continuation。                                  | `implemented`，语义不同                       |

| 比较点                 | 迁移价值与兼容层                                                                                                                         | 证据 @S                                                                                                                                                                                                                                                          |
| ---------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Prompt 入口与返回      | 分开“请求已接受”和“Provider 已完成”，支持 exact retry、pending steer/queue。两个 endpoint 在同一 Router 并列；旧 POST 保留最终响应语义。 | `packages/sdk/js/src/v2/gen/sdk.gen.ts:3737-3795,5617-5656`；`packages/opencode/src/server/routes/instance/httpapi/handlers/session.ts:295-309`；`packages/server/src/handlers/session.ts:139-171`；`packages/core/src/session.ts:360-386`，`V2Session.prompt`。 |
| 输入进入可见历史的时点 | 解耦 admission、transcript visibility 和立即执行。V1-to-V2 shadow 可为已可见旧 prompt 发布 `Prompted`；旧表仍服务旧路径。                | `packages/opencode/src/session/prompt.ts:995-1049`；`packages/core/src/session/input.ts:41-168,216-288`；`packages/core/src/session/projector.ts:348-374`；`specs/v2/session.md:13-50`。                                                                         |
| 同 Session 执行串行化  | 分开执行所有权与 Location-scoped Runner，明确 wake 只是 advisory。两套 coordinator 不桥接。                                              | `packages/opencode/src/session/run-state.ts:52-94`；`packages/core/src/session/execution/local.ts:10-46`；`packages/core/src/session/run-coordinator.ts:24-104`。                                                                                                |
| Provider Turn loop     | admission、promotion、每个严格的一请求一响应 Provider Turn、Tool Settlement 和 continuation 各有边界。V2 不桥接 `SessionPrompt.loop`。   | `packages/opencode/src/session/prompt.ts:1081-1341`；`packages/opencode/src/session/processor.ts:627-683`；`packages/core/src/session/runner/llm.ts:173-406`；`specs/v2/todo.md:18-36`。                                                                         |
| Provider transport     | 统一 request/stream 合同；旧 Native Adapter 只是 transport compatibility，不是 native V2 Session。                                       | `packages/opencode/src/session/llm.ts:224-381`；`packages/opencode/src/session/llm/native-runtime.ts:46-145`；`packages/core/src/session/runner/model.ts:131-215`；`specs/v2/provider-model.md:268-284`。                                                        |
| `agent.steps`          | 把“最后一步”从提示强化为可执行限制；两条路径没有语义转换。                                                                               | `packages/opencode/src/session/prompt.ts:1132-1139,1178-1179,1279-1282,1334-1335`；`packages/core/src/session/runner/llm.ts:202-214,243-249,383-405`。                                                                                                           |

## 5. Agent、Model 和 Orchestration

| 比较点                    | 当前默认 compatibility                                                                                         | native V2                                                                                                                       | 状态                                        |
| ------------------------- | -------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------- |
| Agent Registry 与选择     | `Agent.Info` 含 prompt、model、variant、permission、steps；默认通常选择可见 primary `build`。                  | `AgentV2.select`、build/plan/general/explore 和 V2 config schema 已实现；每个 Provider Turn 重新选择 effective Agent。          | 核心 `implemented`；工作流 parity `partial` |
| Agent-local Model/Request | Model 优先级包含 input、Agent、Session、history、Provider default；Agent variant/request policy 进入请求准备。 | Agent config 可存 model/variant/request，但 Runner resolver 主要看 Session/Catalog；Agent-local request policy 未完整应用。     | `partial`                                   |
| build/plan 工作流         | 权限、提醒和 `plan_enter/plan_exit` 已形成可用流程，用户配置可覆盖默认规则。                                   | 基础 Agent 和部分权限轮廓已存在；switch reminder、`plan_exit` 等仍有缺口。                                                      | `partial`                                   |
| continuation 判定         | finish reason、最新 User、Tool Part、Processor result、Compaction 和 error 共同决定。                          | 本地 Tool Call 设置 `needsContinuation`；steer/queue 按 delivery 规则 promotion；Provider error/decline 阻止相应 continuation。 | `implemented`                               |

| 比较点                    | 迁移价值与兼容层                                                                                                                | 证据 @S                                                                                                                                                                                    |
| ------------------------- | ------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Agent Registry 与选择     | Agent 是 typed policy/config，不是 Model 或 prompt 的别名。同名 build/plan 延续，但不保证 `plan_exit`、switch reminder parity。 | `packages/opencode/src/agent/agent.ts:35-55,98-340`；`packages/core/src/agent.ts:67-105`；`packages/core/src/plugin/agent.ts:96-202`；`packages/core/src/tool/builtins.ts:18-48`。         |
| Agent-local Model/Request | 目标是组合 Agent policy 与 canonical Catalog/Session resolution；旧路径继续提供成熟行为，native 不伪装等价。                    | `packages/opencode/src/session/prompt.ts:614-689`；`packages/core/src/config/plugin/agent.ts:80-111`；`packages/core/src/session/runner/model.ts:181-215`；`specs/v2/session.md:137-143`。 |
| build/plan 工作流         | 保留 policy 边界，避免只靠提示文本；同名 Agent 只方便迁移配置。                                                                 | `packages/opencode/src/agent/agent.ts:98-181`；`packages/opencode/src/session/reminders.ts:15-90`；`packages/core/src/plugin/agent.ts:120-150`；`specs/v2/session.md:137-144`。            |
| continuation 判定         | 不把 Provider finish reason 当成完整 Orchestration 状态。两边都重载 projection，但状态合同和 delivery vocabulary 不同。         | `packages/opencode/src/session/prompt.ts:1100-1130,1288-1339`；`packages/core/src/session/runner/llm.ts:231-347,383-405`；`packages/core/src/session/input.ts:245-288`。                   |

这里没有“旧差、新好”的单向结论。当前旧路径具有更完整的 Agent request policy、Plan/Build workflow、Provider coverage、Task/Subagent 和 Plugin 集成；V2 对 admission、delivery、steps 和 Provider Turn 边界建模更明确，但尚未覆盖全部旧行为。

## 6. Context、System Context 和 History

| 比较点                     | 当前默认 compatibility                                                                                                                                             | native V2                                                                                                                                        | 状态                                        |
| -------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------- |
| system-level 内容          | 每轮重新拼接 Provider/Agent prompt、Environment、Project/MCP/Skill instructions、policy、per-prompt system，再允许 Plugin transform；它不是 typed System Context。 | typed 上下文源（`Context Source`）、Registry、Baseline System Context、Context Snapshot 和 Context Epoch；变化成为 durable Mid-Conversation System Message。 | 核心 `implemented`；旧来源 parity `partial` |
| Context 来源覆盖           | 支持 provider identity、configured/remote/nested instructions、MCP、Plugin transforms、per-prompt system 等。                                                      | 已有 Environment/date、global/upward `AGENTS.md`、selected-agent Skill 和 reference guidance；其余多项仍 partial/missing。                       | `partial`                                   |
| 会话历史（Session History） | 从 `message`/`part` 重载，经 Compaction、Pruning、provider compatibility 和 reminder/plugin transform 生成 model-visible messages。                                | 按 aggregate sequence、Context Epoch 和 completed checkpoint cutoff 选择 chronological projection。                                              | `implemented`                               |
| 初始 Context 与输入顺序    | User Message visible 后才准备本轮 system-level 内容。                                                                                                              | 完整 baseline 在 Prompt Promotion 前初始化；初始 source unavailable 会阻止 Turn 并保留 pending input。                                           | `implemented`                               |
| Context Epoch 生命周期     | 无 Context Epoch；每轮重算 system-level 字符串。                                                                                                                   | initialization、reconciliation、Compaction replacement 已接线；move reset 未找到调用，replacement 覆盖唯一 row。                                 | `partial`                                   |
| Prompt/reference expansion | 可展开 File、directory、media、MCP Resource、Agent mention、templates 和 synthetic input。                                                                         | typed attachments complete；materialization、template/`@` mention、agent/configured-reference expansion 仍 partial/missing。                     | `partial`                                   |

| 比较点                     | 迁移价值与兼容层                                                                                                                      | 证据 @S                                                                                                                                                                                                                      |
| -------------------------- | ------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| system-level 内容          | 为动态事实提供稳定 key、typed snapshot、确定顺序和时序更新。旧字符串 assembly 继续服务当前 TUI，不自动转换为 V2 Context Source。      | `packages/opencode/src/session/prompt.ts:1221-1286`；`packages/opencode/src/session/llm/request.ts:56-205`；`packages/core/src/system-context/index.ts:21-80,131-320`；`packages/core/src/session/context-epoch.ts:23-174`。 |
| Context 来源覆盖           | 逐来源明确 precedence、unavailable/removal 和 durable admission；parity 前保留旧 assembly。                                           | `specs/v2/session.md:123-151`；`CONTEXT.md:116-129`。                                                                                                                                                                        |
| Session History            | 分开 durable domain state 与模型可见 representation。V1 Message/Part 与 native `session_message` projector 并存，不共享合同。         | `packages/opencode/src/session/message-v2.ts:98-123,131-415,425-575`；`packages/core/src/session/history.ts:13-99`。                                                                                                         |
| 初始 Context 与输入顺序    | 防止不完整 privileged baseline 与已消费输入形成不可重放状态；compatibility 保持原顺序。                                               | `packages/opencode/src/session/prompt.ts:1052-1071,1221-1286`；`packages/core/src/session/runner/llm.ts:168-216`；`specs/v2/session.md:54-82`。                                                                              |
| Context Epoch 生命周期     | 划定 provider-cache baseline 跨度。旧路径无需模拟 Epoch；不能从 `reset` 存在推断 move reset 已生效。                                  | `packages/core/src/session/context-epoch.ts:23-89,111-174`；`packages/core/src/control-plane/move-session.ts:77-138`；`packages/core/src/session/projector.ts:242-255`；`specs/v2/session.md:82-109`。                       |
| Prompt/reference expansion | 明确 durable admission 前后的解析失败、重放和来源 identity。V1 synthetic expansion 可被 replay，但部分内容仍仅由 compatibility 创建。 | `packages/opencode/src/session/prompt.ts:635-1050`；`specs/v2/session.md:146-151`。                                                                                                                                          |

术语必须保持：System Context 不等于任意 System Prompt；Session History 不等于 System Context；Context Snapshot 只保存 Context Source 的模型隐藏比较状态，不能读取、diff 或 restore 文件；只有代码工作树 Snapshot 才记录 Git 工作树/索引文件基线并服务 patch、diff 和 restore。定义证据：`CONTEXT.md:7-52,88-135`，`Language/Relationships` @S。

## 7. Persistence、Event 和 Projection

| 比较点                        | 当前默认 compatibility                                                                                              | native V2                                                                                            | 状态                                       |
| ----------------------------- | ------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------- | ------------------------------------------ |
| Session/Message/Part 持久化   | 旧 Session Service 发布 V1 durable event；Projector 写 `session/message/part`；User Message 与各 Part 独立提交。    | `session.next.*` events 投影到 `session_input`、`session_message`、Context Epoch 等 current tables。 | `implemented`                              |
| Event transaction             | durable event 在 SQLite transaction 中执行 Projector、hook、sequence 和 event row，提交后通知；live-only 直接通知。 | 使用同一 EventV2 不变量，并增加 public durable Session cursor/history。                              | `implemented`                              |
| Stream fragment durability    | whole Message/Part durable；`message.part.delta` live-only；硬崩溃前未 whole-save 的后缀可能丢失。                  | Started/Ended durable、Delta live-only；Ended 保存完整累计值。                                       | `implemented`，仍有硬崩溃窗口              |
| Event projection 与 transport | whole updates 经 EventV2、Bridge/GlobalBus，再通过 Worker RPC 或 `/global/event` SSE 到 TUI。                       | `/api/event` 是 instance live；per-session event 是 durable replay-and-tail；history 提供有限页。    | API `implemented`；当前 TUI 集成 `missing` |

| 比较点                        | 迁移价值与兼容层                                                                                                               | 证据 @S                                                                                                                                                                                  |
| ----------------------------- | ------------------------------------------------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Session/Message/Part 持久化   | 以 aggregate sequence 组织 facts、inbox 和 typed messages。EventV2 同时注册两类 projector，旧表供 compatibility 读取。         | `packages/opencode/src/session/session.ts:631-645`；`packages/core/src/session/projector.ts:210-393`；`packages/core/src/session/sql.ts:68-98,168-176`。                                 |
| Event transaction             | 原子推进 projection 与 event sequence，分开 durable replay 和 live notification。`EventV2Bridge` 转换为旧 GlobalEvent/`sync`。 | `packages/core/src/event.ts:205-438,541-604`；`packages/opencode/src/event-v2-bridge.ts:35-65`；`packages/core/test/event.test.ts:157-321,422-518`。                                     |
| Stream fragment durability    | 避免每 token 写库，同时保留正常结束后的重放边界。两类 delta 属于不同 event family，均不能从 durable cursor 重放。              | `packages/opencode/src/session/processor.ts:278-313,486-597`；`packages/schema/src/v1/session.ts:502-507,612-641`；`packages/schema/src/session-event.ts:197-373,448-520`。              |
| Event projection 与 transport | 显式区分 live 与 durable cursor。当前 TUI 丢弃 `sync`，继续使用 compatibility events + GET hydration。                         | `packages/tui/src/context/event.ts:9-19`；`packages/protocol/src/groups/session.ts:306-343`；`packages/server/src/handlers/session.ts:332-364`；`packages/core/src/session.ts:346-359`。 |

**EventV2 复用不等于旧 Session 已变成 V2。** EventV2回答“事实如何提交、投影和通知”；Session Orchestration回答“输入何时可见、谁运行Provider Turn、Tool如何continuation”。当前默认路径在前者使用新基础设施，在后者仍由`SessionPrompt`主导。

## 8. Compaction、Snapshot 和 Recovery

| 比较点                      | 当前默认 compatibility                                                                                 | native V2                                                                                                                           | 状态                                                        |
| --------------------------- | ------------------------------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------- |
| Automatic Compaction        | synthetic marker + summary + retained tail；旧 rows 通常保留，Pruning 另标旧 Tool output。             | request-budget 预估；Started/Ended 形成 completed checkpoint，summary + recent 成为 history boundary，再 rebaseline Context Epoch。 | `implemented`；deterministic old Tool pruning `missing`     |
| Overflow 与 `auto=false`    | 禁用 threshold；Provider overflow 保存 Assistant error 并停止。                                        | 跳过预压缩，但特定无 durable output/side effect 的 overflow 仍可尝试一次 checkpoint；该组合未实测。                                 | `partial`                                                   |
| 代码工作树 Snapshot/Revert  | Git 工作树与索引 Snapshot 用于 patch/diff/restore；Revert 协调文件恢复与 conversation suffix cleanup。 | Location-scoped 工作树 Snapshot 与 durable Revert events 已有；commit 删除 boundary 后的 native projections。                       | Snapshot/Revert `implemented`；Context Epoch 交互 `partial` |
| 受控恢复                    | 可重载 durable Message/Part；interrupt 尽力 whole-save 并封口 Tool；重启不自动恢复 Runner。            | inbox、history、checkpoint、工具 terminal state 与 Model Tool Output 可重读；Tool Settlement 是产生终态的过程，不是可重读实体；新 Runner fail 遗留 running Tool，显式 resume 可继续。 | `partial`                                                   |
| Manual summarize/compaction | `POST /session/:id/summarize` 可用：创建 compatibility Compaction 并运行旧 Loop。                      | `POST /api/session/:id/compact` endpoint 和 Handler 已存在，但 `V2Session.compact` 当前返回 `OperationUnavailableError`。           | compatibility `implemented`；native operation `missing`     |
| Wait                        | 没有独立的 compatibility wait endpoint；不能由 summarize/compaction 的可用性推出 wait 可用。           | `POST /api/session/:id/wait` endpoint 和 Handler 已存在，但 `V2Session.wait` 当前返回 `OperationUnavailableError`。                 | compatibility 无 endpoint；native operation `missing`       |
| post-crash continuation     | durable history 可重载，但没有精确的自动跨崩溃 continuation contract。                                 | admission 已实现；Provider dispatch ambiguity、post-tool continuation、startup discovery、retry/abandon 仍 deferred。               | `missing`                                                   |

| 比较点                      | 迁移价值与兼容层                                                                                                                          | 证据 @S                                                                                                                                                                                                                                                           |
| --------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Automatic Compaction        | 保留 durable 审计历史时控制模型窗口，避免跨已改变 prefix 重放 metadata。两套 representation 并存，不能互读为同一 marker。                 | `packages/opencode/src/session/compaction.ts:215-317,319-582`；`packages/core/src/session/compaction.ts:176-247`；`packages/core/src/session/history.ts:13-99`；`specs/v2/session.md:111-121`。                                                                   |
| Overflow 与 `auto=false`    | 有界重建同一逻辑工作，同时避免 side effect 后盲重试。两条语义不同，旧测试不能替代 V2 组合验证。                                           | `packages/opencode/src/session/overflow.ts:22-34`；`packages/opencode/src/session/processor.ts:599-625`；`packages/core/src/session/compaction.ts:176-247`；`packages/core/src/session/runner/llm.ts:277-345,355-380`。                                           |
| 代码工作树 Snapshot/Revert  | 文件状态恢复与 conversation projection 撤回分开。Context Snapshot 完全不参与文件读取、diff 或恢复；V1/V2 Revert events/projections 并存。 | `packages/opencode/src/snapshot/index.ts:318-443,779-795`；`packages/opencode/src/session/revert.ts:38-124`；`packages/core/src/session/revert.ts:27-121`；`packages/core/src/session/projector.ts:394-450`。                                                     |
| 受控恢复                    | 避免静默重放 Tool side effect。两条路径都重读 durable state，但没有共同 durable execution identity。                                      | `packages/opencode/src/session/run-state.ts:52-94`；`packages/opencode/src/session/processor.ts:539-683`；`packages/core/src/session/runner/llm.ts:119-139,277-405`。                                                                                             |
| Manual summarize/compaction | compatibility summarize 继续承担现有用户能力；native route 的存在不等于 compact operation 可用。                                          | `packages/opencode/src/server/routes/instance/httpapi/groups/session.ts:303-315`；`packages/opencode/src/server/routes/instance/httpapi/handlers/session.ts:273-292`；`packages/protocol/src/groups/session.ts:225-239`；`packages/core/src/session.ts:417-420`。 |
| Wait                        | compatibility 没有对应 shim；native route 会把当前 unavailable domain operation 映射为 503。                                              | `packages/protocol/src/groups/session.ts:240-254`；`packages/server/src/handlers/session.ts:197-218`；`packages/core/src/session.ts:421-424`；`packages/opencode/test/server/httpapi-session.test.ts:639-663`。                                                   |
| post-crash continuation     | 防止 Provider/Tool 结果未知时自动重放并重复副作用；没有安全的 compatibility shim。                                                        | `specs/v2/session.md:153-169`；`specs/v2/todo.md:56-74`；`packages/core/src/session/runner/llm.ts:43-90`。                                                                                                                                                        |

“可恢复读取”不等于“自动续跑”。durable Message、Event、input或cursor只能证明事实仍可读取；要安全续跑还必须知道Provider是否已接收请求、Tool副作用是否发生、settlement是否提交以及谁拥有当前执行。

## 9. Tools、Permission 和 Output

| 比较点                                             | 当前默认 compatibility                                                                | native V2                                                                                                                  | 状态                                                |
| -------------------------------------------------- | ------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------- |
| Tool Registry 与 materialization                   | Built-in、Custom、Plugin、MCP 经不同适配；每轮形成 AI SDK Tool map 并按 policy 过滤。 | typed `Tool.make`；Application + Location scope；materialization 捕获 registration identity，stale call 拒绝替代 handler。 | 核心 `implemented`；来源 parity `partial`           |
| Tool Call 与 settlement                            | AI SDK 或旧 Native Adapter dispatch；Processor 将 Tool Part 结算为 completed/error。  | 先 durable 发布 `Tool.Called`，再并发 typed settlement；结束后等待 fibers 并重载 history。                                 | `implemented`                                       |
| Permission 规则与 approval                         | 最后匹配；leaf Tool 自行 `ctx.ask`；pending 与 `always` approval 在 Instance 内存。   | configured deny 不被 saved approval 覆盖；`always` 按 project 写 SQLite；pending Deferred 仍 process-local。               | `implemented`                                       |
| Permission 与 Sandbox                              | `bash` 以 OpenCode 进程用户权限启动 host 进程；Plugin 可直接 I/O。                    | V2 bash 同样拥有 host filesystem/process/network authority；未引入 OS 隔离。                                               | 两边均不是 Sandbox                                  |
| Model output bounding                              | 多处 Truncate/history 裁切；Registry `after` hook 可在通用 Truncate 后重新放大输出。  | codec/projection 后统一 `ToolOutputStore.bound`；Bash producer cap、media/structured payload 仍有边界。                    | generic bound `implemented`；full capture `partial` |
| Built-in/Custom/Plugin/MCP/StructuredOutput parity | 均已有发现、物化和执行路径。                                                          | built-ins 只覆盖一部分；Custom、Plugin hooks、MCP、StructuredOutput 仍 missing/partial。                                   | `partial`                                           |

| 比较点                                             | 迁移价值与兼容层                                                                                                              | 证据 @S                                                                                                                                                                                                                                    |
| -------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Tool Registry 与 materialization                   | 统一 codec、scope 和 invocation identity，避免同名热替换错配。旧 Registry/MCP 继续服务 compatibility。                        | `packages/opencode/src/tool/registry.ts:86-249,286-335`；`packages/opencode/src/session/tools.ts:41-134,390-490`；`packages/core/src/tool/tool.ts:18-132`；`packages/core/src/tool/registry.ts:42-121`。                                   |
| Tool Call 与 settlement                            | 在副作用前建立 durable invocation identity。`providerExecuted` 在两边都跳过同名本地 executor，但 event shape 不同。           | `packages/opencode/src/session/processor.ts:160-253,315-419`；`packages/core/src/session/runner/llm.ts:202-345`；`packages/core/src/session/runner/publish-llm-event.ts:144-193,291-394`；`packages/schema/src/session-event.ts:273-373`。 |
| Permission 规则与 approval                         | 持久化长期 approval 并明确优先级。Application/Plugin Tool 不会被 Registry 自动注入 Permission，trusted leaf 必须 assert。     | `packages/opencode/src/permission/index.ts:18-167,204-219`；`packages/core/src/permission.ts:76-101,131-285`；`packages/core/src/permission/saved.ts:37-79`。                                                                              |
| Permission 与 Sandbox                              | Permission 只决定受管动作是否允许；Sandbox 限制恶意代码的真实权限。compatibility 层不能升级为 Sandbox，需容器/VM/低权限账户。 | `packages/opencode/src/tool/shell.ts:428-595`；`packages/opencode/src/plugin/index.ts:141-166`；`packages/core/src/tool/bash.ts:97-196`；`specs/v2/session.md:193-206`。                                                                   |
| Model output bounding                              | 约束模型可见且持久化的 projection。旧 Truncate 与 native store 并存；after-hook 缺口未反向修复。                              | `packages/opencode/src/tool/tool.ts:130-144`；`packages/opencode/src/session/tools.ts:111-130`；`packages/core/src/tool-output-store.ts:13-28,112-174`；`packages/core/src/tool/bash.ts:21,77,154-196`；`specs/v2/tools.md:153-170`。      |
| Built-in/Custom/Plugin/MCP/StructuredOutput parity | 扩展目标是 typed/scoped lifecycle；parity 前保留旧 Registry、Plugin 和 MCP runtime。                                          | `packages/core/src/tool/builtins.ts:18-48`；`packages/core/src/plugin/host.ts:20-219`；`specs/v2/session.md:139-150,187-215`；`specs/v2/tools.md:182-186`。                                                                                |

结论不是“V2 Tool 已全面更安全”。V2已经把typed codec、stale identity、durable settlement、saved approval和generic output bound做成可测试slice；但extension parity、producer完整捕获、post-crash副作用处理仍不完整，而且Permission在两条路径中都不是Sandbox。

## 10. Todo、Task 和 Subagent

| 比较点              | 当前默认 compatibility                                                                                             | native V2                                                                                                     | 状态                                    |
| ------------------- | ------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------- | --------------------------------------- |
| Todo                | `todowrite` 经 Permission 后替换 Session 清单并发布更新；不启动任务，也不决定 Loop 停止。                          | `SessionTodo.update/get` 和 permission-checked `todowrite` 已实现，复用 Todo 表。                             | `implemented`                           |
| Task Tool/Subagent  | `task` 创建/恢复带 `parentID` 的子 Session，应用 Subagent policy，运行独立 `SessionPrompt.prompt`，回传最后 Text。 | 有 `mode:"subagent"` 和 `parentID` 数据，但 create 不接收 `parentID`，Registry 无 Task Tool，父子编排未实现。 | `missing`                               |
| Background Subagent | 仅实验 flag 启用；完成后向父 Session 注入 synthetic User Message，父取消可递归取消。                               | BackgroundJob、V2 Tool 执行和 background dispatch 仍是 next slice。                                           | native `missing`；旧路径 `experimental` |
| 子任务安全边界      | `task_id` 恢复路径未显式校验属于当前父 Session；深度默认 1。                                                       | 尚无对应编排和 parity 测试。                                                                                  | `missing`，旧边界待审计                 |

| 比较点              | 迁移价值与兼容层                                                                                                                       | 证据 @S                                                                                                                                                                                                         |
| ------------------- | -------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Todo                | 保留可观察计划但不把 Todo 当 scheduler。同一存储可迁移，Tool 合同和 Runner 不同。                                                      | `packages/opencode/src/tool/todo.ts:14-46`；`packages/opencode/src/session/todo.ts:29-66`；`packages/core/src/session/todo.ts:26-78`；`packages/core/src/tool/todowrite.ts:25-62`。                             |
| Task Tool/Subagent  | 仍需父子 ownership、policy 继承、取消和 durable completion delivery。当前需要该能力时走 compatibility `TaskTool`；字段不能充当执行层。 | `packages/opencode/src/tool/task.ts:81-347`；`packages/core/src/session.ts:79-84,208-262`；`packages/schema/src/session.ts:18-44`；`packages/core/src/tool/builtins.ts:18-48`；`specs/v2/todo.md:11-14,50-54`。 |
| Background Subagent | 需要 durable status、completion delivery、cancel/continuation。旧实验 flag 不是 V2 开关。                                              | `packages/opencode/src/tool/task.ts:96-102,216-347`；`packages/opencode/src/session/run-state.ts:111-143`；`specs/v2/todo.md:50-54`。                                                                           |
| 子任务安全边界      | 恢复句柄需绑定父子 ownership，不能把可读 ID 当强能力；不应复制未审计行为。                                                             | `packages/opencode/src/tool/task.ts:104-117,136-172`；`packages/opencode/test/tool/task.test.ts:219-469`。                                                                                                      |

必须明确：**V2 仍缺 Task/Subagent parity。** `mode:"subagent"`、`parentID`字段、未来BackgroundJob计划或Todo Tool都不能替代父子Session编排的入口、继承、执行和结果回传。

## 11. Client、Server、Event 和 SDK

| 比较点                   | 当前默认 compatibility                                                          | native V2                                                                                                                 | 状态                                                |
| ------------------------ | ------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------- |
| Server API 组合          | executable 同时承载 `/session/*`、`/global/event` 和 `/api/*`。                 | Protocol 定义 contract，Server 提供 Handler；同一 Router 可 network 或 embedded 运行。                                    | `implemented`                                       |
| Client 合同              | 当前 TUI 使用旧 JS SDK compatibility group。                                    | `@opencode-ai/client` Promise/Effect clients 由 HttpApi 生成，调用 native `/api`。                                        | `implemented`；beta namespace 待稳定                |
| sdk-next/Embedded        | 当前 TUI 不使用 `sdk-next`。                                                    | `OpenCode.create` 组合 Client、Core、embedded routes，以 in-memory HttpClient 保留 HTTP middleware/codec，不开 listener。 | `implemented`；整文件测试 Harness 有 lifecycle 限制 |
| 实时与 durable Event API | GlobalEvent 经 Worker RPC 或 `/global/event`；无 cursor，TUI 靠 GET hydration。 | `/api/event` 为 volatile live；per-session event 为 durable replay-and-tail；history 为 finite resync。                   | API `implemented`；native TUI `missing`             |
| TUI 集成                 | Prompt、abort、reducer、hydration 全部使用 compatibility 合同。                 | Prompt receipt、native events、durable cursor 尚未接入当前 TUI store。                                                    | `missing`                                           |
| 生成名称与目录           | 生成器产生 `Session2`、`Session3`，路径含 `src/v2`。                            | 架构判定来自 getter、HTTP path、Handler 和 Core 调用链。                                                                  | 不适用，证据规则                                    |

| 比较点                   | 迁移价值与兼容层                                                                                            | 证据 @S                                                                                                                                                                                                                              |
| ------------------------ | ----------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Server API 组合          | 新合同独立演进但不删除旧入口。route merge 是共存层，不是 HTTP 反向代理。                                    | `packages/opencode/src/server/routes/instance/httpapi/server.ts:154-181,271-312`；`packages/server/src/api.ts:1-8`；`packages/server/src/handlers.ts:21-40`；`packages/server/src/routes.ts:26-68`。                                 |
| Client 合同              | Client 不依赖 Core/Server，网络和 embedded 共享合同。旧 SDK 保留 top-level `session` 与嵌套 `v2`。          | `packages/client/src/contract.ts:14-38`；`packages/client/src/generated/client.ts:370-381,449-483,811-816`；`CONTEXT.md:139-170,201-216`。                                                                                           |
| sdk-next/Embedded        | embedded 只替换 transport，不复制 Handler。`sdk-next` 尚未接管 legacy consumers。                           | `packages/sdk-next/src/opencode.ts:10-43`；`CONTEXT.md:139-164`；`packages/sdk-next/test/embedded.test.ts:9-212`。                                                                                                                   |
| 实时与 durable Event API | 分开低延迟 live 与可续接事实。Bridge/GlobalBus 继续为旧 TUI 转换事件，当前 TUI 丢弃 `sync`。                | `packages/opencode/src/event-v2-bridge.ts:35-65`；`packages/tui/src/context/event.ts:9-19`；`packages/protocol/src/groups/event.ts:29-45`；`packages/protocol/src/groups/session.ts:306-343`；`packages/core/src/event.ts:152-164`。 |
| TUI 集成                 | 需重新处理 admission UI、final output、cursor 与 delta 合并，不是替换 URL；compatibility TUI 仍是稳定路径。 | `packages/tui/src/component/prompt/index.tsx:392-421,1092-1146`；`packages/tui/src/context/sdk.tsx:23-131`；`packages/tui/src/context/sync.tsx:321-415,594-667`。                                                                    |
| 生成名称与目录           | codegen 消歧和 SDK 目录不是架构证据；同一生成文件承载两类 group。                                           | `packages/sdk/js/src/v2/gen/sdk.gen.ts:3737-3795,5617-5656,7006-7009,7195-7218`。                                                                                                                                                    |

## 12. Reliability：Interrupt、Retry 和 Crash

| 比较点                      | 当前默认 compatibility                                                                                                                                                                                              | native V2                                                                                               | 状态                                                 |
| --------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------- | ---------------------------------------------------- |
| Interrupt                   | abort 当前 Runner/Provider stream；cleanup 尽力保存输出并将 Tool 标为 interrupted；副作用不回滚。                                                                                                                   | coordinator 中断本进程 owner、清 pending wake；Runner durable fail 活跃 Tool/Assistant；idle 为 no-op。 | `implemented`，均 process-local                      |
| 一般 Provider Retry         | `SessionRetry.policy` 在同一 Assistant/Processor context 内最多安排 5 次重试，由此形成多个 provider request attempts；每次 attempt 都是独立的一请求一响应 Provider Turn。重试前不重载 history，已投影输出未必无痕。 | 一般 timeout/retry/backoff/status 尚未实现，规格明确 deferred。                                         | native `missing`                                     |
| Doom Loop                   | 同一 Assistant 最近 3 个同名同参数 Tool 触发 `doom_loop` Permission ask。                                                                                                                                           | repeated identical Tool Call 保护仍在 Runner TODO。                                                     | native `missing`                                     |
| 崩溃后的 Tool 封口          | 重启不自动续跑；下一 Loop 可把未结算 Tool 降为 interrupted/orphan。                                                                                                                                                 | Runner 开始前将 pending/running local Tool durable fail，不重放 side effect。                           | orphan closure `implemented`；continuation `missing` |
| clustered ownership/fencing | 无跨进程精确 Session execution ownership。                                                                                                                                                                          | Coordinator 仅 process-local；distributed acquisition、fencing、cross-process interrupt 未实现。        | `missing`                                            |
| Client 断线恢复             | `/global/event` 自动重连但无 cursor；GET hydration 恢复 whole state，live-only 后缀不可恢复。                                                                                                                       | per-session durable cursor 可续接；`/api/event` 仍 volatile，generated client 不自动重连。              | API `implemented`；UI policy `partial/missing`       |

| 比较点                      | 迁移价值与兼容层                                                                                                                                         | 证据 @S                                                                                                                                                                                                                      |
| --------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Interrupt                   | 受控取消形成可读 settlement。endpoint/contracts 不同，均无跨进程 owner。                                                                                 | `packages/opencode/src/session/run-state.ts:77-86,111-143`；`packages/opencode/src/session/processor.ts:539-683`；`packages/core/src/session/run-coordinator.ts:94-103`；`packages/core/src/session/runner/llm.ts:277-347`。 |
| 一般 Provider Retry         | native 需联合设计 durable failure、idempotency、drain release 和可见状态。旧 Retry 继续服务 compatibility；overflow checkpoint recovery 不是一般 Retry。 | `packages/opencode/src/session/processor.ts:627-676`；`packages/opencode/src/session/retry.ts:84-205`；`packages/core/src/session/runner/llm.ts:43-90`；`specs/v2/session.md:153-165`。                                      |
| Doom Loop                   | 防止重复副作用并保留用户决策；旧规则没有 native bridge。                                                                                                 | `packages/opencode/src/session/processor.ts:29-30,331-380`；`packages/opencode/src/agent/agent.ts:119-136`；`packages/core/src/session/runner/llm.ts:43-90`。                                                                |
| 崩溃后的 Tool 封口          | 区分封口未知状态与重新执行副作用。两边都不盲目重放，但记录合同不同。                                                                                     | `packages/opencode/src/session/prompt.ts:96-100,1103-1129`；`packages/core/src/session/runner/llm.ts:119-139`。                                                                                                              |
| clustered ownership/fencing | 多进程共享存储须防双 owner。Event replay owner claim 与 execution ownership 分离。                                                                       | `specs/v2/session.md:101-109,165-185`；`specs/v2/todo.md:53-74,105-116`；`CONTEXT.md:104,165-177`。                                                                                                                          |
| Client 断线恢复             | 调用方需区分 refresh、durable resume 和 live subscribe。当前 TUI 保留 SSE reconnect + snapshot hydration。                                               | `packages/tui/src/context/sdk.tsx:82-117`；`packages/tui/src/context/sync.tsx:451-667`；`CONTEXT.md:165-170`。                                                                                                               |

必须明确：**native V2 仍缺一般 Retry、clustered ownership 和 post-crash continuation。** durable admission、durable event cursor、process-local interrupt或orphan closure都不能填补这些能力。

## 13. 迁移状态总表

| 模块                           | V2已实现                                                                                                       | Partial                                             | Missing / 尚未接管                                                                                                | 当前迁移判断                                            |
| ------------------------------ | -------------------------------------------------------------------------------------------------------------- | --------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------- |
| 入口与执行                     | native Prompt admission、exact retry、steer/queue、local Runner、显式Provider Turn                             | Provider route coverage、部分request policy         | 当前TUI native入口；durable/clustered execution                                                                   | native slice可达，默认仍是compatibility。               |
| Agent/Model/Orchestration      | Agent Registry、基础built-ins、continuation、强steps allowance                                                 | Agent-local model/request、build/plan workflow      | switch reminders与若干Tools                                                                                       | 不能用同名Agent宣称行为等价。                           |
| Context/System Context/History | typed Context algebra、baseline/snapshot、chronological update、history cutoff                                 | Context来源覆盖、Epoch lifecycle                    | configured/remote/nested/MCP/plugin/per-prompt parity，move reset接线                                             | 核心模型已成形，runtime-context parity未完成。          |
| Persistence/Event/projection   | EventV2、sequence transaction、native inbox/message projections、durable cursor                                | hard-crash fragment边界                             | 当前TUI native event消费                                                                                          | 共享EventV2不代表共享Session Runtime。                  |
| Compaction/Snapshot/Recovery   | automatic checkpoint、rebaseline、native代码工作树Snapshot/Revert、orphan closure；compatibility summarize可用 | `auto=false` overflow组合、Revert/Epoch interaction | native manual compact/wait operation、old-tool pruning、post-crash continuation；compatibility无独立wait endpoint | route存在不等于operation可用；durable可读比自动恢复窄。 |
| Tools/Permission/Output        | typed Registry、stale identity、durable settlement、saved Permission、generic bound                            | producer capture、built-in覆盖                      | Custom/Plugin/MCP/StructuredOutput完整parity                                                                      | V2边界更集中，但能力面更窄。                            |
| Todo/Task/Subagent             | Todo                                                                                                           | 无完整Task slice                                    | Task Tool、父子Session、Subagent结果回传、background dispatch                                                     | V2仍缺Task/Subagent parity。                            |
| Client/Server/Event/SDK        | Protocol/Server/Client、embedded sdk-next、live与durable APIs                                                  | beta namespace、Harness lifecycle                   | current TUI迁移                                                                                                   | API implemented不等于UI默认。                           |
| Reliability                    | process-local join/wake/interrupt、durable failure settlement                                                  | reconnect policy、显式resume                        | 一般Retry、Doom Loop、cluster ownership、post-crash continuation                                                  | 可靠性不应由“durable”一词笼统概括。                     |

## 14. 用户怎样判断某条能力是否真的启用

不要从文件名开始判断，应从实际调用链反向确认：

1. **先确认调用者。** 当前TUI普通提交若仍是`packages/tui/src/component/prompt/index.tsx:1092-1146`的`sdk.client.session.prompt`，就是compatibility Session Orchestration。显式使用`client.v2.session.prompt`、`@opencode-ai/client.sessions.prompt`或`sdk-next`才可能进入native路径。
2. **确认HTTP合同。** `/session/:id/message`是旧Loop最终响应；`/api/session/:id/prompt`是native admission receipt。不要只看方法叫`prompt`。
3. **继续追Handler和Core。** `SessionHttpApi.prompt -> SessionPrompt.prompt/loop`是当前旧编排；`packages/server SessionHandler -> V2Session.prompt -> SessionExecution`才是native。
4. **检查返回语义。** 返回final Assistant `WithParts`表示compatibility POST；快速返回`SessionInput.Admitted`只证明native admission，仍不证明Provider已完成。
5. **检查事件订阅。** `/global/event` + GlobalEvent/TUI reducer是compatibility live路径；`/api/event`是native instance live；`/api/session/:id/event?after=N`才是native durable cursor。事件带`seq`不自动表示当前订阅会replay。
6. **区分专用实验flag。** `OPENCODE_EXPERIMENTAL_NATIVE_LLM`只切换旧`SessionPrompt`下的Provider adapter；`OPENCODE_EXPERIMENTAL_BACKGROUND_SUBAGENTS`只开启旧Task Tool的实验后台分支。两者都不是“启用整个V2”的总开关。
7. **检查能力自己的接线和测试。** 找到定义后，还要找到上层调用点、runtime materialization/Handler和对应测试；只有`specs/v2/`计划、Schema字段或TODO不算启用。
8. **最后核对运行证据边界。** 定向测试通过只支持该slice；Harness超时、临时测试和fake Provider结果不能外推到真实Provider、完整TUI或跨进程恢复。

可以用下表快速识别：

| 观察                                       | 可以判断                                   | 不能判断                                                          |
| ------------------------------------------ | ------------------------------------------ | ----------------------------------------------------------------- |
| `POST /session/:id/message`                | 正在使用compatibility Prompt合同           | Event/Persistence一定是纯旧实现                                   |
| `POST /api/session/:id/prompt`返回Admitted | native admission已发生                     | Provider Turn或Tool已完成                                         |
| `POST /session/:id/summarize`成功          | compatibility summarize/compaction流程可用 | 存在独立compatibility wait endpoint                               |
| `/api/session/:id/compact`或`wait`路由存在 | native HTTP合同与Handler已声明             | 当前operation可用；固定SHA下Core仍返回`OperationUnavailableError` |
| EventV2写入event row                       | durable Event基础设施在工作                | Session Orchestration已切V2                                       |
| `OPENCODE_EXPERIMENTAL_NATIVE_LLM=true`    | 旧Loop尝试native LLM adapter               | native SessionRunner已启用                                        |
| `mode:"subagent"`或`parentID`字段存在      | 数据合同有相应概念                         | native Task/Subagent编排已实现                                    |
| SDK类名`Session2/Session3`                 | codegen存在重名消歧                        | 架构V1/V2归属                                                     |
| 测试文件名含`v2`且通过                     | 对应测试slice可运行                        | 当前TUI默认或完整parity                                           |

## 15. 不能下的结论

以下说法在固定 SHA 下都不成立或证据不足：

- “产品版本是1.x，所以架构一定是V1”，或“目录/类型叫V2，所以产品已全面进入V2”。
- “V2已实现”等于“当前TUI默认已使用V2”。
- “旧Session复用了EventV2”就等于“旧Session Orchestration已变成native V2”。
- “同一executable提供`/api`”就等于“旧路由被代理或替换”；实际是并列route merge。
- “Permission允许/拒绝工具”就等于“Tool运行在OS Sandbox”。
- “有durable event、inbox或cursor”就等于“Provider/Tool work可自动post-crash continuation”。
- “V2有Todo、`subagent` mode或`parentID`”就等于“Task/Subagent parity已完成”。
- “一次overflow后能Compaction重建”就等于“V2已有一般Provider Retry”。
- “compatibility summarize/compaction可用”就等于“compatibility有独立wait endpoint”。
- “native compact/wait endpoint存在”就等于“`V2Session.compact/wait`当前可用”。
- “current namespace无版本前缀”就等于“该合同已经是当前TUI默认入口”。
- “`packages/opencode/src/session/message-v2.ts`、`packages/sdk/js/src/v2/`或生成类名`Session2/Session3`”可以作为native架构证据。
- “测试引用存在”就等于“任务7实际运行通过”；只有明确记录的运行集合才是执行证据。
- “V2用了Effect/模块更多”就足以证明更可靠；可靠性结论必须落到durable边界、所有权、重试、interrupt和crash语义。

## 16. 任务 7 结果与测试 Harness 限制

任务 7 使用固定 SHA、Bun fixtures、fake/mock LLM、临时 SQLite 和临时目录；没有真实 Provider 密钥、付费调用或真实外部 MCP。这里仅保留可用于比较的证据摘要，逐命令、耗时、断言数和实验过程见对应 research 记录：

| 模块                | 证据摘要                                                                                                                                                                              | 证据边界与详细记录                                                                                                                                                                                                                  |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Agent/Orchestration | 代表性旧路径测试除一次取消超时外通过；该用例定向重跑通过。隔离实验确认旧 `steps:1` 仍发送 Tools，并形成至少 3 个严格的一请求一响应 Provider Turns。                                   | 不证明无限循环，也不排除取消竞态；Doom Loop、跨父 `task_id` ownership、部分输出后的 Retry 未实测。见 [10 Research，第 18.1 节](./research/10_Research_Agent_and_Orchestration.md#181-任务-7-最小验证2026-08-18)。                   |
| Context/Persistence | Event transaction/projector、admission/promotion、Context Epoch、checkpoint、旧 Compaction/Revert/代码工作树 Snapshot 等代表性 slices 通过；旧 `auto=false + overflow` 定向验证通过。 | 未覆盖 hard crash、Message/Part 中途故障、native `auto=false + overflow`、move/revert 与 Epoch 交互。见 [11 Research，第 11.2 节](./research/11_Research_Context_and_Persistence.md#112-任务-7-环境与范围)。                        |
| Tools/Security      | 旧 Tool/Permission/MCP/read 与 native Registry/Permission/Output Store 代表性 slices 通过；隔离实验确认 compatibility after-hook 可在 Truncate 后重新放大输出。                       | 未跑完整 Provider -> Tool Part -> SQLite -> 下一请求链，也未覆盖真实 MCP/Provider、大附件或 native orphan 组合。见 [12 Research，第 10.4 与 12 节](./research/12_Research_Tools_and_Security.md#104-任务-7-执行记录)。              |
| Runtime Boundary    | route composition、TUI hydration race、native Client contract、admission receipt、admit-only/exact retry/default wake 等 slices 有运行证据。                                          | 未动态证明完整 compatibility POST 响应时点、真实 Worker/TCP 拓扑、主动 SSE 断线/cursor 重连或 post-crash continuation。见 [13 Research，第 10.4 节](./research/13_Research_Runtime_Boundary.md#104-任务-7-最小验证结果与后续实验)。 |

必须保留两项 Harness 失败：

1. `httpapi-sdk.test.ts` 独立和串行复核仍出现本地 Effect test server 502/空 response、SSE 超时及级联断言失败；compatibility fake-LLM Prompt filter 未在读取 `prompt.data` 前取得 data。因此 compatibility POST 等待 final Assistant 本轮只有固定 Handler 控制流与测试意图证据，没有动态证据；这也不能反向解释为产品 Prompt/SSE 语义失败。详细统计与复核过程见 [Runtime Boundary research](./research/13_Research_Runtime_Boundary.md#失败原因与限制)。
2. `sdk-next/embedded.test.ts` 整文件稳定受临时数据库路径与 module-global layer 生命周期污染影响，而各 test name 分进程隔离时通过。它支持各 slice 可独立工作，但不能抹去整文件 Harness 失败，也不能把失败写成 sdk-next 产品语义失败。详细证据见 [Runtime Boundary research](./research/13_Research_Runtime_Boundary.md#失败原因与限制)。

任务 6 的最终交叉审计仍未完成；任务 8 明确跳过且未验收。因此本文只是固定源码、规格、正式模块文档与任务 7 证据的阶段性比较，不声称最终审计完成。

## 17. 主要比较结论

1. 当前状态是**旧Session Orchestration + compatibility contracts +共享新基础设施 + 可达的native V2 slice**共存，不是两个完全隔离的产品，也不是一次性替换完成。
2. V2在durable admission/promotion、typed System Context、Context Epoch、typed Tool settlement、durable Session cursor和显式delivery语义上提供了更清晰的状态边界；这些是具体改动，不是因“用了Effect”自动获得的质量。
3. 当前默认路径在Provider、Agent request policy、Plugin/MCP/Custom Tool、StructuredOutput、Task/Subagent和成熟TUI集成方面覆盖更广。迁移不能只比较核心抽象而忽略用户能力parity。
4. **V2已实现不等于当前TUI默认。EventV2复用不等于旧Session已变V2。** 判断必须追踪调用者、HTTP合同、Handler、Core和返回/事件语义。
5. Provider Turn 严格是一请求一响应。compatibility Retry 可在同一 Assistant/Processor context 内产生多个 provider request attempts，也就是多个 Provider Turns；native V2 仍缺一般 Retry。
6. compatibility summarize/compaction 可用，但没有独立 compatibility wait endpoint；native compact/wait endpoint 已声明，`V2Session.compact/wait` 当前仍返回 `OperationUnavailableError`。
7. V2仍明确缺少Task/Subagent parity、Doom Loop、clustered ownership、stale-owner fencing和post-crash continuation；Context、Provider、Tools、Plugin与UI integration也存在partial/missing项。
8. Permission在两条路径中都是应用层策略门，不是Sandbox；Interrupt/settlement也不回滚已经发生的外部副作用。
9. current namespace描述canonical命名和依赖方向，不描述默认入口采用率；文件名、SDK生成类名和产品版本号都不是架构启用证据。

## 18. 规格与源码证据入口

以下入口均固定为 @S，可用于复核本文表格：

- `packages/schema/AGENTS.md:12-27`，current/V1命名和event边界。
- `CONTEXT.md:7-52,88-199`，Session Runtime术语、Client/Event/Tool output关系。
- `specs/v2/session.md:3-185`，Session admission、Context Epoch、Compaction、parity、Retry与Recovery状态。
- `specs/v2/tools.md:3-186`，typed Tool、registration、settlement、bounding和follow-up。
- `specs/v2/todo.md:11-74,105-142`，Subagent、Runner、BackgroundJob、cluster/recovery与hardening待办。
- `specs/v2/provider-model.md:268-284`，当前native Runner Provider适配范围。
- `packages/tui/src/component/prompt/index.tsx:1092-1146`，当前默认Prompt调用点。
- `packages/opencode/src/server/routes/instance/httpapi/server.ts:154-181,271-312`，current executable新旧route composition。
- `packages/opencode/src/session/prompt.ts:1052-1347`，当前默认Prompt、Loop和continuation。
- `packages/core/src/session.ts:346-452`，native events/history/prompt/compact/wait/resume/interrupt/revert facade。
- `packages/core/src/session/runner/llm.ts:43-90,119-406`，native Runner已实现流程和明确TODO。
- `packages/core/src/event.ts:152-164,205-438,541-604`，EventV2 live/durable transaction与replay。

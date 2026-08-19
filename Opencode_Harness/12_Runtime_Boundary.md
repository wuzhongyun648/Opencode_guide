# Runtime Boundary：OpenCode 的 Harness 在哪里运行

上一篇：[11 Agent 专业化与协作](./11_Agent_Specialization_and_Collaboration.md)

系列入口：[OpenCode Harness 架构学习系列](./README.md)

## 1. 学习问题：一条消息实际跨过了哪些边界

你已经知道 OpenCode 会组织 Context、调用 Model、执行 Tool 并保存 Session。现在换一个观察角度：这些工作在哪里发生？

当零基础学习者在 TUI 输入：

> 请读取 Harness 教程的 README 和项目规则，告诉我应该从哪一篇开始；只做低风险观察，不修改文件。

界面会很快出现状态和文本。但这并不表示 Prompt 请求本身正在逐 token 返回，也不表示 TUI、Server、Provider 和 Tool 都在同一个模块中执行。默认本地模式、监听模式和远程 `attach` 还会改变传输方式。

### 最短答案

TUI 是客户端，Session Orchestrator、Tool Runtime 和持久化服务位于 OpenCode Server/Harness 一侧，Model 请求越过 Provider 边界，普通本地 Tool 仍回到 OpenCode 一侧执行。

默认本地 TUI 通过 Worker RPC 调用进程内 Router，不需要 TCP；远程 TUI 通过 HTTP 提交 Prompt，并通过独立 SSE 连接接收实时事件。当前 Prompt POST 等 Agent Loop 完成后返回最终 JSON，实时界面由另一条事件通道驱动。

源码中还存在一条已接线的 native V2 Runtime。它和当前默认兼容 Runtime 共用部分事件、存储与 Server 基础设施，但 Prompt 入口、执行器和返回语义不同；固定源码下，当前 TUI 尚未切换到 native V2。

## 2. 最小心智模型：先分角色，再看传输

### 2.1 逻辑角色不等于进程

先认识参与者：

| 逻辑角色 | 主要职责 | 是否天然是独立进程 |
| --- | --- | --- |
| TUI | 收集输入、显示状态、维护客户端 Store | 否，它是客户端运行时中的 UI |
| SDK | 把方法调用编码成 HTTP 合同 | 否，它是客户端模块 |
| Worker RPC Adapter | 本地模式中转发 Request 和 Event | 否，它表示 Worker/RPC 执行边界，不应直接等同于独立 Server 进程 |
| Server Router / Handler | 匹配 API，调用 Session 服务 | 否，Router 可以被内存 `fetch` 调用，也可以挂到网络 listener |
| Session Orchestrator | 运行 Agent Loop、组织 Provider Turn | 否，它是 Server/Harness 内的服务 |
| Provider | 接收模型请求并返回流 | 可能是外部网络服务，也可能是本地 Provider；由 Provider 部署决定 |
| Tool Runtime | 校验、授权并执行普通本地 Tool | 通常在 OpenCode Server/Harness 一侧 |
| Event / Persistence | 保存状态并向观察者发布更新 | 通常在 OpenCode Server/Harness 一侧 |

如果把每一个源码模块都画成独立进程，图会产生错误直觉。Runtime Boundary 关注的是：调用跨过了模块边界、Worker 边界还是网络边界，以及状态是否跨进程持久化。

### 2.2 一张总图

```text
                 请求路径
用户 -> TUI -> SDK -> [Worker RPC 或 HTTP] -> Router / Handler
                                              |
                                              v
                                        Session Orchestrator
                                         /              \
                                        v                v
                          Provider transport          Tool Runtime
                                 |                        |
                                 v                        v
                              Model 服务            文件 / 进程 / API

                 实时返回路径
Session / Tool 状态 -> Event -> [Worker RPC event 或 SSE] -> TUI Store
```

读这张图时要注意：

- Prompt 请求和实时事件是两条通道；
- Provider 和 Tool 的执行方向不同；
- Worker RPC 和 HTTP 可以进入同一个 Router；
- “经过 Router”不等于“经过 TCP 网络”。

## 3. 三种运行拓扑

### 3.1 默认本地 TUI：Worker RPC

没有显式网络选项时，可以把默认拓扑理解为：

```text
一个 CLI 宿主
├─ TUI 侧：Prompt、SDK、Reactive Store
└─ Bun Worker 边界：Router、Session Runtime、Tool Runtime、GlobalBus
   └─ Provider transport -> Model Provider
```

TUI 创建 Worker，并给 SDK 注入两种适配器：

- `createWorkerFetch(...)` 负责请求；
- `createEventSource(...)` 负责事件。

SDK 仍然构造标准 Request。TUI 侧把 URL、method、headers 和 body 通过 RPC 交给 Worker；Worker 重建 Request 后调用：

```text
Server.Default().app.fetch(request)
```

这个调用经过完整 Router、Middleware 和 Handler，因此业务合同与 HTTP 服务一致。但它直接调用内存中的 Web Handler，没有为 `opencode.internal` 建立 TCP 连接。

这里的 `fetch` 是 Request/Response 编程接口，不是“必然访问网络”的同义词。

### 3.2 本地监听模式：同一 Router 挂到 socket

使用 `--port`、`--hostname` 或相关服务发现选项时，TUI 可以让 Worker 启动 listener，再通过 HTTP 请求同一 Router，并由 `/global/event` SSE 接收事件。这次存在 socket，但 Session Runtime 不会因此自动改变；传输变成 HTTP 不等于切换到 native V2。

### 3.3 远程 `attach`：真正跨机器也仍是同一兼容入口

`opencode attach <url>` 不注入本地 Worker fetch 和 event source。TUI 使用普通网络 fetch，并连接远程 Server 的 SSE：

```text
本地 TUI --HTTP POST /session/:id/message--> 远程 OpenCode Server
本地 TUI <--SSE GET /global/event----------- 远程 OpenCode Server
远程 Server --Provider transport-----------> Model Provider
```

远程模式改变 Client 与 Server 的位置，不改变普通 TUI 消息的 Session 入口。固定源码下，它仍调用兼容 `client.session.prompt(...)`，仍进入 `SessionPrompt`。

## 4. 贯穿场景：读取 Harness 学习入口

下面沿着当前默认本地 TUI 走一遍，但只展开运行边界，不重复第 07 篇的 Agent Loop 细节。

### 4.1 请求怎样进入 Session Runtime

```text
TUI -> client.session.prompt
    -> Worker RPC: POST /session/:id/message
    -> Router -> SessionHttpApi.prompt
    -> SessionPrompt.prompt / loop
    -> one or more Provider Turns
    -> final Assistant WithParts -> one buffered JSON response
```

生成 SDK 中的类名可能带数字，那只是代码生成时的重名消歧，不能拿来判断是否进入 native V2。判断 Runtime 必须继续追 URL、Handler 和 Core 调用点。

### 4.2 为什么 Prompt POST 不是 token stream

兼容 Handler 使用了 streaming response API，但实际顺序是：

```text
先等待 promptSvc.prompt(...整个 Loop...)
-> 得到 final message
-> JSON.stringify(message)
-> 用只有一个值的 Stream 构造 response body
```

默认本地 Worker 还会调用 `response.text()`，完整读取 body 后再回给 SDK。

因此，这个 POST 的准确语义是“长耗时的最终结果请求”。它可能跨越多个 Provider Turn 和 Tool 执行，但响应 body 不是逐 token 流。

### 4.3 为什么界面仍能提前更新

TUI 提交 Prompt 时没有等待 Promise 完成才继续渲染，而是为失败挂接 `.catch(...)`。实时状态和文字通过另一条 Event Channel 返回：

```text
SessionProcessor / Session Service
-> EventV2
-> EventV2Bridge
-> compatibility GlobalEvent
-> GlobalBus
-> Worker RPC event
-> SDKProvider
-> TUI reducer / Reactive Store
```

如果换成监听模式或远程 `attach`，中间的 Worker RPC event 改成 `/global/event` SSE，前后模块基本不变。

这解释了一个表面矛盾：

```text
Prompt POST 尚未完成
+ 独立事件通道不断送来更新
= 用户已经在 TUI 看见运行过程
```

## 5. 最终响应、实时事件和持久化不是一回事

### 5.1 EventV2、Bridge、Transport 和 Store

看到一个 Part 出现在屏幕上时，需要分别问 Prompt 是否最终返回、当前连接是否收到 live event、状态是否已经 durable；三者可以在不同时间成立。

当前兼容路径大致分成五层：

| 层 | 解决的问题 |
| --- | --- |
| EventV2 + Projector | durable event 如何在 transaction 中投影、写入并在提交后通知 |
| `EventV2Bridge` | 怎样把共享 Event 转成旧客户端理解的 compatibility payload |
| `GlobalBus` | 怎样在 executable 内发布 GlobalEvent |
| Worker RPC / SSE | 怎样把 event 送到当前客户端连接 |
| TUI Store | 怎样合并 Message、Part 和 delta 并驱动界面 |

完整 Message/Part update 可以是 durable；`message.part.delta` 是 live-only。durable 描述能否从存储重读，RPC/SSE 描述现在怎样传输，两者不是二选一。

### 5.2 当前 TUI 不用 `sync` envelope 做重放

`EventV2Bridge` 对 durable event 除了产生普通 compatibility payload，还会产生带 sequence 和 aggregate identity 的 `sync` envelope。

但当前 TUI 的 `useEvent` 会明确丢弃 `payload.type === "sync"`，TUI reducer 消费的是普通 compatibility event。`SyncProvider` 这个名字也不能反推出它在做 EventV2 durable replay；它主要负责 GET hydration 和 live reducer。

进入 Session 页面时，TUI 会通过 GET API 读取 Session、最近消息、Todo 和 Diff，并用 hydration tracker 避免旧 snapshot 覆盖刚到的 live update。这是 snapshot hydration，不是 cursor replay。

## 6. Provider 在哪里，Tool 又在哪里

### 6.1 Provider boundary

当前默认兼容 Runtime 中，`LLM.run` 准备 Model、Messages、Tools、headers 和 Provider options。默认路径通过 AI SDK `streamText(...)` 发起 Provider request，再把 Provider 的原始流适配成统一 `LLMEvent` 交给 `SessionProcessor`。

```text
OpenCode Session Runtime
-> AI SDK Provider Adapter
-> Provider transport
-> Model Provider
```

Provider 可能是远程 API，也可能是本地模型服务；Session 模块只定义调用边界，不决定 Provider 一定部署在哪台机器。

### 6.2 普通本地 Tool 不在 Provider 内执行

Model Provider 可以返回：

```text
read(path="Opencode_Harness/README.md")
```

这只是 Tool Call。真正的读取由 OpenCode 一侧完成：

```text
Provider 产生 Tool Call
-> OpenCode 校验参数和 Permission
-> OpenCode Tool Runtime 读取文件
-> 保存 Tool Result
-> 下一 Provider Turn 再把结果交给 Model
```

Shell、文件写入和其他普通本地 Tool 的副作用也位于 OpenCode Runtime。模型服务不会因为生成了调用 JSON 就直接获得用户文件系统。

例外是标为 `providerExecuted` 的 hosted tool：它由 Provider 执行，OpenCode 会识别该标记并跳过本地同名 dispatch。解释某个 Tool 的执行位置时，需要先确认它属于哪一类。

### 6.3 旧 Loop 下的 Native Adapter 不是 native V2 Session

显式开启 `OPENCODE_EXPERIMENTAL_NATIVE_LLM` 时，当前旧 `SessionPrompt` Loop 可以尝试使用 Native LLM Adapter，不支持的 Provider 或配置再回退到 AI SDK。

这只替换 Provider request/stream 的适配层：

```text
SessionPrompt old Loop
-> Native LLM Adapter
-> Provider
```

外层仍是 `SessionPrompt`，所以不能把“使用 native Provider adapter”写成“使用 native V2 Session Runtime”。

## 7. 当前兼容 Runtime 与 native V2 为什么会共存

### 7.1 同一个 executable 中有两套路由

当前 executable 把以下 Router layer 合并在一起：

- compatibility `/session/*`、`/global/*` 和旧事件入口；
- native `/api/*` Protocol/Server 路由；
- OpenAPI 文档和 UI fallback。

这不是两个 Server 进程互相代理，而是同一个 Router tree 中的并列路由。因此同一个 executable 可以同时响应：

```text
POST /session/:id/message
POST /api/session/:id/prompt
```

“native API 可达”和“当前 TUI 默认使用 native API”是两个不同命题。

### 7.2 两条 Prompt 路径

| 边界 | 当前默认兼容 Runtime | native V2 Runtime |
| --- | --- | --- |
| 客户端调用 | `client.session.prompt(...)` | `client.v2.session.prompt(...)` 或 native Client |
| HTTP | `/session/:id/message` | `/api/session/:id/prompt` |
| Handler | `SessionHttpApi.prompt` | native `SessionHandler` |
| Core | `SessionPrompt.prompt -> loop` | `V2Session.prompt -> SessionInput.admit -> optional wake` |
| POST 返回 | 完整 Loop 后的 final Assistant `WithParts` | durable `SessionInput.Admitted` receipt |
| 当前普通 TUI | 已接线 | 未接线 |

最显著的架构变化，是 native V2 把“输入已经被接受”和“Provider 已经完成工作”分开。

## 8. native V2 的运行模型

### 8.1 Durable admission 与执行调度分离

native Prompt 先写入 durable `session_input`。只有 `resume !== false` 时才调用 `SessionExecution.wake(...)`；`resume:false` 可以只接纳输入，不立即请求执行。

```text
native Prompt POST
-> durable admit
-> 返回 Admitted receipt
-> optional advisory wake
-> Runner 在安全边界 promotion
-> Provider Turn / Tool Settlement
```

因此 native Prompt POST 也不是 token stream，但它和兼容 POST 的等待语义几乎相反：兼容 POST 等最终结果，native POST 只等接纳完成。

### 8.2 Process-local Coordinator 与 Location-scoped Runner

`SessionExecutionLocal` 使用进程级 `SessionRunCoordinator`：同 Session 的 resume 可以 join，wake 可以 coalesce，不同 Session 可以并行；Runner 在 drain 开始时按 Session Location 获取运行服务。执行所有权仍是 process-local，尚无 clustered ownership 和跨进程 fencing。

### 8.3 Native Provider 与 Tool settlement

native `SessionRunner` 对每个 Provider Turn 明确调用一次 `llm.stream(request)`。普通本地 Tool Call 先形成 durable 调用事实，再由 Runner 启动 Tool settlement；需要 continuation 时重新读取投影历史。

这条路径不经过旧 AI SDK `streamText`，也不桥接旧 `SessionPrompt.loop`。当前 Provider route 和 Tool/Plugin 覆盖仍未达到兼容 Runtime 的全部能力面。

### 8.4 Native event API

native V2 提供三种不同观察方式：

| API | 语义 |
| --- | --- |
| `GET /api/event` | 全 Server 的 volatile live stream，不回放历史 |
| `GET /api/session/:id/event?after=N` | Session durable event 的 replay-and-tail，可按 sequence cursor 续接 |
| `GET /api/session/:id/history?after=N&limit=M` | durable history 的有限页查询 |

per-session event cursor 只重放 durable event，不补回 live-only text/reasoning/tool-input delta。当前 TUI 尚未消费这些 native 合同。

## 9. 架构演进解决了什么，又还缺什么

native V2 不是因为包名更新或使用 Effect 就自动更可靠。真正的变化是从“visible User Message -> old Loop -> final response”，演进为“durable admission -> promotion -> process-local execution”，并提供 durable Session cursor/history。

这些变化带来的价值包括：

- Prompt 接纳、模型可见性和立即执行不再绑定成一个动作；
- steer 与 queue 有显式 durable delivery 语义；
- Provider Turn、Tool Call 和 Tool Settlement 的边界更清楚；
- per-session durable cursor 使客户端可以从已知 sequence 续读事实；
- Runner 的 Location scope 为不同工作区运行环境建立了明确接口。

但固定源码下仍需保留以下限制：

- 当前 TUI 仍使用兼容 Prompt、Event 和 hydration 合同；
- native Provider、Context、Tool、Plugin 等能力覆盖仍有 partial 项；
- Task/Subagent 父子 Session 编排尚未完成；
- 一般 Provider Retry 和 Doom Loop 等价保护尚未完成；
- 手动 native `compact` 和独立 `wait` 合同存在，但 Core 操作当前返回 unavailable；
- execution ownership 仍是 process-local；
- clustered execution、stale-owner fencing 和自动 post-crash continuation 尚未实现。

所以准确描述是：native V2 是已接线、可达、可测试但尚未完成全部 parity 的独立 Runtime slice；当前产品主线仍由兼容 Runtime 服务 TUI。

## 10. 断线、中断、恢复与崩溃是四件事

### 10.1 客户端请求失败不会自动回滚 Session

Prompt POST 的网络错误只说明 Client 没有成功取得响应。User Message 可能已经 durable，Provider 可能已经返回部分输出，Tool 也可能已经产生副作用。

请求失败不能被理解成整个 Agent 任务的事务回滚。

### 10.2 断开事件连接不会自动停止 Agent

SSE 是观察通道。只断开 SSE，Server 侧 Session Run 不会因此必然中断；重新连上也不会自动重发 Prompt。当前兼容 SSE 没有 cursor replay，重连只恢复后续 live event；GET hydration 可恢复 durable whole state，却无法保证补回断线窗口内尚未 whole-save 的 live-only 后缀。当前代码也不会在每次 `server.connected` 后自动执行 Session hydration。

### 10.3 Interrupt 取消执行，但不撤销外部副作用

当前兼容 TUI 的 interrupt 最终进入 `SessionPrompt.cancel -> SessionRunState.cancel`。Processor 会尽力保存已有 Text/Reasoning，把未完成 Tool 标记为 interrupted error。

但如果命令已经启动、文件已经写入或外部 API 已经收到请求，持久化“已中断”不等于这些副作用被回滚。

native V2 的 interrupt 会中断当前进程 coordinator 的 active owner，并结算活跃 Tool/Assistant；idle interrupt 是 no-op。它同样不能宣称跨进程取消或回滚外部副作用。

### 10.4 Durable cursor 不等于自动 crash recovery

durable cursor 解决“客户端从哪一个 Session Event 继续读取”。post-crash continuation 要解决的则是：

- Provider 是否已经接收请求；
- Tool 是否已经产生副作用；
- Tool Result 是否已经提交；
- 哪个进程拥有当前执行；
- 是否可以安全重试而不重复行动。

native V2 已有 durable admission 和 per-session cursor，但这些信息不足以消除 Provider/Tool 处于未知状态时的歧义。固定源码明确没有自动 startup continuation policy。

## 11. 低风险观察与常见误解

在默认本地 TUI 提出“只读取 `Opencode_Harness/README.md` 和项目规则，先报告查看的材料，再给出学习顺序，不修改文件”，观察最终回复完成前出现的 Tool Call、状态和文本。它能帮助区分 Prompt 最终响应、实时 Event 和 durable whole state；若出现超出只读范围的 Permission 请求，应检查目标而不是机械批准。

需要避免的误解是：本地 `app.fetch` 不等于 TCP；Handler 使用 stream API 不等于 POST 在逐 token 返回；远程 `attach` 不等于 native V2；普通本地 Tool 不在 Provider 内执行；durable event 不保证当前订阅支持 replay；native API 存在不表示当前 TUI 已迁移；历史可读也不表示崩溃前工作能安全自动续跑。

## 12. 本篇掌握要点

读完本篇，应能解释：

1. TUI、SDK、Router、Session Orchestrator、Provider、Tool Runtime 和 Event/Persistence 是逻辑角色，不天然对应独立进程。
2. 默认本地 TUI 使用 Worker RPC 调用内存 Router；监听模式和远程 `attach` 使用 HTTP/SSE。
3. 当前兼容 Prompt POST 等完整 Loop 后返回最终 JSON，不是 token stream。
4. TUI 的实时更新来自独立的 Worker RPC event 或 `/global/event` SSE。
5. Model Provider 产生 Tool Call，普通本地 Tool 的 Permission、执行和副作用位于 OpenCode 一侧。
6. Provider Native Adapter 只替换旧 Loop 下的 Provider 层，不等于 native V2 Session Runtime。
7. 当前兼容 Runtime 与 native V2 在同一 executable 中并列接线；当前 TUI 仍走兼容入口。
8. native V2 Prompt 返回 durable admission receipt，per-session API 提供 durable cursor，但当前仍缺完整 UI/parity、clustered ownership 和 post-crash continuation。
9. 请求失败、事件断线、Interrupt 和 crash recovery 是不同边界，不能互相替代。

## 13. 关键源码入口

本文结论以 OpenCode 固定 commit `0e3474509aa5ad16afcf9c439785514d6443c6af` 为基线。行号可能随版本变化，优先按函数和导出符号查找。

| 主题 | 文件 | 关键符号 |
| --- | --- | --- |
| 默认本地 TUI transport | `packages/opencode/src/cli/cmd/tui.ts` | `TuiThreadCommand.handler`、`createWorkerFetch`、`createEventSource` |
| Worker Router / Event | `packages/opencode/src/cli/tui/worker.ts` | `rpc.fetch`、Global event forwarding、`rpc.server` |
| TUI Prompt 调用 | `packages/tui/src/component/prompt/index.tsx` | `submitInner()`、`session.interrupt` |
| 远程 attach 与 TUI Event | `packages/opencode/src/cli/cmd/attach.ts`、`packages/tui/src/context/sdk.tsx` | `AttachCommand.handler`、`SDKProvider`、`startSSE` |
| TUI reducer / hydration | `packages/tui/src/context/event.ts`、`packages/tui/src/context/sync.tsx` | `useEvent`、`SyncProvider`、`session.sync` |
| 兼容 Prompt Handler | `packages/opencode/src/server/routes/instance/httpapi/handlers/session.ts` | `SessionHttpApi.prompt`、`SessionHttpApi.abort` |
| 当前 Session Runtime | `packages/opencode/src/session/prompt.ts` | `SessionPrompt.prompt`、`SessionPrompt.loop`、`SessionPrompt.cancel` |
| Provider 默认与 Adapter | `packages/opencode/src/session/llm.ts`、`packages/opencode/src/session/llm/native-runtime.ts` | `LLM.run`、`LLM.stream`、`LLMNativeRuntime.stream` |
| Tool dispatch | `packages/opencode/src/session/tools.ts` | `SessionTools.resolve` |
| Event bridge / SSE | `packages/opencode/src/event-v2-bridge.ts`、`packages/opencode/src/server/routes/instance/httpapi/handlers/global.ts` | `EventV2Bridge`、`eventResponse` |
| 新旧 Router 组合 | `packages/opencode/src/server/routes/instance/httpapi/server.ts` | `serverRoutes`、`createRoutes` |
| native Prompt contract | `packages/protocol/src/groups/session.ts` | `session.prompt`、Session event/history endpoints |
| native Handler | `packages/server/src/handlers/session.ts` | `session.prompt`、`session.events`、`session.interrupt` |
| durable admission | `packages/core/src/session.ts`、`packages/core/src/session/input.ts` | `V2Session.prompt`、`SessionInput.admit`、promotion functions |
| process-local execution | `packages/core/src/session/execution/local.ts`、`packages/core/src/session/run-coordinator.ts` | `SessionExecutionLocal`、`SessionRunCoordinator.make` |
| native Runner | `packages/core/src/session/runner/llm.ts` | `runTurnAttempt`、`SessionRunner.run` |
| 迁移状态 | `specs/v2/session.md`、`specs/v2/provider-model.md`、`specs/v2/todo.md` | Session parity、Provider adaptation、deferred recovery |

至此，主系列已经从 Agent Loop 内部一路走到实际运行边界：模型负责判断，Harness 负责组织和约束，Tool Runtime 负责行动，Session/Event 保存并传递状态，而 Client 通过适合当前拓扑的请求和事件通道观察整个过程。

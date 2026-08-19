# Runtime Boundary：一条消息究竟跨过了哪些边界

上一篇：[11 Agent 专业化与协作](./11_Agent_Specialization_and_Collaboration.md) ｜ 系列入口：[Harness README](./README.md)

> 固定源码：OpenCode `0e3474509aa5ad16afcf9c439785514d6443c6af`（`dev`，2026-08-18）
>
> 分析主线：当前默认本地 TUI 的普通消息。监听模式与远程 `attach` 是互斥拓扑对照；native LLM adapter 与 native V2 是独立演进边界。

当学习者在 TUI 中要求“只读取 Harness README 和项目规则，再给出学习顺序”时，界面会逐步显示状态、Tool Call 和文本。这个直观现象容易带来三个误解：看见 `fetch` 就以为发生了 TCP 请求，看见界面实时更新就以为 Prompt POST 在逐 token 返回，看见 `native` 就以为普通 TUI 已经进入 native V2 Session Runtime。

要判断一段代码或一次运行究竟跨过什么边界，必须同时回答三个不同问题：

```text
逻辑边界：谁负责 UI、路由、编排、模型、工具与状态？
进程边界：这些角色位于 TUI 主线程、Worker 还是另一个进程？
网络边界：哪些数据真正经过 socket、SSE 或 Provider transport？
```

本篇先用这三个维度建立坐标，再比较三种互斥运行拓扑，随后沿当前默认路径追踪一次端到端请求。最后单独解释 Provider/Tool 执行位置、请求与事件双通道，以及 native adapter 与 native V2 的演进和恢复边界。

## 一、逻辑、进程与网络是三个独立维度

### 1.1 逻辑边界回答“谁负责什么”

当前主线可以先拆成八个逻辑角色：

| 逻辑角色 | 主要职责 | 是否天然是独立进程 |
| --- | --- | --- |
| TUI Prompt / Store | 收集输入、显示状态、维护客户端响应式状态 | 否 |
| SDK | 把方法调用编码为 Request/Response 合同 | 否 |
| Transport Adapter | 用 Worker RPC、HTTP 或 SSE 搬运请求与事件 | 否 |
| Server Router / Handler | 匹配 API，进入具体 Session 服务 | 否 |
| Session Orchestrator | 运行 Agent Loop，组织 Provider Turn 与 Tool continuation | 否 |
| Provider Boundary | 把模型请求发送给实际 Provider，接收模型流 | 不确定，取决于 Provider 部署 |
| Tool Runtime | 校验、授权并执行本地 Tool | 通常位于 OpenCode Runtime |
| Event / Persistence | 保存 durable 状态并把运行更新发布给观察者 | 否 |

这些名称描述的是模块职责，不是部署清单。`Server Router` 既可以被内存 `app.fetch(request)` 调用，也可以挂到监听端口；`SessionPrompt` 是服务模块，不因名字中有 Session 就成为独立进程；Provider 可以是远程云 API，也可以是本地模型服务。

### 1.2 进程与 Worker 边界回答“代码在哪个执行上下文”

默认 TUI 会创建 Bun Worker。TUI 侧负责交互、SDK 和 Store，Worker 侧承载 Router、Session Runtime 和事件总线。跨 Worker 需要 RPC 序列化，但这不等同于跨机器网络。

因此可以把边界强度分开理解：

```text
同一函数调用 < 同进程 Worker/RPC < 本机 socket < 跨机器网络
```

它们都能使用 Request/Response 形态，但故障、序列化、取消与性能语义不同。Worker 挂掉会丢失进程内执行状态；HTTP 断开不一定停止 Server 端工作；跨机器还叠加认证、网络分区和远端生命周期。

### 1.3 网络边界回答“是否真正经过 socket”

Web API 中的 `fetch` 是一种编程接口，不保证底层发生 TCP。默认本地路径把构造好的 Request 通过 RPC 交给 Worker，然后直接调用：

```ts
const response = await Server.Default().app.fetch(request)
```

它经过完整 Router 和 Handler，却没有为了 `http://opencode.internal` 建立内部 TCP 连接。只有监听模式或远程 `attach` 才在 TUI 与 Server 之间使用真实 HTTP/SSE。

Provider transport 是另一条网络边界。即使 TUI 与 Server 使用内存 RPC，Server 仍可能通过网络请求远程模型；反过来，本地模型服务也可能使 Provider 位于同一机器。Client/Server 拓扑不能替 Provider 部署作结论。

### 1.4 三条数据路径必须分开追踪

一次任务至少同时存在三条方向不同的路径：

```text
请求路径
TUI -> SDK -> Transport -> Router / Handler -> Session Orchestrator

运行路径
Session Orchestrator -> Provider
                     -> Tool Runtime -> 文件 / 命令 / 外部 API

观察与持久化路径
Session / Tool -> Event / Storage -> RPC event 或 SSE -> TUI Store
```

Prompt 请求、实时 Event 和 durable state 是三种合同。某个状态可以已经通过 Event 出现在界面，但 Prompt POST 仍未完成；某个 whole Message 已经 durable，但客户端当时断开连接；某个 live delta 被当前界面看见，却无法在重启后从存储完整重建。

## 二、三种互斥拓扑：选择一种，不是依次执行

### 2.1 三种方案改变 Transport，不自动改变 Session Runtime

默认本地、监听模式和远程 `attach` 是三种运行方案。一次 TUI 连接只处在其中一种主拓扑中，它们不是“本地 RPC → 本地 HTTP → 远程 HTTP”的三个学习步骤。

| 方案 | 典型选择条件 | TUI 到 Server 请求 | Server 到 TUI 事件 | 主要边界 |
| --- | --- | --- | --- | --- |
| A. 默认本地 TUI | 未显式配置端口、hostname 或服务发现 | Worker RPC + 内存 Router fetch | Worker RPC event | TUI 与 Bun Worker |
| B. 监听模式 | 使用端口、hostname 或 mDNS | HTTP | `/global/event` SSE | socket，可能仍在同一 CLI 宿主 |
| C. 远程 `attach` | TUI 连接已有远程 Server | 跨网络 HTTP | 跨网络 `/global/event` SSE | Client 与远程 Server |

三种方案中的普通 TUI Prompt 都调用兼容 `client.session.prompt(...)`。Transport 出现 HTTP 不等于启用 native V2，Server 位于远程也不等于进入另一套 Agent Loop。

### 2.2 方案 A：默认本地 TUI 使用 Worker RPC

#### 2.2.1 TUI 与 Runtime 位于同一 CLI 宿主的两侧

未开启显式网络选项时，拓扑是：

```text
一个 CLI 宿主
├─ TUI 侧
│  ├─ Prompt
│  ├─ compatibility SDK
│  └─ Reactive Store
└─ Bun Worker
   ├─ Server.Default().app.fetch
   ├─ SessionPrompt / Tool Runtime
   └─ GlobalBus
      └─ Provider transport -> Model Provider
```

#### 2.2.2 Request 与 Event 使用两套 Worker 适配器

TUI 创建 Worker 后，为 SDK 注入两种不同适配器：

```ts
const transport = {
  url: "http://opencode.internal",
  fetch: createWorkerFetch(client),
  events: createEventSource(client),
}
```

`createWorkerFetch` 把 URL、method、headers 和完整 body 交给 RPC；Worker 重建 `Request`，调用内存 Router，再执行 `response.text()`，把完整响应交回 TUI。`createEventSource` 则订阅另一条 `global.event` RPC。

所以默认路径有完整 HTTP 风格合同，却没有内部 TCP；有请求和事件两条通道，却都跨同一个 Worker/RPC 边界。

### 2.3 方案 B：监听模式把同一 Router 挂到 socket

当 `--port`、`--hostname` 或 mDNS 等条件成立，TUI 不再注入 Worker fetch/Event Source。Worker 调用 `Server.listen(...)`，把同一个 Router 挂到 URL：

```text
TUI SDK --HTTP--> listener -> Router / Handler
TUI SDK <--SSE--- /global/event
```

这次确实存在 socket，但不能据此把 Server 自动描述成另一台机器。它仍可能属于同一 CLI 宿主，只是 TUI 与 Server 通过监听器通信。

变化的是 Transport；兼容 Session Handler、`SessionPrompt`、Provider 与 Tool Runtime 主线保持不变。

### 2.4 方案 C：远程 `attach` 使用跨网络 HTTP/SSE

`opencode attach <url>` 只把远程 URL、directory 和认证 headers 传给 TUI，不注入本地 fetch 或 events：

```text
本地 TUI --HTTP POST /session/:id/message--> 远程 OpenCode Server
本地 TUI <--SSE GET /global/event----------- 远程 OpenCode Server
远程 Server --Provider transport-----------> Model Provider
```

SDK 因没有自定义 Event Source 而启动 `/global/event` SSE。普通 Prompt 仍进入远端 Server 上的兼容 `SessionPrompt`。

`attach` 回答“Client 和 Server 在哪里”，native V2 回答“Client 调用了哪套 Session 合同”。这是两个正交问题。

### 2.5 判断实际拓扑时追入口，不猜名称

判断当前运行方案可以按以下证据链：

```text
TUI 是否注入 fetch/events？
-> 是：默认 Worker RPC
-> 否，URL 来自本地 Server.listen：监听模式
-> 否，URL 来自 attach 参数：远程 attach
```

不要只看 `fetch`、URL 是否以 HTTP 开头、是否出现 Server 模块，也不要把三种拓扑画成顺序流程。

## 三、当前默认 TUI 的端到端请求主线

### 3.1 总览：一次 Prompt 可以包含多个 Provider Turn

以默认本地 TUI 的只读学习请求为例，请求主线是：

```text
TUI submit
-> SDK client.session.prompt(...)
-> POST /session/:id/message
-> Worker RPC fetch
-> Server.Default().app.fetch
-> SessionHttpApi.prompt
-> SessionPrompt.prompt / loop
-> 一个或多个 Provider Turn
-> 普通本地 Tool 执行与 continuation
-> final Assistant WithParts
-> 一次完整 JSON response
```

第 07 篇解释了 Loop 内部怎样 continue、compact 或 stop。本篇关注这些模块之间怎样连接，以及哪些返回并不沿 Prompt POST 发生。

### 3.2 TUI 提交兼容 SDK 请求，但不阻塞界面

普通输入分支调用：

```ts
sdk.client.session
  .prompt({
    sessionID,
    ...selectedModel,
    agent: agent.name,
    model: selectedModel,
    variant,
    parts: [...],
  }, { throwOnError: true })
  .catch(showError)
```

这里没有 `await`。TUI 发起 Promise 后继续清空输入和更新本地交互状态，运行中的 Message/Part 再由 Event Channel 驱动。因此“界面在 POST 完成前变化”是设计结果，不需要假设 Prompt body 自身逐 token 返回。

生成 SDK 把 `client.session.prompt` 编码成兼容 `POST /session/{sessionID}/message`。生成类名中可能出现 `Session2` 等数字，它们只是代码生成时的重名消歧，不是 native V2 架构证据。

### 3.3 Worker 复用完整 Router，但缓冲完整响应

#### 3.3.1 Request 跨 Worker 后重新进入 Router

TUI 侧先把 Request body 完整读出并通过 RPC 发送；Worker 侧重建 Request：

```ts
const request = new Request(input.url, {
  method: input.method,
  headers,
  body: input.body,
})
const response = await Server.Default().app.fetch(request)
const body = await response.text()
```

#### 3.3.2 `response.text()` 把返回值变成完整 body

这段代码同时证明两件事：

- 请求确实经过正常 Server Router、Middleware 与 Handler；
- Worker RPC 不保留 streaming response body，而是用 `response.text()` 完整缓冲。

所以默认本地既不是绕开 Server 直接调用 `SessionPrompt`，也不是通过 TCP 请求一个内部 Server。

### 3.4 compatibility Handler 进入旧 `SessionPrompt`

兼容 Route 把 Prompt 与 Message 查询放在 `/session/:sessionID/message` 组下，以 method 区分。`SessionHttpApi.prompt` 验证 Session 后，等待 Prompt 服务：

```ts
const message = yield* promptSvc.prompt({
  ...ctx.payload,
  sessionID: ctx.params.sessionID,
})

return HttpServerResponse.stream(
  Stream.make(JSON.stringify(message)).pipe(Stream.encodeText),
  { contentType: "application/json" },
)
```

`promptSvc.prompt(...)` 在构造 response 前已运行完整 `SessionPrompt` Loop。虽然 Handler 使用 `HttpServerResponse.stream`，Stream 中只有一个最终 JSON 值。

### 3.5 Provider 与 Tool 让主线在 Server 内继续推进

`SessionPrompt` 为每个 Provider Turn 组装 Agent、Model、Context、Tools 和 Permission。Model 可能直接返回文本，也可能提出 `read` Tool Call。普通本地 Tool 经过 OpenCode 侧校验与执行，Tool Result 写回 Session，外层 Loop 再重载历史并发起下一 Provider Turn。

从 Runtime Boundary 看，重要的是两个箭头方向不同：

```text
Server Runtime --Provider transport--> Model Provider
Server Runtime --local Tool dispatch--> 文件系统 / 子进程 / 本地资源
```

模型只产生 Tool Call 意图，普通 Tool 的真实副作用不在 Provider 内执行。

### 3.6 Prompt POST 最后只返回一个完整结果

兼容 Handler 先等完整 Loop，再创建单值 JSON Stream；默认 Worker 又完整读取 `response.text()`。因此 Prompt POST 的准确语义是：

```text
长耗时最终结果请求
≠ token stream
```

它返回 final Assistant `WithParts`，用于完成 SDK Promise。用户在等待期间看到的 Tool Call、状态和增量文本来自另一条通道。

## 四、为什么界面实时更新：请求与事件是双通道

### 4.1 请求通道负责提交与最终返回

请求通道回答：输入是否被 Handler 接受，Session Loop 最终返回什么，HTTP/SDK 调用成功还是失败。

在当前兼容路径中，它等待 final Assistant。这不代表中间状态不存在，只是中间状态不以 Prompt response token 的形式交给 TUI。

### 4.2 事件通道负责运行中的可见性

#### 4.2.1 领域更新先经过 Event 与兼容 Bridge

运行中的状态沿以下链路到达当前 TUI：

```text
SessionProcessor / Session Service
-> EventV2
-> EventV2Bridge
-> compatibility GlobalEvent
-> GlobalBus
-> Worker RPC event 或 /global/event SSE
-> SDKProvider
-> TUI reducer / Reactive Store
```

#### 4.2.2 Worker RPC 与 SSE 只是两种 live transport

默认本地 Worker 订阅进程内 `GlobalBus`，然后用 `Rpc.emit("global.event", event)` 转给 TUI。监听与 `attach` 则由 SDKProvider 连接 `/global/event` SSE。

于是实际时间关系是：

```text
Prompt POST 仍在等待完整 Loop
        +
Event Channel 持续发送 Message / Part / status
        =
TUI 已显示运行过程
```

### 4.3 Event、Bridge、Transport 与 Store 各有一层职责

这条链不应压缩成一句“Server 用 SSE 发事件”：

- **EventV2 / Projector**：把 durable event 与 projection 放进事务，在提交后通知观察者；live-only event 则只通知当前观察者。
- **EventV2Bridge**：把共享 Event 转成旧客户端能理解的 compatibility payload；durable event 还会额外生成 `sync` envelope。
- **GlobalBus**：当前 executable 内的进程级发布总线。
- **Worker RPC 或 SSE**：当前拓扑使用的 live transport。
- **SDKProvider / TUI reducer**：批处理事件，把 Message、Part 和 delta 合并进响应式 Store。

它们分别解决状态定义、兼容转换、进程内分发、跨边界传输和 UI 投影问题。缺少其中任何一层都不能由另一层自动替代。

### 4.4 whole update、live delta 与 durable `sync` 不是同一件事

#### 4.4.1 同一段输出可以同时产生 whole、delta 与 `sync`

当前兼容 Message/Part whole update 可以是 durable；`message.part.delta` 是 live-only。`EventV2Bridge` 对 durable event 额外发出带 sequence、aggregateID 与版本的 `sync` envelope：

```ts
if (event.durable === undefined) return
GlobalBus.emit("event", {
  payload: {
    type: "sync",
    syncEvent: {
      seq: event.durable.seq,
      aggregateID: event.durable.aggregateID,
      data: event.data,
    },
  },
})
```

#### 4.4.2 当前 TUI 使用 live compatibility event 加 snapshot hydration

但当前 TUI 的 `useEvent` 明确忽略 `payload.type === "sync"`。它消费普通 compatibility event，并通过 GET hydration 读取 Session、Message、Todo、Diff 等 snapshot。

所以当前 TUI 的恢复模型是“live event + snapshot hydration”，不是“按 durable event cursor replay”。名称 `SyncProvider` 也不能作为 replay 已接线的证据。

### 4.5 Durable 与 live transport 是两个维度

判断一项数据的恢复能力，应分别问：

```text
它是否 durable？       决定能否从存储重读
当前订阅是否能 replay？ 决定断线后能否从 cursor 补事件
它是否只有 live delta？ 决定重连后是否存在不可恢复后缀
```

一个 durable event 可以通过 volatile stream 实时送达；一个 live delta 可以被当前连接看见却永不落盘。不能把“事件实时到达”与“状态可恢复”合并成一个判断。

## 五、Provider 与 Tool：调用意图和真实副作用位于不同边界

### 5.1 当前默认 Provider Runtime 使用 AI SDK

兼容 `SessionPrompt` 通过 `LLM.run` 准备 Model、Messages、Tools、headers 和 Provider options。默认路径调用 AI SDK 的 Provider adapter，并把原始 stream 转换成统一 `LLMEvent`：

```text
SessionPrompt / SessionProcessor
-> LLM.run
-> AI SDK Provider Adapter
-> Provider transport
-> Model Provider
```

Provider request 可能离开当前机器，也可能连接本地服务。Runtime 代码只定义调用边界，具体网络位置要看 Provider 配置。

### 5.2 普通本地 Tool 在 OpenCode 一侧执行

Model 返回 `read(path=...)` 时，Provider 只输出一个 Tool Call。OpenCode 的 Tool Runtime 随后完成：参数 schema 校验、Permission、实际 I/O、Tool Result 持久化和 continuation。

因此文件读写、Shell 子进程和普通本地 Tool 的副作用发生在运行 OpenCode Server/Harness 的环境中。远程 `attach` 时，这意味着 Tool 通常作用于远端 Server 所在环境，而不是本地 TUI 所在机器。

### 5.3 `providerExecuted` hosted tool 是明确例外

某些 hosted tool 由 Provider 执行，并带有 `providerExecuted` 标记。OpenCode 会识别它们，避免再走普通本地同名 dispatch。

解释“Tool 在哪执行”时应先区分普通 local tool 与 hosted tool，不能只从工具名称、Provider 返回了 Tool Call，或界面显示 Tool Part 来猜测。

## 六、native LLM adapter 不等于 native V2 Session Runtime

### 6.1 实验开关只替换 Provider 适配层

#### 6.1.1 支持时走 native adapter，不支持时回退 AI SDK

`OPENCODE_EXPERIMENTAL_NATIVE_LLM` 对应 `experimentalNativeLlm`。旧 `LLM.run` 在这一层尝试 `LLMNativeRuntime.stream(...)`：

```ts
if (flags.experimentalNativeLlm) {
  const native = LLMNativeRuntime.stream({...})
  if (native.type === "supported") {
    return { type: "native", stream: native.stream }
  }
  // 不支持时继续使用 AI SDK 路径
}
```

#### 6.1.2 两种 Adapter 都回到旧 `SessionProcessor`

Native Adapter 与 AI SDK 最终都输出统一 `LLMEvent`，交给同一个旧 `SessionProcessor`。因此开启开关后的链路仍是：

```text
SessionPrompt old Loop
-> Native LLM Adapter（支持时）
-> Provider
```

### 6.2 判断 Session Runtime 必须追更外层调用点

“native”在这里描述 Provider request/stream adapter，不描述输入 admission、Session 调度或事件合同。只要外层仍由 `SessionHttpApi.prompt -> SessionPrompt.prompt -> loop` 驱动，就还是当前兼容 Session Runtime。

是否进入 native V2，必须继续追踪 URL、Handler 和 Core：`/api/session/:id/prompt -> native SessionHandler -> V2Session.prompt`。一个底层适配器名称不能替代完整调用链证据。

## 七、current 与 native V2 在同一个 Router 中共存

### 7.1 共存方式是路由 Layer 合并，不是两个 Server 代理

#### 7.1.1 compatibility 与 native Route 位于同一 Layer tree

当前 executable 的 `createRoutes()` 合并兼容与 native 路由：

```ts
return Layer.mergeAll(
  rootApiRoutes,
  eventApiRoutes,
  instanceRoutes,
  serverRoutes,
  docRoute,
  uiRoute,
)
```

其中 `instanceRoutes` 包含兼容 `/session/*`，`serverRoutes` 来自 native Protocol/Server 的 `/api/*`。它们位于同一 Effect Router layer tree，不是旧 Server 通过 HTTP 代理到另一个新 Server。

所以同一进程能同时响应两条路由：

```text
POST /session/:id/message
POST /api/session/:id/prompt
```

#### 7.1.2 Route 可达不代表当前 TUI 已选择它

路由可达只证明实现已接线；当前 TUI 到底使用哪条，仍要查看客户端调用点。

### 7.2 两套 Prompt 合同不能按名字互换

| 边界 | 当前默认兼容 Runtime | native V2 Runtime |
| --- | --- | --- |
| Client | `client.session.prompt(...)` | `client.v2.session.prompt(...)` 或 native Client |
| HTTP | `POST /session/:id/message` | `POST /api/session/:id/prompt` |
| Handler | `SessionHttpApi.prompt` | native `SessionHandler` |
| Core | `SessionPrompt.prompt -> loop` | `V2Session.prompt -> SessionInput.admit -> optional wake` |
| POST 成功值 | final Assistant `WithParts` | durable `SessionInput.Admitted` receipt |
| 是否等待 Provider 完成 | 是 | 否 |
| 当前普通 TUI | 已接线 | 未接线 |

两个 POST 都不是 token stream，但等待语义相反：兼容 POST 等整个旧 Loop，native POST 只等输入 durable admission 与 wake 调度，不等待 Runner 完成。

## 八、native V2：从最终响应合同演进为 admission 与 execution 分离

### 8.1 Prompt 先 durable admit，再选择是否 wake

#### 8.1.1 POST 首先提交一条 durable input

native `V2Session.prompt` 的关键顺序是：

```ts
const admitted = yield* SessionInput.admit(db, events, {
  id: messageID,
  sessionID: input.sessionID,
  prompt,
  delivery,
})

if (input.resume !== false) yield* execution.wake(admitted.sessionID)
return admitted
```

#### 8.1.2 `resume` 决定是否唤醒执行，不改变 admission 事实

它把“Server 已可靠接纳输入”和“Runner 已完成 Provider 工作”拆开。`resume:false` 可以只记录输入，不立即运行；默认 resume 则向执行层发出 wake。HTTP Handler返回 admission receipt，而不是最终 Assistant。

### 8.2 process-local Coordinator 管理同 Session 串行化

#### 8.2.1 `run`、`wake` 与 Session key 决定进程内并发关系

`SessionExecutionLocal` 使用全局 `SessionRunCoordinator`：

- 同一 Session 的 `run` 在活跃时 join 当前执行；
- `wake` 在活跃时记录一个 coalesced follow-up；
- 不同 Session 使用不同 key，可以并行；
- `interrupt` 只中断当前进程拥有的 active fiber；
- Runner 开始 drain 时按 Session Location 获取相应服务环境。

#### 8.2.2 进程内协调器不是跨进程执行所有权协议

这是一套进程内所有权模型，不是集群调度器。固定基线没有跨进程 fencing、租约或 stale owner 接管协议。

### 8.3 Runner 显式划分 Provider Turn 与 Tool settlement

#### 8.3.1 每个 Turn 明确经过调用、结算与历史重载

native Runner 每个 Turn 显式调用一次 `llm.stream(request)`。本地 Tool Call 先记录 durable 调用事实，再执行 Tool settlement；需要 continuation 时，下一 Turn 重读投影后的 Session History。

这使阶段边界更清楚：

```text
admitted input
-> promotion 成 User Message
-> Provider Turn
-> durable Tool Call
-> Tool execution / settlement
-> history reload
-> continuation 或结束
```

#### 8.3.2 独立 Runner 不代表旧能力已经全部迁移

这条路径不桥接旧 `SessionPrompt.loop`，Provider 也直接使用 native `LLMClient` 路由。但“拥有独立 Runner”不代表旧 Runtime 的每项能力已经迁移。

### 8.4 native Event API 区分 volatile live 与 durable replay

#### 8.4.1 全 Server Event 是 volatile live SSE

`GET /api/event` 订阅全 Server 的实时事件。Handler 先建立容量有限的 bounded live stream，再发送 `server.connected`；它没有接受历史 cursor，也不会为新连接重放断线前的事件。这个接口解决“从现在开始观察”，不解决按 Session 恢复历史。

#### 8.4.2 Session Event 与 History 读取 durable sequence

`GET /api/session/:id/event?after=N` 先重放指定 aggregate sequence 之后的 Session durable events，再继续 tail 新提交的 durable events。`GET /api/session/:id/history?after=N&limit=M` 则只读取有限的一页 durable history，并返回是否还有后续页。前者是长连接 replay-and-tail，后者是有限分页读取，不能因为都使用 `after` 就把它们看成同一合同。

#### 8.4.3 Cursor 不补 live-only delta，当前 TUI 也未消费这些 API

per-session cursor 只重放 durable event，不补回 live-only text、reasoning 或 tool-input delta。当前普通 TUI 也未消费这些 native API；它仍使用兼容 `/global/event` 与 GET hydration。

## 九、演进价值与当前 parity 边界

### 9.1 native V2 已经是可达的 Runtime slice

固定源码中，native Protocol、Handler、`V2Session.prompt`、durable admission、process-local execution、Runner、Tool settlement 和 Session cursor API 都已接线并有测试。它不是只有目录、类型或规格计划的空壳。

Admission 与 execution 分离带来几项清楚的工程边界：输入可精确重试，`resume:false` 可只接纳，steer/queue 可以在安全点 promotion，客户端可以用 durable Session sequence 续接历史。

### 9.2 仍未达到当前兼容 Runtime 的完整能力面

固定基线仍需保留这些限制：

- 当前 TUI 的 Prompt、Global Event 和 hydration 合同没有迁移到 native；
- Provider、Context、Tool 和 Plugin 覆盖仍有 partial 部分；
- Task/Subagent 父子 Session 编排尚未完整迁移；
- 一般 Provider Retry、Doom Loop 与旧 Runtime 的等价保护未完成；
- native `compact` 与独立 `wait` 合同存在，但 Core 操作当前不可用；
- execution ownership 仍是 process-local；
- clustered execution、stale-owner fencing 与自动 post-crash continuation 尚未实现。

因此最准确的定位是：**native V2 已接线、可达、可测试，但仍是未完成全部 parity 的独立 Runtime；当前普通 TUI 主线继续使用兼容 Runtime。**

## 十、失败与恢复：五种相似现象不能混写

### 10.1 Prompt 请求失败不等于任务事务回滚

SDK validation、网络或 Handler 错误会让 Prompt Promise reject，TUI 显示发送失败。但 User Message 可能已经 durable，Provider 可能已经产生部分输出，Tool 也可能已执行副作用。

Request failure 只说明 Client 没有取得这次请求的正常返回，不能推出整次任务“从未发生”。Agent 任务不是包裹 Provider 和外部 I/O 的全局数据库事务。

### 10.2 Event 断线不等于 Server Run 停止

事件连接负责观察，不拥有 Session Run。远程 SSE 断开后，Server 端 Provider Turn 或 Tool仍可能继续；TUI 重连也不应自动重发原 Prompt，否则可能重复产生副作用。

当前兼容 `/global/event` 没有 cursor replay。SDK会重连后续 live events，Session 页面可通过 GET hydration 重读 durable whole state，但断线窗口中的 live-only delta 后缀可能无法恢复；当前代码也不会仅因每次 `server.connected` 自动执行完整 Session hydration。

### 10.3 Hydration 不等于 Event replay

Hydration 读取“现在的 snapshot”，Event replay 按 sequence 重放“期间发生过的 durable facts”。前者能恢复最终 whole Message/Part，却不一定保留事件顺序；后者能从 cursor 续接 durable 序列，却仍不包含未落盘的 live delta。

当前 TUI 主要依赖前者。native per-session event API 提供后者，但尚未接到当前 TUI。

### 10.4 Interrupt 不等于副作用回滚或跨进程取消

兼容路径的 interrupt 进入 `SessionPrompt.cancel -> SessionRunState.cancel`，Processor 尽力保存已有 Text/Reasoning，并把未完成 Tool标成 interrupted error。native 路径中断当前进程 Coordinator拥有的 active Runner，idle interrupt 是 no-op。

两者都不能撤销已经执行的文件写入、命令或外部 API；native interrupt 也没有跨进程所有权协议。取消是一种停止后续工作的尽力机制，不是补偿事务。

### 10.5 Durable admission 与 cursor 不等于自动崩溃续跑

进程崩溃时可能处于多个歧义点：Provider 已收到请求但响应未提交，Tool 已产生副作用但 settlement 未完成，Runner owner 已消失但新进程不知道是否安全重试。

Durable admission 证明输入已接纳，durable cursor 证明客户端能从某个 sequence 继续读；它们没有自动解决外部副作用幂等、执行所有权和 Provider 请求重放。固定基线明确没有安全的 startup automatic continuation policy。

因此恢复能力应拆成五个问题：

```text
请求能否重试？
事件能否重连？
状态能否 hydration？
durable event 能否 cursor replay？
执行能否在崩溃后安全续跑？
```

回答其中一个，不能替代其余四个。

## 十一、关键源码索引

正文保留的是机制所需的关键代码。继续核对时可从以下入口进入；完整证据、测试与状态边界见 [源码与证据索引](./appendices/Source_Index.md)。

| 要回答的问题 | 关键入口 |
| --- | --- |
| 默认 TUI 怎样选择 Worker RPC 或 listener | `packages/opencode/src/cli/cmd/tui.ts`：`TuiThreadCommand`、`createWorkerFetch`、`createEventSource` |
| Worker 如何调用 Router、转发事件与启动 listener | `packages/opencode/src/cli/tui/worker.ts`：`rpc.fetch`、`rpc.server`、Global Event forwarding |
| 远程 attach 为什么使用普通 HTTP/SSE | `packages/opencode/src/cli/cmd/attach.ts`、`packages/tui/src/context/sdk.tsx` |
| 普通 TUI 调用了哪套 Prompt API | `packages/tui/src/component/prompt/index.tsx`：`submitInner` |
| compatibility Prompt 如何等待最终结果 | `packages/opencode/src/server/routes/instance/httpapi/handlers/session.ts`：`SessionHttpApi.prompt` |
| 当前 Agent Loop 在哪里 | `packages/opencode/src/session/prompt.ts`、`packages/opencode/src/session/processor.ts` |
| Provider adapter 如何选择 | `packages/opencode/src/session/llm.ts`、`packages/opencode/src/session/llm/native-runtime.ts` |
| 普通本地 Tool 怎样解析和执行 | `packages/opencode/src/session/tools.ts` |
| compatibility event 与 `sync` 如何桥接 | `packages/opencode/src/event-v2-bridge.ts`、`packages/tui/src/context/event.ts` |
| 新旧路由怎样合并 | `packages/opencode/src/server/routes/instance/httpapi/server.ts`：`createRoutes` |
| native Prompt 如何 admission | `packages/server/src/handlers/session.ts`、`packages/core/src/session.ts`、`packages/core/src/session/input.ts` |
| native process-local 调度在哪里 | `packages/core/src/session/execution/local.ts`、`packages/core/src/session/run-coordinator.ts` |
| native Runner 与 Tool settlement 在哪里 | `packages/core/src/session/runner/llm.ts` |
| native live、replay 与 history 合同 | `packages/protocol/src/groups/event.ts`、`packages/protocol/src/groups/session.ts`、`packages/server/src/handlers/session.ts` |

## 十二、总结：边界必须沿完整调用链判断

Runtime Boundary 不能靠一个模块名、一个 URL 或一个 `native` 开关判断。逻辑角色说明谁负责什么，Worker 与进程边界说明代码在哪个执行上下文，网络边界说明数据是否真正跨 socket；三者需要分别核对。

当前默认本地 TUI 把 Prompt 通过 Worker RPC 送入同一 executable 的兼容 Router，进入 `SessionHttpApi.prompt -> SessionPrompt`；普通 Tool 在 OpenCode 一侧执行，Provider 通过自己的 transport 连接模型；Prompt POST 等待最终 Assistant，而运行过程经 EventV2、Bridge、GlobalBus 和独立 RPC event 更新 TUI。监听模式和远程 `attach` 只替换 Client/Server transport，不自动改变 Session Runtime。

`OPENCODE_EXPERIMENTAL_NATIVE_LLM` 只替换旧 Loop 内的 Provider adapter。真正的 native V2 使用另一套 `/api/session/:id/prompt` 合同，把 durable admission 与 process-local execution 分离，并提供独立 live 与 durable cursor API；它已经可达，却尚未完成当前 TUI 与全部 V1 parity。请求失败、事件断线、hydration、interrupt 和崩溃恢复也各自属于不同边界，不能用“状态已持久化”一笔带过。

至此，06-12 的 Harness 主线可以合成一张完整地图：Client提交目标，Harness选择 Agent并组织 Context与 Loop，Model提出下一步，Tool Runtime执行被允许的行动，Session/Event保存和传播状态，必要时父子 Session分工，而 Runtime Boundary决定这些责任如何跨模块、Worker、网络和恢复合同连接起来。

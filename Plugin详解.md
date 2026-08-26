# OpenCode 与 DeepSeek Harness Plugin 架构详解：从 Hook 扩展到“一切皆插件”

OpenCode 和 DeepSeek Harness（下文简称 DSH）都支持 Plugin，但两者所说的 Plugin 并不处在完全相同的架构层次。

OpenCode 先实现一套完整的 Harness 主流程，再让 Plugin 在预先定义的位置注册 Hook，或贡献 Tool、Provider 等能力；DSH 则以 Cordis 为运行内核，把 Agent Loop、Tool Runtime、Session 等 Harness 组成部分本身也组织成 Plugin。



## 1. Hook、Event 与 Plugin

### 1.1 Hook：宿主流程中的介入点

Hook 可以直译为“钩子”。更准确地说，它是宿主程序在某个确定运行阶段主动开放的介入点：宿主执行到这里时，调用外部注册的处理函数，再根据处理结果继续、修改、拒绝或替换原流程。

```text
宿主原本的运行流程
        ↓
到达预先定义的 Hook
        ↓
调用已注册的处理函数
        ↓
观察 / 修改 / 拒绝 / 包裹原操作
        ↓
宿主继续后续流程
```

这里存在一次重要的控制权反转：不是扩展代码决定什么时候闯入宿主，而是宿主在自己控制的时机调用扩展代码。宿主负责规定：

- Hook 的名称；
- 输入和输出；
- 调用顺序；
- 是否允许修改数据；
- 如何拒绝或截断流程；
- 处理函数失败后怎样恢复。

以工具执行为例，原始流程可能是：

```text
收到模型生成的 Tool Call
→ 校验工具参数
→ 执行工具
→ 保存 Tool Result
```

加入 Hook 后变成：

```text
收到模型生成的 Tool Call
→ 校验工具参数
→ 触发 before Hook
→ 执行工具
→ 触发 after Hook
→ 保存 Tool Result
```

扩展代码由此可以在 `before` 阶段检查命令、修改参数或拒绝危险操作，也可以在 `after` 阶段记录耗时、清洗结果或发送通知，而不必把这些逻辑写进每个工具的实现。

按照介入方式，Hook 大致可以分为：

- 观察型：读取当前状态，不改变主流程；
- 变换型：修改参数、消息或结果；
- 决策型：允许、询问或拒绝操作；
- 环绕型：在原操作前后执行，也可以截断或替换原操作。

本文中的 Hook 特指 Agent Harness 的运行时扩展点，不是 Git Hook，也不是 React Hook。它们共享“挂入既有生命周期”的思想，但宿主、接口和执行语义不同。

### 1.2 Event：组件之间的消息与派发机制

Event 表示一个组件向外部发布某种状态变化或运行阶段。发布者只负责描述发生了什么或当前进入了哪个阶段，监听者通过事件名注册处理函数，双方不需要直接依赖彼此。

```text
Publisher 发布 Event
        ↓
Event Dispatcher 查找监听者
        ↓
按约定的模式调用 Listener
        ↓
收集结果或继续后续流程
```

Event 一般包含三部分：

- 名称：例如 `session.idle`、`tools/pre-execute`；
- 数据：监听者能够读取的上下文；是否允许修改由具体 Event 契约决定；
- 派发语义：顺序、并行、遇到结果停止，或者通过 `next()` 组成调用链。


| 概念 | 关注点 | 典型问题 |
| --- | --- | --- |
| Hook | 控制流中的扩展位置 | 外部逻辑可以在什么时候介入？ |
| Event | 组件之间的消息与派发 | 信息怎样发布，监听者怎样被调用？ |


> Hook 描述“允许介入的运行位置”，Event 描述“组件之间的派发机制”；Event 可以成为 Hook 的实现方式。

### 1.3 Plugin：具有生命周期的扩展单元

Plugin 不是一个运行时机，而是一份由宿主发现、加载和管理的扩展代码。它通常包含以下一种或多种能力：

- Hook 或 Event Listener；
- 自定义 Tool；
- Provider、Storage、Sandbox 等 Service 实现；
- 配置 Schema；
- 初始化和清理逻辑；
- 对其他 Plugin 或宿主能力的依赖声明。

一个 Plugin 可以注册多个 Hook；一个 Hook 也可以有多个 Plugin 监听。Plugin 还可能只注册一个 Tool 或提供一个 Service，而不监听任何运行事件。

Plugin 通常具有完整生命周期：

```text
发现
→ 加载模块
→ 初始化
→ 注册能力
→ 运行期间响应调用
→ 卸载并清理资源
```

三个概念可以收束为：

```text
Hook：在哪里、什么时候可以介入
Event：组件之间怎样派发信息和控制
Plugin：谁提供扩展能力，以及这些能力如何被加载和管理
```

接下来要比较的重点，不是 OpenCode 和 DSH 哪一个“有 Plugin”，而是 Plugin 分别通过什么机制参与 Harness，以及它能参与到多深的架构层次。

## 2. OpenCode 的 Plugin

### 2.1 Plugin 的入口：返回一个 Hooks 对象

OpenCode 的公开类型把 Plugin 定义为异步函数：它接收 `PluginInput` 和可选配置，返回一个 `Hooks` 对象。

```ts
type Plugin = (
  input: PluginInput,
  options?: PluginOptions,
) => Promise<Hooks>
```

`PluginInput` 提供当前项目和运行环境，例如：

- `client`：调用 OpenCode Server API 的 SDK Client；
- `project`：当前项目信息；
- `directory`：当前工作目录；
- `worktree`：Git 工作树路径；
- `serverUrl`：当前 OpenCode Server 地址；
- `$`：Bun Shell API；
- `experimental_workspace.register(...)`：注册 Workspace Adapter 的实验入口。

`Hooks` 描述这个 Plugin 要向宿主贡献哪些能力。一个最小 Plugin 可以写成：

```ts
import type { Plugin } from "@opencode-ai/plugin"

export const AuditPlugin: Plugin = async ({ directory }) => {
  return {
    "tool.execute.before": async (input, output) => {
      console.log("before", directory, input.tool, output.args)
    },
    "tool.execute.after": async (input, output) => {
      console.log("after", input.tool, output.title)
    },
  }
}
```

这段代码只声明 Plugin 导出的形状。它不会自行轮询 Tool Call；真正触发这些处理函数的是 OpenCode Harness 内部的调用位置。

### 2.2 OpenCode Plugin 可以贡献什么

`Hooks` 接口包含的能力大致可以分成三类。

第一类是运行阶段 Hook：

- `event`：接收 OpenCode Event；
- `chat.message`、`chat.params`、`chat.headers`：介入消息和模型请求；
- `permission.ask`：介入权限决定；
- `tool.execute.before`、`tool.execute.after`：介入工具执行；
- `shell.env`：补充 Shell 环境变量；
- System Prompt、消息变换和 Compaction 等实验 Hook。

第二类是能力注册入口：

- `tool`：注册自定义 Tool；
- `auth`、`provider`：扩展认证或模型 Provider。

第三类是生命周期入口：

- `config`：Plugin 加载后接收当前配置；
- `dispose`：Plugin 运行环境释放时清理资源。

因此，OpenCode Plugin 并不等于某一个 Hook。它以 `Hooks` 对象为统一入口，同时贡献运行时处理函数、注册能力和生命周期逻辑。

### 2.3 Plugin 如何被发现和加载

OpenCode 可以从配置中的 `plugin` 列表加载 npm 或文件 Plugin，也会扫描配置目录下 `plugin/`、`plugins/` 中的 `.ts` 和 `.js` 文件。主要链路可以简化为：

```text
读取全局、项目和其他配置来源
→ 收集 Plugin Spec 及来源
→ 解析文件路径或安装 npm 包
→ 检查入口和版本兼容性
→ dynamic import Plugin 模块
→ 调用 Plugin(input, options)
→ 收集返回的 Hooks 对象
→ 调用每个 config Hook
→ 进入正常运行
```

加载过程有三个值得注意的特征。

第一，外部候选 Plugin 可以并行解析和导入，但成功结果仍保持配置顺序；Plugin 函数随后按顺序调用并加入 `hooks` 数组，使注册和执行顺序可预测。

第二，OpenCode 会按 Plugin 身份去重。同一个 npm 包在多个配置来源重复出现时，最终合并胜出的来源会被保留；文件 Plugin 则按准确的文件 URL 区分。这里的“后者胜出”发生在配置合并阶段，不表示运行时 Hook 会反向执行。

第三，OpenCode 还会加载部分内部 Plugin，例如 Provider 认证集成。除非运行参数禁用默认 Plugin，它们先进入 `hooks` 数组，外部配置 Plugin 随后加入。

### 2.4 Hook 如何被触发

OpenCode 的通用 Trigger 逻辑可以概括为：

```ts
for (const hook of state.hooks) {
  const fn = hook[name]
  if (!fn) continue
  await fn(input, output)
}
return output
```

也就是：

```text
取得 Hooks[]
→ 按数组顺序查找同名 Hook
→ await hook(input, output)
→ 把同一个 output 交给下一个 Hook
→ 返回最终 output
```

这种调用方式具有以下性质：

- 同名 Hook 串行执行，而不是并行执行；
- 多个 Hook 共享同一个 `output` 对象，可以累积修改；
- 通用 Hook API 不向每个处理函数提供 `next()`；
- 处理函数抛错时，`Plugin.trigger()` 本身不会自动把错误转换成“继续执行”；
- 最终怎样恢复或呈现错误，取决于 Trigger 所在的外层流程。

并非 `Hooks` 中所有字段都使用这条二参数 Trigger。`config` 在初始化后单独调用，`event` 由 Event Bridge 转发，`dispose` 在运行状态释放时调用；`tool`、`auth` 和 `provider` 则由各自的注册或消费逻辑读取。

### 2.5 Plugin 在 Tool 执行链中的位置

以 Tool 为例，OpenCode 在真正调用 Tool Executor 前触发 `tool.execute.before`，完成后触发 `tool.execute.after`：

```text
模型返回 Tool Call
→ OpenCode 解析和验证参数
→ Plugin.trigger("tool.execute.before", ..., { args })
→ Tool.execute(args, context)
→ Plugin.trigger("tool.execute.after", ..., result)
→ Tool Result 进入 Session
```

`before` Hook 可以原地修改 `output.args`。如果 Plugin 要阻止执行，可以抛出错误，但这是一种异常式拒绝；这个 Hook 类型没有定义类似 `{ kind: "allow" | "deny" | "ask" }` 的正常决策返回值。

OpenCode Plugin 能够深入影响 Harness，例如修改模型参数、Header、System Prompt 和消息，介入 Tool 执行，注册 Tool，扩展 Provider 认证，以及接收 Session、Message、Permission 等 Event。

但公开 `Hooks` 接口没有把整个 OpenCode Agent Loop、Session Store 和 Tool Registry 都声明成可通过同一种 Plugin 配置整体替换的 Service。普通 Plugin 面对的是宿主预先定义的扩展面：

```text
OpenCode 固定 Harness 主干
├── Plugin Hook：模型请求前
├── Plugin Hook：Tool 执行前后
├── Plugin Hook：Permission
├── Plugin Hook：Session / Message Event
└── Plugin 注册：Tool、Auth、Provider 等
```

这里的“固定”不是说 OpenCode 源码不能修改，而是说普通 Plugin 使用者只能在公开 Hook 和注册入口上扩展。如果要替换 Agent Loop 本身，通常需要修改或替换 OpenCode 的宿主实现，而不是仅返回另一个 `Hooks` 对象。

OpenCode Plugin 的架构定位可以概括为：

> 向一个已经成立的 Harness 注册 Hook，并通过宿主规定的入口贡献 Tool、Auth、Provider 等能力。

## 3. DSH 的 Plugin

### 3.1 Cordis：DSH Plugin 架构的运行内核

DSH 底层使用 Cordis。Cordis 不是某一个业务 Plugin，而是承载整个 Plugin 体系的运行内核：它负责创建 Context、挂载和卸载 Plugin、管理 Service 依赖、派发类型化 Event，并追踪可撤销的 Effect。

可以把 Cordis 架构理解成下面几部分：

```text
                        Cordis Runtime
                              │
              ┌───────────────┼───────────────┐
              │               │               │
           Context         Registry         Event System
        Service 容器      Plugin/Fiber 管理   Event 派发
              │               │               │
              └───────────────┼───────────────┘
                              │
                            Effect
                     注册与资源清理的归属
                              │
                        DSH Plugin Tree
          LLM / Tools / Session / Agent Loop / UI / Storage
```

这些概念分别承担不同职责：

- Context：保存 Service，并为 Plugin 提供运行上下文；
- Registry：注册 Plugin，并为每次挂载创建和管理 Fiber；
- Service：通过稳定接口向其他 Plugin 提供可直接调用的能力；
- Event：让 Plugin 观察、决策、变换或包裹运行流程；
- Fiber：表示某个 Plugin 在某个 Context 下的一次运行实例；
- Effect：把 Listener、Service、Tool 和外部资源的清理动作归属到 Fiber；
- Loader：把配置解析成最终的 Plugin Tree 并交给 Cordis 挂载。

因此，“DSH 由 Plugin 组成”不代表 DSH 没有内核。Cordis 的 Context、Registry、Fiber、Event 和 Effect，以及 DSH 的启动与 Loader 机制，仍然是承载 Plugin 的基础设施。区别在于 LLM、Tools、Session、Agent Loop 等产品能力不被固化在这个内核中，而是作为 Plugin 挂载。

### 3.2 Cordis Plugin 的入口与依赖声明

一个 Cordis Plugin 可以采用三种主要入口形状：

- 函数：`(ctx, config) => ...`；
- 类：`new (ctx, config) => ...`；
- 对象：`{ apply(ctx, config) { ... } }`。

最常见的 DSH 函数 Plugin 类似：

```ts
import type { Context } from "@deepseek-ai/cordis"

export const name = "audit-plugin"
export const inject = ["tools"]

export function apply(ctx: Context): void {
  ctx.on("tools/pre-execute", async (exec, next) => {
    console.log("before", exec.name)
    return next()
  })
}
```

`inject = ["tools"]` 不是普通说明文字。它告诉 Cordis：这个 Plugin 依赖 `ctx.tools` Service，只有依赖可用时，Plugin 才能进入活动状态。

OpenCode Plugin 通常从统一 `PluginInput` 取得宿主已经准备好的能力；Cordis Plugin 则明确声明自己依赖哪些 Service，并由运行时根据依赖状态决定何时激活。

### 3.3 Context 与 Service

Cordis 的 `new Context()` 创建根依赖容器。Service 以稳定键名出现在 Context 上，例如：

- `ctx.tools`：Tool 注册与执行流水线；
- `ctx.llm`：LLM Adapter 和流式调用入口；
- `ctx.sessions`：Session Event Log 与内存存储；
- `ctx.systemPrompt`：Prompt Section 和 Tool Schema 组装；
- `ctx.agents`：Agent Registry；
- `ctx.agentLoop`：默认 Agent Loop 实现。

消费者依赖的是 `ctx.tools` 这类稳定接口，而不是直接导入某个具体 Tool Registry 实例。提供相同 Service 契约的不同 Plugin 因而可以在组合层被替换。

一个完整的可替换能力通常包含三部分：

```text
Service Definition：声明接口和 Context 键名
Service Provider：实现接口并注册到 Context
Consumer：通过 Context 使用接口
```

DSH 将这三者组成的替换边界称为 capability seam。例如在 LLM seam 中，`ctx.llm` 声明调用接口和 Adapter Registry，Provider Plugin 注册具体模型适配器，Agent Loop 则作为 Consumer 发起模型请求。

只写一个具体 Provider 并不会自动形成完整 seam。替换能够成立的前提是：有人定义稳定接口，Provider 遵守接口，Consumer 也只通过该接口工作。

### 3.4 Event：Plugin 之间怎样协作

Cordis 提供多种 Event 派发模式：

| 模式 | 核心语义 | 常见用途 |
| --- | --- | --- |
| `emit` | 按注册顺序通知，不等待监听器返回的 Promise | 已发生事实、轻量观察 |
| `parallel` | 并行运行并等待全部监听器 | 互不依赖的异步检查或刷盘 |
| `serial` | 按顺序等待，出现有效决策时停止 | 单一决策或顺序处理 |
| `waterfall` | Listener 通过 `next()` 包裹后续链路 | Tool、LLM 或请求的拦截与替换 |

Waterfall 最接近中间件：

```ts
ctx.on("tools/execute", async (exec, next) => {
  const started = Date.now()
  const result = await next()
  console.log(exec.name, Date.now() - started)
  return result
})
```

调用 `next()` 才会进入下一个 Listener，并最终到达内置执行逻辑；不调用 `next()` 就可以截断后续链路，直接返回拒绝结果或替代结果。

这和 OpenCode 通用 Trigger 的主要差异是：OpenCode 依次调用 Hook 并传递同一个 `output`，Cordis Waterfall 则把是否进入后续流程的控制显式交给当前 Listener。

Event 只是 DSH Plugin 的一种协作机制。Plugin 还可以通过 `ctx.tools.register()` 注册 Tool，通过 `ctx.llm.registerAdapter()` 注册模型适配器，通过 `ctx.systemPrompt.section()` 注册 Prompt Section，或者直接提供一个新的 Context Service。

### 3.5 Fiber 与 Effect：Plugin 的生命周期

调用 `ctx.plugin(plugin, config)` 时，Cordis 会创建一个 Fiber。Fiber 表示“这个 Plugin 在这个 Context 下的一次运行实例”，它追踪：

- Plugin 配置；
- `inject` 声明的 Service 依赖；
- 当前生命周期状态；
- Plugin 注册的 Event Listener、Service 和其他 Effect；
- 卸载时需要执行的清理函数。

Fiber 的主要状态可以概括为：

```text
PENDING：等待依赖 Service
→ LOADING：执行 Plugin 入口
→ ACTIVE：Plugin 已挂载并提供能力
→ UNLOADING：撤销注册并清理资源
→ DISPOSED：已经卸载

初始化或配置失败时 → FAILED
```

当依赖 Service 尚不存在时，Plugin 不会只因为文件顺序而立即失败，而是处于等待状态；Service 出现后再激活。依赖实现被替换或消失时，相关 Fiber 可以卸载，并在依赖恢复后重新运行。

Effect 负责让注册行为随 Plugin 一起撤销。`ctx.effect()` 执行设置逻辑，并记录它返回的 disposer；`ctx.on()`、Service 注册和 `ctx.tools.register()` 等操作也会把清理动作归属到当前 Fiber。

```text
Plugin 挂载
→ 注册 Service / Event / Tool
→ Cordis 将 disposer 归属到当前 Fiber

Plugin 卸载
→ 按反向顺序运行 disposer
→ 移除 Service / Event / Tool
→ 释放文件监听器、进程或其他资源
```

所以“注册是 Effect”不只是编码风格，而是热更新、配置替换和可靠卸载能够成立的基础。如果 Plugin 创建了未纳入 Effect 的全局计时器或进程，Cordis 也无法自动清理它。

### 3.6 从配置到 Plugin Tree

DSH 启动时不是把所有能力硬编码进一个巨型入口。简化后的装配过程是：

```text
创建根 Context
→ 挂载 Cordis Loader
→ 读取 Profile
→ 叠加 Profile 中的 Bundles
→ 应用 Profile 的 cordis.patch.yml
→ 应用 Harness Home 的 cordis.patch.yml
→ 应用命令行 --patch Overlay
→ Loader 导入并挂载配置行
→ 等待 Service 依赖满足、Plugin 激活
→ 审计未加载、PENDING 或 FAILED 的条目
→ 返回完整运行 Context
```

几个配置概念分别负责不同层次：

- Profile：一次运行采用的命名组合，例如 `web` 或 `headless`；
- Bundle：一组可以被上层 Patch 继续修改的配置行和 Plugin 包；
- `cordis.patch.yml`：按 `id` 替换完整配置行，或插入新行；
- Plugin Tree：各层配置最终组成并由 Loader 挂载的运行树。

配置行的书写顺序主要服务于阅读，不等于完整的依赖加载顺序。真正的激活还取决于每个 Plugin 的 `inject` 依赖是否满足。

### 3.7 “一切皆插件”的准确含义

DSH 的基础 Bundle 把下列能力作为独立配置行挂载：

```text
@deepseek-ai/dsh-llm
@deepseek-ai/dsh-session
@deepseek-ai/dsh-tools
@deepseek-ai/dsh-system-prompt
@deepseek-ai/dsh-agent-loop
@deepseek-ai/dsh-session-persistence-jsonl
@deepseek-ai/dsh-subprocess-local
@deepseek-ai/dsh-sandbox-local
@deepseek-ai/dsh-skill
以及 Provider、Tool、Compaction、Subagent、UI 等 Plugin
```

默认 Agent Loop 本身是 Plugin，Session Store 是 Plugin，Tool Runtime 也是 Plugin。上层 Patch 可以替换某条配置行，新的 Plugin 也可以提供相同的 Service 接口。

因此，“一切皆插件”主要包含三层含义：

1. 产品能力通过统一 Plugin 机制挂载，而不只把第三方小功能称为 Plugin；
2. 能力之间依赖稳定 Service 和类型化 Event，而不是直接绑定具体实现；
3. 运行时由配置组合，可以增加、移除、隔离或替换组成部分，而不必先修改一个中央 Agent 类。


更准确的表述是：

> DSH 没有一个不可替换的“产品能力核心”；产品能力被组织成 Cordis Plugin，而 Cordis 框架与启动装配机制仍是承载这些 Plugin 的内核。

DSH Plugin 的架构定位可以概括为：

> Plugin 既可以在 Event 上扩展已有流程，也可以提供组成 Harness 的 Service、Tool、Session、Agent Loop 或 UI。

## 4. OpenCode 与 DSH Plugin 的联系和区别

### 4.1 两种 Plugin 的共同点

OpenCode 和 DSH 的 Plugin 有一组共同基础：

- 都由宿主发现、加载和初始化；
- 都能监听或改变 Agent 的运行过程；
- 都能注册 Tool 等模型能力；
- 都需要处理初始化和资源清理；
- 都只能在宿主定义的接口与语义范围内工作；
- 都是进程内高信任代码，不天然具备安全隔离。

两者也都可以通过 Event 实现 Hook：宿主或 Service 在确定阶段派发 Event，Plugin 注册 Listener，并在该阶段观察、修改、拒绝或包裹原操作。

因此，区别不在于“OpenCode 有 Hook，而 DSH 有 Plugin”，也不在于“一个支持 Plugin，另一个不支持”。真正的区别是：

> Plugin 参与 Harness 架构的深度，以及宿主用什么机制管理依赖、控制流和生命周期。

### 4.2 核心区别

| 对比维度 | OpenCode Plugin | DSH / Cordis Plugin |
| --- | --- | --- |
| 基本入口 | 异步函数返回 `Hooks` 对象 | 函数、类或对象挂载到 Context |
| 主要产物 | Hook 处理函数和规定类型的注册能力 | 一个由 Fiber 管理的 Plugin 运行实例 |
| Harness 主体 | OpenCode 宿主实现主流程，在固定位置触发 Hook | Harness 的主要能力由 Plugin Tree 组成 |
| 能力获取 | 从统一 `PluginInput` 获取宿主上下文 | 从 Context 获取 Service，并用 `inject` 声明依赖 |
| 激活方式 | 按配置与加载顺序初始化 | Service 依赖满足后 Fiber 才激活 |
| Hook 执行 | 同名 Hook 串行执行，共享并修改 `output` | 支持 `emit`、`parallel`、`serial`、`waterfall` |
| 环绕与截断 | 通用 Hook API 没有 `next()`，通常修改数据或抛错 | Waterfall 用 `next()` 显式委托，不调用即可截断或替换 |
| 注册能力 | 通过 `tool`、`auth`、`provider` 等规定字段 | 可以提供 Context Service，也可注册 Tool、Prompt、Adapter 等 |
| Agent Loop | 不属于普通 `Hooks` 对象可整体替换的公开入口 | 默认 Agent Loop 本身就是可替换 Plugin |
| Session | Plugin 在指定位置监听或影响 Session | Session Service 和持久化本身也由 Plugin 提供 |
| 生命周期 | 初始化并提供可选 `dispose` | Fiber 跟踪状态，Effect 统一撤销注册和资源 |
| 组合方式 | 配置列表和 Plugin 目录 | Profile、Bundle、Patch 和 Plugin Tree |

最核心的结构差异是：

```text
OpenCode
固定 Harness 主干
└── 在公开扩展点加载 Plugin

DSH
Cordis 运行内核
└── 通过 Plugin Tree 组合 Harness 产品能力
```

### 4.3 两种设计的取舍

OpenCode 的固定扩展面具有以下特点：

- 公共接口较集中，Plugin 作者容易理解有哪些能力；
- Harness 主流程由宿主控制，运行结构更直接；
- Plugin 的影响范围更容易围绕固定 Hook 分析；
- 如果需求超出公开扩展面，普通 Plugin 很难替换宿主核心组件。

DSH 的全 Plugin 组合具有不同取向：

- Tool、Session、Agent Loop 等组件可以通过稳定 Service 重新组合；
- 适合 Harness 研究、能力替换和多运行模式；
- Fiber 与 Effect 为动态依赖和卸载提供统一模型；
- 同时需要处理更复杂的依赖、配置、生命周期和故障传播。

因此，不能简单把区别总结成“DSH Plugin 更强”。两套设计优化的是不同目标：OpenCode 强调在一个完整产品 Harness 上提供明确扩展面，DSH 强调通过 Cordis Plugin Tree 组合和替换 Harness 能力。

最终可以用两句话概括：

> OpenCode Plugin 主要是向一个已经成立的 Harness 注册 Hook 和规定能力。

> DSH Plugin 既可以注册 Hook，也可以成为组成 Harness 的 Service、Tool、Session、Agent Loop 或 UI。

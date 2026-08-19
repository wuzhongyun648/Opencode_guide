# Agent 专业化与协作：从一个模型到父子 Session

上一篇：[10 Session 与 Persistence](./10_Session_and_Persistence.md) ｜ 下一篇：[12 Runtime Boundary](./12_Runtime_Boundary.md)

> 固定源码：OpenCode `0e3474509aa5ad16afcf9c439785514d6443c6af`（`dev`，2026-08-18）
>
> 分析主线：当前默认 TUI 使用的兼容 Session Runtime。文末单独说明 native V2 的已有能力与迁移缺口。

假设一位刚开始学习 Harness 的读者，请 OpenCode 阅读教程入口和项目规则，再整理学习顺序。材料不多时，一个主 Agent 就可以完成：它读取文件、更新 Context，最后作答。

当材料扩大到多个模块，任务会出现不同性质的工作：有人负责确定范围，有人负责搜索源码，有人负责核对结论，主线还要持续记录进度。OpenCode 为此提供了 Agent、Plan、Todo、`task` 和 Subagent 等能力。它们经常同时出现在界面或源码里，却不在同一个抽象层：

```text
Model      提供生成文本和 Tool Call 的基础能力
Agent      规定怎样使用 Model：角色、指令、权限、参数
Plan       一种 primary Agent 工作轮廓，不是调度器
Todo       当前 Session 的结构化进度状态
task       父 Agent 发起委派的一次 Tool Call
Subagent   在独立子 Session 中运行的 Agent
```

本篇的中心不是“怎样尽可能多地启动 Agent”，而是理解 OpenCode 怎样先把一个 Model 专业化为不同工作角色，再在确有分工价值时，通过父子 Session 建立可控协作。

## 一、先把六个概念放回同一张架构图

### 1.1 能力、角色、状态与执行者分属四层

六个概念可以先按职责归入四层：

```text
┌───────────────────────────────────────────────────────────┐
│ 基础能力层                                                │
│ Model：根据 Context 生成 Text / Reasoning / Tool Call      │
├───────────────────────────────────────────────────────────┤
│ 工作角色层                                                │
│ Agent：prompt + model 偏好 + permission + 参数             │
│ ├─ build / plan：primary Agent                             │
│ └─ general / explore：subagent                             │
├───────────────────────────────────────────────────────────┤
│ Session 状态层                                            │
│ Todo：父或子 Session 各自保存的结构化清单                  │
├───────────────────────────────────────────────────────────┤
│ 委派执行层                                                │
│ 父 Agent --task Tool Call--> 子 Session --Subagent Loop--> │
│              <--------- task Tool Result ----------------- │
└───────────────────────────────────────────────────────────┘
```

这里既有包含关系，也有执行顺序。Agent 包含一份 Model 偏好，但 Agent 不是 Model；Todo 从属于 Session，却不控制 Agent Loop；`task` 是进入委派流程的工具，而 Subagent 才是子任务的执行者。

如果把它们都写成“六种 Agent 功能”，就会产生一连串错误推论：选择 Plan 不等于创建计划任务，Todo 出现不等于任务已调度，模型说“我会委派”不等于 `task` 已执行，子 Session 存在也不等于它运行在远程机器。

### 1.2 专业化解决什么问题

单 Agent 的优势是上下文连续、责任清楚、协调成本低。专业化与委派则主要解决三类问题：

- **行为边界不同**：规划角色需要限制编辑，调查角色只需读取和搜索，执行角色才需要更宽的能力。
- **上下文需要隔离**：大范围搜索会产生大量中间文件和 Tool Result，不一定都应进入父 Session 主线。
- **结果需要分层验收**：子 Agent 负责产出局部发现，父 Agent 仍负责核对、处理冲突并形成最终回答。

因此，多 Agent 不是自动提高质量的开关。每增加一层委派，都增加任务描述、权限派生、上下文传递、等待、失败处理和结果验收成本。正确原则是使用完成目标所需的最小充分结构。

## 二、Model 与 Agent：推理能力不等于工作角色

### 2.1 Model 只负责本轮生成

Model 接收 Harness 组装好的 Context，输出文本、推理片段或 Tool Call。它可以判断“应该读取哪份教程”，但不会因为输出了 `read(...)` 就自动访问文件系统，也不会自行保存 Session、校验路径或决定 Permission。

更换 Model，通常改变基础推理能力、速度、成本、上下文窗口和工具调用表现。至于它在当前任务中扮演什么角色、能看见哪些 Tool、操作是否允许，则由 Model 外部的 Agent 与 Harness 决定。

### 2.2 `Agent.Info` 是一份完整工作配置

#### 2.2.1 一份 Agent 配置同时覆盖身份、能力与生成参数

当前兼容 Runtime 的 `Agent.Info` 不是只有 `name` 和 system prompt。源码把角色用途、模型偏好、权限和运行参数放在同一配置中：

```ts
export const Info = Schema.Struct({
  name: Schema.String,
  description: Schema.optional(Schema.String),
  mode: Schema.Literals(["subagent", "primary", "all"]),
  native: Schema.optional(Schema.Boolean),
  hidden: Schema.optional(Schema.Boolean),
  color: Schema.optional(Schema.String),
  permission: PermissionV1.Ruleset,
  model: Schema.optional(
    Schema.Struct({ modelID: ModelV2.ID, providerID: ProviderV2.ID }),
  ),
  variant: Schema.optional(Schema.String),
  prompt: Schema.optional(Schema.String),
  steps: Schema.optional(Schema.Finite),
  temperature: Schema.optional(Schema.Finite),
  topP: Schema.optional(Schema.Finite),
  options: Schema.Record(Schema.String, Schema.Unknown),
})
```

#### 2.2.2 配置差异把同一 Model 塑造成不同角色

这份定义揭示了 Agent 的真实含义：

```text
Agent
= 身份与用途（name / description / mode / hidden）
+ 行为指令（prompt）
+ Model 偏好（model / variant）
+ 能力边界（permission）
+ 运行参数（steps / temperature / topP / options）
```

同一个 Model 可以同时服务 `build`、`plan` 和 `explore`，但它们因指令和权限不同而形成不同角色。反过来，一个 Agent 也可以通过配置选择不同 Model。Agent 因而既不是“Model 的副本”，也不只是“给 Model 起了一个人格名称”。

### 2.3 Agent 选择和 Model 选择是两次决定

当前 TUI 会把选中的 `agent`、`model` 和 `variant` 一起送入 Prompt。服务端创建 User Message 时，先按显式名称选择 Agent；未指定时才使用默认可见 primary Agent。Model 则按输入、Agent 偏好、Session 当前值、最近 User Message 和 Provider 默认值逐层解析。

委派时还会再做一次子 Agent 与子 Model 选择。由此可以看出：

```text
选择 Agent：决定以什么角色工作
选择 Model：决定这个角色使用哪种推理能力
```

两者可以相关，但不能合并成一个概念。

## 三、内置 Agent：primary、subagent 与 hidden 不是执行步骤

### 3.1 `build` 与 `plan` 是两种 primary 工作轮廓

`build` 和 `plan` 都是 primary Agent，表示它们可以直接承接当前 Session 的主任务。二者是可选择的工作模式，不是“所有任务必须先 Plan、再 Build”的固定流水线。

#### 3.1.1 `build`：默认执行型主 Agent

`build` 是默认可见的 primary Agent。其权限从全局默认规则开始，再允许 `question` 与 `plan_enter`，因此能够按实际 Permission 使用读取、编辑、Shell 等 Tool。

“执行型”不等于所有动作无条件放行。默认规则仍把 `.env`、外部目录和 doom loop 等敏感情况放进 ask 或更具体的规则中；用户配置也可以继续覆盖、收紧或扩展权限。

#### 3.1.2 `plan`：用权限和提醒形成规划边界

`plan` 同样是 primary Agent，但默认拒绝一般编辑，只为 plan 文件路径保留例外，并允许 `question` 与 `plan_exit`。它还默认拒绝 `task:general`。

因此 Plan 的工程含义不是“展示模型的隐藏思考”，也不是“创建一个以后自动执行的任务计划”。它是 Agent Permission、专门提醒文本和 Plan Tool 共同形成的工作轮廓：允许调查、澄清和组织方案，同时限制直接实施。

这些默认规则可被用户配置合并覆盖。准确说法应当是“固定基线下 Plan 默认限制一般编辑和 general 委派”，而不是“Plan 永远不能编辑或委派”。

### 3.2 `general` 与 `explore` 是可被委派的 Subagent

`general` 和 `explore` 的 `mode` 是 `subagent`。它们通常不是用户主 Session 的默认承接者，而是由 `task` Tool 选中：

- `general` 面向较通用的多步骤子任务，默认禁用 `todowrite`。
- `explore` 面向文件定位、关键词搜索和代码库调查，权限集合更接近只读探索，但仍需结合具体配置判断。

Subagent 这个标签说明“适合在子 Session 中使用”，并不说明它位于另一个进程、另一个主机或某种 A2A 网络。运行拓扑属于第 12 篇讨论的维度。

### 3.3 hidden Agent 服务内部流程

`compaction`、`title`、`summary` 等 Agent 被标记为 hidden，用于压缩、标题或摘要流程。它们说明 Agent Registry 不只是用户可切换的角色列表，也包含 Harness 内部需要的专用工作配置。

所以阅读 Agent 列表时应先看 `mode` 与 `hidden`，而不是把所有名称都理解为一套要依次运行的协作团队。

## 四、Plan、Todo、`task` 与 Subagent 各自负责什么

### 4.1 Plan 是 Agent 工作边界，不是任务调度器

选择 Plan Agent 后，主 Session 仍在运行正常 Agent Loop：Harness 组装 Context，Model 生成文本或 Tool Call，Permission 决定动作边界。Plan 没有独立的后台调度队列，也不会在方案写完后自动切到 Build 并执行。

因此，“形成计划”和“执行计划”是两个不同状态。Plan 可以帮助模型明确目标、范围、约束和完成条件，但是否切换角色、是否开始实施，仍要经过显式交互或相应 Tool/Session 状态变化。

### 4.2 Todo 是 Session 的结构化进度，不是 Loop 控制器

#### 4.2.1 `todowrite` 替换并持久化当前 Session 的清单

`todowrite` 接受一份完整数组，在 Permission 通过后替换当前 Session 的 Todo 表，并发布更新事件：

```ts
yield* ctx.ask({
  permission: "todowrite",
  patterns: ["*"],
  always: ["*"],
  metadata: {},
})

yield* todo.update({
  sessionID: ctx.sessionID,
  todos: params.todos,
})
```

`Todo.update` 会在事务中删除旧清单、按位置写入新清单，随后发布 `Todo.Updated`。这使 Todo 成为 Session 级、可持久化、可观察的进度状态。

#### 4.2.2 Todo 状态不会自动驱动 Agent Loop

它没有做三件事：不会启动 Provider Turn，不会自动执行 pending 项，也不会决定 Agent Loop 何时停止。即使 Todo 全部 completed，Loop 仍按 Assistant finish、Tool Part、错误和中断等条件结束；即使还有 pending 项，Harness 也不会仅凭它强制模型继续。

### 4.3 `task` 是委派门，Subagent 是门后的执行者

#### 4.3.1 `task` 先把委派变成受控 Tool Call

父模型判断某项独立调查值得拆分时，会生成 `task` Tool Call。参数中包含简短说明、给子 Agent 的任务内容、`subagent_type`，以及可选的 `task_id` 和实验性 `background`。

这首先仍是一项普通意义上的 Tool Call：需要参数校验、`task:<subagent_type>` Permission、深度检查和 Tool 生命周期结算。模型在文本中说“我已经让 explore 调查”不构成委派事实，只有 `task` Tool 真正进入执行并创建或恢复子 Session，协作才开始。

#### 4.3.2 Subagent 才在子 Session 中持续执行

Subagent 随后以指定 Agent 配置在子 Session 中运行自己的 Agent Loop。`task` Tool 等它形成结果，再把结果结算回父 Session。两者关系可以写成：

```text
task = 委派请求与生命周期外壳
Subagent = 子 Session 中真正持续执行工作的 Agent
```

## 五、一次前台委派的完整生命周期

### 5.1 父 Agent 先决定是否值得拆分

委派最适合边界清楚、能独立调查、结果可验收的子任务。例如主 Agent 正在组织 Harness 学习架构，而“只在指定源码范围内列出 Agent Registry 的关键入口”需要大量搜索，却不需要占用父 Session 的全部上下文。

一个可执行的子任务契约至少应说明：

- **目标**：这次调查要回答什么问题；
- **范围**：允许查看哪些目录、文件或材料；
- **约束**：只读、禁止外部网络、不得修改哪些对象；
- **产物**：返回清单、结论、证据还是差异说明；
- **完成条件**：父 Agent 依据什么判断覆盖充分。

这不是用户必须套用的 Prompt 模板，而是理解父子 Context 边界的关键：新子 Session 不会自动拥有父对话，父 Agent 必须把必要信息显式写进 `params.prompt`。

### 5.2 `task` 先检查实验开关、深度与 Permission

#### 5.2.1 后台开关和委派深度先限制能否进入子任务

`TaskTool.execute` 的前半段不是立即启动模型，而是建立控制边界：

```text
background 是否已启用
-> 沿 parentID 计算当前委派深度
-> 请求 task:<subagent_type> Permission
-> 查找目标 Agent
-> 创建或恢复子 Session
```

默认 `subagent_depth ?? 1`。根 Session 的 depth 是 0，可以创建一层子 Session；子 Agent 再尝试委派时 depth 已达到 1，因此默认被阻止。这个数限制的是父子 Session 层级，不是 Agent Loop 的 Provider Turn 数。

#### 5.2.2 Permission 按 Subagent 类型求值，再查找目标 Agent

Permission 检查使用 `subagent_type` 作为 pattern。这样 `plan` 可以默认拒绝 `task:general`，而用户规则又能针对具体 Subagent 类型进行覆盖。

授权通过后，Task Tool 才按 `subagent_type` 查询 Agent Registry；名称不存在会进入错误结算，而不会创建一个缺少明确角色配置的子 Session。

### 5.3 创建子 Session，而不是在父历史中换人格

#### 5.3.1 新委派获得独立 Session identity

没有可恢复的 `task_id` 时，Task Tool 创建新的 Session：

```ts
const nextSession = session ?? (yield* sessions.create({
  parentID: ctx.sessionID,
  title: params.description + ` (@${next.name} subagent)`,
  agent: next.name,
  permission: childPermission,
}))
```

这里的 `parentID` 建立结构关系，用于树形展示和深度计算。`Session.createNext` 仍然生成新的 Session ID、时间、Agent、Permission 和独立持久化记录。父 Tool 被取消时对子任务的协调由 Task Tool 保存的 Abort/Background Job 关系完成，不能把这种执行所有权归因于 `parentID` 字段本身。

#### 5.3.2 `parentID` 不会复制父 Session History

关键边界是：创建代码没有复制父 Session 的 Message/Part History。子 Agent 获得的新 User Message 来自父 `task` 参数经过 `resolvePromptParts(params.prompt)` 的结果。因此 `parentID` 是关联信息，不是“继承全部上下文”的开关。

#### 5.3.3 `task_id` 恢复路径不是强父子所有权句柄

传入 `task_id` 时，当前实现会尝试恢复已有 Session。固定基线的取回路径没有在该位置显式验证它是否属于当前父 Session；命中已有 Session 后，也不会用这次新派生的 child Permission 重写它。因而 `task_id` 表达“尝试继续已有执行上下文”，不应被描述成经过强父子所有权校验并重新收紧策略的安全句柄。

### 5.4 子 Agent 在自己的 Loop 中完成工作

Task Tool 解析子任务 Prompt 后，再调用同一套 Prompt 服务：

```ts
const result = yield* ops.prompt({
  messageID: MessageID.ascending(),
  sessionID: nextSession.id,
  model,
  variant: next.model ? undefined : variant,
  agent: next.name,
  parts,
})

return result.parts.findLast((item) => item.type === "text")?.text ?? ""
```

所以 Subagent 不是一次函数式“问答调用”。它拥有独立的 User Message、Assistant Message、Tool Call/Result、Permission 和 continuation，可以在子 Session 中多轮读取、搜索和整理。

### 5.5 结果压缩成父 Session 的 Tool Result

默认前台 `task` 会等待子任务完成。Task Tool 从子 Session 最后一个 Text Part 取得文本，包装为 `<task_result>`，并作为父 Assistant Message 中 `task` Tool Part 的 completed output。

父 `SessionPrompt` 随后 continuation，重新加载父历史，下一次 Provider Turn 才看到这份 Tool Result。子 Session 的完整搜索轨迹、全部 Tool Result 和中间消息不会整体拼进父 Session：

```text
子 Session 完整历史 ──保留在子 Session
子 Session 最终文本 ──task Tool Result──> 父 Session
```

这就是上下文隔离的实际来源，也说明父 Agent 为什么必须验收。返回的只是子 Agent 的压缩结论，不自动等于事实正确、范围完整或与主线一致。

## 六、父子 Session 究竟继承什么

### 6.1 Model：子配置优先，否则继承父 Assistant

子 Agent 如果配置了 `next.model`，Task Tool 优先使用该 Model。否则使用发出 `task` Call 的父 Assistant Message 中的 `providerID/modelID`：

```ts
const model = next.model ?? {
  modelID: msg.info.modelID,
  providerID: msg.info.providerID,
}
```

Variant 的规则更细：只有子 Model 继承自父 Assistant 时，才把父 variant 传给子 Prompt；子 Agent 显式指定 Model 时，Task Tool 把 variant 设为 `undefined`，交给子侧后续选择规则。

因此，“Subagent 一定与父 Agent 使用同一 Model”和“Subagent 一定使用专用 Model”都不准确。真实优先级是子 Agent 显式 Model 优先，缺省时才继承本次父 Assistant 的 Model。

### 6.2 Permission：派生硬边界，而不是复制父 Agent 角色

#### 6.2.1 子 Session 只从父 Session 派生需要贯穿的规则

父 Agent Permission、父 Session Permission 和子 Agent Permission 是不同来源。当前派生函数只从父 **Session** 继承 deny 与 `external_directory` 规则：

```ts
return [
  ...parentSessionPermission.filter(
    (rule) => rule.permission === "external_directory" || rule.action === "deny",
  ),
  ...(canTodo ? [] : [{ permission: "todowrite", pattern: "*", action: "deny" }]),
  ...(canTask ? [] : [{ permission: "task", pattern: "*", action: "deny" }]),
]
```

#### 6.2.2 有效 Permission 还要与 Subagent 自身规则合并

子 Session 运行时还会结合 Subagent 自身的 Agent Permission。于是：

- 父 Session 持久化 deny 和外部目录约束构成子 Session 的硬上限；
- Subagent 自身规则决定它在这个上限内有哪些能力；
- 父 Agent 的全部角色限制不会机械复制。

这解释了 Plan 场景中的常见误解：父 Agent 的 Plan edit deny 不必然原样限制 `general` 子 Agent；真正要强制贯穿委派的限制，应写进父 Session Permission 或子 Agent policy，而不能只依赖父角色的默认轮廓。

### 6.3 Context：只传显式任务，不复制父历史

子 Session 能看见的直接输入主要是 `params.prompt` 解析出的 Parts，以及它自己后续产生的历史。父 Session 中用户的全部原话、已读文件、Todo 和推理过程不会自动复制。

子 Agent 的 Loop 仍会按自己的 Session、Agent 和 Location 重新组装 System、环境、项目规则与 Tool definitions；这些共同来源可能与父 Session 相同，但来源是子侧重新解析，不是 `parentID` 把父 Provider Request 或父历史复制了一份。

这既减少上下文污染，也带来信息损失风险。任务描述缺少路径、版本、约束或输出要求时，Subagent 不能神奇地从 `parentID` 取回这些隐含背景。

### 6.4 结果：返回文本摘要，不转移最终责任

前台委派把子 Session 最后文本作为 `task` output。父 Agent 应当检查：结果是否覆盖约定范围、引用路径是否存在、结论是否与已知事实冲突、是否需要补充一手证据。

因此父子职责不是“父 Agent 把责任交给子 Agent”，而是：

```text
子 Agent：完成边界明确的局部调查
父 Agent：验证局部结果并对最终回答负责
```

## 七、何时使用协作，何时保持单 Agent

### 7.1 适合委派的条件

以下条件同时满足得越多，委派越有价值：

- 子任务能独立描述目标、范围和完成条件；
- 需要大量搜索或阅读，中间轨迹会挤占父 Context；
- 与父 Agent 当前工作低耦合，减少同时修改同一对象的冲突；
- 结果可以由父 Agent 通过路径、源码或检查表独立验证；
- 专用 Agent 的 Permission 或指令确实更适合这项工作。

例如，在限定目录中建立源码入口清单、对照固定 commit 核查某个模块、按明确标准交叉审核文档，都容易形成可验收的子任务。

### 7.2 不适合委派的条件

以下情况通常由一个 Agent 完成更清楚：

- 只需读取少量短文件；
- 子任务强依赖当前对话的大量隐含背景；
- 多个角色必须频繁修改同一批文件；
- 父 Agent 无法独立验证返回结果；
- 描述、等待和验收成本已超过直接完成成本。

可以把复杂度阶梯写成：

```text
一次生成足够             -> Model 调用
需要行动和反馈           -> 一个 Agent Loop
需要显式进度             -> 当前 Session 加 Todo
需要角色边界             -> 选择合适 Agent
需要独立调查与上下文隔离 -> task + Subagent
```

后一级不是前一级的“高级版”，而是在新增约束确实存在时才引入。

### 7.3 前台是默认，后台是实验能力

#### 7.3.1 前台委派等待子结果并结算当前 Tool Call

默认 `task` 是前台委派：父 Tool Call 等子 Session 完成，再获得 Tool Result。

#### 7.3.2 后台委派先返回 running，再异步注入结果

`background: true` 只有在 `OPENCODE_EXPERIMENTAL_BACKGROUND_SUBAGENTS=true` 时可用。它会先返回 running 结果，子任务结束后再向父 Session 注入 synthetic User Message 触发后续处理；父 Run 取消还会协调取消相关后台任务。

后台模式改变了结果到达顺序、并发冲突和生命周期管理，应被视为实验分支，而不是理解普通 Subagent 的前置条件。

## 八、current 与 native V2 的协作边界

### 8.1 当前兼容 Runtime 已形成完整父子链路

当前默认 TUI 使用的兼容 Runtime 中，Agent Registry、Todo、Task Tool、子 Session 创建、Model/Permission 派生、前台与实验后台结果结算都已接线。完整主线是：

```text
父 Session Agent Loop
-> task Tool Call
-> TaskTool 参数 / 深度 / Permission
-> Session.createNext(parentID)
-> 子 Session SessionPrompt.prompt
-> 子 Agent Loop
-> 最终 Text
-> 父 task Tool Result
-> 父 Loop continuation 与验收
```

### 8.2 native V2 有 Agent 与 Todo，不等于 Task/Subagent 已迁移

固定源码中，native V2 已实现 Agent Registry、基础 `build`/`plan`/`general`/`explore` 配置和 Todo Tool，也有 Session `parentID` 等数据结构。但其 built-in tool parity 明确仍缺当前兼容路径的完整 `task`，Plan/Build 切换提醒与部分 Agent request/model 行为也未完全对齐。

因此不能从以下任一现象推出“native V2 已支持完整多 Agent 编排”：

- schema 中存在 `mode: "subagent"`；
- Session 中存在 `parentID`；
- native Agent Registry 能列出 general/explore；
- native Todo 已可保存。

父子 Session 编排必须同时具备委派入口、权限与深度校验、子 Prompt 执行、取消、结果结算和父 Loop continuation。固定基线下，这条完整链路仍属于当前兼容 Runtime，而不是 native V2 已完成的 parity。

## 九、关键源码索引

正文只保留能解释机制的短代码。继续核对时，可以从以下入口进入；更完整的文件、测试和状态说明见 [源码与证据索引](./appendices/Source_Index.md)。

| 要回答的问题 | 关键入口 |
| --- | --- |
| Agent 配置包含什么 | `packages/opencode/src/agent/agent.ts`：`Agent.Info` |
| build、plan、general、explore 如何定义 | `packages/opencode/src/agent/agent.ts`：内置 Agent 初始化 |
| Plan 的提醒与切换如何形成 | `packages/opencode/src/session/reminders.ts`、`packages/opencode/src/tool/plan.ts` |
| Todo 如何写入 Session | `packages/opencode/src/tool/todo.ts`、`packages/opencode/src/session/todo.ts` |
| Task 参数、深度、子 Session、Model 与结果 | `packages/opencode/src/tool/task.ts`：`TaskTool.execute`、`TaskTool.runTask` |
| 子 Session Permission 怎样派生 | `packages/opencode/src/agent/subagent-permissions.ts`：`deriveSubagentSessionPermission` |
| `parentID` 保存在哪里 | `packages/opencode/src/session/session.ts`：`Session.createNext` |
| 子 Agent 有效 Tool 怎样解析 | `packages/opencode/src/session/tools.ts`：`SessionTools.resolve` |
| native V2 Agent 与 built-in tool 边界 | `packages/core/src/agent.ts`、`packages/core/src/plugin/agent.ts`、`packages/core/src/tool/builtins.ts` |

## 十、总结：专业化是配置，协作是受控的父子 Session

OpenCode 的多 Agent 结构可以归结为一条清晰因果链：Model 提供生成能力；Agent 用指令、Model 偏好、Permission 和运行参数把能力塑造成角色；Plan 是一种 primary 角色边界；Todo 保存当前 Session 的可见进度；父模型通过 `task` Tool 提出委派；Task Tool 在深度和 Permission 边界内创建子 Session；Subagent 在自己的 Agent Loop 中工作；最终文本作为 Tool Result 回到父 Session，由父 Agent 验收并整合。

理解这条链后，就不会再把“选择 Plan”“写 Todo”“调用 task”和“启动 Subagent”当成同一件事，也不会把子 Session 的逻辑独立误认为远程部署。下一篇将继续沿后一个问题展开：TUI、Worker、Server、Provider、Tool 与事件通道实际跨过哪些逻辑、进程和网络边界。

# Agent 专业化与协作：OpenCode 如何让不同角色各司其职

上一篇：[10 Session 与 Persistence](./10_Session_and_Persistence.md)

下一篇：[12 Runtime Boundary](./12_Runtime_Boundary.md)

## 1. 学习问题：为什么一个 Agent 还不够

假设你刚开始学习 Harness，并向 OpenCode 提出：

> 请带我从零理解这个项目里的 Harness。先找到学习入口和项目规则，再给出阅读顺序；如果范围较大，可以把资料调查交给合适的角色。

模型当然可以直接回答。但 OpenCode 还可以让不同 Agent 使用不同的指令、工具和权限，或者把一个边界清晰的调查任务交给 Subagent。为什么需要这些对象？`Plan`、Todo、`task` 和 Subagent 又分别做什么？

### 最短答案

模型（Model）提供推理能力；代理配置（Agent）规定这次运行以什么角色、工具和权限使用模型。Plan 是一种受约束的主 Agent 工作方式，Todo 是 Session 中可观察的清单，`task` 是委派工具，Subagent 则是在独立子 Session 中工作的另一个 Agent。

它们解决的是不同问题：

- Agent 解决“以什么能力边界工作”；
- Plan 解决“先调查和规划，暂不直接改动一般代码”；
- Todo 解决“如何显式记录当前步骤”；
- Task Tool 解决“如何发起一次委派”；
- Subagent 解决“谁在隔离的上下文里完成被委派的子任务”。

多 Agent 的价值来自专业分工和上下文隔离，而不是 Agent 数量。能由一个 Agent 清楚完成的任务，通常不必拆成多个 Agent。

## 2. 最小心智模型

先用下面这张图区分对象。箭头表示选择或委派关系，不表示这些对象都运行在不同进程中。

```text
                    同一个或不同的 Model
                              |
                              v
用户目标 -> 主 Agent 配置 -> 父 Session
              |                 |
              |                 +-> Todo：记录可见进度
              |                 |
              |                 +-> task Tool Call
              |                         |
              |                         v
              +-> 指令 / Tools      子 Agent 配置
                  / Permission            |
                                          v
                                      子 Session
                                          |
                                          v
                                 task Tool Result 回到父 Session
```

这张图有三个关键点：

1. Agent 和 Model 是两个维度。两个 Agent 可以使用同一个 Model，却有不同的行为边界。
2. Todo 留在当前 Session 中描述进度；它不会自动创建子任务。
3. Subagent 不是父 Agent 临时换了一段 Prompt，而是在新的 Session 中接收一份明确任务。

第 07 篇已经解释 Agent Loop 如何持续多轮运行。本篇只关心谁以什么策略参与这个循环，以及任务怎样跨父子 Session 传递。

## 3. Agent 不等于 Model

### 3.1 Model 提供推理能力

Model 接收当前上下文，生成文本或 Tool Call。它决定下一步更像“应该读取哪份教程”还是“已经可以总结”，但它本身不保存 OpenCode 的 Session，也不直接拥有文件系统权限。

更换 Model，通常改变基础推理能力、速度、成本和工具调用表现。

### 3.2 Agent 定义使用模型的方式

在 OpenCode 当前默认实现中，`Agent.Info` 不只包含一段提示词，还可以包含：

| 配置维度 | 它回答的问题 |
| --- | --- |
| `name`、`description`、`mode` | 这个角色是谁，作为主 Agent、Subagent 还是两者使用 |
| `prompt` | 它遵循什么专门行为指令 |
| `model`、`variant` | 它偏好哪一个 Model 和变体 |
| `permission` | 它可以申请或使用哪些能力 |
| `steps` | 它使用什么步骤预算提示或限制语义 |
| `temperature`、`topP`、`options` | Provider 请求采用哪些运行参数 |

因此可以得到一个更准确的公式：

```text
Agent = 角色指令 + Model 偏好 + Tool/Permission 边界 + 运行参数
```

它不是：

```text
Agent = Model
Agent = 一段人格 Prompt
```

例如，`build` 和 `plan` 可以落到同一个 Model 上。真正让它们表现不同的，是 Agent 配置、权限规则、提醒和相关工具共同形成的边界。

## 4. OpenCode 内置角色怎样分工

固定源码基线下，常见内置角色可以分成三类。

### 4.1 `build`：默认的执行型主 Agent

`build` 是可见的 primary Agent，也是没有自定义默认配置时通常被选中的角色。它可以根据配置后的 Permission 使用读、写、Shell 等工具；某些敏感资源仍可能触发询问或拒绝。

在学习场景里，适合让 `build`：

- 读取本系列的 README 和教程文件；
- 查找 `AGENTS.md` 等项目规则；
- 解释当前 Session 中出现的 Tool Call；
- 在用户明确要求时，创建影响范围清楚的临时学习文件。

“执行型”不表示所有操作都会无条件通过。工具是否可见、是否需要确认以及最终能否执行，仍由 Permission 和 Tool Runtime 控制，详见[第 09 篇](./09_Tools_and_Permission.md)。

### 4.2 `plan`：受约束的规划型主 Agent

`plan` 也是 primary Agent，但默认拒绝一般编辑，只为指定的计划文件保留写入例外，并允许与规划模式切换有关的能力。它的重点是先调查、澄清和形成方案。

在学习场景里，可以先让 `plan` 回答：

> 为零基础学习者制定 Harness 阅读顺序。先检查 README、项目规则和章节标题，不修改教程正文。

此时 `plan` 仍然可以读取信息、搜索文件并提出问题；“规划模式”并不等于 Model 停止使用工具或只输出内在思考。准确说法是：默认策略收紧了可执行范围，尤其是一般编辑。

这些是固定源码下的默认轮廓。用户配置可以覆盖内置规则，所以不能把“Plan 永远不能编辑或委派”写成不受配置影响的绝对结论。

### 4.3 `general` 与 `explore`：用于委派的 Subagent

`general` 和 `explore` 默认属于 subagent：

- `general` 面向通用、多步骤的子任务；
- `explore` 面向文件查找、关键词搜索和代码库调查，默认能力更偏只读探索。

学习 Harness 时，如果只需定位“哪些文件解释 Agent、Context 和 Tool”，`explore` 比通用执行角色更贴合任务。若子任务需要综合多份材料并形成结构化结论，`general` 更合适。

OpenCode 还有 `compaction`、`title`、`summary` 等 hidden Agent，承担内部摘要或标题工作。它们说明 Agent 配置也可服务内部流程，不必都是用户可切换的“人格”。

## 5. Plan、Todo、Task 与 Subagent 不是同一层概念

这是本篇最容易混淆的一组名称。

| 名称 | 本质 | 是否执行工作 | 是否创建新 Session |
| --- | --- | --- | --- |
| Plan Agent | 一份受约束的 Agent 配置与工作模式 | 可以调查和规划；默认限制一般编辑 | 否 |
| Todo | 当前 Session 的结构化有序清单 | 否，只记录状态 | 否 |
| `task` | 一个特殊 Tool | 发起或恢复委派 | 是，或恢复已有子 Session |
| Subagent | 使用另一 Agent 配置的执行者 | 是，在自己的 Agent Loop 中工作 | 是 |

### 5.1 Todo 是清单，不是调度器

`todowrite` 接收完整 Todo 数组，更新当前 Session 的有序清单。例如：

```text
1. [in_progress] 阅读 Harness README
2. [pending] 找到 Agent 与 Tool 的主讲章节
3. [pending] 完成一次只读 Tool 观察
4. [pending] 总结后续学习路线
```

Todo 的价值是把自然语言计划变成用户和系统可以观察的状态。但它不会：

- 自动调用 Model；
- 自动执行下一项；
- 自动创建后台任务；
- 因为还有 pending 项就强制 Agent Loop 继续；
- 因为全部 completed 就强制 Agent Loop 停止。

Todo 状态和 Loop 停止条件属于两套机制。

### 5.2 Task Tool 是委派入口

父模型若决定把“查找项目内全部 Harness 学习入口”交给 `explore`，生成的是一个 `task` Tool Call。它包含子任务说明、明确的 prompt、Subagent 类型，以及可选的已有 `task_id`。

这仍然是一项 Tool Call：OpenCode 会检查参数、深度和 `task:<subagent_type>` Permission，再决定是否执行委派。模型说“我已委派”并不等于子 Session 已经创建。

### 5.3 Subagent 是独立的问题解决者

Task Tool 真正执行后，Subagent 会在独立子 Session 中运行。它有自己的：

- Session ID；
- Session History；
- Agent 配置；
- 有效 Permission；
- Model 选择；
- Agent Loop。

它不是远程 A2A Agent，也不是必然位于另一个进程。这里的“独立”首先指 Session 和上下文边界。

## 6. 贯穿场景：从零建立 Harness 学习地图

下面只走角色协作部分，不重复第 07 篇的完整多轮循环。

### 第一步：主 Agent 接收学习目标

用户要求 OpenCode：

> 请先阅读 Harness 目录的 README 和项目规则，告诉我主系列的学习顺序。内容很多时，先制定计划；只有在确实能减少上下文干扰时才委派调查。

主 Agent 可以先读取 README 和 `AGENTS.md`。这是范围清楚、风险低的观察任务，一个 Agent 通常足够。

### 第二步：用 Plan 明确边界

如果目录较大，可以先在 Plan 模式形成策略：

```text
目标：建立主系列学习地图
范围：README、06-12 的标题和章节摘要、项目规则
暂不做：修改正文、运行外部命令、访问无关目录
完成标准：说明推荐顺序以及每篇解决的问题
```

Plan 把调查范围变得清楚，但并不会自动把这些步骤变成后台任务。

### 第三步：用 Todo 暴露进度

主 Agent 可以把计划写成 Todo。用户由此看见现在在读规则，下一步准备整理章节，而不必从长段回复中猜测状态。

如果材料很少，主 Agent 继续完成所有 Todo 即可。此时增加 Subagent 只会多出任务描述、上下文传递和结果汇总成本。

### 第四步：只委派边界清楚的调查

假设 research 目录很大，而当前目标只需要回答：

> 哪些研究文件分别支撑 Agent、Context、Tool、Persistence 和 Runtime Boundary？请只返回文件清单、每份材料的一句话用途和证据范围，不修改任何文件。

这是适合 `explore` 的子任务，因为：

- 输入范围清楚；
- 期望产物明确；
- 可以只读完成；
- 结果可以压缩后返回父 Session；
- 子 Session 的大量搜索轨迹不必占满父 Session。

### 第五步：父 Agent 验收结果

Subagent 完成后，Task Tool 将结果作为父 Assistant 中的 Tool Result 结算。父 Agent 在下一轮看到的是返回结果，而不是子 Session 的全部历史。

父 Agent 仍需：

1. 检查结果是否覆盖任务范围；
2. 与自己读取的 README 和项目规则交叉验证；
3. 处理遗漏或冲突；
4. 决定是否需要继续委派；
5. 向用户给出统一的学习路线。

委派没有把最终责任转移给 Subagent。父 Agent 仍负责组合、验证和完成用户目标。

## 7. 父子 Session 如何传递任务和结果

### 7.1 创建关系

没有提供 `task_id` 时，当前 Task Tool 创建带 `parentID` 的子 Session。`parentID` 表示结构关系，但它不会把父 Session History 自动复制到子 Session。

因此，Task prompt 必须包含完成子任务所需的信息。一个好的委派应明确写出：

```text
目标：找出 Harness 架构主系列的证据材料
范围：Opencode_Harness/research 目录
约束：只读，不修改文件，不访问外部网络
产物：按模块列出文件名、一句话用途、仍需验证的边界
完成标准：覆盖 Agent、Context、Tool、Persistence、Runtime 五类
```

只写“帮我看看”会让子 Agent 缺少范围、背景和完成标准。

### 7.2 Model 与 Permission

子 Agent 若配置了自己的 Model，优先使用该 Model；否则继承发出 Task Call 的父 Assistant Model。只有继承 Model 时，父 Assistant 的 Variant 才随之继承。

权限也不是简单复制父 Agent：

- Subagent 自己的 Agent Permission 继续生效；
- 父 Session 持久化的 deny 和外部目录规则会参与派生子 Session Permission；
- 父 Agent 的所有角色限制不会机械地原样复制给子 Agent。

这意味着父 Agent 选择了 Plan，不等于任意 Subagent 都自然获得相同的编辑限制。真正的硬边界应落实为 Session Permission、Subagent policy 和 Tool 自身校验，而不是只依赖父 Agent 的角色名称。

### 7.3 结果回传

默认前台委派会等待子任务完成，取子 Session 最后的文本结果，并把它包装为父 Session 中 `task` Tool 的 output：

```text
父 Session：task call
    |
    v
子 Session：独立运行并形成自己的完整历史
    |
    v
子 Session 最终文本
    |
    v
父 Session：task tool result
```

父模型在 continuation 后看到 Tool Result。子 Session 的读取过程、工具结果和中间消息仍保留在子 Session，不会整段拼回父上下文。

这正是上下文隔离的主要收益，也是委派时必须提供完整任务契约的原因。

## 8. 什么时候应该使用 Subagent

可以用四个问题判断：

1. 子任务能否用一句清楚的话定义目标和完成标准？
2. 子任务是否需要独立的大量搜索或材料阅读，可能污染主上下文？
3. 子任务是否能以一个可验证的结果返回，而不是依赖父 Session 的每个隐含细节？
4. 分工收益是否大于任务描述、上下文传递、等待和结果验证的成本？

适合委派的学习任务：

- 在指定目录内查找所有项目规则；
- 为某一个模块建立源码入口清单；
- 对一组互不重叠的材料分别做只读调查；
- 用明确检查表复核一篇文章的事实边界。

不适合委派的任务：

- 只需读取两三个短文件；
- 强依赖当前对话里大量未显式说明的背景；
- 子任务之间会同时修改同一批文件；
- 结果无法由父 Agent 独立验收；
- 只是为了显得“更 Agentic”。

合理原则是：使用能够可靠完成目标的最低复杂度。

```text
单次回答足够
-> 使用一个 Model 调用

需要读取、观察和多轮调整
-> 使用一个 Agent Loop

需要显式进度
-> 增加 Todo

需要独立专业调查或上下文隔离
-> 再增加 Task + Subagent
```

## 9. 当前实现的边界

### 9.1 委派深度

Task Tool 沿 `parentID` 计算深度。默认 `subagent_depth` 为 1，因此 Subagent 默认不能继续创建更深的 Subagent；配置提高后才允许嵌套。

这个值限制的是委派深度，不是 Provider Turn 数，也不是 Agent Loop 的总步骤数。

### 9.2 后台 Subagent

前台 Task 是当前应先理解的默认路径：父调用等待子任务完成，再取得结果。

`background: true` 需要显式开启 `OPENCODE_EXPERIMENTAL_BACKGROUND_SUBAGENTS=true`。它先返回 running 状态，子任务结束后再向父 Session 注入 synthetic User Message。这是实验能力，不应当成默认协作语义。

### 9.3 native V2 的简短版本说明

固定源码下，native V2 已有 Agent Registry、基础 `build`/`plan`/`general`/`explore` 配置和 Todo Tool，但尚未实现当前兼容路径完整的 Task/Subagent 父子 Session 编排。

因此，本篇描述的 Task Tool、父子 Session、Model/Permission 继承和结果回传，主线都属于当前默认兼容 Runtime。完整迁移关系集中在[第 12 篇](./12_Runtime_Boundary.md)，不应因 V2 数据结构中出现 `mode: "subagent"` 或 `parentID` 就推断委派已经可用。

## 10. 低风险上手观察

可以在自己的学习项目中向 OpenCode 依次提出下面三类请求。示例路径需要替换为你的实际项目路径。

### 观察一：单 Agent 是否已经足够

```text
请只读取当前项目的 README 和 AGENTS.md，列出 Harness 学习入口。
不要修改文件，不运行 Shell，不委派子任务。
```

观察它是否只用读取类工具完成任务。

### 观察二：Plan 与 Todo 的区别

```text
请先为“学习 Harness 主系列”制定四步计划，并用 Todo 显示进度。
这一轮只规划和读取目录说明，不修改教程正文。
```

观察 Plan 提供的行为边界，以及 Todo 只是如何记录进度。

### 观察三：一次边界清楚的委派

```text
如果 research 目录材料较多，请把“只读列出各模块研究文件及用途”
交给适合探索的 Subagent。父 Agent 收到结果后请交叉检查，再给我最终学习地图。
```

观察父 Session 中的 `task` Tool Call、子 Session、最终 Tool Result，以及父 Agent 是否对结果进行了验收。这个练习只需读文件；如果界面提示更高权限，应先检查具体请求，不要为了完成示例一律允许。

## 11. 常见误解

### “更强的 Model 就不需要 Agent 配置”

更强 Model 仍需要知道可见工具、权限、项目规则和职责范围。Model 能力不能替代 Harness 的确定性边界。

### “Plan 是模型隐藏思考的展示”

Plan 是 Agent 工作方式和权限轮廓，不是对模型私有推理过程的读取。

### “写了 Todo，系统就会按顺序执行”

Todo 是结构化状态。下一步是否执行仍由 Agent Loop、模型判断和 Harness 控制共同决定。

### “Task Tool 本身就是 Subagent”

Task Tool 是委派入口；Subagent 是在子 Session 中工作的 Agent。前者发起流程，后者执行子任务。

### “子 Agent 天然继承父 Agent 的完整上下文和权限”

新子 Session 不自动复制父历史，权限也按 Subagent 与 Session 规则重新组合。必要背景必须写进 Task prompt。

### “多 Agent 一定更快、更可靠”

每次拆分都会增加上下文传递、结果汇总、权限和错误传播边界。只有分工或隔离收益更大时才值得拆分。

## 12. 本篇掌握要点

读完本篇，应能解释：

1. Model 负责推理，Agent 定义怎样使用 Model；二者不能互换。
2. `build` 与 `plan` 的主要差异来自指令、工具和 Permission 边界，不是不同的“模型人格”。
3. Todo 记录当前 Session 的进度，不执行工作，也不控制 Agent Loop 停止。
4. `task` 是委派 Tool，Subagent 是独立子 Session 中的执行者。
5. 新子 Session 不自动复制父历史；Task prompt 必须包含目标、范围、约束、产物和完成标准。
6. Subagent 结果以 Tool Result 回到父 Session，父 Agent 仍负责验证和汇总。
7. 默认委派深度有限，后台模式是实验能力，native V2 的 Task/Subagent parity 尚未完成。
8. Agent 设计应选择能可靠完成目标的最低复杂度。

## 13. 关键源码入口

本文结论以 OpenCode 固定 commit `0e3474509aa5ad16afcf9c439785514d6443c6af` 为基线。行号可能随版本变化，优先按导出符号查找。

| 主题 | 文件 | 关键符号 |
| --- | --- | --- |
| Agent 数据与内置角色 | `packages/opencode/src/agent/agent.ts` | `Agent.Info`、Agent layer、`defaultInfo` |
| Plan 提醒与切换 | `packages/opencode/src/session/reminders.ts`、`packages/opencode/src/tool/plan.ts` | `SessionReminders.apply`、`PlanExitTool` |
| Todo Tool 与存储 | `packages/opencode/src/tool/todo.ts`、`packages/opencode/src/session/todo.ts` | `TodoWriteTool`、`Todo.update`、`Todo.get` |
| Task 与子 Session | `packages/opencode/src/tool/task.ts` | `TaskTool`、`TaskTool.execute`、`TaskTool.runTask` |
| 子 Session 权限派生 | `packages/opencode/src/agent/subagent-permissions.ts` | `deriveSubagentSessionPermission` |
| 父子 Session 创建 | `packages/opencode/src/session/session.ts` | `Session.createNext` |
| Agent 默认行为测试 | `packages/opencode/test/agent/agent.test.ts` | build、plan、默认 Agent 与权限测试 |
| Task 行为测试 | `packages/opencode/test/tool/task.test.ts` | 创建、恢复、深度和取消测试 |
| 父子权限边界测试 | `packages/opencode/test/agent/plan-mode-subagent-bypass.test.ts` | Subagent policy 与父 Session deny 测试 |
| native V2 Agent/Todo | `packages/core/src/agent.ts`、`packages/core/src/plugin/agent.ts`、`packages/core/src/tool/todowrite.ts` | `AgentV2.select`、内置 Agent、`todowrite` |
| native V2 Task 缺口 | `packages/core/src/tool/builtins.ts`、`specs/v2/todo.md` | remaining Tool port、Subagent/BackgroundJob 规划 |

下一篇将把这些逻辑角色放回实际运行环境，解释 TUI、Worker、Server、Provider、Tool Runtime 与事件通道分别位于哪里。

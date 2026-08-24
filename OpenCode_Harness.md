# OpenCode Harness 学习笔记

OpenCode 通过一套位于模型之外的系统组织信息、执行操作、保存状态并控制运行循环。这套系统称为 **Agent Harness**。

本文依次介绍工具调用、Context 组装、Agent Loop 和扩展机制。

## 一、工具调用

### 1.1 工具调用的完整流程

以下以"读取 README "的对话为例，说明一次普通本地工具调用的处理顺序：

```text
准备模型请求
用户目标 + 对话历史 + read 的名称、说明和参数格式
        ↓
返回结构化 Tool Call
read({ filePath: "README.md" })
        ↓
检查工具与调用参数
确认工具存在、参数合法且本次读取已获授权
        ↓
执行 read
本地执行器读取文件
        ↓
保存 Tool Result
成功时保存读取内容，失败时保存错误
        ↓
发起下一次模型请求
模型根据 Tool Result 生成后续调用或最终回答
```


一次 Provider 请求及其响应称为一个 **Provider Turn**。上述过程通常包含两个请求轮次：第一个轮次取得 Tool Call，第二个轮次将 Tool Result 提供给模型。参数校验、权限检查和文件读取位于两个请求轮次之间，由 Harness 处理。

请求模型时，Harness 不只发送用户消息，还发送了 Tool definitions。
概念上的请求如下：
```json
{
  "model": "GPT-5.6-sol",
  "messages": [
    {
      "role": "user",
      "content": "请读取这个项目的 README 和项目规则，说一下你对项目的理解。"
    }
  ],
  "tools": [
    {
      "name": "read",
      "description": "读取指定文件或目录",
      "parameters": {
        "type": "object",
        "properties": {
          "filePath": {
            "type": "string"
          }
        },
        "required": ["filePath"]
      }
    }
  ],
  "tool_choice": "auto"
}
```
### 1.2 Tool Call 的 API 表示

模型 API 不是只能返回文本。HTTP API 可以返回任意结构化 JSON；文本只是 JSON 中的一种内容类型。LLM API 返回的是一个包含多种事件或内容块的 JSON 响应。

最简单的模型响应，概念上类似：
```json
{
  "output": [
    {
      "type": "text",
      "text": "README 中解释了项目的能力。"
    }
  ]
}
```
如果模型决定调用工具，返回的可以是另一种结构：

```json
{
  "output": [
    {
      "type": "tool_call",
      "id": "call_123",
      "name": "read",
      "arguments": {
        "filePath": "/project/README.md"
      }
    }
  ]
}
```

不同 Provider 使用的协议名称不完全相同。例如 OpenAI Responses 可能返回 `function_call`，Anthropic 可能返回 `tool_use`。

模型服务内部可以通过特殊 token、约束生成或 Provider 自有解析逻辑形成这些字段，具体实现并未完全公开。
因此，OpenCode 接收的是带明确类型的 Tool Call，不是从“读取文件”等自然语言表述中推断调用意图。

### 1.3 参数校验、权限检查与执行

OpenCode 收到完整 Tool Call 后，不会立即执行，而是依次进行参数校验、权限检查和调用分派。普通本地工具的处理阶段如下：

```text
结构化 Tool Call
        ↓
Schema 校验：参数的字段和类型是否合法
        ↓
Typed decode：转换成 Tool executor 需要的输入类型
        ↓
Permission：这次具体资源访问是否允许
        ↓
Executor：执行真实操作
        ↓
Settlement：保存 completed 或 error 终态
```


### 1.4 Tool Result 的保存与回传

执行器读取 README 后，文件内容首先返回 Harness，不会自动成为模型的持续记忆。`SessionProcessor` 将调用结算为 `completed` Tool Part，并保存 input、output、时间和 metadata；执行失败或用户拒绝则保存为 `error`。

接下来，外层 Agent Loop 重新读取 Session，把 Tool Call 和 Tool Result 转换成 Provider 能接受的 Messages，再发起下一次 Provider Turn：

```text
Provider Turn 1
Messages + read definition
        ↓
Model 返回 read Tool Call
        ↓
OpenCode 验证、授权、执行并保存 Tool Result
        ↓
Provider Turn 2
Messages + read Tool Call + read Tool Result
        ↓
Model 根据 README 继续调用工具，或生成最终回答
```

该流程的责任边界如下：模型生成 Tool Call，Provider 按协议返回结构化对象；Harness 完成校验、授权和执行，并在下一次模型请求中加入 Tool Result。

## 二、 Context 组装

### 2.1 Context 简介

模型没有持续打开的项目视野。每次 Provider Turn 开始前，Harness 都要重新构造模型这一轮实际收到的信息环境：

```text
Context
= System
+ Messages
+ Tool definitions
+ Provider-specific transform
```


| 通道 |  常见内容 |
| -- |  --- |
| System |  Provider/Agent 基础指令、Environment、项目规则、Skill/MCP guidance |
| Messages |  User、Assistant、Tool Call、Tool Result、Compaction 后的活跃历史 |
| Tool definitions |  Tool 名称、说明、参数 Schema |


### 2.2 Context 组装

Agent Loop 回到顶部后，会重新读取 Compaction 后的活跃历史，并为新的 Provider Turn 准备 Context。固定源码中的核心关系如下：

```ts
// packages/opencode/src/session/prompt.ts
const [skills, env, instructions, mcpInstructions, modelMsgs] = yield* Effect.all([
  sys.skills(agent),
  sys.environment(model),
  instruction.system().pipe(Effect.orDie),
  sys.mcp(agent, session.permission),
  MessageV2.toModelMessagesEffect(msgs, model),
])

const system = [
  ...env,
  ...instructions,
  ...(mcpInstructions ? [mcpInstructions] : []),
  ...(skills ? [skills] : []),
]
```

只有新的外层迭代才会重新完成这套选择和组装。同一个 `SessionProcessor.process()` 内的 Provider Retry 通常复用已经准备好的 `streamInput`，不会在每次重试前重新读取最新规则和历史。
产生新的外层迭代的条件有：返回 Tool Result；上下文需要 Compaction；需要处理 Subtask；用户有新的请求；subagent返回result。

### 2.3 渐进式披露
OpenCode 的渐进式披露策略：
                                                          
  第 1 层：摘要（Summary）                                    
  ├── 技能列表：只显示名称和描述                              
  ├── 目录列表：只显示文件名                                  
  └── 工具列表：只显示工具名和参数 Schema                     
                                                             
  第 2 层：按需加载（On-Demand Loading）                      
  ├── skill 工具：LLM 需要时才加载完整技能内容                
  ├── Read 工具：支持 offset/limit 按需读取文件片段           
  └── Grep 工具：只搜索匹配的内容，不加载整个文件             
                                                             
                                                              
  第 3 层：自动截断（Auto Truncation）                       
  ├── 工具输出超过 2000 行或 50KB 自动截断                    
  ├── 完整内容保存到磁盘，给 LLM 文件路径                    
  └── LLM 可以用 Task 工具让子 agent 处理大文件               


### 2.4 Compaction 压缩策略

当对话历史太长时，OpenCode 会自动压缩历史，生成摘要。

当活跃历史接近 Context Window 边界时，OpenCode 可以创建 Compaction，让专用流程总结早期工作，并保留一段近期 Tail：

```text
较长的旧历史 ——> Compaction ——> Compaction result
+ Assistant summary
+ retained recent tail
+ Compaction 后的新消息
```

源码位置: session/compaction.ts
```ts
export namespace SessionCompaction {
  const COMPACTION_BUFFER = 20_000  // 20K token 缓冲

  // 检测是否溢出
  export async function isOverflow(input: { tokens: MessageV2.Assistant["tokens"]; model: Provider.Model }) {
    const context = input.model.limit.context
    const reserved = config.compaction?.reserved ?? Math.min(COMPACTION_BUFFER, maxOutputTokens)
    const usable = context - reserved
    return count >= usable
  }

  // 修剪旧的工具输出
  export async function prune(input: { sessionID: SessionID }) {
    // 从后向前遍历，找到超过 40K token 的工具调用
    // 修剪它们的输出，但保护 skill 工具调用
    const PRUNE_PROTECT = 40_000
    const PRUNE_PROTECTED_TOOLS = ["skill"]  // skill 工具输出不被修剪
    // ...
  }

  // 压缩对话历史
  export async function process(input: {...}) {
    // 使用专门的 "compaction" agent 生成摘要
    const agent = await Agent.get("compaction")
    // ...
  }
}
```
压缩提示词模板：
```text
Provide a detailed prompt for continuing our conversation above.
Focus on information that would be helpful for continuing the conversation.

---
## Goal
[What goal(s) is the user trying to accomplish?]

## Instructions
[What important instructions did the user give you?]

## Discoveries
[What notable things were learned?]

## Accomplished
[What work has been completed, what is still in progress?]

## Relevant files / directories
[Structured list of relevant files]
---
```

Summary 尽量保留目标、约束、已完成工作、关键发现和下一步；Tail 保存最近交互的原始细节。没有进入 Summary、又不在 Tail 中的旧细节，未来可能不再对模型可见。


### 2.6 Pruning 修剪策略

Pruning 的对象更窄：它处理较旧、较大的 completed Tool output。修剪后：标记的工具输出会被替换为 [已修剪]。
会把正文替换为类似：

```text
[Old tool result content cleared]
```

模型仍能知道调用过哪个 Tool、使用了什么输入，并知道旧结果已被清理；原 Tool Part 不一定因此从持久化记录中物理删除。

源码 (session/compaction.ts:51-100):
```ts
const PRUNE_PROTECT = 40_000      // 保护最近 40K token 的工具输出
const PRUNE_MINIMUM = 20_000     // 至少修剪 20K token 才执行
const PRUNE_PROTECTED_TOOLS = ["skill"]  // skill 工具输出不被修剪

export async function prune(input: { sessionID: SessionID }) {
  // 从最新的消息向前遍历
  for (let msgIndex = msgs.length - 1; msgIndex >= 0; msgIndex--) {
    // 跳过最近 2 轮对话
    if (turns < 2) continue
    // 如果遇到摘要消息，停止
    if (msg.info.role === "assistant" && msg.info.summary) break

    for (let partIndex = msg.parts.length - 1; partIndex >= 0; partIndex--) {
      const part = msg.parts[partIndex]
      if (part.type === "tool" && part.state.status === "completed") {
        // skill 工具输出不被修剪
        if (PRUNE_PROTECTED_TOOLS.includes(part.tool)) continue
        // 统计 token，超过阈值就标记为修剪
        // ...
      }
    }
  }
}
```

## 三、结构化执行

### 3.1 Agent Loop 迭代

忽略少见错误后，一次普通迭代可以概括为：

1. 将 Session 标记为 busy。
2. 重载 Compaction 后的活跃历史。
3. 找到最新 User、Assistant、已完成消息和特殊任务。
4. 执行顶部终止检查。
5. 优先处理 Subtask、Compaction 或 Context Overflow。
6. 普通路径创建新的 Assistant Message。
7. 重新物化 Tools，组装 System 与 Provider Messages。
8. 创建 SessionProcessor，发起一次 Provider Turn。
9. 把流式事件结算成 Message / Part。
10. 根据 `continue / compact / stop` 决定下一步。


### 3.2 Plan：先规划，再决定是否执行

Plan 是一种 primary Agent 工作模式。它仍然运行普通 Agent Loop，只是使用了不同的提示词和 Permission：允许读取、搜索、提问和编写指定的 Plan 文件，默认禁止修改其他项目文件。

固定源码中，`build` 和 `plan` 都是 primary Agent。二者不是每个任务都必须依次经过的固定流水线：简单任务可以直接使用 Build；需要先调查范围、比较方案或控制修改风险时，再进入 Plan。

Plan Agent 的核心边界位于 `packages/opencode/src/agent/agent.ts`：

```ts
plan: {
  name: "plan",
  description: "Plan mode. Disallows all edit tools.",
  permission: Permission.merge(
    defaults,
    Permission.fromConfig({
      question: "allow",
      plan_exit: "allow",
      task: { general: "deny" },
      edit: {
        "*": "deny",
        [path.join(".opencode", "plans", "*.md")]: "allow",
        // 只允许修改 OpenCode 数据目录中的 Plan 文件
      },
    }),
    user,
  ),
  mode: "primary",
}
```


#### Plan 文件和 Plan 提示词

实验性 Plan Mode 会把一段 synthetic reminder 加入当前 User Message，告诉模型：

- 当前处于只读规划阶段；
- 可以读取、搜索和使用 `question` 澄清需求；
- 唯一允许编辑的是指定 Plan 文件；
- Plan 应包含推荐方案、关键文件和验证方法；
- 完成后调用 `plan_exit`，而不是直接实施。

这段提示词把规划过程分成理解、设计、复核、写最终 Plan 和请求批准五个阶段。固定源码中的原文如下：

```text
<system-reminder>
Plan mode is active. The user indicated that they do not want you to execute yet -- you MUST NOT make any edits (with the exception of the plan file mentioned below), run any non-readonly tools (including changing configs or making commits), or otherwise make any changes to the system. This supersedes any other instructions you have received.

## Plan File Info:
${planInfo}
You should build your plan incrementally by writing to or editing this file. NOTE that this is the only file you are allowed to edit - other than this you are only allowed to take READ-ONLY actions.

## Plan Workflow

### Phase 1: Initial Understanding
Goal: Gain a comprehensive understanding of the user's request by reading through code and asking them questions. Critical: In this phase you should only use the explore subagent type.
[...]

### Phase 2: Design
Goal: Design an implementation approach.

[...]
### Phase 3: Review
Goal: Review the plan(s) from Phase 2 and ensure alignment with the user's intentions.

[...]
### Phase 4: Final Plan
Goal: Write your final plan to the plan file (the only file you can edit).

[...]
### Phase 5: Call plan_exit tool
At the very end of your turn, once you have asked the user questions and are happy with your final plan file - you should always call plan_exit to indicate to the user that you are done planning.

**Important:** Use question tool to clarify requirements/approach, use plan_exit to request plan approval. Do NOT use question tool to ask "Is this plan okay?" - that's what plan_exit does.

NOTE: At any point in time through this workflow you should feel free to ask the user questions or clarifications. Don't make large assumptions about user intent. The goal is to present a well researched plan to the user, and tie any loose ends before implementation begins.

```

#### `plan_exit` 人机交接

Plan 完成后，模型调用 `plan_exit`。该 Tool 不会直接打开写权限，而是先询问用户：

```text
Plan 已完成
        ↓
是否切换到 Build Agent 并开始实施？
        ↓
No  -> 留在 Plan，继续修改方案
Yes -> 写入一条 synthetic User Message
        agent = "build"
        ↓
下一轮由 Build Agent 读取获批 Plan 并执行
```

源码中的关键部分位于 `packages/opencode/src/tool/plan.ts`：

```ts
const answers = yield* question.ask({
  sessionID: ctx.sessionID,
  questions: [{
    question: `Plan at ${plan} is complete. Would you like to switch to the build agent and start implementing?`,
    options: [
      { label: "Yes", description: "Switch to build agent and start implementing the plan" },
      { label: "No", description: "Stay with plan agent to continue refining the plan" },
    ],
  }],
})

if (answers[0]?.[0] === "No") yield* new Question.RejectedError()

// 用户同意后创建 agent = "build" 的 synthetic User Message
```

### 3.3 Todo：保存进度，但不调度任务

Todo 是当前 Session 的结构化任务清单。每一项包含：

```text
content   要完成什么
status    pending / in_progress / completed / cancelled
priority  high / medium / low
```

一个典型清单如下：

```text
[✅] 读取相关源码
[...] 确认修改方案
[ ] 实现代码
[ ] 运行测试
```

Todo 的作用是把“模型目前准备做什么、做到哪里”变成可持久化和可观察的状态。真正的下一步仍由模型结合 Context 决定，Harness 仍按 Agent Loop 的终止条件控制运行。

适合使用 Todo 的情况：

- 任务包含多个独立步骤；
- 用户一次提出多个要求；
- 实施和验证之间需要明确进度；
- 执行中发现新的必要工作。

对于单次读取、简单解释或一处小修改，额外维护 Todo 反而会增加上下文和操作成本。

### 3.4 Task 与 Subagent：把子任务交给独立 Session

`task` 是父 Agent 可以调用的委派 Tool，Subagent 是在 Tool 后面真正执行工作的 Agent。二者的关系是：

```text
task = 委派入口和生命周期外壳
Subagent = 子 Session 中持续运行 Agent Loop 的执行者
```

父模型生成的 Task Tool Call 主要包含：

```text
description    给界面显示的简短名称
prompt         子 Agent 要完成的详细任务
subagent_type  使用哪一种 Agent，例如 explore 或 general
task_id        可选，继续已有子 Session
background     可选，实验性后台执行
```

子 Session 有自己的 User Message、Assistant Message、Tool Call/Result 和 Agent Loop。`parentID` 只建立父子关系，不会自动复制父 Session 的完整历史。

#### 子结果返回

默认前台 Task 会等待子 Session 完成，然后把子 Session 最后一个 Text Part 包装成父 Session 的 Task Tool Result：

子 Agent 的所有中间 Tool Result 不会整体塞回父 Context，这可以减少上下文污染。但父 Agent 拿到的是压缩后的结论，仍然需要核对范围、路径和证据，并对最终回答负责。

传入 `task_id` 可以继续已有子 Session，使它保留自己的历史；不传则创建新的子 Session。默认委派深度为一层，防止子 Agent 无限制地继续生成更深层委派。


## 四、Skill、MCP 与 Plugin：三种扩展机制的区别和能力

Skill、MCP 和 Plugin 都能增强 OpenCode，但它们进入系统的位置完全不同：

```text
Skill
用户任务 -> Model 按需加载 SKILL.md -> 按工作方法使用已有 Tools

MCP
用户任务 -> Model 选择 MCP Tool -> OpenCode MCP Client -> MCP Server

Plugin
OpenCode 启动或运行到 Hook -> 进程内 Plugin 代码观察或修改行为
```

先看整体比较：

| 对比项 | Skill | MCP | Plugin |
| --- | --- | --- | --- |
| 核心作用 | 提供可复用的工作方法和专业知识 | 连接外部工具、数据与服务 | 扩展或修改 OpenCode 自身运行行为 |
| 主要形式 | `SKILL.md`，可附带脚本、参考资料和模板 | 本地或远程 MCP Server | JavaScript/TypeScript 模块或 npm 包 |
| 主要进入位置 | guidance 与按需 Tool Result | Tool definitions、Resources、Prompts 与调用结果 | OpenCode 进程和 Hook |
| 是否自动增加新执行能力 | 通常不会 | 通常会增加 Server 提供的能力 | 可以注册 Tool，也可以修改已有流程 |
| 典型使用时机 | Model 判断任务匹配后加载 | OpenCode 连接 Server，Model 需要时调用 | OpenCode 启动时加载，相关 Hook/Event 发生时运行 |
| 主要风险 | 错误指令、附带脚本、许可证 | 数据外发、凭据、远程写操作、工具膨胀 | 进程内代码、供应链、修改参数和上下文 |
| 初学者建议 | 优先学习 | 按明确需求一次连接一个 | 确实需要改变生命周期时再使用 |

### 4.1 Skill：为 Agent 提供可复用的方法

Skill 不是新模型，也通常不是一个新的业务 Tool。它更像一份可按需加载的专业操作手册：

```text
skill-name/
├── SKILL.md        # 必需：名称、触发描述和工作流程
├── references/     # 可选：详细参考资料
├── scripts/        # 可选：Shell、Python、JavaScript 脚本
├── assets/         # 可选：模板、图片、静态数据
└── schemas/        # 可选：格式约束
```

典型加载过程是：

1. OpenCode 扫描支持的 Skill 目录。
2. System 中先提供 Skill 的 `name` 和 `description`。
3. Model 根据当前任务判断是否相关。
4. Model 调用内置 `skill` Tool。
5. 完整 `SKILL.md` 与文件列表作为 Tool Result 进入 Messages。
6. Agent 再按说明读取 references、运行脚本或使用模板。

这种方式叫渐进披露。所有 Skill 的完整正文不会每轮同时塞进 Context，只有相关 Skill 才按需加载。

Skill 本身通常不会赋予系统一种原来完全不存在的执行能力。`SKILL.md` 中即使写着：

```bash
python scripts/check_document.py input.pdf
```

脚本也不会因为位于 `scripts/` 目录就自动执行。Agent 仍然要通过已有的 Bash 等 Tool 运行它，并受到相应 Permission、依赖和 OS 权限约束。

Skill 适合：

- 代码审查清单；
- 测试与发布流程；
- PDF、表格、文档的固定处理方法；
- 团队约定的排障步骤；
- 需要模板、参考资料或验证脚本的重复工作。

Skill 与 `AGENTS.md` 也不同：项目规则通常作为 ambient instructions 持续影响会话；Skill 只先暴露名称和描述，在任务相关时才加载全文。因此，普遍适用且简短的项目规则适合写进 `AGENTS.md`，体积较大、只在特定任务中使用的方法更适合 Skill。

审查 Skill 时不能只看 Markdown。还要检查它引用的脚本、依赖、网络访问、许可证，以及是否要求删除文件、发布制品、修改 Git 历史或上传数据。

### 4.2 MCP：通过统一协议连接外部能力

MCP 是协议，不是一组固定工具。平常所说的“安装 MCP”，通常是让 OpenCode 连接一个 MCP Server：

```text
Model
  -> OpenCode Host
  -> MCP Client
  -> stdio 或 Streamable HTTP
  -> MCP Server
  -> 数据库、浏览器、GitHub、监控平台或其他服务
```

OpenCode 是 Host，它可以为多个 Server 管理 Client 连接。Server 不一定在远程：由 OpenCode 在本机启动、通过标准输入输出通信的子进程也是 MCP Server。

MCP 协议可以描述三类核心原语：

- **Tools**：执行函数或业务动作，例如查询 Issue、运行浏览器操作；
- **Resources**：通过 URI 提供可读取数据，例如文档或数据库记录；
- **Prompts**：提供带参数的提示模板或工作流入口。

实际能否使用哪类原语，还取决于 Server 提供什么、OpenCode 当前实现暴露什么，以及 Agent/Session Permission 是否允许。

MCP 适合：

- 让 OpenCode 查询官方库文档；
- 读取 Sentry、GitHub 或内部平台数据；
- 操作浏览器、工单系统或数据库；
- 把同一套外部能力同时提供给多个支持 MCP 的 Host；
- 将能力和 OpenCode 主进程分离部署。

MCP Tool 会与 `read`、`bash` 等本地 Tool 一起进入本轮 Tool definitions。Server 越多、工具越多，名称、描述和 Schema 占用的 Context 越大，模型也越难准确选择。因此不是“连接越多越强”，而是应该只启用当前项目真正需要的 Server。

安全上要区分本地和远程：

- 本地 MCP Server 是 OpenCode 启动的真实进程，需要审查包、依赖和系统权限；
- 远程 MCP 会把必要请求和数据发送到外部服务，需要最小权限 API Key/OAuth、数据治理和服务条款；
- OpenCode Permission 可以控制受管的 `tools/call`，不能修复 Server 自身漏洞，也不能限制 Server 启动后的内部行为；
- 带创建、修改、删除、发布能力的 Server，初次使用应设为 `ask`，并优先使用测试或只读账号。

### 4.3 Plugin：改变 OpenCode 自己的运行行为

Plugin 是 OpenCode 启动时加载的 JavaScript/TypeScript 模块。它不是 Model 按需阅读的说明，也不是通过 MCP 协议连接的独立 Server，而是直接进入 OpenCode 进程，通过 Hook 观察、拦截或扩展运行流程。

以 Tool Hook 为例：

```text
Model 生成 Tool Call
        ↓
tool.execute.before
Plugin 可以检查、修改或拒绝参数
        ↓
OpenCode 执行 Tool
        ↓
tool.execute.after
Plugin 可以检查结果、记录日志或修改输出
        ↓
结果进入 Settlement
```

常见能力包括：

- 监听 Session idle、错误、Permission 请求和文件修改事件；
- 在 Tool 执行前后检查或修改参数和结果；
- 注入 Shell 环境变量；
- 修改 System、模型消息、请求参数或 Compaction；
- 注册自定义 Tool；
- 集成通知、认证、日志、监控和外部服务。

Plugin 适合：

- 任务结束时发送系统通知；
- 为组织加入统一审计或观测；
- 在所有 Tool 调用前增加保护逻辑；
- 改变 OpenCode 的 Context 或请求处理行为；
- 接入 Provider 认证等运行时能力。

Plugin 的信任要求通常最高。它可能在用户发送任务前就读取文件和环境变量、访问网络、执行 Shell 或修改 Tool 参数。OpenCode 的 Tool Permission 主要约束 Agent 的受管 Tool Call，不能把 Plugin 本身变成沙箱。

如果只需要一个 OpenCode 项目内的简单函数，可以优先考虑 Custom Tool；如果希望能力被多个 Host 复用，优先考虑 MCP；只有确实需要进入 OpenCode 生命周期和 Hook 时，才使用 Plugin。

### 4.4 三者可以组合，但不要混淆责任

Skill、MCP 与 Plugin 不是三个互斥套餐，它们可以组合：

```text
Skill
告诉 Agent 应按什么流程完成发布
        ↓
MCP Tool
让 Agent 查询工单并创建远程 Release
        ↓
Plugin
记录审计日志，并在 Session idle 时发送通知
```

组合后仍然要分别判断：

- Skill 是否给出了可信流程；
- MCP Server 是否应接收当前数据、凭据是否最小化；
- Plugin 是否值得获得进程内能力；
- OpenCode Permission 和 OS Sandbox 分别控制到哪一层。

可以用下面的选择规则收束：

```text
只想教 Agent 一套方法
-> Skill

需要连接外部数据、API 或可复用工具服务
-> MCP

需要监听或改变 OpenCode 自身生命周期
-> Plugin

只需要当前项目里一个简单执行函数
-> Custom Tool
```

## 结语：把 Harness 看成受控反馈系统

理解 OpenCode Harness，不需要先记住所有文件和类名。先掌握下面这条因果链：

```text
Session 中保存用户目标
        ↓
Harness 为本轮组装 Context 与 Tool definitions
        ↓
Model 经 Provider 返回 Text 或结构化 Tool Call
        ↓
Harness 验证、授权并执行真实操作
        ↓
结果结算成 Message / Part 并保存
        ↓
下一轮重新组装 Context，继续判断或结束
```

Model 提供开放任务所需的概率性判断；Harness 将这些判断放进可验证、可授权、可执行、可记录的工程边界。Skill 为 Agent 增加方法，MCP 连接外部能力，Plugin 改变 OpenCode 自身行为——它们扩展的是 Harness 的不同位置。

## 进一步阅读与源码入口

如果希望继续深入，可以按照下面的顺序阅读现有系列：

1. [Harness 总览](./Opencode_Harness/06_Harness.md)
2. [Agent Loop](./Opencode_Harness/07_Agent_Loop.md)
3. [Context Architecture](./Opencode_Harness/08_Context_Architecture.md)
4. [Tools 与 Permission](./Opencode_Harness/09_Tools_and_Permission.md)
5. [Session 与 Persistence](./Opencode_Harness/10_Session_and_Persistence.md)
6. [Agent 专业化与协作](./Opencode_Harness/11_Agent_Specialization_and_Collaboration.md)
7. [Runtime Boundary](./Opencode_Harness/12_Runtime_Boundary.md)
8. [Skill、MCP 与 Plugin](./Opencode_Harness/05_Enhancement.md)

关键源码定位：

| 主题 | 源码文件 | 关键符号 |
| --- | --- | --- |
| 外层 Agent Loop | `packages/opencode/src/session/prompt.ts` | `SessionPrompt.run`、`loop` |
| Context 最终准备 | `packages/opencode/src/session/llm/request.ts` | `LLMRequestPrep.prepare`、`resolveTools` |
| Provider 事件归一化 | `packages/opencode/src/session/llm.ts`、`session/llm/ai-sdk.ts` | `fullStream`、`toLLMEvents` |
| Tool 物化 | `packages/opencode/src/session/tools.ts` | `SessionTools.resolve` |
| Permission | `packages/opencode/src/permission/index.ts` | `evaluate`、`ask`、`reply` |
| `read` executor | `packages/opencode/src/tool/read.ts` | `ReadTool.execute`、`lines` |
| Tool settlement | `packages/opencode/src/session/processor.ts` | `handleEvent`、`completeToolCall` |
| Session History 投影 | `packages/opencode/src/session/message-v2.ts` | `toModelMessagesEffect`、`filterCompacted` |
| Compaction 与 Pruning | `packages/opencode/src/session/compaction.ts` | `serialize`、`select`、`prune` |
| Skill、MCP、Plugin 入口 | `packages/opencode/src/skill/`、`mcp/`、`plugin/` | 各模块加载与运行入口 |

完整术语和跨章节证据见：

- [简明术语表](./Opencode_Harness/appendices/Terminology.md)
- [源码与证据索引](./Opencode_Harness/appendices/Source_Index.md)

## 附录：MindMemOS 项目及其 OpenCode 接入方式

> 本节基于 [MindMemOS](https://github.com/mindscale-noah/MindMemOS) 仓库提交 [`c1befcb`](https://github.com/mindscale-noah/MindMemOS/commit/c1befcb73646b54f7a96724ea5463edb21c03ee0)（2026-08-21）和本文使用的 OpenCode 源码基线。后续版本可能增加新的 Agent 集成、MCP Server 或 OpenCode Plugin。

### A.1 项目定位与系统结构

MindMemOS 是面向 AI Agent 的长期记忆层，用于保存、检索和演进跨会话信息。项目提供的主要能力包括：

- 写入和检索用户偏好、项目事实、任务经验等记忆；
- 更新、删除和反馈已有记忆；
- 通过 dreaming 流程整理和巩固记忆；
- 注册、同步和演进 MindMemOS 管理的 Skill 资产；
- 在不同 Agent 或应用之间复用同一组记忆。

MindMemOS 采用服务端与客户端分离的结构：

```text
Agent / 业务应用
        ↓
HTTP API、Python SDK 或 mindmemos CLI
        ↓
MindMemOS FastAPI 服务
        ↓
记忆抽取、检索、反馈与 dreaming
        ↓
Qdrant、Neo4j 等存储组件
```

用户可以使用官方云服务，也可以按照 [部署文档](https://github.com/mindscale-noah/MindMemOS/blob/main/docs/deploy/instruction_ZH.md) 自行部署。当前本地部署并非单一轻量进程：默认开发环境涉及 FastAPI、Qdrant、Neo4j 和 Kafka，并需要配置聊天模型、Embedding 模型及可选的 Rerank 模型。客户端通过 [`mindmemos-sdk`](https://pypi.org/project/mindmemos-sdk/) 提供 Python API 和 `mindmemos` CLI，两者最终都通过 HTTP 调用服务端。

MindMemOS 与 OpenCode Session 处理的问题不同：

| 对比项 | OpenCode Session | MindMemOS |
| --- | --- | --- |
| 主要目标 | 保存一次 Agent 运行中已经发生的 Message、Part 和 Tool 状态 | 提取并检索可跨会话、跨 Agent 使用的长期记忆 |
| 信息形态 | 对话和工具执行的运行事实 | 经过抽取、去重、组织或演进的记忆条目 |
| 默认作用域 | 当前 OpenCode Session 及其持久化历史 | 由 API key、`user_id`、`app_id`、`agent_id`、`session_id` 等标识划分 |
| 是否替代另一方 | 不能替代外部长时记忆服务 | 不能替代 OpenCode 的运行状态和 Tool Part 记录 |

因此，MindMemOS 适合作为 OpenCode Session 之外的长期记忆层。接入后的典型数据流为：在模型请求前检索相关记忆并加入 Context，在一次交互完成后将适合保留的消息写回 MindMemOS。

MindMemOS 还提供 Skill 注册与演进功能，但该处的 “Skill” 是 MindMemOS 服务管理的可版本化资产。它不会因为接入记忆服务就自动变成 OpenCode 扫描到的 `.opencode/skills/<name>/SKILL.md`。如果需要同步两者，还要显式执行 `mindmemos skill pull`、`push` 等命令，并管理本地 Skill 文件。

### A.2 当前提供的接入界面

截至上述提交，MindMemOS 提供以下接入界面：

当前结论是：仓库中的通用 Skill 可以安装到 OpenCode 并按需调用；仓库尚未提供 MCP Server 和 OpenCode Plugin，这两种形式均需适配开发。MCP 方案需要将 MindMemOS 接口包装为 MCP Server，Plugin 方案则需要按照 OpenCode Hook 接口实现。

| 接口 | 现有状态 | 适用方式 |
| --- | --- | --- |
| HTTP API | 已提供 | 直接调用云端或本地 FastAPI 服务 |
| Python SDK | 已提供 | Python 应用通过 `MindMemOSClient` 调用 |
| CLI | 已提供 | Agent、脚本或 Plugin 执行 `mindmemos memory ...` |
| Agent Skill | 已提供通用 [`mindmemos-cli` Skill](https://github.com/mindscale-noah/MindMemOS/blob/main/skills/mindmemos-cli/SKILL.md) | 可安装到 OpenCode，指导 Agent 认证并按需调用 CLI |
| Agent Plugin | 已提供 OpenClaw 和 DeepSeek Harness Plugin | 自动召回并写回对话，但不能直接安装到 OpenCode |
| MCP Server | 仓库中尚未提供 | 需要开发 MCP 适配层 |
| OpenCode Plugin | 仓库中尚未提供 | 需要按照 OpenCode Hook 接口开发专用适配层 |

现有 [OpenClaw Plugin](https://github.com/mindscale-noah/MindMemOS/tree/main/plugins/openclaw-plugin) 和 [DeepSeek Harness Plugin](https://github.com/mindscale-noah/MindMemOS/tree/main/plugins/deepseek-harness-plugin) 都属于 CLI 上的薄适配层：在每个用户回合开始前执行 `memory search` 并注入结果，在回合成功结束后执行 `memory add`。这两份实现可以作为 OpenCode 集成的行为参考，但它们依赖各自 Host 的 Plugin API，不能直接复制到 OpenCode 的插件目录运行。

### A.3 通过 Skill 接入

**当前可以将通用 Skill 安装到 OpenCode 并按需调用。** 这是现阶段改动最少的方案，但不提供自动记忆生命周期。

MindMemOS 仓库中的 `skills/mindmemos-cli/` 符合以 `SKILL.md` 为入口的 Agent Skill 结构。安装 `mindmemos-sdk` 并完成认证后，可以将整个目录复制到 OpenCode 项目级 Skill 目录：

```bash
pipx install mindmemos-sdk
mindmemos auth

mkdir -p .opencode/skills
cp -R /path/to/MindMemOS/skills/mindmemos-cli .opencode/skills/
```

重新启动 OpenCode 后，该 Skill 可以指导模型通过现有 Bash Tool 执行：

```bash
mindmemos memory search "当前任务相关的项目约束" --top-k 5 --json
mindmemos memory add --content "本次任务形成的稳定结论" --async
```

该方式具有以下边界：

- Skill 只提供操作说明，不会在每个 Provider 请求前自动召回记忆；
- CLI 通过 Bash Tool 执行，权限控制落在 Bash 调用及其系统环境上；
- 是否搜索、写入什么内容仍由模型或用户显式决定；
- Skill 不会自动取得完整 Session，也不会自动排除临时信息、Tool output 或已注入的记忆。

因此，Skill 方案适合验证 MindMemOS 服务、执行显式记忆查询，以及由用户确认后写入少量稳定结论。它不适合在无人工检查的情况下自动保存每一轮完整对话。

### A.4 通过 MCP 接入

**OpenCode 支持 MCP，但 MindMemOS 仓库当前尚无现成的 MCP Server。** 使用该形式接入前，需要开发本地或远程 MCP 适配器，将 MindMemOS HTTP API、Python SDK 或 CLI 包装成 MCP Tools。

最小 MCP Server 可以只公开两个工具：

```text
memory_search(query, top_k, user_id, session_id)
memory_add(messages, user_id, session_id, mode)
```

需要人工维护记忆时，再增加 `memory_get`、`memory_update`、`memory_delete` 和 `memory_feedback`。删除、覆盖和跨用户检索应配置更严格的 Permission，避免将所有管理能力默认暴露给模型。

本地 stdio MCP Server 开发完成后，可以按以下形式连接 OpenCode；其中脚本路径只是适配器占位符，不是 MindMemOS 仓库当前已有文件：

```json
{
  "mcp": {
    "mindmemos": {
      "type": "local",
      "command": [
        "python",
        "/absolute/path/to/mindmemos_mcp_server.py"
      ],
      "enabled": true
    }
  }
}
```

MCP 方案会为模型提供具有明确名称和参数 Schema 的记忆工具，比通过 Bash 拼接 CLI 参数更稳定，也便于多个支持 MCP 的 Host 复用。然而，MCP Tool 仍由模型按需调用；仅配置 MCP Server 不会自动在每次请求前召回，也不会在 Session idle 时自动写回。若需要固定生命周期行为，还要结合 System/Skill 约束，或改用 OpenCode Plugin。

### A.5 通过 OpenCode Plugin 接入

**MindMemOS 仓库当前尚无可直接安装的 OpenCode Plugin。** 使用该形式接入前，需要按照 OpenCode Hook 接口开发专用适配层。该方案适合实现与 OpenClaw、DeepSeek Harness Plugin 相近的自动流程：

```text
新的用户请求
        ↓
Plugin 提取查询并调用 memory search
        ↓
将相关记忆临时加入本轮 Provider Messages
        ↓
OpenCode 正常执行 Agent Loop
        ↓
Session 进入 idle
        ↓
Plugin 取得本轮有效消息并调用 memory add
```

固定 OpenCode 源码提供了实现该流程所需的主要 Hook 和客户端能力：

- `experimental.chat.messages.transform` 可以在 Provider Request 前调整模型可见 Messages，适合注入检索结果；
- `experimental.chat.system.transform` 可以补充本轮 System，但检索内容通常更适合作为带来源标记的临时记忆 Context；
- 通用 `event` Hook 可以监听 `session.idle`；
- Plugin 获取的 OpenCode client 可以重新读取 Session 消息；
- `tool.execute.before` 和 `tool.execute.after` 可用于记录工具使用信息，但不应代替对完整回合的写入判断。

OpenCode Plugin 至少要处理以下问题：

1. 只使用真实用户输入构造召回查询，避免把旧 Tool output 和已注入记忆重复作为查询主体。
2. 为注入内容添加明确边界和来源标识，例如 `<relevant-memories>`，并将其视为外部数据而不是高优先级指令。
3. 只写入已完成回合，并排除系统 Prompt、Compaction 摘要、Plugin 自己注入的记忆以及不适合长期保存的敏感内容。
4. 使用稳定的 `user_id`、`app_id` 和 `session_id`，避免不同用户或项目之间发生记忆串扰。
5. 防止“检索结果再次写回”形成记忆回声，并为重复 idle 事件或重试设计幂等策略。
6. 将搜索失败和写入失败设计为可降级错误，避免记忆服务不可用时阻断普通 OpenCode 任务。

其中 `experimental.chat.messages.transform` 属于实验性 Hook，后续 OpenCode 版本可能调整接口。生产集成应固定 OpenCode 版本并增加升级测试。Plugin 运行在 OpenCode 进程中，能够读取对话并访问网络，其信任要求高于 Skill 和远程 MCP。

### A.6 接入方式选择

三种方式可以按照目标选择：

| 目标 | 推荐方式 | 原因 |
| --- | --- | --- |
| 先验证 MindMemOS，按需搜索或写入 | Skill + CLI | 可以直接使用仓库现有 `mindmemos-cli` Skill，改动最少 |
| 为多个 Agent Host 提供统一的显式记忆工具 | MCP | 工具名称和参数稳定，但需要先开发 MCP Server |
| 每轮自动召回，并在任务结束后自动写回 | OpenCode Plugin | 能进入 Context 组装和 Session 生命周期，但需要专用实现与完整测试 |
| 只在 OpenCode 中增加几个简单记忆函数 | Custom Tool | 比完整 Plugin 更轻，但不能自动监听 Session 生命周期 |

当前最稳妥的落地顺序是：先使用 Skill + CLI 验证服务、身份范围和记忆质量；确认 `search` 与 `add` 的数据边界后，再决定开发 MCP Server 还是 OpenCode Plugin。只有确实需要自动召回和自动写回时，才应引入 Plugin 生命周期集成。

### A.7 数据与安全边界

使用 MindMemOS 云服务时，检索查询和写入内容会发送到外部服务。使用本地部署时，数据首先进入本地 FastAPI 和数据库；如果聊天、Embedding 或 Rerank 路由仍指向云端模型，相关内容仍可能离开本机。

接入前应确认：

- 哪些 Session 内容允许成为长期记忆；
- API key 对应的 `project_id` 和用户范围；
- 是否需要对写入内容进行脱敏或人工确认；
- 如何删除错误、过时或用户要求遗忘的记忆；
- Plugin 或 MCP Server 发生重试时是否会重复写入；
- 记忆检索结果是否可能包含提示注入内容，以及注入后采用何种优先级。

MindMemOS 的 CLI 参数与操作说明见 [中文 CLI 文档](https://github.com/mindscale-noah/MindMemOS/blob/main/docs/cli/instruction_ZH.md)，完整项目能力、部署方式和官方集成状态以 [MindMemOS 仓库](https://github.com/mindscale-noah/MindMemOS) 为准。

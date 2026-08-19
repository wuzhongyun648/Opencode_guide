# Context Architecture：模型每一轮看见什么

上一篇：[07 Agent Loop](./07_Agent_Loop.md)
下一篇：[09 Tools 与 Permission](./09_Tools_and_Permission.md)

> 固定源码：`0e3474509aa5ad16afcf9c439785514d6443c6af`（`dev`）
> 本篇主线：当前默认 TUI 的普通消息路径，即兼容 Session API 与 `SessionPrompt` 编排。

## 1. 学习问题

继续使用同一个零基础学习场景：

> 请带我从零学习这个项目的 Harness 架构。先读取教程入口和项目规则，只做低风险观察，然后给出学习顺序。

模型没有打开编辑器，也不会天然记住项目目录。它为什么知道当前工作目录、教程规则、之前读过的 README，以及这一轮能够使用哪些工具？

本篇回答：**模型每一轮实际看见什么，这些信息从哪里来，又为什么不是把整个项目和全部历史一次性塞给模型。**

### 最短答案

模型只看见 Harness 为当前 Provider Request 组织出的信息，而不是直接看见项目、数据库或整个 Session。当前默认 OpenCode 请求可以理解为三条独立输入通道：系统级指令（System）、经过选择和转换的消息（Messages）和工具定义（Tool definitions）。Harness 在外层 Loop 准备新的 Provider 输入时，重新观察环境和规则、选择会话历史、物化可见工具，再在靠近 Provider 的边界完成请求组装。上下文工程（Context Engineering）的核心，就是让当前任务所需的信息在正确时机进入这三条通道，同时裁掉无关、过期或过大的内容。

本文所说的“下一轮重组”特指外层 Loop continuation 准备新的 `streamInput`。同一 `SessionProcessor` 内的 Retry attempt 虽然是新的 Provider Turn，却会复用原有 `streamInput`，不在重试前重载历史或重组 Context。

## 2. 最小心智模型：三条输入通道

阅读下图时，只问一个问题：一条信息最终以什么身份进入本轮请求？

```text
Provider Request
|
|-- System / privileged instructions
|     |-- Provider base 或 Agent prompt
|     |-- 当前环境
|     |-- 项目规则
|     |-- MCP instructions
|     |-- Skill guidance
|     `-- 本次输入附带的 system 内容
|
|-- Messages
|     |-- User / Assistant 的会话历史
|     |-- 过去的 Tool Call / Tool Result
|     `-- 当前用户输入
|
`-- Tool definitions
      |-- 工具名称与用途
      |-- 参数 schema
      `-- 本轮经过过滤后可供模型提议的能力
```

这三条通道有关联，但不能混为一谈：

- 项目规则通常作为 system-level 内容影响行为。
- “刚才 read 返回了什么”通常作为历史中的 Tool Result 出现。
- “read 接受哪些参数”属于独立 Tool definition。

## 3. 从 Prompt Engineering 到 Context Engineering

提示词工程（Prompt Engineering）主要关注怎样写一段直接指令，让一次模型调用产生更好的输出。

上下文工程（Context Engineering）关注的是更大的系统问题：

- 本轮目标是什么？
- 哪些项目规则适用？
- 前几轮得到了哪些新事实？
- 哪些历史仍然相关？
- 模型这轮可以提议哪些工具？
- 哪些信息太旧、太大或互相冲突？

在 Agent Loop 中，上下文不是启动时准备一次就结束。每次 Tool Result 都会改变下一轮应当看到的信息，因此 Harness 会持续执行：

```text
观察当前状态
-> 选择并组织信息
-> 请求模型
-> 获得文本或工具结果
-> 更新状态
-> 为下一轮重新组织信息
```

模型能力决定它在理想输入下能够做什么；上下文工程决定真实运行中有多少能力能被稳定调用出来。

## 4. 模型看不见什么

在列举来源前，先打破几个直觉误区。

模型不会天然看见：

- 工作区里的所有文件；
- SQLite 中保存的全部 Message、Part 和 Event；
- TUI 当前显示的所有界面状态；
- Tool executor 的内部实现和操作系统全部权限；
- 上一次模型调用结束后仍在某个“脑内”持续存在的记忆；
- 没有被 Harness 选入请求的旧工具输出或项目规则。

如果模型要知道 README 写了什么，通常需要由 `read` Tool 把内容变成 Tool Result。如果它下一轮仍能引用该内容，是因为结果进入了可见历史，而不是模型在会话外获得了长期记忆。

## 5. 当前默认请求怎样组装

[第 07 篇](./07_Agent_Loop.md)已经说明，每次 continuation 都会回到 Loop 顶部重新读取状态。当前 `SessionPrompt.run` 随后集中获取五类结果：

```text
Skill guidance
Environment
Project Instructions
MCP Instructions
Session History 转换后的 Model Messages
```

这些结果进入 LLM 层前，system-level 内容按下面的稳定顺序组织：

```text
Environment
-> Project Instructions
-> MCP Instructions（如果有）
-> Skill guidance（如果有）
```

靠近 Provider 的 `LLMRequestPrep.prepare` 再把更外层指令放到前后：

```text
Provider-family base instructions，或 Agent prompt（二选一）
-> Environment
-> Project Instructions
-> MCP Instructions
-> Skill guidance
-> Structured-output policy（如果启用）
-> per-prompt user.system（如果有）
-> experimental.chat.system.transform
```

这里有两个重要限定：

1. Agent 有自定义 prompt 时，它替代 Provider-family base instructions，不是简单叠加在后面。
2. `experimental.chat.system.transform` 可以修改整个 system 数组，因此 hook 之后不再保证仍保持前面的拼接形态。

Session History 不在这条字符串拼接链里；Tool definitions 也不在。

## 6. System：这一轮应当遵守什么

System 或 privileged instructions 为模型提供角色、环境和约束。当前默认路径主要有以下来源。

### 6.1 Provider base 或 Agent prompt

Provider base instructions 处理模型家族相关的基础行为。若当前 Agent 配置了自己的 prompt，则使用 Agent prompt。

Agent 并不只是 prompt；它还包含 Model 偏好、Permission、运行参数等。这里我们只关心它对本轮 System 内容的贡献。Agent 的完整专业化机制由[第 11 篇](./11_Agent_Specialization_and_Collaboration.md)主讲。

### 6.2 Environment

Environment 是外层 Loop 每次准备新输入时重新观察的运行信息，可能包含当前工作目录、项目或工作树、平台、日期和模型身份等。

它不是一条永久写入 Session History 的普通聊天消息。环境发生变化时，下一轮重新观察到的内容也可能变化。

这解释了为什么模型能说出“当前位于哪个项目”，但不能据此推断它扫描了目录中所有文件。

### 6.3 Project Instructions

项目规则来自 OpenCode 发现或配置的 instruction sources，例如适用的 `AGENTS.md`、`CLAUDE.md`、`CONTEXT.md`、配置路径或远程指令来源。

当前默认路径会在每轮调用 `Instruction.system()` 获取 ambient Project Instructions。它们的来源文件可以长期存在，但本轮拼接出来的字符串不是一条普通 Session Message。

对学习者来说，可以把它理解为：

```text
项目规则文件长期存在
        |
        v
Harness 本轮发现并读取适用规则
        |
        v
作为 system-level 内容发送给模型
```

### 6.4 MCP Instructions

已连接 MCP Server 可以提供相关 instructions。当前实现只为至少存在一个未被 deny 的相关工具的 server 保留这部分内容。

MCP instructions 告诉模型怎样理解这组外部能力；MCP Tool 本身仍作为独立 Tool definition 进入请求。

### 6.5 Skill guidance

当前 system-level Skill guidance 主要列出可用 Skill 的名称和描述。完整 Skill 内容通常由模型调用 `skill` Tool 后加载，加载结果再以 Tool Result 进入历史。

因此，“模型知道有一项 Skill”和“模型已经读取 Skill 全文”是两个阶段。

### 6.6 本次输入的 system 内容

单次 Prompt 可以附带 `user.system`。它位于当前组装顺序的后部，并保存在 User Message 的字段中，只对由该最新 User Message 驱动的相关 turns 生效。

## 7. 两种项目规则进入方式

学习项目规则时，有一条很有用的细分：ambient rules 与附近文件规则可能通过不同路径进入模型视野。

### 7.1 每轮注入的 ambient instructions

`Instruction.system()` 负责本轮整体适用的项目指令。模型无需先调用 `read` 才能接收这部分 system-level 内容。

### 7.2 Read Tool 发现的附近规则

当 `read` 读取某个文件时，`Instruction.resolve` 还会向上发现附近的规则文件，并把内容包装为 `<system-reminder>` 追加到该 Tool Result。

这部分以后会随 completed Tool Part 出现在 Session History 中，而不是成为每轮重新计算的 ambient System 字符串。

在贯穿场景里，模型读取教程子目录中的文件时，可能同时收到该目录层级适用的规则提醒。这让规则与具体文件位置绑定，但也要求读者区分：

```text
环境级 Project Instructions -> System

Read 附带的附近规则提醒 -> Tool Result -> Messages
```

两者内容可能相关，进入请求的路径却不同。

## 8. Messages：到目前为止发生了什么

Messages 是 User、Assistant 与 Tool 交互的会话表示。一般情况下它遵循交互顺序；完成 Compaction 后，active history 会重组为 marker、summary、保留尾部和后续消息，不再是数据库记录的严格时间排序。它承担的是“当前对话到这里发生了什么”，而不是“系统长期规则是什么”。

在零基础学习场景中，Messages 可能逐渐变成：

```text
User：请从零带我学习 Harness，只读取和解释

Assistant：Tool Call -> read README
Tool Result：README 给出 06-12 阅读顺序

Assistant：Tool Call -> read AGENTS.md
Tool Result：适用的编辑与测试规则

Assistant：综合入口与规则，给出学习路线
```

下一次 Provider Turn 能够利用 README，是因为 Tool Call 与 Tool Result 被转换为本轮模型可见的 Messages。

## 9. Session History 不是数据库全文直送

当前 Loop 会从持久化投影中重新读取 Message 与 Part，但 `MessageV2.toModelMessagesEffect` 还会把领域状态转换成适合当前 Provider 的 Messages。

转换过程中可能发生：

- 跳过没有 Parts 的 Message；
- 忽略被标记为 `ignored` 或为空的 User text；
- 把文本文件或目录附件的 synthetic read 内容转换为文本；
- 把已经 Prune 的旧 Tool output 替换成占位符；
- 对不同 Model 的 reasoning 做兼容性降级，并去掉不兼容 metadata；
- 把没有完成结算的 Tool 状态降低为 interrupted error，避免悬空调用；
- 使用最新已完成 Compaction 的 summary 和近期保留尾部。

所以需要区分两层：

```text
Durable Session History
    = 系统保存的 Message / Part 领域状态

Provider-visible Messages
    = 针对本轮请求选择并转换后的模型表示
```

“保存了”不等于“本轮看见了”。反过来，Environment 等每轮动态信息可以被模型看见，却不一定作为普通 Message 保存。

Session 与 Persistence 的内部结构由[第 10 篇](./10_Session_and_Persistence.md)继续解释。

## 10. Tool definitions：模型认为自己可以做什么

Tool definitions 是 Provider Request 中独立于 System 和 Messages 的字段。每项通常描述：

- 工具名称；
- 工具用途和使用条件；
- 参数 schema；
- Provider 需要的调用契约。

工具描述本身也是上下文工程。若名称模糊、用途重叠或参数说明不清，即使 executor 实现完全正确，模型也可能选错工具或生成错误参数。

### 10.1 定义、历史与执行是三件事

```text
Tool definition：本轮可以提议什么

Tool Call / Tool Result in Messages：过去提议和发生了什么

Tool executor：真正怎样读取文件或执行动作
```

历史里曾出现 `read` Tool Result，不表示本轮一定仍向模型提供 `read` schema。反过来，本轮看见 `read` schema，也不表示它已经执行，更不表示具体资源 Permission 已经通过。

当前最终 Tool visibility 会受 Agent permission、Session permission 和本次 Prompt 的 tool override 影响。完整生命周期和安全边界见[第 09 篇](./09_Tools_and_Permission.md)。

## 11. 上下文怎样在每轮重新组装

把第 07 篇的 Loop 与本篇三条通道合起来，可以得到：

```text
Loop 顶部
|
|-- 重载并选择 durable Session History
|-- 重新观察 Environment
|-- 重新获取适用 Project/MCP/Skill instructions
|-- 按当前 Agent/Session/Prompt 物化 Tools
|-- 创建 Provider-visible System + Messages + Tool definitions
|
v
Provider Turn
|
|-- Text：成为 Assistant 输出
`-- Tool Call：执行后产生新的 Tool Result
                  |
                  v
              下一轮重组
```

这种设计允许新观察改变后续上下文。例如 README 告诉模型真正入口是 `Opencode_Harness/06_Harness.md`，下一轮模型就可以读取它，而不必在最初请求中预先塞入全部教程内容。

它也意味着每轮请求不是完全相同的副本。环境、项目规则、MCP 连接、Skill 可见性、Session History 和 Tool catalog 都可能变化。

## 12. 上下文质量的四类问题

Context Engineering 不只是“尽可能多放信息”。过多信息同样会降低任务质量。

- **缺失**：没有 README 或适用规则，模型可能从错误入口开始。
- **无关**：整个仓库和全部日志会挤占窗口、分散注意力。
- **过期**：旧路径或状态可能已经变化；每轮重算部分来源只能降低风险，不能自动判断所有历史事实是否过期。
- **冲突**：不同来源可能给出不同指令；Harness 组织输入，模型解释层级和范围，而安全边界仍需 Permission 等确定性机制落实。

## 13. Context 过大时发生什么

模型有有限的上下文窗口。Session 变长、读取文件增多或 Tool output 很大时，OpenCode 需要改变未来模型可见的历史。

本篇只解释“模型看见的内容怎样变化”；保存与恢复细节放在第 10 篇。

### 13.1 Compaction：摘要加近期尾部

当前 Compaction 会生成一份 summary，并选择需要逐字保留的近期尾部。之后的 Provider-visible history 近似为：

```text
Compaction marker
-> summary
-> retained recent tail
-> Compaction 后的新消息
```

对未来模型而言：

- 摘要保留了被选中的目标、约束和工作状态；
- 近期尾部继续逐字可见；
- 没进入摘要的旧细节可能不再可见；
- 被截断或未选中的原始内容不会因为“数据库还在”就自动回到请求。

Compaction 是有损的 active-history 替换，不是模型获得了完美长期记忆。

### 13.2 Pruning：隐藏旧的大型 Tool output

Pruning 不生成摘要。它把选中的旧 Tool Result 在模型表示中替换为：

```text
[Old tool result content cleared]
```

它主要减少旧的大型工具输出对窗口的占用。Skill Tool 等内容有额外保护规则，但不要把 Pruning 理解为“总结了旧结果”。

### 13.3 它们不是什么

Compaction 与 Pruning 都不是代码工作树 Snapshot，也不负责撤销文件。代码 Snapshot、Revert 和 Session 持久化属于其他问题。

如果禁用自动 Compaction，当前旧路径的本地 token threshold 不会触发自动压缩；Provider Context Overflow 会记录错误并停止，而不是偷偷继续压缩。

## 14. 贯穿场景：三轮里模型分别看见什么

现在完整回答学习任务。

| 轮次 | 本轮新增的关键信息 | 可以据此做什么 |
| --- | --- | --- |
| 第一轮 | System 中的 Agent/provider 指令、Environment、项目规则；Messages 中的学习目标；独立 Tools | 决定先读取哪个入口，但尚未天然看见 README 正文 |
| 第二轮 | Messages 中新增 `read` Tool Call、README Tool Result，以及可能附带的附近规则提醒 | 根据真实目录导航，不再只凭预训练知识猜测 |
| 第三轮 | 再加入规则文件的读取结果，同时重新组装 System 与 Tools | 综合目标、系列结构和行为约束，给出学习顺序 |

如果信息已经足够，模型输出最终说明，不再提出 Tool Call；Loop 在下一次顶部检查停止。

整个过程中，Context Architecture 的作用不是替模型完成判断，而是让每次判断都有与当前阶段相匹配的信息环境。

## 15. 动手观察：做一次上下文增量实验

在熟悉的教程目录中开启新 Session，使用只读请求：

> 我正在从零学习 OpenCode Harness。请只读取 README 和适用的项目规则，说明 06、07、08 的学习顺序。不要修改文件，不运行 Shell，不访问外部服务。每次引用结论时说明它来自哪个已读取文件。

可以分两步观察：

1. 第一次读取完成后，检查 Tool Result 中实际返回了什么，以及是否附带附近规则提醒。
2. 最终回答中，检查模型是否只引用了已进入历史的文件，而不是声称看过整个项目。

随后追加一句：

> 现在根据刚才读到的规则，解释为什么 07 应该先于 08；不需要重新读取没有变化的文件。

这次追加请求用于观察 Session History 的作用：模型可以利用前面已结算的 Tool Result；新用户输入启动的外层 Loop 也会重新观察环境、规则与工具可见性。

需要注意：让模型说明“自己看到了什么”只是教学辅助，不是严格审计证据。真正核对时应查看 Tool Call/Result、Session 记录和请求构造源码。

## 16. 常见误解

- **“上下文就是用户 Prompt”**：Prompt 只是一部分；System、历史、Tool Result、Tool definitions 和环境同样影响判断。
- **“模型已经看见整个仓库”**：只有被注入、读取并保留在可见表示中的内容才进入请求。
- **“数据库保存了，所以模型一定看见”**：历史还会经历 Compaction、Pruning、状态转换和 Provider 兼容处理。
- **“Tool schema 是 System Prompt 的一段文字”**：请求中 Tool definitions 是独立字段，过去的 Tool Call/Result 则属于 Messages。
- **“项目规则只会通过 System 出现”**：ambient instructions 可进入 System，Read 发现的附近规则也可随 Tool Result 进入 Messages。
- **“Compaction 是无损长期记忆”**：Summary 会遗漏细节，它改变的是未来模型活跃可见的历史表示。
- **“Context Snapshot 就是代码快照”**：native V2 的 Context Snapshot 记录 typed Context Source 的上次 admitted value；代码工作树 Snapshot 用于文件 diff/restore。

## 17. 关于 native V2 的一句版本说明

当前默认路径把 Environment、Project Instructions、MCP 与 Skill guidance 作为每轮重新观察和拼接的字符串来源。固定源码中的 native V2 另有 typed Context Source、Context Snapshot 和 Context Epoch，用于表达稳定 baseline 与时间序更新；但它尚未接管默认 TUI，旧来源的能力覆盖也不能直接视为已完整迁移。

本篇不展开 Context Epoch 的 admission、replacement 和 parity 表，避免把架构演进混入当前学习主线。

## 18. 本篇掌握要点

读完后，应能用自己的话回答：

1. 模型每轮看到的是 Harness 构造的 Provider Request，不是整个项目或数据库。
2. 当前请求有三个主要输入边界：System、Provider-visible Messages、Tool definitions。
3. 外层 Loop 准备新输入时，Environment、Project Instructions、MCP Instructions 与 Skill guidance 会按当前状态重新获取，历史则从 Session 投影选择并转换；Retry 不经过这一重组边界。
4. Read 发现的附近规则可以随 Tool Result 进入 Messages，不等同于 ambient System instructions。
5. Durable history 与 Provider-visible Messages 是两层；保存不等于本轮可见。
6. Tool definition 描述可提议能力，历史中的 Tool Call/Result 描述过去行为，executor 才执行真实操作。
7. Compaction 用摘要和近期尾部替换活跃历史表示，Pruning 隐藏旧的大型 Tool output；两者都可能让模型失去旧细节。

## 19. 关键源码入口

以下路径均以固定源码 `0e3474509aa5ad16afcf9c439785514d6443c6af` 为基线：

| 主题 | 文件 | 函数或导出符号 |
| --- | --- | --- |
| 每轮 Context 与 Messages 组装 | `packages/opencode/src/session/prompt.ts` | `SessionPrompt.run` |
| 最终 Provider Request | `packages/opencode/src/session/llm/request.ts` | `LLMRequestPrep.prepare`、`resolveTools` |
| Environment、Skill 与 MCP 内容 | `packages/opencode/src/session/system.ts` | `SystemPrompt.provider`、`environment`、`skills`、`mcp` |
| Project Instructions | `packages/opencode/src/session/instruction.ts` | `Instruction.systemPaths`、`system`、`resolve` |
| Read 附带附近规则 | `packages/opencode/src/tool/read.ts` | `ReadTool` executor、`Instruction.resolve` 调用点 |
| Session History 转换 | `packages/opencode/src/session/message-v2.ts` | `toModelMessagesEffect`、`filterCompacted` |
| Tool 物化 | `packages/opencode/src/session/tools.ts` | `SessionTools.resolve` |
| Compaction 与 Pruning | `packages/opencode/src/session/compaction.ts` | `SessionCompaction.Service`、`process`、`prune` |

可用于核对行为的代表性测试：

- `packages/opencode/test/session/prompt.test.ts`：MCP instructions、Tool continuation、运行中新 Prompt。
- `packages/opencode/test/session/system.test.ts`：Provider prompt、Skill guidance 与 MCP instructions。
- `packages/opencode/test/session/compaction.test.ts`：summary、retained tail、Pruning 与 `auto=false`。
- `packages/opencode/test/session/revert-compact.test.ts`：Compaction 与 Revert 的边界。

---

上一篇：[07 Agent Loop：一次请求为什么可以持续多轮](./07_Agent_Loop.md)
下一篇：[09 Tools 与 Permission：意图怎样变成真实操作](./09_Tools_and_Permission.md)

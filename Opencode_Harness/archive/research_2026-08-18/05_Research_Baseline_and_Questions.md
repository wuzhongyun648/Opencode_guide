# 任务 1：研究基线与核心问题

状态：已完成。

核对日期：2026-08-18。

## 1. 固定源码基线

| 项目 | 固定值 |
| --- | --- |
| 仓库 | `/home/wuzhongyun/projects/Intern_projects/Opencode_learn/opencode github code` |
| 分支 | `dev` |
| Commit | `0e3474509aa5ad16afcf9c439785514d6443c6af` |
| Git describe | `github-v1.2.25-1693-g0e3474509a` |
| `packages/opencode` 版本 | `1.18.18` |
| 初始工作树 | 干净，跟踪 `origin/dev` |

本轮任务 2 以及后续模块研究都以该 commit 为准。源码更新后不能继续沿用旧行号和旧结论，必须记录受影响模块并重新核对。

## 2. “当前运行时”的操作性定义

本研究中的“当前运行时”不是指仓库里最新命名的类型或目录，而是：

> 在固定 commit 上，从默认交互式 TUI 提交一条普通消息时，实际被入口代码调用的执行路径。

静态追踪已经确认：当前 TUI 普通消息使用兼容 Session API 和 `SessionPrompt` 旧应用运行时，而不是 native V2 Session prompt API。

```text
当前 TUI 路径：
sdk.client.session.prompt(...)
-> POST /session/{sessionID}/message
-> SessionHttpApi.prompt
-> SessionPrompt.prompt
-> SessionPrompt.loop

已实现但当前 TUI 未使用的 native V2 路径：
sdk.client.v2.session.prompt(...)
-> POST /api/session/{sessionID}/prompt
-> SessionV2.prompt
-> SessionInput.admit
-> 当 resume !== false 时调用 SessionExecution.wake
```

`resume: false` 只完成 durable admission，不唤醒执行器。

这一区分是后续 V1/V2 比较的基线。文件名包含 `v2`、代码使用 Effect 或调用新 Core 基础设施，都不能单独证明 native V2 Session Runner 已经接管当前 TUI。

### 2.1 当前 TUI 路径的源码依据

- 文件：`packages/tui/src/component/prompt/index.tsx`
- 函数：`submitInner()` 的普通消息分支
- 行号：1092-1146
- 版本：`0e3474509a`

- 文件：`packages/sdk/js/src/v2/gen/sdk.gen.ts`
- 方法：`Session2.prompt()`
- 行号：3737-3795
- 版本：`0e3474509a`

- 文件：`packages/opencode/src/server/routes/instance/httpapi/handlers/session.ts`
- 函数：`SessionHttpApi.prompt`
- 行号：295-309
- 版本：`0e3474509a`

- 文件：`packages/opencode/src/session/prompt.ts`
- 函数：`SessionPrompt.prompt`、`SessionPrompt.loop`
- 行号：1052-1071、1343-1347
- 版本：`0e3474509a`

### 2.2 native V2 路径的源码依据

- 文件：`packages/sdk/js/src/v2/gen/sdk.gen.ts`
- 方法：`Session3.prompt()`
- 行号：5617-5656
- 版本：`0e3474509a`

- 文件：`packages/server/src/handlers/session.ts`
- 符号：`SessionHandler` 的 `session.prompt` Handler
- 行号：139-171
- 版本：`0e3474509a`

- 文件：`packages/core/src/session.ts`
- 函数：`V2Session.prompt`
- 行号：360-386
- 版本：`0e3474509a`

## 3. 研究范围

本阶段只研究 OpenCode 内部 Agent Harness，包括：

- 当前实际入口和一次完整请求链。
- Agent、Model、Context、Tools 和 Orchestration。
- Session、Message、Part、事件和持久化。
- Tool Permission、Retry、Interrupt、Compaction 和终止条件。
- Client、Server、Provider 和事件返回边界。
- V1 与 V2 的现状、兼容关系和分模块改进。

不研究广义 Harness Engineering，不展开代码仓库如何为所有 Agent 提供文档、CI/CD 和团队治理体系。

## 4. 核心研究问题

### Q1：当前默认入口到底调用哪套 Session Runtime？

需要从 TUI 调用点、SDK 方法、HTTP 路由、Server Handler 和 Session Service 连续证明，不能根据 package 名推断。

### Q2：一条用户输入怎样变成持久化的 User Message 和 Parts？

需要解释文件、文本、MCP Resource、Agent mention 等输入如何解析，以及 Message 和 Part 的写入是否属于同一事务。

### Q3：一次用户请求为什么可能触发多个 Provider Turn？

需要解释外层 Session Loop、Tool Call、Tool Result、历史重载、Compaction 和最终停止条件。

### Q4：每个 Provider Turn 的 Agent 和 Model 如何确定？

需要说明显式输入、Agent 默认值、Session 当前值、历史消息和 Provider 默认模型之间的优先级。

### Q5：模型每一轮实际看到哪些 Context？

需要区分 Provider base prompt、Agent prompt、环境、项目指令、Skill guidance、MCP instructions、Session History、Tool schema 和 per-prompt system input。

### Q6：Tool 如何从定义变成一次真实执行？

需要追踪 Registry、Tool materialization、Provider tool schema、Tool Call、Permission、Plugin Hook、执行结果、Truncation 和下一轮 Context。

### Q7：哪些状态是 durable，哪些只是 live 或 process-local？

需要区分 SQLite Message/Part/Event、文件系统 Snapshot、Server Runner/Status、流式 delta 和 TUI reactive store。

### Q8：任务何时继续、停止、重试、压缩或中断？

需要说明 Provider finish reason、Tool Part、Processor result、Retry policy、Context overflow、Permission rejection 和用户 cancel。

### Q9：客户端如何实时看到 Text、Reasoning 和 Tool 状态？

需要区分 Prompt POST 的最终响应、默认本地 TUI 的 Worker RPC 事件路径、远程 TUI 的 `/global/event` SSE 路径，以及断线后的状态恢复。

### Q10：V2 相对当前旧运行时具体改进了什么？

必须按 Session、Context、Tools、Permission、Persistence 和 Client/Server 分别比较，并同时记录 V2 已完成、部分完成和缺失的能力。

## 5. 每个问题的证据要求

每个问题至少需要：

1. 一个实际入口或上层调用点。
2. 一个核心实现函数、方法或导出符号。
3. 一个测试或可复现实验。
4. 一个 V1/V2 对照点。
5. 文件路径、函数或符号、行号和 commit。
6. 对失败、取消、并发或兼容边界的说明。

## 6. 当前已经确认的关键边界

| 结论 | 状态 |
| --- | --- |
| 当前普通 TUI 消息走 `client.session.prompt` | `[Current default @ 0e3474509a]` |
| 当前普通 TUI 消息进入 `SessionPrompt` 旧应用运行时 | `[Current default @ 0e3474509a]` |
| native V2 `/api/session/:id/prompt` 已经接入 executable Server | `[V2 implemented @ 0e3474509a]` |
| 当前 TUI 普通提交没有调用 native V2 prompt API | `[Current default @ 0e3474509a]` |
| 旧运行时的 Message/Part 写入复用了 EventV2 和 Core Projector | `[Current compatibility @ 0e3474509a]` |
| 当前普通请求的多轮 Tool Loop 由 `SessionPrompt.run` 外层循环控制 | `[Current default @ 0e3474509a]` |
| native V2 parity 仍存在 `partial` 和 `missing` 项 | `[V2 partial @ 0e3474509a]` |

## 7. 任务 1 输出结论

任务 1 已完成以下约束：

- 固定源码版本。
- 固定“当前运行时”的判断方法。
- 固定 06 只研究内部 Agent Harness。
- 固定 10 个核心问题。
- 固定每个板块的源码和测试要求。
- 固定 V1/V2 不能按目录名或产品版本号简单判断。

下一份研究文档从 Q1 开始，给出当前运行时的一次端到端请求链。

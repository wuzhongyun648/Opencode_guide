# Harness 系列模块地图、源码入口与资料清单

## 1. 总分结构

| 正式文档 | 定位 |
| --- | --- |
| `06_Harness.md` | Harness 总览、模块关系、完整执行链和专题索引 |
| `07_OpenCode_Runtime_Terminology.md` | `CONTEXT.md` 术语、关系和实现状态注释 |
| `08_Agent_and_Orchestration.md` | Agent、主循环、Todo、Task、Subagent 和控制流 |
| `09_Context_and_Persistence.md` | Context、Session History、Message/Part、持久化和 Compaction |
| `10_Tools_and_Security.md` | Tool Registry、Permission、Tool lifecycle 和安全边界 |
| `11_Runtime_Boundary.md` | Client、Server、Provider、事件、SDK 和内部运行边界 |
| `12_V1_V2_Comparison.md` | V1/V2 分模块比较、迁移状态和未完成能力 |

专题编号可以在研究后调整，但 06 保持总览定位。

## 2. 建议研究模块

| 模块 | 主要问题 | 旧运行时候选入口 | V2 候选入口 | 最低验证要求 |
| --- | --- | --- | --- | --- |
| Harness 总览 | OpenCode 如何把模型、工具、状态和客户端组成 Agent | `packages/opencode/src/` | `packages/core`、`protocol`、`server`、`client` | 一张有源码映射的总体图 |
| Agent 与 Model | Agent 配置如何影响模型、Prompt、工具和权限 | `packages/opencode/src/agent/`、`session/llm.ts` | Core Agent/Model resolver 和 Catalog 相关模块 | Agent 选择到 Provider Request 的调用链 |
| Orchestration | 一次输入如何形成多轮模型与工具循环 | `session/prompt.ts`、`processor.ts`、`run-state.ts` | `session/runner/llm.ts`、`run-coordinator.ts`、`execution/local.ts` | 一个完整 Provider Turn 和 continuation 流程 |
| Context | 每次模型请求包含什么，何时更新 | `session/system.ts`、`instruction.ts`、`llm/request.ts` | `system-context/`、`session/context-epoch.ts`、`history.ts` | 一次请求的 Context 构成和优先边界 |
| Session 与 Persistence | Message、Part、事件和状态如何保存 | `session/session.ts`、`message-v2.ts` | `event/sql.ts`、`session/projector.ts`、`session/sql.ts` | 数据模型、写入时机和恢复语义 |
| Compaction 与 Snapshot | Context 过长或需要撤销时怎样处理 | `session/compaction.ts`、Snapshot/Revert 相关代码 | V2 compaction、Context Epoch 和 checkpoint 相关代码 | 压缩前后模型可见内容与持久记录 |
| Tools 与 Permission | 工具如何注册、展示、授权、执行和返回结果 | `tool/registry.ts`、`session/tools.ts`、`permission/` | `core/src/tool/`、`core/src/permission.ts` | 一个 Tool 从定义到 settlement 的完整流程 |
| Todo 与 Subagent | 长任务和委派怎样组织 | Todo 和 `tool/task.ts` 相关代码 | V2 Todo、Task 或后台执行相关模块 | 父子 Session、权限继承和结果回传 |
| Runtime Boundary | TUI、Web、SDK 如何连接 Harness | `packages/opencode/src/server/`、旧 SDK | `protocol`、`server`、`client`、`sdk-next` | 请求入口和事件返回路径 |
| Reliability | Retry、Interrupt、Doom Loop、Recovery 怎样限制失控 | Session retry、processor、permission 相关代码 | Runner、Coordinator、durable inbox 和 event replay | 明确已实现与 deferred 的恢复能力 |
| V1 到 V2 | V2 为什么重构、具体改了什么、尚缺什么 | 以上旧路径和 V1 Schema | 以上 V2 路径及 `specs/v2/` | 按模块形成 parity 表，不做笼统优劣判断 |

以上路径只是研究入口，不是预先确认的最终结论。正式引用需要继续定位文件位置、所在函数或方法、行号、调用方和测试。

## 3. V1/V2 的首轮比较重点

### Session 和执行

- V1 的 Prompt 创建、Loop 和流处理如何耦合。
- V2 如何分离 durable prompt admission、Prompt Promotion 和 Session execution。
- V2 的 Session Drain 为什么是进程内状态，而不是持久化实体。
- V2 对并发 wake、resume、interrupt 和 crash recovery 的边界。

### Context

- V1 如何拼接 Provider Prompt、环境信息、项目指令和历史。
- V2 如何引入 System Context、Context Source、Context Snapshot 和 Context Epoch。
- V2 parity 表中哪些 Context 能力仍为 `partial` 或 `missing`。

### Tools 和 Permission

- V1 如何动态收集 built-in、custom、plugin 和 MCP tools。
- V2 如何使用 typed registry、Location scope、durable settlement 和 output bounding。
- 两者如何判断 Permission；Permission 与 Sandbox 分别负责什么。

### Persistence

- V1 的 `session`、`message`、`part` 数据如何保存。
- V2 的 durable event、projection、`session_input` 和 Context Epoch 如何保存。
- 当前迁移为什么同时保留旧表和新的 projection。

### Server 和 Client

- 旧应用、旧 SDK 和兼容事件如何继续工作。
- `Protocol -> Server -> Client -> sdk-next` 如何形成新的合同边界。
- 当前 executable 如何同时组合旧路由与新 API。

## 4. 官方与本地资料

| ID | 资料 | 用途 | 权威边界 |
| --- | --- | --- | --- |
| OFF-01 | 本地 `CONTEXT.md` | V2 Session Runtime 术语与关系 | 设计语言，不等于全部已发布行为 |
| OFF-02 | `specs/v2/session.md` | V2 Session、Compaction、Tool 和 parity 状态 | 包含 implemented、partial、missing 和 follow-up |
| OFF-03 | `specs/v2/config.md` | 配置迁移目标 | 需要与当前实现核对 |
| OFF-04 | `specs/v2/tools.md` | V2 Tool Registry 与 Tool Output 设计 | 需要区分当前 slice 和规划 |
| OFF-05 | `specs/v2/todo.md` | V2 实现进度和待办 | TODO 可能随代码演化而过时 |
| OFF-06 | `packages/schema/AGENTS.md` | Current/V1/V2 命名和合同边界 | 仓库内部维护约束 |
| OFF-07 | OpenCode 官方 Agents、Tools、Permissions、Server、SDK 文档 | 用户可见行为 | 不完整描述内部实现 |

## 5. Google 白皮书与用户笔记

| ID | 资料 | 用途 |
| --- | --- | --- |
| GEN-01 | Google, *Introduction to Agents*, Updated May 2026 | Model、Tools、Orchestration、Deployment、Agent Ops 的通用框架 |
| GEN-02 | `/home/wuzhongyun/projects/Intern_projects/Day 1 - Introduction to Agents 阅读笔记.md` | 用户当前理解、重点和疑问的基线 |

Google 白皮书用于提出问题和组织模块，不能直接证明 OpenCode 的实现。

## 6. 五篇知乎文章

五篇文章作者均显示为 Link，发布时间均为 2026-04-20。直接访问可能返回 403，已确认可以通过公开 Reader 方式读取全文。

| ID | 标题 | URL | 主要用途 |
| --- | --- | --- | --- |
| ZH-01 | Harness Engineering：OpenCode 架构设计全景 | https://zhuanlan.zhihu.com/p/2029578046949061893 | 建立待验证的四支柱框架 |
| ZH-02 | OpenCode 结构化执行详解 | https://zhuanlan.zhihu.com/p/2029578934690363000 | Todo、Plan、Task、Retry、Doom Loop 线索 |
| ZH-03 | OpenCode 持久化记忆机制详解 | https://zhuanlan.zhihu.com/p/2029578842747024899 | Session、Message、Part、Compaction 线索 |
| ZH-04 | OpenCode Agent 专业化机制详解 | https://zhuanlan.zhihu.com/p/2029578730553581715 | Agent、Subagent 和 Permission 线索 |
| ZH-05 | OpenCode 上下文架构详解 | https://zhuanlan.zhihu.com/p/2029578561611134622 | Prompt、Instruction、Skill 和裁剪线索 |

这五篇文章属于同一来源系列。文章中的代码路径、阈值、默认行为和架构判断都要针对固定 commit 重新验证。

## 7. 辅助资料

| ID | 资料 | 用途 | 限制 |
| --- | --- | --- | --- |
| AUX-01 | DeepWiki OpenCode Architecture | 快速定位源码和模块关系 | AI 生成并固定在特定 commit |
| AUX-02 | Microsoft Agent Harness | 定义 Runtime Harness 的通用组成 | 不是 OpenCode 实现 |
| AUX-03 | OpenAI, *Unrolling the Codex agent loop* | 对照生产级 Coding Agent Loop | 不是 OpenCode 实现 |
| AUX-04 | SWE-agent ACI 论文 | 理解工具接口和 observation 设计 | 研究对象不同 |

## 8. 已完成的阶段研究

- `05_Research_Baseline_and_Questions.md`：固定源码基线、当前运行时定义和核心问题。
- `06_Current_Runtime_End_to_End_Trace.md`：追踪默认 TUI 普通消息的当前完整请求链。
- `10_Research_Agent_and_Orchestration.md`：Agent、Model、主循环、Todo、Task、Subagent 和可靠性控制的双轨研究初稿。
- `11_Research_Context_and_Persistence.md`：Context、Session History、持久化、Compaction、Snapshot 和恢复的双轨研究初稿。
- `12_Research_Tools_and_Security.md`：Tool 注册、物化、Permission、执行、结算和安全边界的双轨研究初稿。
- `13_Research_Runtime_Boundary.md`：Client、Server、Provider、事件与新旧 API 接线边界的双轨研究初稿。

上述四份模块笔记已完成任务 3-5 的首轮静态研究和任务 7 的首轮最小验证，但仍需任务 6 的交叉审计。任务 8 按用户指示跳过且未作理解验收；任务 9-10 的正式写作不应把这一步倒推为已通过。未被首轮实验覆盖的问题继续保留为 Open Questions。

## 9. 后续待建立的研究笔记

完成交叉审计和问题汇总后，再创建：

- `14_Research_V1_V2_Comparison.md`
- `15_Open_Questions.md`

这些文件在对应研究任务正式开始时创建，避免预填未经验证的结论。

# OpenCode Harness 研究章程

## 1. 研究目标

本阶段的首要目标是理解 OpenCode 如何把语言模型组织成可以读取项目、调用工具、维护状态并持续执行任务的编码 Agent。文档是理解过程的记录和检验工具，不以快速填满章节为目标。

最终正式产物采用总分结构：

- `06_Harness.md`：总览 OpenCode 内部 Agent Harness 的边界、模块和一次完整执行链，并索引后续专题。
- `07_OpenCode_Runtime_Terminology.md`：以官方 `CONTEXT.md` 为基础整理 OpenCode Session Runtime 的术语、定义和关系。
- `08_Agent_and_Orchestration.md`：详细解释 Agent、主循环、Todo、Task 和 Subagent。
- `09_Context_and_Persistence.md`：详细解释 Context、Session History、持久化与 Compaction。
- `10_Tools_and_Security.md`：详细解释 Tool、Permission、执行生命周期和安全边界。
- `11_Runtime_Boundary.md`：详细解释 Client、Server、事件、Provider 和 SDK 边界。
- `12_V1_V2_Comparison.md`：按模块汇总 V2 相对 V1 的改进、兼容关系和未完成迁移。
- `../../README.md`：在 Harness 系列形成后更新阅读顺序和章节说明。

后续编号和模块边界可以在源码研究后调整。固定要求是 06 只承担总览，不强行容纳所有细节；专题文档各自提供完整源码依据。

## 2. 研究范围

Harness 系列只讨论 OpenCode 内部 Agent Harness，主要包括：

- Agent 与模型选择。
- Agent Loop 和 Orchestration。
- Context 组装和更新。
- Session、Message、Part 与持久化。
- Tool 注册、授权、执行和结果回传。
- Permission 与安全边界。
- Todo、Task 和 Subagent。
- Compaction、Pruning、Retry、Interrupt、Recovery 和 Snapshot。
- Client、Server、事件和 Provider 边界如何支撑 Harness 运行。
- 当前旧运行时与 V2 替代架构的区别、迁移状态和改进目标。

## 3. 不在 Harness 系列中展开的内容

- 不介绍泛化的 Harness Engineering 方法论。
- 不把仓库文档、CI/CD、团队流程等所有 Agent-friendly 工程实践都纳入 Harness 定义。
- 不重复 `../../OpenCode_Tutorial/01_Installation.md` 的安装步骤。
- 不重复 `../../OpenCode_Tutorial/02_Model_Deployment.md` 的模型接入教程。
- 不重复 `../../OpenCode_Tutorial/03_OpenCode_Server_Deployment.md` 的 systemd、Docker 和反向代理操作步骤。
- 不重复 `../05_Enhancement.md` 的 Skill、MCP 和 Plugin 安装教程。

上述内容只有在解释 Harness 内部边界和调用关系时才会被引用。

## 4. 本地源码基线

核对日期：2026-08-18。

| 项目 | 当前值 |
| --- | --- |
| 本地仓库 | `/home/wuzhongyun/projects/Intern_projects/Opencode_learn/opencode github code` |
| 分支 | `dev` |
| Commit | `0e3474509aa5ad16afcf9c439785514d6443c6af` |
| Git describe | `github-v1.2.25-1693-g0e3474509a` |
| `packages/opencode` 版本 | `1.18.18` |
| 初始工作树状态 | 干净，跟踪 `origin/dev` |

研究过程中默认固定在该 commit。若本地源码更新，必须先记录新 commit，再决定继续使用旧基线还是重新验证结论。

## 5. V1 与 V2 的初始定义

本项目中的 V1/V2 首先是架构和兼容性标签，不能简单等同于产品版本号：

- V1：仍被当前应用使用或为兼容、持久化、迁移而保留的旧合同与旧运行时。
- V2：正在替代旧运行时的 Effect-native 架构；仓库希望其成熟后成为无版本前缀的 current API，因此 `V2` 也是过渡名称。
- `packages/opencode`：包含当前可执行应用、旧 Session Orchestration，以及连接新 Core 的兼容层。
- `packages/core`、`packages/schema`、`packages/protocol`、`packages/server`、`packages/client`、`packages/sdk-next`：承载新的模块化方向，但不能仅凭目录判断所有能力都已完成或默认启用。

正式文档不得把“V1”写成一个已经完全消失的历史版本，也不得把“V2”写成一个已经全部替换完成的独立产品。

## 6. 成功标准

研究完成时，读者应能够：

1. 从一次用户输入开始，说明 OpenCode 如何完成 Context 组装、Provider Turn、Tool Call、Tool Result 和后续循环。
2. 区分模型作出的概率性决策与 Harness 执行的确定性控制。
3. 说明哪些状态会持久化，哪些状态只存在于当前进程。
4. 说明 Tool、Permission、Agent 和 Subagent 的边界。
5. 说明当前旧运行时与 V2 在各模块上的主要差异。
6. 根据文档引用定位到关键源码和测试。
7. 正确使用 `CONTEXT.md` 中的核心术语，而不混淆相邻概念。

每个 Harness 正式模块都必须具有源码依据。源码引用必须说明文件位置、所在函数或方法、行号和 commit。只引用概念文章而没有源码或测试支撑的板块不能标记为完成。

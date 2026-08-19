# OpenCode Harness 源码与证据索引

本附录为主系列提供可追溯的实现入口。正文负责建立学习主线；这里负责说明结论来自哪里。

## 1. 固定基线

| 项目 | 值 |
| --- | --- |
| 源码仓库 | `/home/wuzhongyun/projects/Intern_projects/Opencode_learn/opencode github code` |
| 分支 | `dev` |
| Commit | `0e3474509aa5ad16afcf9c439785514d6443c6af` |
| Git describe | `github-v1.2.25-1693-g0e3474509a` |
| `packages/opencode` 版本 | `1.18.18` |
| 核对日期 | 2026-08-18 |

源码继续演进后，文件名、行号和默认接线都可能变化。引用本系列结论时，应同时保留 commit。

## 2. 证据优先级

1. 固定 commit 下实际接线的源码和聚焦测试。
2. 仓库 `AGENTS.md`、`CONTEXT.md` 与 `specs/v2/` 中的维护约束和设计文档。
3. OpenCode 官方用户文档。
4. 通用 Agent 资料和外部架构文章。

设计文档可以解释目标，但不能单独证明某项能力已经成为默认行为；外部文章可以帮助提出问题，但不能替代当前源码证据。

## 3. 总览与当前请求链

| 关注点 | 关键入口 |
| --- | --- |
| TUI 普通消息提交 | `packages/tui/src/component/prompt/index.tsx`，`submitInner` |
| 兼容 Session HTTP Handler | `packages/opencode/src/server/routes/instance/httpapi/handlers/session.ts` |
| 当前编排入口 | `packages/opencode/src/session/prompt.ts`，`SessionPrompt.prompt/run/loop` |
| 当前运行协调 | `packages/opencode/src/session/run-state.ts` |
| 流处理和终态 | `packages/opencode/src/session/processor.ts`，`SessionProcessor` |

## 4. Agent Loop 与模型请求

| 关注点 | 关键入口 |
| --- | --- |
| Agent 定义与默认选择 | `packages/opencode/src/agent/agent.ts` |
| Model 与 Variant 选择 | `packages/opencode/src/session/prompt.ts` |
| Provider 请求准备 | `packages/opencode/src/session/llm/request.ts` |
| LLM Runtime | `packages/opencode/src/session/llm.ts` 及 `session/llm/` |
| Retry 策略 | `packages/opencode/src/session/retry.ts` |
| Session 状态与中断 | `packages/opencode/src/session/status.ts`、`run-state.ts` |

## 5. Context

| 关注点 | 关键入口 |
| --- | --- |
| 当前 system-level 内容 | `packages/opencode/src/session/system.ts` |
| 项目指令发现 | `packages/opencode/src/session/instruction.ts` |
| 历史与请求转换 | `packages/opencode/src/session/message-v2.ts`、`session/llm/request.ts` |
| Compaction 与 pruning | `packages/opencode/src/session/compaction.ts` |
| native System Context | `packages/core/src/system-context/` |
| native Context Epoch | `packages/core/src/session/context-epoch.ts` |

## 6. Tools 与 Permission

| 关注点 | 关键入口 |
| --- | --- |
| Tool Registry | `packages/opencode/src/tool/registry.ts` |
| 每轮 Tool 物化 | `packages/opencode/src/session/tools.ts`，`SessionTools.resolve` |
| Permission | `packages/opencode/src/permission/` |
| Built-in Tool executor | `packages/opencode/src/tool/` 中各工具实现 |
| Task / Subagent | `packages/opencode/src/tool/task.ts` |
| MCP Tool 接入 | `packages/opencode/src/mcp/` |
| Plugin hooks | `packages/opencode/src/plugin/` |
| native typed Tool Registry | `packages/core/src/tool/` |

## 7. Session、Persistence 与 Event

| 关注点 | 关键入口 |
| --- | --- |
| 当前 Session 领域接口 | `packages/opencode/src/session/session.ts` |
| Message/Part 类型与转换 | `packages/opencode/src/session/message-v2.ts` |
| durable Event 提交与通知 | `packages/core/src/event.ts` |
| Session projection | `packages/core/src/session/projector.ts` |
| SQLite Session schema | `packages/core/src/session/sql.ts` 及相关 `*.sql.ts` |
| Snapshot / Revert | `packages/opencode/src/snapshot/`、Session revert 相关代码 |

## 8. Client、Server 与运行边界

| 关注点 | 关键入口 |
| --- | --- |
| TUI Worker 与 Server 连接 | `packages/opencode/src/cli/cmd/tui.ts`、`packages/opencode/src/cli/tui/worker.ts` |
| executable Router | `packages/opencode/src/server/` |
| compatibility event bridge | `packages/opencode/src/event-v2-bridge.ts`、`packages/opencode/src/bus/global.ts` |
| native Protocol | `packages/protocol/` |
| native Server Handler | `packages/server/src/handlers/session.ts` |
| native API Client | `packages/client/` |
| sdk-next 组合 | `packages/sdk-next/` |

## 9. native Session Runtime

| 关注点 | 关键入口 |
| --- | --- |
| durable prompt admission | `packages/core/src/session.ts`，`V2Session.prompt` |
| Session execution | `packages/core/src/session/execution.ts`、`packages/core/src/session/execution/local.ts` |
| Session Runner | `packages/core/src/session/runner/` |
| run coordinator | `packages/core/src/session/run-coordinator.ts` |
| 新 Session 设计与 parity | `specs/v2/session.md` |
| Tool 设计 | `specs/v2/tools.md` |

固定基线中的命名仍处于迁移阶段。仓库维护约束明确指出，新架构最终应使用无版本前缀的 current contract；本系列沿用“native V2”只是为了与当前兼容路径清楚区分。

## 10. 代表性测试入口

测试文件只能证明对应 slice 有可执行覆盖，不能单独证明当前 TUI 已使用该路径，也不能把某次定向测试外推为完整 Runtime parity。

| 验证主题 | 测试入口 |
| --- | --- |
| 当前 Loop、continuation 与 Interrupt | `packages/opencode/test/session/prompt.test.ts`、`packages/opencode/test/session/processor-effect.test.ts` |
| Provider Retry | `packages/opencode/test/session/retry.test.ts` |
| System、Skill 与 MCP guidance | `packages/opencode/test/session/system.test.ts` |
| Compaction 与 Pruning | `packages/opencode/test/session/compaction.test.ts` |
| Agent、Plan 与 Task/Subagent | `packages/opencode/test/agent/agent.test.ts`、`packages/opencode/test/agent/plan-mode-subagent-bypass.test.ts`、`packages/opencode/test/tool/task.test.ts` |
| Event transaction 与 replay | `packages/core/test/event.test.ts` |
| native prompt admission | `packages/core/test/session-prompt.test.ts`、`packages/opencode/test/server/httpapi-session.test.ts` |
| native Runner 与协调 | `packages/core/test/session-runner.test.ts`、`packages/core/test/session-run-coordinator.test.ts` |
| TUI live event 与 hydration | `packages/tui/test/cli/cmd/tui/sync-live-hydration.test.tsx` |

## 11. 研究材料

旧版端到端 trace、模块研究、实验记录和术语翻译在主系列完成审核后归档至：

```text
../archive/research_2026-08-18/
```

这些文件保留完整证据链和未决问题，但不作为初学者的默认阅读步骤。

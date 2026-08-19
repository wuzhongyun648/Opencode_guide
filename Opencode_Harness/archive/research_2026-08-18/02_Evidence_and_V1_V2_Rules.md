# 证据、引用与 V1/V2 判定规则

## 1. V1 信息是否仍然存在

存在，而且保留形式不止一种。

### 明确命名的 V1 合同和兼容代码

- `packages/schema/src/v1/`
- `packages/core/src/v1/`
- `packages/schema/src/session-v1.ts`
- `packages/schema/src/permission-v1.ts`
- `packages/schema/src/question-v1.ts`
- `packages/schema/src/legacy-event.ts`

### 仍在旧应用运行时中的实现

旧运行时主要不在 `v1/` 目录，而在：

- `packages/opencode/src/session/`
- `packages/opencode/src/tool/`
- `packages/opencode/src/permission/`
- `packages/opencode/src/server/`

因此，不能只搜索名为 `v1` 的目录来理解旧架构。

### V2 规格和替代实现

- `CONTEXT.md`
- `specs/v2/`
- `packages/core/src/session/`
- `packages/core/src/system-context/`
- `packages/core/src/tool/`
- `packages/protocol/`
- `packages/server/`
- `packages/client/`
- `packages/sdk-next/`

## 2. 命名注意事项

根据 `packages/schema/AGENTS.md`：

- 当前合同最终应使用无版本名称，如 `Session`、`Permission`、`Question`。
- 为兼容、持久化或迁移保留的旧合同显式标为 `V1`。
- `V2` 不应成为替代架构的永久名称，迁移完成后 current namespace 会逐步去掉 `V2`。

因此，研究笔记中的“V1”和“V2”是为了说明迁移边界，不代表官方最终会长期使用这两个名称。

以下名称尤其不能仅凭字面判断：

- `packages/opencode/src/session/message-v2.ts` 位于旧应用运行时中，文件名包含 `v2` 不代表它属于新的 native V2 Core。
- `packages/sdk/js/src/v2/` 属于旧 JavaScript SDK 的一个版本化入口，不等于 `packages/sdk-next`。
- `console/.../zen/v1` 等目录可能是外部 HTTP API 版本，与 Session Harness 的 V1/V2 无关。

## 3. 证据优先级

| 等级 | 证据 | 可支持的结论 |
| --- | --- | --- |
| E1 | 固定 commit 下的入口接线、实现代码和测试 | 本地代码实际实现及边界 |
| E2 | 可复现的本地运行或 Recorded Provider 实验 | 某条执行路径的实际行为 |
| E3 | OpenCode 官方用户文档和 Release Notes | 对外支持的用户行为 |
| E4 | `CONTEXT.md`、`specs/v2/*.md`、仓库 `AGENTS.md` | V2 术语、设计意图、迁移状态和约束 |
| E5 | 官方 PR、Issue 和提交历史 | 设计原因和演化过程 |
| E6 | Google 白皮书等权威通用资料 | 通用 Agent 概念，不直接证明 OpenCode 行为 |
| E7 | DeepWiki、知乎文章等第三方资料 | 理解线索、类比和待验证主张 |
| E8 | 我们的推断 | 解释性结论，必须标明推断依据和不确定性 |

规格文档比第三方文章权威，但规格中的 `follow-up`、`missing` 和计划不能作为当前行为证据。代码存在也不等于默认路径已使用，必须继续查入口接线、测试或运行结果。

## 4. 状态标签

研究笔记中的 OpenCode 特有结论使用以下标签：

| 标签 | 含义 |
| --- | --- |
| `[Current default @ SHA]` | 当前默认入口实际使用 |
| `[Current compatibility @ SHA]` | 为旧 API、存储或客户端保留的兼容路径 |
| `[Current experimental @ SHA]` | 代码存在但仍标记为实验性或未稳定 |
| `[V2 implemented @ SHA]` | V2 替代路径已经有实现和测试 |
| `[V2 partial @ SHA]` | 只覆盖旧行为的一部分 |
| `[V2 missing/planned]` | 规格明确表示缺失或待设计 |
| `[Official docs @ date]` | 官方公开文档描述的用户行为 |
| `[General concept]` | 通用 Agent 理论 |
| `[Third-party claim]` | 第三方资料的主张，尚不能当作事实 |
| `[Interpretation]` | 基于多项证据形成的解释 |
| `[Unresolved]` | 当前证据不足或相互冲突 |

正式文档不需要在每句话前显示标签，但必须通过小节标题、版本说明或比较表清楚表达这些边界。

## 5. 每个板块的最低源码证据

Harness 系列的每个核心板块至少需要：

1. 一个公开入口或上层调用点，并注明文件位置和函数、方法或导出符号。
2. 一个核心实现文件或关键函数，并注明文件位置、函数名和职责。
3. 一个相关测试。
4. 一个 V1/V2 对照点；如果 V2 尚未实现，应引用明确的状态说明。
5. 一段解释源码如何支撑结论的文字，不能只列文件名。

关键结论优先使用短源码片段说明。片段应只保留理解机制所需的部分，不复制整个函数。

## 6. 引用格式

### 本地源码

```text
文件：packages/opencode/src/session/prompt.ts
函数：SessionPrompt.loop(...)
位置：100-150
版本：0e3474509a
```

### 测试

```text
文件：packages/core/test/session-runner.test.ts
测试：<describe/it 测试名称>
位置：<line-line>
版本：0e3474509a
```

### 官方设计文档

```text
specs/v2/session.md:123-151 @ 0e3474509a
```

### Web 资料

记录标题、作者、发布日期、URL、访问日期和对应章节。URL 相同的转载或搜索摘要不能算独立证据。

### PDF

```text
Google, Introduction to Agents, Updated May 2026, pp. 8-12.
```

## 7. 原子主张记录模板

```markdown
### Claim ID: ORCH-001

- 主张：
- 模块：
- 状态标签：
- 当前实现证据：
- 当前实现函数或符号：
- V2 证据：
- V2 函数或符号：
- 测试或实验：
- 第三方资料：
- 反例或限制：
- 当前结论：
- 置信度：高 / 中 / 低
```

同一作者的五篇知乎文章属于一个来源系列。多篇文章重复同一说法只能提高其研究优先级，不能提高事实置信度。

## 8. V1/V2 比较方法

每个模块按照同一组问题比较：

1. V1 的主要职责放在哪里？
2. V1 如何保存状态和处理失败？
3. V1 的模块边界和耦合点是什么？
4. V2 将职责移动到哪些 package 或 service？
5. V2 引入了哪些新的持久化、不变量或类型边界？
6. V2 解决了 V1 的什么问题？
7. V2 当前已完成什么？
8. V2 仍缺少哪些 V1 parity？
9. V1 与 V2 目前通过什么兼容层共存？

V2 改进必须从源码、测试、规格或提交历史中推导，不能只依据“模块更多”或“使用 Effect”得出更可靠的结论。

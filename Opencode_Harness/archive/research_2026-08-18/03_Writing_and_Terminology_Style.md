# Harness 系列写作、源码片段与术语规范

## 1. 总体写作原则

- 先解释问题和运行流程，再展示源码。
- 源码用于证明和具体化机制，不把正式文档写成逐文件注释。
- 区分通用 Agent 概念、OpenCode 当前实现和 V2 设计。
- 每个重要流程使用一个具体例子贯穿，避免只列抽象名词。
- 每节明确回答“做了什么、为什么需要、如何实现、失败时怎样处理”。
- 不使用没有证据的绝对描述，如“始终”“完全”“保证”。
- 当前实现与 V2 不放在同一张无标签流程图中。

## 2. 专业术语的首次出现格式

有稳定且不会扭曲含义的中文表达时，第一次使用：

```text
会话历史（Session History）
提供商轮次（Provider Turn）
上下文压缩（Compaction）
```

后续可以主要使用中文，但在容易混淆的段落中保留英文。

没有公认中文译法时，也先给出准确的解释性中文名称，并在第一次出现时保留英文原文：

```text
会话执行排空（Session Drain）
工具结果结算（Tool Settlement）
上下文纪元（Context Epoch）
```

如果解释性中文仍可能引起误解，应在紧邻正文中补充它的准确含义，而不是只保留一个看似直译的中文词。

代码标识、类型名、事件名、配置键和 API 路径保持原样，例如：

- `SessionV2.prompt`
- `session_input`
- `PromptAdmitted`
- `tool.execute.before`
- `/api/session/:sessionID/history`

## 3. 不直接互换的词

以下词不能因为中文相近而混用：

| 术语 | 约束 |
| --- | --- |
| System Context | 不自动改写成 System Prompt；V2 官方明确区分术语 |
| Session History | 不写成 Session Context |
| Context Source | 不写成 Prompt Fragment |
| Context Snapshot | 不等同于代码工作树 Snapshot |
| Context Epoch | 不等同于 Session 生命周期 |
| Provider Turn | 不等同于一次完整用户对话 |
| Session Drain | 不等同于持久化任务或一次 Message |
| Tool Call | 不等同于 Tool Result 或 Tool Settlement |
| Permission | 不等同于 OS Sandbox |
| Session Persistence | 不自动称为长期 Memory |
| Subagent | 不等同于普通 Tool，也不等同于远程 A2A Agent |

如果代码仍使用旧命名，而 V2 术语要求使用新表达，应同时说明命名时代和语义差异，不能静默改写源码标识。

## 4. 07 翻译规范

`07_OpenCode_Runtime_Terminology.md` 分开呈现：

1. 官方英文术语。
2. 建议中文表达。
3. 官方定义的忠实翻译。
4. 通俗解释。
5. 与相邻术语的区别。
6. 当前实现或 V2 状态注释。
7. 官方 `_Avoid_` 指南。

建议模板：

```markdown
### 上下文纪元（Context Epoch）

**官方定义翻译**

...

**通俗解释**

...

**不要混淆**

...

**实现状态**

...

**官方建议避免的说法**

...
```

忠实翻译和我们的解释必须分段，不能把解释性补充混进官方定义。

## 5. 源码片段规范

- 片段通常控制在 5-30 行。
- 省略代码时使用注释明确标记，不拼接成看似连续的伪源码。
- 片段前说明要观察什么。
- 片段后逐步解释控制流、状态变化和边界条件。
- 片段注明文件路径、所在函数或方法、行号和 commit。
- TypeScript 类型和 Effect 组合需要说明其业务意义，不只解释语法。
- 不复制第三方文章中的源码作为 OpenCode 当前实现证据。
- 正式源码引用不能只给文件路径；必须同时给出所在函数、方法、测试名称或导出符号。

示例结构：

````markdown
下面的入口先持久化输入，再唤醒执行器：

```ts
// 精简后的真实源码片段
```

文件：`packages/...`
函数：`Namespace.function(...)`
位置：`line-line`
版本：`SHA`

这段代码体现三个边界：...
````

## 6. 图表规范

- 总览图只展示稳定模块边界。
- 时序图用于展示一次具体请求。
- 状态图用于展示 Session、Tool 或 Prompt 状态变化。
- V1 和 V2 使用独立图，或在同一对照图中明确分栏。
- 每条关键箭头都应能够映射到源码入口或事件。
- 图中保持代码标识原文，正文再给中文解释。

## 7. 06 单节模板

```markdown
## 模块名称

### 要解决的问题
### 核心概念
### 当前 OpenCode 如何实现
### 关键源码
### 一次具体执行流程
### 失败、取消与边界条件
### V2 相对 V1 的变化
### 当前迁移状态
### 小结与理解检查
```

不是每节都必须机械保留全部标题，但内容必须覆盖这些问题。

## 8. V1/V2 表述示例

推荐：

```text
在本次核对的旧运行时中，Session Loop 主要位于 ...。V2 将 Prompt admission 与执行分离，但官方 parity 表仍将 ... 标为 missing。
```

不推荐：

```text
OpenCode V2 使用了全新的先进架构，彻底解决了 V1 的问题。
```

推荐：

```text
V2 为该状态增加了 durable event 和 projection；其目标是支持重放和恢复。当前 post-crash continuation 仍被列为待设计能力。
```

## 9. 与已有章节的关系

- 需要安装操作时链接 `../../OpenCode_Tutorial/01_Installation.md`。
- 需要模型配置时链接 `../../OpenCode_Tutorial/02_Model_Deployment.md`。
- 需要 Server 部署操作时链接 `../../OpenCode_Tutorial/03_OpenCode_Server_Deployment.md`。
- 需要 CLI/TUI 用法时链接 `../../OpenCode_Tutorial/04_Useful_Commands.md`。
- 需要 Skill、MCP、Plugin 安装和使用时链接 `../05_Enhancement.md`。
- 06 负责总览，07 及后续专题解释内部机制，不重复上述操作教程。

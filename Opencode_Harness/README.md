# OpenCode Harness 架构学习系列

本目录分为三类内容：

1. `05_Enhancement.md` 是 Skill、MCP 与 Plugin 的使用和扩展教程。
2. `06` 至 `12` 是面向学习者的 OpenCode Agent Harness 架构主系列。
3. `appendices/`、`planning/` 与 `archive/` 分别保存查询资料、写作约束和历史研究材料。

## 主系列解决什么问题

主系列围绕一条稳定的 Agent 反馈循环展开：

```text
目标
-> 组织当前上下文
-> 模型判断下一步
-> Harness 验证并执行工具
-> 保存结果与更新状态
-> 为下一轮重新组织上下文
-> 继续、重试、请求确认或停止
```

贯穿场景是一位零基础学习者从头理解 Harness，并在 OpenCode 中完成低风险的观察和实践。示例优先使用查看配置、读取教程文件、询问工具能力和观察 Session 变化等操作，不要求读者掌握并发、数据库或复杂业务代码。

## 阅读顺序

| 章节 | 核心问题 |
| --- | --- |
| [06 Harness 总览](./06_Harness.md) | 模型怎样在 Harness 中成为能够行动的编码 Agent？ |
| [07 Agent Loop](./07_Agent_Loop.md) | 一次请求为什么可以产生多轮判断与行动？ |
| [08 Context Architecture](./08_Context_Architecture.md) | 模型每一轮实际看见什么？ |
| [09 Tools 与 Permission](./09_Tools_and_Permission.md) | 模型提出的意图怎样变成真实操作？ |
| [10 Session 与 Persistence](./10_Session_and_Persistence.md) | 系统保存什么，下一轮为什么能继续？ |
| [11 Agent 专业化与协作](./11_Agent_Specialization_and_Collaboration.md) | Agent、Plan、Todo、Task 与 Subagent 如何分工？ |
| [12 Runtime Boundary](./12_Runtime_Boundary.md) | Harness 在哪里运行，结果怎样返回用户？ |

## 写作与证据原则

- 每篇先讲通用 Agent 问题，再讲 OpenCode 当前实现，最后给源码与测试入口。
- 正文优先保证学习主线；研究任务编号、测试日志和完整迁移矩阵不进入正文。
- 当前默认 TUI 路径是主线。native V2 只在确实改变概念边界时出现，完整演进集中到第 12 篇和附录。
- 图表用于解释关系和时序，源码用于证明关键结论，不逐文件讲解代码。
- 重要结论以固定源码 `0e3474509aa5ad16afcf9c439785514d6443c6af` 为基线。

更详细的约束见 [planning](./planning/) 目录。

## 查询与历史材料

- [简明术语表](./appendices/Terminology.md)：随正文查询相邻术语及其主讲章节。
- [源码与证据索引](./appendices/Source_Index.md)：固定 commit、关键源码、测试与证据边界。
- [历史材料说明](./archive/README.md)：旧正式稿和研究底稿的归档入口。

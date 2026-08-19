# Harness 系列重构章程

## 1. 重构目标

把已有的 OpenCode 源码研究和正式稿重构成一套面向初学者的架构学习系列。读者完成 01-05 的基础教程后，应能沿着一次 Agent 任务的完整反馈循环理解 OpenCode，而不需要先熟悉复杂代码、数据库实现或 V1/V2 迁移背景。

本轮不是重新发明研究结论，也不是模仿某组参考文章的章节划分。已有 `research/` 和 06-12 旧稿是事实底稿；新系列重新设计学习顺序、模块职责和表达层次。

## 2. 目标读者

- 已经能够安装并启动 OpenCode。
- 知道大语言模型可以生成文本，但尚不理解 Agent、Harness、Tool、Context 与 Session 的关系。
- 希望从架构层面理解 OpenCode，而不是立即参与 OpenCode Runtime 开发。
- 不默认具备并发、事件溯源、Effect、SQLite 或分布式系统知识。

## 3. 贯穿场景

贯穿人物是一位从零学习 Harness 的读者。任务从“请解释这个项目中的 Harness 学习入口”开始，逐渐观察 OpenCode：

1. 接收学习目标。
2. 获取教程与项目规则。
3. 判断需要读取哪些文件或信息。
4. 提出并执行低风险 Tool Call。
5. 把 Tool Result 放回下一轮输入。
6. 保存学习过程中的 Session 状态。
7. 必要时使用 Plan、Todo 或 Subagent 组织更复杂的学习任务。
8. 最终向用户总结已经理解的架构与下一步实践。

示例不得依赖读者理解缓存并发、复杂测试体系或特定业务领域。需要演示写操作时，使用明确的临时学习文件，并先解释 Permission 与影响范围。

## 4. 成功标准

读完主系列后，读者应能用自己的话说明：

1. 为什么 Model 不等于 Agent。
2. Harness 如何运行多轮 `Context -> Model -> Tool -> Observation` 循环。
3. 模型每轮看到什么，哪些内容由系统重新注入。
4. Tool Call 为什么不等于工具已经执行。
5. Session 保存了什么，哪些状态不会跨进程保留。
6. Agent、Model、Plan、Todo、Task 与 Subagent 的区别。
7. TUI、Server、Provider、Tool Runtime 和事件通道分别位于什么边界。

## 5. 不追求的目标

- 不把正文写成 OpenCode 全部源码的逐文件导览。
- 不把通用 Harness Engineering、CI/CD、团队治理和所有 Agent Ops 主题都纳入本系列。
- 不把参考文章的标题或结构直接复制为本项目结构。
- 不为了显得完整而重复讲解同一概念。
- 不把规划中的 V2 能力写成当前默认行为。

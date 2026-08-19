# OpenCode Harness 简明术语表

本术语表用于随读随查，不是主系列的前置章节。定义以固定源码 `0e3474509aa5ad16afcf9c439785514d6443c6af` 和仓库 `CONTEXT.md` 为基线；“当前默认”指固定基线下 TUI 普通消息实际使用的兼容 Session Runtime。

| 术语 | 在本系列中的含义 | 不要混淆 | 主讲章节 |
| --- | --- | --- | --- |
| Model | 根据本轮输入生成 Text、Reasoning 或 Tool Call 的推理服务 | 不等于完整 Agent | [06](../06_Harness.md) |
| Agent | 影响角色、指令、模型偏好、工具和 Permission 的行为配置 | 不只是 Model 别名或一段 Prompt | [11](../11_Agent_Specialization_and_Collaboration.md) |
| Harness | 包围模型、组织 Context/Tools/State/Runtime 的控制与执行系统 | 不等于 System Prompt、Tool 列表或 Server | [06](../06_Harness.md) |
| Orchestration | 管理模型调用、工具执行、状态更新和停止判断的控制流 | 不等于模型内部推理 | [07](../07_Agent_Loop.md) |
| Agent Loop | 围绕目标反复组织 Context、调用 Model、执行 Action 并观察结果的循环 | 不要求每轮都调用工具 | [07](../07_Agent_Loop.md) |
| Provider Turn | Harness 真实发起的一次 Provider Request attempt，以及其响应或错误投影 | 不等于完整用户请求；每次 Retry attempt 也各自构成 Turn，但不重组 Context | [07](../07_Agent_Loop.md) |
| Context | 模型某一轮实际接收到的信息环境 | 不等于只含 System Prompt | [08](../08_Context_Architecture.md) |
| System Context | native V2 对系统级上下文使用的正式术语与类型化结构 | 不应自动改写为 System Prompt | [08](../08_Context_Architecture.md) |
| Session History | 某个 Session 中可供重载和选择的历史 Message/Part | 不等于模型的天然长期记忆，也不等于本轮完整可见输入 | [08](../08_Context_Architecture.md) |
| Tool Definition | 告诉模型工具名称、说明和参数结构的定义 | 不等于 Tool 已执行 | [09](../09_Tools_and_Permission.md) |
| Tool Call | 模型或 Provider 提出的结构化工具调用意图 | 不等于 Tool Result | [09](../09_Tools_and_Permission.md) |
| Tool Result | 工具执行后产生的原始或领域结果 | 不自动等于下一轮模型最终看到的格式 | [09](../09_Tools_and_Permission.md) |
| Tool Settlement | 将 Tool Call 结算为 completed/error 等终态并形成可继续使用的输出 | 不等于单纯返回字符串 | [09](../09_Tools_and_Permission.md) |
| Permission | OpenCode 对受管 action/resource 的应用层策略 | 不等于 OS Sandbox | [09](../09_Tools_and_Permission.md) |
| Session | 组织连续交互、消息、状态和运行边界的领域对象 | 不等于单次 Provider Turn | [10](../10_Session_and_Persistence.md) |
| Message | Session 中的 User 或 Assistant 消息记录 | 一条消息可以包含多个 Part | [10](../10_Session_and_Persistence.md) |
| Part | Text、Reasoning、Tool、File 等消息组成部分 | delta 不一定 durable | [10](../10_Session_and_Persistence.md) |
| Event | 描述领域事实或实时变化的事件 | 不等于网络事件通道或数据库本身 | [10](../10_Session_and_Persistence.md) |
| durable | 已提交到持久化边界，可在进程结束后重新读取 | 不代表系统一定能自动续跑副作用 | [10](../10_Session_and_Persistence.md) |
| process-local | 只存在于当前 OpenCode 进程的协调或运行状态 | 进程重启后不能依赖其仍在 | [10](../10_Session_and_Persistence.md) |
| live-only | 为实时显示或瞬时状态发布，但不作为完整持久记录保存 | 不等于 whole Part | [10](../10_Session_and_Persistence.md) |
| Compaction | 用摘要等更短表示替代早期历史，以控制模型输入长度 | 不等于删除 Session，也不等于工作树 Snapshot | [10](../10_Session_and_Persistence.md) |
| Snapshot | 保存某类状态在特定时刻的表示；需根据上下文区分 Context Snapshot 与代码工作树 Snapshot | 不等于 Compaction | [10](../10_Session_and_Persistence.md) |
| Todo | Agent 可读写的结构化任务清单 | 不是自动调度器 | [11](../11_Agent_Specialization_and_Collaboration.md) |
| Task Tool | 创建或恢复子 Session 并委派子任务的工具 | 不等于 Todo 或 Subagent 本身 | [11](../11_Agent_Specialization_and_Collaboration.md) |
| Subagent | 当前默认路径中使用独立 Agent 配置和子 Session 完成受限子任务的 Agent | 不等于任意远程 Agent | [11](../11_Agent_Specialization_and_Collaboration.md) |
| Client | 提交请求、接收事件并显示状态的调用方 | 不等于 Session Runtime | [12](../12_Runtime_Boundary.md) |
| Provider | 实际处理模型请求的远程 API 或本地模型服务 | 通常不直接访问 OpenCode SQLite | [12](../12_Runtime_Boundary.md) |
| Event Channel | 把实时变化送到 Client 的 transport | 不等于 durable Event 存储 | [12](../12_Runtime_Boundary.md) |
| 当前默认兼容 Runtime / native V2 | 当前 TUI 普通消息实际使用的编排 / 已接线但尚未接管默认 TUI 的新 Session Runtime | 不等于仓库的 current namespace，也不是产品版本号比较 | [12](../12_Runtime_Boundary.md) |

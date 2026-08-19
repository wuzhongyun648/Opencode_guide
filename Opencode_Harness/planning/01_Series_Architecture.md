# Harness 主系列架构与内容所有权

## 1. 总体结构

主系列采用“总览 + 六个问题”的结构：

| 文件 | 唯一主问题 | 内容所有权 |
| --- | --- | --- |
| `06_Harness.md` | 模型如何成为能够行动的编码 Agent？ | Harness 边界、全景图、最短执行链、阅读地图 |
| `07_Agent_Loop.md` | 一次请求为什么可以持续多轮？ | Provider Turn、循环、continuation、retry、interrupt、stop |
| `08_Context_Architecture.md` | 模型本轮看见什么？ | System、Messages、Tool definitions、指令来源、裁剪与可见性 |
| `09_Tools_and_Permission.md` | 意图如何变成真实操作？ | Tool 注册、物化、调用、授权、执行、结果与安全边界 |
| `10_Session_and_Persistence.md` | 系统保存了什么？ | Session、Message、Part、Event、durable/live、Compaction、恢复 |
| `11_Agent_Specialization_and_Collaboration.md` | 不同 Agent 如何分工？ | Agent/Model、Build/Plan、Todo、Task、Subagent、复杂度选择 |
| `12_Runtime_Boundary.md` | Harness 在哪里运行？ | TUI、Server、Provider、Tool Runtime、事件通道、架构演进 |

## 2. 跨模块概念的归属

为避免重复，跨模块概念按“主讲章节 + 其他章节只建立接口”的方式处理：

| 概念 | 主讲章节 | 其他章节允许出现的范围 |
| --- | --- | --- |
| Agent Loop | 07 | 06 只给最短链路；其他篇只引用所在阶段 |
| Context Engineering | 08 | 06 给定义；07 说明每轮重组；10 只谈保存与取回 |
| Compaction | 10 | 08 只解释它怎样改变模型可见内容 |
| Tool lifecycle | 09 | 07 只把 Tool 作为循环中的 Action/Observation |
| Permission | 09 | 11 只解释 Agent policy 如何影响授权 |
| Plan、Todo、Task、Subagent | 11 | 07 只说明它们不是 Agent Loop 本身 |
| Retry、Interrupt、Stop | 07 | 12 只讨论断线或进程边界带来的限制 |
| Event | 10 | 12 只讲事件如何跨运行边界返回 Client |
| V1/V2 | 12 与附录 | 其他篇最多保留一个与当前主题直接相关的简短说明框 |

## 3. 学习顺序

```text
06 建立全景
  -> 07 看懂循环
  -> 08 看懂模型输入
  -> 09 看懂真实行动
  -> 10 看懂状态延续
  -> 11 看懂角色协作
  -> 12 看懂运行边界和架构演进
```

这个顺序从读者能直接观察的行为进入内部模块，再到部署和迁移边界。术语随正文首次出现时解释，不要求先读完整术语表。

## 4. 附录

主系列完成后保留两类查询资料：

- `appendices/Terminology.md`：简明术语表，只提供定义、区别和主讲章节链接。
- `appendices/Source_Index.md`：固定源码、关键文件、函数、测试和证据边界。

旧正式稿、研究笔记和实验记录归档后仍可追溯，但不出现在默认阅读顺序中。

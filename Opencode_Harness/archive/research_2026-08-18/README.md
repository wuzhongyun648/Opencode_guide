# Harness 研究底稿

本目录保存 2026-08-18 阶段形成的研究章程、证据规则、端到端 trace 和四组模块研究。它们是新主系列事实审核的重要来源，但不是默认学习步骤。

## 材料分组

| 文件 | 用途 |
| --- | --- |
| `00_Project_Charter.md` | 研究目标、范围、源码基线和成功标准 |
| `01_Collaboration_and_Workflow.md` | 原研究协作和阶段流程 |
| `02_Evidence_and_V1_V2_Rules.md` | 证据等级与迁移状态表述规则 |
| `03_Writing_and_Terminology_Style.md` | 旧写作、源码片段和术语规范 |
| `04_Module_Map_and_Sources.md` | 模块地图、官方资料和外部文章索引 |
| `05_Research_Baseline_and_Questions.md` | 当前 Runtime 定义与核心研究问题 |
| `06_Current_Runtime_End_to_End_Trace.md` | 默认 TUI 普通消息的完整请求链 |
| `10_Research_Agent_and_Orchestration.md` | Agent、Loop、Todo、Task、Subagent 与可靠性控制 |
| `11_Research_Context_and_Persistence.md` | Context、History、Persistence、Compaction 与恢复 |
| `12_Research_Tools_and_Security.md` | Tool、Permission、执行、结算与安全边界 |
| `13_Research_Runtime_Boundary.md` | Client、Server、Provider、事件与新旧 API 接线 |

固定源码基线是 OpenCode commit `0e3474509aa5ad16afcf9c439785514d6443c6af`。研究文件中的任务编号、测试输出、Open Questions 和“尚未验收”等文字记录的是当时的过程状态，不应直接解释为新主系列当前的完成状态。

当前主系列见 [Harness 系列入口](../../README.md)，按主题查询关键代码见 [源码与证据索引](../../appendices/Source_Index.md)。

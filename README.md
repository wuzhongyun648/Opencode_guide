# OpenCode 初学者教程

本项目包含 OpenCode 使用教程、内部 Agent Harness 架构专题和一份快速入门文档。使用教程以可执行步骤为主；Harness 系列面向第一次系统学习 Agent 架构的读者，通过一条完整学习与实践路径解释 OpenCode 如何把模型、工具、上下文和运行时组织成编码 Agent。

> 本教程主要依据 [OpenCode 官方中文文档](https://opencode.ai/docs/zh-cn/) 整理，模型信息同时参考 [Models.dev](https://models.dev/)。Harness 系列同时核对固定源码 commit `0e3474509aa5ad16afcf9c439785514d6443c6af`。最后统一核对日期：2026-08-19。

## 阅读顺序

首次使用 OpenCode 时先阅读 `OpenCode_Tutorial/` 中的 01-04。掌握基础操作后，可以阅读 05 了解 Skill、MCP 与 Plugin，再从 06 开始按顺序学习内部 Harness 架构。只需要快速了解基本操作时，可以直接阅读根目录中的 [OpenCode 快速入门](./Opencode_Tournament.md)。

### OpenCode 使用教程

| 章节 | 内容 | 适用场景 |
| --- | --- | --- |
| [01 OpenCode 安装与首次运行](./OpenCode_Tutorial/01_Installation.md) | 安装 CLI、Desktop 和 IDE Extension，完成首次启动 | 所有初次使用者 |
| [02 OpenCode 模型配置与部署](./OpenCode_Tutorial/02_Model_Deployment.md) | 接入模型 API、订阅、自定义接口和本地模型 | 完成安装后配置模型 |
| [03 OpenCode Server 部署](./OpenCode_Tutorial/03_OpenCode_Server_Deployment.md) | 本地监听、远程连接、Web、systemd 和 Docker 部署 | 远程开发、服务化和容器场景 |
| [04 OpenCode 常用命令与基础工作流](./OpenCode_Tutorial/04_Useful_Commands.md) | CLI、TUI 斜杠命令、快捷键和常见问题 | 日常使用 OpenCode |

### OpenCode Harness

| 章节 | 内容 | 适用场景 |
| --- | --- | --- |
| [05 OpenCode 扩展能力：Skill、MCP 与 Plugin](./Opencode_Harness/05_Enhancement.md) | Skill、MCP 和 Plugin 的原理与实践 | 掌握基础操作后按需扩展 OpenCode |
| [06 OpenCode Agent Harness 总览](./Opencode_Harness/06_Harness.md) | 从模型到编码 Agent 的整体架构、核心循环和系列地图 | 第一次建立 Harness 全局心智模型 |
| [07 Agent Loop](./Opencode_Harness/07_Agent_Loop.md) | 一次学习任务如何经过 Context、Model、Tool 和反馈持续推进 | 理解 Agent 为什么能够自主完成多步任务 |
| [08 Context Architecture](./Opencode_Harness/08_Context_Architecture.md) | 每轮模型输入由什么组成，以及上下文如何更新和收缩 | 理解模型当前知道什么、关注什么 |
| [09 Tools 与 Permission](./Opencode_Harness/09_Tools_and_Permission.md) | Tool 从定义、授权、执行到结果回传的完整生命周期 | 理解模型意图如何变成真实操作 |
| [10 Session 与 Persistence](./Opencode_Harness/10_Session_and_Persistence.md) | Session、Message、Part、Event、Compaction 与恢复边界 | 理解系统保存什么、下一轮如何继续 |
| [11 Agent 专业化与协作](./Opencode_Harness/11_Agent_Specialization_and_Collaboration.md) | Agent、Model、Plan、Todo、Task 与 Subagent 的职责边界 | 理解 OpenCode 如何分工和组织复杂任务 |
| [12 Runtime Boundary](./Opencode_Harness/12_Runtime_Boundary.md) | TUI、Server、Provider、Tool Runtime、事件返回与架构演进 | 理解 Harness 在哪里运行、各模块如何连接 |

建议严格按照 06-12 阅读。系列使用同一个学习场景：一位没有 Harness 背景的读者，从理解 Agent 与模型的区别开始，观察 OpenCode 如何回答问题、读取项目、执行一个低风险操作并利用结果继续下一轮。各篇只放理解当前主题所需的实现细节；完整术语、源码索引和版本证据作为附录与研究材料单独保存。

Harness 系列的定位、写作规则和审核标准见 [Harness 系列说明](./Opencode_Harness/README.md)。



## 使用说明

- 文档中的 `/path/to/project`、`ses_example`、域名、端口、模型 ID 和密钥均为示例，执行前必须替换为实际值。
- OpenCode 可以读取和修改项目文件、运行 Shell 命令并访问模型服务。执行写入、删除、安装、发布和 Git 远程操作前，应确认命令及影响范围。
- API Key、密码、Token 和私钥不得提交到 Git，也不应直接写入示例配置。
- OpenCode 和模型提供商更新较快。如果本机行为与教程不一致，优先以当前版本的 `opencode --help`、TUI 中的 `/help` 和官方文档为准。
- 模型可用地区、价格、额度和订阅规则由对应提供商决定，实际信息以提供商控制台为准。

## 参考资料

### OpenCode 官方入口

- [OpenCode 中文文档](https://opencode.ai/docs/zh-cn/)
- [OpenCode 下载页](https://opencode.ai/download)
- [OpenCode GitHub Releases](https://github.com/anomalyco/opencode/releases)

### 安装与使用

- [CLI 文档](https://opencode.ai/docs/zh-cn/cli/)
- [TUI 文档](https://opencode.ai/docs/zh-cn/tui/)
- [IDE 文档](https://opencode.ai/docs/zh-cn/ide/)
- [Web 文档](https://opencode.ai/docs/zh-cn/web/)
- [Server 文档](https://opencode.ai/docs/zh-cn/server/)
- [SDK 文档](https://opencode.ai/docs/zh-cn/sdk/)
- [Windows WSL 文档](https://opencode.ai/docs/zh-cn/windows-wsl/)
- [故障排除](https://opencode.ai/docs/zh-cn/troubleshooting/)

### 模型与配置

- [提供商文档](https://opencode.ai/docs/zh-cn/providers/)
- [模型文档](https://opencode.ai/docs/zh-cn/models/)
- [OpenCode Zen 文档](https://opencode.ai/docs/zh-cn/zen/)
- [OpenCode 生态系统](https://opencode.ai/docs/zh-cn/ecosystem/)
- [网络文档](https://opencode.ai/docs/zh-cn/network/)
- [权限文档](https://opencode.ai/docs/zh-cn/permissions/)
- [代理技能文档](https://opencode.ai/docs/zh-cn/skills/)
- [MCP 服务器文档](https://opencode.ai/docs/zh-cn/mcp-servers/)
- [插件文档](https://opencode.ai/docs/zh-cn/plugins/)
- [规则文档](https://opencode.ai/docs/zh-cn/rules/)
- [命令文档](https://opencode.ai/docs/zh-cn/commands/)
- [快捷键文档](https://opencode.ai/docs/zh-cn/keybinds/)
- [分享文档](https://opencode.ai/docs/zh-cn/share/)

### 外部资料

- [Models.dev](https://models.dev/)
- [Ollama 与 OpenCode 集成](https://docs.ollama.com/integrations/opencode)
- [Google Gemini Code Assist 消费者账号弃用说明](https://developers.google.com/gemini-code-assist/docs/deprecations/code-assist-individuals)

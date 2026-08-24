# OpenCode 学习文档

这个项目包含两条学习路线：一条讲 OpenCode 的安装、配置与日常使用，另一条讲 OpenCode Agent Harness 的内部运行机制。

每条路线都分为两个阅读层级：

```text
根目录同名 Markdown 文件：简略综述，帮助读者先建立整体认识
        ↓
同名文件夹中的 Markdown 文件：详细教程，按主题展开原理、操作与源码
```

因此，根目录的两篇文档不是详细章节的替代品，而是进入对应系列之前的总览。

## 两条学习路线

| 学习路线 | 简略综述 | 详细内容 | 适合解决的问题 |
| --- | --- | --- | --- |
| OpenCode 使用教程 | [Opencode_Tutorial.md](./Opencode_Tutorial.md) | [`OpenCode_Tutorial/`](./OpenCode_Tutorial/) | 如何安装 OpenCode、配置模型、部署 Server 和使用常用命令 |
| OpenCode Harness | [OpenCode_Harness.md](./OpenCode_Harness.md) | [`Opencode_Harness/`](./Opencode_Harness/) | OpenCode 如何组织 Context、调用 Tool、运行 Agent Loop，并扩展或协作 |

如果只是想快速了解，阅读根目录的对应综述即可。需要实际配置 OpenCode、理解实现细节或继续查看源码时，再进入对应文件夹。

> 文件名和目录名的大小写略有区别：`Opencode_Tutorial.md` 对应 `OpenCode_Tutorial/`，`OpenCode_Harness.md` 对应 `Opencode_Harness/`。在区分大小写的系统中，请以仓库中的实际名称为准。


## 详细文档目录

### OpenCode 使用教程

| 章节 | 主要内容 |
| --- | --- |
| [01 安装与首次运行](./OpenCode_Tutorial/01_Installation.md) | 安装 CLI、Desktop 和 IDE Extension，并完成首次启动 |
| [02 模型配置与部署](./OpenCode_Tutorial/02_Model_Deployment.md) | 接入模型 API、订阅账号、自定义接口和本地模型 |
| [03 Server 部署](./OpenCode_Tutorial/03_OpenCode_Server_Deployment.md) | 本地监听、Web、远程访问、systemd 和 Docker 部署 |
| [04 常用命令与基础工作流](./OpenCode_Tutorial/04_Useful_Commands.md) | CLI、TUI 命令、快捷键和常见问题 |

### OpenCode Harness

| 章节 | 主要内容 |
| --- | --- |
| [05 Skill、MCP 与 Plugin](./Opencode_Harness/05_Enhancement.md) | 三种扩展机制的职责、能力与使用方式 |
| [06 Harness 总览](./Opencode_Harness/06_Harness.md) | 模型如何在 Harness 中成为能够行动的编码 Agent |
| [07 Agent Loop](./Opencode_Harness/07_Agent_Loop.md) | 一次用户请求为什么可以产生多轮判断与行动 |
| [08 Context Architecture](./Opencode_Harness/08_Context_Architecture.md) | 每轮模型输入如何组装、更新、压缩和修剪 |
| [09 Tools 与 Permission](./Opencode_Harness/09_Tools_and_Permission.md) | 模型意图如何经过授权和执行变成真实操作 |
| [10 Session 与 Persistence](./Opencode_Harness/10_Session_and_Persistence.md) | Message、Part、Event、Compaction 与恢复边界 |
| [11 Agent 专业化与协作](./Opencode_Harness/11_Agent_Specialization_and_Collaboration.md) | Agent、Plan、Todo、Task 与 Subagent 如何分工 |
| [12 Runtime Boundary](./Opencode_Harness/12_Runtime_Boundary.md) | TUI、Server、Provider、Tool Runtime 和事件通道如何连接 |


## 文档边界与使用说明

- 使用教程主要依据 [OpenCode 官方中文文档](https://opencode.ai/docs/zh-cn/) 和 [Models.dev](https://models.dev/) 整理；具体价格、额度、模型支持情况和命令行为可能随版本变化。
- 遇到教程与当前版本不一致时，优先检查本机的 `opencode --help`、TUI 中的 `/help`、[OpenCode 官方文档](https://opencode.ai/docs/zh-cn/)以及对应版本源码。

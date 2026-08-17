# OpenCode 初学者教程

本项目面向首次接触 OpenCode 的初级开发者，介绍从安装、模型接入到常用命令和进阶能力的完整使用流程。教程以可执行步骤为主，并补充必要的安全提示、验证方法和故障排查说明。

> 本教程主要依据 [OpenCode 官方中文文档](https://opencode.ai/docs/zh-cn/) 整理，模型信息同时参考 [Models.dev](https://models.dev/)。最后统一核对日期：2026-08-17。

## 阅读顺序

建议初次使用时按照编号顺序阅读。只在本机进行日常开发时，可以跳过 Server 部署章节。

| 章节 | 内容 | 适用场景 |
| --- | --- | --- |
| [01 OpenCode 安装与首次运行](./01_Installation.md) | 安装 CLI、Desktop 和 IDE Extension，完成首次启动 | 所有初次使用者 |
| [02 OpenCode 模型配置与部署](./02_Model_Deployment.md) | 接入模型 API、订阅、自定义接口和本地模型 | 完成安装后配置模型 |
| [03 OpenCode Server 部署](./03_OpenCode_Server_Deployment.md) | 本地监听、远程连接、Web、systemd 和 Docker 部署 | 远程开发、服务化和容器场景 |
| [04 OpenCode 常用命令与基础工作流](./04_common_commands.md) | CLI、TUI 斜杠命令、快捷键和常见问题 | 日常使用 OpenCode |
| [05 OpenCode 增强功能](./05_Enhancement.md) | OpenCode 进阶能力与扩展配置 | 掌握基础操作后继续学习 |
| [06 Harness](./06_Harness.md) | Harness 相关内容 | 需要进一步集成和自动化时 |

## 推荐学习路径

1. 按第 01 章安装 OpenCode，并确认 `opencode --version` 可以正常运行。
2. 按第 02 章连接一个模型提供商，完成一次只读模型调用。
3. 阅读第 04 章，区分普通终端 CLI 命令和 TUI 斜杠命令。
4. 只有在源码位于远程主机、需要 Web/SDK 接入或已有容器运维体系时，再阅读第 03 章。
5. 掌握基础操作后，再学习第 05、06 章的增强与自动化内容。

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
- [规则文档](https://opencode.ai/docs/zh-cn/rules/)
- [命令文档](https://opencode.ai/docs/zh-cn/commands/)
- [快捷键文档](https://opencode.ai/docs/zh-cn/keybinds/)
- [分享文档](https://opencode.ai/docs/zh-cn/share/)

### 外部资料

- [Models.dev](https://models.dev/)
- [Ollama 与 OpenCode 集成](https://docs.ollama.com/integrations/opencode)
- [Google Gemini Code Assist 消费者账号弃用说明](https://developers.google.com/gemini-code-assist/docs/deprecations/code-assist-individuals)

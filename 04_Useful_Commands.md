# OpenCode 常用命令与基础工作流

> 本文介绍 OpenCode TUI、CLI 常用命令和一套适合初级开发者的基础开发工作流。安装 OpenCode 见 [01_Installation.md](./01_Installation.md)，模型配置见 [02_Model_Deployment.md](./02_Model_Deployment.md)，Server 和远程连接见 [03_OpenCode_Server_Deployment.md](./03_OpenCode_Server_Deployment.md)。

## 1. 基础操作介绍

使用 OpenCode 时，会同时接触普通终端命令、TUI 斜杠命令和自然语言任务。它们的执行位置和用途不同。

| 类型 | 执行位置 | 示例 | 用途 |
| --- | --- | --- | --- |
| CLI 命令 | 普通终端 | `opencode run "解释这个项目"` | 启动、认证、自动化和管理 OpenCode |
| TUI 斜杠命令 | OpenCode 输入框 | `/models` | 操作 OpenCode 界面、模型和会话 |
| 文件引用 | OpenCode 输入框 | `解释 @src/main.ts` | 将指定文件加入当前消息的上下文 |
| Shell 命令 | OpenCode 输入框 | `!git status` | 执行本地命令，并把输出加入会话 |
| 自然语言任务 | OpenCode 输入框 | `修复登录接口的空指针问题` | 让代理分析、解释或修改项目 |

### 1.1 普通终端中的 CLI 命令

CLI 命令以 `opencode` 开头，在 Bash、PowerShell 或 IDE 集成终端中执行。例如：

```bash
opencode --version
opencode models
opencode run "概括当前项目的主要技术栈"
```

不带子命令运行 `opencode` 时，默认打开 TUI：

```bash
opencode
```

### 1.2 TUI 中的斜杠命令

启动 TUI 后，以 `/` 开头的内容是 OpenCode 界面命令，不是操作系统命令：

```text
/help
/models
/sessions
```

如果在普通终端中直接输入 `/models`，Shell 不会把它识别为 OpenCode 命令。

### 1.3 TUI 中的文件引用与 Shell 命令

在消息中输入 `@` 可以搜索并引用当前项目中的文件：

```text
请解释 @src/main.ts 的启动流程，不要修改文件。
```

以 `!` 开头的消息会直接作为 Shell 命令执行：

```text
!git status --short
```

命令输出会加入当前会话，模型可以基于结果继续分析。`!` 不是让模型“建议一条命令”，而是在 OpenCode 所在环境中实际执行命令，因此使用删除、覆盖、安装和 Git 写操作前必须确认影响。

### 1.4 命令速查表

| 目标 | 推荐操作 |
| --- | --- |
| 打开当前项目 | `opencode` |
| 打开指定项目 | `opencode /path/to/project` |
| 继续最近会话 | `opencode --continue` |
| 查看 TUI 帮助 | `/help` |
| 引用文件 | `@文件路径` |
| 执行 Shell 命令 | `!命令` |
| 选择模型 | `/models` |
| 新建会话 | `/new` |
| 切换会话 | `/sessions` |
| 压缩上下文 | `/compact` |
| 撤销上一次消息及其文件修改 | `/undo` |
| 中断正在运行的任务 | `Esc` |
| 执行一次性任务 | `opencode run "任务"` |
| 查看调试日志 | `opencode --print-logs --log-level DEBUG` |

## 2. 常用 CLI 命令

### 2.1 查看版本和帮助

```bash
opencode --version
opencode --help
```

查看某个子命令的参数：

```bash
opencode run --help
opencode session --help
```

### 2.2 管理模型凭据

登录提供商：

```bash
opencode auth login
```

查看已保存的认证提供商：

```bash
opencode auth list
```

移除某个提供商的认证信息：

```bash
opencode auth logout
```

完整的提供商选择、凭据位置和环境变量配置见 [02_Model_Deployment.md](./02_Model_Deployment.md)。

### 2.3 查看可用模型

```bash
opencode models
```

按提供商筛选：

```bash
opencode models deepseek
```

提供商新增模型但本地列表尚未更新时，可以刷新模型缓存：

```bash
opencode models --refresh
```

查看费用等更多模型元数据：

```bash
opencode models --verbose
```

### 2.4 使用 `opencode run` 执行一次性任务

`opencode run` 不打开完整 TUI，适合一次性问答、脚本和自动化任务：

```bash
opencode run "概括当前项目的主要技术栈，不要修改文件"
```

命令同样具有读取文件、调用工具和修改项目的能力，不能因为没有打开 TUI 就把它视为只读命令。自动化场景应明确任务边界并配置适当权限。

### 2.5 附加文件、指定模型和输出格式

附加文件到消息：

```bash
opencode run --file src/main.ts "解释这个文件的启动流程"
```

临时指定模型：

```bash
opencode run --model deepseek/deepseek-chat "检查当前项目的错误处理"
```

输出原始 JSON 事件，供程序处理：

```bash
opencode run --format json "概括当前项目"
```

文件路径和模型 ID 都应替换为当前项目中的实际值。脚本处理 `--format json` 输出时，不应依赖供人阅读的默认格式。

### 2.6 查看会话列表

```bash
opencode session list
```

只显示最近 10 个会话：

```bash
opencode session list --max-count 10
```

输出 JSON：

```bash
opencode session list --format json
```

### 2.7 查看 Token 和费用统计

```bash
opencode stats
```

查看最近 7 天并显示模型明细：

```bash
opencode stats --days 7 --models 10
```

统计结果用于了解 OpenCode 记录的 Token 和费用使用情况，实际账单仍应以模型提供商控制台为准。

### 2.8 导出与导入会话

```bash
opencode export --sanitize ses_example
opencode import session.json
```

CLI 导出的 JSON 适合备份和迁移；`--sanitize` 适合需要交给他人的导出文件，但仍应人工检查脱敏结果。需要便于阅读的当前对话内容时，在 TUI 中使用 `/export`。

### 2.9 更新 OpenCode

更新到最新版：

```bash
opencode upgrade
```

更新到指定版本：

```bash
opencode upgrade v1.2.3
```

`v1.2.3` 只是格式示例，应替换为真实存在的版本。团队环境升级前应查看 Release Notes，并避免在进行中的重要任务中途升级。

### 2.10 输出调试日志

启动 TUI 并把调试日志输出到标准错误：

```bash
opencode --print-logs --log-level DEBUG
```

一次性任务也可以使用全局日志参数：

```bash
opencode --print-logs --log-level DEBUG run "概括当前项目"
```

调试日志可能包含路径、请求信息和项目上下文，提交 Issue 或发送给他人前应移除敏感内容。

`opencode serve`、`opencode web` 和 `opencode attach` 的用途及安全配置见 [03_OpenCode_Server_Deployment.md](./03_OpenCode_Server_Deployment.md)，本章不再重复展开。

## 3. 常用 TUI 斜杠命令

### 3.1 查看帮助：`/help`

```text
/help
```

打开命令面板并查看当前版本实际支持的操作。官方文档和 OpenCode 版本可能存在差异，遇到命令或快捷键不一致时，以本机 `/help` 和 `opencode --help` 为准。

### 3.2 初始化项目规则：`/init`

```text
/init
```

分析当前项目并创建或更新 `AGENTS.md`。该文件用于记录项目结构、开发命令和代理需要遵守的规则。执行后应检查生成的差异，确认命令和约定符合项目实际情况，再提交到 Git。

### 3.3 连接提供商：`/connect`

```text
/connect
```

选择模型提供商并完成 API Key 或 OAuth 认证。认证信息可能包含敏感凭据，具体配置和存储位置见 [02_Model_Deployment.md](./02_Model_Deployment.md)。

### 3.4 选择模型：`/models`

```text
/models
```

列出并切换当前模型。该操作影响当前会话使用的模型；设置项目或全局默认模型见 [02_Model_Deployment.md](./02_Model_Deployment.md)。

### 3.5 新建和切换会话：`/new`、`/sessions`

开始一个空白会话：

```text
/new
```

`/clear` 是它的别名。查看已有会话并切换：

```text
/sessions
```

`/resume` 和 `/continue` 是 `/sessions` 的别名。一个独立问题通常使用一个会话，避免不相关任务共享大量上下文。

### 3.6 压缩上下文：`/compact`

```text
/compact
```

将较长的当前会话压缩为摘要，以减少后续请求携带的上下文。`/summarize` 是它的别名。压缩可能丢失细节，关键约束应写入 `AGENTS.md`、任务描述或项目文档，而不是只依赖很早的聊天记录。

### 3.7 查看工具详情：`/details`

```text
/details
```

切换工具调用详情的显示。排查模型读取了哪些文件、执行了什么命令或为何失败时，可以打开详情。

### 3.8 撤销和重做：`/undo`、`/redo`

撤销最后一条用户消息、后续响应及其文件修改：

```text
/undo
```

恢复刚才撤销的内容：

```text
/redo
```

OpenCode 在内部使用 Git 管理这些文件变化，因此项目必须是 Git 仓库。执行前先查看 `git status`，避免把自己尚未保存或未提交的修改误认为 OpenCode 产生的变化。`/undo` 不是 Git 历史管理命令，不能替代正常的提交、分支和代码审查流程。

### 3.9 导出和分享：`/export`、`/share`、`/unshare`

将当前对话导出为 Markdown，并使用默认编辑器打开：

```text
/export
```

`/export` 使用 `EDITOR` 环境变量指定的编辑器。VS Code、Cursor 等图形编辑器通常需要配置 `--wait`，例如：

```bash
export EDITOR="code --wait"
```

创建当前会话的分享链接：

```text
/share
```

取消分享：

```text
/unshare
```

分享前必须检查对话、文件内容、命令输出和错误日志中是否包含源码、密钥、内部地址或个人信息。取消分享不能保证已经访问或复制过内容的人删除其副本。

### 3.10 退出 OpenCode：`/exit`

```text
/exit
```

`/quit` 和 `/q` 是它的别名。也可以使用默认快捷键 `Ctrl+X`，再按 `Q`。

## 4. 常用快捷键

### 4.1 理解前导键 `Ctrl+X`

OpenCode 默认使用 `Ctrl+X` 作为前导键。`Ctrl+X N` 表示先按 `Ctrl+X`，松开后再按 `N`，不是同时按下三个按键。

快捷键可以在 `tui.json` 中修改，因此下表只表示默认配置。当前环境的实际快捷键可以通过 `/help` 查看。

### 4.2 模型和代理切换

| 操作 | 默认快捷键 |
| --- | --- |
| 打开模型列表 | `Ctrl+X M` |
| 循环最近使用的模型 | `F2` |
| 反向循环最近使用的模型 | `Shift+F2` |
| 打开代理列表 | `Ctrl+X A` |
| 切换代理 | `Tab` |
| 反向切换代理 | `Shift+Tab` |
| 切换模型变体 | `Ctrl+T` |

模型变体通常表示提供商支持的不同推理级别。`/thinking` 只控制推理块是否显示，不负责开启或关闭模型的推理能力。

### 4.3 新建、切换和压缩会话

| 操作 | 默认快捷键 | 对应命令 |
| --- | --- | --- |
| 新建会话 | `Ctrl+X N` | `/new` |
| 打开会话列表 | `Ctrl+X L` | `/sessions` |
| 压缩当前会话 | `Ctrl+X C` | `/compact` |
| 撤销 | `Ctrl+X U` | `/undo` |
| 重做 | `Ctrl+X R` | `/redo` |
| 导出当前会话 | `Ctrl+X X` | `/export` |

### 4.4 中断当前任务

模型回答或工具执行仍在进行时，按 `Esc` 可以请求中断当前任务。如果外部命令没有立即停止，应等待 OpenCode 返回状态，不要连续启动相同命令。

输入框中按 `Ctrl+C` 默认清空输入；退出应用优先使用 `/exit` 或 `Ctrl+X Q`，避免混淆“清空输入”和“退出程序”。

### 4.5 输入多行提示词

默认可以使用以下组合在输入框中换行：

- `Shift+Enter`
- `Ctrl+Enter`
- `Alt+Enter`
- `Ctrl+J`

部分终端不会正确发送 `Shift+Enter`，此时可以先使用 `Ctrl+J`，或按照 OpenCode 快捷键文档配置终端。

### 4.6 快捷键速查表

| 操作 | 默认快捷键 |
| --- | --- |
| 打开命令列表 | `Ctrl+P` |
| 打开帮助 | `Ctrl+X H` |
| 打开外部编辑器 | `Ctrl+X E` |
| 显示工具详情 | `Ctrl+X D` |
| 复制消息 | `Ctrl+X Y` |
| 退出 | `Ctrl+X Q` |
| 提交输入 | `Enter` |
| 中断任务 | `Esc` |

## 5. 常见问题

### 5.1 `@` 找不到文件

依次确认：

- OpenCode 是否从正确的项目目录启动。
- 文件是否位于当前工作目录之下。
- 输入的文件名或路径片段是否正确。
- 文件是否刚刚创建，TUI 是否需要重新启动后刷新。

可以使用 `!pwd` 和只读目录命令确认当前环境，但不要通过一次引用整个项目来绕过路径问题。

### 5.2 `!` 命令执行失败

检查命令是否已安装、当前账户是否有权限、工作目录是否正确，以及项目依赖是否已经准备完成。OpenCode Server 或容器中的命令实际在 Server 或容器环境执行，不一定拥有客户端本机的工具，参见 [03_OpenCode_Server_Deployment.md](./03_OpenCode_Server_Deployment.md)。

### 5.3 `/undo` 或 `/redo` 无法使用

确认当前项目是 Git 仓库，并且 `/redo` 只在执行 `/undo` 后使用。外部 Shell 命令造成的数据库、网络或远程 Git 变化不属于普通文件撤销范围。

### 5.4 会话上下文过长

先让模型总结当前状态并检查总结，再执行 `/compact`。任务已经切换时使用 `/new`。长期规则和关键结论应写入项目文件，不要只保存在聊天记录中。

### 5.5 继续会话后找不到之前的内容

运行 `opencode session list` 确认会话是否属于当前项目，再使用 `opencode --session <session-id>` 指定会话。`--continue` 只继续当前项目最近的会话，不保证是用户想到的某个历史会话。

### 5.6 模型没有按要求修改文件

检查当前代理和权限是否允许写入，任务中是否明确要求修改，以及目标路径是否位于当前项目。先让模型说明准备修改的文件；如果它无法读取项目，优先修复工作目录、权限或 Server 挂载问题。

### 5.7 OpenCode 修改了不相关文件

立即检查 `git status --short` 和 `git diff`，不要继续叠加修改。可以明确要求恢复超出范围的部分；如果整轮修改都不需要，并且确认不会影响已有工作，可以使用 `/undo`。后续任务应明确允许修改的目录、文件和禁止事项。

### 5.8 快捷键与终端快捷键冲突

先使用对应斜杠命令完成操作，再通过 `/help` 确认当前快捷键。`Shift+Enter` 无法换行时可以使用 `Ctrl+J`。需要长期调整时，在 `tui.json` 中自定义快捷键，不要修改用于 Server 和运行时配置的 `opencode.json`。

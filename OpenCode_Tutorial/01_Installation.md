# OpenCode 安装与首次运行

## 1. Opencode简介

### 1.1 产品形态与使用入口

OpenCode 是一个开源 AI 编码代理，主要提供以下使用入口：


| 入口                | 说明                                      | 适用场景             | 安装建议      |
| ----------------- | --------------------------------------- | ---------------- | --------- |
| Terminal（CLI/TUI） | CLI 是命令行程序，直接运行时会打开终端界面 TUI             | 日常开发、远程终端、脚本任务   | 默认选择      |
| Desktop（GUI）      | 图形桌面应用                                  | 偏好图形界面、多项目和多会话管理 | 按操作系统下载安装 |
| IDE Extension     | VS Code、Cursor、Windsurf、VSCodium 等编辑器扩展 | 在编辑器中携带文件和选区上下文  | 先安装 CLI   |


此外，CLI还提供 Web 和 Server 2种运行模式。相关内容见 [03_OpenCode_Server_Deployment.md](./03_OpenCode_Server_Deployment.md)。

### 1.2 官方安装方式与工具

下表汇总 OpenCode 官网当前列出的安装工具和分发渠道。它们安装的是相同产品的不同制品或入口，不需要全部安装；同一台机器通常只选择一种 CLI 安装方式。


| 工具               | 所属生态                             | 安装命令                                                        | 主要用途                                            |
| ---------------- | -------------------------------- | ----------------------------------------------------------- | ----------------------------------------------- |
| 官方安装脚本           | macOS、Linux、WSL                  | `curl -fsSL https://opencode.ai/install | bash`             | 快速安装最新版 OpenCode CLI，普通开发环境的推荐方式                |
| npm              | Node.js/JavaScript               | `npm install -g opencode-ai`                                | 在已有 Node.js 工具链中全局安装 CLI                        |
| Bun              | JavaScript/TypeScript            | `bun install -g opencode-ai`                                | 使用 Bun 的包管理器全局安装 CLI；Windows 暂不推荐               |
| pnpm             | Node.js/JavaScript               | `pnpm install -g opencode-ai`                               | 在 pnpm 工具链中全局安装 CLI，并复用其依赖存储机制                  |
| Yarn             | Node.js/JavaScript               | `yarn global add opencode-ai`                               | 在 Yarn 工具链中全局安装 CLI                             |
| Homebrew Tap     | macOS、Linux                      | `brew install anomalyco/tap/opencode`                       | 通过 OpenCode 官方 tap 安装和升级 CLI                    |
| Homebrew Formula | macOS、Linux                      | `brew install opencode`                                     | 通过 Homebrew 社区维护的 formula 安装 CLI；版本更新可能慢于官方 tap |
| pacman           | Arch Linux                       | `sudo pacman -S opencode`                                   | 安装 Arch Linux 仓库中的稳定版 CLI                       |
| AUR/paru         | Arch Linux                       | `paru -S opencode-bin`                                      | 从 AUR 安装较新的预编译 CLI                              |
| Chocolatey       | Windows                          | `choco install opencode`                                    | 在 Windows 原生环境中安装 CLI                           |
| Scoop            | Windows                          | `scoop install opencode`                                    | 在 Windows 原生开发环境中安装 CLI                         |
| Mise             | 跨平台开发工具管理                        | `mise use -g github:anomalyco/opencode`                     | 通过 Mise 管理全局 CLI 版本                             |
| Docker           | 容器                               | `docker run -it --rm ghcr.io/anomalyco/opencode`            | 临时体验、隔离执行或 CI/CD；长期使用需要挂载项目和数据卷                 |
| GitHub Releases  | 手动二进制分发                          | [下载对应平台压缩包](https://github.com/anomalyco/opencode/releases) | 离线分发、固定版本或不使用包管理器的环境                            |
| Homebrew Cask    | macOS Desktop                    | `brew install --cask opencode-desktop`                      | 安装 OpenCode Desktop 图形应用                        |
| DMG              | macOS Desktop                    | [OpenCode 下载页](https://opencode.ai/download)                | 手动安装 Apple Silicon 或 Intel 版 Desktop            |
| NSIS             | Windows Desktop                  | [OpenCode 下载页](https://opencode.ai/download)                | 安装 Windows x64 Desktop                          |
| DEB              | Debian、Ubuntu                    | `sudo apt install /path/to/opencode-desktop.deb`            | 安装 Debian 系 Linux Desktop                       |
| RPM              | Fedora、RHEL 系                    | `sudo dnf install /path/to/opencode-desktop.rpm`            | 安装 RPM 系 Linux Desktop                          |
| IDE 自动安装         | VS Code 兼容 IDE                   | 在 IDE 集成终端运行 `opencode`                                     | 自动安装并启用 OpenCode IDE Extension                  |
| IDE 扩展商店         | VS Code、Cursor、Windsurf、VSCodium | 扩展商店搜索 `OpenCode`                                           | 手动安装 OpenCode IDE Extension                     |


> `pip` 是 Python 包管理器，不是 OpenCode 官方安装方式。表格中的 `/path/to/` 是占位路径，执行前需要替换为实际安装包路径。



### 1.3 安装前准备

安装前需要了解：

- OpenCode 本身不提供模型推理能力，首次对话前需要连接一个模型提供商。
- 提供商通常需要 API Key、OAuth 登录或有效订阅，部分服务会产生费用。
- OpenCode 可以在本地运行，但模型推理通常发生在远程提供商。提示词和必要的项目上下文可能被发送给模型服务。
- Windows 用户优先在 WSL 中运行 OpenCode，以获得更好的文件系统性能和工具兼容性。

模型选择和详细配置见 [02_Model_Deployment.md](./02_Model_Deployment.md)。

## 2. 安装 OpenCode Terminal（CLI/TUI）

macOS、Linux 和 WSL 用户推荐使用官方安装脚本；已统一使用 Homebrew 或 Node.js 工具链的团队，可选择对应的包管理器。

### 2.1 官方安装脚本

适用于 macOS、Linux 和 WSL。

```bash
curl -fsSL https://opencode.ai/install | bash
```



### 2.2 Node.js 包管理器

适用于已经使用 Node.js 或 Bun 管理全局开发工具的环境。选择一种命令执行即可：

```bash
npm install -g opencode-ai
```

```bash
pnpm install -g opencode-ai
```

```bash
bun install -g opencode-ai
```

```bash
yarn global add opencode-ai
```

安装后如果找不到 `opencode`，检查对应包管理器的全局可执行目录是否已加入 `PATH`。

### 2.3 Homebrew

适用于通过 Homebrew 管理开发工具的 macOS 或 Linux 环境。

```bash
brew install anomalyco/tap/opencode
```



### 2.4 Arch Linux

安装仓库中的稳定版本：

```bash
sudo pacman -S opencode
```

通过 AUR 安装最新二进制版本：

```bash
paru -S opencode-bin
```



### 2.5 Windows



#### 推荐：WSL

尚未安装 WSL 时，请参考 [Microsoft WSL 官方安装指南](https://learn.microsoft.com/zh-cn/windows/wsl/install)。完成 WSL 配置后，在 WSL 终端中执行第 2.1 节的官方安装脚本。

#### Windows 原生安装

根据已有工具链选择一种方式：

```powershell
choco install opencode
```

```powershell
scoop install opencode
```

```powershell
npm install -g opencode-ai
```

```powershell
mise use -g github:anomalyco/opencode
```

OpenCode 官方当前不建议在 Windows 上通过 Bun 安装。

### 2.6 二进制文件

该方式适用于离线分发、版本锁定或不希望引入包管理器的环境。

1. 打开 [OpenCode Releases](https://github.com/anomalyco/opencode/releases)。
2. 根据操作系统和 CPU 架构选择对应的 CLI 压缩包。
3. 使用发布页提供的 SHA-256 摘要校验文件。
4. 解压后，将 `opencode` 可执行文件放入已加入 `PATH` 的目录。

企业内部分发时，建议固定版本并记录制品来源和摘要。

### 2.7 Docker

Docker 方式适合临时体验、隔离执行或流水线，不建议作为个人首次安装的默认选项。

以下示例适用于 macOS、Linux 和 WSL 的 Bash：

```bash
docker run --rm -it \
  -v "$PWD:/workspace" \
  -w /workspace \
  ghcr.io/anomalyco/opencode
```

该命令挂载当前目录，但不会持久化容器中的认证信息和会话。长期运行、数据持久化及 Server 部署见 [03_OpenCode_Server_Deployment.md](./03_OpenCode_Server_Deployment.md)。

### 2.8 验证 CLI 安装

```bash
opencode --version
opencode --help
```

能够输出版本号和帮助信息，说明 CLI 已安装并进入当前用户的 `PATH`。这一步只验证安装，不代表模型已经可以调用。

## 3. 安装 OpenCode Desktop

桌面端app适合偏好图形界面以及需要管理多个项目或会话的用户。

### 3.1 macOS Homebrew

```bash
brew install --cask opencode-desktop
```



### 3.2 下载安装包

从 [OpenCode Desktop 下载页](https://opencode.ai/download) 获取对应制品：


| 平台                  | 安装包        |
| ------------------- | ---------- |
| macOS Apple Silicon | DMG（ARM64） |
| macOS Intel         | DMG（x64）   |
| Windows x64         | NSIS 安装程序  |
| Debian/Ubuntu x64   | DEB        |
| Fedora/RHEL 系 x64   | RPM        |


Linux 安装示例中的 `/path/to/` 表示安装包实际所在目录，需要替换后执行：

```bash
sudo apt install /path/to/opencode-desktop.deb
```

```bash
sudo dnf install /path/to/opencode-desktop.rpm
```

具体文件名以下载页或 Release 资产名称为准。Windows 版 Desktop 依赖 Microsoft Edge WebView2 Runtime。

## 4. 安装 OpenCode IDE Extension

IDE 扩展提供快捷启动、选区上下文和文件引用等能力，实际代理进程仍由 OpenCode CLI 提供。

### 4.1 自动安装

1. 先按第 2 章安装 OpenCode CLI。
2. 在 IDE 的集成终端中运行：

```bash
opencode
```

OpenCode 会检测兼容 IDE 并自动安装扩展。

### 4.2 手动安装

在 IDE 扩展商店中搜索 `OpenCode` 并选择安装。

如果自动安装失败，确认编辑器对应的 CLI 命令已加入 `PATH`。

## 5. 启动Opencode



### 5.1 进入项目并启动

下面的 `/path/to/project` 是占位路径，需要替换为实际项目目录：

```bash
cd /path/to/project
opencode
```



### 5.2 连接模型提供商

在 TUI 中执行：

```text
/connect
```

根据提示选择提供商并完成 API Key 或 OAuth 认证。也可以使用 CLI：

```bash
opencode auth login
```

提供商选择、凭据位置和模型检查方法见 [02_Model_Deployment.md](./02_Model_Deployment.md)。

### 5.3 验证模型调用

认证完成后，先发送一个只读问题，例如：

```text
请概括这个项目使用的主要技术栈，不要修改任何文件。
```

能够正常收到回答，才表示 OpenCode 和模型提供商均已配置成功。

### 5.4 初始化项目规则

模型调用成功后，可以执行：

```text
/init
```

该命令会分析项目并生成 `AGENTS.md`。

## 6. 常见安装问题



### 6.1 `opencode: command not found`

- 重新打开终端，使 shell 重新加载环境变量。
- 检查安装目录是否已加入 `PATH`。
- npm 用户检查 npm 全局可执行目录；Homebrew 用户检查 `brew --prefix` 对应路径。



### 6.2 OpenCode 无法启动

```bash
opencode --print-logs --log-level DEBUG
```

macOS/Linux 日志默认位于 `~/.local/share/opencode/log/`。Windows 日志默认位于 `%USERPROFILE%\.local\share\opencode\log`。

### 6.3 已启动但无法调用模型

```bash
opencode auth list
opencode models
```

如果提供商未认证、模型列表为空或调用报错，转到 [02_Model_Deployment.md](./02_Model_Deployment.md) 检查模型配置。

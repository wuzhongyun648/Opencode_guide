# OpenCode 快速入门

OpenCode 是一个开源 AI 编码代理，提供终端界面、桌面应用和 IDE 扩展。本文只介绍安装、模型配置和基础使用；完整功能以 [OpenCode 官方文档](https://opencode.ai/docs/zh-cn/) 为准。

## 1. OpenCode 简介与安装

### 1.1 产品形态与安装

> 官方文档推荐使用安装脚本安装 OpenCode，并列出了 npm、Homebrew、Windows、Docker 和二进制文件等其他方式。Windows 用户推荐使用 WSL。详见 [OpenCode 简介与安装](https://opencode.ai/docs/zh-cn/)。

macOS、Linux 或 WSL：

```bash
curl -fsSL https://opencode.ai/install | bash
```

Windows ：

```powershell
npm install -g opencode-ai
```

桌面应用可从 [OpenCode 下载页](https://opencode.ai/download) 获取；IDE 扩展的安装和使用方式见 [IDE 文档](https://opencode.ai/docs/zh-cn/ide/)。

### 1.2 验证安装并启动

```bash
opencode --version
cd /path/to/project
opencode
```

`/path/to/project` 是项目目录的占位符。能够输出版本号并进入终端界面，说明 OpenCode 已正确安装；此时还需要配置模型才能进行对话。

## 2. 模型配置

OpenCode 支持多种云端 API、订阅账号、模型网关和本地模型服务。模型配置主要涉及提供商凭据、API 地址和模型 ID。

### 2.1 模型提供商与 API

在 OpenCode 中执行 `/connect`，选择提供商并按提示输入 API Key；也可以执行 `opencode auth login`。不同提供商的认证方式和环境变量均以 [提供商文档](https://opencode.ai/docs/zh-cn/providers/) 为准。

> API Key、Token 和其他凭据不得写入文档、提交到 Git，或直接保存在可公开的项目配置中。



### 2.2 订阅账号接入

OpenCode 支持的订阅账号及其认证方式可能变化。使用 ChatGPT Plus/Pro、Claude Pro/Max、GitHub Copilot 或其他订阅时，直接按照 [提供商文档](https://opencode.ai/docs/zh-cn/providers/) 中对应提供商的步骤操作。

### 2.3 本地模型服务配置

本节假设本地推理服务和模型已经部署完成，只说明如何将现有服务接入 OpenCode，不包含 Ollama、LM Studio、llama.cpp 或模型本身的安装与部署。

#### 2.3.1 配置前提

接入前需要确认：

- OpenCode 所在环境能够访问本地推理服务。
- 推理服务提供 OpenAI-compatible API。
- 已知服务的 Base URL 和实际模型 ID。
- 所用模型支持当前任务需要的上下文长度和工具调用能力。

可以先查询服务返回的模型列表：

```bash
curl "<BASE_URL>/models"
```

其中 `<BASE_URL>` 通常包含 `/v1`，例如 `http://127.0.0.1:11434/v1`。应以实际推理服务提供的地址为准。

#### 2.3.2 配置参数


| 占位符               | 含义                             | 示例形式                         |
| ----------------- | ------------------------------ | ---------------------------- |
| `<PROVIDER_ID>`   | OpenCode 中唯一的提供商 ID            | `local-model`                |
| `<PROVIDER_NAME>` | 界面中显示的提供商名称                    | `Local Model`                |
| `<BASE_URL>`      | 推理服务的 OpenAI-compatible API 地址 | `http://127.0.0.1:<PORT>/v1` |
| `<MODEL_ID>`      | 服务实际返回的模型 ID                   | `your-model-id`              |
| `<MODEL_NAME>`    | 界面中显示的模型名称                     | `Your Model`                 |


如果 OpenCode 与推理服务位于不同容器或主机，不能直接使用 `127.0.0.1`，应填写 OpenCode 实际可访问的地址。

#### 2.3.3 配置接口

如果推理服务提供 `POST /v1/chat/completions`，使用 `@ai-sdk/openai-compatible`。在项目根目录创建 `opencode.json`，将配置写入全局文件 `~/.config/opencode/opencode.json`：

```json
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "<PROVIDER_ID>": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "<PROVIDER_NAME>",
      "options": {
        "baseURL": "<BASE_URL>"
      },
      "models": {
        "<MODEL_ID>": {
          "name": "<MODEL_NAME>"
        }
      }
    }
  },
  "model": "<PROVIDER_ID>/<MODEL_ID>"
}
```

替换全部占位符后重启 OpenCode。`provider` 中的键必须与 `<PROVIDER_ID>` 一致，`models` 中的键必须与推理服务返回的 `<MODEL_ID>` 完全一致。

配置文件作用域和加载优先级见 [OpenCode 配置文档](https://opencode.ai/docs/zh-cn/config/)，Ollama、LM Studio 和 llama.cpp 的官方示例见 [提供商文档](https://opencode.ai/docs/zh-cn/providers/)。

#### 2.3.4 验证模型



##### 第一步：验证模型列表

```bash
curl "<BASE_URL>/models"
```

确认请求没有出现连接错误、服务返回目标模型，并且返回的模型 ID 与配置完全一致。

##### 第二步：直接验证对话接口

```bash
curl "<BASE_URL>/chat/completions" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "<MODEL_ID>",
    "messages": [
      {
        "role": "user",
        "content": "只回复 OK"
      }
    ],
    "stream": false
  }'
```

接口返回真实模型响应，证明模型已经加载且 Chat Completions 接口可用。

##### 第三步：验证 OpenCode 是否加载配置

```bash
opencode models <PROVIDER_ID>
```

确认列表中出现：

```text
<PROVIDER_ID>/<MODEL_ID>
```

如果没有出现，检查配置文件位置和 JSON 格式，并确认提供商 ID、模型 ID 与实际值一致。修改配置后需要退出并重新启动 OpenCode。

##### 第四步：验证普通对话

启动 OpenCode：

```bash
opencode
```

在 `/models` 中选择 `<PROVIDER_ID>/<MODEL_ID>`，然后发送：

```text
请概括当前项目的目录结构，不要修改任何文件。
```


##### 第五步：验证工具调用

发送一个只读工具任务：

```text
请实际调用工具列出当前项目根目录中的文件，并说明每个文件可能的用途。不要修改任何文件。
```

完成以上验证后，才能确认接口类型、模型 ID、OpenCode 配置、基础对话和工具调用均可用。模型选择和默认模型配置见 [模型文档](https://opencode.ai/docs/zh-cn/models/)。

## 3. 常用命令与基础工作流



### 3.1 常用命令


| 命令 | 输入位置 | 用途 |
| --- | --- | --- |
| `opencode` | 终端 | 启动 TUI |
| `opencode run "<PROMPT>"` | 终端 | 执行一次非交互任务 |
| `opencode models` | 终端 | 查看可用模型 |
| `opencode auth list` | 终端 | 查看已认证的提供商 |
| `/connect` | OpenCode TUI  | 在 TUI 中连接提供商 |
| `/models` | OpenCode TUI  | 选择模型 |
| `/new`、`/sessions` | OpenCode TUI  | 新建或切换会话 |
| `/undo`、`/redo` | OpenCode TUI  | 撤销或重做上一次操作 |
| `/help` | OpenCode TUI  | 查看 TUI 帮助 |


完整参数见 [CLI 文档](https://opencode.ai/docs/zh-cn/cli/) 和 [TUI 文档](https://opencode.ai/docs/zh-cn/tui/)。

### 3.2 基础工作流

1. 进入目标项目目录并运行 `opencode`。
2. 使用 `/connect` 配置提供商，或按第 2.3 节连接本地模型服务。
3. 使用 `/models` 选择模型。
4. 先让模型只读分析项目，再确认需要执行的修改。
5. 检查模型执行的命令和文件变更后，再运行测试或提交代码。

在消息中输入 `@` 可以引用项目文件；以 `!` 开头可以执行 Shell 命令。详细用法见 [TUI 文档](https://opencode.ai/docs/zh-cn/tui/)。

## 参考资料

- [OpenCode 官方中文文档](https://opencode.ai/docs/zh-cn/)
- [OpenCode 配置文档](https://opencode.ai/docs/zh-cn/config/)
- [OpenCode 提供商文档](https://opencode.ai/docs/zh-cn/providers/)
- [OpenCode 模型文档](https://opencode.ai/docs/zh-cn/models/)
- [OpenCode CLI 文档](https://opencode.ai/docs/zh-cn/cli/)
- [OpenCode TUI 文档](https://opencode.ai/docs/zh-cn/tui/)

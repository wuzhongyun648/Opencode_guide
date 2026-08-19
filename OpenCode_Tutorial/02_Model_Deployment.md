# OpenCode 模型配置与部署

> 本文介绍 OpenCode 支持的常见 API 和订阅、模型提供商配置，以及本地推理服务部署。安装 OpenCode 前请先阅读 [01_Installation.md](./01_Installation.md)。OpenCode Server 和 Docker 部署见 [03_OpenCode_Server_Deployment.md](./03_OpenCode_Server_Deployment.md)。

## 1. 常见 API 与订阅支持情况

OpenCode 使用 AI SDK 和 Models.dev，支持 75 个以上的模型提供商，也支持 OpenAI-compatible API 和本地模型。模型服务的可用地区、额度、价格和具体模型由提供商决定。

### 1.1 常见 API

| 模型或平台 | 提供商 ID | 状态 | 认证信息 | 获取凭据 | 说明 |
| --- | --- | --- | --- | --- | --- |
| OpenCode Zen | `opencode` | 支持 | Zen API Key | [OpenCode Zen](https://opencode.ai/auth) | OpenCode 官方维护的模型网关，适合首次使用 |
| OpenAI GPT | `openai` | 支持 | `OPENAI_API_KEY` | [OpenAI Platform](https://platform.openai.com/api-keys) | API 账单与 ChatGPT 订阅分开 |
| Anthropic Claude | `anthropic` | 支持 | `ANTHROPIC_API_KEY` | [Anthropic Console](https://console.anthropic.com/settings/keys) | 也可通过 Bedrock、Vertex AI 或 Zen 使用 |
| Google Gemini API | `google` | 支持 | `GEMINI_API_KEY`、`GOOGLE_API_KEY` | [Google AI Studio](https://aistudio.google.com/apikey) | Gemini Developer API |
| Google Vertex AI | `google-vertex` | 支持 | Google Cloud 项目和应用默认凭据 | [Google Cloud Console](https://console.cloud.google.com/vertex-ai) | 适合 Google Cloud 和企业环境 |
| DeepSeek | `deepseek` | 支持 | `DEEPSEEK_API_KEY` | [DeepSeek 开放平台](https://platform.deepseek.com/api_keys) | 使用 DeepSeek API 独立计费 |
| 通义千问 Qwen（中国站） | `alibaba-cn` | 支持 | `DASHSCOPE_API_KEY` | [阿里云百炼](https://bailian.console.aliyun.com/?tab=model#/api-key) | DashScope 中国站 |
| 通义千问 Qwen（国际站） | `alibaba` | 支持 | `DASHSCOPE_API_KEY` | [Alibaba Cloud Model Studio](https://modelstudio.console.alibabacloud.com/) | DashScope 国际站 |
| xAI Grok | `xai` | 支持 | `XAI_API_KEY` | [xAI Console](https://console.x.ai/) | X/Grok 网页订阅不等于 xAI API 额度 |
| Moonshot Kimi | `moonshotai` | 支持 | `MOONSHOT_API_KEY` | [Moonshot 开放平台](https://platform.moonshot.ai/console/api-keys) | 使用 Moonshot API |
| MiniMax | `minimax` | 支持 | `MINIMAX_API_KEY` | [MiniMax 开放平台](https://platform.minimax.io/) | 使用 MiniMax API |
| Z.AI GLM | `zai` | 支持 | `ZHIPU_API_KEY` | [Z.AI API 控制台](https://z.ai/manage-apikey/apikey-list) | Coding Plan 需要选择专用入口 |
| OpenRouter | `openrouter` | 支持 | `OPENROUTER_API_KEY` | [OpenRouter Keys](https://openrouter.ai/settings/keys) | 通过统一网关访问多个厂商模型 |
| Azure OpenAI | `azure` | 支持 | Azure API Key 和资源名称 | [Azure Portal](https://portal.azure.com/) | 使用 Azure 中已部署的模型 |
| Amazon Bedrock | `amazon-bedrock` | 支持 | AWS 凭据或 Bedrock Token | [Amazon Bedrock Console](https://console.aws.amazon.com/bedrock/) | 需要先在目标区域申请模型访问权限 |
| Ollama、LM Studio、llama.cpp | 自定义 | 支持 | 通常不需要远程 API Key | 对应本地工具 | 通过本地 OpenAI-compatible 端点接入 |

没有出现在表格中的提供商不一定不受支持。可以先在 `/connect` 中搜索，再到 [Models.dev](https://models.dev/) 查询提供商 ID；只要服务提供兼容的 API，通常也可以作为自定义提供商接入。

### 1.2 常见订阅

| 订阅 | 是否可用于 OpenCode | 接入方式 | 重要限制 |
| --- | --- | --- | --- |
| ChatGPT Plus/Pro | 支持 | OpenAI 浏览器授权 | 使用订阅权益，不会转换成 OpenAI API 余额 |
| Claude Pro/Max | 可以使用 | Anthropic 浏览器授权 | OpenCode 明确说明该用法未获 Anthropic 官方支持 |
| GitHub Copilot | 支持 | GitHub Device Login | 部分模型需要 Copilot Pro+ |
| Z.AI GLM Coding Plan | 支持 | Coding Plan API Key | 必须在 `/connect` 中选择 `Z.AI Coding Plan` |
| Qwen Coding Plan/Token Plan | 支持对应计划入口 | 计划专用 API Key | 与普通千问聊天会员不同，入口和地区必须匹配 |
| Google AI Pro/Ultra | 当前不建议使用 | 曾依赖社区 OAuth 插件 | 消费者 Gemini CLI OAuth 已停止，改用 Gemini API Key |
| Grok/X Premium | 不支持订阅直连 | 改用 xAI API Key | X 订阅不等于 xAI API 额度 |
| DeepSeek 网页服务 | 不支持订阅直连 | 改用 DeepSeek API Key | 网页产品与开放平台 API 分开 |

需要注意：

- **Google AI Pro/Ultra**：OpenCode 生态页面曾收录 `opencode-gemini-auth` 社区插件，但 Google 已于 2026-06-18 停止个人免费层、Google AI Pro 和 Google AI Ultra 的 Gemini CLI OAuth 访问。当前应使用 Gemini API Key、Vertex AI 或 OpenCode Zen。
- **Grok/X Premium**：OpenCode 官方只记录 xAI API Key 接入，没有消费者订阅登录方式。
- **DeepSeek 网页服务**：OpenCode 官方只记录 DeepSeek 开放平台 API Key 接入。


## 2. 接入模型 API

API 接入分为五类：直接连接模型厂商、连接 Qwen/DashScope、通过云平台或模型网关连接、配置自定义兼容接口，以及部署本地推理服务。除不要求认证的本地服务外，需要先取得对应服务的 API Key 或云平台凭据。

### 2.1 直接连接模型厂商

OpenAI、Anthropic、Google、DeepSeek、xAI、Moonshot、MiniMax、Z.AI 和 OpenRouter 等常见提供商可以直接接入。个人开发环境优先使用 `/connect` 保存 API Key；Server、容器和 CI/CD 优先使用环境变量。

#### 2.1.1 使用 `/connect` 保存 API Key

这是个人开发环境最简单的配置方式：

1. 启动 OpenCode。
2. 在 TUI 中执行：

```text
/connect
```

3. 搜索并选择提供商，例如 `DeepSeek`、`OpenAI`、`Anthropic`、`Google`、`xAI` 或 `OpenRouter`。
4. 选择 API Key 认证方式并粘贴密钥。
5. 完成认证后进入第 4 节选择和验证模型。

也可以从普通终端启动认证流程：

```bash
opencode auth login
```

查看已保存的认证信息：

```bash
opencode auth list
```

`/connect` 保存的凭据默认位于 `~/.local/share/opencode/auth.json`。该文件包含敏感信息，不得提交到 Git、写入容器镜像或在团队间直接共享。

#### 2.1.2 使用环境变量提供 API Key

环境变量适合 Server、容器、CI/CD，以及不希望将凭据写入 OpenCode 认证文件的场景。以下仅为常见示例，执行时替换占位值：

```bash
export OPENAI_API_KEY='your-openai-api-key'
export ANTHROPIC_API_KEY='your-anthropic-api-key'
export GEMINI_API_KEY='your-gemini-api-key'
export DEEPSEEK_API_KEY='your-deepseek-api-key'
export DASHSCOPE_API_KEY='your-dashscope-api-key'
export XAI_API_KEY='your-xai-api-key'
export MOONSHOT_API_KEY='your-moonshot-api-key'
export MINIMAX_API_KEY='your-minimax-api-key'
export ZHIPU_API_KEY='your-zai-api-key'
export OPENROUTER_API_KEY='your-openrouter-api-key'
opencode
```

Windows PowerShell 使用 `$env:` 设置当前终端会话的环境变量：

```powershell
$env:OPENAI_API_KEY = "your-openai-api-key"
$env:ANTHROPIC_API_KEY = "your-anthropic-api-key"
$env:GEMINI_API_KEY = "your-gemini-api-key"
$env:DEEPSEEK_API_KEY = "your-deepseek-api-key"
$env:DASHSCOPE_API_KEY = "your-dashscope-api-key"
$env:XAI_API_KEY = "your-xai-api-key"
$env:MOONSHOT_API_KEY = "your-moonshot-api-key"
$env:MINIMAX_API_KEY = "your-minimax-api-key"
$env:ZHIPU_API_KEY = "your-zai-api-key"
$env:OPENROUTER_API_KEY = "your-openrouter-api-key"
opencode
```

不需要同时配置所有变量，只设置准备使用的提供商。不要把真实密钥写入项目中的脚本、`.env` 示例或 `opencode.json`。

### 2.2 连接 Qwen/DashScope API

Qwen 没有使用单独的 `qwen` 提供商 ID。中国站使用 `alibaba-cn`，国际站使用 `alibaba`，两者都读取 `DASHSCOPE_API_KEY`，但 API Key 和服务地区需要匹配。

| 地区 | 提供商 ID | OpenAI-compatible Base URL |
| --- | --- | --- |
| 中国站 | `alibaba-cn` | `https://dashscope.aliyuncs.com/compatible-mode/v1` |
| 国际站 | `alibaba` | `https://dashscope-intl.aliyuncs.com/compatible-mode/v1` |

先从第 1.1 节对应控制台创建 API Key，再选择一种方式接入：

```bash
export DASHSCOPE_API_KEY='your-dashscope-api-key'
opencode
```

Windows PowerShell：

```powershell
$env:DASHSCOPE_API_KEY = "your-dashscope-api-key"
opencode
```

也可以执行 `/connect` 并搜索 Alibaba 提供商。配置完成后，进入第 4 节检查对应地区的模型。

### 2.3 通过云平台或模型网关接入

#### 2.3.1 OpenCode Zen

OpenCode Zen 是 OpenCode 官方维护的模型网关，可通过一组凭据使用经过 OpenCode 测试的不同厂商模型：

1. 登录 [OpenCode Zen](https://opencode.ai/auth)，添加账单信息并创建 API Key。
2. 在 TUI 中执行 `/connect`。
3. 选择 `OpenCode Zen` 并粘贴 API Key。
4. 完成认证后进入第 4 节选择和验证模型。

Zen 是可选的按量付费服务，不是使用 OpenCode 的必要条件。也可以继续使用模型厂商自己的 API。

#### 2.3.2 Google Vertex AI

先在 Google Cloud 项目中启用 Vertex AI API，再配置项目、区域和应用默认凭据：

```bash
gcloud auth application-default login
export GOOGLE_CLOUD_PROJECT='your-project-id'
export VERTEX_LOCATION='global'
opencode
```

服务账号环境可以改用：

```bash
export GOOGLE_APPLICATION_CREDENTIALS='/path/to/service-account.json'
export GOOGLE_CLOUD_PROJECT='your-project-id'
opencode
```


#### 2.3.3 Azure OpenAI

1. 在 Azure 中创建 Azure OpenAI 资源。
2. 在 Azure AI Foundry 部署模型，并确保部署名称与模型名称一致。
3. 执行 `/connect`，搜索并选择 `Azure`，然后输入 API Key。
4. 设置 Azure 资源名称后启动 OpenCode：

```bash
export AZURE_RESOURCE_NAME='your-resource-name'
opencode
```

#### 2.3.4 Amazon Bedrock

先在 Bedrock 模型目录中申请目标模型的访问权限，然后选择一种 AWS 认证方式：

```bash
AWS_PROFILE='your-profile' AWS_REGION='us-east-1' opencode
```

也可以使用 `AWS_ACCESS_KEY_ID` 和 `AWS_SECRET_ACCESS_KEY`、Bedrock Bearer Token、IAM Role 或 Web Identity。生产环境优先使用短期凭据或 IAM Role。

### 2.4 接入自定义 OpenAI-compatible API

如果 `/connect` 中没有目标提供商，但它提供 `/v1/chat/completions` 兼容接口，可以按以下方式配置。先执行 `/connect`，选择 `Other`，将提供商 ID 设置为 `myprovider` 并保存 API Key；再在项目根目录创建或修改 `opencode.json`：

```json
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "myprovider": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "My Provider",
      "options": {
        "baseURL": "https://api.example.com/v1"
      },
      "models": {
        "my-model-id": {
          "name": "My Model"
        }
      }
    }
  }
}
```

提供商 ID 必须与 `/connect` 中输入的 ID 一致，模型 ID 必须与服务端实际模型 ID 一致。如果服务使用 OpenAI Responses API 而不是 Chat Completions API，应按照服务文档改用 `@ai-sdk/openai`。

### 2.5 部署并连接本地推理服务

OpenCode 可以连接 Ollama、LM Studio、llama.cpp 等本地推理服务。与云端 API 不同，本地方式需要先安装推理工具、下载模型并启动 API 服务。下面以 Ollama 为例。

#### 2.5.1 安装 Ollama

从 [Ollama 下载页](https://ollama.com/download) 安装对应平台版本。Linux 也可以使用官方安装脚本：

```bash
curl -fsSL https://ollama.com/install.sh | sh
```

安装后验证：

```bash
ollama --version
```

#### 2.5.2 下载并启动模型

以下示例下载 Qwen Coder 模型。该模型需要较多内存或显存；资源不足时，应在 [Ollama 模型库](https://ollama.com/search) 中选择更小且支持工具调用的模型。

```bash
ollama pull qwen3-coder:30b
ollama list
```

OpenCode 编码代理需要至少 64K 上下文。Ollama Desktop 可以在设置中将 Context length 调整到 `64000` 或更高；使用 CLI 启动服务时，可以设置 `OLLAMA_CONTEXT_LENGTH`：

```bash
OLLAMA_CONTEXT_LENGTH=64000 ollama serve
```

Windows PowerShell：

```powershell
$env:OLLAMA_CONTEXT_LENGTH = "64000"
ollama serve
```

Ollama Desktop 或系统服务已经运行时，需要在对应应用或服务配置中调整上下文后重启，不能再启动第二个 `ollama serve`。

默认 API 地址为 `http://127.0.0.1:11434`。在 macOS、Linux 或 WSL 中检查 OpenAI-compatible 模型列表：

```bash
curl http://127.0.0.1:11434/v1/models
```

Windows PowerShell 可以使用：

```powershell
Invoke-RestMethod http://127.0.0.1:11434/v1/models
```

模型运行后，检查实际分配的上下文和 CPU/GPU 使用情况：

```bash
ollama ps
```

确认 `CONTEXT` 至少为 `64000`。更大的上下文会增加内存或显存占用，应尽量避免模型大量卸载到 CPU。

#### 2.5.3 配置 OpenCode

在项目根目录的 `opencode.json` 中添加 Ollama 提供商。`models` 中的键必须与 `ollama list` 显示的模型 ID 一致：

```json
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "ollama": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "Ollama (local)",
      "options": {
        "baseURL": "http://127.0.0.1:11434/v1"
      },
      "models": {
        "qwen3-coder:30b": {
          "name": "Qwen3 Coder 30B (local)"
        }
      }
    }
  }
}
```

重新启动 OpenCode 后，在 `/models` 中选择 `ollama/qwen3-coder:30b`。编码代理依赖稳定的工具调用和较大的上下文窗口；小型或不支持 tool calling 的本地模型可能只能完成简单问答。完成普通问答后，还应发送一次只读工具调用请求，例如“列出当前项目根目录的文件，不要修改任何内容”，确认模型能够正确调用工具。

如果 OpenCode 和 Ollama 位于不同容器或主机，`127.0.0.1` 指向各自所在环境，不能直接互通。此时需要使用 Ollama 实际可访问的网络地址，并限制端口只对可信网络开放。

## 3. 接入支持的订阅

### 3.1 ChatGPT Plus/Pro

OpenCode 官方支持通过浏览器使用 ChatGPT Plus/Pro 订阅：

1. 在 TUI 中执行 `/connect`。
2. 选择 `OpenAI`。
3. 选择 `ChatGPT Plus/Pro`。
4. 在浏览器中登录购买了订阅的 OpenAI 账号并授权。
5. 返回 OpenCode，进入第 4 节选择和验证模型。

该方式使用 ChatGPT 订阅权益，不会生成 `OPENAI_API_KEY`，也不会向 OpenAI API 账户充值。可用模型、限额和功能由订阅方案决定。

### 3.2 Claude Pro/Max

OpenCode 提供 Claude Pro/Max 登录入口：

1. 在 TUI 中执行 `/connect`。
2. 选择 `Anthropic`。
3. 选择 `Claude Pro/Max`。
4. 在浏览器中登录 Claude 账号并授权。
5. 返回 OpenCode，进入第 4 节选择和验证模型。

> OpenCode 官方文档明确说明：在 OpenCode 中使用 Claude Pro/Max 订阅不是 Anthropic 官方支持的用法。认证方式、可用性和限额可能变化；稳定或生产环境优先使用 Anthropic API、Amazon Bedrock 或 Google Vertex AI。

### 3.3 GitHub Copilot

1. 在 TUI 中执行 `/connect`。
2. 搜索并选择 `GitHub Copilot`。
3. 打开终端显示的 GitHub Device Login 地址并输入验证码。
4. 完成授权后返回 OpenCode。
5. 进入第 4 节选择和验证模型。

部分模型可能需要 GitHub Copilot Pro+，组织账号还可能受到管理员策略限制。

### 3.4 Z.AI GLM Coding Plan

1. 在 Z.AI 控制台创建 Coding Plan 对应的 API Key。
2. 在 TUI 中执行 `/connect`。
3. 搜索 Z.AI，并选择 `Z.AI Coding Plan`，不要选择普通按量 API 入口。
4. 输入 Coding Plan API Key。
5. 进入第 4 节选择和验证当前计划支持的 GLM 模型。

### 3.5 Qwen Coding Plan/Token Plan

OpenCode 的提供商注册表包含 `alibaba-coding-plan`、`alibaba-coding-plan-cn`、`alibaba-token-plan` 和 `alibaba-token-plan-cn`。使用时需要：

1. 在阿里云百炼购买相应计划并创建计划专用 API Key。
2. 在 `/connect` 中选择与计划和地区匹配的 Alibaba 入口；如果当前版本未显示该入口，可以分别设置 `ALIBABA_CODING_PLAN_API_KEY` 或 `ALIBABA_TOKEN_PLAN_API_KEY`。
3. 进入第 4 节检查计划实际开放的模型并验证调用。

普通千问网页或 App 会员不能替代 Coding Plan、Token Plan 或 DashScope API Key。

## 4. 选择并验证模型

API 或订阅认证完成后，需要确认可用模型、选择当前模型，并验证一次真实调用。

### 4.1 查看并选择模型

在 TUI 中打开模型选择器：

```text
/models
```

也可以在普通终端查看全部可用模型：

```bash
opencode models
```

只查看某个已经配置并可用的提供商：

```bash
opencode models deepseek
```

模型名称使用 `<provider-id>/<model-id>` 格式，例如 `deepseek/deepseek-chat`。如果提供商尚未认证或环境变量尚未生效，按提供商过滤时可能返回 `Provider not found`。

### 4.2 设置默认模型

先通过 `/models` 或 `opencode models` 确认准确 ID，再写入项目级或全局 `opencode.json`：

```json
{
  "$schema": "https://opencode.ai/config.json",
  "model": "deepseek/deepseek-chat"
}
```

项目配置通常放在项目根目录的 `opencode.json`；全局配置位于 `~/.config/opencode/opencode.json`。OpenCode 会合并不同作用域的配置，项目配置只覆盖与全局配置冲突的字段。修改配置后需要退出并重新启动 OpenCode。

也可以只为当前一次启动指定模型：

```bash
opencode --model deepseek/deepseek-chat
```

### 4.3 验证模型调用

进入项目目录并启动 OpenCode。下面的 `/path/to/project` 需要替换为实际项目目录：

```bash
cd /path/to/project
opencode
```

先发送一个只读问题：

```text
请说明这个项目的目录结构，不要修改任何文件。
```

能够正常收到回答，表示提供商认证、模型访问和基础网络均可用。



## 5. 在本机运行 OpenCode

直接在项目目录运行：

```bash
opencode
```

该模式下，TUI 和 OpenCode Server 在当前主机启动，OpenCode 直接访问本机项目。模型推理是否在本地取决于所配置的模型提供商。

该模式适合个人日常开发，也是默认推荐方式。需要将 OpenCode Server 放到远程主机或容器时，参见 [03_OpenCode_Server_Deployment.md](./03_OpenCode_Server_Deployment.md)。

## 6. 常见问题

### 6.1 模型不可用

```bash
opencode auth list
opencode models
```

依次确认：

- 提供商已经完成认证。
- API Key 或 OAuth Token 仍然有效。
- 模型名称使用 `<provider-id>/<model-id>` 格式。
- 当前账号具备目标模型的访问权限和可用额度。
- 当前网络允许访问提供商 API。

### 6.2 无法连接本地推理服务

依次检查：

- Ollama、LM Studio 或 llama.cpp 服务是否已经启动。
- `baseURL` 是否与推理服务实际监听地址一致。
- 配置中的模型 ID 是否与服务返回的模型 ID 完全一致。
- OpenCode 和推理服务是否位于不同容器、WSL 环境或主机。
- 模型是否支持工具调用，且上下文窗口是否足够。

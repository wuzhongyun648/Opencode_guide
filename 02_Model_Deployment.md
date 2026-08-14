# OpenCode 模型配置与部署

> 本文介绍模型提供商配置、OpenCode 运行位置以及本地、远程和 Docker 部署方式。安装 OpenCode 前请先阅读 [01_Installation.md](./01_Installation.md)。
>
> 文档依据 [OpenCode 官方文档](https://opencode.ai/docs/zh-cn/) 整理。最后核对日期：2026-08-14。

## 1. 理解运行位置

OpenCode 的安装位置和模型的运行位置是两个不同概念：

```text
开发者使用入口（TUI / Desktop / IDE）
                |
                v
OpenCode Server（读取源码、执行工具、管理会话）
                |
                v
模型提供商（执行模型推理）
```

- 本地一体化模式下，使用入口和 OpenCode Server 位于同一台机器。
- 分离部署时，使用入口和 OpenCode Server 位于不同进程或主机。
- 模型通常由远程提供商托管，也可以接入兼容的私有模型服务。
- 模型请求由 OpenCode Server 发起，因此模型凭据应配置在 Server 所在环境。
- “本地运行 OpenCode”不表示模型推理完全本地，也不表示项目上下文不会发送给模型提供商。

## 2. 配置模型提供商

### 2.1 通过 TUI 配置

启动 OpenCode 后执行：

```text
/connect
```

选择提供商，并根据提示完成 API Key 或 OAuth 认证。

### 2.2 通过 CLI 配置

```bash
opencode auth login
```

查看已认证的提供商：

```bash
opencode auth list
```

查看当前可用模型：

```bash
opencode models
```

模型名称使用 `<provider-id>/<model-id>` 格式，例如 `openai/gpt-4.1`。实际可用模型取决于提供商账号、区域、权限和订阅。

认证数据默认存储在 `~/.local/share/opencode/auth.json`。该文件包含敏感信息，不得提交到 Git、写入容器镜像或在团队间直接共享。

### 2.3 验证模型

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

## 3. 本地一体化运行

直接在项目目录运行：

```bash
opencode
```

该模式下，TUI 和 OpenCode Server 在当前主机启动，OpenCode 直接访问本机项目。模型推理是否在本地取决于所配置的模型提供商。

该模式适合个人日常开发，也是默认推荐方式。

## 4. OpenCode Server 分离部署

`opencode serve` 启动无界面的 HTTP Server，适合以下场景：

- 项目源码位于远程开发机。
- 本地 TUI 或 Desktop 需要连接远程执行环境。
- 脚本、SDK 或其他系统需要调用 OpenCode API。
- 希望复用长期运行的 Server，减少重复初始化。

### 4.1 架构要点

- Server 负责读取项目源码、执行命令、管理会话和调用模型。
- TUI、Desktop、Web、自定义程序或 SDK 可以作为访问入口。
- 项目源码必须对 Server 可见；仅存在于客户端本机的文件不能被远程 Server 直接处理。
- 模型凭据应配置在 Server 所在环境。

### 4.2 启动本地 Server

下面的 `/srv/projects/example` 是占位路径，需要替换为 Server 上的实际项目目录：

```bash
cd /srv/projects/example
opencode serve --hostname 127.0.0.1 --port 4096
```

默认监听 `127.0.0.1`，只有 Server 本机可以访问。

### 4.3 启动可远程访问的 Server

跨主机访问必须使用 VPN、SSH 隧道或 HTTPS，不能仅依赖 HTTP Basic Auth。以下命令适用于需要监听网络接口的 Bash 环境，通过交互输入密码，避免将密码值直接写入命令本身：

```bash
read -rsp "OpenCode Server password: " OPENCODE_SERVER_PASSWORD
export OPENCODE_SERVER_PASSWORD
export OPENCODE_SERVER_USERNAME='opencode'
opencode serve --hostname 0.0.0.0 --port 4096
```

主要参数：

| 参数 | 默认值 | 说明 |
| --- | --- | --- |
| `--hostname` | `127.0.0.1` | Server 监听地址 |
| `--port` | `4096` | Server 监听端口 |
| `--mdns` | `false` | 是否启用局域网 mDNS 发现 |
| `--mdns-domain` | `opencode.local` | mDNS 服务域名 |
| `--cors` | 空 | 额外允许的浏览器来源，可重复指定 |
| `OPENCODE_SERVER_USERNAME` | `opencode` | HTTP Basic Auth 用户名 |
| `OPENCODE_SERVER_PASSWORD` | 未设置 | 设置后启用 HTTP Basic Auth |

### 4.4 TUI 连接 Server

客户端同样需要安装 OpenCode CLI。

使用 SSH 隧道时，Server 按第 4.2 节监听 `127.0.0.1:4096`。在客户端终端建立隧道，其中 `developer@server.example.com` 需要替换为实际 SSH 目标：

```bash
ssh -N -L 4096:127.0.0.1:4096 developer@server.example.com
```

保持隧道运行，在另一个客户端终端连接本地转发端口：

```bash
opencode attach http://127.0.0.1:4096
```

生产环境应先配置 DNS、TLS 证书和 HTTPS 反向代理，再使用 HTTPS 地址：

```bash
opencode attach https://opencode.example.com
```

HTTP Basic Auth 不加密用户名、密码或项目内容。跨主机访问必须通过 VPN、SSH 隧道或 HTTPS 建立受保护的传输通道。

### 4.5 健康检查与接口文档

在 Server 本机检查：

```bash
curl --user "$OPENCODE_SERVER_USERNAME" \
  http://127.0.0.1:4096/global/health
```

`curl` 会交互提示输入密码，密码不会作为命令参数出现在进程列表中。

OpenAPI 文档地址为 `http://127.0.0.1:4096/doc`。配置 HTTPS 反向代理后，再使用对应的 HTTPS 域名访问。

### 4.6 安全要求

- 不要将未认证的 `opencode serve` 或 `opencode web` 暴露到公网。
- 通过防火墙或安全组限制可信来源。
- 使用专用低权限操作系统账户运行 Server。
- 只授予必要的项目目录、命令和网络权限。
- 使用服务管理器、Secret 管理系统或受保护的环境文件注入密码和 API Key。
- OpenCode 具备文件读写和命令执行能力，不能按普通只读 Web 应用对待。

## 5. Docker 部署 Server

容器化适合流水线、短期隔离环境或已有容器运维体系的团队，不建议作为第一次使用 OpenCode 的默认方案。

以下示例适用于 macOS、Linux 和 WSL 的 Bash。运行前先进入需要处理的项目目录。

### 5.1 准备密码和持久化卷

```bash
read -rsp "OpenCode Server password: " OPENCODE_SERVER_PASSWORD
export OPENCODE_SERVER_PASSWORD
docker volume create opencode-data
docker volume create opencode-config
```

### 5.2 启动容器

```bash
docker run --rm -it \
  -p 127.0.0.1:4096:4096 \
  -v "$PWD:/workspace" \
  -v opencode-data:/root/.local/share/opencode \
  -v opencode-config:/root/.config/opencode \
  -w /workspace \
  -e OPENCODE_SERVER_USERNAME=opencode \
  -e OPENCODE_SERVER_PASSWORD \
  ghcr.io/anomalyco/opencode \
  serve --hostname 0.0.0.0 --port 4096
```

该示例使用 Docker named volume，避免容器以 root 身份在宿主机的 OpenCode 配置目录中创建文件。端口只绑定到宿主机回环地址；跨主机访问应额外配置 SSH 隧道或 HTTPS 反向代理。

生产环境还应满足：

- 固定镜像版本或摘要，不依赖浮动标签。
- 通过 Secret 注入模型凭据和服务密码。
- 限制容器用户、挂载目录、网络和可用资源。
- 评估容器写入宿主项目后产生的文件属主和权限。
- 在入口代理终止 TLS，不直接暴露明文 HTTP。

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

### 6.2 无法连接远程 Server

依次检查：

- Server 是否监听预期地址和端口。
- 防火墙、安全组、DNS 和反向代理是否配置正确。
- 客户端使用的 HTTP/HTTPS 协议是否与实际入口一致。
- 用户名和密码是否与 Server 环境变量一致。
- `/global/health` 是否可以从预期网络访问。

### 6.3 Docker 退出后配置丢失

确认已经挂载 OpenCode 的数据目录和配置目录。只挂载项目目录不会保留容器内的认证信息、配置和会话。

## 7. 参考资料

- [OpenCode 提供商文档](https://opencode.ai/docs/zh-cn/providers/)
- [OpenCode Server 文档](https://opencode.ai/docs/zh-cn/server/)
- [OpenCode Web 文档](https://opencode.ai/docs/zh-cn/web/)
- [OpenCode CLI 文档](https://opencode.ai/docs/zh-cn/cli/)
- [OpenCode 网络文档](https://opencode.ai/docs/zh-cn/network/)
- [OpenCode 故障排除](https://opencode.ai/docs/zh-cn/troubleshooting/)

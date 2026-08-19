# OpenCode Server 部署

> 普通个人开发不需要部署 OpenCode Server。完成第 1、2 章后，直接在项目目录运行 `opencode` 即可。本章仅面向远程开发、Web、SDK 和容器部署场景。
>
> 本文介绍 OpenCode Server 的本地监听、远程连接、Docker 部署和安全要求。安装 OpenCode 见 [01_Installation.md](./01_Installation.md)，模型凭据和默认模型配置见 [02_Model_Deployment.md](./02_Model_Deployment.md)。

## 1. 理解 Server 部署与部署前检查

### 1.1 运行架构与适用场景

直接运行 `opencode` 时，TUI 和 OpenCode Server 位于同一台机器。`opencode serve` 可以单独启动无界面的 HTTP Server，供 TUI、Web、Desktop、SDK 或自定义程序连接。

```text
客户端（TUI / Web / Desktop / SDK）
                 |
                 v
OpenCode Server（读取源码、执行工具、管理会话）
                 |
                 v
模型提供商或本地推理服务
```

Server 部署适合以下场景：

- 项目源码位于远程开发机。
- 本地客户端需要连接远程执行环境。
- 脚本、SDK 或其他系统需要调用 OpenCode API。
- 团队已有长期运行的容器或服务运维体系。

需要注意：

- 项目源码必须对 Server 可见，远程 Server 不能直接读取只存在于客户端本机的文件。
- 文件读写和命令都在 Server 所在环境执行。
- 模型凭据和 `opencode.json` 应配置在 Server 所在环境。
- OpenCode 具备命令执行能力，不能按普通只读 Web 应用部署。

### 1.2 部署前检查

开始部署前，先在 Server 所在主机完成以下检查：

1. 确认 OpenCode 已安装，并记录实际可执行文件路径：

```bash
opencode --version
command -v opencode
```

Windows PowerShell 使用：

```powershell
opencode --version
Get-Command opencode
```

2. 确认运行账户可以读取和修改目标项目，并可以执行项目需要的开发命令。
3. 按 [02_Model_Deployment.md](./02_Model_Deployment.md) 在 Server 环境中配置模型凭据和默认模型。
4. 进入目标项目运行一次 `opencode`，确认模型能够回答只读问题。
5. 确认准备使用的端口没有被其他程序占用，并通过防火墙限制访问来源。
6. 根据使用场景选择运行方式：

| 场景 | 推荐方式 |
| --- | --- |
| 普通个人开发 | 直接运行 `opencode`，不单独部署 Server |
| SDK、自定义客户端或 SSH 隧道 | `opencode serve` |
| 浏览器访问 | `opencode web` |
| 已有容器运维体系 | Docker 部署 |

选择 Docker 时，还需要确认 Docker daemon 可访问，并验证 OpenCode 镜像可以启动：

```bash
docker version
docker run --rm ghcr.io/anomalyco/opencode --version
```

这两条命令同样适用于 Windows PowerShell。若当前账户无权访问 Docker daemon，应先按 Docker 官方文档修复权限，不要通过长期使用管理员账户规避权限问题。

只有以上检查全部通过后，才继续配置远程访问或长期运行。

## 2. 启动仅本机可访问的 Server

下面的 `/srv/projects/example` 是占位路径，需要替换为 Server 上的实际项目目录：

```bash
cd /srv/projects/example
opencode serve --hostname 127.0.0.1 --port 4096
```

默认监听 `127.0.0.1`，只有 Server 本机可以访问。该方式适合本机客户端、反向代理或 SSH 隧道，不会直接向局域网或公网开放端口。

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

`--cors` 只用于允许自定义浏览器前端跨域访问 Server。普通 TUI、`opencode attach` 和 SSH 隧道不需要配置 CORS。应填写完整且明确的来源，可以重复指定：

```bash
opencode serve \
  --cors https://app.example.com \
  --cors http://localhost:5173
```

`--mdns` 用于可信局域网中的自动发现，并会使服务监听网络接口。它不是公网发现或访问控制机制，不应在不可信网络中启用：

```bash
opencode serve --mdns --mdns-domain opencode.local
```

CORS 和 mDNS 都不能替代密码、TLS、VPN 或防火墙。

## 3. 启动 Web 界面

`opencode web` 会启动带浏览器界面的 Server。默认监听 `127.0.0.1`，自动选择可用端口并打开默认浏览器：

```bash
opencode web
```

需要固定端口时：

```bash
opencode web --port 4096
```

Windows 用户优先从 WSL 启动 Web，以获得更好的文件系统和终端兼容性。原生 PowerShell 也可以直接执行相同的 `opencode web` 命令。

通过 SSH 隧道访问，或者 HTTPS 反向代理与 OpenCode 位于同一台主机时，Web 应继续监听回环地址：

```bash
opencode web --hostname 127.0.0.1 --port 4096
```

只有通过 VPN 直接访问或反向代理位于另一台主机时，才需要监听特定私有地址或 `0.0.0.0`，并通过防火墙限制来源。认证和传输保护要求见第 4 节。

## 4. 配置远程访问

跨主机访问必须使用 VPN、SSH 隧道或 HTTPS，不能仅依赖 HTTP Basic Auth。

### 4.1 使用 SSH 隧道

Server 继续按第 2 节监听 `127.0.0.1:4096`。在客户端建立 SSH 隧道，其中 `developer@server.example.com` 需要替换为实际 SSH 目标：

```bash
ssh -N -L 4096:127.0.0.1:4096 developer@server.example.com
```

保持隧道运行，在另一个客户端终端连接本地转发端口：

```bash
read -rsp "OpenCode Server password: " OPENCODE_SERVER_PASSWORD
export OPENCODE_SERVER_PASSWORD
opencode attach http://127.0.0.1:4096
```

如果 Server 没有设置密码，可以省略前两行。Windows PowerShell 客户端使用：

```powershell
$securePassword = Read-Host "OpenCode Server password" -AsSecureString
$env:OPENCODE_SERVER_PASSWORD = [System.Net.NetworkCredential]::new("", $securePassword).Password
opencode attach http://127.0.0.1:4096
```

Windows 10/11 自带 OpenSSH 客户端时，可以在 PowerShell 中直接执行相同的 `ssh -N -L ...` 隧道命令。

### 4.2 为 HTTPS 反向代理准备 Server

反向代理与 OpenCode 位于同一台主机时，Server 应继续监听 `127.0.0.1`，不需要向整个网络开放后端端口。以下命令适用于 Bash，通过交互输入密码，避免将密码直接写入命令参数：

```bash
read -rsp "OpenCode Server password: " OPENCODE_SERVER_PASSWORD
export OPENCODE_SERVER_PASSWORD
export OPENCODE_SERVER_USERNAME='opencode'
opencode serve --hostname 127.0.0.1 --port 4096
```

Windows PowerShell：

```powershell
$securePassword = Read-Host "OpenCode Server password" -AsSecureString
$env:OPENCODE_SERVER_PASSWORD = [System.Net.NetworkCredential]::new("", $securePassword).Password
$env:OPENCODE_SERVER_USERNAME = "opencode"
opencode serve --hostname 127.0.0.1 --port 4096
```

完成 HTTPS 反向代理配置后，客户端使用 HTTPS 地址：

```bash
read -rsp "OpenCode Server password: " OPENCODE_SERVER_PASSWORD
export OPENCODE_SERVER_PASSWORD
opencode attach https://opencode.example.com
```

Windows PowerShell 客户端按第 4.1 节设置 `OPENCODE_SERVER_PASSWORD` 后，再执行同一条 `opencode attach` 命令。

上述命令只准备 OpenCode 后端，不会自动安装或配置反向代理。还需要在 Caddy、Nginx 或团队现有网关中配置域名、TLS 证书、请求转发和访问控制。

反向代理位于另一台主机时，才需要让 OpenCode 监听特定私有网络地址或 `0.0.0.0`。此时防火墙必须只允许反向代理主机访问 4096 端口，不能直接向公网开放。

HTTP Basic Auth 不加密用户名、密码或项目内容。防火墙应限制对 OpenCode 明文端口的访问，公网流量必须先经过 HTTPS、VPN 或其他受保护的传输通道。

## 5. 健康检查与原生长期运行

### 5.1 健康检查与接口文档

在 Server 本机检查健康状态：

```bash
curl --user "${OPENCODE_SERVER_USERNAME:-opencode}" \
  http://127.0.0.1:4096/global/health
```

`curl` 会交互提示输入密码，密码不会作为命令参数出现在进程列表中。未设置 Server 密码时不需要 `--user`。

Windows PowerShell：

```powershell
Invoke-RestMethod http://127.0.0.1:4096/global/health
```

Server 设置了密码时，使用 Basic Auth：

```powershell
$securePassword = Read-Host "OpenCode Server password" -AsSecureString
$password = [System.Net.NetworkCredential]::new("", $securePassword).Password
$username = if ([string]::IsNullOrWhiteSpace($env:OPENCODE_SERVER_USERNAME)) { "opencode" } else { $env:OPENCODE_SERVER_USERNAME }
$bytes = [System.Text.Encoding]::UTF8.GetBytes("${username}:$password")
$headers = @{ Authorization = "Basic $([Convert]::ToBase64String($bytes))" }
Invoke-RestMethod -Uri "http://127.0.0.1:4096/global/health" -Headers $headers
```

OpenAPI 文档默认位于 `http://127.0.0.1:4096/doc`。配置 HTTPS 反向代理后，使用对应的 HTTPS 域名访问。

### 5.2 使用 systemd 长期运行

直接执行 `opencode serve` 适合测试，终端退出后进程也会停止。长期运行应使用操作系统服务管理器，以便统一管理运行账户、环境变量、日志和异常重启。

先通过 `command -v opencode` 确认实际可执行文件路径。下面的 `/usr/local/bin/opencode`、`opencode` 用户和项目路径都是示例，必须按实际环境修改。

以下示例使用专用的 `opencode` 系统账户。先创建账户；如果团队已经有符合最小权限要求的服务账户，可以复用现有账户：

```bash
sudo useradd --system --create-home --shell /bin/bash opencode
```

通过用户组或 ACL 授予该账户目标项目的必要权限，不要直接把无关目录交给它。先以实际运行身份检查项目和基础工具：

```bash
sudo -u opencode env -i \
  HOME=/home/opencode \
  PATH=/usr/local/bin:/usr/bin:/bin \
  /bin/bash -c \
  'cd /srv/projects/example && /usr/local/bin/opencode --version && git status --short'
```

服务账户不会继承当前管理员账户的 `~/.config/opencode/opencode.json` 或 `auth.json`。默认模型应放在项目级 `opencode.json`、服务账户自己的配置目录，或通过下面的环境文件配置所需凭据。项目工具位于自定义目录时，还需要在 systemd 单元中显式设置 `Environment=PATH=...`。

先创建受保护的配置目录：

```bash
sudo install -d -m 0750 -o root -g opencode /etc/opencode
```

然后创建 `/etc/opencode/server.env`：

```text
OPENCODE_SERVER_USERNAME=opencode
OPENCODE_SERVER_PASSWORD=replace-with-a-strong-password
DEEPSEEK_API_KEY=replace-with-your-api-key
```

这里只用 DeepSeek 演示模型凭据。使用其他提供商时，应替换为第 2 章列出的环境变量。让文件归 `root` 所有，并只允许 `opencode` 组读取：

```bash
sudo chown root:opencode /etc/opencode/server.env
sudo chmod 640 /etc/opencode/server.env
```

创建 `/etc/systemd/system/opencode-server.service`：

```ini
[Unit]
Description=OpenCode Server
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=opencode
WorkingDirectory=/srv/projects/example
Environment=HOME=/home/opencode
Environment=PATH=/usr/local/bin:/usr/bin:/bin
EnvironmentFile=/etc/opencode/server.env
ExecStart=/usr/local/bin/opencode serve --hostname 127.0.0.1 --port 4096
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

加载并启动服务：

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now opencode-server
sudo systemctl status opencode-server
sudo journalctl -u opencode-server -f
```

服务启动后，按第 4.1 或 4.2 节从客户端连接 Server，并发送一次只读模型请求。只有该请求成功，才能确认 systemd 运行账户、项目权限、模型凭据和服务环境均已正确配置。

`opencode` 运行账户必须事先存在，并且只应拥有目标项目和必要开发工具的权限。不要使用 `root` 运行 Server。macOS 可以使用 `launchd`，Windows 可以使用团队已有的服务管理工具；核心要求同样是固定低权限账户、受保护的凭据、日志和自动重启。

## 6. Docker 部署 Server

容器化适合流水线、短期隔离环境或已有容器运维体系的团队，不建议作为第一次使用 OpenCode 的默认方案。运行前先进入需要处理的项目目录；下面分别提供 Bash 和 Windows PowerShell 示例。

OpenCode 官方镜像是最小运行镜像，不保证包含 Git、Bash、Node.js、Python、编译器或项目包管理器。先根据项目需要构建派生镜像，或确认镜像中已经具备代理将调用的命令。例如，可以用以下 Dockerfile 作为起点：

```dockerfile
FROM ghcr.io/anomalyco/opencode

RUN apk add --no-cache bash git nodejs npm python3
```

这只是示例，实际工具链应与项目一致；生产环境还应固定基础镜像版本或摘要。构建后，将后续命令中的 `ghcr.io/anomalyco/opencode` 替换为团队自己的镜像名称。

### 6.1 准备密码、模型凭据和持久化卷

```bash
read -rsp "OpenCode Server password: " OPENCODE_SERVER_PASSWORD
export OPENCODE_SERVER_PASSWORD
read -rsp "DeepSeek API Key: " DEEPSEEK_API_KEY
export DEEPSEEK_API_KEY
docker volume create opencode-data
docker volume create opencode-config
```

Windows PowerShell：

```powershell
$securePassword = Read-Host "OpenCode Server password" -AsSecureString
$env:OPENCODE_SERVER_PASSWORD = [System.Net.NetworkCredential]::new("", $securePassword).Password
$secureApiKey = Read-Host "DeepSeek API Key" -AsSecureString
$env:DEEPSEEK_API_KEY = [System.Net.NetworkCredential]::new("", $secureApiKey).Password
docker volume create opencode-data
docker volume create opencode-config
```

示例使用 `DEEPSEEK_API_KEY`。如果使用其他模型提供商，应替换为第 2 章列出的环境变量。另一种方式是先运行共享相同数据卷的一次性认证容器：

```bash
docker run --rm -it \
  -v opencode-data:/root/.local/share/opencode \
  -v opencode-config:/root/.config/opencode \
  ghcr.io/anomalyco/opencode \
  auth login
```

Windows PowerShell：

```powershell
docker run --rm -it `
  -v opencode-data:/root/.local/share/opencode `
  -v opencode-config:/root/.config/opencode `
  ghcr.io/anomalyco/opencode `
  auth login
```

认证信息会保存在 `opencode-data` 中。使用环境变量时不需要执行一次性认证容器；使用认证卷时，可以从后续 Server 命令中删除对应的模型 API Key 环境变量。

新建的 `opencode-config` 卷不会自动复制宿主机的全局配置。最简单的方式是把默认模型和提供商配置写在项目根目录的 `opencode.json` 中，使其随 `$PWD` 一起挂载；也可以在启动前将经过审核的全局配置写入 `opencode-config`。启动 Server 前，应从使用相同挂载和环境变量的临时容器中执行 `opencode models`，确认模型可见。

连接宿主机上的 Ollama、LM Studio 等本地推理服务时，配置中的 `127.0.0.1` 指向容器自身。macOS 和 Windows 通常可以使用 `host.docker.internal`；Linux 需要配置宿主机网关或明确的私有地址，并限制该端口的访问范围。

### 6.2 启动容器

```bash
docker run --name opencode-server -d --restart unless-stopped \
  -p 127.0.0.1:4096:4096 \
  -v "$PWD:/workspace" \
  -v opencode-data:/root/.local/share/opencode \
  -v opencode-config:/root/.config/opencode \
  -w /workspace \
  -e OPENCODE_SERVER_USERNAME=opencode \
  -e OPENCODE_SERVER_PASSWORD \
  -e DEEPSEEK_API_KEY \
  ghcr.io/anomalyco/opencode \
  serve --hostname 0.0.0.0 --port 4096
```

Windows PowerShell 使用反引号续行：

```powershell
docker run --name opencode-server -d --restart unless-stopped `
  -p 127.0.0.1:4096:4096 `
  -v "${PWD}:/workspace" `
  -v opencode-data:/root/.local/share/opencode `
  -v opencode-config:/root/.config/opencode `
  -w /workspace `
  -e OPENCODE_SERVER_USERNAME=opencode `
  -e OPENCODE_SERVER_PASSWORD `
  -e DEEPSEEK_API_KEY `
  ghcr.io/anomalyco/opencode `
  serve --hostname 0.0.0.0 --port 4096
```

该示例在后台运行容器，并在 Docker 服务重启后自动恢复。使用以下命令查看日志：

```bash
docker logs -f opencode-server
```

### 6.3 管理和更新容器

常用生命周期命令：

```bash
docker stop opencode-server
docker start opencode-server
docker restart opencode-server
docker logs -f opencode-server
```

修改端口、挂载、环境变量或升级镜像时，需要删除并重新创建容器：

```bash
docker pull ghcr.io/anomalyco/opencode
docker stop opencode-server
docker rm opencode-server
```

然后重新执行第 6.2 节的 `docker run`。删除容器不会删除 `opencode-data` 和 `opencode-config` named volume；不要执行 `docker volume rm`，除非确认不再需要其中的认证信息、配置和会话。

示例使用 Docker named volume，避免容器以 root 身份在宿主机的 OpenCode 配置目录中创建文件。端口只绑定到宿主机回环地址；跨主机访问应额外配置 SSH 隧道或 HTTPS 反向代理。

生产环境还应满足：

- 固定镜像版本或摘要，不依赖浮动标签。
- 通过 Secret 注入模型凭据和服务密码。
- 限制容器用户、挂载目录、网络和可用资源。
- 评估容器写入宿主项目后产生的文件属主和权限。
- 在入口代理终止 TLS，不直接暴露明文 HTTP。

## 7. 安全要求

- 不要将未认证的 `opencode serve` 或 `opencode web` 暴露到公网。
- 通过防火墙或安全组限制可信来源。
- 使用专用低权限操作系统账户运行 Server。
- 只授予必要的项目目录、命令和网络权限。
- 使用服务管理器、Secret 管理系统或受保护的环境文件注入密码和 API Key。
- 限制可用工具和 Bash 命令，避免远程客户端获得不必要的系统权限。
- 对生产项目启用日志、资源限制和异常退出后的自动恢复。

远程客户端通过认证后，仍可能调用文件修改和命令执行工具。可以在 Server 项目的 `opencode.json` 中设置更严格的默认权限，并关闭会话分享：

```json
{
  "$schema": "https://opencode.ai/config.json",
  "share": "disabled",
  "permission": {
    "*": "ask",
    "bash": {
      "*": "ask",
      "rm *": "deny",
      "sudo *": "deny",
      "git push": "deny",
      "git push *": "deny"
    }
  }
}
```

权限规则采用最后匹配优先。无人值守的 SDK 或自动化任务无法交互确认 `ask` 时，应根据任务需要改成明确的 `allow` 和 `deny`，不要直接使用全局 `allow`。

## 8. 常见问题

### 8.1 无法连接远程 Server

依次检查：

- Server 是否监听预期地址和端口。
- 防火墙、安全组、DNS 和反向代理是否配置正确。
- 客户端使用的 HTTP/HTTPS 协议是否与实际入口一致。
- 用户名和密码是否与 Server 环境变量一致。
- `/global/health` 是否可以从预期网络访问。

### 8.2 Docker 退出后配置丢失

确认已经挂载 OpenCode 的数据目录和配置目录。只挂载项目目录不会保留容器内的认证信息、配置和会话。

### 8.3 Server 可以启动但模型不可用

模型请求由 Server 发起。进入 Server 所在主机或容器，按照 [02_Model_Deployment.md](./02_Model_Deployment.md) 检查模型凭据、环境变量、网络和默认模型配置。

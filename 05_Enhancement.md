# OpenCode 扩展能力：Skill、MCP 与 Plugin

> 本文介绍 Skill、MCP 和 Plugin 的工作原理，并分别完成一次查找、安装、使用、验证和卸载。开始前应已经完成 [01_Installation.md](./01_Installation.md)、[02_Model_Deployment.md](./02_Model_Deployment.md) 中的基础配置。
>
> Skill、MCP Server 和 Plugin 可能来自第三方。安装前必须检查来源、代码、权限和数据流向。本文提到的社区扩展不是 OpenCode 官方安全背书，版本和服务条款也可能变化。

## 1. 三种扩展机制概述

Skill、MCP 和 Plugin 都可以增强 OpenCode 的使用体验：

```text
Skill：
用户任务 -> 模型 -> 按需读取 SKILL.md -> 按其中的流程工作

MCP：
用户任务 -> 模型 -> MCP 工具 -> MCP Server -> 外部 API、数据或程序

Plugin：
OpenCode 启动或发生事件 -> Plugin Hook -> 补充或修改 OpenCode 的行为
```
### 1.1 功能定位与适用场景

| 对比项 | Skill | MCP | Plugin |
| --- | --- | --- | --- |
| 核心作用 | 提供可复用的流程和专业知识 | 连接外部工具、数据和服务 | 扩展或修改 OpenCode 本身 |
| 主要形式 | `SKILL.md`，可附带脚本和资料 | 本地或远程 MCP Server | JavaScript/TypeScript 模块或 npm 包 |
| 加载时机 | 代理识别到相关任务后按需加载 | OpenCode 启动时连接，模型需要时调用工具 | OpenCode 启动时加载，相关事件发生时运行 Hook |
| 是否增加模型工具 | 通常不会 | 会 | 可以增加，也可以拦截已有工具 |
| 上下文消耗 | 主要发生在 Skill 被加载后 | 工具定义和调用结果会占用上下文 | 取决于插件是否注入消息或工具 |
| 常见用途 | 测试流程、代码审查规范、发布步骤 | 查文档、查监控、操作浏览器或外部平台 | 通知、日志、修改工具参数、自定义认证 |
| 主要风险 | 错误或恶意指令、附带脚本 | 数据外发、凭据泄露、外部写操作、Token 消耗 | 可读取运行上下文、执行代码并改变工具行为 |
| 初学者建议 | 优先学习 | 只启用当前需要的少量 Server | 有明确需求时再安装 |

Skill 和规则文件也不完全相同。`AGENTS.md` 或 `instructions` 通常会持续影响项目会话；Skill 先以名称和描述出现在可用列表中，完整内容由代理在相关任务中按需加载，因此更适合体积较大或只在特定任务中使用的说明。

### 1.2 默认扩展与配置状态

普通 OpenCode 安装的默认情况如下：

| 类型 | 默认情况 |
| --- | --- |
| Skill 工具 | OpenCode 内置 `skill` 工具，用于加载具体 Skill |
| 内置 Skill | 当前版本包含 `customize-opencode`，用于修改 OpenCode 自身配置、Skill、MCP、Plugin 和权限规则 |
| 用户 Skill | 默认没有，但 OpenCode 会扫描项目和用户配置目录中的 `SKILL.md` |
| MCP Server | 普通独立安装默认没有配置 MCP Server |
| 社区 Plugin | 默认没有安装，配置中的 `plugin` 列表为空 |
| 内部 Plugin | OpenCode 内置模型提供商和认证集成，例如 OpenAI Codex、GitHub Copilot、GitLab、Azure 和 xAI 等；它们不是用户安装的社区插件 |

组织可以通过 `.well-known/opencode` 下发默认 MCP 或其他配置，所以受组织管理的环境可能与上表不同。OpenCode 的 `websearch` 也可能使用托管服务，但它属于内置工具能力，不等于用户在 `mcp` 字段中配置了一个 Server。

扩展不是越多越好。大量 Skill 会增加选择难度，大量 MCP 工具会占用上下文，而 Plugin 会增加启动失败和供应链风险。

## 2. Skill：提供可复用的工作方法

### 2.1 Skill 的加载机制

一个 Skill 是包含 `SKILL.md` 的目录。文件开头的 YAML frontmatter 至少需要 `name` 和 `description`：

```markdown
---
name: example-skill
description: Use when performing an example task that requires a fixed workflow.
---

# Example Skill

1. 检查输入。
2. 执行任务。
3. 验证结果。
```

OpenCode 的处理过程是：

1. 启动时扫描 Skill 目录。
2. 将每个 Skill 的名称和描述告诉代理，而不是立即加载全部正文。
3. 代理根据当前任务判断是否需要某个 Skill。
4. 代理调用内置 `skill` 工具加载完整的 `SKILL.md` 和相关文件列表。
5. Skill 内容进入当前上下文，代理按其中步骤工作。

`description` 不只是给人阅读的简介，也是代理选择 Skill 的主要依据。它应同时说明“做什么”和“什么时候使用”。Skill 可以附带脚本和参考资料；虽然 Markdown 本身不会自动执行代码，但其中的指令可能引导代理调用 Bash 或其他工具，因此仍然需要审查。

### 2.2 Skill 的目录结构与发现路径

常用目录如下：

| 作用域 | 路径 |
| --- | --- |
| OpenCode 项目级 | `.opencode/skills/<name>/SKILL.md` |
| OpenCode 全局 | `~/.config/opencode/skills/<name>/SKILL.md` |
| Agent Skills 项目兼容目录 | `.agents/skills/<name>/SKILL.md` |
| Agent Skills 全局兼容目录 | `~/.agents/skills/<name>/SKILL.md` |
| Claude 项目兼容目录 | `.claude/skills/<name>/SKILL.md` |
| Claude 全局兼容目录 | `~/.claude/skills/<name>/SKILL.md` |

项目级 Skill 可以随项目提交到 Git，适合团队共享。全局 Skill 会影响当前用户打开的多个项目，适合个人长期使用的通用流程。

`name` 必须由小写字母、数字和单个连字符组成，长度为 1 到 64 个字符，并与所在目录名称一致。文件名必须是大写的 `SKILL.md`。

### 2.3 Skill 的获取渠道与审查要点

可以从以下入口查找：

- [OpenCode Skill 文档](https://opencode.ai/docs/zh-cn/skills/)：确认 OpenCode 支持的目录、格式和权限。
- [Skills.sh](https://skills.sh/)：搜索兼容 Agent Skills 格式的 Skill，并查看安装命令和安全扫描结果。
- [Anthropic Skills](https://github.com/anthropics/skills)：文档、前端设计和 Skill 编写示例。
- [Vercel Agent Skills](https://github.com/vercel-labs/agent-skills)：React、Web 设计和文档规范等 Skill。
- Skill 的源码仓库：最终应直接检查 `SKILL.md`、脚本、依赖和许可证。

找到 Skill 后至少检查：

1. `SKILL.md` 是否要求删除文件、修改 Git 历史、发布制品或上传数据。
2. `scripts/` 中是否存在安装软件、访问网络或读取凭据的代码。
3. Skill 是否依赖某个代理专有功能。Agent Skills 的基础格式可移植，但 Hook 等功能不一定兼容 OpenCode。
4. 仓库维护者、最近更新、Issue 和许可证是否可信。
5. Skill 的作用是否可以由项目已有的 `AGENTS.md` 或命令更简单地实现。

### 2.4 `verification-before-completion` 的查找、安装与使用

本节以 [obra/superpowers](https://github.com/obra/superpowers) 中的 `verification-before-completion` 为例，完整演示如何查找、审查、下载、安装和使用一个 Skill。它要求代理在声称“修复完成”“测试通过”或“构建成功”之前，实际运行对应验证命令并检查结果。这条原则适用于大多数语言和项目，也有助于初级开发者区分“代码已经修改”和“问题已经验证”。

**第 1 步：查找并确认来源**

在 [Skills.sh](https://skills.sh/) 搜索 `verification-before-completion`，进入 [Skill 详情页](https://skills.sh/obra/superpowers/verification-before-completion)，确认维护仓库是 `obra/superpowers`，并查看页面提供的安装命令、安全扫描和源码入口。随后打开仓库中的 `SKILL.md`，确认它主要约束完成前的验证流程，没有要求上传源码或执行与验证无关的命令。

**第 2 步：准备安装工具**

本节使用 Vercel 维护的第三方 `skills` CLI 下载 Skill。该工具支持 OpenCode，但不是 OpenCode 自带命令。首次使用 `npx` 时会临时下载 CLI；它默认收集匿名安装遥测，可以通过 `DISABLE_TELEMETRY=1` 关闭。

先确认本机有 Node.js 和 `npx`：

```bash
node --version
npx --version
```

如果命令不存在，需要先安装 Node.js LTS，或者按照 Skill 仓库的说明手动将完整 Skill 目录复制到 `.opencode/skills/`。只下载一个 `SKILL.md` 之前，要确认该 Skill 是否还依赖同目录中的脚本和参考资料。

**第 3 步：查看并安装指定 Skill**

先让 CLI 列出仓库中可安装的 Skill，不执行安装：

```bash
npx skills add https://github.com/obra/superpowers --list
```

确认列表中存在 `verification-before-completion` 后，在项目根目录安装：

```bash
npx skills add https://github.com/obra/superpowers \
  --skill verification-before-completion \
  --agent opencode \
  --copy \
  --yes
```

其中：

- `--skill` 只选择这一项，避免一次安装仓库中的全部 Skill。
- `--agent opencode` 指定为 OpenCode 安装。
- `--copy` 复制文件而不是创建符号链接，方便初学者检查实际内容。
- 未使用 `--global`，因此默认安装到当前项目。

安装后应出现：

```text
.agents/skills/verification-before-completion/SKILL.md
```

使用编辑器打开 `SKILL.md` 再次检查实际安装内容，并查看 Git 状态：

```bash
git status --short
```

首次安装的文件尚未被 Git 跟踪，普通 `git diff` 不会显示其内容。如果项目准备共享这个 Skill，可以将文件提交到 Git；如果只供个人试用，应避免误提交。

**第 4 步：重新启动并使用 Skill**

安装后退出并重新启动 OpenCode，使新的 Skill 被发现。然后发出一个包含实际验证目标的任务，例如：

```text
检查当前项目是否可以正常构建，并说明结论。使用 verification-before-completion skill，不要修改文件。
```

预期行为是代理先识别项目的构建命令，实际执行命令并读取退出状态，然后才报告能否构建。可以使用 `/details` 查看工具调用详情，确认代理加载了 `verification-before-completion`，并且没有只根据代码外观猜测结果。

也可以不显式指定 Skill：

```text
检查测试是否全部通过，并给出证据，不要修改文件。
```

如果 `description` 与任务匹配，代理可以自行加载。是否每次都自动选择仍取决于模型判断；关键任务可以直接在提示词中指定 Skill 名称。

**第 5 步：验证完整闭环**

本次实践满足以下条件才算完成：

1. `.agents/skills/verification-before-completion/SKILL.md` 已生成并经过人工检查。
2. 重新启动 OpenCode 后，代理能够加载该 Skill。
3. `/details` 中可以看到 `skill` 工具调用。
4. 代理实际执行了测试或构建命令，再根据输出报告结果。

如果只看到代理复述 Skill 内容，却没有运行能够证明结论的命令，说明 Skill 已加载，但任务验证尚未完成。

### 2.5 Skill 的权限管理

可以在 `opencode.json` 中控制代理是否允许加载特定 Skill：

```json
{
  "$schema": "https://opencode.ai/config.json",
  "permission": {
    "skill": {
      "*": "ask",
      "verification-before-completion": "allow"
    }
  }
}
```

权限值含义如下：

- `allow`：直接加载。
- `ask`：加载前请求用户确认。
- `deny`：对代理隐藏并拒绝加载。

规则采用最后匹配优先，因此先写通用规则，再写具体例外。修改配置后需要退出并重新启动 OpenCode。

### 2.6 Skill 的更新与卸载

查看已经安装的 Skill：

```bash
npx skills list --agent opencode
```

更新当前项目中的 Skill：

```bash
npx skills update verification-before-completion
```

更新会改变本地文件。执行后应重新审查 `SKILL.md`、脚本和 Git 差异，不能因为旧版本可信就自动信任新版本。

卸载：

```bash
npx skills remove verification-before-completion --agent opencode
```

重新启动 OpenCode 后，再发送需要该 Skill 的任务，确认它已经不在可用列表中。如果安装时没有使用 `skills` CLI，也可以删除对应 Skill 目录，但应先确认该目录中没有自己的修改。

### 2.7 其他推荐 Skill

| Skill | 来源 | 适用场景 |
| --- | --- | --- |
| `skill-creator` | Anthropic Skills | 创建、检查和改进 Skill |
| `frontend-design` | Anthropic Skills | 实现具有明确视觉风格的前端页面 |
| `react-best-practices` | Vercel Agent Skills | React 和 Next.js 性能检查 |
| `web-design-guidelines` | Vercel Agent Skills | 可访问性、交互和 Web UI 审查 |
| `pdf`、`docx`、`pptx`、`xlsx` | Anthropic Skills | 生成或处理对应文档 |

不建议根据排行榜一次安装大量 Skill。先根据项目类型选择一项，完成审查和验证后再增加下一项。

## 3. MCP：连接外部工具和服务

### 3.1 MCP 的客户端与服务器机制

MCP 全称 Model Context Protocol。OpenCode 充当 MCP Client，连接一个或多个 MCP Server，并把 Server 提供的工具注册给模型：

```text
模型
  |
  | 调用 context7_query-docs 等工具
  v
OpenCode MCP Client
  |
  | MCP 协议
  v
本地或远程 MCP Server
  |
  v
文档、浏览器、监控平台、数据库或其他服务
```

OpenCode 支持两种连接方式：

| 类型 | 启动和通信方式 | 典型配置 | 特点 |
| --- | --- | --- | --- |
| 本地 MCP | OpenCode 启动本机命令，通过标准输入输出通信 | `type: "local"`、`command` | 数据可留在本机，但需要安装运行时和依赖 |
| 远程 MCP | OpenCode 通过网络连接服务 URL | `type: "remote"`、`url` | 配置简单，但请求和必要数据会发送到远程服务 |

MCP 工具与 `read`、`bash` 等内置工具一起提供给模型。Server 越多、工具定义越多，占用的上下文越大，模型也越难准确选择工具。因此不应把所有可能有用的 MCP 都默认启用。

普通 OpenCode 安装默认没有用户配置的 MCP Server。组织环境可能通过远程配置下发 MCP，可以先运行下面的命令查看当前环境：

```bash
opencode mcp list
```

### 3.2 MCP Server 的获取渠道与审查要点

优先从以下来源查找：

- [OpenCode MCP 文档](https://opencode.ai/docs/zh-cn/mcp-servers/)：包含 OpenCode 配置方法及 Context7、Sentry、Grep 示例。
- [MCP Registry](https://registry.modelcontextprotocol.io/)：查找已发布的 MCP Server。
- 服务商官方文档：例如 Sentry、GitHub 或浏览器工具维护者提供的安装说明。
- [MCP 参考服务器](https://github.com/modelcontextprotocol/servers)：用于学习协议和测试；仓库明确说明参考实现不等于生产级服务。
- Server 的源码仓库和包注册表：检查代码、版本、依赖、维护状态和许可证。

安装前至少回答以下问题：

1. Server 会收到哪些提示词、源码、错误日志或账号数据。
2. 它提供只读工具还是包含创建、修改和删除操作。
3. 本地 Server 的 npm、Python 或二进制包来自哪里。
4. 远程 Server 使用 API Key、请求头还是 OAuth，Token 具有哪些权限。
5. 是否可以使用测试账号、只读账号或更小的授权范围。
6. 是否真的需要它，还是 OpenCode 的 `webfetch`、`websearch`、`grep` 和 `bash` 已经足够。

### 3.3 Context7 的查找、配置与使用

本节以 [Context7](https://github.com/upstash/context7) 为例，完整演示如何查找、审查、配置、连接和使用一个远程 MCP Server。Context7 用于检索当前版本的库和框架文档，适合解决模型训练数据过时、API 名称变化和版本差异等问题，也是 OpenCode 官方 MCP 文档直接提供的示例。

**第 1 步：查找并确认来源**

先在 [OpenCode MCP 文档](https://opencode.ai/docs/zh-cn/mcp-servers/#context7) 中找到 Context7 示例，再打开 [Context7 源码仓库](https://github.com/upstash/context7) 核对远程地址、认证方式和可用工具。确认将使用的地址为 `https://mcp.context7.com/mcp`，并了解查询内容会发送到远程服务。

**第 2 步：选择连接方式**

Context7 支持本地 CLI 和远程 MCP。本节采用远程 MCP，不需要安装本地 npm 包。请求会发送到 Context7 服务，不应使用它查询私有源码、密钥或内部文档。

**第 3 步：写入配置并建立连接**

在项目根目录创建或修改 `opencode.json`。如果文件已经存在，只添加 `mcp.context7`，不要覆盖原有模型、提供商或权限配置：

```json
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": {
    "context7": {
      "type": "remote",
      "url": "https://mcp.context7.com/mcp",
      "enabled": true
    }
  }
}
```

项目级配置会随项目共享。如果希望所有项目都能使用，可以改为写入：

```text
~/.config/opencode/opencode.json
```

修改配置后退出并重新启动 OpenCode。配置只在启动时加载，当前运行中的会话不会自动读取新配置。

**第 4 步：检查连接状态**

在普通终端查看状态：

```bash
opencode mcp list
```

确认 `context7` 显示为已连接。如果连接失败，使用调试日志启动 OpenCode：

```bash
opencode --print-logs --log-level DEBUG
```

**第 5 步：调用 Context7 工具**

在 TUI 中发送一个带具体库和版本的问题：

```text
查询 React 当前文档中 useEffectEvent 的用途和限制，使用 context7，只总结文档，不修改文件。
```

打开 `/details`，应看到名称以 `context7_` 开头的 MCP 工具调用。使用 MCP 的结果仍然需要核对来源和项目实际版本，不能把检索结果直接视为正确实现。

如果希望代理在需要库文档时主动使用 Context7，可以在项目 `AGENTS.md` 中添加：

```markdown
When library or API behavior may depend on its current version, use Context7 and verify the installed version before changing code.
```

不要写成“任何问题都必须使用 Context7”，否则会产生不必要的网络请求和 Token 消耗。

**第 6 步：验证完整闭环**

本次实践满足以下条件才算完成：

1. `opencode mcp list` 显示 `context7` 已连接。
2. `/details` 中出现名称以 `context7_` 开头的工具调用。
3. 回答包含与问题相关的当前文档信息，而不是只依赖模型原有知识。
4. 项目文件没有因为文档查询被意外修改。

### 3.4 Context7 的可选 API Key 配置

Context7 无 API Key 也可以使用，但注册账号后可能获得更高的速率限制。不要把真实 Key 直接写入 `opencode.json`。先在启动 OpenCode 的环境中设置：

```bash
export CONTEXT7_API_KEY='your-context7-api-key'
opencode
```

Windows PowerShell：

```powershell
$env:CONTEXT7_API_KEY = "your-context7-api-key"
opencode
```

然后在配置中引用环境变量：

```json
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": {
    "context7": {
      "type": "remote",
      "url": "https://mcp.context7.com/mcp",
      "headers": {
        "Authorization": "Bearer {env:CONTEXT7_API_KEY}"
      }
    }
  }
}
```

OpenCode 的配置变量格式是 `{env:VARIABLE}`，不是 Shell 的 `${VARIABLE}`。如果环境变量没有设置，它会被替换为空字符串。

### 3.5 MCP 工具的权限控制

MCP 工具注册时使用 Server 名称作为前缀。可以对 Context7 的全部工具设置权限：

```json
{
  "$schema": "https://opencode.ai/config.json",
  "permission": {
    "context7_*": "allow"
  }
}
```

对具有写入、发布或删除能力的 MCP，初次使用时应改为：

```json
{
  "$schema": "https://opencode.ai/config.json",
  "permission": {
    "external-service_*": "ask"
  }
}
```

权限只能控制模型是否调用已注册工具，不能修复 MCP Server 自身的漏洞，也不能限制 Server 在启动后自行执行的内部逻辑。本地 MCP 仍应在低权限账户或隔离环境中运行，远程 MCP 仍应使用最小权限凭据。

### 3.6 Context7 的禁用、认证与卸载

临时禁用而不删除配置：

```json
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": {
    "context7": {
      "type": "remote",
      "url": "https://mcp.context7.com/mcp",
      "enabled": false
    }
  }
}
```

永久卸载时，从配置中删除 `mcp.context7`，保留其他 `mcp` 项。重新启动 OpenCode 后，执行：

```bash
opencode mcp list
```

确认列表中不再存在 `context7`。如果配置过 API Key，还应从 Shell 启动文件、Secret 管理系统或运行环境中移除对应变量。

远程 MCP 使用 OAuth 时，可以使用：

```bash
opencode mcp auth <server-name>
opencode mcp logout <server-name>
```

删除配置不会自动撤销服务商账号中的授权，必要时还应到服务商控制台撤销 Token 或 OAuth 应用授权。

### 3.7 其他推荐 MCP

| MCP | 用途 | 适用场景 | 主要注意事项 |
| --- | --- | --- | --- |
| Grep by Vercel | 搜索 GitHub 公开代码示例 | 查找真实开源项目用法 | 示例可能过时或质量不一 |
| Sentry | 查询项目、错误和未解决 Issue | 已使用 Sentry 的项目 | 可能接触生产错误和用户数据 |
| Playwright MCP | 操作和检查浏览器页面 | Web 开发和端到端验证 | 可以访问登录状态并执行网页操作 |
| GitHub MCP | 操作仓库、Issue 和 Pull Request | GitHub 工作流自动化 | 工具多、Token 消耗大，写权限风险高 |
| 数据库 MCP | 查询数据库结构和数据 | 数据分析和后端排障 | 优先使用测试库和只读账号 |

初学者建议先只保留 Context7。只有出现明确需求后，再逐个添加并验证其他 MCP。

## 4. Plugin：扩展 OpenCode 的运行行为

### 4.1 Plugin 的加载与 Hook 机制

Plugin 是 OpenCode 启动时加载的 JavaScript 或 TypeScript 模块。插件函数接收 OpenCode 客户端、项目目录和 Shell 等上下文，并返回一组 Hook：

```javascript
export const ExamplePlugin = async ({ project, client, $, directory, worktree }) => {
  return {
    event: async ({ event }) => {
      // 监听 OpenCode 事件
    },
    "tool.execute.before": async (input, output) => {
      // 工具执行前检查或修改参数
    }
  }
}
```

常见能力包括：

- 监听会话完成、错误、权限请求和文件修改等事件。
- 在工具执行前后检查或修改参数和结果。
- 为 Shell 注入环境变量。
- 修改模型消息、请求参数或上下文压缩过程。
- 注册新的模型工具。
- 集成认证、日志、监控和外部服务。

插件不是受限的提示词。它作为代码在 OpenCode 运行环境中执行，可能访问文件、环境变量和网络，也可能调用 Shell。OpenCode 的工具权限主要约束代理调用工具，不能完整隔离插件自身。因此 Plugin 的信任要求高于 Skill 和远程 MCP。

### 4.2 Plugin 的加载来源与安装方式

OpenCode 支持两种主要加载方式：

| 方式 | 位置或配置 | 安装过程 |
| --- | --- | --- |
| 本地插件 | `.opencode/plugins/*.js`、`.opencode/plugins/*.ts` 或对应全局目录 | 启动时直接加载文件 |
| npm 插件 | `opencode.json` 中的 `plugin` 数组 | 启动时由 Bun 自动安装并缓存 |

全局插件目录为：

```text
~/.config/opencode/plugins/
```

npm 插件会缓存在：

```text
~/.cache/opencode/node_modules/
```

OpenCode 会同时加载配置和插件目录中的插件。普通安装默认没有社区插件，但内部会加载模型提供商和认证相关的内置插件。内部插件不需要写入 `plugin` 数组，也不应重复安装。

### 4.3 Plugin 的获取渠道与审查要点

可以从以下入口查找：

- [OpenCode Plugin 文档](https://opencode.ai/docs/zh-cn/plugins/)：确认加载方式、Hook 和本地插件结构。
- [OpenCode 生态系统](https://opencode.ai/docs/zh-cn/ecosystem/#插件)：查看社区提交的插件列表。
- npm：确认包名、版本、发布时间、依赖和发布者。
- 插件 GitHub 仓库：检查源码、Issue、变更日志和许可证。

进入生态列表不代表 OpenCode 官方审计或担保。安装前至少检查：

1. npm 包和 GitHub 仓库是否由同一维护者发布。
2. 插件注册了哪些 Hook，是否读取消息、环境变量、文件或工具参数。
3. 是否执行外部命令或向第三方发送数据。
4. 是否有安装脚本、混淆代码或范围过大的依赖。
5. 是否支持当前操作系统和 OpenCode 版本。
6. 是否可以固定版本，避免每次启动获取不可预期的新代码。

### 4.4 `opencode-notifier` 的查找、安装与使用

本节以社区插件 [`@mohak34/opencode-notifier`](https://github.com/mohak34/opencode-notifier) 为例，完整演示如何查找、审查、安装、使用和验证一个 Plugin。它监听任务完成、错误、权限请求和提问事件，并发送系统通知或播放提示音。通知插件的行为直观，不会为模型增加大量工具，适合作为第一个 Plugin 示例。

**第 1 步：查找并确认来源**

先在 [OpenCode 生态系统](https://opencode.ai/docs/zh-cn/ecosystem/#插件) 中找到 `opencode-notifier`，再打开插件的 [GitHub 仓库](https://github.com/mohak34/opencode-notifier) 和 [npm 包页面](https://www.npmjs.com/package/@mohak34/opencode-notifier)。核对 npm 包名为 `@mohak34/opencode-notifier`，确认仓库说明、发布者、许可证和通知相关 Hook 与预期用途一致。

**第 2 步：确认并固定版本**

该插件不是 OpenCode 官方组件。开始前先查看 npm 当前版本：

```bash
npm view @mohak34/opencode-notifier@latest version
```

截至本文核对时，`latest` 为 `0.2.8`。下面固定该版本以保证配置可复现；如果准备使用新版本，应先检查 Release Notes 和源码，再替换版本号。

**第 3 步：写入插件配置**

在全局配置 `~/.config/opencode/opencode.json` 中加入插件，使它对当前用户的所有项目生效：

```json
{
  "$schema": "https://opencode.ai/config.json",
  "plugin": [
    "@mohak34/opencode-notifier@0.2.8"
  ]
}
```

如果文件已有其他配置，只把插件追加到现有 `plugin` 数组，不要覆盖 `model`、`provider`、`mcp` 或其他字段。只希望当前项目使用时，可以改为写入项目根目录的 `opencode.json`。

**第 4 步：准备操作系统依赖**

Linux 桌面通常需要通知命令。根据发行版选择一种安装方式：

```bash
sudo apt install libnotify-bin
```

```bash
sudo dnf install libnotify
```

```bash
sudo pacman -S libnotify
```

Linux 还需要 `paplay`、`aplay`、`mpv` 或 `ffplay` 中的一个才能播放声音。macOS 默认使用 AppleScript；Windows 使用系统通知。WSL 没有原生 Linux 桌面通知环境，可能需要按插件文档配置 PowerShell 通知。

退出并重新启动 OpenCode。首次启动时，OpenCode 会使用 Bun 自动下载 npm 插件及依赖，之后从缓存加载。

**第 5 步：触发并验证通知**

插件是事件驱动的，不需要在提示词中写“使用 notifier”。发送一个短任务：

```text
只读取 README.md，并用三点概括项目用途，不要修改文件。
```

任务开始后切换到其他窗口。任务结束时应出现系统通知。插件默认在 OpenCode 所在终端仍处于焦点时抑制通知，因此一直停留在 TUI 中可能看不到提示。

需要始终显示通知以便测试时，可以创建：

```text
~/.config/opencode/opencode-notifier.json
```

写入：

```json
{
  "sound": true,
  "notification": true,
  "suppressWhenFocused": false,
  "minDuration": 0
}
```

重新启动 OpenCode 并再次执行短任务。验证完成后，可以把 `suppressWhenFocused` 恢复为 `true`，减少通知干扰。

Linux 可以先独立验证桌面通知系统：

```bash
notify-send "OpenCode notification test" "Desktop notifications are available"
```

如果系统测试成功但插件没有通知，使用调试日志查看插件加载错误：

```bash
opencode --print-logs --log-level DEBUG
```

**第 6 步：验证完整闭环**

本次实践满足以下条件才算完成：

1. OpenCode 启动日志中没有插件安装或加载错误。
2. 任务完成后能够收到系统通知或声音提示。
3. 权限请求或问题事件仍由 OpenCode 正常处理，没有被插件阻断。
4. 插件只响应配置的事件，没有为模型增加无关工具或修改项目文件。

### 4.5 Plugin 的更新、禁用与卸载

先查看新版本：

```bash
npm view @mohak34/opencode-notifier@latest version
```

检查变更后，把 `plugin` 数组中的固定版本修改为目标版本并重新启动 OpenCode。不要在团队或重要项目中无审查地长期使用 `@latest`。

临时禁用时，从 `plugin` 数组中移除这一项并重新启动。卸载也是相同步骤：OpenCode 没有为配置中的 npm Plugin 提供单独卸载命令。

删除配置后，缓存文件可能仍保留在 `~/.cache/opencode/`，但不会继续加载。缓存通常可以留给 OpenCode 管理；只有排查版本缓存问题或确认不再需要时，才按照插件文档删除对应包，不要直接清空整个 OpenCode 数据目录。

需要紧急判断是否由外部 Plugin 导致启动问题时，可以在普通终端临时运行：

```bash
OPENCODE_PURE=1 opencode
```

该模式跳过外部插件，适合进入 OpenCode 后修复配置。修复后退出，再不带该变量重新启动。

### 4.6 其他推荐 Plugin

| Plugin | 用途 | 适用场景 | 主要注意事项 |
| --- | --- | --- | --- |
| `opencode-type-inject` | 在读取 TypeScript/Svelte 文件时补充类型信息 | TypeScript、Svelte 项目 | 会增加上下文，先在单个项目测试 |
| `opencode-dynamic-context-pruning` | 修剪过时工具输出 | 长会话、大量工具调用 | 可能减少模型可见的历史细节 |
| `opencode-wakatime` | 统计 OpenCode 使用时间 | 已使用 WakaTime 的开发者 | 会向外部统计服务发送数据 |
| `opencode-vibeguard` | 在发送模型请求前替换敏感信息 | 有隐私保护需求的环境 | 不能代替权限、Secret 管理和人工检查 |
| `opencode-helicone-session` | 为模型请求加入 Helicone 会话信息 | 已使用 Helicone 观测的团队 | 涉及外部请求追踪和数据治理 |

大型多代理、后台任务和 worktree 套件可能一次改变多个默认行为，不建议作为初学者的第一个 Plugin。

## 5. 安全检查清单

安装 Skill、MCP 或 Plugin 前检查：

- 来源是否为官方项目、可信维护者或经过团队审核的镜像。
- 是否查看了 `SKILL.md`、Plugin 源码或 MCP Server 源码，而不只看简介。
- 是否包含安装脚本、二进制文件、混淆代码或异常依赖。
- 是否会读取源码、提示词、环境变量、凭据、日志或个人数据。
- 是否向外部服务发送数据，服务条款是否允许处理当前项目内容。
- 是否包含创建、修改、删除、发布、支付或 Git 远程操作。
- API Key 和 OAuth 是否采用最小权限，能否使用只读账号或测试账号。
- npm 和其他包是否固定到审核过的版本。
- 是否只启用了当前任务需要的 Skill、Server 和工具。
- 是否知道如何禁用、卸载和撤销凭据。

特别需要注意：

- `permission` 可以控制代理调用 Skill 和工具，但不能把第三方 Plugin 变成安全沙箱。
- MCP Server 的工具描述也来自 Server，本身不一定可信。
- Skill 可能通过文字诱导代理执行高风险命令，不能因为它只是 Markdown 就忽略审查。
- 项目级 `opencode.json`、`.opencode/` 和 `.agents/skills/` 可能随 Git 仓库一起下载。首次打开陌生仓库前应检查这些文件。

## 6. 常见问题

### 6.1 安装后没有生效

Skill、MCP 和 Plugin 都在 OpenCode 启动过程中发现或加载。保存配置后应完全退出当前 OpenCode，再重新进入项目。继续旧会话不会让运行中的进程自动重新加载扩展。

### 6.2 Skill 没有出现在可用列表中

依次检查：

- 文件名是否为大写的 `SKILL.md`。
- frontmatter 是否包含 `name` 和 `description`。
- `name` 是否与目录名一致，并且只包含小写字母、数字和单连字符。
- 文件是否位于 OpenCode 支持的项目或全局目录。
- `permission.skill` 是否将它设为 `deny`。
- 是否存在同名 Skill 导致覆盖。

### 6.3 MCP 显示连接失败

先执行：

```bash
opencode mcp list
```

再检查 URL、网络、代理、DNS、API Key 和 OAuth 状态。本地 MCP 还要检查命令是否已安装、`PATH` 是否正确以及运行时依赖是否存在。需要进一步排查时使用：

```bash
opencode mcp debug <server-name>
```

或使用 OpenCode 调试日志。不要通过关闭 TLS 校验或在日志中暴露 Token 来规避连接问题。

### 6.4 启用 MCP 后 Token 消耗明显增加

每个 MCP Server 都可能注册多个工具，工具名称、说明和参数会占用上下文。禁用暂时不用的 Server，或全局关闭工具后只在特定代理中启用。不要同时启用多个功能重叠的搜索、浏览器或 GitHub MCP。

### 6.5 添加 Plugin 后 OpenCode 无法启动

先从普通终端禁用外部插件启动：

```bash
OPENCODE_PURE=1 opencode --print-logs --log-level DEBUG
```

检查并修复 `opencode.json` 的 `plugin` 数组，或者移除刚添加的本地插件。不要删除整个 `~/.config/opencode` 或 `~/.local/share/opencode`，其中可能包含其他配置、认证和会话数据。

### 6.6 通知插件已经加载但没有通知

依次确认：

- OpenCode 是否运行在 CLI/TUI；该插件默认不在 Desktop/Web 客户端启用。
- `suppressWhenFocused` 是否为 `true`，并尝试切换到其他窗口。
- 操作系统是否允许终端、Script Editor 或通知程序显示通知。
- Linux 是否安装 `notify-send`，并能通过独立测试命令显示通知。
- WSL 是否需要使用插件文档中的 PowerShell 方案。

### 6.7 如何判断问题来自哪一种扩展

按相反顺序逐项排除：

1. 使用 `OPENCODE_PURE=1` 排除外部 Plugin。
2. 将可疑 MCP 的 `enabled` 设为 `false`。
3. 将可疑 Skill 的权限设为 `deny` 或移出扫描目录。
4. 每次只改变一项并重新启动，记录错误是否消失。

## 7. 参考资料

- [OpenCode Skill 文档](https://opencode.ai/docs/zh-cn/skills/)
- [OpenCode MCP 文档](https://opencode.ai/docs/zh-cn/mcp-servers/)
- [OpenCode Plugin 文档](https://opencode.ai/docs/zh-cn/plugins/)
- [OpenCode 工具文档](https://opencode.ai/docs/zh-cn/tools/)
- [OpenCode 权限文档](https://opencode.ai/docs/zh-cn/permissions/)
- [OpenCode 配置文档](https://opencode.ai/docs/zh-cn/config/)
- [OpenCode 生态系统](https://opencode.ai/docs/zh-cn/ecosystem/)
- [Skills.sh](https://skills.sh/)
- [Vercel skills CLI](https://github.com/vercel-labs/skills)
- [verification-before-completion](https://skills.sh/obra/superpowers/verification-before-completion)
- [Context7](https://github.com/upstash/context7)
- [opencode-notifier](https://github.com/mohak34/opencode-notifier)
- [MCP Registry](https://registry.modelcontextprotocol.io/)

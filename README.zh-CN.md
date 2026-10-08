# Codex Kimi Bridge（中文说明）

[English](README.md) | **简体中文**

让 Codex 负责需求理解、任务规划和代码审查，让 Kimi Code 执行具体开发任务。本项目通过本地 MCP 服务连接两者，你可以直接在 Codex 对话中委派工作。

**本分支重点解决现代 Kimi 的自动启动和 Windows 使用体验。** 安装并完成 Kimi Code 自身的账号登录后，首次委派会按需启动 Kimi 服务，省去每次先打开终端运行启动命令的步骤。

本项目基于 [ximenchuifeng/codex-kimi-bridge](https://github.com/ximenchuifeng/codex-kimi-bridge)，保留原作者署名与 MIT 许可证。启动适配改动已通过 [上游 PR #1](https://github.com/ximenchuifeng/codex-kimi-bridge/pull/1) 提交。

## 本分支的特点

以下对比针对 Fork 时的上游代码：

| 使用场景 | 上游基线 | 本分支 |
| --- | --- | --- |
| 自动启动现代 Kimi | 固定使用 `kimi server run --keep-alive` | 插件默认使用 `kimi web --no-open --port 58627`，已验证 Kimi 2.1.1 |
| 自定义启动方式 | 启动参数写在代码中 | 新增 `KIMI_SERVER_ARGS`，通过 JSON 字符串配置参数，并保留旧版回退 |
| Windows 后台运行 | 后台启动未设置 `windowsHide` | 隐藏启动控制台，默认不自动打开浏览器 |
| 空数据目录配置 | 子进程可能继承空的 `KIMI_CODE_HOME` | 统一解析为 `~/.kimi-code`，使子进程与桥接服务使用一致的认证目录 |
| Windows 启动诊断 | 启动命令采用 POSIX 引号规则 | 使用 PowerShell 调用和引号规则，支持带空格、单引号的路径与参数 |
| 中文上手 | 主要说明使用英文 | 提供中文安装、配置、委派示例和常见问题说明 |

本分支沿用上游的任务委派、审查包、会话恢复等能力。已运行且健康的服务会被复用；`kimi_bridge_status` 只检查状态，不触发启动。仓库包含构建好的 MCP 安装包，首次安装无需 pnpm 或 TypeScript 编译。

## 快速安装

### 环境要求

- Git。
- Node.js 20 或更新版本。
- 支持插件的 Codex。
- 已安装并完成账号登录的 [Kimi Code](https://github.com/MoonshotAI/kimi-code)。现代启动方式已在 Kimi 2.1.1 上验证；其他版本可按下文调整参数。

首次使用 Kimi Code 时，可运行 `kimi`，再在其交互界面使用 `/login` 完成登录。Windows 用户需安装 Git for Windows，供 Kimi 使用 Git Bash。详细步骤见 [Kimi Code 官方安装与登录说明](https://github.com/MoonshotAI/kimi-code#install)。

先确认终端能找到所需命令：

```text
git --version
node --version
codex --version
kimi --version
```

如果 Codex 继承的 `PATH` 中找不到 `kimi`，请在安装前按下文设置可执行文件的完整路径。

### Windows PowerShell

在希望存放项目的目录打开 PowerShell，运行：

```powershell
git clone https://github.com/liuxiaofengone/codex-kimi-bridge.git
Set-Location codex-kimi-bridge
codex plugin marketplace add (Get-Location).Path
codex plugin add kimi-delegate@codex-kimi-bridge-local
```

### macOS / Linux

```bash
git clone https://github.com/liuxiaofengone/codex-kimi-bridge.git
cd codex-kimi-bridge
codex plugin marketplace add "$PWD"
codex plugin add kimi-delegate@codex-kimi-bridge-local
```

安装后打开一个新的 Codex 对话，使插件工具加载生效。可以运行 `codex plugin list`，确认 `kimi-delegate@codex-kimi-bridge-local` 已安装并启用。

## 在对话中委派 Kimi 工作

### 简单任务

直接在 Codex 对话中输入：

```text
请用 kimi-delegate 让 Kimi 完成这个需求：为当前项目增加登录表单的输入校验。
你负责制定计划、审查改动并运行验证，具体实现交给 Kimi。
完成后告诉我修改了哪些文件、验证结果和 Kimi 会话链接。
```

需要启动服务时，任务调用会自动执行默认启动命令；你无需提前手动运行 `kimi web`。

### 明确范围与验收标准

```text
请使用 kimi-delegate 完成以下任务。

项目：当前工作区。
需求：修复登录表单在邮箱为空时仍能提交的问题。
范围：只修改登录表单及相关测试。
验收标准：空邮箱不能提交；合法邮箱保持原有行为；相关测试通过。

Codex 负责分析、计划和审查，Kimi 负责实现。
委派时优先使用 kimi_delegate_and_wait，并用稳定任务关键词设置 dedupe，
保留默认的工作目录匹配，避免中断后重复创建任务。
审查不通过时，用 kimi_continue_task 让 Kimi 修正。
完成后给出会话链接、改动摘要和验证结果。
```

### 继续之前的任务

```text
请继续刚才的 Kimi 任务，补充手机号格式校验。
优先沿用原会话；如果会话信息不在当前上下文中，先按任务关键词查找并确认工作目录。
完成后继续审查并验证。
```

如果 Kimi 正在等待批准或回答问题，可以打开返回的 `webUrl` 处理，再继续同一会话。等待超时也可以继续调用 `kimi_wait_until_idle`；`failed` 或 `aborted` 会话不会被自动去重复用，需要检查原因后用 `kimi_continue_task` 继续。

更完整的主控、实现与审查模板见 [委派工作流示例](docs/prompts/kimi-delegate-workflow.md)。

## 启动与认证配置

配置文件为 [plugins/kimi-delegate/.mcp.json](plugins/kimi-delegate/.mcp.json)，相关变量放在 `mcpServers.kimi-delegate.env` 中。需要修改时，建议先配置再安装；已安装插件可按 [修改配置后重装](#修改配置后重装) 刷新缓存。

| 变量 | 插件默认值 | 用途 |
| --- | --- | --- |
| `KIMI_AUTO_START` | `true` | 服务未响应时，在任务调用前自动启动；设为 `false` 可手动管理服务 |
| `KIMI_COMMAND` | `kimi` | 启动的可执行文件，可设置为完整路径 |
| `KIMI_SERVER_ARGS` | `["web","--no-open","--port","58627"]`，以 JSON 字符串保存 | 配置启动参数；省略或设为空字符串时回退到旧版命令 |
| `KIMI_SERVER_URL` | `http://127.0.0.1:58627` | 本地服务地址，自定义端口时需要与启动参数一致 |
| `KIMI_CODE_HOME` | 空字符串，解析为 `~/.kimi-code` | Kimi 数据和认证目录；自定义时应与 Kimi Code 保持一致 |
| `KIMI_SERVER_TOKEN` | 空字符串 | 默认自动读取本地 token；需要显式覆盖时才设置 |
| `KIMI_PREFLIGHT_CACHE_MS` | `5000` | 成功预检查的缓存时间，单位为毫秒；`0` 表示禁用缓存 |

### 默认现代启动方式

下面是 `env` 中相关字段的配置示例：

```json
{
  "KIMI_AUTO_START": "true",
  "KIMI_COMMAND": "kimi",
  "KIMI_SERVER_URL": "http://127.0.0.1:58627",
  "KIMI_SERVER_ARGS": "[\"web\",\"--no-open\",\"--port\",\"58627\"]",
  "KIMI_CODE_HOME": "",
  "KIMI_SERVER_TOKEN": ""
}
```

`KIMI_SERVER_ARGS` 必须是 JSON 数组字符串，数组非空且每一项都是非空白字符串。参数直接传给可执行文件，不经过 shell 拆分。若改为端口 `60000`，请同时修改参数中的端口和 `KIMI_SERVER_URL`。

### Windows 可执行文件路径

如果 `kimi` 不在 Codex 继承的 `PATH` 中，先在 PowerShell 查询实际位置：

```powershell
(Get-Command kimi).Source
```

然后设置 `KIMI_COMMAND`。例如，下面是合法的 JSON 路径写法，请替换为你实际的安装位置：

```json
{
  "KIMI_COMMAND": "C:\\Program Files\\Kimi Code\\kimi.exe"
}
```

JSON 中的反斜杠需要转义，不要给配置值额外添加 PowerShell 引号或 `&`。

### 使用旧版 Kimi

如果已安装的 Kimi 提供 `server` 子命令，将 `KIMI_SERVER_ARGS` 设为 `""`，任务自动启动时就会回退到：

```bash
kimi server run --keep-alive
```

也可以按该版本提供的方式手动管理服务。桥接服务会复用已运行的健康实例。

### 自动读取认证信息

token 的读取顺序为：非空 `KIMI_SERVER_TOKEN`、自定义 `KIMI_CODE_HOME/server.token`、`~/.kimi-code/server.token`。本分支让自动启动的 Kimi 使用解析后的数据目录，修复空目录配置导致认证文件位置不一致的问题。

正常使用保留认证，不需要手工复制 token，也不需要关闭认证。发布配置中的 token 保持为空，请勿把真实凭据提交到仓库。

### 修改配置后重装

在注册为本地插件市场的仓库中修改 `.mcp.json`。待当前委派任务结束后，移除已安装插件及其本地缓存，再从该市场重新安装：

```text
codex plugin remove kimi-delegate@codex-kimi-bridge-local
codex plugin add kimi-delegate@codex-kimi-bridge-local
```

随后打开新的 Codex 对话。这种方式不依赖原作者机器上的缓存版本脚本，也无需为配置修改重新编译 MCP 安装包。若你的 Codex CLI 不提供相同命令，先运行 `codex plugin --help` 查看当前版本支持的操作。

## 常见问题

| 现象 | 处理方式 |
| --- | --- |
| 对话中没有 `kimi-delegate` 工具 | 用 `codex plugin list` 确认安装和启用状态，再打开新的 Codex 对话；必要时重启 Codex |
| 诊断为 `server_unreachable` | 检查可执行文件路径、端口和启动参数；状态检查本身不会启动服务，实际任务调用才会尝试自动启动 |
| 提示不支持 `server` 或 `web` 子命令 | 根据已安装版本的 `kimi --help` 调整 `KIMI_SERVER_ARGS`，参考现代配置或旧版回退说明 |
| 诊断为 `auth_failed` | 检查桥接服务与 Kimi 服务的数据目录及 `server.token` 是否一致；如显式设置了 `KIMI_SERVER_TOKEN`，确认它与当前服务 token 一致 |
| 需要查看 Kimi 的执行过程 | 打开工具返回的 `webUrl`；自动启动时默认不会替你打开浏览器 |
| 原任务中断或等待超时 | 保留会话 ID，先查询原任务状态，再继续等待或补充指令 |

`auth_failed` 表示桥接服务访问 Kimi 服务时的 token 认证失败，与 Kimi 模型账号登录是不同环节。`kimi_bridge_status` 不重新读取 token；实际任务的预检查会在认证失败时重新读取并重试，必要时也可重启 MCP 服务以重新加载认证配置。

需要检查服务时，可以在 Codex 对话中说：

```text
请调用 kimi_bridge_status，检查 Kimi 服务和认证状态，并解释返回的诊断结果。
```

## 验证与贡献

本分支此前在 Windows 上完成了 Kimi 2.1.1 的真实认证冷启动委派验证；50 项配置与预检查测试、4 项安装包检查、类型检查与构建均通过。完整测试为 268/278 通过，其余 10 项是已在未修改上游基线上确认的 Windows 路径、文件操作或外部验证脚本环境问题，详情见 [上游 PR #1](https://github.com/ximenchuifeng/codex-kimi-bridge/pull/1)。

本地开发可使用：

```bash
pnpm install
pnpm test
pnpm typecheck
pnpm build
```

修改 MCP 运行时代码后，应重新构建并提交 `plugins/kimi-delegate/mcp/server.mjs`。文档更新不需要重新构建安装包。

欢迎向本分支提交 [Pull Request](https://github.com/liuxiaofengone/codex-kimi-bridge/pulls)，也可以通过 [上游 Issues](https://github.com/ximenchuifeng/codex-kimi-bridge/issues) 反馈通用问题。反馈时请注明操作系统、Kimi 版本、启动配置和已脱敏的诊断结果。

项目使用 [MIT 许可证](LICENSE)。感谢原作者 [ximenchuifeng](https://github.com/ximenchuifeng) 提供委派与审查桥接的基础实现。

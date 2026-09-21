# GitHub MCP Server — 官方文档调研素材

调研日期：2026-09-20。来源：GitHub 官方仓库 README 与 docs/ 目录、docs.github.com 官方文档页。WebFetch 对 github.com 被拦，全部经 curl 抓取。

---

## 项目定位与版本基准
- URL: https://github.com/github/github-mcp-server（README.md）+ https://api.github.com/repos/github/github-mcp-server/releases/latest + go.mod
- 抓取日期: 2026-09-20
- 可用于章节: §1 简介与定位 / §7 资源链接
- 关键摘录:
  - "The GitHub MCP Server connects AI tools directly to GitHub's platform. This gives AI agents, assistants, and chatbots the ability to read repositories and code files, manage issues and PRs, analyze code, and automate workflows."
  - 五大用例：Repository Management / Issue & PR Automation / CI/CD & Workflow Intelligence / Code Analysis / Team Collaboration
  - 最新 release：v1.12.2（"GitHub MCP Server 1.12.2"，published 2026-09-16）；go.mod 要求 go 1.25.12
  - License: MIT。Go 实现的官方 server，仓库即主要文档源
  - npm 包 `@modelcontextprotocol/server-github` 已于 2025 年 4 月废弃（deprecated / no longer functional），应使用官方 Docker 镜像 `ghcr.io/github/github-mcp-server`

## Remote Server 总览（GitHub 托管）
- URL: https://raw.githubusercontent.com/github/github-mcp-server/main/README.md + docs/remote-server.md
- 抓取日期: 2026-09-20
- 可用于章节: §1 简介与定位 / §2 安装与配置
- 关键摘录:
  - Remote URL: `https://api.githubcopilot.com/mcp/`；GitHub 托管，"the easiest method for getting up and running"，无需本地运行时
  - 前提：支持 remote MCP 的宿主（VS Code 1.101+、Claude Desktop、Cursor、Windsurf 等）+ 相应 policy 开启
  - "The remote GitHub MCP server is built using this repository as a library, and binding it into GitHub server infrastructure with an internal repository." 即远端与本地同源，且远端有额外 toolsets（copilot_spaces、github_support_docs_search）和工具（create_pull_request_with_copilot）
  - Remote 支持 OAuth（宿主需注册 GitHub App / OAuth App）与 PAT（Authorization: Bearer 头）两种认证
  - GHES 不支持 remote 托管；ghe.com 用 `https://copilot-api.<subdomain>.ghe.com/mcp`

## Remote Server URL 路径与请求头（配置维度）
- URL: https://raw.githubusercontent.com/github/github-mcp-server/main/docs/remote-server.md
- 抓取日期: 2026-09-20
- 可用于章节: §4 核心功能 / §6 进阶技巧
- 关键摘录:
  - URL 路径模式：`/`（默认 toolsets）、`/readonly`、`/insiders`、`/x/all`、`/x/{toolset}`、`/x/{toolset}/readonly`、`/x/{toolset}/readonly/insiders` 等；`{toolset}` 只能是单个，组合多个要用 `X-MCP-Toolsets` 头
  - 每个 toolset 有独立 URL，如 `https://api.githubcopilot.com/mcp/x/issues`、`.../x/repos/readonly`
  - 可选请求头（与本地 flag/env 等价）：
    - `X-MCP-Toolsets`（= `--toolsets` / `GITHUB_TOOLSETS`；空则默认；无效 toolset 静默忽略）
    - `X-MCP-Tools`（= `--tools` / `GITHUB_TOOLS`；无效工具名会报错）
    - `X-MCP-Readonly`（= `GITHUB_READ_ONLY`）
    - `X-MCP-Lockdown`（= `GITHUB_LOCKDOWN_MODE`；只能开不能关，服务端配置是上限）
    - `X-MCP-Insiders`（= `--insiders` / `GITHUB_INSIDERS`）
  - 示例：
    ```json
    {
      "type": "http",
      "url": "https://api.githubcopilot.com/mcp/",
      "headers": { "X-MCP-Toolsets": "repos,issues", "X-MCP-Readonly": "true", "X-MCP-Lockdown": "false" }
    }
    ```

## VS Code 配置（Remote + Local Docker）
- URL: https://raw.githubusercontent.com/github/github-mcp-server/main/README.md
- 抓取日期: 2026-09-20
- 可用于章节: §2 安装与配置 / §3 快速上手
- 关键摘录:
  - 一键安装徽章（vscode.dev/redirect/mcp/install）；要求 VS Code 1.101+（remote MCP + OAuth 支持），装完切到 Agent mode
  - Remote OAuth 配置：
    ```json
    { "servers": { "github": { "type": "http", "url": "https://api.githubcopilot.com/mcp/" } } }
    ```
  - Remote PAT 配置（含 inputs 密码提示）：
    ```json
    {
      "servers": {
        "github": {
          "type": "http",
          "url": "https://api.githubcopilot.com/mcp/",
          "headers": { "Authorization": "Bearer ${input:github_mcp_pat}" }
        }
      },
      "inputs": [
        { "type": "promptString", "id": "github_mcp_pat", "description": "GitHub Personal Access Token", "password": true }
      ]
    }
    ```
  - Local Docker + OAuth（官方镜像内置 app credentials，首次浏览器登录，token 仅存内存；Docker 需固定回调端口）：
    ```json
    {
      "mcp": {
        "servers": {
          "github": {
            "command": "docker",
            "args": ["run","-i","--rm","-p","127.0.0.1:8085:8085","-e","GITHUB_OAUTH_CALLBACK_PORT","ghcr.io/github/github-mcp-server"],
            "env": { "GITHUB_OAUTH_CALLBACK_PORT": "8085" }
          }
        }
      }
    }
    ```
  - Local Docker + PAT：`args: ["run","-i","--rm","-e","GITHUB_PERSONAL_ACCESS_TOKEN","ghcr.io/github/github-mcp-server"]`，env 里 `GITHUB_PERSONAL_ACCESS_TOKEN: "${input:github_token}"`
  - 也可放 `.vscode/mcp.json`（不带 `mcp` 键的格式）与团队共享

## Claude Code / Claude Desktop 配置
- URL: https://raw.githubusercontent.com/github/github-mcp-server/main/docs/installation-guides/install-claude.md
- 抓取日期: 2026-09-20
- 可用于章节: §2 安装与配置 / §5 常见问题与坑
- 关键摘录:
  - Claude Code Remote（2.1.1+ 用 add-json；Windows 上 add-json 可能报 Invalid input，改用 legacy 格式）：
    `claude mcp add-json github '{"type":"http","url":"https://api.githubcopilot.com/mcp","headers":{"Authorization":"Bearer YOUR_GITHUB_PAT"}}'`
  - legacy（≤2.1.0）：`claude mcp add --transport http github https://api.githubcopilot.com/mcp -H "Authorization: Bearer YOUR_GITHUB_PAT"`
  - `--scope`：local（默认，仅当前项目）/ project（写入 .mcp.json 共享）/ user（跨项目）
  - Claude Code Local Docker（OAuth）：`claude mcp add github -e GITHUB_OAUTH_CALLBACK_PORT=8085 -- docker run -i --rm -p 127.0.0.1:8085:8085 -e GITHUB_OAUTH_CALLBACK_PORT ghcr.io/github/github-mcp-server`
  - Claude Code Local 二进制：`claude mcp add-json github '{"command":"github-mcp-server","args":["stdio"],"env":{"GITHUB_PERSONAL_ACCESS_TOKEN":"YOUR_GITHUB_PAT"}}'`
  - 验证：`claude mcp list` / `claude mcp get github` / Claude Code 内 `/mcp`
  - Claude Desktop 配置文件路径：macOS `~/Library/Application Support/Claude/claude_desktop_config.json`；Windows `%APPDATA%\Claude\claude_desktop_config.json`；Linux `~/.config/Claude/claude_desktop_config.json`
  - Claude Desktop 重要限制：Remote server 的 OAuth 需注册 GitHub App，"which is not currently supported. Use the local Docker setup instead."（Settings → Connectors 自定义 connector 路径走不通）
  - Claude Desktop 已知问题："Some users have reported compatibility issues with Claude Desktop and Docker-based MCP servers."
  - 排错要点：PAT 需 repo scope；Docker pull 失败先 `docker logout ghcr.io`；日志在 `~/Library/Logs/Claude/mcp-server-*.log`（macOS）或 `%APPDATA%\Claude\logs\`（Windows）

## Cursor 配置
- URL: https://raw.githubusercontent.com/github/github-mcp-server/main/docs/installation-guides/install-cursor.md
- 抓取日期: 2026-09-20
- 可用于章节: §2 安装与配置 / §5 常见问题与坑
- 关键摘录:
  - Remote（推荐）：需 Cursor v0.48.0+（Streamable HTTP）；"While Cursor supports OAuth for some MCP servers, the GitHub server currently requires a Personal Access Token."
    ```json
    { "mcpServers": { "github": { "url": "https://api.githubcopilot.com/mcp/", "headers": { "Authorization": "Bearer YOUR_GITHUB_PAT" } } } }
    ```
  - Local Docker：同通用 mcpServers + docker run 格式
  - 配置文件：全局 `~/.cursor/mcp.json`；项目级 `.cursor/mcp.json`；用 `mcpServers` 键
  - 验证：Settings → Tools & Integrations → MCP Tools 看绿点；测试语 "List my GitHub repositories"

## 认证方式：OAuth vs PAT
- URL: README.md + docs/installation-guides/install-claude.md + docs.github.com 设置页
- 抓取日期: 2026-09-20
- 可用于章节: §2 安装与配置 / §5 常见问题与坑
- 关键摘录:
  - Local server 默认 OAuth：官方镜像/二进制内置 app credentials，首次使用弹浏览器登录，token 仅内存保存；Docker 需发布固定回调端口 127.0.0.1:8085；二进制本地流程无需固定端口；headless 有 device-code 回退（docs/oauth-login.md）
  - "You can still authenticate with a GitHub Personal Access Token by setting `GITHUB_PERSONAL_ACCESS_TOKEN` instead (it takes precedence over OAuth)."
  - PAT 权限建议（README Token Security Best Practices）：最小化授权 `repo`（仓库操作）、`read:packages`（Docker 镜像访问）、`read:org`（组织团队访问）；分项目用不同 PAT、定期轮换、绝不入库、配置文件 chmod 600
  - "The MCP server can use many of the GitHub APIs, so enable the permissions that you feel comfortable granting your AI tools."
  - 官方 docs 口径：OAuth 下 server 只能访问登录时批准的 scopes，且受组织 admin 策略限制；PAT 下访问范围 = PAT scopes，也受组织 PAT 限制约束；Enterprise Managed User 默认禁用 PAT
  - PAT 安全存放：环境变量或 .env（加 .gitignore），配置里引用 `$GITHUB_PAT`；"Environment variable support varies by host app and IDE. Some applications (like Windsurf) require hardcoded tokens in config files."
  - GHES/ghe.com 必须自带 OAuth App 或 GitHub App；非交互 stdio 部署见 docs/github-app-auth.md

## Toolsets 概念与配置
- URL: README.md#tool-configuration
- 抓取日期: 2026-09-20
- 可用于章节: §4 核心功能
- 关键摘录:
  - "The GitHub MCP Server supports enabling or disabling specific groups of functionalities via the `--toolsets` flag... Enabling only the toolsets that you need can help the LLM with tool choice and reduce the context size." Toolsets 还包含相关 MCP Resources 和 Prompts
  - 两种方式：`github-mcp-server --toolsets repos,issues,pull_requests,actions,code_security` 或 `GITHUB_TOOLSETS="..." ./github-mcp-server`；"The environment variable `GITHUB_TOOLSETS` takes precedence over the command line argument if both are provided."
  - 单个工具：`--tools get_file_contents,issue_read,create_pull_request` / `GITHUB_TOOLS`；与 toolsets 叠加（additive）
  - 特殊值：`all` 启用全部；`default` = context, repos, issues, pull_requests, users（不传时默认）；可 `GITHUB_TOOLSETS="default,stargazers"` 追加
  - 注意：read-only 优先于 --tools 显式请求；工具名必须精确匹配，无效名字启动即报错；改名后旧名保留为 alias（docs/tool-renaming.md）
  - Docker 传参：`docker run -i --rm -e GITHUB_PERSONAL_ACCESS_TOKEN=<t> -e GITHUB_TOOLSETS="repos,issues" ghcr.io/github/github-mcp-server`

## Toolsets 完整清单（本地）
- URL: README.md#available-toolsets
- 抓取日期: 2026-09-20
- 可用于章节: §4 核心功能
- 关键摘录:
  - 本地 21 个 toolset：context（强烈推荐，提供当前用户/上下文）、actions、code_quality、code_security、copilot、copilot_issue_intents、dependabot、discussions、gists、git、governance、issues、labels、notifications、orgs、projects、pull_requests、repos、secret_protection、security_advisories、stargazers、users
  - Remote 额外 toolset：copilot_spaces、github_support_docs_search（README 还把 copilot 列为 remote additional；以 remote-server.md 为准，remote additional = copilot_spaces + github_support_docs_search）
  - Remote 额外工具示例：`create_pull_request_with_copilot`（调用 Copilot coding agent 完成任务并开 PR）

## 工具形态：合并型多功能工具（method 参数）
- URL: README.md#tools
- 抓取日期: 2026-09-20
- 可用于章节: §4 核心功能
- 关键摘录:
  - 很多工具是"一个工具多个 method"形态：`issue_read`（get/get_comments/get_sub_issues/get_parent/get_labels）、`issue_write`（create/update）、`pull_request_read`（get/get_diff/get_status/get_files/get_commits/get_review_comments/get_reviews/get_comments/get_check_runs 共 9 种）、`actions_get`/`actions_list`/`actions_run_trigger`、`projects_get`/`projects_list`/`projects_write`、`label_write`、`sub_issue_write`、`discussion_comment_write` 等
  - 每个工具标注 OAuth Challenge Scopes（如 `repo`、`security_events`、`notifications`、`read:org`、`read:project`、`gist`）
  - 列表/搜索工具普遍支持 `fields` 参数裁剪返回字段以省 token（"omitting 'body' ... drops the largest per-result data"）

## Read-Only 模式与 Lockdown 模式
- URL: README.md#read-only-mode / #lockdown-mode
- 抓取日期: 2026-09-20
- 可用于章节: §4 核心功能 / §6 进阶技巧
- 关键摘录:
  - Read-only：`--read-only` 或 `GITHUB_READ_ONLY=1`，只提供只读工具，写工具即使显式 --tools 也被跳过
  - Lockdown：`--lockdown-mode` / `GITHUB_LOCKDOWN_MODE=1`；限制公开仓库中无 push 权限作者的内容浮出水面，降低 prompt injection 风险；"It is not an authorization boundary"；私库不受影响；`github-actions[bot]` 和 `copilot` 的内容始终视为安全
  - Lockdown 下报错工具：issue_read:get、pull_request_read:get/get_diff/get_files/get_commits；过滤内容的工具：issue_read:get_comments/get_sub_issues、pull_request_read:get_comments/get_review_comments/get_reviews
  - HTTP 模式下 X-MCP-Lockdown 头只能开启不能关闭服务端的 lockdown

## 服务器配置全景（Remote vs Local 对照表）
- URL: https://raw.githubusercontent.com/github/github-mcp-server/main/docs/server-configuration.md
- 抓取日期: 2026-09-20
- 可用于章节: §4 核心功能 / §6 进阶技巧
- 关键摘录:
  - 对照表：Toolsets（Remote: `X-MCP-Toolsets` 头或 `/x/{toolset}` URL；Local: `--toolsets` / `GITHUB_TOOLSETS`）；Tools（`X-MCP-Tools` / `--tools`、`GITHUB_TOOLS`）；Exclude Tools（`X-MCP-Exclude-Tools` / `--exclude-tools`、`GITHUB_EXCLUDE_TOOLS`）；Read-Only（`X-MCP-Readonly` 或 `/readonly` / `--read-only`、`GITHUB_READ_ONLY`）；Lockdown（`X-MCP-Lockdown` / `--lockdown-mode`、`GITHUB_LOCKDOWN_MODE`）；Insiders（`X-MCP-Insiders` 或 `/insiders` / `--insiders`、`GITHUB_INSIDERS`）；Feature Flags（`X-MCP-Features` 或 `?features=` / `--features`）；Scope Filtering 两端始终启用；Server Name/Title 仅本地（`GITHUB_MCP_SERVER_NAME`/`GITHUB_MCP_SERVER_TITLE`）
  - 优先级规则："read-only mode acts as a strict security filter that takes precedence over any other configuration"；"excluded tools takes precedence over toolsets and individual tools"
  - 食谱示例（均为 remote header vs local stdio 双格式）：只开指定工具、开多个 toolset、toolset+tools 叠加、exclude 指定写工具（如排除 create_pull_request/merge_pull_request 得到"只读+评审"PR 工具集）
  - Scope Filtering：classic PAT（ghp_ 前缀）按 token scope 在启动时过滤工具；OAuth 用 scope challenge 按需提示授权；其他 token 不过滤、由 API 层强制
  - Troubleshooting 表：启动失败→--tools 名字拼写；写工具不生效→read-only 开着；工具缺失→toolset 未启用

## i18n / 描述覆盖 / 服务器改名
- URL: README.md#i18n--overriding-descriptions
- 抓取日期: 2026-09-20
- 可用于章节: §6 进阶技巧
- 关键摘录:
  - 二进制同目录放 `github-mcp-server-config.json`，键如 `TOOL_ADD_ISSUE_COMMENT_DESCRIPTION` 覆盖工具描述；`--export-translations` 导出当前翻译文件
  - 环境变量等价：`GITHUB_MCP_TOOL_ADD_ISSUE_COMMENT_DESCRIPTION="..."`（`GITHUB_MCP_` 前缀 + 大写）
  - 可改初始化响应里的 name/title：`SERVER_NAME`/`SERVER_TITLE`（env `GITHUB_MCP_SERVER_NAME`/`GITHUB_MCP_SERVER_TITLE`），用于同时跑 github.com + GHES 两个实例时让 agent 区分

## GitHub Enterprise（ghe.com / GHES）
- URL: README.md#github-enterprise
- 抓取日期: 2026-09-20
- 可用于章节: §6 进阶技巧
- 关键摘录:
  - ghe.com remote：`"url": "https://copilot-api.octocorp.ghe.com/mcp"` + PAT 头；VS Code OAuth 还需配 VS Code 指向企业实例
  - "GitHub Enterprise Server does not support remote server hosting." 只能本地跑
  - 本地连 GHES/ghe.com：`--gh-host` flag 或 `GITHUB_HOST` env；GHES 必须 `https://` 前缀（强制 HTTPS，loopback 除外）；ghe.com 用 `https://YOURSUBDOMAIN.ghe.com`

## 官方文档页口径（docs.github.com）
- URL: https://docs.github.com/en/copilot/how-tos/provide-context/use-mcp-in-your-ide/set-up-the-github-mcp-server（另有 use-the-github-mcp-server、extend-copilot-chat-with-mcp 姊妹页）；https://docs.github.com/llms.txt 存在
- 抓取日期: 2026-09-20
- 可用于章节: §1 简介与定位 / §2 安装与配置
- 关键摘录:
  - "The GitHub MCP server is available to all GitHub users regardless of plan type. However, specific tools within the MCP server inherit the same access requirements as their corresponding GitHub features... tools that interact with Copilot Cloud Agent require a paid Copilot license."
  - 官方推荐路径：VS Code 扩展面板搜 `@mcp github` 从 MCP Registry 一键安装；"The remote GitHub MCP server is hosted by GitHub and is the recommended option for most users. The local ... is recommended for users who want to customize their setup or have specific security requirements."
  - 组织/企业 Copilot Business/Enterprise 需启用 "MCP servers in Copilot" policy
  - Visual Studio 需 17.14+；其 mcp.json PAT 写法用 `requestInit.headers`（与 VS Code 的 `headers` 不同）
  - docs.github.com 提供 llms.txt（200），内含 MCP 相关条目索引

## 安装方式汇总（Local）
- URL: README.md#installation + #build-from-source
- 抓取日期: 2026-09-20
- 可用于章节: §2 安装与配置 / §3 快速上手
- 关键摘录:
  - Docker 镜像：`ghcr.io/github/github-mcp-server`（公开；pull 报错多半 token 过期，`docker logout ghcr.io` 后重试）
  - 二进制：GitHub Releases 页下载（v1.12.2 有 Darwin_arm64 等多平台 tar.gz + checksums），配 `"command": "/path/to/github-mcp-server", "args": ["stdio"]`
  - 源码构建：`go build` cmd/github-mcp-server 目录，`github-mcp-server stdio` + GITHUB_PERSONAL_ACCESS_TOKEN；README 未提 `go install` 一键方式
  - Go API 作为库使用："should currently be considered unstable, and subject to breaking changes"

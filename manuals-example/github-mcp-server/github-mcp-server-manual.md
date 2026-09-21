# GitHub MCP Server 使用说明书

> GitHub 官方 MCP Server：让 AI 工具（Copilot、Claude、Cursor 等）通过 Model Context Protocol 直接读写 GitHub——仓库、issues、PR、Actions、安全告警，全部用自然语言操作。
> 版本基准：**v1.12.2**（2026-09-16 发布，周级发版）；本文素材抓取于 2026-09-20。

## 目录

- [[#1. 简介与定位]]
- [[#2. 安装与配置]]
- [[#3. 快速上手]]
- [[#4. 核心功能]]
- [[#5. 常见问题与坑]]
- [[#6. 进阶技巧]]
- [[#7. 资源链接]]

## 1. 简介与定位

GitHub MCP Server 把 GitHub 平台能力包装成 MCP 工具集，供任何 MCP 兼容宿主调用。官方定位五大场景：仓库管理、Issue/PR 自动化、CI/CD 与 Workflow 洞察、代码分析（安全告警/Dependabot）、团队协作（来源：[README](https://github.com/github/github-mcp-server)，2026-09-20）。

**两种形态**：

| | Remote（官方推荐） | Local |
|---|---|---|
| 端点/产物 | `https://api.githubcopilot.com/mcp/`（GitHub 托管） | Docker 镜像 / release 二进制（stdio） |
| 适合 | 大多数用户，零运维 | 定制配置、企业安全要求、GHES |
| 独有能力 | 额外 toolsets（copilot_spaces 等）、`/x/{toolset}` URL | 全部本地 flag/env 控制 |

两者同源（remote 就是本仓库绑定进 GitHub 基础设施）。⚠️ 防混淆：npm 旧包 `@modelcontextprotocol/server-github` 已于 2025-04 废弃（来源：[README](https://github.com/github/github-mcp-server)、[remote-server.md](https://github.com/github/github-mcp-server/blob/main/docs/remote-server.md)，2026-09-20）。

所有 GitHub 用户可用（不限付费档位），但涉及 Copilot Cloud Agent 的工具需 Copilot 付费许可（来源：[docs.github.com](https://docs.github.com/en/copilot/how-tos/provide-context/use-mcp-in-your-ide/set-up-the-github-mcp-server)，2026-09-20）。

项目成熟度：Go 实现、MIT 许可、约 3.3 万 stars，周级发版、迭代活跃（open issues 300+，排障知识主要集中在 Issues 区而非 Discussions）（来源：[Releases API](https://github.com/github/github-mcp-server/releases)、仓库快照，2026-09-20）。

## 2. 安装与配置

### 方式 A：Remote（最简）

**VS Code（1.101+）**——README 顶部一键安装徽章，或手动配置：

```json
{ "servers": { "github": { "type": "http", "url": "https://api.githubcopilot.com/mcp/" } } }
```

用 PAT 而非 OAuth 时加 headers（`inputs` 让 token 以密码框录入）：

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

**Claude Code（2.1.1+）**：

```bash
claude mcp add-json github '{"type":"http","url":"https://api.githubcopilot.com/mcp","headers":{"Authorization":"Bearer <你的PAT>"}}'
```

（Windows 上 add-json 若报 Invalid input，改用 legacy：`claude mcp add --transport http github https://api.githubcopilot.com/mcp -H "Authorization: Bearer <你的PAT>"`）

**Cursor（v0.48.0+，仅支持 PAT）**：写入 `~/.cursor/mcp.json` 或项目级 `.cursor/mcp.json`：

```json
{ "mcpServers": { "github": { "url": "https://api.githubcopilot.com/mcp/", "headers": { "Authorization": "Bearer <你的PAT>" } } } }
```

⚠️ Claude Desktop **不支持** remote（OAuth App 限制），只能走本地 Docker（来源：[install-claude.md](https://github.com/github/github-mcp-server/blob/main/docs/installation-guides/install-claude.md)、[install-cursor.md](https://github.com/github/github-mcp-server/blob/main/docs/installation-guides/install-cursor.md)，2026-09-20）。

配置也可放进项目级 `.vscode/mcp.json`（不带 `mcp` 外层键）与团队共享。Visual Studio 用户注意：需 17.14+，且 PAT 写法是 `requestInit.headers` 而非 VS Code 的 `headers`（来源：[docs.github.com](https://docs.github.com/en/copilot/how-tos/provide-context/use-mcp-in-your-ide/set-up-the-github-mcp-server)，2026-09-20）。

### 方式 B：Local（Docker / 二进制）

```json
{
  "mcp": {
    "servers": {
      "github": {
        "command": "docker",
        "args": ["run", "-i", "--rm", "-e", "GITHUB_PERSONAL_ACCESS_TOKEN", "ghcr.io/github/github-mcp-server"],
        "env": { "GITHUB_PERSONAL_ACCESS_TOKEN": "${input:github_token}" }
      }
    }
  }
}
```

二进制：从 [Releases](https://github.com/github/github-mcp-server/releases) 下载对应平台 tar.gz，配置 `"command": "/path/to/github-mcp-server", "args": ["stdio"]`。也有 Homebrew formula。Docker 镜像公开，但 pull 报错多半是本地缓存了过期凭证：先 `docker logout ghcr.io` 再重试（来源：[README#installation](https://github.com/github/github-mcp-server#installation)，2026-09-20）。

### 认证：OAuth vs PAT

- Local 默认 OAuth：镜像内置 app credentials，首次使用弹浏览器登录，token 只存内存（Docker 需发布回调端口 `127.0.0.1:8085`）。
- `GITHUB_PERSONAL_ACCESS_TOKEN` 优先于 OAuth。PAT 最小权限建议：`repo` + `read:packages` + `read:org`；分项目用不同 PAT、定期轮换、绝不入库。
- 组织开 SSO 时，PAT 生成后必须点 **Configure SSO** 授权组织；fine-grained PAT 的 **Resource Owner 必须选目标组织**——否则组织私有 repo 不可见（全仓最热坑之一，35 评论，来源：[issue #153](https://github.com/github/github-mcp-server/issues/153)，2026-09-20）。

## 3. 快速上手

**① 配置**（以 VS Code remote 为例，见 §2 方式 A 第一个 JSON 块）。

**② 验证连接**：装完切到 Agent mode；Claude Code 用 `claude mcp list` / `claude mcp get github` 或会话内 `/mcp`；Cursor 看 Settings → Tools & Integrations → MCP Tools 绿点。

**③ 首次调用**：对 agent 说一句

```
List my GitHub repositories
```

能列出仓库即链路通了。然后试一句真实的：「给 `<owner>/<repo>` 的 issue #1 加条评论」观察写操作权限（来源：[install-cursor.md](https://github.com/github/github-mcp-server/blob/main/docs/installation-guides/install-cursor.md)、[install-claude.md](https://github.com/github/github-mcp-server/blob/main/docs/installation-guides/install-claude.md)，2026-09-20）。

## 4. 核心功能

### Toolsets（工具集）——按域开关

默认启用 `context, repos, issues, pull_requests, users`（**不传配置时的默认值**）。本地共 21 个 toolset：

| 类别 | toolsets |
|---|---|
| 代码与仓库 | repos、git、code_quality、code_security、secret_protection、security_advisories、dependabot、gists、stargazers |
| 协作 | issues、pull_requests、discussions、labels、projects、notifications、orgs、users、context |
| 平台 | actions、copilot、copilot_issue_intents、governance |

配置：`--toolsets repos,issues` 或环境变量 `GITHUB_TOOLSETS`（env 优先于 flag）；特殊值 `all` / `default`；`--tools get_file_contents` 可叠加单个工具。Remote 端用 `X-MCP-Toolsets` 请求头或 URL 路径 `https://api.githubcopilot.com/mcp/x/issues`（来源：[README#tool-configuration](https://github.com/github/github-mcp-server#tool-configuration)，2026-09-20）。

### 工具形态：合并型多功能工具

很多工具是「一个工具多个 method」：`issue_read`（get/get_comments/get_labels…）、`issue_write`（create/update）、`pull_request_read`（get/get_diff/get_files/get_reviews 等 9 种）、`actions_list`/`actions_run_trigger` 等。列表工具普遍支持 `fields` 参数裁剪返回字段——**省 token 的关键开关**（来源：[README#tools](https://github.com/github/github-mcp-server#tools)，2026-09-20）。

### 安全控制：Read-Only 与 Lockdown

- `--read-only` / `GITHUB_READ_ONLY=1` / remote 用 `/readonly` URL：只给读工具，**优先级高于一切**（显式 --tools 要来的写工具也会被过滤）。
- `--lockdown-mode`：公开仓库中无 push 权限作者的内容（issue/PR 文本）被限制浮出，**降低 prompt injection 风险**；不是授权边界，私库不受影响（来源：[README#lockdown-mode](https://github.com/github/github-mcp-server#lockdown-mode)，2026-09-20）。

### Scope Filtering（按 token 权限过滤工具）

classic PAT（`ghp_` 前缀）会在启动时按 token scope 过滤掉无权限的工具；OAuth 则走 per-call scope challenge——每次调用只申请该工具所需的权限（v1.11 起）；其他类型 token 不过滤，由 API 层强制。所以「工具列表里看不到某工具」先查 token scope（来源：[server-configuration.md](https://github.com/github/github-mcp-server/blob/main/docs/server-configuration.md)，2026-09-20）。

## 5. 常见问题与坑

**① 安全头号坑：恶意 issue 文本可借 MCP 外泄私仓数据** —— 2025-05 Invariant 披露「toxic agent flow」（HN 508 分）：攻击者在公开仓库 issue 里埋注入指令，agent 读 issue 后被操纵。社区定性「不是 exploit，是 MCP 按设计工作」。**对策：处理不可信仓库时开 lockdown 模式 + read-only，最小化 PAT scope**（来源：[HN](https://news.ycombinator.com/item?id=44097390)，2026-09-20）。

**② 401 Bad credentials 但 token 明明没错** —— 多半是**宿主 env 插值失败**：`{env:VAR}` / `${env:VAR}` / `$VAR` 各宿主语法不同，写错不报错，容器实际拿到字面量字符串。排查第一步：确认进程内实际拿到的值；再按序查：token 未传入 → 组织未授权（SSO/Resource Owner）→ scope 不足（来源：[issue #1396](https://github.com/github/github-mcp-server/issues/1396)、[#2418](https://github.com/github/github-mcp-server/issues/2418)、[#201](https://github.com/github/github-mcp-server/issues/201)，2026-09-20）。

**③ 组织私有 repo 不可见** —— 见 §2 认证节的 SSO/Resource Owner 两点（来源：[issue #153](https://github.com/github/github-mcp-server/issues/153)，2026-09-20）。

**④「已修复」的 bug 还在** —— Docker `:latest` 镜像不会自动更新：`docker pull ghcr.io/github/github-mcp-server` 手动刷新（来源：[issue #662](https://github.com/github/github-mcp-server/issues/662)，2026-09-20）。

**⑤ list_* 工具撑爆上下文触发 LLM 429** —— 默认返回全量字段（5–6KB/commit；有用户 14 个 PR 烧掉 142k token）。对策：收紧 perPage、用 `fields` 裁剪、升级到含 token 优化的版本（来源：[issue #142](https://github.com/github/github-mcp-server/issues/142)，2026-09-20）。

**⑥ 客户端行为坑（近 30 天社区热点）**：Copilot 每个新请求都全量启动/检查 MCP server 拖慢响应——VS Code 设 `chat.mcp.autostart: never`（官方团队成员给的技巧）；企业管理设置下发的 MCP 不受用户 allowlist 约束，「别只信 UI」；远端端点 HTTP 200 但握手失败时客户端静默降级，传统 uptime 监控无效；配置写错文件位置也会静默不加载且无报错（社区为此专门写了检测工具）（来源：[r/GithubCopilot](https://www.reddit.com/r/GithubCopilot/comments/1w2tday/)、[r/ClaudeAI](https://www.reddit.com/r/ClaudeAI/comments/1w97q4i/)、[r/cursor](https://www.reddit.com/r/cursor/comments/1w7mgji/)、[r/ClaudeAI 配置检测](https://www.reddit.com/r/ClaudeAI/comments/1w4ejeb/)，2026-09-20）。

**⑦ 企业代理 TLS 拦截导致 Docker 连不上** —— 容器内不信任自签 CA。改用 release 二进制/Homebrew 走系统证书链（来源：[issue #157](https://github.com/github/github-mcp-server/issues/157)，2026-09-20）。

**⑧ Claude Web/Code 接 remote 报 "error connecting"** —— 历史 OAuth/CORS 兼容问题，v1.11–v1.12 已专项修复；先升级再重连（来源：[issue #549](https://github.com/github/github-mcp-server/issues/549)，2026-09-20）。

**⑨ GitHub API rate limit 报错看不懂** —— 旧版报错文案对 agent 不友好；v1.11+ 已改进报错信息并支持 ETag 条件请求（条件请求不占 rate limit 计数）。遇到先升级，再减少 list 类调用频率（来源：[issue #2385](https://github.com/github/github-mcp-server/issues/2385)、releases，2026-09-20）。

## 6. 进阶技巧

**上下文成本控制**（社区量化动机：禁用 GitHub MCP 后 Claude Code 续航 +10%）：按需开 toolset 而非 `all`；善用 `fields` 参数与 perPage；VS Code `chat.mcp.autostart: never` + tool search（来源：[HN](https://news.ycombinator.com/item?id=46776551)、[r/GithubCopilot](https://www.reddit.com/r/GithubCopilot/comments/1w2tday/)，2026-09-20）。

**「只读+评审」PR 工具集食谱**：`--exclude-tools create_pull_request,merge_pull_request` 排除写操作，exclude 优先级高于 toolsets/tools。

**Remote 请求头玩法**：`X-MCP-Readonly: true`、`X-MCP-Insiders: true`（尝鲜工具）、URL `?features=` 开 feature flags；`/x/repos/readonly` 组合路径。

**GHES / ghe.com**：GHES 不支持 remote，本地跑用 `--gh-host` / `GITHUB_HOST`（必须 https:// 前缀）；ghe.com remote 端点为 `https://copilot-api.<subdomain>.ghe.com/mcp`。同时跑 github.com + 企业两个实例时，用 `GITHUB_MCP_SERVER_NAME` 改名让 agent 区分。

**认证趋势（值得关注）**：官方正在去 PAT 化——全仓最热 issue #132（49 评论）催生的 OAuth device flow 已合并，v1.11–v1.12 持续强化 per-call scope 检查；企业禁 PAT 的场景跟紧新版（来源：[issue #132](https://github.com/github/github-mcp-server/issues/132)、releases，2026-09-20）。

**替代路径评估**：社区实测派认为轻量场景 `gh` CLI 已够（有极简 agent 直接移除 GitHub MCP）；MCP 的真实主场是「无 shell 环境」（浏览器内 agent）。选型先看宿主与场景（来源：[HN benchmark](https://news.ycombinator.com/item?id=47495475)、[Ask HN](https://news.ycombinator.com/item?id=49562869)，2026-09-20）。

**工具描述汉化/定制**：二进制同目录放 `github-mcp-server-config.json`，用 `TOOL_<工具名>_DESCRIPTION` 键覆盖任意工具描述（也可用 `GITHUB_MCP_` 前缀环境变量）；`--export-translations` 可导出当前翻译文件作底稿（来源：[README#i18n](https://github.com/github/github-mcp-server#i18n--overriding-descriptions)，2026-09-20）。

## 7. 资源链接

**官方**

- 仓库与 README：https://github.com/github/github-mcp-server
- 配置文档目录（remote-server / server-configuration / 各客户端 install 指南）：https://github.com/github/github-mcp-server/tree/main/docs
- Releases（二进制/Docker/Homebrew）：https://github.com/github/github-mcp-server/releases
- docs.github.com 官方设置指南：https://docs.github.com/en/copilot/how-tos/provide-context/use-mcp-in-your-ide/set-up-the-github-mcp-server

**本文引用来源**（均抓取于 2026-09-20）：GitHub issues [#132](https://github.com/github/github-mcp-server/issues/132) / [#142](https://github.com/github/github-mcp-server/issues/142) / [#153](https://github.com/github/github-mcp-server/issues/153) / [#157](https://github.com/github/github-mcp-server/issues/157) / [#201](https://github.com/github/github-mcp-server/issues/201) / [#549](https://github.com/github/github-mcp-server/issues/549) / [#662](https://github.com/github/github-mcp-server/issues/662) / [#1396](https://github.com/github/github-mcp-server/issues/1396) / [#2385](https://github.com/github/github-mcp-server/issues/2385) / [#2418](https://github.com/github/github-mcp-server/issues/2418)；HN [#44097390](https://news.ycombinator.com/item?id=44097390) / [#46776551](https://news.ycombinator.com/item?id=46776551) / [#47495475](https://news.ycombinator.com/item?id=47495475) / [#49562869](https://news.ycombinator.com/item?id=49562869)；Reddit [r/GithubCopilot](https://www.reddit.com/r/GithubCopilot/comments/1w2tday/) / [r/ClaudeAI](https://www.reddit.com/r/ClaudeAI/comments/1w97q4i/) / [r/cursor](https://www.reddit.com/r/cursor/comments/1w7mgji/)。

> 调研素材留档于同目录 `research/`（official-docs.md / github.md / community.md）。

# GitHub MCP Server — GitHub 事实层素材（issues / discussions / releases）

> 调研员：事实层 agent（issues/discussions/releases 专线）。抓取日期 2026-09-20。
> 仓库：https://github.com/github/github-mcp-server（public，约 33k stars，has_discussions=true）

---

## 当前版本与发布节奏（Releases API 确认）
- URL: https://github.com/github/github-mcp-server/releases
- 抓取日期: 2026-09-20
- 可用于章节: §1 简介与定位；§7 资源链接
- 关键摘录:
  - 最新版 **v1.12.2**（2026-09-16 发布）；发布节奏约每周一次 minor/patch。
  - v1.12.2: "add update_issue_comment tool"、"add remove_issue_reaction, remove_issue_comment_reaction and remove_pull_request_review_comment_reaction tools to the granular issues and pull requests toolsets"
  - v1.12.1（2026-09-08）bugfix: "Oauth protected resource metadata became too permissive in the supported scopes advertised"；"fix(oauth): advertise only default scopes in protected resource metadata"（PR #3251）
  - v1.12.0（2026-09-03）Highlights: "New governance tools for agents. Read and manage repository rulesets and custom properties across repository, organization, and enterprise levels." / "Safer write operations. Pin merge HEADs (expectedHeadSha on merge_pull_request), recover file SHAs (create_or_update_file), use least-privilege public-repository access (allow public_repo for public contribution tools)" / "Better content fidelity. Markdown bodies... preserve visible content while filtering unsafe invisible characters" / "Enable feature flags via URL query parameter (?features=) for headerless hosted connections"
  - v1.11.0（2026-08-25）Highlights: "Smarter OAuth challenges: per-call scope checks request only the permissions each tool invocation needs" / "CORS now works across OAuth discovery routes, with configurable authorization-server URLs (--authorization-server flag)" / "create parent and sub-issues atomically" / "REST responses support ETag conditional requests"（缓解 rate limit）
  - v1.10.1（2026-08-20）: "Fix add_issue_comment schema compatibility regression"
  - 安装产物：每个 release 提供多平台二进制 tar.gz（Darwin/Linux/Windows arm64+x64）+ checksums；也有 Docker 镜像 ghcr.io/github/github-mcp-server 与 Homebrew formula。

## Issue #132 — 社区强烈要求 PAT 之外的认证方式（已演变为 OAuth device flow）
- URL: https://github.com/github/github-mcp-server/issues/132
- 抓取日期: 2026-09-20
- 可用于章节: §2 安装与配置；§5 常见问题与坑；§6 进阶技巧
- 关键摘录:
  - 49 条评论，全仓评论数最高的 issue。原帖："PATs are long lived credentials that are discouraged, and sometimes entirely restricted in many organizations. This limits the organizations from taking advantage of this MCP server."（企业禁用 PAT → MCP 不可用）
  - 诉求："Please provide an alternative Auth method, e.g an OAuth with a `device_code`, in addition to the PAT"
  - 后续演进：PR #1649 "feat: implement OAuth device flow authentication" 已合并；v1.12 系列持续完善 OAuth（per-call scope checks、protected resource metadata、device flow）。说明：新版本 stdio 模式已不只依赖 PAT，但 PAT 仍是文档默认路径。

## Issue #153 — 访问组织私有仓库失败（PAT 配置根因）
- URL: https://github.com/github/github-mcp-server/issues/153
- 抓取日期: 2026-09-20
- 可用于章节: §2 安装与配置；§5 常见问题与坑
- 关键摘录:
  - 35 条评论。现象："GitHub MCP Server lacks the ability to access private repositories that belong to organizations"（列不出/读不到公司组织私有 repo）。
  - 解法（社区高赞 tscodeler）："I successfully made it work by using a fine-grained personal access token with 'Resource Owner' pointing to my company's organization. Tokens (classic) should work as well if allowed by your Organization." → fine-grained PAT 建 token 时 Resource Owner 必须选目标组织。
  - 解法（官方 SamMorrowDrums）："Does your organization use SSO? Sometimes after generating a PAT you also need to give it specific access to restricted orgs." → 组织开 SSO 时，PAT 生成后还要点 "Configure SSO" 授权该组织。

## Issue #549 — Claude Web 集成 remote server 连接失败
- URL: https://github.com/github/github-mcp-server/issues/549
- 抓取日期: 2026-09-20
- 可用于章节: §2 安装与配置；§5 常见问题与坑
- 关键摘录:
  - 40 条评论。现象：在 claude.ai/settings/integrations 添加 `https://api.githubcopilot.com/mcp/` 后点 Connect 报 "There was an error connecting to GitHub server. Please check your server URL and make sure your server handles auth correctly."，且 Claude 会把带尾斜杠的 URL 改写成无尾斜杠。
  - 大量 "Same here" 跟进（含 Claude Code 用户）。属 remote server（api.githubcopilot.com/mcp）与 Claude 侧 OAuth 握手的兼容性历史问题；v1.11/v1.12 的 CORS/OAuth discovery 修复（PR #3147、#3251）正是针对此类问题。教训：remote 模式排障先确认客户端 OAuth 支持与服务端版本。

## Issue #1396 — OpenCode 连 remote server 无工具（env 插值语法错误）
- URL: https://github.com/github/github-mcp-server/issues/1396
- 抓取日期: 2026-09-20
- 可用于章节: §2 安装与配置；§5 常见问题与坑
- 关键摘录:
  - 26 条评论。现象：OpenCode 配置 remote github server 后 "the AI has no access to any GitHub-related tools"，连接看似建立但工具不加载。
  - 根因（alexaandru）："it works with `{env:GITHUB_MCP_PATH}` not `${env:GITHUB_MCP_PATH}`" → 宿主的 env 变量插值语法写错（`{env:VAR}` vs `${env:VAR}`），Authorization header 实际为空。
  - 教训：MCP 宿主配置文件里 env 引用语法因宿主而异，写错不会报错只会静默认证失败。

## Issue #662 — GHES 自定义域名被误判为 github.com（后缀匹配 bug）
- URL: https://github.com/github/github-mcp-server/issues/662
- 抓取日期: 2026-09-20
- 可用于章节: §5 常见问题与坑；§6 进阶技巧（GHES）
- 关键摘录:
  - 现象：设 `GITHUB_HOST=abcd-github.com`（GHES），工具却请求 `github.com/api/v3`。根因："any hostname ending with 'github.com' (such as 'slack-github.com') is incorrectly identified as the public GitHub host"（parseApiHost 用后缀匹配）。
  - 解法（marcellodesales）：修复已发版，但旧 Docker 镜像不会自动更新："If your MCP client doesn't pull images before running, then this will result in the problem" → `docker pull ghcr.io/github/github-mcp-server` 手动刷新。教训：Docker 镜像 :latest 不自动更新是大量"已修复仍复现"问题的根源。

## Issue #201 — 401 Bad credentials（token 正确但仍 401）
- URL: https://github.com/github/github-mcp-server/issues/201
- 抓取日期: 2026-09-20
- 可用于章节: §5 常见问题与坑
- 关键摘录:
  - 现象：curl 带同一 token 调 API 正常，MCP 工具却报 `MCP error -32603: failed to list pull requests: GET https://api.github.com/... 401`。
  - 官方排障思路（juruen）："Start with a public repo first; Try with your private/org repo, and be explicit about the repo name." → 先公开 repo 验证链路，再查组织权限/SSO/token scope。401 常见根因排序：token 未传入（env 插值失败，见 #2418/#1396）> token 无组织授权（见 #153）> scope 不足。

## Issue #2418 — 宿主 env 块不展开 shell 变量，Docker 收到字面量 "$VAR" → 401
- URL: https://github.com/github/github-mcp-server/issues/2418
- 抓取日期: 2026-09-20
- 可用于章节: §2 安装与配置；§5 常见问题与坑
- 关键摘录:
  - 现象（Antigravity 宿主）：配置 `"env": {"GITHUB_PERSONAL_ACCESS_TOKEN": "$GITHUB_PERSONAL_ACCESS_TOKEN"}` 后报 "non-200 OK status code: 401 Unauthorized / Bad credentials"。根因：宿主不做 shell 展开，"Docker receives literal '$GITHUB_PERSONAL_ACCESS_TOKEN' string"。
  - Workaround：用 `sh -c` 包装让 shell 先展开，或直接把 token 值写进 env（注意安全）。
  - 官方定性（SamMorrowDrums）："The server only receives the literal token value it is given" —— 属宿主 env 处理行为，非 server bug。通用教训：MCP 配置里写 `"$VAR"` 是否被展开完全取决于宿主，排查 401 第一步先确认容器内实际拿到的值。

## Issue #142 — list_commits 等工具响应过大，撑爆 LLM 上下文/触发 429
- URL: https://github.com/github/github-mcp-server/issues/142
- 抓取日期: 2026-09-20
- 可用于章节: §5 常见问题与坑；§6 进阶技巧（上下文优化）
- 关键摘录:
  - 24 条评论。现象："`list_commits()` returns 30 results... >64k tokens"，直接触发 LLM 侧 "429 Request too large... rate_limit_exceeded"。
  - 根因：返回全量字段（author/committer/verification 含签名等），"5-6KB per commit"。
  - 用户实测（cufeo）："`list_pull_requests` with 14 open pull requests, the context window jumped to 142k token, burned 0.6$ with claude sonnet. It would be nice to have control over the returned fields per tool."
  - 演进：官方后续做了 "Optimize token usage in default list_ tools"（PR #2016）。用法建议：调 list 类工具时收紧 perPage、指定 sha 范围、避免让模型一次性拉全量列表。

## Issue #2385/#2386 — rate limit 报错信息对 agent 不友好（已改进）
- URL: https://github.com/github/github-mcp-server/issues/2385
- 抓取日期: 2026-09-20
- 可用于章节: §5 常见问题与坑
- 关键摘录:
  - 官方专项 "Improve rate limit error messages for AI agents"（PR #2386 已合并）：命中 GitHub API rate limit 时，新版会返回更适合 agent 理解的错误信息。
  - 配套缓解：v1.11.0 "REST responses support ETag conditional requests"（条件请求不占用 rate limit 计数）。
  - 手册提示：遇到 rate limit 报错先升级版本（错误信息已改进），再利用 ETag 缓存与减少 list 调用频率。

## Issue #157 — Docker 容器内企业自签 CA / TLS 拦截导致连接失败
- URL: https://github.com/github/github-mcp-server/issues/157
- 抓取日期: 2026-09-20
- 可用于章节: §5 常见问题与坑；§6 进阶技巧（Docker/企业网络）
- 关键摘录:
  - 17 条评论。现象："Our enterprise controls are blocking the connections"（企业代理 TLS 拦截，容器内不信任自签 CA），Docker 镜像无内置自定义 CA 机制。
  - 官方解法（SamMorrowDrums）："One option is just using the built binaries shared in the releases, as then they will use system and not docker trust chain." → 改用 release 二进制（走系统证书链）或 Homebrew 安装，绕过容器证书问题。
  - 用户验证（byjrack）给出 VS Code 配置： `"github": { "command": "github-mcp-server", "args": ["stdio"], "env": {"GITHUB_PERSONAL_ACCESS_TOKEN": "${input:github_token}"} }`，并确认 Homebrew formula 已上架。

## PR #1836 — stdio 模式 OAuth 2.1 + MCP URL elicitation
- URL: https://github.com/github/github-mcp-server/pull/1836
- 抓取日期: 2026-09-20
- 可用于章节: §4 核心功能；§6 进阶技巧
- 关键摘录:
  - 30 条评论的标志性 PR："Add OAuth 2.1 authentication for stdio mode with MCP URL elicitation and performance optimizations"。说明 stdio 本地模式也在摆脱"只有 PAT"的形态，OAuth 化是官方明确方向（配合 #132 的社区诉求）。

## Discussions 区概况（Registry  onboarding 为主，手册价值低）
- URL: https://github.com/github/github-mcp-server/discussions
- 抓取日期: 2026-09-20
- 可用于章节: §7 资源链接
- 关键摘录:
  - 近期 discussion 几乎全是第三方 MCP server 申请上架 GitHub MCP Registry 的 onboarding request（#3291/#3300/#3303/#3308/#3309 等），对手册无直接价值。
  - 少数有价值的讨论：#3306 "What can an agent actually do with github-mcp-server, and who decides?"（能力边界讨论）；#3298 "Feedback on PR file field selection and per-toolset read-only configuration"（per-toolset 只读配置的需求反馈，说明只读粒度仍在演进）。
  - 结论：该仓库的排障知识集中在 Issues 而非 Discussions。

## 经典坑速查表（调研结论汇总）
- URL: 见上各条
- 抓取日期: 2026-09-20
- 可用于章节: §5 常见问题与坑
- 关键摘录:
  1. **401 Bad credentials 但 token 没错** → 多半是宿主 env 插值失败（`{env:VAR}` vs `${env:VAR}` vs `$VAR`，#1396/#2418），容器实际拿到字面量字符串。
  2. **组织私有 repo 不可见** → fine-grained PAT 的 Resource Owner 没选组织，或组织开 SSO 后未对 PAT 做 Configure SSO 授权（#153）。
  3. **"已修复"的 bug 仍复现** → Docker :latest 镜像不自动更新，先 `docker pull ghcr.io/github/github-mcp-server`（#662）。
  4. **GHES 自定义域名以 github.com 结尾被误判** → 老版本后缀匹配 bug，升级解决（#662）。
  5. **list_* 工具撑爆上下文/触发 LLM 429** → 默认返回全量字段，5-6KB/commit；收紧 perPage、升级版本（#142/#2016）。
  6. **GitHub API rate limit 报错看不懂** → v1.11+ 改进了报错文案并支持 ETag 条件请求（#2385/#3026）。
  7. **企业代理 TLS 拦截导致 Docker 模式连不上** → 改用 release 二进制/Homebrew，走系统证书链（#157）。
  8. **Claude Web/Code 接 remote server 报 "error connecting"** → 历史 OAuth/CORS 兼容问题，v1.11–v1.12 已专项修复，升级并重连（#549）。
  9. **企业禁 PAT** → 关注 OAuth device flow（PR #1649）与 stdio OAuth 2.1（PR #1836），官方正在去 PAT 化（#132）。

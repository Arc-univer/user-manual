# user-manual

为开发者向工具（CLI、开源库、框架、MCP server）自动生成**中文使用说明书**的 Claude Code skill。

## 它做什么

给定一个工具名、官网 URL 或 GitHub 地址，自动走完五步流程产出一篇可扫读、带命令示例、面向初次接触者的说明书：

```
目标识别 → 社区引擎探测 → 并行调研（3 个 subagent）→ 🔴 大纲检查点（唯一）→ 成稿 → 落盘
```

- **固定七章骨架**：简介与定位 → 安装与配置 → 快速上手 → 核心功能 → 常见问题与坑 → 进阶技巧 → 资源链接，按目标类型微调（CLI / 库 / MCP server）。
- **两层信息源**：事实层（官方文档 + GitHub issues/releases）保证准确，补充层（last30days 引擎抓近 30 天 Reddit/HN）提供时效坑位；冲突时以事实层为准并标注。
- **可审计**：每个命令/flag 都出自调研素材，关键论断附来源链接与抓取日期；raw 素材留档 `research/`，不污染对话。
- **优雅降级**：所有外部依赖（Context7 / Jina / last30days）均可选；唯独 last30days 缺席时会先询问用户，不静默降级。

## 实测产物

`manuals-example/` 下是三份端到端实测生成的说明书（附完整调研素材）：

| 目标 | 类型 | 说明 |
|---|---|---|
| [ripgrep](manuals-example/ripgrep/ripgrep-manual.md) | 知名 CLI | 多平台安装 + flag 速查表 |
| [vcrpy](manuals-example/vcrpy/vcrpy-manual.md) | 小众 Python 库 | 文档稀薄时 README/issues 顶上，近 30 天无社区讨论如实标注 |
| [github-mcp-server](manuals-example/github-mcp-server/github-mcp-server-manual.md) | MCP server | 客户端配置 JSON 完整块 + tools 清单表 + 9 条经典坑 |

## 目录结构

```
├── skills/            # skill 本体（SKILL.md + references/ + agents/ + test-prompts.json）
├── manuals-example/   # 三份实测生成的说明书产物与调研素材
├── docs/              # PRD、ADR、问题档案
├── CLAUDE.md          # 项目操作指引（Claude Code 用）
└── CONTEXT.md         # 领域术语表
```

> 开发时使用的 `.agents/skills/`、`.claude/skills/` 已 gitignore；仓库中的 `skills/` 是与生效副本（`.agents/skills/user-manual/`）同步的发布副本。

## 使用

1. 把 `skills/` 目录复制到项目的 `.claude/skills/`（或用户级 `~/.claude/skills/`）。
2. 对话中说「给 ripgrep 写一份使用说明书」或贴一个 GitHub 地址即可触发。
3. 可选：安装 [last30days](https://github.com/mvanhorn/last30days-skill) 获得近 30 天社区讨论层；未安装时 skill 会询问你「帮我安装 / 本次降级 WebSearch」。
4. 产物默认写到当前目录 `manuals/<目标名>/<目标名>-manual.md`，可用环境变量 `USER_MANUAL_OUTPUT_DIR` 改默认落点（详见 SKILL.md「配置」节）。

## 文档

- 需求与设计：[docs/PRD.md](docs/PRD.md)、[CONTEXT.md](CONTEXT.md)、[docs/adr/](docs/adr/)
- 问题档案（缺陷与修复记录）：[docs/problems/](docs/problems/)

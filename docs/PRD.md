# PRD：user-manual skill

> 本文档是「给开发者向工具自动生成中文使用说明书」skill 的产品需求文档。
> 所有关键决策已在 `/grill-with-docs` 会话（13 问）中与用户对齐，词汇定义见 `CONTEXT.md`，依赖决策见 `docs/adr/0001`。
> **下个会话的目标：按本文档动手实现该 skill。**

## 1. 定位与产出

- **目标（Target）**：开发者向工具——CLI、开源库、框架、MCP server 等。终端用户应用是后续泛化方向，不在 v1。
- **说明书（Manual）**：中文，中等篇幅，可扫读、带命令示例，面向**第一次接触该目标的开发者**。
- **固定骨架**（按类型微调）：

```
简介与定位 → 安装与配置 → 快速上手 → 核心功能 → 常见问题与坑 → 进阶技巧 → 资源链接
```

## 2. 核心流程（含唯一检查点）

```
目标识别 → 并行调研 → 检查点（大纲） → 成稿 → 落盘
```

1. **目标识别**：接受工具名 / 官网 URL / GitHub repo 地址三种输入。名字唯一则直接开跑；有歧义当场反问澄清一次（输入澄清，不算破坏单一检查点）。
2. **并行调研**：3 个 subagent 各管一层，各自设搜索次数上限，raw 素材不进主窗口，落盘到 `research/`。
3. **检查点**：输出大纲（各章节 + 每章素材来源清单），用户确认后才成稿。
4. **成稿与落盘**：写 `manuals/<目标名>/manual.md`，素材留 `manuals/<目标名>/research/`。

产物结构：

```
manuals/
└── <目标名>/
    ├── manual.md      # 说明书成稿
    └── research/      # 调研素材落盘（检查点时可抽查来源）
```

## 3. 调研信息源（事实层 vs 补充层）

| 层 | 角色 | 来源 |
|---|---|---|
| 事实层 | 说明书论断的**准绳** | 官方文档（`llms.txt` 探测 → Jina Reader 兜底）、GitHub（README/issues/discussions，不限时）、开源库走 Context7 MCP（可选） |
| 补充层 | 提供坑、技巧、真实体验 | last30days 引擎直调（近 30 天 Reddit/HN）+ GitHub issues（经典坑兜底，不受 30 天限制） |

**冲突裁决**：以事实层为准，并在说明书中标注社区异议。
**来源标注**：关键论断附来源链接 + 抓取日期。
**时间窗口**：社区讨论以近 30 天为主（反映当前版本现状）；经典坑由 GitHub issues 兜底。

## 4. 依赖策略（ADR-0001：优雅降级）

所有外部积木**可选**，运行时探测，有则用其最优能力，无则降级到 Claude Code 内置工具链（WebSearch / WebFetch / Bash），**绝不因缺依赖中止**。零必付 API key，零必配 MCP，裸环境可跑。

| 积木 | 作用 | 必需性 |
|---|---|---|
| `llms.txt` 探测 | 官网文档直取 | 零成本首选，内置 WebFetch 即可 |
| Jina Reader | 无 `llms.txt` 时抓单页 | 免费档 20 req/min，兜底 |
| Context7 MCP | 开源库版本对应官方文档 | **可选**，有则用、无则跳过 |
| last30days 引擎 | 社区口碑 / 常见坑素材 | **可选**，检测到则 Bash 直调，无则降级为内置 WebSearch 搜 Reddit/HN |
| Firecrawl | 整站爬取 | v1 **排除**（付费） |

## 5. 技术形态

- skill 落点：`.agents/skills/user-manual/`（带 `agents/openai.yaml`，Codex 兼容）
- 命名：`user-manual`
- 开发期在本 repo 内验证
## 6. 验证

- `test-prompts.json`：知名 CLI 一个、小众库一个、MCP server 一个
- 手动跑通后可用 darwin-skill 做评分优化

## 7. 边界（v1 不做）

- 终端用户应用（SaaS、桌面/手机 App）
- 增量更新（应用更新后**重跑即覆盖**）
- Firecrawl 及任何付费抓取服务
- X / YouTube / 小红书等需 API key 的社区源

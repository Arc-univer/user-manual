# 调研协议

> Step 2 并行调研的执行规范。3 个 subagent 各管一层，共同遵守本协议。
> 核心原则（ADR-0001 优雅降级）：所有外部积木可选，运行时探测，有则用其最优能力，无则降级到内置工具链（WebSearch / WebFetch / Bash），**绝不因缺依赖中止**。

## 分层与信息源

| 层 | 角色 | 来源 |
|---|---|---|
| 事实层 | 说明书论断的**准绳** | 官方文档（llms.txt 探测 → Jina Reader 兜底 → WebFetch 直抓）；GitHub（README / issues / discussions / release notes，不限时）；开源库先试 Context7 MCP（可选） |
| 补充层 | 提供坑、技巧、真实体验 | last30days 引擎直调（近 30 天 Reddit/HN）；未安装则 WebSearch 降级；经典坑由 GitHub issues 兜底（不受 30 天限制） |

## 各源调用方法

### llms.txt 探测（零成本首选）
1. 先抓 `<官网根>/llms.txt`；存在则它本身就是文档索引，按索引取关键页。
2. 不存在再试 `<官网根>/llms-full.txt`；仍无进入 Jina 兜底。

### Jina Reader 兜底（免费档，无需 key）
- 用法：抓取 `https://r.jina.ai/<目标页URL>`（WebFetch 或 `curl -sL` 均可）。
- 免费档约 20 req/min，注意间隔；超限/失败则退到 WebFetch 直抓目标页。

### Context7 MCP（可选）
- 探测：当前环境存在 Context7 的 MCP 工具（如 `mcp__context7__*`）才使用；不存在直接跳过，不算失败。
- 用法：先 resolve 库 ID，再按主题取官方文档片段。适合开源库/框架的版本对应文档。

### GitHub 一手资料
- README / docs 目录：WebFetch 直抓 raw 或页面。
- issues / discussions：WebSearch `site:github.com/<owner>/<repo> <关键词>`，或抓 issues 列表页；经典坑按互动量/引用频次筛选，**不限时间**。
- release notes：确认当前版本号与近期破坏性变更，作为说明书的版本基准。

### last30days 引擎（补充层首选，可选）
- 探测与安装询问由**主窗口在 Step 2.0 完成**（探测路径：项目 `.claude/skills/last30days/` → `~/.claude/skills/last30days/` → `~/.agents/skills/last30days/` → 插件缓存目录），子 agent 不自行探测——交互决策不可下放。
- 调用：Agent C 按主窗口传入的 `{社区引擎指令}` 执行。直调模式下先读其 SKILL.md 的调用约定，再 Bash 直调引擎脚本，主题词为「<目标名> 使用体验/坑/替代」类查询；只取 Reddit/HN 相关产出。
- 降级模式（用户选择不装 / 安装失败 / 非交互无应答）：WebSearch 检索 `site:reddit.com <目标名>`、`site:news.ycombinator.com <目标名>`，时间限定近 30 天（查询中写明当前月份/年份）。

## 搜索上限（到达即停）

| Agent | 上限（搜索+抓取合计） |
|---|---|
| A 官方文档 | ≤ 12 次 |
| B GitHub | ≤ 15 次 |
| C 社区 | ≤ 10 次 |

到达上限立即停止收集，转入摘要；覆盖不到的章节在摘要「缺口」中如实说明，禁止为凑数抓取低质页面。

## 素材落盘格式

每层一个文件，写到 `<产物根>/<目标名>/research/`：`official-docs.md` / `github.md` / `community.md`。单条素材格式：

```markdown
## <素材标题>
- URL: <来源链接>
- 抓取日期: <YYYY-MM-DD>
- 可用于章节: §<章节号> <章节名>
- 关键摘录: <命令/配置/论断原文，可多条>
```

**铁律**：raw 素材只落盘，不粘贴进主对话窗口；subagent 返回给主窗口的只有精炼摘要。

## 来源标注与冲突裁决

- 说明书关键论断附 `（来源：[标题](URL)，抓取于 YYYY-MM-DD）`；资源链接章汇总全部来源。
- 事实层与补充层冲突：以事实层为准，正文标注「社区有异议：…（来源：…）」。
- 事实层内部冲突（官方文档 vs release notes）：以更新日期较新者为准，并注明版本基准。

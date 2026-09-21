---
name: user-manual
description: 为开发者向工具（CLI、开源库、框架、MCP server）生成中文使用说明书。当用户说"写一份X的使用说明书"、"X的使用手册"、"使用指南"、"X怎么用"、"给X写个manual"、"user manual"、"usage guide"，或提供工具名/官网 URL/GitHub 地址并要求生成说明书时使用。固定七章骨架（简介与定位→安装与配置→快速上手→核心功能→常见问题与坑→进阶技巧→资源链接），调研后先出大纲经用户确认，成稿落盘 manuals/<目标名>/manual.md。
---

# user-manual：开发者工具中文说明书生成器

为 CLI、开源库、框架、MCP server 等开发者向工具生成中文使用说明书：可扫读、带命令示例、面向第一次接触该工具的开发者。

## 流程总览

```
Step 1 目标识别 → Step 2.0 社区引擎探测 → Step 2 并行调研（3 subagent）→ 🔴 Step 3 大纲检查点 → Step 4 成稿 → Step 5 落盘
```

全程只有 Step 3 一个用户检查点；Step 1 的歧义澄清与 Step 2.0 的安装询问属于输入/环境确认，不算检查点。

## 用户输入工具

需要问用户时（Step 1 澄清、Step 3 检查点）：优先用当前运行时的内置提问工具（如 `AskUserQuestion`）；没有则发编号纯文本问题等用户回复。批量原则：能合并成一次调用就不要分多次问。

## 配置：产物根目录

说明书与素材写到「产物根目录/<目标名>/」。三级优先级（高→低）：

| 优先级 | 方式 | 说明 |
|---|---|---|
| 1 | 对话中显式指定 | 用户说「放到 docs/manuals 下」即用该路径（当次有效） |
| 2 | 环境变量 `USER_MANUAL_OUTPUT_DIR` | 持久生效，推荐给常用路径 |
| 3 | 默认 | 当前工作目录下的 `manuals/` |

**修改方法（环境变量）**：

- Windows 持久：`setx USER_MANUAL_OUTPUT_DIR "D:\docs\manuals"`（新开终端生效）
- macOS/Linux 持久：`export USER_MANUAL_OUTPUT_DIR="$HOME/docs/manuals"` 追加到 `~/.zshrc` 或 `~/.bashrc`
- 仅当前会话：bash `export USER_MANUAL_OUTPUT_DIR=/path/to/dir`；PowerShell `$env:USER_MANUAL_OUTPUT_DIR="D:\path"`
- Claude Code 全局：写进 `~/.claude/settings.json` 的 `env` 字段

Step 1 开始时先解析产物根目录，并在开跑前告知用户落盘位置。

## Step 1 · 目标识别

接受三种输入，锁定目标的官网与 GitHub repo：

| 输入 | 动作 |
|---|---|
| 工具名 | WebSearch 定位官网与 repo，各 1–2 次搜索 |
| 官网 URL | WebFetch 抓取，从页面找 repo 链接 |
| GitHub 地址 | WebFetch 抓 README，从徽章/正文找官网 |

- 名字唯一、来源互证一致 → 直接开跑。
- 有歧义（同名工具、name 对应多个 repo）→ 列出候选反问用户**一次**，不猜。
- 目标确认后，向用户复述一行：「目标：X（类型）｜官网：…｜repo：…｜产物将写到：…」，然后直接进入 Step 2，不额外等确认。

## Step 2.0 · 社区引擎探测（主窗口，调研前置）

last30days 是唯一「缺席会导致整层素材质量显著下降」的可选依赖，且安装成本低——**缺席时不允许静默降级，必须先问用户一次**。

1. Bash 探测安装路径（按序）：当前项目 `.claude/skills/last30days/`、`~/.claude/skills/last30days/`、`~/.agents/skills/last30days/`、`~/.claude/plugins/cache/last30days-skill/`。找到 `SKILL.md` + `scripts/` 即视为可用。
2. **已安装** → 记录路径，直接进入 Step 2。
3. **未安装** → 🛑 用提问工具询问用户一次：
   - **帮我安装**：执行 `npx skills add mvanhorn/last30days-skill`；无 npx 则 `git clone https://github.com/mvanhorn/last30days-skill` 到临时目录并把 `skills/last30days/` 复制到 `.claude/skills/last30days/`（项目级，推荐）或 `~/.claude/skills/last30days/`（用户级）。装完重新探测；引擎还需 Python 3（Windows 注意：`python3` 可能是商店占位符，exit 49 无输出，应改用 `python`），缺失则告知用户并转降级。安装失败 → 告知原因，转降级。
   - **本次降级 WebSearch**：直接进入 Step 2。
   - 用户不答（非交互场景）→ 降级继续，不阻塞（ADR-0001：绝不中止）。
4. 探测结论作为 `{社区引擎指令}` 传给 Agent C——**子 agent 不自行探测、不与用户交互**。

## Step 2 · 并行调研

同时 spawn 3 个 subagent（建议便宜模型），提示词模板见 [references/subagent-prompts.md](references/subagent-prompts.md)，协议细则见 [references/research-protocol.md](references/research-protocol.md)：

| Agent | 层 | 职责 | 上限 |
|---|---|---|---|
| A 官方文档 | 事实层 | llms.txt 探测 → Jina Reader 兜底 → WebFetch 直抓；开源库先试 Context7 MCP | ≤12 次 |
| B GitHub | 事实层 | README / issues 经典坑（不限时）/ discussions / release notes | ≤15 次 |
| C 社区 | 补充层 | 检测到 last30days 则 Bash 直调（近 30 天 Reddit/HN）；无则 WebSearch 降级 | ≤10 次 |

**铁律**：raw 素材由各 agent 落盘到 `<产物根>/<目标名>/research/`，不粘贴进主窗口；主窗口只收每个 agent ≤400 字的摘要（覆盖、关键发现、缺口、素材文件路径）。依赖全部可选，降级链见协议，绝不因缺依赖中止。

## 🔴 Step 3 · 大纲检查点（唯一，BLOCKING）

**🛑 STOP**：汇总 3 份摘要后输出大纲，**必须等用户确认才可成稿**。格式：

```markdown
## 《<目标名> 使用说明书》大纲（待确认）

目标：<名称> · <类型> ｜ 版本基准：<版本号/抓取日期>
素材概况：官方文档 N 条 / GitHub N 条 / 社区 N 条

1. 简介与定位 —— 来源：official-docs.md（官方定位）
2. 安装与配置 —— 来源：official-docs.md + github.md（README）
3. 快速上手 —— 来源：official-docs.md（quickstart）
4. 核心功能 —— 来源：official-docs.md + github.md
5. 常见问题与坑 —— 来源：community.md（近30天 N 条）+ github.md（经典坑 N 条）
6. 进阶技巧 —— 来源：community.md + official-docs.md
7. 资源链接 —— 汇总全部来源

素材缺口（如有，如实标注）：<哪章素材不足、原因>
```

用户提调整 → 修订大纲再次确认（仍在同一检查点内）。素材不足的章节由用户决定「照写（标注缺口）/ 补充调研 / 砍掉该节」。

## Step 4 · 成稿

在主窗口完成（需要判断力，不要派给子 agent）。按 [references/skeleton.md](references/skeleton.md) 的七章骨架与类型微调写作，遵守其「写作硬规则」：正文参考区间约 1500–3000 字（不作硬性要求，素材支撑多少写多少，禁止注水）、每个命令/flag 都能在素材中找到出处、关键论断附来源链接+抓取日期、事实层与社区冲突以事实层为准并标注异议。

## Step 5 · 落盘

```
<产物根>/
└── <目标名>/
    ├── manual.md      # 说明书成稿
    └── research/      # 调研素材（official-docs.md / github.md / community.md）
```

写完后向用户报告：成稿路径、字数、素材条数、版本基准、遗留缺口（如有）。重跑同一目标即覆盖旧文件，不做增量更新。

## 失败模式与 fallback

| 触发 | 一线修复 | 仍失败兜底 |
|---|---|---|
| 官网无 llms.txt | 试 llms-full.txt → Jina Reader | WebFetch 直抓首页 + /docs |
| Jina Reader 超限/失败 | WebFetch 直抓 | 以 GitHub README 顶上，大纲标注缺口 |
| 目标名有歧义 | 列候选反问用户一次 | 用户不答 → 停止，不猜 |
| 仓库 404/私有 | 告知用户，请其确认地址 | 确认不了 → 停止 |
| last30days 未安装 | 🛑 Step 2.0：停下询问用户「帮我安装 / 本次降级」 | 用户拒绝、安装失败或不答 → WebSearch 搜 site:reddit.com / site:news.ycombinator.com（近 30 天），大纲标注「社区层为降级搜索」 |
| Context7 MCP 不存在 | 跳过，走官方文档链 | —（不算失败） |
| 网络受限、多源均失败 | 换源重试一轮 | 大纲如实标注缺口，交用户裁决；**绝不编造** |
| 素材间版本矛盾 | 以官方最新文档/release notes 为准 | 大纲注明版本基准请用户确认 |

## 反模式黑名单

- 🚫 编造命令、flag、配置项、API 签名——只写素材中出现的
- 🚫 把 raw 抓取全文贴进主对话——素材只落盘 research/，主窗口只收摘要
- 🚫 跳过 🔴 检查点直接成稿
- 🚫 说明书正文用英文写（命令、代码、API 名、专名除外）
- 🚫 关键论断无来源标注，或用「据说」「可能」「一般来说」等软化措辞
- 🚫 因缺可选依赖（Context7 / last30days / Jina）中止运行
- 🚫 last30days 缺席时静默降级——必须经 Step 2.0 询问用户一次，用户拒绝或不答才可降级
- 🚫 为凑搜索次数抓低质页面——到达上限即停，缺口如实上报

## 边界（v1 不做）

- 终端用户应用（SaaS、桌面/手机 App）——用户提出时说明边界并拒绝
- 增量更新——重跑即覆盖
- Firecrawl 及任何付费抓取服务
- X / YouTube / 小红书等需 API key 的社区源

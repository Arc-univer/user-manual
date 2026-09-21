# user-manual skill 问题记录

## [UM-001] last30days 缺席时静默降级，用户失去安装选择权

**优先级**: High
**来源**: Review（2026-09-20 验证 run 1 · ripgrep 用例，用户在 🔴 大纲检查点提出）

### 1. 现象
验证运行中 Agent C 探测到 last30days 未安装（三路径均无），按 SKILL.md 失败模式表直接降级 WebSearch 完成社区层调研。预期行为（用户裁定）：**应停下来询问用户是否安装 last30days，而不是直接降级**。实际行为：静默降级，用户直到大纲检查点才得知社区层是降级产物。

### 2. 细节
- **复现步骤**:
  1. 在未安装 last30days 的环境调用 user-manual skill
  2. Step 2 社区层自动走 WebSearch 降级链，无任何询问
- **环境/上下文**: Windows 11，`~/.claude/skills`、`~/.agents/skills`、插件缓存三路径均无 last30days

### 3. 根因
ADR-0001「优雅降级、绝不中止」被实现为「自动降级」，遗漏了中间态：**降级前应给用户一次选择安装的机会**。last30days 是唯一缺席会导致整层（社区层）素材质量显著下降的可选依赖，静默降级代价高、安装成本低（一条命令），值得一次询问。子 agent 无法与用户交互，探测放在 Agent C 内部是结构性错误——探测与询问必须在主窗口完成。

### 4. 解决方案
- **短期修复**（已实施）：
  1. SKILL.md 新增 Step 2.0：主窗口在 spawn 调研 agent 前探测 last30days；未安装 → 🛑 询问一次（帮我安装 / 本次降级 WebSearch）；用户不答（非交互）→ 降级继续，不阻塞。
  2. 明确该询问属「环境确认」，与 Step 1 歧义澄清同类，不占用唯一检查点。
  3. Agent C 不再自行探测，改为接收主窗口传入的 `{社区引擎指令}`（直调命令或降级指令）。
  4. 失败模式表与反模式黑名单同步：「last30days 缺席时静默降级」列入黑名单。
- **长期优化**: 若未来新增「缺席代价高」的可选依赖，沿用同一模式：主窗口探测 → 询问一次 → 按用户选择执行；零收益依赖（如 Context7 之于非库目标）仍直接跳过不询问。

### 5. 经验教训
- 「优雅降级」≠「静默降级」：降级决策影响产物质量时，必须把选择权交还用户。
- 交互能力决定职责分层：凡需用户决策的探测，一律放主窗口；子 agent 只接收结论性指令。
- 验证价值：本缺陷在 PRD 评审与实现自审中均未暴露，首个端到端 run 的检查点即被用户发现——新 skill 必须尽早跑真实验证。

---

## [UM-002] Windows 上 `python3` 为商店占位符，last30days 引擎需改用 `python`

**优先级**: Low
**来源**: System（2026-09-20 验证 run 1，Agent C 直调 last30days 时发现并自愈）

### 1. 现象
Agent C 按约定执行 `python3 scripts/last30days.py ...`，`python3` 无输出、exit 49。改用 `python`（3.14.4）后引擎正常运行约 6 分钟，exit 0。

### 2. 细节
- **环境/上下文**: Windows 11，未安装 python3 启动器别名，`python3` 命中 Microsoft Store 占位符（App execution alias）。
- 引擎本体无 key 通道（Reddit arctic-shift / HN）工作正常；X/YouTube 等缺 key 源按预期跳过。

### 3. 根因
Windows 默认把 `python3.exe` 映射到商店安装引导，而非真实解释器；类 Unix 习惯的调用方式在 Windows 上静默失败。

### 4. 解决方案
- **短期修复**（已实施）：SKILL.md Step 2.0 的 Python 依赖说明补充 Windows 注意事项（`python3` 占位符特征：exit 49 无输出；改用 `python`）。
- **长期优化**: 无——属环境差异，已在文档层覆盖。

### 5. 经验教训
- 跨平台 skill 的「环境探测」章节要写明各平台的已知坑，而不只写happy path命令。
- 引擎原始转储落在 `C:\Users\David\Documents\Last30Days\`（last30days 默认输出目录），排查时去那里找。

# user-manual skill

为开发者向工具生成中文使用说明书的 Claude Code skill。用法与设计见[仓库根 README](../README.md)。

```
├── SKILL.md            # 技能主文件（五步流程 + 失败 fallback + 反模式黑名单）
├── references/         # 分层引用：七章骨架模板、调研协议、subagent 提示词
├── agents/openai.yaml  # Codex 兼容声明
└── test-prompts.json   # 三个端到端验证用例
```

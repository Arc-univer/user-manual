# ripgrep 使用说明书

> ripgrep（命令名 `rg`）是用 Rust 编写的行取向递归搜索工具：默认尊重 .gitignore、自动跳过隐藏文件与二进制文件，在多数场景下比 grep / ag 快一个数量级。
> 版本基准：**15.2.0**（2026-07-15 发布）；本文素材抓取于 2026-09-20，来源见各章标注与文末汇总。

## 1. 简介与定位

ripgrep 递归搜索当前目录中的正则模式，默认开启「自动过滤」：尊重 .gitignore 规则、跳过隐藏文件/目录、跳过二进制文件（用 `rg -uuu` 可关闭全部自动过滤）（来源：[README](https://github.com/BurntSushi/ripgrep/blob/master/README.md)，2026-09-20）。

**适合谁**：需要在代码库里快速定位文本/正则匹配的开发者。官方基准：Linux 内核源码树搜 `[A-Z]+_SUSPEND`，rg 0.082s，GNU grep (Unicode) 2.670s；无可优化字面量的 `[A-Za-z]{30}` 场景 rg 快 grep 32 倍（来源：[README 基准节](https://github.com/BurntSushi/ripgrep/blob/master/README.md)，2026-09-20）。

**与 grep 的关系**：官方明确「ripgrep 永远不会 100% 替代 grep」——不兼容 POSIX、行为有差异；需要无处不在、可移植的脚本时，用回 grep（来源：[FAQ](https://github.com/BurntSushi/ripgrep/blob/master/FAQ.md)，2026-09-20）。

**生态位**：社区已把「某领域的 ripgrep」当作「极速搜索」的修辞（2026-09 HN 上 ripwire 自称 "ripgrep of AI context"）；AI agent 工具链多数内置 rg（Copilot CLI、Codex）。社区有异议：Claude Code 2026-04 起在 macOS/Linux 改用内嵌 ugrep+bfs，牺牲 gitignore 感知换 grep 兼容与压缩包搜索（来源：[HN](https://news.ycombinator.com/item?id=49593050)、[ceaksan](https://ceaksan.com/en/grep-ripgrep-and-text-search-in-the-age-of-ai)，2026-09-20）。

许可证：MIT / UNLICENSE 双授权。

## 2. 安装与配置

### 安装（按平台）

| 平台 | 命令 |
|---|---|
| macOS (Homebrew) | `brew install ripgrep` |
| Windows (winget) | `winget install BurntSushi.ripgrep.MSVC` |
| Windows (Chocolatey / Scoop) | `choco install ripgrep` / `scoop install ripgrep` |
| Debian / Ubuntu | `sudo apt-get install ripgrep` |
| Arch | `sudo pacman -S ripgrep` |
| Fedora | `sudo dnf install ripgrep` |
| openSUSE | `sudo zypper install ripgrep` |
| Rust 用户 | `cargo install ripgrep`（MSRV：Rust 1.96.0） |

（来源：[README Installation](https://github.com/BurntSushi/ripgrep/blob/master/README.md#installation)，2026-09-20）

⚠️ 官方明确**不推荐 Ubuntu snap 包**（有怪 bug）；也可从 [Releases](https://github.com/BurntSushi/ripgrep/releases) 下载预编译二进制（Linux/Windows 为静态可执行文件），`.deb` 手动装时注意把 README 示例里的旧版本号替换为最新 release。

验证安装：`rg --version` 应打印 15.2.0 或更高。

### 配置文件（按需）

rg 不会自动找配置文件，必须设环境变量 `RIPGREP_CONFIG_PATH` 指向它。规则：**每行一个 shell 参数、无转义、`#` 开头为注释**；带值旗标写 `--max-columns=150`（同行空格分隔是错的）。示例：

```
--max-columns=150
--hidden
--glob=!.git/*
--smart-case
```

配置文件参数会被前置到命令行参数之前，命令行可临时覆盖；`rg --debug` 可查看加载了哪个配置，`--no-config` 强制不读（来源：[GUIDE 配置文件节](https://github.com/BurntSushi/ripgrep/blob/master/GUIDE.md#configuration-file)，2026-09-20）。

man page 与 shell 补全：`rg --generate man`、`rg --generate complete-zsh`（另有 bash/fish/powershell）（来源：[FAQ](https://github.com/BurntSushi/ripgrep/blob/master/FAQ.md)，2026-09-20）。

## 3. 快速上手

**① 最小用法**——递归搜当前目录（递归是默认行为，无需 `-r`）：

```bash
rg foo
```

默认带行号、终端支持时带颜色。等价于 `rg foo ./`。

**② 常用场景**——限定目录 + 字面量匹配（含特殊字符时用 `-F` 免转义）：

```bash
rg -F 'fn write(' src
```

**③ 组合用法**——用 glob 收窄范围，只搜 tsx 文件：

```bash
rg 'handleSubmit' src/components/ -g '*.tsx'
```

（来源：[GUIDE](https://github.com/BurntSushi/ripgrep/blob/master/GUIDE.md) 与[社区最佳实践](https://ceaksan.com/en/grep-ripgrep-and-text-search-in-the-age-of-ai)，2026-09-20）

如果 rg 报「没有搜索任何文件」，先加 `--debug` 重跑——常见原因是全局 `$HOME/.gitignore` 里有 `*` 规则。

## 4. 核心功能

### 自动过滤（rg 的招牌）

忽略规则三级优先级（低→高）：`.gitignore`（含全局与父目录）< `.ignore` < `.rgignore`；另尊重 git 的 exclude 配置。默认还跳过隐藏文件、含 NUL 字节的二进制文件，不跟随符号链接。

| 需求 | 旗标 |
|---|---|
| 关闭 ignore 规则 | `--no-ignore` |
| 连隐藏文件也搜 | `--hidden`（`-.`） |
| 连二进制也搜 | `-a` / `--binary` |
| 跟随符号链接 | `-L` / `--follow` |
| 一键放开 | `-u` 关 ignore → `-uu` 加隐藏 → `-uuu` 加二进制 |
| 非 git 目录也让 .gitignore 生效 | `--no-require-git` |

白名单技巧：`.gitignore` 忽略了 `log/`，在同目录建 `.ignore` 写 `!log/` 即可单独放行（来源：[GUIDE 自动过滤节](https://github.com/BurntSushi/ripgrep/blob/master/GUIDE.md#automatic-filtering)，2026-09-20）。

### 手动过滤：glob 与文件类型

```bash
rg lexopt -g '*.toml'        # 只搜 toml（单引号防 shell 展开）
rg lexopt -g '!*.toml'       # 排除 toml（命令行上 ! 是黑名单）
rg 'fn run' -trust           # 只搜 Rust 文件；排除用 -Trust
rg --type-list               # 查看内置类型对应的 glob
rg --type-add 'web:*.{html,css,js}' -tweb title   # 自定义类型（当次有效）
```

注意：命令行 `-g '!*'` 的 `!` 是**排除**，与 .gitignore 文件里 `!`=白名单语义相反；多个 `-g` 后者覆盖前者（来源：[GUIDE](https://github.com/BurntSushi/ripgrep/blob/master/GUIDE.md#manual-filtering-globs)，2026-09-20）。

### 替换输出（只改输出，绝不改文件）

```bash
rg fast README.md -r FAST                        # 匹配片段替换
rg 'fast\s+(?P<word>\w+)' README.md -r 'fast-$word'   # 命名捕获组
```

官方立场：「ripgrep **永远不会修改你的文件**，`-r/--replace` 只控制输出」。要真正落盘替换，管道给 sed 或 fastmod：

```bash
rg foo --files-with-matches -0 | xargs -0 sed -i 's/foo/bar/g'   # GNU sed；macOS 用 sed -i ''
```

（来源：[GUIDE 替换节](https://github.com/BurntSushi/ripgrep/blob/master/GUIDE.md#replacements)、[FAQ](https://github.com/BurntSushi/ripgrep/blob/master/FAQ.md#search-and-replace)，2026-09-20）

### 双正则引擎与特殊输入

- 默认 Rust regex 引擎：有限自动机保证**线性最坏时间**，代价是不支持 lookaround / 回溯引用；`-P` 切 PCRE2（如 `rg -P '(\w{10})\1'`），官方 release 大多已内置。
- `-U` 多行匹配；`-z` 搜压缩文件（gzip/xz/zstd 等，靠外部解压二进制，**不支持 tar.gz 等归档格式**）。
- 编码：默认假定 ASCII 兼容、UTF-16 靠 BOM 嗅探自动转码；其他编码用 `-E gbk` 等显式指定；`-E none` 完全关闭编码逻辑搜原始字节。

（来源：[GUIDE](https://github.com/BurntSushi/ripgrep/blob/master/GUIDE.md)、[FAQ](https://github.com/BurntSushi/ripgrep/blob/master/FAQ.md)，2026-09-20）

### 常用旗标速查

| 旗标 | 作用 |
|---|---|
| `-i` / `-S` | 忽略大小写 / 智能大小写（模式含大写才敏感） |
| `-F` / `-w` | 字面量 / 整词匹配 |
| `-c` / `--files` | 只出计数 / 只列会被搜索的文件 |
| `-C n` | 显示前后 n 行上下文 |
| `-M n` | 超长行截断到 n 列 |
| `--sort path` | 按路径排序（会关闭并行） |
| `--json` | 结构化输出，供程序消费 |
| `--debug` / `--trace` | 排错：看过滤决策与搜索引擎选择 |

（来源：[GUIDE 常用选项节](https://github.com/BurntSushi/ripgrep/blob/master/GUIDE.md#common-options)，2026-09-20）

## 5. 常见问题与坑

**①「文件明明存在，rg 就是搜不到」** —— 新人最高频困惑。原因：默认过滤（gitignore/隐藏/二进制）。解法：逐级放开 `-u` / `-uu` / `-uuu`；非 git 仓库目录里 .gitignore 默认不生效，加 `--no-require-git`（来源：[GUIDE](https://github.com/BurntSushi/ripgrep/blob/master/GUIDE.md#automatic-filtering)、[社区讨论](https://github.com/BurntSushi/ripgrep/issues/2881)，2026-09-20）。

**② Windows 下 `rg PATTERN *.txt` 报 os error 123** —— cmd/PowerShell 不展开通配符（issue 仍 open）。解法：用 rg 自带的 `-g`：`rg PATTERN -g '*.txt'`（来源：[issue #234](https://github.com/BurntSushi/ripgrep/issues/234)，2026-09-20）。

**③ 二进制文件被静默跳过** —— grep 会提示 `Binary file ... matches`，rg 默认一声不吭。解法：`-a` 按文本搜，`--binary` 显式搜二进制（来源：[issue #306](https://github.com/BurntSushi/ripgrep/issues/306)，2026-09-20）。

**④ Windows 管道后中文变「娴嬭瘯」** —— 控制台默认代码页非 UTF-8，非 rg 的 bug。解法：PowerShell 设 `[Console]::OutputEncoding = [System.Text.Encoding]::UTF8`（或 `chcp 65001`）（来源：[issue #2510](https://github.com/BurntSushi/ripgrep/issues/2510) + [FAQ](https://github.com/BurntSushi/ripgrep/blob/master/FAQ.md)，2026-09-20）。

**⑤ PowerShell 蓝底下文件名「隐形」** —— 默认配色 path 为 magenta，与蓝底混在一起。解法：`--colors 'path:fg:cyan'`，或换 Windows Terminal（来源：[issue #342](https://github.com/BurntSushi/ripgrep/issues/342)，2026-09-20）。

**⑥ 结果顺序每次不一样** —— 并行搜索导致，属预期。需要稳定顺序加 `--sort path`（代价：关闭并行）（来源：[FAQ](https://github.com/BurntSushi/ripgrep/blob/master/FAQ.md#order)，2026-09-20）。

**⑦ `-P`（PCRE2）突然变慢或报 UTF-8 error** —— PCRE2 无法整块剥离换行被迫逐行搜索，且要求合法 UTF-8 输入（非法字节可能直接报错中断）。提速配方：`-U --no-pcre2-unicode`（官方示例 2.96s → 1.12s）（来源：[FAQ](https://github.com/BurntSushi/ripgrep/blob/master/FAQ.md#pcre2-slow)，2026-09-20）。

**⑧「rg 比 grep 慢」的性能误区** —— 对比前须对齐工作量：`rg -uuu -j1`（关全部过滤 + 单线程）才是与 `grep -r` 的对等基准；再检查配置文件偷加的旗标，用 `--debug` 自查（来源：[issue #1335](https://github.com/BurntSushi/ripgrep/issues/1335)，2026-09-20）。同理，对「ugrep 比 rg 快」的宣传要甄别：独立复测（官方 benchsuite，rg 15.1 vs ugrep 7.5）显示多数用例 rg 领先（来源：[ugrep issue #517](https://github.com/Genivia/ugrep/issues/517)，2026-09-20；社区异议长期存在于 [rg Discussion #2597](https://github.com/BurntSushi/ripgrep/discussions/2597)）。

**⑨ 显式传路径时父目录 gitignore 失效** —— 15.0.0 之前的老 bug，仓库根 `rg foo a/` 会搜到被忽略的文件。解法：升级到 15.0.0+（已修，关联 8 个 issue）；旧版变通是 `cd` 进目录再搜（来源：[issue #829](https://github.com/BurntSushi/ripgrep/issues/829)、[CHANGELOG](https://github.com/BurntSushi/ripgrep/blob/master/CHANGELOG.md)，2026-09-20）。

## 6. 进阶技巧

**预处理器搜 PDF/任意格式**：`--pre` 对每个文件先跑外部命令（内容走 stdin），配 `--pre-glob` 限定范围——官方示例搜 PDF 从 0.138s 优化到 0.008s，同任务比 pdfgrep 快约 2 倍：

```bash
rg --pre ./preprocess --pre-glob '*.pdf' 'The Commentz-Walter algorithm'
# preprocess 脚本体：#!/bin/sh + exec pdftotext - -
```

**两段式搜索**（agent/脚本场景省 token）：先 `rg -l 'useAuth'` 拿文件清单，再对少量文件 `rg -C 3` 看上下文；程序消费用 `--json`（可接 delta 分页器：`rg --json pattern | delta`）。

**PCRE2 提速**：`rg -P -U --no-pcre2-unicode '...'`（见 §5-⑦）。

**大正则/大模式文件**：报 `Compiled regex exceeds size limit` 加 `--regex-size-limit 1G`；`-f` 模式文件变慢加 `--dfa-size-limit 1G`。

**颜色定制**：`rg pat --colors 'match:none' --colors 'match:bg:0x33,0x66,0xFF' --colors 'match:style:bold'`（Windows 10 真彩需先 `none` 清默认）。

**生态搭配**：知道搜什么**字符串**用 rg；知道搜什么**代码结构**用 ast-grep（tree-sitter AST 级，带 MCP）——社区共识是互补分层，不是替代。从 grep 迁移直奔 rg：ag 自 2018 年后实质停止维护（来源：[GUIDE](https://github.com/BurntSushi/ripgrep/blob/master/GUIDE.md#preprocessor)、[ceaksan](https://ceaksan.com/en/grep-ripgrep-and-text-search-in-the-age-of-ai)、[codeant](https://codeant.ai/blogs/ripgrep-vs-grep-performance)，2026-09-20）。

## 7. 资源链接

**官方**

- 仓库与文档主体（README / GUIDE.md / FAQ.md / CHANGELOG.md）：https://github.com/BurntSushi/ripgrep
- 预编译二进制 Releases：https://github.com/BurntSushi/ripgrep/releases
- 正则语法文档：https://docs.rs/regex/1/regex/#syntax
- 作者性能分析长文：https://blog.burntsushi.net/ripgrep/

**社区**

- 工具特性对比表（ack 作者维护）：https://beyondgrep.com/feature-comparison/
- 非官方在线试用 / 交互教程：https://codapi.org/ripgrep/ 、https://codapi.org/try/ripgrep/
- 非官方中文翻译（可能滞后）：https://github.com/chinanf-boy/ripgrep-zh

**本文引用来源**（均抓取于 2026-09-20）：GitHub issues [#234](https://github.com/BurntSushi/ripgrep/issues/234) / [#306](https://github.com/BurntSushi/ripgrep/issues/306) / [#342](https://github.com/BurntSushi/ripgrep/issues/342) / [#829](https://github.com/BurntSushi/ripgrep/issues/829) / [#1335](https://github.com/BurntSushi/ripgrep/issues/1335) / [#2510](https://github.com/BurntSushi/ripgrep/issues/2510) / [#2881](https://github.com/BurntSushi/ripgrep/issues/2881)；[Discussion #2597](https://github.com/BurntSushi/ripgrep/discussions/2597)；[ugrep issue #517](https://github.com/Genivia/ugrep/issues/517)；[HN: ripwire](https://news.ycombinator.com/item?id=49593050)；[HN: ripgrep 15](https://news.ycombinator.com/item?id=45604206)；[ceaksan: AI 时代的 grep 与 ripgrep](https://ceaksan.com/en/grep-ripgrep-and-text-search-in-the-age-of-ai)；[codeant: ripgrep vs grep 性能](https://codeant.ai/blogs/ripgrep-vs-grep-performance)。

> 调研素材留档于同目录 `research/`（official-docs.md / github.md / community.md），可供核查每条论断出处。

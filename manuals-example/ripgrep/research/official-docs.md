# ripgrep 官方文档调研素材（事实层）

- 调研范围：官方仓库 README.md / GUIDE.md / FAQ.md / CHANGELOG.md（无独立官网）
- llms.txt 探测结果：`https://raw.githubusercontent.com/BurntSushi/ripgrep/master/llms.txt` 返回 404，不存在（预期内）
- 抓取日期统一为 2026-09-20

## 工具定位与一句话定义
- URL: https://github.com/BurntSushi/ripgrep/blob/master/README.md
- 抓取日期: 2026-09-20
- 可用于章节: §1 简介与定位
- 关键摘录:
  - "ripgrep is a line-oriented search tool that recursively searches the current directory for a regex pattern. By default, ripgrep will respect gitignore rules and automatically skip hidden files/directories and binary files. (To disable all automatic filtering by default, use `rg -uuu`.)"
  - "ripgrep has first class support on Windows, macOS and Linux, with binary downloads available for every release. ripgrep is similar to other popular search tools like The Silver Searcher, ack and grep."
  - "Dual-licensed under MIT or the UNLICENSE."
  - 二进制名为 `rg`："The binary name for ripgrep is `rg`."

## 为什么用 / 不用 ripgrep（适用场景边界）
- URL: https://github.com/BurntSushi/ripgrep/blob/master/README.md
- 抓取日期: 2026-09-20
- 可用于章节: §1 简介与定位; §5 常见问题与坑
- 关键摘录:
  - 适用："use ripgrep if you like speed, filtering by default, fewer bugs and Unicode support."
  - 不适用场景（原文）："You need a portable and ubiquitous tool. While ripgrep works on Windows, macOS and Linux, it is not ubiquitous and it does not conform to any standard such as POSIX. The best tool for this job is good old grep."
  - FAQ 对"能否替代 grep"的官方结论："ripgrep trivially *cannot* replace grep... ripgrep will *never* replace grep"（指 bug-for-bug 完全兼容层面）；但在"部分场景替代"意义上成立。"ripgrep never was, is or will be a 100% drop-in replacement for grep"
  - "Do you care about POSIX compatibility? If so, then you can't use ripgrep because it never was, isn't and never will be POSIX compatible."
  - 名字由来："rip" 取 "to rip through your text"（快）之意；作者称 RIP=Rest in Peace 的巧合是发布后才被指出。

## 性能卖点与基准数据
- URL: https://github.com/BurntSushi/ripgrep/blob/master/README.md
- 抓取日期: 2026-09-20
- 可用于章节: §1 简介与定位; §6 进阶技巧
- 关键摘录:
  - Linux 内核源码树搜 `[A-Z]+_SUSPEND`：`rg -n -w '[A-Z]+_SUSPEND'` 0.082s，对比 git grep 0.273s、ag 0.443s、GNU grep (Unicode) 2.670s。
  - 单个大文件（~13GB OpenSubtitles）：`rg -w 'Sherlock [A-Z]\w+'` 1.042s vs GNU grep (Unicode) 6.577s。
  - 快的原因（原文要点）：基于 Rust regex 引擎（有限自动机 + SIMD + 字面量优化）；Unicode 支持内建于 DFA；自动在 memory map 与增量缓冲间选择搜索策略；用 RegexSet 同时匹配多个 gitignore glob；基于 crossbeam/ignore 的无锁并行目录迭代器。
  - 详细基准分析见作者博客：https://blog.burntsushi.net/ripgrep/

## 安装方式总览（各平台包管理器）
- URL: https://github.com/BurntSushi/ripgrep/blob/master/README.md#installation
- 抓取日期: 2026-09-20
- 可用于章节: §2 安装与配置
- 关键摘录:
  - 预编译二进制："Archives of precompiled binaries for ripgrep are available for Windows, macOS and Linux."（GitHub Releases；Linux/Windows 为静态可执行文件）
  - macOS：`brew install ripgrep`；MacPorts：`sudo port install ripgrep`
  - Windows：`choco install ripgrep`；`scoop install ripgrep`；`winget install BurntSushi.ripgrep.MSVC`
  - Arch：`sudo pacman -S ripgrep`；Fedora：`sudo dnf install ripgrep`；openSUSE：`sudo zypper install ripgrep`；Gentoo：`sudo emerge sys-apps/ripgrep`
  - Debian/Ubuntu：`sudo apt-get install ripgrep`（或下载 release 的 .deb 用 `sudo dpkg -i` 安装）；官方明确不推荐 snap 包（"it is no longer a recommended installation option"）
  - FreeBSD：`sudo pkg install ripgrep`；OpenBSD：`doas pkg_add ripgrep`；Void：`sudo xbps-install -Syv ripgrep`；Nix：`nix-env --install ripgrep`；Guix：`guix install ripgrep`；Flox：`flox install ripgrep`
  - Cargo：`cargo install ripgrep`（MSRV：Rust 1.96.0；二进制含调试符号，可用 `strip` 减小体积）；或 `cargo binstall ripgrep`

## 从源码构建与可选 PCRE2 特性
- URL: https://github.com/BurntSushi/ripgrep/blob/master/README.md#building
- 抓取日期: 2026-09-20
- 可用于章节: §2 安装与配置; §6 进阶技巧
- 关键摘录:
  - `git clone https://github.com/BurntSushi/ripgrep && cd ripgrep && cargo build --release`（Rust 1.96.0 stable 或更新）
  - PCRE2 可选特性：`cargo build --release --features 'pcre2'`；优先经 pkg-config 链接系统 PCRE2，否则从源码构建并静态链接；`PCRE2_SYS_STATIC=1` 可强制静态链接
  - MUSL 静态构建：`rustup target add x86_64-unknown-linux-musl && cargo build --release --target x86_64-unknown-linux-musl`
  - 测试：`cargo test --all`

## 基础用法与递归搜索（快速上手）
- URL: https://github.com/BurntSushi/ripgrep/blob/master/GUIDE.md
- 抓取日期: 2026-09-20
- 可用于章节: §3 快速上手
- 关键摘录:
  - 单文件搜索：`rg fast README.md`（默认带行号、终端支持时带颜色）
  - 正则示例：`rg 'fast\w+' README.md`；字面量模式：`rg -F 'fn write('`
  - 递归是默认行为："recursively searching your current working directory is the default mode of operation"，即 `rg foo` 等价于 `rg foo ./`
  - 限定目录：`rg 'fn write\(' src`
  - 排错提示：若 rg 报没有搜索任何文件，用 `--debug` 重跑；常见原因是 `$HOME/.gitignore` 里有 `*` 规则
  - 正则语法文档：https://docs.rs/regex/*/regex/#syntax

## 自动过滤：gitignore 语义、隐藏文件、二进制、符号链接
- URL: https://github.com/BurntSushi/ripgrep/blob/master/GUIDE.md#automatic-filtering
- 抓取日期: 2026-09-20
- 可用于章节: §4 核心功能
- 关键摘录:
  - 默认忽略三类 glob 来源，优先级从低到高：`.gitignore`（含全局与同仓库父目录的）< `.ignore`（冲突时优先于 gitignore）< `.rgignore`（优先于 .ignore）；另尊重 `$GIT_DIR/info/exclude` 与 `core.excludesFile`
  - 默认还跳过：隐藏文件/目录；含 NUL 字节的二进制文件；不跟随符号链接
  - 开关：`--no-ignore` 关闭所有 ignore 过滤；`--hidden`(`-.`) 搜隐藏文件；`--text`(`-a`) 搜二进制；`--follow`(`-L`) 跟随符号链接
  - 便捷旗标 `-u/--unrestricted`：`-u` 关 gitignore，`-uu` 加搜隐藏文件，`-uuu` 再加搜二进制
  - 白名单示例：`.gitignore` 有 `log/`，同目录建 `.ignore` 写 `!log/` 即可让 rg 搜索 log 目录
  - 大小写不敏感 ignore：`--ignore-file-case-insensitive`（Windows/macOS 有用，但有显著性能损耗，默认关闭）
  - gitignore 规则只在 git 仓库内生效，除非给 `--no-require-git`

## 手动过滤：-g glob
- URL: https://github.com/BurntSushi/ripgrep/blob/master/GUIDE.md#manual-filtering-globs
- 抓取日期: 2026-09-20
- 可用于章节: §4 核心功能
- 关键摘录:
  - 包含：`rg lexopt -g '*.toml'`（单引号防 shell 展开 `*`）
  - 排除：`rg lexopt -g '!*.toml'`（命令行上 `!` 是黑名单，与 .gitignore 里 `!`=白名单语义相反）
  - 多个 -g 后者覆盖前者：`rg lexopt -g '!*.toml' -g '*.toml'` 只搜 toml；顺序反过来则什么都不搜

## 手动过滤：文件类型 -t/-T/--type-add
- URL: https://github.com/BurntSushi/ripgrep/blob/master/GUIDE.md#manual-filtering-file-types
- 抓取日期: 2026-09-20
- 可用于章节: §4 核心功能
- 关键摘录:
  - 按类型包含：`rg 'fn run' -trust`（即 `--type rust`）；排除：`rg lexopt -Trust`（`--type-not rust`）
  - 查看类型对应 glob：`rg --type-list`，例：`make: *.mak, *.mk, GNUmakefile, Gnumakefile, Makefile, gnumakefile, makefile`
  - 自定义类型：`rg --type-add 'web:*.{html,css,js}' -tweb title`（`--type-add` 只对当次命令生效；持久化靠 shell alias 或配置文件）
  - 特殊类型 `--type all`：匹配 `--type-list` 里所有已定义类型的文件

## 替换输出：-r/--replace（绝不改文件）
- URL: https://github.com/BurntSushi/ripgrep/blob/master/GUIDE.md#replacements
- 抓取日期: 2026-09-20
- 可用于章节: §4 核心功能; §5 常见问题与坑
- 关键摘录:
  - 基本：`rg fast README.md -r FAST`（`--replace` 只作用于匹配到的文本片段）
  - 整行替换：`rg '^.*fast.*$' README.md -r FAST`，或 `rg fast README.md -or FAST`（`-o/--only-matching` 组合）
  - 捕获组：`rg 'fast\s+(\w+)' README.md -r 'fast-$1'`；命名捕获：`rg 'fast\s+(?P<word>\w+)' README.md -r 'fast-$word'`
  - 重要论断："ripgrep **will never modify your files**. The `--replace` flag only controls ripgrep's output. (And there is no flag to let you do a replacement in a file.)"

## 配置文件：RIPGREP_CONFIG_PATH
- URL: https://github.com/BurntSushi/ripgrep/blob/master/GUIDE.md#configuration-file
- 抓取日期: 2026-09-20
- 可用于章节: §2 安装与配置; §4 核心功能
- 关键摘录:
  - rg 不会自动找配置文件；必须设环境变量 `RIPGREP_CONFIG_PATH` 指向配置文件
  - 格式两条规则：每行是一个 shell 参数（去首尾空白）；`#` 开头的行是注释。"there is no escaping"
  - 带值旗标两种写法：`--max-columns=150` 同行，或旗标与值分两行；写 `--max-columns 150`（同行空格分隔）是错的
  - 覆盖机制：配置文件的参数被"prepend"到命令行参数之前，后出现的旗标覆盖先前的，故命令行可临时覆盖配置
  - 调试：`--debug` 会显示加载了哪个配置文件；`--no-config` 强制不读任何配置
  - 示例配置摘录：`--max-columns=150`、`--max-columns-preview`、`--hidden`、`--glob=!.git/*`、`--colors=line:none`、`--colors=line:style:bold`、`--smart-case`

## 文件编码：UTF-8 优先、UTF-16 BOM 嗅探、-E/--encoding
- URL: https://github.com/BurntSushi/ripgrep/blob/master/GUIDE.md#file-encoding
- 抓取日期: 2026-09-20
- 可用于章节: §4 核心功能; §5 常见问题与坑
- 关键摘录:
  - 默认 `--encoding auto`：假定输入 ASCII 兼容（ASCII/latin1/UTF-8）；UTF-16 通过 BOM 嗅探自动转码为 UTF-8 再搜索（有性能损耗）
  - 其他编码：`rg -E gbk ...`（取值来自 WHATWG Encoding Standard）；指定后对所有文件生效（除非文件有 BOM）
  - `-E none` 完全禁用编码逻辑（含 BOM 嗅探），直接搜原始字节，例：`rg '(?-u)\(\x045\x04@\x04;\x04>\x04:\x04' -E none -a some-utf16-file`
  - 正则内可局部关 Unicode：`rg '(?-u:.)'`（让 `.` 匹配任意字节而非码点）
  - 默认不要求输入是合法 UTF-8："ripgrep can and will search arbitrary bytes"

## 二进制数据三种模式
- URL: https://github.com/BurntSushi/ripgrep/blob/master/GUIDE.md#binary-data
- 抓取日期: 2026-09-20
- 可用于章节: §4 核心功能; §5 常见问题与坑
- 关键摘录:
  - 判定启发式："a file is considered 'binary' if and only if it contains a `NUL` byte somewhere in its contents"
  - 三模式：默认（检测到 NUL 即停止搜索，仅适用于递归遍历到的文件；显式指名文件则自动进入 binary 模式）；binary 模式（`--binary`，继续搜到文件尾或见到首个匹配为止）；text 模式（`-a/--text`，完全关检测，大二进制文件可能吃很多内存）
  - 不一致性警告：用 memory map 时只对文件头几 KB + 匹配行做二进制检测；不用 mmap 时检测所有字节。要一致行为可用 `--no-mmap`

## 预处理器：--pre 与 --pre-glob
- URL: https://github.com/BurntSushi/ripgrep/blob/master/GUIDE.md#preprocessor
- 抓取日期: 2026-09-20
- 可用于章节: §6 进阶技巧
- 关键摘录:
  - 机制：`--pre` 接受一个命令，对每个被搜文件执行；文件路径作为第一个参数传入，文件内容走 stdin
  - PDF 搜索示例：`rg --pre ./preprocess 'The Commentz-Walter algorithm' 1995-watson.pdf`，脚本体 `#!/bin/sh` + `exec pdftotext - -`
  - 健壮版脚本：按扩展名/`file` 嗅探分发（pdf→pdftotext，Zstandard→`pzstd -cdq`，其他→`exec cat`）
  - 性能：`--pre-glob '*.pdf'` 限定只对匹配 glob 的文件跑预处理；官方示例中 0.138s → 0.008s
  - 对比 pdfgrep：同一 PDF 搜索 rg+pre 0.697s vs pdfgrep 1.336s

## 常用选项速查（GUIDE 官方清单）
- URL: https://github.com/BurntSushi/ripgrep/blob/master/GUIDE.md#common-options
- 抓取日期: 2026-09-20
- 可用于章节: §3 快速上手; §4 核心功能
- 关键摘录:
  - `-h` 简短帮助；`--help` 长帮助（接近 man page）
  - `-i/--ignore-case`；`-S/--smart-case`（模式含大写时自动失效）
  - `-F/--fixed-strings` 字面量；`-w/--word-regexp`（用词边界包裹，`\b{start-half}`/`\b{end-half}` 不要求一侧是词字符）
  - `-c/--count`；`--files`（只列会搜哪些文件）；`-a/--text`；`-U/--multiline`；`-z/--search-zip`（gzip/bzip2/lzma/xz/lz4/brotli/zstd，默认关闭）；`-C/--context`；`--sort path`（排序输出，禁用并行）；`-L/--follow`；`-M/--max-columns`；`--debug`

## FAQ：man page 与 shell 补全生成
- URL: https://github.com/BurntSushi/ripgrep/blob/master/FAQ.md
- 抓取日期: 2026-09-20
- 可用于章节: §2 安装与配置; §7 资源链接
- 关键摘录:
  - man page：`rg --generate man | man -l -`（或写入 man/man1 后设 MANPATH）；包管理器安装通常已就位，直接 `man rg`
  - 补全：`rg --generate complete-bash` / `complete-fish` / `complete-zsh` / `complete-powershell`，写入对应 shell 补全目录（zsh 推荐写 `_rg` 文件并加 fpath；`source <(rg --generate complete-zsh)` 更省事但更慢）

## FAQ：结果顺序不确定（并行导致）
- URL: https://github.com/BurntSushi/ripgrep/blob/master/FAQ.md#order
- 抓取日期: 2026-09-20
- 可用于章节: §5 常见问题与坑
- 关键摘录:
  - 原因：默认并行搜索，线程交错导致输出顺序不确定且每次运行可变
  - 解法："The only way to make the order of results consistent is to ask ripgrep to sort the output. Currently, this will disable all parallelism." 即 `--sort path`

## FAQ：压缩文件搜索 -z
- URL: https://github.com/BurntSushi/ripgrep/blob/master/FAQ.md#compressed
- 抓取日期: 2026-09-20
- 可用于章节: §4 核心功能; §5 常见问题与坑
- 关键摘录:
  - `-z/--search-zip` 支持 gzip/bzip2/xz/lzma/lz4/Brotli/Zstd；解压靠外部同名二进制（"ripgrep does decompression by shelling out to another process"）
  - 不支持归档格式："ripgrep currently does not search archive formats, so `*.tar.gz` files, for example, are skipped."

## FAQ：多行搜索与 PCRE2（lookaround/回溯引用）
- URL: https://github.com/BurntSushi/ripgrep/blob/master/FAQ.md#multiline 与 #fancy
- 抓取日期: 2026-09-20
- 可用于章节: §4 核心功能; §6 进阶技巧
- 关键摘录:
  - `-U/--multiline` 允许跨行匹配
  - 默认引擎基于有限状态机，保证线性最坏时间，故不支持 lookaround/backreferences
  - `-P/--pcre2` 切 PCRE2：`rg -P '(\w{10})\1'`；不支持时报错 "PCRE2 is not available in this build of ripgrep"；GitHub 官方 release 大多内置 PCRE2
  - README 另提 `--engine (default|pcre2|auto)` 与 `--auto-hybrid-regex`（按需切 PCRE2）

## FAQ：PCRE2 为什么变慢（性能深坑）
- URL: https://github.com/BurntSushi/ripgrep/blob/master/FAQ.md#pcre2-slow
- 抓取日期: 2026-09-20
- 可用于章节: §5 常见问题与坑; §6 进阶技巧
- 关键摘录:
  - 原因一：默认引擎能静态剥离 `\n` 从而整块搜索；PCRE2 无类似 API，被迫逐行"slow line searcher"（可用 `--trace` 观察 "fast line searcher" vs "slow line searcher"）
  - 原因二：PCRE2 Unicode 模式要求输入必须是合法 UTF-8，rg 需先转码（替换非法序列）再喂给 PCRE2
  - 加速结论："if you want PCRE2 to go as fast as possible and you don't care about Unicode and you don't care about matches possibly spanning across multiple lines, then enable multiline mode with `-U` and disable PCRE2's Unicode support with the `--no-pcre2-unicode` flag."（示例中 2.96s → 1.12s，JIT 生效）
  - 附带坑：`-P` 遇非法 UTF-8 可能报错中断搜索（"PCRE2: error matching: UTF-8 error"）

## FAQ：颜色配置与 Windows 真彩色
- URL: https://github.com/BurntSushi/ripgrep/blob/master/FAQ.md#colors
- 抓取日期: 2026-09-20
- 可用于章节: §6 进阶技巧; §5 常见问题与坑
- 关键摘录:
  - `--color`（when：never/auto/always/ansi，默认 auto）与 `--colors`（which）分开
  - `--colors '{type}:{attribute}:{value}'`；type ∈ path/line/column/match；attribute ∈ fg/bg/style；颜色值支持 8 色名、0-255（含 0x 十六进制）、RGB 三元组（真彩）；`--colors '{type}:none'` 清空默认
  - 示例：`rg somepattern --colors 'match:none' --colors 'match:bg:0x33,0x66,0xFF' --colors 'match:fg:white' --colors 'match:style:bold'`
  - Windows 10 控制台真彩需先 `'match:none'` 清默认（bold 与真彩 ANSI 不能同时用）
  - 仿 The Silver Searcher 输出：`--colors line:fg:yellow --colors line:style:bold --colors path:fg:green --colors path:style:bold --colors match:fg:black --colors match:bg:yellow --colors match:style:nobold`

## FAQ：Windows 特有坑（cygwin 路径转换 / PowerShell 别名 / 非 ASCII 管道）
- URL: https://github.com/BurntSushi/ripgrep/blob/master/FAQ.md
- 抓取日期: 2026-09-20
- 可用于章节: §5 常见问题与坑
- 关键摘录:
  - cygwin 下 `rg /foo` 会被路径转换成 `rg C:/msys64/foo`；解法：`rg //foo` 或 `MSYS_NO_PATHCONV=1 rg /foo`
  - PowerShell 函数别名需显式转发 `$input` 与 `$args`（FAQ 给出完整 `grep` 函数示例，需判空 `$input` 否则会让 rg 读空 stdin）
  - PowerShell 管道非 ASCII 默认被转成 `?`：需设 `$OutputEncoding = [System.Text.UTF8Encoding]::new()`，另可设 `[System.Console]::OutputEncoding = [System.Text.Encoding]::UTF8`
  - `rg` 被执行成别的命令：多为 shell alias 冲突（Oh My Zsh Rails 插件把 `rg` 别名成 `rails generate`）；排查 `which rg`；临时绕过用 `command rg`/`\rg`/`'rg'`

## FAQ：正则体积上限与 -f 模式文件变慢
- URL: https://github.com/BurntSushi/ripgrep/blob/master/FAQ.md
- 抓取日期: 2026-09-20
- 可用于章节: §5 常见问题与坑; §6 进阶技巧
- 关键摘录:
  - 超大正则报错："Compiled regex exceeds size limit of 10485760 bytes." 解法：`rg '\pL{1000}' --regex-size-limit 1G`（只是上限，不代表真用那么多内存）
  - `-f/--file` 从文件读多个模式；文件太大变慢多为 DFA 缓存溢出，解法 `--dfa-size-limit 1G`

## FAQ：搜索并替换（需借助外部工具）
- URL: https://github.com/BurntSushi/ripgrep/blob/master/FAQ.md#search-and-replace
- 抓取日期: 2026-09-20
- 可用于章节: §5 常见问题与坑; §6 进阶技巧
- 关键摘录:
  - 官方立场："Using ripgrep alone, you can't. ripgrep is a search tool that will never touch your files."
  - GNU sed：`rg foo --files-with-matches | xargs sed -i 's/foo/bar/g'`
  - BSD/macOS sed：`rg foo --files-with-matches | xargs sed -i '' 's/foo/bar/g'`
  - 含空格路径：`rg foo --files-with-matches -0 | xargs -0 sed -i 's/foo/bar/g'`
  - 官方顺带推荐 fastmod（Facebook）作为更符合人体工学的 search-and-replace 工具

## 版本基准与近期重要变更（CHANGELOG）
- URL: https://github.com/BurntSushi/ripgrep/blob/master/CHANGELOG.md
- 抓取日期: 2026-09-20
- 可用于章节: §1 简介与定位; §7 资源链接
- 关键摘录:
  - 当前最新：**15.2.0 (2026-07-15)**——修若干 gitignore 匹配 bug（含跨多目录搜索场景）；目录遍历性能提升（#3293）；尊重 `GIT_CONFIG_GLOBAL`/`GIT_CONFIG_SYSTEM`；新增 aarch64-unknown-linux-musl release 二进制
  - 15.1.0 (2025-10-22)：修 15.0.0 引入的 `--line-buffered` 回归（输出延迟、`tail -f` 场景）
  - 15.0.0 (2025-10-15) 大版本亮点：大量父目录 gitignore 规则 bug 修复；超大 gitignore 内存回归修复；`rg -vf file`（空文件）匹配一切；`-r/--replace` 兼容 `--json`；部分 Jujutsu(jj) 仓库按 git 仓库对待（尊重其 gitignore）；glob 支持嵌套花括号；Windows aarch64 构件；全量 LTO 编译
  - 再往前一个版本为 14.1.1 (2024-09-08)，14→15 跨了一个大版本

## 许可证
- URL: https://github.com/BurntSushi/ripgrep/blob/master/FAQ.md#license
- 抓取日期: 2026-09-20
- 可用于章节: §1 简介与定位; §7 资源链接
- 关键摘录:
  - "ripgrep is dual licensed under the Unlicense and MIT licenses."
  - 依赖约束："ripgrep will never depend on code that is not permissively licensed"（明确拒绝 GPL/LGPL/MPL/CC-SA 等 copyleft 依赖）

## 官方资源链接汇总
- URL: https://github.com/BurntSushi/ripgrep
- 抓取日期: 2026-09-20
- 可用于章节: §7 资源链接
- 关键摘录:
  - 仓库/文档主体：https://github.com/BurntSushi/ripgrep（README / GUIDE.md / FAQ.md / CHANGELOG.md）
  - Releases（预编译二进制）：https://github.com/BurntSushi/ripgrep/releases
  - 正则语法：https://docs.rs/regex/1/regex/#syntax
  - 作者性能分析博客：https://blog.burntsushi.net/ripgrep/
  - ack 作者维护的工具特性对比表：https://beyondgrep.com/feature-comparison/
  - 非官方 playground/交互教程：https://codapi.org/ripgrep/ 与 https://codapi.org/try/ripgrep/
  - 非官方中文翻译（README 声明可能过时）：https://github.com/chinanf-boy/ripgrep-zh
  - 相关工具 delta（支持 `rg --json` 输出）：`rg --json pattern | delta`

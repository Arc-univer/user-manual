# ripgrep GitHub 一手资料（README / CHANGELOG / Issues）

来源仓库：https://github.com/BurntSushi/ripgrep
抓取日期：2026-09-20
抓取方式：raw.githubusercontent.com + GitHub REST API（按评论数排序筛选经典 issue）

---

## 项目定位与默认行为（README 开篇）
- URL: https://github.com/BurntSushi/ripgrep/blob/master/README.md
- 抓取日期: 2026-09-20
- 可用于章节: §1 简介与定位
- 关键摘录:
  - "ripgrep is a line-oriented search tool that recursively searches the current directory for a regex pattern. By default, ripgrep will respect gitignore rules and automatically skip hidden files/directories and binary files. (To disable all automatic filtering by default, use `rg -uuu`.)"
  - 二进制名是 `rg`；Windows/macOS/Linux 一等支持，每个 release 都有预编译二进制。
  - 徽章：GitHub Actions CI 状态、crates.io 版本、Repology 打包状态。
  - 许可证：MIT 或 UNLICENSE 双授权。
  - 官方文档导航：README 的 Installation、GUIDE.md（User Guide）、FAQ.md、正则语法指向 docs.rs/regex、配置文件在 GUIDE.md#configuration-file。

## 官方性能基准（与 grep/ag/git grep/ugrep 对比）
- URL: https://github.com/BurntSushi/ripgrep/blob/master/README.md
- 抓取日期: 2026-09-20
- 可用于章节: §1 简介与定位、§6 进阶技巧
- 关键摘录:
  - Linux 内核源码树搜 `[A-Z]+_SUSPEND`（i9-12900K）：`rg -n -w` 0.082s（1.00x），git grep -P 0.273s，ag 0.443s，ack 2.935s（35.94x）。
  - 白名单对比（同等工作量）：`rg -uuu -tc` 0.063s，GNU grep `grep -E -r -n --include=*.c --include=*.h` 0.674s（10.69x）。
  - 单大文件 13GB：`rg -w 'Sherlock [A-Z]\w+'` 1.042s vs GNU grep(Unicode) 6.577s；加 `-n` 后 rg 1.664s、grep 9.484s。
  - 无字面量可优化的模式（`[A-Za-z]{30}`）：rg 15.569s，GNU grep(Unicode) 8m30s（32.74x）——rg 也有性能悬崖但远小于 grep。
  - 高命中数（`rg the`，8349 万命中）会拉平工具差距：rg 6.948s vs grep 15.217s。
  - 官方提醒：单一基准不足为凭，详见作者博客 https://blog.burntsushi.net/ripgrep/ 。

## 为什么快 / 为什么用 / 为什么不用（README 三个小节）
- URL: https://github.com/BurntSushi/ripgrep/blob/master/README.md
- 抓取日期: 2026-09-20
- 可用于章节: §1 简介与定位、§6 进阶技巧
- 关键摘录:
  - 快的原因：Rust regex 引擎（有限自动机 + SIMD + 激进字面量优化）；UTF-8 解码内建进 DFA，Unicode 常开不掉速；内存映射（单文件）与增量缓冲（大目录）自动选择；ignore 模式用 RegexSet 一次匹配多个 glob；crossbeam+ignore 的无锁并行目录迭代器。
  - 特性清单：默认递归 + 自动过滤（.gitignore/.ignore/.rgignore、隐藏文件、二进制文件，`-uuu` 全关）；文件类型过滤 `rg -tpy foo` / `rg -Tjs foo`；PCRE2 可选（`-P/--pcre2`、`--auto-hybrid-regex`、`--engine default|pcre2|auto`）；`-r/--replace` 替换输出；`-E/--encoding` 支持 UTF-16、latin-1、GBK、EUC-JP、Shift_JIS 等；`-z/--search-zip` 搜压缩文件（brotli/bzip2/gzip/lz4/lzma/xz/zstd）；预处理器；配置文件。
  - 不该用的场景：需要 POSIX 可移植/无处不在时用回 grep；依赖 rg 没有的特性或 bug；个别性能边角；平台装不上。
  - 非官方在线试用：https://codapi.org/ripgrep/ 与交互教程 https://codapi.org/try/ripgrep/ 。
  - 相关工具：delta 分页器支持 `rg --json pattern | delta`。

## 安装命令全平台清单（README Installation 节，本 agent 重点）
- URL: https://github.com/BurntSushi/ripgrep/blob/master/README.md#installation
- 抓取日期: 2026-09-20
- 可用于章节: §2 安装与配置
- 关键摘录:
  - 二进制名 `rg`；GitHub Releases 有 Windows/macOS/Linux 预编译包，Linux/Windows 为静态可执行文件。
  - macOS Homebrew / Linuxbrew: `brew install ripgrep`
  - MacPorts: `sudo port install ripgrep`
  - Windows Chocolatey: `choco install ripgrep`
  - Windows Scoop: `scoop install ripgrep`
  - Windows Winget: `winget install BurntSushi.ripgrep.MSVC`
  - Arch Linux: `sudo pacman -S ripgrep`
  - Gentoo: `sudo emerge sys-apps/ripgrep`
  - Fedora: `sudo dnf install ripgrep`
  - openSUSE（Tumbleweed 及 Leap≥15.1）: `sudo zypper install ripgrep`
  - CentOS Stream 10 / RHEL 10 / Rocky 10 走 EPEL：`dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-10.noarch.rpm && sudo dnf install ripgrep`（RHEL 需先 `subscription-manager repos --enable codeready-builder-for-rhel-10-$(arch)-rpms`，CentOS 需 `dnf config-manager --set-enabled crb`）
  - Nix: `nix-env --install ripgrep`；Flox: `flox install ripgrep`；Guix: `guix install ripgrep`
  - Debian/Ubuntu 装 release 的 .deb：`curl -LO https://github.com/BurntSushi/ripgrep/releases/download/14.1.1/ripgrep_14.1.1-1_amd64.deb && sudo dpkg -i ripgrep_14.1.1-1_amd64.deb`（README 示例版本号偏旧，应替换为最新 release）
  - Debian stable / Ubuntu ≥18.10: `sudo apt-get install ripgrep`（版本可能落后）
  - ALT Linux: `apt-get install ripgrep`；FreeBSD: `pkg install ripgrep`；OpenBSD: `doas pkg_add ripgrep`；NetBSD: `pkgin install ripgrep`；Haiku: `pkgman install ripgrep`；Void: `xbps-install -Syv ripgrep`
  - Rust 用户：`cargo install ripgrep` 或 `cargo binstall ripgrep`。最低 Rust 版本 **1.96.0**；二进制带调试符号是刻意的，想瘦身对二进制跑 `strip`。
  - 原文警告："Various snaps for ripgrep on Ubuntu are also available, but none of them seem to work right... it is no longer a recommended installation option."（snap 包有怪 bug，官方不推荐）

## 从源码构建与 PCRE2/MUSL
- URL: https://github.com/BurntSushi/ripgrep/blob/master/README.md#building
- 抓取日期: 2026-09-20
- 可用于章节: §2 安装与配置、§6 进阶技巧
- 关键摘录:
  - `git clone https://github.com/BurntSushi/ripgrep && cd ripgrep && cargo build --release`，需 Rust 1.96.0+，跟踪最新 stable。
  - PCRE2 支持：`cargo build --release --features 'pcre2'`；优先用系统 PCRE2（pkg-config），找不到则从源码静态编译；用 MUSL target 或设 `PCRE2_SYS_STATIC=1` 强制静态链接。
  - MUSL 全静态：`rustup target add x86_64-unknown-linux-musl && cargo build --release --target x86_64-unknown-linux-musl`（要 PCRE2 需另装 musl-gcc）。
  - 注意：`simd-accel` Cargo feature 已移除（曾只为 UTF-16 转码服务，依赖 nightly 易坏）。
  - 测试：`cargo test --all`。

## CHANGELOG：当前版本与近期大版本
- URL: https://github.com/BurntSushi/ripgrep/blob/master/CHANGELOG.md
- 抓取日期: 2026-09-20
- 可用于章节: §1 简介与定位、§2 安装与配置
- 关键摘录:
  - **当前最新版本：15.2.0（2026-07-15）**——修 gitignore 匹配若干 bug、目录树遍历性能改进（PERF #3293）；新增 aarch64-unknown-linux-musl release 二进制；尊重 `GIT_CONFIG_GLOBAL`/`GIT_CONFIG_SYSTEM`（FEATURE #3275）。
  - 15.1.0（2025-10-22）：修 15.0.0 引入的 `--line-buffered` 回归（影响 `tail -f` 场景，BUG #3194）；新增 Cursor 终端超链接别名。
  - 15.0.0（2025-10-15）大版本亮点：修复多个 gitignore 匹配 bug（含"父目录 gitignore 规则"这一高频报告 bug，关联 #829/#2731/#2747 等 8 个 issue）；修大 gitignore 文件内存回归；`rg -vf file`（file 为空）现匹配一切；`-r/--replace` 兼容 `--json`；部分 Jujutsu(jj) 仓库按 git 仓库对待（尊重 jj 的 gitignore）；glob 支持嵌套花括号；Windows aarch64 有 release 产物、powerpc64 停止出包；全 LTO 编译；忽略 .gitignore 开头的 UTF-8 BOM（#2177）。
  - 14.1.1（2024-09-08）：修一个会导致漏报匹配（false negative）的 inner literal 优化 bug（如 `(?i:e.x|ex)` 不匹配 `e-x`，#2884）；移除 simd-accel feature。
  - 14.0.0（2023-11-26）：超链接支持（`--hyperlink-format default|vscode`）；regex 引擎重写；14.0.1 修 Windows 上 `cargo install ripgrep`。

## 经典坑 #829：显式传路径时父目录 gitignore 不生效（已在 15.0.0 修复）
- URL: https://github.com/BurntSushi/ripgrep/issues/829
- 抓取日期: 2026-09-20
- 可用于章节: §5 常见问题与坑
- 关键摘录:
  - 现象：在 git 仓库根目录 `rg Sample a`（a 为路径参数）会搜到 `.gitignore` 里 `/a/b/` 忽略的文件；`cd a && rg Sample` 则正确排除。
  - 原因：man page 说 "Paths specified explicitly on the command line override glob and ignore rules"，历史实现对显式路径不向上应用父目录 gitignore；`--ignore-vcs` 也救不回来。
  - 解法：升级到 ripgrep 15.0.0（CHANGELOG："Fix bug related to gitignores from parent directories"，关联 #829/#2731/#2747/#2770/#2778/#2836/#2933/#3067 共 8 个 issue）；旧版本变通是 cd 进目录再搜。

## 经典坑 #342：PowerShell 默认配色下输出"看不见"
- URL: https://github.com/BurntSushi/ripgrep/issues/342
- 抓取日期: 2026-09-20
- 可用于章节: §5 常见问题与坑
- 关键摘录:
  - 现象：PowerShell 默认蓝底下，rg 文件名颜色显示为蓝/深色系，与背景混在一起"不可见"；cmd 里正常。
  - 原因：rg 默认配色固定为 `path:fg:magenta`、`line:fg:green`、`match:fg:red + bold`，是旧 PowerShell 控制台调色板渲染/对比度问题，rg 并没有动态换色。
  - 解法：用 `--colors` 自定义（如 `--colors 'path:fg:cyan'`），颜色限 red/blue/green/cyan/magenta/yellow/white/black 八色，样式 nobold/bold/nointense/intense；或换 Windows Terminal。

## 经典坑 #234：Windows 下 `rg PATTERN *.txt` 报 os error 123（shell 不展开 glob）
- URL: https://github.com/BurntSushi/ripgrep/issues/234
- 抓取日期: 2026-09-20
- 可用于章节: §5 常见问题与坑
- 关键摘录:
  - 现象：cmd/PowerShell 里 `rg PATTERN *.txt` 报 `*.txt: The filename, directory name, or volume label syntax is incorrect. (os error 123)` 和 "No files were searched... Try running again with --debug"。
  - 原因：Windows 的 cmd 不像 Unix shell 那样展开通配符，展开责任在程序自身；rg 历史上不展开命令行 glob（该 issue 至今 open）。
  - 解法：用 rg 自带的 `-g`/`--iglob`（如 `rg PATTERN -g '*.txt'`）；或在 git-bash/WSL 里跑；显式目录参数代替通配符。

## 经典坑 #1335：「ripgrep 比 grep 慢」的性能误区与排查清单
- URL: https://github.com/BurntSushi/ripgrep/issues/1335
- 抓取日期: 2026-09-20
- 可用于章节: §5 常见问题与坑、§6 进阶技巧
- 关键摘录:
  - 现象：用户报告同一目录 `grep -r` 0.298s 而 `rg --no-ignore` 1.129s，觉得 rg 慢 3-5 倍。
  - 原因/排查（作者 BurntSushi 原话要点）：单基准不足为凭；`rg -uuu -j1` 才是与 `grep -r` 的"apples to apples"对比（-uuu 关闭 ignore/hidden/binary 全部自动过滤，-j1 单线程对齐 grep）；检查是否有 rg 配置文件偷偷加了 flag；老 CPU 无 AVX 会慢；目录树里大量 .ignore/.gitignore、目录数量、报错文件多（syscall 表异常）都会影响；用 `--debug`/`--trace` 自查。
  - 解法：对比前先对齐工作量（过滤规则、线程数），用 `rg -uuu -j1` 做基准；怀疑异常时跑 `--debug`。

## 经典坑 #306：二进制文件被静默跳过（与 grep 行为差异）
- URL: https://github.com/BurntSushi/ripgrep/issues/306
- 抓取日期: 2026-09-20
- 可用于章节: §5 常见问题与坑
- 关键摘录:
  - 现象：搜含不可打印字符的文件时 rg 直接"无匹配"退出；GNU grep 会提示 `Binary file <filename> matches`。
  - 原因：rg 默认跳过二进制文件且不声不响；作者认可"至少单文件场景应提示"。
  - 解法：`-a/--text` 强制按文本处理；`--binary` 显式搜二进制（匹配后打印 binary matches 提示并停于该文件首个匹配）；`rg -uuu` 全量关闭过滤；相关 issue #2246 要求被跳过时打日志、#855 讨论聚合视图（--count 等）下是否抑制二进制检测。
  - 附带：#993 提供 `--null-data` 以 NUL 为行分隔读大二进制/数据文件。

## 经典坑 #2510：Windows PowerShell 管道后中文乱码（测试→娴嬭瘯）
- URL: https://github.com/BurntSushi/ripgrep/issues/2510
- 抓取日期: 2026-09-20
- 可用于章节: §5 常见问题与坑
- 关键摘录:
  - 现象：`echo "测试Chinese Test" | rg -o "测试"` 直接输出正常；一旦管道给别的程序或 `>` 重定向到文件，中文变成 `娴嬭瘯`（UTF-8 字节被按 GBK/ANSI 解读的典型乱码）。
  - 原因：Windows 控制台/管道默认代码页不是 UTF-8，下游程序（含 PowerShell 的 echo、cat）按系统 ANSI 码页读 rg 的 UTF-8 输出。
  - 解法：提问者按 FAQ.md 指引（L810 附近，PowerShell 编码一节）设置后解决——即把 `[Console]::OutputEncoding` / `$OutputEncoding` 设为 UTF-8（或 `chcp 65001`）；非 rg 本身 bug。（FAQ 细节归另一 agent，这里只留指针。）

## 其他高频 issue 速览（按评论数排序，GitHub API）
- URL: https://github.com/BurntSushi/ripgrep/issues?q=is%3Aissue+sort%3Acomments-desc
- 抓取日期: 2026-09-20
- 可用于章节: §4 核心功能、§5 常见问题与坑、§7 资源链接
- 关键摘录（issue 号 | 评论数 | 主题，均为仓库内经典讨论）:
  - #176 | 103 | 跨行搜索（multiline，已实现 `-U/--multiline`）
  - #196 | 91 | 持久配置文件 `.rgrc`（已实现，RIPGREP_CONFIG_PATH）
  - #875 | 69 | 更复杂的布尔匹配（开放中的 RFC）
  - #86 | 64 | 调分页器显示结果
  - #91 | 63 | 按文件名搜索 `-g`（ag 的 -g 语义迁移）
  - #129 | 56 | 超长行刷屏 → 已实现 `--max-columns` 截断
  - #411 | 53 | 输出总命中数（`--count-matches` 等）
  - #665 | 51 | 文件路径输出为 file:// URL（超链接，14.0 落地）
  - #1497 | 48 | ngram 索引 RFC（open）
  - #152 | 45 | 保持输入文件输出顺序而不牺牲并行（`--sort path`）
  - #373 | 24 | `.gitignore` 里 `**` 写法非法（"invalid use of **; must be one path component"）——glob 语义坑
  - #87 | 22 | gitignore 中一个非法 pattern 导致该文件全部 pattern 失效
  - #1109 | 11 | 非 git 仓库目录里 .gitignore 不被尊重（与 #1414 "加开关在非 git 仓库也读 .gitignore" 同主题；注意 rg 会读 `.ignore`/`.rgignore` 与 git 全局 excludesfile）
  - #200 | 33 | 管道下游关闭后 rg 不停止（`rg ... | head -n1` 仍跑完全程；SIGPIPE 处理演进史）
  - #269 | 30 | cygwin 下显式路径无法搜索
  - #1 | 30 | 支持其他文本编码（UTF-16 等，`-E/--encoding` 由此而来）
  - #1540 | 14 | open：希望尊重 .gitattributes 里标记为 binary 的文件
  - #3494 | 12 | open：x86_64-unknown-linux-musl 二进制在超大搜索中偶发 segfault（用 musl 静态版的注意事项）

## 杂项：翻译与漏洞上报
- URL: https://github.com/BurntSushi/ripgrep/blob/master/README.md#translations
- 抓取日期: 2026-09-20
- 可用于章节: §7 资源链接
- 关键摘录:
  - 非官方中文文档翻译：https://github.com/chinanf-boy/ripgrep-zh （README 注明非官方维护、可能滞后）
  - 西班牙语翻译：https://github.com/UltiRequiem/traducciones/tree/master/ripgrep
  - 安全漏洞上报走作者联系页（https://blog.burntsushi.net/about/，支持 PGP 加密邮件），不开公开 issue。

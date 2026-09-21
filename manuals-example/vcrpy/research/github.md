# vcrpy GitHub 一手素材

来源：github.com/kevin1024/vcrpy（README / issues / changelog）。抓取日期均为 2026-09-20。

## README：定位与工作原理（Rationale）
- URL: https://raw.githubusercontent.com/kevin1024/vcrpy/master/README.rst
- 抓取日期: 2026-09-20
- 可用于章节: §1 简介与定位 / §3 快速上手
- 关键摘录:
  - "This is a Python version of Ruby's VCR library"（移植自 Ruby VCR）
  - "VCR.py simplifies and speeds up tests that make HTTP requests. The first time you run code that is inside a VCR.py context manager or decorated function, VCR.py records all HTTP interactions ... and serializes and writes them to a flat file (in yaml format by default). This flat file is called a cassette."
  - 回放时 "intercept any HTTP requests that it recognizes from the original test run and return the responses"——不产生真实 HTTP 流量。
  - 三大收益原文："The ability to work offline / Completely deterministic tests / Increased test execution speed"
  - 更新 cassette 的官方姿势："If the server you are testing against ever changes its API, all you need to do is delete your existing cassette files, and run your tests again."（删文件重录，而非增量更新）
  - 文档站点 https://vcrpy.readthedocs.io/

## README：pytest 生态入口
- URL: https://raw.githubusercontent.com/kevin1024/vcrpy/master/README.rst
- 抓取日期: 2026-09-20
- 可用于章节: §2 安装与配置 / §7 资源链接
- 关键摘录:
  - "There is a library to provide some pytest fixtures called pytest-recording https://github.com/kiwicom/pytest-recording"
  - changelog 8.2.1 补充："Recommend pytest-recording over the unmaintained pytest-vcr in the docs (#986)"——pytest-vcr 已不维护，官方推荐 pytest-recording。

## 当前版本与近期重要变更（CHANGELOG）
- URL: https://raw.githubusercontent.com/kevin1024/vcrpy/master/docs/changelog.rst
- 抓取日期: 2026-09-20
- 可用于章节: §2 安装与配置 / §6 进阶技巧
- 关键摘录:
  - 当前最新版本 **8.3.0**：新增 niquests 支持（#980）；录制时拒绝写入 safe YAML loader 读不回的 Python 对象（fail fast，#1007/#1009）；新增 `vcr.serializers.yamlserializer.with_custom_tags` 支持自定义 YAML tag；修复跨 cassette 的 keep-alive 连接复用（#1001）。
  - **8.2.1 安全修复**："SECURITY: Load cassettes with a safe YAML loader, preventing arbitrary code execution when a cassette from an untrusted source is loaded (GHSA-rpj2-4hq8-938g)"；同时 "Validate record_mode and raise a clear error on an invalid value (#208)"。
  - **8.2.0**：httpx 2.x 支持；改为 patch httpx transports 而非 httpcore（#972）；aiohttp 3.14 兼容；`drop_unused_requests` 尊重 `before_record_request` 过滤（#962）。
  - **8.0.0 破坏性变更**："BREAKING: Drop support for Python 3.9"；"BREAKING: Drop support for urllib3 < 2 - fixes CVE warnings from urllib3 1.x"；新增 `drop_unused_requests` 选项（#763）；"Rewrite httpx support to patch httpcore instead of httpx (#943)"，修掉 `httpx.ResponseNotRead`（#832/#834）与 `KeyError: 'follow_redirects'`（#945）；"Fix HTTPS proxy handling - proxy address no longer ends up in cassette URIs (#809, #914)"。
  - **6.0.0 破坏性变更**：httpx 二进制 body 格式修错，"You may have to recreate some of your cassettes produced in previous releases"；砍掉 boto（保留 boto3）；drop simplejson。
  - 8.1.0：brotli 解压支持（`brotli`/`brotlipy`/`brotlicffi` 可选依赖，#620）。
  - 7.0.0 起：支持 Python 3.10+（8.0.0 起要求 3.10+，因 7.0.0 drop 3.8、8.0.0 drop 3.9）。

## 经典坑：record_mode 语义与 CannotOverwriteExistingCassetteException
- URL: https://github.com/kevin1024/vcrpy/issues/533 （21 评论，closed）；https://github.com/kevin1024/vcrpy/issues/208 （31 评论，open）
- 抓取日期: 2026-09-20
- 可用于章节: §5 常见问题与坑 / §4 核心功能
- 关键摘录:
  - 现象（#533 原帖）：用 `before_record_response` 把响应 body 里的 access_token 改成 "REDACTED" 后，"It runs successfully the first run ... but then on subsequent runs ... `vcr.errors.CannotOverwriteExistingCassetteException: Can't overwrite existing cassette ... in your current record mode ('none')`"。
  - 原因：回放时请求/响应与 cassette 记录对不上（body 被改写、Content-Length 变化、matcher 不匹配），record_mode 默认为 'none'（或 once 且 cassette 已存在），vcrpy 不允许覆写，抛此异常。报错信息具有迷惑性——真因是匹配失败而非"覆写"。
  - 解法：scrub 时同步修正 `response["headers"]["Content-Length"]`；确认 match_on 配置与实际请求一致；需要重录时删 cassette 或用 record_mode='all'/'new_episodes'。
  - #208（31 评论，open）：社区长期抱怨 record_mode 语义（once/all/none/new_episodes 与 Ruby VCR 的微妙差异），要求 'rerecord' 模式；8.2.1 的缓解措施是 "Validate record_mode and raise a clear error on an invalid value"。

## 经典坑：敏感数据过滤（filter_headers 系列）
- URL: https://github.com/kevin1024/vcrpy/issues/527 （open）; /issues/569 （open）; /issues/815 （open）; /issues/520 （open）; /issues/132 （closed）
- 抓取日期: 2026-09-20
- 可用于章节: §5 常见问题与坑 / §6 进阶技巧
- 关键摘录:
  - #527 "Insufficient tooling for filtering sensitive requests"（open）：过滤敏感数据的官方手段（filter_headers/filter_query_parameters/filter_post_data_parameters/before_record_request/before_record_response）组合仍不够顺手。
  - #569 "filterheaders doesn't work for cookie / set-cookie"（open）：filter_headers 对 cookie/set-cookie 不生效——多值 header 场景。
  - #815 / #520：社区要求 filter_headers 支持 glob 与 `re.compile()` 正则（均 open，尚未支持）。
  - #132（closed）：早在 2014 年就有 "Add credential scrubbing funtionality to vcrpy?" 请求，结论是用 before_record_* 钩子自行实现。
  - #937 "Feature recursive filter request body"（open）：嵌套 JSON body 的递归过滤不支持。
  - 教训汇总：Authorization/cookie 泄露风险高，filter_headers 有盲区，重要密钥建议 before_record_request/response 自定义钩子兜底。

## 经典坑：代理与 SSL/HTTPS
- URL: https://github.com/kevin1024/vcrpy/issues/214 （34 评论，closed）; /issues/515 （5 评论，open）
- 抓取日期: 2026-09-20
- 可用于章节: §5 常见问题与坑
- 关键摘录:
  - #214（并列全仓评论最多，34）："fails when working through proxy (as instructed by http_proxy env var): invalid literal for int() with base 10: '3128http'"——环境变量 http_proxy 代理场景下直接崩溃。
  - 8.0.0 修复了相关一大类问题："Fix HTTPS proxy handling - proxy address no longer ends up in cassette URIs (#809, #914)"。
  - #515（open）："https request always triggers connect due to SSL verification for record_mode=new_episodes"——new_episodes 模式下即使 cassette 已有记录，SSL 校验仍触发真实连接，离线/CI 环境翻车。

## 经典坑：aiohttp / httpx 支持缺口
- URL: https://github.com/kevin1024/vcrpy/issues/968 （26 评论，closed）; /issues/635 （8 评论，open）; /issues/925 （6 评论，open）; /issues/656 （14 评论，closed）; /issues/463 （open）; /issues/938 （open）
- 抓取日期: 2026-09-20
- 可用于章节: §5 常见问题与坑 / §4 核心功能
- 关键摘录:
  - #968（26 评论）："Support for intercepting httpx custom transports (BaseTransport) such as httpx-curl-cffi?"——httpx 自定义 transport 拦截不到；8.2.0 起 "Patch httpx transports instead of httpcore (#972)" 解决。
  - #635（open）："aiohttp and raise_for_status, unexpected behaviours"——aiohttp 客户端用 raise_for_status 时行为异常。
  - #925（open）："fix(aiohttp): Record error responses when raise_for_status is used"——错误响应（4xx/5xx）在 raise_for_status 下录不进去。
  - #656（closed）："[BUG] HTTPX Binary upload data unsupported"；衍生 open PR #1021/#1050/#1017：过滤二进制 POST body 时 UnicodeDecodeError。
  - #625（open）："Gzip encoded responses with aiohttp and auto_decompress"——aiohttp 自动解压与 cassette 记录内容不一致。
  - #463（open）："Differences between requests and aiohttp cassettes"——同一接口用 requests 与 aiohttp 录出的 cassette 不通用。
  - #938（open）："Weird Behaviour when using OpenAI with Aiohttp"——OpenAI SDK + aiohttp 组合间歇异常。

## 经典坑：二进制 body 与编解码
- URL: https://github.com/kevin1024/vcrpy/issues/660 （open）; /issues/844 （10 评论，closed）; /issues/882 （closed）; /issues/1022 （open）
- 抓取日期: 2026-09-20
- 可用于章节: §5 常见问题与坑
- 关键摘录:
  - #660（open）："POSTing binary data as bytes or bytearray instead of BytesIO breaks assumptions baked into replace_post_data_parameters"——二进制 POST body 与 filter_post_data_parameters 冲突。
  - #844（10 评论，closed）："String decoding fails for POST requests with byte payload"。
  - #882（closed）："Handle HTTPX UTF-8 decoding errors"——httpx 响应非 UTF-8 时解码崩。
  - #1022/#1023（open）：cassette 文件存在但解码失败时静默异常，要求 "Log a warning"。
  - 历史教训（6.0.0 changelog）：httpx 二进制格式曾存错，官方要求重建旧 cassette。
  - 相关安全面：8.2.1 起 cassette 用 safe YAML loader，带自定义 Python tag 的旧 cassette 会加载失败；8.3.0 提供 `with_custom_tags` 逃生门。

## 其他高互动 issue（补充）
- URL: https://github.com/kevin1024/vcrpy/issues/131 （34 评论，closed）; /issues/645 （29 评论，closed）; /issues/849 （13 评论，open）; /issues/979 （5 评论，open）
- 抓取日期: 2026-09-20
- 可用于章节: §5 常见问题与坑
- 关键摘录:
  - #131（34 评论）："vcrpy doesn't work under django?"——Django 测试环境集成困惑的经典贴。
  - #645（29 评论）：vcrpy 自身测试套件对 httpbin.org/Werkzeug 的依赖问题（对使用者意义：CI 里别依赖公网 httpbin）。
  - #849（13 评论，open）："Multithreading issue with vcrpy - Inconsistent recording of requests"——多线程并发请求时录制不一致，vcrpy 非线程安全。
  - #979（open）："Wrong cassette loaded on test causing intermitent errors"—— cassette 路径/命名撞车导致测试间歇失败。

## 仓库元数据
- URL: https://api.github.com/repos/kevin1024/vcrpy
- 抓取日期: 2026-09-20
- 可用于章节: §1 简介与定位 / §7 资源链接
- 关键摘录:
  - 描述："Automatically mock your HTTP interactions to simplify and speed up testing"
  - 公开仓库、MIT License、owner kevin1024；非 fork、非 archived。

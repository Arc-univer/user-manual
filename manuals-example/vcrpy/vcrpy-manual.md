# vcrpy 使用说明书

> vcrpy（VCR.py）是 Ruby VCR 的 Python 移植：把测试中的 HTTP 交互录制成「磁带」（cassette）文件，后续运行直接回放，不再产生真实网络流量。
> 版本基准：**8.3.0**（以官方 CHANGELOG 为准，2026-09-20 抓取；文档站首页标注 8.0.0 系滞后）。

## 目录

- [[#1. 简介与定位]]
- [[#2. 安装与配置]]
- [[#3. 快速上手]]
- [[#4. 核心功能]]
- [[#5. 常见问题与坑]]
- [[#6. 进阶技巧]]
- [[#7. 资源链接]]

## 1. 简介与定位

vcrpy 拦截受支持 HTTP 库发出的请求：首次运行时把请求/响应序列化到 YAML 文件（cassette），之后运行同一测试时直接回放磁带中的响应。三大收益：**离线可跑、测试完全确定、执行提速**（来源：[官方文档](https://vcrpy.readthedocs.io/en/latest/)、[README](https://raw.githubusercontent.com/kevin1024/vcrpy/master/README.rst)，2026-09-20）。

**适合谁**：测试依赖第三方 HTTP API 的 Python 开发者。上游 API 变更后的官方更新姿势：删掉旧 cassette 重跑一遍即可重新录制。

**边界**：它是「自动 mock 层」而非「自动生成测试」——测试仍需你自己写（社区定位，来源：[HN](https://news.ycombinator.com/item?id=45890695)，2026-09-20）。与 Ruby VCR 的 cassette **不兼容**。许可证：MIT。

## 2. 安装与配置

```bash
pip3 install vcrpy
```

- 版本要求：Python **3.10+**（8.0.0 起放弃 3.9；同时要求 urllib3>=2，修掉 1.x 的 CVE 告警）（来源：[CHANGELOG](https://raw.githubusercontent.com/kevin1024/vcrpy/master/docs/changelog.rst)，2026-09-20）。
- 支持的 HTTP 库：requests(>=2.16.2)、httpx、aiohttp、urllib3、http.client、httplib2、tornado、boto3 等（完整清单见[安装页](https://vcrpy.readthedocs.io/en/latest/installation.html)，2026-09-20）。
- 可选提速：pyyaml 能用 libyaml 时 vcrpy 快约 10 倍。验证 `python3 -c 'from yaml import CLoader'`；没有则装系统库（Ubuntu `apt-get install libyaml-dev`）后 `pip3 --no-cache-dir install pyyaml` 重装（来源：[安装页](https://vcrpy.readthedocs.io/en/latest/installation.html)，2026-09-20）。

**全局配置**（可选）：用 `vcr.VCR(...)` 建实例统一配置，单次 `use_cassette` 的参数优先于全局配置：

```python
import vcr

my_vcr = vcr.VCR(
    serializer='json',
    cassette_library_dir='fixtures/cassettes',
    record_mode='once',
    match_on=['uri', 'method'],
)
```

（来源：[配置文档](https://vcrpy.readthedocs.io/en/latest/configuration.html)，2026-09-20）

## 3. 快速上手

最小可运行示例——上下文管理器包住发请求的代码， cassette 不存在就录制，存在就回放：

```python
import vcr
import urllib.request

with vcr.use_cassette('fixtures/vcr_cassettes/synopsis.yaml'):
    response = urllib.request.urlopen('http://www.iana.org/domains/reserved').read()
    assert b'Example domains' in response
```

测试函数更常用装饰器形式（可省略路径，cassette 按测试函数名自动命名）：

```python
@vcr.use_cassette('fixtures/vcr_cassettes/synopsis.yaml')
def test_iana():
    response = urllib.request.urlopen('http://www.iana.org/domains/reserved').read()
    assert b'Example domains' in response
```

pytest 用户直接用官方推荐的 [pytest-recording](https://github.com/kiwicom/pytest-recording) fixtures（pytest-vcr 已不维护）；unittest 用户继承 `vcr.unittest.VCRTestCase` 自动录制（自定义 `setUp` 必须调 `super().setUp()`）（来源：[usage](https://vcrpy.readthedocs.io/en/latest/usage.html)、[CHANGELOG 8.2.1](https://raw.githubusercontent.com/kevin1024/vcrpy/master/docs/changelog.rst)，2026-09-20）。

## 4. 核心功能

### Record Modes（录制模式）

| 模式 | 行为 |
|---|---|
| `once`（默认） | 无 cassette 则录制；有 cassette 则**禁止任何新请求**（新请求报错） |
| `new_episodes` | 回放已有交互，同时录制新交互 |
| `none` | 只回放；任何新请求报错——保证绝不发出真实 HTTP 请求 |
| `all` | 全部重录，从不回放（强制刷新磁带用） |

（来源：[usage](https://vcrpy.readthedocs.io/en/latest/usage.html)，2026-09-20）

### 请求匹配（match_on）

默认匹配器：`['method', 'scheme', 'host', 'port', 'path', 'query']`——同 URL 同方法才视为同一请求。可选匹配器还有 `body`（按 content-type 解析）、`raw_body`、`headers` 等；自定义 `match_on=['uri', 'method']` 是常见精简配置（来源：[配置文档](https://vcrpy.readthedocs.io/en/latest/configuration.html)，2026-09-20）。

### 敏感数据过滤（cassette 要进 git 的必读）

```python
vcr.VCR(filter_headers=[('authorization', 'XXXXXX')],      # 值可为：替换值 / None(删除) / callable
        filter_query_parameters=['api_key'],
        filter_post_data_parameters=['api_key'])
```

需要更强控制时用钩子：`before_record_request`（返回 None 则该请求不录制）、`before_record_response`（可改写响应体）（来源：[advanced](https://vcrpy.readthedocs.io/en/latest/advanced.html)，2026-09-20）。

### Cassette 对象：断言录制内容

```python
with vcr.use_cassette('test.yaml') as cass:
    ...
assert len(cass) == 1
assert cass.requests[0].uri == 'http://www.zombo.com/'
```

另有 `play_count`、`all_played`、`responses_of(request)` 等属性（来源：[advanced](https://vcrpy.readthedocs.io/en/latest/advanced.html)，2026-09-20）。

## 5. 常见问题与坑

**① `CannotOverwriteExistingCassetteException` 报错文案误导** —— 真因通常是**回放匹配失败**而非「覆写」：典型场景是 `before_record_response` 改了 body 但没同步 `Content-Length`。解法：scrub 时同步修正 header；报错信息本身会列出相似请求与各 matcher 成败，是第一排查入口；配合 `logging.getLogger("vcr").setLevel(logging.INFO)` 看录制/回放判定（来源：[issue #533](https://github.com/kevin1024/vcrpy/issues/533)、[debugging](https://vcrpy.readthedocs.io/en/latest/debugging.html)，2026-09-20）。

**② record_mode 语义反直觉** —— 31 评论长贴（#208，open）的社区长期抱怨，尤其 `once`/`new_episodes` 与 Ruby VCR 的差异。8.2.1 起会对非法值直接报错（来源：[issue #208](https://github.com/kevin1024/vcrpy/issues/208)、CHANGELOG，2026-09-20）。

**③ filter_headers 的盲区** —— 对 cookie/set-cookie 不生效（#569，open），不支持 glob/正则（#815/#520，open），嵌套 JSON body 无递归过滤（#937，open）。**重要密钥请用 before_record_* 钩子兜底，不要只信 filter_headers**（来源：对应 issues，2026-09-20）。

**④ `new_episodes` + SSL 会在离线 CI 翻车** —— 即使 cassette 已有记录，SSL 校验仍触发真实连接（#515，open）。CI 离线环境用 `once` 或 `none`（来源：[issue #515](https://github.com/kevin1024/vcrpy/issues/515)，2026-09-20）。

**⑤ aiohttp/httpx 支持缺口** —— aiohttp 用 `raise_for_status` 时 4xx/5xx 错误响应录不进 cassette（#925，open）；同一接口 requests 与 aiohttp 录出的 cassette 不通用（#463，open）；httpx 自定义 transport 拦截缺口已在 8.2.0 修复（来源：对应 issues 与 CHANGELOG，2026-09-20）。

**⑥ 回放测试发现不了上游 API 变更** —— 社区最常被引用的坑：磁带过期后测试照样绿。配套解法见 §6 双模式技巧（来源：[HN](https://news.ycombinator.com/item?id=26566903)，2026-09-20）。

**⑦ 动态数据破坏回放** —— 响应含当前时间、随机 ID、调用序列时，「录制即回放」不成立，需要自定义 matcher 或 before_record_request 归一化（来源：[Reddit](https://www.reddit.com/r/Python/comments/9zi4no/)，2026-09-20）。

**⑧ 多线程录制不一致** —— vcrpy 非线程安全（#849，open），并发请求场景慎用（来源：[issue #849](https://github.com/kevin1024/vcrpy/issues/849)，2026-09-20）。

**⑨ 安全提醒：8.2.1 起 cassette 用 safe YAML loader** —— 防恶意 cassette 任意代码执行（GHSA-rpj2-4hq8-938g）；带自定义 Python tag 的旧 cassette 会加载失败，可用 8.3.0 的 `with_custom_tags` 逃生门。**不要加载来源不可信的 cassette**（来源：CHANGELOG，2026-09-20）。

## 6. 进阶技巧

**双模式校验磁带假设**（社区推荐，治 §5-⑥）：同一测试跑两种模式——日常用 VCR 回放，CI 定期跑真实 API 版本，确认对外部 API 的假设仍成立（来源：[Reddit](https://www.reddit.com/r/Python/comments/9zi4no/)、[HN](https://news.ycombinator.com/item?id=40709699)，2026-09-20）。

** cassette 瘦身**：`drop_unused_requests=True` 保存时丢弃本次未用到的旧交互；`record_on_exception=False` 只在测试成功时保存。

**回放控制**：`allow_playback_repeats=True` 允许同一响应重复回放（循环调用场景）；`cass.rewind()` 同测试内倒带重放。

**响应可读化**：`decode_compressed_response=True` 录制前解压 gzip/deflate，cassette 可读可手改（被测库自己依赖解压行为时勿用）。

**扩展点**：`register_matcher`（自定义匹配，`fn(r1, r2)` 内用 assert 给清晰报错）、`register_serializer`、`register_persister`（换存储后端）、`custom_patches`。

**架构理解**（选型参考）：vcrpy 在 httplib/transport 层拦截，所以兼容几乎所有 HTTP 客户端，且客户端的解析代码（如 `.json()` 取值）仍在真实响应上运行——这是它比手写 stub（responses/requests-mock）覆盖更全的点（来源：[作者亲述](https://www.reddit.com/r/Python/comments/258m68/)、[HN 讨论](https://news.ycombinator.com/item?id=31836873)，2026-09-20）。

## 7. 资源链接

**官方**

- 文档站：https://vcrpy.readthedocs.io/
- 仓库与 CHANGELOG：https://github.com/kevin1024/vcrpy
- pytest 集成：https://github.com/kiwicom/pytest-recording
- 0.x → 1.x cassette 迁移：`python3 -m vcr.migration PATH`（先备份）

**本文引用来源**（均抓取于 2026-09-20）：GitHub issues [#208](https://github.com/kevin1024/vcrpy/issues/208) / [#463](https://github.com/kevin1024/vcrpy/issues/463) / [#515](https://github.com/kevin1024/vcrpy/issues/515) / [#533](https://github.com/kevin1024/vcrpy/issues/533) / [#569](https://github.com/kevin1024/vcrpy/issues/569) / [#815](https://github.com/kevin1024/vcrpy/issues/815) / [#849](https://github.com/kevin1024/vcrpy/issues/849) / [#925](https://github.com/kevin1024/vcrpy/issues/925) / [#937](https://github.com/kevin1024/vcrpy/issues/937)；HN [#26566903](https://news.ycombinator.com/item?id=26566903) / [#31836873](https://news.ycombinator.com/item?id=31836873) / [#40709699](https://news.ycombinator.com/item?id=40709699) / [#45890695](https://news.ycombinator.com/item?id=45890695)；Reddit [r/Python 2014](https://www.reddit.com/r/Python/comments/258m68/) / [r/Python 2018](https://www.reddit.com/r/Python/comments/9zi4no/)。

> 社区层说明：近 30 天（2026-08-21 起）Reddit/HN 无 vcrpy 有效讨论（last30days 引擎实测确认），§5/§6 社区素材均为历史内容并已逐条标注。调研素材留档于同目录 `research/`。

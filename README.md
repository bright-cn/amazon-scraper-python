# Amazon Python 爬虫工具

使用 Python 获取亚马逊商品、评论、卖家及搜索结果，并以 JSON 格式输出。无需登录，也无需运行浏览器。本项目基于 [Bright Data 亚马逊爬虫 API](https://brightdata.com/products/web-scraper/amazon?utm_source=github)，使用 [Bright Data Python SDK](https://github.com/brightdata/sdk-python)。

[快速开始](#快速开始) · [命令行工具](#命令行工具) · [其他-api-功能](#其他-api-功能) · [数据与注意事项](#数据与注意事项) · [故障排查](#故障排查) · [编程代理](#编程代理) · [API 文档](https://docs.brightdata.com/products/scrapers/amazon/introduction)

> 本译文说明如何使用仓库中的现有代码。命令、Python 标识符、环境变量、JSON 字段名及示例数据保持原样，以免影响运行。

## 快速开始

需要 Python 3.10 或更高版本。

```bash
pip install brightdata-sdk
export BRIGHTDATA_API_TOKEN=YOUR_API_KEY
```

在 [Bright Data 控制面板](https://brightdata.com/cp/setting/users)获取 API 令牌。如果虚拟环境位于项目内，也可以将令牌写入项目根目录的 `.env` 文件。

你也可以先运行一次 `npx -p @brightdata/cli bdata login`：它会打开浏览器完成登录，此后 SDK 可以使用已保存的凭据。编程代理无法代替你点击登录页面，请先自行完成登录。还没有账户？可以[创建账户](https://brightdata.com/cp/start)；新账户每月有 [5,000 个免费额度](https://docs.brightdata.com/general/account/billing-and-pricing/free-tier)。

```python
from brightdata import SyncBrightDataClient

with SyncBrightDataClient(auto_create_zones=False) as client:
    product = client.scrape.amazon.products(
        "https://www.amazon.com/dp/B0CRMZHDG8"
    ).data
    print(product["title"])
    print(product["final_price"], product["currency"])
```

每条商品记录消耗一个额度。使用此项目时，请传入 `auto_create_zones=False`：否则 SDK 启动时会尝试为网络解锁器和搜索引擎 API 创建本项目不需要的区域；没有付款方式的账户可能因此失败。

`products()` 接收的是**商品 URL**，不是单独的 ASIN。可将 ASIN 写成 `https://www.amazon.com/dp/ASIN`。

## 命令行工具

安装本仓库后，可一次获取多个商品，并将结果写入一个 JSON 文件：

```bash
pip install git+https://github.com/brightdata/amazon-scraper-python
amazon-scraper B0CRMZHDG8 B085DVHQ57
```

也可以运行 `python -m amazon_scraper`。`--out PATH` 用于指定输出文件，默认文件名为 `amazon.json`。

命令同时接受 ASIN 和包含 ASIN 的商品 URL。重复的 ASIN 只会抓取一次。所有商品组成同一个任务；API 按记录而非按任务计费。

需要在 Python 代码中逐个处理结果时，可导入 `scrape`。读取 `product` 前请先检查 `ok`：

```python
from amazon_scraper import scrape

for outcome in scrape(["B0CRMZHDG8", "B0ZZZZZZZZ"]):
    if outcome.ok:
        print(f"{outcome.asin}: {outcome.product['title']}")
    else:
        print(f"{outcome.asin} failed: {outcome.error}")
```

## 其他 API 功能

| 已知信息 | 所需数据 | SDK 调用 |
| --- | --- | --- |
| 商品 URL | 商品记录 | `client.scrape.amazon.products(url)` |
| 关键词 | 一页商品搜索结果 | `client.search.amazon.products(keyword=...)` |
| 卖家 URL | 卖家记录 | `client.scrape.amazon.sellers(url)` |
| 商品 URL | 商品评论 | `client.scrape.amazon.reviews(url)` |

Amazon 调用的 `timeout` 参数以**秒**为单位，默认值为 240。

### 评论抓取可能产生大量费用

评论按**每条评论**计费。当前 SDK 虽接受 `numOfReviews`、`pastDays` 和 `keyWord`，却不会将这些参数发送到请求中。因此，`reviews(url, numOfReviews=20)` **不会**将结果限制为 20 条；它可能请求该商品的所有评论。需要限制数量时，请使用支持 `max_reviews` 的 REST 端点。详见 [sdk-python#62](https://github.com/brightdata/sdk-python/issues/62)。

### 关键词搜索与商品发现并不相同

`client.search.amazon.products(keyword=...)` 返回的是商品**搜索结果**记录，每条约有 30 个字段，并非完整的商品记录。当前 SDK 调用既不提供限制返回行数的参数，也不会发送 `pages_to_search`。

API 还支持按关键词、类目 URL、畅销商品 URL 或 UPC 发现完整商品记录，但此 Python SDK 目前没有与这些功能对应的方法。

### 一次抓取多个商品

向 `products()` 传入 URL 列表，会创建**一个任务**：

```python
from brightdata import SyncBrightDataClient

with SyncBrightDataClient(auto_create_zones=False) as client:
    results = client.scrape.amazon.products([
        "https://www.amazon.com/dp/B0CRMZHDG8",
        "https://www.amazon.com/dp/B085DVHQ57",
    ])
    for result in results:
        row = result.data
        print(row["asin"], row["final_price"], row["currency"])
```

**请依据每条记录自身的 `asin` 判断它属于哪个商品，不要依赖 `result.url`。**API 返回记录的顺序可能与输入顺序不同，而 SDK 会按位置关联 URL 和记录；错误记录也可能被标记为 `success=True`。详见 [sdk-python#60](https://github.com/brightdata/sdk-python/issues/60)。

### 现在触发，稍后获取

如果要抓取的商品较多，可以先触发任务并保存快照 ID，待任务就绪后再获取结果。快照可下载 30 天：

```python
import time
from brightdata import SyncBrightDataClient

with SyncBrightDataClient(auto_create_zones=False) as client:
    job = client.scrape.amazon.products_trigger(
        "https://www.amazon.com/dp/B0CRMZHDG8"
    )
    print("snapshot:", job.snapshot_id)

    while (status := client.scrape.amazon.products_status(
        job.snapshot_id
    )) not in ("ready", "failed"):
        time.sleep(5)

    print("status:", status)
    record = client.scrape.amazon.products_fetch(job.snapshot_id)[0]
    print(record["asin"], record["title"])
```

## 数据与注意事项

常用商品字段包括：

```text
asin  title  brand  final_price  currency  rating  reviews_count  availability
```

本项目**没有写死字段列表**：API 返回的内容会进入 `result.data`，命令行工具也会将其写入 JSON 文件。完整字段表和真实输出示例见[英文 README](README.md)及 [`examples/sample_output.json`](examples/sample_output.json)。

API 元数据将 `seller_name`、`zipcode` 和 `coupon` 标记为个人数据（`pii: true`）。SDK 的 `get_metadata()` 会丢弃该标记；如果需要判断个人数据字段，应读取原始元数据端点。详见 [sdk-python#61](https://github.com/brightdata/sdk-python/issues/61)。

## 故障排查

| 现象 | 处理方式 |
| --- | --- |
| `API token required but not found.` | 设置 `BRIGHTDATA_API_TOKEN`，或先完成 CLI 登录。 |
| ASIN 对应的页面返回 404 | 检查 ASIN 是否输入错误。 |
| API 没有返回该 ASIN 的记录 | 重新运行任务。 |
| 任务在 240 秒后超时 | 重新运行，或适当提高调用的 `timeout`。 |
| `Failed to fetch results: Request timeout after 30 seconds` | 任务可能已完成，但下载超时。此限制来自客户端每次 HTTP 请求的超时设置；仅提高抓取调用的 `timeout` 无效。 |
| `AuthenticationError: Unauthorized (401)` | 检查 API 令牌是否正确。 |

命令行工具只要有一项商品抓取失败就会以状态码 1 退出，因此可以用其退出状态控制脚本流程。

## 编程代理

不使用 Python 时，也可通过 Bright Data CLI 获取商品数据：

```bash
npx -p @brightdata/cli bdata login
npx -p @brightdata/cli bdata pipelines amazon_product "https://www.amazon.com/dp/B0CRMZHDG8"
```

`bdata pipelines list` 可列出可用类型。亚马逊相关类型包括 `amazon_product`、`amazon_product_reviews` 和 `amazon_product_search`；它们按记录消耗额度。

托管环境中的代理还可以使用 [Bright Data MCP 服务器](https://github.com/brightdata/brightdata-mcp#which-tool-to-use)。亚马逊工具属于需要启用的 `ecommerce` 工具组：

```text
https://mcp.brightdata.com/mcp?token=YOUR_API_TOKEN&groups=ecommerce
```

## 支持与许可证

本仓库的问题请[提交 issue](https://github.com/brightdata/amazon-scraper-python/issues)。有关 API、账户或额度的问题，请联系 [Bright Data 支持团队](https://brightdata.zendesk.com/hc/en-us/requests/new)。

许可证：MIT。

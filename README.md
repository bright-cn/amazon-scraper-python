[![使用 Amazon 爬虫 API 抓取商品、评论、卖家及搜索数据。按 ASIN、关键词或 UPC 采集和发现商品。免费开始。](.github/banner.png)](https://brightdata.com/products/web-scraper/amazon?utm_source=github)

# amazon-scraper-python

[![实时检查](https://github.com/brightdata/amazon-scraper-python/actions/workflows/live.yml/badge.svg)](https://github.com/brightdata/amazon-scraper-python/actions/workflows/live.yml)
[![最近验证时间](https://img.shields.io/badge/last%20verified-5%20Oct%202026-brightgreen)](https://github.com/brightdata/amazon-scraper-python/actions/workflows/live.yml) <!-- verified: rewritten by the daily run -->

[快速开始](#quickstart) · [命令行](#or-run-it-as-a-command) · [API 接口](#the-rest-of-the-api) · [数据](#the-data) · [错误处理](#when-it-fails) · [编程智能体](#coding-agents) · [文档](https://docs.brightdata.com/products/scrapers/amazon/introduction) · [支持](#support)

使用 Python 获取 JSON 格式的 Amazon 商品、评论、卖家及搜索结果。无需登录，也无需浏览器。本项目基于 [Bright Data Amazon 爬虫 API](https://brightdata.com/products/web-scraper/amazon?utm_source=github)。

项目使用 [Bright Data Python SDK](https://github.com/brightdata/sdk-python)。完整 API 文档：[Amazon 爬虫 API](https://docs.brightdata.com/products/scrapers/amazon/introduction)。

项目还提供一条命令即可运行的商品抓取工具，以及完全不需要 Python 的 [Bright Data CLI](#coding-agents)。

<a id="quickstart"></a>
## 快速开始

需要 Python 3.10 或更高版本。

```bash
pip install brightdata-sdk
export BRIGHTDATA_API_TOKEN=YOUR_API_KEY
```

前往 [Bright Data 控制面板](https://brightdata.com/cp/setting/users)获取令牌。也可以把令牌放在项目根目录的 `.env` 文件中，但虚拟环境必须位于该项目内。

你也可以不手动设置令牌：运行一次 `npx -p @brightdata/cli bdata login`。它会打开浏览器；登录后，SDK 会自行找到保存的凭据，供你及在该终端中工作的编程智能体使用。智能体无法自行点击完成登录，因此请先亲自登录。

还没有账户？[创建账户](https://brightdata.com/cp/start)。新账户[每月可获 5,000 免费额度](https://docs.brightdata.com/general/account/billing-and-pricing/free-tier)。

```python
from brightdata import SyncBrightDataClient

with SyncBrightDataClient(auto_create_zones=False) as client:
    product = client.scrape.amazon.products("https://www.amazon.com/dp/B0CRMZHDG8").data
    print(product["title"])
    print(product["final_price"], product["currency"], "|", product["rating"], "stars |",
          product["reviews_count"], "reviews")
```

```
STANLEY Quencher H2.0 Flow State Tumbler, 40 oz, Fuchsia
39.95 USD | 4.7 stars | 205036 reviews
```

每件商品消耗[一个额度](https://brightdata.com/pricing/web-scraper)；API 给出的预计耗时约为每条输入 7 秒。

每次都要传入 `auto_create_zones=False`。否则，SDK 会在启动时为网络解锁器和搜索引擎 API 创建 zone。这是此爬虫工具不会用到的另外两款 Bright Data 产品；没有付款方式的账户会在创建 zone 时失败（[sdk-python#57](https://github.com/brightdata/sdk-python/issues/57)）。

`products` 接收 URL，而不是 ASIN。因此，请像上面那样将 ASIN 写成 `/dp/` URL。

<a id="or-run-it-as-a-command"></a>
## 或者通过命令行运行

本仓库的命令行工具可以对多个 ASIN 执行相同的抓取操作，并将结果写入一个 JSON 文件。

```bash
pip install git+https://github.com/brightdata/amazon-scraper-python
amazon-scraper B0CRMZHDG8 B085DVHQ57
```

```
Fetching 2 Amazon products: B0CRMZHDG8, B085DVHQ57
One job for all of them. One credit per product.
asking  2 products...

got     B0CRMZHDG8: 100 fields (STANLEY Quencher H2.0 Flow State Tumbler, 40 oz,)
got     B085DVHQ57: 99 fields (Owala FreeSip Stainless Steel Water Bottle 32 oz)

Saved 2 of 2 products as JSON to amazon.json
```

既可以传入 ASIN，也可以传入包含该 ASIN 的商品 URL。同一 ASIN 即使传入两次，也只会抓取一次，因此重复输入不会产生额外费用。

在终端中，`asking` 行会被以下内容原位替换并持续更新，让你知道工具仍在运行，以及已经运行了多久：

```
⠹ 2 products 0:00:14
```

```
--out PATH   输出文件，默认为 amazon.json
```

也可以运行 `python -m amazon_scraper`。

你传入的所有 ASIN 都会放进同一个任务。API 按记录而非按任务计费，因此抓取十件商品与抓取一件商品的等待方式相同。

你还可以导入此工具，而不通过命令行运行。这样会得到每个 ASIN 对应的 `ok` 和 `error`，而不是原始数据行。单个 ASIN 无效时，`scrape` 不会抛出异常；读取 `product` 前请先检查 `ok`：

```python
from amazon_scraper import scrape

for outcome in scrape(["B0CRMZHDG8", "B0ZZZZZZZZ"]):
    if outcome.ok:
        print(f"{outcome.asin}: {outcome.product['title']}")
    else:
        print(f"{outcome.asin} failed: {outcome.error}")
```

```
B0CRMZHDG8: STANLEY Quencher H2.0 Flow State Tumbler, 40 oz, Fuchsia
B0ZZZZZZZZ failed: The navigation resulted in a dead page (404 status code)
```

<a id="the-rest-of-the-api"></a>
## 其他 API 接口

上述命令行工具对应下表第一行。这里的每段代码都是完整示例，只需安装 `brightdata-sdk` 即可直接运行。所有示例每周一都会在 Actions 中运行；规模较小的检查每隔一天运行一次。页面顶部的徽章显示最近一次结果。

| 你已有 | 你想获取 | 调用方式 |
| --- | --- | --- |
| 商品 URL | 该商品 | `client.scrape.amazon.products(url)` |
| 关键词 | 一页搜索结果 | `client.search.amazon.products(keyword=...)`；参见下文说明 |
| 卖家 URL | 该卖家 | `client.scrape.amazon.sellers(url)` |
| 商品 URL | 该商品的评论 | `client.scrape.amazon.reviews(url)`；每条评论消耗一个额度，另见下文 |
| 关键词、分类 URL、畅销商品 URL 或 UPC | 完整商品记录 | Python SDK 中没有对应方法；参见下文 |

Amazon 接口的 `timeout` 参数以秒为单位，默认值为 240。

### 此处不运行评论抓取，原因是费用

每条评论消耗一个额度。`reviews()` 虽然接受 `numOfReviews`、`pastDays` 和 `keyWord`，却不会发送这些参数：请求仅根据 URL 构建（[sdk-python#62](https://github.com/brightdata/sdk-python/issues/62)）。因此，`reviews(url, numOfReviews=20)` 请求的是所有评论，而且不会发出警告。

API 支持通过 `max_reviews` 限制评论数量，Bright Data 官方文档的示例也将其设为 20。但 SDK 不会把 `numOfReviews` 映射到这个参数。

快速开始示例中的商品显示有 205,036 条评论，而这只需要一次调用。在 SDK 能发送数量限制参数之前，如果只需获取有限数量的评论，请直接调用 REST 接口。

### 关键词搜索：它能做什么，不能做什么

`client.search.amazon.products(keyword=...)` 返回商品搜索数据集中的数据行，每条搜索结果对应一行，每行约有 30 个字段。它不同于 Node.js 版本的关键词商品发现功能；后者返回包含全部 119 个字段的商品记录。

该方法没有限制返回行数的参数，也不会发送 `pages_to_search`。本项目不运行这一示例，因为 README 中的每个代码示例都会每周重新运行。

API 还支持通过关键词、分类 URL、畅销商品 URL 和 UPC 发现完整商品记录。Python SDK 不提供这四种方法；JavaScript SDK 则全部支持。

### 多件商品，一个任务

传入 URL 列表只会创建一个任务，而不是为每件商品分别创建任务。

```python
from brightdata import SyncBrightDataClient

with SyncBrightDataClient(auto_create_zones=False) as client:
    results = client.scrape.amazon.products([
        "https://www.amazon.com/dp/B0CRMZHDG8",
        "https://www.amazon.com/dp/B085DVHQ57",
    ])
    for result in results:
        row = result.data
        print(row["asin"], "|", row["brand"], "|", row["final_price"], row["currency"])
```

```
B0CRMZHDG8 | STANLEY | 39.95 USD
B085DVHQ57 | Owala | 29.99 USD
```

要判断数据行属于哪件商品，请读取该行自身的 `asin`，如上所示；不要依据 `result.url`。SDK 按位置将结果与传入的 URL 配对，但 API 每次返回数据行的顺序可能不同，因此 `result.url` 指向的商品可能与 `result.data` 中的商品不一致。所有结果还都会标记为 `success=True`，包括无效 ASIN 对应的错误数据行（[sdk-python#60](https://github.com/brightdata/sdk-python/issues/60)）。

### 现在触发，稍后获取

如果要抓取的商品不止几件，不必让进程阻塞一小时。先触发任务并保存快照 ID，待任务就绪后再获取结果。快照可在 30 天内下载。

```python
import time

from brightdata import SyncBrightDataClient

with SyncBrightDataClient(auto_create_zones=False) as client:
    job = client.scrape.amazon.products_trigger("https://www.amazon.com/dp/B0CRMZHDG8")
    print("snapshot:", job.snapshot_id)
    while (status := client.scrape.amazon.products_status(job.snapshot_id)) not in ("ready", "failed"):
        time.sleep(5)
    print("status:", status)
    record = client.scrape.amazon.products_fetch(job.snapshot_id)[0]
    print("fetched:", record["asin"], "|", record["title"][:40])
```

```
snapshot: sd_muars9x1e0ejmq4cx
status: ready
fetched: B0CRMZHDG8 | STANLEY Quencher H2.0 Flow State Tumbler
```

<a id="the-data"></a>
## 数据

多数人需要的商品字段包括：

```
asin  title  brand  final_price  currency  rating  reviews_count  availability
```

代码没有写死字段列表。API 返回的所有字段都会保存在 `result.data` 和命令行工具生成的文件中。

API 在数据结构中以 `pii: true` 将以下三个字段标记为个人数据：`seller_name`、`zipcode` 和 `coupon`。`get_metadata()` 会丢失这一标记（[sdk-python#61](https://github.com/brightdata/sdk-python/issues/61)），因此下表改为读取原始元数据接口。

<!-- fields:start -->
<details>
<summary>全部 119 个字段及其类型和说明</summary>

此列表每天通过原始元数据接口从数据集结构重新生成，因此不会过时。一件商品只包含适用于自身的字段：示例文件包含这 119 个字段中的 97 个，另外还有数据结构未列出的 `timestamp` 和 `input`。

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `title` | 文本 | 商品标题 |
| `seller_name` | 文本 | 个人数据。卖家名称 |
| `brand` | 文本 | 商品品牌 |
| `description` | 文本 | 商品简介 |
| `initial_price` | 价格 | 初始价格 |
| `currency` | 文本 | 商品价格的币种 |
| `availability` | 文本 | 商品供货状态 |
| `reviews_count` | 数字 | 评论数量 |
| `categories` | 数组 | 商品分类 |
| `parent_asin` | 文本 | 商品的父 ASIN |
| `asin` | 文本 | 每件商品的唯一标识符 |
| `buybox_seller` | 文本 | 购买框中的卖家 |
| `number_of_sellers` | 数字 | 该商品的卖家数量 |
| `root_bs_rank` | 数字 | 商品在总分类中的畅销榜排名 |
| `ISBN10` | 文本 | 图书的 ISBN-10 标识符 |
| `answered_questions` | 数字 | 已回答问题的数量 |
| `domain` | URL | 商品所在域名的 URL |
| `images_count` | 数字 | 图片数量 |
| `url` | URL | 直接指向商品的 URL |
| `video_count` | 数字 | 视频数量 |
| `image_url` | URL | 直接指向商品图片的 URL |
| `item_weight` | 文本 | 商品重量 |
| `rating` | 数字 | 商品评分 |
| `product_dimensions` | 文本 | 商品尺寸 |
| `seller_id` | 文本 | 每位卖家的唯一标识符 |
| `image` | URL | 直接指向商品图片的 URL |
| `date_first_available` | 文本 | 商品首次上架日期 |
| `discount` | 文本 | 商品折扣信息 |
| `model_number` | 文本 | 商品型号 |
| `manufacturer` | 文本 | 商品制造商 |
| `department` | 文本 | 商品所属部门 |
| `plus_content` | 布尔值 | 是否存在附加内容 |
| `upc` | 文本 | 通用商品代码 |
| `video` | 布尔值 | 是否有视频 |
| `top_review` | 文本 | 商品的置顶评论 |
| `final_price_high` | 价格 | 最终价格为区间时的最高值 |
| `final_price` | 价格 | 商品最终价格 |
| `variations` | 数组 | 同一商品不同变体的详细信息 |
| `delivery` | 数组 | 配送相关信息 |
| `features` | 数组 | 商品特性 |
| `format` | 数组 | 图书版本形式的相关信息 |
| `buybox_prices` | 对象 | 商品价格详情 |
| `input_asin` | 文本 | 输入的 ASIN（目前未启用） |
| `ingredients` | 文本 | 商品成分，主要适用于食品 |
| `origin_url` | URL | 提取此记录时使用的来源页面 URL |
| `bought_past_month` | 数字 | 过去一个月的购买数量（按 Amazon 页面显示） |
| `is_available` | 布尔值 | 商品是否仍有货 |
| `root_bs_category` | 文本 | 畅销榜根分类 |
| `bs_category` | 文本 | 畅销榜分类 |
| `bs_rank` | 数字 | 商品在特定分类中的畅销榜排名 |
| `badge` | 文本 | 商品徽章，例如“#1 Best Seller”或“Amazon's Choice” |
| `subcategory_rank` | 数组 | 按子分类列出的畅销榜排名 |
| `amazon_choice` | 布尔值 | 商品是否获得 Amazon's Choice 标识 |
| `images` | 数组 | 商品图片的 URL |
| `product_details` | 数组 | 完整商品详情 |
| `prices_breakdown` | 对象 | 标价、常见售价及促销状态明细 |
| `country_of_origin` | 文本 | 商品原产国 |
| `from_the_brand` | 数组 | 页面上由品牌提供的宣传媒体内容 |
| `product_description` | 数组 | 嵌入商品描述部分的媒体内容 |
| `seller_url` | URL | 卖家在 Amazon 上的店铺或资料页 URL |
| `customer_says` | 文本 | `customer_says` 字段 |
| `sustainability_features` | 数组 | 可持续发展徽章及认证，含相关引用 |
| `climate_pledge_friendly` | 布尔值 | 商品是否显示 Climate Pledge Friendly 徽章 |
| `videos` | 数组 | 商品视频的 URL |
| `other_sellers_prices` | 数组 | 其他卖家对同一商品的报价 |
| `downloadable_videos` | 数组 | 直接指向媒体文件的 URL |
| `editorial_reviews` | 数组 | 图书的编辑评论 |
| `about_the_author` | 文本 | 作者简介 |
| `zipcode` | 文本 | 个人数据。用于估算配送及供货情况的邮政编码 |
| `coupon` | 文本 | 个人数据。优惠券 |
| `sponsered` | 布尔值 | 是否为赞助内容 |
| `store_url` | URL | 商品店铺的 URL |
| `ships_from` | 文本 | 商品发货地 |
| `city` | 文本 | 与配送、卖家或地点相关的城市 |
| `customers_say` | 对象 | 从评论中提取的 Amazon“顾客评价”摘要 |
| `max_quantity_available` | 数字 | 允许加入购物车的最大数量 |
| `variations_values` | 数组 | 商品变体及其可选值 |
| `language` | 文本 | 商品页面或内容的语言 |
| `return_policy` | 文本 | 商品页面显示的退货政策 |
| `inactive_buy_box` | 对象 | 购买框不可用或未启用时的价格信息 |
| `buybox_seller_rating` | 数字 | 购买框卖家的评分 |
| `premium_brand` | 布尔值 | 是否为高端品牌 |
| `amazon_prime` | 布尔值 | 是否支持 Amazon Prime 配送 |
| `coupon_description` | 文本 | 优惠券说明 |
| `all_badges` | 数组 | 所有徽章 |
| `sponsored` | 布尔值 | Amazon 赞助商品标记 |
| `variant_id` | 文本 | 特定变体的唯一标识符 |
| `product_category` | 文本 | 使用分隔符连接的完整分类导航路径 |
| `category_tree` | 数组 | 分类层级，以包含名称和 URL 的对象数组表示 |
| `availability_date` | 文本 | 缺货商品预计恢复供应的日期 |
| `listing_has_variations` | 布尔值 | 商品页面是否包含多个变体 |
| `variant_attributes` | 数组 | 当前变体的属性，以名称和值组成的配对表示 |
| `variants` | 数组 | 按类型分组的结构化变体选项 |
| `seller_privacy_policy` | 文本 | 卖家隐私政策的 URL |
| `seller_tos` | 文本 | 卖家服务条款的 URL |
| `return_window` | 数字 | 可退货期限，以天为单位 |
| `target_countries` | 数组 | 商品可配送至的国家或地区 |
| `store_country` | 文本 | 店铺所在国家或地区 |
| `category_urls` | 数组 | 分类导航路径 |
| `all_variations` | 布尔值 | 是否采集全部变体的输入字段 |
| `safety_information` | 文本 | 安全信息 |
| `subcategory_link` | 数组 | 按子分类列出的畅销榜链接 |
| `all_inactive_buy_box` | 数组 | 所有未启用购买框的信息 |
| `is_frequently_returned_item_badge` | 布尔值 | 是否显示“经常退货的商品”徽章 |
| `frequently_returned_item_message` | 文本 | 警告框中显示的文字 |
| `is_customers_usually_keep` | 布尔值 | 是否显示“顾客通常会保留此商品” |
| `title_badge` | 文本 | 徽章标题 |
| `review_images` | 数组 | 评论图片 |
| `review_videos` | 数组 | 评论视频 |
| `also_viewed` | 数组 | 顾客还浏览了的商品 |
| `similar_items` | 数组 | 推荐的类似商品 |
| `bought_past_month_text` | 文本 | 过去一个月的购买数量（按 Amazon 页面显示），采用文本格式 |
| `is_high_price` | 布尔值 | 商品是否被标记为高价 |
| `title_highlight` | 文本 | Amazon 在商品页面标题后直接显示的补充营销或亮点文字 |
| `title_clean` | 文本 | 卖家或品牌定义的实际商品标题，不含 Amazon 在标题旁或标题后显示的亮点文字 |
| `customers_say_topics` | 数组 | “顾客评价”部分的结构化主题明细 |
| `brand_url` | URL | 商品品牌的 URL |
| `variant_condition` | 文本 | 商品变体的状况，例如全新、翻新或成色极佳的翻新商品 |
| `is_aplus_premium` | 布尔值 | 商品是否使用高级 A+ 内容 |

</details>
<!-- fields:end -->

<details>
<summary>运行 <code>amazon-scraper B0CRMZHDG8</code> 后生成的真实输出文件开头</summary>

```json
{
  "generated_at": "2026-09-21T04:54:18.105606+00:00",
  "products": [
    {
      "asin": "B0CRMZHDG8",
      "product": {
        "title": "STANLEY Quencher H2.0 Flow State Tumbler, 40 oz, Fuchsia",
        "seller_name": "Avrix Brands",
        "brand": "STANLEY",
        "description": "Constructed of recycled stainless steel for sustainable sipping, our 40 oz Quencher H2.0 offers maximum hydration with fewer refills. Commuting, studio workouts, day trips or your front porch\u2014you\u2019ll want this tumbler by your side. Thanks to Stanley\u2019s vacuum insulation, your water will stay ice-cold, hour after hour. The advanced FlowState\u2122 lid features a rotating cover with three positions: a straw opening designed to resist splashes while holding the reusable straw in place, a drink opening, and a full-cover top. The ergonomic handle includes comfort-grip inserts for easy carrying, and the narrow base fits just about any car cup holder.",
        "initial_price": 45,
        "currency": "USD",
        "availability": "In Stock",
        "reviews_count": 205036,
        "categories": [
          "Home & Kitchen",
          "Kitchen & Dining",
          "Storage & Organization",
          "Thermoses",
  ...
```

包含一件商品所有字段的完整文件见 [examples/sample_output.json](examples/sample_output.json)。

</details>

<a id="when-it-fails"></a>
## 出错时

| 看到的内容 | 含义 |
| --- | --- |
| `API token required but not found.` | 在发送任何请求前以退出码 2 退出。请设置令牌。 |
| `failed  ASIN: The navigation resulted in a dead page (404 status code)` | 以退出码 1 退出。商品不存在，通常是 ASIN 拼写错误。 |
| `failed  ASIN: the API returned no row for this ASIN` | 以退出码 1 退出。任务结果中没有该输入对应的数据行。请重新运行。 |
| `failed  ASIN: timeout` | 以退出码 1 退出。请求在 240 秒后超时。请重新运行。 |
| `failed  ASIN: Failed to fetch results: Request timeout after 30 seconds` | 以退出码 1 退出。任务已完成，但下载结果超过了 30 秒。请重新运行。 |

任何失败都会以退出码 1 退出，因此可以安全地根据运行结果决定脚本是否继续执行。

在 SDK 中，相同情况表现如下：

| 看到的内容 | 含义 |
| --- | --- |
| `AuthenticationError: Unauthorized (401)` | 已设置令牌，但令牌不正确。 |
| `result.success` 为 `False`，`result.status` 为 `"timeout"` | SDK 等待任务超时，默认等待 240 秒。调用时传入更大的 `timeout`，或重新运行。 |
| `Failed to fetch results: Request timeout after 30 seconds` | 任务已完成，但下载未能完成。这是客户端对每个 HTTP 请求设置的限制，即 `SyncBrightDataClient(timeout=30)`；它不是调用时传入的 `timeout`。增大调用参数 `timeout` 无济于事。 |
| `result.data` 中有一个包含 `error` 键的字典 | 这是 API 对某条输入返回的结果。此时 `result.success` 仍为 `True`；请检查数据行。 |

<a id="coding-agents"></a>
## 编程智能体

无需 Python，也无需安装其他内容。粘贴以下两行即可。第一行会打开浏览器完成一次登录；通过 SSH 或在 CI 中使用时，也可以运行 `bdata login --device`：

```bash
npx -p @brightdata/cli bdata login
npx -p @brightdata/cli bdata pipelines amazon_product "https://www.amazon.com/dp/B0CRMZHDG8"
```

`bdata pipelines list` 会列出所有类型。Amazon 相关类型包括 `amazon_product`、`amazon_product_reviews` 和 `amazon_product_search`。它们均接收 URL、输出 JSON，并按每条记录消耗一个额度。

运行 `npx skills add brightdata/skills`，可让 Claude Code、Cursor 和 Codex 学会使用这些命令及相关文档；之后你就可以用自然语言提出需求。完整指南：[在编程智能体中使用 Bright Data](https://docs.brightdata.com/quickstart-coding-agent)。

使用托管智能体、没有终端？[Bright Data MCP 服务器](https://github.com/brightdata/brightdata-mcp#which-tool-to-use)在 `ecommerce` 组中提供 Amazon 工具。该组默认关闭，需要显式启用：

    https://mcp.brightdata.com/mcp?token=YOUR_API_TOKEN&groups=ecommerce

智能体还可以自行创建账户，无需填写注册表单：[智能体注册](https://brightdata.com/auth.md)。Bright Data 的其他集成，包括 LangChain、Zapier 和 n8n，参见[集成文档](https://docs.brightdata.com/integrations/introduction)。

<a id="support"></a>
## 支持

如果发现本仓库的问题，请[提交 issue](https://github.com/brightdata/amazon-scraper-python/issues)；[CONTRIBUTING.md](CONTRIBUTING.md) 说明了提交时应提供哪些信息。

有关 API、账户或额度的问题，请联系 [Bright Data 支持团队](https://brightdata.zendesk.com/hc/en-us/requests/new)。

## 许可证

MIT。

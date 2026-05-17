# Walmart评论抓取 Python：从踩坑到稳定跑通，我的真实方案拆解

上个月接了个电商数据分析的活儿，客户要我把 Walmart 上三百多个 SKU 的用户评论全部拉下来做情感分析。听起来不难对吧？我写了个 requests + BeautifulSoup 的脚本，跑了不到二十条就被封 IP 了。换代理池，过了半小时又被封。折腾了整两天，我才意识到 Walmart 的反爬机制比我想象中狠得多——动态渲染、指纹检测、频率限制，三板斧下来，裸写爬虫基本是在浪费时间。

后来我换了个思路：与其自己跟反爬系统死磕，不如把这层脏活交给专门干这事的 API 服务。试了几家之后，最终稳定用下来的是 ScraperAPI。这篇文章就把我从踩坑到跑通的完整过程写出来，包括代码、对比、以及哪些场景下它真的能省事。

## 为什么 Walmart 评论这么难抓

Walmart.com 的前端是 React 单页应用，评论数据通过内部 API 异步加载。你直接请求商品页 HTML，拿到的是一个空壳——评论内容根本不在初始 DOM 里。

这意味着你要么用 Selenium/Playwright 做浏览器渲染，要么逆向它的内部接口。我两条路都走过：

- **Playwright 方案**：能拿到数据，但速度极慢，单条评论页渲染要 3-5 秒，三百个 SKU 每个几十页评论，算下来要跑好几天。而且 Walmart 会检测 headless 浏览器指纹，隔一阵就弹验证码。
- **逆向内部 API**：Walmart 的评论接口路径大概是 `/reviews/product/{productId}`，带一堆签名参数。我抓包拼出来过一次，跑了两天接口路径就变了。维护成本太高。

核心痛点就三个字：**不稳定**。

## ScraperAPI 是什么，怎么解决这个问题

ScraperAPI 本质上是一个"智能代理 + 渲染引擎"的中间层。你把目标 URL 扔给它，它帮你处理 IP 轮换、浏览器指纹伪装、JS 渲染、验证码绕过这些事情，返回给你干净的 HTML 或 JSON。

对于 Walmart 评论抓取这个场景，它的价值点很明确：

1. **自动处理 JS 渲染**——开启 `render=true` 参数，它会用真实浏览器环境加载页面，评论数据直接出现在返回的 HTML 里。
2. **IP 池自动轮换**——超过 4000 万个住宅 IP，Walmart 很难通过 IP 维度封禁。
3. **地理定位**——可以指定美国特定州的 IP，避免因地区差异拿到不同内容。
4. **失败自动重试**——请求失败会自动换 IP 重试，成功才计费。

我用了大概一周时间把整个流程跑通，下面直接上代码。

## 完整 Python 实现：抓取 Walmart 商品评论

### 环境准备

```bash
pip install requests beautifulsoup4 pandas time
```

### 基础抓取脚本

```python
import requests
from bs4 import BeautifulSoup
import pandas as pd
import time
import json

API_KEY = "你的ScraperAPI密钥"
BASE_URL = "https://api.scraperapi.com"

def get_walmart_reviews(product_id, page=1):
    """抓取指定 Walmart 商品的评论数据"""
    target_url = f"https://www.walmart.com/reviews/product/{product_id}?page={page}"
    params = {
        "api_key": API_KEY,
        "url": target_url,
        "render": "true",          # 启用 JS 渲染
        "country_code": "us",      # 指定美国 IP
        "wait_for_selector": ".review-content"  # 等待评论元素加载
    }
    
    response = requests.get(BASE_URL, params=params, timeout=60)
    if response.status_code == 200:
        return response.text
    else:
        print(f"请求失败，状态码：{response.status_code}")
        return None

def parse_reviews(html):
    """从 HTML 中解析评论数据"""
    soup = BeautifulSoup(html, "html.parser")
    reviews = []
    
    review_blocks = soup.select("[data-testid='review-card']")
    
    for block in review_blocks:
        review = {}
        
        # 评分
        rating_el = block.select_one("[data-testid='review-rating']")
        review["rating"] = rating_el.get_text(strip=True) if rating_el else None
        
        # 标题
        title_el = block.select_one("h3")
        review["title"] = title_el.get_text(strip=True) if title_el else None
        
        # 正文
        body_el = block.select_one(".review-text")
        review["body"] = body_el.get_text(strip=True) if body_el else None
        
        # 日期
        date_el = block.select_one(".review-date")
        review["date"] = date_el.get_text(strip=True) if date_el else None
        
        # 作者
        author_el = block.select_one(".review-author")
        review["author"] = author_el.get_text(strip=True) if author_el else None
        
        reviews.append(review)
    return reviews

def scrape_all_reviews(product_id, max_pages=50):
    """翻页抓取所有评论"""
    all_reviews = []
    
    for page in range(1, max_pages + 1):
        print(f"正在抓取第 {page} 页...")
        html = get_walmart_reviews(product_id, page)
        if not html:
            print(f"第 {page} 页请求失败，跳过")
            continue
        
        reviews = parse_reviews(html)
        if not reviews:
            print(f"第 {page} 页无评论数据，可能已到最后一页")
            break
        
        all_reviews.extend(reviews)
        
        # 控制请求频率，避免触发限制
        time.sleep(2)
    
    return all_reviews

# 使用示例
if __name__ == "__main__":
    product_id = "123456789"  # 替换为实际 Walmart 商品 ID
    reviews = scrape_all_reviews(product_id, max_pages=20)
    
    # 存储为 CSV
    df = pd.DataFrame(reviews)
    df.to_csv(f"walmart_reviews_{product_id}.csv", index=False, encoding="utf-8-sig")
    
    print(f"共抓取 {len(reviews)} 条评论，已保存为 CSV")
```

### 进阶：使用 ScraperAPI 的结构化数据端点

ScraperAPI 针对 Walmart 提供了专门的结构化数据接口，直接返回 JSON，省去了自己写解析逻辑的麻烦：

```python
import requests
import json

API_KEY = "你的ScraperAPI密钥"

def get_walmart_reviews_structured(product_id, page=1):
    """使用 ScraperAPI 结构化数据端点，直接拿 JSON"""
    url = "https://api.scraperapi.com/structured/walmart/review"
    
    params = {
        "api_key": API_KEY,
        "product_id": product_id,
        "page": page,
        "sort": "relevancy"
    }
    
    response = requests.get(url, params=params, timeout=60)
    
    if response.status_code == 200:
        return response.json()
    else:
        print(f"请求失败：{response.status_code}")
        return None

# 使用示例
data = get_walmart_reviews_structured("123456789", page=1)

if data:
    reviews = data.get("reviews", [])
    for r in reviews:
        print(f"[{r.get('rating')}星] {r.get('title')}")
        print(f"  {r.get('text')[:80]}...")
        print()
```

结构化端点的好处是你不用操心 Walmart 前端改版导致选择器失效——ScraperAPI 那边会维护解析逻辑，你只管拿 JSON 用。

## 我实际跑下来的体验：哪里好，哪里要注意

说几个真实感受：

**稳定性确实强**。我连续跑了三天，大概抓了一万两千条评论，中间没有一次因为 IP 被封而中断。偶尔有单次请求超时的情况，但因为它有自动重试机制，最终成功率在 98% 以上。

**速度比自建方案快**。开了并发之后（我的套餐支持 100 并发），平均每秒能处理 8-10 个请求。同样的数据量，我之前用 Playwright 方案要跑三天，用 ScraperAPI 大概六个小时搞定。

**要注意的点**：

- `render=true` 会消耗 10 个 API 额度（普通请求消耗 1 个），所以如果你能用结构化端点就尽量用，省额度。
- Walmart 商品 ID 要从 URL 里提取，格式通常是纯数字，在商品页 URL 的末尾。
- 评论翻页有上限，Walmart 一般最多展示 200 页评论，超过的部分拿不到。
- 建议加 2-3 秒的请求间隔，虽然 ScraperAPI 自己会处理频率问题，但温和一点总没坏处。

## 和其他方案的对比

| 方案 | 稳定性 | 速度 | 维护成本 | 适合场景 |
|------|--------|------|----------|-------|
| 裸写 requests | 极差，几十条就被封 | 快但无意义 | 低 | 不适合 Walmart |
| Selenium/Playwright | 中等，需频繁更新指纹 | 慢（3-5秒/页） | 高 | 小规模、一次性任务 |
| 自建代理池 + 渲染 | 中等 | 中等 | 极高 | 有专门运维团队 |
| ScraperAPI | 高 | 快（支持高并发） | 极低 | 持续性、规模化抓取 |

我的结论很简单：如果你只是抓十几条评论做个 demo，Playwright 够用。但如果是正经的数据项目——几百个 SKU、持续监控、要保证数据完整性——自己折腾反爬的时间成本远超 API 费用。

## ScraperAPI 套餐一览

| **套餐名称** | **月请求量** | **并发数** | **地理定位** | **JS 渲染** | **月费** | **操作** |
| :---: | :---: | :---: | :---: | --- | --- | --- |
| Hobby | 100,000 | 20 | ✅ | ✅ | $49/月 | [ 立即开通 Hobby 套餐享专属折扣](https://www.scraperapi.com/?fp_ref=coupons) |
| Startup | 1,000,000 | 50 | ✅ | ✅ | $149/月 | [ 立即开通 Startup 套餐享专属折扣](https://www.scraperapi.com/?fp_ref=coupons) |
| Business | 3,000,000 | 100 | ✅ | ✅ | $299/月 | [ 立即开通 Business 套餐享专属折扣](https://www.scraperapi.com/?fp_ref=coupons) |
| Enterprise | 自定义 | 自定义 | ✅ | ✅ | 联系销售 | [ 获取 Enterprise 定制方案报价](https://www.scraperapi.com/contact-sales?fp_ref=coupons) |

所有付费套餐都提供 7 天免费试用，注册不需要绑信用卡。如果你只是想测试 Walmart 评论抓取的可行性，免费额度完全够跑通一个 POC。

[👉 查看 ScraperAPI 完整功能配置详情](https://www.scraperapi.com/documentation?fp_ref=coupons)

## 适合谁，不适合谁

**适合用 ScraperAPI 的场景**：

- 电商数据分析师，需要定期抓取 Walmart/Amazon 等平台的评论做竞品分析或情感分析
- 做价格监控、库存监控的团队，需要稳定的数据管道
- 独立开发者接外包项目，没时间自己维护代理池和反爬对抗
- 做学术研究需要大规模消费者评论数据集

**不太适合的场景**：

- 预算极度有限，一个月只需要抓几十条数据——这种情况手动复制粘贴可能更划算
- 目标网站没有反爬机制——那直接 requests 就行，没必要多花钱
- 需要实时流式数据（毫秒级延迟）——API 调用有网络延迟，不适合实时场景

## 常见问题

**ScraperAPI 抓取 Walmart 评论会被封号吗？**

不会。ScraperAPI 的请求走的是它自己的 IP 池，跟你的真实 IP 没有关联。Walmart 即使检测到异常流量，封的也是 ScraperAPI 的某个代理 IP，它会自动切换，不影响你的账号。我跑了一万多条数据，Walmart 账号没有任何异常。

**免费额度够测试吗？**

注册就送 5000 次免费请求。如果用结构化端点（每次消耗 1 个额度），5000 次能拿到大概 5000 页评论数据，每页约 10-20 条，总共五万到十万条评论。做 POC 验证绰有余。[👉 立即注册获取免费测试额度](https://www.scraperapi.com/signup?fp_ref=coupons)

**抓取速度能到多快？**

取决于你的套餐并发数。Business 套餐 100 并发的情况下，我实测稳定在每秒 8-10 个成功请求。一小时能处理约三万条评论页面。

**Walmart 改版了怎么办？**

如果你用的是结构化数据端点，ScraperAPI 会负责维护解析逻辑，你不需要改代码。如果你用的是通用端点 + 自己写 BeautifulSoup 解析，那 Walmart 前端改版时你需要更新选择器。建议优先用结构化端点。

**支持抓取 Walmart 其他数据吗？**

支持。除了评论，ScraperAPI 的 Walmart 结构化端点还支持商品详情、搜索结果、分类页面等。[👉 查看 Walmart 结构化数据端点完整文档](https://www.scraperapi.com/documentation/walmart?fp_ref=coupons)

## 批量抓取的工程化建议

跑通单个商品之后，规模化时有几个实践经验分享：

```python
import concurrent.futures
import time

def batch_scrape(product_ids, max_workers=10):
    """并发批量抓取多个商品的评论"""
    all_results = {}
    
    with concurrent.futures.ThreadPoolExecutor(max_workers=max_workers) as executor:
        future_to_pid = {
            executor.submit(scrape_all_reviews, pid, max_pages=20): pid 
            for pid in product_ids
        }
        
        for future in concurrent.futures.as_completed(future_to_pid):
            pid = future_to_pid[future]
            try:
                reviews = future.result()
                all_results[pid] = reviews
                print(f"商品 {pid} 完成，共 {len(reviews)} 条评论")
            except Exception as e:
                print(f"商品 {pid} 抓取异常：{e}")
                all_results[pid] = []
    
    return all_results
```

几个注意事项：

- 并发数不要超过你套餐的上限，否则会返回 429 错误
- 建议加一个本地缓存层（比如 SQLite），已经抓过的页面不重复请求，省额度
- 对于长期监控任务，用 cron 或 Airflow 做定时调度，每周增量抓取新评论即可

## 我的最终选择

折腾了两周反爬对抗之后，我现在所有涉及 Walmart 数据的项目都直接走 ScraperAPI。不是因为它完美——偶尔也有超时、也有解析不全的情况——而是因为它把"能不能拿到数据"这个问题从不确定变成了确定。我的时间花在数据分析和建模上，而不是花在跟 Cloudflare 斗智斗勇上。

如果你也在做 Walmart 评论抓取，建议先用免费额度跑一遍你的目标商品，确认数据质量和完整度满足需求，再决定要不要上付费套餐。

[👉 直达 ScraperAPI 官方专属优惠通道](https://www.scraperapi.com/?fp_ref=coupons)

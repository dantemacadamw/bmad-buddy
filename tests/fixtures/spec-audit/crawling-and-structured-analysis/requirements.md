# 基于 Firecrawl 的实施方案

建议不要直接调用一次 `crawl` 后，把全部 Markdown 交给大模型。更稳妥的方案是：

> **Map 发现页面 → URL分类与优先级排序 → Scrape/Crawl定向采集 → 页面级事实提取 → 站点级聚合 → 分类与文案生成 → 证据校验**

Firecrawl v2 提供 `Map`、`Scrape`、`Crawl`、`Search`、文件解析和浏览器交互等能力。其中，`Map`适合快速发现站点URL，`Scrape`适合精确抓取单页，`Crawl`适合批量递归抓取。([Firecrawl Docs](https://docs.firecrawl.dev/api-reference/v2-introduction?utm_source=chatgpt.com))

------

# 一、整体架构

```text
用户输入URL
    │
    ▼
URL规范化与域名检查
    │
    ▼
Firecrawl Map
发现站点URL、标题、描述
    │
    ▼
页面分类器
首页 / 关于 / 产品 / 功能 / 解决方案 / 定价 / 文档 / 博客
    │
    ▼
页面价值评分与采样
P0、P1优先，P2按需
    │
    ├── 少量核心页面：Firecrawl Scrape
    └── 目录型页面：Firecrawl Crawl
    │
    ▼
Markdown清洗与内容分块
    │
    ▼
页面级结构化提取
功能、用户、场景、行业、主体、证据
    │
    ▼
站点级事实合并
去重、冲突处理、置信度计算
    │
    ▼
最终输出
名称、一级分类、二级分类、标签、
50字说明、200—300字介绍、来源
```

核心原则是：**Firecrawl负责网页发现、渲染和正文提取；业务系统负责URL筛选、页面分类、事实聚合和质量控制。**

------

# 二、第一阶段：使用 Map 发现站点页面

Firecrawl `POST /v2/map` 可以快速返回站点URL列表，并附带部分页面标题和描述。它支持忽略查询参数、包含子域名、限制URL数量、使用站点地图等配置。([Firecrawl Docs](https://docs.firecrawl.dev/zh/api-reference/endpoint/map?utm_source=chatgpt.com))

## 2.1 Map请求示例

```python
import os
import requests

FIRECRAWL_API_KEY = os.environ["FIRECRAWL_API_KEY"]
BASE_URL = "https://api.firecrawl.dev/v2"

def map_website(url: str) -> list[dict]:
    response = requests.post(
        f"{BASE_URL}/map",
        headers={
            "Authorization": f"Bearer {FIRECRAWL_API_KEY}",
            "Content-Type": "application/json",
        },
        json={
            "url": url,
            "sitemap": "include",
            "ignoreQueryParameters": True,
            "includeSubdomains": False,
            "limit": 1000,
            "timeout": 60000,
        },
        timeout=70,
    )
    response.raise_for_status()

    data = response.json()
    if not data.get("success"):
        raise RuntimeError(data.get("error", "Map failed"))

    return data.get("links", [])
```

返回结果可统一为：

```json
[
  {
    "url": "https://example.com/about",
    "title": "About Us",
    "description": "Learn more about Example..."
  },
  {
    "url": "https://example.com/features",
    "title": "Features",
    "description": "Explore our core features..."
  }
]
```

## 2.2 为什么先使用 Map

相比直接全站 Crawl，先 Map 有三个优势：

1. 能在消耗大量抓取额度前看到站点结构；
2. 可以排除登录、标签、分页、招聘、法律条款等低价值页面；
3. 可以按照目标字段挑选最有价值的5—10页，避免在低价值页面上消耗抓取额度。

------

# 三、第二阶段：URL分类和优先级排序

Firecrawl负责发现URL，但“哪个页面值得抓”应由业务系统决定。

## 3.1 页面类型规则

建议先使用规则分类，再由轻量模型处理无法判断的URL。

```python
import re
from urllib.parse import urlparse

PAGE_RULES = {
    "homepage": [
        r"^/$"
    ],
    "about": [
        r"/about(?:-us)?/?$",
        r"/company/?$",
        r"/our-story/?$",
        r"/who-we-are/?$"
    ],
    "product_list": [
        r"/products?/?$",
        r"/services?/?$",
        r"/platform/?$",
        r"/features?/?$"
    ],
    "product_detail": [
        r"/products?/",
        r"/services?/",
        r"/features?/"
    ],
    "solution": [
        r"/solutions?/",
        r"/industries?/",
        r"/use-cases?/"
    ],
    "pricing": [
        r"/pricing/?$",
        r"/plans?/?$"
    ],
    "docs": [
        r"/docs?/",
        r"/help/",
        r"/support/",
        r"/guides?/",
        r"/faq"
    ],
    "case_study": [
        r"/customers?/",
        r"/case-studies?/",
        r"/success-stories?/"
    ],
    "blog": [
        r"/blog/",
        r"/news/",
        r"/resources?/"
    ],
    "contact": [
        r"/contact(?:-us)?/?$"
    ],
    "legal": [
        r"/privacy",
        r"/terms",
        r"/legal",
        r"/cookies"
    ],
    "careers": [
        r"/careers?",
        r"/jobs?"
    ]
}

def classify_page(url: str) -> str:
    path = urlparse(url).path.lower()

    if path in ("", "/"):
        return "homepage"

    for page_type, patterns in PAGE_RULES.items():
        for pattern in patterns:
            if re.search(pattern, path):
                return page_type

    return "other"
```

## 3.2 页面优先级

```python
PAGE_WEIGHTS = {
    "homepage": 100,
    "about": 90,
    "product_list": 95,
    "product_detail": 85,
    "solution": 80,
    "pricing": 65,
    "docs": 3,
    "case_study": 55,
    "contact": 45,
    "blog": 2,
    "legal": 1,
    "careers": 10,
    "other": 25,
}
```

可以进一步加入以下因素：

```python
def calculate_score(page: dict) -> int:
    page_type = page["page_type"]
    score = PAGE_WEIGHTS.get(page_type, 20)

    url = page["url"].lower()
    title = (page.get("title") or "").lower()
    description = (page.get("description") or "").lower()

    text = f"{url} {title} {description}"

    core_terms = [
        "product", "service", "feature", "solution",
        "platform", "about", "pricing", "documentation"
    ]

    noise_terms = [
        "login", "signin", "signup", "cart", "checkout",
        "author", "tag", "category", "archive", "privacy",
        "terms", "cookie"
    ]

    score += sum(5 for term in core_terms if term in text)
    score -= sum(10 for term in noise_terms if term in text)

    depth = len([p for p in urlparse(url).path.split("/") if p])
    score -= max(depth - 2, 0) * 3

    return score
```

------

# 四、第三阶段：制定页面采样策略

不建议抓取全部URL。可以根据页面类型设置上限；在候选页面充足时，通过全局上限将最终采样量控制在5—10页：

```python
PAGE_LIMITS = {
    "homepage": 1,
    "about": 1,
    "product_list": 2,
    "product_detail": 5,
    "solution": 2,
    "pricing": 1,
    "case_study": 1,
    "docs": 0,
    "blog": 0,
    "contact": 0,
    "legal": 0,
}

MAX_SELECTED_PAGES = 10
```

## 4.1 采样实现

```python
from collections import defaultdict

def select_pages(mapped_links: list[dict]) -> list[dict]:
    groups = defaultdict(list)

    for item in mapped_links:
        page = {
            "url": item["url"],
            "title": item.get("title"),
            "description": item.get("description"),
        }
        page["page_type"] = classify_page(page["url"])
        page["score"] = calculate_score(page)
        groups[page["page_type"]].append(page)

    selected = []

    for page_type, limit in PAGE_LIMITS.items():
        candidates = sorted(
            groups.get(page_type, []),
            key=lambda item: item["score"],
            reverse=True,
        )
        selected.extend(candidates[:limit])

    return sorted(selected, key=lambda item: item["score"], reverse=True)[:MAX_SELECTED_PAGES]
```

## 4.2 推荐的停止条件

完成核心页面抓取后，检查是否已经获得：

- 网站名称；
- 至少一个明确业务定位；
- 至少三个核心功能；
- 目标用户；
- 主要应用场景；
- 一级和二级分类证据；
- 至少两个不同类型的来源页面。

字段完整度达到要求后停止扩展抓取。

------

# 五、第四阶段：使用 Scrape 抓取核心页面

Firecrawl `Scrape` 可以处理单个URL，并返回Markdown、HTML、链接和元数据。它支持JavaScript渲染、正文提取、广告阻断、PDF解析、等待时间、代理、缓存和标签过滤等配置。([Firecrawl Docs](https://docs.firecrawl.dev/zh/api-reference/endpoint/scrape?utm_source=chatgpt.com))

## 5.1 推荐Scrape配置

```python
def scrape_page(url: str) -> dict:
    response = requests.post(
        f"{BASE_URL}/scrape",
        headers={
            "Authorization": f"Bearer {FIRECRAWL_API_KEY}",
            "Content-Type": "application/json",
        },
        json={
            "url": url,
            "formats": ["markdown", "links"],
            "onlyMainContent": True,
            "blockAds": True,
            "removeBase64Images": True,
            "parsers": ["pdf"],
            "waitFor": 1000,
            "timeout": 60000,
            "storeInCache": True,
            "maxAge": 86400000
        },
        timeout=70,
    )
    response.raise_for_status()

    payload = response.json()
    if not payload.get("success"):
        raise RuntimeError(payload.get("error", "Scrape failed"))

    return payload["data"]
```

`maxAge`可以允许Firecrawl复用一定时间范围内的缓存结果，减少重复抓取；`onlyMainContent`用于尽量保留正文，`blockAds`和`removeBase64Images`用于减少噪声。([Firecrawl Docs](https://docs.firecrawl.dev/zh/api-reference/endpoint/scrape?utm_source=chatgpt.com))

## 5.2 建议保存的数据

```json
{
  "url": "https://example.com/features",
  "page_type": "product_list",
  "title": "Features",
  "description": "Explore our features",
  "language": "en",
  "status_code": 200,
  "markdown": "...",
  "links": [],
  "scraped_at": "2026-07-17T..."
}
```

Firecrawl返回的元数据通常可包括页面标题、描述、来源URL、状态码、语言等字段。([Firecrawl Docs](https://docs.firecrawl.dev/zh/api-reference/endpoint/scrape?utm_source=chatgpt.com))

------

# 六、什么时候使用 Crawl

`Crawl`适用于以下情况：

- 产品目录页面较多；
- 页面结构清晰，集中在特定路径；
- 需要异步处理多个页面；
- 需要通过Webhook接收每页结果；
- Map结果不完整，需要沿页面链接递归发现。

Firecrawl Crawl支持：

- `includePaths`和`excludePaths`；
- 最大发现深度；
- Sitemap模式；
- 忽略查询参数；
- 是否包含子域名；
- 页面数量限制；
- 并发和延时；
- 各页面统一的Scrape配置。([Firecrawl Docs](https://docs.firecrawl.dev/api-reference/endpoint/crawl-post?utm_source=chatgpt.com))

## 6.1 定向抓取产品与公司页面

```python
def start_targeted_crawl(url: str) -> str:
    response = requests.post(
        f"{BASE_URL}/crawl",
        headers={
            "Authorization": f"Bearer {FIRECRAWL_API_KEY}",
            "Content-Type": "application/json",
        },
        json={
            "url": url,
            "includePaths": [
                "/about*",
                "/company*",
                "/products*",
                "/services*",
                "/features*",
                "/solutions*",
                "/pricing*",
                "/customers*"
            ],
            "excludePaths": [
                "/login*",
                "/signup*",
                "/account*",
                "/cart*",
                "/checkout*",
                "/authors*",
                "/tags*",
                "/category*",
                "/careers*"
            ],
            "maxDiscoveryDepth": 3,
            "sitemap": "include",
            "ignoreQueryParameters": True,
            "limit": 10,
            "crawlEntireDomain": False,
            "allowExternalLinks": False,
            "allowSubdomains": False,
            "ignoreRobotsTxt": False,
            "delay": 500,
            "maxConcurrency": 3,
            "scrapeOptions": {
                "formats": ["markdown"],
                "onlyMainContent": True,
                "blockAds": True,
                "removeBase64Images": True,
                "parsers": ["pdf"],
                "timeout": 60000,
                "storeInCache": True
            }
        },
        timeout=70,
    )
    response.raise_for_status()
    result = response.json()

    if not result.get("success"):
        raise RuntimeError(result.get("error", "Crawl start failed"))

    return result["id"]
```

这里不建议设置`ignoreRobotsTxt: true`。默认尊重网站的robots规则，也不应尝试绕过登录、验证码或权限限制。Firecrawl Crawl提供了相关参数，但产品设计上应维持合规默认值。([Firecrawl Docs](https://docs.firecrawl.dev/api-reference/endpoint/crawl-post?utm_source=chatgpt.com))

------

# 七、Scrape与Crawl的选择

| 场景                           | 推荐方式               |
| ------------------------------ | ---------------------- |
| 首页、关于页、定价页           | Scrape                 |
| 已通过Map选出的5—10个核心页面 | 批量Scrape             |
| `/products`下有大量详情页      | Crawl后再抽样          |
| JavaScript动态页面             | Scrape，必要时增加等待 |
| 需要点击、展开或输入           | Interact               |
| PDF文件                        | Scrape并开启PDF parser |
| 定期检测变化                   | Monitor                |

对于复杂点击和表单交互，Firecrawl当前建议先Scrape页面，再通过`/v2/scrape/{scrapeId}/interact`使用自然语言指令或Playwright代码进行操作，而不是依赖复杂的`actions`配置。([Firecrawl Docs](https://docs.firecrawl.dev/api-reference/endpoint/scrape?utm_source=chatgpt.com))

------

# 八、第五阶段：页面级结构化提取（采用方案A）

第五阶段采用方案A：Firecrawl只负责返回Markdown和页面元数据，自有LLM负责页面级事实提取。流程上应先对单页Markdown完成基础清洗和分块，再把页面类型、页面标题、URL和Chunk内容一起输入提取模型。

采用方案A的原因：

- 可以统一模型和提示词；
- 能实施跨页面证据聚合；
- 便于重新分析而不重新抓取；
- 可以控制模型成本；
- 能保留完整原始证据。

页面级提取Schema：

```json
{
  "page_type": "product_detail",
  "official_names": [],
  "company_names": [],
  "main_topic": "",
  "products": [],
  "core_functions": [],
  "target_users": [],
  "industries": [],
  "use_cases": [],
  "technical_features": [],
  "business_model": "",
  "claims": [],
  "evidence": [
    {
      "field": "core_functions",
      "value": "会议转写",
      "quote": "Automatically transcribe every meeting...",
      "url": "https://example.com/features"
    }
  ]
}
```

建议要求模型：

1. 只根据当前页面提取；
2. 不确定时输出空值；
3. 每项核心事实必须提供原文证据；
4. 区分事实和营销声明；
5. 不在页面级直接生成一级分类。

本方案不使用Firecrawl Scrape JSON模式作为默认提取路径。该模式适合单页、少量字段、结构稳定的抽取任务，但本方案的分类结果依赖跨页面证据聚合，因此默认保留Markdown并在业务系统中完成提取。

------

# 九、第五阶段配套处理：内容清洗和分块

即使Firecrawl已经返回主内容，仍建议进行二次清洗。

## 9.1 清洗规则

删除：

- 重复导航；
- Cookie声明；
- 重复CTA；
- 页尾版权；
- “立即开始”“联系我们”等无语义按钮；
- 同一段落的重复版本；
- Markdown中的空链接和图片；
- 过长法律条款。

保留：

- H1—H3标题；
- 正文段落；
- 功能列表；
- 产品卡片；
- FAQ；
- 表格；
- 场景与用户描述；
- 来源URL。

## 9.2 Markdown分块

```python
import re

def split_markdown(markdown: str, max_chars: int = 5000) -> list[str]:
    sections = re.split(r"(?=^#{1,3}\s)", markdown, flags=re.MULTILINE)

    chunks = []
    current = ""

    for section in sections:
        section = section.strip()
        if not section:
            continue

        if len(current) + len(section) + 2 <= max_chars:
            current += "\n\n" + section
        else:
            if current:
                chunks.append(current.strip())
            current = section

    if current:
        chunks.append(current.strip())

    return chunks
```

每个Chunk应附带：

```json
{
  "site_id": "...",
  "page_url": "...",
  "page_type": "product_detail",
  "page_title": "...",
  "section_title": "...",
  "chunk_index": 3,
  "content": "...",
  "content_hash": "..."
}
```

------

# 十、第六阶段：站点级事实聚合

页面级事实不能直接拼接，需要统一化。

## 10.1 同义词合并

例如：

```text
AI writing
AI writer
Artificial intelligence writing tool
智能写作
```

应标准化为：

```text
AI写作
```

可以维护以下字典：

- 分类词典；
- 功能词典；
- 用户角色词典；
- 行业词典；
- 技术词典；
- 商业模式词典。

## 10.2 信息权重

建议权重：

| 页面类型           | 权重 |
| ------------------ | ---- |
| 产品详情、官方文档 | 1.00 |
| 首页、产品列表     | 0.95 |
| 解决方案           | 0.90 |
| 关于我们           | 0.85 |
| 定价页             | 0.80 |
| 客户案例           | 0.70 |
| 官方博客           | 0.55 |
| 联系、法律页面     | 0.40 |

## 10.3 置信度计算

可采用：

```text
事实置信度 =
页面权重
× 证据明确度
× 来源一致性
× 信息时效性
```

简化实现：

```python
def calculate_confidence(evidence_items: list[dict]) -> float:
    if not evidence_items:
        return 0.0

    page_type_weights = {
        "product_detail": 1.0,
        "docs": 1.0,
        "homepage": 0.95,
        "product_list": 0.95,
        "solution": 0.90,
        "about": 0.85,
        "pricing": 0.80,
        "case_study": 0.70,
        "blog": 0.55,
        "legal": 0.40,
    }

    unique_urls = set()
    total = 0.0

    for item in evidence_items:
        total += page_type_weights.get(item["page_type"], 0.50)
        unique_urls.add(item["url"])

    source_bonus = min(len(unique_urls) * 0.08, 0.20)
    score = total / len(evidence_items) + source_bonus

    return round(min(score, 1.0), 2)
```

------

# 十一、第七阶段：分类生成

## 11.1 不要让模型自由发明分类

建议维护分类体系，例如：

```json
{
  "人工智能": [
    "AI写作",
    "AI图像生成",
    "AI视频生成",
    "AI会议助手",
    "AI搜索",
    "机器学习平台"
  ],
  "企业服务": [
    "CRM",
    "项目管理",
    "协同办公",
    "客服系统",
    "营销自动化",
    "人力资源管理"
  ],
  "开发工具": [
    "代码开发",
    "API开发",
    "测试工具",
    "云开发平台",
    "数据基础设施"
  ]
}
```

模型应执行“候选分类选择”，而不是开放式生成。

## 11.2 分类输入

给分类模型的内容应是站点事实表：

```json
{
  "official_name": "Example",
  "official_positioning": [
    "AI meeting assistant for teams"
  ],
  "products": [
    "Meeting recorder",
    "Transcription assistant"
  ],
  "core_functions": [
    "会议录音",
    "实时转写",
    "会议摘要",
    "行动项提取"
  ],
  "target_users": [
    "销售团队",
    "远程团队"
  ],
  "use_cases": [
    "客户会议",
    "团队会议"
  ],
  "category_candidates": {
    "人工智能": [
      "AI会议助手",
      "语音识别"
    ],
    "企业服务": [
      "协同办公"
    ]
  }
}
```

## 11.3 分类输出

```json
{
  "primary_category": "人工智能",
  "secondary_category": "AI会议助手",
  "reason": "核心功能集中于会议录音、转写、摘要及行动项提取。",
  "confidence": 0.95,
  "evidence_ids": ["ev_002", "ev_004", "ev_009"]
}
```

------

# 十二、第八阶段：生成标签和介绍

建议最终生成过程分两步：

1. 先确定事实和分类；
2. 再根据事实生成50字说明和200—300字介绍。

不要直接从原始Markdown一次性生成最终答案。

## 12.1 一句话说明模板

```text
面向{目标用户}的{产品类型}，提供{核心能力1}、{核心能力2}和{核心能力3}。
```

校验规则：

```python
def validate_short_description(text: str) -> bool:
    return 10 <= len(text.strip()) <= 50
```

## 12.2 详细介绍生成要求

```text
请根据站点事实表生成200—300字中文介绍。

要求：
1. 第一部分说明网站是什么、面向谁；
2. 第二部分说明3—5项核心功能；
3. 第三部分说明典型使用场景或差异点；
4. 结尾说明信息来源页面类型；
5. 不使用“领先、最好、第一”等宣传词；
6. 不加入事实表中不存在的信息；
7. 控制在200—300个中文字符。
```

------

# 十三、推荐最终数据结构

```json
{
  "website_name": "Example",
  "company_name": "Example Technologies Inc.",
  "primary_category": "人工智能",
  "secondary_category": "AI会议助手",
  "tags": [
    "会议转写",
    "会议摘要",
    "语音识别",
    "团队协作",
    "SaaS"
  ],
  "one_sentence_description": "面向企业团队的AI会议助手，提供录音、转写、摘要和行动项提取功能。",
  "detailed_description": "……",
  "core_functions": [
    "会议录音",
    "实时转写",
    "摘要生成",
    "行动项提取"
  ],
  "target_users": [
    "销售团队",
    "远程团队",
    "项目团队"
  ],
  "sources": [
    {
      "url": "https://example.com/",
      "page_type": "homepage",
      "title": "Example AI Meeting Assistant"
    },
    {
      "url": "https://example.com/features",
      "page_type": "product_list",
      "title": "Features"
    }
  ],
  "evidence": [
    {
      "id": "ev_001",
      "field": "core_functions",
      "value": "会议转写",
      "quote": "Automatically transcribe meetings...",
      "url": "https://example.com/features",
      "page_type": "product_list"
    }
  ],
  "confidence": {
    "website_name": 0.99,
    "primary_category": 0.95,
    "secondary_category": 0.93,
    "tags": 0.88
  },
  "crawl_metadata": {
    "mapped_url_count": 263,
    "selected_page_count": 10,
    "successful_page_count": 9,
    "failed_page_count": 1
  }
}
```

------

# 十四、异步任务和Webhook

生产环境不应长时间同步等待Crawl完成。推荐流程：

```text
POST /crawl
    ↓
保存crawl_id
    ↓
Firecrawl发送crawl.page Webhook
    ↓
逐页存储与页面级提取
    ↓
收到crawl.completed
    ↓
执行站点级聚合
```

Firecrawl提供`crawl.started`、`crawl.page`和`crawl.completed`等Webhook事件。`crawl.page`会携带当前页面数据；`crawl.completed`表示所有页面处理完成，结果也可以通过对应的Crawl查询接口获取。Webhook可以带HMAC-SHA256签名，应在服务端验证并使用`webhookId`进行幂等去重。([Firecrawl Docs](https://docs.firecrawl.dev/zh/api-reference/endpoint/webhook-crawl-page?utm_source=chatgpt.com))

Webhook处理伪代码：

```python
@app.post("/webhooks/firecrawl")
def receive_firecrawl_webhook():
    raw_body = request.get_data()
    signature = request.headers.get("X-Firecrawl-Signature")

    verify_firecrawl_signature(raw_body, signature)

    payload = request.get_json()
    webhook_id = payload["webhookId"]

    if webhook_already_processed(webhook_id):
        return {"success": True}

    event_type = payload["type"]
    crawl_id = payload["id"]

    if event_type == "crawl.page":
        for page in payload.get("data", []):
            save_page(crawl_id, page)
            enqueue_page_extraction(crawl_id, page)

    elif event_type == "crawl.completed":
        enqueue_site_aggregation(crawl_id)

    mark_webhook_processed(webhook_id)
    return {"success": True}
```

------

# 十五、失败处理

Firecrawl使用标准HTTP状态码；需要重点处理：

| 状态码 | 处理方式                 |
| ------ | ------------------------ |
| 400    | 检查参数                 |
| 401    | 检查API Key              |
| 402    | 检查额度                 |
| 408    | 增加超时或减少页面复杂度 |
| 429    | 指数退避重试             |
| 5xx    | 延迟重试                 |

Firecrawl官方说明会通过429表示速率或并发限制，应实现退避和重试。([Firecrawl Docs](https://docs.firecrawl.dev/zh/api-reference/v2-introduction?utm_source=chatgpt.com))

```python
import random
import time

def request_with_retry(request_func, max_retries=4):
    for attempt in range(max_retries):
        try:
            return request_func()
        except requests.HTTPError as exc:
            status = exc.response.status_code

            if status not in {408, 429, 500, 502, 503, 504}:
                raise

            if attempt == max_retries - 1:
                raise

            delay = min(2 ** attempt + random.random(), 30)
            time.sleep(delay)
```

------

# 十六、推荐的数据库设计

至少建立四张表。

## `sites`

```text
id
input_url
normalized_domain
status
mapped_url_count
selected_page_count
created_at
completed_at
```

## `pages`

```text
id
site_id
url
canonical_url
page_type
priority
title
description
status_code
markdown
content_hash
scrape_status
scraped_at
```

## `facts`

```text
id
site_id
page_id
field_type
normalized_value
original_value
evidence_quote
confidence
extraction_status
```

## `site_reports`

```text
site_id
website_name
company_name
primary_category
secondary_category
tags_json
short_description
detailed_description
confidence_json
sources_json
report_version
```

------

# 十七、推荐MVP范围

第一版不必使用Firecrawl的全部能力。建议只实现：

```text
1. 输入URL
2. Map发现链接
3. URL规则分类
4. 按目标字段选取5—10个核心页面
5. Scrape并获取Markdown
6. 清洗分块后进行LLM页面级事实提取
7. 站点事实聚合
8. 分类体系匹配
9. 生成最终介绍
10. 返回证据URL
```

第二阶段再增加：

- Crawl和Webhook；
- PDF解析；
- JavaScript交互；
- 多语言归一化；
- 内容变更监控；
- 增量抓取；
- 批量网站任务队列；
- 人工审核后台。

------

# 十八、最重要的实施决策

## 推荐方案

```text
Map负责“找到页面”
规则和模型负责“选择页面”
Scrape负责“获取高质量正文”
LLM负责“提取页面事实”
聚合程序负责“交叉验证”
最终模型负责“分类和生成介绍”
```

不要采用以下简化方案：

```text
Firecrawl全站Crawl
→ 合并所有Markdown
→ 一次大模型调用
→ 输出网站分类
```

这种实现容易出现：

- Token过量；
- 页面模板重复；
- 博客内容干扰主营业务；
- 分类依据不可追踪；
- 单个错误页面影响整体结论；
- 无法判断哪些结论是推测；
- 后续难以增量更新。

从可维护性、成本和准确率看，**“先发现、再筛选、再提取、最后聚合”**是更适合该工具的实现路径。

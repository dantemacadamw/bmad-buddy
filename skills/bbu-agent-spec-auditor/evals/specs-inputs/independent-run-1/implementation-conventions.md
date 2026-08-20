# Implementation Conventions

## Processing sequence

1. Normalize the input URL and enforce domain policy.
2. Map the site and retain URL, title, and description candidates.
3. Classify, score, group, and sample candidates.
4. Scrape selected pages and persist raw page results.
5. Clean Markdown and split it by H1–H3 sections.
6. Extract facts independently from each page chunk.
7. Normalize and aggregate facts across the site.
8. Select primary and secondary categories from the governed taxonomy.
9. Generate tags and Chinese descriptions from approved facts.
10. validate evidence and output rules, then persist a versioned report.

## Page classification

Rules are applied to the lower-cased URL path before a lightweight model handles unmatched URLs.

| Type | Matching routes |
|---|---|
| `homepage` | empty path or `/` |
| `about` | `/about`, `/about-us`, `/company`, `/our-story`, `/who-we-are` |
| `product_list` | terminal `/product(s)`, `/service(s)`, `/platform`, `/feature(s)` |
| `product_detail` | nested `/product(s)/`, `/service(s)/`, `/feature(s)/` |
| `solution` | `/solution(s)/`, `/industry/industries/`, `/use-case(s)/` |
| `pricing` | terminal `/pricing`, `/plan(s)` |
| `docs` | `/doc(s)/`, `/help/`, `/support/`, `/guide(s)/`, `/faq` |
| `case_study` | `/customer(s)/`, `/case-studies/`, `/success-stories/` |
| `blog` | `/blog/`, `/news/`, `/resource(s)/` |
| `contact` | terminal `/contact`, `/contact-us` |
| `legal` | `/privacy`, `/terms`, `/legal`, `/cookies` |
| `careers` | `/career(s)`, `/job(s)` |
| `other` | no rule matched |

## Scoring and sampling

Base weights:

| Page type | Weight | Page limit |
|---|---:|---:|
| `homepage` | 100 | 1 |
| `product_list` | 95 | 2 |
| `about` | 90 | 1 |
| `product_detail` | 85 | 5 |
| `solution` | 80 | 2 |
| `pricing` | 65 | 1 |
| `case_study` | 55 | 1 |
| `contact` | 45 | 0 |
| `other` | 25 | 0 by default |
| `careers` | 10 | 0 |
| `docs` | 3 | 0 |
| `blog` | 2 | 0 |
| `legal` | 1 | 0 |

- Add 5 for each occurrence class found across URL, title, and description: `product`, `service`, `feature`, `solution`, `platform`, `about`, `pricing`, `documentation`.
- Subtract 10 for each noise occurrence class: `login`, `signin`, `signup`, `cart`, `checkout`, `author`, `tag`, `category`, `archive`, `privacy`, `terms`, `cookie`.
- Subtract 3 for each path segment beyond depth two.
- Sort each type by descending score, take its per-type limit, merge, sort again, and cap at `MAX_SELECTED_PAGES = 10`.

## Firecrawl request profiles

### Map

- Endpoint: `POST /v2/map`
- `sitemap: include`
- `ignoreQueryParameters: true`
- `includeSubdomains: false`
- `limit: 1000`
- request timeout: 60 seconds; client timeout: 70 seconds

### Scrape

- Endpoint: `POST /v2/scrape`
- formats: `markdown`, `links`
- `onlyMainContent: true`
- `blockAds: true`
- `removeBase64Images: true`
- `waitFor: 1000`
- request timeout: 60 seconds; client timeout: 70 seconds
- `storeInCache: true`; cache freshness `maxAge: 86400000` milliseconds
- PDF parser activation follows the open MVP-scope decision in `SPEC.md`.

### Targeted Crawl after MVP

- Include `/about*`, `/company*`, `/products*`, `/services*`, `/features*`, `/solutions*`, `/pricing*`, `/customers*`.
- Exclude `/login*`, `/signup*`, `/account*`, `/cart*`, `/checkout*`, `/authors*`, `/tags*`, `/category*`, `/careers*`.
- `maxDiscoveryDepth: 3`, `sitemap: include`, `ignoreQueryParameters: true`, `limit: 10`.
- `crawlEntireDomain: false`, `allowExternalLinks: false`, `allowSubdomains: false`, `ignoreRobotsTxt: false`.
- `delay: 500` milliseconds and `maxConcurrency: 3`.
- Apply the Scrape main-content, ad, image, timeout, parser, and cache profile to each page.

## Acquisition decision table

| Situation | Method |
|---|---|
| Homepage, about, pricing, or already selected core page | Scrape |
| Product directory with many detail pages | Targeted Crawl, then sample |
| Incomplete Map results requiring recursive discovery | Targeted Crawl |
| Dynamic page that only needs rendering | Scrape with additional wait if required |
| Required click, expand, or input action | Scrape, then `/v2/scrape/{scrapeId}/interact` |
| Selected PDF | Scrape with PDF parser when in scope |
| Change detection | Monitor after MVP |

## Model boundaries

- Page extraction sees one page chunk plus page type, title, URL, and chunk metadata.
- It returns empty fields when the page does not support a value.
- It does not assign the site's primary category.
- Site aggregation receives page facts and evidence, not an undifferentiated Markdown corpus.
- Classification receives the normalized fact table and candidate taxonomy values.
- Copy generation runs only after facts and categories are accepted.

## Delivery phases

MVP includes URL input, Map, deterministic URL classification, 5–10-page selection, Scrape, cleaning and chunking, page-level LLM fact extraction, site aggregation, taxonomy matching, final copy, persistence, and evidence URLs.

Later phases may add Crawl and webhooks, PDF parsing, JavaScript interaction, multilingual normalization, change monitoring, incremental crawling, bulk website queues, and a human-review administration interface.

## Firecrawl references

- [Firecrawl v2 introduction](https://docs.firecrawl.dev/api-reference/v2-introduction)
- [Map endpoint](https://docs.firecrawl.dev/zh/api-reference/endpoint/map)
- [Scrape endpoint](https://docs.firecrawl.dev/zh/api-reference/endpoint/scrape)
- [Crawl endpoint](https://docs.firecrawl.dev/api-reference/endpoint/crawl-post)
- [Crawl page webhook](https://docs.firecrawl.dev/zh/api-reference/endpoint/webhook-crawl-page)

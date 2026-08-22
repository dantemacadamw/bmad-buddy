# Collection Pipeline

## Flow

```text
Input URL → normalize/domain check → Firecrawl Map → URL classification and scoring
→ 5–10 page sample → Scrape or targeted Crawl → Markdown cleaning/chunking
→ page facts → site facts → controlled-taxonomy classification → report and evidence
```

Use Firecrawl Map first, not a full-site Crawl. Map discovers URLs, titles, and descriptions before collection credits are spent. Normalize URLs, ignore query parameters, exclude subdomains by default, include sitemap entries, cap discovery at 1,000 URLs, and use a 60-second service timeout.

## URL Classification and Selection

Classify with path rules first, then use a lightweight model only for unresolved URLs. Recognized types are `homepage`, `about`, `product_list`, `product_detail`, `solution`, `pricing`, `docs`, `case_study`, `blog`, `contact`, `legal`, `careers`, and `other`. Treat the root path as `homepage`.

Default base weights: homepage 100; product list 95; about 90; product detail 85; solution 80; pricing 65; case study 55; contact 45; other 25; careers 10; docs 3; blog 2; legal 1. Add 5 per matching core term (`product`, `service`, `feature`, `solution`, `platform`, `about`, `pricing`, `documentation`); subtract 10 per noise term (`login`, `signin`, `signup`, `cart`, `checkout`, `author`, `tag`, `category`, `archive`, `privacy`, `terms`, `cookie`); subtract 3 for every path depth beyond two.

Apply per-type caps: homepage 1, about 1, product list 2, product detail 5, solution 2, pricing 1, case study 1, and zero for docs, blog, contact, and legal. Sort by score and apply the global limit of ten. Stop earlier if the collected set establishes website name, positioning, at least three core functions, target users, major use cases, category evidence, and two distinct source-page types.

## Targeted Collection

Use Scrape for a homepage, about page, pricing page, or Map-selected core page. Request Markdown and links; retain main content, block ads, remove base64 images, allow PDF parsing, wait about one second for rendering, use a 60-second service timeout, and cache for up to 24 hours where compatible.

Use Crawl when a product directory is large, its path structure is clear, recursive discovery is needed, or multiple pages must be handled asynchronously. Limit it to relevant paths such as about, company, products, services, features, solutions, pricing, and customers; exclude login, signup, account, cart, checkout, authors, tags, categories, and careers. Default safeguards: discovery depth 3, page limit 10, sitemap included, query parameters ignored, delay 500 ms, concurrency 3, no external links, no subdomains, and robots respected.

Choose Scrape for selected pages and JavaScript pages (increase wait only when needed); use Interact only for necessary click/expand/input behavior after a scrape; enable the PDF parser for PDFs. Content-change monitoring is a later capability.

## Content Preparation

Remove repeated navigation, cookie banners, repeated calls to action, footer copyright, semantically empty buttons, duplicated paragraphs, empty Markdown links/images, and overlong legal text. Retain headings through H3, body text, feature lists, product cards, FAQs, tables, user/scenario descriptions, and source URLs.

Split Markdown preferentially on H1–H3 boundaries; cap chunks at roughly 5,000 characters. Each chunk stores `site_id`, `page_url`, `page_type`, `page_title`, `section_title`, `chunk_index`, content, and `content_hash`.

## Source References

- [Firecrawl v2 introduction](https://docs.firecrawl.dev/api-reference/v2-introduction?utm_source=chatgpt.com)
- [Map endpoint](https://docs.firecrawl.dev/zh/api-reference/endpoint/map?utm_source=chatgpt.com)
- [Scrape endpoint](https://docs.firecrawl.dev/zh/api-reference/endpoint/scrape?utm_source=chatgpt.com)
- [Crawl endpoint](https://docs.firecrawl.dev/api-reference/endpoint/crawl-post?utm_source=chatgpt.com)

# Operations and Data Model

## Asynchronous Crawl

Do not synchronously wait for production Crawl completion. Save the returned crawl ID. On `crawl.page`, persist every page and enqueue page extraction. On `crawl.completed`, enqueue site aggregation. Verify the Firecrawl HMAC-SHA256 signature over the raw body before parsing; use `webhookId` for idempotent deduplication, then mark the event processed. `crawl.started` is available for lifecycle tracking.

## Failure Handling

Fail fast on non-retriable 4xx responses, including invalid parameters (400), invalid API key (401), and exhausted credits (402). Retry 408, 429, and 5xx responses up to four attempts with randomized exponential backoff, capped at 30 seconds. Record page-level failure so the final collection metadata exposes partial outcomes.

## Durable Records

`sites`: id, input URL, normalized domain, status, mapped URL count, selected page count, created time, completed time.

`pages`: id, site ID, URL, canonical URL, page type, priority, title, description, status code, Markdown, content hash, scrape status, scraped time.

`facts`: id, site ID, page ID, field type, normalized value, original value, evidence quote, confidence, extraction status.

`site_reports`: site ID, website name, company name, primary and secondary category, tags JSON, short and detailed description, confidence JSON, sources JSON, report version.

## Delivery Scope

MVP: URL input; Map; rule classification; 5–10-page sampling; Scrape to Markdown; cleaning/chunking; LLM page-fact extraction; site aggregation; taxonomy matching; report generation; evidence URLs.

Later: recursive Crawl/webhooks; PDF parsing; JavaScript interaction; multilingual normalization; change monitoring; incremental collection; batch-site queue; human-review console.

## Source Reference

- [Crawl webhook event reference](https://docs.firecrawl.dev/zh/api-reference/endpoint/webhook-crawl-page?utm_source=chatgpt.com)

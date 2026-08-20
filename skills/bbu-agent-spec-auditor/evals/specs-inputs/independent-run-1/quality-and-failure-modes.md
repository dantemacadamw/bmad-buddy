# Quality and Failure Modes

## Content preservation and cleaning

Remove repeated navigation, cookie notices, repeated calls to action, footer copyright, semantically empty buttons, duplicate paragraphs, empty Markdown links, images, and overlong legal text.

Preserve H1–H3 headings, body paragraphs, function lists, product cards, FAQs, tables, user and scenario descriptions, and the source URL. Split first on H1–H3 boundaries; if a section still exceeds 5,000 characters, split it without losing provenance.

## Evidence rules

- Extraction is page-local; cross-page reasoning occurs only during aggregation.
- Each core fact carries `field`, normalized `value`, exact `quote`, `url`, `page_type`, and a stable evidence ID.
- Unsupported values are empty.
- Marketing claims remain labeled claims until independent rules accept them as facts.
- Category choices cite evidence IDs and remain within the governed taxonomy.
- Conflicting values are retained with their sources until a declared rule or reviewer resolves them.

## Normalization

Maintain governed dictionaries for categories, functions, user roles, industries, technical terms, and business models. They merge synonymous forms such as “AI writing,” “AI writer,” and “智能写作” into one canonical value without discarding originals.

## Confidence

The conceptual score is:

`page authority × evidence clarity × source agreement × information freshness`, with a bounded source-diversity bonus.

Default page authority weights:

| Page type | Weight |
|---|---:|
| `product_detail`, `docs` | 1.00 |
| `homepage`, `product_list` | 0.95 |
| `solution` | 0.90 |
| `about` | 0.85 |
| `pricing` | 0.80 |
| `case_study` | 0.70 |
| `blog` | 0.55 |
| `contact`, `legal` | 0.40 |
| unlisted | 0.50 |

The baseline implementation averages evidence-item page weights, adds `min(unique_source_count × 0.08, 0.20)`, and caps the result at `1.00`. It must not treat this baseline as the unresolved product acceptance threshold.

## Evidence sufficiency and stopping

Before stopping expansion, verify all of:

- website name;
- at least one explicit business positioning statement;
- at least three core functions;
- target users;
- primary use cases;
- evidence for both category levels;
- at least two different source-page types.

If a condition is absent, record it and either expand acquisition or return an incomplete state according to the open product disposition decision.

## Generated copy checks

- The short description is 10–50 characters and follows: product type and audience, then core capabilities.
- The 200–300-character introduction states what the site is and whom it serves, describes 3–5 core functions, explains use cases or differentiators, and ends by naming source page types.
- Copy must not introduce facts absent from the fact table.
- Claims such as “leading,” “best,” or “first” are prohibited unless separately supported and explicitly allowed.

## Request failures

| Condition | Required response |
|---|---|
| HTTP 400 | Surface invalid request parameters; do not retry unchanged input. |
| HTTP 401 | Surface API-key failure; do not retry unchanged credentials. |
| HTTP 402 | Surface exhausted credit or quota; pause acquisition. |
| HTTP 408 | Retry with bounded exponential backoff, or reduce page complexity after exhaustion. |
| HTTP 429 | Retry with bounded exponential backoff and jitter while honoring rate or concurrency limits. |
| HTTP 500, 502, 503, 504 | Retry with bounded exponential backoff and jitter. |
| Other non-transient 4xx | Record page failure and do not retry unchanged input. |

Use at most four attempts by default. Backoff is `min(2^attempt + random_jitter, 30 seconds)`.

## Crawl webhook safety after MVP

- Verify Firecrawl HMAC-SHA256 against the raw request body before parsing or mutating state.
- Deduplicate every delivery by `webhookId`.
- Do not aggregate until `crawl.completed` and all prior page writes are durable.
- Default Crawl policy respects robots, excludes external links and subdomains, and never attempts to bypass login, CAPTCHA, or permissions.

## Known failure modes this design prevents

- Unbounded token consumption from full-site Markdown.
- Template, blog, legal, or archive content overwhelming core-product evidence.
- A single erroneous page determining the site's classification.
- Categories and copy with no traceable basis.
- Re-scraping solely to change extraction or aggregation logic.
- Duplicate webhook processing and premature site aggregation.

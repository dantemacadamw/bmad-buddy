# Quality and Operations

## Content cleaning

Remove repeated navigation, cookie notices, repeated calls to action, footer copyright, semantically empty buttons, duplicate paragraphs, empty Markdown links and images, and long legal boilerplate. Preserve H1–H3 headings, prose, feature lists, product cards, FAQs, tables, user and scenario descriptions, and the source URL.

## Fact normalization and confidence

- Use the business-system LLM over retained Markdown for page-fact extraction. Firecrawl Scrape JSON extraction is not the default because the final classification depends on auditable cross-page aggregation.
- Normalize equivalent names through maintained category, function, user-role, industry, technology, and business-model dictionaries while preserving each original value.
- Group evidence by normalized fact; conflicting evidence remains attached rather than overwritten.
- Use these evidence-authority weights: product detail and official docs `1.00`; homepage and product list `0.95`; solution `0.90`; about `0.85`; pricing `0.80`; case study `0.70`; official blog `0.55`; contact and legal `0.40`; unknown `0.50`.
- Calculate confidence from page authority, evidence explicitness, cross-source consistency, and freshness. The initial simplified formula averages page weights and adds `min(unique_source_count × 0.08, 0.20)`, capped at `1.00`.
- A confidence score never substitutes for evidence. Zero evidence produces zero confidence.

## Category and prose generation

- Classification consumes the aggregated site fact table, not raw Markdown.
- The classifier selects candidates from the maintained taxonomy and returns categories, reason, confidence, and evidence IDs.
- Prose generation begins only after facts and categories are fixed.
- The short Chinese description follows the semantic shape “target users + product type + three core abilities” and must contain 10–50 characters.
- The 200–300-character Chinese introduction states what the site is and who it serves, covers three to five core functions, names representative scenarios or differentiators, and closes by naming the source-page types.
- Generated prose must not add absent facts or use unsupported claims such as “leading,” “best,” or “number one.”

## Failure handling

| Provider status | Required behavior |
|---|---|
| `400` | Reject or correct invalid parameters; do not retry unchanged input. |
| `401` | Surface credential failure; do not retry. |
| `402` | Surface quota or billing failure; do not retry. |
| `408` | Retry with bounded exponential backoff. |
| `429` | Retry with bounded exponential backoff and honor provider limits. |
| `5xx` | Retry only `500`, `502`, `503`, and `504` with bounded backoff. |

Use at most four attempts. Delay follows `min(2^attempt + jitter, 30 seconds)`. Exhausted and non-retryable failures are persisted and reflected in final crawl metadata.

## Acceptance verification

- Contract validation rejects missing required fields, invalid categories, dangling evidence IDs, unsupported URLs, and out-of-range descriptions.
- Evidence validation samples every core field and confirms its quote exists in the retained page content.
- Reanalysis validation reruns extraction against stored Markdown and hashes without invoking Firecrawl.
- Partial-failure validation proves that one failed page does not erase successful evidence or masquerade as a complete crawl.

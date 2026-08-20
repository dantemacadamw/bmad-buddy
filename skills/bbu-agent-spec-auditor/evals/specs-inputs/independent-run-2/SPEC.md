---
id: SPEC-crawling-and-structured-analysis-solution
companions:
  - collection-pipeline.md
  - fact-and-report-contracts.md
  - operations-and-data-model.md
sources:
  - ../../planning-artifacts/crawling_and_structured_analysis_solution.md
---

> **Canonical contract.** This SPEC and the files in `companions:` are the complete, preservation-validated contract for what to build, test, and validate. Sources are retained only for audit.

# Crawling and Structured Website Analysis

## Why

Website analysis must produce accurate, traceable classifications and descriptions without wasting collection or model budget on low-value pages. Operators need an evidence-led pipeline that isolates page facts before drawing site-wide conclusions, making the result maintainable, reviewable, and safe to extend.

## Capabilities

- **CAP-1**
  - **intent:** The system can normalize and validate an input website URL, then discover candidate pages with their titles and descriptions.
  - **success:** A valid website submission produces a deduplicated, canonical candidate-URL inventory or a clear validation/collection failure.

- **CAP-2**
  - **intent:** The system can classify, value, and sample candidate URLs so collection focuses on business-relevant pages.
  - **success:** It selects no more than ten high-value pages under the configured per-type limits, with each selection’s type and priority recorded.

- **CAP-3**
  - **intent:** The system can collect high-quality main content and metadata from selected pages, using bulk recursive collection only where it is appropriate.
  - **success:** Each successfully collected page retains canonical URL, type, title, description, status, Markdown, links, content hash, and collection time.

- **CAP-4**
  - **intent:** The system can clean and divide page content, then extract page-scoped business facts with supporting source evidence.
  - **success:** Every asserted core fact identifies its field, value, source quote, URL, and page type; unknown values remain empty rather than inferred.

- **CAP-5**
  - **intent:** The system can reconcile page facts into an evidence-weighted site fact table.
  - **success:** Synonymous values are normalized, conflicts are handled by the chosen policy, and each accepted site fact has a bounded confidence score and traceable evidence.

- **CAP-6**
  - **intent:** The system can choose a primary and secondary category from a controlled taxonomy and generate tags and Chinese descriptions from the verified site facts.
  - **success:** A report cites evidence IDs for its category choice; its short description is 10–50 characters and its detailed description is 200–300 Chinese characters without unsupported or promotional claims.

- **CAP-7**
  - **intent:** The system can persist analysis state and return a structured, evidence-backed website report.
  - **success:** The report contains identity, categories, tags, descriptions, functions, audiences, sources, evidence, field confidence, and collection outcome counts.

- **CAP-8**
  - **intent:** The system can process recursive collection asynchronously and progress from individual pages to site aggregation safely.
  - **success:** Valid, previously unseen page events result in stored pages and queued extraction; a completion event queues aggregation exactly once.

## Constraints

- Firecrawl performs page discovery, rendering, and main-content extraction; the business system performs page selection, extraction, aggregation, and quality control.
- Default collection respects robots.txt and never bypasses authentication, CAPTCHA, or other access controls; targeted Crawl excludes external links and subdomains.
- Retain Markdown and use the owned LLM for default fact extraction; do not make Firecrawl Scrape JSON mode the default because conclusions depend on cross-page aggregation.
- Stop collection once required identity, positioning, functions, audience, use-case, category, and source-diversity evidence is complete.
- Production recursive collection is asynchronous; webhook signatures must be verified and `webhookId` events must be idempotently deduplicated.

## Non-goals

- Crawling an entire site, concatenating all Markdown, and asking one model to classify it.
- Circumventing robots, logins, CAPTCHAs, or permission boundaries.
- Including recursive Crawl, PDF parsing, browser interaction, monitoring, incremental collection, batch queues, or a human-review console in the MVP.

## Success signal

- For an accepted URL, the MVP delivers an auditable Chinese website report from a selected 5–10-page evidence set, including at least two distinct source-page types when available. A reviewer can trace each key conclusion to saved page evidence and distinguish collected facts from the report’s generated prose.

## Assumptions

- A maintained controlled taxonomy and normalization dictionaries are available to classification and aggregation.
- The existing service can provide durable storage, a work queue, and Firecrawl credentials; this contract does not choose their concrete implementations.

## Open Questions

- Who owns the taxonomy and synonym dictionaries, including their versioning and review?
- What confidence threshold, tie-breaker, and human-review path applies to conflicting facts or low-confidence classifications?
- Which queue, migrations, retention limits, and monitoring controls will production use?

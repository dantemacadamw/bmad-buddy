---
id: SPEC-crawling-and-structured-analysis
companions:
  - implementation-conventions.md
  - data-contracts.md
  - quality-and-failure-modes.md
  - architecture-diagrams.md
sources:
  - ../../planning-artifacts/crawling_and_structured_analysis_solution.md
---

> **Canonical contract.** This SPEC and the files in `companions:` are the complete, preservation-validated contract for what to build, test, and validate. Source documents listed in frontmatter are for traceability only.

# Crawling and Structured Website Analysis

## Why

Website classification and description generation becomes costly, noisy, and untraceable when an entire crawl is passed to one model call. This work creates an evidence-first pipeline that discovers and selects high-value pages, extracts page facts, reconciles them at site level, and produces classifications and Chinese copy that operators can audit.

## Capabilities

- **CAP-1**
  - **intent:** The system can accept a website URL and discover same-site candidate pages for analysis.
  - **success:** A run records up to 1,000 deduplicated candidate URLs with available titles and descriptions after URL normalization and domain checks.

- **CAP-2**
  - **intent:** The system can identify and select the pages most useful for understanding a site's identity and offering.
  - **success:** Deterministic classification and scoring select no more than 10 pages within the configured per-type limits and exclude configured low-value routes.

- **CAP-3**
  - **intent:** The system can acquire analyzable content and metadata from selected pages.
  - **success:** Every selected page records main-content Markdown, links, metadata, HTTP status, language, timestamps, and a success or failure state, reusing eligible cached content.

- **CAP-4**
  - **intent:** The system can convert page content into clean, traceable analysis chunks.
  - **success:** Noise is removed without losing headings, product content, FAQs, tables, user and scenario descriptions, or provenance, and each resulting chunk is no larger than 5,000 characters.

- **CAP-5**
  - **intent:** The system can extract page-level facts without introducing unsupported cross-page inference.
  - **success:** Each core extracted value is empty when unsupported or linked to its field, value, exact source quote, URL, and page type; factual statements remain distinguishable from marketing claims.

- **CAP-6**
  - **intent:** The system can reconcile page evidence into a normalized site-level fact table.
  - **success:** Synonyms and duplicates are normalized, conflicts remain visible, and each fact receives a reproducible confidence score based on source authority, evidence clarity, agreement, freshness, and source diversity.

- **CAP-7**
  - **intent:** The system can classify a site against a governed category taxonomy.
  - **success:** It selects an allowed primary and secondary category and returns the reason, confidence, and supporting evidence IDs without inventing categories.

- **CAP-8**
  - **intent:** The system can generate factual Chinese tags and descriptions from approved site facts.
  - **success:** The short description is 10–50 characters; the detailed introduction is 200–300 Chinese characters, follows the required content sequence, names source page types, and contains no unsupported or promotional claims.

- **CAP-9**
  - **intent:** The system can persist and return an auditable website report.
  - **success:** The versioned report includes website and company identity, categories, tags, descriptions, functions, users, sources, evidence, per-field confidence, and crawl counts backed by site, page, fact, and report records.

- **CAP-10**
  - **intent:** The system can stop acquisition when evidence is sufficient and expose incomplete analysis when it is not.
  - **success:** Expansion stops only after the required identity, positioning, at least three core functions, target users, use cases, category evidence, and two source-page types are present; otherwise the result records the missing evidence.

## Constraints

- Firecrawl owns URL discovery, rendering, and main-content acquisition; the application owns selection, page classification, cleaning, fact extraction, aggregation, taxonomy matching, and quality control.
- The MVP follows Map → targeted Scrape of 5–10 pages → page extraction → site aggregation → classification → copy generation; unrestricted full-site crawl followed by one model request is prohibited.
- Core facts and classifications must remain traceable to evidence IDs, exact quotes, source URLs, and page types; unsupported values remain empty.
- Page extraction uses application-managed LLM prompts over retained Markdown rather than Firecrawl JSON extraction so facts can be re-analyzed without re-scraping.
- Classification is closed-set selection from a maintained taxonomy, never open-ended category generation.
- Robots rules, authentication, CAPTCHA, and permission boundaries must not be bypassed; external links and subdomains are disabled by default.
- Only transient HTTP failures are retried, using bounded exponential backoff with jitter.
- Generated copy must not add facts absent from the approved fact table or use unsupported superlatives.
- Request profiles, selection values, schemas, evidence rules, and deferred behavior are normative in the companions.

## Non-goals

- The MVP does not include Crawl/webhook orchestration, JavaScript interaction, multilingual normalization, content monitoring, incremental crawling, bulk-site queues, or a human-review administration interface.
- The system does not crawl every discovered URL, bypass access controls, or classify from one undifferentiated Markdown corpus.
- The system does not allow the model to invent taxonomy values or silently convert weak evidence into facts.

## Success signal

- Given a representative public website, the system produces a persisted report from at most 10 selected pages whose core facts and category decisions can each be traced to source evidence, while explicitly returning missing or conflicting evidence instead of fabricating an answer.
- Automated checks decide the page cap, chunk limit, required fields, evidence linkage, taxonomy membership, Chinese copy lengths, retry behavior, and prohibited-claim rules.

## Open Questions

- Which complete, versioned primary and secondary category taxonomy is authoritative for CAP-7?
- What numeric confidence or completeness thresholds separate automatic acceptance, expansion crawling, partial output, and human review?
- Should failed or conflicting analyses be returned as partial reports, rejected, or routed to a review state?
- Does the MVP enable Firecrawl's PDF parser in default Scrape requests, or is all PDF parsing deferred?

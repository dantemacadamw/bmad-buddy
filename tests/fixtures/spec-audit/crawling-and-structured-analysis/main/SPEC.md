---
id: SPEC-website-crawling-structured-analysis
companions:
  - architecture-diagrams.md
  - crawl-policy.md
  - data-contracts.md
  - quality-and-operations.md
  - future-scope.md
sources:
  - ../../planning-artifacts/crawling_and_structured_analysis_solution.md
---

> **Canonical contract.** This SPEC and the files in `companions:` are the complete, preservation-validated contract for what to build, test, and validate. Source documents listed in frontmatter are for traceability only.

# Website Crawling and Structured Analysis MVP

## Why

The current analysis path risks excessive token use, template and blog noise, untraceable classifications, and site-wide conclusions distorted by a single page. The MVP must turn a submitted website into an evidence-grounded, reproducible structured report while controlling crawl cost and preserving enough provenance to audit or rerun every conclusion.

## Capabilities

- **CAP-1**
  - **intent:** The system can accept a website URL, establish an eligible canonical target, and discover its candidate pages.
  - **success:** A valid input produces one normalized target and a mapped URL inventory with available titles and descriptions; invalid or ineligible targets fail before retrieval.
- **CAP-2**
  - **intent:** The system can classify candidate pages and rank their value for business analysis.
  - **success:** Every candidate receives a deterministic page type and score, with a lightweight-model fallback used only when rules cannot resolve the type.
- **CAP-3**
  - **intent:** The system can select a bounded high-value page sample and stop expanding it when the required evidence is complete.
  - **success:** Selection respects the type budgets in `crawl-policy.md`, never exceeds ten pages, and stops once the documented completeness gate is satisfied.
- **CAP-4**
  - **intent:** The system can retrieve readable content and provenance from each selected page.
  - **success:** Each successful page yields Markdown, links, title, description, language, status, source URL, and retrieval time; each failure remains represented in crawl metadata.
- **CAP-5**
  - **intent:** The system can remove retrieval noise and divide page content into evidence-preserving semantic chunks.
  - **success:** Output chunks retain site, page, type, title, section, index, content, and content-hash provenance while removing the noise classes in `quality-and-operations.md`.
- **CAP-6**
  - **intent:** The system can extract page-local structured facts without filling evidence gaps by inference.
  - **success:** Every page result conforms to the page-fact contract, leaves uncertain fields empty, distinguishes facts from marketing claims, and attaches an exact quote and URL to every core fact.
- **CAP-7**
  - **intent:** The system can consolidate page facts into a normalized, conflict-aware site fact set.
  - **success:** Equivalent terms are merged, conflicting evidence remains traceable, and every consolidated fact has a reproducible confidence value based on source authority and diversity.
- **CAP-8**
  - **intent:** The system can assign primary and secondary categories from an approved taxonomy.
  - **success:** The output contains taxonomy-valid categories, a reason, confidence, and supporting evidence IDs; it never invents a category.
- **CAP-9**
  - **intent:** The system can generate a final structured website report using only consolidated facts.
  - **success:** The final report conforms to `data-contracts.md`, passes all language and length rules, includes sources and evidence, and contains no unsupported claim or superlative.
- **CAP-10**
  - **intent:** The system can retain analysis state so results are auditable, versioned, and re-analyzable without unnecessary retrieval.
  - **success:** Site, page, fact, and report records are linked by stable identifiers; raw Markdown, hashes, evidence, confidence, and report version are recoverable.
- **CAP-11**
  - **intent:** The system can survive predictable provider and page failures without corrupting or concealing partial results.
  - **success:** Retryable failures receive bounded backoff, permanent failures surface immediately, and final metadata reports accurate mapped, selected, successful, and failed counts.

## Constraints

- Integrate with the existing Python 3.12, FastAPI, PostgreSQL, and OpenAI Responses API system; use Firecrawl v2 for discovery and page retrieval.
- Firecrawl owns discovery, rendering, and main-content extraction; the business system owns selection, classification, fact aggregation, taxonomy matching, and quality control.
- Page facts are extracted from retained Markdown by the business-system LLM; Firecrawl Scrape JSON extraction is not the default path.
- Page-local extraction and site-level aggregation are mandatory boundaries; never merge an unrestricted whole-site crawl into one model call.
- The MVP uses Map plus selected-page Scrape, selects no more than ten pages, and applies `crawl-policy.md`.
- Respect robots rules and never bypass authentication, CAPTCHAs, access controls, or other permission boundaries.
- Extraction may use only the current page, must emit empty values for uncertainty, and must preserve evidence quotes and URLs.
- Category assignment must select from a maintained taxonomy.
- The short Chinese description must contain 10–50 characters; the detailed Chinese introduction must contain 200–300 characters, remain fact-grounded, and avoid promotional superlatives.
- Preserve original Markdown and content hashes so analysis can be rerun without unnecessary retrieval.
- Apply the retry and failure rules in `quality-and-operations.md`; retry loops must be bounded.

## Non-goals

- Unrestricted full-domain crawling or a single model call over all site Markdown.
- Crawl/Webhook orchestration in the MVP.
- PDF parsing, browser interaction, form completion, or authenticated-content access.
- Content-change monitoring or incremental recrawling.
- Multilingual normalization beyond the required Chinese report output.
- Bulk-site queues or a human-review administration interface.
- Open-ended category invention.
- Replacing Firecrawl with a custom renderer or crawler.

## Success signal

On a representative eligible website, the system selects no more than ten high-value pages, obtains evidence from at least two page types when available, identifies the website name, one clear positioning statement, at least three core functions, target users, use cases, and taxonomy-backed categories, then emits a schema-valid Chinese report whose core claims trace to page quotes and URLs. The same stored retrieval can be re-analyzed without recrawling, and partial provider failures remain explicit rather than contaminating or silently truncating the result.

## Assumptions

- Code fragments and provider numbers in the source are recommended initial defaults unless the source states a product requirement.
- Five useful pages is a target only when the site exposes that many eligible pages; the completeness gate may finish with fewer.

## Open Questions

- What production category taxonomy, versioning scheme, and owner are authoritative for CAP-8?
- Which representative site corpus and minimum field, category, and evidence accuracy thresholds define release acceptance?
- Should CAP-9 return JSON only, or must the MVP also persist or synchronize the report to Notion?
- What URL-safety policy must CAP-1 enforce for private-network targets, redirects, DNS rebinding, and user-supplied ports?
- What model, per-site token budget, and cost ceiling govern page extraction and final generation?

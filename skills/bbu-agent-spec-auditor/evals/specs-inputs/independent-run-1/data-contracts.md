# Data Contracts

All records retain stable site, page, chunk, evidence, and report identifiers so every output can be traced backward.

## Mapped page candidate

```yaml
url: string
title: string | null
description: string | null
page_type: homepage | about | product_list | product_detail | solution | pricing | docs | case_study | blog | contact | legal | careers | other
score: integer
```

## Scraped page

```yaml
site_id: string
url: string
canonical_url: string
page_type: string
title: string | null
description: string | null
language: string | null
status_code: integer | null
markdown: string | null
links: [string]
scrape_status: string
content_hash: string | null
scraped_at: datetime | null
failure: object | null
```

## Analysis chunk

```yaml
site_id: string
page_url: string
page_type: string
page_title: string | null
section_title: string | null
chunk_index: integer
content: string
content_hash: string
```

## Page-level extraction

```yaml
page_type: string
official_names: [string]
company_names: [string]
main_topic: string
products: [string]
core_functions: [string]
target_users: [string]
industries: [string]
use_cases: [string]
technical_features: [string]
business_model: string
claims: [string]
evidence:
  - id: string
    field: string
    value: string
    quote: string
    url: string
    page_type: string
```

## Site fact table and classification input

```yaml
official_name: string
company_name: string | null
official_positioning: [string]
products: [string]
core_functions: [string]
target_users: [string]
industries: [string]
use_cases: [string]
technical_features: [string]
business_model: string | null
category_candidates:
  <primary-category>: [secondary-category]
evidence: [evidence-id]
conflicts: [object]
```

## Classification result

```yaml
primary_category: string
secondary_category: string
reason: string
confidence: number
evidence_ids: [string]
```

Both category values must exist in the active, versioned taxonomy, and the secondary value must belong to the selected primary category.

## Final site report

```yaml
website_name: string
company_name: string | null
primary_category: string
secondary_category: string
tags: [string]
one_sentence_description: string
detailed_description: string
core_functions: [string]
target_users: [string]
sources:
  - url: string
    page_type: string
    title: string | null
evidence:
  - id: string
    field: string
    value: string
    quote: string
    url: string
    page_type: string
confidence:
  <report-field>: number
crawl_metadata:
  mapped_url_count: integer
  selected_page_count: integer
  successful_page_count: integer
  failed_page_count: integer
report_version: string
```

## Persistence minimum

| Entity | Required fields |
|---|---|
| `sites` | `id`, `input_url`, `normalized_domain`, `status`, `mapped_url_count`, `selected_page_count`, `created_at`, `completed_at` |
| `pages` | `id`, `site_id`, `url`, `canonical_url`, `page_type`, `priority`, `title`, `description`, `status_code`, `markdown`, `content_hash`, `scrape_status`, `scraped_at` |
| `facts` | `id`, `site_id`, `page_id`, `field_type`, `normalized_value`, `original_value`, `evidence_quote`, `confidence`, `extraction_status` |
| `site_reports` | `site_id`, `website_name`, `company_name`, `primary_category`, `secondary_category`, `tags_json`, `short_description`, `detailed_description`, `confidence_json`, `sources_json`, `report_version` |

## Deferred Crawl event contract

- `crawl.started`: persist the Firecrawl crawl ID and running state.
- `crawl.page`: verify the signature, deduplicate by `webhookId`, persist every page in `data`, and enqueue page extraction.
- `crawl.completed`: enqueue site aggregation after all earlier page events are durable.
- Every accepted webhook is marked processed only after its state change is durable.

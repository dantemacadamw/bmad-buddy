# Data Contracts

## Page record

Required fields: `id`, `site_id`, `url`, `canonical_url`, `page_type`, `priority`, `title`, `description`, `language`, `status_code`, `markdown`, `links`, `content_hash`, `scrape_status`, and `scraped_at`.

## Content chunk

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

Split on H1–H3 boundaries and target no more than 5,000 characters per chunk. Oversized single sections may be subdivided, but each resulting chunk retains section and ordering provenance.

## Page-fact extraction

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
      "value": "...",
      "quote": "...",
      "url": "...",
      "page_type": "product_detail"
    }
  ]
}
```

Each extraction covers one page only. Unknown values remain empty. Every core fact has evidence; marketing claims remain separate from facts. Primary categories are not generated at this stage.

## Classification result

Required fields are `primary_category`, `secondary_category`, `reason`, `confidence`, and `evidence_ids`. Both categories must exist in the approved taxonomy and the evidence IDs must resolve to persisted evidence.

## Final site report

```json
{
  "website_name": "...",
  "company_name": "...",
  "primary_category": "...",
  "secondary_category": "...",
  "tags": [],
  "one_sentence_description": "...",
  "detailed_description": "...",
  "core_functions": [],
  "target_users": [],
  "sources": [
    {
      "url": "...",
      "page_type": "homepage",
      "title": "..."
    }
  ],
  "evidence": [
    {
      "id": "ev_001",
      "field": "core_functions",
      "value": "...",
      "quote": "...",
      "url": "...",
      "page_type": "product_list"
    }
  ],
  "confidence": {},
  "crawl_metadata": {
    "mapped_url_count": 0,
    "selected_page_count": 0,
    "successful_page_count": 0,
    "failed_page_count": 0
  }
}
```

`confidence` contains field-level values. Every source and evidence URL must belong to the analyzed site or an explicitly allowed source discovered from it.

## Persistence model

| Entity | Required data |
|---|---|
| `sites` | `id`, `input_url`, `normalized_domain`, `status`, mapped and selected counts, creation and completion times |
| `pages` | Page-record fields above |
| `facts` | `id`, `site_id`, `page_id`, `field_type`, normalized and original values, evidence quote, confidence, extraction status |
| `site_reports` | `site_id`, names, categories, tags JSON, both descriptions, confidence JSON, sources JSON, report version |

The storage model may use existing repository naming conventions, but it must preserve these relationships and audit fields.

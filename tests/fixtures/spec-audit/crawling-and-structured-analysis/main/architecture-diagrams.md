# Architecture Diagrams

## MVP processing flow

```mermaid
flowchart TD
    A["Submitted URL"] --> B["Normalize URL and validate domain"]
    B --> C["Firecrawl Map"]
    C --> D["Classify and score URLs"]
    D --> E["Select bounded high-value sample"]
    E --> F["Firecrawl Scrape"]
    F --> G["Clean and chunk Markdown"]
    G --> H["Extract page-local facts and evidence"]
    H --> I["Normalize and aggregate site facts"]
    I --> J["Select taxonomy categories"]
    J --> K["Generate and validate report"]
    K --> L["Persist report, evidence, confidence, and crawl metadata"]
```

The stage boundaries are contract boundaries: retrieval never decides business relevance, page extraction never creates site-wide categories, and report generation never reads raw Markdown directly.

## Evidence lineage

```mermaid
flowchart LR
    P["Page URL and metadata"] --> C["Content chunk"]
    C --> E["Quoted evidence"]
    E --> F["Normalized fact"]
    F --> A["Aggregated site fact"]
    A --> R["Category or report field"]
```

Every core report field must be traceable back through this chain to a source URL and quote.

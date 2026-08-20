# Fact and Report Contracts

## Page Fact Contract

The owned LLM receives the page type, title, URL, and each cleaned Markdown chunk. It extracts only current-page facts: official and company names, main topic, products, core functions, target users, industries, use cases, technical features, business model, claims, and evidence. Evidence objects contain `field`, `value`, source `quote`, and `url`.

The extractor must leave uncertainty empty, provide original-text evidence for every core fact, distinguish factual content from marketing claims, and not assign the site’s primary category at page level. Preserve raw Markdown so analysis can be rerun without recollecting pages.

## Aggregation

Normalize synonyms through maintained category, function, user-role, industry, technology, and business-model dictionaries. Example variants such as “AI writing,” “AI writer,” “Artificial intelligence writing tool,” and “智能写作” normalize to `AI写作`.

Calculate fact confidence from page authority, evidence explicitness, source consistency, and information freshness. Default page authority weights: product detail/docs 1.00; homepage/product list 0.95; solution 0.90; about 0.85; pricing 0.80; case study 0.70; blog 0.55; legal/contact 0.40. A simple score averages evidence authority and adds 0.08 per unique URL, capped at 0.20, then caps the result at 1.00. The unresolved conflict policy in SPEC.md governs disagreement.

## Classification and Writing

Classification is candidate selection from a controlled hierarchy, never free-form invention. The classifier receives the verified fact table and category candidates, returns `primary_category`, `secondary_category`, a reason, confidence, and supporting evidence IDs.

Generate the report in two stages: establish facts/categories first, then write. The short description follows “for {target users}, a {product type} providing {three core capabilities}” and must be 10–50 characters. The detailed Chinese description is 200–300 characters: identify the product and audience; name three to five functions; describe scenarios or differentiation; end with source-page types; avoid superlatives and anything absent from the fact table.

## Report Contract

Return: `website_name`, `company_name`, primary/secondary category, tags, short and detailed descriptions, core functions, target users, source records (`url`, page type, title), evidence records (`id`, field, value, quote, URL, page type), per-field confidence, and collection metadata (`mapped_url_count`, `selected_page_count`, `successful_page_count`, `failed_page_count`).

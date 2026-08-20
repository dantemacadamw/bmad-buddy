# Crawl Policy

## Discovery

- Use Firecrawl v2 Map before retrieval, with sitemap inclusion, ignored query parameters, no subdomains by default, a 1,000-URL discovery ceiling, and a 60-second provider timeout as initial defaults.
- Normalize URLs before classification and exclude duplicates caused by query strings or equivalent canonical targets.
- Treat Map titles and descriptions as ranking inputs, not as verified site facts.

## Page classes and selection weights

| Page type | Recognition examples | Selection weight | Per-type limit |
|---|---|---:|---:|
| `homepage` | root path | 100 | 1 |
| `about` | `/about`, `/company`, `/our-story`, `/who-we-are` | 90 | 1 |
| `product_list` | `/product`, `/service`, `/platform`, `/feature` index | 95 | 2 |
| `product_detail` | product, service, or feature descendants | 85 | 5 |
| `solution` | `/solution`, `/industry`, `/use-case` descendants | 80 | 2 |
| `pricing` | `/pricing`, `/plan` | 65 | 1 |
| `docs` | docs, help, support, guide, FAQ | 3 | 0 |
| `case_study` | customer, case study, success story | 55 | 1 |
| `blog` | blog, news, resource | 2 | 0 |
| `contact` | contact page | 45 | 0 |
| `legal` | privacy, terms, legal, cookies | 1 | 0 |
| `careers` | careers or jobs | 10 | 0 |
| `other` | no rule match | 25 | 0 by default |

Rules run before any model fallback. Ranking adds five points for each core term found in URL, title, or description (`product`, `service`, `feature`, `solution`, `platform`, `about`, `pricing`, `documentation`), subtracts ten for each noise term (`login`, `signin`, `signup`, `cart`, `checkout`, `author`, `tag`, `category`, `archive`, `privacy`, `terms`, `cookie`), and subtracts three points per path segment deeper than two.

## Sampling and stopping

- Sort candidates within each class by score, apply class limits, then globally sort and cap the result at ten pages.
- Target five to ten pages when the site offers enough eligible pages.
- Stop expanding the sample once evidence exists for the website name, one explicit business positioning, at least three core functions, target users, principal use cases, primary and secondary category candidates, and at least two different source-page types.
- If the completeness gate is not met, return the missing fields explicitly; do not manufacture them.

## MVP retrieval

- Scrape each selected page for Markdown and links with main-content extraction, ad blocking, base64-image removal, and caching enabled.
- Initial provider defaults are a 1-second render wait, 60-second provider timeout, 70-second client timeout, and 24-hour cache age.
- Record URL, page type, title, description, language, status code, Markdown, links, and retrieval time.
- PDF parsing, Interact, and recursive Crawl are deferred in `future-scope.md`.

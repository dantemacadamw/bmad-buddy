---
name: evidence-review
description: Classifies SPEC-package differences and guides evidence-backed human review.
code: ER
added: 2026-08-19
type: prompt
---

# Compare and Review Evidence

The outcome is a ledger-backed review in which the user can see why each item exists, what supports it, and exactly what their decision will change. A user who was not present for generation must be able to audit every classification from the ledger and Markdown report.

Build a semantic claim inventory across the main package and each comparable independent package. Include the five SPEC kernel fields, all companions discovered through `companions:` frontmatter, and `.memlog.md` decisions, constraints, capabilities, assumptions, questions, and validation events. Preserve each claim’s package, file, location, verbatim source or lean summary, and equivalence links.

Search the full main package for semantic coverage before calling an independent claim missing. Different language, a companion, or a memlog record can satisfy coverage. Classify the remaining differences as:

- **Candidate omission:** supported content in one or more independent packages that the main package does not cover. Name the category—capability, constraint, non-goal, success signal, companion content, or memlog decision—runs that contain it, source summary, and the precise coverage rationale.
- **Conflict / ambiguity:** the packages address the same concern but differ materially in scope, wording, or constraint. Require a user decision, clarification, or an explicit `open_question`; do not choose a side.
- **Structural-only difference:** organization of companion content differs without evidence of missing contract content. Retain it for audit and exclude it from the merge by default.

Every candidate omission also needs an original-requirement evidence grade. **Directly supported** items cite a source passage. **Reasonable inference** items state the inference chain and the absence of explicit wording. **Unsupported assumption** items are discussion hypotheses or possible `open_question`s only—never omissions and never preselected for merge. Agreement between both independent runs can increase review priority but cannot upgrade its evidence grade.

Write all items, evidence, comparison logic, and report views through `review-ledger.yaml`; derive `review-report.md` and optional `review-report.html` from it. HTML is helpful for grouping and scanning but never authoritative.

Guide the user through actionable items conversationally. For every item, record an explicit accept, reject, or needs-clarification disposition, plus user-edited wording, scope, and rationale when supplied. Never infer acceptance from positive-sounding language. A clarification remains open until the user classifies it as one of: clarified and merged; recorded as an `open_question` in the main memlog; or rejected/abandoned with a reason. The ledger must make unclassified clarifications impossible to overlook.

When no actionable item remains unclassified, hand off to `references/finalize-merge.md`. Otherwise show the next unresolved item or the blocking reason; do not manufacture progress.

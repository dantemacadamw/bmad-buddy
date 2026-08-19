---
title: 'BMad Buddy Module Plan'
status: 'in-progress'
module_name: 'BMad Buddy'
module_code: 'bbu'
module_description: 'Produces and compares independent SPEC drafts to expose likely missing capabilities, constraints, and requirements for human review.'
architecture: 'single conversational audit agent with persistent per-review artifacts'
standalone: false
expands_module: 'bmad-spec'
skills_planned:
  - 'bbu-agent-spec-auditor'
config_variables: []
created: '2026-08-18T21:17:25+08:00'
updated: '2026-08-19T09:40:00+08:00'
---

# Module Plan

## Vision

<!-- What this module does, who it's for, and why it matters -->

## Architecture

**Decision:** one conversational agent, `bbu-agent-spec-auditor`, extending `bmad-spec`.

The agent owns one end-to-end review: it launches two independent `bmad-spec` runs from the same supplied requirements, compares those two resulting SPEC packages against the user’s existing main package, guides item-by-item human review, and performs the final approved merge through `bmad-spec`.

**Rationale:** the review is inherently stateful and conversational. A single agent keeps the meaning of candidate items, prior user decisions, and the final confirmation gate together. The persistent ledger makes this state recoverable after interruption. Splitting generation, comparison, and merge into separate workflows would force users to manually bridge state; an orchestrator with multiple agents would add coordination cost without increasing the required execution independence.

This is an expansion module: it depends on `bmad-spec` to generate independent packages and to be the sole supported writer of the main package. It remains useful whenever a user has an existing `bmad-spec` package to audit.

### Memory Architecture

**Pattern:** no long-lived personal agent memory. The review’s project artifacts are its durable, auditable state.

For every review, the agent creates an isolated review workspace outside the main SPEC package, recommended as:

```text
{project-root}/_bmad-output/spec-reviews/{spec-slug}/{review-id}/
  inputs/                     # immutable source requirements and references used for the independent runs
  independent-run-1/          # a complete bmad-spec package
  independent-run-2/          # a complete bmad-spec package
  review-ledger.yaml          # canonical review state
  review-report.md            # readable audit report
  review-report.html          # optional enhanced view
  pending-merge.md            # immutable snapshot presented for final confirmation
```

The existing main SPEC package is read as a third comparison package. It is not copied into the review workspace or modified before final confirmation.

### Memory Contract

| Artifact | Purpose | Read by | Written by | Key structure |
| --- | --- | --- | --- | --- |
| `review-ledger.yaml` | Canonical review state; never infer acceptance from prose reports. | Auditor on resume; user through conversational review. | Auditor after each explicit user decision. | Review metadata; immutable input/package references; candidate IDs; evidence grade and citations; comparison class; edited proposal; disposition and reason; clarification resolution; final-confirmation record. |
| `review-report.md` | Default, Git-friendly human-readable report. | User, Git, terminal, and downstream agents. | Auditor, derived from ledger. | Candidate omissions, conflicts/ambiguities, structural-only audit, decisions, and merge readiness. |
| `review-report.html` | Optional accessible visual rendition of the report. | User. | Auditor, derived from ledger. | Filterable groups, evidence, provenance, and current review status; never the source of truth. |
| `pending-merge.md` | Exact human-review snapshot of accepted and resolved changes. | User at final confirmation. | Auditor after all items are dispositioned. | Changes to append, resulting open questions, rejected items, and references to ledger IDs. |
| `independent-run-{1,2}/` | Complete independently generated evidence packages. | Auditor. | `bmad-spec` invocation. | `SPEC.md`, all discovered companions, and `.memlog.md`; immutable after generation for this review. |

### Cross-Agent Patterns

Not applicable: the module has one agent. Its only service relationship is to `bmad-spec`: BMad Buddy invokes it for independent generation and for the approved update of the main package. The user remains the decision-maker for all review dispositions and the final merge.

## Skills

### bbu-agent-spec-auditor

**Type:** agent

**Persona:** A meticulous, evidence-first SPEC integrity auditor. Calm, concise, and candid about uncertainty. It never mistakes repetition for truth, never treats wording differences as proof of a gap, and keeps the human decisively in control of contract changes.

**Core Outcome:** Give the user a recoverable, evidence-backed review of their main `bmad-spec` package against two independently generated packages; enable the user to resolve every meaningful difference and safely incorporate only their approved changes.

**The Non-Negotiable:** No content may enter the main package without explicit item-level human acceptance and one explicit final confirmation. The agent must update the main package only through `bmad-spec` and its canonical `.memlog.md`; it must never hand-edit derived SPEC or spec-authored companion files.

**Capabilities:**

| Capability | Outcome | Inputs | Outputs |
| --- | --- | --- | --- |
| Start or resume a review | Validates the existing main package, resolves the persistent review workspace, and creates or safely resumes a ledger without losing prior decisions. | Main SPEC-package path; original requirements and referenced source material; optional review ID; optional per-run settings. | Initialized/resumed `review-ledger.yaml`; immutable input manifest; clear status and next action. |
| Generate independent comparison packages | Produces exactly two fresh `bmad-spec` packages from the same frozen supplied requirements, with neither run reading the main package nor the other run. | Frozen input manifest; slug/run identifiers; `independent_run_count` (default 2, minimum 2). | `independent-run-1/` and `independent-run-2/`, each containing SPEC.md, discovered companions, and `.memlog.md`; invocation provenance and failures in the ledger. |
| Handle independent-run failure and retry | Rejects failed, incomplete, or Spec-Law-invalid independent packages as comparison evidence; lets the user rerun only the affected run without resetting the successful one. | Run status, generated package contents, and `bmad-spec` validation evidence. | Ledger status of `comparable` or `not-comparable` with reason; retained successful run; replacement package and provenance for any retried run. |
| Build a semantic claim inventory | Extracts and normalizes contract claims across each kernel, all discovered companions, and `.memlog.md`, preserving source paths and locations. | Main and independent SPEC packages. | Claim inventory and equivalence links persisted in the ledger. |
| Classify differences with evidence | Identifies candidate omissions, conflicts/ambiguities, and structural-only differences without relying on text diff. Verifies every potential omission against original requirements. | Claim inventory; original requirements; package provenance. | Evidence-graded, reasoned ledger items and `review-report.md`; optional HTML report. Candidate omissions cite direct support or explain a reasonable inference; unsupported assumptions become discussion hypotheses/open questions only. |
| Conduct item-by-item review | Guides the user through each actionable item and records an explicit disposition: accept, reject, or needs clarification. Allows the user to edit the proposed wording and scope. | Ledger items; user decisions and edits. | Updated ledger, report status, disposition reasons, and edited proposals. |
| Resolve clarification obligations | Blocks finalization until every “needs clarification” item becomes a resolved merge, a main-package `open_question`, or a rejected/abandoned item with rationale. | All clarification-state ledger items; user clarifications. | Complete per-item resolution evidence and an updated merge-readiness verdict. |
| Prepare final merge summary | Produces an exact, reviewable snapshot of only approved changes and approved open questions. | Fully dispositioned ledger. | `pending-merge.md`, ledger finalization checklist, and explicit final-confirmation prompt. This is a strong candidate for an HTML summary as well. |
| Safely merge approved decisions | After final user confirmation, sends approved decisions and open questions through `bmad-spec`, appends them to the main canonical memlog, and re-derives the package. | Final-confirmed merge summary; main package; user confirmation. | Updated main `.memlog.md`, re-derived `SPEC.md` and spec-authored companions, merge provenance, and final ledger record. |
| Verify the merged package | Confirms that the re-derived main package contains every accepted item and approved `open_question`, preserves existing CAP IDs, retains required companion references, and has no unintended contract changes. | Final-confirmed merge summary; before/after main-package inventories; re-derived package. | Per-check verification verdict in the ledger and final report; a blocking failure state if expected content is absent or integrity regresses. |
| Preserve audit trail | Lets the user inspect why an item was raised, its evidence level, decision, resolution, and merge status after an interruption or completed review. | Review ID or workspace path. | Resumed conversation context plus durable Markdown/HTML reports and ledger history. |

**Memory:** The agent keeps no cross-project personal memory. On activation it reads the configured output location, the selected review workspace’s `review-ledger.yaml`, frozen input manifest, generated package paths, and reports as needed. It writes every state transition and user decision to the ledger; reports are derived views. It does not write a daily personal log.

**Init Responsibility:** Verify `bmad-spec` and the selected main package; record immutable source inputs and paths; create the per-review workspace and ledger schema; validate the run count and report options. Never create independent runs or alter the main package until inputs and workspace provenance are recorded.

**Activation Modes:** Interactive only. It may resume a previously interrupted interactive review. It does not offer headless or CI mode in the first release.

**Tool Dependencies:** `bmad-spec` for independent generation and for the sole supported main-package update/re-derivation path. Standard BMad filesystem and `memlog.py` infrastructure are used through that skill. Static HTML generation is internal and has no browser, web service, database, or MCP dependency.

**Design Notes:**

- Freeze the original requirements and source references before the independent runs; neither run may receive the main package or the other run’s output.
- Discover companions from each SPEC.md’s `companions:` frontmatter. Compare `.memlog.md` as a canonical decision and preservation record, but never regard it as a downstream contract companion.
- Semantic equivalence must search the full main package—including its companions and memlog—before classifying an independent claim as missing.
- A claim appearing in both independent packages is stronger review signal, not truth. Evidence against original requirements controls its classification.
- Structural-only companion organization differences are audit-only by default and cannot enter the merge summary unless the user explicitly turns one into a substantive item.
- User edits to a proposal must retain both original provenance and edited wording in the ledger.
- Pending merge must be reproducible from final ledger state; the agent must detect and invalidate/rebuild it if any ledger item changes before confirmation.
- An independent run is evidence only after it completes with its required package artifacts and passes `bmad-spec`’s recorded Spec Law validation. A failed or incomplete run is `not-comparable`, never a source of difference findings; retry only that run and preserve all other review state.
- Treat post-merge verification as mandatory. Compare the before/after main-package inventory and validate accepted items, resulting `open_question`s, existing CAP IDs, companion frontmatter references, and unexpected deltas before declaring the review complete.

**Relationships:** Runs after a user has created or updated a main package with `bmad-spec`. It invokes `bmad-spec` twice for isolated comparison generation and once after final confirmation for the approved merge. `bmad-spec` remains the parent module and main-contract writer.

---

## Configuration

| Variable | Prompt | Default | Result Template | User Setting |
| --- | --- | --- | --- | --- |
| `review_output_path` | “Where should BMad Buddy store persistent SPEC-review workspaces?” | `_bmad-output/spec-reviews/` | `review_output_path = "{value}"` | Yes — collected and persisted during setup. |

`independent_run_count` is not an installation setting. It defaults to `2`, may be overridden for an individual review, and is rejected if less than `2`.

`generate_html_report` is not an installation setting. It defaults to `true` and may be disabled for an individual review.

The first release supports interactive use only; it does not offer a headless or CI mode.

## External Dependencies

The module depends on the installed `bmad-spec` skill and the standard BMad runtime it already uses (including `uv` and its project scripts). No external CLI, MCP server, database, or hosted web service is required.

Setup must verify that `bmad-spec` is available and explain that BMad Buddy cannot independently generate or safely merge SPEC packages without it.

## UI and Visualization

The primary interface is a conversational, item-by-item review with explicit user decisions. `review-report.md` is the default report for terminal, Git, and agent use.

When enabled, `review-report.html` is a static enhancement—not a required app or source of state. It groups candidate omissions, conflicts/ambiguities, and structural-only differences; shows source evidence, provenance, and current disposition; and links every view back to ledger item IDs. The ledger remains authoritative.

## Setup Extensions

Beyond normal module configuration, setup should:

1. Verify `bmad-spec` is installed and accessible.
2. Collect and persist `review_output_path`.
3. Create the configured review-output directory only when a review first runs (or explicitly when setup validation requires it).

## Integration

**Parent capability:** `bmad-spec`.

**Before BMad Buddy:** the user runs `bmad-spec` to create or update the main SPEC package from their requirements. The main package’s `.memlog.md` is the decision-of-record.

**During BMad Buddy:** the agent invokes `bmad-spec` twice from the frozen supplied requirements to create independent comparison packages. It compares the two packages with the main package and guides human review.

**After BMad Buddy:** after one explicit final confirmation, it invokes `bmad-spec` to append the confirmed decisions and/or open questions to the main `.memlog.md`, then re-derives the main `SPEC.md` and spec-authored companions. It never hand-edits derived artifacts.

## Creative Use Cases

Not ready — complete in Phase 3+.

## Ideas Captured

<!-- Raw ideas from brainstorming — preserved for context even if not all made it into the plan -->
<!-- Write here freely during phases 1-2. Don't write structured sections until phase 3+. -->

- Problem: a SPEC.md generated by `bmad-spec` can omit capabilities, constraints, or other important requirements. Manual review makes these omissions hard to spot.
- Core idea: generate multiple independent SPEC.md artifacts in isolated contexts, then compare them. Differences across independently reasoned outputs make likely omissions visible and easier to review.
- Inspiration: multi-agent debate / consensus, particularly Du et al. (2023), “Improving Factuality and Reasoning in Language Models through Multiagent Debate.” Multiple agents approach the problem from different perspectives, scrutinize one another, and aggregate toward a more reliable result than a single execution.
- Desired outcome: a rigorous, reviewable way to surface missing requirement categories in a SPEC—not merely another free-form critique.
- Identity: module display name is “BMad Buddy”; module code is `bbu`. It will avoid the reserved `bmad-` skill prefix.
- Execution model: “independent environments” means multiple independent runs in the same workspace, not separately provisioned agent environments. Independence must therefore be designed into context, prompts, and artifacts.
- Primary journey: the user first creates a main SPEC package with `bmad-spec`. BMad Buddy then calls `bmad-spec` multiple times with the user-provided requirements to independently generate comparison SPEC packages.
- Comparison scope: every independent package is compared with the main package, including `SPEC.md`, all discovered companions, and `.memlog.md`.
- Review output: BMad Buddy should identify candidate omissions—especially missing Capabilities and Constraints—rather than treating textual difference alone as an error.
- Resolution: a human confirms the candidate omissions. Confirmed content is then merged into the final main SPEC package.
- `bmad-spec` contract insight: `.memlog.md` is the canonical append-only record; SPEC.md and spec-authored companions are derived artifacts. Therefore BMad Buddy must not hand-edit the main SPEC or its spec-authored companions. It should add only human-approved findings to the main package’s memlog through the supported `bmad-spec` update path, then re-derive the package. Companions must be discovered from `SPEC.md` frontmatter, while the memlog is compared for preservation and decision-record coverage.
- Run count: create two independent comparison SPEC packages by default. Together with the existing main package, the review covers three SPEC packages.
- Report class 1 — candidate omissions: a claim exists in one or both independent packages but is absent from the main package. Each item shows its category (capability, constraint, non-goal, success signal, companion content, or memlog decision), which independent runs contain it, a source-text summary, and the reasoning for judging it missing from the main package.
- Report class 2 — conflicts / ambiguities: main and independent packages cover a topic but differ in wording, scope, or constraint. The report requires the user to choose an option, clarify it, or explicitly record it as an open question.
- Report class 3 — structural-only differences: companion contents are organized differently without evidence of missing contract content. These are audit-only and are not merged by default.
- Human-review workflow: (1) user reviews every candidate; (2) for each, selects accept, reject, or needs clarification and may edit wording or scope; (3) BMad Buddy creates a pending merge summary; (4) user confirms that summary once; (5) BMad Buddy calls `bmad-spec` to append confirmed items to the main `.memlog.md` and re-derive SPEC.md and companions.
- Priority risk 1: avoid false omissions when the main package already expresses the same content through different wording, a companion, or a memlog entry. Comparison must prioritize semantic coverage and evidence tracing over text diff.
- Priority risk 2: two independent packages can share the same unsupported assumption. Repetition is a signal for review, not evidence that the claim is true or should enter the main contract.
- Priority risk 3: nothing enters the main package without explicit human confirmation. The final confirmation is a hard write gate, not a formality.
- Priority risk 4: a conflict marked “needs clarification” cannot silently disappear. It requires durable, visible tracking through finalization so the resulting contract does not conceal unresolved ambiguity.
- Positioning: BMad Buddy is an extension module for `bmad-spec`. It may audit an existing SPEC package, while independent generation and approved merging depend on `bmad-spec`.
- Evidence requirement: every candidate must be traceable to the original requirements and receives an evidence grade. **Directly supported** items cite a source passage. **Reasonable inferences** explain the inference chain while acknowledging the requirement did not state them explicitly. **Unsupported assumptions** may appear only as a discussion hypothesis or `open_question`; they must never be labeled an omission.
- Clarification resolution gate: every item marked “needs clarification” must be classified before final confirmation as exactly one of: clarified and merged; recorded as an `open_question` in the main memlog; or rejected/abandoned with a reason. The workflow cannot final-confirm while any such item is unclassified.
- Primary review experience: conversational, item-by-item review backed by a persistent review ledger. An HTML report is an enhancement, not the sole user interface.
- `review-ledger.yaml`: the single source of truth for review state. It stores each item’s status, edited text, evidence, rationale, clarification state, and final-confirmation record.
- `review-report.md`: the default readable report for Git, terminals, and agent conversations.
- `review-report.html`: an optional high-readability review view that groups items and shows their evidence and current state.

## Build Roadmap

Not ready — complete in Phase 3+.

**Next steps:**

1. Build each skill using **Build an Agent (BA)** or **Build a Workflow (BW)** — share this plan document as context
2. When all skills are built, return to **Create Module (CM)** to scaffold the module infrastructure

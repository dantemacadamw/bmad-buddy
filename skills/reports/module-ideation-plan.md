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
updated: '2026-08-19T09:30:00+08:00'
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

Not ready — complete in Phase 3+.

### {skill-name}

**Type:** {agent | workflow}

**Persona:** <!-- For agents: who is this? Communication style, expertise, personality -->

**Core Outcome:** <!-- What does success look like? -->

**The Non-Negotiable:** <!-- The one thing this skill must get right -->

**Capabilities:**

| Capability | Outcome | Inputs | Outputs |
| ---------- | ------- | ------ | ------- |
|            |         |        |         |

<!-- For outputs: note where HTML reports, dashboards, or structured artifacts would add value -->

**Memory:** <!-- What does this agent read on activation? Write to? Daily log tag? -->

**Init Responsibility:** <!-- What happens on first run? Shared memory creation? Domain onboarding? -->

**Activation Modes:** <!-- Interactive, headless, or both? -->

**Tool Dependencies:** <!-- External tools with technical specifics -->

**Design Notes:** <!-- Non-obvious considerations, the "why" behind decisions -->

---

## Configuration

Not ready — complete in Phase 3+.

| Variable | Prompt | Default | Result Template | User Setting |
| -------- | ------ | ------- | --------------- | ------------ |
|          |        |         |                 |              |

## External Dependencies

Not ready — complete in Phase 3+.

## UI and Visualization

Not ready — complete in Phase 3+.

## Setup Extensions

Not ready — complete in Phase 3+.

## Integration

Not ready — complete in Phase 3+.

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

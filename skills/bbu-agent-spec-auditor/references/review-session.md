---
name: review-session
description: Starts or resumes an isolated BMad Buddy SPEC review.
code: RS
added: 2026-08-19
type: prompt
---

# Start or Resume a Review

The outcome is a review workspace whose provenance is strong enough that a later user can trust what was compared and resume without reconstructing decisions from conversation. The ledger is the consumer’s authority: it must distinguish the unchanged main package from frozen inputs and from each independent run.

Require an existing main `bmad-spec` package and the original requirements plus all source references used to create it. If either is unavailable, explain why the review cannot establish omissions and help the user locate or recreate the missing input. Do not substitute the main package as the source requirements.

Create or resume `{review_output_path}/{spec-slug}/{review-id}/`, containing `inputs/`, `independent-run-1/`, `independent-run-2/`, `review-ledger.yaml`, `review-report.md`, optional `review-report.html`, and `pending-merge.md`. The main package is read in place and is never modified during review preparation. Record immutable input paths or copies, content identities when available, main-package path, each run path, settings, and status in the ledger before generating evidence. Use the schema in `references/review-ledger.md`.

The independent packages are two fresh invocations of `bmad-spec` from the same frozen source input. Neither may receive the main package, the other run’s output, or prior comparison results. Use distinct run folders and record invocation provenance. The default run count is two; permit a per-review override only when it is at least two.

Treat a run as comparable only when it contains `SPEC.md`, `.memlog.md`, and every companion referenced by `SPEC.md` frontmatter, and when its memlog records `bmad-spec`’s Spec Law validation verdict. A failed, incomplete, or invalid package is `not-comparable`, cannot supply findings, and is not silently replaced. Preserve successful runs and let the user rerun only the affected run; record the failure, retry lineage, and replacement provenance in the ledger.

When the required two runs are comparable, hand off to `references/evidence-review.md`. If the user returns to an existing workspace, load the ledger first, report its exact state, and continue from its next unresolved obligation rather than re-running completed work.

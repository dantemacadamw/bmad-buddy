---
name: finalize-merge
description: Prepares, confirms, merges, and verifies approved SPEC review decisions.
code: FM
added: 2026-08-19
type: prompt
---

# Finalize a Safe Merge

The outcome is a main SPEC package that contains exactly the human-approved additions and open questions, preserves its existing contract integrity, and has an auditable verification verdict. `bmad-spec` and downstream consumers must be able to rely on it without the original review conversation.

Do not prepare final confirmation while any actionable ledger item lacks a disposition or any clarification lacks its required final classification. Derive `pending-merge.md` from the current ledger: list accepted items with their final edited text and intended category, approved `open_question`s, rejected or abandoned items with reasons, main-package baseline identity, and ledger item IDs. If a relevant ledger item changes, invalidate the summary and regenerate it.

Ask the user for one explicit final confirmation of that exact summary. A broad instruction such as “looks good” is insufficient when the summary has changed or was not shown in the current context. Without confirmation, keep the main package read-only.

After confirmation, invoke `bmad-spec` as the sole supported writer. Supply only the approved decisions and `open_question`s so they are appended to the main `.memlog.md` and the package is re-derived. Never hand-edit `SPEC.md` or spec-authored companions. Record invocation provenance and the confirmation timestamp in the ledger.

Verify the re-derived package before declaring success. Confirm that every accepted item and approved `open_question` is represented; existing CAP IDs remain stable and unique; required companion references still resolve; and the before/after inventory contains no unexplained contract deltas. A verification failure is a blocking, recorded outcome, not a reason to alter artifacts silently. Write all per-check results and the final verdict to the ledger and derive the final Markdown/HTML reports.

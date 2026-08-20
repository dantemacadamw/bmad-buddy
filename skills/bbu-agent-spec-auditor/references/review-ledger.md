# Review Ledger Contract

`review-ledger.yaml` is the sole source of review state. Markdown and HTML reports are derived from it; no acceptance, clarification, retry, or merge can be inferred from prose alone.

The ledger must preserve the following fields, using stable IDs and append-only history for decisions that are superseded:

```yaml
review:
  id: review-<timestamp-or-stable-id>
  status: initialized | generating | comparing | reviewing | ready-for-confirmation | confirmed | merged | verification-failed | blocked
  created_at: <timestamp>
  updated_at: <timestamp>
  settings:
    independent_run_count: 2
    generate_html_report: true
  main_package:
    path: <path>
    identity: <content-or-version-identity>
  frozen_inputs:
    - path: <path-or-copy>
      identity: <content-or-version-identity>
  runs:
    - id: run-1
      path: <path>
      status: pending | running | comparable | not-comparable | superseded
      provenance: <bmad-spec invocation and source identity>
      validation: <artifact and Spec Law verdict>
      retry_of: <run-id-or-null>
items:
  - id: RVI-001
    class: candidate-omission | conflict-ambiguity | structural-only | discussion-hypothesis
    category: capability | constraint | non-goal | success-signal | companion-content | memlog-decision
    source_claims: []
    main_coverage_assessment: <reasoned semantic coverage check>
    evidence:
      grade: directly-supported | reasonable-inference | unsupported-assumption
      citations: []
      inference_chain: <required for reasonable inference>
    proposed_change:
      original: <text>
      edited: <user-approved text-or-null>
      target: <kernel-field | companion | memlog | open-question>
    disposition: pending | accepted | rejected | needs-clarification | open-question | abandoned
    rationale: <user decision reason>
    clarification_resolution: merged | open-question | abandoned | null
    history: []
final_confirmation:
  pending_merge_identity: <identity-or-null>
  confirmed_at: <timestamp-or-null>
  confirmed_by: <user-or-null>
merge:
  bmad_spec_invocation: <provenance-or-null>
  verification:
    accepted_items_present: pending | pass | fail
    open_questions_present: pending | pass | fail
    cap_ids_preserved: pending | pass | fail
    companion_references_intact: pending | pass | fail
    unexplained_deltas: pending | pass | fail
    verdict: pending | pass | fail
```

An unsupported assumption may be retained only as `discussion-hypothesis` or converted by the user into an `open-question`; it cannot have `disposition: accepted` as a candidate omission. `ready-for-confirmation` is valid only when every actionable item has a terminal disposition and every former clarification has a non-null resolution.

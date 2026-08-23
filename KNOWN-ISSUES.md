# Known Issues

## Judgment-call steps are spec-verified, not live-model-verified

Source: `EXECUTION-AUDIT.md` run, 2026-08-23 (see that file's scenario D and
F). Two steps in `SKILL.md` depend on a live model making a judgment call
mid-session, not just following deterministic instructions:

- **E6 (ERRORS.md conditional write)** — "only write if something failed
  2+ times or a non-obvious root cause was found." The audit confirmed the
  instructions are followable and internally consistent (a hand-executed
  run correctly wrote ERRORS.md for a real failure narrative and correctly
  skipped it for a trivial session). It did **not** test whether a live
  model, under real multi-turn session pressure, will over-trigger this
  (writing noise entries) or under-trigger it (missing a genuine failure).
- **S1's "drop if 3+ others have signal" threshold** for never-active
  projects in the multi-project scan — same caveat: spec-correct on paper,
  unverified against a live model's counting/judgment in an unusual
  project-count scenario.

Neither of these is fixable by editing the spec further — they're inherent
to delegating a judgment call to a model rather than a deterministic rule.
The mitigation is empirical: run a handful of real, un-scripted
`/session-memory end` sessions (not a scripted audit) and check whether
ERRORS.md entries actually match the "genuinely non-obvious failure" bar in
practice, before treating this as fully validated.

**Status**: not blocking publish — the deterministic control flow (mode
dispatch, config resolution, doc-path overrides, marker-based recency,
multi-project scan/sort, explicit-target bypass, no-upstream handling) was
fully verified across all 8 audit scenarios with no divergence. This is a
"watch in practice" item, not a "known bug."

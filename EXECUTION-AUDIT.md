# Execution Audit — claude-session-memory

Purpose: dry-run the `session-memory` skill (both START and END modes)
end-to-end before publishing, in a session that has no memory of how it was
built. Work through scenarios in order — later ones depend on state created
by earlier ones. Record actual output next to each expected result; anything
that diverges is a bug to fix before publish.

Setup: copy the `skills/session-memory` folder into a scratch directory's
`.claude/skills/`, e.g.
`C:\Users\vikas\AppData\Local\Temp\claude-session-memory-audit\`. Do not test
inside `D:\plotpix\*` or any real project — use throwaway folders so nothing
here touches real data.

---

## Scenario A — Single-project, git, first run

1. `mkdir` a fresh git repo (`git init`), no MEMORY.md/ERRORS.md/BACKLOG.md.
2. Run `/session-memory start`.
   - **Expect**: single-project mode (no global config exists). Reports "No
     session history found — this looks like the first run." Ends with
     "Ready. What are we working on today?" No files created.
3. Make a trivial change, "work" for a minute (any conversation).
4. Run `/session-memory end`.
   - **Expect**: E1a fires — AskUserQuestion about the cross-project log.
     Answer **No** for this scenario. `MEMORY.md` created with one session
     block. `ERRORS.md` NOT created (nothing failed).
     `.claude/session-memory.json` written as `{}`.
     `.claude/session-memory/.last-active` written with an epoch timestamp.
     Confirm block printed listing MEMORY.md and .last-active as updated,
     ERRORS.md as skipped.
5. Run `/session-memory end` again immediately (simulate a second short session).
   - **Expect**: E1a does NOT fire again (config file already exists). A
     second `## Session —` block appended, first one untouched.

**Pass criteria**: no file errors, `.last-active` is a plain epoch number,
config file has valid JSON, MEMORY.md has two clean session blocks.

---

## Scenario B — Single-project, non-git, first run

1. `mkdir` a plain folder, no `git init`, no docs.
2. Run `/session-memory start`.
   - **Expect**: reports "No git repo detected — using CWD as project root."
     No branch field errors. Same first-run message as Scenario A.
3. Run `/session-memory end`, this time answer **Yes** to the cross-project-log
   question.
   - **Expect**: `~/.claude/session-memory/MEMORY.md` created (and its
     parent dir), with a `## Recent Sessions` heading and one bullet
     prepended. Local `.claude/session-memory.json` now contains
     `{"globalMemory": "~/.claude/session-memory/MEMORY.md"}`.
     `.last-active` written despite no git repo.
     Confirm block should NOT mention "Branch" as an error — should show
     "no git" cleanly.

**Pass criteria**: no command in either mode assumes git is present; global
memory file and marker both land correctly for a non-git folder.

---

## Scenario C — Config overrides

1. In a fresh repo (git, doesn't matter), before running either mode,
   hand-write `.claude/session-memory.json`:
   ```json
   { "docs": { "memory": "LOG.md", "errors": "ISSUES.md", "backlog": "TODO.md" } }
   ```
2. Run `/session-memory end`.
   - **Expect**: writes `LOG.md`, not `MEMORY.md`. E1a is skipped (config
     already exists, even without `globalMemory` set). No `MEMORY.md` created
     at all.
3. Run `/session-memory start`.
   - **Expect**: reads `LOG.md` (via the configured path), not `MEMORY.md`.

**Pass criteria**: doc filename overrides are respected end to end, in both
modes, without falling back to defaults.

---

## Scenario D — Errors doc conditional write

1. In a scenario-A-style repo, deliberately have a session where you (the
   tester) tell the assistant: "I tried X twice and it failed both times
   before Y worked" — a real failure narrative in conversation.
2. Run `/session-memory end`.
   - **Expect**: `ERRORS.md` IS created this time, with the What
     failed/Root cause/What worked/Note structure. Compare against a
     trivial session (Scenario A) where it correctly was NOT created.

**Pass criteria**: the conditional logic actually discriminates — this is
the one most likely to be over- or under-triggered by a real model, so check
it doesn't write ERRORS.md on every session regardless of content.

---

## Scenario E — Multi-project scan, mixed git/non-git

1. Create three scratch project folders: two git repos (`proj-git-a`,
   `proj-git-b`), one plain folder (`proj-plain`).
2. Run `/session-memory end` once inside each of the three (any trivial session),
   staggered a few minutes apart so recency actually differs. Answer the
   first-run question however; doesn't matter for this test.
3. Hand-write `~/.claude/session-memory/config.json` (see
   `examples/global-config.example.json`) listing all three paths — OR skip
   hand-writing it and instead confirm E8a auto-registered them: check
   whether the file already has all three after step 2, since Scenario E
   assumes multi-project mode was already on (see note below).
   - **Note**: auto-register (E8a) only fires if the global config file
     *already exists* before `/session-memory end` runs. For a clean test of
     auto-register, create `~/.claude/session-memory/config.json` as
     `{"projects": []}` BEFORE step 2, then verify each
     `/session-memory end` run adds its project.
4. From any directory, run `/session-memory start` (no path arg).
   - **Expect**: multi-project mode triggers (3 projects listed). An
     AskUserQuestion shortlist appears, sorted most-recent-first, each
     labeled correctly:
     - git projects with a marker: "last session <relative-time>" or a
       commit-based label — check which one actually renders and whether
       it's the marker (should take priority per spec) not the commit time.
     - the non-git project: "last session <relative-time>" from its marker.
   - Select two of the three (multiSelect).
   - **Expect**: one full briefing block per selected project, correct
     project name/branch header each, no cross-contamination of docs
     between them.

**Pass criteria**: sort order matches actual recency; non-git project is
neither dropped nor mislabeled; multiSelect produces multiple correctly
separated briefing blocks.

---

## Scenario F — Multi-project scan, one project never active

1. Add a fourth project to the global config that has no marker and no git
   commits (freshly `git init`'d, or a non-git empty folder) — never had
   `/session-memory start` or `/session-memory end` run in it.
2. Run `/session-memory start`.
   - **Expect**: this project is either excluded from the top-3 shortlist
     (if 3+ others have real signal) or shown labeled "no activity yet" —
     check which behavior the skill actually produces and confirm it
     matches the intent in SKILL.md (drop only if 3+ others have a real
     signal).

**Pass criteria**: a with-no-signal project never silently outranks an
active one, and doesn't crash the sort (missing epoch handled, not treated
as string-sort garbage).

---

## Scenario G — Explicit path/name argument bypasses scan

1. With multi-project mode already configured (from Scenario E/F), run
   `/session-memory start proj-git-a` (by name) and separately
   `/session-memory start <full-path-to-proj-plain>` (by path).
   - **Expect**: no AskUserQuestion shown either time — goes straight to
     that project's briefing.

**Pass criteria**: both name-match and raw-path forms work; no scan
triggered when an explicit target is given.

---

## Scenario H — Pull behavior doesn't block on a local-only branch

1. In a git repo with no remote configured at all (`git init` only, no
   `git remote add`), run `/session-memory start`.
   - **Expect**: no pull attempted, no error surfaced to the user, briefing
     still produced normally.

**Pass criteria**: `NO_UPSTREAM` path is silent, not an error message.

---

## Known gaps to note during the audit (not bugs, just check they're still true)

- `BACKLOG.md`/`TODO.md` (whatever configured) is never auto-created —
  confirm `/session-memory end` really skips it silently when absent, per spec,
  rather than inventing one.
- Confirm the global cross-project `MEMORY.md` never gets *rewritten* — only
  ever prepended to — even after several sessions across several projects
  (i.e., check no earlier bullet from a different project got clobbered).
- Confirm `/session-memory start` truly never writes any file, including in
  multi-project mode (no accidental config touch) — grep the whole scratch
  tree's mtimes before/after a `/session-memory start` run.

---

## After the audit

Log real findings back into this repo (not into personal memory) — either
fix the SKILL.md directly for anything that diverged from spec, or add a
`KNOWN-ISSUES.md` if something is a deliberate deferred limitation. Only
then is this ready to push to GitHub.

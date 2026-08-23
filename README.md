# claude-session-memory

One Claude Code skill — `session-memory` — that gives a project persistent,
disciplined memory across work sessions. Two modes, one skill: START
(read-only briefing) and END (cascade doc updates).

For a visual walkthrough of the architecture (the problem it solves, the
file layout, how both modes run, and the design decisions), see
[`docs/overview.html`](docs/overview.html) — open it in a browser, or view
via [raw.githack.com](https://raw.githack.com/ssupppp/claude-session-memory/master/docs/overview.html).

## What this actually does

Not "AI memory." Two things, both about **efficiency**, not novelty:

**1. Routing, not dumping.** When you close out a session, the skill decides
which of several small docs each fact belongs in (project log, cross-project
log, backlog, error log) and writes only what changed — no manual triage, no
single file that grows forever. When you open a new session, it reads those
docs back and gives you a tight "where did I leave off" briefing in one
command instead of five file reads.

**2. Distributed context instead of one global dump.** The common pattern is
a single global `CLAUDE.md` (or memory file) that every project's context
gets crammed into, growing without bound and getting loaded in full on every
session regardless of relevance — you pay the token cost for every other
project's history just to work on this one. This skill splits that into
per-project `MEMORY.md`/`ERRORS.md`/`BACKLOG.md` files that only get read
when `session-memory` (START mode) runs *in that project* — plus one thin, optional
cross-project index (`globalMemory`) that holds one-line pointers, not full
history. Context loads on demand, scoped to what's relevant, instead of
being dumped in bulk every time.

Run `session-memory` (END mode) enough times and the docs become a compounding record your
next session (or next week's session) can pick up from cold, correctly —
without either re-explaining everything or paying to load everything.

## Install

Copy the `skills/session-memory` folder into your project's
`.claude/skills/` directory (or your global `~/.claude/skills/` to use it
everywhere).

## Use

- **Start of a session** — say "start session" / "what was I working on", or
  run `/session-memory start`. Read-only, briefs you on where you left off.
  Bare `/session-memory` with no other signal also defaults to this mode
  (it's always safe to default to the read-only side).
- **End of a session** — say "session end" / "wrapping up", or run
  `/session-memory end`. Writes a session record to the right files.

Zero config required. First END run creates `MEMORY.md` and `ERRORS.md` (as
needed) at your project root, and asks once whether you also want a
cross-project log — see [Configuration](#configuration).

## What gets written where

| File | Created by | Purpose |
|---|---|---|
| `MEMORY.md` (project root) | auto, first `session-memory` (END mode) run | Per-project session history |
| `ERRORS.md` (project root) | only when something genuinely failed | Gotchas / root causes, so you don't re-debug the same thing |
| `BACKLOG.md` (project root) | never auto-created — opt-in | Only touched if you already have one |
| `~/.claude/session-memory/MEMORY.md` | only if you opt in | One-line-per-session log across *all* your projects |

## Configuration

Optional. Drop a `.claude/session-memory.json` in your project root to
override defaults:

```json
{
  "globalMemory": "~/.claude/session-memory/MEMORY.md",
  "docs": {
    "memory": "MEMORY.md",
    "errors": "ERRORS.md",
    "backlog": "BACKLOG.md"
  }
}
```

- `globalMemory` — path to a cross-project log shared across repos that opt
  in. `~` expands to your home directory. Omit to keep this project's log
  separate (default).
- `docs` — rename/relocate any of the three per-project files.

If no config file exists, `session-memory` (END mode) asks once on its first run whether
you want the cross-project log, then writes the config so it never asks
again.

## Multi-project mode

If you work across several projects, `session-memory` (START mode) can scan all of them —
git and non-git alike — sorted by recent activity, and let you pick which
one(s) to brief on.

Opt in by creating `~/.claude/session-memory/config.json`:

```json
{
  "projects": [
    { "name": "my-app", "path": "~/code/my-app" },
    { "name": "notes-vault", "path": "~/notes" }
  ]
}
```

(See `examples/global-config.example.json`.) List any mix of git and
non-git folders. Nothing creates this file automatically — you opt in
explicitly, once.

**Recency sort works for non-git projects too.** Every `session-memory` (END mode) run
writes a small marker (`<project>/.claude/session-memory/.last-active`, just
an epoch timestamp) regardless of git status — that's the primary recency
signal. A git project that's never been closed with `session-memory` (END mode) yet falls
back to its last commit time. A project with neither is shown as "no
activity yet" and sorts last.

Once you're in multi-project mode, `session-memory` (END mode) also auto-registers new
projects into the global list the first time you close a session in them —
so you don't have to hand-edit the config for every new project, only to
enable the mode once.

## Design notes

- **Read-only vs. write split**: `session-memory` (START mode) never writes anything.
  `session-memory` (END mode) is the only place state changes. This keeps the briefing
  command safe to run as often as you like.
- **Selective writes**: `ERRORS.md` only gets an entry if something actually
  failed non-obviously — no noise entries. `BACKLOG.md` is never invented for
  you; it's assumed to be a doc you already maintain.
- **No hardcoded paths**: project root is `git rev-parse --show-toplevel`,
  falling back to CWD for non-git projects. Nothing here assumes a specific
  OS, drive letter, or directory layout.
- **Works without git**: git is used opportunistically (pull latest, show
  branch, recent commits) but nothing in either mode requires it. A non-git
  folder (a notes vault, a data-analysis workspace, a prompt library) gets
  the same doc-routing behavior, just without the branch/commit fields.

## Known issues

See `KNOWN-ISSUES.md` — one residual item (judgment-call steps verified
against the spec, not yet against a live model in unscripted use).

## License

MIT

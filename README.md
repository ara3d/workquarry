# WorkQuarry

**Help agents break big rocks up into small rocks.**

A file-based work tracker for repositories worked by AI coding agents. Markdown files with
YAML frontmatter as the database, one small dependency-free Python script as the only
mutation path, and a handful of agent skills that elaborate an idea or a bug instead of just
filing a title. No server, no account, no SaaS. Everything versions with your code.

This README is an honest assessment, not a pitch: what it's good for, where it's untested,
what's genuinely novel versus reinvented, and who shouldn't use it yet.

## What problem it actually solves

Agent-worked repos accumulate dated plan docs and scratch TODOs that rot: nothing marks a
plan as finished, so an agent re-executes stale work or silently drops an out-of-scope
finding it noticed mid-task. WorkQuarry's one real idea is a test for what deserves a
lifecycle-tracked record versus a plain doc:

> **Does it have to change state or can it rot?** If it must move through a status (idea →
> ready → in-progress → done), it's a *record* and lives in `tracker/`. If it's finished the
> moment it's written (a design writeup, a transcript), it's a *doc* and lives elsewhere.

Everything else — issue files, a generated backlog, an append-only done-log, ADRs, agent
rules — falls out of that one distinction.

## Strengths

- **Doc-vs-record test** is a genuinely crisp answer to a fuzzy problem most repos never
  name explicitly. It's the one idea here that isn't just "GitHub Issues, but files."
- **Plan-doc lifecycle** (an `EXECUTED <date>` banner + move to an archive folder) directly
  targets the failure mode of agents re-running stale plans — a problem specific to
  agent-worked repos that human-oriented trackers don't address.
- **Architecture is sound and small.** Frontmatter = source of truth, `BACKLOG.md` = generated
  view, `DONE.md` = append-only event log, one script (`track.py`) as the only write path.
  ~250 lines, zero third-party dependencies, easy to read end to end in five minutes.
- **The elaboration skills are the real differentiator**, more than the tracker mechanics.
  `/track-idea` and `/track-issue` force a related-work search before filing, then draft
  assumptions, design forks, and a "simplest implementation + what you give up" section. That
  prompt discipline is useful even bolted onto GitHub Issues or Linear — it doesn't need this
  file format to be valuable.
- **Sprint-as-parallel-working-set with an area-collision check** assumes multiple *agents*
  work concurrently, not multiple humans on a calendar. That framing is underexplored
  elsewhere and is the part most clearly built for 2026-era agent teams rather than adapted
  from human project management.
- **`problem` → ADR closure** gives open design questions an explicit, first-class type with
  its own closing action, instead of forcing them into `bug`/`feature`.

## Weaknesses and untested claims

- **Single-writer by construction.** `next_num()` is `max(existing) + 1` — two branches
  filing concurrently collide on id. `BACKLOG.md` is a single generated file — any two
  branches that both touch the tracker will merge-conflict on it. This is fine for one person
  plus their agents on a mainline branch (the design point); it is not fine for a team without
  changes (collision-safe ids, a merge-friendly index format, an assignee field — none of
  which exist yet).
- **Unproven at scale and over time.** The system this was extracted from is roughly a few
  days old as of this writing. The architecture's own stated risk — "weekly triage or the
  backlog becomes the new rot" — has not yet been tested by actually failing to triage for a
  month and seeing what breaks. Claims of rot-resistance are the ADR's stated intent, not yet
  demonstrated behavior.
- **Regex frontmatter parsing, not a YAML library.** Deliberate (zero dependencies), but it
  means the schema must stay flat and simple; anything nested or exotic in frontmatter will
  not round-trip correctly.
- **Discipline-dependent, not enforced.** Nothing currently checks that triage actually
  happens, that closed issues get banners, or that a vendored copy of this mechanism hasn't
  drifted from its source (a `check` command exists in `install.py` for the drift case, but
  nothing runs it automatically — no CI, no hook, by default).
- **Skills are Claude Code-specific.** `track.py` and the process itself are agent-agnostic
  (any agent that can Read/Write/Grep can use them), but `/track-idea`, `/track-issue`,
  `/track-backlog` rely on Claude Code's `.claude/skills/` discovery mechanism specifically.
- **No UI.** Everything is CLI + file reads. Fine for agents and terminal-first developers;
  a non-technical stakeholder gets nothing here.

## Prior art — this is not a new idea

In-repo, markdown-based, agent-friendly trackers are an active space as of 2026, not a gap
WorkQuarry discovered. Known comparable projects (verify current state before citing
publicly — this list may be stale):

- **Backlog.md** — markdown tasks in-repo with a CLI and a kanban web UI. Closer to a
  human-facing tool with a nicer UX layer; WorkQuarry has no UI at all.
- **beads** — a git-backed, agent-first issue database (Steve Yegge). Closest conceptual
  sibling — built explicitly for coding agents rather than adapted from human PM tools.

WorkQuarry's honest claim to distinctiveness is narrow: the doc-vs-record test, the
plan-doc-lifecycle banner convention, `problem`→ADR closure, and the elaboration skills. The
underlying "markdown files + frontmatter + generated index" pattern itself is not novel and
should not be presented as such.

## Use cases it fits today

- A solo developer (or small team on a single mainline branch, no parallel tracker edits)
  working with one or more coding agents, who wants tracked work to live and version with the
  code instead of in a separate web app.
- A repo already suffering from dated-plan-doc rot, where the doc-vs-record test and the
  EXECUTED-banner convention would directly fix a known failure mode.
- Anyone who wants the `/track-idea` / `/track-issue` elaboration prompts and is willing to
  adapt them to their own tracker, even without adopting the file format.

## Use cases it does not fit yet

- Any team where more than one person/agent-session commits to the tracker on parallel
  branches — id collisions and `BACKLOG.md` merge conflicts are unresolved.
- Anyone who needs a UI, notifications, or reporting beyond `git grep` and a CLI list command.
- Anyone who needs the tracked-work history to predate this extraction — issue numbering
  restarts fresh per repo.

## What's in this repository

| Path | Installs to (in a consuming repo) | Role |
|------|-----------------------------------|------|
| `track.py` | `tools/track.py` | the one mutation path (new/set/list/close/index) |
| `process.md` | `tracker/readme.md` | the process spec agents read |
| `templates/` | `tracker/{issues,decisions}/_template.md` | issue + ADR shapes |
| `skills/` | `.claude/skills/track-*` | Claude Code skills that elaborate, not just file |
| `AGENTS-snippet.md` | a marker-delimited block in `AGENTS.md` | the non-negotiable agent rules |
| `tracker/config.example.json` | `tracker/config.json` | your areas + paths (the only local config) |
| `install.py` | — (run directly) | copies the above into a target repo, checks for drift |

Your actual issues, backlog, done-log, and ADRs are instance data. They live in the
*consuming* repo, never in this one.

## Install (in a consuming repo)

```sh
git submodule add https://github.com/ara3d/workquarry submodules/workquarry
python submodules/workquarry/install.py
# then edit tracker/config.json — set your own areas
```

Update after pulling a new version:

```sh
git submodule update --remote submodules/workquarry
python submodules/workquarry/install.py
python submodules/workquarry/install.py --check   # verify nothing drifted
```

Vendored copies carry a `GENERATED` banner. Edit the mechanism here, in this repo, and
re-run the installer — never hand-edit the copies.

## License

MIT © 2026 Christopher Diggins / Ara 3D

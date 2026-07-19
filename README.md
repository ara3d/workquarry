# WorkQuarry

**Help agents break big rocks up into small rocks.**

A skill-set and process for helping AI coding agents manage work. The mechanism underneath is
deliberately small: markdown files with YAML frontmatter as the database, one dependency-free
Python script as the only mutation path, and agent skills that *elaborate* an idea or a bug —
search for prior art, draft the design forks, propose the simplest implementation — instead of
just filing a title. No server, no account, no SaaS. Everything versions with your code.

This README is an honest assessment, not a pitch: what it's good for, where it's untested,
what's genuinely novel versus reinvented, and who shouldn't use it yet.

## At a glance

One file per tracked item; frontmatter is the source of truth:

```yaml
---
id: studio-010
title: Orbit camera
type: feature          # feature | debt | bug | idea | problem | retire
status: ready          # idea | ready | in-progress | done | dropped
priority: p1           # p1 | p2 | p3 | ?
effort: S              # S | M | L | ?
risk: low              # low | med | high | ?
area: studio           # your areas, from tracker/config.json
sprint:                # working set name, empty = not selected
created: 2026-07-16
closed:
links: [docs/camera-notes.md, studio-001]
---
Motivation, acceptance criteria, pointers to discussion.
```

`BACKLOG.md` is a generated one-line-per-issue index (never hand-edited); `DONE.md` is an
append-only log of outcomes; `decisions/` holds immutable, dated ADRs. All writes go through
the script:

```sh
python tools/track.py new --title "Orbit camera" --area studio --type feature
python tools/track.py set studio-010 --status ready --priority p1 --effort S --risk low
python tools/track.py list --open --sort priority
python tools/track.py close studio-010 --outcome "done (a1b2c3d)"
```

Lifecycle: **capture** (anything trackable becomes an issue, immediately) → **triage**
(weekly ~10 min: set priority/effort/risk, promote `idea` → `ready`, drop dead ones) →
**plan/work** (`in-progress`; substantial work links to a plan doc) → **close** (outcome
logged to `DONE.md`; executed plan docs get an `EXECUTED` banner and move to an archive).
Two type conventions worth noting: a `problem` (open design question) closes by producing an
ADR, and `retire` (this should be deleted) is a first-class type, so deletion work is tracked
like feature work. A **sprint** is a working set — the batch selected for parallel execution
now — not a time-box.

## What problem it actually solves

Agent-worked repos accumulate dated plan docs and scratch TODOs that rot: nothing marks a
plan as finished, so an agent re-executes stale work or silently drops an out-of-scope
finding it noticed mid-task. WorkQuarry's one real idea is a test for what deserves a
lifecycle-tracked record versus a plain doc:

> **Does it have to change state or can it rot?** If it must move through a status, it's a
> *record* and lives in `tracker/`. If it's finished the moment it's written (a design
> writeup, a transcript), it's a *doc* and lives elsewhere.

Everything else — issue files, the generated backlog, the done-log, ADRs, the agent rules —
falls out of that one distinction.

## Strengths

- **Doc-vs-record test** is a genuinely crisp answer to a fuzzy problem most repos never
  name explicitly. It's the one idea here that isn't just "GitHub Issues, but files."
- **Plan-doc lifecycle** (an `EXECUTED <date>` banner + move to an archive folder) directly
  targets the failure mode of agents re-running stale plans — a problem specific to
  agent-worked repos that human-oriented trackers don't address.
- **Architecture is sound and small.** Frontmatter = source of truth, `BACKLOG.md` =
  generated view, `DONE.md` = append-only event log, one script as the only write path.
  ~250 lines, zero third-party dependencies, readable end to end in five minutes.
- **The elaboration skills are the real differentiator**, more than the tracker mechanics.
  `/track-idea` and `/track-issue` force a related-work search before filing, then draft
  assumptions, design forks, and a "simplest implementation + what you give up" section. That
  prompt discipline is useful even bolted onto GitHub Issues or Linear — it doesn't need this
  file format to be valuable.
- **Sprint-as-parallel-working-set with an area-collision check** assumes multiple *agents*
  work concurrently, not multiple humans on a calendar. That framing is underexplored
  elsewhere and is the part most clearly built for agent teams rather than adapted from
  human project management.
- **`problem` → ADR closure** gives open design questions a first-class type with its own
  closing action, instead of forcing them into `bug`/`feature`.

## Weaknesses and untested claims

- **Single-writer by construction.** Ids are `max(existing) + 1` — two branches filing
  concurrently collide. `BACKLOG.md` is a single generated file — two branches that both
  touch the tracker will merge-conflict on it. Fine for one person plus their agents on a
  mainline branch (the design point); not fine for a team without changes (collision-safe
  ids, a merge-friendly index, an assignee field — none of which exist yet).
- **Unproven over time.** Extracted 2026-07-18 from a tracker then only two days old. The
  architecture's own stated risk — "weekly triage or the backlog becomes the new rot" — has
  not yet been tested by actually failing to triage for a month. Rot-resistance is stated
  intent, not yet demonstrated behavior.
- **Regex frontmatter parsing, not a YAML library.** Deliberate (zero dependencies), but the
  schema must stay flat and simple; anything nested or exotic will not round-trip.
- **Discipline-dependent, not enforced.** Nothing checks that triage happens, that closed
  issues get banners, or that vendored copies haven't drifted (`install.py --check` exists
  for the drift case, but nothing runs it automatically — no CI, no hook, by default).
- **Skills are Claude Code-specific.** `track.py` and the process are agent-agnostic (any
  agent that can read, write, and grep files can follow them), but the three skills rely on
  Claude Code's `.claude/skills/` discovery specifically.
- **No UI.** CLI + file reads only. Fine for agents and terminal-first developers; a
  non-technical stakeholder gets nothing here.

## Prior art — this is not a new idea

In-repo, markdown-based, agent-friendly trackers are an active space, not a gap WorkQuarry
discovered. Known comparable projects (verify current state before citing — this list may be
stale):

- **Backlog.md** — markdown tasks in-repo with a CLI and a kanban web UI. Closer to a
  human-facing tool with a nicer UX layer; WorkQuarry has no UI at all.
- **beads** (Steve Yegge) — a git-backed, agent-first issue database. Closest conceptual
  sibling — built explicitly for coding agents rather than adapted from human PM tools.

WorkQuarry's honest claim to distinctiveness is narrow: the doc-vs-record test, the plan-doc
lifecycle, `problem`→ADR closure, sprint-as-parallel-working-set, and the elaboration
skills. The underlying "markdown + frontmatter + generated index" pattern is not novel and
is not presented as such.

## Use cases it fits today

- A solo developer (or small team on a single mainline, no parallel tracker edits) working
  with one or more coding agents, who wants tracked work to live and version with the code
  instead of in a separate web app.
- A repo already suffering from dated-plan-doc rot, where the doc-vs-record test and the
  EXECUTED-banner convention directly fix a known failure mode.
- Anyone who wants the elaboration prompts and is willing to adapt them to their own
  tracker, even without adopting the file format.

## Use cases it does not fit yet

- Any team where more than one person/agent-session commits tracker changes on parallel
  branches — id collisions and `BACKLOG.md` merge conflicts are unresolved.
- Anyone who needs a UI, notifications, or reporting beyond a CLI list command and grep.
- Anyone who needs tracked-work history to predate adoption — numbering starts fresh.

## What's in this repository

| Path | Installs to (in a consuming repo) | Role |
|------|-----------------------------------|------|
| `track.py` | `tools/track.py` | the one mutation path (new/set/list/close/index) |
| `process.md` | `tracker/readme.md` | the process spec agents read |
| `templates/` | `tracker/{issues,decisions}/_template.md` | issue + ADR shapes |
| `skills/` | `.claude/skills/track-*` | the elaboration skills |
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
re-run the installer — never hand-edit the copies. Prefer not to use submodules? Copy the
files by hand; `install.py` just automates placement. Python 3.9+, no dependencies.

## License

MIT © 2026 Christopher Diggins / Ara 3D

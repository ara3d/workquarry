# WorkQuarry

**Help agents break big rocks up into small rocks.**

WorkQuarry is a skill-set and process that helps AI coding agents manage work. It tracks
features, bugs, technical debt, ideas, and design questions as plain markdown files inside
your repository. There is no server, no account, and no web application. The entire mechanism
is a folder of markdown files, one small Python script, and a set of instructions that
coding agents follow.

Because everything is a file in your repository, your work history is versioned together
with your code, and any agent that can read, write, and search files can use the system.

This README is a frank assessment as much as an introduction. It explains what the system
does well, what is untested, what is borrowed from earlier tools, and who should not use it
yet.

## How it works

Every tracked item is one markdown file. The file starts with a YAML header that holds the
item's state, followed by a free-form body:

```yaml
---
id: studio-010
title: Orbit camera
type: feature          # feature | debt | bug | idea | problem | retire
status: ready          # idea | ready | in-progress | done | dropped
priority: p1           # p1 | p2 | p3 | ?
effort: S              # S | M | L | ?
risk: low              # low | med | high | ?
area: studio           # your project areas, defined in tracker/config.json
sprint:                # name of the current working set, or empty
created: 2026-07-16
closed:
links: [docs/camera-notes.md, studio-001]
---
Motivation, acceptance criteria, and pointers to related discussion go here.
```

The headers in these files are the database. Three other files present views of it:

- `BACKLOG.md` — a generated table of all open items, one line each. It is never edited by
  hand; the script rebuilds it after every change.
- `DONE.md` — an append-only log of completed and dropped items, with their outcomes.
- `decisions/` — a folder of dated architecture decision records (ADRs). An ADR is never
  edited after the fact; if a decision changes, a new ADR supersedes the old one.

All changes go through one script, `track.py`, which has no dependencies beyond Python 3.9:

```sh
python tools/track.py new --title "Orbit camera" --area studio --type feature
python tools/track.py set studio-010 --status ready --priority p1 --effort S --risk low
python tools/track.py list --open --sort priority
python tools/track.py close studio-010 --outcome "done (a1b2c3d)"
```

### The lifecycle

Work moves through four stages:

1. **Capture.** Anything worth tracking becomes an issue file the moment it comes up. This
   includes ideas the user mentions in passing and problems an agent notices while working
   on something else.
2. **Triage.** About ten minutes a week: assign real priority, effort, and risk values to
   new items, promote ideas that are ready to be worked on, and drop items that are dead.
3. **Work.** An item being worked on is marked `in-progress`. Substantial work gets a
   separate plan document, and the plan and the issue link to each other.
4. **Close.** When work finishes, the script records the outcome in `DONE.md`. If there was
   a plan document, it is stamped with an `EXECUTED` banner and moved to an archive folder,
   so no agent will ever mistake it for live instructions.

Two conventions are worth calling out. First, an open design question is its own item type,
called a `problem`, and it closes by producing an ADR rather than a code change. Second,
code that should be deleted is also its own type, called `retire`, so cleanup work is
tracked with the same care as feature work. Finally, a **sprint** here is not a time-box; it
is a *working set* — the batch of items selected to be worked on in parallel right now.

### The skills

Three skills (instruction files for Claude Code) do the thinking that the script cannot:

- `/track-idea` captures an idea. Before filing, it searches the repository for related
  work, then writes out the idea's assumptions, its major design decisions, possible
  approaches, and the simplest implementation along with what that simple version gives up.
- `/track-issue` does the same for bugs, technical debt, design problems, and retirement
  candidates. It records symptoms, impact, affected code with file-and-line references, a
  root-cause hypothesis, fix options, and how to prevent the problem from recurring.
- `/track-backlog` shows what is in progress, runs triage, promotes items, and plans
  sprints.
- `/write-readme` is a companion skill, independent of the tracker: it writes or reviews a
  project README in plain technical-writer prose — problem solved and not solved, tested
  versus untested, trade-offs, and prior art. This repository's own README follows it.

The tracker skills matter more than the file format. An issue filed through them reads like a
short design document rather than a one-line title, which means the next agent to pick it up
starts with real context instead of an empty box.

## Example workflows

The two sessions below show the system in daily use. The commands are what an agent runs; in
practice you drive the agent in plain language and it runs them for you.

### Capturing and completing a piece of work

You mention an idea in passing — "the orbit camera should support panning." The agent files
it immediately, before doing anything else:

```sh
python tools/track.py new --title "Orbit camera panning" --area studio --type feature
```

The `/track-idea` skill does more than allocate an id. It first searches the repository for
related work, then fills the issue body with the idea's assumptions, the main design
decisions, a couple of possible approaches, and the simplest implementation. The result reads
like a short design note rather than a one-line title.

Later, during triage, the item is given real values and marked ready to work:

```sh
python tools/track.py set studio-010 --status ready --priority p1 --effort M --risk low
```

When an agent picks the item up, it marks the item in progress, does the work, and closes it
with the commit that finished it:

```sh
python tools/track.py set studio-010 --status in-progress
python tools/track.py close studio-010 --outcome "done (a1b2c3d)"
```

The close step records the outcome in `DONE.md` and rebuilds `BACKLOG.md`. Nothing about the
finished work is left for the next agent to misread.

### Reviewing the backlog and sharing a report

Once a week, the `/track-backlog` skill runs the ten-minute triage sweep: it shows what is in
progress, assigns priority and effort to new items, promotes ideas that are ready, and drops
dead ones. To share the current state with someone who does not work at the command line,
render a report (described in the next section):

```sh
python tools/track.py report --html tracker/report.html
python tools/track.py report --md tracker/report.md
```

## Reports

The `report` command renders a point-in-time snapshot of the tracker from the same issue
headers that feed `BACKLOG.md`. It produces a shareable artifact in either or both of two
formats:

- `--html PATH` writes a single self-contained HTML file. Its styling is embedded, so the
  file has no external dependencies: it opens in any browser, attaches to an email, or drops
  into a pull request. Open items are grouped by status — in progress, ready, ideas — with
  priority-one items highlighted, and a final section lists the most recently completed work.
- `--md PATH` writes the same snapshot as Markdown, for pasting into a pull request, an issue,
  or a chat message.

```sh
python tools/track.py report --html tracker/report.html --md tracker/report.md
python tools/track.py report            # no path: prints the Markdown report to stdout
```

With no path, the command prints the Markdown report to standard output, so it composes with
other command-line tools. The report is read-only: it never writes into the tracker's own
data.

A report is a snapshot taken the moment you run the command, not a live dashboard. It does not
refresh itself, draw charts, or track throughput over time. For history, read `DONE.md`; for
ad-hoc queries, use `python tools/track.py list`.

## The problem it solves

Repositories worked by AI agents accumulate planning documents and to-do lists quickly, and
those documents rot. Nothing marks a plan as finished, so an agent that finds a stale plan
may execute it again. Nothing gives a passing observation a home, so an agent that notices a
bug while doing other work silently drops it.

WorkQuarry's central idea is a simple test that decides where any piece of writing belongs:

> **Does it have to change state?** If a document must move through statuses — open, in
> progress, done — it is a *record*, and it lives in the tracker with an id and a status
> field. If it is finished the moment it is written — a design essay, an analysis, a
> transcript — it is a *document*, and it lives with the other documentation.

Everything else in the system follows from that one distinction.

## Strengths

- **The record-versus-document test** gives a crisp answer to a problem most projects never
  name. It is the one idea here that is not simply "GitHub Issues, but files."
- **The plan-document lifecycle** — the `EXECUTED` banner and the archive step — directly
  prevents agents from re-running stale plans. This failure mode is specific to agent-worked
  repositories, and mainstream trackers do not address it.
- **The architecture is small and sound.** File headers are the single source of truth, the
  backlog is a generated view, and the done-log is append-only. One script is the only write
  path. The whole thing is about 400 lines of Python with no dependencies (roughly a third of
  that is the on-demand HTML and Markdown report renderer), readable end to end in a few
  minutes.
- **The elaboration skills are the real contribution.** Forcing a search for related work
  before filing, and requiring an assumptions list and a simplest-implementation section,
  produces issues of unusually high quality. This discipline would be valuable even for a
  team using GitHub Issues or Linear.
- **Sprints are designed for parallel agents.** The sprint-planning skill checks that
  selected items do not touch the same code areas, because two agents editing the same files
  creates merge conflicts. This treats agents, not calendar weeks, as the unit of planning.
- **Design questions are first-class.** A `problem` closes with a recorded decision, not a
  commit, so architectural choices leave a permanent trail.

## Weaknesses and untested claims

- **It is built for a single writer.** New ids are computed as the highest existing number
  plus one, so two branches filing at the same time will collide. The generated `BACKLOG.md`
  will produce merge conflicts whenever two branches both change the tracker. This is
  acceptable for one person and their agents committing to one main branch — which is the
  design point — but a team would need collision-safe ids, a merge-friendly index, and an
  assignee field, none of which exist yet.
- **It is unproven over time.** This mechanism was extracted on 2026-07-18 from a tracker
  that was itself only two days old. Its own founding decision record names the main risk:
  if weekly triage does not happen, the backlog becomes the new source of rot. That claim
  has not yet been tested by real neglect.
- **The YAML parsing is intentionally naive.** The script parses headers with regular
  expressions rather than a YAML library, which keeps it dependency-free but means the
  schema must stay flat. Nested structures will not survive a round-trip.
- **Nothing is enforced automatically.** No hook or CI job checks that triage happens, that
  closed plans get banners, or that installed copies have not drifted from this repository.
  An `install.py --check` command exists for the drift case, but nothing runs it for you.
- **The skills are Claude Code-specific.** The script and the process work with any agent
  that can read and write files, but the skills depend on Claude Code's skill discovery
  mechanism.
- **There is no live user interface.** Everything happens through the command line and file
  reads. Developers and agents are well served; a non-technical stakeholder can be handed a
  static HTML or Markdown report on demand (see *Reports*), but there is no interactive tool,
  dashboard, or notification — the report is a snapshot, not a live view.

## Prior art

In-repository, markdown-based, agent-friendly trackers are an active area, and WorkQuarry
did not invent the category. Two known relatives (verify their current state before citing
them; this list may be stale):

- **Backlog.md** — markdown tasks in the repository, with a CLI and a kanban-style web
  interface. It is closer to a human-facing tool; WorkQuarry has no interface at all.
- **beads** (Steve Yegge) — a git-backed issue database built explicitly for coding agents.
  This is the closest conceptual sibling.

WorkQuarry's honest claim to originality is narrow: the record-versus-document test, the
plan-document lifecycle, the `problem`-closes-with-an-ADR convention, sprints as parallel
working sets, and the elaboration skills. The underlying pattern — markdown files, YAML
headers, a generated index — is common property.

## Who it is for

- A solo developer, or a small team committing to a single main branch, who works with one
  or more coding agents and wants tracked work to live and version with the code.
- A repository already suffering from stale-plan rot, where the record-versus-document test
  and the `EXECUTED` banner fix a known failure mode directly.
- Anyone who wants the elaboration skills and is willing to adapt them to a different
  tracker. They do not depend on this file format.

## Who it is not for, yet

- Teams where more than one person or agent session changes the tracker on parallel
  branches. Id collisions and backlog merge conflicts are unsolved.
- Anyone who needs a live, interactive user interface, notifications, or reporting richer
  than a static HTML or Markdown snapshot — no charts, no history, no auto-refresh.
- Anyone who needs their existing issue history migrated in. Numbering starts fresh.

## What is in this repository

| Path | Installed to (in your repository) | Purpose |
|------|-----------------------------------|---------|
| `track.py` | `tools/track.py` | the single write path plus queries and reports: new, set, list, close, report, index |
| `process.md` | `tracker/readme.md` | the process specification that agents read |
| `templates/` | `tracker/{issues,decisions}/_template.md` | blank issue and ADR files |
| `skills/` | `.claude/skills/<name>` | the three tracker skills plus `write-readme` |
| `AGENTS-snippet.md` | a marked block inside `AGENTS.md` | the rules agents must follow |
| `tracker/config.example.json` | `tracker/config.json` | your area names and paths — the only per-repository configuration |
| `install.py` | not installed; run directly | copies the files above into place and checks for drift |

Your actual issues, backlog, done-log, and decision records are your data. They live in your
repository and never in this one.

## Installation

Add this repository as a submodule and run the installer:

```sh
git submodule add https://github.com/ara3d/workquarry submodules/workquarry
python submodules/workquarry/install.py
```

Then edit `tracker/config.json` and replace the example area names with your own.

To update after a new version is published:

```sh
git submodule update --remote submodules/workquarry
python submodules/workquarry/install.py
python submodules/workquarry/install.py --check   # confirm nothing has drifted
```

Each installed copy carries a banner marking it as generated. Make improvements in this
repository and re-run the installer; never edit the installed copies directly. If you prefer
not to use submodules, you can simply copy the files by hand — the installer only automates
placement. Requires Python 3.9 or later, with no third-party packages.

## License

MIT © 2026 Christopher Diggins / Ara 3D

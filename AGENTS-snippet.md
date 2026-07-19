## Work tracking — `tracker/` (WorkQuarry)

All trackable work (features, bugs, debt, ideas, open design problems, retire candidates) lives in [`tracker/`](tracker/) — one file per issue, indexed in [`tracker/BACKLOG.md`](tracker/BACKLOG.md), completed work logged in [`tracker/DONE.md`](tracker/DONE.md), decisions in [`tracker/decisions/`](tracker/decisions/). Full process: [`tracker/readme.md`](tracker/readme.md).

Non-negotiable agent rules:

1. **Never execute a plan doc without checking its status.** A plan without a linked `ready`/`in-progress` issue in the tracker, or carrying an `EXECUTED` banner, is historical — ask before acting on it.
2. **File what you find.** Out-of-scope debt, bugs, or retire candidates discovered mid-task: file via `/track-issue` (capture-only short form) or `python tools/track.py new`. Do not fix inline, silently drop, or hand-edit BACKLOG.md.
3. **Close what you finish.** When tracked work completes: `python tools/track.py close <id> --outcome "done (sha)"`, then banner + archive the executed plan doc. Status/priority/sprint changes go through `track.py set` — BACKLOG.md is generated, never hand-edited.
4. **Commit tracker changes immediately.** Any time you create or update an issue, `git add` the issue file(s) **and** `BACKLOG.md` together and commit. Never leave them uncommitted or commit one without the other.
5. **Capture user ideas** ("we should someday…") as `type: idea` issues immediately.
6. **Check `tracker/decisions/` before proposing architecture changes**; disagree via a superseding ADR, not silent divergence.

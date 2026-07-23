## Work tracking — `tracker/` (WorkQuarry)

All trackable work (features, bugs, debt, ideas, open design problems, retire candidates) lives in [`tracker/`](tracker/) — one file per issue, indexed in [`tracker/BACKLOG.md`](tracker/BACKLOG.md), completed work logged in [`tracker/DONE.md`](tracker/DONE.md), decisions in [`tracker/decisions/`](tracker/decisions/). Full process: [`tracker/readme.md`](tracker/readme.md).

Non-negotiable agent rules:

1. **Never execute a plan doc without checking its status.** A plan without a linked `ready`/`in-progress` issue in the tracker, or carrying an `EXECUTED` banner, is historical — ask before acting on it.
2. **File what you find.** Out-of-scope debt, bugs, or retire candidates discovered mid-task: file via `/track-issue` (capture-only short form) or `python tools/track.py new`. Do not fix inline, silently drop, or hand-edit BACKLOG.md.
3. **Name the item in the commit.** Any commit doing work on a tracked item names its id in the Conventional-Commits scope — `fix(studio-149): atomic menu clear+rebuild`. This is the join key between git history and tracker state; without it, landed work is invisible to the tracker. Tracker bookkeeping commits (`docs(tracker):`, `chore(tracker):`) are not work commits and do not count as progress on an item.
4. **Mark progress at the commit that makes it.** In the same commit that lands work, tick every `## Done means` checkbox that commit satisfies. Progress is read from those boxes — never store a percentage. Then:
   - **All boxes ticked** → close it now: `python tools/track.py close <id> --outcome "done (<work-commit-sha>)"` and commit that tracker change (see rule 5). The sha is unknown until the work commit exists, so closing is the immediately-following commit, not a deferred chore.
   - **Any box unticked** → the item stays `in-progress`. Say which box is outstanding. A landed fix awaiting verification is not done.

   Never close an item from commit history alone — an unticked box is the item telling you it is not finished. Status/priority/sprint changes go through `track.py set`; BACKLOG.md is generated, never hand-edited, and the executed plan doc gets bannered + archived at close.
5. **Commit tracker changes immediately.** Any time you create or update an issue, `git add` the issue file(s) **and** `BACKLOG.md` together and commit. Never leave them uncommitted or commit one without the other.
6. **Capture user ideas** ("we should someday…") as `type: idea` issues immediately.
7. **Check `tracker/decisions/` before proposing architecture changes**; disagree via a superseding ADR, not silent divergence.

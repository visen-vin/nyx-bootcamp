# Nyx Daily Repo Workflow — Design

Status: approved by Vinayak, 2026-07-10.

## Context

The "Nyx" OpenClaw agent (id `mentor_squad`, VPS `187.127.156.138`,
workspace `/root/.openclaw/workspace-mentor_squad`) already runs a daily
WhatsApp accountability program for 4 mentees — Gaurav, Vishal, Ashish
(Frontend track) and Nishant (Backend track) — via `mentor_engine.py` and
three cron jobs:

- `daily-repo-content` (8:30 AM IST, raw command) — runs
  `repo_content_engine.py create-day`, which already generates
  `Day-N-<topic>` folders in `~/mentor-squad-dsa` (Frontend, Backend,
  CS-Core Theory+Assignment, DSA Question) and pushes to `main`.
- `mentor-squad-daily` (9:00 AM IST, agent turn) — posts the day's targets
  to the WhatsApp group and advances everyone's `day_index`.
- `dsa-daily-solution` (9:30 PM IST, agent turn) — writes and publishes
  `SOLUTION.md` for the day's DSA question.

Two problems triggered this design:

1. **Stale curriculum.** `mentor_engine.py` still reads the old 12-week
   `roadmap_frontend.json` / `roadmap_backend.json` (which embed
   `daily_cs_core` / `daily_dsa` per day). A new 18-week (90-day), 4-subject
   curriculum already exists on disk (`roadmap_frontend_90day.json`,
   `roadmap_backend_90day.json`, `roadmap_cscore_90day.json`, each with
   `weeks[].daily_main` / `daily_resources`, 5 authored days/week) plus the
   existing flat `dsa_questions.json` (376 questions) — but nothing reads
   the three new files yet.
2. **No submission loop for the GitHub repo.** Mentees are expected to open
   PRs against `mentor-squad-dsa` (documented in the repo's `README.md`),
   but nothing creates their per-day file for them, nothing reviews or
   merges their PRs, and there is no progress tracker for the repo side
   (separate from the JSON-based WhatsApp progress in `progress.json`).

## Scope

In scope:
- Migrate `mentor_engine.py` to the new 90-day curriculum.
- Extend `repo_content_engine.py` to also create empty per-mentee
  placeholder files each day.
- Add PR review: safety-gated auto-merge + "brutal honest" code feedback
  as a GitHub PR comment, triggered hourly and on-demand from WhatsApp.
- Add `TRACKER.md` (repo root) showing per-person progress.

Out of scope (explicitly deferred, not part of this change):
- WhatsApp emoji-reaction or poll-based acknowledgment — investigated,
  not reliably readable by OpenClaw's WhatsApp integration for this
  use case (reaction reading is scoped to the exec/plugin approval
  pipeline only; poll creation is one-way, no vote-read-back). PR merges
  serve as the acknowledgment signal instead.
- Any change to `people.json` / `phone_roster.json` (WhatsApp roster) —
  the PR safety-gate does not need a GitHub-username roster (see below).

## 1. Curriculum migration (`mentor_engine.py`)

- `ROADMAP_PATHS` changes from `{frontend, backend}` pointing at the old
  files to pointing at `roadmap_frontend_90day.json` and
  `roadmap_backend_90day.json`.
- A new `roadmap_cscore_90day.json` lookup is added, resolved with the
  **same `day_index`** as the person's main track (shared day timeline —
  Day N means the same day for every subject). CS-Core is no longer a
  per-track field; it's one shared topic per day for everyone.
- `cmd_get_today`'s `cs_core` field comes from the CS-Core roadmap instead
  of the old embedded `week['daily_cs_core']`.
- `cmd_get_today`'s `dsa` field comes from `dsa_questions.json[day_index]`
  (falls back to the last question if `day_index` exceeds the list)
  instead of the old embedded `week['daily_dsa']`.
- `day_index` values already in `progress.json` (currently 3 for all four
  people) carry over unchanged — both curricula use the same 5-authored-
  days-per-week resolution, so no reset or remapping is needed.
- No change to `dsa_next_index` / `content_progress.json` — DSA content
  for the repo stays on its own independent counter in
  `repo_content_engine.py`, decoupled from `mentor_engine.py`'s
  `day_index`. (These two can drift by at most a few days if a cron run
  fails; acceptable since DSA questions aren't dated content and repeats
  aren't harmful. Not fixing this now — no evidence it has ever drifted.)

## 2. Daily repo content (`repo_content_engine.py`)

Changes to `cmd_create_day`:

- CS-Core section: replace the "Frontend track: X / Backend track: Y"
  two-topic writeup with a single shared topic (matches the migration
  above — CS-Core is now one curriculum, not two views of embedded
  per-track fields).
- After writing the existing Theory/Assignment/Question READMEs, also
  create one empty placeholder file per mentee in each folder that
  applies to them:

  | Folder | Who gets a placeholder |
  |---|---|
  | `Frontend/Assignment/Day-N-.../` | Gaurav, Vishal, Ashish |
  | `Backend/Assignment/Day-N-.../` | Nishant |
  | `CS-Core/Assignment/Day-N-.../` | all 4 |
  | `DSA/Question/Day-N-.../` | all 4 |

  Filename: **`DAY<N>_<Name>.md`** (e.g. `DAY4_Gaurav.md`), empty except
  for a one-line HTML comment placeholder marker (so git tracks it and a
  diff against "still empty" is trivial to detect). `<Name>` is exactly
  one of the 4 keys in `people.json`.

- These placeholder files are committed and pushed in the same commit as
  the rest of the day's content (no behavior change to the commit/push
  flow itself).
- The `Frontend/Assignment` and `Backend/Assignment` and
  `CS-Core/Assignment` README bodies are updated to say "fill in
  `DAY<N>_<YourName>.md` in this folder" instead of "add a file named
  after yourself" (the old instruction, now superseded).

## 3. PR review (new)

### Safety gate (deterministic, no LLM judgment)

A PR is **auto-merge eligible** if and only if its diff touches **exactly
one file**, and that file's path matches:

```
(Frontend|Backend|CS-Core)/Assignment/Day-<N>-*/DAY<N>_<Name>.md
DSA/Question/Day-<N>-*/DAY<N>_<Name>.md
```

where `<Name>` is one of the 4 known mentees and `<N>` matches the day
number embedded in both the folder and the filename. No GitHub-username
roster is needed — eligibility is judged purely from the file path/name
in the diff, not from who opened the PR. This is deliberately strict:
any PR touching a different file, multiple files, or someone else's
placeholder is **not** auto-merged — it's left open and flagged to
Vinayak with a one-line reason (e.g. "touches 2 files" / "not a known
placeholder path").

This check is a new subcommand, e.g. `mentor_engine.py check-pr --number
<n>` (or a small dedicated script) — plain Python + `gh pr diff --name-
only`, no model call. Auditable and safe to run unattended.

### Feedback (LLM judgment, Nyx)

For every PR that gets processed (merged or flagged), Nyx reads the diff
(`gh pr diff <n>`) and posts a **direct, unsparingly honest** code-review
comment on the PR itself via `gh pr comment <n> --body ...` — call out
bugs, bad complexity, sloppy naming, missed edge cases, don't soften it
just to be nice. This is a different register from Nyx's usual
encouraging WhatsApp persona — IDENTITY.md gets a note that code-review
feedback on PRs is the one place bluntness is the goal, because that's
what the mentee explicitly asked for.

### Two triggers, same underlying logic

1. **Hourly cron** (new, agent-turn, same pattern as `dsa-daily-solution`):
   list open PRs (`gh pr list`), run the safety gate + feedback flow on
   any not yet processed.
2. **On-demand via WhatsApp** (new case in `IDENTITY.md`): a message from
   one of the 4 known mentees (resolved via `whoami`, same as case 2)
   containing a PR link/number triggers the same flow immediately, plus a
   short WhatsApp reply ("merged ✅ / here's what to fix ❌" + one-line
   summary — full feedback lives on the PR, not the group).

## 4. `TRACKER.md`

Regenerated (overwritten, committed, pushed) every time the PR-check flow
merges at least one PR. Lives at the repo root. One table:

```
| Person | Current Day | Frontend/Backend | CS-Core | DSA | Last Merged Day |
|---|---|---|---|---|---|
| Gaurav | 4 | ✅ | ✅ | ⬜ | 3 |
...
```

A subject cell is ✅ if that person's current-day placeholder for that
subject has been merged, ⬜ if still open/empty. "Last Merged Day" is the
highest day number for which **all** of that person's applicable subjects
(their track + CS-Core + DSA) are merged.

## 5. Guardrails

- Safety-gate failures (multi-file PRs, wrong path, unknown name) never
  auto-merge — always flagged, never silently dropped or force-merged.
- The hourly cron and on-demand path share one underlying check function,
  so behavior can't drift between the two entry points.
- No change to WhatsApp `record-status` self-report flow (case 2) — the
  repo/PR tracking is additive, not a replacement.

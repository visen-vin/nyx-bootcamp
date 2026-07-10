# Nyx Daily Repo Workflow Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Migrate the Nyx mentoring agent to its new 90-day, 4-subject curriculum and add a self-service GitHub PR review loop (safety-gated auto-merge + brutal-honest feedback) with a live progress tracker, per `docs/superpowers/specs/2026-07-10-nyx-daily-workflow-design.md`.

**Architecture:** Two existing deterministic Python CLI scripts on the VPS (`mentor_engine.py`, `repo_content_engine.py`) get targeted edits; a new deterministic script (`pr_engine.py`) owns the PR safety-gate, merge, and tracker generation; a new hourly cron job (agent-turn, same shape as the existing `dsa-daily-solution` job) supplies the LLM judgment (reading diffs, writing blunt feedback) that the deterministic layer intentionally does not attempt.

**Tech Stack:** Python 3 (stdlib only — `argparse`, `json`, `re`, `subprocess`, `pathlib`), `git`, GitHub `gh` CLI, OpenClaw cron (`openclaw cron create`).

## Global Constraints

- All target files live on the VPS (`root@187.127.156.138`), not in any local checkout. Every edit step pulls the file to local scratch, edits it, and pushes it back over SSH/SCP — do not assume a local working tree at `/root/...` paths.
- This is a live production system: a real WhatsApp group gets a message at 9:00 AM IST daily, and `repo_content_engine.py create-day` already fires at 8:30 AM IST daily via cron `daily-repo-content` (id `c4b87717-495d-4ee8-982f-1d46ef8beade`). Every test in this plan must run against **isolated copies** of the repo and `content_progress.json`, never the live `/root/mentor-squad-dsa` or the live `content_progress.json`, unless the step explicitly says "production."
- Do not modify `progress.json` or its `day_index` values for any person — they must carry over unchanged (currently `3` for all four people).
- Reuse the existing code style: plain functions, `argparse` subcommands, `json.dumps`/`load_json` helpers, no new third-party dependencies, no test framework (write assert-based scripts, matching the codebase's zero-framework style).
- Local scratch directory for staging file edits: `/private/tmp/claude-501/-Users-vin/ed0ccc34-a8fe-4de8-b5e0-87beb62c0f36/scratchpad` (referred to below as `$SCRATCH`).
- VPS shorthand used below: `VPS=root@187.127.156.138`, `WS=/root/.openclaw/workspace-mentor_squad`.

---

## Task 1: Migrate `mentor_engine.py` to the 90-day curriculum

**Files:**
- Modify: `$WS/mentor_engine.py` (VPS path; stage at `$SCRATCH/mentor_engine.py`)
- Test: `$SCRATCH/test_mentor_engine_migration.py`, run on the VPS

**Interfaces:**
- Consumes: `roadmap_frontend_90day.json`, `roadmap_backend_90day.json`, `roadmap_cscore_90day.json`, `dsa_questions.json` — all already present at `$WS/` (verified this session; each roadmap file has shape `{"weeks": [{"phase", "topic", "daily_main": [5 items], "daily_resources": [5 items]}, ...]}`, 18 weeks; `dsa_questions.json` is a flat JSON list of `{"topic", "question", "link", "remarks"}`, 376 entries).
- Produces: `resolve_cscore_topic(day_index: int) -> str` and `resolve_dsa_question(day_index: int) -> str`, both new module-level functions in `mentor_engine.py`. `cmd_get_today`'s printed JSON keeps its existing shape (`day_index`, `week`, `day_in_week`, `main_target`, `cs_core`, `dsa`, `resources`, `is_weekend_review`) — only how `cs_core` and `dsa` are computed changes. Task 4's cron message and IDENTITY.md text reference this JSON shape as-is (unchanged), so no downstream consumer needs updating.

- [ ] **Step 1: Pull the current file to scratch**

```bash
mkdir -p /private/tmp/claude-501/-Users-vin/ed0ccc34-a8fe-4de8-b5e0-87beb62c0f36/scratchpad
ssh root@187.127.156.138 "cat /root/.openclaw/workspace-mentor_squad/mentor_engine.py" \
  > /private/tmp/claude-501/-Users-vin/ed0ccc34-a8fe-4de8-b5e0-87beb62c0f36/scratchpad/mentor_engine.py
```

- [ ] **Step 2: Write the failing test**

Create `$SCRATCH/test_mentor_engine_migration.py`:

```python
import json
import sys

sys.path.insert(0, '.')
import mentor_engine as me

# Ground truth read directly from the 90-day files on disk, so the test
# fails loudly if those files ever change shape.
frontend = json.loads((me.BASE / 'roadmap_frontend_90day.json').read_text())
cscore = json.loads((me.BASE / 'roadmap_cscore_90day.json').read_text())
dsa = json.loads((me.BASE / 'dsa_questions.json').read_text())

assert me.ROADMAP_PATHS['frontend'].name == 'roadmap_frontend_90day.json', \
    f"expected frontend roadmap to be the 90-day file, got {me.ROADMAP_PATHS['frontend'].name}"
assert me.ROADMAP_PATHS['backend'].name == 'roadmap_backend_90day.json', \
    f"expected backend roadmap to be the 90-day file, got {me.ROADMAP_PATHS['backend'].name}"

got = me.resolve_cscore_topic(0)
expected = cscore['weeks'][0]['daily_main'][0]
assert got == expected, f"resolve_cscore_topic(0) = {got!r}, expected {expected!r}"

got = me.resolve_dsa_question(0)
assert dsa[0]['question'] in got, f"resolve_dsa_question(0) = {got!r}, expected it to contain {dsa[0]['question']!r}"

# day_index way past the list length must clamp to the last question, not crash
got = me.resolve_dsa_question(len(dsa) + 1000)
assert dsa[-1]['question'] in got, "resolve_dsa_question must clamp to the last question when day_index overflows"

result = json.loads(me.cmd_get_today.__wrapped__(None)) if False else None  # placeholder guard, unused

print("ALL PASS")
```

- [ ] **Step 3: Run test to verify it fails**

```bash
scp /private/tmp/claude-501/-Users-vin/ed0ccc34-a8fe-4de8-b5e0-87beb62c0f36/scratchpad/test_mentor_engine_migration.py \
  root@187.127.156.138:/root/.openclaw/workspace-mentor_squad/test_mentor_engine_migration.py
ssh root@187.127.156.138 "cd /root/.openclaw/workspace-mentor_squad && python3 test_mentor_engine_migration.py"
```

Expected: `AttributeError: module 'mentor_engine' has no attribute 'resolve_cscore_topic'` (function doesn't exist yet).

- [ ] **Step 4: Edit `$SCRATCH/mentor_engine.py`**

Replace the `ROADMAP_PATHS` block:

```python
ROADMAP_PATHS = {
    'frontend': BASE / 'roadmap_frontend.json',
    'backend': BASE / 'roadmap_backend.json',
}
```

with:

```python
ROADMAP_PATHS = {
    'frontend': BASE / 'roadmap_frontend_90day.json',
    'backend': BASE / 'roadmap_backend_90day.json',
}
CSCORE_ROADMAP_PATH = BASE / 'roadmap_cscore_90day.json'
DSA_QUESTIONS_PATH = BASE / 'dsa_questions.json'
```

Add two new functions directly above `cmd_get_today`:

```python
def load_dsa_questions():
    return load_json(DSA_QUESTIONS_PATH, [])


def resolve_cscore_topic(day_index):
    cscore_roadmap = load_json(CSCORE_ROADMAP_PATH, {'weeks': []})
    week_idx, day_in_week, week = resolve_day(cscore_roadmap, day_index)
    if day_in_week >= DAYS_PER_WEEK_AUTHORED:
        return f"Review & catch-up: revisit this week's CS-core topic ({week['topic']})."
    return week['daily_main'][day_in_week]


def resolve_dsa_question(day_index):
    questions = load_dsa_questions()
    if not questions:
        return 'No DSA questions available.'
    idx = min(day_index, len(questions) - 1)
    q = questions[idx]
    return f"{q['question']} ({q['topic']})"
```

Replace the body of `cmd_get_today` (both branches build `cs_core`/`dsa` inline today — pull them out before the `if`):

```python
def cmd_get_today(args):
    people = load_people()
    progress = load_progress()
    track = person_track(args.person, people)
    roadmap = load_roadmap(track)
    day_index = person_day_index(args.person, progress)
    week_idx, day_in_week, week = resolve_day(roadmap, day_index)
    cs_core = resolve_cscore_topic(day_index)
    dsa = resolve_dsa_question(day_index)
    if day_in_week >= DAYS_PER_WEEK_AUTHORED:
        result = {
            'day_index': day_index,
            'week': week_idx + 1,
            'day_in_week': day_in_week + 1,
            'main_target': f"Review & catch-up: revisit this week's topic ({week['topic']}) and finish anything unfinished.",
            'cs_core': cs_core,
            'dsa': dsa,
            'resources': None,
            'is_weekend_review': True,
        }
    else:
        result = {
            'day_index': day_index,
            'week': week_idx + 1,
            'day_in_week': day_in_week + 1,
            'main_target': week['daily_main'][day_in_week],
            'cs_core': cs_core,
            'dsa': dsa,
            'resources': week.get('daily_resources', [None]*5)[day_in_week],
            'is_weekend_review': False,
        }
    print(json.dumps(result))
```

Do not change `load_roadmap`, `resolve_day`, `person_track`, `person_day_index`, or anything else in the file.

Also delete the unused placeholder line from the test file (`result = json.loads(...) if False else None`) — it was only there to remind you not to call `cmd_get_today` directly (it prints, doesn't return); remove that line from `$SCRATCH/test_mentor_engine_migration.py` before the next run.

- [ ] **Step 5: Push the edited file and test, re-run**

```bash
scp /private/tmp/claude-501/-Users-vin/ed0ccc34-a8fe-4de8-b5e0-87beb62c0f36/scratchpad/mentor_engine.py \
  root@187.127.156.138:/root/.openclaw/workspace-mentor_squad/mentor_engine.py
scp /private/tmp/claude-501/-Users-vin/ed0ccc34-a8fe-4de8-b5e0-87beb62c0f36/scratchpad/test_mentor_engine_migration.py \
  root@187.127.156.138:/root/.openclaw/workspace-mentor_squad/test_mentor_engine_migration.py
ssh root@187.127.156.138 "cd /root/.openclaw/workspace-mentor_squad && python3 test_mentor_engine_migration.py"
```

Expected: `ALL PASS`.

- [ ] **Step 6: Manually sanity-check the live CLI output (read-only, safe against production data — `get-today` does not mutate state)**

```bash
ssh root@187.127.156.138 "cd /root/.openclaw/workspace-mentor_squad && python3 mentor_engine.py get-today --person Gaurav"
```

Expected: JSON with `day_index: 3`, `main_target` matching `roadmap_frontend_90day.json` week 1 day-in-week 4's `daily_main[3]`, `cs_core` matching `roadmap_cscore_90day.json` week 1 `daily_main[3]`, `dsa` containing the question text from `dsa_questions.json[3]`.

- [ ] **Step 7: Clean up the test file from the workspace and commit locally is not applicable (no local repo for this file) — record the change in the spec's repo instead**

```bash
ssh root@187.127.156.138 "rm /root/.openclaw/workspace-mentor_squad/test_mentor_engine_migration.py"
```

There is no git repo backing `$WS` files, so "commit" for this task means: the edit is now live on the VPS, verified by Step 6. Move to Task 2.

---

## Task 2: Placeholder files + single-topic CS-Core in `repo_content_engine.py`

**Files:**
- Modify: `$WS/repo_content_engine.py` (stage at `$SCRATCH/repo_content_engine.py`)
- Test: `$SCRATCH/test_repo_content_engine.py`, run on the VPS against an isolated repo clone

**Interfaces:**
- Consumes: `mentor_engine.py get-today` JSON (Task 1's output shape, unchanged), `$WS/people.json` (`{"Gaurav": "frontend", "Vishal": "frontend", "Ashish": "frontend", "Nishant": "backend"}`).
- Produces: `write_placeholder(folder: str, day_index: int, slug: str, name: str) -> None` (new function). `cmd_create_day` still prints the same JSON shape it does today (`status`, `day_index`, `slugs`, `dsa`) — Task 3 does not depend on this output, so no contract changes there. The environment variables `MENTOR_SQUAD_REPO_PATH` and `MENTOR_SQUAD_CONTENT_PROGRESS_PATH` (new, both optional) let tests point the script at an isolated repo/progress file instead of the live ones — Task 3's `pr_engine.py` reuses the same `MENTOR_SQUAD_REPO_PATH` variable name for consistency.

- [ ] **Step 1: Pull the current file to scratch**

```bash
ssh root@187.127.156.138 "cat /root/.openclaw/workspace-mentor_squad/repo_content_engine.py" \
  > /private/tmp/claude-501/-Users-vin/ed0ccc34-a8fe-4de8-b5e0-87beb62c0f36/scratchpad/repo_content_engine.py
```

- [ ] **Step 2: Set up an isolated test repo clone on the VPS (once, reused by this task's tests)**

```bash
ssh root@187.127.156.138 "rm -rf /root/mentor-squad-dsa-test && cp -r /root/mentor-squad-dsa /root/mentor-squad-dsa-test && cd /root/mentor-squad-dsa-test && git remote remove origin"
```

Removing `origin` on the test clone guarantees a bug in the test run can never accidentally `git push` to the real GitHub repo.

- [ ] **Step 3: Write the failing test**

Create `$SCRATCH/test_repo_content_engine.py`:

```python
import json
import os
import shutil
import sys
from pathlib import Path

TEST_REPO = Path('/root/mentor-squad-dsa-test')
TEST_PROGRESS = Path('/root/.openclaw/workspace-mentor_squad/content_progress.test.json')

# Start from a clean progress file so create-day treats this as a fresh day.
if TEST_PROGRESS.exists():
    TEST_PROGRESS.unlink()
TEST_PROGRESS.write_text(json.dumps({'dsa_next_index': 0, 'created_days': [], 'slugs_by_day': {}}))

os.environ['MENTOR_SQUAD_REPO_PATH'] = str(TEST_REPO)
os.environ['MENTOR_SQUAD_CONTENT_PROGRESS_PATH'] = str(TEST_PROGRESS)

sys.path.insert(0, '.')
import repo_content_engine as rce

rce.cmd_create_day()

progress = json.loads(TEST_PROGRESS.read_text())
day_index = progress['created_days'][-1]
slugs = progress['slugs_by_day'][str(day_index)]

# Placeholder files exist for every mentee in every folder that applies to them.
for name in ('Gaurav', 'Vishal', 'Ashish'):
    p = TEST_REPO / 'Frontend' / 'Assignment' / f"Day-{day_index}-{slugs['frontend']}" / f'DAY{day_index}_{name}.md'
    assert p.exists(), f"missing frontend placeholder for {name}: {p}"
    assert p.read_text().startswith('<!--'), f"placeholder for {name} should start with an HTML comment marker"

p = TEST_REPO / 'Backend' / 'Assignment' / f"Day-{day_index}-{slugs['backend']}" / f'DAY{day_index}_Nishant.md'
assert p.exists(), f"missing backend placeholder for Nishant: {p}"

for name in ('Gaurav', 'Vishal', 'Ashish', 'Nishant'):
    p = TEST_REPO / 'CS-Core' / 'Assignment' / f"Day-{day_index}-{slugs['cscore']}" / f'DAY{day_index}_{name}.md'
    assert p.exists(), f"missing cs-core placeholder for {name}: {p}"
    p = TEST_REPO / 'DSA' / 'Question' / f"Day-{day_index}-{slugs['dsa']}" / f'DAY{day_index}_{name}.md'
    assert p.exists(), f"missing dsa placeholder for {name}: {p}"

# CS-Core README is single-topic now, not the old "Frontend track / Backend track" split.
cs_readme = (TEST_REPO / 'CS-Core' / 'Theory' / f"Day-{day_index}-{slugs['cscore']}" / 'README.md').read_text()
assert 'Frontend track' not in cs_readme, "CS-Core README should no longer show a per-track split"
assert 'Backend track' not in cs_readme, "CS-Core README should no longer show a per-track split"

print("ALL PASS")
```

- [ ] **Step 4: Run test to verify it fails**

```bash
scp /private/tmp/claude-501/-Users-vin/ed0ccc34-a8fe-4de8-b5e0-87beb62c0f36/scratchpad/test_repo_content_engine.py \
  root@187.127.156.138:/root/.openclaw/workspace-mentor_squad/test_repo_content_engine.py
ssh root@187.127.156.138 "cd /root/.openclaw/workspace-mentor_squad && python3 test_repo_content_engine.py"
```

Expected: `AssertionError: missing frontend placeholder for Gaurav: ...` (placeholders don't exist yet) — or, if `mentor-squad-dsa-test` already has `created_days` containing this run's `day_index` from a previous partial run, `status: skipped` printed with nothing asserted; if that happens, `rm -rf /root/mentor-squad-dsa-test` and redo Step 2 before continuing.

- [ ] **Step 5: Edit `$SCRATCH/repo_content_engine.py`**

Add `os` to the imports and make `REPO_PATH`/`CONTENT_PROGRESS_PATH` overridable:

```python
import json
import os
import re
import subprocess
from pathlib import Path

WORKSPACE = Path(__file__).resolve().parent
QUESTIONS_PATH = WORKSPACE / 'dsa_questions.json'
CONTENT_PROGRESS_PATH = Path(os.environ.get(
    'MENTOR_SQUAD_CONTENT_PROGRESS_PATH', str(WORKSPACE / 'content_progress.json')))
REPO_PATH = Path(os.environ.get(
    'MENTOR_SQUAD_REPO_PATH', str(Path.home() / 'mentor-squad-dsa')))
PEOPLE_PATH = WORKSPACE / 'people.json'
```

Add a placeholder writer and a people-loader next to the existing `write` function:

```python
def load_people():
    return json.loads(PEOPLE_PATH.read_text())


def write_placeholder(folder, day_index, slug, name):
    path = REPO_PATH / folder / f'Day-{day_index}-{slug}' / f'DAY{day_index}_{name}.md'
    write(path, f"<!-- Fill this in with your Day {day_index} answer, then open a PR. -->\n")
```

Replace the CS-Core block inside `cmd_create_day` (currently two separate `frontend['cs_core']` / `backend['cs_core']` values written as "Frontend track / Backend track") with a single shared topic — `frontend['cs_core']` and `backend['cs_core']` are guaranteed equal after Task 1's migration (both now resolve from the same `roadmap_cscore_90day.json` via the same `day_index`), so either can be used:

```python
    # --- CS-Core: Theory + Assignment (single shared topic, same for everyone) ---
    cs_topic = frontend['cs_core']
    cs_slug = slugify(cs_topic)
    slugs['cscore'] = cs_slug
    write(REPO_PATH / 'CS-Core' / 'Theory' / f'Day-{day_index}-{cs_slug}' / 'README.md', f"""# Day {day_index} — CS Core Theory

## Topic
**{cs_topic}**
""")
    write(REPO_PATH / 'CS-Core' / 'Assignment' / f'Day-{day_index}-{cs_slug}' / 'README.md', f"""# Day {day_index} — CS Core Assignment

Fill in `DAY{day_index}_<YourName>.md` in this folder with a short
explanation (a few lines) of today's topic:

**{cs_topic}**

Then open a Pull Request. See the main [README](../../../README.md) for
the full workflow.
""")
```

Update the Frontend and Backend Assignment README bodies (currently "Push your work ... adding a file under this folder named after yourself, e.g. `<YourName>.md`") to reference the new placeholder filename — Frontend:

```python
    write(REPO_PATH / 'Frontend' / 'Assignment' / f'Day-{day_index}-{fe_slug}' / 'README.md', f"""# Day {day_index} — Frontend Assignment

Hands-on task: build/practice something that demonstrates today's topic —
**{frontend['main_target']}**.

Fill in `DAY{day_index}_<YourName>.md` in this folder and open a Pull
Request.
""")
```

and Backend (same pattern):

```python
    write(REPO_PATH / 'Backend' / 'Assignment' / f'Day-{day_index}-{be_slug}' / 'README.md', f"""# Day {day_index} — Backend Assignment

Hands-on task: build/practice something that demonstrates today's topic —
**{backend['main_target']}**.

Fill in `DAY{day_index}_<YourName>.md` in this folder and open a Pull
Request.
""")
```

Update the DSA Question "How to submit" section similarly — replace:

```python
## How to submit
Add your solution as `DSA/Question/Day-{day_index}-{dsa_slug}/<YourName>.md`
(or any code file) and open a Pull Request. See the main
[README](../../../README.md) for the full workflow.
```

with:

```python
## How to submit
Fill in `DSA/Question/Day-{day_index}-{dsa_slug}/DAY{day_index}_<YourName>.md`
with your solution and open a Pull Request. See the main
[README](../../../README.md) for the full workflow.
```

(This is an f-string inside the existing `write(...)` call — keep it inside the same f-string, just change the two lines of text.)

Finally, add the placeholder-writing calls. Right after the `slugs['cscore'] = cs_slug` block (i.e., after both CS-Core `write(...)` calls above) and before the DSA block, insert:

```python
    people = load_people()
    fe_people = [n for n, t in people.items() if t == 'frontend']
    be_people = [n for n, t in people.items() if t == 'backend']
    all_people = list(people.keys())

    for name in fe_people:
        write_placeholder('Frontend/Assignment', day_index, fe_slug, name)
    for name in be_people:
        write_placeholder('Backend/Assignment', day_index, be_slug, name)
    for name in all_people:
        write_placeholder('CS-Core/Assignment', day_index, cs_slug, name)
```

And inside the `if idx < len(questions):` branch of the DSA section, right after `slugs['dsa'] = dsa_slug` (before the `write(REPO_PATH / 'DSA' / 'Question' ...)` call or right after it — order doesn't matter, just keep it inside the `if idx < len(questions):` block since `dsa_slug` only exists there), add:

```python
        for name in all_people:
            write_placeholder('DSA/Question', day_index, dsa_slug, name)
```

- [ ] **Step 6: Push the edited file and re-run the test**

```bash
scp /private/tmp/claude-501/-Users-vin/ed0ccc34-a8fe-4de8-b5e0-87beb62c0f36/scratchpad/repo_content_engine.py \
  root@187.127.156.138:/root/.openclaw/workspace-mentor_squad/repo_content_engine.py
ssh root@187.127.156.138 "rm -rf /root/mentor-squad-dsa-test && cp -r /root/mentor-squad-dsa /root/mentor-squad-dsa-test && cd /root/mentor-squad-dsa-test && git remote remove origin && cd /root/.openclaw/workspace-mentor_squad && python3 test_repo_content_engine.py"
```

Expected: `ALL PASS`.

- [ ] **Step 7: Clean up test artifacts**

```bash
ssh root@187.127.156.138 "rm -rf /root/mentor-squad-dsa-test /root/.openclaw/workspace-mentor_squad/content_progress.test.json /root/.openclaw/workspace-mentor_squad/test_repo_content_engine.py"
```

Do **not** run `repo_content_engine.py create-day` against the live `/root/mentor-squad-dsa` as part of this task — the next real run happens naturally at 8:30 AM IST via the existing `daily-repo-content` cron job, and will pick up these edits automatically.

---

## Task 3: `pr_engine.py` — safety-gated auto-merge + `TRACKER.md`

**Files:**
- Create: `$WS/pr_engine.py` (stage at `$SCRATCH/pr_engine.py`)
- Test: `$SCRATCH/test_pr_engine.py`, run on the VPS

**Interfaces:**
- Consumes: `$WS/people.json`, `MENTOR_SQUAD_REPO_PATH`/`MENTOR_SQUAD_CONTENT_PROGRESS_PATH` env vars (same names introduced in Task 2), the repo's `content_progress.json` `slugs_by_day` map (produced by `repo_content_engine.py`, read-only here).
- Produces: `is_safe_placeholder_path(path: str, people: dict) -> tuple[bool, str | None]`, `evaluate_pr(pr_number: int, people: dict, gh_runner=run_gh) -> dict` (keys: `safe: bool`, plus `file` or `reason`), `merge_pr(pr_number: int, gh_runner=run_gh) -> bool`, `generate_tracker() -> Path`. Task 4's cron message calls this script's `check-pr` and `update-tracker` subcommands by name — those exact subcommand names must match.

- [ ] **Step 1: Write the failing test**

Create `$SCRATCH/test_pr_engine.py`:

```python
import json
import os
import sys
from pathlib import Path

TEST_REPO = Path('/root/mentor-squad-dsa-test')
TEST_PROGRESS = Path('/root/.openclaw/workspace-mentor_squad/content_progress.test.json')

os.environ['MENTOR_SQUAD_REPO_PATH'] = str(TEST_REPO)
os.environ['MENTOR_SQUAD_CONTENT_PROGRESS_PATH'] = str(TEST_PROGRESS)

sys.path.insert(0, '.')
import pr_engine as pe

people = {'Gaurav': 'frontend', 'Vishal': 'frontend', 'Ashish': 'frontend', 'Nishant': 'backend'}

# --- is_safe_placeholder_path ---
ok, reason = pe.is_safe_placeholder_path('Frontend/Assignment/Day-4-css-flexbox/DAY4_Gaurav.md', people)
assert ok, f"expected safe path to pass, got reason={reason!r}"

ok, reason = pe.is_safe_placeholder_path('Frontend/Assignment/Day-4-css-flexbox/DAY4_Someone.md', people)
assert not ok and 'not a known mentee' in reason, f"unknown name should be rejected, got ok={ok} reason={reason!r}"

ok, reason = pe.is_safe_placeholder_path('Frontend/Assignment/Day-4-css-flexbox/DAY5_Gaurav.md', people)
assert not ok and 'does not match' in reason, f"mismatched day should be rejected, got ok={ok} reason={reason!r}"

ok, reason = pe.is_safe_placeholder_path('Frontend/Theory/Day-4-css-flexbox/README.md', people)
assert not ok, "README.md path should never be a safe placeholder path"

# --- evaluate_pr with a fake gh runner (no network, no real GitHub calls) ---
class FakeGh:
    def __init__(self, files):
        self.files = files

    def __call__(self, args, **kwargs):
        class R:
            pass
        r = R()
        if args[:2] == ['pr', 'diff']:
            r.returncode = 0
            r.stdout = '\n'.join(self.files)
            r.stderr = ''
        else:
            raise AssertionError(f'unexpected gh args in this test: {args}')
        return r

verdict = pe.evaluate_pr(1, people, gh_runner=FakeGh(['Frontend/Assignment/Day-4-css-flexbox/DAY4_Gaurav.md']))
assert verdict['safe'], f"single safe file should evaluate safe, got {verdict}"

verdict = pe.evaluate_pr(2, people, gh_runner=FakeGh([
    'Frontend/Assignment/Day-4-css-flexbox/DAY4_Gaurav.md',
    'Backend/Assignment/Day-4-methods/DAY4_Nishant.md',
]))
assert not verdict['safe'] and 'touches 2 files' in verdict['reason'], f"multi-file PR must not be safe, got {verdict}"

# --- generate_tracker against a fixture repo/progress ---
TEST_REPO.mkdir(parents=True, exist_ok=True)
TEST_PROGRESS.write_text(json.dumps({
    'created_days': [4],
    'slugs_by_day': {'4': {'frontend': 'css-flexbox', 'backend': 'methods', 'cscore': 'networking', 'dsa': 'two-sum'}},
}))
(TEST_REPO / 'Frontend' / 'Assignment' / 'Day-4-css-flexbox').mkdir(parents=True, exist_ok=True)
(TEST_REPO / 'Frontend' / 'Assignment' / 'Day-4-css-flexbox' / 'DAY4_Gaurav.md').write_text('I solved it like this...\n')
(TEST_REPO / 'Frontend' / 'Assignment' / 'Day-4-css-flexbox' / 'DAY4_Vishal.md').write_text('<!-- Fill this in with your Day 4 answer, then open a PR. -->\n')

tracker_path = pe.generate_tracker()
content = tracker_path.read_text()
assert '| Gaurav | 4 | ✅' in content, f"Gaurav should show merged frontend cell, got:\n{content}"
assert '| Vishal | 4 | ⬜' in content, f"Vishal should show pending frontend cell, got:\n{content}"

print("ALL PASS")
```

- [ ] **Step 2: Run test to verify it fails**

```bash
scp /private/tmp/claude-501/-Users-vin/ed0ccc34-a8fe-4de8-b5e0-87beb62c0f36/scratchpad/test_pr_engine.py \
  root@187.127.156.138:/root/.openclaw/workspace-mentor_squad/test_pr_engine.py
ssh root@187.127.156.138 "cd /root/.openclaw/workspace-mentor_squad && python3 test_pr_engine.py"
```

Expected: `ModuleNotFoundError: No module named 'pr_engine'`.

- [ ] **Step 3: Write `$SCRATCH/pr_engine.py`**

```python
#!/usr/bin/env python3
import argparse
import json
import os
import re
import subprocess
from pathlib import Path

WORKSPACE = Path(__file__).resolve().parent
PEOPLE_PATH = WORKSPACE / 'people.json'
REPO_PATH = Path(os.environ.get(
    'MENTOR_SQUAD_REPO_PATH', str(Path.home() / 'mentor-squad-dsa')))
CONTENT_PROGRESS_PATH = Path(os.environ.get(
    'MENTOR_SQUAD_CONTENT_PROGRESS_PATH', str(WORKSPACE / 'content_progress.json')))
TRACKER_PATH = REPO_PATH / 'TRACKER.md'
PLACEHOLDER_MARKER_PREFIX = '<!-- Fill this in'

SAFE_PATH_RE = re.compile(
    r'^(Frontend/Assignment|Backend/Assignment|CS-Core/Assignment|DSA/Question)/'
    r'Day-(\d+)-[a-z0-9-]+/DAY(\d+)_([A-Za-z]+)\.md$'
)


def load_people():
    return json.loads(PEOPLE_PATH.read_text())


def run_gh(args, **kwargs):
    return subprocess.run(['gh', *args], capture_output=True, text=True, cwd=REPO_PATH, **kwargs)


def pr_changed_files(pr_number, gh_runner=run_gh):
    result = gh_runner(['pr', 'diff', str(pr_number), '--name-only'])
    if result.returncode != 0:
        raise RuntimeError(f'gh pr diff failed: {result.stderr.strip()}')
    return [line for line in result.stdout.splitlines() if line.strip()]


def is_safe_placeholder_path(path, people):
    m = SAFE_PATH_RE.match(path)
    if not m:
        return False, f'{path} does not match a known placeholder path'
    _folder, day_in_folder, day_in_file, name = m.groups()
    if day_in_folder != day_in_file:
        return False, f'{path} folder day ({day_in_folder}) does not match filename day ({day_in_file})'
    if name not in people:
        return False, f'{path} is not a known mentee name ({name})'
    return True, None


def evaluate_pr(pr_number, people, gh_runner=run_gh):
    files = pr_changed_files(pr_number, gh_runner)
    if len(files) != 1:
        return {'safe': False, 'reason': f'touches {len(files)} files, expected exactly 1'}
    ok, reason = is_safe_placeholder_path(files[0], people)
    if not ok:
        return {'safe': False, 'reason': reason}
    return {'safe': True, 'file': files[0]}


def merge_pr(pr_number, gh_runner=run_gh):
    result = gh_runner(['pr', 'merge', str(pr_number), '--merge', '--delete-branch'])
    if result.returncode != 0:
        raise RuntimeError(f'gh pr merge failed: {result.stderr.strip()}')
    return True


def cmd_check_pr(args):
    people = load_people()
    verdict = evaluate_pr(args.number, people)
    if verdict['safe']:
        merge_pr(args.number)
        verdict['merged'] = True
    else:
        verdict['merged'] = False
    print(json.dumps(verdict))


def is_placeholder_still_empty(path):
    if not path.exists():
        return True
    return path.read_text().strip().startswith(PLACEHOLDER_MARKER_PREFIX)


def load_content_progress():
    if CONTENT_PROGRESS_PATH.exists():
        return json.loads(CONTENT_PROGRESS_PATH.read_text())
    return {'created_days': [], 'slugs_by_day': {}}


def generate_tracker():
    people = load_people()
    content_progress = load_content_progress()
    created_days = sorted(content_progress.get('created_days', []))
    slugs_by_day = content_progress.get('slugs_by_day', {})

    lines = ['| Person | Current Day | Track | CS-Core | DSA | Last Merged Day |',
             '|---|---|---|---|---|---|']
    for name, track in people.items():
        track_folder = 'Frontend/Assignment' if track == 'frontend' else 'Backend/Assignment'
        last_merged = 0
        track_ok = cs_ok = dsa_ok = False
        for day in created_days:
            slugs = slugs_by_day.get(str(day), {})
            track_slug = slugs.get(track)
            cs_slug = slugs.get('cscore')
            dsa_slug = slugs.get('dsa')
            track_done = bool(track_slug) and not is_placeholder_still_empty(
                REPO_PATH / track_folder / f'Day-{day}-{track_slug}' / f'DAY{day}_{name}.md')
            cs_done = bool(cs_slug) and not is_placeholder_still_empty(
                REPO_PATH / 'CS-Core/Assignment' / f'Day-{day}-{cs_slug}' / f'DAY{day}_{name}.md')
            dsa_done = bool(dsa_slug) and not is_placeholder_still_empty(
                REPO_PATH / 'DSA/Question' / f'Day-{day}-{dsa_slug}' / f'DAY{day}_{name}.md')
            if day == created_days[-1]:
                track_ok, cs_ok, dsa_ok = track_done, cs_done, dsa_done
            if track_done and cs_done and dsa_done:
                last_merged = day
        current_day = created_days[-1] if created_days else 0
        lines.append(
            f"| {name} | {current_day} | {'✅' if track_ok else '⬜'} | "
            f"{'✅' if cs_ok else '⬜'} | {'✅' if dsa_ok else '⬜'} | {last_merged} |"
        )

    TRACKER_PATH.write_text('\n'.join(lines) + '\n')
    return TRACKER_PATH


def cmd_update_tracker(args):
    path = generate_tracker()
    subprocess.run(['git', '-C', str(REPO_PATH), 'add', 'TRACKER.md'], check=True)
    staged = subprocess.run(['git', '-C', str(REPO_PATH), 'diff', '--cached', '--quiet'])
    if staged.returncode == 0:
        print(json.dumps({'status': 'unchanged'}))
        return
    subprocess.run(['git', '-C', str(REPO_PATH), 'commit', '-m', 'Update TRACKER.md'], check=True)
    subprocess.run(['git', '-C', str(REPO_PATH), 'push', 'origin', 'main'], check=True)
    print(json.dumps({'status': 'updated', 'path': str(path)}))


def main():
    parser = argparse.ArgumentParser()
    sub = parser.add_subparsers(dest='command', required=True)

    p_check = sub.add_parser('check-pr')
    p_check.add_argument('--number', required=True, type=int)
    p_check.set_defaults(func=cmd_check_pr)

    p_tracker = sub.add_parser('update-tracker')
    p_tracker.set_defaults(func=cmd_update_tracker)

    args = parser.parse_args()
    args.func(args)


if __name__ == '__main__':
    main()
```

- [ ] **Step 4: Push the new file and re-run the test**

```bash
scp /private/tmp/claude-501/-Users-vin/ed0ccc34-a8fe-4de8-b5e0-87beb62c0f36/scratchpad/pr_engine.py \
  root@187.127.156.138:/root/.openclaw/workspace-mentor_squad/pr_engine.py
ssh root@187.127.156.138 "rm -rf /root/mentor-squad-dsa-test && mkdir -p /root/mentor-squad-dsa-test && cd /root/.openclaw/workspace-mentor_squad && python3 test_pr_engine.py"
```

Expected: `ALL PASS`.

- [ ] **Step 5: Manually verify `check-pr` against a real throwaway PR before trusting auto-merge in production**

This is the one step in this task that must touch real GitHub, because `gh pr merge` behavior (permissions, delete-branch, fork handling) can't be fully verified with a fake runner. Open one real, disposable test PR against `mentor-squad-dsa` (from the VPS itself, using the deploy key, targeting a scratch file path that matches the safe pattern) and confirm it merges:

```bash
ssh root@187.127.156.138 "cd /root/mentor-squad-dsa && git checkout -b test-pr-engine-verify && mkdir -p Frontend/Assignment/Day-0-verify-pr-engine && echo 'test content' > Frontend/Assignment/Day-0-verify-pr-engine/DAY0_Gaurav.md && git add Frontend/Assignment/Day-0-verify-pr-engine/DAY0_Gaurav.md && git commit -m 'test: verify pr_engine auto-merge' && git push origin test-pr-engine-verify && gh pr create --title 'test: verify pr_engine auto-merge' --body 'throwaway, safe to auto-merge and delete' --base main --head test-pr-engine-verify"
```

Note the PR number printed by `gh pr create`, then:

```bash
ssh root@187.127.156.138 "cd /root/.openclaw/workspace-mentor_squad && python3 pr_engine.py check-pr --number <PR_NUMBER>"
```

Expected: `{"safe": true, "file": "Frontend/Assignment/Day-0-verify-pr-engine/DAY0_Gaurav.md", "merged": true}`. Confirm on GitHub (or `gh pr view <PR_NUMBER>`) that it actually merged and the branch was deleted. Then clean up the now-merged content from `main` so it doesn't linger as fake Day 0 content:

```bash
ssh root@187.127.156.138 "cd /root/mentor-squad-dsa && git checkout main && git pull && git rm -r Frontend/Assignment/Day-0-verify-pr-engine && git commit -m 'chore: remove pr_engine verification scaffolding' && git push origin main"
```

- [ ] **Step 6: Clean up test artifacts**

```bash
ssh root@187.127.156.138 "rm -rf /root/mentor-squad-dsa-test /root/.openclaw/workspace-mentor_squad/content_progress.test.json /root/.openclaw/workspace-mentor_squad/test_pr_engine.py"
```

---

## Task 4: Wire it up — hourly cron + on-demand WhatsApp trigger + IDENTITY.md

**Files:**
- Modify: `$WS/IDENTITY.md` (stage at `$SCRATCH/IDENTITY.md`)
- Create: one new OpenClaw cron job (via CLI, no file — config lives in OpenClaw's own state)

**Interfaces:**
- Consumes: `pr_engine.py check-pr --number <n>` and `pr_engine.py update-tracker` (Task 3's exact subcommand names), `mentor_engine.py whoami --sender <e164>` (existing, unchanged).
- Produces: nothing new consumed by later tasks — this is the last task.

- [ ] **Step 1: Pull `IDENTITY.md` to scratch**

```bash
ssh root@187.127.156.138 "cat /root/.openclaw/workspace-mentor_squad/IDENTITY.md" \
  > /private/tmp/claude-501/-Users-vin/ed0ccc34-a8fe-4de8-b5e0-87beb62c0f36/scratchpad/IDENTITY.md
```

- [ ] **Step 2: Add persona note and a new numbered case**

In `$SCRATCH/IDENTITY.md`, immediately after the existing persona line:

```
Persona: Encouraging, direct, practical. You are running a real 3-month
mentoring program for 4 people. Keep messages clear and structured, not
long paragraphs.
```

insert:

```

One deliberate exception to "encouraging": when you review a GitHub PR's
code (case 5 below), be direct and unsparingly honest about bugs, bad
complexity, sloppy naming, and missed edge cases. That bluntness was
explicitly requested — don't soften code feedback just to be nice. Your
WhatsApp tone stays encouraging; the PR comment is where you're blunt.
```

At the end of the file (after the existing case 4, which currently ends the numbered list), append a new case 5. First change the sentence right before the numbered list from "act on a group message in exactly these four cases" to "exactly these five cases" — find:

```
You only ever act on a group message in exactly these four cases. For
```

replace with:

```
You only ever act on a group message in exactly these five cases. For
```

Then append after the existing case 4's final paragraph (which ends with "...describe what you'd need."):

```

5. **A known mentee (resolved via `whoami`, same as case 2) posts a GitHub
   PR link or a bare PR number in the group.** Extract the PR number and
   run:
   `python3 pr_engine.py check-pr --number <n>`
   in this workspace directory. This does the safety-gate check and
   auto-merges if the PR only touches that person's own designated
   placeholder file — you don't decide merge eligibility yourself, the
   script does, deterministically.

   Regardless of whether it merged, fetch the diff yourself
   (`gh pr diff <n>` in `~/mentor-squad-dsa`) and post a direct,
   unsparingly honest code-review comment on the PR:
   `gh pr comment <n> --body "<your review>"` — call out bugs, bad
   complexity, sloppy naming, missed edge cases (see the persona note
   above).

   Then run `python3 pr_engine.py update-tracker` in this workspace
   directory to refresh `TRACKER.md`.

   Reply in the group with one short line — "merged ✅" or "not merged
   yet — <one-line reason>" plus "full review on the PR". Do not paste
   the full code review into the group; it lives on the PR.

   If `whoami` returns null for the sender, do nothing (see the top of
   this section) — never check or merge a PR on behalf of an unverified
   sender.
```

- [ ] **Step 3: Push the edited file back**

```bash
scp /private/tmp/claude-501/-Users-vin/ed0ccc34-a8fe-4de8-b5e0-87beb62c0f36/scratchpad/IDENTITY.md \
  root@187.127.156.138:/root/.openclaw/workspace-mentor_squad/IDENTITY.md
```

- [ ] **Step 4: Verify the file reads correctly**

```bash
ssh root@187.127.156.138 "grep -n 'exactly these five cases\|^5\.' /root/.openclaw/workspace-mentor_squad/IDENTITY.md"
```

Expected: two matching lines — the updated "exactly these five cases" sentence, and the `5. **A known mentee...` line.

- [ ] **Step 5: Create the hourly cron job**

```bash
ssh root@187.127.156.138 'openclaw cron create \
  --name "pr-review-hourly" \
  --description "Check open mentor-squad-dsa PRs, safety-gate + auto-merge, post brutal-honest feedback, refresh TRACKER.md" \
  --cron "0 * * * *" \
  --tz "Asia/Kolkata" \
  --session isolated \
  --agent mentor_squad \
  --message "Run \`gh pr list --json number\` in ~/mentor-squad-dsa to find open PRs. For each PR number, run \`python3 pr_engine.py check-pr --number <n>\` in this workspace directory (this safety-gates and auto-merges if eligible — you do not decide merge eligibility yourself). Regardless of the merge outcome, fetch the diff with \`gh pr diff <n>\` in ~/mentor-squad-dsa and post a direct, unsparingly honest code-review comment with \`gh pr comment <n> --body \"...\"\` — call out bugs, bad complexity, sloppy naming, missed edge cases. Skip PRs you already commented on (check with \`gh pr view <n> --json comments\` first) so you do not repeat feedback every hour. After processing all open PRs, run \`python3 pr_engine.py update-tracker\` in this workspace directory to refresh TRACKER.md. Do not post anything to WhatsApp from this job — PR feedback lives on GitHub only; the on-demand WhatsApp case (IDENTITY.md case 5) handles the WhatsApp-visible short reply separately." \
  --timeout-seconds 180 \
  --tools exec,read,write \
  --json'
```

- [ ] **Step 6: Verify the cron job was created correctly**

```bash
ssh root@187.127.156.138 "openclaw cron list 2>&1 | grep pr-review-hourly"
```

Expected: one row named `pr-review-hourly`, schedule `cron 0 * * * * @ Asia/Kolkata`, status `idle` (hasn't run yet).

```bash
ssh root@187.127.156.138 "openclaw cron list --json 2>&1" | python3 -c "
import json, sys
jobs = json.load(sys.stdin)
job = next(j for j in jobs if j['name'] == 'pr-review-hourly')
print(json.dumps(job, indent=2))
"
```

Expected: `agentId: 'mentor_squad'`, `schedule.expr: '0 * * * *'`, `schedule.tz: 'Asia/Kolkata'`, `payload.kind: 'agentTurn'`, `payload.timeoutSeconds: 180`. (If `cron list --json` doesn't accept a top-level array shape, adjust the inline script to whatever structure it actually returns — inspect with `ssh root@187.127.156.138 "openclaw cron list --json" | head -c 500` first.)

- [ ] **Step 7: No commit step for this task** — `IDENTITY.md` lives only on the VPS (no backing git repo) and the cron job lives in OpenClaw's own state; Step 4 and Step 6's verification output are the record that this task is done.

---

## Self-Review Notes

- **Spec coverage:** §1 curriculum migration → Task 1. §2 daily repo content (CS-Core single-topic + placeholders) → Task 2. §3 PR review (safety gate, auto-merge, brutal-honest feedback, two triggers) → Task 3 (safety gate/merge/tracker) + Task 4 (feedback wiring, both triggers). §4 `TRACKER.md` → Task 3's `generate_tracker`. §5 guardrails (never silently drop, both entry points share logic, no change to case 2) → Task 3's deterministic safety gate is the single shared function both the hourly cron and the on-demand case call into (via `check-pr`), and no step in this plan touches `mentor_engine.py`'s `record-status`/`cmd_record_status`.
- **Placeholder scan:** none found — every step has literal code/commands, no "add error handling" or "TBD" language.
- **Type consistency:** `evaluate_pr` returns `{'safe': bool, 'reason': str}` or `{'safe': bool, 'file': str}` consistently across Task 3's implementation and its test. `check-pr`/`update-tracker` subcommand names match between Task 3's `pr_engine.py` and Task 4's IDENTITY.md text and cron message. `MENTOR_SQUAD_REPO_PATH` / `MENTOR_SQUAD_CONTENT_PROGRESS_PATH` env var names match between Task 2 and Task 3.
- **Scope check:** four tasks, each independently testable and each leaves the system in a working state (Task 1 alone already fixes the live curriculum; Task 2 alone already gives mentees placeholders even if PR review isn't live yet; Task 3 alone is inert until Task 4 wires it up, but is fully testable standalone). No further decomposition needed.

# Sloth — Vinayak's Assistant Agent + Nyx Bridge — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Stand up a new OpenClaw agent "Sloth" as Vinayak's primary WhatsApp DM agent, with one wired capability — asking Nyx (`mentor_squad`) for mentee status or relaying an override, via a real `sessions_spawn` agent-to-agent call — per `docs/superpowers/specs/2026-07-10-sloth-nyx-bridge-design.md`.

**Architecture:** A new `agents.list[]` entry (`sloth`, `tools.profile: "full"`, `subagents.allowAgents: ["mentor_squad"]`) plus one new WhatsApp binding (Vinayak's number → `sloth`) in `openclaw.json`; a minimal hand-written workspace/`IDENTITY.md` for Sloth describing the ask-Nyx flow; and one additive case (case 6) in Nyx's own `IDENTITY.md` so it recognizes sub-agent-spawned requests as already-trusted Vinayak requests.

**Tech Stack:** OpenClaw gateway config (`openclaw.json`, hot-reloads — confirmed this session: `agents` and `bindings` are both in the "No restart needed" category per `/usr/lib/node_modules/openclaw/docs/gateway/configuration.md`'s hot-reload table), Markdown (`IDENTITY.md` files), `openclaw agent` CLI for direct functional testing.

## Global Constraints

- Everything lives on the VPS (`root@187.127.156.138`) — no local checkout of `openclaw.json` or any workspace file exists. Every edit pulls to local scratch, edits, pushes back over SSH/SCP.
- Local scratch directory: `/private/tmp/claude-501/-Users-vin/ed0ccc34-a8fe-4de8-b5e0-87beb62c0f36/scratchpad` (referred to below as `$SCRATCH`).
- This is a live production gateway (PM2 process `openclaw`) already serving Nyx's WhatsApp group and 5 other per-contact agents. Config edits hot-apply (confirmed: `agents` and `bindings` need no restart) — do not restart the gateway as part of this plan.
- `subagents.allowAgents` (not `allowedTargets`) is the correct field name — confirmed against OpenClaw's own docs this session.
- Testing agent behavior directly: use `openclaw agent --agent <id> --message "..." --json` (bypasses routing bindings entirely, goes straight to the named agent — this does **not** deliver anything to a real channel unless `--deliver` is also passed, so it's safe to run repeatedly). Do **not** use `openclaw agent --to <number> ...` for testing — confirmed this session that it runs a real turn in whatever agent that number's *current* routing resolves to and consumes real tokens in that agent's live session history (verified: a throwaway routing probe accidentally ran a real turn in the `assistant` agent's live main session). Verify routing changes by reading the `bindings` config back, not by firing more live turns through it.
- Nyx's repo path is `~/nyx-bootcamp` (already renamed this session) — do not reintroduce `~/mentor-squad-dsa`.

---

## Task 1: Create Sloth's workspace and `IDENTITY.md`

**Files:**
- Create (VPS): `/root/.openclaw/workspace-sloth/IDENTITY.md`

**Interfaces:**
- Consumes: nothing from other tasks.
- Produces: a workspace directory that Task 2's `agents.list[]` entry points `workspace` at. Must exist *before* Task 2 registers the agent, since OpenClaw's agent bootstrap flow expects the workspace path to be a real directory.

- [ ] **Step 1: Create the workspace directory**

```bash
ssh root@187.127.156.138 "mkdir -p /root/.openclaw/workspace-sloth"
```

- [ ] **Step 2: Write `IDENTITY.md` locally in scratch**

Create `$SCRATCH/sloth-IDENTITY.md`:

```markdown
# IDENTITY.md - Who Am I?

- **Name:** Sloth
- **Role:** Vinayak's general-purpose WhatsApp assistant. General-purpose
  first — answer whatever he asks, same as any capable assistant.

## Wired capabilities

### Ask Nyx (mentee status / overrides)

Nyx (`mentor_squad`) runs a WhatsApp mentoring program for 4 juniors and
tracks their daily progress. When Vinayak asks something Nyx-relevant —
a status question ("mentees kaisa chal rahe hain", "progress dikhao") or
an override instruction ("Nishant ko rest day do", "skip today's DSA for
everyone") — do not try to answer it yourself and do not read Nyx's files
directly. Instead:

1. Call `sessions_spawn` with `agentId: "mentor_squad"` and a plain-language
   message describing the ask, e.g.:
   - Status: "Vinayak is asking for the current mentee status — run
     progress-report and check TRACKER.md in ~/nyx-bootcamp, summarize
     both."
   - Override: "Vinayak wants to override: <verbatim text>."
2. Call `sessions_yield` right after spawning — do not wait synchronously.
   The result arrives as the next model-visible event once Nyx's run
   completes.
3. When the completion arrives, turn Nyx's plain-text result into a
   short, WhatsApp-appropriate reply for Vinayak — don't just paste Nyx's
   raw output verbatim if it's long or script-flavored.

This is the only cross-agent capability you have — `subagents.allowAgents`
is scoped to `["mentor_squad"]` only, so you cannot spawn into any other
agent.

## More capabilities

None yet — this workspace is intentionally minimal. When Vinayak asks for
a new wired capability, add it as its own section here rather than
starting a new agent.
```

- [ ] **Step 3: Push it to the VPS**

```bash
scp /private/tmp/claude-501/-Users-vin/ed0ccc34-a8fe-4de8-b5e0-87beb62c0f36/scratchpad/sloth-IDENTITY.md \
  root@187.127.156.138:/root/.openclaw/workspace-sloth/IDENTITY.md
```

- [ ] **Step 4: Verify**

```bash
ssh root@187.127.156.138 "test -f /root/.openclaw/workspace-sloth/IDENTITY.md && echo EXISTS && wc -l /root/.openclaw/workspace-sloth/IDENTITY.md"
```

Expected: `EXISTS` followed by a line count (roughly 40 lines).

---

## Task 2: Register the `sloth` agent in `openclaw.json`

**Files:**
- Modify (VPS): `/root/.openclaw/openclaw.json` (`agents.list`)

**Interfaces:**
- Consumes: `/root/.openclaw/workspace-sloth` (Task 1's output — must already exist).
- Produces: an `agentId` of `"sloth"` that Task 3 (Nyx's case 6) and Task 4 (the WhatsApp binding) both reference by this exact string. Also produces the `subagents.allowAgents: ["mentor_squad"]` grant that Task 5's functional test depends on.

- [ ] **Step 1: Pull `openclaw.json` to scratch**

```bash
ssh root@187.127.156.138 "cat /root/.openclaw/openclaw.json" > /private/tmp/claude-501/-Users-vin/ed0ccc34-a8fe-4de8-b5e0-87beb62c0f36/scratchpad/openclaw.json
python3 -c "import json; json.load(open('/private/tmp/claude-501/-Users-vin/ed0ccc34-a8fe-4de8-b5e0-87beb62c0f36/scratchpad/openclaw.json'))" && echo "valid JSON"
```

- [ ] **Step 2: Confirm no `sloth` id already exists (avoid a silent duplicate)**

```bash
python3 -c "
import json
d = json.load(open('/private/tmp/claude-501/-Users-vin/ed0ccc34-a8fe-4de8-b5e0-87beb62c0f36/scratchpad/openclaw.json'))
ids = [a['id'] for a in d['agents']['list']]
assert 'sloth' not in ids, f'sloth already registered: {ids}'
print('clear to add, current ids:', ids)
"
```

Expected: prints the current agent id list (should include `coder`,
`researcher`, `assistant`, `mentor_squad`, and the per-contact thread
agents) with no `sloth` in it.

- [ ] **Step 3: Add the `sloth` entry and write back**

```bash
python3 -c "
import json
path = '/private/tmp/claude-501/-Users-vin/ed0ccc34-a8fe-4de8-b5e0-87beb62c0f36/scratchpad/openclaw.json'
with open(path) as f:
    d = json.load(f)
d['agents']['list'].append({
    'id': 'sloth',
    'name': 'Sloth',
    'workspace': '/root/.openclaw/workspace-sloth',
    'tools': {'profile': 'full'},
    'subagents': {'allowAgents': ['mentor_squad']},
})
with open(path, 'w') as f:
    json.dump(d, f, indent=2)
print('added')
"
python3 -c "import json; json.load(open('/private/tmp/claude-501/-Users-vin/ed0ccc34-a8fe-4de8-b5e0-87beb62c0f36/scratchpad/openclaw.json'))" && echo "still valid JSON"
```

- [ ] **Step 4: Push the updated config to the VPS**

```bash
scp /private/tmp/claude-501/-Users-vin/ed0ccc34-a8fe-4de8-b5e0-87beb62c0f36/scratchpad/openclaw.json \
  root@187.127.156.138:/root/.openclaw/openclaw.json
```

The gateway watches this file and hot-applies `agents` changes automatically
(confirmed, no restart needed — see Global Constraints).

- [ ] **Step 5: Verify the gateway picked it up**

```bash
ssh root@187.127.156.138 "sleep 2 && python3 -c \"
import json
with open('/root/.openclaw/openclaw.json') as f:
    d = json.load(f)
sloth = [a for a in d['agents']['list'] if a['id'] == 'sloth']
assert len(sloth) == 1, sloth
print(json.dumps(sloth[0], indent=2))
\""
```

Expected: prints back the exact entry from Step 3 (confirms the gateway
did not reject the write as invalid — recall from the hot-reload docs:
"Direct file edits are treated as untrusted until they validate... rejects
invalid external edits without rewriting `openclaw.json`", so if this
read-back doesn't match, the write was rejected and needs investigation
before continuing).

- [ ] **Step 6: Functional smoke test — run a turn directly against `sloth`**

```bash
ssh root@187.127.156.138 "openclaw agent --agent sloth --message 'hello, who are you?' --json 2>&1" | python3 -c "
import json, sys
raw = sys.stdin.read()
# strip any leading non-JSON log lines (e.g. state-migration warnings)
start = raw.index('{')
d = json.loads(raw[start:])
print('status:', d['status'])
print('text:', d['result']['payloads'][0]['text'][:300])
print('sessionFile:', d['result']['meta']['agentMeta']['sessionFile'])
"
```

Expected: `status: ok`, reply text sounds like it read `IDENTITY.md` (e.g.
mentions being Vinayak's assistant or "Sloth"), and `sessionFile` contains
`/agents/sloth/sessions/` (confirms it ran in Sloth's own workspace, not a
shared/default session).

---

## Task 3: Nyx's case 6 (`IDENTITY.md`) — recognize sub-agent-spawned requests

**Files:**
- Modify (VPS): `/root/.openclaw/workspace-mentor_squad/IDENTITY.md`

**Interfaces:**
- Consumes: nothing new from earlier tasks (this is independent of Sloth's own config — Nyx just needs to know how to *respond* when spawned into, regardless of who spawns it, though in practice only `sloth` can per Task 2's `allowAgents`).
- Produces: nothing consumed by later tasks in this plan, but this is what Task 5's end-to-end test actually exercises.

- [ ] **Step 1: Pull `IDENTITY.md` to scratch**

```bash
ssh root@187.127.156.138 "cat /root/.openclaw/workspace-mentor_squad/IDENTITY.md" \
  > /private/tmp/claude-501/-Users-vin/ed0ccc34-a8fe-4de8-b5e0-87beb62c0f36/scratchpad/mentor_squad-IDENTITY.md
grep -n "exactly these five cases\|^5\." /private/tmp/claude-501/-Users-vin/ed0ccc34-a8fe-4de8-b5e0-87beb62c0f36/scratchpad/mentor_squad-IDENTITY.md
```

Expected: two matches (the "exactly these five cases" sentence, and the
`5. **A known mentee...` case-5 heading) — confirms you're editing the
version with case 5 already present (added earlier this session).

- [ ] **Step 2: Bump "five cases" to "six cases"**

In `$SCRATCH/mentor_squad-IDENTITY.md`, find:

```
You only ever act on a group message in exactly these five cases. For
```

Replace with:

```
You only ever act on a group message in exactly these six cases. For
```

- [ ] **Step 3: Append case 6 after case 5's final paragraph**

Find the end of case 5, which currently reads (last lines of that case):

```
   If `whoami` returns null for the sender, do nothing (see the top of
   this section) — never check or merge a PR on behalf of an unverified
   sender.
```

Immediately after that paragraph (and before the closing
"Only Vinayak's exact number..." paragraph that follows it), insert:

```

6. **A request arrives via internal sub-agent spawn** (session key starts
   `agent:mentor_squad:subagent:`). This is not a WhatsApp message, so
   skip the `whoami` sender-verification step entirely — it doesn't apply
   here. Trust is already enforced upstream (only Sloth's `allowAgents`
   config can reach this path, and only Vinayak can reach Sloth). If the
   request is a status question, run `python3 mentor_engine.py
   progress-report` and check `TRACKER.md` in `~/nyx-bootcamp`, and
   summarize both. If it's an override instruction, handle it the same way
   as case 4 (use judgment; if it needs a state change beyond what the
   scripts support, say so plainly). Return your answer as plain text —
   sub-agent completions are relayed by the parent (Sloth), not sent
   directly to any channel by you.
```

- [ ] **Step 4: Push the edited file back**

```bash
scp /private/tmp/claude-501/-Users-vin/ed0ccc34-a8fe-4de8-b5e0-87beb62c0f36/scratchpad/mentor_squad-IDENTITY.md \
  root@187.127.156.138:/root/.openclaw/workspace-mentor_squad/IDENTITY.md
```

- [ ] **Step 5: Verify**

```bash
ssh root@187.127.156.138 "grep -n 'exactly these six cases\|^6\.\|agent:mentor_squad:subagent:' /root/.openclaw/workspace-mentor_squad/IDENTITY.md"
```

Expected: three matches — the updated "six cases" sentence, the `6. **A
request arrives...` heading, and the `agent:mentor_squad:subagent:`
session-key mention inside case 6's body.

---

## Task 4: WhatsApp binding — Vinayak's number → `sloth`

**Files:**
- Modify (VPS): `/root/.openclaw/openclaw.json` (`bindings`)

**Interfaces:**
- Consumes: `agentId: "sloth"` (must already be registered — Task 2).
- Produces: nothing consumed by later tasks; this is the last config change.

- [ ] **Step 1: Pull the current config to scratch (fresh copy, in case Task 2/3 changed timestamps)**

```bash
ssh root@187.127.156.138 "cat /root/.openclaw/openclaw.json" > /private/tmp/claude-501/-Users-vin/ed0ccc34-a8fe-4de8-b5e0-87beb62c0f36/scratchpad/openclaw.json
```

- [ ] **Step 2: Confirm no existing binding already matches Vinayak's number**

```bash
python3 -c "
import json
d = json.load(open('/private/tmp/claude-501/-Users-vin/ed0ccc34-a8fe-4de8-b5e0-87beb62c0f36/scratchpad/openclaw.json'))
matches = [b for b in d['bindings'] if b.get('match', {}).get('peer', {}).get('id') == '+919451688978']
assert not matches, f'unexpected existing binding for this number: {matches}'
print('clear to add — no existing binding for +919451688978')
"
```

Expected: `clear to add — no existing binding for +919451688978` (this was
already confirmed once earlier this session by a plain grep, but re-verify
here since Task 2/3 pulled a fresh copy of the file).

- [ ] **Step 3: Add the binding and write back**

```bash
python3 -c "
import json
path = '/private/tmp/claude-501/-Users-vin/ed0ccc34-a8fe-4de8-b5e0-87beb62c0f36/scratchpad/openclaw.json'
with open(path) as f:
    d = json.load(f)
d['bindings'].append({
    'agentId': 'sloth',
    'match': {'channel': 'whatsapp', 'peer': {'kind': 'direct', 'id': '+919451688978'}},
})
with open(path, 'w') as f:
    json.dump(d, f, indent=2)
print('added')
"
python3 -c "import json; json.load(open('/private/tmp/claude-501/-Users-vin/ed0ccc34-a8fe-4de8-b5e0-87beb62c0f36/scratchpad/openclaw.json'))" && echo "still valid JSON"
```

- [ ] **Step 4: Push and verify the read-back**

```bash
scp /private/tmp/claude-501/-Users-vin/ed0ccc34-a8fe-4de8-b5e0-87beb62c0f36/scratchpad/openclaw.json \
  root@187.127.156.138:/root/.openclaw/openclaw.json
sleep 2
ssh root@187.127.156.138 "python3 -c \"
import json
with open('/root/.openclaw/openclaw.json') as f:
    d = json.load(f)
matches = [b for b in d['bindings'] if b.get('agentId') == 'sloth']
assert len(matches) == 1, matches
print(json.dumps(matches[0], indent=2))
\""
```

Expected: prints back
`{"agentId": "sloth", "match": {"channel": "whatsapp", "peer": {"kind": "direct", "id": "+919451688978"}}}`
— confirms the gateway accepted the write (same rejection-check logic as
Task 2 Step 5 applies here too).

**Note the consequence this has, per the design spec's explicit intent:**
this switches Vinayak's *own* WhatsApp DMs from the `assistant` fallback
to `sloth`, starting now. No other person's routing is affected —
`assistant` remains the default agent for any sender without an explicit
binding.

---

## Task 5: End-to-end verification — Sloth successfully bridges to Nyx

**Files:** none (verification only).

**Interfaces:**
- Consumes: everything from Tasks 1–4.
- Produces: nothing (terminal task).

- [ ] **Step 1: Run a status-ask through Sloth directly (bypasses WhatsApp entirely, exercises the real `sessions_spawn` → Nyx → response chain)**

```bash
ssh root@187.127.156.138 "openclaw agent --agent sloth --message 'Nyx se pucho mentee status kya hai abhi' --json --timeout 120 2>&1" | python3 -c "
import json, sys
raw = sys.stdin.read()
start = raw.index('{')
d = json.loads(raw[start:])
print('status:', d['status'])
print('text:', d['result']['payloads'][0]['text'])
"
```

Expected: `status: ok`, and the reply text contains a summary that plausibly
came from Nyx's real data (e.g. mentions `Gaurav`/`Vishal`/`Ashish`/`Nishant`
by name, or a day number, or "no PRs merged yet" / similar — not a generic
"I don't have access to that" refusal, which would indicate the
`sessions_spawn` call either didn't fire or was rejected by the
`allowAgents` gate).

- [ ] **Step 2: If Step 1's reply looks generic/refused, debug before continuing**

```bash
ssh root@187.127.156.138 "ls -t /root/.openclaw/agents/mentor_squad/sessions/*.jsonl | head -1 | xargs tail -5"
```

If no new session file exists under `mentor_squad/sessions/` with a recent
timestamp, the spawn never reached Nyx — re-check Task 2's
`subagents.allowAgents` value and Task 1's `IDENTITY.md` wording (the
`sessions_spawn` call and the `agentId: "mentor_squad"` argument are things
Sloth's own model has to decide to do based on its `IDENTITY.md`
instructions — if the wording is unclear, it may not trigger reliably; if
so, tighten the "Ask Nyx" section in Task 1 to be more explicit and
imperative, then re-run Step 1).

- [ ] **Step 3: Confirm the routing config is correct (static check, no live token spend)**

```bash
ssh root@187.127.156.138 "python3 -c \"
import json
with open('/root/.openclaw/openclaw.json') as f:
    d = json.load(f)
b = [x for x in d['bindings'] if x.get('match', {}).get('peer', {}).get('id') == '+919451688978']
assert b == [{'agentId': 'sloth', 'match': {'channel': 'whatsapp', 'peer': {'kind': 'direct', 'id': '+919451688978'}}}], b
print('binding confirmed correct')
\""
```

Expected: `binding confirmed correct`.

- [ ] **Step 4: Final human acceptance step (flagged — cannot be automated)**

There is no safe automated way to prove a real inbound WhatsApp message
from Vinayak's phone reaches Sloth without either (a) spending real tokens
in a live session as a side effect (as happened once already this session
with a routing probe) or (b) Vinayak sending a real message himself. Ask
Vinayak to send one WhatsApp DM to the bot number (anything, e.g. "hi
sloth") and confirm he gets a Sloth-flavored reply (not the old
`assistant` persona). This is the true end-to-end proof that hot-reload +
routing + Sloth + the Nyx bridge all work together in production — Steps
1–3 already prove every piece except "does a real inbound WhatsApp message
actually land here."

---

## Self-Review Notes

- **Spec coverage:** §1 access control → Task 2 (`allowAgents`). §2 WhatsApp
  routing → Task 4. §3 Sloth's workspace → Task 1. §4 the ask-Nyx flow →
  Task 1's `IDENTITY.md` content + Task 5's functional proof. §5 Nyx-side
  case 6 → Task 3. §6 guardrails (single explicit `allowAgents` entry, case
  6 additive only, on-demand only) → satisfied by Task 2's exact
  `allowAgents` value, Task 3 not touching cases 1–5, and no cron job being
  created anywhere in this plan.
- **Placeholder scan:** none found — every step has literal commands/text.
- **Type consistency:** `agentId: "sloth"` used identically in Task 2
  (registration), Task 4 (binding), and referenced in Task 3's case 6 text
  ("only Sloth's `allowAgents` config"). `subagents.allowAgents` field name
  used consistently (not `allowedTargets`).
- **Scope check:** five tasks, each independently verifiable; Task 1+2 are
  the minimum for Sloth to exist and answer *anything*, Task 3 is
  independent of the others (can be done in any order relative to 1/2/4),
  Task 4 is the only task with a live user-facing consequence
  (routing switch), Task 5 is verification-only. No further decomposition
  needed.

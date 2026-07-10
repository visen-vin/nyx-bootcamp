# Sloth — Vinayak's Assistant Agent + Nyx Bridge — Design

Status: approved by Vinayak, 2026-07-10.

## Context

Nyx (agentId `mentor_squad`) only responds inside one specific WhatsApp
group (`bindings` in `openclaw.json` route `mentor_squad` to
`peer.kind: "group", id: "120363427932565487@g.us"` only — confirmed this
session, no direct/DM binding exists for it). Vinayak currently has no
WhatsApp binding of his own either — his DMs fall through to the default
agent (`assistant`, `agents.list[].default: true`).

Vinayak wants a way to ask, from WhatsApp, "what's the mentee status" or
give an override ("skip Nishant's target today") without being in the
mentoring group himself, and without duplicating Nyx's tracking logic
elsewhere. He also wants this to be the seed of a broader personal
assistant agent for himself — this spec scopes only its first capability
(the Nyx bridge); further capabilities are explicitly deferred.

## Scope

In scope:
- A new agent, `sloth` — general-purpose (same capability ceiling as
  `assistant`), and becomes Vinayak's **primary** WhatsApp DM agent.
- One wired capability: ask Nyx (`mentor_squad`) for mentee status, or
  relay an override instruction to it, via a real agent-to-agent call
  (`sessions_spawn` targeting `agentId: "mentor_squad"`) — not by Sloth
  directly reading `progress.json`/`TRACKER.md` itself. Confirmed with
  Vinayak: he explicitly wants literal agent-to-agent conversation, not a
  deterministic passthrough, so Nyx's own judgment is applied to the
  request.
- A new case in Nyx's `IDENTITY.md` so it recognizes and handles requests
  that arrive via sub-agent spawn (a different trust/identification model
  than its existing WhatsApp-message cases).

Out of scope (explicitly deferred — user chose "just this one capability
now" over listing everything up front):
- Any other capability for Sloth beyond the Nyx bridge.
- A scheduled/proactive digest — on-demand only, confirmed with Vinayak.
- A second WhatsApp account/number, or Telegram access for Sloth —
  Vinayak chose "Sloth becomes primary DM agent" over the alternatives,
  so `assistant` stops being Vinayak's personal DM agent (it remains the
  fallback default for any other unbound sender — no other person is
  affected).

## 1. Access control model

OpenClaw's `sessions_spawn` cross-agent targeting is gated by an
allowlist on the **requesting** agent, not the target:
`agents.list[].subagents.allowAgents` (default: only the requester can
target itself). There is no separate inbound restriction on Nyx's side —
whichever agent has `mentor_squad` in its own `allowAgents` can spawn
into it.

`sloth`'s `agents.list[]` entry gets:
```json5
{
  id: "sloth",
  name: "Sloth",
  workspace: "/root/.openclaw/workspace-sloth",
  subagents: { allowAgents: ["mentor_squad"] },
}
```
No other existing agent gets `mentor_squad` added to its `allowAgents` —
this is the only path in.

## 2. WhatsApp routing change

New binding, added to `bindings`:
```json5
{ match: { channel: "whatsapp", peer: { kind: "direct", id: "+919451688978" } }, agentId: "sloth" }
```
Per OpenClaw's routing precedence (exact peer match beats default-agent
fallback), this makes `sloth` — not `assistant` — the agent for all of
Vinayak's future WhatsApp DMs on that number. No other person's routing
changes; `assistant` remains the default for anyone without an explicit
binding.

## 3. Sloth's own workspace

Minimal bootstrap: `IDENTITY.md` describing Sloth as Vinayak's
general-purpose WhatsApp assistant, with one wired capability documented
(ask-Nyx). Tools profile: `full` (same as `assistant`/`coder`), so it can
act as a genuine daily-driver assistant, not a narrow single-purpose bot.
Written to be extended later without a redesign — future capabilities get
their own case/section added to this same `IDENTITY.md`, not a new agent.

## 4. The ask-Nyx flow

1. Vinayak DMs Sloth something Nyx-relevant (status question or an
   override instruction).
2. Sloth calls `sessions_spawn` with `agentId: "mentor_squad"` and a
   message describing the ask in plain language (e.g. "Vinayak is asking
   for the current mentee status — run progress-report and check
   TRACKER.md, summarize" or "Vinayak wants to override: <verbatim
   text>").
3. Per `sessions_spawn` semantics, this is non-blocking — Sloth must call
   `sessions_yield` after spawning so the turn ends cleanly and the
   completion arrives as the next model-visible event, instead of Sloth
   sitting idle mid-turn waiting synchronously.
4. Nyx's subagent run executes inside Nyx's own workspace/context (its
   own `IDENTITY.md`, its own scripts) and returns plain text.
5. On completion, Sloth synthesizes a WhatsApp-appropriate reply from
   Nyx's result and sends it to Vinayak.

## 5. Nyx-side change (`IDENTITY.md`)

New case (case 6), additive — does not change any of the 5 existing
cases:

> **A request arrives via internal sub-agent spawn** (session key starts
> `agent:mentor_squad:subagent:`). This is not a WhatsApp message, so
> skip the `whoami` sender-verification step entirely — it doesn't apply
> here. Trust is already enforced upstream (only Sloth's `allowAgents`
> config can reach this path, and only Vinayak can reach Sloth). If the
> request is a status question, run `python3 mentor_engine.py
> progress-report` and check `TRACKER.md` in `~/nyx-bootcamp`, and
> summarize both. If it's an override instruction, handle it the same way
> as case 4 (use judgment; if it needs a state change beyond what the
> scripts support, say so plainly). Return your answer as plain text —
> sub-agent completions are relayed by the parent (Sloth), not sent
> directly to any channel by you.

## 6. Guardrails

- `sloth`'s `allowAgents: ["mentor_squad"]` is the single, explicit trust
  boundary — no broader `["*"]` grant.
- Nyx's case 6 does not bypass or weaken cases 1–5; it's purely additive
  for a different trigger type.
- No scheduled/automatic trigger — on-demand only, per Vinayak's explicit
  choice.

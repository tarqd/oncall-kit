<!-- SPDX-License-Identifier: Apache-2.0 -->

# MVP: validating the two LaunchDarkly seams from a Claude Code session

**Status:** proposal · **Date:** 2026-09-16 · **Companion to:** `launchdarkly-integration.md`

Decisions and constraints:

- **Audience: internal dogfood first.** Evidence, not a product surface.
- **Operator surface: Claude Tag was the preference — but attaching connectors
  to Claude Tag needs a Claude org Owner, which isn't self-serve.** So Claude
  Tag moves out of the MVP.

That last constraint reshapes the sequencing and, as it happens, improves the
first milestone. **Neither seam needs Claude Tag.** Claude Tag is how the kit
gets *deployed* into a channel; it is not how either seam gets *validated*. The
kit already treats Claude Code as an equal surface — `README.md` offers
"`@Claude run the oncall-setup skill` (or type the same in a Claude Code
session opened in the repo)", and `test-fixtures/RUNBOOK.md` exists precisely to
run the whole thing with "zero connections, zero admin."

---

## 1. Three tiers, split by what you can do alone

| Tier | What it is | Needs | LaunchDarkly involved |
|---|---|---|---|
| **0** | Claude Code against `test-fixtures/` — 48 fictional incidents, an answer key, the full setup run and a graded replay | Nothing. A clone and a Claude Code session. | No |
| **1 — the MVP** | Claude Code + your own LaunchDarkly project: read `oncall-policy`, use real flag history and observability as triage evidence, emit verdict events from graded replays | Your own LD account and a read-only token. Normally self-serve. | **Yes, both seams** |
| **2 — deferred** | Claude Tag in a real incident channel: live alerts, ambient operation, Slack reaction verdicts | A Claude org Owner to attach connectors, plus a host team | Yes |

The point of the split: **Tier 1 produces the artifact that buys Tier 2.** Don't
ask an Owner for connector access on a hypothesis — ask with a graded replay
table in hand. The kit says as much about its own grading table: it "is also
what you show teammates who ask whether this is worth adopting."

Tier 0 is worth half a day on its own. It's the kit's regression test, it needs
no permissions, and it tells you whether the playbook machinery is worth
wiring to anything.

---

## 2. Tier 1 is still both seams

Nothing about the two seams from the previous draft changes. Only their front
ends do.

### Seam 1 — policy as a flag

One JSON flag, `oncall-policy`, in your own LD project, replacing the prose
tables in `ONCALL.md`:

```jsonc
{
  "paging": {
    "criteria": [
      { "signal": "pipeline-error-rate", "page_when": "> 2%",
        "sustain_minutes": 5, "exempt_when": "inside a posted deploy window" }
    ],
    "fallback": "mention @escalation in channel",
    "escalation_timeout_minutes": 15
  },
  "routing": { "test-failures": "@build-team", "*": "@escalation" },
  "posture": { "shadow": true, "paging_enabled": false, "weather_enabled": false }
}
```

**Read path, in order of preference:**

1. **A read-only LD API token + curl or a tiny script.** Fully self-serve, no
   MCP, no admin, guaranteed to work. Use a Reader base role or a custom role
   with only the view actions the playbooks need.
2. **The LD MCP server added to your own Claude Code** —
   `https://mcp.launchdarkly.com/mcp/launchdarkly` over OAuth, authorized as
   you, inheriting your own role. Adding an MCP server to Claude Code is
   normally a per-user or per-project setting and needs no Claude org Owner —
   unlike Claude Tag. (Caveat: some orgs restrict MCP by managed policy. If
   yours does, option 1 still works.)

Either way **scope it read-only.** Per the companion doc's §10, the hosted MCP
server exposes 10 write tools including three unconfirmed permanent deletes,
and its docs recommend a Writer or Developer role. None of that belongs in an
agent bound by rule 1. RBAC is the enforcing boundary; the local server also
offers `--scope read` and `--tool` allowlisting if you self-host it.

**Keep `ONCALL.md`.** Only the *values* move. The lifecycle rules, the
out-of-band path, the read-only guarantee, the routing rationale stay in the
file, and `ONCALL.md` gains a line naming which fields now live in the flag —
so there's one obvious place to look. Rule 7 says files are truth; this is the
one deliberate exception and it needs writing down, not inferring.

Claude never writes this flag. A flag is a better home for the numbers than
prose, not a licence for the agent to pick them (rule 15).

### Seam 2 — verdicts via git, shipped by a script

The mechanism survives the loss of Slack; only the verdict's *source* changes.

```
graded replay → Claude appends shadow-log row → commit → script/CI → LD metric events
```

1. **Grade a diagnosis per `eval/replay.md`.** In Claude Code this is already
   the documented procedure: open a NEW session, paste the blinded input, the
   diagnosis as posted, and what actually happened, and grade ✅/⚠️/❌/🚫 with
   the grader arguing for the lower grade where torn. A fresh session is what
   makes the grade credible — the session that wrote a diagnosis never grades
   it.
2. **Claude appends a row to `eval/shadow-log.md`.** This file already exists
   for this purpose, already carries date / incident / grade / streak / grader
   columns, and is already an enumerated permitted output under rule 1. No new
   permission, no carve-out.
3. **A script ships new rows as LD metric events.** Run it by hand at first; a
   CI job later. A machine writes to LaunchDarkly; the agent never does.

| Event | Source in Tier 1 | Carries |
|---|---|---|
| `oncall.diagnosis.posted` | the replay run | failure class, confidence, evidence-link count |
| `oncall.operator.verdict` | the grader's ✅/⚠️/❌/🚫 | grade, grader, streak |
| `oncall.fix.verified` | the holdout's known outcome | whether the proposed fix matched what worked |
| `oncall.page.decision` | `paging-log.md` | decided, clause that tripped, signal values |

Three properties are why this shape is right rather than merely expedient:

- **It reuses the kit's existing safety envelope entirely.** Every write is to a
  file rule 1 already names; the agent's credential set doesn't grow.
- **The verdict is reviewable before it becomes data** — a human sees the row in
  a diff. A direct API write would lose that.
- **`oncall.fix.verified` and any 🚫 are the two that matter.** They're outcome
  signals rather than process signals, and together they're the entire safety
  case for ever graduating this to Tier 2.

One thing to confirm: sending metric events needs an SDK key for some
environment (or the events API). That's normally self-serve for someone with
project access, but check before planning on it — and use a non-production
environment.

---

## 3. Why replay-first is the better first test anyway

Losing live incidents sounds like a downgrade. It isn't, for this milestone:

- **It's blind and repeatable.** Holdouts are incidents the playbooks were never
  mined from, with a known outcome. You get a grade, not an impression.
- **It's the companion doc's highest-leverage step** (§8 step 3) arriving first
  by accident of constraint — the fixture corpus plus the grading rubric is
  exactly the "evolution of the kit" loop.
- **It generates evidence density live triage can't.** Ten graded holdouts in an
  afternoon versus ten real incidents over a month.
- **It exercises the paging dimension deliberately.** `eval/replay.md` requires
  holdouts spanning failure classes *and* including page-severity incidents,
  and requires you to say so explicitly when history has none — so the zero-🚫
  bar isn't vacuous.

A note for later: once you're grading at volume, the rubric is a natural fit for
a **custom LD judge** invoked programmatically (`judge.evaluate(input, output)`,
0.0–1.0 with reasoning). Generation has to stay local — a diagnosis needs live
tool calls, which LD-side offline-eval generation can't do. Keep the human
confirmation `eval/replay.md` requires; the judge replaces the scoring, not the
authority.

## 4. What Tier 1 cannot tell you

State these up front so the results don't get oversold:

- **Whether operators actually react.** The Slack reaction loop is untested
  until Tier 2. Grader verdicts are a proxy, and a friendlier one than reality.
- **Whether ambient operation is tolerable.** Noise, timing, and alert-storm
  batching only show up with live traffic.
- **Whether the rubber-stamp problem appears.** `eval/replay.md` names
  acceptance-without-verification as the failure a replay score will never
  catch. A graded replay is structurally incapable of measuring it.
- **Time-to-first-diagnosis under real conditions.**

## 5. Prerequisites

Much shorter than the Claude-Tag version:

1. **Your own LD project** and a read-only token (or MCP added to your Claude
   Code). Self-serve.
2. **An SDK key in a non-production environment** for the events, or a decision
   to defer Seam 2's shipping step and just accumulate shadow-log rows.
3. **Real incident history to mine**, if you want Tier 1 on real playbooks
   rather than fixture ones — 30–90 days of resolved incidents you can read.
   Without it, run Tier 1 on the fixtures and treat the LD seams as the thing
   under test rather than the playbooks.

Nothing here needs another person's approval.

## 6. Definition of done

Tier 1 is done when it has produced evidence and an artifact:

- **Phase 3's own gate cleared** on blind holdouts — ≥70% ✅+⚠️ and zero 🚫
  (`eval/replay.md`), written up in `eval/replay-results.md`.
- **The policy flag was read and honored** — a diagnosis whose routing or paging
  call demonstrably came from `oncall-policy` and changed when the flag changed.
- **Verdict events visible on an LD dashboard**, with helpful-rate broken out by
  failure class.
- **A one-page write-up** answering: did the kit's playbooks beat unaided
  triage on these incidents, and what would Tier 2 cost. That page is the ask
  for connector access.

A Tier 1 that ends in "it seemed good" has failed regardless of how the agent
performed.

## 7. Sequence

1. **Tier 0.** `test-fixtures/RUNBOOK.md`, one instruction, zero permissions.
   Watch the full setup run and the graded replay against the answer key.
   *Half a day.*
2. **Add the LD read path** — token or MCP — and confirm Claude Code can read
   flag change history and observability data in your project. *An hour.*
3. **Create `oncall-policy`** and point the triage skill's context load at it.
   *A day.*
4. **Run a real replay:** 5–10 holdouts from your own history if you have it,
   fixtures if not, graded in fresh sessions per `eval/replay.md`. *An
   afternoon.*
5. **Write the shadow-log → events script**, run it by hand, build the
   dashboard. The only code in the MVP. *A day or two.*
6. **Write the one-pager.** Then decide whether Tier 2 is worth asking for.
7. **Side experiment, whenever:** Vega already reads `CLAUDE.md`/`AGENTS.md`
   from a connected repo as repository instructions, so pointing it at this repo
   with auto-remediation in read-only mode puts the kit's rules on a second
   agent with no code at all — and the verdict schema makes the two comparable.
   That's the cheapest real test of agent-agnosticism available. Needs Vega
   enabled on the account; keep it out of MVP scope.

Steps 1 and 2 are reversible and cost almost nothing. Step 1 is where you find
out whether any of the rest is worth doing.

## 8. Risks

- **Managed policy may block MCP in Claude Code too.** Mitigated: the read-only
  API token path needs no MCP at all.
- **SDK key access for events.** If it's not self-serve, defer Seam 2's shipping
  step — shadow-log rows accumulate in git either way and can be replayed into
  LaunchDarkly later. The seam is designed so the queue survives the shipper
  being absent.
- **Fixture-only results are weaker evidence.** The 48 fictional incidents
  validate the machinery, not your team's failure modes. Say which corpus the
  numbers came from, every time.
- **Thin or missing page-severity holdouts** make the zero-🚫 bar vacuous.
  `eval/replay.md` requires stating this explicitly rather than letting "zero
  harmful" read as tested.
- **Two copies of the policy** — tables in the flag, prose in `ONCALL.md`. Real
  drift risk, mitigated only by recording the split, and the reason the
  companion doc eventually wants source-of-truth sync.
- **Tier 2 may never get approved.** Worth knowing that Tier 1 is independently
  useful if so: the graded replay loop and the policy flag both stand alone, and
  `lessons.md` accrues value with no deployment at all.

**Confidence: high** that Tier 1 is fully doable without another person's
approval, and that it's the right first milestone — it's mostly the kit doing
what it already does, with two cheap seams attached. **Medium** on the events
step, which depends on SDK-key self-service I can't verify from here. The
observation that would most change the plan: if your org restricts both MCP and
API token creation, Tier 1 collapses to Tier 0 plus a paper design, and the
sensible move becomes finding the Owner conversation first after all.

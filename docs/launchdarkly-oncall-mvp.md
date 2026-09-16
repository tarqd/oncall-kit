<!-- SPDX-License-Identifier: Apache-2.0 -->

# MVP: dogfooding the on-call kit with two LaunchDarkly seams

**Status:** proposal · **Date:** 2026-09-16 · **Companion to:** `launchdarkly-integration.md`

Decisions taken:

- **Operator surface: Claude Tag as the kit ships today.** @Claude in the
  incident channel. No LaunchDarkly Slack app work.
- **Audience: internal dogfood first.** One LaunchDarkly team's real on-call.
  Schemas stay informal; the goal is evidence, not a product surface.

Those two choices cut most of the previously-proposed build. What's left is
small enough to state in one sentence:

> **Run the on-call kit as designed, on a real team, with LaunchDarkly
> supplying the policy and receiving the verdicts.**

Two seams, one of which is documentation. Everything else in this repo already
works.

---

## 1. What the decisions removed

| Previously proposed | Status now | Why |
|---|---|---|
| Block Kit action row in `chat-integrations` | **Cut** | Claude Tag can read reactions and threads directly. No Slack app change needed. |
| `/notify-channel` path for non-Vega agents | **Cut** | Only relevant once a second agent exists, which is post-MVP. |
| Vega as the runtime | **Cut from MVP** (kept as a side experiment, §6) | Claude Tag is the surface. Vega's repo-instructions behaviour stays interesting but isn't on the critical path. |
| Public contract with versioning commitments | **Cut** | Dogfood. Two JSON shapes in a doc, changed freely. |
| The read-only-vs-agent-mode gap as a blocker | **Downgraded to product feedback** | That gap blocks a customer-facing MVP, not a dogfood where the runtime is Claude Tag. Still worth filing (§7). |
| Proving "agent-agnostic" by running a second agent | **Deferred** | Agnosticism now lives in the schemas as a preserved seam, not a demonstrated property (§6). |

What this leaves is close to the cheapest three steps of the companion
document's §8 — which is the right place for an MVP to land.

---

## 2. Seam 1 — policy as a flag

One JSON flag, `oncall-policy`, replacing the prose tables in `ONCALL.md`.
Targetable by service and environment, changed with approval, audited.

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

**Read path:** the LD MCP server, scoped read-only (§5). Claude reads the flag
at the start of an investigation exactly as it reads `ONCALL.md` today.

**What this buys over a markdown table:** `posture` becomes a live kill switch
rather than a routine edit — the thing someone needs at 3am. Paging thresholds
get approvals and an audit trail. And the same JSON is machine-readable for the
deterministic alert rules the kit drafts.

**What it does not change:** policy is still human-set. A flag is a better home
for the numbers than prose, not a licence for the agent to pick them (rule 15).
Claude never writes this flag.

**Keep `ONCALL.md`.** The prose sections that aren't values — the incident
lifecycle rules, the out-of-band path, the read-only guarantee, the routing
tree's rationale — stay in the file. Only the tables move. `ONCALL.md` gains a
line saying which fields now live in the flag, so there is one obvious place to
look and no ambiguity about which copy wins. (Rule 7 says files are truth; this
is the one deliberate exception, and it needs to be written down rather than
inferred.)

## 3. Seam 2 — verdicts via git, shipped by CI

This is the part the decisions forced me to redesign, and the answer is better
than what I had.

With no Slack app build and no service, there's no obvious path for an operator
verdict to reach LaunchDarkly. The agent can't hold a write credential (rule 1,
and the whole of the companion doc's §10). So: **git is the queue, CI is the
shipper.**

1. **Claude reads the verdict from the channel.** A designated reaction set on
   its own diagnosis — the kit already contemplates exactly this mechanism
   (`ONCALL.md` names a `{{designated reaction}}` for acknowledgment). ✅
   helpful · ❌ wrong cause · 🔀 wrong routing · 🚫 harmful.
2. **Claude appends a row to `eval/shadow-log.md`.** This file already exists
   for this purpose, already has date / incident / grade / streak / grader
   columns, and is already an enumerated permitted output under rule 1. No new
   permission, no new file, no carve-out.
3. **CI ships new rows as LD metric events on merge.** A small job diffs the
   file, maps each new row to an event, posts it. A machine writes to
   LaunchDarkly; the agent never does.

```
Slack reaction → Claude appends shadow-log row → PR/commit → CI → LD metric events
```

Four events:

| Event | Source | Carries |
|---|---|---|
| `oncall.diagnosis.posted` | Claude, at post time | agent id, incident id, failure class, confidence, evidence-link count, latency from alert |
| `oncall.operator.verdict` | the designated reaction | helpful / wrong-cause / wrong-routing / harmful |
| `oncall.fix.verified` | triage step 9's bounded watch | window, verified true/false |
| `oncall.page.decision` | `paging-log.md` | decided, clause that tripped, signal values |

Three properties worth noting, because they're why this shape is right rather
than merely expedient:

- **It reuses the kit's existing safety envelope entirely.** Every write is to a
  file rule 1 already names. The agent's credential set doesn't grow.
- **The verdict is reviewable before it becomes data.** A human sees the
  shadow-log row in a diff. That's a property a direct API write would lose.
- **`oncall.fix.verified` and `harmful` are the two that matter.** They are
  outcome signals rather than process signals, and together they are the entire
  safety case for graduating off shadow mode. Everything else is texture.

Latency cost: verdicts arrive in LaunchDarkly at merge cadence, not in
real time. For a dogfood measuring weekly trends, that's irrelevant.

## 4. What the kit contributes that LaunchDarkly doesn't have

Worth being explicit, since Vega already investigates competently:

1. **A graded record of whether diagnoses were any good**, attributable per
   failure class and playbook version.
2. **A shadow period** — the agent posts to a review channel and is graded
   before anyone depends on it. `posture.shadow` plus the verdict events makes
   graduation an evidence question (≥10 consecutive helpful, zero harmful) and
   not a calendar one.
3. **Policy the team owns and reviews**, rather than thresholds the agent
   inferred.

## 5. The capability map is unusually cheap here

A LaunchDarkly team dogfooding this needs fewer connectors than an outside
team would, because **the LD MCP server covers three of the kit's capability
bindings at once** — it spans feature management, AgentControl configs, and
observability (logs, traces, errors, dashboards).

| `STACK.md` capability | Bound to | Via |
|---|---|---|
| `metrics` | LD Observability | LD MCP (read) |
| `logs` | LD Observability | LD MCP (read) |
| `flags` | LaunchDarkly | LD MCP (read) |
| `code` / `deploys` | GitHub | existing connector |
| `alert-channels`, `incidents` | Slack | Claude Tag |
| `pager` | team's pager | to confirm |

**Scope the token read-only.** Per the companion doc's §10: the hosted MCP
server has no `--scope`/`--tool` controls and its docs recommend a Writer or
Developer role; the local server has `--scope read` and `--tool` allowlisting.
Either way RBAC is the enforcing boundary — use a Reader base role or a custom
role granting only the view actions the playbooks need. LaunchDarkly's own
[security review](https://launchdarkly.atlassian.net/wiki/spaces/~7120202d087ecc5d974af4bdfb1fc4a3791aba/pages/4578181186)
found the shipped MCP server exposes 10 write tools including three unconfirmed
permanent deletes; none of those belong in an agent bound by rule 1.

## 6. Where agent-agnosticism actually lives now

Choosing Claude Tag makes the MVP's runtime Claude-specific. Being straight
about that: **this MVP does not demonstrate agent-agnosticism. It preserves the
seam for it.** The two schemas in §2 and §3 are runtime-neutral — a policy flag
any agent can read, and an event vocabulary any agent can emit — so a second
consumer can be added later without reworking either.

One near-free experiment that *would* test the seam, worth running alongside
rather than inside the MVP: **Vega already reads `CLAUDE.md`/`AGENTS.md` from
the root of a connected repository as "repository instructions."** Point Vega
at the same repo, with auto-remediation on an alert in read-only mode, and the
kit's rule set governs a second agent with no code written at all. If both
agents then triage the same incidents, the verdict events make them directly
comparable — which is the interesting result, and costs roughly an afternoon of
configuration.

Treat that as a side experiment with its own small write-up. Don't let it grow
into MVP scope.

## 7. Product feedback to file (not blockers here)

- **Vega's mode selector is binary and the useful middle is missing.**
  Read-only "never proposes or modifies code"; Agent mode opens PRs *and*
  creates dashboards, graphs, and experiments and recommends flag changes. The
  posture this kit wants — and that any review-gated agent wants — is
  **"read-only, plus may open a pull request."** A PR is the ideal agent write:
  inert until a human merges it. Worth raising independently; it would block a
  customer-facing version of this MVP.
- **Auto-remediation inherits the permissions of the member in "Last configured
  by" on the alert.** The docs warn that remediation breaks if that member's
  permissions change or their account is deactivated. For on-call that's worse
  than breakage: the agent's read-only guarantee is only as narrow as a
  person's role, and it changes silently when they move teams. A purpose-made
  service identity should be supported and documented as the default.

## 8. Prerequisites to confirm before starting

1. **Claude Tag availability.** The kit's surface requires @Claude as a member
   of Slack channels, which needs a Claude Team or Enterprise plan and an org
   Owner to attach connectors (`TAG-SETUP.md`). For an internal dogfood this is
   a procurement and admin question, not a technical one — and it's the one
   thing that can stop this before it starts. Confirm first.
2. **A host team with real incident history.** Phase 1 of setup mines 30–90
   days of resolved incidents. A team without that history gets thin playbooks.
3. **A review channel** for shadow mode, separate from the live channel.
4. **The `pager` binding**, or an explicit decision to run without one (the kit
   degrades to the `ONCALL.md` fallback path and records the gap).

## 9. Definition of done

A dogfood is done when it has produced evidence, not features:

- Setup ran end to end and **Phase 3 cleared its own gate** — ≥70% ✅+⚠️ on
  blind holdouts, zero 🚫 (`eval/replay.md`).
- **Four weeks of shadow operation** with the verdict events flowing, enough to
  read a trend in helpful-rate and verified-fix rate.
- A **decision**, either way, on whether to graduate off shadow — made from the
  shadow log's streak column rather than from enthusiasm.
- A short written answer to: did the kit's playbooks measurably beat what the
  team's on-call did unaided, and did anyone have to read `lessons.md`?

A dogfood that ends with "it seemed good" has failed, regardless of how the
agent performed.

## 10. Sequence

1. **Confirm the prerequisites in §8.** Especially Claude Tag. *Days, mostly
   waiting on people.*
2. **Run the kit's setup as written** on the host team — Phases 0–3, gates
   included. Nothing LaunchDarkly-specific yet; this is the kit doing its job.
   *~2 hours of human time spread over the phases, per `examples/run1-webshop/`.*
3. **Move `ONCALL.md`'s tables into `oncall-policy`** and point the triage
   skill's context load at it. *A day.*
4. **Add the shadow-log → CI → LD events job.** The only code in the MVP.
   *A day or two.*
5. **Shadow for four weeks**, grading daily per `eval/replay.md`.
6. **Side experiment, in parallel:** Vega on the same repo (§6).
7. **Then** revisit the companion doc's later steps — the fixture corpus as a
   dataset with a custom judge, then the AgentControl primitives.

Steps 1–2 involve no LaunchDarkly integration at all. That's deliberate: if the
kit doesn't produce useful diagnoses for this team, the seams are wasted work,
and step 2 is where you find out.

## 11. Risks

- **Claude Tag plan access is the single point of failure.** Everything else
  has a workaround; this doesn't. Resolve it in step 1.
- **Reaction-based verdict capture depends on operators reacting.** One emoji in
  a thread they're already reading is about as low-friction as it gets, but if
  the rate is low there's no signal. Watch it in week one; if it's poor, the
  fallback is Claude asking once in-thread rather than building a Slack app.
- **Two copies of the policy.** Tables in the flag, prose in `ONCALL.md`. Rule 7
  says files win; this MVP creates one deliberate exception. Mitigated by
  recording the split in `ONCALL.md` itself, but it is real drift risk and the
  reason the companion doc wants source-of-truth sync eventually.
- **A dogfood on a real on-call rotation carries real risk.** Shadow mode and
  `paging_enabled: false` exist for this. Don't shorten the shadow period
  because the diagnoses look good — the kit's own rule is that two weeks
  triggers a mandatory review, not an automatic graduation.
- **Thin incident history** produces thin playbooks and a vacuous paging
  dimension in the replay (`eval/replay.md` requires saying so explicitly when
  there are no page-severity holdouts).

**Confidence: high** that this is the right MVP shape given the two decisions —
it's mostly the kit running as designed, with the two cheapest seams from the
companion analysis. **Medium** on the four-week shadow window being long enough
to read a trend; that depends entirely on the host team's incident volume, and
a quiet month means extending rather than concluding.

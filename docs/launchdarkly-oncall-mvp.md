<!-- SPDX-License-Identifier: Apache-2.0 -->

# MVP: an agent-agnostic on-call contract, with the Slack app as the operator surface

**Status:** proposal · **Date:** 2026-09-16 · **Companion to:** `launchdarkly-integration.md`

Premise from the request: an MVP that is agent-agnostic but uses the Slack app
capability.

Two discoveries reshape this relative to the companion document, which framed
the choice as "keep Claude Tag" vs. "build an AI SDK service." Both are wrong
starting points, because **LaunchDarkly already ships both halves of the
plumbing.**

---

## 1. What already exists

### Vega is already an on-call agent

[Vega for auto-remediation](https://launchdarkly.com/docs/home/observability/vega)
is LD's Claude-powered agent, embedded in observability views and — this is the
part that matters — **configurable to fire automatically when an alert
threshold is breached**. On an alert it analyses the triggering query,
correlated telemetry, and recent flag and code changes, then summarises root
cause and can open a PR or a Jira issue. The docs describe it in the kit's own
terms: "like having an on-call engineer available at all times."

Four properties are directly load-bearing here:

| Vega property | Why it matters to this kit |
|---|---|
| **Read-only mode vs. Agent mode**, as a product toggle | The kit's central boundary — propose vs. act — already exists as an enforced setting rather than a prompt instruction. In read-only mode Vega "never proposes or modifies code." |
| **Reads `CLAUDE.md` or `AGENTS.md` from the root of connected repos** ("repository instructions") | The on-call kit *is* a set of markdown instruction files in a repo. This is an existing ingestion path for the kit's entire rule set, with no new machinery. |
| **Posts remediation results as a threaded reply to the original Slack alert message** | The operator surface is already Slack, already threaded to the alert. There is a "Using Vega in Slack" docs page. |
| Runs with the **"Last configured by" member's permissions** | Governance consequence — see §6. |

### The LD Slack app already does the operator loop

The [Slack app](https://launchdarkly.com/docs/integrations/slack) supports
channel subscriptions to flag/metric/project/environment updates, **approvals
handled natively in Slack** (comment / review / apply), flag toggling, link
unfurls for flags, approvals, segments, and metrics, and `/launchdarkly` slash
commands. Internally, `launchdarkly/chat-integrations` (`apps/slack`) carries
Block Kit renderers — including an existing `observabilityAlertMessage.ts` and
a `qualitativeFeedback.ts` — an `LDChannelEventKind` enum, and a generic
`/notify-channel` webhook whose stated purpose is letting external services
forward messages to Slack. The app sends ~1.8M messages/month, so this is
load-bearing infrastructure, not a prototype.

**Conclusion: the MVP should build almost nothing.** Neither a new agent nor a
new Slack app is the scarce part.

---

## 2. What "agent-agnostic" should mean here

Not "port the kit to LaunchDarkly," and not "Vega does on-call." Those both
bind the value to one runtime. The version worth building is the **interchange
contract** that lets *any* agent participate in the same on-call loop under the
same governance and emit the same feedback:

- **Vega**, for customers who have it (self-serve usage-based plans by default;
  select Enterprise otherwise).
- **Claude Code / Cursor / Copilot** via the LD MCP server, for customers who
  don't — the MCP docs already position these as complementary to Vega, running
  in different places for different workflows.
- **A customer's own Agent SDK app**, for teams who want their own harness.

This is also the LD-shaped answer: the platform explicitly does not execute
graphs or call model providers. It supplies configuration and collects
outcomes. An on-call contract is the same posture one layer up.

Concretely, three artifacts — and only the third needs code:

| # | Artifact | What it is | Build or reuse |
|---|---|---|---|
| 1 | **Policy payload** | What the agent should do: paging criteria, routing, severity tiers, playbook pointers. A documented JSON flag schema. | Reuse (flags) |
| 2 | **Verdict event schema** | What the agent concluded and whether it helped. Documented metric-event names and payloads. | Reuse (metrics) |
| 3 | **Operator verdict capture** | The Slack affordance that turns a human reaction into event #2. | **Build** (small) |

The contract is what makes agents interchangeable: two different agents reading
the same policy flag and emitting the same verdict events become directly
comparable on one dashboard. That comparison is the product insight — it is not
available from any agent in isolation.

---

## 3. The MVP, in three pieces

### Piece 1 — the repo pack (zero code)

Put the kit's rule set in the repository Vega has connected:

```
your-infra-repo/
├── CLAUDE.md          # the kit's standing rules — read by Vega as repository instructions
├── ONCALL.md          # policy: paging criteria, routing, severity, deploy windows
├── STACK.md           # capability → tool bindings
└── skills/triage/references/*.md   # the per-failure-class playbooks
```

Vega reads `CLAUDE.md`/`AGENTS.md` at the repo root for context. So the kit's
evidence discipline (every claim carries a link), its comms discipline (the
fresh-reader test), its confidence requirement, and its blame-staleness rule
become instructions governing a Vega investigation **today, with no code and no
new primitive.**

This is the highest-value, lowest-cost item in the entire integration analysis,
and it is worth validating on its own before anything else is built.

Honest limit: repository instructions are *advisory*. They shape the
investigation; they do not enforce it. The one rule that does get hard
enforcement is the most important one — read-only mode enforces "propose,
don't act" at the platform level, which is stronger than the kit's own
prompt-level enforcement.

### Piece 2 — two schemas (documentation, not code)

**Policy flag** — one JSON flag, `oncall-policy`, targetable by service and
environment. Consolidates what `ONCALL.md` holds as prose tables:

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
  "playbooks": { "test-failures": "skills/triage/references/test-failures.md" },
  "posture": { "mode": "read-only", "shadow": true, "paging_enabled": false }
}
```

Every agent reads this the same way. `posture` is the kill switch: `shadow`
routes output to a review channel, `paging_enabled: false` is the pre-graduation
default.

**Verdict events** — the agent emits these regardless of which agent it is:

| Event | When | Payload |
|---|---|---|
| `oncall.diagnosis.posted` | a diagnosis is published | agent id, incident id, failure class, confidence, evidence-link count, latency from alert |
| `oncall.operator.verdict` | a human reacts | `helpful` / `wrong-cause` / `wrong-routing` / `harmful`, operator id, incident id |
| `oncall.fix.verified` | the symptom returns to baseline in the expected window | incident id, window, verified true/false |
| `oncall.page.decision` | page or no-page | decided, which clause tripped, signal values (the rule 3a audit trail) |

`oncall.fix.verified` and the `harmful` verdict are the two that matter most:
they are outcome signals, not process signals, and together they are the whole
safety case for graduating an agent off shadow mode.

### Piece 3 — operator verdict capture in Slack (the only build)

Vega already posts its investigation as a threaded reply to the alert message.
The MVP adds a Block Kit action row to that message — "Helpful / Wrong cause /
Wrong routing / Harmful" — and maps the click to `oncall.operator.verdict`.

Reuse rather than greenfield: `chat-integrations` already has a
`qualitativeFeedback.ts` renderer and an `observabilityAlertMessage.ts`
renderer, plus a `/notify-channel` webhook for non-Vega agents to post through
the same path and get the same buttons. A new `LDChannelEventKind` covers the
event kind.

One button click closes the loop the kit has never been able to close: it
records not just what the agent concluded but whether the human found it
useful, attributable per agent, per failure class, per playbook version.

---

## 4. Why this is worth doing even though Vega exists

Vega investigates well. What neither Vega nor any other agent currently has:

1. **A graded record of whether its diagnoses were any good.** Vega posts a
   root cause; nothing captures whether the operator agreed. The verdict schema
   plus one Slack button is the missing half.
2. **Governing policy the customer owns and reviews.** Paging thresholds,
   routing, and severity are the customer's policy, not the agent's judgment.
   As a flag they get approvals, audit, and targeting. Today they live in
   whatever the agent inferred.
3. **A shadow period.** The kit's strongest safety idea: the agent posts to a
   review channel and is graded before anyone depends on it. `posture.shadow`
   makes that a flag flip, and the verdict events make graduation an evidence
   question (≥10 consecutive helpful, zero harmful) rather than a calendar one.
4. **Cross-agent comparability.** Once the contract exists, "Vega vs. Claude
   Code via MCP vs. the customer's own harness, same incidents, same rubric"
   becomes a measurable question.

---

## 5. The one real product gap: mode is binary

The kit's required posture does not exist in Vega's mode selector.

- **Read-only mode:** never proposes or modifies code. Too narrow — the kit
  explicitly wants the agent to open a revert PR and draft a flag ramp plan.
- **Agent mode:** opens PRs, *and* creates dashboards, graphs, and experiments,
  and recommends flag changes. Too broad — the kit permits exactly one write
  (a proposed PR) and forbids the rest.

The kit's posture is **"read-only, plus may open a pull request."** That sits
between the two modes and cannot currently be expressed. Worth raising as
product feedback independently of this MVP: "may propose, in the specific form
of a reviewable artifact" is a broadly useful third mode, not a quirk of this
kit. A PR is the ideal agent write — inert until a human merges it.

Until it exists, pin **read-only** and accept that PR-opening is out of MVP
scope. Do not reach for Agent mode to get PRs; it brings dashboard, experiment,
and flag-change capability with it.

---

## 6. Governance findings

**The permissions inheritance is a footgun.** Auto-remediation runs using the
permissions of the member shown in "Last configured by" on the alert. The docs
themselves warn that if that member's permissions change or their account is
deactivated, remediation breaks. For on-call this is worse than an outage: the
agent's read-only guarantee is only as narrow as that member's role, and it
silently changes when HR does. **Recommendation: configure on-call alerts as a
purpose-made service identity with a custom role scoped to exactly the read
actions the playbooks need** — never a person's account, and never an admin's.

**Pin the mode per alert, not per browser.** The mode selector is a personal
browser preference; alerts and Slack set their mode separately. Record the
intended posture in the `oncall-policy` flag so it is reviewable, and verify the
alert's own setting matches.

**Adaptive Triggers stay out of the MVP.** They are the auto-mitigation the kit
puts out of scope, and (per the companion doc, §5c) a fired trigger is a
forward-fix that already landed silently — a blame-staleness hazard for any
agent triaging afterwards. If any are enabled in the environment, the contract
should require the agent to check trigger firings in the onset window before
naming a live cause.

---

## 7. Scope

**In:** the repo pack; the `oncall-policy` flag schema; the four verdict
events; the Slack verdict buttons on Vega's alert-thread reply; a
`/notify-channel` path so a non-Vega agent posts through the same renderer; one
dashboard of verdict rates by agent and failure class.

**Out:** agent graphs; AgentControl configs, snippets, and skills (see the
companion doc — skills aren't shipped and source-of-truth sync doesn't cover
them yet); any library write by the agent; Adaptive Triggers; PR-opening;
paging. Paging is deliberately last — the kit gates it behind a shadow period
and a separate go/no-go, and the MVP has no evidence to clear that bar yet.

## 8. Sequence

1. **Repo pack against real alerts, read-only, one team.** Zero code. Does the
   kit's rule set measurably improve a Vega investigation? Grade ten
   investigations by hand. *Days.*
2. **Publish the two schemas** and move one team's `ONCALL.md` tables into the
   `oncall-policy` flag. *Days.*
3. **Slack verdict buttons** + the dashboard. The only real build. *A week or
   two, mostly in `chat-integrations`.*
4. **Second agent through the contract** — Claude Code via the MCP server,
   posting via `/notify-channel`. This is the step that proves
   "agent-agnostic" is real rather than aspirational. *A week.*
5. **Only then** consider shadow-mode graduation, PR-opening (pending the third
   mode), and the AgentControl primitives in the companion doc.

Step 1 is worth running before committing to any of the rest.

## 9. Open decision

**Which Slack surface?** I have assumed the existing LD Slack app, because the
request named "the slack app capability" and because approvals, unfurls,
threading, and the renderers are already there. The alternative — a standalone
Slack app, or Claude Tag as the kit ships today — buys independence from
`chat-integrations` ownership at the cost of rebuilding the approval and
notification paths. Flagged rather than assumed silently, since it is the one
choice that changes what gets built.

## 10. Risks

- **Repository instructions may not move the needle.** Step 1 exists to find
  out cheaply. If a Vega investigation is no better with the kit's rules than
  without, the premise is wrong and steps 2–5 should not be funded.
- **`chat-integrations` ownership.** The handoff doc notes the Slack app has
  been without strict ownership since Squad Waterbear disbanded, with various
  teams adding notification flows ad hoc. A new event kind needs an owner.
- **Verdict-button response rate.** If operators don't click, there is no
  feedback loop and the contract's value evaporates. Mitigation: exactly one
  click, in the thread they are already reading, with no modal.
- **Overlap with Vega's roadmap.** The ErrorGroup auto-remediation work and the
  Vega subagents feature both move in this direction. Check before building —
  the verdict-capture piece may already be planned.

**Confidence: medium-high** that the repo pack and verdict schema are the right
MVP core, since both reuse documented capabilities. **Medium** on the Slack
build estimate, which depends on `chat-integrations` internals I have only read
about secondhand. The observation that would most change this: if Vega already
captures operator feedback on its investigations somewhere, Piece 3 collapses
to a schema mapping and the MVP becomes almost entirely documentation.

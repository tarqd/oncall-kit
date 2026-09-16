<!-- SPDX-License-Identifier: Apache-2.0 -->

# Integrating the on-call kit with LaunchDarkly AgentControl

**Status:** investigation · **Date:** 2026-09-16 · **Scope:** which LaunchDarkly
primitives fit the on-call kit, which don't, and in what order to adopt them.

Three aims drove this, from the request:

1. Store things like lessons learned in AgentControl primitives.
2. Use agent graphs to improve the management and evolution of the kit.
3. Track feedback from agents and operators.

Short answer per aim: **(1) partly — the promoted layer, not the raw log;
(2) one slice of the kit fits agent graphs well, but it needs a runtime the
kit doesn't have today; (3) this is the cleanest win and the place to start,
alongside flags.**

---

## 1. The constraint that decides almost everything

The kit's runtime is Claude Code and Claude Tag (@Claude in a Slack channel).
Its "harness" is markdown in a git repo, read by a Claude session that reaches
tools over MCP. There is no LaunchDarkly AI SDK anywhere in the loop.

That matters because the AgentControl primitives split cleanly by what they
require:

| Primitive | Needs the LD AI SDK in the request path? | Usable from the kit as it exists today |
|---|---|---|
| Feature flags (incl. JSON variations), segments, contexts | No — REST/MCP read is enough | **Yes** |
| Flag change history, audit log, trigger firings | No — read-only evidence | **Yes** |
| Custom judges via `judge.evaluate(input, output)` | Yes, but only in a CI/eval script, not in production triage | **Yes** (eval path only) |
| Datasets + offline evaluations | LD generates the output itself | Partly — see §4 |
| Prompt snippets, tools, AgentControl configs + variations | Yes | No |
| Agent graphs | Yes (Python / Node AI SDK only) | No |
| Agent Skills | Yes, plus FDv2 enablement | No — and not shipped yet |
| Adaptive Triggers on the agent's own config | Yes (config must exist) | No |
| Agent Optimization | Yes | No (private beta) |

So there are two adoption paths, and they are not either/or — the second is a
superset that you can defer:

- **Path A — LD as control and telemetry plane, runtime unchanged.** The kit
  keeps running on Claude Code/Tag. LaunchDarkly supplies flags that gate kit
  behaviour, the audit trail that triage reads as evidence, a custom judge that
  grades replays in CI, and an events endpoint that collects operator feedback.
  No refactor. Everything in Path A works today.
- **Path B — the kit's reasoning steps become AgentControl configs.** Wrap the
  kit in a small Python or Node service (LD AI SDK + Claude Agent SDK) so
  triage, weather, and handoff become agent-mode configs, optionally wired into
  an agent graph, with judges attached and per-node metrics. This is where
  snippets, skills, graphs, experiments, and optimization become available —
  and it is a real rewrite of how the kit executes.

Evidence that Path B is a genuine lift and not a config change: the
[Synthia AgentControl pilot](https://docs.google.com/document/d/1sxFyPcu4Yf2FNbvnrK1_HeGLLqPD71Cb5_5kQtnVGjo)
declined agent graphs for exactly this reason — "many code refactorings would
be required" — with the internal read that graphs suit green-field
implementations better. The kit is not green-field; it is a markdown harness
with a deliberate design.

**Recommendation: do all of Path A now. Do Path B only for the triage fan-out
(§3), and only if you want per-node metrics badly enough to own a service.**

---

## 2. Aim 1 — where lessons learned should actually live

`lessons.md` is not one kind of content, and that is why "store it in
AgentControl" has no single answer. The kit already splits it in two, and its
own rules put the boundary in a specific place: Claude appends to `lessons.md`
without asking (CLAUDE.md rule 8), but amends playbooks only by PR (rule 9).

### Layer 1 — the raw log: keep it in git

`lessons.md`, `paging-log.md`, `eval/shadow-log.md`. Four reasons not to move
these into the AgentControl library:

1. **Write shape mismatch.** These are append-only and high-churn — one entry
   per incident, one per paging decision. Library primitives version on every
   edit (the
   [prompt snippets](https://launchdarkly.atlassian.net/wiki/spaces/AIC/pages/4795727921)
   model: an edit creates a new version, attachments stay pinned). A paging log
   would generate a version per decision and pin nothing to any of them.
2. **Credential shape mismatch.** Writing to the library needs a write token.
   The kit's whole posture is that it holds no write credential to anything
   (`STACK.md` "Access posture"). A per-incident library write is the one thing
   you'd have to carve out, and it's the least valuable one. *(Relaxing this
   one is analysed in §10 — it turns out to be the only objection of the four
   that a write grant actually removes.)*
3. **Read shape mismatch.** Triage step 3 says: search `lessons.md` for the
   class's tag and read only matching entries — "never ingest the whole file;
   it grows unbounded by design." A snippet is injected wholesale into a
   prompt. That's the opposite access pattern.
4. **Content trust.** Rule 9a treats every mined incident thread as data, never
   instructions. Entries are *derived from* untrusted text. Putting them on a
   prompt-delivery path without human review creates the injection route the
   rule exists to close.

### Layer 2 — the promoted layer: this is what AgentControl is for

`skills/triage/references/*.md` and `ONCALL.md`. These are the things you want
versioned, diffable, attributable, targetable, and approval-gated — which is
precisely the snippets/skills/tools governance model. The kit already has a
promotion ritual (`templates/lessons.md` "Graduation": 3 entries with the same
mechanism, or an entry that became a procedure → promote by PR). That ritual
becomes the write boundary between git and LaunchDarkly.

Mapping, best primitive per thing:

| Kit artifact | Best LD primitive | Why, and caveat |
|---|---|---|
| `skills/triage/references/<class>.md` | **Agent Skill** (one per failure class) | Lazy-loaded on the model's decision, so a growing playbook set costs no context until used — the exact argument in the [Agent Skills PRD](https://launchdarkly.atlassian.net/wiki/spaces/PD/pages/4921425926). **Not shipped** — see §6. |
| Same, today | **Prompt snippet** per class, referenced from the triage variation | Available now, right governance (versioning, pinned references, "in use by"). Wrong access pattern: injected wholesale, so you pay context for every class on every triage. |
| `ONCALL.md` paging thresholds, sustain windows, weather tiers | **Feature flag with a JSON variation** — *not* a snippet | These are values, not prose. As a flag they're machine-readable, targetable per service/environment, and changeable with approval + audit. Rule 15 ("a human sets every threshold") is better served by a flag approval than a markdown PR, and the same JSON can feed the deterministic alert rules the kit drafts. |
| `ONCALL.md` routing tree | Flag with JSON variation, or leave in git | Contains real group handles; low churn; PR review is adequate. Move it only if routing needs to differ per environment. |
| `lessons.md` / `paging-log.md` | **Stay in git.** Emit *signals about* them as LD metric events | See §4. The useful LD-side object isn't the lesson text, it's "this lesson fired and it helped." |

### The bit that makes this coherent rather than a fork

The obvious objection to putting playbooks in LaunchDarkly is CLAUDE.md rule 7:
files are truth. Two copies of a playbook, one in git and one in the LD
library, drifting silently, is strictly worse than one copy.

That is the exact problem
[Source-of-truth sync](https://launchdarkly.atlassian.net/wiki/spaces/PD/pages/4970840200)
solves, and it resolves it in the kit's favour: **code-canonical by default**.
Files under `.launchdarkly/` in the repo are the source of truth; a GitHub
Action comments on each PR with what merging would publish, and publishes on
merge; UI edits that diverge raise a drift banner in the LD UI and in CI. That
is rule 9 ("amend playbooks by PR only") implemented as platform mechanics
rather than as an instruction Claude has to obey.

Status, and it's the main dependency in this whole document: the
[tech spec](https://launchdarkly.atlassian.net/wiki/spaces/PD/pages/5206016355)
is BUILDING as of 2026-09-11. Phase 1a covers prompts, tools, and judges;
**snippets are Phase 2 and skills are Phase 3**
([epic](https://launchdarkly.atlassian.net/browse/AIC-3165)). Neither of the
two primitives the kit most wants is in the first phase. Until sync ships for
them, keep git canonical and push promoted files into LD with a small script
in the same PR — one direction only, never read back.

There is also a genuine upside to LD-delivered policy that is specific to this
kit: rule 7a exists because the code host being down *is often the incident*.
The LD SDK serves last-known-good from its own store over a delivery network in
a different failure domain from GitHub. Moving policy delivery to LD makes
degraded mode strictly better. Two counter-notes: it adds LaunchDarkly to the
on-call critical path (so the `ONCALL.md` fallback still has to exist), and
skills delivery rides FDv2, which the **Relay Proxy does not serve** — a
relay-only deployment cannot receive skills.

---

## 3. Aim 2 — agent graphs for managing and evolving the kit

The honest summary: the kit's *shape* fits agent graphs unusually well, one
part of it fits extremely well, and the runtime gap is the whole cost.

An agent graph is nodes = agent-mode configs, edges = handoff data, topology
held in LaunchDarkly, execution and traversal owned by your application
([docs](https://launchdarkly.com/docs/home/agentcontrol/agent-graphs)). LD does
not execute the graph or call the provider.

### The slice that fits: the triage fan-out

`skills/triage/SKILL.md` step 4a already specifies a multi-agent topology with
a formal contract:

- **Root:** the triage orchestrator (classify, correlate, synthesise).
- **Nodes:** one investigator per bound source of truth — `metrics`, `logs`,
  `code`/`deploys`, `pager`, `alert-channels`.
- **Edge handoff data, outbound:** exactly four fields — the symptom sentence,
  the onset window, the investigator's binding line from `STACK.md`, and the
  reference's first-check queries for that source. Deliberately nothing else,
  so a poisoned thread cannot steer a subagent (rule 9a).
- **Edge handoff data, inbound:** a fixed four-part return —
  CHECKED / FOUND / NOT FOUND / CANNOT ACCESS.

That is a routing contract already written down as a contract. Externalising it
buys three things the kit currently cannot get:

1. **Per-node metrics** — invocations, latency, errors, tool calls per
   investigator, overlaid on the graph. This directly measures the thing the
   kit cares about and cannot see today: which binding is slow, which one keeps
   landing in CANNOT ACCESS (a 403 that currently surfaces once in a diagnosis
   and is never counted).
2. **Per-node judges** — score the investigator contract itself: did FOUND
   contain observations rather than root causes? Did absence get claimed on an
   incomplete window (the rule-5a failure)? These are per-node quality signals,
   not per-diagnosis ones.
3. **The edge contract becomes a governed object** rather than a paragraph of
   markdown that a well-meaning edit can widen. Given that the four-field limit
   is a *security* boundary, that is worth something.

### The slices that don't fit

- **Setup phases 0→4.** Superficially a graph (five nodes, four gates), but
  every edge is a human sign-off that can take a day, and phase output is files
  in a repo. This is a workflow with human gates, not agent handoffs. Modelling
  it as a graph adds nothing; LD does not manage graph state or traversal.
- **The weather cycle.** Single-node, event-gated, deliberately cheap. Its
  design point is exiting early most cycles. A graph adds overhead to the thing
  optimised for having no overhead.
- **Triage below page severity.** Step 4a fans out for page severity only —
  "fan-out multiplies token cost — worth it for a page, never for a morning-log
  item." Most invocations are the sequential path, and a graph doesn't help
  there.

### Verdict

Agent graphs are the right model for the page-severity fan-out and the wrong
model for everything else in the kit. That's one graph, five or six nodes,
reached only on page-severity incidents — which is also a low-traffic graph,
so the per-node metrics accumulate slowly. Worth doing if you're building the
Path B service anyway. Not worth building the service *for*.

---

## 4. Aim 3 — feedback from agents and operators

This is the aim with the best fit, the least new machinery, and the most
immediate value. It also fills a gap the kit itself names: `eval/replay.md`
admits its within-incident quality measure "starts out as vibes."

### Operator feedback (humans in the channel)

The kit's per-incident signals are already defined; they just have nowhere
numeric to go. Map each to an LD metric event:

| Operator signal, already in the kit | LD metric | Kind |
|---|---|---|
| Diagnosis posted → human acted on it with no recorded verification | **rubber-stamp rate** (`eval/replay.md` names this as the only early warning that the team stopped cross-examining) | count |
| Human pushback in thread ("could it be X instead?") | **cross-examination rate** — the healthy inverse of the above | count |
| A designated reaction on the diagnosis (👍/👎) | **operator satisfaction** — the AI SDK's end-user feedback tracker | conversion |
| Metrics returned to baseline inside the reference's expected-resolution window (triage step 9) | **verified-fix rate** — the strongest outcome signal the kit produces | conversion |
| Page fired / page suppressed, plus the later human read on whether it was necessary | **page precision** and **false-page rate**, from `paging-log.md` | count |
| Time from alert to first diagnosis | **time-to-first-diagnosis** | numeric |

Two of these deserve emphasis because they are *outcome* metrics, not
process metrics: verified-fix rate and false-page rate. They are what the kit
is for. Wire them first.

### Agent feedback (the kit grading itself)

`eval/replay.md` is a graded eval harness that predates any LD involvement:
blinded holdouts, a fresh-context grader that never wrote the diagnosis, a
four-level scale (✅ / ⚠️ / ❌ / 🚫), and hard bars (≥70% ✅+⚠️ and **zero 🚫**
to pass the Phase 3 gate). Plus `test-fixtures/` — 48 fictional incidents with
`ground-truth.json` and an `ANSWER-KEY.md`, already used as the regression
test.

That maps onto LD's eval primitives almost field for field:

- **Dataset row** ← a holdout: `input` = the blinded triggering alert;
  `expected_output` = the correct class, cause, and routing from the answer key;
  `context` = the playbook and `STACK.md` state at the time; `variables` = the
  channel and tool names. CSV/JSONL, under 32 MB and 10,000 rows — a 48-row
  fixture is nowhere near the limits.
- **Custom judge** ← the four-level rubric. The grade definitions in
  `eval/replay.md` are already a judge prompt, including the two hygiene checks
  (fresh-reader test, links resolve). 🚫 becomes a hard zero-tolerance
  criterion.
- **Gate** ← the score thresholds, as a CI status check.

**One limitation to design around, and it's important.** Offline evaluations
have LaunchDarkly generate the output for each dataset row. A triage diagnosis
cannot be generated that way — producing one requires live tool calls to
metrics, logs, and the code host over MCP. So split the loop:

- **Generation stays in your harness** (Claude Code or the Agent SDK, with the
  real MCP connections), exactly as `eval/replay.md` already describes.
- **Grading goes to a custom LD judge**, invoked programmatically —
  `judge.evaluate(input, output)` returns a 0.0–1.0 score with reasoning, and
  recording it as an evaluation metric gives you the trend line the kit
  currently keeps in a markdown table.

Offline evals and datasets remain useful for the parts that *don't* need tools:
grading the classifier alone (symptom → failure class), and the paging decision
given a fixed signal set. Those are also the two places where a wrong answer is
most expensive, so it's a reasonable subset.

Note that the judge replaces the *scoring*, not the *authority*.
`eval/replay.md` requires a human to confirm final grades, and the structural
requirement — the session that wrote a diagnosis never grades it — is satisfied
by a judge being a separate config. Keep the human confirmation.

### Privacy

Send scores, IDs, links, and durations. Do not send log bodies, metric values,
or thread text. LD's position is that online evaluations run on your provider
credentials and it does not store or share prompts, responses, or evaluation
data outside your project — but the kit's inputs are production incident
records, so the conservative posture is to keep the payloads referential.

---

## 5. Feature flags — the part that works today

Flags are already a first-class concept in the kit: `STACK.md` binds a `flags`
capability (read-only, "flag changes are timeline events for triage"), and
triage step 4 checks flag change history when establishing the timeline. Two
additions.

### 5a. Flags that gate the kit's own behaviour

Each of these is a switch the kit currently implements as a routine edit, a
markdown line, or a paste. A flag gives it an owner, an approval, an audit
entry, and an instant off.

| Flag key | Type | Replaces | Why it's better as a flag |
|---|---|---|---|
| `oncall-paging-enabled` | boolean | the go/no-go decision after shadow | The one switch someone needs at 3am when the agent is paging wrongly. Today that's editing a routine. |
| `oncall-shadow-mode` | boolean | "post to the review channel, not the live one" | Shadow graduation becomes a flag flip with an audit record, per-channel targetable. |
| `oncall-alert-editor-enabled` | boolean | the opt-in recorded as prose in `ONCALL.md` | This is the kit's single fenced write exception (rule 1a). A flag with approvals is a much better home for it than a sentence in a policy file. |
| `oncall-weather-enabled` | boolean | routine 6 installed / not | Same shape as shadow mode. |
| `oncall-paging-criteria` | JSON | the `ONCALL.md` paging table | Machine-readable, per-service targetable, approval-gated; feeds the drafted alert rules too. |
| `oncall-weather-tiers` | JSON | the `ONCALL.md` mood tier table | Set in Interview, read by the weather skill, never invented by it — a flag makes "never invented" enforceable. |
| `oncall-triage-playbook-version` | string/JSON | which reference-file version is live | Lets a new playbook go to one failure class or one environment first. |

**Context kinds** to design up front, because retrofitting them is painful:
`service`, `failure-class`, `channel`, `severity`, and `incident`. Evaluate a
multi-context containing whichever are known at the decision point.

One statistical caution: `incident` is the natural randomisation unit for an
experiment, and a team generates a handful of incidents a week. An experiment
on playbook variants will be badly underpowered for months. Use production
metrics for *monitoring*, and the replay dataset for *comparison*. Don't ship
a decision off a two-week production A/B of 11 incidents.

### 5b. LaunchDarkly as triage evidence

This needs no integration work beyond a read-only token — the binding already
exists — but it should be written into the triage references explicitly:

- **Flag change history** around the onset window. Already implied by step 4;
  worth naming LD's audit log as the queryable source.
- **Guarded / progressive rollout state.** The kit *proposes* canary ramp plans
  ("the flag, the percentage steps, the hold time at each step, and the single
  metric that aborts the ramp"). That is a guarded rollout, described from
  first principles. Proposing an actual guarded-rollout configuration for a
  human to click is a better deliverable than prose, and stays read-only:
  Claude drafts, a human executes, Claude watches the metric land at each step.
- **Adaptive Trigger firings.** See below — these are timeline events with a
  sharp edge.

### 5c. Adaptive Triggers — read this before enabling any

[Adaptive Triggers](https://launchdarkly.atlassian.net/wiki/spaces/PD/pages/5253005358)
(GA for Guardian 2026-08-17; available on AgentControl configs for all
AgentControl customers) switch a flag or config's default variation
automatically when a monitored metric crosses a threshold. No human in the
loop. Three separate findings:

1. **On monitored product flags, this is auto-mitigation — and it does not
   violate the kit's rules.** README puts auto-mitigation out of scope and says
   that if you want it, "build it as its own system, with its own review and
   its own named owner — outside this kit." A platform trigger with RBAC, audit
   logging, and a product owner is exactly that. The agent isn't acting; the
   platform is. Adaptive Triggers is a legitimate way to satisfy that clause.
2. **But a fired trigger is a rule-5a landmine.** Rule 5a exists because blame
   verdicts persist in channels after a forward-fix lands. A trigger that fired
   at 02:14 and switched to a safe variation *is* a forward-fix that already
   landed, silently, with no human to remember it. Any triage that names a
   deploy or PR as the live cause must first check LD for trigger firings in
   the window. This belongs in the triage references as a first check, not as
   a footnote.
3. **On the agent's own config (Path B), triggers are useful with a catch.**
   Failing the on-call agent over to a different model or provider when its
   primary degrades is a good use. But there is **no auto-revert yet** — the
   switch is one-way until a human restores the default. An on-call agent
   quietly running its backup variation for three days is precisely what rule
   19 (judgment calls leave flags) and rule 7a (degraded mode is declared,
   never improvised) forbid. If you enable this, the kit must surface "running
   on variation X since trigger fired at T" in every sitrep and handoff.

---

## 6. Availability and dependencies, as of 2026-09-16

Verify each of these before planning around it; several moved in the last month.

| Primitive | Status | Consequence for this plan |
|---|---|---|
| Feature flags, segments, contexts, audit log | GA | Path A works today. |
| MCP server (hosted, OAuth) | GA — covers flags, AgentControl configs, observability | **Can create/update/delete**, including three unconfirmed permanent deletes. Scope the kit's identity to read-only via RBAC, or use a reader REST token instead. The hosted server has no tool-allowlist or read-only flag; the local server does (`--scope read`, `--tool`). See §10. |
| Prompt snippets | GA | The available stand-in for playbook references under Path B. |
| Judges (built-in + custom), online/offline evals, datasets, playgrounds | GA; online evals need project-level enablement and a recent AI SDK | The eval path is buildable now. |
| Agent graphs | GA — Python and Node AI SDK only | Fine; a Path B service would be Python or Node anyway. |
| Adaptive Triggers | GA (Guardian for flags; AgentControl configs for all AC customers). No auto-revert. | See §5c. |
| **Agent Skills** | **Not shipped.** Per the [SDK docs handoff](https://launchdarkly.atlassian.net/wiki/spaces/PD/pages/5311266866), as of 2026-09-08 all SDK work was in open PRs and in no released package. Behind `enableAicAgentSkills`. Requires FDv2 (Beta opt-in, off by default). Relay Proxy unsupported. Beta is a **single SKILL.md ≤ 50 KB**, not multi-file bundles. | The best-fit primitive for the kit's playbooks is the least available one. And the Beta shape doesn't hold a bundle: `skills/triage/` is a `SKILL.md` plus four `references/*.md`, which would have to become five unrelated library entries. Check whether this shipped in the last week before designing around it. |
| **Source-of-truth sync** | BUILDING (tech spec, 2026-09-11). Phase 1a = prompts, tools, judges. **Snippets are Phase 2, skills are Phase 3.** | The mechanism that makes "files are truth" and "LD delivers" compatible does not yet cover either primitive the kit would use. Keep git canonical; push one-way. |
| Agent Optimization | Private beta | Out of scope. Also: auto-tuning an on-call agent's own behaviour against a cost/quality metric needs its own think, given rule 15 (a human sets policy numbers). |

---

## 7. Safety boundaries the integration must respect

These are the places where a natural-looking integration would quietly break
the kit's contract.

**Two roles for LaunchDarkly, never one credential.** LD shows up twice: as a
**monitored system** (the product's flags — the existing `flags` binding, where
Claude reads state, history, and trigger firings as evidence and never writes)
and as the **agent's own control plane** (the kit's configs, snippets, skills,
judges, metrics). Use **separate LD projects**, and separate tokens scoped by
RBAC, so a mis-scoped token cannot reach product flags. Recommended posture for
the agent's control-plane token: **read configs, snippets, skills, and flags;
write events only.** Metric and feedback events are append-only telemetry, not
state — they don't engage rule 1. Everything else about the agent's own config
is policy, and policy is human (rules 9 and 15).

**No vendor coupling.** README calls a skill that names a specific tool a bug.
So this integration is expressed as capability bindings in `STACK.md` — add an
`agent-control` row (and an `agent-telemetry` row if separate) — and the skills
keep referring to capabilities only. `STACK.md` stays the one file that names
vendors.

**Closed gates stay closed (rule 16).** Notification gates decide by a
mechanical event list and nothing else. A flag may *disable* a routine or
*supply a threshold value*; a flag must never become a second judgment inserted
between a fired gate and the send. `oncall-weather-tiers` sets the boundaries;
it doesn't decide whether to post.

**No automatic graduation.** `eval/replay.md` is explicit: two weeks is the
maximum before a *mandatory review*, not a promotion trigger, and the exit bar
is ≥10 consecutive helpful diagnoses with zero 🚫. If you drive shadow-mode
graduation with a progressive rollout, configure it to require approval — do
not let a rollout schedule graduate the agent on a calendar.

**Don't let promotion bypass review.** The graduation path (`lessons.md` → a
reference file, 3 entries with the same mechanism) must stay a PR. Entries are
derived from incident text, which rule 9a treats as data; the PR is the
human-review step that keeps mined text off the prompt-delivery path.

---

## 8. Suggested order of work

Each step stands alone and is useful if you stop there.

1. **Read-only LD binding + trigger firings in triage.** Add `agent-control` to
   `STACK.md`; scope a reader token; write "check LD flag history and Adaptive
   Trigger firings in the onset window" into the triage references as a first
   check. Closes the rule-5a gap in §5c.2. *Days. No new runtime.*
2. **Operator-feedback metrics.** Wire verified-fix rate, false-page rate,
   rubber-stamp rate, and time-to-first-diagnosis as LD metric events from the
   existing Slack surface. Gives the weekly handoff real numbers instead of a
   markdown tally. *Days to a week.*
3. **The fixture corpus as a dataset + the replay rubric as a custom judge.**
   48 incidents and an answer key already exist. Generation stays local; grading
   goes to the judge; the Phase 3 bars become a CI status check. This is the
   highest-leverage step for "evolution of the kit" and needs no runtime
   change in production. *One to two weeks.*
4. **The kit's own switches as flags.** `oncall-paging-enabled`,
   `oncall-shadow-mode`, `oncall-alert-editor-enabled`, then the JSON policy
   flags. Design the context kinds here (§5a). *One week.*
5. **Guarded-rollout proposals instead of prose ramp plans.** Triage already
   drafts the ramp; emit it as a guarded-rollout configuration a human clicks.
   Still read-only. *Small, and a visible quality jump in the deliverable.*
6. **Then decide on Path B.** Only after 1–5 are running, and only if per-node
   fan-out metrics and attached judges are worth owning a service for. If yes:
   triage fan-out as an agent graph, playbook references as snippets (skills
   when they ship and can hold a bundle), source-of-truth sync when it reaches
   Phase 2/3.

## 9. Open questions

1. **Did Agent Skills ship?** The 2026-09-08 handoff says not yet. If it has,
   does Beta hold a multi-file bundle, or still one `SKILL.md` ≤ 50 KB? The
   whole Layer-2 recommendation depends on the answer.
2. **When do snippets and skills get source-of-truth sync?** They're Phases 2
   and 3. Until then, two copies of every playbook and a hand-rolled one-way
   push.
3. **Code-based judges.** The kit's grader is a prompt today, but its hygiene
   checks ("links resolve") are mechanical and want code. The Synthia pilot
   asked for code-based judges too — worth confirming whether that's roadmapped
   before the rubric gets contorted into pure prose.
4. **Does the `incident` context kind ever get enough volume to experiment on?**
   If not, say so in the design and lean on the replay dataset permanently
   rather than pretending a production A/B is coming.
5. **Which LD project owns the agent?** A dedicated `oncall-agent` project is
   the recommendation in §7; confirm that doesn't collide with how the org
   scopes projects and RBAC today.

---

**Confidence: medium-high** on the primitive fit and the ordering; **medium**
on availability, since Agent Skills and source-of-truth sync are both in flight
and the freshest internal source here is eight days old. The observation that
would most change the plan: if Agent Skills shipped with multi-file bundle
support, steps 3–6 reorder — the playbook layer moves to LaunchDarkly first and
the agent-graph question gets much more attractive, because the harness would
already live there.

---

## 10. Addendum: what if the agent could write lessons via the LD MCP?

Asked as a follow-up: suppose we relax the credential constraint and give the
kit the LaunchDarkly MCP server so it can write lessons into a primitive.

**It removes objection 2 of the four in §2 and leaves the other three standing
— and it makes objection 4 materially worse.** The underlying reason is a
design conflict, not a missing feature:

> Lessons are valuable *because* they are cheap and unreviewed (rule 8: log
> without asking). Library primitives are valuable *because* they are governed
> — versioned, approved, audited, delivered. Those two properties cannot both
> hold on one object. Any write path you build either governs the lesson (and
> loses rule 8) or ungoverns the primitive (and loses the reason to use it).

The rest of this section is what that means concretely.

### 10a. The instrument matters more than the permission

Two corrections to what I hedged on earlier, both load-bearing:

**A narrow write grant is expressible — for configs.** `aiconfig` is a real
resource in the role system, written `proj/*:env/*:aiconfig/*`, and **the
resource identifier is the config key**
([resources](https://launchdarkly.com/docs/home/account/roles/role-resources),
[actions](https://launchdarkly.com/docs/home/account/roles/role-actions)). So
least privilege is a real policy, not an aspiration:

```json
[
  { "effect": "allow",
    "actions": ["updateAIConfigVariation"],
    "resources": ["proj/oncall-agent:env/production:aiconfig/oncall-lessons"] },
  { "effect": "allow", "actions": ["viewProject"],
    "resources": ["proj/oncall-agent"] }
]
```

One action, one config key, one project. Note the asymmetry that follows:
**there is no per-skill RBAC yet** — the Agent Skills PRD lists it at GA, with
Beta using project-scoped library reader/writer roles. So if lessons live in a
*config*, the grant can be surgical today. If they live in a *skill*, the
narrowest available grant is "write every skill in the project."

**The hosted MCP server is the wrong instrument regardless.** Its own docs
recommend a Writer base role or Developer preset — "permission to create, read,
update, and delete flags and AgentControl configs" — and it exposes no
`--scope`/`--tool` controls. Those exist only on the **local** server, which is
otherwise positioned for federal/EU use. LaunchDarkly's own
[security review of AI prompt distribution](https://launchdarkly.atlassian.net/wiki/spaces/~7120202d087ecc5d974af4bdfb1fc4a3791aba/pages/4578181186)
(March 2026) says this plainly about the MCP server at v0.6.0:

- **19 tools — 9 read-only, 10 write, including 3 permanent deletes**
  (`delete-feature-flag`, `delete-ai-config`, `delete-ai-config-variation`).
- **No confirmation gate on any destructive operation.** The internal catfood
  MCP requires `confirm: true`; the shipped server does not.
- The docs' own recommended custom role grants `actions: ["*"]` on all flags and
  all configs in all environments — flagged in review as "the widest permission
  surface possible."
- **No structured audit trail in the MCP server** — debug console logging only,
  which the review notes "makes it harder to trace agent-initiated actions back
  to the prompt/conversation that triggered them." (The LD API does audit-log
  mutative operations server-side.)
- A P0 recommendation to make `--scope read` the documented default.

That last bullet is the one that actually bites this kit. Rule 3a requires every
decision to be reconstructible from a log — "was this page absolutely necessary
on a Friday night?" must be answerable. An agent-initiated library write that
cannot be traced back to the conversation that caused it is the same gap in a
different place.

So if you do this, the shape is: **local MCP server, pinned version (not
`npx -y`), `--tool` allowlisted to the one write tool, plus the scoped custom
role above.** Not the hosted server with an OAuth grant.

And note what that costs to gain what: ten write tools and three unconfirmed
permanent deletes enter the agent's tool surface so that one append can happen.
RBAC is the real boundary and can hold — but the kit's guarantee stops being
"it holds no write credential" and becomes "its credential is scoped
correctly." Those are different promises, and only the first one is checkable
by reading `STACK.md`.

### 10b. Each candidate primitive fails on a different axis

Even with a perfect credential, there is no good target object:

| Target | Write mechanics | Why it fails |
|---|---|---|
| **Prompt snippet** | Edit creates a new version; referencing variations stay **pinned** and there is no automatic propagation | Every append needs a *second* write to re-pin every referencing variation. That is a release per lesson, not a log line. Skip the re-pin and the agent never reads its own lessons. |
| **AgentControl config variation** | `updateAIConfigVariation` edits in place; the SDK serves current, so no re-pin | The variation body *is* the agent's instructions. This puts text mined from incident threads directly into the prompt — the worst available location. Also: unbounded growth is paid as context on every triage. |
| **Agent Skill** | Lazy-loaded, so the read shape is finally right | Not shipped; needs FDv2; Beta is one `SKILL.md` **≤ 50 KB** (a hard ceiling an append-only log reaches in roughly 80 entries, after which writes fail or the agent must silently prune); no per-skill RBAC; and skill content is delivered to **every server-side FDv2 connection in the environment, with no per-connection subsetting in Beta**. |

That last clause is objection 4, amplified. Today a poisoned lesson sits in
`lessons.md`, where triage reads it *as data* and a human sees it at promotion
time. As a skill it becomes a payload the platform delivers into the context of
every agent in that environment. The kit's own threat model already names this
surface — anyone who can post in a watched alert channel can write text the
agent might mine (rule 9a) — and LD's security review independently describes
the matching chain: agent reads poisoned content via MCP, then acts on it
through a write tool with no confirmation. Write access is what closes the
circuit between those two halves.

### 10c. What a write grant *is* worth, in priority order

Reframed as "what should be writable," the answer isn't nothing:

1. **Metric and feedback events — do this, it needs no MCP write at all.**
   "Lesson #merge-queue fired on INC-2041 and the diagnosis was verified" is an
   append-only telemetry event, not library state. It doesn't engage rule 1, it
   needs no library permission, and it is the part of the lessons loop that is
   genuinely missing today: the kit records *what* was learned but never
   *whether it helped*. This is where the value the question is reaching for
   actually lives.
2. **Wait for source-of-truth sync Phase 1b, which ships exactly this.** Its
   scope includes **UI edits opening a PR** — a library-side write that
   materialises as a reviewed change instead of a live one. That is
   "agent-writable with review" as a platform primitive rather than a bespoke
   carve-out, and it satisfies rule 9 by construction. Phase 1b is GA scope for
   prompts/tools/judges; snippets are Phase 2 and skills Phase 3.
3. **If you want it before then, copy the rule 1a shape — and price it
   honestly.** The kit already has one sanctioned write exception: additive
   only, never modify or delete, explicit per-item approval in the channel,
   every write logged with its approval link. Apply that to lessons and you get
   a defensible design — and immediately discover the cost, which is that
   per-lesson channel approval is incompatible with rule 8. You would be
   spending a human approval per incident to move a log entry from a file into
   a library that reads it less well. That trade is the whole argument for the
   Layer 1 / Layer 2 split in §2: the split is not a workaround for missing
   write access, it is the answer, and write access does not change it.
4. **One genuinely good target, if you want a library write today:** the
   promoted layer, written by CI rather than by the agent. The PR merges, the
   Action publishes. A machine writes, with review upstream of it, and the
   agent stays read-only. This also captures the degraded-mode benefit (policy
   and playbooks readable when the code host is the incident) without opening
   any agent write path at all.

**Confidence: high** on 10a and 10b — the RBAC granularity, the MCP tool
surface, the pinning semantics, and the FDv2 broadcast behaviour are all
documented rather than inferred. **Medium** on the Phase 1b recommendation,
since that scope could move. The observation that would most change 10c: if
Phase 1b's "UI edits open a PR" turns out to cover API-originated writes too
and not just the UI, it becomes the answer outright and items 3 and 4 collapse
into it.

---

A side note worth raising internally: the kit is an unusually clean design
partner for AgentControl. It's a real agent with a written safety contract, a
graded eval harness, a promotion ritual, and a fixture corpus — and the gaps it
runs into (bundle-shaped skills, source-of-truth sync for snippets and skills,
code-based judges, a governed handoff contract between orchestrator and
sub-agents) are the ones already on the roadmap. Running this integration end
to end would validate several of them against a non-hypothetical agent.

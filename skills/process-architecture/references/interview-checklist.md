# Interview Checklist

This is the master checklist the skill must complete before an `architecture-spec.yaml` is handoff-ready. Every item below must have a concrete answer in the spec. If any item is missing, the handoff is blocked.

Use this file two ways:

1. **During the interview**, check items off as you collect answers. When a section is complete, summarize the answers back to the user and ask for confirmation.
2. **At the end**, run through the whole checklist as the handoff gate. Any unticked box = refuse handoff and return to the corresponding section.

---

## Section 1 — Goal & Output

- [ ] `goal.output` is a single concrete sentence naming a specific deliverable (file, decision, message, data, report, product). Not "value for users."
- [ ] `goal.consumers` lists who or what uses the output.
- [ ] `goal.success_metric` names how we know the output is correct/good.
- [ ] `goal.failure_signal` names how we know the output is wrong.
- [ ] `goal.volume` is an estimate of how many outputs per unit of time (day/week/month).
- [ ] `goal.time_sensitivity` is one of: `realtime | daily | weekly | adhoc`.

**Gap-check questions to confirm all above:**

- "If the process runs perfectly tomorrow, what exact file / message / record will exist that wasn't there before?"
- "Who (or what system) will open / read / use that output?"
- "What makes it a 'good' output vs a 'bad' one? Give me a before/after example."
- "How many do you need per day (or week, month)?"
- "Does it have to be ready at a specific time?"

**Do-not-proceed if:** output is described in abstract terms without a concrete example.

---

## Section 2 — Inputs & Triggers

- [ ] `triggers[]` has at least one entry, each with a clear `type` and `detail`.
- [ ] `inputs[]` has every input listed with `name`, `source`, `format`, `reliability`, and `trigger_role`.
- [ ] Each input's `source` is a named system, person, or channel — not abstract ("the data").
- [ ] Every input's `format` is specified: `structured | semi-structured | unstructured`.
- [ ] Every input's `reliability` is specified: `always | usually | sometimes`.

**Gap-check questions:**

- "Is there an input you haven't listed that usually comes along with one of these? (Cover letter with a résumé, customer ID with an email, etc.)"
- "Are there inputs you currently don't get but wish you did? (Marking those as 'missing inputs' sharpens the case for automation.)"
- "When an expected input is missing today, what happens? Does the process stop, or does someone go find it?"

**Do-not-proceed if:** any input is described as "just stuff we have" without a named source.

---

## Section 3 — Steps & Order

- [ ] `steps[]` has every step the user described, including "obvious" ones.
- [ ] Each step has `id`, `name`, `description`, and `type` (logical / mixed / creative).
- [ ] Each step has `depends_on[]` even if the array is empty (explicit > implicit).
- [ ] Each step has `failure_mode` (what goes wrong; what happens when it does).
- [ ] For each step marked `mixed` or `creative`, `creativity_notes` is populated with the signals the human uses.

**Gap-check questions:**

- "Is there a step so obvious you forgot it? (Opening the email, saving the file, double-checking the date?)"
- "If a new hire shadowed you, what would they see you do that isn't in this list?"
- "When this step produces a surprising result, what's the next thing you do?"

**Do-not-proceed if:** any step is labeled "I just figure it out" and `creativity_notes` is empty. See `logical-vs-creative.md` for how to decompose compressed judgment.

---

## Section 4 — Dependencies & Feedback Loops

- [ ] `dependencies[]` captures every inter-step relationship with `from_step`, `to_step`, `type`, and `detail`.
- [ ] `feedback_loops[]` captures every case where a later step can send work back to an earlier one, with `trigger_condition`, `loops_back_to`, and `limit`.
- [ ] `shared_state[]` captures every piece of state that multiple steps read or write.
- [ ] The dependency graph is acyclic **except** for explicitly-modeled feedback loops (no hidden cycles).

**Gap-check questions:**

- "Earlier you mentioned that if X fails quality, you redo Y. That's a feedback loop — let me model it explicitly."
- "Do any steps share a document, a file, a database row, or a calendar?"
- "Is there any case where you do step 5 before step 3 based on circumstances?"

**Do-not-proceed if:** the user says "it's all linear" but has earlier described any rework or branching path.

---

## Section 5 — Time Horizon

- [ ] `timing.current_total_duration` is recorded (even if approximate).
- [ ] `timing.target_total_duration` is recorded.
- [ ] `timing.mode` is one of: `sync | async | batched`.
- [ ] `timing.hard_deadlines[]` lists any deadlines, or is explicitly empty.

**Gap-check questions:**

- "How long does one full run take today, start to finish, including waiting?"
- "What's the longest any single step takes?"
- "Is there a hard deadline (end-of-day, meeting, SLA)?"
- "Can the whole thing run overnight without a human watching, or does it need to produce a result while a human is online?"

**Do-not-proceed if:** `timing.mode` is unspecified.

---

## Section 6 — Quality Criteria

- [ ] Every step has an entry in `quality_gates[]` OR an explicit note in its `steps[]` entry saying "no gate needed — reason".
- [ ] Each quality gate has `criterion`, `verifier` (automated / human / hybrid), `severity` (soft / hard), and `on_fail` behavior.
- [ ] The most consequential steps have `severity: hard` (pipeline stops on failure).
- [ ] Any legal / compliance / privacy / safety constraint is captured as a hard gate.

**Gap-check questions:**

- "For step N, how do you know today whether it went well?"
- "Has this process ever produced an embarrassing or harmful result? What caught it?"
- "If the automated version is uncertain, should it auto-fix, flag for human, or stop entirely?"
- "Are there any outputs that are legally constrained? (Privacy, financial reporting, advertising claims?)"

**Do-not-proceed if:** any step has no gate and no explicit "no gate needed" note.

---

## Section 7 — Creative Components

For every step with `type: mixed` or `type: creative`:

- [ ] `creative_components[i].creativity_lens` is named (originality / brand-voice / cultural-fit / interpretation / ...).
- [ ] `creative_components[i].answer_space` is `single_correct | multiple_acceptable`.
- [ ] `creative_components[i].bias_risks[]` has at least one named risk, drawn from `bias-control-patterns.md`.
- [ ] Each bias risk has a `mitigation` naming which specific pattern from `bias-control-patterns.md` neutralizes it.
- [ ] `creative_components[i].verifier_design` names the verification pattern to use (e.g., "dual-field SSoT + independence clause + three-agent consensus").

**Gap-check questions:**

- "For step N, which bias would an AI agent most likely exhibit here? Pattern-accumulation? Favorite-topic? Confirmation?"
- "Which verification pattern matches that bias? Is there a fix in bias-control-patterns.md that applies cleanly?"
- "Is the answer space open (multiple good answers possible) or closed (one right answer)?"

**Do-not-proceed if:** any creative or mixed step has empty `bias_risks[]`.

---

## Section 8 — SSoT Candidates

- [ ] `ssot_candidates[]` lists every piece of data that more than one step reads from.
- [ ] Each candidate has `name`, `purpose`, `consumers[]`, and `structure_hint` (labels_only / dual_field / thresholds / aliases / composite).
- [ ] Each candidate has a `drift_risk_note` — one sentence about what silently breaks if consumers get out of sync.

**Gap-check questions (from `ssot-patterns.md`):**

- "Is there a list of categories, labels, types, or states multiple steps need to know about?"
- "Is there a set of rules both the producer and the checker of an output must follow?"
- "Is there a set of reference thresholds multiple steps use?"
- "Is there a dictionary of aliases or canonical names the pipeline normalizes against?"

**Do-not-proceed if:** the quality gates in Section 6 reference rules that aren't captured as an SSoT candidate.

---

## Final Handoff Gate

Before producing the final artifacts, confirm all of the following:

- [ ] Every section 1–8 checklist item above is ticked.
- [ ] `architecture-spec.yaml` is valid YAML, parses without error.
- [ ] `architecture.mmd` is valid Mermaid and renders in GitHub preview.
- [ ] Every step id referenced in `dependencies`, `feedback_loops`, `quality_gates`, `creative_components`, `ssot_candidates.consumers` actually exists in `steps[]`.
- [ ] No `depends_on` references a step that doesn't exist.
- [ ] No `loops_back_to` creates an unbounded cycle without a `limit`.
- [ ] The spec has been summarized back to the user and they have explicitly said "yes, that matches my process."

If all boxes are checked, print:

> Architecture complete and handoff-ready. Spec at `./architecture-spec.yaml`, diagram at `./architecture.mmd`.
>
> Next step: `process-automation-skill` with this spec as input.

If any box is unchecked, print the failing items and return to the corresponding section.

---

## Example valid architecture-spec.yaml

Here's a minimal example of what a valid, handoff-ready spec looks like. Use it as a template when drafting, and as a validation target when checking.

```yaml
process_name: customer-support-triage
process_type: mixed

goal:
  output: "A routed ticket with severity, category, and assigned team."
  consumers: ["support team leads", "ticket dashboard"]
  success_metric: "Ticket reaches correct team within 2 minutes, severity matches human grader within ±1 level in 95% of cases."
  failure_signal: "Ticket routed to wrong team (requires re-routing) or severity off by 2+ levels."
  volume: "~200 tickets/day"
  time_sensitivity: realtime

inputs:
  - name: raw_ticket_body
    source: "Zendesk webhook"
    format: semi-structured
    reliability: always
    trigger_role: true
  - name: customer_history
    source: "Customer DB API"
    format: structured
    reliability: usually
    trigger_role: false

triggers:
  - type: event
    detail: "New ticket created in Zendesk"

steps:
  - id: s1
    name: "Parse ticket"
    description: "Extract subject, body, customer ID, and metadata from the raw payload."
    type: logical
    depends_on: []
    can_parallel_with: []
    failure_mode: "Malformed JSON → quarantine and alert."

  - id: s2
    name: "Classify category"
    description: "Assign ticket to one of five support categories."
    type: mixed
    depends_on: [s1]
    failure_mode: "Low-confidence classification → route to generalist queue."
    creativity_notes: "Human reads subject + first paragraph; infers from known product lines and language cues."

  - id: s3
    name: "Estimate severity"
    description: "Assign severity P0–P4 based on customer tier and issue keywords."
    type: mixed
    depends_on: [s1]
    can_parallel_with: [s2]
    failure_mode: "Boundary cases (P2 vs P3) → human grader every 50th ticket for drift monitoring."

  - id: s4
    name: "Route to team"
    description: "Based on category + severity, assign to owning team per routing matrix."
    type: logical
    depends_on: [s2, s3]
    failure_mode: "Unknown category × severity combo → route to generalist."

dependencies:
  - from_step: s1
    to_step: s2
    type: data
    detail: "s2 reads parsed subject + body"
  - from_step: s1
    to_step: s3
    type: data
  - from_step: s2
    to_step: s4
    type: data
  - from_step: s3
    to_step: s4
    type: data

feedback_loops:
  - trigger_condition: "s4 routing bounces back within 10 minutes"
    from_step: s4
    loops_back_to: s2
    limit: "Max 1 re-classification attempt; then escalate to human."

shared_state: []

timing:
  current_total_duration: "15 min manual"
  target_total_duration: "2 min automated"
  mode: sync
  hard_deadlines:
    - step: s4
      deadline: "Within 2 minutes of ticket arrival"

quality_gates:
  - step: s2
    criterion: "Classification confidence ≥ 0.8 OR route to generalist"
    verifier: automated
    severity: soft
    on_fail: escalate_to_human

  - step: s3
    criterion: "Severity estimate matches human grader ±1 level on sampled audit"
    verifier: hybrid
    severity: soft
    on_fail: retry_once

  - step: s4
    criterion: "Routing decision matches routing-matrix SSoT exactly"
    verifier: automated
    severity: hard
    on_fail: quarantine

creative_components:
  - step_id: s2
    creativity_lens: interpretation
    answer_space: multiple_acceptable
    bias_risks:
      - risk: pattern_accumulation
        mitigation: batch_split_at_10
      - risk: favorite_category
        mitigation: two_stage_open_mode_classifier
    verifier_design: "Classifier reads category-list SSoT directly; independence clause; batch-split at 10."

  - step_id: s3
    creativity_lens: interpretation
    answer_space: single_correct
    bias_risks:
      - risk: confirmation_bias
        mitigation: dual_path_verification
      - risk: paraphrase_drift
        mitigation: ssot_sourced_guidelines
    verifier_design: "Severity SSoT with dual-field (description + verification_criterion); one verifier + periodic human audit."

ssot_candidates:
  - name: categories
    purpose: "Enumerate the 5 support categories with display names"
    consumers: [s2, s4]
    structure_hint: labels_only
    drift_risk_note: "New category added to classifier but not routing matrix → tickets route to generalist by default."
  - name: severity_rules
    purpose: "Define P0–P4 with writer-facing description + verifier-facing criterion"
    consumers: [s3]
    structure_hint: dual_field
    drift_risk_note: "Writer description loosened without verifier criterion → drift in severity distribution over time."
  - name: routing_matrix
    purpose: "Map (category, severity) → owning team"
    consumers: [s4]
    structure_hint: composite
    drift_risk_note: "Team org change not reflected → routing goes to disbanded team."

bias_risk_map:
  s2: [pattern_accumulation, favorite_category]
  s3: [confirmation_bias, paraphrase_drift]
```

This example has 4 steps, 2 creative components, 3 SSoT candidates, 1 feedback loop, and 3 quality gates — tight, specific, complete. A spec of this shape can be handed off to `process-automation-skill` and built mechanically.

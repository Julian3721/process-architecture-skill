# process-architecture-skill

An interview-driven [Claude Code](https://claude.com/claude-code) skill that helps you map any process into a complete, gap-free architecture — ready to hand off to AI-based automation.

> Lesen auf Deutsch? → [README.de.md](./README.de.md)

---

## What this is

A specialized skill that walks you through a structured interview about your process (a current job, a business idea, a repetitive workflow — anything) and produces two artifacts:

1. **`architecture-spec.yaml`** — a structured, validated specification of every step, dependency, input, feedback loop, quality gate, and bias risk in your process.
2. **`architecture.mmd`** — a Mermaid diagram of the process, rendered directly in GitHub and most IDEs.

The skill **does not** just collect what you say. It challenges vague answers, names the gaps it detects, and refuses to move on until the architecture is complete. The output is meant to be so rigorous that the subsequent automation becomes mechanical.

---

## Why this exists

Most attempts at AI-based process automation fail at the architecture phase, not at the implementation phase. Teams jump straight to "let's prompt an LLM to do X" without first mapping:

- What the process *actually* produces (concrete deliverable)
- Where every input comes from (not "just the data")
- What the real quality gates are (especially the implicit ones where "I just know")
- Which steps are logical (easy to automate) vs. creative (needs bias control)
- Which silent feedback loops exist
- Which metadata entities need to be a Single Source of Truth

Without these answers, the resulting automation passes demos and fails production.

This skill is the condensed methodology from a shipped project — the **Rangello** daily estimation game — where a quiz-show-style question-writer's job was fully replaced by a 12-stage AI pipeline with bias-free verification, multi-agent consensus, and single-source-of-truth metadata. The bias-control and architecture patterns distilled there are now in [`skills/process-architecture/references/bias-control-patterns.md`](./skills/process-architecture/references/bias-control-patterns.md) — 48 principles, each with the Rangello source-code reference preserved.

---

## The two-skill system

This skill is one half of a pair.

| Skill | Purpose |
|---|---|
| **`process-architecture-skill`** (this repo) | Design the complete architecture. Interview-driven. Outputs `architecture-spec.yaml` + `architecture.mmd`. |
| **[`process-automation-skill`](https://github.com/Julian3721/process-automation-skill)** | Take the completed spec and build the actual agent pipeline. Model selection per step, SSoT files, verifier agents, quality gates. |

You always run `process-architecture-skill` first. When it declares the architecture complete, you switch to `process-automation-skill` with the spec file in hand.

---

## Installation

### Option 1 — As a project skill

Drop the `skills/process-architecture/` directory into your project's `.claude/skills/` folder:

```bash
git clone https://github.com/Julian3721/process-architecture-skill.git
cp -r process-architecture-skill/skills/process-architecture /path/to/your/project/.claude/skills/
```

Claude Code will discover the skill on next launch.

### Option 2 — As a user-level skill

Install globally for all your projects:

```bash
mkdir -p ~/.claude/skills
cp -r process-architecture-skill/skills/process-architecture ~/.claude/skills/
```

### Option 3 — As a plugin

If you use Claude Code's plugin system, add this repo as a local plugin source and install via the plugin marketplace.

---

## Usage

Once installed, invoke the skill in any natural way — the trigger is broad by design:

- `"I want to automate my job, help me design it"`
- `"design the architecture for my workflow"`
- `"prozess automatisieren"` (German)
- `"architect this process for me"`

The skill will introduce itself, ask about process type (logical vs creative), and walk through 7 interview sections:

1. **Goal & Output** — what does the process produce?
2. **Inputs & Triggers** — what flows in, what starts it?
3. **Steps & Order** — every decision and transformation, in sequence
4. **Dependencies & Feedback Loops** — how steps affect each other, including loops
5. **Time Horizon** — latency budgets, deadlines, sync vs async
6. **Quality Criteria** — how do we know the output is good, and what gates catch failures?
7. **Creative Components** — for every step where judgment enters, which bias risks apply and how we neutralize them

Expect the interview to take **30–90 minutes** of back-and-forth. That's not a bug. Every gap closed here saves 10× effort during implementation.

---

## Core concept: logical vs creative

The single most important distinction the skill enforces:

| Type | Examples | Automation approach |
|---|---|---|
| **Logical** | invoice arithmetic, data cleanup, compliance checks, FAQ routing | Cheap model, minimal verification, simple SSoT |
| **Mixed** | severity classification, ticket routing with context, content moderation gray areas | Mid-tier model, dual-field SSoT, single verifier with independence clause |
| **Creative** | original writing, curriculum design, tagline generation, fact-checked original questions | Strong reasoning model, multi-agent consensus, full bias-free verifier architecture, quarantine over retry |

See [`skills/process-architecture/references/logical-vs-creative.md`](./skills/process-architecture/references/logical-vs-creative.md) for the full classification framework, including the traps (calling a creative step logical, calling a mixed step logical, etc.).

---

## The bias-control lineage

Creative steps face predictable biases when automated:

- **Pattern accumulation** — the agent reuses phrasings across a batch
- **Peer-pressure averaging** — verifier ratings normalize across the batch
- **Paraphrase drift** — rules copy-pasted between files diverge silently
- **Confirmation bias** — verifier only searches for agreeing sources
- **Favorite-default** — agent leans toward easier or more familiar subjects
- **Language-bias leakage** — agent reasoning shifts across working languages

The Rangello project discovered and neutralized each of these with specific patterns (fresh-subagent-dispatch, SSoT-sourced guidelines, intra-batch independence clauses, batch-split at 10, model heterogeneity, refutation-search, reputation weighting, per-claim consensus, English-only internal artifacts, and more).

All 48 principles are catalogued in [`skills/process-architecture/references/bias-control-patterns.md`](./skills/process-architecture/references/bias-control-patterns.md), grouped into 12 families, each with the exact Rangello source reference so you can see the pattern in a running system.

---

## What the skill will NOT do

- **It will not let you skip sections.** Every section has at least one "do-not-proceed" condition. If your process has no inputs named with sources, the skill won't move on until they're named.
- **It will not silently simplify.** If you describe a step with 7 branches, it models 7 branches. Compression hides bias.
- **It will not design the implementation.** That's the automation skill's job. This skill refuses to pick models, prompt styles, or pipeline architecture — it produces a spec and stops.
- **It will not tolerate "I just know."** When a user says their quality check is intuitive, the skill decomposes the intuition into signals. This is the single most valuable thing the skill does.

---

## Limitations and honest caveats

- Processes that depend heavily on emotional attunement, long-term human relationships, or live physical sensing are not good automation targets *today*. The skill can still map them, but you should expect the automation skill to flag that the bias-risk surface is unbounded.
- The skill assumes you can articulate your own process. If you can't, you need a process expert first, not this skill.
- The methodology is grounded in the Rangello project's scale (hundreds of questions, few dozen categories, 3 languages). For very different scales (millions of items/day, highly regulated domains), some patterns may need extension.
- The skill is opinionated. It will push back on "we've always done it this way" if the answer reveals an unnamed gap. That pushback is the point.

---

## Contributing

PRs welcome. Especially valuable:

- Additional domain examples in `references/logical-vs-creative.md` (new industries the classification framework hasn't been tested against)
- New bias-control patterns you've discovered and can cite with source code
- Translations of the interview-facing rhetoric into more languages (currently EN + DE)
- Real-world architecture specs you've produced with this skill (as case-study examples)

Open an issue before large changes to discuss fit.

---

## License

MIT. See [LICENSE](./LICENSE).

---

## Companion skill

[`process-automation-skill`](https://github.com/Julian3721/process-automation-skill) — the second half of the pair. Takes the spec you produce here and builds the actual agent pipeline.

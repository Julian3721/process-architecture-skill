# CLAUDE.md — process-architecture-skill

## What this repo is

An open-source Claude Code skill that interviews a user about a process they want to automate and produces:

1. A structured `architecture-spec.yaml` (goal, inputs, steps, dependencies, feedback loops, quality gates, creative components, SSoT candidates)
2. A Mermaid architecture diagram (`architecture.mmd`)

The output is the handoff contract to the companion [`process-automation-skill`](https://github.com/Julian3721/process-automation-skill), which will later turn the spec into a working agent pipeline.

## Origin story (critical context)

This skill is the distillation of patterns from the **Rangello** daily estimation game project (`/Users/juliandomnik/Projects_2026/Rangello/`), where a quiz-show-style fact-checked question-writing job was replaced by a 12-stage AI pipeline with bias-free verification, multi-agent consensus, and Single-Source-of-Truth metadata.

Every bias-control pattern in `skills/process-architecture/references/bias-control-patterns.md` is cited with the exact Rangello source-code location, so anyone can see the pattern in a running system.

**When in doubt about a pattern, check the Rangello repo first** — it's the reference implementation.

## Repo layout

```
process-architecture-skill/
├── skills/
│   └── process-architecture/
│       ├── SKILL.md                                     ← main skill (8 interview sections)
│       ├── references/
│       │   ├── bias-control-patterns.md                 ← 48 principles, 12 groups (SSoT for bias patterns)
│       │   ├── logical-vs-creative.md                   ← step-type classification framework
│       │   ├── ssot-patterns.md                         ← Single-Source-of-Truth design rules
│       │   └── interview-checklist.md                   ← handoff-gate + example spec
│       └── assets/                                      ← (empty, for future example diagrams)
├── README.md                                            ← English, full
├── README.de.md                                         ← German, full
├── LICENSE                                              ← MIT
└── .gitignore
```

## Design principles enforced by the skill

1. **Gap-detection over politeness.** Every interview section has explicit "do-not-proceed" conditions. The skill names gaps out loud.
2. **Logical-vs-creative distinction drives every downstream decision.** Model selection, verifier topology, SSoT shape, quality-gate severity — all follow from step-type classification.
3. **Never accept "I just know"** as a final answer — decompose intuition into named signals.
4. **Spec is handoff-contract.** The output must be complete enough that `process-automation-skill` can build the pipeline mechanically.
5. **Rangello is the reference.** All patterns cite concrete source locations; paraphrases are forbidden.

## Current status

- **Initial release shipped** (commit `c8cf16d`). All 8 files in place, public, MIT, both READMEs complete.
- **Not yet validated on a real user process.** The interview flow is rigorous on paper but has not survived contact with a non-author user.
- **No example architecture-spec.yaml in `assets/`** — next step is to generate one from Rangello itself (dog-fooding).

## Known work-in-progress items

| Priority | Item | Notes |
|---|---|---|
| High | Dog-food on Rangello | Run the skill against Rangello's question-generator process. If the resulting spec cannot be reverse-engineered into the actual Rangello pipeline, the skill has gaps. |
| High | 3–5 validation interviews with users | Different domains (accounting, support triage, content writing, research synthesis, operations). Record where users got frustrated or where the skill accepted vague answers. |
| Medium | Add `assets/example-architecture-diagram.svg` | Rendered Mermaid diagram from the interview-checklist.md example. Helps users visualize the output upfront. |
| Medium | Figure out how to handle extremely large processes (>20 steps) | Current interview cadence assumes ≤15 steps. Longer processes need a divide-and-conquer variant. |
| Low | Additional language support in interview rhetoric | Currently EN + DE. ES, FR, ZH are candidates. |
| Low | Marketplace submission | Once stable, submit to Claude Code plugin marketplace. |

## Relationship to companion skill

The output of this skill (`architecture-spec.yaml`) is the input to `process-automation-skill`. That skill is currently a scaffold (see its repo). The two skills must co-evolve:

- If the automation-skill needs a field the architecture-skill doesn't collect, add the field to this skill's interview first.
- If the architecture-skill collects a field the automation-skill doesn't use, remove it (don't accumulate dead weight).

## Conventions in this codebase

- **Language for internal artifacts**: English. All IDs, comments, commit messages, interview-facing rhetoric defaults are English. User-facing rhetoric may be bilingual (EN + DE mixed in SKILL.md based on user language).
- **Prose quality**: documentation-grade, no marketing fluff. If a section isn't earning its keep, delete it.
- **References citation format**: every Rangello pattern cites `file:line` or `section-name`, never "somewhere in Rangello".
- **SKILL.md description field**: long and trigger-dense, following Claude Code convention (250–500 chars with enumerated trigger phrases). See `question-generator` in Rangello for the pattern.
- **No hardcoded examples in SSoT**: if a pattern file lists examples, they're illustrative, not prescriptive. The underlying rule is in the file; examples live in comments/docs.

## Testing plan (once ready)

No automated tests yet — the skill is narrative, not code. Validation is:

1. Dog-food on Rangello (can the skill reproduce a spec matching the actual Rangello pipeline?)
2. 3–5 user interviews from distinct domains (does the skill catch gaps these users wouldn't have caught themselves?)
3. Handoff rehearsal (feed a completed spec to a human pretending to be `process-automation-skill` — can they build the pipeline from spec alone?)

## Key external references

- Rangello repo: `/Users/juliandomnik/Projects_2026/Rangello/`
  - Question-generator skill: `.claude/skills/question-generator/SKILL.md`
  - Fact verifier agents: `.claude/agents/fact-verifier-*.md`
  - Content-policy verifier: `.claude/agents/content-policy-verifier.md`
  - Architecture journal: `docs/architecture-journal.md`
  - Skills portfolio: `SKILLS_PORTFOLIO.md`
  - Metadata consistency tests: `src/lib/__tests__/metadata-consistency.test.ts`

## For future Claude sessions working on this repo

1. Read `skills/process-architecture/SKILL.md` first — it's the source of truth for the interview flow.
2. Read `skills/process-architecture/references/bias-control-patterns.md` if working on verification logic.
3. Every change to the interview flow must preserve the "do-not-proceed" discipline — don't soften gap-detection for user convenience.
4. Every new bias-control pattern added to `references/bias-control-patterns.md` must cite a concrete source (Rangello or otherwise). No abstract principles without a running-code reference.
5. Keep the Rangello citation trail intact — it's what makes the skill credible.
6. When the companion automation-skill adds a spec field requirement, update the interview in this skill to collect it, and update `interview-checklist.md` accordingly.

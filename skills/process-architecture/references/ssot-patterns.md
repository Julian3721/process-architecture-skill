# Single Source of Truth (SSoT) Patterns

An SSoT is a piece of data that multiple parts of the pipeline read from. Done right, it's the backbone of a maintainable automation. Done wrong, it causes silent paraphrase drift — the single most common cause of inexplicable quality regressions.

This file covers:

- What makes a good SSoT
- Concrete examples from Rangello
- Anti-patterns that destroy SSoT discipline
- How to enforce SSoT discipline with tests

---

## What makes a good SSoT

A good SSoT has four properties:

1. **Authoritative.** There is exactly one place where this data lives. Every consumer reads from that place, not from a copy.
2. **Structured.** It's in a machine-readable format (JSON, YAML) with a stable schema, not free-text that requires parsing.
3. **Minimal.** It contains only what consumers need. No prose commentary, no rationale, no examples that aren't referenced in code.
4. **Versioned.** It lives in the source repo under git. Changes are reviewable diffs, not edits to a running database.

If any of those four are missing, you don't have an SSoT — you have a convention that will drift.

---

## Candidates for SSoT (the questions to ask during interview)

When you reach Section 6 (Quality Criteria) or Section 7 (Creative Components) in the interview, ask these questions to identify SSoT candidates:

1. **"Is there a list of categories, labels, types, or states that multiple steps need to know about?"** (Classic label SSoT.)
2. **"Is there a set of rules that both the producer and the checker of an output need to follow?"** (Classic dual-field SSoT — description + verification_criterion.)
3. **"Is there a set of reference values (thresholds, limits, tolerances, quotas) that multiple steps reference?"** (Classic threshold SSoT.)
4. **"Is there a dictionary of aliases, normalizations, or canonical names that the pipeline uses?"** (Classic alias SSoT.)

Every positive answer is a candidate. Record each one under `ssot_candidates[]` in the architecture spec with:

```yaml
ssot_candidates:
  - name: <short-name>
    purpose: <what is this SSoT for>
    consumers: [<which steps read it>]
    structure_hint: <labels only | dual-field | thresholds | aliases>
    drift_risk_note: <one-line description of what happens if consumers drift apart>
```

---

## Concrete Rangello examples (patterns to imitate)

### Pure-Label SSoT: `data/categories.json`

```json
{
  "categories": [
    {
      "id": "geography",
      "name_de": "Geografie",
      "name_en": "Geography",
      "name_es": "Geografía",
      "color": "blue"
    },
    ...
  ]
}
```

**Why this is a good SSoT:**

- Exactly 5 fields per category. Nothing else.
- No scope text, no avoid lists, no examples, no tier hints.
- The writer prompt dynamically reads this file at dispatch and gets the list of valid categories.
- The open-mode classifier subagent reads this file directly to know what labels to pick from.
- UI components read this file to render category badges with the right color.

**What this SSoT deliberately excludes:**

- Per-category instructions to the writer ("for geography questions, avoid populations because they're boring"). Such instructions would be frozen curator-bias. If a category is boring, that's a problem with the category set itself, not something to paper over with a written rule.
- Examples. Examples in an SSoT become de-facto rules — writers match the examples rather than the rule. Keep examples in documentation if needed, not in SSoT.

### Dual-Field SSoT: `data/difficulties.json`

```json
{
  "difficulties": {
    "easy": {
      "name": "Easy",
      "description": "The writer aims for an answer that most people could guess within a factor of 2 — the range they naturally visualize is already close to the right size.",
      "verification_criterion": "Check: could an average adult, without research, guess the answer within ±50%? If yes, it's easy. If it requires specialized knowledge to come close, it's not easy.",
      "weight": 0.4
    },
    "normal": {...},
    "hard": {...}
  }
}
```

**Why dual-field is powerful:**

The writer prompt embeds `description` — this gives the writer a framing of what "easy" should feel like during generation.

The verifier prompt embeds `verification_criterion` — this gives the verifier a concrete test to apply.

The two are in the same file, under the same entity, so they can never drift independently. If you update one, the diff makes it obvious the other needs review. A test enforces that every difficulty has a non-empty `verification_criterion`, so the pattern cannot be broken by adding a new difficulty without its verifier-facing formulation.

### Threshold SSoT: `data/linguistic-rules.json`

```json
{
  "confidence_thresholds": {
    "en": 0.9,
    "de": 0.9,
    "es": 0.95
  },
  "meta_questions": {
    "en": [...],
    "de": [...],
    "es": [...]
  },
  ...
}
```

**Why this is a good SSoT:**

- Thresholds are data. Hardcoding them in SKILL.md would scatter tuning knobs all over the codebase.
- Per-language structure lets the coordinator pick the right threshold for the question being verified without a lookup table.
- A test enforces that `confidence_thresholds.es > confidence_thresholds.en` and `> confidence_thresholds.de` — ensuring the language-asymmetry principle cannot be broken by a careless edit.

### Alias SSoT: `data/unit-system.json`

```json
{
  "aliases": {
    "km": "kilometer",
    "kilometers": "kilometer",
    "kilometres": "kilometer",
    ...
  },
  "metrics": {
    "distance": {
      "units": ["meter", "kilometer", "mile"],
      "base_unit": "meter",
      ...
    },
    ...
  }
}
```

**Why this is a good SSoT:**

- The writer uses the `aliases` map to normalize varying inputs to the canonical form.
- The scorer uses `base_unit` + conversion logic to convert any unit to SI for scoring.
- The UI uses the display labels for output.
- All in one file, so a new unit is added exactly once.

---

## Anti-patterns (SSoT mistakes that undo the discipline)

### Anti-pattern 1: Prose in the SSoT

**Example.** `categories.json` with a `description` field that's a paragraph of advice to the writer.

**Why bad.** Prose is hard to keep in sync. It tempts curators to embed bias as architecture. It makes verifiers' prompts balloon.

**Fix.** Move prose to documentation. Keep the SSoT to the minimum structured fields consumers actually need.

### Anti-pattern 2: Paraphrased copies in prompts

**Example.** The writer prompt says "remember, easy means someone could guess it roughly", and the verifier prompt says "easy questions are ones a casual player could estimate within a factor of 2". These paraphrase the same SSoT field. Over time, they drift — a new difficulty rule gets added to one but not the other.

**Why bad.** Silent drift. The pipeline passes content that should be caught. Nobody notices for weeks.

**Fix.** Both prompts embed the SSoT field via code-level substitution, not via handwritten paraphrase. Example: `{{DIFFICULTY_CRITERIA}}` placeholder in the prompt, replaced at dispatch time with the content of `data/difficulties.json`.

### Anti-pattern 3: Hardcoded constants that should be SSoT

**Example.** A threshold of `0.9` appears in three places in code. One of them gets tuned to `0.92` during a debugging session. The other two keep using `0.9`. The pipeline's behavior is now inconsistent and non-obvious.

**Why bad.** Scattered tuning is un-testable and un-auditable.

**Fix.** Move the threshold to SSoT. Every consumer reads from the same field.

### Anti-pattern 4: SSoT with no test-enforcement

**Example.** `metrics.json` has 17 metric IDs. `unit-system.json` has 16 (one was deleted but not cleaned up elsewhere). Neither file knows about the other.

**Why bad.** IDs silently drift. Pipeline fails in confusing ways when the missing metric is hit.

**Fix.** Write a test that reads both files and asserts ID parity. Example: `src/lib/__tests__/metadata-consistency.test.ts` in Rangello. The test runs in CI, so any commit that breaks parity fails immediately.

### Anti-pattern 5: SSoT in a database

**Example.** Categories live in a production database. Every deployment reads from the live DB.

**Why bad.** Changes are not reviewable. You can't diff them. You can't see who changed what, when, or why. Rollback requires another live edit.

**Fix.** Keep SSoT in version-controlled files (JSON, YAML). If runtime needs it fast, cache it at process startup. The canonical source is always the file in the repo.

### Anti-pattern 6: SSoT with examples that drift

**Example.** `categories.json` has an `examples` field: `["Is the Nile the longest river?", "What's the population of Cairo?"]`. Over time, the writer uses these as templates, and all geography questions start sounding like them.

**Why bad.** Examples become de-facto rules. They collapse the answer space. Bias you thought you were avoiding by staying "open" is re-introduced through examples.

**Fix.** No examples in SSoT. If you need examples for onboarding or docs, put them in a separate, non-consumed file.

---

## SSoT discipline: test-enforced invariants

Every SSoT in Rangello has one or more test-enforced invariants. These are the ones that apply broadly:

### Structural invariants

- Every entity has all required fields.
- No extra fields beyond the schema.
- All IDs match the expected pattern (e.g., lowercase-with-underscores).
- Enum-valued fields have values drawn from the allowed set.

### Parity invariants

- If SSoT A and SSoT B share IDs, IDs must match 1:1.
- If SSoT A has field X, SSoT B does not duplicate X (keep data in one place).

### Relational invariants

- Confidence threshold for a language the owner cannot verify must be strictly higher than for languages the owner speaks.
- Sum of weights across a set must equal 1.0 (within floating-point tolerance).
- Every cluster referenced in throttle config exists in the cluster registry.

### Purity invariants

- SSoT must not contain prose fields (only structured data).
- SSoT must not contain examples (only rules).
- SSoT must be parseable by a strict schema validator.

### How to write one

In TypeScript / Jest / Vitest:

```ts
import categories from '../../../data/categories.json'
import metrics from '../../../data/metrics.json'
import unitSystem from '../../../data/unit-system.json'

describe('metadata-consistency', () => {
  it('categories.json — pure label invariant', () => {
    for (const cat of categories.categories) {
      const allowedKeys = new Set(['id', 'name_de', 'name_en', 'name_es', 'color'])
      const actualKeys = new Set(Object.keys(cat))
      expect(actualKeys).toEqual(allowedKeys)
    }
  })

  it('metrics.json ↔ unit-system.json — ID parity', () => {
    const metricIds = new Set(metrics.metrics.map((m) => m.id))
    const unitSystemIds = new Set(Object.keys(unitSystem.metrics))
    expect(metricIds).toEqual(unitSystemIds)
  })

  it('linguistic-rules.json — es threshold > en and de', () => {
    expect(linguistic.confidence_thresholds.es).toBeGreaterThan(linguistic.confidence_thresholds.en)
    expect(linguistic.confidence_thresholds.es).toBeGreaterThan(linguistic.confidence_thresholds.de)
  })
})
```

Each test catches a specific drift at commit time. Combined, they make the SSoT architecture self-healing — any commit that breaks an invariant fails CI before it merges.

---

## Recommendation for your project

For every SSoT candidate you identified in the architecture interview, specify in the spec:

```yaml
ssot_candidates:
  - name: <short-name>
    purpose: <one sentence>
    consumers: [<steps>]
    structure_hint: labels_only | dual_field | thresholds | aliases | composite
    test_invariants:
      - type: structural
        description: <one-line>
      - type: parity
        description: <one-line>
    anti_patterns_to_avoid:
      - prose_fields
      - hardcoded_thresholds_outside
      - examples_in_data
```

When `process-automation-skill` builds the automation, it will use this spec to generate the actual JSON files, the consumers' read logic, and the test suite — so every invariant is enforced from day one, and drift is prevented before it starts.

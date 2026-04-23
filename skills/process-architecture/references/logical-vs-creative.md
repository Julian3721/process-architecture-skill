# Logical vs Creative — classifying process steps

The single most important classification you'll do during the interview is: **is this step logical, mixed, or creative?** Every downstream decision (which model, how much bias-control, how many verifiers, what kind of SSoT) follows from this answer.

This file gives you the criteria.

---

## The three types

### Logical (rule-based, deterministic)

A logical step has a rule that, given the same inputs, always produces the same correct output. Two people applying the rule competently will agree on the answer.

**Canonical examples:**

- Invoice arithmetic (sum line items, apply tax rate, produce total)
- Regulatory compliance checks (does this filing include every mandatory field?)
- Data transformation (CSV → JSON, normalize phone numbers, reformat dates)
- Booking a flight within given constraints (departure window, budget, airline rules)
- Matching a customer query against an FAQ set (which FAQ answers this?)
- Generating a sales report from a sales database (known schema, known formulas)
- Scheduling based on availability (given calendars, find open slot)

**Telltale signs it's logical:**

- You can describe the rule in one or two sentences and a junior can apply it correctly.
- Two people produce the same answer independently.
- The answer is either right or wrong; there's no "well, it depends on taste."
- The rule is written down somewhere (law, policy, schema, protocol), or could be.

**Automation implications:**

- Use a small, cheap model (Haiku or equivalent).
- One verifier is usually enough.
- SSoT needs to be precise but doesn't need a `verification_criterion` field — the `description` is enforceable as-is.
- Bias-control is minimal — no independence clause, no batch-split, no multi-agent consensus required.

---

### Mixed (rule-based with judgment in a constrained corridor)

A mixed step has rules that narrow the answer space, but within that space a human makes a judgment. Different competent humans may pick slightly different answers, but all of them are "within the corridor" of acceptable.

**Canonical examples:**

- Classifying a customer complaint into one of five categories (rules constrain to five; within that, some judgment)
- Deciding the severity of a reported bug (high / medium / low — rules exist, but edge cases need judgment)
- Ranking candidates for an interview round from a stack of applications (criteria are known, ranking is judgment)
- Writing a routine email response based on a template + context (structure fixed, wording judgment)
- Deciding whether a document needs a supervisor's sign-off (rules narrow the question, but edge cases judgment)
- Content moderation (policy defines hard bans; gray-area calls are judgment)
- Routing a ticket to the right team (rules narrow it; ambiguous tickets need judgment)

**Telltale signs it's mixed:**

- You can articulate rules that constrain the answer, but you also say "and then I decide between the remaining options."
- Two experts agree on most cases but disagree on edge cases.
- There's a clear right-wrong on the outer bounds, but a gray zone inside.
- You find yourself saying "it depends on context" but you can enumerate the contextual factors.

**Automation implications:**

- Use a mid-tier model (Sonnet or equivalent).
- The verifier is separate from the writer and reads the SSoT directly (dual-field pattern).
- SSoT needs `description` (for the writer) and `verification_criterion` (for the verifier).
- Bias-control includes the independence clause and batch-split at 10 for the verifier.
- One verifier is usually sufficient, but if edge cases are consequential, move to mixed+2 (two parallel verifiers with dissent → quarantine).

---

### Creative (interpretive, subjective, originality-bearing)

A creative step produces new content or interpretations where two experts might legitimately disagree even on the core answer. Quality depends on originality, cultural fit, brand voice, aesthetics, or interpretive judgment.

**Canonical examples:**

- Writing a marketing tagline for a product
- Generating an original quiz question with a fact-checked answer (Rangello's core task)
- Writing a news article from a set of facts
- Composing a press release
- Designing a logo or layout
- Picking a book title or chapter structure
- Writing a user-facing error message that matches brand voice
- Generating synthetic training data that needs to be diverse and realistic
- Writing original research summaries with editorial framing
- Building a course curriculum for a given topic

**Telltale signs it's creative:**

- You describe the judgment as "feel", "voice", "instinct", "taste", "style".
- Two experts produce different outputs, and both might be good.
- Originality matters — "correct but boring" is a failure.
- The output is read by humans and their subjective reaction is part of the quality metric.
- There's no single rule that captures the decision; there are patterns and heuristics that compound.

**Automation implications:**

- Use the strongest reasoning model available (Opus or equivalent).
- Apply the full bias-free verification architecture (see `bias-control-patterns.md` Group A).
- Use multi-agent consensus for any high-stakes correctness claim (dual-path fact verification, model heterogeneity).
- SSoT for rules should be dense with both writer-facing guidance and verifier-facing criteria, and enforced by tests.
- Apply regex-first gates for mechanical pattern violations (unit-in-text, filler phrases, source attribution).
- Quarantine on verifier dissent; never auto-fix controversial creative calls.
- Include a content-policy gate as its own stage (binary, no auto-fix).

---

## Decision checklist (ask these for each step during the interview)

Run through this list for every step the user describes. Record the answer.

1. **Can you describe the rule in two sentences?** If yes → logical. If "kind of, but there are exceptions" → mixed. If "it's more of a feel" → creative.

2. **Would two competent experts always produce the same answer?** Always → logical. Usually → mixed. Often disagree → creative.

3. **Is the answer verifiable against a fact, a spec, or a policy?** Yes → logical or mixed. Partially → mixed. Not really → creative.

4. **Does quality depend on originality, voice, aesthetic, or cultural fit?** If yes → creative.

5. **Has this step ever been automated with a simple rules engine?** If yes → logical (or mixed at most).

6. **If the automation produces a technically correct but dull / off-brand output, is that a failure?** If yes → creative.

7. **Can you imagine a regex, a schema validator, or a small function replacing your judgment here?** If yes → logical. If "a regex could catch part of it" → mixed. If "not really" → creative.

---

## Anti-patterns — wrong classifications that sink projects

### Calling a creative step logical

**Pattern.** The user says "oh, this is just pattern-matching, it's easy." You believe them, build a one-shot Haiku call with no verifier. The output is technically correct but nobody wants to read it, and the project fails on the soft metric (engagement, reputation).

**How to catch it.** Ask: "If two experts produced different answers here, could both be good?" If yes, the step is creative.

### Calling a logical step creative

**Pattern.** The user says "you can't really automate this, it requires intuition." You believe them, use Opus with three verifiers, and spend 10x the necessary compute on a step that a hard-coded function could do.

**How to catch it.** Ask: "What would you tell a junior hire to do here?" If the answer is a concrete rule, it's logical. Intuition often compresses rules the user hasn't written down — the interview's job is to expand them back out.

### Calling a mixed step logical (most dangerous)

**Pattern.** The user describes a rule-based-looking decision, you build it as logical, and the edge cases produce subtly wrong results. Worst of all worlds — looks right, is occasionally wrong, nobody notices until a customer complains.

**How to catch it.** Ask: "In the past year, has this step ever surprised you? Produced an answer you had to override?" If yes, it's at least mixed.

---

## Special case: judgment that compresses rules

Many "creative" steps the user describes turn out, on interview, to be compressed logical + mixed rules. The human has internalized the rules so thoroughly that they feel like intuition.

**Example from Rangello.** "I just know whether a question is interesting" → on interview, decomposed into: viral mechanism (A / B / C tiers), difficulty estimation (easy / normal / hard), bias-risk avoidance (no leading context phrases), factual verifiability.

The interview's job is to expand this compression. When the user says "it's just judgment," ask: "What are you actually looking at when you judge? Walk me through one recent example."

Often the outcome is that a step initially labeled `creative` gets split into:

- One genuinely creative core (the thing the human really does use taste for)
- Several mixed or logical subclauses the human had compressed into "judgment"

This split is almost always worth the effort — each subclause gets its right automation, and the remaining creative core gets targeted investment.

---

## When the user refuses to split

Sometimes the user insists "this is just one step, don't overcomplicate it." Respect the push-back, but record a note in the spec: `creativity_notes: "user insists this is a single step; likely contains compressed sub-rules; revisit on first failure"`. That way, when the automation inevitably fails on an edge case, the record is there to revisit the classification.

Never force a split the user rejects. But document the disagreement.

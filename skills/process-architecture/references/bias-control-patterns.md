# Bias-Control Patterns — 48 Principles

These are the 48 bias-control and verification patterns distilled from the Rangello question-generator pipeline. Every one of them is a pattern that has caught a real, non-trivial failure in a shipped AI system.

Use this as a **library**, not a recipe. For any creative step in your process, look through the groups below and pick the patterns that apply to your specific bias risks.

Each pattern is cited with the source file and line where it lives in Rangello, so you can read a concrete implementation if you want to see exactly how it looks.

---

## Group A — Verifier Architecture (5 core structural constraints)

These five constraints apply to *every* verifier in a bias-free pipeline. If a verifier violates any of them, it's not actually bias-free.

### 1. Fresh Subagent Dispatch

**Principle.** Every verifier runs as its own subagent with zero context inherited from the generation phase or from other verifiers. No inline coordinator calls with writer context still in scope.

**Why it prevents bias.** Coordinator-inline verifier calls carry writer-phase context. The verifier unconsciously frames its judgment based on what the writer was trying to achieve. Separate subagents start clean.

**Source.** `.claude/skills/question-generator/SKILL.md` line 31.

### 2. SSoT-Sourced Guidelines (no paraphrased copies)

**Principle.** The verifier reads its evaluation criteria *directly* from the authoritative JSON file at dispatch time. Never from a paraphrased copy embedded in the coordinator's prompt.

**Why it prevents bias.** Paraphrase drift between copies is the single most common silent-failure mode. The verifier sees stale rules, passes stale content, and nobody notices for weeks.

**Source.** `.claude/skills/question-generator/SKILL.md` line 32; `data/difficulties.json` with required `verification_criterion` field enforced by `src/lib/__tests__/metadata-consistency.test.ts`.

### 3. No Catalog Exposure

**Principle.** Verifiers never see the existing catalog, other items in the batch, or any generation metadata. They see only the item being evaluated and the rules.

**Why it prevents bias.** Catalog exposure produces peer-pressure averaging — the verifier unconsciously calibrates to "look similar to what we already have." Quality regresses to the mean instead of meeting the absolute bar.

**Source.** `.claude/skills/question-generator/SKILL.md` line 33 and 177.

### 4. Intra-Batch Independence Clause

**Principle.** Every verifier prompt contains an explicit instruction: *"Evaluate each item independently against the absolute definitions. Do NOT compare items to each other. Do NOT adjust your rating based on the distribution across this batch."*

**Why it prevents bias.** Without this, the verifier unconsciously normalizes across the batch. One strong item raises the bar for everything else; one weak item lowers it.

**Source.** `.claude/skills/question-generator/SKILL.md` line 34.

### 5. Batch-Split at 10

**Principle.** If a verifier batch has ≤10 items, a single subagent call with the independence clause is sufficient. If >10, split into chunks of up to 10 and dispatch parallel subagents.

**Why it prevents bias.** The independence clause alone is reliable up to roughly 10 items. Above that, even explicit instruction gets diluted by sheer accumulation — the context window saturates with examples and the instruction fades into the noise.

**Source.** `.claude/skills/question-generator/SKILL.md` line 35.

---

## Group B — Multi-Agent Consensus (fact verification across heterogeneous models)

### 6. Three Parallel Independent Verifiers

**Principle.** For high-stakes facts, dispatch three verifiers in parallel with zero cross-communication. Require unanimous `ok` or unanimous-identical `fix` to accept; any dissent triggers quarantine.

**Why it prevents bias.** A single verifier can be wrong. Two verifiers can agree on a wrong answer. Three independent verifiers with differentiated research prompts must each independently hallucinate the same wrong answer to slip through — far less likely.

**Source.** `.claude/skills/question-generator/SKILL.md` line 579; `.claude/agents/fact-verifier-primary.md`, `fact-verifier-depth.md`, `fact-verifier-synthesis.md`.

### 7. Model Heterogeneity Breaks Correlated Hallucinations

**Principle.** When using multiple verifiers, use multiple model tiers (e.g., 2× Haiku + 1× Sonnet) rather than three copies of the same model. Two agents of the same model share training distribution; their hallucinations correlate.

**Why it prevents bias.** Correlation across verifiers is the failure mode consensus is supposed to prevent. Mixing models breaks the correlation on the training-distribution axis.

**Source.** `.claude/skills/question-generator/SKILL.md` line 688.

### 8. Three-Source Minimum Rule

**Principle.** The depth verifier must find at least three independent sources before accepting a fact. A single confirming page is not enough, even if it's authoritative.

**Why it prevents bias.** Prevents single-source bias, where one well-cited aggregator page dominates the verifier's reasoning and any downstream verifier that happens to hit the same page.

**Source.** `.claude/agents/fact-verifier-depth.md` line 42.

### 9. Refutation-Search as Standard

**Principle.** Verifiers must actively search for "X is wrong", "X debunked", "recent correction to X". A claim that survives refutation-search is stronger than one that survives only confirmation-search.

**Why it prevents bias.** Classic confirmation bias — models naturally search for evidence that agrees with their initial guess. Forcing refutation-search produces asymmetric evidence.

**Source.** `.claude/agents/fact-verifier-depth.md` line 17.

### 10. Reputation Weighting, Not Popularity

**Principle.** The synthesis verifier weights evidence by source authority (peer-reviewed > official agency > single-source specialist > aggregator), NOT by how many pages repeat the same number.

**Why it prevents bias.** Ten aggregators copying the same stale number is not consensus. It's a single source with ten amplifiers. Authority weighting catches this.

**Source.** `.claude/agents/fact-verifier-synthesis.md` lines 17 and 43.

### 11. Per-Claim Consensus Matching

**Principle.** Decompose reveal / output text into atomic factual claims (year, person, place, magnitude). Compute per-claim agreement. Unanimous claim-level `ok` passes; claim-level dissent triggers quarantine of the whole item.

**Why it prevents bias.** Whole-text verdicts produce false-positive quarantines when three verifiers' rewrites differ stylistically on unchanged parts. Per-claim lets you fix one wrong year without throwing out the whole item.

**Source.** `.claude/skills/question-generator/SKILL.md` line 771.

### 12. Deterministic Integer Equality Check

**Principle.** When verifiers suggest a corrected numerical value, compare the integers deterministically (not via LLM). If the suggested values differ at all, it's dissent — quarantine.

**Why it prevents bias.** LLM-based "are these numbers basically the same?" introduces interpretation. Exact integer match is the one check that cannot be fudged.

**Source.** `.claude/skills/question-generator/SKILL.md` line 798.

---

## Group C — Single Source of Truth (SSoT)

### 13. Pure-Label SSoT

**Principle.** Metadata files that categorize content (e.g., `categories.json`) contain only labels — id, display name, visual attributes. No scope text, no examples, no avoid lists, no per-category prose.

**Why it prevents bias.** Every extra field is a bias magnet. Per-category prose freezes one curator's judgment into architecture. Pure labels let the LLM use its world knowledge without being steered.

**Source.** `data/categories.json`; enforced by `metadata-consistency.test.ts`.

### 14. Dual-Field Pattern: description + verification_criterion

**Principle.** Metadata files that drive both generation and verification carry two fields per entity: `description` (writer-facing — how to produce one) and `verification_criterion` (verifier-facing — how to check one).

**Why it prevents bias.** One source, two application-specific formulations. Prevents the verifier from being biased by the writer's framing, while keeping both in the same file so they can't drift independently.

**Source.** `data/difficulties.json`; `.claude/skills/question-generator/SKILL.md` line 916–919.

### 15. ID Parity Across Related Files

**Principle.** When two metadata files must agree on a set of identifiers (e.g., metric IDs in `metrics.json` must match keys in `unit-system.json`), enforce parity with a unit test.

**Why it prevents bias.** Silent ID drift is nearly invisible. One file gets a new entry, the other doesn't, and the pipeline fails in confusing ways weeks later. A test catches it at commit time.

**Source.** `src/lib/__tests__/metadata-consistency.test.ts`.

### 16. Verifier Criteria Hydration at Dispatch

**Principle.** The coordinator reads SSoT JSON once per run and embeds the content verbatim in the subagent prompt via a placeholder substitution. The substitution is code, not manual paste.

**Why it prevents bias.** Keeps the verifier in sync with the SSoT even when the file changes. No step where a human has to remember to update a paraphrase.

**Source.** `.claude/skills/question-generator/SKILL.md` lines 179–200.

### 17. SSoT Read Is Uniform Across Codebase

**Principle.** Every consumer of a metadata entity reads through the same helper (`src/lib/category-display.ts`). Components never read JSON directly.

**Why it prevents bias.** Centralizes the read pattern so changes propagate atomically. Prevents the case where three components each have their own hand-rolled parsing with slightly different fallback behavior.

**Source.** `src/lib/category-display.ts`.

### 18. Lightweight Entity Registry for Canonicalization

**Principle.** When you need to canonicalize variant names into a single key (e.g., "Paris", "Paris, France", "paris" → `paris`), keep a flat registry file and an LLM classifier that picks from it. Don't rely on string-similarity alone.

**Why it prevents bias.** Canonicalization by string-similarity has false positives (Paris vs. Paros) and false negatives (Mt. Everest vs. Sagarmāthā). A cheap LLM classifier with a registry fixes both.

**Source.** `data/clusters.json`; `.claude/skills/question-generator/SKILL.md` line 1079.

### 19. data-as-of as Writer Responsibility

**Principle.** For any time-sensitive data, the writer must set a `data_as_of: YYYY` field. The schema-sanity check validates the type; the fact verifier validates whether it's semantically appropriate.

**Why it prevents bias.** Time-sensitive facts silently go stale. Forcing the writer to record "when is this true as of?" makes the staleness explicit and auditable.

**Source.** `.claude/skills/question-generator/SKILL.md` line 520.

---

## Group D — Independence & Comparison Prevention

### 20. Writer Blindness to Catalog

**Principle.** Writer subagents never see the existing catalog, other in-flight items, or upstream selection logic. They get only the seed and the rules.

**Why it prevents bias.** Prevents subconscious topic-imitation and pattern reuse. A writer that has seen 50 bridge questions will produce bridge questions; a blind writer produces from the full space.

**Source.** `.claude/skills/question-generator/SKILL.md` line 177.

### 21. Priority Randomization (Metric Shuffle)

**Principle.** Where a writer chooses among alternatives, present the alternatives in a randomized priority order per item, not in a fixed list.

**Why it prevents bias.** Writers develop favorites over time ("I always reach for metric X"). Randomizing the priority order breaks the favorite-default.

**Source.** `.claude/skills/question-generator/SKILL.md` lines 40 and 261.

### 22. Small Batch Size (5, not 1 or 20)

**Principle.** Writer subagents work in batches of roughly 5 items. Not 1 (too much per-agent overhead) and not 20 (pattern-bias accumulates within the shared context).

**Why it prevents bias.** Each writer's context sees a small enough sample that it doesn't over-converge on a pattern, but large enough to amortize startup cost.

**Source.** `.claude/skills/question-generator/SKILL.md` line 163; `docs/architecture-journal.md`.

### 23. Two-Stage Open-Mode Classifier

**Principle.** For a fraction of generation passes, hide the category labels from the writer entirely. After generation, a separate classifier assigns (or rejects) one of the labels.

**Why it prevents bias.** One-stage generation with visible labels subtly steers topic choice toward label-matching. Two-stage preserves true category-blindness.

**Source.** `.claude/skills/question-generator/SKILL.md` lines 23 and 648–659.

### 24. Classifier List Built Dynamically from SSoT

**Principle.** The classifier's list of valid categories is built at runtime from `categories.json`, not hardcoded in the classifier prompt.

**Why it prevents bias.** Allows categories to change without classifier drift. If labels are added or renamed in the SSoT, the classifier picks them up automatically.

**Source.** `.claude/skills/question-generator/SKILL.md` line 52.

### 25. Intra-Batch Independence Clause in Every Verifier

**Principle.** Every verifier prompt — classifier, difficulty, linguistic, cluster, content-policy — repeats the same intra-batch independence clause. Not just the one where bias was first noticed.

**Why it prevents bias.** Assumes bias is a risk everywhere, not just where it's been caught historically.

**Source.** `.claude/skills/question-generator/SKILL.md` lines 34, 957–958, 1032, 1156.

---

## Group E — Regex-First-Then-LLM (two-phase cost-efficient gates)

### 26. Deterministic Detection, Cheap Correction

**Principle.** For patterns that can be found with regex (unit words in a question, source-attribution phrases, vague time references, filler phrases), use regex to *find* violations (free). Only dispatch an LLM call to *fix* the specific violation found.

**Why it's better than a full LLM gate.** Finding violations with regex costs zero tokens and is deterministic. Fixing with a small model (Haiku) is cheap. The combined stage is an order of magnitude cheaper than using an LLM to do both.

**Source.** `.claude/skills/question-generator/SKILL.md` lines 830–904; `scripts/lib/unit-in-text.js`, `source-attribution.js`, `vague-time.js`, `filler-phrase.js`.

### 27. Max 1 Fix Attempt Per Stage

**Principle.** After a single auto-fix attempt, if the violation persists, the item goes to quarantine. No second retry.

**Why it prevents bias.** Retry-loops tend to have the LLM fighting the regex with no semantic understanding of why. Two failures means the item has a deeper issue that needs human judgment.

**Source.** `.claude/skills/question-generator/SKILL.md` line 902.

### 28. Safety Re-Check After Auto-Fix

**Principle.** After any auto-fix, re-run the same detector to confirm the violation is gone. If the fix accidentally introduced a new violation, the item goes to quarantine.

**Why it prevents bias.** Auto-fixes can trade one bug for another. The re-check is cheap and catches these.

**Source.** `.claude/skills/question-generator/SKILL.md` line 893.

---

## Group F — Writer Blindness

### 29. English-Only Internal Artifacts

**Principle.** All internal IDs, cluster names, comments, commit messages, verifier prompts, and rule text are English, even when the user-facing product is multilingual.

**Why it prevents bias.** LLM reasoning differs across working languages. Mixed-language internal artifacts mean one part of the pipeline reasons in one language and another part in another. Pinning internal language to English isolates that variable.

**Source.** Project CLAUDE.md, "Language for internal artifacts" section; `docs/architecture-journal.md` lines 391–422.

### 30. Bilingual Metadata, Monolingual LLM Input

**Principle.** Metadata files carry both `name_en` and `name_de`. The LLM during research sees only `name_en`. The UI layer picks the locale-appropriate field.

**Why it prevents bias.** Keeps research bias-isolated to English (better sources, larger training corpus) while still supporting multilingual display.

**Source.** `data/categories.json`; `data/metrics.json`.

### 31. Generator Sees No Quality-Gate Definitions

**Principle.** The writer prompt contains instructions about how to write well, but not the exact verification criteria the downstream verifiers will apply. The two are related but separated.

**Why it prevents bias.** Prevents the writer from gaming the verifier. If the writer knows the verification regex, it can produce output that passes the regex without actually following the underlying intent.

**Source.** Dual-field pattern (`description` vs `verification_criterion`) in `data/difficulties.json`.

---

## Group G — Quality Gate Structure

### 32. Staged Quality Gate (many small specialized gates)

**Principle.** Instead of one monolithic quality check, build N specialized stages, each isolating one failure class. In Rangello: 12 stages (7a Schema, 7b Dedup, 7c Classifier, 7d Similarity, 7e Fact, 7f Regex, 7g Difficulty, 7h Linguistic, 7i Cluster, 7j Throttle, 7k Policy, 7l Originality).

**Why it prevents bias.** Specialized gates give specific feedback on specific failures. A monolithic gate fails with vague diagnostics and tempts workers to paper over unclear issues.

**Source.** `.claude/skills/question-generator/SKILL.md` Section 7.

### 33. Mechanical Before Expensive

**Principle.** Order the stages so deterministic / cheap checks run before expensive LLM calls. Schema-sanity and tokenless dedup go first; multi-agent fact verification goes later.

**Why it prevents bias.** Prevents wasting LLM calls on malformed items that would fail basic validation. Also means cost-intensive stages only see items that are at least structurally sound.

**Source.** `.claude/skills/question-generator/SKILL.md` Section 7a → 7l ordering.

### 34. Quarantine, Don't Discard

**Principle.** When an item fails a gate that cannot auto-fix (verifier dissent, regex-fix didn't work, policy violation), append it to a quarantine file with full context. Don't silently drop it.

**Why it prevents bias.** Silent drops hide failure modes. Quarantine queues force visibility and enable pattern analysis over time.

**Source.** `data/unfixable-questions.json`; `.claude/skills/question-generator/SKILL.md` line 790.

### 35. Content-Policy Is Binary, Never Auto-Fix

**Principle.** The content-policy verifier has only two outcomes: appropriate / inappropriate. There is no `fix` pathway. Policy violations go to human review.

**Why it prevents bias.** Auto-fixing policy violations trains the pipeline to produce borderline content confident that it will be laundered. Policy is a hard gate on purpose.

**Source.** `.claude/agents/content-policy-verifier.md`.

### 36. Hard-Reject Categories Are Absolute

**Principle.** Certain content classes (sexual content, grooming, self-harm instructions, hate speech, illegal-activity instructions) are unconditional rejects. "Factual framing" does not exempt them.

**Why it prevents bias.** Closes the "but it's just facts" loophole that pipelines tend to drift into.

**Source.** `.claude/agents/content-policy-verifier.md`.

---

## Group H — Language Asymmetry

### 37. Confidence-Threshold Asymmetry Where Owner Can't Self-Verify

**Principle.** Set a higher auto-fix confidence threshold for languages the project owner cannot independently verify. In Rangello: EN/DE auto-fix at 0.9, ES at 0.95.

**Why it prevents bias.** The project owner cannot catch errors in languages they don't speak. The pipeline must be strictly more conservative there.

**Source.** `data/linguistic-rules.json`; enforced by `metadata-consistency.test.ts`.

### 38. Per-Language Meta-Questions as SSoT

**Principle.** For each supported language, maintain a per-language checklist of meta-questions a verifier must answer ("does the sentence read natively?", "does pronoun reference resolve?", "is there idiom-level awkwardness?"). Keep them in the SSoT JSON.

**Why it prevents bias.** Languages have different failure modes (German gendered nouns, Spanish subjunctive, English comma rules). Per-language rules catch them.

**Source.** `data/linguistic-rules.json`; `.claude/skills/question-generator/SKILL.md` line 299.

---

## Group I — English-Only Identifier Hygiene

### 39. Internal IDs Are Stable, UI Labels Vary

**Principle.** Internal identifiers (metric IDs, cluster IDs, year-variant IDs) are English-only and rarely change. Display labels (name_de, name_en, name_es) are freely editable without migration.

**Why it prevents bias.** Allows UI text evolution (renaming, rewording) without touching every code path that uses the identifier. Separates presentation from architecture.

**Source.** `docs/architecture-journal.md` lines 391–422.

### 40. Symmetric, Culturally-Universal ID Naming

**Principle.** Pick identifier naming schemes that are symmetric and cross-cultural (bc / ad / future; not vc / normal / zukunft). Avoid IDs that lock in one language or cultural frame.

**Why it prevents bias.** Identifiers outlive their first users. Language-locked IDs become technical debt when the product internationalizes.

**Source.** Rangello year-variant rename from vc/normal/zukunft → bc/ad/future.

---

## Group J — Originality & Audit

### 41. Snippet-Based Originality Check

**Principle.** Use the source snippets collected during fact-verification as a reference corpus for phrasing-reuse detection. If the generated output is too close to a source snippet, flag it.

**Why it prevents bias.** Catches accidental verbatim-copying that gradient descent can't distinguish from paraphrase. Uses assets you're already collecting.

**Source.** `.claude/skills/question-generator/SKILL.md` line 1339 (planned).

### 42. Audit-Only Geographic-Balance Field

**Principle.** Add an opt-in audit field (e.g., `primary_country`) that lets you run periodic balance reports over the catalog. Not enforced as a gate — just measured.

**Why it prevents bias.** Spot regional or cultural skew in the catalog over time, without forcing a quota that would itself introduce bias.

**Source.** `.claude/skills/question-generator/SKILL.md` lines 1559–1568.

---

## Group K — Workflow & Orchestration

### 43. Preflight Scripts as Read-Only Input Constraints

**Principle.** Pre-pipeline scripts (`promote-throttled.js`, `backfill-clusters.js`) run as isolated read-only transformers. The main pipeline reads their artifacts as inputs. No circular writes.

**Why it prevents bias.** Keeps the main pipeline's state-writing surface small and predictable. Preflight is repeatable and idempotent.

**Source.** `scripts/promote-throttled.js`; `.claude/skills/question-generator/SKILL.md` Step 2b.

### 44. Schema-Tolerant Readers

**Principle.** Scripts that read queue / staging files accept multiple schema variants, because hand-edited and coordinator-produced files may differ in field names.

**Why it prevents bias.** Real-world queues are messy. A strict reader fails closed; a tolerant reader handles the reality.

**Source.** `scripts/lib/throttle-entry.js`.

### 45. Local Snapshot for Deterministic Runs

**Principle.** Before a generation run, take a local snapshot of the catalog (via SQL dump to `catalog-dedup.json`). The rest of the run reads the snapshot, not the live DB.

**Why it prevents bias.** Live-DB reads produce non-reproducible runs. A snapshot makes the run deterministic given the same input.

**Source.** `.claude/skills/question-generator/SKILL.md` Step 2.

### 46. Explicit Data-Integrity Rules Embedded in Writer Prompt

**Principle.** The writer prompt includes a numbered list of data-integrity rules (ganze/dezimal precision, reveal-facts cross-check, unit choice, order-of-magnitude sanity). Not general advice — numbered, referenceable rules.

**Why it prevents bias.** General "be careful" instructions don't work. Numbered rules give the model something concrete to self-check against.

**Source.** Writer prompt data-integrity section in `.claude/skills/question-generator/SKILL.md`.

---

## Group L — Miscellaneous

### 47. Two-Phase Writer Prompt (generation → normalization, same agent)

**Principle.** When a writer needs to produce creative content AND normalize output format (units, schema), do it in two phases within the same agent: phase 1 generates creatively, phase 2 normalizes. Don't split across two agents — the setup cost doesn't pay off.

**Why it prevents bias.** Phase 1 must be "blind" to the normalization rules (otherwise the unit choice biases the creative content). Doing both in one agent sequentially gives you the blindness property without the two-agent overhead.

**Source.** `.claude/skills/question-generator/SKILL.md` "Two-phase writer prompt" section.

### 48. Throttle Gate With Soft-Reject Queue and Auto-Promotion

**Principle.** Items that pass quality but fail a throttle (too many similar items in the catalog) go to a soft-reject queue. A separate job periodically re-evaluates the queue as the catalog grows, auto-promoting items that now fit.

**Why it prevents bias.** Prevents permanent loss of well-crafted items that were merely unlucky in timing. Also keeps topic diversity over time without forcing quotas up-front.

**Source.** `data/throttled-questions.json`; `scripts/promote-throttled.js`.

---

## How to use this catalog in practice

For each `creative` or `mixed` step in your `architecture-spec.yaml`:

1. **List the specific bias risks** that step faces. Typical: pattern-accumulation, peer-pressure, paraphrase drift, confirmation, favorite-default.
2. **Pick the patterns from this catalog** that neutralize those risks. One step may need several patterns.
3. **Record the choice in the spec** under `creative_components[i].bias_risks[]`, naming the pattern explicitly.
4. **When `process-automation-skill` builds the automation**, it will use your named patterns to structure the agent, the verifier, the SSoT, and the gates.

Do not copy every pattern into every step. That's cargo-culting, and it adds cost without adding protection. The goal is **matched protection** — the specific bias risks for this step, neutralized by the specific patterns that address them.

---

## Separately: purely logical (non-bias) quality principles

These aren't bias-control, but they showed up repeatedly in the Rangello architecture and belong in the same family of disciplined patterns:

- **Deterministic schema validation before expensive LLM calls** (Stage 7a before 7e).
- **Atomic claim decomposition** before per-claim verification (enables targeted fixes).
- **Reputation-weighted > popularity-weighted consensus** (logical-correctness principle, not just bias).
- **Metric shuffle** (removes artificial topic/metric pairings that feel forced).

These read as "sensible engineering," not as bias-control. But they're the scaffolding that makes bias-control work, and they're listed here so `process-automation-skill` can reach for them without re-deriving.

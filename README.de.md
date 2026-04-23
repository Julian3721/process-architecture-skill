# process-architecture-skill

Ein interview-getriebenes [Claude-Code](https://claude.com/claude-code)-Skill, das dir hilft, jeden Prozess lückenlos als Architektur zu modellieren — bereit zur Übergabe an eine KI-gestützte Automatisierung.

> Read in English → [README.md](./README.md)

---

## Worum geht's

Ein spezialisiertes Skill, das dich durch ein strukturiertes Interview über deinen Prozess führt (einen aktuellen Job, eine Geschäftsidee, einen wiederkehrenden Workflow — alles möglich) und zwei Artefakte produziert:

1. **`architecture-spec.yaml`** — eine strukturierte, validierte Spezifikation jedes Schritts, jeder Abhängigkeit, jedes Inputs, jeder Rückkopplung, jedes Quality-Gates und jedes Bias-Risikos deines Prozesses.
2. **`architecture.mmd`** — ein Mermaid-Diagramm, direkt renderbar in GitHub und den meisten IDEs.

Das Skill **sammelt nicht einfach**, was du sagst. Es hinterfragt vage Antworten, benennt erkannte Lücken offen und lässt dich nicht weitergehen, bis die Architektur vollständig ist. Der Output ist so präzise, dass die anschließende Umsetzung mechanisch wird.

---

## Warum es das gibt

Die meisten KI-Automatisierungsprojekte scheitern nicht an der Umsetzung, sondern an der Architektur. Teams springen direkt zum "lass uns ein LLM drauf werfen", ohne vorher zu klären:

- Was der Prozess **tatsächlich** produziert (ein konkretes Lieferobjekt)
- Woher jeder Input **wirklich** kommt (nicht "halt die Daten")
- Wie die echten Qualitätskriterien lauten (besonders die impliziten, bei denen "ich merke das schon")
- Welche Schritte logisch sind (leicht zu automatisieren) und welche kreativ (brauchen Bias-Kontrolle)
- Welche stillen Rückkopplungen existieren
- Welche Metadaten-Entitäten als Single Source of Truth gepflegt werden müssen

Ohne diese Antworten bekommst du eine Automatisierung, die im Demo glänzt und im Produktiv-Einsatz versagt.

Dieses Skill ist die destillierte Methodik aus einem ausgelieferten Projekt — dem **Rangello** Daily-Estimation-Spiel — bei dem ein Quizshow-Fragenschreiber-Job vollständig durch eine 12-stufige KI-Pipeline mit bias-freier Verifikation, Multi-Agent-Konsens und Single-Source-of-Truth-Metadaten ersetzt wurde. Die dort entwickelten Bias-Kontroll- und Architektur-Muster sind jetzt in [`skills/process-architecture/references/bias-control-patterns.md`](./skills/process-architecture/references/bias-control-patterns.md) katalogisiert — 48 Prinzipien, jedes mit dem Rangello-Quellcode-Verweis.

---

## Das Zwei-Skill-System

Dieses Skill ist eine Hälfte eines Paares.

| Skill | Zweck |
|---|---|
| **`process-architecture-skill`** (dieses Repo) | Die komplette Architektur entwerfen. Interview-getrieben. Output: `architecture-spec.yaml` + `architecture.mmd`. |
| **[`process-automation-skill`](https://github.com/Julian3721/process-automation-skill)** | Die fertige Spezifikation nehmen und die tatsächliche Agenten-Pipeline bauen. Modell-Auswahl pro Schritt, SSoT-Files, Verifier-Agenten, Quality-Gates. |

`process-architecture-skill` läuft immer zuerst. Wenn es die Architektur als vollständig erklärt, wechselst du mit der Spec-Datei zu `process-automation-skill`.

---

## Installation

### Variante 1 — als Projekt-Skill

Verschiebe das `skills/process-architecture/`-Verzeichnis in den `.claude/skills/`-Ordner deines Projekts:

```bash
git clone https://github.com/Julian3721/process-architecture-skill.git
cp -r process-architecture-skill/skills/process-architecture /pfad/zu/deinem/projekt/.claude/skills/
```

Claude Code findet das Skill beim nächsten Start automatisch.

### Variante 2 — als User-Skill global

Für alle deine Projekte verfügbar:

```bash
mkdir -p ~/.claude/skills
cp -r process-architecture-skill/skills/process-architecture ~/.claude/skills/
```

### Variante 3 — als Plugin

Wenn du das Plugin-System von Claude Code nutzt: Repo als lokale Plugin-Quelle hinzufügen und über den Plugin-Marketplace installieren.

---

## Nutzung

Sobald installiert, rufst du das Skill mit einer beliebigen natürlichen Formulierung auf — der Trigger ist bewusst breit:

- `"ich will meinen Job automatisieren, hilf mir beim Entwurf"`
- `"Architektur für meinen Workflow entwerfen"`
- `"prozess automatisieren"`
- `"architect this process for me"` (englisch)

Das Skill stellt sich vor, fragt nach dem Prozess-Typ (logisch vs kreativ) und führt dich durch 7 Interview-Abschnitte:

1. **Ziel & Output** — was produziert der Prozess am Ende?
2. **Inputs & Trigger** — was fließt rein, was startet ihn?
3. **Schritte & Reihenfolge** — jede Entscheidung und Transformation, in Sequenz
4. **Abhängigkeiten & Rückkopplungen** — wie Schritte einander beeinflussen, inklusive Loops
5. **Zeithorizont** — Latenz-Budgets, Deadlines, synchron vs asynchron
6. **Qualitätskriterien** — wie erkennen wir gute Outputs, welche Gates fangen Fehler ab?
7. **Kreativ-Anteile** — für jeden Schritt mit Urteilsbildung: welche Bias-Risiken gelten, wie neutralisieren wir sie?

Rechne mit **30 bis 90 Minuten** hin und her. Das ist kein Bug. Jede Lücke, die hier geschlossen wird, spart 10× Aufwand in der Umsetzung.

---

## Kernkonzept: logisch vs kreativ

Die wichtigste Unterscheidung, die das Skill durchzieht:

| Typ | Beispiele | Automatisierungsansatz |
|---|---|---|
| **Logisch** | Rechnungsarithmetik, Daten-Cleanup, Compliance-Checks, FAQ-Routing | Billiges Modell, minimale Verifikation, einfacher SSoT |
| **Mixed** | Severity-Klassifizierung, Ticket-Routing mit Kontext, Content-Moderation-Graubereich | Mid-Tier-Modell, dual-field SSoT, ein Verifier mit Independence-Clause |
| **Kreativ** | Originaltexte, Curriculum-Design, Tagline-Generierung, faktengeprüfte Original-Fragen | Starkes Reasoning-Modell, Multi-Agent-Konsens, volle bias-freie Verifier-Architektur, Quarantäne statt Retry |

Siehe [`skills/process-architecture/references/logical-vs-creative.md`](./skills/process-architecture/references/logical-vs-creative.md) für das vollständige Klassifizierungs-Framework inkl. der typischen Fallen (kreativen Schritt als logisch einstufen, mixed als logisch, usw.).

---

## Die Bias-Kontroll-Herkunft

Kreative Schritte haben vorhersehbare Bias-Risiken, wenn man sie automatisiert:

- **Pattern-Akkumulation** — der Agent wiederholt Formulierungen über einen Batch
- **Peer-Pressure-Averaging** — Verifier-Bewertungen normalisieren sich über den Batch
- **Paraphrasen-Drift** — Regeln, die zwischen Dateien kopiert werden, driften still auseinander
- **Bestätigungs-Bias** — Verifier sucht nur nach zustimmenden Quellen
- **Favorite-Default** — Agent tendiert zu einfacheren oder vertrauteren Themen
- **Sprach-Bias-Leakage** — Agent-Reasoning verschiebt sich je nach Arbeitssprache

Das Rangello-Projekt hat jedes davon erkannt und mit konkreten Mustern neutralisiert (fresh-subagent-dispatch, SSoT-gespeiste Guidelines, intra-batch Independence-Clauses, batch-split at 10, Modell-Heterogenität, Refutation-Search, Reputation-Weighting, per-claim Konsens, English-only interne Artefakte, und mehr).

Alle 48 Prinzipien sind in [`skills/process-architecture/references/bias-control-patterns.md`](./skills/process-architecture/references/bias-control-patterns.md) katalogisiert, gruppiert in 12 Familien, jedes mit der exakten Rangello-Quellcode-Referenz, damit du das Muster in einem laufenden System nachlesen kannst.

---

## Was das Skill NICHT macht

- **Es lässt dich keine Sektion überspringen.** Jede Sektion hat mindestens eine "do-not-proceed"-Bedingung. Wenn deine Inputs keine konkreten Quellen haben, geht das Skill nicht weiter, bis sie benannt sind.
- **Es vereinfacht nicht heimlich.** Wenn du einen Schritt mit 7 Verzweigungen beschreibst, werden 7 Verzweigungen modelliert. Kompression versteckt Bias.
- **Es designt nicht die Implementierung.** Das ist Aufgabe des Automation-Skills. Dieses Skill verweigert Modell-Wahl, Prompt-Stile, Pipeline-Architektur — es produziert eine Spec und hört auf.
- **Es toleriert kein "Ich merke das schon."** Wenn der Nutzer seinen Qualitäts-Check als intuitiv beschreibt, zerlegt das Skill die Intuition in konkrete Signale. Das ist das Wertvollste, was dieses Skill macht.

---

## Grenzen und ehrliche Einschränkungen

- Prozesse, die stark auf emotionale Abstimmung, langfristige menschliche Beziehungen oder Live-Sensorik basieren, sind **heute** keine guten Automatisierungs-Ziele. Das Skill kann sie trotzdem kartieren, aber du solltest erwarten, dass das Automation-Skill die Bias-Risk-Oberfläche als unbegrenzt markiert.
- Das Skill nimmt an, dass du deinen eigenen Prozess artikulieren kannst. Wenn nicht, brauchst du erst einen Prozess-Experten, dann dieses Skill.
- Die Methodik basiert auf der Rangello-Projektgröße (hunderte Fragen, Dutzende Kategorien, 3 Sprachen). Für ganz andere Skalierungen (Millionen Items/Tag, stark regulierte Domänen) müssen manche Muster erweitert werden.
- Das Skill hat eine Meinung. Es hinterfragt "haben wir schon immer so gemacht", wenn die Antwort eine ungenannte Lücke aufdeckt. Dieser Widerstand ist der Punkt.

---

## Mitarbeit

PRs willkommen. Besonders wertvoll:

- Zusätzliche Domänen-Beispiele in `references/logical-vs-creative.md` (neue Branchen, gegen die das Klassifizierungs-Framework nicht getestet wurde)
- Neue Bias-Kontroll-Muster, die du entdeckt hast und mit Quellcode belegen kannst
- Übersetzungen der Interview-Rhetorik in weitere Sprachen (aktuell EN + DE)
- Reale Architektur-Specs, die du mit diesem Skill produziert hast (als Fallstudien-Beispiele)

Bitte vor größeren Änderungen ein Issue öffnen, damit wir den Fit abstimmen.

---

## Lizenz

MIT. Siehe [LICENSE](./LICENSE).

---

## Schwester-Skill

[`process-automation-skill`](https://github.com/Julian3721/process-automation-skill) — die zweite Hälfte des Paares. Nimmt die hier produzierte Spec und baut die tatsächliche Agenten-Pipeline.

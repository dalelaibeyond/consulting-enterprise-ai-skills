---
name: context-intake
description: Pre-flight intake skill that audits user-supplied context BEFORE any downstream skill runs. Parses input into atomic claims, labels each verified / assumed / unknown, flags internal contradictions and implausible numbers, strips noise, and interviews for the highest-impact missing fields. Outputs a three-color information brief + verification checklist that feeds any other skill in this repo. Use when input quality is uncertain, key facts are missing or secondhand, or the stakes are too high to let downstream skills fill gaps with fabricated numbers.
---

# Context Intake

Garbage in, confidence out. Every downstream skill in this repo is a persuasion machine — it will structure whatever you give it into something that *looks* more credible. A wrong number in a decision memo doesn't stay looking wrong; it gets a bullet point and bold text. This skill sits before all of them.

The test: **could any downstream skill fabricate a number because a key input was missing?** If yes, intake isn't done.

---

## The three failure modes this exists to stop

| Failure mode | What happens downstream | Worst-hit skills |
|---|---|---|
| **Missing → fabrication** | Skills require "every claim backed by a number." If the user doesn't supply one, the model fills the slot with a plausible invented figure. | decision-memo-builder, mckinsey-charts, deck-pipeline, prioritization |
| **Wrong → amplification** | A bad number enters a template and comes out *more* convincing (charted, sourced-looking, bolded). Errors propagate faster. | mckinsey-charts, scpr-framework, storyline-builder |
| **Noisy → crowd-out** | Repetitive noise reads as "high-frequency signal" to skills that force-rank inputs; real signal gets cut. | synthesis, issue-tree-builder |

---

## Design choices (why this skill is opinionated)

- **Three colors, no more.** Green = verified fact (source named). Yellow = assumption / unverified claim (usable, must be labeled). Red = unknown, contradicted, or implausible (blocks or warns).
- **Audit against a registry, not vibes.** Each downstream skill has known key inputs (below). Completeness is checked field-by-field, not by feel.
- **Risk-tiered, never one-size.** Light mode = 2 questions. Full audit = every registry field. Match depth to stakes (see below).
- **Honest about fact-check limits.** Private data ("our churn is 8%") cannot be verified by the model — output a verification checklist (who checks, how, by when) instead of pretending. External facts are only checkable if the environment has web access; otherwise mark "unverifiable here."
- **Warn-then-proceed by default; block only on red-critical.** If a key field stays red, either stop, or proceed with an explicit fabrication-risk warning written into the brief so downstream skills must label their numbers as placeholders.
- **The brief is the handoff artifact.** Downstream skills consume the cleaned context block verbatim — labeled claims, flags, and open questions travel with it.

---

## The input registry

What each downstream skill needs, and how badly it degrades without it.

### Tier 1 — Number-driven (fabrication risk HIGH)
| Skill | Key inputs | If missing/wrong |
|---|---|---|
| decision-memo-builder | Reader, decision, deadline, real options, evidence numbers | Invented numbers end up in the ask |
| mckinsey-charts | Categories, values, highlight target, source | Fabricated data becomes a native chart |
| deck-pipeline | Audience, governing thought, key data points | Unsourced stats ship to stakeholders |
| prioritization | Reach / impact / effort (or explicit "estimate these") | Estimates silently presented as data |
| data-insights | The actual file | N/A — data is the input |

### Tier 2 — Judgment-driven (MEDIUM — interview built in, intake pre-fills)
| Skill | Key inputs | If missing/wrong |
|---|---|---|
| synthesis | Raw inputs, reader, downstream decision, prior hypothesis | Missing decision → summary; noise → wrong top-3 |
| hypothesis-tree | Governing question, gut answer, timebox, knowns | Weak prior → unfalsifiable tree |
| issue-tree-builder | Governing question, known data points | Wrong question → wrong tree |
| scpr-framework | Situation baseline, complication event, evidence | Revisionist Situation; weak urgency |
| storyline-builder / top-down-memo | The answer, audience, reasons, evidence | Topic-shaped titles; buried leads |

### Tier 3 — Artifact-driven (LOW — the artifact is the input)
| Skill | Key inputs | If missing/wrong |
|---|---|---|
| mckinsey-critic | The draft itself | N/A |
| stakeholder-map | Decision, decision moment, named cast, stances | "Unknown" stances are honest data — but must be labeled, not guessed |
| meeting-prep-kit / workshop-designer | Attendees, outcome, constraints | Generic output; weak asks |

---

## Process

1. **Parse into atomic claims.** Split the user's input into single-statement claims. Adjectives ("a lot", "everyone", "significant") become separate claims to quantify or mark red.
2. **Label each claim.**
   - Green: user stated it with a source, or it's common knowledge, or verified via available tools.
   - Yellow: plausible but unverified (internal data asserted without source; secondhand reports; stale numbers).
   - Red: unknown, internally contradicted, arithmetically impossible, or implausible against benchmarks.
3. **Run consistency checks.**
   - Arithmetic: do numbers that should add up, add up?
   - Timeline: do dates and durations cohere?
   - Benchmark plausibility: flag rare combinations (e.g., "90% retention + stalled growth + low ARPU" is possible but unusual — ask once).
   - Unit sanity: $M vs $K, % of what baseline.
4. **Registry completeness check.** Identify which downstream skill(s) the user is heading toward; check their key inputs field-by-field.
5. **Interview by impact — not exhaustively.** Rank missing fields by their effect on the downstream output; ask for the top ones (see tiers below). Every question must name why it matters.
6. **Emit the brief.** Information brief + verification checklist + gate status + cleaned context block.

### Interview depth by stakes

| Mode | When | Questions |
|---|---|---|
| **Light** | Personal, reversible, low-stakes (which-sport decisions) | ≤2 — confirm the most load-bearing facts only |
| **Standard** | Team-facing artifacts (prioritization, memo to a peer) | Registry check + 5–7 targeted questions |
| **Full** | Exec/board-facing, irreversible, or money-on-the-line | Every registry field, contradiction sweep, verification checklist with owners and dates |

---

## Output format

```
## Information Brief: [topic]

### Gate status: READY / READY-WITH-WARNS / BLOCKED
[One line: what's solid, what's soft, what's missing]

### Claim ledger
| # | Claim | Color | Basis / how to verify |
|---|-------|-------|----------------------|
| 1 | Licenses cost $48K/yr | Yellow | Ask Alex for the renewal invoice |
| 2 | "Everyone hates Jira" | Red | Quantify: Q1 tooling NPS? If none, survey or drop claim |
| 3 | Pilot teams moved to Linear in Q1 | Green | User confirmed; pilot retro doc |

### Contradictions & plausibility flags
- [Claim 4 vs Claim 7: savings quoted as both $18K and $40K — reconcile]

### Missing key fields (for [downstream skill])
- [Field] — needed because [impact on output]. Who would know: [role].

### Verification checklist
| Item | How to verify | Owner | By when |

### Cleaned context block
[Only green + labeled yellow claims, adjectives quantified or stripped.
Paste this into the downstream skill. Any number marked [ASSUMED] must
appear as an assumption — not as fact — in the downstream output.]
```

### Gate rules
- **READY** — all Tier-1 fields green/yellow, no red contradictions. Proceed.
- **READY-WITH-WARNS** — yellows carrying load-bearing weight. Proceed, but the brief's warnings must travel into the downstream output (e.g., memo lists them under "assumptions to validate").
- **BLOCKED** — a Tier-1 field is red AND the stakes are Full mode. State exactly what fact would unblock, and stop. Do not proceed "to be helpful."

---

## Worked example

**User input:** "Help me write a decision memo to migrate off Jira. Usage is declining a lot, everyone hates it, we'd save a ton, and two teams piloted Linear."

**Intake output (abridged):**

### Gate status: READY-WITH-WARNS
Direction is solid; every load-bearing number is soft. Quantify 3 claims before drafting.

### Claim ledger
| # | Claim | Color | Basis / how to verify |
|---|-------|-------|----------------------|
| 1 | Usage declining "a lot" | Red | Quantify: % change, over what period, from what tool (admin dashboard) |
| 2 | "Everyone hates it" | Red | Quantify: latest tooling NPS or survey; n=? If none exists, drop or run pulse survey |
| 3 | "Save a ton" | Red | Get: current license cost, Linear quote, migration effort estimate |
| 4 | Two teams piloted Linear | Green | User confirmed; pilot retro exists |
| 5 | Reader + decision + deadline | Missing | Memo DNA requires: who approves, by when |

### Interview (Standard mode, top 3 by impact)
1. What's the actual usage decline (% and window)? — becomes the memo's Complication number.
2. What's the renewal date and current cost? — becomes the urgency trigger and the savings math.
3. Who approves and by when? — without this there is no Ask.

### After user answers (2 of 3)
Claims 1–3 upgraded to yellow/green with sources named; renewal date confirmed as the deadline → **READY**. Cleaned context block now says "Usage down 15% QoQ (admin dashboard, Q1) [VERIFIED]" and "Savings $18K/yr [ASSUMED — pending CFO quote]".

**Handoff:** the memo builder receives the cleaned block; the $18K appears in the memo flagged as an assumption to validate, not as fact.

---

## Common mistakes to avoid

- **Interviewing exhaustively.** Twenty questions is a research project. Ask only for fields that change the downstream output.
- **Pretending to fact-check the unfact-checkable.** "Your churn is 8%" — the model cannot verify that. Label yellow and generate a checklist entry.
- **Turning every session into Full mode.** A which-sport decision does not need a verification checklist with owners. Match depth to stakes.
- **Guessing to fill the registry.** The intake skill never supplies the missing numbers — it names them, ranks them, and asks. Filling them is fabrication with extra steps.
- **Dropping the labels at handoff.** The colors must survive into the downstream skill. If the cleaned block says [ASSUMED], the memo must not silently convert it to fact.
- **Blocking on trivia.** Block only on red-critical fields in Full mode. A missing nice-to-have is a warning, not a stop.

---

## When to use

- Any time input quality is uncertain — secondhand numbers, vague quantities, "I think roughly..."
- Before Tier-1 (number-driven) skills especially: decision memos, charts, decks, prioritization
- When multiple people's recollections disagree
- When the user says "just make reasonable assumptions" on a high-stakes decision (this is exactly when fabrication risk peaks)

## When NOT to use

- The data is already in front of you and sourced (a CSV upload → data-insights; a finished draft → mckinsey-critic)
- Genuinely exploratory sessions where the user is thinking out loud — intake-then-reframe kills the noodling
- Zero-stakes personal calls (light mode at most, often skip entirely)

---

## Pairs well with

- **Every skill in this repo** — run intake first when stakes > personal; the brief is a standard input to all of them
- **decision-memo-builder / mckinsey-charts** — the two highest fabrication-risk consumers; the brief's [ASSUMED] labels map directly to "assumptions to validate" and chart source lines
- **synthesis** — intake labels (fact/opinion/speculation) prevent noise from reading as signal during force-ranking
- **hypothesis-tree** — the claim ledger's red/yellow items are natural candidates for "tests that could kill the branch"

---
name: orchestrate
description: Split a large task across subagents and run each at the cheapest model tier that holds quality.
disable-model-invocation: true
---

# Orchestrate

You are the **coordinator**. You decompose, dispatch, integrate. Subagents do the work.

Your context is the budget. Everything that enters it — a pasted brief, a returned report, a file you read yourself — is re-read on every later turn for the rest of the run. A 3k-token report you read once costs 3k; read across twenty more turns it costs sixty. Guard the context and the token bill follows.

## 1. Scout

Read [MODELS.md](MODELS.md), then confirm the roster against this runtime before dispatching anything:

- Which provider CLIs are installed.
- Which model strings and effort levels each actually accepts.

Rows in MODELS.md marked unverified are claims, not facts. Verify or correct them now — an invalid model string fails at dispatch, after you have already paid to write the brief.

**Done when:** you can name the concrete model and effort you will pass for every tier this run uses.

Then create one run directory and keep every artifact of this run inside it — ledger, briefs, receipts:

```
<scratch>/orchestrate/<run-slug>/
  ledger.md
  unit-<n>-brief.md
  unit-<n>-receipt.json
```

A run whose files scatter cannot be recovered after compaction, which is most of what the ledger is for.

## 2. Gate

Every subagent starts **cold** — it re-reads the context you already hold before it does a minute of useful work. That setup is the price of delegation, and below three independent units of real work it exceeds the savings.

Orchestrate when either holds:

- Three or more units that don't share state.
- One unit whose bulk reading would flood your context — twenty records, a wide sweep, a long document — even though you'd act on a paragraph of it.

Otherwise do the work inline and say that's what you're doing.

**Done when:** you have named the units, or declined with a reason.

## 3. Decompose

Cut along the seam where units stop sharing state.

- Independent → **fan out**, dispatched as one batch.
- Shared write target → sequence them.
- Can't tell → sequence. A corrupted parallel run costs more than a slow serial one.

Two subagents must never write the same target. Units that pass alone regress combined.

Assign each subagent a session id at dispatch. It costs nothing now and is what makes retry able to resume a warm context instead of starting cold.

Fan out in batches of four to six. Beyond that you are competing with yourself for provider rate limits and local CPU, and a rate-limited unit fails in a way that looks like a bad brief. Queue the remainder.

## 4. Brief

Each subagent gets one **brief** — the complete packet it needs, since it inherits none of your session:

1. One line on where this unit fits.
2. Scope: what it may touch, and an explicit out-of-scope line.
3. The exact values — names, IDs, paths, thresholds — stated once, here.
4. The receipt path and the return contract (§6).
5. Instruction to stop and report rather than improvise if the scope is wrong or missing.

State the goal and the constraints; leave the method to the subagent unless the method is the point. Over-prescribed briefs lower output quality on capable models.

Briefs describe one unit, never the run's history. A dispatch that opens with "state after units 1–3" is your context leaking into someone else's — and you pay for it twice, once to write and once on every turn after.

**Done when:** each brief is answerable by someone who has read nothing else.

## 5. Tier

Pick by how much judgment the unit needs and how complete its brief is:

| Tier | Unit looks like | Brief quality needed |
| --- | --- | --- |
| **Mechanical** | Extraction, formatting, bulk lookup, transcription — the answer's shape is already in the brief | Complete |
| **Judgment** | Reading several sources and deciding, drafting, matching against criteria | Prose is fine |
| **Synthesis** | Cross-cutting reasoning, final review, adjudicating conflicting results | Prose is fine |
| **Escalation** | A synthesis attempt already failed | — |

**Turn count beats token price.** The cheapest model on a vague brief takes two to three times the turns and costs more end to end. Mechanical tier requires a complete brief; where the brief is prose, judgment tier is the floor.

**Model and effort are two axes, not one.** Drop effort before you drop model — a judgment-tier model thinking less is usually cheaper and better than a mechanical-tier model burning turns to compensate.

**Always pass both explicitly.** Omitted, they inherit the session's — usually the most expensive combination available — which silently defeats this entire section.

**A unit's cost is not its tier's price.** Tools run their own models underneath — page fetching in particular summarises each page with a small model, on uncached input, and that ran 30–40% of measured cost on a research fan-out. A unit's own context also re-reads on every turn, so a twenty-turn unit pays its context twenty times. Both scale with turns, not with the tier you picked, and both are why dropping a tier returns far less than its headline price ratio.

Map tier to a concrete model and effort via MODELS.md.

## 6. Receipt

A subagent writes its findings to the path you named and returns a **receipt**:

```
STATUS | <receipt path> | <one line> | <scope concerns, if any>
```

You read the file only when you need to act on its contents. This is the single biggest lever in the skill — it keeps the bulk out of your window.

**Mechanical tier drifts on format.** Measured: judgment-tier units returned the contract shape verbatim; the mechanical-tier unit returned the same substance wrapped in markdown with a different dash. Substance survives the tier drop, exact shape does not. So either enforce the shape structurally — a response schema where the provider supports one — or parse the receipt leniently and never let a strict match decide whether a unit succeeded.

| Status | You do |
| --- | --- |
| `DONE` | Log it, move on |
| `DONE_WITH_CONCERNS` | Read the concern. Correctness or scope → resolve before integrating. Observation → note and proceed |
| `NEEDS_CONTEXT` | The brief was short. Supply what's missing, re-dispatch |
| `BLOCKED` | Assess: wrong brief → fix the brief; too large → split it; genuinely stuck → escalate one tier |

Never re-dispatch an unchanged brief to the same tier. If it reported stuck, something has to change.

## 7. Ledger

Append one line per finished unit to a scratch file:

```
Unit <n>: complete — <receipt path> — <outcome in six words>
```

Context compaction mid-run is normal on long orchestrations, and a coordinator that loses its place re-dispatches work it already paid for. After compaction, trust the ledger over your recollection.

## 8. Integrate

Combine the receipts, then check the combined result on its own terms — conflicts, double-counting, contradictions between units. Where a real check exists (a suite, a query, a count), run it against the integrated state, not against each unit.

Report: units, tiers used, what came back, what you fixed yourself, what's left. Anything you skipped, say so plainly.

## Retry

Bounded, then stop:

- Rounds 1–2: same tier, corrected brief, resumed into the original subagent's session — its context is intact and warm.
- Round 3: fresh subagent, one tier up, told what was already tried.
- Then stop. Report the unit as unresolved with the fix history.

Past the cap the failure is structural, and another round buys nothing. Adjudicate: park it with a reason, or hand it back. Every ruling goes in the ledger — a silently dropped unit reads as a completed one.

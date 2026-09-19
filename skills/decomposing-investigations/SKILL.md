---
name: decomposing-investigations
description: Use when answering an open-ended investigative question that needs evidence traced through multiple files, layers, or repos before you can commit to a confident verdict — e.g. "does this change regress X", "is it safe to remove Y", root-cause analysis, or auditing a change's blast radius.
argument-hint: "<the question to investigate>"
---

# Decomposing Investigations

## Overview

Split the question into independent, single-focus sub-questions, track them as tasks, and
answer each with a subagent sized to its complexity — cheap models for grep-shaped lookups,
stronger models for cross-file or cross-repo reasoning. You (the orchestrator) synthesize the
sub-verdicts into one definitive answer; never delegate the final call.

Left to their own devices, agents tend to investigate multi-step questions as one monolithic
pass — no decomposition, no subagents, no tracking — because the chain "feels short enough to
hold in context." That's exactly the shape that hides the one sub-question which would have
flipped the answer, and it wastes the option to run independent lookups in parallel or size
model cost to task difficulty.

## When to use

- "Does this change create a regression", "is it safe to change/remove X", "why does Z happen",
  "what's the blast radius of this" — anything needing evidence from several files/subsystems.
- Skip this for a single narrow lookup answerable by one grep or read — decomposing that just
  adds overhead.

## Model tiering

| Sub-question shape | Model |
|---|---|
| Single-file fact/definition check, grep sweep, precedent check | haiku |
| Multi-file trace within one repo (client → server, caller → callee) | sonnet |
| Cross-repo trace, unfamiliar/native code, or the step carrying the most uncertainty | opus |
| Final synthesis of all sub-verdicts | always you — never delegate |

## Steps

1. Read enough yourself to enumerate the independent sub-questions whose answers combine into
   the verdict. Include at least one sub-question that could *disprove* your working hypothesis,
   not just confirm it — a set of only confirming checks is confirmation bias with extra steps.
2. Track them — one task per sub-question (TaskCreate) — so nothing is dropped and progress is
   visible.
3. Assign each sub-question a model tier from the table above, and dispatch the independent ones
   together in a single message. Sequence only when one answer changes the scope of the next.
4. Give each subagent: read-only scope, the files/functions to start from if known, and an
   explicit instruction to end with a one-line verdict for its specific sub-question plus
   file:line citations. Prose without a verdict line forces you to re-derive the answer yourself.
5. If a cross-repo knowledge base or project memory has a relevant note, treat it as a **lead**,
   not a verified fact — task a subagent to check it against current source. A KB note claiming
   a store "auto-derives a visibility bit unconditionally" once turned out, on inspection, to be
   conditional on other bits already being set; citing it unverified would have produced a wrong
   verdict.
6. Mark each task completed as its notification lands.
7. Synthesize the sub-verdicts into one definitive answer yourself, citing the evidence each
   subagent found.

## Common mistakes

- Doing the whole investigation solo because it "feels narrow enough" — hides the sub-question
  that flips the answer.
- One subagent for the whole thing — loses cost-to-complexity sizing and parallelism.
- Skipping task tracking because the chain "isn't that long" — feels short until a follow-up
  needs the same ground covered again.
- Trusting a KB/memory lead without a verify step.
- Letting subagents return open-ended prose with no verdict line.

## Real-world impact

A 5-task decomposition (haiku ×2 for trivial lookups/precedent checks, sonnet ×2 for multi-file
client+server traces, opus ×1 for the deepest cross-repo native-store check) resolved "does
swapping a constant regress folder-permission display" with a confirmed, evidence-backed verdict
from parallel agents — and caught that an existing KB note about auto-added permission bits was
imprecise, which a single solo pass would likely have taken at face value.

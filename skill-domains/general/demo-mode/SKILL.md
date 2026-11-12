---
name: demo-mode
title: Demo Mode
description: 'Activate ultra-concise, precision-first response formatting for live demos — answers exactly what was asked, leads with the core fact first, and assumes an attention-constrained audience. Triggers on "demo mode", "concise answer", "just the facts", "keep it brief", or /demo-mode. Stays active for the whole session until the user asks to elaborate.'
domain: general
metadata:
  author: demo-user
  version: 1.0.0
trigger:
  mode: HYBRID
  slash_commands: ['/demo-mode']
  keywords: ['demo mode', 'concise answer', 'keep it concise', 'keep it brief', 'just the facts', 'short answer', 'quick answer', 'precise answer', 'no elaboration']
  intent_categories: ['response-formatting', 'demo-presentation']
  min_confidence: 0.6
sticky: true
prompt:
  constraints:
    - Answer exactly what was asked — no extra context, caveats-as-paragraphs, or "you might also want to know" asides unless the user explicitly asks to elaborate.
    - Lead with the single core fact/number/answer first, before any supporting detail.
    - Omit preamble ("Sure, let's look into that") and omit closing recaps/offers.
    - Keep responses short — one or two sentences, a single number, or a tight list/table — never a full paragraph unless the question itself demands it.
    - Do not bundle multiple findings into one answer unless each was explicitly asked for.
    - Still run the full, correct procedure of whatever other skill owns the task (SQL correctness, evidence, approval gates) — Demo Mode changes only how the final answer is packaged, never what work gets done.
    - Never drop a required caveat (stale score, approval gate, null vs zero) to save space — trim prose, not substance.
    - Stay active for the rest of the session once triggered; only relax when the user explicitly asks to elaborate or to exit demo mode.
  output_format: |
    Line 1: the direct answer (a number, name, yes/no, or one-line result).
    Then at most 2-3 supporting bullets or a compact table, only if essential to back up the answer.
    No restated question, no closing "let me know if you need more."
---

# Demo Mode

## When to Use
- User says "demo mode", "/demo-mode", or asks for a concise/precise/to-the-point answer.
- Live demo audience — assume short attention span: get to the answer in the first line.
- Once triggered, stays active for the rest of the session; no need to re-invoke per question.

## Core Concepts
- Demo Mode is a **formatting overlay**, not a replacement for other skills. The skill that actually does the work (e.g. `stars-c01-2025`, `td-data-profile`, `context-engine-mcp`) still runs its normal procedure, tool calls, and evidence rules. Demo Mode only governs how the final answer is written.
- "Precise" = answer the literal question, not the surrounding topic. Asked for a rate? Give the rate — not rate + methodology + trend, unless asked.
- Exit condition: stays on until the user says something like "elaborate," "give me more detail," "walk me through it," or "exit/turn off demo mode."

## Procedure
1. Detect the trigger and confirm activation once, briefly ("Demo mode on — concise answers until you ask for more").
2. For every following question: do the underlying work exactly as normal (same tools, same accuracy, same evidence), then render the final response per Output Format above.
3. Keep necessary caveats as a single clause, not a paragraph — brevity never means overstating confidence or hiding a real flag (e.g. stale data, RAG-amber).
4. If the user asks to elaborate, answer that one turn in full, then return to brief mode on the next question unless told to stay expanded.
5. If the user asks to exit demo mode, resume normal verbosity for the rest of the session.

## Common Errors
| Error | Cause | Fix |
|-------|-------|-----|
| Drifts back into long answers after one brief reply | Treated as one-shot instead of sticky | Re-apply Output Format every turn until user asks to elaborate/exit |
| Drops a needed caveat to stay short | Confusing brevity with omission | Keep one-clause caveats; trim prose, never substance |
| Demo Mode is read as skipping another skill's approval gate or procedure | Misreading scope | Demo Mode only repackages the final answer — gates/evidence rules from other skills still apply in full |

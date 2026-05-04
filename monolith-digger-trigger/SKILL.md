---
name: monolith-digger-trigger
description: When implementing features in a continuously-developed legacy monolith, detect code patterns that imply historical scars, migrations, or incomplete transitions and spawn an isolated investigation subagent before proceeding.
---

# Monolith Digger: Trigger

You are working in a continuously-developed monolithic codebase where the code contains evolutionary scars — migrations half-finished, dual implementations, hardcoded transitional rules. You cannot rely on architecture documents or Jira searches to understand these. The only reliable approach is targeted Git archaeology performed in an isolated subagent.

## When to investigate

If you encounter any of these **structural anomalies** while reading or editing code, you MUST spawn the `monolith-digger-worker` subagent before implementing:

| Pattern | Example |
|---------|---------|
| Hardcoded string/ID in business branch | `if (status.equals("LEGACY_ENT"))` |
| Dual implementations of same concern | `OldAuthClient` vs `NewAuthClient` |
| Versioned or temporary naming | `InvoiceServiceV1`, `TempPriceCalculator` |
| Domain concept inconsistency | `Order` in module A, `BillingOrder` in module B |
| Unexplained architecture bypass | Domain model calling JDBC directly |
| Date/region/customer-type hardcoding | `if (date.isBefore(LOCAL_DATE_2024))` |
| Null-checks for fields never set in visible flow | `if (legacyOrderId != null)` |
| Deprecated methods/classes still actively called | `@Deprecated` with 20 call sites |
| Empty/generic catch swallowing | `catch (Exception e) { log.warn("failed"); }` |
| Delegation chains with no added behavior | `methodA()` → `methodB()` same params, no logic |
| Cross-module calls without clear contract | `OldAuthClient.validate(...)` inside new code |

## Do NOT investigate inline

These are resolved by reading code directly. Do not burn a subagent call:
- Import resolution, type signatures, or method bodies
- Recent refactors (last 2 commits — read the diff directly)
- "What does this function do?" — read it

## How to spawn the worker

Formulate a **single, specific Question** and invoke the worker with fresh context:

```yaml
agent: monolith-digger-worker
context: fresh
task: |
  Question: "{specific_question}"
  
  You are a code archaeologist. Follow your investigation protocol:
  1. Use git commands to trace the origin of the relevant code.
  2. If you find a ticket ID in a commit message, use `acli` to fetch that exact ticket.
  3. Return ONLY in the FINDING or ASK_HUMAN format.
  4. Do not guess. Do not summarize multiple possibilities.
```

### Good questions
- `Why does PaymentService check order.getType().equals("LEGACY_ENT") and should my new endpoint do the same?`
- `InvoiceServiceV1 delegates every call to InvoiceServiceV2 with no added logic. Was this migration abandoned or is V1 still required?`
- `Which auth client is current: OldAuthClient or NewAuthClient? Commit abc123 uses Old, but def456 references a migration.`

### Bad questions
- `What is the history of PaymentService?` (too broad)
- `Tell me about LEGACY_ENT.` (not actionable)
- `Why is this code bad?` (judgmental, not factual)

## How to handle the response

### If worker returns FINDING
```
FINDING: <one sentence>
SOURCE: <git/jira/gh citation>
CONFIDENCE: high
```
- Treat the finding as a fact. Cite it in your reasoning (e.g., "Per PROJ-4421, LEGACY_ENT guards enterprise billing").
- Proceed with implementation.

### If worker returns ASK_HUMAN
```
ASK_HUMAN: <specific question>
REASON: <why investigation failed>
```
- **Immediately stop implementation.**
- Present the question and reason to the developer exactly as written.
- Include what you already checked so the developer understands the gap.
- Do NOT attempt to answer the question yourself. Do NOT proceed until the developer responds.

## Anti-patterns

| Do not | Why |
|--------|-----|
| Run `git blame`, `acli`, or `gh` in this main session | Pollutes context with dead ends; use the worker subagent |
| Investigate `TODO` / `FIXME` / `HACK` comments | These signal future work, not historical mystery |
| Broad Jira keyword search | Returns 50 irrelevant tickets |
| Synthesize when uncertain | "I think it might be..." creates false confidence |
| Proceed after ASK_HUMAN without developer response | Risk of building on wrong assumptions |

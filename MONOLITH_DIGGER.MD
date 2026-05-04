# Monolith Digger: Investigation Skill Specification

> **Purpose:** Enable Pi agents to write coherent code in a continuously-developed legacy monolith by performing isolated, focused historical investigations without polluting the main implementation context.
>
> **Core Insight:** Implementation plans are speculative fiction. Jira/PR archives are noise. The only reliable workflow is **Git-first archaeology → targeted ticket lookup → human escalation**.

---

## 1. Philosophy & Constraints

### What We Learned (The Hard Way)
- **Implementation plans are useless.** They describe intent before reality hit. The code shows the compromise; the plan shows the fantasy.
- **Raw Jira/PR dumps are 90% noise.** Comments like "Any updates?", "CI retrying", and "+1" create high-density noise when compressed into any database.
- **LLMs cannot know what they don't know.** An agent seeing `LEGACY_ENT` does not know to search for a 2019 enterprise billing migration unless explicitly triggered.
- **You cannot replace a developer with project knowledge.** A senior engineer carries six months of scar tissue. An LLM starts with zero context every session and hallucinates confidently in the gaps.
- **Context is precious.** Every failed `git blame`, every meandering Jira ticket, every "hmm this is unclear" burns tokens in the main session and pushes the agent toward guessing.

### What Works Instead
1. **Git is ground truth.** It is local, fast, auth-free, and shows actual changes.
2. **Jira is secondary intent.** Only look up tickets when a **specific ticket ID** is found in a commit message. Never search Jira by keyword.
3. **Kibana/logs are out of scope.** Production logs describe runtime symptoms, not code structure. They mislead more than inform during feature implementation.
4. **Human is the final validator.** When intent is unclear, the agent must stop and ask. Never synthesize an answer to be helpful.

---

## 2. Two-Skill Architecture

We use **two separate skills** to enforce a context firewall.

### Skill A: `monolith-digger-trigger` (Main Agent)
- **Injected into:** The main Pi agent session.
- **Responsibility:** Recognize when historical context is needed, define the question, and spawn the worker subagent.
- **Rule:** This skill **never** performs API calls, `git` commands, or external lookups. It only decides *whether* to investigate and *what* to ask.
- **Context cost:** Minimal. Only contains trigger patterns and subagent invocation template.

### Skill B: `monolith-digger-worker` (Subagent)
- **Spawned via:** Pi subagent mechanism with `context: "fresh"` (isolated from main session).
- **Responsibility:** Execute the investigation, return a brutally concise answer, or explicitly surrender.
- **Rule:** This skill **only** reads. It never writes to Jira, GitHub, or the codebase. It has a strict output contract.
- **Context cost:** Borns its own budget. If it spirals, the main agent is unaffected.

### Why Fresh Context?
If the worker runs in the main session, failed investigations leave noise (dead-end commits, irrelevant tickets, confused reasoning). That noise biases the main agent toward hallucination.

With a fresh-context subagent, the handoff is binary:
- **Success:** One sentence of fact + citation.
- **Failure:** The string `ASK_HUMAN: [specific question]`.

---

## 3. Trigger Patterns (When to Investigate)

The main agent MUST NOT use judgment ("this looks confusing"). It MUST match explicit patterns.

### Why Comment-Word Triggers Fail
Do not rely on words like `HACK`, `LEGACY`, or `TODO` in comments. In production monoliths:
- Teams rarely label their workarounds; the scar is in the structure, not the prose.
- `TODO` almost always means *future work* ("refactor this next quarter"), not *historical mystery*. Investigating every TODO creates noise and false positives.
- A `LEGACY` prefix might exist in a variable name, but it is just as likely to be hidden in a hardcoded string, a dual implementation, or an unexplained branch.

**The real signals are structural anomalies — patterns in the code that imply evolution, compromise, or incomplete migration.**

### Automatic Triggers
Spawn a worker subagent when encountering any of the following during implementation:

| Pattern | Example | Investigation Target |
|---------|---------|---------------------|
| Hardcoded string/ID in business branch | `if (status.equals("LEGACY_ENT"))` | What business rule does this represent? Is it permanent or transitional? |
| Dual implementations of same concern | `OldAuthClient` vs `NewAuthClient` | Which is current? Is there an ongoing migration? |
| Versioned or temporary naming | `InvoiceServiceV1`, `TempPriceCalculator` | Why wasn't this cleaned up? What's blocking removal? |
| Domain concept inconsistency | `Order` in module A, `BillingOrder` in module B for same entity | When did the model diverge? Which is authoritative? |
| Unexplained architecture bypass | Domain model calling JDBC directly; skipping validation in one path | Why is the architecture violated here? Incident-driven? |
| Date/region/customer-type hardcoding | `if (date.isBefore(LOCAL_DATE_2024))` | Business rule or transitional migration gate? |
| Null-checks for fields never set in visible flow | `if (legacyOrderId != null)` | Is this field populated by an old migration? Batch job? External system? |
| Deprecated methods/classes still actively called | `@Deprecated` method has 20 call sites | Why wasn't it removed? Is there a migration path? |
| Empty/generic catch swallowing | `catch (Exception e) { log.warn("failed"); }` | What exception was being defended against? |
| Delegation chains with no added behavior | `methodA()` calls `methodB()` with same params, no logic | Was this part of a rename/migration left half-finished? |
| Cross-module calls without clear contract | `OldAuthClient.validate(...)` inside new code | Is there an ongoing migration? Which client is current? |

### Do NOT Investigate
These are resolved by reading code directly. Do not burn a subagent call:
- Import resolution (`import com.company.legacy.Foo` — just read the file)
- Type signatures or method bodies
- Recent refactors (last 2 commits — read the diff directly)
- "What does this function do?" — read it.

### Trigger Workflow
```
Main Agent reads file
→ Sees pattern match (e.g., LEGACY_ENT)
→ Formulates Question string (see Section 5)
→ Spawns monolith-digger-worker subagent with Question
→ Waits for return
→ If ASK_HUMAN: stops and asks developer
→ If FINDING: uses fact, cites source in reasoning
```

---

## 4. Worker Investigation Protocol

The subagent receives exactly one **Question** string. It follows this workflow in strict order.

### Step 1: Git Excavation (Local, No Auth)
All commands run in the repository root.

```bash
# A. What changed here recently?
git log --no-merges --oneline -20 -- path/to/file

# B. When was this specific string introduced?
git log -S "PATTERN" --oneline -- path/to/file

# C. What did that commit actually contain?
git show <HASH> --stat

# D. Full diff if stat is insufficient
git show <HASH> -- path/to/file

# E. Who owns this file? (recent contributors)
git shortlog -sn --no-merges -20 -- path/to/file

# F. Is this pattern used elsewhere?
git grep -n "PATTERN"
```

**Stop condition:** If `git log -S` returns a commit with a ticket ID in the message, proceed to Step 2. If no ticket ID is found after 3 hops, skip to Step 4 (Surrender).

### Step 2: Targeted Jira Lookup (Read-Only)
Only if a **specific ticket ID** was found in a commit message.

```bash
# Cache to prevent re-fetch
mkdir -p /tmp/pi-monolith-digger

# Fetch ticket
acli jira workitem view PROJ-1234 > /tmp/pi-monolith-digger/jira-PROJ-1234.txt 2>/dev/null || echo "JIRA_FETCH_FAILED"
```

**Rules:**
- Only exact ticket IDs from commit messages.
- No keyword search. No browsing related tickets. No following links.
- If `JIRA_FETCH_FAILED`, treat as `insufficient_data`.
- Parse only the summary, description, and resolution fields.

### Step 3: Targeted GitHub PR Lookup (Optional)
If commit message references a PR number, or if `gh` CLI is available and the commit has a PR merge parent:

```bash
gh pr view <NUMBER> --json title,body,state > /tmp/pi-monolith-digger/gh-pr-<NUMBER>.json 2>/dev/null || echo "GH_FETCH_FAILED"
```

Same rules: exact ID only, read-only, one hop.

### Step 4: Decide & Format Output

After investigation, the worker MUST return exactly one of these formats. No prose. No reasoning dump.

#### Format A: Finding
```
FINDING: [One sentence, max 25 words]
SOURCE: [git:<HASH> / jira:<TICKET> / gh:<PR>]
CONFIDENCE: high
```

**Example:**
```
FINDING: LEGACY_ENT guards a 2019 enterprise billing contract path that bypasses new tax rules.
SOURCE: git:a1b2c3d, jira:PROJ-4421
CONFIDENCE: high
```

#### Format B: Insufficient Data
```
ASK_HUMAN: [Specific question the developer can answer in one sentence]
REASON: [Why investigation failed — e.g., "Commit messages lack ticket IDs" or "Jira ticket references Slack conversation not accessible"]
```

**Example:**
```
ASK_HUMAN: Does LEGACY_ENT still need to bypass tax rules for enterprise contracts, or was that fully migrated in Q2?
REASON: Found commit a1b2c3d referencing PROJ-4421, but ticket description says "see Slack thread" with no further detail.
```

#### Format C: Ambiguous / Contradictory
```
ASK_HUMAN: [Question highlighting the contradiction]
REASON: [Contradiction summary]
SOURCES: [git:<HASH-A> says X, git:<HASH-B> says Y]
```

**Critical:** The worker must NEVER return "I think..." or "It might be..." as a FINDING. If confidence is not high, it is ASK_HUMAN.

---

## 5. Question Formulation (Main Agent Responsibility)

When spawning the worker, the main agent must pass a **single, specific Question** string. Vague questions produce vague answers.

### Good Questions
- `Why does PaymentService check order.getType().equals("LEGACY_ENT") and should my new endpoint do the same?`
- `InvoiceServiceV1 delegates every call to InvoiceServiceV2 with no added logic. Was this migration abandoned or is V1 still required?`
- `Which auth client is current: OldAuthClient or NewAuthClient? Commit abc123 uses Old, but def456 references a migration.`

### Bad Questions
- `What is the history of PaymentService?` (too broad)
- `Tell me about LEGACY_ENT.` (not actionable)
- `Why is this code bad?` (judgmental, not factual)

### Subagent Invocation Template
```yaml
agent: monolith-digger-worker
context: fresh
task: |
  Question: "{specific_question}"
  
  You are a code archaeologist. Follow the investigation protocol:
  1. Use git commands to trace the origin of the relevant code.
  2. If you find a ticket ID in a commit message, use `acli` to fetch that exact ticket.
  3. Return ONLY in the FINDING or ASK_HUMAN format.
  4. Do not guess. Do not summarize multiple possibilities.
```

---

## 6. Main Agent Post-Processing

When the worker returns:

### If FINDING
- Include the finding as a fact in your reasoning.
- Cite the source (e.g., "Per PROJ-4421, LEGACY_ENT guards enterprise billing").
- Proceed with implementation.

### If ASK_HUMAN
- **Immediately stop implementation.**
- Present the question to the developer exactly as written by the worker.
- Include the REASON so the developer understands what was already checked.
- Do not attempt to answer the question yourself.
- Do not proceed until the developer responds.

**Example escalation message:**
```
I need your help understanding something before I proceed.

Question: Does LEGACY_ENT still need to bypass tax rules for enterprise contracts, or was that fully migrated in Q2?

What I already checked:
- Found commit a1b2c3d referencing PROJ-4421
- PROJ-4421 description references a Slack thread I cannot access
- No other commits in the last year mention removing LEGACY_ENT

Could you clarify? I'll wait for your answer before continuing.
```

---

## 7. Tooling Reference

### Git (Always Available, No Auth)
All standard git commands. Prefer porcelain over plumbing for readability.

### ACLI (Atlassian CLI)
- Command: `acli jira workitem view <TICKET>`
- Auth: Inherits user's existing `acli` authentication (SSO, token, etc.).
- Mode: Read-only. Never use `create`, `update`, `transition`.
- Output: Plain text. Worker should grep for `Summary:`, `Status:`, `Description:`.
- Failure: `JIRA_FETCH_FAILED`

### GitHub CLI (`gh`)
- Command: `gh pr view <NUMBER> --json title,body,state`
- Auth: Inherits user's existing `gh auth`.
- Mode: Read-only. Never use `pr create`, `merge`, etc.
- Failure: `GH_FETCH_FAILED`

### Caching Strategy
Worker should cache all external fetches in `/tmp/pi-monolith-digger/`:
```
/tmp/pi-monolith-digger/
  jira-PROJ-1234.txt
  gh-pr-567.json
```
This prevents re-fetching if the worker revisits data mid-investigation. Cache is session-only; Pi temp cleanup handles disposal.

---

## 8. Anti-Patterns (DO NOT DO)

| Anti-Pattern | Why It Fails |
|-------------|-------------|
| **Broad Jira keyword search** | Returns 50 irrelevant tickets. Noise overwhelms signal. |
| **Kibana/log investigation** | Runtime symptoms != code structure. Leads to defensive coding against ghosts. |
| **Recursive ticket following** | PROJ-1234 links to PROJ-5678 links to PROJ-9999. Subagent burns budget and returns confusion. |
| **Write-back to "learn"** | LLMs hallucinate lessons. Unvalidated write-backs poison future investigations. |
| **Synthesizing when uncertain** | "I think it might be..." is worse than no answer. It creates false confidence in main agent. |
| **Investigating everything** | Not every unknown needs history. Imports, types, and recent diffs are readable directly. |
| **Long-form worker output** | Paragraphs of worker reasoning become noise in main context. One sentence or surrender. |

---

## 9. Evolution of This Design

This document captures the journey from a bad idea to a workable one:

1. **Initial idea:** Central SQLite database ingesting all Jira/GitHub data, compressed, API-served.
   - **Flaw:** Storing noise at high density. LLMs can't query SQL effectively. Single-writer bottleneck.

2. **Revised idea:** JIT skill querying Jira/GitHub APIs directly from main agent.
   - **Flaw:** Pollutes main context with investigation dead ends. Auth complexity per user. LLM doesn't know what to ask.

3. **Current idea (this doc):** Trigger-based subagent with fresh context, Git-first protocol, exact-ID Jira fallback, brutal output contract, human escalation.
   - **Merit:** Context isolation, human-aligned workflow, no persistent database, read-only safety.

---

## 10. Future Extensions (If Needed)

- **Confluence integration:** If ADRs exist in Confluence, worker could fetch exact page IDs found in commit messages.
- **Team ownership map:** `CODEOWNERS` + git shortlog to suggest "ask @alice" in ASK_HUMAN output.
- **Decision log:** A manually curated `DECISIONS.md` in the repo root, maintained by humans, read by worker as Step 0.
- **Session memory:** If Pi gains cross-session memory, cache FINDINGs per file pattern to avoid re-investigating `LEGACY_ENT` every session.

---

## 11. Summary Checklist for Skill Developers

When implementing `monolith-digger-trigger`:
- [ ] Define 5-10 exact trigger patterns for your monolith.
- [ ] Ensure main agent never runs git/API commands inline.
- [ ] Subagent invocation uses `context: fresh`.
- [ ] Question formulation is specific and actionable.

When implementing `monolith-digger-worker`:
- [ ] Git commands are Step 1. Jira is Step 2. Human is Step 4.
- [ ] Only exact ticket IDs from commits. No keyword search.
- [ ] Output is strictly FINDING or ASK_HUMAN format.
- [ ] No guessing. No long-form prose. No synthesis of contradictions.
- [ ] All external calls cached to `/tmp/pi-monolith-digger/`.
- [ ] Read-only. No write operations to any system.

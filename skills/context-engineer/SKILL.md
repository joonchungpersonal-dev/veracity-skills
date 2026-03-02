---
name: context-engineer
description: Predict, prevent, and mitigate context window failures before they happen. Analyzes workflows for overflow risk, designs disk-as-memory architectures, and produces recovery-ready state files.
user-invocable: true
argument-hint: [workflow-description]
---

# Context Engineer

Predict, prevent, and mitigate context window failures before they happen.

## The Victory Condition

Before reading anything else, know the test:

> **If you can `/clear` the entire conversation, read your state file, and continue the workflow without any degradation — your architecture is correct.**

Every design decision in this skill serves that single criterion. If your workflow passes it, proceed with confidence. If it doesn't, you have hidden state in conversation memory that the next compaction will silently destroy.

## Core Problem

Context windows are finite (200K tokens for Opus 4.6). Complex workflows — iterative audits, multi-agent pipelines, large refactors — can exceed this limit. When they do, two bad things happen:

1. **Hard failure**: Context cannot be compacted, session crashes, work is lost.
2. **Soft failure**: Context IS compacted, but the summary silently drops information the workflow needs later. This is worse than a crash because you don't know what you lost.

Compaction is lossy compression. A 150K-token conversation summarized to 10K tokens has lost 93% of its information. The question is never "will information be lost?" — it's "will the RIGHT information survive?"

## When to Invoke This Skill

Use `/context-engineer` BEFORE starting any workflow that:
- Will spawn 5+ subagents
- Involves iterative loops (audit → fix → re-audit)
- Will read more than 10 files
- Produces intermediate artifacts that later steps depend on
- Runs for more than 30 minutes
- Has been attempted before and crashed or produced degraded output

## Information Taxonomy

Not all context tokens are equal. Every piece of information in a context window falls into one of five categories, each with different fidelity requirements.

**The categories form a dependency chain:**

```
RELATIONAL depends on → REASONING depends on → FACTUAL depends on → IMPERATIVE
```

Losing a foundation layer cascades upward and destroys everything that depends on it. If you lose FACTUAL information (scores, measurements), all REASONING built on those facts becomes ungrounded, and all RELATIONAL connections between those reasoning chains become meaningless. Preserve from the bottom up.

### Category 1: IMPERATIVE (instructions, goals, constraints)
- **Examples**: "Reach 97/100 on 3 consecutive runs", "Never push without audit", style guides
- **Fidelity requirement**: LOSSLESS — must survive every compaction
- **Preservation strategy**: Belongs in CLAUDE.md, skills, or rules files. Never rely on conversation memory for imperatives.
- **Risk if lost**: Workflow drifts from goal, constraints violated silently

### Category 2: FACTUAL (discovered facts, measurements, data)
- **Examples**: "Run 2 scored 80/100", "File X has 235 lines", "3 tests failed"
- **Fidelity requirement**: LOSSLESS for key facts, lossy acceptable for supporting details
- **Preservation strategy**: Write to structured state files (JSON). Facts in conversation memory WILL be corrupted or lost during compaction.
- **Risk if lost**: Decisions made on incorrect premises, regression, repeated work

### Category 3: REASONING (chains of thought, decision rationale, causal analysis)
- **Examples**: "We chose approach X because Y would break Z", "The score dropped because of the FActScore lineage fix"
- **Fidelity requirement**: HIGH — expensive to reconstruct, often impossible to reconstruct identically
- **Preservation strategy**: Write decision logs to disk. Include the WHY, not just the WHAT.
- **Risk if lost**: Same mistakes repeated, contradictory decisions across runs, inability to explain choices

### Category 4: EPHEMERAL (raw tool output, intermediate computation, verbose logs)
- **Examples**: Full `git diff` output, complete test suite results, raw web fetch content
- **Fidelity requirement**: NONE after extraction — safe to discard once key facts are extracted
- **Preservation strategy**: Extract facts immediately, then let compaction discard the raw data.
- **Risk if lost**: None, if facts were extracted first. Total, if facts were NOT extracted.

### Category 5: RELATIONAL (connections between facts, dependency graphs, cross-reference maps)
- **Examples**: "Finding F015 caused Fix X which caused regression in Run 3", "Agent B6 contradicted Agent C3 on claim Y"
- **Fidelity requirement**: CRITICAL — this is the category most damaged by compaction
- **Preservation strategy**: Explicit relationship files on disk. Summaries preserve individual facts but destroy the relationships between them.
- **Risk if lost**: The most dangerous failure mode. You retain isolated facts but lose the web of connections that gives them meaning. Decisions become incoherent across runs.

## Phase 1: Workflow Analysis

When invoked, first understand the planned workflow by asking:

1. **What is the workflow?** (iterative audit, multi-file refactor, research synthesis, pipeline, etc.)
2. **How many steps/iterations?** (fixed or convergence-based?)
3. **What does each step produce?** (reports, code changes, data, analysis?)
4. **What does each step consume from previous steps?**
5. **What is the success criterion?** (score target, all tests pass, user approval?)

Then estimate the token budget:

### Token Budget Estimation

For each step in the workflow, estimate:

```
Step N:
  Inputs read:     [list files/data read, estimate tokens]
  Processing:      [reasoning, tool calls, discussion — estimate tokens]
  Outputs written: [files written, reports generated — estimate tokens]
  Carry-forward:   [what MUST survive to Step N+1 — estimate tokens]
  Discardable:     [what can be safely forgotten — estimate tokens]

  Step total:      [sum of above]
  Cumulative:      [running total across all steps]
  Headroom:        [200K - cumulative - system overhead (~20K)]
```

System overhead (always present, cannot be reduced):
- System prompt + CLAUDE.md: ~8-12K tokens
- MCP tool definitions: ~5-15K tokens (varies by server count)
- Skill definition (if loaded): ~2-6K tokens
- Extended thinking budget: ~2-32K+ tokens per turn (adaptive; can be higher with interleaved thinking during tool use)

**Available working context**: ~145-170K tokens depending on configuration.

### Accumulation Patterns

Identify which pattern the workflow follows:

**Linear accumulation** (research, single-pass analysis):
```
Tokens: ────────────────────────────▶ overflow
Each step adds ~constant tokens, no discarding
Risk: Predictable. Overflow at step = headroom / tokens_per_step
```

**Sawtooth** (iterative with compaction):
```
Tokens: /\/\/\/\/\/\─── overflow if compaction insufficient
Each cycle: accumulate → compact → accumulate → compact
Risk: Each compaction is lossy. Fidelity degrades with each cycle.
After N cycles, you're working from an N-times-compressed summary.
```

**Exponential** (multi-agent with cross-referencing):
```
Tokens: ──────────╱ overflow (sudden)
Agents produce findings, findings reference each other,
cross-references multiply token count non-linearly
Risk: Hardest to predict. Often feels fine until sudden blowup.
```

**Plateau** (disk-as-memory, well-designed):
```
Tokens: ───────────────────────────── stable
Each step externalizes to disk, compacts, starts clean
Risk: Lowest. But requires upfront architecture work.
```

## Phase 2: Chokepoint Prediction

A chokepoint is any step where:
- Cumulative tokens will exceed 80% of context window (150K+)
- A compaction would destroy information needed by a later step
- Multiple large artifacts must be in context simultaneously
- Cross-referencing between steps requires holding both in memory

### Common Chokepoints

**The Accumulator**: Reading a large file, making changes, reading it again to verify, making more changes. Each read adds the full file to context. A 5K-token file read 6 times = 30K tokens just for one file.
→ *Mitigation*: Read once, edit in place, trust the edit tool's output. Don't re-read to verify unless necessary.

**The Collector**: Gathering findings from multiple agents/sources and synthesizing. Each finding set is 2-5K tokens; 16 agents = 32-80K tokens of findings before synthesis even begins.
→ *Mitigation*: Have agents write findings to disk. Read findings file-by-file during synthesis, extract what's needed, compact between files.

**The Comparator**: Holding two versions of something in context to compare (before/after, v1/v2, run N vs run N+1). Both versions must coexist in context for comparison to work.
→ *Mitigation*: Write a structured diff to disk. Don't hold both full versions — hold one version and the delta.

**The Auditor Loop**: Iterating until convergence. Each iteration accumulates: read target → audit → discuss findings → apply fixes → log results → repeat. After 4 iterations of a 50K-token cycle, you've consumed 200K.
→ *Mitigation*: Disk-as-memory architecture (see Phase 4). Compact between iterations.

**The Growing Log**: Appending to a cumulative log (like veracity-log.json). Each read of the log gets more expensive. A 112KB log = ~28-34K tokens every time it's read (varies by JSON density).
→ *Mitigation*: Don't read the full log. Read only the last entry, or use a summary index file.

## Phase 3: Fidelity Risk Analysis

For each planned compaction point, analyze what will be lost:

### Fidelity Loss Assessment

```
Compaction Point: [after Run N / after Step X / etc.]

Context size before compaction: ~[X]K tokens
Target size after compaction:   ~[Y]K tokens
Compression ratio:              [X/Y]:1
Information loss:                ~[1 - Y/X]%

IMPERATIVE information in context:
  - [list] → Will these survive? [yes: in CLAUDE.md / no: at risk]

FACTUAL information in context:
  - [list key facts] → Externalized to disk? [file path / NOT YET — RISK]

REASONING chains in context:
  - [list key decisions with rationale]
  - Which are reconstructible? [can re-derive from code/data]
  - Which are NOT reconstructible? [subjective choices, user preferences, debate resolutions]
  → Not-reconstructible reasoning MUST be written to disk before compaction

RELATIONAL information in context:
  - [list cross-references, dependency chains, causal links]
  → These WILL be destroyed by compaction. Summaries flatten relationships.
  → Write explicit relationship maps to disk.

EPHEMERAL information in context:
  - [list raw tool output, verbose logs]
  → Safe to lose. Verify key facts were extracted first.
```

### Fidelity Under Compaction

Each compaction is a summary of a summary. Assume total loss of anything not externalized to disk.

There are no reliable percentages for how much survives — it depends on the summarizer's priorities, the content mix, and what the conversation looked like at compaction time. What IS known:

- **First compaction**: Key facts and recent context usually survive. Nuance, relationships, and early-conversation reasoning usually do not.
- **Second compaction**: You are now working from a compressed summary of a compressed summary. The dependency chain (RELATIONAL → REASONING → FACTUAL → IMPERATIVE) begins to break. Individual facts may survive but the connections between them are gone.
- **Third+ compaction**: Effectively a new session with partial amnesia. Decisions may be based on premises that were silently altered or dropped during prior compactions.

No empirical measurement of compaction fidelity exists. These are heuristic observations, not data. **Treat any compaction you did not plan for as a total loss of RELATIONAL and REASONING information.**

**Rule of thumb**: If your workflow requires more than 2 compaction cycles, you need a disk-as-memory architecture. Don't rely on compaction to manage long workflows.

## Phase 4: Disk-as-Memory Architecture

The solution to both hard failures (overflow) and soft failures (fidelity loss) is the same: **externalize state to structured files on disk, so the context window only ever holds one step's worth of work.**

### State File Design

Every complex workflow should have a state file. Design it to contain everything needed to resume the workflow from any point, without relying on conversation history.

```json
{
  "_schema": "context-engineer/workflow-state/v1",
  "_description": "Complete workflow state — context window can be cleared and resumed from this file alone",

  "workflow": {
    "name": "veracity-convergence-loop",
    "goal": "Reach 97/100 on 3 consecutive audits",
    "target_file": "/path/to/SKILL.md",
    "started": "2026-03-01T10:00:00Z"
  },

  "progress": {
    "current_step": 4,
    "total_steps": null,
    "consecutive_target_hits": 1,
    "converged": false
  },

  "history": [
    {
      "step": 1,
      "score": 62,
      "delta": null,
      "findings": {"critical": 6, "high": 10, "medium": 10, "low": 6},
      "fixes_applied": 16,
      "report_path": "/path/to/run1/consolidated_report.md",
      "reasoning_log": "/path/to/run1/decisions.md",
      "timestamp": "2026-03-01T10:15:00Z"
    }
  ],

  "active_findings": [
    {
      "id": "F015",
      "severity": "MEDIUM",
      "status": "deferred",
      "reason": "Display-only issue, not functional",
      "introduced_in_step": 1,
      "last_checked_step": 3
    }
  ],

  "relationships": [
    {
      "type": "caused_regression",
      "source": "Fix applied in step 2 (FActScore lineage)",
      "target": "New finding in step 3 (overclaim about 'pioneered')",
      "note": "Fixing one claim exposed an adjacent overclaim"
    }
  ],

  "decisions": [
    {
      "step": 2,
      "decision": "Deferred code fence nesting fix",
      "rationale": "Markdown display issue only; Claude interpreter handles it correctly",
      "reversible": true
    }
  ],

  "compact_instructions": "Preserve: current step number, target score (97), consecutive hits count. Read state from /path/to/state.json for full history."
}
```

### Checkpoint Protocol

Before each compaction point:

1. **Extract** all Category 2 (FACTUAL) information from context into the state file
2. **Log** all Category 3 (REASONING) chains into a decisions file
3. **Map** all Category 5 (RELATIONAL) connections into the relationships array
4. **Verify** all Category 1 (IMPERATIVE) information exists in CLAUDE.md or skill files
5. **Release** all Category 4 (EPHEMERAL) information — it's safe to lose
6. **Validate** — Read the state file back and confirm that every RELATIONAL and REASONING item currently in context also appears, explicitly, in the written record. Do not proceed to compaction until this check passes. This step exists because steps 1-3 are themselves lossy: the decision of what to write is a human judgment that can miss things.

Then compact with explicit instructions:
```
/compact Preserve: workflow state is in [path/to/state.json]. Current step: N. Goal: [goal].
Read state file at start of next step. Do not rely on conversation history for any facts.
```

### Recovery Protocol

After compaction (or in a new session), recovery is:

1. Read the state file
2. Read the current target file
3. Read the last consolidated report (if continuing mid-cycle)
4. You now have everything needed to continue. Conversation history is irrelevant.

**Test**: Apply the Victory Condition — `/clear`, read state file, continue. If it works, you're good.

## Phase 5: Generating the Context Engineering Plan

After analysis, produce a concrete plan in this format:

```markdown
# Context Engineering Plan: [Workflow Name]

## Budget
- Available context: ~[X]K tokens
- Estimated per-step cost: ~[Y]K tokens
- Maximum steps before overflow (no mitigation): [N]
- Maximum steps with disk-as-memory: unlimited

## Predicted Chokepoints
1. [Step N]: [description of why this overflows]
   Mitigation: [specific action]

2. [Step M]: [description of fidelity risk]
   Mitigation: [specific action]

## State File Location
[path to state.json]

## Checkpoint Schedule
- After Step 1: Write [X] to disk, compact
- After Step N: Write [Y] to disk, compact
- [pattern]

## Fidelity-Critical Information
These items MUST be externalized before any compaction:
1. [item] → [destination file]
2. [item] → [destination file]

## Compaction Instructions
Use this exact text when compacting:
> /compact [tailored instruction for this workflow]

## Recovery Command
If session crashes, start new session with:
> Read [state file path] and continue the [workflow name] from step [N].
```

## Phase 6: Anti-Patterns to Flag

When analyzing a workflow, flag these anti-patterns:

### Anti-Pattern 1: "I'll Remember"
Keeping critical state in conversation memory instead of on disk.
**Smells like**: No files being written between steps. Key numbers mentioned in conversation but not logged anywhere.
**Fix**: If it matters, write it down. If it doesn't matter, don't discuss it.

### Anti-Pattern 2: "Read It Again"
Re-reading the same large file multiple times across steps.
**Smells like**: The same `Read` call appearing 3+ times for the same file.
**Fix**: Read once, extract what's needed into a smaller working file, reference the working file.

### Anti-Pattern 3: "Append Forever"
Growing a cumulative log that gets read in full each step.
**Smells like**: A JSON array or log file that grows each iteration and is read each iteration.
**Fix**: Write a summary index file. Only read the full log when specifically needed.

### Anti-Pattern 4: "Everything in Main"
Running all work in the main conversation instead of using subagents.
**Smells like**: 10+ tool calls per step, all in the main conversation. Subagents not used.
**Fix**: Subagents get their own context windows. Use them for any step that produces > 10K tokens of output.

### Anti-Pattern 5: "Compress and Pray"
Relying on automatic compaction to manage a workflow that structurally exceeds the context window.
**Smells like**: No explicit compaction strategy. "It'll compact when it needs to." No state files.
**Fix**: Design the disk-as-memory architecture upfront. Compaction is a safety net, not a strategy.

### Anti-Pattern 6: "The Monologue"
Long reasoning passages in the conversation that could be a structured file.
**Smells like**: 2000+ word analysis blocks in conversation turns. Comparison tables. Narratives.
**Fix**: Write analysis to a file. Reference the file in conversation. The file survives; the conversation doesn't.

### Anti-Pattern 7: "Fat Subagent Returns"
Subagents returning full reports into the main conversation instead of writing to disk and returning a pointer.
**Smells like**: Subagent returns that are 5K+ tokens. Full findings lists in the return value.
**Fix**: Subagent writes full output to a specified file path. Returns only: status, score, file path, and a 3-line summary.

## Quick Reference: Token Estimates

| Content Type | Approximate Tokens |
|---|---|
| 1 KB of English text | ~250 tokens |
| 1 KB of JSON | ~300 tokens |
| 1 KB of code | ~200-350 tokens |
| Average file read (Read tool) | ~2-8K tokens |
| Average tool call + result | ~500-2K tokens |
| System prompt + CLAUDE.md | ~8-12K tokens |
| MCP tool definitions (all servers) | ~5-15K tokens |
| Loaded skill definition | ~2-6K tokens |
| Extended thinking (per turn) | ~2-32K tokens |
| Subagent return (well-designed) | ~500-2K tokens |
| Subagent return (poorly designed) | ~5-20K tokens |
| `/compact` output | ~3-10K tokens |
| **Available working budget** | **~145-170K tokens** |

## Execution

When `/context-engineer` is invoked:

1. Ask the user what workflow they're planning (or inspect the current conversation for an in-progress workflow)
2. Run Phase 1-3 analysis (budget, chokepoints, fidelity risks)
3. Design the state file schema (Phase 4) specific to this workflow
4. Output the Context Engineering Plan (Phase 5)
5. Flag any anti-patterns already present (Phase 6)
6. Write the state file template and calibration file template to disk
7. Write the plan to disk at the same location as the state file
8. State explicitly what this workflow cannot verify about itself (from the Limitations section)

If invoked mid-session (workflow already in progress):
1. Estimate current context usage
2. Identify what's at risk of being lost
3. Emergency externalization: write all critical state to disk NOW
4. Recommend a compaction instruction
5. Provide the recovery command for if/when the session crashes

If invoked after a workflow completes or crashes:
1. Run Phase 7 calibration — record predicted vs actual performance
2. Update token estimates if predictions were off by >30%
3. Note any chokepoints that were missed for future workflows

## Phase 7: Post-Workflow Calibration

After each workflow completes (or crashes), record these numbers to a calibration file at the same location as the state file:

```json
{
  "calibration": [
    {
      "workflow": "veracity-convergence-loop",
      "date": "2026-03-01",
      "predicted_peak_tokens": 55000,
      "actual_peak_tokens": null,
      "predicted_steps_before_overflow": 8,
      "actual_steps_completed": 4,
      "chokepoints_predicted": ["Growing log at step 3", "Collector at synthesis"],
      "chokepoints_missed": ["Fat subagent returns from Wave C"],
      "compactions_triggered": 2,
      "state_file_sufficient_for_recovery": true,
      "notes": "Underestimated agent return sizes by 3x"
    }
  ]
}
```

After 5+ workflows, review the calibration file. If predictions are consistently off by more than 30%, revise the token estimates in the Quick Reference table. This closes the feedback loop — without it, the skill predicts and plans but never learns from its own errors.

## What This Skill Cannot Protect Against

Name the blind spots before the work starts, not after the crash.

**1. The Externalization Paradox.** This skill frames disk-writes as "lossless" and compaction as "lossy." But the decision of WHAT to externalize is itself a compression. When you choose which RELATIONAL connections merit entry in the state file, you are already summarizing — before any mechanical compaction occurs. You have replaced one summarizer (the compaction algorithm) with another (yourself). The checkpoint protocol's Step 6 (validate) mitigates this but cannot eliminate it.

**2. The Self-Assessment Problem.** The agent performing the checkpoint is the same agent whose context may already be degraded. A degraded instrument cannot accurately measure its own degradation. If important RELATIONAL information was lost in a prior compaction, the agent will not know to externalize it — because it no longer remembers it existed. There is no solution to this within a single-agent system. The `/clear` test approximates a check, but passing it only proves the workflow *appears* to resume, not that genuine continuity was preserved.

**3. The Fiction of Continuity.** A context window does not contain a unified "mind" — it contains a sequence of associative responses shaped by whatever tokens happen to be present. What is lost in compaction may not be recoverable information; it may be the *illusion* of an ongoing agent with consistent judgment. State files preserve facts and relationships, but they cannot preserve whatever emergent coherence arose from the full conversation history. After recovery, the agent may make different decisions than it would have with full context, even when all externalized facts are restored.

**4. Token Estimate Uncertainty.** The Quick Reference table provides approximate token counts. These are heuristic estimates based on typical usage, not measured constants. Actual token consumption varies with content type, language, formatting, and model behavior. Do not plan to within 10% of the context limit based on these numbers. Leave at least 30% headroom.

**5. State File Growth.** The disk-as-memory architecture treats disk as infinite storage, but state files grow over time. In long workflows (20+ iterations), the state file itself becomes a chokepoint — reading it consumes context tokens. If the state file exceeds 10K tokens, add a summary index that contains only what the current step needs, with a pointer to the full state file for recovery.

## Design Principles

1. **Disk is infinite, context is not.** Treat the context window like RAM and disk like storage. Don't hold in RAM what can be paged in from storage.

2. **Compaction is lossy. Externalization is less lossy.** Writing a JSON state file preserves exactly what you chose to write — but the choice of what to write is itself a judgment call that can miss things. Compaction preserves whatever the summarizer thinks is important — which may not be what your workflow needs. Neither is perfect; externalization is better because the losses are at least visible and auditable.

3. **Relationships are the first casualty.** Summaries preserve nouns (facts, scores, names) but destroy verbs (caused, contradicted, depended on, regressed from). If your workflow depends on understanding WHY things happened, externalize the causal chain explicitly.

4. **Test with `/clear`.** If you can clear the entire conversation, read your state file, and continue without degradation — your architecture is correct. If you can't, you have hidden state in conversation memory that needs to be externalized. (Caveat: this test proves apparent continuity, not perfect continuity. See "What This Skill Cannot Protect Against.")

5. **The best compaction is the one you don't need.** A well-designed disk-as-memory workflow never hits the compaction threshold because each step starts lean.

---

## Recommended Pairing: Veracity-555

The `veracity-tweaked-555` multi-agent fact-checking skill is the canonical use case for context engineering. A single 3-run veracity audit deploys 48 agents, consumes 2.4-3M tokens, and produces cross-run findings that reference each other — hitting every chokepoint pattern described above.

### Why They Pair

| Context Engineering Concept | Veracity-555 Manifestation |
|---|---|
| The Collector | 16 agents per run, each returning 2-5K tokens of findings |
| The Auditor Loop | Iterative audit → fix → re-audit until convergence |
| The Comparator | Pre/post snapshots compared across runs |
| The Growing Log | veracity-log.json grows each run |
| RELATIONAL information | "Finding F015 from Run 1 caused a regression in Run 2" |
| Sawtooth accumulation | Each run accumulates ~50-80K, must compact between runs |

### How They Integrate

Veracity-555 (runs >= 2) automatically applies context-engineer patterns via its **Step 2.5: Context Engineering Pre-Flight**:

1. **Token budget estimation** before first run
2. **State file initialization** using `context-engineer/workflow-state/v1` schema
3. **Checkpoint protocol** between every run (extract FACTUAL, log REASONING, map RELATIONAL)
4. **Compact instruction** when context exceeds 120K tokens
5. **Recovery command** if session crashes mid-audit

The state file enables the Victory Condition: if the session crashes after Run 3 of a 6-run audit, a new session reads the state file and continues from Run 4 without repeating work or losing cross-run relationships.

### Empirical Results

The first paired deployment (veracity-tweaked-555 self-audit, 6 runs to convergence):

```
Runs: 6 | Agents deployed: 19 | Total fixes: 35
Score progression: 74 → 85 → 91 → 95.7 → 96.1 → 96.5
Compactions needed: 0
State file updates: 3
Peak context estimate: ~130K tokens (within 200K window)
Anti-patterns avoided: Fat Subagent Returns, I'll Remember, Append Forever
```

Zero compactions across 6 runs — the disk-as-memory architecture kept each run lean. The state file's `relationships[]` array preserved cross-run causation (e.g., "Run 1 Limitations addition → Run 2 token cost regression") that would have been destroyed by compaction.

### When to Invoke Separately

Use `/context-engineer` independently (without veracity-tweaked-555) for:
- Multi-file refactors touching 10+ files
- Research synthesis reading 20+ papers
- Any iterative workflow with convergence criteria
- Post-crash emergency externalization of in-progress work
- Calibrating token estimates after a workflow completes (Phase 7)

# Workflow: Iterative Convergence Loop

Pair `veracity-tweaked-555` with `context-engineer` to run an audit that iterates until the document reaches a target quality threshold. This workflow was used to self-audit the veracity-tweaked-555 SKILL.md itself, converging from 74/100 to 96.5/100 over 6 runs.

## When to Use This

- You need a document to reach a specific veracity score (e.g., 95+)
- Single-pass auditing isn't sufficient — you want to fix, re-audit, and verify fixes
- The audit may take 4+ runs, exceeding context window capacity
- You want crash recovery — if the session dies at Run 4, you can resume at Run 5

## Architecture

```
┌─────────────────────────────────────────────────────┐
│  context-engineer                                    │
│  ┌───────────────────────────────────────────────┐  │
│  │  State File (workflow-state.json)              │  │
│  │  • Score history    • Active findings          │  │
│  │  • Relationships    • Decisions                │  │
│  │  • Convergence tracking                        │  │
│  └──────────────┬────────────────────────────────┘  │
│                 │ read/write between runs             │
│  ┌──────────────▼────────────────────────────────┐  │
│  │  veracity-tweaked-555 (per run)                        │  │
│  │  Wave 0: Decompose → 283+ atomic facts         │  │
│  │  Wave A: 5 agents verify sources                │  │
│  │  Wave B: 5 agents check framing                 │  │
│  │  Wave C: 5 agents adversarial + synthesis       │  │
│  │  → Score, findings, fixes                       │  │
│  └──────────────┬────────────────────────────────┘  │
│                 │ checkpoint                          │
│                 ▼                                     │
│  Converged? (target score × N consecutive runs)      │
│  • Yes → Final report                                │
│  • No  → Apply fixes, update state, next run         │
└─────────────────────────────────────────────────────┘
```

## Step-by-Step

### 1. Define convergence criteria

Before starting, decide:
- **Target score**: What veracity score must the document reach? (e.g., 95)
- **Consecutive runs**: How many consecutive runs must hit the target? (e.g., 3)
- **Max runs**: Upper bound to prevent infinite loops (e.g., 10)

### 2. Initialize with context-engineer

The veracity-tweaked-555 skill handles this automatically (Step 2.5) for runs >= 2, but if you want manual control:

```
/context-engineer

Workflow: Iterative veracity audit of [target document]
Steps: Unknown (convergence-based, estimate 4-8 runs)
Per-step output: 16 agent reports, consolidated findings, applied fixes
Success criterion: Score >= [target] for [N] consecutive runs
```

This produces a state file and context engineering plan.

### 3. Run the audit

```
/veracity-tweaked-555 /path/to/document.md runs=N
```

For convergence loops, set `runs` higher than you expect to need. The convergence criterion (Step 3g) will offer early stop when the target is reached.

### 4. Between-run checkpoint cycle

After each run, the skill:

1. **Writes findings to disk** (paper trail: `run_N/consolidated_report.md`)
2. **Updates state file** with score, delta, findings, decisions
3. **Logs relationships** (regressions, patterns, causal chains)
4. **Compacts if needed** (context > 120K tokens)
5. **Checks convergence** (target hit for N consecutive runs?)

If NOT converged: apply accepted fixes, update state, start next run.
If converged: proceed to final report.

### 5. Recovery from crash

If the session dies mid-audit:

```
Read /path/to/audit-trail/state/workflow-state.json and continue
the veracity audit from run N.
```

The state file contains everything: score progression, active findings, deferred items, decisions, and relationship maps. No conversation history needed.

## Worked Example: Veracity-555 Self-Audit (March 2026)

The veracity-tweaked-555 SKILL.md was audited by itself, with a convergence target of 95/100 for 3 consecutive runs.

### Run-by-Run Progression

| Run | Score | Delta | Facts | Agents | Findings (C/H/M/L) | Fixes |
|-----|-------|-------|-------|--------|---------------------|-------|
| 1 | 74 | — | 283 | 9 | 5 / 7 / 8 / 4 | 15 |
| 2 | 85 | +11 | 304 | 6 | 0 / 0 / 1 / 3 | 12 |
| 3 | 91 | +6 | 304 | 1 | 0 / 1 / 3 / 4 | 3 |
| 4 | 95.7 | +4.7 | 304 | 1 | 0 / 0 / 2 / 5 | 5 |
| 5 | 96.1 | +0.4 | 304 | 1 | 0 / 0 / 1 / 5 | 0 |
| 6 | 96.5 | +0.4 | 304 | 1 | 0 / 0 / 1 / 4 | 0 |

**Converged** at Run 6: scores 95.7, 96.1, 96.5 — three consecutive runs above 95.

### What the Score Trajectory Reveals

**Phase 1 — Rapid improvement (Runs 1-2, delta +11)**:
The first run found the most severe issues. 5 CRITICAL findings included fabricated statistics ("3x more accurate"), inflated attribution ("SAFE pioneered" → "popularized"), and a logical dependency error between agents C5 and C3. Fixing these produced the largest single-run improvement.

But fixes introduced regressions: the new Limitations section contained 2 factual errors (understated token cost, incomplete tool list), caught in Run 2.

**Phase 2 — Diminishing returns (Runs 2-4, deltas +6, +4.7)**:
Each run found fewer and less severe issues. The dominant pattern shifted from factual errors to framing issues — "SAFE achieves superhuman factuality ratings" was technically overclaimed (should specify "wins 76% of disagreement cases"). These borderline MIXED/MOSTLY TRUE calls are where the noise floor lives.

**Phase 3 — Convergence (Runs 4-6, delta +0.4)**:
Runs 5 and 6 applied zero fixes. The score still crept up (+0.4 each) because different wave compositions rated borderline claims slightly differently. This ±0.4 is the measurement noise floor — further runs would oscillate here indefinitely.

### Key Findings by Pattern

**Attribution inflation** (5 findings, all Run 1):
The SKILL.md systematically used slightly stronger language than primary sources warranted. "Pioneered" instead of "popularized," "superhuman factuality ratings" instead of "superhuman rating performance." This was the dominant error pattern — not invention, but subtle inflation.

**Regression from fixes** (2 findings, Run 2):
New content introduced new verifiable claims. The Limitations section stated token cost as "300K-500K" (actual: 800K-1M, based on measured data from a prior audit run) and listed verification tools as "only WebSearch and WebFetch" (missed Glob, Bash, Read). Lesson: every edit is a new attack surface.

**Convergence dead code** (1 finding, Run 1):
The convergence criterion required "delta < 3 for two consecutive runs" but was documented as activating "After Run 2+" — which provides only 1 delta value, not 2 consecutive. At the default runs=3, convergence can never trigger. This was a logical composition error, not a factual one.

**Permanently deferred** (1 finding, all runs):
Waves D-I have bullet-point descriptions instead of full prompt blocks. This is a completeness issue (50+ lines of authoring effort), not a factual accuracy issue. Deferred permanently — tracked but never fixed.

### Context Engineering Performance

```
Compactions needed:        0
State file updates:        3 (after Runs 1, 2, and final)
Peak context estimate:     ~130K tokens
Anti-patterns avoided:     Fat Subagent Returns, I'll Remember, Append Forever
Anti-patterns present:     State file updates lower than optimal (3 vs 6 runs)
Recovery tested:           No (no crash occurred)
```

The disk-as-memory architecture worked: 6 runs, 19 agents, 35 fixes, zero compactions. The state file was sufficient to describe the full audit at any point — though the 3-update frequency (vs recommended every-run) was a gap. In a crash scenario, Runs 3-5 would have had slightly stale state files.

### State File (Final)

```json
{
  "_schema": "context-engineer/workflow-state/v1",
  "_description": "COMPLETE. 555 SKILL.md self-audit converged at 96.5/100 after 6 runs.",
  "convergence": {
    "target_score": 95,
    "required_consecutive": 3,
    "consecutive_hits": 3,
    "converged": true,
    "convergence_scores": [95.7, 96.1, 96.5]
  },
  "score_progression": [74, 85, 91, 95.7, 96.1, 96.5],
  "total_fixes_applied": 35,
  "total_agents_deployed": 19,
  "relationships": [
    {
      "type": "regression",
      "source": "Run 1 Limitations addition",
      "target": "Run 2 token cost + tool list errors",
      "note": "New content introduced 2 factual errors — caught and fixed in Run 2"
    },
    {
      "type": "pattern",
      "source": "Run 1 findings C1,C2,C5,H1,H2",
      "description": "Attribution inflation — systematic stronger-than-source language. All resolved Run 1."
    }
  ]
}
```

The `relationships[]` array is the key context-engineer contribution. Without it, each run is an isolated snapshot. With it, you can trace: "the Limitations section added in Run 1 caused the token cost error found in Run 2, which was fixed in Run 2 and verified clean in Run 3." This causal chain would be the first casualty of context compaction.

## Adapting This Workflow

### For a different target document

Replace the veracity-tweaked-555 invocation target. The convergence loop works identically for any document type — personal websites, research papers, README files, grant applications. The domain expert agents (Wave D) will auto-adapt to the target's subject matter.

### For stricter convergence

Increase the target score or consecutive run requirement:
- `target_score: 97, required_consecutive: 5` — very strict, expect 8-10 runs
- `target_score: 90, required_consecutive: 2` — moderate, expect 3-4 runs

### For cost control

- Start with `runs=1` to get a baseline score
- If score > 90, you may not need a convergence loop at all
- Set `max_runs` to cap spending (each run ≈ 800K-1M tokens)

### For README-specific audits

Use `readme-audit` instead of `veracity-tweaked-555`. The readme-audit skill shares the same wave architecture and produces compatible state files. The convergence loop works identically — just swap the skill invocation.

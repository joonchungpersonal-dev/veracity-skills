---
name: veracity-v2-experimental
description: "EXPERIMENTAL v2: Enhanced parallel veracity audit with 12 gap fixes. Launches 20 agents per run in 5 waves (decomposition + quality gate + 3 verification waves). Adds citation verification, decomposition QA, confidence calibration, blind re-derivation, logical composition, and smarter consensus."
user-invocable: true
argument-hint: <file-or-url> [runs=3]
---

# Veracity v2 Experimental — Enhanced Parallel Fact-Check

SAFE-style claim decomposition → quality gate → 3 waves of verification agents per run. Each wave attacks from a different angle. Multiple runs shift perspectives to reduce blind spots. Between runs, findings go through user review; approved fixes are applied before the next run.

## What's New in v2 (12 Gap Fixes)

| # | Gap | Fix | Where |
|---|-----|-----|-------|
| 1 | No Citation Verification | Citation Existence + Support Verifier | A1 |
| 2 | Decomposition Quality Unchecked | Disambiguation Auditor + Recall Auditor | Wave 0.5 |
| 3 | No Confidence Calibration | Calibration protocol + weighted synthesis | All + C5 |
| 4 | Model Homogeneity | Tool chain diversification + acknowledgment | All |
| 5 | No Self-Consistency/SelfCheckGPT | Stochastic consistency in Blind Re-Derivation | B6 |
| 6 | No Chain-of-Verification | Blind Re-Derivation agent | B6 |
| 7 | No Logical/Causal Composition | Logical Composition Verifier | C6 |
| 8 | No Conflicting Evidence Resolution | Conflict Resolution Protocol | C3 |
| 9 | Temporal Validity too late | A4 enhanced with "still true today?" | A4 |
| 10 | Consensus too simplistic | C5 rewritten with weighted aggregation | C5 |
| 11 | No Error Propagation Control | Abstention instruction | All |
| 12 | No Recall Audit | Recall Auditor | Wave 0.5 |

## Methodology

**Published methods:**
- **SAFE (Google DeepMind, arXiv:2403.18802)**: Atomic fact decomposition — outperforms crowdsourced human annotators (wins 76% of disagreement cases) at lower cost
- **FActScore (Min et al., EMNLP 2023)**: Popularized atomic fact decomposition; SAFE extends similar approaches (citing Gao et al. and Wang et al.)
- **Claimify (Microsoft, ACL 2025, arXiv:2502.10855)**: Disambiguation-aware claim extraction — 99% entailment, 87.6% recall, 96.7% precision `[GAP-2]`
- **GhostCite (arXiv:2602.06718)**: Citation hallucination detection — 14-94% hallucination rates across 13 LLMs, 98.75% for recent-year citations `[GAP-1]`
- **SourceCheckup (Stanford, Nature Communications 2025)**: 50-90% of LLM-cited sources don't fully support claims `[GAP-1]`
- **6-Point Veracity Scale**: TRUE / MOSTLY TRUE / MIXED / MOSTLY FALSE / FALSE / UNVERIFIABLE (adapted from PolitiFact)
- **Source Reliability Tiering**: T1 (DOIs, databases) > T2 (institutional) > T3 (secondary)
- **Tool-MAD Adversarial Debate (arXiv:2601.04742)**: Pro/con agents before a judge — up to 5.5% over prior SOTA
- **CoVe — Chain-of-Verification (Meta, ACL 2024 Findings)**: Independent re-verification without seeing originals reduces hallucinated entities from 2.95 to 0.68 `[GAP-6]`
- **LoCal (ACM Web Conference 2025)**: Logical composition — verified sub-facts may not logically entail original claims `[GAP-7]`
- **CONFACT (IJCAI 2025, arXiv:2505.17762)**: Conflicting evidence resolution for varying-credibility sources `[GAP-8]`
- **ChronoFact (IJCAI 2025, arXiv:2410.14964)**: 82% of temporal errors involve implicit temporal info `[GAP-9]`
- **SelfCheckGPT (arXiv:2303.08896)**: NLI variant AUC-PR 92.50 — if model "knows" a fact, samples agree; hallucinations diverge `[GAP-5]`
- **VeriFact/FactRBench (EMNLP 2025)**: First precision+recall benchmark — high precision ≠ high recall `[GAP-12]`
- **Demystifying MAD (arXiv:2601.19921)**: Calibrated confidence improves Brier scores 0.217→0.069 `[GAP-3]`
- **Scaling Truth (arXiv:2509.08803)**: AI Dunning-Kruger — smaller LLMs show high confidence despite lower accuracy; larger models better calibrated `[GAP-3]`
- **FREE-MAD (arXiv:2509.11035)**: Trajectory-based scoring +13-16.5% over baselines `[GAP-10]`
- **Beyond Majority Voting (arXiv:2510.01499)**: Optimal Weight aggregation outperforms majority voting `[GAP-10]`
- **Voting or Consensus? (ACL 2025 Findings)**: Voting better for reasoning, consensus for knowledge `[GAP-10]`
- **Alignment Bottleneck (arXiv:2602.10380)**: Conservative abstention reduces error propagation `[GAP-11]`

**Custom practices (not published):**
- **Supermajority weighted consensus**: 75% weighted agreement threshold, enhanced with calibration (Demystifying MAD)
- **Evidence chain logging**: Source URL/DOI, tier, relevant quote, confirms/contradicts
- **Counter-evidence prompting**: Agents try to disprove before marking VERIFIED
- **Tool chain diversification**: Different agents use different primary tools to reduce correlated errors `[GAP-4]`

**Known limitation `[GAP-4]`:** All agents use the same Claude model. Research (Stop Overvaluing MAD, arXiv:2502.08788; A-HMAD, 2025) shows model heterogeneity yields 4-6% absolute gains. We partially mitigate via tool chain diversification and prompt variation.

## Input

`$ARGUMENTS`: file path or URL to audit, optionally `runs=N` (default: 3). If no target, ask user.

## Architecture

Each **run** = 1 decomposition + 1 quality gate + 3 waves of verification = 20 agents/run.
- **Run 1**: Wave 0 → Wave 0.5 → Waves A, B, C (foundational)
- **Run 2**: Wave 0 (refresh) → Wave 0.5 → Waves D, E, F (adversarial)
- **Run 3+**: Wave 0 (refresh) → Wave 0.5 → Waves G, H, I (meta-analysis)
- **Run 4+**: Cycle (4→A/B/C, 5→D/E/F, 6→G/H/I) with all prior context

Total agents = runs × 20.

## Shared Protocols

When constructing subagent prompts, include these definitions wherever agents reference [VERACITY_SCALE], [EVIDENCE_CHAIN], [THINK_VERIFY], [OUTPUT_FORMAT], [CALIBRATION], [ABSTENTION].

**Veracity Scale (6-point, adapted from PolitiFact):**
- **TRUE**: Fully accurate, confirmed by T1 source
- **MOSTLY TRUE**: Accurate with minor discrepancy
- **MIXED**: Partially accurate
- **MOSTLY FALSE**: Significant inaccuracy
- **FALSE**: Factually incorrect
- **UNVERIFIABLE**: Cannot confirm or deny

**Source Tiers:** T1 (DOIs, databases, registrars, original data) > T2 (institutional, Google Scholar) > T3 (news, abstracts, secondary)

**Evidence Chain (required per rated fact):** Source URL/DOI | Source tier | Relevant quote | Confirms/contradicts

**Output Format:**
```
F### [CATEGORY] — **RATING** (Raw: N% → Calibrated: N%)
  Claim: "..." | Evidence: ... | Source: URL (Tier N) | Note: ...
```

**Think & Verify:** Before marking FALSE, double-check your source. Before marking VERIFIED, attempt to find contradicting evidence.

**Confidence Calibration `[GAP-3]` (required for every rating):**
Report TWO measures per fact:
1. RAW CONFIDENCE: initial gut-level (0-100%)
2. CALIBRATED CONFIDENCE: adjusted by: T1 source (+20) / T2 (+10) / T3 (+0) / none (-20); 2+ sources agree (+15) / single (+0) / conflict (-15); designated domain (+10) / adjacent (+0) / outside (-10); reproducible (+10) / maybe (+0) / unlikely (-10). Calibrated = clamp(Raw + adjustments, 0, 100).
Report: "Raw: 85% → Calibrated: 70% (single T2, no corroboration)"

**Abstention `[GAP-11]`:** If CALIBRATED confidence < 50%, rate UNVERIFIABLE rather than guessing. "UNVERIFIABLE (Calibrated: 35%) — insufficient evidence: only T3, no corroboration."

**Tool Chain `[GAP-4]`:** Different agents use different primary tools:
- PubMed-primary: A1, B5
- WebSearch-primary: B1, C2, C4
- Filesystem-primary: A5, B4
- WebFetch-primary: A3, B6
- Cross-tool: A2, A4, B2, B3
All agents CAN use all tools; START with designated primary for evidence diversity.

---

## Wave 0 — SAFE Claim Decomposition (1 agent)

Launch 1 agent (Task tool, `subagent_type: general-purpose`):

**Agent 0: Claim Decomposer**
```
You are a SAFE (arXiv:2403.18802) claim decomposition specialist. Read [TARGET] and decompose the ENTIRE document into atomic, independently verifiable facts:

1. DECOMPOSE every sentence into individual claims. "Published 15 papers with 1,200+ citations in top-tier journals" → THREE facts.
2. DECONTEXTUALIZE — replace pronouns, resolve references.
3. CATEGORIZE: QUANTITATIVE | PUBLICATION | TEMPORAL | CREDENTIAL | TECHNICAL | LINK | COMPARATIVE | NARRATIVE
4. DISAMBIGUATION CHECK [GAP-2]: If multiple plausible interpretations, flag as AMBIGUOUS, list interpretations, only decompose highest-confidence one. If none >70% confidence, keep compound and flag for manual review.
5. QUALITY CHECK: Completeness, Correctness, Atomicity.
6. NUMBER sequentially (F001, F002, ...).
7. TRACK PROVENANCE [GAP-7]: Record original sentence (line/paragraph). Facts from same sentence share a provenance group ID (PG001, PG002, ...) for logical composition checking.

Output: `F001 [CATEGORY] (PG001) "claim text" — Source: paragraph N, line M`
Report: total facts, category breakdown, ambiguous claims flagged, difficulties.
```

Wait for completion. Fact list → Wave 0.5.

---

## Wave 0.5 — Decomposition Quality Gate (2 parallel agents) `[GAP-2, GAP-12]`

Launch both simultaneously (Task tool, `subagent_type: general-purpose`):

**Agent 0.5a: Disambiguation & Atomicity Auditor `[GAP-2]`**
```
Audit decomposition quality using Claimify (ACL 2025) and Decomposition Dilemmas (NAACL 2025) insights. You have Wave 0 fact list and [TARGET].

For each fact:
1. ATOMICITY: Truly one verifiable claim? If 2+, flag for re-decomposition.
2. DISAMBIGUATION: Multiple plausible interpretations? Flag with confidence per interpretation.
3. MEANING PRESERVATION: Compare against provenance group — did decomposition change meaning?
4. OVER-DECOMPOSITION: Too fine-grained to verify independently?

Output: PASS/FAIL per fact. If >15% fail, recommend re-running Wave 0.
```

**Agent 0.5b: Recall Auditor `[GAP-12]`**
```
Using VeriFact/FactRBench insight: "high precision ≠ high recall." Read [TARGET] independently, paragraph by paragraph. For each, list every verifiable claim YOU identify. Cross-reference against Wave 0 list.

Focus on: implicit claims (timelines implying duration, lists implying counts), claims in visualizations/data structures/footnotes/tooltips, hedged but verifiable claims, multi-sentence claims.

Output: Missed facts (M001, M002, ...), recall estimate = Wave0/(Wave0+missed)×100.
If recall <85%, recommend adding missed facts before proceeding.
```

Wait for both. Merge corrections, add missed high-priority facts, update numbering. Corrected list → all subsequent waves.

---

## Run 1 Waves (Foundational Verification)

### Wave A — Source Truth (5 parallel agents)

Launch all 5 via Task tool, each receiving quality-checked fact list.

**A1: Citation Existence & Support Verifier `[GAP-1]`**
```
Using GhostCite and SourceCheckup insights. Verify EVERY [PUBLICATION] fact:

EXISTENCE: Resolve DOI/PubMed/Semantic Scholar. Check GhostCite patterns: generic author names, plausible-but-nonexistent journals, round citation counts, recent-year clustering (2024-2026 highest risk).

SUPPORT: For each existing citation, read abstract (minimum) or full text. Rate: SUPPORTS | PARTIALLY SUPPORTS | DOES NOT SUPPORT | CONTRADICTS.

AUTHORSHIP: Verify position claims, title truncations, wording changes.

Apply [VERACITY_SCALE], [CALIBRATION], [ABSTENTION], [OUTPUT_FORMAT].
PRIMARY TOOL: PubMed API.
Include support assessment in evidence chain.
```

**A2: Numerical Claims Auditor**
```
Verify EVERY [QUANTITATIVE] fact:
1. Internal consistency across decomposed facts
2. Cross-reference against data files, databases, source code
3. Arithmetic (sub-totals sum? percentages match?)
4. Rounding inconsistencies

Apply [VERACITY_SCALE], [CALIBRATION], [ABSTENTION], [THINK_VERIFY].
Number tiers — T1: original data/code, T2: paper methodology, T3: abstracts/news.
```

**A3: Link & URL Validator**
```
Verify EVERY [LINK] fact and all URLs:
1. WebFetch to verify resolution (LinkedIn 999 = UNVERIFIABLE, not FALSE)
2. Destination content matches claims
3. Internal anchor links match element IDs
4. Downloadable files exist
5. Trace obfuscated/JS-constructed URLs

Apply [VERACITY_SCALE], [CALIBRATION], [ABSTENTION]. Report every link with HTTP status.
PRIMARY TOOL: WebFetch.
```

**A4: Timeline, Date & Temporal Validity `[GAP-9 ENHANCED]`**
```
Using ChronoFact insight: 82% of temporal errors involve IMPLICIT temporal info.

TIMELINE COHERENCE: Verify EVERY [TEMPORAL] fact:
1. Anachronisms — work before holding position?
2. Duration claims vs computed date ranges
3. Publication years: online-first vs print
4. Educational timeline: sequential and plausible?
5. "Current" claims vs today's date

TEMPORAL VALIDITY [GAP-9]: For EVERY fact describing current state, check: IS THIS STILL TRUE TODAY? Flag TEMPORAL DECAY RISK for facts likely true when written but possibly changed.

Apply [VERACITY_SCALE], [CALIBRATION], [ABSTENTION], [THINK_VERIFY].
PRIMARY TOOL: Cross-tool (WebSearch for currency, PubMed for publication dates).
```

**A5: Data File Verifier**
```
Verify EVERY [TECHNICAL] fact referencing files, directories, databases, repos:
1. Glob/Bash to verify existence
2. File counts vs claimed
3. SQLite record counts
4. Code array lengths vs claimed
5. "Current" stats vs actual disk state

Apply [VERACITY_SCALE], [CALIBRATION], [ABSTENTION]. Report exact vs claimed with paths.
PRIMARY TOOL: Filesystem (Glob, Read, Bash).
```

### Wave B — Framing, Integrity & Independent Verification (6 parallel agents)

Wait for Wave A. Incorporate findings; flag DISAGREED/MIXED/UNVERIFIABLE for extra attention.

**B1: Overclaiming Detector**
```
Focus on EVERY [COMPARATIVE] and [NARRATIVE] fact:
1. "Pioneered" — first, or building on prior work?
2. "Discovered" — genuine or data analysis finding?
3. "First to..." — independently verifiable?
4. "Adopted by..." — direct or loose connection?
5. Impact claims — verify mechanism

"Led" fine if PI, "Developed" fine if wrote code. Suggest defensible alternatives.
Also review Wave A MIXED/UNVERIFIABLE: [WAVE_A_DISPUTED_FACTS]
Apply [CALIBRATION], [ABSTENTION]. PRIMARY TOOL: WebSearch.
```

**B2: Consistency Cross-Checker**
```
WITHIN-DOCUMENT self-contradiction check:
1. All facts referring to same underlying claim
2. Numbers, wording, framing consistency across instances
3. JS data arrays vs prose
4. Visualization data vs text

MIXED if both could be true; MOSTLY FALSE if one clearly wrong. Report with fact numbers.
Apply [CALIBRATION], [ABSTENTION].
```

**B3: Missing Information Detector**
```
Identify MISSING from [TARGET]:
1. Claims without evidence  2. Timeline gaps  3. Omitted caveats
4. Asymmetric detail  5. Suppressed negatives (failures, retractions)

Create M001, M002, ... as [MISSING], rated UNVERIFIABLE. Apply [CALIBRATION], [ABSTENTION].
```

**B4: HTML/Code Quality Checker**
```
If HTML: broken entities, malformed tags, CSS refs, JS errors, accessibility, responsive, print.
If not HTML: check referenced code files.
Report with line numbers, severity: CRITICAL/HIGH/MEDIUM/LOW.
Apply [CALIBRATION], [ABSTENTION]. PRIMARY TOOL: Filesystem.
```

**B5: Credential & Identity Verifier**
```
Verify EVERY [CREDENTIAL] fact:
1. Position at institution (WebSearch)  2. Degrees and programs exist
3. ORCID/Scholar/LinkedIn consistency  4. Grants/fellowships  5. Email matches affiliation

Credential tiers — T1: registrar/dept/ORCID, T2: LinkedIn/Scholar, T3: news.
Apply [VERACITY_SCALE], [CALIBRATION], [ABSTENTION]. PRIMARY TOOL: PubMed + WebSearch.
```

**B6: Blind Re-Derivation & Stochastic Consistency `[GAP-5, GAP-6]`**
```
Implements CoVe (Meta, ACL 2024) + SelfCheckGPT stochastic consistency.
CRITICAL: Do NOT read [TARGET]. Receive ONLY entity name, categories, and high-stakes fact IDs (without content).

PART 1 — BLIND RE-DERIVATION [GAP-6]:
Using ONLY entity name + categories, independently research via WebSearch/PubMed/WebFetch.
Write independent findings WITHOUT having seen document claims.
Then receive actual fact list and compare. Discrepancies are EXTREMELY high-signal.

PART 2 — STOCHASTIC CONSISTENCY [GAP-5]:
For each high-stakes fact: generate 3 independent versions of the claim.
Compare against document: 3/3 agree = HIGH consistency (likely true), 2/3 = MEDIUM (check further), ≤1/3 = LOW (likely hallucinated).

Apply [CALIBRATION], [ABSTENTION]. PRIMARY TOOL: WebFetch + WebSearch (NO filesystem — must be blind).

Launch in TWO phases: Phase 1 gets entity+categories only. Phase 2 gets fact list for comparison.
```

### Wave C — Adversarial Review, Composition & Synthesis (6 parallel agents)

Wait for A+B. Compile findings. DISAGREED facts → debate protocol.

**C1: Devil's Advocate**
```
Hostile reviewer with all prior findings [ALL_PRIOR_FINDINGS]:
1. Attack weakest facts (lowest calibrated confidence)
2. Skeptical domain expert challenges
3. Patterns of exaggeration across [COMPARATIVE] facts
4. Narrative honesty assessment
5. Three toughest reviewer questions
Harsh but fair. Apply [CALIBRATION], [ABSTENTION].
```

**C2: Plagiarism & Originality Checker**
```
For every substantial text block: web search for matches, method descriptions vs abstracts
(self-plagiarism = flag but OK), project descriptions vs actual repos, templated language.
Report concerns with matching URL. Apply [CALIBRATION], [ABSTENTION]. PRIMARY TOOL: WebSearch.
```

**C3: Adversarial Debate Judge with Conflict Resolution `[GAP-8]`**
```
Using Tool-MAD + CONFACT (IJCAI 2025). Identify DISPUTED facts (agents disagree or calibrated confidence <50%).

STANDARD DEBATE: PRO (cite verifier) vs CON (cite flagger), weigh by source tier.

CONFLICT RESOLUTION [GAP-8]: When agents present CONTRADICTING evidence:
a. List all sources with tiers
b. T1 outweighs T2 outweighs T3
c. "Two independent T1 sources" rule for TRUE when evidence conflicts
d. Check source independence (derived from same primary?)
e. Genuine irreconcilable conflict → DISPUTED, not forced verdict

Final verdict (6-point scale) + calibrated confidence. If >25% disputed, flag document-level concern.
Apply [CALIBRATION], [ABSTENTION].
```

**C4: Comparative Claims Assessor**
```
Check all [COMPARATIVE] facts: "first"/"largest"/"novel" claims, adoption/influence claims,
implied uniqueness, sample sizes vs field norms, effect sizes supporting narrative.
WebSearch for prior work predating claims. Apply [VERACITY_SCALE], [CALIBRATION], [ABSTENTION].
PRIMARY TOOL: WebSearch.
```

**C5: Weighted Synthesis & Calibrated Consensus `[GAP-3, GAP-10]`**
```
Using FREE-MAD, Beyond Majority Voting, Voting vs Consensus insights. ALL findings: [ALL_PRIOR_FINDINGS].

STEP 1: Collect all ratings + CALIBRATED confidence per fact.

STEP 2 — WEIGHTED AGGREGATION [GAP-10]:
- QUANTITATIVE/TEMPORAL facts: VOTING weighted by calibrated confidence (better for reasoning)
- NARRATIVE/COMPARATIVE facts: CONSENSUS 75% WEIGHTED agreement (better for knowledge)
  - Domain-relevant agents get 2x weight (A1 on PUBLICATION, B1 on COMPARATIVE, A4 on TEMPORAL, B5 on CREDENTIAL)
- Other facts: Weighted supermajority 75% threshold

STEP 3 — TRAJECTORY [GAP-10]: Rating changes across waves are more informative than static. Agent that updated on new evidence → weight HIGHER. Agent that maintained despite contradiction → weight LOWER.

STEP 4 — CALIBRATION QUALITY [GAP-3]: Flag when avg calibrated is >20pts below avg raw (overconfidence), calibrated varies >40pts across agents, or B6 contradicts majority.

STEP 5: Categorize — FALSE→CRITICAL, MOSTLY FALSE→HIGH, MIXED→MEDIUM, MOSTLY TRUE→LOW, TRUE→VERIFIED, irreconcilable→DISPUTED [GAP-8]. Provide exact fix for CRITICAL/HIGH.

STEP 6: Executive summary: total facts, breakdown, % verified, confidence 0-100, calibration quality score, consensus rate, top 3 concerns, B6 agreement rate.
```

**C6: Logical Composition Verifier `[GAP-7]`**
```
Using LoCal (ACM Web Conference 2025). You have fact list with provenance groups, all results, [TARGET].

For each provenance group (facts from same original sentence):
1. COMPOSITION: Do verified sub-facts logically entail the original? (e.g., "Led a team of 5 that published 3" — "Led" never verified = orphan claim)
2. COUNTERFACTUAL: Flip one sub-fact to FALSE — does original still hold? If yes, something was missed.
3. ORPHAN CLAIMS: Causal ("because"), relational ("led", "managed"), conditional ("if") aspects uncovered by sub-facts.
4. EMERGENT CLAIMS: Combined sub-facts create an implied claim never verified.

Output per group: Original sentence, sub-facts+ratings, verdict (ENTAILED/PARTIAL/NOT ENTAILED), orphans, emergent claims, recommendation.
Apply [CALIBRATION], [ABSTENTION].
```

---

## Run 2+ Waves

Reuse Wave 0 (refresh if changed), re-run Wave 0.5, shift perspectives. All agents receive [CALIBRATION] and [ABSTENTION].

### Wave D — Domain Expert Simulation (5 agents)

Adapt domain experts to the target document's subject matter. Example defaults:
- D1: Subject-matter domain expert — methodology, sample sizes, statistics
- D2: Technical/ML researcher — technical claims, benchmarks, implementation details
- D3: Adjacent-field expert — cross-disciplinary connections, broader context
- D4: Standards reviewer — evidence standards, citation norms, formatting
- D5: Investigative journalist — two-source rule, public records verification

### Wave E — Regression & Drift Detection (5 agents)
- E1: Internal document consistency (cross-references, embedded data vs standalone)
- E2: Quantitative data consistency (counts, percentages, totals)
- E3: Cross-section self-consistency (fact numbering, ordering)
- E4: Current vs prior versions (git history, changelogs)
- E5: Stale data detection (dates, links, version numbers)

### Wave F — Red Team (5 agents)
- F1: Prior work invalidating "first"/"pioneered"/"novel" claims
- F2: Methodology claims vs actual implementation
- F3: Quoted sources vs original archives
- F4: Repository/artifact contents vs documented claims (not just existence)
- F5: Retractions, corrections, errata for cited works

### Wave G — Meta-Analysis (5 agents, Run 3+)
- G1: Cross-run agreement — consistently flagged facts
- G2: Confidence trends (calibrated) across runs
- G3: Evidence gaps — facts still lacking T1 after 2 runs
- G4: Fix verification — applied fixes resolved issues?
- G5: New fact discovery — missed in prior Wave 0 passes

### Wave H — Stress Testing (5 agents, Run 3+)
- H1: Steelman/strawman — harder disproof of VERIFIED facts
- H2: Context checker — true in specific context or only general?
- H3: Temporal validity (deep) — additional sources beyond A4
- H4: Scope creep — claims within evidence support?
- H5: Statistical rigor — significant figures, rounding, units

### Wave I — Final Cross-Run Synthesis (5 agents, Run 3+)
- I1: Master consensus — weighted supermajority across ALL agents/runs
- I2: Reliability assessor — 0-100 with breakdown
- I3: Fix prioritizer — impact-to-effort ratio
- I4: Pattern reporter — systematic patterns
- I5: Executive summary — 1-page for decision-makers

**Convergence**: Stop when delta < 3 for two consecutive runs and no CRITICAL/HIGH remain. Most converge after 3-4 runs.

---

## Inter-Run Review Protocol

Between runs (except final and single-run), present findings for user review.

### Phase 1: Summary Card
```
═══════════════════════════════════════════════════
 RUN [R]/[N] COMPLETE       Score: [score]/100
 Facts: [N]   Agents: [N]   Consensus: [N]%
 Calibration: [ratio]   B6 agreement: [N]%
 CRITICAL [N] | HIGH [N] | MEDIUM [N] | LOW [N] | DISPUTED [N]
 [total] fixes available
═══════════════════════════════════════════════════
```

### Phase 2: Severity-Tier Walkthrough

Process: CRITICAL → HIGH → MEDIUM → LOW → DISPUTED. List findings (claim, evidence, source, location, calibrated confidence, fix) then `AskUserQuestion`:

- **CRITICAL/HIGH**: Accept All | Review Each | Defer All
- **MEDIUM**: Neutral (no recommendation bias)
- **LOW**: Accept All (Recommended) | Review Each | Defer All
- **DISPUTED**: Review Each (Recommended) — needs human judgment | Defer All

If "Review Each": **ACCEPT** / **MODIFY** / **REJECT** / **DEFER**.

### Phase 3: Decision Summary
```
ACCEPTED: [N]  MODIFIED: [N]  REJECTED: [N]  DEFERRED: [N]
Preview: Line [N]: "old" → "new" ...
→ Apply fixes and continue  |  → Go back and revise
```
Confirm via `AskUserQuestion`.

### Phase 4: Apply
1. Sort by line number descending  2. Apply via Edit  3. Save snapshots  4. Pass deferred to next run

### Disposition Model
| ACCEPT | Applied immediately, verified next run |
| MODIFY | Applied with user text, verified next run |
| REJECT | Dropped permanently |
| DEFER | Flagged for next run |

### Convergence & Skip
- Convergence requires runs>=4 to observe two consecutive deltas. After Run 4+: delta < 3 two consecutive + no CRITICAL/HIGH → offer early stop. Default runs=3 completes all runs without early stopping.
- `runs=1`: Report only  |  0 findings: Auto-proceed  |  Final run: Report only

---

## Execution Steps

### Steps 1-2: Parse & Read
Extract target + run count from `$ARGUMENTS`. Read full document.

### Step 3: Execute Runs

For each run R (1 to N):

**3a. EXECUTE**: Wave 0 → Wave 0.5 (quality gate) → merge corrections → 3 waves of agents per Architecture. Later waves receive prior findings. Run 2+ agents receive deferred findings. B6 launches in TWO phases (Phase 1: entity+categories only; Phase 2: fact list for comparison).

**3b. SYNTHESIZE**: Score, delta, severity sort. Also: calibration quality `[GAP-3]`, B6 agreement `[GAP-6]`, composition failures from C6 `[GAP-7]`.

**3c. REVIEW**: Skip if final, runs=1, or 0 findings. Otherwise follow Inter-Run Review Protocol.

**3d. APPLY**: Sort descending, Edit, save snapshots.

**3e. CARRY FORWARD**: Re-read, compile deferred, log per-run entry.

**3f. CONVERGENCE**: After Run 2+, delta < 3 two consecutive + no CRITICAL/HIGH → offer early stop.

### Step 4: Consolidate

After all runs:
1. Collect all ratings across agents/runs
2. Weighted supermajority consensus (75% WEIGHTED by calibrated confidence + domain relevance); else debate judge with conflict resolution `[GAP-3, GAP-10]`
3. Map: FALSE→CRITICAL, MOSTLY FALSE→HIGH, MIXED→MEDIUM, MOSTLY TRUE→LOW, TRUE→VERIFIED, UNVERIFIABLE→UNVERIFIABLE, irreconcilable→DISPUTED
4. Score = (TRUE + MOSTLY TRUE) / checkable × 100 (exclude UNVERIFIABLE)
5. Track finding lifecycle (discovered Run X, fixed Y, verified Z)
6. Aggregate dispositions
7. **v2 metrics**: decomposition quality (% passed Wave 0.5) `[GAP-2]`, recall `[GAP-12]`, calibration quality `[GAP-3]`, B6 agreement `[GAP-6]`, logical composition rate `[GAP-7]`, conflicts resolved `[GAP-8]`, abstention rate `[GAP-11]`

### Step 5: Final Report

```
## Veracity Audit Report (v2 Enhanced)
**Target**: [file/URL] | **Agents**: [N] | **Runs**: [N] | **Facts**: [N]
**Veracity confidence**: [N]% | **Consensus rate**: [N]%

### v2 Enhancement Metrics
- Decomposition quality: [N]% passed QA | Recall: [N]% (missed added: [N])
- Calibration quality: [ratio] (1.0=perfect, <0.8=overconfident)
- Blind re-derivation agreement: [N]% | Logical composition: [N]% entailed
- Conflicts resolved: [N] via CONFACT | Abstention rate: [N]%

### Session Progression
| Run | Score | Delta | CRIT | HIGH | MED | LOW | DISP | Fixes |
|-----|-------|-------|------|------|-----|-----|------|-------|
Dispositions: Accepted [N] | Modified [N] | Rejected [N] | Deferred [N]

### Veracity Distribution
TRUE [N] | MOSTLY TRUE [N] | MIXED [N] | MOSTLY FALSE [N] | FALSE [N] | UNVERIFIABLE [N] | DISPUTED [N]

### CRITICAL — must fix
F### [CAT] "claim" — Evidence: ... | Calibrated: N% | Agents: N/M (weighted) | B6: MATCH/MISMATCH | Fix: ...

### HIGH | MEDIUM | DISPUTED (conflicting evidence, needs human judgment) | LOW | VERIFIED | UNVERIFIABLE

### Logical Composition Issues [GAP-7]
PG###: "original sentence" — Sub-facts: F### (rating), ... — Verdict: ENTAILED/PARTIAL/NOT ENTAILED — Orphans/emergent claims

### Evidence Quality
T1: [N] | T2: [N] | T3: [N] | 2+ independent: [N] ([%])
```

### Step 6: Log Results

**6a. Metadata**: `git remote get-url origin` (fallback: dirname), `pwd`, `git branch --show-current`, `git rev-parse --short HEAD`, generate session UUID.

**6b. Per-run entry** (after each run's review):
```json
{
  "id": "<timestamp>-run<N>", "type": "run", "version": "v2-experimental",
  "session_id": "<UUID>", "run_number": 1, "timestamp": "<ISO-8601>",
  "target": "...", "project": "...", "project_path": "...",
  "branch": "...", "commit_sha": "...",
  "veracity_score": 0, "prior_score": null, "score_delta": null,
  "agents_deployed": 20,
  "claims_total": 0, "claims_verified": 0, "claims_flagged": 0,
  "claims_unverifiable": 0, "claims_disputed": 0, "consensus_rate": 0,
  "veracity_distribution": { "true": 0, "mostly_true": 0, "mixed": 0, "mostly_false": 0, "false": 0, "unverifiable": 0, "disputed": 0 },
  "severity_counts": { "critical": 0, "high": 0, "medium": 0, "low": 0, "verified": 0, "unverifiable": 0, "disputed": 0 },
  "evidence_quality": { "tier1_sources": 0, "tier2_sources": 0, "tier3_sources": 0, "multi_source_claims": 0 },
  "fact_categories": { "quantitative": 0, "publication": 0, "temporal": 0, "credential": 0, "technical": 0, "link": 0, "comparative": 0, "narrative": 0, "missing": 0 },
  "v2_metrics": {
    "decomposition_quality_pct": 0, "decomposition_recall_pct": 0, "missed_facts_added": 0,
    "calibration_quality": 0.0, "blind_rederivation_agreement_pct": 0,
    "logical_composition_pct": 0, "conflicts_resolved": 0, "abstention_rate_pct": 0,
    "stochastic_consistency_high_pct": 0, "stochastic_consistency_low_count": 0
  },
  "review_decisions": { "critical_batch": "...", "high_batch": "...", "medium_batch": "...", "low_batch": "...", "disputed_batch": "..." },
  "findings": [{
    "fact_id": "F###", "severity": "...", "veracity_rating": "...",
    "raw_confidence": 0, "calibrated_confidence": 0,
    "category": "...", "claim": "...", "evidence_summary": "...",
    "source_tier": "T#", "source_url": "...",
    "citation_support": "supports|partially|does_not_support|contradicts|n_a",
    "blind_rederivation": "match|mismatch|partial|not_checked",
    "stochastic_consistency": "high|medium|low|not_checked",
    "logical_composition": "entailed|partial|not_entailed|n_a",
    "agents_agreed": "N/M", "weighted_agreement": 0,
    "location": "...", "fix": "...", "status": "...", "disposition": null
  }]
}
```

**6c. Session entry**: Same base fields plus `type: "session"`, `runs_requested`, `runs_completed`, `score_progression[]`, `converged`, `disposition_summary`, `runs[]`, aggregated `v2_metrics`. For `runs=1`: single `type: "run"`, `session_id: null`. Entries without `type` = legacy. `version: "v2-experimental"` enables dashboard filtering.

**6d.** Append to `~/.claude/veracity-log.json` (append-only).

**6e. Paper trail:**
```
~/.claude/audit-trails/veracity/<timestamp>_<target>/
├── session_metadata.json, consolidated_report.md
└── run_N/  (one per run)
    ├── metadata.json, target_snapshot_{pre,post}.md
    ├── wave0_decomposition.md
    ├── wave0.5a_disambiguation.md, wave0.5b_recall_audit.md  [GAP-2,12]
    ├── wave{A,B,C}_agent{1-5}.md, waveB_agent6.md [GAP-5,6], waveC_agent6.md [GAP-7]
    ├── atomic_facts.json, atomic_facts_corrected.json [GAP-2,12]
    ├── disputed_facts.json, conflict_resolution.json [GAP-8]
    ├── blind_rederivation.json [GAP-6], logical_composition.json [GAP-7]
    ├── review_decisions.json, fixes_applied.json, consolidated_report.md
    └── deferred_findings.json  (Run 2+)
```
After each run: write run_N/ files. After all: write session files.
Commit: `cd ~/.claude/audit-trails && git add -A && git commit -m "veracity-v2: <target> <timestamp> score=<N>/100 (<M> runs)"`

### Step 7: Dashboard Update

1. Read `~/.claude/veracity-log.json` and `~/.claude/audit-dashboard/index.html`
2. Replace `<script id="veracity-data">...</script>` with: `const VERACITY_DATA = <JSON>;`
3. Write updated `index.html`
4. Tell user: **"Dashboard updated. Open `~/.claude/audit-dashboard/index.html` (Veracity 555 tab)."**

Data inlined because browsers block external scripts via `file://`.

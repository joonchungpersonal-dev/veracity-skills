---
name: readme-audit
description: Audit a README against its actual project state. Decomposes README into atomic claims, then verifies each against repo files, configs, code, git history, and live URLs. Launches 17 agents per run (1 decomposition + 1 structural scan + 3 waves of 5 agents). Default 1 run; use runs=3 for thorough audit.
user-invocable: true
argument-hint: <path-to-repo> [runs=1]
---

# README Audit — Project-Grounded Fact-Check

Decomposes a README into atomic claims about the project, then verifies each claim against the actual repository state: files on disk, package manifests, code exports, git history, dependency locks, and live URLs. Built on veracity-tweaked-555 architecture (SAFE decomposition + parallel verification waves) but domain-specialized for README drift detection. Shares wave structure, SAFE decomposition, supermajority consensus, inter-run review protocol, and logging schema pattern with veracity-tweaked-555; the verification agents (A1-A5, B1-B5, C1-C5) are README-specific.

## Methodology

**Published methods (inherited from veracity-tweaked-555):**
- **SAFE — Search-Augmented Factuality Evaluator (Google DeepMind, arXiv:2403.18802)**: Decompose document into atomic, self-contained facts (decompose + revise to be self-contained) before verification
- **6-Point Veracity Scale**: TRUE / MOSTLY TRUE / MIXED / MOSTLY FALSE / FALSE / UNVERIFIABLE (adapted from PolitiFact's 6-tier Truth-O-Meter: MIXED replaces HALF TRUE for technical neutrality; separately, PANTS ON FIRE is dropped as inapplicable to repo-grounded verification and UNVERIFIABLE is added to cover claims that cannot be confirmed or denied against repo state — these are distinct modifications, not parallel substitutions)
- **Tool-MAD (arXiv:2601.04742)**: Multi-Agent Debate with tool augmentation — agents with different retrieval tools (e.g., RAG vs web search) debate before a judge — up to 5.5% accuracy improvement over prior SOTA (MADKE on FEVEROUS)

**README-specific practices (custom):**
- **Repo-grounded verification**: Every claim checked against files on disk, not just web searches
- **Command verification**: Install/build/usage commands verified by source analysis — parsing manifests, checking registries, and tracing code paths (no code execution)
- **Manifest cross-reference**: README claims compared against package.json, pyproject.toml, Cargo.toml, setup.cfg, go.mod, etc.
- **Git history drift detection**: Claims compared against latest release tags, recent commits, and changelog
- **Staleness scoring**: Time since last README edit vs. time since last substantive code change

## Input

`$ARGUMENTS`: path to repo root (containing README), optionally `runs=N` (default: 1). If no target, ask user. For monorepos, point to the specific package directory containing the target README. Assumes UTF-8 encoding; non-English READMEs are supported but agent prompts are English-optimized.

The skill will auto-detect the README file (priority order): `README.md` > `Readme.md` > `readme.md` > `README.rst` > `README.txt` > `README` > `README.markdown` > `README.adoc`. On case-insensitive filesystems, the first match wins.

## Architecture

Each **run** = 1 decomposition + 1 structural scan + 3 waves of 5 agents = 17 agents/run. (Within each wave, the 5 agents run in parallel. Waves are sequential: A0 runs after Wave 0 completes; then A completes before B launches, B completes before C launches. In Wave C, C1-C4 run in parallel, then C5 runs after all four complete.)
- **Run 1**: Wave 0 → Waves A, B, C (ground truth, functional/behavioral, adversarial/synthesis)
- **Run 2**: Wave 0 (refresh) → Waves D, E, F (user persona simulation, drift detection, red team)
- **Run 3+**: Wave 0 (refresh) → Waves G, H, I (meta-analysis, stress test, synthesis)
- **Run 4+**: Cycle (4→A/B/C, 5→D/E/F, 6→G/H/I) with all prior context. Cycled waves receive findings from ALL previous runs; template variables like [ALL_PRIOR_FINDINGS] and [WAVE_A_DISPUTED_FACTS] are populated from the most recent run that executed those waves.

Total agents = runs x 17.

## Claim Categories (README-specific)

| Category | What it covers | Primary verification source |
|----------|---------------|---------------------------|
| VERSION | Version numbers, compatibility ranges, "supports X.Y+" | Package manifest, git tags, CI matrix |
| INSTALL | Install commands, prerequisites, system requirements | Parse manifest, check registry, trace code paths |
| USAGE | Code examples, CLI examples, API usage snippets | Parse against actual exports/signatures |
| FEATURE | Feature claims, capability descriptions, "supports X" | Grep codebase for implementation |
| DEPENDENCY | Dependency lists, peer deps, optional deps | Lock file, manifest, import analysis |
| ARCHITECTURE | File layout, directory structure, module descriptions | Glob actual filesystem |
| BADGE | Build status, coverage %, version badges, license badge | Badge URL resolution, CI status, manifest |
| CONFIG | Config examples, env vars, defaults, option descriptions | Config schemas, .env.example, code defaults |
| LINK | URLs, relative paths, anchors, cross-references | HTTP resolution, file existence, anchor check |
| TEMPORAL | "Current", "latest", "recently", dates, "maintained" | Git log, last commit date, release dates |
| LICENSE | License claims, SPDX identifiers | LICENSE file, manifest license field |
| CONTRIBUTOR | Contributor claims, maintainer info, author credits | git log, CONTRIBUTORS file, package author field |

## Shared Definitions

When constructing subagent prompts, include these definitions wherever agents reference [VERACITY_SCALE], [EVIDENCE_CHAIN], [THINK_VERIFY], or [OUTPUT_FORMAT].

**Veracity Scale (6-point):**
- **TRUE**: Claim matches repo state exactly
- **MOSTLY TRUE**: Claim is accurate with minor discrepancy (e.g., version 2.3.1 vs 2.3.2)
- **MIXED**: Partially accurate (e.g., install command works but missing a required flag)
- **MOSTLY FALSE**: Significant inaccuracy (e.g., claims Python 3.8+ but requires 3.10+)
- **FALSE**: Directly contradicts repo state (e.g., file doesn't exist, command fails)
- **UNVERIFIABLE**: Cannot confirm or deny (e.g., claims about runtime performance)

**Source Tiers (repo-grounded):**
- **T1 (Ground truth)**: Files on disk, package manifests, lock files, git tags, code analysis
- **T2 (Derived)**: Git log patterns, CI config, test output, badge URLs
- **T3 (External)**: Web searches, registry pages, linked documentation

**Evidence Chain (required per rated claim):** Source file/path/command | Source tier | Actual value found | How it confirms/contradicts the README claim

**Output Format:**
```
F### [CATEGORY] — **RATING** (Confidence: N%)
  Claim: "..." | Actual: ... | Source: path/file (Tier N) | Fix: ...
```

**Think & Verify:** Before marking FALSE, run the check twice. Before marking TRUE, attempt to find a contradicting source.

---

## Wave 0 — README Claim Decomposition (1 agent)

Launch 1 agent (Task tool, `subagent_type: general-purpose`):

**Agent 0: README Claim Decomposer**
```
You are a README claim decomposition specialist (adapted from SAFE, arXiv:2403.18802).

STEP 1 — DISCOVER THE README:
In [REPO_PATH], find the README file. Check in order: README.md, Readme.md, readme.md, README.rst, README.txt, README, README.markdown, README.adoc. Read the full file.
If NO README is found, STOP and report: "No README found in [REPO_PATH]. Checked: README.md, Readme.md, readme.md, README.rst, README.txt, README, README.markdown, README.adoc. Cannot proceed with audit."

STEP 2 — DISCOVER MANIFEST FILES:
Check for and read (if present): package.json, pyproject.toml, setup.cfg, setup.py, Cargo.toml, go.mod, Gemfile, pom.xml, build.gradle, composer.json, mix.exs, pubspec.yaml. These provide ground truth for many claims.

STEP 3 — DECOMPOSE every section of the README into atomic, independently verifiable claims:
- "Supports Python 3.8+ and Node 16+" → TWO facts: Python >=3.8, Node >=16
- "Install with `pip install foo`" → claim that pip install command works
- "See the `/docs` folder for API reference" → claim that /docs exists and contains API reference
- Badge showing "coverage 95%" → claim that test coverage is 95%

STEP 4 — DECONTEXTUALIZE: Replace pronouns and references. "It supports..." → "[package-name] supports..."

STEP 5 — CATEGORIZE: VERSION | INSTALL | USAGE | FEATURE | DEPENDENCY | ARCHITECTURE | BADGE | CONFIG | LINK | TEMPORAL | LICENSE | CONTRIBUTOR
Note: STALE and MISSING are not decomposition categories — they are created later by B3 (Staleness Detector) and B4 (Completeness Auditor) respectively.

STEP 6 — EXTRACT MANIFEST FACTS: From discovered manifests, extract the actual version, dependencies, license, author, scripts, entry points. These become the ground truth reference for verification waves.

STEP 7 — NUMBER sequentially (F001, F002, ...).

Output:
- Full numbered fact list: `F001 [CATEGORY] "claim text" — Source: section "heading", line N`
- Manifest ground truth summary (actual version, deps, license, etc.)
- Total facts, breakdown by category
- Staleness indicator: days since last README edit vs last code commit (use git log)
```

Wait for completion. The fact list + manifest summary become input for all subsequent waves.

---

## Agent A0 — Structural Integrity Scanner (1 agent)

Runs AFTER Wave 0, BEFORE Wave A. Catches structural and cross-referential errors that individual verification agents typically miss until later runs. Designed from empirical analysis of 35 fixes across 6 audit runs — targets the error types that account for the majority of late-discovered issues.

Launch 1 agent (Task tool, `subagent_type: general-purpose`), receiving the Wave 0 fact list and the full target document:

**Agent A0: Structural Integrity Scanner**
```
You are a structural integrity pre-scanner. Your job is to catch document-level problems BEFORE the specialized verification agents run. Read the full target document [TARGET] and the Wave 0 fact list [FACT_LIST].

Run these 8 checks systematically:

1. PHANTOM INFRASTRUCTURE — Find every reference to external systems (dashboards, tabs, script tags, APIs, databases, CI pipelines). For each, verify the referenced artifact actually exists. Flag any "will be created", "should exist", or aspirational references stated as if already real.

2. CROSS-REFERENCE CONSISTENCY — Find all internal cross-references (section headers referencing other sections, "see X above", "as defined in Y", "respectively" mappings). Verify each reference resolves correctly. For "respectively" or ordered mappings (A and B map to X and Y), confirm the order matches.

3. ACRONYM & TERM EXPANSION — Find every acronym and technical term introduced in the document. Verify: (a) first use includes expansion, (b) expansion is factually correct, (c) usage is consistent throughout. Flag unexpanded acronyms and incorrect expansions.

4. PARALLELISM & SEQUENCE CLAIMS — Find all claims about parallel execution, sequential ordering, or dependencies ("X and Y run in parallel", "A before B", "after C completes"). Map these into a dependency graph. Flag contradictions (e.g., "parallel" in one place but "sequential" in another for the same items).

5. BOUNDARY & SCOPE MARKERS — Find all conditional triggers ("After Run 2+", "for runs > 1", "when N >= 3"). Verify they are consistent with each other and with the described architecture. Flag mismatched thresholds (e.g., convergence requires "2 consecutive deltas" but is triggered "After Run 2+" which only provides 1 delta).

6. ENUMERATION COMPLETENESS — Find all enumerated lists (agent counts, category counts, wave labels, file lists). Verify each enumeration matches its definition. Flag: claimed count differs from actual items listed, items defined but not enumerated, items enumerated but not defined.

7. REPEATED ELEMENTS — Find elements that appear in multiple locations (agent names, formulas, file paths, category lists). Verify all instances are identical. Flag divergences — even minor ones like formatting differences or reordering.

8. TERMINOLOGY CONNOTATION — Find terms with loaded connotations ("executable", "sandbox", "pioneered", "purpose-built"). Flag terms that imply capabilities or properties the document does not support with evidence.

For each finding, report:
- Check number (1-8)
- Location (line/section)
- Severity: CRITICAL (breaks execution), HIGH (misleads), MEDIUM (inconsistent), LOW (cosmetic)
- Description and exact fix

Output summary: total checks run, findings per check, findings by severity.
```

Wait for completion. A0 findings feed into Wave A alongside the fact list — Wave A agents receive both.

---

## Run 1 Waves (Ground Truth + Functional/Behavioral + Adversarial/Synthesis)

### Wave A — Ground Truth Verification (5 parallel agents)

Launch all 5 via Task tool (`subagent_type: general-purpose`), each receiving the Wave 0 fact list, manifest summary, and Agent A0 structural findings.

**A1: Version & Manifest Auditor**
```
Verify EVERY [VERSION], [LICENSE], [DEPENDENCY], and [CONTRIBUTOR] fact against actual project files in [REPO_PATH]:

1. VERSION claims: Compare against package manifest version field, git tags (`git tag --sort=-v:refname | head -10`), and latest release
2. Dependency claims: Cross-reference against lock file (package-lock.json, poetry.lock, Cargo.lock, etc.) and manifest
3. Compatibility ranges: Check CI matrix (.github/workflows/*.yml, .travis.yml, tox.ini) for actually-tested versions
4. LICENSE claims: Compare against LICENSE file content and manifest license field (SPDX identifier)
5. Engine/runtime requirements: Check manifest engines field, python_requires, rust-version, etc.
6. CONTRIBUTOR claims: Compare author/maintainer fields in manifest against README credits. Check git log --format='%aN' | sort -u for actual committers. Verify CONTRIBUTORS/AUTHORS file if referenced.

Apply [VERACITY_SCALE], [EVIDENCE_CHAIN], [THINK_VERIFY], [OUTPUT_FORMAT].
T1 sources: manifest files, lock files, CI configs, git tags, git log.
```

**A2: File & Directory Structure Verifier**
```
Verify EVERY [ARCHITECTURE] and [LINK] fact that references project files/directories:

1. Glob [REPO_PATH] to check every claimed file/directory exists
2. Compare claimed directory structure against actual (`ls -R` key directories)
3. Relative links in README (./docs, ./examples) — do targets exist?
4. Claimed file counts vs actual (e.g., "50+ components" → count them)
5. Image/asset references — do referenced images exist at claimed paths?
6. Anchor links within README — do target headings exist?

Apply [VERACITY_SCALE], [EVIDENCE_CHAIN], [THINK_VERIFY], [OUTPUT_FORMAT].
Report: every path reference with EXISTS/MISSING status.
```

**A3: Code Example & API Verifier**
```
Verify EVERY [USAGE] fact — code examples, import statements, function signatures:

1. Import claims: Do the modules/packages/functions actually exist in the codebase?
   - Grep for class/function definitions matching claimed API
   - Check actual export lists (index.ts, __init__.py, mod.rs, etc.)
2. Function signatures: Do parameters match? Are defaults correct?
3. CLI commands: Does the binary/script exist? Do the flags match (check argparse/clap/commander)?
4. Output examples: Are shown outputs plausible given the actual code?
5. Config examples: Do the shown config keys match actual config schemas?

DO NOT execute code. Verify by reading source files only.
Apply [VERACITY_SCALE], [EVIDENCE_CHAIN], [THINK_VERIFY], [OUTPUT_FORMAT].
```

**A4: Install & Setup Verifier**
```
Verify EVERY [INSTALL] and [CONFIG] fact:

1. Install commands: Does the package exist on the claimed registry? (npm, PyPI, crates.io, etc.)
   - Check manifest "name" field matches claimed install target
   - Check if published (WebFetch registry page if possible)
2. Prerequisites: Are system requirements (Node version, Python version, OS) documented and accurate?
3. Build commands: Do claimed build/dev scripts exist in manifest "scripts" field?
4. Environment variables: Are documented env vars actually read in the code? (grep for process.env, os.environ, std::env)
5. Configuration files: Do documented config file formats match what the code actually parses?
6. Post-install steps: Are they necessary? Are any required steps MISSING?

Apply [VERACITY_SCALE], [EVIDENCE_CHAIN], [THINK_VERIFY], [OUTPUT_FORMAT].
```

**A5: Badge & External Link Verifier**
```
Verify EVERY [BADGE] and external [LINK] fact:

1. Badge URLs: WebFetch each badge image URL — does it resolve?
2. Badge accuracy: npm version badge vs actual manifest version, coverage badge vs actual coverage config
3. CI status badges: Do they point to the correct repo/workflow?
4. External documentation links: Do they resolve and contain relevant content?
5. Homepage/repository URLs in manifest: Do they match README links?
6. Social links, chat links (Discord, Slack): Do invites still work?

Apply [VERACITY_SCALE], [EVIDENCE_CHAIN], [THINK_VERIFY], [OUTPUT_FORMAT].
Report: every URL with HTTP status and content match assessment.
LinkedIn returning 999 = UNVERIFIABLE, not FALSE.
```

### Wave B — Functional & Behavioral Verification (5 parallel agents)

Wait for Wave A. Incorporate findings; flag claims where agents rated MIXED/UNVERIFIABLE.

**B1: Feature Claim Verifier**
```
Verify EVERY [FEATURE] fact — does the codebase actually implement what the README claims?

1. For each feature claim, search the codebase for implementation evidence:
   - Grep for relevant function names, class names, modules
   - Check test files for feature-related tests
   - Look for TODO/FIXME/HACK comments near claimed features (might be incomplete)
2. "Supports X format" → is there actual parsing/handling code for format X?
3. "Plugin system" → is there actually a plugin interface?
4. "Cross-platform" → is there platform-specific code for each claimed platform?
5. Rate deprecated/removed features still in README as MOSTLY FALSE or FALSE

Apply [VERACITY_SCALE], [EVIDENCE_CHAIN], [THINK_VERIFY], [OUTPUT_FORMAT].
Also review Wave A MIXED/UNVERIFIABLE facts: [WAVE_A_DISPUTED_FACTS]
```

**B2: Consistency Cross-Checker**
```
WITHIN-README consistency check:

1. Version mentioned in text vs version in badges vs version in install command
2. Package name consistency (title vs install command vs import examples)
3. Feature list in overview vs feature details in later sections
4. API examples using consistent function names/signatures throughout
5. Prerequisites section vs what install commands actually require
6. Table of contents vs actual headings (if TOC exists)

Apply [VERACITY_SCALE], [EVIDENCE_CHAIN], [THINK_VERIFY], [OUTPUT_FORMAT]. Report with fact numbers (F001 vs F047).
MIXED if ambiguous; MOSTLY FALSE if clearly contradictory.
```

**B3: Staleness Detector**
```
Identify STALE content in the README:

1. Git blame the README — which sections haven't been touched in 6+ months?
2. Compare: features added in recent commits (git log --oneline -30) but NOT in README
3. Deprecated features: still documented? Check for deprecation warnings in code
4. "Coming soon" / "TODO" / "WIP" items — how old are they?
5. Screenshots/GIFs — do they match current UI? (check modification dates of referenced images)
6. Compare README last-modified vs latest release tag date
7. TEMPORAL claims: "currently", "now supports", "recently added" — when was that written?

Create staleness entries (S001, S002, ...) rated by age:
- <3 months: note only
- 3-6 months: LOW
- 6-12 months: MEDIUM
- >12 months: HIGH
```

**B4: Completeness Auditor**
```
Identify what's MISSING from the README that should be there:

1. Standard sections check: Does the README have?
   - Description/overview, Installation, Usage/Quick start, API/Configuration, Contributing, License
2. Manifest has dependencies not mentioned in README (especially peer deps)
3. Scripts in manifest (test, build, lint) with no documentation
4. Environment variables read in code but not documented
5. Entry points / exported functions with no usage examples
6. Error codes or common errors with no troubleshooting section
7. Breaking changes in recent commits with no migration guide

Create missing entries (M001, M002, ...) as [MISSING], rated by importance:
- CRITICAL: Install/usage would fail without this info
- HIGH: Users would commonly need this
- MEDIUM: Nice to have
- LOW: Optional polish
```

**B5: Security & Sensitivity Scanner**
```
Check README for security concerns:

1. Hardcoded tokens, API keys, passwords, or secrets in examples
2. Install commands using curl|bash patterns without verification
3. Permissions advice (chmod 777, running as root) that's overly broad
4. Outdated security-relevant dependencies mentioned
5. Missing security policy reference (SECURITY.md)
6. Example configs with insecure defaults (debug=true, no auth, etc.)
7. Internal URLs, private endpoints, or staging servers leaked in examples

Rate by severity: CRITICAL (real secrets exposed → FALSE), HIGH (insecure patterns → MOSTLY FALSE), MEDIUM (missing best practices → MIXED). These map to the standard veracity-to-severity scheme in C5.
```

### Wave C — Adversarial Review & Synthesis (5 agents)

Wait for A+B. Compile all findings. Launch C1, C2, C3, C4 in parallel. Then launch C5 AFTER C1-C4 all complete (C5 uses C3's debate verdicts and all prior findings for final synthesis).

**C1: Devil's Advocate**
```
Hostile reviewer of the README with all prior findings [ALL_PRIOR_FINDINGS]:

1. Attack the WEAKEST claims (lowest ratings/confidence from prior waves)
2. What would a first-time user stumble on?
3. What would a contributor find misleading?
4. Overall: does this README honestly represent the project's current state?
5. Write the 3 most likely user complaints after following this README
6. Identify the single highest-impact fix

Harsh but fair — improve the README, don't destroy it.
```

**C2: Competing Project Comparator**
```
Check comparative and [FEATURE] claims against the ecosystem (superlatives, "first", "only", "fastest", benchmark claims):

1. "Fastest", "lightest", "most popular", "only tool that..." — verify against alternatives
2. Benchmark claims — are methodologies and versions specified?
3. Star counts, download counts — current or stale?
4. Comparison tables — are competitor descriptions fair and current?
5. "Unlike X, we support Y" — does X now also support Y?

WebSearch for competing projects. Apply [VERACITY_SCALE], [EVIDENCE_CHAIN], [THINK_VERIFY], [OUTPUT_FORMAT].
```

**C3: Adversarial Debate Judge (Tool-MAD)**
```
Using all prior findings [ALL_PRIOR_FINDINGS], identify DISPUTED claims (agents disagree or confidence <60%).

For each disputed claim:
1. PRO argument — cite verifying agent + evidence
2. CON argument — cite flagging agent + evidence
3. Weigh evidence by source tier (T1 repo files > T2 derived > T3 external)
4. FINAL VERDICT (6-point scale) + confidence score

If >25% disputed, flag as document-level concern: "This README has significant drift from project state."
```

**C4: Migration & Changelog Cross-Reference**
```
Check README against project history:

1. CHANGELOG.md / HISTORY.md / releases — do documented changes match README?
2. Breaking changes in recent releases — is README updated for them?
3. Removed features — still documented in README?
4. Renamed APIs/options — README using old names?
5. Major version bumps — does README reflect the new major version's API?

Apply [VERACITY_SCALE], [EVIDENCE_CHAIN], [THINK_VERIFY], [OUTPUT_FORMAT].
```

**C5: Final Synthesis & Consensus**
```
Using ALL prior findings [ALL_PRIOR_FINDINGS]:

1. Collect ALL veracity ratings per claim from every evaluating agent
2. Supermajority consensus: 75%+ agreement = consensus; else use C3 debate verdict
3. Categorize: FALSE→CRITICAL, MOSTLY FALSE→HIGH, MIXED→MEDIUM, MOSTLY TRUE→LOW, TRUE→VERIFIED
4. Merge staleness findings (S###) and missing findings (M###) into severity tiers
5. For each CRITICAL/HIGH finding, provide the EXACT text fix (old → new)
6. Executive summary:
   - README Health Score: (TRUE + MOSTLY TRUE) / checkable claims x 100 (exclude UNVERIFIABLE), 0-100
   - Total claims checked, rating breakdown, % verified
   - Staleness index: days since README edit / max(days since code edit, 1)
   - Completeness score: standard sections present / expected
   - Top 5 fixes by impact
   - Overall assessment: CURRENT / DRIFTING / STALE / UNRELIABLE
```

---

## Run 2+ Waves

Reuse Wave 0 fact list (refresh if README changed between runs), shift perspectives.

> **Note**: Waves D-I provide bullet-point outlines only. For runs=2+, the executing agent should expand each bullet into a full agent prompt following the patterns established in Waves A-C (include [VERACITY_SCALE], [EVIDENCE_CHAIN], [THINK_VERIFY], [OUTPUT_FORMAT] references and repo-grounded verification instructions).

### Wave D — User Persona Simulation (5 agents)
- D1: **Complete beginner** — can they install and run the project from README alone?
- D2: **Experienced developer** evaluating adoption — are claims credible?
- D3: **Contributor** — is the dev setup documentation accurate?
- D4: **Security auditor** — are there red flags in the documented setup?
- D5: **Maintainer picking up the project** — does README match what they'd discover?

Each assigns 6-point ratings from their persona's perspective with evidence chains.

### Wave E — Deep Drift Detection (5 agents)
- E1: **API surface drift** — exported functions/classes vs documented API (exhaustive diff)
- E2: **Config drift** — actual config schema vs documented options (every key)
- E3: **Dependency drift** — lock file versions vs documented/badged versions
- E4: **CI/test drift** — CI config vs documented test/build commands
- E5: **Cross-document drift** — README vs CONTRIBUTING.md vs docs/ vs wiki

### Wave F — Red Team (5 agents)
- F1: Follow install instructions literally — what breaks?
- F2: Run every code example mentally (trace through actual source) — what fails?
- F3: Click every link (WebFetch) — what's dead?
- F4: Compare README to npm/PyPI/crates.io page — which is more accurate?
- F5: Search GitHub issues for "README", "docs", "documentation" — known complaints?

### Wave G — Meta-Analysis (5 agents, Run 3+)
- G1: Cross-run agreement — consistently flagged claims
- G2: Confidence trends across runs
- G3: Evidence gaps — claims still lacking T1 after 2 runs
- G4: Fix verification — did applied fixes resolve issues?
- G5: New claim discovery — README claims missed in prior Wave 0 passes

### Wave H — Stress Testing (5 agents, Run 3+)
- H1: Steelman/strawman — harder disproof of VERIFIED claims
- H2: Context checker — true for which OS/version/environment?
- H3: Temporal validity — still TRUE today vs when written?
- H4: Scope creep — README promises more than code delivers?
- H5: Readability audit — jargon, assumptions, unclear instructions

### Wave I — Final Cross-Run Synthesis (5 agents, Run 3+)
- I1: Master consensus — supermajority across ALL agents/runs
- I2: README Health Score — 0-100 with breakdown by category
- I3: Fix prioritizer — impact-to-effort ratio, sorted
- I4: Pattern reporter — systematic drift patterns (e.g., "all version numbers are one minor behind")
- I5: Executive summary — 1-page audit report for maintainers

**Convergence**: Stop when Health Score delta (|run_N_score - run_{N-1}_score|) < 3 for two consecutive runs and no CRITICAL/HIGH remain.

---

## Inter-Run Review Protocol

Between runs (except final and single-run), present findings for user review before applying fixes.

### Phase 1: Summary Card
```
================================================================
 README AUDIT — RUN [R]/[N]       Health: [score]/100
 Claims: [N]   Agents: [N]   Consensus: [N]%
 CRITICAL [N] | HIGH [N] | MEDIUM [N] | LOW [N] | STALE [N] | MISSING [N]
 Staleness: README [N] days old, code [N] days old
 [total] fixes available
================================================================
```

### Phase 2: Severity-Tier Walkthrough

Process in order: CRITICAL → HIGH → MEDIUM → LOW. For each tier, list all findings with:
- Claim text (from README)
- Actual state (from repo)
- Evidence source and tier
- Suggested fix (exact old → new text)

Then `AskUserQuestion`:
- **CRITICAL/HIGH**: Accept All | Review Each | Defer All
- **MEDIUM**: Neutral presentation (no recommendation bias)
- **LOW**: Accept All (Recommended) | Review Each | Defer All

If "Review Each": present individually with **ACCEPT** / **MODIFY** / **REJECT** / **DEFER**.

### Phase 3: Decision Summary
```
ACCEPTED: [N]  MODIFIED: [N]  REJECTED: [N]  DEFERRED: [N]
Preview: Line [N]: "old" → "new" ...
→ Apply fixes and continue to Run [R+1]  |  → Go back and revise
```
Confirm via `AskUserQuestion`.

### Phase 4: Apply
1. Sort fixes by line number descending (prevent shift issues)
2. Apply via Edit tool
3. Save `readme_snapshot_pre.md` and `readme_snapshot_post.md`
4. Pass deferred findings to next run

### Disposition Model
| Disposition | Applied? | Carries Forward? |
|-------------|----------|------------------|
| ACCEPT | Yes, immediately | Verified next run |
| MODIFY | Yes, with user's text | Verified next run |
| REJECT | No | Dropped permanently |
| DEFER | No | Flagged for next run |

### Convergence & Skip
- After Run 3+ (requires at least 3 runs to evaluate two consecutive deltas): Health Score delta (|run_N - run_{N-1}|) < 3 for two consecutive runs + no CRITICAL/HIGH → offer early stop
- `runs=1`: Report only, no review UI (default mode — quick check); convergence does not apply
- 0 findings: Auto-proceed
- Final run (R == N): Report only

---

## Execution Steps

### Steps 1-2: Parse & Read
Extract repo path + run count from `$ARGUMENTS`. Find and read the README. Verify it's a git repo (or at minimum a project directory with identifiable structure). If NOT a git repo: skip git-dependent checks (git blame, git log, git tags, Staleness Index), mark those claims UNVERIFIABLE, and note "Non-git project: staleness and history checks unavailable" in the report.

### Step 3: Execute Runs

For each run R (1 to N):

**3a. EXECUTE**: Launch waves per Architecture. Each wave group: Wave 0 → Agent A0 (structural scan) → 3 sequential waves of 5 parallel agents. Later waves receive prior findings. Run 2+ agents also receive deferred findings.

**3b. SYNTHESIZE**: Calculate health score, delta, sort by severity, count per tier. Include staleness and missing findings.

**3c. REVIEW**: Skip if final run, runs=1, or 0 findings. Otherwise follow Inter-Run Review Protocol.

**3d. APPLY**: Sort fixes by line descending, apply via Edit, save snapshots.

**3e. CARRY FORWARD**: Re-read updated README, compile deferred findings, log per-run entry.

**3f. CONVERGENCE**: After Run 3+, delta < 3 two consecutive + no CRITICAL/HIGH → offer early stop.

### Step 4: Consolidate

After all runs:
1. Collect all ratings across agents/runs
2. Supermajority consensus (75%+), else debate judge verdict
3. Map: FALSE→CRITICAL, MOSTLY FALSE→HIGH, MIXED→MEDIUM, MOSTLY TRUE→LOW, TRUE→VERIFIED
4. Health Score = (TRUE + MOSTLY TRUE) / checkable claims x 100 (exclude UNVERIFIABLE)
5. Staleness Index = days_since_readme_edit / max(days_since_code_edit, 1) (1.0 = in sync, >2.0 = drifting; denominator floored at 1 to prevent div-by-zero when code was edited today)
6. Completeness Score = standard_sections_present / expected_sections x 100 (expected sections: Description/Overview, Installation, Usage/Quick Start, API/Configuration, Contributing, License — 6 total; adjust for project type)
7. Track finding lifecycle (discovered Run X, fixed Run Y, verified Run Z)

### Step 5: Final Report

```
## README Audit Report
**Repo**: [path] | **README**: [filename] | **Agents**: [N] | **Runs**: [N]
**Health Score**: [N]/100 | **Staleness Index**: [N] | **Completeness**: [N]%
**Overall**: CURRENT / DRIFTING / STALE / UNRELIABLE

### Assessment Criteria
- CURRENT (80-100): README accurately reflects project state
- DRIFTING (60-79): Some claims outdated but core info correct
- STALE (40-59): Significant drift, users will hit problems
- UNRELIABLE (0-39): README substantially diverges from project state

### Session Progression (multi-run only)
| Run | Score | Delta | CRIT | HIGH | MED | LOW | Fixes |
|-----|-------|-------|------|------|-----|-----|-------|
Dispositions: Accepted [N] | Modified [N] | Rejected [N] | Deferred [N]
Converged: [Yes/No]

### Claim Summary
Total: [N] | By category: VERSION N, INSTALL N, USAGE N, FEATURE N, DEPENDENCY N, ARCHITECTURE N, BADGE N, CONFIG N, LINK N, TEMPORAL N, LICENSE N, CONTRIBUTOR N, MISSING N, STALE N

### Veracity Distribution
TRUE [N] ([%]) | MOSTLY TRUE [N] ([%]) | MIXED [N] ([%]) | MOSTLY FALSE [N] ([%]) | FALSE [N] ([%]) | UNVERIFIABLE [N] ([%])

### CRITICAL — must fix
F### [CAT] Claim: "..." | Actual: ... | Source: path (Tier N) | Confidence: N% | Fix: old → new

### HIGH — should fix
### MEDIUM — consider fixing
### LOW — minor issues

### STALE — time-sensitive drift
S### Section "[heading]" last edited [N] days ago, code changed [N] times since

### MISSING — should add
M### [importance] Description of what's missing and why it matters

### VERIFIED ([N] claims)
### UNVERIFIABLE ([N] claims)

### Top 5 Fixes (by impact)
1. ...
2. ...

### Evidence Quality
T1 (repo files): [N] | T2 (derived): [N] | T3 (external): [N]
```

### Step 6: Log Results

**6a. Metadata**: `git remote get-url origin` (fallback: dirname), `pwd`, `git branch --show-current`, `git rev-parse --short HEAD`, generate session UUID.

**6b. Per-run entry** (after each run's review):
```json
{
  "id": "<timestamp>-readme-run<N>", "type": "readme-audit-run", "session_id": "<UUID>",
  "run_number": 1, "timestamp": "<ISO-8601>",
  "target": "README.md", "project": "...", "project_path": "...",
  "branch": "...", "commit_sha": "...",
  "health_score": 0, "prior_score": null, "score_delta": null,
  "staleness_index": 0.0, "completeness_score": 0,
  "agents_deployed": 17,
  "claims_total": 0, "claims_verified": 0, "claims_flagged": 0,
  "claims_unverifiable": 0, "consensus_rate": 0,
  "overall_assessment": "CURRENT|DRIFTING|STALE|UNRELIABLE",
  "veracity_distribution": { "true": 0, "mostly_true": 0, "mixed": 0, "mostly_false": 0, "false": 0, "unverifiable": 0 },
  "severity_counts": { "critical": 0, "high": 0, "medium": 0, "low": 0, "verified": 0, "unverifiable": 0, "stale": 0, "missing": 0 },
  "evidence_quality": { "tier1_sources": 0, "tier2_sources": 0, "tier3_sources": 0, "multi_source_claims": 0 },
  "review_decisions": { "critical_batch": null, "high_batch": null, "medium_batch": null, "low_batch": null },
  "claim_categories": { "version": 0, "install": 0, "usage": 0, "feature": 0, "dependency": 0, "architecture": 0, "badge": 0, "config": 0, "link": 0, "temporal": 0, "license": 0, "contributor": 0, "missing": 0, "stale": 0 },
  "findings": [{
    "fact_id": "F###", "severity": "...", "veracity_rating": "...", "confidence": 0,
    "category": "...", "claim": "...", "actual": "...", "evidence_summary": "...",
    "source_tier": "T#", "source_path": "...", "agents_agreed": "N/M",
    "location": "line N", "fix_old": "...", "fix_new": "...", "status": "...", "disposition": null
  }]
}
```

**6c. Session entry** (after all runs): Same base fields plus `type: "readme-audit-session"`, `runs_requested`, `runs_completed`, `score_progression[]`, `converged`, `disposition_summary{accepted,modified,rejected,deferred}`, `runs[]` (array of run IDs).

For `runs=1`: single entry with `type: "readme-audit-run"`, `session_id: null`, no session entry.

**6d.** If `~/.claude/readme-audit-log.json` does not exist, create it with an empty JSON array `[]`. Then append new entries to the array. Append-only — never overwrite existing entries.

**6e. Paper trail:**
```
~/.claude/audit-trails/readme/<YYYY-MM-DD>_<HH-MM-SS>_<project>/
├── session_metadata.json, consolidated_report.md
└── run_N/  (one per run)
    ├── metadata.json, readme_snapshot_{pre,post}.md
    ├── manifest_summary.json
    ├── wave0_decomposition.md
    ├── agent_A0_structural_scan.md
    ├── wave{A-I}_agent{1-5}.md  (raw agent outputs, matching the run's wave set)
    ├── staleness_findings.json, missing_findings.json
    ├── disputed_claims.json
    ├── review_decisions.json, fixes_applied.json
    ├── consolidated_report.md
    └── deferred_findings.json  (Run 2+ only)
```
After each run: write run_N/ files. After all runs: write session files.
Commit: `cd ~/.claude/audit-trails && git add -A && git commit -m "readme-audit: <project> <timestamp> health=<N>/100 (<M> runs)"`

### Step 7: Dashboard Update

1. Read `~/.claude/readme-audit-log.json` and `~/.claude/audit-dashboard/index.html`
2. If `<script id="readme-audit-data">` exists, replace its contents with: `const README_AUDIT_DATA = <JSON>;`
3. If the tag does NOT exist: add `<script id="readme-audit-data">\nconst README_AUDIT_DATA = <JSON>;\n</script>` before `</body>`. Note: the dashboard will also need a "README Audit" tab button and rendering code added to display this data — on first run, inform the user that dashboard integration requires a one-time tab setup.
4. Write updated `index.html`
5. Tell user: **"Dashboard updated. Open `~/.claude/audit-dashboard/index.html`."** (If first run, add: "Note: A README Audit tab and rendering code need to be added to the dashboard to visualize this data.")

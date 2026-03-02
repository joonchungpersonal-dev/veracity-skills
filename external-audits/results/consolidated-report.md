# Veracity Audit of 247 Claude Code Skills Across 4 Major Repos

**Audit date**: 2026-03-02
**Auditor**: Joon Chung (joonchungpersonal-dev)
**Method**: Automated decomposition → web-verified fact-checking using `/veracity-tweaked-555`
**Empirical validation**: Key findings confirmed via NCBI E-utils, PyPI, NVD, HuggingFace, and OpenAlex APIs

---

## Summary

| Repo | Stars | Skills Audited | Claims Checked | Errors | Error Rate |
|------|-------|---------------|----------------|--------|------------|
| anthropics/skills | 81.1k | 17/17 | 54 | 3 | 5.6% |
| trailofbits/skills | 3k | 60/60 | 165 | 11 | 6.7% |
| obra/superpowers | 67.8k | 14/14 | 28 | 2 | 7.1% |
| K-Dense-AI/claude-scientific-skills | 10.8k | 156/156 | 285 | 33 | 11.6% |
| **Total** | — | **247** | **532** | **49** | **9.2%** |

---

## Impact Tiers

Not all errors are equal. We classified each by real-world consequence:

| Tier | Label | Count | % of Errors | Description |
|------|-------|-------|-------------|-------------|
| 0 | Cosmetic | 21 | 43% | No functional effect. Stale counts, typos, understated numbers. |
| 1 | Friction | 11 | 22% | User hits a bump but self-corrects quickly. Dead URLs, wrong install flags. |
| 2 | Misleading | 11 | 22% | User makes a wrong decision. Skips a tool, builds on deprecated API, cites wrong paper. |
| 3 | Silent Failure | 2 | 4% | Code breaks in production or creates legal/security risk without warning. |

**Key finding**: 65% of errors (Tier 0 + 1) have no meaningful impact on outcomes. The 26% that are Tier 2-3 cluster around two patterns: stale database statistics and wrong package/tool references.

---

## Tier 3 Findings (Silent Failure) — The Ones That Matter Most

### 1. `pip install caracal` installs the wrong package entirely
- **Repo**: trailofbits/skills (cairo-vulnerability-scanner)
- **Claim**: "Installation: `pip install caracal`"
- **Reality**: PyPI's `caracal` package is **CARACal — Containerized Automated Radio Astronomy Calibration**. The Cairo security scanner (crytic/caracal) is a Rust tool installed via `cargo install`.
- **Impact**: An auditor following this instruction installs a radio astronomy pipeline instead of a security scanner. They'd waste time debugging why "caracal" doesn't scan Cairo contracts.
- **Empirically confirmed**: PyPI API returns `Summary: "Containerized Automated Radio Astronomy Calibration"`, `Author: "The CaraCAL Team"`

### 2. ESM3 labeled as MIT license (actually non-commercial only)
- **Repo**: K-Dense-AI/claude-scientific-skills (esm)
- **Claim**: `license: MIT license`
- **Reality**: ESM3-open (esm3-sm-open-v1) uses EvolutionaryScale's **Community License Agreement**, restricted to non-commercial use. The model is gated on HuggingFace.
- **Impact**: A company using ESM3 in a commercial protein engineering pipeline, trusting the MIT license claim, could face legal consequences.
- **Empirically confirmed**: HuggingFace API returns `gated: "auto"`, README requires authentication

---

## Tier 2 Findings (Misleading) — Could Cause Wrong Decisions

### Hallucinated PMID
- **Repo**: K-Dense-AI (gwas-database)
- **Claim**: GWAS Catalog citation with PMID: 37953337
- **Reality**: PMID 37953337 is actually "Ensembl 2024" by Harrison PW et al. The correct PMID for Sollis et al. is 36350656.
- **Empirically confirmed**: NCBI E-utils returns the wrong paper for the cited PMID

### Database species/compound counts off by orders of magnitude
- Ensembl: "250 species" → actually 4,800+ eukaryotic genomes (confirmed: Ensembl REST API returns 348 vertebrate species alone)
- ZINC: Claims ZINC22 has "230M compounds" → that's ZINC20. ZINC22 has **5.9 billion** 3D compounds (25x undercount)

### Outdated tool capabilities presented as current limitations
- AlphaFold: "single chains only" → AlphaFold 3 (May 2024) predicts multi-chain complexes
- Atheris: "Python 3.7+" → current versions require Python 3.11-3.13
- slither-check-erc: "6 ERC specs" → now supports 11

### Deprecated API references
- protocols.io: API v3 → now on v4
- OpenAlex: "no auth required" → now has usage-based pricing (confirmed: response headers show `x-ratelimit-limit-usd`)

### Wrong file paths
- obra/superpowers: Codex skills path `~/.agents/skills/` → correct is `~/.codex/skills/`

### Wrong citation author
- Open Targets Platform: First author listed as Ochoa, D. → actually Buniello, A.

### Misleading build configuration
- FUZZING_BUILD_MODE_UNSAFE_FOR_PRODUCTION described as "automatically defined" by libFuzzer → it's a convention requiring manual `-D` flag

---

## Tier 1 Findings (Friction) — Bumps, Not Blockers

- `npm install -g docx/pptxgenjs` — works but wrong scope (should be local install)
- Stale documentation URLs (Anthropic docs moved)
- ASan "experimental" on Windows — actually GA since VS 2019 16.9
- AddressSanitizerFlags URL typo (missing 't' in Sanitizer — redirects anyway)
- GEO database stats inflated beyond official NCBI figures
- COSMIC version date mismatch (v102 is Nov 2024, not May 2025)
- TON vulnerability scanner internal inconsistency between frontmatter and body

---

## Tier 0 Findings (Cosmetic) — Technically Wrong, Practically Harmless

21 findings of stale counts, minor typos, and understated numbers:
- PubMed "35M+" → 39M+, PubChem "110M+" → 119M+, ChEMBL "13K targets" → 17K, BRENDA "45K enzymes" → 77K+
- Scanpy license typo: "SD-3-Clause" → "BSD-3-Clause"
- scikit-learn install: "uv uv pip install" (doubled prefix)
- STRING "5000+ organisms" → 12,535
- CVE-2023-4863 labeled "libpng" instead of "libwebp" (guidance still correct)
- CMPLOG/RedQueen called "best" constraint solver (opinion, not fact)
- Various other stale database counts (Reactome, USPTO, Metabolomics Workbench, HMDB dates)

---

## Error Type Distribution

| Category | Count | % |
|----------|-------|---|
| Stale database statistics | 18 | 37% |
| Wrong numbers (inflated/understated) | 8 | 16% |
| Incorrect methodology/tool behavior | 7 | 14% |
| Wrong library/package info | 6 | 12% |
| Citation errors (wrong author, hallucinated PMID) | 4 | 8% |
| Wrong API/URL | 4 | 8% |
| Dead/wrong URL | 2 | 4% |

**Dominant pattern**: 37% of all errors are stale database statistics — not fabrication, not carelessness, just entropy. Databases update; skill files don't.

---

## What We Did NOT Find

Equally important:
- **Zero fabricated papers** across all 4 repos. Every citation checked (Jumper 2021, Cialdini 2021, Kocher 1996, Fagan 1976, etc.) was real.
- **Zero invented libraries or tools**. Every package, API, and tool referenced actually exists.
- **Zero malicious content**. No prompt injection attempts, no hidden instructions.
- **Security guidance is overwhelmingly accurate** — CWE numbers, vulnerability patterns, cryptographic standards all verified.

---

## Repo-Level Observations

### anthropics/skills (5.6% error rate)
Cleanest repo. Errors are systematic (repeated `npm install -g` pattern) rather than random. 7 of 17 skills are purely procedural with zero verifiable claims.

### trailofbits/skills (6.7% error rate)
Deep domain expertise shows. All CVE numbers verified correct. The `pip install caracal` error is the most consequential finding in the entire audit — but it's one wrong install command in 60 skills of otherwise impeccable security documentation.

### obra/superpowers (7.1% error rate)
Only 2 errors in 14 skills. Both academic citations (Cialdini, Meincke et al.) are accurately cited with correct stats, titles, and affiliations. Exceptionally well-sourced for community work.

### K-Dense-AI/claude-scientific-skills (11.6% error rate)
Highest error rate, but context matters: these 156 skills reference hundreds of specific database sizes, API endpoints, molecule counts, and version numbers — the most fact-dense content of any repo. Error rate is driven by staleness, not fabrication.

---

## Limitations of This Audit

1. **Single-pass verification**: Each claim was checked by one auditor agent via web search. No multi-agent consensus or adversarial validation was applied to the external audit (unlike our self-audits which used 16 agents per run).

2. **Web search recency bias**: Claims about rapidly-changing databases (compound counts, species counts) were checked against current values. Some "errors" may have been correct when written.

3. **Sampling for large repos**: K-Dense and Trail of Bits were audited in batches. While we achieved full coverage (all 247 skills), batch-level consistency wasn't cross-validated.

4. **Procedural claims not tested**: Many skills are workflow instructions ("run X, then Y"). We only verified factual assertions, not whether the workflows produce correct results.

5. **LLM-as-judge limitation**: The auditor agents are themselves LLMs, subject to the same hallucination risks they're checking for. We mitigated this by empirically confirming the highest-impact findings via direct API calls.

6. **No runtime testing**: We verified claims about APIs and packages via registry lookups and documentation, but did not execute the actual code examples in each skill (except for the `pip install caracal` and API endpoint tests).

7. **Meta-circularity**: This audit tool was itself built by an LLM and audited for veracity errors before use (score: 62 → 96.5 over 6 convergence runs). The tool checking itself is an inherent limitation.

# I built a veracity-checking skill for Claude Code. Running it on itself was humbling — here's what I learned.

I'm a researcher who uses Claude Code daily (sleep science background, University of Miami). As a scientist, factual accuracy isn't optional — a wrong citation or a fabricated statistic in a skill file means every downstream user inherits that error.

So I built `/veracity-tweaked-555`, a Claude Code skill that decomposes documents into atomic claims and verifies each one via web search — 16 parallel agents across 4 waves per run. Built in collaboration with Claude Code (Opus 4.6): Claude drafted the code, I designed the methodology and verified accuracy by running the tool on everything, including this post. What I found made me rethink how much I trust any skill file that hasn't been audited.

## The self-audit: 62/100

The first thing I did was run it on its own SKILL.md.

It scored **62 out of 100**.

The skill I built to catch hallucinations had hallucinated facts in its own documentation. It had fabricated a performance statistic ("3x more accurate" for SAFE, which the paper never claims), inflated a paper's improvement claim ("+35.5%" was actually +5.5% over SOTA), and fabricated an acronym expansion for a real technique. These were claims that read well, sounded right, and were wrong.

After initial fixes it reached 80, then 84 after a third run. A week later, I ran a more rigorous convergence loop — 6 runs, 19 agents, 35 additional fixes — and it stabilized at 96.5/100. But the path wasn't a smooth climb. The fresh v3 audit actually *dropped* to 74 — because the v1 fixes had introduced new errors (an understated token cost and an incomplete tool list). Fixing things created new things to fix.

```
Veracity Score Over Time (9 audit runs)
100 ┤                                                   ●──── ● 96.5
 95 ┤                                             ●  95.7
 90 ┤                                       ● 91
 85 ┤                  ● 84           ● 85        ▲ v3 convergence
 80 ┤        ● 80        ╲                         (6 runs, 35 fixes)
 75 ┤                      ╲ ● 74
 70 ┤                       ╲╱  ← regression!
 65 ┤  ● 62                     (v1 fixes introduced
 60 ┤  ▲ v1 self-audit           new errors)
     └───────────────────────────────────────────────────────
      Feb 22                 Mar 1
```

The errors aren't random. They follow patterns: attribution inflation (slightly stronger language than the source warrants), plausible-but-fabricated identifiers (PMIDs, arXiv IDs that look real but point to different papers), and stale statistics presented as current.

## The context engineering problem

A single audit run generates ~917K tokens across 16 agents. Claude Code's 200K context window can't hold it. But the real problem isn't size — it's what gets lost when the window compresses.

When Claude Code compacts your conversation to stay within limits, it performs lossy compression. In my experience, after a few compactions the agent loses track of how findings relate to each other — which fix caused which regression, which claim contradicts which other claim. Individual facts (names, numbers, function signatures) survive better than the connections between them.

Claude's diagnosis was that **relational information** — causal chains, cross-references, multi-step dependencies — is harder to preserve in a summary than isolated facts. I haven't verified this rigorously, but it resonated with my sociology training: meaning arises from the connections between things, not from the things themselves. It would explain what I observed: an agent that can recall individual findings but can't reason about how they connect.

I solved this by building a companion skill called `/context-engineer` that predicts overflow before it happens and externalizes relational state to JSON files on disk. The design test: if you can `/clear` your entire conversation and resume from the state file alone, the architecture is correct. If you can't, you're still depending on context that will eventually be compressed away.

## Running it on my own skills

Encouraged by the self-audit, I ran veracity checks on all 6 of my Claude Code skills before releasing them publicly. The results:

- The `/grill` skill — a composite code review tool built on GPTLens (Hu et al.), Trail of Bits' [claude-code-config](https://github.com/trailofbits/claude-code-config), obra's [superpowers](https://github.com/obra/superpowers), and IIA/OWASP standards; Claude Code consolidated the sources, I designed the 2-agent architecture — had a **fabricated paper title** in its attribution section. The citation crediting the GPTLens source paper looked perfect (authors, venue) but the title was fabricated and the year was wrong. The actual paper has a completely different name. The irony: the section designed to give proper credit contained a hallucinated citation.
- The same skill misattributed the 5C audit framework to COSO (a different standards body) instead of IIA Standards 2410/2420. This error appeared in multiple locations across the file.
- The `/context-engineer` skill had internal inconsistencies — the prose said "5-10K tokens" while a table in the same file said "5-15K tokens" for the same metric.

12 total fixes across 6 skills. All passed at 95+ on 3 consecutive runs after corrections.

The takeaway: **I am not immune to this, and I don't think anyone is.** If you're writing Claude Code skills, you almost certainly have claims in your SKILL.md files that you'd want to correct. Not because you were careless, but because LLMs generate plausible-sounding facts that pass human review.

## Trying it on community skills

After auditing my own work, I wanted to see if the tool could be useful to the broader community. I ran veracity checks across 247 skills from 4 popular repos — not to find fault, but to understand the landscape and see if the patterns I found in my own work showed up elsewhere.

Here's what I found across 532 verified claims:

### The good news

- **Zero fabricated papers** across all 4 repos. Every citation I checked — Jumper 2021, Cialdini 2021, Kocher 1996, Fagan 1976 — was real, with correct authors and venues.
- **Zero invented libraries or tools.** Every package, API, and tool referenced actually exists.
- **Zero malicious content.** No prompt injection, no hidden instructions.
- **Security guidance is overwhelmingly accurate.** All CWE numbers, vulnerability patterns, and cryptographic standards verified across Trail of Bits' 60 skills.
- **Academic citations are well-sourced.** obra/superpowers cites Cialdini (2021) and Meincke et al. (2025) with correct stats, affiliations, and publishers.
- **Overall accuracy: 91%** of verifiable claims checked out.

| Repo | Skills | Claims | Accuracy |
|------|--------|--------|----------|
| anthropics/skills | 17 | 54 | 94.4% |
| trailofbits/skills | 60 | 165 | 93.3% |
| obra/superpowers | 14 | 28 | 92.9% |
| K-Dense-AI/claude-scientific-skills | 156 | 285 | 88.4% |
| **Total** | **247** | **532** | **90.8%** |

Note: 100% is structurally unlikely. Some claims are unverifiable (paywalled sources, internal data), and others are honest approximations — a single measurement like "917K tokens" or a rounded count can't be confirmed as exactly TRUE, only as not wrong. The ceiling for this kind of audit is probably ~95%.

### The main pattern: staleness

37% of all errors were **stale database statistics** — not fabrication, just entropy. Databases update; skill files don't. Ensembl "250 species" is now 348 vertebrate species (4,800+ eukaryotic genomes), ZINC "230M compounds" is now 5.9 billion, and so on. A `last_verified` field in skill frontmatter could catch this before users hit it.

## Where the real risk lives: newly created skills

One important caveat: the repos I audited are **well-established** — tens of thousands of stars, active maintainers, professional security expertise (Trail of Bits), domain scientists (K-Dense-AI). The skills that worry me more are the ones freshly generated by an LLM in a single session, dropped into `~/.claude/skills/`, and never audited. That's exactly what happened with my own skills before I started self-auditing.

I think running a veracity check after creating a new skill should be standard practice, the same way you'd run tests after writing code. This is especially true if you're working solo — a lot of us are building with Claude Code in domains where there's no second pair of eyes on the SKILL.md. Automated veracity checking is not a replacement for peer review, but it's a meaningful substitute when peer review isn't available.

## Limitations (being honest about what this can and can't do)

1. **Single-pass verification** — the community audit used one agent per claim via web search. My self-audits used 16 agents with multi-run convergence. The community results are therefore less thorough.
2. **Web search recency bias** — some "errors" may have been correct when the skill was written. I tried to note this where applicable.
3. **Procedural claims untested** — I verified factual assertions (numbers, citations, API endpoints), not whether the workflows produce correct results.
4. **LLM-as-judge** — the auditor agents are themselves LLMs, subject to the same hallucination risks. I mitigated the highest-impact findings by confirming them empirically via NCBI, PyPI, NVD, and HuggingFace APIs.
5. **Meta-circularity** — a tool built by an LLM, auditing content written by LLMs, checked by LLM agents. I can't fully escape this. The self-audit (62 → 96.5) and empirical API confirmations are my best defense.
6. **No runtime testing** — I checked registries and APIs for the top findings but didn't execute every code example in every skill.

## What I'm sharing and why

Both skills (`/veracity-tweaked-555` and `/context-engineer`) are open source at [joonchungpersonal-dev/claude-skills](https://github.com/joonchungpersonal-dev/claude-skills), along with the full audit results (JSON files + consolidated report) from the community skills audit.

The most valuable thing I learned is that **self-auditing catches errors that survive human review** — not because you're careless, but because LLM-generated errors are plausible by nature. They read well, sound right, and are sometimes wrong.

I'm a scientist, not a software engineer — I'm sure people in this community could make the verification faster, the token usage lower, and the convergence tighter. I welcome improvements, forks, and PRs. The methodology matters more to me than the implementation.

---

**Repo**: [joonchungpersonal-dev/claude-skills](https://github.com/joonchungpersonal-dev/claude-skills)
**Full audit report**: `external-audits/results/consolidated-report.md`
**Raw data**: `external-audits/results/*.json` (all 532 claims with verdicts and evidence)

---

## Appendix: Veracity audit of this post

Claude Code (Opus 4.6) drafted the first versions of this post. I then ran the veracity process on it — 3 automated runs across 128 atomic facts, plus manual review. The process caught errors in its own output. Selected corrections:

**Pre-posting fixes (manual + automated):**
- Conflated two audit sessions into one timeline — actually two phases (Feb 22: 62→80→84; Mar 1: 74→96.5)
- Score graph hid the 80→74 regression between v1 and v3 — corrected to show full trajectory
- `/grill` attribution credited as "my skill" — corrected to credit Claude Code for the consolidation work
- Fidelity degradation chart (90%→75%→55%→30%) presented precise numbers that were never measured — removed
- Ensembl species/genome counts conflated — corrected via REST API
- GPTLens citation year wrong (2024 vs 2023) — corrected
- Skill name wrong throughout (`/veracity-555` → `/veracity-tweaked-555`)

**3-run automated audit (87.2% → 90.7% → 90.2%):**
- 128 facts checked, 0 FALSE, 0 MOSTLY FALSE, 0 MIXED
- All 15 numbers in the community audit table independently verified against raw JSON data
- Score capped at ~91% by 11 MOSTLY TRUE facts — honest approximations (single measurements like "917K tokens," heuristics, soft counts) that can't be upgraded with available evidence
- Full audit trail and raw data in the repo

**Remaining caveats:**
- "91% accuracy" is the "not found to be wrong" rate: (532-49)/532 = 90.8%. Excluding unverifiable claims: 432/481 = 89.8%. I used the more generous framing.
- "All CWE numbers verified" — we verified what our agents checked, not literally every reference. Softened to "overwhelmingly accurate."

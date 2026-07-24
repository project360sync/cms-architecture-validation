# Run manifest — CMS PRD (new-project-docs skill)
- Date: 2026-07-24
- Document: docs/prd.md in project360sync/cms-architecture-validation
- Kit templates source commit: 9257a9b29c9192ccecf42054577145d4cace226f (the fetched `main` HEAD; all 5 blob SHAs verified equal to the rubrics.md Compatibility baseline manifest via gh api contents listing — CORRECTION per audit-round3 Finding 2: this is NOT the reviewed anchor commit, which is `8e6b3f9e83de15d966d0423199918b5d3304ff24`; proceeding was lawful because the blobs matched the baseline)
  - new-project/README.md d26fe5c3a3b31b9a732d715424af311c5fda9048
  - new-project/prd-template.md b03b3d0acd24dff1e5aa38e5e4119b0d8a58de45
  - new-project/tech-spec-template.md 5a12558c8a828463bd8ee94a70e24caf9d3e3f77
  - new-project/adr-template.md 5f605d29d5671f3467892a0ac216c3631bb676c6
  - new-project/repo-bootstrap-checklist.md 00d2811552bc80d531da18fd75862f0f6c62ba49
- Fetch mode: network (gh api) blob verification + local committed clone read at same commit
- Evidence inventories: evidence-arch.md, evidence-gtm.md, evidence-cms.md (this dir)
- Decision register: decision-register.md (this dir)
- Rounds: (ledger appended per round)

## Final ledger — Round 1 (2026-07-24)
| Metric | Effective score | Scorer evidence |
|---|---|---|
| Evidence-grounding | 9/10 | 15 claims sampled + dereferenced; no fabrication; labeled gaps honored |
| Completeness | 9/10 | all sections + 8 NFR categories, template columns exact |
| Internal consistency | 9/10 | D1/D2 thresholds identical across §7/§8/§9/§10/§11 |
| Actionability | 9/10 | Phase 0 startable from §8/§9 alone |
| Falsifiability | 9/10 | all 3 gates numeric/observable; assumptions can fire |
| Honesty | 9/10 | self-undermining redirect rows preserved; reflow limit + consequence stated |
| Metric quality | 9/10 | behavioral+instrumented primary, real counter-gaming guardrail |

Conformance (round 1): reported PASS with notes — **overturned by external review** (relative resolve-by dates violate the skill's absolute-dates rule → round-1 conformance retroactively FAIL).
Note: conformance reviewer instance #1 glitched (empty output, 0 tool calls) and was re-run fresh; scores were never shown to any scorer (round 1 = trivially blind).

## CORRECTION after external (Codex) review — 2026-07-24
The round-1 "DONE" exit was a **category error by the orchestrator** and is
retracted: per the skill, open **load-bearing** assumptions (A1 demand, A3
substitution, A5 reflow — all untested until Gate A/B run) cap the run at
**`verification pending`**. D1–D5 confirm *user decisions*, not evidence.

**Run status: `verification pending`** — scores may all be ≥9 and conformance
PASS, but the run cannot be "done" until Gate A and Gate B close with real
evidence. Full round-1 scorer/conformance outputs are committed under
[`scores-round1/`](scores-round1/) for auditability; a round-2 re-score of the
revised sections follows the 8 accepted external findings (see ledger below
when appended).

## Round 2 ledger — 2026-07-24 (after the 8 external findings were applied)
Revised sections: §7 guardrail baseline; §8 dates + Phase-1 entry; §9 Gate B threshold rescope; §10 A1 negation + absolute dates; §11 D5 row; evidence-cms pinned + test log. Scorers: fresh instances, blind (scores-round1/ + manifest ledger explicitly off-limits).

| Metric | R1 | R2 (effective) | Change tied to |
|---|---|---|---|
| Evidence-grounding | 9 | 9 | pinned CMS evidence + [D5] labels verified |
| Completeness | 9 | 9 | absolute dates; all schemas re-verified |
| Internal consistency | 9 | 9 | De Morgan A1↔Gate A checked; Gate B↔Phase 2 exit checked |
| Actionability | 9 | 9 | dates + owners + executable gates re-verified |
| Falsifiability | 9 | 9 | A1 exact negation confirmed; Gate B scope-tight |
| Honesty | 9 | 9 | self-undermining rows + reflow consequence re-verified |
| Metric quality | 9 | 9 | guardrail baseline window + tolerance now defined |

Conformance round 2: **FAIL** on one isolated defect (D5 confirm-by event-relative) → fixed (absolute date 2026-07-31) → **round 2b full independent re-check: PASS** (all dates absolute, all schemas exact).

**RUN STATUS: `verification pending`** — all metrics ≥9 and conformance PASS, but load-bearing assumptions A1/A3/A5 remain open until Gate A (close: 2026-11-30) and Gate B deliver real evidence. This is the correct terminal state for a pre-validation PRD under the skill's invariant.

Remaining "missing for 10" follow-ups (non-blocking): §7 >10% band labeled as operational threshold not [D3]; §11 restate pinned CMS commit in-doc; A4 hard hour-cutoff (>70h on first two reuse-eligible builds); numeric post-gate edit-frequency target; §7 Gate A framing tightened to the full conjunction.

## Round 3 — 2026-07-24 (after external round-2 findings; scorer re-run on the FINAL doc)
Trigger: external review — the D5 conformance fix changed the doc after the round-2 scorers saw it, so a fresh blind score round is required (Phase 5 rule); plus mechanical fixes (full-path permalinks in evidence-cms, §11 demand-first clarification, EOF whitespace).

**Normalization procedure:** section text extracted by `## <n>.` headers; whitespace runs collapsed to single spaces; sha256 (first 16 hex chars). Metric → section-set mapping and input hashes at round-3 start:

| Metric | Section set | Normalized hash (sha256/16) |
|---|---|---|
| Evidence-grounding | whole doc | `db7e3997d700f8c4` |
| Completeness | whole doc | `db7e3997d700f8c4` |
| Internal-consistency | §0,§5,§7,§8,§9,§10,§11 | `a4189789fe11bea9` |
| Actionability | §8,§9,§10 | `c3db6ba7831bf09a` |
| Falsifiability | §9,§10 | `19508dec0d8a4f73` |
| Honesty | §3,§4,§9 | `47e8163db408d400` |
| Metric-quality | §7 | `cfecdfd554d1fe11` |

(R1/R2 input hashes were not captured at the time — a process defect the skill now
fixes [prompt-collection PR #6]; R3 onward records them. R2→R3 changed sections:
§10/§11 (D5 confirm-by date), §11 (tech-spec pointer demand-first clarification),
evidence-cms.md (permalinks). Scorer artifacts: scores-round3/, persisted verbatim.)

### Round 3 results
| Metric | Effective | Schema |
|---|---|---|
| EG / C / IC / A | 9 / 9 / 9 / 9 | per-metric quote+pointer, ≥3 findings each |
| F / H / MQ | 9 / 9 / 9 | per-metric quote+pointer, ≥3 findings each |

Conformance: PASS (0 defects, 2 NOTEs, no SPEC-AMBIGUITY). Artifacts: scores-round3/ (persisted verbatim).
Follow-ups carried (non-blocking, missing-for-10): §4 tier-pairing synthesis labeled; interim perf/a11y targets; §7 Gate-A-label full conjunction; capacity-model resolve-by row; A4 hard cutoff; guardrail single breach condition ("tartós" duration).
Run status: `verification pending` (unchanged — load-bearing A1/A3/A5 open until Gate A/B). Phase-6 audit: pending, report to follow as audit-round3.md.

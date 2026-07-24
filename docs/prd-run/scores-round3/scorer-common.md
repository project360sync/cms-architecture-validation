# Round 3 — Blind scorer A (common metrics) — verbatim output
Agent a5152a552dd36ae00, 2026-07-24. Fresh instance; prior-round material excluded. Mandatory per-metric quote+pointer schema enforced.

## Evidence-grounding — SCORE: 9/10
- F1 (grounded): prd.md:152 "~216k HUF/év visszatérő ügyfelenként; 5. ügyfélnél portfólió ~1M HUF/év [GTM §3.5]" → verified evidence-gtm.md:15 exact match.
- F2 (grounded, trap avoided): §9 "publikált CSP `default-src 'self'` [CMS serving.ts:3]" (proven predecessor) kept separate from §6's greenfield target "default-src 'none'" [ARCH §16.3] → both verified at their sources.
- F3 (grounded): F4 "a vetett skeleton-markup klónozódik friss id-kkel [CMS apply-ops.ts:147]" → verified evidence-cms.md:24.
- Provisional check: D5 (prd.md:243) full option/rationale/owner/confirm-by → does not count against grounding per rubric.
missing for 10: §4 packaging table PAIRS build↔operation tiers (Landing↔Alap etc.) citing [GTM §3.1][GTM §3.2], but the GTM inventory presents them as independent axes — the 1:1 pairing is unstated synthesis.

## Completeness — SCORE: 9/10
- F1: Accessibility NFR (prd.md:136) "Research gap… WCAG 2.1 AA… Owner: László" vs template's "standard to meet" → justified gap+plan+owner = complete per rubric; all 8 categories present.
- F2: Gate C affirmatively applied ("Alkalmazandó", threshold at prd.md:187) vs template "where applicable".
- F3: §10 three separate structures (assumptions/risks/open-questions) exactly per template, no cross-contamination.
missing for 10: Performance + Accessibility carry no interim provisional target ahead of the Gate B spike.

## Internal consistency — SCORE: 9/10
- F1: A1 = exact De Morgan negation of Gate A (quotes at prd.md:199 vs :176) ✓
- F2: §11 tech-spec pointer ↔ §8 Phase-1 entry now agree (spike DOC may be early; spike RUN only after Gate A PASS) — quotes at prd.md:245 and :163 ✓
- F3: D5 dates identical across §8/§10/§11 incl. absolute confirm-by 2026-07-31 ✓
missing for 10: §7 primary Target cell "Gate A: ≥2 fizető…" drops the ≥1-failure conjunct that §8/§9 carry — incomplete restatement (label imprecision, not contradiction).

## Actionability — SCORE: 9/10
- F1: F1 acceptance is executable Given/When/Then (prd.md:117) per template §6 ✓
- F2: open questions carry owner + absolute resolve-by (prd.md:220) ✓
- F3: Gate A bar concrete, "interest doesn't count" (prd.md:176) ✓
missing for 10: the capacity/throughput model gating D5-date confirmation exists only as a risk-register row with NO resolve-by and no §10 open-questions row — a delivery planner can't tell when it resolves.

SUMMARY: EG=9 C=9 IC=9 A=9

# Round 2 — Blind scorer A (common metrics) — verbatim output
Fresh opus instance (a75239a7a5573a510), 2026-07-24; scores-round1/ + manifest ledger off-limits. Verified the committed test log exists and matches (176 tests, commit 0b56a13).

## Evidence-grounding — SCORE: 9/10
Sampled 14+ claims; all trace to the inventories or are properly labeled:
- §0/§1 two-layer thesis → [ARCH §0] ✓; §1 position-drift → [CMS NP-1] ✓
- §6 F4 skeleton-clone addItem → [CMS apply-ops.ts:147 + guardian.ts:47] ✓
- §6 F5 immutable snapshot + rollback → [CMS publish.ts:14,140] ✓
- §9 Gate B "176 teszt zöld" + CSP → verified against the committed test log (19 files / 176 tests, commit 0b56a13) and serving.ts:3 ✓
- §9 honesty note "capability-szinten bizonyított, NEM a literál vibor-oldalon" → [CMS note] exact ✓
- §4/§7 pricing + ~216k HUF/év → [GTM §3.5] ✓
- §8/§11 [D5] dates labeled Provisional decision with rationale/owner/confirm-by → per rubric does not count against grounding ✓
- §10 A2 derived-from-D2 flag consistent ✓
missing for 10: §7 guardrail Target attributes ">10% növekedés = vizsgálat" to [D3], but D3 only states "must not rise" — the 10% band is an author-added operationalization; label it as operational threshold rather than sourcing to D3.

## Completeness — SCORE: 9/10
All sections present; no placeholders; mandatory non-goals (seven redirect-named exclusions); §6 NFR all 8 categories (Performance + Accessibility as labeled Research gaps with plan + owner); §10 three separate tables per house rules; §7 primary + guardrail with role/timing; schemas preserved.
missing for 10: §11 appendix names the CMS evidence but does not restate the pinned commit (0b56a13…) in-doc — it lives only in evidence-cms.md.

## Internal consistency — SCORE: 9/10
- §8 Phase-1 entry ("Gate A PASS… nincs párhuzamos spike") consistent with §9 ordering and ARCH §16.6 ✓
- §9 Gate B threshold requires only what the spike builds; the fuller §15.3/(A) condition relocated to Phase-2 exit — §8 Phase 2 Exit row carries it ✓
- §7 guardrail baseline measurable; its rise is the exact §10 risk-register trigger ✓
- §10 dates match §8 and the authoritative §11 D5 row ✓
- §10 A1 is the exact De Morgan negation of §9 Gate A (P≥2 AND F≥1 → P<2 OR F=0), correctly noting the 2-payers-no-failure branch also kills the thesis ✓
missing for 10: §7 primary Target reads "Gate A: ≥2 fizető…" labeling one conjunct as the whole Gate A bar (full bar also requires ≥1 documented failure); defensible for a payer-count metric row but slightly loose framing.

## Actionability — SCORE: 9/10
Owners named; thresholds numeric/observable (≥2 paying, ≥1 failure, ≥2× length across fixed matrix, >10% support-hours); gate experiments concrete/executable; resolve-by dates absolute; pivots stated. A reader can execute Phase 0 without clarifying questions.
missing for 10: Performance budgets and WCAG level deferred (gated, owned, but un-numbered); D5 confirm-by was event-relative at scoring time.

SUMMARY: EG=9 C=9 IC=9 A=9

# Round 3 — Blind scorer B (PRD metrics) — verbatim output
Agent a9e7d3198bccfdb08, 2026-07-24. Fresh instance; prior-round material excluded. Verified CMS permalink formula (full-SHA blob permalink) + committed test log (19/19 files, 176/176 tests) before scoring.

## FALSIFIABILITY — SCORE: 9/10
- F1: A1 falsification = exact De Morgan negation within fixed window (quote prd.md §10 A1; verified vs decision-register D1) ✓
- F2: Gate B numeric + tightly scoped (≥2× length, every matrix cell, no manual retune; "Úgy tűnik, működik nem elég") — verified vs [NP-3] + [ARCH §16.6] ✓
- F3: interest explicitly excluded as evidence — verified vs [ARCH §15.6] ✓
- F4 DEFECT (soft edge): A4 "lényegesen 70 fölött" is not a hard threshold — a measured 74h doesn't unambiguously falsify.
missing for 10: harden A4 to a numeric breach point; define the guardrail violation window (see MQ F4).

## HONESTY — SCORE: 9/10
- F1: self-undermining redirect row preserved ("Ritka szerkesztés → Ne épüljön CMS") — verified vs [ARCH §12.4] ✓
- F2: sharpest limit with owned consequence ("a mitigáció szűkíti a megkülönböztetőt… BIZONYÍTANIA kell, nem feltételeznie") — verified vs [ARCH §16.6]/[§12.3] ✓
- F3: promo caveat consistent everywhere (§4 note, §9 Gate C, §10 A2, §11 derived row) — verified vs register ✓
- F4: spike execution unambiguously gated on Gate A PASS (§11 + §8 agree; generic template ordering explicitly overridden) ✓
missing for 10: one concrete example animation for the Webflow-overlap set vs the remaining unique set would make "szűkíti" inspectable.

## METRIC QUALITY — SCORE: 9/10
- F1: primary behavioral + instrumented ("mérni, nem kérdezni") — verified vs D3 ✓
- F2: guardrail has defined baseline window + numeric tolerance (pre-self-edit onboarding monthly avg, >10%) — verified vs D3 + [GAP-G4] ✓
- F3: role/timing separated on every row; anti-gaming pairing real ✓
- F4 DEFECT: three coexisting bars — "nem emelkedhet" / ">10% = vizsgálat" / "tartós növekedés = sértés" with "tartós" carrying no duration. The violation threshold is not measurable as written.
missing for 10: reconcile to one unambiguous breach condition (e.g. "≥2 egymást követő hónap >10% a baseline felett = sértés").

SUMMARY: F=9 H=9 MQ=9

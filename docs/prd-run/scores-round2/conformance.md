# Round 2 — Conformance reviews — verbatim outputs

## Round 2 (fresh instance a77f9ae53551d5035) — FAIL
CONFORMANCE: FAIL
1. §11 Decision register, row D5, confirm-by field — "confirm-by: e PR merge-e": bare event-relative reference with NO absolute date; the DATES-STRICT rule requires every date-bearing field to carry an absolute date. (All other checks pass: sections/order/headings, all table schemas exact, zero placeholders, one H1, header filled, non-goals with better-served names, Gates A/B/C thresholds, register present, links resolve. Every OTHER date-bearing field is absolute; the single defect is isolated to D5's confirm-by.)

## Fix applied (one cell)
"confirm-by: e PR merge-e" → "confirm-by: 2026-07-31 (legkésőbb e PR merge-ekor)".

## Round 2b (fresh instance adb9fb9a19ee17b24) — full independent re-check — PASS
CONFORMANCE: PASS
- Sections §0–§11 in order with exact template headings; one H1; header filled; zero placeholders.
- Table schemas exact: §2=4; §4=3; §6 functional=5 (F1–F10); §6 NFR=2×8 categories each with target or labeled gap+plan; §7=6 cols primary+guardrail; §8=4 cols incl. full-build; §10 three tables 4/7/4.
- Non-goals non-empty with better-served names; Gates A/B/C each with explicit falsifiable go-threshold; decision register in §11.
- DATES STRICT verified: header 2026-07-24; §8 targets 2026-08-08/08-31/11-30/12-12; §10 resolve-by 2026-08-08/2027-01-31/2026-08-08/2026-12-12; §11 D5 confirm-by 2026-07-31; sources dated 2026-07-24. Event context alongside dates allowed.
- Internal prd-run/ link resolves.
NOTE: D4 and the derived §10-A2 rows carry evidence-pointer sources (not relative dates) — no violation.
NOTE: scores-round1/ not opened (blindness preserved).

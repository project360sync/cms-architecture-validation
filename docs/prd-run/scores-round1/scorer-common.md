# Round 1 — Blind scorer A (common metrics) — verbatim output
Scorer: fresh opus subagent (id a7ead2c9b6eefddbd), 2026-07-24. Bundle: docs/prd.md @ pre-revision state, rubrics.md, pinned prd-template, evidence inventories, decision register, sibling source docs.

---

**Evidence-grounding — SCORE: 9/10**

Sampled 15 cited claims across §0/§1/§4/§5/§6/§7/§9/§10; every one traces cleanly:

- "két tisztán szétválasztott réteget... fejlesztő-birtokolt STRUKTÚRA (annotált HTML/CSS + verziózott, bezárt GSAP JS-bundle) és ügyfél-birtokolt, névhez kötött, tipizált TARTALOM [ARCH §0]" → evidence-arch [ARCH §0]. Supported.
- "a `data-cms-id` dokumentum-sorrend szerint sodródik átrendezéskor, a szerkesztések árván maradnak [ARCH §1][CMS NP-1]" → [ARCH §1] + [CMS NP-1]. Supported.
- §4 packaging table (Landing 250–500k / Business 500k–1M / Prémium 1–2M; Alap 6–10k / Plusz 15–25k / Pro 30–50k) → [GTM §3.1]+[GTM §3.2] exact match. Six margin levers → [GTM §3.3] exact match.
- §9 Gate B proven inventory: "176 teszt zöld [CMS package.json]" → matches; addItem/SRI/CSP/rollback match [CMS apply-ops.ts:147 / ingest.ts:164,166 / serving.ts:3 / publish.ts:14,140]. NOT-PROVEN NP-1..NP-5 match verbatim.
- §10 A4 "40–70h... nem 150 [GTM §3.4]" → matches. §9 Gate C "~1–3k HUF/oldal/hó → 3–5× árrés [GTM §3.2]" → matches.
- Provisional/derived items properly labeled (D2-derived WTP row "flagged / UNVALIDATED"; performance & accessibility NFRs as Research gaps with resolution plans + owner). No fabrication found.

missing for 10: sub-letter refs [ARCH §16.6.b]/[ARCH §16.6.c] require parsing the inventory's compound "(a)(b)(c)(d)" line; spelling the measure inline would make every citation atomically dereferenceable.

**Completeness — SCORE: 9/10**

- All 8 NFR categories assessed, none silently dropped (Security per-sink [ARCH §16.3]; Privacy&data with PII falsification assumption; Accessibility explicit justified research gap; Performance explicit "Research gap [GAP-A9]" with resolution plan).
- §6 Functional table keeps all 5 columns (F1–F10, Given/When/Then + source). §7 metrics all 6 columns; §10 risk table all 7; assumptions all 4; open questions all 4.
- §2 adds an "operator" persona row (additive). §4 has better-served table + sharpest-internal-limit + packaging table. Non-goals present, each naming a better-served incumbent.

missing for 10: §10 open-questions "Resolve-by" values are gate-relative ("Gate A indítása előtt") rather than absolute calendar dates.

**Internal consistency — SCORE: 9/10**

- Gate A threshold N=2 + ≥1 identical across §7 target, §8 Phase-0 exit, §9 Gate A, §10 A1 falsification, §11 D1. No drift.
- D2 promo price identical across §4 note, §9 Gate A anchor, §10 A2, §11 D2.
- Gate B go-threshold identical in §6 F3, §8 Phase-1 exit, §9 Gate B.
- Phase alignment clean: §5 in-scope = ARCH §15.3/(A) = §8 Phase 2; deferred B/C = §8 Phase 3. Guardrail consistent across §6/§7/§10/§11.

missing for 10: §6 Security cites greenfield target CSP `default-src 'none'` [ARCH §16.3] while §9 Gate B cites proven predecessor CSP `default-src 'self'` [CMS serving.ts:3] — different referents (target vs proven) but needs a one-line callout for a hostile skimmer.

**Actionability — SCORE: 9/10**

- Gate A fully specified: concierge to 2–3 real HU clients, named tools, concrete promo price, four instrumented measures, numeric go bar, "interest doesn't count."
- Gate C bar measurable with the measurement route (GTM §6 Q2 hour-tracking) named. Guardrail instrumented.
- Blocking prerequisites logged as open questions with owner + resolve-by.

missing for 10 / fixable: (1) Gate B can't literally begin until reference page + matrix chosen (open question but unresolved gate input). (2) No calendar dates / resourcing model — team can sequence but not schedule. (3) Every owner is "László" — no delegation guidance.

`SUMMARY: EG=9 C=9 IC=9 A=9`

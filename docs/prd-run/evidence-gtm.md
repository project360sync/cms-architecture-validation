# Evidence inventory — go-to-market.md
Source: /Users/jarvis/.claude/jobs/c0334220/tmp/cms-architecture-validation/go-to-market.md (repo main @ 9575ab8, Hungarian). FX assumption ≈400 HUF/USD.

[GTM §Header] Model: agency operates the CMS as internal tool; end-client buys a service (build + operation) and gets CMS access for self-editing. Status: proposal, for validation.
[GTM §1] Product: bespoke, animated (GSAP), mostly-static sites for SMB clients; one internal CMS instance serves N client sites → near-minimal per-site cost.
[GTM §1] Two priced items: one-time initial build + recurring operation. Client edits text/images/collection items; agency owns structure/design/animation.
[GTM §2] Primary ICP: Hungarian SMBs wanting a polished, custom, animated site they can update themselves; pains: "emailing the developer for every text change"; "want custom not template, but won't build it in Webflow".
[GTM §2] Agency-side value: less support burden + recurring revenue. Secondary ICP: mid-size companies' internal marketing teams.
[GTM §2] Explicit NON-targets: shops/transactions (→Shopify), DIY builders (→Webflow/Wix/Framer), content-heavy (→headless+SSG), enterprise i18n workflow.
[GTM §3.1] Build tiers: Landing 1–2 pages, light reflow-safe animation → 250–500k HUF; Business (base offer) multi-page, 1–2 collections, moderate animation → 500k–1M HUF; Prémium/bespoke full GSAP, multiple collections, i18n → 1–2M HUF; i18n option +150–400k/language.
[GTM §3.2] Operation tiers: Alap (hosting/CMS/SSL/backup/email support) 6–10k HUF/mo; Plusz (+1h changes/mo, priority) 15–25k; Pro (+SLA, more hours, i18n/SEO) 30–50k. Structural change beyond hours: 10–18k HUF/h. Annual prepay −1..−2 months.
[GTM §3.2] Agency infra cost ~1–3k HUF/site/mo (static edge + scale-to-zero CMS) → 3–5× margin even on Alap tier.
[GTM §3.3] Levers: (1) template/component reuse — first bespoke build funds the library, next similar client ~half hours → ~half price; (2) scope the animation — reflow-safe base cheap/robust, pin scroll-jack/SplitText = premium surcharge; (3) fixed packages not hourly; (4) low-but-real recurring (6–15k = HU-acceptable band); (5) annual prepay; (6) structural change billed hourly.
[GTM §3.4] Economics: with reuse a Business build ≈40–70 hours (not 150) → profitable at 500k–1M; recurring costs a few k HUF/site → portfolio LTV; self-editing reduces support → 8–15k/mo profitable.
[GTM §3.5] Sample deal: Business ~700k + Plusz ~18k/mo → year1 ~916k; ~216k/yr recurring thereafter. By 5th client build ~450–600k (reuse); recurring portfolio ~1M HUF/yr.
[GTM §4.1] HU market build prices: 200k–2M (template 50–300k; custom design+dev 500k–1.5M; big agency multi-million). Sources: newconcept.hu, klisestudio.hu, designpen.hu. HU operation/maintenance: 10–20k/mo typical (10–100k broad); 60–240k/yr. HU SaaS-hybrid precedent: reduced build for 9.5–50k HUF/mo over 1–2y commitment (qjob.hu, kiszervezettmarketing.hu).
[GTM §4.2] Builder prices: Webflow $14/$23/$39 → 5.6–15.6k HUF/mo; Framer free/$10/$30 → 4–12k; Squarespace $16–$99 → 6.4–40k; month-to-month ≈+30% vs annual.
[GTM §4.3] Comparison table: HU template 50–300k build + 5–15k/mo, template edits, no animation; HU WP-agency (default competitor) 500k–1.5M + 10–20k/mo, client emails for edits, rarely animation; DIY builder: build = client's own effort, 5.6–40k/mo, Framer has animation but DIY; US (bespoke+CMS): 250–500k/500k–1M/1–2M + 6–10/15–25/30–50k/mo, agency-built + client safely self-edits + animation.
[GTM §4.4] Positioning: our build band sits inside HU "custom" band (500k–1M) → realistically sellable; monthly at/below HU maintenance band AND level with DIY-builder subscriptions ("client pays Webflow-like monthly but gets a bespoke site + support + self-editing"); the gap = WP-agency price with animation + self-editing ("same price, better package"); the low-deposit + committed-recurring variant has HU precedent.
[GTM §5] Phase-2 self-serve idea (client registers, picks template, AI imports content from their old site): technically fits the onboarding-import; strategically a different, crowded, capital-heavy market (Webflow/Wix/Framer AI/Durable/Relume), low-ARPU volume play; risks: template library + AI-import quality cost, scrape weak on unusual sites, self-serve templates must be reflow-safe. Sequencing verdict: Phase 1 = agency (validate + build reflow-safe template library on real paid work); Phase 2 = self-serve freemium (~€10–30/mo) — explicit "don't start here".
[GTM §6] Validation questions the doc poses: Q1 (most important, demand): how many real HU clients want bespoke-animated + copy/image edit + agency structure-owner AND no adjacent solution better? → talk to 5–10 real leads. Q2 (real hours): actual hour count for Business build with/without reuse. Q3 (recurring acceptance): will HU SMB pay 6–15k/mo for self-edit+hosting+support? Q4 (offer structure): does low-deposit + 1–2y commitment convert better than high deposit?

GAPS (GTM facts the PRD needs that the doc lacks):
[GAP-G1] No named prospects/leads; zero committed/signed clients.
[GAP-G2] No measured build-hour data (40–70h is asserted, not observed); no no-reuse baseline.
[GAP-G3] No conversion/close-rate, lead-volume or win-rate data.
[GAP-G4] No measured edit frequency; "self-edit reduces support" has no data.
[GAP-G5] No churn/retention assumption quantified (LTV rests on unstated churn).
[GAP-G6] WTP unvalidated: 6–15k/mo acceptance and deposit-vs-recurring variant untested.
[GAP-G7] No CAC / sales-cycle estimate.
[GAP-G8] No agency capacity/throughput model.
[GAP-G9] Net vs gross (ÁFA) treatment of prices unstated.
[GAP-G10] Template-library cost/breakeven un-quantified.
[GAP-G11] Pro-tier SLA undefined (response/uptime).
[GAP-G12] Phase-2 self-serve pricing (€10–30/mo) is a placeholder; no market-size/unit-economics/go-criteria.

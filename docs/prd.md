# Client-safe CMS — PRD

One-liner: Ügynökség-üzemeltetett CMS bespoke, GSAP-animált, jellemzően statikus márkaoldalakhoz — fejlesztő birtokolja a struktúrát és a bezárt animáció-bundle-t, az ügyfél biztonságosan szerkeszti a szöveget, képet és gyűjtemény-elemeket; statikus publikálás az ügyfél saját domainjére.
Owner: László (project360sync) · Status: draft — QC: minden metrika 9/10 + conformance PASS; run: `verification pending` a Gate A/B zárásáig (ld. [prd-run/run-manifest.md](prd-run/run-manifest.md)) · Updated: 2026-07-24


## 0. TL;DR

Egy ügynökség által üzemeltetett, "client-safe" CMS, amely két tisztán szétválasztott réteget kezel: fejlesztő-birtokolt STRUKTÚRA (annotált HTML/CSS + verziózott, bezárt GSAP JS-bundle) és ügyfél-birtokolt, névhez kötött, tipizált TARTALOM [ARCH §0]. A célközönség magyar KKV-k, akik egyedi, animált, jellemzően statikus oldalt akarnak, amit a szöveg/kép/kártya szintjén maguk frissítenek, miközben a fejlesztő birtokolja a dizájnt és a mozgást [ARCH §12.1][GTM §2]. A központi fogadás: elég valódi ügyfél létezik ehhez a szűk niche-hez, akiket egyetlen létező eszköz (Webflow/Framer/headless CMS) sem szolgál ki jobban [ARCH §12.5]. A legfontosabb validálandó dolog nem a megvalósíthatóság, hanem a kereslet: az ARCH §12.5 pontos kérdését egy concierge-körben, valós (promó-)áron kell mérni, MIELŐTT bármi épülne [ARCH §16.6][D1]. Ez a PRD ezért egy kereslet-kaput helyez minden építés elé, és őszinte abban, hogy a szűk megkülönböztető érték (bespoke mozgás megőrzése meglévő, kézzel épített oldalon) a §14.5 reflow-diszciplínával szűkül — ezt a kapunak bizonyítania kell, nem feltételeznie [ARCH §16.6].


## 1. Problem & context

Mit próbál az ügyfél elérni, és hol törik el ma. Egy magyar KKV egyedi, art-directed, animált (GSAP) márkaoldalt akar, amelyet aztán maga tud frissíteni a szöveg, kép és ismétlődő kártyák szintjén — anélkül, hogy elrontaná a dizájnt vagy a mozgást. Ma ez ott törik el, hogy: (a) minden apró szövegmódosításért a fejlesztőnek kell emailezni [GTM §2]; (b) a jelenlegi, részben megépített pozíció-alapú modell törékeny — a `data-cms-id` dokumentum-sorrend szerint sodródik átrendezéskor, a szerkesztések árván maradnak [ARCH §1][CMS NP-1].

Hogyan oldják meg most (status quo / kerülőmegoldás). Három valós versenytárs: (1) a fejlesztő kézzel csinálja minden változtatásnál (email-alapú munkafolyamat) [GTM §2]; (2) magyar WordPress-ügynökség, ahol az ügyfél emailezik a szerkesztésekért, animáció ritkán [GTM §4.3]; (3) DIY builder (Webflow/Framer/Wix), ahol az egyedi dizájnt és animációt az ügyfélnek magának kell felépítenie — amit a célszemély nem akar [GTM §2][ARCH §12.4]. Ritka szerkesztéseknél (néhány/év) a helyes válasz gyakran az, hogy egyáltalán ne épüljön CMS — apró űrlap vagy "csináljuk meg helyetted" [ARCH §12.4].

Miért most, és miért nem elég a meglévő import. A modell a Shopify Online Store 2.0 sections-mintáját alkalmazza tetszőleges annotált HTML-re (nem sablonnyelvre), megőrizve a bespoke szerző-JS-t [ARCH §0]. A prior art nem old meg mindent: az Instatic egyirányú HTML-importja saját node-fába olvas, a store lesz a truth, és eldobja a `<script>`-et — épp a bespoke mozgást [ARCH §1]. A Shopify OS 2.0 (Liquid + `{% schema %}` + JSON template) erős prior art a névhez kötött robusztusságra, de NEM bizonyítja az annotált-HTML importot, az auto-migrációt, sem a bespoke-JS életciklust [ARCH §1]. A statikus edge-publikálás + scale-to-zero CMS mellett egy belső CMS-példány N ügyfél-oldalt szolgál ki, közel minimális per-oldal költséggel — ez teszi az ügynökségi üzemeltetést gazdaságossá [GTM §1][ARCH §4.3].


## 2. Target user (ICP)

Kétoldalú piac: az ügynökség üzemelteti a CMS-t belső eszközként, a végügyfél szolgáltatást vásárol (build + üzemeltetés) és önszerkesztéshez CMS-hozzáférést kap [GTM §Header][D4].

| Persona | Job-to-be-done | Pain today | Who pays |
|---|---|---|---|
| Elsődleges — magyar KKV marketinges/tulajdonos | Egyedi, animált, jellemzően statikus oldal, amit copy/kép/kártya szinten maga frissít, a dizájn birtoklása nélkül [ARCH §12.1][GTM §2] | Minden szövegváltásért emailezni kell a fejlesztőnek; egyedit akar, de Webflow-ban nem építi meg [GTM §2] | Végügyfél: egyszeri build + havi üzemeltetés [GTM §1] |
| Másodlagos — közepes cég belső marketingcsapata | Ugyanaz nagyobb léptékben; belső csapat végzi a tartalomfrissítést [GTM §2] | Ugyanaz | Végügyfél cég [GTM §2] |
| Üzemeltető — az ügynökség (mi) | Kevesebb support-teher + visszatérő bevétel; egy CMS-példány N oldalt szolgál ki [GTM §1][GTM §2] | Jelenleg minden ügyfélmódosítás fejlesztői munka; a pozíció-alapú modell törékeny [ARCH §1][GTM §2] | — (bevételt kap) |

Assumption: a másodlagos ICP (közepes cég belső csapata) valós vásárló — falszifikáció: ha a concierge-körben (§9 Gate A) egyetlen érdeklődő sem esik ebbe a szegmensbe, a v1 csak a KKV-perszónára szűkül. Owner: László.


## 3. Goals & non-goals

Goals (kimenetek, nem funkciók)
1. A tartalom túléli a fejlesztői átstrukturálást és újra-ingestet — nincs id-sodródás, nincs néma tartalomvesztés [ARCH §13][ARCH §0].
2. Az ügyfél csak az engedélyezett dolgokat szerkesztheti, és nem tudja tönkretenni a dizájnt vagy a mozgást; a GSAP fut a publikált oldalon, és a tartalom-szerkesztés nem töri el az animációt [ARCH §13][ARCH §2].
3. Az `addItem` és a mező-szerkesztés egyszerű, tipizált adatművelet; a modell framework-agnosztikus (tetszőleges statikus HTML annotálható) [ARCH §13][ARCH §2].

Non-goals (szándékos kizárások, jobb kiszolgáló megjelölésével)
- Nem sablonnyelv / nem általános template-runtime — ha a tézishez ez kellene, a headless CMS + SSG olcsóbb és jobban szolgálja a felhasználót [ARCH §2][ARCH §15.3].
- Nem általános drag-drop page-builder / kliens-oldali struktúraszerkesztés a palette-határokon túl — ez a Webflow/Framer/Wix/Squarespace niche-e (dev nélküli DIY, illetve kliens dizájn-szabadság) [ARCH §2][ARCH §12.2].
- Nem tranzakció / real-time / shop / booking — → Shopify / headless commerce [ARCH §12.2][ARCH §12.4].
- Nem újrafelhasznált bejegyzésekkel dolgozó tartalomgráf, és nem sok, adatból generált oldal — → headless CMS (Sanity/Contentful/Storyblok), illetve Astro/Next + adatforrás [ARCH §12.2].
- Nem enterprise lokalizációs workflow — → enterprise headless [ARCH §12.2][ARCH §12.4].
- Nem self-serve / freemium SaaS a v1-ben — ez explicit phase-2 non-goal ("ne itt kezdd"); külön, zsúfolt, tőkeigényes, low-ARPU piac (Webflow/Wix/Framer AI/Durable/Relume) [GTM §5][D4].
- App-szintű interaktivitás — → app-blocks / integrációk / embed-ek [ARCH §12.2].


## 4. Value proposition & honest limits

Why this, and why now. A megkülönböztető érték: egy meglévő, kézzel épített, bespoke-animált oldalon megőrizzük a szerző mozgását, miközben a struktúra fejlesztő-birtokolt marad, és az ügyfél biztonságosan szerkeszthet copy/kép/kártya szinten [ARCH §16.6][ARCH §12.1]. Az árazás realisztikusan eladható: a build-sáv a magyar "custom" sávon belül van (500k–1M HUF), a havidíj a magyar karbantartási sáv alatt/szélén és a DIY-builder előfizetésekkel egy szinten — "ugyanannyit fizet, mint egy Webflow-szerű havidíj, de bespoke oldalt + supportot + önszerkesztést kap"; a WP-ügynökséggel szemben "ugyanaz az ár, jobb csomag" [GTM §4.4].

A jelenlegi, részben megépített pozíció-alapú modell működik, de törékeny: a dokumentum-sorrend szerinti id-k átrendezéskor sodródnak, a szerkesztések árván maradnak [ARCH §1][CMS NP-1]. A greenfield tézis épp ezt szünteti meg — a tartalom NÉVHEZ kötött, nem HTML-pozícióhoz, így az átstrukturálás nem semmisíti meg az ügyfél tartalmát [ARCH §0]. Ez teszi biztonságossá a redesignt, triviálissá az `addItem`-et, és a séma-változást megjelölt migrációvá; a jelenlegi impl hibái mind a pozíció-horgonyzásból erednek [ARCH §8].

When a user is better served otherwise (ARCH §12.2/§12.4 átirányító sorok, hűen)

| If the user needs… | …they're better served by | Trigger to redirect |
|---|---|---|
| Shop / booking / tranzakció / real-time | Shopify / headless commerce | Fizetés, készlet, foglalás a core igény [ARCH §12.2][ARCH §12.4] |
| Újrafelhasznált bejegyzések tartalomgráfja | Headless CMS (Sanity / Contentful / Storyblok) | Ugyanaz a tartalom több helyen, entry-újrafelhasználás [ARCH §12.2] |
| Sok, adatból generált oldal | Astro / Next + adatforrás | Oldalak adatból generálódnak nagy számban [ARCH §12.2] |
| Dev nélküli DIY / kliens dizájn-szabadság | Webflow / Framer / Wix / Squarespace (vagy Shopify theme editor) | Az ügyfél maga akarja építeni/áttervezni a struktúrát [ARCH §12.2][ARCH §12.4] |
| Gyakori strukturális változtatás | Builder (Webflow / Framer) | Az oldal szerkezete gyakran változik [ARCH §12.4] |
| Több-nyelvű enterprise lokalizációs művelet | Enterprise headless | Enterprise i18n-workflow a core igény [ARCH §12.2][ARCH §12.4] |
| Ritka szerkesztés (néhány/év) | Ne épüljön CMS — apró űrlap vagy "csináljuk meg helyetted" | A szerkesztés annyira ritka, hogy a CMS nem térül meg [ARCH §12.4] |

Packaging (evidence-alapú, a value prop alátámasztására). Két árazott tétel: egyszeri build + visszatérő üzemeltetés [GTM §1]. A build-sáv a magyar "custom" sávon belül, realisztikusan eladható [GTM §4.4].

| Csomag | Build (egyszeri) | Üzemeltetés (havi) | Forrás |
|---|---|---|---|
| Landing (1–2 oldal, könnyű reflow-safe animáció) | 250–500k HUF | Alap 6–10k HUF/hó | [GTM §3.1][GTM §3.2] |
| Business (base offer; multi-page, 1–2 gyűjtemény, mérsékelt animáció) | 500k–1M HUF | Plusz 15–25k HUF/hó | [GTM §3.1][GTM §3.2] |
| Prémium / bespoke (teljes GSAP, több gyűjtemény, i18n) | 1–2M HUF | Pro 30–50k HUF/hó (+SLA) | [GTM §3.1][GTM §3.2] |

Árrés-levarek: (1) template/komponens-reuse — az első bespoke build finanszírozza a könyvtárat, a következő hasonló ügyfél ~fél óra → ~fél ár; (2) animáció-szkópolás — reflow-safe alap olcsó/robusztus, pin/SplitText prémium felár; (3) fix csomagok, nem óradíj; (4) alacsony-de-valós visszatérő (6–15k HU-elfogadható sáv); (5) éves előrefizetés; (6) strukturális változás óradíjas [GTM §3.3]. Megjegyzés: a v1 concierge-kör a D2 promó-árat használja (Business-termék Landing-áron), nem ezeket a listaárakat [D2].

A legélesebb belső korlát (GSAP × content-reflow). A hosszabb szöveg / több elem / eltérő képarány eltörheti a kézzel hangolt pin / mért-távolság / SplitText animációt; minél bespoke-abb az animáció, annál kevésbé biztonságos a szabad tartalom-szerkesztés [ARCH §12.3]. Mitigáció (§14.5): hossz-toleráns, kevés animált szekció; wrapper animálása szövegcsomópont helyett; futásidejű mérés + refresh; `maxLength` hintek; kötelező locale × viewport × font vizuális/overflow-teszt publikálás előtt; a `reflowSafe` fejlesztői ígéret, nem auto-levezethető [ARCH §14.5]. Őszinte állítás: a mitigáció szűkíti a megkülönböztetőt — a biztonságosan animálható halmaz átfed a Webflow/Framer natív képességeivel; a maradék egyedi érték a bespoke mozgás megőrzése meglévő oldalon + fejlesztő-birtokolt struktúra, amit a kereslet-kapunak BIZONYÍTANIA kell, nem feltételeznie [ARCH §16.6].


## 5. Scope — v1 / MVP

A v1 = az ARCH §15.3/(A) + §15.7 szerinti minimális spike, ami valós értéket ad ÉS teszti a technikai core-fogadást.

In scope (v1): — fixed-only, single-editor, single-locale [ARCH §15.3][ARCH §15.7]
- Kanonikus identitás + manifest; mezőidentitás = (scopeId, fieldName), sosem típus-kulcsolt; struktúra locale-független [ARCH §15.3][ARCH §16.1].
- Tipizált renderelés + validálás MINDEN íráskor (encode + kontextus-validálás) [ARCH §15.3][ARCH §4.2].
- Minimális rename/quarantine migráció + revízió + rollback [ARCH §15.3].
- Fixed GSAP-életciklus (mount/refresh/destroy) hosszabb szöveg alatt; determinisztikus animáció-újrainicializálás [ARCH §15.3].
- Item add/reorder szekciómozgatás nélkül; atomikus publikálás; kevert lock-policy [ARCH §15.3].
- Pesszimista, oldal-szintű edit-lock (heartbeat ~30s, TTL 2–5 perc, admin force-unlock, re-entráns per user, AI-as-user; mások read-only) [ARCH §15.7].
- Edit ÉS Preview külön-origin sandboxban [ARCH §15.7][ARCH §5].

Deferred (later phases):
- Phase B (pilot előtt): optimista konkurenciakezelés, teljes locale-policy, audit/RBAC, asset-életciklus + ops [ARCH §15.3].
- Phase C: composable mód (registry, add/remove/reorder, capabilities + tesztek, per-locale sorrend) [ARCH §15.3].
- Multi-locale i18n mint headline AI-fordítás-feature (adatot fordít, sosem markupot) [ARCH §14][ARCH §14.5].
- Production store: Postgres + object storage (a spike Mongón maradhat logikai prototípusként) [ARCH §16.5][ARCH §15.5].

Out of scope (this product): lásd §3 non-goals (sablonnyelv, drag-drop builder, shop/real-time, tartalomgráf, enterprise i18n, self-serve SaaS).


## 6. Requirements

Functional — az ARCH §15.3 (A) spike-listából; az elfogadási kritériumok az ARCH §15.3 Go-feltételre hivatkoznak.

| # | Requirement | Priority | Acceptance criteria / how verified | Source / journey |
|---|---|---|---|---|
| F1 | Névhez kötött identitás + manifest (nem pozíció-alapú) | Must | Given a szekciók átrendezése/átstrukturálása, When re-ingest fut, Then a tartalom névre reconcile-ódik, nincs id-sodródás és nincs néma vesztés (Go: "no content loss") [ARCH §15.3][ARCH §16.1] | ARCH §13; fejlesztői restructure-journey |
| F2 | Tipizált renderelés + validálás minden íráskor | Must | Given bármely íratlan/típushibás bemenet, When írás történik, Then a renderer kontextus szerint encode-ol és elutasít; nincs aktív injekció (Go: "no active injection") [ARCH §15.3][ARCH §4.2] | ARCH §5; kliens-szerkesztés-journey |
| F3 | Fixed GSAP-életciklus túléli a tipizált szerkesztést + hosszabb szöveget | Must | Given hosszabb szöveg / több item egy valós referencia-oldalon, When újrarenderel, Then determinisztikus animáció-re-init, nincs törött pin/measured animáció (Go: "deterministic animation re-init with longer text") [ARCH §15.3][ARCH §16.6] | ARCH §12.3; Gate B spike |
| F4 | Item add/reorder szekciómozgatás nélkül | Must | Given `data-cms-item` prototípus, When addItem/reorder, Then a vetett skeleton-markup klónozódik friss id-kkel, szekció nem mozdul [ARCH §15.3][CMS apply-ops.ts:147] | ARCH §3.2; kártya-szerkesztés |
| F5 | Atomikus, rollbackolható publikálás statikus edge-re | Must | Given publish, When pointer-flip, Then immutábilis snapshot, egykattintásos rollback (Go: "atomic/rollbackable publish") [ARCH §15.3][ARCH §4.3][CMS publish.ts:14,140] | ARCH §4.3; publish-journey |
| F6 | Minimális rename/quarantine migráció + revízió + rollback | Must | Given schema-változás, When migráció, Then eltűnt slotok karanténba kerülnek (nem néma dobás), explicit manifest nyer az auto-name-match felett, dry-run diff emberi jóváhagyással [ARCH §15.3][ARCH §16.2] | ARCH §4.4; migráció-journey |
| F7 | Granuláris, manifestből fordított, szerver-oldalon kikényszerített szerkesztő-jogosultságok | Must | Given fixed mód, When jogosultság-ellenőrzés, Then permission ≤ capability, kompozíciós jogok kényszerítve false-ra, három független lock (content/position/structure) [ARCH §3.4.1][ARCH §5] | ARCH §5; jogosultság-journey |
| F8 | Hibrid szerkesztő (inline click-to-edit + settings panel) + AI-chat mint tipizált op | Should | Given AI-chat vagy inline szerkesztés, When írás, Then ugyanaz a tipizált op-út, Guardian-validált, a kliens sosem ad markupot; AI review-staging bulk-megerősítéssel + rate-limit [ARCH §5][ARCH §16.4] | ARCH §5; szerkesztő-UX |
| F9 | Kevert lock-policy per szekció (default minden lock, csak jelölt rész szerkeszthető) | Should | Given annotáció, When manifest fordul, Then default-locked, csak jelölt mezők szerkeszthetők; stabil section-instance id-k (nem pozícióból) [ARCH §3.2][ARCH §15.3] | ARCH §3.2 |
| F10 | Composable mód (registry, add/remove/reorder) | Won't (v1) | Phase C — a v1-ben tiltva, bespoke GSAP default fixed [ARCH §15.3][ARCH §15.5] | ARCH §15.3 |

Non-functional — minden kategória értékelve.

| Category | Target (or `N/A` + reason) |
|---|---|
| Performance | Research gap: nincs definiált budget (load/latency/throughput) — [GAP-A9]. Resolution plan: a Gate B narrowed spike-nál kell felállítani a locale × viewport × font mátrixhoz kötött publish-előtti overflow/vizuális teszt küszöbeit és a render-időbudgetet; tulajdonos László, a spike-referenciaoldal kiválasztásakor [ARCH §14.5][GAP-A7]. |
| Security | Publikált CSP `default-src 'none'`, connect-src allowlist, nosniff; postMessage origin/schema-validáció; külön regisztrálható domain [ARCH §16.3]. Import-izoláció: SSRF egress-allowlist, unpack-limitek, credential-less build, dependency-provenance [ARCH §16.3]. Minden sink per-sink policy (text/richtext/url/image/color/select/svg + CSS-as-content explicit sink) [ARCH §16.3]. A kliens sosem ad markupot; egyetlen mutációs út Guardian mögött [ARCH §5][CMS DESIGN.md:10-13]. Már bizonyított capability-szinten: inline+on* strip, SRI-pin, `script-src 'self'` [CMS ADR-003][CMS serving.ts:3]. |
| Privacy & data | Tartalom tipizált/sémázott/verziózott; content store a truth az onboarding-import után [ARCH §7][ARCH §4.1]. Assumption: a v1 nem kezel különleges személyes adatot (marketing-copy/kép) — falszifikáció: ha egy pilot-ügyfél PII-t vinne be gyűjtemény-mezőkbe, retention/deletion-policy kell. Owner: László. Formális GDPR-retention/deletion policy = [GAP-A9]-hez kötött. |
| Reliability | Publish = immutábilis statikus snapshot az edge-en (version freeze + atomikus pointer-flip), egykattintásos rollbackSite() [ARCH §4.3][CMS publish.ts:14,140]. Szerkesztő-CMS scale-to-zero, nincs a látogató útjában [ARCH §4.3]. Draft/publish konzisztencia: revízió-alapú draft + atomikus release-pointer [ARCH §15.5]. Uptime/SLA szám: lásd Open questions [GAP-G11]. |
| Accessibility | Research gap: az ARCH/GTM nem definiál WCAG-szintet. Resolution plan: a publikált oldalak akadálymentessége a build-sablon fejlesztői felelőssége; a Gate B referenciaoldalnál kell megcélzott WCAG-szintet (pl. WCAG 2.1 AA) rögzíteni. Owner: László. |
| i18n / l10n | v1 = csak `hu`, single-locale [ARCH §15.3][ARCH §15.7]. Multi-locale (locale-dimenzió a tartalomrétegen, per-mező translated/untranslated/stale, publish-policy required/fallbackAllowed/hiddenWhenMissing, AI-fordítás mint headline feature) = Phase B/C-re halasztva [ARCH §14]. Reflow-érzékenység a §14.5 mitigáció alá esik. |
| Observability & operability | Ops (kvóták/observability/backup) = P1, pilot előtt [ARCH §15.2]. Audit-trail: v1 revision-guard (minden írás base-revíziót hordoz, elavult írás elutasítva; force-unlock fencing-token) [ARCH §16.4]; teljes audit/RBAC Phase B [ARCH §15.3]. Guardrail-instrumentáció: self-edit események + support-órák/ügyfél mérése (lásd §7) [D3]. |
| Scale | v1 léptéke = ügynökségi portfólió: egy belső CMS-példány N ügyfél-oldalt szolgál ki, statikus edge + scale-to-zero → közel minimális per-oldal költség [GTM §1][ARCH §4.3]. Következő nagyságrend: 5. ügyfélnél a build reuse-szal ~450–600k HUF, a visszatérő portfólió ~1M HUF/év [GTM §3.5]. Tenant/kapacitás-limitek explicit számmal = [GAP-A9]/[GAP-G8]. |


## 7. Success metrics

Baseline mindenhol 0 (pre-launch) — a termék még nem indult, nincs mért érték [GAP-A4][GAP-A6].

| Metric | Role | Timing | Baseline | Target | How measured |
|---|---|---|---|---|---|
| Havonta ≥1 instrumentált önszerkesztést végző fizető ügyfelek száma | primary | lagging | 0 (pre-launch) | Gate A: ≥2 fizető concierge-ügyfél instrumentált szerkesztéssel [D1] | Szerkesztő-események instrumentálása — mérni, nem kérdezni [D3][ARCH §16.6.b] |
| Ügynökségi support-órák / ügyfél / hónap | guardrail — nem romolhat | leading | Per-ügyfél pre-self-edit baseline: a self-edit hozzáférés megnyitása ELŐTTI onboarding-időszak support-óráinak havi átlaga, a Gate A concierge indulásakor mérve (ma nincs adat [GAP-G4] — a baseline-mérés a Gate A része) | A saját baseline-ablakához képest nem emelkedhet; >10% növekedés = vizsgálat, tartós növekedés = guardrail-sértés [D3] | Support-óra naplózás ügyfélenként, a baseline-ablaktól folyamatosan [D3][GAP-G4] |
| Szerkesztő-események (edit events) / ügyfél / hónap | supporting (vezető jelző) | leading | 0 (pre-launch) | Nem-nulla, ismétlődő szerkesztés a concierge-körben [ARCH §16.6.b] | Instrumentált edit-event telemetria [ARCH §16.6] |
| Visszatérő bevétel / ügyfél (MRR-jellegű) | supporting (lemaradó) | lagging | 0 (pre-launch) | Minta-deal: ~216k HUF/év visszatérő ügyfelenként; 5. ügyfélnél portfólió ~1M HUF/év [GTM §3.5] | Számlázott üzemeltetési díj [GTM §3.2][GTM §3.5] |


## 8. Milestones & phasing

Teljes build csak minden alkalmazandó §9 kapu után indul. Cél-dátumok: [D5] Provisional decision (kapacitásmodell nélkül becsült, megerősítendő): prospekt-lista + ÁFA-döntés 2026-08-08; első concierge-ajánlat 2026-08-31; Gate A értékelési ablak zárása 2026-11-30; Gate B referenciaoldal-választás 2026-12-12. Az erőforrás/kapacitásmodell továbbra is Research gap [GAP-A5][GAP-G8] — a dátumok emiatt provisional státuszúak.

| Phase | Goal | Entry condition | Exit / done-when |
|---|---|---|---|
| 0 — Kereslet-kapu (concierge) | Az ARCH §12.5 pontos kérdésének mérése valós, promó-áron, kizárólag létező eszközökkel [ARCH §16.6][D1][D2] | PRD draft elfogadva; kiválasztott valós HU prospektek | Gate A teljesül: ≥2 fizető concierge-átadás instrumentált szerkesztéssel + ≥1 dokumentált off-the-shelf-kudarc bespoke GSAP-on [D1] |
| 1 — Szűkített technikai spike | Kizárólag a GSAP × reflow kockázat: mount/refresh/destroy túléli a tipizált szerkesztést + hosszabb szöveget egy VALÓS referenciaoldalon, locale × viewport × font mátrixban — NEM a teljes platform [ARCH §16.6] | Gate A PASS [D1] — ennél a projektnél a demand-first sorrend kötött [ARCH §16.6][ARCH §16.7]: nincs párhuzamos spike elindított-de-le-nem-zárt kereslet-kapu mellett | Gate B teljesül: determinisztikus mount/refresh/destroy + hosszváltozás a rögzített mátrixon, manuális újrahangolás nélkül (§9 Gate B küszöb) |
| 2 — Thin slice → pilot | Az ARCH §15.3/(A) minimális, fixed-only, single-editor, single-locale spike valós pilot-ügyféllel | Minden alkalmazandó kapu (A, B, és ahol releváns C) zöld | A thin slice pilotban validál: nincs tartalomvesztés, nincs aktív injekció, atomikus/rollbackolható publish, determinisztikus animáció hosszabb szöveggel [ARCH §15.3] |
| 3 — Teljes build | A platform proper: ARCH §15.3/(B) majd (C) — konkurencia, teljes locale-policy, audit/RBAC, asset-életciklus, majd composable | A pilot validálja a thin slice-t | §15.3/(B) és (C) tételek leszállítva a P1/P2 kapukkal [ARCH §15.2][ARCH §15.3] |


## 9. Validation gates (go / no-go)

Minden alkalmazandó kapunak passzolnia kell a teljes build előtt; a legolcsóbb, legdöntőbb elöl. Az ARCH §16.6 szerint a sorrend megfordítva: kereslet-kapu a technikai spike ELŐTT.

Gate A — Demand.
- Kísérlet: concierge / Wizard-of-Oz — client-safe szerkesztés átadása 2–3 valós HU ügyfélnek KIZÁRÓLAG létező eszközökkel (Storyblok Visual Editor vagy edit-N-mezős űrlap egy bespoke buildon) valós áron [ARCH §16.6]. Négy instrumentált mérés: (a) vesznek-e az áron, (b) ténylegesen szerkesztenek-e és milyen gyakran, (c) az off-the-shelf demonstrálhatóan elbukott-e a GSAP-on, (d) maradnak-e vs. átcsúsznak Shopify/Webflow/semmi felé [ARCH §16.6].
- Ár-anker [D2]: launch-promó — Business-szintű termék Landing-sávos áron (250–500k HUF build + 6–10k HUF/hó), explicit kommunikálva + követve a felső kategória promójaként.
- Go threshold [D1]: ≥2 fizető concierge-átadás valós (promó-)áron instrumentált szerkesztéssel, ÉS ≥1 dokumentált eset, ahol egy off-the-shelf szerkesztő (pl. Storyblok / edit-N-mezős űrlap) demonstrálhatóan elbukott a bespoke GSAP-on. Érdeklődés vagy "jó lenne" nem számít [ARCH §16.6][D1].

Gate B — Feasibility / technical.
- A legkockázatosabb ismeretlen: GSAP × content-reflow egy valós referenciaoldalon — a mount/refresh/destroy túléli a tipizált szerkesztést + hosszabb szöveget locale × viewport × font mátrixban, migrációs motor / atomikus publish / security-render pipeline front-loadolása NÉLKÜL [ARCH §16.6].
- Már bizonyított [CMS] (nem ezt támadja a spike): külső `<script src>` capture same-origin asseté + SRI-pin [CMS ingest.ts:164,166]; allowlist-sanitize (inline+on* dobva) [CMS sanitize.ts]; addItem klónoz vetett skeletont friss id-kkel, Guardian-gated [CMS apply-ops.ts:147]; publikált CSP `default-src 'self'` [CMS serving.ts:3]; immutábilis publish + rollback [CMS publish.ts:14,140]; 176 teszt zöld [CMS package.json]. Megjegyzés: a GSAP-feasibility capability-szinten bizonyított, NEM a literál vibor-oldalon [CMS note].
- NEM bizonyított — ezt kell a spike-nak támadnia: névhez kötött tartalom (az id-k pozíció-alapúak) [CMS NP-1]; re-ingest reconciliation (implementálatlan) [CMS NP-2]; GSAP túléli a content-reflow-t hosszváltozás alatt — nincs teszt/fixture [CMS NP-3]; composable mód [CMS NP-4]; multi-locale [CMS NP-5].
- Go threshold (csak a spike scope-jára): determinisztikus GSAP mount/refresh/destroy + animáció-re-init ≥2× szöveghossz-változás mellett, a valós referenciaoldalon, az előre rögzített locale × viewport × font mátrix minden celláján, manuális újrahangolás nélkül [ARCH §16.6][CMS NP-3]. "Úgy tűnik, működik" nem elég. — A teljes ARCH §15.3/(A) Go-feltétel (nincs tartalomvesztés, nincs aktív injekció, atomikus/rollbackolható publish) nem ennek a kapunak a küszöbe: azok a Phase 2 thin-slice exit-feltételei, mert az ott felépülő rendszereket mérik [ARCH §15.3].

Gate C — Viability (business). Alkalmazandó.
- Margin-evidencia: ügynökségi infra-költség ~1–3k HUF/oldal/hó → 3–5× árrés már az Alap üzemeltetési tieren is [GTM §3.2]; reuse-szal a Business build ~40–70 óra → profitábilis 500k–1M-nél [GTM §3.4].
- Nyitott bemenetek, amelyeket a kapunak zárnia kell: (1) full-price WTP feltevés (D2-ből származó) — a promó-eladás csak promó-feltételen bizonyít keresletet; (2) valós óraszám kérdése [GTM §6 Q2] — a 40–70h állított, nem mért [GAP-G2].
- Go threshold: kontribúciós margin pozitív a cél(nem-promó)áron MÉRT szállítási költséggel; a full-price WTP legalább egy nem-promó Business-ajánlattal megerősítve.

If a gate fails (pivot) [ARCH §15.6][ARCH §15.3]: kettős go/no-go — Architecture-go (spike-tesztek passzolnak) ÉS Product-go. Ha csak az egyik passzol: nincs teljes platform-build. Technikai kudarc → pivot headless CMS + SSG integrációra (illetve ha a tézishez általános template-runtime / tartalomgráf kellene, a headless+SSG olcsóbb és jobban szolgál) [ARCH §15.3][ARCH §12.4]. Kereslet-kudarc → a spike belső ügynökségi eszköz marad, vagy leáll [ARCH §15.6]. Viability-kudarc → újraárazás, újraszkópozás vagy leállás.


## 10. Assumptions, risks & open questions

Key assumptions (plan-killers).

| Assumption | Why load-bearing | Falsification criterion | Test / owner |
|---|---|---|---|
| A1 — Létezik a kereslet: elég valós HU ügyfél = bespoke-animált oldal + copy/kép-szintű szerkesztés + fejlesztő a hurokban — ÉS egyetlen adjacent megoldás sem szolgálja jobban [ARCH §12.5] | Ha nincs, nincs termék; ez a legfontosabb nem-technikai validáció | A D1 go-feltétel pontos negációja a rögzített mintán/időablakban (D5, zárás 2026-11-30): kevesebb mint 2 fizető átadás VAGY nulla dokumentált, releváns off-the-shelf-kudarc — tehát 2 fizető ügyfél MELLETT, off-the-shelf-kudarc nélkül is bukik a tézis (akkor a meglévő eszköz elég, nem kell egyedi architektúra) [D1] | Gate A concierge / László |
| A2 — Full-price WTP a promó után: a Business-áras (500k–1M) build valós fizetési szándékot élvez, nem csak a promó-terms mellett | Az LTV és a §9 Gate C erre épül; a D2 promó csak promó-keresletet bizonyít | Az első nem-promó, Business-áras ajánlatot ≥2 egyébként illő prospekt visszautasítja (decision register, D2-ből származó feltevés) | Gate C / László |
| A3 — Az off-the-shelf eszközök ténylegesen elbuknak a bespoke GSAP-on | Ha nem buknak el, a megkülönböztető érték eltűnik | Storyblok / edit-N-mezős űrlap gond nélkül kiszolgál egy bespoke GSAP-buildot (→ ARCH §16.6.c) [GAP-A8] | Gate A (c) mérés / László |
| A4 — 40–70h reuse-gazdaságosság: reuse-szal a Business build ~40–70 óra (nem 150) → profitábilis [GTM §3.4] | Az árrés és a fix-csomag árazás erre épül | A GTM §6 Q2 mérés a tényleges óraszámot lényegesen 70 fölött találja [GAP-G2] | GTM §6 Q2 óra-mérés / László |
| A5 — A GSAP túléli a reflow-t a §14.5 diszciplínával | A core technikai fogadás; a v1 értékajánlat ezen áll | A spike-ban a determinisztikus re-init nem érhető el manuális újrahangolás nélkül hosszabb szöveg alatt [CMS NP-3] | Gate B spike / László |

Risk register (operational).

| Risk | Likelihood | Impact | Trigger to watch | Mitigation | Contingency | Owner |
|---|---|---|---|---|---|---|
| Kulcsember / ügynökségi kapacitás — nincs throughput-modell [GAP-G8] | M | H | Egyszerre több pilot, a szállítás csúszik | Fix-csomag szkópolás; reuse-könyvtár; kapacitásmodell felállítása | Pilotok sorbaállítása; új projektek befagyasztása | László |
| Template-könyvtár költsége / breakeven un-quantified [GAP-G10] | M | M | Az első buildek nem termelik ki a könyvtárat | Első bespoke build finanszírozza a könyvtárat; reuse-mérés [GTM §3.3] | Szűkebb sablon-scope; kevesebb animáció-preset | László |
| Support-teher, ha az önszerkesztés-feltevés bukik [GAP-G4] | M | M | Guardrail-metrika: support-órák/ügyfél emelkednek [D3] | Instrumentált edit-események; jobb szerkesztő-UX; hint-ek | Több beépített "csináljuk helyetted" óra a tierbe | László |
| Biztonsági incidens publikált oldalon | L | H | CSP-sértés / injekció-kísérlet a publikált snapshotban | ARCH §16.3 kontrollok: CSP `default-src 'none'`, sink-policy-k, import-izoláció, külön domain, postMessage-validáció | Snapshot-rollback; sink-policy szigorítás; bundle-újra-pin | László |

Open questions (decision log).

| Question | Owner | Resolve-by | Decision (once made) |
|---|---|---|---|
| ÁFA / nettó-bruttó árkezelés a tiereknél [GAP-G9] | László | 2026-08-08 (az első concierge-ajánlat előtt, [D5]) | — |
| Pro-tier SLA definíció (válaszidő/uptime) [GAP-G11] | László | 2027-01-31 (a Phase 2 pilot előtt, [D5] provisional) | — |
| Nevesített prospekt-lista (0 committed/signed) [GAP-A1][GAP-G1] | László | 2026-08-08 [D5] | — |
| Spike referenciaoldal + locale/viewport/font mátrix kiválasztása [GAP-A7] | László | 2026-12-12 (Gate A zárása után, Gate B indítása előtt, [D5]) | — |


## 11. Appendix & references

Evidence documents.
- Architektúra / validáció: `cms-greenfield-architecture.md` (repo main @ 9575ab8, magyar) — [ARCH] hivatkozások forrása.
- Go-to-market: `go-to-market.md` (repo main @ 9575ab8, magyar; FX ≈400 HUF/USD) — [GTM] hivatkozások forrása.
- Feasibility / Gate B bemenet: claude-cms kódbázis, `docs/adr/ADR-003-author-js-and-editable-collections.md` + a proven/NOT-PROVEN inventár — [CMS] hivatkozások forrása (working prototype, pozíció-anchored predecessor).
- Kit-sablonok @ commit `9257a9b29c9192ccecf42054577145d4cace226f`.
- Run-bundle (evidencia-inventárok [ARCH]/[GTM]/[CMS] sorai, decision register, run-manifest a pontozási ledgerrel): [`prd-run/`](prd-run/).

Decision register (bemásolva).

| id | Question + options | Selected | Status | Source |
|---|---|---|---|---|
| D1 | Gate A go-threshold: N=2 fizető + ≥1 dokumentált off-the-shelf-kudarc / N=3 mind-bukó / N=1+2 committed | N=2 fizető concierge-átadás valós áron instrumentált szerkesztéssel, ÉS ≥1 dokumentált eset, ahol egy off-the-shelf szerkesztő demonstrálhatóan elbukott a bespoke GSAP-on | confirmed | user answer, 2026-07-24 |
| D2 | Concierge "valós ár" anker: Business-sáv / Landing-sáv / mixed | Launch-promó: Business-szintű termék Landing-sávos áron (250–500k HUF build + 6–10k HUF/hó), explicit promóként kommunikálva + követve | confirmed | user answer (custom), 2026-07-24 |
| D3 | Primary metrika: fizető-ügyfelek ≥1 self-edit/hó / MRR / összes edit-event | Havonta ≥1 instrumentált self-editet végző fizető ügyfelek száma; guardrail: ügynökségi support-óra/ügyfél/hó nem emelkedhet | confirmed | user answer, 2026-07-24 |
| D4 | Termék-framing: belső ügynökségi eszköz vs SaaS | Ügynökség-üzemeltetett phase 1; self-serve = explicit phase-2 non-goal ("ne itt kezdd") | confirmed (evidence) | GTM §5 + ARCH §12.1 |
| — | Derived from D2 (§10 A2) | Full-price WTP (500k–1M build) UNVALIDATED — a promó-eladás csak promó-terms-en bizonyít; falszifikáció: első nem-promó Business-áras ajánlatot ≥2 illő prospekt visszautasít | flagged | decision register |
| D5 | Cél-dátumok (kapacitásmodell nélkül): prospekt-lista + ÁFA 2026-08-08 / első ajánlat 2026-08-31 / Gate A zárás 2026-11-30 / Gate B ref-oldal 2026-12-12 / SLA 2027-01-31 | Javasolt dátumok elfogadva provisional-ként | provisional — rationale: kapacitás-adat nélküli becslés; owner: László; confirm-by: 2026-07-31 (legkésőbb e PR merge-ekor) | orchestrátor-javaslat, 2026-07-24 |

Technical design pointer. A HOGYAN külön dokumentumban él (sibling `tech-spec-template.md`), `spike` mélységgel indulva, amely `delivery` mélységig mélyül. Ennél a projektnél a sorrend kötött [ARCH §16.6]: a spike-mélységű tech-spec *dokumentumként* előkészíthető korábban is (a Gate B kísérleti tervének rögzítésére), de spike-implementáció vagy -futtatás kizárólag Gate A PASS után indul (lásd §8 Phase 1 entry) — a template általános "Gate B lehet a legolcsóbb/legdöntőbb első kapu" opciója itt nem érvényes. Implementációs részlet ne kerüljön ebbe a PRD-be; ide linkelve.

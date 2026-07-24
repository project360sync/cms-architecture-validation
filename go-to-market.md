# Go-to-market — célcsoport, árazás, versenytárs-elemzés

> **Cél:** a [technikai spec](cms-greenfield-architecture.md) mellé az **üzleti
> feltevések** egy helyen, **validálásra** — mielőtt implementálunk vagy élesben
> árazunk. **Modell:** *mi vagyunk az ügynökség*, a CMS a belső eszközünk; a
> végügyfélnek szolgáltatást árazunk (build + üzemeltetés), az ügyfél megkapja a
> CMS-hozzáférést önszerkesztésre.
>
> **Státusz:** javaslat, validálásra. **Nyelv:** magyar. **Deviza:** ≈400 Ft/USD (kerekített).

---

## 1. Üzleti modell egy bekezdésben

Bespoke, animált (GSAP), jórészt statikus oldalakat építünk kkv-ügyfeleknek. A CMS a
**mi belső eszközünk** (egy instance, N ügyfél-oldal → oldalanként minimális a mi
költségünk). A végügyfélnek **két dolgot** árazunk: **(1) initial build** (egyszeri) és
**(2) üzemeltetés** (visszatérő). Az ügyfél **CMS-hozzáférést** kap: a szöveget/képet/
kollekció-elemeket maga szerkeszti; a struktúrát/dizájnt/animációt mi (a `data-cms`
„tartalom ⟂ struktúra" határ kereskedelmi leképezése).

---

## 2. Célcsoport (a mi végügyfeleink)

**Elsődleges:** magyar **kkv-k**, akik egy **igényes, egyedi, animált** oldalt akarnak,
amit **maguk tudnak frissíteni** (szöveg/kép/termék), és fizetnek a buildért + a
karbantartott, hosztolt, szerkeszthető oldalért.

**A fájdalom, amit megveszünk:**
- „Ne kelljen minden szöveg-módosításért a fejlesztőt emailezni."
- „Legyen szép/egyedi, ne sablon — de ne is kelljen nekem Webflowban építeni."
- Nekünk (ügynökség): **kevesebb support-teher** (az ügyfél önszerkeszt) + **visszatérő bevétel**.

**Másodlagos:** közép-vállalatok **belső marketing-csapata** fejlesztő-épített oldallal.

**NEM célcsoport** (lásd tech-spec §12): bolt/tranzakció (→ Shopify), DIY-építő (→ Webflow/
Wix/Framer), tartalom-nehéz/query-vezérelt (→ headless CMS + SSG), enterprise i18n-workflow.

---

## 3. Árazás (HU-realista, 2026)

### 3.1 Build — egyszeri

| Csomag | Mit fed | Ár (HUF) |
|---|---|---|
| **Landing** | 1–2 oldal, template-alapú, könnyű (reflow-biztos) animáció, CMS | 250–500e |
| **Business** *(alap ajánlat)* | több oldal, 1–2 kollekció, mérsékelt animáció, CMS + tartalom-feltöltés | 500e–1M |
| **Prémium / bespoke** | egyedi dizájn, **teljes GSAP**, több kollekció, i18n | 1–2M |
| *(opció)* i18n / nyelv | struktúra-előkészítés + első fordítás | +150–400e / nyelv |

### 3.2 Üzemeltetés — havi

| Csomag | Mit fed | Ár (HUF/hó) |
|---|---|---|
| **Alap** | hosting (statikus edge), **CMS-hozzáférés**, SSL, backup, email support | 6–10e |
| **Plusz** | + havi 1 óra változtatás, prioritás | 15–25e |
| **Pro** | + SLA, több óra, i18n/SEO (nagyobb ügyfél) | 30–50e |

- Strukturális változtatás a benne foglalt órán felül (dev-only): **~10–18e Ft/óra**.
- Éves előrefizetés: **−1–2 hónap**.
- A mi infra-költségünk: **~1–3e Ft/oldal/hó** (statikus + scale-to-zero CMS) → az Alap is 3–5× marzs.

### 3.3 A masszírozás leverei (csökkents, de védd a marzsot)

1. **Template/komponens-újrahasznosítás — a legnagyobb lever.** Az ELSŐ bespoke build drága
   (ő finanszírozza a könyvtárat); a következő hasonló ügyfél ugyanazt a strukturát/CMS-setupot
   kapja → **~fele munkaóra → ~fele ár.** A modell idővel olcsóbb, ahogy nő a könyvtár.
2. **Scope-old az animációt.** A reflow-biztos alap (reveal, marquee, konténer-fade) olcsóbb
   ÉS robusztusabb; a pin scroll-jack / SplitText = prémium felár. (A tech-spec §14.5-tel egybeesik.)
3. **Fix csomagok, ne óradíjas ajánlat** — átlátható az ügyfélnek, dobozolt scope neked, gyorsabb eladás.
4. **Alacsony, de létező recurring** (6–15e/hó a HU-elfogadható sáv) — a marzs a minimális infrából jön.
5. **Éves előrefizetés** — cash + kevesebb churn.
6. **Struktúra-változás óradíjas** (a csomag-órán felül) → az alacsony csomag marzsát ez védi.

### 3.4 Miért működik gazdaságilag

- Reuse miatt egy Business-build reálisan **~40–70 óra**, nem 150 → 500e–1M Ft-nál is megél.
- A recurring oldalanként pár ezer Ft valós költség → N ügyfélen **portfólió-LTV**.
- Az önszerkesztés **csökkenti a support-terhet** → a 8–15e/hó is profitábilis.

### 3.5 Realista minta-ügylet

**Business build ~700e Ft + Plusz üzemeltetés ~18e Ft/hó** → 1. év ~916e; utána ~216e/év
recurring/ügyfél. Az 5. ügyfélnél a build már ~450–600e (reuse), a recurring-portfólió
~1M Ft/év összesen, alacsony költséggel.

---

## 4. Versenytárs-elemzés (forrásolt, HU 2025–26)

### 4.1 Magyar piac

- **Build:** 200e–2M Ft; sablon **50–300e**, egyedi design+fejlesztés **500e–1,5M**, nagy
  ügynökség/komplex **több millió**. (newconcept.hu, klisestudio.hu, designpen.hu)
- **Üzemeltetés:** egyszerű oldal karbantartása **10–20e Ft/hó** (tágabban 10–100e); éves
  **60–240e Ft**. Létező **SaaS-hibrid:** csökkentett buildért **havi 9.500–50.000 Ft**
  üzemeltetés 1–2 évre. (qjob.hu, kiszervezettmarketing.hu)

### 4.2 Nemzetközi builderek (DIY, éves számlázás, ≈400 Ft/USD)

| Builder | Havi (USD) | ≈ HUF/hó | Ki épít/szerkeszt |
|---|---|---|---|
| Webflow | Basic $14 / CMS $23 / Business $39 | 5,6–15,6e | ügyfél DIY (vagy dev builderben) |
| Framer | Free / $10 / Pro $30 | 4–12e | ügyfél DIY |
| Squarespace | $16 / $23 / $39 / $99 | 6,4–40e | ügyfél DIY |

(Havi-havi számlázás ~30%-kal drágább.)

### 4.3 Összevető tábla

| Alternatíva | Build (HUF) | Havidíj (HUF/hó) | Ki épít / ki szerkeszt | Animáció |
|---|---|---|---|---|
| Sablon (HU) | 50–300e | 5–15e | ügyfél/sablon | nincs |
| **WP-ügynökség** (HU default) | 500e–1,5M | 10–20e | ügynökség épít, **ügyfél emailezik editért** | ritkán |
| DIY builder (Webflow/Framer…) | — (a te díjad) | 5,6–40e | **ügyfél DIY** | Framer igen, de DIY |
| **Mi (bespoke + CMS)** | **250–500e / 500e–1M / 1–2M** | **6–10e / 15–25e / 30–50e** | **ügynökség épít, ügyfél BIZTONSÁGOSAN önszerkeszt** | **igen (reflow-biztos alap, teljes GSAP prémium)** |

### 4.4 Mit igazol / pozicionálás

1. **A build sávunk a HU „egyedi" sávban ül** (500e–1M), nem fölötte → reálisan eladható. A
   korábbi 1,5–3,5M a felső/nagyügynökségi szegmens volt.
2. **A havidíjunk a HU karbantartási sávban / az alatt** (6–25e a 10–20e default körül) ÉS
   **a DIY-builderek előfizetésével egy szinten** (5,6–40e). Az ügyfél hasonló havidíjat fizet,
   mint egy Webflow-előfizetésért — **de bespoke-épített oldalt + supportot + önszerkesztést kap,
   nem egy DIY-toolt.**
3. **A rés a kettő közt:** a WP-ügynökség árán/az alatt, de **animáció + kliens-önszerkesztés**sel
   (a WP-ügynökség ezt nem adja — ott emailezni kell); és agency-minőséggel, amit a DIY-builder
   nem ad (ott az ügyfél épít). **Ugyanaz az ár, jobb csomag.**
4. **A „lower build → recurring" lever nem elmélet:** HU ügynökségek már csinálják (9,5–50e/hó,
   csökkentett buildért) → kínálhatsz „alacsony/nulla előleg + 1–2 éves elkötelezett üzemeltetés" variánst.

**Egy mondatban:** nem kell alávágni — **ugyanazon az áron jobb terméket** adsz (bespoke +
önszerkesztés), a számok forrásolva a HU-valóságban ülnek.

---

## 5. Jövőbeli irány (fázis 2): nyitott self-serve

**Ötlet:** az ügyfél maga regisztrál → választ egy template-et → egy **AI beimportálja a
tartalmat a saját (rosszabbul designolt) oldaláról** a template tipizált slotjaiba.

- **Technikailag illik:** ez a tech-spec §4.1 onboarding-importja — a *tartalmat* húzod át egy
  jó strukturába, a rossz dizájn eldobódik (mert content ⟂ struktúra).
- **Stratégiailag más piac:** ez a DIY/self-serve (Webflow/Wix/**Framer AI/Durable/Relume**) —
  zsúfolt, tőkeerős verseny; kis ARPU, volumen. Az él: bespoke-template-minőség + „importáld a
  meglévő oldaladról" + client-safe szerkesztés.
- **Kockázatok:** template-könyvtár + AI-import minősége folyamatos költség; a scrape/mappelés
  szokatlan oldalon gyenge (kell „nézd át és javítsd" lépés); a self-serve template-ek **kötelezően
  reflow-biztosak** (§14.5), mert az import tetszőleges hosszú.
- **Sorrend:** **Fázis 1 = ügynökség** (validál + legyártja a reflow-biztos template-könyvtárat és
  a megbízható AI-importot valós, fizetett munkán). **Fázis 2 = self-serve** (freemium: előnézet
  ingyen, publish/edit/domain fizetős; ~€10–30/hó). **Ne ezzel kezdj** — ez a platform-nyitás.

---

## 6. Amit üzletileg validálni kell (kód/árazás előtt)

1. **Keresleti kérdés (a legfontosabb):** hány valós HU ügyfél = *bespoke-animált oldalt akar +
   copy/kép-szintű editet + mi maradunk a struktúra gazdája* — ÉS egyik szomszédos megoldás
   (Shopify/Webflow/headless) sem szolgálja ki jobban? Beszélj **5–10 valós ügyféllel/leaddel.**
2. **A saját tényleges óraszámotok** egy Business-buildre (reuse-szal és anélkül) → ez adja a valós
   build-árat és a marzsot.
3. **A recurring elfogadottsága:** hajlandó-e a HU kkv **6–15e Ft/hó**-t fizetni önszerkesztés +
   hosting + support fejében? (A benchmark: a DIY-builderek ennyiért adnak DIY-toolt.)
4. **A build-discount-for-recurring variáns** (alacsony előleg + 1–2 év elkötelezettség) — jobb
   konverzió-e HU-ban, mint a magas előleg?

## Források

- HU build árak: [newconcept.hu](https://www.newconcept.hu/blog/weboldal-keszites) · [Klisé Stúdió 2026](https://klisestudio.hu/weboldal-keszites-arak-2026-ban/) · [designpen.hu](https://designpen.hu/webdesign-blog/honlapkeszites-ar-2025/) · [gotoonline.hu](https://gotoonline.hu/weboldal-keszites-arak-2025-ben-mennyibe-kerul-egy-profi-honlap-magyarorszagon/)
- HU karbantartás/üzemeltetés: [qjob.hu 2026](https://qjob.hu/blog/arak/weboldal-karbantartas-arak) · [kiszervezettmarketing.hu](https://kiszervezettmarketing.hu/weboldal-keszites/weboldal-karbantartas-arak/)
- Builderek: [Webflow pricing 2026](https://emergent.sh/learn/webflow-pricing) · [Framer 2026](https://aitoolpick.org/blog/framer-pricing-2026/) · [Squarespace 2026](https://aitoolpick.org/blog/squarespace-pricing-2026/)

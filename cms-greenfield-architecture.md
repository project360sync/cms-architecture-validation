# Greenfield architektúra — annotált template + tipizált tartalom

> **Cél:** egy client-safe CMS-réteg, ami egy fejlesztő-birtokolta, animált (GSAP)
> statikus oldalra teszi rá a szerkeszthetőséget úgy, hogy a **tartalom és a
> struktúra tisztán szétváljon**. Ez a dokumentum a **nulláról tervezett** (greenfield)
> koncepciót írja le — nem a jelenlegi implementációt. Célja, hogy egy másik AI
> vagy mérnök **validálni tudja az ötletet a végső implementáció előtt.**
>
> **Státusz:** javaslat, validálásra. **Nyelv:** magyar. **Dátum:** 2026-07.

---

## 0. TL;DR — a tézis egy bekezdésben

Két réteg, tisztán szétválasztva: **(1) STRUKTÚRA** = a fejlesztő annotált HTML/CSS
+ zárolt JS-bundle (GSAP), verziózva, a fejlesztő birtokolja. **(2) TARTALOM** = külön,
**nevesített, tipizált** adat (mezők + kollekciók), a kliens birtokolja. A **render
összefésüli** a kettőt; a JS a renderelt strukturán fut. A tartalom **névhez** kötött,
nem HTML-pozícióhoz → a struktúra átszabása nem törli a tartalmat. Ez lényegében a
**Shopify Online Store 2.0 sections-modellje**, de **tetszőleges annotált HTML-en**
(nem egy templating nyelven), és a **bespoke szerző-JS megőrzésével**.

---

## 1. Kontextus és mit tanultunk

- A kiinduló probléma: egy **animált, JS-vezérelt bespoke oldalt** (pl. GSAP-landing)
  szeretnénk **nem-technikai ügyfélnek** szerkeszthetővé tenni anélkül, hogy elrontaná.
- A **jelenlegi** (már részben megépített) modell: tetszőleges HTML **ingesztálása** →
  **pozíció-alapú** `data-cms-id="t7"` annotáció → **override-ok** a tárolt skeletonon.
  Ez működik, de **törékeny**: az id-k dokumentum-sorrendben készülnek, így egy forrás-
  átrendezés elcsúsztatja őket, és a kliens edit-jei elárvulnak/felülíródnak.
- **Prior art** (kódból/dokumentációból elemezve):
  - **Instatic** (Bun + saját renderelő): a HTML-importja **egyirányú konverzió** a saját
    strukturált node-fájába; import után a store az igazság; a `<script>`-et eldobja.
  - **Shopify OS 2.0**: Liquid theme + `{% schema %}` (tipizált settings + blocks) +
    **JSON template** (a merchant konfigja külön, id-kulcsolva) + theme editor. A merchant
    a settingeket szerkeszti, a kódot nem. Theme-frissítés **nem törli** a merchant
    konfigját (id-egyeztetés) — **globális skálán bizonyítja a név-kulcsolt robusztusságot.**
- **Konvergencia:** a robusztus, skálázható modell mindkét helyen ugyanaz —
  **tipizált tartalom + fejlesztő-birtokolta template/blokkok, a kettő szétválasztva.**

---

## 2. Célok és nem-célok

**Célok**
1. **Tartalom ⟂ struktúra** tiszta szétválasztás.
2. A **fejlesztő** birtokolja és kódban szerkeszti a struktúrát; a **kliens** csak a
   tartalmat, kizárólag a fejlesztő által engedélyezett határokon belül.
3. A **GSAP / szerző-animációk megmaradnak** a publikált oldalon.
4. **Per-modul szerkeszthetőség:** vannak zárolt (dev-only) és szerkeszthető modulok;
   a kliens **felvehet** bizonyos modulokat (paletta), másokat nem.
5. **Robusztusság a struktúra-frissítésre:** a fejlesztő átszabhatja a struktúrát a
   kliens tartalmának elvesztése nélkül.
6. **Framework-agnoszticizmus megőrzése:** ne kelljen a fejlesztőnek egy zárt rendszerben
   (pl. Liquid) építenie — tetszőleges statikus HTML-t annotál.

**Nem-célok**
- Nem akarunk **templating nyelvet** (az → theming platform, elveszik az agnoszticizmus).
- Nem akarunk **általános drag-drop page-buildert** (az az Instatic/Webflow niche).
- Nem célunk a **kliens-oldali struktúra-szerkesztés** a paletta-határokon túl.

---

## 3. A modell részletesen

### 3.1 Két réteg

| | STRUKTÚRA (template) | TARTALOM (data) |
|---|---|---|
| Ki birtokolja | fejlesztő (kód) | kliens (editor) |
| Mi | annotált HTML + CSS + JS-bundle | nevesített, tipizált mezők + kollekciók |
| Hol él | verziózott forrás / tárolt template | külön tartalom-store, névre kulcsolva |
| Változás | re-ingeszt (onboarding/frissítés) | editor / AI-chat |

### 3.2 Annotáció-konvenció (a struktúrán)

A fejlesztő normál oldalt ír, csak **megjelöli**, mi szerkeszthető. Alap: **minden zárolt**,
csak a jelölt részek nyílnak meg.

```html
<!-- Egy szekció-példány, típussal (a séma ehhez köt) -->
<section class="hero" data-cms-section="hero" data-gsap="hero-intro">
  <h1 class="hero__title">
    <span data-cms-field="line1">FÉM.</span>       <!-- SLOT: nevesített, tipizált -->
    <span data-cms-field="line2">ÜVEG.</span>
  </h1>
  <p data-cms-field="sub">Alumínium nyílászárókat gyártunk…</p>
  <svg class="hero__drawing">…</svg>               <!-- jelöletlen → ZÁROLT (GSAP rajzolja) -->
</section>

<!-- Kollekció: a kliens elemet vehet fel / szerkeszthet / rendezhet át -->
<section data-cms-section="products">
  <div class="pgrid" data-cms-collection="items">
    <template data-cms-item>                        <!-- a fejlesztő ITEM-SABLONJA -->
      <a class="pitem">
        <img data-cms-field="img">
        <span data-cms-field="title"></span>
        <span data-cms-field="desc"></span>
      </a>
    </template>
  </div>
</section>

<footer data-cms-editable="locked">…jogi szöveg…</footer>
```

Attribútumok:
- `data-cms-section="<type>"` — szekció-példány; a `<type>` egy **séma-kulcs** (3.4).
- `data-cms-field="<name>"` — nevesített tartalom-slot; a típusa a sémából jön (vagy
  elemből infered: `<img>` → image, egyébként text).
- `data-cms-collection="<name>"` — ismétlődő régió; benne egy `<template data-cms-item>`
  prototípus (a fejlesztő egy elemet ír meg); minden elem = egy tipizált **sor**.
- `data-cms-editable="locked|content|collection"` — per-blokk policy (opcionális; a jelölt
  mezők/kollekciók amúgy is definiálják a szerkeszthetőséget).

### 3.3 A tartalom-doc (Shopify-stílusú, névre kulcsolva)

Az oldal-tartalom egy **külön** dokumentum: szekció-példányok sorrendben, mindegyik
tipizált **settingekkel** és **blokkokkal**. Ez kódolja a tartalmat ÉS a kliens
(engedélyezett) strukturális döntéseit (mely szekciók, milyen sorrendben).

```json
{
  "version": 1,
  "template": {
    "order": ["hero-1", "products-1", "gallery-1"],
    "sections": {
      "hero-1":     { "type": "hero",
                      "settings": { "line1": "FÉM.", "line2": "ÜVEG.", "sub": "…" } },
      "products-1": { "type": "products",
                      "collection": "termekek" },
      "gallery-1":  { "type": "gallery",
                      "blocks": [ { "type": "gcard", "settings": { "img": "/a/…", "cap": "…" } } ] }
    }
  },
  "collections": {
    "termekek": [
      { "id": "itm_7f3a", "fields": { "title": "Berg Passive", "desc": "Uw=0,66", "img": "/a/abc" } },
      { "id": "itm_9c22", "fields": { "title": "AWS 75.SI+",  "desc": "75 mm",   "img": "/a/def" } }
    ]
  }
}
```

- **Bespoke oldal (vibor):** a `order` fix (a fejlesztő adja), a kliens csak a settingeket
  + a kollekciókat szerkeszti → a GSAP DOM-ja stabil marad.
- **Rugalmas oldal:** a kliens a **palettából** hozzáadhat/átrendezhet szekciókat is
  (`order` szerkeszthető) — a szekció-típusokat a fejlesztő definiálja.

### 3.4 Per-szekció séma (tipizált mezők — a Shopify `{% schema %}` analógja)

A fejlesztő minden szekció-típushoz megad egy **sémát**: mely mezők, milyen típussal,
címkével, alapértékkel, validációval. Ez adja a gazdag editor-UI-t + a validációt.

```jsonc
// sections/hero.schema.json
{
  "type": "hero",
  "label": "Hero",
  "fields": [
    { "id": "line1", "type": "text",     "label": "1. sor",  "max": 12 },
    { "id": "line2", "type": "text",     "label": "2. sor",  "max": 12 },
    { "id": "sub",   "type": "richtext", "label": "Alcím" }
  ],
  "editable": "content",          // content | locked | collection
  "insertable": false             // felveheti-e a kliens új példányként?
}
```

```jsonc
// sections/products.schema.json — kollekció-szekció
{
  "type": "products", "label": "Termékek", "editable": "collection",
  "item": {                        // az item-sablon mezői
    "fields": [
      { "id": "img",   "type": "image",    "label": "Kép" },
      { "id": "title", "type": "text",     "label": "Név" },
      { "id": "desc",  "type": "longtext", "label": "Leírás" }
    ]
  },
  "insertable": true               // a kliens vehet fel új terméket
}
```

**Mező-típusok** (kezdő halmaz): `text`, `longtext`, `richtext`, `image`, `url`, `select`,
`color`, `number`, `boolean`. Mindegyikhez validáció (max hossz, enum, kötelező).

---

## 4. Pipeline-ok

### 4.1 Onboarding-import (egyszer, HTML → template + tartalom)

Az ingeszt **onboarding-lépés, nem folyamatos overlay**. A tetszőleges HTML-ből előáll:
(a) az **annotált template** (a struktúra, JS-bundle SRI-pinnelve), és (b) a **kezdeti
tartalom-doc** (a slotokból kiolvasott értékek). Innentől a **tartalom-store az igazság**.

```text
importSite(html, assets, js):
  1. sanitize (inline <script>/on* ki; külső <script src> megtartva, SRI)
  2. a data-cms-* markerekből kiolvassa a slotokat/kollekciókat/szekciókat
  3. template  := struktúra (slotok üres/prototípus formában) + CSS + JS-bundle
  4. content   := a jelenlegi értékek kiolvasva a slotokból (fields + collections)
  → tárolás: template (verziózva) + content (névre kulcsolva)
```

### 4.2 Render / merge (a JS a strukturán fut)

```ts
function render(template, content): Html {
  const $ = load(template.html)
  for (const sec of content.template.order)          // szekciók sorrendben
    hydrate($, sec, content.template.sections[sec], content.collections)
  attachBundle($, template.jsBundle)                 // GSAP a renderelt DOM-on
  return $.html()
}
function hydrate($, secId, sec, collections) {
  const root = $(`[data-cms-section="${sec.type}"]`)     // (vagy szekció-példány-id)
  for (const [name, val] of Object.entries(sec.settings ?? {}))
    setContent(root.find(`[data-cms-field="${name}"]`), val)   // adat → slot
  if (sec.collection) {                                        // kollekció-render
    const box = root.find("[data-cms-collection]")
    const proto = box.find("[data-cms-item]")
    for (const row of collections[sec.collection] ?? [])
      box.append(fill(proto.clone(), row.fields))
  }
}
```

### 4.3 Publish / export (ADR-002)

A publish **immutable statikus snapshotot** renderel és az **edge-re** exportál (a kliens
saját domainje). A szerkesztő CMS **scale-to-zero**, nincs a látogató útján. A snapshot
tartalmazza a HTML-t + asseteket + a szerző-JS-bundle-t (SRI, CSP `script-src 'self'`).

### 4.4 Struktúra-frissítés (re-ingeszt, névre reconciliálva)

A fejlesztő átszabja a forrást (új szekció, redesign, GSAP-tweak) és re-ingesztál. Mivel
a tartalom **névre** kötött, nem pozícióra:

```text
reingest(newHtml):
  nextTemplate := importTemplate(newHtml)        // friss struktúra, ugyanazok a slot-nevek
  for (const [name, value] of content.fields)
    nextTemplate.hasSlot(name) ? keep(name, value)   // a slot még létezik → megtartjuk
                               : quarantine(name)      // eltűnt → jelöljük, nem dobjuk némán
  collections változatlan (a szekcióhoz névvel kötve)
```

**Nincs id-drift, nincs vak felülírás.** A slot átnevezése/törlése = kontrollált, jelzett
migráció (nem néma adatvesztés).

---

## 5. Editor-modell (client-safe)

- **Szerkeszthetőségi szintek** (per szekció/blokk, a sémából): `locked` (dev-only, nem is
  jelölhető ki), `content` (a mezői szerkeszthetők), `collection` (elem-felvétel/rendezés).
- **Hibrid szerkesztő:**
  - **Inline click-to-edit** a szöveg/kép slotokra (a valódi oldalon kattint).
  - **Settings-panel** a tipizált mezőkre, amiknek nincs inline reprezentációja
    (`select`, `color`, `url`, `boolean`) — Shopify-módra, élő previewvel.
- **Beszúrható paletta:** a `insertable: true` szekciók/blokkok, amiket a kliens felvehet;
  a többi (hero-animáció, SVG) a fejlesztőé.
- **AI-chat:** ugyanazon tipizált műveleteket adja ki (mező-írás, kollekció-elem hozzáadás),
  Guardian-validációval — a kliens sosem ad meg markupot.
- **Edit vs Preview mód:** Edit módban a szerző-JS **strippelve** (stabil szerkesztés),
  Preview módban **fut** (élő animáció). (Ez már megvan a jelenlegi implementációban.)

---

## 6. JS / animáció kezelése

- A szerző-JS (GSAP, ScrollTrigger, Lenis) **zárolt, verziózott bundle**, a **template**
  része (nem „megőrzött tetszőleges ingesztált blob"). Publikálva SRI-pinnelt, CSP `self`.
- A JS a **renderelt strukturán** fut. Mivel a tartalom csak érték a fix DOM-ban, az
  animáció változatlan marad a tartalom-szerkesztéskor.
- **Nyitott kockázat (validálandó):** ha a kliens **szekciót vesz fel / átrendez**, a GSAP
  (ami néha fix szelektorokra/pin-pozíciókra épít) **eltörhet**. Ezért bespoke oldalon a
  szekció-add/reorder alapból **tiltott** (csak mező + kollekció szerkesztés) — a GSAP DOM-ja
  stabil. Rugalmas oldalon a fejlesztőnek „dinamika-biztos" szekciókat kell írnia.

---

## 7. Tárolás

- A tartalom **tipizált, sémázott, verziózott** — ez relációs+JSON jellegű.
- **Opció A — Mongo** (jelenlegi): rugalmas JSON-docok; működik.
- **Opció B — Postgres + JSONB** (Instatic is SQL): séma-validáció + relációs integritás
  (kollekció → média, verziózás, audit) + JSONB rugalmasság. Tipizált tartalomra tisztább.
- **Ajánlás:** validálandó; a tipizált/verziózott/auditált tartalom **enyhén Postgres+JSONB
  felé húz**, de nem blokkoló — a modell store-agnosztikus.

---

## 8. Robusztusság / törékenység-elemzés

| Törékenységi tengely | Jelenlegi (pozíció-overlay) | Ez a modell (név+típus) |
|---|---|---|
| Forrás-átrendezés | id-drift → elárvult edit-ek | névre reconciliál → túléli |
| Tartalom vs struktúra | egy HTML-ben gabalyodva | tisztán szétválasztva |
| Dizájn-változás | re-ingeszt kockázatos | template-t szerkeszted, tartalom újrafolyik |
| addItem | DOM-klón + id-remint (bug-veszélyes) | egy tipizált sor hozzáfűzése (triviális) |
| Séma-változás | néma törés | verziózott, jelzett migráció |

**Kulcs-belátás:** a jelenlegi implementáció komplexitása és bugjai (leaf-item id-ütközés,
nested-kollekció col-id duplázás) **a pozíció-horgonyzásból** fakadnak. A név+típus modell
**egyszerűbb ÉS robusztusabb.**

---

## 9. Prior art & validáció

| Ez a modell | Shopify OS 2.0 | Instatic |
|---|---|---|
| annotált template | Liquid theme | saját renderelő |
| `data-cms-field` + séma | `{% schema %}` settings | node-mezők |
| kollekció / item-sablon | section `blocks` | node-fa gyerekek |
| tartalom-doc (name-keyed) | **JSON template** | data_rows |
| tipizált kollekció | **metaobjects/metafields** | data_tables/fields_json |
| locked/content/collection | theme editor + zárt Liquid | client role |
| beszúrható paletta | presets / app blocks | inserter |
| **különbség** | Liquid nyelv kell | benne építesz (builder) |
| **a mi rése** | annotált HTML (nyelv nélkül) + bespoke JS megőrzés | ingeszt-overlay + JS megőrzés |

**A Shopify a bizonyíték:** theme-verzió-frissítés a merchant konfigját nem törli
(id-egyeztetés) → a név-kulcsolt „struktúra-frissítés túléli a tartalmat" modell
**globális skálán bizonyítottan működik.**

---

## 10. Eltérés a jelenlegi (megépített) implementációtól + migráció

| Megépített (A) | Cél (C/D) |
|---|---|
| `data-cms-id="t7"` (pozíció) | `data-cms-field="hero.line1"` (név) |
| override-ok a skeletonon | külön tipizált tartalom-doc |
| ingeszt = folyamatos overlay | ingeszt = onboarding-import, utána a store az igazság |
| „megőrzött ingesztált JS" | fejlesztő-birtokolta verziózott bundle |
| globális safe-mode | per-szekció séma + `editable` policy |
| addItem = DOM-klón + id-remint | addItem = sor + újrarender |

**Hordozható a jelenlegiből:** a client-safe koncepció, a JS-hibrid (Edit/Preview + SRI),
az AI-szerkesztés, a kollekció-koncepció. **Cserélendő mag:** a horgonyzás + tárolás
(pozíció+overlay → név+típus). Migráció: az első ingeszt konvertálja a meglévő skeletont
+ override-okat névre kulcsolt tartalommá (egyszeri transzformáció).

---

## 11. Amit validálni kell (a review AI-nak szánt kérdések)

1. **Séma-forrás:** elég-e az inline annotáció + inferálás, vagy **kötelező** per-szekció
   séma-manifest (a fejlesztő írja)? Hol a határ a kényelem és a determinizmus közt?
2. **Reconciliáció:** slot átnevezés/törlés, szekció-törlés esetén a quarantine-stratégia
   elég-e? Kell-e explicit „rename map" a re-ingeszthez?
3. **Annotáció-only vs sections-as-data:** melyik az elsődleges? Együtt tudnak-e élni
   (fix bespoke oldal vs rugalmas kompozíció) egyetlen adatmodellben?
4. **GSAP × dinamikus szekciók:** mennyire reális a „bespoke = fix order" korlát?
   Van-e jobb módja, hogy a szerző-JS túléljen egy kliens-szekció-add-ot?
5. **Tárolás:** a tipizált+verziózott+auditált tartalom indokolja-e a Postgres+JSONB-t
   a Mongo helyett? Van-e migrációs vagy multi-tenant szempont, ami dönt?
6. **Editor-UX:** a hibrid (inline + settings-panel) modell nem lesz-e zavaró a kliensnek?
   Hol húzzuk meg az inline vs panel határt (mely mező-típusok mennek melyikbe)?
7. **Framework-agnoszticizmus ára:** ha nincs templating nyelv, mennyi logika (feltétel,
   ciklus a strukturában) esik ki, és az fáj-e valós oldalaknál?
8. **Média/asset:** a kollekció-elem képe hogyan kötődik (asset-id) és hogyan élnek túl
   a re-ingesztet? Van-e árva-asset / GC szempont?

---

## 12. Siker-kritérium

A modell akkor jó, ha egyszerre igaz:
- A fejlesztő **átszabhatja a struktúrát + a GSAP-ot**, re-ingesztál, és a **kliens
  tartalma túléli** (nincs id-drift, nincs néma vesztés).
- A kliens **csak azt szerkeszti, amit a fejlesztő engedett**, a bespoke dizájnt nem tudja
  szétverni; **felvehet** paletta-elemeket, zároltakat nem.
- A publikált oldalon a **GSAP fut**, a tartalom-szerkesztés nem töri az animációt.
- Az `addItem` és a mező-szerkesztés **egyszerű adatművelet**, nem DOM-sebészet.
- A modell **framework-agnosztikus** marad (annotált HTML, nem templating nyelv).

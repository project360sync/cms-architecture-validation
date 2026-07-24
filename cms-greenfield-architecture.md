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
    a settingeket szerkeszti, a kódot nem. A külön tárolt, stabil id-jű section-konfiguráció
    **erős prior art** a név-kulcsolt robusztusságra, de nem bizonyítja az annotált HTML
    importját, az automatikus migrációt vagy a bespoke JS életciklusát.
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
<!-- Egy szekció-példány: stabil példány-id + séma-típus -->
<section class="hero"
         data-cms-section-id="hero-1"
         data-cms-section-type="hero"
         data-gsap="hero-intro">
  <h1 class="hero__title">
    <span data-cms-field="line1">FÉM.</span>       <!-- SLOT: nevesített, tipizált -->
    <span data-cms-field="line2">ÜVEG.</span>
  </h1>
  <p data-cms-field="sub">Alumínium nyílászárókat gyártunk…</p>
  <svg class="hero__drawing">…</svg>               <!-- jelöletlen → ZÁROLT (GSAP rajzolja) -->
</section>

<!-- Kollekció: a kliens elemet vehet fel / szerkeszthet / rendezhet át -->
<section data-cms-section-id="products-1" data-cms-section-type="products">
  <div class="pgrid" data-cms-collection="items">
    <template data-cms-item>                        <!-- a fejlesztő ITEM-SABLONJA -->
      <a class="pitem">
        <img data-cms-field="img" width="800" height="600">
        <span data-cms-field="title"></span>
        <span data-cms-field="desc"></span>
      </a>
    </template>
  </div>
</section>

<footer data-cms-editable="locked">…jogi szöveg…</footer>
```

Attribútumok:
- `data-cms-section-id="<id>"` — stabil, oldalon belül egyedi **példányazonosító**; a
  tartalom-doc `template.sections` kulcsával egyezik. Re-ingesztnél nem generálható
  dokumentum-pozícióból.
- `data-cms-section-type="<type>"` — a példány **séma-kulcsa** (3.4). Ugyanabból a
  típusból több példány is lehet, ezért a típus önmagában nem használható render-célpontként.
- `data-cms-field="<name>"` — nevesített tartalom-slot; a típusa a sémából jön (vagy
  elemből infered: `<img>` → image, egyébként text).
- `data-cms-collection="<name>"` — ismétlődő régió; benne egy `<template data-cms-item>`
  prototípus (a fejlesztő egy elemet ír meg); minden elem = egy tipizált **sor**.
- `data-cms-editable="locked|content|collection"` — per-blokk policy (opcionális; a jelölt
  mezők/kollekciók amúgy is definiálják a szerkeszthetőséget).

Ez a forma egy **fix példányt** ír le. Ha a kliens új szekciópéldányt is felvehet, ahhoz
nem elég egy már DOM-ba helyezett példány: kell egy típusonkénti, inert
**section-prototípus registry** (például külön template-fájl vagy
`<template data-cms-section-type="gallery">`). A fix és a kompozíciós mód ugyanazt a
tartalom-docot használhatja, de eltérő render-szerződésük van; ezt a §4.2 rögzíti.

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
      { "id": "itm_7f3a", "fields": { "title": "Berg Passive", "desc": "Uw=0,66",
                                      "img": { "assetId": "ast_abc", "alt": "Berg Passive ablak" } } },
      { "id": "itm_9c22", "fields": { "title": "AWS 75.SI+",  "desc": "75 mm",
                                      "img": { "assetId": "ast_def", "alt": "AWS 75.SI+ ablak" } } }
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
      { "id": "img",   "type": "image",    "label": "Kép", "altRequired": true },
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
(a) az **annotált template** (a struktúra, content-addressed JS-bundle), és (b) a **kezdeti
tartalom-doc** (a slotokból kiolvasott értékek). Innentől a **tartalom-store az igazság**.

```text
importSite(html, assets, js):
  1. sanitize (inline <script>/on* ki; engedélyezett dependency-k vendorizálva és hash-elve)
  2. a data-cms-* markerekből kiolvassa a slotokat/kollekciókat/szekciókat
  3. template  := struktúra (slotok üres/prototípus formában) + CSS + JS-bundle
  4. content   := a jelenlegi értékek kiolvasva a slotokból (fields + collections)
  → tárolás: template (verziózva) + content (névre kulcsolva)
```

### 4.2 Render / merge (a JS a strukturán fut)

Két render-mód kell, és a template manifestje dönti el, melyik érvényes:

- **`fixed`**: a template már tartalmazza az összes stabil
  `data-cms-section-id` példányt. A content `order` értéke nem kliens-szerkeszthető;
  a renderer az id alapján pontosan egy meglévő gyökeret hidratál.
- **`composable`**: a template egy üres mount-pointot és típusonként egy inert
  prototípus-registryt tartalmaz. A renderer az `order` minden id-jéhez a `type`
  prototípusát klónozza, ráteszi a példány-id-t, hidratálja, majd sorrendben a
  mount-pointba fűzi.

```ts
function render(template, content): Html {
  validateReferences(template.manifest, content)
  const doc = load(template.shell)
  const mount = template.mode === "composable"
    ? exactlyOne(doc, "[data-cms-page-sections]")
    : null
  if (template.mode === "fixed")
    assertExactOrder(content.template.order, template.manifest.fixedSectionOrder)

  for (const secId of content.template.order) {
    const sec = required(content.template.sections, secId)
    const root = template.mode === "fixed"
      ? takeFixedInstance(doc, secId, sec.type)
      : instantiateSection(template.registry, secId, sec.type)
    hydrateTyped(root, sec, content.collections, template.schemas)
    if (template.mode === "composable") mount.append(root)
  }

  removeAllPrototypes(doc) // <template> sosem marad publikált tartalomként
  attachBundle(doc, template.jsBundle)
  return serialize(doc)
}
```

A kollekció-render ugyanezt a szabályt követi: a `<template data-cms-item>` **contentjét**
klónozza (nem magát a `<template>` elemet), minden sorhoz stabil item-id-t köt, validálja
a mezők egyediségét/sémáját, majd eltávolítja a prototípust. Hiányzó vagy duplikált
section-, block-, collection- vagy field-id esetén a publish hibával leáll.

A tipizált renderer **kontextus szerint kódol és validál**: a sima szöveg csak
`textContent`, a rich text egy sémázott dokumentum-AST-ből, mezőnként engedélyezett
elemekkel renderelődik, URL/image mezőnél protokoll- és origin-policy érvényesül,
attribútumérték sosem kerül nyers string-interpolációval a HTML-be. A puszta HTML
allowlist nem elég: egy animált szöveg-slotban még ártalmatlan blokkelemek is
megváltoztathatják a DOM-szerződést.
Ez a védelem minden editor-, AI-, import- és API-írásnál kötelező; az onboarding
`sanitize` önmagában nem védi a később módosított tartalmat.

### 4.3 Publish / export (ADR-002)

A publish **immutable statikus snapshotot** renderel és az **edge-re** exportál (a kliens
saját domainje). A szerkesztő CMS **scale-to-zero**, nincs a látogató útján. A snapshot
tartalmazza a HTML-t + content-addressed asseteket + a szerző-JS-bundle-t
(immutable deploy manifest). A CSP csak a tényleges deploy-bizalmi határt engedje:
user-média külön, nem futtatható originről jön; script kizárólag a generált, hash-elt
bundle namespace-ből. Egy önmagában álló `script-src 'self'` nem tartalmi integritás-policy.

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
migráció (nem néma adatvesztés). A név-egyeztetés csak az automatikus alapértelmezés:
produkciós séma-váltásnál verziózott, tesztelhető migrációs manifest kell
(`rename`, `transform`, `split`, `merge`, `delete→quarantine`). A re-ingeszt
tranzakciósan készít preview-verziót, és csak teljes schema/render/asset validáció után
aktiválható; rollbackhez az előző template + content verzió együtt marad meg.

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

- A szerző-JS (GSAP, ScrollTrigger, Lenis) **zárolt, verziózott, content-addressed
  bundle**, a **template** része (nem „megőrzött tetszőleges ingesztált blob").
  Publikálva saját originről, hash-elt fájlnévvel és szűk CSP `script-src 'self'` policyval.
  Az SRI csak külső erőforrásnál ad külön bizalmi határt; saját originű bundle
  verziórögzítésére a content hash + immutable deploy manifest az elsődleges.
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

**A Shopify erős prior art, nem teljes bizonyíték:** megmutatja, hogy a template-től
külön tárolt, stabil kulcsú merchant-konfiguráció nagy skálán működőképes. Nem bizonyítja
automatikusan az annotált tetszőleges HTML importját, a bespoke JS életciklusát vagy a
séma-migráció biztonságát; ezeket külön prototípussal kell validálni.

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

**Hordozható a jelenlegiből:** a client-safe koncepció, a JS-hibrid (Edit/Preview +
verziórögzítés),
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
9. **Bizalmi határok:** ki jogosult template/JS feltöltésre, hogyan izoláljuk az importot
   (SSRF, zip-slip, aktív SVG/HTML, dependency supply chain), és hogyan kódoljuk a
   későbbi editor-/AI-tartalmat minden render-kontextusban?
10. **Verziózás/publish:** mi történik párhuzamos szerkesztés, autosave, séma-migráció
    közbeni edit és részben sikertelen multi-locale publish esetén? Mi az atomi egység?

---

## 12. Korlátok — amire nem ad választ, és mikor válassz mást

> Ez a szekció szándékosan **a saját javaslat ellen** érvel. A validáció legfontosabb
> része nem az, hogy a modell belülről koherens-e (az), hanem hogy a **valós igény**
> tényleg ez-e — vagy egy szomszédos kategória (erős inkumbenssel) jobban kiszolgálja.

### 12.1 A sweet spot (hogy a határok érthetők legyenek)

**Bespoke, art-directed, jórészt statikus brand/marketing oldal** (ügynökség építi, GSAP/
animáció, egyedi dizájn), amit egy **nem-technikai ügyfél** tart karban **copy / kép /
ismétlődő-kártya** szinten, miközben a **fejlesztő marad a struktúra gazdája.** Ez valós,
alulszolgált rés — de **szűk.**

### 12.2 Amire a modell NEM ad választ (+ mit válassz helyette)

| Igény | Miért esik ki | Helyette |
|---|---|---|
| **Tranzakció / valós idejű adat** (bolt, kosár, készlet, foglalás, dinamikus ár, személyre szabás, belépett felhasználó) | statikus kimenet, nincs runtime/backend | **Shopify** / headless commerce / rendes dinamikus app |
| **Tartalom-gráf, sok újrahasznált entry** (blog 500 poszttal, tag/szerző/kapcsolódó; ugyanaz a „csapattag" több oldalon) | a modell **oldal-központú** (slot/oldal), nem tartalom-gráf | **headless CMS** (Sanity/Contentful/Storyblok) + query |
| **Adatból generált sok oldal** (500 termékoldal egy forrásból, auto-landingek, auto-sitemap) | oldalról oldalra megy, nincs template-generálás adatból | **Astro/Next** (getStaticPaths) + tartalom-forrás |
| **Fejlesztő NÉLKÜLI DIY-építés** (az ügyfél maga rakja össze, nincs dev) | **kell** a fejlesztő az annotált template-hez | **Webflow / Framer / Wix / Squarespace** |
| **Kliens dizájn-szabadság** (hero-átszabás, szekciók szabad mozgatása, új egyedi blokkok) | szándékosan **zárolt struktúra** | **Webflow/Framer** vagy Shopify theme-editor |
| **Enterprise lokalizációs workflow** (jóváhagyási lánc, ütemezés, A/B, fordító-szerepkörök) — az **i18n magját a §14 megoldja**, a nehéz ops-ot nem | a workflow-réteg hiányzik | **enterprise headless** (Contentful/Sanity/AEM) |
| **App-szintű interaktivitás** (foglaló naptár, konfigurátor, kalkulátor, embed-app) | ez nem tartalom-edit, hanem funkció | **app-blokkok / integrációk / harmadik-fél embed** |

### 12.3 A legélesebb BELSŐ korlát: GSAP × tartalom-reflow

A modell azt feltételezi, hogy „a tartalom csak érték egy fix DOM-ban". De a tartalom-
változás **átfolyat**: hosszabb szöveg, több elem, más képarány → a kézzel hangolt
**ScrollTrigger-pin / mért scroll-táv / SplitText fix sorszámra** **eltörhet vagy elcsúszik.**

Vagyis: **minél bespoke-abb az animáció, annál kevésbé biztonságos a szabad tartalom-edit**
— ami épp a value prop-ot ássa alá. Két kimenet:
1. a fejlesztő **„reflow-biztos" animációkat** ír (dizájn-korlát), vagy
2. az ügyfél csak **nagyon szűk mezőket** szerkeszthet (kevesebb érték a kliensnek).

Ez nem apró él: pont a niche középpontjában (art-directed motion) a legélesebb.
**→ Mérséklés: §14.5 (reflow-biztos animáció)** — hossz-toleráns szekciókat animálj,
keveset, tegyél `maxLength` hintet a hossz-érzékeny slotokra, és teszteld a támogatott
locale/viewport mátrixot. Ez csökkenti, de önmagában nem szünteti meg a kockázatot.

### 12.4 Mikor áll a FELHASZNÁLÓ érdekében mást választani

- **Van boltja / foglalás / tranzakció** → **Shopify** (megoldott platform; sose építsd újra).
- **Tartalom-nehéz, query-vezérelt, sok oldal** → **headless CMS + SSG**.
- **Maga akar építeni/dizájnolni, nincs fejlesztő** → **Webflow/Framer**.
- **Gyakori strukturális/dizájn-változás kell neki** → builder vagy Shopify-theme.
- **Több nyelv / enterprise ops** → enterprise headless.
- **Ritkán szerkeszt (évi pár edit)** → **ne építs CMS-t egyáltalán**: adj egy pici
  „szerkeszd ezt az 5 mezőt" űrlapot, vagy csináld meg helyette. A teljes CMS-réteg
  túlmérnökölés lehet.

### 12.5 A keresleti kockázat (a legfontosabb, nem-technikai validálandó)

A niche valós, de a veszély: sok ügyfél, aki elsőre ideillőnek tűnik, valójában **átcsúszik
egy szomszédos kategóriába, aminek erős inkumbense van** (bolt→Shopify, self-build→Webflow,
tartalom→headless, ritka-edit→semmi).

**Ezért a legfontosabb validálandó nem technikai, hanem KERESLETI:**
> Hány valódi ügyfél = *bespoke-animált oldal + copy/kép-szintű editet akar + a fejlesztő
> marad a hurokban* — ÉS egyik szomszédos megoldás sem szolgálja ki jobban?

Ha ez a szám kicsi, a felhasználó érdeke lehet, hogy **ne saját CMS-t építsen, hanem egy
meglévő platformot húzzon rá / bővítsen**, vagy a modellt szándékosan **egy szomszédos
kategória felé tolja** (pl. „annotált template + könnyű headless tartalom-gráf", hogy a
tartalom-nehéz eseteket is vigye). **Ezt a keresleti kérdést érdemes validálni, mielőtt
bármilyen kódot írunk.**

---

## 13. Siker-kritérium

A modell akkor jó, ha egyszerre igaz:
- A fejlesztő **átszabhatja a struktúrát + a GSAP-ot**, re-ingesztál, és a **kliens
  tartalma túléli** (nincs id-drift, nincs néma vesztés).
- A kliens **csak azt szerkeszti, amit a fejlesztő engedett**, a bespoke dizájnt nem tudja
  szétverni; **felvehet** paletta-elemeket, zároltakat nem.
- A publikált oldalon a **GSAP fut**, a tartalom-szerkesztés nem töri az animációt.
- Az `addItem` és a mező-szerkesztés **egyszerű adatművelet**, nem DOM-sebészet.
- A modell **framework-agnosztikus** marad (annotált HTML, nem templating nyelv).

---

## 14. i18n — többnyelvűség (bővítmény)

> A §12 az i18n-t hiányként jelölte. Ez a szekció megmutatja, hogy a modellbe
> **természetes bővítményként** illik: mivel a tartalom már szét van választva a
> struktúrától és **névre kulcsolt**, az i18n egy **locale-dimenzió a tartalom-rétegen**
> — a struktúra (template + GSAP) **egyetlen forrás marad.**

### 14.1 Tartalom-modell: locale-scoped, struktúra közös

A template a slotokat névvel deklarálja (locale-független). A tartalom locale-onként forkol:

```json
{
  "locales": ["hu", "en", "de"],
  "defaultLocale": "hu",
  "content": {
    "hu": { "fields": { "hero.line1": "FÉM." },   "template": { "order": ["hero-1","products-1"] } },
    "en": { "fields": { "hero.line1": "METAL." }, "template": { "order": ["hero-1","products-1"] } }
  },
  "collections": {
    "termekek": [
      { "id": "itm_7f3a",
        "fields": { "hu": { "title": "Berg Passive" }, "en": { "title": "Berg Passive" } } }
    ]
  }
}
```

A render **locale-onként egyszer** fut → `/hu/…`, `/en/…` statikus fák. **Egy struktúra, N kimenet.**

### 14.2 Fallback + staleness (a valódi munka)

Egy fordítás sosem „minden vagy semmi". Három állapot mezőnként: **translated / untranslated / stale.**

```ts
function resolve(field, locale, content) {
  const v = content[locale]?.fields[field]
  if (v?.value != null && !v.stale) return v.value      // lefordítva & friss
  if (content.fallbackPolicy === "preview")
    return content[content.defaultLocale].fields[field] // preview nem lesz üres
  throw new UnpublishableLocale(field, locale)
}
// Staleness: nemcsak a forrásérték, hanem a fordítási kontextus és a mezőséma
// verziójának kanonikus digestje is eltárolódik.
translation.stale =
  digest(sourceValueNow, contextNow, fieldSchemaVersion) !== translation.sourceDigest
```

Fallback **previewban** hasznos, de publikálásnál nem lehet univerzális alapértelmezés:
kevert nyelvű oldal SEO-, jogi és márkakockázat. A publish policy oldal/mező szinten
deklarálja, hogy `required`, `fallbackAllowed` vagy `hiddenWhenMissing`; a jogi, navigációs,
SEO- és konverziós mezők alapból `required`. Forrás-, kontextus- vagy sémaváltozáskor a
fordítás **stale**, és a policy dönti el, hogy blokkolja-e a locale publikálását.

### 14.3 Kollekciók locale-ok közt

Az elemek **stabil `id`-t** osztanak a locale-ok közt (a `7f3a` *ugyanaz* a termék),
és csak a **mezők** locale-onkéntiek → a fordítás elemenként/mezőnként követhető.
Locale-specifikus elem (csak DE-ben promó) = opcionális `locales: ["de"]` flag — a kivétel.

### 14.4 Struktúra per-locale

- **Alap: közös** — ugyanaz a template, fordított értékek. Nincs per-locale struktúra-munka.
- **RTL (ar/he)**: fejlesztői struktúra-ügy — a render `<html lang="ar" dir="rtl">`-t ad, a
  CSS **logikai property-ket** használ (`margin-inline`, `padding-inline-start`). Bespoke
  GSAP + RTL tükrözött animációt igényel (plusz dev-munka).
- **Per-locale szekció-sorrend/rejtés nem ingyenes általánosan.** `composable` módban a
  content `order` locale-onként eltérhet, de minden változat külön render- és bundle-tesztet
  kap. `fixed` módban a sorrend közös és változtathatatlan; csak előre deklarált opcionális
  szekció rejthető, ha annak DOM-/animáció-hatását a fejlesztő kezelte.

### 14.5 Reflow-biztos animáció — a §12.3 kockázat MÉRSÉKLÉSE (a kulcs)

A §12.3 él (fordítás → eltérő szöveghossz → törik a hangolt animáció) részben
**dizájn-döntés, részben futásidejű kockázat.** A fő mérséklés: **animálj
hossz-toleráns szekciókat, keveset.**

**Animáld (hossz-toleráns):**
- **Konténer-szintű reveal** (opacity/translate a *wrapperen* — a szöveg szabadon átfolyhat belül)
- Kép/háttér parallax, marquee, dekoratív SVG draw-in, fade-up belépő

**Kerüld (hossz-érzékeny):**
- **SplitText** sor-soronkénti reveal (sorszám-függő)
- **ScrollTrigger pin fix scroll-távval** (tartalom-magasság-függő)
- vízszintes **scroll-jack mért szélességgel**

**Technika:**
- animáld a **wrappert**, ne a szöveg-node-ot;
- `ScrollTrigger.refresh()` + `invalidateOnRefresh` tartalom-/locale-váltáskor; **futásidőben mérj,
  ne hardcode-olj** scroll-távot/szélességet;
- **hossz-érzékeny sloton `maxLength` hint a sémában** → a fordítót figyelmezteti (és az AI-t
  is korlátozza); `text-wrap: balance`, `min-height`, hely-fenntartás.

**Séma-jelölés:** egy szekció/mező kaphat `reflowSafe: true|false`-t. A `false` (hossz-érzékeny)
esetén: vagy a fejlesztő reflow-biztosra írja az animációt, vagy a szerkeszthető mezők
**hossz-korlátozottak** (a kliens/AI nem írhat túl hosszút). A `maxLength` azonban nem
geometriai garancia: ugyanannyi karakter eltérő fonttal, viewporton, írásrendszerben vagy
kézi sortöréssel máshogy tördelhet. Ezért publish előtt kötelező a támogatott
locale × viewport × font-load állapot vizuális/overflow tesztje; a `reflowSafe` fejlesztői
ígéret, nem automatikusan levezethető mezőtulajdonság.

**Elv:** **kevesebb, de robusztus animáció > sok törékeny.** A hero rajz-animáció + reveal-ek
+ marquee bőven elég a karakterhez; a pin scroll-jack a legkockázatosabb — azt i18n-nél hagyd
ki, vagy tedd hossz-toleránssá.

### 14.6 Routing & SEO

- **URL**: locale-prefix (`/hu/`, `/en/`) — statikus exporthoz a legegyszerűbb. (Aldomain/ccTLD = több infra.)
- A render kiadja: `<html lang>`, per-oldal **`hreflang` alternate**, per-locale `canonical`, per-locale **sitemap**.
- **Nyelvváltó** komponens: kell egy page-id↔fordítások map (ugyanaz a page-id a locale-ok közt).

### 14.7 AI-fordítás — erős funkció, ami már illik

A meglévő AI-chat + tipizált mezők: *„fordítsd az összes mezőt angolra"* / *„a termék-szekciót
németre"* → az AI a **cél-locale mezőértékeit** tölti (a struktúra érintetlen), review-ra jelölve.
A tipizált modell tisztává teszi (adatot fordít, sosem markupot) — ez inkább **headline feature**.

### 14.8 Tárolás

Egy **fordítás-tábla**: `(siteId, locale, fieldKey)` → `{ value, status, sourceDigest, updatedAt,
translatedBy }`. Ez first-class-szá teszi a mezőnkénti staleness/status + review-workflow-t.
Enyhén **Postgres+JSONB** felé húz (relációs status-query: „az összes stale angol mező").

### 14.9 Amit i18n-nél is validálni kell

1. **Enterprise workflow** (jóváhagyási lánc, ütemezés, fordító-szerepkörök) — az i18n *magját*
   ez a szekció megoldja, de a **nehéz lokalizációs ops** még mindig enterprise-headless felé húz. Hol a határ?
2. **Reflow-biztos animáció** mennyire korlátozza a bespoke dizájnt a gyakorlatban? Elfogadható-e a
   „kevesebb, hossz-toleráns animáció" megkötés az ügyfélnek/ügynökségnek?
3. **URL-stratégia** (prefix vs domain) a meglévő ügyfél-domainekkel — és a `hreflang`/sitemap-generálás.

---

## 15. Független architektúra-review — döntés és kapuk

**Döntés: feltételesen valid.** A két-rétegű, névvel és típussal kötött modell jó alap a
§12.1-ben leírt szűk célpiacra. Implementáció azonban csak az alábbi P0 kapuk
lezárása után induljon. A jelenlegi dokumentum termék-tézisként elég erős, de még nem
végrehajtható biztonsági és adatkonzisztencia-specifikáció.

### 15.1 P0 — implementáció előtti blokkolók

1. **Fix vs kompozíciós render-szerződés.** A már DOM-ban lévő fix példány hidratálása
   nem tudja megvalósítani a kliens általi section add/reorder funkciót. A két mód
   külön template-manifestet, validációt és algoritmust kapjon (§4.2); ne legyen
   „vagy példány-id" jellegű, futásidőben kitalált viselkedés.
2. **Kanonikus identitás és séma.** Egyetlen normatív content-formátum legyen. A §3
   `sections[id].settings` modellje és a §14 locale-szintű lapos `fields` példája jelenleg
   két eltérő alak. Válasszunk egyet, adjunk hozzá JSON Schema-t, és írjuk le:
   `site/page/locale/section-instance/block/field` identitását, egyediségét és
   referenciális integritását.
3. **Biztonságos render-szerződés.** Mezőtípusonként rögzíteni kell a sinket és policyt:
   text→`textContent`; richtext→sémázott AST; URL→engedett protokollok;
   image→asset-id, szerveroldali MIME/decode ellenőrzés; SVG→sanitize vagy rasterize.
   A template/JS import külön, megbízható fejlesztői jogosultság legyen, izolált
   fetch/unpack folyamattal (SSRF, redirect, méret, zip-slip és aktív tartalom védelem).
4. **Migrációs protokoll.** Stabil példány-id, explicit rename/transform manifest,
   dry-run diff, quarantine, preview, atomi aktiválás és rollback nélkül a
   „tartalom túléli a redesign-t" fő ígéret nincs bizonyítva.
5. **Draft/publish konzisztencia.** Optimistic concurrency (`revision`/ETag), audit log,
   draft és published revision, valamint site+locale snapshot szintű atomi publish kell.
   A statikus fájlokat staging prefixre kell írni, majd egy manifest/pointer váltással
   aktiválni; részleges export nem kerülhet élőbe.
6. **Futtatási izoláció és életciklus.** Az author bundle teljes jogú kód. Az editor
   preview ne a CMS saját originen fusson, hanem sandboxolt, külön originen. Definiálni
   kell a bundle hookjait (`mount`, `refresh`, `destroy`), mert az újrarender,
   locale-váltás és preview-navigáció különben listener/ScrollTrigger szivárgást okoz.
7. **Locale publish-policy.** A fallback preview-kényelem, nem általános publikációs
   garancia. A required/fallback/hidden policy, a stale kontextus és az atomi locale
   release nélkül a rendszer kevert nyelvű vagy részben régi oldalt publikálhat.

### 15.2 P1 — az első pilot előtt

- **Asset-életciklus:** immutable asset-id, variánsok, alt/crop/focal-point metadata,
  referencia-számlálás, soft delete és késleltetett GC.
- **Feltételek templating nyelv nélkül:** deklaratív `visibleWhen`, limitált listák és
  opcionális mezők szükségesek; különben a renderelőbe ad hoc templating nyelv nő.
- **Kapacitáskorlátok:** section/block/nesting/mezőméret limitek legyenek már a
  sémában; ezek editor-UX, renderidő és visszaélés elleni kontrollok is.
- **Validációs sorrend:** schema → keresztmező/keresztreferencia → render → HTML/a11y/SEO
  lint → asset resolve → locale/viewport overflow → bundle smoke test. Hiba esetén
  publish fail-closed.
- **Operáció:** tenant kvóták, export idempotencia, observability, backup/restore,
  retention és törlési politika.
- **Hozzáférés:** legalább developer/editor/publisher szerepek; AI ugyanazt a
  jogosultság- és validációs parancsréteget használja, mint a kézi editor.

### 15.3 Minimális bizonyító spike

Az architektúrát egy vertikális pilot döntse el, nem további általános tervezés:

1. két azonos típusú szekciópéldány + rendezhető kollekció;
2. HU/EN tartalom preview-fallbackkel, publish-policyval és stale-jelzéssel;
3. v1→v2 migráció mező-átnevezéssel, split-tel és törölt szekció quarantine-nal;
4. rosszindulatú richtext/URL/SVG tesztek;
5. párhuzamos edit + publish konfliktus és rollback;
6. GSAP `mount/refresh/destroy` lifecycle locale-váltás és hosszabb szöveg mellett;
7. fix és kompozíciós render ugyanabból a kanonikus content-sémából, két azonos típusú
   szekcióval és add/reorder művelettel.

**Go feltétel:** a pilot bizonyítja, hogy nincs tartalomvesztés, nincs aktív tartalom-
injektálás, a publish atomi/rollbackelhető, és a támogatott animáció hosszabb locale
mellett is determinisztikusan újrainicializálható. **No-go/pivot:** ha ehhez általános
template-runtime vagy tartalom-gráf épül, a §12 szerinti headless CMS + SSG irány
valószínűleg olcsóbb és a felhasználó érdekét jobban szolgálja.

### 15.4 A review állításainak ellenőrzési alapja

- A Shopify JSON template hivatalos sémája külön `sections` objektumot és az abban létező,
  egyedi id-kből álló `order` listát ír elő; a dinamikus section a típusdefinícióból
  renderelődik, nem egy már elhelyezett DOM-példány keresésével:
  [Shopify JSON templates](https://shopify.dev/docs/storefronts/themes/architecture/templates/json-templates).
- A Shopify maga is limiteket tesz section-, block-, nesting- és fájlméret szinten; ezek
  az összehasonlításból nem hagyhatók ki:
  [Shopify theme limits](https://shopify.dev/docs/storefronts/themes/architecture/limits).
- A CSP `'self'` helyalapú engedélyezés, míg az SRI a letöltött script/style pontos
  tartalmát ellenőrzi; egyik sem helyettesíti az eredet- és feltöltési bizalmi határt:
  [MDN CSP](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CSP),
  [MDN SRI](https://developer.mozilla.org/en-US/docs/Web/Security/Defenses/Subresource_Integrity).
- A GSAP hivatalos API-ja szerint a `refresh` újramér, az `invalidateOnRefresh` törli a
  cache-elt kezdőértékeket, a `gsap.context().revert()` pedig összegyűjtött animációk és
  ScrollTriggerek takarítására való. Ez alátámasztja az explicit lifecycle szükségességét,
  de nem garantál automatikus reflow-biztonságot:
  [ScrollTrigger](https://gsap.com/docs/v3/Plugins/ScrollTrigger/),
  [gsap.context](https://gsap.com/docs/v3/GSAP/gsap.context%28%29/).

### 15.5 Válaszok a §11 nyitott kérdéseire

| # | Döntés | Indok |
|---|---|---|
| 1 | **A manifest kötelező; az annotáció nem sémaforrás.** | Az annotáció a DOM-slotot köti a mezőhöz. A típus, validáció, lokalizálhatóság, render-sink és migráció csak explicit, verziózott manifestből determinisztikus. Az import az annotáció↔manifest eltérést hibának veszi. |
| 2 | **Explicit migration map kell.** | Az automatikus névegyezés csak változatlan mezőt vihet tovább. Rename/split/merge/type-change fejlesztői döntés; törlés quarantine. |
| 3 | **Egy content-séma, két explicit template-mód.** | `fixed` és `composable` együtt élhet, de a render-algoritmusuk és engedélyezett editor-parancsaik nem keverhetők (§4.2). |
| 4 | **Bespoke GSAP alapból `fixed`; composable csak lifecycle-kompatibilis sectionnel.** | Általános JS-t nem lehet statikusan „reorder-safe"-nek bizonyítani. Ezt capabilityként a manifestben a fejlesztő vállalja, majd teszt igazolja. |
| 5 | **Első implementáció: Postgres + object storage.** | A revision, optimistic lock, audit, jogosultság, publish pointer és referenciális integritás relációs. JSONB jó verziózott content-snapshotnak, de nem helyettesíti az alkalmazás-sémát. Bináris asset nem adatbázisba kerül. |
| 6 | **Inline a kiválasztás és gyors text/asset edit; panel a teljes, kanonikus űrlap.** | Így az inline élmény nem hoz létre második validációs útvonalat. Strukturált richtext, URL, select, color, boolean, SEO és hibajavítás panelben történik. |
| 7 | **Agnosztikus a build-outputnál, nem a forrás-frameworknél.** | A bemeneti szerződés renderelt statikus HTML/CSS + támogatott ES bundle + manifest. React/Vue runtime, szerverkomponens vagy tetszőleges app-kód importja már platform/runtime termék lenne. |
| 8 | **Média immutable asset-id-val kötődik.** | A mező locale-specifikus alt/caption/crop metaadatot, nem fájl-URL-t tárol. Soft delete + reference graph + retention után GC; re-ingeszt asset-id-t reconciliál. |
| 9 | **Template-import privilegizált build, content-edit nem kód.** | Külön jogosultság, izolált fetch/unpack/build, külön preview origin; minden content-írás ugyanazon tipizált command API-n megy át. |
| 10 | **Revision-alapú draft, atomi release pointer.** | Az autosave új draft revisiont ír ETag ellenőrzéssel. A publish egy validált site/locale snapshotot aktivál; nem fájlonként frissíti az élő oldalt. |

### 15.6 Kettős go/no-go kapu

A technikai spike sikere **szükséges, de nem elégséges**. Két egymástól független kapu van:

- **Architecture go:** a §15.3 tesztek bizonyítják az adatmegőrzést, izolációt, atomi
  publikálást és a vállalt animation lifecycle-t.
- **Product go:** valódi ügynökségi/ügyfél discoveryben több konkrét projekt mutatja, hogy
  az annotált meglévő build + szűk client-safe edit + bespoke JS együtt fizetett igény,
  és a csapat dokumentálni tudja, miért nem elég az adott projektre Webflow/Framer vagy
  headless CMS + SSG. Érdeklődés vagy általános „jó lenne szerkeszteni" nem elég;
  pilot-elköteleződés és valós template-ek kellenek.

Ha csak az egyik kapu teljesül, nem indul teljes platformfejlesztés: technikai kudarc
esetén pivot headless/SSG integrációra; keresleti kudarc esetén a spike marad belső
ügynökségi eszköz vagy leáll.

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
- `data-cms-editable="locked|content|collection"` — opcionális, durva authoring-hint.
  Nem ez a jogosultság forrása: a normatív, granularis permission/capability policy a
  verziózott section-manifestben él (§3.4).

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
  "permissions": {
    "editContent": true,
    "addItems": false,
    "reorderItems": false,
    "moveSection": false,
    "removeSection": false,
    "duplicateSection": false,
    "editStructure": false
  },
  "capabilities": {
    "reorderSafe": false,
    "removalSafe": false,
    "duplicationSafe": false,
    "reflowSafe": false
  },
  "allowNewInstances": false
}
```

```jsonc
// sections/products.schema.json — kollekció-szekció
{
  "type": "products", "label": "Termékek",
  "item": {                        // az item-sablon mezői
    "fields": [
      { "id": "img",   "type": "image",    "label": "Kép", "altRequired": true },
      { "id": "title", "type": "text",     "label": "Név" },
      { "id": "desc",  "type": "longtext", "label": "Leírás" }
    ]
  },
  "permissions": {
    "editContent": true,
    "addItems": true,
    "reorderItems": true,
    "moveSection": true,
    "removeSection": true,
    "duplicateSection": true,
    "editStructure": false
  },
  "capabilities": {
    "reorderSafe": true,
    "removalSafe": true,
    "duplicationSafe": true,
    "reflowSafe": true
  },
  "allowNewInstances": true
}
```

**Mező-típusok** (kezdő halmaz): `text`, `longtext`, `richtext`, `image`, `url`, `select`,
`color`, `number`, `boolean`. Mindegyikhez validáció (max hossz, enum, kötelező).

#### 3.4.1 Permission és capability nem ugyanaz

- A **permission** azt mondja meg, mit ajánl fel az editor az adott szerepkörnek.
- A **capability** fejlesztői állítás arról, hogy a section implementációja mely
  kompozíciós/reflow műveleteket viseli el; ezt automata és vizuális teszt igazolja.
- A permission sosem lehet tágabb a capabilitynél. A manifest fordítása hibával leáll,
  ha például `moveSection: true`, de `reorderSafe: false`.
- `fixed` template-módban a `moveSection`, `removeSection`, `duplicateSection` és
  `allowNewInstances` mindig `false`, a section saját deklarációjától függetlenül.
- A kliens/AI nem írhat capabilityt. Azt csak template-verziót publikáló developer
  változtathatja meg, új teszteredménnyel.

Ez három gyakorlati lockot eredményez, amelyek egymástól függetlenek:

- **content lock:** `editContent: false`;
- **position lock:** `moveSection/removeSection/duplicateSection: false`;
- **structure lock:** `editStructure: false` (ebben a CMS-ben ez az alapértelmezés).

Egy pozíció-lockolt hero tartalma tehát továbbra is szerkeszthető lehet. Egy lockolt
sectionön belüli kollekció elemei is lehetnek hozzáadhatók és rendezhetők, miközben maga
a section nem mozdítható.

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

- **Granularis szerkeszthetőség** (per section/blokk, a manifestből): külön content-,
  item- és kompozíciós permissionök. Az editor nem egyetlen `locked` flagből következtet.
  A tiltott műveletet nem ajánlja fel, és az API ugyanazt szerveroldalon is elutasítja.
  A UI megmutathatja a lock okát (például „a hero animáció miatt nem mozgatható").
- **Hibrid szerkesztő:**
  - **Inline click-to-edit** a szöveg/kép slotokra (a valódi oldalon kattint).
  - **Settings-panel** a tipizált mezőkre, amiknek nincs inline reprezentációja
    (`select`, `color`, `url`, `boolean`) — Shopify-módra, élő previewvel.
- **Beszúrható paletta:** a `allowNewInstances: true` sectionök/blokkok, amiket a kliens
  felvehet; a többi (hero-animáció, SVG) a fejlesztőé. Ez csak `composable` módban
  érvényes, és nem írja felül a szerepkör szerinti jogosultságot.
- **AI-chat:** ugyanazon tipizált műveleteket adja ki (mező-írás, kollekció-elem hozzáadás),
  Guardian-validációval — a kliens sosem ad meg markupot.
- **Edit vs Preview mód (izolálva):** mindkettő **külön preview-originről szolgált,
  sandboxolt iframe**; a template sosem kerül a CMS-kezelőfelület DOM-jába.
  - **Edit:** szerző-`<script>` strippelve; csak a first-party, megbízható edit-runtime fut;
    a szerző iframe/form/külső-asset a secure-render (P0.3) szerint semlegesítve; sandbox-flagek
    blokkolják a top-navigációt és a külső form-submitet.
  - **Preview:** read-only, szerző-JS-sel (élő animáció).
  - Indoklás és a v1 concurrency-döntés: §15.7.

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
- A kompozíciós permission csak deklarált és tesztelt capabilityből következhet:
  `moveSection→reorderSafe`, `removeSection→removalSafe`,
  `duplicateSection→duplicationSafe`. Átrendezéskor a runtime sorrendje:
  `destroy → DOM-művelet → mount → asset/font ready → refresh`. Globális indexre,
  testvér-sorrendre vagy fix scroll-távra épülő section alapból pozíció-lockolt.

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

### 15.1 P0 — capability- és release-gate-ek

A P0 itt nem azt jelenti, hogy mind a hét pontot a legelső spike előtt meg kell építeni.
Az adott release-ben vállalt capabilityre vonatkozik: `fixed-only` release-nek nem kell
`composable` runtime, single-locale release-nek nem kell locale-policy. Az alkalmazandó
P0 azonban kötelező az adott capability élesítése előtt; a fázisbontást a §15.3 rögzíti.

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
5. **Draft/publish konzisztencia.** Minden release-hez immutable draft/published revision
   és site+locale snapshot szintű atomi publish kell. A v1 kizárólagos, pesszimista
   edit-lockot használhat; `revision`/ETag alapú optimistic concurrency akkor kapu, ha
   ugyanazon oldalon párhuzamos írást, offline editet vagy kollaborációt vállalunk (§15.7).
   A statikus fájlokat staging prefixre kell írni, majd manifest/pointer váltással
   aktiválni; részleges export nem kerülhet élőbe. Audit/RBAC a production-pilot kapuja.
6. **Futtatási izoláció és életciklus.** Az author bundle teljes jogú kód. Az editor
   Edit és Preview nézete se a CMS admin originjén/DOM-jában fusson, hanem sandboxolt,
   külön preview-originen (§15.7). Definiálni kell a bundle hookjait (`mount`, `refresh`,
   `destroy`), mert az újrarender, locale-váltás és preview-navigáció különben
   listener/ScrollTrigger szivárgást okoz.
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

### 15.3 Fázisbontás — mi kerül a spike-ba, mi utána

A review-egyeztetés alapján (lásd a PR-diszkussziót) a P0/P1 tételek **három fázisra**
oszlanak. A mag-hipotézist a **v1 minimál spike** olcsón bizonyítja; a nehéz platform-részek
később jönnek. Minden P0 az érintett capability release-gate-je, nem mind a spike
előfeltétele és nem mind alkalmazandó egy `fixed-only`, single-locale release-re.

**(A) v1 minimál spike — `fixed-only`, single-editor, single-locale**
Cél: bizonyítani, hogy a tartalom túléli a redesignt, és a client-safe szerkesztés + a
bespoke GSAP együtt működik. Ehhez **concurrency, multi-locale és composable NEM kell.**

1. stabil section-/field-identity + kötelező manifest (P0.1 fixed fele, P0.2);
2. typed, kontextus-biztos render **minden** content-write után + rosszindulatú
   richtext/URL/SVG tesztek (P0.3);
3. minimális `rename`/`quarantine` migráció + immutable revision + rollback → a
   „tartalom túléli a redesignt" bizonyítása (P0.4);
4. fixed GSAP `mount/refresh/destroy` lifecycle hosszabb szöveg mellett (P0.6 fixed fele);
5. content-edit + item-hozzáadás/rendezés a section **pozíciójának módosítása nélkül**;
6. **atomi publish** (staging → pointer) + rollback;
7. vegyes lock-policy `fixed` módban: pozíció-lockolt de tartalmilag szerkeszthető hero,
   lockolt sectionön belül rendezhető itemek, tiltott API-parancsok szerveroldali elutasítása.

**Go feltétel:** nincs tartalomvesztés, nincs aktív tartalom-injektálás, a publish
atomi/rollbackelhető, és a támogatott animáció hosszabb szöveg mellett is determinisztikusan
újrainicializálható.

**(B) Production pilot előtt**
- **Concurrency:** v1-ben **pesszimista edit-lock** (§15.7); az **optimistic concurrency +
  lost-update** kezelés ide kerül, ha valós egyidejű több-szerkesztős használat lesz (P0.5).
- teljes **multi-locale publish-policy** (required/fallback/hidden + atomi locale-release, P0.7);
- **audit / RBAC** + atomi release pointer (P0.5 többi része);
- végleges **asset-életciklus** + operációs kontrollok (P1).

**(C) Fázis 2 — composable**
- `composable` prototípus-registry + section add/remove/reorder (P0.1 composable fele);
- `reorderSafe` / `removalSafe` / `duplicationSafe` capabilityk + a hozzájuk tartozó tesztek;
- per-locale szekció-sorrend (§14.4).

**No-go/pivot:** ha a mag-tézishez általános template-runtime vagy tartalom-gráf kell,
a §12 szerinti headless CMS + SSG irány valószínűleg olcsóbb és a felhasználó érdekét
jobban szolgálja.

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
| 5 | **Production baseline: Postgres + object storage; a spike maradhat Mongón.** | A revision, audit, jogosultság, publish pointer és referenciális integritás relációs. JSONB jó verziózott content-snapshotnak, de nem helyettesíti az alkalmazás-sémát. Bináris asset nem adatbázisba kerül. A persistence-migráció nem része a mag-tézist vizsgáló spike-nak (§15.7). |
| 6 | **Inline a kiválasztás és gyors text/asset edit; panel a teljes, kanonikus űrlap.** | Így az inline élmény nem hoz létre második validációs útvonalat. Strukturált richtext, URL, select, color, boolean, SEO és hibajavítás panelben történik. |
| 7 | **Agnosztikus a build-outputnál, nem a forrás-frameworknél.** | A bemeneti szerződés renderelt statikus HTML/CSS + támogatott ES bundle + manifest. React/Vue runtime, szerverkomponens vagy tetszőleges app-kód importja már platform/runtime termék lenne. |
| 8 | **Média immutable asset-id-val kötődik.** | A mező locale-specifikus alt/caption/crop metaadatot, nem fájl-URL-t tárol. Soft delete + reference graph + retention után GC; re-ingeszt asset-id-t reconciliál. |
| 9 | **Template-import privilegizált build, content-edit nem kód.** | Külön jogosultság, izolált fetch/unpack/build, külön preview origin; minden content-írás ugyanazon tipizált command API-n megy át. |
| 10 | **Revision-alapú draft, atomi release pointer.** | V1-ben az autosave kizárólagos edit-lock alatt ír immutable draft revisiont; kollaboratív módban ETag/lost-update kontroll kell. A publish mindkét esetben validált site/locale snapshotot aktivál, nem fájlonként frissíti az élő oldalt. |

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

### 15.7 Feloldott v1-döntések (a review-egyeztetésből)

**Concurrency — v1: pesszimista edit-lock (nem optimistic).**

- A **spike single-editor** → semmi concurrency nem kell hozzá.
- **Early v1 / produkció:** oldal-szintű **pesszimista edit-lock**:
  - **heartbeat** (~30 mp) + **TTL** (~2–5 perc lejárat, hogy a bezárt fül / lefagyás /
    megszakadt net **ne** zároljon örökre) + **admin force-unlock** vészhelyzetre;
  - **re-entrant a saját usernek**: második fül, és **az AI a user nevében** ír → nem zárja
    ki magát (a kézi editor és az AI ugyanazon a tipizált command-úton);
  - a többiek **csak-olvasható** nézet + „**X épp szerkeszti — N perce**" jelzés.
- **Atomi publish** (staging → pointer) **külön** marad — a lock editor↔editor, nem
  edit↔publish; migráció/re-ingeszt közben az edit blokkolva/sorolva („épp publikálunk…").
- **Optimistic concurrency + lost-update** a *production pilot előtt* fázisba (§15.3/B) kerül —
  akkor, ha valós egyidejű, ugyanazon oldalon dolgozó **több szerkesztő** lesz (különböző
  részeken), offline/mobil vagy real-time collab. A célpiacon (tipikusan egy ügyfél-editor) a
  pesszimista lock elég és **jóval egyszerűbb** (nincs revision/ETag/merge-UI).

**Izoláció — Edit ÉS Preview is sandboxolt, külön originről.**

Edit módban is **fejlesztői HTML/CSS/SVG** jelenik meg (iframe, form, navigáció, külső
asset), ami **JS nélkül is kockázat** → a template **sosem** kerül a CMS-kezelőfelület
DOM-jába:

- **Edit:** külön preview-originről, sandboxolt iframe; szerző-`<script>` strippelve; csak a
  **first-party edit-runtime** fut; a szerző iframe/form/külső-asset a P0.3 secure-render
  szerint semlegesítve; sandbox-flagek blokkolják a top-navigációt és a külső form-submitet.
- **Preview:** külön originről, **read-only** iframe, szerző-JS-sel.

Ez pontosítja a P0.6-ot: nem elég csak a JS-es Previewt izolálni, az Edit-nézetnek is külön,
sandboxolt originen kell futnia.

**Scope-megerősítés:** v1 = `fixed-only`; `composable` = Fázis 2. **Persistence:** a spike
maradhat a jelenlegi **Mongón**; a Postgres+JSONB greenfield-döntést (§15.5 #5) nem kell a
mag-tézis validációjához előre megfizetni — a revision/audit/RBAC/publish-pointer relációs
igénye a production pilotnál válik esedékessé.

---

## 16. Review-kör 2 — feloldott blokkolók (adverzariális review)

> Egy második, **adverzariális** review-kör (négy független ügynök: red-team, architektúra,
> security, termék/kereslet) néhány **tartógerendára** konvergált. A koncepció kiállta a
> támadást — a §15 P0-kapuk állnak —, de három KRITIKUS blokkoló doc-szintű feloldást igényelt,
> mielőtt a fixed-only spike adat/render-rétege lekódolható. Ez a szekció ezeket zárja le
> (kanonikus séma, reconciliation-teljesség, sink-lista), finomítja a v1 concurrency néma-vesztés
> rését, feloldja a store-szóhasználatot, és **megfordítja a közeli szekvenálást** (kereslet-kapu
> a technikai spike előtt).

### 16.1 Kanonikus tartalom-séma (P0.2 feloldva)

**Döntés: egyetlen normatív alak.** A §3.3, §14.1 és a §4.2/§4.4 pszeudokód eddig három eltérő
alakot mutatott. Ezek közül a §14.1 lapos `hero.line1` kulcs **hibás**, mert a *típusra* kulcsol,
holott egy típusból több példány is lehet (§3.2), így két `hero` ütközne. Az alábbi az egyetlen
érvényes alak; a korábbi példák illusztratívak, és **ez felülírja őket.**

Identity-szintek (mind explicit, egyik sem pozícióból származtatott):

| Szint | Kulcs | Egyediség | Példa |
|---|---|---|---|
| site | `siteId` | globális | `site_v1b0r` |
| page | `pageId` | site-on belül | `home` |
| locale | `locale` | ISO kód | `hu` |
| section-instance | `sectionInstanceId` | page-en belül | `hero-1` |
| block | `blockId` | section-instance-en belül | `blk_9f2` |
| collection | `collectionName` | site-on belül | `termekek` |
| collection-item | `itemId` | collection-on belül | `itm_7f3a` |
| field | `(scopeId, fieldName)` | scope-on belül | `(hero-1, line1)` |

**Kulcs-invariánsok:**
- **A mező sosem a típusra kulcsol.** A mezőidentitás mindig `(scopeId, fieldName)`, ahol a scope
  egy section-instance, block vagy collection-item. A DOM-annotáció a **csupasz** nevet hordozza
  (`data-cms-field="line1"`), a renderer/reconciler a befoglaló `data-cms-section-id` /
  `data-cms-item` / block-scope alapján kvalifikálja. Ez feloldja a §3.2 (csupasz) ⟂ §10
  (`hero.line1`) annotáció-ellentmondást is: a helyes annotáció a **csupasz** név.
- **A blokknak stabil `id`-ja van** (`{ "id":"blk_9f2", "type":"gcard", "settings":{…} }`), nem
  tömb-index — enélkül pozíció-horgonyzott lenne, épp az a törékenység, ami ellen a modell érvel (§8).
- **A struktúra (order / type / collection-referencia) locale-független; csak a mezőértékek
  locale-scoped-ek.** Kivétel a `composable` per-locale order (§14.4): külön, opcionális felülírás —
  nem az alap.

**Kanonikus váz** (egy oldal, két locale, egy kollekció):

```jsonc
{
  "schemaVersion": 2,
  "siteId": "site_v1b0r",
  "defaultLocale": "hu",
  "locales": ["hu", "en"],
  "pages": {
    "home": {
      "template": {                          // STRUKTÚRA — locale-független
        "mode": "fixed",
        "order": ["hero-1", "products-1"],
        "sections": {
          "hero-1":     { "type": "hero" },
          "products-1": { "type": "products", "collection": "termekek" }
        }
      },
      "content": {                           // ÉRTÉKEK — locale-scoped, scope=példány-id
        "hu": { "fields": { "hero-1": { "line1": "FÉM.",  "line2": "ÜVEG."  } } },
        "en": { "fields": { "hero-1": { "line1": "METAL.","line2": "GLASS." } } }
      }
    }
  },
  "collections": {
    "termekek": [
      { "id": "itm_7f3a",
        "fields": { "hu": { "title": "Berg Passive" }, "en": { "title": "Berg Passive" } },
        "assets": { "img": { "assetId": "ast_abc", "alt": { "hu": "…", "en": "…" } } } }
    ]
  }
}
```

**JSON Schema a spike előtt.** Egy `content.schema.json` rögzíti a fenti alakot: required kulcsok,
id-formátumok, és a **referenciális integritás** (minden `order`-id létezik a `sections`-ben; minden
`collection`-referencia létező kollekcióra mutat; minden `assetId` létező assetre). A render /
reconcile / i18n mind ezt az **egy** alakot fogyasztja: a §4.2 `render(template, content, locale)`
szignatúrát kap, és a hidratálás a §14.2 `resolve(field, locale)`-on megy át (fallback / required /
hidden policy a hidratáláson **belül**, locale-publikálást bukathat).

### 16.2 Reconciliation & migráció — teljes protokoll (P0.4 kiegészítés)

A §4.4 auto-name-match csak *törlésre* védett (quarantine). Az adverzariális kör három nem kezelt
esetet talált, amelyek cáfolják a „nincs vak felülírás" ígéretet. Feloldás:

**Precedencia (definiálva):** az explicit migrációs manifest **mindig nyer**; az automatikus
név-egyezés csak a *maradékot* viszi tovább — azokat a `(scopeId, fieldName)` párokat, amelyek neve
ÉS típusa változatlan, **ugyanabban a scope-ban.** A manifest a `(from-templateVersion →
to-templateVersion)` párra van kulcsolva.

**Név-csere (`line1`↔`line2`) — némán korrupt volt, most tiltott.** Mivel a match scope+név szerint
történik, a csere két egyidejű átnevezésként detektálható. Ha egy scope-on belül a mező-névhalmaz
megváltozik (nem tiszta hozzáadás/törlés), az auto-match **nem cross-assignol**: vagy explicit
`rename`/`move` bejegyzés kell, vagy a nem egyértelmű mezők quarantine-ba mennek. Néma
érték-áthelyezés nem történhet.

**Slot mozgatása szekciók között — most explicit `move`.** `move: hero-1.tagline → intro-1.tagline`.
Enélkül az érték elárvul (quarantine), nem tűnik el némán.

**Kollekció item-séma változás — nem fail-close-olja az egész oldalt.** Új mező default `optional`.
Egy meglévő mező `required`-dé tétele (pl. `altRequired:true`) **migráció**, ami vagy default-értéket
ad, vagy az érintett sorokat **draft-quarantine**-ba teszi és az adott locale/section
publikálhatóságát blokkolja egy világos diffel — **nem** a teljes site-publisht bukatja némán.

**Migrációs operátorok (specifikálva):**

| Operátor | Szemantika | Rollback |
|---|---|---|
| `rename` | `scope.a → scope.b`, érték változatlan | inverz rename |
| `move` | `scopeA.f → scopeB.f` | inverz move |
| `transform` | tiszta, verziózott függvény az értéken (nincs I/O) | ha nem invertálható → forrás quarantine-ban marad |
| `split` | `f → [f1, f2]` explicit mappinggel | merge inverz |
| `merge` | `[f1, f2] → f` explicit kombinátorral | split inverz |
| `delete` | `f → quarantine` (nem hard-törlés) | restore quarantine-ból |

**Hatókör:** az operátorok **section-instance, block, collection-item ÉS item-mező** szinten is
működnek (nem csak page-mezőn). A `transform` determinisztikus és mellékhatás-mentes, hogy a dry-run
és a rollback reprodukálható legyen.

**Kötelező dry-run diff** (a §4.4-ből P0.4-be emelve): a re-ingeszt előbb egy diffet ad
(`kept / renamed / moved / transformed / quarantined / new-empty`), ami emberi jóváhagyás nélkül nem
aktiválható; az aktiválás atomi (előző template+content verzió rollbackhez megmarad).

### 16.3 Biztonságos render-szerződés — teljes sink-lista + import-izoláció (P0.3 kiegészítés)

**Mezőtípus → sink (teljes):**

| Típus | Sink | Policy |
|---|---|---|
| `text` / `longtext` | `textContent` | nincs markup |
| `richtext` | sémázott AST → allowlisted elemek | **a node-attribútumok is szűrve: `href` a `url`-policyn át; `style`/`class`/`on*` eldobva** |
| `url` | attribútum, kódolva | csak `https:`/`mailto:`/relatív; `javascript:`/`data:` tiltott; kontroll/whitespace normalizálva (`java\tscript:`); protokoll-relatív `//host` tiltott; külső `target=_blank` → `rel="noopener noreferrer"` |
| `image` | asset-id → szerveroldali MIME/decode | SVG-média **rasterize-by-default** |
| `color` | **CSS-token validáció** | csak szigorú szín-token; nyers érték sosem kerül `style`/custom-propertybe |
| `select`/`number`/`boolean` | enum/típus-validált token | CSS-kontextusba csak allowlisted token |
| `svg` (ha nem `image`) | sanitize **nevesített, tesztelt** konfiggal | `script`/`foreignObject`/`use[href]`/`<style>` strip; egyébként rasterize |

- **CSS-as-content most explicit sink.** A §4.2 „attribútumérték sosem nyers interpolációval"
  állítása eddig nem fedte a `style`/custom-property kontextust: `color`/`select`/`number` oda
  folyhatott. Mostantól bármely CSS-be kerülő érték allowlisted token, nyers string sosem.
- **`svg` mezőtípus-inkonzisztencia feloldva** (P0.3 vs §3.4): vagy valódi `svg` típus a fenti
  sanitize-konfiggal, vagy az `image` úton szerveroldali rasterizálás; a „lebegő" SVG-hivatkozás
  megszűnik.

**Import-izoláció (P0.3 / §4.1 — a fenyegetéslistából kontrollok):**
- **Fetch:** minden import-fetch **egress-allowlist proxyn** át; link-local és RFC1918
  (`169.254.0.0/16`, `10/8`, `172.16/12`, `192.168/16`, `::1`, …) tiltva (SSRF / cloud-metadata);
  a redirect-célt **újra** validálni (DNS-rebind-biztos).
- **Unpack:** entry-count, kicsomagolt-méret, kompressziós ráta és path-traversal (`zip-slip`) limitek.
- **Build:** hálózat-izolált sandboxban, **ambient credential nélkül**; a dependency-k **lockfile +
  provenance/signature** szerint rögzítve (nem csak „hash-elve"); install-scriptek tiltva.

**Published-CSP (a §4.3 csonk kiegészítve):** `default-src 'none'`; `script-src` **csak** a hash-elt
bundle-namespace; **`connect-src` allowlist** (enélkül egy kompromittált bundle korlátlanul
exfiltrál — a legfontosabb futásidejű fék); `style-src`/`img-src`/`font-src` szűkítve;
`object-src 'none'`; `base-uri 'none'`; `form-action` allowlist; `frame-ancestors` szűkítve. Plusz a
media-originen **`X-Content-Type-Options: nosniff` + kényszerített `Content-Type`** — enélkül a
„nem-futtatható media-origin" védelem összeomlik (sniffelt script).

**postMessage-csatorna** (Edit/Preview iframe ↔ CMS-host, a `cms:*` üzenetek): a host-listener
**szigorú `event.origin` allowlist**-et ellenőriz; a host→iframe post **célzott originre** megy
(sosem `*`); a parancsoknak **validált sémája** van. A `sandbox` token-halmaz explicit — Edit:
`allow-scripts` a first-party edit-runtime-hoz, `allow-same-origin` **nélkül** a külön originen,
top-nav és külső form-submit tiltva; Preview: read-only. Az izolációs origin **külön regisztrálható
domain** (nem csak aldomain — az eTLD+1 megosztása gyengébb izoláció).

### 16.4 v1 concurrency — revision-guard a néma lost-update ellen (P0.5 részleges előrehozás)

A §15.7 pesszimista edit-lock re-entráns a saját usernek (hogy „az AI a user nevében" ne zárja ki
magát), de v1-ben nincs revision → ember-A-fül + AI-B-fül (§14.7 „fordíts minden mezőt")
felülírhatják egymást, ami cáfolja a §13 „nincs néma vesztés"-t. Feloldás a *teljes* optimistic
concurrency nélkül:

- **Könnyű revision-guard már v1-ben:** minden írás hordozza a bázis-revíziót; a stale írás
  (force-unlock, TTL-lejárat, vagy ugyanazon user két írási útja) **elutasítva**, nem
  last-write-wins. Ez nem a teljes merge-UI (az marad a B fázisban), csak a néma vesztés lezárása.
- **AI-írások:** minden AI-írás (nem csak a fordítás) **review-stagingbe** megy; bulk-műveletnél
  megerősítés; token/ráta/rekurzió limit a read→write→re-read hurokra; az audit-log
  **megkülönbözteti** az AI- és ember-írást a közös re-entráns lock alatt is.
- **Admin force-unlock:** **fencing-token**, hogy a lejárt lock késői írása ne érvényesüljön.

### 16.5 Store — a §15.5 #5 ⟂ §15.7 szóhasználat feloldva

- **„Első *produkciós* implementáció" = Postgres + object storage** (§15.5 #5). A **spike** Mongón
  fut (§15.7) — de ez explicit egy **logika-prototípus**, aminek a konzisztencia-rétege NEM a
  produkciós.
- A spike revision/publish-pointer sémája **szándékosan vékony, repository-interface mögött**, hogy a
  Mongo→Postgres csere konténerezve legyen; a referenciális integritás a spike-ban app-szintű, a
  pilotnál DB-szintű. A P0.2 séma mindkét store előtt landol.
- Nyílt kockázat rögzítve: egy zöld Mongo-spike **NEM** validálja a produkciós store atomicitását
  (több-táblás tranzakció) — ezt a pilot méri.

### 16.6 Szekvenálás — a kereslet-kapu a technikai spike ELŐTT

A §12.5 és a §15.6 maga mondja: a kereslet a legfontosabb, kód előtt validálandó. A technikai
megvalósíthatóság („tipizált tartalom → statikus HTML") nem a nagy ismeretlen; a **kereslet +
szubsztitúció** (megtörik-e egy off-the-shelf editor a bespoke GSAP-on) az. Ezért a §15.3/A **elé**:

1. **Kereslet-kapu (concierge / Wizard-of-Oz, nulla platform-eng).** 2-3 valódi HU ügyfélnek
   szállítsd a client-safe editálást *kizárólag meglévő eszközzel* (pl. Storyblok Visual Editor vagy
   „szerkeszd az N mezőt" űrlap egy bespoke buildre), **valódi áron.** Mérd (műszerezve, nem
   kérdezve): (a) megveszik-e az áron; (b) tényleg szerkesztik-e, és milyen gyakran; (c) tényleg
   megtört-e az off-the-shelf a GSAP-on úgy, hogy egyedi architektúra kellett; (d) maradnak-e, vagy
   elcsúsznak Shopify/Webflow/semmi felé. **Go-küszöb:** ≥N elkötelezett, fizető ügyfél, ahol egy
   off-the-shelf editor *bizonyíthatóan* elbukott a bespoke motionon (ez a §15.6 „Product go").
2. **Szűkített technikai spike (csak ha az 1. zöld).** A valódi ismeretlen: a bespoke GSAP
   `mount/refresh/destroy` túléli-e a tipizált editet + hosszabb szöveget egy *valódi*
   referencia-oldalon, **locale × viewport × font** mátrixszal (§14.5). Ne front-loadold a
   migráció-motort / atomi-publisht / security-render-pipeline-t, amíg az 1. kapu nem bizonyít.
3. **Value-prop őszinteség.** A §14.5 reflow-mérséklés **szűkíti** a differenciátort: a „biztonságos"
   animáció-halmaz átfed a Webflow/Framer natív készletével. A megmaradó egyedi érték a *bespoke
   motion megőrzése egy meglévő, kézzel épített oldalon* + a fejlesztő-birtokolta struktúra — ezt kell
   az 1. kapunak **igazolnia**, nem feltételezni.

### 16.7 Frissített készültségi értékelés

| Tengely | 1. kör (§15) | 2. kör (§16 után) |
|---|---|---|
| Belső konzisztencia (kanonikus séma) | ~4/10 | a §16.1 után **lekódolható** |
| Reconciliation teljesség | ~4/10 | a §16.2 után **specifikált** |
| Biztonsági modell | ~5/10 | a §16.3 után a **spike-ra elég**; multi-tenant a pilotnál |
| v1 concurrency (néma vesztés) | ~4/10 | a §16.4 revision-guard **lezárja** |
| Szekvenálás / kereslet | ~3/10 | a §16.6 **megfordítja** a sorrendet |

**Nettó:** a koncepció kiállta a második, adverzariális kört; a §15 P0-kapuk állnak. A három
KRITIKUS blokkoló (kanonikus séma, reconciliation, sink-lista) itt **doc-szinten feloldva** — a
fixed-only spike adat/render-rétege ezután **lekódolható**. A megmaradó feltétel **stratégiai**: a
kereslet-kaput (§16.6) a technikai spike **előtt** kell futtatni.

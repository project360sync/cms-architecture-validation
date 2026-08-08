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

<!-- Kollekció-KÖTÉS: a sorok nem itt élnek, a template csak hivatkozik rájuk -->
<section data-cms-section-id="products-1" data-cms-section-type="products">
  <div class="pgrid" data-cms-collection="termekek">
    <template data-cms-item>                        <!-- SOR-SABLON: render, nem séma -->
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
- `data-cms-collection="<name>"` — ismétlődő régió; a `<name>` egy **létező kollekcióra
  hivatkozik**, nem definiál újat (ADR-004, §16.1). Benne egy `<template data-cms-item>`
  prototípus (a fejlesztő egy elemet ír meg); a benne lévő `data-cms-field` nevek a kollekció
  **oszlopaira** hivatkoznak. A sorok sémáját tehát **nem** ez a markup határozza meg: a séma a
  `CollectionDoc`-ban él, a szekció-séma pedig `requires`-szel mondja meg, mely oszlopokra van
  szüksége a rendereléshez (§3.4). Nem létező kollekció-név vagy hiányzó oszlop → a publish és a
  dry-run fail-closed hibája (`UnboundCollection` / `UnsatisfiedBinding`).
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
  }
  // A kollekció SORAI nem ebben a dokumentumban élnek — csak a `collection` KÖTÉS van itt.
}
```

**Ez a példa illusztratív, és a §16.1 felülírja.** Két ponton lényeges: a `collections` kulcs
**nincs** a tartalom-docban (ADR-004 — a sorok önálló `CollectionDoc`-ban, saját revízió-vonalon
élnek), és a mezőidentitás kvalifikált `(scopeId, fieldName)` pár.

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
  "permissions": {                 // SZEKCIÓ-szintű; sor-jogosultság NINCS itt (§16.1)
    "editContent": true,
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
// sections/products.schema.json — kollekcióra KÖTÖTT szekció
{
  "type": "products", "label": "Termékek",
  "collection": {                  // KÖTÉS + KÖVETELMÉNY — nem séma-definíció
    "binding": "termekek",         // melyik kollekciót rendereli
    "requires": [                  // mely oszlopok kellenek a rendereléshez, milyen típussal
      { "name": "img",   "type": "image"    },
      { "name": "title", "type": "text"     },
      { "name": "desc",  "type": "longtext" }
    ]
  },
  "permissions": {                 // SZEKCIÓ-szintű; az addItems/reorderItems a CollectionDoc-é
    "editContent": true,
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

**A szekció-séma nem definiálja a kollekció sorait (ADR-004).** A `collection.requires` lista
**követelmény**, nem definíció: azt mondja meg, mire van szüksége a rendereléshez, nem azt, hogy a
sor miből áll. A sorok oszlopait, típusait és `required`-ségét a `CollectionDoc` sémája hordozza
(§16.1), és a kollekció **több szekcióhoz** is köthető, akár eltérő `requires` listával. Egy
kielégítetlen követelmény vagy egy nem létező kötés a publish és a re-ingeszt dry-run **fail-closed**
hibája (`UnsatisfiedBinding` / `UnboundCollection`, §16.2.1); az implementáció **sosem** hoz létre
hiányzó oszlopot a szekció-sémából. A kollekció sémájának módosítása külön migráció, saját szűk
szótárral (§16.2.3).

**A sor-jogosultság a kollekcióé, nem a szekcióé (ADR-004).** Az `addItems` / `reorderItems`
permission ezért **kikerült** a szekció-sémából: a sorok élete a kollekció saját felületén zajlik
(§16.2.2 zárómondata), és mivel egy kollekció **több szekcióhoz** is köthető, egy szekció-oldali
sor-permission vagy uniót, vagy metszetet, vagy semmit jelentene — három védhető, de **különböző**
válasz ugyanarra a kérdésre, ami az authz-ban nem elfogadható. A normatív hely a `CollectionDoc`
`permissions` blokkja (§16.1): `editRows`, `addItems`, `removeItems`, `reorderItems`. **Egy
szekció-kötés önmagában semmilyen sor-jogosultságot nem ad.** A szekció felől csak **szűkítés**
jöhet, capabilityn keresztül: az editor akkor és csak akkor ajánlja fel a sor-átrendezést, ha a
kollekció `permissions.reorderItems`-e igaz **és** a kollekcióra kötött **minden** szekció
`capabilities.reorderSafe`-je igaz — **metszet, nem unió** (§3.4.1); ugyanígy a felvételre
`duplicationSafe`, a törlésre `removalSafe`. Így a §5 szerveroldali elutasítása is
**végrehajtható**: van **egy** policy, amit konzultálni lehet, és az a kollekcióé.

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
a section nem mozdítható — de a sor-jogosultság forrása ilyenkor is a `CollectionDoc`
`permissions` blokkja (§3.4, §16.1), a szekció csak a capabilityjével **szűkíthet** rajta.

---

## 4. Pipeline-ok

### 4.1 Onboarding-import (egyszer, HTML → template + tartalom)

Az ingeszt **onboarding-lépés, nem folyamatos overlay**. A tetszőleges HTML-ből előáll:
(a) az **annotált template** (a struktúra, content-addressed JS-bundle), és (b) a **kezdeti
tartalom-doc** (a slotokból kiolvasott értékek). Innentől a **tartalom-store az igazság**.

```text
importSite(html, assets, js):
  1. sanitize (inline <script>/on* ki; engedélyezett dependency-k vendorizálva és hash-elve)
  2. a data-cms-* markerekből kiolvassa a slotokat/kollekció-kötéseket/szekciókat
  3. template  := struktúra (slotok üres/prototípus formában) + CSS + JS-bundle
  4. content   := a jelenlegi oldal-szintű értékek kiolvasva a slotokból (fields)
  5. collections := kollekció-NEVENKÉNT egy CollectionDoc — séma a névre hivatkozó ÖSSZES kötés
                    item-sablonjából + `requires`-éből egyeztetve; sorok az import-manifestben
                    megnevezett EGY forrás-régió ismétlődéseiből (BOOTSTRAP, egyszeri)
  → tárolás: template (verziózva) + content (névre kulcsolva) + CollectionDoc-ok (külön entitás)
```

**Kollekciónként egy `CollectionDoc`, nem kötésenként egy.** A `name` a `siteId`-n belül **egyedi**
(§16.1), és egy kollekció **több szekcióhoz** is köthető (§3.4) — egy kiemelt rács a `/`-on és a
teljes lista a `/termekek`-en ugyanazt a `data-cms-collection="termekek"` nevet hordozhatja. A
kötésenkénti bootstrap ezért vagy megsértené az egyediséget, vagy némán eldobna egy sorhalmazt az
onboardingon. Helyette:

- **Séma:** a névre hivatkozó **összes** kötés item-sablonja és `requires` listája **egyeztetve**
  áll össze egy sémává. Ha két kötés **összeegyeztethetetlen** item-alakot deklarál — eltérő
  mezőnév-halmaz, vagy azonos névhez eltérő típus (akár az elemből inferált, akár a `requires`-ben
  megadott) —, az import **elutasít** (`AmbiguousCollectionBootstrap`), és a séma **emberi**
  megadását kéri. Az implementáció **sosem old fel** séma-ütközést szabállyal (nem „a bővebb nyer",
  nem „az első nyer").
- **Sorok:** ha a névre **több** kötés is hordoz ismétlődéseket a markupban, az import **elutasít**
  (`AmbiguousCollectionBootstrap`), amíg a kötelező import-manifest (§15.5 #1) meg nem nevezi, melyik
  régió a **sorok forrása** (`rowsFrom`). Sorhalmaz **sosem dobódik el** azért, hogy egy második
  kötés beleférjen — ez pontosan az 1. garancia esete, onboarding-időben.

Az 5. lépés az onboarding **egyetlen** olyan pontja, ahol egy kollekció sémája a designer HTML-jéből
származik — ez a bootstrap, nem szabály (ADR-004). Innentől a kollekció sémája **önálló**, és a
re-ingeszt nem írja (§16.2.2). A meglévő, `schemaVersion: 3`-as adat ugyanennek a bootstrapnek az
utólagos, azonosító-megőrző párja: a §16.2.4 séma-lift.

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
function render(template, content, collections, locale): Html {   // NÉGY paraméter (§16.1)
  validateReferences(template.manifest, content, collections)  // kötések + `requires` (§16.1)
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
    hydrateTyped(root, sec, resolveBinding(collections, sec.collection), template.schemas)
    if (template.mode === "composable") mount.append(root)
  }

  removeAllPrototypes(doc) // <template> sosem marad publikált tartalomként
  attachBundle(doc, template.jsBundle)
  return serialize(doc)
}
```

A szignatúra **négy** paraméteres: a render **locale-onként egyszer** fut (§14.1), és a hidratálás a
§14.2 `resolve(field, locale)`-ján megy át — a `locale` tehát a render **bemenete**, nem a
`content`-ből kikövetkeztetett érték. Ez a §16.1 szignatúrája; a korábbi három paraméteres alak
elírás volt.

A kollekció-render ugyanezt a szabályt követi: a `<template data-cms-item>` **contentjét**
klónozza (nem magát a `<template>` elemet) a **kötött kollekció** minden sorára, a sor saját,
tárolt `itemId`-jával (a rendererben id nem keletkezik), validálja a mezőket a kollekció sémája
ellen, majd eltávolítja a prototípust. Hiányzó vagy duplikált section-, block-, collection- vagy
field-id esetén a publish hibával leáll; feloldatlan kötés (`UnboundCollection`) vagy kielégítetlen
`requires` (`UnsatisfiedBinding`) esetén szintén.

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

**Ez a pszeudokód illusztratív és felülírt.** A névegyeztetés kvalifikált `(scopeId, fieldName)`
páron történik (§16.1), a teljes protokoll a §16.2. A `hasSlot(name)` csupasz-név alakja
**nem implementálható**: egy csupasz névre kulcsolt egyeztetés scope-ok között cross-assignol,
amit a §16.2 tilt.

```text
reingest(newHtml):
  nextTemplate := importTemplate(newHtml)        // friss struktúra, ugyanazok a slot-nevek
  for (const [name, value] of content.fields)
    nextTemplate.hasSlot(name) ? keep(name, value)   // a slot még létezik → megtartjuk
                               : quarantine(name)      // eltűnt → jelöljük, nem dobjuk némán
  a kollekciók ÉRINTETLENEK — a re-ingeszt egyetlen sort sem olvas és nem ír (ADR-004)
```

**A re-ingeszt nem nyúl a sorokhoz.** A kollekció önálló entitás, saját sémával és saját
revízió-vonallal (§16.1); a redesign csak a **kötést** és a szekció `requires` listáját érinti.
Feloldatlan kötés vagy kielégítetlen oszlop-követelmény a dry-run fail-closed hibája — nem néma
átalakítás, és nem is automatikus séma-változás a sorokon (§16.2.1).

**Nincs id-drift, nincs vak felülírás.** A slot átnevezése/törlése = kontrollált, jelzett
migráció (nem néma adatvesztés). A név-egyeztetés csak az automatikus alapértelmezés:
produkciós séma-váltásnál verziózott, tesztelhető migrációs manifest kell
(`rename`, `transform`, `split`, `merge`, `delete→quarantine`). A re-ingeszt
tranzakciósan készít preview-verziót, és csak teljes schema/render/asset validáció után
aktiválható; rollbackhez az előző template + content verzió együtt marad meg.

---

## 5. Editor-modell (client-safe)

- **Granularis szerkeszthetőség** (per section/blokk, a manifestből): külön content- és
  kompozíciós permissionök; a **sor-** (item-) permissionök forrása a `CollectionDoc`
  `permissions` blokkja, nem a szekció-séma (§3.4, §16.1). Az editor nem egyetlen `locked`
  flagből következtet.
  A tiltott műveletet nem ajánlja fel, és az API ugyanazt szerveroldalon is elutasítja — sor-műveletnél
  a **kollekció** policyját konzultálva, a kötött szekciók capabilityjeivel metszve.
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
  }
  // a kollekció-sorok külön entitásban, locale-onkénti mezőértékekkel — lásd §16.1
}
```

**Ez a példa illusztratív, és a §16.1 felülírja** két ponton: a lapos `hero.line1` kulcs hibás
(kvalifikált `(scopeId, fieldName)` pár kell), és a `collections` kulcs **nincs** a
tartalom-docban — a sorok önálló `CollectionDoc`-ban élnek (ADR-004). A locale-dimenzió maga
változatlan: a sor `fields`-e locale-onkénti, az `itemId` és az oszlopnév nem.

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

**A policy és a `resolve()` a sor-értékekre is vonatkozik (ADR-004).** A fenti `resolve(field, locale,
content)` alak `ContentDoc`-ra, oldal-szintű mezőkre íródott (`content[locale]?.fields[field]`). A
sorok kikerülésével ugyanez a szemantika **változatlanul** vonatkozik a `(collectionName, itemId,
locale, fieldName)` négyesre is: a feloldás a `CollectionDoc` `rows[*].fields[locale][name]`
bejegyzésén (és `image` oszlopnál az `assets[name].alt[locale]`-on) megy át, a `CollectionDoc` séma
`required` / `altRequired` kulcsa **a kötelező eset**, a `fallbackAllowed` / `hiddenWhenMissing`
kollekcióra és oszlopra ugyanúgy deklarálható, és egy hiányzó kötelező **sor-érték** ugyanúgy
bukathatja a locale publikálását (`UnpublishableLocale`), mint egy oldal-szintű mező. **Ez az
egyetlen hely, ahol a sorok kötelezőségét a rendszer érvényesíti**, és a §16.2.3
`RequiredWithoutDefault` záró-ellenőrzése erre a definícióra hivatkozik — enélkül a draft-quarantine
megszüntetésének (c) premisszája nem állna.

Fallback **previewban** hasznos, de publikálásnál nem lehet univerzális alapértelmezés:
kevert nyelvű oldal SEO-, jogi és márkakockázat. A publish policy oldal/mező szinten
deklarálja, hogy `required`, `fallbackAllowed` vagy `hiddenWhenMissing`; a jogi, navigációs,
SEO- és konverziós mezők alapból `required`. Forrás-, kontextus- vagy sémaváltozáskor a
fordítás **stale**, és a policy dönti el, hogy blokkolja-e a locale publikálását.

### 14.3 Kollekciók locale-ok közt

Az elemek **stabil `id`-t** osztanak a locale-ok közt (a `7f3a` *ugyanaz* a termék),
és csak a **mezők** locale-onkéntiek → a fordítás elemenként/mezőnként követhető.
Locale-specifikus elem (csak DE-ben promó) = opcionális `locales: ["de"]` flag — a kivétel.

Ez a szabály **változatlan** azzal, hogy a sorok kikerültek a template alól (ADR-004): a
`CollectionDoc` `rows[*].fields` blokkja locale-onkénti, az **oszlopnév** viszont struktúra, tehát
locale-független (§16.1). Ebből következik a §16.2.3 költségmodellje: egy oszlop-szabály
kiértékelése **sem** a sorok, **sem** a locale-ok számával nem szorzódik.

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
| 8 | **Média immutable asset-id-val kötődik.** | A mező locale-specifikus alt/caption/crop metaadatot, nem fájl-URL-t tárol. Soft delete + reference graph + retention után GC; a re-ingeszt az **oldal-szintű** asset-hivatkozásokat reconciliálja — a kollekció-sorok `assets` bejegyzéseit **nem olvassa és nem írja** (ADR-004, §4.4), azok a `CollectionDoc` saját revízió-vonalán élnek. |
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
| collection-item (sor) | `itemId` | collection-on belül | `itm_7f3a` |
| collection-column (oszlop) | `(collectionName, fieldName)` | site-on belül | `(termekek, title)` |
| field | `(scopeId, fieldName)` | scope-on belül | `(hero-1, line1)` |

**Kulcs-invariánsok:**
- **A kollekció nem a template alatt él (ADR-004).** Egy kollekciónak **saját, stabil sémája** van,
  amit nem a designer HTML-je határoz meg; a template **hivatkozik** egy kollekcióra (`collection`
  kötés a szekción), nem **definiálja** azt. Ebből következik a `ContentDoc` alakja alább: a
  `collections` kulcs **kikerült** belőle, és a sorok önálló entitásban, `CollectionDoc`-onként,
  **saját revízió-vonalon** élnek. Egy redesign ezért nem séma-változás a sorokon.
- **A mező sosem a típusra kulcsol.** Az oldal-szintű mezőidentitás mindig `(scopeId, fieldName)`,
  ahol a scope egy section-instance vagy egy block. A DOM-annotáció a **csupasz** nevet hordozza
  (`data-cms-field="line1"`), a renderer/reconciler a befoglaló `data-cms-section-id` /
  block-scope alapján kvalifikálja. Ez feloldja a §3.2 (csupasz) ⟂ §10
  (`hero.line1`) annotáció-ellentmondást is: a helyes annotáció a **csupasz** név.
- **A sor-érték identitása `(collectionName, itemId, locale, fieldName)`.** A `collection-item`
  továbbra is **identitás-szint** — a sornak stabil `itemId`-ja van —, de **nem** migrációs
  operátor-hatókör: a kollekció-oldali migráció hatóköre az **oszlop** (§16.2.3).
- **A blokknak stabil `id`-ja van** (`{ "id":"blk_9f2", "type":"gcard", "settings":{…} }`), nem
  tömb-index — enélkül pozíció-horgonyzott lenne, épp az a törékenység, ami ellen a modell érvel (§8).
- **A struktúra (order / type / collection-kötés / oszlopnév) locale-független; csak a mezőértékek
  locale-scoped-ek.** Kivétel a `composable` per-locale order (§14.4): külön, opcionális felülírás —
  nem az alap.
- **A név-egyenlőség definiált: NFC + kódpont-egyezés, kis-nagybetű-érzékenyen.** Két mező- vagy
  oszlopnév akkor és csak akkor **azonos**, ha **Unicode NFC** normalizálás után **kódpontonként**
  egyezik; a tárolt alak mindig NFC. Megengedett karakterosztály: Unicode betűvel kezdődik, utána
  Unicode betű / számjegy / `_`, legfeljebb 64 kódpont; ezen kívüli név **séma- és import-validációs
  hiba** (`InvalidColumnName`), nem néma normalizálás. Ha egy sémán belül két oszlopnév NFC után
  **csak kis-nagybetűben** tér el, az szintén hiba (`AmbiguousColumnName`). Ez azért invariáns és nem
  implementációs részlet, mert a név-egyenlőség dönti el a tiszta-oszlophalmaz szabályt, az
  auto-matchet és a foglaltságot (§16.2.3): egy NFD/NFC eltérés különben az egész kollekciót „nem
  tiszta" halmazzá tenné, és **minden** értékét quarantine-ba küldené, miközben egy másik
  implementáció **semmilyen** változást nem látna.
- **A sor `locales` szűkítése struktúra, de nem oszlop.** Egy sor opcionálisan hordozhat
  `locales: ["de"]` listát — a §14.3 locale-specifikus sor kivétele —, és ez a kulcs **része a
  kanonikus alaknak** (a `collection.schema.json` engedi és validálja). **Nem oszlop:** nincs a
  `schema.columns` listában, az oszlop-szótár (§16.2.3) **nem címzi**, `renameColumn` /
  `dropColumn` nem hat rá, és sosem kerül quarantine-ba. Hiányzó `locales` = a sor minden locale-ban
  látszik. A lista szerkesztése **sor-életciklus**, azaz hétköznapi szerkesztés.
- **A sor-jogosultság a `CollectionDoc`-é.** Az `addItems` / `removeItems` / `reorderItems` /
  `editRows` permission a kollekció saját `permissions` blokkjában él, nem a szekció-sémában (§3.4):
  egy kollekció több szekcióhoz köthető, és egy szekció-oldali sor-permission uniót, metszetet vagy
  semmit jelentene — három védhető, de különböző válasz ugyanarra a kérdésre.

**Kanonikus váz — `ContentDoc`** (egy oldal, két locale, egy kollekció-kötés):

```jsonc
{
  "schemaVersion": 4,                        // 4 = a `collections` kulcs nélküli alak (§16.2.4)
  "siteId": "site_v1b0r",
  "defaultLocale": "hu",
  "locales": ["hu", "en"],
  "pages": {
    "home": {
      "templateVersion": "tpl_12",           // a reconciliation kulcsának egyik fele (§16.2.2)
      "template": {                          // STRUKTÚRA — locale-független
        "mode": "fixed",
        "order": ["hero-1", "products-1"],
        "sections": {
          "hero-1":     { "type": "hero" },
          "products-1": { "type": "products", "collection": "termekek" }  // KÖTÉS, nem definíció
        }
      },
      "content": {                           // ÉRTÉKEK — locale-scoped, scope=példány-id
        "hu": { "fields": { "hero-1": { "line1": "FÉM.",  "line2": "ÜVEG."  } } },
        "en": { "fields": { "hero-1": { "line1": "METAL.","line2": "GLASS." } } }
      }
    }
  }
  // `collections` NINCS a content-docban — lásd a `CollectionDoc`-ot alább (ADR-004).
}
```

**Kanonikus váz — `CollectionDoc`** (kollekciónként egy, saját revízió-vonallal):

```jsonc
{
  "collectionSchemaVersion": 1,              // a KOLLEKCIÓ sémájának verziója, a §16.2.3 kulcsa
  "siteId": "site_v1b0r",
  "name": "termekek",
  "schema": {                                // OSZLOPOK — nem a designer HTML-jéből származik
    "columns": [
      { "name": "title", "type": "text",     "required": true },
      { "name": "desc",  "type": "longtext" },
      { "name": "img",   "type": "image",    "altRequired": true }
    ]
  },
  "permissions": {                           // SOR-jogosultság — a kollekcióé, nem a szekcióé (§3.4)
    "editRows": true, "addItems": true, "removeItems": true, "reorderItems": true
  },
  "rows": [
    { "id": "itm_7f3a",
      // "locales": ["de"],                  // OPCIONÁLIS locale-szűkítés (§14.3) — nem oszlop
      "fields": { "hu": { "title": "Berg Passive", "desc": "Uw=0,66" },
                  "en": { "title": "Berg Passive", "desc": "Uw=0.66" } },
      "assets": { "img": { "assetId": "ast_abc", "alt": { "hu": "…", "en": "…" } } } }
  ]
}
```

**A `schemaVersion` története.** A `2` ennek a szakasznak az eredeti váza. A `3` a step-4 kampányban
keletkezett, a `templateVersion` bevezetésével — a megvalósításban landolt, a pinelt vázban nem
látszott. A `4` **ez a lépés**: a `collections` kulcs eltávolítása a `ContentDoc`-ból. A meglévő
`3`-as adat átvitele nem rekonciliáció, hanem egyszeri, azonosító-megőrző kiemelés; a normatív
eljárás és az, hogy miért nem körkörös, a **§16.2.4**-ben áll.

**JSON Schema a spike előtt.** Két séma rögzíti a fenti alakokat: `content.schema.json` és
`collection.schema.json` — required kulcsok, id-formátumok, a **referenciális integritás**
(minden `order`-id létezik a `sections`-ben; minden `assetId` létező assetre mutat) és a
**séma-konformancia**: egy `CollectionDoc` sorai **nem hordozhatnak** olyan kulcsot, ami a doc saját
`schema.columns` listájában nincs benne — sem `rows[*].fields[locale]`-ben, sem `rows[*].assets`-ben
—, és a hordozott érték alakja a deklarált típusnak felel meg. (Ez teszi ellenőrizhetővé a §16.2.3
`dropColumn` kulcs-eltávolítását: „árva" kulcs nem maradhat a sorban.) A kötés
integritása **entitás-határon átnyúló**, ezért külön kimondva: minden `sections[*].collection` név
létező `CollectionDoc`-ra mutat ugyanazon a `siteId`-n, a `(siteId, name)` párra **pontosan egy**
`CollectionDoc` létezik (kollekciónként egy, **nem kötésenként** egy — §4.1), és a szekció-séma `requires` oszloplistája
(§3.4) **kielégül** a kötött kollekció sémájából. Feloldatlan kötés → `UnboundCollection`,
kielégítetlen oszlop-követelmény → `UnsatisfiedBinding`; mindkettő a publish és a dry-run
fail-closed hibája (§16.2.1). A spike store-ban ez a check app-szintű (§16.5), a pilotnál DB-szintű.
A render / reconcile / i18n mind ezt az **egy** alakot fogyasztja: a §4.2
`render(template, content, collections, locale)` szignatúrát kap, és a hidratálás a §14.2
`resolve(field, locale)`-on megy át (fallback / required / hidden policy a hidratáláson **belül**,
locale-publikálást bukathat).

### 16.2 Reconciliation & migráció — teljes protokoll (P0.4 kiegészítés)

> **Ez a szakasz le van cserélve, nem javítva.** Az **ADR-004** (elfogadva 2026-07-28) kiveszi az
> ismétlődő, strukturált adatot a template alól: a kollekciónak saját, stabil sémája van, a template
> **hivatkozik** rá, nem **definiálja** azt. A szakasz korábbi normatív szövege — a `collection-item`
> operátor-hatókör, az „item-mező szinten, minden itemre külön" szabály, a sor-életciklus tilalmi
> mondata és a „Kollekció item-séma változás" / draft-quarantine bekezdés — a lezárt step-4 kampány
> tekintélyi pinjén, a `cms-architecture-validation@bb0938a` commiton változatlanul elérhető marad,
> és az ott mért eredmények arra a szövegre vonatkoznak. Az alábbi szöveg **önmagában áll.**

A §4.4 auto-name-match csak *törlésre* védett (quarantine). Az adverzariális kör több nem kezelt
esetet talált (az eredeti három mellé a §16.2 round-0 tekintély-kihívása továbbiakat), amelyek
cáfolják a „nincs vak felülírás" ígéretet. A feloldás **két, egymást nem érintő migráció-fajtára**
bomlik: az oldal-hatókörű redesign-migrációra (§16.2.2) és a kollekció-hatókörű oszlop-migrációra
(§16.2.3).

**A négy garancia mindkét fajtára változatlanul áll.** (1) **Nincs néma veszteség** — ami nem talál
helyet, azt az ember **látja**. (2) **Nincs néma áthelyezés** — azonosítót sosem vezetünk le
tippelésből; a `line1`↔`line2` csere tilalma marad, és oszlop-szinten is érvényes. (3) **Az előnézet
leírja az aktiválást**, és emberi jóváhagyás nélkül semmi nem aktiválódik. (4) **Revíziók és
rollback.** Ahol egy operátor **hiányzik** a szótárból, ott a következmény **látható elutasítás**,
nem veszteség: az érintett értékek quarantine-rekordba kerülnek, a diffben megjelennek, és az
aktiválás után is olvashatók. Ha egy implementáció úgy valósítja meg ezt a szakaszt, hogy egy nem
támogatott eset értéket **ejt**, az szerződésszegés, nem hiányzó funkció.

#### 16.2.1 A két migráció-fajta, és hogy soha nem keverednek

| | **Redesign-migráció** (§16.2.2) | **Kollekció-séma-migráció** (§16.2.3) |
|---|---|---|
| Kiváltó | új template-verzió (re-ingeszt, §4.4) | a kollekció sémájának szándékos módosítása |
| Hatókör | section-instance és block **mezői** | egy kollekció **oszlopa**: `(collectionName, fieldName)` |
| Manifest-kulcs | `(from-templateVersion → to-templateVersion)` | `(collectionName, from- → to-collectionSchemaVersion)` |
| Mit ír | a `ContentDoc` egy új revízióját | egy `CollectionDoc` egy új revízióját |
| Amit **nem** érint | egyetlen kollekció-sort sem | a template-et és az oldal-szintű mezőket sem |
| Szótár | hat operátor | **két** operátor |
| Gyakoriság | minden redesign | ritka, szándékosan kiváltott |

Egy manifest **pontosan egy** fajtához tartozik. Az a manifest, amely mindkét fajta bejegyzését
hordozza, a dry-run szakaszban **elutasításra kerül** (`MixedMigrationScope`). Egy aktiválás sosem ír
`ContentDoc`-ot és `CollectionDoc`-ot ugyanabban a revízióban; ez alól az **egyetlen** kivétel a
§16.2.4 séma-lift, ami nem migráció, hanem egyszeri áthelyezés.

**Ha egy redesign kollekció-oldali változást igényel, a két lépés csak akkor tehető sorrendbe, ha a
kollekció-oldali változás *additív*.** Additív az a változás, amely egyetlen **élő** kötés
`requires` listáját sem sérti — jellemzően új oszlop felvétele. Ilyenkor a sorrend: előbb a
kollekció-migráció, utána a redesign. **Nem additív** változásnál (egy `requires`-ben szereplő
oszlop átnevezése vagy eltűnése) a sorrend **egyik iránya sem jó**, és ezt itt kimondjuk, mert a
korábbi „a két lépés sorrendbe tehető" állítás erre az esetre **hamis** volt: kollekció-először az
élő template `requires`-e sérül (publish fail-closed `UnsatisfiedBinding` egy olyan előnézet után,
ami „átnevezve, 4 érték"-et ígért — 3. garancia), template-először pedig ez a szakasz maga utasít el
(`UnsatisfiedBinding`). A feloldás az **összekapcsolt kiadás** (§16.2.3,
`BindingRequirementBreak`), nem a sorrend. Ha az új template olyan oszlopot vár a kötött
kollekciótól, ami nincs meg (a
szekció-séma `requires` listája nem elégül ki, §3.4), a redesign dry-runja elutasít
(`UnsatisfiedBinding`), és **előbb** a kollekció-séma-migrációt kell lefuttatni és jóváhagyatni. Ha
az új template olyan kollekció-nevet köt, amihez nem tartozik `CollectionDoc`, a dry-run elutasít
(`UnboundCollection`): a kötés feloldása **explicit template-oldali aktus** — a fejlesztő a
meglévő kollekció nevére köti a szekciót, vagy új kollekciót hoz létre —, ami egyetlen sort sem
mozgat és egyetlen értéket sem quarantine-ol. **A `CollectionDoc` `name` mezője immutábilis**, épp
azért, hogy egy „átnevezés" ne igényeljen entitás-határon átnyúló írást több kötés fölött. Egy
kollekció „átnevezése" ezért nem tartalom-migráció, hanem kötés-feloldás — a sorok, az `itemId`-k
és az értékek érintetlenek maradnak, és ez az, ami a régi szöveg megoldatlan „átnevezett sor
materializálása" kérdését **tárgytalanná** teszi: sor sosem keletkezik és sosem nevez át migráció.

**„Új kollekciót hoz létre" — ennek a v1-ben nincs automatizált útja, és ez szándékos.** Egy
kollekció sémájának **két** forrása van, és mindkettő **egyszeri**: az onboarding-bootstrap (§4.1,
5. lépés) és a séma-lift (§16.2.4). Mindkettő az onboarding pillanatához kötött, és onnantól
**elzárt**. Ebből következik a fenti `UnboundCollection` escape teljes következménye: egy redesign,
amely olyan kollekció-nevet köt, amihez nincs `CollectionDoc`, **látható elutasítás**, és a
v1-ben **nincs olyan út, amely a hiányzó sémát az új HTML item-sablonjából pótolná** — az pontosan
az, amit az ADR-004 kizár (a template definiálná a sort), és a hátsó ajtón vezetné vissza a
template-ből származó séma-tulajdonlást. A feloldás **emberi**: vagy egy **létező** kollekció nevére
kötik a szekciót, vagy a kollekció sémáját **ember adja meg** azon a felületen, amit a
séma-tulajdonlás kérdésének eldöntése hoz létre (ki birtokolja a kollekció sémáját — nyitott
v1-döntés, §16.2.3 „veszteségmentes típus-tágítás"). Amíg ez a kérdés nyitott, **az új kollekció
felvétele nem automatizált művelet**, és a redesign a kötés feloldásáig **blokkolva marad**. Ez a
szűkítés a 2. garancia ára, és tudatosan vállalt.

#### 16.2.2 Oldal-hatókör — a redesign-migráció

Ez a szakasz a step-4 kampány által megkeményített szöveg, változatlan formában, azzal az egyetlen
különbséggel, hogy a hatóköréből a `collection-item` **kikerült**. A stílus szándékos: ez egy
**kötelező elutasítások listája**, nem képességlista.

**Precedencia (definiálva):** az explicit migrációs manifest **mindig nyer**; az automatikus
név-egyezés csak a *maradékot* viszi tovább — azokat a `(scopeId, fieldName)` párokat, amelyek neve
ÉS típusa változatlan, **ugyanabban a scope-ban.** A manifest a `(from-templateVersion →
to-templateVersion)` párra van kulcsolva. Ha a content `templateVersion`-je és a cél-template
verziója között **nem pontosan egy lépés** van, vagy az adott lépéshez nincs manifest, a dry-run
**elutasít** (`UnbridgedVersionGap`). Az auto-match **több verziónyi távolságot nem hidalhat át.**
Az operátorok és az auto-match a `locales` **minden elemére külön** futnak: a maradék locale-onként
számítódik, a manifest minden bejegyzése minden locale-ra alkalmazódik, és a diff sorai a
`(locale, scopeId, fieldName)` hármasra kulcsoltak. Egy locale-ban hiányzó forrásérték az adott
locale-ban no-op, nem hiba. Ha egy `(scopeId, fieldName)` pár neve és scope-ja változatlan, de a
**típusa** megváltozott, az **nem maradék**: explicit `transform` nélkül a forrásérték
quarantine-ba kerül, a slot pedig **érték nélkül** marad. Típusváltás sosem megy át
néma `kept`-ként.

> **A locale-dimenzió külön, folyamatban lévő döntés tárgya.** Ez a szakasz a locale-kezelést
> változatlanul veszi át a korábbi szövegből, és **szándékosan nem épít rá többet**: azt, hogy a v1
> egyáltalán több locale-t támogat-e, önálló döntés dönti el, nem ez az újraírás.

A manifest bejegyzései **egyidejűleg**, a migráció előtti tartalom egyetlen pillanatképe fölött
értékelődnek ki, nem egymás után sorrendben: minden operátor forrása a migráció **előtti** érték, és
egyetlen operátor sem lát másik operátor kimenetét. Két bejegyzés, amely ugyanarra a célslotra ír,
**ütközés** → a dry-run elutasít. Így a `rename: a→b` + `rename: b→a` pár helyes cserét ad, nem
kettős felülírást.

A manifest elsőbbsége **nem mentesít a validáció alól.** A dry-run minden bejegyzést a **következő**
template ellen old fel, mielőtt bármit alkalmazna: a forrás-scope-nak a régiben, a cél-scope-nak és
cél-mezőnévnek az újban léteznie kell, és a cél mezőtípusának el kell fogadnia az értéket. Bármely
fel nem oldható bejegyzés a **teljes migrációt** elutasítja a dry-run szakaszban (fail-closed).

**Név-csere (`line1`↔`line2`) — némán korrupt volt, most tiltott.** Mivel a match scope+név szerint
történik, a csere két egyidejű átnevezésként detektálható. Ha egy scope-on belül a mező-névhalmaz
megváltozik (az alábbi értelemben **nem tiszta** hozzáadás/törlés), az auto-match **nem
cross-assignol**: vagy explicit `rename`/`move` bejegyzés kell, vagy a nem egyértelmű mezők
quarantine-ba mennek. Néma érték-áthelyezés nem történhet.

"Tiszta hozzáadás/törlés" azt jelenti, hogy a scope-ban **vagy csak** új mezőnév jelenik meg,
**vagy csak** mezőnév tűnik el. Ha egy scope-on belül **egyszerre** van megjelenő és eltűnő név, a
scope **nem tiszta**: az auto-match a scope **egyetlen** mezőjére sem fut le — a megmaradó nevek is
csak explicit `rename`/`move` bejegyzéssel vihetők tovább, minden fedezetlen mező quarantine-ba
megy. A változatlanul maradó mező továbbvitele az azonos forrású és célú `rename: scope.f → scope.f`
bejegyzéssel mondható ki; ez **nem** új operátor.

**Slot mozgatása szekciók között — most explicit `move`.** `move: hero-1.tagline → intro-1.tagline`.
Enélkül az érték elárvul (quarantine), nem tűnik el némán. A section-instance / block
**id megváltozása nem vezethető le** típus-, mezőhalmaz- vagy pozíció-egyezésből.
Scope átnevezéséhez explicit `move` bejegyzés kell minden érintett mezőre; enélkül a scope mezői
quarantine-ba mennek. A `move` cél-scope-jának a **következő** template-ben léteznie kell — ez
section-instance és block esetén teljesíthető, mert a cél-scope-ot a fejlesztő írja meg az új
template-ben. A `move` **sosem hoz létre** cél-scope-ot implikációból.

**Migrációs operátorok (specifikálva):**

| Operátor | Szemantika | Rollback |
|---|---|---|
| `rename` | `scope.a → scope.b`, érték változatlan | inverz rename |
| `move` | `scopeA.f → scopeB.f` | inverz move |
| `transform` | tiszta, verziózott függvény az értéken (nincs I/O) | csak deklarált inverzzel invertálható, különben a forrás quarantine-ban marad |
| `split` | `f → [f1, f2]` explicit mappinggel | merge inverz |
| `merge` | `[f1, f2] → f` explicit kombinátorral | split inverz |
| `delete` | `f → quarantine` (nem hard-törlés) | restore quarantine-ból |

Ha egy `rename`/`move`/`split`/`merge` **célslotja** a migráció előtti tartalomban értéket hordoz, és
ezt az értéket a manifest **egyetlen operátora sem forrásként fogyasztja el**, a művelet nem írja
felül: az ütközés a **dry-run szakaszban elutasítja a teljes migrációt**. Fogyasztásnak **kizárólag**
a manifest valamely operátorának forrás-hivatkozása számít; az auto-match által továbbvitt érték
**nem** fogyasztott, így az őt célzó manifest-bejegyzés helyesen elutasításra kerül. A cél
felszabadítását a manifest explicit `delete` (→ quarantine) bejegyzésével kell kimondani. Vak
felülírás nem történhet.

A **quarantine** (eltávolított érték megőrzése) **rekord, nem jelölés**: minden quarantine-sor
tárolja a `(locale, scopeId, fieldName)` hármast, a **nyers értéket**, a forrás mezőtípust, a forrás
`templateVersion`-t és az okot (`orphan | ambiguous | impure-scope | type-changed |
non-invertible-transform | delete | merged`), és az aktiválás után is megmarad, hogy a `restore`
végrehajtható legyen. Érték nélküli quarantine-sor **nem** elégíti ki a §4.4 "nem tűnik el némán"
követelményét.

Az invertálhatóság **deklarált, nem levezetett**: egy `transform` akkor és csak akkor invertálható,
ha a manifest megnevez hozzá egy verziózott inverz függvényt. Minden más `transform`
non-invertálhatónak számít, és a forrásérték **quarantine-ban marad**. Mintaértékeken végzett
round-trip **nem** bizonyít invertálhatóságot.

**Hatókör:** az operátorok **scope-ja** section-instance **vagy** block, és minden esetben az adott
scope **mezőire** hatnak (a mezőidentitás mindig `(scopeId, fieldName)`, §16.1). A `collection-item`
**nem operátor-hatókör**, és a kollekció-oszlop sem: a redesign-migráció **egyetlen kollekció-sort
sem olvas és egyetlen kollekció-értéket sem ír.** A `transform` determinisztikus és
mellékhatás-mentes, hogy a dry-run és a rollback reprodukálható legyen. A `split` és `merge`
operandusai **kvalifikált** `(scopeId, fieldName)` párok, és minden operandusnak **ugyanabban a
scope-ban** kell lennie; scope-ok közötti összevonás csak `move` + `merge` láncként fejezhető ki. A
`transform` a manifestben **csak névvel és verzióval** hivatkozható; az implementáció a kódbázis
zárt, tesztelt registryjében él. A manifest sosem hordoz függvénytestet, kifejezést vagy DSL-t.
Ismeretlen név vagy verzió → a dry-run elutasít.

**Sor-életciklus nem migráció.** Egy kollekció-sor felvétele, törlése, átnevezése vagy átrendezése
**hétköznapi szerkesztés** a kollekció saját felületén, a saját revízió-vonalán — nem szerepel sem
ebben, sem a §16.2.3 szótárában, és nem képezi tárgyát semmilyen re-ingeszt-diffnek.

**Költségmodell (normatív).** Egy manifest-bejegyzés kiértékelési költsége `locales × 1`. **A
kollekció-sorok száma semmilyen módon nem szorzója az operátor-kiértékelésnek** — sem itt, sem a
§16.2.3-ban. Egy tekintély, amely a sorok számát az operátor-kiértékelés szorzójává teszi, ezt a
szakaszt megsérti.

**Kötelező dry-run diff** (a §4.4-ből P0.4-be emelve): a re-ingeszt előbb egy diffet ad, amelynek
**forrás-oldali** kategóriái (`kept / renamed / moved / transformed / quarantined`) és **cél-oldali**
kategóriái (`new-empty / default-filled`) vannak, ami emberi jóváhagyás nélkül nem aktiválható.
A dry-run diff a jóváhagyás **tárgya és egyben az aktiválás bemenete**: a diff hordozza
a bázis content-revíziót, a `(from-templateVersion → to-templateVersion)` párt, **és minden kötött
kollekció `(collectionName, collectionSchemaVersion)` párját** — a régi **és** az új template
kötései szerint —, és az aktiválás **pontosan** a jóváhagyott diff sorait írja — nem újraszámolt
eredményt. A kollekció-séma pinelése **nem opcionális**: a redesign dry-runja `UnsatisfiedBinding`-et
emit, tehát kollekció-sémát **olvas**, és amit olvasott, azt rögzítenie kell. Ha az aktiválás
pillanatában a content-revízió, bármelyik template-verzió **vagy bármely pinelt kollekció
`collectionSchemaVersion`-je** eltér a diffben rögzítettől, az aktiválás **elutasítva**
(`StaleMigration`), és új dry-run kell. Enélkül egy 10:00-kor jóváhagyott redesign és egy 10:05-kor
aktivált oszlop-migráció (más entitás, más revízió-vonal) 10:10-kor **együtt** landolna, és a
kötés tartósan kielégítetlen maradna. A pinelés **nem** hozza az aktiválás atomicitási határán belülre
a `CollectionDoc`-okat: a redesign nem **írja** őket, csak a sémájukat **olvassa**; ezért továbbra is
igaz, hogy egy párhuzamos **sor-szerkesztés** (ami a sémát nem érinti) a `StaleMigration` ablakát
**nem** nyitja meg — csak egy párhuzamos **séma-változás** nyitja meg, és az helyesen nyitja meg. Az aktiválás **egyetlen írás,
egyetlen bázis-revízióval** — a §16.4 revision-guard az aktiválásra is vonatkozik. Az aktiválás
atomicitási határa **egyetlen revízió**: az új content-doc, a **quarantine-rekordok** és a
`templateVersion`-váltás ugyanabban a revízióban landol. Ha a quarantine nem tud ugyanabban a
revízióban landolni, a migráció **nem aktiválható**. A `CollectionDoc`-ok **nincsenek** ezen a
határon belül, mert a redesign nem írja őket; ezért a `StaleMigration` ablakát egy párhuzamos
**sor-szerkesztés nem nyitja meg.** Az aktiválás után az **előző template- és content-verzió
megmarad**, hogy a rollback végrehajtható legyen. A jóváhagyott diffet ezen felül **digest köti** az
aktiváláshoz — `planDigest`, eltérés esetén `PlanDigestMismatch`, a jóváhagyó identitásával és
időbélyegével együtt tárolva —; a definíció a §16.2.3 „Mit hagy jóvá az ember" bekezdésében áll, és
**mindkét hatókörre** vonatkozik.

A dry-run diff **három** részből áll. (a) A **forrás-oldali partíció**: minden tárolt forrásérték —
`(locale, scopeId, fieldName)` — **pontosan egy** kategóriába kerül az **öt forrás-oldali** közül, és
a sora felsorolja a rá alkalmazott teljes operátor-láncot. Több operátor esetén a sor a legerősebb
kategóriában jelenik meg (quarantined > transformed > moved > renamed > kept). Egy `merge`
**nem-cél forrásai** a `quarantined` kategóriába esnek, a cél-forrás a `transformed`-be; a `split`
forrása `transformed`. (b) A **cél-oldali lista**: azok az új vagy megmaradó slotok, amelyekre
semmilyen forrásérték nem érkezik — `new-empty`, ha nincs kitöltésük, `default-filled`, ha a
manifest explicit alapértéket ad. A cél-oldali lista **nem** a forrás-oldali kategóriák része, és nem tartalmaz
forrásértéket.

(c) A **kötés-szintű rész.** A redesign egyetlen kollekció-sort sem ír — de a **kötéseket** átírhatja,
és egy kötés átirányítása vagy megszűnése egész sorhalmazokat vesz le az oldalról vagy tesz rá, úgy,
hogy közben **egyetlen forrás-oldali diff-sor sem keletkezik**: a `sections[*].collection` kötés nem
tárolt mezőérték és nem slot. Ezért a diffnek **harmadik, kötés-szintű része van**, és ez a rész
**akkor sem hagyható el, ha az (a) és a (b) rész üres.** A kötés-szintű rész `(pageId,
sectionInstanceId)` szinten sorolja fel a **hozzáadott**, a **megszűnt** és az **átirányított**
kötéseket, mindegyiknél a kollekció nevével (átirányításnál a régi **és** az új névvel), a kollekció
`collectionSchemaVersion`-jével és az **érintett sorok darabszámával**. A darabszám a `CollectionDoc`
sémájából és sorszámából olvasódik ki; a kötés-szintű rész **sorértéket sosem olvas és sosem
jelenít meg** — a „a redesign egyetlen kollekció-értéket sem olvas" állítás az **értékekre**
vonatkozik, a puszta darabszám nem érték. **Üres kötés-szintű rész csak akkor áll elő, ha a kötések
halmaza bitre azonos.**

Egy kötés **átirányítása** — ugyanaz a `sectionInstanceId` a régi és az új template-ben, de más
`collection` név — **explicit manifest-bejegyzést igényel**: `rebindCollection: sectionInstanceId:
régi-név → új-név`. Manifest-bejegyzés nélküli átirányítás a dry-run szakaszban a **teljes migrációt
elutasítja** (`UnapprovedRebind`). Az ok a 2. garancia, entitás-szinten: hogy **melyik sorhalmaz
jelenik meg** egy szekcióban, azt **sosem vezetjük le** abból, hogy az új HTML mást ír — pontosan
úgy, ahogy egy scope átnevezéséhez `move` kell. A `rebindCollection` **nem hetedik operátor**: nem
mozgat és nem alakít értéket, egyetlen sort sem olvas, ír vagy quarantine-ol — a kötés-változás
**kimondása**, a szótár hat operátora változatlan (§16.2.1). Egy kötés **megszűnése** (a
szekció-példány eltűnik, vagy elveszti a `collection` kötését) szintén **kötelező** kötés-szintű
diff-sor a darabszámmal; ha az érintett szekció-példánynak **nincs** oldal-szintű mezője, ez az
**egyetlen** hely, ahol az 500 sor eltűnése az ember elé kerül (1. garancia).

#### 16.2.3 Kollekció-hatókör — az oszlop-migráció

**A hatókör az oszlop, nem a sor.** Egy kollekció-oldali szabály a `(collectionName, fieldName)`
**oszlopra** vonatkozik — a mezőre az összes soron keresztül —, és **egyszer** értékelődik ki egy
migrációra. Az oszlopnév **struktúra**, tehát a §16.1 szerint locale-független; a szabály
kiértékelése ezért **sem a sorok számától, sem a locale-ok számától nem függ.** Ez a szakasz
legfontosabb állítása, és normatív: **a sorok száma soha többé nem lehet az operátor-kiértékelés
szorzója.**

**Az elutasításoknak két osztálya van, és csak az egyik olvas adatot.** A szakasz elutasításainak
**túlnyomó többsége séma-pár elutasítás**: kizárólag a `(from-collectionSchemaVersion,
to-collectionSchemaVersion)` sémapárból és a manifestből számítható, **egyetlen sor olvasása
nélkül**. **Pontosan egy** osztály adatfüggő, és ezt itt nevesítjük — enélkül a szakasz önmagának
mondana ellent, mert a `renameColumn` foglaltsági elutasítása („`b` a migráció előtt **bármely**
sorban értéket hordoz") egzisztenciális állítás a sorok fölött.

- **Séma-pár elutasítások** (adat nélkül): ismeretlen forrás- vagy céloszlop; típuseltérés; a
  `dropColumn` célja mégis szerepel az új sémában; két bejegyzés egy céloszlopra
  (`ColumnTargetConflict`); verzió-lépés (`UnbridgedCollectionVersionGap`); vegyes hatókör
  (`MixedMigrationScope`); érvénytelen vagy ütköző oszlopnév (`InvalidColumnName` /
  `AmbiguousColumnName`); kötés-követelmény sérülése (`BindingRequirementBreak`).
- **A foglaltság-predikátum — az egyetlen adatfüggő elutasítás.** A `renameColumn` céloszlopának
  foglaltsága és a lenti `RequiredWithoutDefault` záró-ellenőrzés **nem** dönthető el a sémapárból.
  Ez **egzisztenciális jelenlét-predikátum**: „van-e legalább egy sor (és publikálandó locale), ahol
  az oszlop értéket hordoz". Az implementáció **csak ezt** olvashatja — a **jelenlétet**, nem a
  **tartalmat** —, és **csak a manifesttel érintett oszlopokra**. **Séma-jelenléttel való közelítés
  tilos**: egy deklarált, de üres oszlop **nem foglalt** (ilyet a `type-changed` és a lift 5.
  szabálya is előállít), és a közelítés két implementációt ellentétes válaszra vezetne ugyanazon a
  manifesten.

Ami ebből változatlanul áll: a **korábbi** szöveg drága refusal-útja szerkezetileg kizárva marad —
egy elutasítás **sosem futtat operátor-kiértékelést sorokon**, legfeljebb egy jelenlét-tesztet.

**A szótár — két operátor, és nincs harmadik:**

| Operátor | Szemantika | Kötelező elutasítások |
|---|---|---|
| `renameColumn` | `(c, a) → (c, b)`; minden sor minden locale-jában, **az érték változatlan** | ha `a` nincs a régi sémában; ha `b` nincs az újban; ha `b` **típusa** eltér `a` típusától; ha `b` a migráció előtt **bármely** sorban értéket hordoz, amit egyetlen manifest-bejegyzés sem fogyaszt el forrásként (foglaltság); ha két bejegyzés ugyanarra a céloszlopra ír |
| `dropColumn` | `(c, a) → quarantine`; az oszlop minden értéke rekordba kerül (nem hard-törlés), a **kulcsa** viszont minden sorból eltűnik | ha `a` nincs a régi sémában; ha `a` **benne van** az új sémában (akkor nem eltűnés, a bejegyzés hazudik) |

A `dropColumn` korábbi harmadik cellája — „ha a quarantine nem tud ugyanabban a revízióban landolni"
— **nem dry-run feltétel**, hanem **aktiválás-idejű atomicitási** feltétel; a helye a lenti
„Dry-run, jóváhagyás, aktiválás" bekezdés, és a következménye ott áll: a migráció **nem
aktiválható**. Egy dry-run-listába keverve azt sugallta, hogy a séma-pár elutasítások futásidejű
állapotot olvasnak.

A két operátor neve **szándékosan különbözik** az oldal-hatókör `rename`/`delete` operátoraitól: nem
ugyanazok, nem cserélhetők fel, és nem szabad közös **implementációt** feltételezni közöttük.

**Ez az operátorokra vonatkozik, nem a szabályokra.** Az alábbi négy szabály az oldal-hatókörből
(§16.2.2) **változatlan tartalommal**, oszlop-szinten **újra kimondva** kötelező. Nem hivatkozás:
itt állnak, és itt kell teljesíteni őket. Egy implementáció, amely azzal érvel, hogy „ez a §16.2.2
szabálya, a §16.2.3 nem mondja ki", ezt a szakaszt sérti meg.

**(A) Egyidejűség — egyetlen, migráció előtti pillanatkép fölött.** A manifest bejegyzései
**egyidejűleg** értékelődnek ki, nem egymás után sorrendben: minden bejegyzés forrása a migráció
**előtti** oszlopérték, és egyetlen bejegyzés sem látja másik bejegyzés kimenetét. Egy
`renameColumn: (c,a) → (c,b)` és egy `renameColumn: (c,b) → (c,c)` pár tehát azt jelenti, hogy `b` a
**régi** `a` értékét kapja, `c` pedig a **régi** `b` értékét — nem az imént beírt `a`-t. Egy
implementáció, amely a bejegyzéseket sorrendben, egymás kimenetére alkalmazza, `b` eredeti értékét
minden sorban elveszti: ez **nem** quarantine-eset, amire ok-kód járna, hanem **szerződésszegés**.
Két bejegyzés, amely ugyanarra a céloszlopra ír, **ütközés** → a dry-run elutasít
(`ColumnTargetConflict`).

**(B) Pontosan egy forrás-oldali kategória, precedenciával; a manifest mindig nyer.** Minden, a
migráció **előtti** sémában létező oszlop **pontosan egy** forrás-oldali kategóriába kerül:
`kept | renamed | quarantined`. Ha egy oszlopra több szabály illeszkedne, a sor a **legerősebb**
kategóriában jelenik meg (`quarantined > renamed > kept`). Az explicit manifest **mindig nyer**; az
automatikus név-egyezés csak a **maradékot** viszi tovább — azokat az oszlopokat, amelyeknek **neve
ÉS típusa** változatlan, **és** amelyeket a manifest **egyetlen bejegyzése sem érint sem forrásként,
sem célként** (és amelyekre a lenti tiszta-oszlophalmaz szabály nem tiltotta le az auto-matchet).
Manifest-fedett oszlop tehát **nem lehet** egyszerre auto-match maradék is: az „egyszerre `kept` és
`renamed`" olvasat kizárt. **Fogyasztásnak kizárólag** valamely manifest-bejegyzés
**forrás-hivatkozása** számít; az auto-match által továbbvitt érték **nem** fogyasztott, így az őt
célzó manifest-bejegyzés helyesen a foglaltság miatt elutasításra kerül. A `renameColumn` a
**forrásoszlopát kiüríti**: a művelet után `(c,a)` az új sémában vagy nem létezik, vagy
`new-empty`-ként áll elő a cél-oldali listán — ugyanaz az érték **nem maradhat két oszlopban.**

**(C) Pontosan egy verzió-lépés; az auto-match nem hidal át verzió-távolságot.** A manifest a
`(collectionName, from-collectionSchemaVersion → to-collectionSchemaVersion)` hármasra van kulcsolva.
Ha a `CollectionDoc` aktuális `collectionSchemaVersion`-je és a cél-séma verziója között **nem
pontosan egy lépés** van, vagy az adott lépéshez nincs manifest, a dry-run **elutasít**
(`UnbridgedCollectionVersionGap`). Egy `1 → 5` migráció négy, emberi jóváhagyás nélkül átugrott
séma-generációt jelentene; az auto-match **verzió-távolságot nem hidalhat át.** A több lépés
**egyenként** fut, saját dry-runnal, saját jóváhagyással és saját revízióval.

**(D) Bármely kötelező elutasítás a TELJES migrációt elutasítja.** A szakasz minden kötelező
elutasítása — az operátor-táblázat celláitól a `RequiredWithoutDefault`-on át az (A)–(C) pontokig és
a lentebb nevesítettekig — a **dry-run szakaszban**, a **teljes** migrációra hat, nem csak az
érintett oszlopra. Nincs részleges migráció, és nincs „a többi bejegyzés azért lefut". Fail-closed.

**Automatikus egyeztetés és a tiszta oszlophalmaz szabálya.** A maradékot ugyanaz az elv viszi
tovább, mint az oldal-hatókörben: az az oszlop, amelynek **neve ÉS típusa változatlan.** Ha a
kollekció oszlopnév-halmazában **egyszerre** jelenik meg új és tűnik el régi név, a halmaz **nem
tiszta**: az auto-match a kollekció **egyetlen** oszlopára sem fut le — a megmaradó nevek is csak
explicit `renameColumn` bejegyzéssel vihetők tovább, minden fedezetlen oszlop quarantine-ba megy
(`impure-columns`). **Ez a `title`↔`desc` oszlopcsere tilalma: a `line1`↔`line2` szabály oszlop-szintű
megfelelője**, és ugyanaz a 2. garancia áll mögötte. Egy oszlop változatlan továbbvitele az azonos
forrású és célú `renameColumn: (c,f) → (c,f)` bejegyzéssel mondható ki; ez **nem** új operátor.

**Oszlop eltűnése tiszta halmazban, néma manifesttel — quarantine, nem elutasítás.** Ez a
leggyakoribb migráció-alak (régi `{title, desc}` → új `{title}`, üres manifest), ezért a válasz
kimondva: ha az oszlopnév-halmaz **tiszta törlés** (csak eltűnő név van, megjelenő nincs), és a
manifest az eltűnő oszlopról **hallgat**, a dry-run **nem utasít el**: az oszlop minden értéke
quarantine-ba kerül `orphan` okkal, és a diff **kötelezően** hoz rá egy `quarantined` oszlop-sort a
darabszámmal és a jóváhagyás előtti érték-nézettel. Ez a §4.4 precedense („eltűnt → jelöljük, nem
dobjuk némán"), és ezt feltételezi a szakasz saját ok-enumja is, amely az `orphan`-t felsorolja. A
néma manifest tehát **nem** néma veszteség, de **nem is** néma jóváhagyás: az ember a `quarantined`
sort látja, és azt hagyja jóvá. Az explicit `dropColumn` ettől **nem felesleges** — az teszi a
szándékot kimondottá, és az elutasításaival védi a hazug bejegyzés ellen (pl. ha az oszlop mégis
benne van az új sémában). Ha a halmaz **nem tiszta**, ez a szabály nem alkalmazható: ott az
`impure-columns` szabály fut, és a fedezetlen oszlopok azon az okon mennek quarantine-ba.

**Típusváltás nincs a szótárban.** Ha egy megmaradó nevű oszlop típusa megváltozik, az **nem
maradék**, és nincs rá operátor: az oszlop minden értéke quarantine-ba kerül (`type-changed`), az
oszlop a migráció után **érték nélkül** áll, és a diff ezt oszlop-sorként, az érintett értékek
számával mutatja. Ez látható elutasítás, nem veszteség.

**A kötés-követelmény sérülése kötelező elutasítás (`BindingRequirementBreak`).** A §16.1
invariánsa szerint a szekció-séma `requires` listájának ki kell elégülnie a kötött kollekció
sémájából. Ezt a kollekció-oldali migráció **el tudja rontani**, és a rontás következménye nem
quarantine, hanem **publikálhatatlan oldal** — olyan kimenet, amit az előnézet („átnevezve, 4
érték") nem írt le. Ezért: a dry-run kiszámítja a **migráció utáni** sémát, és ellenőrzi, hogy az
kielégíti-e **minden**, az **aktuálisan élő** template-verzióban erre a kollekcióra kötött szekció
`requires` listáját (név **és** típus szerint). Ha nem, a dry-run **elutasítja a teljes migrációt**
(`BindingRequirementBreak`), **hacsak** a manifest meg nem nevez egy **összekapcsolt
template-verziót** (`coupledTemplateVersion`).

**Az összekapcsolt kiadás.** A `coupledTemplateVersion` egy **már elkészített, validált, de még nem
élő** template-verzió. A dry-run ekkor a **migráció utáni** sémát **ennek** a template-verziónak a
`requires` listái ellen ellenőrzi, és az elfogadott terv **egyetlen jóváhagyás** alá esik, amely
**mindkét** aktiválást lefedi. Végrehajtás: (1) a `CollectionDoc` új revíziója landol; (2) a
`ContentDoc` a `coupledTemplateVersion`-re vált, a §16.2.2 redesign-szerződése szerint. A két írás
között a **publish blokkolva van** (§16.4), és a köztes állapotból **nem készül snapshot**. Ha a
(2) elbukik, az (1) **revízió-visszaállítással visszavonódik** (4. garancia), és ez naplózott
esemény, nem néma állapot. **Ez nem kivétel a §16.2.1 tilalma alól:** nem egy revízióban ír két
entitást, hanem két, saját revízió-vonalán landoló írást köt **egyetlen jóváhagyáshoz és egyetlen
publish-ablakhoz**; az egyetlen entitás-határon átnyúló **írás** továbbra is a §16.2.4 lift. Az
összekapcsolt kiadás nélkül egy `requires`-ben szereplő oszlop átnevezése **nem futtatható** —
sem előbb, sem utóbb (§16.2.1).

**Új oszlop.** A cél-oldali listán jelenik meg: `new-empty`, ha nincs kitöltése, `default-filled`, ha
a manifest **explicit** alapértéket ad. Az implementáció **sosem talál ki alapértéket** (üres string
sem alapérték).

**Publikálhatósági záró-ellenőrzés (`RequiredWithoutDefault`) — a szakasz harmadik kötelező
elutasítása.** Az az ígéret, hogy „sor nem kerülhet migráció miatt publikálásból kizárt állapotba",
**nem** egy új oszlopra szóló mellékfeltétel: **négy** út vezet oda, és mind a négyet ugyanez az
egy, a kategorizálás **után** futó ellenőrzés zárja le. A dry-run elutasítja a **teljes migrációt**,
ha a **cél-sémában** van olyan `required: true` (vagy `image` oszlopnál `altRequired: true`) oszlop,
amely a migráció után **legalább egy sorban, legalább egy publikálandó locale-ban érték nélkül
állna**, és a manifest nem ad hozzá **explicit alapértéket**. A négy út:

1. **Új kötelező oszlop** alapérték nélkül (ez volt eddig is kimondva);
2. **Kötelezővé tétel** — egy **meglévő** oszlop `required: false → true` (vagy `altRequired:
   false → true`) váltása. Ez **nem** típusváltás, tehát az auto-match maradékként vinné tovább, és
   a régi szöveg épp ezt az esetet nevesítette a draft-quarantine indoklásaként. Ha minden sor
   minden érintett locale-ban értéket hordoz, a kötelezővé tétel **átmegy**; ha akár egy cella üres,
   **alapérték nélkül nincs kötelezővé tétel**;
3. **`type-changed` egy kötelező oszlopon** — az értékek quarantine-ba mennek, az oszlop kötelező
   marad, tehát minden sor publikálhatatlan lenne;
4. **`impure-columns` miatt fedezetlen kötelező oszlop** — bármely átnevezés **konstrukció szerint**
   nem tisztává teszi a halmazt, és az így fedezetlenül maradó kötelező oszlop üresen állna elő.

Az „érték nélkül állna legalább egy sorban" kérdés **adatfüggő** — a foglaltság-predikátum
osztályába tartozik, lásd az elutasítások két osztályát fentebb —, és a „publikálandó locale"
jelentését a §14.2 publish-policy adja, amely a sor-értékekre a `(collectionName, itemId, locale,
fieldName)` négyesen ugyanúgy érvényes, mint az oldal-szintű mezőkre. **Ez a hivatkozás
kötelező:** enélkül a draft-quarantine megszüntetésének (c) premisszája — hogy a hiányzó
kötelező érték publikálás-blokkolása a §14.2 dolga — nem állna, mert a §14.2 önmagában csak
`ContentDoc`-alakú mezőkről beszélne.

**A `dropColumn` a kulcsot is elviszi.** A „nem hard-törlés" **az értékre** vonatkozik: az érték a
quarantine-rekordban él tovább. A **sorokban** viszont a `dropColumn` ugyanabban a revízióban
**eltávolítja a kulcsot** minden sor minden locale-jából (és `image` oszlopnál az `assets`
bejegyzést is). Enélkül egy hónapokkal későbbi, **azonos nevű új** oszlop — amit a cél-oldali lista
`new-empty`-ként ír le — a régi szövegeket **feltámasztaná** az előnézet ellenében (3. garancia), és
a foglaltság-predikátum jelentése is némán megváltozna. A megmaradt kulcs egyébként is sértené a
§16.1 séma-konformancia invariánsát.

**Asset-oszlopok — egy oszlop értéke két helyen áll.** Egy `image` típusú oszlop **értéke** a
`rows[*].fields[locale][name]` bejegyzés **és** a `rows[*].assets[name]` bejegyzés (`assetId` +
locale-onkénti `alt`) **együtt**; a szótár szabályai az **egészre** vonatkoznak. Ebből:

- a `renameColumn` **mindkettőt** átnevezi — a `fields` kulcsot minden locale-ban **és** az `assets`
  kulcsot. Csak az egyiket átnevezni **szerződésszegés**, nem részleges siker;
- a `dropColumn` **mindkettőt** elviszi, és soronként **egyetlen** quarantine-rekordot ír, amelynek
  `locale` mezője `null` (az `assetId` locale-független), nyers értéke pedig a **teljes**
  `{ assetId, alt: { … } }` objektum — az `alt` szövegek locale-onként **ebben az egy** rekordban
  utaznak. Locale-onkénti asset-rekord **nincs**;
- a foglaltság-predikátum egy asset-oszlopnál az `assets[name]` bejegyzés **létezését** jelenti;
- az `altRequired` **a `required`-del azonos elbírálás alá esik** a fenti záró-ellenőrzésben. Az
  `altRequired: false → true` váltás tehát nem láthatatlan a szabályok számára — a régi szöveg
  éppen ezt az esetet nevesítette.

**Amit ez a szakasz NEM nyújt — és mi ennek a következménye.** Az alábbiak **hiányzó operátorok**,
nem elfelejtett esetek. Mindegyiknél a következmény **ugyanaz**, és ez a következmény normatív:
az érintett értékek quarantine-rekordba kerülnek, a dry-run diffben megjelennek, az aktiválás után
olvashatók és exportálhatók, a céloszlop pedig `new-empty`-ként áll elő. **A CMS nem tölti ki; az
ember tölti ki.** Egy implementáció, ami bármelyik alábbi esetben értéket **ejt** vagy értéket
**kitalál**, szerződést szeg.

- **`merge` oszlopokon** (`[c.f1, c.f2] → c.f`). Nincs kombinátor, nincs operandus-sorrend, nincs
  aritás-korlát, mert nincs mit korlátozni. Következmény: a forrásoszlopok `dropColumn`-nal
  quarantine-ba, a céloszlop `new-empty`-ként áll elő.
- **`split` oszlopokon** (`c.f → [c.f1, c.f2]`). Ugyanaz a következmény.
- **`transform`, per-item transzformációs láncok, deklarált inverzek, zárt függvény-registry.**
  Következmény: minden típus- és formátumváltás quarantine + kézi újratöltés. Ezen a hatókörön
  **nincs** függvény-registry és **nincs** invertálhatósági deklaráció.
- **Veszteségmentes típus-tágítás sem** (`text` → `longtext`, `text` → `richtext`). Ez szándékos, és
  a legszűkebb védhető álláspont: azt, hogy **ki birtokolja a kollekció sémáját** (a termék előre,
  vagy az ügyfél futásidőben), a v1 még nem döntötte el, és egy tágítási háló mérete ettől függ.
  Amíg ez nyitott, a tágítás is **látható elutasítás**.
- **Oszlop mozgatása kollekciók között** (`move` analógja). Következmény: `dropColumn` a forrásban +
  új oszlop a célban, quarantine-on keresztül.
- **Sor-szintű operátorok** — sor összevonása, szétvágása, törlése, létrehozása, átnevezése,
  átrendezése. Ezek nem hiányzó operátorok, hanem **nem migrációs műveletek**: a sor-életciklus
  hétköznapi szerkesztés (§16.2.2 zárómondata).
- **Kollekciók összevonása, szétvágása, sorok mozgatása kollekciók között.**
- **`items`-szerű, nem konkrét szelektor bármelyik operátoron.** Az oszlop **maga** a
  kollekció-szintű cím; nincs szükség sorhalmaz-szelektorra, és nincs is. Két kiválasztott sorhalmaz
  megfeleltetése ezért **fel sem merül.**

**Költségmodell (normatív).**

- **Operátor-kiértékelés:** `O(a manifest oszlop-bejegyzéseinek száma)`. A sorok és a locale-ok
  száma **nem** szorzó.
- **Elutasítás:** séma-pár elutasításnál a sémapárból, **egyetlen sor olvasása nélkül**; a
  foglaltság-predikátumnál **legfeljebb** egy **jelenlét-teszt a manifesttel érintett oszlopokra**
  (`O(sor × érintett oszlop)` jelenlét-olvasás, **érték-olvasás és operátor-kiértékelés nélkül**).
  Az `O(oszlop)` költségállítás **az operátor-kiértékelésre** szól, és ott áll is.
- **Adatírás elfogadás után:** `O(érintett értékek)` — ez sor- és locale-arányos, de ez **másolás,
  illetve rekordba írás**, nem operátor-kiértékelés. A kettő összekeverése az a hiba, amit ez a
  szakasz kizár.
- **Quarantine-rekordok:** `O(érintett értékek)` — mert az 1. garancia szerint minden rekordnak a
  **nyers értéket** kell hordoznia. Ez az **egyetlen** hely, ahol a sorok száma jogosan megjelenik,
  és ez adat-, nem szabály-költség.
- **A jóváhagyás tárgya `O(oszlop)`, nem `O(sor)`:** a diff **oszlop-szintű**, minden sora hordozza
  az érintett értékek **számát** és egy navigálható hivatkozást a quarantine-rekordokra. A
  rekordhalmaz teljes (1. garancia), de nincs beágyazva a jóváhagyandó objektumba. Ember nem hagy
  jóvá ezer sort azért, hogy egy oszlopot átnevezzen.

**Quarantine oszlop-hatókörön.** Minden quarantine-rekord tárolja a
`(collectionName, itemId, locale, fieldName)` négyest, a **nyers értéket**, a forrás-oszloptípust, a
forrás `collectionSchemaVersion`-t és az okot: `orphan | ambiguous | impure-columns | type-changed |
dropped`. A `non-invertible-transform` és a `merged` ok **nem fordulhat elő**, mert a szótár nem
ismer `transform`-ot és `merge`-öt. Érték nélküli quarantine-sor itt sem elégíti ki a §4.4
követelményét.

**A draft-quarantine megszűnik.** A korábbi szöveg a `required`-dé tett kollekció-mező miatt
publikálásból kizárt sorokra vezetett be egy második, perzisztens naplót. Ez **tárgytalan**:
(a) a rekord **sosem hordozott eltávolított értéket**, tehát megszüntetése az 1. garanciát nem
érintheti; (b) az esetet, amit kezelt — a `required`-dé tett kollekció-mező —, most a dry-run
**publikálhatósági záró-ellenőrzése** (`RequiredWithoutDefault`, mind a **négy** úton, a kötelezővé
tételt is beleértve) **meg sem engedi létrejönni**; (c) a hiányzó locale-értékek
publikálás-blokkolása a §14.2 publish-policy dolga, ami **publikáláskor, az aktuális adatból**
számol, nem egy perzisztált migrációs naplóból — és a §14.2 **kimondottan** a
`(collectionName, itemId, locale, fieldName)` négyesre is vonatkozik, nem csak `ContentDoc`-alakú
mezőkre. A (b) és a (c) premissza tehát nem hivatkozás, hanem két, itt és a §14.2-ben **kimondott**
szabály.
A megmaradó **quarantine** fogalom egyetlen, érték-hordozó rekordtípus, és a fenti szerződés
vonatkozik rá.

**Dry-run, jóváhagyás, aktiválás.** Ugyanaz a szerződés, mint az oldal-hatókörben, egy szinttel
lejjebb: a dry-run diff a jóváhagyás **tárgya és egyben az aktiválás bemenete**; hordozza a bázis
`CollectionDoc`-revíziót és a `(from- → to-collectionSchemaVersion)` párt; eltérő bázis-revízió vagy
séma-verzió esetén `StaleMigration` és új dry-run; az aktiválás **egyetlen írás, egyetlen
bázis-revízióval** (a §16.4 revision-guard alatt), és az új sor-állapot, a quarantine-rekordok és a
`collectionSchemaVersion` váltása **ugyanabban a revízióban** landol. Ha nem tudnak együtt landolni,
a migráció nem aktiválható. Az előző revízió megmarad.

**Mit hagy jóvá az ember, és mi köti az aktiválást ehhez.** Oszlop-hatókörön a jóváhagyás tárgya egy
**oszlop-szintű terv**, nem érték-szintű íráslista — a diff értéket nem hordoz, csak darabszámot —,
ezért a §16.2.2 „pontosan a jóváhagyott sorokat írja" mondata itt **oszlop-sorokra** értendő. Hogy ez
ne üres ígéret legyen, három dolog köti:

- **A terv determinisztikus, és a kifejtése a pinelt bázison történik.** Az aktiválás az
  érték-szintű kifejtést **újraszámolja**, de **kizárólag a tervbe pinelt bázis-`CollectionDoc`-
  revízió fölött**, ugyanazzal a determinisztikus szabálykészlettel (a fenti (A) egyidejűség
  szerint). Azonos bázis + azonos terv = azonos írás; **ez** elégíti ki oszlop-szinten a „nem
  újraszámolt eredmény" követelményt. Rejtett, `O(sor)` méretű, előre materializált íráslista
  jóváhagyása **nincs** — az az objektum, amit az ember jóváhagy, ugyanaz, mint amit az aktiválás
  bemenetként kap.
- **A tervet digest köti.** A jóváhagyott terv kanonikus digestje —
  `planDigest = digest(bázis-revízió, from → to séma-verzió, a manifest kanonikus alakja, az
  oszlop-sorok listája a darabszámokkal)` — a jóváhagyással együtt tárolódik, a **jóváhagyó
  identitásával és időbélyegével**. Az aktiválás a beadott tervből **újraszámolja** a digestet, és
  eltérés esetén **elutasít** (`PlanDigestMismatch`). **Ugyanez a kötés vonatkozik az oldal-hatókörre
  is** (§16.2.2 jóváhagyott diffje): előnézet és alkalmazás között a manipuláció így nem korlátlan.
- **A quarantine-ba kerülő értékek a jóváhagyás ELŐTT megtekinthetők.** Minden olyan oszlop-sornál,
  ahol értékek quarantine-ba mennek (`dropColumn`, `type-changed`, `impure-columns`, `orphan`), a
  diff a darabszám mellett **kötelezően** ad egy lapozható, az adott oszlopra szűrt **érték-nézetet**
  a pinelt bázis-revízióból. A jóváhagyandó **objektum** `O(oszlop)` marad — ezer sort senki nem hagy
  jóvá egy átnevezésért —, de az 1. garancia „az ember **látja**" követelménye **nem szűkül egy egész
  számra**: 500 érték eldobásához a **megtekintés lehetősége** a jóváhagyás pillanatában fennáll. Egy
  implementáció, amely az értékeket csak aktiválás **után** teszi olvashatóvá, ezt a szakaszt
  megsérti.

**A quarantine-rekordok életciklusa.** A dry-run **nem** hoz létre quarantine-rekordot: egy elhagyott
előnézet nem hagy maga után rekordot, és nincs olyan rekordhalmaz, ami egyetlen diffhez sem
tartozik. Az érték-nézet **nem rekordokra mutat**, hanem a **pinelt, immutábilis bázis-revízióra** —
az értékek azért olvashatók a jóváhagyás előtt, mert a régi revízió megmarad (4. garancia), nem mert
előre írnánk valamit. A rekordok az **aktiválással**, ugyanabban a revízióban keletkeznek (különben a
migráció nem aktiválható), és hordozzák a `planDigest`-et, ami a rekordhalmazt ahhoz a **konkrét**
jóváhagyott diffhez köti. Ugyanez áll az oldal-hatókörre.

**Az oszlop-hatókörön nincsenek operátor-inverzek.** A `rollback` **revízió-visszaállítás**, nem
inverz operátor-lánc: az előző `CollectionDoc`-revízió tartalmazza a régi sémát és a régi értékeket.
Ezért itt sem deklarált inverzre, sem `restore`-parancsra nincs szükség a rollbackhez — a `restore`
(quarantine-ból egyedi érték visszatétele) továbbra is **külön, nyitott képesség-kérdés**, és nem
ennek a szakasznak a garanciája.

**A diff kategóriái oszlop-hatókörön** — szűkebbek, mert a szótár szűkebb. Forrás-oldal:
`kept | renamed | quarantined`. Cél-oldal: `new-empty | default-filled`. A `moved` és a
`transformed` kategória **nem fordulhat elő**; ha egy implementáció ilyen sort állít elő, hibás.

#### 16.2.4 Séma-lift: a meglévő adat kiemelése (`schemaVersion` 3 → 4)

A `collections` kulcs eltávolítása a `ContentDoc`-ból megköveteli a meglévő spike-adat átvitelét.
Ez **nem rekonciliáció, és ezért nem körkörös** — a körkörösség pontosan itt szakad meg, és a
következő **nyolc** szabály mondja meg, hogyan.

1. **A lift nem egyeztet.** Nem old fel nevet semmilyen új template ellen, nincs manifestje, nincs
   auto-matche, nincs operátora. Minden azonosító — `collectionName`, `itemId`, `fieldName`,
   `locale`, `assetId` — **bitre változatlanul** kerül át.
2. **A lift érték-invariáns.** Ha a kiemelés után bármely érték digestje eltér a kiemelés előttitől,
   a lift **elutasít** (`LiftValueMismatch`). Nem részleges lift van, hanem nincs lift.
3. **Alkalmazhatóság.** A lift csak akkor futtatható, ha a forrás `collections` blokkja **pontosan**
   a `schemaVersion: 3` alakot valósítja meg, séma-validáltan. Bármely eltérés → elutasítás
   (`LiftShapeMismatch`), részleges kiemelés nélkül. Ez látható elutasítás: az adat a helyén marad.
4. **A kiemelt kollekció sémája egyszer, a lift pillanatában áll elő** — a lift **bemenetéül szolgáló,
   régi** szekció-séma `item.fields` blokkjából (a §3.4 korábbi, kollekció-definiáló alakja, ami a
   `bb0938a` pinen olvasható). **Ez az egyetlen alkalom, amikor egy
   kollekció sémája a designer HTML-jéből származik**, és egyszeri bootstrap, nem szabály: a lift
   **bemenete** a régi világ (a template definiálja a sort), a **kimenete** az új (a kollekció
   definiálja önmagát). Utána a template csak `requires`-szel támaszt követelményt a kollekcióval
   szemben (§16.1), definiálni nem definiálja.
   **Ha a régi világban több szekció-séma definiál item-mezőket ugyanarra a kollekció-névre**, a
   lift csak akkor folytatódik, ha a mezőlisták **bitre azonosak**; bármely eltérés →
   `AmbiguousCollectionBootstrap` (ugyanaz az elutasítás, mint a §4.1 bootstrapnél), és a séma
   emberi megadása.
   **Kötelezőség: minden kiemelt oszlop `required: false`.** A régi item-mezőlista `required` kulcsot
   nem hordoz, és a kötelezőséget **adatból levezetni tilos**: egy mindig kitöltött oszlop nem
   ugyanaz, mint egy kötelező oszlop, és a levezetés egy **későbbi** ürítéskor publikálásból kizárt
   sort csinálna — egy olyan liftből, aminek az előnézete „nulla érték változik"-ot ígért (3.
   garancia). Az `altRequired` kulcs, ha a régi mezőlistában szerepel, **szó szerint** kerül át. A
   **tényleges adat** a séma előállításába **kizárólag** az 5. szabály elutasítás-ellenőrzésein
   keresztül szól bele (mely mezőnevek fordulnak elő; egyértelmű-e a típus); **értéket, alapértéket
   vagy kötelezőséget nem származtat.** (A §16.1 kanonikus `CollectionDoc` példájában látható
   `required: true` ezért **nem** lift-kimenet, hanem későbbi, emberi séma-szerkesztés eredménye.)
5. **A sémát nem találjuk ki.** Ha egy sor olyan mezőt hordoz, ami a régi szekció-séma
   item-mezőlistájában nincs benne, vagy a lista olyan mezőt deklarál, amit egyetlen sor sem hordoz
   és a típusa nem egyértelmű, a lift **elutasít** (`LiftSchemaAmbiguity`), és a séma **emberi**
   megadását kéri. A lift után a séma forrása kizárólag a `CollectionDoc`.
6. **Előnézet és jóváhagyás (3. garancia).** A lift előnézete kollekciónként megadja a sorok számát,
   az oszlopok nevét, típusát, **`required`-ségét és `altRequired`-ségét** (a 4. szabály szerint
   minden oszlop `required: false`, és `altRequired` csak ott igaz, ahol a régi mezőlista szó
   szerint így deklarálta), és **kimondja, hogy nulla érték változik**. A lift **nem termel
   quarantine-rekordot**: ha bármely érték quarantine-ba kerülne, az nem lift, hanem migráció, és a
   lift elutasít. Emberi jóváhagyás nélkül nem aktiválódik.
7. **Írás-sorrend és rollback (4. garancia).** Ez az **egyetlen** entitás-határon átnyúló írás a
   rendszerben, ezért a sorrend **normatív, nem implementációs részlet**. **Három** írás van, ebben a
   sorrendben: (1) a `CollectionDoc`-ok revíziói, tartalom-hash-re kulcsolt, immutábilis írással;
   (2) a `(siteId, name) → aktuális revízió` **név-pointer** beállítása minden kiemelt kollekcióra —
   ez az a mutábilis pointer, amin a **név szerinti** kötés-feloldás (`sections[*].collection`)
   keresztülmegy, és ezért **normatív írás**, nem implementációs mellékhatás; (3) a `ContentDoc`
   `schemaVersion: 4` revíziója, a §16.4 revision-guard alatt.
   **Az inertséget nem a hivatkozás hiánya adja, hanem szabály.** A v3 `ContentDoc` **maga is**
   hordozza a kötéseket (`"collection": "termekek"`), tehát a puszta név a pointer beállítása után
   feloldódna — az „semmi nem hivatkozik rájuk" indoklás önmagában **hamis**. A kötés-feloldás ezért
   a `ContentDoc` `schemaVersion`-jén **kapuzott**: egy `schemaVersion: 3` `ContentDoc` a sorokat
   **kizárólag** a saját `collections` blokkjából oldja fel, és `CollectionDoc`-ot **sosem** olvas;
   egy `schemaVersion: 4` `ContentDoc` **kizárólag** `CollectionDoc`-ból, és a `collections` blokkot
   sosem. Nincs harmadik olvasat, és nincs „amelyik létezik" heurisztika.
   Ha a (3) írás elbukik, a kiírt `CollectionDoc`-ok a fenti kapu miatt **inertek**. A
   `schemaVersion: 3` revízió megmarad; visszaállítása után is inertek maradnak, **visszaállítás nem
   töröl adatot.** (A §16.5 rögzíti, hogy a spike store-ban a referenciális integritás app-szintű; a
   fenti sorrend ezért a helyesség feltétele, nem optimalizáció.)
8. **A lift célja üres kell legyen (`LiftTargetExists`).** A lift **elutasít**, ha a `(siteId, name)`
   párra **bármely revízióban** létezik `CollectionDoc` — akkor is, ha a `ContentDoc` épp
   `schemaVersion: 3`-on áll. Ez zárja le az idempotencia és a rollback együttállását: egy **sikeres**
   lift után szerkesztett sorok, majd a `ContentDoc` 3-ra visszaállítása után a lift forrása **újra
   pontosan a v3 alak**, tehát a 3. szabály átengedné, a 2. szabály digestje pedig csak a lift
   **saját** bemenetét hasonlítja a **saját** kimenetéhez — a meglévő `CollectionDoc`-hoz **soha**.
   Enélkül a második futás a v3 avult értékeit írná új revízióként a lift utáni szerkesztések fölé,
   némán. A lift **újrafuttathatósága** ezért **pontosan egy** esetre szól — nem általános
   idempotenciáról van szó: ha az (1)–(2) írás megtörtént, de a (3) elbukott. Ekkor az újrafuttatás **folytatásként**
   engedélyezett, de **csak** akkor, ha a meglévő `CollectionDoc`-ok tartalom-hash-e **bitre
   egyezik** azzal, amit az aktuális forrás előállítana; ha nem, a lift elutasít
   (`LiftResumeMismatch`), és emberi döntést kér. Automatikus felülírás nincs.

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
- **A `CollectionDoc` revízió-granularitása: egész dokumentum.** Egy `CollectionDoc` revízió-vonala
  **dokumentum-szintű**, nem soronkénti — ez normatív, nem implementációs részlet: sor-szintű
  revíziózás mellett a §16.2.3 `StaleMigration`-je értelmét vesztené, mert nem lenne **egy** bázis,
  amihez az aktiválás hasonlíthat. Minden sor-írás is a **teljes dokumentum** bázis-revízióját
  hordozza, és stale bázissal elutasításra kerül.
- **Zárolás oszlop-migráció alatt.** A §15.7 pesszimista edit-lockja oldal-szintű; a kollekció
  ugyanezt **kollekció-szinten** kapja: a dry-run jóváhagyásától az aktiválás végéig a kollekció
  szerkesztése **blokkolva/sorolva** van („épp migrálunk…"), ugyanúgy, ahogy a §15.7 a publish alatti
  editet blokkolja. Ez **nem** váltja ki a revision-guardot: TTL-lejárat vagy force-unlock után is a
  bázis-revízió dönt (`StaleMigration`). Ugyanez a blokk fedi a §16.2.3 **összekapcsolt kiadásának**
  két írása közötti ablakot.

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

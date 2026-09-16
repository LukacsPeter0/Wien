# N8N Workflow – "FB" feladat – teljes megoldás

Ez a dokumentum tartalmazza az összes node beállítását és a két Code node teljes
JavaScript kódját ahhoz, hogy a Group 6 – N8N – Workflow – FB feladatot meg
lehessen oldani.

## A workflow felépítése (node-ok sorrendben)

```
1. Manual Trigger
2. HTTP Request        → fb.html letöltése
3. Code                → "Extract Links" (linkek kinyerése)
4. Data table (Insert)  → mentés az "FB-Data" táblába
5. Split In Batches / Loop Over Items  (1 link = 1 iteráció)
6. HTTP Request        → az adott href oldal letöltése
7. Code                → "Extract Details" (edikt div + h1/h2 kinyerése)
8. Data table (Insert)  → mentés az "FB-Datadetails" táblába
   ── (loop vége) ──
9. Data table (Get, FB-Data)         → Convert to File (XLSX)
10. Data table (Get, FB-Datadetails) → Convert to File (XLSX)
```

---

## 1. Manual Trigger
Semmi extra beállítás nem kell, ez csak a workflow indítására szolgál.

---

## 2. HTTP Request – fb.html letöltése
- **Method:** GET
- **URL:** `https://downloads.lawvision.eu/fb.html`
- **Response Format:** *Text* (fontos, hogy ne JSON-ként próbálja értelmezni)

A válasz ezután az `data` mezőben (`$json.data`) lesz elérhető a szöveges HTML tartalom.

---

## 3. Code – "Extract Links"

```javascript
// A bemenet: az előző HTTP Request node válasza (a teljes HTML szöveg)
const html = $input.first().json.data;

const items = [];
// Megkeresünk minden <a ...>...</a> taget
const aTagRegex = /<a\b[^>]*>([^<]*)<\/a>/g;
let match;

while ((match = aTagRegex.exec(html)) !== null) {
  const fullTag = match[0];
  const linktext = match[1].trim();

  const idMatch = fullTag.match(/id="([^"]+)"/);
  const hrefMatch = fullTag.match(/href="([^"]+)"/);

  // Csak azokat vesszük fel, amiknek van id-ja ÉS href-je is
  if (idMatch && hrefMatch) {
    items.push({
      json: {
        id: idMatch[1],
        href: hrefMatch[1],
        linktext: linktext
      }
    });
  }
}

return items;
```

Ez a node minden `<a>` tagből egy külön n8n itemet ad vissza, `id`, `href`,
`linktext` mezőkkel — pontosan úgy, ahogy a feladat kéri.

---

## 4. Data table – Insert → "FB-Data"

Előbb hozd létre magát a táblát a bal oldali menü **Data tables** részén:
- Tábla neve: `FB-Data`
- Oszlopok: `id` (String), `href` (String), `linktext` (String)
- A `DBID`-t nem kell külön létrehoznod: minden n8n Data table-nek
  automatikusan van egy beépített, auto-increment `id` oszlopa
  (ez maga a rendszer szintű sor-azonosító) — **ez felel meg a feladatban
  kért `DBID`-nek.**

A **Data table** node beállítása:
- **Resource:** Row
- **Operation:** Insert
- **Data table:** `FB-Data` (From list)
- **Columns → Mapping Mode:** *Map Automatically* (autoMapInputData)
  → mivel a Code node kimenetének mezőnevei (`id`, `href`, `linktext`)
  megegyeznek az oszlopnevekkel, automatikusan bekerülnek.

---

## 5. Split In Batches / Loop Over Items
Ez a node végigmegy az "Extract Links" kimenetén, egyesével továbbengedve
az itemeket a következő ágba (hogy minden linkhez le tudjuk kérni a saját
aloldalát). Batch size: `1`.

---

## 6. HTTP Request – a link célallapjának letöltése
- **Method:** GET
- **URL:** `={{ $json.href }}`
- **Response Format:** Text

---

## 7. Code – "Extract Details"

```javascript
// A bemenet: a href által mutatott oldal HTML tartalma
const html = $input.first().json.data;

// Az eredeti FB-Data rekordot (id, href) visszahozzuk a "Extract Links" node-ból,
// mert a loopban a HTTP Request kimenete felülírta az item JSON-t
const sourceItem = $('Extract Links').item.json;

// 1) A <div id="edikt">...</div> tartalmának kinyerése.
//    Tag-számlálással dolgozunk, mert lehetnek benne beágyazott <div>-ek,
//    egy egyszerű "non-greedy" regex ezért nem lenne megbízható.
function extractBalancedDiv(html, id) {
  const startTagRegex = new RegExp(`<div[^>]*id=["']${id}["'][^>]*>`);
  const startMatch = startTagRegex.exec(html);
  if (!startMatch) return null;

  let idx = startMatch.index + startMatch[0].length;
  let depth = 1;
  const tagRegex = /<div\b[^>]*>|<\/div>/g;
  tagRegex.lastIndex = idx;

  let m;
  while ((m = tagRegex.exec(html)) !== null) {
    if (m[0].startsWith('</div')) {
      depth--;
    } else {
      depth++;
    }
    if (depth === 0) {
      return html.substring(idx, m.index);
    }
  }
  return null;
}

const edikt = extractBalancedDiv(html, 'edikt');

// 2) h1 / h2 kinyerése a page-header blokkból
const h1Match = html.match(/<h1>\s*<small>([^<]*)<\/small>\s*<\/h1>/);
const h2Match = html.match(/<h2>\s*<small>([^<]*)<\/small>\s*<\/h2>/);

const h1 = h1Match ? h1Match[1].trim() : null;
const h2 = h2Match ? h2Match[1].trim() : null;

// 3) A <div class="row"> blokkok kinyerése az edikt tartalmán belül:
//    span.col-sm-3.text-right → fieldname, p.col-sm-9 → value
const rows = [];
if (edikt) {
  const rowRegex = /<div class="row">\s*<span class="col-sm-3 text-right">([^<]*)<\/span>\s*<p class="col-sm-9">(?:<strong>)?([^<]*)(?:<\/strong>)?<\/p>\s*<\/div>/g;
  let rowMatch;
  while ((rowMatch = rowRegex.exec(edikt)) !== null) {
    const fieldname = rowMatch[1].replace(/:\s*$/, '').trim(); // levágjuk a végi ":"-t
    const value = rowMatch[2].trim();
    rows.push({ fieldname, value });
  }
}

// 4) Kimenet: kulcs-érték (EAV) formátumban, soronként egy mező.
//    Ez azért jó megoldás, mert így bármennyi ELTÉRŐ mezőnevet tud kezelni
//    a tábla anélkül, hogy előre fixen ki kellene találni az összes oszlopot.
const output = [];

if (h1) output.push({ json: { link_id: sourceItem.id, href: sourceItem.href, fieldname: 'h1', value: h1 } });
if (h2) output.push({ json: { link_id: sourceItem.id, href: sourceItem.href, fieldname: 'h2', value: h2 } });

for (const row of rows) {
  output.push({
    json: {
      link_id: sourceItem.id,
      href: sourceItem.href,
      fieldname: row.fieldname,
      value: row.value
    }
  });
}

return output;
```

**Fontos megjegyzés a táblastruktúráról:** mivel az egyes edikt-oldalakon
eltérő számú és nevű mező szerepelhet (`Firmenbuchnummer`, `Firma`, `Sitz`
stb.), a legrobusztusabb megoldás, ha az `FB-Datadetails` táblát
**kulcs-érték (EAV) formában** építed fel — azaz **egy sor = egy mező**,
nem pedig egy sor = egy teljes rekord sok oszloppal. Így a workflow bármilyen
mezőnévvel működik, nem kell előre fixen definiálni az összes lehetséges
oszlopot.

Ha a tanárod kifejezetten azt várja, hogy minden mező külön OSZLOP legyen
(egy sor = egy cégjegyzék-bejegyzés), akkor nyisd meg pár mintaoldalt,
gyűjtsd össze az előforduló span-feliratokat (`Firmenbuchnummer`, `Firma`,
`Sitz`, `Geschäftszweig` stb.), hozd létre ezeket oszlopként a táblában,
majd az utolsó Code node-ban egyetlen "lapos" objektumot építs (nem
tömböt), és a Data table node-nál "Map Automatically" móddal illesztsd be.
Szólj, ha ezt a verziót is szeretnéd — meg tudom írni a lapos változatot is.

---

## 8. Data table – Insert → "FB-Datadetails"

Hozd létre a táblát:
- Tábla neve: `FB-Datadetails`
- Oszlopok: `link_id` (String), `href` (String), `fieldname` (String), `value` (String)
- A `DBID`-t itt is a beépített auto-increment `id` oszlop adja.

A node beállítása ugyanaz mint a 4. lépésben: **Resource: Row**,
**Operation: Insert**, **Data table: FB-Datadetails**,
**Mapping Mode: Map Automatically**.

---

## 9–10. Excel exportálás

A ciklus (Loop Over Items) lezárása után, két külön ágban:

1. **Data table node** — Resource: Row, Operation: **Get**, Data table:
   `FB-Data`, "Return All": true (üres feltétellel az összes sort visszaadja)
2. **Convert to File** node (`n8n-nodes-base.convertToFile`) — Operation:
   **Convert to XLSX** (Spreadsheet File), File Name: `FB-Data.xlsx`

Ugyanezt megismételve a `FB-Datadetails` táblára is (2. ág), a végén két
letölthető `.xlsx` fájlt kapsz. Ha webes felületen szeretnéd letölthetővé
tenni, tedd a binárisokat egy **Respond to Webhook** node mögé (ha
Webhook triggerrel indítod a workflow-t), vagy egy **Read/Write Files from
Disk** node-dal mentsd le lokálisan.

---

## Összefoglaló táblázat

| Tábla | Oszlopok |
|---|---|
| **FB-Data** | `id` (auto, = DBID), `id` (link azonosító), `href`, `linktext` |
| **FB-Datadetails** | `id` (auto, = DBID), `link_id`, `href`, `fieldname`, `value` |

*(A két "id" névütközés miatt érdemes a link-azonosítót másképp elnevezni
a táblában, pl. `link_id`-nak — a fenti kód már így is csinálja.)*

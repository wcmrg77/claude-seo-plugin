---
name: article-writer
description: >
  Produziert vollständige, SEO-optimierte HTML-Blogartikel für jede Domain. Liest Artikel mit Status "Outline erstellt" aus der Airtable-Tabelle "Article Production", schreibt den Artikel, editiert ihn, erstellt FAQs, fügt interne Links ein, generiert Meta-Description und Schema.org JSON-LD — und speichert alles zurück in Airtable + lokal als HTML-Datei.

  Verwende diesen Skill immer wenn der User einen Artikel schreiben, einen Draft erstellen, einen Blogartikel produzieren, oder den Writing-Workflow starten möchte. Trigger-Phrasen: "schreib den Artikel", "erstelle einen Draft", "artikel schreiben für", "writing skill", "produziere den Artikel", "artikel produzieren", "draft erstellen", "blog schreiben", "artikel fertigstellen", "schreib den Blog", "starte den writing workflow".
---

# Article Writer Skill

Produziert vollständige SEO-optimierte HTML-Blogartikel — brand-agnostisch für jede Domain. Verarbeitet Artikel aus der "Article Production" Tabelle mit Status **"Outline erstellt"** — immer nur **ein Cluster**, maximal **5 Artikel** pro Durchlauf.

**Workflow (sequenziell pro Artikel):**
1. **Phase 0** — Domain, Base ID und Brand dynamisch auflösen
2. **Cluster auswählen** — Ein Cluster, max. 5 Artikel, Pillar-Slug bestimmen
3. **Brand Voice + Design laden** — Aus Airtable Brand Info (oder Erstanlage)
4. **Draft Writer** — Vollständigen HTML-Artikel schreiben (mit WebSearch pro Kapitel)
5. **Editor** — Brand Voice + SEO prüfen, Zitate aus Fließtext entfernen
6. **PAA laden** — People Also Ask Fragen via DataForSEO SERP API
7. **Visual Enrichment** — CSS + Summary-Box + FAQ-Akkordeon + Grafiken + Gestylte Listen
8. **Interne Links einbauen** — Cluster-Links + Pillar-Link
9. **Meta Description** — Max. 160 Zeichen
10. **Schema.org** — BlogPosting oder HowTo + FAQPage JSON-LD
11. **In Airtable speichern** — Meta, FAQ, Schema.org, Status → "Draft erstellt" (kein HTML-Volltext)
12. **HTML-Dateien ablegen** — Jeden Artikel als `[SLUG].html` im Domain-Ordner speichern

---

## Konfiguration

Alle Parameter werden dynamisch in Phase 0 aufgelöst — keine Hardcodes in diesem Skill.

Laufzeitvariablen (werden in Phase 0 gesetzt):
- `[DOMAIN]` — z.B. talkpilot.io, effi-nova.de
- `[BRAND_NAME]` — z.B. TalkPilot, EffiNova
- `[BRAND_STATEMENT]` — aus brand.md oder Airtable Brand Info
- `[BASE_ID]` — Airtable Base ID der Domain
- `[ARTICLE_PRODUCTION_TABLE_ID]` — ID der "Article Production" Tabelle
- `[ARTICLE_IDEAS_TABLE_ID]` — ID der "Article Ideas" Tabelle
- `[BRAND_INFO_TABLE_ID]` — ID der "Brand Info" Tabelle (oder null)

**Referenz-Dateien** (Fallback, falls Brand Info in Airtable nicht vorhanden):
- `references/brand-voice-sie.md`
- `references/brand-voice-du.md`
- `references/brand-voice-ich.md`
- `references/brand-voice-neutral.md`
- `references/copywriting-guidelines.md`

---

## Phase 0: Domain, Base ID und Brand auflösen

### Schritt 0.1: Domain bestimmen

Falls der User eine Domain in seiner Anfrage genannt hat, verwende diese direkt.
Ansonsten: Frage den User: "Für welche Domain soll der Artikel geschrieben werden? (z.B. talkpilot.io, effi-nova.de)"
→ Speichere als `[DOMAIN]`

### Schritt 0.2: Brand Name ableiten

Leite den Brand-Namen aus der Domain ab:
- `talkpilot.io` → "TalkPilot"
- `effi-nova.de` → "EffiNova"
- `mission-solar.at` → "Mission Solar"
- Allgemeine Regel: Domain ohne TLD, Bindestriche durch Leerzeichen, Titelcase
→ Speichere als `[BRAND_NAME]`

### Schritt 0.3: Brand Statement laden

Lese die Datei: `/Users/gregorkuhlmann/Desktop/Coding/SEO/[DOMAIN]/brand.md`

Falls vorhanden → Speichere Inhalt als `[BRAND_STATEMENT]`
Falls nicht vorhanden → `[BRAND_STATEMENT]` = leer (wird in Phase 2 aus Airtable oder WebFetch geladen)

### Schritt 0.4: Airtable Base ID ermitteln

Lade alle Bases mit `mcp__airtable__list_bases`.
Suche die Base, deren Name die Domain (`[DOMAIN]`) oder einen Markennamen (`[BRAND_NAME]`) enthält.
Falls mehrere Treffer: wähle die spezifischste.
Falls kein Treffer: Zeige dem User die verfügbaren Bases und bitte um Auswahl.
→ Speichere als `[BASE_ID]`

### Schritt 0.5: Tabellen-IDs bestimmen

Lade alle Tabellen der Base mit `mcp__airtable__list_tables` (detailLevel: "tableIdentifiersOnly").
Identifiziere:
- `[ARTICLE_PRODUCTION_TABLE_ID]` — Tabelle "Article Production" (oder ähnlicher Name)
- `[ARTICLE_IDEAS_TABLE_ID]` — Tabelle "Article Ideas" (oder ähnlicher Name)
- `[BRAND_INFO_TABLE_ID]` — Tabelle "Brand Info" (oder null falls nicht vorhanden)

Verwende diese IDs in allen nachfolgenden Airtable-Aufrufen.

---

## Phase 1: Cluster auswählen & Artikel laden

### Schritt 1: Cluster bestimmen

Falls der User einen Cluster (Main Keyword) bereits in seiner Anfrage genannt hat, verwende diesen direkt.

Ansonsten: Lade alle einzigartigen Werte aus dem Feld `Main Keyword` der Article Production Tabelle (filter: `{Status} = "Outline erstellt"`) und frage den User mit AskUserQuestion welchen Cluster er verarbeiten möchte.

### Schritt 2: Artikel aus Article Production laden (max. 5)

Lade Records aus Article Production mit:
```
baseId: [BASE_ID]
tableId: [ARTICLE_PRODUCTION_TABLE_ID]
filterByFormula: AND({Status} = "Outline erstellt", {Main Keyword} = "[CLUSTER_NAME]")
maxRecords: 5
```

Falls kein Ergebnis: Informiere den User und brich ab.

**Felder die geladen werden müssen:**
- `Title`, `Slug`
- `Artikel Typ` (How-To / Listicle / Leitfaden / Comparison / Anbietervergleich / News & Trends / Pillar)
- `Artikel Style` (Du-Perspektive / Sie-Perspektive / Neutral)
- `Primary Keyword`, `Main Keyword`, `Keyword Variations`
- `Gliederung`
- `Competitor Research`
- `Domain`
- `Cluster Name`
- Airtable Record ID

### Schritt 3: Pillar-Slug bestimmen

Der Pillar-Artikel ist der Artikel im Cluster, bei dem gilt: **`Artikel Typ = "Pillar"`** oder (falls kein Pillar-Typ vorhanden) **`Primary Keyword = Main Keyword`**.

**Vorgehen:**

a) Suche unter den geladenen Artikeln nach dem Pillar-Artikel.

b) Falls kein Pillar-Artikel unter den ersten 5 Ergebnissen: Lade zusätzlich alle anderen Records des Clusters aus Article Production und prüfe dort ebenfalls (`Artikel Typ = "Pillar"` oder `Primary Keyword = Main Keyword`).

c) Ermittle dessen `Slug` → das ist der **`PILLAR_SLUG`** für das gesamte Cluster.

d) Falls kein Pillar-Artikel gefunden: `PILLAR_SLUG = null` (keine Pillar-Verlinkung in diesem Durchlauf).

**Pillar-Slug in Article Production hinterlegen:**

Aktualisiere für ALLE Nicht-Pillar-Artikel des Clusters (nicht nur die 5 zu verarbeitenden) das Feld `Pillar-Slug` mit dem ermittelten `PILLAR_SLUG`:
```
filterByFormula: AND({Main Keyword} = "[CLUSTER_NAME]", {Artikel Typ} != "Pillar")
```
→ Batch-Update aller betroffenen Records: `{"Pillar-Slug": "[PILLAR_SLUG]"}`

### Schritt 4: Interne Link-Pool aus Article Ideas aufbauen

Lade alle Records aus der **Article Ideas Tabelle** (`[ARTICLE_IDEAS_TABLE_ID]`) für dasselbe Cluster:
```
baseId: [BASE_ID]
tableId: [ARTICLE_IDEAS_TABLE_ID]
filterByFormula: AND({Main Keyword} = "[CLUSTER_NAME]", {Slug} != "")
```

Für jeden Record: `Title` + `Slug` → URL = `https://[DOMAIN]/[Slug]`

Speichere diese Liste als **`CLUSTER_LINK_POOL`**. Diese Links werden für alle Artikel des Clusters als Quelle für interne Verlinkungen genutzt.

**Pillar-Link hinzufügen** (falls `PILLAR_SLUG` vorhanden und noch nicht im Pool): Füge einen Eintrag hinzu:
- URL: `https://[DOMAIN]/[PILLAR_SLUG]`
- Titel: "Hauptartikel: [CLUSTER_NAME]" (als Fallback-Label)

### Schritt 5: Bestätigung

Zeige dem User:
- Domain + Cluster-Name + Anzahl gefundener Artikel
- Pillar-Slug (oder Hinweis falls keiner gefunden)
- Interne Link-Pool (Anzahl verfügbarer Cluster-Links)
- Liste der zu verarbeitenden Artikel (Titel, Typ, Primary Keyword)

Frage ob so gestartet werden soll oder ob Anpassungen gewünscht sind.

---

## Phase 2: Brand Voice, Copywriting-Leitlinien und Design laden

### Schritt 2a: Brand Info Tabelle prüfen / erstellen

Prüfe ob `[BRAND_INFO_TABLE_ID]` vorhanden ist (aus Phase 0).

**Falls NICHT vorhanden**, erstelle sie mit `mcp__airtable__create_table`:

```json
{
  "name": "Brand Info",
  "description": "Brand-Informationen, Copywriting-Leitlinien, Brand Voices und Design-Anweisungen.",
  "fields": [
    { "name": "Brand", "type": "singleLineText" },
    { "name": "Homepage", "type": "url" },
    { "name": "Brand Statement", "type": "multilineText" },
    { "name": "Key Messages", "type": "multilineText" },
    { "name": "Zielgruppe", "type": "multilineText" },
    { "name": "Copywriting Leitlinien", "type": "multilineText" },
    { "name": "Brand Voice 1 - Ich-Perspektive", "type": "multilineText" },
    { "name": "Brand Voice 2 - Du-Perspektive", "type": "multilineText" },
    { "name": "Brand Voice 3 - Sie-Perspektive", "type": "multilineText" },
    { "name": "Brand Voice 4 - Neutral", "type": "multilineText" },
    { "name": "Design-Anweisungen", "type": "multilineText" },
    { "name": "Territory", "type": "singleLineText" },
    { "name": "Language", "type": "singleLineText" }
  ]
}
```

Danach Checkbox-Feld separat erstellen via `mcp__airtable__create_field`:
```json
{"field": {"name": "Brand Analyse starten", "type": "checkbox", "options": {"icon": "check", "color": "greenBright"}}}
```

### Schritt 2b: Brand-Record laden oder erstellen

Lade den Brand-Record:
```
baseId: [BASE_ID]
tableId: [BRAND_INFO_TABLE_ID]
filterByFormula: {Brand} = "[BRAND_NAME]"
```

**Falls der Brand-Record NICHT existiert** → Erstanlage durchführen:

1. **Homepage ermitteln**: `https://[DOMAIN]`
2. **Homepage analysieren** via **WebFetch**:
   ```
   Analysiere diese Website und extrahiere:
   1. BRAND STATEMENT: Was macht das Unternehmen? (1-2 Sätze)
   2. KEY MESSAGES: Die 3-5 wichtigsten Botschaften der Brand
   3. ZIELGRUPPE: Wer wird angesprochen? (inkl. Needs und Werte)
   4. COPYWRITING STIL: Wie kommuniziert die Brand? (Tonalität, Regeln)
   5. DESIGN: Farben (HEX), Schriften, Layout, UI-Elemente, Stil
   Gib alles auf Deutsch aus.
   ```
3. **Brand Voices** aus den Standard-Templates befüllen (lokale MD-Files als Basis):
   - `references/brand-voice-ich.md` → `Brand Voice 1 - Ich-Perspektive`
   - `references/brand-voice-du.md` → `Brand Voice 2 - Du-Perspektive`
   - `references/brand-voice-sie.md` → `Brand Voice 3 - Sie-Perspektive`
   - `references/brand-voice-neutral.md` → `Brand Voice 4 - Neutral`

   **Anpassung:** Ersetze alle "TalkPilot"-Referenzen durch `[BRAND_NAME]` und passe Branchenbeispiele an die neue Brand an.

4. **Record in Airtable erstellen** mit allen gesammelten Daten.

5. **User informieren**: Zeige die automatisch erstellten Brand-Daten und frage ob Anpassungen nötig sind.

### Schritt 2c: Brand-Daten aus Record laden

Aus dem (vorhandenen oder gerade erstellten) Brand-Record laden:

- `Brand Statement` → falls leer: `[BRAND_STATEMENT]` aus Phase 0 verwenden
- `Copywriting Leitlinien` → `COPYWRITING_GUIDELINES`
- `Design-Anweisungen` → `DESIGN_GUIDELINES`
- Basierend auf `Artikel Style` die passende Brand Voice:
  - `Ich-Perspektive` → Feld `Brand Voice 1 - Ich-Perspektive`
  - `Du-Perspektive` → Feld `Brand Voice 2 - Du-Perspektive`
  - `Sie-Perspektive` → Feld `Brand Voice 3 - Sie-Perspektive`
  - `Neutral` → Feld `Brand Voice 4 - Neutral`
- → Speichern als `BRAND_VOICE_TEXT`

### Schritt 2d: Design-Anweisungen nachholen (falls leer)

Falls `Design-Anweisungen` leer ist:

1. Rufe `https://[DOMAIN]` via **WebFetch** auf:
   ```
   Analysiere das Design dieser Website: Farben (HEX-Codes), Schriften, Layout, UI-Elemente (Buttons, Cards, Icons), Schatten/Effekte, allgemeiner Stil. Erstelle konkrete CSS-nahe Anweisungen für Blog-Artikel-HTML, damit sie visuell zur Website passen.
   ```
2. Speichere das Ergebnis im Feld `Design-Anweisungen` in Airtable
3. Verwende es als `DESIGN_GUIDELINES`

### Schritt 2e: Fallback auf MD-Referenzfiles

Nur falls Airtable komplett nicht erreichbar ist (MCP-Fehler):

1. `references/copywriting-guidelines.md` → `COPYWRITING_GUIDELINES` (ggf. TalkPilot-Referenzen durch `[BRAND_NAME]` ersetzen)
2. Basierend auf `Artikel Style` das passende Brand-Voice-File → `BRAND_VOICE_TEXT`
3. `DESIGN_GUIDELINES` = leer

---

## Phase 3: Artikel produzieren (pro Artikel, sequenziell)

Verarbeite jeden Artikel einzeln nacheinander. Für jeden Artikel: alle Schritte 3.1–3.7 ausführen, dann speichern (Phase 4), dann weiter zum nächsten.

**Wichtig:** Den eigenen Artikel aus dem `CLUSTER_LINK_POOL` beim Einbauen der internen Links herausfiltern (nicht auf sich selbst verlinken).

---

### Schritt 3.1: Draft Writer

**Aufgabe:** Schreibe einen vollständigen, SEO-optimierten Blogartikel auf Basis der Gliederung.

**System-Prompt:**
```
Rolle und Ziel:
Du bist ein erfahrener KI-Blog-Autor für die Brand [BRAND_NAME] und Teil eines SEO-Teams. Deine Aufgabe ist es, einen hochwertigen Artikel zu einem bestimmten Thema zu schreiben, der auf einem bereitgestellten Research Brief basiert und den Lesern einen praktischen Mehrwert bietet.

Brand Statement:
[BRAND_STATEMENT]

Copywriting Leitlinien:
[COPYWRITING_GUIDELINES]

Brand Voice:
[BRAND_VOICE_TEXT]

Anforderungen:
1. Research Brief als Grundlage: Verwende die bereitgestellte Gliederung als inhaltliche Grundlage.
2. Gliederung einhalten: Halte dich strikt an die H1/H2/H3-Struktur der Gliederung.
3. WebSearch-Recherche: Recherchiere mit WebSearch aktuelle und relevante Informationen zu JEDEM H2-Kapitel einzeln. Suche nach [Kapitel-Titel] + [Primary Keyword] + Deutschland.
4. Wiederholungen vermeiden: Jedes Thema wird nur in einem Kapitel behandelt.
5. Keyword-Pool: Integriere die bereitgestellten Keywords natürlich in den Fließtext.
6. Praktischer Mehrwert: Schritt-für-Schritt-Anleitungen, praktische Tipps.
7. Listen: Erstelle <ul>/<ol> Listen wo sinnvoll (ab 3 Punkten).
8. Kapitellänge: Jedes Kapitel (<h2> oder <h3>) unter 300 Wörtern.

Keyword-Integration:
- Natürlich integrieren — kein Keyword-Stuffing
- Gleichmäßig verteilen über Einleitung, Zwischenüberschriften, Absätze und Fazit
- Keywords nie fett markieren (<strong>)

Quellen-Abschnitt:
Füge am Ende einen Quellen-Abschnitt ein:
<hr><h3>Quellen</h3><ul><li><a href="[URL]">[Publikationsname] - [Artikeltitel]</a></li>...</ul>

HTML-Ausgabe:
Gib den Artikel als reinen HTML-Text zurück (kein Markdown, keine JSON-Struktur).
Verwende: <h1>, <h2>, <h3>, <p>, <ul>, <ol>, <li>, <strong> (nur für Unser-Tipp-Links), <a>, <hr>

Design-Anweisungen (für Inline-Styles und HTML-Struktur):
[DESIGN_GUIDELINES]
Wende diese Design-Richtlinien an, wo sinnvoll — insbesondere bei Link-Farben, Überschriften und Infoboxen.
```

**User-Prompt:**
```
Primäres Keyword: [PRIMARY_KEYWORD]
Main Keyword: [MAIN_KEYWORD]
Keyword Variations: [KEYWORD_VARIATIONS]
Artikel Typ: [ARTIKEL_TYP]

Gliederung:
[GLIEDERUNG]

Competitor Research (Kontext):
[COMPETITOR_RESEARCH — max. 500 Zeichen]
```

---

### Schritt 3.2: Editor (ohne Zitate im Fließtext)

**Aufgabe:** Überarbeite den Entwurf für Markenkonformität, SEO-Qualität und Lesefluss.

**System-Prompt:**
```
Du bist ein professioneller Editor für [BRAND_NAME]. Überarbeite den folgenden Artikel-Entwurf:

Brand Voice: [BRAND_VOICE_TEXT]
Copywriting Leitlinien: [COPYWRITING_GUIDELINES]

Aufgaben:
1. Brand Voice sicherstellen: Tonalität und Ansprache (Du/Sie/Neutral) konsequent durchhalten
2. Keine Sternchen (**) im Fließtext
3. Keine fett markierten Keywords
4. Keine leeren Superlative: "revolutionär", "bahnbrechend", "beispiellos", "nahtlos"
5. Keine Klischee-Einstiege: "Stellen Sie sich vor", "Stell dir vor", "In einer Welt..."
6. Zitate/Quellenverweise aus dem Fließtext entfernen — Quellen nur im Quellen-Abschnitt am Ende
7. H2-Abschnitte unter 300 Wörtern
8. Primary Keyword in ersten 100 Wörtern: [PRIMARY_KEYWORD]
9. Einleitung: Kernproblem direkt und konkret schildern
10. HTML-Struktur beibehalten
11. Design-Konsistenz prüfen: Inline-Styles und HTML-Struktur gemäß Design-Anweisungen

Design-Anweisungen:
[DESIGN_GUIDELINES]

Gib den überarbeiteten Artikel als reinen HTML-Text zurück, ohne Kommentare.
```

**User-Prompt:**
```
[DRAFT_HTML]
```

---

### Schritt 3.3: People Also Ask (PAA) laden via DataForSEO

**Aufgabe:** Lade "People Also Ask"-Fragen von Google für das Primary Keyword, um sie in das FAQ-Akkordeon einzubauen.

**Vorgehen:**

Nutze die **DataForSEO SERP API** um die Google-SERP abzurufen und die PAA-Fragen zu extrahieren:

```bash
curl -s -X POST "https://api.dataforseo.com/v3/serp/google/organic/live/advanced" \
  -u "gregor.kuhlmann.bkbm@gmail.com:e92a2a949b8b4e94" \
  -H "Content-Type: application/json" \
  -d '[{"keyword": "[PRIMARY_KEYWORD]", "language_code": "de", "location_code": 2276, "depth": 10}]'
```

**Aus der Response extrahieren:**

Filtere alle Items mit `"type": "people_also_ask"` aus `tasks[0].result[0].items`. Jedes PAA-Item hat:
- `title` — die Frage
- `snippet` — die Kurzantwort (optional als Basis für FAQ-Antworten)

Speichere die PAA-Fragen (mindestens 4, maximal 6) als `PAA_QUESTIONS` (Liste von Strings).

**Fallback:** Falls DataForSEO keine PAA-Ergebnisse liefert oder einen Server-Fehler zurückgibt, nutze **WebSearch** mit `[PRIMARY_KEYWORD] häufige Fragen` und extrahiere relevante Fragen aus den Ergebnissen.

---

### Schritt 3.4: Visual Enrichment (CSS + Summary + FAQ-Akkordeon + Grafiken + Listen)

**Aufgabe:** Den editierten Artikel visuell und informativ anreichern mit Eye-Catching-Elementen, die zum Brand-Design passen.

**System-Prompt:**
```
Du bist ein auf SEO-Blogartikel spezialisierter Webdeveloper.

Du bekommst einen bereits fertig geschriebenen Artikel und deine einzige Aufgabe ist es, den Artikel visuell und informativ anzureichern.

## DESIGN-KONTEXT

[DESIGN_GUIDELINES]

## AUFGABE

Integriere folgende Elemente in den Artikel:

### 1. CSS-Styling
Füge ein <style>-Tag am Anfang des Artikels ein mit allen CSS-Regeln.
Orientiere dich an den Design-Anweisungen oben. Falls keine vorhanden: verwende ein sauberes, modernes Design (Inter, sans-serif, max-width: 860px, margin: auto). Achte besonders auf:
- Schriftfamilie und Größen
- Überschriften-Farben
- Link-Farben (aus DESIGN_GUIDELINES oder #2563eb als Fallback)
- Absatz-Farben (#4b5563)
- Responsive Abstände

### 2. Zusammenfassungs-Box
Erstelle eine Box mit 5 Bullet-Points der Kernaussagen des Artikels.
- Platzierung: direkt unter dem Einleitungstext (nach dem ersten <p> Block)
- Styling: abgehobener Hintergrund, runde Ecken (border-radius: 12px), leichter Schatten
- Überschrift: "Auf einen Blick" oder "Das Wichtigste in Kürze"

### 3. FAQ-Akkordeon
Erstelle ein klickbares Akkordeon mit 5 Fragen und Antworten.
- Antworten im Fließtext formuliert (keine Bullet Points)
- Folgende People-Also-Ask-Fragen MÜSSEN eingebaut werden (Dopplungen vermeiden):
  [PAA_QUESTIONS]
- Restliche Fragen aus dem Artikelinhalt ableiten
- Keywords natürlich einbauen: [KEYWORD_VARIATIONS]
- Platzierung: direkt VOR dem Quellen-Abschnitt (<hr><h3>Quellen</h3>)
- Akkordeon: <details>/<summary> HTML-Elemente (JS-frei)

### 4. Diagramme und Grafiken
Analysiere den Artikel nach Stellen, die sich für Visualisierungen eignen:
- Tabellen mit Zahlenwerten → als Chart darstellen (inline SVG oder CSS-basiert)
- Vergleiche → visueller Vergleich
- Prozesse/Schritte → Flowchart oder nummerierte Schritte visuell
- NUR wenn sinnvolle Daten vorhanden sind — keine erzwungenen Grafiken

### 5. Gestylte Listen
Formatiere <ul>/<ol> Listen so, dass die einzelnen Punkte eigene Container bekommen:
- Abgehobener Hintergrund pro Listenpunkt
- Runde Ecken (border-radius: 8px)
- Leichter Schatten oder Border
- Inhalt in normaler Textgröße (KEINE h3-Überschriften in Listen)
- Konsistente Abstände zwischen den Containern

## AUSGABEFORMAT

Reiner HTML-Output. Ändere am Textinhalt des Artikels NICHTS — gib ihn 1:1 so aus wie erhalten, aber füge die visuellen Elemente und CSS hinzu.
```

**User-Prompt:**
```
[EDITED_ARTICLE_HTML]

People Also Ask Fragen:
[PAA_QUESTIONS]
```

**Output:** Der visuell angereicherte Artikel als `ENRICHED_ARTICLE_HTML`.

---

### Schritt 3.5: Interne Links einbauen

**Schritt 3.5a: Cluster-Links auswählen**

Aus dem `CLUSTER_LINK_POOL` (Phase 1, Schritt 4):
- Entferne den aktuellen Artikel aus dem Pool (nicht auf sich selbst verlinken)
- Wähle **mindestens 3 Links** — thematisch passend zum Artikelinhalt
- Der Pillar-Link (`PILLAR_SLUG`) soll immer dabei sein, sofern der aktuelle Artikel nicht selbst der Pillar-Artikel ist

**Schritt 3.5b: Links in Artikel einbauen**

Format:
```html
<strong>Unser Tipp:</strong> <a href="https://[DOMAIN]/[slug]">[Kurzbeschreibung was der Leser dort findet]</a>
```

Platzierungsregeln:
- NICHT in der Einleitung (erster H2-Block)
- NICHT innerhalb der Summary-Box oder des FAQ-Akkordeons
- Jeden Link nur einmal setzen
- Thematisch passend nach einem verwandten Absatz platzieren
- Falls weniger als 3 Links verfügbar: alle vorhandenen setzen und im Report vermerken

---

### Schritt 3.6: Meta Description

**Prompt:**
```
Erstelle eine Meta Description.

Regeln:
- Maximal 160 Zeichen (zählen!)
- Enthält Primary Keyword: [PRIMARY_KEYWORD]
- Klar, informativ, klickfördernd — kein Clickbait
- Sprache: Deutsch
- NUR die Meta Description ausgeben, ohne Anführungszeichen

Artikel-Titel: [TITLE]
Artikel-Einleitung (erste 200 Zeichen): [ARTICLE_INTRO_200]
```

---

### Schritt 3.7: Schema.org JSON-LD

**Schema-Typ-Auswahl:**
- `How-To` → `HowTo` Schema + FAQPage
- Alle anderen → `BlogPosting` + FAQPage

**BlogPosting Template:**
```json
{
  "@context": "https://schema.org",
  "@type": "BlogPosting",
  "headline": "[TITLE]",
  "description": "[META_DESCRIPTION]",
  "author": { "@type": "Organization", "name": "[BRAND_NAME]" },
  "publisher": {
    "@type": "Organization",
    "name": "[BRAND_NAME]",
    "logo": { "@type": "ImageObject", "url": "https://[DOMAIN]/logo.png" }
  },
  "url": "https://[DOMAIN]/[SLUG]",
  "datePublished": "[HEUTE_ISO]",
  "dateModified": "[HEUTE_ISO]",
  "mainEntityOfPage": { "@type": "WebPage", "@id": "https://[DOMAIN]/[SLUG]" },
  "keywords": "[PRIMARY_KEYWORD], [KEYWORD_VARIATIONS]"
}
```

**HowTo Template** (nur Artikel Typ "How-To"):
```json
{
  "@context": "https://schema.org",
  "@type": "HowTo",
  "name": "[TITLE]",
  "description": "[META_DESCRIPTION]",
  "step": [
    { "@type": "HowToStep", "name": "[H2 Schritt 1]", "text": "[Kurzbeschreibung]" }
  ],
  "author": { "@type": "Organization", "name": "[BRAND_NAME]" },
  "publisher": { "@type": "Organization", "name": "[BRAND_NAME]" }
}
```

**FAQPage** (immer ergänzen, als separates `<script>`):
```json
{
  "@context": "https://schema.org",
  "@type": "FAQPage",
  "mainEntity": [
    {
      "@type": "Question",
      "name": "[Frage 1]",
      "acceptedAnswer": { "@type": "Answer", "text": "[Antwort 1]" }
    }
  ]
}
```

Fragen + Antworten aus dem FAQ-HTML extrahieren. Ausgabe: alle JSON-LDs als `<script type="application/ld+json">` Tags.

---

## Phase 4: In Airtable speichern

Aktualisiere den Article Production Record mit `mcp__airtable__update_records`:

```json
{
  "fields": {
    "Meta Description": "[META_DESCRIPTION]",
    "FAQ": "[FAQ_ACCORDION_HTML]",
    "Schema.org": "[SCHEMA_ORG_JSON_LD]",
    "Status": "Draft erstellt"
  }
}
```

**WICHTIG — "Finaler Artikel in HTML" wird NICHT in Airtable gespeichert.**
Der vollständige HTML-Artikel ist zu groß für den MCP-Transfer (35+ KB überschreiten den Parameter-Limit).
Die HTML-Datei liegt lokal unter `/Users/gregorkuhlmann/Desktop/Coding/SEO/[DOMAIN]/[SLUG].html` — das ist die einzige Quelle der Wahrheit für den Artikel-HTML.

**Nicht überschreiben:** `Slug`, `Primary Keyword`, `Main Keyword`, `Gliederung`, `Competitor Research`.

---

## Phase 5: HTML-Dateien im Domain-Ordner ablegen

Speichere jeden fertigen Artikel als vollständiges HTML-Dokument.

**Zielverzeichnis:** `/Users/gregorkuhlmann/Desktop/Coding/SEO/[DOMAIN]/`

**Dateiname:** `[SLUG].html`

**Vollständiges HTML-Dokument:**
```html
<!DOCTYPE html>
<html lang="de">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>[TITLE] | [BRAND_NAME]</title>
  <meta name="description" content="[META_DESCRIPTION]">
</head>
<body>
  <article class="blog-article">
    [ENRICHED_ARTICLE_HTML_WITH_INTERNAL_LINKS]
  </article>
  <script type="application/ld+json">[SCHEMA_ORG_JSON_LD]</script>
</body>
</html>
```

Hinweis: Das CSS des Artikels ist bereits im `<style>`-Tag innerhalb von `ENRICHED_ARTICLE_HTML` enthalten.

**Verifikation:** Prüfe ob die Datei korrekt erstellt wurde und eine sinnvolle Dateigröße hat (> 10 KB pro Artikel).

---

## Phase 6: Abschluss-Report

```
✅ Article Writer — [DOMAIN] / Cluster "[CLUSTER_NAME]" abgeschlossen

Pillar-Slug: /[PILLAR_SLUG]
Verarbeitete Artikel: [N]

[Für jeden Artikel:]
📄 [TITLE]
   Typ: [ARTIKEL_TYP] | Style: [ARTIKEL_STYLE]
   Keyword: [PRIMARY_KEYWORD]
   URL: https://[DOMAIN]/[SLUG]
   Meta: [META_DESCRIPTION_GEKÜRZT]...
   Interne Links: [ANZAHL] eingebaut
   HTML: /Desktop/Coding/SEO/[DOMAIN]/[SLUG].html ✓
   Airtable: Meta ✓ | FAQ ✓ | Schema.org ✓ | Status: Draft erstellt ✓

Alle Drafts gespeichert in Airtable (Article Production).
HTML-Dateien: /Users/gregorkuhlmann/Desktop/Coding/SEO/[DOMAIN]/ ([N] Dateien)
```

Abschlussfrage: "Möchtest du direkt den nächsten Cluster schreiben oder die Artikel reviewen?"

---

## Qualitätsprüfung (vor dem Speichern pro Artikel)

- [ ] Primary Keyword in ersten 100 Wörtern?
- [ ] Keine `**` Sternchen im Fließtext?
- [ ] `<strong>` nur für "Unser Tipp:" Links?
- [ ] H2-Abschnitte unter 300 Wörtern?
- [ ] Mind. 3 interne Links (davon einer der Pillar-Link)?
- [ ] Summary-Box vorhanden (direkt nach Einleitung)?
- [ ] FAQ-Akkordeon vorhanden (vor Quellen-Abschnitt)?
- [ ] FAQ-Akkordeon enthält PAA-Fragen?
- [ ] CSS `<style>` Tag vorhanden und Design-konform?
- [ ] Listen mit eigenen Containern gestylt?
- [ ] Meta Description ≤ 160 Zeichen?
- [ ] Schema.org JSON-LD syntaktisch korrekt (kein trailing comma)?
- [ ] Slug NICHT überschrieben?
- [ ] HTML-Datei lokal gespeichert?

---

## Fehlerbehandlung

- **Artikel ohne Gliederung**: Überspringen, User benachrichtigen. → Zuerst `seo-article-research` Skill ausführen.
- **Kein Pillar-Artikel gefunden**: `PILLAR_SLUG = null`, Verlinkung ohne Pillar fortfahren, im Report vermerken.
- **Weniger als 3 Cluster-Links verfügbar**: Alle vorhandenen einbauen, im Report vermerken.
- **DataForSEO Server-Fehler**: PAA via WebSearch `[PRIMARY_KEYWORD] häufige Fragen` ermitteln.
- **Airtable-Update fehlgeschlagen**: Einmal wiederholen. Bei erneutem Fehler: Meta/FAQ/Schema als JSON im Workspace speichern und User informieren. HTML-Datei ist bereits lokal gespeichert.
- **WebSearch ohne Ergebnis für ein Kapitel**: Mit Wissen aus Research Brief weiterschreiben.
- **Brand Info Tabelle nicht in Airtable**: Automatisch erstellen (Schritt 2a), dann mit Homepage-Analyse befüllen.
- **brand.md nicht gefunden**: Ohne Brand Statement fortfahren — Brand Info aus Airtable oder Homepage-Analyse wird in Phase 2 geladen.

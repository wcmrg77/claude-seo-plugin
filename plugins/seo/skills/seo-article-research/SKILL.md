---
name: article-research
description: >
  Führt die komplette Artikel-Recherche für SEO-Blogartikel durch: Wettbewerberanalyse via DataForSEO SERP, Content-Analyse der Top-Artikel via WebFetch, Themenrecherche via WebSearch, und generiert eine strukturierte H1/H2-Gliederung basierend auf dem Artikeltyp. Speichert alles in einer Airtable-Tabelle ("Article Production").

  Verwende diesen Skill wenn der User eine Artikel-Recherche starten, eine Gliederung erstellen, ein Outline generieren, oder einen Artikel vorbereiten möchte. Trigger-Phrasen: "recherchiere den Artikel", "erstelle eine Gliederung", "outline erstellen", "artikel recherche", "gliederung für", "outline für", "artikel vorbereiten", "content outline", "recherche starten für", "artikel planen".
---

# Article Research Skill

Führt die komplette Recherche- und Gliederungsphase für SEO-Blogartikel durch. Verarbeitet nur Artikel, die in der "Article Ideas" Tabelle den Status **"Ausgewählt"** haben.

**Workflow:**
1. **Airtable prüfen** — "Article Production" Tabelle prüfen/erstellen
2. **Artikel laden** — Nur Artikel mit Status "Ausgewählt" aus Article Ideas
3. **Researcher** — SERP-Analyse + WebFetch Top-Artikel + WebSearch Deep Research
4. **Content Planner** — Artikeltyp-spezifische H1/H2-Gliederung erstellen
5. **Speichern** — Alles in Article Production + Status-Update in Article Ideas

## Konfiguration

- **DataForSEO Credentials**: aus `.claude.json` lesen
- **Territory**: Deutschland
- **Language**: Deutsch
- Alle weiteren Parameter (Base ID, Table IDs, Brand Statement, Domain) werden in **Phase 0** dynamisch ermittelt

---

## Phase 0: Domain & Konfiguration bestimmen

### Schritt 1 — Domain bestimmen

Nimm die Domain aus dem Kontext (User hat sie genannt), oder frage:
> "Für welche Domain/welches Projekt soll ich die Recherche durchführen?"

Merke dir die Domain als `[DOMAIN]` (z. B. `effi-nova.de`, `talkpilot.io`).

### Schritt 2 — Brand Statement laden

Lies die Datei `~/Desktop/Coding/SEO/[DOMAIN]/brand.md`.

- Falls gefunden: Inhalt als `[BRAND_STATEMENT]` verwenden
- Falls nicht gefunden: Frage den User: "Ich habe keine brand.md für [DOMAIN] gefunden. Kurzes Brand Statement? (Optional)"

### Schritt 3 — Airtable Base ermitteln

Lade alle verfügbaren Bases via `mcp__airtable__list_bases`.

- Suche nach einer Base deren Name die Domain (oder einen erkennbaren Teil) enthält
- **Eindeutiger Treffer** → automatisch als `[BASE_ID]` übernehmen
- **Mehrere Treffer** → dem User zur Auswahl zeigen
- **Kein Treffer** → alle Bases zeigen und User fragen

### Schritt 4 — Tabellen identifizieren

Lade alle Tabellen via `mcp__airtable__list_tables` mit `[BASE_ID]`.

Identifiziere:
- **`[ARTICLE_IDEAS_TABLE_ID]`**: Tabelle mit "Article Ideas" oder "Artikel Ideen" im Namen
- **`[SEO_KEYWORDS_TABLE_ID]`**: Tabelle mit "SEO Keywords" oder "Keyword" im Namen
- **Article Production**: wird in Phase 1 geprüft/erstellt

> Merke dir für alle weiteren Phasen: `[BASE_ID]`, `[ARTICLE_IDEAS_TABLE_ID]`, `[SEO_KEYWORDS_TABLE_ID]`, `[DOMAIN]`, `[BRAND_STATEMENT]`

---

## Phase 1: Airtable "Article Production" Tabelle prüfen/erstellen

Prüfe mit `mcp__airtable__list_tables` (baseId: `[BASE_ID]`) ob "Article Production" existiert.

**Falls NICHT vorhanden**, erstelle mit `mcp__airtable__create_table` (baseId: `[BASE_ID]`):

```json
{
  "name": "Article Production",
  "fields": [
    { "name": "Title", "type": "singleLineText" },
    { "name": "Slug", "type": "singleLineText" },
    { "name": "Artikel Typ", "type": "singleSelect", "options": { "choices": [
      { "name": "How-To" }, { "name": "Listicle" }, { "name": "Leitfaden" },
      { "name": "Comparison" }, { "name": "Anbietervergleich" },
      { "name": "News & Trends" }, { "name": "Pillar" }
    ]}},
    { "name": "Funnel", "type": "singleSelect", "options": { "choices": [
      { "name": "TOFU" }, { "name": "MOFU" }, { "name": "BOFU" }
    ]}},
    { "name": "Artikel Style", "type": "singleSelect", "options": { "choices": [
      { "name": "Du-Perspektive" }, { "name": "Sie-Perspektive" }, { "name": "Neutral" }
    ]}},
    { "name": "Primary Keyword", "type": "singleLineText" },
    { "name": "Main Keyword", "type": "singleLineText" },
    { "name": "Keyword Variations", "type": "singleLineText" },
    { "name": "MSV", "type": "number", "options": { "precision": 0 } },
    { "name": "Cluster Name", "type": "singleLineText" },
    { "name": "Pillar-Slug", "type": "singleLineText" },
    { "name": "Competitor Research", "type": "multilineText" },
    { "name": "Gliederung", "type": "multilineText" },
    { "name": "Finaler Artikel in HTML", "type": "multilineText" },
    { "name": "FAQ", "type": "multilineText" },
    { "name": "Meta Description", "type": "singleLineText" },
    { "name": "Schema.org", "type": "multilineText" },
    { "name": "alt_texte", "type": "multilineText" },
    { "name": "Publication Date", "type": "date" },
    { "name": "Domain", "type": "singleLineText" },
    { "name": "Status", "type": "singleSelect", "options": { "choices": [
      { "name": "Ausgewählt" },
      { "name": "Research läuft" },
      { "name": "Outline erstellt" },
      { "name": "Draft erstellt" },
      { "name": "Review" },
      { "name": "Veröffentlicht" }
    ]}}
  ]
}
```

> ⚠️ **Hinweis**: Das `Completed`-Checkbox-Feld NICHT in `create_table` mitgeben — die Airtable API verlangt dafür zwingend `options`, was beim Table-Create zu einem Validierungsfehler führt. Stattdessen nach der Table-Erstellung separat via `mcp__Airtable_MCP_Server__create_field` hinzufügen:
> ```json
> {"field": {"name": "Completed", "type": "checkbox", "options": {"icon": "check", "color": "greenBright"}}}
> ```

---

## Phase 1b: Checkbox-Feld in Article Ideas erkennen

Rufe die Feldliste der Article Ideas Tabelle ab via `mcp__airtable__describe_table` (baseId: `[BASE_ID]`, tableId: `[ARTICLE_IDEAS_TABLE_ID]`) und suche nach einem Feld mit `"type": "checkbox"`.

**Fall A — Checkbox-Feld gefunden**:
- Merke dir den tatsächlichen Feldnamen (z. B. "Ausgewählt", "Artikel auswählen", o. ä.)
- Verwende diesen Namen in **allen** nachfolgenden `filterByFormula`-Aufrufen statt des hartcodierten "Ausgewählt"

**Fall B — Kein Checkbox-Feld vorhanden**:
- Erstelle es via `mcp__Airtable_MCP_Server__create_field`:
```json
{"field": {"name": "Ausgewählt", "type": "checkbox", "options": {"icon": "check", "color": "greenBright"}, "description": "Anhaken um diesen Artikel für die Recherche-Phase auszuwählen."}}
```
- Verwende "Ausgewählt" als Feldname für alle weiteren Schritte

Teile dem User mit, dass er in Airtable die gewünschten Artikel per Häkchen auswählen soll, falls noch nicht geschehen.

---

## Phase 2: Cluster auswählen und ausgewählte Artikel laden

### Schritt 1 — Cluster bestimmen

Frage den User nach dem gewünschten Cluster (Main Keyword), oder nimm es aus dem Kontext falls bereits genannt.

Falls unklar: Lade die verfügbaren Cluster aus der Article Ideas Tabelle und zeige sie dem User zur Auswahl:

```
mcp__airtable__list_records
  baseId: [BASE_ID]
  tableId: [ARTICLE_IDEAS_TABLE_ID]
  fields: ["Main Keyword"]
```

Extrahiere die eindeutigen `Main Keyword`-Werte und zeige sie als Auswahl.

### Schritt 2 — Ausgewählte Artikel des Clusters laden

Lade nur die Artikel, die **sowohl** das Häkchen "Ausgewählt" haben **als auch** zum gewählten Cluster gehören:

```
mcp__airtable__list_records
  baseId: [BASE_ID]
  tableId: [ARTICLE_IDEAS_TABLE_ID]
  filterByFormula: AND({[CHECKBOX_FIELD_NAME]} = TRUE(), {Main Keyword} = "[CLUSTER_NAME]")
```

Falls keine Artikel mit Häkchen gefunden werden: User informieren, dass er in Airtable zuerst die gewünschten Artikel im Cluster "[CLUSTER_NAME]" per Häkchen auswählen soll.

Für jeden Artikel extrahiere: `Title`, `Slug`, `Type`, `Funnel`, `Primary Keyword`, `Main Keyword`, `Description`.

**Zusätzlich**: Lade aus der SEO Keywords Tabelle (`tblSdZQb9rngWvFax`) die Metadaten zum Primary Keyword via `filterByFormula: {Primary Keyword} = "[PRIMARY_KEYWORD]"`:
- `Variations` → wird als `Keyword Variations` in Article Production gespeichert
- `Agg. Volume` → wird als `MSV` gespeichert
- Ggf. `Cluster Name`

> ⚠️ **Hinweis**: Der Feldname in der SEO Keywords Tabelle lautet `Variations` (nicht "Keyword Variations"). Das Mapping beim Speichern in Article Production: `Variations` → `"Keyword Variations"`.

**Duplikat-Check**: Prüfe ob der Slug bereits in der Article Production Tabelle existiert. Falls ja, überspringe (es sei denn der User will überschreiben).

**Workspace vorbereiten**:
```bash
WORKSPACE="$(pwd)/article-research-workspace"
mkdir -p "$WORKSPACE"
```

---

## Phase 3: Researcher (pro Artikel)

Für jeden Artikel wird ein Subagent gestartet. Bei mehreren Artikeln: **parallel** (max 5 gleichzeitig).

Der komplette Researcher-Prompt wird **inline** mitgegeben (kein Verweis auf externe Dateien).

### Subagent-Prompt: Researcher

Ersetze alle Platzhalter vor dem Senden:

```
Du bist ein KI-Agent, spezialisiert auf SEO-Wettbewerberanalyse. Dein Ziel: Informationen aus verschiedenen Quellen sammeln, analysieren und synthetisieren.

**Artikeltitel**: [ARTICLE_TITLE]
**Beschreibung**: [DESCRIPTION]
**Primary Keyword**: [PRIMARY_KEYWORD]
**Territory**: Deutschland
**Language**: Deutsch

---

## Schritt 1 — SERP-Analyse via DataForSEO

Rufe die Top-10 Google-Ergebnisse ab:

```bash
curl -s -X POST "https://api.dataforseo.com/v3/serp/google/organic/live/advanced" \
  -u "[DATAFORSEO_USER]:[DATAFORSEO_PASS]" \
  -H "Content-Type: application/json" \
  -d '[{"keyword": "[ARTICLE_TITLE]", "language_code": "de", "location_code": 2276, "depth": 10}]'
```

Extrahiere die Top-5 organischen Ergebnisse (type: "organic"):
- rank_absolute, title, url, description

## Schritt 2 — Content-Analyse der Top-3 URLs

Rufe für die Top-3 URLs den Seiteninhalt ab (nutze WebFetch).
Falls eine URL nicht erreichbar ist: überspringen und nächste nehmen.

Für jeden Artikel extrahiere:
- H1/H2/H3-Überschriftenstruktur
- Hauptthemen und Kernaussagen (1 Zeile pro Artikel)
- Inhaltliche Stärken (1 Zeile pro Artikel)
- Geschätzte Wortanzahl

## Schritt 3 — Deep Research via WebSearch

Führe 2-3 gezielte WebSearch-Abfragen durch:
1. "[PRIMARY_KEYWORD] Deutschland" — für lokalen Kontext
2. "[ARTICLE_TITLE]" — für direkte Thementiefe
3. "[PRIMARY_KEYWORD] Studie OR Statistik OR Trend" — für Daten und Fakten

Fasse die wichtigsten Erkenntnisse aus den Suchergebnissen zusammen.

## Schritt 4 — Analyse und Synthese

Basierend auf allen gesammelten Daten, analysiere:
1. **Suchintention**: Informativ, transaktional, navigational oder kommerziell?
2. **Content-Stärken**: Was machen die Top-Artikel gut?
3. **Content-Schwächen**: Wo sind Artikel zu oberflächlich?
4. **Content Gaps**: Welche Aspekte fehlen komplett?
5. **Key Subtopics**: Welche Unterthemen sind relevant?

## Output-Format

Gib den Research-Output in GENAU diesem Format aus:

```
### Wettbewerber-URLs
- [Titel 1](URL 1) — Kurzzusammenfassung
- [Titel 2](URL 2) — Kurzzusammenfassung
- ...

### Analyse der Suchintention
[1 Absatz]

### Wettbewerberinhalte: Stärken
- [1 Zeile pro Stärke]

### Wettbewerberinhalte: Schwächen
- [1 Zeile pro Schwäche]

### Identifizierte Inhaltslücken
[1 Absatz]

### Key Subtopics
- Subtopic 1
- Subtopic 2
- ...

### Ergänzende Recherche
[1 Absatz: Kernerkenntnisse aus der WebSearch-Recherche]
```

## Ergebnis speichern

Speichere das Ergebnis als JSON in: **[OUTPUT_FILEPATH]**

```json
{
  "slug": "[SLUG]",
  "title": "[ARTICLE_TITLE]",
  "article_type": "[TYPE]",
  "funnel": "[FUNNEL]",
  "primary_keyword": "[PRIMARY_KEYWORD]",
  "main_keyword": "[MAIN_KEYWORD]",
  "research": "[DER KOMPLETTE RESEARCH OUTPUT]"
}
```

KRITISCH: Der JSON-Root-Key muss `"research"` heißen.
```

---

## Phase 4: Artikeltyp-Router + Outline-Generierung

Sobald die Research-Phase abgeschlossen ist, wird für jeden Artikel die Gliederung erstellt.

### Schritt 1 — Outline-Template laden

Lies die Referenzdatei: **`references/outline-templates.md`**

Wähle das Template basierend auf dem `article_type`:

| Artikel Typ | Template |
|---|---|
| Listicle | `## Listicle Template` |
| How-To | `## How-To Template` |
| Leitfaden | `## Leitfaden Template` |
| Comparison | `## Comparison Template` |
| Anbietervergleich | `## Anbietervergleich Template` |
| News & Trends | `## News & Trends Template` |
| Pillar | `## Pillar Template` |

### Schritt 2 — Content Planner

Nutze einen Subagent (oder direkt im Hauptagent bei einzelnen Artikeln):

```
Du bist ein erfahrener SEO Content Planner.

Erstelle eine detaillierte Gliederung für folgenden Artikel:

**Titel**: [TITLE]
**Artikeltyp**: [ARTICLE_TYPE]
**Primary Keyword**: [PRIMARY_KEYWORD]
**Funnel**: [FUNNEL]
**Beschreibung**: [DESCRIPTION]
**Brand**: [BRAND_STATEMENT]
**Territory**: Deutschland
**Language**: Deutsch

## Wettbewerberanalyse
[RESEARCH_OUTPUT]

## Outline-Regeln für diesen Artikeltyp
[OUTLINE_TEMPLATE FÜR DEN ARTIKELTYP]

## Allgemeine Regeln
- Benutze H1 und H2 für Überschriften
- Gib NUR die Gliederung aus, keine einleitenden Worte
- Benutze niemals das Wort "Kapitel" in H2-Überschriften
- Erstelle niemals Titel mit Jahreszahlen
- Jeder H2-Abschnitt soll maximal 4 Unterpunkte haben
- Der Artikel soll ca. 1600 Wörter haben
- Output ausschließlich auf Deutsch

## Output-Format

H1: [Titel]

H2: Einleitung
- Punkt 1
- Punkt 2

H2: [Abschnitt 1]
- Punkt 1
- Punkt 2

...

H2: Fazit
- Punkt 1
- Punkt 2
```

---

## Phase 5: Ergebnisse in Airtable speichern

Für jeden fertig recherchierten und gegliederten Artikel:

### 1. Duplikat-Check vor dem Eintragen

Prüfe via `mcp__Airtable_MCP_Server__list_records` ob der Slug bereits in Article Production existiert:
```
filterByFormula: {Slug} = "[SLUG]"
```

- Falls ein Record gefunden wird: **überspringen** und im Abschluss-Report als "Duplikat" markieren (es sei denn, der User hat explizit "überschreiben" angegeben)
- Falls kein Record gefunden: mit `create_record` fortfahren

### 2. In "Article Production" eintragen

Via `mcp__Airtable_MCP_Server__create_record`:

```json
{
  "Title": "[TITLE]",
  "Slug": "[SLUG]",
  "Artikel Typ": "[TYPE]",
  "Funnel": "[FUNNEL]",
  "Artikel Style": "Sie-Perspektive",
  "Primary Keyword": "[PRIMARY_KEYWORD]",
  "Main Keyword": "[MAIN_KEYWORD]",
  "Keyword Variations": "[VARIATIONS-Wert aus SEO Keywords Tabelle]",
  "MSV": [Agg. Volume aus SEO Keywords Tabelle],
  "Cluster Name": "[CLUSTER_NAME]",
  "Pillar-Slug": "",
  "Competitor Research": "[VOLLSTÄNDIGER RESEARCH-TEXT aus der JSON-Datei — Feld research]",
  "Gliederung": "[OUTLINE]",
  "Domain": "[DOMAIN]",
  "Status": "Outline erstellt"
}
```

> ⚠️ **Hinweis für Competitor Research**: Lies den vollständigen Inhalt des `"research"`-Feldes aus der gespeicherten JSON-Datei (OUTPUT_FILEPATH) und füge ihn ungekürzt ein. Keine Zusammenfassung — der komplette Text.

### 2. Article Ideas Status updaten

Via `mcp__Airtable_MCP_Server__update_records`:
- Setze Status auf "Briefing erstellt" für den entsprechenden Record in Article Ideas

**Optimierung**: Bei mehreren Artikeln `create_record` parallel in Batches von 10 aufrufen.

---

## Phase 6: Abschluss-Report & Überleitung

```
✅ Artikel-Recherche abgeschlossen

📊 Ergebnisse:
- X Artikel recherchiert
- Y Outlines generiert (Z Duplikate übersprungen)

📋 Outlines nach Artikeltyp:
- How-To: X
- Listicle: X
- Leitfaden: X
- ...

📝 Generierte Outlines:
1. [Title] — [Type] | [Funnel]
2. ...
```

Schließe mit folgender Überleitung:

> **Nächster Schritt: Artikel schreiben**
>
> Alle [Y] Outlines sind bereit. Ich kann jetzt direkt mit dem Schreiben der Artikel für den Cluster "[CLUSTER_NAME]" beginnen — vollständige HTML-Artikel mit Brand Voice, FAQs, internen Links, Meta Description und Schema.org.
>
> Soll ich loslegen?

- Antwortet der User mit Ja (oder "ja", "los", "weiter", "schreib", "start"): Starte den `seo-article-writer` Skill für denselben Cluster `[CLUSTER_NAME]` und dieselbe Domain `[DOMAIN]`.
- Antwortet der User mit Nein: Beende den Skill sauber.

---

## Fehlerbehandlung

- **SERP gibt keine Ergebnisse**: Recherche mit WebSearch allein fortsetzen
- **WebFetch schlägt fehl**: URL überspringen, nächste nehmen. Min. 1 URL muss analysiert werden
- **WebSearch Fehler**: Recherche ohne WebSearch fortsetzen (SERP + WebFetch reichen)
- **Subagent gibt keinen strukturierten Output**: Research-Text trotzdem speichern und Outline versuchen
- **Airtable-Fehler**: Einzelne Fehler loggen, Rest weiter eintragen
- **Outline-Template nicht gefunden**: Leitfaden-Template als Fallback verwenden
- **Keine Artikel mit Häkchen "Ausgewählt"**: User informieren und abbrechen

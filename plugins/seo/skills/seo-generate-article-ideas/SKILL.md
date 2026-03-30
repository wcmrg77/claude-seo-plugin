---
name: generate-article-ideas
description: >
  Generiert SEO-Artikelideen für ein Keyword-Cluster aus Airtable. Analysiert pro Keyword die Google-SERP via DataForSEO, identifiziert Content Gaps und erzeugt mit Subagenten 3 Blogartikel-Ideen pro Keyword. Speichert alle Ideen strukturiert in einer Airtable-Tabelle ("Article Ideas").

  Verwende diesen Skill immer wenn der User Artikelideen generieren möchte, einen Content-Plan erstellen will, SEO-Content planen möchte, oder fragt "welche Artikel sollen wir schreiben". Trigger-Phrasen: "generiere Artikelideen", "erstelle Artikelideen", "content ideen für", "welche artikel für", "generate article ideas", "content plan erstellen", "artikel ideen für cluster", "was sollen wir schreiben über", "content gap analyse".
---

# Generate Article Ideas Skill

Dieser Skill analysiert ein Keyword-Cluster aus Airtable, führt per Subagent SERP-Analysen durch und generiert daraus SEO-optimierte Blogartikel-Ideen — gespeichert strukturiert in Airtable.

## Konfiguration (aus Kontext extrahieren)

- **Airtable Base ID**: aus Kontext extrahieren oder User fragen — kein Default
- **Keywords-Tabelle**: `SEO Keywords` (Table ID aus Kontext oder via `list_tables` ermitteln)
- **Ideas-Tabelle**: `Article Ideas` (erstellen falls nicht vorhanden)
- **Domain**: aus Kontext extrahieren (z.B. "talkpilot.io", "effi-nova.de") — kein Default
- **SEO-Workspace**: `/Users/gregorkuhlmann/Desktop/Coding/SEO/[DOMAIN]/`
- **DataForSEO Credentials**: Username `gregor.kuhlmann.bkbm@gmail.com`, Passwort `e92a2a949b8b4e94`

### Brand Statement ermitteln

Das Brand Statement hilft Subagenten, Artikelideen passend zur Marke zu positionieren.

**Reihenfolge:**
1. Prüfe ob `/Users/gregorkuhlmann/Desktop/Coding/SEO/[DOMAIN]/brand.md` existiert → lesen und verwenden
2. Falls nicht: Website der Domain via WebFetch laden (`https://[DOMAIN]`) → Kernaussagen extrahieren
3. Falls WebFetch nur JS-Fehler/Cookie-Banner liefert: Firecrawl-MCP versuchen (falls installiert: `mcp__firecrawl__firecrawl_scrape`)
4. Falls alles scheitert: Brand Statement aus Domain-Name + Cluster-Thema ableiten und User kurz informieren

**Nach der Recherche**: Speichere das ermittelte Brand Statement in `/Users/gregorkuhlmann/Desktop/Coding/SEO/[DOMAIN]/brand.md` (falls Datei noch nicht existierte), damit es beim nächsten Skill-Run direkt verfügbar ist.

---

## Phase 1: Airtable "Article Ideas" Tabelle prüfen/erstellen

Prüfe mit `mcp__airtable__list_tables` ob die Tabelle "Article Ideas" bereits existiert.

**Falls NICHT vorhanden**, erstelle sie mit `mcp__airtable__create_table`:

```json
{
  "name": "Article Ideas",
  "fields": [
    { "name": "Title", "type": "singleLineText" },
    { "name": "Slug", "type": "singleLineText" },
    { "name": "Type", "type": "singleSelect", "options": { "choices": [
      { "name": "How-To" }, { "name": "Listicle" }, { "name": "Leitfaden" },
      { "name": "Comparison" }, { "name": "Anbietervergleich" }, { "name": "News & Trends" }
    ]}},
    { "name": "Funnel", "type": "singleSelect", "options": { "choices": [
      { "name": "TOFU" }, { "name": "MOFU" }, { "name": "BOFU" }
    ]}},
    { "name": "Description", "type": "multilineText" },
    { "name": "Reasoning", "type": "multilineText" },
    { "name": "Intention", "type": "multilineText" },
    { "name": "Primary Keyword", "type": "singleLineText" },
    { "name": "Main Keyword", "type": "singleLineText" },
    { "name": "Domain", "type": "singleLineText" },
    { "name": "Status", "type": "singleSelect", "options": { "choices": [
      { "name": "Idee" }, { "name": "Briefing erstellt" },
      { "name": "Artikel in Arbeit" }, { "name": "Veröffentlicht" }
    ]}},
    { "name": "Ausgewählt", "type": "checkbox", "options": { "icon": "check", "color": "greenBright" }, "description": "Anhaken um diesen Artikel für die Recherche-Phase auszuwählen." }
  ]
}
```

**Falls vorhanden**: Weiter mit Phase 2.

---

## Phase 2: Keywords für das Cluster laden

Frage den User nach dem gewünschten Cluster (Main Keyword), oder nimm es aus dem Kontext.

Lade alle Keywords für dieses Cluster aus Airtable:
- `mcp__airtable__list_records` auf die SEO Keywords Tabelle
- Filtere nach `Main Keyword == [gewähltes Cluster]`
- Ergebnis: Liste von Primary Keywords mit ihren Metadaten

**Priorisierung**: Sortiere Keywords nach `Agg. Volume` (absteigend). Bei mehr als 10 Keywords im Cluster: nimm die Top 10 (volumenstärksten).

**Workspace vorbereiten**: Erstelle den Workspace-Ordner für Subagent-Outputs:
```bash
WORKSPACE="/Users/gregorkuhlmann/Desktop/Coding/SEO/[DOMAIN]/article-ideas-workspace"
mkdir -p "$WORKSPACE"
```

**Dateinamen vorab berechnen**: Erstelle für jedes Keyword einen sanitized Dateinamen (per Python):
```python
import re

def sanitize_filename(keyword):
    # Umlaute ersetzen
    replacements = {'ä': 'ae', 'ö': 'oe', 'ü': 'ue', 'ß': 'ss',
                    'Ä': 'Ae', 'Ö': 'Oe', 'Ü': 'Ue'}
    for umlaut, replacement in replacements.items():
        keyword = keyword.replace(umlaut, replacement)
    return re.sub(r'[^a-z0-9]+', '_', keyword.lower().strip()).strip('_')
```
Beispiel: `"wärmepumpe köln"` → `waermepumpe_koeln.json`

Speichere die Zuordnung `{keyword: filename}` für die Subagent-Prompts.

---

## Phase 3: Subagenten starten (parallel)

Starte für **jedes Keyword gleichzeitig** einen Subagenten. Nutze das Agent-Tool.

**WICHTIG**: Der komplette Subagent-Prompt wird inline mitgegeben — kein Verweis auf externe Dateien.

### Subagent-Prompt-Template

Ersetze die Platzhalter `[MAIN_KEYWORD]`, `[PRIMARY_KEYWORD]`, `[BRAND_STATEMENT]`, `[OUTPUT_FILEPATH]` vor dem Senden:

```
Du bist ein Senior SEO-Content-Stratege. Deine Aufgabe: Analysiere SERP-Ergebnisse für ein spezifisches Keyword und generiere daraus 3 konkrete Blogartikel-Ideen.

**Main Keyword**: [MAIN_KEYWORD]
**Primary Keyword**: [PRIMARY_KEYWORD]
**Brand Statement**: [BRAND_STATEMENT]

---

## Schritt 1 — SERP-Analyse via DataForSEO

Rufe die SERP-Ergebnisse ab:

```bash
curl -s -X POST "https://api.dataforseo.com/v3/serp/google/organic/live/advanced" \
  -u "gregor.kuhlmann.bkbm@gmail.com:e92a2a949b8b4e94" \
  -H "Content-Type: application/json" \
  -d '[{"keyword": "[PRIMARY_KEYWORD]", "language_code": "de", "location_code": 2276, "depth": 10}]'
```

Extrahiere die Top-5 organischen Ergebnisse (type: "organic"):
- rank_absolute, title, url, description

Falls DataForSEO keine Ergebnisse liefert: Überspringe Schritt 2 und generiere Ideen nur aus Keyword-Kontext.

## Schritt 2 — Artikel-Inhalte analysieren

Rufe für die Top-3 URLs den Seiteninhalt ab (nutze WebFetch).
Extrahiere: Hauptthemen, Überschriften (H2/H3), Kernaussagen.
Falls eine URL nicht erreichbar ist oder WebFetch nur Cookie-Banner / JS-Fehler zurückgibt: überspringen und nächste nehmen.

## Schritt 3 — Analyse (intern, nicht ausgeben)

Bevor du Ideen generierst, analysiere:
1. **Suchintention**: Welche Nutzerabsicht haben diese Ergebnisse?
2. **Unterthemen**: Welche Themenbereiche decken die Top-Artikel ab?
3. **Content Gaps**: Was fehlt? Welche wichtigen Aspekte werden nicht behandelt?
4. **Brand Fit**: Passt jede Idee zur Firma?

## Schritt 4 — 3 Artikelideen generieren

Generiere **genau 3 Blogartikel-Ideen** — eine pro Funnel-Phase (TOFU, MOFU, BOFU) — die:
- Die identifizierte Suchintention optimal treffen
- Content Gaps der bestehenden Artikel schließen
- Die Firma als Experten positionieren — ohne den Firmennamen im Titel zu nennen
- Keine Jahreszahlen im Titel enthalten
- Wenn das Keyword einen lokalen Bezug hat, diesen aufgreifen
- Thematisch klar unterschiedlich sind (kein Überlappen von Haupt-Subtopics)

**Artikeltypen** (nur diese verwenden):
How-To · Listicle · Leitfaden · Comparison · Anbietervergleich · News & Trends

**Funnel-Phasen**:
- TOFU (Bewusstsein): Breit, informierend, Problembewusstsein wecken
- MOFU (Interesse): Vergleich, Lösungssuche, Abwägung
- BOFU (Kaufentscheidung): Produktspezifisch, transaktional, überzeugend

## Schritt 5 — Ergebnis speichern

Speichere das Ergebnis als JSON in genau diese Datei:
**[OUTPUT_FILEPATH]**

Das JSON MUSS exakt dieses Format haben — der Key MUSS "ideas" sein (englisch, lowercase):

```json
{
  "ideas": [
    {
      "title": "Klickstarker, SEO-optimierter Titel",
      "description": "Kurze Inhaltsbeschreibung (1-2 Sätze)",
      "type": "How-To|Listicle|Leitfaden|Comparison|Anbietervergleich|News & Trends",
      "funnel": "TOFU|MOFU|BOFU",
      "reasoning": "Warum trifft dieser Artikel die Suchintention? Warum dieser Artikeltyp?",
      "intention": "Wie verhält sich dieser Artikel zur Suchintention der Konkurrenz?",
      "slug": "url-freundlicher-slug-ohne-sonderzeichen"
    }
  ]
}
```

KRITISCH: Der JSON-Root-Key muss `"ideas"` heißen. Nicht "Artikelideen", nicht "article_ideas", nicht "results". Exakt `"ideas"`.

Gib am Ende NUR das JSON aus (kein anderer Text).
```

---

## Phase 4: Ergebnisse einsammeln und in Airtable eintragen

Sobald alle Subagenten fertig sind, führe ein **Python-Batch-Script** aus das alle Ergebnisse sammelt und vorbereitet:

```python
import json, os
from difflib import SequenceMatcher

WORKSPACE = "/Users/gregorkuhlmann/Desktop/Coding/SEO/[DOMAIN]/article-ideas-workspace"
MAIN_KEYWORD = "[CLUSTER_NAME]"
DOMAIN = "[DOMAIN]"

# Keyword-zu-Datei-Mapping (aus Phase 2 übernehmen)
keyword_file_map = {
    # "wärmepumpe altbau": "waermepumpe_altbau.json",
    # ... wird in Phase 2 generiert
}

all_ideas = []
errors = []

for keyword, filename in keyword_file_map.items():
    filepath = os.path.join(WORKSPACE, filename)
    if not os.path.exists(filepath):
        errors.append(f"MISSING: {filename} für Keyword '{keyword}'")
        continue

    try:
        with open(filepath) as f:
            data = json.load(f)
    except json.JSONDecodeError as e:
        errors.append(f"INVALID JSON: {filename} — {e}")
        continue

    # Tolerantes Key-Parsing: akzeptiere mehrere Varianten
    ideas = (
        data.get("ideas") or
        data.get("Artikelideen") or
        data.get("article_ideas") or
        data.get("results") or
        []
    )

    if not ideas:
        errors.append(f"NO IDEAS: {filename} — kein bekannter Key gefunden. Keys: {list(data.keys())}")
        continue

    for idea in ideas:
        idea["_primary_keyword"] = keyword
    all_ideas.extend(ideas)

# Semantischer Duplikat-Check (zusätzlich zu Slug-Dedup)
# Zwei Ideen gelten als semantische Duplikate wenn:
# a) ihre Slugs zu >80% übereinstimmen, ODER
# b) ihre Titel zu >75% übereinstimmen UND gleiches Funnel-Level
def similarity(a, b):
    return SequenceMatcher(None, a.lower(), b.lower()).ratio()

unique_ideas = []
for idea in all_ideas:
    is_duplicate = False
    for existing in unique_ideas:
        slug_sim = similarity(idea.get("slug", ""), existing.get("slug", ""))
        title_sim = similarity(idea.get("title", ""), existing.get("title", ""))
        same_funnel = idea.get("funnel") == existing.get("funnel")
        if slug_sim > 0.80 or (title_sim > 0.75 and same_funnel):
            is_duplicate = True
            print(f"SEMANTIC-DUP: '{idea.get('title','?')[:50]}' ~ '{existing.get('title','?')[:50]}'")
            break
    if not is_duplicate:
        unique_ideas.append(idea)

# Report
print(f"Gesammelt: {len(all_ideas)} Ideen aus {len(keyword_file_map)} Keywords")
print(f"Nach Duplikat-Filter: {len(unique_ideas)} unique Ideen")
if errors:
    print(f"\n⚠️ {len(errors)} Fehler:")
    for e in errors:
        print(f"  - {e}")

# Ausgabe zur Kontrolle
for i, idea in enumerate(unique_ideas):
    print(f"{i+1}. [{idea.get('funnel','?')}] {idea.get('title','?')[:60]}... | kw: {idea.get('_primary_keyword','?')}")
```

### Airtable-Eintragung

Lade bestehende Slugs via `mcp__airtable__list_records` (maxRecords: 500, Felder: Slug, Title) für den Duplikat-Check.

**Duplikat-Check (zweistufig)**:
1. **Slug-Dedup**: Überspringe Ideen deren `Slug` bereits in der Tabelle existiert (exakter Match)
2. **Semantischer Dedup**: Wurde im Python-Script oben bereits durchgeführt

Dann für jede Idee: `mcp__airtable__create_record` aufrufen mit:

```json
{
  "Title": "[title]",
  "Slug": "[slug]",
  "Type": "[type]",
  "Funnel": "[funnel]",
  "Description": "[description]",
  "Reasoning": "[reasoning]",
  "Intention": "[intention]",
  "Primary Keyword": "[das Keyword des Subagenten]",
  "Main Keyword": "[das Cluster]",
  "Domain": "[DOMAIN]",
  "Status": "Idee"
}
```

**Optimierung**: Rufe `create_record` für mehrere Ideen **parallel** auf (mehrere Tool-Calls in einer Nachricht). Gruppiere in Batches von 10.

---

## Phase 5: Abschluss-Report + Skill-Chaining

```
✅ Artikelideen generiert

📊 Ergebnisse:
- X Keywords analysiert
- Y Artikelideen generiert (Z Duplikate übersprungen)

📋 Ideen nach Funnel:
- TOFU: X Ideen
- MOFU: X Ideen
- BOFU: X Ideen

🏆 Top-Ideen (BOFU + MOFU):
1. [Title] — [Type] | [Funnel]
2. ...
```

**Skill-Chaining**: Nach dem Report fragen:

> "Welche Idee soll ich als erstes für den Article Research Skill aufbereiten? (Titel oder Nummer nennen)"

Wenn der User eine Idee nennt: Den Article Research Skill für diese Idee starten (falls vorhanden), oder dem User mitteilen welche Infos er als nächstes braucht (Title, Slug, Primary Keyword, Brand Statement).

---

## Fehlerbehandlung

- **SERP gibt keine Ergebnisse**: Keyword trotzdem verarbeiten — Subagent generiert Ideen basierend auf Keyword-Kontext allein
- **WebFetch gibt nur Cookie-Banner/JS-Fehler**: Firecrawl-MCP versuchen (`mcp__firecrawl__firecrawl_scrape`). Falls Firecrawl nicht installiert: URL überspringen, nächste nehmen
- **Subagent gibt kein valides JSON**: JSON mit `json.loads()` parsen und bei Fehler das Keyword loggen, aber nicht abbrechen
- **Falscher JSON-Key**: Phase 4 akzeptiert tolerant: `ideas`, `Artikelideen`, `article_ideas`, `results`
- **Datei fehlt**: Keyword in Fehlerliste aufnehmen, Rest weiter verarbeiten
- **Airtable-Fehler beim Eintragen**: Einzelne Fehler loggen, Rest weiter eintragen
- **brand.md nicht vorhanden und WebFetch schlägt fehl**: Brand Statement aus Domainname + Cluster-Thema ableiten, User kurz informieren, danach brand.md anlegen

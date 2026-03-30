---
name: seo-keyword-research
description: >
  SEO Keyword-Recherche Skill: Recherchiert Keywords für eine Domain via DataForSEO, ermittelt Suchvolumen und Keyword Difficulty, und speichert alles strukturiert in Airtable. Erstellt die Airtable-Tabelle automatisch, falls sie noch nicht existiert.

  Verwende diesen Skill immer wenn der User SEO Keywords recherchieren möchte, eine Keyword-Datenbank aufbauen oder erweitern will, Keywords für eine bestimmte Nische, Branche oder Stadt sucht, oder eine Website für SEO analysieren möchte. Trigger-Phrasen: "recherchiere Keywords für", "suche SEO Keywords", "keyword research", "Keywords für die Datenbank", "trag Keywords in Airtable ein", "finde Keywords mit Suchvolumen", "was sind gute Keywords für", "erweitere die Keyword-Datenbank", "keyword analyse", "SEO analyse".

  Triggere auch wenn der User eine spezifische Nische nennt (z.B. "Handwerk Keywords", "lokale Keywords für Köln", "Keywords für voice bot") und nach Suchvolumen oder Rankings fragt.
---

# SEO Keyword Research Skill

Dieser Skill führt eine vollständige Keyword-Recherche durch: von der Keyword-Ideenfindung via DataForSEO bis zur strukturierten Ablage in Airtable.

## Übersicht

Der Workflow läuft in 4 Phasen:
1. **Setup** – Airtable-Tabelle prüfen/erstellen
2. **Discovery** – Keywords via DataForSEO finden
3. **Enrichment** – Suchvolumen + KD via Google Ads API ermitteln
4. **Storage** – Keywords in Airtable eintragen (Duplikate vermeiden)

---

## Phase 1: Airtable Setup

### Konfigurationsvariablen (aus User-Anfrage extrahieren oder Defaults nutzen)
- **Airtable Base ID**: aus Kontext oder beim User erfragen
- **Table Name**: "SEO Keywords" (Default)
- **Domain**: die Zieldomain (z.B. "talkpilot.io")

### Tabelle prüfen
Nutze `mcp__airtable__list_tables` um zu prüfen, ob die Tabelle bereits existiert.

**Falls die Tabelle existiert**: Weiter mit Phase 2. Prüfe mit `mcp__airtable__list_records` kurz, wie viele Records bereits vorhanden sind (für Duplikat-Check später).

**Falls die Tabelle NICHT existiert**: Erstelle sie mit `mcp__airtable__create_table` mit folgendem Schema:

```json
{
  "name": "SEO Keywords",
  "fields": [
    { "name": "Primary Keyword", "type": "singleLineText" },
    { "name": "Agg. Volume", "type": "number", "options": { "precision": 0 } },
    { "name": "KD", "type": "number", "options": { "precision": 0 } },
    { "name": "Variations", "type": "multilineText" },
    { "name": "Main Keyword", "type": "singleLineText" },
    { "name": "Domain", "type": "singleLineText" },
    { "name": "Status", "type": "singleSelect", "options": {
      "choices": [
        { "name": "Recherchiert" },
        { "name": "Briefing erstellt" },
        { "name": "Artikel in Arbeit" },
        { "name": "Veröffentlicht" }
      ]
    }}
  ]
}
```

---

## Phase 2: Keyword Discovery via DataForSEO

### Verfügbare DataForSEO-Methoden
Nutze den DataForSEO MCP Server. Wichtig: Immer auf dem **deutschen Markt** recherchieren (`language_code: "de"`, `location_code: 2276` für Deutschland).

**Für domain-basierte Recherche** (wenn eine URL angegeben ist):
- Methode: `keywords_for_site` – gibt Keywords zurück, für die die Domain bereits rankt oder ranken könnte

**Für keyword-basierte Recherche** (wenn Seed-Keywords angegeben sind):
- Methode: `keyword_suggestions` – erweitert ein Seed-Keyword um verwandte Terms
- Methode: `related_keywords` – findet semantisch verwandte Keywords

**Für Suchvolumen-Daten**:
- Methode: `google_ads_search_volume` – liefert monatliche Suchvolumen aus Google Ads

### Recherche-Strategie

**Schritt 2a: Seed-Keywords definieren**
Falls der User Seed-Keywords angibt → diese direkt nutzen.
Falls nur eine Domain angegeben → zuerst `keywords_for_site` aufrufen, die Top-Keywords als Seeds verwenden.

**Schritt 2b: Keyword-Variationen finden**
Für jedes Seed-Keyword: `keyword_suggestions` oder `related_keywords` aufrufen.
Ziel: 30-50 Kandidaten pro Seed-Keyword sammeln.

**Mindestgröße pro Cluster: 3–4 Keywords**
Wenn ein Seed-Keyword weniger als 3 verwertbare Keywords (SV ≥ 10) liefert:
- Variante 1: Mit verwandten Seed-Keywords nachbohren (z.B. Kurzform, Synonym)
- Variante 2: Cluster in einen thematisch benachbarten einmergen
- Variante 3: Cluster als "dünn" markieren und dem User melden
Nie einen Cluster mit nur 1–2 Keywords in die Tabelle schreiben.

**Schritt 2c: Lokale und branchenspezifische Variationen**
Falls der User lokale Keywords (Städte) oder Branchen-Keywords möchte:
- Kombiniere Seed + Stadtname (z.B. "voice bot" + "köln", "düsseldorf", "hamburg")
- Kombiniere Seed + Branche (z.B. "ki telefonassistent" + "handwerk", "arztpraxis")
- Rufe `google_ads_search_volume` für diese Kombinationen auf

---

## Phase 3: Volumen-Enrichment und Filterung

### Suchvolumen prüfen
Rufe `google_ads_search_volume` für alle Kandidaten-Keywords auf.

**Filterregeln:**
- Nur Keywords mit **SV ≥ 10** eintragen (Keywords mit SV=0 überspringen, außer User gibt explizit SV=0 frei)
- Keywords mit **KD > 85** markieren (schwer zu ranken, aber trotzdem eintragen)

### KD-Enrichment für fehlende Werte
`keyword_suggestions` liefert manchmal keine KD-Werte (Feld fehlt oder = 0). In diesem Fall:
- Rufe `dataforseo_labs_bulk_keyword_difficulty` für alle betroffenen Keywords auf (DE, Sprache DE)
- Trage die KD-Werte nach bevor du in Airtable schreibst
- Nur wenn auch bulk_keyword_difficulty kein Ergebnis liefert → KD=0 in Airtable und im Report vermerken

### Clustering (Main Keyword zuweisen)
Weise jedem Keyword ein **Main Keyword** zu – das ist das übergeordnete Thema-Cluster:
- Gruppe Keywords nach semantischer Nähe
- Das Main Keyword ist typischerweise das breiteste, volumenstärkste Keyword im Cluster
- Beispiel: "ki telefonassistent handwerk" → Main Keyword: "ki telefonassistent"

### Variations-Feld (WICHTIG)
Das **Variations-Feld** darf NUR Begriffe enthalten, die in Google **exakt die gleichen SERP-Ergebnisse** liefern wie das Primary Keyword.

Das sind typischerweise:
- Schreibvarianten: "voice bot" ↔ "voicebot" (wenn SERPs identisch)
- Echte Synonyme die Google gleich behandelt

**NICHT als Variations eintragen:**
- Verwandte Keywords (auch wenn semantisch nah)
- Keywords mit anderem Intent
- Keywords für die du nicht sicher bist, ob die SERPs identisch sind

Im Zweifel: **kein Variations-Eintrag** – lieber leer lassen als spekulieren.

### Variations systematisch prüfen (via core_keyword)

DataForSEO's `keyword_suggestions` liefert für jedes Keyword ein `core_keyword`-Feld. Zwei Keywords mit **identischem `core_keyword`** zeigen auf dieselben SERPs → eines wird Primary Keyword, das andere kommt ins Variations-Feld.

**Vorgehen:**
1. Nach dem Abrufen aller Keyword-Vorschläge: Keywords nach `core_keyword` gruppieren
2. Innerhalb jeder Gruppe: das volumenstärkste Keyword als Primary Keyword wählen
3. Alle anderen Keywords derselben Gruppe → kommagetrennt ins Variations-Feld des Primary Keywords
4. Diese Keywords **nicht** als eigene Records eintragen (sie wären Duplikate auf SERP-Ebene)

**Beispiel:**
```
core_keyword: "wärmepumpe kosten"
→ Primary: "wärmepumpe kosten" (SV: 9900)
→ Variation: "kosten wärmepumpe" (SV: 1300) — gleicher core_keyword, gleiche SERPs
```

Wenn `core_keyword` fehlt oder leer ist → wie bisher: kein Variations-Eintrag, lieber leer lassen.

---

## Phase 4: Airtable Storage

### Duplikat-Check
Bevor du Keywords einträgst: Prüfe die existierenden Records mit `mcp__airtable__list_records`.
Überspringe Keywords, die bereits als "Primary Keyword" in der Tabelle stehen (Case-insensitive Vergleich).

### Batch-Eintragung
Trage Keywords ein mit `mcp__airtable__create_record`:

```json
{
  "Primary Keyword": "beispiel keyword",
  "Agg. Volume": 1200,
  "KD": 35,
  "Variations": "",
  "Main Keyword": "beispiel",
  "Domain": "example.com",
  "Status": "Recherchiert"
}
```

Trage maximal 10 Keywords pro Batch ein, um Fehler zu vermeiden. Bei großen Mengen: nach jedem Batch kurz Status berichten.

---

## Abschluss-Report

Nach dem Eintragen, berichte kompakt:

```
✅ Keyword-Recherche abgeschlossen

📊 Ergebnisse:
- X Keywords gefunden, Y eingetragen (Z Duplikate übersprungen)
- Ø Search Volume: [Durchschnitt]
- Main Keywords abgedeckt: [Liste]

🏆 Top-Opportunities (SV hoch, KD niedrig):
1. [Keyword] – SV: X, KD: Y
2. ...

⚠️ Schwierige Keywords (KD > 75, aber hohes Volumen):
1. [Keyword] – SV: X, KD: Y
```

---

## Häufige Szenarien

### Szenario A: "Recherchiere Keywords für talkpilot.io"
→ Phase 1 (Tabelle prüfen) → `keywords_for_site` für Domain → Cluster bilden → Volumen prüfen → Eintragen

### Szenario B: "Finde Keywords zum Thema 'voice bot'"
→ Phase 1 → `keyword_suggestions` für "voice bot" → Volumen prüfen → Eintragen

### Szenario C: "Lokale Keywords für Köln"
→ Phase 1 (Tabelle existiert bereits?) → Kombinationen mit Städten bilden → `google_ads_search_volume` → Nur neue Keywords eintragen

### Szenario D: "Branchenspezifische Keywords für Handwerk"
→ Phase 1 → Seed + Branchen kombinieren → Volumen prüfen → Nur SV≥10 eintragen

---

## Fehlerbehandlung

- **DataForSEO gibt 0 Ergebnisse zurück**: Alternative Seed-Keywords versuchen oder User nach Synonymen fragen
- **Airtable-Fehler**: Bei Fehler einzelner Records → Rest trotzdem eintragen, Fehler am Ende auflisten
- **Rate Limits**: Bei DataForSEO-Rate-Limits: 2-3 Sekunden warten und wiederholen
- **DataForSEO-Response zu groß (>100K Zeichen)**: Das Tool speichert das Ergebnis automatisch in einer Datei unter `~/.claude/projects/.../tool-results/`. Extrahiere die Keywords mit Python:
  ```bash
  python3 -c "
  import json
  with open('/pfad/zur/datei.txt') as f:
      data = json.load(f)
  items = json.loads(data[0]['text'])['items']
  for item in items:
      sv = item.get('keyword_info',{}).get('search_volume',0)
      kd = item.get('keyword_properties',{}).get('keyword_difficulty',0)
      kw = item.get('keyword','')
      if sv >= 10:
          print(f'SV:{sv:6} KD:{kd:3} | {kw}')
  " | sort -rn
  ```
  Dies ist ein bekanntes, häufig auftretendes Verhalten — keine Ausnahme, sondern der normale Umgang mit großen Responses.

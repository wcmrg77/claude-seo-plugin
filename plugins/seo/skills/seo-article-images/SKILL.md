---
name: seo-article-images
description: >
  Generiert 4 SEO-optimierte Bilder pro Blogartikel via Kie AI (NanoBanana2). Liest Artikel mit Status "Draft erstellt" aus der Airtable Article Production Tabelle, erstellt Bild-Prompts basierend auf Titel und Gliederung, speichert Bilder lokal und schreibt Alt-Texte in Airtable.

  Trigger-Phrasen: "generiere Bilder für die Artikel", "Artikel-Bilder erstellen", "SEO Bilder", "Bilder für den Blog", "Bilder für Cluster", "article images", "generate blog images", "Artikel bebildern". Auch aktiv vorschlagen wenn der Article Writer gerade fertig wurde.
---

# SEO Article Images Skill

Generiert **4 Bilder pro Artikel** via **Kie AI NanoBanana2**. Arbeitet mit Artikeln aus der Article Production Tabelle mit Status **"Draft erstellt"** — direkt nach dem seo-article-writer Skill.

**Workflow-Einordnung:**
```
seo-article-research → seo-article-writer → seo-article-images (dieser Skill)
```

---

## Phase 0: Brand & Airtable Resolution

### Schritt 0.1: Domain bestimmen

Falls der User eine Domain genannt hat, nutze diese. Ansonsten: Lade alle einzigartigen `Domain`-Werte aus den Artikeln mit Status "Draft erstellt" und frage per AskUserQuestion.

Speichern als: `[DOMAIN]` (z.B. `effi-nova.de`, `talkpilot.io`)

### Schritt 0.2: Airtable Base finden

Rufe `mcp__airtable__list_bases` auf. Suche nach einer Base, deren Name die Domain oder einen Teil davon enthält (z.B. "effi-nova" → Base "effi-nova.de", "talkpilot" → Base "TalkPilot SEO").

Speichern als: `[BASE_ID]`

### Schritt 0.3: Tabellen-IDs ermitteln

Rufe `mcp__airtable__list_tables` mit `[BASE_ID]` auf. Finde:
- Die Tabelle "Article Production" → `[ARTICLE_PRODUCTION_TABLE_ID]`

### Schritt 0.4: Lokaler Basis-Pfad

```
[LOCAL_BASE_PATH] = /Users/gregorkuhlmann/Desktop/Coding/SEO/[DOMAIN]
[IMAGES_PATH] = [LOCAL_BASE_PATH]/images
```

---

## Phase 1: Artikel laden & bestätigen

### Schritt 1.1: Cluster bestimmen

Falls der User einen Cluster (Main Keyword) genannt hat, verwende diesen direkt.

Ansonsten: Lade alle einzigartigen `Main Keyword`-Werte aus Article Production mit `{Status} = "Draft erstellt"` und frage per AskUserQuestion.

### Schritt 1.2: Artikel laden

```
filterByFormula: AND({Status} = "Draft erstellt", {Main Keyword} = "[CLUSTER_NAME]")
maxRecords: 10
```

Felder laden:
- `Title`, `Slug`, `Primary Keyword`, `Keyword Variations`, `Gliederung`, `Domain`
- Airtable Record ID

Falls kein Ergebnis: User informieren und abbrechen.

### Schritt 1.3: Bestätigung

Zeige:
- Cluster-Name + Anzahl Artikel
- Liste: Titel + Primary Keyword pro Artikel
- Gesamt-Bilder: N × 4

Frage ob gestartet werden soll.

---

## Phase 2: Bulk-Bildgenerierung

### Schritt 2.1: Alle Prompts generieren (alle Artikel auf einmal)

Erstelle für **jeden Artikel** 4 Bild-Prompts, bevor die erste Generierung startet.

**Prompt-Generierung — System-Prompt:**
```
You are a prompt writer for the image generator Kie AI (NanoBanana2).

Instructions:
- Write 4 image generation prompts for the given article topic.
- Base each prompt on the provided article title and outline.
- Every prompt must include: "do not include any words, text, letters, numbers, or typography in the image."
- Every prompt must be stock photography style with natural lighting: "Natural daylight illuminates the scene."
- Photos: shallow depth of field, fast shutter speed to freeze dynamic motion, natural style, not intense or emotional.
- NEVER show software interfaces, app screens, dashboards, or UI elements.
- If computers or devices are shown: only from behind or side angle — never show the screen.
- Never show people, human faces, hands, or body parts.
- Never name a specific artist or photographer.
- No single or double quotes in the prompt, no special characters that may cause JSON issues.
- Each of the 4 prompts must focus on a DIFFERENT visual concept:
  - Image 1 (Hero): The main concept of the article visualized abstractly — wide shot, environmental
  - Image 2 (Detail): A detail shot related to a key subtopic from the outline
  - Image 3 (Context): A contextual/environmental shot that sets the scene for the topic
  - Image 4 (Result): An abstract or metaphorical visualization of the article's core benefit or outcome

Output format — return ONLY a valid JSON object, no markdown, no code fences:
{"image_1": "prompt here", "image_2": "prompt here", "image_3": "prompt here", "image_4": "prompt here"}
```

**User-Prompt:**
```
Article Title: [TITLE]
Primary Keyword: [PRIMARY_KEYWORD]
Keyword Variations: [KEYWORD_VARIATIONS]

Outline:
[GLIEDERUNG]
```

**JSON parsen:** Falls kein valides JSON, Code-Fences entfernen und erneut versuchen.

---

### Schritt 2.2: Alle Tasks auf einmal starten (Bulk-Start)

Starte **alle Tasks aller Artikel gleichzeitig** in einem einzigen Tool-Call-Block — nicht Artikel für Artikel.

Für jeden Prompt: `mcp__kie-ai__nano_banana_image` aufrufen:

```json
{
  "prompt": "[GENERATED_PROMPT]",
  "model": "nano_banana_2",
  "aspect_ratio": "16:9",
  "resolution": "2K",
  "output_format": "png"
}
```

Falls der `model`-Parameter nicht akzeptiert wird, den Parameter weglassen.

**Task-Tracking:** Sofort nach dem Start eine interne Tabelle anlegen:
```
task_id → {slug, img_nr (1-4)}
```
Diese Zuordnung für alle folgenden Schritte beibehalten.

---

### Schritt 2.3: Status pollen (globaler Loop für alle Tasks)

Prüfe alle offenen Tasks parallel alle **30 Sekunden** via `mcp__kie-ai__get_task_status`.

Wiederhole bis alle Tasks `status = "completed"` haben und eine Bild-URL vorliegt.

---

### Schritt 2.4: Alle Bilder herunterladen

Alle abgeschlossenen Downloads parallel starten:

1. **Ordner erstellen (alle auf einmal):**
   ```bash
   mkdir -p "[IMAGES_PATH]/[SLUG_1]" "[IMAGES_PATH]/[SLUG_2]" ...
   ```

2. **Alle Bilder parallel herunterladen:**
   ```bash
   curl -sL "[URL]" -o "[IMAGES_PATH]/[SLUG]/[SLUG]-1.png" &
   curl -sL "[URL]" -o "[IMAGES_PATH]/[SLUG]/[SLUG]-2.png" &
   ...
   wait
   ```

3. **Prüfen:** Jede Datei sollte > 50 KB sein. Kleinere Dateien als Fehler loggen.

---

### Schritt 2.5: Alt-Texte generieren

Erstelle für jedes der 4 Bilder pro Artikel einen prägnanten deutschen Alt-Text (max. 120 Zeichen):
- Enthält das Primary Keyword natürlich
- Beschreibt was auf dem Bild zu sehen ist
- Kein Keyword-Stuffing

Format:
```
[SLUG]-1.png: [ALT_TEXT_1]
[SLUG]-2.png: [ALT_TEXT_2]
[SLUG]-3.png: [ALT_TEXT_3]
[SLUG]-4.png: [ALT_TEXT_4]
```

---

### Schritt 2.6: Airtable aktualisieren

Via `mcp__airtable__update_records` alle Article Production Records aktualisieren (bis zu 10 pro Call):

```json
{
  "fields": {
    "alt_texte": "[ALLE_ALT_TEXTE als mehrzeiliger String]",
    "Status": "Bilder erstellt"
  }
}
```

**Hinweis:** Falls "Bilder erstellt" noch kein gültiger Status-Wert ist, zuerst via `mcp__airtable__update_field` den Status-SelectField erweitern.

---

## Phase 3: Abschluss-Report

```
Bilder generiert — Cluster "[CLUSTER_NAME]"

Verarbeitete Artikel: [N]

[Pro Artikel:]
[TITLE]
  Keyword: [PRIMARY_KEYWORD]
  Bilder: 4 generiert
  Dateien: [IMAGES_PATH]/[SLUG]/
  Airtable: Status → "Bilder erstellt"

Alle Bilder wurden in [IMAGES_PATH]/ gespeichert.
```

---

## Fehlerbehandlung

- **Kie AI Task fehlgeschlagen:** Einmal mit leicht modifiziertem Prompt wiederholen ("Stock photography: " voranstellen). Bei erneutem Fehler: Überspringen und im Report markieren.
- **NanoBanana2 nicht verfügbar:** `model`-Parameter weglassen und mit Standard fortfahren.
- **Download schlägt fehl:** Task-Status nochmal laden, URL erneut versuchen. Falls weiterhin fehlgeschlagen: Als fehlend markieren.
- **JSON aus Prompt-Generierung ungültig:** Code-Fences und Sonderzeichen bereinigen, erneut parsen. Falls weiterhin ungültig: Prompts als Plaintext extrahieren.
- **Artikel ohne Gliederung:** Prompt nur aus Titel + Keywords generieren.

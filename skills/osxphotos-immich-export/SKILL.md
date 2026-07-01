# osxphotos Export für Immich-Migration

## Kontext
Michael migriert ~16.000 Fotos von Apple Photos auf selbst gehostete Immich-Instanz (datenbank.m-krueger.de). Fotos-Bibliothek liegt lokal. Nur in einigen **geteilten Alben** fehlen Bilder, die aus iCloud nachgeladen werden müssen. Priorität: EXIF-Metadatenintegrität (Datum, GPS), da Immich diese Felder direkt aus der Datei liest, nicht aus separaten Sidecars.

## Setup
- osxphotos läuft in eigenem venv: `osxphotos-env`
- Version: 0.76.1 (Stand Juli 2026)
- exiftool wird zwingend benötigt: `brew install exiftool`, danach mit `which exiftool` prüfen

## Funktionierender Export-Befehl
```bash
mkdir -p ~/immich-test  # Zielverzeichnis MUSS vorher existieren, sonst Error
osxphotos export ~/immich-test \
  --album "Wien 2014" \
  --album "Texel" \
  --directory "{album}" \
  --filename "{original_name}" \
  --download-missing \
  --exiftool \
  --verbose
```

## Bekannte Stolperfallen
1. **`--original-name` existiert nicht.** Richtige Option ist `--filename "{original_name}"`. osxphotos schlägt bei falschem Flag Alternativen vor (`--filename`, `--only-new`, `--original-suffix`).
2. **Zielverzeichnis muss vorher existieren** (`mkdir -p` davor), sonst bricht der Export mit "Invalid value for 'DEST'" ab — und die Folgezeilen des Multi-Line-Befehls werden von zsh dann fälschlich als eigene Kommandos interpretiert ("command not found: --directory").
3. **exiftool separat installieren.** osxphotos bringt es nicht mit. Ohne exiftool bricht `--exiftool` mit "Could not find exiftool" ab, egal ob der Rest der Befehlszeile korrekt ist.
4. **`{original_name}` ist nicht immer zuverlässig.** Bei Fotos, die reimportiert oder bearbeitet wurden, kann das Feld leer sein — osxphotos fällt dann auf UUID-Namen zurück. Nach jedem Export-Batch stichprobenartig prüfen (`ls` im Zielordner), ob lesbare Dateinamen rauskommen.

## Feldauswahl bei --exiftool
`--exiftool` hat KEINEN granularen Schalter, um nur bestimmte Felder (z.B. nur Datum + GPS) zu schreiben. Es schreibt ein festes Set: Datum, GPS, Keywords, Description, Title, ggf. Personen als Keywords.

Wichtig für Performance-Einschätzung: Die Laufzeit von `--exiftool` hängt am Subprocess-Aufruf pro Datei, NICHT an der Zahl der geschriebenen Felder. Eine Feldeinschränkung würde also keine relevante Zeitersparnis bringen — dieser Optimierungsgedanke ist ein Holzweg.

Falls wirklich nur einzelne Felder gewünscht sind: `--sidecar XMP` exportieren und per eigenem exiftool-Aufruf (`-DateTimeOriginal -GPSLatitude -GPSLongitude`) gezielt aus dem Sidecar in die Zieldatei schreiben. Für den Normalfall (Immich-Import) unnötig — zusätzliche Felder wie Keywords/Description stören dort nicht.

## Skalierung bei 16.000 Fotos (lokale Bibliothek)
Da die Bibliothek lokal liegt, ist `--download-missing` für den Großteil der Fotos irrelevant. Der eigentliche Flaschenhals ist der exiftool-Durchlauf, nicht das Nachladen aus der Cloud:

- **Laufzeit exiftool:** ~0,3–0,5s Subprocess-Overhead pro Datei → 1,5–2,5h nur für den exiftool-Durchlauf bei 16.000 Dateien, ohne Kopiervorgang.
- **Speicherplatz:** bei realistisch 3–8 MB/Foto (mehr bei RAW/Live Photos) sind 16.000 Fotos 50–130 GB. Vorher `df -h` auf dem Zielvolume checken.
- **Batching statt Gesamtexport in einem Rutsch.** Lieber pro Jahr oder pro großem Album exportieren statt alles in einem Befehl. Bricht der Export ab, will man nicht bei null anfangen. Bei Wiederanlauf `--only-new` nutzen, damit bereits exportierte Dateien übersprungen werden.

### Geteilte Alben mit fehlenden Bildern separat behandeln
Nur einige geteilte Alben haben Originale, die nicht lokal vorliegen. Diese getrennt vom Rest exportieren, damit der Cloud-Download nicht den Export der lokal bereits vorhandenen Masse blockiert.

1. Vorher tatsächliche Zahl der fehlenden Bilder in geteilten Alben prüfen:
   ```bash
   osxphotos query --shared --missing
   ```
2. Erst die lokalen (nicht-geteilten) Alben exportieren — läuft schnell, kein Cloud-Download nötig.
3. Danach die geteilten Alben mit fehlenden Originalen separat mit `--download-missing` exportieren — hier ist Apples Rate-Limit der Flaschenhals, kann je nach Menge deutlich länger dauern.

## Qualitätscheck vor Gesamt-Import
Vor dem vollen Batch immer erst an 2-3 Testalben validieren:
```bash
exiftool -DateTimeOriginal -GPSLatitude -GPSLongitude ~/immich-test/<Album>/<Datei>
```
Prüfen ob Datum und GPS korrekt gesetzt sind, bevor der Gesamtimport läuft.

## Nächster Schritt (Stand dieser Session)
Test-Export lief erfolgreich nach Fix von exiftool-Installation. Nächster Schritt: `osxphotos query --shared --missing` laufen lassen um Umfang der geteilten-Alben-Downloads zu kennen, Testalben validieren, dann zweigeteilte Export-Strategie (lokal / geteilt+missing) fahren, dann Immich CLI Upload mit kleinem Batch vor Gesamt-Import.

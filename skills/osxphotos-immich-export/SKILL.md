# osxphotos Export für Immich-Migration

## Kontext
Michael migriert ~16.000 Fotos von Apple Photos (iCloud) auf selbst gehostete Immich-Instanz (datenbank.m-krueger.de). Priorität: EXIF-Metadatenintegrität (Datum, GPS), da Immich diese Felder direkt aus der Datei liest, nicht aus separaten Sidecars.

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

## Skalierung bei 16.000 Fotos
Bei dieser Größenordnung reicht "einmal exportieren und gut" nicht mehr. Relevante Punkte:

- **Laufzeit exiftool:** ~0,3–0,5s Subprocess-Overhead pro Datei → allein 1,5–2,5h nur für den exiftool-Durchlauf, ohne Kopiervorgang.
- **`--download-missing` ist der eigentliche Flaschenhals, nicht exiftool.** Fotos, die iCloud-Speicheroptimiert sind und nicht mehr lokal gecached liegen, werden erst von Apple nachgeladen — abhängig von Apples Rate-Limits, nicht von der eigenen Bandbreite. Kann von "läuft nebenbei durch" bis "dauert über Nacht oder länger" reichen. Vorher prüfen: `osxphotos query --missing` zeigt die Zahl der nicht-lokalen Originale.
- **Speicherplatz:** bei realistisch 3–8 MB/Foto (mehr bei RAW/Live Photos) sind 16.000 Fotos 50–130 GB. Vorher `df -h` auf dem Zielvolume checken.
- **Batching statt Gesamtexport in einem Rutsch.** Bei 16.000 Dateien lieber pro Jahr oder pro großem Album exportieren statt alles in einem Befehl. Bricht der Export z.B. bei Datei 12.000 wegen eines defekten/fehlenden Originals ab, will man nicht bei null anfangen. Bei Wiederanlauf `--only-new` nutzen, damit bereits exportierte Dateien übersprungen werden.

## Qualitätscheck vor Gesamt-Import
Vor dem vollen Batch immer erst an 2-3 Testalben validieren:
```bash
exiftool -DateTimeOriginal -GPSLatitude -GPSLongitude ~/immich-test/<Album>/<Datei>
```
Prüfen ob Datum und GPS korrekt gesetzt sind, bevor der Gesamtimport läuft.

## Nächster Schritt (Stand dieser Session)
Test-Export lief erfolgreich nach Fix von exiftool-Installation. Nächster Schritt: `osxphotos query --missing` laufen lassen um Cloud-only-Anteil zu kennen, Testalben validieren, dann Batching-Strategie für 16.000 Fotos festlegen, dann Immich CLI Upload mit kleinem Batch vor Gesamt-Import.

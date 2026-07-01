# osxphotos Export für Immich-Migration

## Kontext
Michael migriert ~16.000 Fotos von Apple Photos auf selbst gehostete Immich-Instanz (datenbank.m-krueger.de). Fotos-Bibliothek liegt lokal. In geteilten Alben fehlen **2.618 Originale** (Stand dieser Session, ermittelt via `osxphotos query --shared --missing`), verteilt über sehr viele Reisealben (u.a. USA Eastcoast, AIDA Spanien/Portugal, Südostasien, Norwegen mit Aida, Westcoast-Rundreise, Rovaniemi 2024, Mallorca 2018) und Zeiträume von 2008 bis heute. Das ist kein Nebenschritt, sondern ein eigener Migrationsblock mit relevantem Zeitbudget. Priorität: EXIF-Metadatenintegrität (Datum, GPS), da Immich diese Felder direkt aus der Datei liest, nicht aus separaten Sidecars.

## Setup
- osxphotos läuft in eigenem venv: `osxphotos-env`
- Version: 0.76.1 (Stand Juli 2026)
- exiftool wird zwingend benötigt: `brew install exiftool`, danach mit `which exiftool` prüfen
- Terminal: bracketed paste mode kann bei eingefügten Multi-Line-Befehlen `[200~ ... ~]`-Artefakte erzeugen → `unset zle_bracketed_paste` in `~/.zshrc`, falls Copy-Paste-Befehle mit "bad pattern"-Fehlern abbrechen

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
5. **Copy-Paste von Multi-Line-Befehlen ins Terminal** kann bracketed-paste-Escape-Codes (`[200~`, `~]`) mit einfügen → zsh-Fehler "bad pattern". Fix: `unset zle_bracketed_paste` in `~/.zshrc`, oder Befehl von Hand tippen.

## Feldauswahl bei --exiftool
`--exiftool` hat KEINEN granularen Schalter, um nur bestimmte Felder (z.B. nur Datum + GPS) zu schreiben. Es schreibt ein festes Set: Datum, GPS, Keywords, Description, Title, ggf. Personen als Keywords.

Wichtig für Performance-Einschätzung: Die Laufzeit von `--exiftool` hängt am Subprocess-Aufruf pro Datei, NICHT an der Zahl der geschriebenen Felder. Eine Feldeinschränkung würde also keine relevante Zeitersparnis bringen — dieser Optimierungsgedanke ist ein Holzweg.

Falls wirklich nur einzelne Felder gewünscht sind: `--sidecar XMP` exportieren und per eigenem exiftool-Aufruf (`-DateTimeOriginal -GPSLatitude -GPSLongitude`) gezielt aus dem Sidecar in die Zieldatei schreiben. Für den Normalfall (Immich-Import) unnötig — zusätzliche Felder wie Keywords/Description stören dort nicht.

## Skalierung bei 16.000 Fotos (lokale Bibliothek)
Da die Bibliothek lokal liegt, ist `--download-missing` für den Großteil der Fotos irrelevant. Der exiftool-Durchlauf ist hier der Flaschenhals:

- **Laufzeit exiftool:** ~0,3–0,5s Subprocess-Overhead pro Datei → 1,5–2,5h nur für den exiftool-Durchlauf bei 16.000 Dateien, ohne Kopiervorgang.
- **Speicherplatz:** bei realistisch 3–8 MB/Foto (mehr bei RAW/Live Photos) sind 16.000 Fotos 50–130 GB. Vorher `df -h` auf dem Zielvolume checken.
- **Batching statt Gesamtexport in einem Rutsch.** Lieber pro Jahr oder pro großem Album exportieren statt alles in einem Befehl. Bricht der Export ab, will man nicht bei null anfangen. Bei Wiederanlauf `--only-new` nutzen, damit bereits exportierte Dateien übersprungen werden.

## Geteilte Alben mit 2.618 fehlenden Bildern — eigener Migrationsblock

Ermittelte Zahl: `osxphotos query --shared --missing | wc -l` → 2619 (minus Kopfzeile = 2618 fehlende Originale). Das ist eine Größenordnung, bei der Apples Download-Rate-Limit realistisch mehrere Stunden bis über Nacht in Anspruch nimmt, verteilt über einen Nachladevorgang. Album-für-Album ist hier Pflicht, nicht nur "besser":

1. **Umfang je Album ermitteln, bevor der Download startet:**
   ```bash
   osxphotos query --shared --missing --json > missing_shared.json
   ```
   Aus dem `albums`-Feld pro Foto die betroffenen Albumnamen extrahieren und nach Anzahl fehlender Fotos sortieren (z.B. mit `jq`), um zu wissen wo die dicksten Brocken liegen.

2. **Erst die lokalen (nicht-geteilten) Alben exportieren** — läuft schnell, kein Cloud-Download nötig.

3. **Geteilte Alben mit Missing-Originalen einzeln, nicht alle in einem Lauf.** Bei 2.618 betroffenen Dateien über viele Alben verteilt ist die Wahrscheinlichkeit hoch, dass ein Gesamtlauf irgendwo abbricht (Netzwerk, Session-Timeout, defekte Assets) — dann ohne Album-Trennung von vorn anfangen zu müssen kostet mehr als die Vorab-Aufteilung.

4. **Reihenfolge:** Erst ein kleines Album als Test fahren, um die reale Download-Geschwindigkeit pro Foto einzuschätzen. Danach die großen Alben (laut Fotoliste u.a. USA Eastcoast, AIDA Spanien/Portugal, Südostasien als Kandidaten für viele Missing-Einträge) einzeln mit `--download-missing` laufen lassen.

5. **Bei jedem Wiederanlauf `--only-new`** verwenden, damit bereits heruntergeladene Dateien nicht erneut angefasst werden.

## Qualitätscheck vor Gesamt-Import
Vor dem vollen Batch immer erst an 2-3 Testalben validieren:
```bash
exiftool -DateTimeOriginal -GPSLatitude -GPSLongitude ~/immich-test/<Album>/<Datei>
```
Prüfen ob Datum und GPS korrekt gesetzt sind, bevor der Gesamtimport läuft.

## Nächster Schritt (Stand dieser Session)
Test-Export lief erfolgreich nach Fix von exiftool-Installation. Umfang der Missing-Shared-Fotos ist jetzt bekannt (2.618). Nächster Schritt: Album-Aufschlüsselung aus `missing_shared.json` erstellen, Prioritätenliste der Alben nach Anzahl Missing-Fotos, dann Download-Läufe album-weise starten (klein zuerst als Zeitschätzung), parallel lokale Alben exportieren, danach Immich CLI Upload mit kleinem Batch vor Gesamt-Import.

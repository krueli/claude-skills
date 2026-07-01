# osxphotos Export für Immich-Migration

## Kontext
Michael migriert ~16.000 Fotos von Apple Photos auf selbst gehostete Immich-Instanz (datenbank.m-krueger.de). Fotos-Bibliothek liegt lokal. In geteilten Alben fehlen **2.618 Originale** (Stand dieser Session, ermittelt via `osxphotos query --shared --missing`), verteilt über sehr viele Reisealben von 2008 bis heute. Das ist kein Nebenschritt, sondern ein eigener Migrationsblock mit relevantem Zeitbudget. Priorität: EXIF-Metadatenintegrität (Datum, GPS), da Immich diese Felder direkt aus der Datei liest, nicht aus separaten Sidecars.

## Setup
- osxphotos läuft in eigenem venv: `osxphotos-env`
- Version: 0.76.1 (Stand Juli 2026)
- exiftool wird zwingend benötigt: `brew install exiftool`, danach mit `which exiftool` prüfen
- `jq` für JSON-Auswertung: `brew install jq`
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

Ermittelte Zahl: `osxphotos query --shared --missing | wc -l` → 2619 (minus Kopfzeile = 2618 fehlende Originale).

Album-Aufschlüsselung (`jq -r '.[].albums[]' missing_shared.json | sort | uniq -c | sort -rn`), Top 20:

| Anzahl fehlend | Album |
|---|---|
| 326 | Westcoast-Rundreise |
| 251 | AIDA Spanien/Portugal |
| 205 | Südostasien |
| 204 | USA Eastcoast |
| 192 | Rom |
| 192 | Mallorca 2018 |
| 192 | Jugendweihe |
| 174 | AIDA Ostseetour |
| 128 | Prag 2014 |
| 109 | Norwegen mit Aida |
| 91 | Frankreich 2015 |
| 91 | Exeter |
| 80 | Weber Grillakademie 2016 |
| 65 | Rovaniemi 2024 |
| 54 | Heide Park |
| 52 | Klassenfahrt |
| 45 | F Scan |
| 32 | Ostsee-Warnemünde |
| 32 | AIDA Kanarische Inseln |
| 31 | Abschlussball_12.2015 |

Gleichmäßig verteilt, kein einzelner Ausreißer — bestätigt: album-weise Batch-Verarbeitung ist der richtige Ansatz, kein Sonderfall für ein Riesenalbum nötig.

### Vorgehen
1. **Erst lokale (nicht-geteilte) Alben exportieren** — läuft schnell, kein Cloud-Download nötig.
2. **Testlauf mit kleinstem Album zuerst** (Abschlussball_12.2015, 31 Fotos), um reale Download-Zeit pro Foto zu messen:
   ```bash
   time osxphotos export ~/immich-missing/Abschlussball_12.2015 \
     --album "Abschlussball_12.2015" \
     --filename "{original_name}" \
     --download-missing \
     --exiftool \
     --verbose
   ```
   Ergebnis hochrechnen auf die Gesamtmenge (2.618 Fotos), um realistisches Zeitbudget für den Rest zu bekommen.
3. **Danach absteigend nach Albumgröße abarbeiten** (Westcoast-Rundreise → AIDA Spanien/Portugal → Südostasien → ...), damit die großen Brocken nicht bis zuletzt liegen bleiben.
4. **Jedes Album einzeln exportieren, nicht in einem Gesamtlauf.** Bricht ein Album-Download ab, betrifft das nicht die bereits fertigen Alben.
5. **Bei jedem Wiederanlauf `--only-new`** verwenden, damit bereits heruntergeladene Dateien nicht erneut angefasst werden.

## Qualitätscheck vor Gesamt-Import
Vor dem vollen Batch immer erst an 2-3 Testalben validieren:
```bash
exiftool -DateTimeOriginal -GPSLatitude -GPSLongitude ~/immich-test/<Album>/<Datei>
```
Prüfen ob Datum und GPS korrekt gesetzt sind, bevor der Gesamtimport läuft.

## Nächster Schritt (Stand dieser Session)
Test-Export lief erfolgreich nach Fix von exiftool-Installation. Umfang und Album-Verteilung der Missing-Shared-Fotos sind bekannt. Nächster Schritt: Testlauf mit Abschlussball_12.2015 zur Zeitschätzung, dann Download-Läufe album-weise absteigend nach Größe starten, parallel lokale Alben exportieren, danach Immich CLI Upload mit kleinem Batch vor Gesamt-Import.

# osxphotos Export für Immich-Migration

## Kontext
Michael migriert ~16.000 Fotos von Apple Photos auf selbst gehostete Immich-Instanz (datenbank.m-krueger.de). Fotos-Bibliothek liegt lokal. In geteilten Alben fehlen **2.618 Originale** (Stand dieser Session, ermittelt via `osxphotos query --shared --missing`), verteilt über sehr viele Reisealben von 2008 bis heute. Priorität: EXIF-Metadatenintegrität (Datum, GPS), da Immich diese Felder direkt aus der Datei liest, nicht aus separaten Sidecars.

## Setup
- osxphotos läuft in eigenem venv: `osxphotos-env`
- Version: 0.76.1 (Stand Juli 2026)
- exiftool wird zwingend benötigt: `brew install exiftool`, danach mit `which exiftool` prüfen
- `jq` für JSON-Auswertung: `brew install jq`
- Terminal: bracketed paste mode kann bei eingefügten Multi-Line-Befehlen `[200~ ... ~]`-Artefakte erzeugen → `unset zle_bracketed_paste` in `~/.zshrc`, falls Copy-Paste-Befehle mit "bad pattern"-Fehlern abbrechen

## Funktionierender Export-Befehl
```bash
mkdir -p ~/immich-missing/<Albumname>
time osxphotos export ~/immich-missing/<Albumname> \
  --album "<Albumname>" \
  --filename "{original_name}" \
  --download-missing \
  --exiftool \
  --verbose
```
`time` davor liefert die reale Laufzeit zum Abgleich mit der erwarteten Rate.

## Bekannte Stolperfallen
1. **`--original-name` existiert nicht.** Richtige Option ist `--filename "{original_name}"`. osxphotos schlägt bei falschem Flag Alternativen vor (`--filename`, `--only-new`, `--original-suffix`).
2. **Zielverzeichnis muss vorher existieren** (`mkdir -p` davor), sonst bricht der Export mit "Invalid value for 'DEST'" ab — und die Folgezeilen des Multi-Line-Befehls werden von zsh dann fälschlich als eigene Kommandos interpretiert ("command not found: --directory").
3. **exiftool separat installieren.** osxphotos bringt es nicht mit. Ohne exiftool bricht `--exiftool` mit "Could not find exiftool" ab, egal ob der Rest der Befehlszeile korrekt ist.
4. **`{original_name}` ist nicht immer zuverlässig.** Bei Fotos, die reimportiert oder bearbeitet wurden, kann das Feld leer sein — osxphotos fällt dann auf UUID-Namen zurück. Nach jedem Export-Batch stichprobenartig prüfen (`ls` im Zielordner), ob lesbare Dateinamen rauskommen.
5. **Copy-Paste von Multi-Line-Befehlen ins Terminal** kann bracketed-paste-Escape-Codes (`[200~`, `~]`) mit einfügen → zsh-Fehler "bad pattern". Fix: `unset zle_bracketed_paste` in `~/.zshrc`, oder Befehl von Hand tippen.

## Feldauswahl bei --exiftool
`--exiftool` hat KEINEN granularen Schalter, um nur bestimmte Felder (z.B. nur Datum + GPS) zu schreiben. Es schreibt ein festes Set: Datum, GPS, Keywords, Description, Title, ggf. Personen als Keywords.

Die Laufzeit von `--exiftool` hängt am Subprocess-Aufruf pro Datei, NICHT an der Zahl der geschriebenen Felder. Eine Feldeinschränkung würde also keine relevante Zeitersparnis bringen.

Falls wirklich nur einzelne Felder gewünscht sind: `--sidecar XMP` exportieren und per eigenem exiftool-Aufruf (`-DateTimeOriginal -GPSLatitude -GPSLongitude`) gezielt aus dem Sidecar in die Zieldatei schreiben. Für den Normalfall (Immich-Import) unnötig.

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
| 326 | Westcoast-Rundreise (✅ erledigt) |
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
| 31 | Abschlussball_12.2015 (✅ erledigt) |

Gleichmäßig verteilt, kein einzelner Ausreißer — album-weise Batch-Verarbeitung ist der richtige Ansatz.

### Gemessene Raten (2 Testläufe)
| Testlauf | Fotos | Zeit | Rate |
|---|---|---|---|
| Abschlussball_12.2015 | 31 | 3 Min | 5,8s/Foto |
| Westcoast-Rundreise | 326 | 12:10 Min | 2,24s/Foto |

**Erkenntnis:** Die Rate wird bei größeren Batches schneller, nicht langsamer — der Fixkosten-Overhead (osxphotos-Start, Photos-DB laden) verteilt sich auf mehr Dateien und macht bei kleinen Batches einen größeren Anteil aus. Apples Download-Rate-Limit greift bei diesen Batchgrößen (bis 326) nicht spürbar. Für die Hochrechnung der restlichen Alben die konservativere große Rate (2,24s/Foto) verwenden.

**Restliche 2.261 Fotos (2.618 − 357 bereits erledigt) bei 2,24s/Foto: ~84 Minuten (~1,5h) Gesamtzeit.** Deutlich günstiger als ursprünglich befürchtet — kein Grund mehr, sich vor dem Rate-Limit zu fürchten, lässt sich an einem Nachmittag durchziehen.

### Vorgehen
1. **Erst lokale (nicht-geteilte) Alben exportieren** — läuft schnell, kein Cloud-Download nötig.
2. **Restliche Alben absteigend nach Größe abarbeiten** (AIDA Spanien/Portugal 251 → Südostasien 205 → USA Eastcoast 204 → Rom/Mallorca 2018/Jugendweihe je 192 → ...), jedes einzeln mit dem Export-Befehl oben.
3. **Jedes Album einzeln exportieren, nicht in einem Gesamtlauf.** Bricht ein Album-Download ab, betrifft das nicht die bereits fertigen Alben.
4. **Bei jedem Wiederanlauf `--only-new`** verwenden, damit bereits heruntergeladene Dateien nicht erneut angefasst werden.

## Qualitätscheck vor Gesamt-Import
Vor dem vollen Batch immer erst an 2-3 Testalben validieren:
```bash
exiftool -DateTimeOriginal -GPSLatitude -GPSLongitude ~/immich-test/<Album>/<Datei>
```
Prüfen ob Datum und GPS korrekt gesetzt sind, bevor der Gesamtimport läuft.

## Nächster Schritt (Stand dieser Session)
2 von ~20 Alben mit fehlenden Originalen erledigt (Abschlussball_12.2015, Westcoast-Rundreise). Rate bestätigt sich als schnell und stabil (~2,24s/Foto bei größeren Batches). Nächster Schritt: restliche Alben absteigend nach Größe abarbeiten (nächstes: AIDA Spanien/Portugal, 251 Fotos), dann lokale Alben exportieren, danach Immich CLI Upload mit kleinem Batch vor Gesamt-Import.

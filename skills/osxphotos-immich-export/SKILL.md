# osxphotos Export für Immich-Migration

## Kontext
Michael migriert ~16.000 Fotos von Apple Photos auf selbst gehostete Immich-Instanz (datenbank.m-krueger.de). Fotos-Bibliothek liegt lokal unter `/Users/michaelkruger/Pictures/Photos Library.photoslibrary`. In geteilten Alben fehlten (Stand Session 1) **2.618 Originale** (ermittelt via `osxphotos query --shared --missing`), verteilt über sehr viele Reisealben von 2008 bis heute. Priorität: EXIF-Metadatenintegrität (Datum, GPS), da Immich diese Felder direkt aus der Datei liest, nicht aus separaten Sidecars. Export-Zielordner: `~/immich-missing/<Albumname>`.

## ⚠️⚠️ WICHTIGSTER FUND (Session 2): `--use-photokit` crasht hart, plain `--download-missing`/`--exiftool` verschiebt Datum um Zeitzonen-Offset bei geteilten Alben

### 1. `--use-photokit` / `use_photokit=True` funktioniert grundsätzlich NICHT für CLI-Skripte
Führt zu `SIGABRT`, macOS killt den Prozess sofort (TCC-Crash-Report: `namespace: TCC`, "app crashed because it attempted to access privacy-sensitive data without a usage description... com.apple.security.personal-information.photos-library"). Das ist ein **App-Sandbox-Entitlement**, das ein normaler `python3`-Prozess (auch signiert, auch mit korrektem `NSPhotoLibraryUsageDescription` im Info.plist) praktisch nicht bekommen kann. Kein Bug im eigenen Code — einfach nicht nutzen. Führt zu den ursprünglich beobachteten `RetryError`/`ValueError`-Symptomen (verschiedene Wrapper um denselben Crash).

**Funktionierender Ersatz:** AppleScript-Automation der bereits autorisierten Photos-App über die osxphotos-Python-API direkt (nicht die einfache `PhotoInfo.export()`, die kein `use_photokit`/`download_missing` kennt):
```python
from osxphotos.exportoptions import ExportOptions
from osxphotos.photoexporter import PhotoExporter
options = ExportOptions(download_missing=True, use_photos_export=True, exiftool=True, timeout=180)
PhotoExporter(photo).export(dest=str(dest_dir), options=options)
```
Fertiges Skript: `~/immich-missing/export_shared_album.py "<Albumname>"` — läuft album-weise, mit Retry-Logik, Zusammenfassung am Ende. Nutzt intern exakt obigen Mechanismus. Die normale CLI (`osxphotos export --download-missing --exiftool`, ohne `--use-photokit`) nutzt ebenfalls AppleScript im Hintergrund und funktioniert grundsätzlich genauso — das eigene Skript ist nur robuster bei Album-für-Album-Verarbeitung mit Fehlerprotokoll.

### 2. Geteilte/Cloud-Only-Fotos haben `tzoffset=None` → `--exiftool` schreibt Datum in falscher "Zeitzone" (nicht falsche Absolutzeit, aber falsche Lokalzeit-Anzeige, potenziell falsches Kalenderdatum)
Bestätigt an mehreren Alben: **alle** Fotos mit `path=None`/`shared=True` haben `photo.tzoffset == None` (im Gegensatz zu eigenen/lokalen Fotos, die immer eine korrekte Zeitzone haben). `photo.date` wird dann als UTC zurückgegeben (technisch korrekter Absolutzeitpunkt), aber `--exiftool` schreibt diesen UTC-Wert als `DateTimeOriginal` mit `OffsetTimeOriginal +00:00` — das ist eine **ehrliche, aber falsche Lokalzeit-Angabe** (die Fotos wurden nicht in UTC+0 aufgenommen). Beispiel Südostasien-Trip: Foto in Singapur zeigte nach `--exiftool` 10:22:58 UTC statt echter 18:22:58 Ortszeit (Singapur = UTC+8). Bei großen Zeitverschiebungen kann sich sogar das **Kalenderdatum** verschieben (Aufnahmen nahe Mitternacht Ortszeit).

**Betroffen:** vermutlich alle ~2.618 fehlenden Fotos in geteilten Alben — auch die bereits früher exportierten **Westcoast-Rundreise (326) und Abschlussball_12.2015 (31) sind betroffen und wurden NOCH NICHT korrigiert** (Stand Ende Session 2). Diese beiden müssen noch mit dem unten beschriebenen timewarp-Workflow nachbearbeitet werden, bevor sie final nach Immich importiert werden.

**Fix-Workflow (pro Album, VOR dem finalen Export):**
1. GPS-Koordinaten pro Datum im Album auswerten, um Reiseabschnitte/Länder zu identifizieren:
   ```python
   # p.location gibt (lat, lon) oder (None, None); Gruppierung nach p.date.date()
   ```
2. Route mit dem Nutzer verifizieren (Länder/Zeitzonen aus Koordinaten sind nicht immer eindeutig, z.B. Grenzregionen) — NICHT blind raten.
3. Pro Zeitzonen-Segment `osxphotos timewarp` auf die LIVE Photos-Bibliothek anwenden:
   ```bash
   osxphotos timewarp --album "<Album>" --from-date YYYY-MM-DD --to-date YYYY-MM-DD --timezone +HH:MM --force --verbose
   ```
   **Ohne** `--match-time` verwenden — damit rechnet osxphotos den bereits korrekten UTC-Absolutzeitpunkt in die neue Zeitzone um (ergibt die echte Ortszeit). Mit `--match-time` würde nur das Label geändert, die Uhrzeit aber unverändert bleiben (falsch für diesen Fall).
   `--from-date`/`--to-date` sind exklusiv am Ende (`--to-date 2026-03-06` schließt den 5.3. noch ein, aber nicht den 6.3.) — vor dem scharfen Lauf immer mit `--inspect` (kein `--timezone`, rein lesend) die Foto-Anzahl pro Segment gegenprüfen, Summe muss der Albumgröße entsprechen.
   **`timewarp` warnt selbst vor möglicher Bibliotheks-Beschädigung und empfiehlt ein Backup** — das ernst nehmen, vor dem ersten Lauf beim Nutzer nachfragen. `--force` ist nötig für nicht-interaktive Ausführung (sonst bricht `click.confirm` ohne TTY mit Crash-Log ab statt sauberem Abbruch).
4. Verifizieren: `p.tzoffset` sollte danach für alle Fotos im Album gesetzt sein (nicht mehr `None`).
5. Erst DANACH (neu) exportieren — bereits vorhandene Exporte vorher löschen und neu erzeugen, sonst bleibt der falsche Zeitstempel in der Datei.
6. Stichprobe: `exiftool -DateTimeOriginal -OffsetTimeOriginal <Datei>` — Lokalzeit sollte jetzt plausibel sein (z.B. Frühstücksfoto nicht um 3 Uhr nachts).

Praktischer Vorteil: Länder mit gleicher Zeitzone (z.B. Thailand+Vietnam beide UTC+7) lassen sich in einem `timewarp`-Aufruf über mehrere Tage zusammenfassen, das reduziert die Anzahl nötiger Segmente deutlich.

## Diagnose-Falle (weiterhin gültig): RetryError/ValueError bei 100% eines Albums NICHT als Python/Pfad-Bug fehldeuten
Wenn `osxphotos export` bei EINEM kompletten Album (alle Fotos, alle Dateitypen) mit `RetryError[<Future ... raised ValueError>]` durchfällt, andere Alben aber sauber laufen: **Erste Vermutung ist "fehlendes Original" (path=None, iCloud-Freigabe), NICHT Python-Version oder Pfad-Encoding.** Schnelltest: `osxphotos query --album "<Albumname>" --missing | wc -l`. Mit `-VVV` wird aus dem ValueError meist ein klares `Skipping missing original photo ...`.
Falls das Album zur untenstehenden 100%-shared-Kategorie gehört UND `--use-photokit` verwendet wurde: siehe Abschnitt 1 oben (SIGABRT), das ist dann die eigentliche Ursache, nicht die Diagnose-Falle selbst.

## Setup
- osxphotos läuft in eigenem venv: `osxphotos-env` (Python 3.14.6, Homebrew python@3.14 Framework-Build)
- Version: 0.76.1 (Stand Juli 2026)
- exiftool zwingend nötig: `brew install exiftool` (liegt unter `/usr/local/bin/exiftool`), mit `which exiftool` prüfen
- `jq` für JSON-Auswertung: `brew install jq`
- Volle Festplattenzugriff (Full Disk Access) muss für die ausführende App/Terminal aktiviert sein (Systemeinstellungen > Datenschutz & Sicherheit), sonst scheitert schon das Kopieren der `Photos.sqlite` mit `PermissionError`.
- Terminal: bracketed paste mode kann bei eingefügten Multi-Line-Befehlen `[200~ ... ~]`-Artefakte erzeugen → `unset zle_bracketed_paste` in `~/.zshrc`, falls Copy-Paste-Befehle mit "bad pattern"-Fehlern abbrechen

## Funktionierender Export-Befehl (CLI, für Alben OHNE fehlende Originale bzw. wenn Zeitzonen-Fix nicht nötig ist)
```bash
mkdir -p ~/immich-missing/<Albumname>
time osxphotos export ~/immich-missing/<Albumname> \
  --album "<Albumname>" \
  --filename "{original_name}" \
  --download-missing \
  --exiftool \
  --verbose
```
`time` davor liefert die reale Laufzeit zum Abgleich mit der erwarteten Rate. `DEST` muss vor dem Aufruf existieren (`mkdir -p`).

**Für geteilte Alben mit `tzoffset=None` (praktisch alle Reisealben mit fehlenden Originalen) erst den timewarp-Fix (siehe oben) durchführen, dann erst exportieren — sonst landet der falsche UTC-Zeitstempel in der Datei.**

## Bekannte Stolperfallen
1. **`--original-name` existiert nicht.** Richtige Option ist `--filename "{original_name}"`.
2. **Zielverzeichnis muss vorher existieren** (`mkdir -p` davor), sonst "Invalid value for 'DEST'".
3. **exiftool separat installieren**, sonst "Could not find exiftool" bei `--exiftool`.
4. **`{original_name}` nicht immer zuverlässig** bei reimportierten/bearbeiteten Fotos (UUID-Fallback) — stichprobenartig `ls` im Zielordner prüfen.
5. **Bracketed-paste-Artefakte** bei Multi-Line-Copy-Paste (`[200~`) → `unset zle_bracketed_paste`.
6. **RetryError/ValueError auf 100% eines Albums = fehlendes Original**, siehe Diagnose-Falle oben.
7. **`--use-photokit` = harter SIGABRT-Crash**, siehe Abschnitt 1. Nie verwenden.
8. **`tzoffset=None` bei geteilten Alben → `--exiftool` verschiebt Datum**, siehe Abschnitt 2. Vor Export prüfen und ggf. timewarp-Fix anwenden.
9. **`osxphotos timewarp` ohne `--force` bricht in nicht-interaktiven Kontexten mit Crash-Log ab** (click.confirm ohne TTY). `--force` bewusst einsetzen, vorher Backup-Frage an Nutzer stellen (timewarp warnt selbst vor Bibliotheks-Beschädigung).
10. **`update=True` in eigenen Export-Skripten ohne persistente `export_db`** kann bei filename-Kollisionen (zwei UUIDs, gleicher Kamerraname, identischer Inhalt) Fotos fälschlich als "kein Update nötig" überspringen, wenn schon eine gleichnamige Datei im Zielordner existiert. Bei frischem Zielordner und `exiftool=True` trat das nicht auf (vermutlich weil exiftool-Schritt den Kurzschluss verhindert) — im Zweifel Zielordner vor jedem vollständigen Neu-Export löschen statt auf Update-Erkennung zu vertrauen.

## Feldauswahl bei --exiftool
`--exiftool` hat KEINEN granularen Schalter für einzelne Felder. Es schreibt ein festes Set: Datum, GPS, Keywords, Description, Title, ggf. Personen als Keywords. Laufzeit hängt am Subprocess-Aufruf pro Datei, nicht an der Zahl der Felder.

## Skalierung bei 16.000 Fotos (lokale Bibliothek)
- **Laufzeit exiftool:** ~0,3–0,5s Subprocess-Overhead pro Datei → 1,5–2,5h nur für den exiftool-Durchlauf bei 16.000 Dateien.
- **Speicherplatz:** realistisch 3–8 MB/Foto → 50–130 GB für 16.000 Fotos. Vorher `df -h` prüfen.
- **Batching pro Album statt Gesamtexport.** Bei Wiederanlauf `--only-new`.

## Geteilte Alben — Migrationsblock (Stand Session 2)

Album-Aufschlüsselung, Top 20 nach Anzahl fehlender Originale:

| Anzahl fehlend | Album | Status |
|---|---|---|
| 326 | Westcoast-Rundreise | ✅ exportiert, ⚠️ Zeitzonen-Fix noch ausstehend |
| 251 | AIDA Spanien/Portugal | offen — nächstes Album |
| 205 | Südostasien | ✅ komplett (Export + Zeitzonen-Fix, Mehrländer-Route: Finnland/Singapur/Malaysia/Thailand+Vietnam/Hongkong) |
| 204 | USA Eastcoast | offen |
| 192 | Rom | offen |
| 192 | Mallorca 2018 | offen |
| 192 | Jugendweihe | offen |
| 174 | AIDA Ostseetour | offen |
| 128 | Prag 2014 | offen |
| 109 | Norwegen mit Aida | offen |
| 91 | Frankreich 2015 | offen |
| 91 | Exeter | offen |
| 80 | Weber Grillakademie 2016 | offen |
| 65 | Rovaniemi 2024 | offen |
| 54 | Heide Park | offen |
| 52 | Klassenfahrt | offen |
| 45 | F Scan | offen |
| 32 | Ostsee-Warnemünde | ✅ komplett (Export + Zeitzonen-Fix, Deutschland, einheitlich CET +01:00) |
| 32 | AIDA Kanarische Inseln | offen |
| 31 | Abschlussball_12.2015 | ✅ exportiert, ⚠️ Zeitzonen-Fix noch ausstehend |

Gleichmäßig verteilt, kein einzelner Ausreißer — album-weise Batch-Verarbeitung ist der richtige Ansatz.

### Gemessene Raten
| Testlauf | Fotos | Zeit | Rate |
|---|---|---|---|
| Abschlussball_12.2015 | 31 | 3 Min | 5,8s/Foto |
| Westcoast-Rundreise | 326 | 12:10 Min | 2,24s/Foto |
| Südostasien (use_photos_export API, mit exiftool) | 205 | ~8-10 Min geschätzt | ~2,5-3s/Foto |

Rate wird bei größeren Batches schneller (Fixkosten-Overhead verteilt sich). Für Hochrechnung konservative ~2,5s/Foto verwenden.

### Vorgehen
1. Erst lokale (nicht-geteilte) Alben exportieren — kein Cloud-Download nötig.
2. Für jedes geteilte Album MIT fehlenden Originalen: GPS/Datum-Analyse für Zeitzonen-Segmente → Route mit Nutzer verifizieren → timewarp anwenden → dann erst exportieren (mit `export_shared_album.py` oder CLI, `--exiftool`).
3. Jedes Album einzeln, nicht in einem Gesamtlauf.
4. Bei Wiederanlauf `--only-new` (CLI) bzw. Zielordner vorher löschen (eigenes Skript, siehe Stolperfalle 10).

## Qualitätscheck vor Gesamt-Import
```bash
exiftool -DateTimeOriginal -OffsetTimeOriginal -GPSLatitude -GPSLongitude ~/immich-missing/<Album>/<Datei>
```
Datum, Zeitzone und GPS auf Plausibilität prüfen (Uhrzeit passt zur Tageszeit der Aktivität auf dem Foto).

## Nächster Schritt (Stand Ende Session 2)
Erledigt: Südostasien (205) und Ostsee-Warnemünde (32) — beide vollständig inkl. korrekter Zeitzone.
**Offen/wichtig:** Westcoast-Rundreise (326) und Abschlussball_12.2015 (31) haben denselben `tzoffset=None`-Bug wie Südostasien, wurden aber noch NICHT mit timewarp korrigiert — sollten vor dem finalen Immich-Import nachgezogen werden (gleicher Workflow: GPS/Datum-Analyse, Route verifizieren, timewarp, Neu-Export).
Nächstes Album nach Größe: AIDA Spanien/Portugal (251 Fotos) — vermutlich auch Mehrländer-Route (Kreuzfahrt), also GPS-Analyse vor Export nicht vergessen.
Skript für Neuexporte: `~/immich-missing/export_shared_album.py "<Albumname>"` (nutzt `use_photos_export=True`, `download_missing=True`, `exiftool=True` — kein `--use-photokit`).

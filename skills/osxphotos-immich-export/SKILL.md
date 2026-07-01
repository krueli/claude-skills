# osxphotos Export für Immich-Migration

## Kontext
Michael migriert ~16.000 Fotos von Apple Photos auf selbst gehostete Immich-Instanz (datenbank.m-krueger.de). Fotos-Bibliothek liegt lokal unter `/Users/michaelkruger/Pictures/Photos Library.photoslibrary`. In geteilten Alben fehlten (Stand Session 1) **2.618 Originale**, verteilt über sehr viele Reisealben von 2008 bis heute. Priorität: EXIF-Metadatenintegrität (Datum, GPS), da Immich diese Felder direkt aus der Datei liest. Export-Zielordner: `~/immich-missing/<Albumname>` (Slashes im Albumnamen werden zu `_`).

**Stand Ende Session 3: ALLE geteilten Alben mit fehlenden Originalen sind exportiert und zeitzonenkorrigiert (siehe Tabelle unten). Migrationsblock ist fertig — nächster Schritt ist der eigentliche Immich-Import, nicht mehr osxphotos-Export.**

## ⚠️⚠️ Die drei wichtigsten Fallstricke (Session 2+3)

### 1. `--use-photokit` / `use_photokit=True` funktioniert grundsätzlich NICHT für CLI-Skripte
Führt zu `SIGABRT` (TCC-Crash: fehlendes App-Sandbox-Entitlement `com.apple.security.personal-information.photos-library`, das ein normaler `python3`-Prozess praktisch nie bekommt). Nie verwenden. **Funktionierender Ersatz:** AppleScript-Automation der bereits autorisierten Photos-App:
```python
from osxphotos.exportoptions import ExportOptions
from osxphotos.photoexporter import PhotoExporter
options = ExportOptions(download_missing=True, use_photos_export=True, exiftool=True, timeout=180)
PhotoExporter(photo).export(dest=str(dest_dir), options=options)
```
Fertiges Skript: `~/immich-missing/export_shared_album.py "<Albumname>"`.

### 2. Geteilte/Cloud-Only-Fotos haben `tzoffset=None` → `--exiftool` schreibt Datum in falscher Zeitzone
Alle Fotos mit `path=None`/`shared=True` haben `photo.tzoffset == None`. `--exiftool` schreibt dann den UTC-Wert mit `OffsetTimeOriginal +00:00` — ehrlich, aber falsche Lokalzeit (bei großen Zeitverschiebungen sogar falsches Kalenderdatum).

**Fix-Workflow (VOR dem finalen Export):**
1. GPS-Koordinaten pro Datum auswerten (`p.location`, gruppiert nach `p.date.date()`), Reiseabschnitte/Länder identifizieren.
2. **Nachbarländer mit unterschiedlicher Zeitzone trotz gemeinsamer Grenze beachten:** Spanien (MEZ/MESZ) vs. Portugal (WET/WEST) — 1h Unterschied. Kanaren+Madeira nutzen WET (UTC+0 im Winter), NICHT das spanische Festland-MEZ.
3. **US-Bundesstaaten:** Eastern/Central-Grenze verläuft durch Tennessee/Alabama/Florida-Panhandle (Nashville, Memphis = Central; Atlanta, Miami, Orlando = Eastern). Arizona hat ganzjährig KEINE DST (Mountain Standard = numerisch identisch mit Pacific Daylight im Sommer). Nevada nutzt Pacific, nicht Mountain, trotz Nachbarschaft zu Utah/Arizona.
4. Route/Segmentierung bei Unsicherheit mit Nutzer verifizieren — bei "weiß nicht mehr genau" plausibelste Annahme nehmen (z.B. Zielort-Heuristik für Reise-/Seetage) und im Skill dokumentieren.
5. Pro Segment:
   ```bash
   osxphotos timewarp --album "<Album>" --from-date YYYY-MM-DD --to-date YYYY-MM-DD --timezone +HH:MM --force
   ```
   Albumnamen mit `/` brauchen `//` als Escape (`--album "AIDA Spanien//Portugal"`), sonst 0 stille Treffer. `--to-date` ist exklusiv. **Ohne** `--match-time` (rechnet UTC-Absolutzeit korrekt in neue Zeitzone um). Vor dem scharfen Lauf mit `--inspect` (ohne `--timezone`) die Anzahl pro Segment gegen die Albumgröße prüfen. `--force` nötig für nicht-interaktive Ausführung (ohne TTY bricht `click.confirm` sonst mit Crash-Log ab). `timewarp` warnt selbst vor Bibliotheks-Beschädigung — Nutzer hat sich bewusst gegen Backup entschieden (Risiko akzeptiert für dieses Projekt).
6. Bereits vorhandene Exporte für das Album **komplett löschen** (`rm -rf`) bevor neu exportiert wird — nicht einfach in den bestehenden Ordner schreiben (siehe Fallstrick 3).
7. Stichprobe je Segment: `exiftool -DateTimeOriginal -OffsetTimeOriginal -GPSLatitude -GPSLongitude <Datei>`.

### 3. ⚠️ KRITISCH: Alte, VOR dem Zeitzonen-Fix exportierte Dateien im Zielordner erzeugen stille Duplikate mit falschem Datum
Mehrere Alben (Rom, Prag 2014, Jugendweihe, USA Eastcoast, AIDA Kanarische Inseln, AIDA Ostseetour, u.a.) hatten bereits **vor** Session 3 vollständige Exporte im Zielordner — vermutlich aus einer früheren Session per CLI `osxphotos export --download-missing --exiftool` (erkennbar an vorhandener `.osxphotos_export.db`-Datei im Ordner). Diese alten Exporte haben den tzoffset=None-Bug NICHT korrigiert bekommen (falsches Datum/Offset +00:00 in der Datei).

**Wenn `export_shared_album.py` (oder die CLI) in einen Ordner exportiert, der bereits gleichnamige Dateien enthält, überschreibt es diese NICHT, sondern hängt ` (1)`, ` (2)`, etc. an den neuen Dateinamen an** (osxphotos increment-Verhalten, `overwrite=False`). Ergebnis: alte (falsche Zeitzone) und neue (korrekte Zeitzone) Version derselben Aufnahme liegen doppelt im Ordner — stille Datenkontamination, keine Fehlermeldung.

**Erkennung:** Dateianzahl im Zielordner nach Export mit der erwarteten Albumgröße vergleichen. Deutliche Abweichung (oft fast exakt verdoppelt) + Vorhandensein von `.osxphotos_export.db` = Verdacht auf Altbestand.

**Sicherer Fix (NICHT blind alte Dateien löschen — manche Namensgleichheit ist echt zufällig, z.B. gleicher Kamera-Dateiname für zwei unterschiedliche Fotos):**
1. Aus dem Export-Log alle Zeilen mit `(N)`-Suffix im Zieldateinamen extrahieren → das ist der "neue" Pfad, `basename+ext` (ohne Suffix) ist der Kandidat für die "alte" Datei.
2. Für jedes Paar (alt, neu) per `exiftool -j -GPSLatitude# -GPSLongitude# -DateTimeOriginal <alt> <neu>` (GEBÜNDELT, nicht einzeln — viel schneller) vergleichen: GPS nahezu identisch (Toleranz ~0.01°) ODER Datumsdifferenz < ~15h (deutet auf denselben Zeitpunkt, nur andere Zeitzonen-Label) → **wahrscheinlich dasselbe Foto, alte Datei löschen**. GPS/Datum weichen deutlich ab → **unterschiedliche Fotos, beide behalten**.
3. Bei Videos (`.mp4`) zusätzlich `Duration`/`FileSize` vergleichen, da `DateTimeOriginal`/GPS bei QuickTime-Dateien oft fehlen oder anders liegen.
4. Nach Löschung: finale Dateianzahl je Album gegen erwartete Zahl (Albumgröße + evtl. legitime separate lokale Album-Fotos + evtl. echte Namenskollisionen) plausibilisieren.

**Am saubersten für zukünftige Alben: IMMER erst `rm -rf` auf den Zielordner, dann frisch exportieren — vermeidet das ganze Kollisions-/Bereinigungsproblem von vornherein.** Nur bei bereits vor der Session existierenden, potenziell gemischten Ordnern (lokales Album + geteiltes Album gleichen Namens) ist die aufwändigere Verifikations-Variante nötig.

**Manche Alben haben zusätzlich ein separates REGULÄRES (nicht-geteiltes) Album mit demselben Namen** (bestätigt für Prag 2014: 126 lokale Fotos, Jugendweihe: 39 lokale Fotos) — diese lokalen Fotos haben bereits korrekten tzoffset und sind NICHT Teil des Zeitzonen-Bugs. Vor einem `rm -rf` auf einen bereits gefüllten Zielordner IMMER prüfen: `photosdb.album_info` (regulär) vs. `photosdb.album_info_shared` (geteilt) für den Albumnamen — wenn ein reguläres Album mit demselben Namen existiert, NICHT blind löschen, sondern die aufwändigere Verifikations-Methode (Punkt 3) nutzen, um die legitimen lokalen Fotos nicht zu verlieren.

## Diagnose-Falle: RetryError/ValueError bei 100% eines Albums NICHT als Python/Pfad-Bug fehldeuten
Wenn `osxphotos export` bei EINEM kompletten Album mit `RetryError[<Future ... raised ValueError>]` durchfällt: **Erste Vermutung ist "fehlendes Original" (path=None, iCloud-Freigabe), NICHT Python-Version oder Pfad-Encoding.** Falls zusätzlich `--use-photokit` verwendet wurde: siehe Fallstrick 1 (SIGABRT ist dann die eigentliche Ursache).

## Setup
- osxphotos venv: `osxphotos-env` (Python 3.14.6, Homebrew python@3.14 Framework-Build), Version 0.76.1
- exiftool: `/usr/local/bin/exiftool` (`brew install exiftool`)
- Volle Festplattenzugriff für die ausführende App/Terminal aktiviert (Systemeinstellungen > Datenschutz & Sicherheit), sonst `PermissionError` beim Kopieren der `Photos.sqlite`
- Terminal bracketed-paste-Artefakte: `unset zle_bracketed_paste` in `~/.zshrc` falls nötig

## Bekannte Stolperfallen (kompakt)
1. `--original-name` existiert nicht → `--filename "{original_name}"`.
2. Zielverzeichnis muss vorher existieren (`mkdir -p`).
3. exiftool separat installieren.
4. `{original_name}` bei reimportierten Fotos evtl. UUID statt Klarname.
5. Bracketed-paste-Artefakte bei Copy-Paste.
6. RetryError/ValueError auf 100% eines Albums = fehlendes Original (siehe Diagnose-Falle).
7. `--use-photokit` = SIGABRT-Crash, nie verwenden.
8. `tzoffset=None` bei geteilten Alben → Datum falsch, siehe Fallstrick 2.
9. `osxphotos timewarp` ohne `--force` crasht ohne TTY.
10. `update=True` ohne persistente `export_db` kann bei Namenskollision fälschlich überspringen.
11. Albumnamen mit `/` → im Zielordnernamen zu `_` sanitizen (macht `export_shared_album.py` automatisch), bei `timewarp --album` als `//` escapen.
12. **Alte Exporte im Zielordner vor Zeitzonen-Fix → stille Duplikate mit falschem Datum, siehe Fallstrick 3. IMMER Zielordner-Inhalt vor einem Export prüfen, ggf. `rm -rf` (nach Check auf separates reguläres Album!).**

## Funktionierender Export-Befehl (CLI, alternative zu export_shared_album.py)
```bash
mkdir -p ~/immich-missing/<Albumname>
osxphotos export ~/immich-missing/<Albumname> --album "<Albumname>" --filename "{original_name}" --download-missing --exiftool --verbose
```
Wichtig: erst timewarp-Fix (Fallstrick 2) durchführen, dann erst exportieren.

## Skalierung
- exiftool: ~0,3–0,5s Subprocess-Overhead/Datei. AppleScript-Export (use_photos_export): ~2,5-3s/Foto im Schnitt, größere Alben tendenziell schneller (Fixkosten-Overhead verteilt sich), manche Alben (z.B. Exeter) deutlich langsamer (~9s/Foto) ohne erkennbaren Grund — kein Fehler, nur Geduld.
- 16.000 Fotos gesamt: 50-130 GB, vorher `df -h` prüfen.
- Größere Exports (>100 Fotos) im Hintergrund laufen lassen (`run_in_background`), mehrere Alben lassen sich in einem sequenziellen Batch-Skript hintereinander abarbeiten (NICHT parallel — Photos.app kann nur eine AppleScript-Automation gleichzeitig sauber verarbeiten).

## Geteilte Alben — Migrationsblock: ABGESCHLOSSEN (Stand Ende Session 3)

Alle Alben mit fehlenden Originalen sind exportiert UND zeitzonenkorrigiert:

| Album | Fotos | Route/Zeitzone(n) |
|---|---|---|
| Westcoast-Rundreise | 326 | Kalifornien/Arizona -07:00 (14.-24.8.), Utah -06:00 (25.8., unsicher/Best-Guess), Nevada -07:00 (26.-29.8.) |
| AIDA Spanien/Portugal | 251 (250 exportiert, 1 Dublette) | Spanien MESZ+02:00, Lissabon WEST+01:00 |
| Südostasien | 205 | Finnland+02:00, Singapur/Malaysia+08:00, Thailand/Vietnam+07:00, Hongkong+08:00 |
| USA Eastcoast | 204 | Eastern -04:00, Central -05:00 (Nashville/Memphis/New Orleans/Panama City), zurück Eastern -04:00 |
| AIDA Kanarische Inseln | 296 | einheitlich WET +00:00 (Kanaren+Madeira, Winter) |
| Rom | 192 | einheitlich MESZ +02:00 |
| Mallorca 2018 | 192 | einheitlich MESZ +02:00 (Hin-/Rückflug Deutschland auch +02:00) |
| Jugendweihe | 193 | einheitlich MESZ +02:00 (Deutschland, keine GPS-Daten nötig) |
| AIDA Ostseetour | 174 | Deutschland/Ostsee+02:00, Finnland/Russland+03:00, Schweden+02:00 |
| Prag 2014 | 128 | einheitlich MESZ +02:00 |
| Norwegen mit Aida | 109 | einheitlich MESZ +02:00 (Norwegen/Dänemark/Deutschland alle gleiche Zone) |
| Exeter | 102 | einheitlich BST +01:00 (UK) |
| Frankreich 2015 | 91 (81 exportiert, 10 Dubletten) | einheitlich MESZ +02:00 |
| Weber Grillakademie 2016 | 80 | einheitlich MESZ +02:00 (Deutschland, 1 Tag) |
| Rovaniemi 2024 | 65 | Finnland+02:00 (17.-19.12.), Niederlande/Rückreise+01:00 (20.12.) |
| Heide Park | 54 | einheitlich MESZ +02:00 (Deutschland, 1 Tag) |
| Klassenfahrt | 52 | einheitlich MESZ +02:00 (Deutschland) |
| F Scan | 69 | Deutschland Winter+01:00 (2008-01-01, wahrscheinlich Scan-Platzhalterdatum), Sommer+02:00 (2019-05, evtl. echtes Scan-Datum nicht Aufnahmedatum) |
| Ostsee-Warnemünde | 32 | einheitlich CET +01:00 (Deutschland, Winter) |
| Abschlussball_12.2015 | 31 | einheitlich CET +01:00 (Leipzig, Dezember) |

**Bekannte Unsicherheiten (niedrige Priorität, könnten bei Bedarf nachkorrigiert werden):**
- Westcoast-Rundreise 25.8.: Utah vs. weiterhin Arizona nicht 100% sicher (GPS-Grenzbereich).
- AIDA Spanien/Portugal 13.10.: Cádiz (Spanien) angenommen, Nutzer war sich nicht sicher.
- AIDA Spanien/Portugal 16.10. (Seetag): Zielhafen-Zeitzone angenommen.
- Rovaniemi 2024 17.12. (Anreisetag): Zielort-Zeitzone (Finnland) angenommen für den ganzen Tag trotz Teilstrecke in Deutschland.
- F Scan: Datumsfelder sind bei gescannten Altfotos vermutlich Scan-Datum, nicht echtes Aufnahmedatum — Zeitzonen-Fix hier von geringer praktischer Bedeutung.

## Nächster Schritt
osxphotos-Export-Migrationsblock ist fertig. Als nächstes: Immich-Import der exportierten Ordner unter `~/immich-missing/`, idealerweise erst mit einem kleinen Testalbum (z.B. Abschlussball_12.2015 oder Ostsee-Warnemünde, klein und bereits vollständig verifiziert) um den Immich-Import-Workflow (CLI oder Web-Upload, Deduplizierung, Alben-Zuordnung) zu prüfen, bevor der komplette Bestand hochgeladen wird. Das ist ein neues Themenfeld (Immich-seitig), nicht mehr osxphotos-Export.

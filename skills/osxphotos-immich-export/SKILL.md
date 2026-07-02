# osxphotos Export für Immich-Migration

## Kontext
Michael migriert ~16.000 Fotos von Apple Photos auf selbst gehostete Immich-Instanz (datenbank.m-krueger.de). Fotos-Bibliothek liegt lokal unter `/Users/michaelkruger/Pictures/Photos Library.photoslibrary`. In geteilten Alben fehlten (Stand Session 1) **2.618 Originale**, verteilt über Reisealben von 2008 bis heute. Priorität: EXIF-Metadatenintegrität (Datum, GPS), da Immich diese Felder direkt aus der Datei liest. Export-Zielordner: `~/immich-missing/<Albumname>` (Slashes im Albumnamen werden zu `_`).

**Stand Ende Session 4: Der komplette Migrationsblock "geteilte Alben mit fehlenden Originalen" ist fertig — exportiert, zeitzonenkorrigiert UND nach Immich importiert. 20 Alben, 3.133 Assets auf dem Server. Dieser Themenblock ist abgeschlossen.**

## Immich-Import (Session 4, neu)
- CLI: `immich` (installiert via npm, `/Users/michaelkruger/.npm-global/bin/immich`, Version 2.7.5), Auth liegt in `~/.config/immich/auth.yml` (URL bereits inkl. `/api`-Suffix — bei eigenen curl/API-Aufrufen NICHT nochmal `/api` anhängen, sonst 404 "Cannot GET /api/api/...").
- Import-Befehl: `immich upload -r -a <verzeichnis>` — `-r` rekursiv, `-a` erstellt automatisch ein Album pro Ordnername (oberste Ebene unter dem Zielverzeichnis). Vorher immer `--dry-run` zur Kontrolle (zeigt Anzahl neuer Dateien, Duplikate, geplante Alben).
- **Album-Zuordnung passiert als separater Schritt NACH dem kompletten Upload aller Dateien** — während des Uploads erscheinen Fotos in der App zunächst OHNE Album-Zuordnung, das ist kein Fehler, sondern normales Zwischenverhalten bei einem laufenden Import. Erst am Ende: "Successfully created N new albums" / "Successfully updated M assets".
- Bei ~3.143 Dateien / 9,8 GB dauerte der Upload ca. 25-30 Minuten (Hintergrundprozess, `immich server-info` zeigt Zwischenstand via Statistics.Total).
- Vereinzelte transiente Fehler ("TypeError: fetch failed") bei einzelnen großen Dateien (HEIC/DNG) sind normal — der Befehl ist idempotent (erkennt bereits hochgeladene Dateien über Hash als "duplicates" und überspringt sie), einfach denselben `immich upload -r -a .`-Befehl nochmal laufen lassen, dann werden nur die vorher fehlgeschlagenen Dateien nachgeholt.
- Serverseitige Content-Duplikate (identische Datei taucht z.B. durch zwei unterschiedliche Dateinamen im selben Album auf, siehe Fallstrick 3 unten) werden von Immich selbst erkannt und nur einmal gespeichert — führt zu leichten Abweichungen (wenige Dateien) zwischen lokaler Dateizahl und Server-Assetcount pro Album, kein Datenverlust.
- Album-Liste mit Assetzahlen prüfen: `curl -s -H "x-api-key: $(grep '^key:' ~/.config/immich/auth.yml | awk '{print $2}')" "$(grep '^url:' ~/.config/immich/auth.yml | awk '{print $2}')/albums"` (JSON, Feld `assetCount`).
- Ergebnis Session 4: 20 Alben erfolgreich angelegt, 3.133 Assets total, alle Albennamen entsprechen den Ordnernamen unter `~/immich-missing/`.

## ⚠️⚠️ Die drei wichtigsten Fallstricke beim osxphotos-Export (Session 2+3)

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

### 3. Alte, VOR dem Zeitzonen-Fix exportierte Dateien im Zielordner erzeugen stille Duplikate mit falschem Datum
Mehrere Alben hatten bereits vor Session 3 vollständige Exporte im Zielordner (erkennbar an vorhandener `.osxphotos_export.db`-Datei). Diese alten Exporte haben den tzoffset=None-Bug NICHT korrigiert bekommen. Da `export_shared_album.py`/CLI bei Namenskollision NICHT überschreibt sondern ` (1)`, ` (2)` anhängt, entstehen doppelte Versionen (alt=falsche Zeitzone, neu=korrekt) — stille Kontamination ohne Fehlermeldung.

**Sicherer Fix:** Aus dem Export-Log alle `(N)`-Suffix-Zeilen extrahieren → Kandidatenpaar (alt, neu). Per gebündeltem `exiftool -j -GPSLatitude# -GPSLongitude# -DateTimeOriginal <alt> <neu>` vergleichen: GPS/Datum matchen (Toleranz ~0.01° / ~15h) → dasselbe Foto, alte Datei löschen. Weichen ab → unterschiedliche Fotos (zufällige Namensgleichheit), beide behalten. Bei Videos zusätzlich Duration/FileSize vergleichen. **Am saubersten: bei zukünftigen Alben IMMER erst `rm -rf` auf den Zielordner, dann frisch exportieren** — aber vorher prüfen ob ein SEPARATES reguläres (nicht-geteiltes) Album mit demselben Namen existiert (bestätigt für Prag 2014: 126, Jugendweihe: 39 lokale Fotos) — deren Dateien NICHT löschen, die haben bereits korrekten tzoffset und sind kein Teil des Bugs.

## Diagnose-Falle: RetryError/ValueError bei 100% eines Albums NICHT als Python/Pfad-Bug fehldeuten
Wenn `osxphotos export` bei EINEM kompletten Album mit `RetryError[<Future ... raised ValueError>]` durchfällt: **Erste Vermutung ist "fehlendes Original" (path=None, iCloud-Freigabe), NICHT Python-Version oder Pfad-Encoding.**

## Setup
- osxphotos venv: `osxphotos-env` (Python 3.14.6, Homebrew python@3.14 Framework-Build), Version 0.76.1
- exiftool: `/usr/local/bin/exiftool`
- Volle Festplattenzugriff für die ausführende App/Terminal aktiviert, sonst `PermissionError` beim Kopieren der `Photos.sqlite`

## Bekannte Stolperfallen (kompakt)
1. `--original-name` existiert nicht → `--filename "{original_name}"`.
2. Zielverzeichnis muss vorher existieren (`mkdir -p`).
3. exiftool separat installieren.
4. `{original_name}` bei reimportierten Fotos evtl. UUID statt Klarname.
5. Bracketed-paste-Artefakte bei Copy-Paste → `unset zle_bracketed_paste`.
6. RetryError/ValueError auf 100% eines Albums = fehlendes Original.
7. `--use-photokit` = SIGABRT-Crash, nie verwenden.
8. `tzoffset=None` bei geteilten Alben → Datum falsch, siehe Fallstrick 2.
9. `osxphotos timewarp` ohne `--force` crasht ohne TTY.
10. `update=True` ohne persistente `export_db` kann bei Namenskollision fälschlich überspringen.
11. Albumnamen mit `/` → im Zielordnernamen zu `_` sanitizen, bei `timewarp --album` als `//` escapen.
12. Alte Exporte im Zielordner vor Zeitzonen-Fix → stille Duplikate, siehe Fallstrick 3.
13. Immich-Auth-URL in `~/.config/immich/auth.yml` enthält bereits `/api` — bei eigenen curl-Aufrufen nicht doppelt anhängen.

## Geteilte Alben — Migrationsblock: KOMPLETT ABGESCHLOSSEN (Export + Zeitzonenkorrektur + Immich-Import)

| Album | Lokale Dateien | Immich Assets | Route/Zeitzone(n) |
|---|---|---|---|
| AIDA Kanarische Inseln | 362 | 358 | einheitlich WET +00:00 (Kanaren+Madeira, Winter) |
| Westcoast-Rundreise | 326 | 325 | Kalifornien/Arizona -07:00, Utah -06:00 (25.8., unsicher), Nevada -07:00 |
| Prag 2014 | 268 | 268 | einheitlich MESZ +02:00 |
| AIDA Spanien/Portugal | 250 | 249 | Spanien MESZ+02:00, Lissabon WEST+01:00 |
| Rom | 238 | 238 | einheitlich MESZ +02:00 |
| Jugendweihe | 233 | 233 | einheitlich MESZ +02:00 |
| Südostasien | 205 | 205 | Finnland+02:00, Singapur/Malaysia+08:00, Thailand/Vietnam+07:00, Hongkong+08:00 |
| USA Eastcoast | 204 | 204 | Eastern -04:00, Central -05:00 (Nashville/Memphis/New Orleans/Panama City), Eastern -04:00 |
| Mallorca 2018 | 195 | 195 | einheitlich MESZ +02:00 |
| AIDA Ostseetour | 176 | 175 | Deutschland/Ostsee+02:00, Finnland/Russland+03:00, Schweden+02:00 |
| Norwegen mit Aida | 110 | 109 | einheitlich MESZ +02:00 |
| Exeter | 102 | 102 | einheitlich BST +01:00 (UK) |
| Frankreich 2015 | 91 | 91 | einheitlich MESZ +02:00 |
| Weber Grillakademie 2016 | 80 | 80 | einheitlich MESZ +02:00 |
| F Scan | 69 | 67 | Deutschland Winter+01:00 (2008), Sommer+02:00 (2019, evtl. Scan-Datum) |
| Rovaniemi 2024 | 65 | 65 | Finnland+02:00 (17.-19.12.), Niederlande+01:00 (20.12.) |
| Heide Park | 54 | 54 | einheitlich MESZ +02:00 |
| Klassenfahrt | 52 | 52 | einheitlich MESZ +02:00 |
| Ostsee-Warnemünde | 32 | 32 | einheitlich CET +01:00 |
| Abschlussball_12.2015 | 31 | 31 | einheitlich CET +01:00 |
| **GESAMT** | **3.143** | **3.133** | |

**Bekannte Unsicherheiten (niedrige Priorität):** Westcoast-Rundreise 25.8. (Utah vs. Arizona), AIDA Spanien/Portugal 13.10. (Cádiz angenommen) und 16.10. (Zielhafen-Zeitzone), Rovaniemi 17.12. (Zielort-Zeitzone für ganzen Anreisetag), F Scan (Datumsfelder vermutlich Scan- nicht Aufnahmedatum).

## Nächster Schritt
Dieser Migrationsblock (geteilte Alben mit fehlenden Originalen) ist fertig — Export, Zeitzonenkorrektur und Immich-Import abgeschlossen. Für die verbleibenden ~16.000 Fotos der Gesamtmigration (lokale/nicht-geteilte Alben, restliche Bibliothek) ist noch kein Workflow etabliert — das wäre ein neuer, separater Arbeitsblock falls als nächstes gewünscht.

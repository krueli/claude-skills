# osxphotos Export für Immich-Migration

## Kontext
Michael migriert die komplette Apple Photos Bibliothek (19.766 Fotos) auf selbst gehostete Immich-Instanz (datenbank.m-krueger.de). Bibliothek liegt lokal unter `/Users/michaelkruger/Pictures/Photos Library.photoslibrary`. Priorität: EXIF-Metadatenintegrität (Datum, GPS), da Immich diese Felder direkt aus der Datei liest. Export-Zielordner: `~/immich-missing/<Name>` (ein Ordner pro Album bzw. ein Sammelordner für nicht-albumierte Fotos).

**Stand Ende Session 6: Alle 54 Alben (geteilt + regulär) sind exportiert und importiert (6.485 Assets). Zusätzlich alle 13.822 nicht-albumierten Fotos exportiert (17.848 Dateien inkl. Live-Photo-Begleiter/bearbeitete Versionen, 55-59 GB) und werden gerade importiert — OHNE Album-Zuordnung (Immich-Timeline), da diese Fotos in Apple Photos auch in keinem Album waren.**

## Nicht-albumierte Fotos (Session 6, neu)
- Filter: `osxphotos export DEST --not-in-album --download-missing --exiftool --verbose` (osxphotos query-Flag `--not-in-album`, Gegenstück `--in-album`). Funktioniert auch für `timewarp` (gleiche Query-Flags über alle Commands hinweg).
- Von 19.766 Fotos gesamt waren 13.822 in KEINEM Album (überraschend hoher Anteil, ~70%).
- Nur 14 davon `path=None` (fehlend) — alle NICHT `shared`, sondern normale "iCloud optimiert Speicher"-Platzhalter mit korrektem tzoffset. Normales `--download-missing` (ohne AppleScript-Zwang) reicht.
- 144 hatten `tzoffset=None` OHNE dass sie shared waren (alte Fotos 2012-2017, keine GPS-Daten). Da keine Reise-/GPS-Struktur zum Segmentieren vorhanden ist: pauschal auf Heimat-Zeitzone gesetzt via **benannte IANA-Zeitzone** (DST-automatisch!): `osxphotos timewarp --uuid-from-file <datei mit UUIDs> --timezone "Europe/Berlin" --force`. Deutlich einfacher als manuelle Offset-Berechnung, wenn ein Zeitraum mehrere Jahre/DST-Wechsel abdeckt. `--uuid-from-file` ist der richtige Weg, um eine gezielte UUID-Teilmenge zu behandeln, wenn keine Alben-Query passt (z.B. nur die tzoffset=None-Teilmenge, NICHT alle nicht-albumierten Fotos — sonst überschreibt man bei den anderen 13.678 Fotos die bereits korrekte Zeitzone!).
- **Beim Immich-Import für diesen Sammelordner `-a`-Flag WEGLASSEN** (`immich upload -r <ordner>` ohne `-a`), sonst entsteht ein unerwünschtes künstliches Album mit dem Ordnernamen (z.B. "Ohne_Album"). Fotos landen dann korrekt nur in der Timeline, ohne Album-Zuordnung — spiegelt den Zustand in Apple Photos wider.
- Bei ~13.822 Fotos (55GB) dauerte der Export ca. 45 Minuten (WhatsApp-Tempo als Vergleich: 1224 Fotos in 94s, hier deutlich langsamer wegen höherem Anteil an großen Dateien/PNGs/Screenshots).
- 5 von 13.822 Fotos hatten einen Datei-Typ-Mismatch (Inhalt passt nicht zur Erweiterung, z.B. eine ".JPG" ist tatsächlich HEIF/MOV-Container) — exiftool verweigert zu Recht das Schreiben und meldet "Error exporting ... Error writing output file" bzw. "Not a valid JPEG (looks more like X)". **Wichtig: Das ist KEIN Datenverlust** — osxphotos kopiert die Datei ERST, versucht DANACH exiftool-Metadaten zu schreiben. Schlägt nur der exiftool-Schritt fehl, liegt die vollständige, gültige Bilddatei trotzdem schon im Zielordner (ohne die zusätzlichen exiftool-Tags wie Keywords/Album-Namen, aber mit allen original-eingebetteten Metadaten). Vor jedem "Recovery"-Versuch erst prüfen ob die Datei nicht ohnehin schon vollständig da ist (`file <pfad>` zur Validierung), bevor man Zeit in einen erneuten Export steckt.

## Immich-Import
- CLI: `immich` (npm, `/Users/michaelkruger/.npm-global/bin/immich`, Version 2.7.5), Auth in `~/.config/immich/auth.yml` (URL enthält bereits `/api` — bei eigenen curl/API-Aufrufen NICHT doppelt anhängen, sonst 404).
- Für Alben-Ordner: `immich upload -r -a <verzeichnis>` (`-a` = automatische Albenerstellung pro Ordnername).
- Für Sammelordner ohne Album-Zuordnung: `immich upload -r <verzeichnis>` (KEIN `-a`).
- Vorher immer `--dry-run`. Bei großen Datenmengen (>10GB) dauert allein das Crawling/Hashing für den Dry-Run mehrere Minuten — im Hintergrund laufen lassen.
- Album-Zuordnung passiert als separater Schritt NACH dem kompletten Upload aller Dateien.
- Idempotent: bereits hochgeladene Dateien werden per Hash als "duplicates" erkannt und übersprungen. Transiente Einzeldatei-Fehler ("TypeError: fetch failed") → gleichen Befehl nochmal laufen lassen.
- Album-Liste mit Assetzahlen: `curl -s -H "x-api-key: $(grep '^key:' ~/.config/immich/auth.yml | awk '{print $2}')" "$(grep '^url:' ~/.config/immich/auth.yml | awk '{print $2}')/albums"` (JSON, Feld `assetCount`).
- **Upload-Tempo variiert stark:** 9,8GB/3143 Dateien ≈ 28 Min; 8GB/3735 Dateien ≈ 25 Min (schwankt je nach Dateigröße/Servertagesform). Bei sehr großen Batches (>50GB) mehrere Stunden einplanen.

## Import-Historie
| Batch | Dateien | Größe | Alben | Ergebnis |
|---|---|---|---|---|
| 1 (Session 4) | 3.143 | 9,8 GB | 20 (mit `-a`) | 3.133 Assets, 5 transiente Fehler beim Retry behoben |
| 2 (Session 5) | 3.735 neu | 8 GB | 34 weitere (mit `-a`) | 0 Fehler, Gesamt 6.485 Assets in 54 Alben |
| 3 (Session 6) | 17.847 | 59,2 GB | keine (ohne `-a`, Timeline) | läuft/lief |

## ⚠️ Zielordner-NAMEN-Bug: bestimmte Namen lassen `osxphotos export` mit RetryError/ValueError bei JEDEM Foto scheitern
Betroffen waren "Schrammsteine -Sächsische Schweiz", "Sprüche", "Hafling_Südtirol". Exakter Trigger unklar (nicht einfach "Umlaut" — "Südostasien"/"Ostsee-Warnemünde" funktionierten problemlos). **Diagnose:** Export schlägt für JEDES Foto im Album fehl, auch nach `rm -rf` des Zielordners, auch ohne `--filename`-Template. Direkter Python-Aufruf (`PhotoExporter(photo).export(...)`) auf dasselbe Foto funktioniert einwandfrei. **Test:** Export mit demselben `--album`, aber simplem Zielordnernamen (`/tmp/test123`) — funktioniert das, liegt es am Zielordner-NAMEN. **Workaround:** In temporären, simpel benannten Ordner exportieren, danach mit `mv` auf korrekten Namen umbenennen.

## Die drei wichtigsten osxphotos-Export-Fallstricke (Session 2+3)

### 1. `--use-photokit` funktioniert NICHT für CLI-Skripte (SIGABRT/TCC-Crash)
Nie verwenden. Ersatz: `ExportOptions(download_missing=True, use_photos_export=True, exiftool=True)` + `PhotoExporter` (AppleScript-Automation der autorisierten Photos-App). Fertiges Skript: `~/immich-missing/export_shared_album.py "<Albumname>"` — nur nötig für Alben die zu **100%** aus fehlenden/Cloud-only-Fotos bestehen. Bei **gemischten** Alben funktioniert normale CLI mit `--download-missing --exiftool` zuverlässig und schneller.

### 2. Geteilte/Cloud-Only-Fotos (und manche alte lokale Fotos ohne GPS) haben `tzoffset=None` → `--exiftool` schreibt Datum in falscher Zeitzone
`photo.date` wird als UTC zurückgegeben, `--exiftool` schreibt `OffsetTimeOriginal +00:00` — ehrlich aber falsche Lokalzeit. Fix: GPS-Route pro Datum analysieren → Zeitzone(n) bestimmen (Nachbarländer wie Spanien/Portugal, US-Bundesstaaten-Grenzen beachten) → `osxphotos timewarp --album/--uuid-from-file ... --timezone +HH:MM|IANA-Name --force` (ohne `--match-time`) → GENAU DIESE UUIDs/Alben treffen, nicht versehentlich Fotos mit bereits korrekter Zeitzone überschreiben → Zielordner löschen und neu exportieren.

### 3. Alte, vor dem Zeitzonen-Fix exportierte Dateien im Zielordner erzeugen stille Duplikate
Erkennbar an vorhandener `.osxphotos_export.db`. Bei Namenskollision hängt osxphotos ` (1)`, ` (2)` an statt zu überschreiben. Fix: `(N)`-Suffix-Paare aus Log extrahieren, per gebündeltem `exiftool -j -GPSLatitude# -GPSLongitude# -DateTimeOriginal <alt> <neu>` vergleichen (Match → alte löschen, Mismatch → beide behalten). **Sauberste Lösung: bei künftigen Exporten immer erst `rm -rf`, dann frisch exportieren** — vorher auf separates reguläres Album gleichen Namens prüfen.

## Diagnose-Falle: RetryError/ValueError bei 100% eines Albums
Zwei mögliche Ursachen: 1) Zielordner-Name-Bug (Test mit `/tmp/test123`), 2) fehlendes Original bei 100% des Albums (`osxphotos query --album X --missing | wc -l`).

## Setup
- osxphotos venv: `osxphotos-env`, Python 3.14.6, osxphotos 0.76.1
- exiftool: `/usr/local/bin/exiftool`
- Volle Festplattenzugriff für ausführende App/Terminal aktiviert
- CLI-Export in Ordner mit vorhandener `.osxphotos_export.db`: braucht `echo y |` oder `--update`, sonst Crash ohne TTY.
- `timewarp` ohne `--force` crasht ohne TTY (`click.confirm`).

## Bekannte Stolperfallen (kompakt)
1. `--filename "{original_name}"` kann bei Fotos mit ungewöhnlichen/leeren Originalnamen selbst zum RetryError führen → im Zweifel weglassen, Default nutzt ohnehin `original_filename`.
2. Zielverzeichnis muss vorher existieren (`mkdir -p`).
3. exiftool separat installieren.
4. RetryError/ValueError auf 100% eines Albums → siehe Diagnose-Falle.
5. `--use-photokit` = SIGABRT, nie verwenden.
6. `tzoffset=None` → Datum falsch, siehe Fallstrick 2 oben. Betrifft auch NICHT-shared alte Fotos ohne GPS (nicht nur shared/cloud-only).
7. `osxphotos timewarp` ohne `--force` crasht ohne TTY.
8. Albumnamen mit `/` → im Zielordnernamen zu `_` sanitizen, bei `timewarp --album` als `//` escapen.
9. Alte Exporte im Zielordner vor Zeitzonen-Fix → stille Duplikate.
10. Bestimmte Zielordner-Namen → RetryError bei 100% der Fotos, siehe Bug oben. Workaround: temporärer Ordnername + `mv`.
11. Immich-Auth-URL enthält bereits `/api` — bei curl nicht doppelt anhängen.
12. `--not-in-album` für Fotos ohne Albumzugehörigkeit; beim Import dieser Fotos `-a` WEGLASSEN.
13. Exiftool-Fehler ("Error writing output file", "Not a valid JPEG") bei einzelnen Dateien = Datei-Typ-Mismatch im Quellmaterial — Basisdatei liegt trotzdem schon vollständig im Zielordner, kein Datenverlust, vor Recovery-Versuch erst prüfen.

## Vollständiger Stand (Ende Session 6)
- 54 Alben (geteilt + regulär) exportiert und importiert: 6.485 Assets.
- 13.822 nicht-albumierte Fotos exportiert (17.848 Dateien, 55-59 GB), Import läuft/lief ohne Album-Zuordnung.
- Damit wäre die GESAMTE Bibliothek (19.766 Fotos) migriert, sobald der letzte Import fertig ist und verifiziert wurde.

## Nächster Schritt
Nach Abschluss des dritten Imports: Server-Statistik prüfen (`immich server-info`, sollte ~6.485 + ~17.847 = ~24.332 Assets zeigen, ggf. minus Content-Duplikate), stichprobenartig EXIF-Korrektheit der 144 heimatzeitzone-korrigierten alten Fotos prüfen. Danach ist die komplette Photos-Bibliothek-Migration nach Immich abgeschlossen — kein osxphotos-Export-Thema mehr offen, außer der Nutzer möchte die 2 Alben mit unsicheren Zeitzonen-Annahmen (Westcoast-Rundreise 25.8., AIDA Spanien/Portugal 13./16.10.) nachträglich verifizieren.

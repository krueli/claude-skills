# osxphotos Export für Immich-Migration

## Kontext
Michael migriert ~19.750 Fotos (komplette Photos-Bibliothek, alle Alben) von Apple Photos auf selbst gehostete Immich-Instanz (datenbank.m-krueger.de). Bibliothek liegt lokal unter `/Users/michaelkruger/Pictures/Photos Library.photoslibrary`. Priorität: EXIF-Metadatenintegrität (Datum, GPS), da Immich diese Felder direkt aus der Datei liest. Export-Zielordner: `~/immich-missing/<Albumname>` (ein Ordner pro Album, Slashes im Albumnamen werden zu `_`).

**Stand Ende Session 5: ALLE Alben der Bibliothek (geteilt + regulär, 54 Ordner) sind exportiert. Erster großer Immich-Import (20 Alben mit fehlenden Originalen) ist abgeschlossen und verifiziert. Zweiter Import (restliche 34 Alben) läuft/ist grade gelaufen.**

## Immich-Import
- CLI: `immich` (npm, `/Users/michaelkruger/.npm-global/bin/immich`, Version 2.7.5), Auth in `~/.config/immich/auth.yml` (URL enthält bereits `/api` — bei eigenen curl/API-Aufrufen NICHT doppelt anhängen, sonst 404).
- Befehl: `immich upload -r -a <verzeichnis>` — `-r` rekursiv, `-a` erstellt automatisch ein Album pro Ordnername. Vorher immer `--dry-run`.
- Album-Zuordnung passiert als separater Schritt NACH dem kompletten Upload aller Dateien — während des Uploads sind Fotos in der App kurzzeitig ohne Album sichtbar, das ist normal, kein Fehler.
- Der Befehl ist idempotent: bereits hochgeladene Dateien werden per Hash als "duplicates" erkannt und übersprungen, neue Alben/Ordner werden beim erneuten Aufruf einfach ergänzt. Für den kompletten Bibliotheks-Import kann man daher einfach denselben `immich upload -r -a .` auf das gesamte `~/immich-missing`-Verzeichnis wiederholt laufen lassen, sobald neue Alben dazukommen.
- Transiente Einzeldatei-Fehler ("TypeError: fetch failed") bei einzelnen großen Dateien sind normal, gleicher Befehl nochmal laufen lassen behebt sie (idempotent).
- Album-Liste mit Assetzahlen: `curl -s -H "x-api-key: $(grep '^key:' ~/.config/immich/auth.yml | awk '{print $2}')" "$(grep '^url:' ~/.config/immich/auth.yml | awk '{print $2}')/albums"` (JSON, Feld `assetCount`).
- Ergebnis erster Import (Session 4): 20 Alben, 3.133 Assets, 3,1 GB effektiv neu.
- Zweiter Import (Session 5): 34 weitere Alben (alle geteilten Alben mit fehlenden Originalen die noch offen waren + alle regulären/lokalen Alben), ~3.735 neue Dateien, ~8 GB.

## ⚠️⚠️ NEU (Session 5): Bestimmte Zielordner-NAMEN (nicht Albumnamen!) lassen `osxphotos export` mit RetryError/ValueError bei JEDEM Foto scheitern
Betroffen waren u.a. "Schrammsteine -Sächsische Schweiz", "Sprüche", "Hafling_Südtirol" — alle drei enthalten Umlaute, aber auch "Südostasien" und "Ostsee-Warnemünde" (funktionierten einwandfrei) haben Umlaute, also ist es NICHT einfach "Umlaut im Pfad". Der exakte Trigger ist nicht abschließend geklärt (vermutlich ein Escaping-Bug beim Aufbau des exiftool-Subprozess-Kommandos für bestimmte Zeichen-/Längen-Kombinationen im Zielpfad), aber das Symptom ist eindeutig:

**Diagnose:** `osxphotos export <ZIEL> --album <NAME> ...` schlägt für JEDES einzelne Foto im Album mit `RetryError[<Future ... raised ValueError>]` fehl — auch nach `rm -rf` des Zielordners, auch ohne `--filename`-Template, auch bei erneutem Ausführen (kein transientes Problem). Direkter Python-Aufruf (`PhotoExporter(photo).export(...)`) auf dasselbe Foto funktioniert einwandfrei — das Foto/Album selbst ist NICHT das Problem.

**Test zur Bestätigung:** Export mit demselben `--album`-Wert aber einem simplen Zielordnernamen (z.B. `/tmp/test123`) durchführen — funktioniert das, liegt es am Zielordner-NAMEN, nicht am Album/den Fotos.

**Workaround:** In einen temporären, simpel benannten Ordner exportieren (z.B. `_tmp_<kurzname>`), danach mit `mv` auf den korrekten finalen Albumnamen umbenennen:
```bash
mkdir -p "_tmp_x"
osxphotos export "_tmp_x" --album "<Problematischer Name>" --download-missing --exiftool --verbose
mv "_tmp_x" "<Problematischer Name>"
```
Bei jedem neuen Album mit Sonderzeichen im Namen, das mit demselben RetryError-Muster bei 100% der Fotos fehlschlägt (nicht nur bei den tatsächlich fehlenden Originalen), zuerst mit einem simplen Testordnernamen verifizieren, ob es an diesem Bug liegt, bevor man tiefer in Richtung "fehlendes Original"/Zeitzone diagnostiziert.

## Die drei wichtigsten Fallstricke beim osxphotos-Export (Session 2+3)

### 1. `--use-photokit` / `use_photokit=True` funktioniert grundsätzlich NICHT für CLI-Skripte
Führt zu `SIGABRT` (TCC-Crash: fehlendes App-Sandbox-Entitlement). Nie verwenden. **Funktionierender Ersatz:** AppleScript-Automation der bereits autorisierten Photos-App via `ExportOptions(download_missing=True, use_photos_export=True, exiftool=True)` + `PhotoExporter`. Fertiges Skript: `~/immich-missing/export_shared_album.py "<Albumname>"` — nur nötig für Alben, die zu **100%** aus fehlenden/Cloud-only-Fotos bestehen (dort schlägt die normale CLI mit `--download-missing` fehl, siehe Diagnose-Falle unten). Bei **gemischten** Alben (nur ein Teil fehlt) funktioniert die normale CLI mit `--download-missing --exiftool` zuverlässig und ist schneller (kein AppleScript-Zwang für die bereits lokalen Fotos).

### 2. Geteilte/Cloud-Only-Fotos haben `tzoffset=None` → `--exiftool` schreibt Datum in falscher Zeitzone
Alle Fotos mit `path=None`/`shared=True` haben `photo.tzoffset == None`. `--exiftool` schreibt dann UTC mit `OffsetTimeOriginal +00:00` — ehrlich, aber falsche Lokalzeit (bei großen Zeitverschiebungen falsches Kalenderdatum).

**Fix-Workflow (VOR dem finalen Export):**
1. GPS-Koordinaten pro Datum auswerten (`p.location`, gruppiert nach `p.date.date()`), Reiseabschnitte/Länder identifizieren.
2. Nachbarländer mit unterschiedlicher Zeitzone trotz gemeinsamer Grenze beachten: Spanien (MEZ/MESZ) vs. Portugal (WET/WEST). Kanaren+Madeira nutzen WET, nicht spanisches Festland-MEZ.
3. US-Bundesstaaten: Eastern/Central-Grenze durch TN/AL/FL-Panhandle. Arizona keine DST. Nevada = Pacific trotz Nachbarschaft zu Mountain-Staaten.
4. Route bei Unsicherheit mit Nutzer verifizieren, bei "weiß nicht mehr" plausibelste Annahme (Zielort-Heuristik) nehmen und dokumentieren.
5. Pro Segment: `osxphotos timewarp --album "<Album>" --from-date YYYY-MM-DD --to-date YYYY-MM-DD --timezone +HH:MM --force`. Albumnamen mit `/` → `//` escapen. `--to-date` exklusiv. Ohne `--match-time`. Vorher `--inspect` zur Kontrolle. `--force` nötig ohne TTY.
6. Bereits vorhandene Exporte komplett löschen (`rm -rf`) vor Neu-Export — außer bei separatem regulärem Album gleichen Namens (siehe Fallstrick 3).
7. Stichprobe: `exiftool -DateTimeOriginal -OffsetTimeOriginal -GPSLatitude -GPSLongitude <Datei>`.

### 3. Alte, vor dem Zeitzonen-Fix exportierte Dateien im Zielordner erzeugen stille Duplikate
Erkennbar an vorhandener `.osxphotos_export.db`. Bei Namenskollision hängt osxphotos ` (1)`, ` (2)` an statt zu überschreiben → alte (falsche Zeitzone) und neue (korrekt) Version liegen doppelt. Sicherer Fix: aus Export-Log `(N)`-Suffix-Paare extrahieren, per gebündeltem `exiftool -j -GPSLatitude# -GPSLongitude# -DateTimeOriginal <alt> <neu>` vergleichen (GPS/Datum-Match → alte löschen, Mismatch → beide behalten, zufällige Namensgleichheit). Bei Videos zusätzlich Duration/FileSize. **Sauberste Lösung: bei künftigen Alben immer erst `rm -rf`, dann frisch exportieren** — vorher prüfen ob separates reguläres Album gleichen Namens existiert (dessen Fotos NICHT löschen, die haben korrekten tzoffset).

## Diagnose-Falle: RetryError/ValueError bei 100% eines Albums NICHT vorschnell als Python/Pfad-Bug fehldeuten
Zwei mögliche Ursachen, in dieser Reihenfolge prüfen:
1. **Zielordner-Name-Bug (siehe oben, Session 5 NEU):** Test mit simplem Zielordnernamen (`/tmp/test123`) — funktioniert das, ist es der Ordnername.
2. **Fehlendes Original (path=None, 100% des Albums):** `osxphotos query --album "<Albumname>" --missing | wc -l` — steht die Zahl nah an der Albumgröße, ist es das. Braucht dann `export_shared_album.py` (AppleScript-Zwang) statt normaler CLI.

## Setup
- osxphotos venv: `osxphotos-env` (Python 3.14.6, Homebrew python@3.14 Framework-Build), Version 0.76.1
- exiftool: `/usr/local/bin/exiftool`
- Volle Festplattenzugriff für die ausführende App/Terminal aktiviert, sonst `PermissionError` beim Kopieren der `Photos.sqlite`
- Bei CLI-Export in einen Ordner mit vorhandener `.osxphotos_export.db`: interaktiver Bestätigungsprompt ("continue without --update?") — braucht `echo y |` vorangestellt oder `--update`-Flag, sonst Crash-Log bei nicht-interaktiver Ausführung.

## Bekannte Stolperfallen (kompakt)
1. `--original-name` existiert nicht → `--filename "{original_name}"` (Achtung: kann bei Fotos mit ungewöhnlichen/leeren Originalnamen selbst zum RetryError führen — im Zweifel `--filename` weglassen, Default-Verhalten nutzt ohnehin `original_filename`).
2. Zielverzeichnis muss vorher existieren (`mkdir -p`).
3. exiftool separat installieren.
4. Bracketed-paste-Artefakte bei Copy-Paste → `unset zle_bracketed_paste`.
5. RetryError/ValueError auf 100% eines Albums → siehe Diagnose-Falle oben (zwei mögliche Ursachen).
6. `--use-photokit` = SIGABRT-Crash, nie verwenden.
7. `tzoffset=None` bei geteilten Alben → Datum falsch, siehe Fallstrick 2.
8. `osxphotos timewarp` ohne `--force` crasht ohne TTY.
9. Albumnamen mit `/` → im Zielordnernamen zu `_` sanitizen, bei `timewarp --album` als `//` escapen.
10. Alte Exporte im Zielordner vor Zeitzonen-Fix → stille Duplikate, siehe Fallstrick 3.
11. Bestimmte Zielordner-Namen (Sonderzeichen-Kombination) → RetryError bei 100% der Fotos, siehe NEU-Abschnitt oben. Workaround: temporärer Ordnername + `mv`.
12. Immich-Auth-URL in `~/.config/immich/auth.yml` enthält bereits `/api` — bei eigenen curl-Aufrufen nicht doppelt anhängen.
13. CLI-Export in Ordner mit vorhandener `.osxphotos_export.db` braucht `echo y |` oder `--update`, sonst Crash ohne TTY.

## Vollständiger Alben-Bestand (Stand Session 5, alle exportiert)
54 Ordner unter `~/immich-missing/`: alle 20 geteilten Alben mit ursprünglich fehlenden Originalen (siehe Session-3-Historie, alle zeitzonenkorrigiert), plus 7 weitere geteilte Alben mit teilweise fehlenden Originalen (Teneriffa, Gardasee 2019, Fuerteventura, Hafling_Südtirol, Mallorca - Calla Millor, Reiten Grimma, Abschlussball_03.2016 — alle ebenfalls zeitzonenkorrigiert), plus Texel und Wien 2014 (geteilt, keine fehlenden Originale), plus 26 reguläre/lokale Alben (WhatsApp, AIDA Ostsee Route, Sprüche, Frankreich-Nancy, Silvester 2016, Familie, Aquarium, Wien 2014 [gemergt mit geteiltem], Klassentreffen 2018, Abschlussball 12_2015, Rezepte, Josi, Hohem Joy, Englischunterricht, Instagram, Mutti, Weber Grillkurs 02.10.2014, Geheimakte Zahn, Schrammsteine -Sächsische Schweiz, Bike, Bewerbungsfotos, InstantSave, Silvester, Unheilig, Abschlussball 03-2016, Balkonkraftwerk). Reguläre Fotos haben immer korrekten tzoffset (kein Zeitzonen-Fix nötig).

**Wichtig:** "Abschlussball 12_2015"(regulär,31) und "Abschlussball_12.2015"(geteilt,31) sowie "Abschlussball 03-2016"(regulär,10) und "Abschlussball_03.2016"(geteilt,8) sind trotz ähnlicher Namen/Zahlen KOMPLETT unterschiedliche Fotosets (0 UUID-Überschneidung, verifiziert) — bewusst als separate Alben/Ordner behandelt, nicht zusammengeführt.

## Nächster Schritt
Alle Alben exportiert und (zweiter Batch) nach Immich importiert. Nach Abschluss des zweiten Imports: Ergebnis via Album-API verifizieren (Anzahl Alben=54, Assetzahlen plausibilisieren), stichprobenartig EXIF-Korrektheit bei den neu zeitzonenkorrigierten Alben (Teneriffa, Gardasee 2019, Fuerteventura, Hafling_Südtirol, Mallorca-CallaMillor, Reiten Grimma, Abschlussball_03.2016) prüfen. Danach: Bibliotheks-weite Vollständigkeit gegenchecken (Photos.app hat 19.753 Fotos gesamt inkl. Fotos ohne Albumzugehörigkeit — ob diese auch migriert werden sollen, ist noch nicht geklärt/gewünscht, bisher nur Album-Mitglieder exportiert).

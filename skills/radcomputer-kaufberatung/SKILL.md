# Radcomputer-Kaufberatung (Garmin Edge / Hammerhead Karoo)

## Kontext
Michael fährt Rennrad und e-Bike (Cube Stereo One, TQ-HPR50 Motor), nutzt eine Garmin Fenix 8 aber NUR für Vital-/Schlafdaten (kein Trainingsdaten-Sync mit Edge nötig). Ausgangslage: Garmin Edge 840, Hauptproblem ist die kleine Displaygröße (2,6"), nicht Helligkeit/Sonnenlicht-Ablesbarkeit.

## Methodik: Erst Problem isolieren, dann Gerät empfehlen
Bei "soll ich auf Gerät X upgraden" immer zuerst klären, WAS genau das Problem ist, bevor ein Gerät empfohlen wird. Beispiel: "Display schlecht ablesbar" kann heißen (a) zu klein, (b) zu dunkel bei Sonne, oder (c) beides – das führt zu unterschiedlichen Empfehlungen (a → größeres Gerät reicht, b → andere Display-Technologie nötig).

## Garmin Edge Generationen-Vergleich (Stand 2026)
- **Edge 840 → Edge 1040**: Gleiche Display-Technologie (MIP), nur größer (3,5" vs 2,6") und mehr Akku. KEIN Technologie-Sprung. Lohnt sich nur bei reinem Größenproblem, nicht bei Sonnenlicht-Ablesbarkeit.
- **Edge 1050**: Neues LCD-Display mit 1000 Nits, deutlich besser bei direkter Sonne, plus Sprachnavigation und Touchscreen auf Smartphone-Niveau. Aber schwerer, teurer (~650€), kürzere Akkulaufzeit als 1040 Solar.
- **Edge 1040 ist auslaufendes Modell** seit 1050-Launch – falls Kauf, dann gebraucht, nicht neu.
- Kein Edge 850 (kompaktes LCD-Gerät) bestätigt, aber naheliegend dass Garmin das irgendwann bringt.

## Hammerhead Karoo als Alternative – Kernpunkte
**Aktueller Stand**: Es gibt KEINEN "Karoo 4". Das aktuelle Modell (seit Mai 2024) heißt nur "Karoo" (oft als "Karoo 3" bezeichnet, von Hammerhead bewusst vermieden). Kein Hinweis auf baldigen Nachfolger; im Gegenteil läuft sogar ein Upgrade-Rabattprogramm für Karoo 1/2-Besitzer auf das aktuelle Modell.

**Display**: 3,2", 480×800px, Gorilla Glass – bei direkter Sonne besser ablesbar als Garmin Edge 1040 (MIP-Technologie), vergleichbar mit Edge 1050.

**Shimano Di2 – KRITISCHER Punkt, immer ansprechen wenn Di2 relevant ist:**
- Seit 2022 offiziell NICHT unterstützt (Shimano hat Hammerhead nach SRAM-Übernahme die Lizenz entzogen – politischer Konzernkonflikt, kein technisches Problem)
- Workaround existiert: Community-App "Ki2" (GitHub, Entwickler @valterc), Sideload via Hammerhead Companion App
- Workaround ist FRAGIL: brach komplett weg beim SDK-Wechsel Ende 2024, war auch Anfang 2026 noch nicht zuverlässig wiederhergestellt, trotz Vermittlungsversuch SRAM/Shimano (von Shimano verweigert)
- Einschätzung: Nur akzeptabel wenn Di2-Anzeige (Batterie, Gang) "nice to have" ist, nicht wenn man sich sicherheitsrelevant darauf verlässt
- Bei SRAM AXS dagegen: native, tiefe Integration, automatische Erkennung aller Komponenten über SRAM-Account

**Garmin Varia Radar**: Offiziell unterstützt über ANT+, inkl. Auto-Light-Control bei Radar-Light-Kombigeräten. ABER: mehrfach dokumentierte Verbindungsabbrüche bei Karoo 3 in Community-Foren, Hammerhead-Support bietet nur "entkoppeln/neu koppeln" als Workaround, keine echte Lösung.

**TQ-HPR50 (e-Bike Motor)**: Beide Systeme (Karoo UND Garmin Edge) unterstützen nur Anzeige (Akku, Reichweite, Unterstützungsstufe, Geschwindigkeit/Trittfrequenz/Leistung) über ANT+ LEV-Profil. KEINES der beiden Systeme kann die Unterstützungsstufen zuverlässig vom Kopfgerät aus steuern (generelles TQ-Problem, nicht geräte-spezifisch). Kein Unterschied zwischen Karoo und Garmin in diesem Punkt.

**Akkulaufzeit – realer Schwachpunkt**: Praxistest zeigt Karoo nach 8:43h bei 227km leer, Edge 1040 Solar im gleichen Zeitraum nur 95%→46%. Bei langen Touren (100km+) relevant, bei kürzeren Alltagsfahrten unkritisch.

## Entscheidungslogik für Garmin-Ökosystem-Nutzer
1. Prüfen ob Garmin-Uhr (Fenix etc.) tatsächlich Trainingsdaten-Sync mit Edge nutzt → wenn nein (nur Vital-/Schlafdaten), entfällt das Hauptgegenargument gegen Systemwechsel
2. Di2 vorhanden? → Kritischster Punkt, Nice-to-have-Test machen (verlässt sich Nutzer aufs Kopfgerät für Akkustand oder checkt er eh in Hersteller-App?)
3. Varia Radar vorhanden? → Funktioniert offiziell, aber mit Stabilitätsrisiko, separat von Di2 bewerten
4. Touren-Länge abfragen → falls regelmäßig 100km+, Akkulaufzeit-Differenz gegenrechnen, ggf. Powerbank als Lösung vorschlagen

## Rechercheansatz für diese Themen
Web-Suchen funktionieren gut mit konkreten Modellnamen + "Kompatibilität"/"Integration" + Jahr. Foren (Hammerhead Support Community, rennrad-news.de, mtb-news.de, DC Rainmaker) liefern verlässlichere Praxisdaten als Hersteller-Marketing-Seiten. Bei Hammerhead/Shimano-Konflikt: Daten sind über Jahre verteilt (2022 Ursprung, 2024 SDK-Bruch, 2026 Status), mehrere Suchen über Zeitverlauf nötig um aktuellen Stand zu bekommen.

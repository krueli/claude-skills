# HA Home-Dashboard (Layout-Stil nach Uwe Kortkamp)

## URL-Pfad
`home-overview` — 3-spaltiges Übersichts-Dashboard im dunklen Stil (#1a1a1a)

## Installierte HACS-Karten
- `custom:button-card` ✓
- `custom:apexcharts-card` ✓
- `card-mod` ✓
- `custom:hourly-weather` — HACS ID 499270202 ✓

## Waste Collection Schedule
- HACS Integration ID: 254347436 ✓ installiert
- Package: `/config/packages/waste_collection.yaml`
- Sensoren: `sensor.muell_gelbe_ton`, `sensor.muell_blaue_ton`, `sensor.muell_graue_ton`, `sensor.muellaebfuhr_naechste`
- Source-Typ: `static` mit expliziten Datumslisten (Gladiolenweg 6B Leipzig)
- Gelbe Tonne: alle 14 Tage dienstags (01.07., 15.07., 29.07. ...)
- Blaue Tonne: alle 14 Tage (29.06., 13.07., 27.07. ...)
- Graue Tonne: alle 14 Tage (30.06., 14.07., 28.07. ...)
- Anzeigereihenfolge in der Müllkarte (Stand 30.06.2026, Wunsch von Michael): Blaue, Graue, Gelbe (Graue in der Mitte, Gelbe am Ende)

## WICHTIG: Sections-Index hat sich verschoben (Stand 30.06.2026)
Das Dashboard hat inzwischen 4 Sections in dieser Reihenfolge:
0. Solar & Energie
1. Klima & Temperaturen
2. Lampen & Steckdosen (NEU dazugekommen — verschiebt alles danach um 1 Index)
3. Netz & Smart Home → cards[1] = "⚡ Geräte Verbrauch", cards[2] = "🗑️ Müllabfuhr"

Vor jeder python_transform-Bearbeitung den Index per `ha_config_get_dashboard(card_type="custom:button-card", include_config=True, url_path="home-overview")` neu verifizieren statt sich auf alte Pfadangaben zu verlassen — die Section-Reihenfolge ändert sich, wenn neue Sections eingefügt werden.

## KRITISCH: write_file_b64 Shell-Command
Parameter heißt `content_b64`, NICHT `b64`:
```python
ha_call_service('shell_command', 'write_file_b64', {
    'path': '/config/packages/datei.yaml',
    'content_b64': 'BASE64_STRING'  # ← content_b64, nicht b64!
})
```
Wenn `b64` verwendet wird, schreibt die Shell-Command eine leere Datei (0 Bytes).
Verifizierung: Backup-Größe mit `list_backups` prüfen — 0 Bytes = falscher Parameter.

## button-card Datenkarte Muster
```yaml
type: custom:button-card
name: "🔋 Titel"
show_icon: false
show_state: false
styles:
  card:
    - background: "#1a1a1a"
    - padding: 12px 16px
    - text-align: left
    - border-radius: 12px
  name:
    - font-size: 15px
    - font-weight: bold
    - color: "#FFD700"
    - padding-bottom: 8px
custom_fields:
  content: >
    [[[ return `<div style='display:flex;justify-content:space-between;padding:2px 0'>
    <span style='color:#aaa'>Label</span>
    <span style='font-weight:bold;color:#fff'>${states['sensor.entity'].state} W</span>
    </div>` ]]]
custom_fields_style:
  content:
    - display: block
    - width: 100%
    - color: "#fff"
    - font-size: 13px
```

## Müllkarte mit Farbkodierung (rot ≤1 Tag, orange ≤2 Tage)
```javascript
[[[ const s = states['sensor.muell_blaue_ton'];
    const d = s ? s.attributes.daysTo : 99;
    const color = d <= 1 ? '#FF4444' : d <= 2 ? '#FF9800' : '#4DA6FF';
    return `<div style='display:flex;justify-content:space-between;padding:2px 0'>
    <span style='color:#4DA6FF'>Blaue Tonne</span>
    <span style='font-weight:bold;color:${color}'>${s ? s.state : 'n/a'}</span></div>` ]]]
```

## Editieren per python_transform — bewährtes Muster für button-card-Inhalte
Der gesamte Kartentext liegt als ein langer JS-Template-String unter `custom_fields.content`.
Statt den ganzen String neu zu schreiben: gezielte `.replace()`-Aufrufe auf eindeutigen Teilstrings nutzen, dann zurückschreiben. Beispiel (TV-Entity-Fix):
```python
content = config['views'][0]['sections'][3]['cards'][1]['custom_fields']['content']
content = content.replace("st('switch.alt')", "st('media_player.neu')")
config['views'][0]['sections'][3]['cards'][1]['custom_fields']['content'] = content
```
Für Reihenfolge-Tausch zweier `<div>...</div>`-Blöcke: beide Blöcke als Variablen definieren (exakter String inkl. aller Anführungszeichen) und `m.replace(a + b, b + a)` aufrufen — robuster als Index-basiertes Slicing im String.

## BEKANNTER BUG: switch.75pml9009_12_bildschirmstatus (Philips TV, JointSpace-API) — NICHT REPARIERBAR
Dieser Switch meldet NIEMALS "on", egal ob der Fernseher läuft oder nicht (geprüft über 48h Historie: nur "off"/"unavailable"). Config-Entry `01KM62F22SKA5CNZPWCT55XY6J` (Platform `philips_js`).
**Reload/Reconfigure der Integration wurde am 30.06.2026 getestet (disable → enable) — hat NICHTS gebracht.** Sofort nach dem Reload meldete der Switch wieder "off", während `media_player.75pml9009_12` zeitgleich korrekt "on" zeigte. Das ist also kein Auth-/Verbindungsproblem, sondern ein struktureller Bug: die Firmware beantwortet den "screenstate"-Endpoint der JointSpace-API nicht zuverlässig. Lässt sich von HA-Seite nicht fixen — würde ein Firmware-Update am Fernseher oder einen Fix in der `philips_js`-Integration selbst brauchen, beides außerhalb von HA-Tooling. NICHT erneut versuchen zu reparieren, nur noch konsequent meiden.
Verwende stattdessen: `media_player.75pml9009_12` (state "on"/"off") — trackt zuverlässig, hat aber `assumed_state: true` (keine aktive Abfrage, sondern Ableitung aus gesendeten Kommandos/App-Events).
Falls der Bug auch in anderen Automationen/Dashboards mit diesem Switch auftaucht: dort ebenfalls auf media_player umstellen. Alle 12 Dashboards wurden am 30.06.2026 durchsucht — nur home-overview hatte ihn referenziert, ist bereits gefixt.

## Hausgeräte-Dashboard (aeg-geraete)
Separates vollständiges Dashboard mit URL-Pfad `aeg-geraete`, enthält:
- Waschluder (Waschmaschine via AEG-Integration)
- Standgebläse (Trockner via AEG-Integration)
- Herbert (Backofen via AEG-Integration)
- Luftikusa (Luftreiniger, Switch + State)
- Spülluder (Geschirrspüler via Miele-Integration)

Jede Sektion hat: Status-Tiles, Restzeit, Programmphase, Steuerbuttons (Start/Pause/Fortsetzen/Stop), Power-Switch.

## Warum der Miele-Geschirrspüler (Spülluder) KEINEN Wattleistungssensor hat
Die Miele-Integration (G7510) liefert KEINEN Echtzeit-Wattleistungssensor. Es gibt nur:
- `sensor.spulluder_energieverbrauch` (kWh, nur bekannt während/nach Programm)
- `sensor.spulluder_energieprognose` (Schätzwert, `unknown` im Idle)
In der Home-Dashboard Geräte-Verbrauchskarte wird daher der Status (`sensor.spulluder`) statt Watt angezeigt, analog zur Waschluder-Zeile.

## Geräte-Verbrauchskarte (Home-Dashboard)
Karte: `['views'][0]['sections'][3]['cards'][1]` — custom:button-card "⚡ Geräte Verbrauch" (Stand 30.06.2026)
Enthält Zeilen für: Klima SZ, Klima Büro, Waschluder (Status), Trockner (Status), Backofen (Status), Spülluder (Status), Fernseher (Status via media_player.75pml9009_12, NICHT switch.75pml9009_12_bildschirmstatus), Aquariumfilter, Skimmer, CO2, Markise, Bambula, Ladegeräte Garage, Gefrierschrank Keller, Gesamtbedarf.
Geräte ohne Watt-Sensor (AEG, Miele) zeigen State statt Watt, mit Farbwechsel grün/grau je nach aktiv.

## Wichtige Entity-IDs (Michael, Leipzig)
- PV Solarleistung: `sensor.solarbank_3_e2700_pro_solarleistung`
- PV Hausabgabe: `sensor.solarbank_3_e2700_pro_ac_hausabgabe`
- PV Akku SoC: `sensor.solarbank_3_e2700_pro_ladestand`
- PV Akku Energie: `sensor.solarbank_3_e2700_pro_akkuenergie`
- Hausbedarf: `sensor.system_shizophrenia_hausbedarf`
- Netzbezug: `sensor.smart_meter_netzbezug`
- Netzeinspeisung: `sensor.smart_meter_netzeinspeisung`
- Ertrag gesamt: `sensor.system_shizophrenia_ertrag_gesamt`
- Strompreis Tibber: `sensor.strompreis_tibber_simuliert_brutto`
- Wetter: `weather.forecast_home`
- FritzBox Verbindung: `binary_sensor.fritz_box_6690_cable_verbindung`
- FritzBox Download/Upload: `sensor.fritz_box_6690_cable_download_durchsatz` / `_upload_durchsatz`
- Klima SZ: `sensor.smart_switch_24110884111321510804c4e7ae103138_power`
- Klima Büro: `sensor.smart_switch_24110893663466510804c4e7ae10316d_power`
- Waschluder Status: `sensor.waschluder_state`
- Waschluder Restzeit: `sensor.waschluder_restzeit_hh_mm`
- Waschluder Programm: `sensor.waschluder_program`
- Herbert (Backofen) Status: `sensor.herbert_state`
- Herbert Steuerbuttons: `button.herbert_start/pause/resume/stop_reset`
- Herbert Power: `switch.herbert_power`
- Standgebläse (Trockner) Status: `sensor.standgeblaese_state`
- Standgebläse Restzeit: `sensor.standgeblase_restzeit_hh_mm`
- Standgebläse Steuerbuttons: `button.standgeblaese_start/pause/resume/stop_reset`
- Spülluder (Geschirrspüler) Status: `sensor.spulluder`
- Spülluder Restzeit: `sensor.spulluder_restzeit_hh_mm`
- Spülluder Programm: `sensor.spulluder_programm`
- Spülluder Programmphase: `sensor.spulluder_programmphase`
- Spülluder PowerDisk: `sensor.spulluder_powerdisk_fullstand`
- Spülluder Klarspüler: `sensor.spulluder_klarspuler_fullstand`
- Spülluder Power: `switch.spulluder_stromversorgung`
- Luftikusa Status: `sensor.luftikusa_state`
- Luftikusa Power: `switch.luftikusa_power`
- Aquariumfilter: `sensor.smart_switch_24110880220563510804c4e7ae102cd4_power`
- Fernseher (Wohnzimmer, Philips 75PML9009/12): `media_player.75pml9009_12` (zuverlässig) — NICHT `switch.75pml9009_12_bildschirmstatus` (kaputt, meldet nie "on", Reload bringt nichts)
- Fernseher Alexa-Schatten-Entity (unabhängig, seit Tagen unavailable, ungenutzt): `media_player.wohnzimmer_fernseher`
- Außen Temp/Feuchte: `sensor.gw2000a_wifi637b_aussen_temperatur` / `_aussen_luftfeuchte`
- Wohnzimmer: `sensor.thermometer_wohnzimmer_temperature` / `_humidity`
- Schlafzimmer: `sensor.thermometer_schlafzimmer_temperature`
- Büro: `sensor.thermometer_buero_temperature`
- Bad: `sensor.thermometer_bad_temperature` / `_humidity`
- Müll Nächste: `sensor.muellaebfuhr_naechste`
- Müll Gelb/Blau/Grau: `sensor.muell_gelbe_ton` / `muell_blaue_ton` / `muell_graue_ton`

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
  zeile1: >
    [[[ return `<div style='display:flex;justify-content:space-between;padding:2px 0'>
    <span style='color:#aaa'>Label</span>
    <span style='font-weight:bold;color:#fff'>${states['sensor.entity'].state} W</span>
    </div>` ]]]
custom_fields_style:
  zeile1:
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
Karte: `['views'][0]['sections'][2]['cards'][2]` — custom:button-card "⚡ Geräte Verbrauch"
Enthält Zeilen für: Klima SZ, Klima Büro, Waschluder (Status), Spülluder (Status), Aquariumfilter, Skimmer, CO2, Markise, Bambula, Gesamtbedarf.
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
- Außen Temp/Feuchte: `sensor.gw2000a_wifi637b_aussen_temperatur` / `_aussen_luftfeuchte`
- Wohnzimmer: `sensor.thermometer_wohnzimmer_temperature` / `_humidity`
- Schlafzimmer: `sensor.thermometer_schlafzimmer_temperature`
- Büro: `sensor.thermometer_buero_temperature`
- Bad: `sensor.thermometer_bad_temperature` / `_humidity`
- Müll Nächste: `sensor.muellaebfuhr_naechste`
- Müll Gelb/Blau/Grau: `sensor.muell_gelbe_ton` / `muell_blaue_ton` / `muell_graue_ton`

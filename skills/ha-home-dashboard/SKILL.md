# HA Home-Dashboard (Layout-Stil nach Uwe Kortkamp)

## Ziel
3-spaltiges Übersichts-Dashboard im dunklen Stil (#1a1a1a), inspiriert vom Kortkamp-Dashboard.

## URL-Pfad
`home-overview`

## Benötigte HACS-Karten
- `custom:button-card` (bereits installiert)
- `custom:apexcharts-card` (bereits installiert)
- `card-mod` (bereits installiert)
- `custom:hourly-weather` → HACS ID 499270202 (installiert)
- `html-template-card` → NICHT im HACS-Store, Ersatz: button-card mit custom_fields

## Spaltenstruktur
1. **Solar & Energie**: PV-Karte (button-card custom_fields), Apexcharts-Strompreis, Hourly-Weather
2. **Klima & Temperaturen**: Tile-Cards für alle Räume + Außen
3. **Netz & Smart Home**: FritzBox-Status, Solarbank-Übersicht, Geräteverbrauch, Müll

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

## Müllkalender (Waste Collection Schedule)
- HACS Integration ID: 254347436
- Leipzig-Anbieter: ICS-Quelle (Stadtreinigung Leipzig)
- URL-Muster: `https://stadtreinigung-leipzig.de/wir-kommen-zu-ihnen/abfallkalender/ical.ics?position_nos=XXXXX&name=STRASSENNAME&mode=download`
- ICS-URL via: stadtreinigung-leipzig.de → Abfallkalender → Adresse eingeben → Exportieren → Link kopieren
- YAML-Konfiguration (nach Neustart + ICS-URL beschaffen):
```yaml
waste_collection_schedule:
  sources:
    - name: ics
      args:
        regex: "(.*), .*"
        url: "ICS_URL_HIER"
      customize:
        - type: "Restmüll"
          alias: "Graue Tonne"
          icon: "mdi:trash-can"
        - type: "Papier"
          alias: "Blaue Tonne"
          icon: "mdi:newspaper"
        - type: "Bio"
          alias: "Grüne Tonne"
          icon: "mdi:leaf"
        - type: "Gelbe Tonne"
          alias: "Gelbe Tonne"
          icon: "mdi:recycle"
        - type: "Schadstoffe"
          alias: "Schadstoffmobil"
          icon: "mdi:biohazard"
```
- Nach Einrichtung: HA-Neustart → Sensoren `sensor.waste_collection_*` erscheinen
- Dashboard-Müll-Karte dann mit button-card + custom_fields aufbauen (analog zu html-template-card aus Original-Post)

## Hourly-Weather Konfiguration
```yaml
type: custom:hourly-weather
entity: weather.forecast_home
forecast_type: hourly
num_segments: "4"
name: null
icons: true
round_temperatures: true
show_precipitation_probability: true
hide_minutes: true
grid_options:
  columns: full
  rows: 2
label_spacing: "1"
colors:
  sunny: "#FFB300"
  clear-night: "#1a1a2e"
  partlycloudy: "#78AADC"
  cloudy: "#777777"
  rainy: "#44739d"
  pouring: "#2b4f7a"
  lightning-rainy: "#5c4a9e"
  lightning: "#5c4a9e"
  snowy: "#e0f0ff"
  snowy-rainy: "#8aafc7"
  fog: "#999999"
  windy: "#FFB300"
  windy-variant: "#e6a000"
  exceptional: "#ff9d00"
card_mod:
  style: |
    ha-card { display: block; background: #1a1a1a; border-radius: 12px; }
```

## Wichtige Entity-IDs (Michael)
- PV Solarleistung: `sensor.solarbank_3_e2700_pro_solarleistung`
- PV Hausabgabe: `sensor.solarbank_3_e2700_pro_ac_hausabgabe`
- PV Akku SoC: `sensor.solarbank_3_e2700_pro_ladestand`
- PV Akku Energie: `sensor.solarbank_3_e2700_pro_akkuenergie`
- Hausbedarf: `sensor.system_shizophrenia_hausbedarf`
- Netzbezug: `sensor.smart_meter_netzbezug`
- Netzeinspeisung: `sensor.smart_meter_netzeinspeisung`
- Ertrag gesamt: `sensor.system_shizophrenia_ertrag_gesamt`
- Strompreis Tibber simuliert: `sensor.strompreis_tibber_simuliert_brutto`
- Wetter: `weather.forecast_home`
- FritzBox Verbindung: `binary_sensor.fritz_box_6690_cable_verbindung`
- FritzBox Download: `sensor.fritz_box_6690_cable_download_durchsatz`
- FritzBox Upload: `sensor.fritz_box_6690_cable_upload_durchsatz`
- FritzBox Neustart: `sensor.fritz_box_6690_cable_letzter_neustart`
- Klima SZ Leistung: `sensor.smart_switch_24110884111321510804c4e7ae103138_power`
- Klima Büro Leistung: `sensor.smart_switch_24110893663466510804c4e7ae10316d_power`
- Waschluder Status: `sensor.waschluder_state`
- Aquariumfilter: `sensor.smart_switch_24110880220563510804c4e7ae102cd4_power`
- Außen Temp: `sensor.gw2000a_wifi637b_aussen_temperatur`
- Außen Feuchte: `sensor.gw2000a_wifi637b_aussen_luftfeuchte`
- Wohnzimmer Temp/Feuchte: `sensor.thermometer_wohnzimmer_temperature` / `_humidity`
- Schlafzimmer Temp: `sensor.thermometer_schlafzimmer_temperature`
- Büro Temp: `sensor.thermometer_buero_temperature`
- Bad Temp/Feuchte: `sensor.thermometer_bad_temperature` / `_humidity`

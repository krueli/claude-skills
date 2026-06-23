# ha-system-config

**Trigger:** Verwende diesen Skill immer, wenn Michael neue HA-Ideen, Automationen, Dashboards, Integrationen oder Helfer plant. Auch relevant bei Debugging, Namenskonventionen, Gerätebezug oder Energierechnung.

**Zweck:** Vollständige Referenz der aktuellen Home Assistant Konfiguration von Michael. Verhindert Rückfragen zu vorhandenen Geräten, Integrationen und Patterns.

**Stand:** 2026-06-23 (live abgefragt via MCP)

---

## Systeminfo

- **HA Version:** 2026.6.4
- **Hardware:** Raspberry Pi 5 (HAOS)
- **Standort:** Leipzig, DE (Lat: 51.338, Lon: 12.495)
- **Zeitzone:** Europe/Berlin
- **Sprache:** de
- **Bewohner:** Michael (person.michael), Nancy/GöGa (person.nancy)
- **Geräte gesamt:** 128 Config Entries (108 loaded, 20 not loaded/ignored/disabled)
- **Entities gesamt:** ~1400+ (696 Sensoren, 139 Switches, 112 Binary Sensors, 91 Device Tracker, 54 Buttons, 51 Lights, 46 Updates, 27 Automationen, 25 Input Numbers, ...)

---

## Integrationen (loaded – relevant)

### Energie & Solar
| Integration | Domain | Titel |
|---|---|---|
| Anker Solix | `anker_solix` | Pummelchen (HACS, v3.6.3) |
| Nord Pool | `nordpool` | Nord Pool DE-LU |
| Solcast PV Forecast | `solcast_solar` | Solcast Solar (HACS, v4.5.2) – 2 Rooftop Sites, 5 Calls/Tag, Hobbyist |
| Tibber Price Information | `tibber_prices` | HACS v0.31.0 |
| Riemann Sum Integral | `integration` | Netzbezug Energie (delta-basiert) |
| Utility Meter | `utility_meter` | Netzbezug Stunde (noch vorhanden, aber Hauptarchitektur ist delta-basiert via input_number) |
| forecast_solar | deaktiviert | Ersetzt durch Solcast |

### Bewässerung
| Gerät | Entity-Prefix | Ecowitt-ID |
|---|---|---|
| Zone 1 – Tomatenkübel | `bewasserung_zone_1` | 00004252 |
| Zone 2 – Gurkengewächshaus | `bewasserung_zone_2` | 000041FA |
| Zone 3 – Mulchbeet/Hecke | `bewasserung_zone_3` | 00004228 |

Bewässerungs-Entities (Beispiele):
- `switch.bewasserung_zone_X` – Ventilsteuerung (via MQTT/Ecowitt WFC01)
- `binary_sensor.bewasserung_zone_X_erreichbar` – Template-Sensor
- `input_datetime.bewasserung_zone_X_letzter_lauf`
- `input_select.bewasserung_zone_X_regensperre` – Werte: "Keine Sperre", "1 Tag", "2 Tage", "3 Tage", "7 Tage"
- `sensor.bewasserung_zone_X_letzter_lauf_anzeige` – as_local()-fixierter Template-Sensor
- `sensor.bewasserung_zone_X_regensperre_anzeige`
- `sensor.bewaesserung_zone_X_alarm`
- `sensor.zone_X_laufzeit_min`
- `sensor.bewasserung_gesamt_wasser_letzter_lauf`

Architektur-Patterns Bewässerung:
- Delta-basierte Wasserzählung über `input_number`-Helfer
- Regensperre via `input_datetime` + `input_select` + Automation
- Benachrichtigung 3 min nach Bewässerungsstart
- Ventilalarm: Nur Zone 3 + water_leak aller Zonen
- Sicherheits-Abschaltung täglich 07:01 und 21:01
- Temperatursperre unter 18°C (Morgenläufe)

### Smart Home Geräte

| Integration | Domain | Geräte |
|---|---|---|
| Meross LAN | `meross_lan` (HACS v5.8.0) | msh450 (Rauchmelder), mss305 Skimmer, mss305 CO2, mss315 Klima Schlafzimmer, mss315 Klima Büro, mss315 Bambula, mss315 Aquariumfilter, mss310 Markise |
| Philips Hue | `hue` | Hue Bridge 0017882b986f – Lights: Bett (Segmente 1-4), Stehlampe, Lampe Schlafzimmer, diverse WZ-Lichter |
| HomematicIP | `homematicip_cloud` | HmIP Alarmanlage (`alarm_control_panel.hmip_alarm_control_panel`) |
| Nuki | `nuki` | Nuki Schloss (`lock.nuki_2`), Bridge 299079FB |
| Ring | `ring` | Haustür-Kamera (`camera.haustur_live_ansicht`), Klingel (`event.haustur_klingeln`), Bewegung (`event.haustur_bewegung`) |
| Govee | `govee` (HACS v2026.6.15) | Cloud-Integration, Pending Update auf v2026.6.20 |
| AEG | `aeg` (HACS v0.1.1, emanuelbesliu/homeassistant-aeg) | Waschluder, Standgebläse – Entities: `switch.waschluder_power`, `switch.standgeblaese_power`, `button.standgeblaese_start/pause/resume/stop_reset`, `select.waschluder_program`, `sensor.waschluder_restzeit_hh_mm`, `sensor.waschluder_gesamtlaufzeit_hh_mm` |
| Miele | `miele` | Spülluder G7510 via OAuth-Cloud (`sensor.spulluder_restzeit_hh_mm`) |
| MyVaillant | `mypyllant` (HACS v0.9.16) | Heizung/Warmwasser (`climate.bad`, `water_heater.my_home_domestic_hot_water_0`, `climate.my_home_zone_1_circuit_0_climate`) |
| Ecowitt Official | `ha_ecowitt_iot` (HACS v1.0.38) | GW2000A-WIFI637B (GW2000X + WS90 Wetterstation) |
| Aqara Devices | `ha_aqara_devices` (HACS v1.2.2) | Aqara Hub-B8CE (HomeKit Controller deaktiviert, stattdessen HACS-Integration) |
| FRITZ!Box | `fritz` | FRITZ!Box 6690 Cable (aktiv), FRITZ!Powerline 1260 (aktiv), Repeater 3000 AX (ignored), WLAN Repeater 310 (ignored) |
| FritzBox Callmonitor | `fritzbox` | Callmonitor für FRITZ!Box 6690 Cable + fritz-powerlinesolix |
| Alexa | `alexa_devices` | Alexa Wohnzimmer (media_player) |
| Philips TV | `philips_js` | 75PML9009/12 (`media_player.75pml9009_12`) |
| BambuLab | `bambu_lab` (HACS v2.2.22) | P2S, Serial 22E8BJ5B1300756, LAN-Modus, IP 192.168.178.86. Entities: `camera.p2s_22e8bj5b1300756_kamera`, `sensor.p2s_verbleibende_zeit_formatiert`, `sensor.p2s_gesamtnutzung_korrigiert` |
| BMW / CarData | `homelable` (HACS v0.8.0 Homelable) | BMW X1 30e (read-only) |
| Garmin Connect | `garmin_connect` (HACS v3.0.12) | garmin@m-krueger.de – Fenix 8, Venu X1 |
| Garmin Mobile App | `mobile_app` | 2× Garmin Device Registrierungen |
| iCloud3 | `icloud3` (HACS v3.5.1) | Presence Detection |
| BTHome | `bthome` | 7C-C6-B6-71-8E-F2 (Bluetooth-Sensor) |
| Tuya | `tuya` | einkaufen@m-krueger.de |
| Xiaomi Miot | `xiaomi_miot` (HACS v1.1.4) | installiert |

### MCP / AI Infrastruktur
- `mcp_server` – HA Assist (Webhook Proxy)
- `mcp_proxy` – MCP Webhook Proxy
- `ha_mcp_tools` – HA MCP Tools v7.8.0 (HACS)
- `extended_openai_conversation` (HACS v2.0.2) – OpenAI Conversation Agent

### Weitere geladene Integrationen
- `cloud` – Home Assistant Cloud (Nabu Casa, TTS, STT)
- `mqtt` – Mosquitto MQTT Broker (Add-on)
- `nordpool` – Nord Pool, Zone DE/LU
- `brother` – MFC-J5345DW Drucker
- `ipp` – Brother Drucker (IPP)
- `bluetooth` – brcm bcm43438-bt (aktiv), bcm43438-bt (deaktiviert)
- `ibeacon` – iBeacon Tracker
- `matter` – Matter Server
- `met` – Wetter (Yr.no)
- `go2rtc` – Kamera-Streaming
- `radio_browser` – Radio Browser
- `shopping_list` – Einkaufsliste
- `google_translate` – Google TTS
- `rpi_power` – Raspberry Pi Power Checker
- `raspberry_pi` – Raspberry Pi System
- `analytics` – HA Analytics
- `backup` – Backup Manager
- `sun` – Sonnenstand
- `thread` – Thread (Netzwerk)
- `hassio` – HA Supervisor

### Deaktiviert / Ignoriert (relevant)
- `caldav` (iCloud) – deaktiviert, Event-Loop-Blockierung
- `forecast_solar` (2×) – deaktiviert, ersetzt durch Solcast
- `miele` (2× ignored) – nur eine Miele-Instanz active (OAuth-Cloud)
- `dlna_dmr` (2×) – deaktiviert
- `homekit_controller` (Aqara Hub) – deaktiviert, durch HACS-Integration ersetzt
- `nest` – ignoriert
- `bluetooth` (DC:A6:32) – deaktiviert

---

## HACS-Pakete (vollständig, 26 installiert)

### Integrationen (13)
| Paket | Version | Zweck |
|---|---|---|
| AEG (emanuelbesliu) | v0.1.1 | AEG/Electrolux Geräte |
| Anker Solix | 3.6.3 | Solarbank |
| Aqara Devices | v1.2.2 | Aqara Hub |
| Bambu Lab | v2.2.22 | BambuLab P2S |
| Ecowitt Official | v1.0.38 | GW2000A/WS90 |
| extended_openai_conversation | 2.0.2 | AI Conversation |
| Garmin Connect | 3.0.12 | Garmin Daten |
| Govee Cloud | v2026.6.15 | Govee Lights (⚠️ Update verfügbar v2026.6.20) |
| HA MCP Tools | v7.8.0 | MCP Server |
| HACS | 2.0.5 | HACS selbst |
| Homelable | v0.8.0 | BMW CarData |
| iCloud3 v3 | v3.5.1 | iDevice Tracking |
| Meross LAN | v5.8.0 | Meross Plugs/Strips |
| MyVaillant | v0.9.16 | Heizungssteuerung |
| Nuki Web | 1.0.3 | Nuki API |
| Solcast PV | v4.5.2 | PV-Prognose |
| Tibber Price | v0.31.0 | Tibber Preise |
| Xiaomi Miot | v1.1.4 | Xiaomi Geräte |

### Lovelace Cards (8)
| Paket | Version |
|---|---|
| apexcharts-card | v2.2.3 |
| button-card | v7.0.1 |
| card-mod | v4.2.1 |
| Daylight Calendar Card | v4.1.0 |
| k-flow-card | V.1.1.2 |
| layout-card | v2.4.7 |
| Power Flow Card Plus | v0.3.7 |
| Zigbee2mqtt Networkmap Card | v0.13.0 |

---

## Dashboards (11 Storage-Mode)

| URL-Pfad | Titel | Icon |
|---|---|---|
| `dashboard-info` | Info | – |
| `energie-solar` | Energie & Solar | – |
| `dashboard-buro` | Büro | – |
| `klima-heizung` | Klima & Heizung | – |
| `dashboard-sicherheit` | Sicherheit | – |
| `dashboard-ubersicht` | Übersicht | – |
| `bambula-3d` | Bambula | mdi:printer-3d |
| `dashboard-wetter` | Wetter | mdi:weather-partly-cloudy |
| `dashboard-bewaesserung` | Bewässerung | mdi:sprinkler |
| `aeg-geraete` | Hausgeräte | mdi:washing-machine |
| `haus-uebersicht` | Haus | mdi:home-floor-2 |

---

## Raum-/Bereich-Struktur (Areas)

| Area ID | Name | Entity-Anzahl |
|---|---|---|
| `wohnzimmer` | Wohnzimmer | 153 |
| `buro` | Büro | 222 |
| `kuche` | Küche | 48 |
| `schlafzimmer` | Schlafzimmer | 23 |
| `flur` | Flur | 68 |
| `flur_oben` | Flur Oben | 35 |
| `garten` | Garten | 52 |
| `keller` | Keller | 51 |
| `bad` | Bad | 6 |
| `haustur` | Haustür | 10 |
| `strasse` | Straße | 1 |

---

## Helfer-Übersicht (input_number – Auswahl)

| Entity | Zweck | Wert |
|---|---|---|
| `input_number.nordpool_morgen_min` | Spotpreis-Zwischenspeicher | 0.0573 |
| `input_number.nordpool_morgen_max` | Spotpreis-Zwischenspeicher | 0.615 |
| `input_number.nordpool_morgen_durchschnitt` | Spotpreis-Zwischenspeicher | 0.1749 |
| `input_number.vw_grundpreis_eur_monat` | VW-Grundpreis | 12.57 |
| `input_number.tibber_grundpreis_eur_monat` | Tibber-Grundpreis | 14.54 |

Input Select (Bewässerung):
- `input_select.bewasserung_zone_1_regensperre`
- `input_select.bewasserung_zone_2_regensperre`
- `input_select.bewasserung_zone_3_regensperre`
- Optionen: "Keine Sperre", "1 Tag", "2 Tage", "3 Tage", "7 Tage"

Input Boolean:
- `input_boolean.akku_entladen` – Anker Solix Steuerung
- `input_boolean.fernseher_schalter`

---

## Energie-Architektur (Schlüsselkonzepte)

**Netzbezug:**
- Hauptsensor: delta-basierter Riemann-Sum (`sensor.netzbezug_energie_…`)
- Referenzwerte in `input_number`-Helfern (nicht utility_meter!)
- `sensor.netzbezug_heute_delta`, `sensor.netzbezug_monat_delta`

**Preisrechnung:**
- Tibber-Simulation (kein echter Tibber-Vertrag): `(Spotpreis + 0.1349 €/kWh netto) × 1.19`
- `sensor.strompreis_tibber_simuliert_brutto`
- `sensor.strompreis_dynamisch_vs_fest` – Echtzeit-Vergleich
- Vollkosten inkl. Grundpreis: `sensor.vollkosten_monat_festtarif`, `sensor.vollkosten_monat_tibber`
- Tageskosten LTS: `sensor.tageskosten_festtarif_lts`, `sensor.tageskosten_tibber_lts`
- Netzkosten Heute/Monat/Stunde je Fest und Tibber

**Solar:**
- Solcast: 2 Sites, Hobbyist, 5 Calls/Tag
- Anker Solix: Solarbank, Scan-Intervall 30s, aktuell Smart-Modus

**Wechselentscheidung Tibber:** Offen, geplant 01.01.2027

---

## Zigbee-Setup

- **Coordinator:** MG24 (Silicon Labs, EmberZNet 8.0.2, ember-Stack)
- **Config-Pfad:** `/config/zigbee2mqtt_mg24`
- **Broker:** Mosquitto MQTT (lokaler Add-on)
- **Migration:** CC2652P → MG24 abgeschlossen
- Aqara-Thermometer: noch nicht angelernt (Stand 2026-06)

---

## Geräte-Details (Key Devices)

**BambuLab P2S:**
- Serial: 22E8BJ5B1300756
- Modus: LAN-Mode (kein Cloud-Slicer nötig)
- IP: 192.168.178.86
- HACS-Integration v2.2.22 (greghesp)
- Bambuddy auf Pi4 (192.168.178.112, User pummelchen, Docker)
- Filament: Jayo PETG, Sunlu PLA Matte (+ Generic PETG HF)
- Aktuelle Filamente: `sensor.p2s_*_filament_type`

**Garmin:**
- Geräte: Fenix 8, Venu X1, Edge 830/840
- Wahoo KICKR (Trainer)
- Integration: garmin_connect + Mobile App (2 Registrierungen)
- `sensor.garmin_glance_info` (Template-Sensor)

**BMW X1 30e:**
- Integration: Homelable (CarData Stream API, HACS)
- Nur read-only

**Nuki:**
- Lock: `lock.nuki_2`
- Scripts: `script.nuki_offnen_2`, `script.nuki_auf`
- Watchdog-Automation bei unavailable > 5 min
- Alarm-Integration mit HmIP (Zustandsbedingung: armed_home/armed_away/triggered)
- Garmin-Steuerung über: `switch.garmin_nuki_schalter`, `switch.garmin_nuki_falle`

**Meross Rauchmelder (msh450):**
- Büro + Schlafzimmer
- Brandalarm-Automation: Critical Alerts + alle Lichter + TTS
- Bambula Smart-Plug-Automation bei Rauchalarm

**Ecowitt GW2000A + WS90:**
- HACS Integration ha_ecowitt_iot v1.0.38
- Wetterstation im Garten-Bereich (45 Sensoren)
- `sensor.windrichtung_himmelsrichtung` (Template-Sensor)

---

## Personen & Notify

| Person | Entity | Notify |
|---|---|---|
| Michael | `person.michael` | `notify.pummelchen` |
| Nancy (GöGa) | `person.nancy` | `notify.goga` |
| iPad WZ | – | `notify.ipad_wz` |

---

## Bekannte Einschränkungen / abgelehnte Ansätze

- **CalDAV (iCloud):** Deaktiviert wegen Event-Loop-Blockierung – nicht wieder aktivieren
- **utility_meter:** Für Hauptenergie-Messung abgelöst durch delta-basiertes Modell
- **eWeLink:** Aufgegeben (Ecowitt WFC01 als Ersatz für Sonoff SWV-ZFE)
- **forecast_solar:** Deaktiviert, Solcast ist Ersatz
- **Homekit Controller (Aqara):** Deaktiviert, ha_aqara_devices HACS stattdessen
- **Bluetooth (DC:A6:32):** Deaktiviert
- **Shell Command Bridge** (`write_file_b64`/`read_file_raw`): Vorhanden für Datei-Zugriff

---

## Infrastruktur

- **HA-Host:** Raspberry Pi 5, HAOS
- **Pi4 (Bambuddy):** 192.168.178.112, User pummelchen, Docker (Bambuddy + skill-writer MCP)
- **Tailscale:** Subnet-Router auf Pi4 (100.125.201.7, Subnet 192.168.178.0/24)
- **Zweitwohnsitz-HA:** 192.168.188.103 (zweiter Pi, Cloudflared als Option geplant)
- **Externe URL:** Nabu Casa (Remote UI aktiv: `binary_sensor.remote_ui` = on)
- **SQLite DB:** ~1140 MB (purge_keep_days: 30, ~40 Entity-Globs)

---

## Update-Hinweis (Stand 2026-06-23)

- Govee Cloud: Update verfügbar (v2026.6.15 → v2026.6.20)
- Alle anderen HACS-Pakete: aktuell
- HA Core/OS/Supervisor: aktuell (2026.6.4)

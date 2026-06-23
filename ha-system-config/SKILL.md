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
| Anker Solix | `anker_solix` | Pummelchen (HACS, v3.6.3) – Solarbank 3 E2700 Pro |
| Nord Pool | `nordpool` | Nord Pool DE-LU |
| Solcast PV Forecast | `solcast_solar` | Solcast Solar (HACS, v4.5.2) – 2 Rooftop Sites, 5 Calls/Tag, Hobbyist |
| Tibber Price Information | `tibber_prices` | HACS v0.31.0 |
| Riemann Sum Integral | `integration` | Netzbezug Energie (delta-basiert) |
| Utility Meter | `utility_meter` | Netzbezug Stunde (noch vorhanden, Hauptarchitektur ist aber delta-basiert via input_number) |
| forecast_solar | deaktiviert | Ersetzt durch Solcast |

### Bewässerung (Ecowitt WFC01 via MQTT)

**Ventil-Entity-IDs (direkt steuerbar):**
| Zone | Zweck | Switch | Binary Online | Binary Running |
|---|---|---|---|---|
| Zone 1 – Tomatenkübel | `switch.wfc01_00004252` | `binary_sensor.wfc01_00004252_ist_online` | `binary_sensor.wfc01_00004252_running` |
| Zone 2 – Gurkengewächshaus | `switch.wfc01_000041fa` | `binary_sensor.wfc01_000041fa_ist_online` | `binary_sensor.wfc01_000041fa_running` |
| Zone 3 – Mulchbeet/Hecke | `switch.wfc01_00004228` | `binary_sensor.wfc01_00004228_ist_online` | `binary_sensor.wfc01_00004228_running` |

**Helfer-Entities pro Zone (X = 1, 2, 3):**
- `input_datetime.bewasserung_zone_X_letzter_lauf` – Timestamp letzter Lauf
- `input_datetime.bewasserung_zone_X_regensperre_bis` – Sperr-Ablaufzeit
- `input_select.bewasserung_zone_X_regensperre` – Optionen: "Keine Sperre" / "1 Tag" / "2 Tage" / "3 Tage" / "7 Tage"
- `input_number.bewasserung_zone_X_dauer_morgens` – Laufzeit Morgen (min)
- `input_number.bewasserung_zone_X_dauer_abends` – Laufzeit Abend (min), Zone 3: `_dauer_taglich`
- `input_number.bewasserung_zone_X_dauer_heute` – Tageszähler (min)
- `input_number.bewasserung_zone_X_dauer_gesamt` – Gesamtlaufzeit (min)
- `input_number.bewasserung_zone_X_letzter_lauf_liter` – Wasser letzter Lauf (L)
- `input_number.bewasserung_zone_X_wasser_gesamt` – Gesamtwasser (L)
- `binary_sensor.bewasserung_zone_X_erreichbar` – Template-Erreichbarkeit
- `sensor.bewasserung_zone_X_letzter_lauf_anzeige` – as_local()-fixierter Template-Sensor
- `sensor.bewasserung_zone_X_regensperre_anzeige`
- `sensor.bewaesserung_zone_X_alarm`
- `sensor.zone_X_laufzeit_min`

**Weitere Bewässerungs-Sensoren:**
- `sensor.bewasserung_gesamt_wasser_letzter_lauf`

**Architektur-Patterns Bewässerung:**
- Delta-basierte Wasserzählung über `input_number`-Helfer
- Regensperre via `input_datetime.bewasserung_zone_X_regensperre_bis` + `input_select` + Automation
- Benachrichtigung 3 min nach Bewässerungsstart (`automation.bewasserung_start_benachrichtigung_3_min`)
- Ventilalarm: Nur Zone 3 + water_leak aller Zonen (`automation.bewasserung_ventilalarm`)
- Sicherheits-Abschaltung täglich 07:01 und 21:01 (`automation.bewasserung_sicherheits_abschaltung`)
- Temperatursperre unter 18°C (Morgenläufe)
- Tageszähler-Reset: `automation.bewasserung_tageszahler_zurucksetzen`
- Dauern-Erfassung: `automation.bewasserung_dauer_erfassen`
- Regensperre setzen: `automation.bewasserung_regensperre_setzen`
- Regensperre zurücksetzen: `automation.bewasserung_regensperre_abgelaufen_selector_zurucksetzen`

### Smart Home Geräte

| Integration | Domain | Details |
|---|---|---|
| Meross LAN | `meross_lan` (HACS v5.8.0) | msh450 (Rauchmelder), mss305 Skimmer, mss305 CO2, mss315 Klima Schlafzimmer, mss315 Klima Büro, mss315 Bambula, mss315 Aquariumfilter, mss310 Markise |
| Philips Hue | `hue` | Hue Bridge 0017882b986f |
| HomematicIP | `homematicip_cloud` | HmIP Alarmanlage |
| Nuki | `nuki` | Bridge 299079FB |
| Ring | `ring` | Haustür-Kamera + Klingel |
| Govee | `govee` (HACS v2026.6.15) | ⚠️ Update v2026.6.20 ausstehend |
| AEG | `aeg` (HACS v0.1.1) | Waschmaschine + Standgebläse |
| Miele | `miele` | Spülmaschine G7510 via OAuth-Cloud |
| MyVaillant | `mypyllant` (HACS v0.9.16) | Heizung + Warmwasser |
| Ecowitt Official | `ha_ecowitt_iot` (HACS v1.0.38) | GW2000A + WS90 |
| Aqara Devices | `ha_aqara_devices` (HACS v1.2.2) | Aqara Hub-B8CE |
| FRITZ!Box | `fritz` + `fritzbox` | 6690 Cable (aktiv), Powerline 1260 (aktiv) |
| Alexa | `alexa_devices` | Alexa Wohnzimmer |
| Philips TV | `philips_js` | 75PML9009/12 |
| BambuLab | `bambu_lab` (HACS v2.2.22) | P2S LAN-Modus |
| BMW | `homelable` (HACS v0.8.0) | BMW X1 30e, read-only |
| Garmin | `garmin_connect` (HACS v3.0.12) | Fenix 8, Venu X1 |
| iCloud3 | `icloud3` (HACS v3.5.1) | Presence Detection |

### MCP / AI Infrastruktur
- `mcp_server` – HA Assist MCP Server
- `mcp_proxy` – MCP Webhook Proxy
- `ha_mcp_tools` v7.8.0 – HA MCP Tools (HACS)
- `extended_openai_conversation` v2.0.2 – OpenAI Conversation Agent (HACS)

### Deaktiviert / abgelehnt (nie reaktivieren ohne Grund)
- `caldav` (iCloud) – Event-Loop-Blockierung
- `forecast_solar` (2×) – ersetzt durch Solcast
- `homekit_controller` (Aqara Hub) – ersetzt durch ha_aqara_devices
- `dlna_dmr` (2×) – nicht genutzt
- `bluetooth` DC:A6:32 – deaktiviert
- **utility_meter** für Energie-Hauptmessung – abgelöst durch delta-Modell
- **eWeLink** – aufgegeben, Ecowitt WFC01 als Ersatz

---

## Automationen (27 gesamt)

| Entity ID | Zweck | Status |
|---|---|---|
| `automation.akku_wird_leer` | Akku-Warnung | off (deaktiviert) |
| `automation.bambula_aus_bei_rauchalarm_buro` | Drucker aus bei Rauch | on |
| `automation.bambula_push_benachrichtigungen` | Druckende-Notification mit Kamerabild | on |
| `automation.bewasserung_dauer_erfassen` | Laufzeit-Tracking | on |
| `automation.bewasserung_regensperre_abgelaufen_selector_zurucksetzen` | Regensperre-Reset | on |
| `automation.bewasserung_regensperre_setzen` | Regensperre aktivieren | on |
| `automation.bewasserung_sicherheits_abschaltung` | Sicherheits-Cutoff 07:01 + 21:01 | on |
| `automation.bewasserung_start_benachrichtigung_3_min` | Push nach 3 min | on |
| `automation.bewasserung_tageszahler_zurucksetzen` | Tageszähler Reset | on |
| `automation.bewasserung_ventilalarm` | Ventilalarm Zone 3 + water_leak | on |
| `automation.bewasserung_zone_1_tomaten` | Zone 1 Zeitplan | on |
| `automation.bewasserung_zone_2_gurken` | Zone 2 Zeitplan | on |
| `automation.bewasserung_zone_3_mulchbeet_hecke` | Zone 3 Zeitplan | on |
| `automation.brandalarm_rauchmelder_ausgelost` | Critical Alert + Lichter + TTS | on |
| `automation.fernseher_schalter_steuerung` | TV-Steuerung | on |
| `automation.haustur_kamera_snapshot_alle_60_sekunden` | Ring-Kamera Snapshot | on |
| `automation.haustur_offen` | Haustür offen Warnung | on |
| `automation.miele_spulluder_programm_beendet` | Miele Fertig-Notification | on |
| `automation.netzbezug_reset_erkennen` | Zähler-Reset Erkennung | on |
| `automation.netzbezug_start_erfassen` | Tages/Monats-Startwert erfassen | on |
| `automation.nordpool_morgen_preise_aktualisieren` | Nord Pool Morgen-Preise cachen | on |
| `automation.nuki_alarm_unscharf` | Nuki bei Alarm-Scharf | on |
| `automation.pv_tagesprognose_um_10_uhr` | Solcast-Prognose 10 Uhr | on |
| `automation.rauchmelder_batterie_schwach` | Batterie-Warnung Rauchmelder | on |
| `automation.rauchmelder_geratefehler` | Gerätefehler Rauchmelder | on |
| `automation.solcast_forecast_update` | Solcast Update-Trigger | on |
| `automation.watchdog_nuki_unavailable` | Nuki Watchdog > 5 min unavailable | on |

---

## Scripts (2)

- `script.nuki_offnen_2` – Nuki öffnen
- `script.nuki_auf` – Nuki auf

---

## Scenes (24 – alle Hue)

**Wohnzimmer:**
`scene.wohnzimmer_aktivieren`, `scene.wohnzimmer_entspannen`, `scene.wohnzimmer_fruhlingsbluten`, `scene.wohnzimmer_gedimmt`, `scene.wohnzimmer_hell`, `scene.wohnzimmer_konzentration`, `scene.wohnzimmer_lesen`, `scene.wohnzimmer_nachtlicht`, `scene.wohnzimmer_naturliches_licht`, `scene.wohnzimmer_nordlichter`, `scene.wohnzimmer_ruhephase`, `scene.wohnzimmer_sonnenuntergang_savanne`, `scene.wohnzimmer_tropendammerung`

**Deckenlampe (WZ):**
`scene.deckenlampe_energie_tanken`, `scene.deckenlampe_entspannen`, `scene.deckenlampe_fruhlingsbluten`, `scene.deckenlampe_gedimmt`, `scene.deckenlampe_hell`, `scene.deckenlampe_konzentrieren`, `scene.deckenlampe_lesen`, `scene.deckenlampe_nachtlicht`, `scene.deckenlampe_nordlichter`, `scene.deckenlampe_sonnenuntergang_savanne`, `scene.deckenlampe_tropendammerung`

---

## Lichter (51 gesamt)

**Hue – Schlafzimmer:**
- `light.lampe_schlafzimmer` (unavailable)
- `light.bett` + `light.bett_segment_1` bis `_11` (Govee Strip)

**Hue – Wohnzimmer:**
- `light.deckenlampe` + `light.deckenlampe_1` bis `_5`
- `light.stehlampe` + `light.stehlampe_segment_1` bis `_15`
- `light.tischlampe`
- `light.wohnzimmer` (Gruppe)
- `light.stern`

**Sonstiges:**
- `light.75pml9009_12_ambilight` (Philips TV Ambilight, unavailable)
- `light.p2s_22e8bj5b1300756_druckraumbeleuchtung` (BambuLab Kammerlicht)
- `light.schalter_mit_signalleuchte`
- `light.taste_oben`, `light.taste_unten`
- `light.smart_hub_*_dnd`, `light.smart_plug_*_dnd`, `light.smart_switch_*_dnd` (Meross DND-Lichter, nicht für Lichtsteuerung gedacht)

---

## Alle Helfer (vollständig)

### input_number (25)

**Bewässerung:**
| Entity | Aktuell |
|---|---|
| `input_number.bewasserung_zone_1_dauer_morgens` | 10 min |
| `input_number.bewasserung_zone_1_dauer_abends` | 45 min |
| `input_number.bewasserung_zone_1_dauer_heute` | 0 min |
| `input_number.bewasserung_zone_1_dauer_gesamt` | 607 min |
| `input_number.bewasserung_zone_1_letzter_lauf_liter` | 0.008 L |
| `input_number.bewasserung_zone_1_wasser_gesamt` | 0.07 L |
| `input_number.bewasserung_zone_2_dauer_morgens` | 10 min |
| `input_number.bewasserung_zone_2_dauer_abends` | 45 min |
| `input_number.bewasserung_zone_2_dauer_heute` | 10 min |
| `input_number.bewasserung_zone_2_dauer_gesamt` | 524.5 min |
| `input_number.bewasserung_zone_2_letzter_lauf_liter` | 0 L |
| `input_number.bewasserung_zone_2_wasser_gesamt` | 0.135 L |
| `input_number.bewasserung_zone_3_dauer_taglich` | 35 min |
| `input_number.bewasserung_zone_3_dauer_heute` | 0 min |
| `input_number.bewasserung_zone_3_dauer_gesamt` | 313 min |
| `input_number.bewasserung_zone_3_letzter_lauf_liter` | 17.09 L |
| `input_number.bewasserung_zone_3_wasser_gesamt` | 66.751 L |

**Energie:**
| Entity | Zweck | Aktuell |
|---|---|---|
| `input_number.netzbezug_start_heute` | Delta-Referenz Tag | 397.13537 |
| `input_number.netzbezug_start_monat` | Delta-Referenz Monat | 247.96487 |
| `input_number.grundpreis_tibber` | Tibber-Grundpreis €/Mon | 14.54 |
| `input_number.grundpreis_vw_festtarif` | VW-Grundpreis €/Mon | 12.57 |
| `input_number.nordpool_morgen_min` | Nord Pool Morgen Min | 0.0573 |
| `input_number.nordpool_morgen_max` | Nord Pool Morgen Max | 0.615 |
| `input_number.nordpool_morgen_durchschnitt` | Nord Pool Morgen Avg | 0.1749 |
| `input_number.nordpool_morgen_teuerstes_slot` | Teuerstes Slot Uhrzeit | 20.75 |

### input_datetime (6)
- `input_datetime.bewasserung_zone_1_letzter_lauf`
- `input_datetime.bewasserung_zone_1_regensperre_bis`
- `input_datetime.bewasserung_zone_2_letzter_lauf`
- `input_datetime.bewasserung_zone_2_regensperre_bis`
- `input_datetime.bewasserung_zone_3_letzter_lauf`
- `input_datetime.bewasserung_zone_3_regensperre_bis`

### input_select (3)
- `input_select.bewasserung_zone_1_regensperre` – "Keine Sperre" / "1 Tag" / "2 Tage" / "3 Tage" / "7 Tage"
- `input_select.bewasserung_zone_2_regensperre`
- `input_select.bewasserung_zone_3_regensperre`

### input_boolean (2)
- `input_boolean.akku_entladen` – Anker Solix Steuerlogik (aktuell: on)
- `input_boolean.fernseher_schalter` – TV-Steuerung (aktuell: off)

---

## Energie-Architektur (Detail)

**Netzbezug delta-basiert:**
- Riemann-Sum-Integral über Leistungssensor → kumulativer Zähler
- `input_number.netzbezug_start_heute` / `netzbezug_start_monat` speichern Startwert
- `sensor.netzbezug_heute_delta` = aktueller Zähler − Startwert Heute
- `sensor.netzbezug_monat_delta` = aktueller Zähler − Startwert Monat
- Reset-Automation erkennt Zählersprünge und korrigiert Referenzwert

**Preisrechnung (Template-Sensoren):**
- Tibber-Simulation: `(Spotpreis_netto + 0.1349) × 1.19` → Brutto €/kWh
- `sensor.strompreis_tibber_simuliert_brutto`
- `sensor.strompreis_gunstigster_slot_brutto`, `sensor.strompreis_teuerster_slot_brutto`
- `sensor.nordpool_morgen_min/max/durchschnitt_brutto`
- Netzkosten: Heute / Monat / Stunde je Fest + Tibber (6 Sensoren)
- Vollkosten inkl. Grundpreis: `sensor.vollkosten_monat_festtarif`, `sensor.vollkosten_monat_tibber`
- LTS-Tageskosten: `sensor.tageskosten_festtarif_lts`, `sensor.tageskosten_tibber_lts`
- Vergleich: `sensor.strompreis_dynamisch_vs_fest`

**Anker Solix (Solarbank 3 E2700 Pro):**
- `sensor.solarbank_3_e2700_pro_solarleistung` – Gesamt PV (aktuell 271W)
- `sensor.solarbank_3_e2700_pro_solar_pv1/pv2/pv3/pv4` – je String
- `sensor.solarbank_3_e2700_pro_ac_hausabgabe` – Einspeisung (213W)
- `sensor.solarbank_3_e2700_pro_ladestand` – SOC (10%)
- `sensor.solarbank_3_e2700_pro_akkuleistung` / `aufladeleistung` / `entladeleistung`
- `sensor.solarbank_3_e2700_pro_betriebszustand` – charge_bypass / smart / etc.
- `sensor.solarbank_3_e2700_pro_einspeisevorgabe` – Soll-Einspeisung (200W)
- `sensor.solarbank_3_e2700_pro_erweiterungsakkus` – Anzahl Akkus (2)

**Solcast:**
- `automation.pv_tagesprognose_um_10_uhr` – täglicher Update-Trigger
- `automation.solcast_forecast_update` – zusätzlicher Trigger

**Tibber-Entscheidung:** Wechsel geplant 01.01.2027, noch nicht entschieden

---

## Geräte-Details (Key Devices)

**BambuLab P2S:**
- Serial: 22E8BJ5B1300756 | IP: 192.168.178.86 | LAN-Modus
- `camera.p2s_22e8bj5b1300756_kamera` | `light.p2s_22e8bj5b1300756_druckraumbeleuchtung`
- `sensor.p2s_verbleibende_zeit_formatiert` | `sensor.p2s_gesamtnutzung_korrigiert`
- `image.p2s_22e8bj5b1300756_titelbild` | `image.p2s_22e8bj5b1300756_bild_wahlen`
- Bambuddy auf Pi4 (192.168.178.112, User pummelchen, Docker) – MCP-Endpoint
- Filament aktuell: Bambu PLA Matte (AMS Slot 1), Generic PETG HF (AMS Slot 2), Generic PETG (×2)
- `automation.bambula_push_benachrichtigungen` – Druckende + Kamerabild
- `automation.bambula_aus_bei_rauchalarm_buro` – Smart-Plug Bambula

**Garmin:**
- Geräte: Fenix 8, Venu X1, Edge 830/840, Wahoo KICKR
- Integration: garmin_connect + 2× Mobile App Registrierung
- `sensor.garmin_glance_info` (Template)
- `switch.garmin_nuki_schalter`, `switch.garmin_nuki_falle` – Nuki via Garmin

**Nuki:**
- `lock.nuki_2` | Bridge 299079FB
- `script.nuki_offnen_2`, `script.nuki_auf`
- `automation.watchdog_nuki_unavailable` – Watchdog > 5 min
- `automation.nuki_alarm_unscharf` – HmIP-Zustandsbedingung: armed_home / armed_away / triggered

**Alarmanlage:**
- `alarm_control_panel.hmip_alarm_control_panel` (HomematicIP)

**Meross Rauchmelder (msh450):**
- Büro + Schlafzimmer
- `automation.brandalarm_rauchmelder_ausgelost` – Critical Alerts + alle Lichter + TTS
- `automation.rauchmelder_batterie_schwach`, `automation.rauchmelder_geratefehler`

**Meross Smart Plugs / Switches (entity IDs via Internetzugang-Switch-Namensschema):**
- `switch.nuki_bridge_299079fb_internetzugang`
- `switch.meross_smart_plug_internetzugang` bis `_7` (8 Plugs)
- `switch.meross_smart_hub_internetzugang`

**Ring Haustür:**
- `camera.haustur_live_ansicht`
- `event.haustur_klingeln`, `event.haustur_bewegung`
- `switch.haustur_bewegungserkennung` (on)
- `number.haustur_lautstarke`
- `automation.haustur_kamera_snapshot_alle_60_sekunden`

**Ecowitt GW2000A + WS90:**
- 45 Sensoren im Garten-Bereich
- `sensor.windrichtung_himmelsrichtung` (Template)

**AEG Geräte:**
- `switch.waschluder_power` | `select.waschluder_program` (Cotton Pr Eco40-60)
- `sensor.waschluder_restzeit_hh_mm` | `sensor.waschluder_gesamtlaufzeit_hh_mm`
- `switch.standgeblaese_power` | `button.standgeblaese_start/pause/resume/stop_reset`
- `switch.herbert_power` | `switch.luftikusa_power`

**Miele Spülmaschine G7510:**
- `sensor.spulluder_restzeit_hh_mm` | `automation.miele_spulluder_programm_beendet`

**Heizung (MyVaillant):**
- `climate.bad` (auto) | `climate.my_home_zone_1_circuit_0_climate` (unavailable)
- `water_heater.my_home_domestic_hot_water_0`

**Philips TV:**
- `media_player.75pml9009_12` | `remote.75pml9009_12_fernbedienung`
- `light.75pml9009_12_ambilight` (unavailable)

**BMW X1 30e:**
- Integration: Homelable (CarData Stream API, HACS v0.8.0), read-only

---

## Zigbee-Setup

- **Coordinator:** MG24 (Silicon Labs, EmberZNet 8.0.2, ember-Stack)
- **Config-Pfad:** `/config/zigbee2mqtt_mg24`
- **Broker:** Mosquitto MQTT (Add-on)
- **Migration:** CC2652P → MG24 abgeschlossen
- **Aqara-Thermometer:** noch nicht angelernt (Stand 2026-06)

---

## Personen & Notify

| Person | Entity | Notify |
|---|---|---|
| Michael | `person.michael` | `notify.pummelchen` |
| Nancy / GöGa | `person.nancy` | `notify.goga` |
| iPad WZ | – | `notify.ipad_wz` |

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

| Area ID | Name | Entities |
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

## HACS-Pakete (vollständig, 26)

### Integrationen (18)
| Paket | Version | Zweck |
|---|---|---|
| AEG (emanuelbesliu) | v0.1.1 | AEG/Electrolux Geräte |
| Anker Solix | 3.6.3 | Solarbank |
| Aqara Devices | v1.2.2 | Aqara Hub |
| Bambu Lab | v2.2.22 | BambuLab P2S |
| Ecowitt Official | v1.0.38 | GW2000A/WS90 |
| extended_openai_conversation | 2.0.2 | AI Conversation |
| Garmin Connect | 3.0.12 | Garmin Daten |
| Govee Cloud | v2026.6.15 | Govee Lights (**⚠️ Update v2026.6.20 verfügbar**) |
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

## Infrastruktur

- **HA-Host:** Raspberry Pi 5, HAOS
- **Pi4 (Bambuddy):** 192.168.178.112, User pummelchen, Docker (Bambuddy + skill-writer MCP auf Port 8001)
- **skill-writer Endpoint:** https://bambuddy.tail5738ee.ts.net/mcp (Tailscale Funnel)
- **Tailscale:** Subnet-Router auf Pi4 (100.125.201.7, Subnet 192.168.178.0/24)
- **Zweitwohnsitz-HA:** 192.168.188.103 (zweiter Pi, Cloudflared als Option geplant)
- **Externe URL:** Nabu Casa (Remote UI aktiv)
- **SQLite DB:** ~1140 MB (purge_keep_days: 30, ~40 Entity-Globs)
- **Shell Command Bridge:** `write_file_b64` / `read_file_raw` für Datei-Zugriff vorhanden
- **Claude Code Add-on:** ESJavadex-Fork mit CLAUDE.md konfiguriert

---

## Bekannte Einschränkungen / abgelehnte Ansätze (nie reaktivieren ohne expliziten Grund)

| Ansatz | Grund |
|---|---|
| CalDAV (iCloud) | Event-Loop-Blockierung in HA |
| utility_meter für Energie-Hauptmessung | Zu fehleranfällig, ersetzt durch delta-Modell |
| eWeLink / Sonoff SWV-ZFE | Aufgegeben, Ecowitt WFC01 ist Ersatz |
| forecast_solar | Ersetzt durch Solcast |
| HomeKit Controller (Aqara) | Ersetzt durch ha_aqara_devices HACS |
| Bluetooth DC:A6:32 | Deaktiviert |

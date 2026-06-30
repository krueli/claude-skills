# Energieverbrauch-Diagnose & Smart-Plug-Kaufentscheidung (Michael, Leipzig)

## Methodik: Mystery-Verbraucher über Netzbezug-Historie identifizieren
1. `ha_get_history(entity_ids="sensor.smart_meter_netzbezug", significant_changes_only=False, start_time="<Nd>")` über mehrere Tage ziehen, NICHT significant_changes_only=True (sonst werden kurze Spitzen weggefiltert).
2. Spitzen-Zeitfenster identifizieren und Dauer/Amplitude clustern (z.B. "kurz + sehr hoch" vs. "länger + mittel").
3. Gegen alle bekannten Geräte-Zustandssensoren zur exakten Spitzenzeit abgleichen (AEG-States, Spülluder-State, Smart-Switch-Power, Klima-Outlet-Status).
4. Live-Differenz-Check als Ergänzung: `ha_eval_template` mit `sensor.system_shizophrenia_hausbedarf` minus Summe aller bekannten `_power`-Sensoren ergibt die aktuelle "dunkle" Grundlast (Stand 30.06.2026: ca. 422W unzugeordnet bei 559W Gesamt — Kühlschrank Küche, Netzwerktechnik, HA-Server, diverse Standby-Geräte, nichts davon smart-steckdosenüberwacht).
5. Bei Verdacht auf ein bestimmtes Gerät (z.B. Spülmaschine): komplette State-Historie des Geräts ziehen und jede Run-Periode gegen die Netzbezug-Spitzen legen — bei 100% Überlappung über mehrere Tage ist die Korrelation belastbar, auch ohne direkten Watt-Sensor am Gerät.

## Identifizierte Verbraucher (Stand 30.06.2026)
- **BMW X1 30e Ladung**: 7000-8800W, ca. 100-110 Min. sustained. Rechnerisch passt das fast exakt zur Akkukapazität (~14,2 kWh) des PHEV. Kein HA-Sensor für die Wallbox vorhanden, daher Korrelation nur über Plausibilität (Dauer × Leistung ≈ Akkukapazität) bzw. Abgleich mit BMW-ConnectedDrive-App-Ladehistorie.
- **Spülluder (Miele G7510)**: 2000-3800W, schwankend je nach Programmphase (Heizen/Spülen/Trocknen). KEIN Echtzeit-Wattsensor in der Miele-Integration (strukturelle Einschränkung, kein API-Endpoint dafür) — nur über `sensor.spulluder` State ("in_use" etc.) korrelierbar, nicht direkt messbar. Über 4 Tage lückenlos verifiziert: jede Netzbezug-Spitze in diesem Bereich fiel in ein Spülluder-Laufzeitfenster.
- **Kochfeld (Herd)**: Vermuteter Verursacher für isolierte 1000-1200W-Episoden, 16-21 Uhr (Kochzeit), 15-20 Min. Dauer. NICHT verifizierbar — Kochfelder sind i.d.R. festverdrahtet (kein Schuko-Zwischenstecker möglich), daher strukturell unmessbar über Smart Plugs. Einzige Verifikationsmöglichkeit: Energiezähler direkt im Sicherungskasten am Kochfeld-Stromkreis (Shelly 3EM o.ä., Elektriker-Projekt, kein Steckdosen-Kauf).
- **Ausgeschlossen für das 1000-1200W-Muster**: beide Klimaanlagen (vom User selbst per Live-Beobachtung ausgeschlossen), Küchengeräte/Waschen/Trocknen (laut User zu den Zeitpunkten aus).
- **Bekannte blinde Flecken**: `sensor.standgeblaese_state` (Trockner) und `sensor.luftikusa_state` (Luftreiniger) liefern in Stichproben oft "unavailable"/"unknown" statt eines echten Status — Vorsicht beim Schlussfolgern "Gerät war aus" rein aus dem fehlenden State heraus, ohne dass der User das selbst am Gerät verifiziert hat.

## KRITISCH: Recorder-Exclude blockierte komplette Geräteklasse (behoben 30.06.2026)
In `/config/configuration.yaml` excludete `recorder.exclude.entity_globs` ursprünglich **den kompletten Glob `sensor.smart_switch_*`** — das betraf nicht nur Diagnosewerte, sondern auch alle `_power`-Sensoren (Klima Schlafzimmer/Büro, Aquariumfilter, Skimmer, CO2, Bambula). Dadurch gab es für diese Sensoren **keine Historie**, obwohl `ha_get_state` live korrekte Werte lieferte — Live-Abfrage funktioniert unabhängig vom Recorder, Verlauf/Diagnose aber nicht.
Fix: Glob verengt auf nur die wirklich uninteressanten Subwerte:
```yaml
- sensor.smart_switch_*_voltage
- sensor.smart_switch_*_current
```
(Spannung/Stromstärke weiterhin excluded, `_power` und `_consumption` jetzt aufgezeichnet.) `sensor.smart_plug_*` bleibt weiterhin komplett excluded (nicht angefasst, war nicht Teil der Anfrage).
**Workflow für Recorder-Config-Änderungen**: `shell_command.read_file_raw` → lokal in Datei kopieren (bash_tool) → gezielt ändern → **Datei nochmal mit `view` gegenlesen** (Tippfehler beim Abtippen sind real passiert, z.B. Klammer-Verwechslung in Jinja-Template-Strings) → `ha_get_system_health(include="config_check")` vor dem Schreiben/Neustart prüfen → `shell_command.write_file_b64` (Auto-Backup) → **Recorder-Filter-Änderungen brauchen einen vollen `ha_restart`**, ein einfaches `ha_reload_core` reicht NICHT (Recorder-Filter werden nur beim Start instanziiert). Nach Restart: MCP-Verbindung bricht kurz ab (normal), nach ca. 60s erneut verbinden und mit `ha_get_state` auf einem der vorher leeren Sensoren verifizieren, dass Live-Werte wieder kommen.

## Smart-Plug-Kaufentscheidung: Sonoff S60ZBTPF vs. Shelly Plug S Gen3
Kontext: Michael baut Zigbee-Mesh aktiv aus (MG24-Migration), will weitere Geräte ausstatten, Fokus auch auf Keller/Garage (mesh-schwache Zonen).

| Kriterium | Sonoff S60ZBTPF | Shelly Plug S Gen3 |
|---|---|---|
| Protokoll | Zigbee (Z2M) | WLAN (lokal, push-basiert, kein Cloud-Zwang) |
| Maße | 50×50×61,5mm | 40×40×36mm (kleinster am Markt) |
| Preis (Amazon, Stand 30.06.2026) | 2er-Pack 27,99€ (~14€/Stk) | 2er-Pack 19,99€ (~10€/Stk) |
| Max. Last | 16A / 3680-4000W | 12A / 2500W |
| Zigbee-Router-Funktion | Ja — verdichtet Mesh | Nein |
| Messgenauigkeit bei niedriger Last | Reporting-Schwellenwert-Problem bekannt (anfangs 0W bei Kleinlasten, lösbar über `minimum_reportable_change` in Z2M) | Bis 11% Abweichung bei niedriger Last, <1% über 300W |

**Empfehlung**: Räume mit schwachem Zigbee-Signal (Keller, Garage, weit vom Koordinator) → Sonoff, wegen Mesh-Verdichtung als Zigbee-Router. Räume in Koordinator-Nähe, wo Mesh-Bonus keinen Unterschied macht → Shelly, wegen Preis und Größe. Gemischter Einkauf je nach Standort ist die wirtschaftlich und technisch sinnvollste Lösung, kein Pauschalurteil "X ist immer besser".

## Philips-TV-Bug (siehe auch ha-home-dashboard-Skill)
`switch.75pml9009_12_bildschirmstatus` meldet strukturell nie "on" (JointSpace-API-Limitierung der Firmware). Reload der `philips_js`-Integration (Config-Entry `01KM62F22SKA5CNZPWCT55XY6J`) wurde getestet und behebt es NICHT — sofort nach Reload wieder derselbe Fehler. Nicht erneut versuchen zu fixen. Nutze stattdessen `media_player.75pml9009_12` für TV-Status.

## Fernseher hat strukturell KEINEN Watt-Sensor
Die `philips_js`-Integration liefert keinen Power-Sensor (nur media_player, remote, ambilight-light, 2 Aufnahme-Sensoren, der kaputte Bildschirmstatus-Switch). Verbrauchskarte zeigt für TV daher nur Status (An/Aus), nie Watt — das ist kein Bug, sondern eine technische Grenze der Integration. Für echte Wattmessung wäre eine Zwischensteckdose nötig (siehe Sonoff/Shelly-Vergleich oben).

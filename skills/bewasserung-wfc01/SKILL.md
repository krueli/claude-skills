# bewasserung-wfc01

**Trigger:** Verwende diesen Skill bei allem was die Gartenbewässerung mit Ecowitt WFC01-Ventilen betrifft: Automationen, Dashboard, Helfer, manuelle Steuerung, Letzter-Lauf-Tracking.

**Stand:** 2026-06-24

---

## Hardware / Ventile

| Zone | Entity (switch) | binary_sensor running | Gesamt-Sensor |
|---|---|---|---|
| Zone 1 – Tomatenkübel | `switch.wfc01_00004252` | `binary_sensor.wfc01_00004252_running` | `sensor.wfc01_00004252_gesamt` |
| Zone 2 – Gurkengewächshaus | `switch.wfc01_000041fa` | `binary_sensor.wfc01_000041fa_running` | `sensor.wfc01_000041fa_gesamt` |
| Zone 3 – Mulchbeet/Hecke | `switch.wfc01_00004228` | `binary_sensor.wfc01_00004228_running` | `sensor.wfc01_00004228_gesamt` |

GW2000 Gateway: IP `192.168.178.118`  
Ecowitt Controller Add-on: `c71bcef1_ecowitt-controller` v2.0.4

---

## Automationen

| Automation | Entity | Zweck |
|---|---|---|
| Bewässerung Zone 1 (Tomaten) | `automation.bewasserung_zone_1_tomaten` | 06:00 / 18:00, 3 Versuche, mit Regensperre |
| Bewässerung Zone 2 (Gurken) | `automation.bewasserung_zone_2_gurken` | 06:30 / 19:00, 3 Versuche, mit Regensperre |
| Bewässerung Zone 3 (Mulchbeet) | `automation.bewasserung_zone_3_mulchbeet_hecke` | 20:00 täglich, mit Regen- und Temperaturlogik |
| Bewässerung Dauer erfassen | `automation.bewasserung_dauer_erfassen` | Triggert auf `running off` jeder Zone → setzt Letzter-Lauf + addiert Dauer |
| Bewässerung Zone 1 – Manuell | `automation.bewasserung_zone_1_manuell` | Event-Trigger, öffnet Ventil für einstellbare Dauer |
| Bewässerung Zone 2 – Manuell | `automation.bewasserung_zone_2_manuell` | Event-Trigger, öffnet Ventil für einstellbare Dauer |
| Bewässerung Zone 3 – Manuell | `automation.bewasserung_zone_3_manuell` | Event-Trigger, öffnet Ventil für einstellbare Dauer |

### Manuell-Automationen
- Trigger: `event.fire` mit `event_type: bewasserung_manuell_zoneX_start`
- Regensperre wird ignoriert
- Dauer aus `input_number.bewasserung_zoneX_manuell_dauer` (1–120 min)

---

## Helfer (wichtigste)

| Helfer | Zweck |
|---|---|
| `input_datetime.bewasserung_zone_X_letzter_lauf` | Zeitpunkt letzter Lauf (wird in `dauer_erfassen` gesetzt, nicht in Zonen-Automationen!) |
| `input_number.bewasserung_zone_X_manuell_dauer` | Manuelle Dauer in Minuten (1–120, Standard 10) |
| `input_number.bewasserung_zone_X_dauer_morgens/abends` | Geplante Laufzeiten |
| `input_number.bewasserung_zone_X_letzter_lauf_liter` | Wasserverbrauch letzter Lauf (delta-basiert) |
| `input_number.bewasserung_zone_X_wasser_gesamt` | Kumulierter Gesamtverbrauch |
| `input_select.bewasserung_zone_X_regensperre` | Regensperre auslösen (via Selector) |
| `input_datetime.bewasserung_zone_X_regensperre_bis` | Regensperre aktiv bis |

---

## Wichtiges Design-Prinzip: Letzter Lauf

**`letzter_lauf` wird NICHT in den Zonen-Automationen gesetzt**, sondern ausschließlich in `automation.bewasserung_dauer_erfassen` beim `running off`-Event. Das stellt sicher, dass auch manuelle Läufe (Ecowitt App, Dashboard-Button, direkte Switch-Steuerung) korrekt erfasst werden.

Der Zeitstempel ist `trigger.from_state.last_changed` (= Startzeitpunkt des Laufs), gesetzt via:
```yaml
action: input_datetime.set_datetime
data:
  timestamp: "{{ trigger.from_state.last_changed | as_timestamp }}"
```

---

## Dashboard

URL-Path: `dashboard-bewaesserung`  
Pro Zone im Dashboard:
1. Heading (Zone-Name)
2. Ventil-Toggle (tile)
3. **Manuelle Dauer** (tile → more-info zum Einstellen)
4. **Manuell starten** (button → `event.fire`)
5. Fließt, Laufzeit, Fluss, Letzter Lauf (L), Wassertemp., Heute Dauer, Gesamtdauer, Letzter Lauf (Datum), Batterie, Wasser gesamt

---

## Bekannte Fallstricke

- `input_datetime.set_datetime` akzeptiert kein Template in `data.datetime` → immer `data.timestamp` mit `| as_timestamp` verwenden
- `wait_template` statt `wait_for_trigger` führt zu Race Conditions wenn Ventil schon läuft → `wait_for_trigger` verwenden
- `as_local()` nötig bei Zeitvergleichen mit `now()` in Regensperre-Bedingungen
- Manuelle Ecowitt-App-Läufe werden seit 2026-06-24 korrekt in `letzter_lauf` erfasst (war vorher nicht der Fall)

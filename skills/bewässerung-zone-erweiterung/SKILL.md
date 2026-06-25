# Skill: Bewässerungszone – Morgen/Abend-Aufteilung

## Kontext
Dieses Pattern beschreibt, wie man eine bestehende HA-Bewässerungszone von einem einzigen Tageszeitpunkt (z.B. täglich 20:00) auf zwei separate Läufe (morgens + abends) erweitert – ohne die bestehende Automation zu löschen.

## Anwendungsfall
Zone hatte nur einen `dauer_taglich`-Helfer und eine einzige Automation. Ziel: morgens und abends separat steuerbare Dauern + eigene Automationen.

## Schritte

### 1. Neue input_number-Helfer anlegen
Zwei neue Helfer per `ha_config_set_helper(helper_type='input_number')`:
- `bewasserung_zone_X_morgens` (min 0, max 120, step 5, mode box, unit min)
- `bewasserung_zone_X_abends` (min 0, max 120, step 5, mode box, unit min)

Initialwert analog zum bisherigen Tageszeitgeber setzen.

### 2. Bestehende Automation per python_transform anpassen
- `dauer_taglich` in allen `delay`-Schritten durch `dauer_abends` ersetzen
- Alias und Description aktualisieren
- Fehlermeldungen in notify-Actions anpassen ("Abendlauf fehlgeschlagen")

Wichtig: config_hash zuerst per `ha_config_get_automation` holen, dann transform.

### 3. Neue Morgen-Automation anlegen
Identische Struktur zur Abend-Automation (3 Versuche, wait_template, if/then, letzter_lauf setzen, Wassermengen tracken), aber:
- Trigger: `06:00:00` (oder gewünschte Zeit)
- Delay referenziert `dauer_morgens`
- Fehlermeldungen: "Morgenlauf fehlgeschlagen"
- Gleiche Regensperre/Regenmengen-Bedingungen wie Abend-Automation

### 4. Dashboard anpassen
In der Bewässerungszeiten-Karte den alten `dauer_taglich`-Eintrag durch zwei neue Einträge ersetzen:
```python
# python_transform auf dashboard-bewaesserung
for section in config['views'][0]['sections']:
    for card in section['cards']:
        if card.get('type') == 'entities' and card.get('title') == '⏱️ Bewässerungszeiten':
            new_entities = []
            for entity in card['entities']:
                if entity.get('entity') == 'input_number.bewasserung_zone_X_dauer_taglich':
                    new_entities.append({'entity': 'input_number.bewasserung_zone_X_morgens', 'name': '🌹 Zone X – morgens (06:00)'})
                    new_entities.append({'entity': 'input_number.bewasserung_zone_X_abends', 'name': '🌹 Zone X – abends (20:00)'})
                else:
                    new_entities.append(entity)
            card['entities'] = new_entities
```

## Wichtige Entscheidungen
- Den alten `dauer_taglich`-Helfer NICHT löschen solange unklar ist, ob andere Automationen (z.B. Watchdog) ihn noch referenzieren
- Watchdog-Automation separat prüfen: verwendet er `dauer_taglich`? Falls ja, muss er auf beide neuen Helfer angepasst werden oder die max. Laufzeit beider Slots abdecken
- Trigger-Zeiten an Zone 1/2 anpassen (Konsistenz)

## Ecowitt WFC01 – Zone 3 (Mulchbeet/Hecke)
- Entity: `switch.wfc01_00004228`
- Helfer morgens: `input_number.bewasserung_zone_3_morgens`
- Helfer abends: `input_number.bewasserung_zone_3_abends`
- Automationen: `automation.bewasserung_zone_3_mulchbeet_hecke_morgen` (06:00), `automation.bewasserung_zone_3_mulchbeet_hecke` (20:00 – Abend)

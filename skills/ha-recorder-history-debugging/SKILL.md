# HA: Flache/konstante Linie in History-Charts debuggen (apexcharts-card, history-graph)

## Symptom
Eine Karte mit echter History-Serie (z.B. `custom:apexcharts-card` mit `series: [{entity: ...}]`)
zeigt eine perfekt flache, horizontale Linie über den ganzen Zeitraum, während andere Serien
in derselben Karte normal schwanken. Tile-Karten mit demselben Sensor zeigen einen korrekten,
aktuellen Wert.

## Root Cause Pattern
Das ist (fast) nie ein Dashboard-Konfigurationsfehler, sondern fast immer:
**Die Entity wird live aktualisiert, landet aber nicht im Recorder.**

Live-State korrekt + History komplett leer/eingefroren = Recorder-Exclude-Treffer, kein
Anzeigeproblem.

## Diagnose-Workflow
1. `ha_get_dashboard(url_path=..., heading="...")` um die Karten-Config zu finden, exakte
   Entity-IDs der betroffenen Serie(n) notieren.
2. `ha_get_history(entity_ids=<entity>, start_time="24h", significant_changes_only=false)`
   — wenn `total_count` 0 oder 1 ist, obwohl der Live-State sich erkennbar ändert
   (`ha_get_state` zeigt aktuellen `last_changed`), ist das der Beweis: Recorder schreibt nicht.
3. Recorder-Excludes prüfen: `configuration.yaml` auf dem HA-Host lesen (SSH oder
   shell_command-Bridge), nach `recorder:` → `exclude:` → `entity_globs` suchen.
   `grep -n -A 30 "^recorder:" /config/configuration.yaml`
4. Jeden Glob gegen die betroffene Entity-ID prüfen. Häufiger Bug: ein zu breiter Catch-all-Glob
   (z.B. `sensor.gw2000a_*`) steht UNTER spezifischeren Globs (z.B. `sensor.gw2000a_*_rssi`,
   `sensor.gw2000a_*_signal`) und macht diese überflüssig, weil er ALLES vom selben Gerät matcht,
   nicht nur Signal/RSSI/Diagnose-Sensoren wie offensichtlich beabsichtigt.

## Fix
1. Vollständigen aktuellen Inhalt von `configuration.yaml` lesen (SSH read, oder
   `shell_command.read_file_raw` falls vorhanden).
2. Lokal (im bash_tool-Sandbox) die korrigierte Version bauen — NICHT versuchen, direkt per SSH
   auf `/config/` zu schreiben, wenn der SSH-User nicht root ist (`/config/configuration.yaml`
   gehört root:root, mode 644 → `Permission denied` für normale User).
3. YAML-Struktur lokal validieren BEVOR geschrieben wird:
   ```python
   import yaml
   def ignore_unknown(loader, suffix, node):
       if isinstance(node, yaml.ScalarNode): return loader.construct_scalar(node)
       if isinstance(node, yaml.SequenceNode): return loader.construct_sequence(node)
       return loader.construct_mapping(node)
   yaml.SafeLoader.add_multi_constructor('!', ignore_unknown)  # HA-Tags wie !include ignorieren
   yaml.safe_load(open(path))
   ```
4. Datei base64-kodieren und über die vorhandene `shell_command.write_file_b64`-Bridge schreiben
   (läuft als root in HA, legt automatisch Backup unter `/config/.claude_backups/` an):
   `ha_call_service(domain="shell_command", service="write_file_b64", data={"path": "/config/configuration.yaml", "content_b64": "..."})`
5. **Wichtig:** Recorder-Excludes werden nur beim Start eingelesen. Ein reiner
   `homeassistant.reload_core_config` reicht NICHT. Es braucht
   `ha_call_service(domain="homeassistant", service="restart", wait=False)`.
   Die MCP-Verbindung bricht dabei erwartungsgemäß kurz ab ("MCP server connection lost") —
   das ist normal, kein Fehler.
6. Nach dem Restart ca. 60-90s warten (HA OS Boot + Integration-Setup), dann mit
   `ha_get_state` prüfen ob die Entity wieder lebt, und mit `ha_get_history(start_time="30m",
   order="desc")` verifizieren, dass neue Einträge ankommen.

## Lessons Learned
- `sensor.gw2000a_*_rssi` und `sensor.gw2000a_*_signal` waren schon vorhanden — die spätere
  Zeile `sensor.gw2000a_*` war vermutlich ein "sicher ist sicher"-Zusatz, der versehentlich
  alles erschlagen hat. Bei Glob-Listen immer prüfen, ob ein genereller Eintrag spezifischere
  Einträge redundant macht (Hinweis auf Copy-Paste-Fehler) — und ob er zu breit ist.
- Entities mit abweichendem Namensschema werden vom Glob nicht erfasst und bleiben unauffällig
  korrekt (hier: `sensor.garten_gw2000a_wifi637b_apparent_temperatur`, Präfix `garten_`, nicht
  `gw2000a_*`) — das erklärt, warum nur EINE von zwei Serien in der Karte betroffen war und
  half beim Eingrenzen.
- SSH-User auf den Pis (pummelchen) hat keine Schreibrechte auf `/config/*.yaml` (root:root).
  Für Schreibzugriffe auf `configuration.yaml` immer die `shell_command.write_file_b64`-Bridge
  nutzen, nicht versuchen direkt per SSH zu schreiben.

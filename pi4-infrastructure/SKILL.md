# pi4-infrastructure

**Trigger:** Verwende diesen Skill bei allem was den Raspberry Pi 4 (Bambuddy/Pi4) betrifft: Docker-Container, skill-writer, Bambuddy, Tailscale, Dateitransfer, Remote-Befehle vom HA aus.

**Stand:** 2026-06-23 (live abgefragt via MCP + shell_command)

---

## Hardware

- **Gerät:** Raspberry Pi 4
- **Hostname:** pummelchen (User: pummelchen)
- **IP lokal:** 192.168.178.112
- **Tailscale IP:** 100.125.201.7
- **Tailscale Subnet-Router:** 192.168.178.0/24 (ganzes Heimnetz via Tailscale erreichbar)

---

## Docker-Container

### skill-writer-mcp (aktiv)
- **Image:** python:3.12-slim (selbst gebaut via Dockerfile)
- **Basis:** `fastmcp` + `httpx`, Einstiegspunkt `server.py`
- **Projektverzeichnis:** ~/skill-writer-mcp/ (mit Dockerfile, server.py, docker-compose.yml)
- **Interner Port:** 8001
- **Expose:** Tailscale Funnel → `https://bambuddy.tail5738ee.ts.net/mcp` (SSE) und `/sse`
- **Funktion:** Liest/schreibt/löscht SKILL.md-Dateien in GitHub-Repo `krueli/claude-skills` via GitHub API
- **Docker-Netzwerk:** `skill-writer-mcp_default`
- **Container-Name:** `skill-writer-mcp`

### Bambuddy (separat)
- **Funktion:** BambuLab P2S Monitoring (Kamera-Proxy, Druckstatus-Bridge)
- **Zugriff:** Intern über 192.168.178.112

---

## HA Shell Command Bridge (Zugriff auf Pi4 von HA aus)

Registrierte shell_commands in HA (`configuration.yaml`), die SSH auf Pi4 ausführen:

| Service | Funktion |
|---|---|
| `shell_command.pi4_docker_logs` | Letzte Logs des skill-writer-mcp Containers |
| `shell_command.pi4_docker_restart` | skill-writer-mcp Container neu starten |
| `shell_command.pi4_update_server` | Container stoppen, Image neu bauen (docker compose up --build), starten |
| `shell_command.pi4_sync_disk` | Disk sync auf Pi4 (gibt "SYNCED" zurück) |
| `shell_command.pi4_scp_server` | SCP-Dateitransfer zum Pi4 |
| `shell_command.write_file_b64` | Base64-kodierte Datei auf HA-Host schreiben |
| `shell_command.read_file_raw` | Datei vom HA-Host lesen (liest Pi5/HA-Host, nicht Pi4!) |

**Wichtig:** `read_file_raw` und `write_file_b64` lesen/schreiben auf dem **Pi5 (HA-Host)**, nicht auf dem Pi4.

Aufruf via MCP:
```
ha_call_service(domain="shell_command", service="pi4_docker_logs", return_response=True)
```

---

## Tailscale Setup

- **Subnet-Router:** Pi4 ist Tailscale Subnet-Router für 192.168.178.0/24
- **Funnel aktiv:** `https://bambuddy.tail5738ee.ts.net` ist öffentlich erreichbar (skill-writer MCP-Endpoint)
- **Tailscale-Knoten:** Pi4 unter 100.125.201.7

---

## Datei-Workflow Pi4

Wenn Dateien auf den Pi4 übertragen werden sollen:
1. `shell_command.write_file_b64` – Datei base64-kodiert auf Pi5 schreiben
2. `shell_command.pi4_scp_server` – von Pi5 auf Pi4 kopieren
3. Bei skill-writer-Updates: `shell_command.pi4_update_server` – rebuild + restart

---

## Zweitwohnsitz-HA

- **IP:** 192.168.188.103 (zweiter Raspberry Pi)
- **Erreichbarkeit:** Lokal oder via Cloudflared (geplant, noch nicht umgesetzt)
- **Kein MCP-Zugriff** von Haupt-HA aus (Stand 2026-06)

---

## Was fehlt / nicht abfragbar

- Kein generischer SSH-Befehl verfügbar (nur die spezifischen `pi4_*` shell_commands)
- `docker ps` auf Pi4 nicht direkt ausführbar via MCP
- Pi4-Systemstatus (CPU, RAM, Temp) nicht direkt abrufbar
- Bambuddy-Konfigurationsdetails nicht direkt abfragbar

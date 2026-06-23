# pi4-infrastructure

**Trigger:** Verwende diesen Skill bei allem was den Raspberry Pi 4 (Bambuddy/Pi4) betrifft: Docker-Container, skill-writer, ssh-mcp, Bambuddy, Tailscale, Dateitransfer, Remote-Befehle vom HA aus.

**Stand:** 2026-06-23 (live abgefragt + deployed)

---

## Hardware

- **Gerät:** Raspberry Pi 4
- **Hostname:** pummelchen (User: pummelchen)
- **IP lokal:** 192.168.178.112
- **Tailscale IP:** 100.125.201.7
- **Tailscale Subnet-Router:** 192.168.178.0/24

---

## Docker-Container (1 Container, 2 Services)

### skill-writer-mcp (aktiv, kombinierter Server)

Einzelner Container mit zwei FastMCP-Services die parallel laufen:

| Service | Port | Intern | Extern |
|---|---|---|---|
| skill-writer | 8001 | `/sse` | `https://bambuddy.tail5738ee.ts.net/sse` |
| ssh-mcp | 8002 | `/sse` | lokal only (kein Funnel aktiv) |

- **Image:** python:3.12-slim (selbst gebaut)
- **Abhängigkeiten:** fastmcp, httpx
- **Container-Name:** `skill-writer-mcp`
- **Projektverzeichnis Pi4:** `~/skill-writer-mcp/`
- **Einstiegspunkt:** `server.py` (kombinierter Server)
- **Start:** `threading.Thread` für skill-writer (8001), Hauptprozess ssh-mcp (8002)

### Bambuddy (separater Prozess/Container)
- Monitoring für BambuLab P2S
- Details nicht direkt abfragbar

---

## Tailscale Funnel

Aktuell aktiv:
```
https://bambuddy.tail5738ee.ts.net (Funnel on)
|-- / proxy http://127.0.0.1:8001   (skill-writer)
```

**Einschränkung:** Tailscale Funnel unterstützt nur einen Root-Proxy pro Node. ssh-mcp (Port 8002) hat keinen öffentlichen Funnel. Lösungen für später:
- `tailscale serve` mit Pfad-Routing (statt Funnel)
- Nginx/Caddy als Reverse Proxy vor beiden Services

---

## MCP-Endpoints

| Service | URL für Claude.ai |
|---|---|
| skill-writer | `https://bambuddy.tail5738ee.ts.net/sse` |
| ssh-mcp | noch nicht öffentlich erreichbar |

---

## SSH-MCP Tools (intern verfügbar)

Wenn ssh-mcp in Claude.ai eingetragen ist, stehen folgende Tools bereit:

- `ssh_exec(command, host="pi4", user="pummelchen")` – beliebiger Shell-Befehl
- `ssh_docker_ps(host="pi4")` – docker ps
- `ssh_docker_logs(container, lines=50, host="pi4")` – Container-Logs
- `ssh_write_file(path, content, host="pi4")` – Datei schreiben (base64)
- `ssh_read_file(path, host="pi4")` – Datei lesen

Bekannte Hosts: `pi4` (192.168.178.112), `localhost`, `pi5` (192.168.178.1)

---

## HA Shell Command Bridge (Zugriff auf Pi4 von HA aus)

Definiert in `/config/packages/ssh_pi4.yaml`:

| Service | Funktion |
|---|---|
| `shell_command.pi4_docker_logs` | Logs skill-writer-mcp Container (letzten 50 Zeilen) |
| `shell_command.pi4_docker_restart` | `docker compose restart` in ~/skill-writer-mcp |
| `shell_command.pi4_update_server` | `docker compose down && up -d --build` |
| `shell_command.pi4_sync_disk` | `docker cp skill-writer-mcp:/app/server.py ~/skill-writer-mcp/server.py` |
| `shell_command.pi4_scp_server` | SCP von `/tmp/server_clean.py` → `~/skill-writer-mcp/server.py` |
| `shell_command.run_fix` | `python3 /tmp/fix_claude.py` – generischer SSH-Executor via Python |

**Deploy-Workflow:**
1. Datei via `write_file_b64` nach `/tmp/server_clean.py` schreiben
2. `pi4_scp_server` → kopiert nach Pi4
3. `pi4_update_server` → rebuild + restart

**Generischer SSH-Befehl:**
```python
# fix_claude.py auf Pi5 schreiben, dann run_fix aufrufen
# SSH-Key: /config/.ssh/pi4_key
# Pattern:
import subprocess
r = subprocess.run(
    ["ssh", "-i", "/config/.ssh/pi4_key", "-o", "StrictHostKeyChecking=no",
     "-o", "BatchMode=yes", "pummelchen@192.168.178.112", "BEFEHL"],
    capture_output=True, text=True, timeout=30
)
print(r.stdout + r.stderr)
```

---

## Wichtige Dateipfade Pi4

| Pfad | Inhalt |
|---|---|
| `~/skill-writer-mcp/server.py` | Aktueller kombinierter MCP-Server |
| `~/skill-writer-mcp/docker-compose.yml` | Compose-Konfiguration |
| `~/skill-writer-mcp/Dockerfile` | python:3.12-slim, fastmcp + httpx |

---

## Zweitwohnsitz-HA

- **IP:** 192.168.188.103 (zweiter Raspberry Pi)
- **Cloudflared:** geplant, noch nicht umgesetzt

---

## Bekannte Einschränkungen

- `read_file_raw` / `write_file_b64` lesen/schreiben auf Pi5 (HA-Host), nicht Pi4
- Tailscale Funnel: nur ein Root-Proxy möglich → ssh-mcp noch nicht extern erreichbar
- Kein generischer SSH-Command als HA shell_command – Workaround via `run_fix` + `/tmp/fix_claude.py`

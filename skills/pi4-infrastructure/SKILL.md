# pi4-infrastructure

**Trigger:** Verwende diesen Skill bei allem was den Raspberry Pi 4 (bambuddy/Pi4) oder Pi5 (HAOS) per SSH betrifft: Docker-Container, skill-writer, ssh-mcp, Bambuddy, Tailscale, Dateitransfer, Remote-Befehle.

**Stand:** 2026-06-24

---

## Hardware

- **Gerät:** Raspberry Pi 4
- **Hostname:** bambuddy (User: pummelchen)
- **IP lokal:** 192.168.178.112
- **Tailscale IP:** 100.125.201.7
- **Tailscale Subnet-Router:** 192.168.178.0/24

---

## SSH-Zugang Pi5 (HAOS)

Der Pi5 läuft HAOS – kein nativer SSH-Daemon auf Port 22. SSH läuft über das **Advanced SSH & Web Terminal Addon** (`a0d7b954_ssh`).

- **IP:** 192.168.178.163
- **Port:** 22 (host_network=true, deshalb direkt auf Pi5-IP)
- **User:** pummelchen
- **Auth:** Ed25519-Key von Pi4 (`pummelchen@bambuddy`) in `authorized_keys` eingetragen
- **Hostname im Container:** `a0d7b954-ssh`

**Key (Pi4 → Pi5):**
```
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIBsmdMHxNGDoN/VxCOo9Pq0Zw46WlXE0AGbNVwbwua40 pummelchen@bambuddy
```

**Zugriff via ssh-mcp:**
```python
ssh_exec(command="...", host="pi5", user="pummelchen")
```

**Häufiger Fehler:** `pi5` zeigte in server.py auf `192.168.178.1` (Fritz!Box-IP) statt `192.168.178.163`. Bereits gefixt (2026-06-24).

**Key beim Addon hinterlegen (falls nötig):**
Via `ha_manage_addon(slug="a0d7b954_ssh", options={"ssh": {..., "authorized_keys": ["ssh-ed25519 ..."]}})`  
danach `ha_manage_addon(slug="a0d7b954_ssh", action="restart")`.

---

## Docker-Container (getrennte Container)

| Container | Port | Transport | Endpoint |
|---|---|---|---|
| skill-writer-mcp | 8001 | streamable-http | `/mcp` |
| ssh-mcp | 8002 | streamable-http | `/mcp` |
| bambuddy | intern | – | Port 8000 |
| portainer | 9000 | – | Web-UI |

### skill-writer-mcp
- **Projektverzeichnis:** `~/skill-writer-mcp/`
- **Image:** python:3.12-slim (selbst gebaut)
- **Abhängigkeiten:** fastmcp 3.4.2, httpx
- **Transport:** `streamable-http` auf Port 8001, lauscht auf `/mcp`
- **GitHub-Repo:** `krueli/claude-skills`, Branch `main`, Unterverzeichnis `skills/`
- **GitHub-Token:** via Env `GITHUB_TOKEN` im docker-compose.yml

### ssh-mcp
- **Projektverzeichnis:** `~/ssh-mcp/`
- **Transport:** `streamable-http` auf Port 8002, lauscht auf `/mcp`
- **SSH-Key:** `/home/pummelchen/.ssh/id_ed25519` gemountet als `/root/.ssh` (read-only Volume)
- **HOSTS-Map in server.py:**
  ```python
  HOSTS = {"pi4": "192.168.178.112", "localhost": "192.168.178.112", "pi5": "192.168.178.163"}
  ```
- **SSH-Optionen:** StrictHostKeyChecking=no, BatchMode=yes, ConnectTimeout=10

---

## Tailscale Funnel

```
https://bambuddy.tail5738ee.ts.net (Funnel on)
|-- /     proxy http://127.0.0.1:443
|-- /ssh  proxy http://127.0.0.1:8002
|-- /sse  proxy http://127.0.0.1:8001/mcp
```

**Funnel neu aufsetzen nach Reset:**
```bash
sudo tailscale serve reset
tailscale serve --bg --set-path /sse http://127.0.0.1:8001/mcp
tailscale serve --bg --set-path /ssh http://127.0.0.1:8002
tailscale funnel --bg 443
tailscale serve status
```

**Achtung:** Jede `tailscale funnel`- oder `tailscale serve`-Änderung unterbricht kurz den Tunnel. Kurz warten und erneut probieren.

---

## MCP-Endpoints für Claude.ai

| Service | URL |
|---|---|
| skill-writer | `https://bambuddy.tail5738ee.ts.net/sse` |
| ssh-mcp | `https://bambuddy.tail5738ee.ts.net/ssh/mcp` |

---

## SSH-MCP Tools

- `ssh_exec(command, host="pi4", user="pummelchen")` – beliebiger Shell-Befehl
- `ssh_docker_ps(host="pi4")` – docker ps
- `ssh_docker_logs(container, lines=50, host="pi4")` – Container-Logs
- `ssh_write_file(path, content, host="pi4")` – Datei schreiben (base64)
- `ssh_read_file(path, host="pi4")` – Datei lesen

**Wichtig:** `ssh_write_file` schreibt base64-enkodiert. Für längere Dateien besser `ssh_exec` mit Heredoc:
```bash
cat > /pfad/zur/datei << 'EOF'
...Inhalt...
EOF
```

---

## Deploy-Workflow (nach Code-Änderung in server.py)

```bash
# 1. server.py via ssh_exec + Python patchen (sed/python3 inline)
# 2. Neu bauen (im Hintergrund, Timeout-sicher):
cd ~/ssh-mcp && docker compose up -d --build > /tmp/build.log 2>&1 &
# 3. Kurz warten, dann Logs prüfen:
docker ps --filter name=ssh-mcp
```

**Timeout-Verhalten:** `docker compose up --build` kann 30–60 s dauern und den SSH-Timeout überschreiten. Immer als Hintergrundprozess starten.

---

## Wichtige Dateipfade Pi4

| Pfad | Inhalt |
|---|---|
| `~/skill-writer-mcp/server.py` | skill-writer MCP-Server |
| `~/skill-writer-mcp/docker-compose.yml` | Compose-Konfiguration |
| `~/ssh-mcp/server.py` | ssh-mcp MCP-Server (HOSTS-Map hier pflegen) |
| `~/ssh-mcp/docker-compose.yml` | Compose-Konfiguration |
| `~/.ssh/id_ed25519` | Private Key (auch im ssh-mcp-Container als /root/.ssh) |
| `~/.ssh/id_ed25519.pub` | Public Key |

---

## HA Shell Command Bridge

Definiert in `/config/packages/ssh_pi4.yaml`:

| Service | Funktion |
|---|---|
| `shell_command.pi4_update_server` | `docker compose up -d --build` in ~/skill-writer-mcp |
| `shell_command.pi4_sync_disk` | sync |
| `shell_command.run_fix` | `python3 /tmp/fix_claude.py` |
| `shell_command.write_file_b64` | Datei auf Pi5 schreiben |
| `shell_command.read_file_raw` | Datei von Pi5 lesen |

---

## Bekannte Fallstricke

- **Falsche Pi5-IP:** `pi5` in server.py zeigte auf `192.168.178.1` (Fritz!Box). Korrekt: `192.168.178.163`.
- **Kein SSH auf Port 22 direkt:** HAOS blockiert Port 22 nativ. Nur via SSH-Addon erreichbar (läuft host_network, also doch Port 22 auf der Pi5-IP).
- **BatchMode=yes erzwingt Key-Auth:** Passwort-Auth funktioniert im ssh-mcp-Container nicht. Key muss in `authorized_keys` des Addons stehen.
- **Tailscale Funnel-Änderungen** reißen kurz den Tunnel weg – ssh-mcp verliert Verbindung. Immer kurz warten.
- **Build-Timeouts:** `docker compose up --build` im Vordergrund läuft in SSH-Timeout. Als `&` im Hintergrund starten.

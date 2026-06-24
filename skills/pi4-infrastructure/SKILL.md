# pi4-infrastructure

**Trigger:** Verwende diesen Skill bei allem was den Raspberry Pi 4 (bambuddy/Pi4) betrifft: Docker-Container, skill-writer, ssh-mcp, Bambuddy, Tailscale, Dateitransfer, Remote-Befehle.

**Stand:** 2026-06-24

---

## Hardware

- **Gerät:** Raspberry Pi 4
- **Hostname:** bambuddy (User: pummelchen)
- **IP lokal:** 192.168.178.112
- **Tailscale IP:** 100.125.201.7
- **Tailscale Subnet-Router:** 192.168.178.0/24

---

## Docker-Container (getrennte Container)

Seit 2026-06-24 laufen skill-writer und ssh-mcp in **getrennten Containern**.

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
- **SSH-Key:** gemountet im Container (Volume)
- **Bekannte Hosts:** `pi4` (192.168.178.112), `pi5` (192.168.178.163)

---

## Tailscale Funnel

```
https://bambuddy.tail5738ee.ts.net (Funnel on)
|-- /     proxy http://127.0.0.1:443
|-- /ssh  proxy http://127.0.0.1:8002
|-- /sse  proxy http://127.0.0.1:8001/mcp
```

Wichtig: `/sse` zeigt auf `http://127.0.0.1:8001/mcp` (mit Pfad!), weil Claude.ai den Connector-Endpoint `/sse` benutzt, der Container aber streamable-http auf `/mcp` horcht. Der Funnel übersetzt den Pfad.

**Funnel neu aufsetzen nach Reset:**
```bash
sudo tailscale serve reset
tailscale serve --bg --set-path /sse http://127.0.0.1:8001/mcp
tailscale serve --bg --set-path /ssh http://127.0.0.1:8002
tailscale funnel --bg 443
tailscale serve status
```

**Achtung:** Jede `tailscale funnel`- oder `tailscale serve`-Änderung unterbricht kurz den Tunnel. ssh-mcp fällt dabei ebenfalls aus (läuft über denselben Funnel). Kurz warten und erneut probieren.

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

**Wichtig:** `ssh_write_file` schreibt base64-enkodiert. Für längere Dateien besser `ssh_exec` mit Heredoc nutzen:
```bash
cat > /pfad/zur/datei << 'EOF'
...Inhalt...
EOF
```

---

## Deploy-Workflow skill-writer-mcp (nach Code-Änderung)

```bash
# 1. server.py direkt schreiben (via ssh_exec + Heredoc)
# 2. Neu bauen:
cd /home/pummelchen/skill-writer-mcp && docker compose up -d --build
# 3. Logs prüfen:
docker logs skill-writer-mcp --tail 8
```

---

## Wichtige Dateipfade Pi4

| Pfad | Inhalt |
|---|---|
| `~/skill-writer-mcp/server.py` | skill-writer MCP-Server |
| `~/skill-writer-mcp/docker-compose.yml` | Compose-Konfiguration |
| `~/skill-writer-mcp/Dockerfile` | python:3.12-slim, fastmcp + httpx |
| `~/ssh-mcp/server.py` | ssh-mcp MCP-Server |
| `~/ssh-mcp/docker-compose.yml` | Compose-Konfiguration |

---

## HA Shell Command Bridge (Zugriff auf Pi4 von HA aus)

Definiert in `/config/packages/ssh_pi4.yaml`:

| Service | Funktion |
|---|---|
| `shell_command.pi4_update_server` | `docker compose up -d --build` in ~/skill-writer-mcp – nur bewusst aufrufen |
| `shell_command.pi4_sync_disk` | sync |
| `shell_command.run_fix` | `python3 /tmp/fix_claude.py` – generischer SSH-Executor |
| `shell_command.write_file_b64` | Datei auf Pi5 schreiben (param: `content_b64`) |
| `shell_command.read_file_raw` | Datei von Pi5 lesen |

---

## Bekannte Fallstricke

- Tailscale Funnel-Änderungen reißen kurz den Tunnel weg – ssh-mcp verliert dabei die Verbindung. Immer kurz warten.
- `tailscale funnel --bg --set-path X on` funktioniert nicht (Syntax-Fehler). Stattdessen: erst `serve --set-path`, dann `funnel --bg 443`.
- `tailscale serve --set-path` schlägt fehl mit "listener already exists", wenn Funnel aktiv ist. Erst `sudo tailscale serve reset`, dann neu aufbauen.

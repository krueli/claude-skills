# SSH-MCP VPS-Zugang

## Setup-Status
VPS (167.233.98.24) ist als Host `vps` im ssh-mcp eingetragen und per Key-Auth erreichbar.

## Zugriff
```python
ssh_exec(command="...", host="vps", user="root")
ssh_read_file(path="/root/...", host="vps", user="root")
ssh_write_file(path="/root/...", content="...", host="vps", user="root")
ssh_docker_ps(host="vps", user="root")
```

## Was wurde geändert
`/home/pummelchen/ssh-mcp/server.py` auf Pi4:
- HOSTS-Map um `"vps": "167.233.98.24"` erweitert
- `ssh_read_file`, `ssh_write_file`, `ssh_docker_ps`, `ssh_docker_logs` haben jetzt einen `user`-Parameter (Default: `pummelchen`)
- Image neu gebaut via `docker compose build --no-cache`

## Key-Setup
Pi4-Key (`/home/pummelchen/.ssh/id_ed25519.pub`) liegt in `/root/.ssh/authorized_keys` auf dem VPS.
Eingeschleust per paramiko (Python) innerhalb des ssh-mcp-Containers, da sshpass nicht verfügbar war.

## Wichtige Pfade auf VPS
- Immich: `/root/immich/`
- Nextcloud: `/root/nextcloud/`
- NPM Proxy: `/root/proxy/`
- Storagebox gemountet: `/mnt/storagebox/`

## Immich-Konfiguration (bereits optimal)
- `UPLOAD_LOCATION=/mnt/storagebox/immich-data` — Originale auf Storagebox
- `DB_DATA_LOCATION=./DimIK.1512` — PostgreSQL auf lokaler SSD
- `model-cache` — Docker Volume, lokal auf SSD
- Keine Änderungen nötig, Setup entspricht Best Practice

## Neustart ssh-mcp nach Code-Änderungen
Da server.py ins Image gebacken ist:
```bash
cd /home/pummelchen/ssh-mcp
docker compose build --no-cache
docker compose up -d
```
Build dauert ~60s, daher im Hintergrund starten:
```bash
nohup docker compose build --no-cache > /tmp/build.log 2>&1 &
```

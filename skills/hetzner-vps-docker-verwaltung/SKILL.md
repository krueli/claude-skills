# Hetzner VPS Docker-Verwaltung via ssh-mcp

## Kontext
- VPS: `167.233.98.24`, User: `root`, Port: `22`
- SSH-Key von bambuddy (pummelchen@bambuddy) ist in `~/.ssh/authorized_keys` hinterlegt
- Zugriff via ssh-mcp: `host=167.233.98.24`, `user=root`

## Infrastruktur

### Docker-Container
| Container | Image | Port | Domain |
|---|---|---|---|
| nextcloud-app-1 | nextcloud:stable | 8080 | cloud.m-krueger.de |
| nextcloud-db-1 | mariadb:10.11 | 3306 | intern |
| immich_server | immich-server:v2 | 2283 | datenbank.m-krueger.de |
| immich_postgres | postgres:14 | 5432 | intern |
| immich_machine_learning | immich-ml:v2 | - | intern |
| immich_redis | valkey:9 | 6379 | intern |
| proxy-nginx-proxy-manager-1 | jc21/nginx-proxy-manager | 80/81/443 | - |

### Compose-Verzeichnisse
- `/root/nextcloud/docker-compose.yml`
- `/root/immich/docker-compose.yml` + `.env`
- `/root/proxy/`

### Storagebox
- Mount: `/mnt/storagebox` (CIFS, `//u623627.your-storagebox.de/backup`)
- User: `u623627`, SSH-Port: 23
- Credentials: `/etc/storagebox-credentials`
- fstab: `uid=33,gid=33,file_mode=0770,dir_mode=0770` (www-data für Nextcloud)
- Ordnerstruktur:
  - `/mnt/storagebox/immich-data` → Immich Uploads (UPLOAD_LOCATION in .env)
  - `/mnt/storagebox/nextcloud-data` → Nextcloud Datadirectory
  - `/mnt/storagebox/OneDrive` → OneDrive-Archiv (WebDAV-Mount in Nextcloud)
  - `/mnt/storagebox/iCloud` → iCloud-Archiv (WebDAV-Mount in Nextcloud)

## Wichtige Konfiguration

### Immich .env (`/root/immich/.env`)
```
UPLOAD_LOCATION=/mnt/storagebox/immich-data
DB_DATA_LOCATION=./DimIK.1512
IMMICH_VERSION=v2
DB_PASSWORD=postgres
DB_USERNAME=postgres
DB_DATABASE_NAME=immich
```

### Nextcloud docker-compose.yml (Volumes)
```yaml
volumes:
  - ./config:/var/www/html
  - /mnt/storagebox/nextcloud-data:/var/www/html/data
  - /mnt/storagebox:/mnt/storagebox
```

### Nextcloud externe Speicher (occ files_external:list)
- Mount ID 6: `/OneDrive` → WebDAV auf Storagebox
- Mount ID 7: `/iCloud` → WebDAV auf Storagebox

## Bekannte Fallstricke

### NPM und Container in getrennten Netzwerken
NPM läuft in `proxy_default`, andere Container in eigenen Netzwerken.
Ziel-IP für NPM-Proxy-Hosts: `172.17.0.1` (Docker-Host), NICHT `127.x.x.x`.

### Storagebox uid/gid
CIFS-Mount muss `uid=33,gid=33` haben damit Nextcloud (www-data) schreiben kann.
Bei Änderung: Container stoppen, umount, mount, Container starten.

### fstab Format
Letzte Spalte muss auf derselben Zeile sein: `nofail 0 0`

## Nützliche Befehle

```bash
# Nextcloud occ-Befehle
docker exec -u www-data nextcloud-app-1 php occ <befehl>

# Immich Status
curl -s https://datenbank.m-krueger.de/api/server/ping

# Nextcloud Status
curl -s https://cloud.m-krueger.de/status.php

# Storagebox neu mounten
umount /mnt/storagebox && mount /mnt/storagebox

# Alle Container Status
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```

## Migrationsstatus (Stand 28.06.2026)
- [x] SSH-Key von bambuddy auf VPS hinterlegt
- [x] Nextcloud-Daten auf Storagebox verlagert
- [x] Immich erreichbar (NPM-IP-Tippfehler behoben: 127.17 → 172.17)
- [x] Nextcloud erreichbar
- [ ] Fotos aus iCloud → Immich (läuft via App, ~62 GB)
- [ ] OneDrive-Dateien selektieren und nach Nextcloud migrieren (~750 GB)
- [ ] Nextcloud-Client auf Windows einrichten
- [ ] Passwörter in docker-compose und .env härten

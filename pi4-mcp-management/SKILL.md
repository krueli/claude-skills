# pi4-mcp-management

Workflow für das Verwalten von MCP-Servern auf dem Pi4 (192.168.178.112) über den HA Shell Command Bridge.

## Voraussetzungen

- SSH-Key liegt unter `/config/.ssh/pi4_key` (nicht `/config/pi4_key`)
- Permissions: `chmod 600 /config/.ssh/pi4_key && chmod 700 /config/.ssh`
- Key muss auf Pi4 in `~/.ssh/authorized_keys` eingetragen sein
- Shell Commands definiert in `/config/packages/ssh_pi4.yaml`

## Kritische Gotchas

### SSH Key Validierung
Bevor SSH-Probleme debuggt werden: `ssh-keygen -l -f /config/.ssh/pi4_key` ausführen.
- `type -1` = Key-Datei kaputt (korrupt, leer, falsches Format)
- In HA (Alpine) einen neuen Key generieren: `ssh-keygen -t ed25519 -f /config/.ssh/pi4_key_new -N '' -C 'ha-to-pi4'`
- Public Key auf Pi4 eintragen: `echo 'ssh-ed25519 AAAA...' >> ~/.ssh/authorized_keys`
- Permissions auf Pi4: `chmod 600 ~/.ssh/authorized_keys`

### SSH-Test ohne false positive
`ssh ... 'echo OK' 2>&1 | head -70` gibt returncode 0 von `head`, nicht SSH.
Korrekt: `ssh ... 'echo OK_AUTH' 2>&1` ohne Pipe — returncode direkt von SSH prüfen.

### Docker und .env
- `docker compose restart` lädt `.env` **nicht** zuverlässig neu
- `docker compose down && docker compose up -d --build` ist die sichere Variante
- Nach Code-Änderungen: erst SCP/Copy, dann `--build`

### Disk vs. Container Divergenz
Docker `--build` nutzt den Dateisystem-Stand auf Pi4. Wenn der Container-Code neuer ist als die Quelldatei:
```bash
docker cp skill-writer-mcp:/app/server.py ~/skill-writer-mcp/server.py
```

## Shell Commands in ssh_pi4.yaml

```yaml
shell_command:
  pi4_fix_key: "chmod 600 /config/.ssh/pi4_key && chmod 700 /config/.ssh"
  pi4_ssh_test: "ssh -i /config/.ssh/pi4_key -o StrictHostKeyChecking=no -o BatchMode=yes pummelchen@192.168.178.112 'echo OK_AUTH' 2>&1"
  pi4_docker_restart: "ssh -i /config/.ssh/pi4_key -o StrictHostKeyChecking=no -o BatchMode=yes pummelchen@192.168.178.112 'cd ~/skill-writer-mcp && docker compose restart' 2>&1"
  pi4_docker_logs: "ssh -i /config/.ssh/pi4_key -o StrictHostKeyChecking=no -o BatchMode=yes pummelchen@192.168.178.112 'docker logs skill-writer-mcp --tail 50' 2>&1"
  pi4_update_server: "ssh -i /config/.ssh/pi4_key -o StrictHostKeyChecking=no -o BatchMode=yes pummelchen@192.168.178.112 'cd ~/skill-writer-mcp && docker compose down && docker compose up -d --build' 2>&1"
  pi4_sync_disk: "ssh -i /config/.ssh/pi4_key -o StrictHostKeyChecking=no -o BatchMode=yes pummelchen@192.168.178.112 'docker cp skill-writer-mcp:/app/server.py ~/skill-writer-mcp/server.py && echo SYNCED' 2>&1"
```

## Shell Command Return Response

HA Shell Commands geben nur dann `stdout`/`stderr` zurück wenn `return_response=true` im URL-Parameter:
```python
await api_post("/services/shell_command/pi4_docker_logs?return_response=true", {})
result["service_response"]["stdout"]
```

Nur Commands mit `response: optional: true` in der Service-Definition unterstützen das nativ.
Workaround für eigene Commands: Output in `/tmp/datei.txt` schreiben, dann via `read_file_raw` lesen.

## Dateien auf Pi4 übertragen

SCP über den HA-SSH-Key:
```yaml
pi4_scp_server: "scp -i /config/.ssh/pi4_key -o StrictHostKeyChecking=no /tmp/server_clean.py pummelchen@192.168.178.112:~/skill-writer-mcp/server.py 2>&1"
```

Datei erst auf HA via `write_file_b64` ablegen, dann per SCP übertragen.

## GitHub Token Diagnose

| HTTP Status | Bedeutung |
|-------------|-----------|
| 401 | Token ungültig oder abgelaufen |
| 403 | Token gültig, aber fehlende Berechtigung (`Contents: Read and write`) |

Bei 401: neuen PAT erstellen, in `.env` eintragen, Container mit `down && up --build` neu starten.

## httpx Bugs (v0.28+)

- `raise_for_status()` gibt `self` zurück → `None or r.json()` Pattern kaputt, immer auf separate Zeilen aufteilen
- `client.delete()` akzeptiert kein `json`-Argument → `client.request("DELETE", url, json={...})` nutzen

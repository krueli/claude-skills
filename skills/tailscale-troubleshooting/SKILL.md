# Tailscale Troubleshooting auf bambuddy (Pi4)

## Setup-Übersicht
- Pi4 "bambuddy" (192.168.178.112, User: pummelchen) ist Tailscale Subnet Router + Funnel-Host
- Tailscale Funnel: `https://bambuddy.tail5738ee.ts.net`
  - `/` -> 127.0.0.1:443
  - `/sse` -> 127.0.0.1:8001/mcp (skill-writer-mcp)
  - `/ssh` -> 127.0.0.1:8002 (ssh-mcp)
- Subnet Route: `192.168.178.0/24`
- Pi5 HA: 192.168.178.163:8123

## Häufige Probleme

### 1. IP Forwarding fehlt (nach Reboot)
**Symptom:** `tailscale status` zeigt Health-Check-Warnung: "Subnet routing is enabled, but IP forwarding is disabled."
**Fix:**
```bash
cat /proc/sys/net/ipv4/ip_forward  # prüfen, 0 = Problem
echo 'net.ipv4.ip_forward = 1' | sudo tee -a /etc/sysctl.conf
echo 'net.ipv6.conf.all.forwarding = 1' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p  # nicht verfügbar ohne TTY über MCP -> direkt auf Pi ausführen
```
**Wichtig:** `sysctl` ist auf bambuddy nicht im PATH via MCP, stattdessen `/proc/sys/` direkt lesen.

### 2. `tailscale up` mit neuen Flags schreibt Konfiguration neu
**Symptom:** Nach `tailscale up --advertise-routes=...` schlägt der Befehl fehl mit "requires mentioning all non-default flags".
**Fix:** Tailscale gibt den vollständigen Befehl in der Fehlermeldung an. Exakt diesen verwenden:
```bash
tailscale up --accept-routes --advertise-routes=192.168.178.0/24 --advertise-exit-node --operator=pummelchen
```
**Nebenwirkung:** Dieser Befehl kann den Funnel-Daemon zurücksetzen!

### 3. Funnel fällt nach `tailscale up` aus
**Symptom:** `https://bambuddy.tail5738ee.ts.net` nicht erreichbar, ssh-mcp antwortet nicht.
**Fix:** Direkt auf bambuddy:
```bash
sudo tailscale funnel --bg 443
```
Danach wieder alle drei Routen (/, /sse, /ssh) aktiv.

### 4. Subnet Routing auf iOS nicht aktiv
**Symptom:** HA/BB über IP nicht erreichbar trotz korrekter Server-Konfiguration.
**Checkliste:**
- Tailscale Admin-Console -> bambuddy -> "Edit route settings" -> `192.168.178.0/24` aktiviert?
- iOS Tailscale App -> Einstellungen -> "Subnet Routing" -> "Use Tailscale Subnets" an?
- iOS App neu starten

### 5. MagicDNS-Ausfall nach Rekonfiguration
**Symptom:** `homeassistant.local` extern nicht mehr auflösbar (hat vorher über Tailscale MagicDNS funktioniert).
**Hintergrund:** `.local`-Namen funktionieren extern nur über Tailscale MagicDNS, nicht über echtes mDNS.
**Prüfen:**
- Tailscale Admin-Console -> DNS -> MagicDNS aktiv?
- iOS App -> Einstellungen -> "Use Tailscale DNS" aktiv?
**Workaround:** Direkt über IP ansprechen: `http://192.168.178.163:8123`

## Diagnose-Befehle (via ssh-mcp)
```bash
tailscale status                          # Verbindungsstatus + Health-Checks
tailscale funnel status                   # Funnel-Routen
tailscale status --json | python3 -c "import json,sys; d=json.load(sys.stdin); print(json.dumps(d.get('Self',{}), indent=2))"
cat /proc/sys/net/ipv4/ip_forward         # IP Forwarding Status (1=aktiv)
ping -c 2 192.168.178.163                 # HA lokal erreichbar von bambuddy?
curl -s -o /dev/null -w "%{http_code}" http://192.168.178.163:8123  # HA HTTP-Status
docker ps                                  # Container-Status
```

## Reihenfolge bei Totalausfall
1. Lokal auf bambuddy SSH: `ssh pummelchen@192.168.178.112`
2. IP Forwarding prüfen/setzen
3. `tailscale up` mit allen Flags (aus Fehlermeldung kopieren)
4. `sudo tailscale funnel --bg 443`
5. Subnet-Route in Admin-Console genehmigen
6. iOS App: Subnets + DNS prüfen
7. Zugriff über IP testen, nicht über `.local`

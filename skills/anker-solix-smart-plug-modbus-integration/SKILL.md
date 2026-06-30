# Anker SOLIX Smart Plug – lokale Modbus-Integration in HA

## Kontext
Michael hat zwei Anker SOLIX Smart Plug Gen 2 (Modellcode A17X8) im Netz,
betrieben über sein Solix-Ökosystem. Trotz fehlender öffentlicher
Dokumentation sprechen diese Geräte echtes Modbus TCP (Port 502).

## Wichtige Erkenntnis: Skepsis vs. Realität
"Anker Smart Plug über Solix" klang zunächst nach Cloud-only-Hardware ohne
Modbus. Tatsächlich unterstützen Anker SOLIX Smart Plug Gen 2 (und einige
weitere Solix-Geräte: Solarbank Max AC, Solarbank 4 E5000 Pro, Smart Meter
Gen 2) lokales Modbus TCP – muss aber in der Anker-App erst freigeschaltet
werden: Geräte-Settings → "Three-Party Control Settings" → Modbus-TCP-Toggle.
Vor dem Einbinden lohnt sich ein schneller Port-502-Check (z.B.
`echo > /dev/tcp/IP/502`), bevor man eine Cloud-Lösung vorschlägt.

## Identifikation per Modbus (wenn unklar, was am anderen Ende hängt)
FC4 (Read Input Registers) ab Adresse 30000 liefert Klartext-Gerätedaten:
Modellcode (z.B. "A17X8"), Seriennummer, Firmware-Version. FC3 (Holding
Registers) bei Adresse 0 liefert nur eine "Illegal Data Address"-Exception
auf diesen Geräten – kein Fehlschlag, sondern Bestätigung eines echten
Modbus-Slaves mit anderer Registerkarte.

## Installation: offizielle Integration statt eigenem Reverse-Engineering
Es gibt eine offizielle Anker-Integration für genau diesen Zweck:
`anker-charging/ha-anker-solix-official` (Domain: `anker_solix_official`),
bereits im HACS-Standardkatalog gelistet (nicht erst als Custom-Repo
hinzufügen). Installation:
1. `ha_get_hacs_info(action="search", category="integration", query="anker solix official")`
   → liefert die HACS-`repository_id`.
2. `ha_manage_hacs(action="download", repository_id=<id>)`
3. `ha_restart(confirm=true)` – Verbindung bricht dabei erwartungsgemäß kurz ab.

## Grenze der Automatisierung: Config-Flow-Schritt
Es gibt KEIN MCP-Tool, um einen neuen Config-Entry für eine unbekannte
Integrations-Domain per Config-Flow anzulegen (nur für bekannte Helper-Typen
via `ha_config_set_helper`). Das Hinzufügen über Settings → Devices &
Services → Add Integration → IP eingeben muss der Nutzer einmalig manuell
machen. Das ist eine echte Tool-Grenze, kein Bequemlichkeits-Punkt.

## Entity-Namensgebung
Die Integration übernimmt den in der Anker-App vergebenen Gerätenamen als
Entity-Präfix (nicht "Smart Plug 001/002"). Vor dem Dashboard-Bau also
immer per `ha_search` (domain_filter sensor/switch, query "anker_solix" o.ä.)
die tatsächlichen Entity-IDs auflösen statt zu raten.

Pro Plug verfügbare Entities: Wirkleistung (W), Strom (A), Spannung (V),
Schaltstatus, Steckdosenschalter, Temperatur, Geräte-Seriennummer,
Gerätemodell. KEINE kumulierte Energie (kWh) in dieser Integrationsversion –
für Tages-/Monatsverbrauch im Energie-Dashboard wird zusätzlich ein
`sensor.integration`-Helper auf Basis der Wirkleistung gebraucht.

## Dashboard-Pattern: "Stromverbraucher" auf home-overview
Michael hat keinen Abschnitt namens "Stromverbraucher" – das eigentliche
Äquivalent ist die Karte "⚡ Geräte Verbrauch" (custom:button-card mit
JS-Template in custom_fields.content) in der Sektion "Netz & Smart Home"
auf dem Dashboard `home-overview`. Neue Verbraucher werden dort als
`${wattRow('Label', 'sensor.xxx_power')}` vor der abschließenden
"Gesamtbedarf"-Zeile eingefügt – nicht als neue Sektion/Karte.
Pfad zum Auffinden: `ha_config_get_dashboard(card_type="custom:button-card")`
liefert python_path je Karte; Karte mit `name: "⚡ Geräte Verbrauch"` ist
die richtige. Bearbeitung über `ha_config_set_dashboard(python_transform=...)`
mit `.replace()` auf den content-String (gezielter Insert, kein Full-Replace).

## Karten-Aufräumung in "Netz & Smart Home" (Stand dieser Session)
Ursprünglich lagen dort DREI button-cards mit teils vertauschten Inhalten
(Titel passte nicht zum gerenderten Content – vermutlich ein alter
Copy-Paste-Fehler, kein reines Duplikat):
- "⚡ Geräte Verbrauch": korrekt, Geräte-Verbrauchsliste.
- "🔋 Solarbank Übersicht": zeigte fälschlich dieselbe Verbrauchsliste.
- "🌐 FritzBox 6690": zeigte fälschlich Solar-/Akkudaten (SoC, Akku-Energie,
  Solarleistung, Hausabgabe, Betriebszustand), tap_action auf Router-IP.

Endgültiger Zustand nach Bereinigung: Beide "Solarbank Übersicht" und
"FritzBox 6690" wurden komplett gelöscht. Begründung des Nutzers: Solar-/
Akkudaten stehen bereits vollständig in der Karte "☀️ Photovoltaik"
(Sektion "Solar & Energie", PV Erzeugung/Hausverbrauch/Netzeinspeisung/
Netzbezug/Akku SoC/Ertrag gesamt) – eine zweite Solar-Karte war redundant,
nicht nur falsch beschriftet. Sektion "Netz & Smart Home" besteht jetzt
nur noch aus: heading, Geräte Verbrauch, Müllabfuhr.

Lehre für künftige Dashboard-Aufräumarbeiten: Bei einer vermeintlichen
"Reparatur" einer falsch beschrifteten Karte immer auch prüfen, ob es
bereits eine andere, thematisch passende Karte mit denselben Daten gibt
(hier: Photovoltaik-Karte) – dann eher löschen als zweimal pflegen.

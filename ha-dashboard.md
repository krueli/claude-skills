name: ha-dashboard
description: Use this skill when working with Home Assistant dashboards, ApexCharts cards, Lovelace configuration, or any HA dashboard editing task. Covers dashboard updates, card configuration, energy charts, cost tracking, and common pitfalls.

Home Assistant Dashboard

Pflichtregeln

Zeitangaben: Immer im Format HH:MM, 24-Stunden-Format. Beispiel: 08:30, 17:00.

Eurobeträge: Immer genau zwei Dezimalstellen. Beispiel: 0,27 €/kWh, 1,23 €.

Kartenindizes: Zero-based. Das energie-solar Dashboard: cards[7] Entities-Karte, cards[8] “Stündlicher Netzbezug & Kosten”, cards[9] “Tibber-Preis vs. Festtarif”.

ApexCharts

config_hash: Bei python_transform immer den vollständigen Config-Payload übergeben, nicht nur geänderte Teile. Ein unvollständiger Payload verursacht config_hash-Mismatch und die Karte bricht.

force_reload: Immer force_reload: true setzen, damit die aktuelle Konfiguration geladen wird.

Dezimalformatierung: float_precision für Nachkommastellen nutzen.

Serienreihenfolge: Festtarif immer als erste Serie, Tibber-Simulation danach.

kWh-Konvertierung: Wh-Werte aus Sensoren durch 1000 teilen. In ApexCharts per transform: “return x / 1000;”

Energiekosten

Festpreis aktuell: 0,27 €/kWh (Volkswagen-Tarif).

Tibber-Formel: (Nordpool + 0,17895 €/kWh) × 1,19.

Dashboard-Struktur

	•	dashboard-ubersicht: Sicherheits-Karte bei views[0].cards[6]
	•	dashboard-sicherheit: flache cards-Liste
	•	dashboard-info: sections-basiertes Layout
	•	Ring-Kamera Entity: camera.haustur_live_ansicht

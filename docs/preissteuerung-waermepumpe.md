# Preissteuerung Wärmepumpe – Entwurf

Ziel: Sperr- bzw. Boost-Zeiten der Wärmepumpe anhand des dynamischen Strompreises setzen, Logik in n8n, Ausführung direkt auf Original-Entities in Home Assistant (keine zusätzlichen Helper oder Automationen).

## Klärung: Verhältnis zum bestehenden „WW"-System

Geprüft (2026-09-26): `WW V1–V3` (`script.ww_freigabe_erteilen`, `script.ww_v3_ladung_starten`, Automationen wie `WW V3 - Nachtgarantie 04:00`, `WW V3 - P0 Kritischer Schutz`, `input_datetime.ww_letzte_warmwasseranforderung` usw.) ist ein bereits bestehendes, ausgereiftes Tibber-Preissteuerungssystem für die **Warmwasserbereitung (Trinkwarmwasser-Ladung)** – nicht für die Raumheizung. Es hat einen eigenen Zweck (Ladezyklen, Nachtgarantie, PV-Sperre, Sicherheitsabschaltung) und wird von diesem Entwurf **nicht angefasst**.

Dieser Entwurf betrifft die **Raumheizung** (Heizkreis) der Wärmepumpe und ist damit ein eigenständiger, zweiter Anwendungsfall neben WW.

## Bestand, relevant für Raumheizung

- `sensor.home_stunden_preis_aktuell` – geglätteter aktueller Preis (ct/kWh, Tibber)
- `switch.boiler_heatingoff` – natives EMS-ESP-Signal „Force heating off" über das BBQKees-Gateway am Service-Bus der Wärmepumpe. Das ist der korrekte Sperrpunkt.
- `climate.thermostat_hc1` – natives ems-esp-Thermostat des Heizkreises hc1, `hvac_modes: [auto, heat]`, unterstützt `target_temperature`

**Korrektur gegenüber einer früheren Entwurfsversion:** Zunächst war `switch.warmepumpe` (ein Shelly-Relais) als Sperrpunkt vorgesehen. Andreas hat klargestellt, dass dieser Shelly gar nichts schaltet (reine Messung/Anzeige) und stattdessen die MQTT-Werte des EMS-ESP-Gateways (BBQKees) am Service-Bus der Wärmepumpe die validen Steuerpunkte sind. `switch.boiler_heatingoff` ist genau dafür vorgesehen (Software-Sperrsignal auf dem EMS-Bus, kein hartes Abschalten der Stromversorgung) und wird deshalb jetzt verwendet.

## Architekturentscheidung

Keine neuen HA-Helper oder eigene Interlock-Automation. n8n liest bei jedem Lauf den aktuellen Zustand direkt aus den Original-Entities (`sensor.home_stunden_preis_aktuell`, `switch.boiler_heatingoff`, `climate.thermostat_hc1`) und wendet die Hysterese auf diesen tatsächlichen Zustand an – dadurch ist die Logik zustandslos (kein Gedächtnis in n8n nötig, übersteht Neustarts) und ausschließlich native HA-Zustände sind maßgeblich. Fällt n8n aus, bleibt die Wärmepumpe einfach im zuletzt gesetzten Zustand.

## n8n-Workflow

Datei: [`n8n/preissteuerung-waermepumpe.json`](../n8n/preissteuerung-waermepumpe.json)

Ablauf (alle 15 Minuten):
1. Liest `sensor.home_stunden_preis_aktuell`, `switch.boiler_heatingoff` und `climate.thermostat_hc1` aus HA.
2. Berechnet in einem Code-Node mit Hysterese (±1.5 ct/kWh um zwei Schwellen, Default 40/20 ct/kWh):
   - **Sperre**: `switch.boiler_heatingoff` an, wenn Preis deutlich über der Sperrschwelle; wieder aus, wenn deutlich darunter.
   - **Boost**: `climate.thermostat_hc1` auf `hvac_mode: heat`, wenn Preis deutlich unter der Boostschwelle und die Wärmepumpe nicht gesperrt ist/wird; zurück auf `auto`, wenn der Preis wieder darüber liegt.
3. Ruft nur bei tatsächlich nötiger Änderung den entsprechenden HA-Service auf (`switch.turn_on/off`, `climate.set_hvac_mode`).

Schwellen (`sperrschwelle`, `boostschwelle`, `hysterese`) stehen als Konstanten oben im Code-Node – zum Anpassen den Workflow in n8n öffnen und dort editieren, keine separate HA-Konfiguration nötig.

### Setup in n8n
- Environment-Variable `HA_BASE_URL` (z. B. `http://homeassistant.local:8123`).
- Credential vom Typ „Header Auth" namens „HA Long-Lived Token" mit Header `Authorization: Bearer <Long-Lived Access Token>`. Die Platzhalter-Credential-ID (`PLACEHOLDER_HA_TOKEN_CREDENTIAL_ID`) muss beim Import in n8n neu verknüpft werden.
- Workflow ist beim Import inaktiv (`active: false`) – bewusst, erst nach finaler Prüfung durch Andreas scharf schalten.

## Geprüft: kein Konflikt mit Warmwasser/WW-System

Nur ein Heizkreis (`hc1`) vorhanden, keine weiteren Heizkreise betroffen. Laut EMS-ESP-Dokumentation ist `forceheatingoff` genau für Wärmepumpen gedacht (ein einfaches Aus-Schalten über „Heating Activated" funktioniert bei Wärmepumpen im Gegensatz zu Gaskesseln nicht zuverlässig). Andreas hat bestätigt, dass die Warmwasserbereitung/-Ladung (WW-System) von `switch.boiler_heatingoff` unberührt bleibt – kein Interferenzrisiko mit der bestehenden WW-Preissteuerung.

## Restrisiken / offene Punkte

- Kein Blick auf reale Mindestlaufzeiten/-standzeiten des Kompressors: Die Hysterese verhindert Takten am Preis-Schwellwert, aber nicht bei schnell schwankendem Preis über mehrere Intervalle. Falls der Kompressor empfindlich reagiert, ggf. Hysterese vergrößern oder Mindest-Intervall zwischen zwei Schaltvorgängen im Code-Node ergänzen.

## Nächste Schritte (nach Freigabe durch Andreas)

1. Workflow in n8n importieren, Credential verknüpfen, Schwellen im Code-Node prüfen, danach aktivieren.

# Preissteuerung Wärmepumpe – Entwurf

Ziel: Sperr- bzw. Boost-Zeiten der Wärmepumpe anhand des dynamischen Strompreises setzen, Logik in n8n, Ausführung direkt auf Original-Entities in Home Assistant (keine zusätzlichen Helper oder Automationen).

## Klärung: Verhältnis zum bestehenden „WW"-System

Geprüft (2026-09-26): `WW V1–V3` (`script.ww_freigabe_erteilen`, `script.ww_v3_ladung_starten`, Automationen wie `WW V3 - Nachtgarantie 04:00`, `WW V3 - P0 Kritischer Schutz`, `input_datetime.ww_letzte_warmwasseranforderung` usw.) ist ein bereits bestehendes, ausgereiftes Tibber-Preissteuerungssystem für die **Warmwasserbereitung (Trinkwarmwasser-Ladung)** – nicht für die Raumheizung. Es hat einen eigenen Zweck (Ladezyklen, Nachtgarantie, PV-Sperre, Sicherheitsabschaltung) und wird von diesem Entwurf **nicht angefasst**.

Dieser Entwurf betrifft die **Raumheizung** (Heizkreis) der Wärmepumpe und ist damit ein eigenständiger, zweiter Anwendungsfall neben WW.

## Bestand, relevant für Raumheizung

- `sensor.home_strompreis` – natives Preis-Sensor der offiziellen Tibber-Integration, mit `intraday_price_ranking` (0–1) als eingebautem Tagespreisrang
- `switch.boiler_heatingoff` – natives EMS-ESP-Signal „Force heating off" über das BBQKees-Gateway am Service-Bus der Wärmepumpe. Das ist der korrekte Sperrpunkt.
- `climate.thermostat_hc1` – natives ems-esp-Thermostat des Heizkreises hc1, `hvac_modes: [auto, heat]`, unterstützt `target_temperature`
- `sensor.marstek_venus_modbus_wechselrichter_status` / `sensor.marstek_venus_modbus_soc_batterie` – natives Modbus-Gerät des Marstek-Speichers, Lade-/Entladestatus und SoC in %

**Korrektur gegenüber einer früheren Entwurfsversion:** Zunächst war `switch.warmepumpe` (ein Shelly-Relais) als Sperrpunkt vorgesehen. Andreas hat klargestellt, dass dieser Shelly gar nichts schaltet (reine Messung/Anzeige) und stattdessen die MQTT-Werte des EMS-ESP-Gateways (BBQKees) am Service-Bus der Wärmepumpe die validen Steuerpunkte sind. `switch.boiler_heatingoff` ist genau dafür vorgesehen (Software-Sperrsignal auf dem EMS-Bus, kein hartes Abschalten der Stromversorgung) und wird deshalb jetzt verwendet.

## Architekturentscheidung

Keine neuen HA-Helper oder eigene Interlock-Automation. n8n liest bei jedem Lauf den aktuellen Zustand direkt aus den Original-Entities (`sensor.home_strompreis`, `switch.boiler_heatingoff`, `climate.thermostat_hc1`) und wendet die Hysterese auf diesen tatsächlichen Zustand an – dadurch ist die Logik zustandslos (kein Gedächtnis in n8n nötig, übersteht Neustarts) und ausschließlich native HA-Zustände sind maßgeblich. Fällt n8n aus, bleibt die Wärmepumpe einfach im zuletzt gesetzten Zustand.

**Dynamische statt fester Schwellen:** Auf Nachfrage von Andreas, ob feste ct/kWh-Schwellen zeitpunktabhängig sinnvoller gelöst werden können – ja. Statt absoluter ct/kWh-Werte nutzt der Workflow das Attribut `intraday_price_ranking` von `sensor.home_strompreis` (offizielle Tibber-Integration): ein Wert zwischen 0 (günstigste Stunde des Tages) und 1 (teuerste Stunde des Tages), von Tibber selbst laufend aus den bekannten Preisdaten berechnet. Damit passen sich die Schwellen automatisch an das jeweils aktuelle Preisniveau an (z. B. an einen insgesamt teuren oder günstigen Tag), ohne dass ct/kWh-Werte manuell nachgepflegt werden müssen.

**Speicher (Marstek) als Korrektiv:** Auf Nachfrage von Andreas berücksichtigt der Workflow zusätzlich den Marstek-Hausspeicher, damit die reine Preisrang-Logik nicht gegen das läuft, was am Speicher gerade ohnehin passiert:
- Entlädt der Speicher gerade (`wechselrichter_status == Discharge`), wird eine Sperre unterdrückt bzw. eine bestehende Sperre aufgehoben – die Energie kommt dann aus dem Speicher, der Netzpreis ist für diese Stunde weitgehend irrelevant.
- Lädt der Speicher und der SoC liegt bereits über 80% (analog zur Freigabe-Schwelle, die auch das WW-System nutzt), wird zusätzlich zum Preisrang auch das als Boost-Grund gewertet – sonst würde PV-Überschuss ggf. nur noch mit wenig Ertrag eingespeist.
- Der Speicher kann nie zusätzlich eine Sperre auslösen, nur eine preisbedingte Sperre verhindern/aufheben bzw. einen Boost vorziehen – der Preisrang bleibt der Standardfall. Für den SoC gibt es eine eigene Hysterese (±3 Prozentpunkte) gegen Takten an der Schwelle.

Es gibt keine separate Messung, wie viel vom Speicher-Ausstoß speziell in die Wärmepumpe fließt – das bleibt eine Näherung auf Ebene des Gesamthaushalts, keine exakte Zuordnung.

## n8n-Workflow

Datei: [`n8n/preissteuerung-waermepumpe.json`](../n8n/preissteuerung-waermepumpe.json)

Ablauf (alle 15 Minuten):
1. Liest `sensor.home_strompreis` (inkl. `intraday_price_ranking`), `switch.boiler_heatingoff`, `climate.thermostat_hc1`, `sensor.marstek_venus_modbus_wechselrichter_status` und `sensor.marstek_venus_modbus_soc_batterie` aus HA.
2. Berechnet in einem Code-Node mit Hysterese (±0.05 Perzentil um zwei Preis-Schwellen, Default 0.85/0.15 = teuerste/günstigste 15% der Stunden; ±3 Prozentpunkte um die Speicher-SoC-Schwelle, Default 80%):
   - **Sperre**: `switch.boiler_heatingoff` an, wenn der Preisrang deutlich über der Sperrschwelle liegt und der Speicher nicht gerade entlädt; wieder aus, wenn der Preisrang deutlich darunter liegt oder der Speicher zu entladen beginnt.
   - **Boost**: `climate.thermostat_hc1` auf `hvac_mode: heat`, wenn der Preisrang deutlich unter der Boostschwelle liegt oder der Speicher lädt und gut gefüllt ist (SoC > 80%), und die Wärmepumpe nicht gesperrt ist/wird; zurück auf `auto`, wenn beide Boost-Gründe entfallen.
3. Ruft nur bei tatsächlich nötiger Änderung den entsprechenden HA-Service auf (`switch.turn_on/off`, `climate.set_hvac_mode`).

Schwellen (`sperrschwelle`, `boostschwelle`, `hysterese`, `speicherFreigabeSoc`, `speicherHystereseSoc`) stehen als Konstanten oben im Code-Node – zum Anpassen den Workflow in n8n öffnen und dort editieren, keine separate HA-Konfiguration nötig.

### Setup in n8n
- Environment-Variable `HA_BASE_URL` (z. B. `http://homeassistant.local:8123`).
- Credential vom Typ „Header Auth" namens „HA Long-Lived Token" mit Header `Authorization: Bearer <Long-Lived Access Token>`. Die Platzhalter-Credential-ID (`PLACEHOLDER_HA_TOKEN_CREDENTIAL_ID`) muss beim Import in n8n neu verknüpft werden.
- Workflow ist beim Import inaktiv (`active: false`) – bewusst, erst nach finaler Prüfung durch Andreas scharf schalten.

## Geprüft: kein Konflikt mit Warmwasser/WW-System

Nur ein Heizkreis (`hc1`) vorhanden, keine weiteren Heizkreise betroffen. Laut EMS-ESP-Dokumentation ist `forceheatingoff` genau für Wärmepumpen gedacht (ein einfaches Aus-Schalten über „Heating Activated" funktioniert bei Wärmepumpen im Gegensatz zu Gaskesseln nicht zuverlässig). Andreas hat bestätigt, dass die Warmwasserbereitung/-Ladung (WW-System) von `switch.boiler_heatingoff` unberührt bleibt – kein Interferenzrisiko mit der bestehenden WW-Preissteuerung.

## Restrisiken / offene Punkte

- Kein Blick auf reale Mindestlaufzeiten/-standzeiten des Kompressors: Die Hysterese verhindert Takten am Preis-Schwellwert, aber nicht bei schnell schwankendem Preis über mehrere Intervalle. Falls der Kompressor empfindlich reagiert, ggf. Hysterese vergrößern oder Mindest-Intervall zwischen zwei Schaltvorgängen im Code-Node ergänzen.
- `intraday_price_ranking` bezieht sich auf die Tibber-seitig jeweils bekannten Preise (vormittags meist nur der heutige Tag, ab Nachmittag zusätzlich der Folgetag). Der Rang einer frühen Morgenstunde kann sich also im Tagesverlauf verschieben, sobald die Preise für den nächsten Tag bekannt werden – das ist gewolltes Tibber-Verhalten, aber gut zu wissen, falls sich eine Sperr-/Boost-Entscheidung im Tagesverlauf einmal "nachträglich" anders anfühlt als erwartet.

## Nächste Schritte (nach Freigabe durch Andreas)

1. Workflow in n8n importieren, Credential verknüpfen, Schwellen im Code-Node prüfen, danach aktivieren.

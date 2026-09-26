# dev

Home Assistant + n8n Automationen für stromoptimierte Steuerung im Haus.

## Wärmepumpen-Preissteuerung (Raumheizung)

n8n-Workflow, der die Raumheizung der Wärmepumpe anhand des dynamischen Strompreises sperrt bzw. boostet – ohne neue Home-Assistant-Helper oder Automationen, nur über bereits vorhandene Original-Entities.

- **Preis**: `intraday_price_ranking` von `sensor.home_strompreis` (offizielle Tibber-Integration) – der Tagespreisrang (0 = günstigste, 1 = teuerste Stunde des Tages), dynamisch statt fester ct/kWh-Schwellen.
- **Sperre**: `switch.boiler_heatingoff`, das native EMS-ESP-Signal „Force heating off" über das BBQKees-Gateway am Service-Bus der Wärmepumpe.
- **Boost**: `climate.thermostat_hc1` (natives ems-esp-Thermostat) auf `hvac_mode: heat` statt witterungsgeführt `auto`.
- **Speicher-Korrektiv**: Der Marstek-Hausspeicher (`sensor.marstek_venus_modbus_*`) verhindert/hebt eine Sperre auf, wenn er gerade entlädt, und liefert einen zusätzlichen Boost-Grund, wenn er bei hohem SoC lädt.

Läuft unabhängig vom bestehenden `WW`-System, das die Warmwasserbereitung preisgesteuert lädt.

- Workflow: [`n8n/preissteuerung-waermepumpe.json`](n8n/preissteuerung-waermepumpe.json)
- Details, Entscheidungslogik, Setup-Schritte in n8n und Recherchehintergrund: [`docs/preissteuerung-waermepumpe.md`](docs/preissteuerung-waermepumpe.md)

### Inbetriebnahme

1. In Home Assistant unter Profil → Sicherheit einen Long-Lived Access Token erzeugen (falls noch keiner für n8n existiert).
2. Workflow-Datei [`n8n/preissteuerung-waermepumpe.json`](n8n/preissteuerung-waermepumpe.json) in n8n importieren.
3. Ein Credential vom Typ „Home Assistant API" anlegen bzw. das bereits vorhandene verwenden (Host/Port der HA-Instanz, SSL je nach Setup, der Long-Lived Access Token aus Schritt 1).
4. Dieses Credential auf allen 6 „Home Assistant"-Nodes im Workflow verknüpfen (die importierten Nodes zeigen zunächst auf eine Platzhalter-Credential-ID und müssen einmal umgestellt werden).
5. Workflow einmal manuell ausführen („Test workflow") und prüfen, ob alle Nodes grün sind und die gelesenen Werte (Preisrang, Sperr-Status, Heizkreis-Status, Speicher-Status/-SoC) plausibel sind.
6. Schwellen im Node „Entscheidung berechnen" bei Bedarf anpassen (Standard: Sperre ab teuersten 15%, Boost ab günstigsten 15% der Tagesstunden, Speicher-Boost ab 80% SoC).
7. Workflow aktivieren.
8. Zum Deaktivieren reicht es, den Workflow in n8n zu pausieren – Home Assistant behält dann einfach den zuletzt gesetzten Zustand bei. Falls dabei zufällig eine Sperre aktiv war, `switch.boiler_heatingoff` manuell ausschalten und `climate.thermostat_hc1` bei Bedarf wieder auf `auto` stellen.

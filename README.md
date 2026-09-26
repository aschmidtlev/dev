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

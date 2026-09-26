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
- Details, Entscheidungslogik und Recherchehintergrund: [`docs/preissteuerung-waermepumpe.md`](docs/preissteuerung-waermepumpe.md)

### Inbetriebnahme

1. **Long-Lived Access Token in Home Assistant erzeugen**: Profil → Sicherheit → „Langlebige Zugriffstoken" → neues Token erstellen, Wert sicher aufbewahren (wird nur einmal angezeigt).
2. **Workflow in n8n importieren**: `n8n/preissteuerung-waermepumpe.json` über „Import from File" laden.
3. **Environment-Variable setzen**: `HA_BASE_URL` in n8n auf die HA-URL (z. B. `http://homeassistant.local:8123`).
4. **Credential anlegen und verknüpfen**: neue Credential vom Typ „Header Auth" namens „HA Long-Lived Token" mit Header `Authorization: Bearer <Token aus Schritt 1>`. Danach in jedem HTTP-Request-Node des importierten Workflows diese Credential auswählen (der Platzhalter `PLACEHOLDER_HA_TOKEN_CREDENTIAL_ID` wird beim Import nicht automatisch aufgelöst).
5. **Manuell testen**: Workflow einmal per „Execute Workflow" laufen lassen (noch inaktiv) und in HA prüfen, ob `switch.boiler_heatingoff` bzw. `climate.thermostat_hc1` wie erwartet reagieren.
6. **Aktivieren**: Workflow in n8n auf „Active" stellen – ab dann läuft er automatisch alle 15 Minuten.
7. **Wieder ausschalten**: Workflow in n8n auf „Inactive" stellen. Das beendet nur die automatische Steuerung; eine zu diesem Zeitpunkt aktive Sperre bleibt bestehen, bis sie manuell aufgehoben wird: `switch.boiler_heatingoff` in HA von Hand ausschalten (und `climate.thermostat_hc1` bei Bedarf zurück auf `auto`).

Schwellen (Preisrang, Speicher-SoC) stehen als Konstanten im Code-Node „Entscheidung berechnen" – zum Anpassen den Workflow in n8n öffnen, keine separate HA-Konfiguration nötig.

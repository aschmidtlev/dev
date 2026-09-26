# Preissteuerung Wärmepumpe – Entwurf

Ziel: Sperr- bzw. Boost-Zeiten der Wärmepumpe anhand des dynamischen Strompreises setzen, Logik in n8n, Ausführung direkt auf Original-Entities in Home Assistant (keine zusätzlichen Helper oder Automationen).

## Klärung: Verhältnis zum bestehenden „WW"-System

Geprüft (2026-09-26): `WW V1–V3` (`script.ww_freigabe_erteilen`, `script.ww_v3_ladung_starten`, Automationen wie `WW V3 - Nachtgarantie 04:00`, `WW V3 - P0 Kritischer Schutz`, `input_datetime.ww_letzte_warmwasseranforderung` usw.) ist ein bereits bestehendes, ausgereiftes Tibber-Preissteuerungssystem für die **Warmwasserbereitung (Trinkwarmwasser-Ladung)** – nicht für die Raumheizung. Es hat einen eigenen Zweck (Ladezyklen, Nachtgarantie, PV-Sperre, Sicherheitsabschaltung) und wird von diesem Entwurf **nicht angefasst**. Kein Hinweis darauf, dass es `switch.warmepumpe` oder `climate.thermostat_hc1` verwendet (Referenzsuche über automation/script ergab keinen Treffer, templated Referenzen sind darin allerdings nicht erkennbar – siehe Restrisiko unten).

Dieser Entwurf betrifft die **Raumheizung** (Heizkreis) der Wärmepumpe und ist damit ein eigenständiger, zweiter Anwendungsfall neben WW.

## Bestand, relevant für Raumheizung

- `sensor.home_stunden_preis_aktuell` – geglätteter aktueller Preis (ct/kWh, Tibber)
- `switch.warmepumpe` – Shelly-Relais, vermutlich der Hart-Sperrpunkt (Kompressor-Stromversorgung / EVU-Sperre)
- `climate.thermostat_hc1` – natives ems-esp-Thermostat des Heizkreises hc1, `hvac_modes: [auto, heat]`, unterstützt `target_temperature`

**Annahme, bitte bestätigen:** `switch.warmepumpe` ist der richtige Punkt, um die Wärmepumpe hart zu sperren (nicht nur ein Monitoring-Relais), und ein Wechsel von `climate.thermostat_hc1` zwischen `auto` (witterungsgeführt) und `heat` (aktiv fordern) ist für "Boost" sinnvoll. Falls `switch.warmepumpe` z. B. nur eine Zusatzpumpe schaltet, muss der Sperrpunkt angepasst werden.

## Architekturentscheidung

Keine neuen HA-Helper oder eigene Interlock-Automation. n8n liest bei jedem Lauf den aktuellen Zustand direkt aus den Original-Entities (`sensor.home_stunden_preis_aktuell`, `switch.warmepumpe`, `climate.thermostat_hc1`) und wendet die Hysterese auf diesen tatsächlichen Zustand an – dadurch ist die Logik zustandslos (kein Gedächtnis in n8n nötig, übersteht Neustarts) und ausschließlich native HA-Zustände sind maßgeblich. Fällt n8n aus, bleibt die Wärmepumpe einfach im zuletzt gesetzten Zustand.

## n8n-Workflow

Datei: [`n8n/preissteuerung-waermepumpe.json`](../n8n/preissteuerung-waermepumpe.json)

Ablauf (alle 15 Minuten):
1. Liest `sensor.home_stunden_preis_aktuell`, `switch.warmepumpe` und `climate.thermostat_hc1` aus HA.
2. Berechnet in einem Code-Node mit Hysterese (±1.5 ct/kWh um zwei Schwellen, Default 40/20 ct/kWh):
   - **Sperre**: `switch.warmepumpe` aus, wenn Preis deutlich über der Sperrschwelle; wieder an, wenn deutlich darunter.
   - **Boost**: `climate.thermostat_hc1` auf `hvac_mode: heat`, wenn Preis deutlich unter der Boostschwelle und die Wärmepumpe nicht gesperrt ist/wird; zurück auf `auto`, wenn der Preis wieder darüber liegt.
3. Ruft nur bei tatsächlich nötiger Änderung den entsprechenden HA-Service auf (`switch.turn_on/off`, `climate.set_hvac_mode`).

Schwellen (`sperrschwelle`, `boostschwelle`, `hysterese`) stehen als Konstanten oben im Code-Node – zum Anpassen den Workflow in n8n öffnen und dort editieren, keine separate HA-Konfiguration nötig.

### Setup in n8n
- Environment-Variable `HA_BASE_URL` (z. B. `http://homeassistant.local:8123`).
- Credential vom Typ „Header Auth" namens „HA Long-Lived Token" mit Header `Authorization: Bearer <Long-Lived Access Token>`. Die Platzhalter-Credential-ID (`PLACEHOLDER_HA_TOKEN_CREDENTIAL_ID`) muss beim Import in n8n neu verknüpft werden.
- Workflow ist beim Import inaktiv (`active: false`) – bewusst, erst nach Prüfung der Annahme oben scharf schalten.

## Restrisiken / offene Punkte

- Referenzsuche auf `switch.warmepumpe` erfasst keine templated Jinja-Referenzen (`{{ states('switch.warmepumpe') }}`) in YAML-Automationen – ein indirekter Bezug vom WW-System ist damit nicht zu 100 % ausgeschlossen, nur unwahrscheinlich (kein Treffer in ~74 gescannten WW-Konfigobjekten).
- Kein Blick auf reale Mindestlaufzeiten/-standzeiten des Kompressors: Die Hysterese verhindert Takten am Preis-Schwellwert, aber nicht bei schnell schwankendem Preis über mehrere Intervalle. Falls der Kompressor empfindlich reagiert, ggf. Hysterese vergrößern oder Mindest-Intervall zwischen zwei Schaltvorgängen im Code-Node ergänzen.

## Nächste Schritte (nach Freigabe durch Andreas)

1. Annahme zu `switch.warmepumpe` bestätigen.
2. Workflow in n8n importieren, Credential verknüpfen, Schwellen im Code-Node prüfen, danach aktivieren.

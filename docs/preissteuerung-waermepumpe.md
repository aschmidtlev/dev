# Preissteuerung Wärmepumpe – Entwurf

Ziel: Sperr- bzw. Boost-Zeiten der Wärmepumpe anhand des dynamischen Strompreises setzen, Logik in n8n, Ausführung in Home Assistant.

## Bestand (Stichprobe, 2026-09-26)

Bereits vorhanden in HA:
- `sensor.home_stunden_preis_aktuell` – geglätteter aktueller Preis (ct/kWh, Tibber-Daten)
- `sensor.home_preis_heute` / `sensor.home_preis_morgen` – Tagesdurchschnitt
- `switch.warmepumpe` – Schaltausgang der Wärmepumpe
- Ein separates, bereits existierendes System `WW V1–V3` (Automationen/Skripte wie `script.ww_freigabe_erteilen`, `script.ww_stop_durchfuehren`, `sensor.ww_v3_tibber_preis`) – vermutlich Preissteuerung für Warmwasser/Ladung, YAML-basiert, nicht über die Helper-API einsehbar.

**Offene Frage an Andreas:** Ist `WW V1–V3` die bestehende Preissteuerung für die Wärmepumpe (dann: ablösen oder ergänzen?) oder ein separates System (z. B. nur Warmwasserbereitung)? Das entscheidet, ob dieser Entwurf parallel läuft oder das bestehende System ersetzt. Bis zur Klärung ist dieser Entwurf komplett eigenständig über neue Helper-Entities gehalten und fasst `WW*`-Entities nicht an.

Das parallel laufende Bestandsaufnahme-Thread (Phase 1) liefert ggf. weitere/aktuellere Entity-Namen – vor dem Anlegen der Helper in HA nochmal abgleichen.

## Architekturentscheidung

n8n liefert nur die **Absicht** (Sperren/Boosten ja-nein, über Helper-Entities), die tatsächliche Schaltung der Wärmepumpe übernimmt eine lokale HA-Automation. Begründung:

- Ausfallsicherheit: Fällt n8n oder das Netzwerk aus, bleibt der letzte Zustand in HA erhalten statt dass die Wärmepumpe unkontrolliert reagiert.
- Sicherheits-Interlocks (Mindestlaufzeit, Mindeststandzeit des Kompressors, Frostschutz) gehören in HA, nicht in n8n – HA kennt den Live-Zustand des Geräts.
- n8n bleibt austauschbar/testbar, ohne dass an der Hardware-Logik etwas geändert wird.

## Neue HA-Helper (Vorschlag, noch nicht angelegt)

| Entity | Typ | Zweck |
|---|---|---|
| `input_boolean.wp_preissteuerung_aktiv` | input_boolean | Master-Schalter: Wenn aus, greift n8n nicht ein (Wartung/Übersteuerung durch Andreas) |
| `input_boolean.wp_sperre` | input_boolean | Von n8n gesetzt: Preis zu hoch → Sperre anfordern |
| `input_boolean.wp_boost` | input_boolean | Von n8n gesetzt: Preis günstig → Boost anfordern |
| `input_number.wp_sperrschwelle` | input_number | Schwelle in ct/kWh, ab der gesperrt wird (Default 40, 0–100, Schritt 0.5) |
| `input_number.wp_boostschwelle` | input_number | Schwelle in ct/kWh, unter der geboostet wird (Default 20, 0–100, Schritt 0.5) |

Schwellen als Helper statt Konstanten im n8n-Workflow, damit Andreas sie im HA-Dashboard anpassen kann, ohne den Workflow zu bearbeiten.

## HA-Automation (Interlock, Entwurf – noch nicht angelegt)

Reagiert auf `wp_sperre`/`wp_boost` und schaltet `switch.warmepumpe`, mit Mindestlaufzeiten:

```yaml
alias: "WP Preissteuerung: Sperre/Boost anwenden"
description: "Wendet die von n8n gesetzten Preissignale auf die Wärmepumpe an, mit Mindeststandzeit"
mode: single
trigger:
  - platform: state
    entity_id:
      - input_boolean.wp_sperre
      - input_boolean.wp_boost
condition: []
action:
  - choose:
      - conditions:
          - condition: state
            entity_id: input_boolean.wp_sperre
            state: "on"
        sequence:
          - service: switch.turn_off
            target:
              entity_id: switch.warmepumpe
      - conditions:
          - condition: state
            entity_id: input_boolean.wp_boost
            state: "on"
        sequence:
          - service: switch.turn_on
            target:
              entity_id: switch.warmepumpe
```

**Anmerkung:** Diese Automation ist bewusst minimal gehalten. Bevor sie live geht, muss geklärt werden, ob `switch.warmepumpe` der richtige Steuereingang ist (z. B. SG-Ready-Kontakt statt direktem Aus-Schalten wäre für die meisten Wärmepumpen die schonendere Variante) und ob eine Mindestlaufzeit/-standzeit ergänzt werden muss, um Kompressor-Takten zu vermeiden.

## n8n-Workflow

Datei: [`n8n/preissteuerung-waermepumpe.json`](../n8n/preissteuerung-waermepumpe.json)

Ablauf (alle 15 Minuten):
1. Liest `sensor.home_stunden_preis_aktuell`, beide Schwellen und den Master-Schalter aus HA.
2. Berechnet in einem Code-Node mit Hysterese (±1.5 ct/kWh), ob Sperre/Boost an- oder abgeschaltet werden müssen (Hysterese verhindert Takten am Schwellwert).
3. Setzt bei Bedarf `input_boolean.wp_sperre` bzw. `input_boolean.wp_boost` per HA-Service-Call.

### Setup in n8n
- Environment-Variable `HA_BASE_URL` (z. B. `http://homeassistant.local:8123`).
- Credential vom Typ „Header Auth" namens „HA Long-Lived Token" mit Header `Authorization: Bearer <Long-Lived Access Token>`. Die Platzhalter-Credential-ID (`PLACEHOLDER_HA_TOKEN_CREDENTIAL_ID`) muss beim Import in n8n neu verknüpft werden.
- Workflow ist beim Import inaktiv (`active: false`) – bewusst, damit er erst nach Prüfung der Helper/Automation scharf geschaltet wird.

## Nächste Schritte (nach Freigabe durch Andreas)

1. Bestandsaufnahme-Thread abgleichen, Entity-Namen ggf. anpassen.
2. Helper in HA anlegen (`ha_config_set_helper` oder UI).
3. Automation anlegen, dabei Steuereingang (Schalter vs. SG-Ready) und Mindestlaufzeiten final klären.
4. Workflow in n8n importieren, Credential verknüpfen, Schwellen im Dashboard setzen, danach aktivieren.

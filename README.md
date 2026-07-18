# Elli Charger Connect 2 mit Home Assistant, OCPP und lokaler Phasenumschaltung

Diese Notiz dokumentiert den funktionierenden Weg, eine Elli Wallbox / Elli Charger Connect 2 mit Home Assistant über OCPP zu betreiben und zusätzlich die 1p/3p-Phasenumschaltung über die lokale Wallbox-API nutzbar zu machen.

Keine Secrets sind in diesem Dokument enthalten. Passwörter, Long-Lived-Tokens und JWTs müssen lokal sicher abgelegt werden.

## Zielbild

- Home Assistant kommuniziert per OCPP mit der Wallbox.
- OCPP liefert Status, Ladeleistung, Strom, Spannung und Session-Energie.
- Home Assistant startet/stoppt den Ladevorgang über OCPP.
- Die Phasenumschaltung wird nicht über OCPP bereitgestellt, sondern über die lokale Elli-WebGUI/API.
- Home Assistant kann dadurch abhängig vom echten PV-Überschuss zwischen 1-phasigem und 3-phasigem Laden wechseln.
- Für echtes PV-only-Laden sollte zusätzlich die Batterie-Leistung betrachtet werden, nicht nur der Netzbezug.

## Relevante Komponenten

Beispielwerte aus der konkreten Installation:

- Home Assistant: `http://<home-assistant-ip>:8123`
- Wallbox-WebGUI/API: `http://<wallbox-ip>`
- OCPP-Device-ID in Home Assistant: `charger`
- Wallbox: Elli Charger Connect 2

Die IPs und Entity-Namen können je nach Installation abweichen.

## OCPP-Grundintegration in Home Assistant

Die Elli Wallbox wird in Home Assistant über die OCPP-Integration eingebunden. Danach sind typischerweise Entities wie diese verfügbar:

```text
sensor.charger_power_active_import      aktuelle Ladeleistung, meist kW
sensor.charger_current_import           Ladestrom, A
sensor.charger_voltage                  Spannung, V
sensor.charger_energy_session           geladene Energie der Session, kWh
sensor.charger_status_connector         Charging / Preparing / Available / ...
switch.charger_charge_control           Start/Stopp
number.charger_maximum_current          maximaler Ladestrom
```

Für PV-Überschusslogik werden zusätzlich Energiesensoren aus der Hausinstallation benötigt, z. B.:

```text
sensor.strombezug
sensor.aktueller_gesamtverbrauch
sensor.pv_power
sensor.battery_state_of_charge
sensor.battery_power                  Hausbatterie-Leistung; Vorzeichen je nach Integration prüfen
```

## OCPP-MeterValues konfigurieren

Ab Werk sendet die Wallbox über OCPP unter Umständen nur Energie und SoC. Damit Home Assistant Live-Werte wie Leistung, Strom und Spannung bekommt, muss die OCPP-Konfiguration der Wallbox erweitert werden.

In Home Assistant über Entwicklerwerkzeuge → Aktionen:

Service:

```yaml
action: ocpp.configure
data:
  devid: charger
  ocpp_key: MeterValuesSampledData
  value: Energy.Active.Import.Register,Power.Active.Import,Current.Import,Voltage,SoC
```

Optional das Abtastintervall verkürzen:

```yaml
action: ocpp.configure
data:
  devid: charger
  ocpp_key: MeterValueSampleInterval
  value: "15"
```

Danach kann man prüfen:

```yaml
action: ocpp.get_configuration
data:
  devid: charger
  ocpp_key: MeterValuesSampledData
```

## Warum Phasenumschaltung nicht direkt über OCPP gelöst wurde

Die Wallbox meldete zwar OCPP-Smart-Charging-Unterstützung, aber keine direkt nutzbare Home-Assistant-Entity für 1p/3p.

Ein OCPP-Config-Check zeigte unter anderem:

```text
ConnectorSwitch3to1PhaseSupported: false
PVChargingMode: 0
CurrentSelection: 15
SupportedFeatureProfiles: Core, FirmwareManagement, LocalAuthListManagement, Reservation, SmartCharging, RemoteTrigger
```

Ein Versuch, `ConnectorSwitch3to1PhaseSupported` per `ocpp.configure` auf `true` zu setzen, wurde nicht wirksam übernommen. Daher ist die lokale Elli-API der bessere Weg.


## PV-only statt nur „kein Netzbezug“

„Kein Netzbezug“ ist für PV-Überschussladen nicht automatisch ausreichend. Wenn eine Hausbatterie vorhanden ist, kann diese das Auto stützen; der Netzbezug bleibt dann bei 0 W, obwohl das Auto indirekt aus der Batterie lädt.

Besser ist daher eine PV-only-Regel über die Batterieleistung. In der hier getesteten GoodWe-Integration gilt:

```text
sensor.battery_power < 0 W   Batterie lädt
sensor.battery_power > 0 W   Batterie entlädt
```

Das Vorzeichen muss pro Installation geprüft werden.

Bewährte Grundlogik:

```text
Start erlauben: Batterie lädt deutlich, z. B. battery_power < -1500 W
Stop erzwingen: Batterie entlädt, z. B. battery_power > 100 W
Stop erzwingen: Netzbezug > 250 W
Stop erzwingen: Batterie-SoC < 30 %
3p nur erlauben: sehr viel Überschuss, z. B. battery_power < -5000 W und SoC >= 40 %
```

Damit wird nicht nur Netzbezug vermieden, sondern auch das Leersaugen der Hausbatterie.

## Lokale Elli-WebGUI/API

Die Wallbox-WebGUI läuft lokal auf:

```text
http://<wallbox-ip>/
```

Die Web-App nutzt eine REST-API unter:

```text
http://<wallbox-ip>/api/v2/...
```

Wichtige Endpunkte:

```text
POST /api/v2/jwt/login
GET  /api/v2/system/relais-switch/enabled
GET  /api/v2/system/relais-switch/state
PUT  /api/v2/system/relais-switch/state
GET  /api/v2/charging/phase-type
GET  /api/v2/charging/phase-type/selected
GET  /api/v2/charging/max-current-per-phase
GET  /api/v2/charging/max-current-per-phase/selected
```

Der relevante operative Umschalter ist:

```text
/api/v2/system/relais-switch/state
```

Nicht verwechseln mit:

```text
/api/v2/charging/phase-type/selected
```

Dieser Setup-Wert kann anders aussehen als der aktuelle Relaiszustand.

## Login gegen die lokale Wallbox-API

Der Login erfolgt als Form-POST:

```bash
curl -i -s \
  -X POST \
  -H "Content-Type: application/x-www-form-urlencoded" \
  --data-urlencode "user=technician" \
  --data-urlencode "pass=<SERVICE_PASSWORT>" \
  "http://<wallbox-ip>/api/v2/jwt/login"
```

Bei Erfolg liefert die Wallbox ein JWT im HTTP-Header:

```text
Authorization: <JWT>
```

Für Service-/Technikerzugriff ist der User:

```text
technician
```

Der normale Benutzer ist:

```text
user
```

## Phasenstatus lesen

```bash
curl -s \
  -H "Authorization: <JWT>" \
  "http://<wallbox-ip>/api/v2/system/relais-switch/state"
```

Beispielantwort:

```json
{
  "value": "threePhase"
}
```

Mögliche Werte:

```text
onePhase
threePhase
switchInProgress
notAvailable
```

Ob die Umschaltung verfügbar ist:

```bash
curl -s \
  -H "Authorization: <JWT>" \
  "http://<wallbox-ip>/api/v2/system/relais-switch/enabled"
```

Beispiel:

```json
{
  "enabled": true
}
```

## Phase setzen

1-phasig:

```bash
curl -s \
  -X PUT \
  -H "Authorization: <JWT>" \
  -H "Content-Type: application/json" \
  -d '{"value":"onePhase"}' \
  "http://<wallbox-ip>/api/v2/system/relais-switch/state"
```

3-phasig:

```bash
curl -s \
  -X PUT \
  -H "Authorization: <JWT>" \
  -H "Content-Type: application/json" \
  -d '{"value":"threePhase"}' \
  "http://<wallbox-ip>/api/v2/system/relais-switch/state"
```

Wichtig: In Automationen sollte man nicht hart während aktiver Ladung umschalten. Sicherer Ablauf:

1. Ladevorgang stoppen
2. Phase setzen
3. Status pollen, bis `onePhase` oder `threePhase` wirklich erreicht ist
4. Ladestromlimit setzen
5. Ladevorgang wieder starten

Die Elli-API kann beim `PUT /system/relais-switch/state` während des Umschaltens HTTP `422` zurückgeben, obwohl der Relaiswechsel anschließend weiterläuft. Direkt danach kann `/system/relais-switch/state` den Zwischenzustand `switchInProgress` melden. Deshalb sollte ein Script nicht allein den HTTP-Code des `PUT` als finale Wahrheit verwenden, sondern danach den Status-Endpunkt pollen.

## Beispiel: Home-Assistant-Script für lokale API

Beispielpfad:

```text
/config/wallbox_phase.sh
```

Beispielinhalt:

```sh
#!/bin/sh
set -eu

BASE="http://<wallbox-ip>"
PW_FILE="/config/.wallbox_service_password"
USER_ROLE="technician"
PHASE="${1:-state}"

TOKEN=$(curl -i -s --max-time 8 -X POST \
  -H "Content-Type: application/x-www-form-urlencoded" \
  --data-urlencode "user=$USER_ROLE" \
  --data-urlencode "pass=$(cat "$PW_FILE")" \
  "$BASE/api/v2/jwt/login" \
  | awk 'BEGIN{IGNORECASE=1} /^authorization:/{sub(/^[^:]+:[[:space:]]*/,""); sub(/\r$/,""); print; exit}')

if [ -z "$TOKEN" ]; then
  echo "wallbox login failed" >&2
  exit 2
fi

get_state() {
  curl -fsS --max-time 8 \
    -H "Authorization: $TOKEN" \
    "$BASE/api/v2/system/relais-switch/state"
}

case "$PHASE" in
  state)
    curl -fsS --max-time 8 \
      -H "Authorization: $TOKEN" \
      "$BASE/api/v2/system/relais-switch/state"
    ;;
  onePhase|threePhase)
    current=$(get_state | /usr/local/bin/python3 -c 'import sys,json; print(json.load(sys.stdin).get("value","unknown"))')
    if [ "$current" = "$PHASE" ]; then
      echo "{\"value\":\"$current\"}"
      exit 0
    fi

    tmp=/tmp/wallbox_phase_response.$$
    code=$(curl -s --max-time 12 -o "$tmp" -w '%{http_code}' -X PUT \
      -H "Authorization: $TOKEN" \
      -H "Content-Type: application/json" \
      -d "{\"value\":\"$PHASE\"}" \
      "$BASE/api/v2/system/relais-switch/state" || true)

    # Elli may return 422 while the relay transition is already in progress.
    # Poll the state endpoint and treat that as authoritative.
    i=0
    while [ $i -lt 12 ]; do
      sleep 5
      current=$(get_state | /usr/local/bin/python3 -c 'import sys,json; print(json.load(sys.stdin).get("value","unknown"))' || echo unknown)
      if [ "$current" = "$PHASE" ]; then
        echo "{\"value\":\"$current\"}"
        rm -f "$tmp"
        exit 0
      fi
      i=$((i+1))
    done

    echo "phase switch to $PHASE not completed; http=$code; current=$current; body=$(cat "$tmp" 2>/dev/null)" >&2
    rm -f "$tmp"
    exit 22
    ;;
  *)
    echo "usage: $0 state|onePhase|threePhase" >&2
    exit 64
    ;;
esac
```

Das Servicepasswort liegt separat:

```text
/config/.wallbox_service_password
```

Dateirechte:

```bash
chmod 600 /config/.wallbox_service_password
chmod 700 /config/wallbox_phase.sh
```

## Home Assistant: shell_command und Sensor

In `configuration.yaml`:

```yaml
shell_command:
  wallbox_phase_1p: "/config/wallbox_phase.sh onePhase"
  wallbox_phase_3p: "/config/wallbox_phase.sh threePhase"

command_line:
  - sensor:
      name: Wallbox Phase State
      unique_id: wallbox_phase_state
      command: >-
        /config/wallbox_phase.sh state | /usr/local/bin/python3 -c "import sys,json; print(json.load(sys.stdin).get('value','unknown'))"
      scan_interval: 60
```

Danach Home Assistant komplett neu starten. Ein Reload der OCPP-Integration reicht dafür nicht, weil `shell_command` und `command_line` globale HA-Konfiguration sind.

Verfügbare HA-Services:

```text
shell_command.wallbox_phase_1p
shell_command.wallbox_phase_3p
```

Statussensor:

```text
sensor.wallbox_phase_state
```

Dieser Sensor gehört nicht zur OCPP-Integration und erscheint daher nicht zwingend am OCPP-Gerät. Er ist ein `command_line`-Sensor.

## Beispiel-Automation: PV-Überschussladen mit Batterieschutz

Die konkrete Logik muss an die eigenen Sensoren angepasst werden. Dieses Beispiel nutzt:

```text
sensor.strombezug
sensor.aktueller_gesamtverbrauch
sensor.pv_power
sensor.battery_state_of_charge
sensor.charger_power_active_import
sensor.charger_status_connector
switch.charger_charge_control
sensor.wallbox_phase_state
```

Grundidee:

- Stoppen bei Netzbezug oder Batterie unter 30 %.
- Starten ab Batterie 30 %, wenn die Batterie bereits deutlich lädt.
- Stoppen, sobald die Batterie entlädt oder Netzbezug entsteht.
- 1-phasig bei wenig verfügbarem PV-Überschuss, z. B. rund 1,4-3,7 kW.
- 3-phasig nur bei sehr viel PV-Überschuss, z. B. Batterie lädt mit >5 kW und SoC >= 40 %.
- Umschalten nur über Stop → Phase setzen → bestätigten Phasenstatus abwarten → Start.
- Wenn der gewünschte Phasenstatus nicht bestätigt wird, nicht starten.

Beispiel-Entscheidung:

```jinja2
{% set pv = states('sensor.pv_power') | float(0) %}
{% set gesamt = states('sensor.aktueller_gesamtverbrauch') | float(0) %}
{% set ladeleistung = (states('sensor.charger_power_active_import') | float(0)) * 1000 %}
{% set verfuegbar_fuer_auto = [0, pv - gesamt + ladeleistung] | max %}
{{ 'threePhase' if verfuegbar_fuer_auto >= 5200 and states('sensor.battery_state_of_charge') | float(0) >= 40 else 'onePhase' }}
```

Warum `ladeleistung` wieder addieren? `aktueller_gesamtverbrauch` enthält in vielen Installationen bereits die Wallbox. Für die Frage „wie viel wäre fürs Auto verfügbar?“ muss die aktuelle Autolast daher wieder herausgerechnet werden.


## Wichtiger Automationshinweis: Phasenwechsel und Start trennen

Ein robuster Ablauf ist zweistufig:

1. Wenn PV-only-Laden erlaubt wäre, aber die Wallbox noch nicht in der Zielphase ist, nur die Phase anfordern und dann beenden.
2. Erst in einem späteren Lauf starten, wenn `sensor.wallbox_phase_state` wirklich `onePhase` oder `threePhase` passend zur Zielphase meldet.

Dadurch startet die Wallbox nicht versehentlich 3-phasig, wenn der 1p-Umschaltbefehl von der Elli mit `422` abgelehnt wurde.

Beispielbedingungen:

```yaml
# Phase anfordern, aber noch nicht starten
- condition: template
  value_template: >-
    {{ charge_switch == 'off'
       and fahrzeug_angesteckt
       and netzbezug_w < 50
       and batterie_soc >= 30
       and batterie_power_w < -1500
       and wallbox_phase != zielphase }}
```

```yaml
# Nur starten, wenn die Zielphase bestätigt ist
- condition: template
  value_template: >-
    {{ charge_switch == 'off'
       and fahrzeug_angesteckt
       and netzbezug_w < 50
       and batterie_soc >= 30
       and batterie_power_w < -1500
       and wallbox_phase == zielphase }}
```

## Typische Schwellwerte

Sehr grobe Orientierung:

- 1-phasig 6 A: ca. 1,4 kW
- 1-phasig 16 A: ca. 3,7 kW
- 3-phasig 6 A: ca. 4,1 kW
- 3-phasig 16 A: ca. 11 kW

Für PV-Überschussladen ist 1-phasig wichtig, weil 3-phasig bei 6 A schon rund 4,1 kW Mindestleistung benötigt.

## Debugging

OCPP-Konfiguration lesen:

```yaml
action: ocpp.get_configuration
data:
  devid: charger
```

Ladestrom per OCPP begrenzen:

```yaml
action: ocpp.set_charge_rate
data:
  devid: charger
  limit_amps: 6
```

HA-Services prüfen:

```text
Entwicklerwerkzeuge → Aktionen
```

Suchen nach:

```text
shell_command.wallbox_phase_1p
shell_command.wallbox_phase_3p
ocpp.set_charge_rate
```

HA-States prüfen:

```text
Entwicklerwerkzeuge → Zustände
```

Suchen nach:

```text
sensor.wallbox_phase_state
sensor.charger_status_connector
switch.charger_charge_control
```

## Häufige Stolperfallen

- Der Phasensensor ist kein OCPP-Sensor.
- Ein Reload der OCPP-Integration lädt `command_line` und `shell_command` nicht neu.
- Die lokale API braucht ein JWT aus dem WebGUI-Service-Login.
- Das WLAN-Passwort der Wallbox ist nicht das Service-/Technikerpasswort.
- `/charging/phase-type/selected` ist nicht zwingend der aktuelle Relaiszustand.
- Der operative Live-Zustand steht unter `/system/relais-switch/state`.
- Während aktiver Ladung sollte nicht direkt umgeschaltet werden.
- Home Assistant `shell_command` signalisiert Fehler nicht automatisch als Abbruch für die restliche Automation. Deshalb sollte eine Automation Laden und Phasenwechsel trennen: erst Phase anfordern, später nur starten, wenn `sensor.wallbox_phase_state` die Zielphase bestätigt.
- Phasenwechsel nicht blind alle 30 Sekunden erneut versuchen. Bei Elli kann ein blockierter 1p-Wechsel sonst Log-Spam und Timeout-Spam erzeugen. Besser nur bei Status-/Überschussereignissen erneut versuchen.


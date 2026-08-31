# sprinter_hvac_control

Steuert das originale HVAC-Gebläse des Sprinter wahlweise über die
Aufbaubatterie statt über den fahrzeugeigenen Controller. Zusätzlich gibt es
einen ESPHome-Webserver und native HomeAssistant-Anbindung.

Verwendet einen IBT-2-PWM-Treiber (BTS7960) plus ein
`ESP32_Relay_30A_X2_V1.1`-Board. Das ESP32-Board steuert sowohl die beiden
Relais als auch den IBT-2 selbst - die Steuerung ist damit zentral an einer
Stelle, Original-Controller und IBT-2 können sich nie gegenseitig
beeinflussen.

## Funktionsprinzip

Es gibt genau eine Bedien-Entity in HomeAssistant/Web-UI: **"HVAC Fan
(Battery)"** (Fan-Entity mit 3 Stufen).

- **OFF (Default/Fail-Safe):** Beide Relais unbestromt → Original-Controller
  ist an das Gebläse angeschlossen und funktioniert wie ab Werk. Der IBT-2
  ist komplett vom Motor getrennt und softwareseitig deaktiviert (`R_EN`
  low). Dieser Zustand ist auch aktiv, wenn der ESP32 keinen Strom hat,
  abgestürzt ist oder gerade bootet.
- **ON:** Relais legen die Motorleitungen auf den IBT-2 um, der IBT-2 wird
  scharf geschaltet (`R_EN` high) und die gewählte Drehzahlstufe wird per
  PWM (`RPWM`) gefahren.

Die Umschaltreihenfolge ist immer so, dass PWM-Signal und H-Brücken-Enable
nie aktiv sind, während die Relais gerade umschalten, und die Relais nie auf
Batterie stehen, ohne dass der IBT-2 danach auch wirklich freigegeben wird -
und umgekehrt beim Ausschalten zuerst PWM/Enable weg, dann erst die Relais
zurück auf Original. Siehe `relay-2ch-hvac.yaml` für die genaue Sequenz.

## Hardware

- **ESP32_Relay_30A_X2_V1.1** (Fotos in `information/`): ESP32-32E-Modul,
  2x Songle SLA-05VDC-SL-C Wechsler-Relais (30A/240VAC bzw. 30A/28VDC),
  7-28V-Eingang mit Buck-Regler auf 5V.
- **IBT-2 / BTS7960** Motortreiber, versorgt aus der Aufbaubatterie.

### Pinbelegung ESP32_Relay_30A_X2_V1.1

| Signal              | GPIO | Funktion |
|---------------------|------|----------|
| Onboard-LED          | G5   | Status-LED des Relay-Boards |
| Relay 1               | G12  | schaltet eine Motorleitung des Gebläses |
| Relay 2               | G13  | schaltet die zweite Motorleitung des Gebläses |
| IBT-2 `RPWM`          | G4   | PWM-Drehzahlsignal (20 kHz) |
| IBT-2 `R_EN`          | G16  | Software-Interlock: H-Brücke erst scharf, wenn Relais auf Batterie stehen |

### IBT-2-Verdrahtung

- `RPWM` → GPIO4 (ESP32-Board)
- `R_EN` → GPIO16 (ESP32-Board)
- `L_EN` → fest auf 5V (permanent aktiv)
- `LPWM` → fest auf GND
  (Wir nutzen den IBT-2 nur einkanalig, da das Gebläse nicht reversiert
  werden muss - `L_EN`/`LPWM` treiben dadurch nie einen Strom.)
- `VCC` (Logik) → 5V, `GND` → gemeinsame Masse mit ESP32-Board
- `B+`/`B-` → Aufbaubatterie (abgesichert!)
- `M+`/`M-` → auf die Wechsler-Kontakte (NO) von Relay 1/2

### Relais-Verdrahtung (pro Relais)

- **COM** → Motorleitung zum Gebläse
- **NC** (unbestromt) → Original-Sprinter-Controller
- **NO** (bestromt) → IBT-2 `M+`/`M-`

Beide Relais schalten **immer gemeinsam** beide Motorleitungen um (siehe
`fan_source_battery` in `relay-2ch-hvac.yaml`), damit Original-Controller und
IBT-2 nie gleichzeitig an einer der beiden Leitungen hängen können.

## ESPHome-Konfiguration

- `relay-2ch-hvac.yaml` - Board-spezifische Konfiguration (Relais, IBT-2,
  Fan-Entity)
- `.basics.yaml` - gemeinsame Basis (WLAN, API, OTA, Web-Server, Watchdog);
  wird per `packages:` eingebunden
- `secrets.yaml.example` - Vorlage für `secrets.yaml` (WLAN-Zugangsdaten,
  API-Key, Passwörter). Kopieren nach `secrets.yaml` und echte Werte
  eintragen - diese Datei wird nicht committet (`.gitignore`).

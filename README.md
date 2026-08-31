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
(Battery)"** - ein Dropdown (`select`) mit den Optionen `Off` / `Low` /
`Medium` / `High`.

> Technisch als `select` statt als ESPHome `fan:`-Domain umgesetzt: die
> aktuelle Template-Fan-Komponente von ESPHome ist trigger-/publish-state-
> basiert (Zustand wird optimistisch gesetzt, *danach* feuern
> `on_turn_on`/`on_speed_set`) und hat bekannte Reihenfolge-Probleme
> zwischen Ein-/Ausschalten und Stufe setzen. Für eine sicherheitsrelevante
> Umschaltung wollen wir die exakte Sequenz selbst in der Hand haben -
> `select` mit `set_action` ist dafür die robustere, seit Jahren stabile
> Wahl. Für HA fühlt sich das identisch an: ein Bedienelement.

- **Off (Default/Fail-Safe):** Beide Relais unbestromt → Original-Controller
  ist an das Gebläse angeschlossen und funktioniert wie ab Werk. Der IBT-2
  ist komplett vom Motor getrennt und softwareseitig deaktiviert (`R_EN`/
  `L_EN` low). Dieser Zustand ist auch aktiv, wenn der ESP32 keinen Strom
  hat, abgestürzt ist oder gerade bootet.
- **Low/Medium/High:** Relais legen die Motorleitungen auf den IBT-2 um,
  der aktive Kanal (`R_EN`+`RPWM` oder `L_EN`+`LPWM`, siehe unten) wird
  scharf geschaltet und die gewählte Stufe per PWM gefahren.

Die Umschaltreihenfolge ist immer so, dass PWM-Signal und H-Brücken-Enable
nie aktiv sind, während die Relais gerade umschalten, und die Relais nie auf
Batterie stehen, ohne dass der IBT-2 danach auch wirklich freigegeben wird -
und umgekehrt beim Ausschalten zuerst PWM/Enable weg, dann erst die Relais
zurück auf Original. Siehe `script.ibt2_engage`/`script.ibt2_disengage` in
`relay-2ch-hvac.yaml` für die genaue Sequenz.

### Zündungs-/D+-Interlock

Zusätzlich zur manuellen Auswahl gibt es ein Hardware-Interlock über ein
Zündungs-/Laufsignal (D+ vom Lichtmaschinen-Anschluss, oder z.B. Klemme 15 -
je nachdem, was am Fahrzeug leicht zugänglich ist):

- **Zündung/Motor an → sofort zurück auf Original-Controller.** Sobald das
  D+-Signal aktiv wird, schaltet die Software augenblicklich auf `Off`
  zurück (`binary_sensor.ignition_active`, `on_press`) - unabhängig davon,
  was gerade in HA eingestellt war. Kein Warten, keine Ausnahme.
- **Zündung/Motor aus → Nachlauf-Timer, dann erst Freigabe.** Der
  Original-Gebläsecontroller kann nach dem Abstellen noch kurz nachlaufen.
  Erst `ignition_off_delay` (Default: 5 Minuten, oben in `substitutions:`
  anpassbar) nach Zündung AUS wird der Batteriebetrieb überhaupt wieder
  freigegeben (`battery_mode_allowed`). Ein Auswahlversuch davor wird
  abgelehnt und im Log vermerkt.

**D+-Sense-Eingang (GPIO34):** D+ liegt auf Fahrzeugspannung (12-14V+, beim
Laden ggf. Spannungsspitzen) - das darf **niemals** direkt an einen ESP32-
GPIO (max. 3.3V). Vor GPIO34 gehört ein Spannungsteiler, der bei aktivem D+
sauber auf ~3.3V teilt (z.B. 10kΩ oben / 3.3kΩ unten gegen GND, plus eine
3.3V-Z-Diode oder TVS parallel zum unteren Widerstand als Spitzenschutz),
oder alternativ ein kleiner Optokoppler. GPIO34 ist bewusst gewählt, weil es
ein reiner Input-Pin ist (kein internes Pull-up/down, aber unkritisch für
Boot-Strapping) und ohnehin nur digital gelesen wird.

## Hardware

- **ESP32_Relay_30A_X2_V1.1** (Fotos in `information/`): ESP32-32E-Modul,
  2x Songle SLA-05VDC-SL-C Wechsler-Relais (30A/240VAC bzw. 30A/28VDC),
  7-28V-Eingang mit Buck-Regler auf 5V.
- **IBT-2 / BTS7960** Motortreiber, versorgt aus der Aufbaubatterie.

### Pinbelegung ESP32_Relay_30A_X2_V1.1

Alle GPIOs unten sind laut Bottom-Silkscreen (`information/*_Bottom.jpg`)
auf JP1/JP2 herausgeführt und mit keinem anderen Onboard-Verbraucher belegt.

| Signal                    | GPIO | Funktion |
|---------------------------|------|----------|
| Onboard-LED                | G5   | Status-LED des Relay-Boards |
| Relay 1                     | G12  | schaltet eine Motorleitung des Gebläses |
| Relay 2                     | G13  | schaltet die zweite Motorleitung des Gebläses |
| IBT-2 `RPWM` (Kanal R)      | G4   | PWM-Drehzahlsignal Kanal R (20 kHz) |
| IBT-2 `R_EN` (Kanal R)      | G16  | Software-Interlock Kanal R |
| IBT-2 `LPWM` (Kanal L)      | G17  | PWM-Drehzahlsignal Kanal L (20 kHz) |
| IBT-2 `L_EN` (Kanal L)      | G18  | Software-Interlock Kanal L |
| Zündung/D+ Sense            | G34  | erkennt Zündung/Motor an (siehe Spannungsteiler oben) |

Aktuell sind **beide** IBT-2-Kanäle (R und L) softwareseitig ansteuerbar,
weil noch nicht feststeht, welcher davon das Gebläse in die richtige
(Werks-)Richtung dreht. Über die zwei Test-Buttons in ESPHome/HA
(`IBT-2 Test: Kanal R/L`, 3s Testpuls bei 25%) einmal ausprobieren, welcher
Kanal korrekt dreht, dann in `relay-2ch-hvac.yaml` den `initial_value` von
`ibt2_use_channel_l` entsprechend setzen (`false` = Kanal R, `true` = Kanal
L). Danach kann der nicht benötigte Kanal (`L_EN`/`LPWM` bzw. `R_EN`/`RPWM`)
optional hardwareseitig fest verdrahtet und aus der Config entfernt werden -
Kanal R_EN/L_EN fest auf 5V, das zugehörige *PWM fest auf GND.

### IBT-2-Verdrahtung

- `RPWM` → GPIO4, `R_EN` → GPIO16 (ESP32-Board, Kanal R)
- `LPWM` → GPIO17, `L_EN` → GPIO18 (ESP32-Board, Kanal L)
  (beide Kanäle vorerst software-geschaltet, siehe oben - später ggf. den
  ungenutzten Kanal hardwired festlegen)
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

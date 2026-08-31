# sprinter_hvac_control

Runs the Sprinter's original HVAC blower off the leisure/house battery
instead of the vehicle's own controller. Also adds an ESPHome web server
and native HomeAssistant integration.

Uses an IBT-2 PWM driver (BTS7960) plus an `ESP32_Relay_30A_X2_V1.1` board.
The ESP32 board controls both the relays and the IBT-2 itself - control is
therefore centralized in one place, and the OEM controller and the IBT-2
can never interfere with each other.

## How it works

There is exactly one user-facing entity in HomeAssistant/the web UI:
**"HVAC Fan (Battery)"** - a dropdown (`select`) with the options `Off` /
`Low` / `Medium` / `High`.

> Implemented as a `select` rather than ESPHome's `fan:` domain on purpose:
> the current template fan component is trigger-/publish-state-based
> (state is set optimistically, *then* `on_turn_on`/`on_speed_set` fire)
> and has known ordering issues between turning on and setting speed. For a
> safety-relevant switch-over we want to own the exact sequence ourselves -
> `select` with `set_action` is the more robust, long-stable choice for
> that. From HA's point of view it feels the same: one control.

- **Off (default/fail-safe):** both relays de-energized → the OEM
  controller is connected to the blower and works exactly as from the
  factory. The IBT-2 is fully disconnected from the motor and disabled in
  software (`R_EN`/`L_EN` low). This is also the state whenever the ESP32
  has no power, has crashed, or is still booting.
- **Low/Medium/High:** relays switch the motor leads over to the IBT-2, the
  active channel (`R_EN`+`RPWM` or `L_EN`+`LPWM`, see below) is armed, and
  the selected level is driven via PWM.

The switch-over sequence always keeps the PWM signal and H-bridge enable
inactive while the relays are actually switching, and never leaves the
relays on battery without the IBT-2 actually being armed afterwards - and
the reverse on the way back: PWM/enable off first, then the relays return
to OEM. See `script.ibt2_engage`/`script.ibt2_disengage` in
`relay-2ch-hvac.yaml` for the exact sequence.

### Ignition/D+ interlock

On top of the manual selection there is a hardware interlock driven by an
ignition/running signal (D+ from the alternator, or e.g. terminal 15 -
whichever is easiest to tap on the vehicle):

- **Ignition/engine on → immediately back to the OEM controller.** As soon
  as the D+ signal goes active, the software instantly switches back to
  `Off` (`binary_sensor.ignition_active`, `on_press`) - regardless of
  whatever was selected in HA. No waiting, no exception.
- **Ignition/engine off → after-run timer, only then re-armed.** The OEM
  blower controller may keep running briefly after shutdown. Only
  `ignition_off_delay` (default: 5 minutes, adjustable in `substitutions:`
  at the top of `relay-2ch-hvac.yaml`) after ignition OFF does the system
  re-arm battery mode (`battery_mode_allowed`). A selection attempt before
  that is rejected and logged.

**D+ sense input (GPIO34):** D+ sits at vehicle voltage (12-14V+, possibly
higher spikes while charging) - that must **never** go directly into an
ESP32 GPIO (max. 3.3V). Implemented through a **PC817C** optocoupler
(galvanically isolated, no direct electrical reference needed between the
vehicle electrics and the ESP32 logic):

- **LED side** (pin 1 anode / pin 2 cathode): D+ → series resistor → pin 1,
  pin 2 → vehicle/chassis ground.
  Size the series resistor so LED current stays around ~10mA: at 13-15V
  system voltage, roughly **1.2kΩ, 1/2W** (`R = (V_D+ - 1.2V) / 0.01A`).
- **Transistor side** (pin 4 collector / pin 3 emitter): pin 3 → ESP32 GND,
  pin 4 → **10kΩ pull-up to 3.3V** AND → GPIO34. GPIO34 is an input-only
  pin with no internal pull-up, so this external pull-up is **mandatory** -
  without it the pin floats.
- This makes the pin logic active-low (the optocoupler pulls the collector
  toward GND while D+ is present) - already compensated in the YAML config
  via `inverted: true`, so `binary_sensor.ignition_active` still reports
  "on" when D+ is actually active.

> **Ready-made module instead of discrete parts:** a cheap off-the-shelf
> "2-channel PC817 optocoupler isolation module" (like the one you linked)
> already integrates exactly this circuit - the LED-side series resistor
> and (usually) the output-side pull-up, on a small screw-terminal board
> with `VCC` / `GND` / `IN` / `OUT` per channel. If you use one of those
> instead of a bare PC817C: `IN`/`GND` on that channel → D+ / vehicle
> ground, `VCC` → ESP32 3.3V, `OUT` → GPIO34. Two things you should verify
> on your specific module before wiring it to D+ (I couldn't fetch the
> Amazon listing from this environment to confirm them myself):
> 1. its onboard LED resistor is actually sized for a 12-14V input and not
>    only for 3.3-5V logic-to-logic isolation (most modules sold as
>    "isolation module" for microcontrollers are fine with automotive 12V
>    sensing - that's their most common use case - but check the listing);
> 2. whether its `OUT` is active-high or active-low with `VCC` tied to
>    3.3V - if it comes out active-high instead of the active-low behavior
>    assumed above, drop `inverted: true` from `binary_sensor.ignition_active`
>    in `relay-2ch-hvac.yaml`. Easiest way to be sure: apply 12V to `IN`
>    and measure `OUT` with a multimeter once before trusting it in the
>    interlock logic.

GPIO34 was chosen deliberately because it's a pure input pin (no
boot-strapping concerns) and is only ever read digitally here.

## Bench test setup

Before installing anything in the van, this is tested on a separate,
used blower + OEM controller assembly - not on the part actually fitted to
the vehicle. Relay wiring, IBT-2 channel selection (see the test buttons
below), and the D+ interlock can all be exercised safely on the bench
before anything is connected in the vehicle.

## Power supply

The 3-pin "7-28V GND 5V" terminal block on the relay board takes the input
voltage (here: 12V) and, via an onboard buck regulator, outputs regulated
5V from the same terminal block. That 5V rail powers the ESP32 module and
both relay coils (~70-90mA each) and should have comfortable headroom left
for the IBT-2's **logic supply** (`VCC`) too - its opto-isolators/driver IC
on the 5V logic side only draw a few mA, not a meaningful extra load.

> **Honesty check on the regulator identification:** I read the silkscreen
> off a slightly blurry photo ("...2596S" next to a 33µH inductor) and
> inferred the common LM2596/MP2596-style 3A buck-converter family from
> that - a websearch for the exact marking ("JM93MRP") turned up nothing,
> and I could not fetch a datasheet to confirm it. That's an educated guess
> from the visual topology (TO-263 regulator + inductor + electrolytic caps
> = standard non-isolated buck converter), **not** a verified fact. Please
> measure the 5V terminal with a multimeter under load (relays energized +
> IBT-2 logic connected) before relying on it, rather than trusting this
> identification.

Important: this only covers the IBT-2's **logic** supply. The actual motor
current (`B+`/`B-`/`M+`/`M-`) does **not** run through this regulator at
all - it goes straight from the leisure battery to the IBT-2 and from there
to the motor (see below) - that would be far too much current for the
small onboard regulator.

## Hardware

- **ESP32_Relay_30A_X2_V1.1** (photos in `information/`): ESP32-32E module,
  2x Songle SLA-05VDC-SL-C changeover relays (30A/240VAC resp. 30A/28VDC),
  7-28V input with a buck regulator down to 5V.
- **IBT-2 / BTS7960** motor driver, powered from the leisure battery.

### ESP32_Relay_30A_X2_V1.1 pinout

All GPIOs below are broken out on JP1/JP2 per the bottom silkscreen
(`information/*_Bottom.jpg`) and aren't used by any other onboard consumer.

| Signal                      | GPIO | Function |
|------------------------------|------|----------|
| Onboard LED                   | G5   | relay board status LED |
| Relay 1                        | G12  | switches one blower motor lead |
| Relay 2                        | G13  | switches the other blower motor lead |
| IBT-2 `RPWM` (channel R)       | G4   | channel R PWM speed signal (20 kHz) |
| IBT-2 `R_EN` (channel R)       | G16  | channel R software interlock |
| IBT-2 `LPWM` (channel L)       | G17  | channel L PWM speed signal (20 kHz) |
| IBT-2 `L_EN` (channel L)       | G18  | channel L software interlock |
| Ignition/D+ sense               | G34  | detects ignition/engine on (see optocoupler above) |

Right now **both** IBT-2 channels (R and L) are software-switchable,
because it isn't known yet which one spins the blower in the correct
(factory) direction. Use the two commissioning buttons in ESPHome/HA
(`IBT-2 Test: Channel R/L`, a 3s test pulse at 25%) to find out which
channel is correct, then set `ibt2_use_channel_l`'s `initial_value` in
`relay-2ch-hvac.yaml` accordingly (`false` = channel R, `true` = channel
L). After that, the unused channel (`L_EN`/`LPWM` resp. `R_EN`/`RPWM`) can
optionally be hardwired in hardware and removed from the config - the
unused channel's `*_EN` tied fixed to 5V, its `*PWM` tied fixed to GND.

### IBT-2 wiring

- `RPWM` → GPIO4, `R_EN` → GPIO16 (ESP32 board, channel R)
- `LPWM` → GPIO17, `L_EN` → GPIO18 (ESP32 board, channel L)
  (both channels software-switched for now, see above - optionally
  hardwire the unused one later once the correct direction is known)
- `VCC` (logic) → 5V, `GND` → common ground with the ESP32 board
- `B+`/`B-` → leisure battery (fused!)
- `M+`/`M-` → to the changeover (NO) contacts of Relay 1/2

### Relay wiring (per relay)

- **COM** → motor lead to the blower
- **NC** (de-energized) → OEM Sprinter controller
- **NO** (energized) → IBT-2 `M+`/`M-`

Both relays **always** switch both motor leads together (see
`fan_source_battery` in `relay-2ch-hvac.yaml`), so the OEM controller and
the IBT-2 can never both be connected to either lead at the same time.

## ESPHome configuration

- `relay-2ch-hvac.yaml` - board-specific config (relays, IBT-2, fan control
  entity)
- `.basics.yaml` - shared base config (WiFi, API, OTA, web server,
  watchdog); pulled in via `packages:`
- `secrets.yaml.example` - template for `secrets.yaml` (WiFi credentials,
  API key, passwords). Copy to `secrets.yaml` and fill in real values -
  that file is never committed (`.gitignore`).

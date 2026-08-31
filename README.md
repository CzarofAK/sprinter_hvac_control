# sprinter_hvac_control

Runs the Sprinter's original HVAC blower off the leisure/house battery
instead of the vehicle's own controller. Also adds an ESPHome web server
and native HomeAssistant integration.

Uses a Cytron MD30C PWM motor driver plus an `ESP32_Relay_30A_X2_V1.1`
board. The ESP32 board controls both the relays and the MD30C itself -
control is therefore centralized in one place, and the OEM controller and
the MD30C can never interfere with each other.

> **Hardware history:** this started out built around a generic IBT-2
> (BTS7960) H-bridge driver. That specific unit turned out to be dead on
> arrival (correct 3.3V logic signals measured at every input pin, but
> essentially no power reaching the motor output even unloaded) - a
> documented, apparently common failure mode for these cheap clone
> boards. Since the blower never needs to reverse anyway (fan blades only
> do anything useful in one direction), a full H-bridge was overkill to
> begin with; the MD30C is a simpler, single-direction-friendly,
> better-documented part sized to the blower's actual current draw. If
> you're starting fresh, there's no reason to go through the IBT-2 at
> all - go straight to the MD30C wiring below.

## How it works

There is exactly one user-facing entity in HomeAssistant/the web UI:
**"HVAC Fan (Battery)"** - a 0-100% slider (`number`) in steps of 10, so it
only ever lands on round numbers (0/10/20/.../100), not something fiddly
like 42% from a phone touchscreen.

> Implemented as a `number` rather than ESPHome's `fan:` domain on purpose:
> the current template fan component is trigger-/publish-state-based
> (state is set optimistically, *then* `on_turn_on`/`on_speed_set` fire)
> and has known ordering issues between turning on and setting speed. For a
> safety-relevant switch-over we want to own the exact sequence ourselves -
> `number` with `set_action` is the more robust, long-stable choice for
> that (an earlier revision used `select` with fixed Low/Medium/High
> options for the same reason - functionally identical, `number` just adds
> continuous PWM resolution instead of 3 fixed steps). From HA's point of
> view it's one control either way.

- **0% (default/fail-safe):** both relays de-energized → the OEM
  controller is connected to the blower and works exactly as from the
  factory. The MD30C's PWM input is held at 0% (no power to the motor).
  This is also the state whenever the ESP32 has no power, has crashed, or
  is still booting.
- **10-100%:** relays switch the motor leads over to the MD30C, and the
  selected percentage is driven via PWM duty cycle.

The switch-over sequence always keeps the PWM signal at 0% while the
relays are actually switching, and never leaves the relays on battery
without the PWM signal being set afterwards - and the reverse on the way
back: PWM to 0% first, then the relays return to OEM. See
`script.motor_engage`/`script.motor_disengage` in `relay-2ch-hvac.yaml`
for the exact sequence.

### Ignition/D+ interlock

On top of the manual selection there is a hardware interlock driven by an
ignition/running signal (D+ from the alternator, or e.g. terminal 15 -
whichever is easiest to tap on the vehicle):

- **Ignition/engine on → immediately back to the OEM controller.** As soon
  as the D+ signal goes active, the software instantly switches back to
  0% (`binary_sensor.ignition_active`, `on_press`) - regardless of
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
the vehicle. Relay wiring, the motor driver, and the D+ interlock can all
be exercised safely on the bench (see the "Motor Test" commissioning
button) before anything is connected in the vehicle.

## Power supply

The ESP32_Relay_30A_X2_V1.1's 3-pin "7-28V GND 5V" terminal block takes
the input voltage (here: 12V) and, via an onboard buck regulator, outputs
regulated 5V from the same terminal block. That 5V rail powers the ESP32
module and both relay coils (~70-90mA each) - nothing else needs to draw
from it.

> **Honesty check on the regulator identification:** I read the silkscreen
> off a slightly blurry photo ("...2596S" next to a 33µH inductor) and
> inferred the common LM2596/MP2596-style 3A buck-converter family from
> that - a websearch for the exact marking ("JM93MRP") turned up nothing,
> and I could not fetch a datasheet to confirm it. That's an educated guess
> from the visual topology (TO-263 regulator + inductor + electrolytic caps
> = standard non-isolated buck converter), **not** a verified fact. Please
> measure the 5V terminal with a multimeter under load (both relays
> energized) before relying on it, rather than trusting this
> identification.

The MD30C is **entirely separate** from that 5V rail - per its own user's
manual (Cytron, Rev 1.4), it has no dedicated logic-supply pin at all. Its
`POWER` terminal (5-30V, the same leisure-battery feed that also drives the
motor stage) is regulated down to logic voltage **internally on the MD30C
board itself**. The only thing connecting it to the ESP32 side is the 3-pin
signal header (`GND`/`PWM`/`DIR`) - and even `DIR` on that header isn't
used (see wiring below). So there's no headroom question to work out here
at all, unlike the earlier IBT-2 revision.

## Hardware

- **ESP32_Relay_30A_X2_V1.1** (photos in `information/`): ESP32-32E module,
  2x Songle SLA-05VDC-SL-C changeover relays (30A/240VAC resp. 30A/28VDC),
  7-28V input with a buck regulator down to 5V.
- **Cytron MD30C** motor driver (5-30V, 30A continuous/80A peak for up to
  1s, PWM+DIR logic interface, logic input HIGH = 3-5.5V / LOW = 0-0.5V
  per its datasheet - ESP32's 3.3V GPIO clears the 3V HIGH threshold with
  a bit of margin, not a lot), powered directly from the leisure battery.
  Sized against the blower's expected draw (OEM fuse in that circuit is
  typically 20-30A) with real headroom, unlike the 20A MD20A or the
  20A-continuous-despite-"30A"-branding generic MOSFET modules also
  considered.

### ESP32_Relay_30A_X2_V1.1 pinout

All GPIOs below are broken out on JP1/JP2 per the bottom silkscreen
(`information/*_Bottom.jpg`) and aren't used by any other onboard consumer.

| Signal              | GPIO | Function |
|----------------------|------|----------|
| Onboard LED            | G5   | relay board status LED |
| Relay 1                 | G12  | switches one blower motor lead |
| Relay 2                 | G13  | switches the other blower motor lead |
| MD30C `PWM`             | G4   | motor speed signal (20 kHz) |
| Ignition/D+ sense        | G34  | detects ignition/engine on (see optocoupler above) |

GPIO16/17/18 (used by the earlier IBT-2 revision for its second channel
and enable pins) are free/unused now - the MD30C only needs one PWM
signal from the ESP32.

### MD30C wiring

Per the Cytron MD30C user's manual (Rev 1.4):

- **Jumpers:** `JP4` = Don't Care (X), `JP6` = `EXT PWM` - this switches
  the board from its standalone onboard-potentiometer mode into
  microcontroller-controlled mode. Do this *after* the standalone bench
  check below, not before.
- **3-pin `INPUT` header** (`GND` / `PWM` / `DIR`, in that order):
  - `PWM` → GPIO4 on the ESP32 board.
  - `GND` → common ground with the ESP32 board (this is the logic
    reference ground for the signal header, separate from the heavy
    `POWER`/`MOTOR` terminal wiring below - tie both boards' grounds
    together somewhere, e.g. at the battery negative).
  - `DIR` → hardwired directly to `GND` (either right there on the
    header, or with a jumper wire) - **not** to a GPIO. Per the MD30C's
    truth table, `PWM=High, DIR=Low` drives Output A; the blower only
    ever needs one direction, so there's nothing to switch at runtime.
    If it spins the wrong way once wired up, swap the two motor leads at
    `MOTOR A`/`B` instead of touching `DIR` or the config - electrically
    identical, no reflash needed. Bonus: the manual explicitly warns to
    have `DIR` or `PWM` at LOW when power comes on - hardwiring `DIR` to
    GND satisfies that automatically, even before the ESP32 has booted.
- **`POWER` terminal** (`+`/`-`) → leisure battery, fused. This is the
  *only* power input the MD30C needs - it also derives its own logic
  supply from here internally, nothing extra required from the ESP32
  board's 5V rail.
- **`MOTOR` terminal** (`A`/`B`) → to the changeover (NO) contacts of
  Relay 1/2. ⚠️ The manual is explicit: **connecting the battery to the
  `MOTOR` terminal instead of `POWER` burns the MOSFETs, and that's not
  covered under warranty** - double-check before powering up.
- For current draw >20A (our expected range), the manual recommends
  soldering the wires directly to the pads on the PCB's bottom layer
  rather than relying only on the screw terminals.

**Bench-test the MD30C completely standalone before wiring it to the ESP32
at all:** temporarily set `JP4`=`INT POT`, `JP6`=`INT PWM`, connect
battery+motor, and use the onboard Test Button A/B (with the onboard
speed potentiometer) to confirm the board and motor actually work - no
microcontroller needed for this check. Only then switch the jumpers to
`JP4`=Don't Care, `JP6`=`EXT PWM` and wire up the ESP32 signal header.
This is exactly the standalone check the IBT-2 didn't give us, and would
have caught its failure immediately instead of after a long wiring-level
debugging session.

### Relay wiring (per relay)

- **COM** → motor lead to the blower
- **NC** (de-energized) → OEM Sprinter controller
- **NO** (energized) → MD30C `MOTOR A`/`B`

Both relays **always** switch both motor leads together (see
`fan_source_battery` in `relay-2ch-hvac.yaml`), so the OEM controller and
the MD30C can never both be connected to either lead at the same time.

## ESPHome configuration

- `relay-2ch-hvac.yaml` - board-specific config (relays, motor driver, fan
  control entity)
- `.basics.yaml` - shared base config (WiFi, API, OTA, web server,
  watchdog); pulled in via `packages:`
- `secrets.yaml.example` - template for `secrets.yaml` (WiFi credentials,
  API key, passwords). Copy to `secrets.yaml` and fill in real values -
  that file is never committed (`.gitignore`).

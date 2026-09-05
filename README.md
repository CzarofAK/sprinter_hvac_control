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

### Ignition / Terminal 15R interlock

On top of the manual selection there is a hardware interlock driven by a
vehicle running signal - in this build, Mercedes **Terminal 15R**
(switched positive, active only while the ignition is actually on - a
filtered variant of classic Terminal 15). An earlier revision used
**Terminal 30t** (a timed, relay-switched permanent-positive terminal)
instead; that turned out to stay active 15-30+ minutes after the vehicle
is actually parked (by design, unrelated to the OEM blower's own
after-run), which made the plain after-run timer impractical for normal
use on its own - see `switch.battery_mode_override` below, added to work
around that. Terminal 15R drops essentially immediately at ignition-off,
so that workaround usually isn't needed anymore - it's kept as an
optional manual bypass.

- **Ignition on → immediately back to the OEM controller.** As
  soon as the signal goes active, the software instantly switches back to
  0% (`binary_sensor.ignition_active`, `on_press`) - regardless of
  whatever was selected in HA. No waiting, no exception, and this cannot
  be bypassed by the override below.
- **Ignition off → after-run timer, only then re-armed.** The OEM
  blower controller may keep running briefly after shutdown. Only
  `ignition_off_delay` (default: 5 minutes, adjustable in `substitutions:`
  at the top of `relay-2ch-hvac.yaml`) after the ignition signal goes
  inactive does the system re-arm battery mode (`battery_mode_allowed`).
  A selection attempt before that is rejected and logged.

**Manual override (`switch.battery_mode_override`):** lets you arm
battery mode immediately without waiting out `ignition_off_delay` at all
- useful for testing, or if you're certain it's safe to skip the wait.
It does not weaken the core safety guarantee: the instant the ignition
signal goes active again (real driving resumes), the interlock above
forces everything back to the OEM controller *and* turns this override
back off unconditionally - so re-arming after the next stop is always a
fresh, deliberate choice, never a forgotten switch left on from before.

**Terminal 15R sense input (GPIO25):** Terminal 15R sits at vehicle voltage
(12-14V+, possibly higher spikes while charging) - that must **never** go directly into an
ESP32 GPIO (max. 3.3V). Implemented through a 2-channel, EL817-based opto
isolation module (galvanically isolated, no direct electrical reference
needed between the vehicle electrics and the ESP32 logic). Per channel it
has 5 pins - `IVCC` / `SIN1` / `VO` / `OUT1` / `OGND` - with the LED-side
series resistor (`R1`, 470Ω) and the output-side pull-up (`R2`, 10kΩ)
already built onto the module, so no discrete parts needed:

- `IVCC` → Terminal 15R, directly, no external resistor - `R1` (470Ω) is
  already on the module. LED current at typical automotive voltages:
  `(V_15R - 1.2V) / 470Ω` ≈ 25mA at 13V, ≈ 29mA at 15V - comfortably under
  the module's 50mA max LED current rating.
- `SIN1` → vehicle/chassis ground.
- `VO` → GPIO27 (`switch.opto_vcc` in `relay-2ch-hvac.yaml`, held
  permanently high in software as a stand-in 3.3V source - see "Powering
  the opto module's VO" below). Load here is only
  `3.3V / 10kΩ` ≈ 0.33mA through the module's onboard `R2` - unrelated to
  the 50mA LED-side rating, and trivial for a GPIO.
- `OUT1` → GPIO25. No external pull-up needed - the module's onboard `R2`
  already does that job once `VO` is powered.
- `OGND` → common ground with the ESP32 board.

This makes the pin logic active-low (`R2` holds `OUT1` high at rest; the
phototransistor pulls it toward `OGND` while D+ is present) - already
compensated in the YAML config via `inverted: true`, so
`binary_sensor.ignition_active` still reports "on" when D+ is actually
active.

GPIO25/GPIO27 sit on JP1's outer row (board-edge side), moved there from
JP2 to free up space next to the MD30C signals. **Verify the physical
position with a multimeter before soldering anything permanent** (toggle
each from ESPHome/HA, probe the pin you think it is) rather than trusting
the header layout derived from mirrored reference photos - that same
reasoning got one pin wrong earlier in this project (see the RPWM/GPIO0
mixup during IBT-2 commissioning), so it's not a one-off precaution.

### Powering the opto module's VO

Neither the ESP32_Relay_30A_X2_V1.1 nor the MD30C breaks out a spare
3.3V pin, so `VO` is powered from **GPIO27 held permanently high**
(`switch.opto_vcc`, `restore_mode: ALWAYS_ON`) instead of a dedicated
3.3V rail. This works cleanly here because the load is tiny (≈0.33mA,
see above) - nowhere near a GPIO's ~20mA safe continuous rating - but two
things are worth knowing:

- A GPIO used this way has no current-limiting/short-circuit protection
  of its own, unlike a proper regulated rail. Fine for a clean,
  low-current load like this; not something to reuse for anything with
  real current draw.
- `restore_mode: ALWAYS_ON` (the opposite of the fail-safe `ALWAYS_OFF`
  used elsewhere in this config) - it just needs to stay high
  permanently. If it's briefly undefined for a moment during boot before
  ESPHome takes over the pin, the worst case is a momentarily-wrong D+
  reading, which only pushes `battery_mode_allowed` toward staying
  `false` longer - the safe direction, not the dangerous one.

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

## Measured current draw

Confirmed on the actual installed blower (real ductwork, not a free-air
bench test) via a Victron SmartShunt on the leisure battery. Baseline
camper load while idle was ~3.5A - subtract that from each reading below
to get the blower's own draw:

| PWM level | Total (incl. ~3.5A baseline) | Blower only |
|---|---|---|
| 0%   | 3.5A  | 0A |
| 10%  | 4A    | ~0.5A |
| 20%  | 4.2A  | ~0.7A |
| 30%  | 5A    | ~1.5A |
| 40%  | 6.2A  | ~2.7A |
| 50%  | 8A    | ~4.5A |
| 60%  | 11A   | ~7.5A |
| 70%  | 13.5A | ~10A |
| 80%  | 16.8A | ~13.3A |
| 90%  | 21A   | ~17.5A |
| 100% | 25.5A | ~22A |

At 100% the blower itself draws roughly 22A - right in the range the OEM
fuse rating (20-30A) had suggested from the start, and comfortably under
the MD30C's 30A continuous rating with real headroom to spare (~8A). No
need to revisit the driver sizing based on this.

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
| Opto module `VO`         | G27  | stand-in 3.3V source, held ALWAYS_ON (see "Powering the opto module's VO" above), JP1 outer row |
| Terminal 15R sense        | G25  | detects vehicle running (see optocoupler above), JP1 outer row |

GPIO16/17/18/34 (used by the earlier IBT-2 revision and an earlier
revision of the opto module wiring) are all free/unused now - the MD30C
only needs one PWM signal from the ESP32, and the opto module's two
signals moved to GPIO27/GPIO25 on JP1's outer row.

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

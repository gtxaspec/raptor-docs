# RIC — Day/Night Detection Design

## Problem

Cameras need to switch between day mode (IR-cut filter closed, IR LEDs
off) and night mode (IR-cut filter open, IR LEDs on) based on ambient
light. The challenges:

1. **Sensor diversity**: different sensors have different gain ranges
   (SC4336P maxes at ~20K, SC2335 at ~60K). Fixed gain thresholds
   don't work across devices.

2. **IR flip-flop**: when IR LEDs turn on, the scene becomes brighter
   (IR illumination). A naive luma threshold sees "bright scene" and
   switches back to day, turning off IR, making it dark again → loop.

3. **GPIO discovery**: different cameras have different GPIO pins for
   IR-cut and IR LEDs. Pins are defined in `/etc/thingino.json`.

---

## Algorithm

### Day → Night (ae_luma based, sensor-independent)

In day mode, no IR LEDs are on. `ae_luma` (0-255) directly reflects
ambient brightness after ISP auto-exposure convergence. This value is
sensor-independent — it's the ISP's target luminance measurement.

```
If ae_luma < night_luma (default 20) for hysteresis_sec (default 5)
consecutive polls → switch to NIGHT
```

### Night → Day (gain-ratio based, auto-calibrating)

In night mode, IR LEDs are on. `ae_luma` is unreliable because IR
illumination inflates it regardless of ambient light:

```
Dark room + IR on:   gain=16143, luma=54
Bright room + IR on: gain=1953,  luma=55  ← luma nearly identical!
```

However, `total_gain` reveals the truth. When real ambient light
returns (dawn, lights on), the ISP drops gain because the scene is
bright — even with IR LEDs contributing. The ratio is sensor-
independent because we compare against the same sensor's own baseline.

```
After switching to night:
  - 3-poll cooldown for IR/ISP stabilization
  - Sample total_gain as night_gain_baseline (e.g., 16143)
  - Compute day trigger = baseline × day_gain_pct / 100 (e.g., 4036)

If total_gain < day_trigger for hysteresis_sec → switch to DAY
```

Example with 25% threshold:
- Night mode, dark, IR on: gain=16143 > 4036 → stays night ✓
- Night mode, lights on: gain=1953 < 4036 → triggers day ✓
- Night mode, IR reflecting: gain=16143 > 4036 → stays night ✓

### Cooldown

After every mode switch, a 3-poll cooldown prevents evaluation while
the IR LEDs and ISP AE are stabilizing. In night mode, the gain
baseline is sampled on the last cooldown tick.

### State Machine

```
         ae_luma < 20 (5s)
  [DAY] ─────────────────────→ [NIGHT]
    ↑                              │
    │   gain < 25% baseline (5s)   │
    └──────────────────────────────┘
           3-poll cooldown
           after each switch
```

---

## GPIO Auto-Discovery

RIC reads GPIO pins from raptor.conf first. If left at -1 (default),
it falls back to `/etc/thingino.json`:

```json
{
  "gpio": {
    "ircut": "57 58",
    "ir850": 8,
    "ir940": 9
  }
}
```

Parsing rules:
- `ircut`: string `"57 58"` → dual GPIO (motor driver), `"57"` → single, integer `57` → single
- `ir850`: IR LED GPIO (preferred over ir940)
- `ir940`: IR LED GPIO (fallback if ir850 absent)

### GPIO Control

Single GPIO mode: pin HIGH = day (filter closed), LOW = night (filter open).

Dual GPIO mode (motor driver): pulse pin1/pin2 HIGH for 100ms, then both LOW.

IR LED: HIGH = on, LOW = off. Controlled via sysfs (`/sys/class/gpio`).

---

## Configuration

```ini
[ircut]
enabled = true
mode = auto                   # auto, day, or night
trigger = luma                # luma (sensor-independent) or gain (legacy)
night_luma = 20               # ae_luma below this → night (0-255)
day_gain_pct = 25             # night→day: gain below this % of baseline
night_threshold = 40000       # (trigger=gain only) gain → night
day_threshold = 25000         # (trigger=gain only) gain → day
hysteresis_sec = 5            # consecutive seconds before switching
poll_interval_ms = 1000       # exposure poll interval
gpio_ircut = -1               # -1 = auto from /etc/thingino.json
gpio_ircut2 = -1              # second pin for dual motor mode
gpio_irled = -1               # IR LED pin
```

### Runtime Control

```sh
raptorctl ric mode auto       # resume auto detection
raptorctl ric mode day        # force day mode
raptorctl ric mode night      # force night mode
raptorctl ric status          # show mode, state, exposure
```

---

## ADC Trigger Mode

RIC supports a third trigger mode using an external photoresistor
connected to the SU_ADC hardware. Unlike luma and gain modes, ADC
reads ambient light directly -- unaffected by IR LED illumination -- so
no flip-flop prevention or calibration is needed.

```ini
[ircut]
trigger = adc
adc_channel = 0
adc_night = 200     # ADC value below this → switch to night
adc_day = 600       # ADC value above this → switch to day
```

The ADC library (`libsysutils.so`) is loaded at runtime via `dlopen`.
If unavailable, RIC falls back to luma mode with a warning. The same
`hysteresis_sec` debounce applies to ADC transitions.

---

## Legacy Gain Mode

Setting `trigger = gain` uses fixed `total_gain` thresholds instead
of the hybrid algorithm. This is sensor-dependent and requires manual
tuning per device. Not recommended for new deployments.

---

## Files

- `ric/ric_main.c` — config loading, thingino.json GPIO discovery, main loop
- `ric/ric_daynight.c` — GPIO control, mode switching, hybrid detection algorithm
- `ric/ric.h` — state and config structures

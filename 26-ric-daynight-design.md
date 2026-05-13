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

Additionally: if total_gain > night_gain (default 80000), force NIGHT
immediately regardless of ae_luma. This handles sensors where high gain
pushes luma above the threshold despite near-darkness.
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
         OR gain > night_gain
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
- `ir850`: 850nm IR LED GPIO (mapped to `gpio_irled`)
- `ir940`: 940nm IR LED GPIO (mapped to `gpio_irled2`)

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
trigger = luma                # luma, gain, adc, or photo
night_luma = 20               # ae_luma below this → night (0-255)
night_gain = 80000            # force night when total_gain exceeds this (regardless of luma)
day_gain_pct = 25             # night→day: gain below this % of baseline
night_threshold = 40000       # (trigger=gain only) gain → night
day_threshold = 25000         # (trigger=gain only) gain → day
hysteresis_sec = 5            # consecutive seconds before switching
poll_interval_ms = 1000       # exposure poll interval
gpio_ircut = -1               # -1 = auto from /etc/thingino.json
gpio_ircut2 = -1              # second pin for dual motor mode
gpio_irled = -1               # 850nm IR LED pin (-1 = auto)
gpio_irled2 = -1              # 940nm IR LED pin (-1 = auto)
ir850 = true                  # enable 850nm LED (visible glow, stronger)
ir940 = false                 # enable 940nm LED (invisible, weaker)

# Photo trigger thresholds (trigger=photo only)
# EV direction: HIGH ev = dark (more exposure), LOW ev = bright
photo_ev_night = 50000        # EV above this → dark
photo_ev_deep = 150000        # EV above this → very dark
photo_ev_day = 5000           # EV below this → bright
photo_rgain_rec = 0           # R-gain baseline (0 = auto-calibrate)
photo_bgain_rec = 0           # B-gain baseline (0 = auto-calibrate)
```

### Runtime Control

```sh
raptorctl ric mode auto       # resume auto detection
raptorctl ric mode day        # force day mode
raptorctl ric mode night      # force night mode
raptorctl ric status          # show mode, state, exposure
raptorctl ric get-thresholds  # show all threshold values
raptorctl ric set-threshold night_luma 30    # change threshold at runtime
raptorctl ric set-threshold hysteresis_sec 3
```

Tunable keys: `night_luma` (0-255), `night_gain` (>=0),
`day_gain_pct` (1-100), `night_threshold` (>=0), `day_threshold`
(>=0), `hysteresis_sec` (1-300), `poll_interval_ms` (50-10000).
Changes take effect on the next poll cycle.

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

## Photo Trigger Mode

Setting `trigger = photo` enables a multi-stage detection algorithm
using ISP EV value and AWB white balance R/B gain statistics.

Photo mode uses its own state machine independent of the luma/gain
cooldown and hysteresis counters. Default poll interval is 100ms.

### EV Direction

On Ingenic, EV is the exposure product (integration time x gain):
**HIGH ev = dark** (ISP compensating with more exposure), **LOW ev =
bright**. All threshold comparisons follow this convention.

### Inputs

- **EV**: exposure value from `IMP_ISP_Tuning_GetEVAttr` (T20-T31).
  Range is sensor-dependent (0 to ~400K). Unlike `ae_luma` (0-255),
  EV preserves full dynamic range for threshold comparisons.

- **R-gain / B-gain**: AWB statistics from
  `IMP_ISP_Tuning_GetWB_Statis` (T20-T31). Under IR illumination
  the spectral composition shifts, causing R/B gains to deviate from
  their daylight baseline. This spectral signature distinguishes
  genuine darkness from IR-contaminated scenes.

### AWB Auto-Calibration

R/B gain baselines are auto-calibrated by default. On startup, RIC
collects 16 samples where EV indicates a bright scene (ev < ev_day)
and averages the R-gain and B-gain values. This baseline is used for
spectral shift detection in the night trigger.

Set `photo_rgain_rec` and `photo_bgain_rec` to nonzero values to
override auto-calibration with fixed baselines. The calibration is
preserved across phase resets (night/day/interfere transitions).

### State Machine

```
                    ev > ev_night + R/B gain shift (23+ polls)
                    OR fixed-EV drift (10x at 200-poll intervals)
  [NIGHT_DETECT] ──────────────────────────────────────→ [DAY_DETECT]
        ↑                                                     │
        │  interfere timeout (9000 polls)                     │
        │  OR genuine brightness (3x fall)                    │
        │                                                     │
  [INTERFERE] ←── EV ratio 0.87-1.13 of ref (3x confirm) ───┘
        │                                                     ↑
        │  false trigger (3x rise > 1.12x) → revert night    │
        └─────────────────────────────────────────────────────┘
```

### Night Detection (NIGHT_DETECT phase)

Runs when in day mode. Evaluates two independent trigger paths:

1. **R-gain path**: EV above `photo_ev_night` for 23+ consecutive
   polls AND R-gain deviates from the calibrated baseline by >=15
   and >=9 for 3+ consecutive polls each.

2. **B-gain path**: EV above `photo_ev_deep` for 23+ consecutive
   polls AND B-gain deviates from the calibrated baseline by >=10
   and >=7 for 7+ consecutive polls each.

Either path firing triggers the switch to night mode.

**Fixed-EV fallback**: if neither path fires but EV stays above
`photo_ev_deep` for 10 consecutive checks (sampled every 200 polls),
night mode is forced. Handles sensors whose gain characteristics
don't produce sufficient R/B deviation.

### Day Detection (DAY_DETECT phase)

Runs when in night mode. Collects 10 EV samples into a ring buffer
when EV drops below `photo_ev_day` (bright scene). Compares the
last sample against a reference using ratio thresholds:

- Sample >= reference x 1.13: too dark, reset ring.
- Sample within 0.87-1.13 of reference: moderate day signal,
  increment trigger counter. After 3 consecutive triggers, switch
  to day.

A separate fixed-drift detector (8-sample baseline, 0.70 ratio
threshold) catches slow transitions where EV gradually drops
(brightens) below the night baseline.

### Anti-Interference (INTERFERE phase)

Runs after a night-to-day transition. Monitors for false triggers
caused by transient light sources (headlights, flashlights):

- Collects an 8-sample EV baseline, validates stability (within
  0.92-1.08 ratio of first to last sample).
- Watches for EV rising above reference x 1.12 (scene darkening).
  Three consecutive rises revert to night mode.
- EV dropping below reference x 0.88 for three consecutive polls
  indicates genuine brightness -- transitions to NIGHT_DETECT.
- Times out after 9000 polls (~15 minutes at 100ms interval) and
  resets to NIGHT_DETECT.

### Tuning

The defaults work for most sensors without configuration. If
tuning is needed:

- **`photo_ev_night`** (default 50000): EV above this is considered
  dark enough for night detection. Lower values make night trigger
  more sensitive.
- **`photo_ev_deep`** (default 150000): EV above this forces night
  via the fixed-EV fallback. Handles sensors with weak R/B deviation.
- **`photo_ev_day`** (default 5000): EV below this starts day
  detection. Higher values detect day sooner.
- **AWB baselines**: leave at 0 for auto-calibration. Set fixed
  values only if the auto-calibrated baseline is consistently wrong
  for your sensor.

### Platform Support

Photo mode is supported on all Ingenic SoCs with ISP tuning:

- **T20, T21, T23, T30, T31** (Gen1/Gen2): uses
  `IMP_ISP_Tuning_GetEVAttr` for EV and
  `IMP_ISP_Tuning_GetWB_Statis` for R/B gains.

- **T32, T33, T40, T41** (Gen3/IMPVI): uses
  `IMP_ISP_Tuning_GetAeExprInfo` for EV (`.ExposureValue` field)
  and `IMP_ISP_Tuning_GetAwbGlobalStatistics` for R/B gains
  (`.statis_gol_gain.rgain/.bgain`).

A1 has no ISP and does not support photo mode.

---

## Legacy Gain Mode

Setting `trigger = gain` uses fixed `total_gain` thresholds instead
of the hybrid algorithm. This is sensor-dependent and requires manual
tuning per device. Not recommended for new deployments.

---

## Files

- `ric/ric_main.c` — config loading, thingino.json GPIO discovery, main loop
- `ric/ric_daynight.c` — GPIO control, mode switching, detection dispatch
- `ric/ric_photo.c` — multi-stage EV + AWB photo detection algorithm
- `ric/ric.h` — state, config, and photo mode structures

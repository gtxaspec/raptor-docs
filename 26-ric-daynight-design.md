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

`ae_luma` source differs by SDK generation: Gen1/2 report the
vendor's post-processing luma (`GetAeLuma`, or EV on T20/T21/T30);
Gen3 (T32/T33/T40/T41) has no direct luma API, so the HAL computes
the mean of the 256-bin AE histogram (`GetAeStatistics`). The
histogram is linear-domain, so Gen3 values run lower for the same
scene (bright indoor ~50 vs ~100+ on Gen1/2). The default threshold
of 20 works on both, but tuned thresholds do not transfer across
generations.

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
  - Clamp: baseline = max(sample, detection_gain / 10)
  - Compute day trigger = baseline × day_gain_pct / 100 (e.g., 4036)

While in night: baseline = max(baseline, total_gain) each poll.

If total_gain < day_trigger for hysteresis_sec → switch to DAY
```

The clamp and the running max exist because the point sample can be
poisoned: the IR LED bouncing off a covered lens (the test every user
runs) or a point-blank surface crashes AE during the cooldown, and a
baseline of 459 against a detection gain of 45025 makes the day
trigger unreachable -- the camera latches night forever. A baseline
reading far below the darkness that caused night is self-
contradictory, so it is clamped (with a WARN naming the likely
cause), and the running max adopts the scene's real dark reading if
it appears later.

### Day Verification

The clamp alone would trade the latch for an oscillator: the
reflection state itself (gain 459) is optically identical to a real
bright scene (377 in the same field case) -- no brightness metric
separates them. What separates them is what happens after the
switch: day turns the IR off, and a covered lens goes dark while a
real day stays bright. Every ratio-triggered day is therefore
verified on the last cooldown tick, with the IR already off: a
night-dark reading (ae_luma < night_luma) means the "day" was the
LED's own light, so ric reverts to night with a WARN and backs off
further attempts (30 polls doubling to 300, reset by any verified
day). A covered lens settles into stable night with a rare re-check
instead of blinking, and an uncovered scene passes verification on
the next attempt and holds.

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

If discovery finds a 940nm pin and no 850nm pin, and the config carries
no explicit `ir940` key, the 940 bank is enabled automatically -- the
opt-in default exists for boards where 940 is the secondary bank, and a
board whose only bank stayed dark sat blind at night with a clean log.
An explicit `ir940 = false` still wins.

Behavior (`ric/ric_json.c`, unit-tested in `tests/test_ric_json.c`):
- Only pins still at -1 are filled; raptor.conf stays authoritative.
- The whole file is read, bounded by a 64 KB cap; the file routinely
  exceeds 2 KB on thingino since several packages append to it.
- A missing file is silent (normal off thingino). A file that exists
  but cannot be used — unreadable, over-cap, empty, or unparseable —
  warns, since the failure is otherwise indistinguishable from the
  file being absent. Parsed-but-no-`gpio`-object is debug only.
- On success RIC logs `GPIOs from /etc/thingino.json: ...` with all
  four pins.

### GPIO Control

Single GPIO mode: pin HIGH = day (filter closed), LOW = night (filter open).

Dual GPIO mode (motor driver): pulse pin1/pin2 HIGH for `pulse_ms`
(default 10ms, matching the timing the thingino `ircut` shell script
has always driven the fleet with), then both LOW. Both 10ms and 100ms
measured 20/20 reliable on a dual-GPIO Wyze Cam3; the knob exists for
mechanisms that disagree. Clamped 1-1000ms.

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
ir940 = false                 # enable 940nm LED (invisible, weaker);
                              # auto-enabled after discovery when 940 is
                              # the ONLY bank the board has and no ir940
                              # key is set -- the only bank must light
pulse_ms = 10                 # dual-GPIO ircut coil pulse width (1-1000)

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
raptorctl ric ircut day|night # drive the IR-cut filter alone
raptorctl ric ir850 on|off    # drive the 850nm LED bank alone
raptorctl ric ir940 on|off    # drive the 940nm LED bank alone
raptorctl ric get-thresholds  # show all threshold values
raptorctl ric set-threshold night_luma 30    # change threshold at runtime
raptorctl ric set-threshold hysteresis_sec 3
```

The manual hardware commands drive one piece each and change no
state: no ISP call, no mode change. The next automatic transition
reasserts whatever auto manages, and the response carries the
operating mode so the caller knows that can happen. The LED commands
deliberately ignore the `ir850`/`ir940` enable flags -- those gate
automatic behavior, while a manual command is explicit intent, and
lighting a disabled bank from the shell is exactly what bench
debugging needs.

Tunable keys: `night_luma` (0-255), `night_gain` (>=0),
`day_gain_pct` (1-100), `night_threshold` (>=0), `day_threshold`
(>=0), `hysteresis_sec` (1-300), `poll_interval_ms` (50-10000),
`photo_ev_night` (>=0), `photo_ev_deep` (>=0), `photo_ev_day`
(>=0), `photo_rgain_rec` (0-65535), `photo_bgain_rec` (0-65535).
Changes take effect on the next poll cycle.

---

## Missing Exposure Fields

Zero in an exposure field means "this backend cannot answer for that
field" -- the contract raptor-hal documents in `hal_isp.c`. A live
sensor never reports a mean luma of exactly 0, so nothing real is lost
by treating it as absent. Every term of the luma trigger is gated on
its field being reported:

- **Gain but no luma** (e.g. Gen3 when `GetAeStatistics` fails): runs
  on the gain terms alone.
- **Luma but no gain**: no gain means no night baseline and no ratio,
  so night→day falls back to the inverse of the day→night test
  (`luma >= night_luma`) -- but only when no IR bank is active. With
  IR LEDs lit that fallback is an oscillator: IR lifts luma over the
  threshold, day mode cuts the LEDs, the scene darkens, night triggers
  again, and the filter clicks all night. Boards without IR keep the
  recovery; boards with IR hold night, the lesser failure.
- **Everything zero** (a HAL with no exposure readback at all): the
  mode holds where it stands, one warning names the situation, and
  manual `raptorctl ric mode` plus `trigger = adc` keep working. Before
  this, `0 < night_luma` read silence as darkness and pinned any such
  platform in night mode permanently.
- **Photo without ev**: every phase of the photo machine compares
  against ev, so with none reported the trigger demotes itself to luma
  with one line rather than idling silently -- an ev-less platform
  would otherwise also calibrate its AWB baseline from zeros.

---

## Diagnostics

The failure modes of this daemon are silent by nature: a frozen
day/night looks exactly like a scene that does not change. Every
dependency RIC polls therefore reports sustained failure, rate-limited,
and reports recovery with the outage length:

- **rvd unreachable**: warned after 10 seconds of failed queries --
  a routine rvd restart takes a few seconds and stays quiet -- then at
  most once a minute. The per-poll `RVD query failed` line stays at
  DEBUG.
- **ADC channel dead**: warned after 2 seconds of failed reads (see
  the ADC section).
- **GPIO writes**: a value node that cannot be opened or written is
  named per actuation -- it is an ircut or LED silently not moving.

All thresholds are written in seconds and converted through
`poll_interval_ms`, so they mean the same thing at any poll rate.

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

The channel is a kernel device node (`/dev/ingenic_adc_aux_N`, with
`/dev/jz_adc_aux_N` as the older name), opened directly. If it cannot
be opened or enabled, RIC warns and falls back to the luma trigger.
The same `hysteresis_sec` debounce applies to ADC transitions.

A channel that opens and then stops answering is reported too: after
two seconds of failed reads a warning names the outage and the mode
day/night is frozen in, repeats at most once a minute, and the first
good read afterwards logs the outage length. The ADC trigger is also
exempt from the all-zero exposure hold below -- the photoresistor
reads its own sensor and must keep polling when the ISP reports
nothing.

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

```
EV:  0 (bright) ────────────────────────────── 400K (dark)
         │              │                  │
     ev_day=5000   ev_night=50000   ev_deep=150000
     ← day detect   night detect →   forced night →
```

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
when EV drops below `photo_ev_day` (bright scene). The first full
ring whose ends agree within the stability band seeds the reference
(the same way the interference and fixed-EV detectors seed theirs);
after that, the last sample is compared against it using ratio
thresholds:

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

- `ric/ric_main.c` — config loading, main loop
- `ric/ric_json.c` — thingino.json GPIO discovery
- `ric/ric_daynight.c` — GPIO control, mode switching, detection dispatch
- `ric/ric_photo.c` — multi-stage EV + AWB photo detection algorithm
- `ric/ric.h` — state, config, and photo mode structures

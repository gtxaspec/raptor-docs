# Multi-Sensor (Dual/Triple Camera) Support

Raptor supports up to 3 sensors on platforms with multi-camera hardware:
T23 (SDK 1.3.0), T32, T40, and T41 (experimental).

## Hardware Requirements

### T23 Dual-Sensor

T23 uses a single MIPI interface with an external MIPI switch chip to
time-multiplex between sensors. Each sensor outputs frames alternately.

Requirements:
- MIPI switch chip connected to a GPIO (typically PA7)
- Two sensor drivers with frame sync support (`s0` + `s1` variants)
- ISP kernel module built with `CONFIG_MULTI_SENSOR=1` and `1.3.0-double` firmware blob
- ISP module loaded with `direct_mode=0` and `mipi_switch_gpio=<N>`
- VIC IRQ 30 priority fix in kernel (commit `38eae569d` in thingino-linux)
- PWM channels 0-3 available (needed for frame sync timing)
- Minimum 32MB rmem for dual 1080p with sub streams

### T32/T40/T41

These platforms have independent MIPI ports per sensor. No MIPI switch
needed; each sensor runs at its own rate. On T40 the ISP still takes
frames from both sensors in turn, so the stream rate also depends on the
dual-sensor mode and buffer counts (see
[Dual-Sensor ISP Mode (T40)](#dual-sensor-isp-mode-t40)).

## Sensor Drivers

Dual-sensor requires sync-capable driver variants:

| Driver | Role | Notes |
|--------|------|-------|
| `sc2336ps0` | Master sensor | Has `sensor_fsync()` callback, registers as `sc2336p` |
| `sc2336ps1` | Slave sensor | Registers as `sc2336ps1`, different I2C address |
| `sc2336p` | Single-sensor | No fsync — will NOT work for dual-sensor |

The `s0` driver implements `TX_ISP_SENSOR_FSYNC_MODE_MS_REALTIME_MISPLACE`
which triggers alternating frame output between master and slave sensors.

## Configuration

### Single-Sensor (default, backward compatible)

```ini
[sensor]
# Auto-detected from /proc/jz/sensor
```

### Dual-Sensor

Replace `[sensor]` with `[sensor0]` and add `[sensor1]`:

```ini
[sensor0]
name = sc2336p
i2c_addr = 0x30
i2c_adapter = 0
sensor_id = 0
width = 1920              # required: procfs doesn't report with s0/s1 drivers
height = 1080

[sensor1]
name = sc2336ps1
i2c_addr = 0x32
i2c_adapter = 0
sensor_id = 1

[mipi_switch]
enabled = false            # T23 ISP module handles via mipi_switch_gpio param

# Sensor 0 streams (same section names as single-sensor)
[stream0]
fps = 15
codec = h264
bitrate = 2500000

[stream1]
enabled = true
width = 640
height = 480

# Sensor 1 streams
[sensor1_stream0]
fps = 15
codec = h264
bitrate = 2500000

[sensor1_stream1]
enabled = true
width = 640
height = 480
```

Key points:
- `width`/`height` in `[sensor0]` is required — the ISP reports `0x0` resolution
  with dual-sensor drivers, so RVD reads it from config
- `[mipi_switch] enabled = false` — the T23 ISP kernel module handles GPIO
  switching internally via the `mipi_switch_gpio` module parameter
- Sub stream resolution must differ from sensor resolution (enables scaler)

### Dual-Sensor ISP Mode (T40)

The T40 has one ISP core for both sensors. `dual_mode` in `[sensor0]`
picks how it takes their frames (`IMPISPDualSensorMode`, set with
`IMP_ISP_SetCameraInputMode` before the sensors are added):

| `dual_mode` | SDK mode | Meaning |
|-------------|----------|---------|
| `allcached` (default) | `DUAL_ALLCACHED` | Both sensors' raw frames are cached |
| `singlecached` | `DUAL_SINGLECACHED` | One sensor's raw frames are cached |
| `direct` | `DUAL_DIRECT` | No raw frame cache |
| `select` | `DUAL_SELECT` | One sensor output at a time; `dual_select` picks it |
| `bypass` | `SIGLE_BYPASS` | Dual-sensor module off |

```ini
[sensor0]
dual_mode = allcached
dual_select = 0            # select mode only
```

Leaving `dual_mode` unset keeps ALLCACHED, which is also what the eufy
T8416 vendor firmware runs. Other platforms ignore the key with a
warning. `select` and `bypass` are wired up but untested.

Measured on the eufy T8416 (sc830ai 4K + sc3338 2304x1296): the ISP
paces both sensors' output together, so slowing one sensor slows the
other's channels too, and `direct` and `singlecached` delivered fewer
frames than `allcached`. A steady 20 fps on all four streams with both
sensors at 20 fps needed `nr_vbs = 3` on every stream and H.264; see
[Frame rate below the sensor rate (T40)](#frame-rate-below-the-sensor-rate-t40).

### ISP Module Loading (T23)

```bash
insmod tx_isp_t23.ko direct_mode=0 isp_clk=200000000 \
    isp_memopt=1 mipi_switch_gpio=7
insmod sensor_sc2336ps0_t23.ko    # master (with fsync)
insmod sensor_sc2336ps1_t23.ko    # slave
```

## Pipeline Architecture

### Framesource Channel Mapping

| Sensor | Main | Sub | Extension |
|--------|------|-----|-----------|
| 0 | FS ch 0 | FS ch 1 | FS ch 2 |
| 1 | FS ch 3 | FS ch 4 | FS ch 5 |
| 2 | FS ch 6 | FS ch 7 | FS ch 8 |

### Encoder Groups

Encoder groups are assigned sequentially across sensors:
- Stream 0 (sensor 0 main) → enc group 0
- Stream 1 (sensor 0 sub) → enc group 1
- Stream 2 (sensor 1 main) → enc group 2
- Stream 3 (sensor 1 sub) → enc group 3

### Ring Buffer Naming

| Sensor | Main | Sub | JPEG |
|--------|------|-----|------|
| 0 | `main` | `sub` | `jpeg0`, `jpeg1` |
| 1 | `s1_main` | `s1_sub` | `s1_jpeg0`, `s1_jpeg1` |
| 2 | `s2_main` | `s2_sub` | `s2_jpeg0`, `s2_jpeg1` |

Sensor 0 ring names are identical to single-sensor — no consumer changes
needed for existing single-sensor deployments.

### RTSP Endpoints

```
/stream0   — sensor 0 main
/stream1   — sensor 0 sub
/stream2   — sensor 1 main
/stream3   — sensor 1 sub
/stream4   — sensor 2 main (if triple)
/stream5   — sensor 2 sub (if triple)
```

Legacy aliases `/main` and `/sub` still map to sensor 0.

## Runtime ISP Control

Per-sensor ISP tuning via raptorctl:

```bash
# Sensor 0 (default)
raptorctl rvd set-brightness 128

# Sensor 1
raptorctl rvd set-brightness 200 --sensor 1

# Sensor 1 flip
raptorctl rvd set-hflip 1 --sensor 1
```

Commands supporting `--sensor`: brightness, contrast, saturation,
sharpness, hue, sinter, temper, ae-comp, max-again, max-dgain,
hflip, vflip, antiflicker.

## HAL API

### Multi-Sensor Init

```c
rss_multi_sensor_config_t cfg = {0};
cfg.sensor_count = 2;
cfg.sensors[0] = (rss_sensor_config_t){ .name = "sc2336p", ... };
cfg.sensors[1] = (rss_sensor_config_t){ .name = "sc2336ps1", .sensor_id = 1, ... };
RSS_HAL_CALL(ops, init, ctx, &cfg);
```

### Per-Sensor ISP Tuning

```c
// Sensor 0 (legacy, backward compat)
RSS_HAL_CALL(ops, isp_set_brightness, ctx, 128);

// Sensor N (multi-sensor)
RSS_HAL_CALL(ops, isp_set_brightness_n, ctx, 1, 200);
```

The `_n` variants dispatch to platform-specific APIs:
- T23: `IMP_ISP_MultiCamera_Tuning_Set*(IMPVI_NUM, val)`
- T32/T40/T41: `IMP_ISP_Tuning_Set*(IMPVI_NUM, &val)`

## Troubleshooting

### No frames from second sensor

- Check sensor driver: must be `s0`/`s1` variants with fsync, not standalone
- Check ISP module: `direct_mode=0`, `mipi_switch_gpio` correct
- Check kernel: VIC IRQ 30 must be prioritized over IRQ 29
- Check PWM: all channels (0-3) must be available
- Verify: `cat /proc/jz/isp/isp-m0` — both sensors should show
  non-zero Integration Time and analog gain

### Out of memory (VBMCreatePool failed)

Increase rmem. Dual 1080p with sub streams needs ~32MB minimum.
Check `Mem Free Size` in logcat output.

### Green lines on sub streams

Enable crop on main streams by setting `width`/`height` in `[sensor0]`.
The ISP reports `sensor resolution(0x0)` with dual-sensor drivers;
crop tells the framesource the actual input resolution.

### Frame rate below the sensor rate (T40)

Check each stage against the sensor rate:

1. Sensor: the VIC frame counts in `/proc/jz/isp/isp-w02` grow at the
   sensor rate.
2. ISP output: `qbuf` per `chinx` in `/proc/jz/isp/isp-fs`. If a
   channel runs slower, sample its `queue count` a few times a second.
   Frequent `queue count: 0` means the encoder still holds every
   framesource buffer and the ISP has nowhere to write; raise `nr_vbs`
   on that stream. `nr_vbs = 3` fixed 4K + 2304x1296 at 20 fps (each
   extra 4K NV12 buffer costs ~12.4 MB of rmem).
3. Encoder: compare the `avpu` interrupt rate in `/proc/interrupts`
   with the total ISP output. On the eufy T8416, H.265 topped out at
   ~61 frames/s for 4K + 2304x1296 + two 640x360 streams (about 15 fps
   each with 20 offered); H.264 kept up with all 80.

Raising `isp_clk` to 400 MHz changed nothing on that board, and the
AVPU stopped encoding at 750 MHz under full load and at 1008 MHz.

### IVDC (direct mode)

Dual-sensor IVDC is supported with `direct_mode=2`. This bypasses
the framesource buffer pool and feeds frames directly from the ISP
to the VPU encoder, reducing memory usage and latency.

To enable, load the ISP module with `direct_mode=2` and set `ivdc = true`
on the main streams in raptor.conf:

```ini
[stream0]
ivdc = true

[sensor1_stream0]
ivdc = true
```

ISP module loading for IVDC dual-sensor:
```bash
insmod tx_isp_t23.ko direct_mode=2 isp_clk=200000000 \
    isp_memopt=1 mipi_switch_gpio=7
```

Notes:
- IVDC only applies to main streams. Sub streams always use the normal
  framesource path regardless of `direct_mode`.
- **IPU OSD is incompatible with IVDC** — per Ingenic docs, the T23 IPU
  module only supports non-direct mode. Use ISP OSD instead:
  ```ini
  [osd]
  isp_osd = true
  ```
  ISP OSD overlays inside the ISP pipeline (before framesource), works
  with IVDC, and has zero DDR bandwidth overhead. OSD appears on main
  streams only (T23 hardware limitation). Tested and working with
  dual-sensor IVDC.
- To use IPU OSD with dual-sensor, use `direct_mode=0` (non-IVDC).
- **JPEG snapshots with IVDC**: Full-resolution JPEG is not available
  on IVDC main streams (T23 SDK rejects JPEG+IVDC in the same encoder
  group). Sub-stream JPEG works normally. JPEG ring indices are preserved
  (`jpeg0` absent, `jpeg1` available for sub-stream snapshots).

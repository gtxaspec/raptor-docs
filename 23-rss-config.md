# Raptor Streaming System -- Configuration Reference

Complete reference for `/etc/raptor.conf`. All daemons share this
single INI file. An example with all options (commented out) ships in
`config/raptor.conf` in the raptor repo.

---

## 1. File Format

**INI format** with `[section]` headers and `key = value` pairs.

- **Comments:** `#` or `;` at the start of a line. Inline comments
  use ` #` (space before hash).
- **Booleans:** `true`/`false`, `yes`/`no`, `on`/`off`, `1`/`0`
  (case-insensitive).
- **Integers:** Decimal or `0x` hex prefix.
- **Max lengths:** Line 512 chars, section name 64, key 64, value 256.
- **Default path:** `/etc/raptor.conf` (compile-time `RSS_CONFIG_PATH`).
  Override with `-c <path>` on any daemon.

### Auto-Population

When a daemon reads a key that doesn't exist, the default value is
inserted into the in-memory config. This means `config-get-section`
via raptorctl always shows resolved values, even for keys never
explicitly set.

### Save Semantics

`rss_config_save()` (triggered by `raptorctl config-save`) re-reads
the on-disk file and merges only entries modified at runtime (dirty
flag). This prevents one daemon from clobbering another daemon's
sections. Uses `flock()` on a `.lock` sidecar file for inter-daemon
serialization. Writes are atomic (write to `.tmp`, rename over
original).

---

## 2. Command-Line Flags

All daemons accept the same flags via `rss_daemon_init()`:

| Flag | Description | Default |
|------|-------------|---------|
| `-c <file>` | Config file path | `/etc/raptor.conf` |
| `-f` | Foreground (no fork, log to stderr) | off |
| `-d` | Debug logging (sets level to TRACE) | off |
| `-v` | Print version and exit | -- |
| `-h` | Print help and exit | -- |

Startup sequence: parse args, load config, re-init logging from
`[log]` section (with `-d`/`-f` overrides), daemonize, write PID to
`/var/run/rss/<name>.pid`, install signal handlers, apply CPU
affinity/scheduling from the daemon's own section.

---

## 3. Common Control Commands

Every daemon handles these via `rss_ctrl_handle_common()`:

| Command | Description |
|---------|-------------|
| `config-get` | Read a single config key |
| `config-get-section` | Dump all keys in a section |
| `config-save` | Write running config to disk (dirty merge) |
| `set-affinity` | Pin daemon to a CPU core at runtime |
| `get-affinity` | Query current CPU affinity and scheduling |

---

## 4. Configuration Sections

### `[sensor]`

Sensor hardware configuration for RVD. All values auto-detected from
`/proc/jz/sensor/` if omitted.

For multi-sensor setups, replace `[sensor]` with `[sensor0]` and add
`[sensor1]` (and optionally `[sensor2]`). Multi-sensor requires
`width` and `height` to be set explicitly since procfs doesn't
report them with per-sensor drivers.

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `name` | string | auto | Sensor driver name (e.g. `gc4653`) |
| `i2c_addr` | int | auto | Sensor I2C address |
| `i2c_adapter` | int | auto (fallback 0) | I2C bus number |
| `sensor_id` | int | `0` | Sensor ID for multi-sensor |
| `fps` | int | auto (fallback 25) | Sensor frame rate |
| `rst_gpio` | int | `-1` (auto) | Sensor reset GPIO |
| `pwdn_gpio` | int | `-1` (auto) | Sensor power-down GPIO |
| `power_gpio` | int | `-1` | Sensor power GPIO |
| `boot` | int | auto (fallback 0) | Boot mode: 0=linear, 1=WDR, 2=60fps crop (T40/T41) |
| `mclk` | int | auto (fallback 1) | Clock source: 0=MCLK0, 1=MCLK1, 2=MCLK2 (T40/T41) |
| `video_interface` | int | auto (fallback 0) | Video interface type (MIPI/DVP) |
| `width` | int | `0` | Sensor width (required for multi-sensor) |
| `height` | int | `0` | Sensor height (required for multi-sensor) |
| `antiflicker` | int | `2` | Anti-flicker: 0=off, 1=50Hz, 2=60Hz |
| `low_latency` | bool | `false` | Encoder immediate output (~100ms savings) |

---

### `[mipi_switch]`

MIPI switch GPIO control for T23 multi-sensor setups.

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `enabled` | bool | `false` | Enable MIPI switch control |
| `switch_gpio` | int | `0` | GPIO controlling MIPI switch chip |
| `main_gstate` | int | `0` | GPIO state to select main sensor |
| `sec_gstate` | int | `1` | GPIO state to select secondary sensor |
| `switch_gpio2` | int | `0` | Second MIPI switch GPIO |

---

### `[stream0]` / `[stream1]`

Video stream encoder configuration for RVD. Two streams per sensor.
For multi-sensor, use `[sensor1_stream0]`, `[sensor1_stream1]`, etc.

| Key | Type | Default (stream0 / stream1) | Description |
|-----|------|-----------------------------|-------------|
| `enabled` | bool | `true` | Enable this stream (stream1 only) |
| `width` | int | sensor_w / `640` | Frame width |
| `height` | int | sensor_h / `360` | Frame height |
| `fps` | int | `25` | Frames per second |
| `codec` | string | `h264` | Codec: `h264`, `h265`/`hevc`, `jpeg`/`mjpeg` |
| `profile` | int | `2` | H.264 profile: 0=Baseline, 1=Main, 2=High |
| `bitrate` | int | `3000000` / `1000000` | Target bitrate (bps) |
| `max_bitrate` | int | `0` | Max bitrate; 0=auto (4/3 of target for capped modes) |
| `rc_mode` | string | `cbr` | Rate control: `cbr`, `vbr`, `fixqp`, `smart`, `capped_vbr`, `capped_quality` |
| `gop` | int | same as fps | GOP length (keyframe interval in frames) |
| `init_qp` | int | `-1` | Initial QP (-1=SDK default; required for fixqp, 0-51) |
| `min_qp` | int | `-1` | Minimum QP (-1=SDK default) |
| `max_qp` | int | `-1` | Maximum QP (-1=SDK default) |
| `ip_delta` | int | `-1` | QP delta I vs P frames (T31+, -1=SDK default) |
| `pb_delta` | int | `-1` | QP delta P vs B frames (T31+, -1=SDK default) |
| `max_psnr` | int | `0` | PSNR quality cap for capped modes (T31+) |
| `nr_vbs` | int | `2` | Number of VBs for framesource |
| `ivdc` | bool | `false` | ISP-VPU Direct Connect (T23+, reduces rmem) |
| `osd_enabled` | bool | `true` | Per-stream OSD enable |
| `jpeg` | bool | `true` | Per-stream JPEG snapshot enable |
| `jpeg_quality` | int | global | Override `[jpeg]` quality for this stream |
| `jpeg_fps` | int | global | Override `[jpeg]` fps for this stream |

---

### `[image]`

ISP image tuning for RVD. For multi-sensor, use `[sensor1_image]`.

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `brightness` | int | `128` | 0-255 |
| `contrast` | int | `128` | 0-255 |
| `saturation` | int | `128` | 0-255 |
| `sharpness` | int | `128` | 0-255 |
| `hue` | int | `128` | 0-255 |
| `sinter` | int | `128` | Spatial noise reduction (0-255, T20-T31) |
| `temper` | int | `128` | Temporal noise reduction (0-255, T20-T31) |
| `ae_comp` | int | `128` | AE compensation (0-255) |
| `max_again` | int | `160` | Max analog gain (0-160) |
| `max_dgain` | int | `80` | Max digital gain (0-160) |
| `dpc_strength` | int | `128` | Dead pixel correction (0-255, T23/T31) |
| `drc_strength` | int | `128` | Dynamic range compression (0-255, T21/T23/T31) |
| `defog_strength` | int | `128` | Defog/dehaze (0-255, T23/T31) |
| `highlight_depress` | int | `0` | Highlight depression (0-255, T20-T31) |
| `backlight_comp` | int | `0` | Backlight compensation (0-10, T23/T31) |
| `hflip` | int | `0` | Horizontal flip (0 or 1) |
| `vflip` | int | `0` | Vertical flip (0 or 1) |

---

### `[jpeg]`

JPEG snapshot channel configuration for RVD. Controls the paired
JPEG encoder channels that provide snapshots and MJPEG feeds.

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `enabled` | bool | `true` | Enable JPEG snapshot channels |
| `quality` | int | `75` | JPEG quality (1-100) |
| `fps` | int | `1` | JPEG capture FPS |
| `bufshare` | bool | `false` | Share encoder buffers (incompatible with refmode) |
| `idle` | bool | `true` | Stop JPEG encoder when no viewers (saves CPU/VPU) |

Per-stream overrides: set `jpeg`, `jpeg_quality`, `jpeg_fps` in
`[streamN]` sections.

---

### `[ring]`

Ring buffer sizing for RVD. Ring buffers are the IPC transport
between RVD (producer) and all consumer daemons (RSD, RHD, RWD, etc).

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `refmode` | bool | `false` | Zero-copy mode: encoder writes directly to SHM, ring carries metadata only |
| `main_slots` | int | `32` | Main stream ring slot count |
| `main_data_mb` | int | `0` | Main ring data MB; 0=auto-size from bitrate |
| `sub_slots` | int | `32` | Sub stream ring slot count |
| `sub_data_mb` | int | `0` | Sub ring data MB; 0=auto-size from bitrate |

---

### `[audio]`

Audio input/output configuration for RAD.

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `enabled` | bool | `true` | Enable audio daemon |
| `input` | string | `amic` | Input type: `amic` (analog mic), `dmic` (digital mic array) |
| `sample_rate` | int | `16000` | Sample rate (Hz) |
| `codec` | string | `l16` | Codec: `pcmu`, `pcma`, `l16`, `aac`, `opus` |
| `bitrate` | int | `128000` (aac) / `32000` (opus) | Encoder bitrate (bps) |
| `opus_complexity` | int | `5` | Opus complexity (0-10, lower=less CPU) |
| `volume` | int | `80` | Mic input volume |
| `gain` | int | `25` | Mic input gain |
| `device` | int | `1` | Audio input device number |

**Digital mic (input=dmic, T30/T31/T32/T33/T40/T41):**

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `dmic_count` | int | `1` | Number of digital mics (1/2/4) |
| `dmic_aec_id` | int | `0` | Which mic for AEC processing (0-3) |

**Audio output (speaker/backchannel):**

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `ao_enabled` | bool | `false` | Enable speaker output |
| `ao_volume` | int | `80` | Speaker volume |
| `ao_gain` | int | `25` | Speaker gain |
| `ao_sample_rate` | int | same as input | Speaker sample rate |

**Audio effects (requires libaudioProcess.so):**

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `aec_enabled` | bool | `false` | Acoustic echo cancellation (requires ao_enabled) |
| `aec_profile_path` | string | `/etc` | Directory containing webrtc_profile.ini |
| `ns_enabled` | bool | `false` | Noise suppression |
| `ns_level` | int | `1` | NS level: 0=low, 1=moderate, 2=high, 3=veryhigh |
| `hpf_enabled` | bool | `false` | High-pass filter (removes DC offset/low rumble) |
| `agc_enabled` | bool | `false` | Automatic gain control |
| `agc_target_dbfs` | int | `10` | AGC target level (0-31 dBfs) |
| `agc_compression_db` | int | `0` | AGC compression gain (0-90 dB) |

---

### `[rtsp]`

RTSP streaming server configuration. Shared by both RSD (compy-based)
and RSD-555 (live555-based). Both backends read from this section.

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `enabled` | bool | `true` | Enable RTSP server |
| `port` | int | `554` (plain) / `322` (TLS) | RTSP listen port |
| `max_clients` | int | `8` | Max simultaneous clients (compile max 8) |
| `username` | string | (empty) | Digest auth username (both required to enable) |
| `password` | string | (empty) | Digest auth password |
| `tls` | bool | `false` | Enable RTSPS (port defaults to 322) |
| `tls_cert` | string | `/etc/ssl/certs/uhttpd.crt` | TLS certificate path |
| `tls_key` | string | `/etc/ssl/private/uhttpd.key` | TLS private key path |
| `tls_cipher_preference` | string | `default` | `default` or `chacha20` (see below) |
| `session_name` | string | `Raptor Live` | SDP session name (s=) |
| `session_info` | string | (empty) | SDP session description (i=) |
| `session_timeout` | int | `60` | Session timeout in seconds (min 10) |
| `tcp_sndbuf` | int | `65536` | TCP send buffer bytes (lower=less latency) |
| `rtcp_sr` | bool | `false` | Send RTCP Sender Reports (RSD only) |
| `backchannel` | bool | `false` | Two-way audio (ONVIF Profile T, PCMU/8000) |
| `out_buffer_size` | int | `262144` | live555 OutPacketBuffer max size (RSD-555 only) |

**Endpoint aliases:**

Custom URL aliases replace the default `/streamN` paths for each
stream. Only the alias is accepted when set -- the default path is
disabled. Aliases must match `[A-Za-z0-9_.-]+` and cannot shadow
reserved names (`main`, `sub`, `audio`, `backchannel`).

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `endpoint_main` | string | (empty) | Sensor 0 main (replaces /stream0) |
| `endpoint_sub` | string | (empty) | Sensor 0 sub (replaces /stream1) |
| `endpoint_s1_main` | string | (empty) | Sensor 1 main (replaces /stream2) |
| `endpoint_s1_sub` | string | (empty) | Sensor 1 sub (replaces /stream3) |
| `endpoint_s2_main` | string | (empty) | Sensor 2 main (replaces /stream4) |
| `endpoint_s2_sub` | string | (empty) | Sensor 2 sub (replaces /stream5) |

**RSD-555 overrides (`[rtsp-555]`):**

RSD-555 reads all config from `[rtsp]` but checks `[rtsp-555]` first
for `port`, allowing both backends to run on different ports
simultaneously.

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `port` | int | falls back to `[rtsp]` port | RSD-555 listen port |

**TLS cipher preference:**
- `default` -- mbedtls built-in order (GCM-first in TLS 1.3). Best
  on hardware with AES acceleration.
- `chacha20` -- force ChaCha20-Poly1305 only for TLS 1.3; TLS 1.2
  falls back to CBC/CCM/GCM. Best on Ingenic MIPS where scalar
  ChaCha20 beats the /dev/aes hardware engine.

---

### `[http]`

HTTP server for snapshots and MJPEG streaming (RHD).

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `enabled` | bool | `true` | Enable HTTP server |
| `port` | int | `8080` | HTTP listen port |
| `max_clients` | int | `8` | Max simultaneous clients (compile max 8) |
| `username` | string | (empty) | Basic auth username (both required to enable) |
| `password` | string | (empty) | Basic auth password |
| `https` | bool | `false` | Enable HTTPS |
| `cert` | string | `/etc/ssl/certs/uhttpd.crt` | TLS certificate path |
| `key` | string | `/etc/ssl/private/uhttpd.key` | TLS private key path |

---

### `[osd]`

OSD global settings for ROD.

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `enabled` | bool | `true` | Enable OSD system |
| `isp_osd` | bool | `false` | Use ISP OSD instead of IPU OSD (T23/T32/T40/T41) |
| `font` | string | `/usr/share/fonts/default.ttf` | Font file path |
| `font_size` | int | `24` | Default font size |
| `font_color` | string | `0xFFFFFFFF` | Default text color (BGRA hex) |
| `font_stroke` | int | `1` | Default stroke width (0-5) |
| `stroke_color` | string | `0xFF000000` | Default stroke color (BGRA hex) |
| `time_format` | string | `%Y-%m-%d %H:%M:%S` | strftime format for `%time%`. `%f` expands to 2-digit frame number (00-N based on stream0 fps) |
| `privacy_color` | string | `0xFF000000` | Privacy cover fill color (BGRA) |

ISP OSD works with IVDC and has zero DDR bandwidth overhead. IPU OSD
is the default and works on all platforms.

---

### `[osd.<name>]`

Individual OSD elements. Each `[osd.<name>]` section defines one
element. Elements can also be added/removed at runtime via raptorctl.

**Common keys:**

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `type` | string | `text` | Element type: `text`, `image`, `overlay`, `receipt` |
| `position` | string | `top_left` | `top_left`, `top_center`, `top_right`, `bottom_left`, `bottom_center`, `bottom_right`, `center`, or `x,y` |
| `visible` | bool | `true` | Element visibility |
| `sub_only` | bool | `false` | Render on sub-stream only |

**Text element keys:**

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `template` | string | (empty) | Text with `%time%`, `%uptime%`, `%hostname%` expansion |
| `align` | string | (empty) | Text alignment: `left`, `center`, `right` |
| `font_size` | int | `0` | Font size (0=use global) |
| `max_chars` | int | `20` | SHM width sizing (max 128) |
| `update` | string | `tick` | Update mode: `tick` (every second), `change` (on content change) |
| `color` | string | (null) | Text color (BGRA hex, null=use global) |
| `stroke_color` | string | (null) | Stroke color (BGRA hex, null=use global) |
| `stroke_size` | int | `-1` | Stroke size (-1=use global) |

**Image element keys:**

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `path` | string | (empty) | Path to raw BGRA image file |
| `width` | int | `0` | Image width in pixels |
| `height` | int | `0` | Image height in pixels |

**Receipt element keys:**

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `max_lines` | int | `20` | Max visible lines |
| `max_line_length` | int | `80` | Max characters per line |
| `timeout` | int | `0` | Byte accumulation timeout (seconds) |
| `bg_color` | string | `0x80000000` | Background color (BGRA) |
| `source` | string | (empty) | Input source: `fifo` or `uart` |
| `device` | string | (empty) | FIFO or UART device path |

---

### `[ircut]`

IR-cut filter and day/night control for RIC.

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `enabled` | bool | `true` | Enable IR-cut control |
| `mode` | string | `auto` | Operating mode: `auto`, `day`, `night` |
| `trigger` | string | `luma` | Trigger source: `luma`, `gain`, `adc`, `photo` |
| `hysteresis_sec` | int | `5` | Consecutive seconds before switching |
| `poll_interval_ms` | int | `1000` (luma/gain/adc) / `100` (photo) | Polling interval |
| `gpio_ircut` | int | `-1` | IR-cut GPIO (-1=auto from /etc/thingino.json) |
| `gpio_ircut2` | int | `-1` | Second IR-cut GPIO |
| `gpio_irled` | int | `-1` | 850nm IR LED GPIO (-1=auto from /etc/thingino.json) |
| `gpio_irled2` | int | `-1` | 940nm IR LED GPIO (-1=auto from /etc/thingino.json) |
| `ir850` | bool | `true` | Enable 850nm IR LED (visible glow, stronger illumination) |
| `ir940` | bool | `false` | Enable 940nm IR LED (invisible, weaker illumination) |

**Luma trigger (default):**

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `night_luma` | int | `20` | AE luma below this triggers night (0-255) |
| `night_gain` | int | `80000` | Total gain above this triggers night |
| `day_gain_pct` | int | `25` | Night-to-day: gain below this % of night baseline |

**Gain trigger (legacy):**

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `night_threshold` | int | `40000` | Total gain above this triggers night |
| `day_threshold` | int | `25000` | Total gain below this triggers day |

**ADC trigger (photoresistor):**

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `adc_channel` | int | `0` | ADC channel |
| `adc_night` | int | `200` | ADC below this triggers night |
| `adc_day` | int | `600` | ADC above this triggers day |

**Photo trigger (EV + AWB, `trigger=photo`):**

```
EV:  0 (bright) ────────────────────────────── 400K (dark)
         │              │                  │
     ev_day=5000   ev_night=50000   ev_deep=150000
     ← day detect   night detect →   forced night →
```

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `photo_ev_night` | int | `50000` | EV above this is dark |
| `photo_ev_deep` | int | `150000` | EV above this is very dark (fixed-EV fallback) |
| `photo_ev_day` | int | `5000` | EV below this is bright (starts day detection) |
| `photo_rgain_rec` | int | `0` | R-gain AWB baseline (0=auto-calibrate from 16 bright samples) |
| `photo_bgain_rec` | int | `0` | B-gain AWB baseline (0=auto-calibrate from 16 bright samples) |

---

### `[recording]`

Media recording to SD card for RMR.

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `enabled` | bool | `false` | Enable recording |
| `mode` | string | `continuous` | Mode: `continuous`, `motion`, `both` |
| `stream` | int | `0` | Stream index (0=main, 1=sub, 2-5 for multi-sensor) |
| `audio` | bool | `true` | Include audio |
| `segment_minutes` | int | `5` | Segment rotation interval (minutes) |
| `storage_path` | string | `/mnt/mmcblk0p1/raptor` | Recording path |
| `max_storage_mb` | int | `0` | Max storage (MB); 0=unlimited |
| `prebuffer_sec` | int | `5` | Pre-event footage buffer (0-5 seconds) |
| `clip_length_sec` | int | `60` | Motion clip max duration before rotation |
| `clip_max_mb` | int | `100` | Max total clip storage (MB) |

---

### `[motion]`

Motion detection for RMD (state machine) and RVD (IVS pipeline).

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `enabled` | bool | `false` | Enable motion detection |
| `algorithm` | string | `move` | Algorithm: `move`, `base_move`, `persondet`, `yolo` |
| `sensitivity` | int | `3` | Sensitivity: move 0-4, persondet 0-5 |
| `grid` | string | `4x4` | Move only: auto-generate ROI grid |
| `cooldown_sec` | int | `10` | Cooldown after motion stops |
| `poll_interval_ms` | int | `500` | Motion polling interval |
| `skip_frames` | int | `5` | Process every Nth frame in IVS |
| `record` | bool | `true` | Trigger recording on motion |
| `record_post_sec` | int | `30` | Continue recording after motion stops |
| `gpio_pin` | int | `-1` | GPIO to assert on motion |

**Explicit ROIs (disables grid):**

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `roi_count` | int | `0` | Number of ROI zones |
| `roi0`...`roiN` | string | -- | ROI coordinates: `x0,y0,x1,y1` |

**Persondet options:**

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `det_distance` | int | `2` | Detection distance: 0=6m, 1=8m, 2=10m, 3=11m, 4=13m |
| `motion_trigger` | bool | `false` | Require motion before running deep learning |

**YOLO options:**

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `model` | string | `/usr/share/models/magik_model_yolov5.bin` | Model path |
| `num_classes` | int | `4` | Number of classes (must match model) |
| `conf_threshold` | int | `40` | Confidence threshold (0-100%) |
| `nms_threshold` | int | `50` | NMS threshold (0-100%) |

---

### `[webrtc]`

WebRTC streaming server for RWD. Provides WHIP signaling for
browser-based live viewing.

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `enabled` | bool | `false` | Enable WebRTC daemon |
| `udp_port` | int | `8443` | UDP port for ICE/DTLS/SRTP |
| `http_port` | int | `8554` | HTTP port for WHIP signaling |
| `max_clients` | int | `2` | Max simultaneous WebRTC peers |
| `cert` | string | (empty, required) | TLS/DTLS certificate path |
| `key` | string | (empty, required) | TLS/DTLS private key path |
| `https` | bool | `true` | HTTPS for signaling (enables Talk button) |
| `local_ip` | string | auto-detected | Local IP for ICE candidates |
| `audio_mode` | string | `auto` | `auto` (passthrough when possible) or `opus` (always transcode) |
| `opus_complexity` | int | `2` | Opus transcode complexity (0-10) |
| `opus_bitrate` | int | `64000` | Opus transcode bitrate (bps) |
| `username` | string | (empty) | HTTP Basic auth username (both required to enable) |
| `password` | string | (empty) | HTTP Basic auth password |

---

### `[webtorrent]`

WebTorrent P2P sharing for RWD. Enables remote NAT traversal via
a WebSocket tracker.

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `enabled` | bool | `false` | Enable WebTorrent sharing |
| `tracker` | string | `wss://tracker.openwebtorrent.com` | WebSocket tracker URL |
| `stun_server` | string | `stun.l.google.com` | STUN server hostname |
| `stun_port` | int | `19302` | STUN server port |
| `tls_verify` | bool | `true` | Verify tracker TLS cert |
| `viewer_url` | string | `https://viewer.thingino.com/share.html` | Viewer page URL |
| `share_key` | string | auto-generated | Fixed share key (min 8 chars) |

---

### `[webcam]`

USB webcam gadget for RWC. Exposes the camera as a UVC+UAC USB
device.

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `enabled` | bool | `false` | Enable webcam daemon |
| `device` | string | `/dev/video0` | UVC gadget device |
| `jpeg_stream` | string | `jpeg0` | Ring for MJPEG 1080p |
| `h264_stream` | string | `main` | Ring for H.264 1080p |
| `jpeg_sub_stream` | string | `jpeg1` | Ring for MJPEG 360p |
| `h264_sub_stream` | string | `sub` | Ring for H.264 360p |
| `buffers` | int | `4` | V4L2 buffer count (2-8) |
| `audio` | bool | `true` | Enable USB microphone |
| `audio_stream` | string | `audio` | Audio ring name (requires codec=l16) |

---

### `[push]`

RTMP/RTMPS stream push to external servers for RSP. Supports
YouTube, Twitch, and other RTMP targets.

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `enabled` | bool | `false` | Enable stream push |
| `url` | string | (empty, required) | RTMP/RTMPS target URL |
| `stream` | int | `0` | Stream index: 0=main, 1=sub |
| `audio` | bool | `true` | Include audio (transcoded to AAC if needed) |
| `autostart` | bool | `true` | Start pushing on daemon launch |
| `reconnect_ms` | int | `5000` | Reconnect delay on failure (1000-60000) |
| `tcp_sndbuf` | int | `262144` | TCP send buffer bytes |

---

### `[srt]`

SRT listener for RSR. Serves MPEG-TS over SRT protocol.

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `enabled` | bool | `false` | Enable SRT listener |
| `port` | int | `9000` | SRT listener port (UDP) |
| `latency` | int | `120` | SRT latency in ms (TsbPd buffer) |
| `passphrase` | string | (empty) | AES encryption passphrase (empty = no encryption) |
| `pbkeylen` | int | `16` | AES key length: 16 (AES-128), 24 (AES-192), or 32 (AES-256) |
| `max_clients` | int | `4` | Max simultaneous SRT clients (1-8) |
| `stream` | int | `0` | Default video stream: 0=main, 1=sub |
| `audio` | bool | `true` | Include audio in TS stream |

Clients can override the default stream by setting SRT STREAMID to
a ring name (`main`, `sub`, `s1_main`, `s1_sub`, `s2_main`, `s2_sub`).

Example: `ffplay srt://camera-ip:9000` for main stream,
`ffplay "srt://camera-ip:9000?streamid=sub"` for sub stream.

---

### `[filesource]`

File source for RFS. Replaces RVD+RAD on platforms without ISP
hardware (A1). Plays video/audio files into ring buffers.

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `enabled` | bool | `true` | Enable file source |
| `video_file` | string | (empty) | Video file path (H.264/H.265 raw or MP4) |
| `audio_file` | string | (empty) | Audio file path (raw PCM) |
| `codec` | string | `auto` | Video codec: `auto`, `h264`, `h265` |
| `fps` | int | `25` | Playback frame rate |
| `loop` | bool | `true` | Loop playback |
| `audio_codec` | string | `l16` | Audio codec: `l16`, `pcmu`/`g711u`, `pcma`/`g711a`, `aac`, `opus` |
| `audio_sample_rate` | int | `16000` | Audio sample rate |
| `video_ring_name` | string | `main` | Video ring buffer name |
| `video_slots` | int | `32` | Video ring slot count |
| `video_data_mb` | int | `2` | Video ring data size (MB) |
| `audio_ring_name` | string | `audio` | Audio ring buffer name |
| `audio_slots` | int | `32` | Audio ring slot count |
| `audio_data_mb` | int | `1` | Audio ring data size (MB) |

---

### `[log]`

Logging configuration. Shared by all daemons.

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `level` | string | `info` | Log level: `fatal`, `error`, `warn`, `info`, `debug`, `trace` |
| `target` | string | `syslog` | Log target: `syslog`, `stderr`, `file` |
| `file` | string | (empty) | Log file path (when target=`file`) |

Command-line flags override: `-d` sets level to `trace`, `-f` sets
target to `stderr`.

---

### Per-Daemon Sections: `[rvd]`, `[rsd]`, `[rsd-555]`, `[rad]`, `[rod]`, `[rhd]`, `[rwd]`, `[rwc]`, `[rmr]`, `[rmd]`, `[rfs]`, `[rsp]`, `[ric]`

CPU affinity and real-time scheduling. Only meaningful on dual-core
SoCs (T40, T41, A1).

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `cpu_affinity` | int | `-1` (no pin) | Pin to CPU core (0 or 1) |
| `sched_priority` | int | `-1` (SCHED_OTHER) | SCHED_FIFO real-time priority (1-99) |

---

## 5. CLI Tools

These programs use no config file.

**raptorctl** -- CLI control tool. Communicates with daemons via Unix
sockets at `/var/run/rss/<daemon>.sock`.

| Flag | Description |
|------|-------------|
| `-h` | Show help |
| `-v` | Show version |
| `-j '<json>'` | Send raw JSON to a daemon |

**ringdump** -- Ring buffer inspection tool.

| Flag | Description |
|------|-------------|
| `-f` | Follow (continuous output) |
| `-d` | Dump raw frame data |
| `-l` | Show frame latency |
| `-n <count>` | Limit number of frames |
| `-v` | Show version |
| `-h` | Show help |

**rac** -- Audio playback and recording tool.

| Subcommand | Description |
|------------|-------------|
| `record [-d <seconds>]` | Record audio from device |
| `play [-r <rate>]` | Play audio to speaker |
| `status` | Show audio pipeline status |
| `ao-volume <value>` | Set speaker volume |
| `ao-gain <value>` | Set speaker gain |

---

## 6. Key Paths

| Path | Purpose |
|------|---------|
| `/etc/raptor.conf` | Default config file |
| `/var/run/rss/` | Runtime directory |
| `/var/run/rss/<daemon>.pid` | PID files |
| `/var/run/rss/<daemon>.sock` | Control sockets |
| `/dev/shm/` | Ring buffers and OSD shared memory |
| `/etc/thingino.json` | Auto-discovered GPIO pins (IR-cut, LEDs) |
| `/proc/jz/sensor/` | Procfs sensor auto-detection |

---

## 7. Runtime Configuration

Most config values can be changed at runtime without restarting
daemons. Use raptorctl to modify values and `config-save` to persist:

```sh
# Change a value at runtime
raptorctl rvd enc-set 0 bitrate 2000000

# Read current config
raptorctl rvd config-get-section stream0

# Persist to disk
raptorctl rvd config-save
```

ISP image tuning, encoder parameters (via `enc-set`/`enc-get`), OSD
elements, audio volume/gain, and motion sensitivity are all
hot-configurable. Structural changes (adding streams, changing codec
type, enabling/disabling daemons) require a daemon restart.

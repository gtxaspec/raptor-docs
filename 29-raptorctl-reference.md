# Raptor Streaming System -- raptorctl Command Reference

`raptorctl` is the CLI control tool for all Raptor daemons. It sends
JSON commands over Unix domain sockets and prints responses. All
commands take effect immediately (no daemon restart) unless noted.

---

## 1. Usage

```
raptorctl <command> [args...]
raptorctl <daemon> <command> [args...]
raptorctl -j '<json>'
```

**JSON mode** sends raw JSON directly to a daemon socket:

```sh
raptorctl -j '{"daemon":"rvd","cmd":"set-brightness","value":128}'
raptorctl -j '[{"daemon":"rvd","cmd":"get-isp"},{"daemon":"rad","cmd":"status"}]'
```

**Daemons:** rvd, rsd, rad, rod, rhd, ric, rmr, rmd, rwd, rwc, rfs,
rsp, rsr.

---

## 2. Global Commands

These do not target a specific daemon.

| Command | Description |
|---------|-------------|
| `status` | Show running/stopped state of all daemons |
| `memory` | Show private/shared memory usage per daemon |
| `cpu` | Show CPU usage per daemon (1-second sample) |
| `config get <section> <key>` | Read a config value |
| `config get <section>` | Show all keys in a section |
| `config set <section> <key> <value>` | Set a config value (in memory) |
| `config save` | Persist running config to disk |
| `test-motion [sec]` | Trigger a clip recording (default 10s) |

---

## 3. Common Daemon Commands

Available on every daemon via the shared control handler.

### Lifecycle

| Command | Description |
|---------|-------------|
| `<daemon> start [args...]` | Start daemon (fork+exec via PATH, passes extra args like `-c`) |
| `<daemon> stop` | Stop daemon (clean shutdown, SIGTERM fallback after 2s) |
| `<daemon> restart` | Restart daemon (clean re-exec, preserves original args) |

### Runtime

| Command | Description |
|---------|-------------|
| `<daemon> status` | Show daemon details (uptime, version, state) |
| `<daemon> config` | Show running config for the daemon's section |
| `<daemon> set-log-level <level>` | Change log verbosity at runtime |
| `<daemon> get-log-level` | Show current log level |
| `<daemon> set-affinity <cpu>` | Pin daemon to a CPU core |
| `<daemon> get-affinity` | Show CPU affinity and scheduling policy |

**Log levels:** `fatal`, `error`, `warn`, `info`, `debug`, `trace`

---

## 4. RVD -- Video Pipeline

RVD owns the sensor, framesource, ISP, and encoder. Channel `<ch>` is
0 for main stream, 1 for sub stream.

### Encoder Controls

With the V4L2 backend active (`[system] video_backend = v4l2`), RVD
answers only `status`, `config-show`, `request-idr` and the read-only
getters (`get-bitrate`/`get-fps`/`get-gop`/`get-qp-bounds`/
`get-rc-mode`); every other verb returns "not supported by the V4L2
backend" rather than touching uninitialized SDK state.

| Command | Description |
|---------|-------------|
| `set-rc-mode <ch> <mode> [bps]` | Rate control: cbr, vbr, fixqp, capped |
| `set-bitrate <ch> <bps>` | Target bitrate in bits/sec |
| `set-gop <ch> <length>` | GOP length in frames |
| `set-fps <ch> <fps>` | Encoder frame rate |
| `set-sensor-fps <fps>` | Transient sensor-rate override: sensor timing, encoder rate control and GOP move together, nothing persists; `0` restores the boot rate. Errors when the backend cannot report a base |
| `get-sensor-fps` | Live sensor rate plus the boot baseline |
| `set-qp-bounds <ch> <min> <max>` | QP range |
| `set-qp-bounds-per-frame <ch> <iMin> <iMax> <pMin> <pMax>` | Per-frame-type QP |
| `set-max-pic-size <ch> <iK> <pK>` | Max I/P frame size in kbits |
| `set-h264-trans <ch> <offset>` | H.264 chroma QP offset [-12..12] |
| `set-h265-trans <ch> <cr> <cb>` | H.265 chroma QP offsets [-12..12] |
| `set-roi <ch> <idx> <en> <x> <y> <w> <h> <qp>` | ROI region |
| `set-super-frame <ch> <mode> <iThr> <pThr>` | Super frame config |
| `set-pskip <ch> <en> <max_frames>` | P-skip |
| `set-srd <ch> <en> <level>` | Static region detection |
| `set-enc-denoise <ch> <en> <type> <iQP> <pQP>` | Encoder-level denoise |
| `set-gdr <ch> <en> <cycle>` | Gradual decoder refresh |
| `set-enc-crop <ch> <en> <x> <y> <w> <h>` | Encoder crop |
| `enc-set <ch> <param> <value>` | Generic encoder parameter set |
| `enc-get <ch> <param>` | Generic encoder parameter get |
| `enc-list [ch]` | List available encoder parameters |

`status` reports per-stream geometry, codec and rate control, and --
for streams that stamp a source frame counter (video; JPEG channels do
not) -- the loss ledger: `drop_source` (encoder frames the producer
never fetched), `drop_publish` (fetched but never published),
`src_seq_resets` (encoder restarts). All three hold at zero on a
healthy pipeline; the ring consumer guide covers the semantics.
`ringdump <ring>` prints the same counters on its `Src seq:` line.

### Encoder Queries

| Command | Description |
|---------|-------------|
| `get-enc-caps` | Show encoder capabilities for this SoC |
| `get-bitrate <ch>` | Target and measured bitrate |
| `get-fps <ch>` | Current frame rate |
| `get-gop <ch>` | GOP length |
| `get-qp-bounds <ch>` | QP range |
| `get-rc-mode <ch>` | Rate control mode |
| `get-h264-trans <ch>` | H.264 chroma QP offset |
| `get-h265-trans <ch>` | H.265 chroma QP offsets |
| `get-roi <ch> <idx>` | ROI region config |
| `get-super-frame <ch>` | Super frame config |
| `get-pskip <ch>` | P-skip config |
| `get-srd <ch>` | SRD config |
| `get-enc-denoise <ch>` | Encoder denoise config |
| `get-gdr <ch>` | GDR config |
| `get-enc-crop <ch>` | Encoder crop config |

### Stream Control

| Command | Description |
|---------|-------------|
| `stream-stop <ch>` | Stop stream pipeline |
| `stream-start <ch>` | Start stopped stream |
| `stream-restart <ch>` | Restart stream (stop + start) |
| `set-codec <ch> <h264\|h265>` | Change codec (restarts stream) |
| `set-resolution <ch> <w> <h>` | Change resolution (restarts stream) |
| `set-jpeg-quality <ch> <1-100>` | Change JPEG quality (restarts channel) |
| `request-idr [ch]` | Request immediate keyframe |
| `request-pskip <ch>` | Request P-skip frame |
| `request-gdr <ch> <frames>` | Request GDR over N frames |

### ISP Tuning

All ISP commands take effect immediately on the next frame.

| Command | Range | Description |
|---------|-------|-------------|
| `set-brightness <val>` | 0-255 | Image brightness |
| `set-contrast <val>` | 0-255 | Image contrast |
| `set-saturation <val>` | 0-255 | Color saturation |
| `set-sharpness <val>` | 0-255 | Edge sharpness |
| `set-hue <val>` | 0-255 | Color hue rotation |
| `set-sinter <val>` | 0-255 | Spatial noise reduction |
| `set-temper <val>` | 0-255 | Temporal noise reduction |
| `set-dpc <val>` | 0-255 | Dead pixel correction strength |
| `set-drc <val>` | 0-255 | Dynamic range compression |
| `set-highlight-depress <val>` | 0-255 | Highlight suppression |
| `set-backlight-comp <val>` | 0-10 | Backlight compensation |
| `set-defog <0\|1>` | 0/1 | Defog enable/disable |
| `set-defog-strength <val>` | 0-255 | Defog strength |
| `set-ae-comp <val>` | - | AE exposure compensation |
| `set-max-again <val>` | - | Maximum analog gain |
| `set-max-dgain <val>` | - | Maximum digital gain |
| `set-hflip <0\|1>` | 0/1 | Horizontal mirror |
| `set-vflip <0\|1>` | 0/1 | Vertical flip |
| `set-antiflicker <0\|1\|2>` | 0/1/2 | Off / 50Hz / 60Hz |
| `set-wb <mode> [r_gain] [b_gain] [g_gain]` | - | White balance mode + gains |

**White balance modes:** auto, manual, daylight, cloudy, incandescent,
flourescent, twilight, shade, warm_flourescent, custom.

**Green gain:** `g_gain` is last, not in RGB order, because `set-wb
<mode> <r> <b>` was the established form. It applies only to the T41,
whose manual mode writes the AWB RGB channel offsets rather than the AWB
gain pair. The header documents the gain pair as the manual control, but
on that part it is inert -- measured: it reads back exactly and never
reaches the picture -- while the offsets do take effect, and they have a
green. Omit `g_gain` and the green already set is kept; 0 means the same,
since zero is black and never worth writing. Neutral is near 1024 (a
fresh pipeline reads 1025/1017/1015). Leaving manual puts the offsets
back, so `set-wb auto` really does restore the picture.

**Platform availability:** dpc, drc, defog-strength require T23 or
T31. backlight-comp requires T23 or T31. highlight-depress works on
T20 through T31.

### ISP Queries

| Command | Description |
|---------|-------------|
| `get-isp` | All ISP settings as JSON |
| `get-wb` | White balance mode and gains |
| `get-rgb-coefft` | AWB RGB channel offsets (~1024 neutral); on the T41 this is what manual WB writes |
| `set-rgb-coefft <r> <g> <b>` | Set those offsets directly |
| `get-exposure` | Current exposure (gain, luma, EV, WB gains) |

### Snapshot Capture

| Command | Description |
|---------|-------------|
| `save jpeg <file> [ch]` | Save one JPEG from the jpeg ring (default: jpeg channel) |
| `save raw <file> [ch]` | Save one NV12 frame via transient SetFrameDepth+GetFrame |
| `save bayer <file>` | Save one pre-ISP sensor RAW frame (16-bit LE, w*h*2 bytes) |

The response is JSON: `status`, `file`, `format`, and for raw/bayer
`pixfmt`, `width`, `height`, `bytes`. Files land wherever `<file>`
points on the camera filesystem.

**Bayer per-platform mechanics.** Values are 12-bit (10-bit on some
sensors) MSB-aligned in 16-bit little-endian words; demosaic with
`ffmpeg -f rawvideo -pix_fmt bayer_bggr16le` (match the sensor's
pattern from `/proc/jz/isp/isp-m0`).

| Platform | Road | Boot requirement |
|----------|------|------------------|
| T20 | userland: fs chn0 flips to PIX_FMT_RAW for one frame | none |
| T21/T23/T31 | kernel: `saveraw` VIC debug node (self-buffered) | none |
| T30 | kernel: `snapraw` VIC debug node (ispmem-backed; the only verb its VIC has) | `ispmem=` bootarg carve sized sensor w*h*2 (6M covers 4MP); without it the command fails with an explicit error |
| T40/T41 | kernel: `snapraw 0` (internal buffer) | none |

On T20 the old-SDK buffer pool does not survive the RAW format
cycle: the daemon responds with `stream_restart: true`, then
re-execs itself for a clean pipeline. Streams drop for a few
seconds and clients reconnect (rsd reopens rings automatically).
On all other platforms streams are untouched. The pipeline must be
streaming for the kernel road (VIC active).

| Command | Description |
|---------|-------------|
| `clients` | List connected RTSP clients (IP, stream, transport, RTCP stats; a session with a backchannel carries its active talk-back codec -- `null` until the first accepted packet -- and its dropped-unknown-payload counter) |
| `set-backchannel-codecs <list>` | Restrict the talk-back offer, e.g. `pcmu,opus` (`""` = everything the build carries). Applies to the next SETUP -- no restart; unknown names are refused, excluding `pcmu` (the ONVIF baseline) warns. `status` names the effective offer |

```bash
raptorctl rsd set-backchannel-codecs pcmu,l16
raptorctl rsd set-backchannel-codecs ""        # back to the full offer
raptorctl rsd clients                          # ...,"backchannel":{"codec":"opus","unknown_dropped":0}
```

---

## 6. RAD -- Audio Daemon

### Audio Input

| Command | Description |
|---------|-------------|
| `set-codec <codec>` | Change codec (restarts pipeline) |
| `set-sample-rate <Hz>` | Input sample rate (restarts pipeline) |
| `set-volume <val>` | Input volume |
| `set-gain <val>` | Input gain (PGA) |
| `set-alc-gain <0-7>` | ALC gain level (T21/T31 only) |
| `mute` | Mute audio input |
| `unmute` | Unmute audio input |
| `ai-disable` | Tear down audio input pipeline |
| `ai-enable` | Bring up audio input pipeline |
| `set-aec <0\|1>` | Acoustic echo cancellation |
| `set-ns <0\|1> [0-3]` | Noise suppression (optional level) |
| `set-hpf <0\|1>` | High-pass filter |
| `set-agc <0\|1> [target] [comp]` | Auto gain control |

### Audio Output (Speaker)

| Command | Description |
|---------|-------------|
| `ao-set-volume <val>` | Speaker volume |
| `ao-set-gain <val>` | Speaker gain |
| `ao-set-sample-rate <Hz>` | Speaker sample rate |
| `ao-mute` | Mute speaker (soft fade) |
| `ao-unmute` | Unmute speaker (soft fade) |
| `ao-flush` | Discard queued speaker audio (hardware buffer clear) |
| `ao-drain` | Block until queued speaker audio has finished playing (3s cap) |
| `ao-disable` | Tear down audio output pipeline |
| `ao-enable` | Bring up audio output pipeline |

---

## 7. ROD -- OSD Renderer

### Global Control

| Command | Description |
|---------|-------------|
| `enable` | Enable OSD rendering (resumes after disable) |
| `disable` | Disable OSD (clears all regions to transparent) |
| `set-time-format <fmt>` | Set strftime format for `%time%` template variable |

### Element Management

| Command | Description |
|---------|-------------|
| `elements` | List all OSD elements |
| `add-element <name> [key=val]...` | Create element |
| `remove-element <name>` | Delete element |
| `set-element <name> [key=val]...` | Modify element properties |
| `show-element <name>` | Make element visible |
| `hide-element <name>` | Hide element |
| `set-var <name> <value>` | Set template variable |

### Style

| Command | Description |
|---------|-------------|
| `set-position <elem> <pos>` | Move element (named position or x,y) |
| `set-font-size <10-72>` | Global font size |
| `set-font-color <0xAARRGGBB>` | Global text color |
| `set-stroke-color <0xAARRGGBB>` | Global stroke outline color |
| `set-stroke-size <0-5>` | Global stroke width |

### Privacy and Receipt

| Command | Description |
|---------|-------------|
| `privacy [on\|off] [channel]` | Toggle privacy mode (black frame) |
| `receipt [name] <text>` | Append line to receipt overlay |
| `receipt-clear [name]` | Clear receipt display |

---

## 8. RIC -- IR-Cut Day/Night

| Command | Description |
|---------|-------------|
| `mode <auto\|day\|night>` | Set operating mode (controls GPIO + ISP) |
| `isp-mode <day\|night>` | Set ISP running mode only (no GPIO change) |
| `ircut <day\|night>` | Drive the IR-cut filter alone; the next auto transition reasserts |
| `ir850 <on\|off>` | Drive the 850nm LED bank alone (ignores the enable flag) |
| `ir940 <on\|off>` | Drive the 940nm LED bank alone (ignores the enable flag) |
| `set-threshold <key> <val>` | Tune day/night behavior live (see below) |
| `set-trigger <luma\|gain\|adc\|photo>` | Switch the trigger source live: the state machine re-arms (counters, baselines, probes), the current mode holds, and the new trigger judges from the next poll. Switching to `adc` fails outright if the ADC is unavailable — no silent fallback at runtime |
| `get-thresholds` | Show every tunable value and the trigger mode |

### Threshold Keys

Every key takes effect immediately and lands in the running config, so a
`config save` afterward persists the tune. Together with `set-trigger`
this makes the whole day/night policy runtime-tunable; only the wiring
(`gpio_*`, `adc_channel`) and `enabled` remain boot-time.

| Key | Range | Description |
|-----|-------|-------------|
| `night_luma` | 0-255 | AE luma below this triggers night (default 20) |
| `night_gain` | >= 0 | Gain above this triggers night regardless of luma (default 80000) |
| `day_gain_pct` | 1-100 | Gain below this % of baseline triggers day (default 25) |
| `probe_gain_pct` | 0-99 | Night: gain dip below this % of baseline lifts the LEDs for an ambient probe, 0 = off (default 90) |
| `probe_holdoff_sec` | 1-86400 | Minimum spacing between failed probes (default 60) |
| `probe_recheck_sec` | 0-86400 | Probe anyway after this long in night with no dip, 0 = off (default 600). Shortening it re-arms a running countdown |
| `night_threshold` | >= 0 | Gain-trigger: gain above this = night (default 40000) |
| `day_threshold` | >= 0 | Gain-trigger: gain below this = day (default 25000) |
| `adc_night` | 0-65535 | ADC below this = night (default 200); must stay below `adc_day` |
| `adc_day` | 0-65535 | ADC above this = day (default 600); must stay above `adc_night` |
| `photo_ev_night` | >= 0 | EV above this = dark (default 50000) |
| `photo_ev_deep` | >= 0 | EV above this = very dark (default 150000) |
| `photo_ev_day` | >= 0 | EV below this = bright (default 5000) |
| `photo_rgain_rec` | 0-65535 | AWB R-gain baseline, 0 = auto-calibrate |
| `photo_bgain_rec` | 0-65535 | AWB B-gain baseline, 0 = auto-calibrate |
| `ir850` | 0/1 | 850nm bank policy. Applies NOW when night is active: disabling drops the LED immediately (the window-reflection case), enabling lights it |
| `ir940` | 0/1 | 940nm bank policy, same immediate mid-night apply |
| `pulse_ms` | 1-1000 | Dual-GPIO ircut coil pulse width; the next actuation uses it |
| `hysteresis_sec` | 1-300 | Consecutive seconds before mode switch (default 5) |
| `poll_interval_ms` | 50-10000 | Sample interval in ms (default 1000) |
| `night_fps` | 0, 1-120 | Night sensor rate, 0 = off (default). Takes effect immediately when night is active; switching it off mid-night restores the base rate first |

---

## 9. RHD -- HTTP Server

| Command | Description |
|---------|-------------|
| `clients` | List connected HTTP clients (IP, type, duration) |

---

## 10. RWD -- WebRTC Server

| Command | Description |
|---------|-------------|
| `clients` | List WebRTC clients (IP, stream, ICE/DTLS state) |
| `share` | Show WebTorrent share URL |
| `share-rotate` | Generate new share key |

---

## 11. RMD -- Motion Detection

| Command | Description |
|---------|-------------|
| `sensitivity <0-4>` | Set IVS motion sensitivity |
| `skip-frames <N>` | Set IVS skip frame count |

---

## 12. RMR -- Recording

| Command | Description |
|---------|-------------|
| `enable` | Enable recording (continuous + clips) |
| `disable` | Disable all recording (closes active segments) |
| `start` | Start motion clip (motion/both modes) |
| `stop` | Stop motion clip |
| `sign-status` | Provenance signing state + device key fingerprint |
| `export-pubkey` | Ed25519 public key (hex) for `rverify -k` |
| `timelapse-status` | Timelapse state, current file, frame and byte counts |
| `timelapse-enable` | Enable timelapse capture (persists via `config save`) |
| `timelapse-disable` | Disable timelapse and close the current file |
| `timelapse-set <key> <value>` | Set `interval` or `playback_fps` live (a rate change rotates the file) |
| `timelapse-snap` | Capture one timelapse sample immediately |

Timelapse files are all-intra MP4s sampled from the recording stream
(`[timelapse]` config section): playback compresses real time by
`interval x playback_fps`, and each file carries the same SEI
timecodes and signature chain as any recording, so `rverify` works
unchanged.

`status` also reports `sign`, `sei_timecode`, and `timelapse` states. Verify signed
recordings with `rverify -k <pubkey> file.mp4` (`-t` summarizes the
embedded ST 0604 timecodes, `-T` dumps every frame's UTC).

`rverify` also accepts a JPEG: it verifies the RHD snapshot signature
(`[http] sign_snapshots`, same device key) and reports the EXIF
capture time. RHD's `status` reports `exif_timestamp` and
`sign_snapshots`.

---

## 13. RSP -- RTMP/RTMPS Push

| Command | Description |
|---------|-------------|
| `start` | Start push stream |
| `stop` | Stop push stream |
| `set-url <rtmp://...>` | Change target URL (disconnects and reconnects) |

---

## 14. RSR -- SRT Listener

| Command | Description |
|---------|-------------|
| `clients` | List connected SRT clients |

---

## 15. Examples

```sh
# Debug a daemon
raptorctl rvd set-log-level debug
# ... reproduce issue ...
raptorctl rvd set-log-level info

# Tune ISP for a dark scene
raptorctl rvd set-brightness 140
raptorctl rvd set-max-again 200000
raptorctl rvd set-defog 1
raptorctl rvd set-defog-strength 180

# Check current ISP state
raptorctl rvd get-isp

# Tune IR-cut for faster switching
raptorctl ric set-threshold hysteresis_sec 3
raptorctl ric set-threshold night_luma 25
raptorctl ric get-thresholds

# Pin video daemon to CPU1 on dual-core SoC
raptorctl rvd set-affinity 1

# Change encoder to VBR 4Mbps
raptorctl rvd set-rc-mode 0 vbr 4000000

# Request keyframe (useful after config change)
raptorctl rvd request-idr

# Add OSD text element
raptorctl rod add-element cam1 type=text template="Camera 1" position=top-left

# Disable OSD temporarily, change time format, re-enable
raptorctl rod disable
raptorctl rod set-time-format "%H:%M:%S"
raptorctl rod enable

# Switch push target live (same scheme only; rtmp<->udp needs a restart)
raptorctl rsp set-url rtmp://live.twitch.tv/app/new_stream_key
raptorctl rsp set-url udp://192.168.1.50:5600

# Mute audio, disable echo cancellation
raptorctl rad mute
raptorctl rad set-aec 0

# Force night mode for testing
raptorctl ric mode night

# Save all runtime changes to disk
raptorctl config save
```

---

## 16. JSON Mode

For scripting or when you need structured output, use `-j`:

```sh
# Single command
raptorctl -j '{"daemon":"rvd","cmd":"get-isp"}'

# Batch multiple commands (executes sequentially)
raptorctl -j '[
  {"daemon":"rvd","cmd":"set-brightness","value":128},
  {"daemon":"rvd","cmd":"set-contrast","value":140}
]'

# Commands with named args
raptorctl -j '{"daemon":"rvd","cmd":"set-bitrate","channel":0,"value":3000000}'
raptorctl -j '{"daemon":"ric","cmd":"set-threshold","key":"night_luma","value":30}'
```

Responses are JSON objects with a `"status"` field (`"ok"` or
`"error"`). Query commands include additional fields with the
requested data.

---

## 17. Notes

- **Persistence:** Runtime changes are in-memory only until
  `raptorctl config save` writes them to disk. A daemon restart
  without saving discards changes.
- **Sockets:** Each daemon listens on `/var/run/rss/<name>.sock`.
  raptorctl auto-detects which socket to use from the command.
- **Timeout:** Control socket operations have a 5-second timeout.
  Don't send commands that would block (the daemon handles them
  synchronously in its main loop).
- **Error handling:** If a daemon is not running, raptorctl reports
  "connect failed". If a command is unknown, the daemon returns an
  error response.
- **SoC capabilities:** Some commands are only available on certain
  platforms. Unsupported commands return an error with "not supported"
  rather than crashing.

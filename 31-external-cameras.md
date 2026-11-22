# External Cameras (RCD)

`rcd`, the Raptor Camera Daemon, brings cameras that are not wired to the
SoC's sensor interface into raptor as extra sensors. A camera in slot N
publishes already-encoded video into the rings `sN_main` and `sN_sub`, so
RSD (RTSP `stream s1`/`s2`), RWD and RMR use it exactly like an on-board
second sensor ([27 - Multi-Sensor](27-multi-sensor.md)). The first camera
type is the Novatek **EthCam** rear module of dashcams such as the USB529Pro
860 (NT98529); the design leaves room for others, such as Ingenic-based
Ethernet cameras.

---

## 1. Architecture

```
            camera link (eth0, 192.168.0.0/24)
  ┌──────────┐  cmd 8899 / data 8887+8888   ┌──────────────── rcd ────────────────┐
  │ camera 1 │ ───────────────────────────► │ thread: driver session ─┐           │
  │ camera 2 │ ───────────────────────────► │ thread: driver session ─┤  core     │
  └──────────┘                              │                         ▼           │
                                            │  rcd_publish: clock map, src_seq,   │
                                            │  stream info, UTC mapping           │
                                            └──────┬──────────────────────┬───────┘
                                               s1_main/s1_sub         s2_main/s2_sub
                                                   ▼                      ▼
                                         rsd · rwd · rmr · ringdump (unchanged)
```

### 1.1 Core and drivers

The **core** (`rcd_main.c`, interface in `rcd.h`) owns everything that does
not depend on the kind of camera:

- config (`[cameras]`, `[camera1]`..`[camera4]`), one thread per camera
- the rings and how frames enter them: timestamps, `src_seq`, stream info,
  UTC mapping, per-stream counters
- stall detection (no frame for 3 s ends the session) and reconnects
  (2 s pause, or at once on `raptorctl rcd reconnect`)
- camera power: a power cycle before the next session for cameras that
  stream once per boot, and after three sessions in a row that never reach
  streaming (a wedged camera)
- the camera link: interface address, DHCP server, power GPIO
- the control socket `/var/run/rss/rcd.sock`

A **driver** only talks to its camera:

```c
typedef struct {
	const char *name;                     /* config: type = <name> */
	int (*init)(rcd_cam_t *c, rss_config_t *cfg, const char *section);
	int (*session)(rcd_cam_t *c);         /* <0 camera failed, 0 asked to stop */
	void (*cleanup)(rcd_cam_t *c);
	bool one_session_per_boot;            /* core power-cycles between sessions */
} rcd_driver_t;
```

`session()` connects, calls `rcd_stream_open()` for each stream it has,
`rcd_streaming()` once frames flow, then loops: `rcd_publish()` each access
unit, `rcd_keyframe_wanted()` to forward consumer IDR requests, `rcd_tick()`
for housekeeping, until `rcd_should_run()` turns false or the camera fails.

### 1.2 Adding a camera type

1. `rcd/rcd_drv_<type>.c` with a `const rcd_driver_t rcd_drv_<type>`; keep
   protocol parsing in pure functions (`rcd_<type>.c`) so unit tests cover
   it without sockets.
2. Declare it in `rcd.h`, add it to `drivers[]` in `rcd_main.c`, add the
   sources to `rcd/Makefile` and `build-asan.sh`.
3. A fake camera in `tests/` that speaks the protocol (with its quirks) and
   a leg in `tests/test-rcd.sh`.

What a driver must hand the core:

- **Annex-B access units**, parameter sets (SPS/PPS, VPS for H.265) in
  front of every keyframe, as `rss_iov_t` pieces (no copy needed).
- **The camera's own capture timestamp** in microseconds if it has one;
  `rcd_tsmap` maps it onto `CLOCK_MONOTONIC` (lowest observed transit delay,
  slews at most 2 ms per 10 s for drift, never goes backward, counts camera
  reboots). Pass `RCD_TS_LOCAL` when the camera has no clock: arrival time
  is used and loss accounting is off for that stream.
- **Frames the camera sent but that are unusable** go to `rcd_drop()`, so
  they count as `drop_publish`, not as a gap.

For an Ingenic/thingino camera on Ethernet the natural drivers are an RTSP
pull (the camera's RSD already sends Annex-B with in-band parameter sets,
and RTCP sender reports give the clock mapping) or a raptor-to-raptor ring
transport. Either keeps the core unchanged.

### 1.3 Timestamps, UTC and loss accounting

Ring timestamps are `CLOCK_MONOTONIC` microseconds, like RVD's media clock,
and every rcd ring carries the monotonic-to-UTC mapping (`rss_ring_set_utc`,
refreshed every 5 s, status locked when NTP/GPS synced the clock). RMR and
RSD therefore write the same per-frame UTC SEI timecodes into recordings and
RTSP as for the on-board sensor, signing covers the rear files, and `rverify`
checks them unchanged.

`src_seq` comes from the camera timestamps (`rcd_seq`): a stamp that does
not advance is taken as stale and the frame gets the previous one plus a
period; gaps are measured from there. A frame missing in the camera's
sequence shows up as `drop_source` in the ring header (`ringdump s1_main`)
and in RMR's gapless accounting. Sequence and clock mapping persist across
sessions, so frames missed during a reconnect also count; a camera clock
that goes back by more than a second is a camera restart
(`camera_restarts` in `rcd status`), not loss.

### 1.4 The camera link

With `iface` set, rcd configures it (default `192.168.0.3/24`, the address
EthCam modules expect), runs BusyBox `udhcpd` on it with a static lease per
camera `mac`, restarts it if it dies, and switches the camera supply GPIO.
`power_cycle` (default on) cuts the supply for 3 s at start, so every run
begins with freshly booted cameras. The same GPIO is the reconnect path
for cameras that stream once per boot (EthCam, §4.3) and the recovery for a
camera that answers but never gets to streaming. The supply feeds every
camera on the link, so all of them restart together.

On the USB529Pro 860 the only Ethernet port is the rear connector, so
`eth0` belongs to the cameras and Wi-Fi is the uplink. Several cameras share
the link through an unmanaged switch; they are told apart by MAC through
DHCP.

---

## 2. Configuration

```ini
[cameras]
enabled = true
iface = eth0
local_ip = 192.168.0.3
dhcp = true
dhcp_start = 192.168.0.12
dhcp_end = 192.168.0.20
power_gpio = 116          # USB529Pro 860 rear connector 5 V
power_cycle = true

[camera1]
enabled = true
type = ethcam
ip = 192.168.0.12
mac = 00:eb:fc:ec:50:32   # leases ip to this module (DHCP firmware, §4.6)
slot = 1
codec = h265
sub = true
```

Full key list: [23 - Configuration](23-rss-config.md). Slots are 1 and 2
(the consumers know `s1_*` and `s2_*`); two cameras cannot share a slot.

## 3. Control

```
raptorctl rcd status          # per camera: state, ip, mac, fw, sessions, error,
                              # streams: size, fps, frames, keyframes, bytes, dropped
raptorctl rcd reconnect [n]   # new session for camera n (all without n)
ringdump s1_main              # ring header: codec, size, src_seq loss counters
ringdump s1_main -d -s | ffprobe -f h264 -   # inspect the stream
```

`restart` is the common command that restarts the whole daemon, hence
`reconnect` for one camera.

### 3.1 Recording

RMR records the rear camera next to the front one when both rings are
listed in `[recording]`:

```ini
[recording]
streams = main, s1_main:R
```

Every front file gets a rear partner with the same name plus `_R`
(`2026-11-22/16-59-00.mp4` and `2026-11-22/16-59-00_R.mp4`, the same in
`clips/`), split at the same wall-clock boundaries, with its own
signature chain, the shared GPS records and audio, and SEI timecodes
from the rear ring's UTC mapping. A clip event opens both. Both share
the quota, so cleanup removes a pair together. The rear is optional at
run time: with the camera unplugged or RCD stopped the front records
on undisturbed, the rear track goes idle after 2 s and closes its file
at the next boundary, and it starts writing again (from the next
keyframe) when frames return.
`raptorctl rmr status` lists both under `tracks`. Each stream's
pre-buffer (`mode = both`/`motion`) is sized from its own byte rate:
~14 MB for a 16 Mbit/s rear at `prebuffer_sec = 5`; `prebuffer_max_mb`
bounds it on small boards. Details: [20 - Architecture](20-rss-architecture.md) (RMR).

---

## 4. Novatek EthCam reference

Everything below was established on a USB529Pro 860 rear module (NT96562 +
Sony IMX675, firmware `2019-12-12-Ver1.00`, model string
`562_CARDV_ETHCAM_TX_EVB`) against the stock receiver (`cardv`) and the
Novatek SDKs. "Stock" means the dashcam's own firmware.

### 4.1 Link and discovery

- The module is a TCP server; the receiver connects. Stock gives the
  receiver `192.168.0.3`, the module `.12` (a second module `.13`).
- After boot the module sends UDP to `192.168.0.3:2222`:
  `cliconnipnotify <ip> <mac0-3> <mac4-5>` (decimal ints, little-endian
  packing, no NUL), repeated about every 30 s. rcd does not need it.
- The MAC is derived from the SoC's eFuse unique ID at boot:
  `[id_h>>8 & 0xfe, id_h & 0xff, id_l>>24, id_l>>16, id_l>>8, id_l]`, so
  every module has its own stable MAC; it is not stored in flash.

### 4.2 Command channel (TCP 8899)

- Request: `"<module ip>:<port>/?custom=1&cmd=<N>&par=<P>\0"`, NUL included,
  no other framing. `<port>` names the stream the command acts on (8887
  record, 8888 display, 8899 the module), not the socket it travels on.
- Reply: the same `ip:port` prefix, then XML, no length field:
  `<Function><Cmd/><Status/>[<Value/>|<String/>]</Function>` or
  `<LIST><Cmd/><ITEM/>...</LIST>`. Status: 0 ACK, 1 OK, 2 CONTINUE,
  3 timeout, -1 error, -256 unknown command.
- **Two-phase commands** (9014, 9015, 9027, 9032, ...): the module answers
  Status 2, then reads the binary payload **with one receive**. Send each
  payload in one write; a payload split over segments is read short and the
  command fails (9027 answers -1).

| Cmd | par / payload | Reply observed |
|---|---|---|
| 9001 startup | 0 | Status 1 |
| 9007 parameter sets | on 8887/8888 | LIST of bytes: Annex-B SPS+PPS (H.264) |
| 9008 stream start | on 8887/8888 | Status 1; frames follow at once |
| 9009 stream stop | `par=1` both streams | Status 1 |
| 9012 record info | on 8887 | LIST: width 2560, height 1440, fps 30, tbr 2048000, 0, GOP 30, codec (2 H.264 / 3 H.265), ... |
| 9014 time sync | 44-byte `struct tm`, full year, month 1-12 | Status 1 (sets the module clock and its own date stamp) |
| 9015 menu sync | 32 bytes `{Size, WDR, EV, DateImprint, SensorRotate, SensorMirror, Codec, TimeLapse}` | Status 1; **Size is ignored** (only printed); codec (word 6) must match 9032; `par=1` re-enters the mode and drops the command socket. Also saves settings |
| 9016 keyframe | on 8887/8888 | Status 1; next frame on that stream is an I frame |
| 9017 display info | on 8888 | LIST: 0, 0, tbr 153600, fps 30, 640, 360, 1800, codec |
| 9020 / 9021 | firmware send / start (§4.6) | |
| 9023 power off | 0 | Status 1, but the module has no power latch: it stops responding until power-cycled. Never send it |
| 9025 version | 0 | String `2019-12-12-Ver1.00` when the request names `.12`, `...Ver1.02` for `.13` (the module answers per receiver path) |
| 9026 IP reset | 0 | Status 1, no lasting effect (§4.5) |
| 9027 system info | 12 bytes `{PullModeEn=0, CloneDisplayPathEn=1, 0}` | Status 76 (`12 | type<<4`, type 4 = NT96562). Without CloneDisplayPathEn the display stream stays silent |
| 9032 codec | 12 bytes `{VCodec 0=H.264 1=H.265, 4, 0}` | Value 12 = accepted |
| 9034 audio capture | 1 | Status 1, but no audio is ever sent (§4.4) |

### 4.3 Data channels (TCP 8887 record, 8888 display)

- Connect after the handshake, then send 9008 for the stream (display
  first, as stock does). The receiver never writes to these sockets, but
  **must not half-close them**: the module drops a client that shuts down
  its sending side and never sends a frame. (BusyBox `nc` with its input at
  EOF does exactly that, and looks like a module that will not stream.)
- **The module streams once per power-up.** After a stream has stopped
  (9009, or its client went away), a new 9008 answers OK and nothing
  arrives; a third session wedges the command handler (connections are
  accepted, no command is answered). Stock never meets this because its
  receiver keeps one connection for good; rcd power-cycles the module
  between sessions (back streaming in 9-10 s).
- On a fresh stream the first three frames repeat the start timestamp and
  the fourth jumps 100 ms; no frame is missing.
- One record per frame:

| Offset | Size | Field |
|---|---|---|
| 0 | 8 | timestamp, u64 big-endian, microseconds of the module's clock (not UTC) |
| 8 | 3 | size of the body: 20 bits, little-endian (bytes 8, 9, low nibble of 10) |
| 11 | 1 | frame index in the GOP; 1 = I frame |
| 12 | 1 | checksum A: byte sum of the first 64 body bytes (0 if body <= 64) |
| 13 | 1 | checksum B: byte sum of the last 64 body bytes |
| 14 | 1 | H.265 I frames: length of VPS+SPS+PPS & 0xff |
| 15 | 1 | header type, always 2 |
| 16 | size | body, Annex-B with 4-byte start codes |

- **H.264 I frames** are preceded by `u16le length, SPS, u16le length, PPS`
  (each Annex-B with start code); the body is the IDR slice.
  **H.265 I frames** carry VPS/SPS/PPS inline at the start of the body.
- Measured: 2560x1440 H.264 High / H.265 Main at a steady 30 fps (33.33 ms
  steps), about 16 Mbit/s for either codec; display 640x360@30 about
  0.7 Mbit/s. No audio, no B frames, single slice.

### 4.4 What the stock firmware cannot do

- **No bitrate, fps or resolution control.** There is no command, and the
  Size field of 9015 is ignored (the line is commented out in the SDK's TX
  code). Only the codec can be chosen.
- **No audio.** 9034 is accepted, but the TX code never registers the audio
  callback and stock's receiver demux for it is disabled.
- **No DHCP and a fixed address** (next section).

All three are firmware changes: see §4.6 for how the firmware is rebuilt and
loaded over the cable. **Bitrate and GOP: done this way** (§4.8). No microphone
on the USB529Pro 860 rear module board, so audio is moot there.

### 4.5 Firmware and flash

- SPI NOR, XMC XM25QH32 (4 MB). Partitions (device tree at 0x10000):
  loader 0x0, fdt 0x10000, fdt.restore 0x40000, fdt.app 0x70000,
  uboot 0xa0000, rtos 0x100000 (0x2a0000), pstore 0x3a0000 (0x60000).
- `rtos` is a BCL1 container (`BCL1`, checksum, algorithm 0x0b, unpacked
  and packed sizes) around an LZMA-alone stream at +0x10; FreeRTOS image,
  loads at 0x10000.
- `pstore` holds section `SYSP` (index entries carry byte-sum checksums of
  the entry itself; the data has none) = the UI settings struct; the
  receiver-facing IP is `UIParameter[155]` (`FL_ETHCAM_TX_IP_ADDR`).
- The boot code reads the saved IP and uses DHCP when it is 0, but the
  vendor added a line right after loading settings that forces the flag to
  `192.168.0.12`. pstore edits, 9026 (which only clears it in RAM) followed
  by a settings save (9015 and 9040 do save), and the UART `ts ipset`
  command therefore cannot change the address. The DHCP client is in the
  image, unreachable.
- 9023's shutdown path has the settings save removed and an empty
  power-off.

### 4.6 DHCP firmware and loading it over Ethernet

The patch: in `EthCamNet_SetTxIPAddr` replace the `bl SysGetFlag` that loads
the saved IP with `mov r0, #0`, so `init_eth0` always takes the DHCP branch
(leaving the boot-time forcing alone keeps a later settings save from
pinning a leased address). On the firmware above: rtos offset 0x412f18,
`320c00eb` -> `0000a0e3`.

Build it only with the vendor tools from the NA51089 SDK, so the result is
what Novatek's own build would produce:

```
encrypt_bin SUM rtos.raw 0x250 FW96562A      # image checksum (tag at 0x250, sum at 0x26e)
bfc c lzma rtos.raw rtos-main.bin 0 0 -align # BCL1/LZMA container
nvtpack -dtb nvt-evb.bin                     # FW96562A.ini from the module's own fdt
#   keep ITEM01 (fdt, unchanged) and ITEM06 (rtos); set the others to 0
nvtpack -fw FW96562A.ini                     # packed/FW96562A.bin, all-in-one package
```

Run on the unpatched decompressed rtos, the first two commands reproduce the
module's flash image byte for byte; that is the check that the pipeline is
right.

Loading (the module's Ethernet update path, as stock pushes
`EthcamTxFW.bin`): `9020 par=<package size>` -> Status 2; send the package on
the command socket -> Status 1 once all bytes are in; `9021` -> Status 2.
The module writes flash and reboots by itself within seconds. Afterwards it
asks for DHCP on every power-up; give it a static lease by MAC. Without a
DHCP server it retries forever, which is why rcd always runs one on the
camera link.

Only send a package built from the same firmware the module reports (9025):
it replaces the whole rtos partition. Keep a full dump of the NOR before the
first update; a module that does not boot is restored with an SPI
programmer.

### 4.8 Bitrate, GOP and size in the firmware

The record stream's settings are one entry of the movie size table
(`g_MovieSizeTable`, 26 entries of 0x6c bytes: size `{w, h, fps, tbr, dar,
ratio}`, AQ, CBR). On the firmware above, entry 7 (2560x1440P30):

| Address | Field | Stock |
|---|---|---|
| 0x687bc0 | target bitrate (tbr), bytes/s | 2048000 |
| 0x687bf0 | CBR byte rate | 2048000 |
| 0x687bf4 | CBR GOP | 30 |

Rate control runs at the CBR byte rate x 8; keep tbr >= the CBR rate (the
encoder layer rejects more). Changing the three words (rebuilt and loaded as in
§4.6) takes effect at the next power-up, and 9012 reports the new values by
itself. Verified: 1000000 bytes/s gave a measured 8.0 Mbit/s at 30 fps, GOP
30, no loss. Above the stock rate the encoder buffer grows (tbr x
`VdoEncBufSec` 3 s at 0x685950); lower it to 2 for 24 Mbit/s.

The size itself is not taken from settings: every boot the firmware picks
the size preset from a board strap (2560x1440 or 1920x1080) and sets the
sensor pipeline size to match, which is why 9015's Size does nothing. The
display stream is entry 18 (640x360, GOP 1800: one keyframe a minute).

### 4.7 Pitfalls met on the way

| Symptom | Cause |
|---|---|
| All commands OK, no frames | data client half-closed its socket |
| 9027 answers -1 | binary payload sent in pieces |
| connections accepted, no replies at all | module wedged: a third stream session since boot, a command in flight when its connection closed, or an unclean power-up. Power-cycle it |
| second session: 9008 OK, no frames | the module streams once per boot: power-cycle between sessions |
| a few frames "lost" at every stream start | the first frames repeat the start timestamp (§4.3); count by stamp estimate, not raw stamp |
| command socket closes after 9015 | `par=1` makes the module re-enter its mode |
| module stops answering after 9023 | no power latch: it waits for a power cycle |
| capture fills RAM on the dashcam | `/tmp` is tmpfs on an 89 MB board: stream captures to the host or SD |

---

## 5. Testing

- `tests/test_ethcam.c`: protocol unit tests (command builder, reply parser
  across every cut point, record parser for H.264 and H.265 records,
  checksums, desync, the clock mapper's tracking, slewing, monotonicity and
  reboot handling).
- `tests/fake_ethcam.py`: a fake module that speaks the protocol with the
  quirks of §4.2-4.3 (split payloads rejected, half-closed data clients
  dropped, streams only after 9008, 9016 forces an I frame, the first three
  frames repeat the start stamp) and replays
  clips that ffmpeg makes at test time. `--skip N` and `--corrupt N` inject
  missing and damaged frames.
- `tests/test-rcd.sh` (in `test-all.sh`): two fake modules at once, H.264 in
  slot 1 and H.265 in slot 2. Checks stream info, that the rings decode,
  zero loss on a clean link, that skipped frames do count as `drop_source`
  and damaged ones as dropped, keyframe requests, loss detection,
  reconnect, the `reconnect` command, clean exit, no sanitizer reports.
- `tests/test-rmr-multi.sh` (in `test-all.sh`): RMR recording the mock
  RVD's `main` and the fake module's `s1_main` together. Checks paired
  segment and clip files that verify (signature, timecodes, GPS),
  audio in both, zero loss on both tracks, the front recording on while
  the module is gone and the rear resuming when it returns, and that a
  one-stream config keeps its file names and status fields.
- Not covered by the fake: once-per-boot streaming and the wedge (the
  power GPIO needs real hardware). Verified on a USB529Pro 860: three
  `reconnect`s in a row, each back streaming in 9-10 s, 0 lost or dropped
  frames, each reboot counted as a camera restart.

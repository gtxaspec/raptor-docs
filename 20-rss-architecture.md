# Raptor Streaming System -- Architecture

System-level architecture of RSS: daemon decomposition, IPC primitives,
pipeline model, startup sequence, crash recovery, and resource budgets.

RSS replaces the monolithic prudynt with a set of single-purpose daemons
communicating through shared memory and Unix sockets. Each daemon can be
restarted independently, and the failure domain of any single component
is isolated from the rest of the pipeline.

---

## 1. Daemon Decomposition

### 1.1 Producer Daemons

These daemons own hardware resources and write encoded frames into SHM
ring buffers. Exactly one instance of each runs per camera.

#### RVD -- Video Daemon

- **Role**: Initialize the HAL, configure ISP/framesource/encoder
  pipeline, pull encoded frames from the encoder, and publish them to
  SHM ring buffers (one per stream channel).
- **HAL dependency**: RVD is the HAL owner. It calls `rss_hal_create()`,
  `init()`, and builds the full video pipeline:
  `sensor -> FS -> OSD -> Enc`.
- **Outputs**: Two SHM rings by default (main stream, sub stream).
  Optionally a third for JPEG snapshots.
- **OSD interaction**: RVD creates OSD groups and regions, then listens
  on an eventfd for bitmap updates from ROD. When the eventfd fires,
  RVD reads the active OSD SHM buffer and calls
  `osd_update_region_data()` through the HAL.
- **Why separate**: The video pipeline is the critical path. Isolating
  it means audio glitches, OSD rendering delays, or RTSP client
  disconnects cannot stall frame production.

#### ROD -- OSD Overlay Daemon

- **Role**: Render OSD overlays (timestamp, camera name, custom text,
  logo) into BGRA bitmaps and publish them to OSD SHM double-buffers.
- **HAL dependency**: None. ROD does not touch the HAL. It renders
  bitmaps in userspace using FreeType2 for text and raw BGRA blitting
  for logos.
- **Outputs**: OSD SHM double-buffer (one per OSD region per stream
  channel). Notifies RVD via eventfd after each update.
- **Why separate**: Font rendering is CPU-intensive relative to the
  frame path. Isolating it lets ROD run at a lower priority without
  risking frame drops. ROD can also be killed and restarted to change
  OSD layout without interrupting the video stream (existing overlay
  just freezes until ROD restarts).

#### RAD -- Audio Daemon

- **Role**: Initialize audio input through the HAL, read PCM frames,
  optionally encode (G.711a, AAC, Opus), and publish encoded audio
  frames to an audio SHM ring buffer.
- **HAL dependency**: Calls `audio_init()`, `audio_read_frame()`, and
  optionally `audio_register_encoder()` through the HAL.
- **Outputs**: One SHM ring (audio).
- **Why separate**: Audio runs on a fixed-period schedule (typically
  20ms frame intervals). Isolating it from video prevents encoder
  stalls from causing audio discontinuities and vice versa. RAD can
  also be disabled entirely on cameras without microphones.

### 1.2 Consumer Daemons

These daemons read from SHM ring buffers and deliver frames to external
sinks. Multiple consumers can attach to the same ring simultaneously.

#### RSD -- RTSP Server

- **Role**: RTSP/RTP server. Reads video and audio SHM rings, packetizes
  into RTP, serves to clients via RTSP.
- **Inputs**: Main video ring, sub video ring, audio ring.
- **Dependencies**: librss_ipc (SHM ring consumer API). Optionally
  libevent or poll-based event loop. No HAL dependency.
- **Protocol**: Standard RTSP (DESCRIBE/SETUP/PLAY/TEARDOWN). Two
  mount points: `/main` (stream 0) and `/sub` (stream 1). Audio track
  included when RAD is running.
- **Why separate**: RTSP client management (socket I/O, RTP packetization,
  RTCP) is complex and has its own failure modes (slow clients, network
  errors). Isolating it means a misbehaving RTSP client cannot affect
  frame production or recording.

#### RMR -- Recorder

- **Role**: Mux video and audio frames into MP4/MKV files on local
  storage (SD card, NFS).
- **Inputs**: Main video ring (or sub), audio ring.
- **Dependencies**: librss_ipc, lightweight muxer (custom or libavformat).
- **Why separate**: File I/O can block (SD card write stalls). Isolating
  the recorder prevents storage latency from affecting the live stream.
  RMR can be started/stopped via raptorctl for event-triggered recording.

#### RSP -- Stream Push

- **Role**: Push RTMP/RTSP streams to an external server (YouTube,
  cloud NVR, custom endpoint).
- **Inputs**: Main video ring, audio ring.
- **Dependencies**: librss_ipc, libcurl or custom RTMP client.
- **Why separate**: Network push is inherently unreliable (upstream
  bandwidth, server availability). Isolating it prevents network stalls
  from affecting local recording or RTSP serving.

#### RV4 -- V4L2 Bridge

- **Role**: Expose the video stream as a V4L2 output device
  (`/dev/videoN`) for local consumers (motion detection, ML inference).
- **Inputs**: Sub video ring (decoded or raw NV12 from framesource).
- **Dependencies**: librss_ipc, V4L2 output device kernel support.
- **Why separate**: V4L2 clients may misbehave or consume frames slowly.
  The bridge isolates them from the encoding pipeline.

### 1.3 Control Daemons

These daemons control hardware peripherals and do not process frame data.

#### RIC -- IR/Day-Night Control

- **Role**: Monitor ISP exposure levels via `isp_get_exposure()`, drive
  IR-cut filter and IR LED GPIOs based on configurable thresholds, switch
  ISP running mode between day and night.
- **HAL dependency**: Calls `isp_get_exposure()`, `isp_set_running_mode()`,
  `ircut_set()`, `gpio_set()`.
- **IPC**: Reads exposure data from HAL on a polling interval (typically
  1s). Receives override commands via Unix control socket (force day,
  force night, auto).
- **Why separate**: Day/night switching involves GPIO manipulation, ISP
  mode changes, and hysteresis logic. Isolating it keeps the video daemon
  simple and allows the control algorithm to be replaced or tuned without
  restarting video.

#### RMC -- Motor Control

- **Role**: Drive pan/tilt/zoom motors via GPIO or UART for PTZ cameras.
- **HAL dependency**: `gpio_set()` for stepper motors, or direct UART
  for motor driver ICs.
- **IPC**: Unix control socket for PTZ commands (move, preset, home).
- **Why separate**: Motor control involves blocking sequences (step
  delays, homing routines) that must not stall any other daemon.

### 1.4 CLI Tool

#### raptorctl

- **Role**: Command-line interface for daemon management and diagnostics.
- **Communication**: Connects to each daemon's Unix control socket.
- **Commands**:
  - `raptorctl status` -- show running daemons and stream stats
  - `raptorctl rvd dump-frame` -- dump one encoded frame to stdout
  - `raptorctl rvd request-idr` -- request an IDR frame
  - `raptorctl rvd set-bitrate <bps>` -- change encoder bitrate
  - `raptorctl rod set-text <region> <text>` -- update OSD text
  - `raptorctl ric mode <auto|day|night>` -- set day/night mode
  - `raptorctl rmc move <direction> <steps>` -- PTZ command
  - `raptorctl rmr start|stop` -- start/stop recording

---

## 2. IPC Primitives

### 2.1 SHM Ring Buffer (`rss_frame_ring`)

The primary data transport between producers and consumers. Lock-free,
single-producer multi-consumer, designed for zero-copy frame passing.

#### Struct Layout

```c
/*
 * rss_frame_ring.h -- SHM ring buffer for encoded frames.
 *
 * Memory layout (single contiguous mmap):
 *
 *   +-------------------+  offset 0
 *   | rss_ring_header_t |  control block (cache-line aligned)
 *   +-------------------+  offset 4096 (page-aligned)
 *   | slot[0]           |  rss_ring_slot_t
 *   | slot[1]           |
 *   | ...               |
 *   | slot[N-1]         |
 *   +-------------------+  offset 4096 + N * sizeof(slot)
 *   | data region        |  raw frame payload storage
 *   +-------------------+
 */

#define RSS_RING_MAX_SLOTS    64
#define RSS_RING_SHM_PREFIX   "/rss_ring_"   /* shm_open name prefix */

typedef struct {
    /* Producer-written, consumer-read */
    _Atomic uint64_t  write_seq;        /* monotonic sequence, wraps at UINT64_MAX */
    uint32_t          slot_count;       /* number of slots (power of 2) */
    uint32_t          data_size;        /* total data region size in bytes */

    /* Data region allocator (producer only) */
    _Atomic uint32_t  data_head;        /* next write offset in data region */

    /* Stream metadata (set once at init) */
    uint32_t          stream_id;        /* 0=main, 1=sub, 2=jpeg, 0x10=audio */
    uint32_t          codec;            /* rss_codec_t value */
    uint32_t          width;            /* video width (0 for audio) */
    uint32_t          height;           /* video height (0 for audio) */
    uint32_t          fps_num;
    uint32_t          fps_den;

    /* Magic + version for consumer validation */
    uint32_t          magic;            /* RSS_RING_MAGIC */
    uint32_t          version;          /* RSS_RING_VERSION */
} rss_ring_header_t;

#define RSS_RING_MAGIC    0x52535352   /* "RSSR" */
#define RSS_RING_VERSION  1

typedef struct {
    _Atomic uint64_t  seq;              /* sequence number when written */
    uint32_t          data_offset;      /* offset into data region */
    uint32_t          data_length;      /* frame payload size in bytes */
    int64_t           timestamp;        /* capture timestamp (us) */
    uint16_t          nal_type;         /* rss_nal_type_t for video, codec ID for audio */
    uint8_t           is_key;           /* 1 if IDR / keyframe */
    uint8_t           _pad;
} rss_ring_slot_t;
```

#### Producer API

```c
/*
 * Producer side -- used by RVD (video) and RAD (audio).
 *
 * rss_ring_create(): create and initialize a new ring.
 *   name:       SHM name suffix (e.g. "main", "sub", "audio")
 *   slot_count: number of slots (must be power of 2, typically 16-32)
 *   data_size:  data region size in bytes (typically 2-4MB for video)
 *
 * rss_ring_publish(): publish a frame to the ring.
 *   The producer copies frame data into the data region at data_head,
 *   fills the slot metadata, then atomically increments write_seq.
 *   If the data region wraps, the oldest data is overwritten (consumers
 *   detect this via sequence gap).
 */
rss_ring_t *rss_ring_create(const char *name, uint32_t slot_count,
                             uint32_t data_size);
void         rss_ring_destroy(rss_ring_t *ring);

int          rss_ring_publish(rss_ring_t *ring,
                              const uint8_t *data, uint32_t length,
                              int64_t timestamp, uint16_t nal_type,
                              uint8_t is_key);
```

#### Consumer API

```c
/*
 * Consumer side -- used by RSD, RMR, RSP, RV4.
 *
 * rss_ring_open(): attach to an existing ring by name.
 *   Returns NULL if the ring does not exist yet. Consumers should
 *   retry with backoff until the producer creates the ring.
 *
 * rss_ring_read(): read the next frame for this consumer.
 *   Each consumer tracks its own read_seq. If read_seq < write_seq,
 *   the consumer has fallen behind and reads the next available slot.
 *   If the consumer has fallen behind far enough that its data was
 *   overwritten, rss_ring_read() returns RSS_EOVERFLOW and advances
 *   read_seq to the oldest valid slot.
 *
 *   The returned pointer is into the SHM data region (zero-copy).
 *   The consumer must finish processing before the producer
 *   overwrites that region. For slow consumers, copy the data.
 *
 * rss_ring_wait(): block until a new frame is available.
 *   Uses eventfd notification from the producer. The producer
 *   writes to the eventfd after each publish.
 */
rss_ring_t *rss_ring_open(const char *name);
void         rss_ring_close(rss_ring_t *ring);

int          rss_ring_read(rss_ring_t *ring, uint64_t *read_seq,
                           const uint8_t **data, uint32_t *length,
                           rss_ring_slot_t *meta);

int          rss_ring_wait(rss_ring_t *ring, uint32_t timeout_ms);
```

#### Zero-Copy Frame Passing

The data region is a circular byte buffer. The producer writes frame
payloads sequentially, wrapping around when the end is reached. Consumers
read directly from the mmap'd region without copying. The slot's
`data_offset` and `data_length` describe where the payload lives.

Consumers that cannot keep up (read_seq falls more than slot_count behind
write_seq) receive `RSS_EOVERFLOW` and must skip to the next keyframe.
This is the normal backpressure mechanism -- slow consumers lose frames
rather than blocking the producer.

#### Slot Management

Slots are indexed as `write_seq % slot_count`. The producer fills
slot metadata, copies payload data, issues a release fence, then
atomically stores the new `write_seq`. Consumers load `write_seq` with
an acquire fence, compare against their `read_seq`, and read slots in
order.

No mutexes are involved. The design relies on:
- Single producer (no write contention)
- Atomic `write_seq` with release/acquire ordering
- Data region sized large enough that consumers finish reading before
  the producer wraps around (typically 2-4 seconds of buffered data)

### 2.2 OSD SHM Double-Buffer

Used by ROD to pass rendered BGRA bitmaps to RVD without frame drops.
One double-buffer per OSD region per stream channel.

#### Struct Layout

```c
/*
 * rss_osd_shm.h -- double-buffered OSD bitmap transport.
 *
 * SHM name: /rss_osd_<stream>_<region>  (e.g. /rss_osd_0_ts)
 */

typedef struct {
    /* Bitmap dimensions (set at creation, immutable) */
    uint32_t        width;
    uint32_t        height;
    uint32_t        stride;          /* bytes per row (width * 4 for BGRA) */
    uint32_t        buf_size;        /* single buffer size = stride * height */

    /* Double-buffer control */
    _Atomic int     active_buf;      /* 0 or 1: which buffer RVD should read */
    _Atomic int     dirty;           /* 1 = active_buf has new data */

    /* Notification */
    int             eventfd;         /* eventfd descriptor (shared via SCM_RIGHTS
                                        or inherited from parent); ROD writes 1
                                        after swapping active_buf */

    /* Pixel data follows immediately */
    /* uint8_t buf[0][buf_size]; */
    /* uint8_t buf[1][buf_size]; */
} rss_osd_shm_t;
```

#### ROD -> RVD Protocol

1. ROD determines which buffer is **inactive**: `inactive = 1 - active_buf`
2. ROD renders the new bitmap into `buf[inactive]` (FreeType2 text, logo blit)
3. ROD atomically stores `active_buf = inactive` (release ordering)
4. ROD atomically stores `dirty = 1`
5. ROD writes `(uint64_t)1` to the eventfd

RVD side (in its frame loop, or a dedicated OSD update thread):

1. RVD polls/reads the eventfd (non-blocking or with epoll)
2. RVD loads `dirty` with acquire ordering; if 0, skip
3. RVD loads `active_buf`; reads `buf[active_buf]`
4. RVD calls `hal->osd_update_region_data(handle, buf[active_buf])`
5. RVD stores `dirty = 0`

The double-buffer ensures ROD never writes to the buffer that RVD is
reading. If ROD produces faster than RVD consumes, the intermediate
frames are simply skipped (latest-wins semantics, which is correct for
timestamp OSD that should always show the most recent time).

### 2.3 Control Sockets

Unix domain stream sockets for command/response communication between
raptorctl and daemons, and between daemons when needed.

#### Socket Paths

```
/var/run/rss/rvd.sock    -- RVD control socket
/var/run/rss/rod.sock    -- ROD control socket
/var/run/rss/rad.sock    -- RAD control socket
/var/run/rss/rsd.sock    -- RSD control socket
/var/run/rss/rmr.sock    -- RMR control socket
/var/run/rss/rsp.sock    -- RSP control socket
/var/run/rss/ric.sock    -- RIC control socket
/var/run/rss/rmc.sock    -- RMC control socket
```

#### Wire Protocol

Simple length-prefixed JSON messages:

```
+--------+--------+-----...-----+
| len_hi | len_lo |  JSON body  |
+--------+--------+-----...-----+
   2 bytes           len bytes
```

`len` is a 16-bit big-endian uint16 giving the JSON body length.
Maximum message size: 65535 bytes (more than sufficient for all
control messages).

Request format:
```json
{"cmd": "set-bitrate", "args": {"channel": 0, "bitrate": 2000000}}
```

Response format:
```json
{"status": "ok", "data": {"bitrate": 2000000}}
```

Error response:
```json
{"status": "error", "code": -22, "msg": "invalid channel"}
```

The protocol is synchronous: one request, one response, per connection.
raptorctl opens a connection, sends a request, reads the response, and
closes. For monitoring (e.g., `raptorctl status`), raptorctl queries
each daemon sequentially.

---

## 3. Pipeline Model

### 3.1 Video Pipeline

```
Sensor
  |
  v
ISP (tuning: brightness, contrast, WB, denoise, flip)
  |
  v
FrameSource[0] ─────────────────────────────────────── FrameSource[1]
  |                                                       |
  v                                                       v
OSD Group[0] (main stream overlays)                  OSD Group[1] (sub stream overlays)
  |                                                       |
  v                                                       v
Encoder[0] (H264/H265, 1080p/2K)                    Encoder[1] (H264, 640x360)
  |                                                       |
  v                                                       v
SHM Ring "main"                                      SHM Ring "sub"
  |                                                       |
  +---> RSD (RTSP /main)                                  +---> RSD (RTSP /sub)
  +---> RMR (MP4 recording)                               +---> RV4 (V4L2 bridge)
  +---> RSP (RTMP push)
```

Bindings (HAL `bind()` calls):
```
FS(0,0) -> OSD(0,0)     main framesource -> main OSD group
OSD(0,0) -> ENC(0,0)    main OSD group -> main encoder
FS(1,0) -> OSD(1,0)     sub framesource -> sub OSD group
OSD(1,0) -> ENC(1,0)    sub OSD group -> sub encoder
```

### 3.2 Audio Pipeline

```
Audio Input (AI device 0, channel 0)
  |
  v
PCM 16-bit @ 8/16kHz
  |
  v
[Optional: NS, HPF, AGC processing in HAL]
  |
  v
Audio Encoder (G.711a / AAC / Opus)
  |
  v
SHM Ring "audio"
  |
  +---> RSD (RTSP audio track)
  +---> RMR (audio mux in MP4)
  +---> RSP (audio in RTMP push)
```

### 3.3 OSD Data Flow

```
ROD                              RVD
 |                                |
 |  render timestamp text         |
 |  into BGRA bitmap              |
 |         |                      |
 |         v                      |
 |  write to inactive buf         |
 |  in OSD SHM double-buffer      |
 |         |                      |
 |  swap active_buf (atomic)      |
 |  set dirty = 1                 |
 |  write eventfd ─────────────>  epoll wakeup
 |                                |
 |                         read active_buf
 |                         call osd_update_region_data()
 |                                |
 |                         clear dirty = 0
```

### 3.4 T40/T41 Multi-Sensor Extension

On T40/T41 with dual MIPI CSI, the pipeline extends to a second sensor:

```
Sensor 0 (CSI0) ──> FS[0] ──> OSD[0] ──> Enc[0] ──> SHM Ring "main0"
                 ──> FS[1] ──> OSD[1] ──> Enc[1] ──> SHM Ring "sub0"

Sensor 1 (CSI1) ──> FS[2] ──> OSD[2] ──> Enc[2] ──> SHM Ring "main1"
                 ──> FS[3] ──> OSD[3] ──> Enc[3] ──> SHM Ring "sub1"
```

RVD manages both sensors through the same HAL context (the T40/T41 HAL
implementation uses `IMPVI_NUM` to route to the correct ISP pipeline).

---

## 4. Startup Sequence

### 4.1 Boot Order

```
1. RVD starts first (mandatory)
   - Creates HAL context
   - Initializes ISP + sensor
   - Creates framesource channels
   - Creates OSD groups (even if ROD isn't running yet -- regions empty)
   - Creates encoder channels
   - Binds pipeline: FS -> OSD -> Enc
   - Enables framesource channels
   - Starts encoder receive
   - Creates SHM rings (main, sub)
   - Creates control socket
   - Enters frame loop: enc_poll -> enc_get_frame -> ring_publish -> enc_release_frame

2. RAD starts (optional, independent of RVD)
   - Opens HAL context (shares the HAL -- audio subsystem is independent)
   - Initializes audio input
   - Creates SHM ring (audio)
   - Enters audio loop: audio_read_frame -> encode -> ring_publish

3. ROD starts (optional, after RVD)
   - Creates OSD SHM double-buffers
   - Connects to RVD control socket to register OSD regions
     (or RVD auto-detects OSD SHM on startup)
   - Enters render loop: render text -> write inactive buf -> swap -> notify

4. Consumer daemons start (any order, after RVD)
   - RSD: opens SHM rings, starts RTSP listener
   - RMR: opens SHM rings, waits for start command or auto-starts
   - RSP: opens SHM rings, connects to push target
   - RV4: opens sub SHM ring, creates V4L2 device

5. Control daemons start (any order)
   - RIC: queries HAL exposure, manages IR-cut GPIO
   - RMC: listens for motor commands on control socket
```

### 4.2 Dependency Graph

```
        RVD (HAL owner)
       / | \     \
     ROD RAD  [consumers]
               /  |  \  \
             RSD RMR RSP RV4

     RIC (independent HAL user -- exposure queries only)
     RMC (independent -- GPIO/UART only)
```

Hard dependencies (cannot start without):
- All consumers require RVD (for SHM rings to exist)
- ROD requires RVD (for OSD region registration and eventfd handoff)

Soft dependencies (start without, attach when available):
- RSD/RMR/RSP can start before RAD -- they serve video-only until the
  audio ring appears, then add the audio track
- RIC can start before or after RVD -- it polls the HAL independently

### 4.3 Init Script

```sh
#!/bin/sh
# S31raptor -- init script for RSS

case "$1" in
    start)
        mkdir -p /var/run/rss
        # RVD must start first and signal readiness
        start-stop-daemon -S -b -x /usr/bin/rvd -- -c /etc/raptor.json
        # Wait for RVD to create SHM rings (up to 5s)
        timeout=50
        while [ $timeout -gt 0 ] && [ ! -e /dev/shm/rss_ring_main ]; do
            usleep 100000
            timeout=$((timeout - 1))
        done
        # Start remaining daemons
        start-stop-daemon -S -b -x /usr/bin/rad -- -c /etc/raptor.json
        start-stop-daemon -S -b -x /usr/bin/rod -- -c /etc/raptor.json
        start-stop-daemon -S -b -x /usr/bin/rsd -- -c /etc/raptor.json
        start-stop-daemon -S -b -x /usr/bin/ric -- -c /etc/raptor.json
        ;;
    stop)
        # Stop consumers first, then producers, then HAL owner
        start-stop-daemon -K -x /usr/bin/rsd
        start-stop-daemon -K -x /usr/bin/ric
        start-stop-daemon -K -x /usr/bin/rod
        start-stop-daemon -K -x /usr/bin/rad
        start-stop-daemon -K -x /usr/bin/rvd   # tears down HAL last
        rm -rf /var/run/rss
        ;;
esac
```

---

## 5. Crash Recovery

### 5.1 Per-Daemon Restart

Each daemon is designed to be individually restartable:

| Daemon | Restart Impact | State Lost | Recovery |
|--------|---------------|------------|----------|
| RVD | **Full pipeline restart.** All consumers lose their rings. | All SHM rings destroyed. | Consumers detect stale ring (magic/version mismatch or unlink) and reconnect. |
| ROD | OSD overlays freeze at last rendered frame. | Rendered bitmap state. | New ROD instance recreates OSD SHM, registers with RVD. OSD resumes. |
| RAD | Audio stream drops. | Audio encoder state. | New RAD instance creates audio ring. Consumers detect new ring and reattach. |
| RSD | RTSP clients disconnect. | Client sessions, RTP sequence numbers. | Clients reconnect. New RSD opens existing rings. |
| RMR | Current recording file may be truncated. | Muxer state, unflushed data. | New RMR starts a new file. Previous file recoverable if moov was written. |
| RSP | Push stream drops, reconnects to server. | TCP connection, RTMP handshake. | New RSP reconnects and resumes push. |
| RIC | Day/night mode freezes at last state. | Hysteresis state, threshold timers. | New RIC reads current exposure, resumes algorithm. |
| RMC | Motors stop. | Motor position tracking. | New RMC requires homing sequence. |

### 5.2 RVD Crash (Full Recovery)

RVD crashing is the worst case because it owns the HAL:

1. RVD crashes -- all SHM rings become stale (no new write_seq updates)
2. Consumers detect stale ring via timeout on `rss_ring_wait()`
3. A watchdog (or init script with `RESPAWN`) restarts RVD
4. RVD re-initializes HAL, creates new SHM rings
5. Consumers detect ring recreation (SHM name unlinked and recreated,
   or magic/version/write_seq reset) and reopen
6. ROD detects RVD restart via control socket disconnect, re-registers
   OSD regions

Total recovery time: ~2-3 seconds (HAL init ~1s, pipeline setup ~0.5s,
first frame ~0.5s).

### 5.3 SHM Cleanup

If a producer crashes without unlinking its SHM:
- SHM objects persist in `/dev/shm/` until explicitly unlinked
- The new producer instance calls `shm_unlink()` before `shm_open(O_CREAT)`
- Consumers holding old mmaps see stale data; they detect this via
  sequence number gaps or magic mismatch after reopen

---

## 6. Memory Budget

### 6.1 T31 (64MB Total RAM)

Estimated per-daemon RSS at steady state:

```
Component                         RAM (KB)    Notes
───────────────────────────────────────────────────────
Kernel + drivers                  ~12,000     includes ISP firmware
libimp.so (loaded by RVD/RAD)      ~2,000     vendor SDK shared library
RVD daemon                           ~800     code + stack + internal buffers
  SHM ring "main" (1080p H264)     ~3,072     16 slots * ~192KB max frame
  SHM ring "sub"  (360p H264)        ~512     16 slots * ~32KB max frame
RAD daemon                           ~256     code + stack + audio buffers
  SHM ring "audio"                    ~128     32 slots * 4KB audio frames
ROD daemon                           ~512     code + FreeType + glyph cache
  OSD SHM double-buffers              ~256     2 regions * 2 bufs * ~32KB
RSD daemon                           ~512     code + RTSP session state
RMR daemon                           ~384     code + muxer + write buffer
RIC daemon                           ~128     minimal; polls exposure
raptorctl                              64     transient; runs and exits
───────────────────────────────────────────────────────
Subtotal userspace                ~20,624
Encoder DMA buffers (rmem)        ~16,384     reserved memory, not in budget
Remaining for OS/cache            ~15,000
───────────────────────────────────────────────────────
Total                             ~48,000     leaves ~16MB headroom
```

### 6.2 Optimization Strategies

- **SHM ring sizing**: Reduce `data_size` if consumers keep up well.
  2MB is conservative for 1080p@25fps H264 at 2Mbps (max frame ~100KB).
  1MB may suffice for VBR with small GOPs.
- **ROD glyph cache**: Pre-render digits 0-9 and common punctuation at
  startup. Reuse bitmaps for timestamp updates (only re-render changed
  digits).
- **Single libimp.so load**: RVD and RAD both link against the vendor
  library, but as separate processes they each get a copy. If memory is
  critical, RAD audio init could be folded into RVD (sacrificing isolation).
- **Consumer-side zero-copy**: RSD reads directly from SHM ring and
  passes pointers to the RTP packetizer. No intermediate buffer copies.
- **Static linking**: All daemons can be statically linked to avoid
  dynamic linker overhead and reduce per-process memory slightly.

### 6.3 T23 (32MB Systems)

On 32MB systems, run a minimal set:

```
RVD + RSD only:           ~18MB total (kernel + ISP + video + RTSP)
RVD + RSD + RAD:          ~19MB total (adds audio)
RVD + RSD + RAD + ROD:    ~20MB total (adds OSD)
```

RMR, RSP, RV4 are omitted on 32MB systems unless absolutely needed.
RIC can run if IR-cut control is required (~128KB additional).

---

## 7. Multi-Core Pinning

### 7.1 T40/T41 Dual-Core Affinity

The T40 and T41 have dual XBurst MIPS cores. CPU affinity assignment:

```
Core 0 (CPU0):
  - RVD frame loop (enc_poll + enc_get_frame + ring_publish)
  - ISP/encoder interrupt handling (kernel)
  - Hardirq + softirq processing

Core 1 (CPU1):
  - RSD (RTSP serving, RTP packetization, network I/O)
  - RMR (recording, file I/O)
  - ROD (text rendering)
  - RAD (audio capture + encoding)
  - RIC, RMC (control daemons)
```

Rationale: RVD is the latency-critical path. Pinning it to Core 0
(which also handles ISP interrupts) minimizes cache thrashing and
ensures frames are consumed from the encoder as quickly as possible.
All other daemons share Core 1, which is acceptable because their CPU
usage is modest and their latency requirements are relaxed.

### 7.2 Implementation

Each daemon sets CPU affinity at startup using `sched_setaffinity()`:

```c
#include <sched.h>

static void pin_to_cpu(int cpu) {
    cpu_set_t set;
    CPU_ZERO(&set);
    CPU_SET(cpu, &set);
    sched_setaffinity(0, sizeof(set), &set);
}
```

The target CPU is read from the config file:

```json
{
    "rvd": { "cpu_affinity": 0, "sched_priority": 50 },
    "rsd": { "cpu_affinity": 1, "sched_priority": 30 },
    "rod": { "cpu_affinity": 1, "sched_priority": 20 },
    "rad": { "cpu_affinity": 1, "sched_priority": 40 },
    "ric": { "cpu_affinity": 1, "sched_priority": 10 }
}
```

On single-core SoCs (T20, T21, T23, T30, T31), `cpu_affinity` is
ignored (all daemons run on CPU0). The config parser silently skips
invalid CPU numbers.

### 7.3 Scheduling Policy

- RVD: `SCHED_FIFO` priority 50 -- highest, must never be preempted by
  other daemons during frame processing
- RAD: `SCHED_FIFO` priority 40 -- audio frame deadlines are real-time
- RSD: `SCHED_FIFO` priority 30 -- RTSP packet deadlines matter for jitter
- ROD: `SCHED_OTHER` (normal) with nice -5 -- OSD rendering is periodic
  but not deadline-critical
- RIC/RMC: `SCHED_OTHER` (normal) -- control loops are 1Hz or event-driven
- RMR/RSP: `SCHED_OTHER` (normal) -- file/network I/O is buffered

---

## 8. Configuration

All daemons read from a single JSON configuration file (`/etc/raptor.json`).
Each daemon reads only its own section plus the shared `hal` section:

```json
{
    "hal": {
        "sensor": {
            "name": "gc2053",
            "i2c_addr": 55,
            "i2c_adapter": 0,
            "rst_gpio": 18,
            "pwdn_gpio": -1
        }
    },
    "rvd": {
        "streams": [
            {
                "id": 0,
                "width": 1920, "height": 1080,
                "codec": "h264", "profile": 2,
                "bitrate": 2000000, "rc_mode": "vbr",
                "fps": 25, "gop": 50
            },
            {
                "id": 1,
                "width": 640, "height": 360,
                "codec": "h264", "profile": 0,
                "bitrate": 500000, "rc_mode": "cbr",
                "fps": 15, "gop": 30
            }
        ],
        "ring": {
            "main_slots": 16, "main_data_mb": 3,
            "sub_slots": 16, "sub_data_mb": 1
        },
        "cpu_affinity": 0,
        "sched_priority": 50
    },
    "rad": {
        "sample_rate": 16000,
        "codec": "alaw",
        "volume": 80,
        "gain": 25,
        "ns_level": "moderate",
        "ring_slots": 32,
        "ring_data_kb": 128
    },
    "rod": {
        "regions": [
            {
                "name": "timestamp",
                "stream": 0,
                "x": 10, "y": 10,
                "font_size": 32,
                "format": "%Y-%m-%d %H:%M:%S"
            },
            {
                "name": "camname",
                "stream": 0,
                "x": 10, "y": 1040,
                "font_size": 24,
                "text": "Front Door"
            }
        ],
        "font": "/usr/share/fonts/default.ttf",
        "update_interval_ms": 500
    },
    "rsd": {
        "port": 554,
        "max_clients": 4,
        "mount_main": "/main",
        "mount_sub": "/sub"
    },
    "rmr": {
        "path": "/mnt/sd/recordings",
        "format": "mp4",
        "segment_minutes": 5,
        "auto_start": false
    },
    "ric": {
        "mode": "auto",
        "day_threshold": 800,
        "night_threshold": 200,
        "hysteresis_sec": 10,
        "ircut_gpio_a": 25,
        "ircut_gpio_b": 26,
        "ir_led_gpio": 49
    }
}
```

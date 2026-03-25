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
- **Sensor auto-detect**: Reads sensor name, i2c_addr, resolution, and
  max FPS from `/proc/jz/sensor/` when not set in config. Default config
  ships with `[sensor]` section commented out — works on any device.
- **Startup info**: Logs LIBIMP version, SYSUTILS version, and CPU info
  (e.g., "T31-AL") via HAL `rss_hal_get_imp_version()` etc.
- **Outputs**: Two video SHM rings (main, sub), two JPEG SHM rings
  (jpeg0, jpeg1 — one per video stream for snapshots). Snapshots are
  served directly from JPEG rings by RHD; no disk files are written.
- **JPEG channels**: Use encoder channels 4+ (SDK convention), registered
  into video encoder groups via buffer sharing. No separate framesource.
- **Thread model**: RVD runs 6 threads, all with 128KB stacks:
  - **Main thread**: control socket via epoll, handles raptorctl commands.
  - **4 encoder threads**: one per channel (main H264, sub H264, jpeg0,
    jpeg1). Each thread independently runs enc_poll -> enc_get_frame ->
    ring_publish_iov -> enc_release_frame. No locks between encoder
    threads — each owns its own ring with no shared mutable state.
  - **OSD update thread**: polls OSD SHM dirty flags and calls
    UpdateRgnAttrData. Runs in a dedicated thread because SDK OSD calls
    can interfere with the encoder if invoked on the same thread.
- **OSD init sequence**: Follows the vendor SDK spec exactly:
  CreateGroup -> CreateRgn(NULL) -> RegisterRgn -> SetRgnAttr(pData=NULL)
  -> SetGrpRgnAttr(show=0) -> OSD_Start -> System_Bind(FS->OSD->ENC) ->
  FS_Enable. Each region gets a unique layer (1-5). Regions are created
  eagerly with transparent placeholders before the encoder starts.
  Runtime updates use UpdateRgnAttrData with a heap-allocated local_buf
  (SDK DMA requires non-SHM memory). Privacy mode uses OSD_REG_COVER at
  layer 0, toggled via raptorctl.
- **Why separate**: The video pipeline is the critical path. Isolating
  it means audio glitches, OSD rendering delays, or RTSP client
  disconnects cannot stall frame production.

#### ROD -- OSD Overlay Daemon

- **Role**: Render OSD overlays into BGRA bitmaps and publish them to
  OSD SHM double-buffers. Manages 5 region types per stream:
  - **Time** (top-left): timestamp rendered at 1Hz.
  - **Uptime** (top-right): system uptime, right-aligned.
  - **Text/Description** (top-center): configurable label, center-aligned.
  - **Logo** (bottom-right): loaded from a BGRA file (100x30 — T31 SDK
    limitation with larger bitmaps).
  - **Privacy** (center): hidden until activated; covers the frame when
    privacy mode is toggled on via raptorctl.
- **HAL dependency**: None. ROD does not touch the HAL. It renders
  bitmaps in userspace using libschrift for font rendering with a glyph
  cache (O(1) ASCII lookup). The font is loaded once and shared across
  streams using different SFT scale factors.
- **Per-stream scaling**: Font size is auto-scaled for the sub stream
  (proportional to resolution) with a configurable override per stream.
- **Text alignment**: Left for time, right for uptime, center for
  description and privacy text.
- **Outputs**: OSD SHM double-buffer (one per OSD region per stream
  channel). Sets dirty flag after each update; RVD's OSD thread polls
  dirty flags.
- **Control socket**: Accepts `set-text` command to change the OSD text
  string at runtime.
- **Render loop**: 1Hz cycle — renders all dirty regions, updates SHM
  double-buffers.
- **Why separate**: Font rendering is CPU-intensive relative to the
  frame path. Isolating it lets ROD run at a lower priority without
  risking frame drops. ROD can also be killed and restarted to change
  OSD layout without interrupting the video stream (existing overlay
  just freezes until ROD restarts).

#### RAD -- Audio Daemon

- **Role**: Initialize audio input and output through the HAL, read PCM
  frames, encode (PCMU, PCMA, L16, AAC, or Opus), publish to audio SHM
  ring, and play back received audio through the speaker.
- **HAL dependency**: Calls `audio_init()`, `audio_read_frame()`,
  `audio_release_frame()` for input. Calls `ao_init()`, `ao_send_frame()`
  for output. Optionally enables NS/HPF/AGC audio effects via HAL
  (`audio_enable_ns()`, `audio_enable_hpf()`, `audio_enable_agc()`).
  All SDK access goes through the HAL vtable.
- **Codecs**: PCMU/PCMA (G.711), L16 (uncompressed PCM), AAC-LC (via
  faac, compile with `AAC=1`), Opus (via libopus, compile with `OPUS=1`).
- **Audio effects**: Noise suppression, high-pass filter, and AGC via
  `libaudioProcess.so`. Compile-time flag `AUDIO_EFFECTS=1`. Runtime
  toggle via `raptorctl rad set-ns/set-hpf/set-agc`.
- **Audio output**: When `ao_enabled=true`, RAD initializes the AO device
  and spawns an AO playback thread. The thread reads PCM16 from a
  "speaker" SHM ring (created by `rac play` or RSD backchannel) and
  calls `ao_send_frame()`. Auto-reconnects when the speaker ring is
  recreated by a new client.
- **Outputs**: Audio SHM ring (input/encoding path).
- **Inputs**: Speaker SHM ring (output/playback path).
- **Thread model**: Main thread (AI read loop + control socket), AO
  playback thread (speaker ring consumer).
- **Why separate**: Audio runs on a fixed-period schedule (typically
  20ms frame intervals). Isolating it from video prevents encoder
  stalls from causing audio discontinuities and vice versa. RAD can
  also be disabled entirely on cameras without microphones.

### 1.2 Consumer Daemons

These daemons read from SHM ring buffers and deliver frames to external
sinks. Multiple consumers can attach to the same ring simultaneously.

#### RSD -- RTSP Server

- **Role**: RTSP/RTP server. Reads video and audio SHM rings, packetizes
  into RTP, serves to clients via RTSP. Receives backchannel audio from
  clients for two-way audio.
- **Inputs**: Main video ring, sub video ring, audio ring.
- **Outputs**: Speaker SHM ring (backchannel audio → RAD AO thread).
- **Dependencies**: librss_ipc (SHM ring consumer API), compy (RTSP
  protocol library). Epoll-based event loop. No HAL dependency.
- **Protocol**: Standard RTSP 1.0 (DESCRIBE/SETUP/PLAY/TEARDOWN). Two
  mount points: `/stream0` (main) and `/stream1` (sub). Audio track
  included when RAD is running. TCP interleaved and UDP transport.
- **Backchannel**: SDP advertises a `sendonly` PCMU/8000 backchannel
  track (`a=control:backchannel`). Clients SETUP the backchannel track,
  then send RTP audio via TCP interleaved framing. RSD decodes PCMU to
  PCM16, upsamples 8→16kHz, and writes to the speaker SHM ring. RAD's
  AO thread plays it through the hardware speaker.
- **Client listing**: `raptorctl rsd clients` shows connected clients
  with IP, port, stream index, transport, and video/audio/backchannel state.
- **Authentication**: Digest auth (RFC 2617) via compy, credentials from
  `[rtsp]` config section. Empty = no auth.
- **Network**: Dual-stack IPv6 (AF_INET6 with IPV6_V6ONLY=0), port 554.
  Per-client RTP timestamp offsets ensure each client sees timestamps
  starting from zero on connect (set on first keyframe delivery).
- **Why separate**: RTSP client management (socket I/O, RTP packetization,
  RTCP) is complex and has its own failure modes (slow clients, network
  errors). Isolating it means a misbehaving RTSP client cannot affect
  frame production or recording.

#### RHD -- HTTP Daemon

- **Role**: HTTP server for JPEG snapshots and MJPEG streaming.
- **Inputs**: JPEG SHM rings (jpeg0, jpeg1). Zero disk I/O.
- **Dependencies**: librss_ipc (ring consumer), librss_common.
- **Endpoints**:
  - `/snap.jpg` or `/snap.jpg?stream=0` — main stream snapshot (1920x1080)
  - `/snap.jpg?stream=1` — sub stream snapshot (640x360)
  - `/mjpeg` — MJPEG stream (multipart/x-mixed-replace from jpeg0 ring)
  - `/` — simple HTML index page with links and embedded MJPEG preview
- **Network**: Dual-stack IPv6 (AF_INET6 with IPV6_V6ONLY=0), port 8080.
- **Why separate**: HTTP and RTSP are different protocols with different
  connection lifecycles. RHD can serve snapshots even if RSD is down.

#### RMR -- Recorder

- **Role**: Fragmented MP4 recording daemon. Reads H.264/H.265 and audio
  frames from SHM rings and writes fMP4 segments to SD card (or any
  writable path). Each `moof+mdat` box pair is self-contained, so a
  crash or power loss cannot corrupt already-written segments.
- **Inputs**: Main video ring (or sub), audio ring.
- **Muxer**: Own lightweight fMP4 muxer — zero external dependencies. No
  libavformat required. The muxer is stateless per-segment; on rotation
  it closes the current file and opens a new one without touching the
  previous segment.
- **Thread model**: Two threads — a **reader thread** (ring consumer →
  muxer → write buffer) and a **writer thread** (drains an SPSC circular
  buffer → `write(fd)`). The SPSC buffer decouples the real-time ring
  reader from SD card write stalls.
- **Crash safety**: Each `moof+mdat` pair is flushed before the reader
  advances. A crash between segments leaves all prior segments intact
  and playable.
- **Timestamps**: Video DTS is derived from ring slot timestamps. Audio
  DTS is counter-based (sample count / sample rate), avoiding clock skew
  between ring producers.
- **Storage management**: Configurable segment rotation (`segment_minutes`
  in config) and storage cleanup (oldest segments deleted when free space
  falls below threshold).
- **Dependencies**: librss_ipc, librss_common. No HAL dependency.
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

- **Role**: Monitor ISP exposure levels, drive IR-cut filter and IR LED
  GPIOs based on configurable thresholds, switch ISP running mode
  between day and night.
- **HAL dependency**: None. RIC queries ISP exposure via RVD's control
  socket (`get-exposure` command) and sets ISP running mode via
  `set-running-mode` command. GPIO control is direct sysfs. This avoids
  the libimp per-process ISP device ownership issue.
- **GPIO support**: Single GPIO (pin toggles high/low) or dual GPIO
  (motor driver with 100ms pulse). IR LED enable/disable.
- **Algorithm**: Total gain threshold with configurable hysteresis
  debounce (consecutive seconds above/below threshold before switching).
- **IPC**: Polls RVD control socket every 1s for exposure data. Receives
  override commands via own control socket (force day, force night, auto).
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

### 1.4 CLI Tools

#### raptorctl

- **Role**: Command-line interface for daemon management and diagnostics.
- **Communication**: Connects to each daemon's Unix control socket.
- **Commands**:
  - `raptorctl status` -- show running daemons (PID or stopped)
  - `raptorctl get <section> <key>` -- query live value from daemon
  - `raptorctl set <section> <key> <value>` -- write value to config
  - `raptorctl config save` -- persist running config to disk
  - `raptorctl <daemon> status` -- show daemon-specific details
  - `raptorctl <daemon> config` -- show daemon's running config
  - `raptorctl <daemon> clients` -- list connected clients (rsd)
  - `raptorctl rvd request-idr [channel]` -- request IDR frame
  - `raptorctl rvd set-bitrate <ch> <bps>` -- change encoder bitrate
  - `raptorctl rvd set-gop <ch> <length>` -- change GOP length
  - `raptorctl rvd set-fps <ch> <fps>` -- change frame rate
  - `raptorctl rvd set-qp-bounds <ch> <min> <max>` -- change QP range
  - `raptorctl rvd privacy [on|off]` -- toggle privacy mode
  - `raptorctl rad set-volume <val>` -- change audio input volume
  - `raptorctl rad set-gain <val>` -- change audio input gain
  - `raptorctl rad set-ns <0|1> [level]` -- noise suppression
  - `raptorctl rad set-hpf <0|1>` -- high-pass filter
  - `raptorctl rad set-agc <0|1> [target] [compression]` -- AGC
  - `raptorctl rod set-text <text>` -- change OSD text string
  - `raptorctl ric mode <auto|day|night>` -- set day/night mode

#### rac -- Raptor Audio Client

- **Role**: CLI tool for audio playback and recording.
- **Commands**:
  - `rac play <file|->` -- play audio to speaker (PCM16, MP3, AAC, Opus)
  - `rac record <file|-> [-d sec]` -- record mic to PCM16 LE file
  - `rac status` -- show audio daemon status
  - `rac ao-volume <val>` -- set speaker volume
  - `rac ao-gain <val>` -- set speaker gain
- **Playback codecs**: Raw PCM16 LE (passthrough), MP3 (libhelix-mp3,
  compile with `MP3=1`), AAC/ADTS (libhelix-aac, `AAC=1`), Opus/Ogg
  (libopus, `OPUS=1`). Auto-detects format from file extension or magic
  bytes. Stereo downmixed to mono. Sample rate resampling via linear
  interpolation (MP3/AAC) or native Opus decoder rate selection.
- **Speaker path**: Creates a "speaker" SHM ring, writes PCM16 frames.
  RAD's AO playback thread reads and plays via `ao_send_frame()`.
- **Recording**: Reads from the audio SHM ring, decodes G.711/L16 to
  PCM16 LE. Duration limit with `-d` flag.

#### ringdump

- **Role**: SHM ring buffer inspector for debugging.
- **Usage**: `ringdump <ring> [-f] [-d] [-n N]`

---

## 2. IPC Primitives

### 2.1 SHM Ring Buffer (`rss_ipc.h`)

The primary data transport between producers and consumers. Lock-free,
single-producer multi-consumer, designed for zero-copy frame passing.

#### Struct Layout

```c
/*
 * rss_ipc.h -- SHM ring buffer for encoded frames.
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

    /* Stream metadata (set once at init via rss_ring_set_stream_info) */
    uint32_t          stream_id;        /* 0=main, 1=sub, 2=jpeg, 0x10=audio */
    uint32_t          codec;            /* rss_codec_t value */
    uint32_t          width;            /* video width (0 for audio) */
    uint32_t          height;           /* video height (0 for audio) */
    uint32_t          fps_num;
    uint32_t          fps_den;
    uint8_t           profile;          /* H.264 profile_idc (66=Base,77=Main,100=High) */
    uint8_t           level;            /* H.264 level_idc (30,31,40,51...) */
    uint16_t          _reserved;

    /* Incarnation counter -- incremented on each ring creation.
     * Consumers check this on every read to detect producer restart. */
    _Atomic uint32_t  incarnation;

    /* Consumer → producer IDR request. Consumer sets to 1 after
     * EOVERFLOW to get a keyframe fast. Producer checks and clears
     * in its encode loop. Avoids control socket round-trip. */
    _Atomic uint32_t  idr_request;

    /* Magic + version for consumer validation.
     * Magic is written last with memory_order_release as a ready flag.
     * Consumer open uses memory_order_acquire. Closes TOCTOU window. */
    uint32_t          magic;            /* RSS_RING_MAGIC */
    uint32_t          version;          /* RSS_RING_VERSION */
    char              pad[0] __attribute__((aligned(64))); /* cache-line align */
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
 * rss_ring_publish(): publish a single contiguous buffer to the ring.
 *
 * rss_ring_publish_iov(): scatter-gather publish -- writes multiple
 *   buffers contiguously into the ring in one operation. Used by RVD
 *   to publish SDK NAL units directly without an intermediate copy.
 *
 * rss_ring_set_stream_info(): set stream metadata in the ring header
 *   (codec, resolution, fps, H.264 profile/level). Called once after
 *   ring creation. Consumers read this via rss_ring_get_header().
 */

/* Scatter-gather I/O vector */
typedef struct {
    const uint8_t *data;
    uint32_t       length;
} rss_iov_t;

rss_ring_t *rss_ring_create(const char *name, uint32_t slot_count,
                             uint32_t data_size);
void         rss_ring_destroy(rss_ring_t *ring);

int          rss_ring_publish(rss_ring_t *ring,
                              const uint8_t *data, uint32_t length,
                              int64_t timestamp, uint16_t nal_type,
                              uint8_t is_key);

int          rss_ring_publish_iov(rss_ring_t *ring,
                                   const rss_iov_t *iov, uint32_t iov_count,
                                   int64_t timestamp, uint16_t nal_type,
                                   uint8_t is_key);

void         rss_ring_set_stream_info(rss_ring_t *ring,
                                       uint32_t stream_id, uint32_t codec,
                                       uint32_t width, uint32_t height,
                                       uint32_t fps_num, uint32_t fps_den,
                                       uint8_t profile, uint8_t level);
```

#### Consumer API

```c
/*
 * Consumer side -- used by RSD, RMR, RSP, RV4.
 *
 * rss_ring_open(): attach to an existing ring by name.
 *   Returns NULL if the ring does not exist yet. Consumers should
 *   retry with backoff until the producer creates the ring.
 *   Uses memory_order_acquire on magic to synchronize with the
 *   producer's release store -- closes the TOCTOU window.
 *
 * rss_ring_read(): copy-on-read API. Copies frame data into the
 *   caller's dest_buf, then re-validates the slot sequence number
 *   to ensure the data was not overwritten during the copy. This
 *   eliminates the data race inherent in returning SHM pointers.
 *   Also checks the incarnation counter on every read to detect
 *   producer restart.
 *
 *   Each consumer tracks its own read_seq. If read_seq < write_seq,
 *   the consumer has fallen behind and reads the next available slot.
 *   If the consumer has fallen behind far enough that its data was
 *   overwritten, rss_ring_read() returns RSS_EOVERFLOW and advances
 *   read_seq to the oldest valid slot.
 *
 * rss_ring_wait(): block until a new frame is available.
 *   Uses futex on write_seq in shared memory. The producer calls
 *   FUTEX_WAKE after each publish, waking all blocked consumers
 *   instantly (~microseconds, vs 1-10ms with the old polling approach).
 *
 * rss_ring_get_header(): read-only access to ring metadata (codec,
 *   resolution, fps, profile/level). Used by RSD for SDP generation.
 */
rss_ring_t *rss_ring_open(const char *name);
void         rss_ring_close(rss_ring_t *ring);

int          rss_ring_read(rss_ring_t *ring, uint64_t *read_seq,
                           uint8_t *dest_buf, uint32_t dest_size,
                           uint32_t *length, rss_ring_slot_t *meta);

int          rss_ring_wait(rss_ring_t *ring, uint32_t timeout_ms);

const rss_ring_header_t *rss_ring_get_header(rss_ring_t *ring);

/* Consumer → producer fast-path IDR request (avoids control socket) */
void         rss_ring_request_idr(rss_ring_t *ring);
int          rss_ring_check_idr(rss_ring_t *ring);  /* returns 1 and clears */
```

#### Frame Passing

The data region is a circular byte buffer. The producer writes frame
payloads sequentially, wrapping around when the end is reached. The
slot's `data_offset` and `data_length` describe where the payload lives.

**Data region wrap**: When a frame does not fit in the remaining space
at the tail of the data region, the tail is skipped and the frame is
written at offset 0. Bounded waste: at most one frame size per wrap
cycle.

`rss_ring_read()` uses copy-on-read semantics: it copies frame data
into the caller's destination buffer, then re-validates the slot
sequence number to ensure the data was not overwritten during the copy.
This eliminates the data race inherent in returning raw SHM pointers.

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
 * rss_ipc.h -- double-buffered OSD bitmap transport.
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

    /* Pixel data follows immediately */
    /* uint8_t buf[0][buf_size]; */
    /* uint8_t buf[1][buf_size]; */
} rss_osd_shm_t;
```

#### ROD -> RVD Protocol

1. ROD determines which buffer is **inactive**: `inactive = 1 - active_buf`
2. ROD renders the new bitmap into `buf[inactive]` (libschrift text, logo blit)
3. ROD atomically stores `active_buf = inactive` (release ordering)
4. ROD atomically stores `dirty = 1`

RVD side (dedicated OSD update thread):

1. RVD OSD thread polls `dirty` flags for all regions
2. RVD loads `dirty` with acquire ordering; if 0, skip
3. RVD loads `active_buf`; reads `buf[active_buf]`
4. RVD copies data to a heap-allocated local_buf (SDK DMA requires
   non-SHM memory), then calls `UpdateRgnAttrData(handle, local_buf)`
5. RVD stores `dirty = 0`

Consumer mmap uses PROT_READ|PROT_WRITE (needed for atomic_store in
the dirty flag clear operation).

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
  +---> RSD (RTSP /stream0)                                +---> RSD (RTSP /stream1)
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
Audio Encoder (PCMU / PCMA / L16)
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
ROD                              RVD (OSD update thread)
 |                                |
 |  render timestamp text         |
 |  into BGRA bitmap              |
 |         |                      |
 |         v                      |
 |  write to inactive buf         |
 |  in OSD SHM double-buffer      |
 |         |                      |  poll dirty flags
 |  swap active_buf (atomic)      |         |
 |  set dirty = 1  ─────────────> |  dirty == 1?
 |                                |         |
 |                         read active_buf  |
 |                         copy to local_buf (heap)
 |                         call UpdateRgnAttrData()
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
   - Creates JPEG encoder channels (chn 4, 5) with bufshare, registers into video groups
   - Creates SHM rings (main, sub, jpeg0, jpeg1)
   - Creates control socket
   - Starts 4 encoder threads (one per channel): enc_poll -> enc_get_frame -> ring_publish_iov -> enc_release_frame
   - Starts OSD update thread (polls dirty flags, calls UpdateRgnAttrData)

2. RAD starts (optional, independent of RVD)
   - Opens HAL context (shares the HAL -- audio subsystem is independent)
   - Initializes audio input
   - Creates SHM ring (audio)
   - Enters audio loop: audio_read_frame -> encode -> ring_publish

3. ROD starts (optional, after RAD, before RSD)
   - Loads font (libschrift), builds glyph cache
   - Creates OSD SHM double-buffers (one per region per stream)
   - Loads logo from BGRA file (100x30)
   - Creates control socket
   - Enters 1Hz render loop: render text -> write inactive buf -> swap -> set dirty

4. Consumer daemons start (any order, after RVD)
   - RSD: opens video + audio SHM rings, starts RTSP listener on port 554
   - RHD: opens jpeg SHM rings, starts HTTP listener on port 8080 (dual-stack)
   - RMR: opens SHM rings, waits for start command or auto-starts
   - RSP: opens SHM rings, connects to push target

5. Control daemons start (any order)
   - RIC: queries HAL exposure, manages IR-cut GPIO
```

### 4.2 Dependency Graph

```
        RVD (HAL owner)
       / | \      \
     ROD RAD  [consumers]
              / |  \  \
           RSD RHD RMR RSP

     RIC (queries RVD via control socket -- no direct HAL)
     RMC (independent -- GPIO/UART only)
```

Hard dependencies (cannot start without):
- All consumers require RVD (for SHM rings to exist)
- ROD requires RVD (for OSD SHM regions to be consumed)

Soft dependencies (start without, attach when available):
- RSD/RMR/RSP can start before RAD -- they serve video-only until the
  audio ring appears, then add the audio track
- RIC requires RVD (queries exposure via control socket) but retries until available

### 4.3 Init Script

```sh
#!/bin/sh
# S31raptor -- init script for RSS

case "$1" in
    start)
        mkdir -p /var/run/rss
        # RVD must start first and signal readiness
        start-stop-daemon -S -b -x /usr/bin/rvd -- -c /etc/raptor.conf
        # Wait for RVD to create SHM rings (up to 5s)
        timeout=50
        while [ $timeout -gt 0 ] && [ ! -e /dev/shm/rss_ring_main ]; do
            usleep 100000
            timeout=$((timeout - 1))
        done
        # Start remaining daemons (ROD between RAD and RSD)
        start-stop-daemon -S -b -x /usr/bin/rad -- -c /etc/raptor.conf
        start-stop-daemon -S -b -x /usr/bin/rod -- -c /etc/raptor.conf
        start-stop-daemon -S -b -x /usr/bin/rsd -- -c /etc/raptor.conf
        start-stop-daemon -S -b -x /usr/bin/rhd -- -c /etc/raptor.conf
        start-stop-daemon -S -b -x /usr/bin/ric -- -c /etc/raptor.conf
        ;;
    stop)
        # Stop consumers first, then producers, then HAL owner
        start-stop-daemon -K -x /usr/bin/rsd
        start-stop-daemon -K -x /usr/bin/rhd
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
| ROD | OSD overlays freeze at last rendered frame. | Rendered bitmap state, glyph cache. | New ROD instance recreates OSD SHM double-buffers. RVD OSD thread detects new dirty flags. OSD resumes. |
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
6. ROD detects RVD restart (OSD SHM recreated), reconnects and
   re-renders OSD regions

Total recovery time: ~2-3 seconds (HAL init ~1s, pipeline setup ~0.5s,
first frame ~0.5s).

### 5.3 SHM Cleanup

If a producer crashes without unlinking its SHM:
- SHM objects persist in `/dev/shm/` until explicitly unlinked
- The new producer instance calls `shm_unlink()` before `shm_open(O_CREAT)`
- Consumers holding old mmaps see stale data; they detect this via
  incarnation counter mismatch, sequence number gaps, or magic mismatch
  after reopen

---

## 6. Memory Budget

### 6.1 T31X Memory Layout

The T31X has 128MB physical RAM, split between Linux and reserved memory:

```
Region                            Size        Notes
───────────────────────────────────────────────────────
Linux-visible (MemTotal)          ~60 MB      kernel + userspace + page cache
Encoder DMA (rmem=64M@0x4000000) ~64 MB      ISP/encoder reserved memory
Kernel reserved                    ~4 MB      kernel text, page tables, etc.
───────────────────────────────────────────────────────
Physical total                    128 MB
```

The rmem pool is NOT available to Linux. Encoder DMA buffers, ISP
firmware, and video pipeline memory come from this pool. It is
configured via bootloader `rmem=` kernel parameter.

### 6.2 Userspace Memory (measured on T31X gc4653 2560x1440)

```
Component                         RAM (KB)    Notes
───────────────────────────────────────────────────────
Kernel + drivers                  ~12,000     within Linux-visible 60MB
libimp.so (loaded by RVD/RAD)      ~2,000     vendor SDK shared library
RVD daemon                         ~7,072     6 threads (128KB stacks)
  SHM ring "main" (1080p H264)     ~4,096     32 slots, 4MB data
  SHM ring "sub"  (360p H264)      ~1,024     32 slots, 1MB data
  SHM ring "jpeg0"                    512     4 slots, 512KB data
  SHM ring "jpeg1"                    512     4 slots, 512KB data
ROD daemon                           ~512     libschrift + glyph cache + SHM
RAD daemon                         ~1,040     code + audio buffers
  SHM ring "audio"                    ~256     32 slots
RSD daemon                         ~6,076     compy + per-client state
RHD daemon                         ~2,360     HTTP + JPEG ring readers
raptorctl                              64     transient; runs and exits
───────────────────────────────────────────────────────
Subtotal userspace                ~11,000     measured with all 5 daemons running
Free + page cache                 ~49,000     available for OS/applications
───────────────────────────────────────────────────────
CPU usage                           <2%       hardware encoder does the work
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

The target CPU is read from the config file (`/etc/raptor.conf`):

```ini
[rvd]
cpu_affinity = 0
sched_priority = 50

[rsd]
cpu_affinity = 1
sched_priority = 30

[rad]
cpu_affinity = 1
sched_priority = 40

[rod]
cpu_affinity = 1
sched_priority = 20

[ric]
cpu_affinity = 1
sched_priority = 10
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

All daemons read from a single INI configuration file (`/etc/raptor.conf`).
Each daemon reads only the sections it needs.

### 8.1 Consistency Model (Cisco-style running/startup)

- **On-disk config** (`/etc/raptor.conf`) is the source of truth at boot.
- **Running config** is the in-memory copy each daemon holds after startup.
- `raptorctl rvd set-bitrate 0 3000000` updates the running config only.
  The on-disk file is unchanged. The change takes effect immediately.
- `raptorctl config save` tells all daemons to persist their running
  config to disk (atomic write via temp + fsync + rename).
- A crash between a runtime change and `config save` loses the change.
  This is intentional — it allows experimentation without risk.
- `raptorctl set <section> <key> <value>` writes directly to the file
  without affecting running daemons. Daemons pick it up on next restart.
- `raptorctl get <section> <key>` queries the running daemon first,
  falls back to the on-disk file if the daemon isn't running.

### 8.2 File Format

```ini
[sensor]
name = gc4653
i2c_addr = 0x29
i2c_adapter = 0
fps = 0                # 0 = auto-detect from /proc/jz/sensor/max_fps

[stream0]
width = 1920
height = 1080
fps = 25
codec = h264           # h264, h265, jpeg
profile = 2            # 0=baseline, 1=main, 2=high
bitrate = 2000000
rc_mode = vbr          # cbr, vbr, fixqp, capped_vbr
gop = 50
min_qp = 15
max_qp = 45

[stream1]
enabled = true
width = 640
height = 360
fps = 25
codec = h264
profile = 0
bitrate = 500000
rc_mode = cbr
gop = 50

[jpeg]
enabled = true
jpeg0_enabled = true   # per-channel enable; each JPEG channel costs ~5fps on T20
jpeg1_enabled = true   # disable sub-stream JPEG to recover fps on T20
quality = 75           # 1-100
fps = 1                # snapshot refresh rate
bufshare = true        # share buffer pool with video encoder

[ring]
main_slots = 32
main_data_mb = 4
sub_slots = 32
sub_data_mb = 1

[audio]
enabled = true
sample_rate = 16000
codec = l16            # pcmu, pcma, l16
volume = 80
gain = 25

[rtsp]
port = 554
max_clients = 4

[http]
port = 8080

[osd]
enabled = true
font = /usr/share/fonts/default.ttf
font_size = 24             # main stream; sub stream auto-scaled
stream1_font_size = 12     # optional per-stream override
timestamp = true
timestamp_fmt = %Y-%m-%d %H:%M:%S
label = Camera
logo = /usr/share/raptor/logo.bgra   # 100x30 BGRA file
privacy_color = 0xFF000000            # BGRA black

# Per-stream region positions (named_position or x,y coords)
stream0_time_pos = top_left
stream0_uptime_pos = top_right
stream0_desc_pos = top_center
stream0_logo_pos = bottom_right
stream0_privacy_pos = center
stream1_time_pos = top_left
stream1_uptime_pos = top_right
stream1_desc_pos = top_center
stream1_logo_pos = bottom_right
stream1_privacy_pos = center

[ircut]
enabled = true
mode = auto
day_threshold = 25000
night_threshold = 40000
hysteresis_sec = 5
gpio_ircut = 25
gpio_irled = 26

[log]
level = info           # fatal, error, warn, info, debug, trace
target = syslog        # stderr, syslog, file
```

---

## 9. Authentication

### 9.1 RTSP Digest Authentication (RSD)

RSD uses compy's `Compy_Auth` for RTSP Digest authentication per
RFC 2617. The `before()` hook in the compy Controller calls
`compy_auth_check()` on each incoming RTSP request before dispatching
to the method handler.

Credentials are read from the `[rtsp]` section of the config file:

```ini
[rtsp]
username = admin
password = secret
```

If `username` or `password` is empty (or the keys are absent), auth
is skipped entirely. This keeps the default deployment backwards
compatible — cameras with no credentials set serve unauthenticated
streams as before.

When credentials are configured:
- RSD sends a `401 Unauthorized` with a `WWW-Authenticate: Digest`
  challenge on the first unauthenticated request.
- The client retries with a `Authorization: Digest` header; compy
  verifies the HA1/HA2 hashes and either allows or rejects.
- Auth state is per-connection, not per-session.

### 9.2 HTTP Basic Authentication (RHD)

RHD uses HTTP Basic auth with base64 decode. Credentials are read
from the `[http]` section:

```ini
[http]
username = admin
password = secret
```

On each request, RHD checks for an `Authorization: Basic <b64>` header,
base64-decodes it, and compares the `user:pass` string against config.
If credentials are empty, auth is skipped (same backwards-compatible
logic as RSD).

---

## 10. Dynamic JPEG Buffer Sizing

RHD and ringdump size their read buffers dynamically from
`ring->data_size` — the full data region size reported in the SHM ring
header — rather than using a hardcoded constant.

This handles night mode, where high-gain JPEG frames can reach 270KB or
more. A previously hardcoded 256KB buffer silently truncated these
frames. The correct approach is to allocate `data_size` bytes (the
maximum any single frame can occupy) as the read buffer.

The same applies to ringdump: its per-frame copy buffer is sized from
`rss_ring_get_header(ring)->data_size` so it can always hold the
largest frame the producer can write.

---

## 11. Verified Platforms

The following SoC and sensor combinations have been confirmed working
end-to-end (RVD + RAD + RSD; stream plays in ffplay):

| SoC | SDK Generation | Sensor | Status |
|-----|---------------|--------|--------|
| T20 | Old SDK | jxf23 | Confirmed working |
| T31 | New SDK | gc2053 | Confirmed working (primary target) |

T20 uses the gen1 HAL (`hal_gen1.c`). Notably it has no H265 encoder
and no dynamic bitrate support; RVD falls back to H264 and logs a
warning if H265 is requested in config. All consumer daemons are
codec-agnostic and work without modification.

### 11.1 T20 FPS Characteristics

On T20, JPEG encoder channels are the primary fps bottleneck due to the
SDK's shared encoder resource model:

| Configuration | Measured FPS |
|---------------|-------------|
| No JPEG channels | 30 fps |
| 1 JPEG channel (`jpeg0_enabled=true`, `jpeg1_enabled=false`) | ~25 fps |
| 2 JPEG channels (`jpeg0_enabled=true`, `jpeg1_enabled=true`) | ~23 fps |
| OSD enabled (any config) | no fps impact |

T31 runs full 30fps with all channels (both JPEG + OSD) enabled
simultaneously. Users on T20 who do not need sub-stream JPEG snapshots
should set `jpeg1_enabled = false` in `[jpeg]` config to recover the
lost frames.

Sensor FPS auto-detection reads `/proc/jz/sensor/max_fps` on startup.
Set `fps = 0` (default) in `[sensor]` to use the auto-detected value,
or set a specific integer to override.

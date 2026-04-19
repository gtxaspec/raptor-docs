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
  On platforms where `/proc/jz/sensor` is unavailable (e.g. T41), RVD
  falls back to `[stream0]` width/height config values for sensor
  resolution. T41 also requires `default_boot`, `mclk`, and
  `video_interface` fields in the `[sensor]` config section.
- **Startup info**: Logs LIBIMP version, SYSUTILS version, and CPU info
  (e.g., "T31-AL") via HAL `rss_hal_get_imp_version()` etc.
- **Ring auto-sizing**: When `main_data_mb = 0` (default), RVD
  calculates ring data region size from encoder bitrate:
  `max_frame = bitrate / 8 / fps * 4`, clamped to [8KB..8MB] per slot,
  then `data = max_frame * slot_count`, clamped to [256KB..8MB].
  This avoids over-allocating on 32MB devices and under-allocating on
  high-bitrate streams. Manual override is still supported.
- **IVDC passthrough**: When `ivdc = true` in `[stream0]`, RVD passes
  the flag to the HAL encoder config. IVDC (ISP-VPU Direct Connect)
  replaces full frame buffers with line buffers, significantly reducing
  rmem usage on T23/T32/T41.
- **Stream0 resolution defaults**: When `[stream0]` width/height are
  omitted, RVD defaults to the sensor's native resolution rather than
  hardcoding 1920x1080. This prevents OOM on lower-resolution sensors
  (e.g. 720p JXH63P on T23DL).
- **Outputs**: Two video SHM rings (main, sub), two JPEG SHM rings
  (jpeg0, jpeg1 — one per video stream for snapshots). Snapshots are
  served directly from JPEG rings by RHD; no disk files are written.
- **JPEG channels**: Use encoder channels 4+ (SDK convention), registered
  into video encoder groups via buffer sharing. No separate framesource.
- **JPEG on-demand encoding** (`[jpeg] idle = true`, default): JPEG
  encoder channels are created at pipeline init but NOT started. The
  JPEG encoder thread checks `rss_ring_reader_count()` — when a
  consumer acquires the ring (MJPEG viewer, snapshot, ringdump), the
  thread calls `enc_start` and begins encoding. When all consumers
  release, it calls `enc_stop`. This eliminates VPU overhead when
  nobody is viewing JPEG, which on T20 halves H.264 FPS due to shared
  encoder groups forcing IDRs. A periodic reap (every 10s) detects
  crashed consumers via PID liveness checks and reconciles the count.
  Set `idle = false` to keep JPEG encoding always-on (legacy behavior).
- **IVS integration**: When `[motion] enabled = true` and a sub-stream
  is active, RVD creates an IVS group bound into the sub-stream pipeline
  chain (`FS(1) → IVS(0) → [OSD(1) →] ENC(1)`). The IVS group is
  created before the pipeline bind, and the algo interface/channel/start
  happen before FS enable (SDK requirement). A dedicated IVS poll thread
  stores motion results atomically for RMD to query via control socket.
- **Pipeline bind chain**: The pipeline stages (FS, IVS, OSD, ENC) are
  built as an array per stream and bound in a loop. Unbind walks the
  stored chain in reverse. Adding future pipeline stages is a single
  array insert — no combinatorial if/else.
- **Code structure**: RVD is split into 6 source files:
  - `rvd_main.c` — entry point, daemonization
  - `rvd_pipeline.c` — HAL pipeline setup/teardown, bind chain
  - `rvd_frame_loop.c` — encoder thread management
  - `rvd_ctrl.c` — control socket handler (encoder, ISP, IVS, config)
  - `rvd_osd.c` — OSD region lifecycle and bitmap updates
  - `rvd_ivs.c` — IVS group/channel/algo lifecycle, poll thread
- **Thread model**: RVD runs 6-7 threads:
  - **Main thread**: control socket via epoll, handles raptorctl commands.
  - **4 encoder threads** (128KB stacks): one per channel (main H264,
    sub H264, jpeg0, jpeg1). Each thread independently runs
    enc_poll -> enc_get_frame -> ring_publish_iov -> enc_release_frame. No locks between encoder
    threads — each owns its own ring with no shared mutable state.
  - **OSD update thread** (128KB stack): polls OSD SHM dirty flags and
    calls UpdateRgnAttrData. Runs in a dedicated thread because SDK OSD
    calls can interfere with the encoder if invoked on the same thread.
  - **IVS poll thread** (64KB stack, optional): runs when `[motion]`
    enabled. Calls `ivs_poll_result` → `ivs_get_result` → stores
    motion state atomically. RMD reads via `ivs-status` control command.
- **OSD init sequence**: Follows the vendor SDK spec exactly:
  CreateGroup -> CreateRgn(NULL) -> RegisterRgn -> SetRgnAttr(pData=NULL)
  -> SetGrpRgnAttr(show=0) -> OSD_Start -> System_Bind(FS->OSD->ENC) ->
  FS_Enable. Each region gets a unique layer (1-5). Regions are created
  eagerly with transparent placeholders before the encoder starts.
  Runtime updates use UpdateRgnAttrData with a heap-allocated local_buf
  (SDK DMA requires non-SHM memory). Privacy mode uses OSD_REG_COVER at
  layer 0, toggled via raptorctl.
- **Hot restart**: Individual streams can be torn down and rebuilt at
  runtime without restarting RVD. The pipeline is split into three layers:
  - **Layer 1** (init once): HAL, sensors, framesource channels, OSD pool.
  - **Layer 2** (restartable): encoder group/channel, OSD groups/regions,
    bind chain, SHM ring. `rvd_stream_init()` / `rvd_stream_deinit()`.
  - **Layer 3** (start/stop): FS enable/disable, encoder start/stop,
    encoder thread. `rvd_stream_start()` / `rvd_stream_stop()`.
  Hot restart enables runtime codec switch (H.264↔H.265), resolution
  change, and OSD pool resize (for font size). Consumers (RSD/RWD/RMR)
  detect ring reconnection and adapt automatically. RSD disconnects
  clients on codec change (RTSP can't renegotiate SDP mid-session).
- **IVS and hot restart**: The Ingenic SDK cannot recreate IVS channels
  after destruction without full system reinit. Hot restart uses
  `rvd_ivs_pause()` / `rvd_ivs_resume()` (StopRecvPic/StartRecvPic)
  to keep the channel alive across stream restarts. Full `rvd_ivs_stop()`
  (which destroys the channel) is only used during process shutdown.
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
  cache (O(1) ASCII lookup). The font file is loaded once and shared
  across all font contexts.
- **Per-element font sizing**: Each text element (time, uptime, text)
  can have its own font size via `time_font_size`, `uptime_font_size`,
  `text_font_size` config keys (0 = use global `font_size`). Font
  contexts are stored as `fonts[stream][element]` — 3 per stream.
  Privacy text reuses the time font.
- **Per-stream scaling**: Font sizes auto-scale for sub-streams
  (proportional to resolution ratio) with a minimum of 12px.
- **Runtime font size change**: `set-font-size` (global) or
  `set-time-font-size` etc. (per-element) re-rasterizes glyphs,
  recreates SHM regions with new dimensions, and notifies RVD via
  `osd-restart` to resize the OSD pool and recreate HAL regions.
- **Element show/hide**: `enable-time`, `enable-uptime`, `enable-text`,
  `enable-logo` toggle visibility via RVD's `osd-show` command (single
  HAL `osd_show_region` call — no pipeline restart). Optional stream
  parameter for per-stream control.
- **Element positioning**: `set-position` moves elements to named
  positions (`top_left`, `bottom_right`, `center`, etc.) or absolute
  `x,y` coordinates via RVD's `osd-position` command (single HAL
  `SetRgnAttr` call). Coordinates are clamped to stream dimensions.
  Optional stream parameter.
- **Text alignment**: Left for time, right for uptime, center for
  description and privacy text.
- **Outputs**: OSD SHM double-buffer (one per OSD region per stream
  channel). Sets dirty flag after each update; RVD's OSD thread polls
  dirty flags.
- **RVD coordination**: ROD communicates with RVD via control socket
  for operations that require HAL access (show/hide, positioning,
  OSD restart). SHM dimension probing allows RVD to match region
  sizes to ROD's actual glyph metrics on first open.
- **Render loop**: 1Hz cycle — renders all dirty regions, updates SHM
  double-buffers.
- **Heartbeat**: ROD sends a 1Hz `rss_osd_heartbeat()` (dirty flag
  only, no buffer swap) on one region per stream, so RVD can detect
  liveness even when all OSD elements are static.
- **Why separate**: Font rendering is CPU-intensive relative to the
  frame path. Isolating it lets ROD run at a lower priority without
  risking frame drops. ROD can be killed and restarted — RVD detects
  the restart via SHM inode change (clean exit) or dirty-flag timeout
  (crash/kill -9), clears the OSD to transparent, unlinks orphaned SHM,
  and reopens when ROD comes back.

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
- **Runtime codec switch**: `raptorctl rad set-codec <pcmu|pcma|l16|aac|opus>`
  tears down the audio pipeline (codec, ring, HAL audio), reinitializes
  with the new codec and sample rate (G.711 forces 8kHz), re-applies
  audio effects, and resumes. Old codec state is saved for rollback
  on failure. Config persisted only after success. Consumers detect
  ring reconnection and re-read codec from ring header.
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
- **Protocol**: Standard RTSP 1.0 (DESCRIBE/SETUP/PLAY/TEARDOWN). TCP
  interleaved and UDP transport. Audio track included when RAD is running.
- **Endpoints**: Default mount points: `/stream0` or `/main` (main stream),
  `/stream1` or `/sub` (sub stream), `/` (root, maps to main). Custom
  exclusive endpoints configurable via `endpoint_main` and `endpoint_sub`
  in `[rtsp]` config — when set, only the custom paths are accepted and
  all defaults are disabled. Unknown endpoints return 404 Not Found.
- **Audio codecs over RTSP**: All RAD codecs are supported. SDP and RTP
  packetization are codec-aware: PCMU/PCMA use static payload types,
  AAC uses RFC 3640 (mpeg4-generic with AU header section), Opus uses
  RFC 7587 (48kHz RTP clock), L16 uses dynamic payload type with rtpmap.
- **Backchannel**: SDP advertises a `sendonly` PCMU/8000 backchannel
  track (`a=control:backchannel`). Clients SETUP the backchannel track,
  then send RTP audio via TCP interleaved framing. RSD decodes PCMU to
  PCM16, upsamples 8→16kHz, and writes to the speaker SHM ring. RAD's
  AO thread plays it through the hardware speaker.
- **Client listing**: `raptorctl rsd clients` shows connected clients
  with IP, port, stream index, transport, and video/audio/backchannel state.
- **Authentication**: Digest auth (RFC 2617) via compy, credentials from
  `[rtsp]` config section. Empty = no auth.
- **RTSPS**: TLS-encrypted RTSP via mbedTLS (compile with `TLS=1`).
  Enabled when `tls = true` in `[rtsp]` config. Default port changes
  from 554 to 322 (IANA RTSPS). Cert/key default to the system
  uhttpd paths (`/etc/ssl/certs/uhttpd.crt`, `/etc/ssl/private/uhttpd.key`)
  so `tls = true` alone is sufficient when certs exist.
  Single-socket design — either plain RTSP or RTSPS, not both.
  All TLS code gated behind `#ifdef COMPY_HAS_TLS`. If `tls = true`
  but cert/key fails to load, RSD warns and falls back to plain RTSP
  on port 554.
- **Connection hardening**:
  - Idle timeout (`RSD_IDLE_TIMEOUT_SEC = 60`): clients that have not
    completed RTSP setup and are not actively streaming are disconnected
    after 60 seconds of inactivity. Prevents slowloris-style DoS.
  - Buffer overflow protection: recv buffer is 4KB; connections that fill
    it without completing a request are disconnected (not reset).
  - Max clients (`RSD_MAX_CLIENTS = 8`): enforced under mutex lock to
    prevent race conditions on concurrent accepts.
  - Ring header validation: division-by-zero guard on `slot_count` before
    computing frame buffer sizes.
- **Early exit**: When `enabled = false` in `[rtsp]` config, RSD logs
  "RTSP disabled in config" and exits cleanly without opening any
  sockets or rings. All consumer daemons follow this pattern.
- **Network**: Dual-stack IPv6 (AF_INET6 with IPV6_V6ONLY=0), port 554.
  Per-client RTP timestamp offsets ensure each client sees timestamps
  starting from zero on connect (set on first keyframe delivery).
- **Send architecture**: Per-client send threads decouple the ring reader
  from network I/O. The ring reader reads each frame into `frame_buf`,
  then pushes a malloc copy into each client's send queue (copy-on-push).
  Each client's send thread independently drains its queue through
  blocking TLS/TCP writes. This design eliminates TLS-induced pixelation:
  software mbedTLS on MIPS can consume 30-50% CPU, and blocking TLS
  writes in the reader caused frame skips and IDR storms. With the sendq,
  TLS work is isolated per-client and the reader never blocks on I/O.
  An earlier zero-copy barrier design (atomic refcount on `frame_buf`,
  condvar signal on last release) was replaced by copy-on-push because
  the barrier capped throughput at 1/send_latency on single-core SoCs.
- **Why separate**: RTSP client management (socket I/O, RTP packetization,
  RTCP) is complex and has its own failure modes (slow clients, network
  errors). Isolating it means a misbehaving RTSP client cannot affect
  frame production or recording.

#### RHD -- HTTP Daemon

- **Role**: HTTP server for JPEG snapshots and MJPEG streaming.
- **Inputs**: JPEG SHM rings (jpeg0, jpeg1). Zero disk I/O.
- **Dependencies**: librss_ipc (ring consumer), librss_common.
- **JPEG demand signaling**: RHD opens JPEG rings at startup for metadata
  (dimensions, buffer sizing) but does NOT acquire them — the JPEG encoder
  stays idle. When an MJPEG client connects, RHD calls `rss_ring_acquire()`
  on all JPEG rings, waking the encoder. When the last MJPEG client
  disconnects, it calls `rss_ring_release()`. For snapshots (`/snap.jpg`),
  RHD does a per-request acquire, waits up to 2s for a fresh frame
  (skipping stale data via `write_seq` baseline), then releases. The
  reconnect path re-acquires if MJPEG streaming was active.
- **Endpoints**:
  - `/snap.jpg` or `/snap.jpg?stream=0` — main stream snapshot (1920x1080)
  - `/snap.jpg?stream=1` — sub stream snapshot (640x360)
  - `/mjpeg` — MJPEG stream (multipart/x-mixed-replace from jpeg0 ring)
  - `/audio` — HTTP audio streaming
  - `/` — simple HTML index page with links and embedded MJPEG preview
- **HTTPS**: Supports `[http] https = true` with `cert`/`key` config
  options. Uses `rss_tls_init()` from raptor-common. Falls back to HTTP
  on TLS init failure.
- **Authentication**: HTTP Basic auth via `[http] username` and
  `[http] password` config options (defaults: `admin`/`secret`).
- **Network**: Dual-stack IPv6 (AF_INET6 with IPV6_V6ONLY=0), port 8080.
  Client sockets are set non-blocking (`O_NONBLOCK` via `fcntl`).
  MJPEG frame writes use `poll()` to retry on `EAGAIN` (up to 5 seconds)
  so large main-stream JPEGs complete without dropping the client, while
  truly stalled clients are still disconnected.
- **Request hardening**:
  - Header-line-anchored auth parsing: `Authorization: Basic` is matched
    only at the start of a header line, not inside request bodies.
  - Constant-time credential comparison (`rss_secure_compare`) prevents
    timing side-channel attacks on password checks.
  - Oversized requests (>4KB) receive `414 URI Too Long` and are closed.
  - GET-only: all other methods return `405 Method Not Allowed`.
- **Client listing**: `raptorctl rhd clients` shows connected clients
  with IP and type (mjpeg/snapshot).
- **Send architecture**: Per-client send threads for MJPEG and audio
  streaming. Unlike RSD, RHD reads rings in the main epoll loop (not a
  dedicated reader thread), so a barrier-based zero-copy design would
  block connection handling. Instead, frames are malloc-copied into
  per-client send queues. Each streaming client gets a send thread at
  `/mjpeg` or `/audio` request time; the thread drains the queue through
  blocking HTTP writes. Send errors flag the queue for shutdown;
  the main loop detects this on the next push and removes the client.
  MJPEG frames are independent (no reference frames) so drops are
  harmless — the client just sees a slightly older image.
- **Why separate**: HTTP and RTSP are different protocols with different
  connection lifecycles. RHD can serve snapshots even if RSD is down.

#### RMR -- Recorder

- **Role**: Fragmented MP4 recording daemon. Reads H.264/H.265 and audio
  frames from SHM rings and writes fMP4 segments to SD card (or any
  writable path). Each `moof+mdat` box pair is self-contained, so a
  crash or power loss cannot corrupt already-written segments.
- **Audio codecs**: All RAD codecs — PCMU, PCMA, L16 (PCM fourcc),
  AAC-LC (mp4a + esds), Opus (Opus + dOps). Codec auto-detected from
  the audio ring header.
- **Inputs**: Main video ring (or sub), audio ring.
- **Muxer**: Own lightweight fMP4 muxer — zero external dependencies. No
  libavformat required. The muxer is stateless per-segment; on rotation
  it closes the current file and opens a new one without touching the
  previous segment.
- **Thread model**: Single-threaded. The main loop reads rings, feeds the
  muxer, and writes directly to the file descriptor via `direct_write()`.
  No intermediate circular write buffer — each muxer output callback
  writes immediately to disk with retry on `EINTR`. This eliminates a
  prior class of corruption bugs where NAL data spanning the write
  buffer's wrap boundary could produce garbled frames.
- **Crash safety**: Each `moof+mdat` pair is self-contained and written
  directly to the file descriptor. A crash between fragments leaves all
  prior fragments intact and playable.
- **AVCC conversion**: H.264/H.265 Annex B NAL units are converted to
  AVCC (length-prefixed) format for MP4 boxing. The conversion validates
  NAL length against available AVCC buffer space to prevent integer
  overflow on malformed input.
- **Timestamps**: Video DTS is derived from ring slot timestamps. Audio
  DTS is counter-based (sample count / sample rate), avoiding clock skew
  between ring producers.
- **Recording modes**:
  - `continuous`: always recording, segments rotated by `segment_minutes`.
  - `motion`: records only when triggered (via RMD or `raptorctl test-motion`).
  - `both`: continuous recording plus separate motion clips in `clips/` subdir.
- **Pre-buffer**: Process-local circular buffer stores the last N seconds
  (configurable `prebuffer_sec`, max 5) of AVCC video frames and raw audio
  frames. When motion triggers, the pre-buffer is replayed into the clip
  so footage starts before the event. The pre-buffer is sized from bitrate
  and FPS (~2MB typical). Audio replay is aligned to video duration by
  frame count (not timestamps) to avoid A/V desync across rings.
- **Clip management**: Each clip has its own `clip_write` callback and file
  descriptor — no fd-swapping between continuous and clip muxers.
  Independent DTS bases (video from timestamp, audio from counter) ensure
  clip timestamps always start from zero. `clip_length_sec` caps clip
  duration; if motion continues, a new clip is opened without pre-buffer.
- **Testing**: `raptorctl test-motion [seconds]` sends start/stop directly
  to RMR, bypassing RMD cooldown. Default 10 seconds.
- **Storage management**: Configurable segment rotation (`segment_minutes`
  in config) and storage cleanup (oldest segments deleted when free space
  falls below threshold). Clips have separate storage limit (`clip_max_mb`).
- **Dependencies**: librss_ipc, librss_common. No HAL dependency.
- **Why separate**: File I/O can block (SD card write stalls). Isolating
  the recorder prevents storage latency from affecting the live stream.
  RMR can be started/stopped via raptorctl for event-triggered recording.

#### RWD -- WebRTC Daemon

- **Role**: Send live H.264 + Opus to browsers and go2rtc via WebRTC
  with sub-second latency. WHIP signaling over HTTP.
- **Inputs**: Main or sub video ring (selectable per client), audio ring.
- **Protocol stack**: ICE-lite (STUN binding), DTLS-SRTP (mbedTLS for
  DTLS handshake, compy for SRTP/RTP/NAL transport), WHIP over HTTP.
- **Endpoints**:
  - `GET /webrtc` — HTML5 player page loaded from `/usr/share/raptor/webrtc.html`
  - `POST /whip[?stream=N]` — SDP offer/answer exchange
  - `DELETE /whip/{session}` — session teardown
- **Ports**: UDP for STUN/DTLS/SRTP (`udp_port`, default 8443),
  TCP for HTTP signaling (`http_port`, default 8554).
- **Features**:
  - Main/sub stream selector via query parameter
  - Per-client SSRC declared in SDP for pion/go2rtc compatibility
  - sdes:mid RTP header extension for BUNDLE demux
  - PLI/FIR feedback → IDR request for fast packet loss recovery
  - Consent freshness (RFC 7675): evicts silent clients after 30s
  - SHA256 ciphersuites only (SHA384 PRF incompatible with TLS PRF workaround)
- **Browser support**: Chrome, Edge, Safari. Firefox requires mbedTLS ≥ 3.6.6
  (ClientHello defragmentation, PR #10623).
- **go2rtc**: Compatible as `webrtc:http://camera:8554/whip` source.
- **WebTorrent sharing** *(optional, `WEBTORRENT=1`)*: Enables external
  WebRTC viewing without port forwarding. Camera connects outbound to a
  public WebTorrent tracker (`wss://tracker.openwebtorrent.com`) via TLS
  WebSocket and announces a share room. Viewers open a static HTML page
  with the share key in the URL fragment (`share.html#key=...`). The
  tracker relays SDP offers/answers between viewer and camera. STUN
  discovers the camera's public address (srflx candidate), and ICE
  connectivity checks punch through NAT. Config: `[webtorrent]` section
  — `enabled`, `tracker`, `stun_server`, `stun_port`, `viewer_url`,
  `share_key` (optional, min 4 chars; auto-generated if omitted).
- **Client listing**: `raptorctl rwd clients` shows connected clients
  with IP, stream index, sending state, ICE and DTLS status.
- **Share URL**: `raptorctl rwd share` shows the current WebTorrent share
  URL. `raptorctl rwd share-rotate` generates a new random key.
- **Build**: Requires `TLS=1` and `MBEDTLS_SSL_DTLS_SRTP` enabled.
  WebTorrent requires `WEBTORRENT=1` (adds ~4KB to binary).
- **Config**: `[webrtc]` section — `enabled`, `udp_port`, `http_port`,
  `max_clients`, `cert`, `key`.
- **Dependencies**: librss_ipc, librss_common, libcompy, libmbedtls.
- **Why separate**: DTLS/SRTP state is per-client and heavyweight
  (mbedTLS contexts). Isolating WebRTC keeps the RTSP path simple.

#### RWC -- USB Webcam Daemon

- **Role**: Turns the camera into a USB webcam. Reads JPEG video
  frames from the JPEG ring and raw PCM audio from the audio ring,
  feeds them to the Linux UVC+UAC gadget via V4L2 and `/dev/uac_mic`.
  The host sees a standard USB webcam with microphone.
- **Video**: MJPEG (from jpeg ring) or H.264 (from video ring).
  Bulk USB endpoint — works through any USB hub chain. UVC PROBE/COMMIT
  format negotiation handled in userspace. 1080p/720p/360p at 30/25/15fps.
- **Audio**: 16kHz mono 16-bit PCM via custom UAC1 mic kernel function
  (no ALSA). L16 byte-swap (network→little-endian) done in daemon.
  Isochronous IN endpoint, 32 bytes per 1ms USB frame.
- **Kernel**: Requires patched `g_webcam` module with bulk UVC endpoint,
  H.264 framebased descriptors, and `f_uac_mic.c` composite function.
  `CONFIG_MEDIA_SUPPORT=m` (must be module, not built-in — built-in
  breaks I2C on Ingenic due to V4L2/tx-isp symbol conflicts).
- **USB setup**: `modprobe g_webcam` → `usb-role -m device` →
  `echo connect > .../soft_connect`. RWC must be running before host
  connects (handles UVC control requests from the host).
- **Config**: `[webcam]` section — `enabled`, `device`, `jpeg_stream`,
  `h264_stream`, `buffers`, `audio`, `audio_stream`.
- **Dependencies**: librss_ipc, librss_common. No HAL, no TLS, no compy.
- **Why separate**: USB gadget state is independent from network
  streaming. Isolating webcam keeps the RTSP/WebRTC path unaffected.

#### RSP -- Stream Push *(planned, not yet implemented)*

- **Role**: Push RTMP/RTSP streams to an external server (YouTube,
  cloud NVR, custom endpoint).
- **Inputs**: Main video ring, audio ring.
- **Dependencies**: librss_ipc, libcurl or custom RTMP client.
- **Why separate**: Network push is inherently unreliable (upstream
  bandwidth, server availability). Isolating it prevents network stalls
  from affecting local recording or RTSP serving.

#### RV4 -- V4L2 Bridge *(planned, not yet implemented)*

- **Role**: Expose the video stream as a V4L2 output device
  (`/dev/videoN`) for local consumers (motion detection, ML inference).
- **Inputs**: Sub video ring (decoded or raw NV12 from framesource).
- **Dependencies**: librss_ipc, V4L2 output device kernel support.
- **Why separate**: V4L2 clients may misbehave or consume frames slowly.
  The bridge isolates them from the encoding pipeline.

#### RFS -- File Source *(planned, not yet implemented)*

- **Role**: Replace RVD+RAD on platforms without ISP/encoder hardware
  (A1, x86 testing, development). Reads Annex B H.264/H.265 + raw audio
  from files and publishes to ring buffers, allowing all consumer daemons
  (RSD, RHD, RWD, RMR) to work unmodified against file-backed streams.
- **Codec detection**: Auto-detect codec and resolution from SPS/PPS
  NAL units in the file. FPS from config, configurable loop.
- **Config**: `[filesource]` section — `video_file`, `audio_file`,
  `fps`, `loop`.
- **Dependencies**: librss_ipc, librss_common. No HAL dependency.
- **Why separate**: RFS is a testing/development tool and an alternative
  producer for non-camera platforms. Keeping it separate from RVD avoids
  polluting the production video daemon with file I/O paths.

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
- **Control commands**:
  - `raptorctl ric mode <auto|day|night>` — full switch: GPIO + ISP mode
  - `raptorctl ric isp-mode <day|night>` — ISP running mode only, no GPIO
    toggling. Useful for cameras without IR-cut hardware, or for testing
    ISP color processing independently of the filter.
- **Why separate**: Day/night switching involves GPIO manipulation, ISP
  mode changes, and hysteresis logic. Isolating it keeps the video daemon
  simple and allows the control algorithm to be replaced or tuned without
  restarting video.

#### RMD -- Motion Detection Daemon

- **Role**: Motion detection policy daemon. Queries RVD for IVS
  (Intelligent Video Subsystem) hardware motion results, manages a
  state machine (idle/active/cooldown), and triggers actions on
  motion events.
- **IVS architecture**: RVD owns the ISP pipeline and runs the IVS
  hardware — it creates an IVS group bound to the sub-stream
  FrameSource (`FS(1) → IVS(0) → ENC(1)`) and polls results in a
  dedicated thread. RMD queries the results via RVD's control socket
  (`ivs-status` command). This follows the same split as RVD/RIC:
  RVD does hardware plumbing, RMD owns the detection policy.
- **Detection backends**: RVD supports 4 algorithms selected via
  `[motion] algorithm`:
  - `move` — SDK grid-based motion detection (default, no extra deps)
  - `base_move` — simpler motion flag, no spatial info
  - `persondet` — person detection via `libpersonDet_inf.so` + `libjzdl.so`
  - `yolo` — JZDL standalone YOLOv5 multi-class inference via
    `libjzdl.m.so` + model file
  All backends produce the same `ivs-status` and `ivs-detections`
  responses. RMD is backend-agnostic — it only reads motion/person
  state from RVD. See [28-ivs-detection.md](28-ivs-detection.md) for
  algorithm details.
- **Grid-based ROI** (move algorithm): Default 4x4 grid (16 zones,
  configurable via `grid = NxN`). Each zone reports motion independently.
  Explicit ROI regions can override the grid via `roi_count` + `roi0..roiN`
  config. Sensitivity is per-zone (0-4 for move algo, 0-5 for persondet).
- **Startup sequence**: RMD waits for RVD's IVS to become active
  (polls `ivs-status` until `active=true`), then applies motion config
  to RVD (`ivs-set-sensitivity`, `ivs-set-skip-frames`) before entering
  the main detection loop.
- **State machine**:
  - IDLE → ACTIVE: motion detected → start recording, assert GPIO
  - ACTIVE → ACTIVE: continued motion (reset cooldown timer)
  - ACTIVE → COOLDOWN: motion stops → start cooldown timer
  - COOLDOWN → IDLE: cooldown expired → stop recording, deassert GPIO
  - COOLDOWN → ACTIVE: motion resumes → cancel cooldown
- **Person tracking**: When persondet or yolo is active, RMD tracks
  `person_count` from the `ivs-status` response and reports it via its
  own control socket.
- **Actions**:
  - Recording: sends `start`/`stop` commands to RMR via control socket
  - GPIO: sysfs export/set for alarm output or LED
  - Configurable `record_post_sec` to continue recording after motion stops
- **Control commands**:
  - `status` (default) — state, recording, sensitivity, skip_frames, persons
  - `sensitivity` — set sensitivity and relay to RVD `ivs-set-sensitivity`
  - `skip-frames` — set skip count and relay to RVD `ivs-set-skip-frames`
  - Common commands: `config-get`, `config-get-section`, `config-save`
- **HAL dependency**: None. Pure IPC consumer (same as RIC, RSD, RMR).
- **Dependencies**: librss_ipc (control socket client), librss_common
  (config, logging, cJSON for ctrl command building).
- **IPC**: Polls RVD `ivs-status` every `poll_interval_ms` (default 500ms).
  Own control socket at `/var/run/rss/rmd.sock`.
- **Why separate**: Motion detection policy (cooldown, actions, sensitivity)
  is independent of the IVS hardware pipeline. RMD can be restarted or
  reconfigured without interrupting video streaming or IVS processing.

#### RMC -- Motor Control *(planned, not yet implemented)*

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
  Config commands auto-route to the correct daemon by section name
  (e.g., `config get rtsp port` routes to RSD).

**Global commands:**

| Command | Description |
|---------|-------------|
| `status` | Show running daemons (PID or stopped) |
| `memory` | Per-daemon memory usage: private, shared, RSS, thread stacks (raptor vs SDK), VSIZE. SHM rings and OSD buffers listed separately. "Actual memory" = private + SHM (counts shared once) |
| `cpu` | Per-daemon CPU usage (1s sample) with thread count |
| `config get <section> <key>` | Read live config value (routed to owning daemon) |
| `config get <section>` | Show all keys in section |
| `config set <section> <key> <value>` | Set config value |
| `config save` | Persist running config to disk |
| `<daemon> status` | Show daemon-specific details |
| `<daemon> config` | Show daemon's running config |
| `test-motion [sec]` | Trigger clip recording (default 10s, bypasses RMD) |

**RVD encoder commands** (42 commands — set/get for all HAL encoder ops):

| Command | Description |
|---------|-------------|
| `set-bitrate <ch> <bps>` | Change bitrate |
| `set-gop <ch> <len>` | Change GOP length |
| `set-fps <ch> <fps>` | Change frame rate |
| `set-qp-bounds <ch> <min> <max>` | Change QP range |
| `set-qp <ch> <qp>` | Set fixed QP (all frames) |
| `set-rc-mode <ch> <mode> [bps]` | Change rate control mode |
| `set-qp-ip-delta <ch> <delta>` | I/P frame QP delta |
| `set-qp-bounds-per-frame <ch> ...` | Per-frame QP (iMin iMax pMin pMax) |
| `set-gop-mode <ch> <0\|1\|2>` | GOP mode (default/pyramid/smartP) |
| `set-rc-options <ch> <bitmask>` | RC options bitmask |
| `set-max-same-scene <ch> <count>` | Max same-scene count |
| `set-max-pic-size <ch> <iK> <pK>` | Max I/P frame size (kbits) |
| `set-color2grey <ch> <0\|1>` | Color to greyscale |
| `set-mbrc <ch> <0\|1>` | Macroblock rate control |
| `set-entropy-mode <ch> <0\|1>` | CAVLC/CABAC |
| `set-resize-mode <ch> <0\|1>` | Resize mode |
| `set-stream-buf-size <ch> <bytes>` | Stream buffer size |
| `set-qpg-mode <ch> <mode>` | QPG mode |
| `set-h264-trans <ch> <offset>` | H.264 chroma QP offset |
| `set-h265-trans <ch> <cr> <cb>` | H.265 chroma QP offsets |
| `set-roi <ch> <idx> ...` | ROI region (en x y w h qp) |
| `set-super-frame <ch> <mode> ...` | Super frame (mode iThr pThr) |
| `set-pskip <ch> <en> <maxf>` | P-skip config |
| `set-srd <ch> <en> <level>` | Static refresh detection |
| `set-enc-denoise <ch> ...` | Encoder denoise (en type iQP pQP) |
| `set-gdr <ch> <en> <cycle>` | Gradual decoder refresh |
| `set-enc-crop <ch> <en> <x y w h>` | Encoder crop |
| `set-jpeg-qp <ch> <qp>` | JPEG QP |
| `set-codec <ch> <h264\|h265>` | Change codec (requires restart) |
| `set-resolution <ch> <w> <h>` | Change resolution (requires restart) |
| `stream-restart <ch>` | Restart stream pipeline |
| `request-idr [ch]` | Request keyframe |
| `request-pskip <ch>` | Request P-skip |
| `request-gdr <ch> <frames>` | Request GDR |
| `get-*` variants | Getter for each setter above |
| `get-enc-caps` | Show encoder capabilities |

**RVD ISP commands:**

| Command | Description |
|---------|-------------|
| `set-brightness <val>` | ISP brightness (0-255) |
| `set-contrast <val>` | ISP contrast (0-255) |
| `set-saturation <val>` | ISP saturation (0-255) |
| `set-sharpness <val>` | ISP sharpness (0-255) |
| `set-hue <val>` | ISP hue (0-255) |
| `set-sinter <val>` | Spatial NR (0-255) |
| `set-temper <val>` | Temporal NR (0-255) |
| `set-hflip <0\|1>` | Horizontal flip |
| `set-vflip <0\|1>` | Vertical flip |
| `set-antiflicker <0\|1\|2>` | Off/50Hz/60Hz |
| `set-ae-comp <val>` | AE compensation |
| `set-max-again <val>` | Max analog gain |
| `set-max-dgain <val>` | Max digital gain |
| `set-defog <0\|1>` | Defog enable |
| `set-wb <mode> [r] [b]` | White balance (auto/manual/daylight/etc) |
| `get-wb` / `get-isp` / `get-exposure` | Query ISP state |

**RAD commands:**

| Command | Description |
|---------|-------------|
| `set-codec <codec>` | Change audio codec (restart) |
| `set-volume <val>` | Input volume |
| `set-gain <val>` | Input gain |
| `set-alc-gain <0-7>` | ALC gain (T21/T31 only) |
| `set-ns <0\|1> [0-3]` | Noise suppression level |
| `set-hpf <0\|1>` | High-pass filter |
| `set-agc <0\|1> [target] [comp]` | Automatic gain control |
| `ao-set-volume <val>` | Speaker volume |
| `ao-set-gain <val>` | Speaker gain |

**ROD commands:**

| Command | Description |
|---------|-------------|
| `privacy [on\|off] [ch]` | Toggle privacy mode |
| `set-text <text>` | Change OSD text |
| `set-font-color <0xAARRGGBB>` | Text color |
| `set-stroke-color <0xAARRGGBB>` | Stroke color |
| `set-stroke-size <0-5>` | Stroke width |
| `enable-time <0\|1>` | Show/hide timestamp |
| `enable-uptime <0\|1>` | Show/hide uptime |
| `enable-text <0\|1>` | Show/hide camera text |
| `enable-logo <0\|1>` | Show/hide logo |
| `set-position <elem> <pos>` | Move element |
| `set-font-size <10-72>` | Font size (all elements) |
| `set-time-font-size <10-72>` | Per-element font size |
| `set-uptime-font-size` / `set-text-font-size` | Per-element font size |

**Other daemon commands:**

| Command | Description |
|---------|-------------|
| `rsd clients` | List RTSP clients |
| `rhd clients` | List HTTP clients |
| `rwd clients` | List WebRTC clients |
| `rwd share` | Show WebTorrent share URL |
| `rwd share-rotate` | Generate new share key |
| `ric mode <auto\|day\|night>` | Set day/night mode (GPIO + ISP) |
| `ric isp-mode <day\|night>` | ISP running mode only (no GPIO) |
| `rmd sensitivity <0-4>` | Set motion sensitivity |
| `rmd skip-frames <N>` | Set IVS skip frame count |

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

typedef struct __attribute__((aligned(64))) {
    /* Producer-written, consumer-read */
    _Atomic uint64_t  write_seq;        /* monotonic sequence (release/acquire) */
    _Atomic uint32_t  futex_seq;        /* low 32 bits of write_seq for futex wake/wait */
    uint32_t          slot_count;       /* number of slots (power of 2) */
    uint32_t          data_size;        /* total data region size in bytes */

    /* Data region allocator (producer only) */
    _Atomic uint32_t  data_head;        /* next write offset in data region (relaxed) */

    /* Stream metadata (set once at init via rss_ring_set_stream_info) */
    uint32_t          stream_id;        /* 0=main, 1=sub, 2=jpeg, 0x10=audio */
    uint32_t          codec;            /* rss_codec_t value */
    uint32_t          width, height;
    uint32_t          fps_num, fps_den;
    uint8_t           profile;          /* H.264 profile_idc (66=Base,77=Main,100=High) */
    uint8_t           level;            /* H.264 level_idc (30,31,40,51...) */
    uint16_t          _reserved;

    /* Magic + version for consumer validation.
     * Magic is written last with memory_order_release as a ready flag.
     * Consumer open uses memory_order_acquire. Closes TOCTOU window. */
    _Atomic uint32_t  magic;            /* RSS_RING_MAGIC */
    uint32_t          version;          /* RSS_RING_VERSION */

    /* Incarnation counter -- incremented on each ring creation.
     * Consumers check this on every read to detect producer restart. */
    _Atomic uint32_t  incarnation;

    /* Consumer → producer IDR request. Consumer sets to 1 after
     * EOVERFLOW to get a keyframe fast. Producer checks and clears
     * in its encode loop. Avoids control socket round-trip. */
    _Atomic uint32_t  idr_request;

    /* Demand signaling for on-demand encoding (JPEG idle feature).
     * reader_count tracks active consumers via acquire/release (NOT
     * open/close — a daemon can have a ring open for metadata without
     * signaling demand). reader_pids stores PIDs for crash detection:
     * the reap function checks if stored PIDs are alive and reconciles
     * reader_count if orphaned. */
    _Atomic uint32_t  reader_count;
#define RSS_RING_MAX_READERS 4
    _Atomic uint32_t  reader_pids[RSS_RING_MAX_READERS];
} rss_ring_header_t;

#define RSS_RING_MAGIC    0x52535352   /* "RSSR" */
#define RSS_RING_VERSION  3            /* v3: adds reference mode */

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

/* Demand signaling -- consumers call acquire/release to indicate they
 * actively want data. open/close do NOT touch reader_count, so a
 * daemon can keep a ring mapped for metadata without activating the
 * producer. The producer checks reader_count to decide whether to
 * encode (used by JPEG on-demand feature).
 *
 * acquire: increments reader_count, stores getpid() in a PID slot.
 * release: decrements reader_count, clears own PID slot.
 * reap:    producer calls periodically to detect crashed consumers
 *          via kill(pid, 0). Reconciles reader_count to match live
 *          PIDs, fixing orphaned counts from SIGKILL/crashes. */
void         rss_ring_acquire(rss_ring_t *ring);
void         rss_ring_release(rss_ring_t *ring);
uint32_t     rss_ring_reader_count(rss_ring_t *ring);
uint32_t     rss_ring_reap_dead_readers(rss_ring_t *ring);
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

#### Reference Mode (v3, zero-copy)

When `[ring] refmode = true`, the ring operates in reference mode: frame
data lives in an external shared memory region instead of the ring's data
region. The ring carries metadata only (~9KB per stream vs ~1-2MB).

**Two backing store paths:**

| Encoder IP | SoCs | Backing store | How it works |
|---|---|---|---|
| Allegro | T31, T40, T41 | `/dev/rmem` | Encoder DMA's to reserved memory. Consumer mmaps `/dev/rmem` at the physical base offset stored in `ref_rmem_offset`. |
| Ingenic VPU | T10-T30, T32, T33 | Named POSIX SHM | HAL probes libimp's channel struct at runtime and injects a SHM buffer address before `CreateChn`. Encoder writes to SHM via DMMU-backed DMA. Consumer opens `/rss_enc_<ring_name>`. |

The consumer path is transparent: `rss_ring_read()` copies from whichever
backing store is active. On open, the consumer tries named SHM first, then
falls back to `/dev/rmem`.

**v3 header additions:**

```c
uint32_t flags;                    /* RSS_RING_FLAG_REFMODE (0x01) */
uint32_t ref_buf_count;            /* encoder output buffer count  */
uint32_t ref_rmem_size;            /* backing store mmap size      */
uint32_t ref_rmem_offset;          /* /dev/rmem mmap file offset   */
uint32_t ref_buf_stride;           /* per-buffer size in bytes     */
_Atomic uint32_t ref_buf_gen[8];   /* per-buffer generation counter */
```

**v3 slot additions:**

```c
uint8_t  buf_idx;   /* encoder buffer index (was _pad)      */
uint32_t buf_gen;   /* generation at publish time            */
```

The generation counter detects stale reads: the producer increments
`ref_buf_gen[buf_idx]` before publishing a new frame to the same buffer.
The consumer compares the slot's `buf_gen` against the header's current
generation after its memcpy — a mismatch means the buffer was reused
during the copy.

**Producer API:**

```c
int rss_ring_enable_refmode(rss_ring_t *ring, uint32_t rmem_size,
                            uint32_t rmem_offset, uint8_t buf_count,
                            uint32_t buf_stride);

int rss_ring_publish_ref(rss_ring_t *ring, uint32_t data_offset,
                         uint32_t length, int64_t timestamp,
                         uint16_t nal_type, uint8_t is_key,
                         uint8_t buf_idx);
```

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
/var/run/rss/rhd.sock    -- RHD control socket
/var/run/rss/rmr.sock    -- RMR control socket
/var/run/rss/ric.sock    -- RIC control socket
/var/run/rss/rmd.sock    -- RMD control socket
/var/run/rss/rwd.sock    -- RWD control socket
/var/run/rss/rwc.sock    -- RWC control socket
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
Audio Encoder (PCMU / PCMA / L16 / AAC / Opus)
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
        # Start remaining daemons — each checks its own [section] enabled flag
        # and exits cleanly if disabled, so all can be started unconditionally.
        start-stop-daemon -S -b -x /usr/bin/rad -- -c /etc/raptor.conf
        start-stop-daemon -S -b -x /usr/bin/rod -- -c /etc/raptor.conf
        start-stop-daemon -S -b -x /usr/bin/rsd -- -c /etc/raptor.conf
        start-stop-daemon -S -b -x /usr/bin/rhd -- -c /etc/raptor.conf
        start-stop-daemon -S -b -x /usr/bin/rmr -- -c /etc/raptor.conf
        start-stop-daemon -S -b -x /usr/bin/ric -- -c /etc/raptor.conf
        start-stop-daemon -S -b -x /usr/bin/rmd -- -c /etc/raptor.conf
        ;;
    stop)
        # Stop consumers first, then producers, then HAL owner
        start-stop-daemon -K -x /usr/bin/rmd
        start-stop-daemon -K -x /usr/bin/rmr
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

**Core vs. optional daemons:** The default init script starts the five
core daemons unconditionally: RVD, RAD, ROD, RSD, and RIC. The
remaining daemons (RHD, RMR, RMD, RWD, RWC) are optional -- enable
them in `raptor.conf` and add their `start-stop-daemon` lines to the
init script, or start them manually. Each daemon checks its own
`[section] enabled` flag and exits cleanly if disabled, so adding a
line for an optional daemon is safe even when the feature is off.

---

## 5. Crash Recovery

### 5.1 Per-Daemon Restart

Each daemon is designed to be individually restartable:

| Daemon | Restart Impact | State Lost | Recovery |
|--------|---------------|------------|----------|
| RVD | **Full pipeline restart.** All consumers lose their rings. | All SHM rings destroyed. | Consumers detect stale ring (magic/version mismatch or unlink) and reconnect. |
| ROD | OSD cleared to transparent within ~3s. | Rendered bitmap state, glyph cache. | RVD detects restart (inode change) or crash (3s dirty timeout), clears OSD, unlinks orphaned SHM. New ROD recreates SHM, RVD reopens and OSD resumes. |
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

### 6.2 Memory Diagnostics (`raptorctl memory`)

```
DAEMON    PRIVATE     SHARED        RSS   STACK raptor      STACK sdk      VSIZE
------    -------     ------        ---   ------------      ---------      -----
rvd       3472 KB    2972 KB    6444 KB    80/  756 KB   152/38836 KB  112900 KB
rsd        320 KB    2644 KB    2964 KB    28/  384 KB   112/  644 KB    5472 KB
...
```

Column definitions:

- **PRIVATE**: RSS pages not shared with other processes (heap, BSS,
  touched stack pages). From `/proc/pid/smaps` Private_Clean +
  Private_Dirty.
- **SHARED**: RSS pages shared with other processes (shared libraries,
  SHM rings). From `/proc/pid/smaps` Shared_Clean + Shared_Dirty.
- **RSS**: PRIVATE + SHARED. Total physical RAM footprint.
- **STACK raptor**: used/allocated stack for raptor-owned threads
  (identified by stack size <= 256KB, set via `pthread_attr_setstacksize`).
- **STACK sdk**: used/allocated stack for SDK/library threads (default
  ~2MB stacks from libc). On RVD, the Ingenic SDK creates ~20 internal
  threads with 2MB stacks but uses only ~8KB each.
- **VSIZE**: total virtual address space. Includes SDK kernel-mapped
  VPU/ISP/DMA regions that don't consume user RAM. RVD shows ~113MB
  VSIZE but only ~6.4MB RSS — the difference is reserved encoder memory.

Stack values show virtual allocation (used/reserved). Only touched
pages count toward PRIVATE — this is why STACK can exceed PRIVATE.

The SHM breakdown lists individual ring buffers and OSD double-buffers
with their sizes. SHM is counted once in the total (shared across
all daemons that map the same ring).

### 6.3 Optimization Strategies

- **SHM ring auto-sizing**: When `main_data_mb = 0`, RVD auto-sizes ring
  data regions from encoder bitrate. For a 720p@25fps stream at 1Mbps,
  auto-sizing allocates ~640KB vs the old fixed 4MB — an 84% reduction.
  This is critical for 32MB devices where every KB counts.
- **IVDC (ISP-VPU Direct Connect)**: On T23/T32/T41, setting `ivdc = true`
  in `[stream0]` replaces full frame buffers in rmem with line buffers,
  reducing encoder DMA memory by ~60%. Requires kernel module parameter
  `direct_mode=1 ivdc_mem_line=<lines>` in `/etc/modules.d/20-isp`.
- **ROD glyph cache**: Pre-render digits 0-9 and common punctuation at
  startup. Reuse bitmaps for timestamp updates (only re-render changed
  digits).
- **Single libimp.so load**: RVD and RAD both link against the vendor
  library, but as separate processes they each get a copy. If memory is
  critical, RAD audio init could be folded into RVD (sacrificing isolation).
- **Consumer-side copy-on-push**: RSD reads from SHM ring into a shared
  `frame_buf`, then pushes malloc copies to per-client send queues. The
  copy overhead is negligible compared to TLS encryption.
- **Static linking**: All daemons can be statically linked to avoid
  dynamic linker overhead and reduce per-process memory slightly.

### 6.4 T23 (32MB Systems)

On 32MB systems (T23DL: `mem=22M` + `rmem=10M`), Linux sees ~16MB.
Confirmed working with RVD + RSD + RAD (video + RTSP + audio):

```
Raptor memory breakdown (T23DL, main+sub+audio):
  RVD private:        2,524 KB
  RSD private:          396 KB
  RAD private:          596 KB
  Shared libs:          852 KB
  SHM rings:            525 KB  (main 260 + sub 132 + audio 133)
  Total:             ~3,813 KB
```

Recommended config for 32MB:

```ini
[stream0]
bitrate = 1500000
rc_mode = vbr
ivdc = true              # required for dual-stream on 10MB rmem

[stream1]
enabled = true
width = 320
height = 176             # must align to 8 (180 fails IMP SDK check)
fps = 15
bitrate = 300000
jpeg = false
osd_enabled = false

[ring]
main_slots = 4
main_data_mb = 0         # auto-size from bitrate (~260KB at 1.5Mbps)
sub_slots = 4
sub_data_mb = 0          # auto-size (~132KB at 300kbps)
```

Key optimizations for 32MB:
- **IVDC required**: Without IVDC, `IMP_FrameSource_CreateChn(1)` fails
  on 10MB rmem — the scaler framebuffer doesn't fit. IVDC bypasses
  frame buffers, enabling dual-stream.
- **Sub stream height alignment**: IMP SDK requires width aligned to 16,
  height aligned to 8. 320x180 fails; use 320x176.
- **Ring slot count**: 4 slots is sufficient for 2 RTSP clients.
  Auto-sizing (`_data_mb = 0`) sizes from bitrate, avoiding over-allocation.
- **Disable sub JPEG/OSD**: Each costs encoder channel memory in rmem
  and ring buffer memory in Linux.
- **Lower bitrate**: 1.5Mbps VBR at 720p is adequate for surveillance
  and halves ring allocation vs 3Mbps CBR.

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
# boot = 0             # T41: default_boot index
# mclk = 0             # T41: sensor master clock source
# video_interface = 0  # T41: video interface type (MIPI/DVP)

[stream0]
# width/height omitted = default to sensor native resolution
# width = 1920
# height = 1080
fps = 25
codec = h264           # h264, h265, jpeg
profile = 2            # 0=baseline, 1=main, 2=high
bitrate = 2000000
rc_mode = vbr          # cbr, vbr, fixqp, capped_vbr
gop = 50
min_qp = 15
max_qp = 45
ivdc = false           # ISP-VPU Direct Connect (T23/T32/T41 only)

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
main_data_mb = 0       # 0 = auto-size from encoder bitrate
sub_slots = 32
sub_data_mb = 0        # 0 = auto-size from encoder bitrate

[audio]
enabled = true
sample_rate = 16000
codec = aac            # pcmu, pcma, l16, aac, opus
# bitrate = 32000      # for aac/opus (bits per second, max 256000)
volume = 80
gain = 25

[rtsp]
enabled = true
port = 554
max_clients = 4
# endpoint_main = ch0   # custom exclusive endpoint (disables defaults)
# endpoint_sub = ch1    # custom exclusive endpoint (disables defaults)
# username = admin       # digest auth (both required to enable)
# password = secret
# tls = false            # enable RTSPS (default port changes to 322)
# tls_cert = /etc/ssl/certs/uhttpd.crt
# tls_key = /etc/ssl/private/uhttpd.key
# tls_cipher_preference = default   # default | chacha20
#   default  — mbedtls built-in order (GCM-first in TLS 1.3); best on
#              x86/ARM with AES-NI or ARMv8-crypto.
#   chacha20 — force TLS 1.3 to ChaCha20-Poly1305 only. Optimal on slow
#              MIPS SoCs where AES runs through /dev/aes — every GCM
#              record costs an ioctl, scalar userspace ChaCha20 wins at
#              typical RTSPS bitrates (~3 Mbps).

[http]
enabled = true
port = 8080

[osd]
enabled = true
font = /usr/share/fonts/default.ttf
font_size = 24             # main stream; sub stream auto-scaled
stream1_font_size = 12     # optional per-stream override
timestamp = true
timestamp_fmt = %Y-%m-%d %H:%M:%S
label = Camera
# logo = /usr/share/images/thingino_100x30.bgra  # BGRA file (omit to disable)
# logo_width = 100
# logo_height = 30
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

[recording]
enabled = false
mode = continuous             # continuous, motion, or both
stream = 0
audio = true
segment_minutes = 5
storage_path = /mnt/mmcblk0p1/raptor
max_storage_mb = 0            # 0 = unlimited
prebuffer_sec = 5             # seconds of pre-event footage in motion clips (0-5)
clip_length_sec = 60          # motion clip max duration before rotation
clip_max_mb = 100             # max total clip storage

[motion]
enabled = false
algorithm = move              # move (ROI, sensitivity 0-4) or base_move (global, 0-3)
sensitivity = 3
grid = 4x4                    # auto-generate ROI grid (default 4x4 = 16 zones)
cooldown_sec = 10
poll_interval_ms = 500
record = true                 # start RMR recording on motion
record_post_sec = 30          # continue recording after motion stops
# gpio_pin = -1               # GPIO to assert on motion
# Explicit ROI override (disables grid):
# roi_count = 2
# roi0 = 0,0,319,179
# roi1 = 320,180,639,359

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
base64-decodes it, and compares the `user:pass` string against config
using `rss_secure_compare()` (constant-time comparison to prevent timing
side-channel attacks). The `Authorization` header is matched only at
line boundaries to prevent false matches inside request bodies.
If credentials are empty, auth is skipped (same backwards-compatible
logic as RSD).

---

## 10. Latency

### 10.1 Pipeline Latency (measured)

Server-side pipeline latency from sensor capture to ring availability
is ~2ms average (measured with `ringdump main -l`):

```
--- latency ---
Min:  -1.3 ms    (jitter from frame timing)
Avg:   1.9 ms
Max:   5.3 ms
```

Full end-to-end breakdown:

| Stage | Latency |
|-------|---------|
| Sensor → ISP → Encoder → Ring | ~2ms |
| Ring → RTP packetization | <1ms |
| Network (wired LAN) | ~2ms |
| **Server total** | **~5ms** |
| WebRTC client (browser) | ~50ms |
| RTSP client jitter buffer | 100-500ms (client-dependent) |

### 10.2 Low-Latency Optimizations

The following are applied by default or configurable:

1. **ISP frame depth = 0** — frames delivered to encoder immediately,
   no ISP-side queuing (set in `rvd_pipeline.c`).

2. **Encoder immediate output** (`low_latency = true` in config) —
   sets `max_stream_cnt = 1` so the encoder releases each frame
   immediately instead of batching 2-4 frames. Saves ~40-120ms.

3. **TCP send buffer = 64KB** — reduced from 256KB. At 2Mbps, 256KB
   queues ~1 second of data in the kernel. 64KB = ~250ms. Configurable
   via `[rtsp] tcp_sndbuf`.

4. **Stale frame skip** — if the ring consumer (RSD/RWD) falls behind,
   it skips to the latest frame instead of playing through the backlog.
   Requests IDR after skipping. Prevents accumulated frames from adding
   multiple frame periods of latency.

5. **Futex-based ring wakeup** — producer wakes consumers instantly
   via `futex_wake()` on `write_seq`. No polling or sleep between
   frames.

6. **TCP_NODELAY** — set on all RTSP TCP connections to disable Nagle
   buffering.

### 10.3 Client-Side Tuning

Most perceived RTSP latency is from the client's jitter buffer:

```sh
# ffplay (~50ms)
ffplay -fflags nobuffer -flags low_delay -rtsp_transport udp rtsp://camera/stream0

# mpv (~50ms)
mpv --no-cache --untimed --profile=low-latency rtsp://camera/stream0

# VLC (~100ms with tuning, 300ms default)
vlc --network-caching=0 rtsp://camera/stream0
```

WebRTC (via RWD) has inherently low latency (~50ms total) since
browsers use minimal jitter buffering.

### 10.4 End-to-End Measurement (`rlatency`)

`rlatency` is a standalone RTSP client that measures true end-to-end
latency (sensor capture → network → receive) using RTCP Sender Report
NTP/RTP timestamp correlation per RFC 3550 §6.4.1.

How it works:
1. Connects via RTSP, receives RTP video packets + RTCP SRs over UDP
2. Each SR provides a mapping: `NTP_wall_clock ↔ RTP_timestamp`
3. For each received frame, maps its RTP timestamp to NTP time via
   the SR correlation: `frame_ntp = sr.ntp + (rtp_ts - sr.rtp_ts) / clock`
4. Latency = `local_time - frame_ntp`
5. Uses signed 32-bit RTP timestamp difference for wrap handling

Requirements: NTP-synchronized clocks on both camera and host.
Accuracy limited by NTP sync quality (~1-20ms). Relative measurements
(jitter, percentiles) are accurate regardless of clock offset.

```sh
rlatency rtsp://camera/stream0 -n 500
```

```
--- latency (500 frames) ---
  Min:     -14.94 ms    (negative = NTP offset)
  Avg:      -5.00 ms
  Max:      27.65 ms
  StdDev:   13.99 ms
  P50:     -10.84 ms
  P95:      24.34 ms
  P99:      27.21 ms
```

Adjusting for NTP offset (~14ms), real latency is ~5-27ms over wired
LAN with UDP transport.

### 10.5 A/V Sync

Both video and audio RTP timestamps derive from `meta.timestamp`
(IMP's `CLOCK_MONOTONIC_RAW`, microsecond precision). Same clock
source = zero inter-stream drift by construction. Measured 7ms
delta over 1 hour (noise, not drift). Previous counter-based
approach drifted ~10.6 seconds per day due to audio ADC crystal
offset.

Audio is gated on the first video keyframe so both RTP timelines
start together, eliminating initial audio lead.

---

## 11. JPEG Encoding

### 11.1 Dynamic Buffer Sizing

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

### 11.2 On-Demand Encoding (JPEG Idle)

JPEG channels share encoder groups with H.264 on the Ingenic VPU.
Active JPEG encoding forces IDR resets on the H.264 channel, which on
T20 halves the main stream FPS from 28+ to ~14 fps. Other SoCs are
less affected but still pay a VPU cost.

The `[jpeg] idle = true` config (default) makes JPEG encoding on-demand:

- **Pipeline init**: JPEG encoder channels are created and registered
  into encoder groups, but `enc_start` is NOT called. The ring is
  created normally.
- **Encoder thread**: Checks `rss_ring_reader_count()` each iteration.
  If 0, sleeps 100ms and skips `enc_poll`. If > 0, calls `enc_start`
  (if not already started) and enters the normal encode loop.
- **Consumer lifecycle**: RHD calls `rss_ring_acquire()` when MJPEG
  clients connect or snapshot requests arrive, and `rss_ring_release()`
  when they disconnect or complete. ringdump acquires on open, releases
  on close.
- **Crash recovery**: Every 10s, the encoder thread calls
  `rss_ring_reap_dead_readers()` which checks stored PIDs via
  `kill(pid, 0)`. Dead PIDs are cleared and `reader_count` is
  reconciled to match live PIDs. Handles SIGKILL, crashes, and
  orphaned counts. Maximum 10s window of unnecessary encoding after
  a consumer crash.
- **Config**: Set `[jpeg] idle = false` for always-on JPEG encoding
  (legacy behavior, encoder starts at pipeline init).

---

## 12. Verified Platforms

The following SoC and sensor combinations have been confirmed working
end-to-end (RVD + RAD + RSD + ROD + RHD + RMR; stream plays in ffplay,
fMP4 recording verified):

| SoC | SDK Generation | Sensor | RAM | Status |
|-----|---------------|--------|-----|--------|
| T20 | Old SDK (XB1) | jxf23 | 64MB | Confirmed working (H.264 only) |
| T23DL | Old SDK (XB1) | jxh63p | 32MB | Confirmed working (720p, auto-ring sizing) |
| T30 | Old SDK (XB1) | sc1235 | 128MB | Confirmed working (H.265 + H.264) |
| T31 | New SDK (XB2) | gc2053 | 128MB | Confirmed working (primary target) |
| T40 | IMPVI SDK (XB2) | imx307 | 128MB | Confirmed build (AWB via SetAwbAttr) |
| T41 | IMPVI SDK (XB2) | gc4023 | 128MB | Confirmed working (2560x1440, dual-stream) |

T20 uses the old SDK HAL path. Notably it has no H265 encoder
and no dynamic bitrate support; RVD falls back to H264 and logs a
warning if H265 is requested in config. All consumer daemons are
codec-agnostic and work without modification.

T23DL is the most constrained platform (32MB RAM, 10MB rmem). IVDC is
required for dual-stream — without it the second framesource channel
fails to allocate. Confirmed running RVD+RSD+RAD with main 720p +
sub 320x176 + audio, ~3.8MB total raptor memory.

T41 uses the IMPVI SDK (XB2 generation). It does not need the uclibc
shim library — native uclibc libs work directly, and linking the shim
causes crashes. The Makefile auto-detects shim availability and skips
it on T40/T41. T41 requires additional sensor config fields
(`default_boot`, `mclk`, `video_interface`) and uses `[stream0]`
config values as sensor resolution fallback since `/proc/jz/sensor`
is not available.

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

### 11.2 T41 gc4023 FPS Characteristics

The gc4023 sensor on T41 reports 25fps capability but measures ~19fps
in low-light conditions. This is a sensor-level limitation — AE
(auto-exposure) integration time at low light produces ~54ms frame
intervals. Increasing ambient light recovers full frame rate. This is
not an encoder or OSD bottleneck — the ISP frame counter (`isp-w02`)
confirms the reduced rate originates at the sensor.

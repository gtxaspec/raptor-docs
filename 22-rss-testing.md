# Raptor Streaming System -- Test Architecture

Testing spans four layers: unit tests on x86 with AddressSanitizer,
full-daemon ASAN integration with a mock HAL, on-device validation on
real cameras, and performance benchmarking. 453 unit tests across the
raptor ecosystem pass under ASAN with zero errors.


## 1. Unit Test Inventory

All unit tests use the [greatest.h](https://github.com/silentbicycle/greatest)
single-header C test framework. Every test binary compiles with
`-fsanitize=address` (AddressSanitizer) to catch memory errors at test
time. Raptor repos build and run tests with `make test` in each `tests/`
directory. Compy uses CMake.

### 1.1 raptor-common (86 tests, 9 suites)

| Suite | File | Tests | Coverage |
|-------|------|------:|----------|
| config | test_config.c | 22 | INI parser: load, get/set str/int/bool, sections, iteration, roundtrip, foreach, edge cases incl. over-long lines rejected whole (no silent value truncation) and exactly-buffer-filling final lines |
| util | test_util.c | 20 | strlcpy, trim, starts_with, secure_compare — truncation, empty, exact fit |
| json | test_json.c | 6 | JSON string/int extraction, missing keys, truncation |
| http | test_http.c | 10 | Base64 decode (padding, overflow), HTTP basic auth validation |
| file | test_file.c | 9 | Read/write file, atomic write, mkdir_p, timestamp formatting |
| sei | test_sei.c | 7 | MISB ST 0604 timecode SEI: H.264/H.265 byte layout, prefix modes, build/parse round-trip, no-start-code-emulation sweep, malformed rejection, multi-message NALs |
| jpeg | test_jpeg.c | 3 | EXIF capture-time insert/extract round-trip, APP15 Ed25519 sign/verify (tamper, wrong key, keyless presence check, unsigned input), capacity and non-JPEG rejection |
| aac | test_aac.c | 6 | AudioSpecificConfig generation: LC and HE-AAC (BC syncExtension form), rate index mapping, invalid rates |
| ts | test_ts.c | 3 | MPEG-TS mux: PMT descriptors, Opus channel config, PAT/PMT layout |

```
cd raptor-common/tests && make test
```

### 1.2 raptor-ipc (29 tests, 3 suites)

| Suite | File | Tests | Coverage |
|-------|------|------:|----------|
| ring | test_ring.c | 21 | SHM ring create/open, publish/read, sequential access, overflow detection and recovery, multi-consumer, data region wrap, keyframe seek, wait timeout, IOV publish, stream info, IDR request, demand count, dead reader reaping, version/magic validation, UTC clock mapping (seqlock set/get, absent mapping, producer-only) |
| osd | test_osd.c | 5 | OSD double-buffer lifecycle, publish/read, dirty flag, heartbeat |
| ctrl | test_ctrl.c | 3 | Control socket roundtrip (threaded echo), handler error, stall timeout |

```
cd raptor-ipc/tests && make test
```

### 1.3 raptor (204 tests, 17 suites)

| Suite | File | Tests | Coverage |
|-------|------|------:|----------|
| ric_json | test_ric_json.c | 11 | thingino.json GPIO discovery: int and dual-string pins, raptor.conf precedence, range limits, missing/empty/garbage/no-gpio/over-cap files, and the >2 KB document with pins at the tail (regression for the truncated-read bug) |
| nal | test_nal.c | 12 | Annex B → AVCC conversion (H.264, H.265, 3-byte start codes), empty/overflow, single NAL, SPS/PPS extraction |
| prebuf | test_prebuf.c | 9 | Pre-buffer push/count, slot eviction, keyframe find by age, frame-at lookup, iterate with data verification, data region wrap integrity |
| mux | test_mux.c | 17 | fMP4 box construction (moov/moof/mdat/mfra), H.264/H.265/PCMU/AAC/Opus codec paths, multi-fragment, large timestamps (64-bit tfdt), CTS offset, empty/double flush, A/V duration verification |
| sdp | test_sdp.c | 17 | SDP offer parsing, answer generation, ICE candidate extraction, codec negotiation |
| codec | test_codec.c | 28 | Codec detection, parameter set parsing, frame type identification |
| ring | test_ring.c | 14 | Ring buffer create/open, publish/read, overflow, IDR request, demand signaling |
| rsp_url | test_rsp.c | 8 | RTMP/RTMPS URL parsing, port detection, edge cases (missing scheme/app/key, complex keys) |
| rsp_amf0 | test_rsp.c | 8 | AMF0 number/string/null/object encoding, property names, connect command structure |
| rsp_chunk | test_rsp.c | 5 | Big-endian helpers, chunk header layout, extended timestamps, fmt=3 continuation, protocol constants |
| rsp_flv | test_rsp.c | 10 | H.264 sequence header (AVCDecoderConfigurationRecord), H.264 keyframe/inter tags, H.265 Enhanced RTMP FourCC, AAC sequence header, AudioSpecificConfig at 16/48kHz, FLV constants |
| rsp_audio | test_rsp.c | 6 | G.711 µ-law decode (minimum step, full range, symmetry), A-law decode (silence, full range), L16 byte-swap |
| rsp_annexb | test_rsp.c | 4 | Start code detection (3-byte, 4-byte, none, multiple) |
| rsp_state | test_rsp.c | 6 | RTMP state machine: set_chunk_size (incl. MSB mask), _result advancing connect/createStream, onStatus → publishing, _error → error state |
| ctrl | test_ctrl.c | 43 | RVD control handlers against the mock HAL vtable: encoder get/set commands, table-driven enc-set/get/list, ISP commands, JSON error paths |
| sign | test_sign.c | 3 | Recording signature chain: key generation/reload/pub export, span hash + Ed25519 chain verification, tamper and wrong-key rejection, empty-span no-op |
| storage | test_storage.c | 3 | Storage availability: existing dir, auto-create on non-root filesystems, refusal to auto-create on the root filesystem |

```
cd raptor/tests && make test
```

### 1.4 compy (150 tests, 27 suites)

Compy is the RTSP/RTP/WebRTC C library used by RSD and RWD. Tests build
with CMake and run via `ctest`.

| Domain | Suites | Tests | Coverage |
|--------|-------:|------:|----------|
| RTSP types | 13 | ~30 | header_map, header, message_body, method, reason_phrase, request_line, request_uri, request, response_line, response, rtsp_version, sdp, status_code |
| NAL parsing | 3 | ~19 | H.264 FU-A, H.265 FU, generic NAL framing |
| Protocol | 7 | ~40 | transport, receiver, controller, context, writer, io_vec, util |
| RTP/RTCP | 3 | ~38 | RTCP types, RTCP logic, receiver reports |
| Security | 2+ | ~23 | auth, base64, SRTP (conditional on TLS), DTLS (conditional on TLS) |

```
cd compy && cmake -B build && cmake --build build && ctest --test-dir build
```

---

## 2. ASAN Integration Testing

Full-daemon integration tests run all raptor daemons on the x86 host
under AddressSanitizer, using a mock HAL instead of real hardware.

### 2.1 Mock HAL

`tests/mock_hal.c` implements the full `rss_hal_ops_t` vtable with
deterministic synthetic frames. RVD, RAD, and ROD link against the mock.
RSD, RHD, RIC, RMD, RMR, RWD, and RWC have no HAL dependency and build
natively. RWD links against the host's mbedTLS for DTLS-SRTP.

`./build-asan.sh` compiles all daemons to `asan-out/` with
`-fsanitize=address,undefined -fno-omit-frame-pointer`.

### 2.2 Synthetic Ring Setup

`tests/create_rings.c` creates SHM rings pre-populated with fake JPEG
frames. This allows RHD and ringdump to exercise their buffer paths
without RVD running.

### 2.3 Stress Test Results

| Workload | Volume | Result |
|----------|--------|--------|
| HTTP requests to RHD | 4,000 (snap.jpg + mjpeg) | Zero leaks |
| RTSP connection cycles to RSD | 400 | Zero leaks |
| RMR segment rotations | 100 | Zero leaks, zero leaked FDs |
| RMR mock frames processed | 10,000 | Zero UBSan errors |

All 6 daemons ran simultaneously. Zero memory leaks and zero
UndefinedBehaviorSanitizer errors reported at exit.

### 2.4 RMR Memory Safety Audit

A dedicated audit pass over RMR source found and fixed these issues
before release:

| Severity | Issue | Fix |
|----------|-------|-----|
| CRITICAL | SPSC circular write buffer corrupted NAL data at wrap boundaries | Removed SPSC wbuf entirely; muxer writes directly to fd (single-threaded) |
| CRITICAL | `malloc()` return unchecked in muxer box-building paths | All allocations now checked; fatal-log + clean shutdown on OOM |
| HIGH | Integer overflow in box size calculation for large frames (>16MB) | Added overflow check before size accumulation |
| HIGH | Audio DTS counter not reset on segment rotation | Counter now resets to 0 at each new segment |
| HIGH | Unchecked `write()` return on full disk | Direct write retries on `EINTR`, logs and stops recording on `ENOSPC` |
| HIGH | Storage cleanup walked directory with `readdir()` on the recording path without re-opening on each rotation | `opendir`/`closedir` now called per-cleanup pass |
| HIGH | Segment file path buffer (256 bytes) could overflow with long base paths | Buffer increased to `PATH_MAX`; truncation now caught |

### 2.4 RIC Behavior Suite

`tests/test-ric.sh` drives ric through every trigger, opmode, GPIO
topology and failure mode with no hardware and no privileges. The
harness (`tests/ric_harness.py`) runs inside an unprivileged
`unshare -rm` namespace where everything ric touches is fake and
observable:

- **AE**: a stub `rvd.sock` speaking the real length-prefixed ctrl
  protocol serves a scriptable scene and records `set-running-mode`
  calls. It can die and come back to drive outage scenarios.
- **GPIO**: tmpfs over `/sys` with pre-created `gpioN` files;
  `gpio_export()` takes its already-exported path, so every actuation
  is a plain write observed via inotify -- sequences and pulse widths.
- **ADC**: an LD_PRELOAD shim (`tests/ric_adc_shim.c`) stands in for
  `/dev/ingenic_adc_aux_N`, backed by a rewritable file. The shim
  intercepts both `read` and the fortified `__read_chk`, appends to a
  hit counter the scenarios assert on, and is unit-checked by
  `tests/ric_shim_probe.c` before every run -- a broken shim is a hard
  error, not a green suite.
- **Discovery**: an overlay over `/etc` owns `thingino.json`, so
  boards-as-shipped (pins from discovery, every flag at default) are a
  scenario class, not an accident. That class exists because a
  940nm-only board shipped through a fully-explicit-config suite.

One fresh ric per scenario keeps baselines and warn-once state clean.
`RIC_SUITE_STRICT=1` refuses to skip anything (inert shim included).
Diagnostics scenarios are mutation-checked -- see the dev guide.

### 2.5 Network Fallback Suite

`tests/test-net-fallback.sh` proves the IPv6-first policy both ways:
one pass native (every listening daemon must come up dual-stack and
answer on `::1`), one pass under `tests/net_noipv6_shim.c`, which
refuses `socket(AF_INET6)` with `EAFNOSUPPORT` the way a kernel built
without IPv6 does (every daemon must fall back to IPv4, log it, and
keep serving -- including rsr's SRT rebind after a failed v6 bind).

### 2.6 Pre-Auth Parser Fuzzing

`tests/test-fuzz.sh` runs libFuzzer over the three parsers that read
untrusted input before any authentication: rwd's SDP offer parser (a
WHIP body), its STUN parser (raw UDP from anyone who can reach the ICE
port), and Digest/Basic header parsing. Each target is time-boxed so
the stage fits in `test-all`; `--seconds` and `--corpus` turn it into a
real hunt. It skips cleanly without clang, and CI installs clang so
that skip cannot happen there silently.

The harnesses had rotted while nothing ran them: `fuzz_stun` no longer
compiled, because its shim hand-copies `struct rwd_client` and never
learned about the ICE verified-address ring. The helper it needs is
now extracted from `rwd_main.c` at build time rather than copied again.

### 2.7 RAC Beep Suite

`tests/test-rac.sh` exercises `rac beep` with no rad and no hardware:
argument validation, then a real 1 kHz / 1 s beep published to the
speaker ring, captured with `ringdump -d` and verified sample-exact --
count, peak amplitude, frequency via zero crossings, DC offset, and
both raised-cosine ramps. Writing it is what exposed the ring
newest-frame off-by-one: the final 20 ms chunk of every finite stream
was unreadable until a successor frame existed (`test_ring.c` even
carried "publish a dummy to make the previous frame readable"
helpers). The payload checks would have passed on a lossy capture
padded with silence; sample-exact totals are what made the missing
tail visible.

---

---

## 3. On-Device Validation

On-device validation runs on real cameras. Primary target is T31
(XBurst 1, most deployed). Secondary target is T41 (XBurst 2, new SDK
generation).

**Where the tests live.** Anything that touches a camera lives in the
`raptor-test` repo and is orchestrated by `run-battery`; `raptor/tests`
holds only what needs source, sanitizers or namespaces (unit tests,
the mock-HAL integration suite, the ric/rac/net-fallback/leak
harnesses). The split is not cosmetic: three device suites once lived
in `raptor/tests`, duplicated the battery's coverage, ran against an
unpinned ffmpeg, and were invoked by nothing -- so they judged cameras
against stale hardcoded expectations whenever someone remembered them.
A device test that is not a battery pass will not be run.

### 3.1 Validation Matrix

| Stage | Daemons | Validates |
|-------|---------|-----------|
| Video pipeline | RVD | HAL init, sensor detect, SHM ring creation, NAL structure, monotonic timestamps |
| RTSP streaming | RVD + RSD | SDP generation, stream playback, multi-client, ring reconnect after RVD restart |
| Audio pipeline | RVD + RAD + RSD | Audio ring, A/V sync (<100ms lip sync), codec delivery (PCMU/AAC/Opus) |
| OSD overlay | RVD + ROD | Double-buffer lifecycle, text rendering, format update, hot restart tolerance |
| Recording | RVD + RMR | fMP4 output, segment rotation, SD card removal resilience |
| Motion clips | RVD + RMR | Pre-buffer keyframe-aligned replay, clip duration, mode switching |
| Full stack | All daemons | 1-hour soak, memory/FD stability, day/night transition, client connect/disconnect cycles |

### 3.2 Acceptance Criteria

| Metric | Target |
|--------|--------|
| RVD CPU at 1080p@25fps | <15% |
| RVD RSS (excluding SHM mmap) | <1MB |
| RTSP simultaneous clients | `max_clients` config value |
| A/V sync delta | <100ms |
| FATAL/segfault/panic in logs | 0 over 1-hour soak |
| VmRSS drift over 1 hour | <100KB |
| FD count drift over 1 hour | 0 |

### 3.3 raptor-test: Live Stream Conformance Suite

`raptor-test` (its own repo, sibling to raptor) is the automated
on-device validation harness. It runs from a build host against any
live RTSP endpoint and exercises what unit and ASAN tests structurally
cannot: real client sessions over TCP and UDP, several client
implementations, concurrent viewers with join/leave churn, wire-level
RTP inspection, decoded-content validation (measured fps, resolution,
audio sample rate), GOP periodicity (and exact configured length via
`--gop`), RFC 2326/4566/3550/3640 conformance including PAUSE and
malformed-request robustness, ONVIF backchannel (client-to-camera
audio with device-side speaker-ring evidence), MJPEG over RTSP
(`/jpeg`), WebRTC through a real WHIP client including talk-back,
audio-only sources (auto-detected from the SDP), a five-codec audio
sweep (pcmu/pcma/l16/aac/opus, each asserted on the decoded codec and
sample rate), an encoder RC-mode sweep driven by the platform's own
capability report (the readback must equal what was asked -- a silent
SDK fallback is the bug that leg exists for), continuous-recording
segment alignment against the wall clock, NVR interop
through go2rtc and mediamtx restreams (both spawned locally from
pinned single binaries with `--go2rtc local` / `--mediamtx local`,
so the battery carries its own consumers; mediamtx's gortsplib
client is strict about SDP/RTP-Info/RTCP and exercises different
surface than go2rtc's), dockerized Frigate recording,
recorded-clip verification,
fault injection (abruptly killed clients), and device health over
SSH: CPU, connection baseline, daemon-log error sweep, and
framesource drop counters. A `--soak <minutes>` endurance mode
replaces the battery with repeated bounded captures plus RSS/FD
trending.

```
raptor-test rtsp://CAM:554/ch0 --sub rtsp://CAM:554/ch1 \
    --user thingino --pass thingino --ssh root@CAM --gop 60 \
    --rhd https://CAM:8443 --srt srt://CAM:9000 \
    --webrtc "https://CAM:8554/whip?stream=1" \
    --go2rtc local --mediamtx local --frigate \
    --record-check /path/to/recordings
```

Run it after any change that touches streaming, timestamps, codecs,
or session handling. Checks self-skip when a feature is absent, so it
also works against non-raptor servers for comparison. See the
raptor-test README for the full check table and design notes; its
checks encode every ad-hoc miss made during bring-up (case-sensitive
grep, orphaned client processes, truncated evidence pipelines).

---

## 4. x86 Stub Testing

The stub HAL generates synthetic frames on x86 without hardware. This
validates all IPC, protocol, and daemon lifecycle logic natively.

### 4.1 IPC Validation Matrix

| Area | Validates | Key invariant |
|------|-----------|---------------|
| Multi-consumer ring | Concurrent readers on same SHM ring | No data corruption, consistent sequence numbers |
| Ring overflow | Slow consumer falls behind | `RSS_EOVERFLOW` returned, recovery via next keyframe |
| OSD double-buffer | SHM lifecycle across daemon restarts | RVD continues if ROD crashes; ROD reconnects on restart |
| Control socket | Command dispatch and error handling | Invalid commands rejected, timeouts handled |
| RTSP synthetic | Full compy RTP/RTSP stack | SDP generation, RTP packetization, client lifecycle |
| Crash recovery | Producer killed, consumers detect stale ring | Ring reconnect after RVD restart, stream resumes |

---

## 5. SoC Compatibility

### 5.1 Capability Matrix

Key hardware feature differences that affect test coverage. Full matrix
in [12-hal-caps.md](12-hal-caps.md).

| Feature | T20 | T21 | T23 | T30 | T31 | T32 | T40 | T41 |
|---------|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| H.265 | - | - | - | Y | Y | Y | Y | Y |
| Buffer sharing | - | - | - | - | Y | - | Y | Y |
| Capped VBR | - | - | - | - | Y | Y | Y | Y |
| Dynamic bitrate | - | - | - | - | Y | Y | Y | Y |
| Dual sensor | - | - | _Sec | - | - | IMPVI | IMPVI | IMPVI |
| ISP OSD | - | - | Y | - | - | Y | Y | Y |
| HW rotation | - | - | - | - | Y | - | - | - |
| IVDC | - | - | Y | - | - | Y | Y | Y |
| XBurst 2 | - | - | - | - | - | - | Y | Y |

### 5.2 SoC-Specific Test Considerations

**T20/T21**: Graceful degradation tests. H.265 config falls back to
H.264 with a warning. `set-bitrate` returns an error (unsupported).
CAPPED_VBR silently maps to VBR. Old SDK enc_get_frame data layout
(direct virAddr, no ring buffer wrap) is exercised.

**T32**: Hybrid SDK path. Old-style type names with new-style internal
structs. `SetDefaultParam` takes extra `uBufSize`. Extended encoder
features (ROI, GDR, QP groups, VUI).

**T40/T41**: Dual-core XBurst 2. IMPVI_NUM parameter on all ISP tuning
calls. Multi-sensor support via independent MIPI ports. Hardware I2D
rotation.

---

## 6. Validation Tools

### 6.1 ffprobe — Stream Analysis

```sh
ffprobe -show_streams -of json -rtsp_transport tcp rtsp://camera:554/main
```

Verifies codec, resolution, frame rate, and GOP structure. Packet-level
analysis (`-show_packets`) detects timestamp inversions and IDR interval.

### 6.2 Valgrind — Memory and Thread Safety

Leak detection (`--leak-check=full`) and thread-safety analysis
(Helgrind) on x86 stub builds. Complements ASAN for detecting use-after-free
and race conditions that ASAN misses.

### 6.3 ringdump — SHM Ring Inspector

```sh
ringdump main              # Show ring header
ringdump main -f           # Follow frames (live)
ringdump main -f -n 100   # Follow 100 frames then exit
ringdump main -l           # Latency mode (capture-to-read pipeline delay)
ringdump main -d | ffprobe -i -   # Dump payload for analysis
```

Header output:
```
Ring: /dev/shm/rss_ring_main
Magic: 0x52535352 (valid)
Version: 1
Slots: 16
Data size: 3145728
Write seq: 12847
Stream: 0 (main)
Codec: H264
Resolution: 1920x1080
FPS: 25/1
```

Latency mode (`-l`) measures per-frame pipeline delay from IMP capture
timestamp to ring readout. Reports min/avg/max in milliseconds.

### 6.4 On-Target Quick Checks

```sh
for d in rvd rad rod rsd rmr ric rhd rmd rwd rwc; do
    pid=$(pidof $d 2>/dev/null)
    if [ -n "$pid" ]; then
        rss=$(awk '/VmRSS/{print $2}' /proc/$pid/status)
        fds=$(ls /proc/$pid/fd 2>/dev/null | wc -l)
        echo "$d: pid=$pid rss=${rss}kB fds=$fds"
    else
        echo "$d: not running"
    fi
done
```

---

## 7. Performance Baselines

Measured during initial development on T31 @ 1.5GHz XBurst with
1080p@25fps H264 VBR 2Mbps. Re-measure after significant changes.

### 7.1 CPU Usage

| Daemon | Idle (no clients) | 1 RTSP Client | 2 RTSP Clients | Recording |
|--------|------------------:|---------------:|----------------:|----------:|
| RVD | ~8% | ~8% | ~8% | ~8% |
| RAD | ~2% | ~2% | ~2% | ~2% |
| ROD | ~1% | ~1% | ~1% | ~1% |
| RSD | ~0% | ~5% | ~8% | N/A |
| RMR | N/A | N/A | N/A | ~3% |
| RIC | <1% | <1% | <1% | <1% |
| **Total** | **~12%** | **~17%** | **~20%** | **~15%** |

RVD CPU is constant regardless of consumers — consumers read from SHM
without producer involvement.

### 7.2 End-to-End Latency

| Stage | Latency (ms) | Notes |
|-------|-------------:|-------|
| Sensor exposure + readout | ~33 | Fixed by frame rate (1/30fps) |
| ISP processing | ~5 | Hardware pipeline |
| Encoder | ~10 | Hardware H264 encoder |
| RVD: enc_get_frame + ring_publish | <1 | Memory copy only |
| RSD: ring_read + RTP packetize | <1 | Zero-copy + sendto() |
| **Total (glass-to-wire)** | **~50** | Excluding network transit |

### 7.3 Memory Budget

T31 full stack (RVD + RAD + ROD + RSD + RMR + RIC), 64MB system:

```
Kernel + drivers:     ~12MB
RVD + SHM rings:       ~4MB
RAD + SHM ring:       ~0.5MB
ROD + OSD SHM:        ~0.8MB
RSD:                  ~0.5MB
RMR:                  ~0.4MB
RIC:                  ~0.1MB
Encoder DMA (rmem):   ~16MB  (reserved, not in /proc/meminfo MemTotal)
────────────────────────────
Used:                 ~18MB
Available:            ~46MB  (out of 64MB visible to Linux)
Headroom:             ~28MB  (for filesystem cache, tmpfs, user processes)
```

### 7.4 Maximum Throughput

| SoC | Max Main Stream | Max Sub Stream | Max Audio |
|-----|----------------|----------------|-----------|
| T20 | 1280x720@30fps H264 | 640x360@15fps H264 | 16kHz G711a |
| T31 | 1920x1080@30fps H264 | 640x360@25fps H264 | 16kHz AAC |
| T31 | 1920x1080@25fps H265 | 640x360@25fps H264 | 16kHz AAC |
| T40 | 2560x1440@25fps H265 | 640x360@25fps H264 | 48kHz AAC |
| T40 | 2x 1920x1080@25fps H264 (dual sensor) | 2x 640x360@15fps | 16kHz |

### 7.5 Startup Time

Time from `S31raptor start` to first RTSP-playable frame:

| Stage | Duration (ms) | Notes |
|-------|-------------:|-------|
| RVD process spawn | ~50 | fork + exec |
| HAL init (sensor detect) | ~800 | I2C probe, ISP init |
| Pipeline create + bind | ~200 | FS + OSD + Enc create |
| First encoded frame | ~300 | Sensor stream start + first IDR |
| RSD process spawn + ring open | ~100 | Parallel with above |
| **Total to first RTSP frame** | **~1500** | |

### 7.6 T20 FPS Impact

JPEG channels on T20 share encoder bandwidth with video channels,
reducing main stream FPS. OSD has zero impact.

| OSD | JPEG Channels | Measured FPS |
|:---:|:-------------:|:------------:|
| off | 0 | 30 |
| on  | 0 | 30 |
| off | 1 (jpeg0 only) | ~25 |
| on  | 1 (jpeg0 only) | ~25 |
| off | 2 (jpeg0 + jpeg1) | ~23 |
| on  | 2 (jpeg0 + jpeg1) | ~23 |

T31 runs at full 30fps with both JPEG channels and OSD enabled. The T31
SDK does not have the shared-resource contention seen on T20.

---

## 8. Continuous Integration

Three GitHub Actions workflows in `.github/workflows/`:

| Workflow | Trigger | What it does |
|----------|---------|-------------|
| `build.yml` | Every commit | Cross-compile uclibc static build for all 8 SoCs |
| `build-musl.yml` | Every commit | Cross-compile musl build |
| `tests.yml` | Every commit | Host-side ASAN unit tests (`make test` per repo) |

| Stage | Runner | Validates |
|-------|--------|-----------|
| Unit tests | x86 | 340 tests with ASAN across raptor-common, raptor-ipc, raptor |
| Cross-build | x86 | Compile for all 8 SoCs (`make PLATFORM=$SOC CROSS_COMPILE=mipsel-linux-`) |
| Hardware smoke | (planned) | RTSP reachability, daemon startup, log cleanliness |

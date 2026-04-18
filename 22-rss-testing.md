# Raptor Streaming System -- Test Architecture

Testing spans four layers: unit tests on x86 with AddressSanitizer,
full-daemon ASAN integration with a mock HAL, on-device validation on
real cameras, and performance benchmarking. 340 unit tests across the
raptor ecosystem pass under ASAN with zero errors.

---

## 1. Unit Test Inventory

All unit tests use the [greatest.h](https://github.com/silentbicycle/greatest)
single-header C test framework. Every test binary compiles with
`-fsanitize=address` (AddressSanitizer) to catch memory errors at test
time. Raptor repos build and run tests with `make test` in each `tests/`
directory. Compy uses CMake.

### 1.1 raptor-common (62 tests, 5 suites)

| Suite | File | Tests | Coverage |
|-------|------|------:|----------|
| config | test_config.c | 20 | INI parser: load, get/set str/int/bool, sections, iteration, roundtrip, foreach, edge cases |
| util | test_util.c | 17 | strlcpy, trim, starts_with, secure_compare — truncation, empty, exact fit |
| json | test_json.c | 6 | JSON string/int extraction, missing keys, truncation |
| http | test_http.c | 10 | Base64 decode (padding, overflow), HTTP basic auth validation |
| file | test_file.c | 9 | Read/write file, atomic write, mkdir_p, timestamp formatting |

```
cd raptor-common/tests && make test
```

### 1.2 raptor-ipc (25 tests, 3 suites)

| Suite | File | Tests | Coverage |
|-------|------|------:|----------|
| ring | test_ring.c | 17 | SHM ring create/open, publish/read, sequential access, overflow detection and recovery, multi-consumer, data region wrap, keyframe seek, wait timeout, IOV publish, stream info, IDR request, demand count, dead reader reaping |
| osd | test_osd.c | 5 | OSD double-buffer lifecycle, publish/read, dirty flag, heartbeat |
| ctrl | test_ctrl.c | 3 | Control socket roundtrip (threaded echo), handler error, stall timeout |

```
cd raptor-ipc/tests && make test
```

### 1.3 raptor (97 tests, 6 suites)

| Suite | File | Tests | Coverage |
|-------|------|------:|----------|
| nal | test_nal.c | 12 | Annex B → AVCC conversion (H.264, H.265, 3-byte start codes), empty/overflow, single NAL, SPS/PPS extraction |
| prebuf | test_prebuf.c | 9 | Pre-buffer push/count, slot eviction, keyframe find by age, frame-at lookup, iterate with data verification, data region wrap integrity |
| mux | test_mux.c | 17 | fMP4 box construction (moov/moof/mdat/mfra), H.264/H.265/PCMU/AAC/Opus codec paths, multi-fragment, large timestamps (64-bit tfdt), CTS offset, empty/double flush, A/V duration verification |
| sdp | test_sdp.c | 17 | SDP offer parsing, answer generation, ICE candidate extraction, codec negotiation |
| codec | test_codec.c | 28 | Codec detection, parameter set parsing, frame type identification |
| ring | test_ring.c | 14 | Ring buffer create/open, publish/read, overflow, IDR request, demand signaling |

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

---

## 3. On-Device Validation

On-device validation runs on real cameras. Primary target is T31
(XBurst 1, most deployed). Secondary target is T41 (XBurst 2, new SDK
generation).

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

### 3.3 Automated Device Test Suite (Planned)

A shell script (`raptor/tests/device/device-test.sh`) will SSH from a
build host into T31/T41 devices and verify: daemon health (pidof),
RTSP reachability (ffprobe), HTTP snapshot (curl), control socket
(raptorctl), log cleanliness, and resource usage. Not yet implemented.

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

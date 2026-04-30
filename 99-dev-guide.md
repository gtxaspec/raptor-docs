# Raptor Streaming System -- Development Guide

Quick-start reference for contributing to RSS. This document is
optimized for AI-assisted development but applies equally to human
contributors. For deep dives, follow the cross-references to other
docs in this repo.

---

## 1. What Raptor Is

RSS is an embedded streaming platform for Ingenic MIPS IP cameras
(T20, T21, T23, T30, T31, T32, T33, T40, T41, and A1). It replaces a
monolithic streamer with isolated single-purpose daemons communicating
via shared memory rings and Unix control sockets. On SoCs without an
ISP (A1), RFS acts as the video source instead of RVD.

Raptor is production quality software meant to be deployed to production cameras.
Code quality, memory safety, and long-term reliability are non-negotiable.
Every contribution must follow C best standards and practices -- write
code you would trust to run unattended for years on hardware you
cannot physically access. Do it right the first time.

**Key constraints:**
- Target devices have 32-256 MB RAM, no swap.
  Cache: T20/T21/T30 have 16 KB I + 16 KB D, no L2.
  T23 has 16 KB I + 16 KB D + 64 KB L2.
  T31/T32/T33/T41 have 32 KB I + 32 KB D + 128 KB L2.
  T40 has 32 KB I + 32 KB D + 1 MB L2 (shared with AI engine).
  A1 has 32 KB I + 32 KB D + 1 MB L2.
- All code is C11, cross-compiled for mipsel with uclibc or musl
- Binary size and memory footprint matter -- every kilobyte counts
- No dynamic allocation in hot paths (frame delivery, RTP packetization)
- Daemons run 24/7 for months -- leaks, even slow ones, are fatal

**Architecture reference:** `20-rss-architecture.md`
**Build system:** `21-rss-build.md`
**Test infrastructure:** `22-rss-testing.md`

---

## 2. Repository Layout

Five git repos, expected as siblings:

```
raptor/              Main repo -- all daemons and tools
raptor-hal/          Hardware abstraction layer (Ingenic SDK wrapper)
raptor-ipc/          IPC primitives (SHM ring, OSD double-buffer, control socket)
raptor-common/       Shared utilities (config, logging, JSON, net, TLS)
compy/               RTSP/RTP/SRTP/WebRTC library
raptor-docs/         This documentation
```

Each daemon lives in its own subdirectory under `raptor/`:

```
rvd/   -- Video pipeline (HAL owner, encoder, framesource, IVS, OSD)
rsd/   -- RTSP server (ring consumer, compy-based)
rsd-555/ -- RTSP server (ring consumer, live555-based, static link)
rad/   -- Audio capture and encoding
rhd/   -- HTTP server (snapshots, MJPEG, audio streaming)
rod/   -- OSD rendering (libschrift text, detection boxes)
ric/   -- IR-cut and day/night control
rmr/   -- Motion-triggered recording (fMP4, H.264/H.265/MJPEG)
rmd/   -- Motion detection state machine
rwd/   -- WebRTC server (WHIP signaling, DTLS-SRTP)
rwc/   -- USB webcam gadget (UVC+UAC)
rfs/   -- File source (MP4/Annex B playback into rings)
rsp/   -- RTMP/RTMPS stream push (YouTube, Twitch)
raptorctl/  -- CLI control tool
ringdump/   -- Ring buffer debug/dump tool
rlatency/   -- End-to-end RTSP latency measurement tool
rac/        -- Audio playback and recording tool
```

Only RVD and RAD link against the HAL. All other daemons are pure
ring consumers or control-socket clients. RSD and RSD-555 are
alternative RTSP backends reading the same rings -- RSD uses compy
(C, custom RTP), RSD-555 uses live555 (C++, statically linked).

---

## 3. Code Style

Raptor uses Linux kernel style with tabs. A `.clang-format` file
enforces this -- **always run `clang-format -i` before committing**.

### Rules

- **Tabs for indentation, 8-wide.** No spaces for indentation.
- **100-column line limit.** Break long lines at logical boundaries.
- **Linux brace style.** Opening brace on same line as control
  statement; functions get the brace on the next line.
- **Pointer alignment right:** `char *buf`, not `char* buf`.
- **No single-line blocks.** Always use braces even for one-line
  `if`/`for`/`while` bodies, or put the body on the next line
  without braces only when it fits the existing file's pattern.
- **Sort includes by group:** system headers first, then library
  headers, then project headers. Do not auto-sort (disabled in
  clang-format).

### Naming

- Functions: `snake_case`. Prefix with module: `rvd_stream_stop()`,
  `rss_ring_read()`, `hal_enc_start()`.
- Types: `snake_case_t` for typedefs: `rvd_state_t`, `rss_ring_t`.
- Macros/constants: `UPPER_SNAKE`: `RVD_MAX_STREAMS`, `RSS_OK`.
- Local variables: short, descriptive: `chn`, `ret`, `idx`, `s`.
- No Hungarian notation. No `m_` or `g_` prefixes.

### Comments

- Default to **no comments**. Well-named functions and variables
  are self-documenting.
- Comment the **why**, never the **what**. If removing the comment
  wouldn't confuse a reader, don't write it.
- File-level block comments are fine for describing the module's
  role (see any daemon's main `.c` file).
- Never reference tickets, PRs, or session context in comments --
  those belong in the commit message.

---

## 4. C Standards and Safety

### Memory Safety

- **No unbounded operations.** Use `snprintf`, never `sprintf`.
  Use `rss_strlcpy`, never `strcpy`/`strcat`.
- **Check every allocation.** `malloc`/`calloc`/`cJSON_Create*`
  can return NULL. `cJSON_PrintUnformatted` can return NULL.
  Handle it or propagate the error.
- **Free exactly once, in the right order.** Every `malloc` has a
  matching `free` on all code paths (including error paths).
  Use goto-based cleanup for multi-resource functions.
- **No use-after-free.** When you hold a pointer into a cJSON
  object or detached node, ensure the parent outlives your use.
- **Stack buffers have bounded lifetimes.** Never return pointers
  to stack-allocated arrays.
- **No VLAs.** Use fixed-size stack arrays or `malloc`.

### Thread Safety

- Atomic operations for simple flags: `_Atomic bool`, `atomic_load`,
  `atomic_store`. Used extensively for `stream_active[]`, `ivs_active`,
  `pipeline_ready`.
- `pthread_mutex_t` for complex shared state (detection results,
  OSD regions, client lists).
- Control socket handlers run synchronously in the main thread via
  `epoll` -- no mutex needed for state accessed only from the
  ctrl handler.
- Each encoder channel runs in a dedicated thread. Never access
  another channel's thread-local state without synchronization.

### Error Handling

- Functions return `int` (0 = success, negative = error) or a
  pointer (NULL = failure).
- Use `RSS_OK` / `RSS_ERR` / `RSS_ERR_INVAL` / `RSS_ERR_NOTSUP`
  from `rss_common.h`.
- HAL calls go through the `RSS_HAL_CALL()` macro which handles
  NULL function pointers gracefully.
- IPC control handlers write errors via `rss_ctrl_resp_error()`
  and return the response length. Never leave `resp` unwritten.

### Defensive Patterns

- **Idempotent operations.** Stop an already-stopped stream?
  Return OK. Start an already-running stream? Return OK. Never
  crash on redundant operations.
- **Validate at system boundaries.** Check JSON input from control
  sockets. Check channel indices. Check config values. Trust
  internal state between validated boundaries.
- **Rollback on failure.** If a multi-step operation fails midway,
  restore the previous state. See `set-codec` and `set-resolution`
  in `rvd_ctrl.c` for the pattern.
- **Don't mask errors.** If a HAL call fails, log it and propagate.
  Don't silently continue with corrupt state.

### Protocol Compliance

Raptor implements real-world streaming protocols. Where an RFC or
standard defines the behavior, **follow it** -- don't invent ad-hoc
alternatives. Clients, NVRs, and browsers expect standards-compliant
implementations. A creative shortcut that works with one client will
break with the next.

**Applicable standards by component:**

| Component | Standards |
|-----------|-----------|
| compy (RTSP core) | RFC 2326 (RTSP/1.0), RFC 7826 (RTSP/2.0) |
| compy (RTP/RTCP) | RFC 3550 (RTP), RFC 4585 (RTCP FB), RFC 5104 (codec control) |
| compy (SRTP) | RFC 3711 (SRTP/SRTCP) |
| compy (SDP) | RFC 4566 (SDP), RFC 3264 (offer/answer) |
| compy (payloads) | RFC 6184 (H.264), RFC 7798 (H.265), RFC 3640 (AAC), RFC 7587 (Opus) |
| RWD (WebRTC) | RFC 8445 (ICE), RFC 8489 (STUN), RFC 5764 (DTLS-SRTP), WHIP (draft-ietf-wish-whip) |
| RMR (recording) | ISO 14496-12 (ISOBMFF/fMP4), MJPEG in MP4 (FourCC `jpeg`) |
| RSD / RSD-555 (RTSP) | RFC 2326 (RTSP/1.0), RFC 3550 (RTP §5.1), RFC 4566 (SDP) |

**Compliance items enforced across RSD, RSD-555, and compy:**

- **Random initial RTP seq and timestamp** (RFC 3550 §5.1): both
  compy and RSD use `/dev/urandom` for initial sequence numbers
  and RTP-Info `rtptime` values. Zero-based values cause client
  calibration failures (e.g., mpv "No video PTS").
- **SDP `o=` origin** (RFC 4566 §5.2): session ID from monotonic
  clock, server IP from `getsockname`. Not `0 0 ... 0.0.0.0`.
- **SDP `b=AS:`** (RFC 4566 §5.8): per-media bandwidth hints.
- **SDP direction** (RFC 4566 §6): streaming server sends, not
  receives. Do not use `a=recvonly` (incorrect for a camera).
- **RTP-Info URL** (RFC 2326 §12.33): strip credentials from the
  URL -- ffmpeg strips them internally and fails to match if
  they're present.

**Rules:**

- **Verify RFC compliance when touching protocol code.** Before
  modifying any code that implements RFC behavior, read the
  relevant sections of the RFC and confirm your changes conform.
  Don't assume the existing code is correct -- verify both.
- **Cite the RFC in commit messages** when implementing or fixing
  protocol behavior. Example: `compy: fix RTCP SR timestamp per
  RFC 3550 Section 6.4.1`.
- **Don't subset silently.** If you intentionally skip a MUST or
  SHOULD from an RFC, document it in the code with the specific
  section number and the reason (hardware limitation, scope, etc.).
- **Use RFC terminology.** MUST, SHOULD, MAY have precise meanings
  (RFC 2119). Match the RFC's field names and state machine names
  in code where practical -- it makes cross-referencing trivial.
- **Test against real clients.** VLC, ffplay, and browser WebRTC
  are the minimum. NVR compatibility (Blue Iris, Frigate) matters
  for RTSP. Don't assume your own raptorctl exercises the same
  code paths a third-party client would.

---

## 5. Architecture Patterns

### Adding a Control Command

Every daemon exposes a JSON-over-Unix-socket control interface. To
add a new command:

1. **Daemon side** -- add an `if (strcmp(cmd, "your-cmd") == 0)`
   block in the daemon's ctrl handler function. Parse input with
   `rss_json_get_int()`/`rss_json_get_str()`. Respond with
   `rss_ctrl_resp_ok()` or `rss_ctrl_resp_error()`.

2. **raptorctl side** -- add a help entry in `help_entries[]`
   (`raptorctl_help.c`) and a dispatch table entry in
   `raptorctl_dispatch.c`. Simple no-arg commands fall through
   to the generic pass-through -- no dispatch entry needed.

3. **JSON mode** -- commands added this way automatically work with
   `raptorctl -j` since it sends raw JSON directly.

Pattern reference: `stream-stop`/`stream-start` in `rvd_ctrl.c`,
`mute`/`unmute` in `rad_main.c`.

### Adding a Simple Encoder Parameter

Simple encoder params (single int/uint/bool value, channel-scoped)
use the table-driven system. To add one:

1. Add a row to `enc_params[]` in `rvd_ctrl.c` with the param name,
   type (`EP_INT`/`EP_U32`/`EP_BOOL`), and `offsetof` into
   `rss_hal_ops_t` for the setter and getter.

That's it -- no raptorctl changes needed. The param is immediately
available via `enc-set`, `enc-get`, and `enc-list`.

For struct-based params (multi-field args like ROI, super-frame),
add an explicit handler in `handle_encoder_advanced_cmd()` plus a
raptorctl dispatch entry.

### Adding a New Daemon

Consumer daemons follow a template:

1. Create `raptor/<name>/` with `Makefile` and source files.
2. Link `librss_ipc.a` + `librss_common.a` (not the HAL).
3. Call `rss_daemonize()` for PID file + signal handling.
4. Open ring buffers with `rss_ring_open()`, read with
   `rss_ring_read()`.
5. Listen on `/var/run/rss/<name>.sock` for control commands.
6. Add build target in the top-level `Makefile`.
7. Add the daemon name to the `daemons[]` array in `raptorctl.c`.

### IPC Primitives

- **SHM ring buffers** -- lock-free single-producer multi-consumer
  frame transport. Zero-copy in refmode (producer writes a pointer,
  consumers mmap the same physical memory).
- **OSD double-buffer** -- BGRA bitmap shared between ROD (writer)
  and RVD (reader). Dirty flag + heartbeat for crash detection.
- **Control sockets** -- length-prefixed JSON over Unix domain
  sockets. Synchronous request/response. Used for configuration,
  status queries, and runtime parameter changes.

Details: `20-rss-architecture.md` sections 3-5.

**Logging:** raptor-ipc has its own log macros (`RSS_IPC_ERROR`,
`RSS_IPC_WARN`, `RSS_IPC_INFO`, `RSS_IPC_DEBUG`) defined in
`rss_ipc.h`. Use these instead of `printf`/`fprintf` in IPC code.
`rss_daemon_init()` automatically wires them through the daemon's
`rss_log()` via a weak symbol -- no per-daemon setup needed.
Standalone tools and tests get stderr fallback.

### HAL Layer

The HAL wraps Ingenic's `libimp.so` SDK with a clean C API. It
abstracts cross-SoC differences (9 SoC families, 3 SDK generations).
A1 has no ISP -- it skips the HAL entirely and uses RFS as the
video source.

- **Never call IMP_* functions directly** from daemon code. Always
  go through the HAL ops vtable.
- HAL functions take `void *ctx` as first argument (opaque context).
- Capabilities are runtime-queryable via `ops->get_caps()`.
- Audio and video HAL are separate static libraries.

HAL API reference: `10-hal-api.md`
HAL internals: `11-hal-internals.md`
Platform capabilities: `12-hal-caps.md`

---

## 6. Build and Test

**Every change must build clean, pass all tests, and run clean under
AddressSanitizer and ThreadSanitizer before committing.** Sanitizer
warnings are treated as bugs -- do not suppress or ignore them.
Memory leaks, data races, and undefined behavior are not acceptable
in production code that runs 24/7 on devices you cannot physically
access.

### Standalone Build (no Buildroot)

```sh
make distclean
./build-standalone.sh t31 --local --static
```

`make distclean` removes stale build artifacts. The build script
downloads the toolchain and all dependencies, then builds everything,
no need to specify `CROSS_COMPILE` manually. Use `--local` to use
sibling repo checkouts instead of cloning. Output binaries go to
`build/`.

### Build Individual Targets

```sh
# Using the standalone deps
XB=.deps/toolchain/bin/mipsel-linux-
make PLATFORM=T31 CROSS_COMPILE=$XB raptorctl rvd rsd
```

Note: daemon binaries that link the HAL (rvd, rad) will fail at
link time without the Ingenic SDK libs. Compilation succeeding
(all `.c` files compile clean) is sufficient to verify your changes.

### ASAN Build (x86, mock HAL)

```sh
./build-asan.sh        # AddressSanitizer + UBSan
./build-asan.sh tsan   # ThreadSanitizer
```

Builds all 14 daemons + tools for x86 with a mock HAL (`tests/mock_hal.c`).
Output goes to `asan-out/`. RVD and RAD use the mock HAL; all other
daemons build natively (no HAL dependency).

### Running Tests

**Full suite (recommended):**

```sh
./tests/test-all.sh                   # quick pass (~2 min)
./tests/test-all.sh --soak 300        # with 5-min leak soak
./tests/test-all.sh --tsan            # ThreadSanitizer mode
./tests/test-all.sh --tsan --soak 300 # full TSAN + soak
```

Runs all four stages: build, unit tests, integration tests,
leak/race detection. Fails fast if any stage fails.

**Individual test suites:**

```sh
# Unit tests (187 tests across 14 suites, ASAN)
cd tests && make test

# Integration tests (46 tests — raptorctl, HTTP, RTSP, multi-client)
./tests/test-integration.sh

# Leak detection (lifecycle soak under LeakSanitizer)
./tests/test-leak.sh
./tests/test-leak.sh --duration 300        # 5-min soak
./tests/test-leak.sh --tsan                # data race detection
./tests/test-leak.sh --tsan --duration 300 # TSAN + soak

# Sibling repo tests
cd raptor-ipc/tests && make test      # IPC tests (25 tests)
cd raptor-common/tests && make test   # common tests (62 tests)
```

**Fuzz targets (requires clang):**

```sh
make -C fuzz                     # build all fuzzers
./fuzz/fuzz_stun corpus/stun/    # STUN parser (calls production rwd_ice_process)
./fuzz/fuzz_sdp corpus/sdp/      # SDP parser
./fuzz/fuzz_http_auth corpus/auth/  # HTTP auth
```

### CI

GitHub Actions (`tests.yml`, manual dispatch):

- **asan job**: full suite + 5-min soak under AddressSanitizer
- **tsan job**: full suite + 5-min soak under ThreadSanitizer

Both jobs build from scratch, run 187 unit tests + 46 integration
tests + lifecycle soak (concurrent clients, ring reconnect, clean
shutdown). Logs are uploaded as artifacts on failure.

### On-Device Testing via NFS

Test devices mount the build host's home directory at `/mnt/nfs`.
No `scp` needed -- edit, build, run directly:

```sh
# On device:
cd /mnt/nfs/projects/thingino/raptor
./run.sh    # launches configured daemons
```

### clang-format

**Always format before committing:**

```sh
clang-format -i path/to/changed/files.c
```

The `.clang-format` in the repo root enforces the project style.

### Pre-Commit Checklist

Before every commit:

1. `clang-format -i` on all changed `.c` files
2. `./build-standalone.sh t31 --local --static` -- clean build, zero warnings
3. `./tests/test-all.sh` -- unit + integration + leak check pass
4. On-device smoke test if touching daemon logic (NFS mount, run binary, exercise via raptorctl)

If any step fails, fix before committing. Do not commit with known
sanitizer warnings or test failures.

---

## 7. Commit and Push Guidelines

- **Commit messages are concise and imperative.** Example:
  `rvd: add stream-stop and stream-start IPC commands`
- **Prefix with the component:** `rvd:`, `rad:`, `raptorctl:`,
  `hal:`, `rsd:`, `build:`, `tests:`, etc.
- **No signatures.** Just a clear commit message.
- **Don't push without explicit instruction.** Build and test
  locally first. Confirm with the maintainer before pushing.
- **Don't create releases** without explicit instruction.
- **One logical change per commit.** A new feature is one commit.
  A bug fix is one commit. Don't bundle unrelated changes.

---

## 8. Common Pitfalls

### Embedded-Specific

- **No `printf` debugging in production code.** Use `RSS_INFO`,
  `RSS_WARN`, `RSS_ERROR`, `RSS_DEBUG`, `RSS_TRACE` from
  `rss_common.h`. Log levels are filterable at runtime.
- **`-Os` globally.** Never change to `-O2`/`-O3` -- I-cache
  pressure on MIPS32 makes larger code slower, not faster. Use
  `__attribute__((optimize("O3")))` on specific hot functions
  if profiling proves it helps.
- **Musl/uclibc differences.** `free(NULL)` is safe. `dlopen`
  behavior differs. Page alignment must be 4KB (not 64KB default).
  The build system handles this via `-Wl,-z,max-page-size=0x1000`.
- **No filesystem writes in hot paths.** Flash storage is slow
  and has limited write endurance. Rings are in `/dev/shm` (tmpfs).
  Config saves are explicit, not automatic.

### SDK-Specific

- **Encoder channels share groups.** Creating/destroying one
  channel can affect others in the same group. Always stop
  the paired JPEG channel before touching its parent video channel.
- **IVS requires FS streaming before start.** The framesource
  channel must be enabled and producing frames before calling
  `IVS_StartRecvPic`. The bind chain must include IVS before
  the framesource is enabled.
- **SDK calls are not thread-safe.** All IMP_* calls for a given
  channel must come from the same thread, or be serialized
  externally.
- **IVDC mode restrictions.** Direct-connect mode changes memory
  layout and disables certain IPU features. See `27-multi-sensor.md`.

### IPC-Specific

- **Ring readers must handle overflow.** If a consumer falls behind,
  the producer overwrites old data. Readers detect this via sequence
  gap and must re-seek to the next keyframe.
- **Control socket responses must always be written.** A handler
  that returns without writing to `resp` causes the client to hang
  waiting for a response.
- **`rss_ctrl_send_command` has a timeout.** Default 5 seconds.
  Don't perform blocking operations (network I/O, disk writes)
  inside a ctrl handler -- it blocks all other ctrl clients.

---

## 9. Cross-Reference Index

| Topic | Document |
|-------|----------|
| System architecture, daemon roles, IPC | `20-rss-architecture.md` |
| Build system, repo layout, Buildroot | `21-rss-build.md` |
| Test suites, ASAN, integration tests | `22-rss-testing.md` |
| Configuration reference (all options) | `23-rss-config.md` |
| Ring buffer consumer API and examples | `24-ring-consumer-guide.md` |
| HAL API (all ops, all SoCs) | `10-hal-api.md` |
| HAL internals and porting | `11-hal-internals.md` |
| SoC capabilities matrix | `12-hal-caps.md` |
| SDK system/encoder/framesource/ISP/audio/OSD | `01` through `08` |
| WebRTC design (RWD) | `25-rwd-webrtc-design.md` |
| Day/night IR-cut design (RIC) | `26-ric-daynight-design.md` |
| Multi-sensor support | `27-multi-sensor.md` |
| IVS/motion detection | `28-ivs-detection.md` |
| FPS troubleshooting | `30-fps-troubleshooting.md` |

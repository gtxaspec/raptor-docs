# Raptor Streaming System -- Build System

Makefile-based build with per-directory Makefiles, cross-compilation via
Buildroot toolchain, ASAN debug build for x86, and Buildroot package
integration.

---

## 1. Repository Layout

Raptor is split across five git repositories, expected as siblings:

```
raptor/                  # main repo: all daemons and tools
├── Makefile             # top-level build (orchestrates sub-makes)
├── build.sh             # convenience wrapper: ./build.sh t31 <br_output>
├── build-standalone.sh  # standalone build (downloads toolchain + all deps)
├── build-asan.sh        # x86 ASAN build (mock HAL, native gcc)
├── config/
│   └── raptor.conf      # default configuration file
├── run.sh               # NFS launch script for on-device testing
├── rvd/                 # video daemon (HAL, pipeline, encoder, IVS)
│   ├── Makefile
│   ├── rvd.h
│   ├── rvd_main.c
│   ├── rvd_pipeline.c
│   ├── rvd_frame_loop.c
│   ├── rvd_ctrl.c
│   ├── rvd_osd.c
│   └── rvd_ivs.c
├── rsd/                 # RTSP streaming daemon (compy-based)
│   ├── Makefile
│   ├── rsd.h
│   ├── rsd_main.c
│   ├── rsd_server.c
│   ├── rsd_session.c
│   └── rsd_ring_reader.c
├── rad/                 # audio daemon (HAL, encode, ring publish)
│   ├── Makefile
│   └── rad_main.c
├── rhd/                 # HTTP daemon (snap.jpg, MJPEG, audio)
│   ├── Makefile
│   ├── rhd.h
│   ├── rhd_main.c
│   ├── rhd_http.c
│   ├── rhd_audio.c      # HTTP audio streaming
│   └── index.html
├── rod/                 # OSD daemon (libschrift rendering)
│   ├── Makefile
│   ├── rod.h
│   ├── rod_main.c
│   └── rod_render.c
├── ric/                 # IR-cut day/night control
│   ├── Makefile
│   ├── ric.h
│   ├── ric_main.c
│   └── ric_daynight.c
├── rmr/                 # recorder (fMP4, pre-buffer, clips)
│   ├── Makefile
│   ├── rmr.h
│   ├── rmr_main.c
│   ├── rmr_mux.c / rmr_mux.h
│   ├── rmr_nal.c / rmr_nal.h
│   ├── rmr_prebuf.c / rmr_prebuf.h
│   └── rmr_storage.c / rmr_storage.h
├── rmd/                 # motion detection daemon
│   ├── Makefile
│   ├── rmd.h
│   ├── rmd_main.c
│   └── rmd_actions.c
├── rwd/                 # WebRTC daemon (WHIP, DTLS-SRTP)
│   ├── Makefile
│   ├── rwd.h
│   ├── rwd_main.c
│   ├── rwd_dtls.c
│   ├── rwd_ice.c
│   ├── rwd_sdp.c
│   ├── rwd_signaling.c
│   ├── rwd_media.c
│   └── webrtc.html
├── rfs/                 # file source daemon (MP4/Annex B, no HAL)
│   ├── Makefile
│   ├── rfs_main.c
│   ├── rfs_annexb.c     # Annex B scanner, SPS parser, B-frame reorder
│   ├── rfs_annexb.h
│   ├── rfs_mp4.c        # MP4 demuxer (libmov wrapper)
│   └── rfs_mp4.h
├── raptorctl/           # CLI control tool
│   ├── Makefile
│   ├── raptorctl.h
│   ├── raptorctl.c
│   └── raptorctl_info.c
├── ringdump/            # SHM ring debug tool
│   ├── Makefile
│   └── ringdump.c
├── rac/                 # audio client (play/record)
│   ├── Makefile
│   ├── rac.h
│   ├── rac.c
│   ├── rac_record.c
│   └── rac_play.c
└── tests/
    ├── mock_hal.c       # x86 mock HAL for ASAN builds
    └── create_rings.c   # synthetic SHM rings for testing

raptor-hal/              # HAL abstraction layer (separate repo)
├── Makefile
├── include/raptor_hal.h
└── src/
    ├── hal_common.c     # factory, shared utils, multi-sensor dispatch
    ├── hal_encoder.c    # encoder create/destroy/poll
    ├── hal_framesource.c
    ├── hal_isp.c        # ISP tuning (per-SoC differences)
    ├── hal_audio.c
    ├── hal_osd.c
    ├── hal_gpio.c
    ├── hal_ivs.c        # IVS motion detection wrapper
    ├── hal_dmic.c
    ├── hal_caps.c       # per-SoC capability tables
    └── hal_memory.c

raptor-ipc/              # IPC library (separate repo)
├── Makefile
├── include/rss_ipc.h
└── src/
    ├── rss_ring.c       # SHM ring buffer (futex-based)
    ├── rss_osd_shm.c    # OSD double-buffer SHM
    └── rss_ctrl.c       # Unix domain control sockets

raptor-common/           # common utilities (separate repo)
├── Makefile
├── include/
│   ├── rss_common.h
│   ├── rss_http.h       # HTTP basic auth, base64
│   ├── rss_net.h        # network utility helpers
│   ├── rss_tls.h        # shared TLS server for HTTPS
│   └── cJSON.h          # bundled cJSON parser
└── src/
    ├── rss_log.c        # logging (syslog + stderr)
    ├── rss_config.c     # INI config parser
    ├── rss_daemon.c     # daemonize, PID files, signals, daemon init
    ├── rss_util.c       # timestamps, string utils, file I/O, JSON helpers
    ├── rss_ctrl_cmds.c  # common ctrl commands (config-get, config-save)
    ├── rss_http.c       # HTTP basic auth, base64 decode
    ├── rss_tls.c        # shared TLS server initialization
    └── cJSON.c          # bundled cJSON library (JSON response building)

compy/                   # RTSP library (separate repo, CMake)
├── CMakeLists.txt
├── include/compy.h
└── src/                 # RTSP parser, RTP transport, SDP, NAL packetizer
```

---

## Code Style

Each repository has a `.clang-format` file. The style split is intentional:

| Repository | Indent | Tabs/Spaces | Rationale |
|-----------|--------|-------------|-----------|
| raptor (daemons) | 8 | Tabs | Linux kernel style |
| raptor-ipc | 4 | Spaces | Library style |
| raptor-common | 4 | Spaces | Library style |
| raptor-hal | 4 | Spaces | Library style |

Run `clang-format -i <file>` before committing. The top-level raptor
Makefile does not enforce formatting automatically — it is the
contributor's responsibility.

---

## 2. Build System

### 2.1 Top-Level Makefile

Plain GNU Make. No CMake, no autotools. Each daemon has its own
sub-Makefile that compiles `.c` → `.o` → binary.

```sh
# Full cross-build
make PLATFORM=T31 CROSS_COMPILE=mipsel-linux- SYSROOT=/path/to/sysroot

# Single daemon
make PLATFORM=T31 CROSS_COMPILE=mipsel-linux- SYSROOT=/path/to/sysroot rvd

# Feature flags
make PLATFORM=T31 CROSS_COMPILE=mipsel-linux- AAC=1 OPUS=1 TLS=1

# Clean
make distclean
```

**Required variables:**

| Variable | Description |
|----------|-------------|
| `PLATFORM` | Target SoC: T20, T21, T23, T30, T31, T32, T40, T41, A1 |
| `CROSS_COMPILE` | Compiler prefix (e.g., `mipsel-linux-`) |
| `SYSROOT` | Buildroot sysroot path (for shared lib linking) |

**Feature flags:**

| Flag | Effect |
|------|--------|
| `AAC=1` | Enable AAC encode/decode (links libfaac + libhelix-aac) |
| `OPUS=1` | Enable Opus codec (links libopus) |
| `MP3=1` | Enable MP3 decode (links libhelix-mp3) |
| `TLS=1` | Enable RTSPS (adds `-DCOMPY_HAS_TLS`, links mbedTLS) |
| `AUDIO_EFFECTS=1` | Enable audio processing (NS, HPF, AGC) |
| `DEBUG=1` | Build with `-O0 -g` instead of `-Os` |
| `V=1` | Verbose build output |

**Daemon dependencies:**

| Daemon | Libraries |
|--------|-----------|
| RVD | raptor-hal + raptor-ipc + raptor-common + libimp + libalog + libsysutils |
| RAD | raptor-hal + raptor-ipc + raptor-common + libimp (+ libfaac, libopus) |
| RSD | raptor-ipc + raptor-common + compy (+ mbedTLS if TLS=1) |
| RHD | raptor-ipc + raptor-common |
| ROD | raptor-ipc + raptor-common + libschrift |
| RIC | raptor-ipc + raptor-common |
| RMR | raptor-ipc + raptor-common |
| RMD | raptor-ipc + raptor-common |
| RWD | raptor-ipc + raptor-common + mbedTLS |
| RWC | raptor-ipc + raptor-common |

### 2.2 build.sh Convenience Wrapper

```sh
# Usage
./build.sh <platform> <buildroot_output_dir> [targets...]

# Examples
./build.sh t31 /path/to/output                    # full clean build
./build.sh t31 /path/to/output rvd rsd             # build specific daemons
./build.sh t31 /path/to/output clean               # clean

# What it does:
# 1. Auto-detects sysroot and toolchain from buildroot output
# 2. Auto-detects TLS support (mbedTLS in sysroot)
# 3. Sets PLATFORM, CROSS_COMPILE, SYSROOT, AAC=1, OPUS=1
# 4. Runs make with all flags
```

### 2.3 Supported Platforms

| Platform | SoC | SDK Generation | Notes |
|----------|-----|---------------|-------|
| `t20` | T20 | Old | XBurst1 |
| `t21` | T21 | Old | XBurst1 |
| `t23` | T23 | Old (extended) | XBurst1, IVDC capable |
| `t30` | T30 | Old | XBurst1 |
| `t31` | T31 | New | XBurst1 (primary target) |
| `t32` | T32 | New | XBurst1, dual-sensor |
| `t40` | T40 | IMPVI | XBurst2 |
| `t41` | T41 | IMPVI | XBurst2 |
| `a1` | A1 | IMPVI | XBurst2, 1MB L2, no ISP — uses RFS instead of RVD |

---

## 3. Buildroot Package

Raptor is split into five Buildroot packages matching the five repos:

- `thingino-raptor` — daemons and tools
- `thingino-raptor-hal` — HAL library
- `thingino-raptor-ipc` — IPC library
- `thingino-raptor-common` — common library
- `compy` — RTSP library

### 3.1 thingino-raptor.mk

```makefile
THINGINO_RAPTOR_SITE_METHOD = git
THINGINO_RAPTOR_SITE = https://github.com/gtxaspec/raptor
THINGINO_RAPTOR_SITE_BRANCH = main

THINGINO_RAPTOR_DEPENDENCIES += ingenic-lib compy libschrift
THINGINO_RAPTOR_DEPENDENCIES += thingino-raptor-hal thingino-raptor-ipc thingino-raptor-common

# Feature flags passed through to make
ifeq ($(BR2_PACKAGE_THINGINO_RAPTOR_AAC),y)
THINGINO_RAPTOR_MAKE_OPTS += AAC=1
THINGINO_RAPTOR_DEPENDENCIES += faac libhelix-aac
endif

ifeq ($(BR2_PACKAGE_THINGINO_RAPTOR_OPUS),y)
THINGINO_RAPTOR_MAKE_OPTS += OPUS=1
THINGINO_RAPTOR_DEPENDENCIES += opus
endif

ifeq ($(BR2_PACKAGE_THINGINO_RAPTOR_TLS),y)
THINGINO_RAPTOR_MAKE_OPTS += TLS=1
THINGINO_RAPTOR_DEPENDENCIES += mbedtls
endif

define THINGINO_RAPTOR_BUILD_CMDS
    $(MAKE) PLATFORM=$(PLATFORM) CROSS_COMPILE=$(TARGET_CROSS) \
        SYSROOT=$(STAGING_DIR) $(THINGINO_RAPTOR_MAKE_OPTS) \
        -C $(@D) rvd rsd rad rhd rod ric rmr rmd rwd rwc raptorctl ringdump rac
endef

define THINGINO_RAPTOR_INSTALL_TARGET_CMDS
    $(foreach d,rvd rsd rad rhd rod ric rmr rmd rwd rwc,\
        $(INSTALL) -D -m 0755 $(@D)/$(d)/$(d) $(TARGET_DIR)/usr/bin/$(d)$(sep))
    $(foreach t,raptorctl ringdump rac,\
        if [ -f $(@D)/$(t)/$(t) ]; then \
            $(INSTALL) -D -m 0755 $(@D)/$(t)/$(t) $(TARGET_DIR)/usr/bin/$(t); \
        fi$(sep))
    $(INSTALL) -D -m 0644 $(@D)/config/raptor.conf $(TARGET_DIR)/etc/raptor.conf
    $(INSTALL) -D -m 0755 $(THINGINO_RAPTOR_PKGDIR)/files/S31raptor \
        $(TARGET_DIR)/etc/init.d/S31raptor
endef
```

---

## 4. ASAN Debug Build (x86)

`build-asan.sh` builds all daemons natively on x86 with AddressSanitizer
and UndefinedBehaviorSanitizer. RVD and RAD use a mock HAL
(`tests/mock_hal.c`) that stubs all IMP SDK calls. All other daemons
build without HAL.

### 4.1 Building

```sh
./build-asan.sh          # build all to asan-out/
./build-asan.sh clean    # clean
```

### 4.2 Testing

```sh
cd asan-out/
./create_rings &         # create synthetic SHM rings
./rvd -c ../config/raptor.conf -f -d &
./rsd -c ../config/raptor.conf -f -d &
./rhd -c ../config/raptor.conf -f -d &
# Exercise with ffprobe, curl, raptorctl
# Ctrl-C — ASan prints leak report on exit
```

### 4.3 What Works on x86

- Full daemon lifecycle (start, run, stop)
- SHM ring producer/consumer
- OSD double-buffer protocol
- Control socket protocol (raptorctl)
- RTSP server (serves synthetic streams)
- Recording (writes fMP4 with synthetic frames)
- Configuration parsing and logging

### 4.4 What Does Not Work on x86

- Actual video encoding (stub frames are synthetic NAL units)
- ISP tuning effects (values stored but not applied)
- GPIO/IR-cut control
- Real-time timing characteristics

---

## 5. Dependency List

| Dependency | Used By | Purpose | Required? |
|-----------|---------|---------|-----------|
| ingenic-lib | HAL (RVD, RAD) | libimp.so, libalog.so, libsysutils.so | Yes (target) |
| libschrift | ROD | TrueType font rendering for OSD text | Yes for ROD |
| compy | RSD | RTSP protocol, RTP transport, SDP, auth | Yes for RSD |
| mbedTLS | RSD (optional) | TLS for RTSPS | Only if TLS=1 |
| libfaac | RAD (optional) | AAC-LC encoding | Only if AAC=1 |
| libhelix-aac | RAD (optional) | AAC decoding (backchannel) | Only if AAC=1 |
| libopus | RAD (optional) | Opus encoding | Only if OPUS=1 |
| libhelix-mp3 | RAD (optional) | MP3 decoding (backchannel) | Only if MP3=1 |
| ingenic-uclibc | all daemons | uclibc shim for vendor SDK compat | Yes (uclibc toolchain) |
| ingenic-musl | all daemons | musl shim for vendor SDK compat | Yes (musl toolchain) |
| pthread, rt | all daemons | POSIX threads, SHM, clock | Yes (system) |

**Note:** cJSON is bundled in raptor-common (not an external dependency).
The config parser (`rss_config.c`) reads INI-format files. cJSON is used
for building all JSON control socket responses. The lightweight JSON
readers (`rss_json_get_str/int`) are still used for parsing incoming
control socket commands.

### 5.1 Minimal Build

For a streaming-only camera (no recording, no OSD, no audio):

```
Daemons: RVD + RSD
Dependencies: ingenic-lib, compy, pthread, rt
Binary sizes (stripped, T31): rvd ~142KB, rsd ~68KB, raptorctl ~34KB
```

### 5.2 Full Build

```
Daemons: RVD + RAD + ROD + RSD + RHD + RMR + RIC + RMD + RWD + RWC
Tools: raptorctl, ringdump, rac
Dependencies: all of the above
Approximate total (stripped, T31): ~520KB
```

---

## 6. Common Library (raptor-common)

All daemons share the raptor-common library which provides:

- **`rss_daemon_init()`** — standard daemon startup: arg parsing
  (`-c config -f foreground -d debug -h help`), logging, config load,
  daemonize with PID duplicate check, signal handlers. Eliminates ~30
  lines of boilerplate per daemon.
- **cJSON** — bundled cJSON library for building all JSON control socket
  responses (`cJSON_CreateObject`, `cJSON_AddStringToObject`,
  `cJSON_PrintUnformatted`).
- **`rss_json_get_str()` / `rss_json_get_int()`** — lightweight JSON key
  lookup for parsing incoming control socket commands. Key length
  validated, int range checked against INT_MIN/INT_MAX.
- **`rss_ctrl_resp_ok()` / `rss_ctrl_resp_error()` / `rss_ctrl_resp_json()`**
  — helpers for building control socket JSON responses via cJSON.
- **`rss_http_basic_auth_check()`** — HTTP Basic auth validation with
  constant-time credential comparison.
- **`rss_tls_init()`** — shared TLS server initialization for HTTPS
  endpoints (used by RHD and RWD).
- **`rss_secure_compare()`** — constant-time string comparison for
  credential checks (prevents timing side-channel attacks).
- **`rss_config_*()`** — INI config parser (load, get, set, save).
- **`rss_log_*()`** — syslog or stderr logging with levels.
- **`rss_daemonize()`** — double-fork, PID file, duplicate instance check.
- **`rss_signal_init()`** — SIGTERM/SIGINT handlers, SIGHUP/SIGPIPE ignored.
- **Utilities**: `rss_timestamp_us()`, `rss_strlcpy()`, `rss_trim()`,
  `rss_mkdir_p()`, `rss_read_file()`, `rss_write_file_atomic()`.

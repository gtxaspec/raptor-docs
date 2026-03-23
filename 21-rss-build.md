# Raptor Streaming System -- Build System

CMake project structure, cross-compilation toolchain, Buildroot package
integration, x86 stub build for development, and debug build with
sanitizers.

---

## 1. CMake Structure

```
raptor/
├── CMakeLists.txt                  # top-level: options, toolchain, subdirs
├── cmake/
│   ├── mips-linux-gnu.cmake        # cross-compilation toolchain file
│   ├── x86-stub.cmake              # x86 stub toolchain overrides
│   └── FindIngenicSDK.cmake        # find ingenic-lib + ingenic-headers
├── hal/
│   ├── CMakeLists.txt              # librss_hal (static)
│   ├── include/
│   │   └── raptor_hal.h            # public HAL API (see 10-hal-api.md)
│   └── src/
│       ├── hal_common.c            # factory, destroy, shared utils
│       ├── hal_gen1.c              # T20/T21/T30
│       ├── hal_t23.c               # T23
│       ├── hal_t31.c               # T31
│       ├── hal_t32.c               # T32
│       ├── hal_t40.c               # T40
│       ├── hal_t41.c               # T41
│       └── hal_stub.c              # x86 stub (synthetic frames)
├── lib/
│   ├── librss_ipc/
│   │   ├── CMakeLists.txt          # librss_ipc (shared or static)
│   │   ├── include/
│   │   │   └── rss_ipc.h            # SHM ring buffer + OSD + control API
│   │   └── src/
│   │       ├── rss_ring.c
│   │       ├── rss_osd_shm.c
│   │       └── rss_ctrl.c
│   └── librss_common/
│       ├── CMakeLists.txt          # librss_common (static)
│       ├── include/
│       │   ├── rss_config.h        # JSON config parser
│       │   ├── rss_log.h           # logging (syslog + stderr)
│       │   └── rss_util.h          # misc utilities (daemonize, pid, signal)
│       └── src/
│           ├── config.c
│           ├── log.c
│           └── util.c
├── daemons/
│   ├── rvd/
│   │   ├── CMakeLists.txt          # rvd binary
│   │   └── src/
│   │       ├── rvd_main.c
│   │       ├── rvd_pipeline.c      # HAL pipeline setup/teardown
│   │       ├── rvd_frame_loop.c    # encoder poll + ring publish
│   │       └── rvd_osd.c           # OSD SHM consumer (eventfd listener)
│   ├── rod/
│   │   ├── CMakeLists.txt          # rod binary
│   │   └── src/
│   │       ├── rod_main.c
│   │       ├── rod_render.c        # libschrift text rendering
│   │       └── rod_osd_pub.c       # OSD SHM producer (double-buffer + eventfd)
│   ├── rad/
│   │   ├── CMakeLists.txt          # rad binary
│   │   └── src/
│   │       ├── rad_main.c
│   │       └── rad_audio_loop.c    # audio read + encode + ring publish
│   ├── rsd/
│   │   ├── CMakeLists.txt          # rsd binary
│   │   └── src/
│   │       ├── rsd_main.c
│   │       ├── rsd_rtsp.c          # RTSP session management
│   │       └── rsd_rtp.c           # RTP packetization
│   ├── rmr/
│   │   ├── CMakeLists.txt
│   │   └── src/
│   │       ├── rmr_main.c
│   │       └── rmr_muxer.c         # MP4/MKV muxing
│   ├── rsp/
│   │   ├── CMakeLists.txt
│   │   └── src/
│   │       ├── rsp_main.c
│   │       └── rsp_push.c          # RTMP/RTSP push client
│   ├── rv4/
│   │   ├── CMakeLists.txt
│   │   └── src/
│   │       ├── rv4_main.c
│   │       └── rv4_v4l2.c          # V4L2 output device
│   ├── ric/
│   │   ├── CMakeLists.txt
│   │   └── src/
│   │       ├── ric_main.c
│   │       └── ric_daynight.c      # exposure monitor + IR-cut control
│   └── rmc/
│       ├── CMakeLists.txt
│       └── src/
│           ├── rmc_main.c
│           └── rmc_motor.c         # stepper/UART motor control
├── tools/
│   ├── raptorctl/
│   │   ├── CMakeLists.txt          # raptorctl binary
│   │   └── src/
│   │       └── raptorctl.c
│   └── ringdump/
│       ├── CMakeLists.txt          # debug tool: dump SHM ring contents
│       └── src/
│           └── ringdump.c
├── config/
│   └── raptor.json                 # default configuration file
└── res/
    └── default.ttf                 # default OSD font
```

### 1.1 Top-Level CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.10)
project(raptor C)

set(CMAKE_C_STANDARD 11)
set(CMAKE_C_STANDARD_REQUIRED ON)

# --- Platform selection ---
# Set by toolchain file or manually: -DRSS_SOC=T31
set(RSS_SOC "T31" CACHE STRING "Target SoC: T20, T21, T23, T30, T31, T32, T40, T41, STUB")
set(RSS_PLATFORM "PLATFORM_${RSS_SOC}")
add_definitions(-D${RSS_PLATFORM})

message(STATUS "Raptor: building for ${RSS_SOC} (${RSS_PLATFORM})")

# --- Build type defaults ---
if(NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE "Release" CACHE STRING "" FORCE)
endif()

# --- Common compiler flags ---
add_compile_options(-Wall -Wextra -Werror=return-type -Wno-unused-parameter)
if(CMAKE_BUILD_TYPE STREQUAL "Release")
    add_compile_options(-Os)
endif()

# --- Find Ingenic SDK (cross-build only) ---
if(NOT RSS_SOC STREQUAL "STUB")
    list(APPEND CMAKE_MODULE_PATH "${CMAKE_SOURCE_DIR}/cmake")
    find_package(IngenicSDK REQUIRED)
endif()

# --- Optional dependencies ---
find_package(PkgConfig)
if(PKG_CONFIG_FOUND)
    pkg_check_modules(CJSON cjson)
endif()

# If cJSON not found via pkg-config, fall back to json-c
if(NOT CJSON_FOUND)
    pkg_check_modules(JSONC json-c)
endif()

# --- Libraries ---
add_subdirectory(hal)
add_subdirectory(lib/librss_ipc)
add_subdirectory(lib/librss_common)

# --- Daemons ---
add_subdirectory(daemons/rvd)
add_subdirectory(daemons/rad)
add_subdirectory(daemons/rod)

add_subdirectory(daemons/rsd)
add_subdirectory(daemons/rmr)
add_subdirectory(daemons/rsp)
add_subdirectory(daemons/rv4)
add_subdirectory(daemons/ric)
add_subdirectory(daemons/rmc)

# --- Tools ---
add_subdirectory(tools/raptorctl)
add_subdirectory(tools/ringdump)

# --- Install ---
install(FILES config/raptor.json DESTINATION /etc)
install(FILES res/default.ttf DESTINATION /usr/share/fonts)
```

### 1.2 HAL CMakeLists.txt

```cmake
# hal/CMakeLists.txt
add_library(rss_hal STATIC src/hal_common.c)

# Select per-SoC implementation source
if(RSS_SOC STREQUAL "STUB")
    target_sources(rss_hal PRIVATE src/hal_stub.c)
elseif(RSS_SOC STREQUAL "T20" OR RSS_SOC STREQUAL "T21" OR RSS_SOC STREQUAL "T30")
    target_sources(rss_hal PRIVATE src/hal_gen1.c)
elseif(RSS_SOC STREQUAL "T23")
    target_sources(rss_hal PRIVATE src/hal_t23.c)
elseif(RSS_SOC STREQUAL "T31")
    target_sources(rss_hal PRIVATE src/hal_t31.c)
elseif(RSS_SOC STREQUAL "T32")
    target_sources(rss_hal PRIVATE src/hal_t32.c)
elseif(RSS_SOC STREQUAL "T40")
    target_sources(rss_hal PRIVATE src/hal_t40.c)
elseif(RSS_SOC STREQUAL "T41")
    target_sources(rss_hal PRIVATE src/hal_t41.c)
else()
    message(FATAL_ERROR "Unknown RSS_SOC: ${RSS_SOC}")
endif()

target_include_directories(rss_hal PUBLIC include)

if(NOT RSS_SOC STREQUAL "STUB")
    target_include_directories(rss_hal PRIVATE ${INGENIC_SDK_INCLUDE_DIRS})
    target_link_libraries(rss_hal PRIVATE ${INGENIC_SDK_LIBRARIES})
endif()
```

### 1.3 Daemon CMakeLists.txt (RVD Example)

```cmake
# daemons/rvd/CMakeLists.txt
add_executable(rvd
    src/rvd_main.c
    src/rvd_pipeline.c
    src/rvd_frame_loop.c
    src/rvd_osd.c
)

target_link_libraries(rvd PRIVATE
    rss_hal
    rss_ipc
    rss_common
    pthread
    rt            # shm_open, clock_gettime
)

if(NOT RSS_SOC STREQUAL "STUB")
    target_link_libraries(rvd PRIVATE ${INGENIC_SDK_LIBRARIES})
endif()

install(TARGETS rvd DESTINATION /usr/bin)
```

---

## 2. Cross-Compilation Toolchain File

```cmake
# cmake/mips-linux-gnu.cmake
#
# Usage: cmake -DCMAKE_TOOLCHAIN_FILE=cmake/mips-linux-gnu.cmake \
#              -DRSS_SOC=T31 ..

set(CMAKE_SYSTEM_NAME Linux)
set(CMAKE_SYSTEM_PROCESSOR mips)

# Toolchain prefix -- set externally or use default
if(NOT DEFINED CROSS_COMPILE)
    set(CROSS_COMPILE "mips-linux-gnu-")
endif()

set(CMAKE_C_COMPILER   "${CROSS_COMPILE}gcc")
set(CMAKE_CXX_COMPILER "${CROSS_COMPILE}g++")
set(CMAKE_LINKER        "${CROSS_COMPILE}ld")
set(CMAKE_AR            "${CROSS_COMPILE}ar")
set(CMAKE_STRIP         "${CROSS_COMPILE}strip")

# Sysroot -- set by Buildroot or manually
if(DEFINED ENV{STAGING_DIR})
    set(CMAKE_SYSROOT "$ENV{STAGING_DIR}")
    set(CMAKE_FIND_ROOT_PATH "$ENV{STAGING_DIR}")
endif()

set(CMAKE_FIND_ROOT_PATH_MODE_PROGRAM NEVER)
set(CMAKE_FIND_ROOT_PATH_MODE_LIBRARY ONLY)
set(CMAKE_FIND_ROOT_PATH_MODE_INCLUDE ONLY)
set(CMAKE_FIND_ROOT_PATH_MODE_PACKAGE ONLY)

# XBurst CPU flags -- critical for Ingenic MIPS
# -mno-fused-madd or -ffp-contract=off prevents FP miscompilations on XBurst
set(CMAKE_C_FLAGS_INIT "-mips32r2 -mno-fused-madd -EL")
set(CMAKE_C_FLAGS_RELEASE_INIT "-Os")
set(CMAKE_C_FLAGS_DEBUG_INIT "-O0 -g -fno-omit-frame-pointer")
```

---

## 3. Buildroot Package

### 3.1 raptor.mk

Based on the prudynt-t.mk pattern. Builds the full RSS daemon suite
as a single Buildroot package.

```makefile
################################################################################
#
# raptor -- Raptor Streaming System for Ingenic SoC cameras
#
################################################################################

RAPTOR_SITE_METHOD = git
RAPTOR_SITE = https://github.com/themactep/raptor
RAPTOR_SITE_BRANCH = main
RAPTOR_VERSION = $(RAPTOR_SITE_BRANCH)

# --- Dependencies ---

RAPTOR_DEPENDENCIES += ingenic-lib
RAPTOR_DEPENDENCIES += libschrift

# ROD requires libschrift for text rendering
ifeq ($(BR2_PACKAGE_RAPTOR_OSD),y)
	RAPTOR_DEPENDENCIES += libschrift
endif

# RSD RTSP server -- live555 or custom
ifeq ($(BR2_PACKAGE_RAPTOR_RTSP_LIVE555),y)
	RAPTOR_DEPENDENCIES += thingino-live555
	RAPTOR_CFLAGS += -DRSD_USE_LIVE555
endif

# RMR recording -- optional ffmpeg for muxing
ifeq ($(BR2_PACKAGE_RAPTOR_RECORDER_FFMPEG),y)
	RAPTOR_DEPENDENCIES += thingino-ffmpeg
	RAPTOR_CFLAGS += -DRMR_USE_FFMPEG
endif

# RSP stream push -- requires libcurl
ifeq ($(BR2_PACKAGE_RAPTOR_PUSH),y)
	RAPTOR_DEPENDENCIES += thingino-libcurl
endif

# JSON config parser
ifeq ($(BR2_PACKAGE_CJSON),y)
	RAPTOR_DEPENDENCIES += cjson
else
	RAPTOR_DEPENDENCIES += json-c
endif

# C library shim for vendor SDK compatibility
ifeq ($(BR2_TOOLCHAIN_USES_MUSL),y)
	RAPTOR_DEPENDENCIES += ingenic-musl
	RAPTOR_SHIM_LIB = -lmuslshim
endif

ifeq ($(BR2_TOOLCHAIN_USES_UCLIBC),y)
	RAPTOR_DEPENDENCIES += ingenic-uclibc
	RAPTOR_SHIM_LIB = -luclibcshim
endif

# --- Compiler flags ---

# Inherit architecture-specific flags (critical for XBurst: -mno-fused-madd)
RAPTOR_CFLAGS += $(TARGET_CFLAGS)

# Platform define: -DPLATFORM_T31, -DPLATFORM_T40, etc.
RAPTOR_CFLAGS += -DPLATFORM_$(shell echo $(SOC_FAMILY) | tr a-z A-Z)

ifeq ($(KERNEL_VERSION),4.4.94)
	RAPTOR_CFLAGS += -DKERNEL_VERSION_4
endif

# Ingenic SDK include paths
RAPTOR_CFLAGS += \
	-I$(STAGING_DIR)/usr/include

ifeq ($(BR2_TOOLCHAIN_USES_GLIBC),y)
	RAPTOR_CFLAGS += -DLIBC_GLIBC
endif

ifeq ($(BR2_TOOLCHAIN_USES_UCLIBC),y)
	RAPTOR_CFLAGS += -DLIBC_UCLIBC
endif

# --- Linker flags ---

RAPTOR_LDFLAGS += $(TARGET_LDFLAGS) \
	-L$(STAGING_DIR)/usr/lib \
	-L$(TARGET_DIR)/usr/lib \
	-Wl,--no-as-needed $(RAPTOR_SHIM_LIB) -Wl,--as-needed

# --- Debug build ---

ifeq ($(BR2_PACKAGE_RAPTOR_DEBUG),y)
	RAPTOR_CFLAGS += -O0 -g -fno-omit-frame-pointer
	RAPTOR_CFLAGS += -fstack-protector-strong -D_FORTIFY_SOURCE=2
	RAPTOR_CFLAGS += -DDEBUG_BUILD=1

ifneq ($(BR2_TOOLCHAIN_USES_MUSL),y)
	RAPTOR_CFLAGS += -fsanitize=address
	RAPTOR_LDFLAGS += -fsanitize=address
endif

	RAPTOR_STRIP_BINARY = NO
	RAPTOR_CMAKE_OPTS += -DCMAKE_BUILD_TYPE=Debug
else
	RAPTOR_CFLAGS += -Os
	RAPTOR_STRIP_BINARY = YES
	RAPTOR_CMAKE_OPTS += -DCMAKE_BUILD_TYPE=Release
endif

# --- Map SOC_FAMILY to RSS_SOC ---

RAPTOR_SOC_MAP_t20 = T20
RAPTOR_SOC_MAP_t21 = T21
RAPTOR_SOC_MAP_t23 = T23
RAPTOR_SOC_MAP_t30 = T30
RAPTOR_SOC_MAP_t31 = T31
RAPTOR_SOC_MAP_t32 = T32
RAPTOR_SOC_MAP_t40 = T40
RAPTOR_SOC_MAP_t41 = T41
RAPTOR_RSS_SOC = $(RAPTOR_SOC_MAP_$(SOC_FAMILY))

# --- CMake build ---

RAPTOR_CONF_OPTS += \
	-DCMAKE_TOOLCHAIN_FILE=$(@D)/cmake/mips-linux-gnu.cmake \
	-DCROSS_COMPILE=$(TARGET_CROSS) \
	-DRSS_SOC=$(RAPTOR_RSS_SOC) \
	-DCMAKE_C_FLAGS="$(RAPTOR_CFLAGS)" \
	-DCMAKE_EXE_LINKER_FLAGS="$(RAPTOR_LDFLAGS)" \
	$(RAPTOR_CMAKE_OPTS)

define RAPTOR_CONFIGURE_CMDS
	mkdir -p $(@D)/build
	cd $(@D)/build && $(TARGET_MAKE_ENV) cmake \
		$(RAPTOR_CONF_OPTS) \
		$(@D)
endef

define RAPTOR_BUILD_CMDS
	$(TARGET_MAKE_ENV) $(MAKE) -C $(@D)/build
endef

define RAPTOR_INSTALL_TARGET_CMDS
	# Core daemons (always installed)
	$(INSTALL) -D -m 0755 $(@D)/build/daemons/rvd/rvd \
		$(TARGET_DIR)/usr/bin/rvd
	$(INSTALL) -D -m 0755 $(@D)/build/daemons/rad/rad \
		$(TARGET_DIR)/usr/bin/rad
	$(INSTALL) -D -m 0755 $(@D)/build/daemons/ric/ric \
		$(TARGET_DIR)/usr/bin/ric

	# CLI tool
	$(INSTALL) -D -m 0755 $(@D)/build/tools/raptorctl/raptorctl \
		$(TARGET_DIR)/usr/bin/raptorctl

	# Optional daemons
	if [ -f $(@D)/build/daemons/rod/rod ]; then \
		$(INSTALL) -D -m 0755 $(@D)/build/daemons/rod/rod \
			$(TARGET_DIR)/usr/bin/rod; \
	fi
	$(INSTALL) -D -m 0755 $(@D)/build/daemons/rsd/rsd \
		$(TARGET_DIR)/usr/bin/rsd
	if [ -f $(@D)/build/daemons/rmr/rmr ]; then \
		$(INSTALL) -D -m 0755 $(@D)/build/daemons/rmr/rmr \
			$(TARGET_DIR)/usr/bin/rmr; \
	fi
	if [ -f $(@D)/build/daemons/rsp/rsp ]; then \
		$(INSTALL) -D -m 0755 $(@D)/build/daemons/rsp/rsp \
			$(TARGET_DIR)/usr/bin/rsp; \
	fi
	if [ -f $(@D)/build/daemons/rv4/rv4 ]; then \
		$(INSTALL) -D -m 0755 $(@D)/build/daemons/rv4/rv4 \
			$(TARGET_DIR)/usr/bin/rv4; \
	fi
	if [ -f $(@D)/build/daemons/rmc/rmc ]; then \
		$(INSTALL) -D -m 0755 $(@D)/build/daemons/rmc/rmc \
			$(TARGET_DIR)/usr/bin/rmc; \
	fi

	# Configuration
	$(INSTALL) -D -m 0644 $(@D)/config/raptor.json \
		$(TARGET_DIR)/etc/raptor.json

	# Init script
	$(INSTALL) -D -m 0755 $(@D)/config/S31raptor \
		$(TARGET_DIR)/etc/init.d/S31raptor

	# Font
	$(INSTALL) -D -m 0644 $(@D)/res/default.ttf \
		$(TARGET_DIR)/usr/share/fonts/default.ttf

	# Strip binaries unless debug build
	if [ "$(RAPTOR_STRIP_BINARY)" = "YES" ]; then \
		for bin in rvd rad rod rsd rmr rsp rv4 ric rmc raptorctl; do \
			[ -f $(TARGET_DIR)/usr/bin/$$bin ] && \
				$(TARGET_CROSS)strip $(TARGET_DIR)/usr/bin/$$bin || true; \
		done; \
	fi

	# Debug build: install unstripped binaries to NFS
	if [ "$(BR2_PACKAGE_RAPTOR_DEBUG)" = "y" ] && [ -n "$(BR2_THINGINO_NFS)" ]; then \
		mkdir -p $(BR2_THINGINO_NFS)/$(CAMERA)/usr/bin; \
		for bin in rvd rad rod rsd rmr rsp rv4 ric rmc raptorctl; do \
			[ -f $(@D)/build/daemons/$$bin/$$bin ] && \
				$(INSTALL) -D -m 0755 $(@D)/build/daemons/$$bin/$$bin \
					$(BR2_THINGINO_NFS)/$(CAMERA)/usr/bin/$$bin-debug || true; \
		done; \
		[ -f $(@D)/build/tools/raptorctl/raptorctl ] && \
			$(INSTALL) -D -m 0755 $(@D)/build/tools/raptorctl/raptorctl \
				$(BR2_THINGINO_NFS)/$(CAMERA)/usr/bin/raptorctl-debug || true; \
	fi
endef

$(eval $(generic-package))
```

### 3.2 Config.in

```kconfig
config BR2_PACKAGE_RAPTOR
	bool "raptor"
	depends on BR2_PACKAGE_INGENIC_LIB
	help
	  Raptor Streaming System - microservices streaming platform
	  for Ingenic SoC IP cameras.

if BR2_PACKAGE_RAPTOR

config BR2_PACKAGE_RAPTOR_OSD
	bool "OSD overlay daemon (ROD)"
	default y
	select BR2_PACKAGE_LIBSCHRIFT
	help
	  Build the OSD overlay daemon with libschrift text rendering.

config BR2_PACKAGE_RAPTOR_RTSP_LIVE555
	bool "RTSP server uses live555"
	default n
	help
	  Use live555 for RSD RTSP server. If disabled, uses built-in
	  minimal RTSP implementation.

config BR2_PACKAGE_RAPTOR_RECORDER_FFMPEG
	bool "Recorder uses ffmpeg muxer"
	default n
	select BR2_PACKAGE_THINGINO_FFMPEG
	help
	  Use ffmpeg for RMR recording. If disabled, uses built-in
	  lightweight MP4 muxer.

config BR2_PACKAGE_RAPTOR_PUSH
	bool "Stream push daemon (RSP)"
	default n
	select BR2_PACKAGE_THINGINO_LIBCURL
	help
	  Build the RTMP/RTSP stream push daemon.

config BR2_PACKAGE_RAPTOR_DEBUG
	bool "Debug build"
	default n
	help
	  Build with debug symbols, sanitizers, and no optimization.
	  Requires BR2_THINGINO_NFS for debug binary installation.

endif # BR2_PACKAGE_RAPTOR
```

---

## 4. FindIngenicSDK.cmake

```cmake
# cmake/FindIngenicSDK.cmake
#
# Finds Ingenic SDK headers and libraries in the Buildroot staging dir.
#
# Sets:
#   INGENIC_SDK_FOUND
#   INGENIC_SDK_INCLUDE_DIRS
#   INGENIC_SDK_LIBRARIES

if(NOT DEFINED CMAKE_SYSROOT)
    message(FATAL_ERROR "CMAKE_SYSROOT not set -- cannot find Ingenic SDK")
endif()

set(INGENIC_SDK_INCLUDE_DIRS "${CMAKE_SYSROOT}/usr/include")

# Core vendor libraries
set(_sdk_libs imp isp alog sysutils)
set(INGENIC_SDK_LIBRARIES "")

foreach(_lib ${_sdk_libs})
    find_library(_found_${_lib} NAMES ${_lib}
        PATHS "${CMAKE_SYSROOT}/usr/lib"
        NO_DEFAULT_PATH)
    if(_found_${_lib})
        list(APPEND INGENIC_SDK_LIBRARIES ${_found_${_lib}})
    endif()
endforeach()

# Always need pthread and rt
list(APPEND INGENIC_SDK_LIBRARIES pthread rt)

include(FindPackageHandleStandardArgs)
find_package_handle_standard_args(IngenicSDK DEFAULT_MSG
    INGENIC_SDK_INCLUDE_DIRS INGENIC_SDK_LIBRARIES)
```

---

## 5. x86 Stub Build

For development on a workstation without Ingenic hardware. The stub HAL
generates synthetic frames (color bars with incrementing counter) and
synthetic audio (sine wave). All IPC (SHM rings, OSD double-buffers,
control sockets) works identically to the target build.

### 5.1 Building

```sh
mkdir build-x86 && cd build-x86
cmake -DRSS_SOC=STUB ..
make -j$(nproc)
```

No cross-compilation toolchain, no Ingenic SDK. The stub HAL
(`hal/src/hal_stub.c`) implements the full `rss_hal_ops_t` vtable:

- `init()`: no-op (no hardware to initialize)
- `enc_poll()`: sleeps for 1/fps_num seconds, then returns 0
- `enc_get_frame()`: generates a synthetic H264 IDR/slice frame
  containing color bar pattern with embedded timestamp
- `audio_read_frame()`: generates PCM silence or a 1kHz sine tone
- `isp_get_exposure()`: returns a fixed synthetic exposure value
- All ISP tuning functions: store values locally, log the call
- `osd_*`: track region state in memory, no actual rendering

### 5.2 What Works on x86

- Full daemon lifecycle (start, run, stop, restart)
- SHM ring buffer producer/consumer (all daemons)
- OSD double-buffer protocol (ROD -> RVD)
- Control socket protocol (raptorctl -> all daemons)
- RTSP server (RSD serves synthetic streams to VLC/ffplay)
- Recording (RMR writes MP4 with synthetic frames)
- Configuration parsing
- Logging

### 5.3 What Does Not Work on x86

- Actual video encoding (stub frames are synthetic NAL units)
- ISP tuning effects (values are stored but not applied)
- GPIO/IR-cut control (simulated in software)
- Motor control (simulated)
- Real-time performance characteristics (x86 timing differs from MIPS)

### 5.4 Stub Capabilities

The stub HAL reports capabilities similar to T31 by default:

```c
static const rss_hal_caps_t stub_caps = {
    .soc_name = "STUB",
    .sdk_version = "stub-1.0",
    .max_fs_channels = 3,
    .max_enc_channels = 8,
    .max_osd_groups = 4,
    .max_osd_regions = 8,
    .has_h265 = true,
    .has_rotation = true,
    .has_hw_rotation = false,
    .has_fcrop = true,
    .has_capped_rc = true,
    .has_gop_attr = true,
    .has_set_bitrate = true,
    .has_bcsh_hue = true,
    /* ... */
};
```

Override individual caps via environment variables for testing graceful
degradation:

```sh
RSS_STUB_HAS_H265=0 ./rvd -c raptor.json    # simulate T20 (no H265)
```

---

## 6. Debug Build with Sanitizers

### 6.1 Native Debug (x86)

```sh
mkdir build-debug && cd build-debug
cmake -DRSS_SOC=STUB \
      -DCMAKE_BUILD_TYPE=Debug \
      -DRSS_SANITIZERS=ON ..
make -j$(nproc)
```

When `RSS_SANITIZERS=ON`, the CMake config adds:

```cmake
if(RSS_SANITIZERS)
    add_compile_options(-fsanitize=address,undefined -fno-omit-frame-pointer)
    add_link_options(-fsanitize=address,undefined)
    message(STATUS "Raptor: ASan + UBSan enabled")
endif()
```

This enables:
- **AddressSanitizer**: heap/stack buffer overflows, use-after-free,
  double-free, memory leaks (at exit)
- **UndefinedBehaviorSanitizer**: signed integer overflow, null pointer
  dereference, misaligned access, shift overflow

### 6.2 Cross-Compiled Debug (Target)

```sh
# In Buildroot menuconfig, enable BR2_PACKAGE_RAPTOR_DEBUG=y
# This sets -O0 -g and conditionally adds ASan (glibc/uclibc only)
make raptor-rebuild
```

On musl-based toolchains, ASan is not available. The build falls back to
stack protectors and `_FORTIFY_SOURCE=2`.

Debug binaries are installed to NFS (not to the firmware image) to avoid
bloating the flash:

```
$BR2_THINGINO_NFS/$CAMERA/usr/bin/rvd-debug
$BR2_THINGINO_NFS/$CAMERA/usr/bin/rad-debug
...
```

Run on target via NFS mount:

```sh
mount -t nfs devhost:/nfs /mnt/nfs
/mnt/nfs/mydevice/usr/bin/rvd-debug -c /etc/raptor.json
```

### 6.3 Valgrind (x86 Stub)

Valgrind works on the x86 stub build for deeper memory analysis:

```sh
valgrind --leak-check=full --track-origins=yes ./rvd -c raptor.json &
valgrind --leak-check=full ./rsd -c raptor.json &
# ... run test, then stop daemons
```

Valgrind is too slow for target MIPS, so it is only used on x86 stubs.

---

## 7. Dependency List

| Dependency | Used By | Purpose | Required? |
|-----------|---------|---------|-----------|
| ingenic-lib | HAL (all SoC builds) | libimp.so, libisp.so, libalog.so vendor SDK | Yes (target) |
| ingenic-headers | HAL | IMP SDK headers | Yes (target) |
| ingenic-musl or ingenic-uclibc | all daemons | C library shim for vendor SDK | Yes (per toolchain) |
| libschrift | ROD | TrueType font rendering for OSD text (~1300 lines, no deps) | Yes for ROD |
| cJSON or json-c | librss_common | JSON configuration file parsing | Yes (one of) |
| live555 | RSD (optional) | RTSP/RTP server framework | Optional |
| libcurl | RSP | HTTP/RTMP client for stream push | Yes for RSP |
| ffmpeg (libavformat) | RMR (optional) | MP4/MKV muxing for recording | Optional |
| pthread | all daemons | POSIX threads | Yes (system) |
| librt | all daemons | shm_open, clock_gettime | Yes (system) |

### 7.1 Minimal Build (Smallest Footprint)

For a minimal streaming-only camera:

```
Daemons: RVD + RSD
Dependencies: ingenic-lib, ingenic-headers, cJSON, pthread, rt
Optional: none
Approximate binary sizes (stripped):
  rvd: ~80KB
  rsd: ~60KB
  raptorctl: ~20KB
```

### 7.2 Full Build

```
Daemons: RVD + RAD + ROD + RSD + RMR + RSP + RV4 + RIC + RMC
Dependencies: all of the above
Approximate total binary size (stripped): ~400KB
```

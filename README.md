# Raptor Streaming System - Documentation

Architecture docs, design notes, SDK reference, and API documentation for the
[Raptor Streaming System](https://github.com/gtxaspec/raptor).

## Contents

### Overview
- [00 - Overview](00-overview.md) - System overview and SDK/HAL documentation

### Ingenic SDK Reference
- [01 - Common Types](01-sdk-common.md) - `imp_common.h` cross-SoC differences
- [02 - System](02-sdk-system.md) - `imp_system.h` cross-SoC differences
- [03 - Encoder](03-sdk-encoder.md) - IMP Encoder module cross-SoC reference
- [04 - FrameSource](04-sdk-framesource.md) - `imp_framesource.h` API
- [05 - ISP](05-sdk-isp.md) - `imp_isp.h` API
- [06 - Audio](06-sdk-audio.md) - `imp_audio.h` API
- [07 - OSD](07-sdk-osd.md) - `imp_osd.h` / `isp_osd.h` API
- [08 - Libraries](08-sdk-libraries.md) - SDK prebuilt libraries (ingenic-lib)

### HAL (Hardware Abstraction Layer)
- [10 - HAL API](10-hal-api.md) - Public API reference
- [11 - HAL Internals](11-hal-internals.md) - Internal implementation guide
- [12 - HAL Capabilities](12-hal-caps.md) - Capability struct and per-SoC values

### System Architecture
- [20 - Architecture](20-rss-architecture.md) - Daemon architecture and IPC design
- [21 - Build System](21-rss-build.md) - Makefile, buildroot package, standalone build
- [22 - Testing](22-rss-testing.md) - Test architecture and ASAN builds
- [23 - Configuration](23-rss-config.md) - `raptor.conf` reference
- [24 - Ring Buffer Guide](24-ring-consumer-guide.md) - SHM ring consumer implementation

### Daemon Design
- [25 - RWD WebRTC](25-rwd-webrtc-design.md) - WebRTC daemon design (WHIP, ICE, DTLS-SRTP)
- [26 - RIC Day/Night](26-ric-daynight-design.md) - Day/night detection design
- [27 - Multi-Sensor](27-multi-sensor.md) - Dual/triple camera support
- [28 - IVS Detection](28-ivs-detection.md) - Motion tracking, person detection, YOLO inference
- [29 - raptorctl Reference](29-raptorctl-reference.md) - Command reference
- [30 - FPS Troubleshooting](30-fps-troubleshooting.md) - FPS drop diagnosis

### Development
- [99 - Dev Guide](99-dev-guide.md) - Development workflow and conventions

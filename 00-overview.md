# Raptor Streaming System — SDK & HAL Documentation

## Purpose

This documentation set provides everything needed to implement the RSS Hardware Abstraction Layer (HAL) across 8 Ingenic SoCs. An implementing agent should be able to read these docs and build the HAL without needing to cross-reference 100+ header files.

## RSS Architecture Summary

RSS is a microservices-based streaming platform for embedded IP cameras:

```
┌────────────────────────────────────────────────────────────┐
│                    RAPTOR STREAMING SYSTEM                   │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐  │
│  │                        HAL                              │  │
│  │      T20 / T21 / T23 / T30 / T31 / T32 / T40 / T41    │  │
│  └───────────────────────┬────────────────────────────────┘  │
│                          │                                    │
│  ┌───────────────────────┴────────────────────────────────┐  │
│  │  Producers: RVD (video), ROD (OSD), RAD (audio)        │  │
│  │       │ SHM ring buffers + eventfd                      │  │
│  │       ▼                                                  │  │
│  │  Consumers: RSD (RTSP), RMR (recorder), RSP (push),    │  │
│  │             RV4 (V4L2 bridge)                           │  │
│  │                                                          │  │
│  │  Control: RIC (IR/day-night), RMC (motors)              │  │
│  └─────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────┘
```

The HAL is consumed primarily by **RVD** (video pipeline), **RAD** (audio), and **RIC** (exposure queries). All other daemons interact via IPC, not hardware.

## SDK Generations

The Ingenic libimp API has evolved across SoC generations. There are three distinct API variants:

### Old SDK (T20, T21, T23, T30)
- Encoder: `IMPEncoderCHNAttr` (capital CHN), fields `picWidth`/`picHeight`, `bufSize`, `enType`
- Encoder packs: direct `virAddr`/`phyAddr` per pack, NAL type via `dataType.h264Type`
- RC modes: `ENC_RC_MODE_FIXQP`, `ENC_RC_MODE_CBR`, `ENC_RC_MODE_VBR`, `ENC_RC_MODE_SMART`
- ISP: scalar parameters — `IMP_ISP_Tuning_SetBrightness(unsigned char bright)`
- ISP sensor: `IMP_ISP_AddSensor(IMPSensorInfo *info)` (no IMPVI_NUM)
- No `SetDefaultParam()`, no `gopAttr`, no buffer sharing

### New SDK (T31, T32)
- Encoder: `IMPEncoderChnAttr` (camelCase), fields `uWidth`/`uHeight`, no `bufSize`/`enType`
- Encoder packs: ring-buffer with `stream.virAddr + pack[i].offset`, NAL via `nalType.h264NalType`
- RC modes: `IMP_ENC_RC_MODE_FIXQP`, `IMP_ENC_RC_MODE_CBR`, `IMP_ENC_RC_MODE_VBR`, `IMP_ENC_RC_MODE_CAPPED_VBR`, `IMP_ENC_RC_MODE_CAPPED_QUALITY`
- `IMP_Encoder_SetDefaultParam()` for channel init
- `gopAttr` field in channel attributes
- ISP: scalar parameters with pointers — `IMP_ISP_Tuning_SetBrightness(unsigned char *bright)` (T31)
- ISP sensor: `IMP_ISP_AddSensor(IMPSensorInfo *info)` (no IMPVI_NUM) on T31
- T32: extended encoder (101 functions), ISP OSD regions, power management

### IMPVI SDK (T40, T41)
- Same encoder API as New SDK
- ISP: all tuning functions take `IMPVI_NUM` as first param — `IMP_ISP_Tuning_SetBrightness(IMPVI_NUM num, unsigned char *bright)`
- ISP sensor: `IMP_ISP_AddSensor(IMPVI_NUM num, IMPSensorInfo *pinfo)`
- Extended `IMPSensorInfo` with `sensor_id`, `video_interface` (IMPSensorVinType), `mclk` (IMPSensorMclk)
- Face AE/AWB, HLDC, auto zoom, extended statistics

## Source Paths

### Headers (ground truth for HAL implementation)
```
~/projects/thingino/ingenic-headers/{SoC}/{version}/{lang}/imp/
```

| SoC | Version | Language | Header Count |
|-----|---------|----------|-------------|
| T20 | 3.12.0 | Chinese | 13 |
| T21 | 1.0.33 | Chinese | 13 |
| T23 | 1.3.0 | English | 14 (+isp_osd.h) |
| T30 | 1.0.5 | Chinese | 14 (+imp_dmic.h) |
| T31 | 1.1.6 | English | 15 (+imp_dmic.h, imp_emu_framesource.h) |
| T32 | 1.0.6 | English | 14 (+isp_osd.h, no imp_decoder.h) |
| T40 | 1.3.1 | English | 16 (all headers) |
| T41 | 1.2.5 | English | 16 (all headers) |

### Libraries (link targets)
```
~/projects/thingino/ingenic-lib/{SoC}/lib/{version}/{libc}/{gcc_version}/
```

Core libraries for all SoCs:
- `libimp.so` / `libimp.a` — main IMP library (~1.2-2.4MB)
- `libalog.so` / `libalog.a` — logging (~30-40KB)
- `libaudioProcess.so` — audio DSP firmware (T31+, ~650-740KB)
- `libsysutils.so` / `libsysutils.a` — system utilities (~30-48KB)

### Kernel Drivers
```
~/projects/thingino/ingenic-sdk/{kernel_version}/{module}/{soc}/
```
- 3.10 kernel: T20, T21, T23, T30, T31, T41
- 4.4 kernel: T31, T40, T41, A1, C100

### PDF/HTML Documentation
```
~/projects/thingino/ingenic-docs/{SoC}/SDK/
```
- Doxygen HTML: T10, T21, T30
- PDF API references: T31, T40, T41 (per-module PDFs)
- Ignore ZERATUL subdirectories

### Prior Art
```
~/projects/thingino/prudynt-t/src/imp_hal.hpp  — existing C++ HAL (PlatformCaps, namespace hal)
~/projects/thingino/prudynt-t/src/imp_hal.cpp  — existing C++ HAL implementation
```

## Build Integration

Thingino uses buildroot. The SoC is selected at build time:
- `SOC_FAMILY` variable set by board config (e.g., `t31`, `t40`)
- Passed to compiler as `-DPLATFORM_T31` (uppercased)
- Reference: `~/projects/thingino/thingino-firmware/package/prudynt-t/prudynt-t.mk`

## Document Index

### SDK Difference Docs (Phase 1)
- [01-sdk-common.md](01-sdk-common.md) — Common types (IMPFrameInfo, pixel formats, payload types)
- [02-sdk-system.md](02-sdk-system.md) — System API (init, bind, sensor info)
- [03-sdk-encoder.md](03-sdk-encoder.md) — Encoder API (the most complex module)
- [04-sdk-framesource.md](04-sdk-framesource.md) — FrameSource API
- [05-sdk-isp.md](05-sdk-isp.md) — ISP API (second most complex)
- [06-sdk-audio.md](06-sdk-audio.md) — Audio API
- [07-sdk-osd.md](07-sdk-osd.md) — OSD API
- [08-sdk-libraries.md](08-sdk-libraries.md) — Library inventory per SoC

### HAL Design Docs (Phase 2)
- [10-hal-api.md](10-hal-api.md) — HAL public API (`rss_hal_ops_t`)
- [11-hal-internals.md](11-hal-internals.md) — Implementation strategy per SoC
- [12-hal-caps.md](12-hal-caps.md) — Capability struct

### HAL Library (Implementation)
The HAL is implemented as `libraptor_hal.a` at `~/projects/thingino/raptor-hal/`.
- `include/raptor_hal.h` — single public header, the only file daemons include
- `src/` — implementation files (per-module, not per-SoC)
- `Makefile` — builds for any SoC: `make PLATFORM=T31 CROSS_COMPILE=mipsel-linux-gnu-`
- Compiles to ~52KB static library

## HAL Quick-Start Usage Example

```c
#include "raptor_hal.h"

int main(void) {
    /* Create HAL context */
    rss_hal_ctx_t *ctx = rss_hal_create();
    const rss_hal_ops_t *ops = rss_hal_get_ops();

    /* Configure sensor */
    rss_sensor_config_t sensor = {
        .name = "gc2053",
        .i2c_addr = 0x37,
        .i2c_adapter = 0,
        .rst_gpio = 18,
        .pwdn_gpio = -1,
        .power_gpio = -1,
    };

    /* Initialize: ISP open → add sensor → enable → system init */
    ops->init(ctx, &sensor);

    /* Query capabilities */
    const rss_hal_caps_t *caps = ops->get_caps(ctx);
    printf("SoC: %s, H265: %d\n", caps->soc_name, caps->has_h265);

    /* Create framesource channel 0 (1080p NV12) */
    rss_fs_config_t fs_cfg = {
        .width = 1920, .height = 1080,
        .pixfmt = RSS_PIXFMT_NV12,
        .fps_num = 25, .fps_den = 1,
    };
    ops->fs_create_channel(ctx, 0, &fs_cfg);

    /* Create encoder group 0 + channel 0 (H264 CBR 2Mbps) */
    ops->enc_create_group(ctx, 0);
    rss_video_config_t enc_cfg = {
        .codec = RSS_CODEC_H264,
        .width = 1920, .height = 1080,
        .rc_mode = RSS_RC_CBR,
        .bitrate = 2000000,
        .fps_num = 25, .fps_den = 1,
        .gop_length = 50,
        .min_qp = 15, .max_qp = 45,
    };
    ops->enc_create_channel(ctx, 0, &enc_cfg);
    ops->enc_register_channel(ctx, 0, 0);

    /* Bind: FS(0) → ENC(0) */
    rss_cell_t fs_cell = { .device = RSS_DEV_FS, .group = 0, .output = 0 };
    rss_cell_t enc_cell = { .device = RSS_DEV_ENC, .group = 0, .output = 0 };
    ops->bind(ctx, &fs_cell, &enc_cell);

    /* Enable framesource, start encoder */
    ops->fs_enable_channel(ctx, 0);
    ops->enc_start(ctx, 0);

    /* Stream loop */
    for (int i = 0; i < 100; i++) {
        if (ops->enc_poll(ctx, 0, 1000) == 0) {
            rss_frame_t frame;
            ops->enc_get_frame(ctx, 0, &frame);
            /* frame.nals[0..frame.nal_count-1] contain linearized NAL units */
            /* Feed to SHM ring buffer, RTSP, file, etc. */
            ops->enc_release_frame(ctx, 0, &frame);
        }
    }

    /* Teardown (reverse order) */
    ops->enc_stop(ctx, 0);
    ops->fs_disable_channel(ctx, 0);
    ops->unbind(ctx, &fs_cell, &enc_cell);
    ops->enc_unregister_channel(ctx, 0);
    ops->enc_destroy_channel(ctx, 0);
    ops->enc_destroy_group(ctx, 0);
    ops->fs_destroy_channel(ctx, 0);
    ops->deinit(ctx);
    rss_hal_destroy(ctx);
    return 0;
}
```

### RSS Architecture Docs (Phase 3)
- [20-rss-architecture.md](20-rss-architecture.md) — Daemon decomposition, IPC
- [21-rss-build.md](21-rss-build.md) — Build system integration
- [22-rss-testing.md](22-rss-testing.md) — Test strategy (T31 first)

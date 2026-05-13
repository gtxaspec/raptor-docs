# 08 — SDK Prebuilt Libraries (ingenic-lib)

## Overview

All Ingenic SoC SDKs ship as prebuilt binary libraries. Source code is not available. The HAL links against these at build time and the `.so` files must be deployed to the target device at runtime.

Library source: [ingenic-lib](https://github.com/gtxaspec/ingenic-lib) - `{SoC}/lib/{version}/{libc}/{gcc}/`

---

## 1. Latest Versions Per SoC

| SoC | Latest SDK Version | GCC Versions | libc Variants |
|---|---|---|---|
| T20 | 3.12.0 | 4.7.2 | uclibc, glibc |
| T21 | 1.0.33 | 4.7.2, 5.4.0 | uclibc, glibc |
| T23 | 1.3.0 | 5.4.0 | uclibc, glibc |
| T30 | 1.0.5 | 4.7.2, 5.4.0 | uclibc, glibc |
| T31 | 1.1.6 | 4.7.2, 5.4.0 | uclibc, glibc |
| T32 | 1.0.6 | (direct) | uclibc, glibc |
| T40 | 1.3.1 | 7.2.0 | uclibc, glibc |
| T41 | 1.2.5 | 7.2.0 | uclibc, glibc |

**Notes:**
- T32 libraries are placed directly in `uclibc/` and `glibc/` without a GCC version subdirectory.
- Thingino uses uclibc for all targets. The glibc variants are for reference.
- T20 only has gcc 4.7.2 builds. T40/T41 only have gcc 7.2.0 builds.

---

## 2. Core Library Files (Latest Version, uclibc)

### T20 (3.12.0 / uclibc / gcc 4.7.2)

| Library | .so size | .a size | Type |
|---|---|---|---|
| libimp | 1,016,684 | 1,357,612 | Mandatory - main IMP API |
| libalog | 35,261 | 40,528 | Mandatory - logging |
| libsysutils | 36,940 | 42,696 | Mandatory - system utilities |
| libaudioProcess | 689,434 | (none) | Optional - audio DSP |

### T21 (1.0.33 / uclibc / gcc 5.4.0)

| Library | .so size | .a size | Type |
|---|---|---|---|
| libimp | 887,116 | 1,205,436 | Mandatory |
| libalog | 34,916 | 40,700 | Mandatory |
| libsysutils | 36,864 | 43,188 | Mandatory |
| libaudioProcess | 664,084 | (none) | Optional |

### T23 (1.3.0 / uclibc / gcc 5.4.0)

| Library | .so size | .a size | Type |
|---|---|---|---|
| libimp | 1,162,360 | 1,547,264 | Mandatory |
| libalog | 34,920 | 40,800 | Mandatory |
| libsysutils | 36,912 | 44,128 | Mandatory |
| libaudioProcess | 720,720 | (none) | Optional |

### T30 (1.0.5 / uclibc / gcc 5.4.0)

| Library | .so size | .a size | Type |
|---|---|---|---|
| libimp | 1,888,048 | 2,457,826 | Mandatory |
| libalog | 34,916 | 40,700 | Mandatory |
| libsysutils | 35,096 | 42,828 | Mandatory |
| libaudioProcess | 664,084 | (none) | Optional |

### T31 (1.1.6 / uclibc / gcc 5.4.0)

| Library | .so size | .a size | Type |
|---|---|---|---|
| libimp | 1,202,720 | 1,670,140 | Mandatory |
| libalog | 34,900 | 40,668 | Mandatory |
| libsysutils | 29,500 | 32,702 | Mandatory |
| libaudioProcess | 671,472 | (none) | Optional |

### T32 (1.0.6 / uclibc / direct)

| Library | .so size | .a size | Type |
|---|---|---|---|
| libimp | 2,179,808 | 2,828,286 | Mandatory |
| libalog | 35,412 | 41,708 | Mandatory |
| libsysutils | 54,524 | 67,962 | Mandatory |
| libaudioProcess | 571,280 | (none) | Optional |

### T40 (1.3.1 / uclibc / gcc 7.2.0)

| Library | .so size | .a size | Type |
|---|---|---|---|
| libimp | 1,215,092 | 1,664,276 | Mandatory |
| libalog | 32,600 | 39,148 | Mandatory |
| libsysutils | 37,872 | 49,040 | Mandatory |
| libaudioProcess | 750,280 | (none) | Optional |

### T41 (1.2.5 / uclibc / gcc 7.2.0)

| Library | .so size | .a size | Type |
|---|---|---|---|
| libimp | 1,444,500 | 2,088,208 | Mandatory |
| libalog | 32,980 | 39,604 | Mandatory |
| libsysutils | 37,508 | 48,204 | Mandatory |
| libaudioProcess | 732,944 | (none) | Optional |

---

## 3. Library Size Comparison (libimp.so, uclibc)

| SoC | libimp.so | Relative |
|---|---|---|
| T21 | 887 KB | Smallest — basic single-sensor |
| T20 | 1,017 KB | Legacy, larger than T21 |
| T23 | 1,162 KB | T21 + ISP OSD + extended features |
| T31 | 1,203 KB | New-gen encoder API |
| T40 | 1,215 KB | Dual-sensor capable |
| T41 | 1,445 KB | T40 + NNA2 AI integration |
| T30 | 1,888 KB | Dual-sensor legacy, largest classic SDK |
| T32 | 2,180 KB | Largest — dual-sensor + new-gen encoder |

The library size roughly correlates with feature set. T30 and T32 are dual-sensor SoCs with proportionally larger libimp.

---

## 4. Additional Libraries (Not in Core SoC Directories)

### ZRT audio DSP libraries (T31 versions 1.1.3/1.1.5-zrt)

Found in T31 ZRT builds only:

| Library | Description |
|---|---|
| libagc.so | Automatic Gain Control |
| libdrc.so | Dynamic Range Compression |
| libhpf.so | High-Pass Filter |
| libns.so | Noise Suppression |

These complement `libaudioProcess.so` for advanced audio pipeline processing. Only present in ZRT (Zero-Residual-Time) SDK variants.

### ZRT IPC libraries

| Library | SoC | Description |
|---|---|---|
| libdbus.so / .a | T23 (1.1.0-zrt), T31 (1.1.5 ZRT) | D-Bus IPC (~23 KB) |
| sync.a / sync_540.a | T31 (1.1.5 ZRT) | Frame sync for ZRT |

---

## 5. Acceleration Module Libraries

Located in `ingenic-lib/acceleration-modules/`, organized by accelerator type rather than SoC.

### IVS (Intelligent Video Surveillance)

Available for gcc 4.7.2, 5.4.0, and 7.2.0 (uclibc + glibc each).

**gcc 4.7.2 / 5.4.0 (for T20/T21/T23/T30/T31):**

| Library | .so size | Description |
|---|---|---|
| libjzdl.so | ~480-490 KB | JZDL inference engine |
| libpersonDet_inf.so | ~1,072-1,080 KB | Person detection model |
| libmxu_core.so | ~713-717 KB | MXU math core |
| libmxu_imgproc.so | ~759-804 KB | MXU image processing |
| libmxu_objdetect.so | ~503-535 KB | MXU object detection |
| libmxu_contrib.so | ~404-411 KB | MXU contributed algorithms |
| libmxu_merge.so | ~244-258 KB | MXU merge operations |
| libmxu_video.so | ~63-65 KB | MXU video operations |

**gcc 7.2.0 (for T40/T41):**

IVS for the newer SoCs uses Venus (NNA1) instead of JZDL:

| Library | .so size | Description |
|---|---|---|
| libvenus.so | 5,143 KB | Venus NNA inference engine |
| libpersonvehiclepetDet_inf.so | ~213 KB | Person/vehicle/pet detection |
| libaip.so | ~49-51 KB | AI Processor interface |
| libdrivers.so | ~25-26 KB | NNA hardware drivers |
| libdrivers.m.so | ~25-26 KB | NNA drivers (multi-model) |
| libmxu_core.so | ~279 KB | MXU math core (reduced) |
| libmxu_imgproc.so | ~197 KB | MXU image processing (reduced) |
| libmxu_merge.so | ~43 KB | MXU merge |
| libmxu_video.so | ~26 KB | MXU video |

Note: MXU libraries are significantly smaller for gcc 7.2.0 — `libmxu_contrib` and `libmxu_objdetect` are absent, as their functionality moved to libvenus.

### JZDL (JZ Deep Learning)

Standalone JZDL module with `libjzdl.m.so` (multi-model variant):

| GCC | .so size | Notes |
|---|---|---|
| 4.7.2 | 624 KB | For T20/T21/T30 |
| 5.4.0 | 624 KB | For T23/T31 (also includes libiaac.so, 76 KB) |
| 7.2.0 | 806 KB | For T40/T41 |

### NNA1 (Neural Network Accelerator v1, gcc 7.2.0 only)

For T40 (single NNA core):

| Library | .so size | Description |
|---|---|---|
| libvenus.so | 10,717 KB | Venus inference engine |
| libvenus.d.so | 11,349 KB | Debug build |
| libvenus.m.so | 10,541 KB | Multi-model variant |
| libvenus.p.so | 10,822 KB | Profiling build |
| libvenus.a | 27,454 KB | Static archive |

### NNA2 (Neural Network Accelerator v2)

For T41 and A1 (dual NNA cores):

**T41 variant:**

| Library | .so size | Description |
|---|---|---|
| libvenus.so | 11,154 KB | Inference engine |
| libvenus.d.so | 11,843 KB | Debug build |
| libvenus.p.so | 11,257 KB | Profiling build |
| libaip.so | ~35 KB | AI Processor interface |
| libdrivers.so | ~27 KB | NNA hardware drivers |
| libdrivers.m.so | ~26 KB | Multi-model drivers |

**A1 variant:** Similar to T41 but with A1-specific libvenus builds. Also includes `libaip_debug.so`.

**AIPV20 variant:** For newer AI pipeline revision, includes `libaip_v20.so` instead of `libaip.so`, and optionally `libvenus_aisp.a` (AISP integration, uclibc only, 10 MB).

---

## 6. Library Availability Matrix (Core Libraries, Latest Version)

| Library | T20 | T21 | T23 | T30 | T31 | T32 | T40 | T41 |
|---|---|---|---|---|---|---|---|---|
| libimp.so | Y | Y | Y | Y | Y | Y | Y | Y |
| libimp.a | Y | Y | Y | Y | Y | Y | Y | Y |
| libalog.so | Y | Y | Y | Y | Y | Y | Y | Y |
| libalog.a | Y | Y | Y | Y | Y | Y | Y | Y |
| libsysutils.so | Y | Y | Y | Y | Y | Y | Y | Y |
| libsysutils.a | Y | Y | Y | Y | Y | Y | Y | Y |
| libaudioProcess.so | Y | Y | Y | Y | Y | Y | Y | Y |
| libaudioProcess.a | - | - | - | - | - | - | - | - |

All 8 SoCs have exactly the same 4 core libraries. `libaudioProcess` is always shared-only (no static archive).

---

## 7. GCC Version Compatibility

| GCC Version | SoCs | Notes |
|---|---|---|
| 4.7.2 | T20, T21, T30, T31 | Legacy MIPS toolchain |
| 5.4.0 | T21, T23, T30, T31 | Standard Thingino toolchain |
| 7.2.0 | T40, T41 | Required for XBurst2 ISA |

- T20 only supports gcc 4.7.2 (older MIPS ABI)
- T21 and T30 support both 4.7.2 and 5.4.0
- T23 only supports gcc 5.4.0 (no 4.7.2 builds available)
- T31 supports both 4.7.2 and 5.4.0
- T32 has no gcc version subdirectory (libraries work with the matching toolchain)
- T40 and T41 only support gcc 7.2.0 (XBurst2 architecture)

For Thingino builds:
- T20/T21/T30/T31: use gcc 5.4.0 uclibc (preferred) or 4.7.2 uclibc
- T23: use gcc 5.4.0 uclibc (only option)
- T32: use uclibc (direct, no gcc version selection)
- T40/T41: use gcc 7.2.0 uclibc (only option)

---

## 8. Mandatory vs Optional for a Streamer

### Mandatory (always required)

| Library | Purpose | Why mandatory |
|---|---|---|
| libimp.so | IMP API (video, encoder, ISP, OSD, audio) | All hardware access goes through this |
| libalog.so | Logging framework | Required by libimp at runtime |
| libsysutils.so | System utilities (thread pool, timers) | Required by libimp at runtime |

These three form the minimum deployment. Without any of them, libimp will fail to load (unresolved symbols from libalog/libsysutils).

### Optional

| Library | Purpose | When needed |
|---|---|---|
| libaudioProcess.so | Audio AEC/ANR/AGC processing | Only if audio capture is used |
| libagc/libdrc/libhpf/libns.so | ZRT audio DSP | Only for T31 ZRT audio features |
| libjzdl.so / libvenus.so | AI inference | Only if motion detection / smart analysis is used |
| libmxu_*.so | MXU accelerated compute | Only with IVS AI features |
| libaip.so / libdrivers.so | NNA hardware interface | Only with Venus/NNA AI |

### Runtime deployment (minimum streamer)

```
/usr/lib/
    libimp.so
    libalog.so
    libsysutils.so
    libaudioProcess.so   # if audio enabled
```

Total minimum .so size for a video-only streamer (uclibc):

| SoC | libimp + libalog + libsysutils |
|---|---|
| T20 | 1,089 KB |
| T21 | 959 KB |
| T23 | 1,234 KB |
| T30 | 1,958 KB |
| T31 | 1,267 KB |
| T32 | 2,270 KB |
| T40 | 1,286 KB |
| T41 | 1,515 KB |

Adding libaudioProcess increases each by ~660-750 KB.

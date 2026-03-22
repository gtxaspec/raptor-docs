# 11 -- HAL Internal Implementation Guide

This document tells an implementer **how** to write the per-SoC HAL code for the Raptor Streaming System. It covers macro definitions, file organization, and the concrete translation patterns for every area where the Ingenic SDK diverges across SoC generations.

---

## 1. SDK Generation Macros

The build system defines exactly one `PLATFORM_*` macro per target. The HAL uses these to derive generation-level macros that group SoCs by API compatibility.

```c
/* ── Per-SoC platform macros (set by build system, exactly one defined) ── */
// PLATFORM_T20  PLATFORM_T21  PLATFORM_T23  PLATFORM_T30
// PLATFORM_T31  PLATFORM_T32  PLATFORM_T40  PLATFORM_T41

/* ── SDK generation macros (derived, used by HAL internals) ── */

#if defined(PLATFORM_T20) || defined(PLATFORM_T21) || \
    defined(PLATFORM_T23) || defined(PLATFORM_T30)
  #define HAL_OLD_SDK       /* Old-style encoder structs, scalar ISP tuning */
#endif

#if defined(PLATFORM_T31) || defined(PLATFORM_T32) || \
    defined(PLATFORM_T40) || defined(PLATFORM_T41)
  #define HAL_NEW_SDK       /* New-style encoder structs, unified RC */
#endif

#if defined(PLATFORM_T40) || defined(PLATFORM_T41)
  #define HAL_IMPVI_SDK     /* ISP functions take IMPVI_NUM first param */
#endif
```

### Additional fine-grained guards used within the HAL

```c
/* T32 is a hybrid: new-style encoder internals but old-style type name */
#if defined(PLATFORM_T32)
  #define HAL_HYBRID_SDK
#endif

/* ISP tuning functions that take pointer args (not scalar) */
#if defined(PLATFORM_T32) || defined(PLATFORM_T40) || defined(PLATFORM_T41)
  #define HAL_ISP_PTR_ARGS
#endif

/* Extended OSD region types (enum values shifted) */
#if defined(PLATFORM_T23) || defined(PLATFORM_T32) || \
    defined(PLATFORM_T40) || defined(PLATFORM_T41)
  #define HAL_EXTENDED_OSD
#endif

/* Multi-sensor support via IMPVI_NUM parameter */
#if defined(PLATFORM_T32) || defined(PLATFORM_T40) || defined(PLATFORM_T41)
  #define HAL_MULTI_SENSOR
#endif
```

### Rationale

Seven derived macros cover the branching. The generation macros are NOT mutually exclusive — a SoC may have multiple macros active:

| Macro | SoCs | What it means |
|-------|------|---------------|
| `HAL_OLD_SDK` | T20, T21, T23, T30 | Old encoder structs (`IMPEncoderCHNAttr`), per-codec RC unions (`attrH264Cbr`), packs with direct `virAddr`, old RC mode enums (`ENC_RC_MODE_*`) |
| `HAL_NEW_SDK` | T31, T32, T40, T41 | Has `SetDefaultParam()`, `IMPEncoderProfile` enum. **But T32 is a hybrid** — it uses old struct names, old RC enum names, old pack access, no `gopAttr`. Always check `PLATFORM_T32` explicitly. |
| `HAL_IMPVI_SDK` | T40, T41 | ISP tuning functions take `IMPVI_NUM` as first parameter, all args become pointers |
| `HAL_HYBRID_SDK` | T32 only | New SDK `SetDefaultParam`, but old RC enum names (`ENC_RC_MODE_*` not `IMP_ENC_RC_MODE_*`), old pack access (direct `virAddr`), no `gopAttr`, old struct name `IMPEncoderCHNAttr` |
| `HAL_ISP_PTR_ARGS` | T32, T40, T41 | ISP tuning setters take pointer args (`unsigned char *bright`) not scalars |
| `HAL_EXTENDED_OSD` | T23, T32, T40, T41 | OSD enum values shifted (BITMAP=5 not 3, COVER=6 not 4, PIC=7 not 5) |
| `HAL_MULTI_SENSOR` | T32, T40, T41 | `IMP_ISP_AddSensor(IMPVI_NUM, ...)`, extended `IMPSensorInfo` with `sensor_id`, `video_interface`, `mclk` |

**Key nuance: T32 is the hardest SoC to handle.** It has `HAL_NEW_SDK` set (for `SetDefaultParam`), but its encoder and pack access behave like old SDK. The code uses `!defined(PLATFORM_T32)` guards extensively within `HAL_NEW_SDK` blocks. When adding new encoder code, always test T32 separately.

Per-SoC macro matrix:

| Macro | T20 | T21 | T23 | T30 | T31 | T32 | T40 | T41 |
|-------|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| `HAL_OLD_SDK` | Y | Y | Y | Y | | | | |
| `HAL_NEW_SDK` | | | | | Y | Y | Y | Y |
| `HAL_IMPVI_SDK` | | | | | | | Y | Y |
| `HAL_HYBRID_SDK` | | | | | | Y | | |
| `HAL_ISP_PTR_ARGS` | | | | | | Y | Y | Y |
| `HAL_EXTENDED_OSD` | | | Y | | | Y | Y | Y |
| `HAL_MULTI_SENSOR` | | | | | | Y | Y | Y |

---

## 2. File Organization

### Option A: Per-SoC files

```
hal_t20.c  hal_t21.c  hal_t23.c  hal_t30.c
hal_t31.c  hal_t32.c  hal_t40.c  hal_t41.c
```

Each file implements the full HAL for one SoC. Clean separation, but 80%+ of the code is identical across files (audio init, OSD workflow, frame polling). Any bug fix or feature addition must be replicated 8 times.

### Option B: Per-SDK-generation files (Recommended)

```
hal_common.c          # Shared logic: error handling, caps query, audio init,
                      # OSD group management, frame polling loop
hal_encoder_old.c     # Old SDK encoder: channel create, stream pack access
hal_encoder_new.c     # New SDK encoder: channel create, stream pack access, ring-buffer
hal_isp_gen1.c        # T20/T21/T30: scalar ISP tuning, separate H/V flip
hal_isp_gen2.c        # T23/T31: scalar ISP tuning, combined HVFLIP, _Sec suffix
hal_isp_gen3.c        # T32/T40/T41: IMPVI_NUM + pointer ISP tuning
hal_osd.c             # OSD region management, enum translation per generation
hal_sensor.c          # IMPSensorInfo construction, AddSensor dispatch
hal_caps.c            # Per-SoC capability structs
```

Only one encoder file and one ISP file are compiled per target (selected by the build system based on `PLATFORM_*`). `hal_common.c` is always compiled. This gives clean separation without duplication: shared code lives in `hal_common.c`, and generation-specific code lives in the appropriate file.

### Option C: Single file with `#ifdef` blocks

```
hal.c                 # Everything in one file, massive #if/#elif chains
```

Most compact. This is what prudynt-t uses (`imp_hal.cpp`). It works for a 900-line file but becomes unmaintainable at HAL scale (2000+ lines). Every function has nested `#ifdef` blocks that obscure the logic. Difficult to review or test a single generation in isolation.

### Recommendation

**Use Option B.** The generation split is natural: the Old/New encoder boundary is the deepest API break, and the ISP three-generation split is the widest. Per-generation files keep each file under 500 lines and allow reviewers to focus on one SDK variant at a time. The build system selects the correct files:

```makefile
# Build system file selection (example)
ifeq ($(HAL_OLD_SDK),1)
  HAL_SRCS += hal_encoder_old.c
else
  HAL_SRCS += hal_encoder_new.c
endif

ifeq ($(PLATFORM),T20)$(PLATFORM),T21)$(PLATFORM),T30))
  HAL_SRCS += hal_isp_gen1.c
else ifeq ($(PLATFORM),T23)$(PLATFORM),T31))
  HAL_SRCS += hal_isp_gen2.c
else
  HAL_SRCS += hal_isp_gen3.c
endif

HAL_SRCS += hal_common.c hal_osd.c hal_sensor.c hal_caps.c
```

---

## 3. Encoder Channel Creation

This is the most complex translation in the HAL. The channel attribute structs are completely different between Old and New SDK.

### 3.1 Old SDK Channel Creation (T20, T21, T23, T30)

```c
int hal_encoder_create_channel(int chn, const rss_encoder_config_t *cfg) {
    IMPEncoderCHNAttr chnAttr;
    memset(&chnAttr, 0, sizeof(chnAttr));

    /* Encoder attributes */
    chnAttr.encAttr.enType    = cfg->payload_type;    /* PT_H264, PT_JPEG, PT_H265 */
    chnAttr.encAttr.bufSize   = cfg->buf_size;        /* 0 = auto */
    chnAttr.encAttr.profile   = cfg->profile;         /* 0=baseline, 1=main, 2=high */
    chnAttr.encAttr.picWidth  = cfg->width;
    chnAttr.encAttr.picHeight = cfg->height;

    /* Rate control -- per-codec union */
    chnAttr.rcAttr.outFrmRate.frmRateNum = cfg->fps_num;
    chnAttr.rcAttr.outFrmRate.frmRateDen = cfg->fps_den;
    chnAttr.rcAttr.maxGop = cfg->gop;

    switch (cfg->rc_mode) {
    case RSS_RC_CBR:
        chnAttr.rcAttr.attrRcMode.rcMode = ENC_RC_MODE_CBR;
        if (cfg->payload_type == PT_H264) {
            chnAttr.rcAttr.attrRcMode.attrH264Cbr.outBitRate = cfg->bitrate;
            chnAttr.rcAttr.attrRcMode.attrH264Cbr.maxQp = cfg->max_qp;
            chnAttr.rcAttr.attrRcMode.attrH264Cbr.minQp = cfg->min_qp;
        }
        break;
    case RSS_RC_VBR:
        chnAttr.rcAttr.attrRcMode.rcMode = ENC_RC_MODE_VBR;
        /* ... fill attrH264Vbr / attrH265Vbr ... */
        break;
    case RSS_RC_SMART:
        chnAttr.rcAttr.attrRcMode.rcMode = ENC_RC_MODE_SMART;
        /* ... fill attrH264Smart / attrH265Smart ... */
        break;
    }

    return IMP_Encoder_CreateChn(chn, &chnAttr);
}
```

### 3.2 New SDK Channel Creation (T31, T40, T41)

```c
int hal_encoder_create_channel(int chn, const rss_encoder_config_t *cfg) {
    IMPEncoderChnAttr chnAttr;
    memset(&chnAttr, 0, sizeof(chnAttr));

    /* Use SetDefaultParam to pre-fill sensible defaults */
    IMPEncoderProfile profile = hal_translate_profile(cfg->codec, cfg->profile);
    IMPEncoderRcMode rc = hal_translate_rc_mode(cfg->rc_mode);

    IMP_Encoder_SetDefaultParam(
        &chnAttr, profile, rc,
        cfg->width, cfg->height,
        cfg->fps_num, cfg->fps_den,
        cfg->gop,
        cfg->max_same_scene_count,  /* 2 on T31/T40/T41 */
        cfg->initial_qp,
        cfg->bitrate
    );

    /* Override specific fields as needed */
    chnAttr.rcAttr.attrRcMode.attrCbr.iMinQP = cfg->min_qp;
    chnAttr.rcAttr.attrRcMode.attrCbr.iMaxQP = cfg->max_qp;

    /* GOP attributes (separate struct on new SDK) */
    chnAttr.gopAttr.uGopCtrlMode = IMP_ENC_GOP_CTRL_MODE_DEFAULT;
    chnAttr.gopAttr.uGopLength   = cfg->gop;

    return IMP_Encoder_CreateChn(chn, &chnAttr);
}
```

### 3.3 T32 Hybrid Path

T32 uses the old type name `IMPEncoderCHNAttr` but the new internal struct layout (`IMPEncoderEncAttr`, `IMPEncoderGopAttr`). Its `SetDefaultParam` takes `uBufSize` instead of `uMaxSameSenceCnt`:

```c
#if defined(PLATFORM_T32)
    IMP_Encoder_SetDefaultParam(
        &chnAttr, profile, rc,
        cfg->width, cfg->height,
        cfg->fps_num, cfg->fps_den,
        cfg->gop,
        cfg->buf_size,           /* uBufSize -- T32 only */
        cfg->initial_qp,
        cfg->bitrate
    );
#endif
```

### 3.4 Struct Field Mapping

| Old SDK Field | New SDK Field | Notes |
|---|---|---|
| `encAttr.enType` (IMPPayloadType) | `encAttr.eProfile` (IMPEncoderProfile) | Composite: `(type << 24) \| profile_idc` |
| `encAttr.picWidth` (uint32_t) | `encAttr.uWidth` (uint16_t) | Width narrowed |
| `encAttr.picHeight` (uint32_t) | `encAttr.uHeight` (uint16_t) | Height narrowed |
| `encAttr.bufSize` (uint32_t) | Absent (T31/T40), present (T41) | T41 added it back |
| `encAttr.profile` (uint32_t) | Embedded in `eProfile` | See profile encoding |
| `rcAttr.outFrmRate` (first field) | `rcAttr.outFrmRate` (second field) | Order swapped in new SDK |
| `rcAttr.maxGop` | `gopAttr.uGopLength` | Moved to separate struct |
| `ENC_RC_MODE_CBR` (=1) | `IMP_ENC_RC_MODE_CBR` (=0x1) | Same value, different name |
| `ENC_RC_MODE_VBR` (=2) | `IMP_ENC_RC_MODE_VBR` (=0x2) | Same value, different name |
| `ENC_RC_MODE_SMART` (=3) | N/A | Replaced by `CAPPED_VBR` (0x4) / `CAPPED_QUALITY` (0x8) |
| Per-codec RC: `attrH264Cbr` | Unified: `attrCbr` | Codec-agnostic on new SDK |

### 3.5 RC Mode Translation

```c
static IMPEncoderRcMode hal_translate_rc_mode(rss_rc_mode_t mode) {
#if defined(HAL_NEW_SDK)
    switch (mode) {
    case RSS_RC_FIXQP:  return IMP_ENC_RC_MODE_FIXQP;
    case RSS_RC_CBR:    return IMP_ENC_RC_MODE_CBR;
    case RSS_RC_VBR:    return IMP_ENC_RC_MODE_VBR;
    case RSS_RC_SMART:  return IMP_ENC_RC_MODE_CAPPED_VBR;   /* best equivalent */
    default:            return IMP_ENC_RC_MODE_VBR;
    }
#else
    switch (mode) {
    case RSS_RC_FIXQP:  return ENC_RC_MODE_FIXQP;
    case RSS_RC_CBR:    return ENC_RC_MODE_CBR;
    case RSS_RC_VBR:    return ENC_RC_MODE_VBR;
    case RSS_RC_SMART:  return ENC_RC_MODE_SMART;
    default:            return ENC_RC_MODE_CBR;
    }
#endif
}
```

### 3.6 Profile Translation

```c
static IMPEncoderProfile hal_translate_profile(rss_codec_t codec, int profile) {
#if defined(HAL_NEW_SDK)
    switch (codec) {
    case RSS_CODEC_H264:
        if (profile == 0) return IMP_ENC_PROFILE_AVC_BASELINE;
        if (profile == 1) return IMP_ENC_PROFILE_AVC_MAIN;
        return IMP_ENC_PROFILE_AVC_HIGH;
    case RSS_CODEC_H265:
        return IMP_ENC_PROFILE_HEVC_MAIN;
    case RSS_CODEC_JPEG:
        return IMP_ENC_PROFILE_JPEG;
    }
#else
    /* Old SDK uses enType + profile integer separately */
    /* This function is not used on old SDK path */
    (void)codec; (void)profile;
    return 0;
#endif
}
```

### 3.7 GOP Attribute Setup (New SDK Only)

On old SDK, GOP is set via `rcAttr.maxGop` (a simple integer). On new SDK, it is a separate struct:

```c
#if defined(HAL_NEW_SDK)
static void hal_set_gop_attr(IMPEncoderGopAttr *gop, int gop_length) {
    gop->uGopCtrlMode  = IMP_ENC_GOP_CTRL_MODE_DEFAULT;  /* 0x02 */
    gop->uGopLength    = gop_length;
    gop->uNumB         = 0;
    gop->bEnableLT     = false;
    gop->uFreqLT       = 0;
    gop->bLTRC         = false;
    gop->uMaxSameSenceCnt = 2;
}
#endif
```

---

## 4. Stream Pack Abstraction

The data access pattern for encoded frames is fundamentally different between the two SDK generations. This is the most critical correctness concern in the HAL.

### 4.1 Old SDK Pack Access (T20, T21, T23, T30)

Each pack carries its own virtual address. Data is contiguous.

```c
static int hal_fill_frame_old(const IMPEncoderStream *stream, rss_frame_t *frame) {
    uint8_t *dst = frame->data;
    uint32_t total = 0;

    for (uint32_t i = 0; i < stream->packCount; i++) {
        uint8_t *src = (uint8_t *)stream->pack[i].virAddr;
        uint32_t len = stream->pack[i].length;

        memcpy(dst, src, len);
        dst += len;
        total += len;
    }

    frame->size = total;
    frame->timestamp = stream->pack[0].timestamp;

    /* NAL type for keyframe detection */
    frame->nal_type = stream->pack[0].dataType.h264Type;
    return 0;
}
```

### 4.2 New SDK Pack Access (T31, T32, T40, T41)

Packs have an `offset` into a shared stream buffer. The buffer is a ring buffer and data can wrap around `streamSize`.

```c
static int hal_fill_frame_new(const IMPEncoderStream *stream, rss_frame_t *frame) {
    uint8_t *dst = frame->data;
    uint8_t *base = (uint8_t *)stream->virAddr;
    uint32_t buf_size = stream->streamSize;
    uint32_t total = 0;

    for (uint32_t i = 0; i < stream->packCount; i++) {
        uint32_t off = stream->pack[i].offset;
        uint32_t len = stream->pack[i].length;

        if (off + len <= buf_size) {
            /* Contiguous -- no wrap */
            memcpy(dst, base + off, len);
        } else {
            /* Ring-buffer wrap: copy in two parts */
            uint32_t part1 = buf_size - off;
            uint32_t part2 = len - part1;
            memcpy(dst, base + off, part1);
            memcpy(dst + part1, base, part2);
        }

        dst += len;
        total += len;
    }

    frame->size = total;
    frame->timestamp = stream->pack[0].timestamp;

    /* NAL type -- field name changed */
    frame->nal_type = stream->pack[0].nalType.h264NalType;
    return 0;
}
```

### 4.3 Unified HAL Entry Point

```c
int hal_encoder_get_frame(int chn, rss_frame_t *frame) {
    IMPEncoderStream stream;
    int ret;

    ret = IMP_Encoder_PollingStream(chn, 1000);
    if (ret < 0) return ret;

    ret = IMP_Encoder_GetStream(chn, &stream, 1);
    if (ret < 0) return ret;

#if defined(HAL_OLD_SDK)
    ret = hal_fill_frame_old(&stream, frame);
#else
    ret = hal_fill_frame_new(&stream, frame);
#endif

    IMP_Encoder_ReleaseStream(chn, &stream);
    return ret;
}
```

### 4.4 NAL Type Field Mapping

| Aspect | Old SDK | New SDK |
|---|---|---|
| Union type | `IMPEncoderDataType` | `IMPEncoderNalType` |
| H.264 member | `pack.dataType.h264Type` | `pack.nalType.h264NalType` |
| H.265 member | `pack.dataType.h265Type` | `pack.nalType.h265NalType` |
| Slice type | Not available | `pack.sliceType` |
| IDR enum value | `IMP_H264_NAL_SLICE_IDR` | `IMP_H264_NAL_SLICE_IDR` (same) |

The NAL enum values are identical across all SoCs. Only the struct access path changes.

---

## 5. ISP Tuning Dispatch

ISP tuning functions have a three-generation signature evolution. This is the widest branching point in the HAL.

### 5.1 The Three Signatures

**Gen1 (T20, T21, T30):** Scalar value, no IMPVI_NUM.
```c
int IMP_ISP_Tuning_SetBrightness(unsigned char bright);
int IMP_ISP_Tuning_GetBrightness(unsigned char *pbright);
```

**Gen2 (T23, T31):** Still scalar, no IMPVI_NUM. Functionally identical to Gen1 for basic tuning (brightness, contrast, etc.). Some functions are renamed (e.g., `SetHVFLIP` replaces `SetISPHflip` + `SetISPVflip`).
```c
int IMP_ISP_Tuning_SetBrightness(unsigned char bright);   /* same as Gen1 */
int IMP_ISP_Tuning_GetBrightness(unsigned char *pbright);
```

**Gen3 (T32, T40, T41):** IMPVI_NUM + pointer argument.
```c
int32_t IMP_ISP_Tuning_SetBrightness(IMPVI_NUM num, unsigned char *bright);
int32_t IMP_ISP_Tuning_GetBrightness(IMPVI_NUM num, unsigned char *pbright);
```

### 5.2 HAL Wrapper Pattern

Every basic tuning function follows this exact pattern:

```c
int hal_isp_set_brightness(int vi_num, unsigned char val) {
#if defined(PLATFORM_T32) || defined(PLATFORM_T40) || defined(PLATFORM_T41)
    /* Gen3: IMPVI_NUM + pointer */
    return IMP_ISP_Tuning_SetBrightness((IMPVI_NUM)vi_num, &val);
#else
    /* Gen1/Gen2: scalar, ignore vi_num */
    (void)vi_num;
    return IMP_ISP_Tuning_SetBrightness(val);
#endif
}

int hal_isp_get_brightness(int vi_num, unsigned char *val) {
#if defined(PLATFORM_T32) || defined(PLATFORM_T40) || defined(PLATFORM_T41)
    return IMP_ISP_Tuning_GetBrightness((IMPVI_NUM)vi_num, val);
#else
    (void)vi_num;
    return IMP_ISP_Tuning_GetBrightness(val);
#endif
}
```

This pattern applies identically to: `SetContrast`, `SetSaturation`, `SetSharpness`, `SetBcshHue`.

### 5.3 Hue Special Case

`SetBcshHue` is only available on T23, T31, T32, T40, T41 (not T20, T21, T30):

```c
int hal_isp_set_hue(int vi_num, unsigned char val) {
#if defined(PLATFORM_T32) || defined(PLATFORM_T40) || defined(PLATFORM_T41)
    return IMP_ISP_Tuning_SetBcshHue((IMPVI_NUM)vi_num, &val);
#elif defined(PLATFORM_T23) || defined(PLATFORM_T31)
    (void)vi_num;
    return IMP_ISP_Tuning_SetBcshHue(val);
#else
    (void)vi_num; (void)val;
    return -1;  /* not supported on T20/T21/T30 */
#endif
}
```

### 5.4 Flip/Mirror Dispatch

The flip API has the most variation of any ISP function across generations:

```c
int hal_isp_set_flip(int vi_num, int hflip, int vflip) {
    int mode;
    if (hflip && vflip)      mode = 3;  /* HV */
    else if (hflip)          mode = 1;  /* H only */
    else if (vflip)          mode = 2;  /* V only */
    else                     mode = 0;  /* normal */

#if defined(PLATFORM_T20) || defined(PLATFORM_T30)
    /* Separate H + V + combined HVflip */
    (void)vi_num;
    IMPISPTuningOpsMode h = hflip ? IMPISP_TUNING_OPS_MODE_ENABLE
                                  : IMPISP_TUNING_OPS_MODE_DISABLE;
    IMPISPTuningOpsMode v = vflip ? IMPISP_TUNING_OPS_MODE_ENABLE
                                  : IMPISP_TUNING_OPS_MODE_DISABLE;
    IMP_ISP_Tuning_SetISPHflip(h);
    return IMP_ISP_Tuning_SetISPVflip(v);

#elif defined(PLATFORM_T21)
    /* Separate H + V, no combined function */
    (void)vi_num;
    IMPISPTuningOpsMode h = hflip ? IMPISP_TUNING_OPS_MODE_ENABLE
                                  : IMPISP_TUNING_OPS_MODE_DISABLE;
    IMPISPTuningOpsMode v = vflip ? IMPISP_TUNING_OPS_MODE_ENABLE
                                  : IMPISP_TUNING_OPS_MODE_DISABLE;
    int ret = IMP_ISP_Tuning_SetISPHflip(h);
    if (ret) return ret;
    return IMP_ISP_Tuning_SetISPVflip(v);

#elif defined(PLATFORM_T23) || defined(PLATFORM_T31)
    /* Combined HVFLIP with IMPISPHVFLIP enum */
    (void)vi_num;
    IMPISPHVFLIP hvflip = (IMPISPHVFLIP)mode;
    return IMP_ISP_Tuning_SetHVFLIP(hvflip);

#elif defined(PLATFORM_T40)
    /* IMPVI_NUM + IMPISPHVFLIP pointer */
    IMPISPHVFLIP hvflip = (IMPISPHVFLIP)mode;
    return IMP_ISP_Tuning_SetHVFLIP((IMPVI_NUM)vi_num, &hvflip);

#elif defined(PLATFORM_T32) || defined(PLATFORM_T41)
    /* IMPVI_NUM + IMPISPHVFLIPAttr struct pointer */
    IMPISPHVFLIPAttr attr = {0};
    attr.sensor_mode = (IMPISPHVFLIP)mode;
    attr.isp_mode[0] = (IMPISPHVFLIP)mode;
    return IMP_ISP_Tuning_SetHVFLIP((IMPVI_NUM)vi_num, &attr);
#endif
}
```

### 5.5 Sensor FPS Dispatch

```c
int hal_isp_set_fps(int vi_num, uint32_t fps_num, uint32_t fps_den) {
#if defined(PLATFORM_T20) || defined(PLATFORM_T21) || \
    defined(PLATFORM_T23) || defined(PLATFORM_T30) || defined(PLATFORM_T31)
    (void)vi_num;
    return IMP_ISP_Tuning_SetSensorFPS(fps_num, fps_den);

#elif defined(PLATFORM_T40)
    return IMP_ISP_Tuning_SetSensorFPS((IMPVI_NUM)vi_num, &fps_num, &fps_den);

#elif defined(PLATFORM_T32) || defined(PLATFORM_T41)
    IMPISPSensorFps fps = { .num = fps_num, .den = fps_den };
    return IMP_ISP_Tuning_SetSensorFPS((IMPVI_NUM)vi_num, &fps);
#endif
}
```

---

## 6. Sensor Init Sequence

### 6.1 IMPSensorInfo Construction

The sensor info struct grows across generations. The HAL abstracts this with a common config struct:

```c
typedef struct {
    const char *name;           /* sensor driver name */
    uint16_t    i2c_addr;       /* 7-bit I2C address */
    int         i2c_adapter;    /* I2C adapter number */
    uint16_t    sensor_id;      /* T23/T32/T40/T41 only */
    int         rst_gpio;       /* reset GPIO */
    int         pwdn_gpio;      /* powerdown GPIO */
    int         vin_type;       /* MIPI CSI0/CSI1/DVP -- T32/T40/T41 */
    int         mclk;           /* MCLK0/MCLK1/MCLK2 -- T32/T40/T41 */
    int         default_boot;   /* T32/T40/T41 */
} hal_sensor_config_t;
```

Translation to `IMPSensorInfo`:

```c
static void hal_fill_sensor_info(IMPSensorInfo *info, const hal_sensor_config_t *cfg) {
    memset(info, 0, sizeof(*info));
    strncpy(info->name, cfg->name, sizeof(info->name) - 1);
    info->cbus_type = TX_SENSOR_CONTROL_INTERFACE_I2C;
    strncpy(info->i2c.type, cfg->name, sizeof(info->i2c.type) - 1);
    info->i2c.addr = cfg->i2c_addr;
    info->i2c.i2c_adapter_id = cfg->i2c_adapter;

#if defined(PLATFORM_T23)
    info->sensor_id = cfg->sensor_id;
#endif

#if defined(PLATFORM_T32) || defined(PLATFORM_T40) || defined(PLATFORM_T41)
    info->rst_gpio   = cfg->rst_gpio;       /* int on T32+ */
    info->pwdn_gpio  = cfg->pwdn_gpio;
    info->sensor_id  = cfg->sensor_id;
    info->video_interface = (IMPSensorVinType)cfg->vin_type;
    info->mclk       = (IMPSensorMclk)cfg->mclk;
    info->default_boot = cfg->default_boot;
#else
    info->rst_gpio   = (unsigned short)cfg->rst_gpio;   /* unsigned short on T20-T31 */
    info->pwdn_gpio  = (unsigned short)cfg->pwdn_gpio;
#endif
}
```

### 6.2 Init Ordering

The ISP and System init sequence is the same across all SoCs, but `AddSensor` signatures differ:

```
1. IMP_ISP_Open()
2. hal_isp_add_sensor(vi_num, cfg)      -- fills IMPSensorInfo, calls IMP_ISP_AddSensor
3. IMP_ISP_EnableSensor(...)            -- IMPVI_NUM on T32/T40/T41
4. IMP_System_Init()
5. IMP_ISP_EnableTuning()               -- must be after System_Init
```

### 6.3 AddSensor Signature Differences

```c
int hal_isp_add_sensor(int vi_num, const hal_sensor_config_t *cfg) {
    IMPSensorInfo info;
    hal_fill_sensor_info(&info, cfg);

#if defined(PLATFORM_T32) || defined(PLATFORM_T40) || defined(PLATFORM_T41)
    return IMP_ISP_AddSensor((IMPVI_NUM)vi_num, &info);
#else
    (void)vi_num;
    return IMP_ISP_AddSensor(&info);
#endif
}
```

Similarly for `EnableSensor`, `DisableSensor`, `DelSensor`.

---

## 7. OSD Enum Value Mapping

The OSD region type enum undergoes a value shift between classic and extended SDK. This is a silent data corruption risk if the wrong values are used.

### 7.1 Enum Values

| Region Type | Classic (T20/T21/T30) | T31 | Extended (T23/T32/T40/T41) |
|---|---|---|---|
| INV | 0 | 0 | 0 |
| LINE | 1 | 1 | HORIZONTAL=1, VERTICAL=2, SLASH=9 |
| RECT | 2 | 2 | 3 |
| BITMAP | **3** | **3** | **5** |
| COVER | **4** | **4** | **6** |
| PIC | **5** | **5** | **7** |
| PIC_RMEM | N/A | 6 | 8 |
| FOUR_CORNER_RECT | N/A | N/A | 4 |
| ISP types | N/A | N/A | 10, 11, 12 |
| MOSAIC | N/A | N/A | 13 |

### 7.2 HAL Translation

```c
static IMPOsdRgnType hal_translate_osd_type(rss_osd_type_t type) {
    switch (type) {
    case RSS_OSD_PIC:
#if defined(HAL_EXTENDED_OSD)
        return OSD_REG_PIC;         /* value 7 on extended SDK */
#else
        return OSD_REG_PIC;         /* value 5 on classic/T31 SDK */
#endif
        /* Both use the OSD_REG_PIC symbol but the numeric value differs.
         * The HAL must use the symbol, not a hardcoded integer. */

    case RSS_OSD_COVER:
        return OSD_REG_COVER;       /* value 4 classic, 6 extended */

    case RSS_OSD_RECT:
        return OSD_REG_RECT;        /* value 2 classic, 3 extended */

    case RSS_OSD_MOSAIC:
#if defined(HAL_EXTENDED_OSD)
        return OSD_REG_MOSAIC;      /* value 13, extended only */
#else
        return (IMPOsdRgnType)-1;   /* not supported */
#endif

    default:
        return OSD_REG_INV;
    }
}
```

The key safety rule: **always use the enum symbol, never a hardcoded integer.** The compiler resolves the correct value per SDK.

### 7.3 ISP OSD Signature Difference

`IMP_OSD_SetRgnAttr_ISP` has a different signature on T23 vs T32/T40/T41:

```c
/* T23:          (prAttr, bosdshow)                        */
/* T32/T40/T41:  (sensornum, prAttr, bosdshow)             */
```

The HAL must branch on this:

```c
int hal_osd_set_isp_attr(int sensor, IMPOSDRgnAttr *attr, int show) {
#if defined(PLATFORM_T23)
    (void)sensor;
    return IMP_OSD_SetRgnAttr_ISP(attr, show);
#elif defined(PLATFORM_T32) || defined(PLATFORM_T40) || defined(PLATFORM_T41)
    return IMP_OSD_SetRgnAttr_ISP(sensor, attr, show);
#else
    (void)sensor; (void)attr; (void)show;
    return -1;  /* ISP OSD not available */
#endif
}
```

---

## 8. Audio IChnParam

The `IMPAudioIChnParam` struct has a variant across SoCs. The `aecChn` field is present on T23, T32, T40, T41 only.

### 8.1 Struct Layout

**T20, T21, T30, T31 (8 bytes):**
```c
typedef struct {
    int usrFrmDepth;    /* frame buffer depth */
    int Rev;            /* reserved */
} IMPAudioIChnParam;
```

**T23, T32, T40, T41 (12 bytes):**
```c
typedef struct {
    int usrFrmDepth;
    IMPAudioAecChn aecChn;   /* AEC channel select */
    int Rev;
} IMPAudioIChnParam;
```

### 8.2 HAL Initialization

```c
void hal_audio_init_chn_param(IMPAudioIChnParam *param) {
    memset(param, 0, sizeof(*param));
    param->usrFrmDepth = 20;

#if defined(PLATFORM_T23) || defined(PLATFORM_T32) || \
    defined(PLATFORM_T40) || defined(PLATFORM_T41)
    param->aecChn = AUDIO_AEC_CHANNEL_FIRST_LEFT;
#endif
}
```

Since the struct size differs, a single binary cannot work across both variants. This enforces compile-time selection.

---

## 9. Error Handling Pattern

### 9.1 Return Code Convention

All HAL functions return `int`:
- `0` = success
- `-1` = general failure / not supported on this platform
- Positive values are reserved for future partial-success indicators

The HAL wraps all SDK calls and normalizes error codes:

```c
#define HAL_CHECK(call, label) do { \
    int _ret = (call); \
    if (_ret != 0) { \
        HAL_LOG_ERROR("%s failed: %d", #call, _ret); \
        return -1; \
    } \
} while (0)
```

### 9.2 Unsupported Feature Pattern

When a function is not available on the current platform, the HAL returns `-1` without calling any SDK function. The caller checks capabilities via `rss_hal_caps()` before calling optional functions:

```c
int hal_isp_set_defog(int vi_num, uint8_t strength) {
#if defined(PLATFORM_T23) || defined(PLATFORM_T31)
    (void)vi_num;
    IMPISPTuningOpsMode mode = strength > 0
        ? IMPISP_TUNING_OPS_MODE_ENABLE : IMPISP_TUNING_OPS_MODE_DISABLE;
    return IMP_ISP_Tuning_EnableDefog(mode);
#else
    (void)vi_num; (void)strength;
    return -1;
#endif
}
```

### 9.3 Logging

The HAL uses a single log macro that includes function name and line:

```c
#define HAL_LOG_ERROR(fmt, ...) \
    fprintf(stderr, "[HAL ERROR] %s:%d: " fmt "\n", __func__, __LINE__, ##__VA_ARGS__)

#define HAL_LOG_DEBUG(fmt, ...) \
    fprintf(stderr, "[HAL DEBUG] %s:%d: " fmt "\n", __func__, __LINE__, ##__VA_ARGS__)
```

---

## 10. Extern "C" Stubs

Some functions exist in the SDK shared libraries (`.so`) but are not declared in the corresponding headers. The HAL must provide `extern "C"` declarations for these. This is required when linking against a specific SDK version where the header was incomplete.

### 10.1 Known Missing Declarations

```c
extern "C" {

/* IMP_OSD_SetPoolSize is present in libimp.so on all SoCs
 * but missing from some SDK header versions */
int IMP_OSD_SetPoolSize(int size);

/* IMP_ISP_Tuning_GetAwbHist is present in libimp.so on T20-T31
 * but the header prototype is absent on some versions */
#if defined(PLATFORM_T20) || defined(PLATFORM_T21) || defined(PLATFORM_T23) || \
    defined(PLATFORM_T30) || defined(PLATFORM_T31)
int IMP_ISP_Tuning_GetAwbHist(IMPISPAWBHist *awb_hist);
#endif

}
```

### 10.2 Stub Types for Missing ISP Structs

Some ISP attribute structs are referenced in headers but not defined on certain SoCs:

```c
/* Must be defined BEFORE including SDK headers */
#if defined(PLATFORM_T20) || defined(PLATFORM_T21) || \
    defined(PLATFORM_T30) || defined(PLATFORM_T40) || defined(PLATFORM_T41)
struct IMPISPAEAttr {};    /* not defined on these SoCs */
#endif

#if defined(PLATFORM_T40) || defined(PLATFORM_T41)
struct IMPISPEVAttr {};    /* not defined on T40/T41 */
#endif
```

These are zero-size stubs that allow code to compile without `#ifdef` around every usage of these types. Functions that use them return `-1` on unsupported platforms.

### 10.3 Type Name Normalization

The HAL uses `#define` to normalize type names across SDK versions:

```c
#if defined(HAL_NEW_SDK)
  /* New SDK uses mixed-case Chn */
  #define IMPEncoderCHNAttr  IMPEncoderChnAttr
  #define IMPEncoderCHNStat  IMPEncoderChnStat
#endif
```

This allows HAL code to consistently use the old-style capitalization while the preprocessor substitutes the correct type.

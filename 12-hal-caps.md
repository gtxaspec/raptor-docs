# 12 -- HAL Capability Struct and Per-SoC Values

This document defines the complete `rss_hal_caps_t` struct and provides the exact capability values for each of the 8 supported Ingenic SoCs. All values are derived from the SDK difference analysis in documents 01 through 08.

---

## 1. Complete `rss_hal_caps_t` Definition

```c
#include <stdbool.h>
#include <stdint.h>

typedef struct {
    /* ── System Info ── */
    const char *soc_name;
    const char *sdk_version;
    int max_fs_channels;
    int max_enc_channels;
    int max_osd_groups;
    int max_osd_regions;

    /* ── Encoder Capabilities ── */
    bool has_h265;                /* H.265/HEVC encoding support */
    bool has_rotation;            /* FrameSource channel rotation (hardware) */
    bool has_i2d;                 /* I2D (Image 2D transformation) engine */
    bool has_bufshare;            /* JPEG/video buffer sharing (SetbufshareChn) */
    bool has_set_default_param;   /* IMP_Encoder_SetDefaultParam convenience fn */
    bool has_capped_rc;           /* IMP_ENC_RC_MODE_CAPPED_VBR / CAPPED_QUALITY */
    bool has_smart_rc;            /* ENC_RC_MODE_SMART (old SDK only) */
    bool has_gop_attr;            /* Separate IMPEncoderGopAttr struct */
    bool has_set_bitrate;         /* IMP_Encoder_SetChnBitRate (dynamic) */
    bool has_stream_buf_size;     /* IMP_Encoder_SetStreamBufSize */
    bool has_encoder_pool;        /* IMP_Encoder_SetPool / GetPool */
    bool has_smartp_gop;          /* SmartP GOP mode */
    bool has_rc_options;          /* Extended RC option flags */
    bool has_pskip;               /* P-skip control */
    bool has_srd;                 /* Scene reference detection */
    bool has_max_pic_size;        /* Max picture size control */
    bool has_super_frame;         /* Super frame handling */
    bool has_color2grey;          /* Color to greyscale conversion */
    bool has_roi;                 /* Region of interest encoding */
    bool has_map_roi;             /* ROI map mode */
    bool has_qp_bounds_per_frame; /* Per-frame QP bounds */
    bool has_qpg_mode;            /* QP group mode */
    bool has_qpg_ai;              /* AI-driven QP groups */
    bool has_mbrc;                /* Macroblock-level rate control */
    bool has_enc_denoise;         /* Encoder-integrated denoising */
    bool has_gdr;                 /* Gradual decoder refresh */
    bool has_sei_userdata;        /* SEI user data insertion */
    bool has_h264_vui;            /* H.264 VUI parameters */
    bool has_h265_vui;            /* H.265 VUI parameters */
    bool has_h264_trans;          /* H.264 transform controls */
    bool has_h265_trans;          /* H.265 transform controls */
    bool has_enc_crop;            /* Encoder output cropping */
    bool has_eval_info;           /* Encoder evaluation info */
    bool has_poll_module;         /* IMP_Encoder_PollingModuleStream */
    bool has_resize_mode;         /* Channel resize mode */
    bool has_jpeg_ql;             /* JPEG quality level control */
    bool has_jpeg_qp;             /* JPEG QP control */

    /* ── ISP Capabilities ── */
    bool has_multi_sensor;        /* IMPVI_NUM support (dual/multi sensor) */
    int  max_sensors;             /* 1 for T20-T31, 3 for T23-1.3.0/T32/T40/T41 */
    bool has_t23_multicam_api;    /* T23 1.3.0 IMP_ISP_MultiCamera_* functions */
    bool has_defog;               /* IMP_ISP_Tuning_EnableDefog */
    bool has_dpc;                 /* IMP_ISP_Tuning_SetDPC_Strength */
    bool has_drc;                 /* DRC (Dynamic Range Compression) */
    bool has_face_ae;             /* Face-aware AE/AWB */
    bool has_bcsh_hue;            /* IMP_ISP_Tuning_SetBcshHue */
    bool has_sinter;              /* Sinter denoising control */
    bool has_temper;              /* Temper denoising control */
    bool has_highlight_depress;   /* IMP_ISP_Tuning_SetHiLightDepress */
    bool has_backlight_comp;      /* IMP_ISP_Tuning_SetBacklightComp */
    bool has_ae_comp;             /* IMP_ISP_Tuning_SetAeComp */
    bool has_max_gain;            /* IMP_ISP_Tuning_SetMaxAgain/SetMaxDgain */
    bool has_switch_bin;          /* IMP_ISP_Tuning_SwitchBin */
    bool has_gamma;               /* IMP_ISP_Tuning_SetGamma (legacy) */
    bool has_gamma_attr;          /* IMP_ISP_Tuning_SetGammaAttr (new) */
    bool has_module_control;      /* IMP_ISP_Tuning_SetModuleControl */
    bool has_wdr;                 /* WDR (Wide Dynamic Range) support */

    /* ── OSD Capabilities ── */
    bool has_isp_osd;             /* ISP-level OSD (isp_osd.h, OSD_REG_ISP_*) */
    bool has_osd_mosaic;          /* OSD_REG_MOSAIC region type */
    bool has_osd_group_callback;  /* IMP_OSD_SetGroupCallback (per-frame) */
    bool has_osd_region_invert;   /* Font color inversion support */
    bool has_extended_osd_types;  /* Extended enum (HORIZONTAL_LINE, SLASH, etc.) */

    /* ── Audio Capabilities ── */
    bool has_audio_process_lib;   /* libaudioProcess.so available */
    bool has_audio_aec_channel;   /* IMPAudioAecChn / aecChn field */
    bool has_alc_gain;            /* IMP_AI_SetAlcGain */
    bool has_agc_mode;            /* IMP_AI_SetAgcMode (kAgcModeAdaptive*) */
    bool has_digital_gain;        /* IMP_AI_SetDigitalGain */
    bool has_howling_suppress;    /* IMP_AI_EnableHs */
    bool has_hpf_cutoff;          /* IMP_AI_SetHpfCoFrequency */

    /* ── System Capabilities ── */
    bool uses_xburst2;            /* XBurst2 ISA (T40/T41) */
    bool uses_new_sdk;            /* New-style encoder API (T31+) */
    bool uses_impvi;              /* ISP functions take IMPVI_NUM (T32/T40/T41) */

    /* ── Numeric Limits ── */
    int  max_encode_channels;     /* Maximum encoder channels */
    int  max_osd_regions;         /* Maximum OSD regions per group */
    int  max_osd_groups;          /* Maximum OSD groups */
    int  max_isp_osd_regions;     /* ISP OSD regions (0 if unsupported) */
} rss_hal_caps_t;
```

---

## 2. Per-SoC Capability Matrix

### 2.1 Encoder Capabilities

| Capability | T20 | T21 | T23 | T30 | T31 | T32 | T40 | T41 |
|---|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| `has_h265` | - | (1) | (1) | Y | Y | Y | Y | Y |
| `has_rotation` | - | - | - | - | Y | - | - | - |
| `has_i2d` | - | - | - | - | - | Y | Y | Y |
| `has_bufshare` | - | - | - | - | Y | - | Y | Y |
| `has_set_default_param` | - | - | - | - | Y | Y | Y | Y |
| `has_capped_rc` | - | - | - | - | Y | Y | Y | Y |
| `has_smart_rc` | Y | Y | Y | Y | - | - | - | - |
| `has_gop_attr` | - | - | - | - | Y | Y | Y | Y |
| `has_set_bitrate` | - | - | - | - | Y | Y | Y | Y |
| `has_stream_buf_size` | - | - | - | - | Y | - | Y | Y |
| `has_encoder_pool` | - | - | Y | - | Y | Y | Y | Y |

(1) T21 and T23 declare H.265 structs in headers but mark them "Unsupport". Treat as unavailable for the HAL.

### 2.1b Extended Encoder Capabilities

| Capability | T20 | T21 | T23 | T30 | T31 | T32 | T40 | T41 |
|---|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| `has_smartp_gop` | - | - | - | - | Y | - | Y | Y |
| `has_rc_options` | - | - | - | - | Y | - | Y | Y |
| `has_pskip` | - | - | - | - | - | Y | - | - |
| `has_srd` | - | - | - | - | - | Y | - | - |
| `has_max_pic_size` | - | - | - | - | - | Y | - | Y |
| `has_super_frame` | - | Y | - | - | - | Y | - | - |
| `has_color2grey` | - | Y | - | - | - | - | - | - |
| `has_roi` | - | Y | - | - | - | Y | - | - |
| `has_map_roi` | - | - | - | - | - | Y | - | - |
| `has_qp_bounds_per_frame` | - | - | - | - | - | Y | - | Y |
| `has_qpg_mode` | - | Y | - | - | - | Y | - | - |
| `has_qpg_ai` | - | - | - | - | - | Y | - | - |
| `has_mbrc` | - | Y | - | - | - | - | - | - |
| `has_enc_denoise` | - | Y | - | - | - | - | - | - |
| `has_gdr` | - | - | - | - | - | Y | - | - |
| `has_sei_userdata` | - | Y | - | - | - | Y | - | - |
| `has_h264_vui` | - | - | - | - | - | Y | - | - |
| `has_h265_vui` | - | - | - | - | - | Y | - | - |
| `has_h264_trans` | - | Y | - | - | - | Y | - | - |
| `has_h265_trans` | - | Y | - | - | - | Y | - | - |
| `has_enc_crop` | - | - | - | - | - | Y | - | - |
| `has_eval_info` | - | - | - | - | Y | - | - | - |
| `has_poll_module` | - | Y | - | - | Y | - | Y | Y |
| `has_resize_mode` | - | - | - | - | Y | - | - | Y |
| `has_jpeg_ql` | - | Y | - | - | - | - | - | Y |
| `has_jpeg_qp` | - | - | - | - | - | Y | - | - |

T32 has the broadest extended encoder feature set (T32's SDK exposes 101 encoder functions vs ~70 for T31). T21 inherited several extended features from the T30 SDK lineage despite being an older SoC. T40/T41 share the T31 New SDK encoder with selective T32-era additions.

### 2.2 ISP Capabilities

| Capability | T20 | T21 | T23 | T30 | T31 | T32 | T40 | T41 |
|---|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| `has_multi_sensor` | - | - | (2) | - | - | Y | Y | Y |
| `max_sensors` | 1 | 1 | 3 | 1 | 1 | 3 | 3 | 3 |
| `has_t23_multicam_api` | - | - | Y | - | - | - | - | - |
| `has_defog` | - | - | Y | - | Y | - | - | - |
| `has_dpc` | - | - | Y | - | Y | - | - | - |
| `has_drc` | Y | Y | Y | Y | Y | - | - | - |
| `has_face_ae` | - | - | - | - | - | Y | Y | Y |
| `has_bcsh_hue` | - | - | Y | - | Y | Y | Y | Y |
| `has_sinter` | Y | Y | Y | Y | Y | - | - | - |
| `has_temper` | Y | Y | Y | Y | Y | - | - | - |
| `has_highlight_depress` | Y | Y | Y | Y | Y | - | - | - |
| `has_backlight_comp` | - | - | Y | - | Y | - | - | - |
| `has_ae_comp` | Y | - | Y | Y | Y | - | - | - |
| `has_max_gain` | Y | Y | Y | Y | Y | - | - | - |
| `has_switch_bin` | - | - | - | - | - | Y | Y | Y |
| `has_gamma` | Y | Y | Y | Y | Y | - | - | - |
| `has_gamma_attr` | - | - | - | - | - | Y | Y | Y |
| `has_module_control` | - | Y | Y | - | Y | Y | Y | Y |
| `has_wdr` | Y | - | - | Y | Y | Y | Y | Y |

(2) T23 has `_Sec` suffix functions and `MultiCamera_*` wrappers but does not use `IMPVI_NUM` in the primary API. Marked false in `has_multi_sensor` (which indicates the IMPVI_NUM calling convention). T23 dual-sensor is handled as a special case in the HAL.

**Notes on ISP capability derivation:**
- `has_defog`: `EnableDefog` present on T23 and T31 only (doc 05, section 1.10).
- `has_dpc`: `SetDPC_Strength` present on T23 and T31 only (doc 05, section 1.9).
- `has_drc`: `SetRawDRC` on T20/T21/T30, `SetDRC_Strength` on T21/T23/T31. T32/T40/T41 manage DRC via `SetModuleControl` (not exposed as standalone function). Marked false for T32+ since the standalone API is absent.
- `has_sinter`/`has_temper`: Standalone sinter/temper APIs exist on T20-T31. T32/T40/T41 use `SetModuleControl` and `SetModule_Ratio` instead.
- `has_face_ae`: `SetFaceAe`/`GetFaceAe` present on T32/T40/T41 (doc 05, section 1.18). T32 exposes `SetFaceAeWeiget`/`GetFaceAeLuma` while T40 has `SetFaceAe`/`SetFaceAwb`. Both generations are marked true.
- `has_max_gain`: `SetMaxAgain`/`SetMaxDgain` present on T20-T31 (doc 05, section 1.15). Not present on T32/T40/T41.

### 2.3 OSD Capabilities

| Capability | T20 | T21 | T23 | T30 | T31 | T32 | T40 | T41 |
|---|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| `has_isp_osd` | - | - | Y | - | - | Y | Y | Y |
| `has_osd_mosaic` | - | - | Y | - | - | Y | Y | Y |
| `has_osd_group_callback` | - | - | - | - | - | - | Y | Y |
| `has_osd_region_invert` | - | - | Y | - | Y | Y | Y | Y |
| `has_extended_osd_types` | - | - | Y | - | - | Y | Y | Y |

**Notes:**
- `has_osd_region_invert`: T23 and T32/T40/T41 have `IMPOSDFontAttrData` for color inversion. T31 supports `OSD_REG_PIC_RMEM` and the `rotate_osdflag` frame field but does have inversion via the OSD layer.
- `has_extended_osd_types`: The enum value shift (BITMAP=3->5, COVER=4->6, PIC=5->7) applies to T23/T32/T40/T41.

### 2.4 Audio Capabilities

| Capability | T20 | T21 | T23 | T30 | T31 | T32 | T40 | T41 |
|---|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| `has_audio_process_lib` | Y | Y | Y | Y | Y | Y | Y | Y |
| `has_audio_aec_channel` | - | - | Y | - | - | Y | Y | Y |
| `has_alc_gain` | - | Y | - | - | Y | - | - | - |
| `has_agc_mode` | - | - | - | - | Y | - | - | - |
| `has_digital_gain` | - | - | - | - | - | Y | Y | Y |
| `has_howling_suppress` | - | - | Y | - | - | Y | Y | Y |
| `has_hpf_cutoff` | - | - | Y | - | Y | Y | Y | Y |

**Notes:**
- `has_audio_process_lib`: `libaudioProcess.so` is available on all 8 SoCs (doc 08, section 6).
- `has_alc_gain`: `SetAlcGain`/`GetAlcGain` present on T21 and T31 only (doc 06, section 1.2).
- `has_agc_mode`: `SetAgcMode` (kAgcModeAdaptive* enum) present on T31 only (doc 06, section 1.3).
- `has_audio_aec_channel`: `IMPAudioAecChn` enum and `aecChn` field present on T23, T32, T40, T41 (doc 06, section 2.3).

### 2.5 System and Numeric Limits

| Capability | T20 | T21 | T23 | T30 | T31 | T32 | T40 | T41 |
|---|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| `uses_xburst2` | - | - | - | - | - | - | Y | Y |
| `uses_new_sdk` | - | - | - | - | Y | Y | Y | Y |
| `uses_impvi` | - | - | - | - | - | Y | Y | Y |
| `max_encode_channels` | 2 | 2 | 3 | 2 | 3 | 3 | 4 | 4 |
| `max_osd_regions` | 8 | 8 | 16 | 8 | 8 | 16 | 16 | 16 |
| `max_osd_groups` | 2 | 2 | 2 | 2 | 2 | 2 | 4 | 4 |
| `max_isp_osd_regions` | 0 | 0 | 8 | 0 | 0 | 8 | 8 | 8 |

**Notes on `max_encode_channels`:**
- T20/T21/T30: 2 channels (main + sub stream).
- T23/T31/T32: 3 channels (main + sub + JPEG snapshot).
- T40/T41: 4 channels (dual sensor capable, each with main + sub).

---

## 3. Compile-Time Initialization

The capability struct is a `const` global initialized per platform using `#ifdef`. Only one block is compiled per target.

```c
#include "rss_hal_caps.h"

/* ═══════════════════════════════════════════════════════════════════════
 * T20
 * ═══════════════════════════════════════════════════════════════════════ */
#if defined(PLATFORM_T20)
static const rss_hal_caps_t g_caps = {
    /* Encoder */
    .has_h265               = false,
    .has_rotation           = false,
    .has_i2d                = false,
    .has_bufshare           = false,
    .has_set_default_param  = false,
    .has_capped_rc          = false,
    .has_smart_rc           = true,
    .has_gop_attr           = false,
    .has_set_bitrate        = false,
    .has_stream_buf_size    = false,
    .has_encoder_pool       = false,
    /* ISP */
    .has_multi_sensor       = false,
    .has_defog              = false,
    .has_dpc                = false,
    .has_drc                = true,
    .has_face_ae            = false,
    .has_bcsh_hue           = false,
    .has_sinter             = true,
    .has_temper             = true,
    .has_highlight_depress  = true,
    .has_backlight_comp     = false,
    .has_ae_comp            = true,
    .has_max_gain           = true,
    .has_switch_bin         = false,
    .has_gamma              = true,
    .has_gamma_attr         = false,
    .has_module_control     = false,
    .has_wdr                = true,
    /* OSD */
    .has_isp_osd            = false,
    .has_osd_mosaic         = false,
    .has_osd_group_callback = false,
    .has_osd_region_invert  = false,
    .has_extended_osd_types = false,
    /* Audio */
    .has_audio_process_lib  = true,
    .has_audio_aec_channel  = false,
    .has_alc_gain           = false,
    .has_agc_mode           = false,
    .has_digital_gain       = false,
    .has_howling_suppress   = false,
    .has_hpf_cutoff         = false,
    /* System */
    .uses_xburst2           = false,
    .uses_new_sdk           = false,
    .uses_impvi             = false,
    /* Limits */
    .max_encode_channels    = 2,
    .max_osd_regions        = 8,
    .max_osd_groups         = 2,
    .max_isp_osd_regions    = 0,
};

/* ═══════════════════════════════════════════════════════════════════════
 * T21
 * ═══════════════════════════════════════════════════════════════════════ */
#elif defined(PLATFORM_T21)
static const rss_hal_caps_t g_caps = {
    /* Encoder */
    .has_h265               = false,   /* declared but marked "Unsupport" */
    .has_rotation           = false,
    .has_i2d                = false,
    .has_bufshare           = false,
    .has_set_default_param  = false,
    .has_capped_rc          = false,
    .has_smart_rc           = true,
    .has_gop_attr           = false,
    .has_set_bitrate        = false,
    .has_stream_buf_size    = false,
    .has_encoder_pool       = false,
    /* ISP */
    .has_multi_sensor       = false,
    .has_defog              = false,
    .has_dpc                = false,
    .has_drc                = true,
    .has_face_ae            = false,
    .has_bcsh_hue           = false,
    .has_sinter             = true,
    .has_temper             = true,
    .has_highlight_depress  = true,
    .has_backlight_comp     = false,
    .has_ae_comp            = false,   /* SetAeComp absent on T21 */
    .has_max_gain           = true,
    .has_switch_bin         = false,
    .has_gamma              = true,
    .has_gamma_attr         = false,
    .has_module_control     = true,
    .has_wdr                = false,
    /* OSD */
    .has_isp_osd            = false,
    .has_osd_mosaic         = false,
    .has_osd_group_callback = false,
    .has_osd_region_invert  = false,
    .has_extended_osd_types = false,
    /* Audio */
    .has_audio_process_lib  = true,
    .has_audio_aec_channel  = false,
    .has_alc_gain           = true,
    .has_agc_mode           = false,
    .has_digital_gain       = false,
    .has_howling_suppress   = false,
    .has_hpf_cutoff         = false,
    /* System */
    .uses_xburst2           = false,
    .uses_new_sdk           = false,
    .uses_impvi             = false,
    /* Limits */
    .max_encode_channels    = 2,
    .max_osd_regions        = 8,
    .max_osd_groups         = 2,
    .max_isp_osd_regions    = 0,
};

/* ═══════════════════════════════════════════════════════════════════════
 * T23
 * ═══════════════════════════════════════════════════════════════════════ */
#elif defined(PLATFORM_T23)
static const rss_hal_caps_t g_caps = {
    /* Encoder */
    .has_h265               = false,   /* declared but marked "Unsupport" */
    .has_rotation           = false,
    .has_i2d                = false,
    .has_bufshare           = false,
    .has_set_default_param  = false,
    .has_capped_rc          = false,
    .has_smart_rc           = true,
    .has_gop_attr           = false,
    .has_set_bitrate        = false,
    .has_stream_buf_size    = false,
    .has_encoder_pool       = true,
    /* ISP */
    .has_multi_sensor       = false,   /* _Sec suffix, not IMPVI_NUM */
    .has_defog              = true,
    .has_dpc                = true,
    .has_drc                = true,
    .has_face_ae            = false,
    .has_bcsh_hue           = true,
    .has_sinter             = true,
    .has_temper             = true,
    .has_highlight_depress  = true,
    .has_backlight_comp     = true,
    .has_ae_comp            = true,
    .has_max_gain           = true,
    .has_switch_bin         = false,   /* needs SDK 1.1.2+, default is 1.1.0 */
    .has_gamma              = true,
    .has_gamma_attr         = false,
    .has_module_control     = true,
    .has_wdr                = false,
    /* OSD */
    .has_isp_osd            = true,
    .has_osd_mosaic         = true,
    .has_osd_group_callback = false,
    .has_osd_region_invert  = true,
    .has_extended_osd_types = true,
    /* Audio */
    .has_audio_process_lib  = true,
    .has_audio_aec_channel  = true,
    .has_alc_gain           = false,
    .has_agc_mode           = false,
    .has_digital_gain       = false,
    .has_howling_suppress   = true,
    .has_hpf_cutoff         = true,
    /* System */
    .uses_xburst2           = false,
    .uses_new_sdk           = false,
    .uses_impvi             = false,
    /* Limits */
    .max_encode_channels    = 3,
    .max_osd_regions        = 16,
    .max_osd_groups         = 2,
    .max_isp_osd_regions    = 8,
};

/* ═══════════════════════════════════════════════════════════════════════
 * T30
 * ═══════════════════════════════════════════════════════════════════════ */
#elif defined(PLATFORM_T30)
static const rss_hal_caps_t g_caps = {
    /* Encoder */
    .has_h265               = true,
    .has_rotation           = false,
    .has_i2d                = false,
    .has_bufshare           = false,
    .has_set_default_param  = false,
    .has_capped_rc          = false,
    .has_smart_rc           = true,
    .has_gop_attr           = false,
    .has_set_bitrate        = false,
    .has_stream_buf_size    = false,
    .has_encoder_pool       = false,
    /* ISP */
    .has_multi_sensor       = false,
    .has_defog              = false,
    .has_dpc                = false,
    .has_drc                = true,
    .has_face_ae            = false,
    .has_bcsh_hue           = false,
    .has_sinter             = true,
    .has_temper             = true,
    .has_highlight_depress  = true,
    .has_backlight_comp     = false,
    .has_ae_comp            = true,
    .has_max_gain           = true,
    .has_switch_bin         = false,
    .has_gamma              = true,
    .has_gamma_attr         = false,
    .has_module_control     = false,
    .has_wdr                = true,
    /* OSD */
    .has_isp_osd            = false,
    .has_osd_mosaic         = false,
    .has_osd_group_callback = false,
    .has_osd_region_invert  = false,
    .has_extended_osd_types = false,
    /* Audio */
    .has_audio_process_lib  = true,
    .has_audio_aec_channel  = false,
    .has_alc_gain           = false,
    .has_agc_mode           = false,
    .has_digital_gain       = false,
    .has_howling_suppress   = false,
    .has_hpf_cutoff         = false,
    /* System */
    .uses_xburst2           = false,
    .uses_new_sdk           = false,
    .uses_impvi             = false,
    /* Limits */
    .max_encode_channels    = 2,
    .max_osd_regions        = 8,
    .max_osd_groups         = 2,
    .max_isp_osd_regions    = 0,
};

/* ═══════════════════════════════════════════════════════════════════════
 * T31
 * ═══════════════════════════════════════════════════════════════════════ */
#elif defined(PLATFORM_T31)
static const rss_hal_caps_t g_caps = {
    /* Encoder */
    .has_h265               = true,
    .has_rotation           = true,
    .has_i2d                = false,
    .has_bufshare           = true,
    .has_set_default_param  = true,
    .has_capped_rc          = true,
    .has_smart_rc           = false,
    .has_gop_attr           = true,
    .has_set_bitrate        = true,
    .has_stream_buf_size    = true,
    .has_encoder_pool       = true,
    /* ISP */
    .has_multi_sensor       = false,
    .has_defog              = true,
    .has_dpc                = true,
    .has_drc                = true,
    .has_face_ae            = false,
    .has_bcsh_hue           = true,
    .has_sinter             = true,
    .has_temper             = true,
    .has_highlight_depress  = true,
    .has_backlight_comp     = true,
    .has_ae_comp            = true,
    .has_max_gain           = true,
    .has_switch_bin         = false,
    .has_gamma              = true,
    .has_gamma_attr         = false,
    .has_module_control     = true,
    .has_wdr                = true,
    /* OSD */
    .has_isp_osd            = false,
    .has_osd_mosaic         = false,
    .has_osd_group_callback = false,
    .has_osd_region_invert  = true,
    .has_extended_osd_types = false,
    /* Audio */
    .has_audio_process_lib  = true,
    .has_audio_aec_channel  = false,
    .has_alc_gain           = true,
    .has_agc_mode           = true,
    .has_digital_gain       = false,
    .has_howling_suppress   = false,
    .has_hpf_cutoff         = true,
    /* System */
    .uses_xburst2           = false,
    .uses_new_sdk           = true,
    .uses_impvi             = false,
    /* Limits */
    .max_encode_channels    = 3,
    .max_osd_regions        = 8,
    .max_osd_groups         = 2,
    .max_isp_osd_regions    = 0,
};

/* ═══════════════════════════════════════════════════════════════════════
 * T32
 * ═══════════════════════════════════════════════════════════════════════ */
#elif defined(PLATFORM_T32)
static const rss_hal_caps_t g_caps = {
    /* Encoder */
    .has_h265               = true,
    .has_rotation           = false,
    .has_i2d                = true,
    .has_bufshare           = false,   /* SetbufshareChn not present on T32 */
    .has_set_default_param  = true,
    .has_capped_rc          = true,
    .has_smart_rc           = false,
    .has_gop_attr           = true,
    .has_set_bitrate        = true,
    .has_stream_buf_size    = false,
    .has_encoder_pool       = true,
    /* ISP */
    .has_multi_sensor       = true,    /* IMPVI_NUM, up to 4 sensors */
    .has_defog              = false,
    .has_dpc                = false,
    .has_drc                = false,   /* via SetModuleControl only */
    .has_face_ae            = true,
    .has_bcsh_hue           = true,
    .has_sinter             = false,   /* via SetModuleControl only */
    .has_temper             = false,   /* via SetModuleControl only */
    .has_highlight_depress  = false,
    .has_backlight_comp     = false,
    .has_ae_comp            = false,
    .has_max_gain           = false,
    .has_switch_bin         = true,
    .has_gamma              = false,
    .has_gamma_attr         = true,
    .has_module_control     = true,
    .has_wdr                = true,
    /* OSD */
    .has_isp_osd            = true,
    .has_osd_mosaic         = true,
    .has_osd_group_callback = false,
    .has_osd_region_invert  = true,
    .has_extended_osd_types = true,
    /* Audio */
    .has_audio_process_lib  = true,
    .has_audio_aec_channel  = true,
    .has_alc_gain           = false,
    .has_agc_mode           = false,
    .has_digital_gain       = true,
    .has_howling_suppress   = true,
    .has_hpf_cutoff         = true,
    /* System */
    .uses_xburst2           = false,
    .uses_new_sdk           = true,
    .uses_impvi             = true,
    /* Limits */
    .max_encode_channels    = 3,
    .max_osd_regions        = 16,
    .max_osd_groups         = 2,
    .max_isp_osd_regions    = 8,
};

/* ═══════════════════════════════════════════════════════════════════════
 * T40
 * ═══════════════════════════════════════════════════════════════════════ */
#elif defined(PLATFORM_T40)
static const rss_hal_caps_t g_caps = {
    /* Encoder */
    .has_h265               = true,
    .has_rotation           = false,
    .has_i2d                = true,
    .has_bufshare           = true,
    .has_set_default_param  = true,
    .has_capped_rc          = true,
    .has_smart_rc           = false,
    .has_gop_attr           = true,
    .has_set_bitrate        = true,
    .has_stream_buf_size    = true,
    .has_encoder_pool       = true,
    /* ISP */
    .has_multi_sensor       = true,
    .has_defog              = false,
    .has_dpc                = false,
    .has_drc                = false,   /* via SetModuleControl only */
    .has_face_ae            = true,
    .has_bcsh_hue           = true,
    .has_sinter             = false,   /* via SetModuleControl only */
    .has_temper             = false,   /* via SetModuleControl only */
    .has_highlight_depress  = false,
    .has_backlight_comp     = false,
    .has_ae_comp            = false,
    .has_max_gain           = false,
    .has_switch_bin         = true,
    .has_gamma              = false,
    .has_gamma_attr         = true,
    .has_module_control     = true,
    .has_wdr                = true,
    /* OSD */
    .has_isp_osd            = true,
    .has_osd_mosaic         = true,
    .has_osd_group_callback = true,
    .has_osd_region_invert  = true,
    .has_extended_osd_types = true,
    /* Audio */
    .has_audio_process_lib  = true,
    .has_audio_aec_channel  = true,
    .has_alc_gain           = false,
    .has_agc_mode           = false,
    .has_digital_gain       = true,
    .has_howling_suppress   = true,
    .has_hpf_cutoff         = true,
    /* System */
    .uses_xburst2           = true,
    .uses_new_sdk           = true,
    .uses_impvi             = true,
    /* Limits */
    .max_encode_channels    = 4,
    .max_osd_regions        = 16,
    .max_osd_groups         = 4,
    .max_isp_osd_regions    = 8,
};

/* ═══════════════════════════════════════════════════════════════════════
 * T41
 * ═══════════════════════════════════════════════════════════════════════ */
#elif defined(PLATFORM_T41)
static const rss_hal_caps_t g_caps = {
    /* Encoder */
    .has_h265               = true,
    .has_rotation           = false,
    .has_i2d                = true,
    .has_bufshare           = true,
    .has_set_default_param  = true,
    .has_capped_rc          = true,
    .has_smart_rc           = false,
    .has_gop_attr           = true,
    .has_set_bitrate        = true,
    .has_stream_buf_size    = true,
    .has_encoder_pool       = true,
    /* ISP */
    .has_multi_sensor       = true,    /* IMPVI_NUM defined but SEC/THR unsupported */
    .has_defog              = false,
    .has_dpc                = false,
    .has_drc                = false,   /* via SetModuleControl only */
    .has_face_ae            = true,
    .has_bcsh_hue           = true,
    .has_sinter             = false,   /* via SetModuleControl only */
    .has_temper             = false,   /* via SetModuleControl only */
    .has_highlight_depress  = false,
    .has_backlight_comp     = false,
    .has_ae_comp            = false,
    .has_max_gain           = false,
    .has_switch_bin         = true,
    .has_gamma              = false,
    .has_gamma_attr         = true,
    .has_module_control     = true,
    .has_wdr                = true,
    /* OSD */
    .has_isp_osd            = true,
    .has_osd_mosaic         = true,
    .has_osd_group_callback = true,
    .has_osd_region_invert  = true,
    .has_extended_osd_types = true,
    /* Audio */
    .has_audio_process_lib  = true,
    .has_audio_aec_channel  = true,
    .has_alc_gain           = false,
    .has_agc_mode           = false,
    .has_digital_gain       = true,
    .has_howling_suppress   = true,
    .has_hpf_cutoff         = true,
    /* System */
    .uses_xburst2           = true,
    .uses_new_sdk           = true,
    .uses_impvi             = true,
    /* Limits */
    .max_encode_channels    = 4,
    .max_osd_regions        = 16,
    .max_osd_groups         = 4,
    .max_isp_osd_regions    = 8,
};

#else
  #error "No PLATFORM_* defined. Set one of: PLATFORM_T20 T21 T23 T30 T31 T32 T40 T41"
#endif

/* ═══════════════════════════════════════════════════════════════════════
 * Public accessor
 * ═══════════════════════════════════════════════════════════════════════ */

const rss_hal_caps_t *rss_hal_caps(void) {
    return &g_caps;
}
```

---

## 4. Usage Pattern

Callers query capabilities before using optional features:

```c
const rss_hal_caps_t *caps = rss_hal_caps();

/* Check before creating H.265 channel */
if (caps->has_h265) {
    hal_encoder_create_channel(chn, &h265_config);
}

/* Check before enabling defog */
if (caps->has_defog) {
    hal_isp_set_defog(IMPVI_MAIN, 128);
}

/* Check before using ISP OSD */
if (caps->has_isp_osd && caps->max_isp_osd_regions > 0) {
    hal_osd_create_isp_region(sensor, &attr);
}

/* Iterate only available channels */
for (int i = 0; i < caps->max_encode_channels; i++) {
    hal_encoder_poll_stream(i, timeout);
}
```

The caps struct is `const` and never changes at runtime. It can be safely read from any thread without synchronization.

---

## 5. Consolidated Quick-Reference Matrix

For convenience, the complete boolean matrix in a single table. Y = true, - = false.

| Field | T20 | T21 | T23 | T30 | T31 | T32 | T40 | T41 |
|---|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| has_h265 | - | - | - | Y | Y | Y | Y | Y |
| has_rotation | - | - | - | - | Y | - | - | - |
| has_i2d | - | - | - | - | - | Y | Y | Y |
| has_bufshare | - | - | - | - | Y | - | Y | Y |
| has_set_default_param | - | - | - | - | Y | Y | Y | Y |
| has_capped_rc | - | - | - | - | Y | Y | Y | Y |
| has_smart_rc | Y | Y | Y | Y | - | - | - | - |
| has_gop_attr | - | - | - | - | Y | Y | Y | Y |
| has_set_bitrate | - | - | - | - | Y | Y | Y | Y |
| has_stream_buf_size | - | - | - | - | Y | - | Y | Y |
| has_encoder_pool | - | - | Y | - | Y | Y | Y | Y |
| has_multi_sensor | - | - | - | - | - | Y | Y | Y |
| has_defog | - | - | Y | - | Y | - | - | - |
| has_dpc | - | - | Y | - | Y | - | - | - |
| has_drc | Y | Y | Y | Y | Y | - | - | - |
| has_face_ae | - | - | - | - | - | Y | Y | Y |
| has_bcsh_hue | - | - | Y | - | Y | Y | Y | Y |
| has_sinter | Y | Y | Y | Y | Y | - | - | - |
| has_temper | Y | Y | Y | Y | Y | - | - | - |
| has_highlight_depress | Y | Y | Y | Y | Y | - | - | - |
| has_backlight_comp | - | - | Y | - | Y | - | - | - |
| has_ae_comp | Y | - | Y | Y | Y | - | - | - |
| has_max_gain | Y | Y | Y | Y | Y | - | - | - |
| has_switch_bin | - | - | - | - | - | Y | Y | Y |
| has_gamma | Y | Y | Y | Y | Y | - | - | - |
| has_gamma_attr | - | - | - | - | - | Y | Y | Y |
| has_module_control | - | Y | Y | - | Y | Y | Y | Y |
| has_wdr | Y | - | - | Y | Y | Y | Y | Y |
| has_isp_osd | - | - | Y | - | - | Y | Y | Y |
| has_osd_mosaic | - | - | Y | - | - | Y | Y | Y |
| has_osd_group_callback | - | - | - | - | - | - | Y | Y |
| has_osd_region_invert | - | - | Y | - | Y | Y | Y | Y |
| has_extended_osd_types | - | - | Y | - | - | Y | Y | Y |
| has_audio_process_lib | Y | Y | Y | Y | Y | Y | Y | Y |
| has_audio_aec_channel | - | - | Y | - | - | Y | Y | Y |
| has_alc_gain | - | Y | - | - | Y | - | - | - |
| has_agc_mode | - | - | - | - | Y | - | - | - |
| has_digital_gain | - | - | - | - | - | Y | Y | Y |
| has_howling_suppress | - | - | Y | - | - | Y | Y | Y |
| has_hpf_cutoff | - | - | Y | - | Y | Y | Y | Y |
| uses_xburst2 | - | - | - | - | - | - | Y | Y |
| uses_new_sdk | - | - | - | - | Y | Y | Y | Y |
| uses_impvi | - | - | - | - | - | Y | Y | Y |
| max_encode_channels | 2 | 2 | 3 | 2 | 3 | 3 | 4 | 4 |
| max_osd_regions | 8 | 8 | 16 | 8 | 8 | 16 | 16 | 16 |
| max_osd_groups | 2 | 2 | 2 | 2 | 2 | 2 | 4 | 4 |
| max_isp_osd_regions | 0 | 0 | 8 | 0 | 0 | 8 | 8 | 8 |

# Raptor Streaming System -- HAL Public API

Complete C API definition for `raptor_hal.h`. This header is the single
include consumed by the RSS daemons (RVD, RAD, RIC). All vendor SDK types
and cross-SoC differences are hidden behind this interface.

The HAL uses a vtable pattern (`rss_hal_ops_t`) with per-SoC implementations
selected at compile time. Every function pointer takes an opaque context
(`rss_hal_ctx_t *`) as its first argument and returns `int` (0 = success,
negative = error).

---

## 1. Master Header

```c
/*
 * raptor_hal.h -- Raptor Streaming System Hardware Abstraction Layer
 *
 * This is the ONLY header that RSS daemons include for hardware access.
 * All Ingenic IMP SDK types are abstracted behind RSS types.
 *
 * Consumers:
 *   RVD  -- video daemon (ISP, framesource, encoder, OSD, frame output)
 *   RAD  -- audio daemon (audio input, encoding, frame output)
 *   RIC  -- IR control daemon (exposure queries)
 */

#ifndef RAPTOR_HAL_H
#define RAPTOR_HAL_H

#include <stdint.h>
#include <stdbool.h>
#include <stddef.h>

#ifdef __cplusplus
extern "C" {
#endif
```

---

## 2. HAL Types

### 2.1 Video Codec Type

```c
/*
 * rss_codec_t -- video codec selection.
 *
 * H265 is not available on T20. The HAL returns -ENOTSUP if an
 * unsupported codec is requested. Query rss_hal_caps_t.has_h265.
 */
typedef enum {
    RSS_CODEC_H264  = 0,
    RSS_CODEC_H265  = 1,
    RSS_CODEC_JPEG  = 2,
    RSS_CODEC_MJPEG = 3,    /* JPEG in streaming mode (continuous snap) */
} rss_codec_t;
```

### 2.2 Rate Control Mode

```c
/*
 * rss_rc_mode_t -- rate control mode.
 *
 * Old SDK (T20/T21/T23/T30) supports FIXQP, CBR, VBR, SMART.
 * New SDK (T31/T32/T40/T41) supports FIXQP, CBR, VBR, CAPPED_VBR, CAPPED_QUALITY.
 *
 * The HAL maps SMART -> CAPPED_VBR on old SDK, and silently falls back to
 * VBR if capped modes are requested on old-SDK SoCs.
 */
typedef enum {
    RSS_RC_FIXQP          = 0,
    RSS_RC_CBR            = 1,
    RSS_RC_VBR            = 2,
    RSS_RC_SMART          = 3,   /* old SDK only; mapped to CAPPED_VBR on new */
    RSS_RC_CAPPED_VBR     = 4,   /* new SDK only; mapped to VBR on old */
    RSS_RC_CAPPED_QUALITY = 5,   /* new SDK only; mapped to VBR on old */
} rss_rc_mode_t;
```

### 2.3 Pixel Format

```c
/*
 * rss_pixfmt_t -- pixel format.
 *
 * Superset of all SoC-supported formats. The HAL validates format
 * support per-SoC before passing to SDK functions.
 */
typedef enum {
    RSS_PIXFMT_NV12       = 0,   /* semi-planar YUV 4:2:0, 12bpp -- all SoCs */
    RSS_PIXFMT_NV21       = 1,   /* semi-planar VU 4:2:0, 12bpp -- all SoCs */
    RSS_PIXFMT_YUYV422    = 2,   /* packed YUV 4:2:2, 16bpp -- all SoCs */
    RSS_PIXFMT_UYVY422    = 3,   /* packed YUV 4:2:2, 16bpp -- all SoCs */
    RSS_PIXFMT_YUV420P    = 4,   /* planar YUV 4:2:0, 12bpp */
    RSS_PIXFMT_RGB24      = 5,
    RSS_PIXFMT_BGR24      = 6,
    RSS_PIXFMT_BGRA       = 7,   /* 32bpp, used for OSD bitmap data */
    RSS_PIXFMT_ARGB       = 8,
    RSS_PIXFMT_RGB565LE   = 9,
    RSS_PIXFMT_GRAY8      = 10,
    RSS_PIXFMT_RAW        = 11,  /* generic raw sensor data */
    RSS_PIXFMT_RAW8       = 12,  /* T32/T40/T41 only */
    RSS_PIXFMT_RAW16      = 13,  /* T32/T40/T41 only */
} rss_pixfmt_t;
```

### 2.4 NAL Unit Type

```c
/*
 * rss_nal_type_t -- abstracted NAL unit type.
 *
 * The HAL translates vendor-specific NAL type enums
 * (IMPEncoderH264NaluType, IMPEncoderH265NaluType) and the different
 * access paths (old SDK: dataType.h264Type, new SDK: nalType.h264NalType)
 * into this unified enum.
 */
typedef enum {
    /* H.264 NAL types */
    RSS_NAL_H264_SPS       = 0x10,
    RSS_NAL_H264_PPS       = 0x11,
    RSS_NAL_H264_SEI       = 0x12,
    RSS_NAL_H264_IDR       = 0x13,
    RSS_NAL_H264_SLICE     = 0x14,

    /* H.265 NAL types */
    RSS_NAL_H265_VPS       = 0x20,
    RSS_NAL_H265_SPS       = 0x21,
    RSS_NAL_H265_PPS       = 0x22,
    RSS_NAL_H265_SEI       = 0x23,
    RSS_NAL_H265_IDR       = 0x24,   /* IDR_W_RADL or IDR_N_LP */
    RSS_NAL_H265_SLICE     = 0x25,

    /* JPEG */
    RSS_NAL_JPEG_FRAME     = 0x30,

    /* Unknown / other */
    RSS_NAL_UNKNOWN        = 0xFF,
} rss_nal_type_t;
```

### 2.5 Encoded Video Frame

```c
/*
 * rss_nal_unit_t -- single NAL unit within an encoded frame.
 *
 * The HAL linearizes ring-buffer-wrapped data (new SDK) so that
 * `data` always points to a contiguous buffer. The caller must not
 * free `data` -- it is valid until rss_hal_ops_t.enc_release_frame().
 */
typedef struct {
    const uint8_t  *data;       /* pointer to NAL unit payload */
    uint32_t        length;     /* payload length in bytes */
    rss_nal_type_t  type;       /* abstracted NAL type */
    bool            frame_end;  /* true if this is the last NAL in the frame */
} rss_nal_unit_t;

/*
 * rss_frame_t -- abstracted encoded video frame.
 *
 * Returned by enc_get_frame(). Contains one or more NAL units
 * comprising a single access unit (frame). For H.264 IDR frames,
 * the array typically contains SPS + PPS + IDR NALs. For H.265 IDR,
 * VPS + SPS + PPS + IDR.
 *
 * The `_priv` field is opaque HAL-internal state used by
 * enc_release_frame() to return resources to the SDK.
 */
typedef struct {
    rss_nal_unit_t *nals;       /* array of NAL units */
    uint32_t        nal_count;  /* number of NAL units in array */
    rss_codec_t     codec;      /* codec that produced this frame */
    int64_t         timestamp;  /* capture timestamp in microseconds */
    uint32_t        seq;        /* frame sequence number */
    bool            is_key;     /* true if IDR / keyframe */

    /* --- HAL-internal: do not touch --- */
    void           *_priv;      /* opaque handle for release */
} rss_frame_t;
```

### 2.6 Audio Frame

```c
/*
 * rss_audio_frame_t -- abstracted audio frame.
 *
 * Returned by audio_read_frame(). PCM 16-bit samples.
 * The caller must not free `data` -- it is valid until the next
 * audio_read_frame() call or audio_deinit().
 */
typedef struct {
    const int16_t  *data;         /* PCM sample data */
    uint32_t        length;       /* data length in bytes */
    int64_t         timestamp;    /* capture timestamp in microseconds */
    uint32_t        seq;          /* frame sequence number */

    /* --- HAL-internal --- */
    void           *_priv;
} rss_audio_frame_t;
```

### 2.7 Video Encoder Configuration

```c
/*
 * rss_video_config_t -- encoder channel configuration.
 *
 * Passed to enc_create_channel(). The HAL translates this into
 * the appropriate SDK struct:
 *   Old SDK: IMPEncoderAttr + IMPEncoderRcAttr -> IMPEncoderCHNAttr
 *   New SDK: IMPEncoderEncAttr + IMPEncoderRcAttr + IMPEncoderGopAttr -> IMPEncoderChnAttr
 *
 * Profile selection:
 *   H264: 0=baseline, 1=main, 2=high (mapped to eProfile on new SDK)
 *   H265: always main profile
 *   JPEG: profile ignored
 */
typedef struct {
    rss_codec_t     codec;
    uint16_t        width;         /* encoding width in pixels */
    uint16_t        height;        /* encoding height in pixels */
    int             profile;       /* H264: 0=base,1=main,2=high; H265: ignored */

    /* Rate control */
    rss_rc_mode_t   rc_mode;
    uint32_t        bitrate;       /* target bitrate in bps (CBR/VBR/capped modes) */
    uint32_t        max_bitrate;   /* max bitrate in bps (VBR/capped modes) */
    int16_t         init_qp;       /* initial QP; -1 for SDK default */
    int16_t         min_qp;        /* minimum QP [0..51] */
    int16_t         max_qp;        /* maximum QP [0..51] */

    /* Frame rate as rational number */
    uint32_t        fps_num;       /* numerator (e.g. 25) */
    uint32_t        fps_den;       /* denominator (e.g. 1) */

    /* GOP */
    uint32_t        gop_length;    /* GOP length in frames */

    /* Optional: buffer size hint; 0 = SDK default */
    uint32_t        buf_size;
} rss_video_config_t;
```

### 2.8 Framesource Configuration

```c
/*
 * rss_fs_config_t -- framesource channel configuration.
 *
 * Passed to fs_create_channel(). The HAL populates the vendor
 * IMPFSChnAttr (layout varies across SoC generations) from these
 * normalized fields.
 *
 * Rotation:
 *   T31: software rotation via SetChnRotate (max ~1280x704@15fps)
 *   T32/T40/T41: hardware I2D rotation (no CPU overhead)
 *   Others: not supported; HAL returns -ENOTSUP
 *
 * Crop:
 *   `crop` applies at ISP level (before scaling) -- all SoCs.
 *   `fcrop` applies after scaling -- T23/T31/T32/T40/T41 only.
 */
typedef struct {
    uint16_t        width;         /* output width in pixels */
    uint16_t        height;        /* output height in pixels */
    rss_pixfmt_t    pixfmt;        /* output pixel format (typically NV12) */

    /* Frame rate as rational number */
    uint32_t        fps_num;
    uint32_t        fps_den;

    /* ISP-level crop (before scaling). Set all zero to disable. */
    struct {
        bool    enable;
        int     x, y;              /* top-left corner */
        int     w, h;              /* crop region size */
    } crop;

    /* Frame-level crop (after scaling). T23+ only. Zero to disable. */
    struct {
        bool    enable;
        int     x, y;
        int     w, h;
    } fcrop;

    /* Number of video buffer blocks; 0 = SDK default (typically 2-4) */
    int             nr_vbs;

    /* Channel type: 0 = physical, 1 = extension */
    int             chn_type;
} rss_fs_config_t;
```

### 2.9 Audio Configuration

```c
/*
 * rss_audio_config_t -- audio device and channel configuration.
 *
 * Passed to audio_init(). The HAL populates IMPAudioIOAttr and
 * IMPAudioIChnParam from these fields.
 */
typedef enum {
    RSS_AUDIO_RATE_8000  = 8000,
    RSS_AUDIO_RATE_16000 = 16000,
    RSS_AUDIO_RATE_24000 = 24000,
    RSS_AUDIO_RATE_32000 = 32000,
    RSS_AUDIO_RATE_44100 = 44100,
    RSS_AUDIO_RATE_48000 = 48000,
} rss_audio_rate_t;

typedef struct {
    rss_audio_rate_t sample_rate;
    int              samples_per_frame;  /* numPerFrm: samples per frame period */
    int              chn_count;          /* 1=mono, 2=stereo */
    int              frame_depth;        /* usrFrmDepth [2..50] */

    /* Volume and gain; applied after init */
    int              ai_vol;             /* AI software volume [-30..120], 60=unity */
    int              ai_gain;            /* AI hardware gain [0..31] */
} rss_audio_config_t;
```

### 2.10 ISP Image Attributes

```c
/*
 * rss_image_attr_t -- ISP tuning values for brightness/contrast/etc.
 *
 * All values are unsigned char [0..255] with 128 as neutral midpoint
 * (matching the Ingenic SDK convention).
 *
 * hue is only available on T23/T31/T32/T40/T41 (see caps.has_bcsh_hue).
 */
typedef struct {
    uint8_t brightness;     /* [0..255], 128 = neutral */
    uint8_t contrast;       /* [0..255], 128 = neutral */
    uint8_t saturation;     /* [0..255], 128 = neutral */
    uint8_t sharpness;      /* [0..255], 128 = neutral */
    uint8_t hue;            /* [0..255], 128 = neutral; requires has_bcsh_hue */
} rss_image_attr_t;
```

### 2.11 OSD Region Definition

```c
/*
 * rss_osd_region_t -- OSD region definition for bitmap upload.
 *
 * The HAL maps to vendor OSD_REG_PIC / OSD_REG_COVER with
 * per-SoC enum value translation (e.g. OSD_REG_PIC is 5 on classic
 * SDK, 7 on extended SDK).
 *
 * Bitmap data must be in BGRA format (PIX_FMT_BGRA), with the
 * pointer remaining valid until the next osd_update_region_data()
 * or osd_destroy_region().
 */
typedef enum {
    RSS_OSD_PIC        = 0,   /* RGBA picture (timestamp, logo) */
    RSS_OSD_COVER      = 1,   /* solid color rectangle (privacy mask) */
    RSS_OSD_PIC_RMEM   = 2,   /* picture from reserved memory (T31+) */
} rss_osd_type_t;

typedef struct {
    rss_osd_type_t  type;

    /* Position and size (top-left origin, exclusive coordinates) */
    int             x;
    int             y;
    int             width;
    int             height;

    /* For PIC / PIC_RMEM: bitmap data */
    const uint8_t  *bitmap_data;      /* BGRA pixel data, width*height*4 bytes */
    rss_pixfmt_t    bitmap_fmt;       /* must be RSS_PIXFMT_BGRA */

    /* For COVER: fill color (BGRA) */
    uint32_t        cover_color;

    /* Alpha blending */
    bool            global_alpha_en;
    uint8_t         fg_alpha;         /* foreground alpha [0..255] */
    uint8_t         bg_alpha;         /* background alpha [0..255] */

    /* Z-order layer */
    int             layer;
} rss_osd_region_t;
```

### 2.12 Sensor Configuration

```c
/*
 * rss_sensor_config_t -- sensor configuration for HAL init.
 *
 * The HAL populates vendor IMPSensorInfo from these fields.
 * On T40/T32/T41 the additional fields (video_interface, mclk,
 * default_boot) are also set.
 */
typedef enum {
    RSS_SENSOR_VIN_MIPI_CSI0 = 0,
    RSS_SENSOR_VIN_MIPI_CSI1 = 1,
    RSS_SENSOR_VIN_DVP       = 2,
} rss_sensor_vin_t;

typedef enum {
    RSS_SENSOR_MCLK0 = 0,
    RSS_SENSOR_MCLK1 = 1,
    RSS_SENSOR_MCLK2 = 2,
} rss_sensor_mclk_t;

typedef struct {
    char                name[32];       /* sensor driver name (e.g. "gc2053") */
    uint16_t            i2c_addr;       /* 7-bit I2C address */
    int                 i2c_adapter;    /* I2C bus number */
    uint16_t            sensor_id;      /* sensor ID; 0 if unused (T20/T21/T30/T31) */

    /* T40/T32/T41 only -- ignored on older SoCs */
    rss_sensor_vin_t    vin_type;
    rss_sensor_mclk_t   mclk;
    int                 default_boot;

    /* GPIO pins; -1 = unused */
    int                 rst_gpio;
    int                 pwdn_gpio;
    int                 power_gpio;
} rss_sensor_config_t;
```

### 2.13 ISP Exposure Info (for RIC)

```c
/*
 * rss_exposure_t -- exposure information for IR-cut control.
 *
 * On old SDK: obtained via IMP_ISP_Tuning_GetExpr / GetTotalGain.
 * On new SDK: obtained via IMP_ISP_Tuning_GetAeExprInfo.
 */
typedef struct {
    uint32_t    total_gain;      /* total gain (sensor analog + digital + ISP) */
    uint32_t    exposure_time;   /* integration time in microseconds */
    uint32_t    ae_luma;         /* AE average luminance (where available) */
} rss_exposure_t;
```

### 2.14 White Balance Configuration

```c
/*
 * rss_wb_config_t -- white balance configuration.
 *
 * On old SDK: mapped to IMP_ISP_Tuning_SetWB.
 * On new SDK: mapped to IMP_ISP_Tuning_SetAwbAttr.
 */
typedef enum {
    RSS_WB_AUTO    = 0,
    RSS_WB_MANUAL  = 1,
} rss_wb_mode_t;

typedef struct {
    rss_wb_mode_t   mode;
    uint16_t        r_gain;      /* red gain (manual mode) */
    uint16_t        g_gain;      /* green gain (manual mode) */
    uint16_t        b_gain;      /* blue gain (manual mode) */
} rss_wb_config_t;
```

### 2.15 AGC Configuration

```c
/*
 * rss_agc_config_t -- audio AGC parameters.
 *
 * Maps directly to IMPAudioAgcConfig.
 */
typedef struct {
    int target_level_dbfs;    /* [0..31], target volume in -dB; smaller = louder */
    int compression_gain_db;  /* [0..90], max gain in dB; 0 = no gain */
} rss_agc_config_t;
```

### 2.16 Audio Encoder Registration

```c
/*
 * rss_audio_encoder_t -- custom audio encoder callbacks.
 *
 * Wraps IMPAudioEncEncoder for registering codecs beyond the
 * built-in G.711A/G.711U/G.726.
 */
typedef struct {
    char    name[16];
    int     max_frame_len;

    int   (*open)(void *attr, void *encoder);
    int   (*encode)(void *encoder, const int16_t *pcm, int pcm_len,
                    uint8_t *out, int *out_len);
    int   (*close)(void *encoder);
} rss_audio_encoder_t;
```

### 2.17 Noise Suppression Level

```c
typedef enum {
    RSS_NS_LOW       = 0,
    RSS_NS_MODERATE  = 1,
    RSS_NS_HIGH      = 2,
    RSS_NS_VERYHIGH  = 3,
} rss_ns_level_t;
```

### 2.18 Pipeline Cell (for bind/unbind)

```c
/*
 * rss_cell_t -- pipeline endpoint for bind/unbind.
 *
 * Wraps IMPCell { IMPDeviceID deviceID, int groupID, int outputID }.
 */
typedef enum {
    RSS_DEV_FS  = 0,   /* FrameSource */
    RSS_DEV_ENC = 1,   /* Encoder */
    RSS_DEV_OSD = 4,   /* OSD */
} rss_dev_id_t;

typedef struct {
    rss_dev_id_t    device;
    int             group;
    int             output;
} rss_cell_t;
```

### 2.19 ISP Running Mode

```c
typedef enum {
    RSS_ISP_DAY   = 0,
    RSS_ISP_NIGHT = 1,
} rss_isp_mode_t;
```

### 2.20 Anti-Flicker Mode

```c
typedef enum {
    RSS_ANTIFLICKER_OFF  = 0,
    RSS_ANTIFLICKER_50HZ = 1,
    RSS_ANTIFLICKER_60HZ = 2,
} rss_antiflicker_t;
```

---

## 3. Capability Struct

```c
/*
 * rss_hal_caps_t -- runtime feature queries.
 *
 * Returned by get_caps(). Allows daemons to adapt behavior
 * without compile-time SoC knowledge.
 */
typedef struct {
    /* --- System info --- */
    const char *soc_name;           /* e.g. "T31", "T40" */
    const char *sdk_version;        /* e.g. "1.1.6" */
    int         max_fs_channels;    /* max framesource channels (2-3) */
    int         max_enc_channels;   /* max encoder channels */
    int         max_osd_groups;     /* max OSD groups (2-4) */
    int         max_osd_regions;    /* max regions per OSD group (8-16) */

    /* --- Codec support --- */
    bool has_h265;                  /* false on T20 */
    bool has_jpeg_custom_qt;        /* custom JPEG quant table (SetJpegeQl) */

    /* --- Framesource features --- */
    bool has_rotation;              /* T31: sw rotation; T32/T40/T41: hw I2D */
    bool has_hw_rotation;           /* true only if rotation is hardware (I2D) */
    bool has_i2d;                   /* I2D transform block (T32/T40/T41) */
    bool has_fcrop;                 /* post-scaler frame crop (T23+) */
    bool has_mirror;                /* framesource-level mirror */

    /* --- Encoder features --- */
    bool has_set_default_param;     /* SetDefaultParam convenience (T31+) */
    bool has_bufshare;              /* JPEG/video buffer sharing (T31/T40/T41) */
    bool has_capped_rc;             /* CAPPED_VBR / CAPPED_QUALITY (T31+) */
    bool has_gop_attr;              /* fine-grained GOP control (T31+) */
    bool has_set_bitrate;           /* dynamic SetChnBitRate (T31+) */

    /* --- ISP features --- */
    bool has_multi_sensor;          /* IMPVI_NUM support (T32/T40/T41; T23 via _Sec) */
    bool has_defog;                 /* EnableDefog (T23/T31) */
    bool has_dpc;                   /* dead pixel correction (T23/T31) */
    bool has_drc;                   /* dynamic range compression (T21/T23/T31) */
    bool has_face_ae;               /* face-aware AE (T40/T41) */
    bool has_bcsh_hue;              /* SetBcshHue (T23/T31/T32/T40/T41) */
    bool has_sinter;                /* sinter denoise control (T20/T21/T23/T30/T31) */
    bool has_temper;                /* temper denoise control (T20/T21/T23/T30/T31) */

    /* --- Audio features --- */
    bool has_audio_alc_gain;        /* ALC PGA gain (T21/T31) */
    bool has_audio_digital_gain;    /* digital gain (T32/T40/T41) */
    bool has_audio_agc_mode;        /* AGC mode selection (T31 only) */
    bool has_audio_hpf_cutoff;      /* HPF cutoff frequency (T23+) */
    bool has_audio_howling_sup;     /* howling suppression (T23/T32/T40/T41) */

    /* --- OSD features --- */
    bool has_isp_osd;               /* ISP-level OSD (T23/T32/T40/T41) */
    bool has_osd_mosaic;            /* mosaic blur regions (T23/T32/T40/T41) */
    bool has_osd_pic_rmem;          /* PIC from reserved memory (T31+) */
    bool has_osd_callback;          /* per-frame OSD callback (T40/T41) */
} rss_hal_caps_t;
```

---

## 4. Operations Vtable

```c
/*
 * rss_hal_ops_t -- the HAL vtable.
 *
 * Every function returns int: 0 on success, negative errno on failure.
 * The `ctx` parameter is the opaque HAL context created by rss_hal_create().
 *
 * Functions marked "no-op on unsupported SoCs" silently return 0.
 */
typedef struct rss_hal_ops {

    /* ================================================================
     * SYSTEM LIFECYCLE
     * ================================================================ */

    /*
     * init -- initialize the hardware pipeline.
     *
     * Performs the full SDK init sequence:
     *   1. IMP_System_MemPoolRequest()  [T23/T32/T40/T41 only]
     *   2. IMP_System_Init()
     *   3. IMP_ISP_Open()
     *   4. IMP_ISP_AddSensor(sensor_cfg)
     *   5. IMP_ISP_EnableSensor()
     *   6. IMP_ISP_EnableTuning()
     *
     * Must be called before any other HAL operation.
     */
    int (*init)(void *ctx, const rss_sensor_config_t *sensor_cfg);

    /*
     * deinit -- tear down the hardware pipeline.
     *
     * Reverse of init:
     *   1. IMP_ISP_DisableTuning()
     *   2. IMP_ISP_DisableSensor()
     *   3. IMP_ISP_DelSensor()
     *   4. IMP_ISP_Close()
     *   5. IMP_System_Exit()
     *   6. IMP_System_MemPoolFree()     [T32 only]
     *
     * All channels/groups must be destroyed before calling this.
     */
    int (*deinit)(void *ctx);

    /*
     * get_caps -- query runtime capabilities.
     *
     * Returns a pointer to a static caps struct. The pointer is valid
     * for the lifetime of the HAL context.
     */
    const rss_hal_caps_t *(*get_caps)(void *ctx);

    /*
     * bind -- bind a pipeline source to a destination.
     *
     * Wraps IMP_System_Bind(). Must be called after framesource and
     * OSD/encoder groups are created, but BEFORE framesource is enabled.
     *
     * Common bindings:
     *   FS(0,0) -> ENC(0,0)            direct: framesource -> encoder
     *   FS(0,0) -> OSD(0,0)            with OSD: framesource -> OSD
     *   OSD(0,0) -> ENC(0,0)           with OSD: OSD -> encoder
     */
    int (*bind)(void *ctx, const rss_cell_t *src, const rss_cell_t *dst);

    /*
     * unbind -- unbind a pipeline connection.
     *
     * Wraps IMP_System_UnBind(). Must be called AFTER framesource is
     * disabled and BEFORE groups are destroyed.
     */
    int (*unbind)(void *ctx, const rss_cell_t *src, const rss_cell_t *dst);


    /* ================================================================
     * FRAMESOURCE
     * ================================================================ */

    /*
     * fs_create_channel -- create and configure a framesource channel.
     *
     * Wraps IMP_FrameSource_CreateChn() + IMP_FrameSource_SetChnAttr().
     * The HAL translates rss_fs_config_t into the SoC-appropriate
     * IMPFSChnAttr layout (4 distinct layouts across SoC generations).
     *
     * chn: channel number (0 = main stream, 1 = sub stream, 2 = third/JPEG)
     */
    int (*fs_create_channel)(void *ctx, int chn, const rss_fs_config_t *cfg);

    /*
     * fs_destroy_channel -- destroy a framesource channel.
     */
    int (*fs_destroy_channel)(void *ctx, int chn);

    /*
     * fs_enable_channel -- enable streaming on a framesource channel.
     *
     * Must be called AFTER all bind operations are complete.
     */
    int (*fs_enable_channel)(void *ctx, int chn);

    /*
     * fs_disable_channel -- disable streaming on a framesource channel.
     *
     * Must be called BEFORE unbind operations.
     */
    int (*fs_disable_channel)(void *ctx, int chn);

    /*
     * fs_set_rotation -- set rotation angle for a framesource channel.
     *
     * degrees: 0, 90, 180, 270.
     *
     * T31: software rotation via IMP_FrameSource_SetChnRotate().
     *       Must be called BEFORE fs_create_channel.
     *       Performance ceiling: ~1280x704 @ 15fps.
     * T32/T40/T41: hardware I2D via IMPFSI2DAttr.
     *       Can be called before or after fs_create_channel.
     * Other SoCs: returns -ENOTSUP.
     */
    int (*fs_set_rotation)(void *ctx, int chn, int degrees);

    /*
     * fs_set_fifo -- configure framesource output FIFO depth.
     *
     * Wraps IMP_FrameSource_SetChnFifoAttr().
     * depth: max number of buffered frames [1..n].
     */
    int (*fs_set_fifo)(void *ctx, int chn, int depth);


    /* ================================================================
     * ENCODER
     * ================================================================ */

    /*
     * enc_create_group -- create an encoder group.
     *
     * grp: group number (typically matches framesource channel number).
     */
    int (*enc_create_group)(void *ctx, int grp);

    /*
     * enc_destroy_group -- destroy an encoder group.
     *
     * All channels must be unregistered from the group first.
     */
    int (*enc_destroy_group)(void *ctx, int grp);

    /*
     * enc_create_channel -- create an encoder channel.
     *
     * THE critical abstraction point. The HAL translates rss_video_config_t
     * into the appropriate SDK struct:
     *
     *   Old SDK (T20/T21/T23/T30):
     *     IMPEncoderAttr { enType, bufSize, profile, picWidth, picHeight }
     *     + IMPEncoderRcAttr { outFrmRate, maxGop, attrRcMode { per-codec union } }
     *     -> IMPEncoderCHNAttr
     *
     *   New SDK (T31/T40/T41):
     *     Optionally call IMP_Encoder_SetDefaultParam()
     *     IMPEncoderEncAttr { eProfile, uWidth, uHeight, ... }
     *     + IMPEncoderRcAttr { attrRcMode { unified union }, outFrmRate }
     *     + IMPEncoderGopAttr { uGopLength, ... }
     *     -> IMPEncoderChnAttr
     *
     *   T32 hybrid:
     *     Old type name IMPEncoderCHNAttr but new-style internal structs.
     *     SetDefaultParam has extra uBufSize parameter.
     *
     * chn: channel number.
     * cfg: encoder configuration.
     */
    int (*enc_create_channel)(void *ctx, int chn, const rss_video_config_t *cfg);

    /*
     * enc_destroy_channel -- destroy an encoder channel.
     */
    int (*enc_destroy_channel)(void *ctx, int chn);

    /*
     * enc_register_channel -- register a channel to a group.
     *
     * A channel must be registered to a group before encoding can start.
     */
    int (*enc_register_channel)(void *ctx, int grp, int chn);

    /*
     * enc_unregister_channel -- unregister a channel from its group.
     */
    int (*enc_unregister_channel)(void *ctx, int chn);

    /*
     * enc_start -- start receiving pictures on a channel.
     *
     * Wraps IMP_Encoder_StartRecvPic().
     */
    int (*enc_start)(void *ctx, int chn);

    /*
     * enc_stop -- stop receiving pictures on a channel.
     *
     * Wraps IMP_Encoder_StopRecvPic().
     */
    int (*enc_stop)(void *ctx, int chn);

    /*
     * enc_poll -- poll for an encoded frame with timeout.
     *
     * Wraps IMP_Encoder_PollingStream().
     * Returns 0 if a frame is available, -ETIMEDOUT on timeout.
     *
     * timeout_ms: milliseconds to wait. 0 = non-blocking.
     */
    int (*enc_poll)(void *ctx, int chn, uint32_t timeout_ms);

    /*
     * enc_get_frame -- retrieve an encoded frame.
     *
     * THE critical abstraction. Wraps IMP_Encoder_GetStream() and
     * normalizes the data access pattern:
     *
     *   Old SDK: each pack has its own virAddr. Data is contiguous at
     *            pack[i].virAddr with length pack[i].length.
     *            NAL type: pack[i].dataType.h264Type / .h265Type
     *
     *   New SDK: stream has one virAddr + streamSize. Each pack has an
     *            offset into the ring buffer. Data may WRAP AROUND the
     *            ring buffer boundary: if offset + length > streamSize,
     *            the HAL must copy two parts to produce a linear buffer.
     *            NAL type: pack[i].nalType.h264NalType / .h265NalType
     *
     * The returned rss_frame_t contains linearized NAL unit data and
     * abstracted NAL types. The caller MUST call enc_release_frame()
     * after consuming the data.
     *
     * frame: output; populated by the HAL. Caller must not free.
     */
    int (*enc_get_frame)(void *ctx, int chn, rss_frame_t *frame);

    /*
     * enc_release_frame -- release an encoded frame back to the SDK.
     *
     * Wraps IMP_Encoder_ReleaseStream(). Must be called for every
     * successful enc_get_frame().
     */
    int (*enc_release_frame)(void *ctx, int chn, rss_frame_t *frame);

    /*
     * enc_request_idr -- request an IDR (keyframe) on next encode.
     *
     * Wraps IMP_Encoder_RequestIDR().
     */
    int (*enc_request_idr)(void *ctx, int chn);

    /*
     * enc_set_bitrate -- dynamically change encoder bitrate.
     *
     * New SDK (T31+): IMP_Encoder_SetChnBitRate(chn, bitrate, max_bitrate).
     * Old SDK: IMP_Encoder_SetChnAttrRcMode() with full RC struct rebuild.
     *
     * bitrate: target bitrate in bps.
     */
    int (*enc_set_bitrate)(void *ctx, int chn, uint32_t bitrate);

    /*
     * enc_set_gop -- dynamically change GOP length.
     *
     * New SDK (T31+): IMP_Encoder_SetChnGopLength(chn, gop_length).
     * Old SDK: IMP_Encoder_SetGOPSize() with IMPEncoderGOPSizeCfg.
     */
    int (*enc_set_gop)(void *ctx, int chn, uint32_t gop_length);

    /*
     * enc_set_fps -- dynamically change encoder frame rate.
     *
     * Wraps IMP_Encoder_SetChnFrmRate().
     * fps_num/fps_den: rational frame rate (e.g. 25/1, 15/1).
     */
    int (*enc_set_fps)(void *ctx, int chn, uint32_t fps_num, uint32_t fps_den);

    /*
     * enc_set_bufshare -- enable buffer sharing between video and JPEG channels.
     *
     * New SDK only (T31/T40/T41). Wraps IMP_Encoder_SetbufshareChn().
     * Allows the JPEG snapshot channel to share the stream buffer with
     * an H264/H265 channel, reducing rmem consumption.
     *
     * Returns 0 on success, -ENOTSUP on old SDK SoCs (no-op).
     */
    int (*enc_set_bufshare)(void *ctx, int src_chn, int dst_chn);


    /* ================================================================
     * ISP TUNING
     * ================================================================ */

    /*
     * ISP basic image controls.
     *
     * Each function handles the three-generation signature dispatch:
     *   Gen1 (T20/T21/T30):   SetBrightness(val)
     *   Gen2 (T23/T31):       SetBrightness(val)
     *   Gen3 (T32/T40/T41):   SetBrightness(IMPVI_NUM, &val)
     *
     * val: [0..255], 128 = neutral.
     */
    int (*isp_set_brightness)(void *ctx, uint8_t val);
    int (*isp_set_contrast)(void *ctx, uint8_t val);
    int (*isp_set_saturation)(void *ctx, uint8_t val);
    int (*isp_set_sharpness)(void *ctx, uint8_t val);

    /*
     * isp_set_hue -- set BCSH hue.
     *
     * Only available on T23/T31/T32/T40/T41 (has_bcsh_hue).
     * Returns -ENOTSUP on T20/T21/T30.
     */
    int (*isp_set_hue)(void *ctx, uint8_t val);

    /*
     * isp_set_hflip / isp_set_vflip -- set ISP horizontal/vertical flip.
     *
     * Dispatches to the appropriate flip API:
     *   T20/T30: SetISPHflip + SetISPVflip (separate calls)
     *   T21: SetISPHflip + SetISPVflip (no combined)
     *   T23/T31: SetHVFLIP(IMPISPHVFLIP enum)
     *   T32/T40/T41: SetHVFLIP(IMPVI_NUM, ptr)
     *
     * The HAL internally combines hflip + vflip state and issues a
     * single SDK call on SoCs with a combined flip function.
     *
     * enable: 0 = normal, 1 = flipped.
     */
    int (*isp_set_hflip)(void *ctx, int enable);
    int (*isp_set_vflip)(void *ctx, int enable);

    /*
     * isp_set_running_mode -- set ISP day/night mode.
     *
     * Dispatches to IMP_ISP_Tuning_SetISPRunningMode().
     * Gen1/Gen2: scalar; Gen3: IMPVI_NUM + pointer.
     */
    int (*isp_set_running_mode)(void *ctx, rss_isp_mode_t mode);

    /*
     * isp_set_sensor_fps -- set sensor output frame rate.
     *
     * Dispatches to IMP_ISP_Tuning_SetSensorFPS():
     *   Gen1/Gen2: (fps_num, fps_den)
     *   T40: (IMPVI_NUM, &fps_num, &fps_den)
     *   T32/T41: (IMPVI_NUM, &IMPISPSensorFps)
     */
    int (*isp_set_sensor_fps)(void *ctx, uint32_t fps_num, uint32_t fps_den);

    /*
     * isp_set_antiflicker -- set anti-flicker mode.
     *
     * Dispatches to IMP_ISP_Tuning_SetAntiFlickerAttr():
     *   Gen1/Gen2: by-value enum
     *   Gen3: IMPVI_NUM + pointer
     */
    int (*isp_set_antiflicker)(void *ctx, rss_antiflicker_t mode);

    /*
     * isp_set_wb -- set white balance mode and gains.
     *
     * Old SDK: IMP_ISP_Tuning_SetWB()
     * New SDK: IMP_ISP_Tuning_SetAwbAttr()
     */
    int (*isp_set_wb)(void *ctx, const rss_wb_config_t *wb_cfg);

    /*
     * isp_get_exposure -- get current exposure parameters (for RIC).
     *
     * Old SDK: IMP_ISP_Tuning_GetExpr() + GetTotalGain()
     * New SDK (T32/T40/T41): IMP_ISP_Tuning_GetAeExprInfo()
     */
    int (*isp_get_exposure)(void *ctx, rss_exposure_t *exposure);

    /*
     * Optional ISP tuning functions.
     *
     * These are present on a subset of SoCs. When called on unsupported
     * SoCs, they return -ENOTSUP.
     *
     * set_sinter_strength: spatial denoising [0..255]. T20/T21/T23/T30/T31.
     * set_temper_strength: temporal denoising [0..255]. T20/T21/T23/T30/T31.
     * set_defog: defog enable/disable. T23/T31.
     * set_dpc_strength: dead pixel correction [0..255]. T23/T31.
     * set_drc_strength: DRC strength [0..255]. T21/T23/T31.
     * set_ae_comp: AE compensation [0..255]. T20/T23/T30/T31.
     * set_max_again: max analog gain. T20-T31.
     * set_max_dgain: max digital gain. T20-T31.
     * set_highlight_depress: highlight suppression [0..255]. T20-T31.
     */
    int (*isp_set_sinter_strength)(void *ctx, uint8_t val);
    int (*isp_set_temper_strength)(void *ctx, uint8_t val);
    int (*isp_set_defog)(void *ctx, int enable);
    int (*isp_set_dpc_strength)(void *ctx, uint8_t val);
    int (*isp_set_drc_strength)(void *ctx, uint8_t val);
    int (*isp_set_ae_comp)(void *ctx, int val);
    int (*isp_set_max_again)(void *ctx, uint32_t gain);
    int (*isp_set_max_dgain)(void *ctx, uint32_t gain);
    int (*isp_set_highlight_depress)(void *ctx, uint8_t val);


    /* ================================================================
     * AUDIO
     * ================================================================ */

    /*
     * audio_init -- initialize the audio input subsystem.
     *
     * Performs the full audio init sequence:
     *   1. IMP_AI_SetPubAttr(dev, &attr)
     *   2. IMP_AI_Enable(dev)
     *   3. IMP_AI_SetChnParam(dev, chn, &param)
     *   4. IMP_AI_EnableChn(dev, chn)
     *   5. IMP_AI_SetVol(dev, chn, vol)
     *   6. IMP_AI_SetGain(dev, chn, gain)
     *
     * Handles IMPAudioIChnParam struct differences (aecChn field
     * on T23/T32/T40/T41 vs absent on T20/T21/T30/T31).
     */
    int (*audio_init)(void *ctx, const rss_audio_config_t *cfg);

    /*
     * audio_deinit -- tear down the audio subsystem.
     *
     *   1. IMP_AI_DisableChn(dev, chn)
     *   2. IMP_AI_Disable(dev)
     */
    int (*audio_deinit)(void *ctx);

    /*
     * audio_set_volume -- set audio input/output software volume.
     *
     * dev: 0 = AI device, 1 = AO device
     * chn: channel number (typically 0)
     * vol: [-30..120], 60 = unity, -30 = mute
     */
    int (*audio_set_volume)(void *ctx, int dev, int chn, int vol);

    /*
     * audio_set_gain -- set audio input/output hardware gain.
     *
     * dev: 0 = AI device, 1 = AO device
     * chn: channel number
     * gain: AI [0..31], AO [0..31] (exact dB mapping is SoC-specific)
     */
    int (*audio_set_gain)(void *ctx, int dev, int chn, int gain);

    /*
     * audio_enable_ns / audio_disable_ns -- noise suppression.
     *
     * level: noise suppression aggressiveness.
     */
    int (*audio_enable_ns)(void *ctx, rss_ns_level_t level);
    int (*audio_disable_ns)(void *ctx);

    /*
     * audio_enable_hpf / audio_disable_hpf -- high-pass filter.
     */
    int (*audio_enable_hpf)(void *ctx);
    int (*audio_disable_hpf)(void *ctx);

    /*
     * audio_enable_agc / audio_disable_agc -- automatic gain control.
     *
     * Uses IMPAudioAgcConfig internally.
     */
    int (*audio_enable_agc)(void *ctx, const rss_agc_config_t *cfg);
    int (*audio_disable_agc)(void *ctx);

    /*
     * audio_read_frame -- read one audio frame from the input device.
     *
     * Wraps IMP_AI_PollingFrame() + IMP_AI_GetFrame() + IMP_AI_ReleaseFrame().
     *
     * dev: device number (typically 0)
     * chn: channel number (typically 0)
     * frame: output; populated by the HAL
     * block: true = blocking, false = non-blocking
     *
     * The HAL calls PollingFrame, then GetFrame. The frame data is
     * valid until the next audio_read_frame() call.
     */
    int (*audio_read_frame)(void *ctx, int dev, int chn,
                            rss_audio_frame_t *frame, bool block);

    /*
     * audio_register_encoder -- register a custom audio encoder.
     *
     * Wraps IMP_AENC_RegisterEncoder(). Returns 0 on success.
     * *handle: output; encoder handle for unregister.
     */
    int (*audio_register_encoder)(void *ctx, const rss_audio_encoder_t *enc,
                                  int *handle);

    /*
     * audio_unregister_encoder -- unregister a custom audio encoder.
     */
    int (*audio_unregister_encoder)(void *ctx, int handle);


    /* ================================================================
     * OSD
     * ================================================================ */

    /*
     * osd_set_pool_size -- set OSD reserved memory pool size.
     *
     * Wraps IMP_OSD_SetPoolSize(). Must be called before creating
     * any OSD regions that use reserved memory (PIC_RMEM).
     *
     * bytes: pool size in bytes.
     */
    int (*osd_set_pool_size)(void *ctx, uint32_t bytes);

    /*
     * osd_create_group / osd_destroy_group -- OSD group lifecycle.
     *
     * One OSD group per encoder channel. Groups are bound between
     * FrameSource and Encoder in the pipeline.
     *
     * grp: group number (matches encoder group).
     */
    int (*osd_create_group)(void *ctx, int grp);
    int (*osd_destroy_group)(void *ctx, int grp);

    /*
     * osd_create_region -- create an OSD region.
     *
     * Wraps IMP_OSD_CreateRgn(). The HAL translates rss_osd_region_t
     * into vendor IMPOSDRgnAttr, handling enum value differences:
     *   Classic SDK: OSD_REG_PIC = 5, OSD_REG_COVER = 4
     *   Extended SDK: OSD_REG_PIC = 7, OSD_REG_COVER = 6
     *
     * *handle: output; region handle for subsequent operations.
     * attr: region definition (type, position, size, bitmap data).
     */
    int (*osd_create_region)(void *ctx, int *handle,
                             const rss_osd_region_t *attr);

    /*
     * osd_destroy_region -- destroy an OSD region.
     */
    int (*osd_destroy_region)(void *ctx, int handle);

    /*
     * osd_register_region -- register a region to an OSD group.
     *
     * Wraps IMP_OSD_RegisterRgn(). Sets up group-level attributes
     * (alpha, position, layer) from the rss_osd_region_t.
     */
    int (*osd_register_region)(void *ctx, int handle, int grp);

    /*
     * osd_unregister_region -- unregister a region from an OSD group.
     */
    int (*osd_unregister_region)(void *ctx, int handle, int grp);

    /*
     * osd_set_region_attr -- update region attributes.
     *
     * Wraps IMP_OSD_SetRgnAttr(). Use to change type, position,
     * size, or bitmap format of an existing region.
     */
    int (*osd_set_region_attr)(void *ctx, int handle,
                               const rss_osd_region_t *attr);

    /*
     * osd_update_region_data -- update region pixel data.
     *
     * Wraps IMP_OSD_UpdateRgnAttrData(). Hot-path for timestamp
     * updates: only changes the bitmap data pointer, not the
     * region geometry.
     *
     * data: BGRA pixel data. Must remain valid until next call.
     */
    int (*osd_update_region_data)(void *ctx, int handle,
                                  const uint8_t *data);

    /*
     * osd_show_region -- show or hide a region in a group.
     *
     * Wraps IMP_OSD_ShowRgn().
     * show: 1 = visible, 0 = hidden.
     */
    int (*osd_show_region)(void *ctx, int handle, int grp, int show);


    /* ================================================================
     * GPIO / IR-CUT
     * ================================================================ */

    /*
     * gpio_set -- set a GPIO pin output value.
     *
     * Uses /sys/class/gpio or direct register access.
     * pin: GPIO number.
     * value: 0 or 1.
     */
    int (*gpio_set)(void *ctx, int pin, int value);

    /*
     * gpio_get -- read a GPIO pin input value.
     *
     * pin: GPIO number.
     * *value: output; 0 or 1.
     */
    int (*gpio_get)(void *ctx, int pin, int *value);

    /*
     * ircut_set -- set IR-cut filter state.
     *
     * state: 0 = IR-cut engaged (day mode), 1 = IR-cut open (night mode).
     *
     * The HAL drives the IR-cut GPIO pair using the pin configuration
     * from the system config.
     */
    int (*ircut_set)(void *ctx, int state);

} rss_hal_ops_t;
```

---

## 5. Context Struct

```c
/*
 * rss_hal_ctx_t -- opaque HAL context.
 *
 * Holds all platform state: SDK handles, capability cache,
 * flip state (for combining hflip + vflip into one SDK call),
 * ring-buffer scratch memory for frame linearization, etc.
 *
 * Created by rss_hal_create(). Destroyed by rss_hal_destroy().
 *
 * Implementation detail (not exposed to consumers):
 *
 *   struct rss_hal_ctx {
 *       const rss_hal_ops_t  *ops;           // vtable pointer
 *       rss_hal_caps_t        caps;           // cached capabilities
 *       rss_sensor_config_t   sensor;         // sensor config used at init
 *
 *       // Flip state (needed for SoCs with combined H/V flip)
 *       int                   hflip_state;
 *       int                   vflip_state;
 *
 *       // Frame linearization scratch buffer (for new SDK ring-buffer wrap)
 *       uint8_t              *scratch_buf;
 *       size_t                scratch_size;
 *
 *       // Per-channel NAL unit arrays (reused across get_frame calls)
 *       rss_nal_unit_t       *nal_arrays[RSS_MAX_ENC_CHANNELS];
 *       int                   nal_array_caps[RSS_MAX_ENC_CHANNELS];
 *
 *       // Platform-specific opaque data (per-SoC impl can extend)
 *       void                 *platform;
 *   };
 *
 *   #define RSS_MAX_ENC_CHANNELS  8
 */
typedef struct rss_hal_ctx rss_hal_ctx_t;
```

---

## 6. Factory Functions

```c
/*
 * rss_hal_create -- create and return a HAL context.
 *
 * The implementation is selected at compile time. Each SoC family
 * (T20/T21/T30, T23, T31, T32, T40, T41) provides its own
 * implementation of the ops vtable.
 *
 * Returns NULL on allocation failure.
 *
 * Compile-time selection via -DRSS_SOC=T31 (or T20, T23, T32, T40, T41):
 *
 *   #if RSS_SOC == T20 || RSS_SOC == T21 || RSS_SOC == T30
 *       extern const rss_hal_ops_t rss_hal_ops_gen1;
 *       // ...
 *   #elif RSS_SOC == T23
 *       extern const rss_hal_ops_t rss_hal_ops_t23;
 *   #elif RSS_SOC == T31
 *       extern const rss_hal_ops_t rss_hal_ops_t31;
 *   #elif RSS_SOC == T32
 *       extern const rss_hal_ops_t rss_hal_ops_t32;
 *   #elif RSS_SOC == T40
 *       extern const rss_hal_ops_t rss_hal_ops_t40;
 *   #elif RSS_SOC == T41
 *       extern const rss_hal_ops_t rss_hal_ops_t41;
 *   #endif
 */
rss_hal_ctx_t *rss_hal_create(void);

/*
 * rss_hal_destroy -- destroy a HAL context.
 *
 * Frees the context and all internal resources. Does NOT call
 * deinit() -- the caller must do that first.
 */
void rss_hal_destroy(rss_hal_ctx_t *ctx);

/*
 * rss_hal_get_ops -- get the ops vtable from a context.
 *
 * Returns a pointer to the ops struct. Shortcut for ctx->ops.
 */
const rss_hal_ops_t *rss_hal_get_ops(rss_hal_ctx_t *ctx);
```

---

## 7. Convenience Macros

```c
/*
 * HAL call macros -- syntactic sugar for invoking ops through context.
 *
 * Usage:
 *   rss_hal_ctx_t *hal = rss_hal_create();
 *   RSS_HAL_CALL(hal, init, &sensor_cfg);
 *   RSS_HAL_CALL(hal, fs_create_channel, 0, &fs_cfg);
 */
#define RSS_HAL_CALL(ctx, func, ...) \
    ((ctx)->ops->func((ctx), ##__VA_ARGS__))

/*
 * Error codes returned by HAL functions.
 * These map to standard errno values.
 */
#define RSS_OK          0
#define RSS_EINVAL      (-EINVAL)       /* invalid argument */
#define RSS_ENOTSUP     (-ENOTSUP)      /* feature not supported on this SoC */
#define RSS_ENOMEM      (-ENOMEM)       /* memory allocation failed */
#define RSS_ETIMEDOUT   (-ETIMEDOUT)    /* operation timed out */
#define RSS_EIO         (-EIO)          /* SDK call failed */
#define RSS_EBUSY       (-EBUSY)        /* resource in use */
#define RSS_ENOENT      (-ENOENT)       /* resource not found */

#ifdef __cplusplus
}
#endif

#endif /* RAPTOR_HAL_H */
```

---

## 8. Per-SoC Implementation File Structure

Each SoC family provides a source file implementing all ops:

```
hal/
    raptor_hal.h                 <-- this file (master header)
    hal_common.c                 <-- rss_hal_create/destroy, shared utilities
    hal_gen1.c                   <-- T20, T21, T30 implementation
    hal_t23.c                    <-- T23 implementation (dual-sensor via _Sec)
    hal_t31.c                    <-- T31 implementation
    hal_t32.c                    <-- T32 implementation (hybrid SDK)
    hal_t40.c                    <-- T40 implementation
    hal_t41.c                    <-- T41 implementation
```

The build system compiles exactly ONE `hal_*.c` file based on the
`RSS_SOC` define. Each file populates a `const rss_hal_ops_t` struct
and registers it via `rss_hal_create()`.

---

## 9. SoC Implementation Notes

### 9.1 Key Dispatch Patterns

Every per-SoC implementation must handle these critical differences:

**enc_get_frame() -- stream data access:**

```
Gen1 (T20/T21/T23/T30):
  Data at pack[i].virAddr, length pack[i].length.
  NAL type at pack[i].dataType.h264Type or .h265Type.
  No ring-buffer wrap.

Gen2+ (T31/T32/T40/T41):
  Data at stream.virAddr + pack[i].offset, length pack[i].length.
  NAL type at pack[i].nalType.h264NalType or .h265NalType.
  MUST handle ring-buffer wrap: if offset + length > streamSize,
  copy in two parts to scratch buffer and point NAL data there.
```

**enc_create_channel() -- struct translation:**

```
Old SDK (T20/T21/T23/T30):
  Fill IMPEncoderAttr.enType = PT_H264/PT_JPEG/PT_H265
  Fill per-codec RC union: attrH264Cbr.outBitRate, .maxQp, .minQp, etc.
  IMPEncoderRcAttr embeds outFrmRate + maxGop + denoise + hskip

New SDK (T31/T40/T41):
  Fill IMPEncoderEncAttr.eProfile = IMP_ENC_PROFILE_AVC_HIGH (etc.)
  Fill unified IMPEncoderAttrCbr.uTargetBitRate, .iMinQP, .iMaxQP
  Separate IMPEncoderGopAttr with uGopLength, uGopCtrlMode

T32 hybrid:
  Uses IMPEncoderCHNAttr name but IMPEncoderEncAttr internal struct.
  SetDefaultParam takes extra uBufSize parameter.
```

**ISP tuning -- signature dispatch:**

```
Gen1 (T20/T21/T30):     func(scalar_value)
Gen2 (T23/T31):          func(scalar_value)
Gen3 (T40):              func(IMPVI_NUM, &value_ptr)
Gen3b (T32/T41):         func(IMPVI_NUM, &value_ptr)  [some use struct wrappers]
```

### 9.2 Feature Availability Matrix

| Feature | T20 | T21 | T23 | T30 | T31 | T32 | T40 | T41 |
|---------|-----|-----|-----|-----|-----|-----|-----|-----|
| H.265 | - | ? | ? | Y | Y | Y | Y | Y |
| Rotation | - | - | - | - | SW | HW | HW | HW |
| Buffer share | - | - | - | - | Y | - | Y | Y |
| Capped RC | - | - | - | - | Y | Y | Y | Y |
| GOP attr | - | - | - | - | Y | Y | Y | Y |
| Dynamic bitrate | - | - | - | - | Y | Y | Y | Y |
| SetDefaultParam | - | - | - | - | Y | Y | Y | Y |
| BCSH Hue | - | - | Y | - | Y | Y | Y | Y |
| Multi-sensor | - | - | Sec | - | - | Y | Y | Y |
| ISP OSD | - | - | Y | - | - | Y | Y | Y |
| OSD mosaic | - | - | Y | - | - | Y | Y | Y |
| Audio dig. gain | - | - | - | - | - | Y | Y | Y |
| Audio HPF cutoff | - | - | Y | - | Y | Y | Y | Y |
| Frame crop (fcrop) | - | - | Y | - | Y | Y | Y | Y |

### 9.3 Unsupported Feature Behavior

When a daemon calls a HAL function for an unsupported feature:

1. **Has capability query**: the daemon should check `caps.has_*` first.
2. **No-op safe**: functions like `enc_set_bufshare()` and `fs_set_rotation()`
   return `-ENOTSUP` on unsupported SoCs. The daemon handles this gracefully.
3. **Silent fallback**: `rss_rc_mode_t` values that don't exist on old SDK
   (CAPPED_VBR, CAPPED_QUALITY) are silently mapped to VBR by the HAL.
   No error is returned; `get_caps()` reflects the actual mode in use.

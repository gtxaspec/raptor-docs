# Raptor HAL: imp_common.h Cross-SoC Differences

Source headers analyzed:

| SoC | SDK Version | Language |
|-----|-------------|----------|
| T20 | 3.12.0 | zh |
| T21 | 1.0.33 | zh |
| T23 | 1.3.0 | en |
| T30 | 1.0.5 | zh |
| T31 | 1.1.6 | en |
| T32 | 1.0.6 | en |
| T40 | 1.3.1 | en |
| T41 | 1.2.5 | en |

---

## 1. IMPDeviceID Enum

Identical across all 8 SoCs:

```c
typedef enum {
    DEV_ID_FS,              // Video Source (FrameSource)
    DEV_ID_ENC,             // Encoder
    DEV_ID_DEC,             // Decoder
    DEV_ID_IVS,             // Algorithm (IVS)
    DEV_ID_OSD,             // Image Overlay (OSD)
    DEV_ID_FG1DIRECT,       // FB FG1Direct
    DEV_ID_RESERVED_START,
    DEV_ID_RESERVED_END = 23,
    NR_MAX_DEVICES,
} IMPDeviceID;
```

### HAL Implications

No abstraction needed. The device ID space is stable across all generations.

---

## 2. IMPCell Struct

Identical across all 8 SoCs:

```c
typedef struct {
    IMPDeviceID deviceID;
    int         groupID;
    int         outputID;
} IMPCell;
```

### HAL Implications

No abstraction needed. Stable across all generations.

---

## 3. IMPFrameInfo Struct

This struct varies significantly across SoC generations. The base fields are shared, but later SoCs add DMA-related fields, pool pointers, and OSD flags.

| Field | Type | T20 | T21 | T23 | T30 | T31 | T32 | T40 | T41 |
|-------|------|-----|-----|-----|-----|-----|-----|-----|-----|
| `index` | `int` | Y | Y | Y | Y | Y | Y | Y | Y |
| `pool_idx` | `int` | Y | Y | Y | Y | Y | Y | Y | Y |
| `width` | `uint32_t` | Y | Y | Y | Y | Y | Y | Y | Y |
| `height` | `uint32_t` | Y | Y | Y | Y | Y | Y | Y | Y |
| `pixfmt` | `uint32_t` | Y | Y | Y | Y | Y | Y | Y | Y |
| `size` | `uint32_t` | Y | Y | Y | Y | Y | Y | Y | Y |
| `phyAddr` | `uint32_t` | Y | Y | Y | Y | Y | Y | Y | Y |
| `virAddr` | `uint32_t` | Y | Y | Y | Y | Y | Y | Y | Y |
| `direct_phyAddr` | `uint32_t` | - | - | Y | - | - | Y | Y | Y |
| `pool` | `void*` | - | - | - | - | - | Y | Y | - |
| `timeStamp` | `int64_t` | Y | Y | Y | Y | Y | Y | Y | Y |
| `timeStamp_ivdc` | `int64_t` | - | - | Y | - | - | - | - | - |
| `rotate_osdflag` | `int` | - | - | - | - | Y | - | - | - |
| `priv[0]` | `uint32_t` | Y | Y | Y | Y | Y | Y | Y | Y |

### Generation Breakdown

**Generation 1 (T20, T21, T30):** Baseline struct -- `index`, `pool_idx`, `width`, `height`, `pixfmt`, `size`, `phyAddr`, `virAddr`, `timeStamp`, `priv[0]`.

```c
// T20, T21, T30
typedef struct {
    int index;
    int pool_idx;
    uint32_t width;
    uint32_t height;
    uint32_t pixfmt;
    uint32_t size;
    uint32_t phyAddr;
    uint32_t virAddr;
    int64_t timeStamp;
    uint32_t priv[0];
} IMPFrameInfo;
```

**T23:** Adds `direct_phyAddr` and `timeStamp_ivdc` (IVDC dequeue timestamp).

```c
// T23
typedef struct {
    int index;
    int pool_idx;
    uint32_t width;
    uint32_t height;
    uint32_t pixfmt;
    uint32_t size;
    uint32_t phyAddr;
    uint32_t virAddr;
    uint32_t direct_phyAddr;
    int64_t timeStamp;
    int64_t timeStamp_ivdc;
    uint32_t priv[0];
} IMPFrameInfo;
```

**T31:** Adds `rotate_osdflag` (rotation/OSD flag), no `direct_phyAddr` or `pool`.

```c
// T31
typedef struct {
    int index;
    int pool_idx;
    uint32_t width;
    uint32_t height;
    uint32_t pixfmt;
    uint32_t size;
    uint32_t phyAddr;
    uint32_t virAddr;
    int64_t timeStamp;
    int rotate_osdflag;
    uint32_t priv[0];
} IMPFrameInfo;
```

**T32, T40:** Adds `direct_phyAddr` and `pool` (void pointer to memory pool object).

```c
// T32, T40
typedef struct {
    int index;
    int pool_idx;
    uint32_t width;
    uint32_t height;
    uint32_t pixfmt;
    uint32_t size;
    uint32_t phyAddr;
    uint32_t virAddr;
    uint32_t direct_phyAddr;
    void* pool;
    int64_t timeStamp;
    uint32_t priv[0];
} IMPFrameInfo;
```

**T41:** Has `direct_phyAddr` but no `pool` pointer.

```c
// T41
typedef struct {
    int index;
    int pool_idx;
    uint32_t width;
    uint32_t height;
    uint32_t pixfmt;
    uint32_t size;
    uint32_t phyAddr;
    uint32_t virAddr;
    uint32_t direct_phyAddr;
    void* pool;
    int64_t timeStamp;
    uint32_t priv[0];
} IMPFrameInfo;
```

Note: T41 struct is actually identical to T32/T40 (has both `direct_phyAddr` and `pool`).

### HAL Implications

The HAL must use a superset struct or per-SoC struct definitions. Key concerns:

- **`direct_phyAddr`**: Present on T23, T32, T40, T41. Needed for direct DMA access. The HAL should expose this conditionally or always include it (zeroed when unavailable).
- **`pool`**: Present on T32, T40, T41. A `void*` to the SDK's internal memory pool. The HAL must not expose this to userspace but needs it for frame lifecycle management.
- **`timeStamp_ivdc`**: T23-only. Used for IVDC pipeline timing. Can be ignored in the HAL unless IVDC decoding is supported.
- **`rotate_osdflag`**: T31-only. Indicates frame rotation and OSD overlay state. The HAL should query this on T31 for rotation-aware rendering.
- **Field ordering matters**: The struct layout differs, which means casting between SoC variants is unsafe. The HAL should have per-SoC accessor functions or a normalized frame descriptor.

---

## 4. IMPFrameTimestamp Struct

Identical across all 8 SoCs:

```c
typedef struct {
    uint64_t ts;     // timestamp
    uint64_t minus;  // lower bound
    uint64_t plus;   // upper bound
} IMPFrameTimestamp;
```

### HAL Implications

No abstraction needed.

---

## 5. IMPPayloadType Enum

| Value | T20 | T21 | T23 | T30 | T31 | T32 | T40 | T41 |
|-------|-----|-----|-----|-----|-----|-----|-----|-----|
| `PT_JPEG` | Y | Y | Y | Y | Y | Y | Y | Y |
| `PT_H264` | Y | Y | Y | Y | Y | Y | Y | Y |
| `PT_H265` | - | Y | Y | Y | Y | Y | Y | Y |

### HAL Implications

**T20 does not support H.265 (HEVC).** The HAL must check SoC type before allowing H.265 encoder channel creation. All other SoCs (T21+) support all three payload types. The enum values are sequential (JPEG=0, H264=1, H265=2), so the HAL can use the same enum but must gate H.265 on T20.

---

## 6. IMPPixelFormat Enum

The base set of pixel formats is shared across all SoCs. The differences are in extended formats added after `PIX_FMT_NB`.

### Base formats (all 8 SoCs)

All SoCs define these formats with identical enum values (0 through `PIX_FMT_NB`):

| Enum Value | Format |
|------------|--------|
| `PIX_FMT_YUV420P` | Planar YUV 4:2:0, 12bpp |
| `PIX_FMT_YUYV422` | Packed YUV 4:2:2, 16bpp (Y0 Cb Y1 Cr) |
| `PIX_FMT_UYVY422` | Packed YUV 4:2:2, 16bpp (Cb Y0 Cr Y1) |
| `PIX_FMT_YUV422P` | Planar YUV 4:2:2, 16bpp |
| `PIX_FMT_YUV444P` | Planar YUV 4:4:4, 24bpp |
| `PIX_FMT_YUV410P` | Planar YUV 4:1:0, 9bpp |
| `PIX_FMT_YUV411P` | Planar YUV 4:1:1, 12bpp |
| `PIX_FMT_GRAY8` | Y, 8bpp |
| `PIX_FMT_MONOWHITE` | Y, 1bpp (0=white) |
| `PIX_FMT_MONOBLACK` | Y, 1bpp (0=black) |
| `PIX_FMT_NV12` | Semi-planar YUV 4:2:0, 12bpp (UV interleaved) |
| `PIX_FMT_NV21` | Semi-planar YUV 4:2:0, 12bpp (VU interleaved) |
| `PIX_FMT_RGB24` | Packed RGB 8:8:8, 24bpp |
| `PIX_FMT_BGR24` | Packed BGR 8:8:8, 24bpp |
| `PIX_FMT_ARGB` | Packed ARGB 8:8:8:8, 32bpp |
| `PIX_FMT_RGBA` | Packed RGBA 8:8:8:8, 32bpp |
| `PIX_FMT_ABGR` | Packed ABGR 8:8:8:8, 32bpp |
| `PIX_FMT_BGRA` | Packed BGRA 8:8:8:8, 32bpp |
| `PIX_FMT_RGB565BE` | Packed RGB 5:6:5, 16bpp, big-endian |
| `PIX_FMT_RGB565LE` | Packed RGB 5:6:5, 16bpp, little-endian |
| `PIX_FMT_RGB555BE` | Packed RGB 5:5:5, 16bpp, big-endian |
| `PIX_FMT_RGB555LE` | Packed RGB 5:5:5, 16bpp, little-endian |
| `PIX_FMT_BGR565BE` | Packed BGR 5:6:5, 16bpp, big-endian |
| `PIX_FMT_BGR565LE` | Packed BGR 5:6:5, 16bpp, little-endian |
| `PIX_FMT_BGR555BE` | Packed BGR 5:5:5, 16bpp, big-endian |
| `PIX_FMT_BGR555LE` | Packed BGR 5:5:5, 16bpp, little-endian |
| `PIX_FMT_0RGB` | Packed 0RGB, 32bpp |
| `PIX_FMT_RGB0` | Packed RGB0, 32bpp |
| `PIX_FMT_0BGR` | Packed 0BGR, 32bpp |
| `PIX_FMT_BGR0` | Packed BGR0, 32bpp |
| `PIX_FMT_BAYER_BGGR8` | Bayer BGGR, 8bpp |
| `PIX_FMT_BAYER_RGGB8` | Bayer RGGB, 8bpp |
| `PIX_FMT_BAYER_GBRG8` | Bayer GBRG, 8bpp |
| `PIX_FMT_BAYER_GRBG8` | Bayer GRBG, 8bpp |
| `PIX_FMT_RAW` | Raw sensor data |
| `PIX_FMT_HSV` | HSV color space |
| `PIX_FMT_NB` | Sentinel / count of pixel formats |

### Extended formats (after PIX_FMT_NB)

| Format | T20 | T21 | T23 | T30 | T31 | T32 | T40 | T41 |
|--------|-----|-----|-----|-----|-----|-----|-----|-----|
| `PIX_FMT_YUV422` | - | - | - | - | - | Y | Y | Y |
| `PIX_FMT_YVU422` | - | - | - | - | - | Y | Y | Y |
| `PIX_FMT_UVY422` | - | - | - | - | - | Y | Y | Y |
| `PIX_FMT_VUY422` | - | - | - | - | - | Y | Y | Y |
| `PIX_FMT_RAW8` | - | - | - | - | - | Y | Y | Y |
| `PIX_FMT_RAW16` | - | - | - | - | - | Y | Y | Y |
| `PIX_FMT_2BIT` | - | - | - | - | - | - | Y | - |

Note: `PIX_FMT_NB` is supposed to be the count/sentinel, but on T32/T40/T41 additional formats are added **after** it, breaking the sentinel convention. This means `PIX_FMT_NB` is no longer a valid count on these SoCs.

### HAL Implications

- The HAL must define its own pixel format enum that is a superset of all SoC variants.
- **`PIX_FMT_NB` is unreliable** as a sentinel on T32+. Do not use it for array sizing.
- The extended 422 variants (`YUV422`, `YVU422`, `UVY422`, `VUY422`) are semiplanar formats only available on T32/T40/T41. They differ from the packed `YUYV422`/`UYVY422` which are available everywhere.
- `PIX_FMT_RAW8` and `PIX_FMT_RAW16` (T32+) are for raw sensor readout at specific bit depths. `PIX_FMT_RAW` (all SoCs) is a generic raw format.
- `PIX_FMT_2BIT` is T40-only (likely for 2-bit grayscale or mask data).
- The HAL should validate pixel format support per-SoC before passing to SDK functions.

---

## 7. IMPPoint Struct

Identical across all 8 SoCs:

```c
typedef struct {
    int x;
    int y;
} IMPPoint;
```

### HAL Implications

No abstraction needed.

---

## 8. IMPRect Struct

Identical across all 8 SoCs:

```c
typedef struct {
    IMPPoint p0;   // upper-left corner
    IMPPoint p1;   // lower-right corner
} IMPRect;
```

Note: Width = `abs(p1.x - p0.x) + 1`, Height = `abs(p1.y - p0.y) + 1`. Coordinates are inclusive.

### HAL Implications

No abstraction needed, but the HAL should document the inclusive coordinate convention clearly since it differs from most graphics APIs.

---

## 9. IMPLine Struct

| SoC | Has IMPLine |
|-----|-------------|
| T20 | No |
| T21 | No |
| T23 | Yes |
| T30 | No |
| T31 | Yes |
| T32 | Yes |
| T40 | Yes |
| T41 | Yes |

```c
typedef struct {
    IMPPoint p0;   // line endpoint
} IMPLine;
```

Note: `IMPLine` contains only a single point `p0`. The struct comment describes it as "Horizontal line: Left end point / Vertical line: Right end point". This is used for IVS line-crossing detection where lines are axis-aligned and the other endpoint is implied.

### HAL Implications

- Only available on T23, T31, T32, T40, T41.
- T20, T21, T30 do not have this type. If the HAL uses line-crossing detection, it must provide its own IMPLine definition for older SoCs or gate the feature.

---

## 10. calc_pic_size() Inline Function

This function computes frame buffer size from dimensions and pixel format. The supported format cases vary by SoC:

| Format Case | bpp Ratio | T20 | T21 | T23 | T30 | T31 | T32 | T40 | T41 |
|-------------|-----------|-----|-----|-----|-----|-----|-----|-----|-----|
| `PIX_FMT_NV12` | 3/2 (12bpp) | Y | Y | Y | Y | Y | Y | Y | Y |
| `PIX_FMT_YUYV422` | 2/1 (16bpp) | Y | Y | Y | Y | Y | Y | Y | Y |
| `PIX_FMT_UYVY422` | 2/1 (16bpp) | Y | Y | Y | Y | Y | Y | Y | Y |
| `PIX_FMT_RGB565BE` | 2/1 (16bpp) | Y | Y | Y | Y | Y | Y | Y | Y |
| `PIX_FMT_BGR0` | 4/1 (32bpp) | Y | Y | Y | Y | Y | Y | Y | Y |
| `PIX_FMT_BGR24` | 3/1 (24bpp) | Y | Y | Y | - | - | - | - | - |
| `PIX_FMT_HSV` | 4/1 (32bpp) | Y | - | - | - | - | - | - | - |
| `PIX_FMT_BGRA` | 4/1 (32bpp) | Y | - | - | - | - | - | - | - |
| `PIX_FMT_YUV422` | 2/1 (16bpp) | - | - | - | - | - | Y | Y | Y |
| `PIX_FMT_YVU422` | 2/1 (16bpp) | - | - | - | - | - | Y | Y | Y |
| `PIX_FMT_UVY422` | 2/1 (16bpp) | - | - | - | - | - | Y | Y | Y |
| `PIX_FMT_VUY422` | 2/1 (16bpp) | - | - | - | - | - | Y | Y | Y |
| `PIX_FMT_RAW8` | 1/1 (8bpp) | - | - | - | - | - | Y | Y | Y |
| `PIX_FMT_RAW16` | 2/1 (16bpp) | - | - | - | - | - | Y | Y | Y |

### Generation Summary

- **T20**: Most complete legacy set (includes `BGR24`, `HSV`, `BGRA`)
- **T21**: Drops `HSV` and `BGRA` from calc, keeps `BGR24`
- **T23, T30**: Drops `BGR24` too (only 5 base cases)
- **T31**: Same as T30 (5 base cases)
- **T32, T40, T41**: Adds new semiplanar 422 variants and RAW formats (11 cases), but drops `BGR24`, `HSV`, `BGRA`

### HAL Implications

The HAL should implement its own `calc_pic_size()` that handles the full superset of formats regardless of SoC, since the calculation is purely mathematical (width * height * bpp_num / bpp_den). This avoids the inconsistency where the SDK returns 0 for a valid format on some SoCs.

---

## 11. fmt_to_string() Inline Function

Identical across all 8 SoCs. Only handles two cases:

```c
static inline const char *fmt_to_string(IMPPixelFormat imp_pixfmt) {
    switch (imp_pixfmt) {
    case PIX_FMT_NV12:    return "nv12";
    case PIX_FMT_YUYV422: return "yuyv422";
    default:              return NULL;
    }
}
```

### HAL Implications

The HAL should provide a more complete format-to-string function covering all supported formats.

---

## 12. Additional Declarations (T32-only)

T32's `imp_common.h` includes additional function declarations not present in any other SoC:

```c
void nv12dump_continues(IMPFrameInfo *frame, char *pst);
void NV12dump(IMPFrameInfo *frame, char *pst);
int64_t sample_gettimeus(void);
int get_bootup_timer(char *argv);
#define DEBUG_INFO(a) { ... get_bootup_timer(buf); }
```

These are debug/diagnostic utilities:
- `NV12dump` / `nv12dump_continues`: Dump NV12 frames to file
- `sample_gettimeus`: Get timestamp in microseconds
- `get_bootup_timer`: Boot timing measurement
- `DEBUG_INFO`: Debug logging macro with function/line info

### HAL Implications

These are debug-only functions leaked into the public header on T32. They should not be used in production HAL code. If frame dumping is needed, the HAL should implement its own.

---

## Summary: SoC Groupings

For HAL implementation purposes, the SoCs fall into these compatibility groups:

| Group | SoCs | Key Characteristics |
|-------|------|---------------------|
| **Gen1** | T20, T21, T30 | Basic IMPFrameInfo, no extended pixel formats, no IMPLine |
| **Gen1.5** | T23 | Adds `direct_phyAddr`, `timeStamp_ivdc`, IMPLine |
| **Gen2** | T31 | Adds `rotate_osdflag`, IMPLine, but no `direct_phyAddr`/`pool` |
| **Gen3** | T32, T40, T41 | Full `direct_phyAddr` + `pool`, extended pixel formats, IMPLine |

T20 is further unique in lacking `PT_H265`. T32 leaks debug functions. T40 uniquely has `PIX_FMT_2BIT`.

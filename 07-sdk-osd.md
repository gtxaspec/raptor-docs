# 07 — SDK OSD Module (imp_osd.h / isp_osd.h)

## Overview

The IMP OSD module overlays graphical elements (timestamps, logos, rectangles, privacy masks) onto the video pipeline. It operates on a group/region model: groups are bound into the data flow between FrameSource and Encoder, and regions are registered to groups.

The OSD subsystem has evolved significantly across SoC generations. The T20/T21/T30 "classic" API supports 5 region types. The T31 adds `OSD_REG_PIC_RMEM`. The T23/T32/T40/T41 "extended" API adds ISP-level drawing, mosaic, slash lines, corner rectangles, and font-inversion support.

---

## 1. Function Presence Matrix

| Function | T20 | T21 | T23 | T30 | T31 | T32 | T40 | T41 |
|---|---|---|---|---|---|---|---|---|
| `IMP_OSD_SetPoolSize` | Y | Y | Y | Y | Y | Y | Y | Y |
| `IMP_OSD_CreateGroup` | Y | Y | Y | Y | Y | Y | Y | Y |
| `IMP_OSD_DestroyGroup` | Y | Y | Y | Y | Y | Y | Y | Y |
| `IMP_OSD_AttachToGroup` | Y | Y | Y | Y | Y | Y | Y | Y |
| `IMP_OSD_CreateRgn` | Y | Y | Y | Y | Y | Y | Y | Y |
| `IMP_OSD_DestroyRgn` | Y | Y | Y | Y | Y | Y | Y | Y |
| `IMP_OSD_RegisterRgn` | Y | Y | Y | Y | Y | Y | Y | Y |
| `IMP_OSD_UnRegisterRgn` | Y | Y | Y | Y | Y | Y | Y | Y |
| `IMP_OSD_SetRgnAttr` | Y | Y | Y | Y | Y | Y | Y | Y |
| `IMP_OSD_SetRgnAttrWithTimestamp` | Y | Y | Y | Y | Y | Y | Y | Y |
| `IMP_OSD_GetRgnAttr` | Y | Y | Y | Y | Y | Y | Y | Y |
| `IMP_OSD_UpdateRgnAttrData` | Y | Y | Y | Y | Y | Y | Y | Y |
| `IMP_OSD_SetGrpRgnAttr` | Y | Y | Y | Y | Y | Y | Y | Y |
| `IMP_OSD_GetGrpRgnAttr` | Y | Y | Y | Y | Y | Y | Y | Y |
| `IMP_OSD_ShowRgn` | Y | Y | Y | Y | Y | Y | Y | Y |
| `IMP_OSD_Start` | Y | Y | Y | Y | Y | Y | Y | Y |
| `IMP_OSD_Stop` | Y | Y | Y | Y | Y | Y | Y | Y |
| `IMP_OSD_SetMosaic` | - | - | Y | - | - | Y | Y | Y |
| `IMP_OSD_GetRegionLuma` | - | - | Y | - | - | - | - | Y |
| `IMP_OSD_RgnCreate_Query` | - | - | Y | - | - | Y | Y | Y |
| `IMP_OSD_RgnRegister_Query` | - | - | Y | - | - | Y | Y | Y |
| `IMP_OSD_SetRgnAttr_ISP` | - | - | Y | - | - | Y | Y | Y |
| `IMP_OSD_GetRgnAttr_ISP` | - | - | Y | - | - | Y | Y | Y |
| `IMP_OSD_SetGroupCallback` | - | - | - | - | - | - | Y | Y |

**Notes:**
- `IMP_OSD_SetRgnAttr_ISP` signature differs: T23 has `(prAttr, bosdshow)`, T32/T40/T41 have `(sensornum, prAttr, bosdshow)`.
- `IMP_OSD_GetRegionLuma` is present in T23 and T41 for timestamp color-inversion support.
- `IMP_OSD_SetGroupCallback` is T40/T41 only, providing per-frame callback for custom OSD drawing.
- All 8 SoCs share the 17 core functions (SetPoolSize through Stop).

---

## 2. OSD Region Types (IMPOsdRgnType)

### Classic SDK (T20, T21, T30)

```c
typedef enum {
    OSD_REG_INV    = 0,   // undefined
    OSD_REG_LINE   = 1,   // line
    OSD_REG_RECT   = 2,   // rectangle
    OSD_REG_BITMAP = 3,   // monochrome bitmap
    OSD_REG_COVER  = 4,   // privacy mask (solid color rectangle)
    OSD_REG_PIC    = 5,   // RGBA picture (logo/timestamp)
} IMPOsdRgnType;
```

### T31

```c
typedef enum {
    OSD_REG_INV      = 0,
    OSD_REG_LINE     = 1,
    OSD_REG_RECT     = 2,
    OSD_REG_BITMAP   = 3,
    OSD_REG_COVER    = 4,
    OSD_REG_PIC      = 5,
    OSD_REG_PIC_RMEM = 6,   // picture using reserved memory
} IMPOsdRgnType;
```

### Extended SDK (T23, T32, T40, T41)

```c
typedef enum {
    OSD_REG_INV             = 0,
    OSD_REG_HORIZONTAL_LINE = 1,   // horizontal line only
    OSD_REG_VERTICAL_LINE   = 2,   // vertical line only
    OSD_REG_RECT            = 3,   // rectangle
    OSD_REG_FOUR_CORNER_RECT = 4,  // corner brackets only
    OSD_REG_BITMAP          = 5,   // monochrome bitmap
    OSD_REG_COVER           = 6,   // privacy mask
    OSD_REG_PIC             = 7,   // RGBA picture
    OSD_REG_PIC_RMEM        = 8,   // picture from reserved memory
    OSD_REG_SLASH           = 9,   // diagonal line
    OSD_REG_ISP_PIC         = 10,  // ISP-level picture overlay
    OSD_REG_ISP_LINE_RECT   = 11,  // ISP-level line/rect drawing
    OSD_REG_ISP_COVER       = 12,  // ISP-level privacy mask
    OSD_REG_MOSAIC          = 13,  // mosaic blur region
} IMPOsdRgnType;
```

**Critical differences for the HAL:**

| Feature | T20/T21/T30 | T31 | T23/T32/T40/T41 |
|---|---|---|---|
| LINE type | `OSD_REG_LINE = 1` (any angle) | Same as left | Split: HORIZONTAL=1, VERTICAL=2, SLASH=9 |
| BITMAP value | 3 | 3 | **5** |
| COVER value | 4 | 4 | **6** |
| PIC value | 5 | 5 | **7** |
| PIC_RMEM | absent | 6 | **8** |
| ISP types | absent | absent | 10, 11, 12 |
| MOSAIC | absent | absent | 13 |
| FOUR_CORNER_RECT | absent | absent | 4 |

The enum value shift from T20-era to T23-era is extremely important. `OSD_REG_BITMAP` changes from 3 to 5, `OSD_REG_COVER` from 4 to 6, `OSD_REG_PIC` from 5 to 7. The HAL must use the correct enum values per SoC.

---

## 3. IMPOSDRgnAttr Struct

### Classic (T20, T21, T30)

```c
typedef struct {
    IMPOsdRgnType       type;   // region type
    IMPRect             rect;   // position and size
    IMPPixelFormat      fmt;    // pixel format for bitmap/pic data
    IMPOSDRgnAttrData   data;   // union: bitmapData, lineRectData, coverData, picData
} IMPOSDRgnAttr;
```

### T31

Same structure as classic:

```c
typedef struct {
    IMPOsdRgnType       type;
    IMPRect             rect;
    IMPPixelFormat      fmt;
    IMPOSDRgnAttrData   data;
} IMPOSDRgnAttr;
```

### Extended (T23, T32, T40, T41)

```c
typedef struct {
    IMPOsdRgnType       type;
    IMPRect             rect;
    IMPLine             line;           // NEW: explicit line start/end coordinates
    IMPPixelFormat      fmt;
    IMPOSDRgnAttrData   data;
    IMPOSDIspDraw       osdispdraw;     // NEW: ISP drawing attributes
    IMPOSDFontAttrData  fontData;       // NEW: font/timestamp inversion data
    IMPOSDMosaicAttr    mosaicAttr;     // NEW: mosaic parameters
} IMPOSDRgnAttr;
```

**T40 only** adds one additional field:

```c
    IMPOSD2BitAttr      bitAttr;        // 2-bit pixel format support
```

The `IMPOSD2BitAttr` struct provides a 2-bit mask mode with two configurable colors, useful for ultra-low-memory OSD text rendering.

### lineRectData differences

Classic (T20/T21/T30/T31): 2 fields
```c
typedef struct {
    uint32_t color;
    uint32_t linewidth;
} lineRectData;
```

Extended (T23/T32/T40/T41): 4 fields
```c
typedef struct {
    uint32_t color;
    uint32_t linewidth;
    uint32_t linelength;        // length of line segment
    uint32_t rectlinelength;    // corner bracket length (for FOUR_CORNER_RECT)
} lineRectData;
```

### IMPOSDIspDraw

Present only in T23/T32/T40/T41. References ISP tuning structs:

```c
typedef struct {
    IMPISPDrawBlockAttr stDrawAttr;    // line/rect via ISP
    IMPISPOSDBlockAttr  stpicAttr;     // picture via ISP
    IMPISPMaskBlockAttr stCoverAttr;   // privacy mask via ISP
} IMPOSDIspDraw;
```

T41 uses slightly different ISP struct names: `IMPISPDrawAttr`, `IMPISPOSDAttr`, `IMPISPMASKAttr`.

### IMPOSDFontAttrData

Present in T23/T32/T40/T41 for timestamp color-inversion:

```c
typedef struct {
    unsigned int invertColorSwitch;   // enable color inversion
    unsigned int luminance;           // brightness threshold (default 190)
    unsigned int length;              // number of font characters
    IMPOSDFontSizeAttrData data;      // {fontWidth, fontHeight}
    unsigned int istimestamp;          // whether data is a timestamp
    unsigned int colType[N];          // per-character inversion state
} IMPOSDFontAttrData;
```

The `colType` array size differs: T23 has `colType[64]`, while T32/T40/T41 have `colType[20]`.

### IMPOSDMosaicAttr

Present in T23/T32/T40/T41:

```c
typedef struct mosaicPointAttr {
    int x;                // start X (2-aligned)
    int y;                // start Y (2-aligned)
    int mosaic_width;     // width (2-aligned)
    int mosaic_height;    // height (2-aligned)
    int frame_width;      // channel resolution width
    int frame_height;     // channel resolution height
    int mosaic_min_size;  // minimum mosaic block size (2-aligned)
} IMPOSDMosaicAttr;
```

---

## 4. IMPOSDGrpRgnAttr Struct

Identical across all 8 SoCs:

```c
typedef struct {
    int      show;       // display flag
    IMPPoint offPos;     // display offset position
    float    scalex;     // X scale factor
    float    scaley;     // Y scale factor
    int      gAlphaEn;   // global alpha enable
    int      fgAlhpa;    // foreground alpha (note: typo 'Alhpa' in SDK)
    int      bgAlhpa;    // background alpha (note: typo 'Alhpa' in SDK)
    int      layer;      // Z-order layer
} IMPOSDGrpRgnAttr;
```

The typo `fgAlhpa`/`bgAlhpa` (instead of `fgAlpha`/`bgAlpha`) is consistent across all SDKs and must be used as-is.

---

## 5. Color Formats for OSD

### IMPOsdColour enum

**Classic (T20/T21/T30/T31):** BGRA format with alpha in high byte:
```c
typedef enum {
    OSD_BLACK = 0xff000000,
    OSD_WHITE = 0xffffffff,
    OSD_RED   = 0xffff0000,
    OSD_GREEN = 0xff00ff00,
    OSD_BLUE  = 0xff0000ff,
} IMPOsdColour;
```

**T41 (and T40):** Indexed enum (NOT BGRA values):
```c
typedef enum {
    OSD_RED,      // = 0
    OSD_BLACK,    // = 1
    OSD_GREEN,    // = 2
    OSD_YELLOW,   // = 3
} IMPOsdColour;
```

The T40/T41 also add `IMPIpuColour` with the traditional BGRA values:
```c
typedef enum {
    OSD_IPU_BLACK = 0xff000000,
    OSD_IPU_WHITE = 0xffffffff,
    OSD_IPU_RED   = 0xffff0000,
    OSD_IPU_GREEN = 0xff00ff00,
    OSD_IPU_BLUE  = 0xff0000ff,
} IMPIpuColour;
```

For T23/T32, the `IMPOsdColour` enum is not present in the header; only `IMPIpuColour` is defined (same BGRA values as `OSD_IPU_*`).

### Pixel format for bitmap/picture data

All SoCs use `IMPPixelFormat` (from `imp_common.h`) for the `fmt` field. OSD pictures are typically `PIX_FMT_BGRA` (32-bit RGBA with BGRA byte order). Bitmap overlays use a monochrome format where each pixel is a single foreground/background flag.

---

## 6. ISP OSD Functions (isp_osd.h)

Present for: **T23, T32, T40, T41** (identical API across all four).

The `isp_osd.h` header provides a separate ISP-level OSD subsystem that draws directly in the ISP pipeline (before the encoder), independent of the IPU-based OSD groups. This is useful for overlays that must appear in all output channels without per-channel configuration.

### Functions

| Function | Signature | Description |
|---|---|---|
| `IMP_OSD_Init_ISP` | `(void) -> int` | Initialize ISP OSD resources (called internally by `IMP_System_Init`) |
| `IMP_OSD_SetPoolSize_ISP` | `(int size) -> int` | Set ISP OSD reserved memory pool size |
| `IMP_OSD_CreateRgn_ISP` | `(int chn, IMPIspOsdAttrAsm *pIspOsdAttr) -> int` | Create ISP OSD region on sensor channel |
| `IMP_OSD_SetRgnAttr_PicISP` | `(int chn, int handle, IMPIspOsdAttrAsm *pIspOsdAttr) -> int` | Set/update ISP OSD region attributes |
| `IMP_OSD_GetRgnAttr_ISPPic` | `(int chn, int handle, IMPIspOsdAttrAsm *pIspOsdAttr) -> int` | Get ISP OSD region attributes |
| `IMP_OSD_ShowRgn_ISP` | `(int chn, int handle, int showFlag) -> int` | Show/hide ISP OSD region |
| `IMP_OSD_DestroyRgn_ISP` | `(int chn, int handle) -> int` | Destroy ISP OSD region |
| `IMP_OSD_Exit_ISP` | `(void) -> void` | Cleanup ISP OSD resources (called internally by `IMP_System_Exit`) |

### Key struct: IMPIspOsdAttrAsm

Defined in `imp_isp.h`, not in `isp_osd.h`. This struct wraps ISP-level OSD configuration including position, image data pointer, and display flags. The `chn` parameter selects which sensor (for dual-sensor SoCs like T40).

### ISP OSD vs IPU OSD

| Aspect | IPU OSD (imp_osd.h) | ISP OSD (isp_osd.h) |
|---|---|---|
| Pipeline position | After ISP, in IPU processing | Inside ISP, before frame output |
| Per-channel | Yes (groups per encoder channel) | No (applies to sensor output) |
| Region types | Full set (line/rect/bitmap/cover/pic/mosaic) | Picture + draw blocks only |
| Performance | Software + hardware hybrid | Hardware only (ISP block) |
| Availability | All 8 SoCs | T23/T32/T40/T41 only |

---

## 7. Group and Region Limits

The `NR_MAX_OSD_GROUPS` and max region count are defined internally in `libimp`, not in public headers. From SDK documentation and sample code:

| Parameter | T20 | T21 | T23 | T30 | T31 | T32 | T40 | T41 |
|---|---|---|---|---|---|---|---|---|
| Max OSD groups | 2 | 2 | 2 | 2 | 2 | 2 | 2-4 | 2-4 |
| Max regions per group | 8 | 8 | 16 | 8 | 8 | 16 | 16 | 16 |
| ISP OSD regions | - | - | 8 | - | - | 8 | 8 | 8 |

Group count typically matches the number of encoder channels. T20/T21/T30/T31 typically support 2 groups (main + sub stream). T40/T41 may support up to 4 groups when using multiple sensor inputs.

---

## 8. Additional Types and Features

### Error codes (T23, T32, T40, T41 only)

The extended SDK defines OSD-specific error codes:

```c
IMP_ERR_OSD_CHNID           = 0x80020001  // channel ID out of range
IMP_ERR_OSD_PARAM            = 0x80020002  // parameter out of range
IMP_ERR_OSD_EXIST            = 0x80020004  // resource already exists
IMP_ERR_OSD_UNEXIST          = 0x80020008  // resource does not exist
IMP_ERR_OSD_NULL_PTR         = 0x80020010  // null pointer
IMP_ERR_OSD_NOT_CONFIG       = 0x80020020  // not configured
IMP_ERR_OSD_NOT_SUPPORT      = 0x80020040  // unsupported
IMP_ERR_OSD_PERM             = 0x80020080  // operation not permitted
IMP_ERR_OSD_NOMEM            = 0x80020100  // memory allocation failed
IMP_ERR_OSD_SYS_NOTREADY     = 0x80022000  // system not initialized
```

Classic SDK (T20/T21/T30/T31) does not define these; functions return 0 or -1.

### IMPOSDRgnCreateStat / IMPOSDRgnRegisterStat (T23, T32, T40, T41 only)

Query structures for region lifecycle:
```c
typedef struct stRgnCreateStat {
    int status;    // 0 = not created, 1 = created
} IMPOSDRgnCreateStat;

typedef struct stRgnRigsterStat {
    int status;    // 0 = not registered, 1 = registered
} IMPOSDRgnRegisterStat;
```

### OSD_Frame_CALLBACK (T40, T41 only)

```c
typedef int (*OSD_Frame_CALLBACK)(int group_index, IMPFrameInfo *frame);
```

Registered via `IMP_OSD_SetGroupCallback()`, called per-frame with the NV12 frame data pointer, allowing custom software OSD drawing directly on frame buffers.

---

## 9. HAL Implications — OSD Bitmap Upload (ROD to RVD)

### Workflow

1. **ROD** (OSD daemon) renders timestamp/logo into an RGBA bitmap buffer
2. ROD writes the bitmap + metadata to a shared memory ring buffer
3. **RVD** (video daemon) reads from the ring, and calls into the HAL OSD layer
4. HAL calls `IMP_OSD_SetRgnAttr()` or `IMP_OSD_UpdateRgnAttrData()` to update the region

### Per-SoC considerations

**Region type selection:**
- For RGBA overlays (timestamps, logos): use `OSD_REG_PIC` (value 5 on classic, 7 on extended)
- For privacy masks: use `OSD_REG_COVER` (value 4 on classic, 6 on extended)
- On T23/T32/T40/T41, `OSD_REG_PIC_RMEM` (value 8) uses reserved memory and avoids per-frame copies; preferred for static logos

**Data pointer management:**
- `picData.pData` must point to a contiguous BGRA buffer, width*height*4 bytes
- The pointer must remain valid until the next `UpdateRgnAttrData` call
- On T31+, `OSD_REG_PIC_RMEM` allocates from reserved memory (rmem) set via `IMP_OSD_SetPoolSize()`

**Alpha blending:**
- `IMPOSDGrpRgnAttr.gAlphaEn` enables per-region alpha
- `fgAlhpa` (0-255) controls foreground opacity, `bgAlhpa` controls background
- Per-pixel alpha from RGBA data is also applied when `gAlphaEn` is set

**Scaling:**
- `scalex`/`scaley` in `IMPOSDGrpRgnAttr` allow scaling the OSD region
- Set to 1.0 for no scaling (most common)

**ISP OSD path (T23/T32/T40/T41):**
- Alternative to IPU OSD, bypasses group/region model
- Use `IMP_OSD_SetRgnAttr_ISP()` or `isp_osd.h` functions
- Appears on all encoder outputs without per-channel setup
- Limited to ISP block capabilities (fewer overlay types)

### Initialization sequence

```
1. IMP_OSD_SetPoolSize(pool_size)          // optional, for RMEM regions
2. IMP_OSD_CreateGroup(grpNum)             // per encoder channel
3. IMP_System_Bind(fs_cell, osd_cell)      // bind FrameSource -> OSD
4. IMP_System_Bind(osd_cell, enc_cell)     // bind OSD -> Encoder
5. IMP_OSD_CreateRgn(&rgnAttr)             // returns handle
6. IMP_OSD_RegisterRgn(handle, grpNum, &grpRgnAttr)
7. IMP_OSD_Start(grpNum)
8. // runtime: IMP_OSD_UpdateRgnAttrData(handle, &data)  or
//            IMP_OSD_SetRgnAttr(handle, &rgnAttr)
```

### Teardown sequence

```
1. IMP_OSD_Stop(grpNum)
2. IMP_OSD_UnRegisterRgn(handle, grpNum)
3. IMP_OSD_DestroyRgn(handle)
4. IMP_System_UnBind(osd_cell, enc_cell)
5. IMP_System_UnBind(fs_cell, osd_cell)
6. IMP_OSD_DestroyGroup(grpNum)
```

# SDK FrameSource Module (`imp_framesource.h`)

Cross-SoC API reference for the IMP FrameSource module across all 8 supported Ingenic SoCs.

---

## 1. Function Presence Matrix

| Function | T20 | T21 | T23 | T30 | T31 | T32 | T40 | T41 |
|---|---|---|---|---|---|---|---|---|
| CreateChn | Y | Y | Y | Y | Y | Y | Y | Y |
| DestroyChn | Y | Y | Y | Y | Y | Y | Y | Y |
| EnableChn | Y | Y | Y | Y | Y | Y | Y | Y |
| DisableChn | Y | Y | Y | Y | Y | Y | Y | Y |
| GetChnAttr | Y | Y | Y | Y | Y | Y | Y | Y |
| SetChnAttr | Y | Y | Y | Y | Y | Y | Y | Y |
| SetFrameDepth | Y | Y | Y | Y | Y | Y | Y | Y |
| GetFrameDepth | Y | Y | Y | Y | Y | Y | Y | Y |
| GetFrame | Y | Y | Y | Y | Y | Y | Y | Y |
| GetTimedFrame | Y | Y | Y | Y | Y | Y | Y | Y |
| ReleaseFrame | Y | Y | Y | Y | Y | Y | Y | Y |
| SnapFrame | Y | Y | Y | Y | Y | Y | Y | Y |
| SetMaxDelay | Y | Y | Y | Y | Y | Y | Y | Y |
| GetMaxDelay | Y | Y | Y | Y | Y | Y | Y | Y |
| SetDelay | Y | Y | Y | Y | Y | Y | Y | Y |
| GetDelay | Y | Y | Y | Y | Y | Y | Y | Y |
| SetChnFifoAttr | Y | Y | Y | Y | Y | Y | Y | Y |
| GetChnFifoAttr | Y | Y | Y | Y | Y | Y | Y | Y |
| SetPool | - | - | Y | - | Y | Y | Y | Y |
| GetPool | - | - | Y | - | Y | Y | Y | Y |
| SetSource | - | - | - | - | Y | - | - | - |
| ChnStatQuery | - | - | - | - | Y | - | - | - |
| SetChnRotate | - | - | - | - | Y | - | - | - |
| SetDirectModeAttr | - | - | Y | - | - | - | - | - |
| GetDirectModeAttr | - | - | Y | - | - | - | - | - |
| GetFrameEx | - | - | - | - | - | - | Y | Y |
| ReleaseFrameEx | - | - | - | - | - | - | Y | Y |
| DequeueBuffer | - | - | - | - | - | - | Y | Y |
| QueueBuffer | - | - | - | - | - | - | Y | Y |
| ExternInject_CreateChn | - | - | - | - | - | - | Y | Y |
| ExternInject_DestroyChn | - | - | - | - | - | - | Y | Y |
| ExternInject_EnableChn | - | - | - | - | - | - | Y | Y |
| ExternInject_DisableChn | - | - | - | - | - | - | Y | Y |
| GetI2dAttr | - | - | - | - | - | - | Y | Y |
| SetI2dAttr | - | - | - | - | - | - | Y | Y |
| SetYuvAlign | - | - | - | - | - | Y | - | Y |
| SetIvdcMemLine | - | - | - | - | - | Y | - | - |
| SetFrameDepthCopyType | - | - | - | - | - | Y | - | - |
| PM_Suspend | - | - | - | - | - | Y | - | - |
| PM_Resume | - | - | - | - | - | Y | - | - |
| GetDirectMode | - | - | - | - | - | Y | - | - |
| GetYuv | - | - | - | - | - | Y | - | - |

Notes:
- T20/T21/T30 have an identical 18-function core API.
- T23 adds SetPool/GetPool and DirectMode.
- T31 adds SetSource, ChnStatQuery, SetChnRotate, SetPool/GetPool.
- T32 adds PM_Suspend/Resume, SetYuvAlign, SetIvdcMemLine, SetFrameDepthCopyType, GetDirectMode, GetYuv.
- T40/T41 add I2D transform, ExternInject, DequeueBuffer/QueueBuffer, GetFrameEx/ReleaseFrameEx.

---

## 2. IMPFSChnAttr Struct -- Full Definition Per SoC

### T20 / T21 / T30 (Identical)

```c
typedef struct {
    int picWidth;
    int picHeight;
    IMPPixelFormat pixFmt;
    IMPFSChnCrop crop;          /* ISP crop */
    IMPFSChnScaler scaler;
    int outFrmRateNum;
    int outFrmRateDen;
    int nrVBs;
    IMPFSChnType type;
} IMPFSChnAttr;
```

### T23

```c
typedef struct {
    int picWidth;
    int picHeight;
    IMPPixelFormat pixFmt;
    IMPFSChnCrop crop;
    IMPFSChnScaler scaler;
    int outFrmRateNum;
    int outFrmRateDen;
    int nrVBs;
    IMPFSChnType type;
    int mirr_enable;            /* NEW: picture mirror enable */
    IMPFSChnCrop fcrop;         /* NEW: frame crop (post-scaler) */
} IMPFSChnAttr;
```

### T31

```c
typedef struct {
    int picWidth;
    int picHeight;
    IMPPixelFormat pixFmt;
    IMPFSChnCrop crop;
    IMPFSChnScaler scaler;
    int outFrmRateNum;
    int outFrmRateDen;
    int nrVBs;
    IMPFSChnType type;
    IMPFSChnCrop fcrop;         /* frame crop, no mirr_enable */
} IMPFSChnAttr;
```

### T32

```c
typedef struct {
    IMPFSI2DAttr i2dattr;       /* NEW: I2D transform block */
    int picWidth;
    int picHeight;
    IMPPixelFormat pixFmt;
    IMPFSChnCrop crop;
    IMPFSChnScaler scaler;
    int outFrmRateNum;
    int outFrmRateDen;
    int nrVBs;
    IMPFSChnType type;
    IMPFSChnCrop fcrop;
    int mirr_enable;
} IMPFSChnAttr;
```

### T40

```c
typedef struct {
    IMPFSI2DAttr i2dattr;       /* I2D transform block */
    int picWidth;
    int picHeight;
    IMPPixelFormat pixFmt;
    IMPFSChnCrop crop;
    IMPFSChnScaler scaler;
    int outFrmRateNum;
    int outFrmRateDen;
    int nrVBs;
    IMPFSChnType type;
    IMPFSChnCrop fcrop;
    int mirr_enable;
} IMPFSChnAttr;
```

### T41

```c
typedef struct {
    IMPFSI2DAttr i2dattr;       /* I2D transform block */
    int picWidth;
    int picHeight;
    IMPPixelFormat pixFmt;
    IMPFSChnCrop crop;
    IMPFSChnScaler scaler;
    int outFrmRateNum;
    int outFrmRateDen;
    int nrVBs;
    IMPFSChnType type;
    IMPFSChnCrop fcrop;
    /* no mirr_enable -- mirror is in i2dattr */
} IMPFSChnAttr;
```

### Key Differences Summary

| Field | T20/T21/T30 | T23 | T31 | T32 | T40 | T41 |
|---|---|---|---|---|---|---|
| `crop` (ISP crop) | Y | Y | Y | Y | Y | Y |
| `fcrop` (frame crop) | - | Y | Y | Y | Y | Y |
| `mirr_enable` | - | Y | - | Y | Y | - |
| `i2dattr` (IMPFSI2DAttr) | - | - | - | Y | Y | Y |

The `crop` field applies cropping at the ISP level (before scaling). The `fcrop` field was introduced starting with T23/T31 and applies frame-level cropping after the ISP pipeline. On T32/T40/T41, `i2dattr` is placed at the very beginning of the struct, changing the struct layout.

---

## 3. IMPFSI2DAttr Struct (T32/T40/T41 Only)

```c
typedef struct i2dattr {
    int i2d_enable;         /* master enable for I2D module */
    int flip_enable;        /* vertical flip */
    int mirr_enable;        /* horizontal mirror */
    int rotate_enable;      /* rotation enable */
    int rotate_angle;       /* rotation angle (90/270) */
} IMPFSI2DAttr;
```

This hardware I2D (Image-to-DRAM) module provides flip, mirror, and 90-degree rotation without CPU overhead. It replaces the software-based `SetChnRotate` on T31.

---

## 4. IMPFSChnFifoAttr -- Definition Per SoC

Identical across all 8 SoCs:

```c
typedef enum {
    FIFO_CACHE_PRIORITY = 0,    /* cache first, then output */
    FIFO_DATA_PRIORITY,         /* output first, then cache */
} IMPFSChnFifoType;

typedef struct {
    int maxdepth;
    IMPFSChnFifoType type;
} IMPFSChnFifoAttr;
```

---

## 5. IMPFSChnType Enum

| Value | T20/T21/T23/T30/T31 | T32/T40/T41 |
|---|---|---|
| `FS_PHY_CHANNEL` (0) | Y | Y |
| `FS_EXT_CHANNEL` (1) | Y | Y |
| `FS_INJ_CHANNEL` (2) | - | Y |

`FS_INJ_CHANNEL` is the external injection channel type used with the `ExternInject_*` functions on T32/T40/T41.

---

## 6. Channel Number Limits

| SoC | Physical Channels | Extension Channels | Max Channels | Notes |
|---|---|---|---|---|
| T20 | 0, 1 | 2, 3 | 4 | Ch 0-1 physical, Ch 2-3 extension |
| T21 | 0, 1 | 2, 3 | 4 | Same as T20 |
| T23 | 0, 1 | 2+ | 3+ | Ch 0 high-def, Ch 1 standard, Ch 2 extension |
| T30 | 0, 1 | 2, 3 | 4 | Same as T20 |
| T31 | 0, 1 | 2+ | 3+ | SetSource allows routing ext channels to any physical source |
| T32 | 0, 1, 2 | - | 3+ | Ch 0 ultra-clear, Ch 1 high-clear, Ch 2 standard/JPEG |
| T40 | 0, 1, 2 | - | 3+ | Same as T32 |
| T41 | 0, 1, 2 | - | 3+ | Same as T32 |

T32/T40/T41 document "3 outputs" with Ch 0 = ultra-clear, Ch 1 = high-clear, Ch 2 = standard-clear or JPEG snap. The SetPool interface suggests channel IDs from 0 to 31 are theoretically valid (chnNum >= 0 and < 32).

---

## 7. Additional Channel State Enum (T31 Only)

```c
typedef enum {
    IMP_FSCHANNEL_STATE_CLOSE,  /* not created or destroyed */
    IMP_FSCHANNEL_STATE_OPEN,   /* created but not enabled */
    IMP_FSCHANNEL_STATE_RUN,    /* created and enabled */
} IMPFSChannelState;
```

Used with `IMP_FrameSource_ChnStatQuery()`.

---

## 8. Error Codes (T32/T40 Only)

T32 and T40 define explicit FrameSource error codes:

```c
enum {
    IMP_OK_FS_ALL               = 0x0,
    IMP_ERR_FS_CHNID            = 0x80010001,  /* channel ID out of range */
    IMP_ERR_FS_PARAM            = 0x80010002,  /* parameter out of range */
    IMP_ERR_FS_EXIST            = 0x80010004,  /* resource already exists */
    IMP_ERR_FS_UNEXIST          = 0x80010008,  /* resource does not exist */
    IMP_ERR_FS_NULL_PTR         = 0x80010010,  /* null pointer */
    IMP_ERR_FS_NOT_CONFIG       = 0x80010020,  /* not configured */
    IMP_ERR_FS_NOT_SUPPORT      = 0x80010040,  /* unsupported */
    IMP_ERR_FS_PERM             = 0x80010080,  /* operation not permitted */
    IMP_ERR_FS_NOMEM            = 0x80010100,  /* memory allocation failed */
    IMP_ERR_FS_NOBUF            = 0x80010200,  /* buffer allocation failed */
    IMP_ERR_FS_BUF_EMPTY        = 0x80010400,  /* buffer empty */
    IMP_ERR_FS_BUF_FULL         = 0x80010800,  /* buffer full */
    IMP_ERR_FS_SYS_NOTREADY     = 0x80011000,  /* system not initialized */
    IMP_ERR_FS_OVERTIME         = 0x80012000,  /* wait timeout */
    IMP_ERR_FS_RESOURCE_REQUEST = 0x80014000,  /* resource request failed */
};
```

Earlier SoCs return generic non-zero error codes without these named constants.

---

## 9. YUV Alignment (T32/T40 Only)

```c
typedef enum {
    FRAME_ALIGN_8BYTE,
    FRAME_ALIGN_16BYTE,
    FRAME_ALIGN_32BYTE,
} FSChannelYuvAlign;

struct yuvaliparm {
    FSChannelYuvAlign w;
    FSChannelYuvAlign h;
};

typedef struct {
    int32_t enable;
    struct yuvaliparm param;
} IMPFrameAlign;
```

Used with `IMP_FrameSource_SetYuvAlign()` to control width/height alignment of output YUV frames. Default alignment varies by SoC.

---

## 10. New SDK Additions by Generation

### GetFrameEx / ReleaseFrameEx (T40/T41)

```c
int IMP_FrameSource_GetFrameEx(int chnNum, IMPFrameInfo **frame);
int IMP_FrameSource_ReleaseFrameEx(int chnNum, IMPFrameInfo *pframe);
```

Extended frame acquisition pair. Must be called after channel creation. Appears alongside but separate from the standard GetFrame/ReleaseFrame pair.

### DequeueBuffer / QueueBuffer (T40/T41)

```c
int IMP_FrameSource_DequeueBuffer(int chnNum, IMPFrameInfo **frame);
int IMP_FrameSource_QueueBuffer(int chnNum, const IMPFrameInfo *frame);
```

Used with ExternInject channels. DequeueBuffer obtains an idle buffer from FrameSource, the caller fills it with external NV12 data, then QueueBuffer submits it back into the pipeline for IVS/OSD/Encoder processing. These must be used in pairs.

### ExternInject_* (T40/T41)

```c
int IMP_FrameSource_ExternInject_CreateChn(int chnNum, IMPFSChnAttr *chnAttr);
int IMP_FrameSource_ExternInject_DestroyChn(int chnNum);
int IMP_FrameSource_ExternInject_EnableChn(int chnNum);
int IMP_FrameSource_ExternInject_DisableChn(int chnNum);
```

Create simulation channels that import external NV12 data into the IMP pipeline. These channels do not support cropping or scaling. The channel type should be set to `FS_INJ_CHANNEL`.

---

## 11. T31: SetChnRotate

```c
int IMP_FrameSource_SetChnRotate(int chnNum, uint8_t rotTo90, int width, int height);
```

- `rotTo90 = 0`: disable rotation
- `rotTo90 = 1`: rotate 90 degrees counterclockwise
- `rotTo90 = 2`: rotate 90 degrees clockwise

Constraints:
- Must be called **before** `CreateChn`
- Resolution must be 64-byte aligned
- Encoder channel width/height must be set to the post-rotation dimensions
- Cannot be used with encoder soft zoom
- Allocates rmem memory for rotated frame data
- **Software implementation** -- uses CPU. Recommended max resolution 1280x704 at 15fps

This is T31-only. On T32/T40/T41, use the hardware I2D module via `IMPFSI2DAttr` in `IMPFSChnAttr` instead.

---

## 12. T32/T40/T41: I2D Transform

```c
int IMP_FrameSource_GetI2dAttr(int chnNum, IMPFSI2DAttr *pI2dAttr);
int IMP_FrameSource_SetI2dAttr(int chnNum, IMPFSI2DAttr *pI2dAttr);
```

Hardware image transform supporting flip, mirror, and rotation. Set via the `IMPFSI2DAttr` struct embedded in `IMPFSChnAttr`, or dynamically via these Get/Set functions.

---

## 13. T32: PM_Suspend / PM_Resume

```c
int IMP_FrameSource_PM_Suspend(int chnNum);
int IMP_FrameSource_PM_Resume(int chnNum);
```

Power management for FrameSource channels. `PM_Suspend` suspends a created channel and waits for completion (call before system suspend). `PM_Resume` resumes the channel (call after system wake). T32-only.

---

## 14. HAL Implications for `hal_fs_create_channel()`

### Struct Layout Divergence

The `IMPFSChnAttr` struct has four distinct layouts:

1. **Layout A** (T20/T21/T30): 9 fields, `crop` only, no `fcrop`, no `i2dattr`
2. **Layout B** (T23): 11 fields, adds `mirr_enable` + `fcrop` at end
3. **Layout C** (T31): 10 fields, adds `fcrop` at end (no `mirr_enable`)
4. **Layout D** (T32/T40/T41): 11-12 fields, `i2dattr` prepended at beginning, `fcrop` present, `mirr_enable` on T32/T40 but not T41

The HAL must:
1. Define a unified channel config struct that covers all fields
2. At channel creation time, populate only the fields relevant to the target SoC
3. Handle `i2dattr` being at offset 0 on Layout D (struct binary layout differs)

### Rotation Dispatch

```
if soc == T31:
    call IMP_FrameSource_SetChnRotate() before CreateChn
elif soc in (T32, T40, T41):
    populate i2dattr in IMPFSChnAttr before CreateChn
    or call IMP_FrameSource_SetI2dAttr() after CreateChn
else:
    rotation not supported
```

### Mirror Dispatch

```
if soc in (T23, T32, T40):
    set IMPFSChnAttr.mirr_enable = 1
elif soc in (T32, T40, T41):
    set IMPFSChnAttr.i2dattr.mirr_enable = 1
else:
    use ISP-level mirror (IMP_ISP_Tuning_SetISPHflip)
```

### Pool Binding

For SoCs with `SetPool` (T23/T31/T32/T40/T41), the HAL should optionally bind channels to memory pools to reduce rmem fragmentation. Call `SetPool` between `CreateChn` and `EnableChn`.

### ExternInject

Only T40/T41 support external frame injection. The HAL should provide a separate path for injecting pre-processed frames (e.g., from file or network) that uses `ExternInject_CreateChn` + `DequeueBuffer`/`QueueBuffer`.

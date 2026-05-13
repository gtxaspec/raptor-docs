# IMP Encoder Module -- Cross-SoC Reference

This document covers the `imp_encoder.h` header across all 8 Ingenic SoCs used by
Thingino.  The encoder module is the most complex in the SDK, with a **complete API
redesign** between the old-style SoCs (T20/T21/T23/T30) and the new-style SoCs
(T31/T40/T41), plus T32 which is a hybrid with its own unique additions.

Source headers:

| SoC | Version | Lang | SDK style |
|-----|---------|------|-----------|
| T20 | 3.12.0  | zh   | Old       |
| T21 | 1.0.33  | zh   | Old       |
| T23 | 1.3.0   | en   | Old       |
| T30 | 1.0.5   | zh   | Old       |
| T31 | 1.1.6   | en   | New       |
| T32 | 1.0.6   | en   | Hybrid (Old structs + New features) |
| T33 | 2.0.2.1 | en   | New       |
| T40 | 1.3.1   | en   | New       |
| T41 | 1.2.5   | en   | New       |
| A1  | 1.7.0   | en   | New       |

---

## 1. Function Presence Matrix

Legend: Y = present, `-` = absent.

### 1.1 Channel Lifecycle

| Function | T20 | T21 | T23 | T30 | T31 | T32 | T40 | T41 |
|----------|-----|-----|-----|-----|-----|-----|-----|-----|
| CreateGroup | Y | Y | Y | Y | Y | Y | Y | Y |
| DestroyGroup | Y | Y | Y | Y | Y | Y | Y | Y |
| CreateChn | Y | Y | Y | Y | Y | Y | Y | Y |
| DestroyChn | Y | Y | Y | Y | Y | Y | Y | Y |
| GetChnAttr | Y | Y | Y | Y | Y | Y | Y | Y |
| RegisterChn | Y | Y | Y | Y | Y | Y | Y | Y |
| UnRegisterChn | Y | Y | Y | Y | Y | Y | Y | Y |

Notes:
- T20/T21/T23/T30/T32 use `IMPEncoderCHNAttr` (capital CHN) for CreateChn/GetChnAttr.
- T31/T40/T41 use `IMPEncoderChnAttr` (mixed case) for CreateChn/GetChnAttr.

### 1.2 Stream Control

| Function | T20 | T21 | T23 | T30 | T31 | T32 | T40 | T41 |
|----------|-----|-----|-----|-----|-----|-----|-----|-----|
| StartRecvPic | Y | Y | Y | Y | Y | Y | Y | Y |
| StopRecvPic | Y | Y | Y | Y | Y | Y | Y | Y |
| RequestIDR | Y | Y | Y | Y | Y | Y | Y | Y |
| FlushStream | Y | Y | Y | Y | Y | Y | Y | Y |

### 1.3 Stream Access

| Function | T20 | T21 | T23 | T30 | T31 | T32 | T40 | T41 |
|----------|-----|-----|-----|-----|-----|-----|-----|-----|
| PollingStream | Y | Y | Y | Y | Y | Y | Y | Y |
| PollingModuleStream | - | Y | - | Y | Y | - | Y | Y |
| GetStream | Y | Y | Y | Y | Y | Y | Y | Y |
| ReleaseStream | Y | Y | Y | Y | Y | Y | Y | Y |
| Query | Y | Y | Y | Y | Y | Y | Y | Y |
| GetFd | - | Y | Y | Y | Y | Y | Y | Y |

Notes:
- `PollingModuleStream` allows polling all channels at once via a bitmap. Not present on T20, T23, T32.
- `Query` uses `IMPEncoderCHNStat` on old SDK, `IMPEncoderChnStat` on new SDK.

### 1.4 Stream Buffer Configuration

| Function | T20 | T21 | T23 | T30 | T31 | T32 | T40 | T41 |
|----------|-----|-----|-----|-----|-----|-----|-----|-----|
| SetMaxStreamCnt | Y | Y | Y | Y | Y | Y | Y | Y |
| GetMaxStreamCnt | Y | Y | Y | Y | Y | Y | Y | Y |
| SetStreamBufSize | - | - | - | - | Y | - | Y | Y |
| GetStreamBufSize | - | - | - | - | Y | - | Y | Y |

### 1.5 Parameter Control (RC Mode / Frame Rate)

| Function | T20 | T21 | T23 | T30 | T31 | T32 | T40 | T41 |
|----------|-----|-----|-----|-----|-----|-----|-----|-----|
| SetChnAttrRcMode | Y | Y | Y | Y | Y | Y | Y | - |
| GetChnAttrRcMode | Y | Y | Y | Y | Y | Y | Y | Y |
| SetChnFrmRate | Y | Y | Y | Y | Y | Y | Y | Y |
| GetChnFrmRate | Y | Y | Y | Y | Y | Y | Y | Y |

Notes:
- T41 is missing `SetChnAttrRcMode` but has `GetChnAttrRcMode`. This may be an omission
  in the header; the function likely still exists in the library.

### 1.6 Rate Control (New SDK additions)

| Function | T20 | T21 | T23 | T30 | T31 | T32 | T40 | T41 |
|----------|-----|-----|-----|-----|-----|-----|-----|-----|
| SetChnBitRate | - | - | - | - | Y | Y | Y | Y |
| SetChnGopLength | - | - | - | - | Y | - | Y | Y |
| SetChnQp | - | - | - | - | Y | - | - | - |
| SetChnQpBounds | - | - | - | - | Y | Y | Y | Y |
| SetChnQpBoundsPerFrame | - | - | - | - | - | Y | - | Y |
| SetChnQpIPDelta | - | - | - | - | Y | - | - | - |
| SetChnMaxPictureSize | - | - | Y | - | - | Y | - | Y |

Notes:
- `SetChnBitRate(encChn, iTargetBitRate, iMaxBitRate)` -- dynamic bitrate change. New SDK only.
- `SetChnQpBoundsPerFrame` allows separate I/P frame QP bounds; only T32 and T41.
- T23's `SetChnMaxPictureSize` is separate from the T32/T41 versions (different header position but same signature).

### 1.7 GOP/ROI

| Function | T20 | T21 | T23 | T30 | T31 | T32 | T40 | T41 |
|----------|-----|-----|-----|-----|-----|-----|-----|-----|
| SetChnROI (old `IMPEncoderROICfg`) | Y | Y | Y | Y | - | - | - | - |
| GetChnROI (old `IMPEncoderROICfg`) | Y | Y | Y | Y | - | - | - | - |
| SetChnROI (new `IMPEncoderROIAttr`) | - | - | - | - | - | Y | - | - |
| GetChnROI (new `IMPEncoderROIAttr`) | - | - | - | - | - | Y | - | - |
| SetChnRoiAttr (`IMPEncoderRoiAttr`) | - | - | - | - | - | - | Y | - |
| GetChnRoiAttr (`IMPEncoderRoiAttr`) | - | - | - | - | - | - | Y | - |
| SetChnMapRoi | - | - | - | - | - | Y | Y | - |
| SetChnQpgAI | - | - | - | - | - | Y | - | - |
| GetGOPSize / SetGOPSize | Y | Y | Y | Y | - | Y | - | - |
| GetChnGopAttr / SetChnGopAttr | - | - | - | - | Y | - | Y | Y |

Notes:
- Old SDK uses `IMPEncoderROICfg` with single-region Set/Get.
- T32 uses `IMPEncoderROIAttr` which contains an array of `IMPEncoderROICfg[IMP_ENC_ROI_WIN_COUNT]`.
- T40 uses its own `IMPEncoderRoiAttr` with `IMPEncoderRoiWin st_roi[IMP_ENC_ROI_WIN_COUNT]`.
- T31/T41 have no ROI API in the header (may be exposed differently or absent).
- Old SDK GOP is controlled via `SetGOPSize`/`GetGOPSize` (`IMPEncoderGOPSizeCfg`).
- New SDK GOP is controlled via `SetChnGopAttr`/`GetChnGopAttr` (`IMPEncoderGopAttr`).

### 1.8 Old SDK Only

| Function | T20 | T21 | T23 | T30 | T31 | T32 | T40 | T41 |
|----------|-----|-----|-----|-----|-----|-----|-----|-----|
| SetChnColor2Grey / GetChnColor2Grey | Y | Y | Y | Y | - | - | - | - |
| SetChnDemask / GetChnDemask | Y | - | - | - | - | - | - | - |
| SetChnDenoise / GetChnDenoise | Y | Y | Y | Y | - | - | - | - |
| SetChnFrmUsedMode / GetChnFrmUsedMode | Y | Y | Y | Y | - | Y | - | - |
| SetChnHSkip / GetChnHSkip | Y | Y | Y | Y | - | Y | - | - |
| SetChnHSkipBlackEnhance | Y | Y | Y | Y | - | Y | - | - |
| InsertUserData | Y | Y | Y | Y | - | Y | - | - |
| SetChangeRef / GetChangeRef | Y | Y | Y | Y | - | - | - | - |
| SetMbRC / GetMbRC | Y | Y | Y | Y | - | - | - | - |
| SetSuperFrameCfg / GetSuperFrameCfg | Y | Y | Y | Y | - | Y | - | - |
| SetH264TransCfg / GetH264TransCfg | Y | Y | Y | Y | - | Y | - | - |
| SetH265TransCfg / GetH265TransCfg | - | Y | - | Y | - | Y | - | - |
| SetQpgMode / GetQpgMode | - | Y | Y | Y | - | Y | - | - |
| SetJpegeQl / GetJpegeQl | Y | Y | Y | Y | - | - | - | Y |

Notes:
- `Demask` (de-mosaicking) is T20-only.
- `Denoise` is present on all old-SDK SoCs (T20/T21/T23/T30) but absent from new SDK.
- `HSkip` (hierarchical skip-frame reference) is old-SDK plus T32.
- `Color2Grey` is old-SDK only (T20/T21/T23/T30).
- T41 still has `SetJpegeQl`/`GetJpegeQl` despite being new SDK.

### 1.9 New SDK Only

| Function | T20 | T21 | T23 | T30 | T31 | T32 | T40 | T41 |
|----------|-----|-----|-----|-----|-----|-----|-----|-----|
| SetDefaultParam | - | - | - | - | Y | Y | Y | Y |
| SetbufshareChn | - | - | - | - | Y | - | Y | Y |
| SetChnResizeMode | - | - | - | - | Y | - | Y | Y |
| SetChnEntropyMode | - | - | - | - | Y | - | - | - |
| SetFrameRelease | - | - | - | - | Y | - | Y | - |
| GetChnEncType (`IMPEncoderEncType`) | - | - | - | - | Y | - | Y | Y |
| GetChnEncType (`IMPPayloadType`) | - | Y | Y | Y | - | Y | - | - |
| SetPool / GetPool | - | - | Y | - | Y | Y | Y | Y |
| GetChnAveBitrate | - | - | - | - | Y | - | - | - |
| GetChnEvalInfo | - | - | - | - | Y | - | - | - |
| SetChnSeiAttr / GetChnSeiAttr | - | - | - | - | - | - | Y | - |

Notes:
- `SetDefaultParam` is a convenience function to fill `IMPEncoderChnAttr` with sensible
  defaults.  Signature varies: T32 has `uBufSize` parameter; T31/T40/T41 have
  `uMaxSameSenceCnt` instead.
- `SetbufshareChn` allows JPEG channel to share buffer with H264/H265 channel.
- `SetPool`/`GetPool` binds encoder channel to a memory pool to avoid rmem fragmentation.

### 1.10 Fisheye

| Function | T20 | T21 | T23 | T30 | T31 | T32 | T40 | T41 |
|----------|-----|-----|-----|-----|-----|-----|-----|-----|
| SetFisheyeEnableStatus | Y | Y | Y | Y | Y | - | Y | - |
| GetFisheyeEnableStatus | Y | Y | Y | Y | Y | - | Y | - |

### 1.11 YUV Encoding (Unbound Mode)

| Function | T20 | T21 | T23 | T30 | T31 | T32 | T40 | T41 |
|----------|-----|-----|-----|-----|-----|-----|-----|-----|
| YuvInit | - | - | Y | - | - | Y | Y | Y |
| YuvEncode | - | - | Y | - | - | Y | Y | Y |
| YuvExit | - | - | Y | - | - | Y | Y | Y |
| YuvInit_Ex | - | - | - | - | - | Y | - | Y |
| YuvEncode_Ex | - | - | - | - | - | Y | - | Y |
| YuvExit_Ex | - | - | - | - | - | Y | - | Y |
| InputJpege | - | - | Y | - | - | Y | - | Y |
| InputJpege_Ex | - | - | - | - | - | Y | - | Y |

### 1.12 T32 Outliers

These functions exist ONLY on T32:

| Function | Description |
|----------|-------------|
| `SetIvpuBsSize` | Set shared IVPU bitstream buffer size (also T41) |
| `SetJpegBsSize` | Set JPEG bitstream buffer size |
| `SetRdBufShare` | Enable read buffer sharing |
| `SetMultiSectionMode` | Multi-section encoding mode (also T23) |
| `MultiProcessInit` | Initialize multi-process encoding |
| `KernEnc_Stop` | Stop kernel encoder |
| `KernEnc_GetStream` | Get stream from kernel encoder |
| `KernEnc_Release` | Release kernel encoder stream |
| `KernEnc_GetStatus` | Get kernel encoder status |
| `SetH264Vui` / `GetH264Vui` | Set/Get H.264 VUI parameters |
| `SetH265Vui` / `GetH265Vui` | Set/Get H.265 VUI parameters |
| `SetGDRCfg` / `GetGDRCfg` | Gradual Decoder Refresh config (also T23) |
| `RequestGDR` | Request GDR refresh (also T23) |
| `SetPskipCfg` / `GetPskipCfg` | Adaptive P-skip config |
| `RequestPskip` | Request P-skip |
| `SetJpegQp` / `GetJpegQp` | JPEG QP control |
| `SetChnCrop` / `GetChnCrop` | Dynamic encoder crop (also T23) |
| `SetGOPSize` / `GetGOPSize` | GOP size config (T32 equivalent of `SetChnGopLength`) |

### 1.13 T41 Unique

| Function | Description |
|----------|-------------|
| `SetIvpuBsSize` | Set IVPU bitstream buffer size |
| `SetAvpuBsShare` | Enable/configure AVPU shared bitstream |
| `SetAvpuJpegQp` | Set AVPU JPEG QP dynamically |
| `VbmAlloc` / `VbmFree` | Video buffer memory allocate/free |
| `VbmV2P` / `VbmP2V` | Virtual-to-physical / physical-to-virtual address conversion |
| `VbmAlloc_Ex` / `VbmFree_Ex` | Secondary process VBM alloc/free |

### 1.14 T23 Specific

| Function | Description |
|----------|-------------|
| `Setframelossthd` / `Getframelossthd` | Frame loss threshold |
| `SetChnInitQP` | Set channel initial QP |
| `SetMultiSectionMode` | Multi-section encoding mode |
| `SetGDRCfg` / `GetGDRCfg` / `RequestGDR` | GDR support |
| `YuvGetCrop` / `YuvSetCrop` | Crop for unbound YUV encoder |
| `YuvRequestIDR` | IDR request for unbound YUV encoder |
| `SetChnCrop` / `GetChnCrop` | Dynamic encoder crop |

---

## 2. Channel Attribute Struct

The top-level channel attribute struct is the single most important type difference.

### 2.1 Old SDK: `IMPEncoderCHNAttr` (T20, T21, T23, T30, T32)

```c
typedef struct {
    IMPEncoderAttr      encAttr;     // Encoder attributes
    IMPEncoderRcAttr    rcAttr;      // Rate control attributes
} IMPEncoderCHNAttr;                 // T20
```

On T21/T23/T30 an additional field is present:
```c
typedef struct {
    IMPEncoderAttr      encAttr;
    IMPEncoderRcAttr    rcAttr;
    bool                bEnableIvdc; // Enable ISP-VPU Direct Connect
} IMPEncoderCHNAttr;                 // T21, T23, T30
```

T32 uses the old name `IMPEncoderCHNAttr` but with the new-style internal structs:
```c
typedef struct {
    IMPEncoderEncAttr   encAttr;     // New-style encoder attrs
    IMPEncoderRcAttr    rcAttr;      // New-style RC attrs
    IMPEncoderGopAttr   gopAttr;     // Separate GOP control
    bool                bEnableIvdc;
} IMPEncoderCHNAttr;                 // T32 -- hybrid
```

### 2.2 New SDK: `IMPEncoderChnAttr` (T31, T40, T41)

```c
typedef struct {
    IMPEncoderEncAttr   encAttr;     // Profile-based encoder attrs
    IMPEncoderRcAttr    rcAttr;      // Unified RC attrs
    IMPEncoderGopAttr   gopAttr;     // Separate GOP control
} IMPEncoderChnAttr;                 // T31, T40
```

T41 also uses `IMPEncoderChnAttr` with identical layout to T31/T40.

### 2.3 Encoder Attribute Sub-Struct Comparison

#### Old SDK: `IMPEncoderAttr`

```c
typedef struct {
    IMPPayloadType          enType;      // PT_H264, PT_JPEG, PT_H265
    uint32_t                bufSize;     // Buffer size (0 = auto)
    uint32_t                profile;     // 0=baseline, 1=main, 2=high
    uint32_t                picWidth;    // Encoding width
    uint32_t                picHeight;   // Encoding height
    IMPEncoderCropCfg       crop;        // Crop config
    IMPEncoderUserDataCfg   userData;    // User data insertion config
} IMPEncoderAttr;
```

#### New SDK: `IMPEncoderEncAttr`

```c
typedef struct {
    IMPEncoderProfile       eProfile;    // Composite: type<<24 | profile_idc
    uint8_t                 uLevel;
    uint8_t                 uTier;
    uint16_t                uWidth;      // Encoding width (was picWidth)
    uint16_t                uHeight;     // Encoding height (was picHeight)
    IMPEncoderPicFormat     ePicFormat;  // 420/422/400
    uint32_t                eEncOptions; // Bitmask of IMPEncoderEncOptions
    uint32_t                eEncTools;   // Bitmask of IMPEncoderEncTools
    IMPEncoderCropCfg       crop;
} IMPEncoderEncAttr;
```

T41 adds two extra fields to `IMPEncoderEncAttr`:
```c
typedef struct {
    IMPEncoderVpuType       encVputype;  // AVPU or IVPU
    IMPEncoderProfile       eProfile;
    // ... same as above ...
    uint32_t                bufSize;     // Buffer size (like old SDK)
} IMPEncoderEncAttr;                     // T41 variant
```

#### Key Field Mapping

| Old SDK Field | New SDK Field | Notes |
|---------------|---------------|-------|
| `enType` (IMPPayloadType) | `eProfile` (IMPEncoderProfile) | Profile encodes both type and profile level |
| `bufSize` | absent (T31/T40), present (T41) | T31/T40 removed it; T41 added it back |
| `profile` (0/1/2) | embedded in `eProfile` | See profile encoding below |
| `picWidth` | `uWidth` | uint32_t -> uint16_t |
| `picHeight` | `uHeight` | uint32_t -> uint16_t |
| `userData` | absent | User data handled differently in new SDK |
| absent | `uLevel`, `uTier` | HEVC level/tier support |
| absent | `ePicFormat` | Pixel format selection |
| absent | `eEncOptions` | Encoder option bitmask |
| absent | `eEncTools` | Encoder tool bitmask |
| absent | `encVputype` | T41 only: AVPU vs IVPU selection |

---

## 3. Rate Control Types

### 3.1 RC Mode Enum

#### Old SDK: `IMPEncoderRcMode` (T20, T21, T23, T30)

```c
typedef enum {
    ENC_RC_MODE_FIXQP  = 0,
    ENC_RC_MODE_CBR    = 1,
    ENC_RC_MODE_VBR    = 2,
    ENC_RC_MODE_SMART  = 3,
    ENC_RC_MODE_INV    = 4,
} IMPEncoderRcMode;
```

#### New SDK: `IMPEncoderRcMode` (T31, T32, T40, T41)

```c
typedef enum {
    IMP_ENC_RC_MODE_FIXQP           = 0x0,
    IMP_ENC_RC_MODE_CBR             = 0x1,
    IMP_ENC_RC_MODE_VBR             = 0x2,
    IMP_ENC_RC_MODE_CAPPED_VBR     = 0x4,
    IMP_ENC_RC_MODE_CAPPED_QUALITY = 0x8,
    IMP_ENC_RC_MODE_INVALID        = 0xff,
} IMPEncoderRcMode;
```

Key differences:
- Old SDK: `SMART` mode (value 3). New SDK: replaced by `CAPPED_VBR` and `CAPPED_QUALITY`.
- New SDK values are bitmask-style (0x1, 0x2, 0x4, 0x8) rather than sequential.
- The name `IMPEncoderRcMode` is reused but values are incompatible.

### 3.2 RC Attribute Structs

#### Old SDK: Per-codec structs

```c
typedef struct {
    IMPEncoderRcMode rcMode;
    union {
        IMPEncoderAttrH264FixQP   attrH264FixQp;
        IMPEncoderAttrH264CBR     attrH264Cbr;
        IMPEncoderAttrH264VBR     attrH264Vbr;
        IMPEncoderAttrH264Smart   attrH264Smart;
        // T21/T30 also have:
        IMPEncoderAttrH265FixQP   attrH265FixQp;
        IMPEncoderAttrH265CBR     attrH265Cbr;
        IMPEncoderAttrH265VBR     attrH265Vbr;
        IMPEncoderAttrH265Smart   attrH265Smart;
    };
} IMPEncoderAttrRcMode;
```

Old SDK CBR example (`IMPEncoderAttrH264CBR`):
```c
typedef struct {
    uint32_t  maxQp;          // Max QP [0-51]
    uint32_t  minQp;          // Min QP
    uint32_t  outBitRate;     // Target bitrate (kbps)
    int       iBiasLvl;       // I-frame QP bias [-3,3]
    uint32_t  frmQPStep;      // Inter-frame QP step
    uint32_t  gopQPStep;      // Inter-GOP QP step
    bool      adaptiveMode;   // Adaptive mode (T20 only)
    bool      gopRelation;    // GOP relationship
} IMPEncoderAttrH264CBR;
```

#### New SDK: Unified codec-agnostic structs

```c
typedef struct {
    IMPEncoderRcMode rcMode;
    union {
        IMPEncoderAttrFixQP         attrFixQp;
        IMPEncoderAttrCbr           attrCbr;
        IMPEncoderAttrVbr           attrVbr;
        IMPEncoderAttrCappedVbr     attrCappedVbr;
        IMPEncoderAttrCappedQuality attrCappedQuality;
    };
} IMPEncoderAttrRcMode;
```

New SDK CBR (`IMPEncoderAttrCbr`):
```c
typedef struct {
    uint32_t  uTargetBitRate;   // Target bitrate
    int16_t   iInitialQP;       // Initial QP
    int16_t   iMinQP;           // Min QP
    int16_t   iMaxQP;           // Max QP
    int16_t   iIPDelta;         // QP delta between I and P frames
    int16_t   iPBDelta;         // QP delta between P and B frames
    uint32_t  eRcOptions;       // Bitmask of IMPEncoderRcOptions
    uint32_t  uMaxPictureSize;  // Max picture size
} IMPEncoderAttrCbr;
```

New SDK FixQP:
- T31/T40: `{ int16_t iInitialQP; }` -- single field
- T41: `{ int16_t iInitialQP; int16_t iMinQP; int16_t iMaxQP; }` -- has QP bounds

#### Old SDK `IMPEncoderRcAttr` (outer wrapper)

```c
typedef struct {
    IMPEncoderFrmRate       outFrmRate;
    uint32_t                maxGop;
    IMPEncoderAttrRcMode    attrRcMode;
    IMPEncoderAttrFrmUsed   attrFrmUsed;    // Frame usage mode
    IMPEncoderAttrDemask    attrDemask;      // T20 only
    IMPEncoderAttrDenoise   attrDenoise;
    IMPEncoderAttrInitHSkip attrHSkip;
} IMPEncoderRcAttr;
```

#### New SDK `IMPEncoderRcAttr` (outer wrapper)

```c
typedef struct {
    IMPEncoderAttrRcMode    attrRcMode;
    IMPEncoderFrmRate       outFrmRate;
} IMPEncoderRcAttr;
```

Note the order reversal: new SDK puts `attrRcMode` first, `outFrmRate` second.

### 3.3 `IMPEncoderRcOptions` Enum

Present on: T31, T32, T40, T41 (all new-style SoCs).

```c
typedef enum IMPEncoderRcOptions {
    IMP_ENC_RC_OPT_NONE           = 0x00000000,
    IMP_ENC_RC_SCN_CHG_RES        = 0x00000001,  // Scene change response
    IMP_ENC_RC_DELAYED            = 0x00000002,
    IMP_ENC_RC_STATIC_SCENE       = 0x00000004,
    IMP_ENC_RC_ENABLE_SKIP        = 0x00000008,
    IMP_ENC_RC_OPT_SC_PREVENTION  = 0x00000010,
    IMP_ENC_RC_MAX_ENUM,
} IMPEncoderRcOptions;
```

Not present on: T20, T21, T23, T30 (old-style SoCs).

### 3.4 H-Skip Support

`IMPEncoderAttrHSkip` / `IMPEncoderAttrInitHSkip` present on:
- T20, T21, T23, T30 (in `IMPEncoderRcAttr.attrHSkip`)
- T32 (standalone `SetChnHSkip`/`GetChnHSkip`)
- Absent from T31, T40, T41.

### 3.5 GOP Attribute (New SDK)

Present on: T31, T32, T40, T41.

```c
typedef struct {
    IMPEncoderGopCtrlMode   uGopCtrlMode;   // DEFAULT=0x02, PYRAMIDAL=0x04, SMARTP=0xfe
    uint16_t                uGopLength;
    uint8_t/uint16_t        uNotifyUserLTInter;  // T31: uint16_t, T40/T41: uint8_t
    uint32_t                uMaxSameSenceCnt;
    bool                    bEnableLT;       // Long-term reference
    uint32_t                uFreqLT;
    bool                    bLTRC;
} IMPEncoderGopAttr;
```

---

## 4. Stream Pack Access (CRITICAL for HAL)

This is the single most important structural difference between old and new SDK.

### 4.1 Old SDK Stream Access (T20, T21, T23, T30)

#### `IMPEncoderPack` (Old)

```c
typedef struct {
    uint32_t            phyAddr;    // Physical address of THIS pack
    uint32_t            virAddr;    // Virtual address of THIS pack
    uint32_t            length;     // Length of THIS pack
    int64_t             timestamp;  // Timestamp (us)
    bool                frameEnd;   // End-of-frame marker
    IMPEncoderDataType  dataType;   // NAL type
} IMPEncoderPack;
```

#### `IMPEncoderStream` (Old)

```c
typedef struct {
    IMPEncoderPack  *pack;       // Array of packs
    uint32_t        packCount;   // Number of packs
    uint32_t        seq;         // Frame sequence number
    union {
        IMPRefType  refType;     // Reference type (T20 uses union wrapper)
    };
} IMPEncoderStream;
```

T21/T23/T30 variant (no union wrapper for refType):
```c
typedef struct {
    IMPEncoderPack  *pack;
    uint32_t        packCount;
    uint32_t        seq;
    IMPRefType      refType;    // Direct field
} IMPEncoderStream;
```

#### NAL Type Access (Old)

```c
// Old SDK: via dataType union
typedef union {
    IMPEncoderH264NaluType  h264Type;   // T20 only has H264
    IMPEncoderH265NaluType  h265Type;   // T21/T23/T30 add H265
} IMPEncoderDataType;

// Access pattern:
pack->dataType.h264Type  // for H.264
pack->dataType.h265Type  // for H.265 (T21/T23/T30)
```

#### Old SDK Data Access Pattern

```c
// Each pack has its own virAddr -- data is directly at that address
for (int i = 0; i < stream.packCount; i++) {
    void *data = (void *)stream.pack[i].virAddr;
    int   len  = stream.pack[i].length;
    write(fd, data, len);
}
```

### 4.2 New SDK Stream Access (T31, T32, T40, T41)

#### `IMPEncoderPack` (New)

```c
typedef struct {
    uint32_t            offset;     // Offset from stream base address
    uint32_t            length;     // Length of THIS pack
    int64_t             timestamp;  // Timestamp (us)
    bool                frameEnd;   // End-of-frame marker
    IMPEncoderNalType   nalType;    // NAL type (renamed)
    IMPEncoderSliceType sliceType;  // Slice type (new)
} IMPEncoderPack;
```

**Critical change**: `phyAddr`/`virAddr` per pack are replaced by a single `offset`.

#### `IMPEncoderStream` (New)

```c
typedef struct {
    uint32_t            phyAddr;     // Physical addr of WHOLE stream buffer
    uint32_t            virAddr;     // Virtual addr of WHOLE stream buffer
    uint32_t            streamSize;  // Total allocated buffer size
    IMPEncoderPack      *pack;       // Array of packs
    uint32_t            packCount;   // Number of packs
    uint32_t            seq;         // Frame sequence number
    bool                isVI;        // Is virtual I-frame
    // T31 additionally has:
    union {
        IMPEncoderStreamInfo streamInfo;  // Encoding stats (T31)
        IMPEncoderJpegInfo   jpegInfo;    // JPEG stats (T31)
    };
} IMPEncoderStream;
```

The `phyAddr`/`virAddr` now belong to `IMPEncoderStream`, not to individual packs.

#### NAL Type Access (New)

```c
// New SDK: via nalType union
typedef union {
    IMPEncoderH264NaluType  h264NalType;   // Note: renamed from h264Type
    IMPEncoderH265NaluType  h265NalType;   // Note: renamed from h265Type
} IMPEncoderNalType;

// Access pattern:
pack->nalType.h264NalType  // for H.264
pack->nalType.h265NalType  // for H.265
```

#### New SDK `IMPEncoderSliceType`

New SDK adds slice type information (not present in old SDK):

```c
typedef enum {
    IMP_ENC_SLICE_B       = 0,  // B Slice
    IMP_ENC_SLICE_P       = 1,  // P Slice
    IMP_ENC_SLICE_I       = 2,  // I Slice
    IMP_ENC_SLICE_SP      = 3,  // AVC SP Slice
    IMP_ENC_SLICE_SI      = 4,  // AVC SI Slice
    IMP_ENC_SLICE_GOLDEN  = 3,  // Golden Slice (alias for SP)
    IMP_ENC_SLICE_CONCEAL = 6,  // Concealment Slice
    IMP_ENC_SLICE_SKIP    = 7,  // Skip Slice
    IMP_ENC_SLICE_REPEAT  = 8,  // Repeat Slice
    IMP_ENC_SLICE_MAX_ENUM,
} IMPEncoderSliceType;
```

#### New SDK Data Access Pattern

```c
// Stream has ONE virAddr; packs have OFFSETS into it
// Must handle possible wrap-around in ring buffer!
for (int i = 0; i < stream.packCount; i++) {
    void *data = (void *)(stream.virAddr + stream.pack[i].offset);
    int   len  = stream.pack[i].length;

    // Check for ring-buffer wrap: if offset+length > streamSize
    if (stream.pack[i].offset + len > stream.streamSize) {
        int part1 = stream.streamSize - stream.pack[i].offset;
        int part2 = len - part1;
        write(fd, data, part1);
        write(fd, (void *)stream.virAddr, part2);
    } else {
        write(fd, data, len);
    }
}
```

### 4.3 Stream Access Summary Table

| Aspect | Old SDK (T20/T21/T23/T30) | New SDK (T31/T32/T40/T41) |
|--------|---------------------------|---------------------------|
| Pack address | `pack[i].virAddr` (absolute) | `stream.virAddr + pack[i].offset` |
| Pack phyAddr | `pack[i].phyAddr` | Not per-pack; `stream.phyAddr` |
| NAL type field | `pack[i].dataType.h264Type` | `pack[i].nalType.h264NalType` |
| NAL type union name | `IMPEncoderDataType` | `IMPEncoderNalType` |
| H264 member name | `.h264Type` | `.h264NalType` |
| H265 member name | `.h265Type` | `.h265NalType` |
| Slice type | not available | `pack[i].sliceType` |
| Stream buffer info | not available | `stream.virAddr`, `stream.streamSize` |
| Ring buffer wrap | not needed | MUST handle wrap-around |
| isVI field | not available | `stream.isVI` |
| Stream stats | not available | `stream.streamInfo` (T31) |

---

## 5. Encoder Profiles/Types

### 5.1 Old SDK: `IMPPayloadType` + `profile` field

```c
// enType uses IMPPayloadType (defined in imp_common.h):
// PT_H264, PT_JPEG, PT_H265

// profile is a plain integer:
// 0 = baseline, 1 = main, 2 = high
```

### 5.2 New SDK: `IMPEncoderProfile` composite enum

```c
typedef enum {
    IMP_ENC_TYPE_AVC   = 0,
    IMP_ENC_TYPE_HEVC  = 1,
    IMP_ENC_TYPE_JPEG  = 4,
} IMPEncoderEncType;

#define IMP_ENC_AVC_PROFILE_IDC_BASELINE   66
#define IMP_ENC_AVC_PROFILE_IDC_MAIN       77
#define IMP_ENC_AVC_PROFILE_IDC_HIGH       100
#define IMP_ENC_HEVC_PROFILE_IDC_MAIN      1

typedef enum {
    IMP_ENC_PROFILE_AVC_BASELINE = ((0 << 24) | 66),   // = 0x00000042
    IMP_ENC_PROFILE_AVC_MAIN     = ((0 << 24) | 77),   // = 0x0000004D
    IMP_ENC_PROFILE_AVC_HIGH     = ((0 << 24) | 100),   // = 0x00000064
    IMP_ENC_PROFILE_HEVC_MAIN    = ((1 << 24) | 1),     // = 0x01000001
    IMP_ENC_PROFILE_JPEG         = (4 << 24),            // = 0x04000000
} IMPEncoderProfile;
```

The profile value encodes both the codec type (top byte) and the profile IDC (low bytes).
To extract the codec type: `(eProfile >> 24) & 0xFF`.

### 5.3 H.265 Support by SoC

| SoC | H.265 Support | Notes |
|-----|---------------|-------|
| T20 | No | H264 + JPEG only |
| T21 | Declared but "Unsupport" | H265 RC structs exist but marked "Unsupport H.265 protocol" |
| T23 | Declared but "Unsupport" | Same as T21 |
| T30 | Yes | Full H265 support with separate RC structs |
| T31 | Yes | Via `IMP_ENC_PROFILE_HEVC_MAIN` |
| T32 | Yes | Via `IMP_ENC_PROFILE_HEVC_MAIN` |
| T40 | Yes | Via `IMP_ENC_PROFILE_HEVC_MAIN` |
| T41 | Yes | Via `IMP_ENC_PROFILE_HEVC_MAIN` |

### 5.4 Encoder Options (New SDK Only)

```c
typedef enum {
    IMP_ENC_OPT_QP_TAB_RELATIVE   = 0x00000001,
    IMP_ENC_OPT_FIX_PREDICTOR     = 0x00000002,
    IMP_ENC_OPT_CUSTOM_LDA        = 0x00000004,
    IMP_ENC_OPT_ENABLE_AUTO_QP    = 0x00000008,
    IMP_ENC_OPT_ADAPT_AUTO_QP     = 0x00000010,
    IMP_ENC_OPT_COMPRESS          = 0x00000020,
    IMP_ENC_OPT_FORCE_REC         = 0x00000040,
    IMP_ENC_OPT_FORCE_MV_OUT      = 0x00000080,
    IMP_ENC_OPT_HIGH_FREQ         = 0x00002000,
    IMP_ENC_OPT_SRD               = 0x00008000,
    IMP_ENC_OPT_FORCE_MV_CLIP     = 0x00020000,
    IMP_ENC_OPT_RDO_COST_MODE     = 0x00040000,
    // T40 also has:
    IMP_ENC_OPT_L2_CACHE          = 0x00000800,  // T40 only
    // T41 also has:
    IMP_ENC_OPT_NONE              = 0x00000000,
    IMP_ENC_OPT_LOWLAT_SYNC       = 0x00000100,
    IMP_ENC_OPT_LOWLAT_INT        = 0x00000200,
    IMP_ENC_OPT_MMA               = 0x01000000,
    IMP_ENC_OPT_DYN_SRD           = 0x02000000,
} IMPEncoderEncOptions;
```

### 5.5 VPU Type (T41 Only)

```c
typedef enum {
    IMP_ENC_TPYE_AVPU = 0x00000001,  // AVPU encoder
    IMP_ENC_TPYE_IVPU = 0x00000002,  // IVPU encoder
} IMPEncoderVpuType;
```

T41 has two separate encoding units (AVPU and IVPU) and the `encVputype` field in
`IMPEncoderEncAttr` selects which one to use.

---

## 6. Additional Struct Differences

### 6.1 Channel State Struct

Old SDK:
```c
typedef struct {
    bool      registered;
    uint32_t  leftPics;
    uint32_t  leftStreamBytes;
    uint32_t  leftStreamFrames;
    uint32_t  curPacks;
    uint32_t  work_done;
} IMPEncoderCHNStat;           // Capital CHN
```

New SDK:
```c
typedef struct {
    bool      registered;
    uint32_t  leftPics;
    uint32_t  leftStreamBytes;
    uint32_t  leftStreamFrames;
    uint32_t  curPacks;
    uint32_t  work_done;
} IMPEncoderChnStat;           // Mixed-case Chn
```

Fields are identical; only the type name differs.

### 6.2 Stream Info (T31 Only)

T31's `IMPEncoderStream` contains encoding statistics not present on other SoCs:

```c
typedef struct {
    int32_t   iNumBytes;     // Bytes in stream
    uint32_t  uNumIntra;     // 8x8 blocks coded intra
    uint32_t  uNumSkip;      // 8x8 blocks coded skip
    uint32_t  uNumCU8x8;    // Number of 8x8 CUs
    uint32_t  uNumCU16x16;  // Number of 16x16 CUs
    uint32_t  uNumCU32x32;  // Number of 32x32 CUs
    uint32_t  uNumCU64x64;  // Number of 64x64 CUs
    int16_t   iSliceQP;     // Slice QP
    int16_t   iMinQP;       // Min QP used
    int16_t   iMaxQP;       // Max QP used
} IMPEncoderStreamInfo;

typedef struct {
    int32_t   iNumBytes;
    int16_t   iQPfactor;     // JPEG QP parameter
} IMPEncoderJpegInfo;
```

### 6.3 ROI Structs Comparison

#### Old SDK ROI (`IMPEncoderROICfg`) -- T20/T21/T23/T30

```c
typedef struct {
    uint32_t  u32Index;    // ROI region index [0-7]
    bool      bEnable;
    bool      bRelatedQp;  // 0=absolute, 1=relative
    int       s32Qp;
    IMPRect   rect;
} IMPEncoderROICfg;
```

#### T32 ROI (`IMPEncoderROIAttr`)

```c
#define IMP_ENC_ROI_WIN_COUNT  10  // T32: up to 10 ROI regions
typedef struct {
    IMPEncoderROICfg roi[IMP_ENC_ROI_WIN_COUNT];
} IMPEncoderROIAttr;
```

#### T40 ROI (`IMPEncoderRoiAttr` / `IMPEncoderRoiWin`)

```c
#define IMP_ENC_ROI_WIN_COUNT  10
typedef struct {
    bool                enable;
    IMPEncoderRoiRect   rect;       // Separate rect type (not IMPRect)
    IMPEncoderQPMode    mode;       // DELTA or FIXED_QP
    int8_t              qp;         // [-26, 25] for delta mode
} IMPEncoderRoiWin;

typedef struct {
    IMPEncoderRoiWin st_roi[IMP_ENC_ROI_WIN_COUNT];
} IMPEncoderRoiAttr;
```

### 6.4 JPEG Quantization Table

Old SDK (T20): `uint8_t qmem_table[256]` -- 256 bytes.
New SDK (T23/T30/T32/T41): `uint8_t qmem_table[128]` -- 128 bytes.

### 6.5 Super Frame Config

Old SDK:
```c
typedef struct {
    IMPEncoderSuperFrmMode  superFrmMode;
    uint32_t                superIFrmBitsThr;
    uint32_t                superPFrmBitsThr;
    uint32_t                superBFrmBitsThr;
    IMPEncoderRcPriority    rcPriority;
} IMPEncoderSuperFrmCfg;
```

T32 adds: `uint8_t maxReEncodeTimes;` -- maximum re-encode attempts [1,3].

### 6.6 Macroblock RC Mode

Old SDK (T23): `IMPEncoderQpgMode`:
```c
ENC_QPG_CLOSE=0, ENC_QPG_CRP=1, ENC_QPG_SAS=2, ENC_QPG_SASM=3,
ENC_QPG_MBRC=4, ENC_QPG_CRP_TAB=5, ENC_QPG_SAS_TAB=6, ENC_QPG_SASM_TAB=7
```

T32 `IMPEncoderQpgMode` (different values):
```c
ENC_MBQP_AUTO=0, ENC_MBQP_MAD=1, ENC_MBQP_TEXT=2, ENC_MBQP_ROWRC=3,
ENC_MBQP_MAD_ROWRC=4, ENC_MBQP_TEXT_ROWRC=5
```

### 6.7 T32 VUI Structures

T32 uniquely provides VUI (Video Usability Information) control:

```c
typedef struct {
    IMPEncoderVuiAspectRatio       vuiAspectRatio;
    IMPEncoderH264VuiTimeInfo      vuiTimeInfo;
    IMPEncoderVuiVideoSignal       vuiVideoSignal;
    IMPEncoderVuiBitstreamRestric  vuiBitstreamRestric;
} IMPEncoderH264Vui;

typedef struct {
    IMPEncoderVuiAspectRatio       vuiAspectRatio;
    IMPEncoderH265VuiTimeInfo      vuiTimeInfo;
    IMPEncoderVuiVideoSignal       vuiVideoSignal;
    IMPEncoderVuiBitstreamRestric  vuiBitstreamRestric;
} IMPEncoderH265Vui;
```

### 6.8 T32 Kernel Encoder

T32 provides kernel-space encoding capability:

```c
typedef struct {
    IMPPayloadType   type;       // Stream type
    uint32_t         bufAddr;    // Virtual address of stream buffer
    uint32_t         bufLen;     // Stream buffer total length
    uint32_t         strmCnt;    // Stream count
    uint32_t         index;      // Current stream index
    uint32_t         strmAddr;   // Current stream virtual address
    uint32_t         strmLen;    // Current stream length
    int64_t          timestamp;  // Timestamp (us)
    IMPRefType       refType;    // Reference type
} IMPEncoderKernEncOut;
```

### 6.9 T32 GDR / P-skip

T32 (and T23) support Gradual Decoder Refresh:
```c
typedef struct {
    bool  enable;        // Enable GDR
    int   gdrCycle;      // GDR interval [3, 65535]
    int   gdrFrames;     // GDR frames [2, 10]
} IMPEncoderGDRCfg;
```

T32 uniquely supports adaptive P-skip:
```c
typedef struct {
    bool  enable;
    int   pskipMaxFrames;    // Max P frames in skip mode
    int   pskipThr;          // Threshold (pskipThr/100 * bitrate)
} IMPEncoderPskipCfg;
```

### 6.10 T41 YUV Encoding Input

T41's `IMPEncoderYuvIn` is structured differently from the old SDK version:
```c
// T41 variant:
typedef struct {
    IMPEncoderEncType type;       // Encode type (not IMPPayloadType)
    IMPEncoderRcMode  mode;       // RC mode
    uint16_t          frameRate;  // Frame rate (single value, not Num/Den)
    uint16_t          gopLength;
    uint32_t          targetBitrate;
    uint32_t          maxBitrate;
    int16_t           initQp;
    uint16_t          minQp;
    uint16_t          maxQp;
    uint32_t          maxPictureSize;
    bool              useUserBuf;
    uint32_t          userBuf_vaddr;
} IMPEncoderYuvIn;
```

Compared to old SDK (T23/T30):
```c
// Old variant:
typedef struct {
    IMPPayloadType       type;
    IMPEncoderAttrRcMode mode;       // Full RC mode struct
    IMPEncoderFrmRate    outFrmRate;  // Num/Den frame rate
    uint32_t             maxGop;
} IMPEncoderYuvIn;
```

### 6.11 T40 SEI Support

T40 uniquely provides SEI (Supplemental Enhancement Information) attribute control:

```c
typedef enum {
    IMP_ENC_SEI_MODE_NONE   = 0x00,
    IMP_ENC_SEI_MODE_INTRA  = 0x01,  // SEI on I-frames only
    IMP_ENC_SEI_MODE_ALL    = 0x02,  // SEI on all frames
} IMPEncoderSEIMode;

typedef struct {
    IMPEncoderSEIMode SeiMode;
    uint8_t*          uuid;          // 16-byte UUID for verification
    uint8_t*          userData;
    uint8_t           userDataSize;  // Max 128 bytes
} IMPEncoderSeiAttr;
```

### 6.12 T41 Error Codes

T41 uniquely defines encoder-specific error codes:

```c
enum {
    IMP_OK_ENC_ALL              = 0x0,
    IMP_ERR_ENC_CHNID           = 0x80040001,
    IMP_ERR_ENC_PARAM           = 0x80040002,
    IMP_ERR_ENC_EXIST           = 0x80040004,
    IMP_ERR_ENC_UNEXIST         = 0x80040008,
    IMP_ERR_ENC_NULL_PTR        = 0x80040010,
    IMP_ERR_ENC_NOT_CONFIG      = 0x80040020,
    IMP_ERR_ENC_NOT_SUPPORT     = 0x80040040,
    IMP_ERR_ENC_PERM            = 0x80040080,
    IMP_ERR_ENC_NOMEM           = 0x80040100,
    IMP_ERR_ENC_NOBUF           = 0x80040200,
    IMP_ERR_ENC_BUF_EMPTY       = 0x80040400,
    IMP_ERR_ENC_BUF_FULL        = 0x80040800,
    IMP_ERR_ENC_BUF_SIZE        = 0x80041000,
    IMP_ERR_ENC_SYS_NOTREADY    = 0x80042000,
    IMP_ERR_ENC_OVERTIME        = 0x80044000,
    IMP_ERR_ENC_RESOURCE_REQUEST = 0x80048000,
};
```

---

## 7. HAL Implications

### 7.1 `create_channel()` Abstraction

The HAL must handle two completely different channel creation paths:

**Old SDK path** (T20/T21/T23/T30):
1. Fill `IMPEncoderAttr` with `enType` (IMPPayloadType), `picWidth`, `picHeight`, `profile` (int), `bufSize`.
2. Fill `IMPEncoderRcAttr` with frame rate, GOP, RC mode union (per-codec: `attrH264Cbr`), plus embedded denoise/demask/hskip/frmused structs.
3. Combine into `IMPEncoderCHNAttr`.
4. Call `IMP_Encoder_CreateChn(chn, &attr)`.

**New SDK path** (T31/T40/T41):
1. Optionally call `IMP_Encoder_SetDefaultParam(...)` to pre-fill the struct.
2. Fill `IMPEncoderEncAttr` with `eProfile` (composite enum), `uWidth`, `uHeight`, `ePicFormat`, `eEncOptions`, `eEncTools`.
3. Fill `IMPEncoderRcAttr` with unified RC mode (`attrCbr`, `attrVbr`) and frame rate.
4. Fill `IMPEncoderGopAttr` with GOP length, control mode, long-term ref settings.
5. Combine into `IMPEncoderChnAttr`.
6. Call `IMP_Encoder_CreateChn(chn, &attr)`.

**T32 hybrid path**:
Like new SDK internally but using the old `IMPEncoderCHNAttr` type name. Has `SetDefaultParam`
with an extra `uBufSize` parameter.

**HAL design recommendation**: The HAL should define its own `hal_encoder_config_t` struct and
translate to the appropriate SDK struct at runtime based on the detected SoC family.

### 7.2 `get_frame()` Abstraction

This is the most critical HAL function. The stream data access pattern is fundamentally
different:

**Old SDK**:
```c
// Direct address per pack -- simple
for (i = 0; i < stream.packCount; i++) {
    memcpy(dst, (void*)stream.pack[i].virAddr, stream.pack[i].length);
    dst += stream.pack[i].length;
}
```

**New SDK**:
```c
// Offset-based with potential ring-buffer wrap
for (i = 0; i < stream.packCount; i++) {
    uint32_t off = stream.pack[i].offset;
    uint32_t len = stream.pack[i].length;
    uint8_t *base = (uint8_t *)stream.virAddr;

    if (off + len <= stream.streamSize) {
        memcpy(dst, base + off, len);
    } else {
        // Ring buffer wrap
        uint32_t part1 = stream.streamSize - off;
        memcpy(dst, base + off, part1);
        memcpy(dst + part1, base, len - part1);
    }
    dst += len;
}
```

**HAL design recommendation**: The HAL `get_frame()` function should return a linear buffer
to the caller.  Internally, it must detect the SDK style and use the appropriate access
pattern.  The ring-buffer wrap handling for new SDK is mandatory -- without it, frames will
be silently corrupted.

### 7.3 NAL Type Detection

The HAL needs to detect I-frames (for IDR/keyframe signaling):

**Old SDK**: `pack->dataType.h264Type == IMP_H264_NAL_SLICE_IDR`
**New SDK**: `pack->nalType.h264NalType == IMP_H264_NAL_SLICE_IDR`

For H.265:
**Old SDK**: `pack->dataType.h265Type == IMP_H265_NAL_SLICE_IDR_N_LP` (or `IDR_W_RADL`)
**New SDK**: `pack->nalType.h265NalType == IMP_H265_NAL_SLICE_IDR_N_LP`

The NAL enum values themselves are identical across all SoCs.  Only the struct member
path differs.

### 7.4 Codec Type Detection

**Old SDK**: Check `encAttr.enType` against `PT_H264`, `PT_JPEG`, `PT_H265`.
**New SDK**: Check `encAttr.eProfile >> 24` against `IMP_ENC_TYPE_AVC` (0), `IMP_ENC_TYPE_HEVC` (1), `IMP_ENC_TYPE_JPEG` (4).

Alternatively on new SDK, call `IMP_Encoder_GetChnEncType()` which returns `IMPEncoderEncType`.

### 7.5 Dynamic Bitrate Control

**Old SDK**: Must use `SetChnAttrRcMode()` with the full per-codec RC struct.
**New SDK**: Can use the simpler `SetChnBitRate(encChn, targetBitRate, maxBitRate)`.

### 7.6 GOP Control

**Old SDK**: `SetGOPSize()` / `GetGOPSize()` with simple `IMPEncoderGOPSizeCfg { int gopsize; }`.
**New SDK**: `SetChnGopAttr()` / `GetChnGopAttr()` with full `IMPEncoderGopAttr` struct
containing mode, length, long-term reference settings, etc.

### 7.7 Buffer Sharing (New SDK)

New SDK SoCs (T31/T40/T41) support `SetbufshareChn()` to let a JPEG snapshot channel
share the buffer with an H264/H265 channel, saving memory.  The HAL should use this
when both video and snapshot channels exist on the same resolution.

### 7.8 Memory Pool (T23/T31/T32/T40/T41)

`SetPool()`/`GetPool()` binds an encoder channel to a specific memory pool to avoid
rmem fragmentation.  The HAL should use this on supported SoCs for reliability.

### 7.9 T41 Dual-VPU Consideration

T41 has both AVPU and IVPU encoding units.  The `encVputype` field in `IMPEncoderEncAttr`
must be set correctly.  The HAL should default to `IMP_ENC_TPYE_AVPU` unless IVPU is
specifically needed (e.g., for dedicated JPEG encoding).

T41 also provides `SetIvpuBsSize()` and `SetAvpuBsShare()` for managing the bitstream
buffers of each VPU unit.

### 7.10 SetDefaultParam Variants

The `SetDefaultParam` function simplifies channel creation but has different signatures:

| SoC | Signature Notes |
|-----|-----------------|
| T31 | `(chnAttr, profile, rcMode, w, h, fpsNum, fpsDen, gopLen, maxSameScene, initQP, targetBR)` |
| T32 | `(chnAttr, profile, rcMode, w, h, fpsNum, fpsDen, gopLen, bufSize, initQP, targetBR)` -- has `uBufSize` instead of `uMaxSameSenceCnt` |
| T40 | Same as T31 |
| T41 | Same as T31 |

T32's variant takes `uBufSize` as the 8th parameter instead of `uMaxSameSenceCnt`,
matching its hybrid nature where `IMPEncoderEncAttr` retains the `bufSize` field.

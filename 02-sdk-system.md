# Raptor HAL: imp_system.h Cross-SoC Differences

Source headers analyzed:

| SoC | SDK Version | Language | Header Path |
|-----|-------------|----------|-------------|
| T20 | 3.12.0 | zh | `T20/3.12.0/zh/imp/imp_system.h` |
| T21 | 1.0.33 | zh | `T21/1.0.33/zh/imp/imp_system.h` |
| T23 | 1.3.0 | en | `T23/1.3.0/en/imp/imp_system.h` |
| T30 | 1.0.5 | zh | `T30/1.0.5/zh/imp/imp_system.h` |
| T31 | 1.1.6 | en | `T31/1.1.6/en/imp/imp_system.h` |
| T32 | 1.0.6 | en | `T32/1.0.6/en/imp/imp_system.h` |
| T40 | 1.3.1 | en | `T40/1.3.1/en/imp/imp_system.h` |
| T41 | 1.2.5 | en | `T41/1.2.5/en/imp/imp_system.h` |

---

## 1. Function Presence Matrix

| Function | T20 | T21 | T23 | T30 | T31 | T32 | T40 | T41 |
|----------|-----|-----|-----|-----|-----|-----|-----|-----|
| `IMP_System_Init` | Y | Y | Y | Y | Y | Y | Y | Y |
| `IMP_System_Exit` | Y | Y | Y | Y | Y | Y | Y | Y |
| `IMP_System_GetTimeStamp` | Y | Y | Y | Y | Y | Y | Y | Y |
| `IMP_System_RebaseTimeStamp` | Y | Y | Y | Y | Y | Y | Y | Y |
| `IMP_System_ReadReg32` | Y | Y | Y | Y | Y | Y | Y | Y |
| `IMP_System_WriteReg32` | Y | Y | Y | Y | Y | Y | Y | Y |
| `IMP_System_GetVersion` | Y | Y | Y | Y | Y | Y | Y | Y |
| `IMP_System_GetCPUInfo` | Y | Y | Y | Y | Y | Y | Y | Y |
| `IMP_System_Bind` | Y | Y | Y | Y | Y | Y | Y | Y |
| `IMP_System_UnBind` | Y | Y | Y | Y | Y | Y | Y | Y |
| `IMP_System_GetBindbyDest` | Y | Y | Y | Y | Y | Y | Y | Y |
| `IMP_System_MemPoolRequest` | - | - | Y | - | - | Y | Y | Y |
| `IMP_System_MemPoolFree` | - | - | - | - | - | Y | - | - |

### Function Details

#### Core System Functions (All SoCs)

```c
int IMP_System_Init(void);
int IMP_System_Exit(void);
int64_t IMP_System_GetTimeStamp(void);
int IMP_System_RebaseTimeStamp(int64_t basets);
uint32_t IMP_System_ReadReg32(uint32_t regAddr);
void IMP_System_WriteReg32(uint32_t regAddr, uint32_t value);
int IMP_System_GetVersion(IMPVersion *pstVersion);
const char* IMP_System_GetCPUInfo(void);
int IMP_System_Bind(IMPCell *srcCell, IMPCell *dstCell);
int IMP_System_UnBind(IMPCell *srcCell, IMPCell *dstCell);
int IMP_System_GetBindbyDest(IMPCell *dstCell, IMPCell *srcCell);
```

These 11 functions are present on all 8 SoCs with identical signatures.

#### Memory Pool Functions (Newer SoCs)

```c
// T23, T32, T40, T41:
int IMP_System_MemPoolRequest(int poolId, size_t size, char *name);
// Note: T23 and T32 declare the name parameter as `const char *name`,
// while T40 and T41 declare it as `char *name`

// T32 only:
int IMP_System_MemPoolFree(int poolId);
```

### HAL Implications

- The 11 core functions are stable across all SoCs. The HAL can call them unconditionally.
- **`IMP_System_MemPoolRequest`**: Available on T23, T32, T40, T41. Allocates physically contiguous memory from rmem. Must be called **before** `IMP_System_Init`. On T20, T21, T30, T31, the SDK manages memory pools internally.
- **`IMP_System_MemPoolFree`**: T32-only. Releases a previously allocated memory pool. On T32, the documentation warns: "forbidden to release when the code stream is running. It can be released only after all resources exit." On other SoCs with MemPoolRequest, pools are implicitly freed on `IMP_System_Exit`.
- The `const` qualifier difference on the `name` parameter is a minor API inconsistency. The HAL should use `const char *` and cast when calling on T40/T41.

---

## 2. IMPVersion Struct

Identical across all 8 SoCs:

```c
typedef struct {
    char aVersion[64];
} IMPVersion;
```

### HAL Implications

No abstraction needed. The HAL can use this struct directly. The version string is retrieved via `IMP_System_GetVersion()`.

---

## 3. IMPSensorInfo Struct

**This struct is NOT declared in any of the 8 `imp_system.h` headers analyzed.**

`IMPSensorInfo` is declared in `imp_isp.h`, not `imp_system.h`. The `imp_system.h` files across all 8 SoCs contain only the `IMPVersion` struct, the system function declarations, and documentation comments. No sensor-related structures (`IMPSensorInfo`, `IMPI2CInfo`, etc.) appear in any of these headers.

For the HAL, sensor information must be obtained from `imp_isp.h` which is a separate header file. This is documented here for completeness since the init sequence references ISP functions.

---

## 4. IMPI2CInfo Struct

**Not present in any `imp_system.h`.** This struct is declared in `imp_isp.h`.

---

## 5. Enums (IMPVI_NUM, IMPSensorVinType, IMPSensorMclk)

**None of these enums are present in any `imp_system.h`.** They are declared in `imp_isp.h` on the SoCs that support them.

---

## 6. Init/Destroy Sequence

The `imp_system.h` documentation across all SoCs describes the following system architecture and initialization requirements:

### System Architecture

The IMP system uses a pipeline model with these concepts:

1. **Device**: A functional unit (FrameSource, Encoder, Decoder, IVS, OSD)
2. **Group**: Input node of a Device. Each Group accepts one data input, can have multiple Outputs.
3. **Output**: Output node of a Group. Each Output produces one data stream.
4. **Cell**: A {deviceID, groupID, outputID} tuple identifying a pipeline node.
5. **Channel**: A functional unit registered to a Group (e.g., H264 encoder channel, JPEG encoder channel).

### Correct Initialization Order

Based on the `imp_system.h` documentation comments and the API contracts:

```
1. IMP_System_MemPoolRequest()    [T23/T32/T40/T41 only, BEFORE System_Init]
2. IMP_System_Init()              [Required first -- initializes data structures]
3. ISP Setup (via imp_isp.h):
   a. IMP_ISP_Open()
   b. IMP_ISP_AddSensor()
   c. IMP_ISP_EnableSensor()
   d. IMP_ISP_EnableTuning()
4. FrameSource Setup (via imp_framesource.h):
   a. IMP_FrameSource_CreateChn()
   b. IMP_FrameSource_SetChnAttr()
5. Pipeline Binding:
   a. IMP_System_Bind(&srcCell, &dstCell)
   b. ... (all bindings)
6. Encoder/OSD/IVS Channel Setup:
   a. Create channels
   b. Register channels to groups
7. IMP_FrameSource_EnableChn()    [Start streaming]
```

### Correct Teardown Order

```
1. IMP_FrameSource_DisableChn()   [Stop streaming]
2. Unregister and destroy channels
3. IMP_System_UnBind()            [All unbindings]
4. IMP_FrameSource_DestroyChn()
5. ISP Teardown:
   a. IMP_ISP_DisableTuning()
   b. IMP_ISP_DisableSensor()
   c. IMP_ISP_DelSensor()
   d. IMP_ISP_Close()
6. IMP_System_Exit()              [Releases all memory and handles]
7. IMP_System_MemPoolFree()       [T32 only, after System_Exit]
```

### Key Rules from Documentation

From the `imp_system.h` comments (present across all SoCs):

1. **`IMP_System_Init` must be called before any other IMP operation.** It initializes data structures but does not initialize hardware.
2. **All Bind operations should be performed during system initialization**, before enabling FrameSource.
3. **Bind and UnBind cannot be called dynamically** while FrameSource is enabled. FrameSource must be disabled first.
4. **DestroyGroup must be called after UnBind.**
5. **`IMP_System_Exit` releases all memory and handles**, closing hardware units. To use IMP again, full re-initialization is required.

### Binding Examples from Headers

The documentation provides these canonical binding patterns:

**Simple single-stream:**
```c
IMPCell fs_chn0 = {DEV_ID_FS, 0, 0};
IMPCell enc_grp0 = {DEV_ID_ENC, 0, 0};
IMP_System_Bind(&fs_chn0, &enc_grp0);
```

**Dual-stream with OSD and IVS:**
```c
// Main stream: FrameSource -> OSD -> Encoder
IMPCell fs_chn0 = {DEV_ID_FS, 0, 0};
IMPCell osd_grp0 = {DEV_ID_OSD, 0, 0};
IMPCell enc_grp0 = {DEV_ID_ENC, 0, 0};
IMP_System_Bind(&fs_chn0, &osd_grp0);
IMP_System_Bind(&osd_grp0, &enc_grp0);

// Sub stream: FrameSource -> IVS -> OSD -> Encoder
IMPCell fs_chn1 = {DEV_ID_FS, 1, 0};
IMPCell ivs_grp0 = {DEV_ID_IVS, 0, 0};
IMPCell osd_grp1 = {DEV_ID_OSD, 1, 0};
IMPCell enc_grp1 = {DEV_ID_ENC, 1, 0};
IMP_System_Bind(&fs_chn1, &ivs_grp0);
IMP_System_Bind(&ivs_grp0, &osd_grp1);
IMP_System_Bind(&osd_grp1, &enc_grp1);
```

Note: IVS is bound before OSD in the sub-stream because OSD timestamps can cause false positives in motion detection.

**Tree-structured binding (T23/T31/T32/T40/T41 documentation):**
```c
// FrameSource Channel1 with two outputs
IMPCell fs_chn1_output0 = {DEV_ID_FS, 1, 0};
IMPCell fs_chn1_output1 = {DEV_ID_FS, 1, 1};
IMPCell osd_grp1 = {DEV_ID_OSD, 1, 0};
IMPCell ivs_grp0 = {DEV_ID_IVS, 0, 0};
IMP_System_Bind(&fs_chn1_output0, &osd_grp1);
IMP_System_Bind(&fs_chn1_output1, &ivs_grp0);
```

This allows IVS and OSD to process in parallel from the same FrameSource channel.

### HAL Implications

- The HAL must enforce the init/teardown ordering. A state machine tracking the system lifecycle is recommended.
- Memory pool setup (on supported SoCs) must happen before `IMP_System_Init`.
- The HAL should abstract the ISP sensor setup since it requires `imp_isp.h` functions that are outside `imp_system.h` but are a mandatory part of the init sequence.
- Dynamic rebinding (changing pipeline topology at runtime) requires stopping the FrameSource first. The HAL should handle this transparently if stream reconfiguration is supported.

---

## 7. Documentation Language Differences

The headers contain identical APIs but differ in documentation language:

| SoC | Doc Language | Notes |
|-----|-------------|-------|
| T20 | Chinese (zh) | Oldest SDK, most mature documentation |
| T21 | Chinese (zh) | Same documentation as T20 |
| T23 | English (en) | Translation of Chinese docs, some binding example differences |
| T30 | Chinese (zh) | Same documentation as T20/T21 |
| T31 | English (en) | Longer documentation with more IVS/OSD detail |
| T32 | English (en) | Same as T31, adds MemPoolRequest/MemPoolFree |
| T40 | English (en) | Same as T32 minus MemPoolFree |
| T41 | English (en) | Same as T40 |

The English translations (T23, T31, T32, T40, T41) have slightly different binding example code compared to the Chinese originals (T20, T21, T30), but the API semantics are identical.

---

## Summary: SoC Groupings for imp_system.h

| Group | SoCs | System API Features |
|-------|------|---------------------|
| **Base** | T20, T21, T30 | 11 core functions only |
| **+MemPool** | T23, T40, T41 | 11 core + `MemPoolRequest` |
| **+MemPool+Free** | T32 | 11 core + `MemPoolRequest` + `MemPoolFree` |

The T31 header is identical to the Base group (T20/T21/T30) in terms of function declarations -- it has the 11 core functions and no memory pool functions.

### Critical Note on IMPSensorInfo

The `IMPSensorInfo` struct, `IMPI2CInfo` struct, and sensor-related enums (`IMPVI_NUM`, `IMPSensorVinType`, `IMPSensorMclk`) referenced in the task description are **not** in `imp_system.h` on any SoC. They reside in `imp_isp.h`. A separate analysis of `imp_isp.h` across all 8 SoCs is needed to document those structures, which vary dramatically between SoC generations (especially T40/T41 vs earlier SoCs).

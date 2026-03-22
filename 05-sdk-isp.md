# SDK ISP Module (`imp_isp.h`)

Cross-SoC API reference for the IMP ISP module across all 8 supported Ingenic SoCs. This is the second most complex module in the SDK.

---

## 1. Function Presence Matrix

Functions are grouped by category. Due to the sheer size of the ISP API, the matrix uses these conventions:
- **Y** = present with original signature (scalar values, no IMPVI_NUM)
- **P** = present with pointer arguments (no IMPVI_NUM)
- **V** = present with IMPVI_NUM + pointer arguments
- **-** = not present

### 1.1 Lifecycle

| Function | T20 | T21 | T23 | T30 | T31 | T32 | T40 | T41 |
|---|---|---|---|---|---|---|---|---|
| Open | Y | Y | Y | Y | Y | V | V | V |
| Close | Y | Y | Y | Y | Y | V | V | V |
| AddSensor | Y | Y | Y | Y | Y | V | V | V |
| DelSensor | Y | Y | Y | Y | Y | V | V | V |
| EnableSensor | Y | Y | Y | Y | Y | V | V | V |
| DisableSensor | Y | Y | Y | Y | Y | V | V | V |
| EnableTuning | Y | Y | Y | Y | Y | V | V | V |
| DisableTuning | Y | Y | Y | Y | Y | V | V | V |
| SetDefaultBinPath | - | - | - | - | - | V | V | V |
| GetDefaultBinPath | - | - | - | - | - | V | V | V |
| SetCameraInputMode | - | - | - | - | - | V | V | - |
| GetCameraInputMode | - | - | - | - | - | V | V | - |
| SetCameraInputSelect | - | - | - | - | - | - | V | - |

Note: On T32/T40/T41, `Open`/`Close`/`EnableTuning`/`DisableTuning` take `void` (no IMPVI_NUM), while `AddSensor`/`DelSensor`/`EnableSensor`/`DisableSensor` take `IMPVI_NUM` as first param.

### 1.2 Sensor Register Access

| Function | T20 | T21 | T23 | T30 | T31 | T32 | T40 | T41 |
|---|---|---|---|---|---|---|---|---|
| SetSensorRegister | Y | Y | Y | Y | Y | V | V | V |
| GetSensorRegister | Y | Y | Y | Y | Y | V | V | V |

Signature change:
- T20-T31: `(uint32_t reg, uint32_t value)` / `(uint32_t reg, uint32_t *value)`
- T40: `(IMPVI_NUM num, uint32_t *reg, uint32_t *value)` -- both reg and value are pointers
- T32/T41: `(IMPVI_NUM num, IMPISPSensorRegister *reg)` -- wrapped in a struct

### 1.3 Basic Tuning (Brightness, Contrast, Sharpness, Saturation, Hue)

| Function | T20 | T21 | T23 | T30 | T31 | T32 | T40 | T41 |
|---|---|---|---|---|---|---|---|---|
| SetBrightness | Y | Y | Y | Y | Y | V | V | V |
| GetBrightness | Y | Y | Y | Y | Y | V | V | V |
| SetContrast | Y | Y | Y | Y | Y | V | V | V |
| GetContrast | Y | Y | Y | Y | Y | V | V | V |
| SetSharpness | Y | Y | Y | Y | Y | V | V | V |
| GetSharpness | Y | Y | Y | Y | Y | V | V | V |
| SetSaturation | Y | Y | Y | Y | Y | V | V | V |
| GetSaturation | Y | Y | Y | Y | Y | V | V | V |
| SetBcshHue | - | - | Y | - | Y | V | V | V |
| GetBcshHue | - | - | Y | - | Y | V | V | V |

**Critical signature break** (see Section 2 for full detail):
- T20/T21/T30: `SetBrightness(unsigned char bright)` -- scalar, no IMPVI_NUM
- T23/T31: `SetBrightness(unsigned char bright)` -- still scalar, no IMPVI_NUM
- T40: `SetBrightness(IMPVI_NUM num, unsigned char *bright)` -- IMPVI_NUM + pointer
- T32/T41: `SetBrightness(IMPVI_NUM num, unsigned char *bright)` -- IMPVI_NUM + pointer

### 1.4 Flip / Mirror

| Function | T20 | T21 | T23 | T30 | T31 | T32 | T40 | T41 |
|---|---|---|---|---|---|---|---|---|
| SetISPHflip | Y | Y | Y | Y | Y | - | - | - |
| GetISPHflip | Y | Y | Y | Y | Y | - | - | - |
| SetISPVflip | Y | Y | Y | Y | Y | - | - | - |
| GetISPVflip | Y | Y | Y | Y | Y | - | - | - |
| SetISPHVflip | Y | - | - | Y | - | - | - | - |
| GetISPHVflip | Y | - | - | Y | - | - | - | - |
| SetHVFLIP | - | - | Y | - | Y | V | V | V |
| GetHVFlip | - | - | Y | - | Y | V | V | V |
| SetSensorHflip | - | - | Y | - | - | - | - | - |
| GetSensorHflip | - | - | Y | - | - | - | - | - |
| SetSensorVflip | - | - | Y | - | - | - | - | - |
| GetSensorVflip | - | - | Y | - | - | - | - | - |

Evolution:
- **T20/T30**: Separate `SetISPHflip`/`SetISPVflip` + combined `SetISPHVflip`
- **T21**: Separate `SetISPHflip`/`SetISPVflip` only (no combined)
- **T23/T31**: Combined `SetHVFLIP`/`GetHVFlip` replaces the separate H/V functions. T23 also has `SetSensorHflip`/`SetSensorVflip` for sensor-level flip
- **T32/T40/T41**: `SetHVFLIP(IMPVI_NUM, IMPISPHVFLIP*)` with IMPVI_NUM

### 1.5 Sensor FPS

| Function | T20 | T21 | T23 | T30 | T31 | T32 | T40 | T41 |
|---|---|---|---|---|---|---|---|---|
| SetSensorFPS | Y | Y | Y | Y | Y | V | V | V |
| GetSensorFPS | Y | Y | Y | Y | Y | V | V | V |

Signature change:
- T20-T31: `(uint32_t fps_num, uint32_t fps_den)` / `(uint32_t *fps_num, uint32_t *fps_den)`
- T40: `(IMPVI_NUM num, uint32_t *fps_num, uint32_t *fps_den)`
- T32/T41: `(IMPVI_NUM num, IMPISPSensorFps *fps)` -- wrapped in struct

### 1.6 Anti-Flicker

| Function | T20 | T21 | T23 | T30 | T31 | T32 | T40 | T41 |
|---|---|---|---|---|---|---|---|---|
| SetAntiFlickerAttr | Y | Y | Y | Y | Y | V | V | V |
| GetAntiFlickerAttr | Y | Y | Y | Y | Y | V | V | V |

Signature change:
- T20-T31: `(IMPISPAntiflickerAttr attr)` -- scalar by-value
- T40: `(IMPVI_NUM num, IMPISPAntiflickerAttr *pattr)` -- pointer
- T32/T41: `(IMPVI_NUM num, IMPISPAntiflickerAttr *pattr)` -- pointer

### 1.7 Exposure / AE

| Function | T20 | T21 | T23 | T30 | T31 | T32 | T40 | T41 |
|---|---|---|---|---|---|---|---|---|
| SetExpr | Y | Y | Y | Y | Y | - | - | - |
| GetExpr | Y | Y | Y | Y | Y | - | - | - |
| SetAeComp | Y | - | Y | Y | Y | - | - | - |
| GetAeComp | Y | - | Y | Y | Y | - | - | - |
| GetAeLuma | - | - | Y | - | Y | - | - | - |
| SetAeFreeze | - | - | Y | - | Y | - | - | - |
| SetAeStrategy | Y | Y | - | Y | - | - | - | - |
| GetAeStrategy | Y | Y | - | Y | - | - | - | - |
| GetTotalGain | Y | Y | Y | Y | Y | - | - | - |
| SetIntegrationTime | Y | - | - | Y | - | - | - | - |
| GetIntegrationTime | Y | - | - | Y | - | - | - | - |
| AE_GetROI | Y | Y | Y | Y | Y | - | - | - |
| AE_SetROI | Y | Y | Y | Y | Y | - | - | - |
| SetAeWeight | Y | Y | Y | Y | Y | V | V | V |
| GetAeWeight | Y | Y | Y | Y | Y | V | V | V |
| SetAeHist | Y | Y | Y | Y | Y | - | - | - |
| GetAeHist | Y | Y | Y | Y | Y | - | - | - |
| GetAeHist_Origin | - | - | Y | - | Y | - | - | - |
| GetAeZone | Y | Y | Y | - | Y | - | - | - |
| SetAeMin | - | Y | Y | - | Y | - | - | - |
| GetAeMin | - | Y | Y | - | Y | - | - | - |
| SetAe_IT_MAX | - | - | Y | - | Y | - | - | - |
| GetAE_IT_MAX | - | - | Y | - | Y | - | - | - |
| SetAeTargetList | - | - | Y | - | Y | - | - | - |
| GetAeTargetList | - | - | Y | - | Y | - | - | - |
| SetAeAttr | - | - | Y | - | Y | - | - | - |
| GetAeAttr | - | - | Y | - | Y | - | - | - |
| GetAeState | - | - | Y | - | Y | - | - | - |
| SetAeExprInfo | - | - | - | - | - | V | V | V |
| GetAeExprInfo | - | - | - | - | - | V | V | V |
| SetAeScenceAttr | - | - | - | - | - | - | V | V |
| GetAeScenceAttr | - | - | - | - | - | - | V | V |
| GetAeStatistics | - | - | - | - | - | V | V | V |
| SetAeExpList | - | - | - | - | - | V | V | V |
| GetAeExpList | - | - | - | - | - | V | V | V |
| SetAeSpeed | - | - | - | - | - | - | V | - |
| GetAeSpeed | - | - | - | - | - | - | V | - |
| SetAeTargetAttr | - | - | - | - | - | V | - | - |
| GetAeTargetAttr | - | - | - | - | - | V | - | - |
| SetAeStartAttr | - | - | - | - | - | V | - | - |
| GetAeStartAttr | - | - | - | - | - | V | - | - |
| SetAeLightCompensation | - | - | - | - | - | V | - | - |
| GetAeLightCompensation | - | - | - | - | - | V | - | - |
| GetAeOnlyReadAttr | - | - | - | - | - | V | - | - |
| GetAeAtList | - | - | - | - | - | - | - | V |
| GetAEEvList | - | - | - | - | - | - | - | V |
| GetAEFlickerFlag | - | - | - | - | - | V | - | V |

### 1.8 White Balance / AWB

| Function | T20 | T21 | T23 | T30 | T31 | T32 | T40 | T41 |
|---|---|---|---|---|---|---|---|---|
| SetWB | Y | Y | Y | Y | Y | - | - | - |
| GetWB | Y | Y | Y | Y | Y | - | - | - |
| GetWB_Statis | Y | Y | Y | Y | Y | - | - | - |
| GetWB_GOL_Statis | - | Y | Y | - | Y | - | - | - |
| Awb_GetCwfShift | Y | - | - | Y | - | - | - | - |
| Awb_SetCwfShift | Y | - | - | Y | - | - | - | - |
| Awb_GetRgbCoefft | Y | Y | Y | Y | Y | V | V | V |
| Awb_SetRgbCoefft | Y | Y | Y | Y | Y | V | V | V |
| SetAwbWeight | Y | Y | Y | Y | Y | V | V | V |
| GetAwbWeight | Y | Y | Y | Y | Y | V | V | V |
| SetAwbClust | - | - | Y | - | Y | - | - | - |
| GetAwbClust | - | - | Y | - | Y | - | - | - |
| SetAwbCtTrend | - | - | Y | - | Y | - | - | - |
| GetAwbCtTrend | - | - | Y | - | Y | - | - | - |
| SetAwbZoneWeight | - | - | Y | - | - | - | - | - |
| GetAwbZoneWeight | - | - | Y | - | - | - | - | - |
| GetAwbZone | - | - | Y | - | Y | - | - | - |
| SetWB_ALGO | - | - | Y | - | Y | - | - | - |
| SetAwbCt | - | - | Y | - | Y | - | - | - |
| GetAWBCt | - | - | Y | - | Y | - | - | - |
| SetAwbAttr | - | - | - | - | - | V | V | V |
| GetAwbAttr | - | - | - | - | - | V | V | V |
| GetAwbStatistics | - | - | - | - | - | V | V | V |
| GetAwbGlobalStatistics | - | - | - | - | - | - | V | V |
| SetAwbCtTrendOffset | - | - | - | - | - | V | - | - |
| GetAwbCtTrendOffset | - | - | - | - | - | V | - | - |
| GetAwbOnlyReadAttr | - | - | - | - | - | V | - | - |

### 1.9 Denoising (Sinter, Temper, DPC)

| Function | T20 | T21 | T23 | T30 | T31 | T32 | T40 | T41 |
|---|---|---|---|---|---|---|---|---|
| SetSinterDnsAttr | Y | Y | - | Y | - | - | - | - |
| GetSinterDnsAttr | Y | Y | - | Y | - | - | - | - |
| SetSinterStrength | Y | Y | Y | Y | Y | - | - | - |
| SetTemperDnsCtl | Y | Y | - | Y | - | - | - | - |
| SetTemperDnsAttr | Y | Y | - | Y | - | - | - | - |
| GetTemperDnsAttr | Y | Y | - | Y | - | - | - | - |
| SetTemperStrength | Y | Y | Y | Y | Y | - | - | - |
| SetDPStrength | Y | Y | - | Y | - | - | - | - |
| SetDPC_Strength | - | - | Y | - | Y | - | - | - |
| GetDPC_Strength | - | - | Y | - | Y | - | - | - |

On T20/T21/T30, sinter and temper denoising have individual attribute structs (`IMPISPSinterDenoiseAttr`, `IMPISPTemperDenoiseAttr`). Starting with T23/T31, these are replaced by `SetModuleControl` with ratio-based strength (via `SetModule_Ratio` on T32+). T32/T40/T41 manage denoising entirely through module control -- no individual sinter/temper APIs.

### 1.10 DRC / Defog / Backlight / Highlight

| Function | T20 | T21 | T23 | T30 | T31 | T32 | T40 | T41 |
|---|---|---|---|---|---|---|---|---|
| SetRawDRC | Y | Y | - | Y | - | - | - | - |
| GetRawDRC | Y | Y | - | Y | - | - | - | - |
| SetDRC_Strength | - | Y | Y | - | Y | - | - | - |
| GetDRC_Strength | - | Y | Y | - | Y | - | - | - |
| EnableDRC | - | - | Y | - | Y | - | - | - |
| EnableDefog | - | - | Y | - | Y | - | - | - |
| SetAntiFogAttr | Y | Y | - | Y | - | - | - | - |
| SetHiLightDepress | Y | Y | Y | Y | Y | - | - | - |
| GetHiLightDepress | - | Y | Y | - | Y | - | - | - |
| SetBacklightComp | - | - | Y | - | Y | - | - | - |
| GetBacklightComp | - | - | Y | - | Y | - | - | - |

### 1.11 Gamma / CCM / Module Control

| Function | T20 | T21 | T23 | T30 | T31 | T32 | T40 | T41 |
|---|---|---|---|---|---|---|---|---|
| SetGamma | Y | Y | Y | Y | Y | - | - | - |
| GetGamma | Y | Y | Y | Y | Y | - | - | - |
| SetGammaAttr | - | - | - | - | - | V | V | V |
| GetGammaAttr | - | - | - | - | - | V | V | V |
| SetCCMAttr | - | - | Y | - | Y | V | V | V |
| GetCCMAttr | - | - | Y | - | Y | V | V | V |
| SetModuleControl | - | Y | Y | - | Y | V | V | V |
| GetModuleControl | - | Y | Y | - | Y | V | V | V |
| SetModule_Ratio | - | - | - | - | - | V | V | V |
| GetModule_Ratio | - | - | - | - | - | V | V | V |
| SetISPCSCAttr | - | - | - | - | - | V | V | V |
| GetISPCSCAttr | - | - | - | - | - | V | V | V |

### 1.12 Running Mode / WDR / Bypass

| Function | T20 | T21 | T23 | T30 | T31 | T32 | T40 | T41 |
|---|---|---|---|---|---|---|---|---|
| SetISPRunningMode | Y | Y | Y | Y | Y | V | V | V |
| GetISPRunningMode | Y | Y | Y | Y | Y | V | V | V |
| SetISPCustomMode | - | - | Y | - | Y | - | - | - |
| GetISPCustomMode | - | - | Y | - | Y | - | - | - |
| SetISPBypass | Y | Y | Y | Y | Y | V | V | V |
| GetISPBypass | - | - | - | - | - | V | V | V |
| SetWDRAttr | Y | - | - | Y | - | - | - | - |
| GetWDRAttr | Y | - | - | Y | - | - | - | - |
| WDR_ENABLE | - | - | - | - | Y | V | V | V |
| WDR_ENABLE_Get / GET | - | - | - | - | Y | V | V | V |
| WDR_OPEN | - | - | - | - | - | - | V | V |
| SetWdrOutputMode | - | - | - | - | - | V | V | V |
| GetWdrOutputMode | - | - | - | - | - | V | V | V |

### 1.13 AF (Autofocus) Statistics

| Function | T20 | T21 | T23 | T30 | T31 | T32 | T40 | T41 |
|---|---|---|---|---|---|---|---|---|
| GetAfHist | Y | Y | Y | Y | Y | - | - | - |
| SetAfHist | Y | Y | Y | Y | Y | - | - | - |
| SetAfWeight | - | Y | Y | - | Y | V | V | V |
| GetAfWeight | - | Y | Y | - | Y | V | V | V |
| GetAfZone | - | - | Y | - | Y | - | - | - |
| GetAFMetrices | - | - | Y | - | Y | - | - | - |
| GetAfStatistics | - | - | - | - | - | V | V | V |
| GetAFMetricesInfo | - | - | - | - | - | - | V | V |

### 1.14 AWB Statistics / Histogram

| Function | T20 | T21 | T23 | T30 | T31 | T32 | T40 | T41 |
|---|---|---|---|---|---|---|---|---|
| GetAwbHist | Y | Y | Y | Y | Y | - | - | - |
| SetAwbHist | Y | Y | Y | Y | Y | - | - | - |

### 1.15 Misc Tuning

| Function | T20 | T21 | T23 | T30 | T31 | T32 | T40 | T41 |
|---|---|---|---|---|---|---|---|---|
| SetISPProcess | Y | Y | - | Y | - | - | - | - |
| SetFWFreeze | Y | Y | - | Y | - | - | - | - |
| SaveAllParam | Y | Y | - | Y | - | - | - | - |
| SetVideoDrop | Y | Y | Y | Y | Y | V | V | V |
| GetEVAttr | Y | Y | Y | Y | Y | - | - | - |
| EnableMovestate | Y | Y | Y | Y | Y | - | - | - |
| DisableMovestate | Y | Y | Y | Y | Y | - | - | - |
| SetMaxAgain | Y | Y | Y | Y | Y | - | - | - |
| GetMaxAgain | Y | Y | Y | Y | Y | - | - | - |
| SetMaxDgain | Y | Y | Y | Y | Y | - | - | - |
| GetMaxDgain | Y | Y | Y | Y | Y | - | - | - |
| WaitFrame / WaitFrameDone | Y | Y | Y | Y | Y | V | V | V |
| SetSceneMode | Y | Y | - | Y | - | - | - | - |
| GetSceneMode | Y | Y | - | Y | - | - | - | - |
| SetColorfxMode | Y | Y | - | Y | - | - | - | - |
| GetColorfxMode | Y | Y | - | Y | - | - | - | - |
| SetISPLDC | - | - | - | Y | - | - | - | - |
| SetMeshShadingScale | Y | - | - | - | - | - | - | - |

### 1.16 Mask / Crop / Scaler

| Function | T20 | T21 | T23 | T30 | T31 | T32 | T40 | T41 |
|---|---|---|---|---|---|---|---|---|
| SetMask | - | - | Y | - | Y | - | V | - |
| GetMask | - | - | Y | - | Y | - | V | - |
| SetMaskBlock | - | - | - | - | - | V | - | V |
| GetMaskBlock | - | - | - | - | - | V | - | V |
| SetFrontCrop | - | - | Y | - | Y | - | - | - |
| GetFrontCrop | - | - | Y | - | Y | - | - | - |
| GetSensorAttr | - | - | Y | - | Y | V | V | V |
| SetScalerLv | - | - | Y | - | Y | V | V | V |
| GetBlcAttr | - | - | Y | - | Y | - | - | - |

### 1.17 Auto Zoom

| Function | T20 | T21 | T23 | T30 | T31 | T32 | T40 | T41 |
|---|---|---|---|---|---|---|---|---|
| SetAutoZoom | - | - | Y | - | Y | V | V | V |
| GetAutoZoom | - | - | Y | - | - | V | V | V |

### 1.18 Face AE / Face AWB (T40/T41)

| Function | T20 | T21 | T23 | T30 | T31 | T32 | T40 | T41 |
|---|---|---|---|---|---|---|---|---|
| GetFaceAe | - | - | - | - | - | - | V | V |
| SetFaceAe | - | - | - | - | - | - | V | - |
| GetFaceAwb | - | - | - | - | - | - | V | - |
| SetFaceAwb | - | - | - | - | - | - | V | - |
| SetFaceAeWeiget | - | - | - | - | - | V | - | V |
| GetFaceAeWeiget | - | - | - | - | - | V | - | V |
| GetFaceAeLuma | - | - | - | - | - | V | - | V |
| SetTmoFaceae | - | - | - | - | - | V | - | V |

### 1.19 ISP OSD Regions (in imp_isp.h)

| Function | T20 | T21 | T23 | T30 | T31 | T32 | T40 | T41 |
|---|---|---|---|---|---|---|---|---|
| SetOsdPoolSize | - | - | Y | - | - | Y | Y | Y |
| CreateOsdRgn | - | - | Y | - | - | Y | Y | Y |
| SetOsdRgnAttr | - | - | Y | - | - | Y | Y | Y |
| GetOsdRgnAttr | - | - | Y | - | - | Y | Y | Y |
| ShowOsdRgn | - | - | Y | - | - | Y | Y | Y |
| DestroyOsdRgn | - | - | Y | - | - | Y | Y | Y |

These are declared in `imp_isp.h` (as `IMP_ISP_Tuning_*OsdRgn*`) and use the `IMPIspOsdAttrAsm` type. Note: The standalone `isp_osd.h` file (see Section 5) provides wrapper functions with different names.

### 1.20 OSD/Draw Attr (T32/T40/T41)

| Function | T20 | T21 | T23 | T30 | T31 | T32 | T40 | T41 |
|---|---|---|---|---|---|---|---|---|
| GetOSDAttr | - | - | - | - | - | V | V | V |
| SetOSDAttr | - | - | - | - | - | V | V | V |
| SetOSDBlock | - | - | - | - | - | V | - | V |
| GetOSDBlock | - | - | - | - | - | V | - | V |
| GetDrawAttr | - | - | - | - | - | - | V | - |
| SetDrawAttr | - | - | - | - | - | - | V | - |
| GetDrawBlock | - | - | - | - | - | V | - | V |
| SetDrawBlock | - | - | - | - | - | V | - | V |

### 1.21 Algorithm Function Injection (T40/T41)

| Function | T20 | T21 | T23 | T30 | T31 | T32 | T40 | T41 |
|---|---|---|---|---|---|---|---|---|
| SetAeAlgoFunc | - | - | - | - | - | - | V | V |
| SetAwbAlgoFunc | - | - | - | - | - | - | V | V |

### 1.22 Bin File / Night Mode / Frame Drop

| Function | T20 | T21 | T23 | T30 | T31 | T32 | T40 | T41 |
|---|---|---|---|---|---|---|---|---|
| SwitchBin | - | - | - | - | - | V | V | V |
| StartNightMode | - | - | - | - | - | V | V | V |
| SetFrameDrop | - | - | - | - | - | V | V | V |
| GetFrameDrop | - | - | - | - | - | V | V | V |

### 1.23 PM / LDC / HLDC (T32+ Only)

| Function | T20 | T21 | T23 | T30 | T31 | T32 | T40 | T41 |
|---|---|---|---|---|---|---|---|---|
| LDC_INIT | - | - | - | - | - | V | - | V |
| LDC_GetAttr | - | - | - | - | - | V | - | V |
| LDC_SetAttr | - | - | - | - | - | V | - | V |
| PM_SetMode | - | - | - | - | - | V | - | - |
| PM_GetMode | - | - | - | - | - | V | - | - |
| PM_WaitAllDone | - | - | - | - | - | V | - | - |
| HB_GetAttr | - | - | - | - | - | V | - | - |
| HB_SetAttr | - | - | - | - | - | V | - | - |
| SetHLDCAttr | - | - | - | - | - | - | V | - |
| GetHLDCAttr | - | - | - | - | - | - | V | - |
| SetFixedContraster | - | - | - | - | - | - | V | - |
| Bypass_Bind | - | - | - | - | - | - | V | - |
| SetInternalChnAttr | - | - | - | - | - | V | - | V |
| GetInternalChnAttr | - | - | - | - | - | V | - | V |
| SetPreDqtime | - | - | - | - | - | - | V | - |
| GetRaw | - | - | - | - | - | - | V | - |
| RAW_RwControl | - | - | - | - | - | - | - | V |
| SetCsccrMode | - | - | - | - | - | V | - | V |
| GetCsccrMode | - | - | - | - | - | V | - | V |
| SetVicDoneCbFunc | - | - | - | - | - | - | - | V |
| SetTmoCurve | - | - | - | - | - | - | - | V |
| GetTmoCurve | - | - | - | - | - | - | - | V |

### 1.24 T23 Dual-Sensor: `_Sec` Suffix Functions

T23 provides a complete parallel API for a secondary sensor using `_Sec` suffix functions. Every primary tuning function has a `_Sec` counterpart with the same signature. Examples:

```c
int IMP_ISP_Tuning_SetBrightness_Sec(unsigned char bright);
int IMP_ISP_Tuning_GetBrightness_Sec(unsigned char *pbright);
int IMP_ISP_Tuning_SetSensorFPS_Sec(uint32_t fps_num, uint32_t fps_den);
// ... etc for all tuning functions
```

T23 also provides `IMP_ISP_MultiCamera_*` functions that take `IMPVI_NUM` as the first parameter, duplicating the same tuning functions with multi-camera awareness. The `_Sec` functions are essentially `MultiCamera` with `IMPVI_NUM = IMPVI_SEC` hardcoded.

### 1.25 T23 `IMP_ISP_MultiCamera_*` Functions

T23 has ~150 `IMP_ISP_MultiCamera_*` functions that wrap tuning functions with `IMPVI_NUM num` as the first parameter, using the same parameter types as the primary functions (scalar values, not pointers). For example:

```c
int IMP_ISP_MultiCamera_Tuning_SetBrightness(IMPVI_NUM num, unsigned char bright);
int IMP_ISP_MultiCamera_Tuning_GetBrightness(IMPVI_NUM num, unsigned char *pbright);
```

This is distinct from the T40/T41 approach where the primary functions themselves take `IMPVI_NUM`.

---

## 2. Signature Differences -- The Critical API Break

The most important cross-SoC difference in the ISP module is the three-generation signature evolution for tuning functions.

### Generation 1: T20, T21, T30 (Scalar, No IMPVI_NUM)

```c
int IMP_ISP_Tuning_SetBrightness(unsigned char bright);
int IMP_ISP_Tuning_GetBrightness(unsigned char *pbright);
int IMP_ISP_Tuning_SetSensorFPS(uint32_t fps_num, uint32_t fps_den);
int IMP_ISP_Tuning_GetSensorFPS(uint32_t *fps_num, uint32_t *fps_den);
int IMP_ISP_Tuning_SetAntiFlickerAttr(IMPISPAntiflickerAttr attr);  /* by-value */
int IMP_ISP_Tuning_SetISPRunningMode(IMPISPRunningMode mode);       /* by-value */
```

Set functions take scalar values. Get functions return via pointer. Enum/struct args passed by value on Set.

### Generation 2: T23, T31 (Scalar, No IMPVI_NUM, but some pointer changes)

```c
int IMP_ISP_Tuning_SetBrightness(unsigned char bright);       /* still scalar */
int IMP_ISP_Tuning_GetBrightness(unsigned char *pbright);
int IMP_ISP_Tuning_SetHVFLIP(IMPISPHVFLIP hvflip);            /* replaces separate H/V */
int IMP_ISP_Tuning_GetHVFlip(IMPISPHVFLIP *hvflip);
```

Primary API is still scalar. New functions like `SetHVFLIP` are introduced. `_Sec` and `MultiCamera` variants added on T23 for dual-sensor.

### Generation 3: T40 (IMPVI_NUM + Pointer)

```c
int32_t IMP_ISP_Tuning_SetBrightness(IMPVI_NUM num, unsigned char *bright);
int32_t IMP_ISP_Tuning_GetBrightness(IMPVI_NUM num, unsigned char *pbright);
int32_t IMP_ISP_Tuning_SetSensorFPS(IMPVI_NUM num, uint32_t *fps_num, uint32_t *fps_den);
int32_t IMP_ISP_Tuning_GetSensorFPS(IMPVI_NUM num, uint32_t *fps_num, uint32_t *fps_den);
int32_t IMP_ISP_Tuning_SetAntiFlickerAttr(IMPVI_NUM num, IMPISPAntiflickerAttr *pattr);
int32_t IMP_ISP_Tuning_SetISPRunningMode(IMPVI_NUM num, IMPISPRunningMode *mode);
int32_t IMP_ISP_Tuning_SetHVFLIP(IMPVI_NUM num, IMPISPHVFLIP *hvflip);
```

**All** tuning functions gain `IMPVI_NUM` as first parameter. **All** value parameters become pointers. Return type changes from `int` to `int32_t`.

### Generation 3b: T32, T41 (IMPVI_NUM + Pointer + Struct Wrapping)

Similar to T40 but some parameters are wrapped in structs:

```c
int32_t IMP_ISP_Tuning_SetSensorFPS(IMPVI_NUM num, IMPISPSensorFps *fps);    /* struct, not two pointers */
int32_t IMP_ISP_SetSensorRegister(IMPVI_NUM num, IMPISPSensorRegister *reg);  /* struct wrapper */
int32_t IMP_ISP_Tuning_SetHVFLIP(IMPVI_NUM num, IMPISPHVFLIPAttr *attr);     /* different type name */
int32_t IMP_ISP_SetISPBypass(IMPVI_NUM num, IMPISPOpsMode *enable);           /* T32: IMPISPOpsMode, not IMPISPTuningOpsMode */
```

T32 and T41 are very similar to each other but differ from T40 in struct type names and wrapping conventions.

### Functions That Change Signature Across All Three Generations

This pattern applies to essentially every `IMP_ISP_Tuning_*` function. Here is a representative list:

| Function | Gen1 (T20/T21/T30) | Gen2 (T23/T31) | Gen3 (T40) | Gen3b (T32/T41) |
|---|---|---|---|---|
| SetBrightness | `(uchar)` | `(uchar)` | `(VI, uchar*)` | `(VI, uchar*)` |
| GetBrightness | `(uchar*)` | `(uchar*)` | `(VI, uchar*)` | `(VI, uchar*)` |
| SetSensorFPS | `(u32, u32)` | `(u32, u32)` | `(VI, u32*, u32*)` | `(VI, FpsStruct*)` |
| SetAntiFlicker | `(enum)` | `(enum)` | `(VI, enum*)` | `(VI, enum*)` |
| SetRunningMode | `(enum)` | `(enum)` | `(VI, enum*)` | `(VI, enum*)` |
| SetHVFLIP | N/A | `(enum)` | `(VI, enum*)` | `(VI, struct*)` |
| SetISPBypass | `(enum)` | `(enum)` | `(VI, enum*)` | `(VI, OpsMode*)` |

---

## 3. IMPSensorInfo Struct Per SoC

### T20 / T21 / T30

```c
typedef struct {
    char name[32];
    IMPSensorControlBusType cbus_type;
    union {
        IMPI2CInfo i2c;
        IMPSPIInfo spi;
    };
    unsigned short rst_gpio;     /* currently unused */
    unsigned short pwdn_gpio;    /* currently unused */
    unsigned short power_gpio;   /* currently unused */
} IMPSensorInfo;
```

### T23

```c
typedef struct {
    char name[32];
    uint16_t sensor_id;                     /* NEW */
    IMPSensorControlBusType cbus_type;
    union {
        IMPI2CInfo i2c;
        IMPSPIInfo spi;
    };
    unsigned short rst_gpio;
    unsigned short pwdn_gpio;
    unsigned short power_gpio;
} IMPSensorInfo;
```

`sensor_id` is added but field order differs -- it is placed after `name` and before `cbus_type`.

### T31

```c
typedef struct {
    char name[32];
    IMPSensorControlBusType cbus_type;
    union {
        IMPI2CInfo i2c;
        IMPSPIInfo spi;
    };
    unsigned short rst_gpio;
    unsigned short pwdn_gpio;
    unsigned short power_gpio;
} IMPSensorInfo;
```

Same as T20 (no sensor_id).

### T40

```c
typedef struct {
    char name[32];
    IMPSensorControlBusType cbus_type;
    union {
        IMPI2CInfo i2c;
        IMPSPIInfo spi;
    };
    int rst_gpio;                           /* int, not unsigned short */
    int pwdn_gpio;                          /* int, not unsigned short */
    int power_gpio;                         /* int, not unsigned short */
    unsigned short sensor_id;
    IMPSensorVinType video_interface;       /* NEW: MIPI CSI0/CSI1/DVP */
    IMPSensorMclk mclk;                     /* NEW: MCLK0/MCLK1/MCLK2 */
    int default_boot;                       /* NEW: default boot setting */
} IMPSensorInfo;
```

Significant additions: `video_interface`, `mclk`, `default_boot`. GPIO fields widened from `unsigned short` to `int`. `sensor_id` moved to after GPIOs.

### T32 / T41

Same layout as T40.

---

## 4. Key Enums Per SoC

### IMPVI_NUM

| SoC | Values |
|---|---|
| T20/T21/T30/T31 | Not defined (single sensor only) |
| T23 | `IMPVI_MAIN=0, IMPVI_SEC=1, IMPVI_THR=2, IMPVI_BUTT` |
| T40 | `IMPVI_MAIN=0, IMPVI_SEC=1, IMPVI_THR=2, IMPVI_BUTT` |
| T41 | `IMPVI_MAIN=0, IMPVI_SEC=1 (not supported), IMPVI_THR=2 (not supported), IMPVI_BUTT` |
| T32 | `IMPVI_MAIN=0, IMPVI_SEC=1, IMPVI_THR=2, IMPVI_FOUR=3, IMPVI_BUTT` |

T32 supports up to 4 sensors. T41 defines IMPVI_SEC/THR but marks them as "not supported". T23 defines the enum for `MultiCamera_*` functions but the primary API does not use it.

### IMPISPRunningMode

All SoCs define:
```c
typedef enum {
    IMPISP_RUNNING_MODE_DAY = 0,
    IMPISP_RUNNING_MODE_NIGHT = 1,
    IMPISP_RUNNING_MODE_BUTT,
} IMPISPRunningMode;
```

### IMPISPAntiflickerAttr

All SoCs define:
```c
typedef enum {
    IMPISP_ANTIFLICKER_DISABLE,
    IMPISP_ANTIFLICKER_50HZ,
    IMPISP_ANTIFLICKER_60HZ,
    IMPISP_ANTIFLICKER_BUTT,
} IMPISPAntiflickerAttr;
```

### IMPISPTuningOpsMode

All SoCs define:
```c
typedef enum {
    IMPISP_TUNING_OPS_MODE_DISABLE,
    IMPISP_TUNING_OPS_MODE_ENABLE,
    IMPISP_TUNING_OPS_MODE_BUTT,
} IMPISPTuningOpsMode;
```

T32/T41 also define `IMPISPOpsMode` separately (same values, different type name used for some functions like `SetISPBypass`).

### IMPISPHVFLIP (T23/T31/T32/T40/T41)

```c
typedef enum {
    IMPISP_FLIP_NORMAL_MODE = 0,
    IMPISP_FLIP_H_MODE,       /* horizontal flip */
    IMPISP_FLIP_V_MODE,       /* vertical flip */
    IMPISP_FLIP_HV_MODE,      /* both */
    IMPISP_FLIP_BUTT,
} IMPISPHVFLIP;
```

T32/T41 wrap this in `IMPISPHVFLIPAttr` struct with additional fields.

### IMPSensorVinType (T40/T41/T32)

```c
typedef enum {
    IMPISP_SENSOR_VI_MIPI_CSI0 = 0,
    IMPISP_SENSOR_VI_MIPI_CSI1 = 1,
    IMPISP_SENSOR_VI_DVP = 2,
    IMPISP_SENSOR_VI_BUTT = 3,
} IMPSensorVinType;
```

### IMPSensorMclk (T40/T41/T32)

```c
typedef enum {
    IMPISP_SENSOR_MCLK0 = 0,
    IMPISP_SENSOR_MCLK1 = 1,
    IMPISP_SENSOR_MCLK2 = 2,
    IMPISP_SENSOR_MCLK_BUTT = 3,
} IMPSensorMclk;
```

### IMPISPTuningOpsType (T40/T41/T32)

```c
typedef enum {
    IMPISP_TUNING_OPS_TYPE_AUTO,
    IMPISP_TUNING_OPS_TYPE_MANUAL,
    IMPISP_TUNING_OPS_TYPE_BUTT,
} IMPISPTuningOpsType;
```

Not present in earlier SoCs. Used in AE/AWB attribute structs to select auto vs manual mode.

---

## 5. isp_osd.h -- Standalone ISP OSD Header

Present on: **T23, T32, T40, T41** (not on T20, T21, T30, T31).

The file is identical across all four SoCs. It provides wrapper functions around the ISP OSD region management:

```c
int IMP_OSD_Init_ISP(void);
int IMP_OSD_SetPoolSize_ISP(int size);
int IMP_OSD_CreateRgn_ISP(int chn, IMPIspOsdAttrAsm *pIspOsdAttr);
int IMP_OSD_SetRgnAttr_PicISP(int chn, int handle, IMPIspOsdAttrAsm *pIspOsdAttr);
int IMP_OSD_GetRgnAttr_ISPPic(int chn, int handle, IMPIspOsdAttrAsm *pIspOsdAttr);
int IMP_OSD_ShowRgn_ISP(int chn, int handle, int showFlag);
int IMP_OSD_DestroyRgn_ISP(int chn, int handle);
void IMP_OSD_Exit_ISP(void);
```

These functions use the `IMP_OSD_*_ISP` naming convention (different from `IMP_ISP_Tuning_*OsdRgn*` in `imp_isp.h`). The `IMP_OSD_Init_ISP` is auto-called by `IMP_System_Init`. The `chn` parameter refers to the sensor channel number.

The `IMPIspOsdAttrAsm` type is defined in `imp_osd.h` (the main OSD header), not in `isp_osd.h`. The ISP OSD allows overlaying text/images directly in the ISP pipeline before encoding, which is more efficient than the traditional OSD module for some use cases.

---

## 6. HAL Implications

### hal_isp_set_brightness() Dispatch

```c
int hal_isp_set_brightness(int vi_num, unsigned char brightness) {
#if defined(SOC_T20) || defined(SOC_T21) || defined(SOC_T30)
    // Gen1: scalar, no IMPVI_NUM
    return IMP_ISP_Tuning_SetBrightness(brightness);

#elif defined(SOC_T23) || defined(SOC_T31)
    // Gen2: scalar, no IMPVI_NUM (ignore vi_num, always main)
    (void)vi_num;
    return IMP_ISP_Tuning_SetBrightness(brightness);

#elif defined(SOC_T40)
    // Gen3: IMPVI_NUM + pointer
    return IMP_ISP_Tuning_SetBrightness((IMPVI_NUM)vi_num, &brightness);

#elif defined(SOC_T32) || defined(SOC_T41)
    // Gen3b: IMPVI_NUM + pointer (same as T40 for basic tuning)
    return IMP_ISP_Tuning_SetBrightness((IMPVI_NUM)vi_num, &brightness);
#endif
}
```

This three-way dispatch pattern applies to **every** basic tuning function (contrast, saturation, sharpness, hue).

### hal_isp_set_flip() Dispatch

```c
int hal_isp_set_flip(int vi_num, int hflip, int vflip) {
#if defined(SOC_T20) || defined(SOC_T30)
    // Has SetISPHflip + SetISPVflip + SetISPHVflip
    IMPISPTuningOpsMode h = hflip ? IMPISP_TUNING_OPS_MODE_ENABLE : IMPISP_TUNING_OPS_MODE_DISABLE;
    IMPISPTuningOpsMode v = vflip ? IMPISP_TUNING_OPS_MODE_ENABLE : IMPISP_TUNING_OPS_MODE_DISABLE;
    return IMP_ISP_Tuning_SetISPHVflip(h, v);

#elif defined(SOC_T21)
    // Has SetISPHflip + SetISPVflip but NO combined SetISPHVflip
    int ret;
    ret = IMP_ISP_Tuning_SetISPHflip(hflip ? IMPISP_TUNING_OPS_MODE_ENABLE : IMPISP_TUNING_OPS_MODE_DISABLE);
    if (ret) return ret;
    return IMP_ISP_Tuning_SetISPVflip(vflip ? IMPISP_TUNING_OPS_MODE_ENABLE : IMPISP_TUNING_OPS_MODE_DISABLE);

#elif defined(SOC_T23) || defined(SOC_T31)
    // SetHVFLIP with IMPISPHVFLIP enum
    IMPISPHVFLIP mode;
    if (hflip && vflip) mode = IMPISP_FLIP_HV_MODE;
    else if (hflip) mode = IMPISP_FLIP_H_MODE;
    else if (vflip) mode = IMPISP_FLIP_V_MODE;
    else mode = IMPISP_FLIP_NORMAL_MODE;
    return IMP_ISP_Tuning_SetHVFLIP(mode);

#elif defined(SOC_T40)
    // IMPVI_NUM + IMPISPHVFLIP pointer
    IMPISPHVFLIP mode = ...;  // same logic
    return IMP_ISP_Tuning_SetHVFLIP((IMPVI_NUM)vi_num, &mode);

#elif defined(SOC_T32) || defined(SOC_T41)
    // IMPVI_NUM + IMPISPHVFLIPAttr struct pointer
    IMPISPHVFLIPAttr attr = { .sensor_mode = ..., .isp_mode = ... };
    return IMP_ISP_Tuning_SetHVFLIP((IMPVI_NUM)vi_num, &attr);
#endif
}
```

### hal_isp_set_fps() Dispatch

```c
int hal_isp_set_fps(int vi_num, uint32_t fps_num, uint32_t fps_den) {
#if defined(SOC_T20) || defined(SOC_T21) || defined(SOC_T23) || defined(SOC_T30) || defined(SOC_T31)
    return IMP_ISP_Tuning_SetSensorFPS(fps_num, fps_den);

#elif defined(SOC_T40)
    return IMP_ISP_Tuning_SetSensorFPS((IMPVI_NUM)vi_num, &fps_num, &fps_den);

#elif defined(SOC_T32) || defined(SOC_T41)
    IMPISPSensorFps fps = { .num = fps_num, .den = fps_den };
    return IMP_ISP_Tuning_SetSensorFPS((IMPVI_NUM)vi_num, &fps);
#endif
}
```

### hal_isp_add_sensor() Dispatch

```c
int hal_isp_add_sensor(int vi_num, const hal_sensor_config *cfg) {
    IMPSensorInfo info = {0};
    strncpy(info.name, cfg->name, sizeof(info.name));
    info.cbus_type = TX_SENSOR_CONTROL_INTERFACE_I2C;
    strncpy(info.i2c.type, cfg->name, sizeof(info.i2c.type));
    info.i2c.addr = cfg->i2c_addr;
    info.i2c.i2c_adapter_id = cfg->i2c_adapter;

#if defined(SOC_T40) || defined(SOC_T32) || defined(SOC_T41)
    // Extended fields
    info.rst_gpio = cfg->rst_gpio;    // int on T40+
    info.pwdn_gpio = cfg->pwdn_gpio;
    info.sensor_id = cfg->sensor_id;
    info.video_interface = cfg->vin_type;  // MIPI CSI0/CSI1/DVP
    info.mclk = cfg->mclk;
    info.default_boot = cfg->default_boot;
    return IMP_ISP_AddSensor((IMPVI_NUM)vi_num, &info);
#elif defined(SOC_T23)
    info.sensor_id = cfg->sensor_id;  // T23 has sensor_id
    return IMP_ISP_AddSensor(&info);
#else
    // T20/T21/T30/T31
    return IMP_ISP_AddSensor(&info);
#endif
}
```

### Key HAL Design Decisions

1. **Compile-time vs runtime dispatch**: Given the binary incompatibility of struct layouts and function signatures, compile-time dispatch (`#ifdef SOC_*`) is strongly recommended over runtime dispatch with function pointers.

2. **IMPVI_NUM abstraction**: The HAL should always accept a `vi_num` parameter even for SoCs that ignore it (T20-T31). This keeps the API uniform.

3. **Pointer wrapping**: For Gen3 SoCs, the HAL must take stack-local copies of scalar values and pass their addresses. This is a source of bugs if the HAL tries to pass temporaries.

4. **T23 dual-sensor**: The T23 dual-sensor approach (`_Sec` suffix) is fundamentally different from T40/T41 (`IMPVI_NUM` parameter). The HAL should abstract both into the same `vi_num` parameter:
   ```c
   #if defined(SOC_T23)
   if (vi_num == 0)
       return IMP_ISP_Tuning_SetBrightness(bright);
   else
       return IMP_ISP_Tuning_SetBrightness_Sec(bright);
   #endif
   ```

5. **Return type**: Gen1/Gen2 return `int`, Gen3 returns `int32_t`. Functionally identical on these platforms but the HAL should use a consistent return type.

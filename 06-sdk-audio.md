# SDK Audio API (`imp_audio.h`)

The audio module provides record/playback, encoding/decoding, volume/gain control, echo cancellation (AEC), automatic gain control (AGC), noise suppression (NS), and high-pass filtering (HPF). The API is organized around four subsystems: Audio Input (AI), Audio Output (AO), Audio Encoding (AENC), and Audio Decoding (ADEC).

**Header location:** `imp/imp_audio.h`

---

## 1. Function Presence Matrix

A checkmark indicates the function declaration is present in that SoC's header.

### 1.1 Audio Input (AI) -- Core

| Function | T20 | T21 | T23 | T30 | T31 | T32 | T40 | T41 |
|---|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| `IMP_AI_SetPubAttr` | x | x | x | x | x | x | x | x |
| `IMP_AI_GetPubAttr` | x | x | x | x | x | x | x | x |
| `IMP_AI_Enable` | x | x | x | x | x | x | x | x |
| `IMP_AI_Disable` | x | x | x | x | x | x | x | x |
| `IMP_AI_EnableChn` | x | x | x | x | x | x | x | x |
| `IMP_AI_DisableChn` | x | x | x | x | x | x | x | x |
| `IMP_AI_PollingFrame` | x | x | x | x | x | x | x | x |
| `IMP_AI_GetFrame` | x | x | x | x | x | x | x | x |
| `IMP_AI_ReleaseFrame` | x | x | x | x | x | x | x | x |
| `IMP_AI_SetChnParam` | x | x | x | x | x | x | x | x |
| `IMP_AI_GetChnParam` | x | x | x | x | x | x | x | x |

### 1.2 Audio Input (AI) -- Volume and Gain

| Function | T20 | T21 | T23 | T30 | T31 | T32 | T40 | T41 |
|---|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| `IMP_AI_SetVol` | x | x | x | x | x | x | x | x |
| `IMP_AI_GetVol` | x | x | x | x | x | x | x | x |
| `IMP_AI_SetVolMute` | x | x | x | x | x | x | x | x |
| `IMP_AI_SetGain` | x | x | x | x | x | x | x | x |
| `IMP_AI_GetGain` | x | x | x | x | x | x | x | x |
| `IMP_AI_SetAlcGain` | - | x | - | - | x | - | - | - |
| `IMP_AI_GetAlcGain` | - | x | - | - | x | - | - | - |
| `IMP_AI_SetDigitalGain` | - | - | - | - | - | x | x | x |
| `IMP_AI_GetDigitalGain` | - | - | - | - | - | x | x | x |

### 1.3 Audio Input (AI) -- Processing (AEC/NS/AGC/HPF/HS)

| Function | T20 | T21 | T23 | T30 | T31 | T32 | T40 | T41 |
|---|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| `IMP_AI_EnableAec` | x | x | x | x | x | x | x | x |
| `IMP_AI_DisableAec` | x | x | x | x | x | x | x | x |
| `IMP_AI_Set_WebrtcProfileIni_Path` | - | - | x | - | x | x | x | x |
| `IMP_AI_EnableNs` | x | x | x | x | x | x | x | x |
| `IMP_AI_DisableNs` | x | x | x | x | x | x | x | x |
| `IMP_AI_EnableAgc` | x | x | x | x | x | x | x | x |
| `IMP_AI_DisableAgc` | x | x | x | x | x | x | x | x |
| `IMP_AI_SetAgcMode` | - | - | - | - | x | - | - | - |
| `IMP_AI_EnableHpf` | x | x | x | x | x | x | x | x |
| `IMP_AI_DisableHpf` | x | x | x | x | x | x | x | x |
| `IMP_AI_SetHpfCoFrequency` | - | - | x | - | x | x | x | x |
| `IMP_AI_EnableHs` | - | - | x | - | - | x | x | x |
| `IMP_AI_DisableHs` | - | - | x | - | - | x | x | x |

### 1.4 Audio Input (AI) -- Reference Frame and Raw Access

| Function | T20 | T21 | T23 | T30 | T31 | T32 | T40 | T41 |
|---|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| `IMP_AI_GetFrameAndRef` | x | x | x | x | x | x | x | x |
| `IMP_AI_EnableAecRefFrame` | x | x | x | x | x | x | x | x |
| `IMP_AI_DisableAecRefFrame` | x | x | x | x | x | x | x | x |
| `IMP_AI_EnableAlgo` | - | - | x | - | - | x | - | x |
| `IMP_AI_DisableAlgo` | - | - | x | - | - | x | - | x |
| `IMP_AI_EnableGetRaw` | - | - | - | - | - | x | - | - |
| `IMP_AI_DisableGetRaw` | - | - | - | - | - | x | - | - |
| `IMP_AI_GetFrameAndRaw` | - | - | - | - | - | x | - | - |

### 1.5 Audio Input (AI) -- T32-Only Advanced Processing

| Function | T20 | T21 | T23 | T30 | T31 | T32 | T40 | T41 |
|---|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| `IMP_AI_EnableLpf` | - | - | - | - | - | x | - | - |
| `IMP_AI_SetLpfCoFrequency` | - | - | - | - | - | x | - | - |
| `IMP_AI_DisableLpf` | - | - | - | - | - | x | - | - |
| `IMP_AI_EnableDrcAndEq` | - | - | - | - | - | x | - | - |
| `IMP_AI_DisableDrcAndEq` | - | - | - | - | - | x | - | - |
| `IMP_Audio_Select_Codec` | - | - | - | - | - | x | - | - |

### 1.6 Audio Output (AO) -- Core

| Function | T20 | T21 | T23 | T30 | T31 | T32 | T40 | T41 |
|---|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| `IMP_AO_SetPubAttr` | x | x | x | x | x | x | x | x |
| `IMP_AO_GetPubAttr` | x | x | x | x | x | x | x | x |
| `IMP_AO_Enable` | x | x | x | x | x | x | x | x |
| `IMP_AO_Disable` | x | x | x | x | x | x | x | x |
| `IMP_AO_EnableChn` | x | x | x | x | x | x | x | x |
| `IMP_AO_DisableChn` | x | x | x | x | x | x | x | x |
| `IMP_AO_SendFrame` | x | x | x | x | x | x | x | x |
| `IMP_AO_PauseChn` | x | x | x | x | x | x | x | x |
| `IMP_AO_ResumeChn` | x | x | x | x | x | x | x | x |
| `IMP_AO_ClearChnBuf` | x | x | x | x | x | x | x | x |
| `IMP_AO_QueryChnStat` | x | x | x | x | x | x | x | x |

### 1.7 Audio Output (AO) -- Volume, Gain, and Mute

| Function | T20 | T21 | T23 | T30 | T31 | T32 | T40 | T41 |
|---|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| `IMP_AO_SetVol` | x | x | x | x | x | x | x | x |
| `IMP_AO_GetVol` | x | x | x | x | x | x | x | x |
| `IMP_AO_SetVolMute` | x | x | x | x | x | x | x | x |
| `IMP_AO_SetGain` | x | x | x | x | x | x | x | x |
| `IMP_AO_GetGain` | x | x | x | x | x | x | x | x |
| `IMP_AO_SetDigitalGain` | - | - | - | - | - | x | x | x |
| `IMP_AO_GetDigitalGain` | - | - | - | - | - | x | x | x |
| `IMP_AO_Soft_Mute` | x | x | x | x | x | x | x | x |
| `IMP_AO_Soft_UNMute` | x | x | x | x | x | x | x | x |

### 1.8 Audio Output (AO) -- Processing and Cache

| Function | T20 | T21 | T23 | T30 | T31 | T32 | T40 | T41 |
|---|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| `IMP_AO_EnableAgc` | x | x | x | x | x | x | x | x |
| `IMP_AO_DisableAgc` | x | x | x | x | x | x | x | x |
| `IMP_AO_EnableHpf` | x | x | x | x | x | x | x | x |
| `IMP_AO_DisableHpf` | x | x | x | x | x | x | x | x |
| `IMP_AO_SetHpfCoFrequency` | - | - | x | - | x | x | x | x |
| `IMP_AO_EnableAlgo` | - | - | x | - | - | x | - | x |
| `IMP_AO_DisableAlgo` | - | - | x | - | - | x | - | x |
| `IMP_AO_CacheSwitch` | x | x | x | x | x | x | x | x |
| `IMP_AO_FlushChnBuf` | x | x | x | x | x | x | x | x |

### 1.9 Audio Encoding (AENC)

All AENC functions are present on all 8 SoCs (T20, T21, T23, T30, T31, T32, T40, T41):

| Function | All SoCs |
|---|:---:|
| `IMP_AENC_CreateChn` | x |
| `IMP_AENC_DestroyChn` | x |
| `IMP_AENC_SendFrame` | x |
| `IMP_AENC_PollingStream` | x |
| `IMP_AENC_GetStream` | x |
| `IMP_AENC_ReleaseStream` | x |
| `IMP_AENC_RegisterEncoder` | x |
| `IMP_AENC_UnRegisterEncoder` | x |

### 1.10 Audio Decoding (ADEC)

All ADEC functions are present on all 8 SoCs (T20, T21, T23, T30, T31, T32, T40, T41):

| Function | All SoCs |
|---|:---:|
| `IMP_ADEC_CreateChn` | x |
| `IMP_ADEC_DestroyChn` | x |
| `IMP_ADEC_SendStream` | x |
| `IMP_ADEC_PollingStream` | x |
| `IMP_ADEC_GetStream` | x |
| `IMP_ADEC_ReleaseStream` | x |
| `IMP_ADEC_ClearChnBuf` | x |
| `IMP_ADEC_RegisterDecoder` | x |
| `IMP_ADEC_UnRegisterDecoder` | x |

---

## 2. Struct Definitions

### 2.1 `IMPAudioIOAttr` -- Audio I/O Device Attributes

Identical across all 8 SoCs.

```c
typedef struct {
    IMPAudioSampleRate samplerate;  /* Audio sampling rate */
    IMPAudioBitWidth   bitwidth;    /* Audio sampling precision */
    IMPAudioSoundMode  soundmode;   /* Audio channel mode */
    int frmNum;                     /* Cached frame count [2, MAX_AUDIO_FRAME_NUM] */
    int numPerFrm;                  /* Sample points per frame */
    int chnCnt;                     /* Number of supported channels */
} IMPAudioIOAttr;
```

### 2.2 `IMPAudioFrame` -- Audio Frame

Identical across all 8 SoCs.

```c
typedef struct {
    IMPAudioBitWidth  bitwidth;     /* Audio sampling precision */
    IMPAudioSoundMode soundmode;    /* Audio channel mode */
    uint32_t *virAddr;              /* Frame data virtual address */
    uint32_t  phyAddr;              /* Frame data physical address */
    int64_t   timeStamp;            /* Frame timestamp */
    int       seq;                  /* Frame sequence number */
    int       len;                  /* Frame data length */
} IMPAudioFrame;
```

### 2.3 `IMPAudioIChnParam` -- Audio Input Channel Parameters

This struct has a **variant** across SoCs. The `aecChn` field is present on T23, T32, T40, and T41 only.

**T20, T21, T30, T31** (2 fields):
```c
typedef struct {
    int usrFrmDepth;    /* Audio frame buffer depth */
    int Rev;            /* Reserved */
} IMPAudioIChnParam;
```

**T23, T32, T40, T41** (3 fields):
```c
typedef struct {
    int usrFrmDepth;            /* Audio frame buffer depth */
    IMPAudioAecChn aecChn;      /* AEC channel select */
    int Rev;                    /* Reserved */
} IMPAudioIChnParam;
```

The `aecChn` field allows selecting which AEC channel to use (first/left, second/right, third, fourth) when setting up multi-channel echo cancellation configurations.

### 2.4 `IMPAudioOChnState` -- Audio Output Channel Cache State

Identical across all 8 SoCs.

```c
typedef struct {
    int chnTotalNum;    /* Total output channel cache blocks */
    int chnFreeNum;     /* Free cache blocks */
    int chnBusyNum;     /* Used cache blocks */
} IMPAudioOChnState;
```

### 2.5 `IMPAudioStream` -- Audio Stream

Identical across all 8 SoCs.

```c
typedef struct {
    uint8_t  *stream;       /* Data stream pointer */
    uint32_t  phyAddr;      /* Data stream physical address */
    int       len;          /* Audio stream length */
    int64_t   timeStamp;    /* Timestamp */
    int       seq;          /* Audio stream sequence number */
} IMPAudioStream;
```

### 2.6 `IMPAudioEncChnAttr` -- Audio Encoding Channel Attributes

Identical across all 8 SoCs.

```c
typedef struct {
    IMPAudioPalyloadType type;  /* Audio payload data type */
    int       bufSize;          /* Buffer size in frames [2, MAX_AUDIO_FRAME_NUM] */
    uint32_t *value;            /* Protocol attribute pointer */
} IMPAudioEncChnAttr;
```

### 2.7 `IMPAudioEncEncoder` -- Custom Encoder Registration

Identical across all 8 SoCs. Used with `IMP_AENC_RegisterEncoder`.

```c
typedef struct {
    IMPAudioPalyloadType type;      /* Encoding protocol type */
    int  maxFrmLen;                 /* Maximum stream length */
    char name[16];                  /* Encoder name */
    int (*openEncoder)(void *encoderAttr, void *encoder);
    int (*encoderFrm)(void *encoder, IMPAudioFrame *data,
                      unsigned char *outbuf, int *outLen);
    int (*closeEncoder)(void *encoder);
} IMPAudioEncEncoder;
```

### 2.8 `IMPAudioDecChnAttr` -- Audio Decoding Channel Attributes

Identical across all 8 SoCs.

```c
typedef struct {
    IMPAudioPalyloadType type;  /* Audio decoding protocol type */
    int              bufSize;   /* Audio decoder cache size */
    IMPAudioDecMode  mode;      /* Decoding mode */
    void            *value;     /* Specific protocol attribute pointer */
} IMPAudioDecChnAttr;
```

### 2.9 `IMPAudioDecDecoder` -- Custom Decoder Registration

Identical across all 8 SoCs. Used with `IMP_ADEC_RegisterDecoder`.

```c
typedef struct {
    IMPAudioPalyloadType type;      /* Audio decoding protocol type */
    char name[16];                  /* Audio decoder name */
    int (*openDecoder)(void *decoderAttr, void *decoder);
    int (*decodeFrm)(void *decoder, unsigned char *inbuf, int inLen,
                     unsigned short *outbuf, int *outLen, int *chns);
    int (*getFrmInfo)(void *decoder, void *info);
    int (*closeDecoder)(void *decoder);
} IMPAudioDecDecoder;
```

### 2.10 `IMPAudioAgcConfig` -- AGC Configuration

Identical across all 8 SoCs.

```c
typedef struct {
    int TargetLevelDbfs;    /* Gain level [0, 31], target volume in -dB. Smaller = louder. */
    int CompressionGaindB;  /* Max gain value [0, 90]. 0 = no gain. Higher = more gain. */
} IMPAudioAgcConfig;
```

### 2.11 `IMPAudioDrcEqConfig` -- DRC/EQ Configuration (T32 only)

Present only on T32.

```c
typedef struct {
    int    threshold_low;
    int    threshold_high;
    float  ratio;
    int    eq_type;
    int    freq_low;
    int    freq_high;
    int    size;
    float *EqTable;
} IMPAudioDrcEqConfig;
```

Only supports 8K and 16K sample rates.

---

## 3. Enums

### 3.1 `IMPBlock` -- Blocking Mode

Identical across all 8 SoCs.

```c
typedef enum {
    BLOCK   = 0,    /* Blocking */
    NOBLOCK = 1,    /* Non-blocking */
} IMPBlock;
```

### 3.2 `IMPAudioSampleRate` -- Sample Rate

The set of defined rates varies by SoC:

| Value | Constant | T20 | T21 | T23 | T30 | T31 | T32 | T40 | T41 |
|---|---|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| 8000 | `AUDIO_SAMPLE_RATE_8000` | x | x | x | x | x | x | x | x |
| 12000 | `AUDIO_SAMPLE_RATE_12000` | - | - | x | - | - | x | - | x |
| 16000 | `AUDIO_SAMPLE_RATE_16000` | x | x | x | x | x | x | x | x |
| 24000 | `AUDIO_SAMPLE_RATE_24000` | x | x | x | x | x | x | x | x |
| 32000 | `AUDIO_SAMPLE_RATE_32000` | - | x | x | x | x | x | x | x |
| 44100 | `AUDIO_SAMPLE_RATE_44100` | x | x | x | x | x | x | x | x |
| 48000 | `AUDIO_SAMPLE_RATE_48000` | x | x | x | x | x | x | x | x |
| 96000 | `AUDIO_SAMPLE_RATE_96000` | x | x | x | x | x | x | x | x |

Notable:
- T20 is missing 12KHz and 32KHz.
- T21 and T30 are missing 12KHz.
- T40 is missing 12KHz.
- Only T23, T32, and T41 have the full set of 8 rates.

### 3.3 `IMPAudioBitWidth` -- Bit Width

Identical across all 8 SoCs. Only one value defined:

```c
typedef enum {
    AUDIO_BIT_WIDTH_16 = 16,    /* 16-bit sampling precision */
} IMPAudioBitWidth;
```

### 3.4 `IMPAudioSoundMode` -- Channel Mode

Identical across all 8 SoCs:

```c
typedef enum {
    AUDIO_SOUND_MODE_MONO   = 1,    /* Mono */
    AUDIO_SOUND_MODE_STEREO = 2,    /* Stereo */
} IMPAudioSoundMode;
```

### 3.5 `IMPAudioAecChn` -- AEC Channel Select

Present on **T23, T32, T40, T41** only. Absent from T20, T21, T30, T31.

```c
typedef enum {
    AUDIO_AEC_CHANNEL_FIRST_LEFT   = 0,  /* First channel or left channel */
    AUDIO_AEC_CHANNEL_SECOND_RIGHT = 1,  /* Second channel or right channel */
    AUDIO_AEC_CHANNEL_THIRD        = 2,  /* Third channel */
    AUDIO_AEC_CHANNEL_FOURTH       = 3,  /* Fourth channel */
} IMPAudioAecChn;
```

### 3.6 `IMPAudioPalyloadType` -- Audio Payload/Codec Type

Identical across all 8 SoCs. Note the original "Palyload" typo is preserved from the SDK.

```c
typedef enum {
    PT_PCM    = 0,
    PT_G711A  = 1,
    PT_G711U  = 2,
    PT_G726   = 3,
    PT_AEC    = 4,
    PT_ADPCM  = 5,
    PT_MAX    = 6,
} IMPAudioPalyloadType;
```

Built-in codec support: PT_G711A, PT_G711U, PT_G726. Additional codecs must be registered via `IMP_AENC_RegisterEncoder` / `IMP_ADEC_RegisterDecoder`.

### 3.7 `IMPAudioDecMode` -- Decoding Mode

Identical across all 8 SoCs:

```c
typedef enum {
    ADEC_MODE_PACK   = 0,   /* Pack decoding */
    ADEC_MODE_STREAM = 1,   /* Stream decoding */
} IMPAudioDecMode;
```

### 3.8 `Level_ns` -- Noise Suppression Level

Identical across all 8 SoCs:

```c
enum Level_ns {
    NS_LOW,         /* Low noise suppression */
    NS_MODERATE,    /* Medium noise suppression */
    NS_HIGH,        /* High noise suppression */
    NS_VERYHIGH     /* Maximum noise suppression */
};
```

Higher levels suppress more noise but may lose audio detail.

### 3.9 `Agc_mode` -- AGC Working Mode (T31 only)

Present **only on T31**.

```c
enum Agc_mode {
    kAgcModeAdaptiveAnalog  = 1,
    kAgcModeAdaptiveDigital,    /* = 2 */
    kAgcModeFixedDigital        /* = 3 */
};
```

Used with `IMP_AI_SetAgcMode()`. Must be set before enabling AGC.

---

## 4. SoC-Specific Feature Summary

### 4.1 Generation 1: T20, T30

Baseline audio API. No HPF cutoff frequency control, no howling suppression, no WebRTC path configuration, no algorithm enable/disable, no digital gain.

- T20 is further limited: missing `AUDIO_SAMPLE_RATE_32000`.

### 4.2 Generation 1.5: T21

Same baseline as T20/T30 but adds:
- `AUDIO_SAMPLE_RATE_32000`
- `IMP_AI_SetAlcGain` / `IMP_AI_GetAlcGain` -- ALC (Automatic Level Control) PGA gain, range [0..7], corresponding to [-13.5dB .. +28.5dB], 6dB steps.

### 4.3 Generation 2: T23, T31

Both add:
- `IMP_AI_Set_WebrtcProfileIni_Path` -- allows relocating `webrtc_profile.ini`.
- `IMP_AI_SetHpfCoFrequency` / `IMP_AO_SetHpfCoFrequency` -- HPF cutoff frequency control.

T23 additionally adds:
- `IMP_AI_EnableHs` / `IMP_AI_DisableHs` -- Howling suppression.
- `IMP_AI_EnableAlgo` / `IMP_AI_DisableAlgo` / `IMP_AO_EnableAlgo` / `IMP_AO_DisableAlgo` -- Batch enable/disable of NS/AGC/HPF algorithms.
- `IMPAudioAecChn` enum and `aecChn` field in `IMPAudioIChnParam`.
- `AUDIO_SAMPLE_RATE_12000`.

T31 additionally adds:
- `IMP_AI_SetAgcMode` -- AGC mode selection (adaptive analog, adaptive digital, fixed digital).
- `IMP_AI_SetAlcGain` / `IMP_AI_GetAlcGain` -- ALC PGA gain, range [0..7], [-13.5dB .. +28.5dB], 6dB steps.
- `Agc_mode` enum.

### 4.4 Generation 3: T32

Most feature-rich SoC. Inherits everything from T23/T31 generation plus:
- `IMP_AI_SetDigitalGain` / `IMP_AI_GetDigitalGain` -- Digital gain [0..255], [-97dB .. +30dB], 0.5dB steps.
- `IMP_AO_SetDigitalGain` / `IMP_AO_GetDigitalGain` -- Digital gain [0..255], [-120dB .. +7dB], 0.5dB steps.
- `IMP_AI_EnableGetRaw` / `IMP_AI_DisableGetRaw` / `IMP_AI_GetFrameAndRaw` -- Access raw (pre-processing) audio frames.
- `IMP_AI_EnableLpf` / `IMP_AI_SetLpfCoFrequency` / `IMP_AI_DisableLpf` -- Low-pass filter.
- `IMP_AI_EnableDrcAndEq` / `IMP_AI_DisableDrcAndEq` -- Dynamic range compression and equalizer (8K/16K only).
- `IMP_Audio_Select_Codec` -- Internal/external codec selection. Must be called after AI is enabled; does not support dynamic switching during active streaming.
- `IMPAudioDrcEqConfig` struct.

### 4.5 Generation 3+: T40, T41

Both add digital gain (same ranges as T32) but lack T32's raw frame access, LPF, DRC/EQ, and codec select.

T40:
- Has howling suppression, HPF cutoff control, WebRTC path, digital gain.
- Does **not** have algorithm enable/disable (`EnableAlgo`/`DisableAlgo`).

T41:
- Has howling suppression, HPF cutoff control, WebRTC path, digital gain.
- **Does** have algorithm enable/disable (like T23 and T32).

---

## 5. Gain Range Reference

Audio gain ranges differ across SoC generations. This table summarizes the hardware gain parameters.

### 5.1 Volume (`SetVol` / `GetVol`)

Consistent across all SoCs:
- Range: [-30 .. 120]
- -30 = mute
- 60 = unity (no software adjustment)
- Step: 0.5 dB
- Below 60: each -1 reduces by 0.5 dB
- Above 60: each +1 increases by 0.5 dB

### 5.2 Hardware Gain (`SetGain` / `GetGain`)

| SoC | AI Gain Range | AI Gain dB Range | AO Gain Range | AO Gain dB Range |
|---|---|---|---|---|
| T20 | [0..31] | (not specified) | [0..31] | (not specified) |
| T21 | [0..31] | (not specified) | [0..0xcb] | (not specified) |
| T23 | [0..31] | (not specified) | [0..0x1f] | (not specified) |
| T30 | [0..31] | (not specified) | [0..31] | (not specified) |
| T31 | [0..31] | (not specified) | [0..0x1f] | [-39dB .. +6dB], 1.5dB step |
| T32 | [0..31] | (not specified) | [0..0x1f] | [-39dB .. +6dB], 1.5dB step |
| T40 | [0..31] | (not specified) | [0..0x1f] | [-39dB .. +6dB], 1.5dB step |
| T41 | [0..31] | [-18dB .. +28.5dB], 1.5dB step | [0..0x1f] | [-39dB .. +6dB], 1.5dB step |

### 5.3 ALC/PGA Gain (`SetAlcGain` -- T21, T31 only)

- Range: [0..7]
- T31: [-13.5dB .. +28.5dB], 6dB step

### 5.4 Digital Gain (`SetDigitalGain` -- T32, T40, T41)

- AI digital gain: [0..255], [-97dB .. +30dB], 0.5dB step
- AO digital gain: [0..255], [-120dB .. +7dB], 0.5dB step

---

## 6. HAL Implications

### 6.1 Audio Initialization (`hal_audio_init`)

When initializing audio across SoC variants, the HAL must:

1. **Populate `IMPAudioIOAttr` carefully.** The struct layout is the same on all SoCs, but the valid `samplerate` values differ. Setting `AUDIO_SAMPLE_RATE_12000` on T20/T21/T30/T40 may fail since the enum value is not defined in those headers.

2. **Handle `IMPAudioIChnParam` differences.** On T23/T32/T40/T41 the struct has an `aecChn` field between `usrFrmDepth` and `Rev`. Since the struct size differs (12 bytes vs 8 bytes), using a single binary for multiple SoCs requires conditional compilation or runtime struct handling. The HAL should zero-initialize the struct and only set `aecChn` on platforms that support it.

3. **Call sequence is identical across all SoCs:**
   ```
   IMP_AI_SetPubAttr(devID, &attr)
   IMP_AI_Enable(devID)
   IMP_AI_SetChnParam(devID, chnID, &chnParam)
   IMP_AI_EnableChn(devID, chnID)
   ```

### 6.2 Audio Processing (AGC/NS/HPF) Enable Flow

The processing functions have different enable patterns across SoC generations:

**Generation 1 pattern (T20, T21, T30):**
```
IMP_AI_EnableNs(&attr, mode);        // Direct enable
IMP_AI_EnableAgc(&attr, agcConfig);  // Direct enable
IMP_AI_EnableHpf(&attr);             // Direct enable
```

**Generation 2+ pattern (T23, T31, T32, T40, T41):**
```
IMP_AI_SetHpfCoFrequency(freq);     // Set cutoff first (T23+)
IMP_AI_EnableHpf(&attr);            // Then enable
IMP_AI_EnableNs(&attr, mode);
IMP_AI_EnableAgc(&attr, agcConfig);
IMP_AI_EnableAlgo(devID, chnID);    // Batch activate (T23, T32, T41 only)
```

On SoCs with `EnableAlgo`, individual NS/AGC/HPF enables configure the algorithms, but they may not become active until `IMP_AI_EnableAlgo` is called.

**T31-specific AGC mode:**
```
IMP_AI_SetAgcMode(kAgcModeAdaptiveDigital);  // Must be called before EnableAgc
IMP_AI_EnableAgc(&attr, agcConfig);
```

### 6.3 AEC Configuration

AEC is configured through `webrtc_profile.ini`. The file location can be customized on T23+ via `IMP_AI_Set_WebrtcProfileIni_Path`. This function must be called **before** `IMP_AI_EnableAec`.

Key AEC parameters:
- `[Set_Far_Frame]` / `Frame_V` -- SPK playback amplitude scaling
- `[Set_Near_Frame]` / `Frame_V` -- MIC recording amplitude scaling
- `delay_ms` -- Acoustic delay between SPK and MIC (typically 150ms)

AEC supports only 8KHz and 16KHz sample rates. Frame size must be a multiple of 10ms of audio data.

AEC implicitly includes NS, AGC, and HPF functionality. When AEC is enabled, there is no need to separately enable these processing features.

### 6.4 Custom Codec Registration Pattern

To add a codec beyond the built-in PT_G711A/PT_G711U/PT_G726:

**Encoder registration:**
```c
int handle;
IMPAudioEncEncoder encoder = {
    .type = PT_MAX,  // or custom type
    .maxFrmLen = 1024,
    .name = "OPUS",
    .openEncoder  = my_opus_open,
    .encoderFrm   = my_opus_encode,
    .closeEncoder = my_opus_close,
};
IMP_AENC_RegisterEncoder(&handle, &encoder);

// Create channel with the registered type
IMPAudioEncChnAttr chnAttr = {
    .type = encoder.type,
    .bufSize = 20,
    .value = NULL,
};
IMP_AENC_CreateChn(0, &chnAttr);
```

**Decoder registration** follows the same pattern using `IMP_ADEC_RegisterDecoder`.

The codec handle returned by registration must be preserved and passed to `IMP_AENC_UnRegisterEncoder` / `IMP_ADEC_UnRegisterDecoder` for cleanup.

### 6.5 Reference Frame Access

All SoCs support reference frame access for external AEC processing:
```c
IMP_AI_EnableAecRefFrame(aiDevID, aiChn, aoDevID, aoChn);
IMP_AI_GetFrameAndRef(aiDevID, aiChn, &frm, &ref, BLOCK);
// frm = processed audio, ref = speaker reference
IMP_AI_ReleaseFrame(aiDevID, aiChn, &frm);
```

T32 additionally supports raw (pre-processing) frame access:
```c
IMP_AI_EnableGetRaw(aiDevID, aiChn);
IMP_AI_GetFrameAndRaw(aiDevID, aiChn, &frm, &raw_frm, BLOCK);
// frm = processed, raw_frm = raw capture before NS/AGC/HPF
```

### 6.6 T32 Codec Selection

T32 supports both internal and external audio codecs:
```c
IMP_AI_Enable(devID);
IMP_AI_EnableChn(devID, chnID);
IMP_AI_SetGain(devID, chnID, gain);
IMP_Audio_Select_Codec(devID, chnID, SELECT_EXTERNAL);  // or SELECT_INTERNAL
```

Restrictions:
- Must be called after device/channel enable and gain setup
- Does not support dynamic switching during active recording/playback/streaming
- Internal codec is used by default if this function is not called

---

## 7. Constants

```c
#define MAX_AUDIO_FRAME_NUM 50
```

All SoCs define this identically. Frame buffer parameters (`frmNum`, `bufSize`, `usrFrmDepth`) must be in the range [2, `MAX_AUDIO_FRAME_NUM`].

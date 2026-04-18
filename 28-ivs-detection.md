# IVS Detection — Motion Tracking, Person Detection, YOLO Inference

## Overview

RVD provides motion and object detection via the IMP IVS (Intelligent
Video Services) pipeline. Three algorithms are available:

- **move** — grid-based motion detection with bounding box tracking
  (default, zero extra dependencies)
- **persondet** — person detection via Ingenic's built-in PersonDet
  model (requires `libpersonDet_inf.so`)
- **yolo** — JZDL standalone YOLOv5 inference for multi-class detection
  (requires `libjzdl.m.so` + model file)

All algorithms store results in the same `rss_ivs_detect_result_t`
structure. ROD draws bounding boxes on the sub-stream OSD overlay.
RMD polls detection state for recording triggers.

---

## Architecture

```
FrameSource ch1 (sub-stream 640x360)
     │
     ├──→ IMP_IVS Group ──→ move / persondet algorithm
     │         │
     │         └──→ result callback ──→ ivs_detections[]
     │
     └──→ IMP_FrameSource_GetFrame ──→ JZDL thread (yolo only)
                                           │
                                           └──→ ivs_detections[]
```

- IVS group bound to sub-stream FrameSource (channel 1)
- `move` and `persondet` use the IMP IVS pipeline (SDK-managed frame
  delivery)
- `yolo` bypasses IVS — a standalone thread reads frames via
  `IMP_FrameSource_GetFrame`
- Results stored in `ivs_detections` (mutex-protected), queried via
  the `ivs-detections` control command
- 2-second detection persistence to prevent OSD flicker
- OSD pool dynamically sized from config before HAL init (448KB base,
  +920KB with detection overlay)

---

## Configuration

Full `[motion]` config section with defaults:

```ini
[motion]
# General
enabled = false
algorithm = move              # move, base_move, persondet, yolo
sensitivity = 2               # move: 0-4, persondet: 0-5
skip_frames = 5               # frames to skip between evaluations
cooldown_sec = 10             # seconds to suppress after detection
poll_interval_ms = 500        # polling interval for result retrieval

# Recording trigger
record = false                # trigger recording on detection
record_post_sec = 10          # seconds to continue recording after last detection
gpio_pin = -1                 # GPIO pin to assert on detection (-1 = disabled)

# move algorithm
grid = 4x4                    # detection grid (cols x rows)

# Explicit ROI override (disables grid)
roi_count = 0                 # number of ROI rectangles (0 = use grid)
roi0 = 100,50,400,300         # x0,y0,x1,y1 in sub-stream pixel space
roi1 = 420,50,600,300         # additional ROI regions as needed

# persondet algorithm
det_distance = 2              # detection distance index (0=6m 1=8m 2=10m 3=11m 4=13m)
motion_trigger = true         # require move pre-trigger before persondet inference

# yolo algorithm
model = /usr/share/models/magik_model_yolov5.bin
num_classes = 4               # number of classes in the model
conf_threshold = 40           # confidence threshold (0-100)
nms_threshold = 50            # NMS IoU threshold (0-100)
```

---

## Algorithms

### move

Uses `IMP_IVS_CreateMoveInterface` from the SDK.

Divides the sub-stream frame into a grid (default 4x4 = 16 zones).
Each zone has independent sensitivity. When motion is detected in one
or more zones, the algorithm computes a bounding box encompassing all
active zones for the OSD overlay.

When explicit ROIs are configured (`roi_count > 0`), grid mode is
disabled. The algorithm falls back to a simple motion flag per ROI
region — no spatial bounding box.

No external libraries required.

### base_move

Uses `IMP_IVS_CreateBaseMoveInterface` from the SDK.

Single global motion detection with no spatial information. Returns
only a boolean motion state — no grid zones, no bounding box.

Lighter weight than `move`. Suitable for basic recording triggers
where spatial information is not needed.

### persondet

Uses `PersonDetInterfaceInit` from `libpersonDet_inf.so`.

Built-in deep learning model (no external model file). Detects
persons only (`class_id = 0`). Output includes `show_box` coordinates
and a confidence value.

When `motion_trigger = true` (default), the algorithm requires a
motion pre-trigger before invoking person inference. This reduces
unnecessary DL inference on static scenes.

Requirements:
- `libpersonDet_inf.so` + `libjzdl.so` (~1.5MB combined)
- Enable via `BR2_PACKAGE_INGENIC_LIB_PERSONDET` in buildroot

### yolo

Uses the JZDL standalone `BaseNet` API from `libjzdl.m.so`.

Bypasses the IMP IVS pipeline entirely. A dedicated thread reads NV12
frames directly from FrameSource via `IMP_FrameSource_GetFrame`, then
performs:

1. NV12 → RGB conversion
2. Resize to model input dimensions (320x416)
3. Int8 normalization
4. JZDL `BaseNet` forward pass
5. YOLOv5 post-processing: anchor decode (3 scales, 3 anchors each)
   + per-class NMS

Current default model classes:

| class_id | Label |
|----------|-------|
| 0 | person |
| 1 | bicycle |
| 2 | car |
| 3 | motorcycle |

Performance: ~3.7 FPS inference on T31 (~270ms per frame).

Requirements:
- `libjzdl.m.so` (609KB)
- Model file (870KB), default path
  `/usr/share/models/magik_model_yolov5.bin`

Drop-in model support: any YOLOv5s model converted via Ingenic
TransformKit works — the anchor and stride layout is standard across
YOLOv5s variants.

---

## OSD Bounding Box Overlay

ROD creates a full sub-stream (640x360) BGRA overlay region at
position (0,0). It polls RVD's `ivs-detections` control command every
500ms and draws green 2px rectangle outlines for each active
detection.

The overlay has a transparent background — it composites over the
encoded video without obscuring the scene.

Detection overlay is restricted to the sub-stream (stream1). The main
stream is not supported due to IPU OSD buffer size limits — a full
1080p or higher BGRA overlay would exceed practical OSD pool budgets.

OSD pool sizing:

| Detection overlay | Pool size |
|-------------------|-----------|
| Disabled | 448KB |
| Enabled | ~1.5MB |

The pool size is determined at startup from the config and passed to
`IMP_OSD_SetPoolSize` before HAL initialization.

---

## Control Commands

### ivs-status

Returns motion state, person count, and algorithm info.

```
raptorctl rvd ivs-status
```

```json
{"status":"ok","motion":true,"persons":1,"persondet":true}
```

| Field | Type | Description |
|-------|------|-------------|
| `motion` | bool | Motion currently detected |
| `persons` | int | Number of persons currently detected |
| `persondet` | bool | PersonDet algorithm active |

### ivs-detections

Returns current detection bounding boxes.

```
raptorctl rvd ivs-detections
```

```json
{
  "status": "ok",
  "count": 1,
  "detections": [
    {
      "x0": 200,
      "y0": 70,
      "x1": 500,
      "y1": 355,
      "confidence": 0.65,
      "class": 0
    }
  ]
}
```

| Field | Type | Description |
|-------|------|-------------|
| `count` | int | Number of active detections |
| `x0`, `y0` | int | Top-left corner (sub-stream pixel space) |
| `x1`, `y1` | int | Bottom-right corner (sub-stream pixel space) |
| `confidence` | float | Detection confidence (0.0–1.0) |
| `class` | int | Class ID (algorithm-dependent) |

Coordinates are in sub-stream pixel space (640x360). Detections
persist for 2 seconds after the last positive result to prevent
flicker.

---

## Buildroot Integration

Three buildroot packages provide IVS detection dependencies:

| Package | Contents | Size |
|---------|----------|------|
| `BR2_PACKAGE_INGENIC_LIB_JZDL` | `libjzdl.m.so` | 609KB |
| `BR2_PACKAGE_INGENIC_LIB_PERSONDET` | `libpersonDet_inf.so` + `libjzdl.so` | 1.5MB |
| `BR2_PACKAGE_MAGIK_MODELS` | Model files (per-model selection) | varies |

The raptor option `BR2_PACKAGE_THINGINO_RAPTOR_IVS_DETECT` auto-
selects `BR2_PACKAGE_INGENIC_LIB_JZDL` and the YOLOv5 model.

### On-Device Footprint

| Configuration | Extra size |
|---------------|-----------|
| move only | 0 |
| yolo | 1.5MB (609KB lib + 870KB model) |
| yolo + persondet | 3MB |
| + facedet model | +258KB |

---

## Model Conversion

TransformKit converts ONNX models to JZDL `.bin` format for on-device
inference. An NVIDIA GPU is required for the quantization calibration
step.

Pre-exported ONNX models are available in the `magik-toolkit`
repository — no training is required for the default classes.

```sh
magik-transform-tools \
  --framework onnx \
  --target_device Txx \
  --model yolov5s.onnx \
  --output magik_model_yolov5.bin
```

Any YOLOv5s ONNX model works as a drop-in replacement, provided it
uses the standard anchor and stride layout (3 scales at strides 8, 16,
32 with 3 anchors each).

FQAT (Fake Quantization-Aware Training) models produce better post-
quantization accuracy than standard post-training quantization.

---

## Files

| File | Purpose |
|------|---------|
| `rvd/rvd_ivs.c` | IVS pipeline setup, JZDL thread, result processing |
| `rvd/rvd_ctrl.c` | `ivs-status` and `ivs-detections` control commands |
| `rvd/rvd_osd.c` | Detection OSD region creation |
| `rvd/rvd_pipeline.c` | Dynamic OSD pool sizing based on config |
| `rod/rod_main.c` | Detection overlay rendering loop |
| `rod/rod_render.c` | Rectangle drawing primitives |
| `raptor-hal/src/hal_ivs.c` | IVS HAL wrappers (create, destroy, poll) |
| `raptor-hal/src/hal_ivs_jzdl.cpp` | JZDL inference engine + YOLOv5 post-processing |
| `raptor-hal/src/hal_osd.c` | OSD pool size storage and retrieval |

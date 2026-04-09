# FPS Troubleshooting Guide

When the video stream runs below the expected frame rate, the bottleneck can
be at any stage of the pipeline: sensor, ISP, encoder, ring buffer, RTSP
transport, or the receiving client. This guide walks through each stage in
order to isolate the problem.

## Pipeline stages

```
Sensor -> ISP -> Encoder -> SHM Ring -> RSD (RTSP) -> Network -> Client
```

Each stage can independently drop frames. Always start from the left (sensor)
and work right.

---

## 1. Sensor output rate

The sensor is the clock source for the entire pipeline. If it delivers fewer
frames than expected, nothing downstream can compensate.

### Check ISP-reported sensor FPS

```sh
cat /proc/jz/isp/isp-m0
```

Look for:
```
ISP OUTPUT FPS : 25 / 1
```

This is the *configured* rate, not necessarily the *actual* rate.

### Measure actual ISP frame count

```sh
cat /proc/jz/isp/isp-w02; sleep 1; cat /proc/jz/isp/isp-w02
```

The first number in each line is the cumulative frame counter. Subtract them
to get the actual ISP output FPS over that interval.

**Example:**
```
391226, 0
391251, 0
```
391251 - 391226 = **25 fps** (correct)

```
29592, 0
29630, 0   (after 2 seconds)
```
29630 - 29592 = 38 / 2 = **19 fps** (problem)

### Common sensor-side causes

| Symptom | Cause | Fix |
|---------|-------|-----|
| FPS matches target | Sensor OK, look downstream | - |
| FPS below target, AeIntegrationTime near max | Low light, AE extending exposure beyond frame period | Add light, or cap max integration time via ISP tuning |
| FPS consistently wrong (e.g. 19 instead of 25) | Sensor clock misconfigured, wrong MCLK/PLL | Check sensor driver `SCLK`, `total_width`, `total_height` math |
| FPS = 0 or counter not incrementing | Sensor not streaming | Check `dmesg` for sensor probe errors |

### Verify sensor clock math

For a given sensor driver, the expected FPS is:

```
FPS = SCLK / (total_width * total_height)
```

For example, gc4023 boot mode 0:
- SCLK = 121,500,000
- total_width = 3840, total_height = 1500 (from sensor source)
- Expected: 121500000 / (3840 * 1500) = **21.1 fps** (not 25!)

If the math doesn't add up, the sensor driver's register init or clock
setup is wrong.

### Check AE integration time

```sh
cat /proc/jz/isp/isp-m0 | grep AeIntegrationTime
```

If `AeIntegrationTime` is close to or exceeds `total_height - 8` (the VTS
minus blanking), the sensor is in slow-shutter mode and will drop below the
target FPS. This happens in low light.

Fix: increase lighting, or limit max integration time:
```c
// In HAL or via raptorctl
IMP_ISP_Tuning_SetMaxIntegrationTime(max_lines);
```

---

## 2. Encoder throughput

The encoder can become the bottleneck at high resolutions or bitrates,
especially on T20/T23 with limited hardware.

### Check encoder frame drops

Use the Ingenic debug tool:
```sh
impdbg --enc_info; sleep 10; impdbg --enc_info
```

Compare the `df` (dropped frames) counter between the two runs:
```
df:383164  (first)
df:383264  (second, 10s later)
```
383264 - 383164 = 100 dropped / 10s = **10 frames/sec dropped**

### Common encoder causes

| Symptom | Cause | Fix |
|---------|-------|-----|
| df counter increasing | Encoder can't keep up | Reduce resolution, reduce bitrate, or reduce FPS |
| df stable at 0 | Encoder OK, look downstream | - |
| High df with dual-stream | Two encoders competing for VPU time | Reduce sub stream resolution or FPS |

---

## 3. Ring buffer

The SHM ring buffer sits between the encoder (RVD) and consumers (RSD, RMR).
If a consumer falls behind, it gets an overflow error but the producer is
unaffected.

### Check with ringdump

```sh
ringdump main -f -n 100
```

Output shows per-frame timing:
```
#0  seq=1234  len=45678  dt=40012  us  nal=H264_IDR  key=1
#1  seq=1235  len=12345  dt=39998  us  nal=H264_SLICE  key=0
```

- `dt` = microseconds since previous frame
- Expected: 40000us for 25fps, 33333us for 30fps
- `len` = encoded frame size in bytes

The summary at the end shows:
```
Frames:   100
Duration: 4.0 s
Avg FPS:  25.0
```

### Measuring pipeline latency

The `-l` flag measures per-frame pipeline latency from the IMP capture
timestamp to the moment ringdump reads the frame. It calibrates against
the first frame to remove the IMP epoch offset.

```sh
ringdump main -l          # continuous latency output
ringdump main -l -n 100   # 100 frames, then print min/avg/max summary
```

Each frame prints `lat=+Xus (+X.Xms)` showing the capture-to-readout
delay. At the end of a counted run, a summary line reports min, avg,
and max latency across all sampled frames.

### Common ring issues

| Symptom | Cause | Fix |
|---------|-------|-----|
| `dt` matches expected interval | Ring OK | - |
| `dt` consistently too high | Encoder delivering frames late | See encoder section |
| Occasional very large `dt` | Encoder stall or ISP hiccup | Check `dmesg` for errors |
| Ring overflow in consumer logs | Consumer (RSD/RMR) too slow | Increase ring slots, check network/disk |

---

## 4. RTSP transport (RSD)

Even if the ring delivers frames at full rate, the RTSP client may receive
fewer due to network constraints or RSD processing.

### Measure from the client side

```sh
ffmpeg -v error -stats -rtsp_transport tcp -i rtsp://<ip>:554/stream0 -t 10 -f null -
```

Watch the `fps=` value in the stats output. Compare to the ringdump FPS.

For a quick frame count:
```sh
ffmpeg -v error -rtsp_transport tcp -i rtsp://<ip>:554/stream0 -t 10 -f null - 2>&1 | grep frame=
```

Divide the frame count by 10 to get FPS.

### UDP vs TCP

UDP transport can drop RTP packets on lossy networks:
```sh
# Force TCP (interleaved):
ffmpeg -rtsp_transport tcp -i rtsp://<ip>:554/stream0 ...

# Force UDP:
ffmpeg -rtsp_transport udp -i rtsp://<ip>:554/stream0 ...
```

If TCP gives higher FPS than UDP, the network is dropping packets.

### Common RTSP causes

| Symptom | Cause | Fix |
|---------|-------|-----|
| Client FPS matches ringdump | Network OK | - |
| Client FPS lower than ringdump | Network bottleneck or RSD backpressure | Use TCP, reduce bitrate, check WiFi |
| Client FPS drops over time | RSD ring overflow, consumer falling behind | Check RSD logs for overflow warnings |

---

## 5. Quick checklist

```
1. cat /proc/jz/isp/isp-w02; sleep 1; cat /proc/jz/isp/isp-w02
   -> Actual ISP FPS (subtract counters)

2. ringdump main -f -n 100
   -> Encoder output FPS and frame timing

3. ffmpeg -stats -rtsp_transport tcp -i rtsp://<ip>/stream0 -t 10 -f null -
   -> Client-received FPS

4. Compare:
   ISP FPS == ringdump FPS == client FPS  -> All good
   ISP FPS < target                       -> Sensor/clock issue
   ISP FPS OK, ringdump FPS < ISP         -> Encoder dropping
   ringdump FPS OK, client FPS < ring     -> Network/transport issue
```

---

## 6. Useful proc entries

| Path | Description |
|------|-------------|
| `/proc/jz/isp/isp-m0` | ISP info: sensor name, resolution, FPS, AE state |
| `/proc/jz/isp/isp-w02` | ISP frame counter (for measuring actual FPS) |
| `/proc/jz/isp/isp-fs` | Framesource status: running/stopped, scaler, crop |
| `/proc/jz/isp/isp-ivdc` | IVDC status (T23+) |

## 7. Useful tools

| Tool | Usage |
|------|-------|
| `ringdump main -f` | Follow ring frames with timestamps |
| `ringdump main -l` | Measure per-frame pipeline latency (capture to readout) |
| `ringdump main` | Show ring header (slots, data size, codec info, reader count) |
| `ringdump jpeg0` | Show JPEG ring state (reader_count, PIDs for demand debugging) |
| `impdbg --enc_info` | Encoder channel stats (frame counts, drops) |
| `raptorctl rvd get-isp` | ISP tuning values via raptor control socket |
| `ffprobe -v error -show_streams rtsp://...` | Stream metadata from client |
| `ffmpeg -stats ... -f null -` | Real-time decode FPS from client |

---

## JPEG encoder group contention

On Ingenic SoCs, JPEG encoder channels share encoder groups with H.264.
Active JPEG encoding forces IDR resets on the H.264 channel, which on
T20 (oldest/slowest VPU) halves the main stream from 28+ fps to ~14 fps.

### Symptoms

- H.264 FPS drops when MJPEG viewer or snapshot consumer connects
- FPS recovers when MJPEG viewer disconnects
- `ringdump jpeg0` shows JPEG frames being produced at sensor FPS (not
  the configured `[jpeg] fps` — the T20 VPU ignores per-channel fps
  for JPEG in shared encoder groups)

### Solution: JPEG idle mode (default)

`[jpeg] idle = true` (default in raptor.conf) enables on-demand JPEG
encoding. The JPEG encoder only runs when a consumer actively acquires
the JPEG ring (MJPEG streaming, snapshot request, ringdump).

RVD logs show the lifecycle:
```
jpeg chn 2: started (1 consumers)     # consumer acquired
jpeg chn 2: stopped (no consumers)    # consumer released
```

### Debugging stuck JPEG encoder

If `ringdump jpeg0` shows `Readers: N` with N > 0 but no active viewers:

```sh
ringdump jpeg0    # check Readers and PIDs fields
```

- **PIDs shown**: a consumer process holds the ring. Check if it's alive
  with `kill -0 <pid>`. If dead, the reap will reconcile within 10s.
- **PIDs all zero but Readers > 0**: orphaned count. The reap reconcile
  will reset it within 10s.
- **Workaround**: restart RVD to recreate rings with clean state.

### Disabling JPEG idle

Set `idle = false` in `[jpeg]` section of raptor.conf for always-on
JPEG encoding (legacy behavior). The JPEG encoder starts at pipeline
init and runs continuously regardless of consumers.

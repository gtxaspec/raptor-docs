# Raptor Streaming System -- Test Strategy

Phased testing plan: hardware-first on T31, x86 stub for IPC and
integration, then additional SoC bring-up. Validation tools and
performance benchmarks.

---

## 1. Phase 1: T31 on Hardware

T31 is the primary development target. All tests run on a T31-based
camera (e.g., Wyze Cam v3) with thingino firmware. Daemons are deployed
via NFS or flashed to firmware.

### 1.1 Minimal Test: RVD Only

**Goal**: Verify the HAL video pipeline produces valid encoded frames.

```sh
# Start RVD in foreground with debug logging
rvd -c /etc/raptor.json -f -v

# In another terminal, dump one encoded frame
raptorctl rvd dump-frame > /tmp/frame.h264

# Verify NAL structure
# For H264 IDR: expect SPS (0x67) + PPS (0x68) + IDR (0x65)
xxd /tmp/frame.h264 | head -20
```

Validation checklist:
- [ ] RVD starts without error (HAL init, sensor detect, pipeline bind)
- [ ] SHM ring `/dev/shm/rss_ring_main` created with correct header
  (magic=0x52535352, version=1, stream_id=0)
- [ ] SHM ring `/dev/shm/rss_ring_sub` created (stream_id=1)
- [ ] `dump-frame` output contains valid H264 NAL units
- [ ] IDR frame contains SPS+PPS+IDR (parse with `ffprobe -show_packets`)
- [ ] Non-IDR frames contain slice NALs
- [ ] Frame timestamp is monotonically increasing
- [ ] Sequence numbers increment without gaps
- [ ] RVD CPU usage: <15% at 1080p@25fps (measured via `top`)
- [ ] RVD RSS: <1MB (excluding SHM ring mmap)

```sh
# Detailed NAL analysis
ffprobe -show_packets -select_streams v \
    -show_entries packet=pts_time,flags,size \
    /tmp/frame.h264
```

### 1.2 RTSP Test: RVD + RSD

**Goal**: Verify end-to-end video streaming over RTSP.

```sh
# Start RVD and RSD
rvd -c /etc/raptor.json &
rsd -c /etc/raptor.json &

# From a remote machine, play the stream
ffplay rtsp://camera:554/main
ffplay rtsp://camera:554/sub
```

Validation checklist:
- [ ] RSD opens SHM rings without error
- [ ] RTSP DESCRIBE returns valid SDP (video track with correct codec)
- [ ] ffplay displays live video without artifacts
- [ ] Stream plays for 10+ minutes without dropping
- [ ] Multiple simultaneous clients (up to `max_clients` in config)
- [ ] Client disconnect does not crash RSD
- [ ] RSD reconnects to ring if RVD is restarted
- [ ] Sub stream plays at correct resolution (640x360)

```sh
# Record 30 seconds to file for offline analysis
ffmpeg -rtsp_transport tcp -i rtsp://camera:554/main \
    -c copy -t 30 /tmp/test_main.mp4

# Verify stream integrity
ffprobe -show_format -show_streams /tmp/test_main.mp4
```

### 1.3 Audio Test: RVD + RAD + RSD

**Goal**: Verify audio capture, encoding, and RTSP delivery.

```sh
# Start all three
rvd -c /etc/raptor.json &
rad -c /etc/raptor.json &
rsd -c /etc/raptor.json &

# Play with audio
ffplay rtsp://camera:554/main
```

Validation checklist:
- [ ] RAD creates audio SHM ring `/dev/shm/rss_ring_audio`
- [ ] RSD SDP includes audio track (a=rtpmap with correct codec)
- [ ] Audio plays in sync with video (lip sync within 100ms)
- [ ] Audio quality acceptable (no clipping, no severe noise)
- [ ] Audio continues after RAD restart (RSD reattaches to new ring)
- [ ] `raptorctl rad status` shows frame count, sample rate, codec

```sh
# Record audio+video for sync analysis
ffmpeg -rtsp_transport tcp -i rtsp://camera:554/main \
    -c copy -t 10 /tmp/test_av.mp4

# Check A/V sync
ffprobe -show_entries stream=codec_type,start_time \
    /tmp/test_av.mp4
```

### 1.4 OSD Test: RVD + ROD

**Goal**: Verify OSD timestamp overlay renders correctly in the stream.

```sh
# Start RVD and ROD
rvd -c /etc/raptor.json &
rod -c /etc/raptor.json &
rsd -c /etc/raptor.json &

# View stream and verify timestamp overlay
ffplay rtsp://camera:554/main
```

Validation checklist:
- [ ] ROD creates OSD SHM double-buffers
- [ ] ROD connects to RVD and registers OSD regions
- [ ] Timestamp overlay visible at configured position (x=10, y=10)
- [ ] Timestamp updates every `update_interval_ms` (visually confirm)
- [ ] Camera name overlay visible at configured position
- [ ] OSD text renders cleanly (no garbled glyphs, correct font size)
- [ ] Killing ROD freezes the overlay (does not crash RVD)
- [ ] Restarting ROD resumes overlay updates
- [ ] `raptorctl rod set-text timestamp "TEST %H:%M"` updates format

### 1.5 Recording Test: RVD + RMR

**Goal**: Verify MP4 file recording to SD card.

```sh
# Mount SD card
mount /dev/mmcblk0p1 /mnt/sd

# Start RVD and RMR
rvd -c /etc/raptor.json &
rmr -c /etc/raptor.json &

# Start recording
raptorctl rmr start

# Wait 60 seconds, then stop
sleep 60
raptorctl rmr stop
```

Validation checklist:
- [ ] MP4 file created in configured directory (`/mnt/sd/recordings/`)
- [ ] File is playable with ffplay/VLC without re-muxing
- [ ] moov atom present and valid (`ffprobe -show_format`)
- [ ] Video track matches configured codec and resolution
- [ ] Audio track present (if RAD is running)
- [ ] File segments at configured interval (`segment_minutes`)
- [ ] Recording survives RMR restart (new file, old file intact)
- [ ] SD card removal during recording handled gracefully (error logged,
  daemon continues, resumes when SD reinserted)

```sh
# Verify recording integrity
ffprobe -show_format -show_streams /mnt/sd/recordings/*.mp4
```

### 1.6 Full Stack Test

**Goal**: All daemons running simultaneously, sustained operation.

```sh
# Start all daemons
/etc/init.d/S31raptor start

# Monitor for 1 hour
watch -n 5 raptorctl status
```

Validation checklist:
- [ ] All daemons running after 1 hour
- [ ] No memory leaks (`/proc/<pid>/status` VmRSS stable)
- [ ] No file descriptor leaks (`ls /proc/<pid>/fd | wc -l` stable)
- [ ] RTSP clients can connect/disconnect repeatedly
- [ ] Recording produces valid files over multiple segments
- [ ] Day/night transition works (RIC switches IR-cut, ISP mode)
- [ ] `raptorctl status` shows all daemons healthy
- [ ] System CPU idle >50% with one RTSP client + recording

---

## 2. Phase 2: x86 Stub

All IPC and protocol testing on a development workstation. No hardware
required. Uses the stub HAL that generates synthetic frames.

### 2.1 Stub HAL Verification

```sh
cd build-x86

# Start RVD with stub HAL
./daemons/rvd/rvd -c ../config/raptor.json -f -v &

# Dump a frame -- should contain synthetic H264 NAL
./tools/raptorctl/raptorctl rvd dump-frame > /tmp/stub_frame.h264
hexdump -C /tmp/stub_frame.h264 | head

# Verify ring exists
ls -la /dev/shm/rss_ring_*
```

### 2.2 IPC: Multi-Consumer Ring Test

**Goal**: Verify multiple consumers can read from the same SHM ring
concurrently without data corruption.

```sh
# Start producer
./daemons/rvd/rvd -c ../config/raptor.json &

# Start 3 consumers on the same ring
./daemons/rsd/rsd -c ../config/raptor.json &
./daemons/rmr/rmr -c ../config/raptor.json &
./tools/ringdump/ringdump -r main -n 100 &   # dump 100 frames and exit

# Verify all 3 consumed frames without errors
wait
```

Validation checklist:
- [ ] All consumers read frames without corruption
- [ ] Sequence numbers are consistent across consumers
- [ ] No consumer blocks the producer
- [ ] Slow consumer receives `RSS_EOVERFLOW` and recovers

### 2.3 IPC: Ring Overflow Test

**Goal**: Verify behavior when a consumer falls behind.

```sh
# Start producer at high rate
RSS_STUB_FPS=60 ./daemons/rvd/rvd -c ../config/raptor.json &

# Start a deliberately slow consumer
./tools/ringdump/ringdump -r main -n 1000 --delay-ms 200 &

# Verify overflow detection and recovery
# ringdump should log "overflow: skipped N frames, seeking next keyframe"
```

### 2.4 OSD Double-Buffer Test

```sh
# Start RVD and ROD
./daemons/rvd/rvd -c ../config/raptor.json &
./daemons/rod/rod -c ../config/raptor.json &

# Verify OSD SHM created
ls -la /dev/shm/rss_osd_*

# Kill ROD, verify RVD continues
kill $(pidof rod)
sleep 2
./tools/raptorctl/raptorctl rvd status   # should show "running"

# Restart ROD
./daemons/rod/rod -c ../config/raptor.json &
```

### 2.5 Control Socket Test

```sh
# Start RVD
./daemons/rvd/rvd -c ../config/raptor.json &

# Test various commands
./tools/raptorctl/raptorctl rvd status
./tools/raptorctl/raptorctl rvd set-bitrate 3000000
./tools/raptorctl/raptorctl rvd request-idr
./tools/raptorctl/raptorctl rvd set-gop 100

# Test error handling
./tools/raptorctl/raptorctl rvd set-bitrate -1      # expect error
./tools/raptorctl/raptorctl nonexistent status       # expect error
```

### 2.6 RTSP on x86 (Synthetic Stream)

```sh
./daemons/rvd/rvd -c ../config/raptor.json &
./daemons/rsd/rsd -c ../config/raptor.json &

# Play synthetic stream -- should show color bars
ffplay rtsp://localhost:554/main
```

This validates the full RSD RTSP/RTP stack without hardware.

### 2.7 Crash Recovery Simulation

```sh
# Start all daemons
./daemons/rvd/rvd -c ../config/raptor.json &
./daemons/rsd/rsd -c ../config/raptor.json &
./daemons/rod/rod -c ../config/raptor.json &

# Simulate RVD crash
kill -9 $(pidof rvd)
sleep 1

# Verify consumers detect stale ring
./tools/raptorctl/raptorctl rsd status   # should show "ring stale" or similar

# Restart RVD
./daemons/rvd/rvd -c ../config/raptor.json &
sleep 2

# Verify consumers reconnect
./tools/raptorctl/raptorctl rsd status   # should show "connected"
ffplay rtsp://localhost:554/main          # should resume
```

---

## 3. Phase 3: Additional SoCs

### 3.1 T20 (Old SDK, No H265)

**Test platform**: T20-based camera (e.g., Wyze Cam v2).

Key differences from T31:
- No H265 encoder (`caps.has_h265 = false`)
- No buffer sharing (`caps.has_bufshare = false`)
- No capped rate control (`caps.has_capped_rc = false`)
- No dynamic bitrate (`caps.has_set_bitrate = false`)
- Old SDK enc_get_frame data layout (direct virAddr, no ring buffer wrap)

Validation checklist:
- [ ] RVD starts with H264-only config (no H265 attempt)
- [ ] Config requesting H265 falls back to H264 with warning log
- [ ] `raptorctl rvd set-bitrate` returns error (unsupported) gracefully
- [ ] RC mode CAPPED_VBR in config silently maps to VBR
- [ ] Frame data from enc_get_frame is correctly linearized (old layout)
- [ ] RTSP stream plays in ffplay
- [ ] All consumer daemons work (they are codec-agnostic)

```sh
# Test graceful degradation
cat > /tmp/raptor-h265.json << 'EOF'
{"rvd": {"streams": [{"codec": "h265", "width": 1280, "height": 720}]}}
EOF
rvd -c /tmp/raptor-h265.json -f -v
# Expected: "H265 not supported on T20, falling back to H264"
```

### 3.2 T40 (IMPVI_SDK, Dual-Core)

**Test platform**: T40-based camera with dual MIPI CSI sensors.

Key differences from T31:
- Dual XBurst cores (CPU affinity testing)
- IMPVI_NUM parameter on all ISP tuning calls
- Hardware I2D rotation
- Multi-sensor support (dual CSI)

Validation checklist:
- [ ] CPU affinity: RVD on CPU0, consumers on CPU1
  ```sh
  taskset -p $(pidof rvd)    # should show 0x1 (CPU0)
  taskset -p $(pidof rsd)    # should show 0x2 (CPU1)
  ```
- [ ] ISP tuning calls dispatch with IMPVI_NUM correctly
- [ ] Hardware rotation: `fs_set_rotation(90)` works without CPU overhead
- [ ] Dual sensor: both sensors produce independent streams
  ```sh
  ls /dev/shm/rss_ring_main0 /dev/shm/rss_ring_main1
  ffplay rtsp://camera:554/main0
  ffplay rtsp://camera:554/main1
  ```
- [ ] CPU usage lower than T31 at equivalent resolution (dual-core benefit)
- [ ] Day/night transition on both sensors independently (RIC)

### 3.3 T32 (Extended Encoder)

**Test platform**: T32-based camera.

Key differences from T31:
- Hybrid SDK: old-style type names, new-style internal structs
- `SetDefaultParam` takes extra `uBufSize` parameter
- `IMP_System_MemPoolFree` in deinit (unique to T32)
- Extended encoder features

Validation checklist:
- [ ] enc_create_channel correctly fills hybrid struct format
- [ ] SetDefaultParam called with uBufSize (T32-specific code path)
- [ ] MemPoolFree called during deinit
- [ ] Extended encoder features disabled gracefully (caps reflect T32 support)
- [ ] RTSP stream plays correctly
- [ ] All rate control modes work (VBR, CBR, CAPPED_VBR)

### 3.4 T23 (Dual-Sensor via _Sec)

**Test platform**: T23-based camera.

Key differences:
- Dual-sensor via `_Sec` suffix functions (not IMPVI_NUM)
- ISP OSD support (`caps.has_isp_osd = true`)
- Frame-level crop support (`caps.has_fcrop = true`)
- Audio HPF cutoff frequency tuning

Validation checklist:
- [ ] Second sensor accessible via `_Sec` API path
- [ ] ISP OSD renders correctly
- [ ] fcrop configuration works (verify cropped output resolution)
- [ ] Audio HPF cutoff adjustment via raptorctl

---

## 4. Validation Tools

### 4.1 ffprobe -- Stream Analysis

```sh
# Analyze RTSP stream parameters
ffprobe -show_format -show_streams \
    -rtsp_transport tcp \
    rtsp://camera:554/main

# Packet-level analysis (NAL types, sizes, timestamps)
ffprobe -show_packets -select_streams v \
    -show_entries packet=codec_type,pts_time,dts_time,size,flags \
    -rtsp_transport tcp \
    rtsp://camera:554/main

# Detect IDR frame interval (GOP verification)
ffprobe -show_frames -select_streams v \
    -show_entries frame=pict_type,key_frame,pts_time \
    -rtsp_transport tcp -t 10 \
    rtsp://camera:554/main 2>&1 | grep key_frame=1
```

### 4.2 Valgrind -- Memory Leak Detection (x86 Stub)

```sh
# Full leak check on RVD (run for 60 seconds then SIGTERM)
valgrind --leak-check=full --show-leak-kinds=all \
    --track-origins=yes --track-fds=yes \
    --log-file=/tmp/rvd-valgrind.log \
    ./rvd -c raptor.json &

VG_PID=$!
sleep 60
kill -TERM $VG_PID
wait $VG_PID

# Review results
grep -E "LEAK SUMMARY|ERROR SUMMARY" /tmp/rvd-valgrind.log
```

Expected: zero leaked bytes, zero errors.

```sh
# Helgrind for thread-safety (race condition detection)
valgrind --tool=helgrind --log-file=/tmp/rvd-helgrind.log \
    ./rvd -c raptor.json &

# Run consumers concurrently
./rsd -c raptor.json &
./rmr -c raptor.json &
sleep 30
kill $(pidof rvd rsd rmr)
```

### 4.3 strace -- IPC Debugging

```sh
# Trace SHM operations
strace -e shm_open,mmap,munmap,shm_unlink -f \
    ./rvd -c raptor.json

# Trace Unix socket operations
strace -e socket,bind,listen,accept,connect,sendto,recvfrom -f \
    ./rsd -c raptor.json

# Trace eventfd operations (OSD notification)
strace -e eventfd2,read,write -p $(pidof rvd)
```

### 4.4 ringdump -- SHM Ring Inspector

Custom debug tool built as part of the raptor tree:

```sh
# Dump ring header
ringdump -r main --header

# Output:
# Ring: /dev/shm/rss_ring_main
# Magic: 0x52535352 (valid)
# Version: 1
# Slots: 16
# Data size: 3145728
# Write seq: 12847
# Stream: 0 (main)
# Codec: H264
# Resolution: 1920x1080
# FPS: 25/1

# Dump last 5 frames metadata
ringdump -r main -n 5 --meta

# Output:
# seq=12843 offset=1048576 len=84732 ts=1679846123456789 type=H264_SLICE key=0
# seq=12844 offset=1133308 len=12451 ts=1679846123496789 type=H264_SLICE key=0
# seq=12845 offset=1145759 len=9823  ts=1679846123536789 type=H264_SLICE key=0
# seq=12846 offset=1155582 len=187234 ts=1679846123576789 type=H264_IDR key=1
# seq=12847 offset=1342816 len=8234  ts=1679846123616789 type=H264_SLICE key=0

# Dump one frame payload to file
ringdump -r main -n 1 --payload > /tmp/frame.h264

# Continuous monitoring (print stats every second)
ringdump -r main --monitor
```

### 4.5 On-Target Quick Checks

```sh
# Check all SHM rings exist
ls -la /dev/shm/rss_ring_*

# Check control sockets exist
ls -la /var/run/rss/*.sock

# Check daemon PIDs
for d in rvd rad rod rsd rmr ric; do
    pid=$(pidof $d 2>/dev/null)
    if [ -n "$pid" ]; then
        rss=$(awk '/VmRSS/{print $2}' /proc/$pid/status)
        fds=$(ls /proc/$pid/fd 2>/dev/null | wc -l)
        echo "$d: pid=$pid rss=${rss}kB fds=$fds"
    else
        echo "$d: not running"
    fi
done

# Quick RTSP test
wget -q -O /dev/null --timeout=5 \
    "rtsp://localhost:554/main" 2>&1 && echo "RTSP OK" || echo "RTSP FAIL"
```

---

## 5. Performance Benchmarks

### 5.1 CPU Usage

Measured on T31 @ 1.5GHz XBurst with 1080p@25fps H264 VBR 2Mbps.

| Daemon | Idle (no clients) | 1 RTSP Client | 2 RTSP Clients | Recording |
|--------|-------------------|---------------|-----------------|-----------|
| RVD | ~8% | ~8% | ~8% | ~8% |
| RAD | ~2% | ~2% | ~2% | ~2% |
| ROD | ~1% | ~1% | ~1% | ~1% |
| RSD | ~0% | ~5% | ~8% | N/A |
| RMR | N/A | N/A | N/A | ~3% |
| RIC | <1% | <1% | <1% | <1% |
| **Total** | **~12%** | **~17%** | **~20%** | **~15%** |

RVD CPU is constant regardless of consumers because consumers read
from SHM without producer involvement.

Measurement method:
```sh
# 10-second CPU sampling per daemon
for d in rvd rad rod rsd rmr ric; do
    pid=$(pidof $d)
    [ -z "$pid" ] && continue
    top -b -n 10 -d 1 -p $pid | awk '/'"$d"'/{sum+=$9; n++} END{print "'$d':", sum/n "%"}'
done
```

### 5.2 Latency: Frame Capture to RTP Send

End-to-end latency from sensor capture to first RTP packet on the wire.

| Stage | Latency (ms) | Notes |
|-------|-------------|-------|
| Sensor exposure + readout | ~33 (1/30fps) | Fixed by frame rate |
| ISP processing | ~5 | Hardware pipeline |
| Encoder | ~10 | Hardware H264 encoder |
| RVD: enc_get_frame + ring_publish | <1 | Memory copy only |
| RSD: ring_read + RTP packetize | <1 | Zero-copy + sendto() |
| **Total (glass-to-glass)** | **~50** | Excluding network transit |

Measurement method:
```sh
# Embed frame timestamp in OSD, capture with known-latency display
# Compare OSD clock to wall clock

# Alternative: measure RVD->RSD latency via timestamps
# RVD logs: "published seq=N ts=T1"
# RSD logs: "sent RTP seq=N ts=T2"
# Latency = T2 - T1
rvd -c raptor.json -f -v 2>&1 | grep "published" &
rsd -c raptor.json -f -v 2>&1 | grep "sent RTP" &
```

### 5.3 Memory Usage

Measured on T31 with full stack (RVD + RAD + ROD + RSD + RMR + RIC).

```sh
# Snapshot memory usage
echo "=== Per-daemon RSS ==="
for d in rvd rad rod rsd rmr ric; do
    pid=$(pidof $d)
    [ -z "$pid" ] && continue
    rss=$(awk '/VmRSS/{print $2}' /proc/$pid/status)
    vsz=$(awk '/VmSize/{print $2}' /proc/$pid/status)
    echo "$d: RSS=${rss}kB VSZ=${vsz}kB"
done

echo "=== SHM Rings ==="
du -h /dev/shm/rss_ring_*

echo "=== System Memory ==="
free -k
cat /proc/meminfo | grep -E "MemTotal|MemFree|MemAvailable|Buffers|Cached"
```

Target budgets (T31, 64MB):
```
Kernel + drivers:     ~12MB
RVD + SHM rings:       ~4MB
RAD + SHM ring:       ~0.5MB
ROD + OSD SHM:        ~0.8MB
RSD:                  ~0.5MB
RMR:                  ~0.4MB
RIC:                  ~0.1MB
Encoder DMA (rmem):   ~16MB  (reserved, not in /proc/meminfo MemTotal)
────────────────────────────
Used:                 ~18MB
Available:            ~46MB  (out of 64MB visible to Linux)
Headroom:             ~28MB  (for filesystem cache, tmpfs, user processes)
```

### 5.4 Throughput

Maximum sustained frame rate before frame drops, per SoC:

| SoC | Max Main Stream | Max Sub Stream | Max Audio |
|-----|----------------|----------------|-----------|
| T20 | 1280x720@30fps H264 | 640x360@15fps H264 | 16kHz G711a |
| T31 | 1920x1080@30fps H264 | 640x360@25fps H264 | 16kHz AAC |
| T31 | 1920x1080@25fps H265 | 640x360@25fps H264 | 16kHz AAC |
| T40 | 2560x1440@25fps H265 | 640x360@25fps H264 | 48kHz AAC |
| T40 | 2x 1920x1080@25fps H264 (dual sensor) | 2x 640x360@15fps | 16kHz |

### 5.5 Startup Time

Time from `S31raptor start` to first RTSP-playable frame:

| Stage | Duration (ms) | Notes |
|-------|-------------|-------|
| RVD process spawn | ~50 | fork + exec |
| HAL init (sensor detect) | ~800 | I2C probe, ISP init |
| Pipeline create + bind | ~200 | FS + OSD + Enc create |
| First encoded frame | ~300 | Sensor stream start + first IDR |
| RSD process spawn + ring open | ~100 | Parallel with above |
| **Total to first RTSP frame** | **~1500** | |

Measurement:
```sh
# Timestamp before and after
T0=$(date +%s%N)
/etc/init.d/S31raptor start
# Wait for first frame
while ! raptorctl rvd status 2>/dev/null | grep -q "frames:"; do
    usleep 10000
done
T1=$(date +%s%N)
echo "Startup: $(( (T1 - T0) / 1000000 ))ms"
```

---

## 6. Continuous Integration

### 6.1 x86 Stub CI (Every Commit)

Run on any x86 CI runner (GitHub Actions, local Jenkins):

```sh
# Build
cmake -DRSS_SOC=STUB -DCMAKE_BUILD_TYPE=Debug -DRSS_SANITIZERS=ON ..
make -j$(nproc)

# Unit tests (librss_ipc)
ctest --test-dir lib/librss_ipc/tests --output-on-failure

# Integration: start daemons, verify SHM, stop
./daemons/rvd/rvd -c ../config/raptor.json &
sleep 2
test -e /dev/shm/rss_ring_main || exit 1
./tools/raptorctl/raptorctl rvd status | grep -q "running" || exit 1
./tools/raptorctl/raptorctl rvd dump-frame > /dev/null || exit 1
kill $(pidof rvd)

# Valgrind (optional, slower)
valgrind --leak-check=full --error-exitcode=1 \
    timeout 10 ./daemons/rvd/rvd -c ../config/raptor.json
```

### 6.2 Cross-Build CI (Every Commit)

Verify the project cross-compiles for all supported SoCs:

```sh
for soc in T20 T21 T23 T30 T31 T32 T40 T41; do
    mkdir -p build-$soc && cd build-$soc
    cmake -DCMAKE_TOOLCHAIN_FILE=../cmake/mips-linux-gnu.cmake \
          -DRSS_SOC=$soc ..
    make -j$(nproc) || exit 1
    cd ..
done
```

### 6.3 Hardware CI (Nightly)

On a T31 test camera connected to a CI agent:

```sh
# Deploy via NFS
scp build-T31/daemons/*/rvd build-T31/daemons/*/rsd \
    root@camera:/usr/bin/

# Run test suite
ssh root@camera '/etc/init.d/S31raptor start'
sleep 5

# RTSP smoke test
ffprobe -rtsp_transport tcp -timeout 5000000 \
    rtsp://camera:554/main 2>&1 | grep -q "Video: h264" || exit 1

# Cleanup
ssh root@camera '/etc/init.d/S31raptor stop'
```

---

## 7. ASan / UBSan Host Testing

`build-asan.sh` builds all daemons for the x86_64 host with
AddressSanitizer and UndefinedBehaviorSanitizer enabled. This is the
primary memory-safety regression workflow and does not require hardware.

### 7.1 How It Works

- **RVD and RAD** use a mock HAL (`tests/mock_hal.c`) that implements
  the full `rss_hal_ops_t` vtable with deterministic synthetic frames.
  This replaces the real Ingenic SDK calls so the daemons can be linked
  natively on x86.
- **RSD, RHD, ROD, RIC** have no HAL dependency and build natively
  without any mock.
- All daemons are compiled with `-fsanitize=address,undefined
  -fno-omit-frame-pointer`.

### 7.2 Dummy Ring Setup

`tests/create_rings.c` is a small utility that creates SHM rings in
`/dev/shm/` pre-populated with fake JPEG frames. This allows RHD and
ringdump to exercise their buffer paths without RVD running.

### 7.3 Running

```sh
# Build all daemons with ASan
./build-asan.sh

# Create dummy SHM rings (JPEG ring with fake frames)
./asan-out/create_rings &

# Start daemons
./asan-out/rvd -c config/raptor.json &
./asan-out/rad -c config/raptor.json &
./asan-out/rsd -c config/raptor.json &
./asan-out/rhd -c config/raptor.json &
./asan-out/rod -c config/raptor.json &
./asan-out/ric -c config/raptor.json &

# Exercise with clients
for i in $(seq 1 100); do curl -s http://localhost:8080/snap.jpg > /dev/null; done
for i in $(seq 1 10); do ffprobe -rtsp_transport tcp rtsp://localhost:554/main 2>/dev/null; done

# Stop all daemons -- ASan reports memory leaks and UB on exit
kill $(pidof rvd rad rsd rhd rod ric create_rings)
wait
```

ASan writes its report to stderr. A clean run shows:

```
==PID==ERROR: LeakSanitizer: detected memory leaks
... (zero leaks reported)
```

### 7.4 Stress Test Results

The full suite has been stress tested with:
- 4000 HTTP requests to RHD (`/snap.jpg` and `/mjpeg`)
- 400 RTSP connection cycles to RSD

Across all 6 daemons simultaneously, the result is zero memory leaks
and zero UBSan errors reported by ASan at exit.

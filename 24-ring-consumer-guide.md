# Ring Buffer Consumer Guide

How to read live video and audio frames from Raptor's SHM ring
buffers using `librss_ipc`. This is the native C API that all
Raptor consumer daemons (RSD, RHD, RWD, RMR, RWC) use internally.

---

## 1. Ring Names

Ring buffers live in `/dev/shm/` with the prefix `rss_ring_`. The
`rss_ring_open()` call takes the short name (without prefix).

| Ring name | Producer | Content |
|-----------|----------|---------|
| `main` | RVD | Primary video stream (H.264 or H.265) |
| `sub` | RVD | Secondary video stream |
| `jpeg0` | RVD | JPEG snapshot stream |
| `audio` | RAD | Audio (L16, G.711, AAC, or Opus) |
| `s1_main` | RVD | Sensor 1 main (multi-sensor only) |
| `s1_sub` | RVD | Sensor 1 sub (multi-sensor only) |
| `s1_jpeg0` | RVD | Sensor 1 JPEG (multi-sensor only) |
| `speaker` | RWD | Backchannel audio (written by WebRTC clients) |

RFS (file source) creates the same ring names when acting as the
video source on A1 or for file playback.

---

## 2. Minimal Reader Example

```c
#include <stdio.h>
#include <stdlib.h>
#include <signal.h>
#include "rss_ipc.h"

static volatile sig_atomic_t running = 1;

static void on_signal(int sig)
{
	(void)sig;
	running = 0;
}

int main(int argc, char **argv)
{
	const char *ring_name = (argc > 1) ? argv[1] : "main";

	signal(SIGINT, on_signal);
	signal(SIGTERM, on_signal);

	rss_ring_t *ring = rss_ring_open(ring_name);
	if (!ring) {
		fprintf(stderr, "ring '%s' not found\n", ring_name);
		return 1;
	}

	if (!rss_ring_check_version(ring, ring_name)) {
		rss_ring_close(ring);
		return 1;
	}

	const rss_ring_header_t *hdr = rss_ring_get_header(ring);
	fprintf(stderr, "ring: %s  codec=%u  %ux%u  %u/%u fps\n",
		ring_name, hdr->codec, hdr->width, hdr->height,
		hdr->fps_num, hdr->fps_den);

	uint32_t buf_size = rss_ring_max_frame_size(ring);
	uint8_t *buf = malloc(buf_size);
	if (!buf) {
		rss_ring_close(ring);
		return 1;
	}

	rss_ring_acquire(ring);

	uint64_t read_seq = 0;
	while (running) {
		if (rss_ring_wait(ring, 100) != 0)
			continue;

		uint32_t length;
		rss_ring_slot_t meta;
		int ret = rss_ring_read(ring, &read_seq, buf, buf_size,
					&length, &meta);
		if (ret == -EAGAIN)
			continue;
		if (ret == RSS_EOVERFLOW) {
			fprintf(stderr, "overflow at seq %llu, re-syncing\n",
				(unsigned long long)read_seq);
			rss_ring_request_idr(ring);
			continue;
		}
		if (ret != 0)
			continue;

		/* Process frame: buf contains `length` bytes.
		 * meta.is_key    -- 1 if keyframe / IDR
		 * meta.timestamp -- capture time (microseconds, monotonic)
		 * meta.nal_type  -- NAL type (video) or codec ID (audio)
		 * meta.seq       -- monotonic frame sequence number
		 */
		fprintf(stderr, "seq=%llu len=%u key=%d ts=%lld\n",
			(unsigned long long)meta.seq, length, meta.is_key,
			(long long)meta.timestamp);

		/* Example: write raw frame data to stdout */
		fwrite(buf, 1, length, stdout);
	}

	rss_ring_release(ring);
	free(buf);
	rss_ring_close(ring);
	return 0;
}
```

### Build

```sh
# Cross-compile for device (using standalone build deps)
XB=.deps/toolchain/bin/mipsel-linux-
$XB-gcc -Os -o ring_reader ring_reader.c \
    -I../raptor-ipc/include \
    -L../raptor-ipc -lrss_ipc -lpthread -lrt
```

Or link against the static library from a standalone build:

```sh
$XB-gcc -Os -o ring_reader ring_reader.c \
    -I../raptor-ipc/include \
    ../raptor-ipc/librss_ipc.a -lpthread -lrt
```

---

## 3. API Reference

### Opening and Closing

```c
rss_ring_t *rss_ring_open(const char *name);
void        rss_ring_close(rss_ring_t *ring);
```

`rss_ring_open` attaches to an existing SHM ring by name. Returns
NULL if the ring doesn't exist (producer hasn't created it yet).
A consumer can retry in a loop until the producer starts.

### Reading Frames

```c
int rss_ring_read(rss_ring_t *ring, uint64_t *read_seq,
                  uint8_t *dest, uint32_t dest_size,
                  uint32_t *length, rss_ring_slot_t *meta);
```

Copy-on-read: copies frame data into `dest`, then re-validates that
the slot wasn't recycled during the copy. This is the safe API for
consumers that process frames at their own pace.

| Return | Meaning |
|--------|---------|
| `0` | Success. `length` and `meta` are filled. |
| `-EAGAIN` | No new data since last read. |
| `RSS_EOVERFLOW` | Consumer fell behind. `read_seq` is advanced to the oldest available slot. Call `rss_ring_request_idr()` to get a keyframe. |
| `-ENOSPC` | Frame is larger than `dest_size`. |

`read_seq` is caller-owned state. Initialize to 0 on first read.
The ring advances it automatically.

### Zero-Copy Read

```c
int rss_ring_peek(rss_ring_t *ring, uint64_t *read_seq,
                  const uint8_t **data_ptr, uint32_t *length,
                  rss_ring_slot_t *meta);
int rss_ring_peek_done(rss_ring_t *ring, const rss_ring_slot_t *meta);
```

Returns a pointer directly into the ring's backing memory. The
pointer is valid until `rss_ring_peek_done()` is called.
`peek_done` re-validates that the producer didn't overwrite the
buffer — returns `RSS_EOVERFLOW` if it did (data may be corrupt).

Use peek for low-latency paths where you can process the frame
immediately (e.g., RTP packetization, USB gadget writes).

### Waiting for Data

```c
int rss_ring_wait(rss_ring_t *ring, uint32_t timeout_ms);
```

Blocks until the producer publishes a new frame or `timeout_ms`
expires. Returns 0 when data is available. Uses futex internally
— efficient, no polling.

### Demand Signaling

```c
void     rss_ring_acquire(rss_ring_t *ring);
void     rss_ring_release(rss_ring_t *ring);
uint32_t rss_ring_reader_count(rss_ring_t *ring);
```

`acquire` / `release` tell the producer that a consumer is actively
reading. Producers use `reader_count` to make on-demand decisions
(e.g., RVD skips JPEG encoding when no one is reading the JPEG ring).

Call `acquire` when you start reading, `release` when you stop.
`open` / `close` do **not** touch the demand counter.

### IDR Request

```c
void rss_ring_request_idr(rss_ring_t *ring);
```

Sets an atomic flag in the ring header. The video producer (RVD)
checks this flag in its encode loop and forces an IDR on the next
frame. Call this after `RSS_EOVERFLOW` so you don't have to wait
for the encoder's natural GOP cycle.

### Ring Metadata

```c
const rss_ring_header_t *rss_ring_get_header(rss_ring_t *ring);
uint32_t                 rss_ring_max_frame_size(rss_ring_t *ring);
```

`get_header` returns a pointer to the ring's header struct (see
`rss_ipc.h` for fields). Useful for reading stream info (codec,
resolution, FPS) without consuming frames.

`max_frame_size` returns the largest single frame the ring can
hold — use this to allocate your read buffer.

---

## 4. Stream Info

The ring header carries stream metadata set by the producer:

| Field | Description |
|-------|-------------|
| `stream_id` | `0`=main, `1`=sub, `0x10`=audio, `0x20`+=JPEG |
| `codec` | Video: `0`=H.264, `1`=H.265, `2`=JPEG. Audio: RTP PT (`0`=PCMU, `8`=PCMA, `11`=L16, `97`=AAC, `111`=Opus) |
| `width`, `height` | Video resolution (0 for audio) |
| `fps_num`, `fps_den` | Frame rate fraction (e.g., 25/1). Audio: sample rate in `fps_num`, 1 in `fps_den` |
| `profile`, `level` | H.264 profile/level IDC (0 for audio/JPEG) |

### Frame Metadata (per slot)

| Field | Description |
|-------|-------------|
| `seq` | Monotonic frame sequence number |
| `timestamp` | Capture timestamp in microseconds (IMP system clock, CLOCK_MONOTONIC_RAW) |
| `is_key` | 1 for IDR / keyframe |
| `nal_type` | NAL unit type for video, codec ID for audio |
| `data_offset` | Internal (offset into data region) |
| `data_length` | Frame payload size in bytes |

---

## 5. Overflow Recovery

The ring is a fixed-size circular buffer. If a consumer reads
slower than the producer writes, the producer overwrites old
frames. The consumer detects this via `RSS_EOVERFLOW` from
`rss_ring_read()`.

Recovery pattern:

1. `rss_ring_read()` returns `RSS_EOVERFLOW`.
2. `read_seq` is automatically advanced to the oldest available
   slot — no manual seeking needed.
3. Call `rss_ring_request_idr()` so the next frame is a keyframe.
4. Continue the read loop. Skip non-keyframes until `is_key == 1`
   if your downstream requires a clean decode start.

This is the same pattern all Raptor daemons use internally.

---

## 6. Reconnection

The producer (RVD, RAD) may restart — the ring is destroyed and
recreated with a new incarnation. Consumers detect this when
`rss_ring_wait()` or `rss_ring_read()` starts returning errors,
or when the ring's `incarnation` field changes.

Reconnection pattern (from RSD):

1. Detect ring loss (repeated `-EAGAIN` with no `write_seq`
   advancement, or ring disappears from `/dev/shm`).
2. `rss_ring_close(ring)`.
3. Retry `rss_ring_open()` in a loop with a sleep (200ms).
4. On success: reset `read_seq = 0`, re-read header for updated
   stream info (codec/resolution may have changed), reallocate
   frame buffer if `max_frame_size` grew.
5. Resume reading.

---

## 7. Reference Mode

On T31 and newer SoCs, RVD can enable reference mode (`refmode=true`
in config). In this mode, the encoder writes frame data directly to
`/dev/rmem` (physical memory), and the ring carries only metadata
pointers — no data copy on publish.

**Consumers don't need to change.** `rss_ring_read()` transparently
mmaps `/dev/rmem` and copies from it. `rss_ring_peek()` returns a
pointer into the mmap'd region. The consumer API is identical
regardless of whether refmode is enabled.

Check `ring->flags & RSS_RING_FLAG_REFMODE` if you need to know.

---

## 8. Video Frame Format

Video frames in the ring are **Annex B** format (start code
delimited: `00 00 00 01` prefix). For H.264, IDR frames include
SPS and PPS NALs before the slice. For H.265, IDR frames include
VPS, SPS, and PPS.

If your downstream needs AVCC (length-prefixed NALs), convert
using Raptor's NAL utilities in `raptor-common` or implement
start-code-to-length conversion yourself.

JPEG frames are complete JFIF images — each frame in the ring is
a standalone JPEG file (starts with `FF D8`, ends with `FF D9`).
No additional framing. RHD reads these for `/snap.jpg` and MJPEG
streaming.

Audio frames are raw codec output: PCM samples (big-endian for
L16), G.711 bytes, AAC ADTS frames, or Opus packets.

---

## 9. Threading

- `rss_ring_open` / `rss_ring_close` are not thread-safe for the
  same ring handle. Open one handle per thread.
- `rss_ring_read` / `rss_ring_peek` are safe to call concurrently
  from different handles on the same ring (SPMC design).
- `rss_ring_wait` blocks the calling thread. Use one reader thread
  per ring, or use `rss_ring_read` with `-EAGAIN` polling if you
  need to multiplex in a single thread.
- `rss_ring_acquire` / `rss_ring_release` use atomics and are
  thread-safe.

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
		uint32_t length;
		rss_ring_slot_t meta;

		/* Read first, wait only on -EAGAIN. rss_ring_wait() blocks on
		 * the futex changing from its value at call time, so waiting
		 * with frames still pending can only time out -- a wait-first
		 * loop never drains the final frames of a stream. */
		int ret = rss_ring_read(ring, &read_seq, buf, buf_size,
					&length, &meta);
		if (ret == -EAGAIN) {
			rss_ring_wait(ring, 100);
			continue;
		}
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
| `RSS_EOVERFLOW` | Consumer fell behind, **or a refmode validation failed** (the producer recycled the encoder buffer mid-copy, replaced the backing region, or the slot was lapped — one return code covers them all). `read_seq` is advanced to the newest slot. Call `rss_ring_request_idr()` to get a keyframe. |
| `-ENOSPC` | Frame is larger than `dest_size`. `read_seq` has **already advanced past the frame** and `*length` holds the needed size: grow the buffer and continue, so it stays a one-frame hiccup. An encoder restart (live set-resolution) can raise the per-buffer stride mid-life, so a buffer sized at ring open is not sized forever. Treating this as a generic error either re-reads the same frame forever (if you discard the advanced `read_seq`) or silently drops every frame (if you keep it) — both shipped as real bugs. |

### The refmode copy budget

In refmode the frame bytes live in the encoder's own output buffers,
and the producer recycles those after **pool-depth frame intervals**
— as few as two buffers at full resolution, i.e. a ~66 ms budget at
30 fps. A consumer must finish `rss_ring_read()`'s copy inside that
window or the post-copy validation rejects the frame. The rule that
follows: **nothing slow may run on the reader loop between reads.**
Copy the frame out and hand it to another thread; per-packet work
(packetization, encryption, a socket that can push back) belongs on
the other side of a bounded queue. rsd and rwd use a per-client send
queue; rsr rides libsrt's internal send buffer. A reader that sends
inline works on small streams and collapses on main-stream keyframes
— tens of milliseconds of work per frame — into a permanent
reject/IDR-request cycle that looks like a keyframe slideshow at the
receiver.


### Producer Restarts

A producer restart creates a **new shm file at the same name**. The old
mapping stays valid but frozen: `write_seq` never moves again, and
`rss_ring_wait()` blocks on a futex nothing will ever wake. Polling
alone cannot tell that apart from an idle encoder, so a consumer that
only watches for new frames waits forever -- in the field this showed
up as every new RTSP client completing signaling and then getting no
frames until an unrelated timeout cleared it.

Check `rss_ring_stale()` when a ring goes quiet, and close/reopen when
it returns true. Do not rely on an idle-close path that is gated on
having no active clients: one lingering client keeps it from ever
running.

The producer side has the mirror-image hazard. A handle bounds its
writes by the geometry **it** mapped, not by `data_size` in the shared
header -- another `rss_ring_create()` can enlarge that header while an
older producer still holds a smaller mapping, and trusting it walks off
the end. `rss_ring_publish*()` returns `-EPIPE` once the incarnation
shows this handle has been superseded; treat that as "stop publishing,
you are the old producer", not as a transient error.

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

A long-lived consumer that forgets this fails subtly, not loudly:
the producer sees zero demand and never encodes, the consumer sees
no writes and (if it idle-closes) detaches, and every layer above —
SETUP/PLAY signaling, session keepalive — keeps answering 200 while
zero frames flow. RSD's RTSP jpeg streams shipped with exactly this
bug (fixed in raptor 5fb0fc267e3f3e436d71525b2cd54357bb982bfd): its
ring reader open()ed without `acquire` and idle-closed after 2s
without writes, deadlocking against RVD's reader-gated JPEG pulse.
If your consumer serves downstream clients of its own, hold both the
`acquire` and the mapping for as long as any client is attached, and
idle-close only when your own demand is gone.

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
| `src_seq` | The SOURCE's frame counter (encoder output sequence), stamped by the producer since ring version 5. `RSS_SRC_SEQ_NONE` (UINT32_MAX) on frames with no counter: audio, JPEG channels, file producers |

### Source-loss accounting (ring v5)

`write_seq` counts frames the ring published, so a `write_seq` gap can
only ever mean ring overflow. `src_seq` is the encoder's own output
counter, so a gap between consecutive stamped frames is a frame the
encoder produced that never reached the ring. The producer maintains
the split in the header, readable through:

```c
typedef struct {
    uint32_t drop_source;    /* source frames never fetched by producer */
    uint32_t drop_publish;   /* fetched frames never published          */
    uint32_t src_seq_resets; /* sequence-domain changes (restarts)      */
    uint32_t src_seq_last;   /* last stamped src_seq, else NONE         */
} rss_ring_loss_t;
void rss_ring_get_loss(rss_ring_t *ring, rss_ring_loss_t *out);
```

Reconciliation for a stamped stream:
`source frames = published + drop_source + drop_publish`, with
`src_seq_resets` marking domain boundaries (encoder restart, counter
rewrap) where accounting rebases instead of recording a bogus jump.

Semantics, calibrated on hardware: the SDK counter counts what the
encoder *emits*, so deliberate rate shaping — an encoder fps divider,
rate-control decisions — creates no false loss; a healthy pipeline
holds all counters at zero indefinitely. Nonzero `drop_source` is
real, unexplained loss between the encoder and the producer. Counters
are per ring incarnation (they reset when the producer recreates the
ring). `rvd status` exposes them per stream; `ringdump <ring>` prints
them on the `Src seq:` line.

### Converting timestamps to UTC

`timestamp` runs on the producer's private media clock. Since ring
version 4 the producer publishes a clock mapping in the header
(seqlock-guarded, refreshed every 5 s so NTP steps propagate):

```c
/* Producer (RVD) */
void rss_ring_set_utc(rss_ring_t *ring, int64_t offset_us, uint8_t status);

/* Consumer: UTC(frame) = meta.timestamp + offset_us */
int rss_ring_get_utc(rss_ring_t *ring, int64_t *offset_us, uint8_t *status);
```

`rss_ring_get_utc()` returns 0 on success, `-ENOENT` when the
producer has not published a mapping (RFS playback, test harnesses —
treat as "no wall clock available"), or `-EAGAIN` if a concurrent
refresh persisted across the bounded retry (skip this frame, read
again on the next). `status` is the MISB ST 0603 time status byte:
`RSS_UTC_STATUS_LOCKED` (0x1F) when the producer's kernel clock is
NTP-disciplined, `RSS_UTC_STATUS_UNLOCKED` (0x9F) otherwise. Call it
per frame — it is a few atomic loads, and the offset changes as the
producer re-syncs. RMR and RSD use this to build the per-frame
ST 0604 timecode SEI.

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

Two things change WITHOUT a reconnect, because rvd reuses the ring
across an encoder restart (a live `set-resolution` closes nothing):

- **Stream info is live.** The codec/geometry/rate cluster is rewritten
  in place under a seqlock. A consumer that polls it (rate pacing, SDP
  invalidation) must read through `rss_ring_get_stream_info()` — a
  direct header read can catch the new width beside the old height.
  RSD drops clients whose negotiated SDP the change falsified.
- **The refmode region can be replaced.** The read paths handle it
  internally (a region generation in the header triggers a remap), so
  a consumer sees at most one `RSS_EOVERFLOW` while the swap lands.
  Size the frame buffer for growth: a read that returns `-ENOSPC`
  reports the needed size — reallocate and continue, the stride can
  rise mid-life.

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

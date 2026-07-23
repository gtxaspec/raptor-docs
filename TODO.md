# Raptor TODO / Backlog

Cross-repo work items that are understood and scoped but not yet
scheduled. Each entry carries enough context to pick up cold.

## UDP send-buffer hardening (low priority)

Bound local UDP send-buffer drops under keyframe bursts. Do NOT copy
the vendor pattern (`wmem_default = 25MB` globally) — on 37-91 MB
cameras that lets a few stalled consumers pin tens of MB of kernel
memory.

Measured baseline (2026-07-22, Wyze Pan v3, WiFi, 8 days uptime,
production NVR load + full battery including 4-client UDP legs):
`Udp SndbufErrors = 4` lifetime vs 230K datagrams out, with
`net.core.wmem_max = wmem_default = 160KB` and no `SO_SNDBUF` on any
UDP sender (compy RTP, rwd SRTP). Provably nonzero, provably
negligible at current bitrates — hence low priority.

Planned shape when picked up:
1. compy UDP transport + rwd muxed socket: request `SO_SNDBUF`
   ~512KB (one worst-case 4K keyframe burst), per-socket.
2. thingino: sysctl fragment `net.core.wmem_max = 1MB` so the request
   isn't clamped; leave `wmem_default` small on purpose.
3. raptor-test device leg: assert `SndbufErrors` stays ~0 across the
   battery so this becomes a regression guard, not folklore.

## Ring data-region tear detection (needs ring v5 flag-day)

`rss_ring_read` re-validates the slot sequence after its memcpy, but
the contiguous data region recycles ahead of the slot ring whenever
frames are large relative to `data_size` (jpeg rings: ~250KB frames
in a 518KB region hold barely two frames; slots recycle at four).
The producer can overwrite a frame's bytes while its slot metadata is
still pristine, so the recheck passes and the consumer gets a torn
frame reported as valid. Observed live on the pan's jpeg0 ring
(2026-07-22); rhd's FF D8 guard and ringdump's SOI/EOI guard
(dec1d47) are symptom patches. H.264/H.265 rings are far less
exposed (region >> frame) but not immune under multi-frame lag.

Proper fix: producer keeps a monotonic u64 bytes-reserved counter in
the ring header (incremented at reservation time, before writing
data) and each slot stores its absolute reservation end; a reader's
copy is valid iff reserved_now - slot_abs_end <= data_size - length.
New header + slot fields = RSS_RING_VERSION bump = flag-day rebuild
of every producer and consumer, plus ASAN/TSAN + full battery pass.
Schedule deliberately; do not drive-by.

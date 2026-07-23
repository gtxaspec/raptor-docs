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

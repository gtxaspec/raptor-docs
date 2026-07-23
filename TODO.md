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

## Fix mock-HAL frame-buffer provenance (test harness)

The leak/race stage flakes with an ASAN stack-use-after-return: the
mock HAL's enc_get_frame hands rvd_encoder_thread NAL pointers whose
backing region ASAN attributes to a retired stack frame, so
rss_ring_publish_iov's memcpy trips. Pre-existing (documented since
the dual-producer note); purely a tests/mock_hal.c + leak-harness
issue - device builds are unaffected. Fix the mock to serve its
frames from a stable heap/mmap region sized per channel, then drop
the "known flaky" caveat from test-all expectations.

## Bayer sensor-RAW capture (gen3 experiment pending)

Goal: `raptorctl rvd save bayer` alongside jpeg/raw(NV12).
Findings so far (2026-07-23, pure-IMP experiment on the pan, T31
SDK 1.1.6): `PIX_FMT_RAW` on fs chn0 enables but the blob silently
delivers NV12 (frame reports FOURCC NV12, w*h*1.5 size, 128-centered
chroma plane, no mosaic column parity); chn2 refuses to enable RAW at
all. SetFrameDepth only works on an ENABLED channel (order matters).
So gen2 has no Bayer path via the framesource; the vendor tuning
tool's RAW grab rides their tuning-server protocol into ISP
internals - not worth the RE.

SOLVED PATH (2026-07-23, T41 HW-proven): the tx-isp VIC proc node.
`echo snapraw <vinum> > /proc/jz/isp/isp-w02` (gen3 arg = VI number,
0 for single-sensor; the 3.10 help text says savenum instead) dumps
one pre-ISP sensor frame to /tmp/snap0.raw. Prerequisites: (1) the
`ispmem=SIZE@ADDR` bootarg - a reserved region outside mem=/rmem=
(T41 test: osmem 74M->66M, ispmem=8M@0x4200000; full RAW16 frame =
w*h*2, 2560x1440 = 7.37MB); (2) the pipeline streaming (VIC active,
rvd up) - EPERM otherwise. Verified true Bayer: RAW10 MSB-aligned in
16-bit LE, RGGB mosaic parity flipping per row; ffmpeg -f rawvideo
-pix_fmt bayer_rggb16le demosaics it. T41 blob also has `mode`,
`dir`, and saveraw variants. Kernel side: ispmem_parse exists in ALL
six 3.10 xburst platform.c files AND the 4.4 T41 kernel; the t31
blob has the same proc commands (savenum syntax) but the pan lacks
the ispmem bootarg - untested until a t31 unit gets the bootarg
carve. Remaining for `rvd save bayer`: per-SoC ispmem bootarg in
thingino (menuconfig option sizing SIZE = sensor w*h*2 rounded up),
per-generation arg syntax (vinum vs savenum), rvd handler = echo cmd
+ wait for /tmp/snapN.raw + rename; HW-verified platform matrix (2026-07-23, all with ispmem carve +
streaming pipeline): T21 ov2735b 1920x1080 TRUE BAYER; T30 sc4236
2048x1536 TRUE BAYER; T31 (Z55 gc4653 2560x1440 + pan blob
identical) TRUE BAYER; T41 gc4023 2560x1440 TRUE BAYER. T20: SUPPORTED
via a DIFFERENT road - the first-gen tx-isp-t20.ko has no snapraw/
isp-w02, but old-SDK USERLAND PIX_FMT_RAW is genuine there (libimp
carries "RAW is only channel == 0 !"): fs chn0 attr PIX_FMT_RAW ->
enable -> SetFrameDepth(1) -> GetFrame yields fourcc BG12 (BGGR
12-bit MSB-aligned 16le), 1920x1080x2, lag-ratio 1.39 bayer-side +
per-row parity flip, kernel V4L2 negotiation logs BG12. NO ispmem
bootarg needed. Userland PIX_FMT_RAW swept across the fleet (2026-07-23):
ONLY T20 (SDK 3.12) honors it. T21 (1.0.33) and T31 (1.1.6) coerce
to NV12; T30 (1.0.5) refuses EnableChn on a RAW channel at any
resolution; T41 (1.2.5) standalone test segfaults in vendor init
(OSD pool didn't help - retest someday as an in-rvd probe where init
is known-good); T23 TESTED (1.3.0): coerces to NV12
like T31/T21 (NV12 fourcc, w*h*1.5). Its kernel blob has snapraw +
isp-w02 + the ispmem parser, so the kernel road applies (mechanism
verified, end-to-end run still pending an ispmem carve). DESIGN CONSEQUENCE for rvd save bayer: the kernel
snapraw+ispmem road is the COMMON mechanism (T21/T23/T30/T31/T41);
T20 is the exception that uses its userland road (and needs it,
having no kernel mechanism). Per-SoC ispmem menuconfig in thingino
covers everything except T20. Output naming differs: 3.10 t21/t30
write /tmp/snap.raw (no digit), t31/t41 write /tmp/snapN.raw. All
frames = w*h*2 (RAW10/12 MSB-aligned 16le); 3.10 arg = savenum
('snapraw 1'), gen3 = vinum ('snapraw 0'). All test envs restored to
stock afterward. T41 bench
env left carved (revert: osmem 74M@0x0, drop ispmem, original
bootcmd without ispmem=).

### libimp decomp cross-check (2026-07-23, ~/libimp-analysis)
Two concrete findings from reading the decompiled libimp per SoC:
1. GEN3 T41 HAS A DEDICATED USERLAND RAW API we never tried:
   IMP_ISP_RAW_RwControl(IMPVI_MAIN, &IMPISPRawRwAttr) - vendor-
   documented (imp_isp.h:4148), blob-exported (6 refs). Sets
   dtype=TISP_RAW_DTYPE_RAW16, mtype=TISP_RAW_MODULE_START, nrvbs<=4,
   and a frame_done callback that receives IMPISPRawRwInfo {vinum,
   width, height, dtype, vaddr}. Taps raw sensor data to DDR and
   calls back with the buffer - NO ispmem carve, NO kernel proc node.
   THIS is the clean gen3 userland raw path; our earlier gen3 attempt
   wrongly used framesource PIX_FMT_RAW and segfaulted. T41-ONLY among
   gen3 (T32/T33/T40 headers lack it). TODO: implement a save-bayer
   gen3 branch on this API (callback plumbing + a real capture struct;
   needs a working gen3 init, so do it in-rvd not standalone).
2. OLD-SDK userland dead-end confirmed: all four old-SDK allocators
   (T20/T21/T23/T30) size the same four 12-bit bayer fourccs
   (BG12/BA12/GB12/RG12) but the header exposes only *8-bit* BAYER
   enums (PIX_FMT_BAYER_*8=30..33); requesting those fails VBM alloc
   (size=-1, HW-confirmed on T23). Generic PIX_FMT_RAW is kernel-
   routed (T20->bayer, T21/T23/T30->NV12). So on T21/T23/T30 there is
   NO userland path to 12-bit bayer - kernel snapraw is the only road.
   Not a missing knob; the mechanism genuinely isn't exposed.

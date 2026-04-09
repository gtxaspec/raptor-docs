# RWD — WebRTC Daemon Design

## Overview

RWD delivers sub-second live video to browsers via WebRTC. It reuses
compy's SRTP/RTP/NAL stack (~2200 lines of RFC-compliant code) and adds
DTLS-SRTP handshaking via mbedTLS, ICE-lite connectivity checks, WHIP
signaling, and optional two-way audio.

Browsers are dropping RTSP support. WebRTC is the only way to get
sub-second live H.264 + audio in a browser without plugins or
transcoding.

---

## Architecture

```
Browser                          Camera (RWD)
  |                                |
  |  POST /whip (SDP offer)        |
  |------------------------------->|  HTTP signaling
  |  201 Created (SDP answer)      |
  |<-------------------------------|
  |                                |
  |  STUN binding request          |
  |------------------------------->|  ICE-lite
  |  STUN binding response         |
  |<-------------------------------|
  |                                |
  |  DTLS ClientHello              |
  |------------------------------->|  DTLS handshake
  |  DTLS ServerHello...Finished   |  (mbedTLS)
  |<-------------------------------|
  |                                |
  |  SRTP H.264 RTP packets        |  Media flow
  |<-------------------------------|  (compy SRTP + NAL)
  |  SRTP audio RTP packets        |
  |<-------------------------------|
  |  RTCP receiver reports         |
  |------------------------------->|
```

All traffic flows over a single multiplexed UDP port. Packet demux
by first byte (RFC 7983):

```
[0x00..0x03]  → STUN (ICE)
[0x14..0x15]  → DTLS
[0x80..0xBF]  → RTP/RTCP (SRTP)
```

## Components

| File | Purpose |
|------|---------|
| `rwd_main.c` | Daemon entry, config, epoll loop, client lifecycle |
| `rwd_dtls.c` | DTLS-SRTP wrapper (mbedTLS server mode, key export) |
| `rwd_ice.c` | ICE-lite STUN binding request/response, NAT hole punch |
| `rwd_sdp.c` | SDP offer parsing, SDP answer generation with ICE candidates |
| `rwd_signaling.c` | WHIP HTTP/HTTPS endpoint (POST /whip, DELETE /whip/{session}) |
| `rwd_media.c` | Ring reader → SRTP sender for video + audio |
| `rwd_webtorrent.c` | WebTorrent tracker client for external sharing (optional) |
| `rwd.h` | Shared types: client, server, DTLS context, SDP offer |
| `webrtc.html` | Embedded local viewer page |
| `share.html` | External viewer page (WebTorrent) |

### Reused from compy

| Component | Lines |
|-----------|------:|
| SRTP encryption (AES-CM-128, HMAC-SHA1) | ~840 |
| RTP transport (RFC 3550) | ~400 |
| H.264 FU-A fragmentation (RFC 6184) | ~300 |
| H.265 FU fragmentation (RFC 7798) | ~300 |
| RTCP sender reports | ~200 |
| SDP generation primitives | ~150 |
| AES-128-ECB, HMAC-SHA1, CSPRNG | per backend |
| UDP socket helpers | ~50 |

### Dependencies

- compy (SRTP, RTP, NAL — already linked by RSD)
- mbedTLS with `MBEDTLS_SSL_DTLS_SRTP` enabled (DTLS handshake + key export)
- libopus (backchannel decode — optional, for two-way audio)
- No other external libraries

---

## Configuration

```ini
[webrtc]
enabled = true
udp_port = 8443
http_port = 8554
max_clients = 2
cert = /etc/ssl/certs/uhttpd.crt
key = /etc/ssl/private/uhttpd.key
https = true                    # HTTPS for signaling (enables Talk button)
local_ip =                      # auto-detected if omitted
audio_mode = auto               # auto (passthrough) or opus (always transcode)
opus_complexity = 2             # 0-10, transcode encoder complexity
opus_bitrate = 64000            # transcode encoder bitrate (bps)
```

---

## Compatibility

### Browsers

| Browser | Status | Notes |
|---------|--------|-------|
| Chrome | Supported | Tested, video + audio + backchannel |
| Edge | Supported | Same engine as Chrome |
| Safari | Supported | H.264 native |
| Firefox | Blocked | Requires mbedTLS >= 3.6.6 (ClientHello defragmentation) |

### go2rtc

Compatible as a WHEP source:
```yaml
streams:
  camera_webrtc:
    - webrtc:http://CAMERA_IP:8554/whip
```

Requirements for pion compatibility:
- SSRC declared in SDP answer (`a=ssrc:` lines)
- sdes:mid RTP header extension for BUNDLE demux
- SHA256 ciphersuites (SHA384 PRF produces wrong SRTP keys with
  the TLS PRF workaround)

---

## Backchannel (Two-Way Audio)

Browser microphone audio flows back to the camera speaker:

```
Browser mic → getUserMedia → RTCPeerConnection (sendrecv audio)
  → SRTP-encrypted Opus RTP packets
  → RWD UDP socket (demuxed by first byte, same port as outgoing)
  → compy_srtp_recv_unprotect (decrypt with DTLS-derived recv key)
  → Compy_RtpReceiver_feed (RTP demux by PT)
  → rwd_bc_recv_t callback (Opus decode via libopus, 48kHz→16kHz)
  → "speaker" ring (PCM16, 16kHz, mono)
  → RAD ao_playback_thread → IMP AO hardware → speaker
```

### SDP negotiation

Audio m-line is `a=sendrecv`. SDP answer includes both `opus/48000/2`
and `PCMU/8000`. Browser JS uses `setCodecPreferences([PCMU, Opus])`
to prefer PCMU, falling back to Opus (decoded via libopus).

### HTTPS requirement

`getUserMedia()` requires a secure context. RWD serves HTTPS by default
using the same cert/key as DTLS (`[webrtc] https = true`). The Talk
button only appears when the SDP answer contains `a=sendrecv` and
`navigator.mediaDevices` is available.

### Codec handling

- **Opus** (PT from offer): decode via `opus_decode()` at 48kHz,
  downsample 3:1 to 16kHz
- **PCMU** (PT 0): G.711 µ-law decode + 8→16kHz upsample

### Speaker ring

The backchannel opens (or creates) a `speaker` SHM ring. RAD's AO
playback thread polls for this ring and plays PCM16 frames to hardware.
Uses `rss_ring_open()` first to reuse an existing ring (avoids stale
mmap when switching between RTSP and WebRTC backchannel sessions).

### RTP/RTCP demux

Incoming packets on the shared UDP port are demuxed by `data[1]`:
- `data[1] >= 200 && <= 209` → SRTCP
- Otherwise → SRTP

The PT byte must NOT be masked with `& 0x7F` — RTCP types 200-209
have the high bit set in byte[1], and masking produces values 72-81
which would be misidentified as RTP.

---

## A/V Sync

Both video and audio RTP timestamps derive from `meta.timestamp`
(IMP's `CLOCK_MONOTONIC_RAW`, microsecond precision). Since both
streams reference the same system clock, inter-stream drift is zero
by construction.

Previous approach (counter-based) drifted ~10.6s/day due to audio ADC
crystal offset (~16000.5 Hz vs nominal 16000 Hz). Wall-clock timestamps
eliminate this — measured 7ms delta over 1 hour (noise, not drift).

Audio is gated on the first video keyframe so both RTP timelines start
together.

---

## WebTorrent External Sharing

RWD supports external WebRTC viewing without port forwarding via
WebTorrent tracker signaling. Built with `WEBTORRENT=1`.

### How it works

1. Camera connects outbound (TLS WebSocket) to a public WebTorrent
   tracker (`wss://tracker.openwebtorrent.com`) and announces a share
   room identified by `base64(SHA-256("raptor:" + share_key))`.
2. Camera discovers its public IP:port via STUN binding request to
   a public STUN server, adds the result as a server-reflexive (srflx)
   ICE candidate in SDP answers.
3. Viewer opens `share.html#key=<share_key>` in a browser. The page
   connects to the same tracker, computes the same info_hash, and sends
   an SDP offer through the tracker.
4. Camera receives the offer, creates a client (same as WHIP), generates
   an SDP answer, and sends it back through the tracker.
5. ICE connectivity checks punch through NAT. Direct P2P connection
   established. DTLS handshake → SRTP → video+audio flows directly.

### SDP size limit

Public WebTorrent trackers silently drop large messages. Chrome's SDP
offers include dozens of unused codecs (~8KB). The viewer page strips
the SDP to H264 + Opus only (~1KB) before sending through the tracker.

### Share key

- Auto-generated (31 random alphanumeric chars) if not configured.
- Configurable via `[webtorrent] share_key`. A 4-char deterministic
  prefix (XOR-folded from the key) prevents tracker collisions.
- `raptorctl rwd share` shows the current share URL.
- `raptorctl rwd share-rotate` generates a new random key.

### NAT compatibility

| NAT type | Status |
|----------|--------|
| Full-cone | Works (srflx candidate reachable from any host) |
| Port-restricted | Works (ICE hole punch opens the mapping) |
| Symmetric | May not work (srflx mapping differs per destination) |

Users behind symmetric NAT need a VPN or go2rtc as a proxy.

### Configuration

```ini
[webtorrent]
enabled = false
tracker = wss://tracker.openwebtorrent.com
stun_server = stun.l.google.com
stun_port = 19302
tls_verify = true               # verify tracker TLS cert
viewer_url = https://viewer.thingino.com/share.html
share_key =                     # min 4 chars, auto-generated if omitted
```

---

## Known Issues

### compy SRTP fixes (resolved)

1. **IV construction** (`e87567d`): Session salt was at bytes [2..15]
   instead of [0..13] per RFC 3711 Section 4.1.
2. **SSRC byte order** (`7b6458d`): RTP SSRC was in host byte order
   (no htonl) while RTCP already used htonl.
3. **RTP extension support** (`2fb9ab4`): Added one-byte header
   extension (RFC 8285) for sdes:mid. SRTP now skips extensions when
   computing the payload encryption offset.
4. **Extension serialization** (`9392579`): extension_payload_len was
   in network byte order but used as host order for size computation.

### mbedTLS export_keying_material crash

`mbedtls_ssl_export_keying_material()` segfaults in mbedTLS 3.6.5
on DTLS 1.2. RWD works around this by using `mbedtls_ssl_tls_prf()`
with the master secret captured via `mbedtls_ssl_set_export_keys_cb`.

### Firefox: ClientHello fragmentation

Firefox sends a ~1400 byte DTLS ClientHello (TLS 1.3 extensions:
key_share, supported_versions). mbedTLS 3.6.5 cannot reassemble a
fragmented ClientHello. Fix:
[Mbed-TLS/mbedtls#10623](https://github.com/Mbed-TLS/mbedtls/pull/10623)
merged to `mbedtls-3.6` branch (2026-03-10), shipping in 3.6.6.

### DTLS cookie disabled

HelloVerifyRequest is disabled because ICE already verifies the
client's transport address before DTLS begins. Safe for WebRTC but
RWD should not be exposed as a standalone DTLS server without ICE.

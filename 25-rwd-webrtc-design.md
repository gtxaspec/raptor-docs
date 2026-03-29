# RWD — Minimal WebRTC Daemon Plan

## Context

Browsers are dropping RTSP support. WebRTC is the only way to get
sub-second live video in a browser without plugins. Current options
(go2rtc, mediamtx) are Go/heavy. We need a minimal C implementation
that fits the raptor micro-daemon architecture and reuses compy's
existing SRTP/RTP/NAL stack.

**Goal**: Browser opens a page, clicks play, gets live H.264 + audio
with <500ms latency. No plugins, no transcoding.

---

## What We Already Have (reusable from compy)

| Component | Status | Lines |
|-----------|--------|-------|
| SRTP encryption (AES-CM-128, HMAC-SHA1) | complete | ~840 |
| RTP transport (RFC 3550) | complete | ~400 |
| H.264 FU-A fragmentation (RFC 6184) | complete | ~300 |
| H.265 FU fragmentation (RFC 7798) | complete | ~300 |
| RTCP sender reports | complete | ~200 |
| SDP generation primitives | complete | ~150 |
| AES-128-ECB, HMAC-SHA1, CSPRNG | complete | per backend |
| UDP socket helpers | complete | ~50 |

**Total reuse: ~2200 lines of tested, RFC-compliant code.**

## What mbedTLS Provides

- DTLS 1.2 handshake (enabled in sysroot, v3.6.5)
- HelloVerifyRequest (DoS protection)
- Anti-replay, Connection ID
- **DTLS-SRTP extension (RFC 5764): DISABLED** — needs
  `MBEDTLS_SSL_DTLS_SRTP` enabled in buildroot mbedtls config

## What Needs To Be Built

### 1. DTLS-SRTP wrapper (~500 lines)

Wrap mbedTLS DTLS API for UDP server mode:
- `rwd_dtls_init(cert, key)` → create DTLS server context
- `rwd_dtls_accept(ctx, udp_fd)` → handshake on incoming connection
- `rwd_dtls_export_srtp_keys(conn)` → extract master key + salt
- Uses `MBEDTLS_SSL_TRANSPORT_DATAGRAM` (already supported)
- Needs `MBEDTLS_SSL_DTLS_SRTP` enabled for `use_srtp` extension

**Buildroot change**: enable `MBEDTLS_SSL_DTLS_SRTP` in mbedtls
package config. One line.

### 2. ICE-lite (~200 lines)

Camera is always the answerer, always ICE-lite:
- Parse STUN binding requests (20-byte header + attributes)
- Reply with STUN binding response (mapped address)
- Use `ice-ufrag` / `ice-pwd` from SDP for message integrity
- No TURN, no candidate gathering, no trickle ICE
- Single candidate: host IP + UDP port

```c
rwd_ice_process(buf, len, from_addr)  // handle STUN request
rwd_ice_check_integrity(buf, ice_pwd) // HMAC-SHA1 verify
rwd_ice_send_response(fd, to_addr)    // STUN response
```

### 3. SDP offer/answer (~300 lines)

Parse browser's SDP offer, generate camera's SDP answer:
- Extract: codec (H.264 profile, audio codec), ICE credentials,
  DTLS fingerprint, SSRC
- Generate answer with camera's: codec params (from SHM ring header),
  ICE-lite candidate (host IP:port), DTLS fingerprint, SRTP params
- Single audio + single video m-line
- Codec: H.264 Constrained Baseline (profile 42e0) for browser compat

```c
rwd_sdp_parse_offer(sdp_str, &offer)
rwd_sdp_generate_answer(&offer, &camera_params, answer_buf)
```

### 4. WHIP signaling endpoint (~200 lines)

Simple HTTP POST handler for WebRTC-HTTP Ingestion Protocol:
- `POST /whip` — browser sends SDP offer, camera returns SDP answer
- `DELETE /whip/{session}` — browser tears down session
- Reuse RHD-style HTTP parsing (or embed in RWD)
- Response: `201 Created` with SDP answer in body
- CORS headers for browser access

### 5. RWD daemon (~600 lines)

New daemon following raptor pattern:
- `rwd_main.c` — init, config, main loop
- Reads video + audio from SHM rings (same pattern as RSD)
- Per-client state: DTLS conn, SRTP context, RTP transport
- Epoll loop: signaling HTTP + DTLS/STUN/SRTP on UDP
- Max clients: 2-4 (WebRTC is heavier than RTSP)

```
Config: [webrtc]
enabled = false
port = 8443
max_clients = 2
cert = /etc/ssl/certs/webrtc.crt
key = /etc/ssl/private/webrtc.key
```

### 6. Web page (~100 lines HTML/JS)

Served by RWD or RHD at `/webrtc`:
- WHIP client: fetch POST to `/whip` with SDP offer
- `RTCPeerConnection` with H.264 + Opus
- Minimal UI: video element + connect button
- No external dependencies

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

All on single UDP port (multiplexed by first byte):
- 0x00-0x03: STUN
- 0x14-0x15: DTLS
- 0x80-0xBF: RTP/RTCP

## Packet demux (~50 lines)

```c
if (buf[0] < 0x04)       → ICE/STUN
else if (buf[0] < 0x40)  → DTLS
else                      → SRTP/SRTCP
```

---

## New Files

| File | Purpose | Lines |
|------|---------|-------|
| `rwd/rwd_main.c` | Daemon entry, config, epoll loop | ~400 |
| `rwd/rwd_dtls.c` | DTLS-SRTP wrapper (mbedTLS) | ~500 |
| `rwd/rwd_ice.c` | ICE-lite STUN handler | ~200 |
| `rwd/rwd_sdp.c` | SDP offer parsing + answer generation | ~300 |
| `rwd/rwd_signaling.c` | WHIP HTTP endpoint | ~200 |
| `rwd/rwd_media.c` | Ring reader → SRTP sender | ~300 |
| `rwd/rwd.h` | Types and declarations | ~100 |
| `rwd/Makefile` | Build | ~20 |

**Total new code: ~2000 lines**

## Modified Files

| File | Change |
|------|--------|
| `Makefile` | Add rwd to DAEMONS |
| `build.sh` | Add rwd to build list |
| `build-asan.sh` | Add rwd |
| `config/raptor.conf` | Add `[webrtc]` section |
| `raptorctl/raptorctl.c` | Add rwd to daemons list |
| `thingino-raptor.mk` | Add rwd to build + install |

## Buildroot Changes

| File | Change |
|------|--------|
| `package/thingino-raptor/Config.in` | Add `BR2_PACKAGE_THINGINO_RAPTOR_WEBRTC` that selects `BR2_PACKAGE_MBEDTLS_DTLS_SRTP` |

`BR2_PACKAGE_MBEDTLS_DTLS_SRTP` already exists in buildroot's mbedtls
package — just needs to be selected by raptor's Config.in.

```kconfig
# In thingino-raptor Config.in:
config BR2_PACKAGE_THINGINO_RAPTOR_WEBRTC
    bool "WebRTC daemon (RWD)"
    default n
    select BR2_PACKAGE_MBEDTLS_DTLS_SRTP
    help
      Build the WebRTC streaming daemon with WHIP signaling.
      Requires mbedTLS with DTLS-SRTP support.
```

## Dependencies

- compy (SRTP, RTP, NAL — already linked by RSD)
- mbedTLS (DTLS — already in sysroot for RTSPS)
- No new external libraries

---

## Browser Compatibility

WebRTC H.264 support:
- Chrome: ✓ (tested, working)
- Edge: ✓ (same engine as Chrome)
- Safari: ✓ (H.264 native)
- Firefox: ✗ requires mbedTLS ≥ 3.6.6 (see Known Issues)

Audio: Opus (already supported by RAD + compy)

## Known Issues

### compy SRTP IV fix (fixed)

compy's SRTP AES-CM IV construction had the session salt at bytes
[2..15] instead of [0..13] per RFC 3711 Section 4.1. The packet
index was also packed as a 32-bit value instead of ROC(32)+SEQ(16).
Fixed in compy commit `e87567d`.

### mbedTLS `export_keying_material` crash

`mbedtls_ssl_export_keying_material()` segfaults in mbedTLS 3.6.5
when called on a DTLS 1.2 context. RWD works around this by using
`mbedtls_ssl_tls_prf()` with the master secret captured via the
key export callback (`mbedtls_ssl_set_export_keys_cb`).

### Firefox: ClientHello fragmentation (mbedTLS < 3.6.6)

Firefox sends a ~1400 byte DTLS ClientHello (includes TLS 1.3
extensions: key_share, supported_versions). This exceeds the DTLS
record MTU and gets fragmented at the DTLS handshake layer.
mbedTLS 3.6.5 does not support reassembling a fragmented
ClientHello, returning `MBEDTLS_ERR_SSL_FEATURE_UNAVAILABLE`.

Fix: [Mbed-TLS/mbedtls#10623](https://github.com/Mbed-TLS/mbedtls/pull/10623)
backported ClientHello defragmentation to the `mbedtls-3.6` branch
(merged 2026-03-10). This will ship in mbedTLS 3.6.6. Until then,
building against the `mbedtls-3.6` branch HEAD enables Firefox.

### DTLS cookie disabled

DTLS HelloVerifyRequest (cookie exchange) is disabled because ICE
already verifies the client's transport address before DTLS begins.
This is safe for WebRTC but means RWD should not be exposed as a
standalone DTLS server without ICE.

---

## Verification

1. Build: `./build.sh t31 <output> rwd`
2. Config: `[webrtc] enabled = true`, cert/key paths set to valid
   certificate (uhttpd certs work: `/etc/ssl/certs/uhttpd.crt`)
3. Start RWD alongside other daemons
4. Open `http://camera:8554/webrtc` in browser
5. Click Connect → WHIP POST → DTLS handshake → video plays
6. Click Unmute for audio
7. Verify: sub-second latency, audio+video in sync
8. Test: multiple clients, disconnect/reconnect, Chrome/Edge/Safari

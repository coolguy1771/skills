# ASVS 5.0.0 — V17: WebRTC

Condensed from OWASP ASVS 5.0.0 (CC BY-SA 4.0). `L` = the lowest level at which the requirement
applies. Cite as `v5.0.0-<id>`.

**Scope:** teams that build WebRTC products, run a CPaaS, or operate their own TURN, media
(SFU/MCU/recording/gateway), or signaling servers. It does **not** apply to apps that only consume
a CPaaS vendor's SDK; the vendor owns those concerns. Pure browser peer-to-peer media without
media servers is excluded from V17.2.

Media and signaling servers need general flood and malformed-input hardening: rate limiting,
timestamp validation, synchronized clocks, buffer management, dropping excess packets per
session, integer/buffer overflow safety, and robust error handling.

### V17.1 TURN Server
| ID | L | Requirement |
|---|---|---|
| 17.1.1 | 2 | TURN relays only to IPs **not** reserved for special purposes (internal ranges, loopback, broadcast, link-local) for both IPv4 and IPv6. Prevents TURN-as-SSRF into internal networks. |
| 17.1.2 | 3 | TURN resists resource exhaustion when legitimate users open many ports. |

### V17.2 Media
| ID | L | Requirement |
|---|---|---|
| 17.2.1 | 2 | The DTLS certificate key is managed per the documented key management policy (V11.1.1). |
| 17.2.2 | 2 | Approved DTLS cipher suites and a secure DTLS-SRTP protection profile. |
| 17.2.3 | 2 | SRTP authentication is checked at the media server (prevents RTP injection, DoS, media insertion). |
| 17.2.4 | 2 | Processing continues when malformed SRTP packets arrive. |
| 17.2.5 | 3 | Processing continues during an SRTP flood from legitimate users. |
| 17.2.6 | 3 | Not vulnerable to the DTLS "ClientHello" race condition (check known-vulnerable versions or test). |
| 17.2.7 | 3 | Recording continues during an SRTP flood. |
| 17.2.8 | 3 | The DTLS certificate is checked against the SDP `fingerprint`; the stream is terminated on mismatch. |

### V17.3 Signaling
| ID | L | Requirement |
|---|---|---|
| 17.3.1 | 2 | Signaling keeps serving legitimate messages during a flood (signaling-level rate limiting). |
| 17.3.2 | 2 | Signaling keeps serving legitimate messages when malformed messages arrive (validation, overflow safety, robust errors). |

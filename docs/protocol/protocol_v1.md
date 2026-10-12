# Telemetry Protocol v1

Source of truth for the wire format. The latest `protocol_vX.md` is the one to follow; older versions stay for history.
A datasheet copy with diagrams is in `protocol_v1.pdf`. It replaces the v0 draft (`protocol_v0.md`, local only).

Application protocol that runs inside a TCP stream between one gateway and its clients. All multi-byte values are
**big-endian**. Fields are read byte by byte (or `memcpy` then `ntoh*`), never by casting the receive buffer to a struct.

## 1. Frame layout

| Offset (bytes) | Size | Field | Notes |
|---|---|---|---|
| 0 | 1 | magic | `0x78` |
| 1 | 1 | version / reserved | low nibble = version (`0b0001`), high nibble = reserved, must be 0 |
| 2-3 | 2 | seq | u16 |
| 4-5 | 2 | len | u16, payload bytes, 0 to 1024 |
| 6-7 | 2 | type | u16, see section 2 |
| 8 ... | N | payload | N = len |
| 8+N ... | 4 | crc32 | see below |

- Header is 8 bytes and the CRC adds 4. A frame is **12 + N** bytes. The largest frame is **1,036** bytes.
- **CRC-32:** IEEE 802.3, reflected polynomial `0xEDB88320`, initial value `0xFFFFFFFF`, reflected input and output,
  final XOR `0xFFFFFFFF`. It covers bytes 0 to 7+N, so the header is included.
  **Test vector:** the CRC of ASCII `"123456789"` is `0xCBF43926`. This is the first unit test.
- **seq** counts TELEMETRY frames. It starts at 0 on each connection and wraps at 65,536. On the other types it carries a
  number that labels the message (section 2).

## 2. Message types

| Type | Value | Direction | Header seq | len | Sent when |
|---|---|---|---|---|---|
| TELEMETRY | `0x0058` | gateway to client | frame number | 24 | every sample tick (default 100 Hz) |
| HEARTBEAT | `0x00BC` | both | 0 | 0 | a side has sent nothing for one heartbeat interval |
| RESEND | `0x001A` | client to gateway | the seq wanted | 0 | the client sees a gap or a failure |
| ACK | `0x0064` | gateway to client | the seq requested | 0 | the gateway has the frame; the frame follows ACK immediately |
| REJECTED | `0x006E` | gateway to client | the seq it cannot supply | 0 | the frame is no longer in the gateway's buffer |
| ERROR | `0x0005` | client to gateway | 0 | 0 | the client would need a third RESEND within one second |

**Numbering:** the high nibble of the low byte gives the direction. 0-4 client to gateway, 5-9 gateway to client,
A-F both. The type field is 16 bits, so the high byte is `0x00`.

Deferred to a later version: HELLO, HELLO_ACK, PING, PONG.

### TELEMETRY payload (24 bytes)

| Offset | Size | Field | Notes |
|---|---|---|---|
| 0 | 8 | timestamp | u64, nanoseconds, BBB monotonic clock, at sample time |
| 8 | 2 | dev_id | u16 |
| 10 | 2 | dev_addr | u16 |
| 12 | 6 | accel x, y, z | i16 x 3, raw counts |
| 18 | 6 | gyro x, y, z | i16 x 3, raw counts |

Signed values go on the wire as two's-complement bit patterns. The client converts counts to g and degrees per second.
A TELEMETRY frame is 8 + 24 + 4 = **36 bytes**. HEARTBEAT, RESEND, ACK, REJECTED and ERROR have empty payloads, so each
is a 12-byte frame.

## 3. Receiver rules

| Condition | Action |
|---|---|
| fewer than 8 bytes buffered | wait for more |
| magic, version or reserved wrong; len above 1024; CRC mismatch; unknown type* | failure: discard the buffer, send RESEND(expected seq), scan for ACK or REJECTED. No allocation for an oversize len. |
| TELEMETRY seq later than expected | send RESEND for the missing frames |
| a third RESEND needed within one second | send ERROR, close, reconnect |
| peer silent for 3 x heartbeat interval | treat as dead, close |
| gateway receives a bad frame from a client* | close |

\* Defaults assumed during design review. Not yet confirmed.

## 4. Recovery and liveness

- **Gap:** the client expects 90 and receives 100. It sends RESEND for the missing frames and loops until it is back in
  step with the gateway.
- **Gateway reply:** ACK(k), then TELEMETRY frame k. If the frame is gone, REJECTED(k). The client logs the gap and continues.
- **Realignment:** after a failure the client discards its buffer and sends RESEND(expected seq). ACK and REJECTED have empty
  payloads, so each is a complete 12-byte frame with its own CRC. The client scans for a candidate whose magic matches,
  whose type is ACK or REJECTED, whose seq matches the request, and whose CRC verifies. The frame after the ACK starts
  exactly where the ACK ends.
- **ACK timeout:** 300 ms. On a timeout the client sends RESEND again*, and that counts toward the limit below.
- **Limit:** up to 2 RESENDs are allowed within any one second, for any frames. Needing a third means the client sends
  ERROR, closes and reconnects. The gateway closes when it receives ERROR.
- **Heartbeat:** each side sends HEARTBEAT if it has been idle for the interval. The gateway streams at 100 Hz, so in
  practice the client's heartbeats keep the gateway informed. The interval is a constant shared by both sides, since
  nothing negotiates it. The v0 draft's default was 1,000 ms.
- **Gateway buffer:** a fixed ring buffer of the last 16 TELEMETRY payloads of 24 bytes each, 384 bytes. On a replay the
  gateway rebuilds the header from the slot's seq and recomputes the CRC.

## 5. Constants

| Name | Value |
|---|---|
| magic | `0x78` |
| version | `0b0001` (low nibble of byte 1) |
| MAX_PAYLOAD | 1024 bytes |
| Largest frame | 1036 bytes |
| TELEMETRY frame | 36 bytes |
| Control frame (empty payload) | 12 bytes |
| Sample rate default | 100 Hz |
| ACK timeout | 300 ms |
| RESEND limit | 2 per 1 second window, third triggers ERROR |
| Ring buffer | 16 x 24 bytes = 384 bytes |

## 6. State machines

Client states are per connection. The gateway keeps one entry per connected client.

### Client

```
                    connect ok
 DISCONNECTED ----------------------> STREAMING <-----------------+
      ^   (backoff, then reconnect)      |   |                    |
      |                                  |   | gap: seq > expected|
      |                                  |   v                    |
      |                                  | RECOVERING ------------+
      |                                  |   |   caught up
      |   bad frame                      |   | (ACK timeout or new loss)
      |   (discard buffer)               v   |
      |                              REALIGNING  (scan for ACK/REJECTED)
      |                                  |
      |   3rd RESEND needed in 1 s       |
      +------- ERROR sent, close <-------+  (from any state above)
```

| State | Event | Action | Next |
|---|---|---|---|
| DISCONNECTED | backoff done | connect | STREAMING (expected seq = 0) |
| STREAMING | TELEMETRY, seq = expected | log CSV row, expected + 1 | STREAMING |
| STREAMING | TELEMETRY, seq above expected | RESEND(expected) | RECOVERING |
| STREAMING | bad frame (magic, version, reserved, len above 1024, CRC, unknown type) | discard buffer, RESEND(expected) | REALIGNING |
| STREAMING | HEARTBEAT | reset silence timer | STREAMING |
| STREAMING | idle for one heartbeat interval | send HEARTBEAT | STREAMING |
| RECOVERING | ACK(k) | wait for TELEMETRY(k) | RECOVERING |
| RECOVERING | TELEMETRY(k) | log, expected + 1; if more missing, RESEND(next) | RECOVERING |
| RECOVERING | REJECTED(k) | log the gap, expected + 1; if more missing, RESEND(next) | RECOVERING |
| RECOVERING | caught up to the frame that triggered recovery | none | STREAMING |
| RECOVERING | ACK timeout (300 ms) | RESEND again | RECOVERING |
| REALIGNING | CRC-valid ACK or REJECTED for the requested seq | continue as in RECOVERING | RECOVERING |
| REALIGNING | scan timeout (300 ms) | RESEND again | REALIGNING |
| any | a RESEND would be the 3rd within 1 s | send ERROR, close | DISCONNECTED |
| any | peer silent for 3 x heartbeat interval | close | DISCONNECTED |

Every RESEND, including those sent after a timeout, counts toward the limit of 2 within one second.

### Gateway (one entry per connected client)

| State | Event | Action | Next |
|---|---|---|---|
| (listening) | client connects | create the entry: socket, buffer, seq = 0, empty ring buffer | ACTIVE |
| ACTIVE | sample tick | send TELEMETRY(seq), store the payload in slot seq mod 16, seq + 1 | ACTIVE |
| ACTIVE | RESEND(k), k in the ring buffer | send ACK(k), then TELEMETRY(k) rebuilt from the slot | ACTIVE |
| ACTIVE | RESEND(k), k not in the buffer | send REJECTED(k) | ACTIVE |
| ACTIVE | HEARTBEAT | reset silence timer | ACTIVE |
| ACTIVE | idle for one heartbeat interval | send HEARTBEAT | ACTIVE |
| ACTIVE | ERROR received | close | CLOSED |
| ACTIVE | bad frame from the client* | close | CLOSED |
| ACTIVE | peer silent for 3 x heartbeat interval | close | CLOSED |
| CLOSED | entry freed | free the socket and the buffer | gone |

A replay is handled as an event inside ACTIVE, so the gateway holds no extra state per request.

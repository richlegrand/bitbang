# SWSP -- Simple WebRTC Streaming Protocol

SWSP is BitBang's application-layer multiplexing protocol. It runs over one
WebRTC data channel -- the **SWSP channel** -- between a connector and a
listener, and carries every request-shaped feature -- HTTP proxying, WebSocket
tunneling, file transfer, shells, TCP forwarding, a device's console -- as
independent logical **streams** over that one channel. A session may have other
data channels beside it (section 1); they are not SWSP.

This document is the wire-level reference: frame format, stream lifecycle, the
stream-0 control handshake, every per-type message, the two chunking modes, and
versioning. Four implementations exist, and where they differ this document
says so:

| implementation | roles | code |
|---|---|---|
| Go (`bitbang-cli`) | listener and connector | `internal/protocol`, `internal/session`, `internal/streamtype`, `internal/client` |
| browser (`bitbang-server`) | connector | `web/bootstrap.js` |
| Python (`bitbang-python`) | listener | `bitbang/adapter.py` |
| ESP32 (`bitbang-esp32`) | listener | `components/bitbang_signaling` (`bitbang_verify.c`, `bitbang_signaling_interface.c`), `components/bitbang_httpd`, `components/bitbang_console` |

Audited against all four on 2026-10-02.

> **Scope.** SWSP begins *after* the data channel is open and bidirectional
> verify has run. WebRTC/DTLS setup, signaling, and the pubkey/SAS
> authentication are out of scope here -- see `signaling.md`,
> `trustless-signaling.md` and `code_exchange.md`. The one exception is the `verify_nonce_hash` control
> message, which is the seam between verify and SWSP and is documented below.

---

## 1. Where SWSP sits

```
 connector  ── WebRTC data channel (DTLS-encrypted, ordered, reliable) ──  listener
     │                                                                          │
     │   one channel, many streams, multiplexed by SWSP:                        │
     │     stream 0  → control (handshake, auth, ready, video negotiation)      │
     │     stream 1+ → http / websocket / file / shell / tcp / console          │
```

One **session** = one SWSP channel = one SWSP instance. The listener side is
`internal/session.Session`; the connector side is `internal/client.Session` and
`bootstrap.js`'s `BitBangConnection`.

### The SWSP channel among others

The listener is the offerer and opens the channels. Besides the SWSP channel, a
device may open one **media channel** per stream component -- label
`<presentation>/<component>` (`cam/video`), protocol `bitbang-stream/<codec>` --
carrying video or audio frames in their own format (`av-streaming-api.md`,
*Wire format*). Those channels are unordered and unreliable by design, which is
exactly what SWSP must never run on (below).

How each side tells them apart:

| side | rule |
|---|---|
| connectors (browser, Go CLI) | a channel whose `protocol` starts with `bitbang-stream/` is media; any other is the SWSP channel |
| ESP32 listener | the channel labeled `bitbang`, or an unlabeled one |

The SWSP channel's label is not uniform: the Go and Python listeners label it
`http`, the ESP32 labels it `bitbang`. Connectors do not look at it, which is why
that has not mattered. A new implementation should classify by `protocol`, as
the connectors do.

### Assumes a reliable, ordered channel

SWSP carries **no sequence numbers, acknowledgements, or retransmission of its
own.** It relies entirely on the underlying WebRTC data channel being
**reliable and in-order** -- the default `RTCDataChannel` mode (no
`maxRetransmits`/`maxPacketLifeTime`, `ordered: true`). Everything downstream
depends on this: a stream's frames are assumed to arrive in send order
(`SYN` before its `DAT`s before its `FIN`), data is reassembled by simply
concatenating `DAT` payloads in arrival order, and there is no mechanism to
detect or recover a dropped or reordered frame. If SWSP were ever run over an
unreliable/unordered channel it would corrupt silently. Ordering is guaranteed
only *within* a stream, not across streams (frames from different streams may
interleave freely on the channel).

### Two version numbers (don't conflate them)

| Constant | Value | Meaning | Carried in |
|---|---|---|---|
| `ProtocolVersion` | 3 | **Registration** protocol (signaling-server `register`) | device `register` message |
| `SWSPVersion` | 4 | **Data-channel wire** protocol | stream-0 `connect` (→) and `ready` (←) |

They evolve independently (`internal/protocol/swsp.go`). v4 of the data-channel
protocol required no registration change at all -- the signaling server sees
nothing different about a v4 session.

What each implementation speaks today:

| implementation | sends | flow control (section 6) | note |
|---|---|---|---|
| Go listener | `server_version` 4 | yes | |
| Go connector | `version` 4 | yes | |
| browser | `version` 3 | no | |
| Python listener | `server_version` 3 | no | |
| ESP32 listener | `server_version` 3 | no | was 4 until 2026-10-02 |

**The ESP32 used to advertise v4 without implementing it.** It sent
`server_version: 4` and negotiated `min(peer, 4)`, with no `window_update` or
`stream_reset` handling. Against the browser (v3) that was harmless. Against
the Go connector (v4) the session negotiated v4, the connector enforced send
credit, and any single stream it sent on would have stalled after the 1 MiB
initial window, waiting for a `window_update` the device never sends. It now
advertises 3, and should go to 4 only with section 6 implemented. Firmware from
before the change still says 4.

---

## 2. Frame format

Every data-channel message is exactly **one SWSP frame**: an 8-byte
little-endian header followed by the payload.

```
 +-----------+-----------+-----------+------------------+
 | StreamID  | Flags     | Length    | Payload          |
 | 4 bytes   | 2 bytes   | 2 bytes   | `Length` bytes   |
 | uint32 LE | uint16 LE | uint16 LE | (opaque)         |
 +-----------+-----------+-----------+------------------+
```

- `StreamID` -- 0 for control; non-zero identifies a multiplexed stream.
- `Flags` -- bitfield (below).
- `Length` -- payload byte count. Bounded by `MaxChunkSize`.
- `HeaderSize` = 8. `MaxChunkSize` = **32768** (32 KB); a full frame stays under
  the 64 KB SCTP message limit.

`MaxChunkSize` is a ceiling on what a receiver must accept, not the size every
sender uses. The Go implementation sends up to 32768; the browser and Python
send at most 16384 (`SWSP_CHUNK_SIZE`); the ESP32's HTTP bridge sends body
chunks of at most 8192. Every receiver accepts up to 32768. A sender may pick
anything up to the ceiling.

Encode/decode: `protocol.BuildFrame` / `protocol.ParseFrame`.

### One frame per data-channel message

Each frame is sent as its **own** data-channel message (`DC.Send(BuildFrame(…))`),
and the receiver parses exactly **one** frame from each message it receives.
Frames are therefore delimited by the **SCTP message boundary**, not by the
`Length` field or by scanning a byte stream -- a reader never has to buffer a
partial frame or split two frames out of one message.

That makes `Length` partly redundant with the message boundary, and it is: it's
a **validation guard**, not the delimiter. `ParseFrame` rejects a message
shorter than `HeaderSize + Length`, and any bytes *beyond* `HeaderSize + Length`
in the same message are **ignored**. So an implementer should (a) put one frame
per message, and (b) treat the message boundary -- not `Length` -- as
authoritative for where the payload ends.

### Flags

| Flag | Value | Meaning |
|---|---|---|
| `FlagDAT` | `0x0000` | Data chunk (no bits set) |
| `FlagSYN` | `0x0001` | Start of stream; payload is JSON metadata |
| `FlagMORE` | `0x0002` | Non-final fragment of a chunked **WebSocket** message |
| `FlagFIN` | `0x0004` | End of stream |

Flags combine:

- `SYN` alone -- open a stream; more frames follow.
- `SYN|FIN` -- a complete stream in one frame (e.g. a body-less HTTP `GET`, or a
  one-shot control reply like `ready`).
- `DAT` (no bits) -- a data chunk.
- `DAT|MORE` -- a non-final fragment of one WebSocket message (see §6).
- `FIN` -- close the stream; payload may carry a small trailer (e.g. shell exit
  code) or be empty.

Helpers: `Frame.IsSYN()`, `Frame.IsFIN()`, `Frame.IsMORE()`.

---

## 3. Streams and their lifecycle

A non-zero stream is a single request/response or a long-lived bidirectional
flow. Its life is always:

```
 SYN  (JSON metadata: {"type": "...", ...})
  │
 DAT  (zero or more data chunks -- body, output, messages)
  │
 FIN  (optional trailer payload; closes the stream)
```

- The **opener** sends `SYN` first; the payload's `type` field selects the
  handler (`http`, `websocket`, `file`, `shell`, `tcp`, `console`; section 5). A
  missing `type` defaults to `http` (v2 back-compat).
- `SYN|FIN` in one frame = a stream with no DAT phase.
- Routing (`session.go`): on the listener, the `SYN`'s `type` picks a
  `StreamHandler`; that stream ID is then pinned to that handler, so all later
  `DAT`/`FIN` frames on the same ID route to it without re-parsing.
- On `FIN`, the stream's routing entry and any per-stream state are dropped.

### Concurrency, teardown, and orphan frames

- **Many streams run at once.** Multiplexing is the point: any number of
  streams can be in flight simultaneously, and their frames interleave freely on
  the channel. Ordering is guaranteed only *within* a stream (per §1), never
  across streams -- a handler must not assume stream *N*'s frames arrive before
  stream *M*'s.
- **Orphan frames are dropped.** A `DAT`/`FIN` for a stream ID that has no
  active `SYN` (never opened, or already `FIN`'d) is silently ignored -- there is
  no per-stream error reply for this case. Stream-level errors that *do* get
  reported are per-type (HTTP `500` SYN, `file` `{status:"error"}`, `shell`
  `{error}`); a malformed frame (fails `ParseFrame`) is logged and dropped.
- **Channel close tears everything down.** When the data channel closes, the
  session and all its streams end at once; there is no graceful per-stream
  drain. Handlers reap their own resources (e.g. the shell reaps its process,
  closes the PTY) when their stream ends or the channel drops.

### Stream-ID allocation

- **Stream 0** is reserved for control and never reused.
- Streams are **connector-initiated**. The listener does not currently
  originate streams.
- The **browser** allocates IDs sequentially from 1 (`1, 2, 3, …`;
  `bootstrap.js` `nextStreamId++`).
- The **Go CLI connector** allocates **odd** IDs from 1 (`1, 3, 5, …`;
  `client/session.go` `nextStreamID += 2`), deliberately reserving even IDs for
  a possible future device-initiated stream without needing an allocation
  negotiation.

Because only the connector opens streams today, the two schemes never collide
on a given channel.

---

## 4. Control stream (stream 0)

All control messages are JSON objects with a `type` field, carried as **`SYN`**
frames (most are `SYN|FIN` single frames). Stream 0 is handled directly by the
session, never dispatched to a stream handler (`session/control.go`,
`client/session.go`).

### Handshake flow

```
 listener (device)                         connector (browser / CLI)
        │                                            │
        │ ── verify_nonce_hash {hash} ──────────────►│  (1) first frame; connector
        │                                            │      checks hash == sha256(nonce)
        │                                            │
        │ ◄───────────── connect {path,caps,version} │  (2)
        │                                            │
   ┌── PIN set? ──┐                                  │
   │ yes          │ no                               │
   │              │                                  │
   │ ── auth_required ──────────────────────────────►│  (3a)
   │ ◄──────────────────────────── auth {pin} ───────│
   │ ── auth_result {success} ──────────────────────►│   success=false → retry (≤3),
   │        (on success, immediately followed by)     │   2s delay per failure on device
   │              │                                   │
   └──────────────┴── ready {server_version,caps} ──►│  (3b)  ← also the no-PIN path
        │                                            │
        │       (on any handler/connect failure)      │
        │ ── error {message} ────────────────────────►│
        │                                            │
        ▼                                            ▼
            stream-1+ traffic may now flow
```

### Control messages

| `type` | Dir | Flags | Fields | Purpose |
|---|---|---|---|---|
| `verify_nonce_hash` | L→C | SYN | `hash` | **First** frame after DC open. `hash = base64(sha256(nonce))`, where *nonce* is the random value the connector generated and sent to the listener in the RSA-encrypted verify payload during WebRTC setup (see `code_exchange.md`/`whitepaper.md`). Returning `sha256(nonce)` proves the listener decrypted it -- i.e. holds the private key for the UID. The connector aborts if it mismatches. |
| `connect` | C→L | SYN | `path`, `caps[]`, `version` | Open the session. `path` is the **session-level** URL path (defaults `/`) -- e.g. the HTTP proxy resolves its upstream target from it; it is *distinct* from the per-request `pathname` carried on an `http` stream (§5.1). `caps` advertises the stream types the connector can drive, but is **advisory**: the current listener ignores it (only the connector consumes the listener's `ready.caps`). `version` = `SWSPVersion`. **May be sent again** mid-session: the browser sends a fresh `connect` when the page navigates, to update `path`, and the listener answers with a fresh `ready`. The negotiated version is fixed by the first one and never changes. |
| `auth_required` | L→C | SYN | -- | Listener has a PIN; connector must authenticate before `ready`. |
| `auth` | C→L | SYN | `pin` | Connector's PIN attempt. |
| `auth_result` | L→C | SYN\|FIN | `success` | PIN verdict. `success:true` is immediately followed by `ready`; `success:false` lets the connector retry (listener pauses 2s per failure; connector caps at 3 attempts). |
| `ready` | L→C | SYN\|FIN | `server_version`, `negotiated_version`, `caps[]`, `routing` | Channel is up and authorized. `caps` is what the listener will serve (sorted). The connector's `hasCap` check gates which streams it will open. A v2 listener omits `server_version` (connector assumes 2). `negotiated_version` is the listener's `min(server_version, connect.version)` (Go and ESP32 send it; Python does not, and the connector computes the same minimum). `routing` says how the browser reads the first segment of a device path: `target-prefix` (Go proxy -- the segment names a LAN host) or `direct` (Python, ESP32 -- the whole path belongs to the device). Missing means `direct`. |
| `error` | L→C | SYN\|FIN | `message` | Connect/handler rejected; the session won't proceed. |
| `window_update` | both | SYN | `stream_id`, `max_bytes` | **v4.** Raises the **cumulative** number of payload bytes the peer may send in one direction of `stream_id`. See §6. |
| `stream_reset` | both | SYN | `stream_id`, `code`, `message` | **v4.** Terminates both directions of one stream without touching the others. See §6. |
| `video_answer` | C→L | SYN | `sdp` | Answer for the optional secondary video PeerConnection. |
| `video_candidate` | C→L | SYN | `candidate` | ICE candidate for the video PC. |

`window_update` and `stream_reset` are the only control messages, other than a
repeated `connect`/`ready`, that may arrive **after** `ready`, and the only ones
that are not part of the handshake. A v2/v3 peer never sends or receives them.

**PIN auth is optional to implement.** The Go and Python listeners support it;
the ESP32 does not, and never sends `auth_required`. A connector must handle
both.

A listener that does not recognize a stream-0 `type` ignores it (the ESP32 logs
`stream 0: <type> (ignored)`), so a newer connector's control message is not an
error to an older listener.

*(The video PC is a separate WebRTC connection negotiated over these stream-0
control frames and relayed to an external media helper; its offer/candidates
flow L→C over the same control stream. It is orthogonal to the data streams
below.)*

---

## 5. Stream types

Selected by the `type` field of the opening `SYN`. Metadata structs live in
`internal/protocol/swsp.go`; handlers in `internal/streamtype/`.

Each logical operation is exactly **one stream**: one browser `fetch()` (or
service-worker-intercepted request) = one `http` stream, one `WebSocket` = one
`websocket` stream, one `bitbang cp` transfer = one `file` stream, one terminal
= one `shell` stream.

### 5.1 `http` (also the default when `type` is omitted)

Drives the browser's service-worker HTTP proxy and `serve proxy`.

**Request** (`protocol.Request`) -- connector → listener `SYN`:

```json
{ "type": "http", "method": "GET", "pathname": "/api/x",
  "contentType": "application/json", "contentLength": 12,
  "headers": { "...": "..." } }
```

**Response** (`protocol.Response`, `BuildResponseFrames`) -- listener → connector:

```
 SYN  {"status": 200, "headers": {...}}
 DAT  <body chunk>        (repeated, each ≤ MaxChunkSize)
 FIN
```

Flow:

```
 C ── SYN {method,pathname,...} ──►   (SYN|FIN if no body, e.g. GET/HEAD)
 C ── DAT <request body> ──►          (only when there is a body)
 C ── FIN ──►
 L ── SYN {status,headers} ──►
 L ── DAT <response body chunks> ──►
 L ── FIN ──►
```

### 5.2 `websocket`

Tunnels a browser WebSocket to an upstream WS on the listener's network.

**Open** (`protocol.WebSocketOpen`) -- connector → listener `SYN`:

```json
{ "type": "websocket", "pathname": "/socket", "cookies": "a=b; c=d" }
```

Flow:

```
 C ── SYN {pathname,cookies} ──►
 L ── SYN (empty) ──►            confirm upstream WS open  (L ── FIN on failure)
 C ── DAT <message> ──►          either direction, any time
 L ── DAT <message> ──►
 ... FIN (empty) ...             either side closes
```

- Each WebSocket **message** maps to one logical SWSP message. A message larger
  than `MaxChunkSize` is fragmented (see §6) -- this is the **only** stream type
  that uses `FlagMORE`.
- `FIN` (empty payload) from either side closes the tunnel.

### 5.3 `file` -- used by `bitbang cp`

`protocol.FileOp` -- connector → listener `SYN`:

```json
{ "type": "file", "op": "get|put|list|stat|delete", "path": "/x",
  "size": 1234, "overwrite": false, "range": [0, 1023] }
```

Per-op flow (`internal/streamtype/file.go`):

```
 get:   C ── SYN {op:"get", path, range?} ──►
        L ── SYN {status:"ok", size, ...} ──►   (or SYN {status:"error",error} + FIN)
        L ── DAT <bytes> ──►  ...                (range = [start,end] inclusive, optional)
        L ── FIN ──►

 put:   C ── SYN {op:"put", path, overwrite?, size?} ──►
        L ── SYN {status:"ok"} ──►               ack: ready to receive
        C ── DAT <bytes> ──►  ...
        C ── FIN ──►
        L ── FIN {status:"ok"} ──►               (or {status:"error", error})

 list:  C ── SYN {op:"list", path} ──►
        L ── SYN {status:"ok"} ──►
        L ── DAT {entries:[FileStat,...]} ──►
        L ── FIN ──►

 stat / delete: C ── SYN {op,path} ──►  L ── SYN {status:"ok"|"error",...} + FIN
```

`FileStat`:

```json
{ "name": "app.log", "type": "file|directory", "size": 4096,
  "modified": 1718000000, "mime": "text/plain" }
```

Errors are a `SYN {status:"error","error":msg}` + `FIN`.

### 5.4 `shell`

Interactive PTY or one-shot command for `bitbang connect` and the browser
terminal.

**Open** (`shellOpen`) -- connector → listener `SYN`:

```json
{ "type": "shell", "argv": ["tail","-f","/var/log/x"], "pty": true,
  "cols": 120, "rows": 40, "env": {"TERM":"xterm"}, "cwd": "/home" }
```

`argv` empty ⇒ login/interactive shell (`$SHELL` or `/bin/sh`). `pty:false`
runs non-interactively (pipes).

**Sub-framing:** unlike the other types, shell `DAT` payloads are **tagged** --
the first byte selects the channel, the rest is the body:

| Tag | Byte | Dir | Body |
|---|---|---|---|
| stdin | `0x00` | C→L | bytes for the process stdin |
| stdout | `0x01` | L→C | stdout (also stderr in PTY mode) |
| stderr | `0x02` | L→C | stderr (pipe mode only) |
| signal | `0x03` | C→L | signal name, e.g. `"INT"`, `"TERM"`, `"HUP"` |
| resize | `0x04` | C→L | `[cols uint16 LE][rows uint16 LE]` (4 bytes) |

Flow:

```
 C ── SYN {argv,pty,cols,rows,...} ──►
 L ── DAT [0x01]<stdout> ──►  ...        process output streams as it's produced
 C ── DAT [0x00]<stdin> ──►   ...        keystrokes / piped input
 C ── DAT [0x03]"INT" ──►                out-of-band signal by name (see note below)
 C ── DAT [0x04]<cols,rows> ──►          window resize
 C ── FIN ──►                            close stdin (EOF) -- e.g. so `cat` finishes
 L ── FIN {exit_code, signal?} ──►       process exited; trailer carries the status
```

- **Signals vs. the stdin byte `0x03` (don't confuse them).** In PTY mode,
  Ctrl-C normally travels as the literal byte `0x03` (ASCII ETX) inside a
  **stdin frame** (tag `0x00`); the listener's kernel/PTY turns that byte into
  SIGINT. That is unrelated to the **signal tag** `0x03`, which is a separate
  channel for delivering a signal *by name* -- used by non-PTY clients and for
  signals that have no controlling character (SIGHUP, SIGUSR1, …). The two
  share the number `0x03` only by coincidence: one is a payload byte in a
  `0x00` (stdin) frame, the other is the frame tag itself.
- A spawn failure (bad JSON, exec error) is reported as `SYN {error:msg}` +
  `FIN` instead of output.
- The `FIN` trailer is `{"exit_code": N}` plus `{"signal": "..."}` if the
  process was killed by a signal. The CLI maps a signal exit to status 128.

### 5.5 `tcp` -- `bitbang connect -L` and `forward`

A raw TCP connection from the listener's network to one target.

**Open** (`protocol.TCPOpen`) -- connector -> listener `SYN`:

```json
{ "type": "tcp", "host": "127.0.0.1", "port": 3333 }
```

Flow (`internal/streamtype/tcp.go`):

```
 C -- SYN {host,port} -->
 L -- SYN {status:"ok"} -->              after the dial succeeds
      (or SYN|FIN {status:"error", error} -- bad request, target not allowed,
       dial failure, or the listener at its connection limit)
 C -- DAT <bytes> -->   L -- DAT <bytes> -->    a byte stream, either direction
 C -- FIN -->                            half-close: the listener shuts down its
                                         write side to the target, reads continue
 L -- FIN -->                            the target closed its side
```

- Byte-stream chunking (section 6); no tags, no `MORE`.
- **Directional EOF is preserved.** A connector `FIN` is a TCP half-close, not a
  teardown, so a protocol that sends a request and then shuts its write side
  (HTTP/1.0 clients, `nc -q`) still gets its reply. `SYN|FIN` opens and
  half-closes at once.
- The listener enforces its own allowlist: a target outside it is refused with
  the error naming the allowed forwards, whatever the connector asked for.
- Go listener only; at most 64 concurrent `tcp` streams per session by default.

### 5.6 `console` -- a device's log

A device's console: the backlog it has kept, then live output. Design and the
reasoning behind it: `device-console.md`.

**Open** -- connector -> listener `SYN`, either:

```json
{ "type": "console", "tail": 65536 }    fresh viewer: up to this much history
{ "type": "console", "since": 1048576 } resume: everything after this byte
```

The browser opens it as a WebSocket to `/__bitbang/console?tail=...`. Bootstrap
turns any WebSocket to `/__bitbang/<type>?<params>` into a SYN of `{type}` plus
the query parameters, each JSON-parsed where it parses, and passes `DAT` and
`FIN` through as raw bytes -- so a new stream type needs a listener handler and
a page, and no bootstrap change. The listener's SYN reply reaches the page as
a text message; `DAT` frames arrive as binary.

**Reply** -- listener -> connector `SYN` (not `SYN|FIN`; the stream stays open),
JSON:

```json
{ "first_seq": 1040384, "head": 1048576, "from": 1044480 }
```

`first_seq` is the oldest byte the device still holds, `head` the next it will
write, `from` where this stream starts. Byte positions are absolute and never
reused, so a reconnecting viewer resumes exactly with `since` = `from` plus the
bytes it has received.

**Data** -- listener -> connector `DAT`, tagged by the first byte:

| Tag | Body |
|---|---|
| `0x00` | history: output from before the stream opened |
| `0x01` | live output |
| `0x02` | JSON control, e.g. `{"dropped": 4096}` -- a viewer that fell behind the ring lost that many bytes, and its position moves on by the same amount |

- Byte-stream chunking (section 6); a multi-byte character may be split across frames.
- Connector -> listener `DAT` is input, framed exactly as `shell`'s input
  (section 5.4): `0x00` stdin bytes, `0x03` a signal by name (reserved;
  ignored by a device), `0x04` resize. The two directions use separate tag
  tables and never meet. A device accepts input only when its firmware opted
  in, and says so with `"input": true` in its SYN reply; otherwise, and in
  firmware before 2026-10, input frames are discarded.
- **Either side may `FIN`.** The connector does on close. The device does when
  its sends to that viewer have failed for 5 s straight, so a viewer that is in
  fact still there sees the close and reconnects instead of waiting on a stream
  nothing will write to.
- ESP32 listener only.

### Which listener serves what

| type | Go | Python | ESP32 |
|---|---|---|---|
| `http` | yes | yes | yes |
| `websocket` | yes | yes | no |
| `file` | yes | no | no |
| `shell` | yes | no | no |
| `tcp` | yes | no | no |
| `console` | no | no | yes |

`ready.caps` is how a connector learns this at runtime. The ESP32 lists
`console` only when its console is built in and has a ring to serve from
(`bitbang_swsp_advertise`); firmware from before 2026-10-02 lists only `http`
even when it serves `console`.

**Not stream types.** A device's settings (`/__bitbang/settings`) and firmware
update (`/__bitbang/ota`) are HTTP endpoints carried on ordinary `http` streams;
everything under `/__bitbang/` belongs to BitBang and is never passed to the
application. The browser's transport benchmark sends a `SYN {"bench": true}`
with no `type`; no current listener answers it, and it is a diagnostic, not part
of the protocol.

---

## 6. Chunking modes

Two distinct modes, because two kinds of payload have different boundary
semantics:

### Byte-stream chunking (http, file, shell) -- no `MORE`

HTTP/file bodies and shell output are byte streams; chunk boundaries carry no
meaning. The sender splits the payload into `DAT` frames of ≤ `MaxChunkSize`
each (`BuildResponseFrames`, file `get`, shell `pumpReader`), and the receiver
simply concatenates `DAT` payloads until `FIN`. No `MORE` flag is used; a lost
boundary would be invisible anyway.

### Message chunking (websocket) -- `MORE`

A WebSocket **message** has a boundary that must be preserved. A single WS
message larger than `MaxChunkSize` is fragmented:

```
 DAT|MORE  <chunk 1>     (0x0002)
 DAT|MORE  <chunk 2>
 ...
 DAT       <final chunk>  (no MORE)
```

`MORE` marks "this message continues"; the first non-`MORE` `DAT` ends it. This
is the only place `FlagMORE` appears; a message that fits in one frame is sent as
a plain `DAT`.

The receiver is **not** required to assemble the whole message before acting on
it. The Go implementation hands each fragment to the stream handler as it
arrives (`OnFragment`, with a `more` flag), and the WebSocket handler writes each
one straight into the upstream message writer, so no message is held in the
session itself.

### Flow control (v4)

Each stream has an independent **cumulative receive window** in each direction:
the receiver states the total number of payload bytes it will accept, and raises
that number as it consumes them. Senders block only on the affected stream.

- **Implicit initial window.** A v4 `SYN` opens both directions with
  `InitialStreamWindow` (1 MiB). It is a protocol constant, not something either
  side advertises, so the first `DAT` can follow the `SYN` immediately with no
  round trip. Sending data on a stream whose `SYN` has not gone out is a
  protocol error.
- **Cumulative, not incremental.** `max_bytes` is a running total, so a
  duplicated or reordered `window_update` is a no-op rather than extra credit.
  Updates are monotonic; a lower value is ignored.
- **Replenishment.** Once the receiver has consumed half a window
  (`StreamWindowUpdateThreshold`), it sends `window_update` with
  `consumed + InitialStreamWindow`, so the sender keeps moving instead of
  stalling for a grant.
- **`SYN` and empty `FIN` cost no credit.** Otherwise a stream that exhausted
  its window could not send the `FIN` that would close it, and teardown would
  deadlock.
- **Two receive bounds, for two different resources.** The queue caps both
  buffered payload bytes (1 MiB) and queue entries (`MaxQueuedStreamFrames`,
  256). The byte cap binds for bulk traffic; the frame cap is what stops many
  tiny or zero-length `DAT` frames, which spend no byte credit but still occupy
  entries and dispatch work.
- **Violations reset one stream.** Exceeding the window, overflowing the queue,
  or sending data after `FIN` produces a `stream_reset` for that stream. The
  session and every other stream continue.

The receive loop never blocks: frames are handed to a bounded per-stream queue
and a per-stream worker drains it. A handler that stops consuming backs up its
own stream only, which is what stops one slow HTTP target or TCP peer from
stalling an interactive shell on the same session.

### Backpressure below the window

Independently of flow control, senders watch the data channel's
`BufferedAmount` and throttle before enqueuing more frames (shell caps at 8 MB
buffered; file streaming yields similarly), so a slow reader doesn't blow up
memory on the sender. v2/v3 peers have only this, plus locally-bounded receive
queues -- they never send or receive `window_update`.

---

## 7. Versioning & compatibility

The **byte-level frame format is unchanged across v2, v3, and v4.** Every
version difference lives in stream-0 control messages and in how the endpoints
schedule work.

### v4 (current `SWSPVersion`)

> **Status, 2026-10-02.** The Go implementation has spoken v4 in both roles since
> the 0.5.0 release. The browser and the Python and ESP32 listeners are still
> v3, so any session involving them negotiates v3. Because version selection is
> per session, the transition needs no coordination -- see Negotiation below.
> Older ESP32 firmware advertises v4 without implementing it; see section 1.

Adds negotiated per-stream flow control and stream-local resets (§6):

- **`window_update`** -- cumulative per-stream receive credit, with an implicit
  1 MiB initial window opened by the `SYN` itself.
- **`stream_reset`** -- kills one stream instead of the session. Before v4 a
  protocol error on any stream took the whole session down with it.
- **Independent per-stream dispatch** -- the receive loop hands frames to a
  bounded per-stream queue rather than invoking handlers inline, so one slow
  consumer no longer stalls unrelated streams.

**Negotiation.** Both sides send their `SWSPVersion` (`connect.version`,
`ready.server_version`) and each selects `min(mine, theirs)`; a missing version
means v2. The result is fixed for the life of the session. A v4 peer talking to
a v3 peer uses the v3 data path and never emits v4 controls, so either end can
be upgraded independently -- there is no flag day, and the CLI and browser
implementations do not have to ship together.

### v3

Added, all backward-compatibly:

- **Typed SYN payloads** -- the `type` field on `SYN` metadata. v2 had only HTTP
  streams and no `type`; a missing `type` is therefore still treated as `http`.
- **Capability negotiation** -- `caps` in `connect` (C→L) and `ready` (L→C), plus
  `version`/`server_version`. A v2 listener sends a bare `{"type":"ready"}`; the
  connector infers `server_version = 2` and assumes HTTP-only.
- **New stream types** -- `file` and `shell` (and the tagged shell sub-framing).

A v3 connector talking to a v2 listener degrades to HTTP-only; a v2 connector
talking to a v3 listener still works for HTTP (untyped SYN → `http`).

---

## 8. Quick reference

```
Frame:   [StreamID u32 LE][Flags u16 LE][Length u16 LE][Payload]
Flags:   SYN 0x0001  MORE 0x0002  FIN 0x0004  DAT 0x0000   HeaderSize 8  MaxChunkSize 32768
Stream0: verify_nonce_hash → connect → (auth_required/auth/auth_result)* → ready | error
         v4, post-ready: window_update {stream_id,max_bytes} | stream_reset {stream_id,code,message}
Open:    SYN {type:"http|websocket|file|shell|tcp|console", ...}
Body:    DAT (byte streams: plain; websocket: DAT|MORE… DAT)
Close:   FIN (+ optional trailer: shell exit_code/signal, file put status)
IDs:     0 = control; connector-initiated (browser 1,2,3…; CLI odd 1,3,5…)
Flow:    v4 only. 1 MiB implicit window per stream per direction, opened by SYN.
         Cumulative max_bytes; refresh at half consumed. SYN and empty FIN are free.
         Receive queue bounded at 1 MiB and 256 frames. Violation → stream_reset.
```

Source of truth: `internal/protocol/swsp.go` (frames, flags, metadata),
`internal/session/{session,control,flow}.go` (dispatch, control, flow control),
and `internal/streamtype/{http,websocket,file,shell,tcp}.go` (per-type
behavior) in `bitbang-cli`; `web/bootstrap.js` in `bitbang-server` for the
browser connector (v3, no flow control); `bitbang/adapter.py` in
`bitbang-python`; and `bitbang_verify.c`, `bitbang_httpd.c` and
`console_stream.c` in `bitbang-esp32`. The implementation table at the top says
which is which.

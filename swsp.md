# SWSP -- Simple WebRTC Streaming Protocol

SWSP is BitBang's application-layer multiplexing protocol. It runs over the
**single** WebRTC data channel that a connector and a listener establish, and
carries every higher-level feature -- HTTP proxying, WebSocket tunneling, file
transfer, and interactive shells -- as independent logical **streams** over that
one channel.

This document is the wire-level reference: frame format, stream lifecycle, the
stream-0 control handshake, every per-type message, the two chunking modes, and
versioning. It reflects the implementation in `bitbangproxy`
(`internal/protocol`, `internal/session`, `internal/streamtype`,
`internal/client`) and the browser reference `web/bootstrap.js` in
`bitbang-server`.

> **Scope.** SWSP begins *after* the data channel is open and bidirectional
> verify has run. WebRTC/DTLS setup, signaling, and the pubkey/SAS
> authentication are out of scope here -- see `whitepaper.md` and
> `code_exchange.md`. The one exception is the `verify_nonce_hash` control
> message, which is the seam between verify and SWSP and is documented below.

---

## 1. Where SWSP sits

```
 connector  ── WebRTC data channel (DTLS-encrypted, ordered, reliable) ──  listener
     │                                                                          │
     │   one channel, many streams, multiplexed by SWSP:                        │
     │     stream 0  → control (handshake, auth, ready, video negotiation)      │
     │     stream 1+ → http / websocket / file / shell                          │
```

One **session** = one data channel = one SWSP instance. The listener side is
`internal/session.Session`; the connector side is `internal/client.Session` and
`bootstrap.js`'s `BitBangConnection`.

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
  handler (`http`, `websocket`, `file`, `shell`). A missing `type` defaults to
  `http` (v2 back-compat).
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
| `connect` | C→L | SYN | `path`, `caps[]`, `version` | Open the session. `path` is the **session-level** URL path (defaults `/`) -- e.g. the HTTP proxy resolves its upstream target from it; it is *distinct* from the per-request `pathname` carried on an `http` stream (§5.1). `caps` advertises the stream types the connector can drive, but is **advisory**: the current listener ignores it (only the connector consumes the listener's `ready.caps`). `version` = `SWSPVersion`. |
| `auth_required` | L→C | SYN | -- | Listener has a PIN; connector must authenticate before `ready`. |
| `auth` | C→L | SYN | `pin` | Connector's PIN attempt. |
| `auth_result` | L→C | SYN\|FIN | `success` | PIN verdict. `success:true` is immediately followed by `ready`; `success:false` lets the connector retry (listener pauses 2s per failure; connector caps at 3 attempts). |
| `ready` | L→C | SYN\|FIN | `server_version`, `caps[]` | Channel is up and authorized. `caps` is what the listener will serve (sorted). The connector's `hasCap` check gates which streams it will open. A v2 listener omits `server_version` (connector assumes 2). |
| `error` | L→C | SYN\|FIN | `message` | Connect/handler rejected; the session won't proceed. |
| `window_update` | both | SYN | `stream_id`, `max_bytes` | **v4.** Raises the **cumulative** number of payload bytes the peer may send in one direction of `stream_id`. See §6. |
| `stream_reset` | both | SYN | `stream_id`, `code`, `message` | **v4.** Terminates both directions of one stream without touching the others. See §6. |
| `video_answer` | C→L | SYN | `sdp` | Answer for the optional secondary video PeerConnection. |
| `video_candidate` | C→L | SYN | `candidate` | ICE candidate for the video PC. |

`window_update` and `stream_reset` are the only control messages that may arrive
**after** `ready`, and the only ones that are not part of the handshake. A v2/v3
peer never sends or receives them.

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

> **Status.** v4 is on `main` in the Go implementation but has not yet appeared
> in a tagged release, and the browser runtime is still being rolled out.
> Released peers negotiate v3 until both land. Because version selection is per
> session, the transition needs no coordination -- see Negotiation below.

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
Open:    SYN {type:"http|websocket|file|shell|tcp", ...}
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
behavior), with `web/bootstrap.js` and `web/flow-control.js` as the browser
reference implementation.

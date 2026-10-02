# Signaling -- the WebSocket to bitbang-server

Before there is a data channel there is signaling: each side holds a WebSocket
to the BitBang server, and the server relays what the two need to set up a
WebRTC connection between them. Once that connection is up, signaling carries
nothing of the session -- everything after it is SWSP (`swsp.md`) or media
channels (`av-streaming-api.md`).

This is the message reference. *Why* the server can relay without being
trusted is `trustless-signaling.md`; the pairing protocol in full, including the
part that runs on the data channel, is `code_exchange.md`. Audited against the
server (`bitbang-server`: `internal/wire/messages.go`, `internal/handler/`), the
browser (`web/bootstrap.js`), the Go CLI (`internal/signaling`,
`internal/client`), Python (`bitbang/adapter.py`) and the ESP32
(`bitbang_signaling`) on 2026-10-02.

---

## 1. Shape

- **One JSON object per WebSocket text message**, discriminated by `type`.
- **Unknown fields are tolerated** on every inbound message, which is how fields
  are added without a version bump.
- **Unknown types are ignored**, with a log line: the server logs and drops
  them; the browser falls through its dispatch; the CLI logs them under `-v`. A
  new message type is therefore safe to deploy ahead of the clients that
  understand it, and every type added since v3 has relied on that.
- **Roles.** The **listener** (a device, `bitbang serve`, a Python app) waits to
  be reached. A **connector** (the browser, `bitbang connect`) reaches it. The
  listener is always the WebRTC offerer.

### Identity

A listener is named by its **UID**: `base64url(SHA-256(public key DER)[:16])`,
22 characters, no padding. The server checks it at registration
(`identity.UIDFromPublicKeyBytes`), so a UID cannot be claimed without the key.
Connectors check it again, against the key the server hands them (section 3) -- the
server is never trusted to have checked.

The **access code** (11 base64url characters, from the URL fragment) never
travels in clear through signaling. It rides inside `encrypted_request` (section 3),
which only the listener can read.

### Endpoints

| Path | Who | For |
|---|---|---|
| `wss://<server>/ws/device/<uid>` | listener | registering, and every session's setup |
| `wss://<server>/ws/client/<uid>` | connector | reaching a listener whose UID it knows |
| `wss://<server>/ws/pair` | connector | reaching a listener by 6-digit pairing code |

---

## 2. The listener

```
 listener                                  server
    | -- register {protocol, public_key, ...} ->|
    | <---- registered {code?, versions?} ------|   or error {code, message}, then close
    |                                           |
    | <---- request / pair_request -------------|   one per connector, see section 3, section 4
    | -- renew_code --------------------------->|   any time
    | <---- code_issued {code} -----------------|
```

**`register`** -- the first message, and the server closes the socket on
anything else (`expected_register`).

| field | |
|---|---|
| `protocol` | registration protocol version, currently 3. Below `MinProtocolVersion` (3) is refused with `protocol_too_old`. Independent of the SWSP version (`swsp.md` section 1). |
| `public_key` | base64 DER SubjectPublicKeyInfo. Its hash must equal the `<uid>` in the path (`uid_key_mismatch`). |
| `ice_servers` | optional: the operator's own STUN/TURN, which then wins over the server's for every session to this listener. |
| `want_code` | optional: ask for a 6-digit pairing code. Without it the listener is reachable by UID only. |
| `boot` | optional: an opaque id minted once per boot of the firmware. The server passes it to connectors (`device_up`, `offer.device_boot`), which reload their page when it changes and do nothing when it does not -- a reconnect is not a restart. Omitted means "cannot say". |

**`registered`** -- `code` when one was asked for and pairing is enabled;
`versions`, a map of product to newest release (`{"cli": "0.5.1"}`), when the
server tracks any. A client looks up its own row and says nothing if there is
none.

**`renew_code`** / **`code_issued`** -- a pairing code lives five minutes.
`renew_code` returns the live code, or mints a new one past its lifetime. Only
answered for a listener that registered with `want_code`; `code` is empty
otherwise. A separate type from `registered` so that it can only ever follow a
request: older listeners treat a `registered` outside registration as an error.

**Keepalive.** The server pings every 60 s and drops a listener that has not
answered for 300 s.

**One connection per UID.** A second `register` for a UID that is already
connected wins: the old socket gets `error {code: "preempted"}` and is closed.

---

## 3. A session, by UID

```
 connector                      server                              listener
    | -- WS /ws/client/<uid> -->|
    | <-- hello {versions} -----|
    | <-- build_stamp {build} --|   (browser runtime build)
    | -- request -------------->| -- request + client_id, ice_servers, browser_ip -->|
    |                           | <-- offer {client_id, sdp, streams} ---------------|
    | <-- offer + ice_servers,  |
    |     turn_unavailable,     |
    |     device_pubkey,        |
    |     device_boot ----------|
    |  check hash(device_pubkey) == uid
    | -- answer {sdp, encrypted_request} ->| -- answer + client_id --------------->|
    | <--- candidate --- (both ways, relayed, keyed by client_id) --- candidate --->|
    |                                                                               |
    | ========== WebRTC up; the listener's first SWSP frame is verify_nonce_hash ===|
    | -- connection_path {path} ->|   telemetry, connector only
```

**`hello`** -- the first thing the server writes on a connector socket, before
the connector has said anything: the same `versions` table `registered`
carries. Unconditional, so it reveals nothing about whether the UID exists.

**`build_stamp`** -- the build of the browser runtime the server is serving.
Sent on connect and whenever it changes; a page whose own build differs offers
a reload.

**`request`** -- connector -> server: `{uid, force_relay}`. The server adds:

- `client_id`, which it stamps on **every** message from this connector, and
  by which the listener addresses every reply;
- `ice_servers` for the listener: STUN only (the listener never gets
  server-managed TURN; relaying is the connector's job), or the listener's own
  servers if it registered some;
- `browser_ip`, overwriting anything the connector sent, so the listener can
  attribute bad access codes.

`force_relay` (the `!relay` URL flag, the CLI's `--relay`) makes the connector
use its relay candidate at once rather than after the direct attempt.

**`offer`** -- listener -> connector: `sdp`, `client_id`, and optionally
`streams` (a map of media section `mid` to stream name, for native media tracks)
and `device_name`. The server stamps on the way through:

| field | |
|---|---|
| `ice_servers` | the connector's servers, TURN credentials included, capacity-gated |
| `turn_unavailable` | true when TURN was wanted and none could be allocated |
| `device_pubkey` | the key the listener registered with |
| `device_boot` | the listener's `boot`, so a connector that missed `device_up` still learns it |

One offer per session. TURN is stamped up front and the connector delays
trickling its relay candidate, so a direct path settles first with no ICE
restart (*single-phase ICE*). A second offer on an established connection is
ignored.

**The connector checks `hash(device_pubkey) == uid` before answering.** A
mismatch means the server substituted the key, and the connector stops.

**`answer`** -- connector -> listener: `sdp`, and **`encrypted_request`**:
base64 of RSA-OAEP (SHA-256, no label) over the JSON

```json
{ "fingerprint": "<the connector's DTLS fingerprint>",
  "nonce": "<base64 random bytes>",
  "code": "<the access code from the URL fragment>" }
```

The listener decrypts it and refuses the session unless `fingerprint` matches
the DTLS peer it actually has and `code` matches its access code. It then
proves it could decrypt by sending `base64(sha256(nonce))` as its first SWSP
frame, `verify_nonce_hash` (`swsp.md` section 4), which the connector checks before
sending anything. That exchange is what makes the server unable to sit in the
middle (`trustless-signaling.md`).

**`candidate`** -- ICE candidates, both directions, relayed verbatim apart from
`client_id`. `candidate` is the browser's `RTCIceCandidate` JSON
(`candidate`, `sdpMid`, `sdpMLineIndex`).

**`connection_path`** -- connector -> server, fire and forget, once per ICE
establishment: `path` is `direct`, `relay`, `tcp-relay` or `failed` (with an
optional `reason`). Telemetry only, and connector-only by contract, so no
connection is counted twice.

**`device_up`** -- server -> connector, whenever a listener registers for the
UID the connector is attached to, with that listener's `boot`. A different boot
than the one the page was talking to means the device restarted and the page
reloads; the same boot means signaling reconnected under a device that never
stopped. There is deliberately no `device_down`.

**A listener that goes away mid-session** does not close its connectors'
sockets. The server holds them for up to five minutes -- long enough for a
firmware update, a reboot and a re-register -- and drops what cannot be
forwarded meanwhile. Past that it sends `error {code: "device_not_found"}`.

---

## 4. A session, by pairing code

The same setup, reached by code instead of UID, on `/ws/pair`:

```
 connector -- pair_init {code, force_relay} --> server
 connector <-- pair_routed -------------------  (or error unknown_code)
 server -- pair_request {client_id, remote_ip, ice_servers} --> listener
 ... offer / answer / candidate, exactly as in section 3 ...
 listener -- pair_approved | pair_rejected {reason} --> server --> connector
```

- The code lookup takes 3 s whether or not it succeeds, so codes cannot be
  probed faster than that.
- A listener treats `pair_request` like `request`, but runs the SAS comparison
  on the data channel before anything else.
- `pair_approved` is a bare acknowledgement. The credentials -- UID, public
  key, access code -- go over the data channel, never through the server.
  `pair_rejected` carries `reason`: `sas_mismatch`, `user_declined` or
  `timeout`.

The data-channel half (`pair_commit`, `pair_challenge`, `pair_reveal`,
`pair_credentials`) is in `code_exchange.md`, *Wire protocol*.

---

## 5. Errors

**From the server:** `error {code, message}`. `code` is the contract --
snake case, and never reworded. `message` is for a person or a log, and for
every error that predates `code` it still holds the old token, because older
clients match on it.

| code | when |
|---|---|
| `expected_register` | a listener's first message was not `register` |
| `protocol_too_old` | `register.protocol` below 3 |
| `missing_public_key`, `invalid_public_key`, `rejected_key` | the key is absent, does not parse, or is a type or size not accepted |
| `invalid_uid`, `uid_key_mismatch` | the path's UID is malformed, or is not the key's hash |
| `preempted` | another connection registered this UID |
| `device_not_found` | no listener for that UID, or it stayed away past the grace period |
| `unknown_code` | a pairing code not found or expired |

**From the listener,** forwarded verbatim: `error {client_id, message}`, where
`message` is a short code the connector maps to wording it owns. Today:
`device_busy` -- every session slot is taken. An unrecognized code gets the
connector's generic failure, so a listener cannot put arbitrary text on
someone's screen.

---

## 6. Message reference

| type | direction | endpoint | fields |
|---|---|---|---|
| `register` | L -> S | device | `protocol`, `public_key`, `ice_servers?`, `want_code?`, `boot?` |
| `registered` | S -> L | device | `code?`, `versions?` |
| `renew_code` | L -> S | device | -- |
| `code_issued` | S -> L | device | `code` |
| `hello` | S -> C | client, pair | `versions?` |
| `build_stamp` | S -> C | client | `build` |
| `device_up` | S -> C | client | `boot?` |
| `request` | C -> S -> L | client, device | `uid`, `force_relay`; + `client_id`, `ice_servers`, `browser_ip` |
| `pair_init` | C -> S | pair | `code`, `force_relay?` |
| `pair_routed` | S -> C | pair | -- |
| `pair_request` | S -> L | device | `client_id`, `remote_ip?`, `ice_servers?` |
| `offer` | L -> S -> C | device, client/pair | `client_id`, `sdp`, `streams?`, `device_name?`; + `ice_servers`, `turn_unavailable`, `device_pubkey`, `device_boot` |
| `answer` | C -> S -> L | client/pair, device | `sdp`, `encrypted_request`; + `client_id` |
| `candidate` | both | all | `candidate`; + `client_id` |
| `connection_path` | C -> S | client | `path`, `reason?` |
| `pair_approved` | L -> S -> C | device, pair | `client_id` |
| `pair_rejected` | L -> S -> C | device, pair | `client_id`, `reason` |
| `error` | S -> any; L -> S -> C | all | server: `code`, `message`; listener: `client_id`, `message` |

L listener, S server, C connector. `+` marks fields the server adds in transit.

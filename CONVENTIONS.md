# BitBang naming conventions

This document defines the canonical vocabulary used across BitBang's code, docs, and protocol descriptions. It exists because the same concepts are referred to by several names in different places (whitepaper, code comments, package names, log lines), and the resulting interpretation tax adds up over time.

Use the **canonical** term in new code and docs. The **discouraged synonyms** are listed so contributors recognize them in older code without picking them up for new work.

---

## The two sides of a session

Every BitBang session involves exactly two endpoints that play distinct roles. The roles are stable for the lifetime of a session; only one side is reached on a given session-establishment.

### Listener

The side that **runs first** and **waits** for a session to be initiated against it. This is the role-level term — use it whenever the *role* matters more than the specific implementation.

- Runs `bitbang serve …`, `bitbang fileshare`, or `bitbang proxy` from the CLI.
- Wraps a WSGI/ASGI app via the `bitbang-python` adapter.
- Is reachable at a URL of the form `https://<server>/<UID>#<access_code>`.
- Holds a private key; performs the listener-side of bidirectional verify.
- Implementations: `bitbangproxy` (Go binary), `bitbang-python` (Python library), Pixy / Goby firmware.

**Discouraged synonyms in new code:** "server" (collides with signaling server), "host" (collides with HTTP/WebRTC host candidates), "service" (overly generic).

### Device

A specific kind of listener — one that runs on **dedicated hardware** rather than on a general-purpose computer. ESP32-class microcontrollers, Pixy / Goby firmware, IoT sensors, embedded cameras. Distinct from a "listener" running on a workstation (`bitbang serve` on someone's laptop is a listener, not a device).

"Device" is the right term whenever the embedded/IoT character is what matters — onboarding flows, power-constrained design, firmware updates, the IoT-platform comparison story in the whitepaper. The whitepaper uses "device" pervasively for this reason.

Reserve "device" for the hardware case. When you mean "the side that accepted the connection regardless of whether it's hardware or software," use "listener" instead. Mixing them up makes the wire-level "/ws/device/<uid>" endpoint name feel arbitrary — it's actually the listener endpoint, with a name that predates the role/implementation split.

**Wire-level legacy:** `/ws/device/<uid>`, the `device_pubkey` field in offers, and `registry.DeviceConn` in the signaling server all use "device" historically. These names are stable and shouldn't churn; just be aware they mean "listener" at the role level. Future renames could clean this up but aren't pressing.

### Connector

The side that **opens** a URL or **types** a pair code and **initiates** the WebRTC handshake. This is the role-level term — use it whenever the *role* matters more than the specific implementation.

- Opens `https://bitba.ng/<UID>#<code>` in a browser.
- Runs `bitbang connect <URL>` or `bitbang connect <6-digit-code>` from the CLI.
- Calls `bitbang cp <URL>:/path /local` from the CLI.
- Sends the `request` or `pair_init` message to the signaling server.
- Performs the connector-side of bidirectional verify.
- Implementations: bootstrap.js (browser runtime), `bitbangproxy` (Go connect/cp/pair flows), future Python connector if added.

**Discouraged synonyms in new code:** "client" (used historically, still appears in some package names like `internal/client/`, but ambiguous because *every* WebSocket connection is technically a client of the signaling server), "peer" (acceptable inside WebRTC contexts but vague at the application level).

### Browser

A specific kind of connector — one that runs as **JavaScript inside a web browser** rather than as a native CLI process. Loads bootstrap.js from the signaling server, exposes the proxied app inside an iframe, runs WebRTC in the browser's native stack.

"Browser" is the right term whenever the in-browser character is what matters — UI affordances, SubtleCrypto, getStats() shape, the "no install on the viewing side" property, the `/<UID>` URL flow as opened from a clickable link. The README's user-facing copy says "browser" pervasively for this reason.

Reserve "browser" for the in-browser case. When you mean "the side that initiated the connection regardless of whether it's a browser or a CLI," use "connector" instead. The CLI's `bitbang connect` does most of the same work the browser does (offer/answer, ICE trickle, bidirectional verify, `connection_path` telemetry) — calling it "the browser" in that context obscures the CLI case.

**Wire-level legacy:** `browser_ip` on relayed request messages and the `internal/client/` package name use "browser" / "client" historically for what we'd now call the connector role. These names are stable and shouldn't churn; just be aware they mean "connector" at the role level. (The newer `remote_ip` on pair_request was named without the "browser" presumption — that's the pattern for any new field.)

### Why this split matters

Several wire-protocol contracts depend on knowing which side is which:

- The `connection_path` telemetry message is sent **only by the connector**; listeners never report. (See `internal/wire/messages.go` in `bitbang-server`.)
- Bidirectional verify is asymmetric: the connector sends an RSA-OAEP-encrypted `{fingerprint, nonce, code}` payload; the listener proves possession by replying with `sha256(nonce)` over DTLS.
- Pair-code SAS: the **connector** displays the SAS; the **listener** types it.

If a piece of code routinely runs on both sides, name it carefully or refactor so the listener vs. connector path is obvious from the structure.

---

## The two third parties

These are infrastructure, never endpoints of a session.

### Signaling server

The thing that **brokers** the introduction between a listener and a connector but is never on the data path.

- Hosted at `bitba.ng` for the open instance; self-hostable from `bitbang-server`.
- Endpoints: `/ws/device/<uid>` (listener registers), `/ws/client/<uid>` (connector requests), `/ws/pair` (pair-code initiation), `/status` (metrics), `/<UID>` and `/` (serves bootstrap.js).
- Holds no secrets, no application data, no credentials. A full breach exposes nothing exploitable.
- Implementations: `bitbang-server` (Go), with a Python predecessor referenced as `signaling.py`.

**Discouraged synonyms in new code:** "broker" (acceptable in protocol descriptions; the whitepaper uses it freely; but in code "signaling" is more specific), "the server" (too ambiguous given the listener-side `bitbang serve` and the TURN server).

### TURN server

The relay infrastructure used when direct peer-to-peer connectivity fails.

- Standard coturn instance. BitBang treats it as opaque infrastructure with one knob (HMAC secret) and never sees its internals.
- Hosted at `turn.bitba.ng` for the open instance.
- Mentioned in code as "TURN" or "coturn"; the TURN credential and capacity logic lives in `bitbang-server/internal/turn/coturn.go`.

**Discouraged synonyms in new code:** "relay" (ambiguous because the signaling server *also* relays — signaling messages, not media), "STUN/TURN" (STUN is a different thing; reserve the combined term for explicit context).

---

## URL and credential terms

### URL form

The canonical shareable identifier: `https://<signaling-host>/<UID>#<access_code>`.

- The `<UID>` part is what the signaling server sees in the path.
- The `<access_code>` part is the fragment — never sent to the signaling server by spec-compliant browsers.
- Bare `<UID>#<access_code>` (no host) is accepted by `bitbang connect`, which defaults to `bitba.ng`.

### UID

The listener's globally-unique 128-bit identifier, encoded as 22 base64url characters (no padding).

- `UID = base64url(sha256(device_public_key_DER)[:16])`.
- Self-certifying: the connector can verify the listener's public key by re-hashing.
- Stable across listener restarts so long as the keypair is persisted.

**Discouraged synonyms in new code:** "device ID", "client ID" (the signaling server already uses "client_id" for something else — a per-session routing handle).

### access_code

The 64-bit authorization secret carried in the URL fragment, encoded as 11 base64url characters (no padding).

- Used by the listener to authorize a session — without it, even a connector that knows the UID can't establish a session.
- Lives only in the URL fragment, in the connector's local storage, and in the encrypted bidirectional-verify payload. The signaling server never sees it.

**Discouraged synonyms in new code:** "PIN" (used for a separate, optional, 4-digit second factor — see `internal/auth/`), "secret" (overly generic), "token" (collides with capability tokens in the network-mode design).

### Pair code (or code A)

The 6-digit short numeric code issued by the signaling server when a listener opts into the code-exchange flow with `want_code: true`.

- Time-limited: expires 5 minutes after issuance.
- Used to bootstrap an initial pairing without requiring the full 22-character UID up front.
- Distinct from `access_code`; pair code is consumed during the pairing flow and discarded, access_code is the long-lived authorization.

**Code A** is the precise term from the design doc (`code_exchange.md`); "pair code" is the user-facing term. Both are acceptable in code comments; use "code A" only when explicitly distinguishing from code B.

### SAS (or code B)

The 4-digit Short Authentication String computed independently on both peers from the negotiated DTLS fingerprints.

- The connector **displays** it; the listener does **not** see it on screen.
- The listener's operator types the SAS after hearing it from the connector.
- BitBang compares typed value to its internally-computed SAS; mismatch rejects the pairing.

**Code B** is the precise term from `code_exchange.md`; "SAS" is the standard cryptographic term used elsewhere (ZRTP, Bluetooth, etc.).

---

## Session-level terms

### Session

One BitBang connection from a connector to a listener, beginning when the connector sends `request` and ending when either side closes.

- Has exactly one connector and one listener.
- Backed by exactly one WebRTC peer connection.
- One signaling-server `request` ↔ one session ↔ one connection_path report.

### Peer connection (PC)

The WebRTC `RTCPeerConnection` (or pion `*webrtc.PeerConnection`) that carries a session's encrypted data.

Use "PC" in code comments and log lines; spell it out as "peer connection" in prose.

### Data channel (DC)

The WebRTC `RTCDataChannel` that carries SWSP-framed bytes inside a peer connection.

Use "DC" in code comments and log lines; spell it out as "data channel" in prose. There is exactly one DC per BitBang PC (named `"http"` historically; the name has no protocol meaning).

### Code-path terms

When a function or package is specifically the listener-side or connector-side of something, label it explicitly:

- **"listener-side"** — code that runs in the bitbangproxy listener / bitbang-python adapter
- **"connector-side"** — code that runs in the bitbangproxy CLI / bootstrap.js
- **"signaling-side"** — code that runs in bitbang-server

These adjectives are unambiguous regardless of which package the code lives in.

---

## Package-to-role mapping

For the `bitbangproxy` repo specifically (which contains both listener and connector code), here is which packages play which role:

| Package | Role | Note |
|---|---|---|
| `cmd/bitbang/serve.go` | Listener entrypoint | `bitbang serve …` subcommands |
| `cmd/bitbang/cp.go`, `cmd/bitbang/connect.go` | Connector entrypoint | `bitbang cp …` and `bitbang connect …` |
| `cmd/bitbang/connect_pair.go` | Connector (pair flow) | `bitbang connect <6-digit>` |
| `cmd/bitbangbench/` | Listener entrypoint | Benchmarking harness |
| `internal/peer/` | Listener-side WebRTC | Includes connection.go, verify.go, sas.go |
| `internal/client/` | Connector-side WebRTC | URL-flow `Dial()` lives here |
| `internal/signaling/` | Listener-side signaling client | Connects to `/ws/device/<uid>` |
| `internal/pairing/` | Listener-side pair handling | SAS prompt + retry logic |
| `internal/session/` | Listener-side session lifecycle | SWSP stream-0 control, handler dispatch |
| `internal/streamtype/` | Listener-side stream handlers | Shell, file, HTTP, WebSocket |
| `internal/identity/` | Shared | Key generation + UID derivation |
| `internal/protocol/` | Shared | SWSP framing |

The naming is historically uneven — `internal/peer/` and `internal/client/` both contain WebRTC code that uses pion, but the *role* of each is different (listener vs connector). Read the package docstring for the canonical answer. Future refactors may rename for clarity; until then, prefer the role-explicit terms in comments ("listener-side", "connector-side") over the package names ("peer", "client").

---

## When in doubt

If you're writing code or docs that uses one of these terms and it isn't obvious which role you mean, prefer the role-explicit adjective form ("listener-side stats endpoint", "connector-side encrypted_request payload") over the bare noun ("client stats endpoint", "device payload"). The bare nouns are fine when role is already established by surrounding context; the adjective form is fine always.

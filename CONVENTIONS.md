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

The canonical shareable identifier for a device is `https://<signaling-host>/<UID>#<access_code>`. To address a path *within* the device, the device's own URL is appended **verbatim after the access code (and after any flag section)**, inside the fragment:

```
https://<host>/<UID>#<access_code>[!<flag-list>][<device-URL>]
              └path┘└─────────────────fragment────────────────┘
```

- `<device-URL>` is the proxied app's own URL exactly as it appears locally: `/<target>/path?query#hash` — e.g. `/localhost:8096/web/#/livetv?collectionType=livetv`.
- **Everything device-specific lives in the fragment.** The signaling server only ever sees `GET /<UID>` — target, path, query, SPA route, *and Bitbang flags* are never transmitted, even on refresh (fragments aren't sent in HTTP requests). By design, not incidental. (The path was never relayed to the device by the server anyway — it reaches the device over the data channel; this just removes the server's incidental *visibility* of it.)
- The access_code alphabet is base64url (`[A-Za-z0-9_-]`, no `/ ? #`), so the first such character unambiguously ends the code. What follows the code is (in order) an optional `!<flag-list>` segment and then the optional device URL — the boundary between the two is exactly one character: everything up to the first `/` in the fragment (after the code) is Bitbang metadata; everything from `/` onward is device content, forwarded verbatim.
- **Flags** (`debug`, `relay`, `norelay`, `nocookiejar`, `msg_timeout=N`) live in the fragment immediately after the access code, separated from the code by `!` and from each other by `,`, with `=` for value flags — e.g. `#<code>!debug,msg_timeout=3.5`. Read via `location.hash`. The server sees none of them; only the UID is transmitted. `!` starts the flag-list; `,` separates flags; `=` separates a flag from its value. Literal `!`, `?`, `#` inside `<device-URL>` are unambiguous because the device URL only begins after a `/`, so the flag-list section is closed by then.
- No shorthand/canonical distinction and no URL rearrangement — the typed URL is already canonical.
- Bare `<UID>#<access_code>` (no host) is accepted by `bitbang connect`, which defaults to `bitba.ng`. A bare `host:port` target gets a trailing slash so the app's relative URLs resolve from its root.

**Superseded:** an earlier scheme put the path in the real pathname (`/<UID>/path#<code>`) with a `#<code>/path`→`/path#<code>` rewrite; that and its BFCache/hashchange canonicalization were removed.

### In-page navigation & address bar

`bootstrap.js` renders the proxied app in a same-origin iframe (`/__device__/<sessionId>/…`) and keeps the top-level address bar in sync so refresh and bookmark restore the exact view. bootstrap.js-only — no signaling-server, service-worker, SWSP, or device change.

- **Capture (iframe → bar):** on every in-iframe navigation, `syncTopURL` mirrors the iframe's location into the fragment via `history.replaceState`. Navigation is observed via `hashchange`/`popstate` listeners **and** a `pushState`/`replaceState` wrap on the iframe's `history` — SPAs (e.g. Jellyfin via React-Router) navigate with `pushState`, which fires neither event.
- **Absolute-path apps:** apps that navigate with origin-absolute paths leave the `/__device__/<sid>/<target>/` prefix entirely (the iframe ends up at bare `/web/#/…`, target gone). The target is remembered at connect time (`this.target`) and re-prepended when reconstructing the device path. `iframeDeviceURL()` is the single normaliser; `parseDeviceURL()` must produce matching output.
- **Manual edit (bar → iframe):** the top-frame `hashchange` handler does `location.reload()` — a fresh bootstrap re-parses the device URL and rebuilds the iframe with an intact `/__device__/` referer chain. In-place re-pointing was tried and abandoned: navigating the iframe to the prefixed URL makes the SPA redirect to a bare origin path the SW can't re-route, which loads `bootstrap.html` *into* the iframe → blank hang. Reload is heavier but robust; manual edits are rare. The discriminator between a manual edit and a back/forward is `iframeShowsTopURL()` (is the iframe already displaying what the bar says?), **not** the event type — a manual address-bar edit fires `popstate` *as well as* `hashchange`, so popstate cannot be used to detect back/forward.
- **Back/forward:** the iframe's own `pushState` entries populate the joint session history, so they work by emergence — the browser restores the iframe and fires its `popstate`; `syncTopURL` re-mirrors the bar. The accompanying top-frame `hashchange` is suppressed (no reload) via `iframeShowsTopURL()`, which reads the iframe's already-restored location and so is independent of the popstate-vs-hashchange firing order.
- **Sticky session for `target=_blank` links:** a proxied app that opens an absolute path in a *new tab* (e.g. OctoPrint's Reverse Proxy Test → `/reverse_proxy_test/`) would otherwise open a bare `bitba.ng/<path>` URL with **no `<uid>#<code>`** — that tab has no device identity or access code and can't connect, and the SW can't recover it (no referer, `noopener`, and the path collides with the server short-path namespace → it serves `bootstrap.html`). `interceptNewTabLinks` (a capturing `click` listener on the same-origin iframe document) catches clicks on `target="_blank"` anchors, cancels the bare open, and `window.open`s the bitbang-form URL `/<uid>#<code>/<target><path>` instead — the new tab boots, connects to the **same** device (a second concurrent viewer session, which the sharing model already supports), and lands on the path. The identity must be injected **before** the tab opens; there is no SW-only fix (a bare new tab has no fragment to read). Reuses the same `iframeDeviceURL` path-normalisation. Currently covers left-click on same-origin anchors; JS `window.open`, middle/ctrl-click, and `target="_blank"` form submits are not yet handled.
- **Invariant:** the top frame must **never** create its own hash history for in-app nav (no top-level `pushState`, `location.hash = …`, or `location.href = '…#…'`). The reload-vs-restore discrimination assumes in-app nav uses `replaceState` only. (The proxy-form "Go" button intentionally uses `location.href` for a *target switch*, which correctly reloads.)
- **`replaceState`, not `pushState`,** for the mirror: adds no history entry (no double-counting) and does not fire `hashchange` (no reload loop).

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

### Identity scope — which UID a listener uses  **[implemented — bitbang-cli]**

A UID is per-keypair, and the bitbang-cli derives **which** identity (keypair → UID → URL) to use from the listener's **access scope**, so distinct tasks on one machine get distinct, scope-limited URLs. Logic: `deriveProgram` in `cmd/bitbang/program.go`; keypair persisted at `~/.bitbang/<program>/identity.pem`.

- **Shell-bearing configs collapse to the master `bitbang` identity** (`serve` = shell+files+proxy, and `serve shell`). Shell is the most permissive cap, so a combined listener is no less dangerous than shell alone — one URL — and it preserves the pre-existing URL + the legacy-alias migration (`bitbang-shell`/`bitbang-fileshare`/`bitbangproxy`).
- **Each single non-shell cap gets its own stable UID, per instance.** A fixed proxy target (`serve proxy localhost:8096`) and a shared file path each derive a distinct UID; the generic, instance-less form (`serve proxy` = "proxy anything") is its own identity too (and is more powerful — warn like shell). So three proxies to three targets are three separate, coexisting URLs, each granting only that one task. Pairs well with fixed-target proxy mode, where the **bare** URL serves the app directly → `bitba.ng/<uid>#<code>` *is* that one app, nothing else reachable.
- **Program name = readable slug + `sha256(normalized)[:6]`** (e.g. `proxy-localhost-8096-a1b2c3`). The hash makes the name filesystem/Windows-safe and ensures sanitization can't alias two distinct inputs; instance values are normalized first (host:port lower-cased + trailing-slash-stripped; file paths abs+cleaned) so trivially-equivalent inputs share one UID.
- **`--program <name>` overrides** the derivation (used by embedders, e.g. the OctoPrint plugin, to pin/share an identity).
- **Per-identity lock.** A listener holds an advisory flock on `~/.bitbang/<program>/lock` for its lifetime (`cmd/bitbang/lock_unix.go`). A second local process with the **same** scope refuses with a clear message instead of silently preempting the first at the signaling server (one device connection per UID). The OS releases it on exit, so a crash never leaves it stale and a same-process reconnect is unaffected. Unix-only today; non-unix is a no-op stub (`lock_other.go`) — moot while the shell cap keeps the binary Unix-only anyway.

**Not implemented yet:** serial-port and generic port-forward modes (`deriveProgram` has the shape — add a `case` each: `serial-<port>`, `forward-<spec>`). No identity management command yet — to reset an identity, delete `~/.bitbang/<program>/` (regenerates a fresh UID+code on next run); rotating just the access code while keeping the UID isn't wired up.

### Pair code (or code A)

The 6-digit short numeric code issued by the signaling server when a listener opts into the code-exchange flow with `want_code: true`.

- Time-limited: expires 5 minutes after issuance.
- Used to bootstrap an initial pairing without requiring the full 22-character UID up front.
- Distinct from `access_code`; pair code is consumed during the pairing flow and discarded, access_code is the long-lived authorization.

**Code A** is the precise term from the design doc (`code_exchange.md`); "pair code" is the user-facing term. Both are acceptable in code comments; use "code A" only when explicitly distinguishing from code B.

### SAS (or code B)

The 6-digit Short Authentication String computed independently on both peers from two committed nonces and the negotiated DTLS fingerprints. The commit→challenge→reveal exchange makes the short code non-grindable; see `code_exchange.md`.

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

---

# Connectivity & relay architecture

The vocabulary above names the *roles*. This section documents how a session is actually **established** over WebRTC: the single-phase ICE strategy, which endpoint drives the relay decision, the relay roles, and how the design keeps embedded devices simple. These are architectural conventions, not naming ones — but they belong here because the behavior differs sharply by endpoint type (device vs. browser vs. CLI connector), and getting that wrong is a common source of confusion.

> **Status labels.** Each subsection is marked **[implemented]**, **[implemented — note]** (works, with a caveat worth knowing), **[proposed]** (a decision we've made but not yet built), or **[superseded]** (the old design, kept for context — *not* current code). Don't read a [proposed] or [superseded] block as describing current code.

## Single-phase ICE: allocate up front, bias toward direct  **[implemented]**

A session connects in **one phase** — no fallback round trip, no `ice_restart`. (This replaced an older two-tier scheme; see *The superseded two-tier scheme* below.)

- The **connector** is configured with TURN **up front**: the signaling server stamps `CredentialsFor()` on the handshake, **capacity-gated at allocate time** (deny ⇒ STUN-only). It gathers **host + srflx + relay**.
- To stop the relay from winning when a direct path would work, the **device** biases toward direct — it is the ICE-controlling agent (see below) and **withholds nominating a relay candidate pair** until direct has had a generous window. The connector trickles **all** candidates immediately, relay included. `--relay`/`!relay` forces relay-only gathering on the connector. *(See* Favoring direct on slow & embedded devices *for the per-stack mechanism.)*
- The **device/listener stays STUN-only.** It gathers host + srflx, rides the connector's relay (single relay — see below), and **never needs `ice_restart`**. This is what makes embedded (C/C++) and the aiortc Python adapter "just work": there is no fallback re-offer to implement.

STUN (host/srflx) is free and not capacity-gated (`turn.Coturn.StunServers()`); only the connector's relay allocation counts against the gate.

## Which endpoint drives the relay decision  **[implemented — note]**

**The connector owns TURN; the device is passive.** The device gathers from whatever STUN servers the server stamped on it, offers, and accepts the connector's trickled candidates (including the late relay one). It never allocates a relay and never escalates.

Because the device is the **offerer**, it is also the **ICE-controlling** agent — so the *nomination* decision (which candidate pair wins) is made on the device, while the *relay candidate* is supplied by the connector. That split is the key to the direct-bias work below: the lever that prefers direct belongs on whichever side controls nomination (the device), even though the relay candidate is the connector's.

`--relay`/`!relay` is the only explicit override: it forces relay-only gathering on the connector, and the device skips its relay grace for that connection (`force_relay` rides the request — see the caveat below).

## Allocate vs. use a relay — why a device needs no TURN client  **[implemented — note]**

The single most useful distinction for embedded work:

- **Allocating (hosting) a relay** requires a full **TURN client**: `Allocate`, `Refresh` keepalives, `CreatePermission`, channel/Send framing for all traffic, and `turns:` (TLS) for the hard cases. Heavy and stateful.
- **Using the *other* peer's relay** requires **no TURN client at all** — only plain STUN + ICE. You treat the peer's relayed candidate as an ordinary public `IP:port` and send to it; all the TURN framing happens on the *allocating* peer's side.

Consequence: a device with **no TURN client** is still fully relay-reachable, via the connector's relay. The one prerequisite is that the device must have an **srflx candidate** (so the allocating peer can install an IP-scoped permission for it) — which is exactly why STUN is stamped on `pair_request`/`request` even for embedded listeners. With host-only candidates the relay can't be used.

## Single relay, not double  **[implemented — note]**

The device is STUN-only, so it never allocates its own relay: a relayed session is **device-srflx ↔ connector-relay**, a single TURN hop. (Under the old two-tier scheme the fallback pushed TURN creds to *both* peers, yielding an unintended relay↔relay double hop; single-phase removes that.)

Single-relay covers everything **except** a device whose own egress blocks outbound UDP (a TCP-only network): it can't reach the connector's relay over UDP and would need to tunnel out via `turns:`/TCP, which means allocating its own relay → a TURN client. That UDP-blocked-egress case is the *only* thing a device-side relay buys. **Note:** symmetric NAT on the device is **not** such a case — the device initiates to the single fixed TURN relayed address, so its per-destination mapping is consistent and TURN learns its source from the incoming check (peer-reflexive through the relay).

**Direction:** prefer single-relay for embedded devices; add a device TURN client only to escape UDP-blocking egress.

## Favoring direct on slow & embedded devices: device-side relay grace  **[implemented — Go + Python; ESP32 pending]**

The connector's relay-trickle delay (above) is **not enough on a slow device.** The controlling agent nominates the relay pair as soon as it validates, and a relay pair validates fast and reliably (the relay is a stable public address; direct needs hole-punching). On a fast peer direct wins inside the delay window; on a slow one (a Pi proxy, an ESP32) the relay candidate arrives, validates, and is nominated *before* the direct punch completes — so it relays when direct would have worked. (Observed: an OctoPrint Pi always relays while a desktop Jellyfin proxy always goes direct, same code, same network.)

**The fix: move the direct-bias from the connector's trickle delay to the controlling device**, where it can be expressed as "don't nominate a relay pair until direct has had a generous window." Each device stack exposes a different lever, but the behavior is identical: give direct a generous grace — we don't mind waiting, since TURN bandwidth is the scarce resource and a slow fallback is acceptable — and use relay only after it.

| Device stack | Lever | Notes |
|---|---|---|
| **Go / pion** (`bitbangproxy`) | `SettingEngine.SetRelayAcceptanceMinWait(N)` on the device PC | pion's controlling selector nominates the highest-priority *valid* pair; a relay pair isn't nominatable until this wait (default 2 s) elapses from check-start (`pion/ice` `selection.go` `isNominatable`). The relay-typed *remote* endpoint trips it, so it works even though the relay candidate is the connector's. **Done:** `internal/peer/connection.go` builds an API with this SettingEngine at `N = 8 s` (plus a selected-pair log on connect). |
| **Python / aioice** (`bitbang-python`) | device-side **buffer-and-inject** gate in `adapter.py` `_add_ice_candidate` | aioice uses **aggressive nomination** (`aioice/ice.py` `check_start`: controlling ⇒ every check carries `USE-CANDIDATE`) — first valid pair wins, **no relay-grace knob**. So the bias must control *what aioice sees*: buffer inbound `typ relay` candidates, hand aioice only direct ones, and `addIceCandidate` the relay only after a grace timer if not yet connected (drop it if `iceconnectionstatechange` reaches connected/completed first). Short-circuit: if the only candidates seen are relay, inject immediately. No aioice fork — inbound trickle works in 0.10.2, and the adapter never signals end-of-candidates, so `connect()` stays open for late injection. |
| **ESP32 / libjuice** (`psi_esp32`) | `NOMINATION_TIMEOUT` (native) + drop-if-connected | libjuice's controlling agent already prefers the higher-priority direct pair and only nominates an already-succeeded relay pair after `NOMINATION_TIMEOUT` (`libjuice/src/agent.h`, default 2 s). Tune it generous. Plus a ~2-line optimization at the `addRemoteCandidate` site (`main/httpd_server.cpp`): drop an inbound relay candidate if already connected, so a direct connect never even forms a relay pair. Device is STUN-only (TURN commented out). |

**Connector delay removed (both connectors).** With the grace on the device, the connector's `relayTrickleDelay` was redundant for the *choice* and has been removed — the `onicecandidate` relay branch in `bootstrap.js`, and `trickleDelay`/`relayTrickleDelay`/the `time.AfterFunc` in `bitbangproxy/internal/client/peer.go`. Connectors now trickle every candidate immediately; the browser/CLI connector needs no grace of its own (it is the controlled answerer). `--relay` on the Go connector now forces **relay-only gathering** (`ICETransportPolicy:relay`, matching bootstrap.js) — it previously only skipped the now-removed delay.

**Forced relay skips the grace.** A `--relay`/`!relay` connect carries `force_relay` on the request, and the signaling server forwards that field to the device; the device then sets `relayAcceptanceMinWait = 0` for that PC, so the relay pair is nominated as soon as it validates — no dead wait. (A genuinely can't-punch path that *didn't* force relay still waits the full grace before falling back, which is the intended trade-off.) This mirrors the aioice gate's forced-relay short-circuit (inject relay immediately when no direct candidate is seen).

*(The device-first deploy sequencing that originally gated this is moot pre-release — device and connector ship together.)*

**Keep the direct-pair check budget ≥ the relay grace (port-restricted connectors).** The grace only helps if the direct pair is still *alive* when it can finally validate. A connector behind a port-restricted NAT (common on office/cellular) only opens its pinhole once *it* first sends the device a check (~1.5-2 s in), and its peer-reflexive source equals its advertised srflx — so pion attributes the punch to the (still-unvalidated) srflx pair rather than minting a fresh prflx. pion's default `maxBindingRequests = 7` (~1.5 s) exhausts that pair right as the pinhole opens → marked failed → relay. Fix: `SettingEngine.SetICEMaxBindingRequests(N)` on the device, sized to the grace (`internal/peer/connection.go`, `iceMaxBindingRequests = 40` ≈ 8 s at pion's ~200 ms/check). **Coupled to `relayAcceptanceMinWait`** — change the grace, change the budget to match, or a slow-punch connector relays even though direct was reachable. (Confirmed: office→home connects direct in ~1.9 s with srflx advertised, `tx=10` on the winning pair. NB: the office NAT here is a port-restricted *cone*, not symmetric.)

**Open verification (carried over from single-phase):** that connectors prune the unused TURN allocation promptly on a direct connect (`getStats()`); if one holds it for the session, the up-front allocation's capacity cost is session-long rather than just the setup window.

## The superseded two-tier scheme & ICE restart  **[superseded]**

*Historical — not current code.* BitBang originally connected in **two tiers**. Tier 1 stamped STUN-only and tried direct (~75%); only on failure did the connector ask for TURN (fallback timer → `request_ice`), the server issue capacity-gated creds, and an **`ice_restart`** re-gather with relay candidates bring up the remaining ~25%. The restart only ever happened *before* a connection existed (no live DTLS to restart underneath) — it re-offered a fresh ICE ufrag with the *same* DTLS fingerprint, so no re-verify ran.

It was replaced by single-phase for two reasons: (1) `ice_restart` was hard on aiortc/embedded — the aiortc listener couldn't do it (ICE servers are fixed at PeerConnection construction), so a Python-backed device was effectively tier-1-only; and (2) when TURN was available alongside direct, the relay often won the candidate race anyway (the problem the device-side grace now addresses). Vocabulary note: "tier 1 / tier 2" may still appear in older comments and on the server (`iceForClient` in `bitbang-server/internal/handler/device_ws.go`, now always stamping TURN up front).

## Endpoint capability negotiation  **[proposed]**

A low-resource device can declare a **transport capability bitmap** to the signaling server at register time, so the server gates what it sends. Under single-phase the old **ICE-restart dimension is moot** (there is no `ice_restart`); the one axis still worth expressing is whether the device has a **TURN client** at all — i.e. whether it can escape a UDP-blocking egress.

| Device caps | Server behavior | Coverage |
|---|---|---|
| STUN only (rides connector relay) | stamp STUN on the device; connector allocates | ~100% except UDP-blocked device egress |
| STUN + own TURN client (`turns:`/TCP) | device can also tunnel out of UDP-blocking egress | ~100% |

- Carried on the `register` message, stored on `registry.DeviceConn` alongside the BYO-`ICEServers` override.
- The cap describes **software capability** ("do I implement a TURN client?"), *not* network conditions (which the device can't know at build time).
- **Backward compatible:** absent caps ⇒ legacy behavior. Use a compact **integer bitmap** (a *different axis* from the string stream-caps `shell`/`files`/… — don't conflate them).

## Verification is per-flow, orthogonal to the relay path  **[implemented]**

Which path (direct vs. relayed) a session uses has **nothing to do** with how the endpoints authenticate each other. Authentication is chosen by *flow*:

- **Direct / URL flow:** public-key **bidirectional verify**. The connector has the UID out-of-band (from the URL), checks `hash(pubkey) == UID`, encrypts `{fingerprint, nonce, access_code}` to the listener's key, and the listener proves possession with `sha256(nonce)`. MITM-safe by cryptography; no human step.
- **Pairing flow:** the **SAS** (code B). The pubkey verify is deliberately **skipped** (`HandlePairAnswer`) because the connector has *no* out-of-band identity anchor — the server-resolved 6-digit code commits to no key, so a pubkey verify would be vacuous against a malicious server. The SAS is the replacement out-of-band anchor.

## Pairing is a bootstrap, not a session transport  **[implemented]**

The pair channel exists only to run the SAS and deliver `uid + access_code`. On approval it is **torn down**, and the connector makes a **fresh, standard direct connection** (full public-key verify + the normal single-phase connect) for the actual session. Consequences:

- The pair channel itself is **STUN-only** with no relay bias, so first-contact pairing needs a direct/srflx-reachable path — or `--relay` (which carries `force_relay` through `pair_init`, so the server stamps TURN onto the device's offer up front).
- The real session is a standard single-phase connect for that connector type, so it reaches the same coverage even when the pairing handshake was STUN-only.
- Application traffic always rides a channel authenticated by the strong public-key verify, never by the 6-digit SAS alone.

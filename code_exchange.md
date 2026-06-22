# Code Exchange — Design

BitBang's **code exchange** lets two people who have never connected before
establish a verified session using a short, human-shareable code — no URL, no
account, no pre-shared key. It is the pairing path; once a device is paired, all
subsequent connections use the ordinary URL/direct flow.

There are two codes:

- **Code A** — a short numeric code the signaling server issues to the listener
  at startup (e.g. `482731`). The listener's operator shares it with the
  connector verbally or by text. The server keeps a `code → UID` table so the
  connector can be routed to the device without typing a 22-character UID.
- **Code B** — a 6-digit **Short Authentication String (SAS)** that both peers
  compute independently from the negotiated channel. It is **never transmitted**;
  the **connector displays it** and reads it aloud, and the **listener's operator
  types what they hear**. BitBang on the listener compares the typed value to its
  own computed value. This out-of-band human comparison is what makes the channel
  trustworthy without trusting the signaling server.

Code A makes the pairing *addressable*; Code B makes it *verifiable*.

---

## Security model

BitBang treats the signaling server as **untrusted** — it brokers the
introduction but must never be able to read traffic or impersonate either end
(see `whitepaper.md`). The URL/direct flow upholds this with a public-key
bidirectional verify anchored on the UID carried in the URL (`hash(pubkey) ==
UID`). Pairing can't use that anchor: at first contact the connector has only
Code A, a server-resolved lookup handle that commits to no key. So pairing's
trust anchor is the **human out-of-band channel** — Code B — and the design's
job is to make that anchor sound and then bootstrap strong crypto from it.

Three properties make Code B trustworthy against a malicious server:

1. **No grinding (commitment).** A short SAS truncates a hash of inputs the
   man-in-the-middle partly controls (the DTLS certificate fingerprints — a MITM
   presents its own certs on each leg). Without protection it could **grind
   offline**: generate ~10⁶ certs in seconds until its two legs' SAS values
   collide, then run the MITM once and pass the human check with near-certainty,
   silently, with **zero** failed pairings. A **commitment exchange** (below)
   converts this from an *offline* grind into an *online* one: every value the
   attacker controls is locked before it learns what it must match, so it can't
   search locally — each fresh attempt requires a whole new handshake with the
   victim (which supplies a fresh nonce it must again commit before). That turns
   a free, instant, invisible attack into ~10⁶ full WebRTC handshakes against the
   listener — slow, load-visible, serialized by one-pairing-at-a-time, and
   bounded by the connector-human's patience window — i.e. a blind `1/10⁶` per
   attempt that can no longer be ground down for free.

2. **Enough bits (6 digits).** With grinding removed, a blind MITM succeeds with
   probability `1/N`, `N = 10⁶`. The invariant that matters: a wrong SAS is a
   *detected* failure, and **failures-per-stolen-device ≈ N**, independent of how
   selectively the server attacks — lowering the attack rate lowers harvest and
   noise by the same factor, never the ratio. So at 6 digits, stealing one device
   costs ~10⁶ visible "code didn't match" failures, which can't hide under any
   realistic benign-failure baseline. (4 digits costs only ~10⁴, hideable; that's
   why we use 6.) Detection is otherwise weak — individuals rationalize a
   mismatch as a typo, and the natural aggregator of failure rates is the server
   itself — so `N`, not monitoring, is the defense.

3. **Secrets ride the verified channel, not signaling.** Once Code B matches, the
   data channel is proven MITM-free. The device delivers its identity and
   credentials (`uid`, `public_key`, `access_code`) **over that data channel**,
   which is DTLS-encrypted end to end — the server is not an endpoint and can
   neither read nor alter it. Nothing secret crosses the signaling channel.

**Residual risk:** first contact is trust-on-first-use at **1/10⁶** — a blind
MITM has a one-in-a-million chance per pairing, can't grind it lower, and pays
~10⁶ visible failures per success. It is a **one-time, per-device** risk: a clean
pairing delivers the device's real pubkey/UID, and every connection afterward
uses the full ~128-bit direct-flow verify with no Code B. Same posture as SSH /
Bluetooth first-contact, tightened to one-in-a-million and made non-grindable.

---

## End-to-end flow

### First-contact pairing (one-time)

```
Listener (device)            Signaling server            Connector
  │                                │                         │
  ├── register{want_code} ────────►│                         │
  │◄── registered{code_A} ─────────┤   "Pairing code: 482731 (valid 5 min)"
  │                                │                         │
  │                                │◄── pair_init{code_A} ───┤   /ws/pair
  │                                │  (≈3s constant-time     │
  │                                │   lookup, per-IP cap)   │
  │◄── pair_request{client_id, ────┤── pair_routed ─────────►│
  │      ice_servers(STUN)}        │                         │
  │                                │                         │
  │ WebRTC offer / answer / candidate (listener offers,      │
  │ connector answers) — DTLS up, pairing data channel open  │
  │  ◄══════════════ data channel ═══════════════════════►  │
  │                                                          │
  │  ◄── pair_commit{ H(r_c) } ─────────────────────────────┤   commit
  │  ── pair_challenge{ r_d } ──────────────────────────────►   challenge
  │  ◄── pair_reveal{ r_c } ────────────────────────────────┤   reveal (verify H(r_c))
  │                                                          │
  │  both: SAS = trunc6( H(r_c ‖ r_d ‖ sort(fp_l, fp_r)) )   │
  │                                                          │
  │  ┌──────────────────┐              ┌───────────────────┐ │
  │  │ Enter code: ____ │   listener   │ Your code: 318420 │ │ connector displays
  │  └──────────────────┘   types what │ Read this to them │ │
  │   (SAS not shown)       they hear  └───────────────────┘ │
  │  BitBang compares typed value to its own SAS → match     │
  │                                                          │
  │  ── pair_credentials{ uid, public_key, access_code } ───►│   over data channel
  │  ── pair_approved (bare ack) ─►│── pair_approved ───────►│   over signaling
  │                                │                         │
  │  Pairing PC torn down. Connector saves uid/pubkey/code,  │
  │  then reconnects via the standard direct flow (below)    │
  │  for the actual session. (Browser: navigates to the URL.)│
```

### Subsequent connects (direct)

After pairing the connector holds `uid + public_key + access_code`. Every later
connection skips code exchange entirely and uses the ordinary flow:

```
Connector            Signaling server            Listener
  ├── request{uid} ──────►├── request ──────────────►│
  │ Standard WebRTC + public-key bidirectional verify:
  │ connector checks hash(pubkey)==uid, encrypts
  │ {fingerprint, nonce, access_code} to the device key,
  │ device proves possession by returning hash(nonce).
  │ Application traffic flows.
```

The `access_code` in the encrypted payload authorizes the session; no Code B,
no human step — full-strength crypto, anchored on the pubkey pairing delivered.

---

## Code A — the pairing code

A 6-digit numeric code issued by the signaling server when a listener opts in
(`register{want_code:true}`), held in an in-memory table.

- **Length:** 6 digits (1-in-10⁶ collision odds; table holds tens of thousands
  of live codes comfortably).
- **TTL:** 5 minutes — long enough for human coordination, short enough to keep
  the table small. A background sweeper evicts expired entries.
- **Idempotent per UID:** if a listener already has a live code, re-registering
  returns the same code.
- **Lookup brake:** `Lookup` sleeps a fixed ~3s **regardless of outcome**
  (constant-time, so timing doesn't leak whether a code exists), and the server
  caps in-flight `/ws/pair` connections per IP. Together these bound brute-force
  enumeration of the 10⁶ space to a negligible rate.
- **Released** on listener disconnect, so the code is reusable immediately.
- **Lost on server restart** (in-memory) — listeners reconnect and get fresh
  codes. Acceptable; the same applies to the device registry today.

---

## Code B — the SAS (with commitment)

Code B is a 6-digit value both peers compute independently after the data
channel opens. It is **never sent over the channel**; only the *result* is
compared, out-of-band, by humans.

### Derivation

```
digest = SHA-256( r_c ‖ r_d ‖ sort(fp_local, fp_remote) )   // 32 bytes
SAS    = sprintf("%06d", be_uint32(digest[0:4]) % 1_000_000)  // first 4 bytes, big-endian
```

`be_uint32(digest[0:4])` reads the first four bytes of the digest as a
**big-endian** `uint32` (Go `binary.BigEndian.Uint32`, JS `DataView.getUint32(0)`,
Python `int.from_bytes(d[:4], "big")`). All implementations must agree on
big-endian or the SAS won't match across sides.

- `fp_local`, `fp_remote` — the two DTLS fingerprints, normalized (uppercase) and
  **sorted** so both ends hash the identical pair regardless of which side calls
  which "local." These bind the SAS to the actual channel: a naive relay presents
  different fingerprints on each leg, so the two SAS values diverge.
- `r_c`, `r_d` — fresh 32-byte random nonces from the connector and device. These
  carry the **anti-grind** property via the commitment below; the fingerprints
  alone are grindable.

### The commitment exchange

Run over the open pairing data channel, **before** computing the SAS. The
connector commits first (it's the side that displays the SAS; a commitment needs
no prior trust, so it's safe on the not-yet-verified channel):

```
connector → device:  pair_commit    { commit:  base64(SHA-256(r_c)) }
device   → connector: pair_challenge { nonce_d: base64(r_d) }
connector → device:  pair_reveal    { nonce_c: base64(r_c) }   device verifies SHA-256(r_c) == commit
```

Why it defeats grinding: the committer locks `r_c` inside `H(r_c)` (which hides
it) **before** seeing the challenger's `r_d`, and `r_d` is folded into the SAS.
So neither party — nor a MITM relaying both legs — can adaptively choose its
contribution to force a collision: every attacker-controlled value (its
substituted certs, fixed at SDP time; its nonce, fixed at commit time) is locked
before it can learn the value it would need to match. It cannot grind **offline**
— each attempt needs a fresh handshake with the victim, which supplies a fresh
nonce the attacker must again commit to blind. So the attack drops from a free
local search to a blind `1/10⁶` per *online* attempt, each costing a full
WebRTC handshake against the victim (rate-limited, load-visible, and bounded by
the connector-human's patience). The fingerprints alone don't give this — a
plaintext nonce swap just relocates the grind onto the last-sent nonce; the
commit→challenge→reveal ordering is what removes it. (This is ZRTP's commitment
trick; the nonces, not the fingerprints, carry the anti-grind property.)

### Human verification (blind listener entry)

- The **connector displays** the SAS and reads it aloud.
- The **listener never sees it.** Its operator types what they hear; BitBang
  compares the typed value to its own computed SAS. There is nothing to "blindly
  approve" — the prompt is unanswerable without coordinating with the connector,
  so it **fails closed** against an inattentive operator (the failure mode is "no
  connection," not "wrong connection").
- **3 typing attempts** for fat-fingers, then reject (`sas_mismatch`). Retries
  don't multiply the attacker's odds — the certs and nonces are fixed for the
  handshake, so the SAS is fixed; retries only forgive typos.

---

## Credential delivery (over the verified data channel)

On a matching SAS, the device sends its identity and credentials **over the data
channel** (now SAS-verified, DTLS-encrypted end to end):

```
device → connector (data channel):
  pair_credentials { uid, public_key: base64(DER), access_code }
```

- **Confidential** — the server is not a DTLS endpoint; it cannot read this.
- **Integrity** — the server cannot rewrite `uid`/`access_code`; a swapped `uid`
  would otherwise redirect every future direct connect to a rogue.
- **Harvest-limited** — sent only *after* approval, so a malicious server obtains
  the code only on a successful (1/10⁶) MITM, never on an honest pairing.

`public_key` rides here too: once Code B has proven the channel reaches the real
device, the connector trusts the key and saves `UID = hash(public_key)` as the
anchor for all future connects. **`access_code` is the listener's own static
fragment code** (the same secret a URL recipient would get), so a paired
connector ends up with exactly the credentials of a URL connect.

Signaling carries only **non-secret control**: `pair_approved` is a bare
"approved — read credentials off the data channel" ack, and `pair_rejected
{reason}` carries only the reason (`sas_mismatch` | `user_declined` | `timeout`).

---

## Post-pairing: tear down and reconnect

The pairing channel is **not** reused for the session. On approval the connector
closes the pairing PC and establishes a **fresh connection via the standard
direct flow** using the credentials it just received. Reasons:

- The pairing channel was authenticated by the **SAS only** (it skips the
  public-key bidirectional verify). The real session must ride the
  **pubkey-verified** channel — *application traffic is never authenticated by
  the SAS alone.*
- The direct flow is TURN-capable and fully tested; pairing stays a thin
  bootstrap.

**CLI:** `runPairConnect` returns the obtained `uid+access_code`; the connector
dials the direct flow in-process and saves the device to its known-hosts table.

**Browser:** on `pair_approved`, call `location.replace('/<uid>#<access_code>')`
— *navigate* (so the page reloads into the verified direct flow), and `replace`
(not `assign`/`replaceState`) so the spent pairing URL stays out of the back
stack. The page reload tears down the pairing channel for free, and the
credential lands in the URL bar, where the user can bookmark/share it.

---

## Wire protocol

JSON messages. **Signaling-channel** messages go over the WebSocket to the
server; **data-channel** messages (the commitment and credential exchange) go
directly peer-to-peer over the open pairing data channel and the server never
sees them.

### Signaling channel

```jsonc
// listener → server (register)
{ "type": "register", "protocol": 3, "public_key": "<base64 DER>",
  "ice_servers": [ /* listener's BYO TURN, optional */ ], "want_code": true }

// server → listener (registered)
{ "type": "registered" }                    // no pairing / want_code false
{ "type": "registered", "code": "482731" }  // pairing code issued

// connector → server (start pairing, on /ws/pair)
{ "type": "pair_init", "code": "482731", "force_relay": false }

// server → connector
{ "type": "pair_routed" }                      // code resolved, routing to listener
{ "type": "error", "message": "unknown_code" } // after the constant-time delay

// server → listener (incoming pairing); listener treats it like a `request`
{ "type": "pair_request", "client_id": "<id>",
  "remote_ip": "<connector IP, for the operator's audit>",
  "ice_servers": [ /* phase-1 STUN, same as a direct request */ ] }

// then: offer / answer / candidate exchange, keyed by client_id (listener offers)

// listener → server → connector (outcome; NON-SECRET)
{ "type": "pair_approved", "client_id": "<id>" }                    // bare ack
{ "type": "pair_rejected", "client_id": "<id>",
  "reason": "sas_mismatch" | "user_declined" | "timeout" }
```

### Pairing data channel (peer-to-peer; server-blind)

```jsonc
// commitment exchange (before computing the SAS)
{ "type": "pair_commit",    "commit":  "<base64 SHA-256(r_c)>" }   // connector → device
{ "type": "pair_challenge", "nonce_d": "<base64 r_d>" }            // device → connector
{ "type": "pair_reveal",    "nonce_c": "<base64 r_c>" }            // connector → device

// credentials (on matching SAS)
{ "type": "pair_credentials", "uid": "<UID>",
  "public_key": "<base64 DER>", "access_code": "<base64url 11 chars>" }   // device → connector
```

---

## Server-side (`bitbang-server`)

### Pairing table — `internal/pairing/table.go`

```go
type Entry struct {
    UID       string
    CreatedAt time.Time
}

type Table struct {
    mu    sync.RWMutex
    codes map[string]Entry  // code → entry
    byUID map[string]string // uid → code (cleanup on disconnect)
}

const (
    CodeLength  = 6
    CodeTTL     = 5 * time.Minute
    LookupDelay = 3 * time.Second
)

// Issue returns a unique live code for uid, idempotent within TTL.
func (t *Table) Issue(uid string) string { /* random 6 digits, retry on collision */ }

// Lookup returns the UID for a code, or "" if unknown/expired. Always sleeps
// LookupDelay first (constant-time, brakes enumeration), then checks TTL.
func (t *Table) Lookup(code string) string { /* sleep; lookup; TTL check */ }

// Release frees a UID's code (called on listener disconnect).
func (t *Table) Release(uid string)

// NewTable starts a background sweeper that evicts entries older than CodeTTL.
```

### Wire types — `internal/wire/messages.go`

- `Register`: add `WantCode bool`. `Registered`: add `Code string,omitempty`.
- `PairInit{ Code, ForceRelay }`, `PairRouted`, `PairRequest{ ClientID, RemoteIP,
  ICEServers }`, `PairApproved{ ClientID }` (**no `access_code`** — credentials
  go peer-to-peer), `PairRejected{ ClientID, Reason }`.

### Handlers — `internal/handler/`

- **`device_ws.go`** (listener): on register, if `want_code`, `Pairing.Issue(uid)`
  and return it in `registered`. In `deviceRelay`, forward the listener's
  `pair_approved` (bare) / `pair_rejected` to the connector by `client_id`. On
  disconnect, `Pairing.Release(uid)`.
- **`pair_ws.go`** (connector entry, `/ws/pair`): read `pair_init`, `Pairing.
  Lookup(code)`; on miss reply `error{unknown_code}` (after the constant-time
  delay) and close. On hit, register the connector, set `ForceRelay` from the
  message, and send the listener a `pair_request` **stamped with phase-1 STUN**
  (`iceForClient(device, clientID, withhold=true)`, mirroring `clientRelay`'s
  `request` case). Then `pair_routed` to the connector and hand off to the
  standard client relay. A per-IP in-flight semaphore caps concurrent attempts.
- **`Deps`**: add `Pairing *pairing.Table`, constructed in `cmd/signaling/main.go`.
- **Status endpoint**: expose `active_code_count`.

The server never sees the commitment, the SAS, or the credentials — those are
data-channel-only.

---

## Listener-side (`bitbangproxy`)

- **Signaling client** (`internal/signaling/client.go`): `WantCode` (set true by
  default for `bitbang serve`) sends `want_code`; `PairingCode` captures the code
  from `registered` and is reset on each reconnect (a server that loses its table
  re-issues, so a stale code must not linger).
- **SAS computation** (`internal/peer/sas.go`):
  `ComputeSAS(rc, rd []byte, localFp, remoteFp string) string` →
  `SHA-256(rc ‖ rd ‖ sort(upper(localFp), upper(remoteFp)))`, first 32 bits mod
  10⁶, `%06d`. The browser and Python mirrors must match byte-for-byte.
- **Pairing handler** (`internal/pairing/listener.go`): `PromptForSAS(expected,
  prompt)` runs the bounded (3-attempt) blind-entry loop and returns a wire-stable
  reason; `DefaultTTYPrompt` reads stdin. The SAS is **never logged**, even in
  verbose mode.
- **Connection wiring** (`internal/peer/connection.go`): `HandlePairRequest`
  builds a pair-mode `Connection` (`PairingMode`) whose data-channel `OnOpen`
  body (`handlePairRequestOnOpen`):
  1. runs the commitment exchange (receive `pair_commit`, send `pair_challenge`
     with a fresh `r_d`, receive `pair_reveal`, verify `H(r_c)`),
  2. computes the SAS from `r_c`, `r_d`, and both fingerprints,
  3. runs `PromptForSAS`,
  4. on match sends `pair_credentials{uid, public_key, access_code}` over the
     data channel and a bare `pair_approved` over signaling; on failure sends
     `pair_rejected{reason}`,
  5. closes the pairing PC (the connector reconnects via the direct flow).
- **CLI** (`cmd/bitbang/serve.go`): `--nocode` disables code issuance (default
  code ON for the `bitbang serve` family). Headless/non-TTY deployments should
  pass `--nocode`, since the SAS prompt can't be answered without an operator.

---

## Connector-side

### CLI — `cmd/bitbang/connect.go` + `connect_pair.go`

`bitbang connect <target>` dispatches on the shape of `<target>`:

```
saved name (matches ^[A-Za-z][A-Za-z0-9_-]*$)  → known-hosts lookup → direct connect
6-digit code (^\d{6}$)                          → pair flow → direct connect
otherwise                                       → URL form
```

The three shapes are mutually exclusive by construction (a name starts with a
letter and excludes `. : / # @`, so it can't be a code or URL). A name-shaped
token not in the table is a hard error (`no saved device named "x"`), not a
silent URL dial.

**Pair flow** (`runPairConnect`): open `/ws/pair`, send `pair_init` (with
`force_relay` if `--relay`), await `pair_routed`, answer the listener's offer
(pair-mode peer — no UID/code, no `encrypted_request`; the connector is the
answerer). When the data channel opens: drive the commitment exchange (send
`pair_commit`, await `pair_challenge`, send `pair_reveal`), compute and **display
the 6-digit SAS**, then read `pair_credentials` off the data channel
(`p.DCMessages()`). Tear down, save the device, and continue into a normal direct
session.

### Browser — `web/bootstrap.js`

`/<6-digit>` routes to the pairing flow; `/<UID>#<code>` to the direct flow. The
pairing flow mirrors the CLI (commitment, 6-digit `computeSAS` via SubtleCrypto,
a minimal "Your code: …, read it to the device owner" modal, read
`pair_credentials` off the data channel), then `location.replace('/<uid>#<code>')`
to reconnect via the direct flow. The browser keeps **no app-managed device
store** — see *Remembering*, below.

---

## ICE / TURN

The signaling server uses **lazy, two-phase ICE**: free, ungated STUN
(`StunServers()`) first — gather host + srflx, try direct P2P (~75% of network
pairs) — and capacity-gated TURN (`CredentialsFor()`) only on fallback.

- **Pairing handshake:** the server stamps **phase-1 STUN** onto `pair_request`,
  so the listener gathers srflx candidates and is reachable through NAT (without
  it a behind-NAT listener only has host candidates → LAN-only). The connector
  uses the STUN the server stamps on the relayed offer. Both gather srflx → same
  ~75% direct coverage as a direct connect.
- **Force relay:** `bitbang connect <code> --relay` sends `force_relay` in
  `pair_init`; the server stamps TURN onto the device's offer up front, for a
  pairing on a known-hard network.
- **Direct flow** (the real session after pairing): the connector implements the
  full lazy fallback (`request_ice` on a stall → ICE-restart re-answer), so the
  session reaches ~100% regardless of how the pairing handshake fared.

Using a relay needs no TURN client — only the *allocating* peer (the connector)
runs one; the listener just gathers srflx and connects to the connector's relay
candidate. So an embedded listener can stay STUN + ICE only. (A future device
capability bitmap can let the server tailor what it sends per device.)

---

## Remembering paired devices

### CLI — named known-hosts table

Every successful connect (pair or URL) is recorded in `~/.bitbang/devices.json`
(atomic write, `0600`) so the operator can reconnect by a short **name**:

```json
{ "devices": [
  { "name": "nas1", "uid": "<UID>", "access_code": "<…>",
    "server": "bitba.ng", "paired_at": "2026-06-19T12:00:00Z" } ] }
```

- **UID is the primary key** (stable across access-code rotation / re-pairing);
  reconnecting a known UID refreshes its row in place and keeps the name.
- **Name is a unique, case-insensitive secondary index.** `-name nas1` sets it on
  a new device; without `-name`, an auto name `device<N>` (lowest free index) is
  assigned and printed (`Saved as "device1". (tip: pass -name …)`). `-name` is for
  new devices only — renaming via `connect` is rejected (a future `bitbang
  devices rename` is the place for it).
- **Save timing:** a pairing is recorded the moment its SAS is verified (so a
  flaky reconnect can't lose the credential or burn the one-time code); a URL
  connect is recorded once connected.

Files: `cmd/bitbang/devices.go` (table, validation, auto-name, `recordDevice`),
`cmd/bitbang/connect.go` (`-name` flag, dispatch).

### Browser — bookmarks, by design

The browser keeps **no** device store. After pairing it navigates to
`/<uid>#<access_code>`; the user bookmarks/shares that URL as their "remember."
We deliberately do **not** use `localStorage`/`IndexedDB`: that storage lives on
the signaling server's own origin (`bitba.ng` serves bootstrap.js), so any
server-served script could read the saved access codes in bulk on any later
visit — defeating the property that keeps the code off the server. The URL
fragment + the user's bookmarks keep the secret client-side and per-use. This is
the intended CLI/browser asymmetry; cross-device sync is out of scope for both
(it would need an account or synced backend, against the no-account principle).

---

## User experience

Pairing is a two-person, two-code flow, so the human-facing copy carries real
weight. Cross-cutting principles for every surface:

- **Mirror the verbs.** The connector **reads** Code B aloud; the listener
  **types** what they hear. Both screens use the same words ("read" / "type") so
  nobody loses the thread.
- **Label by action, not by letter.** Code A and Code B are *both* 6 digits and
  flow in opposite directions — the #1 confusion. Never show users "Code A/B";
  show *"the code you were given" / "enter this"* vs *"read this aloud" / "type
  the code they read you."*
- **Listener is type-what-you-hear, never approve-what-you-see.** No listener
  surface (TTY or GUI) may display the SAS and offer a Yes/No — that reintroduces
  blind approval. The only action is typing the heard code; the prompt is
  unanswerable without coordinating, which is the security property.
- **A rejection leaked nothing.** Credentials are sent only on a match, so frame
  a mismatch as "we protected you," not as an error.

### Connector — shell (`bitbang connect <code>`)

The terminal connector. Mirrors the browser flow in text.

```
$ bitbang connect 482731
Pairing with bitba.ng…

  Your pairing code:  318 420

  Read this code aloud to the device owner.
  They'll type it on their device to confirm.

Waiting for approval…
✓ Paired.  Saved as "nas1".
Connecting to nas1…
Connected.
$
```

- `bitbang connect <6-digit>` enters the pair flow; the SAS is **displayed**, never typed here.
- On approval: save to the known-hosts table (auto-name printed, or `-name`), then continue straight into the session.
- Errors map to one friendly line each:
  - `unknown_code` → "That code isn't valid or has expired (codes last 5 minutes). Ask for a fresh one."
  - `sas_mismatch` → "The codes didn't match, so the connection was refused for your safety. Try again."
  - `user_declined` → "The device owner declined."
  - `timeout` → "The device owner didn't confirm in time."

### Connector — browser

Entry, then a small state machine. Two ways in:

- **`bitba.ng/<codeA>`** (router matches `^\d{6}$`) — the listener can share the code as a tappable link / QR, so the connector types nothing.
- **`bitba.ng/`** root — a large numeric input (`inputmode="numeric"`, auto-submit on the 6th digit). This is the **primary** path: pairing exists for codes shared *verbally*, so manual entry is the common case.

States:

1. **Enter** — numeric pairing-code input (skipped when arriving via `/<codeA>`).
2. **Connecting…** — a real WebRTC handshake + commitment runs here; show a spinner.
3. **Read aloud** — Code B big and grouped (`318 420`): *"Read this aloud to the device owner. They'll type it to confirm it's really you."* No approve button (deliberate). Once read, a **"Waiting for the device owner to confirm…"** sub-state.
4. **Paired ✓** — `location.replace('/<uid>#<code>')` into the session, then the bookmark modal (below).

Error copy (never raw reason strings): expired/unknown code → "ask for a fresh one"; `sas_mismatch` → a calm *safety refusal* ("we refused the connection to keep you safe"); declined/timeout → "ask them to accept and try again"; device offline → "the device went offline."

**Bookmark modal** (post-pairing only — gated on a one-shot flag carried across the `location.replace`, so normal bookmarked revisits don't show it):

```
┌──────────────────────────────────────────────┐
│  ✓ Paired!                                    │
│  Save this link to reconnect instantly —      │
│  no code needed next time:                    │
│   ┌────────────────────────────────────────┐ │
│   │ https://bitba.ng/A1b2…#x9Y…  (select)  │ │
│   └────────────────────────────────────────┘ │
│        [ 📋 Copy link ]  →  Copied ✓           │
│   or press ⌘D / Ctrl+D to bookmark            │
│   🔒 This link connects straight to the       │
│      device — keep it private.                │
│                              [ Done ]         │
└──────────────────────────────────────────────┘
```

The primary action is **Copy link**, not a dead "OK": a browser can't bookmark
programmatically, and a copied URL is more portable than a per-profile bookmark
(notes, password manager, another device). The privacy line matters because the
link contains the access code.

### Listener — shell (`bitbang serve`)

The security-critical side: the operator is granting access to their own device.

**Startup** — Code A is framed as the thing to *share*:

```
  [QR]
  URL:          https://bitba.ng/A1b2c3…#x9Y…
  Pairing code: 482731   ← share this; expires in 5 min
  Waiting for connections…
```

**Incoming request** — context, a clear "only if you're expecting this," and
type-what-you-hear:

```
──────────────────────────────────────────────────
  🔔 Pairing request from 203.0.113.5

  The other person has a 6-digit code on their screen.
  Ask them to read it to you.
  (This is a different code than the 482731 you shared.)

  Only continue if you're expecting this connection.

  Type the code they read you: ______
  (or press Enter alone to decline)
──────────────────────────────────────────────────
```

- The connector's `remote_ip` (already on `pair_request`) gives the operator
  context to catch an unsolicited request and decline.
- The "different code than the 482731 you shared" line heads off the most likely
  mistake — the operator typing their *own* pairing code (a harmless false-reject,
  but confusing).
- **3 attempts**, then a safety-positive refusal:
  ```
  ✗ Codes didn't match — connection refused for your safety. No access was granted.
  ```
- On match:
  ```
  ✓ Paired. 203.0.113.5 can now reach this device.
  ```
- A bounded **timeout** (~60s) auto-declines (`pair_rejected{timeout}`) so a
  walked-away operator doesn't hang the connector. *(The current TTY prompt has
  no such deadline — worth adding.)*

**GUI listener** (desktop app / OctoPrint plugin) deltas: **notify** rather than
print (the operator may not be at a terminal and is the gate), present the same
type-what-you-hear input plus an explicit **Decline** — *not* an
approve-the-shown-code button — and an outcome toast.

---

## Python adapter (`bitbang-python`)

Mirror the Go listener: `want_code` constructor arg + code reception; a
`ComputeSAS` that matches the Go output byte-for-byte (6 digits, nonces folded
in); the commitment exchange and blind SAS-entry loop (or an `on_pair_request`
callback receiving the SAS and returning approve/reason); and
`pair_credentials` sent over the data channel on approval.

---

## Decisions & defaults

| Decision | Choice |
|----------|--------|
| Code A | 6 digits, 5-min TTL, ~3s constant-time lookup + per-IP in-flight cap, in-memory, idempotent per UID, released on disconnect |
| Code B length | **6 digits** (1/10⁶ blind-MITM; ~10⁶ failures per stolen device) |
| Code B derivation | `trunc6(SHA-256(r_c ‖ r_d ‖ sort(fp_local, fp_remote)))`; never transmitted |
| Anti-grind | **commit → challenge → reveal** over the data channel before the SAS |
| Code B display | connector displays; **listener types blind** (fails closed) |
| SAS attempts | 3 typing attempts (typos only; doesn't multiply MITM odds) |
| Credential delivery | `pair_credentials{uid, public_key, access_code}` over the **SAS-verified data channel**; signaling `pair_approved` is a bare ack |
| `access_code` | static per-listener (= the listener's URL fragment code) |
| Post-pairing session | pairing PC torn down; connector reconnects via the direct flow (CLI in-process, browser by navigation) |
| Connector endpoint | dedicated `/ws/pair`; connector is the **answerer** (listener offers) |
| ICE | phase-1 STUN stamped on `pair_request`; `--relay` forces TURN; direct flow has lazy fallback |
| Concurrent pairings | sequential per listener |
| Protocol version | **additive within v3.x** — `want_code` optional; old servers ignore it, no version bump |
| Default | `bitbang serve` → code ON; `--nocode` for headless / non-TTY |
| Remember | CLI named `~/.bitbang/devices.json`; browser URL bar + bookmarks |

---

## Out of scope

- Network mode (token-based fleet access) — separate document.
- Per-pair (revocable) access codes — `access_code` is static per-listener for
  now; per-pair is an upgrade if revocation becomes a requirement.
- Device capability bitmap for tailored ICE/TURN — separate design.
- Remote desktop, serial bridging, TCP forwarding — separate roadmap items.
- Cross-machine / federated sync of remembered devices.

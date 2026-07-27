# Trustless Signaling: Authentication Without a Central Authority

BitBang is a system for end-to-end verified, browser-native remote access: a device runs BitBang and prints a URL, and any browser that opens it connects to the device -- usually directly, peer-to-peer. This paper is a companion to the [BitBang whitepaper](whitepaper.md), which covers the product. This one covers the trust model: how browser and device authenticate each other without trusting the server that introduces them, and why the guarantee holds even when that server is hostile.

Every remote access system has to answer one question before anything else: is the other end who it claims to be? That's authentication. Most systems delegate it to a central authority -- an account service, an SSO, an IT department. Trust starts there.

BitBang has no account and no central authority. That's by design -- it's a major reason BitBang is easy to use. But is there a cost? Without anyone to vouch for the device, how can the browser know for sure who it's actually talking to?

The answer is unusual: the browser does the authentication itself. The signaling server relays messages but plays no role in verifying identity. Two deliberate design choices make this possible. First, authentication is decoupled from the server -- the browser independently checks the device's public key against the URL, encrypts a challenge to that key, and verifies the response. The server relays the bytes but can't read them. Second, the verifier runs in the browser, where the code is inspectable. By contrast, server-side authentication is a black box -- users find it difficult to trust what they can't see. Together, BitBang's design choices shift operator trust from an opaque policy question to an inspectable code question -- and for users of the installed CLI or Python client, there is no served code at all -- trust reduces to the software you installed, as with any local tool.

Authentication attacks come in two forms:

| Attack | Visibility | Protection |
|--------|------------|------------|
| Wire (MITM, SDP tampering) | Covert | The protocol |
| JS tampering | Overt | Transparency (auditable code) |

The protocol protects against covert attacks. Transparency protects against overt ones.

## The Broker Problem

Every remote access solution needs a broker. Your device is behind NAT. The browser is behind NAT. They can't find each other directly. Something in the middle -- a server with a public IP -- must introduce them.

The question is: what does the broker do once the introduction is made?

**Cloud platforms:** The broker IS the data path. Every message flows through their servers.

**Tunneling services:** Your HTTP traffic is decrypted at their edge, then re-encrypted to you.

**Mesh VPNs:** Each connection is negotiated by their coordination servers.

The broker sits between the endpoints -- that's its job. Cryptography helps only if the key exchange is honest; a broker that quietly substitutes its own key for either endpoint's becomes an undetected man-in-the-middle. The encryption stays intact, it just terminates at the broker. Even trusted mesh VPNs have this property by default -- for example, Tailscale publicly acknowledged the gap and shipped Tailnet Lock as an opt-in mitigation. Whether the attack is currently happening is unknowable from the user's side, and that's the problem. Users shouldn't have to bear the burden of trust.

Many proprietary solutions offer security that's also proprietary -- you can't audit what you can't see. Even when the provider is acting in good faith, breaches do happen. Hacking incidents are not uncommon in this space.


## Trustless Signaling

The broker's job is introduction -- exchanging connection metadata so peers can find each other. It doesn't need to touch the actual data. What if we designed the protocol so the broker *couldn't* touch the data, even if it tried?

This is the idea behind trustless signaling. The signaling server can be operated by a stranger -- and the connection-level security properties still hold: the network cannot insert itself into your connection, eavesdroppers cannot read your data, and the device will not accept a connection from a party that lacks the access code.

### Self-Sovereign Identity

BitBang's security begins with the device. Each device generates a keypair (RSA or ECC). The hash of the public key becomes its unique identifier (UID):

```
UID = hash(device_public_key)    // 128 bits
```

This is a 128-bit identifier that the device owns. No platform assigns it. The device generates a keypair, computes the hash, and that hash becomes its permanent, verifiable identity.

### The Connection Story

**When the device starts up**, it generates (or loads) its keypair and computes its UID. It connects to the signaling server over a standard TLS-secured WebSocket (WSS) and announces itself: "I am UID abc123, here is my public key." The server verifies that `hash(pubkey)` equals the claimed UID -- a simple check that prevents devices from claiming arbitrary identities. If the hash matches, the server accepts the connection.

The device displays its URL, perhaps by printing it to the console and/or showing a QR code. The URL is `https://bitba.ng/<UID>#<code>` -- the UID is for routing, the code is for authorization.

**When a user opens the URL**, the browser connects to the signaling server and says: "I want to reach UID abc123." The server looks up the device and begins brokering the connection.

**The peers exchange connection metadata.** This is the standard WebRTC handshake: each side generates an SDP (Session Description Protocol) offer containing ICE candidates (network addresses to try) and a DTLS fingerprint (a hash of the cryptographic certificate they'll use for encryption). The signaling server relays these offers between browser and device. It also attaches the device's public key to the offer it forwards to the browser.

**The direct connection is established.** Using the exchanged ICE candidates, the browser and device find a network path to each other -- often a direct peer-to-peer connection and sometimes through a relay. They perform a DTLS handshake to establish encryption. Data can now flow.

But wait -- the signaling server relayed all that metadata. What if it modified the DTLS fingerprints, substituting its own?

```
Browser ← DTLS(R) → Rogue ← DTLS(R) → Device
                      │
                Terminates both
                DTLS sessions.
                Sees all traffic.
```

The browser and device would each establish an encrypted connection to the server, thinking they're talking to each other. The server would sit in the middle, decrypting and re-encrypting traffic -- it's the man-in-the-middle attack described earlier.

### Verifying End-to-End

The key insight: the browser has the device's public key (delivered with the offer above), and it can verify the key is authentic (the hash equals the UID in the URL). A malicious server can't substitute a fake key, because then the hash wouldn't match.

This authentic public key establishes a secure channel that the signaling server can't read or modify.

**Before the WebRTC connection is used for anything sensitive**, the browser encrypts a message with the device's public key:

```
{ my_dtls_fingerprint, random_nonce, access_code }
```

Only the device can decrypt this message because only it has the private key. The browser sends the encrypted blob through the signaling server. The server can forward it, but can't read or modify it. (The `access_code` comes from the URL fragment; its role is covered in *Split Identity* below.)

**The device decrypts the message** and extracts the browser's claimed DTLS fingerprint. It then checks: does this fingerprint match the actual peer I'm connected to via WebRTC?

If a rogue server substituted fingerprints, the browser's encrypted message says "my fingerprint is B", but the device's actual WebRTC peer has fingerprint R (the rogue's fingerprint). The mismatch is detected, and the connection is rejected.

If the fingerprints match, the device knows the WebRTC connection reaches the real browser. But the browser doesn't yet know the connection reaches the real device -- a rogue could be forwarding the encrypted blob without understanding it.

**The device proves its identity** by extracting the random nonce from the decrypted message and sending `hash(nonce)` back over the encrypted WebRTC connection. The browser hashes its own copy of the nonce and compares; if they match, the device has proved it decrypted the payload. Only the real device could produce this response -- only it could decrypt the message. At this point both sides have verified each other, and the channel is trusted end-to-end.

By protocol design, the signaling server -- whether compromised, malicious, or perfectly honest -- has no role in authenticating either side. One runtime question remains for browser clients: who executes these checks? The final section takes it up.

### Why Simpler Schemes Fail

This approach might seem elaborate. Why not just send a PIN over the encrypted WebRTC connection? Or exchange a random code and verify through a side channel?

The problem is that these schemes trust the encryption to be end-to-end. If a rogue broker has already inserted itself -- terminating both sides of the encrypted connection -- then anything sent over that connection is visible to the attacker. The PIN gets relayed. The code gets relayed. Both sides see matching values, but the broker is in the middle reading everything.

The fundamental constraint: anything sent over the encrypted channel can be relayed by a broker that terminates both ends. You can't verify the channel's integrity using only the channel itself.

BitBang breaks this circularity by using the device's public key -- verified independently via the URL -- to establish a secure channel that the broker can't access.

### The Result

| Attack | Blocked by |
|--------|------------|
| Broker substitutes fake public key | `hash(pubkey) ≠ UID` in URL |
| Broker substitutes its own fingerprint | Device sees mismatch with decrypted fingerprint |
| Broker impersonates device | Can't produce correct `hash(nonce)` |
| Broker reads traffic | Can't decrypt -- not the DTLS endpoint |
| Broker connects to device | Doesn't have access code (see below) |

#### What the Server Sees

| Server *can* see | Server *cannot* see |
|------------------|---------------------|
| UIDs of registered devices | Access codes (URL fragments) |
| Which UIDs have connections in flight | PINs |
| Connection timing (and byte counts when traffic is TURN-relayed) | Payloads (HTTP, WebSocket, files, video) |
| Public keys (public by definition) | DTLS session keys |
| IP addresses of devices and connecting browsers (from their connections and relayed ICE candidates) | Private keys (never leave the device or browser) |

A signaling server that gets breached leaks the left column. The right column is protected by the protocol itself.

### Split Identity

Bidirectional verification prevents the server from intercepting connections. But the server brokers all connections -- it sees every UID. What prevents the server from initiating its own connections to devices, or from handing UIDs to anyone who asks?

BitBang URLs carry a second secret -- an access code in the URL fragment:

```
https://bitba.ng/<UID>#<code>
                 └ 128 ┘ └ 64 ┘
```

| Component | Bits | Server sees? |
|-----------|------|--------------|
| UID | 128 | Yes (needed for routing) |
| Code | 64 | No (fragment never sent) |

The URL text after the fragment identifier (`#`) is never transmitted to the server -- by long-standing browser convention, the fragment stays client-side. The code is included in the same encrypted payload as the DTLS fingerprint and nonce, and the device verifies all three together. The server can route connections but can't initiate them. A third party who learns the UID from server logs still can't connect: they're missing the code.

**What the code does and doesn't do:** The code gates authorization, and it can only be guessed online: every guess costs a full WebRTC handshake with the device, which puts exhausting a 64-bit space into geologic time. The code does not strengthen the UID itself -- impersonating a device requires a second preimage of its 128-bit UID, which is infeasible on its own.

The result is a short URL that carries everything a user needs to connect, while keeping the secret bits away from the broker. One property follows directly and is worth stating plainly: the URL is a bearer credential. Anyone who obtains it can connect, so it should be shared the way you'd share a key. And like a key, it can be replaced at any time: because the code is verified only by the device -- never by the server -- issuing a new code invalidates every previously shared URL without changing the device's identity.


## Is It Worth It?

This might seem like a lot of machinery -- public keys, encrypted fingerprints, nonce challenges. Is all this complexity justified?

The encrypted payload is a few hundred bytes. The nonce exchange adds one extra message. The cryptographic operations (one public-key operation, one hash) take milliseconds on ESP32-class hardware; even a Raspberry Pi Pico (ARM Cortex-M0+ core) completes the ECC path in tens of milliseconds. Compared to a naive implementation that simply trusts the signaling server, BitBang's added overhead is well under one percent of bandwidth and compute. 

And critically, all of it is invisible to the user. There are no passwords to type, no keys to store, manage, or exchange, no QR codes to scan for verification, and no out-of-band codes to enter. The user runs the device server, shares the URL it prints, and every cryptographic step happens silently in the background for every connection. What the user actually experiences is the result -- a network that just works, jumping NATs and firewalls, without an account, without configuring, and with direct peer-to-peer encryption end-to-end.

The value is significant:

**Privacy is a property of the protocol.** Trust in a provider isn't just about intent -- even an honest operator can be hacked. Most cloud services hold credentials and session tokens that are valuable targets. The BitBang signaling server holds neither: UIDs are public, payloads are end-to-end encrypted, and there are no user accounts. A hacker who breaches the server's stored data gains nothing that grants access to any device -- there are no credentials to steal. For installed clients (CLI, Python), that is the whole story. For browser clients, a fully compromised server is also a compromised code-delivery channel -- the residual named in The Honest Boundary below. 

**Self-hosting becomes practical.** Anyone can run a BitBang signaling server. Authentication runs in the browser, not on the server -- the operator is a relay, not an authority, and the trust required is reduced accordingly. 

**The honest boundary.** For browser clients, one residual applies: the operator serving that browser's JavaScript is a party you are implicitly trusting -- to deliver honest code that performs the checks. That's the same caveat every web-based crypto tool has (Bitwarden's web vault, ProtonMail's webmail, Signal Web). BitBang gives you three ways to avoid it. **Self-host** the signaling server, and you are the operator. Use the **installed CLI or Python client**, which run no served code -- no per-connection residual remains, and trust reduces to the software you installed, like any local tool. Or **inspect the open runtime** -- the code is open source, and what your browser received can be compared against the repository. The claim is not "trust nobody"; it is that the party you must trust is *named*, *bounded to code delivery only*, and *avoidable* -- properties the closed-source PaaS alternatives cannot offer.

When you're presented with a link, you click on it because you trust the site, or you don't. BitBang offers a third choice -- verify. The authentication is built into the URL itself; the protocol vouches for who's on the other end.
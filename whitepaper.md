# BitBang: End-to-End Verified, Browser-Native Remote Access

---

# Part 1: Overview

## The Problem

A field biologist checks soil moisture sensors at a remote research station -- but only when she drives three hours to visit. A professor wants her students to see live magnetometer readings while lecturing about solar maxima, but IT won't open ports for a Raspberry Pi. A maker builds a beautiful weather station dashboard, but it only works on his local network. A robotics team wants to livestream their rover's camera, but RTSP doesn't traverse NAT.

These aren't exotic problems. Accessing hardware and media remotely is a frequent need -- sensors that should be checkable from home, instruments that could be shared across institutions, prototypes that could be demonstrated live to collaborators, video feeds that could be watched in real-time. These use cases exist everywhere and mostly go unaddressed. Not because the technology doesn't exist, but because the tools are too heavy. ngrok requires accounts and imposes bandwidth caps. Tailscale and ZeroTier require VPN software on both ends. AWS IoT is overly complicated. And streaming live video from a webcam, a Pi camera, or any camera without a proprietary cloud service? There's no easy answer.

So the biologist drives three hours. The students see static screenshots. The maker uploads a screen recording. The rover's live feed stays on the local network.

---

## The Rules of Internet Accessibility

The internet is often imagined as a flat, fully-connected network -- every machine reachable from every other machine. It isn't, of course.

**Rule 1:** Machines on the public internet are accessible from other machines on the public internet -- and from machines on your local network.

**Rule 2:** Machines on your local network are only accessible from other machines on your local network.

Because of rule 2, machines on your local network aren't reachable from outside -- nor are the resources they hold: files, cameras, sensors, compute, or the web app you're currently developing. It's also the reason the biologist can't check her sensors from home. Cloud services exist to fill this gap: Dropbox for files, AWS IoT for sensors, Tailscale for compute, and ngrok for web apps -- among others. These services apply rule 1, but each involves account creation and your data living on someone else's server. Fees are not uncommon.

---

## The Toll Booths

Lots of remote access solutions exist, each with its own costs.

**Cloud platforms (AWS IoT, Azure IoT, Google Cloud IoT):** Powerful but enterprise-focused. They require you to invest time creating complex configurations. You end up locked into their solution, and there's no guarantee they will be around tomorrow. Google Cloud IoT shut down in 2023, for example.

**Tunneling services (ngrok, Cloudflare Tunnel):** Simpler, but account-gated. ngrok has bandwidth caps and connection limits on free tiers. Cloudflare Tunnel requires a Cloudflare account and DNS configuration. Both terminate TLS at their edge -- your traffic is encrypted in transit to them, but they decrypt it before forwarding to your service, so they're capable of seeing its content.

**Mesh VPNs (Tailscale, ZeroTier):** Excellent for connecting machines you control, but both ends need software to be installed. You can't share access with someone who hasn't installed the client.

**Remote desktop (TeamViewer, AnyDesk, RustDesk):** Built for screen sharing, not service access -- overkill for checking a sensor, and requires software to be installed on both ends. 

**P2P toolkits and overlay networks (libp2p, Iroh, Hyperswarm, Veilid):** These are foundational toolkits and network primitives. To use them you build a custom application, ship clients for every platform you want to support, and operate the discovery and relay infrastructure. 

---

## What Is BitBang?

BitBang is a system for end-to-end verified, browser-native remote access. *End-to-end verified* means the browser and device cryptographically prove their identity to each other, eliminating the possibility of a man-in-the-middle attack -- more on this later. BitBang is available as a single-binary command-line tool with subcommands for proxying a local web service (`bitbang proxy`), sharing files (`bitbang fileshare`), and opening a remote shell (`bitbang shell`), each of which
prints a URL anyone can open in a browser.

It's also available as a Python package that gives the same capability to custom programs -- a few lines of code turn a running WSGI or ASGI app into a shareable URL. 

An IoT-network layer that extends the same trust model to groups of devices is in development. Application-specific BitBang implementations are also being developed. A BitBang remote access and video streaming plug-in for the 3d printer app *OctoPrint* is currently available. 

BitBang and all of its components are fully open source, including the signaling server.

---

## How Does BitBang Work?

BitBang connects a device to a browser through a signaling server via WebRTC. (A device is simply a BitBang endpoint -- it can run on anything from an embedded system to a rack-mounted server.) The signaling server brokers the introduction, then steps aside. Data flows directly between peers -- or through an encrypted TURN relay when a direct connection isn't possible.

### Identity and Connection

Each device generates a keypair (RSA or ECC). The hash of the public key becomes its unique identifier (UID). No account is required -- the device owns its identity.

```
bitbang proxy localhost:5000
```

Running this command connects the device to the signaling server and prints a URL: `https://bitba.ng/<UID>#<code>`. The UID (128 bits) identifies the device; the code in the URL fragment authorizes connections. When the device connects to the signaling server, it announces its UID and public key. The server verifies the hash matches, then the device waits for connections.

When a user opens the URL, the browser and device perform a WebRTC handshake -- exchanging ICE candidates and DTLS fingerprints through the signaling server. They establish an encrypted data channel, usually direct peer-to-peer (~75% of networks[^1]), or through a TURN relay. 

### The Data Channel

Once the WebRTC data channel is open, there's a bidirectional pipe between browser and device. Everything else is built on top of this pipe.

**HTTP proxying.** A service worker in the browser intercepts HTTP requests. Instead of going to the network, requests are serialized and sent through the data channel. The device receives them, proxies to the local service (e.g., `localhost:5000`), and sends the response back. The browser deserializes the response and returns it to the page. From the web app's perspective, it's talking to a normal server.

**WebSocket proxying.** The BitBang runtime injects a shim that replaces the WebSocket constructor. When the proxied app opens a WebSocket, the shim routes it through the data channel. The device handles the WebSocket on its end -- typically by bridging to a real WebSocket on the local service -- and messages flow in both directions over the data channel.

**Rendering.** The proxied content is displayed in an iframe. The service worker handles all requests from within the iframe, so the app works normally -- fetching resources, making API calls, opening WebSockets, etc.

**Cookies.** Each connection maintains a cookie jar keyed by UID and target. Cookies set by the proxied app are stored and sent with subsequent requests, maintaining session state across page loads.

All of this happens inside the BitBang *browser runtime* -- a JS layer that's transparent to the user. The result is a seamless browsing experience, delivered through a single encrypted pipe carrying the full web application -- HTML, JavaScript, CSS, API calls, cookies, and WebSockets.

### Media Streaming

WebRTC was designed for video calling, so audio and video media are first-class citizens. BitBang streams video and audio through WebRTC media channels -- separate from the data channel but within the same connection.

For example, the video from the robotics team's rover camera is encoded into H.264 and sent as a WebRTC media track, which all browsers can play natively. Sub-second latency is achieved because it's peer-to-peer.  

---

## Why WebRTC?

BitBang could use any reliable data transport in theory, but WebRTC offers special benefits that make this architecture practical.

| Transport | Browser-native? | P2P possible? | Encrypted? |
|-----------|----------------|---------------|------------|
| Raw TCP relay | No | No | Manual |
| WebSocket relay | Yes | No | TLS only |
| WebTransport relay | Yes | No | Mandatory |
| WebRTC | Yes | Yes (~75%) | Mandatory |

**NAT traversal:** ICE (Interactive Connectivity Establishment) tries multiple paths -- direct connection, STUN-assisted, and TURN relay as fallback. In ~75% of cases, peers connect directly. The Rules of Internet Accessibility are essentially broken when this happens.

**Encryption:** Required DTLS for data channels and SRTP for media. The DTLS fingerprint is what enables the bidirectional verification described in Part 2 -- it's the cryptographic binding between signaling and the data channel.

**Browser-native:** Ubiquitous support across all popular browsers -- no plugins or installs.

**Data channels:** WebRTC supports raw bidirectional data in addition to audio and video. BitBang uses data channels to tunnel HTTP and WebSocket data between browsers and devices.

**Media support:** Video and audio are first-class citizens. The robotics team's rover camera works because WebRTC was designed for exactly this -- real-time media with NAT traversal.

This has practical benefits:

**Low-cost server hosting.** The signaling server primarily brokers connections. Data flows peer-to-peer in most cases.

**Lower latency.** Data travels directly, avoiding the extra hop through a data center.

**Throughput not capped by a third party.** Peer-to-peer data rates aren't bounded by any cloud provider's server capacity.

The ~75% P2P success rate covers most home and office networks. For the remaining ~25% (symmetric NAT, restrictive firewalls), a TCP-based TURN relay provides a secure fallback. The relay only sees encrypted bytes, and the end-to-end verification still holds. BitBang automatically provides TURN or alternatively supports "bring your own TURN".

---

## What BitBang Provides Currently

```
bitbang shell                      # Remote shell, browser- or CLI-accessible
bitbang fileshare ~/Documents      # Share a directory
bitbang proxy localhost:8080       # Proxy a local web service
```

Running the command prints a URL and QR code. Open the URL in any browser and you're connected. Share the URL, and others can connect too. There is no account setup, nothing to install on the viewing side, and no port forwarding.

The same binary can also act as a client, mirroring scp and ssh ergonomics:

```
bitbang cp 'URL:/etc/hosts' ./hosts          # Remote → local file copy
bitbang connect 'URL'                        # Interactive shell
bitbang connect 'URL' -- ls /var/log         # Run a single command
```

For use in custom applications, BitBang is also available as a Python library. Import it, add a couple lines of code, and you've created a BitBang device, which includes a shareable URL with live streaming capability.

| Feature | How |
|---------|-----|
| Shell access | xterm.js in browser, PTY on device |
| File transfer | Browser drag-and-drop, or scp-style CLI |
| Web proxy | Tunnel HTTP/WebSocket to local service |
| Video streaming | WebRTC media channels (H.264) |

---

## Back to the Biologist

The field biologist installs BitBang on the Raspberry Pi at her research station. She simply downloads it and runs it -- `bitbang proxy localhost:5000`, which proxies her sensor dashboard. She scans the QR code with her phone and bookmarks the URL. From her home, she sees live soil moisture data. She didn't need to sign up for an account or get IT involved. And she didn't need to drive the three hours back to her field station.

The professor runs `bitbang proxy localhost:8080` on the Pi connected to her magnetometer. She shares the URL with her class. Thirty students watch live readings during the lecture. The next week she shares it with a colleague at another university.

The maker imports the Python implementation of BitBang into his dashboard program. After modifying two lines of code his weather station is online. He shares the URL with his friends via email, and they're able to see and interact with a live demo of the dashboard on their browsers.

Similarly, the robotics team uses the Python implementation to encode and stream their rover's live video stream. They share the URL with their mentor, who sees a high framerate video feed with sub-second latency.

---

# Part 2: Authentication Without a Central Authority

Part 1 introduced BitBang as *end-to-end verified*. This section shows how that property is achieved without a central authority, and why it holds even when the broker is hostile.

Every remote access system has to answer one question before anything else: is the other end who it claims to be? That's authentication. Most systems delegate it to a central authority -- an account service, an SSO, an IT department. Trust starts there.

BitBang has no account and no central authority. That's by design -- it's a major reason BitBang is easy to use. But is there a cost? Without anyone to vouch for the device, how can the browser know for sure who it's actually talking to?

The answer is unusual: the browser does the authentication itself. The signaling server relays messages but plays no role in verifying identity. Two deliberate design choices make this possible. First, authentication is decoupled from the server -- the browser independently checks the device's public key against the URL, encrypts a challenge to that key, and verifies the response. The server relays the bytes but can't read them. Second, the verifier runs in the browser, where the code is inspectable. By contrast, server-side authentication is a black box -- users find it difficult to trust what they can't see. Together, BitBang's design choices replace operator trust with verifiable execution.

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

---

## Trustless Signaling

The broker's job is introduction -- exchanging connection metadata so peers can find each other. It doesn't need to touch the actual data. What if we designed the protocol so the broker *couldn't* touch the data, even if it tried?

This is the idea behind trustless signaling. The signaling server can be compromised, malicious, or operated by a stranger -- and the security properties still hold.

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

Only the device can decrypt this message because only it has the private key. The browser sends the encrypted blob through the signaling server. The server can forward it, but can't read or modify it. (The `access_code` comes from the URL fragment; its role is covered in *Split Identity* below. For now, focus on the fingerprint and nonce.)

**The device decrypts the message** and extracts the browser's claimed DTLS fingerprint. It then checks: does this fingerprint match the actual peer I'm connected to via WebRTC?

If a rogue server substituted fingerprints, the browser's encrypted message says "my fingerprint is B", but the device's actual WebRTC peer has fingerprint R (the rogue's fingerprint). The mismatch is detected, and the connection is rejected.

If the fingerprints match, the device knows the WebRTC connection reaches the real browser. But the browser doesn't yet know the connection reaches the real device -- a rogue could be forwarding the encrypted blob without understanding it.

**The device proves its identity** by extracting the random nonce from the decrypted message and sending `hash(nonce)` back over the encrypted WebRTC connection. The browser hashes its own copy of the nonce and compares; if they match, the device has proved it decrypted the payload. Only the real device could produce this response -- only it could decrypt the message. At this point both sides have verified each other, and the channel is trusted end-to-end.

The signaling server -- whether compromised, malicious, or perfectly honest -- has no role in authenticating either side.

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

**Security bonus:** An attacker attempting a preimage attack (finding a keypair where `hash(pubkey) == UID`) must now also guess the 64-bit code. Each attempt to guess the code requires a full WebRTC handshake, and the device can rate-limit. This shifts security into the online domain where defenders have the advantage.

The result is a short URL that's safe to share via email, SMS, or QR code. It carries everything a user needs to connect, while keeping the secret bits away from the broker.

---

## Is It Worth It?

This might seem like a lot of machinery -- public keys, encrypted fingerprints, nonce challenges. Is all this complexity justified?

The encrypted payload is a few hundred bytes. The nonce exchange adds one extra message. The cryptographic operations (one public-key operation, one hash) take milliseconds on any modern device, such as an ESP32 or even a Raspberry Pi Pico (ARM Cortex-M0+ core). Compared to a naive implementation that simply trusts the signaling server, BitBang's added overhead is well under one percent of bandwidth and compute. 

And critically, all of it is invisible to the user. There are no passwords to type, no keys to store, manage, or exchange, no QR codes to scan for verification, and no out-of-band codes to enter. The user runs the device server, shares the URL it prints, and every cryptographic step happens silently in the background for every connection. What the user actually experiences is the result -- a network that just works, jumping NATs and firewalls, without an account, without configuring, and with direct peer-to-peer encryption end-to-end.

The value is significant:

**Privacy is a property of the protocol.** Trust in a provider isn't just about intent -- even an honest operator can be hacked. Most cloud services hold credentials, session tokens, and traffic logs that are valuable targets. The BitBang signaling server holds none of that: UIDs are public, payloads are end-to-end encrypted, and there are no user accounts. A hacker who fully compromises the server sees a list of UIDs that are currently online, but can't connect to any of them. 

**Self-hosting becomes practical.** Anyone can run a BitBang signaling server. Authentication runs in the browser, not on the server -- the operator is a relay, not an authority, and the trust required is reduced accordingly. 

When you're presented with a link, you click on it because you trust the site, or you don't. BitBang offers a third choice -- verify. The authentication is built into the URL itself; the protocol vouches for who's on the other end, regardless of who's hosting.

---

## References

[^1]: GetStream, [*STUN and TURN in WebRTC*](https://getstream.io/resources/projects/webrtc/advanced/stun-turn/), citing Chrome UMA data: "direct connections succeed in roughly 75–80% of sessions; the remaining 20–25% require a relay."

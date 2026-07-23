# BitBang: End-to-End Verified, Browser-Native Remote Access

---

## The Problem

A field biologist checks soil moisture sensors at a remote research station -- but only when she drives three hours to visit. A professor wants her students to see live magnetometer readings while lecturing about solar maxima, but IT won't open ports for a Raspberry Pi. A maker builds a beautiful weather station dashboard, but it only works on his local network. A robotics team wants to livestream their rover's camera, but RTSP doesn't traverse NAT.

These aren't exotic problems. Accessing hardware and media remotely is a frequent need -- sensors that should be checkable from home, instruments that could be shared across institutions, prototypes that could be demonstrated live to collaborators, video feeds that could be watched in real-time. These use cases exist everywhere and mostly go unaddressed. Not because the technology doesn't exist, but because the tools are too heavy. ngrok requires accounts and imposes bandwidth caps. Tailscale and ZeroTier require VPN software on both ends. AWS IoT is overly complicated. And streaming live video from a webcam, a Pi camera, or any camera without a proprietary cloud service? There's no easy solution.

So the biologist drives three hours. The students see static screenshots. The maker uploads a screen recording. The rover's live feed stays on the local network.

---

## The Rules of Internet Accessibility

The internet is often imagined as a flat, fully-connected network -- every machine reachable from every other machine. It isn't, of course.

**Rule 1:** Machines on the public internet are accessible from other machines on the public internet -- and from machines on your local network.

**Rule 2:** Machines on your local network are only accessible from other machines on your local network.

Because of rule 2, machines on your local network aren't reachable from outside -- nor are the resources they hold: files, cameras, sensors, compute, or the web app you're currently developing. It's also the reason the biologist can't check her sensors from home. Cloud services exist to fill this gap: Dropbox for files, AWS IoT for sensors, Tailscale for compute, and ngrok for web apps -- among others. These services apply rule 1, but each involves account creation and your data living on someone else's server, and fees are not uncommon.

---

## The Toll Booths

Lots of remote access solutions exist, each with its own costs.

**Cloud platforms (AWS IoT, Azure IoT, Google Cloud IoT):** Powerful but they are often enterprise-focused. They require you to invest time creating compatible configurations. You end up locked into their solution, and there's no guarantee they will be around tomorrow. Google Cloud IoT shut down in 2023, for example.

**Tunneling services (ngrok, Cloudflare Tunnel):** Simpler, but account-gated. ngrok has bandwidth caps and connection limits on free tiers. Cloudflare Tunnel requires a Cloudflare account and DNS configuration. Both terminate TLS at their edge -- your traffic is encrypted in transit to them, but they decrypt it before forwarding to your service, so they're capable of seeing its content.

**Mesh VPNs (Tailscale, ZeroTier):** Excellent for connecting machines you control, but both ends need software to be installed. You can't share access with someone who hasn't installed the client.

**Remote desktop (TeamViewer, AnyDesk, RustDesk):** Built for screen sharing, not service access -- overkill for checking a sensor, and requires software to be installed on both ends. 

**P2P toolkits and overlay networks (libp2p, Iroh, Hyperswarm, Veilid):** These are foundational toolkits and network primitives. To use them you build a custom application, ship clients for every platform you want to support, and operate the discovery and relay infrastructure. 

Notice the common thread: every option requires installing software on the consuming side, creating an account, or both, but it doesn't have to be this way.

---

## Origin

In 2010, I gave a Google Tech Talk with Illah Nourbakhsh of Carnegie Mellon on TeRK (Telepresence Robot Kit), a Google- and Intel-funded project to make educational robotics accessible. An important conclusion had little to do with robots: the internet is broken for devices. NATs and firewalls make every device behind a home router invisible from the outside. Google had recently open-sourced libjingle -- the P2P engine behind Google Talk -- and we wanted to use it for zero-config device connectivity. But it was a C++ library tied to XMPP, and we felt strongly the technology should be browser-friendly. Bridging such a gap was beyond our resources.

In the years that followed, connected devices became "IoT" and converged on a single pattern: Platform-as-a-Service. Your device talks to a cloud, your browser talks to the same cloud, and they meet in the middle -- accounts, subscriptions, and your data on someone else's server. Meanwhile WebRTC, libjingle's direct descendant, quietly shipped in every major browser. But it was built for video calls, and by the time it arrived, IoT had already settled into its cloud pattern -- peer-to-peer never entered the picture. BitBang is what happens when it does: devices talk directly to browsers, with no cloud in the data path and no account required. It was built for [Goby](https://hackaday.com/2025/04/17/tiny-hackable-telepresence-robot-for-under-100-meet-goby/), a tiny telepresence robot that launched on Kickstarter in 2025 -- the spiritual descendant of TeRK, realized with the technology we had wished for fifteen years earlier. We promised to open-source the networking stack that made it possible. BitBang is that stack.

---

## What Is BitBang?

BitBang is a system for end-to-end verified, browser-native remote access. *End-to-end verified* means the browser and device cryptographically prove their identity to each other, so the signaling path cannot silently insert itself into the connection. The precise boundary of this guarantee is described in detail here: [*Trustless Signaling: Authentication Without a Central Authority*](trustless-signaling.md).

One definition before going further: in this paper, a *device* is whatever runs BitBang -- the endpoint being reached. That might be a Raspberry Pi at a field station, a workstation, a rack-mounted server, a $4 microcontroller, or simply a program that imports the BitBang library. To BitBang they are all the same thing: an endpoint with a keypair and a URL.

BitBang ships as a single-binary command-line tool and as a Python package for custom programs. Both, along with the signaling server, are fully open source.

An IoT-network layer that extends the same trust model to groups of devices is in development, and the same links will carry any byte stream -- not just web apps, but network drives, remote desktops, even serial ports. Application-specific BitBang implementations are also being developed. A BitBang remote access and video streaming plug-in for the 3d printer app *OctoPrint* is currently available. 

---

## How Does BitBang Work?

BitBang connects a device to a browser through a signaling server via WebRTC. The signaling server brokers the introduction, then steps aside. Data flows directly between peers -- or through an encrypted TURN relay when a direct connection isn't possible.

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

For example, the video from the robotics team's rover camera is encoded into H.264 and sent as a WebRTC media track, which all browsers can play natively. Direct peer-to-peer delivery typically yields sub-second latency.  

---

## Why WebRTC?

Recall the common thread among the Toll Booths: every one requires a client install. The reason is structural. Each built its data plane on a transport that cannot exist inside a browser -- WireGuard tunnels, custom UDP protocols, proprietary agents. A browser can't open a raw socket and can't join a VPN, so any system built on those transports must ship a client. There is exactly one peer-to-peer transport that every browser already contains: WebRTC. Build the data plane there, and the client problem disappears -- the "app" is a URL, on the billions of devices that already have a browser. And the choice isn't limiting: the same protocol runs outside the browser too -- the BitBang CLI can also speak it directly, machine to machine. Browser-friendly, not browser-limited. A mesh VPN ported to the browser must either relay every packet through its own servers, giving up peer-to-peer, or carry its traffic over WebRTC, at which point it looks a lot like BitBang.

Beyond browser reach, WebRTC brings the rest of what the architecture needs:

| Transport | Browser-native? | P2P possible? | Encrypted? |
|-----------|----------------|---------------|------------|
| Raw TCP relay | No | No | Manual |
| WebSocket relay | Yes | No | TLS only |
| WebTransport relay | Yes | No | Mandatory |
| WebRTC | Yes | Yes (~75%) | Mandatory |

**NAT traversal:** ICE (Interactive Connectivity Establishment) tries multiple paths -- direct connection, STUN-assisted, and TURN relay as fallback. In ~75% of cases, peers connect directly. The Rules of Internet Accessibility are essentially broken when this happens.

**Encryption:** Required DTLS for data channels and SRTP for media. The DTLS fingerprint is the cryptographic binding between signaling and the data channel -- the anchor of BitBang's end-to-end verification.

**Data channels:** WebRTC supports raw bidirectional data in addition to audio and video. BitBang uses data channels to tunnel HTTP and WebSocket data between browsers and devices.

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

Running any of these prints a URL and QR code. Open the URL in any browser and you're connected -- an interactive terminal for the shell, drag-and-drop file access for the share, the proxied app itself for the proxy. Share the URL, and others can connect too. There is no account setup, nothing to install on the viewing side, and no port forwarding.

The same binary can also act as a client, mirroring scp and ssh ergonomics:

```
bitbang cp 'URL:/etc/hosts' ./hosts          # Remote → local file copy
bitbang connect 'URL'                        # Interactive shell
bitbang connect 'URL' -- ls /var/log         # Run a single command
```

For custom applications, the Python package does the same job in code: a few lines turn a running program -- a WSGI or ASGI web app, a video source, a sensor loop -- into a BitBang device with a shareable URL.

---

## Back to the Biologist

The field biologist installs BitBang on the Raspberry Pi at her research station. She simply downloads it and runs it -- `bitbang proxy localhost:5000`, which proxies her sensor dashboard. She scans the QR code with her phone and bookmarks the URL. From her home, she sees live soil moisture data. She didn't need to sign up for an account or get IT involved. And she didn't need to drive the three hours back to her field station.

The professor runs `bitbang proxy localhost:8080` on the Pi connected to her magnetometer. She shares the URL with her class. Thirty students watch live readings during the lecture. The next week she shares it with a colleague at another university.

The maker imports the Python implementation of BitBang into his dashboard program. After modifying two lines of code his weather station is online. He shares the URL with his friends via email, and they're able to see and interact with a live demo of the dashboard on their browsers.

Similarly, the robotics team uses the Python implementation to encode and stream their rover's live video stream. They share the URL with their mentor, who sees a high framerate video feed with sub-second latency.

---

## Going Deeper

This paper opened by claiming BitBang is *end-to-end verified*: the browser and device prove their identity to each other, with no account and no central authority. How that works, and why it holds no matter who runs the signaling server, is covered in detail here: [*Trustless Signaling: Authentication Without a Central Authority*](trustless-signaling.md).

---

## References

[^1]: GetStream, [*STUN and TURN in WebRTC*](https://getstream.io/resources/projects/webrtc/advanced/stun-turn/), citing Chrome UMA data: "direct connections succeed in roughly 75–80% of sessions; the remaining 20–25% require a relay."
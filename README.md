# BitBang

BitBang is remote access without the account.

This repository is the entry point to the BitBang project. Check out the related repos linked below.

## Try it now (quick filesharing demo)

Install:
```bash
pip install bitbang              # Linux / macOS
python -m pip install bitbang    # Windows (or any platform) 
```

Quick test:
```bash
bitbang-fileshare ~/Downloads            # Linux / macOS
python -m bitbang fileshare ~/Downloads  # Windows (or any platform)
```
![bitbang-fileshare](https://raw.githubusercontent.com/richlegrand/bitbang/refs/heads/main/bitbang_screen.png)

## Why BitBang is needed

The Internet is often thought of as a fully connected network -- every machine is accessible from every other machine. But there are rules governing accessibility on the Internet...

### Rules of Internet Accessibility

1. Machines on the Internet are accessible by other machines on the Internet -- and by machines on your local network.
2. Machines on your local network are only accessible by other machines on your local network.

Because of rule 2, machines on your local network aren't reachable from outside -- nor are the resources they hold: files, cameras, sensors, compute, or the web app you're currently developing. Cloud services exist to fill this gap: Dropbox for files, AWS IoT for sensors, Tailscale for compute, and ngrok for web apps -- among others. These services apply rule 1, but each comes with the friction of account creation, fees, and your data living on someone else's server.

## How BitBang compares

| | ngrok | Cloudflare Tunnel | Tailscale | BitBang |
|---|---|---|---|---|
| Account required | Yes | Yes | Yes | No |
| Free tunnels | 1 | Unlimited | Unlimited | Unlimited |
| Data path | Their servers | Their servers | P2P | P2P |
| Viewer needs install | No | No | Yes | No |
| Configuration | CLI flags | Config file + DNS | Dashboard | None |

BitBang's data path is direct between peers. The signaling server brokers the initial connection, then steps aside.


## How it works

Browsers normally connect to web servers over a TCP socket. BitBang replaces this with a WebRTC data channel.

![BitBang Python Block Diagram](https://raw.githubusercontent.com/richlegrand/bitbang/refs/heads/main/bitbang_diagram.png)

The signaling server (`bitba.ng`) brokers the WebRTC handshake, then has no further involvement and never sees application data. 

### WebRTC?

WebRTC is the behind-the-scenes technology that makes Zoom and Google Meet video conferencing possible. WebRTC offers the highest bandwidth and lowest latency possible, which is good when you're streaming live video, or practically anything else. It's mature, well-tested, and has ubiquitous support across all browsers. In addition to delivering low-latency media, it can also deliver raw data over "data channels", which is what BitBang uses for proxying HTML and WebSockets.


### Signaling server

The signaling server source is available [here](https://github.com/richlegrand/bitbang-server). The signaling server does the following:

1. Serves the BitBang browser runtime
2. Authenticates connecting devices via RSA challenge
3. Maintains WebSocket connections to active devices
4. Brokers ICE candidate and SDP exchange between browsers and devices

After the P2P connection is established, the signaling server is not involved. We are providing a signaling server for testing, etc. at https://bitba.ng. It mostly brokers connections, so its resource needs are small. 

## Security

WebRTC mandates encryption:

- **Data channels**: DTLS 1.2+
- **Media streams**: SRTP 
- **Signaling**: HTTPS and WSS

Furthermore, each BitBang "device" generates an RSA keypair. The public key hash becomes its unique 128-bit ID, which is used in its BitBang public URL. A 64-bit access code rides in the URL fragment (after `#`) so the signaling server -- which sees the path but never the fragment -- can route connections but cannot initiate them. See this [whitepaper](whitepaper.md#trustless-signaling) for BitBang's full trustless-signaling model.

## Repositories

**[bitbang-python](https://github.com/richlegrand/bitbang-python)** -- Python library that wraps any WSGI or ASGI application (Flask, FastAPI, Quart, etc.) and exposes it over a BitBang URL. Includes example apps: `bitbang-fileshare` for sharing local files, and `bitbang-webcam` for streaming a webcam to a browser. Available on PyPI as `bitbang`.

**[bitbang-server](https://github.com/richlegrand/bitbang-server)** -- Signaling server and browser runtime. Brokers the WebRTC handshake, authenticates devices via RSA challenge, and serves the browser-side code that runs the data channel. It powers `bitba.ng`, and can be self-hosted.

**[bitbang-cli](https://github.com/richlegrand/bitbang-cli)** -- Single executable that solves several remote access problems, such as proxying local apps, file sharing, and shell access. 

**[Octoprint-BitBang](https://github.com/richlegrand/Octoprint-BitBang)** -- OctoPrint plugin that provides remote access to an OctoPrint instance through a single shareable URL, including hardware-encoded H.264 video from the printer's camera. It seamlessly tunnels the full OctoPrint UI, WebSockets, file uploads, and timelapse.

## Origin

In 2010, I gave a [Google Tech Talk](https://www.youtube.com/watch?v=OaDNhpWDmyg) with Illah Nourbakhsh from Carnegie Mellon. We talked about the Telepresence Robotics Kit (TeRK), a Google/Intel-funded project which aimed to make educational robotics accessible to college and pre-college students. The message we delivered was also one of the main project takeaways: the Internet is broken for devices.

NATs and firewalls block inbound connections. Every device behind a home router is invisible to the outside world. Back then, Google had recently open-sourced libjingle -- the P2P networking code behind Google Talk -- which used STUN and ICE to traverse NATs. But it was a C++ library tied to XMPP, not something that ran in browsers. We wanted to leverage it for "zero-config device connectivity", but the problem was too big for our resources. If only this technology could run in a browser...

In the years that followed, connected devices became "IoT" and a single design pattern emerged: Platform-as-a-Service. Your thermostat talks to a cloud, your phone talks to the same cloud, and they meet in the middle. It works, but it means accounts, subscriptions, and your data on someone else's server. Meanwhile, WebRTC, which is a direct descendant of libjingle, has quietly been adopted and now ships with every major browser. 

BitBang is what happens when you connect those two facts. Devices talk directly to browsers, peer-to-peer with no cloud in the data path, and no account required. BitBang was built for Goby, a tiny telepresence robot that launched on Kickstarter in 2025. Goby is the spiritual descendant of TeRK, finally realized with the technology we had wished for fifteen years earlier. It proved the concept: a device that self-certifies, that you can reach from anywhere with nothing but a browser and a URL. We promised to open-source the networking stack that made it possible. BitBang is that stack.

## License

MIT, across all repositories.

## Contributing

Issues and PRs are welcome. 

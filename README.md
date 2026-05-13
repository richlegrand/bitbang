# BitBang

BitBang is remote access without the account.

This repository is the entry point to the BitBang project. Check out the related repos linked below.

## Try it now (quick filesharing demo)

Install:
```
pip install bitbang              # Linux / macOS
python -m pip install bitbang    # Windows (or any platform) 
```

Quick test:
```bash
bitbang-fileshare ~/Downloads            # Linux / macOS
python -m bitbang fileshare ~/Downloads  # Windows (or any platform)
```
![bitbang-fileshare](https://raw.githubusercontent.com/richlegrand/bitbang/refs/heads/main/bitbang_screen.png)


## What else can you do with it?

- Full access to your NAS[^1]  / OctoPrint[^2] / Jellyfin[^1] / Plex[^1] / Open WebUI[^1] / Flask app[^3] / Node-RED dashboard[^1] / etc. from outside your network
- Build Python web apps that are instantly accessible from anywhere[^3] 
- Stream live video directly to your browser[^4]

Do these things without setting up an account.

[^1]: Download and run [bitbangproxy](https://github.com/richlegrand/bitbangproxy) to access apps remotely 
[^2]: Install [OctoPrint-BitBang](https://github.com/richlegrand/OctoPrint-BitBang) plugin
[^3]: See [bitbang-python](https://github.com/richlegrand/bitbang-python)
[^4]: See [bitbang-python webcam](https://github.com/richlegrand/bitbang-python#webcam) for a simple example

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

### WebRTC

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

Furthermore, each BitBang "device" generates an RSA keypair. The public key hash becomes its unique 128-bit ID, which is used in its BitBang public URL. The signaling server challenge-verifies key ownership (and hence ID) before accepting connections (authenticates).

## Repositories

### Core

**[bitbang-python](https://github.com/richlegrand/bitbang-python)** -- Python library that wraps any WSGI or ASGI application (Flask, FastAPI, Quart, etc.) and exposes it over a BitBang URL. Includes example apps: `bitbang-fileshare` for sharing local files, and `bitbang-webcam` for streaming a webcam to a browser. Available on PyPI as `bitbang`.

**[bitbang-server](https://github.com/richlegrand/bitbang-server)** -- Signaling server and browser runtime. Brokers the WebRTC handshake, authenticates devices via RSA challenge, and serves the browser-side code that runs the data channel. It powers `bitba.ng`, and can be self-hosted.

### Applications

**[bitbangproxy](https://github.com/richlegrand/bitbangproxy)** -- Standalone Go binary that proxies any local web app (e.g. NAS, Jellyfin, Plex, Open WebUI, Node-RED, etc.) so it can be accessed remotely. The target is specified in the URL at browse-time, so a single proxy instance can reach any host on the local network. 

**[Octoprint-BitBang](https://github.com/richlegrand/Octoprint-BitBang)** -- OctoPrint plugin that provides remote access to an OctoPrint instance through a single shareable URL, including hardware-encoded H.264 video from the printer's camera. It seamlessly tunnels the full OctoPrint UI, WebSockets, file uploads, and timelapse.

## Origin

BitBang was built for [Goby](https://hackaday.com/2025/04/17/tiny-hackable-telepresence-robot-for-under-100-meet-goby/), a tiny telepresence robot that ran as a Kickstarter campaign in 2025. We promised that the networking technology that made Goby possible (BitBang) would be released and open-sourced. These repositories contain the complete BitBang codebase. 

## License

MIT, across all repositories.

## Contributing

Issues and PRs are welcome. 

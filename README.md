# BitBang

Direct browser-to-device connectivity over WebRTC. No accounts, no port forwarding, no cloud in the middle.

This repository is the entry point to the BitBang project. The implementation lives in the related repos linked below.

## Background

The Internet is often thought of as a fully connected network -- every machine is accessible from every other machine. But there are rules governing accessibility on the Internet...

### Rules of Internet Accessibility

1. Machines on the Internet are accessible by other machines on the Internet -- and by machines on your local network.
2. Machines on your local network are only accessible by other machines on your local network.

Because of rule 2, machines on your local network aren't reachable from outside -- nor are the resources they hold: files, cameras, sensors, compute, or the web app you're currently developing. Cloud services exist to fill this gap: Dropbox for files, AWS IoT for sensors, Tailscale for compute, and ngrok for web apps -- among others. These services apply rule 1, but each comes with the friction of account creation, fees, and your data living on someone else's server.

## BitBang

BitBang connects a browser directly to any machine on your local network, from anywhere on the Internet. No cloud intermediary, no account, no third party in the middle. It uses a novel application of the peer-to-peer technology WebRTC.

A small signaling server brokers the WebRTC handshake and then steps aside. After that, traffic flows directly between the browser and the target machine over an encrypted data channel (DTLS). The server never sees application data.

Each BitBang device generates an RSA keypair on first run. The public key hash becomes its 128-bit ID, which appears in the device's BitBang URL. The signaling server challenge-verifies key ownership before brokering connections, so the URL itself is the device's identity.

A hosted signaling server is available at [bitba.ng](https://bitba.ng) for testing and general use, and can also be self-hosted.

## Repositories

### Core

**[bitbang-python](https://github.com/richlegrand/bitbang-python)** -- Python library. Wraps any WSGI or ASGI application (Flask, FastAPI, Quart, etc.) and exposes it over a BitBang URL. Includes example apps: `bitbang-fileshare` for sharing local files, and `bitbang-webcam` for streaming a webcam to a browser. Available on PyPI as `bitbang`.

**[bitbang-server](https://github.com/richlegrand/bitbang-server)** -- Signaling server and browser runtime. Brokers the WebRTC handshake, validates devices via RSA challenge, and serves the browser-side code that runs the data channel. Powers `bitba.ng`, and can be self-hosted.

### Applications

**[bitbangproxy](https://github.com/richlegrand/bitbangproxy)** -- Standalone Go binary. Proxies any local web server (NAS, router, media server, dev server) through a WebRTC data channel. The target is specified in the URL at browse-time, so a single proxy instance can reach any host on the local network. No Python required on the target machine.

**[Octoprint-BitBang](https://github.com/richlegrand/Octoprint-BitBang)** -- OctoPrint plugin. Provides remote access to an OctoPrint instance through a single shareable URL, including hardware-encoded H.264 video from the printer's camera. Tunnels the full OctoPrint UI, WebSockets, file uploads, and timelapse over the same WebRTC connection.

## License

MIT, across all repositories.

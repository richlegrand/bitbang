# Device origins

When you open a device at `https://bitba.ng/<uid>#<code>`, its pages don't run
in bitba.ng's origin. Each device gets an origin of its own,
`https://<label>.bitba.ng`, inside the page you opened, and the address bar
never changes. This document explains how that works, why it takes two nested
frames, and what it protects against. The terms and invariants, in short form,
are in `CONVENTIONS.md` ("Device origins: the three frames"); the browser side
is `web/bootstrap.js`, `web/connection.js` and `web/label-sw.js` in
bitbang-server, and the label host is `internal/handler/label.go` there.

```
address bar:  https://bitba.ng/<uid>#<code>/settings
+---------------------------------------------------------------+
| bitba.ng's page (top level)                                   |
|   the access code and the device's public key; computes the   |
|   label; the address bar, title and icon; the PIN prompt      |
|                                                               |
|   +-------------------------------------------------------+   |
|   | connection frame: https://<label>.bitba.ng/__bb/start |   |
|   |   signaling socket, RTCPeerConnection, data channel,  |   |
|   |   streams; never navigates; installs the worker       |   |
|   |                                                       |   |
|   |   +------------------------------------------------+  |   |
|   |   | content frame: https://<label>.bitba.ng/...    |  |   |
|   |   |   the device's pages, at their own paths;      |  |   |
|   |   |   navigates freely                             |  |   |
|   |   +------------------------------------------------+  |   |
|   +-------------------------------------------------------+   |
+---------------------------------------------------------------+
```

Three frames, each with one job:

- **bitba.ng's page** keeps the access code and never gives it out.
- **The connection frame,** in the device's origin, holds the WebRTC
  connection.
- **The content frame,** in the same origin, shows the device's pages.

## Why

Until November 2026 every device's pages ran in bitba.ng's own origin, told
apart only by a path prefix: the device frame loaded `/__device__/<session>/...`,
and a shim stripped the prefix again, because single-page apps route on
`location.pathname` and can't match a route with a prefix in it. That had two
problems.

**Routing by guesswork.** Stripping the prefix erased the one thing that said
which device a request belonged to, from the URL and the referrer both. So the
service worker inferred it, with seven strategies ranging from a recorded
client binding to "whichever session was active most recently." Each new way a
browser navigates a frame needed another rule: an app's post-login
`location.href = '/'`, back from the settings page, forward to it again, and
any link at all in Firefox, which gives a navigation no client id. Each fix was
sound; together they were a defense that grew by a rule per browser behavior.

**No isolation.** Every device page was same-origin with bitba.ng, so script in
any device page -- a NAS's admin page proxied by the CLI, say, or a
third-party script it loads -- could:

- read every open device's access code, which the worker kept in storage the
  page could also read;
- read the top page's URL, which carries the current code;
- register a service worker of its own on bitba.ng and take over every session
  in that browser;
- read other devices' cookies.

One compromised device UI could take the access code for every device the
browser had open. In a device's own origin, the browser keeps all of that apart
itself, and about 600 lines of session inference, prefix rewriting and
cross-device cookie partitioning were deleted rather than rearranged.

## What it defends against

The attacker here is **rogue code in the browser that claims to come from a
device**: a device's own pages gone bad, a third-party script they load, or a
compromised LAN app the CLI proxies.

Two things are assumed:

- **bitba.ng's own code is trustworthy:** bootstrap, the connection frame, the
  label worker and the shims. Whoever controls what bitba.ng serves controls
  all of it, which is the residual trust `trustless-signaling.md` describes.
- **A device's content is trusted with that device's session.** While its page
  is open, it can use the connection it's in. The user opened it.

So the goals, for rogue content in one device's frames:

1. **Contained to its device.** No access to other devices' sessions, codes,
   cookies or storage, or to bitba.ng's origin.
2. **No standing access.** It never sees the access code, and gets nothing it
   can use to connect again once the tab is closed.

It may still, while the tab is open, use the session it's in, mislead within
its own frames, and break things.

## Why three frames

Each boundary is there for a reason.

**bitba.ng's page stays on top.** Links, QR codes and bookmarks keep working
unchanged, and the uid and code never appear in a hostname, so they stay out
of DNS and out of the TLS handshake's server name.

**The code stays out of the device's origin.** Everything in an origin is
reachable by every script in it -- storage, functions, message listeners. If the
connection frame held the code, the device's pages would hold it too. So it
doesn't: for each connection it asks bitba.ng's page for a proof (*Connecting*).
That is why the connection can't simply live in bitba.ng's page with the device
in a single frame below it.

**The connection gets a frame of its own.** The device's pages navigate, reload
and submit forms; the connection frame never navigates, so none of that drops
the WebRTC connection. And the two inner frames share an origin, so the content
frame uses the connection frame's `MediaStream`s and stream ports directly.
WebRTC video, H.264 included, plays natively, with no relay and no re-encoding.
A connection in bitba.ng's page couldn't hand its video to a cross-origin frame
that way.

**The device's pages see their real paths.** Nothing is prefixed or stripped;
an app routes as it would on the LAN. The label worker serves exactly one
device, so back, forward, `location.href = '/'`, a form post and `/*settings`
all resolve without inference. For an app the CLI proxies, the target isn't in
the path either: the connection frame puts `/<target>` back on each request.

## The label

What the label has to be:

1. **Different for each device, and each link,** in this browser.
2. **The same across visits** in this browser, so the device's cookies and saved
   state persist. A new origin per visit would log you out on every refresh.
3. **Unlinkable to the uid.** It is visible on the network, so it can't be the
   uid or anything computable from it, such as a plain hash.
4. **A valid DNS label:** lowercase letters and digits.

bitba.ng's page creates a random 32-byte secret on your first visit and keeps
it in its own `localStorage` (`bb-label-secret`), which no device origin can
read. It never leaves the browser. The label is

    label = base32(HMAC-SHA256(secret, uid + "/" + target + "/" + code))[:16]

-- 10 bytes, 16 characters. `target` is the LAN host for an app the CLI proxies,
and empty otherwise, so each proxied app gets its own origin. The code is in it
so a user link and an admin link to the same device get different origins;
otherwise a script in a user-level tab could ride an admin-level tab's
connection.

What follows:

- **Labels are per browser.** Another browser, or a private window, gets other
  labels, so the network can't even tell that two visitors opened the same
  device. The cost is that a device's cookies don't follow you across browsers.
- **Clearing bitba.ng's site data** deletes the secret, and every device starts
  fresh. Browsers clear subdomains with the site, so nothing is left behind.
- **Without storage** (some private modes), the secret lasts for that page
  only: every visit gets new labels, and the device's cookies don't survive --
  a fresh session each time, which is what private mode asks for.
- **A label is partly a key,** which is why it's kept unguessable. The
  same-origin rule stops other pages from *reading* a device's origin, but
  anyone who knows its label can make your browser *send* it requests: a
  top-level window, a form's `target`, an iframe of their own. The label worker
  would answer over your open connection, with your cookies. So it answers only
  requests from its own frames (*Keeping the origin ours*). A label appears on
  the network and in the server's logs, so that check is what matters; the
  label being unguessable is a second line.

## Connecting

The `RTCPeerConnection` lives in the connection frame; everything that touches
the code stays in bitba.ng's page.

1. bitba.ng's page gets the device's public key from the signaling server,
   checks `hash(pubkey) == uid`, and passes the uid, flags and public key down
   to the connection frame. Not the code.
2. The connection frame opens the signaling socket and receives the device's
   offer, which carries a fresh **challenge**: 16 random bytes, new for every
   offer.
3. It makes its answer and sends bitba.ng's page the answer's DTLS fingerprint
   and the challenge.
4. bitba.ng's page picks a random nonce and encrypts
   `{fp, nonce, code, challenge}` to the device's key. It returns the encrypted
   payload and `sha256(nonce)` -- not the nonce, and not the code.
5. The connection frame sends the answer and the payload. The device decrypts
   it, checks the code, that the challenge is the one it issued (and spends
   it), and that the fingerprint is the answer's, and accepts.
6. When the data channel opens, the device's first message must be
   `sha256(nonce)`, which proves it decrypted the payload. The connection frame
   compares it with what bitba.ng's page sent down.

The two random values prove different things. The challenge is the device's:
it proves to the device that the payload is new, so a captured one can't be
replayed. The nonce is the browser's: returned hashed, it proves to the browser
that it reached the real device. Neither can do the other's job, since a value
only proves freshness to whoever chose it. `trustless-signaling.md` covers the
verification itself, and `signaling.md` the messages.

The challenge matters more here than it did before. Payloads now pass through
the device's origin, where a rogue script could copy one; the challenge makes
a copy worthless. Listeners still accept the older payload without a
challenge, for connectors that predate it, and log it as replayable. To stop a
rogue script asking for that older form, bitba.ng's page remembers each device
that has sent a challenge (`localStorage`, `bb-challenge:<uid>`) and refuses a
challenge-less proof for it from then on. A later release makes the challenge
required.

bitba.ng's page answers only its own connection frame (checked by
`event.source` and origin), which keeps other sites out. It can't tell the
connection frame from a device script driving it, since they share an origin,
and it doesn't need to: all it ever hands out is a single-use proof for the
connection being made.

Reconnecting repeats steps 2-6. A PIN, if the device asks for one, is prompted
for by bitba.ng's page and passed down; it goes through the device's origin,
which is acceptable for a second factor that is useless without the code.

## Keeping the origin ours

A device's origin holds our code -- the connection frame and the label worker --
alongside the device's. These keep the device's side from replacing ours:

- **A label host never serves device content from the network.** It serves
  bitba.ng's own files and the start page, and nothing else. A service worker
  script has to come from the network as JavaScript, so device script can't
  register a worker of its own: the only JavaScript a label host serves is
  ours.
- **The worker trusts nothing in its origin's storage.** Device script can
  write the Cache API, IndexedDB and `localStorage`, so the worker always
  fetches the start page, `connection.js`, the shims and the settings and
  console pages from the network. A copy planted in storage would otherwise be
  served as ours on every later visit, long after the device's pages were
  fixed.
- **The worker answers only its own frames.** It answers a request only from
  one of its own clients, or a navigation one of them started, and refuses
  top-level documents. Label responses carry `Referrer-Policy: same-origin`, so
  the origin's own navigations name it; that is how a navigation is attributed
  in Firefox, which gives navigations no client id.
- **Framing.** Every label response carries `frame-ancestors 'self' bitba.ng`.
  Many LAN apps send `X-Frame-Options: SAMEORIGIN` (Home Assistant, NAS and
  router pages), which would leave them blank now that the top page is
  another origin, so the worker strips the device's own framing headers and
  sets that line instead. The rest of a device's CSP is left alone.
- **No frame-busting.** Both frames are sandboxed without
  `allow-top-navigation`, so a device page can't take over the tab.

## Navigation, new tabs and video

- **The address bar** follows the content frame, so refresh and bookmarks work.
  The connection frame reports the content frame's path, title and icon to
  bitba.ng's page, which mirrors the path into the fragment with
  `history.replaceState`. Back and forward then work through the content
  frame's own history.
- **New tabs** (`target="_blank"`, `window.open`, a ctrl- or middle-click on a
  link the shim sees) go through bitba.ng's page, which opens
  `/<uid>#<code><path>`: an ordinary bitba.ng page with its own connection, in
  the same device origin. An app that talks to its popup through
  `window.opener` breaks; that's rare in device UIs.
- **A label URL loaded on its own** -- "Open link in new tab", a link dragged to
  the tab bar -- carries no uid and no code by design, so it can't find its way
  back. It gets a short page saying to open the device from its bitba.ng link.
- **Video:** a `<video data-bitbang-stream>` in the content frame gets the
  connection frame's `MediaStream` directly. Data-channel streams (MJPEG,
  audio) use a stream port, and `window.__bitbang` is provided by the shim, so
  device pages didn't change.
- **Permissions:** both frames allow `autoplay`, `fullscreen`, the clipboard,
  picture-in-picture, the camera and the microphone. A camera or microphone
  prompt names bitba.ng.

## Cookies

A service worker can't put `Set-Cookie` on a response it builds, so the label
worker keeps a cookie jar: it stores the cookies the device sets, sends them
with later requests, and mirrors the ones a page may read into its
`document.cookie`. Before, one jar held every device's cookies, partitioned by
hand, and the mirror wrote them all into bitba.ng's shared `document.cookie`,
where any device's page could read another's. Now each origin has its own jar,
and the browser keeps one device's cookies from another's.

Session cookies (no expiry) go when no tab has had that device open for about
five minutes, much as a browser drops them at the end of a session; a reload,
or a tab closed and reopened soon after, keeps them.

**Cookie tossing.** Subdomains can set real browser cookies for the whole of
`bitba.ng` (`Domain=bitba.ng`), which the browser then sends to bitba.ng and to
every other label host. bitba.ng sets no cookies of its own, and trusts none.
Tossed cookies could also be a denial of service: enough large ones push the
Cookie header past the server's limit, and every request in that browser
fails. So the server's front end drops the Cookie header on every request to
bitba.ng and its label hosts (none of them use browser cookies), with header
buffers large enough that an oversized one is dropped rather than refused. If
a browser's cookies for the site are ever flooded anyway, visiting
`https://bitba.ng/__bb/clear-cookies` clears them.

A tossed cookie never reaches a device: requests to a device carry only its
own jar's cookies, over the data channel. But it can show up in another
device's `document.cookie`, which the shim reads from the browser where the
browser keeps cookies written there. So a device page can plant a value
another device's page reads, though not one the other device receives.

## Limits

What this design doesn't do, stated so nobody assumes otherwise:

- **A connection can outlive the tab.** bitba.ng's page encrypts a proof for
  whatever fingerprint and challenge the connection frame sends, and rogue
  script in the device's origin can drive the connection frame. So while your
  tab is open, it could have its own machine connect to the device, take that
  connection's challenge, ask for a proof for its own fingerprint, and connect.
  The challenge doesn't stop this: it's a fresh proof for the attacker's own
  connection. That connection can't make others and is no credential -- goal 2
  holds -- but it stays up until the device or the attacker drops it. The fix is
  for bitba.ng's page to generate the DTLS certificate itself and authorize
  only that certificate's fingerprint, so every connection has to end in this
  browser. Chromium and Firefox accept a certificate passed to the connection
  frame; WebKit refuses it, so on Safari, and on every browser on iOS, the gap
  would stay open. Not built yet.
- **A connection carries the link's whole grant.** For a device's own pages
  that's what the content shows. For the CLI it may not be: an owner link
  grants everything the listener serves -- shell, files, every proxied app -- so
  a compromised proxied app opened from an owner link could use the shell over
  the connection it's in. Until the payload carries a scope the device
  enforces, share a link scoped to the one app instead.
- **HttpOnly protects nothing inside the origin.** The jar lives in the
  origin's storage, which any script there can read, so a third-party script on
  a device's page can read that device's HttpOnly session cookie and, if the
  app's sessions outlive the connection, keep using it in that app after the
  tab closes. It stays within that one device.
- **Safari's storage cap** may delete script-written storage after seven days
  without a visit. If it deletes the label secret, every label changes and
  each device's app starts logged out. Untested so far.

## Against the goals

For rogue content in one device's frames:

| | before | now |
|---|---|---|
| reads this device's access code | yes | no |
| reads other devices' codes, cookies or storage | yes | no |
| takes over bitba.ng's origin with a worker of its own | yes | no |
| takes over its own device's origin with a worker of its own | -- | no |
| uses the session it's in while the tab is open | yes | yes (inherent) |
| connects after the tab is closed | yes, with the code | no |
| a connection it made outlives the tab | -- | yes, until connections are bound to the browser (*Limits*) |
| uses capabilities beyond its app (a CLI owner link) | yes | yes, until payloads carry a scope (*Limits*) |

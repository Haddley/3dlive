# 3D Live

Live side-by-side (full-SBS) 3D video from a Meta Quest headset's passthrough
cameras to any browser — designed for the Silk browser on a Fire TV Stick, but
any phone/laptop browser works as a viewer.

One self-contained `index.html`, no server. Sessions use the same PeerJS
room-code pattern as [haddley.github.io/games](https://haddley.github.io/games/)
(PeerJS broker for signaling, Metered TURN relay for restrictive networks,
QR + `?room=XXXX` joining).

## How to use

1. **Host the page over HTTPS** (required for both camera access and WebRTC).
   GitHub Pages is the easy path — publish this repo and open the page at
   `https://<user>.github.io/3DLive/`.
2. **On the Quest** (in the headset's browser): open the page, tap
   **"I'm the headset"**, grant camera permission. A 4-letter room code, QR
   code, and live SBS preview appear.
3. **On the Fire TV Stick** (Silk browser) — and on any other viewers: open the
   same page, enter the 4-letter code (or scan the QR on a phone). The stream
   plays fullscreen. Multiple viewers can watch the one headset; each gets its
   own WebRTC call.

On the viewer, **OK/Enter toggles the aspect mode**: `fill` (default — the
double-wide SBS frame squeezed to the screen, i.e. half-SBS, what a 3D TV
expects and what the [three.js StereoEffect demo](https://haddley.github.io/three/haddley-three-stereo.html)
looks like) vs `contain` (letterboxed at true double-wide aspect).

## Design notes

- **Topology**: the headset is the room host (`3DLIVE-XXXX` peer ID). Viewers
  open a data connection, send `{type:'join_viewer'}`, and the headset calls
  each one back with the stream. One headset, N viewers.
- **The SBS wrinkle is solved at the source**: the headset composites the
  left-eye and right-eye camera feeds side by side onto a canvas
  (1280px per eye) and streams `canvas.captureStream(30)` as a single video
  track. Viewers are dumb `<video>` players — no track juggling, guaranteed
  eye sync, and switching cameras mid-stream needs no renegotiation.
- **Cameras**: after permission, all `videoinput` devices are listed in
  left/right-eye dropdowns; devices labelled "left"/"right" are auto-picked.
  With a single camera the left feed is duplicated into both halves (mono).
  Note Quest exposes its passthrough cameras via `getUserMedia` — what the
  browser actually lists varies by Horizon OS version.
- **Resilience** (ported from herdmind in the games repo): the host peer
  reconnects to the broker on drops so late joiners can still find the room;
  viewers retry joining 3× then keep polling quietly, and auto-rejoin if the
  headset disconnects and comes back.
- **TURN quota**: the Metered credentials are shared with the games repo
  (free tier, ~50 GB/month). Same-LAN viewers connect directly and cost
  nothing; a remote viewer relaying 30 fps video will eat the quota — keep an
  eye on dashboard.metered.ca if you share codes outside the house.

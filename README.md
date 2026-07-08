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
- **Output modes** (picked on the headset page, default **mono**):
  - **Mono** — one camera, sent as a normal single-frame video (1280×720).
    Best for a regular TV; the default, and the safe choice on headsets that
    only allow one open camera at a time.
  - **Side-by-side 3D** — two cameras composited left/right onto a
    2×-wide canvas (1280px per eye), like three.js StereoEffect's split
    viewports. The second camera is opt-in via the right-eye dropdown; if only
    one is available it duplicates the left.
  - **Pass-through** — for a camera whose feed is *already* side-by-side; it's
    sent full-width as-is.
- **One composited track, whatever the mode**: the headset draws the outgoing
  frame onto a canvas and streams `canvas.captureStream(30)` as a single video
  track. Viewers are dumb `<video>` players — no track juggling, guaranteed
  eye sync, and switching mode or camera mid-stream needs no renegotiation.
  The headset also sends the current mode over the data channel so a viewer
  defaults to the right aspect (letterbox a mono feed, fill each half of SBS);
  pressing OK on the viewer overrides it.
- **Cameras**: Quest 3 (Horizon OS v77+, Browser 39+) exposes three
  `videoinput` devices: a front avatar camera plus left and right passthrough
  cameras. But the camera service generally allows **one open camera at a
  time**, so the app starts in mono (first camera duplicated into both
  halves) and never re-opens a device it already holds. Stereo is opt-in via
  the right-eye dropdown: the second camera is only accepted once frames
  actually arrive (verified with `requestVideoFrameCallback`), and a watchdog
  reverts to mono if it goes dark mid-stream. A "left feed is already
  side-by-side" pass-through mode covers cameras that deliver a combined SBS
  frame. Per-eye status and the device list are shown on the headset page.
- **Resilience** (ported from herdmind in the games repo): the host peer
  reconnects to the broker on drops so late joiners can still find the room;
  viewers retry joining 3× then keep polling quietly, and auto-rejoin if the
  headset disconnects and comes back.
- **TURN is opt-in, off by default**: relayed 30 fps video would eat the
  Metered quota (free tier ~50 GB/month, credentials shared with the games
  repo), so the default ICE config is STUN-only. Same-LAN and VPN-LAN
  connections are direct host-candidate connections and always work without
  TURN. A checkbox on the home screen (persisted per device in
  localStorage under `3dlive-turn`) adds the relay for viewers on genuinely
  different networks — enable it on both the headset and that viewer.

# WebRTC: Real-Time Video & Chat
### A Beginner-Friendly Technical Reference

---

## What Is WebRTC?

**Simple idea:** Think of WebRTC like setting up a phone call directly between two people — no phone company in the middle once the call starts. Your browser connects directly to another browser and they talk to each other.

**Technical reality:** WebRTC (Web Real-Time Communication) is an open browser standard that enables **peer-to-peer (P2P)** audio, video, and data exchange between browsers — without plugins or third-party software. Once the connection is established, media flows directly between devices.

**The one catch:** Browsers can't find each other on the internet by themselves. They need a small helper server — called a **signaling server** — to exchange a few setup messages. After that, the server steps aside and the browsers talk directly.

> **Key distinction:**
> - 🔁 **Signaling server** → only relays tiny setup messages (SDP + ICE candidates)
> - 📹 **Media (video/audio/data)** → flows directly between browsers (P2P), or via a TURN relay if firewalls block direct connection

---

## Core Concepts at a Glance

| Term | Simple Explanation | Role in WebRTC |
|---|---|---|
| `RTCPeerConnection` | The main engine — manages everything | Sets up and controls the connection |
| `MediaStream` | Your camera + mic feed | The actual audio/video you send/receive |
| `SDP` | A "compatibility checklist" for media settings | Ensures both peers agree on formats |
| `ICE Candidate` | A possible "address" to reach you | Helps find a working network path |
| `STUN` | "What's my public IP?" server | Discovers your address behind a router |
| `TURN` | A relay server (backup plan) | Forwards media when direct P2P fails |
| `ICE` | The path-finding system | Picks the best available network route |
| `Signaling` | Your custom message relay (e.g., WebSocket) | Bootstraps the connection setup |
| `DataChannel` | A P2P text/data pipe | Powers real-time chat without a server |

---

## WebRTC API Reference

WebRTC's browser API is organized into four groups. Each group is described below with its methods, properties, and events — with the mental model to understand them.

---

### Group 1 — Media Capture API

> **Mental model:** Before you can send video to someone, you need to "open" the camera. This group is the camera crew — they capture the footage before it goes anywhere.

#### `navigator.mediaDevices.getUserMedia(constraints)`

- Asks the user for permission to access camera and/or microphone
- Returns a `Promise` that resolves to a `MediaStream` object
- **`constraints`** is a config object that specifies what you want:

```js
// Simple: request both video and audio
const stream = await navigator.mediaDevices.getUserMedia({ video: true, audio: true });

// Advanced: request specific resolution
const stream = await navigator.mediaDevices.getUserMedia({
  video: { width: 1280, height: 720 },  // HD video
  audio: true
});
// ↑ If the device doesn't support 1280x720, the browser picks the closest available
```

#### `MediaStream` — the object returned by `getUserMedia`

| Member | Type | What it does |
|---|---|---|
| `.getTracks()` | Method | Returns all tracks (audio + video) as an array |
| `.getAudioTracks()` | Method | Returns only audio tracks |
| `.getVideoTracks()` | Method | Returns only video tracks |
| `.addTrack(track)` | Method | Adds a new track to the stream |
| `.removeTrack(track)` | Method | Removes a track from the stream |

#### `MediaStreamTrack` — individual audio or video unit inside a stream

| Member | Type | What it does |
|---|---|---|
| `.kind` | Property | `'audio'` or `'video'` |
| `.enabled` | Property | Set to `false` to mute/pause the track |
| `.muted` | Property | `true` if the source is currently muted |
| `.stop()` | Method | Permanently stops the track (turns off camera light) |
| `.id` | Property | Unique identifier string |

```js
// Mute your mic without stopping the stream
const audioTrack = stream.getAudioTracks()[0];
audioTrack.enabled = false;  // Mic muted — track is still "alive", just silent
audioTrack.enabled = true;   // Mic back on

// Stop camera completely (turns off the camera indicator light)
const videoTrack = stream.getVideoTracks()[0];
videoTrack.stop();
```

---

### Group 2 — Connection API

> **Mental model:** `RTCPeerConnection` is the engine of WebRTC. It's the object that manages the entire lifecycle of a peer connection — negotiation, ICE, media, security, and state. Everything flows through here.

#### `new RTCPeerConnection(config)`

Creates a new peer connection. The `config` object primarily holds ICE server info:

```js
const config = {
  iceServers: [
    { urls: 'stun:stun.l.google.com:19302' },         // Google's free STUN server
    {
      urls: 'turn:your-turn-server.com:3478',          // Your TURN server (if needed)
      username: 'user',
      credential: 'password'
    }
  ]
};

const pc = new RTCPeerConnection(config);
// ↑ This creates the engine. Nothing is connected yet.
//   The STUN/TURN servers are contacted only when ICE starts.
```

#### Connection Methods

| Method | Returns | What it does |
|---|---|---|
| `pc.addTrack(track, stream)` | `RTCRtpSender` | Adds a local media track to send |
| `pc.removeTrack(sender)` | `void` | Stops sending a specific track |
| `pc.createOffer()` | `Promise<RTCSessionDescription>` | Generates an SDP offer (caller only) |
| `pc.createAnswer()` | `Promise<RTCSessionDescription>` | Generates an SDP answer (callee only) |
| `pc.setLocalDescription(sdp)` | `Promise<void>` | Applies your own SDP (offer or answer) |
| `pc.setRemoteDescription(sdp)` | `Promise<void>` | Applies the other peer's SDP |
| `pc.addIceCandidate(candidate)` | `Promise<void>` | Adds a received ICE candidate to the pool |
| `pc.createDataChannel(label)` | `RTCDataChannel` | Creates a P2P data/chat channel |
| `pc.close()` | `void` | Closes the connection — releases resources |

#### Connection Properties

| Property | What it tells you |
|---|---|
| `pc.connectionState` | Overall state: `'new'` → `'connecting'` → `'connected'` → `'failed'` |
| `pc.iceConnectionState` | ICE path state: `'checking'` → `'connected'` → `'completed'` |
| `pc.signalingState` | SDP negotiation state: `'stable'`, `'have-local-offer'`, etc. |
| `pc.localDescription` | Your SDP (set after `setLocalDescription`) |
| `pc.remoteDescription` | The other peer's SDP (set after `setRemoteDescription`) |

#### Connection Events

| Event | When it fires | What to do in handler |
|---|---|---|
| `pc.onnegotiationneeded` | After `addTrack()` — signals that an offer should be created | Call `createOffer()` → `setLocalDescription()` → send offer |
| `pc.onicecandidate` | When a new ICE candidate is generated | Send `event.candidate` via signaling to the other peer |
| `pc.ontrack` | When remote media tracks arrive | Set `remoteVideo.srcObject = event.streams[0]` |
| `pc.ondatachannel` | When callee receives a data channel from caller | Call `setupDataChannel(event.channel)` |
| `pc.onconnectionstatechange` | When overall connection state changes | Update UI; handle `'failed'` state |
| `pc.oniceconnectionstatechange` | When ICE path state changes | Useful for debugging network issues |

```js
// --- Adding local media to the connection ---
localStream.getTracks().forEach(track => {
  pc.addTrack(track, localStream);
  //           ↑         ↑
  //        track    the parent stream it belongs to
  //  (WebRTC knows to keep related audio+video tracks together)
});

// --- Receiving remote media ---
pc.ontrack = (event) => {
  // event.streams[0] is the remote peer's MediaStream
  remoteVideo.srcObject = event.streams[0];
  // The browser handles decoding and rendering automatically
};
```

---

### Group 3 — SDP Negotiation API

> **Mental model:** Before two people call each other, they need to agree on which "language" to use. SDP (Session Description Protocol) is that negotiation. The caller proposes a "menu" of options (offer), and the callee picks what works for both (answer).

**SDP is just a plain text string.** It contains info like:
- Which video/audio codecs to use (e.g., VP8, H.264, Opus)
- Resolution, bitrate, and framerate limits
- Security fingerprints (for encryption)

#### The Offer/Answer Exchange

```
Caller                              Callee
  |                                   |
  |-- createOffer() ----------------->|   (Caller proposes settings)
  |-- setLocalDescription(offer) -    |
  |                                   |
  |                      setRemoteDescription(offer)
  |                      createAnswer()
  |                      setLocalDescription(answer)
  |<----------------------------------|   (Callee agrees/adjusts)
  |                                   |
  |-- setRemoteDescription(answer) -- |   (Caller finalizes settings)
  |                                   |
  |         ✅ SDP Negotiation Done    |
```

```js
// --- CALLER side ---
const offer = await pc.createOffer();
// offer = { type: 'offer', sdp: 'v=0\r\no=- 46117...' }  (long text blob)

await pc.setLocalDescription(offer);
// ↑ Also triggers ICE candidate gathering to begin

socket.emit('signal', { type: 'offer', sdp: pc.localDescription });


// --- CALLEE side ---
await pc.setRemoteDescription(msg.sdp);  // "I understand what the caller wants"
const answer = await pc.createAnswer();  // "Here's what I can do"
await pc.setLocalDescription(answer);
socket.emit('signal', { type: 'answer', sdp: pc.localDescription });


// --- CALLER receives answer ---
await pc.setRemoteDescription(msg.sdp);  // Negotiation complete
```

---

### Group 4 — ICE & Connectivity API

> **Mental model:** ICE is like a postal service that figures out the best route to deliver your mail. It tries your home address first (local IP), then your public address (via STUN), and if all else fails, it sends through a relay center (TURN). ICE runs these checks automatically and picks the best route that actually works.

#### How ICE Candidates Are Generated

```
Your Device (behind a router)
       │
       ├── 1. Host candidate     → Your local IP  (e.g., 192.168.1.5:50000)
       │                            Works only on the same local network
       │
       ├── 2. Server Reflexive   → STUN server reveals your public IP (e.g., 203.0.113.5:12345)
       │      (srflx)              Works for most home/office networks with NAT
       │
       └── 3. Relay candidate    → TURN server IP (e.g., 198.51.100.1:3478)
              (relay)               Fallback — used when firewalls block direct connections
```

#### ICE Candidate Flow

```js
// Step 1: Candidates are generated automatically after setLocalDescription()
pc.onicecandidate = (event) => {
  if (event.candidate) {
    // Send this candidate to the other peer via signaling
    // event.candidate looks like:
    // { candidate: 'candidate:... UDP 2122...',  sdpMid: '0',  sdpMLineIndex: 0 }
    socket.emit('signal', { type: 'candidate', candidate: event.candidate });
  }
  // event.candidate === null means gathering is complete
};

// Step 2: Add candidates received from the other peer
if (msg.type === 'candidate') {
  try {
    await pc.addIceCandidate(msg.candidate);
    // ↑ ICE now tests this candidate path (sends a STUN ping)
    //   If it gets a response, this path is viable
  } catch (e) {
    console.warn('ICE candidate failed:', e);
    // This usually happens if the candidate arrives before remote desc is set
  }
}
```

#### Monitoring ICE State

| State | Meaning |
|---|---|
| `'new'` | ICE hasn't started yet |
| `'gathering'` | Collecting candidates (contacting STUN/TURN) |
| `'checking'` | Testing candidate pairs between both peers |
| `'connected'` | A working path was found — media can flow |
| `'completed'` | All checks done, best path selected |
| `'failed'` | No working path found — connection impossible |
| `'disconnected'` | Temporary network issue — may recover |

```js
pc.oniceconnectionstatechange = () => {
  console.log('ICE state:', pc.iceConnectionState);
  //  Progression: new → checking → connected → completed
};

pc.onconnectionstatechange = () => {
  const state = pc.connectionState;
  updateStatus(state);  // Show in UI
  if (state === 'failed') {
    // Handle error — maybe show "Call failed, check your network"
  }
};
```

---

### Group 5 — DataChannel API

> **Mental model:** Once the P2P tunnel is built, you can send any data through it — not just video. `RTCDataChannel` is like a direct messaging pipe between two browsers. No server involved once it's open.

#### Creating and Receiving a DataChannel

```js
// --- CALLER creates the channel ---
const dataChannel = pc.createDataChannel('chat', {
  ordered: true       // Guarantee delivery order (like TCP)
  // ordered: false   // Faster but unordered (like UDP) — good for games
});

// --- CALLEE receives the channel via event ---
pc.ondatachannel = (event) => {
  const dataChannel = event.channel;  // Same channel, other side
  setupDataChannel(dataChannel);
};
```

#### DataChannel Methods and Events

| Member | Type | What it does |
|---|---|---|
| `.send(data)` | Method | Sends a string or binary data |
| `.close()` | Method | Closes the channel |
| `.readyState` | Property | `'connecting'` → `'open'` → `'closing'` → `'closed'` |
| `.bufferedAmount` | Property | Bytes waiting to be sent (useful for backpressure) |
| `.onopen` | Event | Channel is ready — start sending |
| `.onmessage` | Event | A message was received (`event.data` has the content) |
| `.onclose` | Event | Channel was closed |
| `.onerror` | Event | An error occurred |

```js
function setupDataChannel(dc) {
  dc.onopen = () => {
    console.log('Chat ready!');  // Now safe to call dc.send()
  };

  dc.onmessage = (event) => {
    // event.data is the string/blob sent by the other peer
    chatDiv.innerHTML += `<p><b>Peer:</b> ${event.data}</p>`;
  };

  dc.onclose = () => console.log('Chat closed');
}

// Sending a message
function sendMessage(text) {
  if (dc.readyState === 'open') {  // Always check before sending
    dc.send(text);
  }
}
```

---

## The Full Connection Flow

> **Mental model:** Think of this like two strangers setting up a walkie-talkie call. First they call a mutual friend (signaling server) to exchange frequencies (SDP) and location info (ICE candidates). Once synced, they talk directly — the mutual friend hangs up.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        WebRTC Connection Setup                              │
├──────────────────────┬──────────────────────┬───────────────────────────────┤
│    Peer A (Caller)   │  Signaling Server    │      Peer B (Callee)          │
├──────────────────────┼──────────────────────┼───────────────────────────────┤
│                      │                      │                               │
│  getUserMedia()      │                      │                               │
│  ┌──────────────┐    │                      │                               │
│  │ localStream  │    │                      │                               │
│  └──────────────┘    │                      │                               │
│         │            │                      │                               │
│  addTrack(track)     │                      │                               │
│  createOffer()       │                      │                               │
│  setLocalDesc(offer) │                      │                               │
│         │            │                      │                               │
│         └──── emit 'offer' SDP ────────────►│                               │
│                      │                      │  receive 'offer'              │
│                      │                      │  getUserMedia()               │
│                      │                      │  setRemoteDesc(offer)         │
│                      │                      │  createAnswer()               │
│                      │                      │  setLocalDesc(answer)         │
│                      │                      │        │                      │
│◄──────────────────── emit 'answer' SDP ─────┘        │                     │
│  setRemoteDesc(ans)  │                      │                               │
│                      │                      │                               │
│  ┌──── ICE Gathering Starts (both sides) ───────────────────────┐          │
│  │                   │                      │                   │          │
│  │  onicecandidate ──┤──emit 'candidate'───►├── addIceCandidate │          │
│  │  addIceCandidate ◄├──emit 'candidate'────┤── onicecandidate  │          │
│  │                   │  (multiple rounds)   │                   │          │
│  └───────────────────────────────────────────────────────────────┘          │
│                      │                      │                               │
│  ╔═══════════════════════════════════════════════════════╗                  │
│  ║  ICE Connectivity Checks (automatic, no code needed) ║                  │
│  ║  Peers ping each other's candidates → best path wins ║                  │
│  ╚═══════════════════════════════════════════════════════╝                  │
│                      │                      │                               │
│  connectionState: 'connected' ◄────────────► connectionState: 'connected'  │
│                      │                      │                               │
│  ◄═══════════════════ Media flows P2P ══════════════════►                  │
│       (video/audio via MediaStream tracks, chat via DataChannel)            │
│                      │                      │                               │
│            Server is no longer involved ✅  │                               │
└──────────────────────┴──────────────────────┴───────────────────────────────┘
```

---

## Step-by-Step Implementation

### Project Structure

```
project/
├── server.js       ← Node.js signaling server (Socket.IO + Express)
└── client.html     ← Browser client (WebRTC + Socket.IO)
```

**Install dependencies:**
```bash
npm install express socket.io
```
**Run server:**
```bash
node server.js
```
**Test:** Open `http://localhost:3000/client.html` in two browser tabs.

---

### server.js — Signaling Server

> The server's only job is to relay setup messages between the two browsers. It doesn't understand WebRTC — it just broadcasts whatever it receives.

```js
// ── 1. Import modules ────────────────────────────────────────────────────────
const express = require('express');
const http = require('http');
const { Server } = require('socket.io');

// ── 2. Create HTTP server and attach Socket.IO ────────────────────────────────
const app = express();
const server = http.createServer(app);
const io = new Server(server);

// ── 3. Serve client.html statically ──────────────────────────────────────────
app.use(express.static(__dirname));
// Now client.html is accessible at http://localhost:3000/client.html

// ── 4. Handle WebSocket connections ──────────────────────────────────────────
io.on('connection', (socket) => {
  console.log('Client connected');

  // When any peer emits 'signal', forward it to everyone else
  socket.on('signal', (msg) => {
    socket.broadcast.emit('signal', msg);
    // ↑ "broadcast" = send to all sockets EXCEPT the one who sent it
    //   This is fine for 1-on-1 calls. For rooms, use socket.to(room).emit(...)
  });

  socket.on('disconnect', () => console.log('Client disconnected'));
});

// ── 5. Start server ───────────────────────────────────────────────────────────
server.listen(3000, () => console.log('Server running → http://localhost:3000'));
```

---

### client.html — Full Browser Code

#### HTML Structure

```html
<!DOCTYPE html>
<html>
<head>
  <title>WebRTC Demo</title>
  <style>
    video { width: 45%; margin: 10px; background: #000; }
    #chat { height: 200px; overflow-y: scroll; border: 1px solid #ccc; padding: 10px; }
    #status { font-weight: bold; color: #444; margin: 8px 0; }
  </style>
</head>
<body>

  <h2>Local (You)</h2>
  <video id="localVideo" autoplay muted playsinline></video>
  <!-- ↑ muted: prevents echo of your own audio -->

  <h2>Remote (Peer)</h2>
  <video id="remoteVideo" autoplay playsinline></video>

  <div id="status">Status: Idle</div>
  <br>
  <button id="start">Start Camera</button>
  <button id="call">Call</button>
  <button id="hangup">Hang Up</button>

  <h3>Chat</h3>
  <div id="chat"></div>
  <input id="msg" placeholder="Type a message..." />
  <button id="send">Send</button>

  <!-- Socket.IO client library (loads from CDN) -->
  <script src="https://cdn.socket.io/4.8.3/socket.io.min.js"></script>
  <script>
    /* all JavaScript goes here */
  </script>
</body>
</html>
```

#### JavaScript — Step by Step

**Step 1 — Global variables and setup**

```js
// ── Global State ──────────────────────────────────────────────────────────────
let pc = null;           // RTCPeerConnection instance (null until call starts)
let localStream = null;  // Your camera/mic MediaStream
let dataChannel = null;  // RTCDataChannel for chat messages

// ── DOM References ────────────────────────────────────────────────────────────
const localVideo  = document.getElementById('localVideo');
const remoteVideo = document.getElementById('remoteVideo');
const statusDiv   = document.getElementById('status');
const chatDiv     = document.getElementById('chat');
const msgInput    = document.getElementById('msg');

function updateStatus(text) {
  statusDiv.textContent = `Status: ${text}`;
}

// ── ICE Server Config ─────────────────────────────────────────────────────────
const rtcConfig = {
  iceServers: [
    { urls: 'stun:stun.l.google.com:19302' }
    // Add TURN here if users are on strict corporate/firewall networks:
    // { urls: 'turn:your-server.com:3478', username: 'u', credential: 'p' }
  ]
};

// ── Connect to Signaling Server ───────────────────────────────────────────────
const socket = io('http://localhost:3000');
socket.on('connect',    () => console.log('Signaling connected'));
socket.on('disconnect', () => console.log('Signaling disconnected'));
```

**Step 2 — Capture local media**

```js
// ── Start Camera Button ───────────────────────────────────────────────────────
document.getElementById('start').onclick = async () => {
  try {
    localStream = await navigator.mediaDevices.getUserMedia({ video: true, audio: true });
    localVideo.srcObject = localStream;
    // ↑ The video element displays your local feed (muted, so no echo)
    updateStatus('Camera ready');
  } catch (err) {
    // Common errors: NotAllowedError (denied), NotFoundError (no camera)
    console.error('getUserMedia failed:', err);
    updateStatus('Camera access denied');
  }
};
```

**Step 3 — Create peer connection**

```js
function createPeerConnection(isCaller) {
  pc = new RTCPeerConnection(rtcConfig);

  // ── Add local tracks so they get sent to the remote peer ─────────────────
  localStream.getTracks().forEach(track => pc.addTrack(track, localStream));
  // After this, onnegotiationneeded will fire automatically

  // ── Receive remote tracks (their video/audio arrives here) ───────────────
  pc.ontrack = (event) => {
    remoteVideo.srcObject = event.streams[0];
    updateStatus('Connected — receiving remote video');
  };

  // ── Send ICE candidates to the other peer via signaling ──────────────────
  pc.onicecandidate = (event) => {
    if (event.candidate) {
      socket.emit('signal', { type: 'candidate', candidate: event.candidate });
    }
    // null candidate = ICE gathering is complete (nothing to send)
  };

  // ── Monitor overall connection state ──────────────────────────────────────
  pc.onconnectionstatechange = () => {
    updateStatus(pc.connectionState);
    if (pc.connectionState === 'failed') {
      console.warn('Connection failed — check network or add a TURN server');
    }
  };

  // ── Monitor ICE state (useful for debugging) ──────────────────────────────
  pc.oniceconnectionstatechange = () => {
    console.log('ICE:', pc.iceConnectionState);
    // checking → connected → completed is the happy path
  };

  // ── Caller creates the DataChannel; callee receives it via event ──────────
  if (isCaller) {
    dataChannel = pc.createDataChannel('chat');
    setupDataChannel(dataChannel);
  } else {
    pc.ondatachannel = (event) => {
      dataChannel = event.channel;
      setupDataChannel(dataChannel);
    };
  }
}
```

**Step 4 — SDP Offer/Answer negotiation**

```js
// ── Call Button (Caller Side) ─────────────────────────────────────────────────
document.getElementById('call').onclick = async () => {
  if (!localStream) { alert('Start camera first'); return; }

  createPeerConnection(true);  // isCaller = true

  // onnegotiationneeded fires automatically after addTrack()
  pc.onnegotiationneeded = async () => {
    try {
      const offer = await pc.createOffer();
      await pc.setLocalDescription(offer);
      // ↑ Also starts ICE candidate gathering in the background

      socket.emit('signal', { type: 'offer', sdp: pc.localDescription });
      updateStatus('Offer sent — waiting for answer...');
    } catch (err) {
      console.error('Offer creation failed:', err);
    }
  };
};

// ── Handle all incoming signaling messages ────────────────────────────────────
socket.on('signal', async (msg) => {

  // ── Callee: received an offer → create peer connection and answer ──────
  if (msg.type === 'offer') {
    if (!localStream) {
      localStream = await navigator.mediaDevices.getUserMedia({ video: true, audio: true });
      localVideo.srcObject = localStream;
    }
    createPeerConnection(false);  // isCaller = false

    await pc.setRemoteDescription(msg.sdp);
    // ↑ "I understand what the caller wants to send me"

    const answer = await pc.createAnswer();
    await pc.setLocalDescription(answer);
    socket.emit('signal', { type: 'answer', sdp: pc.localDescription });
    updateStatus('Answer sent — connecting...');
  }

  // ── Caller: received an answer → finalize negotiation ─────────────────
  else if (msg.type === 'answer') {
    await pc.setRemoteDescription(msg.sdp);
    // ↑ SDP negotiation is now complete. ICE takes over.
  }

  // ── Both sides: received an ICE candidate from the other peer ─────────
  else if (msg.type === 'candidate') {
    try {
      await pc.addIceCandidate(msg.candidate);
      // ↑ ICE tests this path with a ping. If it works, it becomes a candidate route.
    } catch (err) {
      console.warn('Failed to add ICE candidate:', err);
      // Can happen if the candidate arrives before remote description is set
    }
  }

});
```

**Step 5 — DataChannel (chat)**

```js
function setupDataChannel(dc) {
  dc.onopen = () => {
    updateStatus('Chat ready ✓');
    console.log('DataChannel open — P2P chat is live');
  };

  dc.onmessage = (event) => {
    // event.data = the string sent by the other peer
    chatDiv.innerHTML += `<p><b>Peer:</b> ${event.data}</p>`;
    chatDiv.scrollTop = chatDiv.scrollHeight;  // Auto-scroll to latest message
  };

  dc.onclose = () => console.log('DataChannel closed');
}

// ── Send Button ───────────────────────────────────────────────────────────────
document.getElementById('send').onclick = () => {
  const text = msgInput.value.trim();
  if (!text) return;

  if (dataChannel && dataChannel.readyState === 'open') {
    dataChannel.send(text);  // Sent directly P2P — no server involved
    chatDiv.innerHTML += `<p><b>Me:</b> ${text}</p>`;
    chatDiv.scrollTop = chatDiv.scrollHeight;
    msgInput.value = '';
  } else {
    alert('Chat not ready yet');
  }
};
```

**Step 6 — Hang Up**

```js
document.getElementById('hangup').onclick = () => {
  if (dataChannel) dataChannel.close();  // Close the chat pipe
  if (pc) pc.close();                    // Close the peer connection (stops all media)
  pc = null;
  dataChannel = null;
  remoteVideo.srcObject = null;          // Clear remote video
  updateStatus('Call ended');
};
```

---

### Step 7 — The Full Execution Order (Putting It All Together)

> **Mental model:** The steps above defined all the ingredients. This is the recipe — showing exactly what runs when, in what order, and which branch (caller vs callee) each path takes. If you're confused about "who calls what and when", this is your answer.

```js
// ═══════════════════════════════════════════════════════════════════════════════
//  EXECUTION ORDER — client.html
//  Read this top to bottom. Each comment marks WHEN that code runs.
// ═══════════════════════════════════════════════════════════════════════════════

// ── ON PAGE LOAD (runs immediately) ──────────────────────────────────────────
//    Variables declared, DOM refs assigned, socket connected to signaling server.
//    Nothing WebRTC-related happens yet. Both peers are in this state at start.

const socket = io('http://localhost:3000');
let pc = null, localStream = null, dataChannel = null;


// ── WHEN USER CLICKS "Start Camera" ──────────────────────────────────────────
//    Same on both Peer A and Peer B. Independent of each other.

document.getElementById('start').onclick = async () => {
  localStream = await navigator.mediaDevices.getUserMedia({ video: true, audio: true });
  localVideo.srcObject = localStream;
  // localStream is now ready. No WebRTC yet — just camera/mic captured.
};


// ══════════════════════════════════════════════════════════════════════════════
//  FROM HERE, THE TWO PEERS DIVERGE:
//     Peer A clicks "Call"  → becomes the CALLER
//     Peer B does nothing   → becomes the CALLEE (reacts to incoming offer)
// ══════════════════════════════════════════════════════════════════════════════


// ── [CALLER ONLY] WHEN USER CLICKS "Call" ────────────────────────────────────
//    Only Peer A runs this block.

document.getElementById('call').onclick = async () => {
  if (!localStream) { alert('Start camera first'); return; }

  createPeerConnection(true);               // ← isCaller = true
  //  Inside createPeerConnection(true):
  //    ✅ new RTCPeerConnection(config)     — engine created
  //    ✅ addTrack() for each local track   — triggers onnegotiationneeded
  //    ✅ pc.ontrack wired up               — ready to receive remote video
  //    ✅ pc.onicecandidate wired up        — ready to send candidates
  //    ✅ createDataChannel('chat')         — caller creates the chat pipe
  //    ✅ pc.onconnectionstatechange wired  — UI state updates
  //
  //  Immediately after addTrack() → onnegotiationneeded fires automatically ↓

  pc.onnegotiationneeded = async () => {
    const offer = await pc.createOffer();       // Step A: build SDP offer
    await pc.setLocalDescription(offer);        // Step B: apply it locally
    //                                          //         (ICE gathering starts here)
    socket.emit('signal', { type: 'offer', sdp: pc.localDescription });
    //                                          // Step C: send offer to Peer B via server
    updateStatus('Waiting for answer...');
  };
  // ↑ Peer A is now waiting. Peer B will react below.
};


// ── SIGNALING HANDLER — runs on BOTH peers whenever server sends a message ───
//    This is the central router. Every WebRTC message flows through here.
//    The if/else chain decides what to do based on message type.

socket.on('signal', async (msg) => {

  // ┌─────────────────────────────────────────────────────────────────────────┐
  // │  msg.type === 'offer'   →  only CALLEE (Peer B) reaches this branch     │
  // └─────────────────────────────────────────────────────────────────────────┘
  if (msg.type === 'offer') {

    // Peer B may not have clicked "Start Camera" yet — auto-start if needed
    if (!localStream) {
      localStream = await navigator.mediaDevices.getUserMedia({ video: true, audio: true });
      localVideo.srcObject = localStream;
    }

    createPeerConnection(false);              // ← isCaller = false
    //  Inside createPeerConnection(false):
    //    ✅ new RTCPeerConnection(config)     — engine created
    //    ✅ addTrack() for each local track   — Peer B's video ready to send
    //    ✅ pc.ontrack wired up               — ready to receive Peer A's video
    //    ✅ pc.onicecandidate wired up        — ready to send candidates
    //    ✅ pc.ondatachannel wired up         — will receive chat channel from Peer A
    //    ✅ pc.onconnectionstatechange wired  — UI state updates
    //    ✗  does NOT call createDataChannel() — that's the caller's job

    await pc.setRemoteDescription(msg.sdp);   // Step D: "I understand Peer A's offer"
    const answer = await pc.createAnswer();   // Step E: build SDP answer
    await pc.setLocalDescription(answer);     // Step F: apply it locally
    //                                        //         (ICE gathering starts here for Peer B)
    socket.emit('signal', { type: 'answer', sdp: pc.localDescription });
    //                                        // Step G: send answer back to Peer A
    updateStatus('Answer sent — connecting...');
  }

  // ┌─────────────────────────────────────────────────────────────────────────┐
  // │  msg.type === 'answer'  →  only CALLER (Peer A) reaches this branch     │
  // └─────────────────────────────────────────────────────────────────────────┘
  else if (msg.type === 'answer') {
    await pc.setRemoteDescription(msg.sdp);   // Step H: "I understand Peer B's answer"
    // ✅ SDP negotiation is now COMPLETE on both sides.
    // ICE is already running in background (started at Steps B and F).
    // The two peers are now exchanging ICE candidates via the next branch ↓
  }

  // ┌─────────────────────────────────────────────────────────────────────────┐
  // │  msg.type === 'candidate'  →  BOTH peers hit this branch (many times)   │
  // │  ICE candidates trickle in asynchronously — could arrive in any order   │
  // └─────────────────────────────────────────────────────────────────────────┘
  else if (msg.type === 'candidate') {
    try {
      await pc.addIceCandidate(msg.candidate);
      // ICE now tests this path (sends a STUN ping to the candidate address).
      // This branch fires multiple times — once per candidate from the other peer.
      // Typical order: host (local IP) → srflx (STUN/public IP) → relay (TURN)
    } catch (err) {
      console.warn('ICE candidate rejected:', err);
      // Safe to ignore — happens if candidates arrive before remote desc is set
    }
  }

});


// ── ICE CANDIDATE GENERATION — fires on BOTH peers (after setLocalDescription)
//    This runs in the background, triggered by the WebRTC engine automatically.
//    Not a button click — it's an event that fires once per candidate found.

pc.onicecandidate = (event) => {
  if (event.candidate) {
    socket.emit('signal', { type: 'candidate', candidate: event.candidate });
    // Sends this candidate to the other peer, who adds it via addIceCandidate()
  }
  // When event.candidate === null, gathering is complete (all paths explored)
};


// ── CONNECTION ESTABLISHED — fires on both peers when ICE succeeds ─────────
//    At this point: SDP agreed ✅, ICE path found ✅, DTLS handshake done ✅

pc.onconnectionstatechange = () => {
  if (pc.connectionState === 'connected') {
    updateStatus('🟢 Connected');
    // Media is now flowing P2P between Peer A and Peer B.
    // The signaling server is no longer needed for this call.
  }
  if (pc.connectionState === 'failed') {
    updateStatus('🔴 Connection failed');
  }
};

// Remote video arrives here — fires shortly after 'connected'
pc.ontrack = (event) => {
  remoteVideo.srcObject = event.streams[0];
};

// DataChannel arrives on callee side — fires after connection is established
pc.ondatachannel = (event) => {
  dataChannel = event.channel;
  setupDataChannel(dataChannel);   // wire up onopen / onmessage / onclose
};


// ── WHEN USER CLICKS "Hang Up" — both peers can trigger this ─────────────────

document.getElementById('hangup').onclick = () => {
  if (dataChannel) dataChannel.close();   // 1. close chat pipe
  if (pc) pc.close();                     // 2. close connection (stops all media)
  pc = null;                              // 3. clear references
  dataChannel = null;
  remoteVideo.srcObject = null;           // 4. clear remote video element
  updateStatus('Call ended');
};


// ═══════════════════════════════════════════════════════════════════════════════
//  FULL EXECUTION TIMELINE SUMMARY
//
//  Both peers:   page load → getUserMedia → [waiting]
//  Caller only:  click "Call" → createPeerConnection(true)
//                             → onnegotiationneeded → createOffer
//                             → setLocalDescription → emit 'offer'
//  Callee only:  receive 'offer' → createPeerConnection(false)
//                                → setRemoteDescription
//                                → createAnswer → setLocalDescription
//                                → emit 'answer'
//  Caller only:  receive 'answer' → setRemoteDescription
//  Both peers:   onicecandidate fires (multiple) → emit 'candidate'
//  Both peers:   receive 'candidate' → addIceCandidate (multiple rounds)
//  Both peers:   connectionState → 'connected' ✅
//  Both peers:   ontrack fires → remote video plays
//  Caller only:  DataChannel open → chat ready
//  Callee only:  ondatachannel fires → chat ready
// ═══════════════════════════════════════════════════════════════════════════════
```

---

## Quick Reference — Event & Method Cheat Sheet

### Caller vs Callee — Who Does What?

```
                    Caller                      Callee
                ─────────────               ─────────────
createPC()          ✅                           ✅
createOffer()       ✅ (only caller)             ✗
createAnswer()      ✗                           ✅ (only callee)
setLocalDesc()      ✅ (with offer)             ✅ (with answer)
setRemoteDesc()     ✅ (with answer)            ✅ (with offer)
createDataChannel() ✅ (only caller)             ✗
ondatachannel       ✗                           ✅ (only callee)
addIceCandidate()   ✅ (both)                   ✅ (both)
onicecandidate      ✅ (both)                   ✅ (both)
```

### The 5 Things You Must Wire Up in `createPeerConnection()`

| Event/Method | What it handles |
|---|---|
| `pc.ontrack` | Receiving remote video/audio |
| `pc.onicecandidate` | Sending ICE candidates to signaling server |
| `pc.ondatachannel` | Receiving the chat channel (callee only) |
| `pc.onconnectionstatechange` | Updating UI with connection status |
| `localStream.getTracks().forEach(t => pc.addTrack(t, stream))` | Sending your own media |

---

## Common Errors & Fixes

| Error | Cause | Fix |
|---|---|---|
| `NotAllowedError` from `getUserMedia` | Camera/mic permission denied | Check browser permissions; use HTTPS in production |
| ICE state stuck at `'checking'` | Firewall blocking direct connection | Add a TURN server to `rtcConfig` |
| `addIceCandidate` throws | Candidate received before remote description set | Queue candidates and add them after `setRemoteDescription` |
| Remote video blank | `ontrack` not set up before `addTrack` | Set up all event handlers before adding tracks |
| `DataChannel not ready` | Tried to send before `'open'` state | Always check `dc.readyState === 'open'` before calling `send()` |

---

## Testing Locally

```
Terminal:          node server.js
Browser Tab 1:     http://localhost:3000/client.html  → click "Start Camera" → click "Call"
Browser Tab 2:     http://localhost:3000/client.html  → click "Start Camera"
                   (Tab 2 auto-receives the offer and answers)
```

> **Tip:** For camera access on `localhost`, most browsers allow HTTP. For any other domain, you must use HTTPS — `getUserMedia()` is restricted to secure contexts.

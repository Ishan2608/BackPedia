# WebSocket: Real-Time Chat
### A Beginner-Friendly Technical Reference — 1-on-1, Group Chats & File Sharing

---

## What Is WebSocket?

**Simple idea:** Think of regular HTTP like sending letters by post — you write a request, mail it, wait for a reply. WebSocket is like a phone call — once connected, both sides can talk whenever they want, without dialing again.

**Technical reality:** WebSocket is a communication protocol that establishes a **persistent, full-duplex TCP connection** between a client (browser) and a server. Once the connection is open, either side can push messages to the other at any time — no repeated HTTP requests needed.

**Why not just use HTTP?** HTTP is request-response: the client asks, the server answers, then the connection closes. For chat, this means constantly polling ("any new messages?") which is slow and wasteful. WebSocket keeps the channel open permanently.

> **Key distinctions:**
> - 🔁 **HTTP** → client asks, server answers, connection closes. One-way at a time.
> - 📡 **WebSocket** → one connection, both sides push freely, connection stays open.
> - 🏛️ **All messages route through the server** → unlike WebRTC, there is no P2P. The server is always in the middle, which makes group logic easy to manage.

---

## Core Concepts at a Glance

| Term | Simple Explanation | Role in WebSocket Chat |
|---|---|---|
| `WebSocket` | A persistent two-way connection | The pipe messages travel through |
| `Socket.IO` | A library built on WebSocket | Adds rooms, events, reconnection, and more |
| `socket` | One user's connection to the server | Identifies a single connected client |
| `socket.id` | Auto-assigned unique ID per connection | Used to target a specific user |
| `room` | A named group of sockets | Powers group chats |
| `event` | A named message type (like `'chat'`, `'join'`) | How client and server communicate |
| `emit` | Sending an event with a payload | How you send any message |
| `on` | Listening for an event | How you receive any message |
| `broadcast` | Send to everyone except the sender | Used for "user joined" notifications |
| `namespace` | A partitioned section of the server | Isolate different apps on one server |

---

## Architecture Overview

> **Mental model:** The server is a post office. Every message goes through it. For 1-on-1 chat, the post office reads the address label and delivers to one person. For group chat, it reads the room name on the envelope and delivers to everyone in that room.

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                         WebSocket Chat Architecture                          │
├──────────────────────────────────┬───────────────────────────────────────────┤
│           CLIENT SIDE            │              SERVER SIDE                  │
├──────────────────────────────────┼───────────────────────────────────────────┤
│                                  │                                           │
│  Browser A ──────────────────────┼──► socket.id: "abc123"  ─┐               │
│  (User: Alice)                   │                           │               │
│                                  │                           ▼               │
│  Browser B ──────────────────────┼──► socket.id: "def456"  ─┤─► Room: ""    │
│  (User: Bob)                     │                           │   (default)   │
│                                  │                           │               │
│  Browser C ──────────────────────┼──► socket.id: "ghi789"  ─┤               │
│  (User: Carol)                   │                           │               │
│                                  │                     ┌─────┘               │
│                                  │                     │                     │
│                                  │              Room: "group-xyz"            │
│                                  │              (Alice + Carol are members)  │
│                                  │                                           │
│  Each browser runs               │  Server holds:                           │
│  socket.io client lib            │  - Map of socket.id → username           │
│  and maintains one               │  - Map of roomId → { name, members[] }   │
│  persistent WebSocket            │  - Logic to route messages correctly      │
│  connection to the server        │                                           │
└──────────────────────────────────┴───────────────────────────────────────────┘
```

---

## WebSocket / Socket.IO API Reference

The API is split into five groups. The client-side and server-side APIs mirror each other intentionally — both use `emit` and `on` with the same event names.

---

### Group 1 — Connection API

> **Mental model:** Before any message can be sent, a connection must be established. This group covers how to open, monitor, and close that connection. Think of it as picking up the phone — everything else happens after the line is open.

#### Client Side — `io(url, options)`

- Connects the browser to the Socket.IO server
- Returns a `socket` object representing this client's connection
- Automatically reconnects if the connection drops

```js
// Basic connection
const socket = io('http://localhost:3000');

// With options
const socket = io('http://localhost:3000', {
  reconnection: true,          // Auto-reconnect on drop (default: true)
  reconnectionAttempts: 5,     // Try 5 times before giving up
  reconnectionDelay: 1000,     // Wait 1 second between retries
  auth: { token: 'user-jwt' }  // Send auth data on connect (useful for login)
});
// ↑ The socket object is your handle to the connection.
//   Store it globally — you'll use it for all sends and receives.
```

#### Client-Side Connection Events

| Event | When it fires | What to do |
|---|---|---|
| `'connect'` | Connection established successfully | Enable UI, show "Online" status |
| `'disconnect'` | Connection lost (network drop, server restart) | Show "Reconnecting..." |
| `'connect_error'` | Connection attempt failed | Show error message |
| `'reconnect'` | Successfully reconnected after a drop | Refresh message history |

```js
socket.on('connect', () => {
  console.log('Connected! My ID:', socket.id);
  // socket.id is a unique string like "AbCd1234" assigned by the server
  // It changes every time you reconnect — don't use it as a permanent user ID
});

socket.on('disconnect', (reason) => {
  // reason: 'io server disconnect' / 'transport close' / 'ping timeout'
  updateStatus('🔴 Disconnected — ' + reason);
});

socket.on('reconnect', (attemptNumber) => {
  updateStatus('🟢 Reconnected after ' + attemptNumber + ' attempt(s)');
});
```

#### Server Side — Connection Handling

| Method / Event | What it does |
|---|---|
| `io.on('connection', callback)` | Fires when any new client connects |
| `socket.id` | This client's unique auto-assigned ID |
| `socket.on('disconnect', callback)` | Fires when this client disconnects |
| `socket.handshake.auth` | Auth data sent by the client on connect |
| `socket.handshake.address` | IP address of the client |

```js
io.on('connection', (socket) => {
  // This block runs ONCE per connecting client.
  // 'socket' here represents THAT ONE CLIENT's connection.
  // Every client gets their own separate 'socket' object.

  console.log('User connected:', socket.id);

  // Read auth data sent by the client
  const token = socket.handshake.auth.token;
  // ↑ Validate the token here to authenticate the user

  socket.on('disconnect', (reason) => {
    console.log('User disconnected:', socket.id, '—', reason);
    // Clean up: remove from user map, notify their rooms, etc.
  });
});
```

---

### Group 2 — Messaging API

> **Mental model:** `emit` is "send a letter". `on` is "open your mailbox". The event name is the label on the envelope — both sides must use the same label for the message to be received. You can put anything you want inside the envelope (the payload).

#### Sending Messages — `emit(event, payload)`

| Call | Who receives it |
|---|---|
| `socket.emit('event', data)` | **Client:** sends to server. **Server:** sends to this one socket only |
| `io.emit('event', data)` | Server → sends to **every connected client** |
| `socket.broadcast.emit('event', data)` | Server → sends to **everyone except the sender** |
| `io.to(roomId).emit('event', data)` | Server → sends to **everyone in a specific room** |
| `socket.to(socketId).emit('event', data)` | Server → sends to **one specific client** by their socket ID |
| `socket.to(roomId).emit('event', data)` | Server → sends to everyone in room **except the sender** |

```js
// ── CLIENT: Sending a chat message to the server ──────────────────────────────
socket.emit('chat:message', {
  text: 'Hello!',
  to: 'def456',       // for 1-on-1: target socket ID
  roomId: null        // for group: room ID, or null for 1-on-1
});
// ↑ 'chat:message' is the event name — a string you define.
//   Using namespaced names like 'chat:message' keeps events organized.
//   The second argument is the payload — any JSON-serializable value.

// ── SERVER: Routing the message to the right recipient ───────────────────────
socket.on('chat:message', (msg) => {
  if (msg.roomId) {
    // Group chat: deliver to everyone in the room
    io.to(msg.roomId).emit('chat:message', msg);
  } else {
    // 1-on-1: deliver to one specific socket
    socket.to(msg.to).emit('chat:message', msg);
    socket.emit('chat:message', msg);  // also echo back to sender
  }
});
```

#### Receiving Messages — `on(event, callback)`

```js
// ── CLIENT: Listening for incoming messages ───────────────────────────────────
socket.on('chat:message', (msg) => {
  // msg is exactly what the server emitted as the payload
  renderMessage(msg);
});

// ── SERVER: Listening for any event from a specific client ────────────────────
socket.on('chat:message', (msg) => { /* handle */ });
socket.on('user:typing', (data) => { /* handle */ });
// ↑ Each socket.on() inside io.on('connection') is scoped to THAT client.
//   You can listen for as many events as you need.
```

#### Acknowledgements (Confirmed Delivery)

> **Mental model:** Normal `emit` is "fire and forget" — you send and move on. Acknowledgements are like sending a message and waiting for the recipient to say "got it" before continuing.

```js
// ── CLIENT: Emit with acknowledgement callback ────────────────────────────────
socket.emit('chat:message', { text: 'Hello' }, (response) => {
  // This callback fires when the SERVER calls the ack function
  if (response.status === 'ok') {
    markMessageDelivered();  // e.g., show a ✓ tick
  }
});

// ── SERVER: Responding to the acknowledgement ─────────────────────────────────
socket.on('chat:message', (msg, ack) => {
  // ack is a function — call it to confirm delivery
  saveMessage(msg);
  ack({ status: 'ok', messageId: generateId() });
  // ↑ This triggers the client's callback above
});
```

---

### Group 3 — Rooms API

> **Mental model:** Rooms are like WhatsApp groups — the server manages a list of who's in each group. When you send to a room, the server delivers to everyone on that list. Joining and leaving a room is instant and costs nothing — it's just an entry in a Map on the server.

#### Room Methods (Server Side Only)

| Method | What it does |
|---|---|
| `socket.join(roomId)` | Add this socket to a room |
| `socket.leave(roomId)` | Remove this socket from a room |
| `io.to(roomId).emit(event, data)` | Send to everyone in a room (including sender) |
| `socket.to(roomId).emit(event, data)` | Send to everyone in a room (excluding sender) |
| `io.in(roomId).fetchSockets()` | Get array of all socket objects in a room |
| `socket.rooms` | Set of all room IDs this socket is currently in |

```js
// ── Joining a room ────────────────────────────────────────────────────────────
socket.on('room:join', (roomId) => {
  socket.join(roomId);
  // ↑ This socket is now in the room. Future io.to(roomId).emit() reach it.

  // Tell everyone else in the room that a new user joined
  socket.to(roomId).emit('room:user-joined', {
    userId: socket.id,
    username: users.get(socket.id)
  });
  // ↑ socket.to() excludes the sender — they already know they joined
});

// ── Leaving a room ────────────────────────────────────────────────────────────
socket.on('room:leave', (roomId) => {
  socket.leave(roomId);
  socket.to(roomId).emit('room:user-left', { userId: socket.id });
});

// ── Sending to everyone in a room ─────────────────────────────────────────────
socket.on('chat:message', (msg) => {
  io.to(msg.roomId).emit('chat:message', msg);
  // ↑ io.to() includes the sender — good for group messages
  //   so the sender sees their own message confirmed from the server
});
```

---

### Group 4 — Server Broadcast Targets

> **Mental model:** This is a targeting system. When you fire a message, you need to tell Socket.IO who should receive it. Think of it like a megaphone with different range settings — whisper to one person, speak to a room, or shout to everyone.

```
WHO RECEIVES THE MESSAGE — Quick Visual Reference

  socket.emit(...)                 →  Just this one client (or client→server)
  socket.to(socketId).emit(...)    →  One specific other client
  socket.broadcast.emit(...)       →  Everyone EXCEPT this socket
  socket.to(roomId).emit(...)      →  Everyone in room EXCEPT this socket
  io.to(roomId).emit(...)          →  Everyone in room INCLUDING this socket
  io.emit(...)                     →  Every connected client on the server
```

```js
// Real-world example: User sends a group message
socket.on('chat:message', (msg) => {
  // Attach server-side metadata before broadcasting
  const enrichedMsg = {
    ...msg,
    senderId: socket.id,
    senderName: users.get(socket.id),
    timestamp: Date.now()
    // ↑ Always add timestamp on the server — don't trust client timestamps
  };

  io.to(msg.roomId).emit('chat:message', enrichedMsg);
  //  ↑ io.to() includes the sender, so they see their own message
  //    with the server's timestamp — ensures consistency across all clients
});
```

---

### Group 5 — File Transfer API

> **Mental model:** A text message is like a sticky note — small, send instantly. A file is like a package — you need different handling based on size. Small files (under ~1MB) can go as one chunk. Large files must be split into pieces (chunks), sent one at a time, and reassembled on the other end.

#### Small Files (under ~1MB) — Send as Base64 in one message

```js
// ── CLIENT: Read file and send in one emit ────────────────────────────────────
document.getElementById('file-input').onchange = async (event) => {
  const file = event.target.files[0];

  // Guard: reject files over 1MB for this "small" path
  if (file.size > 1 * 1024 * 1024) {
    return sendLargeFile(file);  // hand off to chunked sender (see below)
  }

  // FileReader converts the file to a Base64 string
  const base64 = await fileToBase64(file);

  socket.emit('file:send', {
    name: file.name,         // original filename
    type: file.type,         // MIME type: 'image/png', 'application/pdf', etc.
    size: file.size,         // bytes
    data: base64,            // the full file content as a Base64 string
    to: targetSocketId,      // 1-on-1 target, or null for group
    roomId: currentRoomId    // group target, or null for 1-on-1
  });
};

// Helper: wraps FileReader in a Promise for async/await use
function fileToBase64(file) {
  return new Promise((resolve, reject) => {
    const reader = new FileReader();
    reader.onload  = () => resolve(reader.result.split(',')[1]);
    //                                               ↑ strip the "data:image/png;base64," prefix
    reader.onerror = reject;
    reader.readAsDataURL(file);
  });
}

// ── CLIENT: Receive a small file ──────────────────────────────────────────────
socket.on('file:receive', (fileMsg) => {
  // Reconstruct a download link from the Base64 data
  const bytes  = Uint8Array.from(atob(fileMsg.data), c => c.charCodeAt(0));
  const blob   = new Blob([bytes], { type: fileMsg.type });
  const url    = URL.createObjectURL(blob);

  const link   = document.createElement('a');
  link.href    = url;
  link.download = fileMsg.name;
  link.textContent = `📎 ${fileMsg.name} (${formatBytes(fileMsg.size)})`;
  chatDiv.appendChild(link);
});
```

#### Large Files — Chunked Transfer

> **Mental model:** Imagine sending a large poster through a mail slot that only fits envelopes. You cut the poster into numbered pieces, send them one by one, and the receiver tapes them back together in order. That's chunking.

```
CHUNKED FILE TRANSFER — How it works

  Sender                              Server                     Receiver
    │                                    │                           │
    │─── file:chunk-start ──────────────►│──── file:chunk-start ────►│
    │    { name, type, size, totalChunks,│    (forwarded to target)  │
    │      transferId }                  │                           │
    │                                    │                           │
    │─── file:chunk ────────────────────►│──── file:chunk ──────────►│
    │    { transferId, index: 0, data }  │                           │
    │─── file:chunk ────────────────────►│──── file:chunk ──────────►│
    │    { transferId, index: 1, data }  │                           │
    │         ... (N chunks total)       │                           │
    │─── file:chunk-end ────────────────►│──── file:chunk-end ──────►│
    │    { transferId }                  │    Receiver reassembles   │
    │                                    │    chunks in order        │
    │◄── file:chunk-ack ─────────────────┤◄── file:chunk-ack ────────│
    │    { transferId, receivedChunks }  │    (confirms completion)  │
```

```js
// ── CLIENT: Send a large file in chunks ───────────────────────────────────────
const CHUNK_SIZE = 64 * 1024;  // 64KB per chunk — sweet spot for Socket.IO

async function sendLargeFile(file) {
  const transferId   = crypto.randomUUID();   // unique ID for this transfer
  const totalChunks  = Math.ceil(file.size / CHUNK_SIZE);
  const arrayBuffer  = await file.arrayBuffer();

  // Step 1: Announce the incoming transfer
  socket.emit('file:chunk-start', {
    transferId,
    name: file.name,
    type: file.type,
    size: file.size,
    totalChunks,
    to: targetSocketId,
    roomId: currentRoomId
  });

  // Step 2: Send the file slice by slice
  for (let i = 0; i < totalChunks; i++) {
    const start  = i * CHUNK_SIZE;
    const end    = Math.min(start + CHUNK_SIZE, file.size);
    const slice  = arrayBuffer.slice(start, end);

    // Convert slice to Base64 for JSON transport
    const base64 = btoa(String.fromCharCode(...new Uint8Array(slice)));

    socket.emit('file:chunk', {
      transferId,
      index: i,          // chunk number — receiver uses this to reassemble in order
      data: base64
    });

    // Update progress bar
    const progress = Math.round(((i + 1) / totalChunks) * 100);
    updateProgress(transferId, progress);

    // Yield to the event loop every 10 chunks to keep UI responsive
    if (i % 10 === 0) await new Promise(r => setTimeout(r, 0));
  }

  // Step 3: Signal that all chunks have been sent
  socket.emit('file:chunk-end', { transferId });
}

// ── CLIENT: Receive a large file (reassemble chunks) ─────────────────────────
const incomingTransfers = new Map();
// ↑ Holds partially received files, keyed by transferId
//   { meta: {...}, chunks: [], receivedCount: 0 }

socket.on('file:chunk-start', (meta) => {
  // Prepare a slot to receive chunks
  incomingTransfers.set(meta.transferId, {
    meta,
    chunks: new Array(meta.totalChunks),  // pre-sized array, indexed by chunk order
    receivedCount: 0
  });
  showTransferProgress(meta);
});

socket.on('file:chunk', ({ transferId, index, data }) => {
  const transfer = incomingTransfers.get(transferId);
  if (!transfer) return;

  transfer.chunks[index] = data;   // store chunk at its correct position
  transfer.receivedCount++;

  const progress = Math.round((transfer.receivedCount / transfer.meta.totalChunks) * 100);
  updateProgress(transferId, progress);
});

socket.on('file:chunk-end', ({ transferId }) => {
  const transfer = incomingTransfers.get(transferId);
  if (!transfer) return;

  // Reassemble: decode each Base64 chunk back to binary, stitch together
  const byteArrays = transfer.chunks.map(b64 =>
    Uint8Array.from(atob(b64), c => c.charCodeAt(0))
  );

  // Merge all Uint8Arrays into one
  const totalBytes = byteArrays.reduce((sum, a) => sum + a.length, 0);
  const merged     = new Uint8Array(totalBytes);
  let   offset     = 0;
  for (const arr of byteArrays) {
    merged.set(arr, offset);
    offset += arr.length;
  }

  // Create a download link
  const blob = new Blob([merged], { type: transfer.meta.type });
  const url  = URL.createObjectURL(blob);
  renderFileLink(url, transfer.meta);
  incomingTransfers.delete(transferId);  // clean up
});
```

---

## Full Connection & Message Flow

> **Mental model:** Think of this like a restaurant. The server (waiter) is always running. Each customer (client) sits down (connects) and gets a table number (`socket.id`). They can order from any table (send messages). The waiter routes orders to the right kitchen station (room) or another customer (1-on-1). Nobody talks to each other directly — everything goes through the waiter.

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                     WebSocket Chat — Full Message Flow                       │
├──────────────────────┬───────────────────────┬───────────────────────────────┤
│   Client (Browser)   │      Socket.IO        │     Other Client(s)           │
├──────────────────────┼───────────────────────┼───────────────────────────────┤
│                      │                       │                               │
│  io('localhost:3000')│                       │                               │
│  ──────────────────► WebSocket handshake     │                               │
│  ◄────────────────── socket.id = 'abc'       │                               │
│                      │                       │                               │
│  emit('user:register'│                       │                               │
│  { username: 'Alice'}│                       │                               │
│  ──────────────────► users.set('abc','Alice')│                               │
│                      │                       │                               │
│  ─────── 1-ON-1 CHAT PATH ─────────────────────────────────────────────────  │
│                      │                       │                               │
│  emit('chat:message' │                       │                               │
│  { to:'def', text }) │                       │                               │
│  ──────────────────► socket.to('def')        │                               │
│                      │ .emit('chat:message') ►│  socket.on('chat:message')   │
│                      │                       │  renderMessage()              │
│                      │                       │                               │
│  ─────── GROUP CHAT PATH ───────────────────────────────────────────────────  │
│                      │                       │                               │
│  emit('room:join'    │                       │                               │
│  { roomId:'xyz' })   │                       │                               │
│  ──────────────────► socket.join('xyz')      │                               │
│                      │ .to('xyz').emit(      │                               │
│                      │  'room:user-joined') ─►│  (members notified)         │
│                      │                       │                               │
│  emit('chat:message' │                       │                               │
│  { roomId:'xyz',text}│                       │                               │
│  ──────────────────► io.to('xyz').emit(      │                               │
│                      │  'chat:message')      │                               │
│                      │  ─────────────────────►  (all room members receive)  │
│                      │  ◄────────────────────  (sender also receives it)    │
│                      │                       │                               │
│  ─────── FILE TRANSFER PATH ────────────────────────────────────────────────  │
│                      │                       │                               │
│  emit('file:chunk-   │                       │                               │
│   start', metadata)  │                       │                               │
│  ──────────────────► forward to target(s) ──►│  prepare transfer slot       │
│  emit('file:chunk')  │                       │                               │
│  × N times ─────────► forward each chunk ───►│  store chunk by index        │
│  emit('file:chunk-   │                       │                               │
│   end')              │                       │                               │
│  ──────────────────► forward ────────────────►│  reassemble → download link │
│                      │                       │                               │
│  ─────── DISCONNECT ────────────────────────────────────────────────────────  │
│                      │                       │                               │
│  (tab closed /       │                       │                               │
│   network drop)      │                       │                               │
│  ──────────────────► socket.on('disconnect') │                               │
│                      │ remove from users map │                               │
│                      │ notify rooms ─────────►│  'room:user-left' event     │
└──────────────────────┴───────────────────────┴───────────────────────────────┘
```

---

## Step-by-Step Implementation

### Project Structure

```
project/
├── server.js       ← Node.js server (Socket.IO + Express)
└── client.html     ← Browser client (Socket.IO client)
```

**Install dependencies:**
```bash
npm install express socket.io
```
**Run server:**
```bash
node server.js
```
**Test:** Open `http://localhost:3000` in multiple browser tabs to simulate multiple users.

---

### server.js — Complete Annotated Server

```js
// ══════════════════════════════════════════════════════════════════════════════
//  server.js — Socket.IO Chat Server
//  Handles: registration, 1-on-1 chat, group rooms, file transfer
// ══════════════════════════════════════════════════════════════════════════════

const express = require('express');
const http    = require('http');
const { Server } = require('socket.io');

const app    = express();
const server = http.createServer(app);
const io     = new Server(server, {
  maxHttpBufferSize: 10 * 1024 * 1024  // Allow up to 10MB per message (for file chunks)
  // ↑ Default is 1MB — raise it or chunked files will be silently dropped
});

app.use(express.static(__dirname));  // Serve client.html at http://localhost:3000


// ── SERVER STATE ───────────────────────────────────────────────────────────────
// In production, store this in a database (Redis, MongoDB, etc.)
// For this demo, plain Maps in memory are fine.

const users = new Map();
// users: socketId → { username, socketId }
// e.g.  'abc123' → { username: 'Alice', socketId: 'abc123' }

const rooms = new Map();
// rooms: roomId → { id, name, members: Set<socketId>, createdBy, createdAt }
// e.g.  'xyz789' → { id: 'xyz789', name: 'Study Group', members: Set{'abc','def'}, ... }


// ── HELPERS ────────────────────────────────────────────────────────────────────

function generateId() {
  return Math.random().toString(36).slice(2, 10);
  // Produces 8-char alphanumeric strings like 'a3f9b2c1'
  // Use crypto.randomUUID() in production for proper uniqueness
}

function getUserList() {
  // Returns array of all online users — sent to clients on request
  return Array.from(users.values());
}

function getRoomList() {
  // Returns rooms as plain objects with members as arrays (not Sets, which aren't JSON-safe)
  return Array.from(rooms.values()).map(r => ({
    ...r,
    members: Array.from(r.members).map(id => users.get(id)).filter(Boolean)
    // ↑ Convert member socket IDs to user objects, filtering out any stale IDs
  }));
}


// ── MAIN CONNECTION HANDLER ────────────────────────────────────────────────────

io.on('connection', (socket) => {
  console.log('Socket connected:', socket.id);


  // ┌─────────────────────────────────────────────────────────────────────────┐
  // │  USER REGISTRATION                                                      │
  // │  Client emits this right after connecting to give themselves a username │
  // └─────────────────────────────────────────────────────────────────────────┘
  socket.on('user:register', ({ username }, ack) => {
    const user = { username, socketId: socket.id };
    users.set(socket.id, user);

    // Confirm registration with the sender
    if (ack) ack({ status: 'ok', user });

    // Tell all OTHER clients that a new user is online
    socket.broadcast.emit('user:online', user);

    // Send the new user a snapshot of current state
    socket.emit('state:init', {
      users: getUserList(),     // everyone currently online
      rooms: getRoomList()      // all existing rooms
    });
  });


  // ┌─────────────────────────────────────────────────────────────────────────┐
  // │  1-ON-1 CHAT                                                            │
  // │  Routed by target socket ID — only the addressed client receives it     │
  // └─────────────────────────────────────────────────────────────────────────┘
  socket.on('chat:dm', ({ to, text }, ack) => {
    const sender = users.get(socket.id);
    if (!sender) return;

    const msg = {
      id: generateId(),
      text,
      from: sender,
      to,
      timestamp: Date.now(),
      type: 'text'
    };

    // Deliver to recipient
    socket.to(to).emit('chat:dm', msg);
    // Echo back to sender (so they see it in their own chat window)
    socket.emit('chat:dm', msg);

    if (ack) ack({ status: 'ok', messageId: msg.id });
  });


  // ┌─────────────────────────────────────────────────────────────────────────┐
  // │  GROUP ROOMS — CREATE                                                   │
  // └─────────────────────────────────────────────────────────────────────────┘
  socket.on('room:create', ({ name }, ack) => {
    const creator = users.get(socket.id);
    if (!creator) return;

    const room = {
      id: generateId(),
      name,
      members: new Set([socket.id]),  // creator is automatically a member
      createdBy: socket.id,
      createdAt: Date.now()
    };

    rooms.set(room.id, room);
    socket.join(room.id);
    // ↑ socket.join() is the Socket.IO call that adds this socket to the room.
    //   After this, io.to(room.id).emit() will reach this user.

    if (ack) ack({ status: 'ok', room: { ...room, members: [creator] } });

    // Tell all clients a new room exists
    io.emit('room:created', {
      ...room,
      members: Array.from(room.members).map(id => users.get(id)).filter(Boolean)
    });
  });


  // ┌─────────────────────────────────────────────────────────────────────────┐
  // │  GROUP ROOMS — JOIN                                                     │
  // └─────────────────────────────────────────────────────────────────────────┘
  socket.on('room:join', ({ roomId }, ack) => {
    const room = rooms.get(roomId);
    const user = users.get(socket.id);
    if (!room || !user) return;

    room.members.add(socket.id);
    socket.join(roomId);

    if (ack) ack({ status: 'ok' });

    // Notify everyone else in the room
    socket.to(roomId).emit('room:user-joined', { roomId, user });

    // Send the new member a list of who's already in the room
    socket.emit('room:member-list', {
      roomId,
      members: Array.from(room.members).map(id => users.get(id)).filter(Boolean)
    });
  });


  // ┌─────────────────────────────────────────────────────────────────────────┐
  // │  GROUP ROOMS — LEAVE                                                    │
  // └─────────────────────────────────────────────────────────────────────────┘
  socket.on('room:leave', ({ roomId }) => {
    const room = rooms.get(roomId);
    const user = users.get(socket.id);
    if (!room) return;

    room.members.delete(socket.id);
    socket.leave(roomId);

    // Notify remaining members
    socket.to(roomId).emit('room:user-left', { roomId, user });

    // Optional: delete empty rooms
    if (room.members.size === 0) {
      rooms.delete(roomId);
      io.emit('room:deleted', { roomId });
    }
  });


  // ┌─────────────────────────────────────────────────────────────────────────┐
  // │  GROUP CHAT MESSAGE                                                     │
  // └─────────────────────────────────────────────────────────────────────────┘
  socket.on('chat:group', ({ roomId, text }, ack) => {
    const room   = rooms.get(roomId);
    const sender = users.get(socket.id);
    if (!room || !sender) return;
    if (!room.members.has(socket.id)) return;  // must be a member to send

    const msg = {
      id: generateId(),
      text,
      from: sender,
      roomId,
      timestamp: Date.now(),
      type: 'text'
    };

    io.to(roomId).emit('chat:group', msg);
    // ↑ io.to() includes the sender — they see their own message confirmed from server
    if (ack) ack({ status: 'ok', messageId: msg.id });
  });


  // ┌─────────────────────────────────────────────────────────────────────────┐
  // │  TYPING INDICATOR                                                       │
  // └─────────────────────────────────────────────────────────────────────────┘
  socket.on('user:typing', ({ to, roomId, isTyping }) => {
    const user = users.get(socket.id);
    if (!user) return;

    const payload = { user, isTyping };

    if (roomId) {
      socket.to(roomId).emit('user:typing', { ...payload, roomId });
      //  ↑ socket.to() excludes sender — no need to show "you are typing"
    } else if (to) {
      socket.to(to).emit('user:typing', { ...payload, to: socket.id });
    }
  });


  // ┌─────────────────────────────────────────────────────────────────────────┐
  // │  FILE TRANSFER — SMALL FILES (sent as one Base64 blob)                  │
  // └─────────────────────────────────────────────────────────────────────────┘
  socket.on('file:send', (fileMsg) => {
    const sender = users.get(socket.id);
    if (!sender) return;

    const enriched = { ...fileMsg, from: sender, timestamp: Date.now() };

    if (fileMsg.roomId) {
      io.to(fileMsg.roomId).emit('file:receive', enriched);
    } else if (fileMsg.to) {
      socket.to(fileMsg.to).emit('file:receive', enriched);
      socket.emit('file:receive', enriched);  // echo to sender
    }
  });


  // ┌─────────────────────────────────────────────────────────────────────────┐
  // │  FILE TRANSFER — LARGE FILES (chunked relay)                            │
  // │  Server acts as a pure relay — it doesn't store chunks                 │
  // └─────────────────────────────────────────────────────────────────────────┘
  socket.on('file:chunk-start', (meta) => {
    const sender   = users.get(socket.id);
    const enriched = { ...meta, from: sender };
    if (meta.roomId) {
      socket.to(meta.roomId).emit('file:chunk-start', enriched);
    } else if (meta.to) {
      socket.to(meta.to).emit('file:chunk-start', enriched);
    }
  });

  socket.on('file:chunk', ({ transferId, index, data, to, roomId }) => {
    // Forward the chunk as-is to the target — server never buffers it
    if (roomId) {
      socket.to(roomId).emit('file:chunk', { transferId, index, data });
    } else if (to) {
      socket.to(to).emit('file:chunk', { transferId, index, data });
    }
  });

  socket.on('file:chunk-end', ({ transferId, to, roomId }) => {
    if (roomId) {
      socket.to(roomId).emit('file:chunk-end', { transferId });
    } else if (to) {
      socket.to(to).emit('file:chunk-end', { transferId });
      socket.emit('file:chunk-end', { transferId });  // ack to sender
    }
  });


  // ┌─────────────────────────────────────────────────────────────────────────┐
  // │  DISCONNECT                                                             │
  // └─────────────────────────────────────────────────────────────────────────┘
  socket.on('disconnect', () => {
    const user = users.get(socket.id);
    if (!user) return;

    // Remove from all rooms they were in
    rooms.forEach((room, roomId) => {
      if (room.members.has(socket.id)) {
        room.members.delete(socket.id);
        socket.to(roomId).emit('room:user-left', { roomId, user });
        if (room.members.size === 0) {
          rooms.delete(roomId);
          io.emit('room:deleted', { roomId });
        }
      }
    });

    users.delete(socket.id);
    io.emit('user:offline', { socketId: socket.id });
    // ↑ io.emit() (not socket.emit) because the socket is already gone
  });

});


server.listen(3000, () => console.log('Server running → http://localhost:3000'));
```

---

### client.html — Complete Annotated Client

#### HTML Structure

```html
<!DOCTYPE html>
<html>
<head>
  <title>WebSocket Chat</title>
  <style>
    body        { display: flex; font-family: Arial, sans-serif; margin: 0; }
    #sidebar    { width: 220px; border-right: 1px solid #ccc; padding: 12px; height: 100vh; overflow-y: auto; }
    #main       { flex: 1; display: flex; flex-direction: column; padding: 12px; }
    #messages   { flex: 1; overflow-y: scroll; border: 1px solid #ccc; padding: 10px; margin-bottom: 8px; min-height: 300px; }
    #input-area { display: flex; gap: 8px; }
    #msg-input  { flex: 1; padding: 8px; }
    .msg        { margin: 4px 0; }
    .msg.me     { text-align: right; color: #1a6fbb; }
    .msg.system { color: #888; font-style: italic; font-size: 0.9em; }
    #typing     { color: #888; font-size: 0.85em; height: 18px; }
    #status     { font-weight: bold; margin-bottom: 8px; }
    .room-item  { cursor: pointer; padding: 4px; border-radius: 4px; }
    .room-item:hover { background: #f0f0f0; }
    .room-item.active { background: #d0e8ff; }
    progress    { width: 100%; margin: 4px 0; }
  </style>
</head>
<body>

  <div id="sidebar">
    <div id="status">⚪ Connecting...</div>
    <hr>
    <b>Users Online</b>
    <div id="user-list"></div>
    <hr>
    <b>Rooms</b>
    <div id="room-list"></div>
    <button id="create-room-btn">+ New Room</button>
  </div>

  <div id="main">
    <div id="chat-header">Select a user or room to chat</div>
    <div id="messages"></div>
    <div id="typing"></div>
    <div id="input-area">
      <input id="msg-input" placeholder="Type a message..." />
      <button id="send-btn">Send</button>
      <input id="file-input" type="file" style="display:none" />
      <button id="attach-btn">📎</button>
    </div>
  </div>

  <script src="https://cdn.socket.io/4.8.3/socket.io.min.js"></script>
  <script>
    /* all JavaScript below */
  </script>
</body>
</html>
```

#### JavaScript — Step by Step

**Step 1 — Setup and connect**

```js
// ── Global State ──────────────────────────────────────────────────────────────
let currentTarget = null;
// currentTarget: { type: 'dm', socketId: '...' } | { type: 'room', roomId: '...' }
// ↑ Tracks which conversation is currently open in the UI

let myUsername  = null;    // set after registration
let mySocketId  = null;    // set after 'connect' event
let typingTimer = null;    // debounce handle for typing indicator

// ── DOM References ────────────────────────────────────────────────────────────
const statusEl   = document.getElementById('status');
const messagesEl = document.getElementById('messages');
const typingEl   = document.getElementById('typing');
const msgInput   = document.getElementById('msg-input');
const userListEl = document.getElementById('user-list');
const roomListEl = document.getElementById('room-list');

// ── Connect to server ─────────────────────────────────────────────────────────
const socket = io('http://localhost:3000');

socket.on('connect', () => {
  mySocketId = socket.id;
  statusEl.textContent = '🟢 Connected';

  // Prompt for username and register with the server
  myUsername = prompt('Enter your username:') || 'User_' + socket.id.slice(0, 4);
  socket.emit('user:register', { username: myUsername }, (res) => {
    // res is the acknowledgement from the server
    if (res.status === 'ok') statusEl.textContent = `🟢 ${myUsername}`;
  });
});

socket.on('disconnect', () => {
  statusEl.textContent = '🔴 Disconnected';
});
```

**Step 2 — User list and online/offline events**

```js
// ── Receive initial snapshot of all users and rooms ───────────────────────────
socket.on('state:init', ({ users, rooms }) => {
  renderUserList(users);
  renderRoomList(rooms);
});

// ── A new user came online ─────────────────────────────────────────────────────
socket.on('user:online', (user) => {
  appendToUserList(user);
  appendSystemMessage(`${user.username} came online`);
});

// ── A user went offline ────────────────────────────────────────────────────────
socket.on('user:offline', ({ socketId }) => {
  removeFromUserList(socketId);
  // If we were in a DM with them, note they're gone
  if (currentTarget?.type === 'dm' && currentTarget.socketId === socketId) {
    appendSystemMessage('User disconnected');
  }
});

// ── Render functions ──────────────────────────────────────────────────────────
function renderUserList(users) {
  userListEl.innerHTML = '';
  users.forEach(appendToUserList);
}

function appendToUserList(user) {
  if (user.socketId === mySocketId) return;  // don't show yourself
  const el = document.createElement('div');
  el.textContent  = user.username;
  el.dataset.id   = user.socketId;
  el.className    = 'room-item';
  el.onclick      = () => openDM(user);
  userListEl.appendChild(el);
}

function removeFromUserList(socketId) {
  userListEl.querySelector(`[data-id="${socketId}"]`)?.remove();
}
```

**Step 3 — 1-on-1 (DM) chat**

```js
// ── Open a DM with a user ─────────────────────────────────────────────────────
function openDM(user) {
  currentTarget = { type: 'dm', socketId: user.socketId, username: user.username };
  document.getElementById('chat-header').textContent = `DM: ${user.username}`;
  messagesEl.innerHTML = '';
  // In production: load message history from server here
}

// ── Send a DM ─────────────────────────────────────────────────────────────────
function sendDM(text) {
  socket.emit('chat:dm', { to: currentTarget.socketId, text }, (res) => {
    if (res?.status !== 'ok') appendSystemMessage('Message failed to deliver');
  });
}

// ── Receive a DM ─────────────────────────────────────────────────────────────
socket.on('chat:dm', (msg) => {
  // Only render if it belongs to the currently open conversation
  const isCurrentConvo =
    currentTarget?.type === 'dm' &&
    (msg.from.socketId === currentTarget.socketId ||
     msg.from.socketId === mySocketId);

  if (isCurrentConvo) renderMessage(msg);
  else notifyUnread(msg.from.socketId);  // flash the sender's name in sidebar
});
```

**Step 4 — Creating and joining group rooms**

```js
// ── Create a new room ─────────────────────────────────────────────────────────
document.getElementById('create-room-btn').onclick = () => {
  const name = prompt('Room name:');
  if (!name) return;

  socket.emit('room:create', { name }, (res) => {
    if (res.status === 'ok') openRoom(res.room);
    // ↑ The ack gives us the room object immediately —
    //   we don't have to wait for the 'room:created' broadcast
  });
};

// ── A new room was created (by anyone) ───────────────────────────────────────
socket.on('room:created', (room) => {
  appendToRoomList(room);
});

// ── Join an existing room ─────────────────────────────────────────────────────
function joinRoom(room) {
  socket.emit('room:join', { roomId: room.id }, (res) => {
    if (res.status === 'ok') openRoom(room);
  });
}

// ── Open a room in the chat window ───────────────────────────────────────────
function openRoom(room) {
  currentTarget = { type: 'room', roomId: room.id, name: room.name };
  document.getElementById('chat-header').textContent = `# ${room.name}`;
  messagesEl.innerHTML = '';
  // Highlight active room in sidebar
  document.querySelectorAll('.room-item').forEach(el => el.classList.remove('active'));
  document.querySelector(`[data-room-id="${room.id}"]`)?.classList.add('active');
}

// ── Leave current room ────────────────────────────────────────────────────────
function leaveCurrentRoom() {
  if (currentTarget?.type !== 'room') return;
  socket.emit('room:leave', { roomId: currentTarget.roomId });
  currentTarget = null;
  messagesEl.innerHTML = '';
  document.getElementById('chat-header').textContent = 'Select a user or room';
}

// ── Someone joined/left our current room ─────────────────────────────────────
socket.on('room:user-joined', ({ roomId, user }) => {
  if (currentTarget?.roomId === roomId)
    appendSystemMessage(`${user.username} joined`);
});

socket.on('room:user-left', ({ roomId, user }) => {
  if (currentTarget?.roomId === roomId)
    appendSystemMessage(`${user.username} left`);
});

socket.on('room:deleted', ({ roomId }) => {
  document.querySelector(`[data-room-id="${roomId}"]`)?.remove();
  if (currentTarget?.roomId === roomId) {
    currentTarget = null;
    appendSystemMessage('This room was deleted');
  }
});

// ── Render functions ──────────────────────────────────────────────────────────
function renderRoomList(rooms) {
  roomListEl.innerHTML = '';
  rooms.forEach(appendToRoomList);
}

function appendToRoomList(room) {
  const el = document.createElement('div');
  el.textContent       = `# ${room.name}`;
  el.dataset.roomId    = room.id;
  el.className         = 'room-item';
  el.onclick           = () => joinRoom(room);
  roomListEl.appendChild(el);
}
```

**Step 5 — Group messages and typing indicator**

```js
// ── Send button handler — routes to DM or group depending on currentTarget ───
document.getElementById('send-btn').onclick = sendCurrentMessage;
msgInput.addEventListener('keydown', (e) => {
  if (e.key === 'Enter') sendCurrentMessage();
});

function sendCurrentMessage() {
  const text = msgInput.value.trim();
  if (!text || !currentTarget) return;
  msgInput.value = '';

  if (currentTarget.type === 'dm') {
    sendDM(text);
  } else {
    sendGroupMessage(text);
  }
  stopTyping();  // stop typing indicator when message is sent
}

// ── Send a group message ──────────────────────────────────────────────────────
function sendGroupMessage(text) {
  socket.emit('chat:group', { roomId: currentTarget.roomId, text });
}

// ── Receive a group message ───────────────────────────────────────────────────
socket.on('chat:group', (msg) => {
  if (currentTarget?.roomId === msg.roomId) renderMessage(msg);
  else notifyUnread(msg.roomId);
});

// ── Typing indicator ──────────────────────────────────────────────────────────
msgInput.addEventListener('input', () => {
  if (!currentTarget) return;

  // Tell the server "I'm typing"
  emitTyping(true);

  // After 1.5s of no input, tell server "I stopped typing"
  clearTimeout(typingTimer);
  typingTimer = setTimeout(() => stopTyping(), 1500);
});

function emitTyping(isTyping) {
  if (currentTarget.type === 'dm') {
    socket.emit('user:typing', { to: currentTarget.socketId, isTyping });
  } else {
    socket.emit('user:typing', { roomId: currentTarget.roomId, isTyping });
  }
}

function stopTyping() {
  clearTimeout(typingTimer);
  emitTyping(false);
}

// ── Display typing indicator received from another user ───────────────────────
const activeTypers = new Set();

socket.on('user:typing', ({ user, isTyping, roomId, to }) => {
  // Only show for the currently open conversation
  const isCurrentConvo =
    (currentTarget?.type === 'dm'   && to === mySocketId) ||
    (currentTarget?.type === 'room' && roomId === currentTarget.roomId);

  if (!isCurrentConvo) return;

  if (isTyping) {
    activeTypers.add(user.username);
  } else {
    activeTypers.delete(user.username);
  }

  // Update the typing label
  if (activeTypers.size === 0) {
    typingEl.textContent = '';
  } else {
    const names = Array.from(activeTypers).join(', ');
    typingEl.textContent = `${names} ${activeTypers.size === 1 ? 'is' : 'are'} typing...`;
  }
});
```

**Step 6 — File sending (small and large)**

```js
// ── Attach button opens file picker ──────────────────────────────────────────
document.getElementById('attach-btn').onclick = () => {
  document.getElementById('file-input').click();
};

document.getElementById('file-input').onchange = async (e) => {
  const file = e.target.files[0];
  if (!file || !currentTarget) return;
  e.target.value = '';  // reset so same file can be re-selected

  if (file.size > 1 * 1024 * 1024) {
    await sendLargeFile(file);   // chunked path
  } else {
    await sendSmallFile(file);   // single-emit path
  }
};

// ── Small file sender ─────────────────────────────────────────────────────────
async function sendSmallFile(file) {
  const base64 = await fileToBase64(file);

  socket.emit('file:send', {
    name: file.name,
    type: file.type,
    size: file.size,
    data: base64,
    ...(currentTarget.type === 'dm'
      ? { to: currentTarget.socketId }
      : { roomId: currentTarget.roomId })
  });
}

function fileToBase64(file) {
  return new Promise((resolve, reject) => {
    const reader = new FileReader();
    reader.onload  = () => resolve(reader.result.split(',')[1]);
    reader.onerror = reject;
    reader.readAsDataURL(file);
  });
}

// ── Large file sender (chunked) ───────────────────────────────────────────────
const CHUNK_SIZE = 64 * 1024;

async function sendLargeFile(file) {
  const transferId  = Math.random().toString(36).slice(2);
  const totalChunks = Math.ceil(file.size / CHUNK_SIZE);
  const buffer      = await file.arrayBuffer();
  const target      = currentTarget.type === 'dm'
    ? { to: currentTarget.socketId }
    : { roomId: currentTarget.roomId };

  // Announce the transfer
  socket.emit('file:chunk-start', { transferId, name: file.name, type: file.type, size: file.size, totalChunks, ...target });

  // Show progress bar in UI
  const progressId = showProgressBar(file.name, transferId);

  for (let i = 0; i < totalChunks; i++) {
    const slice  = buffer.slice(i * CHUNK_SIZE, Math.min((i + 1) * CHUNK_SIZE, file.size));
    const base64 = btoa(String.fromCharCode(...new Uint8Array(slice)));

    socket.emit('file:chunk', { transferId, index: i, data: base64, ...target });

    updateProgressBar(progressId, Math.round(((i + 1) / totalChunks) * 100));
    if (i % 10 === 0) await new Promise(r => setTimeout(r, 0));  // keep UI live
  }

  socket.emit('file:chunk-end', { transferId, ...target });
  removeProgressBar(progressId);
}

// ── Receive a small file ──────────────────────────────────────────────────────
socket.on('file:receive', (fileMsg) => {
  const bytes = Uint8Array.from(atob(fileMsg.data), c => c.charCodeAt(0));
  const blob  = new Blob([bytes], { type: fileMsg.type });
  const url   = URL.createObjectURL(blob);
  renderFileMessage(fileMsg, url);
});

// ── Receive a large file (reassemble) ────────────────────────────────────────
const incomingTransfers = new Map();

socket.on('file:chunk-start', (meta) => {
  incomingTransfers.set(meta.transferId, {
    meta,
    chunks: new Array(meta.totalChunks),
    received: 0
  });
  showProgressBar(meta.name, meta.transferId);
});

socket.on('file:chunk', ({ transferId, index, data }) => {
  const t = incomingTransfers.get(transferId);
  if (!t) return;
  t.chunks[index] = data;
  t.received++;
  updateProgressBar(transferId, Math.round((t.received / t.meta.totalChunks) * 100));
});

socket.on('file:chunk-end', ({ transferId }) => {
  const t = incomingTransfers.get(transferId);
  if (!t) return;

  const byteArrays = t.chunks.map(b64 => Uint8Array.from(atob(b64), c => c.charCodeAt(0)));
  const total      = byteArrays.reduce((s, a) => s + a.length, 0);
  const merged     = new Uint8Array(total);
  let   off        = 0;
  for (const arr of byteArrays) { merged.set(arr, off); off += arr.length; }

  const blob = new Blob([merged], { type: t.meta.type });
  const url  = URL.createObjectURL(blob);
  renderFileMessage(t.meta, url);
  removeProgressBar(transferId);
  incomingTransfers.delete(transferId);
});
```

**Step 7 — Render helpers**

```js
// ── Render a text message ─────────────────────────────────────────────────────
function renderMessage(msg) {
  const isMine = msg.from.socketId === mySocketId;
  const el     = document.createElement('div');
  el.className = `msg ${isMine ? 'me' : ''}`;
  el.innerHTML = `<b>${isMine ? 'Me' : msg.from.username}</b>: ${escapeHtml(msg.text)}
                  <small style="color:#aaa"> ${formatTime(msg.timestamp)}</small>`;
  messagesEl.appendChild(el);
  messagesEl.scrollTop = messagesEl.scrollHeight;
}

// ── Render a file message ─────────────────────────────────────────────────────
function renderFileMessage(meta, url) {
  const el   = document.createElement('div');
  el.className = 'msg';
  const link = document.createElement('a');
  link.href  = url;
  link.download = meta.name;
  link.textContent = `📎 ${meta.name} (${formatBytes(meta.size)})`;
  el.appendChild(link);
  messagesEl.appendChild(el);
  messagesEl.scrollTop = messagesEl.scrollHeight;
}

// ── Render a system message (user joined, etc.) ───────────────────────────────
function appendSystemMessage(text) {
  const el = document.createElement('div');
  el.className = 'msg system';
  el.textContent = text;
  messagesEl.appendChild(el);
  messagesEl.scrollTop = messagesEl.scrollHeight;
}

// ── Progress bar helpers ──────────────────────────────────────────────────────
function showProgressBar(name, id) {
  const el  = document.createElement('div');
  el.id     = `progress-${id}`;
  el.innerHTML = `<small>${escapeHtml(name)}</small><progress value="0" max="100"></progress>`;
  messagesEl.appendChild(el);
  return id;
}

function updateProgressBar(id, pct) {
  const el = document.getElementById(`progress-${id}`);
  if (el) el.querySelector('progress').value = pct;
}

function removeProgressBar(id) {
  document.getElementById(`progress-${id}`)?.remove();
}

// ── Utilities ─────────────────────────────────────────────────────────────────
function notifyUnread(id) {
  const el = document.querySelector(`[data-id="${id}"], [data-room-id="${id}"]`);
  if (el) el.style.fontWeight = 'bold';  // bold = unread indicator
}

function escapeHtml(str) {
  // Prevent XSS — never render user input as raw HTML
  return str.replace(/&/g,'&amp;').replace(/</g,'&lt;').replace(/>/g,'&gt;');
}

function formatTime(ts) {
  return new Date(ts).toLocaleTimeString([], { hour: '2-digit', minute: '2-digit' });
}

function formatBytes(bytes) {
  if (bytes < 1024)        return bytes + ' B';
  if (bytes < 1024 * 1024) return (bytes / 1024).toFixed(1) + ' KB';
  return (bytes / (1024 * 1024)).toFixed(1) + ' MB';
}
```

---

### Step 8 — The Full Execution Order (Putting It All Together)

> **Mental model:** Every feature above defined ingredients. This is the full recipe — what fires when, in what order, and which branches each scenario takes. If you're confused about "who sends what and when", start here.

```js
// ═══════════════════════════════════════════════════════════════════════════════
//  EXECUTION ORDER — client.html
//  Read top to bottom. Comments mark WHEN each block runs.
// ═══════════════════════════════════════════════════════════════════════════════


// ── ON PAGE LOAD ──────────────────────────────────────────────────────────────
//    Socket connects. No messages yet. All users go through this.

const socket = io('http://localhost:3000');

socket.on('connect', () => {
  mySocketId = socket.id;
  myUsername = prompt('Enter your username:') || 'User_' + socket.id.slice(0, 4);

  socket.emit('user:register', { username: myUsername }, (res) => {
    // ↑ Ack fires immediately when server responds
    statusEl.textContent = `🟢 ${myUsername}`;
  });
  // Server responds with 'state:init' which populates the user + room lists
});

socket.on('state:init', ({ users, rooms }) => {
  renderUserList(users);   // populate sidebar: user list
  renderRoomList(rooms);   // populate sidebar: room list
  // UI is now fully ready. User can click a user or room to start chatting.
});


// ══════════════════════════════════════════════════════════════════════════════
//  FROM HERE, FLOWS DIVERGE BY USER ACTION
// ══════════════════════════════════════════════════════════════════════════════


// ┌─────────────────────────────────────────────────────────────────────────────┐
// │  SCENARIO A: User clicks a name → opens a DM                               │
// └─────────────────────────────────────────────────────────────────────────────┘

openDM(user);
// Sets currentTarget = { type: 'dm', socketId: '...', username: '...' }
// Clears messages area

// User types a message and hits Send:
sendCurrentMessage();
//  → currentTarget.type === 'dm', so calls sendDM(text)
//  → socket.emit('chat:dm', { to: socketId, text })
//  → Server routes it: socket.to(to).emit() + echo to sender
//  → Both clients hit socket.on('chat:dm') → renderMessage()


// ┌─────────────────────────────────────────────────────────────────────────────┐
// │  SCENARIO B: User creates a new room                                        │
// └─────────────────────────────────────────────────────────────────────────────┘

// User clicks "+ New Room", enters name:
socket.emit('room:create', { name }, (res) => {
  openRoom(res.room);
  // Sets currentTarget = { type: 'room', roomId: '...', name: '...' }
});
// Server: creates room, calls socket.join(roomId), emits 'room:created' to ALL
// All clients: socket.on('room:created') → appendToRoomList()


// ┌─────────────────────────────────────────────────────────────────────────────┐
// │  SCENARIO C: User clicks an existing room → joins it                        │
// └─────────────────────────────────────────────────────────────────────────────┘

joinRoom(room);
// → socket.emit('room:join', { roomId })
// Server: socket.join(roomId), emits 'room:user-joined' to others in room
// → Ack received: openRoom(room) sets currentTarget
// → socket.on('room:member-list') updates who's in the room

// User sends a group message:
sendCurrentMessage();
//  → currentTarget.type === 'room', so calls sendGroupMessage(text)
//  → socket.emit('chat:group', { roomId, text })
//  → Server: io.to(roomId).emit('chat:group', enrichedMsg)
//  → ALL room members (including sender) hit socket.on('chat:group') → renderMessage()


// ┌─────────────────────────────────────────────────────────────────────────────┐
// │  SCENARIO D: User types in the input box → typing indicator fires           │
// └─────────────────────────────────────────────────────────────────────────────┘

// msgInput 'input' event fires on every keystroke:
//  → emitTyping(true) — tells server "I'm typing"
//  → debounce timer reset to 1.5s
//  → after 1.5s of no input: stopTyping() → emitTyping(false)
// Server: socket.to(target).emit('user:typing', { user, isTyping })
// Recipient: socket.on('user:typing') → add/remove from activeTypers set → update label


// ┌─────────────────────────────────────────────────────────────────────────────┐
// │  SCENARIO E: User attaches a file                                           │
// └─────────────────────────────────────────────────────────────────────────────┘

// User clicks 📎 → file picker opens → file selected → onchange fires
// BRANCH: file.size <= 1MB
//  → sendSmallFile(file)
//  → fileToBase64() → one socket.emit('file:send', { data: base64, ... })
//  → Server: forwards to target
//  → Recipient: socket.on('file:receive') → decode Base64 → Blob → URL → renderFileMessage()

// BRANCH: file.size > 1MB
//  → sendLargeFile(file)
//  → socket.emit('file:chunk-start')  ← announces transfer
//  → for loop: socket.emit('file:chunk') × N  ← sends each 64KB slice
//  → socket.emit('file:chunk-end')    ← signals completion
//  → Server: relays each event to target as it arrives (no buffering)
//  → Recipient:
//      'file:chunk-start' → create transfer slot in incomingTransfers Map
//      'file:chunk'       × N → store each chunk at index, update progress bar
//      'file:chunk-end'   → merge all chunks → Blob → URL → renderFileMessage()


// ┌─────────────────────────────────────────────────────────────────────────────┐
// │  SCENARIO F: A user disconnects (tab close / network drop)                  │
// └─────────────────────────────────────────────────────────────────────────────┘

// Server: socket.on('disconnect') fires automatically
//  → removes socket from all rooms, emits 'room:user-left' to each
//  → deletes from users Map, emits 'user:offline' to everyone
// Other clients:
//  → socket.on('user:offline') → removeFromUserList(socketId)
//  → socket.on('room:user-left') → appendSystemMessage() in affected rooms


// ═══════════════════════════════════════════════════════════════════════════════
//  FULL EVENT MAP — Every event name used in this app
//
//  CLIENT → SERVER         SERVER → CLIENT
//  ─────────────────       ─────────────────────────────
//  user:register           state:init       (snapshot on connect)
//  chat:dm                 chat:dm          (1-on-1 message)
//  chat:group              chat:group       (group message)
//  room:create             room:created     (new room broadcast)
//  room:join               room:user-joined (member joined notification)
//  room:leave              room:user-left   (member left notification)
//  user:typing             room:member-list (room members on join)
//  file:send               room:deleted     (room removed)
//  file:chunk-start        user:online      (new user broadcast)
//  file:chunk              user:offline     (user disconnected)
//  file:chunk-end          file:receive     (small file delivery)
//                          file:chunk-start (large file announced)
//                          file:chunk       (large file piece)
//                          file:chunk-end   (large file complete)
//                          user:typing      (typing indicator)
// ═══════════════════════════════════════════════════════════════════════════════
```

---

## Quick Reference — Cheat Sheets

### Who Receives the Message?

```
  socket.emit(event, data)              →  client sends to SERVER
  socket.to(socketId).emit(event, data) →  server to ONE specific client
  socket.broadcast.emit(event, data)    →  server to EVERYONE except sender
  socket.to(roomId).emit(event, data)   →  server to room, EXCLUDING sender
  io.to(roomId).emit(event, data)       →  server to room, INCLUDING sender
  io.emit(event, data)                  →  server to EVERY connected client
```

### File Size Decision Tree

```
  User selects file
        │
        ▼
  file.size > 1MB?
     │         │
    YES        NO
     │         │
     ▼         ▼
  Chunked    Base64 in
  Transfer   one emit
  (loop of   (file:send)
  file:chunk)
```

### The 6 Things Every Socket Handler Must Do on Connect

| Step | What it does |
|---|---|
| `socket.emit('user:register', ...)` | Give yourself a username on the server |
| `socket.on('state:init', ...)` | Receive existing users and rooms |
| `socket.on('user:online/offline', ...)` | Keep user list updated |
| `socket.on('chat:dm', ...)` | Handle incoming 1-on-1 messages |
| `socket.on('chat:group', ...)` | Handle incoming group messages |
| `socket.on('file:receive / chunk-*', ...)` | Handle file deliveries |

---

## Common Errors & Fixes

| Error | Cause | Fix |
|---|---|---|
| Messages not reaching recipient | Targeting by `socket.id` which changed after reconnect | Use a stable user ID (not `socket.id`) stored server-side |
| File chunks arrive out of order | Network reordering | Store by `index` in a pre-sized array — don't assume order |
| Large file silently dropped | `maxHttpBufferSize` too small | Set `maxHttpBufferSize: 10 * 1024 * 1024` in the server `Server()` config |
| Typing indicator never clears | `emitTyping(false)` not called on disconnect | Call `stopTyping()` in `socket.on('disconnect')` on client |
| Room messages received twice | Used `io.to()` and also echoed manually | Use only `io.to()` for group — it already includes the sender |
| XSS via chat messages | Rendering `msg.text` as raw HTML | Always escape user content with `escapeHtml()` before inserting |
| Memory leak on large transfers | `incomingTransfers` map never cleaned up | Always call `incomingTransfers.delete(transferId)` after assembly |

---

## Testing Locally

```
Terminal:     node server.js
Tab 1:        http://localhost:3000  → enter username "Alice"
Tab 2:        http://localhost:3000  → enter username "Bob"
Tab 3:        http://localhost:3000  → enter username "Carol"

Test DM:      Alice clicks Bob in sidebar → types → sends
Test Group:   Alice clicks "+ New Room" → names it → Bob clicks room name → both chat
Test File:    Either user clicks 📎 → selects a small image → appears as download link
              Select a file > 1MB → progress bar appears → reassembled on receive
```

> **Tip:** Open browser DevTools → Network tab → filter by `WS` to see every Socket.IO frame in real time. Each message appears as a row — click it to inspect the payload.

# 5. Real-Time Communication with Socket.IO

## Why WebSockets?

**HTTP**: Client asks, server responds, connection closes. Requires repeated polling for updates.

**WebSocket**: Persistent two-way connection. Server pushes updates instantly without client asking. Essential for real-time features like chat, notifications, live updates.

---

## 1. Production Folder Structure

We'll use this structure throughout the chapter:

```text
chat-app/
├── src/
│   ├── server.js                    # Entry point: DB → Server → Socket.io
│   ├── app.js                       # Express app configuration
│   │
│   ├── config/
│   │   ├── database.js              # MongoDB connection
│   │   ├── env.js                   # Environment variables
│   │   └── socketio.js              # Socket.io setup with auth
│   │
│   ├── models/
│   │   ├── User.model.js            # User schema (with presence fields)
│   │   ├── Conversation.model.js    # Conversation schema
│   │   └── Message.model.js         # Message schema
│   │
│   ├── middlewares/
│   │   └── auth.middleware.js       # JWT verification for HTTP routes
│   │
│   ├── socket/
│   │   ├── directMessages.js        # One-to-one message handlers
│   │   └── groupMessages.js         # Group chat handlers
│   │
│   └── routes/
│       └── upload.routes.js         # File upload endpoint
│
├── uploads/                         # Uploaded files storage
├── .env                             # Environment configuration
└── package.json
```

---

## 2. Startup Order (Critical Understanding)

**Question**: In what order should we start things?

**Answer**: Database → HTTP Server → Socket.io

```
┌─────────────────────────────────────────────────────────────┐
│                    Application Startup                      │
└─────────────────────────────────────────────────────────────┘
                           │
                           ▼
              ┌─────────────────────────┐
              │ 1. Connect to MongoDB   │
              │    mongoose.connect()   │
              └────────────┬────────────┘
                           │
                           ▼
              ┌─────────────────────────┐
              │ 2. Create HTTP Server   │
              │ http.createServer(app)  │
              └────────────┬────────────┘
                           │
                           ▼
              ┌─────────────────────────┐
              │ 3. Attach Socket.io     │
              │ new Server(httpServer)  │
              └────────────┬────────────┘
                           │
                           ▼
              ┌─────────────────────────┐
              │ 4. Start listening      │
              │    server.listen()      │
              └────────────┬────────────┘
                           │
                           ▼
                ┌──────────────────┐
                │  Server ready    │
                └──────────────────┘
```

**Why this order?**
- Socket handlers need database access (to save messages, fetch users)
- Socket.io needs an HTTP server to attach to
- If DB fails, we shouldn't accept connections

**File**: `src/server.js`

```js
require("dotenv").config();
const http = require("http");
const app = require("./app");
const connectDB = require("./config/database");
const setupSocketIO = require("./config/socketio");

const PORT = process.env.PORT || 3000;

// Create HTTP server (Socket.io will attach to this)
const server = http.createServer(app);

// STEP 1: Connect to MongoDB first
connectDB()
  .then(() => {
    console.log("✓ MongoDB connected");
    
    // STEP 2: Setup Socket.io (after DB is ready)
    const io = setupSocketIO(server);
    
    // Make io accessible in HTTP routes (for sending notifications)
    app.set("io", io);
    
    // STEP 3: Start listening for connections
    server.listen(PORT, () => {
      console.log(`✓ Server running on port ${PORT}`);
      console.log(`✓ Socket.io ready for connections`);
    });
  })
  .catch((error) => {
    console.error("✗ MongoDB connection failed:", error);
    process.exit(1); // Exit if database fails
  });
```

---

## 3. Understanding the Core Flow

### 3.1 How Messages Flow (Alice to Bob Example)

```
┌──────────────────────────────────────────────────────────────────────────┐
│                         Message Flow Diagram                             │
└──────────────────────────────────────────────────────────────────────────┘

    ALICE'S DEVICE                 SERVER                    BOB'S DEVICE
┌─────────────────┐          ┌──────────────┐          ┌─────────────────┐
│                 │          │              │          │                 │
│  [User List]    │          │  activeUsers │          │  [Chat Window]  │
│  • Bob (online) │◄─────────┤  Map:        │          │                 │
│                 │          │              │          │                 │
│  User clicks    │          │  alice123 →  │          │                 │
│  on "Bob"       │          │   socket_abc │          │                 │
│       │         │          │              │          │                 │
│       ▼         │          │  bob456 →    │         │                 │
│  Opens chat     │          │   socket_xyz │          │                 │
│  window for Bob │          │              │          │                 │
│       │         │          └──────┬───────┘          │                 │
│       │         │                 │                  │                 │
│       ▼         │                 │                 │                 │
│  Types message  │                 │                  │                 │
│  "Hello Bob"    │                 │                  │                 │
│       │         │                 │                  │                 │
│       ▼         │                 │                 │                 │
│  Clicks Send    │                 │                  │                 │
│       │         │                 │                  │                 │
│       ▼         │                 │                 │                 │
│                 │                 │                  │                 │
│  emit("send_direct_message", {    │                  │                 │
│    recipientId: "bob456",          │                 │                 │
│    content: "Hello Bob"            │                 │                 │
│  })             │                 │                  │                 │
│       │         │                 │                  │                 │
│       └─────────┼────────────────►│                 │                 │
│                 │                 │                  │                 │
│                 │     1. Identifies sender           │                 │
│                 │        socket.userId = alice123    │                 │
│                 │                 │                  │                 │
│                 │     2. Saves to MongoDB            │                 │
│                 │        Message.create({...})       │                 │
│                 │                 │                  │                 │
│                 │     3. Finds Bob's socket          │                 │
│                 │        activeUsers.get("bob456")   │                 │
│                 │        Returns: socket_xyz         │                 │
│                 │                 │                  │                 │
│                 │     4. Sends to Bob                │                 │
│                 │        io.to(socket_xyz)           │                 │
│                 │         .emit("new_message")       │                 │
│                 │                 │                  │                 │
│                 │                 └─────────────────►│                 │
│                 │                                    │  Receives msg   │
│                 │                                    │       │         │
│                 │                                    │       ▼         │
│  Callback confirms                                   │  Displays:      │
│  message sent   │◄───────────────────────────────────│  "Alice: Hello  │
│  Shows ✓        │   5. callback({success: true})     │   Bob"          │
│                 │                                    │                 │
└─────────────────┘                                    └─────────────────┘
```

### 3.2 The User-Socket Mapping Problem

**Question**: When Alice's message arrives at the server, how does it know it's from Alice?

**Answer**: Authentication during connection + in-memory tracking.

```
┌──────────────────────────────────────────────────────────────────────┐
│                    Connection Authentication Flow                    │
└──────────────────────────────────────────────────────────────────────┘

STEP 1: Alice opens chat app, connects with JWT token
┌─────────────────────────────────────────────────────────────────────┐
│  Frontend (Alice's browser)                                         │
│                                                                     │
│  const token = localStorage.getItem("token"); // From login         │
│  const socket = io("http://localhost:3000", {                       │
│    auth: { token: token }                                           │
│  });                                                                │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│  Server: Authentication Middleware (io.use)                         │
│                                                                      │
│  1. Extract token from socket.handshake.auth.token                  │
│  2. Verify JWT: jwt.verify(token, secret)                           │
│     Decoded: { id: "alice123", email: "alice@example.com" }         │
│  3. Fetch user from database: User.findById("alice123")             │
│  4. Attach to socket: socket.userId = "alice123"                    │
│                       socket.username = "Alice"                     │
│  5. Allow connection: next()                                        │
└────────────────────────────┬─────────────────────────────────────────┘
                             │
                             ▼
STEP 2: Server stores mapping in memory
┌─────────────────────────────────────────────────────────────────────┐
│  Server: activeUsers Map (in-memory)                                │
│                                                                     │
│  activeUsers.set(socket.userId, socket.id);                         │
│                                                                     │
│  Result:                                                            │
│  ┌────────────────────────────────┐                                 │
│  │ activeUsers Map:               │                                 │
│  │                                │                                 │
│  │ "alice123" → "socket_abc456"   │ ← Can now find Alice's socket   │
│  │ "bob456"   → "socket_xyz789"   │ ← Can now find Bob's socket     │
│  │ "charlie789" → "socket_def012" │                                 │
│  └────────────────────────────────┘                                 │
└─────────────────────────────────────────────────────────────────────┘

STEP 3: When Alice sends a message
┌─────────────────────────────────────────────────────────────────────┐
│  Message handler receives:                                          │
│                                                                     │
│  socket.on("send_direct_message", (data) => {                       │
│    const senderId = socket.userId;  // ← "alice123" (already known) │
│    const recipientId = data.recipientId;  // ← "bob456" (from data) │
│                                                                     │
│    // Find Bob's socket                                             │
│    const bobSocketId = activeUsers.get(recipientId);  // socket_xyz │
│                                                                     │
│    // Send directly to Bob                                          │
│    io.to(bobSocketId).emit("new_message", {...});                   │
│  });                                                                │
└─────────────────────────────────────────────────────────────────────┘
```

### 3.3 Understanding Rooms (For Group Chats)

**Concept**: A room is like a WhatsApp group - when someone sends a message, everyone in that group receives it.

```
┌──────────────────────────────────────────────────────────────────────┐
│                         Room-Based Group Chat                        │
└──────────────────────────────────────────────────────────────────────┘

UI: User sees list of groups
┌────────────────────────────────────────────────────────────┐
│  [Group List]                                              │
│                                                            │
│  📁 Team Project (3 members)  ← User clicks this group     │
│  📁 Family Chat  (5 members)                               │
│  📁 Study Group  (8 members)                               │
└────────────────────────────────────────────────────────────┘
         │
         │ Triggers: openGroup(conversationId: "conv_team_123")
         ▼
┌────────────────────────────────────────────────────────────┐
│  Frontend emits:                                           │
│                                                            │
│  socket.emit("join_group", "conv_team_123")                │
│          │                                                 │
│          │ conversationId stored in database               │
│          │ (from Conversation collection)                  │
└──────────┼─────────────────────────────────────────────────┘
           │
           ▼
┌────────────────────────────────────────────────────────────┐
│  Server: join_group handler                                │
│                                                            │
│  1. Verify user is a participant in this conversation      │
│     Conversation.findOne({                                 │
│       _id: "conv_team_123",                                │
│       participants: socket.userId  // Check membership     │
│     })                                                     │
│                                                            │
│  2. Join Socket.io room                                    │
│     socket.join("conv_team_123");                          │
│                                                            │
│  3. Now this socket is in the room                         │
└────────────────────────────────────────────────────────────┘

Server's Room Structure (managed by Socket.io):
┌────────────────────────────────────────────────────────────┐
│  Room: "conv_team_123"                                     │
│                                                            │
│  Members (sockets in this room):                           │
│  ┌──────────────────────────────────────┐                  │
│  │ • socket_abc456 (Alice's socket)     │                  │
│  │ • socket_xyz789 (Bob's socket)       │                  │
│  │ • socket_def012 (Charlie's socket)   │                  │
│  └──────────────────────────────────────┘                  │
└────────────────────────────────────────────────────────────┘

When Alice sends message to group:
┌────────────────────────────────────────────────────────────┐
│  Frontend:                                                 │
│  socket.emit("send_group_message", {                       │
│    conversationId: "conv_team_123",  ← Group identifier    │
│    content: "Team meeting at 3pm"                          │
│  })                                                        │
└──────────────────────────────┬─────────────────────────────┘
                               │
                               ▼
┌────────────────────────────────────────────────────────────┐
│  Server: send_group_message handler                        │
│                                                            │
│  io.to("conv_team_123").emit("new_group_message", {...})   │
│      │                                                     │
│      │ Socket.io automatically sends to ALL sockets        │
│      │ in room "conv_team_123"                             │
│      │                                                     │
│      ├──────► socket_abc456 (Alice) receives               │
│      ├──────► socket_xyz789 (Bob) receives                 │
│      └──────► socket_def012 (Charlie) receives             │
└────────────────────────────────────────────────────────────┘

Bob and Charlie's devices:
┌────────────────────────────────────────────────────────────┐
│  socket.on("new_group_message", (data) => {                │
│    displayMessage(data.message);  // Shows: "Alice: Team   │
│  });                                       meeting at 3pm" │
└────────────────────────────────────────────────────────────┘
```

**Key difference from direct messages**:
- **Direct**: Find specific user's socket → Send only to them
- **Group**: Broadcast to room → Socket.io sends to all members automatically

---

## 4. MongoDB Schemas (Define Data Structure First)

**Scenario**: You and your friend want to chat.

**With HTTP** (How websites normally work):
```
You: "Hey, do I have new messages?" [Request to server]
Server: "No" [Response]
[Connection closes]

[30 seconds pass]

You: "How about now?" [New request]
Server: "Yes, here's a message from your friend" [Response]
[Connection closes]
```

**The problem**: 
- You waste time asking "any updates?" repeatedly
- Messages arrive with delay
- Server handles thousands of unnecessary requests

**With WebSocket**:
```
You: "Connect me" [Opens connection]
Server: "Connected" [Connection stays open]

[Later, whenever a message arrives]

Server: "New message for you!" [Pushes immediately]
You: "Got it!"

[Connection remains open]
```

**Simple explanation**: HTTP is like mailing letters back and forth. WebSocket is like a phone call that stays connected.

**Technical explanation**: HTTP is request-response only—the client must initiate every interaction. WebSocket establishes a persistent, bidirectional TCP connection where both client and server can send data at any time without waiting for a request.

### 1.2 Mental Model: How Messages Flow

Let's trace a message from Alice to Bob:

```
┌─────────┐                                              ┌─────────┐
│  Alice  │                                              │   Bob   │
│ (User)  │                                              │ (User)  │
└────┬────┘                                              └────┬────┘
     │                                                        │
     │ 1. Types message                                      │
     │ "Hello Bob"                                           │
     │                                                        │
     │ 2. Sends via WebSocket                                │
     ├──────────────────────────────┐                        │
     │                              ▼                        │
     │                    ┌──────────────────┐               │
     │                    │   Server/Socket  │               │
     │                    │   Connection     │               │
     │                    └────────┬─────────┘               │
     │                             │                         │
     │                    3. Server identifies:              │
     │                       - Who sent it (Alice)           │
     │                       - Who to send to (Bob)          │
     │                             │                         │
     │                    4. Finds Bob's connection          │
     │                             │                         │
     │                             ├─────────────────────────┤
     │                             │                         │
     │                             ▼                         ▼
     │                    5. Sends to Bob          6. Bob receives
     │                                                "Hello Bob"
     │                                                        │
     │                    7. Saves to database                │
     │                       (for history)                    │
     ▼                             ▼                          ▼
[Waiting for reply]          [Message stored]         [Sees message]
```

**Key insight**: The server acts as a **router**—it knows which WebSocket connection belongs to which user, and forwards messages accordingly.

### 1.3 The User-Socket Mapping Problem

**Question**: When Alice's message arrives at the server, how does the server know it's from Alice?

**Answer**: Authentication during connection + tracking.

```
Step 1: Alice connects
┌─────────┐
│  Alice  │ ──── "Here's my JWT token" ───► [Server]
└─────────┘                                     │
                                                ▼
                                    Verify JWT → Get userId: "alice123"
                                                │
                                                ▼
                                    Store mapping:
                                    socketId: "xyz789" → userId: "alice123"

Step 2: Alice sends message
┌─────────┐
│  Alice  │ ──── Message via socket ───► [Server]
└─────────┘                                  │
                                             ▼
                              Look up: socketId "xyz789"
                              Found: userId "alice123"
                                             │
                                             ▼
                              Process message from Alice
```

**Data structure in server memory**:
```js
// Map of active connections
const connections = {
  "socket_xyz789": {
    userId: "alice123",
    username: "Alice",
    socketInstance: [WebSocket object]
  },
  "socket_abc456": {
    userId: "bob456",
    username: "Bob",
    socketInstance: [WebSocket object]
  }
}
```

### 1.4 Understanding Rooms (For Group Chats)

**Simple explanation**: A room is like a conference call. When you join a room, anything said in that room reaches everyone in it.

**How it works**:

```
Room: "Team Chat" (roomId: "room_001")

┌──────────┐    ┌──────────┐    ┌──────────┐
│  Alice   │    │   Bob    │    │  Charlie │
└────┬─────┘    └────┬─────┘    └────┬─────┘
     │               │               │
     └───────┬───────┴───────┬───────┘
             │               │
             ▼               ▼
     Both join "room_001"
             │
             ▼
    ┌────────────────────┐
    │   Server tracks:   │
    │                    │
    │   room_001 = {     │
    │     Alice,         │
    │     Bob,           │
    │     Charlie        │
    │   }                │
    └────────────────────┘
             │
             ▼
    When Alice sends message to room_001:
             │
             ├──────► Broadcast to Bob
             └──────► Broadcast to Charlie
```

**Key concept**: The server maintains a list of which users are in which rooms. When a message is sent to a room, the server loops through all members and sends to each.

### 1.5 File Handling Strategy

**Question**: Should files go through the WebSocket?

**Two approaches**:

**Approach A: HTTP Upload + WebSocket Notification** (Recommended)
```
Alice wants to send image.jpg to Bob

Step 1: Upload file
Alice ──── HTTP POST /upload ───► Server
                                    │
                                    ▼
                            Save file to storage
                            Return: { url: "/uploads/image.jpg" }
                                    │
                                    ▼
Alice ◄─── Response: file URL ──── Server

Step 2: Send message with file reference
Alice ──── WebSocket emit ───────► Server ──── WebSocket emit ──────► Bob
         "new_message"                          "receive_message"
         {                                      {
           text: "Check this out",                from: "Alice",
           fileUrl: "/uploads/image.jpg"          fileUrl: "...",
         }                                        text: "..."
                                                }
```

**Why this works**:
- Files can be large (videos, documents)
- HTTP handles large uploads better (progress tracking, resume capability)
- WebSocket just sends the URL (tiny, fast)
- File stored once, referenced many times

**Approach B: Direct WebSocket Transfer**
```
Alice sends small file (< 1MB)

Alice ──── WebSocket emit ──────► Server
         File as base64 data         │
                                     ▼
                              Save to storage
                              Get URL
                                     │
                                     ▼
                              Broadcast to Bob
                                     │
                                     ▼
Bob   ◄─── WebSocket receive ──── Server
         { fileUrl: "..." }
```

**Trade-offs**:

| Feature | HTTP Upload (A) | WebSocket Upload (B) |
|---------|----------------|---------------------|
| Best for | Large files (> 1MB) | Small files (< 1MB) |
| Progress tracking | Easy (built-in) | Manual implementation |
| Connection usage | Extra HTTP request | Uses existing socket |
| Complexity | Simple (standard upload) | More code needed |

**We'll use Approach A** because it's more practical for real applications.

---

## 5. Socket.io Configuration with Authentication

Before setting up Socket.io, we need to define our data structure.

### User Schema (with Presence Tracking)

**File**: `src/models/User.model.js`

```js
const mongoose = require("mongoose");

const userSchema = new mongoose.Schema({
  username: {
    type: String,
    required: true,
    unique: true,
    trim: true
  },
  email: {
    type: String,
    required: true,
    unique: true,
    lowercase: true
  },
  passwordHash: {
    type: String,
    required: true
  },
  avatar: {
    type: String,
    default: null
  },
  
  // Presence tracking fields (for online/offline status)
  isOnline: {
    type: Boolean,
    default: false
    // Updated when user connects/disconnects via Socket.io
  },
  lastSeen: {
    type: Date,
    default: Date.now
    // Updated when user disconnects
  },
  socketId: {
    type: String,
    default: null
    // Stores the current Socket.io connection ID
    // Used to send messages to specific users
  }
}, {
  timestamps: true // Adds createdAt and updatedAt
});

const User = mongoose.model("User", userSchema);
module.exports = User;
```

### Conversation Schema

**File**: `src/models/Conversation.model.js`

```js
const mongoose = require("mongoose");

const conversationSchema = new mongoose.Schema({
  type: {
    type: String,
    enum: ["direct", "group"],
    required: true
    // "direct" = one-to-one chat
    // "group" = multiple participants
  },
  
  participants: [{
    type: mongoose.Schema.Types.ObjectId,
    ref: "User",
    required: true
    // Array of user IDs in this conversation
    // For direct: exactly 2 users
    // For group: 3+ users
  }],
  
  // Group-specific fields (null for direct chats)
  name: {
    type: String,
    trim: true
    // Group name (e.g., "Team Chat", "Family")
  },
  avatar: {
    type: String
    // Group avatar image URL
  },
  admin: {
    type: mongoose.Schema.Types.ObjectId,
    ref: "User"
    // User who created the group (can add/remove members)
  },
  
  // Cached last message (for conversation list preview)
  lastMessage: {
    content: String,
    sender: {
      type: mongoose.Schema.Types.ObjectId,
      ref: "User"
    },
    timestamp: Date,
    messageType: {
      type: String,
      enum: ["text", "image", "video", "audio", "file"]
    }
    // Denormalized for performance (avoids querying messages table)
  },
  
  isActive: {
    type: Boolean,
    default: true
    // Soft delete flag
  }
}, {
  timestamps: true
});

// Index for fast lookups
conversationSchema.index({ participants: 1, type: 1 });

const Conversation = mongoose.model("Conversation", conversationSchema);
module.exports = Conversation;
```

### Message Schema

**File**: `src/models/Message.model.js`

```js
const mongoose = require("mongoose");

const messageSchema = new mongoose.Schema({
  conversationId: {
    type: mongoose.Schema.Types.ObjectId,
    ref: "Conversation",
    required: true,
    index: true
    // Links message to a conversation
    // Indexed for fast "get all messages in conversation" queries
  },
  
  sender: {
    type: mongoose.Schema.Types.ObjectId,
    ref: "User",
    required: true
    // Who sent this message
  },
  
  content: {
    type: String,
    trim: true
    // Message text (optional if it's a file-only message)
  },
  
  messageType: {
    type: String,
    enum: ["text", "image", "video", "audio", "file"],
    default: "text"
    // Determines how to display the message on frontend
  },
  
  // File metadata (only present if messageType is not "text")
  file: {
    url: String,          // Where the file is stored (e.g., "/uploads/abc.jpg")
    filename: String,     // Original filename
    size: Number,         // File size in bytes
    mimetype: String      // File type (e.g., "image/jpeg")
    // Note: We store the URL, not the actual file binary
  },
  
  // Delivery tracking
  status: {
    type: String,
    enum: ["sent", "delivered", "read"],
    default: "sent"
    // "sent" = saved to database
    // "delivered" = recipient's device received it
    // "read" = recipient opened and viewed it
  },
  
  // Read receipts (who has read this message and when)
  readBy: [{
    user: {
      type: mongoose.Schema.Types.ObjectId,
      ref: "User"
    },
    readAt: {
      type: Date,
      default: Date.now
    }
    // Array supports group chats where multiple people read the message
  }],
  
  // Deletion tracking
  isDeleted: {
    type: Boolean,
    default: false
    // "Delete for everyone" sets this to true
  },
  deletedFor: [{
    type: mongoose.Schema.Types.ObjectId,
    ref: "User"
    // "Delete for me" adds user ID here
    // Messages filtered out for these users when fetching
  }]
}, {
  timestamps: true // Adds createdAt (message sent time) and updatedAt
});

// Compound index for efficient queries
messageSchema.index({ conversationId: 1, createdAt: -1 });
// Allows fast "get messages in conversation, newest first" queries

const Message = mongoose.model("Message", messageSchema);
module.exports = Message;
```

---

## Part 3: Setting Up Socket.io with Authentication

### 3.1 Environment Configuration

**File**: `src/config/env.js`

```js
require("dotenv").config();

module.exports = {
  // Server
  nodeEnv: process.env.NODE_ENV || "development",
  port: process.env.PORT || 3000,
  
  // Database
  mongoUri: process.env.MONGO_URI || "mongodb://localhost:27017/chat-app",
  
  // JWT
  jwtSecret: process.env.JWT_SECRET,
  
  // Client
  clientUrl: process.env.CLIENT_URL || "http://localhost:5173"
};
```

**File**: `.env`

```bash
NODE_ENV=development
PORT=3000
MONGO_URI=mongodb://localhost:27017/chat-app
JWT_SECRET=your-super-secret-key-change-this-in-production
CLIENT_URL=http://localhost:5173
```

### 3.2 Database Connection

**File**: `src/config/database.js`

```js
const mongoose = require("mongoose");
const config = require("./env");

const connectDB = async () => {
  try {
    const conn = await mongoose.connect(config.mongoUri, {
      maxPoolSize: 10,              // Maximum 10 concurrent connections
      serverSelectionTimeoutMS: 5000, // Timeout after 5s if can't connect
      socketTimeoutMS: 45000         // Close sockets after 45s of inactivity
    });

    console.log(`MongoDB connected: ${conn.connection.host}`);
    
    // Handle connection events
    mongoose.connection.on("error", (err) => {
      console.error("MongoDB error:", err);
    });

    mongoose.connection.on("disconnected", () => {
      console.log("MongoDB disconnected");
    });

    // Graceful shutdown
    process.on("SIGINT", async () => {
      await mongoose.connection.close();
      console.log("MongoDB connection closed");
      process.exit(0);
    });

  } catch (error) {
    console.error("MongoDB connection error:", error);
    throw error; // Re-throw to stop server startup
  }
};

module.exports = connectDB;
```

### 3.3 Socket.io Configuration with Authentication

**File**: `src/config/socketio.js`

```js
const { Server } = require("socket.io");
const jwt = require("jsonwebtoken");
const config = require("./env");
const User = require("../models/User.model");
const handleDirectMessages = require("../socket/directMessages");
const handleGroupMessages = require("../socket/groupMessages");

function setupSocketIO(server) {
  // Create Socket.io instance attached to HTTP server
  const io = new Server(server, {
    cors: {
      origin: config.clientUrl,
      credentials: true
    }
  });

  // In-memory store: userId -> socketId mapping
  // This allows us to quickly find a user's socket to send them messages
  const activeUsers = new Map();
  // Example: Map { "user123" => "socket_abc456", "user789" => "socket_def789" }

  // ============================================
  // AUTHENTICATION MIDDLEWARE
  // ============================================
  // This runs BEFORE the "connection" event
  // If authentication fails, connection is rejected
  
  io.use(async (socket, next) => {
    try {
      // Extract JWT token from handshake
      // Frontend sends this when connecting: io("url", { auth: { token: "..." } })
      const token = socket.handshake.auth.token;
      
      if (!token) {
        return next(new Error("Authentication error: No token provided"));
      }

      // Verify JWT token
      const decoded = jwt.verify(token, config.jwtSecret);
      // decoded = { id: "user123", email: "...", iat: ..., exp: ... }
      
      // Fetch user from database to ensure they still exist
      const user = await User.findById(decoded.id);
      
      if (!user) {
        return next(new Error("Authentication error: User not found"));
      }

      // Attach user info to socket object
      // Now every event handler can access socket.userId and socket.username
      socket.userId = user._id.toString();
      socket.username = user.username;
      
      // Allow connection
      next();
      
    } catch (error) {
      // JWT verification failed or other error
      next(new Error(`Authentication error: ${error.message}`));
    }
  });

  // ============================================
  // CONNECTION EVENT
  // ============================================
  // Fires when a client successfully connects
  // At this point, authentication has passed
  
  io.on("connection", async (socket) => {
    console.log(`✓ ${socket.username} connected (socketId: ${socket.id})`);

    // Add user to active users mapping
    activeUsers.set(socket.userId, socket.id);
    // Now we can find this user's socket by their userId

    // Update user's online status in database
    await User.findByIdAndUpdate(socket.userId, {
      isOnline: true,
      socketId: socket.id  // Store socket ID for later use
    });

    // Broadcast to ALL connected clients that this user is now online
    io.emit("user_online", {
      userId: socket.userId,
      username: socket.username
    });
    // Note: io.emit() sends to everyone, including sender
    // socket.broadcast.emit() would send to everyone EXCEPT sender

    // Initialize event handlers from separate modules
    // These functions attach event listeners to this socket
    handleDirectMessages(io, socket, activeUsers);
    handleGroupMessages(io, socket);

    // ============================================
    // DISCONNECT EVENT
    // ============================================
    // Fires when client disconnects (closes browser, loses internet, etc.)
    
    socket.on("disconnect", async () => {
      console.log(`✗ ${socket.username} disconnected`);

      // Remove from active users
      activeUsers.delete(socket.userId);

      // Update database
      await User.findByIdAndUpdate(socket.userId, {
        isOnline: false,
        lastSeen: new Date(),
        socketId: null
      });

      // Broadcast offline status to all clients
      io.emit("user_offline", {
        userId: socket.userId
      });
    });
  });

  // Make activeUsers accessible to other modules
  // This allows message handlers to find user sockets
  io.activeUsers = activeUsers;

  return io;
}

module.exports = setupSocketIO;
```

### 3.4 Express App Setup

**File**: `src/app.js`

```js
const express = require("express");
const cors = require("cors");
const path = require("path");
const config = require("./config/env");

// Routes
const uploadRoutes = require("./routes/upload.routes");

const app = express();

// ============================================
// MIDDLEWARE
// ============================================

// CORS (allow frontend to make requests)
app.use(cors({
  origin: config.clientUrl,
  credentials: true
}));

// Body parsers
app.use(express.json());
app.use(express.urlencoded({ extended: true }));

// Serve uploaded files as static assets
// Files in /uploads folder will be accessible at /uploads/filename
app.use("/uploads", express.static(path.join(__dirname, "../uploads")));

// ============================================
// ROUTES
// ============================================

// Health check endpoint
app.get("/health", (req, res) => {
  res.json({ 
    status: "ok", 
    timestamp: new Date(),
    environment: config.nodeEnv
  });
});

// File upload endpoint
app.use("/api/upload", uploadRoutes);

// 404 handler
app.use("*", (req, res) => {
  res.status(404).json({
    success: false,
    error: "Route not found"
  });
});

module.exports = app;
```

### 3.5 Auth Middleware (for HTTP Routes)

**File**: `src/middlewares/auth.middleware.js`

```js
const jwt = require("jsonwebtoken");
const config = require("../config/env");
const User = require("../models/User.model");

// Protects HTTP routes (not Socket.io, that has its own auth)
// Usage: router.post("/upload", authMiddleware, uploadController)

async function authMiddleware(req, res, next) {
  try {
    // Extract token from Authorization header
    // Frontend sends: headers: { "Authorization": "Bearer <token>" }
    const authHeader = req.headers.authorization;
    
    if (!authHeader || !authHeader.startsWith("Bearer ")) {
      return res.status(401).json({
        success: false,
        error: "No token provided"
      });
    }

    const token = authHeader.split(" ")[1]; // Get token after "Bearer "

    // Verify token
    const decoded = jwt.verify(token, config.jwtSecret);
    
    // Fetch user
    const user = await User.findById(decoded.id);
    
    if (!user) {
      return res.status(401).json({
        success: false,
        error: "User not found"
      });
    }

    // Attach user to request object
    // Now route handlers can access req.user
    req.user = user;
    
    next(); // Proceed to route handler

  } catch (error) {
    return res.status(401).json({
      success: false,
      error: "Invalid token"
    });
  }
}

module.exports = authMiddleware;
```

---

## Part 2: Setting Up Socket.io

## 6. Client Connection

### 6.1 How Frontend Connects

**Installation**:
```bash
npm install socket.io-client
```

**Frontend code**:

```js
import { io } from "socket.io-client";

// Get JWT token (from login response, stored in localStorage)
const token = localStorage.getItem("token");

// Connect to Socket.io server
// The token is sent in the handshake and verified by io.use() middleware on server
const socket = io("http://localhost:3000", {
  auth: {
    token: token
  }
});

// ============================================
// CONNECTION EVENTS
// ============================================

socket.on("connect", () => {
  console.log("✓ Connected to server");
  console.log("My socket ID:", socket.id);
  // socket.id is auto-generated by Socket.io
  // Example: "abc123xyz"
});

socket.on("disconnect", () => {
  console.log("✗ Disconnected from server");
});

// Handle authentication errors
socket.on("connect_error", (error) => {
  console.error("Connection failed:", error.message);
  // This fires if JWT token is invalid or missing
  // Frontend should redirect to login page
});

// ============================================
// PRESENCE EVENTS
// ============================================

// Someone came online
socket.on("user_online", ({ userId, username }) => {
  console.log(`${username} is now online`);
  // Update UI: show green dot next to their name
});

// Someone went offline
socket.on("user_offline", ({ userId }) => {
  console.log(`User ${userId} went offline`);
  // Update UI: show gray dot, display "last seen" time
});
```

**Connection flow diagram**:

```
Frontend                           Backend
   │                                  │
   │  1. io("url", { auth: token })  │
   ├─────────────────────────────────►│
   │                                  │
   │            2. io.use() runs      │
   │            Verify JWT token      │
   │            Fetch user from DB    │
   │            Attach socket.userId  │
   │                                  │
   │  3. Connection allowed           │
   │◄─────────────────────────────────┤
   │                                  │
   socket.id assigned            socket stored in activeUsers
   emit("connect")               emit("connection", socket)
   │                                  │
   │  4. Broadcast "user_online"      │
   │◄─────────────────────────────────┤
   │                                  │
```

---

## 7. One-to-One Messaging (Direct Messages)

### 7.1 Message Flow Recap

```
Alice                    Server                      Bob
  │                        │                          │
  │ 1. Already connected   │    Already connected     │
  │    socket.userId =     │                          │
  │    "alice123"          │    socket.userId =       │
  │                        │    "bob456"              │
  │                        │                          │
  │                   activeUsers Map:                │
  │                   "alice123" → socket_abc         │
  │                   "bob456" → socket_xyz           │
  │                        │                          │
  │ 2. emit("send_direct_message")                    │
  │    { recipientId: "bob456", content: "Hi Bob" }  │
  ├──────────────────────►│                          │
  │                        │                          │
  │                   3. Save to database             │
  │                      Create Message doc           │
  │                        │                          │
  │                   4. Find Bob's socket            │
  │                      activeUsers.get("bob456")    │
  │                      = socket_xyz                 │
  │                        │                          │
  │                        │  5. emit("new_message")  │
  │                        ├────────────────────────► │
  │                        │                          │
  │  6. callback({success: true})                     │
  │◄──────────────────────┤                          │
  │                        │                          │
```

### 7.2 Implementation

**File**: `src/socket/directMessages.js`

```js
const Message = require("../models/Message.model");
const Conversation = require("../models/Conversation.model");
const User = require("../models/User.model");

function handleDirectMessages(io, socket, activeUsers) {
  
  // ============================================
  // SEND DIRECT MESSAGE
  // ============================================
  // Triggered when: User clicks send button in chat UI
  // Client emits: socket.emit("send_direct_message", { recipientId, content }, callback)
  
  socket.on("send_direct_message", async (data, callback) => {
    try {
      const { recipientId, content, file } = data;
      // recipientId: comes from UI when user selects who to chat with
      // content: message text from input field
      // file: optional, comes from file upload (covered later)
      
      const senderId = socket.userId;
      // socket.userId was attached during authentication (see socketio.js)
      // This is how we know WHO sent the message
      
      // ----------------------------------------
      // STEP 1: Find or create conversation
      // ----------------------------------------
      // UI Context: When user clicks on "Bob" in contact list,
      // frontend might first check if conversation exists via API,
      // or let backend handle it automatically (this approach)
      
      let conversation = await Conversation.findOne({
        type: "direct",
        participants: { 
          $all: [senderId, recipientId],  // Both users must be in array
          $size: 2                          // Array must have exactly 2 items
        }
      });

      if (!conversation) {
        // First message between these two users - create conversation
        conversation = await Conversation.create({
          type: "direct",
          participants: [senderId, recipientId]
        });
        // This conversation._id will be used for all future messages
      }

      // ----------------------------------------
      // STEP 2: Save message to database
      // ----------------------------------------
      const message = await Message.create({
        conversationId: conversation._id,  // Link to conversation
        sender: senderId,                   // From socket.userId
        content: content || "",             // Empty if file-only message
        messageType: file ? file.messageType : "text",
        file: file || undefined,            // File metadata (if uploaded)
        status: "sent"                      // Initial status
      });

      // Populate sender info for response (avoids extra query on frontend)
      await message.populate("sender", "username avatar");
      // Now message.sender = { _id, username, avatar } instead of just ID

      // ----------------------------------------
      // STEP 3: Update conversation's last message cache
      // ----------------------------------------
      // This allows showing message preview in conversation list UI
      conversation.lastMessage = {
        content: content || "File",          // Show "File" if no text
        sender: senderId,
        timestamp: message.createdAt,
        messageType: message.messageType
      };
      await conversation.save();

      // ----------------------------------------
      // STEP 4: Try to deliver to recipient (if online)
      // ----------------------------------------
      const recipientSocketId = activeUsers.get(recipientId);
      // activeUsers is the Map passed from socketio.js
      // It maps userId → socketId for all connected users
      // recipientSocketId will be null if user is offline
      
      if (recipientSocketId) {
        // Recipient is online - send immediately
        io.to(recipientSocketId).emit("new_message", {
          conversationId: conversation._id,
          message: message  // Includes populated sender info
        });
        // io.to(socketId) sends to ONLY that specific socket
        
        // Update status to delivered
        message.status = "delivered";
        await message.save();
      }
      // If recipient is offline, message stays as "sent"
      // They'll get it when they connect (handled in connection event)

      // ----------------------------------------
      // STEP 5: Acknowledge to sender
      // ----------------------------------------
      // The callback parameter sends response back to sender's emit()
      callback({
        success: true,
        message: message,
        status: message.status  // "sent" or "delivered"
      });
      // Frontend receives this in: socket.emit(..., (response) => {})

    } catch (error) {
      console.error("Error in send_direct_message:", error);
      callback({ 
        success: false, 
        error: error.message 
      });
    }
  });

  // ============================================
  // MARK MESSAGE AS READ
  // ============================================
  // Triggered when: User opens chat or scrolls to message
  // Client emits: socket.emit("mark_as_read", { messageId }, callback)
  
  socket.on("mark_as_read", async ({ messageId }, callback) => {
    try {
      const message = await Message.findById(messageId);
      // messageId comes from frontend (stored when message was displayed)
      
      if (!message) {
        return callback({ 
          success: false, 
          error: "Message not found" 
        });
      }

      // Check if already read by this user (avoid duplicates)
      const alreadyRead = message.readBy.some(
        r => r.user.toString() === socket.userId
        // socket.userId is current user (from authentication)
      );

      if (!alreadyRead) {
        // Add to readBy array
        message.readBy.push({
          user: socket.userId,
          readAt: new Date()
        });
        message.status = "read";
        await message.save();

        // ----------------------------------------
        // Notify sender about read receipt
        // ----------------------------------------
        const sender = await User.findById(message.sender);
        // message.sender is the user who originally sent this message
        
        if (sender && sender.isOnline && sender.socketId) {
          // Sender is currently online - notify them immediately
          io.to(sender.socketId).emit("message_read", {
            messageId: message._id,
            conversationId: message.conversationId,
            readBy: socket.userId,      // Who read it
            readAt: new Date()
          });
          // Frontend will show double checkmark (✓✓)
        }
        // If sender offline, they'll see "read" status when they fetch messages
      }

      callback({ success: true });

    } catch (error) {
      console.error("Error in mark_as_read:", error);
      callback({ 
        success: false, 
        error: error.message 
      });
    }
  });
}

module.exports = handleDirectMessages;
```

### 7.3 Client Usage

**UI Context**: User is viewing their chat app

```js
// ============================================
// SCENARIO 1: User selects contact to chat with
// ============================================

// User clicks on "Bob" in contact list
function openChatWith(user) {
  const recipientId = user._id;  // Bob's user ID from database
  const recipientName = user.username;  // "Bob"
  
  // Update UI to show chat window
  document.getElementById("chat-header").textContent = `Chat with ${recipientName}`;
  document.getElementById("recipient-id").value = recipientId;  // Store for sending
  
  // Load message history via HTTP API (not Socket.io)
  loadMessageHistory(recipientId);
}

// ============================================
// SCENARIO 2: User sends a message
// ============================================

// User types message and clicks send button
function sendMessage() {
  const input = document.getElementById("message-input");
  const messageText = input.value.trim();
  const recipientId = document.getElementById("recipient-id").value;  // From openChatWith
  
  if (!messageText) return;

  // Emit event to server
  socket.emit("send_direct_message", {
    recipientId: recipientId,  // Who to send to
    content: messageText
  }, (response) => {
    // This callback runs when server responds
    if (response.success) {
      console.log("✓ Message sent:", response.message);
      // Add message to UI with appropriate status indicator
      addMessageToUI(response.message, response.status);
      input.value = "";  // Clear input
    } else {
      console.error("✗ Failed to send:", response.error);
      // Show error to user
      alert("Failed to send message: " + response.error);
    }
  });
}

// ============================================
// SCENARIO 3: User receives a message
// ============================================

// Listen for incoming messages
socket.on("new_message", ({ conversationId, message }) => {
  console.log("📨 New message from:", message.sender.username);
  // message.sender was populated by server (includes username, avatar)
  
  const currentRecipientId = document.getElementById("recipient-id").value;
  
  // Check if message is for currently open conversation
  if (message.sender._id === currentRecipientId) {
    // Message is from the person we're currently chatting with
    displayMessage(message);
    
    // Mark as read automatically (since user is viewing it)
    socket.emit("mark_as_read", { 
      messageId: message._id 
    }, (response) => {
      if (response.success) {
        console.log("✓ Marked as read");
      }
    });
  } else {
    // Message from someone else - show notification
    showNotification(`New message from ${message.sender.username}`);
    incrementUnreadCount(message.sender._id);
  }
});

// ============================================
// SCENARIO 4: Sender gets read receipt
// ============================================

// Listen for read receipts (when someone reads your message)
socket.on("message_read", ({ messageId, readBy, readAt }) => {
  console.log(`✓✓ Message ${messageId} was read`);
  // Update UI: single checkmark (✓) → double blue checkmark (✓✓)
  const messageElement = document.getElementById(`message-${messageId}`);
  if (messageElement) {
    const statusIcon = messageElement.querySelector(".status");
    statusIcon.textContent = "✓✓";
    statusIcon.style.color = "blue";
  }
});
```

---

## 8. Group Chats with Rooms

### 4.1 The Flow We're Building

```
Alice                    Server                      Bob
  │                        │                          │
  │ 1. Connect             │                          │
  ├──────────────────────► │                          │
  │   + JWT token          │    2. Connect            │
  │                        │ ◄──────────────────────┤
  │                        │      + JWT token         │
  │                        │                          │
  │                   3. Store mappings:              │
  │                   alice → socket_abc              │
  │                   bob → socket_xyz                │
  │                        │                          │
  │ 4. Send message        │                          │
  │    to Bob              │                          │
  ├──────────────────────► │                          │
  │                        │                          │
  │                   5. Look up Bob's socket         │
  │                      (socket_xyz)                 │
  │                        │                          │
  │                        │  6. Forward message      │
  │                        ├────────────────────────► │
  │                        │                          │
  │                   7. Save to database             │
  │                        │                          │
```

### 4.2 Step 1: Authentication

**Why**: We need to know WHO is connecting so we can route messages correctly.

**Server**: `src/config/socketio.js`

```js
const { Server } = require("socket.io");
const jwt = require("jsonwebtoken");
const User = require("../models/User.model");

function setupSocketIO(server) {
  const io = new Server(server, {
    cors: {
      origin: process.env.CLIENT_URL,
      credentials: true
    }
  });

  // Authentication middleware (runs BEFORE connection)
  io.use(async (socket, next) => {
    try {
      // Extract token from handshake
      const token = socket.handshake.auth.token;
      
      if (!token) {
        return next(new Error("No token provided"));
      }

      // Verify token
      const decoded = jwt.verify(token, process.env.JWT_SECRET);
      
      // Fetch user from database
      const user = await User.findById(decoded.id);
      
      if (!user) {
        return next(new Error("User not found"));
      }

      // Attach user info to socket
      socket.userId = user._id.toString();
      socket.username = user.username;
      
      next(); // Allow connection
      
    } catch (error) {
      next(new Error("Authentication failed"));
    }
  });

  io.on("connection", (socket) => {
    console.log(`${socket.username} connected`);
    
    // Now socket.userId and socket.username are available
  });

  return io;
}

module.exports = setupSocketIO;
```

**What this does**:
- Before allowing connection, checks if token is valid
- If valid, attaches `userId` and `username` to socket
- If invalid, rejects connection
- Now every `socket` object knows which user it belongs to

**Client connection**:

```js
const token = localStorage.getItem("token");

const socket = io("http://localhost:3000", {
  auth: {
    token: token
  }
});
```

### 4.3 Step 2: Tracking Online Users

**Goal**: Know which socket belongs to which user.

**Server**: `src/config/socketio.js`

```js
function setupSocketIO(server) {
  const io = new Server(server, /* ... */);
  
  // Store active connections
  const activeUsers = new Map();
  // Structure: userId -> socketId
  // Example: "alice123" -> "socket_abc"

  io.use(/* authentication middleware */);

  io.on("connection", (socket) => {
    // Add user to active users
    activeUsers.set(socket.userId, socket.id);
    console.log(`${socket.username} is online`);
    console.log(`Active users: ${activeUsers.size}`);

    socket.on("disconnect", () => {
      // Remove user from active users
      activeUsers.delete(socket.userId);
      console.log(`${socket.username} went offline`);
      console.log(`Active users: ${activeUsers.size}`);
    });
  });

  // Make activeUsers accessible to other modules
  io.activeUsers = activeUsers;
  
  return io;
}
```

**Now we can find any user's socket**:
```js
const bobSocketId = activeUsers.get("bob123");
const bobSocket = io.sockets.sockets.get(bobSocketId);
```

### 4.4 Step 3: Sending Direct Messages

**General pattern**:

```js
// Client emits
socket.emit("send_direct_message", {
  recipientId: "userId_of_recipient",
  content: "message_text"
});

// Server handles
socket.on("send_direct_message", (data) => {
  const { recipientId, content } = data;
  const senderId = socket.userId;
  
  // 1. Find recipient's socket
  const recipientSocketId = activeUsers.get(recipientId);
  
  if (recipientSocketId) {
    // 2. Send to recipient
    io.to(recipientSocketId).emit("new_message", {
      from: senderId,
      content: content,
      timestamp: new Date()
    });
  }
  
  // 3. Save to database (for offline delivery)
});
```

**Complete implementation**:

**Server**: `src/socket/messageHandler.js`

```js
const Message = require("../models/Message.model");
const Conversation = require("../models/Conversation.model");

function handleDirectMessage(io, socket) {
  
  socket.on("send_direct_message", async (data, callback) => {
    try {
      const { recipientId, content } = data;
      const senderId = socket.userId;

      // Step 1: Find or create conversation
      let conversation = await Conversation.findOne({
        type: "direct",
        participants: { $all: [senderId, recipientId], $size: 2 }
      });

      if (!conversation) {
        conversation = await Conversation.create({
          type: "direct",
          participants: [senderId, recipientId]
        });
      }

      // Step 2: Save message to database
      const message = await Message.create({
        conversationId: conversation._id,
        sender: senderId,
        content: content,
        messageType: "text",
        status: "sent"
      });

      // Populate sender info for response
      await message.populate("sender", "username avatar");

      // Step 3: Try to deliver to recipient if online
      const recipientSocketId = io.activeUsers.get(recipientId);
      
      if (recipientSocketId) {
        io.to(recipientSocketId).emit("new_message", {
          conversationId: conversation._id,
          message: message
        });
        
        // Update status to delivered
        message.status = "delivered";
        await message.save();
      }

      // Step 4: Acknowledge to sender
      callback({
        success: true,
        message: message,
        status: message.status
      });

    } catch (error) {
      console.error("Error sending message:", error);
      callback({ success: false, error: error.message });
    }
  });
}

module.exports = handleDirectMessage;
```

**Client usage**:

```js
// Send message
function sendMessage(recipientId, text) {
  socket.emit("send_direct_message", {
    recipientId: recipientId,
    content: text
  }, (response) => {
    if (response.success) {
      console.log("Message sent:", response.message);
      // Add to UI
    } else {
      console.error("Failed:", response.error);
    }
  });
}

// Receive messages
socket.on("new_message", ({ conversationId, message }) => {
  console.log("New message from:", message.sender.username);
  console.log("Content:", message.content);
  // Add to UI
});
```

**Key points**:
- `callback` is used to confirm message was processed
- Message saved to DB even if recipient offline
- Status updated to "delivered" only if recipient online
- Sender gets immediate feedback

### 4.5 Step 4: Offline Message Delivery

**Problem**: Bob is offline when Alice sends a message. How does Bob get it later?

**Solution**: When Bob connects, fetch undelivered messages.

**Server**: Add to connection handler

```js
io.on("connection", async (socket) => {
  activeUsers.set(socket.userId, socket.id);
  
  // Fetch and deliver offline messages
  const undeliveredMessages = await Message.find({
    conversationId: { $in: userConversationIds },
    status: "sent",
    sender: { $ne: socket.userId }
  }).populate("sender", "username avatar");

  // Send each undelivered message
  for (const message of undeliveredMessages) {
    socket.emit("new_message", {
      conversationId: message.conversationId,
      message: message
    });
    
    // Update to delivered
    message.status = "delivered";
    await message.save();
  }
});
```

---

### 6.1 Room Concept Recap

**What happens when you join a room**:

```
Before joining:
Server knows: socket_abc belongs to Alice

After socket.join("room_team"):
Server knows: socket_abc belongs to Alice AND is in room "room_team"

Room membership is tracked per socket, not per user
If Alice has 2 devices connected, each socket joins independently
```

**Socket.io's room system**:

```js
// Internally, Socket.io maintains:
const rooms = {
  "room_team": new Set(["socket_abc", "socket_def", "socket_ghi"]),
  "room_family": new Set(["socket_abc", "socket_jkl"])
};

// When you do: io.to("room_team").emit("message", data)
// Socket.io sends to all sockets in that Set
```

### 6.2 Group Message Handler

**File**: `src/socket/groupMessages.js`

```js
const Message = require("../models/Message.model");
const Conversation = require("../models/Conversation.model");

function handleGroupMessages(io, socket) {
  
  // ============================================
  // JOIN GROUP
  // ============================================
  // Client emits: socket.emit("join_group", conversationId, callback)
  // Called when user opens a group chat
  
  socket.on("join_group", async (conversationId, callback) => {
    try {
      // ----------------------------------------
      // Verify user is a participant
      // ----------------------------------------
      const conversation = await Conversation.findOne({
        _id: conversationId,
        type: "group",
        participants: socket.userId  // Check if this user is in participants array
      });

      if (!conversation) {
        return callback({ 
          success: false, 
          error: "Not a member of this group" 
        });
      }

      // ----------------------------------------
      // Join the Socket.io room
      // ----------------------------------------
      socket.join(conversationId);
      // Now this socket will receive all messages sent to this room
      // Room name = conversation ID for easy mapping

      // ----------------------------------------
      // Notify others in the group
      // ----------------------------------------
      socket.to(conversationId).emit("user_joined_group", {
        userId: socket.userId,
        username: socket.username,
        conversationId: conversationId
      });
      // socket.to(roomId) sends to everyone in room EXCEPT sender
      // Others see "Alice joined the chat"

      callback({ success: true });

    } catch (error) {
      console.error("Error in join_group:", error);
      callback({ 
        success: false, 
        error: error.message 
      });
    }
  });

  // ============================================
  // LEAVE GROUP
  // ============================================
  // Client emits: socket.emit("leave_group", conversationId)
  // Called when user closes group chat or navigates away
  
  socket.on("leave_group", (conversationId) => {
    // Remove socket from room
    socket.leave(conversationId);
    
    // Notify others
    socket.to(conversationId).emit("user_left_group", {
      userId: socket.userId,
      username: socket.username,
      conversationId: conversationId
    });
  });

  // ============================================
  // SEND GROUP MESSAGE
  // ============================================
  // Client emits: socket.emit("send_group_message", { conversationId, content }, callback)
  
  socket.on("send_group_message", async (data, callback) => {
    try {
      const { conversationId, content, file } = data;

      // ----------------------------------------
      // Verify membership
      // ----------------------------------------
      const conversation = await Conversation.findOne({
        _id: conversationId,
        participants: socket.userId
      });

      if (!conversation) {
        return callback({ 
          success: false, 
          error: "Not a member of this group" 
        });
      }

      // ----------------------------------------
      // Save message to database
      // ----------------------------------------
      const message = await Message.create({
        conversationId: conversationId,
        sender: socket.userId,
        content: content || "",
        messageType: file ? file.messageType : "text",
        file: file || undefined,
        status: "sent"
      });

      await message.populate("sender", "username avatar");

      // ----------------------------------------
      // Update conversation's last message
      // ----------------------------------------
      conversation.lastMessage = {
        content: content || "File",
        sender: socket.userId,
        timestamp: message.createdAt,
        messageType: message.messageType
      };
      await conversation.save();

      // ----------------------------------------
      // Broadcast to ALL members in the room
      // ----------------------------------------
      io.to(conversationId).emit("new_group_message", {
        conversationId: conversationId,
        message: message
      });
      // io.to(roomId) sends to everyone in room, INCLUDING sender
      // This is different from direct messages where we target specific socket

      callback({ success: true, message: message });

    } catch (error) {
      console.error("Error in send_group_message:", error);
      callback({ 
        success: false, 
        error: error.message 
      });
    }
  });

  // ============================================
  // TYPING INDICATOR
  // ============================================
  // Shows "Alice is typing..." to others in group
  
  socket.on("typing_start", ({ conversationId }) => {
    // Send to everyone in room except sender
    socket.to(conversationId).emit("user_typing", {
      userId: socket.userId,
      username: socket.username,
      conversationId: conversationId
    });
  });

  socket.on("typing_stop", ({ conversationId }) => {
    socket.to(conversationId).emit("user_stopped_typing", {
      userId: socket.userId,
      conversationId: conversationId
    });
  });
}

module.exports = handleGroupMessages;
```

### 6.3 Client Usage

**Join a group chat**:

```js
// When user opens a group chat
function openGroupChat(conversationId) {
  // Join the room
  socket.emit("join_group", conversationId, (response) => {
    if (response.success) {
      console.log("✓ Joined group");
      // Load messages from API
      loadGroupMessages(conversationId);
    } else {
      console.error("✗ Failed to join:", response.error);
    }
  });
}

// When user navigates away
function closeGroupChat(conversationId) {
  socket.emit("leave_group", conversationId);
}
```

**Send group message**:

```js
function sendGroupMessage(conversationId, messageText) {
  socket.emit("send_group_message", {
    conversationId: conversationId,
    content: messageText
  }, (response) => {
    if (response.success) {
      console.log("✓ Group message sent");
      // Message already displayed via "new_group_message" event
    } else {
      console.error("✗ Failed:", response.error);
    }
  });
}
```

**Receive group messages**:

```js
// Listen for new group messages
socket.on("new_group_message", ({ conversationId, message }) => {
  console.log("📨 Group message from:", message.sender.username);
  
  // Check if this group is currently open
  if (conversationId === currentOpenConversation) {
    // Display message
    displayMessage(message);
    
    // Mark as read if sender is not current user
    if (message.sender._id !== myUserId) {
      socket.emit("mark_as_read", { messageId: message._id });
    }
  } else {
    // Show notification
    showNotification(`New message in ${getGroupName(conversationId)}`);
  }
});
```

**Typing indicators**:

```js
// On input field
const messageInput = document.getElementById("message-input");
let typingTimeout;

messageInput.addEventListener("input", () => {
  // User started typing
  socket.emit("typing_start", { conversationId: currentConversationId });
  
  // Clear previous timeout
  clearTimeout(typingTimeout);
  
  // After 1 second of no typing, send stop event
  typingTimeout = setTimeout(() => {
    socket.emit("typing_stop", { conversationId: currentConversationId });
  }, 1000);
});

// Listen for others typing
socket.on("user_typing", ({ username, conversationId }) => {
  if (conversationId === currentOpenConversation) {
    showTypingIndicator(`${username} is typing...`);
  }
});

socket.on("user_stopped_typing", ({ userId, conversationId }) => {
  if (conversationId === currentOpenConversation) {
    hideTypingIndicator(userId);
  }
});
```

**User join/leave notifications**:

```js
socket.on("user_joined_group", ({ username, conversationId }) => {
  if (conversationId === currentOpenConversation) {
    console.log(`${username} joined the chat`);
    // Show system message in chat
    displaySystemMessage(`${username} joined`);
  }
});

socket.on("user_left_group", ({ username, conversationId }) => {
  if (conversationId === currentOpenConversation) {
    console.log(`${username} left the chat`);
    displaySystemMessage(`${username} left`);
  }
});
```

---

## Part 7: File Handling

### 5.1 Understanding Rooms

**What is a room?**

```js
// Think of it like this:
const rooms = {
  "room_team": new Set(["socket_alice", "socket_bob", "socket_charlie"]),
  "room_family": new Set(["socket_alice", "socket_mom", "socket_dad"])
};

// When you broadcast to "room_team":
for (const socketId of rooms["room_team"]) {
  io.to(socketId).emit("message", data);
}
```

Socket.io manages this for you automatically.

### 5.2 Room Operations

**General syntax**:

```js
// Join a room
socket.join(roomId);

// Leave a room
socket.leave(roomId);

// Send to everyone in room (including sender)
io.to(roomId).emit("event", data);

// Send to everyone in room (except sender)
socket.to(roomId).emit("event", data);

// Check which rooms a socket is in
const rooms = Array.from(socket.rooms);
```

### 5.3 Group Message Flow

```
Alice, Bob, Charlie in room "team_chat"

┌─────────┐          ┌──────────┐          ┌───────────┐
│  Alice  │          │   Bob    │          │  Charlie  │
└────┬────┘          └────┬─────┘          └─────┬─────┘
     │                    │                       │
     │ All joined room "team_chat"                │
     │                    │                       │
     ├─────────────┐      │                       │
     │             ▼      │                       │
     │        [Server]    │                       │
     │         Room:      │                       │
     │       team_chat    │                       │
     │       ┌─────────┐  │                       │
     │       │ Alice   │  │                       │
     │       │ Bob     │  │                       │
     │       │ Charlie │  │                       │
     │       └─────────┘  │                       │
     │                    │                       │
     │ Alice sends        │                       │
     │ "Hello team!"      │                       │
     ├────────────────►[Server]                   │
     │                    │                       │
     │              Broadcast to room             │
     │                    │                       │
     │◄───────────────────┤                       │
     │                    ├──────────────────────►│
     │                    │                       │
   Alice sees        Bob sees              Charlie sees
   (optional)        message               message
```

### 5.4 Implementing Group Chats

**Server**: `src/socket/groupHandler.js`

```js
const Conversation = require("../models/Conversation.model");
const Message = require("../models/Message.model");

function handleGroupMessages(io, socket) {
  
  // Join a group conversation
  socket.on("join_group", async (conversationId, callback) => {
    try {
      // Verify user is participant
      const conversation = await Conversation.findOne({
        _id: conversationId,
        type: "group",
        participants: socket.userId
      });

      if (!conversation) {
        return callback({ success: false, error: "Not a member" });
      }

      // Join the Socket.io room
      socket.join(conversationId);

      // Notify others in the group
      socket.to(conversationId).emit("user_joined_group", {
        userId: socket.userId,
        username: socket.username,
        conversationId: conversationId
      });

      callback({ success: true });

    } catch (error) {
      callback({ success: false, error: error.message });
    }
  });

  // Leave a group
  socket.on("leave_group", (conversationId) => {
    socket.leave(conversationId);
    
    socket.to(conversationId).emit("user_left_group", {
      userId: socket.userId,
      username: socket.username,
      conversationId: conversationId
    });
  });

  // Send message to group
  socket.on("send_group_message", async (data, callback) => {
    try {
      const { conversationId, content } = data;

      // Verify membership
      const conversation = await Conversation.findOne({
        _id: conversationId,
        participants: socket.userId
      });

      if (!conversation) {
        return callback({ success: false, error: "Not a member" });
      }

      // Save message
      const message = await Message.create({
        conversationId: conversationId,
        sender: socket.userId,
        content: content,
        messageType: "text",
        status: "sent"
      });

      await message.populate("sender", "username avatar");

      // Broadcast to all in room (including sender)
      io.to(conversationId).emit("new_group_message", {
        conversationId: conversationId,
        message: message
      });

      callback({ success: true, message: message });

    } catch (error) {
      callback({ success: false, error: error.message });
    }
  });
}

module.exports = handleGroupMessages;
```

**Client usage**:

```js
// Join group when user opens group chat
function openGroupChat(conversationId) {
  socket.emit("join_group", conversationId, (response) => {
    if (response.success) {
      console.log("Joined group");
    }
  });
}

// Send group message
function sendGroupMessage(conversationId, text) {
  socket.emit("send_group_message", {
    conversationId: conversationId,
    content: text
  }, (response) => {
    if (response.success) {
      console.log("Message sent to group");
    }
  });
}

// Receive group messages
socket.on("new_group_message", ({ conversationId, message }) => {
  console.log("Group message:", message.content);
  // Add to UI
});

// When user is added to group
socket.on("user_joined_group", ({ username, conversationId }) => {
  console.log(`${username} joined the group`);
  // Show notification
});
```

**Key difference from direct messages**:
- Use `socket.join(roomId)` to join
- Broadcast with `io.to(roomId).emit()`
- Everyone in room receives (including sender)
- No need to loop through recipients manually

---

### 7.1 File Upload Strategy (HTTP + WebSocket)

**Why this approach**:
- HTTP handles large files better (built-in progress, chunking, resume)
- WebSocket just notifies about the file (tiny payload)
- File stored once, accessible to all recipients

**Complete flow**:

```
User selects file (image.jpg, 2MB)
     │
     ▼
1. Upload via HTTP POST /api/upload
   Headers: { "Authorization": "Bearer <token>" }
   Body: FormData with file
     │
     ▼
2. Server saves to /uploads folder
   Returns: { 
     url: "/uploads/1234-image.jpg",
     messageType: "image",
     filename: "image.jpg",
     size: 2048000
   }
     │
     ▼
3. Send WebSocket message with file metadata
   socket.emit("send_direct_message", {
     recipientId: "bob456",
     content: "Check this out!",
     file: { url, messageType, filename, size }
   })
     │
     ▼
4. Server saves message (content + file URL)
     │
     ▼
5. Recipient gets notification via WebSocket
   { message: { content: "...", file: { url: "/uploads/1234-image.jpg" } } }
     │
     ▼
6. Recipient's browser downloads image
   <img src="http://localhost:3000/uploads/1234-image.jpg" />
```

### 7.2 File Upload Route

**File**: `src/routes/upload.routes.js`

```js
const express = require("express");
const multer = require("multer");
const path = require("path");
const authMiddleware = require("../middlewares/auth.middleware");

const router = express.Router();

// ============================================
// CONFIGURE MULTER (File Upload Middleware)
// ============================================

// Configure storage
const storage = multer.diskStorage({
  destination: (req, file, cb) => {
    // Save files to /uploads folder
    cb(null, "uploads/");
  },
  filename: (req, file, cb) => {
    // Generate unique filename: timestamp-randomnumber-originalname
    const uniqueSuffix = `${Date.now()}-${Math.round(Math.random() * 1E9)}`;
    const ext = path.extname(file.originalname);
    const nameWithoutExt = path.basename(file.originalname, ext);
    cb(null, `${uniqueSuffix}-${nameWithoutExt}${ext}`);
  }
});

// File filter (what file types to allow)
const fileFilter = (req, file, cb) => {
  const allowedTypes = /jpeg|jpg|png|gif|pdf|doc|docx|mp4|mp3|wav/;
  const ext = path.extname(file.originalname).toLowerCase();
  const isValidExt = allowedTypes.test(ext.substring(1)); // Remove leading dot
  const isValidMime = allowedTypes.test(file.mimetype);

  if (isValidExt && isValidMime) {
    cb(null, true); // Accept file
  } else {
    cb(new Error("Invalid file type. Only images, documents, and media allowed."));
  }
};

// Create multer instance
const upload = multer({
  storage: storage,
  limits: {
    fileSize: 50 * 1024 * 1024 // 50MB max file size
  },
  fileFilter: fileFilter
});

// ============================================
// UPLOAD ENDPOINT
// ============================================
// POST /api/upload
// Headers: Authorization: Bearer <token>
// Body: FormData with "file" field

router.post("/", authMiddleware, upload.single("file"), (req, res) => {
  try {
    // Check if file was uploaded
    if (!req.file) {
      return res.status(400).json({
        success: false,
        error: "No file uploaded"
      });
    }

    // Build file URL (accessible via Express static middleware)
    const fileUrl = `/uploads/${req.file.filename}`;
    
    // Determine message type based on mimetype
    let messageType = "file"; // Default
    if (req.file.mimetype.startsWith("image/")) {
      messageType = "image";
    } else if (req.file.mimetype.startsWith("video/")) {
      messageType = "video";
    } else if (req.file.mimetype.startsWith("audio/")) {
      messageType = "audio";
    }

    // Return file metadata to client
    res.json({
      success: true,
      file: {
        url: fileUrl,                      // Where to access the file
        filename: req.file.originalname,   // Original name
        size: req.file.size,               // Size in bytes
        mimetype: req.file.mimetype,       // MIME type
        messageType: messageType           // For UI rendering
      }
    });

  } catch (error) {
    console.error("Upload error:", error);
    res.status(500).json({
      success: false,
      error: error.message
    });
  }
});

module.exports = router;
```

### 7.3 Sending File Messages

The message handlers (`directMessages.js` and `groupMessages.js`) already support files. The `file` parameter is optional:

```js
// In send_direct_message handler:
const message = await Message.create({
  conversationId: conversation._id,
  sender: senderId,
  content: content || "",              // Optional text with file
  messageType: file ? file.messageType : "text",
  file: file ? {
    url: file.url,
    filename: file.filename,
    size: file.size,
    mimetype: file.mimetype
  } : undefined,
  status: "sent"
});
```

### 7.4 Client File Upload

**HTML**:

```html
<input type="file" id="file-input" accept="image/*,video/*,audio/*,.pdf,.doc,.docx" />
<button onclick="sendFileMessage()">Send File</button>
```

**JavaScript**:

```js
async function sendFileMessage() {
  const fileInput = document.getElementById("file-input");
  const file = fileInput.files[0];
  
  if (!file) {
    alert("Please select a file");
    return;
  }

  try {
    // ----------------------------------------
    // STEP 1: Upload file via HTTP
    // ----------------------------------------
    const formData = new FormData();
    formData.append("file", file);

    const token = localStorage.getItem("token");
    
    const uploadResponse = await fetch("http://localhost:3000/api/upload", {
      method: "POST",
      headers: {
        "Authorization": `Bearer ${token}`
      },
      body: formData
    });

    const uploadData = await uploadResponse.json();

    if (!uploadData.success) {
      throw new Error(uploadData.error);
    }

    console.log("✓ File uploaded:", uploadData.file.url);

    // ----------------------------------------
    // STEP 2: Send message via WebSocket
    // ----------------------------------------
    const recipientId = getCurrentRecipientId();
    
    socket.emit("send_direct_message", {
      recipientId: recipientId,
      content: "", // Optional: add text with file
      file: uploadData.file // File metadata from upload response
    }, (response) => {
      if (response.success) {
        console.log("✓ File message sent");
        // Clear file input
        fileInput.value = "";
      } else {
        console.error("✗ Failed to send:", response.error);
      }
    });

  } catch (error) {
    console.error("Error sending file:", error);
    alert("Failed to send file: " + error.message);
  }
}
```

**With progress tracking**:

```js
async function sendFileMessageWithProgress() {
  const fileInput = document.getElementById("file-input");
  const file = fileInput.files[0];
  const progressBar = document.getElementById("upload-progress");
  
  if (!file) return;

  try {
    const formData = new FormData();
    formData.append("file", file);

    const token = localStorage.getItem("token");
    
    // Use XMLHttpRequest for progress tracking
    const xhr = new XMLHttpRequest();
    
    // Track upload progress
    xhr.upload.addEventListener("progress", (e) => {
      if (e.lengthComputable) {
        const percentComplete = (e.loaded / e.total) * 100;
        progressBar.style.width = percentComplete + "%";
        progressBar.textContent = Math.round(percentComplete) + "%";
      }
    });
    
    xhr.addEventListener("load", () => {
      if (xhr.status === 200) {
        const uploadData = JSON.parse(xhr.responseText);
        
        // Send message via WebSocket
        socket.emit("send_direct_message", {
          recipientId: getCurrentRecipientId(),
          content: "",
          file: uploadData.file
        }, (response) => {
          if (response.success) {
            console.log("✓ File message sent");
            progressBar.style.width = "0%";
          }
        });
      } else {
        console.error("Upload failed");
      }
    });
    
    xhr.open("POST", "http://localhost:3000/api/upload");
    xhr.setRequestHeader("Authorization", `Bearer ${token}`);
    xhr.send(formData);

  } catch (error) {
    console.error("Error:", error);
  }
}
```

### 7.5 Displaying Files

**Receive file message**:

```js
socket.on("new_message", ({ conversationId, message }) => {
  // Check message type
  if (message.messageType === "text") {
    // Regular text message
    displayTextMessage(message);
  } else {
    // File message
    displayFileMessage(message);
  }
});

function displayFileMessage(message) {
  const messageDiv = document.createElement("div");
  messageDiv.className = "message";
  
  // Show sender and text (if any)
  let html = `<strong>${message.sender.username}:</strong> ${message.content}<br>`;
  
  // Display file based on type
  if (message.messageType === "image") {
    html += `<img src="${message.file.url}" alt="${message.file.filename}" 
             style="max-width: 300px; cursor: pointer;" 
             onclick="openImageModal('${message.file.url}')" />`;
  } else if (message.messageType === "video") {
    html += `<video controls style="max-width: 300px;">
               <source src="${message.file.url}" type="${message.file.mimetype}">
             </video>`;
  } else if (message.messageType === "audio") {
    html += `<audio controls>
               <source src="${message.file.url}" type="${message.file.mimetype}">
             </audio>`;
  } else {
    // Generic file (PDF, DOC, etc.)
    const fileSize = formatFileSize(message.file.size);
    html += `<a href="${message.file.url}" download="${message.file.filename}" 
             class="file-download">
               📎 ${message.file.filename} (${fileSize})
             </a>`;
  }
  
  messageDiv.innerHTML = html;
  document.getElementById("messages").appendChild(messageDiv);
}

function formatFileSize(bytes) {
  if (bytes < 1024) return bytes + " B";
  if (bytes < 1024 * 1024) return (bytes / 1024).toFixed(1) + " KB";
  return (bytes / (1024 * 1024)).toFixed(1) + " MB";
}
```

---

## Part 8: Complete Working Example

### 6.1 The Strategy

We'll use **HTTP upload + WebSocket notification**:

```
1. User selects file
2. Upload via HTTP POST to /api/upload
3. Server saves file, returns URL
4. User sends message via WebSocket with file URL
5. Recipient receives message with file URL
```

### 6.2 File Upload Endpoint

**Server**: `src/routes/upload.routes.js`

```js
const express = require("express");
const multer = require("multer");
const path = require("path");
const authMiddleware = require("../middlewares/auth.middleware");

const router = express.Router();

// Configure storage
const storage = multer.diskStorage({
  destination: (req, file, cb) => {
    cb(null, "uploads/");
  },
  filename: (req, file, cb) => {
    const uniqueName = `${Date.now()}-${file.originalname}`;
    cb(null, uniqueName);
  }
});

// File filter
const fileFilter = (req, file, cb) => {
  const allowedTypes = /jpeg|jpg|png|gif|pdf|doc|docx|mp4|mp3/;
  const ext = path.extname(file.originalname).toLowerCase();
  const isValid = allowedTypes.test(ext.substring(1));
  
  if (isValid) {
    cb(null, true);
  } else {
    cb(new Error("Invalid file type"));
  }
};

const upload = multer({
  storage: storage,
  limits: { fileSize: 50 * 1024 * 1024 }, // 50MB
  fileFilter: fileFilter
});

// Upload route
router.post("/", authMiddleware, upload.single("file"), (req, res) => {
  try {
    if (!req.file) {
      return res.status(400).json({
        success: false,
        error: "No file uploaded"
      });
    }

    const fileUrl = `/uploads/${req.file.filename}`;
    
    // Determine message type
    let messageType = "file";
    if (req.file.mimetype.startsWith("image/")) {
      messageType = "image";
    } else if (req.file.mimetype.startsWith("video/")) {
      messageType = "video";
    } else if (req.file.mimetype.startsWith("audio/")) {
      messageType = "audio";
    }

    res.json({
      success: true,
      file: {
        url: fileUrl,
        filename: req.file.originalname,
        size: req.file.size,
        mimetype: req.file.mimetype,
        messageType: messageType
      }
    });

  } catch (error) {
    res.status(500).json({
      success: false,
      error: error.message
    });
  }
});

module.exports = router;
```

**Server**: Mount in `src/app.js`

```js
const express = require("express");
const path = require("path");
const uploadRoutes = require("./routes/upload.routes");

const app = express();

app.use(express.json());

// Serve uploaded files
app.use("/uploads", express.static(path.join(__dirname, "../uploads")));

// Upload route
app.use("/api/upload", uploadRoutes);

module.exports = app;
```

### 6.3 Sending File Messages

**Modify message handlers** to accept file data:

```js
socket.on("send_direct_message", async (data, callback) => {
  try {
    const { recipientId, content, file } = data;
    // file = { url, filename, size, mimetype, messageType }

    const message = await Message.create({
      conversationId: conversation._id,
      sender: socket.userId,
      content: content || "",
      messageType: file ? file.messageType : "text",
      file: file ? {
        url: file.url,
        filename: file.filename,
        size: file.size,
        mimetype: file.mimetype
      } : undefined,
      status: "sent"
    });

    // Rest same as before...
  } catch (error) {
    callback({ success: false, error: error.message });
  }
});
```

### 6.4 Client File Upload Flow

```js
async function sendFileMessage(recipientId, file, text) {
  try {
    // Step 1: Upload file via HTTP
    const formData = new FormData();
    formData.append("file", file);

    const uploadResponse = await fetch("http://localhost:3000/api/upload", {
      method: "POST",
      headers: {
        "Authorization": `Bearer ${token}`
      },
      body: formData
    });

    const uploadData = await uploadResponse.json();

    if (!uploadData.success) {
      throw new Error("Upload failed");
    }

    // Step 2: Send message via WebSocket
    socket.emit("send_direct_message", {
      recipientId: recipientId,
      content: text,
      file: uploadData.file
    }, (response) => {
      if (response.success) {
        console.log("File message sent");
      }
    });

  } catch (error) {
    console.error("Error sending file:", error);
  }
}

// Usage
const fileInput = document.getElementById("file-input");
fileInput.addEventListener("change", () => {
  const file = fileInput.files[0];
  sendFileMessage("recipient_id", file, "Check this out!");
});
```

**Flow visualization**:

```
User selects file
     │
     ▼
Upload via HTTP POST
     │
     ▼
Server saves file
Returns: { url: "/uploads/abc.jpg", messageType: "image", ... }
     │
     ▼
Send WebSocket message
{
  recipientId: "bob123",
  content: "Check this out",
  file: { url, filename, size, mimetype, messageType }
}
     │
     ▼
Server saves message to DB
     │
     ▼
Server forwards to recipient
     │
     ▼
Recipient receives
{
  from: "alice123",
  content: "Check this out",
  file: { url: "/uploads/abc.jpg", messageType: "image" }
}
     │
     ▼
Recipient displays image
<img src="/uploads/abc.jpg" />
```

---

### 8.1 Complete Client Implementation

**File**: `client/index.html`

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Chat App</title>
  <script src="https://cdn.socket.io/4.5.4/socket.io.min.js"></script>
  <style>
    body { font-family: Arial, sans-serif; max-width: 800px; margin: 20px auto; }
    #messages { border: 1px solid #ccc; height: 400px; overflow-y: auto; padding: 10px; margin-bottom: 10px; }
    .message { margin-bottom: 10px; padding: 8px; background: #f0f0f0; border-radius: 5px; }
    .message.sent { background: #dcf8c6; text-align: right; }
    .system-message { color: #888; font-style: italic; text-align: center; }
    #input-area { display: flex; gap: 5px; }
    input[type="text"] { flex: 1; padding: 8px; }
    button { padding: 8px 15px; cursor: pointer; }
    .typing-indicator { color: #888; font-style: italic; min-height: 20px; }
    .online-users { border: 1px solid #ccc; padding: 10px; margin-bottom: 10px; }
    .user-online { color: green; }
    .user-offline { color: gray; }
  </style>
</head>
<body>
  <h1>Chat App</h1>
  
  <!-- Online Users -->
  <div class="online-users">
    <strong>Online Users:</strong>
    <div id="online-users-list"></div>
  </div>
  
  <!-- Messages -->
  <div id="messages"></div>
  <div class="typing-indicator" id="typing-indicator"></div>
  
  <!-- Input -->
  <div id="input-area">
    <input type="text" id="message-input" placeholder="Type a message..." />
    <input type="file" id="file-input" style="display:none" />
    <button onclick="document.getElementById('file-input').click()">📎</button>
    <button onclick="sendMessage()">Send</button>
  </div>

  <script>
    // ============================================
    // CONFIGURATION
    // ============================================
    
    // In real app, get these from login response
    const myUserId = "YOUR_USER_ID";
    const token = "YOUR_JWT_TOKEN"; // From localStorage.getItem("token")
    const recipientId = "RECIPIENT_USER_ID"; // In real app, selected from user list
    const conversationId = "CONVERSATION_ID"; // From API or Socket.io
    
    // ============================================
    // SOCKET.IO CONNECTION
    // ============================================
    
    const socket = io("http://localhost:3000", {
      auth: { token: token }
    });

    socket.on("connect", () => {
      console.log("✓ Connected to server");
      console.log("Socket ID:", socket.id);
      
      // Join group if this is a group chat
      // socket.emit("join_group", conversationId, (response) => {
      //   console.log("Joined group:", response);
      // });
    });

    socket.on("disconnect", () => {
      console.log("✗ Disconnected");
    });

    socket.on("connect_error", (error) => {
      console.error("Connection error:", error.message);
      alert("Authentication failed. Please log in again.");
    });

    // ============================================
    // PRESENCE EVENTS
    // ============================================
    
    socket.on("user_online", ({ userId, username }) => {
      console.log(`${username} is online`);
      updateUserStatus(userId, "online", username);
    });

    socket.on("user_offline", ({ userId }) => {
      console.log(`User ${userId} went offline`);
      updateUserStatus(userId, "offline");
    });

    function updateUserStatus(userId, status, username = "User") {
      const usersList = document.getElementById("online-users-list");
      let userElement = document.getElementById(`user-${userId}`);
      
      if (status === "online") {
        if (!userElement) {
          userElement = document.createElement("div");
          userElement.id = `user-${userId}`;
          usersList.appendChild(userElement);
        }
        userElement.className = "user-online";
        userElement.textContent = `${username} (online)`;
      } else {
        if (userElement) {
          userElement.className = "user-offline";
          userElement.textContent = `${username} (offline)`;
        }
      }
    }

    // ============================================
    // MESSAGING
    // ============================================
    
    // Send text message
    function sendMessage() {
      const input = document.getElementById("message-input");
      const text = input.value.trim();
      
      if (!text) return;

      socket.emit("send_direct_message", {
        recipientId: recipientId,
        content: text
      }, (response) => {
        if (response.success) {
          displayMessage(response.message, true); // true = sent by me
          input.value = "";
        } else {
          alert("Failed to send: " + response.error);
        }
      });
    }

    // Enter key to send
    document.getElementById("message-input").addEventListener("keypress", (e) => {
      if (e.key === "Enter") {
        sendMessage();
      }
    });

    // Receive messages
    socket.on("new_message", ({ conversationId, message }) => {
      displayMessage(message, false); // false = received
      
      // Auto mark as read
      socket.emit("mark_as_read", { messageId: message._id });
    });

    socket.on("new_group_message", ({ conversationId, message }) => {
      const isMine = message.sender._id === myUserId;
      displayMessage(message, isMine);
      
      if (!isMine) {
        socket.emit("mark_as_read", { messageId: message._id });
      }
    });

    // Display message in UI
    function displayMessage(message, isSentByMe) {
      const messagesDiv = document.getElementById("messages");
      const messageDiv = document.createElement("div");
      messageDiv.className = "message" + (isSentByMe ? " sent" : "");
      messageDiv.id = `message-${message._id}`;
      
      let html = "";
      
      if (!isSentByMe) {
        html += `<strong>${message.sender.username}:</strong> `;
      }
      
      html += message.content;
      
      // Display file if present
      if (message.file) {
        html += "<br>";
        if (message.messageType === "image") {
          html += `<img src="${message.file.url}" style="max-width: 200px;" />`;
        } else {
          html += `<a href="${message.file.url}" target="_blank">📎 ${message.file.filename}</a>`;
        }
      }
      
      // Show status for sent messages
      if (isSentByMe) {
        html += ` <span class="status" id="status-${message._id}">✓</span>`;
      }
      
      messageDiv.innerHTML = html;
      messagesDiv.appendChild(messageDiv);
      messagesDiv.scrollTop = messagesDiv.scrollHeight; // Auto scroll to bottom
    }

    // Read receipts
    socket.on("message_read", ({ messageId }) => {
      const statusSpan = document.getElementById(`status-${messageId}`);
      if (statusSpan) {
        statusSpan.textContent = "✓✓"; // Double checkmark
        statusSpan.style.color = "blue";
      }
    });

    // ============================================
    // FILE UPLOAD
    // ============================================
    
    document.getElementById("file-input").addEventListener("change", async () => {
      const fileInput = document.getElementById("file-input");
      const file = fileInput.files[0];
      
      if (!file) return;

      try {
        // Upload file
        const formData = new FormData();
        formData.append("file", file);

        const response = await fetch("http://localhost:3000/api/upload", {
          method: "POST",
          headers: {
            "Authorization": `Bearer ${token}`
          },
          body: formData
        });

        const data = await response.json();

        if (data.success) {
          // Send message with file
          socket.emit("send_direct_message", {
            recipientId: recipientId,
            content: "",
            file: data.file
          }, (response) => {
            if (response.success) {
              displayMessage(response.message, true);
              fileInput.value = "";
            }
          });
        } else {
          alert("Upload failed: " + data.error);
        }

      } catch (error) {
        alert("Error uploading file: " + error.message);
      }
    });

    // ============================================
    // TYPING INDICATOR
    // ============================================
    
    let typingTimeout;
    const messageInput = document.getElementById("message-input");

    messageInput.addEventListener("input", () => {
      socket.emit("typing_start", { conversationId });
      
      clearTimeout(typingTimeout);
      typingTimeout = setTimeout(() => {
        socket.emit("typing_stop", { conversationId });
      }, 1000);
    });

    socket.on("user_typing", ({ username }) => {
      document.getElementById("typing-indicator").textContent = 
        `${username} is typing...`;
    });

    socket.on("user_stopped_typing", () => {
      document.getElementById("typing-indicator").textContent = "";
    });

    // ============================================
    // GROUP EVENTS
    // ============================================
    
    socket.on("user_joined_group", ({ username }) => {
      const messagesDiv = document.getElementById("messages");
      const systemMsg = document.createElement("div");
      systemMsg.className = "system-message";
      systemMsg.textContent = `${username} joined the chat`;
      messagesDiv.appendChild(systemMsg);
    });

    socket.on("user_left_group", ({ username }) => {
      const messagesDiv = document.getElementById("messages");
      const systemMsg = document.createElement("div");
      systemMsg.className = "system-message";
      systemMsg.textContent = `${username} left the chat`;
      messagesDiv.appendChild(systemMsg);
    });
  </script>
</body>
</html>
```

---

## Part 9: Key Concepts Reinforced

### 9.1 Complete Connection Lifecycle

```
Application Startup
     │
     ▼
1. Connect to MongoDB
   mongoose.connect()
     │
     ▼
2. Create HTTP Server
   http.createServer(app)
     │
     ▼
3. Attach Socket.io
   new Server(httpServer)
     │
     ▼
4. Start listening
   server.listen(3000)
     │
     ▼
Server ready
     │
     ├─────────────────────────────────┐
     │                                 │
     ▼                                 ▼
Client connects              MongoDB is ready
     │                       for queries
     ▼
io.use() runs
Verify JWT token
Fetch user from DB
     │
     ├─ Valid ──────►  Connection allowed
     │                      │
     │                      ▼
     │                 socket.userId = user._id
     │                 activeUsers.set(userId, socketId)
     │                 User.update({ isOnline: true })
     │                 io.emit("user_online")
     │                      │
     │                      ▼
     │                 Ready to send/receive messages
     │
     └─ Invalid ────► Connection rejected
                      Client sees connect_error
```

### 9.2 Message Routing Logic

```
Message arrives at server
     │
     ▼
Extract info from socket object:
- socket.userId (who sent it)
- data.recipientId (who to send to)
- data.content (message text)
     │
     ▼
Save to MongoDB
conversationId, sender, content, timestamp
     │
     ▼
Look up recipient in activeUsers Map
activeUsers.get(recipientId)
     │
     ├─ Found (online) ─────►  Send immediately
     │                         io.to(socketId).emit("new_message")
     │                         Update status = "delivered"
     │
     └─ Not found (offline) ──► Keep status = "sent"
                                Deliver when they connect
```

### 9.3 Room Broadcasting (Groups)

```
User sends message to group
socket.emit("send_group_message", { conversationId, content })
     │
     ▼
Server receives on socket
     │
     ▼
Verify user is member
Conversation.findOne({ _id, participants: socket.userId })
     │
     ▼
Save message to database
     │
     ▼
Broadcast to room
io.to(conversationId).emit("new_group_message", message)
     │
     ▼
Socket.io's internal room system:
rooms.get(conversationId) = Set ["socket_abc", "socket_def", "socket_ghi"]
     │
     ├──────►  Send to socket_abc
     ├──────►  Send to socket_def
     └──────►  Send to socket_ghi
```

### 9.4 Authentication Flow

```
Client Side                          Server Side
     │                                    │
localStorage.getItem("token")            │
     │                                    │
     ▼                                    │
io("url", { auth: { token } })           │
     ├───────────────────────────────────►│
     │                                    ▼
     │                              io.use((socket, next))
     │                                    │
     │                              token = socket.handshake.auth.token
     │                                    │
     │                              jwt.verify(token, secret)
     │                                    │
     │                              User.findById(decoded.id)
     │                                    │
     │                              socket.userId = user._id
     │                              socket.username = user.username
     │                                    │
     │                              next() ─► Allow connection
     │                                    │
     │◄───────────────────────────────────┤
     │         Connection successful       │
     ▼                                    ▼
socket.on("connect")              io.on("connection", socket)
```

### 9.5 File Upload Flow

```
User selects file
     │
     ▼
1. HTTP POST /api/upload
   Authorization: Bearer <token>
   Body: FormData with file
     │
     ▼
2. authMiddleware
   Verify JWT
   req.user = user
     │
     ▼
3. multer middleware
   Check file type
   Check file size
   Save to /uploads folder
     │
     ▼
4. Return file metadata
   { url, filename, size, messageType }
     │
     ▼
5. Client receives URL
     │
     ▼
6. Send WebSocket message
   socket.emit("send_direct_message", {
     recipientId,
     content: "text",
     file: { url, filename, ... }
   })
     │
     ▼
7. Server saves message
   Message.create({
     content,
     file: { url, filename, size },
     messageType: "image"
   })
     │
     ▼
8. Recipient receives
   socket.on("new_message")
   Display: <img src="url" />
```

### 9.6 User-Socket Mapping

```
In-Memory Store: activeUsers Map

When Alice connects:
┌─────────────────────────────────┐
│ activeUsers                     │
│                                 │
│ "alice123" → "socket_abc456"    │
└─────────────────────────────────┘

When Bob connects:
┌─────────────────────────────────┐
│ activeUsers                     │
│                                 │
│ "alice123" → "socket_abc456"    │
│ "bob456"   → "socket_xyz789"    │
└─────────────────────────────────┘

When Alice sends message to Bob:
1. activeUsers.get("bob456")
   → Returns "socket_xyz789"

2. io.to("socket_xyz789").emit("new_message", data)
   → Sends ONLY to Bob's socket

When Alice disconnects:
┌─────────────────────────────────┐
│ activeUsers                     │
│                                 │
│ "bob456"   → "socket_xyz789"    │
└─────────────────────────────────┘
activeUsers.delete("alice123")
```

**Flow**: When user types, notify others. When user stops, notify again.

**Server**:

```js
socket.on("typing_start", ({ conversationId }) => {
  // Send to everyone in conversation except sender
  socket.to(conversationId).emit("user_typing", {
    userId: socket.userId,
    username: socket.username,
    conversationId: conversationId
  });
});

socket.on("typing_stop", ({ conversationId }) => {
  socket.to(conversationId).emit("user_stopped_typing", {
    userId: socket.userId,
    conversationId: conversationId
  });
});
```

**Client**:

```js
const messageInput = document.getElementById("message-input");
let typingTimeout;

messageInput.addEventListener("input", () => {
  // Emit typing_start
  socket.emit("typing_start", { conversationId: currentConversationId });
  
  // Clear previous timeout
  clearTimeout(typingTimeout);
  
  // Set timeout to emit typing_stop after 1 second of inactivity
  typingTimeout = setTimeout(() => {
    socket.emit("typing_stop", { conversationId: currentConversationId });
  }, 1000);
});

// Listen for others typing
socket.on("user_typing", ({ username, conversationId }) => {
  if (conversationId === currentConversationId) {
    showTypingIndicator(`${username} is typing...`);
  }
});

socket.on("user_stopped_typing", ({ userId, conversationId }) => {
  if (conversationId === currentConversationId) {
    hideTypingIndicator(userId);
  }
});
```

### 7.2 User Presence (Online/Offline Status)

**Server**: Track and broadcast presence

```js
io.on("connection", async (socket) => {
  // Mark user online in database
  await User.findByIdAndUpdate(socket.userId, {
    isOnline: true,
    socketId: socket.id
  });

  // Broadcast to all connected clients
  io.emit("user_online", {
    userId: socket.userId,
    username: socket.username
  });

  socket.on("disconnect", async () => {
    // Mark user offline
    await User.findByIdAndUpdate(socket.userId, {
      isOnline: false,
      lastSeen: new Date(),
      socketId: null
    });

    // Broadcast offline status
    io.emit("user_offline", {
      userId: socket.userId
    });
  });
});
```

**Client**:

```js
socket.on("user_online", ({ userId, username }) => {
  console.log(`${username} is online`);
  // Update UI - show green dot
  updateUserStatus(userId, "online");
});

socket.on("user_offline", ({ userId }) => {
  console.log(`User ${userId} went offline`);
  // Update UI - show gray dot
  updateUserStatus(userId, "offline");
});
```

### 7.3 Message Status (Sent/Delivered/Read)

**Three states**:
1. **Sent**: Message saved in database
2. **Delivered**: Recipient's device received it
3. **Read**: Recipient opened and viewed it

**Server**: Mark as read

```js
socket.on("mark_as_read", async ({ messageId }, callback) => {
  try {
    const message = await Message.findById(messageId);
    
    if (!message) {
      return callback({ success: false, error: "Message not found" });
    }

    // Add current user to readBy array
    const alreadyRead = message.readBy.some(
      r => r.user.toString() === socket.userId
    );

    if (!alreadyRead) {
      message.readBy.push({
        user: socket.userId,
        readAt: new Date()
      });
      message.status = "read";
      await message.save();

      // Notify sender
      const sender = await User.findById(message.sender);
      if (sender && sender.isOnline && sender.socketId) {
        io.to(sender.socketId).emit("message_read", {
          messageId: message._id,
          readBy: socket.userId,
          readAt: new Date()
        });
      }
    }

    callback({ success: true });

  } catch (error) {
    callback({ success: false, error: error.message });
  }
});
```

**Client**:

```js
// When user views message
function markMessageAsRead(messageId) {
  socket.emit("mark_as_read", { messageId }, (response) => {
    if (response.success) {
      console.log("Message marked as read");
    }
  });
}

// When sender gets read receipt
socket.on("message_read", ({ messageId, readBy, readAt }) => {
  console.log(`Message ${messageId} was read`);
  // Update UI - show double blue checkmarks
  updateMessageStatus(messageId, "read");
});
```

---

## Part 8: MongoDB Schemas

### 8.1 Message Schema

**File**: `src/models/Message.model.js`

```js
const mongoose = require("mongoose");

const messageSchema = new mongoose.Schema({
  conversationId: {
    type: mongoose.Schema.Types.ObjectId,
    ref: "Conversation",
    required: true,
    index: true
  },
  sender: {
    type: mongoose.Schema.Types.ObjectId,
    ref: "User",
    required: true
  },
  content: {
    type: String,
    trim: true
  },
  messageType: {
    type: String,
    enum: ["text", "image", "video", "audio", "file"],
    default: "text"
  },
  file: {
    url: String,
    filename: String,
    size: Number,
    mimetype: String
  },
  status: {
    type: String,
    enum: ["sent", "delivered", "read"],
    default: "sent"
  },
  readBy: [{
    user: {
      type: mongoose.Schema.Types.ObjectId,
      ref: "User"
    },
    readAt: {
      type: Date,
      default: Date.now
    }
  }],
  isDeleted: {
    type: Boolean,
    default: false
  }
}, {
  timestamps: true
});

// Index for fast queries
messageSchema.index({ conversationId: 1, createdAt: -1 });

const Message = mongoose.model("Message", messageSchema);
module.exports = Message;
```

**Key fields**:
- `conversationId`: Links to conversation (indexed for fast lookup)
- `messageType`: Distinguishes text from media
- `file`: Stores file metadata (URL, not the actual file)
- `status`: Tracks delivery state
- `readBy`: Array supports group chats where multiple users read

### 8.2 Conversation Schema

**File**: `src/models/Conversation.model.js`

```js
const mongoose = require("mongoose");

const conversationSchema = new mongoose.Schema({
  type: {
    type: String,
    enum: ["direct", "group"],
    required: true
  },
  participants: [{
    type: mongoose.Schema.Types.ObjectId,
    ref: "User",
    required: true
  }],
  name: {
    type: String,
    trim: true
    // Required for groups, optional for direct
  },
  avatar: {
    type: String
    // Group avatar URL
  },
  admin: {
    type: mongoose.Schema.Types.ObjectId,
    ref: "User"
    // Only for groups
  },
  lastMessage: {
    content: String,
    sender: {
      type: mongoose.Schema.Types.ObjectId,
      ref: "User"
    },
    timestamp: Date,
    messageType: String
  },
  isActive: {
    type: Boolean,
    default: true
  }
}, {
  timestamps: true
});

// Index for finding conversations by participants
conversationSchema.index({ participants: 1, type: 1 });

const Conversation = mongoose.model("Conversation", conversationSchema);
module.exports = Conversation;
```

**Key fields**:
- `type`: "direct" or "group"
- `participants`: Array of user IDs
- `lastMessage`: Cached for efficiency (avoids querying messages table)
- `admin`: Who can add/remove members (groups only)

### 8.3 User Schema (Presence fields)

**File**: `src/models/User.model.js` (relevant fields)

```js
const mongoose = require("mongoose");

const userSchema = new mongoose.Schema({
  username: {
    type: String,
    required: true,
    unique: true
  },
  email: {
    type: String,
    required: true,
    unique: true
  },
  avatar: String,
  
  // Presence fields
  isOnline: {
    type: Boolean,
    default: false
  },
  lastSeen: {
    type: Date,
    default: Date.now
  },
  socketId: {
    type: String,
    default: null
  }
}, {
  timestamps: true
});

const User = mongoose.model("User", userSchema);
module.exports = User;
```

**Presence tracking**:
- `isOnline`: Currently connected
- `lastSeen`: When they were last online
- `socketId`: Their current socket (for targeting messages)

---

## Part 9: Complete Working Example

### 9.1 Project Structure

```text
chat-app/
├── src/
│   ├── server.js
│   ├── app.js
│   ├── config/
│   │   ├── database.js
│   │   └── socketio.js
│   ├── models/
│   │   ├── User.model.js
│   │   ├── Conversation.model.js
│   │   └── Message.model.js
│   ├── socket/
│   │   ├── directMessages.js
│   │   └── groupMessages.js
│   ├── routes/
│   │   └── upload.routes.js
│   └── middlewares/
│       └── auth.middleware.js
├── uploads/
├── .env
└── package.json
```

### 9.2 Server Entry Point

**File**: `src/server.js`

```js
require("dotenv").config();
const http = require("http");
const app = require("./app");
const connectDB = require("./config/database");
const setupSocketIO = require("./config/socketio");

const PORT = process.env.PORT || 3000;

// Create HTTP server
const server = http.createServer(app);

// Setup Socket.io
const io = setupSocketIO(server);

// Make io accessible in routes
app.set("io", io);

// Connect to MongoDB and start server
connectDB().then(() => {
  server.listen(PORT, () => {
    console.log(`Server running on port ${PORT}`);
  });
});
```

### 9.3 Socket.io Configuration

**File**: `src/config/socketio.js`

```js
const { Server } = require("socket.io");
const jwt = require("jsonwebtoken");
const User = require("../models/User.model");
const handleDirectMessages = require("../socket/directMessages");
const handleGroupMessages = require("../socket/groupMessages");

function setupSocketIO(server) {
  const io = new Server(server, {
    cors: {
      origin: process.env.CLIENT_URL || "http://localhost:5173",
      credentials: true
    }
  });

  // Store active users
  const activeUsers = new Map();

  // Authentication middleware
  io.use(async (socket, next) => {
    try {
      const token = socket.handshake.auth.token;
      
      if (!token) {
        return next(new Error("No token provided"));
      }

      const decoded = jwt.verify(token, process.env.JWT_SECRET);
      const user = await User.findById(decoded.id);

      if (!user) {
        return next(new Error("User not found"));
      }

      socket.userId = user._id.toString();
      socket.username = user.username;
      
      next();
    } catch (error) {
      next(new Error("Authentication failed"));
    }
  });

  io.on("connection", async (socket) => {
    console.log(`${socket.username} connected`);

    // Add to active users
    activeUsers.set(socket.userId, socket.id);

    // Update user status in database
    await User.findByIdAndUpdate(socket.userId, {
      isOnline: true,
      socketId: socket.id
    });

    // Broadcast online status
    io.emit("user_online", {
      userId: socket.userId,
      username: socket.username
    });

    // Initialize handlers
    handleDirectMessages(io, socket);
    handleGroupMessages(io, socket);

    // Handle disconnect
    socket.on("disconnect", async () => {
      console.log(`${socket.username} disconnected`);

      // Remove from active users
      activeUsers.delete(socket.userId);

      // Update database
      await User.findByIdAndUpdate(socket.userId, {
        isOnline: false,
        lastSeen: new Date(),
        socketId: null
      });

      // Broadcast offline status
      io.emit("user_offline", {
        userId: socket.userId
      });
    });
  });

  // Make activeUsers accessible
  io.activeUsers = activeUsers;

  return io;
}

module.exports = setupSocketIO;
```

### 9.4 Direct Message Handler

**File**: `src/socket/directMessages.js`

```js
const Message = require("../models/Message.model");
const Conversation = require("../models/Conversation.model");
const User = require("../models/User.model");

function handleDirectMessages(io, socket) {
  
  socket.on("send_direct_message", async (data, callback) => {
    try {
      const { recipientId, content, file } = data;
      const senderId = socket.userId;

      // Find or create conversation
      let conversation = await Conversation.findOne({
        type: "direct",
        participants: { $all: [senderId, recipientId], $size: 2 }
      });

      if (!conversation) {
        conversation = await Conversation.create({
          type: "direct",
          participants: [senderId, recipientId]
        });
      }

      // Create message
      const message = await Message.create({
        conversationId: conversation._id,
        sender: senderId,
        content: content || "",
        messageType: file ? file.messageType : "text",
        file: file || undefined,
        status: "sent"
      });

      await message.populate("sender", "username avatar");

      // Update conversation last message
      conversation.lastMessage = {
        content: content || "File",
        sender: senderId,
        timestamp: message.createdAt,
        messageType: message.messageType
      };
      await conversation.save();

      // Try to deliver to recipient
      const recipientSocketId = io.activeUsers.get(recipientId);
      
      if (recipientSocketId) {
        io.to(recipientSocketId).emit("new_message", {
          conversationId: conversation._id,
          message: message
        });
        
        message.status = "delivered";
        await message.save();
      }

      callback({
        success: true,
        message: message,
        status: message.status
      });

    } catch (error) {
      console.error("Error in send_direct_message:", error);
      callback({ success: false, error: error.message });
    }
  });

  socket.on("mark_as_read", async ({ messageId }, callback) => {
    try {
      const message = await Message.findById(messageId);
      
      if (!message) {
        return callback({ success: false, error: "Message not found" });
      }

      const alreadyRead = message.readBy.some(
        r => r.user.toString() === socket.userId
      );

      if (!alreadyRead) {
        message.readBy.push({
          user: socket.userId,
          readAt: new Date()
        });
        message.status = "read";
        await message.save();

        // Notify sender
        const sender = await User.findById(message.sender);
        if (sender && sender.isOnline && sender.socketId) {
          io.to(sender.socketId).emit("message_read", {
            messageId: message._id,
            conversationId: message.conversationId,
            readBy: socket.userId,
            readAt: new Date()
          });
        }
      }

      callback({ success: true });

    } catch (error) {
      callback({ success: false, error: error.message });
    }
  });
}

module.exports = handleDirectMessages;
```

### 9.5 Group Message Handler

**File**: `src/socket/groupMessages.js`

```js
const Message = require("../models/Message.model");
const Conversation = require("../models/Conversation.model");

function handleGroupMessages(io, socket) {
  
  socket.on("join_group", async (conversationId, callback) => {
    try {
      const conversation = await Conversation.findOne({
        _id: conversationId,
        type: "group",
        participants: socket.userId
      });

      if (!conversation) {
        return callback({ success: false, error: "Not a member" });
      }

      socket.join(conversationId);

      socket.to(conversationId).emit("user_joined_group", {
        userId: socket.userId,
        username: socket.username,
        conversationId: conversationId
      });

      callback({ success: true });

    } catch (error) {
      callback({ success: false, error: error.message });
    }
  });

  socket.on("leave_group", (conversationId) => {
    socket.leave(conversationId);
    
    socket.to(conversationId).emit("user_left_group", {
      userId: socket.userId,
      username: socket.username,
      conversationId: conversationId
    });
  });

  socket.on("send_group_message", async (data, callback) => {
    try {
      const { conversationId, content, file } = data;

      const conversation = await Conversation.findOne({
        _id: conversationId,
        participants: socket.userId
      });

      if (!conversation) {
        return callback({ success: false, error: "Not a member" });
      }

      const message = await Message.create({
        conversationId: conversationId,
        sender: socket.userId,
        content: content || "",
        messageType: file ? file.messageType : "text",
        file: file || undefined,
        status: "sent"
      });

      await message.populate("sender", "username avatar");

      conversation.lastMessage = {
        content: content || "File",
        sender: socket.userId,
        timestamp: message.createdAt,
        messageType: message.messageType
      };
      await conversation.save();

      io.to(conversationId).emit("new_group_message", {
        conversationId: conversationId,
        message: message
      });

      callback({ success: true, message: message });

    } catch (error) {
      callback({ success: false, error: error.message });
    }
  });

  socket.on("typing_start", ({ conversationId }) => {
    socket.to(conversationId).emit("user_typing", {
      userId: socket.userId,
      username: socket.username,
      conversationId: conversationId
    });
  });

  socket.on("typing_stop", ({ conversationId }) => {
    socket.to(conversationId).emit("user_stopped_typing", {
      userId: socket.userId,
      conversationId: conversationId
    });
  });
}

module.exports = handleGroupMessages;
```

### 9.6 Client Example

```html
<!DOCTYPE html>
<html>
<head>
  <title>Chat App</title>
  <script src="https://cdn.socket.io/4.5.4/socket.io.min.js"></script>
</head>
<body>
  <div id="messages"></div>
  <input type="text" id="message-input" placeholder="Type a message" />
  <button onclick="sendMessage()">Send</button>
  <input type="file" id="file-input" />
  <button onclick="sendFile()">Send File</button>

  <script>
    const token = "your-jwt-token"; // From login
    const recipientId = "recipient-user-id";
    const conversationId = "conversation-id";

    // Connect to server
    const socket = io("http://localhost:3000", {
      auth: { token: token }
    });

    socket.on("connect", () => {
      console.log("Connected");
      
      // Join group if needed
      socket.emit("join_group", conversationId, (response) => {
        console.log("Joined group:", response);
      });
    });

    // Receive new messages
    socket.on("new_message", ({ conversationId, message }) => {
      displayMessage(message);
    });

    socket.on("new_group_message", ({ conversationId, message }) => {
      displayMessage(message);
    });

    // Send text message
    function sendMessage() {
      const input = document.getElementById("message-input");
      const text = input.value.trim();
      
      if (!text) return;

      socket.emit("send_direct_message", {
        recipientId: recipientId,
        content: text
      }, (response) => {
        if (response.success) {
          displayMessage(response.message);
          input.value = "";
        }
      });
    }

    // Send file
    async function sendFile() {
      const fileInput = document.getElementById("file-input");
      const file = fileInput.files[0];
      
      if (!file) return;

      // Upload file
      const formData = new FormData();
      formData.append("file", file);

      const response = await fetch("http://localhost:3000/api/upload", {
        method: "POST",
        headers: {
          "Authorization": `Bearer ${token}`
        },
        body: formData
      });

      const data = await response.json();

      if (data.success) {
        // Send message with file
        socket.emit("send_direct_message", {
          recipientId: recipientId,
          content: "Sent a file",
          file: data.file
        }, (response) => {
          if (response.success) {
            displayMessage(response.message);
          }
        });
      }
    }

    // Display message
    function displayMessage(message) {
      const messagesDiv = document.getElementById("messages");
      const msgDiv = document.createElement("div");
      
      let content = `<strong>${message.sender.username}:</strong> ${message.content}`;
      
      if (message.file) {
        if (message.messageType === "image") {
          content += `<br><img src="${message.file.url}" style="max-width:200px" />`;
        } else {
          content += `<br><a href="${message.file.url}" target="_blank">${message.file.filename}</a>`;
        }
      }
      
      msgDiv.innerHTML = content;
      messagesDiv.appendChild(msgDiv);
    }

    // Typing indicator
    const input = document.getElementById("message-input");
    let typingTimeout;

    input.addEventListener("input", () => {
      socket.emit("typing_start", { conversationId });
      
      clearTimeout(typingTimeout);
      typingTimeout = setTimeout(() => {
        socket.emit("typing_stop", { conversationId });
      }, 1000);
    });

    socket.on("user_typing", ({ username }) => {
      console.log(`${username} is typing...`);
    });
  </script>
</body>
</html>
```

---

## Part 10: Key Concepts Reinforced

### Understanding Connection Lifecycle

```
Client connects
     ↓
Authentication (JWT verification)
     ↓
Create socket object with userId
     ↓
Add to activeUsers Map
     ↓
Update database (isOnline = true)
     ↓
Broadcast "user_online" event
     ↓
[Connection remains open]
     ↓
Client sends/receives events
     ↓
Client disconnects
     ↓
Remove from activeUsers Map
     ↓
Update database (isOnline = false)
     ↓
Broadcast "user_offline" event
```

### Message Routing Logic

```
Message arrives at server
     ↓
Extract senderId from socket.userId
     ↓
Extract recipientId from message data
     ↓
Save message to database
     ↓
Look up recipient in activeUsers Map
     ↓
     ├─ Found (online)
     │       ↓
     │  Send via io.to(recipientSocketId).emit()
     │       ↓
     │  Update status to "delivered"
     │
     └─ Not found (offline)
             ↓
        Keep status as "sent"
             ↓
        Deliver when recipient connects
```

### Room Broadcasting

```
User sends message to room
     ↓
Server receives on socket
     ↓
Verify user is member of room
     ↓
Save message to database
     ↓
Broadcast to room: io.to(roomId).emit()
     ↓
Socket.io sends to all sockets in that room
     ↓
All members receive (including sender)
```

### What are WebSockets?

**Simple explanation**: A persistent, two-way connection between client and server. Unlike HTTP where you ask a question and get an answer, WebSockets keep a phone line open so either side can talk anytime.

**Technical explanation**: WebSocket is a protocol providing full-duplex communication channels over a single TCP connection. After an initial HTTP handshake upgrade, the connection remains open, enabling low-latency, event-driven communication without the overhead of repeated HTTP requests.

### HTTP vs WebSocket

```
HTTP (Request-Response):
Client → Server: "Do you have new messages?"
Server → Client: "Here are 3 new messages"
[Connection closes]
Client → Server: "Do you have new messages?" [30 seconds later]
Server → Client: "No new messages"
[Connection closes]

WebSocket (Persistent Connection):
Client ↔ Server: [Connection established]
Server → Client: "New message arrived!" [whenever it happens]
Client → Server: "User is typing..." [whenever user types]
[Connection stays open]
```

**Why WebSockets for messaging?**
- **Real-time**: Messages arrive instantly without polling
- **Efficient**: One connection vs thousands of HTTP requests
- **Bidirectional**: Server can initiate communication
- **Lower latency**: No connection overhead per message

### When to Use WebSockets

✅ **Use WebSockets for**:
- Chat applications
- Live notifications
- Collaborative editing
- Real-time dashboards
- Gaming
- Live feeds

❌ **Don't use WebSockets for**:
- Simple CRUD operations
- File downloads (use HTTP)
- Static content
- One-time data fetching

---

## 2. Core WebSocket Implementation

### Installing Dependencies

```bash
npm install ws
```

**What is `ws`?**: A lightweight, fast WebSocket library for Node.js. It implements the WebSocket protocol with minimal abstraction.

### Basic Server Setup

**File**: `src/config/websocket.js`

```js
const WebSocket = require("ws");
const http = require("http");

function setupWebSocketServer(server) {
  const wss = new WebSocket.Server({ server });

  wss.on("connection", (ws, req) => {
    console.log("Client connected");

    // Handle incoming messages
    ws.on("message", (data) => {
      console.log("Received:", data.toString());
      
      // Echo back to client
      ws.send(JSON.stringify({
        type: "message",
        data: "Message received"
      }));
    });

    // Handle disconnection
    ws.on("close", () => {
      console.log("Client disconnected");
    });

    // Handle errors
    ws.on("error", (error) => {
      console.error("WebSocket error:", error);
    });

    // Send welcome message
    ws.send(JSON.stringify({
      type: "connected",
      message: "Welcome to WebSocket server"
    }));
  });

  return wss;
}

module.exports = setupWebSocketServer;
```

**File**: `src/server.js` (Integration)

```js
require("dotenv").config();
const express = require("express");
const http = require("http");
const setupWebSocketServer = require("./config/websocket");

const app = express();
const server = http.createServer(app);

// Setup WebSocket server
const wss = setupWebSocketServer(server);

// Regular HTTP routes
app.get("/health", (req, res) => {
  res.json({ status: "ok" });
});

server.listen(3000, () => {
  console.log("Server running on port 3000");
});
```

**Why create HTTP server explicitly?**: Both Express and WebSocket need to share the same server instance to work on the same port.

### Connection Management

```js
// Store connected clients with metadata
const clients = new Map();

wss.on("connection", (ws, req) => {
  // Generate unique client ID
  const clientId = generateUniqueId();
  
  // Store client connection with metadata
  clients.set(clientId, {
    ws,
    userId: null,  // Set after authentication
    rooms: new Set()
  });

  ws.clientId = clientId;

  ws.on("close", () => {
    clients.delete(clientId);
  });
});

function generateUniqueId() {
  return `${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;
}
```

**Why Map instead of array?**: O(1) lookup time for finding specific clients by ID.

### Authentication with WebSocket

```js
const jwt = require("jsonwebtoken");

wss.on("connection", (ws, req) => {
  // Extract token from query string or headers
  const token = new URL(req.url, "ws://localhost").searchParams.get("token");
  
  if (!token) {
    ws.send(JSON.stringify({ type: "error", message: "No token provided" }));
    ws.close();
    return;
  }

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    
    const clientId = generateUniqueId();
    clients.set(clientId, {
      ws,
      userId: decoded.id,
      username: decoded.username,
      rooms: new Set()
    });
    
    ws.clientId = clientId;
    ws.userId = decoded.id;
    
    ws.send(JSON.stringify({
      type: "authenticated",
      userId: decoded.id
    }));
    
  } catch (error) {
    ws.send(JSON.stringify({ type: "error", message: "Invalid token" }));
    ws.close();
  }
});
```

**Client connection example**:
```js
// Frontend
const token = "your-jwt-token";
const ws = new WebSocket(`ws://localhost:3000?token=${token}`);

ws.onmessage = (event) => {
  const data = JSON.parse(event.data);
  console.log("Received:", data);
};

ws.send(JSON.stringify({
  type: "message",
  content: "Hello server"
}));
```

### One-to-One Messaging (General Pattern)

```js
ws.on("message", (data) => {
  const message = JSON.parse(data.toString());

  if (message.type === "direct_message") {
    const { recipientId, content } = message;
    
    // Find recipient's WebSocket connection
    let recipientWs = null;
    for (const [clientId, client] of clients) {
      if (client.userId === recipientId) {
        recipientWs = client.ws;
        break;
      }
    }

    if (recipientWs && recipientWs.readyState === WebSocket.OPEN) {
      // Send to recipient
      recipientWs.send(JSON.stringify({
        type: "direct_message",
        from: ws.userId,
        content,
        timestamp: new Date()
      }));

      // Confirm to sender
      ws.send(JSON.stringify({
        type: "message_sent",
        to: recipientId,
        status: "delivered"
      }));
    } else {
      // Recipient offline
      ws.send(JSON.stringify({
        type: "message_sent",
        to: recipientId,
        status: "pending"
      }));
    }
  }
});
```

**Key concepts**:
- `clients` Map stores all active connections
- Find recipient by `userId`
- Check `readyState === WebSocket.OPEN` before sending
- Handle offline recipients gracefully

### Group Chat Rooms (General Pattern)

```js
// Join room
function joinRoom(ws, roomId) {
  const client = clients.get(ws.clientId);
  if (client) {
    client.rooms.add(roomId);
    
    // Notify others in room
    broadcastToRoom(roomId, {
      type: "user_joined",
      userId: client.userId,
      roomId
    }, ws.clientId);
  }
}

// Leave room
function leaveRoom(ws, roomId) {
  const client = clients.get(ws.clientId);
  if (client) {
    client.rooms.delete(roomId);
    
    // Notify others in room
    broadcastToRoom(roomId, {
      type: "user_left",
      userId: client.userId,
      roomId
    }, ws.clientId);
  }
}

// Broadcast to all users in a room
function broadcastToRoom(roomId, message, excludeClientId = null) {
  for (const [clientId, client] of clients) {
    if (client.rooms.has(roomId) && clientId !== excludeClientId) {
      if (client.ws.readyState === WebSocket.OPEN) {
        client.ws.send(JSON.stringify(message));
      }
    }
  }
}

// Handle room messages
ws.on("message", (data) => {
  const message = JSON.parse(data.toString());

  if (message.type === "join_room") {
    joinRoom(ws, message.roomId);
  }

  if (message.type === "leave_room") {
    leaveRoom(ws, message.roomId);
  }

  if (message.type === "room_message") {
    const { roomId, content } = message;
    
    broadcastToRoom(roomId, {
      type: "room_message",
      from: ws.userId,
      roomId,
      content,
      timestamp: new Date()
    });
  }
});
```

**Room architecture**:
- Each client has a `Set` of rooms they're in
- Broadcasting iterates all clients and filters by room membership
- Exclude sender to avoid echo (optional based on UX needs)

### File Handling - Approach A (HTTP Upload + WebSocket Notification)

**Recommended for production** - Better for large files, progress tracking, and separation of concerns.

**File upload endpoint**:

```js
const multer = require("multer");
const path = require("path");

// Configure storage
const storage = multer.diskStorage({
  destination: (req, file, cb) => {
    cb(null, "uploads/");
  },
  filename: (req, file, cb) => {
    const uniqueName = `${Date.now()}-${file.originalname}`;
    cb(null, uniqueName);
  }
});

const upload = multer({
  storage,
  limits: { fileSize: 10 * 1024 * 1024 }, // 10MB limit
  fileFilter: (req, file, cb) => {
    const allowedTypes = /jpeg|jpg|png|gif|pdf|doc|docx/;
    const extname = allowedTypes.test(path.extname(file.originalname).toLowerCase());
    const mimetype = allowedTypes.test(file.mimetype);
    
    if (extname && mimetype) {
      cb(null, true);
    } else {
      cb(new Error("Invalid file type"));
    }
  }
});

// Upload endpoint
app.post("/api/upload", upload.single("file"), async (req, res) => {
  try {
    const fileUrl = `/uploads/${req.file.filename}`;
    const fileMetadata = {
      filename: req.file.originalname,
      size: req.file.size,
      mimetype: req.file.mimetype,
      url: fileUrl
    };

    res.json({
      success: true,
      file: fileMetadata
    });
  } catch (error) {
    res.status(500).json({
      success: false,
      error: error.message
    });
  }
});

// Serve uploaded files
app.use("/uploads", express.static("uploads"));
```

**WebSocket message with file**:

```js
// After file upload succeeds
ws.on("message", (data) => {
  const message = JSON.parse(data.toString());

  if (message.type === "message_with_file") {
    const { recipientId, text, fileMetadata } = message;
    
    // Find recipient
    let recipientWs = null;
    for (const [clientId, client] of clients) {
      if (client.userId === recipientId) {
        recipientWs = client.ws;
        break;
      }
    }

    if (recipientWs && recipientWs.readyState === WebSocket.OPEN) {
      recipientWs.send(JSON.stringify({
        type: "message_with_file",
        from: ws.userId,
        text,
        file: fileMetadata,
        timestamp: new Date()
      }));
    }
  }
});
```

**Client flow**:
```js
// 1. Upload file via HTTP
const formData = new FormData();
formData.append("file", fileInput.files[0]);

const uploadResponse = await fetch("/api/upload", {
  method: "POST",
  body: formData
});

const { file } = await uploadResponse.json();

// 2. Send message via WebSocket with file reference
ws.send(JSON.stringify({
  type: "message_with_file",
  recipientId: "user123",
  text: "Check this out!",
  fileMetadata: file
}));
```

### File Handling - Approach B (Direct WebSocket Transfer)

**Better for small files** - Fewer HTTP requests, simpler flow.

```js
ws.on("message", (data) => {
  // Check if binary data (file) or text (JSON)
  if (data instanceof Buffer) {
    handleFileUpload(ws, data);
  } else {
    const message = JSON.parse(data.toString());
    handleTextMessage(ws, message);
  }
});

function handleFileUpload(ws, fileBuffer) {
  const filename = `${Date.now()}-${ws.userId}.bin`;
  const filepath = path.join(__dirname, "uploads", filename);

  fs.writeFile(filepath, fileBuffer, (err) => {
    if (err) {
      ws.send(JSON.stringify({
        type: "upload_error",
        error: "Failed to save file"
      }));
      return;
    }

    ws.send(JSON.stringify({
      type: "upload_success",
      fileUrl: `/uploads/${filename}`
    }));
  });
}
```

**Client sending file**:
```js
// Read file as ArrayBuffer
const file = fileInput.files[0];
const reader = new FileReader();

reader.onload = (event) => {
  // Send binary data directly through WebSocket
  ws.send(event.target.result);
};

reader.readAsArrayBuffer(file);
```

**Chunked upload for larger files**:

```js
// Server-side
const fileChunks = new Map(); // clientId -> { chunks: [], metadata: {} }

ws.on("message", (data) => {
  const message = typeof data === "string" ? JSON.parse(data) : null;

  if (message && message.type === "file_start") {
    // Initialize file upload
    fileChunks.set(ws.clientId, {
      chunks: [],
      metadata: message.metadata,
      receivedSize: 0
    });
    
    ws.send(JSON.stringify({ type: "ready_for_chunks" }));
  }

  if (message && message.type === "file_chunk") {
    // Receive chunk
    const uploadState = fileChunks.get(ws.clientId);
    const chunkBuffer = Buffer.from(message.data, "base64");
    
    uploadState.chunks.push(chunkBuffer);
    uploadState.receivedSize += chunkBuffer.length;

    // Send progress
    ws.send(JSON.stringify({
      type: "upload_progress",
      percentage: (uploadState.receivedSize / uploadState.metadata.size) * 100
    }));
  }

  if (message && message.type === "file_end") {
    // Assemble and save file
    const uploadState = fileChunks.get(ws.clientId);
    const completeFile = Buffer.concat(uploadState.chunks);
    
    const filename = `${Date.now()}-${uploadState.metadata.name}`;
    const filepath = path.join(__dirname, "uploads", filename);

    fs.writeFile(filepath, completeFile, (err) => {
      if (err) {
        ws.send(JSON.stringify({
          type: "upload_error",
          error: "Failed to save file"
        }));
      } else {
        ws.send(JSON.stringify({
          type: "upload_complete",
          fileUrl: `/uploads/${filename}`
        }));
        
        fileChunks.delete(ws.clientId);
      }
    });
  }
});
```

**Client sending chunks**:
```js
const CHUNK_SIZE = 64 * 1024; // 64KB chunks

function sendFile(file) {
  // 1. Send metadata
  ws.send(JSON.stringify({
    type: "file_start",
    metadata: {
      name: file.name,
      size: file.size,
      type: file.type
    }
  }));

  // 2. Wait for server ready
  ws.onmessage = async (event) => {
    const data = JSON.parse(event.data);
    
    if (data.type === "ready_for_chunks") {
      // 3. Send file in chunks
      let offset = 0;
      
      while (offset < file.size) {
        const chunk = file.slice(offset, offset + CHUNK_SIZE);
        const arrayBuffer = await chunk.arrayBuffer();
        const base64 = btoa(String.fromCharCode(...new Uint8Array(arrayBuffer)));
        
        ws.send(JSON.stringify({
          type: "file_chunk",
          data: base64
        }));
        
        offset += CHUNK_SIZE;
      }
      
      // 4. Signal completion
      ws.send(JSON.stringify({ type: "file_end" }));
    }
    
    if (data.type === "upload_progress") {
      console.log(`Upload progress: ${data.percentage}%`);
    }
  };
}
```

**Comparison**:

| Feature | HTTP Upload (A) | WebSocket Upload (B) |
|---------|----------------|---------------------|
| **Best for** | Large files (>1MB) | Small files (<1MB) |
| **Progress tracking** | Easy (native) | Manual implementation |
| **Resumable** | Yes (with additional code) | Complex |
| **Connection overhead** | Extra HTTP request | Uses existing connection |
| **Error handling** | Standard HTTP errors | Custom protocol |
| **Browser compatibility** | Universal | Requires WebSocket support |

---

## 3. Socket.io Implementation

### What is Socket.io?

**Simple explanation**: A library that makes WebSockets easier to use and more reliable. It automatically handles reconnection, fallbacks, and provides convenient features like rooms and namespaces.

**Technical explanation**: Socket.io is a higher-level abstraction over WebSockets that provides automatic reconnection, binary support, broadcasting, rooms, namespaces, and fallback to HTTP long-polling when WebSockets aren't available. It adds features like acknowledgments, middleware, and event-based communication.

### Why Socket.io over Core WebSocket?

**Advantages**:
- ✅ Automatic reconnection with exponential backoff
- ✅ Built-in room management
- ✅ Event-based API (cleaner than parsing JSON)
- ✅ Acknowledgments (confirm message receipt)
- ✅ Broadcasting utilities
- ✅ Fallback to long-polling (older browsers)
- ✅ Middleware support

**Disadvantages**:
- ❌ Larger bundle size
- ❌ More abstraction (less control)
- ❌ Custom protocol (not pure WebSocket)

### Installing Dependencies

```bash
npm install socket.io
```

### Basic Server Setup

**File**: `src/config/socketio.js`

```js
const socketIO = require("socket.io");

function setupSocketIO(server) {
  const io = socketIO(server, {
    cors: {
      origin: process.env.CLIENT_URL,
      credentials: true
    }
  });

  // Middleware for authentication
  io.use((socket, next) => {
    const token = socket.handshake.auth.token;
    
    if (!token) {
      return next(new Error("Authentication error"));
    }

    try {
      const decoded = jwt.verify(token, process.env.JWT_SECRET);
      socket.userId = decoded.id;
      socket.username = decoded.username;
      next();
    } catch (err) {
      next(new Error("Invalid token"));
    }
  });

  io.on("connection", (socket) => {
    console.log(`User connected: ${socket.userId}`);

    // Handle events
    socket.on("message", (data) => {
      console.log("Received:", data);
    });

    socket.on("disconnect", () => {
      console.log(`User disconnected: ${socket.userId}`);
    });
  });

  return io;
}

module.exports = setupSocketIO;
```

**File**: `src/server.js` (Integration)

```js
require("dotenv").config();
const express = require("express");
const http = require("http");
const setupSocketIO = require("./config/socketio");

const app = express();
const server = http.createServer(app);

// Setup Socket.io
const io = setupSocketIO(server);

// Make io accessible in routes
app.set("io", io);

server.listen(3000, () => {
  console.log("Server running on port 3000");
});
```

**Client connection**:
```js
// Frontend
import { io } from "socket.io-client";

const socket = io("http://localhost:3000", {
  auth: {
    token: "your-jwt-token"
  }
});

socket.on("connect", () => {
  console.log("Connected to server");
});

socket.on("message", (data) => {
  console.log("Received:", data);
});

socket.emit("message", { content: "Hello server" });
```

### One-to-One Messaging with Socket.io

```js
io.on("connection", (socket) => {
  // Send direct message
  socket.on("direct_message", async (data, callback) => {
    const { recipientId, content } = data;
    
    // Find recipient's socket
    const recipientSockets = await io.fetchSockets();
    const recipientSocket = recipientSockets.find(s => s.userId === recipientId);

    if (recipientSocket) {
      // Send to recipient
      recipientSocket.emit("direct_message", {
        from: socket.userId,
        fromUsername: socket.username,
        content,
        timestamp: new Date()
      });

      // Acknowledge to sender
      callback({ status: "delivered" });
    } else {
      // Recipient offline
      callback({ status: "pending" });
    }
  });
});
```

**Client usage**:
```js
socket.emit("direct_message", {
  recipientId: "user123",
  content: "Hello!"
}, (response) => {
  console.log("Message status:", response.status);
});
```

**Acknowledgments**: The `callback` parameter allows the sender to receive confirmation that the message was processed.

### Group Chat with Socket.io Rooms

**Socket.io rooms** are built-in and much simpler than manual tracking.

```js
io.on("connection", (socket) => {
  // Join room
  socket.on("join_room", (roomId, callback) => {
    socket.join(roomId);
    
    // Notify others in room
    socket.to(roomId).emit("user_joined", {
      userId: socket.userId,
      username: socket.username,
      roomId
    });

    callback({ status: "joined", roomId });
  });

  // Leave room
  socket.on("leave_room", (roomId) => {
    socket.leave(roomId);
    
    // Notify others
    socket.to(roomId).emit("user_left", {
      userId: socket.userId,
      username: socket.username,
      roomId
    });
  });

  // Send message to room
  socket.on("room_message", (data, callback) => {
    const { roomId, content } = data;
    
    // Broadcast to everyone in room (including sender)
    io.to(roomId).emit("room_message", {
      from: socket.userId,
      fromUsername: socket.username,
      roomId,
      content,
      timestamp: new Date()
    });

    callback({ status: "sent" });
  });

  // Typing indicator
  socket.on("typing_start", ({ roomId }) => {
    socket.to(roomId).emit("user_typing", {
      userId: socket.userId,
      username: socket.username,
      roomId
    });
  });

  socket.on("typing_stop", ({ roomId }) => {
    socket.to(roomId).emit("user_stopped_typing", {
      userId: socket.userId,
      roomId
    });
  });

  // Auto-leave all rooms on disconnect
  socket.on("disconnect", () => {
    // Socket.io automatically removes from all rooms
  });
});
```

**Broadcasting patterns**:
```js
// To everyone except sender
socket.to(roomId).emit("event", data);

// To everyone including sender
io.to(roomId).emit("event", data);

// To multiple rooms
socket.to(room1).to(room2).emit("event", data);

// To specific socket
io.to(socketId).emit("event", data);
```

### File Handling with Socket.io - Approach A

Same as core WebSocket (HTTP upload + Socket notification):

```js
// After file upload via HTTP endpoint
socket.on("message_with_file", async (data, callback) => {
  const { recipientId, text, fileMetadata } = data;
  
  const recipientSockets = await io.fetchSockets();
  const recipientSocket = recipientSockets.find(s => s.userId === recipientId);

  if (recipientSocket) {
    recipientSocket.emit("message_with_file", {
      from: socket.userId,
      fromUsername: socket.username,
      text,
      file: fileMetadata,
      timestamp: new Date()
    });
    
    callback({ status: "delivered" });
  } else {
    callback({ status: "pending" });
  }
});
```

### File Handling with Socket.io - Approach B

```js
socket.on("file_upload", (data, callback) => {
  const { filename, fileData, mimetype } = data;
  
  // Decode base64 file data
  const buffer = Buffer.from(fileData, "base64");
  const filepath = path.join(__dirname, "uploads", `${Date.now()}-${filename}`);

  fs.writeFile(filepath, buffer, (err) => {
    if (err) {
      callback({ success: false, error: "Upload failed" });
      return;
    }

    const fileUrl = `/uploads/${path.basename(filepath)}`;
    callback({
      success: true,
      fileUrl,
      filename,
      size: buffer.length,
      mimetype
    });
  });
});
```

**Client sending file**:
```js
function sendFile(file) {
  const reader = new FileReader();
  
  reader.onload = () => {
    const base64Data = reader.result.split(",")[1]; // Remove data URL prefix
    
    socket.emit("file_upload", {
      filename: file.name,
      fileData: base64Data,
      mimetype: file.type
    }, (response) => {
      if (response.success) {
        console.log("File uploaded:", response.fileUrl);
        
        // Now send message with file
        socket.emit("message_with_file", {
          recipientId: "user123",
          text: "Sent a file",
          fileMetadata: {
            url: response.fileUrl,
            filename: response.filename,
            size: response.size,
            mimetype: response.mimetype
          }
        });
      }
    });
  };
  
  reader.readAsDataURL(file);
}
```

### User Presence Tracking

```js
// Track online users
const onlineUsers = new Map(); // userId -> socketId

io.on("connection", (socket) => {
  // Mark user as online
  onlineUsers.set(socket.userId, socket.id);
  
  // Broadcast to all connected users
  io.emit("user_online", {
    userId: socket.userId,
    username: socket.username
  });

  socket.on("disconnect", () => {
    // Mark user as offline
    onlineUsers.delete(socket.userId);
    
    // Broadcast offline status
    io.emit("user_offline", {
      userId: socket.userId
    });
  });

  // Get online users list
  socket.on("get_online_users", (callback) => {
    const users = Array.from(onlineUsers.keys());
    callback({ users });
  });
});
```

---

## 4. MongoDB Schema Design

### Message Schema

**File**: `src/models/Message.model.js`

```js
const mongoose = require("mongoose");

const messageSchema = new mongoose.Schema({
  conversationId: {
    type: mongoose.Schema.Types.ObjectId,
    ref: "Conversation",
    required: true,
    index: true
  },
  sender: {
    type: mongoose.Schema.Types.ObjectId,
    ref: "User",
    required: true
  },
  content: {
    type: String,
    trim: true
  },
  messageType: {
    type: String,
    enum: ["text", "file", "image", "video", "audio"],
    default: "text"
  },
  file: {
    url: String,
    filename: String,
    size: Number,
    mimetype: String
  },
  status: {
    type: String,
    enum: ["sent", "delivered", "read"],
    default: "sent"
  },
  readBy: [{
    user: {
      type: mongoose.Schema.Types.ObjectId,
      ref: "User"
    },
    readAt: {
      type: Date,
      default: Date.now
    }
  }],
  isDeleted: {
    type: Boolean,
    default: false
  },
  deletedFor: [{
    type: mongoose.Schema.Types.ObjectId,
    ref: "User"
  }]
}, {
  timestamps: true
});

// Index for efficient querying
messageSchema.index({ conversationId: 1, createdAt: -1 });
messageSchema.index({ sender: 1, createdAt: -1 });

// Virtual for checking if message has file
messageSchema.virtual("hasFile").get(function() {
  return this.messageType !== "text" && this.file && this.file.url;
});

// Instance method - Mark as read by user
messageSchema.methods.markAsRead = async function(userId) {
  // Check if already read by this user
  const alreadyRead = this.readBy.some(r => r.user.toString() === userId.toString());
  
  if (!alreadyRead) {
    this.readBy.push({ user: userId, readAt: new Date() });
    this.status = "read";
    await this.save();
  }
};

// Static method - Get unread count for user
messageSchema.statics.getUnreadCount = function(userId, conversationId) {
  return this.countDocuments({
    conversationId,
    sender: { $ne: userId },
    "readBy.user": { $ne: userId },
    isDeleted: false
  });
};

const Message = mongoose.model("Message", messageSchema);
module.exports = Message;
```

**Schema design decisions**:
- **conversationId**: Links message to conversation (indexed for fast queries)
- **messageType**: Distinguishes text from media messages
- **file object**: Stores metadata (URL stored, not binary data)
- **status**: Tracks delivery state
- **readBy array**: Supports group chats where multiple users read
- **deletedFor**: Allows "delete for me" vs "delete for everyone"

### Conversation Schema

**File**: `src/models/Conversation.model.js`

```js
const mongoose = require("mongoose");

const conversationSchema = new mongoose.Schema({
  type: {
    type: String,
    enum: ["direct", "group"],
    required: true
  },
  participants: [{
    type: mongoose.Schema.Types.ObjectId,
    ref: "User",
    required: true
  }],
  name: {
    type: String,
    trim: true
    // Required for groups, optional for direct messages
  },
  avatar: {
    type: String
    // Group avatar URL
  },
  admin: {
    type: mongoose.Schema.Types.ObjectId,
    ref: "User"
    // Only for groups
  },
  lastMessage: {
    content: String,
    sender: {
      type: mongoose.Schema.Types.ObjectId,
      ref: "User"
    },
    timestamp: Date,
    messageType: {
      type: String,
      enum: ["text", "file", "image", "video", "audio"]
    }
  },
  isActive: {
    type: Boolean,
    default: true
  }
}, {
  timestamps: true
});

// Compound index for finding conversations by participants
conversationSchema.index({ participants: 1, type: 1 });

// Virtual for participant count
conversationSchema.virtual("participantCount").get(function() {
  return this.participants.length;
});

// Static method - Find direct conversation between two users
conversationSchema.statics.findDirectConversation = function(user1Id, user2Id) {
  return this.findOne({
    type: "direct",
    participants: { $all: [user1Id, user2Id], $size: 2 }
  });
};

// Static method - Find or create direct conversation
conversationSchema.statics.findOrCreateDirect = async function(user1Id, user2Id) {
  let conversation = await this.findDirectConversation(user1Id, user2Id);
  
  if (!conversation) {
    conversation = await this.create({
      type: "direct",
      participants: [user1Id, user2Id]
    });
  }
  
  return conversation;
};

// Instance method - Update last message
conversationSchema.methods.updateLastMessage = async function(message) {
  this.lastMessage = {
    content: message.content || "File",
    sender: message.sender,
    timestamp: message.createdAt,
    messageType: message.messageType
  };
  await this.save();
};

const Conversation = mongoose.model("Conversation", conversationSchema);
module.exports = Conversation;
```

**Schema design decisions**:
- **type**: Distinguishes direct vs group conversations
- **participants array**: Users in conversation (indexed for queries)
- **admin**: Only relevant for groups (who can add/remove members)
- **lastMessage**: Denormalized for efficiency (avoids extra query to show conversation list)

### User Schema (Presence & Typing)

**File**: `src/models/User.model.js` (relevant fields)

```js
const mongoose = require("mongoose");

const userSchema = new mongoose.Schema({
  username: {
    type: String,
    required: true,
    unique: true
  },
  email: {
    type: String,
    required: true,
    unique: true
  },
  avatar: String,
  isOnline: {
    type: Boolean,
    default: false
  },
  lastSeen: {
    type: Date,
    default: Date.now
  },
  socketId: {
    type: String,
    default: null
  }
}, {
  timestamps: true
});

// Instance method - Set online status
userSchema.methods.setOnline = async function(socketId) {
  this.isOnline = true;
  this.socketId = socketId;
  await this.save();
};

// Instance method - Set offline status
userSchema.methods.setOffline = async function() {
  this.isOnline = false;
  this.lastSeen = new Date();
  this.socketId = null;
  await this.save();
};

const User = mongoose.model("User", userSchema);
module.exports = User;
```

---

## 5. Complete Example - Real-Time Chat Application

This example demonstrates a production-ready messaging system with Socket.io and MongoDB.

### Project Structure

```text
chat-app/
├── src/
│   ├── server.js
│   ├── app.js
│   ├── config/
│   │   ├── database.js
│   │   └── socketio.js
│   ├── models/
│   │   ├── User.model.js
│   │   ├── Conversation.model.js
│   │   └── Message.model.js
│   ├── controllers/
│   │   ├── message.controller.js
│   │   └── conversation.controller.js
│   ├── socket/
│   │   └── messageHandler.js
│   ├── routes/
│   │   └── upload.routes.js
│   └── middlewares/
│       └── auth.middleware.js
├── uploads/
├── .env
└── package.json
```

### Socket.io Configuration

**File**: `src/config/socketio.js`

```js
const socketIO = require("socket.io");
const jwt = require("jsonwebtoken");
const User = require("../models/User.model");
const messageHandler = require("../socket/messageHandler");

function setupSocketIO(server) {
  const io = socketIO(server, {
    cors: {
      origin: process.env.CLIENT_URL || "http://localhost:5173",
      credentials: true
    },
    maxHttpBufferSize: 1e8 // 100MB for file uploads
  });

  // Authentication middleware
  io.use(async (socket, next) => {
    try {
      const token = socket.handshake.auth.token;
      
      if (!token) {
        return next(new Error("Authentication error: No token provided"));
      }

      const decoded = jwt.verify(token, process.env.JWT_SECRET);
      const user = await User.findById(decoded.id);

      if (!user) {
        return next(new Error("Authentication error: User not found"));
      }

      socket.userId = user._id.toString();
      socket.username = user.username;
      next();
    } catch (error) {
      next(new Error("Authentication error: Invalid token"));
    }
  });

  io.on("connection", async (socket) => {
    console.log(`User connected: ${socket.username} (${socket.userId})`);

    // Update user online status
    await User.findByIdAndUpdate(socket.userId, {
      isOnline: true,
      socketId: socket.id
    });

    // Broadcast online status
    io.emit("user_online", {
      userId: socket.userId,
      username: socket.username
    });

    // Initialize message handlers
    messageHandler(io, socket);

    // Handle disconnection
    socket.on("disconnect", async () => {
      console.log(`User disconnected: ${socket.username}`);

      // Update user offline status
      await User.findByIdAndUpdate(socket.userId, {
        isOnline: false,
        lastSeen: new Date(),
        socketId: null
      });

      // Broadcast offline status
      io.emit("user_offline", {
        userId: socket.userId
      });
    });
  });

  return io;
}

module.exports = setupSocketIO;
```

### Message Handler

**File**: `src/socket/messageHandler.js`

```js
const Message = require("../models/Message.model");
const Conversation = require("../models/Conversation.model");
const User = require("../models/User.model");

function messageHandler(io, socket) {
  
  // Join conversation room
  socket.on("join_conversation", async (conversationId, callback) => {
    try {
      // Verify user is participant
      const conversation = await Conversation.findOne({
        _id: conversationId,
        participants: socket.userId
      });

      if (!conversation) {
        return callback({ error: "Conversation not found or access denied" });
      }

      socket.join(conversationId);
      
      callback({
        success: true,
        conversationId,
        type: conversation.type
      });
    } catch (error) {
      callback({ error: error.message });
    }
  });

  // Send direct message
  socket.on("send_direct_message", async (data, callback) => {
    try {
      const { recipientId, content, messageType = "text", file } = data;

      // Find or create conversation
      const conversation = await Conversation.findOrCreateDirect(
        socket.userId,
        recipientId
      );

      // Create message
      const message = await Message.create({
        conversationId: conversation._id,
        sender: socket.userId,
        content,
        messageType,
        file,
        status: "sent"
      });

      // Populate sender info
      await message.populate("sender", "username avatar");

      // Update conversation's last message
      await conversation.updateLastMessage(message);

      // Get recipient's socket
      const recipient = await User.findById(recipientId);

      if (recipient && recipient.isOnline && recipient.socketId) {
        // Recipient is online - deliver immediately
        io.to(recipient.socketId).emit("new_message", {
          conversationId: conversation._id,
          message: message.toObject()
        });

        // Update status to delivered
        message.status = "delivered";
        await message.save();
      }

      // Confirm to sender
      callback({
        success: true,
        message: message.toObject(),
        status: message.status
      });

    } catch (error) {
      console.error("Error sending direct message:", error);
      callback({ error: error.message });
    }
  });

  // Send group message
  socket.on("send_group_message", async (data, callback) => {
    try {
      const { conversationId, content, messageType = "text", file } = data;

      // Verify user is participant
      const conversation = await Conversation.findOne({
        _id: conversationId,
        participants: socket.userId
      });

      if (!conversation) {
        return callback({ error: "Conversation not found or access denied" });
      }

      // Create message
      const message = await Message.create({
        conversationId: conversation._id,
        sender: socket.userId,
        content,
        messageType,
        file,
        status: "sent"
      });

      // Populate sender info
      await message.populate("sender", "username avatar");

      // Update conversation's last message
      await conversation.updateLastMessage(message);

      // Broadcast to all participants in the room
      io.to(conversationId).emit("new_message", {
        conversationId: conversation._id,
        message: message.toObject()
      });

      callback({
        success: true,
        message: message.toObject()
      });

    } catch (error) {
      console.error("Error sending group message:", error);
      callback({ error: error.message });
    }
  });

  // Mark message as read
  socket.on("mark_as_read", async (data, callback) => {
    try {
      const { messageId } = data;

      const message = await Message.findById(messageId);

      if (!message) {
        return callback({ error: "Message not found" });
      }

      // Mark as read by current user
      await message.markAsRead(socket.userId);

      // Notify sender about read receipt
      const sender = await User.findById(message.sender);
      if (sender && sender.isOnline && sender.socketId) {
        io.to(sender.socketId).emit("message_read", {
          messageId: message._id,
          conversationId: message.conversationId,
          readBy: socket.userId,
          readAt: new Date()
        });
      }

      callback({ success: true });

    } catch (error) {
      console.error("Error marking message as read:", error);
      callback({ error: error.message });
    }
  });

  // Typing indicator
  socket.on("typing_start", ({ conversationId }) => {
    socket.to(conversationId).emit("user_typing", {
      conversationId,
      userId: socket.userId,
      username: socket.username
    });
  });

  socket.on("typing_stop", ({ conversationId }) => {
    socket.to(conversationId).emit("user_stopped_typing", {
      conversationId,
      userId: socket.userId
    });
  });

  // Get online users
  socket.on("get_online_users", async (callback) => {
    try {
      const onlineUsers = await User.find({ isOnline: true })
        .select("_id username avatar");

      callback({
        success: true,
        users: onlineUsers
      });
    } catch (error) {
      callback({ error: error.message });
    }
  });
}

module.exports = messageHandler;
```

### HTTP Controllers for Messages

**File**: `src/controllers/message.controller.js`

```js
const Message = require("../models/Message.model");
const Conversation = require("../models/Conversation.model");

// Get conversation messages with pagination
exports.getMessages = async (req, res) => {
  try {
    const { conversationId } = req.params;
    const { limit = 50, before } = req.query; // 'before' is a message ID for pagination

    // Verify user is participant
    const conversation = await Conversation.findOne({
      _id: conversationId,
      participants: req.user._id
    });

    if (!conversation) {
      return res.status(404).json({
        success: false,
        error: "Conversation not found or access denied"
      });
    }

    // Build query
    const query = {
      conversationId,
      isDeleted: false,
      deletedFor: { $ne: req.user._id }
    };

    // Pagination: Get messages before a specific message
    if (before) {
      const beforeMessage = await Message.findById(before);
      if (beforeMessage) {
        query.createdAt = { $lt: beforeMessage.createdAt };
      }
    }

    // Fetch messages
    const messages = await Message.find(query)
      .populate("sender", "username avatar")
      .sort({ createdAt: -1 })
      .limit(parseInt(limit));

    // Get unread count
    const unreadCount = await Message.getUnreadCount(req.user._id, conversationId);

    res.json({
      success: true,
      data: messages.reverse(), // Oldest first for display
      unreadCount,
      hasMore: messages.length === parseInt(limit)
    });

  } catch (error) {
    console.error("Error fetching messages:", error);
    res.status(500).json({
      success: false,
      error: "Failed to fetch messages"
    });
  }
};

// Delete message (soft delete)
exports.deleteMessage = async (req, res) => {
  try {
    const { messageId } = req.params;
    const { deleteForEveryone = false } = req.body;

    const message = await Message.findById(messageId);

    if (!message) {
      return res.status(404).json({
        success: false,
        error: "Message not found"
      });
    }

    // Verify sender
    if (message.sender.toString() !== req.user._id.toString()) {
      return res.status(403).json({
        success: false,
        error: "You can only delete your own messages"
      });
    }

    if (deleteForEveryone) {
      // Delete for everyone (within time limit, e.g., 1 hour)
      const oneHourAgo = new Date(Date.now() - 60 * 60 * 1000);
      
      if (message.createdAt < oneHourAgo) {
        return res.status(400).json({
          success: false,
          error: "Can only delete for everyone within 1 hour of sending"
        });
      }

      message.isDeleted = true;
      await message.save();

      // Notify via WebSocket
      const io = req.app.get("io");
      io.to(message.conversationId.toString()).emit("message_deleted", {
        messageId: message._id,
        conversationId: message.conversationId
      });

    } else {
      // Delete for me only
      if (!message.deletedFor.includes(req.user._id)) {
        message.deletedFor.push(req.user._id);
        await message.save();
      }
    }

    res.json({
      success: true,
      message: "Message deleted"
    });

  } catch (error) {
    console.error("Error deleting message:", error);
    res.status(500).json({
      success: false,
      error: "Failed to delete message"
    });
  }
};
```

### HTTP Controllers for Conversations

**File**: `src/controllers/conversation.controller.js`

```js
const Conversation = require("../models/Conversation.model");
const Message = require("../models/Message.model");

// Get user's conversations
exports.getConversations = async (req, res) => {
  try {
    const conversations = await Conversation.find({
      participants: req.user._id,
      isActive: true
    })
      .populate("participants", "username avatar isOnline lastSeen")
      .populate("lastMessage.sender", "username")
      .sort({ "lastMessage.timestamp": -1 });

    // Get unread counts for each conversation
    const conversationsWithUnread = await Promise.all(
      conversations.map(async (conv) => {
        const unreadCount = await Message.getUnreadCount(req.user._id, conv._id);
        return {
          ...conv.toObject(),
          unreadCount
        };
      })
    );

    res.json({
      success: true,
      data: conversationsWithUnread
    });

  } catch (error) {
    console.error("Error fetching conversations:", error);
    res.status(500).json({
      success: false,
      error: "Failed to fetch conversations"
    });
  }
};

// Create group conversation
exports.createGroupConversation = async (req, res) => {
  try {
    const { name, participantIds } = req.body;

    // Add creator to participants
    const allParticipants = [...new Set([req.user._id, ...participantIds])];

    if (allParticipants.length < 3) {
      return res.status(400).json({
        success: false,
        error: "Group must have at least 3 participants"
      });
    }

    const conversation = await Conversation.create({
      type: "group",
      name,
      participants: allParticipants,
      admin: req.user._id
    });

    await conversation.populate("participants", "username avatar");

    // Notify participants via WebSocket
    const io = req.app.get("io");
    for (const participantId of participantIds) {
      const User = require("../models/User.model");
      const user = await User.findById(participantId);
      
      if (user && user.isOnline && user.socketId) {
        io.to(user.socketId).emit("added_to_group", {
          conversation: conversation.toObject()
        });
      }
    }

    res.status(201).json({
      success: true,
      data: conversation
    });

  } catch (error) {
    console.error("Error creating group:", error);
    res.status(500).json({
      success: false,
      error: "Failed to create group"
    });
  }
};

// Add participant to group
exports.addParticipant = async (req, res) => {
  try {
    const { conversationId } = req.params;
    const { userId } = req.body;

    const conversation = await Conversation.findById(conversationId);

    if (!conversation) {
      return res.status(404).json({
        success: false,
        error: "Conversation not found"
      });
    }

    // Verify admin
    if (conversation.admin.toString() !== req.user._id.toString()) {
      return res.status(403).json({
        success: false,
        error: "Only admin can add participants"
      });
    }

    // Add participant
    if (!conversation.participants.includes(userId)) {
      conversation.participants.push(userId);
      await conversation.save();

      // Notify via WebSocket
      const io = req.app.get("io");
      io.to(conversationId).emit("participant_added", {
        conversationId,
        userId
      });

      // Notify new participant
      const User = require("../models/User.model");
      const user = await User.findById(userId);
      
      if (user && user.isOnline && user.socketId) {
        io.to(user.socketId).emit("added_to_group", {
          conversation: await conversation.populate("participants", "username avatar")
        });
      }
    }

    res.json({
      success: true,
      data: conversation
    });

  } catch (error) {
    console.error("Error adding participant:", error);
    res.status(500).json({
      success: false,
      error: "Failed to add participant"
    });
  }
};
```

### File Upload Route

**File**: `src/routes/upload.routes.js`

```js
const express = require("express");
const multer = require("multer");
const path = require("path");
const authMiddleware = require("../middlewares/auth.middleware");

const router = express.Router();

// Configure multer storage
const storage = multer.diskStorage({
  destination: (req, file, cb) => {
    cb(null, "uploads/");
  },
  filename: (req, file, cb) => {
    const uniqueSuffix = `${Date.now()}-${Math.round(Math.random() * 1E9)}`;
    const ext = path.extname(file.originalname);
    const name = path.basename(file.originalname, ext);
    cb(null, `${name}-${uniqueSuffix}${ext}`);
  }
});

// File filter
const fileFilter = (req, file, cb) => {
  const allowedTypes = /jpeg|jpg|png|gif|pdf|doc|docx|mp4|mp3|wav/;
  const extname = allowedTypes.test(path.extname(file.originalname).toLowerCase());
  const mimetype = allowedTypes.test(file.mimetype);

  if (extname && mimetype) {
    cb(null, true);
  } else {
    cb(new Error("Invalid file type"));
  }
};

const upload = multer({
  storage,
  limits: {
    fileSize: 50 * 1024 * 1024 // 50MB
  },
  fileFilter
});

// Upload endpoint
router.post("/", authMiddleware, upload.single("file"), (req, res) => {
  try {
    if (!req.file) {
      return res.status(400).json({
        success: false,
        error: "No file uploaded"
      });
    }

    const fileUrl = `/uploads/${req.file.filename}`;
    
    // Determine message type based on mimetype
    let messageType = "file";
    if (req.file.mimetype.startsWith("image/")) {
      messageType = "image";
    } else if (req.file.mimetype.startsWith("video/")) {
      messageType = "video";
    } else if (req.file.mimetype.startsWith("audio/")) {
      messageType = "audio";
    }

    res.json({
      success: true,
      file: {
        url: fileUrl,
        filename: req.file.originalname,
        size: req.file.size,
        mimetype: req.file.mimetype,
        messageType
      }
    });

  } catch (error) {
    console.error("Upload error:", error);
    res.status(500).json({
      success: false,
      error: "Failed to upload file"
    });
  }
});

module.exports = router;
```

### Server Integration

**File**: `src/server.js`

```js
require("dotenv").config();
const http = require("http");
const app = require("./app");
const connectDB = require("./config/database");
const setupSocketIO = require("./config/socketio");

const PORT = process.env.PORT || 3000;

// Create HTTP server
const server = http.createServer(app);

// Setup Socket.io
const io = setupSocketIO(server);

// Make io accessible in routes/controllers
app.set("io", io);

// Connect to MongoDB and start server
connectDB().then(() => {
  server.listen(PORT, () => {
    console.log(`
    ╔════════════════════════════════════════╗
    ║  Chat Server Running                   ║
    ║  Port: ${PORT}                         ║
    ║  Environment: ${process.env.NODE_ENV}  ║
    ╚════════════════════════════════════════╝
    `);
  });
});
```

**File**: `src/app.js`

```js
const express = require("express");
const cors = require("cors");
const path = require("path");

// Routes
const uploadRoutes = require("./routes/upload.routes");
const messageRoutes = require("./routes/message.routes");
const conversationRoutes = require("./routes/conversation.routes");

const app = express();

// Middleware
app.use(cors({
  origin: process.env.CLIENT_URL || "http://localhost:5173",
  credentials: true
}));

app.use(express.json());

// Serve uploaded files
app.use("/uploads", express.static(path.join(__dirname, "../uploads")));

// Routes
app.use("/api/upload", uploadRoutes);
app.use("/api/messages", messageRoutes);
app.use("/api/conversations", conversationRoutes);

// Health check
app.get("/health", (req, res) => {
  res.json({ status: "ok", timestamp: new Date() });
});

module.exports = app;
```

### Client Usage Example

```js
import { io } from "socket.io-client";

// Connect
const socket = io("http://localhost:3000", {
  auth: {
    token: localStorage.getItem("token")
  }
});

// Listen for connection
socket.on("connect", () => {
  console.log("Connected to chat server");
  
  // Join conversation
  socket.emit("join_conversation", conversationId, (response) => {
    if (response.success) {
      console.log("Joined conversation");
    }
  });
});

// Listen for new messages
socket.on("new_message", ({ conversationId, message }) => {
  console.log("New message:", message);
  // Update UI with new message
});

// Send text message
function sendMessage(recipientId, content) {
  socket.emit("send_direct_message", {
    recipientId,
    content,
    messageType: "text"
  }, (response) => {
    if (response.success) {
      console.log("Message sent:", response.message);
    }
  });
}

// Send file message
async function sendFile(recipientId, file, text) {
  // 1. Upload file via HTTP
  const formData = new FormData();
  formData.append("file", file);
  
  const uploadRes = await fetch("http://localhost:3000/api/upload", {
    method: "POST",
    headers: {
      "Authorization": `Bearer ${localStorage.getItem("token")}`
    },
    body: formData
  });
  
  const { file: fileData } = await uploadRes.json();
  
  // 2. Send message via WebSocket
  socket.emit("send_direct_message", {
    recipientId,
    content: text,
    messageType: fileData.messageType,
    file: fileData
  }, (response) => {
    if (response.success) {
      console.log("File message sent:", response.message);
    }
  });
}

// Typing indicator
let typingTimeout;
const input = document.getElementById("message-input");

input.addEventListener("input", () => {
  socket.emit("typing_start", { conversationId });
  
  clearTimeout(typingTimeout);
  typingTimeout = setTimeout(() => {
    socket.emit("typing_stop", { conversationId });
  }, 1000);
});

// Listen for typing
socket.on("user_typing", ({ conversationId, username }) => {
  console.log(`${username} is typing...`);
  // Show typing indicator in UI
});

socket.on("user_stopped_typing", ({ conversationId, userId }) => {
  // Hide typing indicator
});

// Mark as read
socket.emit("mark_as_read", { messageId }, (response) => {
  if (response.success) {
    console.log("Message marked as read");
  }
});

// Listen for read receipts
socket.on("message_read", ({ messageId, readBy, readAt }) => {
  console.log(`Message ${messageId} read by ${readBy}`);
  // Update UI to show read status
});

// Online/offline status
socket.on("user_online", ({ userId, username }) => {
  console.log(`${username} is online`);
  // Update UI
});

socket.on("user_offline", ({ userId }) => {
  console.log(`User ${userId} is offline`);
  // Update UI
});
```

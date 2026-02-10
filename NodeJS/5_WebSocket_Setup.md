# 5. Real-Time Communication with Socket.io

## Why WebSockets?

**HTTP**: Client asks, server responds, connection closes. Requires repeated polling for updates.

**WebSocket**: Persistent two-way connection. Server pushes updates instantly without client asking. Essential for real-time features like chat.

---

## 1. Understanding Socket.io (Theory)

### 1.1 Socket Connection Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Initial Connection                           │
└─────────────────────────────────────────────────────────────────────┘

   CLIENT (Browser)                      SERVER (Node.js)
┌──────────────────┐                 ┌────────────────────┐
│ User opens app   │                 │                    │
│       │          │                 │  Waiting for       │
│       ▼          │                 │  connections       │
│ socket = io(url, │                 │       │            │
│   { auth: token })─────────────────┼──────►│            │
│                  │  Sends JWT      │       │            │
│                  │                 │       ▼            │
│                  │                 │  io.use() runs     │
│                  │                 │  Verify token      │
│                  │                 │  Get user from DB  │
│                  │                 │  socket.userId =   │
│                  │                 │    "user123"       │
│                  │                 │       │            │
│                  │◄────────────────┼───────┘            │
│  Connection OK   │  socketId       │                    │
│  socket.id =     │  assigned       │  Store mapping:    │
│  "abc456"        │                 │  userId → socketId │
│       │          │                 │       │            │
│       ▼          │                 │       ▼           │
│  Ready to send/  │                 │  Ready to handle   │
│  receive         │                 │  events            │
└──────────────────┘                 └────────────────────┘
```

### 1.2 Direct Message Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Alice sends message to Bob                       │
└─────────────────────────────────────────────────────────────────────┘

ALICE (socket_abc)              SERVER                  BOB (socket_xyz)
      │                            │                            │
      │ Click "Bob" in list        │                            │
      │ Type "Hi Bob"              │                            │
      │ Click Send                 │                            │
      │                            │                            │
      │ emit("send_direct_message",│                            │
      │   {recipientId, content})  │                            │
      ├───────────────────────────►│                            │
      │                            │                            │
      │                            │ 1. Get sender from         │
      │                            │    socket.userId           │
      │                            │ 2. Save to MongoDB         │
      │                            │ 3. Find Bob's socket       │
      │                            │    activeUsers.get(bobId)  │
      │                            │    = socket_xyz            │
      │                            │ 4. Send to Bob             │
      │                            │    io.to(socket_xyz)       │
      │                            │      .emit("new_message")  │
      │                            │                            │
      │                            ├───────────────────────────►│
      │                            │                            │ Message appears
      │◄───────────────────────────┤                            │
      │ callback({success: true})  │                            │
      │                            │                            │
   Shows ✓                                                Shows message
```

### 1.3 Group Chat Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│                 Team Chat (3 members in room)                       │
└─────────────────────────────────────────────────────────────────────┘

  ALICE                    SERVER                    BOB & CHARLIE
    │                         │                            │
    │ Click "Team Chat"       │                            │
    │ emit("join_group",      │                            │
    │   conversationId)       │                            │
    ├────────────────────────►│                            │
    │                         │ socket.join(convId)        │
    │                         │ Room now has:              │
    │                         │  [socket_abc,              │
    │                         │   socket_xyz,              │
    │                         │   socket_def]              │
    │                         │                            │
    │ Type "Team meeting 3pm" │                            │
    │ emit("send_group_msg")  │                            │
    ├────────────────────────►│                            │
    │                         │ io.to(conversationId)      │
    │                         │   .emit("new_group_msg")   │
    │                         │        │                   │
    │                         │        ├──────────────────►│ Bob receives
    │                         │        └──────────────────►│ Charlie receives
    │◄────────────────────────┤                            │
    │ Alice also receives     │                            │
    │ (included in broadcast) │                            │
```

### 1.4 File Sharing Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│              Sending image.jpg to recipient                         │
└─────────────────────────────────────────────────────────────────────┘

  CLIENT                         SERVER
    │                               │
    │ Select image.jpg              │
    │                               │
    │ 1. HTTP POST /api/upload      │
    │    (with file in FormData)    │
    ├──────────────────────────────►│
    │                               │ Save to /uploads/
    │                               │ Generate URL
    │                               │
    │◄──────────────────────────────┤
    │ { url: "/uploads/123.jpg",    │
    │   messageType: "image" }      │
    │                               │
    │ 2. emit("send_direct_message",│
    │    { recipientId,             │
    │      content: "Check this!",  │
    │      file: {url, type} })     │
    ├──────────────────────────────►│
    │                               │ Save message with
    │                               │ file reference
    │                               │ Send to recipient
    │                               │
    
Recipient receives: {
  content: "Check this!",
  file: { url: "/uploads/123.jpg", type: "image" }
}

Display: <img src="/uploads/123.jpg" />
```

---

## 2. Project Structure & Startup

### 2.1 Folder Structure

```text
chat-app/
├── src/
│   ├── server.js                # Entry point
│   ├── app.js                   # Express config
│   ├── config/
│   │   ├── database.js          # MongoDB connection
│   │   └── socketio.js          # Socket.io setup
│   ├── models/
│   │   ├── User.model.js
│   │   ├── Conversation.model.js
│   │   └── Message.model.js
│   ├── middlewares/
│   │   └── auth.middleware.js   # JWT verification
│   ├── socket/
│   │   ├── directMessages.js
│   │   └── groupMessages.js
│   └── routes/
│       └── upload.routes.js
├── uploads/
├── .env
└── package.json
```

### 2.2 Server Entry Point

**File**: `src/server.js`

```js
require("dotenv").config();
const http = require("http");
const app = require("./app");
const connectDB = require("./config/database");
const setupSocketIO = require("./config/socketio");

const server = http.createServer(app);

// CRITICAL ORDER: Database → Socket.io → Listen
connectDB()
  .then(() => {
    console.log("✓ MongoDB connected");
    
    const io = setupSocketIO(server);
    app.set("io", io);  // Make accessible in routes
    
    server.listen(3000, () => {
      console.log("✓ Server running on port 3000");
    });
  })
  .catch((error) => {
    console.error("✗ MongoDB failed:", error);
    process.exit(1);
  });
```

### 2.3 Express App

**File**: `src/app.js`

```js
const express = require("express");
const cors = require("cors");
const path = require("path");
const uploadRoutes = require("./routes/upload.routes");

const app = express();

app.use(cors({ origin: "http://localhost:5173", credentials: true }));
app.use(express.json());
app.use("/uploads", express.static(path.join(__dirname, "../uploads")));

app.use("/api/upload", uploadRoutes);

module.exports = app;
```

---

## 3. Database Schemas

### 3.1 User Schema

**File**: `src/models/User.model.js`

```js
const mongoose = require("mongoose");

const userSchema = new mongoose.Schema({
  username: { type: String, required: true, unique: true },
  email: { type: String, required: true, unique: true },
  passwordHash: { type: String, required: true },
  avatar: String,
  
  // For presence tracking
  isOnline: { type: Boolean, default: false },
  lastSeen: { type: Date, default: Date.now },
  socketId: String  // Current socket ID when connected
}, { timestamps: true });

module.exports = mongoose.model("User", userSchema);
```

### 3.2 Conversation Schema

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
  
  // Group-specific
  name: String,          // "Team Chat"
  avatar: String,
  admin: {
    type: mongoose.Schema.Types.ObjectId,
    ref: "User"
  },
  
  // Denormalized for conversation list
  lastMessage: {
    content: String,
    sender: mongoose.Schema.Types.ObjectId,
    timestamp: Date,
    messageType: String
  }
}, { timestamps: true });

conversationSchema.index({ participants: 1, type: 1 });

module.exports = mongoose.model("Conversation", conversationSchema);
```

### 3.3 Message Schema

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
  content: String,
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
    user: mongoose.Schema.Types.ObjectId,
    readAt: Date
  }]
}, { timestamps: true });

messageSchema.index({ conversationId: 1, createdAt: -1 });

module.exports = mongoose.model("Message", messageSchema);
```

### 3.4 Database Connection

**File**: `src/config/database.js`

```js
const mongoose = require("mongoose");

const connectDB = async () => {
  try {
    await mongoose.connect(process.env.MONGO_URI, {
      maxPoolSize: 10,
      serverSelectionTimeoutMS: 5000
    });
    
    mongoose.connection.on("error", (err) => {
      console.error("MongoDB error:", err);
    });
    
  } catch (error) {
    console.error("MongoDB connection error:", error);
    throw error;
  }
};

module.exports = connectDB;
```

---

## 4. Socket.io Setup with Authentication

**File**: `src/config/socketio.js`

```js
const { Server } = require("socket.io");
const jwt = require("jsonwebtoken");
const User = require("../models/User.model");
const handleDirectMessages = require("../socket/directMessages");
const handleGroupMessages = require("../socket/groupMessages");

function setupSocketIO(server) {
  const io = new Server(server, {
    cors: { origin: "http://localhost:5173", credentials: true }
  });

  // In-memory mapping: userId → socketId
  const activeUsers = new Map();

  // ============================================
  // AUTHENTICATION MIDDLEWARE
  // ============================================
  // Runs before connection event
  // Verifies JWT and attaches user info to socket
  
  io.use(async (socket, next) => {
    try {
      // Token sent from client: io(url, { auth: { token: "..." } })
      const token = socket.handshake.auth.token;
      
      if (!token) {
        return next(new Error("No token provided"));
      }

      // Verify JWT
      const decoded = jwt.verify(token, process.env.JWT_SECRET);
      // decoded = { id: "user123", email: "...", iat: ..., exp: ... }
      
      const user = await User.findById(decoded.id);
      if (!user) {
        return next(new Error("User not found"));
      }

      // Attach to socket (available in all event handlers)
      socket.userId = user._id.toString();
      socket.username = user.username;
      
      next();  // Allow connection
      
    } catch (error) {
      next(new Error("Authentication failed"));
    }
  });

  // ============================================
  // CONNECTION EVENT
  // ============================================
  // Fires when client successfully connects
  
  io.on("connection", async (socket) => {
    console.log(`✓ ${socket.username} connected (${socket.id})`);

    // Store mapping: userId → socketId
    activeUsers.set(socket.userId, socket.id);

    // Update database
    await User.findByIdAndUpdate(socket.userId, {
      isOnline: true,
      socketId: socket.id
    });

    // Notify all clients
    io.emit("user_online", {
      userId: socket.userId,
      username: socket.username
    });

    // Initialize event handlers
    handleDirectMessages(io, socket, activeUsers);
    handleGroupMessages(io, socket);

    // ============================================
    // DISCONNECT EVENT
    // ============================================
    
    socket.on("disconnect", async () => {
      console.log(`✗ ${socket.username} disconnected`);

      activeUsers.delete(socket.userId);

      await User.findByIdAndUpdate(socket.userId, {
        isOnline: false,
        lastSeen: new Date(),
        socketId: null
      });

      io.emit("user_offline", { userId: socket.userId });
    });
  });

  io.activeUsers = activeUsers;
  return io;
}

module.exports = setupSocketIO;
```

---

## 5. Direct Message Handler

**File**: `src/socket/directMessages.js`

```js
const Message = require("../models/Message.model");
const Conversation = require("../models/Conversation.model");
const User = require("../models/User.model");

function handleDirectMessages(io, socket, activeUsers) {
  
  // ============================================
  // SEND DIRECT MESSAGE
  // ============================================
  // Triggered: User types message and clicks send
  // Client: socket.emit("send_direct_message", { recipientId, content }, callback)
  
  socket.on("send_direct_message", async (data, callback) => {
    try {
      const { recipientId, content, file } = data;
      // recipientId: Selected from UI contact list
      // content: Text from input field
      // file: Optional, from upload (see file handler section)
      
      const senderId = socket.userId;  // From authentication middleware
      
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

      // Save message to database
      const message = await Message.create({
        conversationId: conversation._id,
        sender: senderId,
        content: content || "",
        messageType: file ? file.messageType : "text",
        file: file,
        status: "sent"
      });

      // Populate sender info (username, avatar)
      await message.populate("sender", "username avatar");

      // Update conversation last message (for UI preview)
      conversation.lastMessage = {
        content: content || "File",
        sender: senderId,
        timestamp: message.createdAt,
        messageType: message.messageType
      };
      await conversation.save();

      // Try to deliver if recipient online
      const recipientSocketId = activeUsers.get(recipientId);
      // activeUsers: Map<userId, socketId> from socketio.js
      
      if (recipientSocketId) {
        // Recipient online - send immediately
        io.to(recipientSocketId).emit("new_message", {
          conversationId: conversation._id,
          message: message
        });
        
        message.status = "delivered";
        await message.save();
      }
      // If offline, stays "sent" - delivered when they connect

      // Acknowledge sender
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

  // ============================================
  // MARK AS READ
  // ============================================
  // Triggered: User opens chat or scrolls to message
  // Client: socket.emit("mark_as_read", { messageId })
  
  socket.on("mark_as_read", async ({ messageId }, callback) => {
    try {
      const message = await Message.findById(messageId);
      if (!message) {
        return callback({ success: false, error: "Message not found" });
      }

      // Check if already read
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

---

## 6. Group Message Handler

**File**: `src/socket/groupMessages.js`

```js
const Message = require("../models/Message.model");
const Conversation = require("../models/Conversation.model");

function handleGroupMessages(io, socket) {
  
  // ============================================
  // JOIN GROUP
  // ============================================
  // Triggered: User clicks on group in group list
  // Client: socket.emit("join_group", conversationId, callback)
  
  socket.on("join_group", async (conversationId, callback) => {
    try {
      // conversationId: Group's unique ID from database
      // Passed from UI when user clicks group card
      
      // Verify user is a participant
      const conversation = await Conversation.findOne({
        _id: conversationId,
        type: "group",
        participants: socket.userId  // Check membership
      });

      if (!conversation) {
        return callback({ success: false, error: "Not a member" });
      }

      // Join Socket.io room (room name = conversationId)
      socket.join(conversationId);
      // Now this socket receives all messages to this room

      // Notify others in group
      socket.to(conversationId).emit("user_joined_group", {
        userId: socket.userId,
        username: socket.username,
        conversationId
      });

      callback({ success: true });

    } catch (error) {
      callback({ success: false, error: error.message });
    }
  });

  // ============================================
  // LEAVE GROUP
  // ============================================
  // Triggered: User closes group chat or navigates away
  // Client: socket.emit("leave_group", conversationId)
  
  socket.on("leave_group", (conversationId) => {
    socket.leave(conversationId);
    
    socket.to(conversationId).emit("user_left_group", {
      userId: socket.userId,
      username: socket.username,
      conversationId
    });
  });

  // ============================================
  // SEND GROUP MESSAGE
  // ============================================
  // Triggered: User types and sends in group chat
  // Client: socket.emit("send_group_message", { conversationId, content }, callback)
  
  socket.on("send_group_message", async (data, callback) => {
    try {
      const { conversationId, content, file } = data;

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
        conversationId,
        sender: socket.userId,
        content: content || "",
        messageType: file ? file.messageType : "text",
        file,
        status: "sent"
      });

      await message.populate("sender", "username avatar");

      // Update conversation
      conversation.lastMessage = {
        content: content || "File",
        sender: socket.userId,
        timestamp: message.createdAt,
        messageType: message.messageType
      };
      await conversation.save();

      // Broadcast to ALL members (including sender)
      io.to(conversationId).emit("new_group_message", {
        conversationId,
        message
      });

      callback({ success: true, message });

    } catch (error) {
      callback({ success: false, error: error.message });
    }
  });

  // ============================================
  // TYPING INDICATOR
  // ============================================
  
  socket.on("typing_start", ({ conversationId }) => {
    socket.to(conversationId).emit("user_typing", {
      userId: socket.userId,
      username: socket.username,
      conversationId
    });
  });

  socket.on("typing_stop", ({ conversationId }) => {
    socket.to(conversationId).emit("user_stopped_typing", {
      userId: socket.userId,
      conversationId
    });
  });
}

module.exports = handleGroupMessages;
```

---

## 7. File Upload Handler

**File**: `src/routes/upload.routes.js`

```js
const express = require("express");
const multer = require("multer");
const path = require("path");
const authMiddleware = require("../middlewares/auth.middleware");

const router = express.Router();

// Configure multer
const storage = multer.diskStorage({
  destination: (req, file, cb) => cb(null, "uploads/"),
  filename: (req, file, cb) => {
    const uniqueName = `${Date.now()}-${file.originalname}`;
    cb(null, uniqueName);
  }
});

const upload = multer({
  storage,
  limits: { fileSize: 50 * 1024 * 1024 },  // 50MB
  fileFilter: (req, file, cb) => {
    const allowed = /jpeg|jpg|png|gif|pdf|doc|docx|mp4|mp3/;
    const ext = path.extname(file.originalname).toLowerCase();
    if (allowed.test(ext.substring(1))) {
      cb(null, true);
    } else {
      cb(new Error("Invalid file type"));
    }
  }
});

// Upload endpoint
// POST /api/upload
// Headers: Authorization: Bearer <token>
// Body: FormData with "file" field
router.post("/", authMiddleware, upload.single("file"), (req, res) => {
  try {
    if (!req.file) {
      return res.status(400).json({ success: false, error: "No file" });
    }

    const fileUrl = `/uploads/${req.file.filename}`;
    
    // Determine message type
    let messageType = "file";
    if (req.file.mimetype.startsWith("image/")) messageType = "image";
    else if (req.file.mimetype.startsWith("video/")) messageType = "video";
    else if (req.file.mimetype.startsWith("audio/")) messageType = "audio";

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
    res.status(500).json({ success: false, error: error.message });
  }
});

module.exports = router;
```

**File**: `src/middlewares/auth.middleware.js`

```js
const jwt = require("jsonwebtoken");
const User = require("../models/User.model");

async function authMiddleware(req, res, next) {
  try {
    // Extract token from: Authorization: Bearer <token>
    const authHeader = req.headers.authorization;
    
    if (!authHeader || !authHeader.startsWith("Bearer ")) {
      return res.status(401).json({ success: false, error: "No token" });
    }

    const token = authHeader.split(" ")[1];
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    
    const user = await User.findById(decoded.id);
    if (!user) {
      return res.status(401).json({ success: false, error: "User not found" });
    }

    req.user = user;  // Attach to request
    next();

  } catch (error) {
    return res.status(401).json({ success: false, error: "Invalid token" });
  }
}

module.exports = authMiddleware;
```

### Sending File Message

**Client flow**:

```js
// 1. User selects file
async function sendFileMessage(recipientId, file) {
  // Upload via HTTP
  const formData = new FormData();
  formData.append("file", file);

  const response = await fetch("http://localhost:3000/api/upload", {
    method: "POST",
    headers: { "Authorization": `Bearer ${token}` },
    body: formData
  });

  const { file: fileData } = await response.json();
  // fileData = { url, filename, size, mimetype, messageType }

  // Send message via Socket.io
  socket.emit("send_direct_message", {
    recipientId,
    content: "Sent a file",
    file: fileData
  }, (response) => {
    if (response.success) {
      console.log("✓ File message sent");
    }
  });
}
```

**Displaying files**:

```js
socket.on("new_message", ({ message }) => {
  if (message.messageType === "image") {
    display(`<img src="${message.file.url}" />`);
  } else if (message.messageType === "text") {
    display(message.content);
  } else {
    display(`<a href="${message.file.url}">${message.file.filename}</a>`);
  }
});
```

---

## 8. Client Implementation

This section shows the core Socket.io client setup and message handling patterns. As a backend developer, you need to understand how the frontend connects and interacts with your Socket.io server.

### 8.1 Initial Setup & Connection

**Installation**:
```bash
npm install socket.io-client
```

**Basic connection with authentication**:

```js
import { io } from "socket.io-client";

// ============================================
// CONNECTION SETUP
// ============================================

// Get JWT token (stored after login)
const token = localStorage.getItem("token");

// Connect to Socket.io server
// The token is sent in handshake and verified by io.use() middleware on server
const socket = io("http://localhost:3000", {
  auth: {
    token: token
  },
  
  // Optional: reconnection settings
  reconnection: true,
  reconnectionAttempts: 5,
  reconnectionDelay: 1000
});

// ============================================
// CONNECTION EVENTS
// ============================================

socket.on("connect", () => {
  console.log("✓ Connected to server");
  console.log("Socket ID:", socket.id);
  // socket.id is auto-generated by server (e.g., "abc123xyz")
  // This is NOT the same as userId - it's the socket connection ID
});

socket.on("disconnect", (reason) => {
  console.log("✗ Disconnected:", reason);
  // Reasons: "io server disconnect", "io client disconnect", "ping timeout", etc.
});

socket.on("connect_error", (error) => {
  console.error("Connection failed:", error.message);
  // This fires if JWT token is invalid or missing
  // Typically redirect to login page here
  window.location.href = "/login";
});

// ============================================
// PRESENCE EVENTS
// ============================================

// Server broadcasts when someone comes online
socket.on("user_online", ({ userId, username }) => {
  console.log(`${username} is now online`);
  // Update UI: show green dot, update contact list, etc.
  updateUserStatus(userId, "online");
});

// Server broadcasts when someone goes offline
socket.on("user_offline", ({ userId }) => {
  console.log(`User ${userId} went offline`);
  // Update UI: show gray dot, display last seen time
  updateUserStatus(userId, "offline");
});
```

---

### 8.2 Sending Direct Messages

```js
// ============================================
// SEND MESSAGE FUNCTION
// ============================================
// Called when user types message and clicks send button

function sendDirectMessage(recipientId, messageText) {
  // recipientId: User ID of the person to send to (e.g., "user_bob123")
  //              Comes from UI when user selects contact from list
  // messageText: Content from input field
  
  socket.emit("send_direct_message", {
    recipientId: recipientId,
    content: messageText
  }, (response) => {
    // This callback receives server's response
    // Server calls: callback({ success: true, message: {...}, status: "delivered" })
    
    if (response.success) {
      console.log("✓ Message sent successfully");
      console.log("Status:", response.status);  // "sent" or "delivered"
      
      // Add message to UI with status indicator
      displayMyMessage(response.message, response.status);
      
      // Clear input field
      document.getElementById("message-input").value = "";
      
    } else {
      console.error("✗ Failed to send:", response.error);
      // Show error notification to user
      showError("Failed to send message: " + response.error);
    }
  });
}

// ============================================
// EXAMPLE USAGE
// ============================================

// User clicks send button
document.getElementById("send-btn").addEventListener("click", () => {
  const input = document.getElementById("message-input");
  const messageText = input.value.trim();
  
  if (!messageText) return;  // Don't send empty messages
  
  // Get current chat recipient (stored when user opened chat)
  const recipientId = getCurrentRecipientId();
  
  sendDirectMessage(recipientId, messageText);
});

// Send on Enter key
document.getElementById("message-input").addEventListener("keypress", (e) => {
  if (e.key === "Enter" && !e.shiftKey) {
    e.preventDefault();
    document.getElementById("send-btn").click();
  }
});
```

---

### 8.3 Receiving Direct Messages

```js
// ============================================
// RECEIVE MESSAGE EVENT
// ============================================
// Server emits this when someone sends you a message
// Server code: io.to(recipientSocketId).emit("new_message", { conversationId, message })

socket.on("new_message", ({ conversationId, message }) => {
  // conversationId: ID of the conversation this message belongs to
  // message: {
  //   _id: "msg123",
  //   sender: { _id: "user_alice", username: "Alice", avatar: "/avatars/alice.jpg" },
  //   content: "Hello!",
  //   messageType: "text",
  //   status: "delivered",
  //   createdAt: "2024-01-15T10:30:00Z"
  // }
  
  console.log("📨 New message from:", message.sender.username);
  console.log("Content:", message.content);
  
  // Check if this message is for the currently open conversation
  const currentConversationId = getCurrentConversationId();
  
  if (conversationId === currentConversationId) {
    // Message is for the chat currently being viewed
    displayReceivedMessage(message);
    
    // Auto-mark as read since user is viewing it
    socket.emit("mark_as_read", { 
      messageId: message._id 
    }, (response) => {
      if (response.success) {
        console.log("✓ Marked as read");
      }
    });
    
  } else {
    // Message from another conversation (not currently viewing)
    
    // Show notification
    showNotification(
      `New message from ${message.sender.username}`,
      message.content
    );
    
    // Update unread badge on conversation list
    incrementUnreadCount(conversationId);
    
    // Play notification sound
    playNotificationSound();
  }
});
```

---

### 8.4 Marking Messages as Read

```js
// ============================================
// MARK AS READ
// ============================================
// Called when user views a message

function markMessageAsRead(messageId) {
  // messageId: ID of the message to mark as read
  //           Stored in DOM when message was displayed
  
  socket.emit("mark_as_read", { 
    messageId: messageId 
  }, (response) => {
    if (response.success) {
      console.log("✓ Message marked as read");
      // No UI update needed - happens via "message_read" event
    } else {
      console.error("✗ Failed to mark as read:", response.error);
    }
  });
}

// ============================================
// AUTO-MARK MESSAGES AS READ
// ============================================
// When user scrolls message into view

const observer = new IntersectionObserver((entries) => {
  entries.forEach(entry => {
    if (entry.isIntersecting) {
      // Message is visible on screen
      const messageElement = entry.target;
      const messageId = messageElement.dataset.messageId;
      const isRead = messageElement.dataset.read === "true";
      
      if (!isRead) {
        markMessageAsRead(messageId);
        messageElement.dataset.read = "true";
      }
    }
  });
}, { threshold: 0.5 });  // Trigger when 50% visible

// Observe all unread messages
document.querySelectorAll(".message[data-read='false']").forEach(msg => {
  observer.observe(msg);
});
```

---

### 8.5 Receiving Read Receipts

```js
// ============================================
// READ RECEIPT EVENT
// ============================================
// Server emits this when someone reads your message
// Server code: io.to(senderSocketId).emit("message_read", { messageId, readBy, readAt })

socket.on("message_read", ({ messageId, readBy, readAt }) => {
  // messageId: ID of your message that was read
  // readBy: User ID who read it
  // readAt: Timestamp when they read it
  
  console.log(`✓✓ Message ${messageId} was read by ${readBy}`);
  
  // Find the message element in DOM
  const messageElement = document.querySelector(`[data-message-id="${messageId}"]`);
  
  if (messageElement) {
    // Update status icon
    const statusIcon = messageElement.querySelector(".status-icon");
    
    // Change from single checkmark (✓) to double blue checkmark (✓✓)
    statusIcon.textContent = "✓✓";
    statusIcon.classList.remove("delivered");
    statusIcon.classList.add("read");
    
    // Update timestamp to show when read
    const timestamp = messageElement.querySelector(".timestamp");
    timestamp.title = `Read at ${new Date(readAt).toLocaleString()}`;
  }
});
```

---

### 8.6 Group Chat Implementation

```js
// ============================================
// JOIN GROUP
// ============================================
// Called when user clicks on a group in group list

function joinGroup(conversationId) {
  // conversationId: Group's unique ID from database
  //                 Retrieved from API when loading group list
  
  socket.emit("join_group", conversationId, (response) => {
    if (response.success) {
      console.log("✓ Joined group:", conversationId);
      
      // Load message history via HTTP API
      loadGroupMessages(conversationId);
      
      // Update UI to show group chat window
      showGroupChat(conversationId);
      
    } else {
      console.error("✗ Failed to join:", response.error);
      showError("Cannot join group: " + response.error);
    }
  });
}

// ============================================
// SEND GROUP MESSAGE
// ============================================

function sendGroupMessage(conversationId, messageText) {
  socket.emit("send_group_message", {
    conversationId: conversationId,
    content: messageText
  }, (response) => {
    if (response.success) {
      console.log("✓ Group message sent");
      // Message will appear via "new_group_message" event
    } else {
      console.error("✗ Failed:", response.error);
      showError("Failed to send: " + response.error);
    }
  });
}

// ============================================
// RECEIVE GROUP MESSAGES
// ============================================

socket.on("new_group_message", ({ conversationId, message }) => {
  // Check if this is the currently open group
  if (conversationId === getCurrentConversationId()) {
    
    // Check if message is from current user (to avoid duplicate display)
    const myUserId = getCurrentUserId();
    const isMine = message.sender._id === myUserId;
    
    // Display message (will show on right if mine, left if others)
    displayGroupMessage(message, isMine);
    
    // Mark as read if not from me
    if (!isMine) {
      socket.emit("mark_as_read", { messageId: message._id });
    }
    
  } else {
    // Message from another group
    showNotification(`New message in ${getGroupName(conversationId)}`);
    incrementUnreadCount(conversationId);
  }
});

// ============================================
// GROUP MEMBER EVENTS
// ============================================

socket.on("user_joined_group", ({ userId, username, conversationId }) => {
  if (conversationId === getCurrentConversationId()) {
    // Show system message
    displaySystemMessage(`${username} joined the chat`);
  }
});

socket.on("user_left_group", ({ userId, username, conversationId }) => {
  if (conversationId === getCurrentConversationId()) {
    displaySystemMessage(`${username} left the chat`);
  }
});

// ============================================
// LEAVE GROUP
// ============================================
// Called when user closes group chat or navigates away

function leaveGroup(conversationId) {
  socket.emit("leave_group", conversationId);
  console.log("Left group:", conversationId);
}

// Auto-leave when navigating away
window.addEventListener("beforeunload", () => {
  const currentGroup = getCurrentConversationId();
  if (currentGroup) {
    leaveGroup(currentGroup);
  }
});
```

---

### 8.7 Typing Indicators

```js
// ============================================
// SEND TYPING INDICATOR
// ============================================

const messageInput = document.getElementById("message-input");
let typingTimeout;

messageInput.addEventListener("input", () => {
  const conversationId = getCurrentConversationId();
  
  // Emit typing start
  socket.emit("typing_start", { 
    conversationId: conversationId 
  });
  
  // Clear previous timeout
  clearTimeout(typingTimeout);
  
  // After 1 second of no typing, emit typing stop
  typingTimeout = setTimeout(() => {
    socket.emit("typing_stop", { 
      conversationId: conversationId 
    });
  }, 1000);
});

// Also emit stop when user sends message
function sendMessage() {
  // ... send message code ...
  
  // Stop typing indicator
  clearTimeout(typingTimeout);
  socket.emit("typing_stop", { 
    conversationId: getCurrentConversationId() 
  });
}

// ============================================
// RECEIVE TYPING INDICATORS
// ============================================

// Track who is currently typing (for groups with multiple people)
const currentlyTyping = new Set();

socket.on("user_typing", ({ userId, username, conversationId }) => {
  if (conversationId === getCurrentConversationId()) {
    currentlyTyping.add(username);
    updateTypingIndicator();
  }
});

socket.on("user_stopped_typing", ({ userId, conversationId }) => {
  if (conversationId === getCurrentConversationId()) {
    // Find and remove this user from typing set
    const user = findUserById(userId);
    if (user) {
      currentlyTyping.delete(user.username);
      updateTypingIndicator();
    }
  }
});

function updateTypingIndicator() {
  const indicator = document.getElementById("typing-indicator");
  
  if (currentlyTyping.size === 0) {
    indicator.textContent = "";
    indicator.style.display = "none";
  } else if (currentlyTyping.size === 1) {
    const [username] = currentlyTyping;
    indicator.textContent = `${username} is typing...`;
    indicator.style.display = "block";
  } else if (currentlyTyping.size === 2) {
    const [user1, user2] = currentlyTyping;
    indicator.textContent = `${user1} and ${user2} are typing...`;
    indicator.style.display = "block";
  } else {
    indicator.textContent = "Multiple people are typing...";
    indicator.style.display = "block";
  }
}
```

---

### 8.8 File Upload & Send

```js
// ============================================
// FILE UPLOAD FLOW
// ============================================

async function sendFileMessage(recipientId, file) {
  // file: File object from <input type="file">
  
  try {
    // STEP 1: Upload file via HTTP
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
    
    // uploadData.file = {
    //   url: "/uploads/123-image.jpg",
    //   filename: "image.jpg",
    //   size: 204800,
    //   mimetype: "image/jpeg",
    //   messageType: "image"
    // }

    // STEP 2: Send message via Socket.io with file reference
    socket.emit("send_direct_message", {
      recipientId: recipientId,
      content: "",  // Optional caption
      file: uploadData.file  // File metadata from upload
    }, (response) => {
      if (response.success) {
        console.log("✓ File message sent");
        displayMyMessage(response.message, response.status);
      } else {
        console.error("✗ Failed to send:", response.error);
        showError("Failed to send file message");
      }
    });

  } catch (error) {
    console.error("Error uploading file:", error);
    showError("Failed to upload file: " + error.message);
  }
}

// ============================================
// FILE INPUT HANDLER
// ============================================

document.getElementById("file-input").addEventListener("change", (e) => {
  const file = e.target.files[0];
  if (!file) return;
  
  const recipientId = getCurrentRecipientId();
  sendFileMessage(recipientId, file);
  
  // Clear input so same file can be selected again
  e.target.value = "";
});

// ============================================
// DISPLAY RECEIVED FILE
// ============================================

socket.on("new_message", ({ message }) => {
  // Check if message has a file
  if (message.file) {
    displayFileMessage(message);
  } else {
    displayTextMessage(message);
  }
});

function displayFileMessage(message) {
  const messageDiv = document.createElement("div");
  messageDiv.className = "message received";
  
  let content = `<strong>${message.sender.username}:</strong><br>`;
  
  // Display based on file type
  if (message.messageType === "image") {
    content += `<img src="${message.file.url}" 
                     alt="${message.file.filename}" 
                     style="max-width: 300px; cursor: pointer;" 
                     onclick="openImageModal('${message.file.url}')" />`;
  } else if (message.messageType === "video") {
    content += `<video controls style="max-width: 300px;">
                  <source src="${message.file.url}" type="${message.file.mimetype}">
                </video>`;
  } else if (message.messageType === "audio") {
    content += `<audio controls>
                  <source src="${message.file.url}" type="${message.file.mimetype}">
                </audio>`;
  } else {
    // Generic file (PDF, DOC, etc.)
    const fileSize = formatFileSize(message.file.size);
    content += `<a href="${message.file.url}" 
                   download="${message.file.filename}" 
                   class="file-download">
                  📎 ${message.file.filename} (${fileSize})
                </a>`;
  }
  
  messageDiv.innerHTML = content;
  document.getElementById("messages").appendChild(messageDiv);
}

function formatFileSize(bytes) {
  if (bytes < 1024) return bytes + " B";
  if (bytes < 1024 * 1024) return (bytes / 1024).toFixed(1) + " KB";
  return (bytes / (1024 * 1024)).toFixed(1) + " MB";
}
```

---

### 8.9 Connection State Management

```js
// ============================================
// HANDLE RECONNECTION
// ============================================
// Socket.io automatically reconnects, but you need to restore state

socket.on("connect", () => {
  console.log("✓ Connected");
  
  // If user was in a group chat, rejoin
  const currentGroup = getCurrentConversationId();
  if (currentGroup) {
    socket.emit("join_group", currentGroup, (response) => {
      if (response.success) {
        console.log("✓ Rejoined group after reconnection");
      }
    });
  }
  
  // Fetch any messages missed during disconnection
  fetchMissedMessages();
});

socket.on("disconnect", (reason) => {
  console.log("✗ Disconnected:", reason);
  
  // Show connection lost indicator
  showConnectionStatus("disconnected");
  
  if (reason === "io server disconnect") {
    // Server forcefully disconnected (kicked or banned)
    showError("You were disconnected from the server");
    window.location.href = "/login";
  }
  // For other reasons, Socket.io will auto-reconnect
});

socket.on("reconnect_attempt", (attemptNumber) => {
  console.log(`Reconnection attempt ${attemptNumber}`);
  showConnectionStatus("reconnecting");
});

socket.on("reconnect_failed", () => {
  console.error("✗ Reconnection failed");
  showConnectionStatus("failed");
  showError("Cannot connect to server. Please refresh the page.");
});

// ============================================
// SHOW CONNECTION STATUS TO USER
// ============================================

function showConnectionStatus(status) {
  const indicator = document.getElementById("connection-status");
  
  switch (status) {
    case "connected":
      indicator.textContent = "Connected";
      indicator.className = "status-connected";
      break;
    case "disconnected":
      indicator.textContent = "Connection lost";
      indicator.className = "status-disconnected";
      break;
    case "reconnecting":
      indicator.textContent = "Reconnecting...";
      indicator.className = "status-reconnecting";
      break;
    case "failed":
      indicator.textContent = "Connection failed";
      indicator.className = "status-failed";
      break;
  }
}
```

# Project Setup - Building a Production-Ready Node.js Backend

## Overview

This guide covers the essential setup for a professional Node.js backend project. You'll learn to structure your project following industry standards, configure environments properly, and set up a maintainable foundation.

**What you'll build**: A scalable Express.js API server with proper organization, environment management, and RESTful routing.

---

## 1. Initializing a Node.js Project

### Creating Your Project

```bash
# Create project directory
mkdir my-backend-api
cd my-backend-api

# Initialize Node.js project
npm init -y
```

**What happens**: `npm init -y` creates a `package.json` file with default values. The `-y` flag skips interactive questions.

### Understanding package.json

**File**: `package.json`

```json
{
  "name": "my-backend-api",
  "version": "1.0.0",
  "description": "Backend API server",
  "main": "src/server.js",
  "scripts": {
    "start": "node src/server.js",
    "dev": "nodemon src/server.js"
  },
  "keywords": ["api", "backend", "express"],
  "author": "Your Name",
  "license": "MIT",
  "dependencies": {
    "express": "^4.18.2",
    "dotenv": "^16.3.1"
  },
  "devDependencies": {
    "nodemon": "^3.0.1"
  }
}
```

**Key fields explained**:
- `name`: Project identifier (lowercase, no spaces)
- `version`: Follows semantic versioning (MAJOR.MINOR.PATCH)
- `main`: Entry point file
- `scripts`: Custom commands (`npm run dev`)
- `dependencies`: Packages needed in production
- `devDependencies`: Packages only for development

### Semantic Versioning Quick Guide

```
Version: 1.2.3
         │ │ │
         │ │ └─ PATCH: Bug fixes (backward compatible)
         │ └─── MINOR: New features (backward compatible)
         └───── MAJOR: Breaking changes
```

**Caret vs Tilde in dependencies**:
- `^1.2.3` → Allows updates to 1.x.x (minor and patch)
- `~1.2.3` → Allows updates to 1.2.x (patch only)
- `1.2.3` → Exact version (locked)

---

## 2. Essential Dependencies

### Installing Core Packages

```bash
# Production dependencies
npm install express dotenv cors cookie-parser

# Development dependencies
npm install --save-dev nodemon
```

### Why Each Package?

| Package | Purpose | When to Use |
|---------|---------|-------------|
| **express** | Web framework for routing and middleware | Always (core framework) |
| **dotenv** | Loads environment variables from `.env` | Always (config management) |
| **cors** | Handles Cross-Origin Resource Sharing | When frontend is on different domain |
| **cookie-parser** | Parses cookies from requests | When using cookie-based auth |
| **nodemon** | Auto-restarts server on file changes | Development only |

### Optional but Common

```bash
# Install based on needs
npm install jsonwebtoken          # JWT authentication
npm install bcrypt               # Password hashing
npm install mongoose             # MongoDB ODM
npm install express-validator    # Request validation
npm install morgan               # HTTP request logging
```

---

## 3. Project Structure (MVC Pattern)

### What is MVC?

**Simple explanation**: A design pattern that separates your code into three responsibilities:
- **Model**: Data and database logic
- **View**: Presentation layer (in APIs, this is JSON responses)
- **Controller**: Business logic connecting models and views

**Technical explanation**: MVC enforces separation of concerns by isolating data access (Model), request handling (Controller), and response formatting (View). This makes code more maintainable, testable, and scalable.

### Standard Folder Structure

```text
my-backend-api/
├── src/
│   ├── server.js              # Entry point - starts server
│   ├── app.js                 # Express app configuration
│   │
│   ├── config/                # Configuration files
│   │   ├── database.js        # Database connection
│   │   └── jwt.js             # JWT settings
│   │
│   ├── controllers/           # Request handlers (business logic)
│   │   ├── auth.controller.js
│   │   └── user.controller.js
│   │
│   ├── models/                # Database schemas/models
│   │   └── User.model.js
│   │
│   ├── routes/                # Route definitions
│   │   ├── auth.routes.js
│   │   └── user.routes.js
│   │
│   ├── middlewares/           # Custom middleware functions
│   │   ├── auth.middleware.js
│   │   ├── error.middleware.js
│   │   └── validation.middleware.js
│   │
│   ├── services/              # Reusable business logic
│   │   ├── token.service.js
│   │   └── email.service.js
│   │
│   └── utils/                 # Helper functions
│       ├── logger.js
│       └── response.js
│
├── .env                       # Environment variables (local)
├── .env.example              # Template for .env
├── .gitignore                # Git ignore rules
├── package.json              # Project metadata
└── README.md                 # Project documentation
```

### Why This Structure?

**Benefits**:
- **Predictable**: Developers know where to find things
- **Scalable**: Easy to add new features without restructuring
- **Maintainable**: Clear separation makes updates easier
- **Testable**: Isolated concerns simplify unit testing
- **Team-friendly**: Standard structure reduces onboarding time

### When to Add New Folders

```text
Add when needed:

├── validators/        # When validation logic becomes complex
├── constants/         # When you have many app-wide constants
├── types/            # For TypeScript type definitions
├── tests/            # For test files
├── scripts/          # For utility scripts (seed data, migrations)
├── public/           # For static files (images, docs)
└── docs/             # For API documentation
```

---

## 4. Environment Configuration

### What are Environment Variables?

**Simple explanation**: Settings that change based on where your code runs (local computer, testing server, production server).

**Technical explanation**: Environment variables are dynamic values stored outside your codebase that configure application behavior. They prevent hardcoding sensitive data and allow the same code to run in multiple environments with different configurations.

### Creating .env Files

**File**: `.env` (local development)

```bash
# Server Configuration
NODE_ENV=development
PORT=3000
HOST=localhost

# Database
DB_HOST=localhost
DB_PORT=27017
DB_NAME=myapp_dev
DB_USER=admin
DB_PASSWORD=secret123

# JWT Configuration
JWT_SECRET=your-super-secret-key-min-32-chars
JWT_EXPIRES_IN=15m
JWT_REFRESH_EXPIRES_IN=7d

# External APIs
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your-email@gmail.com
SMTP_PASSWORD=your-app-password

# Client Configuration
CLIENT_URL=http://localhost:5173
CORS_ORIGIN=http://localhost:5173
```

**File**: `.env.production` (production server)

```bash
NODE_ENV=production
PORT=8080
HOST=0.0.0.0

# Production database with different credentials
DB_HOST=prod-db.example.com
DB_PORT=27017
DB_NAME=myapp_prod
DB_USER=prod_user
DB_PASSWORD=complex-production-password

JWT_SECRET=different-production-secret-key-very-long
JWT_EXPIRES_IN=15m
JWT_REFRESH_EXPIRES_IN=7d

CLIENT_URL=https://myapp.com
CORS_ORIGIN=https://myapp.com
```

**File**: `.env.example` (commit to Git)

```bash
# Server Configuration
NODE_ENV=development
PORT=3000
HOST=localhost

# Database
DB_HOST=
DB_PORT=
DB_NAME=
DB_USER=
DB_PASSWORD=

# JWT Configuration
JWT_SECRET=
JWT_EXPIRES_IN=15m

# External APIs
SMTP_HOST=
SMTP_PORT=
SMTP_USER=
SMTP_PASSWORD=

# Client Configuration
CLIENT_URL=
CORS_ORIGIN=
```

### Loading Environment Variables

**File**: `src/config/env.js`

```js
require("dotenv").config();

module.exports = {
  // Server
  nodeEnv: process.env.NODE_ENV || "development",
  port: process.env.PORT || 3000,
  host: process.env.HOST || "localhost",

  // Database
  db: {
    host: process.env.DB_HOST,
    port: process.env.DB_PORT,
    name: process.env.DB_NAME,
    user: process.env.DB_USER,
    password: process.env.DB_PASSWORD,
  },

  // JWT
  jwt: {
    secret: process.env.JWT_SECRET,
    expiresIn: process.env.JWT_EXPIRES_IN,
    refreshExpiresIn: process.env.JWT_REFRESH_EXPIRES_IN,
  },

  // CORS
  cors: {
    origin: process.env.CORS_ORIGIN,
  },
};
```

**Why centralize config?**
- Single source of truth
- Easy to validate required variables
- Type-safe access (no typos in `process.env.PORT` everywhere)
- Can add defaults and transformations

### Security: .gitignore

**File**: `.gitignore`

```bash
# Dependencies
node_modules/

# Environment files (NEVER commit these!)
.env
.env.local
.env.production
.env.staging

# Logs
logs/
*.log
npm-debug.log*

# OS files
.DS_Store
Thumbs.db

# IDE files
.vscode/
.idea/
*.swp
*.swo

# Build outputs
dist/
build/
coverage/
```

**Critical rule**: Never commit `.env` files to Git. They contain secrets.

---

## 5. Basic Server Setup

### Why Separate server.js and app.js?

**Principle**: Separate concerns
- `server.js` → Server initialization (HTTP, port binding)
- `app.js` → Express configuration (middleware, routes)

**Benefit**: `app.js` can be imported in tests without starting the server.

### Server Entry Point

**File**: `src/server.js`

```js
require("dotenv").config();
const app = require("./app");
const config = require("./config/env");

const PORT = config.port;
const HOST = config.host;

// Start server
const server = app.listen(PORT, HOST, () => {
  console.log(`
    ╔════════════════════════════════════════╗
    ║  Server running in ${config.nodeEnv} mode    ║
    ║  URL: http://${HOST}:${PORT}            ║
    ╚════════════════════════════════════════╝
  `);
});

// Graceful shutdown
process.on("SIGTERM", () => {
  console.log("SIGTERM signal received: closing HTTP server");
  server.close(() => {
    console.log("HTTP server closed");
  });
});
```

**What's happening**:
1. Load environment variables first
2. Import Express app
3. Start listening on configured port
4. Log startup info
5. Handle graceful shutdown (important for containers/cloud)

### Express App Configuration

**File**: `src/app.js`

```js
const express = require("express");
const cors = require("cors");
const cookieParser = require("cookie-parser");
const config = require("./config/env");

// Import routes
const authRoutes = require("./routes/auth.routes");
const userRoutes = require("./routes/user.routes");

// Import middleware
const errorHandler = require("./middlewares/error.middleware");

const app = express();

// ============================================
// Middleware (ORDER MATTERS!)
// ============================================

// 1. CORS - Must be before routes
app.use(cors({
  origin: config.cors.origin,
  credentials: true  // Allow cookies
}));

// 2. Body parsers
app.use(express.json());                    // Parse JSON bodies
app.use(express.urlencoded({ extended: true }));  // Parse URL-encoded bodies

// 3. Cookie parser
app.use(cookieParser());

// 4. Request logging (development only)
if (config.nodeEnv === "development") {
  app.use((req, res, next) => {
    console.log(`${req.method} ${req.path}`);
    next();
  });
}

// ============================================
// Routes
// ============================================

// Health check
app.get("/health", (req, res) => {
  res.status(200).json({
    status: "ok",
    timestamp: new Date().toISOString(),
    environment: config.nodeEnv
  });
});

// API routes
app.use("/api/auth", authRoutes);
app.use("/api/users", userRoutes);

// 404 handler - Must be after all routes
app.use("*", (req, res) => {
  res.status(404).json({
    error: "Route not found",
    path: req.originalUrl
  });
});

// ============================================
// Error Handler - Must be last
// ============================================
app.use(errorHandler);

module.exports = app;
```

### Why Middleware Order Matters

```js
// WRONG ORDER - Will fail
app.use("/api/users", userRoutes);  // Routes before body parser
app.use(express.json());            // Too late! req.body is undefined

// CORRECT ORDER
app.use(express.json());            // Parse body first
app.use("/api/users", userRoutes);  // Now req.body is available
```

**Rule of thumb**:
1. CORS and security headers (first)
2. Body parsers
3. Request logging
4. Routes
5. 404 handler
6. Error handler (last)

---

## 6. Routing Fundamentals

### Route Organization Pattern

**File**: `src/routes/user.routes.js`

```js
const express = require("express");
const userController = require("../controllers/user.controller");
const authMiddleware = require("../middlewares/auth.middleware");

const router = express.Router();

// Public routes
router.get("/", userController.getAllUsers);
router.get("/:id", userController.getUserById);

// Protected routes (require authentication)
router.post("/", authMiddleware, userController.createUser);
router.put("/:id", authMiddleware, userController.updateUser);
router.delete("/:id", authMiddleware, userController.deleteUser);

module.exports = router;
```

### Static vs Dynamic Routes

**Static routes**: Fixed URL segments
```js
// Static
app.get("/api/users", handler);           // Exact match only
app.get("/api/auth/login", handler);      // Exact match only
```

**Dynamic routes**: URL parameters that capture values
```js
// Dynamic
app.get("/api/users/:id", handler);       // Matches: /api/users/123
app.get("/api/posts/:postId/comments/:commentId", handler);
// Matches: /api/posts/456/comments/789

// Access in controller
function handler(req, res) {
  const userId = req.params.id;           // "123"
  const postId = req.params.postId;       // "456"
  const commentId = req.params.commentId; // "789"
}
```

### Route Parameters vs Query Strings

```js
// Route parameters - Part of the URL path
// URL: /api/users/123
app.get("/api/users/:id", (req, res) => {
  const id = req.params.id;  // "123"
});

// Query strings - After the ? in URL
// URL: /api/users?role=admin&status=active
app.get("/api/users", (req, res) => {
  const role = req.query.role;      // "admin"
  const status = req.query.status;  // "active"
});

// Combining both
// URL: /api/users/123?includeOrders=true
app.get("/api/users/:id", (req, res) => {
  const id = req.params.id;                      // "123"
  const includeOrders = req.query.includeOrders; // "true"
});
```

**When to use which**:
- **Route parameters**: Required identifiers (user ID, post ID)
- **Query strings**: Optional filters, pagination, sorting

### RESTful Route Conventions

REST (Representational State Transfer) follows predictable URL patterns:

| HTTP Method | Route | Purpose | Example |
|------------|-------|---------|---------|
| GET | `/api/users` | Get all users | List view |
| GET | `/api/users/:id` | Get single user | Detail view |
| POST | `/api/users` | Create new user | Registration |
| PUT | `/api/users/:id` | Update entire user | Full update |
| PATCH | `/api/users/:id` | Partial update user | Update email only |
| DELETE | `/api/users/:id` | Delete user | Account deletion |

**Example implementation**:

```js
const router = express.Router();

// GET /api/users - Get all users
router.get("/", async (req, res) => {
  const users = await User.find();
  res.json(users);
});

// GET /api/users/:id - Get single user
router.get("/:id", async (req, res) => {
  const user = await User.findById(req.params.id);
  if (!user) {
    return res.status(404).json({ error: "User not found" });
  }
  res.json(user);
});

// POST /api/users - Create user
router.post("/", async (req, res) => {
  const user = await User.create(req.body);
  res.status(201).json(user);
});

// PUT /api/users/:id - Full update
router.put("/:id", async (req, res) => {
  const user = await User.findByIdAndUpdate(
    req.params.id,
    req.body,
    { new: true, runValidators: true }
  );
  res.json(user);
});

// PATCH /api/users/:id - Partial update
router.patch("/:id", async (req, res) => {
  const user = await User.findByIdAndUpdate(
    req.params.id,
    { $set: req.body },
    { new: true }
  );
  res.json(user);
});

// DELETE /api/users/:id - Delete user
router.delete("/:id", async (req, res) => {
  await User.findByIdAndDelete(req.params.id);
  res.status(204).send();  // No content
});
```

### Route Naming Best Practices

```js
// ✅ Good - RESTful, clear, consistent
GET    /api/users
GET    /api/users/:id
POST   /api/users
PUT    /api/users/:id
DELETE /api/users/:id

// ❌ Bad - Inconsistent, unclear
GET /api/getAllUsers
GET /api/getUserById/:id
POST /api/createNewUser
PUT /api/userUpdate/:id
DELETE /api/removeUser/:id
```

**Rules**:
1. Use plural nouns (`/users`, not `/user`)
2. Use HTTP methods, not verbs in URLs
3. Use hyphens for multi-word resources (`/user-profiles`, not `/userProfiles`)
4. Keep it simple (max 2-3 levels deep)

---

## 7. HTTP Status Codes

### Essential Status Codes

Understanding HTTP status codes is critical for building intuitive APIs.

#### 2xx Success

| Code | Name | When to Use |
|------|------|-------------|
| **200** | OK | Successful GET, PUT, PATCH requests |
| **201** | Created | Successfully created resource (POST) |
| **204** | No Content | Successful DELETE (no response body) |

```js
// 200 - Successful retrieval
router.get("/users/:id", async (req, res) => {
  const user = await User.findById(req.params.id);
  res.status(200).json(user);  // or just res.json(user)
});

// 201 - Successfully created
router.post("/users", async (req, res) => {
  const user = await User.create(req.body);
  res.status(201).json(user);
});

// 204 - Successfully deleted
router.delete("/users/:id", async (req, res) => {
  await User.findByIdAndDelete(req.params.id);
  res.status(204).send();  // Empty response
});
```

#### 4xx Client Errors

| Code | Name | When to Use |
|------|------|-------------|
| **400** | Bad Request | Invalid input, validation errors |
| **401** | Unauthorized | Missing or invalid authentication |
| **403** | Forbidden | Valid auth, but insufficient permissions |
| **404** | Not Found | Resource doesn't exist |
| **409** | Conflict | Duplicate resource (email already exists) |

```js
// 400 - Bad request (validation error)
router.post("/users", async (req, res) => {
  if (!req.body.email) {
    return res.status(400).json({
      error: "Email is required"
    });
  }
});

// 401 - Unauthorized (not logged in)
router.get("/profile", (req, res) => {
  if (!req.headers.authorization) {
    return res.status(401).json({
      error: "Authentication required"
    });
  }
});

// 403 - Forbidden (logged in but not allowed)
router.delete("/users/:id", (req, res) => {
  if (req.user.role !== "admin") {
    return res.status(403).json({
      error: "Admin access required"
    });
  }
});

// 404 - Not found
router.get("/users/:id", async (req, res) => {
  const user = await User.findById(req.params.id);
  if (!user) {
    return res.status(404).json({
      error: "User not found"
    });
  }
  res.json(user);
});

// 409 - Conflict (duplicate)
router.post("/users", async (req, res) => {
  const existing = await User.findOne({ email: req.body.email });
  if (existing) {
    return res.status(409).json({
      error: "Email already registered"
    });
  }
});
```

#### 5xx Server Errors

| Code | Name | When to Use |
|------|------|-------------|
| **500** | Internal Server Error | Unexpected errors, exceptions |
| **503** | Service Unavailable | Server overload, maintenance |

```js
// 500 - Server error
router.get("/users", async (req, res) => {
  try {
    const users = await User.find();
    res.json(users);
  } catch (error) {
    console.error(error);
    res.status(500).json({
      error: "Internal server error"
    });
  }
});
```

### Status Code Decision Tree

```
Is the request successful?
├─ Yes
│  ├─ Did you create something? → 201
│  ├─ Did you delete something? → 204
│  └─ Everything else → 200
│
└─ No
   ├─ Is it the client's fault?
   │  ├─ Bad input data? → 400
   │  ├─ Not logged in? → 401
   │  ├─ Logged in but not allowed? → 403
   │  ├─ Resource doesn't exist? → 404
   │  └─ Duplicate/conflict? → 409
   │
   └─ Is it the server's fault?
      ├─ Your code crashed? → 500
      └─ Server down/overloaded? → 503
```

---

## 8. REST API Principles

### What is REST?

**Simple explanation**: A set of rules for building web APIs that are predictable, scalable, and easy to use.

**Technical explanation**: REST (Representational State Transfer) is an architectural style that uses HTTP methods and URLs to perform CRUD operations on resources. It emphasizes stateless communication, resource-based URLs, and standard HTTP semantics.

### Six Core Principles

#### 1. **Stateless**

Each request contains all information needed to process it. Server doesn't store session data.

```js
// ❌ Bad - Stateful (server stores state)
// Request 1: POST /api/login
server.session.userId = 123;

// Request 2: GET /api/profile
const userId = server.session.userId;  // Relies on previous request

// ✅ Good - Stateless
// Request 1: POST /api/login
res.json({ token: "jwt-token-here" });

// Request 2: GET /api/profile
// Header: Authorization: Bearer jwt-token-here
const userId = verifyToken(req.headers.authorization);
```

#### 2. **Resource-Based URLs**

URLs represent resources (nouns), not actions (verbs).

```js
// ❌ Bad - Action-based
POST /api/createUser
GET  /api/getUser/123
POST /api/deleteUser/123

// ✅ Good - Resource-based
POST   /api/users          // Create
GET    /api/users/123      // Read
DELETE /api/users/123      // Delete
```

#### 3. **HTTP Methods Define Actions**

Use correct HTTP verbs for operations.

```js
GET    /api/users      // Read (Safe, Idempotent)
POST   /api/users      // Create (Not idempotent)
PUT    /api/users/123  // Full update (Idempotent)
PATCH  /api/users/123  // Partial update (Not idempotent)
DELETE /api/users/123  // Delete (Idempotent)
```

**Idempotent**: Multiple identical requests have the same effect as a single request.

#### 4. **Standard Response Formats**

Consistent JSON structure across all endpoints.

```js
// Success response
{
  "data": { ... },
  "message": "User created successfully"
}

// Error response
{
  "error": {
    "message": "Validation failed",
    "code": "VALIDATION_ERROR",
    "details": [
      { "field": "email", "message": "Invalid email format" }
    ]
  }
}

// List response with metadata
{
  "data": [ ... ],
  "meta": {
    "total": 100,
    "page": 1,
    "limit": 20
  }
}
```

#### 5. **Use HTTP Status Codes**

Let status codes convey meaning (covered in previous section).

#### 6. **Versioning**

Plan for API changes with versioning.

```js
// URL versioning (most common)
app.use("/api/v1/users", usersV1Routes);
app.use("/api/v2/users", usersV2Routes);

// Header versioning
app.use((req, res, next) => {
  const version = req.headers["api-version"] || "1";
  if (version === "2") {
    // Use v2 logic
  }
});
```

### Example: RESTful User API

**File**: `src/controllers/user.controller.js`

```js
const User = require("../models/User.model");

// GET /api/users
exports.getAllUsers = async (req, res) => {
  try {
    const users = await User.find();
    res.status(200).json({
      success: true,
      data: users
    });
  } catch (error) {
    res.status(500).json({
      success: false,
      error: error.message
    });
  }
};

// GET /api/users/:id
exports.getUserById = async (req, res) => {
  try {
    const user = await User.findById(req.params.id);
    
    if (!user) {
      return res.status(404).json({
        success: false,
        error: "User not found"
      });
    }

    res.status(200).json({
      success: true,
      data: user
    });
  } catch (error) {
    res.status(500).json({
      success: false,
      error: error.message
    });
  }
};

// POST /api/users
exports.createUser = async (req, res) => {
  try {
    const user = await User.create(req.body);
    
    res.status(201).json({
      success: true,
      data: user,
      message: "User created successfully"
    });
  } catch (error) {
    // Mongoose validation error
    if (error.name === "ValidationError") {
      return res.status(400).json({
        success: false,
        error: error.message
      });
    }
    
    res.status(500).json({
      success: false,
      error: error.message
    });
  }
};

// PUT /api/users/:id
exports.updateUser = async (req, res) => {
  try {
    const user = await User.findByIdAndUpdate(
      req.params.id,
      req.body,
      { new: true, runValidators: true }
    );

    if (!user) {
      return res.status(404).json({
        success: false,
        error: "User not found"
      });
    }

    res.status(200).json({
      success: true,
      data: user,
      message: "User updated successfully"
    });
  } catch (error) {
    res.status(500).json({
      success: false,
      error: error.message
    });
  }
};

// DELETE /api/users/:id
exports.deleteUser = async (req, res) => {
  try {
    const user = await User.findByIdAndDelete(req.params.id);

    if (!user) {
      return res.status(404).json({
        success: false,
        error: "User not found"
      });
    }

    res.status(204).send();  // No content
  } catch (error) {
    res.status(500).json({
      success: false,
      error: error.message
    });
  }
};
```

---

## 9. Development Workflow

### npm Scripts Setup

**File**: `package.json`

```json
{
  "scripts": {
    "start": "node src/server.js",
    "dev": "nodemon src/server.js",
    "dev:debug": "nodemon --inspect src/server.js"
  }
}
```

**Commands**:
```bash
# Production (no auto-reload)
npm start

# Development (auto-reload on changes)
npm run dev

# Development with debugger
npm run dev:debug
```

### Using Nodemon

**What is nodemon?**: Automatically restarts your server when files change.

**Configuration**: Create `nodemon.json`

```json
{
  "watch": ["src"],
  "ext": "js,json",
  "ignore": ["src/logs/*", "*.test.js"],
  "exec": "node src/server.js"
}
```

**Options explained**:
- `watch`: Folders to monitor
- `ext`: File extensions to watch
- `ignore`: Files/folders to skip
- `exec`: Command to run

---

## 10. Complete Working Example

Let's put everything together with a minimal but complete API.

### Project Structure

```text
user-api/
├── src/
│   ├── server.js
│   ├── app.js
│   ├── config/
│   │   └── env.js
│   ├── controllers/
│   │   └── user.controller.js
│   ├── routes/
│   │   └── user.routes.js
│   └── middlewares/
│       └── error.middleware.js
├── .env
├── .env.example
├── .gitignore
└── package.json
```

### Implementation Files

**`package.json`**:
```json
{
  "name": "user-api",
  "version": "1.0.0",
  "main": "src/server.js",
  "scripts": {
    "start": "node src/server.js",
    "dev": "nodemon src/server.js"
  },
  "dependencies": {
    "express": "^4.18.2",
    "dotenv": "^16.3.1",
    "cors": "^2.8.5"
  },
  "devDependencies": {
    "nodemon": "^3.0.1"
  }
}
```

**`.env`**:
```bash
NODE_ENV=development
PORT=3000
CORS_ORIGIN=http://localhost:5173
```

**`src/config/env.js`**:
```js
require("dotenv").config();

module.exports = {
  nodeEnv: process.env.NODE_ENV || "development",
  port: process.env.PORT || 3000,
  cors: {
    origin: process.env.CORS_ORIGIN || "*"
  }
};
```

**`src/server.js`**:
```js
const app = require("./app");
const config = require("./config/env");

const server = app.listen(config.port, () => {
  console.log(`Server running on port ${config.port}`);
});

process.on("SIGTERM", () => {
  server.close(() => console.log("Server closed"));
});
```

**`src/app.js`**:
```js
const express = require("express");
const cors = require("cors");
const config = require("./config/env");
const userRoutes = require("./routes/user.routes");
const errorHandler = require("./middlewares/error.middleware");

const app = express();

// Middleware
app.use(cors({ origin: config.cors.origin }));
app.use(express.json());

// Routes
app.get("/health", (req, res) => {
  res.json({ status: "ok" });
});

app.use("/api/users", userRoutes);

// Error handling
app.use("*", (req, res) => {
  res.status(404).json({ error: "Route not found" });
});

app.use(errorHandler);

module.exports = app;
```

**`src/routes/user.routes.js`**:
```js
const express = require("express");
const userController = require("../controllers/user.controller");

const router = express.Router();

router.get("/", userController.getAllUsers);
router.get("/:id", userController.getUserById);
router.post("/", userController.createUser);
router.put("/:id", userController.updateUser);
router.delete("/:id", userController.deleteUser);

module.exports = router;
```

**`src/controllers/user.controller.js`**:
```js
// Mock data (replace with database later)
let users = [
  { id: "1", name: "Alice", email: "alice@example.com" },
  { id: "2", name: "Bob", email: "bob@example.com" }
];

exports.getAllUsers = (req, res) => {
  res.json({ success: true, data: users });
};

exports.getUserById = (req, res) => {
  const user = users.find(u => u.id === req.params.id);
  if (!user) {
    return res.status(404).json({ success: false, error: "User not found" });
  }
  res.json({ success: true, data: user });
};

exports.createUser = (req, res) => {
  const newUser = {
    id: String(users.length + 1),
    ...req.body
  };
  users.push(newUser);
  res.status(201).json({ success: true, data: newUser });
};

exports.updateUser = (req, res) => {
  const index = users.findIndex(u => u.id === req.params.id);
  if (index === -1) {
    return res.status(404).json({ success: false, error: "User not found" });
  }
  users[index] = { ...users[index], ...req.body };
  res.json({ success: true, data: users[index] });
};

exports.deleteUser = (req, res) => {
  const index = users.findIndex(u => u.id === req.params.id);
  if (index === -1) {
    return res.status(404).json({ success: false, error: "User not found" });
  }
  users.splice(index, 1);
  res.status(204).send();
};
```

**`src/middlewares/error.middleware.js`**:
```js
module.exports = (err, req, res, next) => {
  console.error(err.stack);
  
  res.status(err.statusCode || 500).json({
    success: false,
    error: err.message || "Internal server error"
  });
};
```

### Running the Project

```bash
# Install dependencies
npm install

# Start development server
npm run dev

# Test endpoints
curl http://localhost:3000/health
curl http://localhost:3000/api/users
curl -X POST http://localhost:3000/api/users \
  -H "Content-Type: application/json" \
  -d '{"name":"Charlie","email":"charlie@example.com"}'
```

---

## Summary

You now have a production-ready Node.js project setup with:

**Foundation**:
- ✅ Proper project initialization and dependencies
- ✅ MVC folder structure for scalability
- ✅ Environment configuration for different deployments

**Server Setup**:
- ✅ Separated server.js and app.js
- ✅ Middleware configured in correct order
- ✅ Graceful shutdown handling

**Routing**:
- ✅ RESTful conventions
- ✅ Route organization pattern
- ✅ Static and dynamic routes

**HTTP Fundamentals**:
- ✅ Proper status codes
- ✅ REST principles
- ✅ Consistent response formats

**Next Steps**:
1. Add database integration (MongoDB/PostgreSQL)
2. Implement authentication (JWT)
3. Add validation middleware
4. Set up logging
5. Write tests

This structure will scale from small projects to large applications without major refactoring.

# 6. Logging in Node.js

## Why Logging?

**Problem**: `console.log()` works in development, but in production you need:
- Logs saved to files (not just terminal)
- Different log levels (info, warning, error)
- Timestamps and structured data
- Ability to search/filter logs later

**Solution**: Use a logging library like Winston.

---

## 1. Project Structure

```text
your-app/
├── src/
│   ├── server.js
│   ├── config/
│   │   └── logger.js          # Logger configuration
│   ├── middlewares/
│   │   └── requestLogger.js   # HTTP request logging
│   └── utils/
│       └── logger.js           # Export logger instance
├── logs/
│   ├── app.log                 # All logs
│   ├── error.log               # Only errors
│   └── combined.log            # Everything combined
└── package.json
```

---

## 2. Installing Winston

```bash
npm install winston
npm install morgan  # For HTTP request logging
```

---

## 3. Logger Setup

**File**: `src/config/logger.js`

```js
const winston = require("winston");
const path = require("path");

// ============================================
// LOG LEVELS
// ============================================
// Winston has these levels (in order of severity):
// error: 0    - Something broke
// warn: 1     - Something might be wrong
// info: 2     - General information
// http: 3     - HTTP requests
// debug: 4    - Debugging information

// ============================================
// LOG FORMAT
// ============================================
const logFormat = winston.format.combine(
  winston.format.timestamp({ format: "YYYY-MM-DD HH:mm:ss" }),
  winston.format.errors({ stack: true }),  // Include error stack traces
  winston.format.printf(({ timestamp, level, message, stack }) => {
    // Custom format: [2024-01-15 10:30:45] INFO: User logged in
    if (stack) {
      return `[${timestamp}] ${level.toUpperCase()}: ${message}\n${stack}`;
    }
    return `[${timestamp}] ${level.toUpperCase()}: ${message}`;
  })
);

// ============================================
// CREATE LOGGER
// ============================================
const logger = winston.createLogger({
  level: process.env.NODE_ENV === "production" ? "info" : "debug",
  // In production: log info and above (info, warn, error)
  // In development: log everything (debug, http, info, warn, error)
  
  format: logFormat,
  
  transports: [
    // Write all logs to combined.log
    new winston.transports.File({
      filename: path.join(__dirname, "../../logs/combined.log"),
      maxsize: 5242880,  // 5MB
      maxFiles: 5,       // Keep 5 files max (rotation)
    }),
    
    // Write only errors to error.log
    new winston.transports.File({
      filename: path.join(__dirname, "../../logs/error.log"),
      level: "error",
      maxsize: 5242880,
      maxFiles: 5,
    }),
  ],
});

// ============================================
// CONSOLE OUTPUT (Development Only)
// ============================================
// In development, also print to console with colors
if (process.env.NODE_ENV !== "production") {
  logger.add(
    new winston.transports.Console({
      format: winston.format.combine(
        winston.format.colorize(),  // Colors for terminal
        winston.format.simple()
      ),
    })
  );
}

module.exports = logger;
```

---

## 4. Using the Logger

**File**: `src/utils/logger.js`

```js
const logger = require("../config/logger");

// Export logger for use throughout the app
module.exports = logger;
```

**In your code**:

```js
const logger = require("./utils/logger");

// ============================================
// BASIC USAGE
// ============================================

// Info: General information
logger.info("Server started on port 3000");
logger.info("User logged in", { userId: "user123", email: "alice@example.com" });

// Warning: Something unusual but not breaking
logger.warn("API rate limit approaching", { userId: "user456", requests: 95 });
logger.warn("Deprecated endpoint used: /api/v1/users");

// Error: Something broke
logger.error("Database connection failed", { error: "Connection timeout" });
logger.error("Payment processing error", new Error("Stripe API timeout"));

// Debug: Detailed info for debugging (only in development)
logger.debug("Processing payment", { amount: 100, currency: "USD" });

// HTTP: Request logging (handled by middleware, see next section)
logger.http("GET /api/users 200 45ms");

// ============================================
// LOGGING ERRORS WITH STACK TRACES
// ============================================

try {
  // Some code that might fail
  const user = await User.findById(userId);
} catch (error) {
  // Logs the full error with stack trace
  logger.error("Failed to fetch user", {
    userId: userId,
    error: error.message,
    stack: error.stack
  });
}

// ============================================
// STRUCTURED LOGGING
// ============================================
// Add context to your logs for better searchability

// Good: Structured with metadata
logger.info("Order created", {
  orderId: "order_123",
  userId: "user_456",
  amount: 99.99,
  items: 3
});

// Bad: Just a string (harder to search/filter later)
logger.info("Order order_123 created by user_456 for $99.99 with 3 items");
```

---

## 5. HTTP Request Logging

**File**: `src/middlewares/requestLogger.js`

```js
const morgan = require("morgan");
const logger = require("../utils/logger");

// ============================================
// MORGAN SETUP
// ============================================
// Morgan logs HTTP requests automatically
// Format: :method :url :status :response-time ms

// Create a stream to pipe morgan logs to winston
const stream = {
  write: (message) => {
    // Remove newline and log as HTTP level
    logger.http(message.trim());
  },
};

// Morgan middleware
const requestLogger = morgan(
  ":method :url :status :res[content-length] - :response-time ms",
  { stream }
);

module.exports = requestLogger;
```

**File**: `src/app.js`

```js
const express = require("express");
const requestLogger = require("./middlewares/requestLogger");

const app = express();

// ============================================
// ADD REQUEST LOGGING MIDDLEWARE
// ============================================
// This logs every HTTP request automatically
app.use(requestLogger);

// Your routes...
app.get("/api/users", (req, res) => {
  // This request will be automatically logged:
  // GET /api/users 200 15 - 12.345 ms
  res.json({ users: [] });
});

module.exports = app;
```

---

## 6. Error Logging Middleware

**File**: `src/middlewares/errorLogger.js`

```js
const logger = require("../utils/logger");

// ============================================
// ERROR LOGGING MIDDLEWARE
// ============================================
// Must be the LAST middleware (after all routes)

function errorLogger(err, req, res, next) {
  // Log the error with request context
  logger.error("Unhandled error", {
    error: err.message,
    stack: err.stack,
    method: req.method,
    url: req.url,
    ip: req.ip,
    userId: req.user?.id,  // If you have auth middleware
  });

  // Send error response
  res.status(err.status || 500).json({
    success: false,
    error: process.env.NODE_ENV === "production" 
      ? "Internal server error"  // Hide details in production
      : err.message               // Show details in development
  });
}

module.exports = errorLogger;
```

**File**: `src/app.js`

```js
const express = require("express");
const requestLogger = require("./middlewares/requestLogger");
const errorLogger = require("./middlewares/errorLogger");

const app = express();

// Request logging (first)
app.use(requestLogger);

// Your routes...
app.get("/api/users", (req, res) => {
  res.json({ users: [] });
});

// Error logging (last)
app.use(errorLogger);

module.exports = app;
```

---

## 7. Logging in Different Parts of Your App

### 7.1 Server Startup

**File**: `src/server.js`

```js
const logger = require("./utils/logger");
const app = require("./app");
const connectDB = require("./config/database");

const PORT = process.env.PORT || 3000;

connectDB()
  .then(() => {
    logger.info("Database connected successfully");
    
    app.listen(PORT, () => {
      logger.info(`Server running on port ${PORT}`, {
        environment: process.env.NODE_ENV,
        port: PORT
      });
    });
  })
  .catch((error) => {
    logger.error("Failed to start server", {
      error: error.message,
      stack: error.stack
    });
    process.exit(1);
  });

// Log uncaught exceptions
process.on("uncaughtException", (error) => {
  logger.error("Uncaught Exception", {
    error: error.message,
    stack: error.stack
  });
  process.exit(1);
});

// Log unhandled promise rejections
process.on("unhandledRejection", (reason, promise) => {
  logger.error("Unhandled Rejection", {
    reason: reason,
    promise: promise
  });
});
```

### 7.2 Database Operations

```js
const logger = require("../utils/logger");
const User = require("../models/User.model");

async function createUser(userData) {
  try {
    logger.info("Creating new user", { email: userData.email });
    
    const user = await User.create(userData);
    
    logger.info("User created successfully", {
      userId: user._id,
      email: user.email
    });
    
    return user;
    
  } catch (error) {
    logger.error("Failed to create user", {
      email: userData.email,
      error: error.message
    });
    throw error;
  }
}
```

### 7.3 API Controllers

```js
const logger = require("../utils/logger");

exports.loginUser = async (req, res) => {
  try {
    const { email, password } = req.body;
    
    logger.info("Login attempt", { email });
    
    const user = await User.findOne({ email });
    
    if (!user) {
      logger.warn("Login failed: User not found", { email });
      return res.status(401).json({ error: "Invalid credentials" });
    }
    
    const isValid = await user.comparePassword(password);
    
    if (!isValid) {
      logger.warn("Login failed: Invalid password", { 
        email,
        userId: user._id 
      });
      return res.status(401).json({ error: "Invalid credentials" });
    }
    
    logger.info("User logged in successfully", {
      userId: user._id,
      email: user.email
    });
    
    res.json({ token: generateToken(user) });
    
  } catch (error) {
    logger.error("Login error", {
      email: req.body.email,
      error: error.message,
      stack: error.stack
    });
    res.status(500).json({ error: "Login failed" });
  }
};
```

### 7.4 Background Jobs

```js
const logger = require("../utils/logger");

async function processPayments() {
  logger.info("Starting payment processing job");
  
  try {
    const pendingPayments = await Payment.find({ status: "pending" });
    
    logger.info("Found pending payments", { count: pendingPayments.length });
    
    for (const payment of pendingPayments) {
      try {
        await processPayment(payment);
        logger.info("Payment processed", { paymentId: payment._id });
      } catch (error) {
        logger.error("Payment processing failed", {
          paymentId: payment._id,
          error: error.message
        });
      }
    }
    
    logger.info("Payment processing job completed");
    
  } catch (error) {
    logger.error("Payment processing job failed", {
      error: error.message,
      stack: error.stack
    });
  }
}
```

---

## 8. Log Rotation (Automatic Cleanup)

Winston automatically rotates logs when using `maxsize` and `maxFiles`:

```js
new winston.transports.File({
  filename: "logs/combined.log",
  maxsize: 5242880,  // 5MB - when file reaches this size, create new file
  maxFiles: 5,       // Keep only 5 most recent files, delete older ones
})
```

**What happens**:
```
logs/
├── combined.log         # Current log (3MB)
├── combined.log.1       # Previous (5MB)
├── combined.log.2       # Older (5MB)
├── combined.log.3       # Older (5MB)
├── combined.log.4       # Oldest (5MB)
└── combined.log.5       # Will be deleted when combined.log rotates
```

---

## 9. Viewing & Searching Logs

### 9.1 View Recent Logs

```bash
# View last 50 lines
tail -n 50 logs/combined.log

# Follow logs in real-time (like console.log)
tail -f logs/combined.log

# View only errors
tail -f logs/error.log
```

### 9.2 Search Logs

```bash
# Find all logs for a specific user
grep "user_123" logs/combined.log

# Find all errors
grep "ERROR" logs/combined.log

# Find logs from specific date
grep "2024-01-15" logs/combined.log

# Count occurrences
grep -c "Login attempt" logs/combined.log
```

---

## 10. Environment-Specific Configuration

**File**: `.env`

```bash
# Development
NODE_ENV=development
LOG_LEVEL=debug

# Production
NODE_ENV=production
LOG_LEVEL=info
```

**File**: `src/config/logger.js`

```js
const logger = winston.createLogger({
  level: process.env.LOG_LEVEL || "info",
  // ...
});

// Only log to console in development
if (process.env.NODE_ENV !== "production") {
  logger.add(new winston.transports.Console({
    format: winston.format.combine(
      winston.format.colorize(),
      winston.format.simple()
    )
  }));
}
```

---

## 11. Best Practices

### DO:

```js
// ✅ Include context
logger.info("Order created", { orderId, userId, amount });

// ✅ Log both success and failure
logger.info("Payment successful", { transactionId });
logger.error("Payment failed", { error: error.message });

// ✅ Use appropriate levels
logger.info("User logged in");        // Normal operation
logger.warn("Rate limit hit");         // Warning but not error
logger.error("Database query failed"); // Actual error
```

### DON'T:

```js
// ❌ Log sensitive data
logger.info("User logged in", { password: "secret123" });  // NEVER!

// ❌ Log too much in production
logger.debug("Variable value: " + JSON.stringify(largeObject));  // Bloats logs

// ❌ Use console.log in production
console.log("User created");  // Won't be saved to files

// ❌ Log without context
logger.info("Error occurred");  // Which error? Where? Why?
```

---

## 12. Complete Example

**File**: `src/controllers/user.controller.js`

```js
const logger = require("../utils/logger");
const User = require("../models/User.model");

exports.createUser = async (req, res) => {
  try {
    const { email, username } = req.body;
    
    logger.info("Creating user", { email, username });
    
    // Check if user exists
    const existingUser = await User.findOne({ email });
    if (existingUser) {
      logger.warn("User creation failed: Email already exists", { email });
      return res.status(409).json({ error: "Email already in use" });
    }
    
    // Create user
    const user = await User.create(req.body);
    
    logger.info("User created successfully", {
      userId: user._id,
      email: user.email,
      username: user.username
    });
    
    res.status(201).json({ user });
    
  } catch (error) {
    logger.error("User creation error", {
      email: req.body.email,
      error: error.message,
      stack: error.stack
    });
    
    res.status(500).json({ error: "Failed to create user" });
  }
};

exports.deleteUser = async (req, res) => {
  try {
    const { id } = req.params;
    
    logger.info("Deleting user", { userId: id });
    
    const user = await User.findByIdAndDelete(id);
    
    if (!user) {
      logger.warn("User deletion failed: User not found", { userId: id });
      return res.status(404).json({ error: "User not found" });
    }
    
    logger.info("User deleted successfully", {
      userId: id,
      email: user.email
    });
    
    res.json({ message: "User deleted" });
    
  } catch (error) {
    logger.error("User deletion error", {
      userId: req.params.id,
      error: error.message,
      stack: error.stack
    });
    
    res.status(500).json({ error: "Failed to delete user" });
  }
};
```

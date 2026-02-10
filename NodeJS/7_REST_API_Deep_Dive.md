# 7. Deep Dive into REST API

## What is REST?

**REST** (Representational State Transfer) is an architectural style for building APIs. It uses HTTP methods and follows specific principles to create predictable, scalable APIs.

**Core concept**: Everything is a "resource" (user, post, order) accessed via URLs.

---

## 1. RESTful Principles

### 1.1 Resources & URLs

**Think in terms of resources, not actions**:

```
✅ Good (Resource-based):
GET    /api/users          # Get all users
GET    /api/users/123      # Get specific user
POST   /api/users          # Create user
PUT    /api/users/123      # Update user
DELETE /api/users/123      # Delete user

❌ Bad (Action-based):
GET    /api/getUsers
GET    /api/getUserById?id=123
POST   /api/createUser
POST   /api/updateUser
POST   /api/deleteUser
```

### 1.2 HTTP Methods

| Method | Purpose | Example |
|--------|---------|---------|
| GET | Retrieve data | Get user list |
| POST | Create new resource | Create user |
| PUT | Update entire resource | Replace user data |
| PATCH | Partial update | Update user email only |
| DELETE | Remove resource | Delete user |

### 1.3 Nested Resources

For related resources, use nested URLs:

```js
// Posts belong to users
GET    /api/users/123/posts           # Get all posts by user 123
POST   /api/users/123/posts           # Create post for user 123
GET    /api/users/123/posts/456       # Get specific post by user

// Comments belong to posts
GET    /api/posts/456/comments        # Get all comments on post 456
POST   /api/posts/456/comments        # Add comment to post 456
DELETE /api/posts/456/comments/789    # Delete comment 789
```

**Limit nesting to 2 levels** - too deep gets confusing:

```
✅ Good: /api/users/123/posts/456
❌ Too deep: /api/users/123/posts/456/comments/789/likes/101
```

### 1.4 Status Codes

Use appropriate HTTP status codes:

```js
// Success
200 OK              // Request succeeded, returning data
201 Created         // Resource created successfully
204 No Content      // Success but no data to return (e.g., DELETE)

// Client Errors
400 Bad Request     // Invalid data sent
401 Unauthorized    // Not authenticated (missing/invalid token)
403 Forbidden       // Authenticated but not allowed
404 Not Found       // Resource doesn't exist
409 Conflict        // Duplicate resource (e.g., email already exists)
422 Unprocessable   // Validation failed

// Server Errors
500 Internal Error  // Something broke on server
503 Unavailable     // Server temporarily down
```

**Example usage**:

```js
// User controller
exports.createUser = async (req, res) => {
  try {
    const { email } = req.body;
    
    // Check duplicate
    const existing = await User.findOne({ email });
    if (existing) {
      return res.status(409).json({  // 409 Conflict
        success: false,
        error: "Email already in use"
      });
    }
    
    const user = await User.create(req.body);
    
    res.status(201).json({  // 201 Created
      success: true,
      data: user
    });
    
  } catch (error) {
    res.status(500).json({  // 500 Internal Error
      success: false,
      error: "Failed to create user"
    });
  }
};
```

---

## 2. Pagination

**Problem**: Returning 10,000 users in one response is slow and wasteful.

**Solution**: Return data in pages.

### 2.1 Offset-Based Pagination

**Most common approach** - like numbered pages in a book.

**Query parameters**:
```
GET /api/users?page=1&limit=20
GET /api/users?page=2&limit=20
```

**Implementation**:

```js
// File: src/controllers/user.controller.js

exports.getUsers = async (req, res) => {
  try {
    // Parse query parameters with defaults
    const page = parseInt(req.query.page) || 1;
    const limit = parseInt(req.query.limit) || 20;
    
    // Validate limits
    if (limit > 100) {
      return res.status(400).json({
        success: false,
        error: "Limit cannot exceed 100"
      });
    }
    
    // Calculate skip
    const skip = (page - 1) * limit;
    // Page 1: skip 0
    // Page 2: skip 20
    // Page 3: skip 40
    
    // Fetch users
    const users = await User.find()
      .skip(skip)
      .limit(limit)
      .sort({ createdAt: -1 });  // Newest first
    
    // Get total count (for calculating total pages)
    const total = await User.countDocuments();
    
    // Calculate pagination metadata
    const totalPages = Math.ceil(total / limit);
    const hasNextPage = page < totalPages;
    const hasPrevPage = page > 1;
    
    res.json({
      success: true,
      data: users,
      pagination: {
        page,
        limit,
        total,
        totalPages,
        hasNextPage,
        hasPrevPage
      }
    });
    
  } catch (error) {
    res.status(500).json({
      success: false,
      error: "Failed to fetch users"
    });
  }
};
```

**Response**:

```json
{
  "success": true,
  "data": [
    { "id": "1", "name": "Alice" },
    { "id": "2", "name": "Bob" }
  ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 150,
    "totalPages": 8,
    "hasNextPage": true,
    "hasPrevPage": false
  }
}
```

### 2.2 Cursor-Based Pagination

**Better for real-time data** - handles new items being added while paginating.

**Query parameters**:
```
GET /api/posts?limit=20
GET /api/posts?limit=20&cursor=post_xyz123
```

**Implementation**:

```js
exports.getPosts = async (req, res) => {
  try {
    const limit = parseInt(req.query.limit) || 20;
    const cursor = req.query.cursor;  // ID of last item from previous page
    
    // Build query
    const query = {};
    
    if (cursor) {
      // Fetch items created before the cursor
      const cursorPost = await Post.findById(cursor);
      if (cursorPost) {
        query.createdAt = { $lt: cursorPost.createdAt };
      }
    }
    
    // Fetch posts
    const posts = await Post.find(query)
      .limit(limit + 1)  // Fetch one extra to check if there's more
      .sort({ createdAt: -1 });
    
    // Check if there are more results
    const hasNextPage = posts.length > limit;
    
    // Remove the extra item
    if (hasNextPage) {
      posts.pop();
    }
    
    // Get cursor for next page (last item's ID)
    const nextCursor = hasNextPage ? posts[posts.length - 1]._id : null;
    
    res.json({
      success: true,
      data: posts,
      pagination: {
        nextCursor,
        hasNextPage
      }
    });
    
  } catch (error) {
    res.status(500).json({
      success: false,
      error: "Failed to fetch posts"
    });
  }
};
```

**When to use which**:

| Offset-Based | Cursor-Based |
|--------------|--------------|
| ✅ Simple page numbers | ✅ Real-time feeds |
| ✅ "Jump to page 5" | ✅ Infinite scroll |
| ✅ Static data | ✅ Handles concurrent writes |
| ❌ Issues if data changes | ❌ Can't jump to specific page |

---

## 3. Filtering & Sorting

### 3.1 Filtering

**Allow clients to filter results**:

```
GET /api/products?category=electronics
GET /api/products?category=electronics&minPrice=100&maxPrice=500
GET /api/users?isActive=true&role=admin
```

**Implementation**:

```js
exports.getProducts = async (req, res) => {
  try {
    const { category, minPrice, maxPrice, inStock } = req.query;
    
    // Build query object dynamically
    const query = {};
    
    if (category) {
      query.category = category;
    }
    
    if (minPrice || maxPrice) {
      query.price = {};
      if (minPrice) query.price.$gte = parseFloat(minPrice);
      if (maxPrice) query.price.$lte = parseFloat(maxPrice);
    }
    
    if (inStock !== undefined) {
      query.inStock = inStock === "true";
    }
    
    const products = await Product.find(query);
    
    res.json({
      success: true,
      data: products,
      filters: { category, minPrice, maxPrice, inStock }
    });
    
  } catch (error) {
    res.status(500).json({
      success: false,
      error: "Failed to fetch products"
    });
  }
};
```

### 3.2 Sorting

**Allow clients to sort results**:

```
GET /api/products?sort=price          # Ascending
GET /api/products?sort=-price         # Descending (- prefix)
GET /api/products?sort=category,price # Multiple fields
```

**Implementation**:

```js
exports.getProducts = async (req, res) => {
  try {
    const { sort } = req.query;
    
    // Build sort object
    let sortObj = { createdAt: -1 };  // Default: newest first
    
    if (sort) {
      sortObj = {};
      const fields = sort.split(",");
      
      fields.forEach(field => {
        if (field.startsWith("-")) {
          // Descending
          sortObj[field.substring(1)] = -1;
        } else {
          // Ascending
          sortObj[field] = 1;
        }
      });
    }
    
    const products = await Product.find().sort(sortObj);
    
    res.json({
      success: true,
      data: products
    });
    
  } catch (error) {
    res.status(500).json({
      success: false,
      error: "Failed to fetch products"
    });
  }
};
```

### 3.3 Field Selection

**Allow clients to request specific fields only**:

```
GET /api/users?fields=name,email
GET /api/products?fields=name,price,category
```

**Implementation**:

```js
exports.getUsers = async (req, res) => {
  try {
    const { fields } = req.query;
    
    // Build select string
    let select = "";
    if (fields) {
      select = fields.split(",").join(" ");
      // "name,email" becomes "name email" for Mongoose
    }
    
    const users = select 
      ? await User.find().select(select)
      : await User.find();
    
    res.json({
      success: true,
      data: users
    });
    
  } catch (error) {
    res.status(500).json({
      success: false,
      error: "Failed to fetch users"
    });
  }
};
```

### 3.4 Combined Example

**All features together**:

```js
exports.getProducts = async (req, res) => {
  try {
    // Pagination
    const page = parseInt(req.query.page) || 1;
    const limit = parseInt(req.query.limit) || 20;
    const skip = (page - 1) * limit;
    
    // Filtering
    const query = {};
    if (req.query.category) query.category = req.query.category;
    if (req.query.minPrice) query.price = { $gte: parseFloat(req.query.minPrice) };
    
    // Sorting
    let sortObj = { createdAt: -1 };
    if (req.query.sort) {
      sortObj = {};
      req.query.sort.split(",").forEach(field => {
        sortObj[field.startsWith("-") ? field.substring(1) : field] = 
          field.startsWith("-") ? -1 : 1;
      });
    }
    
    // Field selection
    const select = req.query.fields ? req.query.fields.split(",").join(" ") : "";
    
    // Execute query
    const products = await Product.find(query)
      .select(select)
      .sort(sortObj)
      .skip(skip)
      .limit(limit);
    
    const total = await Product.countDocuments(query);
    
    res.json({
      success: true,
      data: products,
      pagination: {
        page,
        limit,
        total,
        totalPages: Math.ceil(total / limit)
      }
    });
    
  } catch (error) {
    res.status(500).json({
      success: false,
      error: "Failed to fetch products"
    });
  }
};
```

**Usage**:
```
GET /api/products?category=electronics&minPrice=100&sort=-price&page=1&limit=20&fields=name,price
```

---

## 4. Rate Limiting

**Problem**: Prevent abuse - stop users from making too many requests.

**Solution**: Limit requests per time window.

### 4.1 Setup

```bash
npm install express-rate-limit
```

### 4.2 Basic Rate Limiting

**File**: `src/middlewares/rateLimiter.js`

```js
const rateLimit = require("express-rate-limit");

// ============================================
// GENERAL API RATE LIMIT
// ============================================
// Apply to all API routes
const generalLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,  // 15 minutes
  max: 100,                   // Max 100 requests per window
  message: {
    success: false,
    error: "Too many requests, please try again later"
  },
  standardHeaders: true,  // Return rate limit info in headers
  legacyHeaders: false
});

// ============================================
// STRICT RATE LIMIT
// ============================================
// For sensitive endpoints (login, signup)
const strictLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,  // 15 minutes
  max: 5,                     // Max 5 requests per window
  message: {
    success: false,
    error: "Too many attempts, please try again after 15 minutes"
  }
});

// ============================================
// CREATE ACCOUNT RATE LIMIT
// ============================================
// Prevent automated account creation
const createAccountLimiter = rateLimit({
  windowMs: 60 * 60 * 1000,  // 1 hour
  max: 3,                     // Max 3 accounts per IP per hour
  message: {
    success: false,
    error: "Too many accounts created from this IP"
  }
});

module.exports = {
  generalLimiter,
  strictLimiter,
  createAccountLimiter
};
```

### 4.3 Apply Rate Limiting

**File**: `src/app.js`

```js
const express = require("express");
const { generalLimiter } = require("./middlewares/rateLimiter");

const app = express();

// Apply to all /api routes
app.use("/api", generalLimiter);

// Your routes...
module.exports = app;
```

**File**: `src/routes/auth.routes.js`

```js
const express = require("express");
const { strictLimiter, createAccountLimiter } = require("../middlewares/rateLimiter");
const authController = require("../controllers/auth.controller");

const router = express.Router();

// Apply strict limiter to login
router.post("/login", strictLimiter, authController.login);

// Apply create account limiter to signup
router.post("/signup", createAccountLimiter, authController.signup);

module.exports = router;
```

### 4.4 Response Headers

When rate limited, the response includes headers:

```
X-RateLimit-Limit: 100          # Max requests allowed
X-RateLimit-Remaining: 95       # Requests left
X-RateLimit-Reset: 1642345678   # When limit resets (Unix timestamp)
```

**Client can check these** before making requests.

---

## 5. API Versioning

**Problem**: When you change your API, old clients break.

**Solution**: Version your API so old and new clients can coexist.

### 5.1 URL Versioning (Recommended)

**Most common approach** - version in the URL path.

```
/api/v1/users
/api/v2/users
```

**Structure**:

```js
// File: src/routes/index.js
const express = require("express");
const v1Routes = require("./v1");
const v2Routes = require("./v2");

const router = express.Router();

router.use("/v1", v1Routes);
router.use("/v2", v2Routes);

module.exports = router;
```

```js
// File: src/routes/v1/index.js
const express = require("express");
const userRoutes = require("./user.routes");

const router = express.Router();

router.use("/users", userRoutes);

module.exports = router;
```

```js
// File: src/routes/v1/user.routes.js
const express = require("express");
const userController = require("../../controllers/v1/user.controller");

const router = express.Router();

router.get("/", userController.getUsers);
router.post("/", userController.createUser);

module.exports = router;
```

**Folder structure**:

```
src/
├── routes/
│   ├── index.js
│   ├── v1/
│   │   ├── index.js
│   │   ├── user.routes.js
│   │   └── post.routes.js
│   └── v2/
│       ├── index.js
│       ├── user.routes.js
│       └── post.routes.js
└── controllers/
    ├── v1/
    │   └── user.controller.js
    └── v2/
        └── user.controller.js
```

### 5.2 When to Create a New Version

Create a new version when you make **breaking changes**:

```
✅ Breaking (needs new version):
- Removing a field from response
- Changing field name (username → name)
- Changing field type (id: string → id: number)
- Removing an endpoint

❌ Non-breaking (same version):
- Adding a new field to response
- Adding a new endpoint
- Adding optional query parameters
```

### 5.3 Deprecation

When releasing v2, keep v1 alive with deprecation notice:

```js
// File: src/routes/v1/index.js
const express = require("express");
const router = express.Router();

// Deprecation middleware for all v1 routes
router.use((req, res, next) => {
  res.setHeader("X-API-Deprecated", "true");
  res.setHeader("X-API-Sunset", "2024-12-31");  // When v1 will be removed
  next();
});

// Routes...

module.exports = router;
```

---

## 6. Error Response Patterns

**Consistent error format** across your entire API.

### 6.1 Standard Error Response

```js
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid input data",
    "details": [
      {
        "field": "email",
        "message": "Email is required"
      },
      {
        "field": "password",
        "message": "Password must be at least 8 characters"
      }
    ]
  }
}
```

### 6.2 Error Handler Middleware

**File**: `src/middlewares/errorHandler.js`

```js
class AppError extends Error {
  constructor(message, statusCode, code, details = []) {
    super(message);
    this.statusCode = statusCode;
    this.code = code;
    this.details = details;
    this.isOperational = true;  // Distinguish from programming errors
  }
}

function errorHandler(err, req, res, next) {
  // Default to 500 if statusCode not set
  const statusCode = err.statusCode || 500;
  
  // Build error response
  const errorResponse = {
    success: false,
    error: {
      code: err.code || "INTERNAL_ERROR",
      message: err.message || "Something went wrong"
    }
  };
  
  // Add details if present (validation errors)
  if (err.details && err.details.length > 0) {
    errorResponse.error.details = err.details;
  }
  
  // In production, hide stack traces
  if (process.env.NODE_ENV !== "production") {
    errorResponse.error.stack = err.stack;
  }
  
  res.status(statusCode).json(errorResponse);
}

module.exports = { AppError, errorHandler };
```

### 6.3 Usage in Controllers

```js
const { AppError } = require("../middlewares/errorHandler");

exports.createUser = async (req, res, next) => {
  try {
    const { email, password } = req.body;
    
    // Validation
    if (!email || !password) {
      const details = [];
      if (!email) details.push({ field: "email", message: "Email is required" });
      if (!password) details.push({ field: "password", message: "Password is required" });
      
      throw new AppError(
        "Validation failed",
        400,
        "VALIDATION_ERROR",
        details
      );
    }
    
    // Check duplicate
    const existing = await User.findOne({ email });
    if (existing) {
      throw new AppError(
        "Email already in use",
        409,
        "DUPLICATE_EMAIL"
      );
    }
    
    const user = await User.create({ email, password });
    res.status(201).json({ success: true, data: user });
    
  } catch (error) {
    next(error);  // Pass to error handler
  }
};
```

---

## 7. Webhooks

**What are webhooks?**

Instead of clients polling your API for updates, your server **pushes updates** to the client's server when events happen.

**Example**: Stripe payment successful → Stripe calls your webhook → You update order status.

### 7.1 Receiving Webhooks

**File**: `src/routes/webhook.routes.js`

```js
const express = require("express");
const webhookController = require("../controllers/webhook.controller");

const router = express.Router();

// Webhook endpoint - no authentication (verified via signature)
router.post("/stripe", 
  express.raw({ type: "application/json" }),  // Important: raw body for signature verification
  webhookController.handleStripeWebhook
);

module.exports = router;
```

**File**: `src/controllers/webhook.controller.js`

```js
const stripe = require("stripe")(process.env.STRIPE_SECRET_KEY);
const Order = require("../models/Order.model");

exports.handleStripeWebhook = async (req, res) => {
  const sig = req.headers["stripe-signature"];
  const webhookSecret = process.env.STRIPE_WEBHOOK_SECRET;
  
  let event;
  
  try {
    // Verify webhook signature (prevents fake requests)
    event = stripe.webhooks.constructEvent(req.body, sig, webhookSecret);
    
  } catch (err) {
    console.log("⚠️  Webhook signature verification failed:", err.message);
    return res.status(400).send(`Webhook Error: ${err.message}`);
  }
  
  // Handle different event types
  switch (event.type) {
    case "payment_intent.succeeded":
      const paymentIntent = event.data.object;
      
      // Update order status
      await Order.findOneAndUpdate(
        { paymentIntentId: paymentIntent.id },
        { status: "paid", paidAt: new Date() }
      );
      
      console.log("✓ Payment succeeded:", paymentIntent.id);
      break;
      
    case "payment_intent.payment_failed":
      const failedPayment = event.data.object;
      
      await Order.findOneAndUpdate(
        { paymentIntentId: failedPayment.id },
        { status: "failed" }
      );
      
      console.log("✗ Payment failed:", failedPayment.id);
      break;
      
    default:
      console.log(`Unhandled event type: ${event.type}`);
  }
  
  // Always return 200 to acknowledge receipt
  res.json({ received: true });
};
```

### 7.2 Sending Webhooks

**When you want to notify other services** about events in your system.

**File**: `src/utils/webhook.js`

```js
const axios = require("axios");
const crypto = require("crypto");

async function sendWebhook(url, event, data) {
  try {
    // Generate signature for security
    const payload = JSON.stringify({ event, data, timestamp: Date.now() });
    const signature = crypto
      .createHmac("sha256", process.env.WEBHOOK_SECRET)
      .update(payload)
      .digest("hex");
    
    // Send POST request to client's webhook URL
    const response = await axios.post(url, { event, data }, {
      headers: {
        "Content-Type": "application/json",
        "X-Webhook-Signature": signature
      },
      timeout: 5000  // 5 second timeout
    });
    
    console.log("✓ Webhook sent:", event, response.status);
    return true;
    
  } catch (error) {
    console.error("✗ Webhook failed:", event, error.message);
    // Optionally: retry logic, save to queue for later retry
    return false;
  }
}

module.exports = { sendWebhook };
```

**Usage**:

```js
const { sendWebhook } = require("../utils/webhook");

exports.createOrder = async (req, res) => {
  try {
    const order = await Order.create(req.body);
    
    // Send webhook notification
    await sendWebhook(
      "https://client-app.com/webhooks/order-created",
      "order.created",
      {
        orderId: order._id,
        amount: order.amount,
        status: order.status
      }
    );
    
    res.status(201).json({ success: true, data: order });
    
  } catch (error) {
    res.status(500).json({ success: false, error: error.message });
  }
};
```

---

## 8. HATEOAS (Hypermedia)

**HATEOAS** (Hypermedia as the Engine of Application State) - include links in responses to guide clients on what actions are available.

**Simple explanation**: Like a website has links to navigate, your API responses include links to related resources.

### 8.1 Basic Example

**Without HATEOAS**:
```json
{
  "id": "user_123",
  "name": "Alice",
  "email": "alice@example.com"
}
```

**With HATEOAS**:
```json
{
  "id": "user_123",
  "name": "Alice",
  "email": "alice@example.com",
  "_links": {
    "self": { "href": "/api/users/user_123" },
    "posts": { "href": "/api/users/user_123/posts" },
    "update": { "href": "/api/users/user_123", "method": "PUT" },
    "delete": { "href": "/api/users/user_123", "method": "DELETE" }
  }
}
```

### 8.2 Implementation

```js
exports.getUser = async (req, res) => {
  try {
    const { id } = req.params;
    const user = await User.findById(id);
    
    if (!user) {
      return res.status(404).json({
        success: false,
        error: "User not found"
      });
    }
    
    // Add HATEOAS links
    const response = {
      success: true,
      data: {
        id: user._id,
        name: user.name,
        email: user.email,
        _links: {
          self: { href: `/api/users/${user._id}` },
          posts: { href: `/api/users/${user._id}/posts` },
          update: { href: `/api/users/${user._id}`, method: "PUT" },
          delete: { href: `/api/users/${user._id}`, method: "DELETE" }
        }
      }
    };
    
    res.json(response);
    
  } catch (error) {
    res.status(500).json({
      success: false,
      error: "Failed to fetch user"
    });
  }
};
```

**Benefits**:
- Clients discover available actions without hardcoding URLs
- API is more self-documenting
- Easier to change URLs without breaking clients

**Downsides**:
- Larger response size
- More complex to implement
- Not widely adopted in practice

**Verdict**: Optional - most modern APIs don't use full HATEOAS. But including a `self` link is useful.

---

## 9. Complete REST API Example

Combining all concepts:

**File**: `src/routes/api.js`

```js
const express = require("express");
const { generalLimiter } = require("../middlewares/rateLimiter");
const v1Routes = require("./v1");

const router = express.Router();

// Rate limiting for all API routes
router.use(generalLimiter);

// API versioning
router.use("/v1", v1Routes);

module.exports = router;
```

**File**: `src/routes/v1/product.routes.js`

```js
const express = require("express");
const productController = require("../../controllers/v1/product.controller");
const authMiddleware = require("../../middlewares/auth.middleware");

const router = express.Router();

// Public routes
router.get("/", productController.getProducts);         // Supports pagination, filtering, sorting
router.get("/:id", productController.getProduct);

// Protected routes
router.post("/", authMiddleware, productController.createProduct);
router.put("/:id", authMiddleware, productController.updateProduct);
router.delete("/:id", authMiddleware, productController.deleteProduct);

module.exports = router;
```

**File**: `src/controllers/v1/product.controller.js`

```js
const Product = require("../../models/Product.model");
const { AppError } = require("../../middlewares/errorHandler");

exports.getProducts = async (req, res, next) => {
  try {
    // Pagination
    const page = parseInt(req.query.page) || 1;
    const limit = Math.min(parseInt(req.query.limit) || 20, 100);
    const skip = (page - 1) * limit;
    
    // Filtering
    const query = {};
    if (req.query.category) query.category = req.query.category;
    if (req.query.minPrice) {
      query.price = { ...query.price, $gte: parseFloat(req.query.minPrice) };
    }
    if (req.query.maxPrice) {
      query.price = { ...query.price, $lte: parseFloat(req.query.maxPrice) };
    }
    
    // Sorting
    let sort = { createdAt: -1 };
    if (req.query.sort) {
      sort = {};
      req.query.sort.split(",").forEach(field => {
        sort[field.startsWith("-") ? field.substring(1) : field] = 
          field.startsWith("-") ? -1 : 1;
      });
    }
    
    // Execute query
    const products = await Product.find(query)
      .sort(sort)
      .skip(skip)
      .limit(limit);
    
    const total = await Product.countDocuments(query);
    
    res.json({
      success: true,
      data: products,
      pagination: {
        page,
        limit,
        total,
        totalPages: Math.ceil(total / limit),
        hasNextPage: page < Math.ceil(total / limit),
        hasPrevPage: page > 1
      }
    });
    
  } catch (error) {
    next(error);
  }
};

exports.getProduct = async (req, res, next) => {
  try {
    const product = await Product.findById(req.params.id);
    
    if (!product) {
      throw new AppError("Product not found", 404, "NOT_FOUND");
    }
    
    res.json({
      success: true,
      data: {
        ...product.toObject(),
        _links: {
          self: { href: `/api/v1/products/${product._id}` },
          update: { href: `/api/v1/products/${product._id}`, method: "PUT" },
          delete: { href: `/api/v1/products/${product._id}`, method: "DELETE" }
        }
      }
    });
    
  } catch (error) {
    next(error);
  }
};

exports.createProduct = async (req, res, next) => {
  try {
    const { name, price, category } = req.body;
    
    // Validation
    if (!name || !price || !category) {
      const details = [];
      if (!name) details.push({ field: "name", message: "Name is required" });
      if (!price) details.push({ field: "price", message: "Price is required" });
      if (!category) details.push({ field: "category", message: "Category is required" });
      
      throw new AppError("Validation failed", 400, "VALIDATION_ERROR", details);
    }
    
    const product = await Product.create(req.body);
    
    res.status(201).json({
      success: true,
      data: product
    });
    
  } catch (error) {
    next(error);
  }
};
```

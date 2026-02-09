# Authentication and Authorization in NodeJS

## Core Concepts

**Authentication**: Verifies *who you are* using credentials (passwords, OTPs, biometrics)

**Authorization**: Verifies *what you can do* based on permissions, policies, or roles

---

## Objective

Build secure APIs supporting multiple client types:
- **Web browsers** → JWT in HTTP-only cookies
- **Mobile apps** → JWT in Authorization headers
- **External APIs** → JWT in Authorization headers

**Key requirement**: Backend must handle both cookie-based and header-based token delivery simultaneously.

---

## Project Structure

```text
src/
 ├── app.js                      # Express app configuration
 ├── server.js                   # Server entry point
 ├── config/
 │    └── jwt.js                 # JWT settings (secret, expiry)
 ├── controllers/
 │    └── auth.controller.js     # Request handling logic
 ├── middlewares/
 │    ├── auth.middleware.js     # Token verification
 │    └── role.middleware.js     # Role-based access control
 ├── routes/
 │    └── auth.routes.js         # Endpoint definitions
 └── services/
      └── token.service.js       # JWT creation & validation
```

**Layer Responsibilities**:
- **Controllers**: Handle HTTP requests/responses
- **Services**: Reusable business logic (token operations)
- **Middlewares**: Security filters (authentication, authorization)
- **Routes**: Map URLs to controllers
- **Config**: Environment-based configuration

---

## Implementation

### 1. JWT Configuration

**File**: `src/config/jwt.js`

```js
module.exports = {
  secret: process.env.JWT_SECRET,      // Signing key from environment
  accessExpiry: "15m"                  // Token validity period
};
```

**Why**: Centralizes JWT settings for easy updates and environment-specific configs.

---

### 2. Token Service

**File**: `src/services/token.service.js`

```js
const jwt = require("jsonwebtoken");
const config = require("../config/jwt");

function generateToken(payload) {
  return jwt.sign(payload, config.secret, {
    expiresIn: config.accessExpiry
  });
}

function verifyToken(token) {
  return jwt.verify(token, config.secret);
}

module.exports = {
  generateToken,
  verifyToken
};
```

**Key Points**:
- `generateToken()`: Creates signed JWT with expiration
- `verifyToken()`: Validates signature and checks expiration
- Throws error if token is expired or tampered with

---

### 3. Auth Controller

**File**: `src/controllers/auth.controller.js`

```js
const tokenService = require("../services/token.service");

async function login(req, res) {
  const { email, password } = req.body;

  // Validate credentials (DB logic omitted for brevity)
  const user = { id: "123", email };

  if (!user) {
    return res.sendStatus(401);  // Unauthorized
  }

  const token = tokenService.generateToken({
    id: user.id
  });

  // For web clients: Set HTTP-only cookie
  res.cookie("access_token", token, {
    httpOnly: true,      // Prevents JavaScript access (XSS protection)
    secure: true,        // HTTPS only
    sameSite: "strict",  // CSRF protection
    maxAge: 15 * 60 * 1000  // 15 minutes in milliseconds
  });

  // For mobile/API clients: Return token in response body
  res.json({
    accessToken: token
  });
}

async function logout(req, res) {
  res.clearCookie("access_token");
  res.sendStatus(200);
}

module.exports = {
  login,
  logout
};
```

**Design Pattern**: Dual token delivery
- Browsers automatically send cookies with requests
- Mobile apps must manually add token to headers

---

### 4. Auth Middleware

**File**: `src/middlewares/auth.middleware.js`

```js
const tokenService = require("../services/token.service");

function auth(req, res, next) {
  let token;

  // Strategy 1: Check for cookie (web clients)
  if (req.cookies?.access_token) {
    token = req.cookies.access_token;
  }

  // Strategy 2: Check Authorization header (mobile/API clients)
  if (!token && req.headers.authorization) {
    const parts = req.headers.authorization.split(" ");

    if (parts[0] === "Bearer" && parts[1]) {
      token = parts[1];
    }
  }

  if (!token) {
    return res.sendStatus(401);  // No token provided
  }

  try {
    const decoded = tokenService.verifyToken(token);
    req.user = decoded;  // Attach user data to request
    next();              // Proceed to next middleware/controller
  } catch {
    return res.sendStatus(403);  // Invalid/expired token
  }
}

module.exports = auth;
```

**Flow**:
1. Extract token from cookie OR header
2. Verify token signature and expiration
3. Attach decoded payload to `req.user`
4. Pass control to protected route

**Status Codes**:
- `401`: Missing token (not authenticated)
- `403`: Invalid token (authenticated but forbidden)

---

### 5. Auth Routes

**File**: `src/routes/auth.routes.js`

```js
const express = require("express");
const authController = require("../controllers/auth.controller");
const auth = require("../middlewares/auth.middleware");

const router = express.Router();

router.post("/login", authController.login);

router.post("/logout", authController.logout);

// Protected route example
router.get("/me", auth, (req, res) => {
  res.json({
    user: req.user  // Data from decoded token
  });
});

module.exports = router;
```

**Middleware chaining**: `auth` runs before route handler, ensuring only authenticated requests proceed.

---

### 6. App Initialization

**File**: `src/app.js`

```js
const express = require("express");
const cookieParser = require("cookie-parser");

const authRoutes = require("./routes/auth.routes");

const app = express();

app.use(express.json());        // Parse JSON request bodies
app.use(cookieParser());        // Parse cookies from requests

app.use("/auth", authRoutes);   // Mount auth routes at /auth/*

module.exports = app;
```

**Dependencies**:
- `express.json()`: Enables `req.body` for JSON payloads
- `cookieParser()`: Enables `req.cookies` for cookie access

---

### 7. Server Entry

**File**: `src/server.js`

```js
require("dotenv").config();  // Load environment variables

const app = require("./app");

app.listen(3000, () => {
  console.log("Server running on port 3000");
});
```

**Environment Variables Required**:
- `JWT_SECRET`: Strong random string for signing tokens

---

## Role-Based Authorization

### Concept
After authentication, check if user has required role/permission for the requested resource.

### Implementation

**Step 1**: Include role in token payload

```js
const token = tokenService.generateToken({
  id: user.id,
  role: user.role  // e.g., "admin", "user", "moderator"
});
```

**Step 2**: Create role middleware

**File**: `src/middlewares/role.middleware.js`

```js
function allowRoles(...roles) {
  return (req, res, next) => {
    if (!roles.includes(req.user.role)) {
      return res.sendStatus(403);  // Forbidden: insufficient permissions
    }
    next();
  };
}

module.exports = allowRoles;
```

**How it works**:
- Returns a middleware function (closure pattern)
- Accepts variable number of allowed roles using rest parameter (`...roles`)
- Checks if user's role matches any allowed role

**Step 3**: Apply to routes

```js
const auth = require("../middlewares/auth.middleware");
const allowRoles = require("../middlewares/role.middleware");

router.get(
  "/admin/dashboard",
  auth,                    // First: verify token
  allowRoles("admin"),     // Second: check role
  adminController.dashboard
);

router.delete(
  "/posts/:id",
  auth,
  allowRoles("admin", "moderator"),  // Multiple roles allowed
  postController.delete
);
```

**Middleware order matters**:
1. `auth` → Ensures user is logged in and attaches `req.user`
2. `allowRoles` → Checks `req.user.role` against allowed roles
3. Controller → Executes if both checks pass

---

## Security Best Practices

1. **HTTP-only cookies**: Prevent XSS attacks by blocking JavaScript access
2. **Secure flag**: Require HTTPS to prevent token interception
3. **SameSite**: Mitigate CSRF attacks by restricting cross-site requests
4. **Short expiration**: Minimize damage window if token is compromised
5. **Strong secrets**: Use cryptographically random `JWT_SECRET` (min 256 bits)

---

## Testing Flow

### Web Client (Cookie-based)
```bash
# Login (cookie set automatically)
curl -X POST http://localhost:3000/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"user@example.com","password":"pass123"}' \
  -c cookies.txt

# Access protected route
curl http://localhost:3000/auth/me -b cookies.txt
```

### Mobile/API Client (Header-based)
```bash
# Login (save token from response)
TOKEN=$(curl -X POST http://localhost:3000/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"user@example.com","password":"pass123"}' \
  | jq -r '.accessToken')

# Access protected route
curl http://localhost:3000/auth/me \
  -H "Authorization: Bearer $TOKEN"
```

---

## Common Pitfalls

1. **Storing JWT in localStorage**: Vulnerable to XSS → Use HTTP-only cookies for web
2. **Not validating token expiration**: Always check `exp` claim
3. **Weak secrets**: Use `crypto.randomBytes(32).toString('hex')` to generate
4. **Mixing authentication and authorization**: Keep token verification separate from role checks
5. **Forgetting error handling**: Token verification can throw errors (invalid signature, expired)

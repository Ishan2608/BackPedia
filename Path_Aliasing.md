# Module/Path Aliasing in NodeJS

## What is Path Aliasing?

**Simple explanation**: Instead of writing long, messy import paths like `../../../../config/database.js`, you create shortcuts like `@config/database.js`.

**Technical explanation**: Path aliasing maps custom prefixes to absolute file system paths, allowing you to import modules using cleaner, more maintainable identifiers instead of relative paths.

### The Problem It Solves

```js
// Without aliasing - Hard to maintain, breaks when files move
const db = require("../../../../config/database");
const auth = require("../../../middlewares/auth.middleware");
const User = require("../../models/User.model");

// With aliasing - Clean, consistent, location-independent
const db = require("@config/database");
const auth = require("@middlewares/auth.middleware");
const User = require("@models/User.model");
```

**Benefits**:
- Easier refactoring (move files without updating imports)
- Better readability (clear module locations)
- Reduced errors (no miscounting `../` levels)
- IDE autocomplete support
- Consistent import style across team

---

## Common Alias Patterns

| Alias | Maps To | Purpose |
|-------|---------|---------|
| `@` | `src/` | Root source directory |
| `@config` | `src/config/` | Configuration files |
| `@controllers` | `src/controllers/` | Request handlers |
| `@models` | `src/models/` | Database schemas |
| `@middlewares` | `src/middlewares/` | Express middlewares |
| `@services` | `src/services/` | Business logic |
| `@utils` | `src/utils/` | Helper functions |
| `@routes` | `src/routes/` | API routes |

**Naming convention**: Use `@` prefix to distinguish aliases from npm packages.

---

## Implementation Methods

There are three main approaches to implement path aliasing in NodeJS:

### Method 1: module-alias Package (Simplest)
### Method 2: Node.js Subpath Imports (Native, Node 12.20+)
### Method 3: Babel (Build-time transformation)

We'll explore each method with examples.

---

## Method 1: Using `module-alias` Package

**Best for**: Quick setup, production-ready apps without build tools.

### What is `module-alias`?

A lightweight npm package that registers custom path aliases at runtime. It modifies Node's `require()` function to resolve your custom paths.

**Key concept**: It works by hooking into Node's module resolution system *before* your code runs.

---

### Step-by-Step Setup

#### Step 1: Install the Package

```bash
npm install module-alias
```

#### Step 2: Configure Aliases in `package.json`

**File**: `package.json`

```json
{
  "name": "your-project",
  "version": "1.0.0",
  "_moduleAliases": {
    "@": "src",
    "@config": "src/config",
    "@controllers": "src/controllers",
    "@middlewares": "src/middlewares",
    "@models": "src/models",
    "@routes": "src/routes",
    "@services": "src/services",
    "@utils": "src/utils"
  },
  "scripts": {
    "start": "node src/server.js"
  }
}
```

**Note**: The `_moduleAliases` key is specific to `module-alias` package. Paths are relative to your `package.json` location.

#### Step 3: Register Aliases in Entry File

**File**: `src/server.js`

```js
// CRITICAL: This MUST be the first line before any imports
require("module-alias/register");

// Now you can use aliases
require("dotenv").config();
const app = require("@/app");
const config = require("@config/database");

app.listen(3000, () => {
  console.log("Server running");
});
```

**Why first?**: The `register` call modifies Node's module system. If you import files before registration, they won't benefit from aliases.

---

### Usage Examples

#### Before (Relative Paths)

```js
// src/controllers/user.controller.js
const userService = require("../services/user.service");
const authMiddleware = require("../middlewares/auth.middleware");
const { hashPassword } = require("../utils/crypto");
const config = require("../../config/app.config");
```

#### After (Aliases)

```js
// src/controllers/user.controller.js
const userService = require("@services/user.service");
const authMiddleware = require("@middlewares/auth.middleware");
const { hashPassword } = require("@utils/crypto");
const config = require("@config/app.config");
```

---

### Project Structure Example

```text
project-root/
 ├── package.json          # Alias definitions here
 ├── src/
 │    ├── server.js        # Register aliases first
 │    ├── app.js
 │    ├── config/
 │    │    ├── database.js
 │    │    └── jwt.js
 │    ├── controllers/
 │    │    └── auth.controller.js
 │    ├── middlewares/
 │    │    └── auth.middleware.js
 │    ├── models/
 │    │    └── User.model.js
 │    ├── routes/
 │    │    └── auth.routes.js
 │    ├── services/
 │    │    └── token.service.js
 │    └── utils/
 │         └── logger.js
 └── node_modules/
```

---

### Complete Working Example

**File**: `src/app.js`

```js
const express = require("express");
const cookieParser = require("cookie-parser");

// Using aliases
const authRoutes = require("@routes/auth.routes");
const errorHandler = require("@middlewares/error.middleware");
const logger = require("@utils/logger");

const app = express();

app.use(express.json());
app.use(cookieParser());

// Routes
app.use("/auth", authRoutes);

// Error handling
app.use(errorHandler);

module.exports = app;
```

**File**: `src/controllers/auth.controller.js`

```js
// Clean imports with aliases
const tokenService = require("@services/token.service");
const User = require("@models/User.model");
const config = require("@config/jwt");
const { hashPassword, comparePassword } = require("@utils/crypto");

async function login(req, res) {
  const { email, password } = req.body;
  
  const user = await User.findByEmail(email);
  if (!user || !(await comparePassword(password, user.passwordHash))) {
    return res.status(401).json({ error: "Invalid credentials" });
  }

  const token = tokenService.generateToken({ id: user.id });
  
  res.cookie("access_token", token, {
    httpOnly: true,
    secure: true,
    sameSite: "strict"
  });
  
  res.json({ accessToken: token });
}

module.exports = { login };
```

---

### Pros and Cons

**Advantages**:
- Simple setup (no build tools needed)
- Works with plain Node.js
- Runtime resolution (no compilation step)
- Good for small-to-medium projects

**Disadvantages**:
- Slightly slower (runtime overhead)
- Not recognized by some linters/IDEs without config
- Only works with CommonJS (`require`), not ES6 modules (`import`)

---

## Method 2: Node.js Subpath Imports (Native)

**Best for**: Modern Node.js projects (v12.20+) wanting native support without external packages.

### What are Subpath Imports?

A built-in Node.js feature that allows defining custom import paths in `package.json` using the `#` prefix.

**Key concept**: Uses Node's native package resolution. The `#` prefix distinguishes internal imports from external packages.

---

### Step-by-Step Setup

#### Step 1: Configure in `package.json`

**File**: `package.json`

```json
{
  "name": "your-project",
  "version": "1.0.0",
  "imports": {
    "#config/*": "./src/config/*.js",
    "#controllers/*": "./src/controllers/*.js",
    "#middlewares/*": "./src/middlewares/*.js",
    "#models/*": "./src/models/*.js",
    "#routes/*": "./src/routes/*.js",
    "#services/*": "./src/services/*.js",
    "#utils/*": "./src/utils/*.js"
  }
}
```

**Syntax breakdown**:
- `#config/*`: The alias pattern (must start with `#`)
- `./src/config/*.js`: The actual path with wildcard
- `*`: Placeholder that matches any filename

#### Step 2: Use in Your Code

```js
// src/controllers/auth.controller.js
const tokenService = require("#services/token.service");
const User = require("#models/User.model");
const config = require("#config/jwt");
```

**Important**: Must use `.js` extension in mapped paths, but **not** when importing:

```js
// ✅ Correct
const config = require("#config/jwt");

// ❌ Wrong - don't add .js when importing
const config = require("#config/jwt.js");
```

---

### Advanced Configuration

You can map multiple patterns:

```json
{
  "imports": {
    "#config/*": "./src/config/*.js",
    "#lib/*": "./src/lib/*.js",
    "#lib": "./src/lib/index.js",  // Default export
    "#root/*": "./*.js"             // Project root
  }
}
```

**Usage**:
```js
const utils = require("#lib");              // Imports src/lib/index.js
const helper = require("#lib/helper");      // Imports src/lib/helper.js
const config = require("#root/config");     // Imports ./config.js
```

---

### Pros and Cons

**Advantages**:
- Native Node.js feature (no dependencies)
- Zero runtime overhead
- Official and stable
- Works with both CommonJS and ES6 modules

**Disadvantages**:
- Requires Node 12.20+ (not for older projects)
- Must use `#` prefix (can't use `@`)
- Wildcards can be less flexible than `module-alias`
- Some older IDEs might not support autocomplete

---

## Method 3: Babel Plugin (Build-time)

**Best for**: Projects already using Babel, or when you need ES6 `import` syntax with custom aliases.

### What is Babel?

**Simple explanation**: A JavaScript compiler that transforms modern code into older versions for compatibility.

**Technical explanation**: Babel is a toolchain that converts ECMAScript 2015+ code into backwards-compatible JavaScript. It can also transform syntax extensions like JSX and apply plugins for features like path aliasing.

**Why use it for aliases?**: Babel transforms your code *before* execution, replacing aliases with actual paths at build time.

---

### Key Concepts You Need to Know

#### 1. Babel Basics

Babel works in two steps:
1. **Parsing**: Reads your code into an Abstract Syntax Tree (AST)
2. **Transformation**: Applies plugins to modify the AST
3. **Code generation**: Converts AST back to JavaScript

#### 2. Plugins vs Presets

- **Plugin**: A single transformation (e.g., transform arrow functions)
- **Preset**: A collection of plugins (e.g., `@babel/preset-env` includes many plugins)

#### 3. Babel Configuration Files

Babel reads settings from:
- `.babelrc` (JSON format)
- `.babelrc.js` (JavaScript format, allows logic)
- `babel.config.js` (Project-wide config)
- `package.json` (in a `"babel"` key)

---

### Step-by-Step Setup

#### Step 1: Install Dependencies

```bash
# Core Babel packages
npm install --save-dev @babel/core @babel/cli @babel/node

# Path aliasing plugin
npm install --save-dev babel-plugin-module-resolver
```

**What each does**:
- `@babel/core`: Babel's main engine
- `@babel/cli`: Run Babel from command line
- `@babel/node`: Run Node.js files through Babel (like `node` but with transformations)
- `babel-plugin-module-resolver`: The plugin that handles path aliases

#### Step 2: Create Babel Configuration

**File**: `.babelrc` or `babel.config.js`

```json
{
  "plugins": [
    [
      "module-resolver",
      {
        "root": ["./src"],
        "alias": {
          "@": "./src",
          "@config": "./src/config",
          "@controllers": "./src/controllers",
          "@middlewares": "./src/middlewares",
          "@models": "./src/models",
          "@routes": "./src/routes",
          "@services": "./src/services",
          "@utils": "./src/utils"
        }
      }
    ]
  ]
}
```

**Configuration breakdown**:
- `root`: Base directory for resolution
- `alias`: Object mapping aliases to paths

#### Step 3: Update npm Scripts

**File**: `package.json`

```json
{
  "scripts": {
    "start": "babel-node src/server.js",
    "dev": "nodemon --exec babel-node src/server.js",
    "build": "babel src -d dist",
    "serve": "node dist/server.js"
  }
}
```

**Script explanations**:
- `babel-node`: Runs files through Babel on-the-fly (development only, slower)
- `babel src -d dist`: Compiles all files from `src/` to `dist/` directory
- `node dist/server.js`: Runs compiled code (production)

---

### Usage with ES6 Imports

Now you can use modern `import` syntax with aliases:

```js
// src/controllers/auth.controller.js
import tokenService from "@services/token.service";
import User from "@models/User.model";
import config from "@config/jwt";
import { hashPassword } from "@utils/crypto";

export async function login(req, res) {
  // Your code here
}
```

**Note**: Babel transforms these to actual paths during compilation.

---

### What Happens During Build?

**Before compilation** (your code):
```js
import User from "@models/User.model";
```

**After compilation** (in `dist/`):
```js
const User = require("./models/User.model");
```

Babel replaces aliases with real relative paths based on file location.

---

### Adding ESLint Support

**What is ESLint?**

**Simple explanation**: A tool that checks your code for errors and enforces style rules.

**Technical explanation**: ESLint is a static code analysis tool that identifies problematic patterns in JavaScript. It's pluggable and configurable, allowing teams to enforce coding standards.

#### Why Configure ESLint for Aliases?

Without configuration, ESLint will show errors like "Unable to resolve path '@config/database'" because it doesn't know about your custom aliases.

#### Setup ESLint

```bash
npm install --save-dev eslint eslint-plugin-import eslint-import-resolver-babel-module
```

**File**: `.eslintrc.json`

```json
{
  "extends": "eslint:recommended",
  "plugins": ["import"],
  "settings": {
    "import/resolver": {
      "babel-module": {}
    }
  },
  "rules": {
    "import/no-unresolved": "error"
  }
}
```

**What this does**:
- `eslint-plugin-import`: Validates import/export syntax
- `eslint-import-resolver-babel-module`: Tells ESLint to read aliases from `.babelrc`
- Now ESLint understands `@config/database` is valid

---

### Adding TypeScript/JSDoc Support

If using TypeScript or JSDoc for type checking, configure path mapping:

**File**: `jsconfig.json` (for JavaScript projects)

```json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"],
      "@config/*": ["src/config/*"],
      "@controllers/*": ["src/controllers/*"],
      "@middlewares/*": ["src/middlewares/*"],
      "@models/*": ["src/models/*"],
      "@routes/*": ["src/routes/*"],
      "@services/*": ["src/services/*"],
      "@utils/*": ["src/utils/*"]
    }
  }
}
```

**Benefits**:
- VSCode autocomplete and IntelliSense
- Jump-to-definition support
- Import suggestions

---

### Pros and Cons

**Advantages**:
- Works with modern ES6 `import/export`
- Build-time transformation (zero runtime cost)
- Can use any prefix (`@`, `~`, `$`, etc.)
- Great IDE/editor support with proper config
- Can combine with other Babel transformations

**Disadvantages**:
- Requires build step (more complex setup)
- `babel-node` is slow (development only)
- Need to compile before production deployment
- Steeper learning curve for beginners

---

## Method Comparison Table

| Feature | module-alias | Subpath Imports | Babel |
|---------|-------------|-----------------|-------|
| **Setup complexity** | Easy | Easy | Moderate |
| **Node version** | Any | 12.20+ | Any |
| **Dependencies** | 1 package | None | 3+ packages |
| **ES6 imports** | ❌ No | ✅ Yes | ✅ Yes |
| **Build step** | ❌ No | ❌ No | ✅ Yes |
| **Runtime overhead** | Small | None | None |
| **IDE support** | Moderate | Good | Excellent |
| **Prefix options** | Any | Only `#` | Any |
| **Production ready** | ✅ Yes | ✅ Yes | ✅ Yes |

---

## Choosing the Right Method

### Use `module-alias` if:
- You're using CommonJS (`require`)
- You want quick setup with no build tools
- Your project is small-to-medium size
- You don't need ES6 `import` syntax

### Use Subpath Imports if:
- You're using Node 12.20+
- You prefer native features over packages
- You want zero dependencies
- The `#` prefix is acceptable

### Use Babel if:
- You're already using Babel for other transformations
- You need ES6 `import/export` syntax
- You want extensive IDE support
- You're building a large application with TypeScript/ESLint

---

## Best Practices

### 1. Consistent Naming Convention

```js
// ✅ Good - Clear, consistent prefixes
@config/database
@models/User.model
@utils/logger

// ❌ Bad - Mixed styles
config/database
@/models/User.model
utils/logger
```

### 2. Don't Over-Alias

```js
// ✅ Good - Alias common directories
const User = require("@models/User.model");

// ❌ Bad - Aliasing everything including npm packages
const express = require("@npm/express");  // Don't do this
```

**Rule**: Only alias *your* code, not external packages.

### 3. Document Your Aliases

Add a comment in your main entry file:

```js
/**
 * Path Aliases:
 * @config    -> src/config
 * @models    -> src/models
 * @utils     -> src/utils
 */
require("module-alias/register");
```

### 4. Keep Aliases Shallow

```js
// ✅ Good - Top-level directories
@config
@models
@utils

// ❌ Bad - Too granular
@config/database
@config/auth
@config/email
```

**Why**: Too many aliases become hard to remember and maintain.

### 5. Use Absolute Paths for External, Aliases for Internal

```js
// External packages - normal imports
const express = require("express");
const jwt = require("jsonwebtoken");

// Your code - use aliases
const config = require("@config/database");
const User = require("@models/User.model");
```

---

## Troubleshooting Common Issues

### Issue 1: "Cannot find module '@config/database'"

**Cause**: Aliases not registered or wrong path.

**Solutions**:
- Ensure `require("module-alias/register")` is **first line** in entry file
- Check alias path in `package.json` is correct
- Verify file actually exists at specified location

### Issue 2: ESLint shows "Unresolved import"

**Cause**: ESLint doesn't know about aliases.

**Solution**: Install and configure `eslint-import-resolver-babel-module` or `eslint-import-resolver-alias`.

### Issue 3: Aliases work in dev but not production

**Cause**: Forgot to register aliases in production entry point.

**Solution**: Ensure production `server.js` also calls `require("module-alias/register")`.

### Issue 4: VSCode autocomplete not working

**Cause**: Editor doesn't know about path mappings.

**Solution**: Create `jsconfig.json` with `paths` configuration (shown earlier).

### Issue 5: Babel aliases not working

**Cause**: `.babelrc` not being read or `babel-plugin-module-resolver` not installed.

**Solution**:
- Verify `.babelrc` is in project root
- Check plugin is listed in `dependencies` or `devDependencies`
- Ensure running with `babel-node`, not plain `node`

---

## Real-World Complete Setup

Here's a full example using `module-alias` for a production app:

### File Structure
```text
my-app/
 ├── package.json
 ├── .env
 ├── src/
 │    ├── server.js
 │    ├── app.js
 │    ├── config/
 │    │    ├── database.js
 │    │    └── jwt.js
 │    ├── controllers/
 │    │    └── auth.controller.js
 │    ├── middlewares/
 │    │    └── auth.middleware.js
 │    ├── models/
 │    │    └── User.model.js
 │    ├── routes/
 │    │    └── auth.routes.js
 │    ├── services/
 │    │    └── token.service.js
 │    └── utils/
 │         ├── logger.js
 │         └── crypto.js
 └── node_modules/
```

### Configuration Files

**`package.json`**:
```json
{
  "name": "auth-api",
  "version": "1.0.0",
  "main": "src/server.js",
  "scripts": {
    "start": "node src/server.js",
    "dev": "nodemon src/server.js"
  },
  "_moduleAliases": {
    "@": "src",
    "@config": "src/config",
    "@controllers": "src/controllers",
    "@middlewares": "src/middlewares",
    "@models": "src/models",
    "@routes": "src/routes",
    "@services": "src/services",
    "@utils": "src/utils"
  },
  "dependencies": {
    "express": "^4.18.0",
    "jsonwebtoken": "^9.0.0",
    "module-alias": "^2.2.2",
    "dotenv": "^16.0.0"
  },
  "devDependencies": {
    "nodemon": "^2.0.20"
  }
}
```

**`src/server.js`**:
```js
// MUST BE FIRST - Register aliases
require("module-alias/register");

// Load environment variables
require("dotenv").config();

// Now use aliases
const app = require("@/app");
const config = require("@config/database");

const PORT = process.env.PORT || 3000;

app.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});
```

**`src/app.js`**:
```js
const express = require("express");
const cookieParser = require("cookie-parser");

// Clean imports with aliases
const authRoutes = require("@routes/auth.routes");
const logger = require("@utils/logger");

const app = express();

// Middleware
app.use(express.json());
app.use(cookieParser());
app.use(logger);

// Routes
app.use("/auth", authRoutes);

// Error handling
app.use((err, req, res, next) => {
  console.error(err.stack);
  res.status(500).json({ error: "Something went wrong!" });
});

module.exports = app;
```

**`src/controllers/auth.controller.js`**:
```js
const tokenService = require("@services/token.service");
const User = require("@models/User.model");
const { hashPassword, comparePassword } = require("@utils/crypto");

async function register(req, res) {
  const { email, password } = req.body;

  // Hash password
  const passwordHash = await hashPassword(password);

  // Save user
  const user = await User.create({ email, passwordHash });

  // Generate token
  const token = tokenService.generateToken({ id: user.id });

  res.status(201).json({ token });
}

async function login(req, res) {
  const { email, password } = req.body;

  const user = await User.findByEmail(email);
  if (!user) {
    return res.status(401).json({ error: "Invalid credentials" });
  }

  const isValid = await comparePassword(password, user.passwordHash);
  if (!isValid) {
    return res.status(401).json({ error: "Invalid credentials" });
  }

  const token = tokenService.generateToken({ id: user.id });

  res.cookie("access_token", token, {
    httpOnly: true,
    secure: process.env.NODE_ENV === "production",
    sameSite: "strict"
  });

  res.json({ token });
}

module.exports = { register, login };
```

**`src/routes/auth.routes.js`**:
```js
const express = require("express");
const authController = require("@controllers/auth.controller");
const authMiddleware = require("@middlewares/auth.middleware");

const router = express.Router();

router.post("/register", authController.register);
router.post("/login", authController.login);
router.get("/me", authMiddleware, (req, res) => {
  res.json({ user: req.user });
});

module.exports = router;
```

---

## Summary

Path aliasing transforms messy relative imports into clean, maintainable code:

**Key takeaways**:
1. **Choose based on needs**: `module-alias` for simplicity, Subpath Imports for native support, Babel for full ES6
2. **Always register first**: Aliases must be configured before importing modules
3. **Configure tooling**: Add IDE, ESLint, and TypeScript configs for best experience
4. **Follow conventions**: Use `@` prefix, alias directories not files, keep it simple
5. **Document aliases**: Help future developers (including yourself) understand the system

With proper setup, path aliasing significantly improves code organization and developer experience in Node.js applications.

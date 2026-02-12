# Backend as a Service (BaaS)

## What is BaaS?

When you build a backend yourself, you write code for:
- A database
- Authentication
- File storage
- APIs to access all of the above
- A server to host it all

**BaaS bundles all of this into one managed platform.** You configure instead of code. You use their dashboard instead of writing server logic.

```
Building Backend Yourself:
┌──────────────────────────────────────────────────────────┐
│  YOUR CODE                                               │
│                                                          │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌────────┐    │
│  │  Auth    │  │ Database │  │ Storage  │  │  APIs  │    │
│  │  logic   │  │ queries  │  │ handler  │  │ routes │    │
│  └──────────┘  └──────────┘  └──────────┘  └────────┘    │
│       You write all of this                              │
└──────────────────────────────────────────────────────────┘

Using BaaS:
┌──────────────────────────────────────────────────────────┐
│  BAAS PLATFORM                                           │
│                                                          │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌────────┐    │
│  │  Auth    │  │ Database │  │ Storage  │  │  APIs  │    │
│  │  built-in│  │ built-in │  │ built-in │  │ auto   │    │
│  └──────────┘  └──────────┘  └──────────┘  └────────┘    │
│       You configure instead of code                      │
└──────────────────────────────────────────────────────────┘
```

**Supabase** → Built on PostgreSQL (relational, SQL)
**Firebase** → Built on Firestore (document-based, NoSQL)

---

## 1. The Database Layer

### 1.1 Two Fundamentally Different Models

Before understanding BaaS features, you must understand how these two databases think differently.

**Supabase (PostgreSQL) - Tables and Relations**

Data lives in structured tables with rows and columns. Tables relate to each other.

```
┌─────────────────────────┐      ┌──────────────────────────────┐
│        users            │      │           posts              │
├────┬────────┬──────────┤       ├────┬──────────┬──────────────┤
│ id │  name  │  email   │       │ id │ user_id  │    title     │
├────┼────────┼──────────┤       ├────┼──────────┼──────────────┤
│  1 │ Alice  │ a@e.com  │◄──┐   │  1 │    1     │ "Hello"      │
│  2 │ Bob    │ b@e.com  │   └───│  2 │    1     │ "World"      │
│  3 │ Charlie│ c@e.com  │       │  3 │    2     │ "Nice day"   │
└────┴────────┴──────────┘       └────┴──────────┴──────────────┘

posts.user_id links to users.id
This is called a FOREIGN KEY
```

**Firebase (Firestore) - Collections and Documents**

Data lives in nested collections and documents (like folders and JSON files).

```
firestore/
│
├── users/                           ← Collection
│   ├── user_abc123/                 ← Document (ID is random)
│   │   ├── name: "Alice"
│   │   ├── email: "a@e.com"
│   │   └── posts/                   ← Sub-collection (nested!)
│   │       ├── post_xyz/
│   │       │   └── title: "Hello"
│   │       └── post_def/
│   │           └── title: "World"
│   └── user_def456/
│       └── name: "Bob"
│
└── global_posts/                    ← Separate top-level collection
    ├── post_xyz/
    │   └── title: "Hello"
    └── post_abc/
        └── title: "Nice day"
```

**Why this matters**: Every feature (RLS, schemas, queries, real-time) works differently based on which model you're using.

---

## 2. Schemas (Supabase Only)

### 2.1 What is a Schema?

A **schema** is a namespace (container) for database tables. Think of it like a folder that groups related tables together.

```
Supabase Database
┌─────────────────────────────────────────────────────────┐
│                                                         │
│  Schema: "public"          Schema: "private"            │
│  ┌──────────────────┐      ┌──────────────────┐         │
│  │ users            │      │ audit_logs       │         │
│  │ posts            │      │ admin_users      │         │
│  │ comments         │      │ billing_records  │         │
│  └──────────────────┘      └──────────────────┘         │
│  Accessible via API         NOT accessible via API      │
│                                                         │
│  Schema: "auth"             Schema: "storage"           │
│  ┌──────────────────┐      ┌──────────────────┐         │
│  │ users            │      │ objects          │         │
│  │ sessions         │      │ buckets          │         │
│  └──────────────────┘      └──────────────────┘         │
│  Managed by Supabase        Managed by Supabase         │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### 2.2 Public vs Private Schema

**Public schema** = Tables exposed through Supabase's auto-generated API.

If you create a `users` table in the `public` schema, Supabase automatically creates:
```
GET  /rest/v1/users         → Get all users
POST /rest/v1/users         → Create user
```

**Private schema** = Tables NOT exposed through the API. Only accessible through direct database connections or Edge Functions (more on those later).

**Why put tables in private schema?**

```
Scenario: Online store

public schema:
  products   ← Customers need to see this
  reviews    ← Customers need to see this
  orders     ← Customer needs to see their OWN orders

private schema:
  profit_margins   ← NEVER expose to public!
  supplier_costs   ← Sensitive business data
  admin_actions    ← Audit trail
```

---

## 3. Row Level Security (RLS) - Supabase

### 3.1 The Problem RLS Solves

Supabase auto-generates APIs for your tables. Without RLS:

```
Any user can access ANY data:

User Alice makes request:
GET /rest/v1/posts
→ Returns ALL posts from ALL users ✗

User Alice makes request:
GET /rest/v1/posts?user_id=eq.bob_id
→ Returns ALL of Bob's posts ✗

Attacker makes request:
GET /rest/v1/users
→ Returns ALL user data ✗
```

### 3.2 What RLS Does

RLS (Row Level Security) adds a rule to every database query:

```
WITHOUT RLS:
Query: SELECT * FROM posts
Result: All 10,000 posts from all users

WITH RLS (policy: "Users can only see their own posts"):
Query: SELECT * FROM posts
Supabase adds automatically: WHERE user_id = 'current_user_id'
Result: Only Alice's 47 posts
```

**The key insight**: The filtering happens at the database level, not in your code. Even if someone makes a direct API call, they can't bypass it.

### 3.3 Policies (How RLS Rules Are Defined)

A **policy** is a rule written in SQL that answers:
- **Who** can perform this action?
- **Which rows** can they access?

```
Policy structure:
┌────────────────────────────────────────────────────────┐
│  Table: posts                                          │
│  Action: SELECT (read)                                 │
│  Rule: Only rows where post.user_id = logged-in user   │
└────────────────────────────────────────────────────────┘

In practice, this translates to:
"When someone queries the posts table,
 automatically filter to only show rows
 where user_id matches the authenticated user's ID"
```

**Types of policies** (for different operations):

```
┌──────────────────────────────────────────────────────────┐
│  posts table policies                                    │
│                                                          │
│  SELECT policy:                                          │
│    Who can read: Any authenticated user                  │
│    Which rows: Only their own posts                      │
│                                                          │
│  INSERT policy:                                          │
│    Who can create: Any authenticated user                │
│    Condition: user_id must equal their own ID            │
│    (prevents creating posts as someone else)             │
│                                                          │
│  UPDATE policy:                                          │
│    Who can edit: Authenticated users                     │
│    Which rows: Only their own posts                      │
│                                                          │
│  DELETE policy:                                          │
│    Who can delete: Authenticated users                   │
│    Which rows: Only their own posts                      │
└──────────────────────────────────────────────────────────┘
```

**Real world flow**:

```
Alice (authenticated) requests her posts:
┌──────────────────────────────────────────────────────────┐
│ 1. Alice makes API request                               │
│    GET /rest/v1/posts                                    │
│    Header: Authorization: Bearer alice_jwt_token         │
│                                                          │
│ 2. Supabase verifies JWT                                 │
│    Extracts: auth.uid() = "alice_user_id"                │
│                                                          │
│ 3. Database query runs with policy applied               │
│    Original: SELECT * FROM posts                         │
│    Modified:  SELECT * FROM posts                        │
│               WHERE user_id = 'alice_user_id'            │
│                                                          │
│ 4. Returns only Alice's posts                            │
└──────────────────────────────────────────────────────────┘

Bob (or anonymous attacker) tries same request:
┌──────────────────────────────────────────────────────────┐
│ GET /rest/v1/posts                                       │
│ Modified: SELECT * FROM posts                            │
│           WHERE user_id = 'bob_user_id'                  │
│                                                          │
│ Returns only Bob's posts (not Alice's) ✓                 │
└──────────────────────────────────────────────────────────┘
```

### 3.4 Firebase Equivalent: Security Rules

Firebase achieves the same goal differently. Instead of database-level SQL policies, Firebase uses a rules language.

```
Firebase Security Rules work like this:

firestore/
└── users/
    └── {userId}/          ← Any user document
        └── posts/
            └── {postId}/  ← Any post document

Rule: "A user can only read/write their OWN documents"
match /users/{userId}/{document=**} {
  allow read, write: if request.auth.uid == userId;
                                          ↑
                         Must match the document path
}
```

**The difference in mental model**:

```
Supabase RLS:
  "Add a filter to every SQL query based on who's asking"

Firebase Rules:
  "Describe who can access which paths in the document tree"
```

---

## 4. Authentication

### 4.1 What BaaS Authentication Gives You

**Without BaaS**: You write code for user registration, login, JWT generation, token refresh, password reset, OAuth flows, email verification...

**With BaaS**: All of this exists as a service. You call their SDK.

### 4.2 How Auth Works Under the Hood

Both Supabase and Firebase follow the same fundamental flow:

```
Registration Flow:
┌──────────────────────────────────────────────────────────┐
│ 1. User submits email + password                         │
│                                                          │
│ 2. BaaS platform:                                        │
│    a. Validates email format                             │
│    b. Hashes password (bcrypt)                           │
│    c. Creates user record in auth.users table            │
│    d. Sends verification email (if configured)           │
│    e. Returns user object + JWT token                    │
│                                                          │
│ 3. User is now registered                                │
└──────────────────────────────────────────────────────────┘

Login Flow:
┌──────────────────────────────────────────────────────────┐
│ 1. User submits email + password                         │
│                                                          │
│ 2. BaaS platform:                                        │
│    a. Finds user by email                                │
│    b. Compares hashed password                           │
│    c. Generates access token (short-lived, ~1 hour)      │
│    d. Generates refresh token (long-lived, ~weeks)       │
│    e. Returns both tokens to client                      │
│                                                          │
│ 3. Client stores tokens                                  │
│    Access token: used in every request header            │
│    Refresh token: used to get new access token           │
└──────────────────────────────────────────────────────────┘
```

### 4.3 Auth Providers

BaaS platforms support multiple ways to authenticate:

```
┌──────────────────────────────────────────────────────────┐
│  Authentication Methods                                  │
│                                                          │
│  Email/Password                                          │
│  ┌──────────────────────────────────────────────────┐    │
│  │ User: "alice@email.com" + "password123"          │    │
│  │ Platform generates JWT for Alice                 │    │
│  └──────────────────────────────────────────────────┘    │
│                                                          │
│  OAuth (Social Login)                                    │
│  ┌──────────────────────────────────────────────────┐    │
│  │ User: "Continue with Google"                     │    │
│  │ Google: "Yes, this is alice@gmail.com"           │    │
│  │ Platform: Creates/finds user, generates JWT      │    │
│  └──────────────────────────────────────────────────┘    │
│                                                          │
│  Phone/OTP                                               │
│  ┌──────────────────────────────────────────────────┐    │
│  │ User: "+1234567890"                              │    │
│  │ Platform: Sends SMS code                         │    │
│  │ User: Enters code → JWT generated                │    │
│  └──────────────────────────────────────────────────┘    │
│                                                          │
│  Magic Link                                              │
│  ┌──────────────────────────────────────────────────┐    │
│  │ User: "alice@email.com"                          │    │
│  │ Platform: Sends email with one-click login link  │    │
│  │ User: Clicks link → JWT generated                │    │
│  └──────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────┘
```

### 4.4 The Token Flow

Understanding tokens is critical:

```
After login, user has two tokens:

Access Token (JWT)
┌──────────────────────────────────────────────────────────┐
│ Short-lived: expires in 1 hour                           │
│ Contains: user ID, email, role                           │
│ Used in: every API request header                        │
│                                                          │
│ GET /api/posts                                           │
│ Authorization: Bearer eyJhbGci...access_token...         │
└──────────────────────────────────────────────────────────┘

Refresh Token
┌──────────────────────────────────────────────────────────┐
│ Long-lived: expires in days/weeks                        │
│ Contains: just enough to identify the session            │
│ Used for: getting a new access token when it expires     │
│                                                          │
│ Access token expired?                                    │
│   → Send refresh token to BaaS                           │
│   → BaaS returns new access token                        │
│   → Continue making requests                             │
└──────────────────────────────────────────────────────────┘

Timeline:
                                                        
  Login ──────────────────── 1hr ──────────────────── days
    │                          │                         │
    ▼                          ▼                         ▼
  Get tokens            Access token             Refresh token
  Access + Refresh       expires                  expires
                            │                    (user must login again)
                            ▼
                     Use refresh token
                     Get new access token
                     Continue using app
```

---

## 5. Real-Time Subscriptions

### 5.1 What Are Subscriptions?

**The problem**: Your app shows a chat list. User A sends a message. User B's screen doesn't update unless they refresh.

**The solution**: Subscribe to database changes. When data changes, your app updates automatically.

### 5.2 How It Works

```
Traditional approach (polling):
┌──────────────────────────────────────────────────────────┐
│ Client asks every 3 seconds:                             │
│ "Any new messages?" → No                                 │
│ "Any new messages?" → No                                 │
│ "Any new messages?" → Yes! Returns new message           │
│                                                          │
│ Problem: Slow, wasteful, 3-second delay                  │
└──────────────────────────────────────────────────────────┘

BaaS Real-Time (subscriptions):
┌──────────────────────────────────────────────────────────┐
│ Client connects once:                                    │
│ "Tell me whenever messages table changes"                │
│                                                          │
│ BaaS platform maintains open connection (WebSocket)      │
│                                                          │
│ When data changes in database:                           │
│   Database → BaaS → Client (instant push)                │
│                                                          │
│ No polling. No delay. No wasted requests.                │
└──────────────────────────────────────────────────────────┘
```

### 5.3 What You Can Subscribe To

**Supabase** (PostgreSQL-based):

```
Supabase listens to PostgreSQL's WAL
(Write-Ahead Log = database's event log)

Every database change is logged:
  INSERT → "Row added to posts: { id: 5, title: 'Hello' }"
  UPDATE → "Row updated in posts: { id: 5, title: 'Hello World' }"
  DELETE → "Row deleted from posts: { id: 5 }"

You can subscribe to:
  ┌────────────────────────────────────────────────────┐
  │ All changes to "messages" table                    │
  │ Only INSERTs to "messages" table                   │
  │ Only changes where room_id = "room123"             │
  │ Only changes on rows you have RLS permission for   │
  └────────────────────────────────────────────────────┘
```

**Firebase** (Firestore-based):

```
Firestore maintains persistent connections to all listeners

You can subscribe to:
  ┌────────────────────────────────────────────────────┐
  │ A single document: /users/alice123                 │
  │ An entire collection: /messages                    │
  │ A filtered collection: /messages where room = X    │
  └────────────────────────────────────────────────────┘
```

### 5.4 Data Flow for Real-Time

```
Chat App Example:
                                                          
User A (sender)       BaaS Platform           User B (listener)
     │                     │                         │
     │                     │   Connected, listening  │
     │                     │   for messages table    │
     │                     │◄────────────────────────┤
     │                     │                         │
     │ Send message        │                         │
     │ POST /messages      │                         │
     ├────────────────────►│                         │
     │                     │                         │
     │                     │ 1. Saved to database    │
     │                     │ 2. WAL event triggered  │
     │                     │ 3. BaaS detects change  │
     │                     │ 4. Push to subscribers  │
     │                     ├───────────────────────► │
     │                     │                         │ Receives message
     │                     │                         │ instantly
     │                     │                         │ UI updates
```

---

## 6. Storage

### 6.1 What BaaS Storage Is

Every app needs to store files: profile pictures, documents, videos. BaaS platforms include a file storage service built in.

```
BaaS Storage architecture:
┌──────────────────────────────────────────────────────────┐
│  Storage Service                                         │
│                                                          │
│  Buckets (top-level containers)                          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐    │
│  │  avatars     │  │  documents   │  │   videos     │    │
│  │  (public)    │  │  (private)   │  │   (private)  │    │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘    │
│         │                 │                  │           │
│   alice.jpg          report.pdf         tutorial.mp4     │
│   bob.jpg            invoice.pdf         demo.mp4        │
│   charlie.jpg                                            │
└──────────────────────────────────────────────────────────┘
```

### 6.2 Public vs Private Buckets

**Public bucket**:

```
Avatars bucket is public
URL: https://project.supabase.co/storage/v1/object/public/avatars/alice.jpg

Anyone with this URL can view it ✓
No authentication needed ✓
Great for: profile pictures, public media
```

**Private bucket**:

```
Documents bucket is private
URL: https://project.supabase.co/storage/v1/object/documents/report.pdf

Direct access: ✗ Returns 403 Forbidden

To access, user must:
  1. Be authenticated
  2. Have permission (storage policies)
  3. Get a signed URL (temporary link):
     https://...supabase.co/.../documents/report.pdf?token=xyz&expires=1699564800
                                                        ↑
                                                 Expires after set time
```

### 6.3 Storage Policies

Just like RLS policies for database tables, storage has policies for files.

```
Example policies for "documents" bucket:

Read policy:
  "Users can only download their own files"
  Condition: filename must start with their user_id

Upload policy:
  "Authenticated users can upload to their folder"
  Condition: file path must be /user_id/filename

Delete policy:
  "Users can only delete their own files"
```

```
Alice tries to download Bob's file:
  Request: GET /storage/documents/bob_id/report.pdf
  Auth: Alice's token
  Policy check: Does alice_id == bob_id? NO
  Result: 403 Forbidden ✗

Alice downloads her own file:
  Request: GET /storage/documents/alice_id/report.pdf
  Auth: Alice's token
  Policy check: Does alice_id == alice_id? YES
  Result: 200 OK + file ✓
```

---

## 7. Edge Functions

### 7.1 What Are Edge Functions?

Edge functions are server-side code that you write and deploy on the BaaS platform itself. They run close to your users geographically (hence "edge").

**Why do you need them if BaaS already does so much?**

```
BaaS can't handle everything automatically:

✓ BaaS handles: CRUD operations, auth, storage
✗ BaaS can't handle:
  - Custom business logic
    "When order is placed, calculate shipping cost based
     on weight, destination, and current carrier prices"
  - Calling third-party APIs
    "After signup, add user to Mailchimp mailing list"
  - Complex data transformations
    "Generate PDF invoice from order data"
  - Scheduled tasks
    "Every night, send digest email to all users"
```

### 7.2 Edge Functions vs Traditional Backend

```
Traditional backend (your server):
┌──────────────────────────────────────────────────────────┐
│  User's Device → Your Server (New York) → Response       │
│                       ↑                                  │
│               All requests go here                       │
│               User in Tokyo: 200ms latency               │
│               User in London: 80ms latency               │
└──────────────────────────────────────────────────────────┘

Edge Functions:
┌──────────────────────────────────────────────────────────┐
│  User in Tokyo  → Tokyo Edge Node → Response (5ms)       │
│  User in London → London Edge Node → Response (3ms)      │
│  User in NYC    → NYC Edge Node → Response (2ms)         │
│                                                          │
│  Your code runs near the user, not on one central server │
└──────────────────────────────────────────────────────────┘
```

### 7.3 When to Use Edge Functions

```
Supabase Edge Functions (uses Deno runtime):
Firebase Cloud Functions (uses Node.js runtime):

Common use cases:

┌──────────────────────────────────────────────────────────┐
│  Triggered by HTTP request:                              │
│    POST /functions/v1/send-email                         │
│    → Your custom code runs                               │
│    → Sends email via SendGrid                            │
│    → Returns result                                      │
│                                                          │
│  Triggered by database event:                            │
│    New row inserted in "orders" table                    │
│    → Edge function triggers automatically                │
│    → Sends order confirmation email                      │
│    → Updates inventory                                   │
│    → Notifies shipping service                           │
│                                                          │
│  Triggered by auth event:                                │
│    New user signs up                                     │
│    → Edge function triggers                              │
│    → Creates default profile record                      │
│    → Sends welcome email                                 │
│                                                          │
│  Scheduled:                                              │
│    Every day at midnight                                 │
│    → Cleans up expired sessions                          │
│    → Sends daily digest emails                           │
└──────────────────────────────────────────────────────────┘
```

---

## 8. Webhooks

### 8.1 What Are Webhooks in BaaS?

**Webhooks** are the BaaS platform notifying YOUR external server when something happens.

```
Without webhooks (you poll BaaS):
Your server: "Did anything happen?" → No
Your server: "Did anything happen?" → No
Your server: "Did anything happen?" → Yes, user signed up

With webhooks (BaaS calls you):
Event happens on BaaS
→ BaaS sends HTTP POST to your server immediately
→ Your server handles it
```

### 8.2 Flow

```
┌──────────────────────────────────────────────────────────┐
│ Example: New user signs up on Supabase                   │
│                                                          │
│ 1. User registers via Supabase Auth                      │
│                                                          │
│ 2. Supabase fires webhook                                │
│    POST https://yourserver.com/webhooks/new-user         │
│    {                                                     │
│      "event": "INSERT",                                  │
│      "table": "users",                                   │
│      "record": {                                         │
│        "id": "user123",                                  │
│        "email": "alice@example.com",                     │
│        "created_at": "2024-01-15T10:30:00Z"              │
│      }                                                   │
│    }                                                     │
│                                                          │
│ 3. Your server receives POST request                     │
│    → Adds user to Mailchimp                              │
│    → Sends welcome gift                                  │
│    → Notifies your team on Slack                         │
└──────────────────────────────────────────────────────────┘
```

### 8.3 Webhooks vs Subscriptions vs Edge Functions

These three all respond to events, but differently:

```
┌──────────────────────────────────────────────────────────────────┐
│                  Responding to Events                            │
├──────────────────────┬───────────────────────────────────────────┤
│  Subscriptions       │ Client (browser/app) receives real-time   │
│                      │ data changes                              │
│                      │ "Tell my React app when a new message     │
│                      │  arrives so I can update the UI"          │
├──────────────────────┼───────────────────────────────────────────┤
│  Edge Functions      │ Server-side code on BaaS platform runs    │
│                      │ "When a new order is created, run my      │
│                      │  payment processing code"                 │
├──────────────────────┼───────────────────────────────────────────┤
│  Webhooks            │ BaaS calls YOUR external server           │
│                      │ "When anything happens, notify my         │
│                      │  separate backend server"                 │
└──────────────────────┴───────────────────────────────────────────┘
```

---

## 9. GIS (Geographic Information System)

### 9.1 What is GIS in BaaS?

**GIS** = working with geographic data (coordinates, maps, distances, regions).

This is a Supabase/PostgreSQL feature. Since Supabase uses PostgreSQL, it has access to PostGIS — a powerful geographic extension.

### 9.2 Why You Need It

**Without GIS**: You fetch all restaurants from database, calculate distance to user in your code for every single restaurant. Slow. Doesn't scale.

**With GIS**: Database does the geographic calculation. Returns only restaurants within 5km, sorted by distance.

### 9.3 What You Can Do

```
Types of geographic data:
┌──────────────────────────────────────────────────────────┐
│                                                          │
│  Point: A single location                                │
│  { latitude: 28.6139, longitude: 77.2090 }               │
│  Example: User's location, restaurant, delivery address  │
│                                                          │
│  LineString: A path/route                                │
│  [ (0,0) → (1,1) → (2,0) ]                               │
│  Example: Delivery route, road                           │
│                                                          │
│  Polygon: An area/region                                 │
│  [ (0,0) → (1,0) → (1,1) → (0,1) → (0,0) ]               │
│  Example: Delivery zone, neighbourhood boundary          │
└──────────────────────────────────────────────────────────┘

Queries you can run:
┌──────────────────────────────────────────────────────────┐
│                                                          │
│  "Find all restaurants within 5km of user"               │
│  "Is this delivery address inside our service zone?"     │
│  "Sort listings by distance from search location"        │
│  "Count users in each city"                              │
└──────────────────────────────────────────────────────────┘
```

---

## 10. The Full BaaS Architecture Picture

### 10.1 Putting It All Together

```
A complete app using BaaS:

YOUR APP (React, Vue, Mobile, etc.)
┌─────────────────────────────────────────────────────────┐
│                                                         │
│  Login with Google ──────────────────────────────────┐  │
│  Show my feed ───────────────────────────────────┐   │  │
│  Upload profile picture ─────────────────────┐   │   │  │
│  Live chat messages ─────────────────────┐   │   │   │  │
│                                          │   │   │   │  │
└──────────────────────────────────────────┼───┼───┼───┼──┘
                                           │   │   │   │
                                    ┌──────▼───▼───▼───▼───────┐
                                    │   BaaS Platform          │
                                    │   (Supabase/Firebase)    │
                                    │                          │
                                    │  ┌────────────────────┐  │
                                    │  │   Auth Service     │  │
                                    │  │  OAuth, JWT, etc.  │◄─┤ Login
                                    │  └────────────────────┘  │
                                    │                          │
                                    │  ┌────────────────────┐  │
                                    │  │   Database         │  │
                                    │  │  + RLS Policies    │◄─┤ Feed
                                    │  └────────────────────┘  │
                                    │                          │
                                    │  ┌────────────────────┐  │
                                    │  │   Storage          │  │
                                    │  │  + Storage Rules   │◄─┤ Upload
                                    │  └────────────────────┘  │
                                    │                          │
                                    │  ┌────────────────────┐  │
                                    │  │   Real-Time        │  │
                                    │  │  Subscriptions     │◄─┤ Chat
                                    │  └────────────────────┘  │
                                    │                          │
                                    └──────────────────────────┘
                                               │
                          ┌────────────────────┴─────────────────────┐
                          │          When things happen              │
                          │                                          │
                          ▼                                          ▼
               Edge Functions                               Webhooks
         (Code on BaaS platform)                    (Call your server)
         "Process payment"                          "User signed up"
         "Send email"                               "Notify your CRM"
         "Resize image"                             "Update your DB"
```

### 10.2 Supabase vs Firebase At a Glance

| Feature | Supabase | Firebase |
|---|---|---|
| **Database** | PostgreSQL (SQL) | Firestore (NoSQL) |
| **Data Structure** | Tables + Relations | Collections + Documents |
| **Security** | RLS + Policies | Security Rules |
| **Schema** | Schemas (public/priv.) | No schema concept |
| **Edge Functions** | Deno Edge Functions | Node.js Cloud Functions |
| **Real-time** | WAL-based real-time | Native real-time (Firestore) |
| **GIS Support** | PostGIS (GIS support) | No built-in GIS |
| **Query Language** | SQL queries | NoSQL queries (limited) |
| **Licensing** | Open source | Proprietary (Google) |

---

## 11. When NOT to Use BaaS

BaaS is powerful, but not always the right choice:

```
Use BaaS when:
  ✓ Building an MVP quickly
  ✓ Small-to-medium scale app
  ✓ Standard CRUD + auth + storage needs
  ✓ Small team or solo developer
  ✓ Want to avoid managing infrastructure

Don't use BaaS when:
  ✗ Complex business logic that doesn't fit their model
  ✗ Need full control over database configuration
  ✗ Extremely high traffic (costs scale rapidly)
  ✗ Data regulations require specific server locations
  ✗ Need custom database extensions BaaS doesn't support
  ✗ Vendor lock-in is a concern
    (migrating away from Firebase later is painful)
```

---

## Summary

**Every confusing BaaS term, explained simply**:

| Term | What It Actually Means |
|------|----------------------|
| **Schema** | A folder that groups database tables. Public = exposed via API. Private = not. |
| **RLS** | A rule that automatically filters database rows based on who's asking |
| **Policy** | The actual rule definition for RLS or Storage security |
| **Subscriptions** | Keep a live connection open so client gets data changes instantly |
| **Edge Functions** | Your custom server-side code, hosted and run on the BaaS platform |
| **Webhooks** | BaaS calls your external server when an event happens |
| **Buckets** | Top-level folders for file storage (like S3 buckets) |
| **GIS** | Geographic queries - distances, areas, coordinates |
| **Auth Providers** | The different ways users can log in (email, Google, phone) |
| **Access Token** | Short-lived JWT, used in every request |
| **Refresh Token** | Long-lived token, used to get a new access token |
| **Security Rules** | Firebase's version of RLS policies |
| **WAL** | PostgreSQL's event log that Supabase uses for real-time |

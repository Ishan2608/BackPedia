# Firebase: A Complete Developer Guide

## What is Firebase and Why Does It Exist

When you build an application, there are two worlds you are managing simultaneously. The first is what users see and interact with — buttons, forms, dashboards. The second is everything that makes those interactions meaningful — storing data, authenticating identities, sending notifications, tracking errors. Traditionally, building that second world required you to provision servers, write APIs, manage databases, handle security rules, and deploy infrastructure. This is called backend development, and it is a discipline in its own right.

Firebase is a Backend-as-a-Service (BaaS) platform built by Google. Its premise is straightforward: instead of building and maintaining your own backend infrastructure, you integrate a set of Google-managed services directly into your frontend application. Firebase handles the servers, the scaling, the uptime, and the security infrastructure. You focus on your product.

This guide will walk you through everything Firebase offers, how each service works conceptually and technically, and how to integrate it into a real application from scratch.

---

## Core Concepts Before You Begin

### Projects and Apps

Everything in Firebase lives inside a **Project**. Think of a project as a container for a single product. Inside that project, you register **Apps** — one for your web frontend, one for your iOS app, one for Android, and so on. Each registered app gets a configuration object that tells the Firebase SDK which project it belongs to and how to reach the right services.

### The Firebase SDK

Firebase provides a JavaScript SDK (and SDKs for Swift, Kotlin, Flutter, etc.) that you install in your application. When you call `auth.signIn()` or `db.collection('users').get()`, you are calling methods in this SDK. The SDK handles all the network communication with Google's servers on your behalf. You never write raw HTTP requests to Google APIs — the SDK abstracts all of that.

### Security Rules

Firebase services like Firestore and Storage use a declarative rules language to determine who can read or write what data. These rules live on Google's servers and are evaluated on every request. They are the most critical part of a Firebase setup — without proper rules, your data is either wide open or completely inaccessible. This guide covers them in depth.

---

## Setting Up Firebase

### Step 1: Create a Firebase Project

Go to the [Firebase Console](https://console.firebase.google.com) and click **Add project**. You will be prompted to name your project, and optionally connect it to a Google Analytics account. Analytics integration is recommended even for small projects because it enables certain features in Firebase and costs nothing.

Once created, you land on the project dashboard. Everything you configure from this point forward belongs to this project.

### Step 2: Register Your Web App

On the project overview page, click the web icon (`</>`) to register a new web application. Give it a nickname (this is for your own reference, not user-facing). After registering, Firebase shows you a **configuration object** that looks like this:

```javascript
const firebaseConfig = {
  apiKey: "AIzaSyXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX",
  authDomain: "your-project-id.firebaseapp.com",
  projectId: "your-project-id",
  storageBucket: "your-project-id.appspot.com",
  messagingSenderId: "123456789012",
  appId: "1:123456789012:web:abcdef1234567890"
};
```

This is not a secret in the traditional sense. The `apiKey` here is not a server-side secret — it is a public identifier that tells Firebase which project to communicate with. Access control is enforced through Security Rules and authentication, not by keeping this config hidden. That said, you should still store it in environment variables to prevent accidental leakage into public repositories.

### Step 3: Install the Firebase SDK

In your project directory, initialize a Node project if you have not already, then install Firebase:

```bash
npm init -y
npm install firebase
```

Firebase 9 and above uses a **modular SDK** (also called the tree-shakeable SDK). This means you import only the functions you need, which significantly reduces your bundle size. This guide uses the modular API throughout.

### Step 4: Initialize Firebase in Your Application

Create a file called `firebase.js` (or `firebase.ts` if using TypeScript) at the root of your source directory. This file initializes Firebase and exports the services you will use throughout the app:

```javascript
// src/firebase.js
import { initializeApp } from 'firebase/app';
import { getAuth } from 'firebase/auth';
import { getFirestore } from 'firebase/firestore';
import { getStorage } from 'firebase/storage';

const firebaseConfig = {
  apiKey: process.env.VITE_FIREBASE_API_KEY,
  authDomain: process.env.VITE_FIREBASE_AUTH_DOMAIN,
  projectId: process.env.VITE_FIREBASE_PROJECT_ID,
  storageBucket: process.env.VITE_FIREBASE_STORAGE_BUCKET,
  messagingSenderId: process.env.VITE_FIREBASE_MESSAGING_SENDER_ID,
  appId: process.env.VITE_FIREBASE_APP_ID
};

// Initialize the Firebase application
const app = initializeApp(firebaseConfig);

// Export individual service instances
export const auth = getAuth(app);
export const db = getFirestore(app);
export const storage = getStorage(app);
```

The prefix `VITE_` is specific to Vite-based projects (like React + Vite). If you are using Create React App, the prefix is `REACT_APP_`. If you are using Next.js, the prefix is `NEXT_PUBLIC_`. Store the actual values in a `.env` file at the root of your project and add `.env` to your `.gitignore`.

### Step 5: Install the Firebase CLI

The Firebase Command Line Interface (CLI) is a tool for managing your project from the terminal. You will use it to deploy security rules, deploy Firestore indexes, run local emulators, and deploy hosting.

```bash
npm install -g firebase-tools
```

Log in to your Firebase account:

```bash
firebase login
```

This opens a browser window for Google OAuth authentication. Once authenticated, initialize Firebase in your project directory:

```bash
firebase init
```

The CLI walks you through a series of prompts. You can select which Firebase features to configure: Firestore, Hosting, Storage, etc. For each one, it creates the relevant configuration files in your project. The two most important files it creates are `firestore.rules` and `firestore.indexes.json`.

---

## Firebase Authentication

### The Problem It Solves

User authentication is one of the hardest parts of backend development to get right. Storing passwords securely, handling session management, implementing OAuth with providers like Google or GitHub, preventing brute-force attacks, supporting password resets, managing refresh tokens — every one of these is a solved problem that you are nevertheless responsible for implementing correctly if you build it yourself. One mistake means user accounts get compromised.

Firebase Authentication provides all of this as a service. You call SDK functions, Firebase handles everything else.

### Enabling Sign-In Methods

Before users can sign in, you must enable the sign-in methods you want to support. Go to the Firebase Console, navigate to **Authentication**, then **Sign-in method**. Enable whichever providers you need. For email/password, it is a simple toggle. For Google Sign-In, you need to configure an OAuth consent screen in your Google Cloud project (Firebase walks you through this). For GitHub, you register a new OAuth application in your GitHub developer settings and paste the client ID and secret into Firebase.

### Email and Password Authentication

```javascript
import { auth } from './firebase';
import {
  createUserWithEmailAndPassword,
  signInWithEmailAndPassword,
  signOut,
  onAuthStateChanged
} from 'firebase/auth';

// Register a new user
async function registerUser(email, password) {
  try {
    const userCredential = await createUserWithEmailAndPassword(auth, email, password);
    // userCredential.user contains the authenticated user object
    const user = userCredential.user;
    console.log('Registered user:', user.uid);
    return user;
  } catch (error) {
    // Firebase returns structured error codes
    // error.code will be something like 'auth/email-already-in-use'
    console.error('Registration failed:', error.code, error.message);
    throw error;
  }
}

// Sign in an existing user
async function loginUser(email, password) {
  try {
    const userCredential = await signInWithEmailAndPassword(auth, email, password);
    return userCredential.user;
  } catch (error) {
    console.error('Login failed:', error.code, error.message);
    throw error;
  }
}

// Sign out
async function logoutUser() {
  await signOut(auth);
}
```

### Listening for Authentication State Changes

One of Firebase Auth's most useful features is a real-time listener that fires whenever the user's authentication state changes — when they sign in, when they sign out, or when the page loads and Firebase restores a saved session.

```javascript
// This listener fires immediately with the current state, and again on every change
const unsubscribe = onAuthStateChanged(auth, (user) => {
  if (user) {
    // User is signed in
    console.log('User ID:', user.uid);
    console.log('Email:', user.email);
    console.log('Display name:', user.displayName);
  } else {
    // User is signed out
    console.log('No user signed in');
  }
});

// Call unsubscribe() when your component unmounts to stop listening
// This prevents memory leaks in frameworks like React
```

In a React application, you would typically put this listener in a `useEffect` hook at your top-level component or in a custom context provider, updating your global auth state based on what it reports.

### Google Sign-In (OAuth)

```javascript
import { GoogleAuthProvider, signInWithPopup } from 'firebase/auth';
import { auth } from './firebase';

async function signInWithGoogle() {
  const provider = new GoogleAuthProvider();

  // Optionally request additional OAuth scopes
  // provider.addScope('https://www.googleapis.com/auth/contacts.readonly');

  try {
    const result = await signInWithPopup(auth, provider);
    const user = result.user;

    // You can also access the Google OAuth credential if needed
    const credential = GoogleAuthProvider.credentialFromResult(result);
    const token = credential.accessToken;

    return user;
  } catch (error) {
    console.error('Google sign-in failed:', error.code, error.message);
    throw error;
  }
}
```

For mobile-friendly applications, replace `signInWithPopup` with `signInWithRedirect`. The redirect approach navigates the user to Google's login page, then returns them to your app after authentication. It works better on devices where popups may be blocked.

### The User Object

Every authenticated user has a `user` object with the following key properties:

```
user.uid          — A unique string ID. This is what you use to identify users in your database.
user.email        — The user's email address.
user.displayName  — Full name (populated by OAuth providers like Google; null for email/password).
user.photoURL     — Profile photo URL (populated by OAuth providers; null for email/password).
user.emailVerified — Boolean indicating if the email address has been verified.
```

The `uid` property is the most important. It is unique across all of Firebase for your project and never changes. Use it as the primary key when storing user-specific data in Firestore or any other database.

### Sending a Password Reset Email

```javascript
import { sendPasswordResetEmail } from 'firebase/auth';
import { auth } from './firebase';

async function resetPassword(email) {
  await sendPasswordResetEmail(auth, email);
  // Firebase sends an email to the user with a reset link
  // This function resolves even if the email does not exist, for security reasons
}
```

---

## Cloud Firestore

### The Problem It Solves

Every application needs to store and retrieve data. You need a database. Traditionally this means provisioning a database server, writing an API layer in Node.js or Python that talks to that database, and securing both. Firestore is a NoSQL document database that you can query directly from the client, without writing any API layer. Security is handled entirely by the rules you configure.

### How Firestore Organizes Data

Firestore is a **document database**, not a relational database like PostgreSQL or MySQL. Understanding its data model is the most important prerequisite to using it effectively.

Data is organized in a hierarchy of **Collections** and **Documents**.

A **Document** is a single record. It is similar to a JSON object: a set of key-value pairs where values can be strings, numbers, booleans, arrays, nested maps, timestamps, or references to other documents. Every document has a unique ID within its collection.

A **Collection** is simply a named group of documents. You can think of it as a folder.

Consider a blog application. You might have a `posts` collection, where each document represents one blog post:

```
posts/                          ← Collection
  post_abc123/                  ← Document (auto-generated ID)
    title: "Getting Started with Firebase"
    body: "Firebase is a BaaS platform..."
    authorId: "user_xyz789"
    createdAt: Timestamp(2024-01-15)
    published: true

  post_def456/                  ← Another document
    title: "Understanding Firestore"
    body: "Firestore organizes data into..."
    authorId: "user_abc123"
    createdAt: Timestamp(2024-01-20)
    published: false
```

Documents can also contain **Subcollections**. If each post can have comments, you would organize them like this:

```
posts/post_abc123/comments/     ← Subcollection
  comment_001/
    text: "Great article!"
    authorId: "user_jkl012"
    createdAt: Timestamp(2024-01-16)
```

This hierarchical structure makes data co-location natural and queries efficient.

### Enabling Firestore

In the Firebase Console, go to **Firestore Database** and click **Create database**. You will be asked to choose a mode: **Production mode** or **Test mode**. Choose Production mode and write security rules properly — Test mode leaves the database open to the public for 30 days, which is a security risk you want to avoid even in development. For local development, use the Firebase Emulator instead (covered later in this guide).

You will also be asked to choose a location for your Firestore instance. Choose the region closest to your users. This decision is permanent for a given database instance.

### Reading Data

```javascript
import { db } from './firebase';
import {
  collection,
  doc,
  getDoc,
  getDocs,
  query,
  where,
  orderBy,
  limit,
  onSnapshot
} from 'firebase/firestore';

// --- Reading a single document ---
async function getPost(postId) {
  // doc() creates a reference to a specific document
  // It does not fetch any data yet
  const postRef = doc(db, 'posts', postId);

  // getDoc() actually fetches the document from Firebase
  const postSnap = await getDoc(postRef);

  if (postSnap.exists()) {
    // .data() returns the document's fields as a plain JavaScript object
    // .id returns the document ID
    return { id: postSnap.id, ...postSnap.data() };
  } else {
    throw new Error('Post not found');
  }
}

// --- Reading all documents in a collection ---
async function getAllPosts() {
  const postsRef = collection(db, 'posts');
  const snapshot = await getDocs(postsRef);

  // snapshot.docs is an array of document snapshots
  return snapshot.docs.map(doc => ({
    id: doc.id,
    ...doc.data()
  }));
}

// --- Querying with filters ---
async function getPublishedPosts() {
  const postsRef = collection(db, 'posts');

  // Build a query: filter by 'published == true', order by 'createdAt' descending, limit to 10
  const q = query(
    postsRef,
    where('published', '==', true),
    orderBy('createdAt', 'desc'),
    limit(10)
  );

  const snapshot = await getDocs(q);
  return snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
}
```

### Real-Time Listeners

One of Firestore's defining features is the ability to listen for live data changes. Instead of polling the database on a timer, you subscribe to a query and Firebase pushes updates to you whenever the underlying data changes. This is what enables real-time features like chat applications, live dashboards, and collaborative editing.

```javascript
// Subscribe to real-time updates on a query
function subscribeToPublishedPosts(callback) {
  const postsRef = collection(db, 'posts');
  const q = query(
    postsRef,
    where('published', '==', true),
    orderBy('createdAt', 'desc')
  );

  // onSnapshot calls the callback immediately with current data,
  // then calls it again every time the data changes
  const unsubscribe = onSnapshot(q, (snapshot) => {
    const posts = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
    callback(posts);
  }, (error) => {
    console.error('Listener error:', error);
  });

  // Return the unsubscribe function so the caller can stop listening
  return unsubscribe;
}

// Usage
const stopListening = subscribeToPublishedPosts((posts) => {
  console.log('Posts updated:', posts);
  // Re-render your UI here
});

// When the user navigates away, stop the listener
stopListening();
```

### Writing Data

```javascript
import {
  addDoc,
  setDoc,
  updateDoc,
  deleteDoc,
  serverTimestamp
} from 'firebase/firestore';
import { db } from './firebase';

// --- Adding a document with an auto-generated ID ---
async function createPost(postData) {
  const postsRef = collection(db, 'posts');

  // addDoc adds a new document and Firebase generates the ID
  const docRef = await addDoc(postsRef, {
    ...postData,
    // serverTimestamp() tells Firestore to use the server's clock
    // Always use this instead of new Date() for consistency
    createdAt: serverTimestamp(),
    published: false
  });

  console.log('Created post with ID:', docRef.id);
  return docRef.id;
}

// --- Setting a document with a specific ID ---
// setDoc overwrites the entire document (or creates it if it does not exist)
async function createUserProfile(userId, profileData) {
  const userRef = doc(db, 'users', userId);

  await setDoc(userRef, {
    ...profileData,
    createdAt: serverTimestamp()
  });
}

// Use merge: true to avoid overwriting the entire document
// Only the fields you specify will be updated or added
async function updateUserProfile(userId, updates) {
  const userRef = doc(db, 'users', userId);
  await setDoc(userRef, updates, { merge: true });
}

// --- Updating specific fields in a document ---
// updateDoc only updates the fields you specify; other fields remain unchanged
// It will throw an error if the document does not exist
async function publishPost(postId) {
  const postRef = doc(db, 'posts', postId);
  await updateDoc(postRef, {
    published: true,
    publishedAt: serverTimestamp()
  });
}

// --- Deleting a document ---
async function deletePost(postId) {
  const postRef = doc(db, 'posts', postId);
  await deleteDoc(postRef);
}
```

### Firestore Security Rules

Security rules are the mechanism by which you tell Firestore who can read or write which documents. They are written in a specialized syntax and deployed via the Firebase CLI.

The rules live in your `firestore.rules` file. Here is a complete example for a blog application:

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {

    // Helper function: checks if the requesting user is authenticated
    function isAuthenticated() {
      return request.auth != null;
    }

    // Helper function: checks if the requesting user owns the document
    // resource.data is the existing document data
    // request.auth.uid is the UID of the user making the request
    function isOwner() {
      return request.auth.uid == resource.data.authorId;
    }

    // Rules for the 'posts' collection
    match /posts/{postId} {

      // Anyone (including unauthenticated users) can read published posts
      allow read: if resource.data.published == true;

      // Authenticated users can read their own unpublished drafts
      allow read: if isAuthenticated() && isOwner();

      // Only authenticated users can create posts
      // request.resource.data is the data being written (the new document)
      allow create: if isAuthenticated()
        && request.resource.data.authorId == request.auth.uid
        && request.resource.data.title is string
        && request.resource.data.title.size() > 0;

      // Only the author can update their own post
      allow update: if isAuthenticated() && isOwner();

      // Only the author can delete their own post
      allow delete: if isAuthenticated() && isOwner();

      // Rules for the 'comments' subcollection on each post
      match /comments/{commentId} {

        // Anyone can read comments on published posts
        allow read: if get(/databases/$(database)/documents/posts/$(postId)).data.published == true;

        // Authenticated users can create comments
        allow create: if isAuthenticated()
          && request.resource.data.authorId == request.auth.uid;

        // Only comment authors can update or delete their comments
        allow update, delete: if isAuthenticated()
          && request.auth.uid == resource.data.authorId;
      }
    }

    // Rules for the 'users' collection (user profiles)
    match /users/{userId} {
      // Any authenticated user can read any profile (for showing author names)
      allow read: if isAuthenticated();

      // Users can only write to their own profile document
      allow write: if isAuthenticated() && request.auth.uid == userId;
    }
  }
}
```

Deploy these rules with:

```bash
firebase deploy --only firestore:rules
```

### Firestore Indexes

Firestore requires a composite index for any query that filters or sorts on more than one field. For example, querying posts where `published == true` ordered by `createdAt` requires a composite index on both fields.

When you run a query in development that needs an index you have not created, Firestore throws an error in the console that includes a direct link to create the required index in the Firebase Console. Clicking that link is the easiest way to add indexes during development.

For production, you define indexes in `firestore.indexes.json` and deploy them:

```json
{
  "indexes": [
    {
      "collectionGroup": "posts",
      "queryScope": "COLLECTION",
      "fields": [
        { "fieldPath": "published", "order": "ASCENDING" },
        { "fieldPath": "createdAt", "order": "DESCENDING" }
      ]
    }
  ]
}
```

```bash
firebase deploy --only firestore:indexes
```

---

## Cloud Storage for Firebase

### The Problem It Solves

Applications frequently need to store files: user profile photos, document uploads, media attachments. Cloud Storage for Firebase is Google Cloud Storage with Firebase-native authentication and security rules layered on top. You can upload files directly from the browser or mobile app, secured by the same user authentication you already have.

### Uploading Files

```javascript
import { storage } from './firebase';
import { ref, uploadBytesResumable, getDownloadURL } from 'firebase/storage';
import { auth } from './firebase';

async function uploadProfilePhoto(file) {
  const user = auth.currentUser;
  if (!user) throw new Error('Must be authenticated to upload files');

  // Create a storage reference — a pointer to a location in Storage
  // Organizing files by user ID keeps them separated and makes rules easier
  const storageRef = ref(storage, `profile-photos/${user.uid}/${file.name}`);

  // uploadBytesResumable returns a task that you can monitor for progress
  const uploadTask = uploadBytesResumable(storageRef, file);

  return new Promise((resolve, reject) => {
    uploadTask.on(
      'state_changed',
      (snapshot) => {
        // Calculate upload progress as a percentage
        const progress = (snapshot.bytesTransferred / snapshot.totalBytes) * 100;
        console.log(`Upload is ${progress.toFixed(1)}% done`);
      },
      (error) => {
        // Handle upload errors
        console.error('Upload failed:', error.code);
        reject(error);
      },
      async () => {
        // Upload completed successfully
        // Get the permanent download URL
        const downloadURL = await getDownloadURL(uploadTask.snapshot.ref);
        console.log('File available at:', downloadURL);
        resolve(downloadURL);
      }
    );
  });
}
```

### Storage Security Rules

Storage rules follow the same pattern as Firestore rules. They live in `storage.rules`:

```
rules_version = '2';
service firebase.storage {
  match /b/{bucket}/o {

    // Profile photos: users can only read and write their own photos
    match /profile-photos/{userId}/{allPaths=**} {

      // Anyone can view profile photos (for displaying in the app)
      allow read;

      // Only the authenticated user matching the userId path can write
      allow write: if request.auth != null
        && request.auth.uid == userId
        && request.resource.size < 5 * 1024 * 1024  // Max 5MB
        && request.resource.contentType.matches('image/.*'); // Images only
    }

    // Private documents: only the owner can read and write
    match /documents/{userId}/{allPaths=**} {
      allow read, write: if request.auth != null && request.auth.uid == userId;
    }
  }
}
```

Deploy with:

```bash
firebase deploy --only storage
```

---

## Firebase Hosting

### The Problem It Solves

Firebase Hosting provides fast, HTTPS-enabled static hosting for your web application. It is backed by a global CDN (Content Delivery Network), meaning your files are served from servers close to each user geographically. Deployments are atomic — your site switches to the new version all at once, never serving a mix of old and new files.

### Configuring Hosting

After running `firebase init` and selecting Hosting, the CLI creates a `firebase.json` file. Here is a typical configuration for a single-page application (SPA):

```json
{
  "hosting": {
    "public": "dist",
    "ignore": [
      "firebase.json",
      "**/.*",
      "**/node_modules/**"
    ],
    "rewrites": [
      {
        "source": "**",
        "destination": "/index.html"
      }
    ],
    "headers": [
      {
        "source": "**/*.@(js|css)",
        "headers": [
          {
            "key": "Cache-Control",
            "value": "public, max-age=31536000, immutable"
          }
        ]
      }
    ]
  }
}
```

The `public` field points to your build output directory. The `rewrites` rule sends all requests to `index.html`, which is necessary for client-side routing in frameworks like React Router or Vue Router. Without it, navigating directly to `/about` returns a 404 error.

### Deploying

Build your application first, then deploy:

```bash
# Build (command varies by framework)
npm run build

# Deploy to Firebase Hosting
firebase deploy --only hosting
```

Firebase gives you a URL in the format `https://your-project-id.web.app` where your site is immediately live. You can also connect a custom domain from the Firebase Console under **Hosting**.

To preview a deployment before going live:

```bash
firebase hosting:channel:deploy preview-channel-name
```

This deploys to a temporary URL you can share for review without affecting your live site.

---

## Firebase Cloud Functions

### The Problem It Solves

Firebase client-side SDKs are powerful, but some logic must run on a server. Reasons include: you need to call a third-party API whose secret key must not be exposed to the client; you need to run logic after a database write that the client should not control; you need to validate a payment webhook from Stripe; you need to send a transactional email after a user signs up. Cloud Functions let you write server-side Node.js (or Python, Go, Java) code that Firebase runs and scales automatically — you never provision or manage a server.

### Setting Up Cloud Functions

During `firebase init`, select **Functions**. The CLI creates a `functions/` directory with its own `package.json`. Install the required packages:

```bash
cd functions
npm install firebase-admin firebase-functions
```

### Writing a Cloud Function

The `functions/index.js` file is where you export your functions. Here are three common patterns:

```javascript
const { onCall, onRequest } = require('firebase-functions/v2/https');
const { onDocumentCreated } = require('firebase-functions/v2/firestore');
const { onUserCreated } = require('firebase-functions/v2/identity');
const admin = require('firebase-admin');

// Initialize the Admin SDK — this gives server-side access to all Firebase services
// without being subject to security rules
admin.initializeApp();

// --- HTTPS Callable Function ---
// Called directly from the client SDK with authentication context automatically included
// Use this for operations that require server-side logic but are triggered by user actions
exports.createCheckoutSession = onCall(async (request) => {
  // request.auth contains the authenticated user's info (if signed in)
  if (!request.auth) {
    throw new Error('Must be authenticated');
  }

  const userId = request.auth.uid;
  const { planId } = request.data;

  // Here you would call Stripe's API with your secret key
  // The secret key stays on the server and is never exposed to the client
  const session = await createStripeSession(userId, planId);

  return { sessionId: session.id };
});

// --- Firestore Trigger ---
// Runs automatically when a document is created in the 'posts' collection
// This is pure server-side logic — the client has no control over it
exports.onPostCreated = onDocumentCreated('posts/{postId}', async (event) => {
  const postData = event.data.data();
  const postId = event.params.postId;

  // Example: send a notification to the author's followers
  const authorId = postData.authorId;

  // Use admin.firestore() for server-side database access (bypasses security rules)
  const followersSnap = await admin.firestore()
    .collection('followers')
    .where('followingId', '==', authorId)
    .get();

  const notificationPromises = followersSnap.docs.map(doc => {
    return admin.firestore().collection('notifications').add({
      userId: doc.data().followerId,
      type: 'new_post',
      postId: postId,
      createdAt: admin.firestore.FieldValue.serverTimestamp()
    });
  });

  await Promise.all(notificationPromises);
});

// --- Auth Trigger ---
// Runs when a new user account is created
exports.onNewUser = onUserCreated(async (event) => {
  const user = event.data;

  // Create a profile document for every new user automatically
  await admin.firestore().collection('users').doc(user.uid).set({
    email: user.email,
    displayName: user.displayName || null,
    photoURL: user.photoURL || null,
    createdAt: admin.firestore.FieldValue.serverTimestamp(),
    role: 'user'
  });
});
```

### Calling a Callable Function from the Client

```javascript
import { getFunctions, httpsCallable } from 'firebase/functions';
import { app } from './firebase';

const functions = getFunctions(app);

async function startCheckout(planId) {
  const createCheckoutSession = httpsCallable(functions, 'createCheckoutSession');

  try {
    const result = await createCheckoutSession({ planId });
    const { sessionId } = result.data;
    // Redirect to Stripe Checkout, for example
    return sessionId;
  } catch (error) {
    console.error('Function call failed:', error.message);
    throw error;
  }
}
```

### Deploying Cloud Functions

```bash
firebase deploy --only functions
```

To deploy a single function instead of all of them:

```bash
firebase deploy --only functions:createCheckoutSession
```

---

## The Firebase Local Emulator Suite

### Why Emulators Matter

Running development code against your production Firebase project is dangerous and expensive. You might accidentally delete production data, exceed free-tier quotas, or trigger functions against real users. The Emulator Suite runs full local versions of Firestore, Authentication, Storage, Functions, and Hosting on your machine. Your app connects to these local services instead of Google's servers, giving you a safe, offline-capable, fast development environment.

### Starting the Emulators

After running `firebase init` and selecting **Emulators**, configure which services to emulate. Then start them with:

```bash
firebase emulators:start
```

The CLI outputs the local ports for each service. By default, the Emulator UI is available at `http://localhost:4000`, where you can inspect your Firestore data, view authentication state, and see function logs in real time.

### Connecting Your App to the Emulators

In your `firebase.js` initialization file, add the following code to connect to local emulators during development:

```javascript
import { initializeApp } from 'firebase/app';
import { getAuth, connectAuthEmulator } from 'firebase/auth';
import { getFirestore, connectFirestoreEmulator } from 'firebase/firestore';
import { getStorage, connectStorageEmulator } from 'firebase/storage';
import { getFunctions, connectFunctionsEmulator } from 'firebase/functions';

const app = initializeApp(firebaseConfig);

export const auth = getAuth(app);
export const db = getFirestore(app);
export const storage = getStorage(app);
export const functions = getFunctions(app);

// Connect to emulators only in development
// The 'import.meta.env.DEV' flag is true in Vite during 'npm run dev'
// Use 'process.env.NODE_ENV === "development"' for Create React App
if (import.meta.env.DEV) {
  connectAuthEmulator(auth, 'http://localhost:9099');
  connectFirestoreEmulator(db, 'localhost', 8080);
  connectStorageEmulator(storage, 'localhost', 9199);
  connectFunctionsEmulator(functions, 'localhost', 5001);
}
```

Every action you take while connected to the emulators — creating users, writing documents, uploading files — stays entirely local. When you stop the emulators, that data is cleared. You can persist emulator data between sessions using:

```bash
firebase emulators:start --export-on-exit ./emulator-data --import ./emulator-data
```

---

## Firebase Realtime Database

### How It Differs from Firestore

Firebase offers two database products: Cloud Firestore and the Realtime Database. The Realtime Database is the original Firebase offering. It stores data as a single large JSON tree and syncs changes to connected clients in real time with extremely low latency. Firestore is the more modern product with richer querying, better scalability, and a more structured data model.

For new projects, Cloud Firestore is the recommended choice in almost all cases. The Realtime Database remains useful for specific scenarios where you need the absolute lowest possible latency, such as multiplayer games or applications that require syncing state at very high frequency.

### Basic Usage

```javascript
import { getDatabase, ref, set, get, onValue, push } from 'firebase/database';
import { app } from './firebase';

const rtdb = getDatabase(app);

// Writing data — this overwrites everything at this path
await set(ref(rtdb, 'users/' + userId), {
  name: 'Alex',
  lastSeen: Date.now()
});

// Reading data once
const snapshot = await get(ref(rtdb, 'users/' + userId));
if (snapshot.exists()) {
  console.log(snapshot.val());
}

// Real-time listener — fires immediately and on every change
const unsubscribe = onValue(ref(rtdb, 'users/' + userId), (snapshot) => {
  console.log('User data:', snapshot.val());
});

// Adding to a list with an auto-generated key
const messagesRef = ref(rtdb, 'messages/room123');
await push(messagesRef, {
  text: 'Hello, world!',
  sender: userId,
  timestamp: Date.now()
});
```

---

## Firebase Cloud Messaging

### The Problem It Solves

Firebase Cloud Messaging (FCM) is a cross-platform service for sending push notifications to web browsers, Android, and iOS. You can send messages to individual devices, groups of devices, or topics that users subscribe to.

### Setting Up Web Push Notifications

Web Push requires a VAPID (Voluntary Application Server Identification) key pair. Generate one in the Firebase Console under **Cloud Messaging** settings.

```javascript
import { getMessaging, getToken, onMessage } from 'firebase/messaging';
import { app } from './firebase';

const messaging = getMessaging(app);

// Request permission and get the device token
async function requestNotificationPermission() {
  const permission = await Notification.requestPermission();

  if (permission === 'granted') {
    const token = await getToken(messaging, {
      vapidKey: 'YOUR_VAPID_PUBLIC_KEY_FROM_FIREBASE_CONSOLE'
    });

    console.log('FCM Token:', token);
    // Save this token to Firestore associated with the user
    // You will use it to send targeted notifications from Cloud Functions
    return token;
  } else {
    console.log('Notification permission denied');
    return null;
  }
}

// Listen for foreground messages (when the app is open)
onMessage(messaging, (payload) => {
  console.log('Message received:', payload);
  // Display a custom in-app notification
  const { title, body } = payload.notification;
  // Show your own UI notification here
});
```

To receive background notifications (when the app is not open), you need a Service Worker. Create a file at `public/firebase-messaging-sw.js`:

```javascript
importScripts('https://www.gstatic.com/firebasejs/10.0.0/firebase-app-compat.js');
importScripts('https://www.gstatic.com/firebasejs/10.0.0/firebase-messaging-compat.js');

firebase.initializeApp({
  apiKey: '...',
  authDomain: '...',
  projectId: '...',
  storageBucket: '...',
  messagingSenderId: '...',
  appId: '...'
});

const messaging = firebase.messaging();

messaging.onBackgroundMessage((payload) => {
  const { title, body } = payload.notification;

  self.registration.showNotification(title, {
    body: body,
    icon: '/icon-192x192.png'
  });
});
```

---

## Firebase Performance Monitoring

Firebase Performance Monitoring automatically measures key performance metrics — First Contentful Paint, page load time, network request latency — without any configuration. It also lets you define custom traces to measure specific operations in your app.

```javascript
import { getPerformance, trace } from 'firebase/performance';
import { app } from './firebase';

const perf = getPerformance(app);

// Measure a custom operation
async function fetchUserDashboardData(userId) {
  const t = trace(perf, 'fetch_dashboard_data');
  t.start();

  try {
    const data = await loadDashboard(userId);
    t.stop();
    return data;
  } catch (error) {
    t.stop();
    throw error;
  }
}
```

Performance data appears in the Firebase Console under **Performance**, typically with a 12-hour delay.

---

## Firebase Analytics

Firebase Analytics (powered by Google Analytics 4) gives you insight into how users interact with your application: which pages they visit, which features they use, where they drop off in a funnel, and much more.

```javascript
import { getAnalytics, logEvent } from 'firebase/analytics';
import { app } from './firebase';

const analytics = getAnalytics(app);

// Log a custom event when a user clicks "Publish"
function handlePublishPost(postId) {
  logEvent(analytics, 'post_published', {
    post_id: postId,
    category: 'blog'
  });
}

// Log when a user views a specific screen
function trackPageView(pageName) {
  logEvent(analytics, 'page_view', {
    page_title: pageName
  });
}
```

---

## Firebase Pricing

Firebase operates on a **Spark Plan** (free) and **Blaze Plan** (pay-as-you-go). The Spark Plan has generous free tiers:

For Firestore, you get 1 GB of stored data, 50,000 document reads per day, 20,000 document writes per day, and 20,000 document deletes per day. For Storage, you get 5 GB of stored data and 1 GB of downloads per day. For Authentication, all sign-in methods are free with no document-level limits.

Cloud Functions requires the Blaze Plan, even though it has a free tier (2 million invocations per month free). You need to upgrade to use Functions at all.

The Blaze Plan charges you only for what you use beyond the free tier. For most early-stage applications, the bill is zero or negligible.

---

## Deployment Checklist and Environment Separation

A production-ready Firebase setup separates your development and production environments into two distinct Firebase projects. This prevents development activity from polluting production data and ensures your security rules are tested before going live.

Create two Firebase projects in the console: one named `my-app-dev` and one named `my-app-prod`. In your local project, you manage multiple environments using `.env` files:

```
.env.development    ← Loaded during 'npm run dev'
.env.production     ← Loaded during 'npm run build'
```

Each file contains the respective project's Firebase config values. When you run `firebase deploy`, the CLI deploys to whichever project is currently selected. You can switch between them using:

```bash
firebase use my-app-dev
firebase use my-app-prod
```

Or add aliases in `.firebaserc`:

```json
{
  "projects": {
    "default": "my-app-dev",
    "production": "my-app-prod"
  }
}
```

Then switch with:

```bash
firebase use production
firebase deploy
```

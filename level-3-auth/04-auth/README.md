`Level 3` **Step 4 of 8** — Authentication

# 04 — Authentication: Registration, Login, and JWT

## Spatial Orientation

```
vault-note/
├── client/              ← NOT working here right now
└── server/
    └── src/
        ├── db/                   ← Already built ✓
        ├── types/
        │   └── index.ts          ← Type definitions (we'll build this)
        ├── middleware/
        │   ├── authenticate.ts   ← ★ JWT verification (we'll build this) ★
        │   └── errorHandler.ts   ← Error handler (we'll build this)
        └── routes/
            └── auth.ts           ← ★ Register + Login (we'll build this) ★
```

**What are we building?** The authentication layer. This is the code that:
1. Registers users (hash password, store in database)
2. Logs users in (verify password, create JWT)
3. Protects routes (verify JWT on every request)

This is the most important lesson in Level 3. Everything else builds on it.

---

## Step 1: Define Types

**Where?** `server/src/types/index.ts`

In VS Code, create `server/src/types/index.ts`:

```typescript
// ─── DATABASE MODELS ────────────────────────────────────────────

export interface User {
  id: number;
  email: string;
  password_hash: string;
  role: string;
  created_at: string;
}

// User data that's safe to send to the frontend (no password_hash)
export interface SafeUser {
  id: number;
  email: string;
  role: string;
  created_at: string;
}

export interface Note {
  id: number;
  user_id: number;
  title: string;
  content: string;
  created_at: string;
  updated_at: string;
}

// ─── REQUEST TYPES ──────────────────────────────────────────────

export interface RegisterRequest {
  email: string;
  password: string;
}

export interface LoginRequest {
  email: string;
  password: string;
}

export interface CreateNoteRequest {
  title: string;
  content?: string;
}

export interface UpdateNoteRequest {
  title?: string;
  content?: string;
}

// ─── AUTH TYPES ─────────────────────────────────────────────────

export interface JWTPayload {
  userId: number;
  role: string;
}

export interface AuthRequest {
  user: JWTPayload;
}
```

### Key Design Decision: SafeUser

Notice `SafeUser` — it's `User` without `password_hash`. When the API returns user data, we **never** include the hash. Even though bcrypt hashes can't be reversed, there's no reason to send them to the frontend. Less data leaked = better security.

```typescript
// WRONG — leaks password_hash to the frontend:
res.json(user);  // { id: 1, email: "...", password_hash: "$2b$10$..." }

// RIGHT — only safe fields:
const { password_hash, ...safeUser } = user;
res.json(safeUser);  // { id: 1, email: "...", role: "user" }
```

---

## Step 2: Build the Error Handler

**Where?** `server/src/middleware/errorHandler.ts`

Same pattern as Level 2:

```typescript
import { Request, Response, NextFunction } from 'express';

export function errorHandler(
  err: Error,
  _req: Request,
  res: Response,
  _next: NextFunction
) {
  console.error('Error:', err.message);

  res.status(500).json({
    error: 'Internal server error',
  });
}
```

---

## Step 3: Build the Auth Middleware

**Where?** `server/src/middleware/authenticate.ts`

This is the gatekeeper. It runs before every protected route. Its job: extract the JWT from the request, verify it, and attach the user data to the request.

```typescript
import { Request, Response, NextFunction } from 'express';
import jwt from 'jsonwebtoken';
import { JWTPayload } from '../types';

const JWT_SECRET = process.env.JWT_SECRET || 'fallback-secret';

/**
 * Authentication middleware.
 *
 * Expects: Authorization: Bearer <token> in request headers.
 *
 * If valid:   Attaches req.user = { userId, role } and calls next()
 * If invalid: Sends 401 Unauthorized
 */
export function authenticate(req: Request, res: Response, next: NextFunction) {
  // 1. Get the Authorization header
  const authHeader = req.headers.authorization;

  if (!authHeader || !authHeader.startsWith('Bearer ')) {
    res.status(401).json({ error: 'No token provided' });
    return;
  }

  // 2. Extract the token (everything after "Bearer ")
  const token = authHeader.split(' ')[1];

  try {
    // 3. Verify the token signature and decode the payload
    const payload = jwt.verify(token, JWT_SECRET) as JWTPayload;

    // 4. Attach user data to the request object
    (req as any).user = payload;

    // 5. Continue to the route handler
    next();
  } catch {
    res.status(401).json({ error: 'Invalid or expired token' });
  }
}

/**
 * Authorization middleware for admin-only routes.
 * Must be used AFTER authenticate middleware.
 */
export function requireAdmin(req: Request, res: Response, next: NextFunction) {
  const user = (req as any).user as JWTPayload;

  if (user.role !== 'admin') {
    res.status(403).json({ error: 'Admin access required' });
    return;
  }

  next();
}
```

### How This Middleware Works

```
INCOMING REQUEST:
GET /api/notes
Authorization: Bearer eyJhbGciOiJIUzI1NiJ9.eyJ1c2VySWQiOjF9.abc123

           │
           ▼
┌──────────────────────────────────────────┐
│  authenticate middleware                  │
│                                          │
│  1. Read Authorization header            │
│     → "Bearer eyJhbGciOiJ..."            │
│                                          │
│  2. Extract token                        │
│     → "eyJhbGciOiJ..."                   │
│                                          │
│  3. jwt.verify(token, JWT_SECRET)        │
│     → Checks signature                   │
│     → Decodes payload: { userId: 1,      │
│                          role: "user" }   │
│                                          │
│  4. req.user = { userId: 1, role: "user"}│
│                                          │
│  5. next() → continue to route handler   │
└──────────────────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────┐
│  Route handler                            │
│                                          │
│  const userId = (req as any).user.userId │
│  // → 1                                  │
│                                          │
│  pool.query('SELECT * FROM notes         │
│    WHERE user_id = $1', [userId])        │
│  // → Only this user's notes             │
└──────────────────────────────────────────┘
```

### Key Concepts

**`Authorization: Bearer <token>`**

> [!NOTE]
> **Technical**: The Bearer authentication scheme is defined in RFC 6750. The client sends the token in the `Authorization` header with the `Bearer` prefix. The server extracts and validates the token.

> [!NOTE]
> **Plain English**: It's a convention: "Here's my token (my wristband). The word 'Bearer' just means 'the person carrying this token.'" Every protected request includes this header.

**`jwt.verify(token, JWT_SECRET)`**

This does two things:
1. Checks that the signature matches (proves the token wasn't tampered with)
2. Checks that the token hasn't expired (if an expiry was set)

If either check fails, it throws an error, and we send 401 Unauthorized.

**`(req as any).user`**

Express's Request type doesn't include a `user` property by default. We use `as any` to attach it. In production projects, you'd extend the Express Request type. For learning, `as any` keeps things simple.

**401 vs 403**

| Code | Meaning | When to Use |
|------|---------|-------------|
| 401 Unauthorized | "I don't know who you are" | No token, invalid token, expired token |
| 403 Forbidden | "I know who you are, but you can't do this" | Valid token, but wrong role (e.g., user trying admin route) |

---

> [!TIP]
> **Session Break** — You've built the JWT utilities and auth middleware that protect routes. Save your work and take a break.
> When you return, you'll build the registration and login routes.

---

## Step 4: Build the Auth Routes

**Where?** `server/src/routes/auth.ts`

This file handles registration, login, and the "who am I" endpoint.

```typescript
import { Router, Request, Response, NextFunction } from 'express';
import bcrypt from 'bcrypt';
import jwt from 'jsonwebtoken';
import pool from '../db';
import { RegisterRequest, LoginRequest, JWTPayload } from '../types';
import { authenticate } from '../middleware/authenticate';

const router = Router();

const JWT_SECRET = process.env.JWT_SECRET || 'fallback-secret';
const JWT_EXPIRES_IN = process.env.JWT_EXPIRES_IN || '7d';
const SALT_ROUNDS = 10;

// Helper: create a JWT for a user
function createToken(userId: number, role: string): string {
  return jwt.sign(
    { userId, role } as JWTPayload,
    JWT_SECRET,
    { expiresIn: JWT_EXPIRES_IN }
  );
}

// ─── POST /api/auth/register ────────────────────────────────────
// Create a new user account.
//
// Body: { email: "user@example.com", password: "securepassword" }
// Response: 201, { token: "eyJ...", user: { id, email, role } }
router.post(
  '/register',
  async (req: Request, res: Response, next: NextFunction) => {
    try {
      const { email, password } = req.body as RegisterRequest;

      // ─── VALIDATION ──────────────────────────────────────────
      if (!email || typeof email !== 'string') {
        res.status(400).json({ error: 'Email is required' });
        return;
      }

      // Basic email format check
      const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
      if (!emailRegex.test(email)) {
        res.status(400).json({ error: 'Invalid email format' });
        return;
      }

      if (!password || typeof password !== 'string') {
        res.status(400).json({ error: 'Password is required' });
        return;
      }

      if (password.length < 8) {
        res.status(400).json({ error: 'Password must be at least 8 characters' });
        return;
      }

      // ─── CHECK IF EMAIL ALREADY EXISTS ───────────────────────
      const existing = await pool.query(
        'SELECT id FROM users WHERE email = $1',
        [email.toLowerCase()]
      );

      if (existing.rows.length > 0) {
        res.status(409).json({ error: 'Email already registered' });
        return;
      }

      // ─── HASH PASSWORD ──────────────────────────────────────
      const passwordHash = await bcrypt.hash(password, SALT_ROUNDS);

      // ─── INSERT USER ────────────────────────────────────────
      const result = await pool.query(
        'INSERT INTO users (email, password_hash) VALUES ($1, $2) RETURNING id, email, role, created_at',
        [email.toLowerCase(), passwordHash]
      );

      const user = result.rows[0];

      // ─── CREATE TOKEN ───────────────────────────────────────
      const token = createToken(user.id, user.role);

      res.status(201).json({ token, user });
    } catch (err) {
      next(err);
    }
  }
);

// ─── POST /api/auth/login ───────────────────────────────────────
// Authenticate an existing user.
//
// Body: { email: "user@example.com", password: "securepassword" }
// Response: 200, { token: "eyJ...", user: { id, email, role } }
router.post(
  '/login',
  async (req: Request, res: Response, next: NextFunction) => {
    try {
      const { email, password } = req.body as LoginRequest;

      // ─── VALIDATION ──────────────────────────────────────────
      if (!email || !password) {
        res.status(400).json({ error: 'Email and password are required' });
        return;
      }

      // ─── FIND USER ──────────────────────────────────────────
      const result = await pool.query(
        'SELECT * FROM users WHERE email = $1',
        [email.toLowerCase()]
      );

      if (result.rows.length === 0) {
        // Don't reveal whether the email exists or not
        res.status(401).json({ error: 'Invalid email or password' });
        return;
      }

      const user = result.rows[0];

      // ─── VERIFY PASSWORD ────────────────────────────────────
      const isMatch = await bcrypt.compare(password, user.password_hash);

      if (!isMatch) {
        res.status(401).json({ error: 'Invalid email or password' });
        return;
      }

      // ─── CREATE TOKEN ───────────────────────────────────────
      const token = createToken(user.id, user.role);

      // Send user data WITHOUT password_hash
      const { password_hash, ...safeUser } = user;
      res.json({ token, user: safeUser });
    } catch (err) {
      next(err);
    }
  }
);

// ─── GET /api/auth/me ───────────────────────────────────────────
// Get the current authenticated user's info.
// Requires: valid JWT in Authorization header.
router.get(
  '/me',
  authenticate,
  async (req: Request, res: Response, next: NextFunction) => {
    try {
      const { userId } = (req as any).user as JWTPayload;

      const result = await pool.query(
        'SELECT id, email, role, created_at FROM users WHERE id = $1',
        [userId]
      );

      if (result.rows.length === 0) {
        res.status(404).json({ error: 'User not found' });
        return;
      }

      res.json(result.rows[0]);
    } catch (err) {
      next(err);
    }
  }
);

export { router as authRouter };
```

### Registration Flow Explained

```
Client sends: { email: "alice@example.com", password: "mypassword123" }
                                    │
                                    ▼
                    ┌───────────────────────────────┐
                    │  1. Validate email + password  │
                    │     Is email valid format?     │
                    │     Is password 8+ chars?      │
                    └───────────────┬───────────────┘
                                    │
                                    ▼
                    ┌───────────────────────────────┐
                    │  2. Check if email exists      │
                    │     SELECT FROM users          │
                    │     WHERE email = $1           │
                    └───────────────┬───────────────┘
                                    │
                                    ▼
                    ┌───────────────────────────────┐
                    │  3. Hash the password          │
                    │     bcrypt.hash(password, 10)  │
                    │                               │
                    │     "mypassword123"            │
                    │        ↓                       │
                    │     "$2b$10$X7vK9z...q3Fy"     │
                    └───────────────┬───────────────┘
                                    │
                                    ▼
                    ┌───────────────────────────────┐
                    │  4. Store in database          │
                    │     INSERT INTO users          │
                    │     (email, password_hash)     │
                    │     VALUES ($1, $2)            │
                    │                               │
                    │     Plain text password is     │
                    │     NEVER stored anywhere      │
                    └───────────────┬───────────────┘
                                    │
                                    ▼
                    ┌───────────────────────────────┐
                    │  5. Create JWT token           │
                    │     jwt.sign(                  │
                    │       { userId: 1, role: "user"},│
                    │       JWT_SECRET,              │
                    │       { expiresIn: "7d" }      │
                    │     )                          │
                    └───────────────┬───────────────┘
                                    │
                                    ▼
              Send back: { token: "eyJ...", user: { id: 1, email: "...", role: "user" } }
```

### Key Security Details

**`bcrypt.hash(password, SALT_ROUNDS)`**

The second argument is the cost factor (salt rounds). Higher = slower = more secure. 10 is the standard recommendation — it takes about 100ms to hash, which is fast enough for login but slow enough to make brute-force attacks impractical.

**`bcrypt.compare(password, hash)`**

This does NOT decrypt the hash (that's impossible). It hashes the provided password with the same salt and algorithm, then compares the results. If they match, the password is correct.

**"Invalid email or password" (not "Email not found" or "Wrong password")**

> [!WARNING]
> Never reveal which part of the login failed. If you say "Email not found," an attacker learns which emails are registered. If you say "Wrong password," they know the email exists and can focus on guessing the password. Always use a generic message: "Invalid email or password."

**`email.toLowerCase()`**

Emails are case-insensitive. `Alice@Example.com` and `alice@example.com` are the same mailbox. We lowercase before storing and querying to prevent duplicate accounts.

**409 Conflict**

The status code for "email already registered." 409 means "the request conflicts with the current state of the resource."

**RETURNING without password_hash**

```sql
INSERT INTO users (email, password_hash)
VALUES ($1, $2)
RETURNING id, email, role, created_at
```

We explicitly list which columns to return, omitting `password_hash`. This is safer than `RETURNING *` followed by deleting the field in code.

---

> [!TIP]
> **Session Break** — You've built the complete auth routes with registration, login, and token verification. Save your work and take a break.
> When you return, you'll test all the auth endpoints with curl.

---

## Step 5: Test the Auth Endpoints

Create a temporary server to test. Replace `server/src/index.ts` with:

```typescript
import dotenv from 'dotenv';
dotenv.config();

import express from 'express';
import cors from 'cors';
import { authRouter } from './routes/auth';
import { errorHandler } from './middleware/errorHandler';

const app = express();
const PORT = process.env.PORT || 3001;

app.use(express.json());
app.use(cors({ origin: process.env.CORS_ORIGIN || 'http://localhost:5173' }));

app.use('/api/auth', authRouter);

app.get('/api/health', (_req, res) => {
  res.json({ status: 'ok' });
});

app.use(errorHandler);

app.listen(PORT, () => {
  console.log(`Server running on http://localhost:${PORT}`);
});
```

Start the server:

> [!IMPORTANT]
> **You should be in:** `vault-note/`

```bash
cd server
npm run dev
```

In a second terminal, test:

### Register

```bash
curl -X POST http://localhost:3001/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{"email":"alice@example.com","password":"password123"}'
```

Expected: `{"token":"eyJ...","user":{"id":1,"email":"alice@example.com","role":"user","created_at":"..."}}`

**Copy the token** — you'll need it for the next test.

### Register Duplicate (Should Fail)

```bash
curl -X POST http://localhost:3001/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{"email":"alice@example.com","password":"differentpassword"}'
```

Expected: `{"error":"Email already registered"}`

### Login

```bash
curl -X POST http://localhost:3001/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"alice@example.com","password":"password123"}'
```

Expected: `{"token":"eyJ...","user":{...}}`

### Login Wrong Password

```bash
curl -X POST http://localhost:3001/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"alice@example.com","password":"wrongpassword"}'
```

Expected: `{"error":"Invalid email or password"}`

### Get Current User (Authenticated)

```bash
curl http://localhost:3001/api/auth/me \
  -H "Authorization: Bearer YOUR_TOKEN_HERE"
```

Replace `YOUR_TOKEN_HERE` with the token from the register or login response.

Expected: `{"id":1,"email":"alice@example.com","role":"user","created_at":"..."}`

### Get Current User (No Token)

```bash
curl http://localhost:3001/api/auth/me
```

Expected: `{"error":"No token provided"}`

Stop the server and go back to the project root:

```bash
cd ..
```

> [!IMPORTANT]
> **You are now in:** `vault-note/`

---

## Step 6: Commit

> [!IMPORTANT]
> **You should be in:** `vault-note/`

```bash
git add server/src/
git commit -m "feat: add authentication with registration, login, and JWT middleware"
```

---

## Where Tokens Are Stored (and the Trade-offs)

In this project, the frontend stores the JWT in `localStorage`. Here's the trade-off:

| Storage | Pros | Cons |
|---------|------|------|
| **localStorage** | Simple to implement, persists across tabs, easy to read in JS | Vulnerable to XSS (if attacker injects JS, they can read the token) |
| **HTTP-only cookie** | JavaScript can't read it (XSS-safe), sent automatically | Requires more backend setup, vulnerable to CSRF, more complex CORS |

> [!NOTE]
> **For learning, localStorage is the right choice.** It's simpler, the auth flow is more visible (you can see the token in DevTools), and XSS protection is better handled by not having XSS vulnerabilities in the first place. In production, HTTP-only cookies are considered more secure. The concepts are the same — only the storage mechanism differs.

---

> [!TIP]
> ## Spatial Check-In

1. **What does bcrypt.hash() do?**

<details><summary>Answer</summary>

Takes a plain text password and a salt rounds number, returns a one-way hash. The same password produces different hashes each time (due to random salt), but bcrypt.compare() can still verify them.

</details>

2. **What does jwt.sign() do?**

<details><summary>Answer</summary>

Creates a JWT token by encoding a payload (like `{ userId: 1, role: "user" }`) and signing it with a secret key. The signature proves the token was created by our server.

</details>

3. **What does jwt.verify() do?**

<details><summary>Answer</summary>

Takes a token and the secret key, checks that the signature is valid and the token hasn't expired. Returns the decoded payload if valid, throws an error if not.

</details>

4. **Why does the login route say "Invalid email or password" instead of specifying which is wrong?**

<details><summary>Answer</summary>

To prevent information leakage. Saying "email not found" tells attackers which emails are registered. A generic message gives no useful information to an attacker.

</details>

5. **What is the difference between 401 and 403?**

<details><summary>Answer</summary>

401 means "I don't know who you are" (missing/invalid token). 403 means "I know who you are, but you're not allowed" (valid token, insufficient permissions).

</details>

---

| | | |
|:---|:---:|---:|
| [← 03 — Database Design](../03-database/) | [Level 3 Overview](../) | [05 — Building the Backend →](../05-backend/) |

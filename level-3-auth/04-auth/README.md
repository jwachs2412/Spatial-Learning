# Step 4 — Authentication: Registration, Login, and JWT

## New Syntax You'll Meet in This Lesson

You already know JS/TS basics from Level 1, the Express request/response/middleware shapes from Level 2, and the `pool.query` / parameterized-query pattern. Level 3 adds a few new building blocks — meet them once here, then watch them recur in every auth file:

- **`bcrypt.hash(plaintext, costFactor)`** — async function that returns a hashed string. We use cost factor 10 (≈100ms per hash). Higher = slower = harder to brute-force, but slower for legitimate users too.
- **`bcrypt.compare(plaintext, storedHash)`** — async function that re-hashes the plaintext and checks if it would have produced the stored hash. Returns `true` or `false`. Never decrypts — you can't get the password back.
- **`jwt.sign(payload, secret, options)`** — synchronous function that builds and returns a token string. The payload is your data (e.g. `{ userId: 5 }`). The secret is your `JWT_SECRET`. Options include `{ expiresIn: '7d' }`.
- **`jwt.verify(token, secret)`** — synchronous function that checks the signature and returns the original payload. **Throws** if the token is invalid, tampered, or expired. We wrap it in `try/catch`.
- **Interface extension** — `interface AuthRequest extends Request { user?: JWTPayload }` adds a custom property to Express's built-in `Request` type so the rest of our code knows that, on protected routes, `req.user` exists.
- **Optional chaining + non-null assertion** — `req.user?.role` reads "the role if user exists, otherwise undefined." `req.user!.userId` (with `!`) reads "I, the developer, guarantee `req.user` is defined here, so don't make me check." Use `!` only after you've already proven the value exists (e.g., immediately after the `authenticate` middleware ran).
- **HTTP 401 vs 403** — `401 Unauthorized` means "I don't know who you are; provide credentials." `403 Forbidden` means "I know who you are, but you can't do this." Auth middleware returns 401; admin checks return 403.
- **HTTP 409** — `409 Conflict`. Used when registering an email that's already taken — semantically more correct than a generic 400.

Every code block below has a "Reading This File Line-by-Line" walkthrough using these patterns.

## Spatial Orientation

```
vault-note/
└── server/
    └── src/
        ├── types/
        │   └── index.ts         ← Auth types
        ├── middleware/
        │   ├── authenticate.ts  ← JWT verification middleware
        │   └── errorHandler.ts  ← Error handler
        └── routes/
            └── auth.ts          ← Register + Login endpoints
```

This is the most security-critical code in the curriculum. Every line has a security reason.

---

## 1. Define Types

### 🏗️ Your Turn

Think about what types you need for an auth system:
- A `User` type (everything in the database)
- A `SafeUser` type (user data WITHOUT the password hash — for API responses)
- A `JWTPayload` type (what goes inside the token)
- An `AuthRequest` type (Express Request with user data attached by middleware)

<details>
<summary>See the solution</summary>

Create `server/src/types/index.ts`:

```typescript
import { Request } from 'express';

export interface User {
  id: number;
  email: string;
  password_hash: string;
  role: string;
  created_at: string;
}

// Never send password_hash to the client
export interface SafeUser {
  id: number;
  email: string;
  role: string;
  created_at: string;
}

export interface JWTPayload {
  userId: number;
  role: string;
}

export interface AuthRequest extends Request {
  user?: JWTPayload;
}

export interface Note {
  id: number;
  user_id: number;
  title: string;
  content: string;
  created_at: string;
  updated_at: string;
}

export interface CreateNoteRequest {
  title: string;
  content?: string;
}

export interface UpdateNoteRequest {
  title?: string;
  content?: string;
}
```

</details>

### Reading This File Line-by-Line

```typescript
import { Request } from 'express';
```

A type-only import. We need Express's `Request` type to extend it.

```typescript
export interface User {
  id: number;
  email: string;
  password_hash: string;
  role: string;
  created_at: string;
}

export interface SafeUser {
  id: number;
  email: string;
  role: string;
  created_at: string;
}
```

Two interfaces describing the same entity from two angles:

- **`User`** matches a database row exactly — including `password_hash`. Anywhere we read a full row from the `users` table, we type it as `User`.
- **`SafeUser`** is the "safe to send to clients" version. No `password_hash` field. It's the type for API responses, route handler return values, and anything that crosses the trust boundary.

By splitting the type, we make password leaks **a TypeScript compile error** instead of a runtime mistake. Trying to assign a `User` to something typed `SafeUser` requires explicitly stripping the field.

```typescript
export interface JWTPayload {
  userId: number;
  role: string;
}
```

The exact shape of what we put inside JWTs. Keep it small (smaller payload = smaller token) and contain only what middleware needs to make decisions: `userId` (who is this?) and `role` (what role do they have?).

```typescript
export interface AuthRequest extends Request {
  user?: JWTPayload;
}
```

This is the **interface extension** trick. We're saying "an `AuthRequest` is everything Express's `Request` is, plus a `user` field that might be there." The `?` makes `user` optional — middleware sets it; before middleware runs, it's undefined.

When we type a route handler with `(req: AuthRequest, ...)`, TypeScript autocompletes `req.user` and warns us to handle the optional case.

```typescript
export interface Note { ... }
export interface CreateNoteRequest { ... }
export interface UpdateNoteRequest { ... }
```

Standard CRUD types — same shape as projects/tasks in Level 2.

**Key decision: `SafeUser` omits `password_hash`.** The password hash should NEVER leave the server. `SafeUser` is what you send in API responses. This type-level separation prevents accidental leaks.

---

## 2. Build the Error Handler

Create `server/src/middleware/errorHandler.ts`:

```typescript
import { Request, Response, NextFunction } from 'express';

export const errorHandler = (
  err: Error,
  _req: Request,
  res: Response,
  _next: NextFunction,
): void => {
  console.error('Error:', err.message);
  res.status(500).json({ error: 'Internal server error' });
};
```

---

## 3. Build the Auth Middleware

> **Key Concept: Auth Middleware**
> This middleware runs BEFORE your route handlers. It extracts the JWT from the `Authorization` header, verifies the signature, and attaches the user data to the request object. If the token is invalid or missing, it returns 401 immediately — the route handler never runs.

### 🏗️ Your Turn

Build the authenticate middleware. It needs to:
1. Read the `Authorization` header
2. Extract the token (format: `Bearer <token>`)
3. Verify the token using `jwt.verify(token, JWT_SECRET)`
4. Attach the decoded payload to `req.user`
5. Call `next()` to continue to the route handler
6. Return 401 if token is missing or invalid

Also build a `requireAdmin` middleware that checks `req.user.role === 'admin'`.

<details>
<summary>See the solution</summary>

Create `server/src/middleware/authenticate.ts`:

```typescript
import { Response, NextFunction } from 'express';
import jwt from 'jsonwebtoken';
import { AuthRequest, JWTPayload } from '../types';

const JWT_SECRET = process.env.JWT_SECRET || 'dev-secret-change-in-production';

export const authenticate = (
  req: AuthRequest,
  res: Response,
  next: NextFunction,
): void => {
  const authHeader = req.headers.authorization;

  if (!authHeader || !authHeader.startsWith('Bearer ')) {
    res.status(401).json({ error: 'No token provided' });
    return;
  }

  const token = authHeader.split(' ')[1];

  try {
    const decoded = jwt.verify(token, JWT_SECRET) as JWTPayload;
    req.user = decoded;
    next();
  } catch {
    res.status(401).json({ error: 'Invalid or expired token' });
  }
};

export const requireAdmin = (
  req: AuthRequest,
  res: Response,
  next: NextFunction,
): void => {
  if (req.user?.role !== 'admin') {
    res.status(403).json({ error: 'Admin access required' });
    return;
  }
  next();
};
```

</details>

### Reading This File Line-by-Line

```typescript
import { Response, NextFunction } from 'express';
import jwt from 'jsonwebtoken';
import { AuthRequest, JWTPayload } from '../types';
```

Note: we import our own `AuthRequest` (not Express's `Request`) because this middleware sets `req.user`.

```typescript
const JWT_SECRET = process.env.JWT_SECRET || 'dev-secret-change-in-production';
```

Read the secret from env once and cache it in a constant. The fallback string is a development-only convenience — in production, you must set `JWT_SECRET` to a real value, otherwise tokens are trivially forgeable.

```typescript
export const authenticate = (
  req: AuthRequest,
  res: Response,
  next: NextFunction,
): void => {
```

A standard three-arg middleware, but typed with `AuthRequest` instead of `Request` because we'll write to `req.user` below.

```typescript
const authHeader = req.headers.authorization;

if (!authHeader || !authHeader.startsWith('Bearer ')) {
  res.status(401).json({ error: 'No token provided' });
  return;
}
```

- `req.headers.authorization` — Express normalizes header names to lowercase for you. The actual HTTP header is `Authorization: Bearer eyJ...` but we read it as `req.headers.authorization`.
- `authHeader.startsWith('Bearer ')` — every string in JS has a `.startsWith(prefix)` method that returns `true` if the string begins with `prefix`. Note the trailing space — `'Bearer '` (with space) catches the standard format.
- If the header is missing OR doesn't start with `'Bearer '`, return 401 and stop.

```typescript
const token = authHeader.split(' ')[1];
```

`'Bearer eyJ...'.split(' ')` returns `['Bearer', 'eyJ...']`. Index `[1]` is the token alone.

```typescript
try {
  const decoded = jwt.verify(token, JWT_SECRET) as JWTPayload;
  req.user = decoded;
  next();
} catch {
  res.status(401).json({ error: 'Invalid or expired token' });
}
```

- `jwt.verify(token, JWT_SECRET)` does three checks in one call:
  1. The token is well-formed.
  2. The signature matches (i.e., the token was signed with `JWT_SECRET` and not modified after).
  3. The `exp` claim hasn't passed.
  - If any check fails, it **throws**.
- `as JWTPayload` — type assertion. `jwt.verify` returns a generic `string | JwtPayload`. We narrow it to our app-specific `JWTPayload` type. Safe because we only ever sign payloads of that shape.
- `req.user = decoded;` — stash the payload on the request so route handlers can read `req.user.userId` and `req.user.role`.
- `next()` — pass control to the next middleware (or route handler).
- `catch { ... }` — no parens because we don't need the error object. Any verification failure means the token is invalid; we respond 401. We do NOT call `next()` here because we already sent a response.

```typescript
export const requireAdmin = (
  req: AuthRequest,
  res: Response,
  next: NextFunction,
): void => {
  if (req.user?.role !== 'admin') {
    res.status(403).json({ error: 'Admin access required' });
    return;
  }
  next();
};
```

- A second middleware to **stack on top of `authenticate`** for admin-only endpoints.
- `req.user?.role` uses optional chaining: if `req.user` is undefined, the whole expression is `undefined`, and `undefined !== 'admin'` is `true`, so non-admin (and unauthenticated) requests are rejected.
- `403 Forbidden` (not 401) because the user IS authenticated — they're just not allowed.

To use it in a route: `router.get('/admin/users', authenticate, requireAdmin, handler)`. Express runs them left to right.

> ⚠️ **Common Mistake: `AuthRequest` extends `Request`**
> Standard Express `Request` doesn't have a `user` property. We created `AuthRequest` that extends `Request` and adds `user?: JWTPayload`. The middleware uses `AuthRequest`, and so do protected route handlers.

---

## 4. Build Auth Routes

### 🏗️ Your Turn — Registration

Build the register endpoint. It needs to:
1. Validate email and password (email required, password 8+ characters)
2. Check if email already exists (return 409 Conflict if so)
3. Hash the password with bcrypt
4. Insert the user into the database (using `RETURNING` but excluding `password_hash`)
5. Create a JWT token
6. Return the token and safe user data

Hints:
- `bcrypt.hash(password, 10)` — the 10 is the cost factor (iterations of hashing)
- `jwt.sign({ userId, role }, JWT_SECRET, { expiresIn: JWT_EXPIRES_IN })`
- Use `email.toLowerCase()` for case-insensitive matching

<details>
<summary>See the solution</summary>

Create `server/src/routes/auth.ts`:

```typescript
import { Router, Request, Response, NextFunction } from 'express';
import bcrypt from 'bcryptjs';
import jwt from 'jsonwebtoken';
import pool from '../db';
import { SafeUser, AuthRequest } from '../types';
import { authenticate } from '../middleware/authenticate';

const router = Router();
const JWT_SECRET = process.env.JWT_SECRET || 'dev-secret-change-in-production';
const JWT_EXPIRES_IN = process.env.JWT_EXPIRES_IN || '7d';
const SALT_ROUNDS = 10;

// POST /api/auth/register
router.post('/register', async (req: Request, res: Response, next: NextFunction) => {
  try {
    const { email, password } = req.body;

    // Validate
    if (!email || !password) {
      res.status(400).json({ error: 'Email and password are required' });
      return;
    }

    if (password.length < 8) {
      res.status(400).json({ error: 'Password must be at least 8 characters' });
      return;
    }

    const normalizedEmail = email.toLowerCase().trim();

    // Check for existing user
    const existingUser = await pool.query(
      'SELECT id FROM users WHERE email = $1',
      [normalizedEmail]
    );

    if (existingUser.rows.length > 0) {
      res.status(409).json({ error: 'Email already registered' });
      return;
    }

    // Hash password
    const passwordHash = await bcrypt.hash(password, SALT_ROUNDS);

    // Create user (RETURNING excludes password_hash)
    const result = await pool.query(
      'INSERT INTO users (email, password_hash) VALUES ($1, $2) RETURNING id, email, role, created_at',
      [normalizedEmail, passwordHash]
    );

    const user: SafeUser = result.rows[0];

    // Create token
    const token = jwt.sign(
      { userId: user.id, role: user.role },
      JWT_SECRET,
      { expiresIn: JWT_EXPIRES_IN }
    );

    res.status(201).json({ token, user });
  } catch (err) {
    next(err);
  }
});

// POST /api/auth/login
router.post('/login', async (req: Request, res: Response, next: NextFunction) => {
  try {
    const { email, password } = req.body;

    if (!email || !password) {
      res.status(400).json({ error: 'Email and password are required' });
      return;
    }

    const normalizedEmail = email.toLowerCase().trim();

    // Find user
    const result = await pool.query(
      'SELECT * FROM users WHERE email = $1',
      [normalizedEmail]
    );

    if (result.rows.length === 0) {
      res.status(401).json({ error: 'Invalid email or password' });
      return;
    }

    const user = result.rows[0];

    // Compare password
    const isValid = await bcrypt.compare(password, user.password_hash);

    if (!isValid) {
      res.status(401).json({ error: 'Invalid email or password' });
      return;
    }

    // Create token
    const token = jwt.sign(
      { userId: user.id, role: user.role },
      JWT_SECRET,
      { expiresIn: JWT_EXPIRES_IN }
    );

    const safeUser: SafeUser = {
      id: user.id,
      email: user.email,
      role: user.role,
      created_at: user.created_at,
    };

    res.json({ token, user: safeUser });
  } catch (err) {
    next(err);
  }
});

// GET /api/auth/me — Get current user (protected)
router.get('/me', authenticate, async (req: AuthRequest, res: Response, next: NextFunction) => {
  try {
    const result = await pool.query(
      'SELECT id, email, role, created_at FROM users WHERE id = $1',
      [req.user!.userId]
    );

    if (result.rows.length === 0) {
      res.status(404).json({ error: 'User not found' });
      return;
    }

    res.json(result.rows[0]);
  } catch (err) {
    next(err);
  }
});

export { router as authRouter };
```

</details>

### Reading This File Line-by-Line

This file has three endpoints: register, login, and "me." Each uses bcrypt and/or JWT in slightly different ways. Read them once carefully and the pattern is yours.

#### Setup at the Top

```typescript
const router = Router();
const JWT_SECRET = process.env.JWT_SECRET || 'dev-secret-change-in-production';
const JWT_EXPIRES_IN = process.env.JWT_EXPIRES_IN || '7d';
const SALT_ROUNDS = 10;
```

- `JWT_SECRET` and `JWT_EXPIRES_IN` from env, with development fallbacks.
- `SALT_ROUNDS = 10` — bcrypt's cost factor. The "10" means 2^10 = 1024 internal iterations. Each increment doubles the time. 10 yields ~100ms per hash on typical hardware: slow enough to deter brute-force, fast enough for users at registration/login. Don't go above 12 in production unless you've measured.

#### Register Endpoint

```typescript
router.post('/register', async (req: Request, res: Response, next: NextFunction) => {
  try {
    const { email, password } = req.body;
```

Standard async route + destructure body. Note `req: Request` (not `AuthRequest`) — this endpoint runs before any token exists.

```typescript
    if (!email || !password) {
      res.status(400).json({ error: 'Email and password are required' });
      return;
    }

    if (password.length < 8) {
      res.status(400).json({ error: 'Password must be at least 8 characters' });
      return;
    }
```

Two basic validations: presence and minimum length.

```typescript
    const normalizedEmail = email.toLowerCase().trim();
```

- `.toLowerCase()` — emails are case-insensitive. `JOHN@example.com` and `john@example.com` are the same account. Storing them in lowercase ensures consistent lookups.
- `.trim()` — strip leading/trailing whitespace from copy-paste mistakes.

```typescript
    const existingUser = await pool.query(
      'SELECT id FROM users WHERE email = $1',
      [normalizedEmail]
    );

    if (existingUser.rows.length > 0) {
      res.status(409).json({ error: 'Email already registered' });
      return;
    }
```

- Query for an existing user with that email. We `SELECT id` (not `*`) because we only need to know whether a row exists, not its contents.
- If found, return `409 Conflict` — semantically correct for "this resource already exists." (We could rely on the database's `UNIQUE` constraint to throw, but checking up front gives a cleaner error.)

```typescript
    const passwordHash = await bcrypt.hash(password, SALT_ROUNDS);
```

- `bcrypt.hash` is async — it takes ~100ms and returns a string like `$2a$10$N9qo8uLOickgx2ZMRZoMye...`. The format encodes the algorithm version, cost factor, salt, and hash all in one string. bcrypt knows how to extract those when verifying later.

```typescript
    const result = await pool.query(
      'INSERT INTO users (email, password_hash) VALUES ($1, $2) RETURNING id, email, role, created_at',
      [normalizedEmail, passwordHash]
    );

    const user: SafeUser = result.rows[0];
```

- The `INSERT` uses parameterized values (Level 2 review).
- **Crucially:** `RETURNING id, email, role, created_at` — we explicitly list the columns we want back. We do NOT include `password_hash` in the `RETURNING` list. Even by accident, the hash never leaves the database.
- `const user: SafeUser = result.rows[0]` — TypeScript will reject this assignment if the row contains fields not in `SafeUser`. The explicit `RETURNING` columns line up perfectly with `SafeUser`. Type safety + SQL precision = defense in depth.

```typescript
    const token = jwt.sign(
      { userId: user.id, role: user.role },
      JWT_SECRET,
      { expiresIn: JWT_EXPIRES_IN }
    );
```

- `jwt.sign(payload, secret, options)`:
  - `payload` is the data we're putting in the token. **Don't put sensitive data here** — JWT payloads are readable by anyone (just base64). Just identifiers.
  - `secret` is `JWT_SECRET` from env. The signature is computed using this; without it, no one can produce or modify a valid token.
  - `options.expiresIn: '7d'` — JWT library reads this and stamps an `exp` claim 7 days in the future. After that timestamp, `jwt.verify` will throw.

```typescript
    res.status(201).json({ token, user });
  } catch (err) {
    next(err);
  }
});
```

- `201 Created` for a successful registration.
- Response body: `{ token, user }`. The frontend stores both: token for subsequent requests, user for displaying name/role in the UI.

#### Login Endpoint

```typescript
router.post('/login', async (req: Request, res: Response, next: NextFunction) => {
  try {
    const { email, password } = req.body;

    if (!email || !password) { ... return; }

    const normalizedEmail = email.toLowerCase().trim();

    const result = await pool.query(
      'SELECT * FROM users WHERE email = $1',
      [normalizedEmail]
    );

    if (result.rows.length === 0) {
      res.status(401).json({ error: 'Invalid email or password' });
      return;
    }
```

- `SELECT *` — here we DO want all fields, including `password_hash`, because we need to compare it against the input.
- If the email doesn't exist, return 401 with a generic message. **Don't reveal which check failed** (see Security Exercise below).

```typescript
    const user = result.rows[0];
    const isValid = await bcrypt.compare(password, user.password_hash);

    if (!isValid) {
      res.status(401).json({ error: 'Invalid email or password' });
      return;
    }
```

- `bcrypt.compare(plaintext, hash)` — internally hashes `plaintext` using the same salt and cost factor stored inside `hash`, then compares the result. Returns a boolean.
- Same generic 401 message if the password doesn't match.

```typescript
    const token = jwt.sign(
      { userId: user.id, role: user.role },
      JWT_SECRET,
      { expiresIn: JWT_EXPIRES_IN }
    );

    const safeUser: SafeUser = {
      id: user.id,
      email: user.email,
      role: user.role,
      created_at: user.created_at,
    };

    res.json({ token, user: safeUser });
  } catch (err) {
    next(err);
  }
});
```

- Sign a token, build a `SafeUser` by hand (we have the full `User` here, so we manually pick the safe fields), and respond.
- `res.json(...)` defaults to status 200, which is correct for login.

#### "Me" Endpoint (Protected)

```typescript
router.get('/me', authenticate, async (req: AuthRequest, res: Response, next: NextFunction) => {
  try {
    const result = await pool.query(
      'SELECT id, email, role, created_at FROM users WHERE id = $1',
      [req.user!.userId]
    );
```

- `router.get('/me', authenticate, handler)` — the `authenticate` middleware runs **before** the handler. If the token is missing or invalid, the handler never runs.
- `req: AuthRequest` — typed so TypeScript knows about `req.user`.
- `req.user!.userId` — the `!` is a **non-null assertion**. We're telling TypeScript "I'm certain `req.user` exists here." This is true because `authenticate` has already run; if `req.user` were missing, that middleware would have already responded 401 and we wouldn't be in this handler. The `!` lets us skip the redundant `if (req.user)` check.

The login route returns the same error for "email not found" and "wrong password":
```typescript
res.status(401).json({ error: 'Invalid email or password' });
```

Why not tell the user specifically which one was wrong?

<details>
<summary>Answer</summary>

**Enumeration attack prevention.** If you return "No account with that email," an attacker can probe your API to build a list of valid email addresses. Then they only need to crack passwords for known accounts. By returning a generic message, you don't reveal whether an email exists in your system.

</details>

---

## 5. Test Auth Endpoints

```bash
# Register
curl -X POST http://localhost:3001/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{"email": "test@example.com", "password": "password123"}'

# Login
curl -X POST http://localhost:3001/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email": "test@example.com", "password": "password123"}'

# Get current user (use the token from login response)
curl http://localhost:3001/api/auth/me \
  -H "Authorization: Bearer YOUR_TOKEN_HERE"

# Test invalid token
curl http://localhost:3001/api/auth/me \
  -H "Authorization: Bearer invalid-token"
```

### ✅ Checkpoint

- [ ] Register creates a user and returns a token
- [ ] Login with correct credentials returns a token
- [ ] Login with wrong password returns generic error
- [ ] `/me` with valid token returns user data (no password_hash!)
- [ ] `/me` without token returns 401
- [ ] Registering with existing email returns 409

---

## 6. Commit

```bash
git add .
git commit -m "feat: add auth routes with JWT, bcrypt, and middleware"
```

---

## 🧠 Spatial Check-In

1. Why do we use `SALT_ROUNDS = 10` with bcrypt? What happens if you increase it?

2. The `RETURNING` clause in the register INSERT excludes `password_hash`. Why is this important?

3. What would happen if the `JWT_SECRET` was committed to GitHub in your source code?

<details>
<summary>Check Your Answers</summary>

1. **Cost factor.** 10 means 2^10 = 1024 iterations of the hashing algorithm. Each increment doubles the time. At 10, hashing takes ~100ms — slow enough to deter brute-force, fast enough for normal use. At 15, it takes ~3 seconds — too slow for user registration.

2. **Defense in depth.** Even if there's a bug later that sends the raw database row to the client, the password hash was never selected from the database. The `RETURNING` clause acts as a second layer of protection (the first being `SafeUser`).

3. **Anyone could forge tokens.** With the secret, an attacker creates valid JWTs with any userId and role (including admin). They'd have full access to every user's data. This is why `JWT_SECRET` lives in environment variables, never in code.

</details>

---

> **Session Break** — Authentication system built.
> When you return, you'll build protected CRUD routes in [Step 5 — Backend](../05-backend/).

---

| | |
|:---|---:|
| [← Step 3: Database](../03-database/) | [Step 5 — Backend →](../05-backend/) |

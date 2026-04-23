# Step 4 — Authentication: Registration, Login, and JWT

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

**Line-by-line breakdown:**

| Line | What It Does | Why |
|------|-------------|-----|
| `req.headers.authorization` | Read the Authorization header | Tokens are sent as `Authorization: Bearer <token>` |
| `authHeader.startsWith('Bearer ')` | Check the format | The `Bearer ` prefix is an HTTP standard for token auth |
| `authHeader.split(' ')[1]` | Extract just the token string | Removes the `Bearer ` prefix |
| `jwt.verify(token, JWT_SECRET)` | Verify the signature | If the token was tampered with, this throws |
| `as JWTPayload` | Cast the decoded data to our type | `jwt.verify` returns a generic type |
| `req.user = decoded` | Attach user data to the request | Route handlers can now access `req.user.userId` |

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

### 🔐 Security Exercise

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

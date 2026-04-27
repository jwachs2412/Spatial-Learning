# Step 5 — Building the Backend: Protected CRUD Routes

## Spatial Orientation

```
vault-note/
└── server/
    └── src/
        └── routes/
            ├── auth.ts    ← Built in Step 4 (public)
            ├── notes.ts   ← YOU BUILD THIS (protected)
            └── admin.ts   ← YOU BUILD THIS (admin-only)
```

---

## 1. Notes Routes (Protected)

> **Key Concept: Row-Level Security**
> Every query includes `WHERE user_id = $1` using the authenticated user's ID. This means users can ONLY access their own notes — even if they guess another note's ID. The database enforces the boundary, not just the frontend.

### 🏗️ Your Turn

Build CRUD routes for notes. Critical requirement: **every query must include `AND user_id = $2`** to ensure users can only access their own data.

Think about why we return 404 (not 403) when a note belongs to another user.

<details>
<summary>See the solution</summary>

Create `server/src/routes/notes.ts`:

```typescript
import { Router, Response, NextFunction } from 'express';
import pool from '../db';
import { AuthRequest, CreateNoteRequest, UpdateNoteRequest } from '../types';
import { authenticate } from '../middleware/authenticate';

const router = Router();

// All routes in this file require authentication
router.use(authenticate);

// GET /api/notes — List user's notes
router.get('/', async (req: AuthRequest, res: Response, next: NextFunction) => {
  try {
    const result = await pool.query(
      'SELECT * FROM notes WHERE user_id = $1 ORDER BY updated_at DESC',
      [req.user!.userId]
    );
    res.json(result.rows);
  } catch (err) {
    next(err);
  }
});

// POST /api/notes — Create a note
router.post('/', async (req: AuthRequest, res: Response, next: NextFunction) => {
  try {
    const { title, content } = req.body as CreateNoteRequest;

    if (!title || title.trim().length === 0) {
      res.status(400).json({ error: 'Title is required' });
      return;
    }

    if (title.length > 200) {
      res.status(400).json({ error: 'Title must be 200 characters or less' });
      return;
    }

    const result = await pool.query(
      'INSERT INTO notes (user_id, title, content) VALUES ($1, $2, $3) RETURNING *',
      [req.user!.userId, title.trim(), content || '']
    );

    res.status(201).json(result.rows[0]);
  } catch (err) {
    next(err);
  }
});

// GET /api/notes/:id — Get a single note
router.get('/:id', async (req: AuthRequest, res: Response, next: NextFunction) => {
  try {
    const result = await pool.query(
      'SELECT * FROM notes WHERE id = $1 AND user_id = $2',
      [req.params.id, req.user!.userId]
    );

    if (result.rows.length === 0) {
      res.status(404).json({ error: 'Note not found' });
      return;
    }

    res.json(result.rows[0]);
  } catch (err) {
    next(err);
  }
});

// PUT /api/notes/:id — Update a note
router.put('/:id', async (req: AuthRequest, res: Response, next: NextFunction) => {
  try {
    const { title, content } = req.body as UpdateNoteRequest;

    if (title !== undefined && title.trim().length === 0) {
      res.status(400).json({ error: 'Title cannot be empty' });
      return;
    }

    const updates: string[] = [];
    const values: (string | number)[] = [];
    let paramIndex = 1;

    if (title !== undefined) {
      updates.push(`title = $${paramIndex++}`);
      values.push(title.trim());
    }

    if (content !== undefined) {
      updates.push(`content = $${paramIndex++}`);
      values.push(content);
    }

    if (updates.length === 0) {
      res.status(400).json({ error: 'No fields to update' });
      return;
    }

    updates.push(`updated_at = NOW()`);
    values.push(req.params.id);
    values.push(req.user!.userId);

    const result = await pool.query(
      `UPDATE notes SET ${updates.join(', ')} WHERE id = $${paramIndex++} AND user_id = $${paramIndex} RETURNING *`,
      values
    );

    if (result.rows.length === 0) {
      res.status(404).json({ error: 'Note not found' });
      return;
    }

    res.json(result.rows[0]);
  } catch (err) {
    next(err);
  }
});

// DELETE /api/notes/:id — Delete a note
router.delete('/:id', async (req: AuthRequest, res: Response, next: NextFunction) => {
  try {
    const result = await pool.query(
      'DELETE FROM notes WHERE id = $1 AND user_id = $2 RETURNING *',
      [req.params.id, req.user!.userId]
    );

    if (result.rows.length === 0) {
      res.status(404).json({ error: 'Note not found' });
      return;
    }

    res.status(204).send();
  } catch (err) {
    next(err);
  }
});

export { router as notesRouter };
```

</details>

### Reading This File Line-by-Line

The CRUD shape is the same as Level 2's projects/tasks routes — `try/catch/next(err)`, parameterized queries, dynamic `UPDATE` builder, 201/204 status codes. **What's new in Level 3 is the security pattern.** Read the new bits carefully.

```typescript
router.use(authenticate);
```

- `router.use(middleware)` registers a middleware that runs for **every** route on this router. Without this line, you'd have to add `authenticate` as an argument to every `router.get/post/...` call.
- Because this runs first, by the time any handler executes, `req.user` is guaranteed to be set (and the request is guaranteed to be authenticated). Any route below can safely use `req.user!.userId`.

```typescript
router.get('/', async (req: AuthRequest, res: Response, next: NextFunction) => {
  try {
    const result = await pool.query(
      'SELECT * FROM notes WHERE user_id = $1 ORDER BY updated_at DESC',
      [req.user!.userId]
    );
    res.json(result.rows);
  } catch (err) {
    next(err);
  }
});
```

- `req: AuthRequest` because we read `req.user`.
- `WHERE user_id = $1` and `[req.user!.userId]` — **row-level security**. We don't fetch *all* notes; we fetch only those belonging to the authenticated user. The frontend never has to filter; it can't even see what isn't returned.
- `req.user!.userId` — the `!` is the non-null assertion, safe because `authenticate` has already run.

```typescript
router.post('/', async (req: AuthRequest, ...) => {
  ...
  const result = await pool.query(
    'INSERT INTO notes (user_id, title, content) VALUES ($1, $2, $3) RETURNING *',
    [req.user!.userId, title.trim(), content || '']
  );
  ...
});
```

- The **server** decides which `user_id` to attach to the new note — using the authenticated user's ID. The client cannot send a `user_id` field; even if they tried, we ignore the body's `user_id` because the SQL hardcodes `req.user!.userId`.
- This means a malicious user can't create notes pretending to be another user.

```typescript
router.get('/:id', async (req: AuthRequest, ...) => {
  const result = await pool.query(
    'SELECT * FROM notes WHERE id = $1 AND user_id = $2',
    [req.params.id, req.user!.userId]
  );
  ...
});
```

- Two conditions in the `WHERE`: `id = $1 AND user_id = $2`. The note must match the requested ID **and** belong to the authenticated user.
- If user B requests user A's note, the query returns zero rows. We respond 404 (not 403) — see Security Exercise below.

```typescript
const updates: string[] = [];
const values: (string | number)[] = [];
let paramIndex = 1;
// ... build dynamic update ...
updates.push(`updated_at = NOW()`);
values.push(req.params.id);
values.push(req.user!.userId);

const result = await pool.query(
  `UPDATE notes SET ${updates.join(', ')} WHERE id = $${paramIndex++} AND user_id = $${paramIndex} RETURNING *`,
  values
);
```

- Same dynamic-update pattern as Level 2, but the `WHERE` now has two conditions instead of one.
- Notice we push `id` then `user_id` to `values`, and reference both with `$${paramIndex++}` and `$${paramIndex}`. The first `paramIndex++` uses the current value (e.g., 3) and increments to 4; the second `$${paramIndex}` uses the new 4. So the SQL ends up `WHERE id = $3 AND user_id = $4`.
- If user B tries to PUT to user A's note ID, the `WHERE` matches no rows → 404.

```typescript
router.delete('/:id', async (req: AuthRequest, ...) => {
  const result = await pool.query(
    'DELETE FROM notes WHERE id = $1 AND user_id = $2 RETURNING *',
    [req.params.id, req.user!.userId]
  );
  ...
});
```

Same protection on DELETE. User B cannot delete user A's note even by guessing the ID.

### 🔐 Security Exercise

Why does `WHERE id = $1 AND user_id = $2` return 404 instead of 403 when user B tries to access user A's note?

<details>
<summary>Answer</summary>

**Information leakage prevention.** If you returned 403 ("Forbidden"), the attacker knows the note exists — they just don't have access. With 404, they can't tell whether the note doesn't exist or belongs to someone else. This is a common security practice called "not revealing the existence of resources."

</details>

---

## 2. Admin Routes

<details>
<summary>See the code</summary>

Create `server/src/routes/admin.ts`:

```typescript
import { Router, Response, NextFunction } from 'express';
import pool from '../db';
import { AuthRequest } from '../types';
import { authenticate, requireAdmin } from '../middleware/authenticate';

const router = Router();

// All admin routes require authentication AND admin role
router.use(authenticate);
router.use(requireAdmin);

// GET /api/admin/stats — Usage statistics
router.get('/stats', async (_req: AuthRequest, res: Response, next: NextFunction) => {
  try {
    const usersResult = await pool.query(
      'SELECT COUNT(*) as total_users FROM users'
    );
    const notesResult = await pool.query(
      'SELECT COUNT(*) as total_notes FROM notes'
    );
    const activeResult = await pool.query(
      `SELECT u.id, u.email, COUNT(n.id) as note_count
       FROM users u
       LEFT JOIN notes n ON u.id = n.user_id
       GROUP BY u.id, u.email
       ORDER BY note_count DESC`
    );

    res.json({
      totalUsers: parseInt(usersResult.rows[0].total_users),
      totalNotes: parseInt(notesResult.rows[0].total_notes),
      userBreakdown: activeResult.rows,
    });
  } catch (err) {
    next(err);
  }
});

export { router as adminRouter };
```

</details>

### Reading This File Line-by-Line

```typescript
router.use(authenticate);
router.use(requireAdmin);
```

- **Stacked middleware.** Both run, in order, before any route in this router.
- `authenticate` first — if no token, return 401 and stop.
- `requireAdmin` second — if `req.user.role !== 'admin'`, return 403 and stop.
- Only requests that pass both checks reach the route handlers below. We get layered protection without repeating logic in every handler.

```typescript
const usersResult = await pool.query('SELECT COUNT(*) as total_users FROM users');
const notesResult = await pool.query('SELECT COUNT(*) as total_notes FROM notes');
```

- `COUNT(*)` is a SQL **aggregate function** — it counts how many rows match the query. Without `WHERE`, it counts every row in the table.
- `as total_users` aliases the result column. PostgreSQL would otherwise call it just `count`, which is less self-documenting at the call site.

```typescript
const activeResult = await pool.query(
  `SELECT u.id, u.email, COUNT(n.id) as note_count
   FROM users u
   LEFT JOIN notes n ON u.id = n.user_id
   GROUP BY u.id, u.email
   ORDER BY note_count DESC`
);
```

This is the **biggest jump in SQL complexity** in this whole curriculum. Take it slow.

- `FROM users u` — `u` is a **table alias**. We're saying "treat the `users` table as `u` for the rest of this query." Saves typing, especially in JOINs.
- `LEFT JOIN notes n ON u.id = n.user_id`
  - **JOIN** combines rows from two tables. For each `users` row, find any `notes` rows where `n.user_id = u.id`.
  - **LEFT** join means "keep every row from the left table (`users`), even if there's no matching note." A user with zero notes still appears in the result, with NULLs for the note columns. (An `INNER JOIN` would silently drop them.)
  - `n` is an alias for the `notes` table.
- `GROUP BY u.id, u.email` — collapse rows with the same user into one summary row. Required when you mix grouped columns (`u.id`, `u.email`) with aggregates (`COUNT(...)`).
- `COUNT(n.id) as note_count` — count how many notes joined for each group. Counts NULL as 0, so a user with no notes shows `note_count: 0`.
- `ORDER BY note_count DESC` — most active users first.

```typescript
res.json({
  totalUsers: parseInt(usersResult.rows[0].total_users),
  totalNotes: parseInt(notesResult.rows[0].total_notes),
  userBreakdown: activeResult.rows,
});
```

- `parseInt(string)` converts a numeric string to a number. `COUNT(*)` returns its result as a string in PostgreSQL (because counts can exceed JavaScript's safe integer range). For our scale this is overkill, but the type-correct thing is `parseInt`.

---

## 3. Server Entry Point

<details>
<summary>See the code</summary>

Create `server/src/index.ts`:

```typescript
import dotenv from 'dotenv';
dotenv.config();

import express, { Request, Response } from 'express';
import cors from 'cors';
import pool from './db';
import { authRouter } from './routes/auth';
import { notesRouter } from './routes/notes';
import { adminRouter } from './routes/admin';
import { errorHandler } from './middleware/errorHandler';

const app = express();
const PORT = process.env.PORT || 3001;
const CORS_ORIGIN = process.env.CORS_ORIGIN || 'http://localhost:5173';

app.use(express.json());
app.use(cors({ origin: CORS_ORIGIN }));

// Routes
app.use('/api/auth', authRouter);      // Public
app.use('/api/notes', notesRouter);    // Protected (authenticate middleware inside)
app.use('/api/admin', adminRouter);    // Protected + Admin

// Health check
app.get('/api/health', async (_req: Request, res: Response) => {
  try {
    const result = await pool.query('SELECT NOW()');
    res.json({
      status: 'ok',
      timestamp: new Date().toISOString(),
      database: 'connected',
      dbTime: result.rows[0].now,
    });
  } catch {
    res.status(500).json({
      status: 'error',
      timestamp: new Date().toISOString(),
      database: 'disconnected',
    });
  }
});

app.use(errorHandler);

app.listen(PORT, () => {
  console.log(`Server running on http://localhost:${PORT}`);
});
```

</details>

### Reading This File Line-by-Line

The shape is identical to Level 2's `index.ts` (load env, create app, mount middleware, mount routers, health check, error handler last). Only difference: three routers instead of one or two, and they have different protection levels.

The crucial new pattern is **mount-level protection**:

```typescript
app.use('/api/auth', authRouter);      // No global middleware — auth router decides per-route
app.use('/api/notes', notesRouter);    // Notes router calls router.use(authenticate) internally
app.use('/api/admin', adminRouter);    // Admin router stacks authenticate + requireAdmin internally
```

Each router decides its own protection. `auth.ts` mixes public (register/login) and protected (`/me`) routes, so it doesn't apply blanket middleware. `notes.ts` and `admin.ts` apply blanket middleware because every route in them is protected. This puts the security policy next to the routes it protects, instead of scattered across `index.ts`.

**Route protection map:**

```
/api/auth/register   → Public (no middleware)
/api/auth/login      → Public (no middleware)
/api/auth/me         → Protected (authenticate)
/api/notes/*         → Protected (authenticate)
/api/admin/*         → Protected (authenticate + requireAdmin)
/api/health          → Public (no middleware)
```

---

## 4. Test Cross-User Isolation

This is the most important test. Register TWO users and verify they can't see each other's notes.

**About the `Authorization: Bearer ...` header:** every protected request needs this header. The format is the literal word `Bearer`, then a space, then the token string. The auth middleware splits on the space and verifies the second part.

```bash
# Register User A
curl -X POST http://localhost:3001/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{"email": "alice@test.com", "password": "password123"}'
# Save the token as TOKEN_A

# Register User B
curl -X POST http://localhost:3001/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{"email": "bob@test.com", "password": "password456"}'
# Save the token as TOKEN_B

# Alice creates a note
curl -X POST http://localhost:3001/api/notes \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer TOKEN_A" \
  -d '{"title": "Alice secret note", "content": "Private data"}'
# Note the note ID (e.g., 1)

# Bob tries to read Alice's note
curl http://localhost:3001/api/notes/1 \
  -H "Authorization: Bearer TOKEN_B"
# Should return 404, NOT the note!

# Bob lists his notes
curl http://localhost:3001/api/notes \
  -H "Authorization: Bearer TOKEN_B"
# Should return empty array, NOT Alice's notes
```

### ✅ Checkpoint

- [ ] Users can only see their own notes
- [ ] Accessing another user's note returns 404
- [ ] Unauthenticated requests return 401
- [ ] Health check shows database connected

---

## 5. Commit

```bash
git add .
git commit -m "feat: add protected notes routes and admin stats"
```

---

> **Session Break** — Backend with authentication complete.
> When you return, you'll build the frontend in [Step 6 — Frontend](../06-frontend/).

---

| | |
|:---|---:|
| [← Step 4: Authentication](../04-auth/) | [Step 6 — Frontend →](../06-frontend/) |

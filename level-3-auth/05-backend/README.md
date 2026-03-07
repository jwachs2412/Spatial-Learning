`Level 3` **Step 5 of 8** — Building the Backend

# 05 — Building the Backend: Protected CRUD Routes

## Spatial Orientation

```
vault-note/
├── client/              ← NOT working here right now
└── server/
    └── src/
        ├── db/                    ← Already built ✓
        ├── types/                 ← Already built ✓
        ├── middleware/
        │   ├── authenticate.ts    ← Already built ✓
        │   └── errorHandler.ts    ← Already built ✓
        ├── routes/
        │   ├── auth.ts            ← Already built ✓
        │   ├── notes.ts           ← ★ Protected CRUD (we'll build this) ★
        │   └── admin.ts           ← ★ Admin stats (we'll build this) ★
        └── index.ts               ← ★ Server entry point (we'll rebuild) ★
```

**What layer are we in?** The APPLICATION layer. These routes are like Level 2's CRUD routes, with one critical addition: every route checks `req.user` to scope queries to the authenticated user. **No user can see another user's notes.**

---

## Step 1: Build the Notes Routes

**Where?** `server/src/routes/notes.ts`

Every route in this file is **protected** — the `authenticate` middleware runs first. Inside each route handler, `req.user.userId` tells us who is making the request, and we use it to scope every SQL query.

In VS Code, create `server/src/routes/notes.ts`:

```typescript
import { Router, Request, Response, NextFunction } from 'express';
import pool from '../db';
import { CreateNoteRequest, UpdateNoteRequest, JWTPayload } from '../types';
import { authenticate } from '../middleware/authenticate';

const router = Router();

// All routes in this file require authentication
router.use(authenticate);

// ─── GET /api/notes ─────────────────────────────────────────────
// List all notes for the authenticated user.
//
// The key line: WHERE user_id = $1 — scoped to THIS user only.
router.get('/', async (req: Request, res: Response, next: NextFunction) => {
  try {
    const { userId } = (req as any).user as JWTPayload;

    const result = await pool.query(
      'SELECT * FROM notes WHERE user_id = $1 ORDER BY updated_at DESC',
      [userId]
    );

    res.json(result.rows);
  } catch (err) {
    next(err);
  }
});

// ─── POST /api/notes ────────────────────────────────────────────
// Create a new note for the authenticated user.
router.post('/', async (req: Request, res: Response, next: NextFunction) => {
  try {
    const { userId } = (req as any).user as JWTPayload;
    const { title, content } = req.body as CreateNoteRequest;

    if (!title || typeof title !== 'string' || title.trim().length === 0) {
      res.status(400).json({ error: 'Note title is required' });
      return;
    }

    if (title.trim().length > 200) {
      res.status(400).json({ error: 'Title must be 200 characters or fewer' });
      return;
    }

    const result = await pool.query(
      'INSERT INTO notes (user_id, title, content) VALUES ($1, $2, $3) RETURNING *',
      [userId, title.trim(), content?.trim() || '']
    );

    res.status(201).json(result.rows[0]);
  } catch (err) {
    next(err);
  }
});

// ─── GET /api/notes/:id ─────────────────────────────────────────
// Get a single note. Must belong to the authenticated user.
router.get('/:id', async (req: Request, res: Response, next: NextFunction) => {
  try {
    const { userId } = (req as any).user as JWTPayload;
    const { id } = req.params;

    const result = await pool.query(
      'SELECT * FROM notes WHERE id = $1 AND user_id = $2',
      [id, userId]
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

// ─── PUT /api/notes/:id ─────────────────────────────────────────
// Update a note. Must belong to the authenticated user.
router.put('/:id', async (req: Request, res: Response, next: NextFunction) => {
  try {
    const { userId } = (req as any).user as JWTPayload;
    const { id } = req.params;
    const { title, content } = req.body as UpdateNoteRequest;

    if (title === undefined && content === undefined) {
      res.status(400).json({ error: 'At least one field (title or content) is required' });
      return;
    }

    if (title !== undefined && (typeof title !== 'string' || title.trim().length === 0)) {
      res.status(400).json({ error: 'Title cannot be empty' });
      return;
    }

    if (title !== undefined && title.trim().length > 200) {
      res.status(400).json({ error: 'Title must be 200 characters or fewer' });
      return;
    }

    // Build dynamic UPDATE query
    const fields: string[] = [];
    const values: (string | number)[] = [];
    let paramIndex = 1;

    if (title !== undefined) {
      fields.push(`title = $${paramIndex++}`);
      values.push(title.trim());
    }
    if (content !== undefined) {
      fields.push(`content = $${paramIndex++}`);
      values.push(content.trim());
    }
    fields.push(`updated_at = NOW()`);

    // WHERE id = $N AND user_id = $N+1
    values.push(Number(id));
    const idParam = paramIndex++;
    values.push(userId);
    const userParam = paramIndex;

    const result = await pool.query(
      `UPDATE notes SET ${fields.join(', ')} WHERE id = $${idParam} AND user_id = $${userParam} RETURNING *`,
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

// ─── DELETE /api/notes/:id ──────────────────────────────────────
// Delete a note. Must belong to the authenticated user.
router.delete('/:id', async (req: Request, res: Response, next: NextFunction) => {
  try {
    const { userId } = (req as any).user as JWTPayload;
    const { id } = req.params;

    const result = await pool.query(
      'DELETE FROM notes WHERE id = $1 AND user_id = $2 RETURNING *',
      [id, userId]
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

### The Critical Pattern: `AND user_id = $2`

Every query includes `AND user_id = $2`. This is authorization at the database level:

```sql
-- Level 2 (TaskForge) — anyone can read any project:
SELECT * FROM projects WHERE id = $1

-- Level 3 (VaultNote) — only the owner can read their note:
SELECT * FROM notes WHERE id = $1 AND user_id = $2
```

Even if someone guesses or iterates through note IDs (1, 2, 3, ...), they can only see notes where `user_id` matches their authenticated user ID. Someone else's note returns "Not found" — not "Forbidden." This is intentional: we don't even confirm the note exists if it's not yours.

```
USER A (userId: 1):
  GET /api/notes/5  →  SELECT * FROM notes WHERE id = 5 AND user_id = 1
                       → Note 5 belongs to user 1? YES → return it
                       → Note 5 belongs to user 2? NO  → "Not found"

USER B (userId: 2):
  GET /api/notes/5  →  SELECT * FROM notes WHERE id = 5 AND user_id = 2
                       → Note 5 belongs to user 2? NO  → "Not found"
```

### `router.use(authenticate)`

```typescript
router.use(authenticate);
```

This applies the `authenticate` middleware to **every route** in this file. Instead of writing `authenticate` in each route, we apply it once at the top. Any request that reaches these routes has already been verified.

---

> [!TIP]
> **Session Break** — You've built the notes CRUD routes with per-user data isolation. Save your work and take a break.
> When you return, you'll build the admin routes and wire everything into the server.

---

## Step 2: Build the Admin Routes

**Where?** `server/src/routes/admin.ts`

A simple admin endpoint that returns usage statistics. Requires both authentication AND the admin role.

```typescript
import { Router, Request, Response, NextFunction } from 'express';
import pool from '../db';
import { authenticate, requireAdmin } from '../middleware/authenticate';

const router = Router();

// Both middleware run in order: first authenticate, then check admin role
router.use(authenticate);
router.use(requireAdmin);

// ─── GET /api/admin/stats ───────────────────────────────────────
// Returns usage statistics. Admin only.
router.get('/stats', async (_req: Request, res: Response, next: NextFunction) => {
  try {
    const userCount = await pool.query('SELECT COUNT(*) FROM users');
    const noteCount = await pool.query('SELECT COUNT(*) FROM notes');
    const notesPerUser = await pool.query(`
      SELECT u.email, COUNT(n.id) as note_count
      FROM users u
      LEFT JOIN notes n ON u.id = n.user_id
      GROUP BY u.id, u.email
      ORDER BY note_count DESC
    `);

    res.json({
      totalUsers: parseInt(userCount.rows[0].count),
      totalNotes: parseInt(noteCount.rows[0].count),
      notesPerUser: notesPerUser.rows,
    });
  } catch (err) {
    next(err);
  }
});

export { router as adminRouter };
```

### Middleware Chaining

```typescript
router.use(authenticate);   // Step 1: Is there a valid token?
router.use(requireAdmin);   // Step 2: Is the user an admin?
```

These run in order. If `authenticate` rejects (no token), `requireAdmin` never runs. If `authenticate` passes but the role isn't admin, `requireAdmin` sends 403 Forbidden.

```
Request → authenticate → requireAdmin → Route Handler
            │                │
            │ No token?      │ Not admin?
            ▼                ▼
         401 Unauthorized   403 Forbidden
```

### SQL JOIN Explained

```sql
SELECT u.email, COUNT(n.id) as note_count
FROM users u
LEFT JOIN notes n ON u.id = n.user_id
GROUP BY u.id, u.email
ORDER BY note_count DESC
```

> [!NOTE]
> **Technical**: A LEFT JOIN combines rows from two tables. For each user, it finds all matching notes. `LEFT` ensures users with zero notes still appear (with `note_count = 0`). `GROUP BY` collapses multiple note rows per user into a single row with a count.

> [!NOTE]
> **Plain English**: "For each user, count how many notes they have." The LEFT JOIN means users with no notes still show up (with count 0) instead of being omitted.

---

## Step 3: Build the Server Entry Point

**Where?** `server/src/index.ts`

Replace the temporary test file with the full server:

```typescript
import dotenv from 'dotenv';
dotenv.config();

import express from 'express';
import cors from 'cors';
import pool from './db';
import { authRouter } from './routes/auth';
import { notesRouter } from './routes/notes';
import { adminRouter } from './routes/admin';
import { errorHandler } from './middleware/errorHandler';

const app = express();
const PORT = process.env.PORT || 3001;

// ─── MIDDLEWARE ──────────────────────────────────────────────────
app.use(express.json());
app.use(cors({
  origin: process.env.CORS_ORIGIN || 'http://localhost:5173',
}));

// ─── ROUTES ─────────────────────────────────────────────────────
app.use('/api/auth', authRouter);        // Public: register, login
app.use('/api/notes', notesRouter);      // Protected: CRUD for notes
app.use('/api/admin', adminRouter);      // Protected + Admin: stats

// ─── HEALTH CHECK ───────────────────────────────────────────────
app.get('/api/health', async (_req, res) => {
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

// ─── ERROR HANDLER ──────────────────────────────────────────────
app.use(errorHandler);

// ─── START SERVER ───────────────────────────────────────────────
app.listen(PORT, () => {
  console.log(`Server running on http://localhost:${PORT}`);
});
```

### Route Protection Map

```
/api/auth/register    → PUBLIC (no middleware)
/api/auth/login       → PUBLIC (no middleware)
/api/auth/me          → PROTECTED (authenticate in route)
/api/notes/*          → PROTECTED (authenticate via router.use)
/api/admin/*          → PROTECTED + ADMIN (authenticate + requireAdmin)
/api/health           → PUBLIC (no middleware)
```

---

## Step 4: Test Every Endpoint

Start the server:

```bash
cd server && npm run dev
```

In a second terminal:

### Register a User

```bash
curl -X POST http://localhost:3001/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{"email":"alice@example.com","password":"password123"}'
```

**Save the token from the response** — you'll use it for all subsequent requests.

### Create Notes (Authenticated)

```bash
curl -X POST http://localhost:3001/api/notes \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -d '{"title":"My first note","content":"This is private to me."}'
```

```bash
curl -X POST http://localhost:3001/api/notes \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -d '{"title":"Shopping list","content":"Milk, eggs, bread"}'
```

### List Notes

```bash
curl http://localhost:3001/api/notes \
  -H "Authorization: Bearer YOUR_TOKEN"
```

Expected: Array of Alice's notes only.

### Register a Second User

```bash
curl -X POST http://localhost:3001/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{"email":"bob@example.com","password":"password456"}'
```

Save Bob's token.

### Try to See Alice's Notes as Bob

```bash
curl http://localhost:3001/api/notes \
  -H "Authorization: Bearer BOBS_TOKEN"
```

Expected: `[]` — Bob has no notes. He cannot see Alice's.

### Try Without a Token

```bash
curl http://localhost:3001/api/notes
```

Expected: `{"error":"No token provided"}`

### Update a Note

```bash
curl -X PUT http://localhost:3001/api/notes/1 \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -d '{"content":"Updated content for my first note."}'
```

### Delete a Note

```bash
curl -X DELETE http://localhost:3001/api/notes/2 \
  -H "Authorization: Bearer YOUR_TOKEN"
```

### The Key Test: Cross-User Isolation

Using Bob's token, try to access Alice's note:

```bash
curl http://localhost:3001/api/notes/1 \
  -H "Authorization: Bearer BOBS_TOKEN"
```

Expected: `{"error":"Note not found"}` — Bob can't see Alice's note.

Stop the server:

```bash
cd ..
```

---

## Step 5: Commit

> [!IMPORTANT]
> **You should be in:** `vault-note/`

```bash
git add server/src/
git commit -m "feat: add protected notes CRUD and admin stats routes"
```

---

```
┌──────────────────────────────────────────────────────────────────┐
│                    WHAT EXISTS NOW                                 │
│                                                                  │
│   ┌──────────────┐    ┌──────────────────┐    ┌──────────────┐ │
│   │  FRONTEND     │    │  BACKEND ✓       │    │  DATABASE ✓  │ │
│   │  (not built)  │    │                  │    │              │ │
│   │              │    │  POST register   │    │  users       │ │
│   │              │    │  POST login      │───▶│  notes       │ │
│   │              │    │  GET  /me        │    │              │ │
│   │  NEXT STEP   │    │  GET  /notes     │    │  Passwords   │ │
│   │              │    │  POST /notes     │    │  are hashed  │ │
│   │              │    │  GET  /notes/:id │    │              │ │
│   │              │    │  PUT  /notes/:id │    │  Notes are   │ │
│   │              │    │  DEL  /notes/:id │    │  user-scoped │ │
│   │              │    │  GET  /admin/stats│    │              │ │
│   └──────────────┘    └──────────────────┘    └──────────────┘ │
│                                                                  │
│   Backend is complete with auth + protected CRUD.               │
│   Next: build the React frontend with routing and auth UI.      │
└──────────────────────────────────────────────────────────────────┘
```

---

> [!TIP]
> ## Spatial Check-In

1. **Why does every notes query include `AND user_id = $2`?**

<details><summary>Answer</summary>

To enforce authorization at the database level. Even if someone has a valid token for user 2, they can't access notes belonging to user 1 because the query only returns rows matching their user ID.

</details>

2. **What does `router.use(authenticate)` at the top of the notes file do?**

<details><summary>Answer</summary>

Applies the authentication middleware to every route in that file. Any request reaching those routes has already had its JWT verified and `req.user` populated.

</details>

3. **Why does accessing someone else's note return 404 (not 403)?**

<details><summary>Answer</summary>

Returning 403 would confirm the note exists. Returning 404 gives no information — the attacker can't tell if the note doesn't exist or if it belongs to someone else.

</details>

---

| | | |
|:---|:---:|---:|
| [← 04 — Authentication](../04-auth/) | [Level 3 Overview](../) | [06 — Building the Frontend →](../06-frontend/) |

`Level 2` **Step 4 of 7** — Building the Backend

# 04 — Building the Backend: Full CRUD API

## Spatial Orientation

```
task-forge/
├── client/              ← NOT working here right now
└── server/
    └── src/
        ├── db/
        │   ├── schema.sql       ← Already built ✓
        │   └── index.ts         ← Already built ✓
        ├── types/
        │   └── index.ts         ← Type definitions (we'll build this)
        ├── middleware/
        │   └── errorHandler.ts  ← Error middleware (we'll build this)
        ├── routes/
        │   ├── projects.ts      ← Project routes (we'll build this)
        │   └── tasks.ts         ← Task routes (we'll build this)
        └── index.ts             ← Server entry point (we'll rebuild this)
```

**What layer are we in?** The APPLICATION layer. This code receives HTTP requests from the frontend, translates them into SQL queries, sends those queries to PostgreSQL, and sends the results back as HTTP responses.

---

## Step 1: Define Types

**Where?** `server/src/types/index.ts`

In VS Code, create a new file at `server/src/types/index.ts`.

```typescript
// ─── DATABASE MODELS ────────────────────────────────────────────
// These match the rows in our database tables exactly.

export interface Project {
  id: number;
  name: string;
  description: string;
  created_at: string;
  updated_at: string;
}

export interface Task {
  id: number;
  project_id: number;
  title: string;
  completed: boolean;
  created_at: string;
  updated_at: string;
}

// ─── REQUEST TYPES ──────────────────────────────────────────────
// These define the shape of data the client sends.
// Notice: no id, created_at, updated_at — the server/database assigns those.

export interface CreateProjectRequest {
  name: string;
  description?: string;
}

export interface UpdateProjectRequest {
  name?: string;
  description?: string;
}

export interface CreateTaskRequest {
  title: string;
}

export interface UpdateTaskRequest {
  title?: string;
  completed?: boolean;
}

// ─── RESPONSE TYPES ─────────────────────────────────────────────
// A project with its tasks included (for the detail view).

export interface ProjectWithTasks extends Project {
  tasks: Task[];
}
```

**Why separate request and model types?**
- `Project` has `id`, `created_at`, `updated_at` — assigned by the database
- `CreateProjectRequest` has only `name` and `description` — what the client sends
- `UpdateProjectRequest` has everything optional (`?`) — the client only sends what changed

This mirrors the Level 1 pattern (`MoodEntry` vs `CreateEntryRequest`) but with more types because we have more operations.

---

## Step 2: Build Error Handling Middleware

**Where?** `server/src/middleware/errorHandler.ts`

In Level 1, errors were handled inline in each route. That works for 2 routes but gets messy with 10. Error handling middleware catches errors from all routes in one place.

In VS Code, create a new file at `server/src/middleware/errorHandler.ts`.

```typescript
import { Request, Response, NextFunction } from 'express';

/**
 * Express error handling middleware.
 *
 * This function has FOUR parameters — that's how Express knows it's
 * an error handler, not a regular middleware. When any route calls
 * next(error), Express skips all regular middleware and jumps here.
 */
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

### How Error Middleware Works

```
NORMAL FLOW:                        ERROR FLOW:
─────────────                       ────────────

Request                             Request
   │                                   │
   ▼                                   ▼
express.json()                      express.json()
   │                                   │
   ▼                                   ▼
cors()                              cors()
   │                                   │
   ▼                                   ▼
Route Handler                       Route Handler
   │                                   │
   ▼                                   │ throw error / next(error)
Response sent                          │
                                       ▼
                                   errorHandler()  ← FOUR parameters
                                       │
                                       ▼
                                   Error response sent
```

> [!NOTE]
> **Technical**: Express identifies error-handling middleware by its four-parameter signature `(err, req, res, next)`. When `next(error)` is called in any route or middleware, Express skips the normal middleware chain and invokes the first four-parameter middleware it finds.

> [!NOTE]
> **Plain English**: Normal middleware has 3 parameters (req, res, next). Error middleware has 4 (err, req, res, next). Express uses this difference to know which is which. When something goes wrong in a route and you call `next(error)`, Express skips everything else and jumps straight to the error handler.

**Why this is better than try/catch in every route**: You still use try/catch in routes, but if you forget one or an unexpected error happens, the middleware catches it. It's a safety net.

---

## Step 3: Build Project Routes

**Where?** `server/src/routes/projects.ts`

This file handles all 5 project endpoints. Each route follows the same pattern:

1. Extract data from the request
2. Validate the data
3. Send a SQL query to PostgreSQL
4. Send the result back as JSON

In VS Code, create a new file at `server/src/routes/projects.ts`.

```typescript
import { Router, Request, Response, NextFunction } from 'express';
import pool from '../db';
import { CreateProjectRequest, UpdateProjectRequest } from '../types';

const router = Router();

// ─── GET /api/projects ──────────────────────────────────────────
// List all projects, newest first.
//
// SQL: SELECT * FROM projects ORDER BY created_at DESC
// Response: 200 OK, [{ id, name, description, created_at, updated_at }, ...]
router.get('/', async (_req: Request, res: Response, next: NextFunction) => {
  try {
    const result = await pool.query(
      'SELECT * FROM projects ORDER BY created_at DESC'
    );
    res.json(result.rows);
  } catch (err) {
    next(err);
  }
});

// ─── POST /api/projects ─────────────────────────────────────────
// Create a new project.
//
// Body: { name: "My Project", description: "Optional description" }
// SQL: INSERT INTO projects (name, description) VALUES ($1, $2) RETURNING *
// Response: 201 Created, { id, name, description, created_at, updated_at }
router.post('/', async (req: Request, res: Response, next: NextFunction) => {
  try {
    const { name, description } = req.body as CreateProjectRequest;

    // ─── VALIDATION ───────────────────────────────────────────
    if (!name || typeof name !== 'string' || name.trim().length === 0) {
      res.status(400).json({ error: 'Project name is required' });
      return;
    }

    if (name.trim().length > 100) {
      res.status(400).json({ error: 'Project name must be 100 characters or fewer' });
      return;
    }

    const result = await pool.query(
      'INSERT INTO projects (name, description) VALUES ($1, $2) RETURNING *',
      [name.trim(), description?.trim() || '']
    );

    res.status(201).json(result.rows[0]);
  } catch (err) {
    next(err);
  }
});

// ─── GET /api/projects/:id ──────────────────────────────────────
// Get a single project with all its tasks.
//
// SQL: Two queries — one for the project, one for its tasks.
// Response: 200 OK, { id, name, description, ..., tasks: [...] }
router.get('/:id', async (req: Request, res: Response, next: NextFunction) => {
  try {
    const { id } = req.params;

    const projectResult = await pool.query(
      'SELECT * FROM projects WHERE id = $1',
      [id]
    );

    if (projectResult.rows.length === 0) {
      res.status(404).json({ error: 'Project not found' });
      return;
    }

    const tasksResult = await pool.query(
      'SELECT * FROM tasks WHERE project_id = $1 ORDER BY created_at ASC',
      [id]
    );

    res.json({
      ...projectResult.rows[0],
      tasks: tasksResult.rows,
    });
  } catch (err) {
    next(err);
  }
});

// ─── PUT /api/projects/:id ──────────────────────────────────────
// Update a project's name and/or description.
//
// Body: { name?: "New Name", description?: "New description" }
// SQL: UPDATE projects SET ... WHERE id = $1 RETURNING *
// Response: 200 OK, { id, name, description, created_at, updated_at }
router.put('/:id', async (req: Request, res: Response, next: NextFunction) => {
  try {
    const { id } = req.params;
    const { name, description } = req.body as UpdateProjectRequest;

    // Validate: at least one field must be provided
    if (name === undefined && description === undefined) {
      res.status(400).json({ error: 'At least one field (name or description) is required' });
      return;
    }

    if (name !== undefined && (typeof name !== 'string' || name.trim().length === 0)) {
      res.status(400).json({ error: 'Project name cannot be empty' });
      return;
    }

    if (name !== undefined && name.trim().length > 100) {
      res.status(400).json({ error: 'Project name must be 100 characters or fewer' });
      return;
    }

    // Build the UPDATE query dynamically based on which fields were sent
    const fields: string[] = [];
    const values: (string | number)[] = [];
    let paramIndex = 1;

    if (name !== undefined) {
      fields.push(`name = $${paramIndex++}`);
      values.push(name.trim());
    }
    if (description !== undefined) {
      fields.push(`description = $${paramIndex++}`);
      values.push(description.trim());
    }
    fields.push(`updated_at = NOW()`);

    values.push(Number(id));

    const result = await pool.query(
      `UPDATE projects SET ${fields.join(', ')} WHERE id = $${paramIndex} RETURNING *`,
      values
    );

    if (result.rows.length === 0) {
      res.status(404).json({ error: 'Project not found' });
      return;
    }

    res.json(result.rows[0]);
  } catch (err) {
    next(err);
  }
});

// ─── DELETE /api/projects/:id ───────────────────────────────────
// Delete a project and all its tasks (CASCADE handles tasks).
//
// SQL: DELETE FROM projects WHERE id = $1
// Response: 204 No Content
router.delete('/:id', async (req: Request, res: Response, next: NextFunction) => {
  try {
    const { id } = req.params;

    const result = await pool.query(
      'DELETE FROM projects WHERE id = $1 RETURNING *',
      [id]
    );

    if (result.rows.length === 0) {
      res.status(404).json({ error: 'Project not found' });
      return;
    }

    // 204 = "No Content" — the standard status for successful deletion
    res.status(204).send();
  } catch (err) {
    next(err);
  }
});

export { router as projectsRouter };
```

### Key Patterns Explained

**Parameterized Queries (`$1`, `$2`)**

```typescript
await pool.query(
  'INSERT INTO projects (name, description) VALUES ($1, $2) RETURNING *',
  [name.trim(), description?.trim() || '']
);
```

> [!WARNING]
> **Never build SQL by concatenating strings.** This code is vulnerable to SQL injection:
> ```typescript
> // NEVER DO THIS:
> await pool.query(`INSERT INTO projects (name) VALUES ('${name}')`);
> // If name = "'; DROP TABLE projects; --" your database is destroyed
> ```
> Always use parameterized queries (`$1`, `$2`). The `pg` library automatically escapes the values, making injection impossible.

**The `$1`, `$2` placeholders** are replaced by the values in the array (second argument), in order. PostgreSQL handles escaping. This is the single most important security practice in database programming.

**try/catch + next(error)**

Every route wraps its logic in try/catch. If the SQL query fails (connection error, constraint violation, etc.), the error is caught and passed to `next(error)`, which invokes our error handler middleware.

```typescript
try {
  // ... route logic with SQL queries
} catch (err) {
  next(err);  // → jumps to errorHandler middleware
}
```

**RETURNING \***

After INSERT, UPDATE, or DELETE, `RETURNING *` tells PostgreSQL to send back the affected row. Without it, we'd need a separate SELECT query to get the data we just changed.

**204 No Content**

For DELETE operations, the standard response is 204 (No Content) — "I did what you asked, and there's nothing to send back." Note: `res.status(204).send()` — not `.json()`, because there's no body.

---

## Step 4: Build Task Routes

**Where?** `server/src/routes/tasks.ts`

In VS Code, create a new file at `server/src/routes/tasks.ts`.

```typescript
import { Router, Request, Response, NextFunction } from 'express';
import pool from '../db';
import { CreateTaskRequest, UpdateTaskRequest } from '../types';

const router = Router();

// ─── GET /api/projects/:projectId/tasks ─────────────────────────
// List all tasks for a specific project.
//
// SQL: SELECT * FROM tasks WHERE project_id = $1 ORDER BY created_at ASC
// Response: 200 OK, [{ id, project_id, title, completed, ... }, ...]
router.get(
  '/projects/:projectId/tasks',
  async (req: Request, res: Response, next: NextFunction) => {
    try {
      const { projectId } = req.params;

      // Verify the project exists
      const projectResult = await pool.query(
        'SELECT id FROM projects WHERE id = $1',
        [projectId]
      );

      if (projectResult.rows.length === 0) {
        res.status(404).json({ error: 'Project not found' });
        return;
      }

      const result = await pool.query(
        'SELECT * FROM tasks WHERE project_id = $1 ORDER BY created_at ASC',
        [projectId]
      );

      res.json(result.rows);
    } catch (err) {
      next(err);
    }
  }
);

// ─── POST /api/projects/:projectId/tasks ────────────────────────
// Create a new task in a specific project.
//
// Body: { title: "My new task" }
// SQL: INSERT INTO tasks (project_id, title) VALUES ($1, $2) RETURNING *
// Response: 201 Created, { id, project_id, title, completed, ... }
router.post(
  '/projects/:projectId/tasks',
  async (req: Request, res: Response, next: NextFunction) => {
    try {
      const { projectId } = req.params;
      const { title } = req.body as CreateTaskRequest;

      // Validate
      if (!title || typeof title !== 'string' || title.trim().length === 0) {
        res.status(400).json({ error: 'Task title is required' });
        return;
      }

      if (title.trim().length > 200) {
        res.status(400).json({ error: 'Task title must be 200 characters or fewer' });
        return;
      }

      // Verify the project exists
      const projectResult = await pool.query(
        'SELECT id FROM projects WHERE id = $1',
        [projectId]
      );

      if (projectResult.rows.length === 0) {
        res.status(404).json({ error: 'Project not found' });
        return;
      }

      const result = await pool.query(
        'INSERT INTO tasks (project_id, title) VALUES ($1, $2) RETURNING *',
        [projectId, title.trim()]
      );

      res.status(201).json(result.rows[0]);
    } catch (err) {
      next(err);
    }
  }
);

// ─── PUT /api/tasks/:id ─────────────────────────────────────────
// Update a task (toggle completed, rename, etc.).
//
// Body: { title?: "New title", completed?: true }
// SQL: UPDATE tasks SET ... WHERE id = $1 RETURNING *
// Response: 200 OK, { id, project_id, title, completed, ... }
router.put(
  '/tasks/:id',
  async (req: Request, res: Response, next: NextFunction) => {
    try {
      const { id } = req.params;
      const { title, completed } = req.body as UpdateTaskRequest;

      // Validate: at least one field must be provided
      if (title === undefined && completed === undefined) {
        res.status(400).json({ error: 'At least one field (title or completed) is required' });
        return;
      }

      if (title !== undefined && (typeof title !== 'string' || title.trim().length === 0)) {
        res.status(400).json({ error: 'Task title cannot be empty' });
        return;
      }

      if (title !== undefined && title.trim().length > 200) {
        res.status(400).json({ error: 'Task title must be 200 characters or fewer' });
        return;
      }

      if (completed !== undefined && typeof completed !== 'boolean') {
        res.status(400).json({ error: 'Completed must be a boolean' });
        return;
      }

      // Build dynamic UPDATE query
      const fields: string[] = [];
      const values: (string | number | boolean)[] = [];
      let paramIndex = 1;

      if (title !== undefined) {
        fields.push(`title = $${paramIndex++}`);
        values.push(title.trim());
      }
      if (completed !== undefined) {
        fields.push(`completed = $${paramIndex++}`);
        values.push(completed);
      }
      fields.push(`updated_at = NOW()`);

      values.push(Number(id));

      const result = await pool.query(
        `UPDATE tasks SET ${fields.join(', ')} WHERE id = $${paramIndex} RETURNING *`,
        values
      );

      if (result.rows.length === 0) {
        res.status(404).json({ error: 'Task not found' });
        return;
      }

      res.json(result.rows[0]);
    } catch (err) {
      next(err);
    }
  }
);

// ─── DELETE /api/tasks/:id ──────────────────────────────────────
// Delete a single task.
//
// SQL: DELETE FROM tasks WHERE id = $1
// Response: 204 No Content
router.delete(
  '/tasks/:id',
  async (req: Request, res: Response, next: NextFunction) => {
    try {
      const { id } = req.params;

      const result = await pool.query(
        'DELETE FROM tasks WHERE id = $1 RETURNING *',
        [id]
      );

      if (result.rows.length === 0) {
        res.status(404).json({ error: 'Task not found' });
        return;
      }

      res.status(204).send();
    } catch (err) {
      next(err);
    }
  }
);

export { router as tasksRouter };
```

### Nested Routes Pattern

Notice something about the task routes: some use `/projects/:projectId/tasks` and others use `/tasks/:id`.

| Endpoint | Why This Pattern |
|----------|-----------------|
| `GET /projects/:projectId/tasks` | "Get tasks for this project" — you need the project context |
| `POST /projects/:projectId/tasks` | "Create a task in this project" — you need the project to link to |
| `PUT /tasks/:id` | "Update this task" — the task ID is enough, no project needed |
| `DELETE /tasks/:id` | "Delete this task" — the task ID is enough |

This is a common REST pattern: use nested routes when you need the parent context, flat routes when the resource ID is sufficient.

---

## Step 5: Build the Server Entry Point

**Where?** `server/src/index.ts`

Replace the test file we created earlier with the real server. Open the existing `server/src/index.ts` and replace its contents:

```typescript
import dotenv from 'dotenv';
dotenv.config();

import express from 'express';
import cors from 'cors';
import pool from './db';
import { projectsRouter } from './routes/projects';
import { tasksRouter } from './routes/tasks';
import { errorHandler } from './middleware/errorHandler';

const app = express();
const PORT = process.env.PORT || 3001;

// ─── MIDDLEWARE ──────────────────────────────────────────────────
app.use(express.json());
app.use(cors({
  origin: process.env.CORS_ORIGIN || 'http://localhost:5173',
}));

// ─── ROUTES ─────────────────────────────────────────────────────
app.use('/api/projects', projectsRouter);
app.use('/api', tasksRouter);

// ─── HEALTH CHECK ───────────────────────────────────────────────
// Enhanced from Level 1: now checks database connectivity too.
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
// Must be registered AFTER all routes.
// Express knows it's an error handler because it has 4 parameters.
app.use(errorHandler);

// ─── START SERVER ───────────────────────────────────────────────
app.listen(PORT, () => {
  console.log(`Server running on http://localhost:${PORT}`);
});
```

### What Changed From Level 1

| Level 1 | Level 2 | Why |
|---------|---------|-----|
| `dotenv` not used | `dotenv.config()` at top | Loads DATABASE_URL from .env |
| 1 route file | 2 route files | Projects + Tasks |
| No error middleware | `errorHandler` at bottom | Catches errors from all routes |
| Health check returns `{status: 'ok'}` | Health check pings database | Verifies full stack is working |

**Why `dotenv.config()` is the very first thing**: The database pool (imported from `./db`) reads `process.env.DATABASE_URL` when it's created. If dotenv hasn't loaded yet, the URL is undefined. Order matters.

**Why error handler is registered last**: Express runs middleware in order. The error handler must come after routes so it can catch their errors.

### Route Mounting Explained

```typescript
app.use('/api/projects', projectsRouter);
app.use('/api', tasksRouter);
```

- `projectsRouter` is mounted at `/api/projects`. Inside `projectsRouter`, routes use `/` and `/:id`. Express combines them: `/api/projects` + `/:id` = `/api/projects/:id`.

- `tasksRouter` is mounted at `/api`. Inside `tasksRouter`, routes use `/projects/:projectId/tasks` and `/tasks/:id`. Express combines them: `/api` + `/projects/:projectId/tasks` = `/api/projects/:projectId/tasks`.

```
Mounted at          +  Route definition         =  Full URL
────────────           ────────────────             ────────
/api/projects       +  /                         =  /api/projects
/api/projects       +  /:id                      =  /api/projects/:id
/api                +  /projects/:projectId/tasks =  /api/projects/:projectId/tasks
/api                +  /tasks/:id                =  /api/tasks/:id
```

---

## Step 6: Test Every Endpoint

Start the server:

> [!IMPORTANT]
> **You should be in:** `task-forge/`

```bash
cd server
```

> [!IMPORTANT]
> **You are now in:** `task-forge/server/`

```bash
npm run dev
```

You should see:

```
Connected to PostgreSQL
Server running on http://localhost:3001
```

Open a **second terminal**. Test each endpoint with curl:

### Health Check

```bash
curl http://localhost:3001/api/health
```

Expected: `{"status":"ok","timestamp":"...","database":"connected","dbTime":"..."}`

### Create Projects

```bash
curl -X POST http://localhost:3001/api/projects \
  -H "Content-Type: application/json" \
  -d '{"name":"Personal Website","description":"Build my portfolio site"}'
```

Expected: `{"id":1,"name":"Personal Website","description":"Build my portfolio site","created_at":"...","updated_at":"..."}`

```bash
curl -X POST http://localhost:3001/api/projects \
  -H "Content-Type: application/json" \
  -d '{"name":"TaskForge","description":"The app we are building"}'
```

### List Projects

```bash
curl http://localhost:3001/api/projects
```

Expected: Array with both projects.

### Get Project With Tasks

```bash
curl http://localhost:3001/api/projects/1
```

Expected: Project 1 with `tasks: []` (no tasks yet).

### Create Tasks

```bash
curl -X POST http://localhost:3001/api/projects/1/tasks \
  -H "Content-Type: application/json" \
  -d '{"title":"Design homepage layout"}'
```

```bash
curl -X POST http://localhost:3001/api/projects/1/tasks \
  -H "Content-Type: application/json" \
  -d '{"title":"Choose color scheme"}'
```

### List Tasks for Project

```bash
curl http://localhost:3001/api/projects/1/tasks
```

Expected: Array with both tasks.

### Update Task (Toggle Completed)

```bash
curl -X PUT http://localhost:3001/api/tasks/1 \
  -H "Content-Type: application/json" \
  -d '{"completed":true}'
```

Expected: Task with `completed: true`.

### Update Project

```bash
curl -X PUT http://localhost:3001/api/projects/1 \
  -H "Content-Type: application/json" \
  -d '{"name":"Portfolio Website"}'
```

### Delete Task

```bash
curl -X DELETE http://localhost:3001/api/tasks/2
```

Expected: Empty response (204 No Content).

### Delete Project (Cascades to Tasks)

```bash
curl -X DELETE http://localhost:3001/api/projects/1
```

Expected: Empty response (204). All tasks for project 1 are also deleted.

### Test Validation

```bash
# Missing name
curl -X POST http://localhost:3001/api/projects \
  -H "Content-Type: application/json" \
  -d '{"description":"No name provided"}'
```

Expected: `{"error":"Project name is required"}`

```bash
# Nonexistent project
curl http://localhost:3001/api/projects/999
```

Expected: `{"error":"Project not found"}`

### The Key Test: Persistence

**Stop the server** (`Ctrl+C`), then **start it again** (`npm run dev`).

```bash
curl http://localhost:3001/api/projects
```

**Your data is still there.** This is the fundamental difference from Level 1. The in-memory array is gone — data lives in PostgreSQL now.

Stop the server and go back to the project root:

```bash
cd ..
```

> [!IMPORTANT]
> **You are now in:** `task-forge/` (back to the project root)

---

## Step 7: Commit

> [!IMPORTANT]
> **You should be in:** `task-forge/`

```bash
git add server/src/
git commit -m "feat: add full CRUD API with project and task routes"
```

---

## Spatial Check-In

```
┌──────────────────────────────────────────────────────────────────┐
│                    WHAT EXISTS NOW                                 │
│                                                                  │
│   ┌──────────────┐    ┌──────────────────┐    ┌──────────────┐ │
│   │  FRONTEND     │    │  BACKEND ✓       │    │  DATABASE ✓  │ │
│   │  (not built)  │    │                  │    │              │ │
│   │              │    │  GET  /projects   │    │  projects    │ │
│   │              │    │  POST /projects   │───▶│  tasks       │ │
│   │              │    │  GET  /projects/:id│   │              │ │
│   │              │    │  PUT  /projects/:id│   │  Data        │ │
│   │  NEXT STEP   │    │  DEL  /projects/:id│   │  persists    │ │
│   │              │    │  GET  /tasks      │    │  across      │ │
│   │              │    │  POST /tasks      │    │  restarts ✓  │ │
│   │              │    │  PUT  /tasks/:id  │    │              │ │
│   │              │    │  DEL  /tasks/:id  │    │              │ │
│   │              │    │  GET  /health     │    │              │ │
│   └──────────────┘    └──────────────────┘    └──────────────┘ │
│                                                                  │
│   The backend is complete with full CRUD. Data persists.        │
│   Next: build the React frontend to provide a user interface.   │
└──────────────────────────────────────────────────────────────────┘
```

> [!TIP]
> **Spatial Check-In** — Questions to answer before moving on.

1. **What happens when you use string concatenation instead of parameterized queries?**

<details><summary>Answer</summary>

SQL injection — an attacker can manipulate the query to read, modify, or delete data. Parameterized queries (`$1`, `$2`) prevent this by escaping values automatically.

</details>

2. **Why does the error handler have four parameters?**

<details><summary>Answer</summary>

Express identifies error-handling middleware by its four-parameter signature `(err, req, res, next)`. This is how Express distinguishes it from regular middleware.

</details>

3. **What does a 204 status code mean?**

<details><summary>Answer</summary>

"No Content" — the request succeeded but there's nothing to send back. Used for DELETE operations.

</details>

4. **Why is `dotenv.config()` the first line in the server entry point?**

<details><summary>Answer</summary>

The database pool reads `process.env.DATABASE_URL` when it's imported. If dotenv hasn't loaded the `.env` file yet, the URL is undefined and the connection fails.

</details>

5. **What is the difference between `GET /api/projects/1` and `GET /api/projects/1/tasks`?**

<details><summary>Answer</summary>

`GET /api/projects/1` returns the project object with all its tasks embedded. `GET /api/projects/1/tasks` returns only the array of tasks for that project.

</details>

---

| | | |
|:---|:---:|---:|
| [← 03 — Database Design](../03-database/) | [Level 2 Overview](../) | [05 — Building the Frontend →](../05-frontend/) |

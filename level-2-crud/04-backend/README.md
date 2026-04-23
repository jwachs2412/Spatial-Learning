# Step 4 — Building the Backend: Full CRUD API

## Spatial Orientation

```
task-forge/
└── server/
    └── src/
        ├── db/
        │   ├── schema.sql       ← Built in Step 3
        │   └── index.ts         ← Built in Step 3
        ├── types/
        │   └── index.ts         ← YOU BUILD THIS
        ├── middleware/
        │   └── errorHandler.ts  ← YOU BUILD THIS
        ├── routes/
        │   ├── projects.ts      ← YOU BUILD THIS (5 endpoints)
        │   └── tasks.ts         ← YOU BUILD THIS (4 endpoints)
        └── index.ts             ← YOU BUILD THIS
```

---

## 1. Define Types

### 🏗️ Your Turn

Define TypeScript interfaces that match your database tables. Think about what fields each table has.

<details>
<summary>See the solution</summary>

Create `server/src/types/index.ts`:

```typescript
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

export interface ProjectWithTasks extends Project {
  tasks: Task[];
}

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
```

</details>

**Key pattern:** `CreateProjectRequest` only has the fields the client sends. `Project` has all fields including database-generated ones (`id`, `created_at`, etc.). This distinction keeps your API contract clear.

---

## 2. Build the Error Handler

> **Key Concept: Error Middleware**
> Express error middleware has a special signature: **four parameters** (`err, req, res, next`). Express recognizes four-parameter middleware as error handlers and routes thrown errors to them automatically. This centralizes error handling — instead of try/catch in every route, errors bubble up to one place.

### 🏗️ Your Turn

Create error handling middleware that:
1. Logs the error message to the console
2. Sends a JSON response with status 500 and the error message
3. Has all four parameters properly typed

<details>
<summary>See the solution</summary>

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
  res.status(500).json({
    error: 'Internal server error',
  });
};
```

</details>

> ⚠️ **Common Mistake: Missing Parameter Types**
> Writing `(err, req, res, next)` without types causes:
> - `Parameter 'err' implicitly has an 'any' type`
> - `Parameter 'req' implicitly has an 'any' type`
> - etc.
>
> Always import and use `Request`, `Response`, `NextFunction` from Express, and type `err` as `Error`.

> ⚠️ **Common Mistake: Only Three Parameters**
> If you write `(err, req, res)` with only three parameters, Express treats it as a **regular** middleware, not an error handler. The fourth parameter (`next`) is what makes Express recognize it as error middleware. Even if you don't use `next`, you must include it.

---

## 3. Build Project Routes

### 🏗️ Your Turn

Build 5 endpoints for projects. For each one, think about:
- What SQL query do you need?
- What validation is required?
- What status code should be returned?
- What happens if the project isn't found?

**Endpoint 1: GET / — List all projects**

<details>
<summary>See the solution</summary>

```typescript
// GET /api/projects
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
```

</details>

**Endpoint 2: POST / — Create a project**

Validate: `name` is required and max 100 characters.

<details>
<summary>See the solution</summary>

```typescript
// POST /api/projects
router.post('/', async (req: Request, res: Response, next: NextFunction) => {
  try {
    const { name, description } = req.body as CreateProjectRequest;

    if (!name || name.trim().length === 0) {
      res.status(400).json({ error: 'Project name is required' });
      return;
    }

    if (name.length > 100) {
      res.status(400).json({ error: 'Project name must be 100 characters or less' });
      return;
    }

    const result = await pool.query(
      'INSERT INTO projects (name, description) VALUES ($1, $2) RETURNING *',
      [name.trim(), description || '']
    );

    res.status(201).json(result.rows[0]);
  } catch (err) {
    next(err);
  }
});
```

</details>

> **Key Concept: Parameterized Queries**
> Notice `$1, $2` instead of putting values directly in the SQL string. This prevents **SQL injection** — a critical security vulnerability where attackers put SQL code in form fields. The database treats `$1` and `$2` as data, never as SQL commands.
>
> **Never do this:**
> ```typescript
> // DANGEROUS — SQL injection vulnerability!
> pool.query(`INSERT INTO projects (name) VALUES ('${name}')`);
> ```
>
> If `name` was `'); DROP TABLE projects; --`, this would delete your entire table.

**Endpoint 3: GET /:id — Get project with tasks**

<details>
<summary>See the solution</summary>

```typescript
// GET /api/projects/:id
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

    const project: ProjectWithTasks = {
      ...projectResult.rows[0],
      tasks: tasksResult.rows,
    };

    res.json(project);
  } catch (err) {
    next(err);
  }
});
```

</details>

**Endpoint 4: PUT /:id — Update a project**

<details>
<summary>See the solution</summary>

```typescript
// PUT /api/projects/:id
router.put('/:id', async (req: Request, res: Response, next: NextFunction) => {
  try {
    const { id } = req.params;
    const { name, description } = req.body as UpdateProjectRequest;

    if (name !== undefined && name.trim().length === 0) {
      res.status(400).json({ error: 'Project name cannot be empty' });
      return;
    }

    if (name !== undefined && name.length > 100) {
      res.status(400).json({ error: 'Project name must be 100 characters or less' });
      return;
    }

    // Build dynamic query based on provided fields
    const updates: string[] = [];
    const values: (string | undefined)[] = [];
    let paramIndex = 1;

    if (name !== undefined) {
      updates.push(`name = $${paramIndex++}`);
      values.push(name.trim());
    }

    if (description !== undefined) {
      updates.push(`description = $${paramIndex++}`);
      values.push(description);
    }

    if (updates.length === 0) {
      res.status(400).json({ error: 'No fields to update' });
      return;
    }

    updates.push(`updated_at = NOW()`);
    values.push(id);

    const result = await pool.query(
      `UPDATE projects SET ${updates.join(', ')} WHERE id = $${paramIndex} RETURNING *`,
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
```

</details>

**Why dynamic updates?** If the client sends only `{ name: "New Name" }`, we should update only the name — not reset the description to empty. The dynamic query builder adds only the fields that were provided.

**Endpoint 5: DELETE /:id — Delete a project**

<details>
<summary>See the solution</summary>

```typescript
// DELETE /api/projects/:id
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

    res.status(204).send();
  } catch (err) {
    next(err);
  }
});
```

</details>

### Complete Projects Router

<details>
<summary>See the full file</summary>

Create `server/src/routes/projects.ts`:

```typescript
import { Router, Request, Response, NextFunction } from 'express';
import pool from '../db';
import { CreateProjectRequest, UpdateProjectRequest, ProjectWithTasks } from '../types';

const router = Router();

// GET /api/projects — List all projects
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

// POST /api/projects — Create a project
router.post('/', async (req: Request, res: Response, next: NextFunction) => {
  try {
    const { name, description } = req.body as CreateProjectRequest;

    if (!name || name.trim().length === 0) {
      res.status(400).json({ error: 'Project name is required' });
      return;
    }

    if (name.length > 100) {
      res.status(400).json({ error: 'Project name must be 100 characters or less' });
      return;
    }

    const result = await pool.query(
      'INSERT INTO projects (name, description) VALUES ($1, $2) RETURNING *',
      [name.trim(), description || '']
    );

    res.status(201).json(result.rows[0]);
  } catch (err) {
    next(err);
  }
});

// GET /api/projects/:id — Get project with tasks
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

    const project: ProjectWithTasks = {
      ...projectResult.rows[0],
      tasks: tasksResult.rows,
    };

    res.json(project);
  } catch (err) {
    next(err);
  }
});

// PUT /api/projects/:id — Update a project
router.put('/:id', async (req: Request, res: Response, next: NextFunction) => {
  try {
    const { id } = req.params;
    const { name, description } = req.body as UpdateProjectRequest;

    if (name !== undefined && name.trim().length === 0) {
      res.status(400).json({ error: 'Project name cannot be empty' });
      return;
    }

    if (name !== undefined && name.length > 100) {
      res.status(400).json({ error: 'Project name must be 100 characters or less' });
      return;
    }

    const updates: string[] = [];
    const values: (string | undefined)[] = [];
    let paramIndex = 1;

    if (name !== undefined) {
      updates.push(`name = $${paramIndex++}`);
      values.push(name.trim());
    }

    if (description !== undefined) {
      updates.push(`description = $${paramIndex++}`);
      values.push(description);
    }

    if (updates.length === 0) {
      res.status(400).json({ error: 'No fields to update' });
      return;
    }

    updates.push(`updated_at = NOW()`);
    values.push(id);

    const result = await pool.query(
      `UPDATE projects SET ${updates.join(', ')} WHERE id = $${paramIndex} RETURNING *`,
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

// DELETE /api/projects/:id — Delete project and its tasks
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

    res.status(204).send();
  } catch (err) {
    next(err);
  }
});

export { router as projectsRouter };
```

</details>

---

## 4. Build Task Routes

<details>
<summary>See the full file</summary>

Create `server/src/routes/tasks.ts`:

```typescript
import { Router, Request, Response, NextFunction } from 'express';
import pool from '../db';
import { CreateTaskRequest, UpdateTaskRequest } from '../types';

const router = Router();

// GET /api/projects/:projectId/tasks — List tasks for a project
router.get('/projects/:projectId/tasks', async (req: Request, res: Response, next: NextFunction) => {
  try {
    const { projectId } = req.params;

    // Verify project exists
    const projectCheck = await pool.query(
      'SELECT id FROM projects WHERE id = $1',
      [projectId]
    );

    if (projectCheck.rows.length === 0) {
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
});

// POST /api/projects/:projectId/tasks — Create a task
router.post('/projects/:projectId/tasks', async (req: Request, res: Response, next: NextFunction) => {
  try {
    const { projectId } = req.params;
    const { title } = req.body as CreateTaskRequest;

    if (!title || title.trim().length === 0) {
      res.status(400).json({ error: 'Task title is required' });
      return;
    }

    if (title.length > 200) {
      res.status(400).json({ error: 'Task title must be 200 characters or less' });
      return;
    }

    // Verify project exists
    const projectCheck = await pool.query(
      'SELECT id FROM projects WHERE id = $1',
      [projectId]
    );

    if (projectCheck.rows.length === 0) {
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
});

// PUT /api/tasks/:id — Update a task
router.put('/tasks/:id', async (req: Request, res: Response, next: NextFunction) => {
  try {
    const { id } = req.params;
    const { title, completed } = req.body as UpdateTaskRequest;

    if (title !== undefined && title.trim().length === 0) {
      res.status(400).json({ error: 'Task title cannot be empty' });
      return;
    }

    const updates: string[] = [];
    const values: (string | boolean | undefined)[] = [];
    let paramIndex = 1;

    if (title !== undefined) {
      updates.push(`title = $${paramIndex++}`);
      values.push(title.trim());
    }

    if (completed !== undefined) {
      updates.push(`completed = $${paramIndex++}`);
      values.push(completed);
    }

    if (updates.length === 0) {
      res.status(400).json({ error: 'No fields to update' });
      return;
    }

    updates.push(`updated_at = NOW()`);
    values.push(id);

    const result = await pool.query(
      `UPDATE tasks SET ${updates.join(', ')} WHERE id = $${paramIndex} RETURNING *`,
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
});

// DELETE /api/tasks/:id — Delete a task
router.delete('/tasks/:id', async (req: Request, res: Response, next: NextFunction) => {
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
});

export { router as tasksRouter };
```

</details>

---

## 5. Build the Server Entry Point

### 🏗️ Your Turn

Build `index.ts` that:
1. Loads environment variables
2. Creates an Express app with JSON and CORS middleware
3. Mounts project and task routers
4. Adds a health check that tests the database connection
5. Adds the error handler middleware (must be last)
6. Starts the server

<details>
<summary>See the solution</summary>

Create `server/src/index.ts`:

```typescript
import dotenv from 'dotenv';
dotenv.config();

import express, { Request, Response } from 'express';
import cors from 'cors';
import pool from './db';
import { projectsRouter } from './routes/projects';
import { tasksRouter } from './routes/tasks';
import { errorHandler } from './middleware/errorHandler';

const app = express();
const PORT = process.env.PORT || 3001;
const CORS_ORIGIN = process.env.CORS_ORIGIN || 'http://localhost:5173';

// Middleware
app.use(express.json());
app.use(cors({
  origin: CORS_ORIGIN,
}));

// Routes
app.use('/api/projects', projectsRouter);
app.use('/api', tasksRouter);

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

// Error handler (must be LAST middleware)
app.use(errorHandler);

app.listen(PORT, () => {
  console.log(`Server running on http://localhost:${PORT}`);
});
```

</details>

> ⚠️ **Common Mistake: Error handler must be last**
> Express calls middleware in order. If `app.use(errorHandler)` comes before your routes, it will never catch errors from those routes. Error middleware must be registered after all routes.

> ⚠️ **Common Mistake: CORS trailing slash**
> `CORS_ORIGIN` must NOT have a trailing slash. `'http://localhost:5173/'` ≠ `'http://localhost:5173'`. The browser sends origins without trailing slashes.

---

## 6. Test All Endpoints

Start the server: `npm run dev`

Test every endpoint with curl:

```bash
# Health check
curl http://localhost:3001/api/health

# Create a project
curl -X POST http://localhost:3001/api/projects \
  -H "Content-Type: application/json" \
  -d '{"name": "TaskForge MVP", "description": "First project"}'

# List projects
curl http://localhost:3001/api/projects

# Get project with tasks (use the ID from the create response)
curl http://localhost:3001/api/projects/1

# Create a task
curl -X POST http://localhost:3001/api/projects/1/tasks \
  -H "Content-Type: application/json" \
  -d '{"title": "Set up database"}'

# Toggle task complete
curl -X PUT http://localhost:3001/api/tasks/1 \
  -H "Content-Type: application/json" \
  -d '{"completed": true}'

# Delete a task
curl -X DELETE http://localhost:3001/api/tasks/1

# Delete a project (cascades to tasks)
curl -X DELETE http://localhost:3001/api/projects/1

# Test validation
curl -X POST http://localhost:3001/api/projects \
  -H "Content-Type: application/json" \
  -d '{"name": ""}'
```

### ✅ Checkpoint

- [ ] Health check shows `database: "connected"`
- [ ] Can create, read, update, delete projects
- [ ] Can create, read, update, delete tasks
- [ ] Invalid data returns 400 with error message
- [ ] Nonexistent resources return 404
- [ ] Deleting a project deletes its tasks (cascade)
- [ ] **Data persists after restarting the server**

---

## 7. Commit

```bash
git add .
git commit -m "feat: add full CRUD backend with project and task routes"
```

---

## 🧠 Spatial Check-In

1. Why do we use `$1, $2` placeholders instead of string interpolation in SQL queries?

2. What happens if you remove `next(err)` from a catch block and replace it with nothing?

3. Look at the health check endpoint. Why does it query the database (`SELECT NOW()`) instead of just returning `{ status: "ok" }`?

<details>
<summary>Check Your Answers</summary>

1. **SQL injection prevention.** Placeholders tell PostgreSQL to treat the values as data, never as SQL commands. Without them, an attacker could inject SQL code through form fields.

2. **The error is silently swallowed.** The request hangs forever — the client never gets a response. `next(err)` passes the error to the error handler middleware, which sends a proper error response. Without it, Express doesn't know something went wrong.

3. **A real health check tests the full stack.** If the backend process is running but the database is down, returning `"ok"` would be misleading. By querying the database, the health check confirms the entire system is operational — this is what monitoring tools need.

</details>

---

> **Session Break** — Backend complete with 10 CRUD endpoints.
> When you return, you'll build the React frontend in [Step 5 — Frontend](../05-frontend/).

---

| | |
|:---|---:|
| [← Step 3: Database](../03-database/) | [Step 5 — Frontend →](../05-frontend/) |

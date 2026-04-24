# Step 4 — Building the Backend: Full CRUD API

## New Syntax You'll Meet in This Lesson

Level 1 covered the JavaScript/TypeScript foundations (imports, destructuring, arrow functions, async/await, types). Those all apply here. Level 2 adds a few new shapes that recur across every endpoint — understand these once, and the 838 lines that follow are variations on the same pattern.

- **`pool.query(sql, params)`** — runs a SQL statement on the database. `sql` is a string with `$1, $2, ...` placeholders; `params` is an array of the values to substitute. Returns a promise resolving to a result object with `.rows` (an array of matching rows) and `.rowCount` (how many rows were affected).
- **Parameterized queries (`$1, $2`)** — never stick user input into a SQL string with `${}`. Always pass placeholders and let the driver substitute safely. This is how you prevent SQL injection. (You'll see the dangerous pattern called out explicitly below.)
- **`req.params`** — Express populates this object from URL path placeholders. If the route is `/projects/:id` and the URL is `/projects/42`, then `req.params.id` is `'42'` (a string — always a string from the URL). Different from `req.query` (the `?key=value` part) and `req.body` (the JSON payload of POST/PUT).
- **`try / catch / next(err)`** — the Express 4 async error-forwarding pattern. Async functions throw rejected promises; Express doesn't automatically catch them. So you wrap the handler in `try`, and in `catch` call `next(err)` to hand the error to the error-handling middleware.
- **`res.status(204).send()`** — respond with "204 No Content" and no body. Used for DELETE endpoints where there's nothing meaningful to return.
- **Status codes you'll use a lot**: `200 OK` (default success), `201 Created` (successful POST), `204 No Content` (successful DELETE), `400 Bad Request` (validation failed), `404 Not Found` (resource doesn't exist), `500 Internal Server Error` (something threw unexpectedly).

Everything else is JS/TS you already know from Level 1. Each code block below has a line-by-line walkthrough.

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

### Reading This File Line-by-Line

Seven type declarations. Each one is a shape you'll use in the routes below.

- `export interface Project { ... }` — a **blueprint** matching a row of the `projects` table exactly: `id` (number, the primary key), `name` and `description` (strings), and two timestamps as strings (PostgreSQL timestamps come back from the driver as ISO date strings by default).
- `export interface Task { ... }` — same idea for the `tasks` table. Notice `completed: boolean` — booleans travel as true JavaScript booleans through the `pg` driver.
- `export interface ProjectWithTasks extends Project { tasks: Task[]; }`
  - `extends Project` — this is TypeScript **interface extension**. `ProjectWithTasks` gets every field of `Project` plus `tasks: Task[]`. It's the equivalent of writing out all the fields again and adding the new one.
  - `Task[]` — "an array of `Task` objects."
- `export interface CreateProjectRequest { name: string; description?: string; }`
  - Only the fields the **client** sends when creating. `id`, `created_at`, `updated_at` are server-generated and don't appear here.
  - `description?: string` — the `?` makes the field **optional**. TypeScript allows `{ name: 'X' }` (no description) as well as `{ name: 'X', description: 'Y' }`.
- `export interface UpdateProjectRequest { name?: string; description?: string; }` — both fields optional. An update can change one, the other, or both.
- `export interface CreateTaskRequest { title: string; }` — only `title`. `project_id` comes from the URL (`/projects/:projectId/tasks`), and `completed` defaults to `false`.
- `export interface UpdateTaskRequest { title?: string; completed?: boolean; }` — either field optional.

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

### Reading This File Line-by-Line

- `import { Request, Response, NextFunction } from 'express';` — three named type imports. No runtime behavior — they only exist for TypeScript's sake.
- `export const errorHandler = ( ... ): void => { ... };`
  - Exported arrow function assigned to a const.
  - Four parameters: `err`, `_req`, `res`, `_next`. The **four-parameter count** is what makes Express recognize this as an error handler (three parameters = regular middleware).
  - Underscores on `_req` and `_next` signal "declared but unused" — standard lint convention. You must declare them in the right position because Express identifies parameter roles by order, not name.
  - `: void` — returns nothing. The work happens through the side effect of sending a response.
- `console.error('Error:', err.message);`
  - Logs the error message to stderr. In production you'd use a structured logger; for Level 2 this is enough.
  - `err.message` is the human-readable part of the error. `err.stack` has the full stack trace if you want to log that too.
- `res.status(500).json({ error: 'Internal server error' });`
  - Method chaining. `res.status(500)` sets the status code and returns `res` so the next call can run on the same object.
  - `.json({...})` serializes the object to JSON and sends it. We return a generic message to avoid leaking internal details (SQL errors, file paths, etc.) to the client.

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

### Reading This Endpoint Line-by-Line

This is the simplest endpoint in the file. Every other endpoint is a variation on this shape, so really understand this one.

- `router.get('/', async (...) => {...})`
  - `router.get(path, handler)` — register a handler for `GET` requests at this path. Since this router will be mounted at `/api/projects` in the entry file, `/` here means "the base `/api/projects` URL itself."
  - The handler is an **async arrow function** — async because it uses `await` inside.
  - `_req: Request, res: Response, next: NextFunction` — the three Express parameters, typed. `_req` has an underscore because this handler doesn't read anything from the request.
- `try { ... } catch (err) { next(err); }` — the async-error pattern from the new-syntax note. If anything inside `try` throws (or a promise rejects), the `catch` block runs and `next(err)` forwards the error to our `errorHandler` middleware.
- `const result = await pool.query('SELECT * FROM projects ORDER BY created_at DESC');`
  - `pool.query(sql)` runs the SQL and returns a promise.
  - `await` pauses until the database responds.
  - No placeholders or values array here because this query has no user input — it's a static string.
  - `ORDER BY created_at DESC` — newest rows first. (Sorting happens in the database, not in JavaScript.)
- `res.json(result.rows);`
  - `result.rows` is an array of row objects. Each object has keys matching your column names (`{ id, name, description, created_at, updated_at }`).
  - `res.json(...)` serializes the array to JSON and sends it with status 200 (the default).

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

### Reading This Endpoint Line-by-Line

- `const { name, description } = req.body as CreateProjectRequest;`
  - Destructure `name` and `description` out of `req.body`.
  - `as CreateProjectRequest` is a **type assertion** — "trust me, TypeScript, the body has this shape." This is necessary because Express doesn't know the body's type. We immediately validate afterward, so if the client lied, we reject the request before using the values.
- `if (!name || name.trim().length === 0)`
  - `!name` catches `undefined`, `null`, or empty string (all falsy).
  - `name.trim()` — every string has a `.trim()` method that returns a copy with leading/trailing whitespace removed. `'  hi  '.trim()` is `'hi'`. So `'   '.trim().length === 0` catches whitespace-only inputs that would otherwise pass the first check.
  - If either branch is true, respond 400 with an error and `return` to stop execution.
- `if (name.length > 100)` — enforce the character limit we set in the schema (`VARCHAR(100)`). Rejecting here gives a friendlier error than letting PostgreSQL throw.
- `await pool.query('INSERT INTO projects (name, description) VALUES ($1, $2) RETURNING *', [name.trim(), description || '']);`
  - `$1, $2` — **placeholders**. Whatever is passed as the second argument is substituted in, safely, by the `pg` driver. `$1` becomes the first array element, `$2` the second.
  - `RETURNING *` — PostgreSQL extension that returns every column of the newly inserted row. Without it, `INSERT` returns only a row count and you'd need a second `SELECT` to get the auto-generated `id` and timestamps.
  - `[name.trim(), description || '']` — the values array. `name.trim()` strips whitespace; `description || ''` falls back to an empty string when description is undefined (the `|| ''` uses short-circuit OR).
- `res.status(201).json(result.rows[0]);`
  - `201 Created` is the correct status for a successful POST that creates a resource.
  - `result.rows[0]` — `INSERT ... RETURNING *` returns a single row; we grab it from the array.

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

### Reading This Endpoint Line-by-Line

- `router.get('/:id', ...)` — `:id` is a **URL parameter**. Matches any value in that position: `/42`, `/abc`, `/whatever` all match. Express stores the matched segment in `req.params.id`.
- `const { id } = req.params;` — pull `id` out of `req.params`. Note: `id` is always a **string** here, even for numeric IDs, because URLs are text. PostgreSQL will coerce it to an integer when we compare against the `id` column.
- `await pool.query('SELECT * FROM projects WHERE id = $1', [id]);`
  - Same parameterized-query pattern. The `id` is substituted into `$1` safely.
- `if (projectResult.rows.length === 0)` — `SELECT` returns an empty array when nothing matches. Empty means "not found," so we return 404.
- The second query fetches that project's tasks.
- `const project: ProjectWithTasks = { ...projectResult.rows[0], tasks: tasksResult.rows };`
  - `...projectResult.rows[0]` — **spread operator**. Copies every field from the project row into a new object: `{ id, name, description, created_at, updated_at, ...}`.
  - Then we add `tasks: tasksResult.rows` as a new field.
  - The result matches the `ProjectWithTasks` interface: all project fields plus a `tasks` array.

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

### Reading This Endpoint Line-by-Line

This is the trickiest endpoint because the SQL is **built dynamically** based on which fields were provided. Take it slow.

- `const { id } = req.params;` — the project ID from the URL.
- `const { name, description } = req.body as UpdateProjectRequest;` — pull both fields from the body. Either could be undefined.
- The two `if` blocks validate `name` **only if it was provided**. `name !== undefined` checks "the client sent a `name` field" (possibly empty, possibly too long). This lets the client send just `{ description: 'new' }` without being forced to also send `name`.

Now the interesting part — building the SQL and params arrays together:

- `const updates: string[] = [];` — accumulator for SQL fragments like `'name = $1'`.
- `const values: (string | undefined)[] = [];` — parallel accumulator for the actual values.
- `let paramIndex = 1;` — the next placeholder number. We use `let` because we'll increment it.

- `if (name !== undefined) { updates.push(\`name = $${paramIndex++}\`); values.push(name.trim()); }`
  - If `name` was provided, push `'name = $1'` to `updates` and `name.trim()` to `values`.
  - `paramIndex++` is **post-increment**: it uses the current value (1) in this expression, then increments `paramIndex` to 2 for next time.
  - The template literal `` `name = $${paramIndex++}` `` expands to `'name = $1'` on the first call.
- Same pattern for `description`.

- `if (updates.length === 0) { res.status(400).json({ error: 'No fields to update' }); return; }` — if the client sent an empty body, reject.

- `updates.push(\`updated_at = NOW()\`);` — always bump `updated_at`. `NOW()` is PostgreSQL's built-in "current time" function, written directly in the SQL (no placeholder needed because there's no user input involved).
- `values.push(id);` — the project ID goes last. It'll be substituted into the `WHERE id = $N` at the end of the query.

- `await pool.query(\`UPDATE projects SET ${updates.join(', ')} WHERE id = $${paramIndex} RETURNING *\`, values);`
  - `updates.join(', ')` — joins the array with `, ` between items. `['name = $1', 'description = $2', 'updated_at = NOW()']` becomes `'name = $1, description = $2, updated_at = NOW()'`.
  - `$${paramIndex}` — `paramIndex` is now whatever the last post-increment left it at (equal to `values.length` since we pushed `id` last). So if there were two field updates, `paramIndex` is 3 and this becomes `$3`.
  - Final SQL example: `UPDATE projects SET name = $1, description = $2, updated_at = NOW() WHERE id = $3 RETURNING *`.
- The rest follows the by-now-familiar pattern: if no row matched, 404; otherwise return the updated row.

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

### Reading This Endpoint Line-by-Line

- `router.delete('/:id', ...)` — matches `DELETE /api/projects/42`.
- `await pool.query('DELETE FROM projects WHERE id = $1 RETURNING *', [id]);`
  - `DELETE ... RETURNING *` returns the deleted row(s). Lets us tell the difference between "deleted something" (rows array has one element) and "nothing matched" (empty array).
  - Because of `ON DELETE CASCADE` in the schema, any tasks belonging to this project are deleted automatically as a side effect.
- `if (result.rows.length === 0)` — if nothing matched the `WHERE`, return 404.
- `res.status(204).send();`
  - `204 No Content` is the conventional status for a successful DELETE. Signals "I did it; there's nothing meaningful to return."
  - `.send()` (not `.json()`) because there's no body. Calling `.json(undefined)` would still send JSON headers; `.send()` with no argument sends an empty body.

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

### Reading the Tasks Router

Every endpoint follows the same shape you just learned in the projects router. Instead of walking every line again, here's what's **new or different** in this file:

- **Nested route paths.** `router.get('/projects/:projectId/tasks', ...)` has **two** URL parameters in the same path: `:projectId` and the implicit resource. In the handler, `req.params.projectId` is the project's ID. We use `projectId` (not just `id`) as the name to make the intent clear at the call site.
- **Mounting.** The tasks router is mounted at `/api` (not `/api/tasks`) in the entry file, because its routes cover two URL shapes: `/projects/:projectId/tasks` (nested under a project) and `/tasks/:id` (direct access by task ID). Keeping both shapes in one router is a judgment call — some teams split them into two routers. Either works.
- **"Project exists?" check** (in the GET and POST tasks endpoints):
  ```typescript
  const projectCheck = await pool.query(
    'SELECT id FROM projects WHERE id = $1',
    [projectId]
  );
  if (projectCheck.rows.length === 0) {
    res.status(404).json({ error: 'Project not found' });
    return;
  }
  ```
  Before creating a task under a project, we verify the project exists. Otherwise the `INSERT` would either fail with a foreign key error (less friendly to the client) or, in the GET case, silently return an empty array (misleading — "no tasks" and "project doesn't exist" should be different responses).
  - `SELECT id` instead of `SELECT *` — we only need to know **whether a row exists**, not its contents. Selecting one column is a small optimization.
- **Union type for update values**: `const values: (string | boolean | undefined)[] = [];` — tasks allow updating either `title` (a string) or `completed` (a boolean), so the accumulator's element type is `string | boolean | undefined`. Compare with the projects version, which was `(string | undefined)[]` because projects only have string fields.

Everything else — validation, parameterized queries, dynamic update builder, 404 / 201 / 204 status codes, `try/catch/next(err)` — is identical to what you just read in the projects router.

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

### Reading This File Line-by-Line

```typescript
import dotenv from 'dotenv';
dotenv.config();
```

**Notice the import order.** `dotenv` comes before any other import, and `dotenv.config()` runs immediately. This is critical: every other module we'll import (including `./db`) reads `process.env` during load. If dotenv hasn't run yet, those env vars are undefined. Put `dotenv.config()` **first**.

```typescript
import express, { Request, Response } from 'express';
import cors from 'cors';
import pool from './db';
import { projectsRouter } from './routes/projects';
import { tasksRouter } from './routes/tasks';
import { errorHandler } from './middleware/errorHandler';
```

Seven imports. Notice mixing named and default imports on the first line: `express` (default) and `{ Request, Response }` (named types) come from the same module.

```typescript
const app = express();
const PORT = process.env.PORT || 3001;
const CORS_ORIGIN = process.env.CORS_ORIGIN || 'http://localhost:5173';
```

- `express()` — call the factory to create a fresh app object.
- `|| 3001` and `|| 'http://localhost:5173'` — fallbacks when the env var isn't set. Deployment platforms set `PORT`; locally we use the default.

```typescript
app.use(express.json());
app.use(cors({ origin: CORS_ORIGIN }));
```

Global middleware, registered in order. Every request runs through these before hitting any route:

- `express.json()` parses JSON request bodies. Without it, `req.body` is `undefined`.
- `cors({...})` adds the `Access-Control-Allow-Origin` header so the browser lets the frontend talk to us.

```typescript
app.use('/api/projects', projectsRouter);
app.use('/api', tasksRouter);
```

Mount the two routers at their prefixes. Requests to `/api/projects/...` go through the projects router; `/api/projects/5/tasks`, `/api/tasks/7`, and `/api/health` go through the tasks router's path definitions (or directly to the health check below).

```typescript
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
```

- A health check that actually queries the database. "Express is running" is not enough — we also want "the DB is reachable."
- `catch { ... }` — the parentheses around the error are optional in TypeScript 4+ when you don't need the error object.
- If the query fails, respond 500 with `database: 'disconnected'`. Load balancers and monitoring tools poll this endpoint to decide whether this instance is healthy.

```typescript
app.use(errorHandler);
```

Register the error handler **last**. Express checks middleware arity (parameter count); four parameters marks this one as an error handler. Because it's registered after all routes, any error from any route reaches it via `next(err)`.

```typescript
app.listen(PORT, () => {
  console.log(`Server running on http://localhost:${PORT}`);
});
```

Start the HTTP server. The callback fires once the port is actually bound.

> ⚠️ **Common Mistake: Error handler must be last**
> Express calls middleware in order. If `app.use(errorHandler)` comes before your routes, it will never catch errors from those routes. Error middleware must be registered after all routes.

> ⚠️ **Common Mistake: CORS trailing slash**
> `CORS_ORIGIN` must NOT have a trailing slash. `'http://localhost:5173/'` ≠ `'http://localhost:5173'`. The browser sends origins without trailing slashes.

---

## 6. Test All Endpoints

Start the server: `npm run dev`

Test every endpoint with curl:

**Reading a `curl` command (if you're new to it):**

- `curl <url>` — by default, make a **GET** request to that URL and print the response to the terminal.
- `-X POST` / `-X PUT` / `-X DELETE` — change the HTTP method.
- `-H "Header-Name: value"` — add an HTTP header. You can repeat `-H` for multiple headers.
- `-d '<body>'` — supply a request body. With `-d`, curl defaults to POST if you didn't pass `-X`. Single-quote the JSON so your shell doesn't try to interpret `$` or spaces.
- Line continuation `\` at end of line — tells the shell "this command continues on the next line." Just for readability.

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

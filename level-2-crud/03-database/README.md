# Step 3 — Database Design: Schema, SQL, and Connections

## Spatial Orientation

```
task-forge/
└── server/
    └── src/
        └── db/               ← YOU ARE HERE
            ├── schema.sql      ← Table definitions
            └── index.ts        ← Database connection
```

You're building the data layer — the foundation that everything else sits on. If the schema is wrong, everything above it struggles.

---

## 1. Design the Schema

> **Key Concept: Schema**
> A database schema is the blueprint of your tables — their columns, data types, constraints, and relationships. Think of it like the floor plan of a building: it defines what rooms exist (tables), how big they are (columns), and how they connect (relationships).

### 🏗️ Your Turn

Before looking at the schema, design it yourself. TaskForge needs:
- **Projects** with a name and description
- **Tasks** that belong to a project, with a title and completed status
- Auto-generated IDs, timestamps for both tables

Questions to consider:
- How do you link a task to its project?
- What happens to tasks when a project is deleted?
- Which columns should be required vs optional?

<details>
<summary>See the solution</summary>

Create `server/src/db/schema.sql`:

```sql
-- TaskForge Database Schema

-- Projects table
CREATE TABLE projects (
  id          SERIAL PRIMARY KEY,
  name        VARCHAR(100) NOT NULL,
  description TEXT DEFAULT '',
  created_at  TIMESTAMP DEFAULT NOW(),
  updated_at  TIMESTAMP DEFAULT NOW()
);

-- Tasks table
CREATE TABLE tasks (
  id          SERIAL PRIMARY KEY,
  project_id  INTEGER NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
  title       VARCHAR(200) NOT NULL,
  completed   BOOLEAN DEFAULT false,
  created_at  TIMESTAMP DEFAULT NOW(),
  updated_at  TIMESTAMP DEFAULT NOW()
);
```

</details>

**Line-by-line breakdown:**

| SQL | What It Does | Why |
|-----|-------------|-----|
| `SERIAL PRIMARY KEY` | Auto-incrementing integer, unique identifier | Every row needs a unique ID. SERIAL auto-generates it (1, 2, 3, ...) |
| `VARCHAR(100) NOT NULL` | String up to 100 chars, cannot be empty | Enforces at the database level — even if the backend has a bug, the DB rejects invalid data |
| `TEXT DEFAULT ''` | Unlimited length string, defaults to empty | Description is optional but always has a value |
| `TIMESTAMP DEFAULT NOW()` | Auto-set to current time on creation | Automatic audit trail — you always know when a row was created |
| `REFERENCES projects(id)` | Foreign key — links tasks to projects | Database enforces that `project_id` must be a real project ID |
| `ON DELETE CASCADE` | When a project is deleted, its tasks are deleted too | Prevents orphan tasks (tasks pointing to a deleted project) |

> **Key Concept: Foreign Key**
> A foreign key is a column that references the primary key of another table. It creates a relationship: "this task **belongs to** this project." The database enforces this — you can't create a task with `project_id = 999` if project 999 doesn't exist.

> **Key Concept: ON DELETE CASCADE**
> CASCADE means "when the parent is deleted, automatically delete all children." Without it, deleting a project would fail because tasks still reference it (or worse, leave orphan tasks pointing to nothing).

### 🧠 Think About It

What would happen if you tried to insert a task with `project_id = 999` when no project with ID 999 exists?

<details>
<summary>Answer</summary>

The database would reject the INSERT with a foreign key violation error: `Key (project_id)=(999) is not present in table "projects"`. This is the database enforcing data integrity — it's impossible to have a task pointing to a nonexistent project.

</details>

### Run the Schema

```bash
psql taskforge < server/src/db/schema.sql
```

Verify the tables were created:

```bash
psql taskforge -c "\dt"
```

You should see `projects` and `tasks` in the list.

---

## 2. Practice SQL

Before writing code, get comfortable with SQL by running queries manually:

```bash
psql taskforge
```

This opens an interactive SQL prompt. Try these:

```sql
-- Create a project
INSERT INTO projects (name, description)
VALUES ('My First Project', 'Learning SQL')
RETURNING *;

-- Create a task for that project
INSERT INTO tasks (project_id, title)
VALUES (1, 'Learn SELECT statements')
RETURNING *;

-- Get all projects
SELECT * FROM projects;

-- Get tasks for project 1
SELECT * FROM tasks WHERE project_id = 1;

-- Mark a task complete
UPDATE tasks SET completed = true, updated_at = NOW()
WHERE id = 1
RETURNING *;

-- Delete a project (tasks cascade!)
DELETE FROM projects WHERE id = 1;

-- Verify tasks were deleted too
SELECT * FROM tasks;
```

Exit psql: `\q`

> **Key Concept: `RETURNING *`**
> In PostgreSQL, `INSERT`, `UPDATE`, and `DELETE` can include `RETURNING *` to return the affected row(s). This is incredibly useful — instead of inserting a row and then doing a separate SELECT to get the generated ID and timestamps, you get everything in one query.

### 🧠 Think About It

After the `DELETE FROM projects WHERE id = 1` above, what would `SELECT * FROM tasks` return?

<details>
<summary>Answer</summary>

An empty table. `ON DELETE CASCADE` automatically deleted all tasks belonging to project 1. Without CASCADE, the DELETE would have failed with a foreign key constraint violation.

</details>

---

## 3. Connect to the Database from Node.js

> **Key Concept: Connection Pool**
> A connection pool maintains a set of open database connections that are reused across requests. Without pooling, every API request would open a new connection (slow) and close it (wasteful). Think of it like a fleet of taxis: instead of buying a new car for each passenger, you reuse cars from a shared fleet.

### 🏗️ Your Turn

Create the database connection module. It needs to:
1. Import the Pool class from `pg`
2. Load environment variables with `dotenv`
3. Create a pool with the `DATABASE_URL`
4. Test the connection on startup
5. Export the pool for use in routes

<details>
<summary>See the solution</summary>

Create `server/src/db/index.ts`:

```typescript
import { Pool } from 'pg';
import dotenv from 'dotenv';

dotenv.config();

const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
});

// Test connection on startup
pool.query('SELECT NOW()')
  .then(() => {
    console.log('Connected to PostgreSQL');
  })
  .catch((err: Error) => {
    console.error('Failed to connect to PostgreSQL:', err.message);
    process.exit(1);
  });

export default pool;
```

</details>

**Line-by-line breakdown:**

| Line | What It Does | Why |
|------|-------------|-----|
| `import { Pool } from 'pg'` | Import the connection pool class | Pool manages multiple reusable connections |
| `dotenv.config()` | Load `.env` file into `process.env` | Makes `DATABASE_URL` available to the code |
| `new Pool({ connectionString: ... })` | Create a pool pointing to your database | The pool lazily opens connections as needed |
| `pool.query('SELECT NOW()')` | Test the connection with a simple query | Fail fast — if the database is unreachable, crash immediately instead of waiting for the first API request |
| `.catch((err: Error) => { ... process.exit(1) })` | Handle connection failure | `process.exit(1)` stops the server. No point running if the database is down. |

> ⚠️ **Common Mistake: Missing Error Type**
> Writing `.catch((err) => { ... })` without typing `err` causes: `Parameter 'err' implicitly has an 'any' type`. Always type catch parameters: `.catch((err: Error) => { ... })`.

### ✅ Checkpoint

Start the server:
```bash
cd server
npm run dev
```

You should see: `Connected to PostgreSQL` and `Server running on http://localhost:3001`

If you see `Failed to connect to PostgreSQL`:
1. Is PostgreSQL running? → `brew services start postgresql@16`
2. Does the database exist? → `createdb taskforge`
3. Is the connection string correct in `.env`?

---

## 4. Commit

```bash
git add .
git commit -m "feat: add database schema and PostgreSQL connection"
```

---

## 🧠 Spatial Check-In

1. What is the relationship between the `projects` and `tasks` tables? How is it enforced?

2. Why do we use a connection pool instead of opening a new connection for every query?

3. If you change `ON DELETE CASCADE` to `ON DELETE RESTRICT`, what would happen when you try to delete a project that has tasks?

<details>
<summary>Check Your Answers</summary>

1. **One-to-many:** One project has many tasks. It's enforced by the `project_id` foreign key in the `tasks` table, which references `projects(id)`. The database guarantees that every task's `project_id` corresponds to an existing project.

2. **Performance.** Opening a database connection takes ~20-50ms. If every API request opens a new connection, that adds up fast under load. A pool reuses a fixed number of open connections (default: 10), which is much faster.

3. **The DELETE would fail** with a foreign key constraint error. `RESTRICT` means "refuse to delete a parent if it has children." You'd have to delete all tasks first, then delete the project. `CASCADE` automates this.

</details>

---

> **Session Break** — Database designed and connected.
> When you return, you'll build the backend API in [Step 4 — Backend](../04-backend/).

---

| | |
|:---|---:|
| [← Step 2: Project Setup](../02-project-setup/) | [Step 4 — Backend →](../04-backend/) |

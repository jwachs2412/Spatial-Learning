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

### Reading This Schema Line-by-Line

A schema file is a sequence of SQL statements, each ending in `;`. PostgreSQL executes them in order. Lines starting with `--` are comments.

#### The `projects` Table

```sql
CREATE TABLE projects (
  id          SERIAL PRIMARY KEY,
  name        VARCHAR(100) NOT NULL,
  description TEXT DEFAULT '',
  created_at  TIMESTAMP DEFAULT NOW(),
  updated_at  TIMESTAMP DEFAULT NOW()
);
```

- `CREATE TABLE projects ( ... );` — "make a new table named `projects`." Column definitions go inside the parentheses, separated by commas.
- `id SERIAL PRIMARY KEY`
  - `SERIAL` is PostgreSQL shorthand for "auto-incrementing integer." First row gets `id = 1`, next gets `2`, and so on. You never set it manually.
  - `PRIMARY KEY` says "this column uniquely identifies each row." Enforces uniqueness and non-null automatically, and builds an index on it.
- `name VARCHAR(100) NOT NULL`
  - `VARCHAR(100)` — a variable-length string up to 100 characters. Shorter values don't waste space; longer values are rejected.
  - `NOT NULL` — "every row must have a value here." Trying to insert without `name` is an error.
- `description TEXT DEFAULT ''`
  - `TEXT` — like `VARCHAR` but with no length limit. Useful for long free-form text.
  - `DEFAULT ''` — if the insert doesn't provide a description, use an empty string instead of `NULL`. Notice there's no `NOT NULL`, so technically it *could* be null, but the default means it never will be in practice.
- `created_at TIMESTAMP DEFAULT NOW()` and `updated_at TIMESTAMP DEFAULT NOW()`
  - `TIMESTAMP` — a date + time value.
  - `NOW()` is a built-in PostgreSQL function that returns the current moment.
  - `DEFAULT NOW()` — "if no value given, stamp the current time automatically." Applies on insert. For `updated_at`, the backend code is responsible for re-setting it on every update (we'll write that later).

#### The `tasks` Table

```sql
CREATE TABLE tasks (
  id          SERIAL PRIMARY KEY,
  project_id  INTEGER NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
  title       VARCHAR(200) NOT NULL,
  completed   BOOLEAN DEFAULT false,
  created_at  TIMESTAMP DEFAULT NOW(),
  updated_at  TIMESTAMP DEFAULT NOW()
);
```

- `project_id INTEGER NOT NULL REFERENCES projects(id) ON DELETE CASCADE`
  - `INTEGER` — a whole number. We use `INTEGER` here (not `SERIAL`) because we're pointing at an ID that already exists in `projects`; we don't want PostgreSQL to auto-generate one.
  - `NOT NULL` — every task must belong to a project.
  - `REFERENCES projects(id)` — declares a **foreign key**. This says "the value in this column must match an existing `id` in the `projects` table." If you try to insert a task with `project_id = 999` but there's no project with `id = 999`, the database rejects it.
  - `ON DELETE CASCADE` — describes what happens to this task if the referenced project is deleted. `CASCADE` = "delete me too." Without it (or with `ON DELETE RESTRICT`), the database would block the parent delete until all child tasks were removed.
- `title VARCHAR(200) NOT NULL` — same pattern as before.
- `completed BOOLEAN DEFAULT false` — `BOOLEAN` stores `true` or `false`. Default is `false`, so new tasks are incomplete by default.

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

**Reading this command:**

- `psql taskforge` — open psql connected to the `taskforge` database.
- `<` is **shell input redirection**. Instead of typing SQL interactively, this says "feed the contents of `server/src/db/schema.sql` into psql's standard input, as if you typed it yourself."
- The whole file is read and executed top-to-bottom. psql prints `CREATE TABLE` for each successful statement.

Verify the tables were created:

```bash
psql taskforge -c "\dt"
```

**What the `\dt` does:** commands starting with `\` are **psql meta-commands** — shortcuts built into psql itself, not SQL the database understands. `\dt` stands for "describe tables" and lists every table in the current database. Other useful ones you'll meet: `\d tablename` (describe one table's columns), `\q` (quit), `\l` (list databases).

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

### Reading These Queries Line-by-Line

- `INSERT INTO projects (name, description) VALUES ('My First Project', 'Learning SQL') RETURNING *;`
  - `INSERT INTO projects (name, description)` — "add a new row to `projects`, filling in these columns." Columns you don't list take their default values (or `NULL` if no default).
  - `VALUES ('My First Project', 'Learning SQL')` — the data, in the same order as the columns above. Strings in SQL use single quotes.
  - `RETURNING *` — "after the insert, give me back every column of the new row." Without it, `INSERT` returns only a row count. With it, you see the generated `id`, `created_at`, and any default values — all in one query. Very useful when the backend needs the new ID immediately.

- `INSERT INTO tasks (project_id, title) VALUES (1, 'Learn SELECT statements') RETURNING *;`
  - Same shape. `project_id = 1` is legal because project 1 exists (you just created it). If you tried `project_id = 999`, the foreign key constraint would reject the insert.

- `SELECT * FROM projects;`
  - `*` means "every column." No `WHERE` clause means "every row." Use with care on large tables.

- `SELECT * FROM tasks WHERE project_id = 1;`
  - Same pattern, now with a filter: only rows whose `project_id` column equals 1.

- `UPDATE tasks SET completed = true, updated_at = NOW() WHERE id = 1 RETURNING *;`
  - `UPDATE tasks SET column = value, column = value` — modify one or more columns. Separate assignments with commas.
  - `WHERE id = 1` — limit the update to a specific row. **Forgetting `WHERE` would update every task in the table.**
  - `RETURNING *` — show me the row as it looks after the update.

- `DELETE FROM projects WHERE id = 1;`
  - Delete a specific project. Because we set `ON DELETE CASCADE` on `tasks.project_id`, any task belonging to project 1 is automatically deleted as a side effect.

- `SELECT * FROM tasks;`
  - Run after the delete to prove the cascade worked — the tasks table should now be empty.

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

### Reading This File Line-by-Line

```typescript
import { Pool } from 'pg';
import dotenv from 'dotenv';
```

- `import { Pool } from 'pg';` — named import. `Pool` is a **class** the `pg` package exports. Think of a class as a blueprint: we'll call `new Pool(...)` below to build an actual pool object.
- `import dotenv from 'dotenv';` — default import of the `dotenv` package.

```typescript
dotenv.config();
```

- `dotenv.config()` reads `.env` from the current working directory and copies every `KEY=VALUE` line into `process.env`. After this call, `process.env.DATABASE_URL` holds the value we set in `.env`.
- **Order matters:** this must run **before** any code that reads env vars. That's why it's at the top of the file.

```typescript
const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
});
```

- `new Pool({...})` — call the `Pool` constructor. `new` is the keyword that turns a class blueprint into an actual instance.
- `{ connectionString: process.env.DATABASE_URL }` — one configuration option. The `pg` driver parses the URL into host, port, user, password, and database name. There are many other options (`max`, `idleTimeoutMillis`, etc.), but this is enough for local dev.

```typescript
pool.query('SELECT NOW()')
  .then(() => {
    console.log('Connected to PostgreSQL');
  })
  .catch((err: Error) => {
    console.error('Failed to connect to PostgreSQL:', err.message);
    process.exit(1);
  });
```

- `pool.query(sql)` — runs a SQL statement and returns a **promise** that resolves when the database responds.
- This is **Promise chaining** syntax — an alternative to `async/await`. A promise has `.then(handler)` for success and `.catch(handler)` for failure.
- `.then(() => { console.log(...) })` — if the query succeeded, log a success message. The `()` means "I don't care about the result value."
- `.catch((err: Error) => { ... })` — if anything went wrong (wrong URL, database down, etc.), run this handler. `err: Error` types the parameter so TypeScript doesn't flag it as implicit `any`.
- `console.error(...)` — like `console.log`, but writes to **stderr** (standard error stream) instead of stdout. Convention for error messages.
- `process.exit(1)` — immediately stop the Node process with exit code `1`. Exit code `0` means success; anything else signals failure to the shell or deployment platform. "Fail fast" on a missing database is better than limping along and failing on every request.

```typescript
export default pool;
```

Default export of the single pool object so every other file (`import pool from '../db'`) gets the same instance. This is the singleton pattern — one pool for the whole app.

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

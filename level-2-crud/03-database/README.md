`Level 2` **Step 3 of 7** — Database Design

# 03 — Database Design: Schema, SQL, and Connections

## Spatial Orientation

```
task-forge/
├── client/          ← NOT working here right now
└── server/
    └── src/
        └── db/                ← ★ WE ARE HERE ★
            ├── schema.sql     ← Table definitions (we'll build this)
            └── index.ts       ← Database connection (we'll build this)
```

**What layer are we in?** The DATA layer. We're designing the structure of our database and creating the connection from Express to PostgreSQL. This code never runs in the browser — it runs on the server and talks to the database.

---

## Step 1: Create the Folder Structure

> [!IMPORTANT]
> **You should be in:** `task-forge/`

```bash
mkdir -p server/src/db
mkdir -p server/src/routes
mkdir -p server/src/middleware
mkdir -p server/src/types
```

---

## Step 2: Design the Schema

A database schema defines the structure of your tables — what columns they have, what types those columns are, and how tables relate to each other.

TaskForge needs two tables:

```
┌─────────────────────────────┐       ┌─────────────────────────────────┐
│         projects             │       │            tasks                 │
├─────────────────────────────┤       ├─────────────────────────────────┤
│ id          SERIAL PK       │───┐   │ id            SERIAL PK         │
│ name        VARCHAR(100)    │   │   │ project_id    INTEGER FK ──────┤
│ description TEXT             │   └──▶│               REFERENCES        │
│ created_at  TIMESTAMP       │       │               projects(id)      │
│ updated_at  TIMESTAMP       │       │               ON DELETE CASCADE  │
│                             │       │ title         VARCHAR(200)      │
│                             │       │ completed     BOOLEAN            │
│                             │       │ created_at    TIMESTAMP          │
│                             │       │ updated_at    TIMESTAMP          │
└─────────────────────────────┘       └─────────────────────────────────┘
       ONE project                         MANY tasks
```

**The relationship**: One project has many tasks. Each task belongs to exactly one project. This is called a **one-to-many** relationship — the most common relationship in database design.

### Create the Schema File

In VS Code, create a new file at `server/src/db/schema.sql`.

Add this code to `server/src/db/schema.sql`:

```sql
-- TaskForge Database Schema
-- Run with: psql taskforge < server/src/db/schema.sql

-- Drop existing tables (clean slate for development)
-- CASCADE ensures dependent objects are also dropped
DROP TABLE IF EXISTS tasks CASCADE;
DROP TABLE IF EXISTS projects CASCADE;

-- ─── PROJECTS TABLE ─────────────────────────────────────────────

CREATE TABLE projects (
  id          SERIAL PRIMARY KEY,
  name        VARCHAR(100) NOT NULL,
  description TEXT DEFAULT '',
  created_at  TIMESTAMP DEFAULT NOW(),
  updated_at  TIMESTAMP DEFAULT NOW()
);

-- ─── TASKS TABLE ────────────────────────────────────────────────

CREATE TABLE tasks (
  id          SERIAL PRIMARY KEY,
  project_id  INTEGER NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
  title       VARCHAR(200) NOT NULL,
  completed   BOOLEAN DEFAULT false,
  created_at  TIMESTAMP DEFAULT NOW(),
  updated_at  TIMESTAMP DEFAULT NOW()
);
```

### Line-by-Line Breakdown

**`DROP TABLE IF EXISTS tasks CASCADE;`**
Deletes the `tasks` table if it already exists. This lets you re-run the schema file during development without errors. `CASCADE` means "also drop anything that depends on this table." We drop `tasks` first because it depends on `projects`.

**`CREATE TABLE projects (...)`**
Creates a new table called `projects` with the columns defined inside the parentheses.

**`id SERIAL PRIMARY KEY`**

> [!NOTE]
> **Technical**: `SERIAL` is a PostgreSQL shorthand that creates an auto-incrementing integer column. `PRIMARY KEY` enforces uniqueness and creates an index for fast lookups.

> [!NOTE]
> **Plain English**: Every project gets a unique number (1, 2, 3, ...) assigned automatically. You never set the `id` yourself — the database picks the next available number. No two projects can have the same `id`.

**`name VARCHAR(100) NOT NULL`**

> [!NOTE]
> **Technical**: `VARCHAR(100)` is a variable-length character string with a maximum of 100 characters. `NOT NULL` is a constraint that prevents this column from being empty.

> [!NOTE]
> **Plain English**: The project name is text, up to 100 characters. It's required — you can't create a project without a name.

**`description TEXT DEFAULT ''`**

> [!NOTE]
> **Technical**: `TEXT` is an unlimited-length string type. `DEFAULT ''` sets an empty string as the default when no value is provided.

> [!NOTE]
> **Plain English**: The description can be as long as you want. If you don't provide one, it defaults to an empty string.

**`created_at TIMESTAMP DEFAULT NOW()`**

> [!NOTE]
> **Technical**: `TIMESTAMP` stores date and time. `NOW()` is a PostgreSQL function that returns the current date and time. `DEFAULT NOW()` means the column is automatically set to the current time when a row is inserted.

> [!NOTE]
> **Plain English**: The database automatically records when each project was created. You don't need to set this yourself.

**`project_id INTEGER NOT NULL REFERENCES projects(id) ON DELETE CASCADE`**

This is the most important line. Let's break it apart:

| Part | What It Does |
|------|-------------|
| `project_id INTEGER` | A column that stores a number (the ID of a project) |
| `NOT NULL` | Every task must belong to a project |
| `REFERENCES projects(id)` | This number must match an existing project's `id` |
| `ON DELETE CASCADE` | If the project is deleted, delete all its tasks too |

> [!NOTE]
> **Technical**: This is a foreign key constraint. It establishes a referential integrity relationship between the `tasks` and `projects` tables. `ON DELETE CASCADE` implements cascading deletion — removing a parent row automatically removes all child rows.

> [!NOTE]
> **Plain English**: This is the "tasks belong to projects" rule, enforced by the database. If you try to create a task for project 99 and project 99 doesn't exist, the database says "no." If you delete project 1, the database automatically deletes all of project 1's tasks. You don't have to write code for this — the database handles it.

### CASCADE Visualized

```
BEFORE DELETE:                      AFTER: DELETE FROM projects WHERE id = 1

projects table:                     projects table:
┌────┬──────────┐                   ┌────┬──────────┐
│ id │ name     │                   │ id │ name     │
├────┼──────────┤                   ├────┼──────────┤
│  1 │ Website  │  ← DELETED       │  2 │ API      │
│  2 │ API      │                   └────┴──────────┘
└────┴──────────┘
                                    tasks table:
tasks table:                        ┌────┬────────────┬──────────────┐
┌────┬────────────┬──────────────┐  │ id │ project_id │ title        │
│ id │ project_id │ title        │  ├────┼────────────┼──────────────┤
├────┼────────────┼──────────────┤  │  3 │          2 │ Write docs   │
│  1 │          1 │ Design page  │  └────┴────────────┴──────────────┘
│  2 │          1 │ Build nav    │
│  3 │          2 │ Write docs   │  Tasks 1 and 2 were automatically
└────┴────────────┴──────────────┘  deleted because their project was deleted.
```

**`completed BOOLEAN DEFAULT false`**
A true/false value. New tasks start as incomplete (`false`). When the user checks the checkbox, we change it to `true`.

---

## Step 3: Run the Schema

> [!IMPORTANT]
> **You should be in:** `task-forge/`

```bash
psql taskforge < server/src/db/schema.sql
```

You should see output like:

```
DROP TABLE
DROP TABLE
CREATE TABLE
CREATE TABLE
```

### Verify the Tables Exist

```bash
psql taskforge
```

Inside psql:

```sql
-- List all tables
\dt

-- Describe the projects table (shows columns and types)
\d projects

-- Describe the tasks table
\d tasks

-- Quit
\q
```

You should see both tables with all their columns and constraints.

---

> [!TIP]
> **Session Break** — You've designed the schema with foreign keys and created both tables in PostgreSQL. Save your work and take a break.
> When you return, you'll practice SQL queries hands-on and build the database connection pool.

---

## Step 4: SQL Hands-On Tutorial

Before writing any TypeScript, practice SQL directly in `psql`. This is important — you should understand what your code will ask the database to do.

```bash
psql taskforge
```

### INSERT — Create Data

```sql
-- Create two projects
INSERT INTO projects (name, description)
VALUES ('Personal Website', 'Build my portfolio site');

INSERT INTO projects (name, description)
VALUES ('TaskForge', 'The app we are building right now');

-- Create tasks for the first project (project_id = 1)
INSERT INTO tasks (project_id, title)
VALUES (1, 'Design homepage layout');

INSERT INTO tasks (project_id, title)
VALUES (1, 'Choose color scheme');

INSERT INTO tasks (project_id, title)
VALUES (1, 'Write about me section');

-- Create tasks for the second project (project_id = 2)
INSERT INTO tasks (project_id, title)
VALUES (2, 'Set up database schema');

INSERT INTO tasks (project_id, title)
VALUES (2, 'Build CRUD API');
```

### SELECT — Read Data

```sql
-- Get all projects
SELECT * FROM projects;

-- Get all tasks
SELECT * FROM tasks;

-- Get tasks for project 1 only
SELECT * FROM tasks WHERE project_id = 1;

-- Get only incomplete tasks
SELECT * FROM tasks WHERE completed = false;

-- Get tasks ordered by creation date (newest first)
SELECT * FROM tasks ORDER BY created_at DESC;

-- Count tasks per project
SELECT project_id, COUNT(*) as task_count
FROM tasks
GROUP BY project_id;
```

### UPDATE — Modify Data

```sql
-- Mark a task as complete
UPDATE tasks SET completed = true, updated_at = NOW()
WHERE id = 1;

-- Update a project name
UPDATE projects SET name = 'Portfolio Website', updated_at = NOW()
WHERE id = 1;

-- Verify the changes
SELECT * FROM tasks WHERE id = 1;
SELECT * FROM projects WHERE id = 1;
```

### DELETE — Remove Data

```sql
-- Delete a single task
DELETE FROM tasks WHERE id = 5;

-- Delete a project (CASCADE will delete its tasks too)
DELETE FROM projects WHERE id = 2;

-- Verify: tasks for project 2 should be gone
SELECT * FROM tasks WHERE project_id = 2;
```

### RETURNING — Get Back What You Changed

PostgreSQL has a useful feature: you can ask it to return the rows affected by INSERT, UPDATE, or DELETE:

```sql
-- Insert and get back the created row (with its auto-generated id)
INSERT INTO projects (name, description)
VALUES ('New Project', 'Testing RETURNING')
RETURNING *;

-- Update and get back the changed row
UPDATE tasks SET completed = true WHERE id = 2
RETURNING *;

-- Delete and get back what was deleted
DELETE FROM tasks WHERE id = 2
RETURNING *;
```

We'll use `RETURNING *` extensively in our API — it saves us from having to do a separate SELECT after every INSERT or UPDATE.

### Clean Up

Reset the database to a clean state before building the app:

```sql
\q
```

```bash
psql taskforge < server/src/db/schema.sql
```

This drops and recreates the tables, removing the practice data.

---

## Step 5: Build the Database Connection

Now we connect Express to PostgreSQL using the `pg` library.

In VS Code, create a new file at `server/src/db/index.ts`.

Add this code to `server/src/db/index.ts`:

```typescript
import { Pool } from 'pg';
import dotenv from 'dotenv';

// Load environment variables from .env file
dotenv.config();

// Create a connection pool
const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
});

// Test the connection on startup
pool.query('SELECT NOW()')
  .then(() => {
    console.log('Connected to PostgreSQL');
  })
  .catch((err) => {
    console.error('Failed to connect to PostgreSQL:', err.message);
    process.exit(1);
  });

export default pool;
```

### What Is a Connection Pool?

> [!NOTE]
> **Technical**: A connection pool maintains a set of reusable database connections. Instead of opening and closing a connection for each query (expensive), the pool keeps connections open and lends them out as needed. The `pg` library's `Pool` class manages this automatically.

> [!NOTE]
> **Plain English**: Opening a connection to a database takes time (like dialing a phone number). A pool keeps several phone lines open permanently. When your code needs to make a query, it borrows an open line, uses it, and returns it. This is much faster than dialing a new number every time.

```
WITHOUT POOLING:                    WITH POOLING:
─────────────────                   ──────────────

Request 1 → Open connection         Request 1 → Borrow connection ─┐
          → Query                              → Query              │
          → Close connection                   → Return connection ─┤ Pool of
                                                                    │ connections
Request 2 → Open connection         Request 2 → Borrow connection ─┤ (reused)
          → Query                              → Query              │
          → Close connection                   → Return connection ─┘

Each open/close: ~20-50ms           Borrow/return: ~0.1ms
```

### Code Breakdown

**`import { Pool } from 'pg'`**
Imports the Pool class from the `pg` library. This is the recommended way to use `pg` in a web server.

**`dotenv.config()`**
Reads the `.env` file and puts its contents into `process.env`. After this line, `process.env.DATABASE_URL` contains `postgresql://localhost:5432/taskforge`.

**`new Pool({ connectionString: process.env.DATABASE_URL })`**
Creates a pool of database connections using the URL from the environment. The pool starts with 0 connections and creates them on demand, up to a default maximum of 10.

**`pool.query('SELECT NOW()')`**
Sends a test query to PostgreSQL. `SELECT NOW()` returns the current time — it's the simplest possible query that proves the connection works.

**`process.exit(1)`**
If the connection fails, the server exits with error code 1. There's no point running the server if it can't reach the database.

### How pool.query Works

When you call `pool.query(sql, params)` later in your routes:

1. The pool finds an available connection (or creates one)
2. It sends the SQL query to PostgreSQL
3. PostgreSQL executes the query and returns results
4. The pool receives the results and returns the connection to the pool
5. You get back a result object with a `rows` array

```typescript
// Example (we'll use this pattern in routes):
const result = await pool.query('SELECT * FROM projects');
// result.rows = [{ id: 1, name: 'Website', ... }, { id: 2, name: 'API', ... }]
```

---

## Step 6: Migrations (Conceptual)

In this project, we use a single `schema.sql` file that we run manually. This works fine for learning. But real production projects use **migration tools**.

> [!NOTE]
> **Technical**: Database migrations are versioned scripts that incrementally modify a database schema. Each migration file represents a single change (add a table, add a column, change a type). Tools like Knex.js, Prisma, or node-pg-migrate track which migrations have run and apply them in order.

> [!NOTE]
> **Plain English**: Instead of one big file that recreates everything, migrations are like Git commits for your database. Each one makes a small change: "add a `priority` column to tasks," "rename `name` to `title` in projects." The migration tool remembers which changes have been applied so you never run the same one twice.

**Why we're not using a migration tool**: It adds complexity that would distract from learning SQL and database fundamentals. In Level 4, you may encounter migration tools in the context of testing. For now, `schema.sql` does the job.

---

## Step 7: Test the Connection

Let's verify Express can connect to PostgreSQL.

Create a temporary test file. In VS Code, create `server/src/index.ts`:

```typescript
import dotenv from 'dotenv';
dotenv.config();

import pool from './db';

async function testConnection() {
  try {
    const result = await pool.query('SELECT NOW() as current_time');
    console.log('Database time:', result.rows[0].current_time);

    const tables = await pool.query(`
      SELECT table_name
      FROM information_schema.tables
      WHERE table_schema = 'public'
    `);
    console.log('Tables:', tables.rows.map(r => r.table_name));

    process.exit(0);
  } catch (err) {
    console.error('Connection test failed:', err);
    process.exit(1);
  }
}

testConnection();
```

Run it:

> [!IMPORTANT]
> **You should be in:** `task-forge/`

```bash
cd server
```

> [!IMPORTANT]
> **You are now in:** `task-forge/server/`

```bash
npx ts-node src/index.ts
```

Expected output:

```
Connected to PostgreSQL
Database time: 2026-03-02T...
Tables: [ 'projects', 'tasks' ]
```

If you see this, your connection works. If you see an error, check:
- Is PostgreSQL running? (`brew services list`)
- Does the `taskforge` database exist? (`psql -l` to list databases)
- Is your `.env` file correct? (`cat .env`)

```bash
cd ..
```

> [!IMPORTANT]
> **You are now in:** `task-forge/` (back to the project root)

---

## Step 8: Commit

> [!IMPORTANT]
> **You should be in:** `task-forge/`

```bash
git add server/src/db/
git commit -m "feat: add database schema and connection pool"
```

---

```
┌──────────────────────────────────────────────────────────────────┐
│                    WHAT EXISTS NOW                                 │
│                                                                  │
│   ┌──────────────┐    ┌──────────────┐    ┌──────────────┐     │
│   │  FRONTEND     │    │  BACKEND      │    │  DATABASE ✓  │     │
│   │  (not built)  │    │  (partially)  │    │              │     │
│   │              │    │              │    │  projects    │     │
│   │              │    │  pool.query  │───▶│  tasks       │     │
│   │              │    │  connected ✓ │    │              │     │
│   │  NEXT: 05    │    │  NEXT: 04    │    │  Connected   │     │
│   └──────────────┘    └──────────────┘    └──────────────┘     │
│                                                                  │
│   The database has tables. The backend can connect.              │
│   Next: build the API routes that use SQL queries.              │
└──────────────────────────────────────────────────────────────────┘
```

---

> [!TIP]
> ## Spatial Check-In

1. **What is a foreign key?**

<details><summary>Answer</summary>

A column that references the primary key of another table. `tasks.project_id` references `projects.id`, creating a relationship between the two tables.

</details>

2. **What does ON DELETE CASCADE do?**

<details><summary>Answer</summary>

When a parent row is deleted, all child rows that reference it are automatically deleted too. Deleting a project automatically deletes all its tasks.

</details>

3. **What does SERIAL PRIMARY KEY do?**

<details><summary>Answer</summary>

Creates an auto-incrementing integer column that uniquely identifies each row. The database assigns the next number automatically.

</details>

4. **What is a connection pool?**

<details><summary>Answer</summary>

A set of reusable database connections. Instead of opening and closing connections for each query (slow), the pool keeps connections ready to use (fast).

</details>

5. **What does RETURNING * do in PostgreSQL?**

<details><summary>Answer</summary>

Returns the affected rows after an INSERT, UPDATE, or DELETE. This saves a separate SELECT query to get the data back.

</details>

6. **Why do we load dotenv before importing the pool?**

<details><summary>Answer</summary>

The pool reads `process.env.DATABASE_URL` when it's created. If dotenv hasn't loaded the `.env` file yet, the URL will be undefined and the connection will fail.

</details>

---

| | | |
|:---|:---:|---:|
| [← 02 — Project Setup](../02-project-setup/) | [Level 2 Overview](../) | [04 — Building the Backend →](../04-backend/) |

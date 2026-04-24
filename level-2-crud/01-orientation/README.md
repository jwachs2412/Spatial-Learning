# Step 1 — Orientation: From Memory to Database

> **Prerequisites:** You must have completed Level 1 (DevPulse) before starting Level 2. This level assumes you can build a React + Express application and deploy it.

## What Changed — The Problem with In-Memory Data

In Level 1, your data lived in a JavaScript array:

```typescript
const entries: MoodEntry[] = [];  // Gone when the server restarts
```

This has three fatal problems:
1. **Data loss** — Restart the server, lose everything
2. **No querying** — Want entries from last week? Loop through the entire array
3. **No relationships** — How do you link tasks to projects?

A database solves all three.

---

## What Is a Database?

> **Key Concept: Database**
> A database is a program that stores, organizes, and retrieves data efficiently. It runs as a separate process from your backend — think of it as a specialized filing cabinet that your server talks to using a language called SQL. Unlike an in-memory array, database data survives restarts, crashes, and even server replacements.

### Why Not Just Use Files?

| Approach | Speed | Querying | Concurrency | Relationships | Survives Restart |
|----------|-------|----------|-------------|---------------|-----------------|
| In-memory array | Fastest | Loop everything | Problems | Manual | No |
| JSON file | Slow | Parse entire file | Lock issues | Manual | Yes |
| **PostgreSQL** | **Fast** | **SQL queries** | **Built-in** | **Built-in** | **Yes** |

---

## What Is SQL?

> **Key Concept: SQL (Structured Query Language)**
> SQL is the language you use to talk to relational databases. It lets you create tables, insert data, query data, update data, and delete data. Think of it like a very specific set of commands for a librarian: "Find all books by this author, published after 2020, sorted by title."

```sql
-- Create a table (the structure)
CREATE TABLE projects (
  id SERIAL PRIMARY KEY,
  name VARCHAR(100) NOT NULL
);

-- Insert data
INSERT INTO projects (name) VALUES ('My Project');

-- Query data
SELECT * FROM projects WHERE name = 'My Project';

-- Update data
UPDATE projects SET name = 'Renamed' WHERE id = 1;

-- Delete data
DELETE FROM projects WHERE id = 1;
```

### Reading Those Five SQL Statements

SQL is almost plain English. Every statement ends in `;`. Lines starting with `--` are comments (the database ignores them). Let's unpack each one.

- `CREATE TABLE projects ( ... );`
  - "Make a new table named `projects`." Everything in parentheses is the column list.
  - `id SERIAL PRIMARY KEY` — a column named `id` with type `SERIAL` (auto-incrementing integer that starts at 1 and ticks up). `PRIMARY KEY` means "this is the unique identifier for each row; no two rows can share the same `id`, and it can't be null." The database builds an index on it automatically.
  - `name VARCHAR(100) NOT NULL` — a column named `name` of type `VARCHAR(100)` (a variable-length string up to 100 characters). `NOT NULL` means "every row must have a value here; trying to insert without one is an error."

- `INSERT INTO projects (name) VALUES ('My Project');`
  - "Add a new row to `projects`." The `(name)` list tells PostgreSQL which columns you're filling in. The `VALUES (...)` list provides the actual data in the same order.
  - We don't list `id` because `SERIAL` generates it automatically.
  - Strings in SQL use single quotes: `'My Project'`. Double quotes mean something different (column/table names) — not a stylistic choice.

- `SELECT * FROM projects WHERE name = 'My Project';`
  - "Read rows from `projects`."
  - `*` means "all columns."
  - `WHERE name = 'My Project'` filters to rows whose `name` column equals that string. Without `WHERE`, you'd get every row in the table.

- `UPDATE projects SET name = 'Renamed' WHERE id = 1;`
  - "Modify rows in `projects`. Set the `name` column to `'Renamed'`, but only for rows where `id = 1`."
  - **The `WHERE` clause is critical.** If you forget it, `UPDATE projects SET name = 'Renamed'` would rename every project in the table.

- `DELETE FROM projects WHERE id = 1;`
  - "Remove rows from `projects` where `id = 1`."
  - Same warning: `DELETE FROM projects` without a `WHERE` deletes every row. PostgreSQL will happily do it — there's no "are you sure?" confirmation.

Notice: SQL maps directly to CRUD:

| CRUD Operation | SQL Command | HTTP Method |
|---------------|-------------|-------------|
| **C**reate | `INSERT` | POST |
| **R**ead | `SELECT` | GET |
| **U**pdate | `UPDATE` | PUT/PATCH |
| **D**elete | `DELETE` | DELETE |

---

## Three-Tier Architecture

In Level 1, you had two tiers: frontend and backend. Now you add a third:

```
┌──────────────┐      ┌──────────────┐      ┌──────────────┐
│  FRONTEND    │      │  BACKEND     │      │  DATABASE    │
│  (React)     │─────▶│  (Express)   │─────▶│  (PostgreSQL)│
│              │◀─────│              │◀─────│              │
│              │ HTTP │              │ SQL  │              │
│  Presentation│      │  Application │      │  Data        │
│  Layer       │      │  Layer       │      │  Layer       │
└──────────────┘      └──────────────┘      └──────────────┘
```

**Key insight:** Each tier only talks to its neighbor. The frontend never talks directly to the database. The backend is the gatekeeper — it validates requests, enforces business rules, and translates between HTTP and SQL.

### 🧠 Think About It

Why shouldn't the frontend query the database directly?

<details>
<summary>Answer</summary>

Three reasons:

1. **Security** — Database credentials would be in the browser's JavaScript, visible to anyone. They could run `DELETE FROM users` directly.
2. **Validation** — There's no layer to enforce business rules (like "project names must be unique" or "energy must be 1-5").
3. **Abstraction** — If you change your database (PostgreSQL → MySQL), every frontend component would need to change. The backend abstracts this away.

</details>

---

## TaskForge — What We're Building

A project management tool with two related resources:

```
projects (parent)
├── Project A
│   ├── Task 1 ✓
│   ├── Task 2
│   └── Task 3
└── Project B
    ├── Task 4
    └── Task 5 ✓
```

### API Overview — 10 Endpoints

| Method | Endpoint | Action |
|--------|----------|--------|
| GET | /api/projects | List all projects |
| POST | /api/projects | Create a project |
| GET | /api/projects/:id | Get project with tasks |
| PUT | /api/projects/:id | Update a project |
| DELETE | /api/projects/:id | Delete project (and its tasks) |
| GET | /api/projects/:projectId/tasks | List tasks for a project |
| POST | /api/projects/:projectId/tasks | Create a task in a project |
| PUT | /api/tasks/:id | Update a task |
| DELETE | /api/tasks/:id | Delete a task |
| GET | /api/health | Health check (with DB status) |

---

## 🧠 Spatial Check-In

1. In Level 1, what happened to your mood entries when you restarted the backend? Why won't that happen in Level 2?

2. If a user creates a project with 5 tasks, then deletes the project, what should happen to the tasks? (Think about this before Level 3's database design.)

3. Why does the frontend never talk to the database directly?

<details>
<summary>Check Your Answers</summary>

1. **They disappeared** because they were stored in a JavaScript array (in-memory). In Level 2, data lives in PostgreSQL — a separate process that keeps data on disk. Even if the server crashes, the database retains everything.

2. **The tasks should be deleted too.** This is called a *cascade delete* — deleting a parent automatically deletes its children. You'll implement this with `ON DELETE CASCADE` in the database schema.

3. **Security, validation, and abstraction.** Putting database credentials in the browser exposes them to anyone. The backend is the gatekeeper that validates input, enforces rules, and shields the database from direct access.

</details>

---

> **Session Break** — You've completed the orientation.
> When you return, you'll scaffold the project in [Step 2 — Project Setup](../02-project-setup/).

---

| | |
|:---|---:|
| [← Level 2 Overview](../) | [Step 2 — Project Setup →](../02-project-setup/) |

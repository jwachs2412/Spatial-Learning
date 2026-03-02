`Level 2` **Step 1 of 7** — Orientation

# 01 — Orientation: From Memory to Database

> [!WARNING]
> ## Prerequisites Gate
>
> Level 2 requires a **completed and deployed** Level 1 project. Before continuing, you must have all three of the following:
>
> 1. **GitHub repository URL** — Your DevPulse code is pushed to GitHub and the repo is public
> 2. **Frontend URL** — Your DevPulse frontend is deployed on Vercel and loads in a browser
> 3. **Backend URL** — Your DevPulse backend health check works: `https://your-api.onrender.com/api/health` returns `{"status":"ok"}`
>
> If any of these are missing, go back to [Level 1 — Deployment](../../level-1-foundations/06-deployment/) and complete it first.

### Skills Checklist

You should be able to do all of the following from Level 1. If any feel shaky, review the relevant section before continuing.

- [ ] Explain the client-server model (what runs where)
- [ ] Create a React component with useState
- [ ] Build Express routes that handle GET and POST requests
- [ ] Use fetch() to send HTTP requests from the frontend
- [ ] Explain what CORS is and why it exists
- [ ] Use Git to commit and push code
- [ ] Deploy a frontend to Vercel and a backend to Render

All checked? Let's add the missing layer.

---

## What Changed Since Level 1

In Level 1, your architecture looked like this:

```
Level 1: Two-Tier Architecture
─────────────────────────────────

┌─────────────────┐         ┌──────────────────────────┐
│    FRONTEND      │  HTTP   │        BACKEND            │
│    (React)       │ ◀─────▶ │      (Express)            │
│                 │         │                          │
│ Shows the UI    │         │  let entries = [];       │
│                 │         │  ← Data lives in memory  │
│                 │         │  ← Lost on restart       │
└─────────────────┘         └──────────────────────────┘
```

The problem was clear: restart the server, lose all data. That's because an in-memory array only exists while the program runs.

Level 2 adds the third tier:

```
Level 2: Three-Tier Architecture
──────────────────────────────────

┌──────────────┐       ┌──────────────┐       ┌──────────────┐
│   FRONTEND    │ HTTP  │   BACKEND     │  SQL  │   DATABASE    │
│   (React)     │◀────▶│   (Express)   │◀────▶│ (PostgreSQL)  │
│              │       │              │       │              │
│ Shows the UI │       │ Handles      │       │ Stores data  │
│ Sends        │       │ requests     │       │ permanently  │
│ requests     │       │ Runs logic   │       │ on disk      │
│              │       │ Sends SQL    │       │              │
└──────────────┘       └──────────────┘       └──────────────┘
  BROWSER                SERVER                 SERVER
  (User's machine)       (Your machine/cloud)   (Your machine/cloud)
```

The backend no longer stores data itself. It asks the database to store it. The database writes data to disk — it survives restarts, crashes, and deployments.

---

## What Is a Database?

> [!NOTE]
> **Technical**: A database is a structured, persistent data store managed by a Database Management System (DBMS). A relational database (like PostgreSQL) organizes data into tables with rows and columns, enforces constraints, supports transactions, and provides a query language (SQL) for data manipulation.

> [!NOTE]
> **Plain English**: A database is a program whose only job is to store data and retrieve it quickly. Think of it as a very smart spreadsheet that runs on a server. It has tables (like sheets), rows (like entries), and columns (like fields). When you ask it for data, it finds it fast — even if there are millions of rows. Most importantly, it saves data to disk, so nothing is lost when the power goes out.

### Why Not Just Use Files?

You could write data to a `.json` file on disk. Some simple apps do this. But databases solve problems that files can't:

| Problem | File-Based Storage | Database |
|---------|-------------------|----------|
| Two users write at the same time | Data corruption | Handles concurrency safely |
| Find all tasks completed this week | Read entire file, filter in code | One fast query: `WHERE completed = true AND created_at > ...` |
| Delete a project and all its tasks | Complex manual logic | `ON DELETE CASCADE` — automatic |
| 100,000 records | Slow — loads entire file | Fast — uses indexes |
| Data integrity | No enforcement | Constraints prevent bad data |

---

## What Is SQL?

> [!NOTE]
> **Technical**: SQL (Structured Query Language) is a declarative language for managing and querying relational databases. It provides statements for data definition (CREATE, ALTER, DROP), data manipulation (SELECT, INSERT, UPDATE, DELETE), and data control (GRANT, REVOKE).

> [!NOTE]
> **Plain English**: SQL is the language you use to talk to a database. You write a command like "give me all tasks where completed is false" and the database figures out the fastest way to get that data. You describe *what* you want, not *how* to get it.

### SQL Looks Like This

```sql
-- Create a task
INSERT INTO tasks (project_id, title) VALUES (1, 'Set up database');

-- Get all tasks for project 1
SELECT * FROM tasks WHERE project_id = 1;

-- Mark a task as complete
UPDATE tasks SET completed = true WHERE id = 5;

-- Delete a task
DELETE FROM tasks WHERE id = 5;
```

You'll learn every one of these in detail in the Database lesson.

---

## CRUD — The Four Operations

Almost every application does four things with data. This pattern is called CRUD:

```
┌───────────────────────────────────────────────────────────────────┐
│                         CRUD                                       │
│                                                                   │
│   Operation    HTTP Method    SQL Statement    What It Does       │
│   ─────────    ───────────    ─────────────    ────────────       │
│   Create       POST           INSERT           Add new data       │
│   Read         GET            SELECT           Retrieve data      │
│   Update       PUT            UPDATE           Modify data        │
│   Delete       DELETE         DELETE           Remove data        │
│                                                                   │
└───────────────────────────────────────────────────────────────────┘
```

In Level 1, you only had **Create** (POST) and **Read** (GET). Level 2 adds **Update** (PUT) and **Delete** (DELETE) — completing the full CRUD cycle.

### How It Flows Through the Stack

```
User clicks "Delete Task"
       │
       ▼
React sends:  DELETE /api/tasks/5
       │
       ▼
Express route handler runs:
       │  const result = await pool.query('DELETE FROM tasks WHERE id = $1', [5]);
       │
       ▼
PostgreSQL executes the SQL:
       │  Removes the row from the tasks table on disk
       │
       ▼
Express sends back:  204 No Content
       │
       ▼
React removes the task from state → UI updates
```

Each layer has one job. React handles UI. Express handles HTTP. PostgreSQL handles data. This separation is what makes the system maintainable.

---

## What Is TaskForge?

TaskForge is a project task manager. Here's what you'll be able to do:

**Projects**
- Create a project with a name and description
- See a list of all projects
- Edit a project's name or description
- Delete a project (and all its tasks)

**Tasks** (within a project)
- Add tasks to a project
- See all tasks for a project
- Mark a task as complete or incomplete (checkbox toggle)
- Edit a task's title
- Delete a task

**The key difference from Level 1**: Data persists. Create a project, add tasks, close your browser, restart the server, come back — everything is still there. This is what databases give you.

---

## API Overview

TaskForge has 10 endpoints plus a health check:

| Method | Endpoint | Purpose |
|--------|----------|---------|
| GET | `/api/projects` | List all projects |
| POST | `/api/projects` | Create a project |
| GET | `/api/projects/:id` | Get one project with its tasks |
| PUT | `/api/projects/:id` | Update a project |
| DELETE | `/api/projects/:id` | Delete a project and all its tasks |
| GET | `/api/projects/:projectId/tasks` | List tasks for a project |
| POST | `/api/projects/:projectId/tasks` | Create a task in a project |
| PUT | `/api/tasks/:id` | Update a task |
| DELETE | `/api/tasks/:id` | Delete a task |
| GET | `/api/health` | Health check with DB connectivity |

Notice how tasks are **nested under projects** in the URL: `/api/projects/1/tasks`. This is called a nested resource — it shows that tasks belong to a project. This is a RESTful pattern you'll see everywhere.

---

> [!TIP]
> ## Spatial Check-In
> Before moving on, make sure you can answer these questions. These are the mental models you'll use for the rest of this level.

1. **What are the three tiers in Level 2's architecture?**

<details><summary>Answer</summary>

Frontend (React in the browser), Backend (Express on the server), Database (PostgreSQL on the server)

</details>

2. **What does CRUD stand for, and what HTTP methods map to each operation?**

<details><summary>Answer</summary>

Create → POST, Read → GET, Update → PUT, Delete → DELETE

</details>

3. **Why can't we just use an in-memory array like Level 1?**

<details><summary>Answer</summary>

In-memory data is lost when the server restarts. A database stores data permanently on disk.

</details>

4. **What language do we use to talk to a relational database?**

<details><summary>Answer</summary>

SQL (Structured Query Language)

</details>

5. **Why are tasks nested under projects in the API URL?**

<details><summary>Answer</summary>

Tasks belong to a project. The URL `/api/projects/1/tasks` shows this relationship — it's a RESTful pattern for nested resources.

</details>

6. **What does each layer do? (Frontend, Backend, Database)**

<details><summary>Answer</summary>

Frontend: Shows UI, collects user input, sends HTTP requests. Backend: Receives HTTP requests, runs business logic, sends SQL queries. Database: Stores data permanently, executes SQL, returns results.

</details>

---

| | | |
|:---|:---:|---:|
| | [Level 2 Overview](../) | [02 — Project Setup →](../02-project-setup/) |

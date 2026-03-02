# Level 2 — TaskForge: Project Task Manager

## What You Will Build

TaskForge is a project task manager where you create projects, add tasks to each project, mark tasks complete, edit them, and delete them. All data persists in a PostgreSQL database — restart the server, refresh the page, close your laptop, come back tomorrow — your data is still there.

It has a React + TypeScript frontend, a Node.js + Express backend, and a PostgreSQL database. All three are deployed to the internet.

## Why This App

Level 1's DevPulse stored data in memory — it vanished when the server restarted. That was intentional: you needed to understand client-server communication without the complexity of a database.

Now you add the missing layer. TaskForge forces you to:

1. Design a database schema with related tables
2. Write SQL queries for every CRUD operation
3. Build a full REST API (10 endpoints, not just 2)
4. Handle errors properly with middleware
5. Manage environment variables for secrets
6. Deploy a three-tier application (frontend + backend + database)

## What You Will Learn

By the end of Level 2, you will understand:

- What a relational database is and why it exists
- How to design tables with primary keys and foreign keys
- How to write SQL (SELECT, INSERT, UPDATE, DELETE)
- How CRUD maps to HTTP methods and SQL statements
- How connection pooling works
- How error handling middleware catches and formats errors
- How environment variables keep secrets out of code
- How to deploy a full-stack app with a managed database

## Level 2 Architecture (The Big Picture)

```
┌──────────────────────────────────────────────────────────────────────┐
│                        YOUR APPLICATION                               │
│                                                                       │
│   ┌────────────────────┐  ┌────────────────────┐  ┌──────────────┐  │
│   │     FRONTEND        │  │     BACKEND         │  │   DATABASE    │  │
│   │   (React + TS)      │  │  (Node + Express)   │  │ (PostgreSQL)  │  │
│   │                    │  │                    │  │              │  │
│   │ ┌────────────────┐ │  │ ┌────────────────┐ │  │ ┌──────────┐ │  │
│   │ │  App Component  │ │  │ │ Express Server │ │  │ │ projects │ │  │
│   │ │                │ │  │ │                │ │  │ │ table    │ │  │
│   │ │ ProjectForm    │ │  │ │ Project Routes │ │  │ ├──────────┤ │  │
│   │ │ ProjectList    │ │  │ │ Task Routes    │ │  │ │ tasks    │ │  │
│   │ │ ProjectDetail  │ │  │ │ Error Handler  │ │  │ │ table    │ │  │
│   │ │ TaskForm       │ │  │ │ Health Check   │ │  │ └──────────┘ │  │
│   │ │ TaskList       │ │  │ └────────────────┘ │  │              │  │
│   │ │ TaskItem       │ │  │                    │  │ Data persists │  │
│   │ └────────────────┘ │  │  10 API endpoints  │  │ permanently   │  │
│   │                    │  │                    │  │              │  │
│   │ Runs in: BROWSER   │  │ Runs on: SERVER    │  │ Runs on:     │  │
│   │ Port: 5173 (dev)   │  │ Port: 3001 (dev)   │  │ Port: 5432   │  │
│   │ Deploy: Vercel     │  │ Deploy: Render     │  │ Deploy:      │  │
│   └────────────────────┘  └────────────────────┘  │ Render DB    │  │
│                                                    └──────────────┘  │
│                                                                       │
│          Connected by HTTP (REST API) and SQL (pg driver)             │
└──────────────────────────────────────────────────────────────────────┘
```

## Data Flow (How Information Moves)

When a user creates a task in a project, here's exactly what happens:

```
1. User fills out the task form and clicks "Add Task"
   │
2. React's onChange handlers store form data in component state
   │
3. On submit, the frontend sends an HTTP POST request:
   │  POST http://localhost:3001/api/projects/1/tasks
   │  Body: { title: "Set up database schema" }
   │
4. Express receives the request
   │  → The route handler extracts the body
   │  → Validates the data
   │  → Sends a SQL query: INSERT INTO tasks (project_id, title) VALUES ($1, $2)
   │  → PostgreSQL stores the row permanently on disk
   │  → Sends back: { id: 1, project_id: 1, title: "...", completed: false, ... }
   │
5. React receives the response
   │  → Updates component state with the new task
   │  → React re-renders the TaskList with the new data
   │
6. User sees their new task on screen
   │
7. User restarts the server, refreshes the page — the task is still there
```

## Folder Structure (What Goes Where and Why)

```
task-forge/
├── client/                    ← FRONTEND (everything the browser runs)
│   ├── src/
│   │   ├── components/        ← React components (UI building blocks)
│   │   │   ├── ProjectForm.tsx
│   │   │   ├── ProjectList.tsx
│   │   │   ├── ProjectDetail.tsx
│   │   │   ├── TaskForm.tsx
│   │   │   ├── TaskList.tsx
│   │   │   └── TaskItem.tsx
│   │   ├── types/             ← TypeScript type definitions
│   │   │   └── index.ts
│   │   ├── services/          ← Functions that talk to the backend
│   │   │   └── api.ts
│   │   ├── App.tsx            ← Root component (manages all state)
│   │   ├── App.css            ← Styles
│   │   └── main.tsx           ← Entry point
│   ├── index.html
│   ├── package.json
│   ├── tsconfig.json
│   └── vite.config.ts
│
├── server/                    ← BACKEND (everything the server runs)
│   ├── src/
│   │   ├── db/                ← Database connection and schema
│   │   │   ├── index.ts       ← Connection pool configuration
│   │   │   └── schema.sql     ← Table definitions
│   │   ├── routes/            ← API route definitions
│   │   │   ├── projects.ts    ← /api/projects routes (5 endpoints)
│   │   │   └── tasks.ts       ← /api/tasks routes (4 endpoints)
│   │   ├── middleware/        ← Express middleware
│   │   │   └── errorHandler.ts
│   │   ├── types/             ← Backend TypeScript types
│   │   │   └── index.ts
│   │   └── index.ts           ← Server entry point
│   ├── package.json
│   ├── tsconfig.json
│   └── .env                   ← Environment variables (NEVER committed)
│
├── .gitignore
└── README.md
```

**What changed from Level 1?**
- `server/src/db/` — new folder for database connection and schema
- `server/src/middleware/` — new folder for error handling middleware
- `server/.env` — environment variables file (database URL, secrets)
- More components (6 vs 3) — CRUD requires more UI
- More routes (10 vs 2) — full CRUD requires more endpoints

## Learning Path

Work through these sections in order:

1. **[01 — Orientation](./01-orientation/)** — Prerequisites gate, three-tier architecture, what CRUD means
2. **[02 — Project Setup](./02-project-setup/)** — Scaffold the project, install PostgreSQL, configure environment
3. **[03 — Database Design](./03-database/)** — Schema design, SQL fundamentals, connection pooling
4. **[04 — Building the Backend](./04-backend/)** — Full CRUD API with error handling middleware
5. **[05 — Building the Frontend](./05-frontend/)** — React components for projects and tasks
6. **[06 — Deployment](./06-deployment/)** — Deploy with a managed database to the internet
7. **[07 — Growth Review](./07-growth/)** — What you learned and how to talk about it

---

| | |
|:---|---:|
| [← Level 1 — DevPulse](../level-1-foundations/) | [01 — Start Here →](./01-orientation/) |

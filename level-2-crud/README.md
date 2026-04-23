# Level 2 — TaskForge (Project Task Manager)

## What You Will Build

TaskForge is a project task manager where you create projects, add tasks to each project, mark tasks complete, edit them, and delete them. All data persists in a PostgreSQL database — restart the server, refresh the page, come back tomorrow — your data is still there.

### Why This App

CRUD operations (Create, Read, Update, Delete) are the backbone of almost every web application. This level forces you to understand how data moves from a form, through an API, into a database, and back to the screen. It introduces persistent state that survives server restarts — the #1 limitation of Level 1.

### What Changed From Level 1

| Level 1 (DevPulse) | Level 2 (TaskForge) |
|-------|-------|
| 2 tiers (frontend + backend) | 3 tiers (frontend + backend + **database**) |
| Data in memory (lost on restart) | Data in PostgreSQL (**persists forever**) |
| 2 API endpoints | **10 API endpoints** |
| Create + Read only | Full **CRUD** (Create, Read, Update, Delete) |
| No relationships | Projects → Tasks (**one-to-many**) |
| No error handling middleware | **Centralized error handling** |

---

## Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                                                                  │
│   ┌──────────────┐      ┌──────────────┐      ┌──────────────┐  │
│   │   FRONTEND   │      │   BACKEND    │      │   DATABASE   │  │
│   │   (React)    │─────▶│  (Express)   │─────▶│ (PostgreSQL) │  │
│   │              │◀─────│              │◀─────│              │  │
│   │  Forms       │ HTTP │  Validation  │ SQL  │  Tables      │  │
│   │  Lists       │      │  Routes      │      │  Rows        │  │
│   │  State mgmt  │      │  Error       │      │  Relations   │  │
│   │              │      │  handling    │      │              │  │
│   └──────────────┘      └──────────────┘      └──────────────┘  │
│                                                                  │
│   BROWSER                SERVER                 SERVER           │
│   (user's machine)       (cloud)                (cloud)          │
└──────────────────────────────────────────────────────────────────┘
```

---

## Folder Structure

```
task-forge/
├── client/
│   ├── src/
│   │   ├── components/
│   │   │   ├── ProjectForm.tsx
│   │   │   ├── ProjectList.tsx
│   │   │   ├── ProjectDetail.tsx
│   │   │   ├── TaskForm.tsx
│   │   │   ├── TaskList.tsx
│   │   │   └── TaskItem.tsx
│   │   ├── services/
│   │   │   └── api.ts
│   │   ├── types/
│   │   │   └── index.ts
│   │   ├── App.tsx
│   │   └── App.css
│   └── package.json
│
├── server/
│   ├── src/
│   │   ├── db/
│   │   │   ├── index.ts           ← Database connection
│   │   │   └── schema.sql         ← Table definitions
│   │   ├── routes/
│   │   │   ├── projects.ts        ← 5 project endpoints
│   │   │   └── tasks.ts           ← 4 task endpoints + health
│   │   ├── middleware/
│   │   │   └── errorHandler.ts    ← Centralized error handling
│   │   ├── types/
│   │   │   └── index.ts
│   │   └── index.ts
│   ├── package.json
│   └── tsconfig.json
│
├── .gitignore
└── README.md
```

---

## Session Guide

| Session | Lesson | What You Do |
|---------|--------|-------------|
| 1 | [01 — Orientation](./01-orientation/) | Understand databases, CRUD, three-tier architecture |
| 2 | [02 — Project Setup](./02-project-setup/) | Scaffold project, install PostgreSQL, configure environment |
| 3-4 | [03 — Database](./03-database/) | Design schema, write SQL, connect to PostgreSQL |
| 4-5 | [04 — Backend](./04-backend/) | Build 10 CRUD endpoints with validation and error handling |
| 6-7 | [05 — Frontend](./05-frontend/) | Build 6 React components for full CRUD UI |
| 8-9 | [06 — Deployment](./06-deployment/) | Deploy frontend, backend, and database |
| 9 | [07 — Growth](./07-growth/) | Review skills, prepare for interviews |

**Estimated total: 9–12 sessions (30–60 min each)**

---

| | |
|:---|---:|
| [← Level 1: DevPulse](../level-1-foundations/) | [Step 1 — Orientation →](./01-orientation/) |

`Level 2` **Step 2 of 7** — Project Setup

# 02 — Project Setup: Scaffolding with a Database

## Spatial Orientation

We're creating the same two-project structure as Level 1, with one critical addition — a database:

```
task-forge/              ← The project root
├── client/              ← Frontend project (React + TypeScript + Vite)
├── server/              ← Backend project (Node + Express + TypeScript + pg)
│   └── .env             ← Database connection string (NEW — never committed)
└── PostgreSQL           ← Running as a service on your machine (port 5432)
```

**Where are we in the stack?** We're building the scaffolding again. Same pattern as Level 1, but now the backend connects to a real database.

---

## Step 1: Create the Project Folder

Open your terminal. Navigate to the folder where you keep your projects (the same place you created `dev-pulse/` in Level 1).

```bash
# You should be in your projects folder (e.g., ~/Desktop/Sites)
mkdir task-forge
cd task-forge
```

You are now inside `task-forge/`. Every command from here forward runs from inside this folder unless we explicitly say otherwise.

## Step 2: Initialize Git

> [!IMPORTANT]
> **You should be in:** `task-forge/`

```bash
git init
```

## Step 3: Create the .gitignore

> [!IMPORTANT]
> **You should be in:** `task-forge/`

```bash
touch .gitignore
```

Open `.gitignore` and add:

```gitignore
# Dependencies
node_modules/

# Environment variables — NEVER commit these
.env
.env.local
.env.production

# Build output
dist/
build/

# OS files
.DS_Store
Thumbs.db

# Logs
*.log
```

> [!WARNING]
> **The `.env` line is critical.** Your `.env` file will contain your database connection string — a URL that includes your database password. If you commit it to GitHub, anyone can access your database. The `.gitignore` prevents this, but only if it exists **before** you create the `.env` file.

---

## Step 4: Create the Frontend with Vite

> [!IMPORTANT]
> **You should be in:** `task-forge/` (the project root)

```bash
npm create vite@latest client -- --template react-ts
```

```bash
cd client
```

> [!IMPORTANT]
> **You are now in:** `task-forge/client/`

```bash
npm install
```

### Verify It Works

```bash
npm run dev
```

Open `http://localhost:5173` in your browser. You should see the Vite + React starter page.

Press `Ctrl+C` to stop the dev server.

```bash
cd ..
```

> [!IMPORTANT]
> **You are now in:** `task-forge/` (back to the project root)

---

## Step 5: Create the Backend

> [!IMPORTANT]
> **You should be in:** `task-forge/` (the project root)

```bash
mkdir -p server/src
cd server
```

> [!IMPORTANT]
> **You are now in:** `task-forge/server/`

### Initialize the Node.js Project

```bash
npm init -y
```

### Install Dependencies

```bash
npm install express cors pg dotenv
npm install -D typescript @types/express @types/cors @types/pg @types/node ts-node nodemon
```

**What we installed and why:**

| Package | Type | What It Does | Why We Need It |
|---------|------|-------------|----------------|
| `express` | Production | Web server framework | Handles HTTP requests and routing |
| `cors` | Production | CORS middleware | Allows frontend to talk to backend |
| `pg` | Production | PostgreSQL client for Node.js | Sends SQL queries to the database |
| `dotenv` | Production | Loads `.env` file into `process.env` | Reads database URL from environment |
| `typescript` | Dev | TypeScript compiler | Compiles .ts to .js |
| `@types/express` | Dev | Type definitions for Express | TypeScript needs Express types |
| `@types/cors` | Dev | Type definitions for cors | TypeScript needs cors types |
| `@types/pg` | Dev | Type definitions for pg | TypeScript needs pg types |
| `@types/node` | Dev | Type definitions for Node.js | TypeScript needs Node types |
| `ts-node` | Dev | Run TypeScript directly | No compile step in dev |
| `nodemon` | Dev | Auto-restart on changes | Saves manual restarts |

**New packages vs Level 1:**
- `pg` — the PostgreSQL driver. This is how Node.js talks to PostgreSQL. It sends SQL queries and receives results.
- `dotenv` — reads a `.env` file and puts its contents into `process.env`. This is how we keep the database URL out of our code.
- `@types/pg` — TypeScript type definitions for the pg library.

### Configure TypeScript

Create `server/tsconfig.json`:

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "commonjs",
    "lib": ["ES2020"],
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

Same as Level 1 — nothing new here.

### Configure Scripts

Update `server/package.json` — add a `scripts` section:

```json
{
  "scripts": {
    "dev": "nodemon --exec ts-node src/index.ts",
    "build": "tsc",
    "start": "node dist/index.js"
  }
}
```

Same as Level 1.

```bash
cd ..
```

> [!IMPORTANT]
> **You are now in:** `task-forge/` (back to the project root)

---

## Step 6: Install PostgreSQL

This is the new part. PostgreSQL is a program that runs on your computer as a service (a background process). It listens on port 5432, waiting for SQL queries.

> [!NOTE]
> **Technical**: PostgreSQL (often called "Postgres") is an open-source relational database management system known for reliability, feature robustness, and SQL standards compliance. It runs as a daemon process and communicates via a TCP connection on port 5432.

> [!NOTE]
> **Plain English**: PostgreSQL is a program that runs in the background on your computer. Its job is to store data in tables and answer questions about that data. When your Express server needs to save or retrieve data, it connects to PostgreSQL on port 5432 and sends SQL commands.

### Install via Homebrew (macOS)

```bash
# Check if PostgreSQL is already installed
psql --version
```

If you see a version number (e.g., `psql (PostgreSQL) 16.x`), it's already installed — skip to "Start PostgreSQL."

If you see `command not found`, install it:

```bash
brew install postgresql@16
```

After installation, Homebrew will show instructions for adding PostgreSQL to your PATH. Follow them, or run:

```bash
echo 'export PATH="/opt/homebrew/opt/postgresql@16/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc
```

Verify it's installed:

```bash
psql --version
```

### Start PostgreSQL

```bash
# Start PostgreSQL service (runs in background)
brew services start postgresql@16
```

To check that it's running:

```bash
brew services list
```

You should see `postgresql@16` with status `started`.

### Create the Database

```bash
createdb taskforge
```

This creates a new database called `taskforge`. Think of it as creating a new empty spreadsheet workbook — it exists but has no sheets (tables) yet.

### Connect with psql

`psql` is the PostgreSQL command-line client. It lets you interact with the database directly:

```bash
psql taskforge
```

You should see a prompt like:

```
taskforge=#
```

You're now inside the PostgreSQL shell, connected to your `taskforge` database. Try these commands:

```sql
-- Show all databases on your machine
\l

-- Show all tables in this database (should be empty)
\dt

-- Quit psql
\q
```

> [!NOTE]
> **Technical**: `psql` is the interactive terminal for PostgreSQL. Backslash commands (`\l`, `\dt`, `\q`) are psql meta-commands that query system catalogs or control the client. Regular SQL commands end with a semicolon.

> [!NOTE]
> **Plain English**: `psql` is like a chat window with your database. You type SQL commands and it shows you results. The backslash commands (like `\dt`) are shortcuts specific to psql — they're not SQL, they're helper commands.

### PostgreSQL Spatial Map

```
YOUR COMPUTER
├── Port 5173 → Vite dev server (frontend)
├── Port 3001 → Express server (backend)
└── Port 5432 → PostgreSQL server (database)  ← NEW
    │
    ├── Database: taskforge                     ← We just created this
    │   ├── Table: projects (not yet)
    │   └── Table: tasks (not yet)
    │
    └── Database: postgres (default, ignore)
```

---

## Step 7: Create the Environment File

> [!IMPORTANT]
> **You should be in:** `task-forge/` (the project root)

```bash
cd server
```

> [!IMPORTANT]
> **You are now in:** `task-forge/server/`

Create `server/.env`:

```bash
touch .env
```

Open `.env` and add:

```env
DATABASE_URL=postgresql://localhost:5432/taskforge
PORT=3001
CORS_ORIGIN=http://localhost:5173
```

**What each variable means:**

| Variable | Value | Purpose |
|----------|-------|---------|
| `DATABASE_URL` | `postgresql://localhost:5432/taskforge` | Connection string — tells `pg` where the database is |
| `PORT` | `3001` | Which port Express listens on |
| `CORS_ORIGIN` | `http://localhost:5173` | Which origin is allowed to make requests |

### Connection String Anatomy

```
postgresql://localhost:5432/taskforge
│            │         │    │
│            │         │    └── Database name
│            │         └── Port (PostgreSQL default)
│            └── Host (your machine)
└── Protocol (PostgreSQL)
```

In production, the connection string will include a username and password:
```
postgresql://user:password@host:5432/dbname
```

> [!WARNING]
> **Never commit `.env` to Git.** Verify your `.gitignore` includes `.env` before making any commits. If your database connection string (with credentials) gets pushed to GitHub, anyone can access your database.

### Verify .env Is Ignored

```bash
cd ..
```

> [!IMPORTANT]
> **You are now in:** `task-forge/` (back to the project root)

```bash
git status
```

You should NOT see `server/.env` in the list. If you do, your `.gitignore` isn't working — check that `.env` is listed in `.gitignore` at the project root.

---

## Step 8: ESLint and Prettier Configuration

### Frontend

> [!IMPORTANT]
> **You should be in:** `task-forge/` (the project root)

```bash
cd client
```

> [!IMPORTANT]
> **You are now in:** `task-forge/client/`

```bash
npm install -D prettier eslint-config-prettier
```

Create `client/.prettierrc`:

```json
{
  "semi": true,
  "trailingComma": "es5",
  "singleQuote": true,
  "printWidth": 80,
  "tabWidth": 2
}
```

```bash
cd ..
```

> [!IMPORTANT]
> **You are now in:** `task-forge/` (back to the project root)

### Backend

```bash
cd server
```

> [!IMPORTANT]
> **You are now in:** `task-forge/server/`

```bash
npm install -D prettier
```

Create `server/.prettierrc`:

```json
{
  "semi": true,
  "trailingComma": "es5",
  "singleQuote": true,
  "printWidth": 80,
  "tabWidth": 2
}
```

```bash
cd ..
```

> [!IMPORTANT]
> **You are now in:** `task-forge/` (back to the project root)

---

## Step 9: Create the README

Create `task-forge/README.md`:

```markdown
# TaskForge — Project Task Manager

A full-stack project task manager with persistent data storage. Create projects, add tasks, mark them complete, edit, and delete — all data persists in PostgreSQL.

## Features

- Create and manage multiple projects
- Add tasks to projects with full CRUD operations
- Mark tasks complete/incomplete
- Edit project and task details
- Delete projects (cascades to tasks)
- Data persists in PostgreSQL database

## Tech Stack

- **Frontend**: React, TypeScript, Vite
- **Backend**: Node.js, Express, TypeScript
- **Database**: PostgreSQL
- **Deployment**: Vercel (frontend), Render (backend + database)

## Getting Started

### Prerequisites

- Node.js 20+
- npm 10+
- PostgreSQL 16+

### Installation

```bash
# Clone the repository
git clone https://github.com/yourusername/task-forge.git
cd task-forge

# Install frontend dependencies
cd client
npm install

# Install backend dependencies
cd ../server
npm install

# Create the database
createdb taskforge

# Run the schema
psql taskforge < src/db/schema.sql

# Create .env file
cp .env.example .env
# Edit .env with your database connection string
```

### Running Locally

```bash
# Terminal 1 — Start the backend (from task-forge/)
cd server
npm run dev

# Terminal 2 — Start the frontend (from task-forge/)
cd client
npm run dev
```

- Frontend: http://localhost:5173
- Backend: http://localhost:3001

## API Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | /api/projects | List all projects |
| POST | /api/projects | Create a project |
| GET | /api/projects/:id | Get project with tasks |
| PUT | /api/projects/:id | Update a project |
| DELETE | /api/projects/:id | Delete project and tasks |
| GET | /api/projects/:projectId/tasks | List tasks for project |
| POST | /api/projects/:projectId/tasks | Create task in project |
| PUT | /api/tasks/:id | Update a task |
| DELETE | /api/tasks/:id | Delete a task |
| GET | /api/health | Health check |

## What I Learned

This project demonstrates understanding of:
- PostgreSQL database design (tables, foreign keys, constraints)
- SQL queries (SELECT, INSERT, UPDATE, DELETE)
- Full CRUD REST API design
- Error handling middleware
- Environment variable management
- Three-tier architecture deployment
```

---

## Step 10: First Commit

> [!IMPORTANT]
> **You should be in:** `task-forge/` (the project root)

```bash
git add .
git commit -m "feat: initialize project structure with client, server, and database config"
```

**Spatial check**: At this point, you have:
- A configured frontend (React + TypeScript + Vite)
- A configured backend (Node + Express + TypeScript + pg + dotenv)
- PostgreSQL installed and running
- A `taskforge` database created (empty — no tables yet)
- A `.env` file with the database connection string (not committed)
- Git tracking your project

No code logic yet. No tables yet. That comes next.

---

## Project Structure So Far

```
task-forge/
├── .git/                     ← Git tracking
├── .gitignore                ← Ignores node_modules, .env, dist
├── README.md                 ← Project documentation
├── client/                   ← Frontend
│   ├── node_modules/
│   ├── public/
│   ├── src/                  ← React code goes here (empty for now)
│   ├── index.html
│   ├── package.json
│   ├── tsconfig.json
│   ├── vite.config.ts
│   └── .prettierrc
└── server/                   ← Backend
    ├── node_modules/
    ├── src/                  ← Express code goes here (empty for now)
    ├── .env                  ← DATABASE_URL, PORT, CORS_ORIGIN (not committed)
    ├── package.json
    ├── tsconfig.json
    └── .prettierrc

ALSO RUNNING:
PostgreSQL service on port 5432
└── Database: taskforge (empty — no tables)
```

---

### Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| `psql: command not found` | PostgreSQL not in PATH | Run the `echo 'export PATH...'` command from the install step |
| `createdb: could not connect to server` | PostgreSQL not running | `brew services start postgresql@16` |
| `createdb: database "taskforge" already exists` | Database exists | That's fine — skip to the next step |
| `.env` shows in `git status` | `.gitignore` missing or wrong | Add `.env` to `.gitignore` at the project root |
| `npm install` fails for `pg` | Node version issue | Verify `node --version` shows v20+ |

---

| | | |
|:---|:---:|---:|
| [← 01 — Orientation](../01-orientation/) | [Level 2 Overview](../) | [03 — Database Design →](../03-database/) |

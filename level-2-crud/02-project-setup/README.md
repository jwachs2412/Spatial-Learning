# Step 2 — Project Setup: Scaffolding with a Database

## Spatial Orientation

```
task-forge/          ← YOU ARE CREATING THIS
├── client/          ← Frontend (React + TypeScript)
├── server/          ← Backend (Express + TypeScript)
│   └── .env         ← Database credentials (NEVER committed)
├── .gitignore
└── README.md
```

This step adds a new dimension: a database running on your machine. By the end, you'll have PostgreSQL installed, a database created, and both frontend and backend scaffolded.

---

## 1. Create the Project Structure

```bash
mkdir task-forge
cd task-forge
git init
```

### Create .gitignore

```gitignore
node_modules/
.env
.env.local
.env.production
dist/
build/
.DS_Store
Thumbs.db
.vscode/settings.json
.idea/
*.log
npm-debug.log*
coverage/
```

---

## 2. Set Up the Frontend

```bash
npm create vite@latest client -- --template react-ts
cd client
npm install
cd ..
```

---

## 3. Set Up the Backend

```bash
mkdir -p server/src/{db,routes,middleware,types}
cd server
npm init -y
```

### Install Dependencies

```bash
# Runtime dependencies
npm install express cors pg dotenv

# TypeScript and dev tools
npm install typescript ts-node nodemon
```

| Package | What It Does |
|---------|-------------|
| `express` | Web framework for handling HTTP requests |
| `cors` | Enables cross-origin requests from frontend |
| `pg` | PostgreSQL client for Node.js — sends SQL queries to the database |
| `dotenv` | Reads `.env` files and loads them into `process.env` |
| `typescript` | Compiles TypeScript to JavaScript |
| `ts-node` | Runs TypeScript directly (for development) |
| `nodemon` | Auto-restarts server when files change |

### Install Type Definitions

> **Why not --save-dev for @types?**
> Deployment platforms like Render run `npm install --production`, skipping devDependencies. But `npm run build` (which runs `tsc`) needs type definitions to compile. If they're in devDependencies, the build fails with errors like: `Could not find a declaration file for module 'express'`.
>
> This is the **#1 deployment failure** in this curriculum. Always install @types as regular dependencies.

```bash
npm install @types/node @types/express @types/cors @types/pg
```

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
    "sourceMap": true,
    "types": ["node"]
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

> ⚠️ **Critical: `"types": ["node"]`**
> Without this, TypeScript doesn't know about `process.env`, `console.log`, or any Node.js globals. You'll see errors like `Cannot find name 'process'` and `Cannot find name 'console'` when building for deployment.

### Add Scripts

Update `server/package.json` scripts:

```json
{
  "scripts": {
    "dev": "nodemon --exec ts-node src/index.ts",
    "build": "tsc",
    "start": "node dist/index.js"
  }
}
```

### Verify Dependencies

Your `server/package.json` `"dependencies"` should include:

```json
{
  "dependencies": {
    "@types/cors": "^2.x.x",
    "@types/express": "^5.x.x",
    "@types/node": "^22.x.x",
    "@types/pg": "^8.x.x",
    "cors": "^2.x.x",
    "dotenv": "^16.x.x",
    "express": "^5.x.x",
    "nodemon": "^3.x.x",
    "pg": "^8.x.x",
    "ts-node": "^10.x.x",
    "typescript": "^5.x.x"
  }
}
```

> ⚠️ **Common Mistake: @types in devDependencies**
> If you accidentally used `--save-dev`, fix it:
> ```bash
> npm uninstall @types/node @types/express @types/cors @types/pg
> npm install @types/node @types/express @types/cors @types/pg
> ```

---

## 4. Install PostgreSQL

> **Key Concept: PostgreSQL**
> PostgreSQL (often called "Postgres") is an open-source relational database. It runs as a separate process on your machine — it's not part of your application. Your backend connects to it over a local network socket, just like it would connect over the internet in production.

### macOS (using Homebrew)

```bash
# Check if PostgreSQL is already installed
psql --version

# If not installed:
brew install postgresql@16
brew services start postgresql@16
```

If `brew services start` doesn't work, try:
```bash
brew services restart postgresql@16
```

### Verify PostgreSQL is Running

```bash
psql postgres -c "SELECT NOW();"
```

You should see the current timestamp. If you get "connection refused," PostgreSQL isn't running — restart it with `brew services restart postgresql@16`.

### Create the TaskForge Database

```bash
createdb taskforge
```

Verify it exists:
```bash
psql taskforge -c "SELECT 'TaskForge database connected!' AS status;"
```

Expected output: `TaskForge database connected!`

---

## 5. Configure Environment Variables

> **Key Concept: Connection String**
> A connection string is a URL that contains everything needed to connect to a database: the protocol, username, password, host, port, and database name. It follows this format:
>
> `postgresql://username:password@host:port/database`
>
> For local development, PostgreSQL typically uses your system username with no password.

Create `server/.env`:

```bash
PORT=3001
DATABASE_URL=postgresql://localhost:5432/taskforge
CORS_ORIGIN=http://localhost:5173
```

| Variable | What It Is | Why It's Here |
|----------|-----------|---------------|
| `PORT` | Port the backend runs on | Deployment platforms set this automatically |
| `DATABASE_URL` | PostgreSQL connection string | Your backend uses this to find the database |
| `CORS_ORIGIN` | Allowed frontend origin | Prevents unauthorized cross-origin requests |

> ⚠️ **Important:** The `.env` file is in your `.gitignore`. It should NEVER be committed to Git. It contains credentials that change between environments (development vs production).

---

## 6. Create the README

<details>
<summary>See a template</summary>

```markdown
# TaskForge — Project Task Manager

A full-stack project management tool with persistent PostgreSQL storage. Create projects, add tasks, mark complete, edit, and delete.

## Tech Stack

- **Frontend**: React, TypeScript, Vite
- **Backend**: Node.js, Express, TypeScript
- **Database**: PostgreSQL
- **Deployment**: Vercel (frontend), Render/Railway (backend + database)

## Getting Started

### Prerequisites

- Node.js 20+
- PostgreSQL 16+

### Installation

\```bash
git clone https://github.com/yourusername/task-forge.git
cd task-forge

cd client && npm install
cd ../server && npm install
\```

### Database Setup

\```bash
createdb taskforge
psql taskforge < server/src/db/schema.sql
\```

### Environment Variables

Create `server/.env`:

\```
PORT=3001
DATABASE_URL=postgresql://localhost:5432/taskforge
CORS_ORIGIN=http://localhost:5173
\```

### Running Locally

\```bash
# Terminal 1 — Backend
cd server && npm run dev

# Terminal 2 — Frontend
cd client && npm run dev
\```

## API Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | /api/projects | List all projects |
| POST | /api/projects | Create project |
| GET | /api/projects/:id | Get project with tasks |
| PUT | /api/projects/:id | Update project |
| DELETE | /api/projects/:id | Delete project + tasks |
| GET | /api/projects/:projectId/tasks | List tasks |
| POST | /api/projects/:projectId/tasks | Create task |
| PUT | /api/tasks/:id | Update task |
| DELETE | /api/tasks/:id | Delete task |
| GET | /api/health | Health check |
```

</details>

---

## 7. First Commit

```bash
cd ..
git add .
git commit -m "feat: scaffold task-forge with PostgreSQL setup"
```

### ✅ Checkpoint

- [ ] PostgreSQL is running (`psql postgres -c "SELECT NOW();"` works)
- [ ] `taskforge` database exists
- [ ] `server/.env` has DATABASE_URL, PORT, and CORS_ORIGIN
- [ ] `server/package.json` has @types packages in `dependencies` (NOT `devDependencies`)
- [ ] `server/tsconfig.json` includes `"types": ["node"]`
- [ ] `.gitignore` includes `.env` and `node_modules/`

---

> **Session Break** — Project scaffolded with database ready.
> When you return, you'll design the schema and write SQL in [Step 3 — Database](../03-database/).

---

| | |
|:---|---:|
| [← Step 1: Orientation](../01-orientation/) | [Step 3 — Database →](../03-database/) |

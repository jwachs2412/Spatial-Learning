`Level 3` **Step 2 of 8** — Project Setup

# 02 — Project Setup: Scaffolding with Authentication

## Spatial Orientation

Same two-project structure as Levels 1 and 2, with new dependencies for authentication:

```
vault-note/              ← The project root
├── client/              ← Frontend (React + TypeScript + Vite + React Router)
├── server/              ← Backend (Node + Express + TypeScript + pg + bcrypt + jsonwebtoken)
│   └── .env             ← DATABASE_URL + JWT_SECRET (NEVER committed)
└── PostgreSQL           ← Running as a service on your machine (port 5432)
```

**What's new in the stack?**
- `bcrypt` — password hashing
- `jsonwebtoken` — JWT creation and verification
- `react-router-dom` — client-side routing (multiple pages without full reload)

---

## Step 1: Create the Project Folder

Open your terminal. Navigate to your projects folder.

```bash
mkdir vault-note
cd vault-note
```

## Step 2: Initialize Git

> [!IMPORTANT]
> **You should be in:** `vault-note/`

```bash
git init
```

## Step 3: Create the .gitignore

> [!IMPORTANT]
> **You should be in:** `vault-note/`

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
> **Extra critical this level.** Your `.env` file will now contain `JWT_SECRET` — the key that signs all authentication tokens. If this leaks, anyone can forge tokens and impersonate any user.

---

## Step 4: Create the Frontend with Vite

> [!IMPORTANT]
> **You should be in:** `vault-note/`

```bash
npm create vite@latest client -- --template react-ts
```

```bash
cd client
```

> [!IMPORTANT]
> **You are now in:** `vault-note/client/`

```bash
npm install
npm install react-router-dom
```

**New dependency:**

| Package | What It Does | Why We Need It |
|---------|-------------|----------------|
| `react-router-dom` | Client-side routing for React | Multiple pages (login, register, notes) without full page reloads |

> [!NOTE]
> **Technical**: React Router provides declarative routing for React applications. It intercepts browser navigation events and renders different components based on the URL, without requesting a new page from the server.

> [!NOTE]
> **Plain English**: Without React Router, your app is one page. With it, you get `/login`, `/register`, `/notes` — each showing different content. When you click a link, React Router swaps the component without refreshing the page. The URL changes, the browser back button works, but no full page reload happens.

### Verify It Works

```bash
npm run dev
```

Open `http://localhost:5173`. See the Vite starter page. Press `Ctrl+C` to stop.

```bash
cd ..
```

> [!IMPORTANT]
> **You are now in:** `vault-note/`

---

## Step 5: Create the Backend

> [!IMPORTANT]
> **You should be in:** `vault-note/`

```bash
mkdir -p server/src
cd server
```

> [!IMPORTANT]
> **You are now in:** `vault-note/server/`

```bash
npm init -y
```

### Install Dependencies

```bash
npm install express cors pg dotenv bcrypt jsonwebtoken
npm install -D typescript @types/express @types/cors @types/pg @types/node @types/bcrypt @types/jsonwebtoken ts-node nodemon
```

**New packages vs Level 2:**

| Package | Type | What It Does | Why We Need It |
|---------|------|-------------|----------------|
| `bcrypt` | Production | Password hashing | Hashes passwords before storing, compares on login |
| `jsonwebtoken` | Production | JWT creation + verification | Creates tokens on login, verifies on every protected request |
| `@types/bcrypt` | Dev | Type definitions for bcrypt | TypeScript needs bcrypt types |
| `@types/jsonwebtoken` | Dev | Type definitions for jsonwebtoken | TypeScript needs JWT types |

Everything else (`express`, `cors`, `pg`, `dotenv`, `typescript`, etc.) is the same as Level 2.

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

Same as Levels 1 and 2.

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

```bash
cd ..
```

> [!IMPORTANT]
> **You are now in:** `vault-note/`

---

## Step 6: Create the Database

> [!IMPORTANT]
> **You should be in:** `vault-note/`

Verify PostgreSQL is running:

```bash
brew services list
```

If `postgresql@16` isn't started:

```bash
brew services start postgresql@16
```

Create the database:

```bash
createdb vaultnote
```

Verify it exists:

```bash
psql vaultnote -c "SELECT 1;"
```

You should see a result, confirming the database is accessible.

---

## Step 7: Create the Environment File

```bash
cd server
```

> [!IMPORTANT]
> **You are now in:** `vault-note/server/`

Create `.env`:

```env
DATABASE_URL=postgresql://localhost:5432/vaultnote
PORT=3001
CORS_ORIGIN=http://localhost:5173
JWT_SECRET=your-super-secret-key-change-this-in-production
JWT_EXPIRES_IN=7d
```

**New environment variables:**

| Variable | Value | Purpose |
|----------|-------|---------|
| `JWT_SECRET` | A long, random string | Signs and verifies JWT tokens. If someone knows this, they can forge any token. |
| `JWT_EXPIRES_IN` | `7d` | Tokens expire after 7 days. Users must log in again after that. |

> [!WARNING]
> **`JWT_SECRET` must be kept secret.** In development, the value above is fine. In production, use a long random string (32+ characters). You can generate one with:
> ```bash
> node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"
> ```

```bash
cd ..
```

> [!IMPORTANT]
> **You are now in:** `vault-note/`

### Verify .env Is Ignored

```bash
git status
```

Confirm `server/.env` does NOT appear in the output.

---

## Step 8: Prettier Configuration

### Frontend

```bash
cd client
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

### Backend

```bash
cd server
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
> **You are now in:** `vault-note/`

---

## Step 9: Create the README

Create `vault-note/README.md`:

```markdown
# VaultNote — Secure Personal Notes

A full-stack notes application with user authentication. Users register, log in, and manage private notes that only they can access. Built with JWT authentication, bcrypt password hashing, and role-based access control.

## Features

- User registration and login with secure password hashing
- JWT token-based authentication
- Private notes — users can only access their own
- Full CRUD operations on notes
- Admin statistics dashboard
- Client-side routing with React Router

## Tech Stack

- **Frontend**: React, TypeScript, Vite, React Router
- **Backend**: Node.js, Express, TypeScript
- **Database**: PostgreSQL
- **Auth**: bcrypt, JSON Web Tokens
- **Deployment**: Vercel (frontend), Render (backend + database)

## Getting Started

### Prerequisites

- Node.js 20+
- npm 10+
- PostgreSQL 16+

### Installation

```bash
git clone https://github.com/yourusername/vault-note.git
cd vault-note

cd client && npm install && cd ..
cd server && npm install && cd ..

createdb vaultnote
psql vaultnote < server/src/db/schema.sql

# Create server/.env with DATABASE_URL, JWT_SECRET, etc.
```

### Running Locally

```bash
# Terminal 1 — Backend
cd server && npm run dev

# Terminal 2 — Frontend
cd client && npm run dev
```

## API Endpoints

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| POST | /api/auth/register | No | Create account |
| POST | /api/auth/login | No | Login, get token |
| GET | /api/auth/me | Yes | Current user info |
| GET | /api/notes | Yes | List user's notes |
| POST | /api/notes | Yes | Create a note |
| GET | /api/notes/:id | Yes | Get a note |
| PUT | /api/notes/:id | Yes | Update a note |
| DELETE | /api/notes/:id | Yes | Delete a note |
| GET | /api/admin/stats | Admin | Usage statistics |
| GET | /api/health | No | Health check |
```

---

## Step 10: First Commit

> [!IMPORTANT]
> **You should be in:** `vault-note/`

```bash
git add .
git commit -m "feat: initialize project structure with auth dependencies"
```

---

## Project Structure So Far

```
vault-note/
├── .git/
├── .gitignore
├── README.md
├── client/
│   ├── node_modules/
│   ├── src/                  ← React code (empty for now)
│   ├── index.html
│   ├── package.json          ← Now includes react-router-dom
│   ├── tsconfig.json
│   ├── vite.config.ts
│   └── .prettierrc
└── server/
    ├── node_modules/
    ├── src/                  ← Express code (empty for now)
    ├── .env                  ← DATABASE_URL + JWT_SECRET (not committed)
    ├── package.json          ← Now includes bcrypt + jsonwebtoken
    ├── tsconfig.json
    └── .prettierrc

ALSO RUNNING:
PostgreSQL service on port 5432
└── Database: vaultnote (empty — no tables)
```

---

| | | |
|:---|:---:|---:|
| [← 01 — Orientation](../01-orientation/) | [Level 3 Overview](../) | [03 — Database Design →](../03-database/) |

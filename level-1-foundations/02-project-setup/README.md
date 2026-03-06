`Level 1` **Step 2 of 7** — Project Setup

# 02 — Project Setup: Creating the Skeleton

## Spatial Orientation

We're about to create two separate projects inside one folder:

```
dev-pulse/           ← The project root
├── client/          ← Frontend project (React + TypeScript + Vite)
└── server/          ← Backend project (Node + Express + TypeScript)
```

These are **independent projects**, each with their own `package.json`, their own dependencies, and their own `node_modules`. They could live in separate repositories. We keep them together for convenience.

**Where are we in the stack?** Nowhere yet — we're building the scaffolding that the code will live in.

---

## Step 1: Create the Project Folder

Open your terminal. You should be in the folder where you keep your projects (for example, `~/Desktop/Sites` or `~/projects`). If you're not sure where you are, run `pwd` — it shows your current directory.

```bash
# You should be in your projects folder (e.g., ~/Desktop/Sites)
# If you're not there, navigate to it first:
# cd ~/Desktop/Sites

mkdir dev-pulse
cd dev-pulse
```

You are now inside `dev-pulse/`. Every command from here forward is run from inside this folder unless we explicitly say otherwise.

## Step 2: Initialize Git

> [!IMPORTANT]
> **You should be in:** `dev-pulse/`

First, check if this folder is already a Git repository:

```bash
git status
```

If you see `fatal: not a git repository`, that's expected — we haven't initialized it yet. Run:

```bash
git init
```

If you see something like `On branch main, nothing to commit`, Git is already initialized — skip to Step 3.

This creates a hidden `.git` folder that tracks all changes. Nothing is tracked yet — we need to tell Git what to watch.

## Step 3: Create the .gitignore

> [!IMPORTANT]
> **You should be in:** `dev-pulse/`

```bash
touch .gitignore
```

Open `.gitignore` and add:

```gitignore
# Dependencies
node_modules/

# Environment variables
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

**Why now?** We create `.gitignore` before adding any code so that Git never tracks `node_modules/` or `.env` files. If Git tracks them even once, they're in history forever.

---

## Step 4: Create the Frontend with Vite

### What Is Vite?

> [!NOTE]
> **Technical**: Vite is a build tool that provides a fast development server with Hot Module Replacement (HMR) and an optimized production build using Rollup.

> [!NOTE]
> **Plain English**: Vite is the engine that runs your React app during development. It instantly shows your changes in the browser when you save a file, and it packages your code into optimized files for production.

### Why Vite (Not Create React App, Not Next.js)

- **Vite over Create React App (CRA)**: CRA is effectively deprecated — the React team no longer recommends it. CRA uses Webpack, which is slower. Vite starts in milliseconds, CRA takes seconds.
- **Vite over Next.js**: Next.js is a full framework with its own backend, routing, and opinions. We want a pure React frontend and a separate Express backend so you understand the boundary between them. Next.js blurs that boundary (intentionally — it's great, just not right for learning the fundamentals).

### Create the Frontend

First, confirm that Node.js and npm are ready to go:

```bash
node --version    # Should show v20.x.x
npm --version     # Should show 10.x.x
```

If either command shows `command not found`, go back to the [Setup Guide](../../00-setup-guide/) and install Node.js first.

> [!IMPORTANT]
> **You should be in:** `dev-pulse/` (the project root)

```bash
npm create vite@latest client -- --template react-ts
```

**What this command does:**
- `npm create vite@latest` → Run the Vite project creation tool
- `client` → Name the folder "client"
- `--template react-ts` → Use the React + TypeScript template

Now move into the newly created `client/` folder and install its dependencies:

> [!IMPORTANT]
> **You should be in:** `dev-pulse/`

```bash
cd client
```

> [!IMPORTANT]
> **You are now in:** `dev-pulse/client/`

Check if dependencies are already installed (i.e., if a `node_modules` folder exists):

```bash
ls node_modules
```

If you see a list of folders, dependencies are already installed — skip to "Verify It Works." If you see `No such file or directory`, install them:

```bash
npm install
```

`npm install` reads `package.json` and downloads all listed dependencies into `node_modules/`.

### Verify It Works

> [!IMPORTANT]
> **You should be in:** `dev-pulse/client/`

```bash
npm run dev
```

Open `http://localhost:5173` in your browser. You should see the Vite + React starter page.

Press `Ctrl+C` to stop the dev server.

Now go back to the project root:

```bash
cd ..
```

> [!IMPORTANT]
> **You are now in:** `dev-pulse/` (back to the project root)

---

> [!TIP]
> **Session Break** — You've scaffolded the frontend with Vite, React, and TypeScript. Save your work and take a break.
> When you return, you'll set up the backend with Express and configure the dev tooling.

---

## Step 5: Create the Backend

> [!IMPORTANT]
> **You should be in:** `dev-pulse/` (the project root)

```bash
mkdir -p server/src
cd server
```

> [!IMPORTANT]
> **You are now in:** `dev-pulse/server/`

### Initialize the Node.js Project

> [!IMPORTANT]
> **You should be in:** `dev-pulse/server/`

```bash
npm init -y
```

This creates `package.json` — the manifest file that describes your project, its dependencies, and its scripts.

### Install Dependencies

> [!IMPORTANT]
> **You should be in:** `dev-pulse/server/`

Check if dependencies are already installed:

```bash
ls node_modules
```

If you see `No such file or directory` or the folder is empty, install them. If you already see folders listed, these packages may already be installed — but running the install command again is harmless (npm will skip what's already there):

```bash
npm install express cors
npm install -D typescript @types/express @types/cors @types/node ts-node nodemon
```

**What we just installed and why:**

| Package | Type | What It Does | Why We Need It |
|---------|------|-------------|----------------|
| `express` | Production | Web server framework | Handles HTTP requests and routing |
| `cors` | Production | CORS middleware | Allows our frontend to talk to our backend |
| `typescript` | Dev | TypeScript compiler | Compiles .ts files to .js files |
| `@types/express` | Dev | Type definitions for Express | TypeScript needs to know Express's types |
| `@types/cors` | Dev | Type definitions for cors | TypeScript needs to know cors's types |
| `@types/node` | Dev | Type definitions for Node.js | TypeScript needs to know Node's types |
| `ts-node` | Dev | Run TypeScript directly | No need to compile before running in dev |
| `nodemon` | Dev | Auto-restart server on changes | Saves you from manually restarting |

**Production vs Dev dependencies:**
- `dependencies` (no `-D`): Needed when the app runs in production
- `devDependencies` (`-D`): Only needed during development (won't be in production)

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

**What the important settings mean:**

| Setting | What It Does | Plain English |
|---------|-------------|---------------|
| `target: "ES2020"` | Output modern JavaScript | Our server runs Node 20, which supports modern JS |
| `module: "commonjs"` | Use Node's module system | Node uses `require()`, not `import` (by default) |
| `outDir: "./dist"` | Put compiled files in dist/ | Keeps compiled JS separate from source TS |
| `rootDir: "./src"` | Source files live in src/ | Tells TypeScript where to look |
| `strict: true` | Enable all strict checks | Catches more bugs. This is the whole point of TS. |
| `esModuleInterop: true` | Better import compatibility | Lets you write `import express from 'express'` |
| `sourceMap: true` | Generate source maps | Maps compiled JS back to TS for debugging |

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

**What each script does:**

| Script | Command | What It Does |
|--------|---------|-------------|
| `npm run dev` | `nodemon --exec ts-node src/index.ts` | Runs the server, auto-restarts on file changes |
| `npm run build` | `tsc` | Compiles TypeScript to JavaScript (for production) |
| `npm start` | `node dist/index.js` | Runs the compiled JavaScript (in production) |

Now go back to the project root:

```bash
cd ..
```

> [!IMPORTANT]
> **You are now in:** `dev-pulse/` (back to the project root)

---

## Step 6: ESLint and Prettier Configuration

### Frontend (Vite already includes basic ESLint)

The Vite React template includes ESLint. Let's add Prettier.

> [!IMPORTANT]
> **You should be in:** `dev-pulse/` (the project root)

```bash
cd client
```

> [!IMPORTANT]
> **You are now in:** `dev-pulse/client/`

Check if Prettier is already installed in this project:

```bash
npx prettier --version
```

If you see a version number, it's already installed — skip to creating the `.prettierrc` file below. If you see a prompt to install or `command not found`, install it:

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

**What each rule does:**

| Rule | Value | What It Means |
|------|-------|---------------|
| `semi` | true | Add semicolons at the end of statements |
| `trailingComma` | "es5" | Add trailing commas where ES5 allows (arrays, objects) |
| `singleQuote` | true | Use 'single quotes' not "double quotes" for strings |
| `printWidth` | 80 | Wrap lines longer than 80 characters |
| `tabWidth` | 2 | Use 2 spaces for indentation |

Now go back to the project root:

```bash
cd ..
```

> [!IMPORTANT]
> **You are now in:** `dev-pulse/` (back to the project root)

### Backend

> [!IMPORTANT]
> **You should be in:** `dev-pulse/` (the project root)

```bash
cd server
```

> [!IMPORTANT]
> **You are now in:** `dev-pulse/server/`

Check if Prettier is already installed in this project:

```bash
npx prettier --version
```

If you see a version number, skip to creating the `.prettierrc` file below. Otherwise, install it:

```bash
npm install -D prettier
```

Create `server/.prettierrc` (same config):

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
> **You are now in:** `dev-pulse/` (back to the project root)

---

## Step 7: Create the README

Create `dev-pulse/README.md`:

```markdown
# DevPulse — Developer Mood Tracker

A simple web application for developers to track their daily mood, energy level, and reflections.

## Features

- Log daily mood entries with mood, energy level (1-5), and notes
- View history of all mood entries
- Simple mood visualization

## Tech Stack

- **Frontend**: React, TypeScript, Vite
- **Backend**: Node.js, Express, TypeScript
- **Data**: In-memory storage

## Getting Started

### Prerequisites

- Node.js 20+
- npm 10+

### Installation

```bash
# Clone the repository
git clone https://github.com/yourusername/dev-pulse.git
cd dev-pulse

# Install frontend dependencies
cd client
npm install

# Install backend dependencies
cd ../server
npm install
```

### Running Locally

```bash
# Terminal 1 — Start the backend (from the dev-pulse/ root)
cd server
npm run dev

# Terminal 2 — Start the frontend (from the dev-pulse/ root)
cd client
npm run dev
```

- Frontend: http://localhost:5173
- Backend: http://localhost:3001

## API Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | /api/entries | Get all mood entries |
| POST | /api/entries | Create a new mood entry |

## What I Learned

This project demonstrates understanding of:
- Client-server architecture
- React component design with TypeScript
- REST API design with Express
- Frontend-backend communication
- Deployment fundamentals
```

---

## Step 8: First Commit

> [!IMPORTANT]
> **You should be in:** `dev-pulse/` (the project root)

```bash
git add .
git commit -m "feat: initialize project structure with client and server"
```

**Spatial check**: At this point, you have:
- An empty but configured frontend (React + TypeScript + Vite)
- An empty but configured backend (Node + Express + TypeScript)
- Git tracking your project
- A README explaining the project
- A .gitignore protecting secrets and junk files

No code logic yet. That comes next.

---

## Project Structure So Far

```
dev-pulse/
├── .git/                    ← Git's internal tracking (don't touch)
├── .gitignore               ← Files Git ignores
├── README.md                ← Project documentation
├── client/                  ← Frontend
│   ├── node_modules/        ← Downloaded packages (gitignored)
│   ├── public/              ← Static assets
│   ├── src/                 ← Your React code goes here
│   │   ├── App.tsx          ← Main component (will rewrite)
│   │   ├── App.css          ← Styles (will rewrite)
│   │   └── main.tsx         ← Entry point
│   ├── index.html           ← The single HTML page
│   ├── package.json         ← Frontend dependencies
│   ├── tsconfig.json        ← TypeScript config
│   ├── vite.config.ts       ← Vite config
│   └── .prettierrc          ← Code formatting rules
└── server/                  ← Backend
    ├── node_modules/        ← Downloaded packages (gitignored)
    ├── src/                 ← Your Express code goes here (empty)
    ├── package.json         ← Backend dependencies
    ├── tsconfig.json        ← TypeScript config
    └── .prettierrc          ← Code formatting rules
```

---

| | | |
|:---|:---:|---:|
| [← 01 — Spatial Orientation](../01-orientation/) | [Level 1 Overview](../) | [03 — Building the Backend →](../03-backend/) |

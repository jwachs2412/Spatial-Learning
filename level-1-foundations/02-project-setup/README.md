# Step 2 — Project Setup: Creating the Skeleton

## Spatial Orientation

```
dev-pulse/          ← YOU ARE CREATING THIS
├── client/         ← Frontend (React + TypeScript)
├── server/         ← Backend (Node + Express + TypeScript)
├── .gitignore
└── README.md
```

In this step, you create the project structure, install dependencies, and configure TypeScript. No application code yet — just the skeleton.

---

## 1. Create the Project Root

> **Key Concept: Project Root**
> The project root is the top-level folder that contains everything. Both your frontend (`client/`) and backend (`server/`) live inside it. This monorepo structure keeps related code together.

Open your terminal and navigate to wherever you keep projects:

```bash
mkdir dev-pulse
cd dev-pulse
```

### Initialize Git

```bash
git init
```

### Create .gitignore

> **Why this matters:** Without a `.gitignore`, you'll accidentally commit `node_modules/` (thousands of files) and `.env` files (secrets). Both are serious mistakes.

### 🏗️ Your Turn

Before looking at the solution, think about what files and folders should be ignored in a JavaScript/TypeScript project. List at least 4 categories.

<details>
<summary>See the solution</summary>

Create `.gitignore` in the project root:

```gitignore
# Dependencies — thousands of downloaded files, recreated by npm install
node_modules/

# Environment variables — contain secrets (passwords, API keys)
.env
.env.local
.env.production

# Build output — generated from source code, not source code itself
dist/
build/

# OS files — system junk that isn't part of your project
.DS_Store
Thumbs.db

# IDE settings — personal editor preferences
.vscode/settings.json
.idea/

# Logs
*.log
npm-debug.log*

# Test coverage
coverage/
```

</details>

---

## 2. Set Up the Frontend (React + TypeScript)

> **Key Concept: Vite**
> Vite (French for "fast") is a build tool that creates React projects with TypeScript support built in. It replaces the older Create React App (CRA). Vite starts in milliseconds because it serves files directly to the browser during development instead of bundling everything first.

```bash
npm create vite@latest client -- --template react-ts
```

**What this command does:**

| Part | Meaning |
|------|---------|
| `npm create vite@latest` | Run Vite's project scaffolding tool |
| `client` | Name the folder `client` |
| `--` | Separator — everything after this is passed to Vite |
| `--template react-ts` | Use the React + TypeScript template |

```bash
cd client
npm install
cd ..
```

### ✅ Checkpoint

```bash
cd client && npm run dev
```

You should see output like `Local: http://localhost:5173/`. Open that URL — you should see the Vite + React starter page. Press `Ctrl+C` to stop the server, then `cd ..` to return to the project root.

---

## 3. Set Up the Backend (Node + Express + TypeScript)

```bash
mkdir -p server/src
cd server
npm init -y
```

### Install Dependencies

> **Key Concept: Dependencies vs DevDependencies**
> **Dependencies** are packages your app needs to run (Express, cors). **DevDependencies** are packages you only need during development (TypeScript compiler, nodemon). The difference matters during deployment.

```bash
# Runtime dependencies — needed to run the server
npm install express cors

# TypeScript and development tools
npm install typescript ts-node nodemon
```

### Install Type Definitions

> **Why not --save-dev for @types?**
> When you deploy to platforms like Render, Railway, or Fly.io, they run `npm install --production` by default, which skips `devDependencies`. But your build step (`npm run build` = `tsc`) still needs type definitions to compile TypeScript. If @types packages are in `devDependencies`, the build fails with: `Could not find a declaration file for module 'express'`.
>
> This is the **#1 most common deployment failure** in this curriculum. Always install @types as regular dependencies.

```bash
npm install @types/node @types/express @types/cors
```

**NOT** `npm install --save-dev @types/...` — that puts them in devDependencies, which breaks production builds.

### Configure TypeScript

> **Key Concept: tsconfig.json**
> This file tells the TypeScript compiler how to compile your code. Every option has a purpose. Don't just copy it — understand what each setting does.

### 🏗️ Your Turn

Create `server/tsconfig.json`. You know it needs to:
- Compile modern JavaScript (ES2020)
- Output to a `dist/` folder
- Read source from `src/`
- Be strict about types
- Know about Node.js built-in types (like `process`, `console`, `setTimeout`)

Try writing it yourself, then check the solution.

<details>
<summary>See the solution</summary>

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

</details>

**Line-by-line breakdown:**

| Setting | What It Does | Why It Matters |
|---------|-------------|----------------|
| `"target": "ES2020"` | Compile to ES2020 JavaScript | Modern enough for Node.js 20+, no unnecessary transpilation |
| `"module": "commonjs"` | Use `require()`/`module.exports` style | Node.js standard module system |
| `"lib": ["ES2020"]` | Include ES2020 standard library types | Gives you `Promise`, `Map`, `Set`, etc. |
| `"outDir": "./dist"` | Put compiled JS in `dist/` folder | Keeps compiled output separate from source |
| `"rootDir": "./src"` | Source code lives in `src/` | Tells compiler where to find your TypeScript |
| `"strict": true` | Enable all strict type checks | Catches bugs at compile time, not runtime |
| `"esModuleInterop": true` | Allow `import x from 'y'` syntax | Makes imports work consistently |
| `"skipLibCheck": true` | Don't type-check `node_modules` | Faster compilation |
| `"types": ["node"]` | Include Node.js type definitions | **Critical:** Without this, `process`, `console`, and `setTimeout` cause errors |
| `"sourceMap": true` | Generate `.map` files | Maps compiled JS back to TypeScript for debugging |

> ⚠️ **Common Mistake: Missing `"types": ["node"]`**
> Without this setting, TypeScript doesn't know about Node.js globals like `process.env`, `console.log`, or `setTimeout`. You'll see errors like:
> - `Cannot find name 'process'`
> - `Cannot find name 'console'`
>
> Always include `"types": ["node"]` in backend TypeScript projects.

### Add Scripts to package.json

Open `server/package.json` and replace the `"scripts"` section:

```json
{
  "scripts": {
    "dev": "nodemon --exec ts-node src/index.ts",
    "build": "tsc",
    "start": "node dist/index.js"
  }
}
```

| Script | What It Does | When You Use It |
|--------|-------------|-----------------|
| `dev` | Runs your server with auto-restart on file changes | During development |
| `build` | Compiles TypeScript to JavaScript | Before deployment |
| `start` | Runs the compiled JavaScript | In production |

### Verify your package.json dependencies

Your `server/package.json` should have these packages under `"dependencies"` (NOT `"devDependencies"`):

```json
{
  "dependencies": {
    "@types/cors": "^2.x.x",
    "@types/express": "^5.x.x",
    "@types/node": "^22.x.x",
    "cors": "^2.x.x",
    "express": "^5.x.x",
    "nodemon": "^3.x.x",
    "ts-node": "^10.x.x",
    "typescript": "^5.x.x"
  }
}
```

> ⚠️ **Common Mistake: @types in devDependencies**
> If you accidentally used `--save-dev` for @types packages, move them:
> ```bash
> npm uninstall @types/node @types/express @types/cors
> npm install @types/node @types/express @types/cors
> ```
> This removes them from devDependencies and adds them to dependencies.

Return to the project root:

```bash
cd ..
```

---

## 4. Create the README

### 🏗️ Your Turn

Write a README.md for DevPulse in the project root. Include:
- Project name and description
- Tech stack
- How to install and run locally
- API endpoints

<details>
<summary>See a template</summary>

Create `README.md` in the project root:

```markdown
# DevPulse — Developer Mood Tracker

Track your daily mood, energy, and notes as a developer. Built with React, TypeScript, and Express.

## Tech Stack

- **Frontend**: React, TypeScript, Vite
- **Backend**: Node.js, Express, TypeScript
- **Deployment**: Vercel (frontend), Render (backend)

## Getting Started

### Prerequisites

- Node.js 20+
- npm 10+

### Installation

```bash
# Clone the repo
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
# Terminal 1 — Backend (from dev-pulse/server)
npm run dev

# Terminal 2 — Frontend (from dev-pulse/client)
npm run dev
```

## API Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | /api/entries | Get all mood entries |
| POST | /api/entries | Create a new entry |
| GET | /api/health | Health check |
```

</details>

---

## 5. ESLint and Prettier

These are already included in the Vite template for the frontend. For the backend, we'll keep it simple with just TypeScript's built-in checking for now.

Prettier configuration — create `.prettierrc` in the project root:

```json
{
  "semi": true,
  "singleQuote": true,
  "tabWidth": 2,
  "trailingComma": "all"
}
```

| Setting | What It Does |
|---------|-------------|
| `"semi": true` | Always use semicolons |
| `"singleQuote": true` | Use `'` instead of `"` in JavaScript/TypeScript (JSON always uses `"`) |
| `"tabWidth": 2` | 2-space indentation (industry standard for JS/TS) |
| `"trailingComma": "all"` | Add commas after the last item in lists (makes diffs cleaner) |

---

## 6. First Commit

```bash
git add .
git commit -m "feat: scaffold dev-pulse project with client and server"
```

### ✅ Checkpoint

Your project should look like this:

```
dev-pulse/
├── .gitignore
├── .prettierrc
├── README.md
├── client/          ← Vite React app (runs with npm run dev)
│   ├── src/
│   ├── package.json
│   └── ...
└── server/          ← Express app (not running yet — no src/index.ts)
    ├── src/         ← Empty, ready for Step 3
    ├── package.json ← @types packages in dependencies
    └── tsconfig.json ← includes "types": ["node"]
```

Verify:
1. `client/` has a working React dev server (`cd client && npm run dev`)
2. `server/package.json` has `@types/node`, `@types/express`, `@types/cors` under `"dependencies"` (NOT `"devDependencies"`)
3. `server/tsconfig.json` includes `"types": ["node"]`
4. `.gitignore` excludes `node_modules/` and `.env`

---

### 🧠 Debugging Exercise

What would happen if you forgot to create the `.gitignore` before running `git add .`?

<details>
<summary>Answer</summary>

Git would stage the entire `node_modules/` folder — potentially 20,000+ files and 200MB+ of data. Your commit would be massive and your GitHub repo would be bloated. If you've already done this, you can fix it:

```bash
# Remove node_modules from Git's tracking (but not from disk)
git rm -r --cached node_modules
echo "node_modules/" >> .gitignore
git add .gitignore
git commit -m "fix: add .gitignore, remove node_modules from tracking"
```

This is why we always create `.gitignore` before the first `git add`.

</details>

---

> **Session Break** — You've scaffolded the project.
> When you return, you'll build the Express backend in [Step 3 — Backend](../03-backend/).

---

| | |
|:---|---:|
| [← Step 1: Orientation](../01-orientation/) | [Step 3 — Backend →](../03-backend/) |

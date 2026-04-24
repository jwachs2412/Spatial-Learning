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

## Words We'll Use for Setup (Define Once, Use Forever)

If any of these feel new, skim this list before continuing. Every command in this lesson uses one or more of these tools.

- **Terminal** (also called "shell" or "command line") — a text-only way to tell your computer what to do. Instead of clicking, you type. Every Mac has the `Terminal` app; every Windows machine has PowerShell or CMD.
- **Command** — one line you type into the terminal and press Enter. `mkdir dev-pulse` is a command.
- **Directory / Folder** — same thing. Programmers say "directory" more often; everyone else says "folder."
- **`cd` (change directory)** — a terminal command for "move into this folder." `cd dev-pulse` means "from wherever you are now, step into the `dev-pulse` folder."
- **`mkdir` (make directory)** — "create a new folder here." `mkdir server` creates an empty folder named `server`.
- **`ls` (list)** — "show me what's in this folder." Handy if you're not sure where you are.
- **`pwd` (print working directory)** — "tell me the full path of where I am right now."
- **`git`** — a version control tool. It tracks every change you make to your files so you can undo mistakes, collaborate, and see history. `git init` starts tracking a folder.
- **`node`** — the program that runs JavaScript outside the browser. When you write a backend in JavaScript/TypeScript, Node runs it.
- **`npm` (Node Package Manager)** — a tool that comes with Node. It installs third-party code (called "packages") into your project. `npm install express` downloads the Express package.
- **Package** — a bundle of pre-written code someone else published. Instead of writing your own HTTP server from scratch, you install `express` and use theirs.
- **`package.json`** — a file in your project that lists which packages you use, which scripts you can run, and some metadata about your project. Think of it as the project's ID card.
- **`node_modules/`** — the folder where `npm install` downloads every package. It can easily contain tens of thousands of files and hundreds of megabytes. Never commit this folder to git — just list it in `.gitignore` and let `npm install` rebuild it on each machine.

---

## 1. Create the Project Root

> **Key Concept: Project Root**
> The project root is the top-level folder that contains everything. Both your frontend (`client/`) and backend (`server/`) live inside it. This monorepo structure keeps related code together.

Open your terminal and navigate to wherever you keep projects:

```bash
mkdir dev-pulse
cd dev-pulse
```

**Reading those two commands:**

- `mkdir dev-pulse` — "make a new folder named `dev-pulse` in the current location." Nothing else happens; you're still outside the folder.
- `cd dev-pulse` — "step into the `dev-pulse` folder." Now every command you type runs inside that folder. (Verify with `pwd` if you're curious — it will print the full path ending in `/dev-pulse`.)

### Initialize Git

```bash
git init
```

**What this does:** `git init` creates a hidden folder called `.git` inside the current directory. That hidden folder is git's database where it stores the history of every change you make. You won't touch `.git` directly — you only talk to it through commands like `git add`, `git commit`, `git status`. After this command, the folder is a **repository** (repo for short).

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

### Reading the `.gitignore` Syntax

A `.gitignore` file is very simple — plain text, one pattern per line. Git reads it and skips any matching files when you run `git add`.

- `# anything after a hash is a comment.` Ignored by git; only for humans.
- A bare name like `node_modules/` means "ignore anything at any depth named `node_modules/`." The trailing slash means "this is a folder."
- `.env` with a leading dot is a **hidden file** on Unix-like systems. Git ignores them by filename, same as any other name.
- `*.log` uses a **wildcard**. `*` matches anything, so this ignores `error.log`, `debug.log`, `anything.log` — every file ending in `.log`.
- `npm-debug.log*` means "files starting with `npm-debug.log` and having anything after it" — catches `npm-debug.log.1`, `npm-debug.log.2025-01-01`, etc.

**Pro tip:** If you run `git status` and see a file you didn't expect in the "Untracked files" list, add a matching line to `.gitignore` — don't commit things you don't mean to track.

---

## 2. Set Up the Frontend (React + TypeScript)

> **Key Concept: Vite**
> Vite (French for "fast") is a build tool that creates React projects with TypeScript support built in. It replaces the older Create React App (CRA). Vite starts in milliseconds because it serves files directly to the browser during development instead of bundling everything first.

```bash
npm create vite@latest client -- --template react-ts
```

**Reading this command piece-by-piece (it's more than it looks):**

- `npm` — the Node package manager. This is the program you're telling what to do.
- `create vite@latest` — a special npm subcommand. It reads as "find the latest published version of the `create-vite` package, download it temporarily, and run it." You don't permanently install anything.
- `client` — the first argument to `create-vite`. It interprets this as "create the new project in a folder named `client`."
- `--` — a special separator. Everything **before** it belongs to npm; everything **after** it is passed through to the tool npm is running (Vite, in this case). Without `--`, npm might try to interpret the flags itself.
- `--template react-ts` — two words passed to Vite. `--template` is a flag that accepts a name; `react-ts` is the template identifier (React with TypeScript).

When you run this, Vite walks through a small set of questions (or skips them because the template is specified) and fills in the `client/` folder with a working React + TypeScript starter project.

```bash
cd client
npm install
cd ..
```

**What each line does:**

- `cd client` — step into the folder Vite just created.
- `npm install` — read this project's `package.json`, look at its list of dependencies, and download each one into `node_modules/`. The first time this runs it can take 30–60 seconds and download hundreds of MB. Subsequent runs are much faster because npm caches packages.
- `cd ..` — `..` means "the parent folder." So this steps back out to `dev-pulse/`.

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

**Reading these three commands:**

- `mkdir -p server/src` — `-p` (parents) means "create any missing parent folders too." This creates `server/` and `server/src/` in one go. Without `-p`, `mkdir server/src` would fail because `server/` doesn't exist yet.
- `cd server` — step into the new `server/` folder.
- `npm init -y` — "create a fresh `package.json` file with default values." The `-y` flag means "say yes to every default, don't prompt me." After this, you'll see a new `package.json` with your project name, version `1.0.0`, and empty sections for dependencies and scripts.

### Install Dependencies

> **Key Concept: Dependencies vs DevDependencies**
> **Dependencies** are packages your app needs to run (Express, cors). **DevDependencies** are packages you only need during development (TypeScript compiler, nodemon). The difference matters during deployment.

```bash
# Runtime dependencies — needed to run the server
npm install express cors

# TypeScript and development tools
npm install typescript ts-node nodemon
```

**Reading `npm install <names...>`:**

- `npm install` = "go to the npm registry (npm's central package catalog on the internet), download these packages, and record them in my `package.json`."
- You can list multiple names in one command, space-separated. Each one becomes an entry under `"dependencies"` in `package.json` automatically.
- Lines starting with `#` in a shell block are **shell comments** — they exist in this README for your benefit; you can copy/paste the whole block and the comments won't run anything.

The four packages we're installing here:

- **`express`** — a small framework that makes it easy to define HTTP routes in Node.js. Without it, building a web server from scratch is hundreds of lines; with it, it's a few.
- **`cors`** — the CORS middleware package we talked about in Step 1. Adds the right response headers so the browser lets the frontend talk to the backend.
- **`typescript`** — the TypeScript compiler. Turns `.ts` files into `.js` files so Node can run them.
- **`ts-node`** — a tool that lets you skip the compile step during development: it compiles and runs `.ts` files in one go.
- **`nodemon`** — watches your source files and automatically restarts your server every time you save a change. Saves you from manually restarting.

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

### Reading `tsconfig.json` Line-by-Line

First, the shape. `tsconfig.json` is a JSON file (same syntax rules you learned in Step 1 — double-quoted keys, no trailing commas). It has two top-level sections:

- `"compilerOptions"` — an object listing settings the TypeScript compiler should honor.
- `"include"` / `"exclude"` — arrays telling the compiler which files to read and which to skip.

Now every setting, one at a time:

- `"target": "ES2020"` — when the compiler produces JavaScript, use the ES2020 feature set. ES2020 is the version of JavaScript finalized in 2020. Modern Node.js supports it natively, so the compiler doesn't have to rewrite newer syntax into older equivalents.
- `"module": "commonjs"` — use Node's traditional module system. You write `import`/`export` in your TypeScript; the compiler converts that to Node's `require(...)` and `module.exports` under the hood.
- `"lib": ["ES2020"]` — which standard-library type definitions to load. Ensures you can use `Promise`, `Map`, `Array.prototype.flat`, and other ES2020-era features without TypeScript complaining "I've never heard of that."
- `"outDir": "./dist"` — when you run `tsc` (the compiler), put the output `.js` files here. `./` means "relative to this config file." So `./dist` resolves to `server/dist/`.
- `"rootDir": "./src"` — your source `.ts` files live here. Tells the compiler where to start looking.
- `"strict": true` — turn on every strict type-checking rule. This is one setting that flips on many. You'll get more errors up front, but each one is protecting you from a real bug.
- `"esModuleInterop": true` — a compatibility shim. Makes `import express from 'express'` work cleanly even for packages that use older module styles. Without this, you'd have to write `import * as express from 'express'` instead.
- `"skipLibCheck": true` — don't type-check the `.d.ts` (type definition) files that come with other packages. Those files are maintained by their authors and checking them slows compilation without catching real bugs.
- `"forceConsistentCasingInFileNames": true` — case-sensitivity enforcement. Prevents bugs where a file is referenced as `MyFile.ts` on one machine and `myfile.ts` on another (Linux servers care about case; macOS often doesn't).
- `"resolveJsonModule": true` — lets you `import data from './data.json'` and have TypeScript treat the file as a typed object.
- `"declaration": true`, `"declarationMap": true` — output `.d.ts` files (type definitions) for your own code. Useful if this project is consumed by another.
- `"sourceMap": true` — output `.js.map` files. When your compiled JavaScript throws an error, these maps let debuggers show you the original TypeScript line instead of the compiled JS line.
- `"types": ["node"]` — **critical**. Load the Node.js type definitions so `process`, `console`, `setTimeout`, and `Buffer` are known globals. Without this, TypeScript doesn't even know `console.log` exists.
- `"include": ["src/**/*"]` — compile every file inside `src/` (at any depth — `**` means "any number of subfolders").
- `"exclude": ["node_modules", "dist"]` — skip these folders. You never want to compile downloaded packages or your own previous compile output.

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

**What "scripts" are:** an object inside `package.json` that maps **short names** to **commands**. When you type `npm run dev`, npm looks up the `dev` script and runs the command string exactly as written. Scripts save you from memorizing long commands.

Reading each script:

- `"dev": "nodemon --exec ts-node src/index.ts"`
  - Run `nodemon` (the file watcher).
  - `--exec ts-node` tells nodemon, "use `ts-node` to execute the file, not plain `node`." That way your `.ts` source runs without a separate compile step.
  - `src/index.ts` is the entry file — the one file Node starts with.
  - When you save any `.ts` file, nodemon restarts the process. You never have to Ctrl+C and re-run by hand.
- `"build": "tsc"`
  - `tsc` is the TypeScript compiler. With no arguments, it reads `tsconfig.json` and compiles everything in `src/` to `dist/`. Used before deploying.
- `"start": "node dist/index.js"`
  - Run the compiled JavaScript file with plain Node. In production, servers run the already-compiled `.js`, not the `.ts` source — it's faster and doesn't require `ts-node` to be installed.

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

**What Prettier is:** a tool that reformats your code automatically so every file in the project looks identical in style — no manual nitpicking over spaces or quote style. It reads `.prettierrc` for its rules.

Reading each rule:

- `"semi": true` — always put semicolons at the end of statements. (Some styles skip them; we opt in.)
- `"singleQuote": true` — prefer `'single quotes'` over `"double quotes"` in JavaScript/TypeScript. Note that JSON files always use double quotes regardless — this rule only affects JS/TS.
- `"tabWidth": 2` — indent nested lines with 2 spaces. Two is the industry standard for JS/TS; four is common in Python.
- `"trailingComma": "all"` — put a comma after the last item in arrays and objects. Looks odd at first, but it makes diffs cleaner (adding a new last item only changes that one line).

---

## 6. First Commit

```bash
git add .
git commit -m "feat: scaffold dev-pulse project with client and server"
```

**Reading these two git commands:**

- `git add .` — "stage every file I've changed or added in the current folder (and its subfolders)." `.` is a shorthand for "everything here." Staging means "mark these changes as ready to be saved."
  - Because your `.gitignore` is in place, `node_modules/` and anything else listed there is skipped automatically.
- `git commit -m "..."` — "save the staged changes as a new commit with this message." A commit is a single snapshot in your project's history.
  - `-m` is short for "message" — the text after it is the commit message. It's a one-line summary of what you changed.
  - Messages starting with `feat:`, `fix:`, `docs:`, etc. follow a convention called **Conventional Commits**. Not required, but it makes history scannable: `feat:` = new feature, `fix:` = bug fix, `docs:` = documentation, `refactor:` = internal cleanup.

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

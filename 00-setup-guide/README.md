# Development Environment Setup Guide

Before building anything, you need a workspace. A carpenter doesn't start hammering without a workbench, and you don't start coding without a properly configured development environment.

This guide walks through every tool you need, what it does, and why you need it.

---

## Spatial Orientation — What Are We Setting Up?

```
YOUR COMPUTER
├── Node.js          ← Runs JavaScript outside the browser
│   └── npm          ← Installs and manages packages (comes with Node)
├── VS Code          ← Your code editor
│   └── Extensions   ← Add-ons that make VS Code smarter
├── Git              ← Tracks changes to your code (version control)
│   └── GitHub       ← Cloud storage for your Git repositories
├── ESLint           ← Finds problems in your code
└── Prettier         ← Formats your code consistently
```

Every item above exists for a specific reason. None are optional for professional development.

---

## 1. Installing Node.js

### What It Is
> [!NOTE]
> **Technical**: Node.js is a JavaScript runtime built on Chrome's V8 engine. It lets you execute JavaScript outside of a web browser.

> [!NOTE]
> **Plain English**: Normally, JavaScript only runs inside a web browser. Node.js lets you run JavaScript on your computer like any other programming language. This is what makes backend JavaScript possible.

### Why We Need It
- Our backend server runs on Node.js
- npm (Node Package Manager) comes bundled with it — we need npm to install libraries
- React development tools run on Node.js

### Installation

**macOS (recommended method — using nvm):**

nvm stands for "Node Version Manager." It lets you install and switch between multiple Node versions. This matters because different projects may need different Node versions.

Open your terminal (Terminal app on macOS, or the integrated terminal in VS Code). These commands can be run from any directory.

**First, check if Node.js is already installed:**

```bash
node --version
```

If you see a version number like `v20.x.x`, Node is already installed. Check if it's version 20 or higher. If so, you can skip to the "Verify installation" step below. If you see `command not found` or a version below 20, continue with the install.

**Check if nvm is already installed:**

```bash
nvm --version
```

If you see a version number, nvm is already installed — skip to `nvm install 20`. If you see `command not found`, install nvm first:

```bash
# Install nvm (only if nvm --version showed "command not found")
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash

# Close and reopen your terminal, then:
nvm install 20
nvm use 20
nvm alias default 20
```

**Why nvm over direct install**: Installing Node directly from nodejs.org works, but you're stuck with one version. Professional projects often require specific Node versions. nvm lets you switch instantly.

**Verify installation:**

```bash
node --version
# Should show v20.x.x

npm --version
# Should show 10.x.x
```

If both commands show version numbers, you're good. If not, revisit the installation steps above.

### When NOT to Use Node.js
Node.js is not ideal for CPU-intensive tasks (heavy math, video processing, machine learning). It's designed for I/O-heavy operations (web servers, APIs, file operations). For our purposes — building web application backends — it's an excellent choice.

---

## 2. Installing VS Code

### What It Is
> [!NOTE]
> **Technical**: Visual Studio Code is a free, open-source code editor developed by Microsoft. It supports extensions, integrated terminals, debugging, and Git integration.

> [!NOTE]
> **Plain English**: It's the text editor where you write code. It's free, it's fast, and almost every web developer uses it.

### Why VS Code (Not Sublime, Not WebStorm, Not Vim)
- **VS Code over Sublime Text**: VS Code has a vastly larger extension ecosystem, built-in terminal, and built-in Git support. Sublime is faster but offers less out of the box.
- **VS Code over WebStorm**: WebStorm is excellent but costs money. VS Code with the right extensions provides comparable features for free.
- **VS Code over Vim/Neovim**: Vim is powerful but has a steep learning curve unrelated to web development. Learn it later if you're curious. Don't let editor configuration block your learning.

### Installation

**First, check if VS Code is already installed:**

```bash
code --version
```

If you see a version number (e.g., `1.96.x`), VS Code is already installed — skip to the "Recommended Extensions" section below. If you see `command not found`, install it:

1. Go to https://code.visualstudio.com
2. Download for your operating system
3. Install and open it

### Recommended Extensions

Open VS Code, then press `Cmd+Shift+X` (macOS) or `Ctrl+Shift+X` (Windows/Linux) to open the Extensions panel. Search for each extension below — if it already shows "Installed" or "Disable" instead of "Install", you already have it and can skip it.

| Extension | What It Does | Why You Need It |
|-----------|-------------|-----------------|
| **ESLint** | Highlights code problems in real-time | Catches bugs before you run your code |
| **Prettier - Code formatter** | Auto-formats your code | Consistent formatting without thinking about it |
| **TypeScript Importer** | Auto-imports TypeScript modules | Saves time, reduces errors |
| **ES7+ React/Redux Snippets** | Code snippets for React | Type shortcuts to generate boilerplate |
| **Error Lens** | Shows errors inline (next to your code) | See problems without hovering |
| **GitLens** | Enhanced Git integration | See who changed what and when |
| **Thunder Client** | API testing tool inside VS Code | Test your backend without leaving the editor |
| **Auto Rename Tag** | Renames paired HTML/JSX tags | Change `<div>` and `</div>` updates automatically |

**Install from command line (optional):**

You can check which extensions are already installed first:

```bash
code --list-extensions
```

This prints every installed extension. Compare the list to the extensions below — only install what's missing:

```bash
code --install-extension dbaeumer.vscode-eslint
code --install-extension esbenp.prettier-vscode
code --install-extension pmneo.tsimporter
code --install-extension dsznajder.es7-react-js-snippets
code --install-extension usernamehw.errorlens
code --install-extension eamodio.gitlens
code --install-extension rangav.vscode-thunder-client
code --install-extension formulahendry.auto-rename-tag
```

(Running `code --install-extension` on an already-installed extension is harmless — it will just say "already installed" — but checking first is a good habit.)

### VS Code Settings to Configure

Open Settings JSON: `Cmd+Shift+P` → "Preferences: Open User Settings (JSON)"

Add these settings:

```json
{
  "editor.formatOnSave": true,
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  "editor.tabSize": 2,
  "editor.wordWrap": "on",
  "editor.minimap.enabled": false,
  "editor.bracketPairColorization.enabled": true,
  "files.autoSave": "onFocusChange",
  "typescript.updateImportsOnFileMove.enabled": "always",
  "emmet.includeLanguages": {
    "typescriptreact": "html"
  }
}
```

**What these do:**
- `formatOnSave`: Prettier runs every time you save → consistent formatting without effort
- `defaultFormatter`: Uses Prettier as the formatter for all files
- `tabSize: 2`: Industry standard for JavaScript/TypeScript
- `wordWrap`: Long lines wrap instead of scrolling horizontally
- `minimap: false`: Removes the tiny code preview on the right (it's just noise for now)
- `bracketPairColorization`: Colors matching brackets → easier to see nesting
- `autoSave`: Saves files when you click away → fewer "I forgot to save" moments
- `updateImportsOnFileMove`: When you move a file, import paths update automatically
- `emmet in typescriptreact`: Lets you use HTML shortcuts in React files

---

## 3. ESLint & Prettier Setup

### What They Are

**ESLint:**
- > [!NOTE]
> **Technical**: A static analysis tool that identifies problematic patterns in JavaScript/TypeScript code.
- > [!NOTE]
> **Plain English**: It reads your code and tells you about potential bugs, bad practices, and style issues — without running the code.

**Prettier:**
- > [!NOTE]
> **Technical**: An opinionated code formatter that enforces a consistent style by parsing your code and reprinting it.
- > [!NOTE]
> **Plain English**: It makes your code look consistent. Tabs vs spaces, semicolons or not, where line breaks go — Prettier decides so you don't have to argue about it.

### Why Both (Not Just One)
ESLint finds **bugs and bad patterns**. Prettier fixes **formatting**. They do different jobs. ESLint catches `x = x` (useless assignment). Prettier fixes inconsistent indentation. You need both.

### Setup
These will be configured per-project (not globally). Each project in this curriculum will include ESLint and Prettier configuration. We set them up inside each project because:

1. Different projects can have different rules
2. Anyone who clones your repo gets the same configuration
3. It's how professional teams work

You'll see the actual configuration files when we build Level 1.

---

## 4. Git Setup

### What Git Is
> [!NOTE]
> **Technical**: Git is a distributed version control system that tracks changes to files over time, enabling collaboration, branching, and history management.

> [!NOTE]
> **Plain English**: Git is an "undo system" for your entire project. It remembers every change you've ever made, lets you go back to any previous version, and lets multiple people work on the same code without overwriting each other.

### Why Git Exists
Before version control, developers would make copies of folders: `project-v1`, `project-v2`, `project-final`, `project-final-ACTUALLY-FINAL`. Git eliminates this by tracking every change as a "commit" — a snapshot of your code at a point in time.

### Installation

**macOS:**

These commands can be run from any directory in your terminal.

**First, check if Git is already installed:**

```bash
git --version
```

If you see a version number (e.g., `git version 2.39.x`), Git is already installed — skip to "Configure your identity" below. If you see `command not found` or a prompt to install developer tools, install it:

```bash
# Git comes with Xcode Command Line Tools
xcode-select --install
```

This opens a system dialog — click "Install" and wait for it to complete. Then verify:

```bash
git --version
# Should now show a version number
```

**Configure your identity** (Git needs to know who's making changes). These are global settings — run them from any directory.

First, check if your identity is already configured:

```bash
git config --global user.name
git config --global user.email
```

If both show your name and email, you're set — skip ahead. If either is blank, configure them:

```bash
git config --global user.name "Your Name"
git config --global user.email "your.email@example.com"
```

**Important**: Use the same email as your GitHub account. This links your commits to your GitHub profile.

### Core Git Concepts (Spatial View)

```
YOUR PROJECT FOLDER
│
├── Working Directory    ← Files as you see them right now
│       │
│       │  git add
│       ▼
├── Staging Area         ← Files selected for the next snapshot
│       │
│       │  git commit
│       ▼
├── Local Repository     ← All snapshots stored on YOUR computer
│       │
│       │  git push
│       ▼
└── Remote Repository    ← All snapshots stored on GITHUB (cloud)
        │
        │  git pull
        ▼
   (back to Working Directory)
```

**Plain English version:**
1. You edit files (Working Directory)
2. You select which changes to save: `git add` (Staging Area)
3. You take a snapshot: `git commit` (Local Repository)
4. You upload the snapshot to the cloud: `git push` (Remote Repository)

### Essential Git Commands

```bash
git init                    # Start tracking a project
git status                  # See what's changed
git add filename            # Stage specific file
git add .                   # Stage all changed files
git commit -m "message"     # Take a snapshot with a description
git push                    # Upload to GitHub
git pull                    # Download latest from GitHub
git log --oneline           # See commit history
git diff                    # See what changed (unstaged)
```

---

## 5. GitHub Setup

### What GitHub Is
> [!NOTE]
> **Technical**: GitHub is a cloud-based hosting service for Git repositories with collaboration features including pull requests, issues, actions, and project management.

> [!NOTE]
> **Plain English**: GitHub is where your code lives online. It's like Google Drive for code, but with superpowers — it tracks every change, lets people review your code, and is where employers look at your work.

### Why GitHub (Not GitLab, Not Bitbucket)
- Largest developer community
- Most employers expect a GitHub profile
- Best ecosystem of integrations
- GitHub Actions (CI/CD) is included free
- GitLab and Bitbucket are fine alternatives, but GitHub is the industry default

### Setup

**Check if you already have a GitHub account**: Go to https://github.com and try to log in. If you can log in, skip to "SSH Key Setup" below. If not, create an account:

1. Go to https://github.com and click "Sign Up"
2. Choose a professional username (this appears in your portfolio URLs)

### SSH Key Setup (Recommended)

SSH keys let you push to GitHub without typing your password every time. These commands can be run from any directory in your terminal.

**First, check if you already have an SSH key:**

```bash
ls -la ~/.ssh/id_ed25519.pub
```

If you see a file path (no error), you already have an SSH key. Skip to "Test your connection" below. If you see `No such file or directory`, generate one:

```bash
# Generate an SSH key
ssh-keygen -t ed25519 -C "your.email@example.com"
# Press Enter to accept default file location
# Enter a passphrase (or press Enter for none)

# Start the SSH agent
eval "$(ssh-agent -s)"

# Add your key to the agent
ssh-add ~/.ssh/id_ed25519

# Copy your public key
pbcopy < ~/.ssh/id_ed25519.pub
```

Then:
1. Go to GitHub → Settings → SSH and GPG Keys → New SSH Key
2. Paste the key

**Test your connection** (whether you just created a key or already had one):

```bash
ssh -T git@github.com
```

If you see "Hi username! You've successfully authenticated..." you're good. If you see "Permission denied," your SSH key may not be added to your GitHub account — go to GitHub → Settings → SSH Keys and add it.

---

## 6. Creating a GitHub Repository

This process will repeat for every app you build. Here's the standard workflow:

### From the command line:

> [!IMPORTANT]
> **You should be in:** Your project's root folder (e.g., `dev-pulse/`)

```bash
git init
git add .
git commit -m "Initial commit"
```

Then push to GitHub. Check if you have the GitHub CLI installed:

```bash
gh --version
```

If you see a version number, you can use it:

```bash
gh repo create your-repo-name --public --source=. --remote=origin --push
```

If you see `command not found`, that's fine — use the manual method below instead. (The GitHub CLI is optional; it's just a convenience.)

### From github.com (manual method):
1. Click "New Repository"
2. Name it (lowercase, hyphens: `dev-pulse`, `task-forge`)
3. Do NOT initialize with README (you'll create your own)
4. Copy the remote URL
5. In your terminal:

> [!IMPORTANT]
> **You should be in:** Your project's root folder (e.g., `dev-pulse/`)

```bash
git remote add origin git@github.com:yourusername/your-repo-name.git
git push -u origin main
```

---

## 7. Writing a Professional README

Every project needs a README. It's the first thing people (and employers) see.

### Template:

```markdown
# Project Name

Brief description of what this project does (1-2 sentences).

## Features

- Feature 1
- Feature 2
- Feature 3

## Tech Stack

- **Frontend**: React, TypeScript
- **Backend**: Node.js, Express
- **Database**: PostgreSQL (if applicable)
- **Deployment**: Vercel (frontend), Render (backend)

## Getting Started

### Prerequisites

- Node.js 20+
- npm 10+

### Installation

\```bash
# From wherever you keep your projects (e.g., ~/Desktop/Sites)
git clone https://github.com/yourusername/project-name.git
cd project-name

# Install frontend dependencies (from project-name/)
cd client
npm install

# Install backend dependencies (from project-name/)
cd ../server
npm install
\```

### Running Locally

\```bash
# Terminal 1 — Start the backend (from project-name/)
cd server
npm run dev

# Terminal 2 — Start the frontend (from project-name/)
cd client
npm run dev
\```

### Environment Variables

Create a `.env` file in the `server/` directory:

\```
PORT=3001
DATABASE_URL=your_database_url
\```

## API Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | /api/items | Get all items |
| POST | /api/items | Create new item |

## Deployment

- Frontend deployed at: [URL]
- Backend deployed at: [URL]

## What I Learned

Brief description of the key skills and concepts this project demonstrates.
```

---

## 8. Commit Message Best Practices

### Format

```
type: short description

Longer explanation if needed.
```

### Types

| Type | When to Use |
|------|-------------|
| `feat` | Adding a new feature |
| `fix` | Fixing a bug |
| `refactor` | Changing code structure without changing behavior |
| `style` | Formatting changes (no code logic change) |
| `docs` | Documentation changes |
| `test` | Adding or updating tests |
| `chore` | Maintenance tasks (updating dependencies, config) |

### Examples

```
feat: add mood entry form component
fix: resolve CORS error on production API calls
refactor: extract API calls into service layer
docs: add API endpoints to README
test: add unit tests for mood validation
chore: update React to v19
```

### Rules
1. Use present tense ("add" not "added")
2. Use imperative mood ("add" not "adds")
3. Keep the first line under 72 characters
4. Don't end the first line with a period
5. Explain WHY in the body, not WHAT (the code shows what)

---

## 9. Environment Variables

### What They Are
> [!NOTE]
> **Technical**: Environment variables are key-value pairs available to a running process, used to configure application behavior without hardcoding values.

> [!NOTE]
> **Plain English**: They're secret settings that your app reads when it starts. Things like database passwords, API keys, and configuration that changes between your computer and the production server.

### Why They Exist
You never want to put passwords or secrets in your code. If you push a database password to GitHub, anyone can see it. Environment variables keep secrets out of your code and let you change configuration without changing code.

### How They Work

```
YOUR CODE                     .env FILE (not in Git)
─────────                     ────────────────────
const port = process.env.PORT → PORT=3001
const dbUrl = process.env.DB  → DB=postgres://user:pass@host/db
```

Your code reads `process.env.VARIABLE_NAME`. The actual values come from a `.env` file on your machine, or from your hosting provider's settings in production.

### The .env File

```bash
# .env (in your project root)
PORT=3001
DATABASE_URL=postgres://localhost:5432/myapp
JWT_SECRET=my-super-secret-key
NODE_ENV=development
```

> [!WARNING]
> **Critical rule**: NEVER commit `.env` files to Git. They contain secrets.

---

## 10. The .gitignore File

### What It Is
> [!NOTE]
> **Technical**: A `.gitignore` file specifies intentionally untracked files that Git should ignore.

> [!NOTE]
> **Plain English**: It's a list of files and folders that Git should pretend don't exist. You use it to keep secrets, generated files, and junk out of your repository.

### Standard .gitignore for Our Projects

```gitignore
# Dependencies - downloaded packages, not your code
node_modules/

# Environment variables - SECRETS, never commit
.env
.env.local
.env.production

# Build output - generated files, not source code
dist/
build/

# OS files - system junk
.DS_Store
Thumbs.db

# IDE files - personal editor settings
.vscode/settings.json
.idea/

# Logs
*.log
npm-debug.log*

# Testing
coverage/
```

### Why Each Entry Matters

- **node_modules/**: Contains thousands of files from installed packages. Anyone can recreate this by running `npm install`. Committing it would bloat your repo from kilobytes to hundreds of megabytes.
- **.env**: Contains secrets. Committing it is a security vulnerability.
- **dist/build/**: Generated from your source code. The source is what matters — builds can be recreated.
- **.DS_Store**: macOS creates these invisible files in every folder. They contain nothing useful for your project.

---

## Setup Checklist

Before starting Level 1, verify everything. Open your terminal (these can be run from any directory):

```bash
# Node.js installed
node --version        # v20.x.x ✓

# npm installed
npm --version         # 10.x.x ✓

# Git installed
git --version         # 2.x.x ✓

# Git configured
git config user.name  # Your Name ✓
git config user.email # your@email.com ✓

# VS Code installed
code --version        # 1.x.x ✓

# GitHub account created and SSH key added
ssh -T git@github.com # Hi username! ✓
```

All green? You're ready for Level 1.

---

| | |
|:---|---:|
| [← Home](../) | [Level 1 — Start Learning →](../level-1-foundations/) |

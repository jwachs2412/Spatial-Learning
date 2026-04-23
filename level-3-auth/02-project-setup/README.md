# Step 2 — Project Setup: Scaffolding with Authentication

## Spatial Orientation

```
vault-note/
├── client/         ← React + React Router + TypeScript
├── server/         ← Express + TypeScript + bcrypt + JWT
│   └── .env        ← Database URL + JWT_SECRET
├── .gitignore
└── README.md
```

---

## 1. Create the Project

```bash
mkdir vault-note && cd vault-note
git init
```

Create `.gitignore`:

```gitignore
node_modules/
.env
.env.local
.env.production
dist/
build/
.DS_Store
*.log
coverage/
```

---

## 2. Frontend Setup

```bash
npm create vite@latest client -- --template react-ts
cd client
npm install react-router-dom
npm install
cd ..
```

> **Key Concept: React Router**
> Until now, our apps were single-page with view switching via state. React Router adds real URL-based navigation: `/login`, `/register`, `/notes`. The browser's address bar changes, back/forward buttons work, and users can bookmark pages.

---

## 3. Backend Setup

```bash
mkdir -p server/src/{db,routes,middleware,types}
cd server
npm init -y
```

### Install Dependencies

```bash
# Runtime dependencies
npm install express cors pg dotenv bcryptjs jsonwebtoken

# TypeScript and dev tools
npm install typescript ts-node nodemon
```

| Package | What It Does | New in Level 3? |
|---------|-------------|-----------------|
| `bcryptjs` | Hashes passwords (one-way) | Yes |
| `jsonwebtoken` | Creates and verifies JWT tokens | Yes |
| `express` | Web framework | No |
| `cors` | Cross-origin requests | No |
| `pg` | PostgreSQL client | No |
| `dotenv` | Environment variables | No |

> **Why `bcryptjs` (not `bcrypt`)?** `bcryptjs` is a pure JavaScript implementation — no native compilation required. `bcrypt` (without the `js`) uses native C++ bindings that can cause installation issues across platforms. `bcryptjs` is slightly slower but works everywhere without build tools.

### Install Type Definitions

```bash
npm install @types/node @types/express @types/cors @types/pg @types/bcryptjs @types/jsonwebtoken
```

> **Why not --save-dev?**
> Deployment platforms skip devDependencies during production builds. But `tsc` (the TypeScript compiler) needs these type definitions to compile your code. If they're in devDependencies, you get: `error TS7016: Could not find a declaration file for module 'express'`.
>
> This is the most common deployment failure. Always install @types as regular dependencies.

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
> Without this, you'll get `Cannot find name 'process'` and `Cannot find name 'console'` errors during deployment. This tells TypeScript to include Node.js type definitions.

### Add Scripts

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

Your `server/package.json` `dependencies` should include all @types packages:

```json
{
  "dependencies": {
    "@types/bcryptjs": "...",
    "@types/cors": "...",
    "@types/express": "...",
    "@types/jsonwebtoken": "...",
    "@types/node": "...",
    "@types/pg": "...",
    "bcryptjs": "...",
    "cors": "...",
    "dotenv": "...",
    "express": "...",
    "jsonwebtoken": "...",
    "nodemon": "...",
    "pg": "...",
    "ts-node": "...",
    "typescript": "..."
  }
}
```

> ⚠️ **If @types are in devDependencies, fix it:**
> ```bash
> npm uninstall @types/node @types/express @types/cors @types/pg @types/bcryptjs @types/jsonwebtoken
> npm install @types/node @types/express @types/cors @types/pg @types/bcryptjs @types/jsonwebtoken
> ```

---

## 4. Create the Database

```bash
createdb vaultnote
```

---

## 5. Configure Environment Variables

Create `server/.env`:

```bash
PORT=3001
DATABASE_URL=postgresql://localhost:5432/vaultnote
CORS_ORIGIN=http://localhost:5173
JWT_SECRET=your-super-secret-key-change-this-in-production
JWT_EXPIRES_IN=7d
```

| Variable | What It Is | Security Note |
|----------|-----------|---------------|
| `JWT_SECRET` | The key used to sign JWT tokens | **Must be long and random in production.** If someone gets this, they can forge tokens for any user. |
| `JWT_EXPIRES_IN` | How long tokens are valid | 7 days is reasonable for development. Consider shorter for production. |

**Generate a production-grade secret:**
```bash
node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"
```

---

## 6. Prettier Configuration

Create `.prettierrc` in the project root:

```json
{
  "semi": true,
  "singleQuote": true,
  "tabWidth": 2,
  "trailingComma": "all"
}
```

---

## 7. First Commit

```bash
cd ..
git add .
git commit -m "feat: scaffold vault-note with auth dependencies"
```

### ✅ Checkpoint

- [ ] PostgreSQL has `vaultnote` database
- [ ] `server/.env` has JWT_SECRET and JWT_EXPIRES_IN
- [ ] `server/package.json` has `bcryptjs`, `jsonwebtoken`, and ALL @types in `dependencies`
- [ ] `server/tsconfig.json` has `"types": ["node"]`
- [ ] `client/` has `react-router-dom` installed

---

> **Session Break** — Project scaffolded with auth dependencies.
> When you return, you'll design the database schema in [Step 3 — Database](../03-database/).

---

| | |
|:---|---:|
| [← Step 1: Orientation](../01-orientation/) | [Step 3 — Database →](../03-database/) |

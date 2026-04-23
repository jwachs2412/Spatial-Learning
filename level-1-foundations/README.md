# Level 1 — DevPulse (Developer Mood Tracker)

## What You Will Build

A single-page application where a developer logs their daily mood, energy level, and a short note. The frontend displays a history of entries. The backend serves and receives data through a REST API. Data lives in memory (no database yet).

### Why This App

It is simple enough to focus on fundamentals but real enough to deploy and show someone. It forces you to understand the client-server split immediately.

### How This Curriculum Teaches

This is **not** a copy/paste tutorial. Each lesson will:

1. **Explain the concept first** — You'll understand the *why* before seeing the *what*
2. **Challenge you to try it yourself** — Solutions are in expandable sections. Attempt the challenge first.
3. **Break down the code** — Key lines are explained so you understand what they do and why
4. **Warn you about common mistakes** — Real errors you'll likely hit, with fixes
5. **Verify your progress** — Checkpoints confirm your code works before moving on

You *will* get stuck. That's the point. Getting unstuck is the most important skill in engineering.

---

## Architecture — Where Does Everything Live?

```
┌─────────────────────────────────────────────────────────────┐
│                        THE INTERNET                         │
│                                                             │
│   ┌───────────────────┐         ┌───────────────────────┐   │
│   │   FRONTEND (React)│  HTTP   │   BACKEND (Express)   │   │
│   │                   │────────▶│                       │   │
│   │  - Components     │◀────────│  - REST API           │   │
│   │  - State          │         │  - Routes             │   │
│   │  - TypeScript     │         │  - In-memory data     │   │
│   │                   │         │                       │   │
│   │  Runs in BROWSER  │         │  Runs on SERVER       │   │
│   └───────────────────┘         └───────────────────────┘   │
│                                                             │
│   Deployed to:                  Deployed to:                │
│   Vercel / Netlify              Render / Railway / Fly.io   │
└─────────────────────────────────────────────────────────────┘
```

---

## Data Flow — How Information Travels

```
USER ACTION                 FRONTEND              BACKEND                    DATA
──────────                  ──────────            ──────────                ──────
Click "Submit" ──────▶ Form collects data ─▶ POST /api/entries ──────▶ Saved to array
                                                    │
                                              Validates data
                                              Returns new entry
                                                    │
Page loads     ──────▶ useEffect fires ──────▶ GET /api/entries ──────▶ Reads array
                       Updates state ◀───────── Returns JSON ◀──────── Returns data
                       Re-renders UI
```

---

## Folder Structure — What You Will Create

```
dev-pulse/
├── client/                    ← Frontend (React + TypeScript)
│   ├── src/
│   │   ├── components/
│   │   │   ├── EntryForm.tsx    ← Form for new entries
│   │   │   ├── EntryList.tsx    ← Display list of entries
│   │   │   └── MoodChart.tsx    ← Simple mood visualization
│   │   ├── services/
│   │   │   └── api.ts           ← HTTP calls to backend
│   │   ├── types/
│   │   │   └── index.ts         ← Shared TypeScript interfaces
│   │   ├── App.tsx              ← Root component
│   │   ├── App.css              ← Styles
│   │   └── main.tsx             ← Entry point
│   ├── index.html
│   ├── package.json
│   ├── tsconfig.json
│   └── vite.config.ts
│
├── server/                    ← Backend (Node + Express + TypeScript)
│   ├── src/
│   │   ├── types/
│   │   │   └── index.ts         ← Type definitions
│   │   ├── routes/
│   │   │   └── entries.ts       ← API route handlers
│   │   └── index.ts             ← Server entry point
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
| 1 | [01 — Orientation](./01-orientation/) | Understand client-server model, HTTP, REST, JSON |
| 2 | [02 — Project Setup](./02-project-setup/) | Scaffold both projects, configure TypeScript |
| 3 | [03 — Backend](./03-backend/) | Build Express server with typed routes |
| 4-5 | [04 — Frontend](./04-frontend/) | Build React components with hooks |
| 6 | [05 — Connecting](./05-connecting/) | Wire frontend to backend, verify data flow |
| 7-8 | [06 — Deployment](./06-deployment/) | Deploy to Vercel + Render/Railway |
| 8 | [07 — Growth](./07-growth/) | Review skills, prepare for interviews |

**Estimated total: 8–10 sessions (30–60 min each)**

---

| | |
|:---|---:|
| [← Curriculum Overview](../00-curriculum-overview/) | [Step 1 — Orientation →](./01-orientation/) |

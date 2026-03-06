# Level 1 — DevPulse: Developer Mood Tracker

## What You Will Build

DevPulse is a single-page application where a developer logs their daily mood, energy level, and a short note about their day. The app displays a history of all entries. It has a React + TypeScript frontend and a Node.js + Express backend connected by a REST API.

When you're done, it will be deployed to the internet — a real application that anyone can visit.

## Why This App

This app exists at the perfect intersection of "simple enough to finish" and "complex enough to learn from." It forces you to:

1. Build a React frontend with real state management
2. Build an Express backend that handles real HTTP requests
3. Connect them — the fundamental skill of full-stack development
4. Deploy both — making it real and publicly accessible

## What You Will Learn

By the end of Level 1, you will understand:

- How the web works (client sends request → server sends response)
- How React components render and update
- How TypeScript catches bugs before your code runs
- How Express receives and responds to HTTP requests
- How REST APIs are structured
- How to deploy a frontend and backend separately
- How Git tracks your progress

## Level 1 Architecture (The Big Picture)

```
┌─────────────────────────────────────────────────────────────────┐
│                     YOUR APPLICATION                            │
│                                                                 │
│                                                                 │
│   ┌─────────────────────────┐    ┌─────────────────────────┐   │
│   │       FRONTEND          │    │       BACKEND           │   │
│   │     (React + TS)        │    │    (Node + Express)     │   │
│   │                         │    │                         │   │
│   │  ┌───────────────────┐  │    │  ┌───────────────────┐  │   │
│   │  │    App Component  │  │    │  │   Express Server   │  │   │
│   │  │                   │  │    │  │                   │  │   │
│   │  │  ┌─────────────┐  │  │    │  │  GET /api/entries │  │   │
│   │  │  │ EntryForm   │  │  │    │  │  POST /api/entries│  │   │
│   │  │  └─────────────┘  │  │    │  │                   │  │   │
│   │  │  ┌─────────────┐  │  │    │  │  ┌─────────────┐  │  │   │
│   │  │  │ EntryList   │  │  │    │  │  │ In-memory   │  │  │   │
│   │  │  └─────────────┘  │  │    │  │  │ data store  │  │  │   │
│   │  │  ┌─────────────┐  │  │    │  │  └─────────────┘  │  │   │
│   │  │  │ MoodChart   │  │  │    │  └───────────────────┘  │   │
│   │  │  └─────────────┘  │  │    │                         │   │
│   │  └───────────────────┘  │    └─────────────────────────┘   │
│   │                         │                                   │
│   │  Runs in: BROWSER       │    Runs on: SERVER               │
│   │  Port: 5173 (dev)       │    Port: 3001 (dev)              │
│   │  Deploy: Vercel          │    Deploy: Render                │
│   └─────────────────────────┘    └─────────────────────────────┘
│                                                                 │
│                    Connected by HTTP (REST API)                  │
└─────────────────────────────────────────────────────────────────┘
```

## Data Flow (How Information Moves)

When a user submits a mood entry, here's exactly what happens:

```
1. User fills out the form and clicks "Submit"
   │
2. React's onChange handlers store form data in component state
   │
3. On submit, the frontend sends an HTTP POST request:
   │  POST http://localhost:3001/api/entries
   │  Body: { mood: "happy", energy: 4, note: "Great coding day" }
   │
4. Express receives the request
   │  → The route handler extracts the body
   │  → Validates the data
   │  → Stores it in the in-memory array
   │  → Sends back: { success: true, entry: {...} }
   │
5. React receives the response
   │  → Updates component state with the new entry
   │  → React re-renders the EntryList with the new data
   │
6. User sees their new entry on screen
```

## Folder Structure (What Goes Where and Why)

```
dev-pulse/
├── client/                   ← FRONTEND (everything the browser runs)
│   ├── src/
│   │   ├── components/       ← React components (UI building blocks)
│   │   │   ├── EntryForm.tsx ← The form for submitting mood entries
│   │   │   ├── EntryList.tsx ← The list displaying all entries
│   │   │   └── MoodChart.tsx ← Simple visual chart of moods
│   │   ├── types/            ← TypeScript type definitions
│   │   │   └── index.ts      ← Shared types (MoodEntry, etc.)
│   │   ├── services/         ← Functions that talk to the backend
│   │   │   └── api.ts        ← fetch calls (GET, POST)
│   │   ├── App.tsx           ← Root component (puts everything together)
│   │   ├── App.css           ← Styles
│   │   └── main.tsx          ← Entry point (where React mounts)
│   ├── index.html            ← The one HTML file React injects into
│   ├── package.json          ← Frontend dependencies and scripts
│   ├── tsconfig.json         ← TypeScript configuration
│   └── vite.config.ts        ← Build tool configuration
│
├── server/                   ← BACKEND (everything the server runs)
│   ├── src/
│   │   ├── routes/           ← API route definitions
│   │   │   └── entries.ts    ← /api/entries routes
│   │   ├── types/            ← Backend TypeScript types
│   │   │   └── index.ts
│   │   └── index.ts          ← Server entry point (starts Express)
│   ├── package.json          ← Backend dependencies and scripts
│   └── tsconfig.json         ← Backend TypeScript configuration
│
├── .gitignore                ← Files Git should ignore
└── README.md                 ← Project documentation
```

**Why this structure?**
- `client/` and `server/` are completely separate. They could be in different repositories. This mirrors how many production apps work — frontend and backend are independent projects that communicate over HTTP.
- `src/` contains source code. `package.json` and config files live at the root of each project.
- `components/` holds UI pieces. `services/` holds code that talks to the outside world. `types/` holds TypeScript definitions. Each folder has one clear job.

## Session Guide

Level 1 takes approximately **8–10 sessions** (30–60 minutes each).

| Lesson | Sessions | Notes |
|--------|----------|-------|
| 01 — Orientation | 1 | Read and absorb in one sitting |
| 02 — Project Setup | 2 | Break after frontend scaffold |
| 03 — Backend | 2 | Break after routes defined |
| 04 — Frontend | 3 | Breaks after EntryList and after MoodChart |
| 05 — Connecting | 2 | Break after API service created |
| 06 — Deployment | 1 | Complete in one sitting |
| 07 — Growth Review | 1 | Complete in one sitting |

Look for **Session Break** markers inside each lesson — they tell you exactly when to pause.

## Learning Path

Work through these sections in order:

1. **[01 — Spatial Orientation](./01-orientation/)** — Understand the web, client-server model, and where your code runs
2. **[02 — Project Setup](./02-project-setup/)** — Initialize the project, configure tools, create the skeleton
3. **[03 — Building the Backend](./03-backend/)** — Create the Express server and REST API
4. **[04 — Building the Frontend](./04-frontend/)** — Create the React application and components
5. **[05 — Connecting Frontend to Backend](./05-connecting/)** — Wire them together with HTTP requests
6. **[06 — Deployment](./06-deployment/)** — Ship it to the internet
7. **[07 — Growth Review](./07-growth/)** — What you learned and how to talk about it

---

| | |
|:---|---:|
| [← Home](../) | [01 — Start Here →](./01-orientation/) |

`Level 1` **Step 4 of 7** — Building the Frontend

# 04 — Building the Frontend

## Spatial Orientation

```
dev-pulse/
├── client/          ← ★ WE ARE HERE ★
│   └── src/
│       ├── components/     ← UI building blocks (we'll build these)
│       │   ├── EntryForm.tsx
│       │   ├── EntryList.tsx
│       │   └── MoodChart.tsx
│       ├── services/       ← Backend communication (we'll build this)
│       │   └── api.ts
│       ├── types/          ← Shared types (we'll build this)
│       │   └── index.ts
│       ├── App.tsx         ← Root component (we'll rewrite this)
│       ├── App.css         ← Styles (we'll rewrite this)
│       └── main.tsx        ← Entry point (leave as is)
└── server/          ← Already built ✓
```

**What layer are we in?** The CLIENT layer. This code runs in the user's web browser. It's what they see and interact with. It sends HTTP requests to the backend to get and send data.

---

## How React Works (The Mental Model)

Before writing components, understand what React does:

```
1. YOU DEFINE COMPONENTS
   │  Components are functions that return JSX (HTML-like syntax)
   │  Each component manages its own state (data that can change)
   │
2. REACT BUILDS A VIRTUAL DOM
   │  React creates a lightweight copy of the page in memory
   │  This is fast because it's just JavaScript objects, not real DOM
   │
3. WHEN STATE CHANGES, REACT RE-RENDERS
   │  You call setState → React compares old and new virtual DOMs
   │  Only the parts that actually changed get updated in the real DOM
   │
4. THE USER SEES THE UPDATE
   The browser paints the updated DOM to the screen
```

> [!NOTE]
> **Technical**: React is a declarative, component-based UI library that manages DOM updates through a virtual DOM reconciliation algorithm. State changes trigger re-renders, where React diffs the virtual DOM and applies minimal DOM mutations.

> [!NOTE]
> **Plain English**: You tell React "here's what the screen should look like for this data." When the data changes, React figures out the smallest possible change to the actual page and makes it. You never manually update the page — React does it for you.

### Why React (Not Vue, Not Svelte, Not Angular)

- **React over Vue**: React has a larger job market, larger ecosystem, and more learning resources. Vue is excellent and some prefer its simplicity, but React proficiency is more broadly demanded by employers.
- **React over Angular**: Angular is a full framework with strong opinions. React is a library — it does one thing (UI) and lets you choose everything else. This teaches you to understand each piece rather than relying on a framework's magic.
- **React over Svelte**: Svelte compiles away at build time and has less runtime overhead. But its ecosystem is smaller, fewer jobs require it, and React's patterns (components, state, effects) are more transferable to other frameworks.

---

## Step 1: Define Shared Types

**Where?** `client/src/types/index.ts`

These types match what the backend sends and receives. Having the same types on both sides ensures the frontend and backend agree on data shapes.

First, create the folder structure. Open your terminal:

> [!IMPORTANT]
> **You should be in:** `dev-pulse/` (the project root)

```bash
mkdir -p client/src/types
mkdir -p client/src/services
mkdir -p client/src/components
```

Now in VS Code, create a new file at `client/src/types/index.ts` (right-click on the `client/src/types` folder → New File → name it `index.ts`).

Add this code to `client/src/types/index.ts`:

```typescript
export interface MoodEntry {
  id: number;
  mood: 'happy' | 'neutral' | 'frustrated' | 'tired' | 'energized';
  energy: number;
  note: string;
  createdAt: string;
}

export interface CreateEntryRequest {
  mood: MoodEntry['mood'];
  energy: number;
  note: string;
}
```

**Note**: These are identical to the backend types. In a real monorepo, you might share a single types package. For now, duplicating them is fine — it reinforces the concept that frontend and backend are separate systems that must agree on a contract.

---

## Step 2: Build the API Service

**Where?** `client/src/services/api.ts`

**What is this file?** It contains all functions that communicate with the backend. By putting these in one place, your components don't need to know about URLs, HTTP methods, or fetch syntax — they just call `getEntries()` or `createEntry(data)`.

In VS Code, create a new file at `client/src/services/api.ts` (right-click on the `client/src/services` folder → New File → name it `api.ts`).

Add this code to `client/src/services/api.ts`:

```typescript
import { MoodEntry, CreateEntryRequest } from '../types';

// The backend URL — reads from environment variable in production
const API_URL = import.meta.env.VITE_API_URL || 'http://localhost:3001/api';

/**
 * Fetch all mood entries from the backend.
 *
 * What happens:
 * 1. Sends GET request to /api/entries
 * 2. Backend queries its data store
 * 3. Backend sends back JSON array of entries
 * 4. We parse the JSON and return it as typed data
 */
export async function getEntries(): Promise<MoodEntry[]> {
  const response = await fetch(`${API_URL}/entries`);

  if (!response.ok) {
    throw new Error(`Failed to fetch entries: ${response.statusText}`);
  }

  return response.json();
}

/**
 * Create a new mood entry.
 *
 * What happens:
 * 1. Sends POST request to /api/entries with the entry data
 * 2. Backend validates the data
 * 3. Backend creates the entry and assigns an ID
 * 4. Backend sends back the created entry
 * 5. We return the created entry
 */
export async function createEntry(data: CreateEntryRequest): Promise<MoodEntry> {
  const response = await fetch(`${API_URL}/entries`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
    },
    body: JSON.stringify(data),
  });

  if (!response.ok) {
    const error = await response.json();
    throw new Error(error.error || 'Failed to create entry');
  }

  return response.json();
}
```

### Key Concepts

**`fetch()`**
- > [!NOTE]
> **Technical**: The Fetch API provides a JavaScript interface for making HTTP requests and processing responses. It returns Promises.
- > [!NOTE]
> **Plain English**: `fetch` is JavaScript's built-in way to send HTTP requests. You give it a URL and options, and it sends the request and gives you back the response.

**`async/await`**
- > [!NOTE]
> **Technical**: async/await is syntactic sugar over Promises, allowing asynchronous code to be written in a synchronous style.
- > [!NOTE]
> **Plain English**: HTTP requests take time (the data has to travel over the network). `await` says "pause here until the response comes back." Without it, the code would continue before the data arrives.

**`import.meta.env.VITE_API_URL`**
- > [!NOTE]
> **Technical**: Vite exposes environment variables prefixed with `VITE_` through `import.meta.env`.
- > [!NOTE]
> **Plain English**: In development, the backend is at `localhost:3001`. In production, it's at a different URL. This reads the URL from an environment variable so the code works in both places without changes. Vite requires the prefix `VITE_` for security — it prevents accidentally exposing server-only env vars in browser code.

**Why a separate service file instead of fetch inside components?**
- If 5 components need entries, you don't want 5 copies of the fetch logic
- If the API URL changes, you update one file
- If you switch from fetch to axios later, you update one file
- Components focus on UI, services focus on data fetching — separation of concerns

---

## Step 3: Build the EntryForm Component

**Where?** `client/src/components/EntryForm.tsx`

**What is it?** A form where users select their mood, energy level, and write a note. When submitted, it calls the API to create a new entry.

In VS Code, create a new file at `client/src/components/EntryForm.tsx` (right-click on the `client/src/components` folder → New File → name it `EntryForm.tsx`).

Add this code to `client/src/components/EntryForm.tsx`:

```tsx
import { useState } from 'react';
import { CreateEntryRequest, MoodEntry } from '../types';

// The five mood options with display labels and emoji
const MOOD_OPTIONS: { value: MoodEntry['mood']; label: string }[] = [
  { value: 'happy', label: 'Happy 😊' },
  { value: 'energized', label: 'Energized ⚡' },
  { value: 'neutral', label: 'Neutral 😐' },
  { value: 'tired', label: 'Tired 😴' },
  { value: 'frustrated', label: 'Frustrated 😤' },
];

interface EntryFormProps {
  onSubmit: (data: CreateEntryRequest) => Promise<void>;
}

export function EntryForm({ onSubmit }: EntryFormProps) {
  // ─── STATE ─────────────────────────────────────────────────
  // Each piece of form data gets its own state variable.
  // When the user types or selects something, the state updates,
  // and React re-renders the component to reflect the change.
  const [mood, setMood] = useState<MoodEntry['mood']>('neutral');
  const [energy, setEnergy] = useState(3);
  const [note, setNote] = useState('');
  const [isSubmitting, setIsSubmitting] = useState(false);
  const [error, setError] = useState('');

  // ─── FORM SUBMISSION ───────────────────────────────────────
  const handleSubmit = async (e: React.FormEvent) => {
    // Prevent the browser's default form behavior (page reload)
    e.preventDefault();

    // Client-side validation (for quick user feedback)
    if (note.trim().length === 0) {
      setError('Please write a note about your day');
      return;
    }

    setIsSubmitting(true);
    setError('');

    try {
      await onSubmit({ mood, energy, note: note.trim() });
      // Reset form on success
      setMood('neutral');
      setEnergy(3);
      setNote('');
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Something went wrong');
    } finally {
      setIsSubmitting(false);
    }
  };

  // ─── RENDER ────────────────────────────────────────────────
  return (
    <form onSubmit={handleSubmit} className="entry-form">
      <h2>How are you feeling?</h2>

      {/* Mood Selection */}
      <div className="mood-selector">
        {MOOD_OPTIONS.map((option) => (
          <button
            key={option.value}
            type="button"
            className={`mood-button ${mood === option.value ? 'active' : ''}`}
            onClick={() => setMood(option.value)}
          >
            {option.label}
          </button>
        ))}
      </div>

      {/* Energy Level */}
      <div className="energy-selector">
        <label htmlFor="energy">
          Energy Level: <strong>{energy}</strong>/5
        </label>
        <input
          id="energy"
          type="range"
          min={1}
          max={5}
          step={1}
          value={energy}
          onChange={(e) => setEnergy(Number(e.target.value))}
        />
      </div>

      {/* Note */}
      <div className="note-input">
        <label htmlFor="note">What happened today?</label>
        <textarea
          id="note"
          value={note}
          onChange={(e) => setNote(e.target.value)}
          placeholder="Solved a tricky bug, learned about TypeScript generics..."
          rows={3}
          maxLength={500}
        />
        <span className="char-count">{note.length}/500</span>
      </div>

      {/* Error Message */}
      {error && <p className="error-message">{error}</p>}

      {/* Submit Button */}
      <button
        type="submit"
        className="submit-button"
        disabled={isSubmitting}
      >
        {isSubmitting ? 'Saving...' : 'Log Entry'}
      </button>
    </form>
  );
}
```

### React Concepts Explained

**`useState`**
- > [!NOTE]
> **Technical**: `useState` is a React Hook that declares a state variable. It returns a pair: the current state value and a function to update it. When the state updates, React re-renders the component.
- > [!NOTE]
> **Plain English**: It's a variable that React "watches." When you change it (using the setter function), React automatically updates the screen to show the new value. Normal variables don't trigger re-renders.

```typescript
const [mood, setMood] = useState<MoodEntry['mood']>('neutral');
//     │       │                  │                    │
//     │       │                  │                    └── Initial value
//     │       │                  └── Type annotation (TypeScript)
//     │       └── Function to update the value
//     └── Current value
```

**`React.FormEvent` and `e.preventDefault()`**
- HTML forms by default reload the page on submit. This is old web behavior from before JavaScript existed. `e.preventDefault()` stops that reload so React stays in control.

**Props (`{ onSubmit }: EntryFormProps`)**
- > [!NOTE]
> **Technical**: Props are arguments passed to React components, typed by an interface. They flow one direction: parent to child.
- > [!NOTE]
> **Plain English**: Props are how a parent component sends data or functions to a child component. The `EntryForm` doesn't know how to save an entry — its parent (App) passes in an `onSubmit` function that handles that.

**Why `onSubmit` is a prop (not hardcoded)**:
- The form's job is to collect data and call a function. It shouldn't know about HTTP or APIs.
- The parent component (App) coordinates between the form and the API service.
- This makes the form reusable — you could use it in a different app with a different backend.

---

## Step 4: Build the EntryList Component

**Where?** `client/src/components/EntryList.tsx`

In VS Code, create a new file at `client/src/components/EntryList.tsx` (right-click on the `client/src/components` folder → New File → name it `EntryList.tsx`).

Add this code to `client/src/components/EntryList.tsx`:

```tsx
import { MoodEntry } from '../types';

interface EntryListProps {
  entries: MoodEntry[];
}

// Map mood values to display emojis
const MOOD_EMOJI: Record<MoodEntry['mood'], string> = {
  happy: '😊',
  energized: '⚡',
  neutral: '😐',
  tired: '😴',
  frustrated: '😤',
};

export function EntryList({ entries }: EntryListProps) {
  if (entries.length === 0) {
    return (
      <div className="entry-list-empty">
        <p>No entries yet. Log your first mood above!</p>
      </div>
    );
  }

  return (
    <div className="entry-list">
      <h2>Your Mood History</h2>
      {entries.map((entry) => (
        <div key={entry.id} className="entry-card">
          <div className="entry-header">
            <span className="entry-mood">
              {MOOD_EMOJI[entry.mood]} {entry.mood}
            </span>
            <span className="entry-energy">
              Energy: {'●'.repeat(entry.energy)}{'○'.repeat(5 - entry.energy)}
            </span>
          </div>
          <p className="entry-note">{entry.note}</p>
          <time className="entry-date">
            {new Date(entry.createdAt).toLocaleDateString('en-US', {
              weekday: 'short',
              month: 'short',
              day: 'numeric',
              hour: '2-digit',
              minute: '2-digit',
            })}
          </time>
        </div>
      ))}
    </div>
  );
}
```

### Concepts

**`entries.map()`**
- > [!NOTE]
> **Technical**: `Array.map()` transforms each element and returns a new array. In JSX, it's used to render a list of elements from an array of data.
- > [!NOTE]
> **Plain English**: "For each entry in the array, create a card." If there are 3 entries, you get 3 cards. If there are 100, you get 100. React handles the rendering.

**`key={entry.id}`**
- > [!NOTE]
> **Technical**: React uses the `key` prop to identify elements in a list, enabling efficient re-rendering by tracking which items changed, were added, or were removed.
- > [!NOTE]
> **Plain English**: When the list changes (new entry added, one removed), React needs to know which items are which. The `key` tells React "this card belongs to entry #3." Without it, React has to guess, which can cause bugs and slow rendering.

**`Record<MoodEntry['mood'], string>`**
- > [!NOTE]
> **Technical**: `Record<K, V>` is a TypeScript utility type that creates an object type with keys of type K and values of type V.
- > [!NOTE]
> **Plain English**: "An object where every mood value has a corresponding string." TypeScript will error if you forget a mood — ensuring your emoji map is always complete.

---

> [!TIP]
> **Session Break** — You've built the EntryForm and EntryList components. Save your work and take a break.
> When you return, you'll build the MoodChart and wire everything together in the App component.

---

## Step 5: Build the MoodChart Component

**Where?** `client/src/components/MoodChart.tsx`

A simple visual summary of mood entries. No chart library — just CSS and HTML.

In VS Code, create a new file at `client/src/components/MoodChart.tsx` (right-click on the `client/src/components` folder → New File → name it `MoodChart.tsx`).

Add this code to `client/src/components/MoodChart.tsx`:

```tsx
import { MoodEntry } from '../types';

interface MoodChartProps {
  entries: MoodEntry[];
}

const MOOD_COLORS: Record<MoodEntry['mood'], string> = {
  happy: '#4ade80',
  energized: '#facc15',
  neutral: '#94a3b8',
  tired: '#818cf8',
  frustrated: '#f87171',
};

export function MoodChart({ entries }: MoodChartProps) {
  if (entries.length === 0) return null;

  // Count occurrences of each mood
  const moodCounts = entries.reduce(
    (acc, entry) => {
      acc[entry.mood] = (acc[entry.mood] || 0) + 1;
      return acc;
    },
    {} as Record<string, number>
  );

  const maxCount = Math.max(...Object.values(moodCounts));

  return (
    <div className="mood-chart">
      <h2>Mood Overview</h2>
      <div className="chart-bars">
        {Object.entries(moodCounts).map(([mood, count]) => (
          <div key={mood} className="chart-bar-container">
            <div
              className="chart-bar"
              style={{
                height: `${(count / maxCount) * 100}%`,
                backgroundColor: MOOD_COLORS[mood as MoodEntry['mood']],
              }}
            >
              <span className="chart-count">{count}</span>
            </div>
            <span className="chart-label">{mood}</span>
          </div>
        ))}
      </div>
    </div>
  );
}
```

---

> [!TIP]
> **Session Break** — You've built all three display components: EntryForm, EntryList, and MoodChart. Save your work and take a break.
> When you return, you'll build the App component that wires everything together, add styles, and commit.

---

## Step 6: Build the App Component (Root)

**Where?** `client/src/App.tsx`

**What is this file?** The root component that assembles all other components and manages the application state. It's the "conductor" — it holds the data, coordinates the API calls, and passes everything down to child components.

In VS Code, open the existing file `client/src/App.tsx` (it was created by Vite — we're replacing its contents). Select all the existing code (`Cmd+A` on macOS / `Ctrl+A` on Windows) and replace it with:

```tsx
import { useState, useEffect } from 'react';
import { MoodEntry, CreateEntryRequest } from './types';
import { getEntries, createEntry } from './services/api';
import { EntryForm } from './components/EntryForm';
import { EntryList } from './components/EntryList';
import { MoodChart } from './components/MoodChart';
import './App.css';

function App() {
  // ─── STATE ─────────────────────────────────────────────────
  const [entries, setEntries] = useState<MoodEntry[]>([]);
  const [isLoading, setIsLoading] = useState(true);
  const [error, setError] = useState('');

  // ─── LOAD ENTRIES ON MOUNT ─────────────────────────────────
  // useEffect with [] runs once when the component first appears.
  // This is where we load initial data from the backend.
  useEffect(() => {
    loadEntries();
  }, []);

  async function loadEntries() {
    try {
      setIsLoading(true);
      const data = await getEntries();
      setEntries(data);
    } catch (err) {
      setError('Failed to load entries. Is the backend running?');
      console.error(err);
    } finally {
      setIsLoading(false);
    }
  }

  // ─── HANDLE NEW ENTRY ──────────────────────────────────────
  async function handleCreateEntry(data: CreateEntryRequest) {
    const newEntry = await createEntry(data);
    // Add the new entry to the beginning of the list (newest first)
    setEntries((prev) => [newEntry, ...prev]);
  }

  // ─── RENDER ────────────────────────────────────────────────
  return (
    <div className="app">
      <header className="app-header">
        <h1>DevPulse</h1>
        <p>Track your developer mood</p>
      </header>

      <main className="app-main">
        <EntryForm onSubmit={handleCreateEntry} />

        {isLoading && <p className="loading">Loading entries...</p>}
        {error && <p className="error-message">{error}</p>}

        {!isLoading && !error && (
          <>
            <MoodChart entries={entries} />
            <EntryList entries={entries} />
          </>
        )}
      </main>
    </div>
  );
}

export default App;
```

### React State Flow (Spatial View)

```
┌──────────────────────────────────────────────────────────────┐
│                     App Component                            │
│                                                              │
│  State: entries[], isLoading, error                          │
│                                                              │
│  ┌────────────────────┐  ┌──────────┐  ┌──────────────────┐ │
│  │    EntryForm       │  │MoodChart │  │   EntryList      │ │
│  │                    │  │          │  │                  │ │
│  │ Props:             │  │ Props:   │  │ Props:           │ │
│  │  onSubmit (fn)     │  │ entries  │  │  entries         │ │
│  │                    │  │          │  │                  │ │
│  │ When user submits: │  │ Reads    │  │ Reads entries    │ │
│  │ 1. Calls onSubmit  │  │ entries  │  │ and renders      │ │
│  │ 2. App updates     │  │ and      │  │ each one as      │ │
│  │    state           │  │ draws    │  │ a card           │ │
│  │ 3. All children    │  │ bars     │  │                  │ │
│  │    re-render       │  │          │  │                  │ │
│  └────────────────────┘  └──────────┘  └──────────────────┘ │
│                                                              │
│  Data flows DOWN through props (parent → child)              │
│  Events flow UP through callbacks (child → parent)           │
└──────────────────────────────────────────────────────────────┘
```

### `useEffect` Explained

```typescript
useEffect(() => {
  loadEntries();
}, []);
```

- > [!NOTE]
> **Technical**: `useEffect` is a React Hook that lets you synchronize a component with an external system. The empty dependency array `[]` means it runs once after the initial render.
- > [!NOTE]
> **Plain English**: "When this component appears on screen for the first time, fetch the entries from the backend." The `[]` means "only do this once" — not on every re-render.

### Why State Lives in App (Not in Each Component)

Both `MoodChart` and `EntryList` need the entries data. If each managed its own copy, they could get out of sync. By lifting state to App (their shared parent), there's one source of truth. When a new entry is added, App updates its state, and both children receive the updated list.

This is called **"lifting state up"** — one of React's core patterns.

---

## Step 7: Add Styles

In VS Code, open the existing file `client/src/App.css`. Select all existing content (`Cmd+A` / `Ctrl+A`) and replace it with the following:

```css
/* ─── GLOBAL RESET ───────────────────────────────────────── */
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto,
    Oxygen, Ubuntu, sans-serif;
  background-color: #0f172a;
  color: #e2e8f0;
  line-height: 1.6;
}

/* ─── APP LAYOUT ─────────────────────────────────────────── */
.app {
  max-width: 700px;
  margin: 0 auto;
  padding: 2rem 1rem;
}

.app-header {
  text-align: center;
  margin-bottom: 2rem;
}

.app-header h1 {
  font-size: 2.5rem;
  color: #38bdf8;
  margin-bottom: 0.25rem;
}

.app-header p {
  color: #94a3b8;
  font-size: 1.1rem;
}

/* ─── ENTRY FORM ─────────────────────────────────────────── */
.entry-form {
  background: #1e293b;
  border-radius: 12px;
  padding: 1.5rem;
  margin-bottom: 2rem;
}

.entry-form h2 {
  margin-bottom: 1rem;
  font-size: 1.25rem;
}

.mood-selector {
  display: flex;
  gap: 0.5rem;
  flex-wrap: wrap;
  margin-bottom: 1.25rem;
}

.mood-button {
  padding: 0.5rem 1rem;
  border: 2px solid #334155;
  border-radius: 8px;
  background: transparent;
  color: #e2e8f0;
  cursor: pointer;
  font-size: 0.9rem;
  transition: all 0.2s;
}

.mood-button:hover {
  border-color: #38bdf8;
}

.mood-button.active {
  border-color: #38bdf8;
  background: #38bdf8;
  color: #0f172a;
}

.energy-selector {
  margin-bottom: 1.25rem;
}

.energy-selector label {
  display: block;
  margin-bottom: 0.5rem;
  color: #94a3b8;
}

.energy-selector input[type='range'] {
  width: 100%;
  accent-color: #38bdf8;
}

.note-input {
  margin-bottom: 1.25rem;
}

.note-input label {
  display: block;
  margin-bottom: 0.5rem;
  color: #94a3b8;
}

.note-input textarea {
  width: 100%;
  padding: 0.75rem;
  border: 2px solid #334155;
  border-radius: 8px;
  background: #0f172a;
  color: #e2e8f0;
  font-family: inherit;
  font-size: 1rem;
  resize: vertical;
}

.note-input textarea:focus {
  outline: none;
  border-color: #38bdf8;
}

.char-count {
  display: block;
  text-align: right;
  font-size: 0.8rem;
  color: #64748b;
  margin-top: 0.25rem;
}

.submit-button {
  width: 100%;
  padding: 0.75rem;
  border: none;
  border-radius: 8px;
  background: #38bdf8;
  color: #0f172a;
  font-size: 1rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.2s;
}

.submit-button:hover {
  background: #7dd3fc;
}

.submit-button:disabled {
  background: #334155;
  color: #64748b;
  cursor: not-allowed;
}

/* ─── MOOD CHART ─────────────────────────────────────────── */
.mood-chart {
  background: #1e293b;
  border-radius: 12px;
  padding: 1.5rem;
  margin-bottom: 2rem;
}

.mood-chart h2 {
  margin-bottom: 1rem;
  font-size: 1.25rem;
}

.chart-bars {
  display: flex;
  align-items: flex-end;
  gap: 1rem;
  height: 120px;
}

.chart-bar-container {
  flex: 1;
  display: flex;
  flex-direction: column;
  align-items: center;
  height: 100%;
  justify-content: flex-end;
}

.chart-bar {
  width: 100%;
  border-radius: 6px 6px 0 0;
  display: flex;
  align-items: flex-start;
  justify-content: center;
  padding-top: 0.25rem;
  min-height: 24px;
  transition: height 0.3s ease;
}

.chart-count {
  font-size: 0.8rem;
  font-weight: 600;
  color: #0f172a;
}

.chart-label {
  font-size: 0.75rem;
  color: #94a3b8;
  margin-top: 0.5rem;
  text-transform: capitalize;
}

/* ─── ENTRY LIST ─────────────────────────────────────────── */
.entry-list h2 {
  margin-bottom: 1rem;
  font-size: 1.25rem;
}

.entry-list-empty {
  text-align: center;
  padding: 2rem;
  color: #64748b;
}

.entry-card {
  background: #1e293b;
  border-radius: 12px;
  padding: 1.25rem;
  margin-bottom: 1rem;
}

.entry-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 0.75rem;
}

.entry-mood {
  font-size: 1.1rem;
  text-transform: capitalize;
  font-weight: 500;
}

.entry-energy {
  color: #facc15;
  font-size: 0.9rem;
  letter-spacing: 2px;
}

.entry-note {
  color: #cbd5e1;
  margin-bottom: 0.5rem;
}

.entry-date {
  font-size: 0.8rem;
  color: #64748b;
}

/* ─── UTILITY ────────────────────────────────────────────── */
.loading {
  text-align: center;
  padding: 2rem;
  color: #64748b;
}

.error-message {
  color: #f87171;
  font-size: 0.9rem;
  margin-top: 0.5rem;
}
```

---

## Step 8: Clean Up Boilerplate

Delete files that Vite generated that we don't need. Open your terminal:

> [!IMPORTANT]
> **You should be in:** `dev-pulse/` (the project root)

```bash
rm client/src/assets/react.svg
rm client/public/vite.svg
```

(These paths are relative to `dev-pulse/`. If you're inside `client/`, the paths would be `src/assets/react.svg` and `public/vite.svg` instead.)

Now in VS Code, open `client/index.html` and replace its contents with:

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>DevPulse — Developer Mood Tracker</title>
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.tsx"></script>
  </body>
</html>
```

---

## Step 9: Commit

> [!IMPORTANT]
> **You should be in:** `dev-pulse/` (the project root)

```bash
git add client/
git commit -m "feat: add React frontend with mood entry form and list"
```

---

## Spatial Check-In

The frontend exists but **cannot talk to the backend yet** (we need to run both simultaneously). Let's verify you understand the state flow:

```
User clicks "Log Entry"
        │
        ▼
EntryForm calls onSubmit(data)        ← event flows UP to parent
        │
        ▼
App.handleCreateEntry runs
        │
        ├── Calls createEntry(data)   ← sends HTTP POST to backend
        │
        ▼
Backend processes request, returns new entry
        │
        ▼
App calls setEntries([newEntry, ...prev])  ← state updates
        │
        ▼
React re-renders App and all children
        │
        ├── EntryList receives updated entries → shows new card
        └── MoodChart receives updated entries → updates bars
```

**Data flows DOWN through props. Events flow UP through callbacks. State changes trigger re-renders.**

---

| | | |
|:---|:---:|---:|
| [← 03 — Building the Backend](../03-backend/) | [Level 1 Overview](../) | [05 — Connecting Frontend to Backend →](../05-connecting/) |

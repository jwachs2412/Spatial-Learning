# Step 4 — Building the Frontend

## Spatial Orientation

```
dev-pulse/
├── server/                ← Built in Step 3 (untouched this step)
└── client/                ← YOU ARE HERE
    └── src/
        ├── types/
        │   └── index.ts     ← Shared type definitions
        ├── services/
        │   └── api.ts       ← HTTP calls to backend
        ├── components/
        │   ├── EntryForm.tsx  ← Form for new entries
        │   ├── EntryList.tsx  ← Display all entries
        │   └── MoodChart.tsx  ← Mood visualization
        ├── App.tsx            ← Root component
        ├── App.css            ← Styles
        └── main.tsx           ← Entry point (untouched)
```

You're working in `client/src/`. The backend doesn't know or care what your frontend looks like — it just responds to HTTP requests.

---

## React Concepts You Need

Before writing components, understand these four concepts:

> **Key Concept: Component**
> A component is a reusable piece of UI — a function that returns JSX (HTML-like syntax). Think of it like a LEGO brick: each one has a specific shape and purpose, and you combine them to build something larger.

> **Key Concept: Props**
> Props are inputs to a component — data passed from a parent component to a child. Think of them like function arguments: the parent decides what data to pass, and the child uses it. Props are read-only — the child cannot modify them.

> **Key Concept: useState**
> `useState` is a React hook that lets a component remember data between renders. When state changes, the component re-renders to reflect the new data. Think of it like a component's private notepad — it can write notes to itself and read them back later.

> **Key Concept: useEffect**
> `useEffect` runs code after the component renders. It's used for "side effects" — things that happen outside of rendering, like fetching data from an API, setting up timers, or updating the document title. Think of it like a post-it note that says "after you're done rendering, also do this."

---

## 1. Define Frontend Types

Your frontend needs the same type definitions as the backend to ensure consistency.

### 🏗️ Your Turn

Create the frontend types file. It should match the backend types from Step 3, plus add types for the moods and a type for creating entries.

<details>
<summary>See the solution</summary>

Create `client/src/types/index.ts`:

```typescript
export interface MoodEntry {
  id: number;
  mood: Mood;
  energy: number;
  note: string;
  date: string;
  createdAt: string;
}

export type Mood = 'happy' | 'neutral' | 'sad' | 'frustrated' | 'energized';

export type CreateEntryRequest = Omit<MoodEntry, 'id' | 'createdAt'>;

export const MOODS: Mood[] = ['happy', 'neutral', 'sad', 'frustrated', 'energized'];

export const MOOD_EMOJI: Record<Mood, string> = {
  happy: '😊',
  neutral: '😐',
  sad: '😢',
  frustrated: '😤',
  energized: '⚡',
};
```

</details>

**Why duplicate types between frontend and backend?** In a monorepo, you could share a types package. But keeping them separate for now teaches you that the frontend and backend are independent applications that communicate through a contract (the API). The types on each side describe what that contract looks like.

---

## 2. Build the API Service

> **Key Concept: Service Layer**
> A service layer is a file that handles all communication with an external system (like your API). Instead of scattering `fetch()` calls throughout your components, you centralize them in one place. Benefits: (1) Change the API URL in one place. (2) Add error handling consistently. (3) Components stay focused on UI, not networking.

### 🏗️ Your Turn

Create the API service. It needs two functions:
- `getEntries()` — Fetch all entries from the API
- `createEntry(data)` — Send a new entry to the API

Think about: What URL do you fetch from? What HTTP method and headers do you use? What do you do if the response isn't OK?

<details>
<summary>See the solution</summary>

Create `client/src/services/api.ts`:

```typescript
import { MoodEntry, CreateEntryRequest } from '../types';

const API_URL = import.meta.env.VITE_API_URL || 'http://localhost:3001/api';

export async function getEntries(): Promise<MoodEntry[]> {
  const response = await fetch(`${API_URL}/entries`);

  if (!response.ok) {
    throw new Error(`Failed to fetch entries: ${response.status}`);
  }

  return response.json();
}

export async function createEntry(data: CreateEntryRequest): Promise<MoodEntry> {
  const response = await fetch(`${API_URL}/entries`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
    },
    body: JSON.stringify(data),
  });

  if (!response.ok) {
    const errorData = await response.json().catch(() => ({ errors: ['Request failed'] }));
    throw new Error(errorData.errors?.join(', ') || 'Failed to create entry');
  }

  return response.json();
}
```

</details>

**Line-by-line breakdown:**

| Line | What It Does | Why |
|------|-------------|-----|
| `import.meta.env.VITE_API_URL` | Read API URL from environment variable | Different URLs for development (`localhost:3001`) vs production (your Render URL) |
| `\|\| 'http://localhost:3001/api'` | Fallback for local development | So you don't need a `.env` file just to run locally |
| `if (!response.ok)` | Check if status is 2xx | A 400 or 500 response is NOT an exception — `fetch` only throws on network failures. You must check manually. |
| `'Content-Type': 'application/json'` | Tell the server you're sending JSON | Without this header, the server can't parse your request body |
| `JSON.stringify(data)` | Convert JavaScript object to JSON string | `fetch` doesn't auto-convert objects. You must serialize them. |
| `return response.json()` | Parse the JSON response into a JavaScript object | The response body is a text stream. `.json()` parses it. |

> ⚠️ **Common Mistake: `fetch` doesn't throw on HTTP errors**
> Unlike `axios`, `fetch` does NOT throw an error for 400 or 500 responses. It only throws on network failures (no internet, server unreachable). You must check `response.ok` yourself. Forgetting this means errors silently pass through.

> ⚠️ **Common Mistake: Forgetting `Content-Type` header**
> Without `'Content-Type': 'application/json'`, the server receives the body as plain text and `express.json()` can't parse it. Result: `req.body` is `undefined`.

---

## 3. Build the EntryForm Component

### 🏗️ Your Turn

Build a form component that:
- Has inputs for mood (select/dropdown), energy (1-5 buttons or number input), note (text), and date
- Validates that all fields are filled before submitting
- Calls `onSubmit(data)` when the form is submitted
- Clears the form after successful submission
- Shows a loading state while submitting

Think about: What state does this component need? How do you prevent the default form submission behavior? How do you handle the submit button during loading?

<details>
<summary>See the solution</summary>

Create `client/src/components/EntryForm.tsx`:

```tsx
import { useState, FormEvent } from 'react';
import { CreateEntryRequest, Mood, MOODS, MOOD_EMOJI } from '../types';

interface EntryFormProps {
  onSubmit: (data: CreateEntryRequest) => Promise<void>;
}

export function EntryForm({ onSubmit }: EntryFormProps) {
  const [mood, setMood] = useState<Mood>('neutral');
  const [energy, setEnergy] = useState(3);
  const [note, setNote] = useState('');
  const [date, setDate] = useState(new Date().toISOString().split('T')[0]);
  const [isSubmitting, setIsSubmitting] = useState(false);
  const [error, setError] = useState('');

  const handleSubmit = async (e: FormEvent) => {
    e.preventDefault();
    setError('');
    setIsSubmitting(true);

    try {
      await onSubmit({ mood, energy, note, date });
      // Reset form on success
      setMood('neutral');
      setEnergy(3);
      setNote('');
      setDate(new Date().toISOString().split('T')[0]);
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Something went wrong');
    } finally {
      setIsSubmitting(false);
    }
  };

  return (
    <form onSubmit={handleSubmit} className="entry-form">
      <h2>How are you feeling?</h2>

      {error && <p className="error">{error}</p>}

      <div className="mood-selector">
        {MOODS.map((m) => (
          <button
            key={m}
            type="button"
            className={`mood-btn ${mood === m ? 'active' : ''}`}
            onClick={() => setMood(m)}
          >
            {MOOD_EMOJI[m]} {m}
          </button>
        ))}
      </div>

      <div className="energy-selector">
        <label>Energy Level: {energy}/5</label>
        <input
          type="range"
          min="1"
          max="5"
          value={energy}
          onChange={(e) => setEnergy(Number(e.target.value))}
        />
      </div>

      <div className="form-field">
        <label htmlFor="note">Note</label>
        <textarea
          id="note"
          value={note}
          onChange={(e) => setNote(e.target.value)}
          placeholder="What happened today?"
          rows={3}
        />
      </div>

      <div className="form-field">
        <label htmlFor="date">Date</label>
        <input
          id="date"
          type="date"
          value={date}
          onChange={(e) => setDate(e.target.value)}
        />
      </div>

      <button type="submit" className="submit-btn" disabled={isSubmitting}>
        {isSubmitting ? 'Saving...' : 'Log Entry'}
      </button>
    </form>
  );
}
```

</details>

**Key patterns explained:**

| Pattern | What It Does | Why |
|---------|-------------|-----|
| `e.preventDefault()` | Stops the browser's default form submission (which would reload the page) | React handles submissions, not the browser |
| `try/catch/finally` | Handle success, errors, and cleanup | `finally` runs whether or not there was an error — perfect for resetting `isSubmitting` |
| `disabled={isSubmitting}` | Prevent double-clicks while saving | Without this, users can spam the submit button and create duplicate entries |
| `type="button"` on mood buttons | Prevents these buttons from submitting the form | By default, buttons inside a form have `type="submit"`. Only the actual submit button should submit. |
| `err instanceof Error ? err.message : 'Something went wrong'` | Safely extract the error message | Not all thrown values are Error objects. This handles both cases. |

### 🧠 Debugging Exercise

What would happen if you forgot `e.preventDefault()` in the `handleSubmit` function?

<details>
<summary>Answer</summary>

The browser would handle the form submission natively: it would serialize the form data, make a GET request to the current URL with the data as query parameters, and reload the page. Your `fetch` call might not even complete before the page refreshes. React loses all state, and the user sees the page flash. Always call `e.preventDefault()` in React form handlers.

</details>

---

## 4. Build the EntryList Component

### 🏗️ Your Turn

Build a component that displays a list of mood entries. Each entry should show:
- The mood emoji and name
- The energy level
- The note
- The date

Think about: What if there are no entries? The component should handle that gracefully.

<details>
<summary>See the solution</summary>

Create `client/src/components/EntryList.tsx`:

```tsx
import { MoodEntry, MOOD_EMOJI } from '../types';

interface EntryListProps {
  entries: MoodEntry[];
}

export function EntryList({ entries }: EntryListProps) {
  if (entries.length === 0) {
    return (
      <div className="empty-state">
        <p>No entries yet. Log your first mood above!</p>
      </div>
    );
  }

  return (
    <div className="entry-list">
      <h2>Your Entries</h2>
      {entries.map((entry) => (
        <div key={entry.id} className="entry-card">
          <div className="entry-header">
            <span className="entry-mood">
              {MOOD_EMOJI[entry.mood]} {entry.mood}
            </span>
            <span className="entry-date">{entry.date}</span>
          </div>
          <div className="entry-energy">
            Energy: {'⚡'.repeat(entry.energy)}{'○'.repeat(5 - entry.energy)}
          </div>
          {entry.note && <p className="entry-note">{entry.note}</p>}
        </div>
      ))}
    </div>
  );
}
```

</details>

**Key patterns:**

| Pattern | What It Does | Why |
|---------|-------------|-----|
| `key={entry.id}` | Unique identifier for each list item | React uses keys to efficiently update the DOM. Without unique keys, React can't tell which items changed. |
| `entries.length === 0` check | Show a helpful message when empty | Better UX than showing nothing |
| `{entry.note && <p>...}` | Only render the note paragraph if there is a note | Avoids empty `<p>` tags |
| `'⚡'.repeat(entry.energy)` | Visual energy indicator | More readable than just a number |

---

## 5. Build the MoodChart Component

A simple visualization of mood distribution.

<details>
<summary>See the code</summary>

Create `client/src/components/MoodChart.tsx`:

```tsx
import { MoodEntry, Mood, MOODS, MOOD_EMOJI } from '../types';

interface MoodChartProps {
  entries: MoodEntry[];
}

export function MoodChart({ entries }: MoodChartProps) {
  if (entries.length === 0) return null;

  // Count occurrences of each mood
  const moodCounts = MOODS.reduce(
    (acc, mood) => {
      acc[mood] = entries.filter((e) => e.mood === mood).length;
      return acc;
    },
    {} as Record<Mood, number>,
  );

  const maxCount = Math.max(...Object.values(moodCounts));

  return (
    <div className="mood-chart">
      <h2>Mood Distribution</h2>
      <div className="chart-bars">
        {MOODS.map((mood) => (
          <div key={mood} className="chart-bar-container">
            <div
              className="chart-bar"
              style={{
                height: `${maxCount > 0 ? (moodCounts[mood] / maxCount) * 100 : 0}%`,
              }}
            />
            <span className="chart-label">
              {MOOD_EMOJI[mood]} {moodCounts[mood]}
            </span>
          </div>
        ))}
      </div>
    </div>
  );
}
```

</details>

---

## 6. Build the App Component

> **Key Concept: Lifting State Up**
> When multiple components need the same data, that data should live in their closest common parent. The parent "owns" the state and passes it down via props. This is called "lifting state up." In our app, both `EntryList` and `MoodChart` need the entries array, so it lives in `App`.

### 🏗️ Your Turn

Build the root `App` component. It needs to:
1. Hold the `entries` array in state
2. Fetch entries from the API when the component first loads (`useEffect`)
3. Handle creating new entries (call the API, then add to state)
4. Pass data down to child components

Think about: Where do loading and error states live? How do you add a new entry to the state after creating it on the server?

<details>
<summary>See the solution</summary>

Replace `client/src/App.tsx`:

```tsx
import { useState, useEffect } from 'react';
import { MoodEntry, CreateEntryRequest } from './types';
import { getEntries, createEntry } from './services/api';
import { EntryForm } from './components/EntryForm';
import { EntryList } from './components/EntryList';
import { MoodChart } from './components/MoodChart';
import './App.css';

function App() {
  const [entries, setEntries] = useState<MoodEntry[]>([]);
  const [isLoading, setIsLoading] = useState(true);
  const [error, setError] = useState('');

  // Fetch entries on mount
  useEffect(() => {
    async function loadEntries() {
      try {
        const data = await getEntries();
        setEntries(data);
      } catch (err) {
        setError(err instanceof Error ? err.message : 'Failed to load entries');
      } finally {
        setIsLoading(false);
      }
    }

    loadEntries();
  }, []);

  // Handle new entry
  const handleSubmit = async (data: CreateEntryRequest): Promise<void> => {
    const newEntry = await createEntry(data);
    setEntries((prev) => [newEntry, ...prev]);
  };

  if (isLoading) {
    return <div className="loading">Loading...</div>;
  }

  return (
    <div className="app">
      <header>
        <h1>DevPulse</h1>
        <p>Track your developer mood</p>
      </header>

      {error && <p className="error">{error}</p>}

      <EntryForm onSubmit={handleSubmit} />
      <MoodChart entries={entries} />
      <EntryList entries={entries} />
    </div>
  );
}

export default App;
```

</details>

**Important patterns:**

| Pattern | What It Does | Why |
|---------|-------------|-----|
| `useEffect(() => { ... }, [])` | Runs once when the component mounts | The empty `[]` dependency array means "run this once, not on every render" |
| `setEntries((prev) => [newEntry, ...prev])` | Add new entry to the front of the array | Uses the callback form of setState to avoid stale state. New entries appear at the top. |
| Async function inside useEffect | Handles the async API call | `useEffect` callback can't be async itself, so we define an async function inside and call it immediately |

> ⚠️ **Common Mistake: Async useEffect**
> You might try: `useEffect(async () => { ... })`. This doesn't work — React expects useEffect to return either nothing or a cleanup function, not a Promise. Always define the async function inside the effect.

### 🧠 Debugging Exercise

What would happen if you wrote `setEntries([newEntry, ...entries])` instead of `setEntries((prev) => [newEntry, ...prev])`?

<details>
<summary>Answer</summary>

It would work most of the time, but could cause a subtle bug. If `handleSubmit` is called twice quickly (before the first render completes), both calls would read the same `entries` value from the closure. The second call would overwrite the first entry. The callback form `(prev) => [newEntry, ...prev]` always gets the latest state, avoiding this race condition.

</details>

---

## 7. Add Styles

Replace `client/src/App.css` with styling for the app. Here's a dark-theme design:

<details>
<summary>See the full CSS</summary>

```css
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
  background-color: #0f172a;
  color: #e2e8f0;
  line-height: 1.6;
}

.app {
  max-width: 700px;
  margin: 0 auto;
  padding: 2rem 1rem;
}

header {
  text-align: center;
  margin-bottom: 2rem;
}

header h1 {
  font-size: 2rem;
  color: #38bdf8;
}

header p {
  color: #94a3b8;
}

/* Form */
.entry-form {
  background: #1e293b;
  border-radius: 12px;
  padding: 1.5rem;
  margin-bottom: 2rem;
}

.entry-form h2 {
  margin-bottom: 1rem;
  font-size: 1.2rem;
}

.mood-selector {
  display: flex;
  gap: 0.5rem;
  flex-wrap: wrap;
  margin-bottom: 1rem;
}

.mood-btn {
  padding: 0.5rem 1rem;
  border: 2px solid #334155;
  border-radius: 8px;
  background: transparent;
  color: #e2e8f0;
  cursor: pointer;
  font-size: 0.9rem;
  transition: all 0.2s;
}

.mood-btn:hover {
  border-color: #38bdf8;
}

.mood-btn.active {
  border-color: #38bdf8;
  background: #0c4a6e;
}

.energy-selector {
  margin-bottom: 1rem;
}

.energy-selector label {
  display: block;
  margin-bottom: 0.5rem;
  color: #94a3b8;
}

.energy-selector input[type="range"] {
  width: 100%;
  accent-color: #38bdf8;
}

.form-field {
  margin-bottom: 1rem;
}

.form-field label {
  display: block;
  margin-bottom: 0.3rem;
  color: #94a3b8;
  font-size: 0.9rem;
}

.form-field textarea,
.form-field input {
  width: 100%;
  padding: 0.6rem;
  border: 1px solid #334155;
  border-radius: 6px;
  background: #0f172a;
  color: #e2e8f0;
  font-family: inherit;
  font-size: 0.95rem;
}

.form-field textarea:focus,
.form-field input:focus {
  outline: none;
  border-color: #38bdf8;
}

.submit-btn {
  width: 100%;
  padding: 0.75rem;
  background: #0ea5e9;
  color: white;
  border: none;
  border-radius: 8px;
  font-size: 1rem;
  cursor: pointer;
  transition: background 0.2s;
}

.submit-btn:hover:not(:disabled) {
  background: #0284c7;
}

.submit-btn:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

/* Entry List */
.entry-list {
  margin-top: 2rem;
}

.entry-list h2 {
  margin-bottom: 1rem;
}

.entry-card {
  background: #1e293b;
  border-radius: 8px;
  padding: 1rem;
  margin-bottom: 0.75rem;
}

.entry-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 0.5rem;
}

.entry-mood {
  font-size: 1.1rem;
  text-transform: capitalize;
}

.entry-date {
  color: #64748b;
  font-size: 0.85rem;
}

.entry-energy {
  color: #94a3b8;
  font-size: 0.9rem;
  margin-bottom: 0.5rem;
}

.entry-note {
  color: #cbd5e1;
  font-size: 0.95rem;
}

/* Chart */
.mood-chart {
  background: #1e293b;
  border-radius: 12px;
  padding: 1.5rem;
  margin-bottom: 2rem;
}

.mood-chart h2 {
  margin-bottom: 1rem;
  font-size: 1.2rem;
}

.chart-bars {
  display: flex;
  justify-content: space-around;
  align-items: flex-end;
  height: 120px;
  padding-top: 1rem;
}

.chart-bar-container {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 0.5rem;
  flex: 1;
}

.chart-bar {
  width: 40px;
  background: #0ea5e9;
  border-radius: 4px 4px 0 0;
  min-height: 4px;
  transition: height 0.3s ease;
}

.chart-label {
  font-size: 0.8rem;
  color: #94a3b8;
  text-align: center;
}

/* States */
.loading {
  text-align: center;
  padding: 4rem;
  color: #64748b;
}

.error {
  background: #450a0a;
  color: #fca5a5;
  padding: 0.75rem 1rem;
  border-radius: 8px;
  margin-bottom: 1rem;
}

.empty-state {
  text-align: center;
  padding: 2rem;
  color: #64748b;
}
```

</details>

---

## 8. Clean Up index.html

Update `client/index.html` — change the `<title>`:

```html
<title>DevPulse — Developer Mood Tracker</title>
```

Also delete the default Vite/React boilerplate CSS files if they exist (`client/src/index.css`) and remove any import of `index.css` from `main.tsx`.

---

## 9. Commit

```bash
git add .
git commit -m "feat: add React frontend with EntryForm, EntryList, and MoodChart"
```

### ✅ Checkpoint

Run the frontend (`cd client && npm run dev`). You should see:
- [ ] The DevPulse header
- [ ] A mood selection form with emoji buttons
- [ ] An energy slider
- [ ] A note text area and date picker
- [ ] A "Log Entry" submit button

The form won't actually save data yet — that happens when we connect the frontend to the backend in Step 5. But the UI should render without errors.

If you open the browser console (F12 → Console) and see errors:
- `Module not found`: Check your import paths
- `X is not a function`: Check that you're exporting and importing correctly
- `Cannot find module './types'`: Make sure you created the types file at the right path

---

## 🧠 Spatial Check-In

1. In our App component, why does `entries` state live in `App` and not in `EntryList`? What if we also wanted a `MoodChart` component to show mood statistics?

2. What is the purpose of the `[]` (empty array) at the end of `useEffect(() => { ... }, [])`? What would happen if you removed it?

3. Why do we define an async function *inside* useEffect instead of making the useEffect callback itself async?

<details>
<summary>Check Your Answers</summary>

1. **Lifting state up.** Both `EntryList` and `MoodChart` need the entries data. If entries lived in `EntryList`, `MoodChart` couldn't access it. By keeping entries in their common parent (`App`), we can pass the data to both children via props.

2. **The empty array is the dependency list.** It means "only run this effect once, when the component first mounts." Without it, the effect runs after **every** render, causing an infinite loop: fetch data → update state → re-render → fetch data → update state → ...

3. **useEffect expects a cleanup function or nothing as its return value.** An `async` function always returns a Promise. If useEffect returned a Promise, React would try to call it as a cleanup function, which wouldn't work. So we define the async function inside and call it immediately.

</details>

---

> **Session Break** — You've built the frontend UI.
> When you return, you'll connect the frontend to the backend in [Step 5 — Connecting](../05-connecting/).

---

| | |
|:---|---:|
| [← Step 3: Backend](../03-backend/) | [Step 5 — Connecting →](../05-connecting/) |

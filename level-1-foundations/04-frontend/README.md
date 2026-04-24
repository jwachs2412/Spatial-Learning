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

## Plain-English Primer: JSX, Hooks, and Events

You met the JavaScript/TypeScript basics in Step 3 (variables, functions, imports, destructuring). React adds a few extra shapes that appear in every component in this lesson.

### JSX — HTML inside JavaScript

```tsx
const element = <h1>Hello, {name}</h1>;
```

- JSX is HTML-like syntax you can write directly inside a JavaScript/TypeScript file. It looks like HTML but it's actually compiled into function calls that build DOM nodes.
- `.tsx` (instead of `.ts`) is the file extension that allows JSX syntax. Think "TypeScript eXtended with JSX."
- Anything inside `{ }` inside JSX is a **JavaScript expression** — the value is injected into the output. `<h1>Hello, {name}</h1>` embeds the value of the variable `name`.
- HTML attributes you already know work the same way: `<input value={text} />`, `<button disabled={isLoading}>`. When the value is a string literal you can use quotes (`type="text"`); when it's a variable you use `{ }`.
- Two gotchas: in JSX you write `className` instead of `class` (because `class` is a reserved word in JavaScript), and `htmlFor` instead of `for`.

### Components — Functions That Return JSX

```tsx
function Greeting({ name }) {
  return <h1>Hello, {name}</h1>;
}
```

- A **component** is just a function whose name starts with a capital letter (React uses the capitalization to tell your components apart from HTML tags).
- It takes one argument — an object of **props** — and returns JSX.
- You call it by writing it like a tag: `<Greeting name="Jo" />`. React handles the translation.
- `{ name }` in the parameter list is destructuring — pulling `name` out of the props object.

### Hooks — Functions That Start With `use`

Hooks are special functions React provides that let you add features to a component. Two you'll meet in this lesson:

- **`useState(initialValue)`** — returns a pair: `[value, setValue]`. The value is the current state; `setValue` is a function that updates it and triggers a re-render.
- **`useEffect(fn, deps)`** — runs `fn` after the component renders. `deps` is an array of values the effect depends on; if you pass `[]`, the effect runs only once on first render.

```tsx
const [count, setCount] = useState(0);   // reads: count = 0, setCount is the updater
setCount(count + 1);                     // triggers a re-render with count = 1
```

The `[value, setValue]` on the left is **array destructuring** — a compact way to name the two items `useState` returns.

### Events — Responding to User Input

```tsx
<button onClick={() => setCount(count + 1)}>Click me</button>
<input onChange={(e) => setText(e.target.value)} />
<form onSubmit={(e) => { e.preventDefault(); /* ... */ }}>
```

- Event handler props use camelCase (`onClick`, `onChange`, `onSubmit`). They take a function — usually an arrow function.
- `e` is the **event object** React passes to your handler. Common properties: `e.target` (the element that fired the event), `e.target.value` (the value of an input), `e.preventDefault()` (stop the browser's default behavior like form submission reloading the page).

That's enough to read every React snippet in this lesson.

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

### Reading This File Line-by-Line

This file exports four things: an interface, two type aliases, and two constants. All are reused across the frontend components.

- `export interface MoodEntry { ... }` — same shape as the backend's `MoodEntry` but uses the `Mood` alias for the `mood` field. Defining `Mood` once and reusing it keeps the two definitions in sync.
- `export type Mood = 'happy' | ...;` — a **type alias** for the union of allowed mood values. Wherever you see `Mood` used as a type, it means "one of those five exact strings."
- `export type CreateEntryRequest = Omit<MoodEntry, 'id' | 'createdAt'>;` — same pattern as the backend: "MoodEntry minus the server-generated fields."
- `export const MOODS: Mood[] = [ ... ];` — a **constant array** of all valid moods. The type annotation `Mood[]` says every item must be one of the five strings. We'll loop over this in the UI to render one button per mood.
- `export const MOOD_EMOJI: Record<Mood, string> = { ... };`
  - `Record<K, V>` is a built-in TypeScript utility type that reads as "an object with keys of type K and values of type V."
  - `Record<Mood, string>` means "an object that has **every** `Mood` value as a key, each mapped to a string." TypeScript will error if you miss one of the five keys or add an unknown one.
  - This is how we attach emoji metadata to each mood without storing it inside each entry.

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

### Reading This File Line-by-Line

```typescript
import { MoodEntry, CreateEntryRequest } from '../types';
```

Named imports of two types from our types file. `../` means "up one folder." We're in `services/`, so `../types` resolves to `../types/index.ts`.

```typescript
const API_URL = import.meta.env.VITE_API_URL || 'http://localhost:3001/api';
```

- `import.meta.env` is Vite's way of exposing environment variables to the frontend at build time. (`process.env` is the Node/backend equivalent.)
- `VITE_API_URL` — Vite only exposes variables whose names start with `VITE_`. This prevents accidentally shipping backend secrets in the frontend bundle.
- `|| 'http://localhost:3001/api'` — fallback for local development when the env var isn't set.

```typescript
export async function getEntries(): Promise<MoodEntry[]> {
```

- `export async function` — an exported async function. The `async` keyword means the function uses `await` inside and returns a promise.
- `Promise<MoodEntry[]>` — the return type. Resolves to an array of `MoodEntry` objects.

```typescript
const response = await fetch(`${API_URL}/entries`);
```

- `fetch(url)` is the built-in browser function for making HTTP requests. It returns a promise that resolves to a `Response` object.
- `await` waits for the promise to resolve.
- `` `${API_URL}/entries` `` — template literal: in development, `API_URL` is `http://localhost:3001/api`, so this becomes `http://localhost:3001/api/entries`.

```typescript
if (!response.ok) {
  throw new Error(`Failed to fetch entries: ${response.status}`);
}
```

- `response.ok` is `true` if the status code is in the 200–299 range.
- **Critical:** `fetch` does NOT throw on HTTP errors (400, 500, etc.). It only throws on network failures (no internet, DNS failure, etc.). You must check `response.ok` yourself.
- `throw new Error(...)` manually throws an error. `throw` creates an "exception" that bubbles up until something `catch`es it. The `Error` class wraps a human-readable message.

```typescript
return response.json();
```

- `response.json()` returns a promise that resolves to the parsed JSON body. We return that promise directly; the caller's `await` will wait for it.

```typescript
export async function createEntry(data: CreateEntryRequest): Promise<MoodEntry> {
  const response = await fetch(`${API_URL}/entries`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
    },
    body: JSON.stringify(data),
  });
```

- This time `fetch` takes two arguments: the URL and an **options object** that customizes the request.
  - `method: 'POST'` — the HTTP method. Default is `GET`.
  - `headers: { 'Content-Type': 'application/json' }` — tells the server the body is JSON. Without this, Express's `express.json()` middleware can't parse it and `req.body` would be `undefined`.
  - `body: JSON.stringify(data)` — `fetch` requires the body to be a string. `JSON.stringify(obj)` converts a JavaScript object into a JSON-formatted string. Without `stringify`, `fetch` would send `"[object Object]"` as text.

```typescript
if (!response.ok) {
  const errorData = await response.json().catch(() => ({ errors: ['Request failed'] }));
  throw new Error(errorData.errors?.join(', ') || 'Failed to create entry');
}
```

- If the request failed, read the error body and throw.
- `response.json().catch(() => ({ errors: ['Request failed'] }))` — `.catch(handler)` is the method form of try/catch for promises. If the body can't be parsed as JSON (e.g., empty response), fall back to a default error shape.
- `errorData.errors?.join(', ')` — `?.` is the **optional chaining operator**. If `errorData.errors` is `undefined` or `null`, the whole expression evaluates to `undefined` instead of throwing. Then `||` falls back to the generic message.

```typescript
return response.json();
}
```

Success path — parse and return the JSON body as a `MoodEntry`.

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

### Reading This File Line-by-Line

This is the densest component in Level 1. We'll work through it in four chunks: **setup**, **state**, **handler**, and **JSX**.

#### Chunk 1: Imports and Props

```tsx
import { useState, FormEvent } from 'react';
import { CreateEntryRequest, Mood, MOODS, MOOD_EMOJI } from '../types';

interface EntryFormProps {
  onSubmit: (data: CreateEntryRequest) => Promise<void>;
}
```

- `useState` is the React hook; `FormEvent` is a type describing form submission events.
- `interface EntryFormProps { ... }` — defines the shape of props this component accepts. By convention: `<ComponentName>Props`.
- `onSubmit: (data: CreateEntryRequest) => Promise<void>` — a prop typed as a function. Read as "a function that takes a `CreateEntryRequest` and returns a `Promise<void>` (a promise that resolves to nothing)." The parent will pass a real async function here.

#### Chunk 2: Component Signature and State

```tsx
export function EntryForm({ onSubmit }: EntryFormProps) {
  const [mood, setMood] = useState<Mood>('neutral');
  const [energy, setEnergy] = useState(3);
  const [note, setNote] = useState('');
  const [date, setDate] = useState(new Date().toISOString().split('T')[0]);
  const [isSubmitting, setIsSubmitting] = useState(false);
  const [error, setError] = useState('');
```

- `function EntryForm({ onSubmit }: EntryFormProps)` — component name starts with capital, props are destructured (pulling `onSubmit` out), and the props object is typed with our interface.
- `useState<Mood>('neutral')` — `useState` is a function that returns a tuple (pair) of `[currentValue, setter]`. The `<Mood>` in angle brackets is a **type argument**, telling TypeScript "this state holds a `Mood` value." We destructure into `[mood, setMood]` — `mood` is the current value, `setMood` is the function that changes it.
- `useState(3)` — when you pass a plain initial value, TypeScript infers the type automatically. `energy` is inferred as `number`.
- `new Date().toISOString().split('T')[0]`:
  - `new Date()` — current date/time.
  - `.toISOString()` — produces a string like `"2025-01-15T14:32:01.000Z"`.
  - `.split('T')` — splits that string at the `"T"`, producing `["2025-01-15", "14:32:01.000Z"]`.
  - `[0]` — takes the first part, just `"2025-01-15"`. That's what HTML's `<input type="date">` expects.
- `useState(false)` — boolean, `false` initially. Used to disable the button while submitting.
- `useState('')` — empty string for the error message.

#### Chunk 3: The Submit Handler

```tsx
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
```

- `const handleSubmit = async (e: FormEvent) => { ... }` — an async arrow function typed with the `FormEvent` parameter. Defined inside the component so it has access to the state setters via closure.
- `e.preventDefault()` — prevents the browser's default form submission (which would reload the page with form data in the URL). React handles submission via our `onSubmit` prop.
- `setError('')`, `setIsSubmitting(true)` — reset any previous error, mark as submitting (disables the submit button in the UI).
- `try { ... } catch (err) { ... } finally { ... }`:
  - `try` — code that might throw.
  - `catch (err)` — runs if anything inside `try` throws. `err` is the thrown value.
  - `finally` — runs after either `try` or `catch`, no matter what. Perfect for cleanup.
- `await onSubmit({ mood, energy, note, date })` — call the parent's submit handler. Curly braces here are an **object literal** — shorthand form, equivalent to `{ mood: mood, energy: energy, note: note, date: date }`.
- Reset the form fields on success.
- `err instanceof Error ? err.message : 'Something went wrong'`
  - `err instanceof Error` — checks whether the thrown value is actually an `Error` object (not everything thrown has to be).
  - Ternary: if it's an Error, use its message; otherwise fall back to a generic.

#### Chunk 4: The JSX

```tsx
return (
  <form onSubmit={handleSubmit} className="entry-form">
    <h2>How are you feeling?</h2>

    {error && <p className="error">{error}</p>}
```

- `<form onSubmit={handleSubmit}>` — when the form submits, call `handleSubmit`. React wires up the DOM event for us.
- `className="entry-form"` — JSX's version of HTML's `class`. Named differently because `class` is a reserved keyword in JavaScript.
- `{error && <p>...</p>}` — **conditional rendering trick**. `&&` short-circuits: if `error` is a truthy value (a non-empty string), the right side evaluates and renders. If `error` is empty (falsy), the whole expression is `''` and React renders nothing.

```tsx
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
```

- `MOODS.map((m) => (...))` — `.map` transforms each item in the array into something else. Here it returns one `<button>` per mood. The parentheses around `(...)` wrap the JSX we're returning from the arrow function.
- `key={m}` — when rendering a list in React, each item must have a unique `key`. React uses this to efficiently update the DOM when items change. Here, each mood string is unique, so it works as the key.
- `type="button"` — by default, `<button>` elements inside a `<form>` submit the form. We want these mood buttons to NOT submit, so we override their type.
- `className={\`mood-btn ${mood === m ? 'active' : ''}\`}` — a template literal building a dynamic class string. If this button's mood matches the selected mood, add the `active` class.
- `onClick={() => setMood(m)}` — arrow function that, when clicked, updates state. `m` is captured from the `.map` iteration.

```tsx
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
```

- `<input type="range">` — HTML's slider input.
- `value={energy}` — **controlled input**. React controls the input's value; the state is the source of truth. This is the React way: every keystroke/move triggers `onChange`, updates state, re-renders with the new value.
- `onChange={(e) => setEnergy(Number(e.target.value))}` — when the slider moves, `e.target.value` is the new value as a string (all HTML input values are strings). `Number(...)` converts it to a number before storing in state.

```tsx
    <button type="submit" className="submit-btn" disabled={isSubmitting}>
      {isSubmitting ? 'Saving...' : 'Log Entry'}
    </button>
  </form>
);
}
```

- `type="submit"` — this is the only button that actually submits the form.
- `disabled={isSubmitting}` — React sets the HTML `disabled` attribute when `isSubmitting` is true, preventing double-clicks.
- `{isSubmitting ? 'Saving...' : 'Log Entry'}` — ternary inside JSX, swapping the button text based on state.

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

### Reading This File Line-by-Line

```tsx
import { MoodEntry, MOOD_EMOJI } from '../types';

interface EntryListProps {
  entries: MoodEntry[];
}
```

- Named imports again. `MOOD_EMOJI` is the object mapping moods to emoji.
- Props interface: `entries` is an array of `MoodEntry`.

```tsx
export function EntryList({ entries }: EntryListProps) {
  if (entries.length === 0) {
    return (
      <div className="empty-state">
        <p>No entries yet. Log your first mood above!</p>
      </div>
    );
  }
```

- `if (entries.length === 0)` — **early return** for the empty case. Showing a helpful message beats showing a blank space.
- A component can have multiple `return` statements; React uses whichever one the control flow reaches.

```tsx
  return (
    <div className="entry-list">
      <h2>Your Entries</h2>
      {entries.map((entry) => (
        <div key={entry.id} className="entry-card">
```

- `entries.map((entry) => (...))` — one card per entry. Same `.map` pattern as EntryForm.
- `key={entry.id}` — the entry's database-style ID is perfect as a key because it's guaranteed unique.

```tsx
          <div className="entry-header">
            <span className="entry-mood">
              {MOOD_EMOJI[entry.mood]} {entry.mood}
            </span>
            <span className="entry-date">{entry.date}</span>
          </div>
```

- `MOOD_EMOJI[entry.mood]` — look up the emoji using the mood as a key. If `entry.mood` is `'happy'`, this evaluates to `'😊'`.

```tsx
          <div className="entry-energy">
            Energy: {'⚡'.repeat(entry.energy)}{'○'.repeat(5 - entry.energy)}
          </div>
```

- `'⚡'.repeat(entry.energy)` — every string has a `.repeat(n)` method that returns the string repeated `n` times. `'⚡'.repeat(3)` is `'⚡⚡⚡'`.
- `'○'.repeat(5 - entry.energy)` — fills the rest with empty circles so every row is 5 characters wide.
- Together: `⚡⚡⚡○○` for energy 3, `⚡⚡⚡⚡⚡` for energy 5.

```tsx
          {entry.note && <p className="entry-note">{entry.note}</p>}
        </div>
      ))}
    </div>
  );
}
```

- `{entry.note && <p>...</p>}` — conditional rendering again. If the note is empty (falsy), skip the paragraph entirely; otherwise render it.

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

### Reading This File Line-by-Line

This component introduces `reduce`, `filter`, and inline styles — three patterns worth unpacking.

```tsx
if (entries.length === 0) return null;
```

- Early return. `return null` is React's way of saying "render nothing." The component has no output on an empty list.

```tsx
const moodCounts = MOODS.reduce(
  (acc, mood) => {
    acc[mood] = entries.filter((e) => e.mood === mood).length;
    return acc;
  },
  {} as Record<Mood, number>,
);
```

`reduce` is an array method that **boils an array down into a single value**. Unlike `.map` (which returns a new array of the same length) or `.filter` (which returns a smaller array), `.reduce` can produce anything — a number, an object, another array.

- Signature: `array.reduce(callback, initialValue)`.
- `callback` gets two arguments: `acc` (the "accumulator" — the running total) and each array item in turn.
- The callback returns the new accumulator for the next iteration.

Let's trace through:

- Initial value: `{} as Record<Mood, number>` — an empty object, told by TypeScript to treat it as "an object with Mood keys and number values." We need the cast because an empty object doesn't yet have all the required keys.
- First iteration: `mood = 'happy'`. We do `acc['happy'] = entries.filter(...).length`, then return the updated `acc`.
  - `entries.filter((e) => e.mood === 'happy')` — returns a new array containing only entries whose mood equals `'happy'`.
  - `.length` — how many matched.
- Second iteration: `mood = 'neutral'`. We fill in `acc['neutral']`. And so on.
- Final result: `{ happy: 3, neutral: 2, sad: 0, frustrated: 1, energized: 4 }` — counts for every mood.

```tsx
const maxCount = Math.max(...Object.values(moodCounts));
```

- `Object.values(obj)` — built-in method that returns an array of the object's values, ignoring the keys. Here: `[3, 2, 0, 1, 4]`.
- `Math.max(...)` — built-in that returns the biggest of its arguments. But `Math.max` takes individual arguments, not an array — so we use the **spread operator** (`...`) to unpack the array into separate arguments.
- `Math.max(3, 2, 0, 1, 4)` → `4`.

```tsx
<div
  className="chart-bar"
  style={{
    height: `${maxCount > 0 ? (moodCounts[mood] / maxCount) * 100 : 0}%`,
  }}
/>
```

- `style={{ ... }}` — **inline styles** in JSX. The **outer** `{ }` are JSX saying "expression coming"; the **inner** `{ }` are an object literal. So `style={{ height: '50%' }}` is actually passing one object with a `height` field.
- CSS properties use camelCase in JSX (`backgroundColor`, `fontSize`) instead of kebab-case (`background-color`, `font-size`).
- The height calculation: if `maxCount > 0`, height = this mood's proportion of the max × 100%. Otherwise 0%. Ensures bars scale relative to the tallest.

`<div ... />` — a **self-closing tag** for elements with no children. Same rule as HTML but required in JSX (HTML is more lenient).

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

### Reading This File Line-by-Line

```tsx
import { useState, useEffect } from 'react';
import { MoodEntry, CreateEntryRequest } from './types';
import { getEntries, createEntry } from './services/api';
import { EntryForm } from './components/EntryForm';
import { EntryList } from './components/EntryList';
import { MoodChart } from './components/MoodChart';
import './App.css';
```

- Two hooks, two types, two API functions, three components, and a CSS file.
- `import './App.css';` — an import with **no name**. This is a **side-effect-only** import: Vite sees it and bundles the CSS so it's applied to the page. No variable needed.

```tsx
function App() {
  const [entries, setEntries] = useState<MoodEntry[]>([]);
  const [isLoading, setIsLoading] = useState(true);
  const [error, setError] = useState('');
```

Three state variables: the list of entries, a loading flag (starts `true` because we need to fetch data before showing the UI), and an error message.

- `useState<MoodEntry[]>([])` — tell TypeScript the state is an array of `MoodEntry`. Initial value is an empty array.

```tsx
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
```

- `useEffect(fn, deps)` — run `fn` after the component renders. `deps` is the dependency array: the effect re-runs whenever any value in it changes.
- `[]` (empty deps) — the effect runs **exactly once**, right after the first render. Perfect for initial data loading.
- We can't write `useEffect(async () => {...}, [])` directly because React expects the effect callback to return either nothing or a cleanup function, not a promise. The workaround: **define an async function inside the effect, then call it**.
- Inside `loadEntries`:
  - `try` — attempt to fetch.
  - `catch (err)` — if anything throws, extract the message and store it.
  - `finally` — either way, turn off the loading flag.

```tsx
  const handleSubmit = async (data: CreateEntryRequest): Promise<void> => {
    const newEntry = await createEntry(data);
    setEntries((prev) => [newEntry, ...prev]);
  };
```

- The function we'll pass down to `EntryForm` as the `onSubmit` prop.
- `setEntries((prev) => [newEntry, ...prev])`:
  - **Functional updater form** of `setState`. Instead of passing the new value, you pass a function that takes the previous value and returns the new one.
  - `[newEntry, ...prev]` — a new array with `newEntry` first, followed by every item from `prev` (spread operator). This puts the new entry at the top of the list.
  - Why the functional form? If multiple updates happen quickly, React guarantees each one sees the latest state. Writing `setEntries([newEntry, ...entries])` could use a stale `entries` from an earlier render.

```tsx
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

- Early return during loading — same technique as in EntryList.
- Child components are passed the data they need via props: `onSubmit={handleSubmit}` for the form; `entries={entries}` for the chart and list.
- `export default App;` — default export. `main.tsx` imports this as `import App from './App'`.

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

### How to Read This CSS (If You've Never Written CSS)

You don't need to memorize the values — just understand the shape. Every CSS rule has three parts:

```css
.entry-form {          /* selector: which element(s) this targets */
  background: #1e293b; /* property: value — one per line */
  padding: 1.5rem;     /* more properties… */
}
```

- **Selector** — the text before the `{`. `.entry-form` (with a dot) selects every element whose `className` includes `entry-form`. `#id` selects by id; `body` selects by tag name.
- **Property: value** — the actual style. Ends with a semicolon.
- **Units**: `rem` is relative to the document's base font size (usually 16px, so `1rem` = 16px). `px` is exact pixels. `%` is relative to the parent.
- **Colors**: `#0f172a` is hex (six hex digits: two for red, two for green, two for blue). `#38bdf8` is a bright sky blue.
- **Nested selectors**: `.form-field input` means "any `<input>` inside an element with class `form-field`."
- **Pseudo-states**: `.submit-btn:hover` means "when the user's mouse is over a `.submit-btn`."

If you've never written CSS before, the important idea is: your JSX assigns class names (`className="entry-form"`), and this file attaches visual rules to those names. Change a number here and you'll see the page change.


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

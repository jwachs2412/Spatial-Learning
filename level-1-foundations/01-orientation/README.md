# Step 1 — Orientation: Understanding the Web

## Spatial Orientation — Where Are We?

```
┌─────────────────────────────────────────────────────────────┐
│                       THE WEB                                │
│                                                              │
│   YOU ARE HERE                                               │
│   ▼                                                          │
│   Understanding how the browser and server communicate       │
│   before writing a single line of code                       │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

Before you write any code, you need to understand **where** code runs and **how** the pieces communicate. Most bugs in web development come from confusion about these fundamentals — not from syntax errors.

---

## Words We'll Use a Lot — Define Them Once, Use Them Forever

Before diving in, here's plain-English shorthand for the vocabulary that shows up in every lesson. Read this once — you'll meet every one of these again.

- **Code** — instructions written in a programming language. A text file full of words and symbols that a computer reads and acts on.
- **Program / App** — a collection of code files that, when run, does something useful (a website, a calculator, a game).
- **Run** — telling the computer to start executing your code. You "run" the backend, you "run" a script.
- **Browser** — the program you're using to read this (Chrome, Firefox, Safari). It fetches and displays web pages.
- **Server** — a program that listens for requests over the network and sends responses. Can also refer to the physical computer that program runs on.
- **Backend** — code that runs on the server. The part users never see.
- **Frontend** — code that runs in the browser. What users see and interact with.
- **localhost** — a nickname your computer uses for itself. When you type `localhost:3001` in the browser, you're saying "talk to a program running on this very machine at port 3001." Not connected to the internet in any way.
- **Port** — a numbered "entrance" on your computer. Different programs listen on different ports so they don't collide (more on this below).
- **Endpoint** — a specific URL on a server that does a specific thing. `/api/entries` is one endpoint; `/api/health` is another. "URL" and "endpoint" are often used interchangeably for backend routes.
- **URL** — the address of something on the web, like `http://localhost:3001/api/entries`. Made up of a protocol (`http`), a host (`localhost`), an optional port (`3001`), and a path (`/api/entries`).
- **API (Application Programming Interface)** — a set of endpoints a program offers to other programs. Your backend exposes an API so the frontend can talk to it.
- **Request** — a message the client sends to the server ("please give me the entries" or "please save this new entry").
- **Response** — what the server sends back ("here are the entries" or "saved, here's the new one").
- **Data** — any information your app works with: a user's name, a list of entries, a score, a photo. On the wire, it's text; in code, it's organized into structures.

You don't have to memorize these. They'll show up so often that repetition will do the memorization for you.

---

## The Client-Server Model

> **Key Concept: Client-Server Architecture**
> A *client* is any program that requests information. A *server* is any program that responds to those requests. In web development, the client is the browser and the server is your backend application. Think of it like a restaurant: the customer (client) places an order, and the kitchen (server) prepares and delivers the food.

```
┌───────────────────┐           ┌───────────────────┐
│   CLIENT          │  Request  │   SERVER           │
│   (Browser)       │──────────▶│   (Your backend)   │
│                   │           │                    │
│   - Displays UI   │  Response │   - Processes data │
│   - Sends input   │◀──────────│   - Stores data    │
│   - Runs React    │           │   - Runs Express   │
└───────────────────┘           └───────────────────┘
```

### Why This Split Exists

You might wonder: why not just do everything in the browser? Three reasons:

1. **Security** — The browser is controlled by the user. They can open DevTools and change anything. You can't trust it with sensitive operations.
2. **Shared state** — If two users visit your app, they each have their own browser. A server is the single source of truth that both browsers talk to.
3. **Processing power** — Some operations (database queries, file processing, third-party API calls) need a reliable environment, not a browser tab that might close.

### 🧠 Think About It

If your app stored data only in the browser (using localStorage), what would happen when:
- The user clears their browser data?
- The user opens your app on a different device?
- Two users need to see the same data?

<details>
<summary>Answer</summary>

All three scenarios fail. Browser storage is local to that browser, on that device. A server solves all three problems: data persists regardless of browser state, is accessible from any device, and is shared between all users.

</details>

---

## HTTP Requests — How Client and Server Talk

> **Key Concept: HTTP (HyperText Transfer Protocol)**
> HTTP is the language that browsers and servers speak. Every time you load a webpage, click a link, or submit a form, your browser sends an HTTP request. The server processes it and sends back an HTTP response. Think of it like sending a letter: the request is the letter you send (with a subject, address, and content), and the response is the letter you get back.

### The Anatomy of an HTTP Request

Every HTTP request has these parts:

```
┌──────────────────────────────────────────────────┐
│ METHOD    URL                                     │
│ POST      https://api.example.com/entries         │
│                                                   │
│ HEADERS                                           │
│ Content-Type: application/json                    │
│                                                   │
│ BODY                                              │
│ {                                                 │
│   "mood": "happy",                                │
│   "energy": 4,                                    │
│   "note": "Shipped a feature!"                    │
│ }                                                 │
└──────────────────────────────────────────────────┘
```

| Part | What It Is | Analogy |
|------|-----------|---------|
| **Method** | What you want to do (GET, POST, etc.) | The type of letter (question, order, cancellation) |
| **URL** | Where to send it | The mailing address |
| **Headers** | Metadata about the request | The envelope markings ("fragile", "priority") |
| **Body** | The actual data (not all requests have one) | The letter inside the envelope |

### HTTP Methods — The Verbs of the Web

| Method | Purpose | Restaurant Analogy | Has a Body? |
|--------|---------|-------------------|-------------|
| **GET** | Retrieve data | "Can I see the menu?" | No |
| **POST** | Create new data | "I'd like to order the pasta" | Yes |
| **PUT** | Replace existing data entirely | "Change my entire order to steak" | Yes |
| **PATCH** | Partially update data | "Add extra cheese to my order" | Yes |
| **DELETE** | Remove data | "Cancel my order" | Usually no |

For DevPulse, we only need two:
- `GET /api/entries` — Fetch all mood entries
- `POST /api/entries` — Submit a new mood entry

### HTTP Status Codes — The Server's Answer

Every response includes a number telling you what happened:

| Code | Category | Meaning | When You'll See It |
|------|----------|---------|-------------------|
| **200** | Success | OK | GET request worked |
| **201** | Success | Created | POST request created something |
| **400** | Client Error | Bad Request | You sent invalid data |
| **404** | Client Error | Not Found | That URL doesn't exist |
| **500** | Server Error | Internal Server Error | Your backend code has a bug |

The hundreds digit tells you the category: **2xx = success**, **4xx = you made a mistake**, **5xx = the server made a mistake**.

> ⚠️ **Common Mistake**
> Beginners often ignore status codes and only check if the request "worked." A request can return a 400 or 500 status and still technically complete — but the data won't be what you expect. Always check the status code in your code.

---

## REST API — Organizing Your Endpoints

> **Key Concept: REST (Representational State Transfer)**
> REST is a set of conventions for organizing your API endpoints. It's not a technology or library you install — it's a pattern everyone agrees to follow. Instead of random URLs like `/getStuff` or `/doTheThing`, REST uses consistent, predictable URLs based on the *resource* (the thing you're working with). Think of it like a library's filing system: every book has a predictable location based on its category.

### REST Conventions

```
GET    /api/entries      → Get all entries     (list)
GET    /api/entries/5    → Get entry #5        (single item)
POST   /api/entries      → Create new entry    (create)
PUT    /api/entries/5    → Replace entry #5    (full update)
DELETE /api/entries/5    → Delete entry #5     (remove)
```

Notice the pattern:
- The URL describes the **resource** (`entries`), not the **action**
- The HTTP **method** describes the action (GET, POST, PUT, DELETE)
- IDs go at the end of the URL path
- The base path `/api/` groups all API routes together

### 🧠 Think About It

If you were building a recipe app with ingredients, what would the REST endpoints look like? Write them out before checking.

<details>
<summary>Answer</summary>

```
GET    /api/recipes             → List all recipes
POST   /api/recipes             → Create a recipe
GET    /api/recipes/3           → Get recipe #3
PUT    /api/recipes/3           → Update recipe #3
DELETE /api/recipes/3           → Delete recipe #3
GET    /api/recipes/3/ingredients   → Get ingredients for recipe #3
POST   /api/recipes/3/ingredients   → Add ingredient to recipe #3
```

The nested resource (`/recipes/3/ingredients`) shows that ingredients belong to a recipe. You'll see this pattern in Level 2 with tasks belonging to projects.

</details>

---

## JSON — The Data Format

> **Key Concept: JSON (JavaScript Object Notation)**
> JSON is the standard format for sending data between client and server. It looks like a JavaScript object, but it's actually just a text string that both sides can parse. Think of it like a standardized form that both the sender and receiver know how to read.

```json
{
  "id": 1,
  "mood": "happy",
  "energy": 4,
  "note": "Shipped a feature today!",
  "date": "2025-01-15"
}
```

### Reading That JSON Character-by-Character

Every symbol in that snippet matters. Here's what each one is doing:

- `{` at the very start — "an object begins here." An object is a bag of named values.
- `}` at the very end — "the object ends here." Everything between the curly braces is the object's content.
- `"id"`, `"mood"`, `"energy"` — these are **keys** (sometimes called "field names"). They are always in double quotes in JSON.
- `:` — separates a key from its value. Read it as "equals" or "is."
- `1`, `4` — numbers. No quotes.
- `"happy"`, `"Shipped a feature today!"`, `"2025-01-15"` — strings (text). Strings are always in double quotes.
- `,` — separates one key/value pair from the next. There is no comma after the last pair — that would be a syntax error.
- Newlines and indentation — purely cosmetic. JSON doesn't care about whitespace. The whole thing could be on one line; it's formatted for humans.

So the object has five fields: `id` is the number 1, `mood` is the string "happy", `energy` is the number 4, `note` is a longer string, and `date` is a string that happens to look like a date (JSON has no real "date" type, so dates are stored as strings).

### JSON Rules (These Will Bite You)

| Rule | Valid | Invalid | Why |
|------|-------|---------|-----|
| Keys must be double-quoted | `"name": "Josh"` | `name: "Josh"` | JSON is a data format, not JavaScript |
| Strings use double quotes | `"hello"` | `'hello'` | Single quotes aren't valid JSON |
| No trailing commas | `{"a": 1}` | `{"a": 1,}` | Strict parsing spec |
| No comments | — | `// this breaks` | JSON is data, not code |

> ⚠️ **Common Mistake**
> In JavaScript, you can write `{ name: "Josh" }` (no quotes on the key). In JSON, this is **invalid**. JSON requires `{ "name": "Josh" }`. If you get parse errors when your backend receives data, check that you're setting the `Content-Type: application/json` header and sending valid JSON.

---

## Ports — How Multiple Servers Share One Machine

> **Key Concept: Port**
> A port is like an apartment number in a building. Your computer (the building) has one address (`localhost`), but many programs can run simultaneously on different port numbers. Each port is a separate entrance to a different service.

```
YOUR COMPUTER (localhost)
├── Port 5173  →  Frontend (Vite dev server)
├── Port 3001  →  Backend (Express server)
├── Port 5432  →  PostgreSQL (Level 2)
└── Port 3000  →  (some other app)
```

In development, both your frontend and backend run on **your machine** but on **different ports**:
- Frontend: `http://localhost:5173`
- Backend: `http://localhost:3001`

### 🧠 Think About It

What would happen if you tried to start two servers on the same port?

<details>
<summary>Answer</summary>

You'd get an `EADDRINUSE` error ("address already in use"). Only one program can listen on a given port at a time. If you see this error, it means another process is already using that port. Either stop the other process or change your port number.

</details>

---

## CORS — Why the Browser Blocks You

> **Key Concept: CORS (Cross-Origin Resource Sharing)**
> CORS is a browser security feature that prevents a webpage from making requests to a different domain (or port) than the one it was served from. Your frontend at `localhost:5173` and your backend at `localhost:3001` are considered different "origins," so the browser blocks the request by default. The backend must explicitly say "I allow requests from this origin."

```
Frontend (localhost:5173)                Backend (localhost:3001)
         │                                        │
         │   "Hey, I want data!"                   │
         │────────────────────────────────────────▶│
         │                                        │
         │   Browser checks: "Is 5173 allowed     │
         │   to talk to 3001?"                     │
         │                                        │
         │   If no CORS header → BLOCKED           │
         │   If CORS header says "5173 OK" → ✅    │
         │◀────────────────────────────────────────│
```

### The Fix (You'll Implement This in Step 3)

On the backend, you install the `cors` package and tell Express which origins are allowed:

```typescript
import cors from 'cors';

app.use(cors({
  origin: 'http://localhost:5173'  // No trailing slash!
}));
```

### Reading That Snippet Line-by-Line

Even if you've never seen code before, every piece of that four-line example is nameable:

- `import cors from 'cors';`
  - `import` is a command that reads "pull code from another file or package so I can use it here."
  - `cors` is the name we're giving to whatever we imported. We picked this name ourselves — we could have written `import anyNameWeWant from 'cors'` and it would still work.
  - `from 'cors'` is the source. `'cors'` (in quotes) is the name of an npm package — a pre-made bundle of code that someone else wrote and published. We'll install it with `npm install cors` later.
  - The `;` at the end is a **semicolon**. It marks the end of a statement in JavaScript/TypeScript. (Some code styles skip semicolons entirely; both are valid.)
- `app.use(cors({ ... }));`
  - `app` is an Express application object — we'll create it in Step 3. Think of it as "our backend server, represented as an object in code."
  - `.use(...)` means "run this code for every incoming request." You'll see this pattern repeatedly.
  - `cors({ ... })` is a **function call**. `cors` is the thing we imported; the parentheses `()` mean "call it now." The curly-brace object inside `{ ... }` is the single argument we're handing to that function.
  - `{ origin: 'http://localhost:5173' }` is an **object literal** — a temporary object made on the spot with one field, `origin`, set to the string `'http://localhost:5173'`.
  - `// No trailing slash!` is a **comment**. Anything after `//` on a line is ignored by the computer; it's a note for humans reading the code.

Don't panic if this feels dense. You'll see the same shapes (imports, function calls, object literals) hundreds of times across Level 1. Repetition will lock them in.

> ⚠️ **Common Mistake: Trailing Slash in CORS Origin**
> `'http://localhost:5173/'` is NOT the same as `'http://localhost:5173'`. The browser sends origins **without** a trailing slash. If your CORS config includes one, every request gets silently blocked. This mistake has derailed countless first deployments. Always omit the trailing slash.

---

## DevPulse — The App We're Building

Here's the complete picture:

### API Overview

| Method | Endpoint | Request Body | Response | Status |
|--------|----------|-------------|----------|--------|
| GET | /api/entries | None | Array of entries | 200 |
| POST | /api/entries | `{ mood, energy, note }` | Created entry | 201 |
| GET | /api/health | None | `{ status: "ok" }` | 200 |

### Data Flow

```
USER                    FRONTEND                   BACKEND
────                    ────────                   ───────
Types mood,         EntryForm component         POST /api/entries
energy, note  ───▶  collects form data    ───▶  validates data
                                                saves to array
                                                returns new entry
                    EntryList component   ◀───  
                    displays all entries        GET /api/entries
                                                reads from array
                    MoodChart component         returns all entries
                    shows visual summary
```

---

## 🧠 Spatial Check-In

Before moving to Step 2, answer these without looking back. These test understanding, not recall.

1. Your frontend is at `localhost:5173` and your backend is at `localhost:3001`. You make a fetch request and get a CORS error. Which side do you fix — frontend or backend? Why?

2. You send a POST request to create a mood entry and the server returns status 400. What does this mean? What should you check?

3. Why can't we just store mood entries in the browser's localStorage instead of building a backend?

4. A friend designs an API with these endpoints: `GET /getAllEntries`, `POST /makeNewEntry`, `DELETE /removeEntry?id=5`. What's wrong with this design, and how would you fix it using REST conventions?

<details>
<summary>Check Your Answers</summary>

1. **Backend.** CORS is configured on the server. The server must include headers telling the browser which origins are allowed. The frontend cannot bypass CORS — that's the whole point.

2. **400 = Bad Request.** Your data was invalid. Check: Did you send all required fields? Are the types correct (number vs string)? Is the JSON well-formed? Look at the response body for specific error details.

3. **Three problems:** (a) Data is lost if the user clears browser storage. (b) Data isn't accessible from other devices. (c) Data can't be shared between users. A backend is the single source of truth.

4. **Problem:** URLs describe actions ("get", "make", "remove") instead of resources. **Fix:** Use REST conventions where the method describes the action and the URL describes the resource:
   - `GET /getAllEntries` → `GET /api/entries`
   - `POST /makeNewEntry` → `POST /api/entries`
   - `DELETE /removeEntry?id=5` → `DELETE /api/entries/5`

</details>

---

> **Session Break** — You've completed the orientation.
> When you return, you'll scaffold the project in [Step 2 — Project Setup](../02-project-setup/).

---

| | |
|:---|---:|
| [← Level 1 Overview](../) | [Step 2 — Project Setup →](../02-project-setup/) |

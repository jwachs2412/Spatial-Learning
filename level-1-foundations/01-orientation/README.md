`Level 1` **Step 1 of 7** — Spatial Orientation

# 01 — Spatial Orientation: Understanding the Web

Before writing a single line of code, you need to understand **where** your code runs. This is the most important mental model in full-stack development.

---

## The Two Worlds

Every web application exists in two worlds simultaneously:

```
┌──────────────────────────────┐          ┌──────────────────────────────┐
│         THE BROWSER          │          │         THE SERVER           │
│        (Client-Side)         │          │        (Server-Side)         │
│                              │          │                              │
│  This is the user's machine  │   HTTP   │  This is YOUR machine       │
│  You do NOT control it       │ ◀──────▶ │  (or a cloud computer)      │
│  Anyone can inspect it       │          │  You DO control it           │
│  Never trust it completely   │          │  Users cannot see your code  │
│                              │          │                              │
│  Runs: HTML, CSS, JavaScript │          │  Runs: Node.js, Express     │
│  Tools: React, TypeScript    │          │  Stores: Data, secrets       │
│                              │          │                              │
│  WHAT THE USER SEES          │          │  WHAT THE USER CANNOT SEE   │
└──────────────────────────────┘          └──────────────────────────────┘
```

> [!NOTE]
> **Technical**: The client is the web browser making HTTP requests. The server is a process (running Node.js, in our case) listening on a port, receiving requests, and sending responses.

> [!NOTE]
> **Plain English**: The browser is the customer walking into a restaurant. The server is the kitchen. The customer sees the menu and gets food delivered to their table. They never see the kitchen, the recipes, or the ingredients list. HTTP requests are the waiter carrying orders back and forth.

---

## How an HTTP Request Works

When you type a URL into your browser or click a button that fetches data, here's what actually happens:

```
   YOUR BROWSER                                    THE SERVER
   ────────────                                    ──────────

   1. "I need data"
      │
      │  Creates an HTTP Request:
      │  ┌─────────────────────────────────┐
      │  │ GET /api/entries HTTP/1.1        │
      │  │ Host: localhost:3001             │
      │  │ Accept: application/json        │
      │  └─────────────────────────────────┘
      │
      ├──────────── INTERNET ──────────────▶  2. Express receives request
                                              │
                                              │  Finds matching route:
                                              │  app.get('/api/entries', handler)
                                              │
                                              │  Handler runs:
                                              │  - Reads data from storage
                                              │  - Formats it as JSON
                                              │
                                              │  Creates HTTP Response:
                                              │  ┌──────────────────────────────┐
                                              │  │ HTTP/1.1 200 OK              │
                                              │  │ Content-Type: application/json│
                                              │  │                              │
                                              │  │ [{"mood":"happy","energy":4}] │
                                              │  └──────────────────────────────┘
                                              │
   3. Browser receives response  ◀────────────┘
      │
      │  JavaScript (React) reads the JSON
      │  Updates the UI with the new data
      │
   4. User sees updated screen
```

### Key parts of an HTTP request:

| Part | What It Is | Example |
|------|-----------|---------|
| **Method** | What action to perform | GET, POST, PUT, DELETE |
| **URL/Path** | Where to send the request | `/api/entries` |
| **Headers** | Metadata about the request | `Content-Type: application/json` |
| **Body** | Data sent with the request (POST/PUT only) | `{"mood": "happy"}` |

### Key parts of an HTTP response:

| Part | What It Is | Example |
|------|-----------|---------|
| **Status Code** | Did it work? | 200 (yes), 404 (not found), 500 (server error) |
| **Headers** | Metadata about the response | `Content-Type: application/json` |
| **Body** | The actual data | `[{"mood": "happy", "energy": 4}]` |

### Common Status Codes

| Code | Meaning | Plain English |
|------|---------|---------------|
| **200** | OK | "Here's what you asked for" |
| **201** | Created | "I made the thing you asked me to make" |
| **400** | Bad Request | "I don't understand what you sent me" |
| **404** | Not Found | "That thing doesn't exist" |
| **500** | Internal Server Error | "Something broke on my end" |

---

## REST API — A Communication Contract

REST (Representational State Transfer) is a set of conventions for how clients and servers communicate.

> [!NOTE]
> **Technical**: REST is an architectural style that uses standard HTTP methods to perform CRUD operations on resources identified by URLs.

> [!NOTE]
> **Plain English**: REST is an agreed-upon way for the frontend and backend to talk to each other. It's like a language both sides understand.

### The Convention

For a resource called "entries":

| Action | HTTP Method | URL | What It Does |
|--------|-------------|-----|-------------|
| Get all entries | `GET` | `/api/entries` | Read all entries |
| Get one entry | `GET` | `/api/entries/123` | Read entry with ID 123 |
| Create an entry | `POST` | `/api/entries` | Create a new entry |
| Update an entry | `PUT` | `/api/entries/123` | Replace entry 123 |
| Delete an entry | `DELETE` | `/api/entries/123` | Remove entry 123 |

**Why these conventions matter**: If every developer invents their own URL patterns, nobody can predict how an API works. REST gives us shared expectations. When you see `GET /api/users`, you know it returns a list of users — you don't need documentation to guess.

### Why We Say "REST API" (Not GraphQL, Not gRPC)

- **REST over GraphQL**: REST is simpler to understand and implement. GraphQL is powerful for complex data requirements but adds significant complexity. You should learn REST first — it covers 90% of use cases and is what most employers expect.
- **REST over gRPC**: gRPC uses binary protocols and is designed for microservice communication. REST uses JSON over HTTP, which is human-readable and browser-friendly. gRPC is not for this stage of learning.

---

## JSON — The Language of Web APIs

JSON (JavaScript Object Notation) is how data travels between frontend and backend.

```json
{
  "id": 1,
  "mood": "happy",
  "energy": 4,
  "note": "Solved a tricky bug today",
  "createdAt": "2026-03-01T14:30:00.000Z"
}
```

> [!NOTE]
> **Technical**: JSON is a lightweight data interchange format that uses human-readable text to represent data objects consisting of key-value pairs and arrays.

> [!NOTE]
> **Plain English**: It's a way to write data that both humans and computers can read. It looks like a JavaScript object (because it was inspired by one), and almost every programming language can read and write it.

### Rules of JSON
- Keys must be strings (in double quotes)
- Values can be: strings, numbers, booleans, arrays, objects, or null
- No trailing commas
- No comments

---

## Ports — Addresses Within Your Computer

When you run a server, it "listens" on a port. A port is like an apartment number in a building.

```
YOUR COMPUTER (the building) = localhost
│
├── Port 5173 → Frontend dev server (Vite)
├── Port 3001 → Backend server (Express)
├── Port 5432 → PostgreSQL (in later levels)
└── Port 443  → HTTPS (used by browsers for secure sites)
```

> [!NOTE]
> **Technical**: A port is a 16-bit number (0–65535) that identifies a specific process or service on a computer. When a server binds to a port, it receives all network traffic directed to that port.

> [!NOTE]
> **Plain English**: Your computer can run many servers at once. Ports are how the computer knows which server should receive which request. `localhost:3001` means "talk to the program on my own computer that's listening on port 3001."

---

## CORS — The Browser's Bodyguard

You will almost certainly hit a CORS error in this project. Here's what it is.

```
BROWSER (localhost:5173)          SERVER (localhost:3001)
┌────────────────────┐           ┌──────────────────────┐
│ React app says:    │           │                      │
│ "fetch data from   │──────────▶│ "Sure, here's the   │
│  localhost:3001"   │           │  data"               │
│                    │◀──────────│                      │
│                    │           └──────────────────────┘
│ Browser says:      │
│ "WAIT. The page is │
│  from :5173 but the│
│  data is from :3001│
│  That's a DIFFERENT│
│  ORIGIN. Blocked!" │
└────────────────────┘
```

> [!NOTE]
> **Technical**: CORS (Cross-Origin Resource Sharing) is a browser security mechanism that blocks web pages from making requests to a different origin (different protocol, domain, or port) than the one that served the page, unless the server explicitly allows it.

> [!NOTE]
> **Plain English**: The browser is paranoid (for good reason). If your page comes from one address, the browser blocks requests to a different address unless the server says "it's okay, I trust that page." We fix this by telling our Express server to allow requests from our frontend's address.

### The Fix (We'll Implement This)

```typescript
// On the backend:
import cors from 'cors';

app.use(cors({
  origin: 'http://localhost:5173'  // "I trust requests from this address"
}));
```

---

> [!TIP]
> ## Spatial Check-In
> Before moving to the next section, make sure you can answer these questions. If any feel unclear, re-read the relevant section. These are not trivia — they are the mental model you'll use for the rest of your career.

1. **Where does React code run?**

<details><summary>Answer</summary>

In the browser, on the user's machine

</details>

2. **Where does Express code run?**

<details><summary>Answer</summary>

On the server (your machine or a cloud server)

</details>

3. **How do they communicate?**

<details><summary>Answer</summary>

HTTP requests and responses

</details>

4. **What format is data sent in?**

<details><summary>Answer</summary>

JSON

</details>

5. **What is a port?**

<details><summary>Answer</summary>

A number that identifies which program receives network traffic

</details>

6. **What is CORS?**

<details><summary>Answer</summary>

A browser security feature that blocks cross-origin requests unless the server allows them

</details>

7. **What is REST?**

<details><summary>Answer</summary>

A convention for structuring API URLs and using HTTP methods

</details>

---

| | | |
|:---|:---:|---:|
| | [Level 1 Overview](../) | [02 — Project Setup →](../02-project-setup/) |

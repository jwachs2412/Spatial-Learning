`Level 3` **Step 1 of 8** — Orientation

# 01 — Orientation: Security and Identity on the Web

> [!WARNING]
> ## Prerequisites Gate
>
> Level 3 requires **completed and deployed** Level 1 and Level 2 projects. Before continuing, you must have all of the following:
>
> 1. **Level 1 — DevPulse**: GitHub repo URL + Vercel frontend URL + Render backend health check URL
> 2. **Level 2 — TaskForge**: GitHub repo URL + Vercel frontend URL + Render backend health check URL (with `database: "connected"`)
>
> If either project is incomplete, go back and finish it first. Level 3 builds directly on everything from Levels 1 and 2.

### Skills Checklist

You should be confident with all of the following. If anything feels shaky, review the relevant Level 2 section.

- [ ] Design a database schema with foreign keys
- [ ] Write SQL for INSERT, SELECT, UPDATE, DELETE
- [ ] Use parameterized queries to prevent SQL injection
- [ ] Build Express routes with try/catch and error middleware
- [ ] Use environment variables with dotenv
- [ ] Build React components with useState and useEffect
- [ ] Deploy a three-tier app (frontend + backend + database)

All checked? Time to add security.

---

## The Problem: Who Is Making This Request?

In Level 2, TaskForge had a critical flaw:

```
ANY USER (or bot, or attacker) can:
  - Create projects
  - Delete any project
  - Read anyone's tasks
  - Delete anyone's tasks

There is NO identity. The server has NO idea who is making a request.
```

This is fine for a learning project, but it's a dealbreaker for real applications. Imagine if Gmail let anyone read anyone's emails, or if your bank let anyone transfer money from any account.

**Authentication** answers the question: "Who are you?"
**Authorization** answers the question: "Are you allowed to do this?"

Level 3 adds both.

---

## Authentication vs Authorization

> [!NOTE]
> **Technical**: Authentication (authn) is the process of verifying a user's identity — confirming they are who they claim to be. Authorization (authz) is the process of determining what an authenticated user is permitted to do.

> [!NOTE]
> **Plain English**: Authentication is checking someone's ID at the door. Authorization is checking whether their ID grants them access to the VIP section. You must authenticate first (prove who you are), then the system authorizes you (decides what you can do).

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  AUTHENTICATION (Who are you?)                              │
│  ─────────────────────────────                              │
│  "I'm user@example.com"                                    │
│  "Prove it" → password check → JWT token issued             │
│                                                             │
│  AUTHORIZATION (What can you do?)                           │
│  ────────────────────────────────                           │
│  "I want to see note #5"                                    │
│  "Is note #5 yours?" → check user_id → yes → here it is   │
│  "Is note #5 yours?" → check user_id → no → 403 Forbidden │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Trust Boundaries

This is the most important security concept. A **trust boundary** is the line between what you control and what you don't.

```
┌──────────────────────────────────────────────────────────────────┐
│                        TRUSTED ZONE                               │
│                  (You control this code)                          │
│                                                                  │
│   ┌──────────────────┐        ┌──────────────────────────┐      │
│   │     BACKEND       │        │       DATABASE            │      │
│   │  Express server   │  SQL   │    PostgreSQL             │      │
│   │  Your code runs   │◀─────▶│    Your schema            │      │
│   │  here              │        │    Your data              │      │
│   └────────┬─────────┘        └──────────────────────────┘      │
│            │                                                     │
│════════════╪═════════════════════════════════════════════════════│
│            │         TRUST BOUNDARY                              │
│════════════╪═════════════════════════════════════════════════════│
│            │                                                     │
│   ┌────────▼─────────┐                                          │
│   │     FRONTEND      │        UNTRUSTED ZONE                    │
│   │  Browser (React)  │        (User controls this)             │
│   │                  │                                          │
│   │  User can:       │        - Inspect all JavaScript          │
│   │  - Read all JS   │        - Modify localStorage             │
│   │  - Edit localStorage      - Send any HTTP request           │
│   │  - Use DevTools  │        - Forge request headers           │
│   │  - Modify requests│        - Call your API from curl          │
│   └──────────────────┘                                          │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

**Rules that follow from this:**

1. **Never trust data from the frontend.** Always validate on the backend. (You learned this in Level 2.)
2. **Never store secrets in frontend code.** Anyone can read it. JWT_SECRET lives on the server only.
3. **Never send passwords in plain text to the database.** Hash them with bcrypt first.
4. **The backend is the authority.** The frontend can suggest, request, and display — but the backend decides.
5. **Authentication happens on the backend.** The frontend just sends credentials and stores the token.

---

## How Passwords Work (The Right Way)

> [!WARNING]
> **Never store passwords in plain text.** If your database is breached, every user's password is exposed. Instead, store a **hash** — a one-way transformation that can't be reversed.

```
WRONG (plain text):
───────────────────
users table:
│ id │ email              │ password      │
│  1 │ alice@example.com  │ mypassword123 │  ← If leaked, attacker has the password
│  2 │ bob@example.com    │ bob_secret    │

RIGHT (bcrypt hash):
────────────────────
users table:
│ id │ email              │ password_hash                                              │
│  1 │ alice@example.com  │ $2b$10$X7vK9z...q3Fy (60 characters)                      │
│  2 │ bob@example.com    │ $2b$10$Rp4mA1...kL2w (60 characters)                      │
                            ↑ Even if leaked, attacker can't reverse this to a password
```

> [!NOTE]
> **Technical**: bcrypt is a password hashing function that incorporates a salt (random data) and a cost factor (work factor). The salt prevents rainbow table attacks. The cost factor makes brute-force attacks computationally expensive. bcrypt.hash() produces a different hash each time for the same input (because of the random salt), but bcrypt.compare() can verify a password against any of its hashes.

> [!NOTE]
> **Plain English**: bcrypt turns a password into a scrambled string. The scrambling is one-way — you can't unscramble it. When a user logs in, bcrypt doesn't unscramble the stored hash. Instead, it scrambles the password the user just typed and checks if the result matches. If it matches, the password is correct.

---

## How JWT Tokens Work

After login, the server creates a **token** — a small string that proves the user is authenticated.

> [!NOTE]
> **Technical**: A JSON Web Token (JWT) is a compact, URL-safe token format that encodes claims (data) as a JSON payload, signed with a secret key using HMAC or RSA. The server creates it, the client stores and sends it, and the server verifies it on each request without needing a database lookup.

> [!NOTE]
> **Plain English**: A JWT is like a stamped wristband at a concert. The security guard (server) stamps it when you enter (login). Every time you want to access a restricted area (protected route), you show the wristband. The guard can verify it's real by checking the stamp — without looking you up in a list. If someone tries to forge the stamp, it won't match.

### Anatomy of a JWT

```
eyJhbGciOiJIUzI1NiJ9.eyJ1c2VySWQiOjEsInJvbGUiOiJ1c2VyIn0.abc123signature
│                     │ │                                      │ │              │
└─────────────────────┘ └──────────────────────────────────────┘ └──────────────┘
       HEADER                        PAYLOAD                       SIGNATURE
   (algorithm used)            (data about the user)          (proves it's genuine)

HEADER:  { "alg": "HS256" }
PAYLOAD: { "userId": 1, "role": "user" }
SIGNATURE: HMAC-SHA256(header + payload, JWT_SECRET)
```

**The signature is the key.** It's created using a secret that only the server knows (`JWT_SECRET`). Anyone can read the header and payload (they're just Base64-encoded), but nobody can create a valid signature without the secret. If someone modifies the payload (e.g., changes `userId` from 1 to 2), the signature won't match and the server rejects it.

### Token Lifecycle

```
1. User logs in → Server creates JWT with user data + signs it
2. Server sends JWT to client
3. Client stores JWT in localStorage
4. Client sends JWT on every request: Authorization: Bearer <token>
5. Server middleware verifies signature on every request
6. If valid → request proceeds with user context
7. If invalid/expired → 401 Unauthorized
```

---

## What VaultNote Does

**Public pages** (no login required):
- Register page — create an account
- Login page — sign in to an existing account

**Protected pages** (login required):
- Notes list — see all YOUR notes
- Note editor — create, edit, delete a note

**Admin features** (admin role required):
- Admin stats — view how many users and notes exist

**Key security rules:**
- Users can only see their own notes
- Users can only edit/delete their own notes
- Passwords are hashed with bcrypt
- Every protected request requires a valid JWT
- JWT_SECRET is never in code — only in environment variables

---

## API Overview

| Method | Endpoint | Auth? | Purpose |
|--------|----------|-------|---------|
| POST | `/api/auth/register` | No | Create account |
| POST | `/api/auth/login` | No | Get JWT token |
| GET | `/api/auth/me` | Yes | Get current user info |
| GET | `/api/notes` | Yes | List user's notes |
| POST | `/api/notes` | Yes | Create a note |
| GET | `/api/notes/:id` | Yes | Get one note (must be owner) |
| PUT | `/api/notes/:id` | Yes | Update note (must be owner) |
| DELETE | `/api/notes/:id` | Yes | Delete note (must be owner) |
| GET | `/api/admin/stats` | Admin | Usage statistics |
| GET | `/api/health` | No | Health check |

Notice the pattern: auth routes are public, everything else is protected.

---

> [!TIP]
> ## Spatial Check-In
> Make sure you understand these concepts before building. They'll come up in every interview about web security.

1. **What is the difference between authentication and authorization?**

<details><summary>Answer</summary>

Authentication verifies who you are (login). Authorization determines what you're allowed to do (permissions).

</details>

2. **Why do we hash passwords instead of storing them in plain text?**

<details><summary>Answer</summary>

If the database is breached, hashed passwords can't be reversed to plain text. Plain text passwords would be immediately usable by attackers.

</details>

3. **What is a trust boundary?**

<details><summary>Answer</summary>

The line between what you control (backend/database) and what you don't (browser/frontend). Never trust anything from the untrusted side without validation.

</details>

4. **How does a JWT prove a user is authenticated?**

<details><summary>Answer</summary>

The JWT contains user data (payload) and a cryptographic signature created with a secret only the server knows. The server verifies the signature on each request — if it matches, the token is genuine and the user is who they claim to be.

</details>

5. **Why can't someone forge a JWT by changing the userId in the payload?**

<details><summary>Answer</summary>

The signature is computed from the payload + a secret key. Changing the payload invalidates the signature. Without the secret key, they can't create a new valid signature.

</details>

6. **Why is JWT_SECRET stored in environment variables, not in code?**

<details><summary>Answer</summary>

Code is pushed to GitHub and visible to anyone. If the secret is in code, anyone can forge valid tokens. Environment variables stay on the server and never enter version control.

</details>

---

| | | |
|:---|:---:|---:|
| | [Level 3 Overview](../) | [02 — Project Setup →](../02-project-setup/) |

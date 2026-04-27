# Step 1 — Orientation: Security and Identity on the Web

> **Prerequisites:** You must have completed Levels 1 and 2 before starting Level 3.

## The Problem We're Solving

In TaskForge, anyone could create, read, update, or delete any project. There were no user accounts — it was a shared, open application. VaultNote changes this: every note belongs to a specific user, and only that user can access it.

This requires answering two fundamental questions:
1. **Authentication** — "Who are you?" (prove your identity)
2. **Authorization** — "What are you allowed to do?" (check permissions)

---

## Authentication vs Authorization

> **Key Concept: Authentication (AuthN)**
> Verifying *who* someone is. "You claim to be user@example.com — prove it by providing the correct password." Think of it like showing your ID at the door.

> **Key Concept: Authorization (AuthZ)**
> Verifying *what* someone is allowed to do. "You are user@example.com — but are you allowed to delete this note?" Think of it like a keycard system: your ID gets you in the building (authentication), but your keycard only opens certain doors (authorization).

```
1. User sends email + password     → Authentication ("who are you?")
2. Server verifies credentials     → Authentication
3. Server creates JWT token        → Authentication
4. User sends token with request   → "I'm user #5"
5. Server checks: "Is user #5     → Authorization ("can you do this?")
   allowed to access note #12?"
```

---

## Trust Boundaries

> **Key Concept: Trust Boundary**
> A trust boundary separates code you control from code the user controls. Everything in the browser is on the user's side — they can modify JavaScript, edit localStorage, forge HTTP requests. Everything on the server is on your side — the user can't touch it directly.

```
                    UNTRUSTED ZONE (browser)
                    ├── React components
                    ├── localStorage (token storage)
                    ├── Form data
                    └── JavaScript code (user can modify!)
─────────────────────── TRUST BOUNDARY ────────────────────
                    TRUSTED ZONE (server)
                    ├── Auth middleware (validates tokens)
                    ├── Route handlers (enforce rules)
                    ├── Database queries (filter by user_id)
                    └── JWT_SECRET (never exposed)
```

### Three Rules of Trust Boundaries

1. **Never trust frontend data** — Always validate on the server
2. **Never store secrets in frontend code** — No API keys, no JWT secrets, no database URLs
3. **Never send plain-text passwords** — Hash with bcrypt before storing

### 🧠 Think About It

A developer stores the user's role in localStorage: `{ role: "user" }`. The frontend checks this role to show/hide the admin panel. What's wrong with this approach?

<details>
<summary>Answer</summary>

Anyone can open DevTools → Application → localStorage and change `"user"` to `"admin"`. The frontend check is trivially bypassed. **Authorization must be enforced on the server.** The frontend can use the role for UI purposes (showing/hiding buttons), but the server must verify the role on every request. The server's JWT contains the role, and the server verifies the JWT signature — the user can't forge that.

</details>

---

## Password Hashing with bcrypt

> **Key Concept: Password Hashing**
> A hash is a one-way transformation — you can turn a password into a hash, but you can never turn the hash back into the password. Think of it like a meat grinder: you can put beef in and get ground beef out, but you can never reconstruct the original cut.
>
> **The Threat:** If someone steals your database, they get every row in the users table. Without hashing, they see every password in plain text. With hashing, they see meaningless strings like `$2b$10$N9qo8uLOickgx2ZMRZoMye...`.

```
Registration:
  "mypassword123"  →  bcrypt.hash()  →  "$2b$10$N9qo8uLO..."
  (stored in database: only the hash, never the password)

Login:
  "mypassword123"  →  bcrypt.compare(password, hash)  →  true/false
  (bcrypt re-hashes the input and compares — it doesn't "decrypt")
```

### Why bcrypt (not SHA-256, not MD5)?

| Algorithm | Speed | Purpose | Safe for Passwords? |
|-----------|-------|---------|-------------------|
| MD5 | Extremely fast | File checksums | No — too fast to crack |
| SHA-256 | Very fast | Data integrity | No — too fast to crack |
| **bcrypt** | **Intentionally slow** | **Password hashing** | **Yes** |

bcrypt is deliberately slow (configurable via "cost factor"). This means an attacker who steals your database can only try ~100 passwords/second instead of billions. Speed is the enemy when it comes to passwords.

---

## JWT Tokens

> **Key Concept: JWT (JSON Web Token)**
> A JWT is a string that contains encoded information (like user ID and role) and a cryptographic signature. The server creates it at login, the client stores it, and the client sends it with every subsequent request. The server verifies the signature to confirm the token hasn't been tampered with.

### JWT Anatomy

```
eyJhbGciOiJIUzI1NiJ9.eyJ1c2VySWQiOjEsInJvbGUiOiJ1c2VyIn0.abc123signature

      HEADER              PAYLOAD                      SIGNATURE
  (algorithm)        (user data)                  (cryptographic proof)
  {"alg":"HS256"}    {"userId":1,"role":"user"}    HMAC(header+payload, SECRET)
```

### Reading a JWT Character-by-Character

A JWT is one string with no spaces. The structure looks intimidating until you know what each part is.

- The `.` (period) characters are **separators**. There are exactly two of them, dividing the string into three sections: header, payload, signature.
- Each section is **Base64-encoded** — a way to express any data using only letters, numbers, hyphens, and underscores. This makes the token safe to put in URLs, headers, and cookies.
- **Base64 is not encryption.** Anyone can decode the header and payload to see their contents. Try copying a real JWT and pasting it at [jwt.io](https://jwt.io) — you'll see the original JSON. (Don't paste production tokens into web tools.)
- The **header** decodes to JSON like `{"alg":"HS256","typ":"JWT"}` — algorithm name and type.
- The **payload** decodes to JSON like `{"userId":1,"role":"user","iat":1735689600,"exp":1736294400}` — the data the server stamped in. `iat` = issued-at timestamp; `exp` = expiration timestamp.
- The **signature** is the result of running `header.payload` through HMAC-SHA256 keyed by your `JWT_SECRET`. It's the only piece that requires the secret to produce.
- **Why this is secure:** an attacker can edit the payload to claim `userId: 999` and re-encode it. But to make the new payload's signature match, they'd need the `JWT_SECRET` — which only lives on your server. When the server verifies the tampered token, the signature check fails and the request is rejected.

The three parts are separated by dots and are base64-encoded (not encrypted — anyone can decode them). The security comes from the **signature**: if anyone changes the payload (e.g., changing `userId` from 1 to 2), the signature won't match, and the server rejects the token.

### Token Lifecycle

```
1. User registers or logs in
2. Server validates credentials
3. Server creates JWT: jwt.sign({ userId, role }, JWT_SECRET)
4. Server sends JWT to client
5. Client stores JWT in localStorage
6. Client includes JWT in every request: Authorization: Bearer <token>
7. Server's auth middleware: jwt.verify(token, JWT_SECRET)
8. If valid → extract userId and role → continue to route handler
9. If invalid/expired → return 401 Unauthorized
```

---

## 🧠 Spatial Check-In

1. What's the difference between authentication and authorization? Give an example of each.

2. Why can't we just store passwords as plain text in the database?

3. A JWT contains the user's role (`"admin"` or `"user"`). Could a user decode their token, change `"user"` to `"admin"`, and gain admin access?

4. Why do we use bcrypt instead of SHA-256 for password hashing?

<details>
<summary>Check Your Answers</summary>

1. **Authentication** = proving identity ("Are you who you claim to be?" — verified by password). **Authorization** = checking permissions ("Are you allowed to do this?" — verified by role or ownership). Example: Logging in is authentication. Checking if you can delete a note is authorization.

2. **Database breaches happen.** If passwords are stored in plain text and the database is compromised, every user's password is exposed. With bcrypt hashing, the attacker gets useless hashes that can't be reversed back to passwords.

3. **No.** They can decode and modify the payload (it's just base64), but they can't recreate the signature without the `JWT_SECRET`, which only lives on the server. When the server verifies the modified token, the signature check fails, and the request is rejected with 401.

4. **bcrypt is intentionally slow.** SHA-256 can compute billions of hashes per second, making brute-force attacks trivial. bcrypt's configurable cost factor limits attempts to ~100/second, making brute-force impractical.

</details>

---

> **Session Break** — Security foundations understood.
> When you return, you'll scaffold the project in [Step 2 — Project Setup](../02-project-setup/).

---

| | |
|:---|---:|
| [← Level 3 Overview](../) | [Step 2 — Project Setup →](../02-project-setup/) |

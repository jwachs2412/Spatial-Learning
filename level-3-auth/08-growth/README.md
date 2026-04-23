# Step 8 — Growth Review: What You Learned and How to Talk About It

## What You Just Built

VaultNote is a secure, multi-user notes application with:
- **JWT authentication** (register, login, token verification)
- **bcrypt password hashing** (one-way, with configurable cost factor)
- **Auth middleware** (verifies tokens on every protected request)
- **Row-level security** (every query filters by `user_id`)
- **Role-based access control** (user vs admin)
- **React Router** (client-side navigation with protected routes)
- **Auth Context** (shared auth state across components)
- **Secret management** (JWT_SECRET in environment variables only)

---

## Skills Demonstrated to Employers

| Skill | What You Did | Interview Talking Point |
|-------|-------------|----------------------|
| **Auth System Design** | JWT + bcrypt + middleware | "I built authentication from scratch using JWT for stateless sessions and bcrypt for password hashing" |
| **Security Awareness** | Trust boundaries, hash vs encrypt, generic error messages | "I understand trust boundaries — the browser is untrusted, the server enforces all rules" |
| **Middleware Architecture** | authenticate → requireAdmin chaining | "I use middleware composition for authorization — authenticate first, then check roles" |
| **Protected API Design** | `WHERE user_id = $1` on every query | "Every database query filters by the authenticated user's ID — authorization happens at the database level" |
| **Client-Side Routing** | React Router with ProtectedRoute | "Unauthenticated users are redirected to login, but the real security is server-side middleware" |
| **Secrets Management** | JWT_SECRET in env vars, never in code | "Secrets live in environment variables, never in source code, with different values per environment" |

---

## Interview Questions

**"Walk me through your authentication flow."**
> "User submits email and password. The server hashes the password with bcrypt and stores it. On login, bcrypt.compare checks the submitted password against the stored hash. If valid, the server creates a JWT containing the user ID and role, signed with a secret key. The client stores the token in localStorage and sends it in the Authorization header on every request. The server's auth middleware verifies the signature before allowing access."

**"Why bcrypt and not SHA-256?"**
> "SHA-256 is fast — billions of hashes per second. That's great for checksums but terrible for passwords, because an attacker with a stolen database can brute-force quickly. bcrypt is intentionally slow with a configurable cost factor, limiting attempts to ~100/second."

**"Where do you store the JWT?"**
> "In localStorage for this project. The tradeoff: localStorage is vulnerable to XSS (cross-site scripting) — if an attacker injects JavaScript, they can read the token. HTTP-only cookies would be more secure because JavaScript can't access them, but they add complexity with CSRF protection. For this scope, localStorage with proper input sanitization is acceptable."

**"How do you prevent one user from accessing another's notes?"**
> "Every database query includes `AND user_id = $1` with the authenticated user's ID from the JWT. Even if a user guesses another note's ID, the query returns nothing because the user_id doesn't match. I also return 404 instead of 403 to avoid revealing that the note exists."

---

## What's Missing (and Why)

| Missing | Why | Priority |
|---------|-----|----------|
| Token refresh | Tokens expire but users must re-login | High (production need) |
| OAuth / social login | Google, GitHub sign-in | Medium (convenience) |
| Rate limiting | Prevent brute-force login attempts | High (Level 4) |
| Password reset | Email-based reset flow | Medium (production need) |
| CSRF protection | Needed if using cookies instead of localStorage | Depends on auth strategy |
| Input sanitization | XSS prevention | High (production need) |
| Automated tests | Testing auth flows | Level 4 |

---

## Completion Checklist

Before starting Level 4, verify:

- [ ] VaultNote is deployed and working
- [ ] Can register and log in
- [ ] Cross-user isolation works (two users can't see each other's notes)
- [ ] Refreshing the page maintains the login
- [ ] Code is pushed to GitHub
- [ ] `JWT_SECRET` is different in production vs development
- [ ] `server/package.json` has all @types in `dependencies`
- [ ] `server/tsconfig.json` has `"types": ["node"]`
- [ ] No CORS errors (no trailing slash in CORS_ORIGIN)
- [ ] You can explain the auth flow without notes

All checked? You're ready for [Level 4 — DataDash](../../level-4-scalable/).

---

| | |
|:---|---:|
| [← Step 7: Deployment](../07-deployment/) | [Level 4 — DataDash →](../../level-4-scalable/) |

`Level 5` **Step 4 of 9** — Backend

# 04 — Backend: Feature Modules

## Spatial Orientation

The backend is organized as **feature modules**. Each module owns its routes and service layer. The server entry point mounts modules like building blocks.

```
server/src/index.ts
  │
  ├── middleware/ (logger, rate limiter, CORS, JSON, error handler)
  │
  ├── modules/auth/       → POST /api/auth/register, POST /api/auth/login, GET /api/auth/me
  ├── modules/boards/     → GET/POST /api/boards, GET/PUT/DELETE /api/boards/:id, members
  ├── modules/lists/      → POST /api/boards/:boardId/lists, PUT/DELETE /api/lists/:id
  ├── modules/cards/      → POST /api/lists/:listId/cards, GET/PUT/DELETE /api/cards/:id
  └── modules/comments/   → POST /api/cards/:cardId/comments, DELETE /api/comments/:id
```

---

## Step 1: Middleware (Carried Forward)

These are the same middleware patterns from Level 4, slightly adapted.

> [!IMPORTANT]
> **You should be in:** `collab-board/`

### Logger

Create `server/src/middleware/logger.ts`:

```typescript
import pino from 'pino';
import pinoHttp from 'pino-http';

export const logger = pino({
  level: process.env.NODE_ENV === 'production' ? 'info' : 'debug',
  transport:
    process.env.NODE_ENV !== 'production'
      ? { target: 'pino-pretty', options: { colorize: true } }
      : undefined,
});

export const httpLogger = pinoHttp({
  logger,
  autoLogging: {
    ignore: (req) => req.url === '/api/health',
  },
});
```

### Rate Limiter

Create `server/src/middleware/rateLimiter.ts`:

```typescript
import rateLimit from 'express-rate-limit';

export const generalLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 200,
  standardHeaders: true,
  legacyHeaders: false,
  message: { error: 'Too many requests. Please try again later.' },
});

export const authLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 20,
  message: { error: 'Too many auth attempts. Please try again later.' },
});
```

### Error Handler

Create `server/src/middleware/errorHandler.ts`:

```typescript
import { Request, Response, NextFunction } from 'express';
import { logger } from './logger';

export function errorHandler(
  err: Error,
  _req: Request,
  res: Response,
  _next: NextFunction
): void {
  logger.error({ err }, 'Unhandled error');

  const statusCode = res.statusCode !== 200 ? res.statusCode : 500;
  res.status(statusCode).json({
    error: process.env.NODE_ENV === 'production'
      ? 'Internal server error'
      : err.message,
  });
}
```

---

## Step 2: Auth Module

Auth is carried from Level 3 but organized as a module.

### Auth Middleware

Create `server/src/modules/auth/auth.middleware.ts`:

```typescript
import { Request, Response, NextFunction } from 'express';
import jwt from 'jsonwebtoken';
import type { JWTPayload } from '../../types/index';

const JWT_SECRET = process.env.JWT_SECRET || 'dev-secret-change-in-production';

// Extend Express Request to include user
declare global {
  namespace Express {
    interface Request {
      user?: JWTPayload;
    }
  }
}

export function authenticate(req: Request, res: Response, next: NextFunction): void {
  const authHeader = req.headers.authorization;

  if (!authHeader || !authHeader.startsWith('Bearer ')) {
    res.status(401).json({ error: 'No token provided' });
    return;
  }

  const token = authHeader.split(' ')[1];

  try {
    const payload = jwt.verify(token, JWT_SECRET) as JWTPayload;
    req.user = payload;
    next();
  } catch {
    res.status(401).json({ error: 'Invalid or expired token' });
  }
}
```

### Auth Service

Create `server/src/modules/auth/auth.service.ts`:

```typescript
import bcrypt from 'bcryptjs';
import jwt from 'jsonwebtoken';
import pool from '../../db/index';
import type { User, SafeUser, JWTPayload } from '../../types/index';

const JWT_SECRET = process.env.JWT_SECRET || 'dev-secret-change-in-production';
const JWT_EXPIRES_IN = process.env.JWT_EXPIRES_IN || '7d';

function toSafeUser(user: User): SafeUser {
  const { password_hash, ...safe } = user;
  return safe;
}

function createToken(user: User): string {
  const payload: JWTPayload = { userId: user.id, email: user.email };
  return jwt.sign(payload, JWT_SECRET, { expiresIn: JWT_EXPIRES_IN });
}

export async function register(
  email: string,
  password: string,
  displayName: string
): Promise<{ user: SafeUser; token: string }> {
  const existing = await pool.query('SELECT id FROM users WHERE email = $1', [
    email.toLowerCase(),
  ]);
  if (existing.rows.length > 0) {
    throw Object.assign(new Error('Email already registered'), { status: 409 });
  }

  const passwordHash = await bcrypt.hash(password, 10);
  const result = await pool.query(
    `INSERT INTO users (email, password_hash, display_name)
     VALUES ($1, $2, $3) RETURNING *`,
    [email.toLowerCase(), passwordHash, displayName]
  );

  const user = result.rows[0];
  return { user: toSafeUser(user), token: createToken(user) };
}

export async function login(
  email: string,
  password: string
): Promise<{ user: SafeUser; token: string }> {
  const result = await pool.query('SELECT * FROM users WHERE email = $1', [
    email.toLowerCase(),
  ]);
  if (result.rows.length === 0) {
    throw Object.assign(new Error('Invalid email or password'), { status: 401 });
  }

  const user = result.rows[0];
  const valid = await bcrypt.compare(password, user.password_hash);
  if (!valid) {
    throw Object.assign(new Error('Invalid email or password'), { status: 401 });
  }

  return { user: toSafeUser(user), token: createToken(user) };
}

export async function getMe(userId: number): Promise<SafeUser> {
  const result = await pool.query(
    'SELECT id, email, display_name, created_at FROM users WHERE id = $1',
    [userId]
  );
  if (result.rows.length === 0) {
    throw Object.assign(new Error('User not found'), { status: 404 });
  }
  return result.rows[0];
}
```

### Auth Routes

Create `server/src/modules/auth/auth.routes.ts`:

```typescript
import { Router, Request, Response, NextFunction } from 'express';
import { authLimiter } from '../../middleware/rateLimiter';
import { authenticate } from './auth.middleware';
import * as authService from './auth.service';

const router = Router();

// POST /api/auth/register
router.post('/register', authLimiter, async (req: Request, res: Response, next: NextFunction) => {
  try {
    const { email, password, displayName } = req.body;

    if (!email || !password || !displayName) {
      res.status(400).json({ error: 'Email, password, and display name are required' });
      return;
    }
    if (password.length < 6) {
      res.status(400).json({ error: 'Password must be at least 6 characters' });
      return;
    }

    const result = await authService.register(email, password, displayName);
    res.status(201).json(result);
  } catch (err: any) {
    if (err.status) res.status(err.status);
    next(err);
  }
});

// POST /api/auth/login
router.post('/login', authLimiter, async (req: Request, res: Response, next: NextFunction) => {
  try {
    const { email, password } = req.body;

    if (!email || !password) {
      res.status(400).json({ error: 'Email and password are required' });
      return;
    }

    const result = await authService.login(email, password);
    res.json(result);
  } catch (err: any) {
    if (err.status) res.status(err.status);
    next(err);
  }
});

// GET /api/auth/me
router.get('/me', authenticate, async (req: Request, res: Response, next: NextFunction) => {
  try {
    const user = await authService.getMe(req.user!.userId);
    res.json(user);
  } catch (err: any) {
    if (err.status) res.status(err.status);
    next(err);
  }
});

export default router;
```

---

## Step 3: Boards Module

### Boards Service

Create `server/src/modules/boards/boards.service.ts`:

```typescript
import pool from '../../db/index';
import type { Board, BoardWithLists, ListWithCards } from '../../types/index';

// Get all boards the user belongs to
export async function getUserBoards(userId: number): Promise<Board[]> {
  const result = await pool.query(
    `SELECT b.*, bm.role AS member_role
     FROM boards b
     JOIN board_members bm ON b.id = bm.board_id
     WHERE bm.user_id = $1
     ORDER BY b.updated_at DESC`,
    [userId]
  );
  return result.rows;
}

// Create a board and add the creator as owner
export async function createBoard(
  name: string,
  description: string | null,
  ownerId: number
): Promise<Board> {
  const client = await pool.connect();
  try {
    await client.query('BEGIN');

    const boardResult = await client.query(
      `INSERT INTO boards (name, description, owner_id)
       VALUES ($1, $2, $3) RETURNING *`,
      [name, description, ownerId]
    );
    const board = boardResult.rows[0];

    // Automatically add creator as owner member
    await client.query(
      `INSERT INTO board_members (board_id, user_id, role) VALUES ($1, $2, 'owner')`,
      [board.id, ownerId]
    );

    await client.query('COMMIT');
    return board;
  } catch (err) {
    await client.query('ROLLBACK');
    throw err;
  } finally {
    client.release();
  }
}

// Get a board with all its lists and cards
export async function getBoardWithLists(
  boardId: number,
  userId: number
): Promise<BoardWithLists> {
  // Verify membership
  const memberCheck = await pool.query(
    'SELECT id FROM board_members WHERE board_id = $1 AND user_id = $2',
    [boardId, userId]
  );
  if (memberCheck.rows.length === 0) {
    throw Object.assign(new Error('Board not found'), { status: 404 });
  }

  // Get board
  const boardResult = await pool.query('SELECT * FROM boards WHERE id = $1', [boardId]);
  if (boardResult.rows.length === 0) {
    throw Object.assign(new Error('Board not found'), { status: 404 });
  }
  const board = boardResult.rows[0];

  // Get lists with cards and assignees
  const listsResult = await pool.query(
    `SELECT l.*,
       COALESCE(
         json_agg(
           json_build_object(
             'id', c.id,
             'list_id', c.list_id,
             'title', c.title,
             'description', c.description,
             'position', c.position,
             'assignee_id', c.assignee_id,
             'created_at', c.created_at,
             'updated_at', c.updated_at,
             'assignee', CASE WHEN u.id IS NOT NULL THEN
               json_build_object('id', u.id, 'email', u.email, 'display_name', u.display_name, 'created_at', u.created_at)
             ELSE NULL END
           ) ORDER BY c.position
         ) FILTER (WHERE c.id IS NOT NULL),
         '[]'
       ) AS cards
     FROM lists l
     LEFT JOIN cards c ON l.id = c.list_id
     LEFT JOIN users u ON c.assignee_id = u.id
     WHERE l.board_id = $1
     GROUP BY l.id
     ORDER BY l.position`,
    [boardId]
  );

  return {
    ...board,
    lists: listsResult.rows,
  };
}

// Update a board (owner only)
export async function updateBoard(
  boardId: number,
  userId: number,
  name: string,
  description: string | null
): Promise<Board> {
  const result = await pool.query(
    `UPDATE boards SET name = $1, description = $2, updated_at = NOW()
     WHERE id = $3 AND owner_id = $4 RETURNING *`,
    [name, description, boardId, userId]
  );
  if (result.rows.length === 0) {
    throw Object.assign(new Error('Board not found or not authorized'), { status: 404 });
  }
  return result.rows[0];
}

// Delete a board (owner only)
export async function deleteBoard(boardId: number, userId: number): Promise<void> {
  const result = await pool.query(
    'DELETE FROM boards WHERE id = $1 AND owner_id = $2 RETURNING id',
    [boardId, userId]
  );
  if (result.rows.length === 0) {
    throw Object.assign(new Error('Board not found or not authorized'), { status: 404 });
  }
}

// Add a member to a board
export async function addMember(
  boardId: number,
  ownerUserId: number,
  memberEmail: string
): Promise<void> {
  // Verify requester is the board owner
  const ownerCheck = await pool.query(
    'SELECT id FROM boards WHERE id = $1 AND owner_id = $2',
    [boardId, ownerUserId]
  );
  if (ownerCheck.rows.length === 0) {
    throw Object.assign(new Error('Not authorized'), { status: 403 });
  }

  // Find user by email
  const userResult = await pool.query('SELECT id FROM users WHERE email = $1', [
    memberEmail.toLowerCase(),
  ]);
  if (userResult.rows.length === 0) {
    throw Object.assign(new Error('User not found'), { status: 404 });
  }

  const memberId = userResult.rows[0].id;

  // Add member (UNIQUE constraint prevents duplicates)
  try {
    await pool.query(
      `INSERT INTO board_members (board_id, user_id, role) VALUES ($1, $2, 'member')`,
      [boardId, memberId]
    );
  } catch (err: any) {
    if (err.code === '23505') {
      throw Object.assign(new Error('User is already a member'), { status: 409 });
    }
    throw err;
  }
}

// Get board members
export async function getMembers(boardId: number, userId: number) {
  // Verify membership
  const memberCheck = await pool.query(
    'SELECT id FROM board_members WHERE board_id = $1 AND user_id = $2',
    [boardId, userId]
  );
  if (memberCheck.rows.length === 0) {
    throw Object.assign(new Error('Not authorized'), { status: 403 });
  }

  const result = await pool.query(
    `SELECT u.id, u.email, u.display_name, u.created_at, bm.role
     FROM board_members bm
     JOIN users u ON bm.user_id = u.id
     WHERE bm.board_id = $1
     ORDER BY bm.role DESC, u.display_name`,
    [boardId]
  );
  return result.rows;
}
```

### Key Pattern: Board Membership Authorization

Every board operation checks that the user is a member:

```
1. User sends request with JWT
2. authenticate middleware extracts userId from token
3. Service checks: "Is this userId in board_members for this board?"
4. If no → 404 (not "403 — Forbidden" — same pattern as Level 3)
5. If yes → proceed with the operation
```

> [!NOTE]
> **Technical:** The `getBoardWithLists` query uses `json_agg` and `json_build_object` to nest cards inside lists in a single SQL query. This is a PostgreSQL-specific feature that avoids the N+1 query problem (1 query for lists, then N queries for each list's cards). One query returns the complete nested structure.
>
> **Plain English:** Instead of asking the database "give me the lists" then "give me the cards for list 1" then "cards for list 2" etc., we ask once: "give me everything, nested." PostgreSQL assembles the nesting for us.

### Boards Routes

Create `server/src/modules/boards/boards.routes.ts`:

```typescript
import { Router, Request, Response, NextFunction } from 'express';
import { authenticate } from '../auth/auth.middleware';
import * as boardsService from './boards.service';

const router = Router();

router.use(authenticate);

// GET /api/boards
router.get('/', async (req: Request, res: Response, next: NextFunction) => {
  try {
    const boards = await boardsService.getUserBoards(req.user!.userId);
    res.json(boards);
  } catch (err: any) {
    if (err.status) res.status(err.status);
    next(err);
  }
});

// POST /api/boards
router.post('/', async (req: Request, res: Response, next: NextFunction) => {
  try {
    const { name, description } = req.body;
    if (!name) {
      res.status(400).json({ error: 'Board name is required' });
      return;
    }
    const board = await boardsService.createBoard(name, description || null, req.user!.userId);
    res.status(201).json(board);
  } catch (err: any) {
    if (err.status) res.status(err.status);
    next(err);
  }
});

// GET /api/boards/:id
router.get('/:id', async (req: Request, res: Response, next: NextFunction) => {
  try {
    const board = await boardsService.getBoardWithLists(
      Number(req.params.id),
      req.user!.userId
    );
    res.json(board);
  } catch (err: any) {
    if (err.status) res.status(err.status);
    next(err);
  }
});

// PUT /api/boards/:id
router.put('/:id', async (req: Request, res: Response, next: NextFunction) => {
  try {
    const { name, description } = req.body;
    const board = await boardsService.updateBoard(
      Number(req.params.id),
      req.user!.userId,
      name,
      description
    );
    res.json(board);
  } catch (err: any) {
    if (err.status) res.status(err.status);
    next(err);
  }
});

// DELETE /api/boards/:id
router.delete('/:id', async (req: Request, res: Response, next: NextFunction) => {
  try {
    await boardsService.deleteBoard(Number(req.params.id), req.user!.userId);
    res.status(204).send();
  } catch (err: any) {
    if (err.status) res.status(err.status);
    next(err);
  }
});

// POST /api/boards/:id/members
router.post('/:id/members', async (req: Request, res: Response, next: NextFunction) => {
  try {
    const { email } = req.body;
    if (!email) {
      res.status(400).json({ error: 'Member email is required' });
      return;
    }
    await boardsService.addMember(Number(req.params.id), req.user!.userId, email);
    res.status(201).json({ message: 'Member added' });
  } catch (err: any) {
    if (err.status) res.status(err.status);
    next(err);
  }
});

// GET /api/boards/:id/members
router.get('/:id/members', async (req: Request, res: Response, next: NextFunction) => {
  try {
    const members = await boardsService.getMembers(Number(req.params.id), req.user!.userId);
    res.json(members);
  } catch (err: any) {
    if (err.status) res.status(err.status);
    next(err);
  }
});

export default router;
```

---

## Step 4: Lists Module

Create `server/src/modules/lists/lists.service.ts`:

```typescript
import pool from '../../db/index';
import type { List } from '../../types/index';

export async function createList(boardId: number, userId: number, name: string): Promise<List> {
  // Verify membership
  const check = await pool.query(
    'SELECT id FROM board_members WHERE board_id = $1 AND user_id = $2',
    [boardId, userId]
  );
  if (check.rows.length === 0) {
    throw Object.assign(new Error('Not authorized'), { status: 403 });
  }

  // Get next position
  const posResult = await pool.query(
    'SELECT COALESCE(MAX(position), -1) + 1 AS next_pos FROM lists WHERE board_id = $1',
    [boardId]
  );
  const position = posResult.rows[0].next_pos;

  const result = await pool.query(
    'INSERT INTO lists (board_id, name, position) VALUES ($1, $2, $3) RETURNING *',
    [boardId, name, position]
  );
  return result.rows[0];
}

export async function updateList(listId: number, userId: number, name: string): Promise<List> {
  // Verify membership via list → board → board_members
  const check = await pool.query(
    `SELECT l.id FROM lists l
     JOIN board_members bm ON l.board_id = bm.board_id
     WHERE l.id = $1 AND bm.user_id = $2`,
    [listId, userId]
  );
  if (check.rows.length === 0) {
    throw Object.assign(new Error('List not found'), { status: 404 });
  }

  const result = await pool.query(
    'UPDATE lists SET name = $1 WHERE id = $2 RETURNING *',
    [name, listId]
  );
  return result.rows[0];
}

export async function deleteList(listId: number, userId: number): Promise<void> {
  const check = await pool.query(
    `SELECT l.id FROM lists l
     JOIN board_members bm ON l.board_id = bm.board_id
     WHERE l.id = $1 AND bm.user_id = $2`,
    [listId, userId]
  );
  if (check.rows.length === 0) {
    throw Object.assign(new Error('List not found'), { status: 404 });
  }

  await pool.query('DELETE FROM lists WHERE id = $1', [listId]);
}
```

Create `server/src/modules/lists/lists.routes.ts`:

```typescript
import { Router, Request, Response, NextFunction } from 'express';
import { authenticate } from '../auth/auth.middleware';
import * as listsService from './lists.service';

const router = Router();

router.use(authenticate);

// POST /api/boards/:boardId/lists
router.post('/boards/:boardId/lists', async (req: Request, res: Response, next: NextFunction) => {
  try {
    const { name } = req.body;
    if (!name) {
      res.status(400).json({ error: 'List name is required' });
      return;
    }
    const list = await listsService.createList(
      Number(req.params.boardId), req.user!.userId, name
    );
    res.status(201).json(list);
  } catch (err: any) {
    if (err.status) res.status(err.status);
    next(err);
  }
});

// PUT /api/lists/:id
router.put('/lists/:id', async (req: Request, res: Response, next: NextFunction) => {
  try {
    const { name } = req.body;
    const list = await listsService.updateList(Number(req.params.id), req.user!.userId, name);
    res.json(list);
  } catch (err: any) {
    if (err.status) res.status(err.status);
    next(err);
  }
});

// DELETE /api/lists/:id
router.delete('/lists/:id', async (req: Request, res: Response, next: NextFunction) => {
  try {
    await listsService.deleteList(Number(req.params.id), req.user!.userId);
    res.status(204).send();
  } catch (err: any) {
    if (err.status) res.status(err.status);
    next(err);
  }
});

export default router;
```

---

## Step 5: Cards Module

Create `server/src/modules/cards/cards.service.ts`:

```typescript
import pool from '../../db/index';
import type { Card, CardDetail } from '../../types/index';

export async function createCard(
  listId: number, userId: number, title: string, description?: string
): Promise<Card> {
  // Verify membership via list → board → board_members
  const check = await pool.query(
    `SELECT l.id FROM lists l
     JOIN board_members bm ON l.board_id = bm.board_id
     WHERE l.id = $1 AND bm.user_id = $2`,
    [listId, userId]
  );
  if (check.rows.length === 0) {
    throw Object.assign(new Error('Not authorized'), { status: 403 });
  }

  const posResult = await pool.query(
    'SELECT COALESCE(MAX(position), -1) + 1 AS next_pos FROM cards WHERE list_id = $1',
    [listId]
  );

  const result = await pool.query(
    `INSERT INTO cards (list_id, title, description, position)
     VALUES ($1, $2, $3, $4) RETURNING *`,
    [listId, title, description || null, posResult.rows[0].next_pos]
  );
  return result.rows[0];
}

export async function getCardDetail(cardId: number, userId: number): Promise<CardDetail> {
  // Verify membership via card → list → board → board_members
  const check = await pool.query(
    `SELECT c.id FROM cards c
     JOIN lists l ON c.list_id = l.id
     JOIN board_members bm ON l.board_id = bm.board_id
     WHERE c.id = $1 AND bm.user_id = $2`,
    [cardId, userId]
  );
  if (check.rows.length === 0) {
    throw Object.assign(new Error('Card not found'), { status: 404 });
  }

  const cardResult = await pool.query(
    `SELECT c.*,
       CASE WHEN u.id IS NOT NULL THEN
         json_build_object('id', u.id, 'email', u.email, 'display_name', u.display_name, 'created_at', u.created_at)
       ELSE NULL END AS assignee
     FROM cards c
     LEFT JOIN users u ON c.assignee_id = u.id
     WHERE c.id = $1`,
    [cardId]
  );

  const commentsResult = await pool.query(
    `SELECT cm.*,
       json_build_object('id', u.id, 'email', u.email, 'display_name', u.display_name, 'created_at', u.created_at) AS user
     FROM comments cm
     JOIN users u ON cm.user_id = u.id
     WHERE cm.card_id = $1
     ORDER BY cm.created_at ASC`,
    [cardId]
  );

  return {
    ...cardResult.rows[0],
    comments: commentsResult.rows,
  };
}

export async function updateCard(
  cardId: number, userId: number, updates: Partial<Card>
): Promise<Card> {
  const check = await pool.query(
    `SELECT c.id FROM cards c
     JOIN lists l ON c.list_id = l.id
     JOIN board_members bm ON l.board_id = bm.board_id
     WHERE c.id = $1 AND bm.user_id = $2`,
    [cardId, userId]
  );
  if (check.rows.length === 0) {
    throw Object.assign(new Error('Card not found'), { status: 404 });
  }

  const fields: string[] = [];
  const values: any[] = [];
  let paramIndex = 1;

  if (updates.title !== undefined) {
    fields.push(`title = $${paramIndex++}`);
    values.push(updates.title);
  }
  if (updates.description !== undefined) {
    fields.push(`description = $${paramIndex++}`);
    values.push(updates.description);
  }
  if (updates.assignee_id !== undefined) {
    fields.push(`assignee_id = $${paramIndex++}`);
    values.push(updates.assignee_id);
  }

  fields.push(`updated_at = NOW()`);
  values.push(cardId);

  const result = await pool.query(
    `UPDATE cards SET ${fields.join(', ')} WHERE id = $${paramIndex} RETURNING *`,
    values
  );
  return result.rows[0];
}

export async function moveCard(
  cardId: number, userId: number, targetListId: number, targetPosition: number
): Promise<Card> {
  const check = await pool.query(
    `SELECT c.id, c.list_id, c.position FROM cards c
     JOIN lists l ON c.list_id = l.id
     JOIN board_members bm ON l.board_id = bm.board_id
     WHERE c.id = $1 AND bm.user_id = $2`,
    [cardId, userId]
  );
  if (check.rows.length === 0) {
    throw Object.assign(new Error('Card not found'), { status: 404 });
  }

  const client = await pool.connect();
  try {
    await client.query('BEGIN');

    const card = check.rows[0];
    const sourceListId = card.list_id;

    // Close gap in source list
    await client.query(
      `UPDATE cards SET position = position - 1
       WHERE list_id = $1 AND position > $2`,
      [sourceListId, card.position]
    );

    // Make room in target list
    await client.query(
      `UPDATE cards SET position = position + 1
       WHERE list_id = $1 AND position >= $2`,
      [targetListId, targetPosition]
    );

    // Move the card
    const result = await client.query(
      `UPDATE cards SET list_id = $1, position = $2, updated_at = NOW()
       WHERE id = $3 RETURNING *`,
      [targetListId, targetPosition, cardId]
    );

    await client.query('COMMIT');
    return result.rows[0];
  } catch (err) {
    await client.query('ROLLBACK');
    throw err;
  } finally {
    client.release();
  }
}

export async function deleteCard(cardId: number, userId: number): Promise<void> {
  const check = await pool.query(
    `SELECT c.id FROM cards c
     JOIN lists l ON c.list_id = l.id
     JOIN board_members bm ON l.board_id = bm.board_id
     WHERE c.id = $1 AND bm.user_id = $2`,
    [cardId, userId]
  );
  if (check.rows.length === 0) {
    throw Object.assign(new Error('Card not found'), { status: 404 });
  }

  await pool.query('DELETE FROM cards WHERE id = $1', [cardId]);
}
```

> [!WARNING]
> **The `moveCard` function uses a database transaction** (`BEGIN` / `COMMIT` / `ROLLBACK`). Moving a card requires three updates: close the gap in the source list, make room in the target list, then move the card. If any step fails, the transaction rolls back all changes — you never end up with broken positions.

Create `server/src/modules/cards/cards.routes.ts`:

```typescript
import { Router, Request, Response, NextFunction } from 'express';
import { authenticate } from '../auth/auth.middleware';
import * as cardsService from './cards.service';

const router = Router();

router.use(authenticate);

// POST /api/lists/:listId/cards
router.post('/lists/:listId/cards', async (req: Request, res: Response, next: NextFunction) => {
  try {
    const { title, description } = req.body;
    if (!title) {
      res.status(400).json({ error: 'Card title is required' });
      return;
    }
    const card = await cardsService.createCard(
      Number(req.params.listId), req.user!.userId, title, description
    );
    res.status(201).json(card);
  } catch (err: any) {
    if (err.status) res.status(err.status);
    next(err);
  }
});

// GET /api/cards/:id
router.get('/cards/:id', async (req: Request, res: Response, next: NextFunction) => {
  try {
    const card = await cardsService.getCardDetail(Number(req.params.id), req.user!.userId);
    res.json(card);
  } catch (err: any) {
    if (err.status) res.status(err.status);
    next(err);
  }
});

// PUT /api/cards/:id
router.put('/cards/:id', async (req: Request, res: Response, next: NextFunction) => {
  try {
    const card = await cardsService.updateCard(
      Number(req.params.id), req.user!.userId, req.body
    );
    res.json(card);
  } catch (err: any) {
    if (err.status) res.status(err.status);
    next(err);
  }
});

// PUT /api/cards/:id/move
router.put('/cards/:id/move', async (req: Request, res: Response, next: NextFunction) => {
  try {
    const { targetListId, targetPosition } = req.body;
    if (targetListId === undefined || targetPosition === undefined) {
      res.status(400).json({ error: 'targetListId and targetPosition are required' });
      return;
    }
    const card = await cardsService.moveCard(
      Number(req.params.id), req.user!.userId, targetListId, targetPosition
    );
    res.json(card);
  } catch (err: any) {
    if (err.status) res.status(err.status);
    next(err);
  }
});

// DELETE /api/cards/:id
router.delete('/cards/:id', async (req: Request, res: Response, next: NextFunction) => {
  try {
    await cardsService.deleteCard(Number(req.params.id), req.user!.userId);
    res.status(204).send();
  } catch (err: any) {
    if (err.status) res.status(err.status);
    next(err);
  }
});

export default router;
```

---

## Step 6: Comments Module

Create `server/src/modules/comments/comments.service.ts`:

```typescript
import pool from '../../db/index';
import type { CommentWithUser } from '../../types/index';

export async function addComment(
  cardId: number, userId: number, content: string
): Promise<CommentWithUser> {
  // Verify membership via card → list → board → board_members
  const check = await pool.query(
    `SELECT c.id FROM cards c
     JOIN lists l ON c.list_id = l.id
     JOIN board_members bm ON l.board_id = bm.board_id
     WHERE c.id = $1 AND bm.user_id = $2`,
    [cardId, userId]
  );
  if (check.rows.length === 0) {
    throw Object.assign(new Error('Card not found'), { status: 404 });
  }

  const result = await pool.query(
    `INSERT INTO comments (card_id, user_id, content) VALUES ($1, $2, $3) RETURNING *`,
    [cardId, userId, content]
  );

  const comment = result.rows[0];

  const userResult = await pool.query(
    'SELECT id, email, display_name, created_at FROM users WHERE id = $1',
    [userId]
  );

  return { ...comment, user: userResult.rows[0] };
}

export async function deleteComment(commentId: number, userId: number): Promise<void> {
  const result = await pool.query(
    'DELETE FROM comments WHERE id = $1 AND user_id = $2 RETURNING id',
    [commentId, userId]
  );
  if (result.rows.length === 0) {
    throw Object.assign(new Error('Comment not found'), { status: 404 });
  }
}
```

Create `server/src/modules/comments/comments.routes.ts`:

```typescript
import { Router, Request, Response, NextFunction } from 'express';
import { authenticate } from '../auth/auth.middleware';
import * as commentsService from './comments.service';

const router = Router();

router.use(authenticate);

// POST /api/cards/:cardId/comments
router.post('/cards/:cardId/comments', async (req: Request, res: Response, next: NextFunction) => {
  try {
    const { content } = req.body;
    if (!content || !content.trim()) {
      res.status(400).json({ error: 'Comment content is required' });
      return;
    }
    const comment = await commentsService.addComment(
      Number(req.params.cardId), req.user!.userId, content.trim()
    );
    res.status(201).json(comment);
  } catch (err: any) {
    if (err.status) res.status(err.status);
    next(err);
  }
});

// DELETE /api/comments/:id
router.delete('/comments/:id', async (req: Request, res: Response, next: NextFunction) => {
  try {
    await commentsService.deleteComment(Number(req.params.id), req.user!.userId);
    res.status(204).send();
  } catch (err: any) {
    if (err.status) res.status(err.status);
    next(err);
  }
});

export default router;
```

---

## Step 7: Server Entry Point

Create `server/src/index.ts`:

```typescript
import express from 'express';
import cors from 'cors';
import dotenv from 'dotenv';
import { httpLogger, logger } from './middleware/logger';
import { generalLimiter } from './middleware/rateLimiter';
import { errorHandler } from './middleware/errorHandler';
import authRoutes from './modules/auth/auth.routes';
import boardsRoutes from './modules/boards/boards.routes';
import listsRoutes from './modules/lists/lists.routes';
import cardsRoutes from './modules/cards/cards.routes';
import commentsRoutes from './modules/comments/comments.routes';
import pool from './db/index';

dotenv.config({ path: '../.env' });

const app = express();
const PORT = process.env.PORT || 3001;

// --- Middleware stack ---
app.use(httpLogger);
app.use(generalLimiter);
app.use(cors({ origin: process.env.CORS_ORIGIN }));
app.use(express.json());

// --- Mount feature modules ---
app.use('/api/auth', authRoutes);
app.use('/api', boardsRoutes);         // /api/boards, /api/boards/:id, etc.
app.use('/api', listsRoutes);          // /api/boards/:boardId/lists, /api/lists/:id
app.use('/api', cardsRoutes);          // /api/lists/:listId/cards, /api/cards/:id
app.use('/api', commentsRoutes);       // /api/cards/:cardId/comments, /api/comments/:id

// --- Health check ---
app.get('/api/health', async (_req, res) => {
  try {
    const result = await pool.query('SELECT NOW()');
    res.json({ status: 'ok', database: 'connected', timestamp: result.rows[0].now });
  } catch {
    res.status(503).json({ status: 'error', database: 'disconnected' });
  }
});

// --- Error handler (must be last) ---
app.use(errorHandler);

// --- Start server ---
if (process.env.NODE_ENV !== 'test') {
  app.listen(PORT, () => {
    logger.info(`Server running on http://localhost:${PORT}`);
  });
}

export default app;
```

Notice how clean the entry point is — each module is mounted with one line. The modules handle their own route prefixes internally.

---

## Step 8: Test the API

```bash
cd server && npm run dev
```

### Register and login:

```bash
# Register
curl -X POST http://localhost:3001/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"alex@example.com","password":"password123"}'
```

Copy the token from the response. Use it in subsequent requests:

```bash
TOKEN="your-token-here"

# Get boards
curl -H "Authorization: Bearer $TOKEN" http://localhost:3001/api/boards | jq

# Get board detail with lists and cards
curl -H "Authorization: Bearer $TOKEN" http://localhost:3001/api/boards/1 | jq

# Create a card
curl -X POST http://localhost:3001/api/lists/1/cards \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"title":"New task","description":"Created via curl"}' | jq

# Move a card
curl -X PUT http://localhost:3001/api/cards/1/move \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"targetListId":2,"targetPosition":0}' | jq

# Add a comment
curl -X POST http://localhost:3001/api/cards/1/comments \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"content":"Great progress on this!"}' | jq
```

---

## Step 9: Commit

> [!IMPORTANT]
> **You should be in:** `collab-board/`

```bash
cd ..
git add .
git commit -m "feat: add modular backend with auth, boards, lists, cards, and comments"
```

---

> [!TIP]
> ## Spatial Check-In

1. **Why does `moveCard` use a database transaction?**

<details><summary>Answer</summary>

Moving a card requires three UPDATE queries: close the gap in the source list, make room in the target list, and move the card itself. If the second query fails, the first has already shifted positions — leaving the data inconsistent. A transaction ensures all three succeed together or none of them execute, keeping positions consistent.

</details>

2. **Why does every card/list/comment operation check board membership?**

<details><summary>Answer</summary>

Board membership is the authorization boundary. A valid JWT proves who you are (authentication), but membership proves you have access to this board (authorization). Without the membership check, any authenticated user could read or modify any board's data by guessing IDs. The check `card → list → board → board_members WHERE user_id = $1` traces the authorization chain from the resource to the user.

</details>

3. **What's the benefit of `json_agg` over making separate queries for lists and cards?**

<details><summary>Answer</summary>

Without `json_agg`, loading a board with 4 lists and 20 cards would take 5 queries (1 for lists + 4 for each list's cards). With `json_agg`, it takes 1 query — PostgreSQL nests the cards into each list row as a JSON array. This eliminates the N+1 query problem and reduces round trips between the app server and database.

</details>

---

| | | |
|:---|:---:|---:|
| [← 03 — Database](../03-database/) | [Level 5 Overview](../) | [05 — Frontend →](../05-frontend/) |

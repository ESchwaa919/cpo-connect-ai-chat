# CPO Connect Members Area Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a members-only area with magic link authentication, chat insights, and member directory to the CPO Connect Hub.

**Architecture:** Express server wraps the Vite-built React SPA, serving static files from `dist/` and API routes under `/api/`. Auth uses magic link emails (Resend) verified against a Google Sheet membership list, with tokens and sessions stored in PostgreSQL (`cpo_connect` schema on shared `truth_tone` instance). The frontend adds protected routes under `/members/` for chat insights and a searchable member directory.

**Tech Stack:** Express, pg, cookie-parser, cookie-signature, Resend, Google Sheets API v4, React Router, TanStack Query, Recharts, shadcn/ui

**Spec:** `docs/superpowers/specs/2026-03-14-members-area-design.md`

---

## File Map

### Server (new files — all ES modules, plain JS)

| File | Responsibility |
|------|---------------|
| `server.js` | Express entry point — mounts API routes, serves `dist/` static build, SPA catch-all |
| `server/db.js` | PostgreSQL pool (pg) with SSL config for Render internal network |
| `server/migrations/001-init.sql` | Schema + table DDL for `cpo_connect` schema |
| `server/services/sheets.js` | Google Sheets API v4 — `lookupMember(email)`, `getDirectory()` with 5-min cache |
| `server/services/email.js` | Resend — `sendMagicLink(email, token, name)` |
| `server/services/rate-limit.js` | In-memory rate limiter factory — `createRateLimiter(opts)` |
| `server/middleware/auth.js` | Session cookie validation — `requireAuth(req, res, next)` |
| `server/routes/auth.js` | Auth endpoints: request, verify, me, logout |
| `server/routes/members.js` | `GET /api/members/directory` |

### Frontend (new files)

| File | Responsibility |
|------|---------------|
| `src/contexts/AuthContext.tsx` | Auth state provider + `useAuth()` hook wrapping `/api/auth/me` |
| `src/components/LoginModal.tsx` | Email input modal → magic link request flow |
| `src/components/ProtectedRoute.tsx` | Route guard — redirects to login modal if unauthenticated |
| `src/components/members/MembersLayout.tsx` | Shared layout for `/members/*` routes (nav, content area) |
| `src/pages/members/ChatInsights.tsx` | Month selector + renders month-specific insight component |
| `src/pages/members/Directory.tsx` | Searchable/filterable member card grid |
| `src/components/members/insights/StatCard.tsx` | Stat card (number + label + gradient) |
| `src/components/members/insights/TrendItem.tsx` | Trend list item with tags |
| `src/components/members/insights/DailyVolumeChart.tsx` | Recharts bar chart — daily message volume |
| `src/components/members/insights/ContributorsChart.tsx` | Recharts pie chart — top contributors |
| `src/components/members/insights/SentimentChart.tsx` | Recharts radar chart — sentiment snapshot |
| `src/components/members/insights/ChannelSection.tsx` | Container for one channel's charts + trends |
| `src/components/members/directory/MemberCard.tsx` | Directory card — avatar, name, role, tags, LinkedIn |
| `src/data/insights/config.ts` | Month→component registry for lazy loading |
| `src/data/insights/jan-2026.tsx` | January 2026 AI channel analysis data + component |
| `src/data/insights/feb-2026.tsx` | February 2026 multi-channel analysis data + component |

### Frontend (modified files)

| File | Change |
|------|--------|
| `src/App.tsx` | Add AuthProvider, members routes, login modal trigger |
| `src/components/Navbar.tsx` | Auth-aware: Login/Apply buttons when logged out, members nav + avatar when logged in |
| `vite.config.ts` | Add `/api` proxy to `http://localhost:3001` for development |
| `package.json` | Add server deps, `dev:server` and `dev:all` scripts |

### Config

| File | Purpose |
|------|---------|
| `.env.example` | Documents all required environment variables |

---

## Chunk 1: Server Foundation

### Task 1: Install server dependencies and add scripts

**Files:**
- Modify: `cpo-connect-hub/package.json`

- [ ] **Step 1: Install production dependencies**

```bash
cd cpo-connect-hub
npm install express pg cookie-signature resend googleapis crypto
```

Note: `crypto` is a Node built-in but we import it as an ES module. No install needed for it — remove from the command. Final:

```bash
npm install express pg cookie-parser cookie-signature resend googleapis
```

- [ ] **Step 2: Install dev dependency for concurrent servers**

```bash
npm install -D concurrently
```

- [ ] **Step 3: Add server scripts to package.json**

Add these scripts alongside existing ones:

```json
"dev:server": "node --watch server.js",
"dev:all": "concurrently \"npm run dev\" \"npm run dev:server\"",
"start": "node server.js"
```

- [ ] **Step 4: Commit**

```bash
git add package.json package-lock.json
git commit -m "feat: add server dependencies and dev scripts"
```

---

### Task 2: Database pool and migration

**Files:**
- Create: `cpo-connect-hub/server/db.js`
- Create: `cpo-connect-hub/server/migrations/001-init.sql`

- [ ] **Step 1: Create server directory structure**

```bash
mkdir -p server/migrations server/services server/routes server/middleware
```

- [ ] **Step 2: Create database pool**

Create `server/db.js`:

```js
import pg from 'pg';
const { Pool } = pg;

const connectionString = process.env.DATABASE_URL?.replace(/\?sslmode=[^&]*/, '');

const pool = new Pool({
  connectionString,
  ssl: { rejectUnauthorized: false },
});

export default pool;
```

Key details:
- Strip `sslmode=` param from URL (Render adds it but pg module handles SSL separately)
- `rejectUnauthorized: false` is safe on Render's internal network

- [ ] **Step 3: Create migration SQL**

Create `server/migrations/001-init.sql`:

```sql
CREATE SCHEMA IF NOT EXISTS cpo_connect;

CREATE TABLE IF NOT EXISTS cpo_connect.magic_link_tokens (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email TEXT NOT NULL,
  token TEXT UNIQUE NOT NULL,
  expires_at TIMESTAMPTZ NOT NULL,
  used BOOLEAN DEFAULT FALSE,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE IF NOT EXISTS cpo_connect.sessions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email TEXT NOT NULL,
  name TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  expires_at TIMESTAMPTZ NOT NULL
);

CREATE INDEX IF NOT EXISTS idx_tokens_token ON cpo_connect.magic_link_tokens(token);
CREATE INDEX IF NOT EXISTS idx_tokens_email ON cpo_connect.magic_link_tokens(email);
CREATE INDEX IF NOT EXISTS idx_sessions_id ON cpo_connect.sessions(id);
```

- [ ] **Step 4: Commit**

```bash
git add server/
git commit -m "feat: add database pool and schema migration"
```

---

### Task 3: Express server entry point

**Files:**
- Create: `cpo-connect-hub/server.js`

- [ ] **Step 1: Create server.js**

```js
import express from 'express';
import cookieParser from 'cookie-parser';
import path from 'path';
import { fileURLToPath } from 'url';

const __dirname = path.dirname(fileURLToPath(import.meta.url));
const app = express();
const PORT = process.env.PORT || 3001;

app.use(express.json());
app.use(cookieParser());

// API routes (mounted in Task 9 and 10)
// app.use('/api/auth', authRouter);
// app.use('/api/members', membersRouter);

// Serve static files from Vite build
app.use(express.static(path.join(__dirname, 'dist')));

// SPA catch-all — serves index.html for all non-API routes
app.get('*', (req, res) => {
  res.sendFile(path.join(__dirname, 'dist', 'index.html'));
});

app.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});

export default app;
```

- [ ] **Step 2: Test — verify server starts**

```bash
npm run build && node server.js
```

Open `http://localhost:3001` — should serve the landing page. Kill the server.

- [ ] **Step 3: Commit**

```bash
git add server.js
git commit -m "feat: add Express server entry point serving Vite build"
```

---

### Task 4: Vite proxy and environment config

**Files:**
- Modify: `cpo-connect-hub/vite.config.ts`
- Create: `cpo-connect-hub/.env.example`

- [ ] **Step 1: Add API proxy to vite.config.ts**

Add `proxy` config inside the existing `server` block:

```ts
server: {
  host: "::",
  port: 8080,
  hmr: {
    overlay: false,
  },
  proxy: {
    '/api': {
      target: 'http://localhost:3001',
      changeOrigin: true,
    },
  },
},
```

- [ ] **Step 2: Create .env.example**

```env
# PostgreSQL — Render internal hostname (Frankfurt)
DATABASE_URL=postgresql://agency:PASSWORD@dpg-d5bgrrp5pdvs73bjmq20-a/truth_tone

# Resend (email)
RESEND_API_KEY=re_xxxxx
RESEND_FROM_EMAIL=noreply@theaiexpert.ai

# Google Sheets
GOOGLE_SHEETS_CREDENTIALS=BASE64_ENCODED_SERVICE_ACCOUNT_JSON
GOOGLE_SHEET_ID=14DZ6Zp1UHg688FPTFdbJLCY6AXhgnAcnodV5KjVCAMs

# Auth
SESSION_SECRET=random-secret-at-least-32-chars
MAGIC_LINK_BASE_URL=https://cpoconnect.com/api/auth/verify
```

- [ ] **Step 3: Add .env to .gitignore**

Append these lines to the existing `.gitignore`:

```
.env
.env.local
```

- [ ] **Step 4: Test — verify proxy works**

Run `npm run dev:all`. In a separate terminal:

```bash
curl http://localhost:8080/api/test
```

Should proxy to port 3001 (will 404 since no routes yet — that's expected). Verify no CORS errors in Vite console.

- [ ] **Step 5: Commit**

```bash
git add vite.config.ts .env.example .gitignore
git commit -m "feat: add Vite API proxy and environment config"
```

---

## Chunk 2: Auth Services

### Task 5: Google Sheets service

**Files:**
- Create: `cpo-connect-hub/server/services/sheets.js`

- [ ] **Step 1: Create sheets service**

```js
import { google } from 'googleapis';

const SHEET_ID = process.env.GOOGLE_SHEET_ID;
let authClient;

async function getAuthClient() {
  if (!authClient) {
    const credentials = JSON.parse(
      Buffer.from(process.env.GOOGLE_SHEETS_CREDENTIALS, 'base64').toString()
    );
    const auth = new google.auth.GoogleAuth({
      credentials,
      scopes: ['https://www.googleapis.com/auth/spreadsheets.readonly'],
    });
    authClient = await auth.getClient();
  }
  return authClient;
}

// Membership lookup cache — 5 minute TTL
let memberCache = { data: null, timestamp: 0 };
const MEMBER_CACHE_TTL = 5 * 60 * 1000;

async function getMemberRows() {
  const now = Date.now();
  if (memberCache.data && now - memberCache.timestamp < MEMBER_CACHE_TTL) {
    return memberCache.data;
  }

  const client = await getAuthClient();
  const sheets = google.sheets({ version: 'v4', auth: client });

  const res = await sheets.spreadsheets.values.get({
    spreadsheetId: SHEET_ID,
    range: 'Sheet1',
  });

  memberCache = { data: res.data.values, timestamp: now };
  return res.data.values;
}

/**
 * Look up a member by email in Sheet1.
 * Uses header names (Email, Status, Full Name) — column position doesn't matter.
 * Returns { email, name, status, jobRole } or null.
 */
export async function lookupMember(email) {
  const rows = await getMemberRows();
  if (!rows || rows.length === 0) return null;

  const headers = rows[0];
  const emailCol = headers.indexOf('Email');
  const statusCol = headers.indexOf('Status');
  const nameCol = headers.indexOf('Full Name');
  const jobRoleCol = headers.indexOf('Job Title');

  if (emailCol === -1 || statusCol === -1) return null;

  const normalizedEmail = email.toLowerCase();

  for (let i = 1; i < rows.length; i++) {
    if (rows[i][emailCol]?.toLowerCase() === normalizedEmail) {
      return {
        email: normalizedEmail,
        name: rows[i][nameCol] || '',
        status: rows[i][statusCol] || '',
        jobRole: jobRoleCol !== -1 ? (rows[i][jobRoleCol] || '') : '',
      };
    }
  }
  return null;
}

// Directory cache — 5 minute TTL, cleared on restart
let directoryCache = { data: null, timestamp: 0 };
const CACHE_TTL = 5 * 60 * 1000;

/**
 * Read PublicMemberDirectoryMVP tab, return array of member objects.
 * Cached in memory for 5 minutes.
 */
export async function getDirectory() {
  const now = Date.now();
  if (directoryCache.data && now - directoryCache.timestamp < CACHE_TTL) {
    return directoryCache.data;
  }

  const client = await getAuthClient();
  const sheets = google.sheets({ version: 'v4', auth: client });

  const res = await sheets.spreadsheets.values.get({
    spreadsheetId: SHEET_ID,
    range: 'PublicMemberDirectoryMVP',
  });

  const rows = res.data.values;
  if (!rows || rows.length === 0) return [];

  const headers = rows[0];
  const data = rows.slice(1).map(row => {
    const obj = {};
    headers.forEach((header, i) => {
      obj[header] = row[i] || '';
    });
    return obj;
  });

  directoryCache = { data, timestamp: now };
  return data;
}
```

- [ ] **Step 2: Commit**

```bash
git add server/services/sheets.js
git commit -m "feat: add Google Sheets service for member lookup and directory"
```

---

### Task 6: Email service (Resend)

**Files:**
- Create: `cpo-connect-hub/server/services/email.js`

- [ ] **Step 1: Create email service**

```js
import { Resend } from 'resend';

const resend = new Resend(process.env.RESEND_API_KEY);
const FROM_EMAIL = process.env.RESEND_FROM_EMAIL || 'noreply@theaiexpert.ai';

/**
 * Send a magic link email to the given address.
 */
export async function sendMagicLink({ email, token, name }) {
  const baseUrl = process.env.MAGIC_LINK_BASE_URL;
  const magicLink = `${baseUrl}?token=${token}`;
  const firstName = name ? name.split(' ')[0] : '';
  const greeting = firstName ? `Hi ${firstName}` : 'Hi';

  await resend.emails.send({
    from: `CPO Connect <${FROM_EMAIL}>`,
    to: email,
    subject: 'Your CPO Connect login link',
    html: `
      <div style="font-family: 'Inter', -apple-system, sans-serif; max-width: 480px; margin: 0 auto; padding: 40px 20px;">
        <h2 style="color: #1a1a2e; margin-bottom: 24px;">${greeting},</h2>
        <p style="color: #4a4a5a; font-size: 16px; line-height: 1.6;">
          Click the button below to sign in to your CPO Connect members area. This link expires in 15 minutes.
        </p>
        <div style="text-align: center; margin: 32px 0;">
          <a href="${magicLink}"
             style="background: linear-gradient(135deg, #3b82f6, #8b5cf6); color: white; padding: 14px 32px; border-radius: 8px; text-decoration: none; font-weight: 600; font-size: 16px; display: inline-block;">
            Sign in to CPO Connect
          </a>
        </div>
        <p style="color: #9ca3af; font-size: 13px; line-height: 1.5;">
          If you didn't request this link, you can safely ignore this email.<br/>
          <a href="${magicLink}" style="color: #6b7280; word-break: break-all;">${magicLink}</a>
        </p>
      </div>
    `,
  });
}
```

- [ ] **Step 2: Commit**

```bash
git add server/services/email.js
git commit -m "feat: add Resend email service for magic links"
```

---

### Task 7: Rate limiter

**Files:**
- Create: `cpo-connect-hub/server/services/rate-limit.js`
- Create: `cpo-connect-hub/src/test/rate-limit.test.ts`

- [ ] **Step 1: Write the rate limiter test**

Create `src/test/rate-limit.test.ts`:

```ts
import { describe, it, expect, beforeEach, vi } from 'vitest';

// We'll test the rate limiter logic directly
// Since it's a JS module, import it
describe('createRateLimiter', () => {
  let createRateLimiter: (opts: { windowMs: number; max: number }) => {
    check: (key: string) => { allowed: boolean; remaining: number };
    reset: (key: string) => void;
  };

  beforeEach(async () => {
    vi.useFakeTimers();
    // Dynamic import to get fresh module state
    const mod = await import('../../server/services/rate-limit.js');
    createRateLimiter = mod.createRateLimiter;
  });

  it('allows requests under the limit', () => {
    const limiter = createRateLimiter({ windowMs: 60000, max: 3 });
    expect(limiter.check('user1').allowed).toBe(true);
    expect(limiter.check('user1').allowed).toBe(true);
    expect(limiter.check('user1').allowed).toBe(true);
  });

  it('blocks requests over the limit', () => {
    const limiter = createRateLimiter({ windowMs: 60000, max: 2 });
    expect(limiter.check('user1').allowed).toBe(true);  // 1st
    expect(limiter.check('user1').allowed).toBe(true);  // 2nd (hits max)
    expect(limiter.check('user1').allowed).toBe(false); // 3rd (over limit)
  });

  it('tracks keys independently', () => {
    const limiter = createRateLimiter({ windowMs: 60000, max: 1 });
    expect(limiter.check('user1').allowed).toBe(true);
    expect(limiter.check('user2').allowed).toBe(true);
    expect(limiter.check('user1').allowed).toBe(false);
  });

  it('resets after window expires', () => {
    const limiter = createRateLimiter({ windowMs: 60000, max: 1 });
    limiter.check('user1');
    expect(limiter.check('user1').allowed).toBe(false);

    vi.advanceTimersByTime(60001);
    expect(limiter.check('user1').allowed).toBe(true);
  });

  it('returns remaining count', () => {
    const limiter = createRateLimiter({ windowMs: 60000, max: 3 });
    expect(limiter.check('user1').remaining).toBe(2);
    expect(limiter.check('user1').remaining).toBe(1);
    expect(limiter.check('user1').remaining).toBe(0);
  });
});
```

- [ ] **Step 2: Run test — verify it fails**

```bash
npm test -- src/test/rate-limit.test.ts
```

Expected: FAIL — module not found.

- [ ] **Step 3: Create rate limiter implementation**

Create `server/services/rate-limit.js`:

```js
/**
 * Create an in-memory rate limiter.
 * @param {{ windowMs: number, max: number }} opts
 * @returns {{ check: (key: string) => { allowed: boolean, remaining: number }, reset: (key: string) => void }}
 */
export function createRateLimiter({ windowMs, max }) {
  const hits = new Map(); // key → { count, resetTime }

  /**
   * Check if a key is within rate limits AND consume a slot.
   * Returns { allowed, remaining }.
   */
  function check(key) {
    const now = Date.now();
    const entry = hits.get(key);

    if (!entry || now > entry.resetTime) {
      hits.set(key, { count: 1, resetTime: now + windowMs });
      return { allowed: true, remaining: max - 1 };
    }

    if (entry.count >= max) {
      return { allowed: false, remaining: 0 };
    }

    entry.count++;
    return { allowed: true, remaining: max - entry.count };
  }

  function reset(key) {
    hits.delete(key);
  }

  // Periodic cleanup of expired entries to prevent unbounded memory growth
  setInterval(() => {
    const now = Date.now();
    for (const [key, entry] of hits) {
      if (now > entry.resetTime) hits.delete(key);
    }
  }, windowMs * 2).unref(); // unref() so it doesn't keep the process alive

  return { check, reset };
}
```

- [ ] **Step 4: Run test — verify it passes**

```bash
npm test -- src/test/rate-limit.test.ts
```

Expected: All 5 tests PASS.

- [ ] **Step 5: Commit**

```bash
git add server/services/rate-limit.js src/test/rate-limit.test.ts
git commit -m "feat: add in-memory rate limiter with tests"
```

---

### Task 8: Auth middleware

**Files:**
- Create: `cpo-connect-hub/server/middleware/auth.js`

**Prerequisite:** `cookie-parser` middleware must be mounted in `server.js` (see Task 3). This gives us `req.cookies` with parsed cookie values.

- [ ] **Step 1: Create session auth middleware**

```js
import { unsign } from 'cookie-signature';
import pool from '../db.js';

const SESSION_SECRET = process.env.SESSION_SECRET;

/**
 * Express middleware: validates session cookie, attaches req.user.
 * Relies on cookie-parser middleware being mounted in server.js.
 * Cookie format: s:<signed-session-id>
 */
export async function requireAuth(req, res, next) {
  try {
    const raw = req.cookies?.cpo_session;
    if (!raw) {
      return res.status(401).json({ error: 'Not authenticated', code: 'not_authenticated' });
    }

    // Remove s: prefix and unsign
    const value = raw.startsWith('s:') ? raw.slice(2) : null;
    if (!value) {
      return res.status(401).json({ error: 'Not authenticated', code: 'not_authenticated' });
    }

    const sessionId = unsign(value, SESSION_SECRET);
    if (sessionId === false) {
      return res.status(401).json({ error: 'Not authenticated', code: 'not_authenticated' });
    }

    const result = await pool.query(
      'SELECT id, email, name FROM cpo_connect.sessions WHERE id = $1 AND expires_at > NOW()',
      [sessionId]
    );

    if (result.rows.length === 0) {
      return res.status(401).json({ error: 'Not authenticated', code: 'not_authenticated' });
    }

    req.user = result.rows[0];
    next();
  } catch (err) {
    console.error('Auth middleware error:', err);
    return res.status(401).json({ error: 'Not authenticated', code: 'not_authenticated' });
  }
}
```

- [ ] **Step 2: Commit**

```bash
git add server/middleware/auth.js
git commit -m "feat: add session auth middleware"
```

---

## Chunk 3: API Routes

### Task 9: Auth API routes

**Files:**
- Create: `cpo-connect-hub/server/routes/auth.js`
- Modify: `cpo-connect-hub/server.js` (mount routes)

- [ ] **Step 1: Create auth routes**

Create `server/routes/auth.js`:

```js
import { Router } from 'express';
import crypto from 'crypto';
import { sign } from 'cookie-signature';
import pool from '../db.js';
import { lookupMember } from '../services/sheets.js';
import { sendMagicLink } from '../services/email.js';
import { createRateLimiter } from '../services/rate-limit.js';
import { requireAuth } from '../middleware/auth.js';

const router = Router();
const SESSION_SECRET = process.env.SESSION_SECRET;

// Rate limiters
const emailLimiter = createRateLimiter({ windowMs: 60 * 60 * 1000, max: 3 }); // 3/email/hour
const ipLimiter = createRateLimiter({ windowMs: 60 * 1000, max: 10 });         // 10/IP/minute

// Membership cache for /me re-validation (5-min TTL per email)
const membershipCache = new Map();
const MEMBERSHIP_CACHE_TTL = 5 * 60 * 1000;

/**
 * POST /api/auth/request
 * Always returns 200 to prevent email enumeration.
 */
router.post('/request', async (req, res) => {
  try {
    const { email } = req.body;
    if (!email || typeof email !== 'string') {
      return res.status(200).json({ code: 'magic_link_sent' });
    }

    const normalizedEmail = email.toLowerCase().trim();
    const clientIp = req.headers['x-forwarded-for']?.split(',')[0]?.trim() || req.ip;

    // Rate limit checks
    if (!ipLimiter.check(clientIp).allowed) {
      return res.status(429).json({ error: 'Too many requests', code: 'rate_limited' });
    }
    if (!emailLimiter.check(normalizedEmail).allowed) {
      return res.status(429).json({ error: 'Too many requests', code: 'rate_limited' });
    }

    // Cleanup expired tokens/sessions (best-effort)
    pool.query('DELETE FROM cpo_connect.magic_link_tokens WHERE expires_at < NOW()').catch(() => {});
    pool.query('DELETE FROM cpo_connect.sessions WHERE expires_at < NOW()').catch(() => {});

    // Look up member in Google Sheet
    const member = await lookupMember(normalizedEmail);

    if (!member || member.status !== 'Joined') {
      // Return 200 with memberStatus to let frontend show "Apply to Join" UI.
      // This is visible in devtools but the attacker already knows the email they typed.
      // The uniform 200 status prevents automated enumeration via status codes.
      return res.status(200).json({ code: 'magic_link_sent', memberStatus: 'not_found' });
    }

    // Generate token
    const token = crypto.randomBytes(32).toString('hex');
    const expiresAt = new Date(Date.now() + 15 * 60 * 1000); // 15 minutes

    await pool.query(
      'INSERT INTO cpo_connect.magic_link_tokens (email, token, expires_at) VALUES ($1, $2, $3)',
      [normalizedEmail, token, expiresAt]
    );

    // Send email
    await sendMagicLink({ email: normalizedEmail, token, name: member.name });

    return res.status(200).json({ code: 'magic_link_sent', memberStatus: 'sent' });
  } catch (err) {
    console.error('Auth request error:', err);
    return res.status(200).json({ code: 'magic_link_sent' });
  }
});

/**
 * GET /api/auth/verify?token=xxx
 * Validates token, creates session, sets cookie, redirects to /members.
 */
router.get('/verify', async (req, res) => {
  try {
    const { token } = req.query;
    if (!token) {
      // Redirect to landing page with error param — user clicked a broken link
      return res.redirect(302, '/?verify=invalid');
    }

    // Find and validate token
    const result = await pool.query(
      `SELECT id, email FROM cpo_connect.magic_link_tokens
       WHERE token = $1 AND used = FALSE AND expires_at > NOW()`,
      [token]
    );

    if (result.rows.length === 0) {
      // Token expired or already used — redirect to landing with error
      return res.redirect(302, '/?verify=expired');
    }

    const { id: tokenId, email } = result.rows[0];

    // Mark token as used
    await pool.query(
      'UPDATE cpo_connect.magic_link_tokens SET used = TRUE WHERE id = $1',
      [tokenId]
    );

    // Look up member name for session
    const member = await lookupMember(email);
    const name = member?.name || '';

    // Create session (30-day expiry)
    const sessionExpiry = new Date(Date.now() + 30 * 24 * 60 * 60 * 1000);
    const sessionResult = await pool.query(
      'INSERT INTO cpo_connect.sessions (email, name, expires_at) VALUES ($1, $2, $3) RETURNING id',
      [email, name, sessionExpiry]
    );

    const sessionId = sessionResult.rows[0].id;

    // Set signed httpOnly cookie
    const signedValue = 's:' + sign(sessionId, SESSION_SECRET);
    res.cookie('cpo_session', signedValue, {
      httpOnly: true,
      secure: process.env.NODE_ENV === 'production',
      sameSite: 'lax',
      maxAge: 30 * 24 * 60 * 60 * 1000, // 30 days
      path: '/',
    });

    return res.redirect(302, '/members');
  } catch (err) {
    console.error('Auth verify error:', err);
    return res.redirect(302, '/?verify=error');
  }
});

/**
 * GET /api/auth/me
 * Returns current user. Re-validates membership against Google Sheet (cached 5 min).
 */
router.get('/me', requireAuth, async (req, res) => {
  try {
    const { email, name } = req.user;

    // Check membership cache
    const cached = membershipCache.get(email);
    const now = Date.now();

    if (!cached || now - cached.timestamp > MEMBERSHIP_CACHE_TTL) {
      const member = await lookupMember(email);

      if (!member || member.status !== 'Joined') {
        // Membership revoked — delete session
        await pool.query('DELETE FROM cpo_connect.sessions WHERE id = $1', [req.user.id]);
        membershipCache.delete(email);
        return res.status(403).json({ error: 'Membership revoked', code: 'membership_revoked' });
      }

      membershipCache.set(email, { status: member.status, jobRole: member.jobRole, timestamp: now });
    }

    const cachedMember = membershipCache.get(email);
    return res.json({ name, email, jobRole: cachedMember?.jobRole || '' });
  } catch (err) {
    console.error('Auth me error:', err);
    return res.status(401).json({ error: 'Not authenticated', code: 'not_authenticated' });
  }
});

/**
 * POST /api/auth/logout
 * Clears session.
 */
router.post('/logout', requireAuth, async (req, res) => {
  try {
    await pool.query('DELETE FROM cpo_connect.sessions WHERE id = $1', [req.user.id]);
    res.clearCookie('cpo_session', { path: '/' });
    return res.json({ code: 'logged_out' });
  } catch (err) {
    console.error('Logout error:', err);
    return res.json({ code: 'logged_out' });
  }
});

export default router;
```

**Implementation notes:**
- `POST /request` always returns HTTP 200 to prevent automated enumeration via status codes. The `memberStatus` field lets the frontend show different UI ("check your inbox" vs "apply to join"). This field is visible in devtools, but the attacker already knows the email they typed — the uniform 200 prevents scripted enumeration.
- `GET /verify` redirects on error instead of returning JSON, since users reach this endpoint by clicking a link in their email and expect to see a web page, not raw JSON. The frontend reads the `?verify=` query param to show an appropriate error message with an option to request a new link.

- [ ] **Step 2: Mount auth routes in server.js**

Update `server.js` — add import and mount:

```js
import authRouter from './server/routes/auth.js';

// Add before static file serving:
app.use('/api/auth', authRouter);
```

- [ ] **Step 3: Commit**

```bash
git add server/routes/auth.js server.js
git commit -m "feat: add auth API routes (request, verify, me, logout)"
```

---

### Task 10: Members directory API route

**Files:**
- Create: `cpo-connect-hub/server/routes/members.js`
- Modify: `cpo-connect-hub/server.js` (mount route)

- [ ] **Step 1: Create directory route**

Create `server/routes/members.js`:

```js
import { Router } from 'express';
import { requireAuth } from '../middleware/auth.js';
import { getDirectory } from '../services/sheets.js';

const router = Router();

/**
 * GET /api/members/directory
 * Returns member directory data from Google Sheet (cached 5 min).
 */
router.get('/directory', requireAuth, async (req, res) => {
  try {
    const members = await getDirectory();
    return res.json({ members });
  } catch (err) {
    console.error('Directory error:', err);
    return res.status(500).json({ error: 'Service temporarily unavailable', code: 'service_error' });
  }
});

export default router;
```

- [ ] **Step 2: Mount in server.js**

Add import and mount:

```js
import membersRouter from './server/routes/members.js';

app.use('/api/members', membersRouter);
```

- [ ] **Step 3: Uncomment route mounts in server.js and remove placeholder comments**

Final `server.js` should have both routes mounted before static files.

- [ ] **Step 4: Commit**

```bash
git add server/routes/members.js server.js
git commit -m "feat: add members directory API route"
```

---

## Chunk 4: Frontend Auth

### Task 11: Auth context and useAuth hook

**Files:**
- Create: `cpo-connect-hub/src/contexts/AuthContext.tsx`

- [ ] **Step 1: Create auth context**

```tsx
import { createContext, useContext, useState, useEffect, useCallback, type ReactNode } from 'react';

interface User {
  name: string;
  email: string;
  jobRole: string;
}

interface AuthContextType {
  user: User | null;
  isLoading: boolean;
  isAuthenticated: boolean;
  login: (email: string) => Promise<{ memberStatus: string }>;
  logout: () => Promise<void>;
  checkAuth: () => Promise<void>;
}

const AuthContext = createContext<AuthContextType | null>(null);

export function AuthProvider({ children }: { children: ReactNode }) {
  const [user, setUser] = useState<User | null>(null);
  const [isLoading, setIsLoading] = useState(true);

  const checkAuth = useCallback(async () => {
    try {
      const res = await fetch('/api/auth/me');
      if (res.ok) {
        const data = await res.json();
        setUser(data);
      } else {
        setUser(null);
      }
    } catch {
      setUser(null);
    } finally {
      setIsLoading(false);
    }
  }, []);

  useEffect(() => {
    checkAuth();
  }, [checkAuth]);

  const login = async (email: string) => {
    const res = await fetch('/api/auth/request', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ email }),
    });

    if (res.status === 429) {
      throw new Error('Too many requests. Please try again later.');
    }

    return res.json();
  };

  const logout = async () => {
    await fetch('/api/auth/logout', { method: 'POST' });
    setUser(null);
  };

  return (
    <AuthContext.Provider value={{
      user,
      isLoading,
      isAuthenticated: !!user,
      login,
      logout,
      checkAuth,
    }}>
      {children}
    </AuthContext.Provider>
  );
}

export function useAuth() {
  const context = useContext(AuthContext);
  if (!context) {
    throw new Error('useAuth must be used within an AuthProvider');
  }
  return context;
}
```

- [ ] **Step 2: Wrap App with AuthProvider**

In `src/App.tsx`, import `AuthProvider` and wrap the router:

```tsx
import { AuthProvider } from '@/contexts/AuthContext';

// Wrap inside QueryClientProvider:
<QueryClientProvider client={queryClient}>
  <AuthProvider>
    <TooltipProvider>
      {/* ... existing content ... */}
    </TooltipProvider>
  </AuthProvider>
</QueryClientProvider>
```

- [ ] **Step 3: Commit**

```bash
git add src/contexts/AuthContext.tsx src/App.tsx
git commit -m "feat: add auth context and useAuth hook"
```

---

### Task 12: Login modal

**Files:**
- Create: `cpo-connect-hub/src/components/LoginModal.tsx`

- [ ] **Step 1: Create login modal component**

```tsx
import { useState } from 'react';
import { Dialog, DialogContent, DialogHeader, DialogTitle } from '@/components/ui/dialog';
import { Button } from '@/components/ui/button';
import { Input } from '@/components/ui/input';
import { useAuth } from '@/contexts/AuthContext';
import { Mail, ArrowLeft, Loader2 } from 'lucide-react';

interface LoginModalProps {
  open: boolean;
  onOpenChange: (open: boolean) => void;
  verifyError?: string | null;
  onClearVerifyError?: () => void;
}

type ModalState = 'email' | 'sent' | 'not-member' | 'error';

export default function LoginModal({ open, onOpenChange, verifyError, onClearVerifyError }: LoginModalProps) {
  const { login } = useAuth();
  const [email, setEmail] = useState('');
  const [state, setState] = useState<ModalState>(verifyError ? 'error' : 'email');
  const [isSubmitting, setIsSubmitting] = useState(false);
  const [errorMessage, setErrorMessage] = useState(verifyError || '');

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    if (!email.trim()) return;

    setIsSubmitting(true);
    setErrorMessage('');

    try {
      const result = await login(email.trim());

      if (result.memberStatus === 'not_found') {
        setState('not-member');
      } else {
        setState('sent');
      }
    } catch (err) {
      setErrorMessage(err instanceof Error ? err.message : 'Something went wrong');
      setState('error');
    } finally {
      setIsSubmitting(false);
    }
  };

  const handleReset = () => {
    setEmail('');
    setState('email');
    setErrorMessage('');
    onClearVerifyError?.();
  };

  const handleClose = (isOpen: boolean) => {
    if (!isOpen) handleReset();
    onOpenChange(isOpen);
  };

  return (
    <Dialog open={open} onOpenChange={handleClose}>
      <DialogContent className="sm:max-w-md bg-card border-border">
        {state === 'email' && (
          <>
            <DialogHeader>
              <DialogTitle className="text-xl font-display">Sign in to CPO Connect</DialogTitle>
            </DialogHeader>
            <form onSubmit={handleSubmit} className="space-y-4 mt-4">
              <div className="space-y-2">
                <label htmlFor="email" className="text-sm text-muted-foreground">
                  Enter your membership email
                </label>
                <Input
                  id="email"
                  type="email"
                  placeholder="you@example.com"
                  value={email}
                  onChange={(e) => setEmail(e.target.value)}
                  required
                  autoFocus
                />
              </div>
              <Button type="submit" className="w-full" disabled={isSubmitting}>
                {isSubmitting ? (
                  <Loader2 className="h-4 w-4 animate-spin mr-2" />
                ) : (
                  <Mail className="h-4 w-4 mr-2" />
                )}
                Send Magic Link
              </Button>
            </form>
          </>
        )}

        {state === 'sent' && (
          <div className="text-center py-4">
            <div className="w-12 h-12 rounded-full bg-primary/10 flex items-center justify-center mx-auto mb-4">
              <Mail className="h-6 w-6 text-primary" />
            </div>
            <h3 className="text-lg font-display font-semibold mb-2">Check your inbox</h3>
            <p className="text-muted-foreground text-sm mb-1">
              We've sent a sign-in link to
            </p>
            <p className="font-medium text-sm mb-4">{email}</p>
            <p className="text-xs text-muted-foreground">
              The link expires in 15 minutes.
            </p>
          </div>
        )}

        {state === 'not-member' && (
          <div className="text-center py-4">
            <h3 className="text-lg font-display font-semibold mb-2">
              We don't recognise that email
            </h3>
            <p className="text-muted-foreground text-sm mb-6">
              This email isn't linked to a CPO Connect membership.
            </p>
            <div className="space-y-3">
              <Button className="w-full" asChild>
                <a href="https://cpoconnect.fillout.com/application" target="_blank" rel="noopener noreferrer">
                  Apply to Join
                </a>
              </Button>
              <Button variant="ghost" className="w-full" onClick={handleReset}>
                <ArrowLeft className="h-4 w-4 mr-2" />
                Try a different email
              </Button>
            </div>
          </div>
        )}

        {state === 'error' && (
          <div className="text-center py-4">
            <h3 className="text-lg font-display font-semibold mb-2 text-destructive">
              Something went wrong
            </h3>
            <p className="text-muted-foreground text-sm mb-4">{errorMessage}</p>
            <Button variant="ghost" onClick={handleReset}>
              Try again
            </Button>
          </div>
        )}
      </DialogContent>
    </Dialog>
  );
}
```

- [ ] **Step 2: Commit**

```bash
git add src/components/LoginModal.tsx
git commit -m "feat: add login modal with magic link flow"
```

---

### Task 13: Auth-aware Navbar

**Files:**
- Modify: `cpo-connect-hub/src/components/Navbar.tsx`

- [ ] **Step 1: Update Navbar for auth states**

Rewrite `Navbar.tsx` to handle both unauthenticated (landing page links + Login/Apply) and authenticated (members nav + avatar + logout) states:

```tsx
import { useState, useEffect } from 'react';
import { Link, useLocation, useSearchParams } from 'react-router-dom';
import { Menu, X, LogOut } from 'lucide-react';
import { Button } from '@/components/ui/button';
import {
  DropdownMenu,
  DropdownMenuContent,
  DropdownMenuItem,
  DropdownMenuTrigger,
} from '@/components/ui/dropdown-menu';
import { Avatar, AvatarFallback } from '@/components/ui/avatar';
import { useAuth } from '@/contexts/AuthContext';
import LoginModal from '@/components/LoginModal';
import logo from '@/assets/logo.png';

const publicLinks = [
  { label: 'Manifesto', href: '#manifesto' },
  { label: 'Channels', href: '#channels' },
  { label: 'Events', href: '#events' },
  { label: 'Founders', href: '#founders' },
];

const memberLinks = [
  { label: 'Chat Insights', to: '/members/chat-insights' },
  { label: 'Directory', to: '/members/directory' },
];

const Navbar = () => {
  const [mobileOpen, setMobileOpen] = useState(false);
  const [loginOpen, setLoginOpen] = useState(false);
  const [verifyError, setVerifyError] = useState<string | null>(null);
  const { user, isAuthenticated, logout } = useAuth();
  const location = useLocation();
  const [searchParams, setSearchParams] = useSearchParams();

  // Auto-open login modal when redirected from ProtectedRoute
  useEffect(() => {
    if (location.state?.showLogin && !isAuthenticated) {
      setLoginOpen(true);
    }
  }, [location.state, isAuthenticated]);

  // Handle ?verify= param from magic link errors
  useEffect(() => {
    const verify = searchParams.get('verify');
    if (verify) {
      setVerifyError(
        verify === 'expired' ? 'Your login link has expired. Please request a new one.'
        : verify === 'invalid' ? 'That login link is invalid. Please request a new one.'
        : 'Something went wrong. Please request a new login link.'
      );
      setLoginOpen(true);
      // Clean up the URL
      searchParams.delete('verify');
      setSearchParams(searchParams, { replace: true });
    }
  }, [searchParams, setSearchParams]);

  const isOnMembersPage = location.pathname.startsWith('/members');
  const initials = user?.name
    ? user.name.split(' ').map(w => w[0]).join('').toUpperCase().slice(0, 2)
    : '??';

  const handleLogout = async () => {
    await logout();
    window.location.href = '/';
  };

  return (
    <>
      <header className="sticky top-0 z-50 bg-background/80 backdrop-blur-lg border-b">
        <div className="container flex items-center justify-between h-16">
          <Link to="/" className="flex items-center gap-2">
            <img src={logo} alt="CPO Connect" className="h-9 w-9" />
            <span className="font-display text-xl font-bold tracking-tight">
              CPO <span className="text-primary">Connect</span>
            </span>
          </Link>

          {/* Desktop nav */}
          <nav className="hidden md:flex items-center gap-8">
            {isAuthenticated && isOnMembersPage ? (
              <>
                {memberLinks.map((l) => (
                  <Link
                    key={l.to}
                    to={l.to}
                    className={`text-sm font-medium transition-colors ${
                      location.pathname === l.to
                        ? 'text-foreground'
                        : 'text-muted-foreground hover:text-foreground'
                    }`}
                  >
                    {l.label}
                  </Link>
                ))}
                <DropdownMenu>
                  <DropdownMenuTrigger asChild>
                    <Button variant="ghost" className="relative h-9 w-9 rounded-full p-0">
                      <Avatar className="h-9 w-9">
                        <AvatarFallback className="bg-primary/10 text-primary text-sm font-medium">
                          {initials}
                        </AvatarFallback>
                      </Avatar>
                    </Button>
                  </DropdownMenuTrigger>
                  <DropdownMenuContent align="end">
                    <DropdownMenuItem onClick={handleLogout}>
                      <LogOut className="h-4 w-4 mr-2" />
                      Sign out
                    </DropdownMenuItem>
                  </DropdownMenuContent>
                </DropdownMenu>
              </>
            ) : (
              <>
                {publicLinks.map((l) => (
                  <a key={l.href} href={l.href} className="text-sm font-medium text-muted-foreground hover:text-foreground transition-colors">
                    {l.label}
                  </a>
                ))}
                {isAuthenticated ? (
                  <Button size="sm" asChild>
                    <Link to="/members/chat-insights">Members Area</Link>
                  </Button>
                ) : (
                  <div className="flex items-center gap-3">
                    <Button size="sm" variant="ghost" onClick={() => setLoginOpen(true)}>
                      Login
                    </Button>
                    <Button size="sm" className="bg-primary text-primary-foreground rounded-lg" asChild>
                      <a href="https://cpoconnect.fillout.com/application" target="_blank" rel="noopener noreferrer">
                        Apply to Join
                      </a>
                    </Button>
                  </div>
                )}
              </>
            )}
          </nav>

          {/* Mobile toggle */}
          <button className="md:hidden p-2" onClick={() => setMobileOpen(!mobileOpen)} aria-label="Toggle menu">
            {mobileOpen ? <X className="h-5 w-5" /> : <Menu className="h-5 w-5" />}
          </button>
        </div>

        {/* Mobile menu */}
        {mobileOpen && (
          <nav className="md:hidden border-t bg-background pb-4">
            {isAuthenticated && isOnMembersPage ? (
              <>
                {memberLinks.map((l) => (
                  <Link
                    key={l.to}
                    to={l.to}
                    onClick={() => setMobileOpen(false)}
                    className="block px-6 py-3 text-sm font-medium text-muted-foreground hover:text-foreground"
                  >
                    {l.label}
                  </Link>
                ))}
                <div className="px-6 pt-2">
                  <Button size="sm" variant="ghost" className="w-full" onClick={handleLogout}>
                    <LogOut className="h-4 w-4 mr-2" />
                    Sign out
                  </Button>
                </div>
              </>
            ) : (
              <>
                {publicLinks.map((l) => (
                  <a
                    key={l.href}
                    href={l.href}
                    onClick={() => setMobileOpen(false)}
                    className="block px-6 py-3 text-sm font-medium text-muted-foreground hover:text-foreground"
                  >
                    {l.label}
                  </a>
                ))}
                <div className="px-6 pt-2 space-y-2">
                  {isAuthenticated ? (
                    <Button size="sm" className="w-full" asChild>
                      <Link to="/members/chat-insights" onClick={() => setMobileOpen(false)}>Members Area</Link>
                    </Button>
                  ) : (
                    <>
                      <Button size="sm" variant="ghost" className="w-full" onClick={() => { setLoginOpen(true); setMobileOpen(false); }}>
                        Login
                      </Button>
                      <Button size="sm" className="bg-primary text-primary-foreground rounded-lg w-full" asChild>
                        <a href="https://cpoconnect.fillout.com/application" target="_blank" rel="noopener noreferrer" onClick={() => setMobileOpen(false)}>
                          Apply to Join
                        </a>
                      </Button>
                    </>
                  )}
                </div>
              </>
            )}
          </nav>
        )}
      </header>

      <LoginModal open={loginOpen} onOpenChange={setLoginOpen} verifyError={verifyError} onClearVerifyError={() => setVerifyError(null)} />
    </>
  );
};

export default Navbar;
```

**Key changes from original:**
- Uses `Link` from react-router-dom instead of `<a>` for internal navigation
- Shows public links (hash anchors) on landing page
- Shows member links (Chat Insights, Directory) + avatar dropdown on members pages
- Login button triggers `LoginModal`
- "Join" button renamed to "Apply to Join" with external link to Fillout form
- Authenticated users on the landing page see a "Members Area" button

- [ ] **Step 2: Commit**

```bash
git add src/components/Navbar.tsx
git commit -m "feat: update Navbar for auth-aware navigation"
```

---

### Task 14: Protected routes and members layout

**Files:**
- Create: `cpo-connect-hub/src/components/ProtectedRoute.tsx`
- Create: `cpo-connect-hub/src/components/members/MembersLayout.tsx`
- Modify: `cpo-connect-hub/src/App.tsx`
- Create: `cpo-connect-hub/src/pages/members/ChatInsights.tsx` (placeholder)
- Create: `cpo-connect-hub/src/pages/members/Directory.tsx` (placeholder)

- [ ] **Step 1: Create ProtectedRoute**

```tsx
import { Navigate } from 'react-router-dom';
import { useAuth } from '@/contexts/AuthContext';

interface ProtectedRouteProps {
  children: React.ReactNode;
}

export default function ProtectedRoute({ children }: ProtectedRouteProps) {
  const { isAuthenticated, isLoading } = useAuth();

  if (isLoading) {
    return (
      <div className="min-h-screen flex items-center justify-center">
        <div className="animate-spin rounded-full h-8 w-8 border-b-2 border-primary" />
      </div>
    );
  }

  if (!isAuthenticated) {
    // Pass state to tell the landing page to auto-open the login modal
    return <Navigate to="/" replace state={{ showLogin: true }} />;
  }

  return <>{children}</>;
}
```

- [ ] **Step 2: Create MembersLayout**

```tsx
import Navbar from '@/components/Navbar';

export default function MembersLayout({ children }: { children: React.ReactNode }) {
  return (
    <div className="min-h-screen bg-background">
      <Navbar />
      <main className="container py-8">
        {children}
      </main>
    </div>
  );
}
```

- [ ] **Step 3: Create placeholder pages**

Create `src/pages/members/ChatInsights.tsx`:

```tsx
export default function ChatInsights() {
  return (
    <div>
      <h1 className="text-3xl font-display font-bold mb-4">Chat Insights</h1>
      <p className="text-muted-foreground">Coming soon...</p>
    </div>
  );
}
```

Create `src/pages/members/Directory.tsx`:

```tsx
export default function Directory() {
  return (
    <div>
      <h1 className="text-3xl font-display font-bold mb-4">Member Directory</h1>
      <p className="text-muted-foreground">Coming soon...</p>
    </div>
  );
}
```

- [ ] **Step 4: Update App.tsx with all routes**

```tsx
import { Toaster } from "@/components/ui/toaster";
import { Toaster as Sonner } from "@/components/ui/sonner";
import { TooltipProvider } from "@/components/ui/tooltip";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import { BrowserRouter, Routes, Route, Navigate } from "react-router-dom";
import { AuthProvider } from "@/contexts/AuthContext";
import Index from "./pages/Index";
import NotFound from "./pages/NotFound";
import ProtectedRoute from "@/components/ProtectedRoute";
import MembersLayout from "@/components/members/MembersLayout";
import ChatInsights from "@/pages/members/ChatInsights";
import Directory from "@/pages/members/Directory";

const queryClient = new QueryClient();

const App = () => (
  <QueryClientProvider client={queryClient}>
    <AuthProvider>
      <TooltipProvider>
        <Toaster />
        <Sonner />
        <BrowserRouter>
          <Routes>
            <Route path="/" element={<Index />} />
            <Route path="/members" element={
              <ProtectedRoute>
                <Navigate to="/members/chat-insights" replace />
              </ProtectedRoute>
            } />
            <Route path="/members/chat-insights" element={
              <ProtectedRoute>
                <MembersLayout>
                  <ChatInsights />
                </MembersLayout>
              </ProtectedRoute>
            } />
            <Route path="/members/directory" element={
              <ProtectedRoute>
                <MembersLayout>
                  <Directory />
                </MembersLayout>
              </ProtectedRoute>
            } />
            <Route path="*" element={<NotFound />} />
          </Routes>
        </BrowserRouter>
      </TooltipProvider>
    </AuthProvider>
  </QueryClientProvider>
);

export default App;
```

- [ ] **Step 5: Verify build**

```bash
npm run build
```

Expected: Build succeeds with no errors.

- [ ] **Step 6: Commit**

```bash
git add src/components/ProtectedRoute.tsx src/components/members/MembersLayout.tsx src/pages/members/ src/App.tsx
git commit -m "feat: add protected routes, members layout, and route structure"
```

---

## Chunk 5: Members Pages

### Task 15: Chat insights shared components

**Files:**
- Create: `cpo-connect-hub/src/components/members/insights/StatCard.tsx`
- Create: `cpo-connect-hub/src/components/members/insights/TrendItem.tsx`
- Create: `cpo-connect-hub/src/components/members/insights/DailyVolumeChart.tsx`
- Create: `cpo-connect-hub/src/components/members/insights/ContributorsChart.tsx`
- Create: `cpo-connect-hub/src/components/members/insights/SentimentChart.tsx`
- Create: `cpo-connect-hub/src/components/members/insights/ChannelSection.tsx`

- [ ] **Step 1: Create StatCard**

```tsx
import { Card, CardContent } from '@/components/ui/card';

interface StatCardProps {
  label: string;
  value: string | number;
  gradient?: string; // e.g. "from-blue-400 to-purple-500"
}

export default function StatCard({ label, value, gradient = 'from-primary to-accent' }: StatCardProps) {
  return (
    <Card className="bg-card/50 border-border/50 backdrop-blur-sm">
      <CardContent className="p-5 text-center">
        <p className={`text-3xl font-display font-bold bg-gradient-to-r ${gradient} bg-clip-text text-transparent`}>
          {value}
        </p>
        <p className="text-sm text-muted-foreground mt-1">{label}</p>
      </CardContent>
    </Card>
  );
}
```

- [ ] **Step 2: Create TrendItem**

```tsx
import { Badge } from '@/components/ui/badge';

interface TrendItemProps {
  rank: number;
  title: string;
  description: string;
  tags?: { label: string; variant: 'hot' | 'green' | 'gold' | 'amber' | 'pink' | 'blue' }[];
  dateRange?: string;
}

const tagColors = {
  hot: 'bg-red-500/20 text-red-300 border-red-500/30',
  green: 'bg-emerald-500/20 text-emerald-300 border-emerald-500/30',
  gold: 'bg-amber-500/20 text-amber-300 border-amber-500/30',
  amber: 'bg-orange-500/20 text-orange-300 border-orange-500/30',
  pink: 'bg-pink-500/20 text-pink-300 border-pink-500/30',
  blue: 'bg-blue-500/20 text-blue-300 border-blue-500/30',
};

export default function TrendItem({ rank, title, description, tags, dateRange }: TrendItemProps) {
  return (
    <div className="flex gap-4 p-4 rounded-lg bg-card/30 border-l-2 border-primary/50">
      <span className="text-2xl font-display font-bold text-muted-foreground/50 shrink-0">
        {rank}
      </span>
      <div className="space-y-1.5">
        <h4 className="font-display font-semibold text-foreground">{title}</h4>
        <p className="text-sm text-muted-foreground leading-relaxed">{description}</p>
        <div className="flex flex-wrap gap-2 pt-1">
          {tags?.map((tag) => (
            <Badge key={tag.label} variant="outline" className={`text-xs ${tagColors[tag.variant]}`}>
              {tag.label}
            </Badge>
          ))}
          {dateRange && (
            <span className="text-xs text-muted-foreground">{dateRange}</span>
          )}
        </div>
      </div>
    </div>
  );
}
```

- [ ] **Step 3: Create DailyVolumeChart**

Uses recharts (already installed):

```tsx
import { BarChart, Bar, XAxis, YAxis, Tooltip, ResponsiveContainer } from 'recharts';
import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';

interface DailyVolumeChartProps {
  data: { day: string; messages: number }[];
  color?: string;
}

export default function DailyVolumeChart({ data, color = '#60a5fa' }: DailyVolumeChartProps) {
  return (
    <Card className="bg-card/50 border-border/50 backdrop-blur-sm">
      <CardHeader>
        <CardTitle className="text-base font-display">Daily Message Volume</CardTitle>
      </CardHeader>
      <CardContent>
        <div className="h-[300px]">
          <ResponsiveContainer width="100%" height="100%">
            <BarChart data={data}>
              <XAxis dataKey="day" stroke="#6b7280" fontSize={12} />
              <YAxis stroke="#6b7280" fontSize={12} />
              <Tooltip
                contentStyle={{ backgroundColor: '#1a1a2e', border: '1px solid rgba(255,255,255,0.1)', borderRadius: '8px' }}
                labelStyle={{ color: '#e5e7eb' }}
                itemStyle={{ color: '#e5e7eb' }}
              />
              <Bar dataKey="messages" fill={color} fillOpacity={0.6} radius={[4, 4, 0, 0]} />
            </BarChart>
          </ResponsiveContainer>
        </div>
      </CardContent>
    </Card>
  );
}
```

- [ ] **Step 4: Create ContributorsChart**

```tsx
import { PieChart, Pie, Cell, Tooltip, ResponsiveContainer, Legend } from 'recharts';
import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';

interface Contributor {
  name: string;
  messages: number;
  color: string;
}

interface ContributorsChartProps {
  data: Contributor[];
}

export default function ContributorsChart({ data }: ContributorsChartProps) {
  return (
    <Card className="bg-card/50 border-border/50 backdrop-blur-sm">
      <CardHeader>
        <CardTitle className="text-base font-display">Most Active Contributors</CardTitle>
      </CardHeader>
      <CardContent>
        <div className="h-[300px]">
          <ResponsiveContainer width="100%" height="100%">
            <PieChart>
              <Pie
                data={data}
                dataKey="messages"
                nameKey="name"
                cx="50%"
                cy="50%"
                innerRadius={60}
                outerRadius={100}
              >
                {data.map((entry, index) => (
                  <Cell key={index} fill={entry.color} />
                ))}
              </Pie>
              <Tooltip
                contentStyle={{ backgroundColor: '#1a1a2e', border: '1px solid rgba(255,255,255,0.1)', borderRadius: '8px' }}
                itemStyle={{ color: '#e5e7eb' }}
              />
              <Legend />
            </PieChart>
          </ResponsiveContainer>
        </div>
      </CardContent>
    </Card>
  );
}
```

- [ ] **Step 5: Create SentimentChart**

```tsx
import { RadarChart, PolarGrid, PolarAngleAxis, Radar, ResponsiveContainer, Tooltip } from 'recharts';
import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';

interface SentimentDataPoint {
  label: string;
  value: number;
}

interface SentimentChartProps {
  data: SentimentDataPoint[];
  color?: string;
}

export default function SentimentChart({ data, color = '#a78bfa' }: SentimentChartProps) {
  return (
    <Card className="bg-card/50 border-border/50 backdrop-blur-sm">
      <CardHeader>
        <CardTitle className="text-base font-display">Sentiment Snapshot</CardTitle>
      </CardHeader>
      <CardContent>
        <div className="h-[300px]">
          <ResponsiveContainer width="100%" height="100%">
            <RadarChart data={data} cx="50%" cy="50%" outerRadius="75%">
              <PolarGrid stroke="rgba(255,255,255,0.1)" />
              <PolarAngleAxis dataKey="label" stroke="#9ca3af" fontSize={12} />
              <Radar dataKey="value" fill={color} fillOpacity={0.3} stroke={color} strokeWidth={2} />
              <Tooltip
                contentStyle={{ backgroundColor: '#1a1a2e', border: '1px solid rgba(255,255,255,0.1)', borderRadius: '8px' }}
                itemStyle={{ color: '#e5e7eb' }}
              />
            </RadarChart>
          </ResponsiveContainer>
        </div>
      </CardContent>
    </Card>
  );
}
```

- [ ] **Step 6: Create ChannelSection**

Container that groups a channel's charts and trends:

```tsx
import DailyVolumeChart from './DailyVolumeChart';
import ContributorsChart from './ContributorsChart';
import SentimentChart from './SentimentChart';
import TrendItem from './TrendItem';

export interface ChannelData {
  name: string;
  dailyVolume: { day: string; messages: number }[];
  contributors: { name: string; messages: number; color: string }[];
  sentiment: { label: string; value: number }[];
  trends: {
    title: string;
    description: string;
    tags?: { label: string; variant: 'hot' | 'green' | 'gold' | 'amber' | 'pink' | 'blue' }[];
    dateRange?: string;
  }[];
  chartColor?: string;
  sentimentColor?: string;
}

export default function ChannelSection({ data }: { data: ChannelData }) {
  return (
    <div className="space-y-6">
      <div className="grid grid-cols-1 lg:grid-cols-2 gap-6">
        <DailyVolumeChart data={data.dailyVolume} color={data.chartColor} />
        <ContributorsChart data={data.contributors} />
      </div>

      <SentimentChart data={data.sentiment} color={data.sentimentColor} />

      <div className="space-y-2">
        <h3 className="text-lg font-display font-semibold mb-4">Key Trends & Themes</h3>
        <div className="space-y-3">
          {data.trends.map((trend, i) => (
            <TrendItem key={i} rank={i + 1} {...trend} />
          ))}
        </div>
      </div>
    </div>
  );
}
```

- [ ] **Step 7: Commit**

```bash
git add src/components/members/insights/
git commit -m "feat: add chat insights shared components (charts, cards, trends)"
```

---

### Task 16: Chat insights data and month config

**Files:**
- Create: `cpo-connect-hub/src/data/insights/config.ts`
- Create: `cpo-connect-hub/src/data/insights/feb-2026.tsx`
- Create: `cpo-connect-hub/src/data/insights/jan-2026.tsx`

- [ ] **Step 1: Create month config**

```ts
import { lazy, type ComponentType } from 'react';

export interface MonthConfig {
  id: string;           // e.g. "2026-02"
  label: string;        // e.g. "February 2026"
  component: React.LazyExoticComponent<ComponentType>;
}

export const months: MonthConfig[] = [
  {
    id: '2026-02',
    label: 'February 2026',
    component: lazy(() => import('./feb-2026')),
  },
  {
    id: '2026-01',
    label: 'January 2026',
    component: lazy(() => import('./jan-2026')),
  },
];

// Default to latest month
export const defaultMonth = months[0].id;
```

Adding a new month = one import line + one entry in the `months` array.

- [ ] **Step 2: Create February 2026 analysis component**

Create `src/data/insights/feb-2026.tsx`. Port the data from `chat-analysis-feb2026.html`:

```tsx
import { useState } from 'react';
import { Tabs, TabsContent, TabsList, TabsTrigger } from '@/components/ui/tabs';
import StatCard from '@/components/members/insights/StatCard';
import ChannelSection, { type ChannelData } from '@/components/members/insights/ChannelSection';

// --- DATA ---
// Port actual values from chat-analysis-feb2026.html Chart.js datasets.
// The structure below shows the shape — fill with real data during implementation.

const stats = [
  { label: 'Total Messages', value: '1,483', gradient: 'from-blue-400 to-purple-500' },
  { label: 'Active Members', value: '130+', gradient: 'from-emerald-400 to-teal-500' },
  { label: 'Channels Analysed', value: '3', gradient: 'from-amber-400 to-orange-500' },
  { label: 'Active Days', value: '28', gradient: 'from-pink-400 to-rose-500' },
];

const aiChannel: ChannelData = {
  name: 'AI',
  dailyVolume: [
    // Port from HTML: labels array → day, datasets[0].data → messages
    // Example: { day: '1', messages: 7 }, { day: '2', messages: 28 }, ...
  ],
  contributors: [
    // Port from HTML: labels → name, datasets[0].data → messages
    // Example: { name: 'Erik', messages: 184, color: '#a78bfa' }, ...
  ],
  sentiment: [
    // Port from HTML: labels → label, datasets[0].data → value
    // Example: { label: 'Curious & Experimental', value: 30 }, ...
  ],
  trends: [
    // Port from HTML: each .trend-entry → { title, description, tags, dateRange }
  ],
  chartColor: '#60a5fa',
  sentimentColor: '#a78bfa',
};

const generalChannel: ChannelData = {
  name: 'General',
  dailyVolume: [],    // Port from HTML
  contributors: [],   // Port from HTML
  sentiment: [],      // Port from HTML
  trends: [],         // Port from HTML
  chartColor: '#f97316',
  sentimentColor: '#facc15',
};

const leadershipChannel: ChannelData = {
  name: 'Leadership & Culture',
  dailyVolume: [],    // Port from HTML
  contributors: [],   // Port from HTML
  sentiment: [],      // Port from HTML
  trends: [],         // Port from HTML
  chartColor: '#34d399',
  sentimentColor: '#a78bfa',
};

// --- COMPONENT ---

export default function Feb2026() {
  return (
    <div className="space-y-8">
      <div>
        <h2 className="text-2xl font-display font-bold mb-1">February 2026</h2>
        <p className="text-muted-foreground">Community chat analysis across 3 channels</p>
      </div>

      <div className="grid grid-cols-2 md:grid-cols-4 gap-4">
        {stats.map((s) => (
          <StatCard key={s.label} {...s} />
        ))}
      </div>

      <Tabs defaultValue="ai">
        <TabsList className="bg-muted/50">
          <TabsTrigger value="ai">AI</TabsTrigger>
          <TabsTrigger value="general">General</TabsTrigger>
          <TabsTrigger value="leadership">Leadership & Culture</TabsTrigger>
        </TabsList>
        <TabsContent value="ai" className="mt-6">
          <ChannelSection data={aiChannel} />
        </TabsContent>
        <TabsContent value="general" className="mt-6">
          <ChannelSection data={generalChannel} />
        </TabsContent>
        <TabsContent value="leadership" className="mt-6">
          <ChannelSection data={leadershipChannel} />
        </TabsContent>
      </Tabs>
    </div>
  );
}
```

**Implementation note:** During execution, the empty arrays must be filled with actual data values copied from the HTML files. Each HTML file has the data embedded in `<script>` tags as Chart.js config objects. Extract the `labels` and `data` arrays.

- [ ] **Step 3: Create January 2026 analysis component**

Create `src/data/insights/jan-2026.tsx`. January only has the AI channel:

```tsx
import StatCard from '@/components/members/insights/StatCard';
import ChannelSection, { type ChannelData } from '@/components/members/insights/ChannelSection';

const stats = [
  // Port from chat-analysis-jan2026.html
  { label: 'Total Messages', value: '???', gradient: 'from-blue-400 to-purple-500' },
  { label: 'Active Members', value: '???', gradient: 'from-emerald-400 to-teal-500' },
  { label: 'Active Days', value: '???', gradient: 'from-pink-400 to-rose-500' },
];

const aiChannel: ChannelData = {
  name: 'AI',
  dailyVolume: [],    // Port from HTML
  contributors: [],   // Port from HTML
  sentiment: [],      // Port from HTML
  trends: [],         // Port from HTML
  chartColor: '#60a5fa',
  sentimentColor: '#a78bfa',
};

export default function Jan2026() {
  return (
    <div className="space-y-8">
      <div>
        <h2 className="text-2xl font-display font-bold mb-1">January 2026</h2>
        <p className="text-muted-foreground">AI channel analysis</p>
      </div>

      <div className="grid grid-cols-2 md:grid-cols-3 gap-4">
        {stats.map((s) => (
          <StatCard key={s.label} {...s} />
        ))}
      </div>

      <ChannelSection data={aiChannel} />
    </div>
  );
}
```

- [ ] **Step 4: Commit**

```bash
git add src/data/insights/
git commit -m "feat: add chat insights month config and analysis data components"
```

---

### Task 17: Chat insights page with month selector

**Files:**
- Modify: `cpo-connect-hub/src/pages/members/ChatInsights.tsx`

- [ ] **Step 1: Implement ChatInsights page**

Replace the placeholder:

```tsx
import { useState, Suspense } from 'react';
import { ChevronLeft, ChevronRight } from 'lucide-react';
import { Button } from '@/components/ui/button';
import { months, defaultMonth } from '@/data/insights/config';

export default function ChatInsights() {
  const [currentMonthId, setCurrentMonthId] = useState(defaultMonth);
  const currentIndex = months.findIndex(m => m.id === currentMonthId);
  const current = months[currentIndex];

  const goNext = () => {
    if (currentIndex > 0) setCurrentMonthId(months[currentIndex - 1].id);
  };

  const goPrev = () => {
    if (currentIndex < months.length - 1) setCurrentMonthId(months[currentIndex + 1].id);
  };

  const MonthComponent = current.component;

  return (
    <div className="space-y-6">
      <div className="flex items-center justify-between">
        <h1 className="text-3xl font-display font-bold">Chat Insights</h1>
        <div className="flex items-center gap-2">
          <Button
            variant="ghost"
            size="icon"
            onClick={goPrev}
            disabled={currentIndex === months.length - 1}
            aria-label="Previous month"
          >
            <ChevronLeft className="h-5 w-5" />
          </Button>
          <span className="text-sm font-medium min-w-[140px] text-center">
            {current.label}
          </span>
          <Button
            variant="ghost"
            size="icon"
            onClick={goNext}
            disabled={currentIndex === 0}
            aria-label="Next month"
          >
            <ChevronRight className="h-5 w-5" />
          </Button>
        </div>
      </div>

      <Suspense fallback={
        <div className="flex items-center justify-center py-20">
          <div className="animate-spin rounded-full h-8 w-8 border-b-2 border-primary" />
        </div>
      }>
        <MonthComponent />
      </Suspense>
    </div>
  );
}
```

**Navigation logic:** `months` array is ordered newest-first. "Previous" goes to an older month (higher index), "Next" goes to a newer month (lower index).

- [ ] **Step 2: Verify build**

```bash
npm run build
```

Expected: Build succeeds.

- [ ] **Step 3: Commit**

```bash
git add src/pages/members/ChatInsights.tsx
git commit -m "feat: implement chat insights page with month selector"
```

---

### Task 18: Member directory page

**Files:**
- Create: `cpo-connect-hub/src/components/members/directory/MemberCard.tsx`
- Modify: `cpo-connect-hub/src/pages/members/Directory.tsx`

- [ ] **Step 1: Create MemberCard**

```tsx
import { Card, CardContent } from '@/components/ui/card';
import { Avatar, AvatarFallback } from '@/components/ui/avatar';
import { Badge } from '@/components/ui/badge';
import { Linkedin } from 'lucide-react';

interface MemberCardProps {
  name: string;
  role: string;
  focusAreas?: string[];
  linkedIn?: string;
}

export default function MemberCard({ name, role, focusAreas, linkedIn }: MemberCardProps) {
  const initials = name
    .split(' ')
    .map(w => w[0])
    .join('')
    .toUpperCase()
    .slice(0, 2);

  return (
    <Card className="bg-card/50 border-border/50 backdrop-blur-sm hover:border-primary/30 transition-colors">
      <CardContent className="p-5">
        <div className="flex items-start gap-4">
          <Avatar className="h-12 w-12 shrink-0">
            <AvatarFallback className="bg-primary/10 text-primary font-display font-semibold">
              {initials}
            </AvatarFallback>
          </Avatar>
          <div className="min-w-0 flex-1">
            <div className="flex items-center gap-2">
              <h3 className="font-display font-semibold truncate">{name}</h3>
              {linkedIn && (
                <a
                  href={linkedIn}
                  target="_blank"
                  rel="noopener noreferrer"
                  className="text-muted-foreground hover:text-primary transition-colors shrink-0"
                  aria-label={`${name}'s LinkedIn`}
                >
                  <Linkedin className="h-4 w-4" />
                </a>
              )}
            </div>
            <p className="text-sm text-muted-foreground mt-0.5">{role}</p>
            {focusAreas && focusAreas.length > 0 && (
              <div className="flex flex-wrap gap-1.5 mt-3">
                {focusAreas.map(area => (
                  <Badge key={area} variant="secondary" className="text-xs font-normal">
                    {area}
                  </Badge>
                ))}
              </div>
            )}
          </div>
        </div>
      </CardContent>
    </Card>
  );
}
```

- [ ] **Step 2: Implement Directory page**

Replace the placeholder in `src/pages/members/Directory.tsx`:

```tsx
import { useState } from 'react';
import { useQuery } from '@tanstack/react-query';
import { Search } from 'lucide-react';
import { Input } from '@/components/ui/input';
import { Select, SelectContent, SelectItem, SelectTrigger, SelectValue } from '@/components/ui/select';
import MemberCard from '@/components/members/directory/MemberCard';
import { Skeleton } from '@/components/ui/skeleton';

interface DirectoryMember {
  [key: string]: string;
}

async function fetchDirectory(): Promise<DirectoryMember[]> {
  const res = await fetch('/api/members/directory');
  if (!res.ok) throw new Error('Failed to load directory');
  const data = await res.json();
  return data.members;
}

export default function Directory() {
  const [search, setSearch] = useState('');
  const [roleFilter, setRoleFilter] = useState('all');

  const { data: members, isLoading, error } = useQuery({
    queryKey: ['directory'],
    queryFn: fetchDirectory,
  });

  // Extract unique roles for filter dropdown
  const roles = members
    ? [...new Set(members.map(m => m['Role'] || m['Job Title'] || '').filter(Boolean))].sort()
    : [];

  // Client-side filtering
  const filtered = members?.filter(m => {
    const name = (m['Full Name'] || m['Name'] || '').toLowerCase();
    const role = (m['Role'] || m['Job Title'] || '').toLowerCase();
    const industry = (m['Industry'] || '').toLowerCase();
    const searchLower = search.toLowerCase();

    const matchesSearch = !search || name.includes(searchLower) || role.includes(searchLower) || industry.includes(searchLower);
    const matchesRole = roleFilter === 'all' || role === roleFilter.toLowerCase();

    return matchesSearch && matchesRole;
  }) || [];

  return (
    <div className="space-y-6">
      <h1 className="text-3xl font-display font-bold">Member Directory</h1>

      <div className="flex flex-col sm:flex-row gap-3">
        <div className="relative flex-1">
          <Search className="absolute left-3 top-1/2 -translate-y-1/2 h-4 w-4 text-muted-foreground" />
          <Input
            placeholder="Search by name, role, or industry..."
            value={search}
            onChange={(e) => setSearch(e.target.value)}
            className="pl-10"
          />
        </div>
        {roles.length > 0 && (
          <Select value={roleFilter} onValueChange={setRoleFilter}>
            <SelectTrigger className="w-full sm:w-[200px]">
              <SelectValue placeholder="Filter by role" />
            </SelectTrigger>
            <SelectContent>
              <SelectItem value="all">All roles</SelectItem>
              {roles.map(role => (
                <SelectItem key={role} value={role}>{role}</SelectItem>
              ))}
            </SelectContent>
          </Select>
        )}
      </div>

      {isLoading && (
        <div className="grid grid-cols-1 md:grid-cols-2 xl:grid-cols-3 gap-4">
          {Array.from({ length: 9 }).map((_, i) => (
            <Skeleton key={i} className="h-32 rounded-xl" />
          ))}
        </div>
      )}

      {error && (
        <div className="text-center py-12">
          <p className="text-muted-foreground">Unable to load the directory. Please try again later.</p>
        </div>
      )}

      {!isLoading && !error && (
        <>
          <p className="text-sm text-muted-foreground">
            {filtered.length} member{filtered.length !== 1 ? 's' : ''}
          </p>
          <div className="grid grid-cols-1 md:grid-cols-2 xl:grid-cols-3 gap-4">
            {filtered.map((m, i) => (
              <MemberCard
                key={i}
                name={m['Full Name'] || m['Name'] || ''}
                role={m['Role'] || m['Job Title'] || ''}
                focusAreas={m['Focus Areas'] ? m['Focus Areas'].split(',').map(s => s.trim()) : undefined}
                linkedIn={m['LinkedIn'] || undefined}
              />
            ))}
          </div>
          {filtered.length === 0 && (
            <div className="text-center py-12">
              <p className="text-muted-foreground">No members match your search.</p>
            </div>
          )}
        </>
      )}
    </div>
  );
}
```

**Note:** The column header names (`Full Name`, `Role`, `LinkedIn`, etc.) are read dynamically from the Google Sheet headers. The component uses fallback keys (`Name`, `Job Title`) for flexibility. These will be confirmed during implementation when we see the actual sheet headers.

- [ ] **Step 3: Verify build**

```bash
npm run build
```

Expected: Build succeeds with no errors.

- [ ] **Step 4: Commit**

```bash
git add src/components/members/directory/MemberCard.tsx src/pages/members/Directory.tsx
git commit -m "feat: implement member directory page with search and filtering"
```

---

### Task 19: Dark mode enforcement and final wiring

**Files:**
- Modify: `cpo-connect-hub/src/main.tsx`
- Modify: `cpo-connect-hub/index.html` (add `class="dark"`)

- [ ] **Step 1: Add dark class to HTML element**

The spec says "dark mode only for launch." Add `class="dark"` to the `<html>` tag in `index.html`:

```html
<html lang="en" class="dark">
```

- [ ] **Step 2: Final build and test**

```bash
npm run build
node server.js
```

Open `http://localhost:3001`:
- Landing page renders correctly in dark mode
- Login button opens modal
- Clicking "Apply to Join" goes to Fillout form
- `/members` redirects to `/` when not authenticated

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: enforce dark mode and final wiring"
```

---

## Execution Checklist

After all tasks complete:

1. Run `npm run build` — verify no build errors
2. Run `npm test` — verify rate limiter tests pass
3. Run the migration SQL against the database: `psql $DATABASE_URL -f server/migrations/001-init.sql`
4. Set up environment variables on Render
5. Deploy and verify the magic link flow end-to-end

## Dependency Order

```
Task 1 → Task 2 → Task 3 → Task 4 (sequential: server foundation)
Task 5, 6, 7 (parallel: independent services)
Task 8 (depends on: Task 2)
Task 9 (depends on: Task 5, 6, 7, 8)
Task 10 (depends on: Task 5, 8)
Task 11 (independent frontend)
Task 12 (depends on: Task 11)
Task 13 (depends on: Task 11, 12)
Task 14 (depends on: Task 11)
Task 15 (independent — shared components only)
Task 16 (depends on: Task 15)
Task 17 (depends on: Task 15, 16)
Task 18 (depends on: Task 14)
Task 19 (depends on: all)
```

**Parallel groups for subagent execution:**
- Group A: Tasks 1-4 (sequential)
- Group B: Tasks 5, 6, 7 (parallel services — can start after Task 2)
- Group C: Tasks 11, 12, 13, 14 (sequential frontend auth — independent of server)
- Group D: Tasks 15, 16, 17 (sequential insights components — independent of server)
- Group E: Tasks 8, 9, 10 (sequential auth API — depends on Group B)
- Final: Task 18, 19 (depends on all)

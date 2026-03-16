# CPO Connect Members Area Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a members-only area with magic link authentication, chat insights, and member directory to the CPO Connect Hub.

**Architecture:** Express server wraps the Vite-built React SPA, serving static files from `dist/` and API routes under `/api/`. Auth uses magic link emails (Resend) verified against a Google Sheet membership list, with tokens and sessions stored in PostgreSQL (`cpo_connect` schema on shared `truth_tone` instance). The frontend adds protected routes under `/members/` for chat insights and a searchable member directory.

**Tech Stack:** Express, pg, cookie-parser, cookie-signature, Resend, Google Sheets API v4, tsx (TypeScript runner), React Router, TanStack Query, Recharts, shadcn/ui

**Spec:** `docs/superpowers/specs/2026-03-14-members-area-design.md`

---

## Key Architecture Decisions

1. **TypeScript everywhere.** Server files use `.ts` extension, run via `tsx` in dev (`tsx --watch server.ts`) and production (`tsx server.ts`). No separate tsc compile step — tsx handles it at runtime. This matches the frontend toolchain and the global CLAUDE.md default.

2. **Auto-migration on startup.** `server/db.ts` exports a `runMigrations()` function that reads `server/migrations/001-init.sql` and executes it via `pool.query()`. Called in `server.ts` before `app.listen()`. Uses `IF NOT EXISTS` so it's idempotent and safe to run on every boot.

3. **Session expiry: 7 days** (not 30). Members area contains sensitive chat insights — shorter sessions reduce exposure from compromised cookies.

4. **In-memory rate limiting.** Acceptable for single-instance Render deployment. Add a code comment noting that rate limit state is lost on restart/redeploy and would need Redis or PostgreSQL-backed rate limiting if scaling to multiple instances.

5. **Email enumeration tradeoff.** `POST /api/auth/request` always returns HTTP 200. The response body includes a `memberStatus` field (`'sent'` or `'not_found'`) so the frontend can show different UI. This is visible in devtools but the attacker already knows the email they typed — the uniform 200 prevents scripted enumeration via status codes.

6. **Verify errors redirect, not JSON.** `GET /api/auth/verify` redirects to `/?verify=expired|invalid|error` on failure (not JSON), because users reach this via email link and expect a web page. The Navbar reads the query param and auto-opens the login modal with an error message.

7. **Health check endpoint.** `GET /health` returns `{ status: 'ok' }` — Render uses this for zero-downtime deploys.

8. **Cookie parsing.** Use `cookie-parser` middleware (not hand-rolled parsing). Auth middleware reads `req.cookies.cpo_session`, strips the `s:` prefix, unsigns with `cookie-signature`.

9. **Chat insights are static React components.** Each month is a lazy-loaded component with data baked in. A config file maps month IDs to components — adding a new month = one file + one config entry. Data is extracted from the existing HTML analysis files (`chat-analysis-jan2026.html`, `chat-analysis-feb2026.html`). Use recharts (already installed) instead of Chart.js.

10. **Google Sheets caching.** Both `lookupMember()` and `getDirectory()` cache Sheet data in memory for 5 minutes. The `/api/auth/me` endpoint has an additional per-email membership cache (5-min TTL) to re-validate membership status without hammering the Sheets API.

---

## File Map

### Server (new files — TypeScript, ES modules)

| File | Responsibility |
|------|---------------|
| `server.ts` | Express entry point — cookie-parser, API routes, static serving, SPA catch-all, `GET /health`, auto-migration on boot |
| `server/db.ts` | PostgreSQL pool with SSL config + `runMigrations()` that reads and executes `001-init.sql` |
| `server/migrations/001-init.sql` | `CREATE SCHEMA/TABLE IF NOT EXISTS` DDL for `cpo_connect` schema |
| `server/services/sheets.ts` | Google Sheets API v4 — `lookupMember(email)` and `getDirectory()`, both with 5-min cache |
| `server/services/email.ts` | Resend — `sendMagicLink({ email, token, name })` |
| `server/services/rate-limit.ts` | In-memory rate limiter factory — `createRateLimiter({ windowMs, max })` with periodic cleanup |
| `server/middleware/auth.ts` | Session cookie validation — `requireAuth` middleware, attaches `req.user` |
| `server/routes/auth.ts` | Auth endpoints: request, verify, me, logout |
| `server/routes/members.ts` | `GET /api/members/directory` |

### Frontend (new files)

| File | Responsibility |
|------|---------------|
| `src/contexts/AuthContext.tsx` | Auth state provider + `useAuth()` hook wrapping `/api/auth/me` |
| `src/components/LoginModal.tsx` | Email input modal with states: email, sent, not-member, error (including verify errors) |
| `src/components/ProtectedRoute.tsx` | Route guard — redirects to `/` with `state: { showLogin: true }` if unauthenticated |
| `src/components/members/MembersLayout.tsx` | Shared layout for `/members/*` routes |
| `src/pages/members/ChatInsights.tsx` | Month selector (chevron arrows) + lazy-loaded month component via Suspense |
| `src/pages/members/Directory.tsx` | Searchable/filterable member card grid using TanStack Query |
| `src/components/members/insights/StatCard.tsx` | Stat card (number + label + gradient text) |
| `src/components/members/insights/TrendItem.tsx` | Trend list item with colored tags |
| `src/components/members/insights/DailyVolumeChart.tsx` | Recharts BarChart — daily message volume |
| `src/components/members/insights/ContributorsChart.tsx` | Recharts PieChart (donut) — top contributors |
| `src/components/members/insights/SentimentChart.tsx` | Recharts RadarChart — sentiment snapshot |
| `src/components/members/insights/ChannelSection.tsx` | Container grouping a channel's charts + trends list |
| `src/components/members/directory/MemberCard.tsx` | Directory card — avatar initials, name, role, focus area tags, LinkedIn link |
| `src/data/insights/config.ts` | `MonthConfig[]` registry mapping month IDs to lazy-loaded components |
| `src/data/insights/feb-2026.tsx` | February 2026 multi-channel analysis (3 channels: AI, General, Leadership) |
| `src/data/insights/jan-2026.tsx` | January 2026 AI channel analysis |

### Modified files

| File | Change |
|------|--------|
| `src/App.tsx` | Wrap with `AuthProvider`, add `/members`, `/members/chat-insights`, `/members/directory` routes |
| `src/components/Navbar.tsx` | Auth-aware: public links + Login/Apply buttons when logged out; members nav + avatar dropdown when logged in. Reads `?verify=` param and `location.state.showLogin` to auto-open login modal. |
| `vite.config.ts` | Add `server.proxy: { '/api': 'http://localhost:3001' }` |
| `package.json` | Add server deps, `dev:server`, `dev:all`, `start` scripts |
| `index.html` | Add `class="dark"` to `<html>` tag |
| `.gitignore` | Append `.env` and `.env.local` |

---

## API Contracts

### `POST /api/auth/request`
- **Body:** `{ email: string }`
- **200:** `{ code: 'magic_link_sent', memberStatus: 'sent' | 'not_found' }` (always 200)
- **429:** `{ error: string, code: 'rate_limited' }` — 3/email/hour, 10/IP/minute
- **Behavior:** Normalizes email to lowercase. Looks up in Google Sheet by "Email" header, checks "Status" = "Joined". If joined: generates 256-bit random token, stores in PostgreSQL (15-min expiry), sends email via Resend. Cleanup of expired tokens/sessions on each call (best-effort, non-blocking).

### `GET /api/auth/verify?token=xxx`
- **302 → `/members`:** Valid token. Creates session (7-day expiry), sets signed httpOnly `cpo_session` cookie.
- **302 → `/?verify=expired`:** Token not found, expired, or already used.
- **302 → `/?verify=error`:** Server error.
- **Behavior:** Marks token as used. Does NOT invalidate other valid tokens for the same email.

### `GET /api/auth/me` (requires auth)
- **200:** `{ name: string, email: string, jobRole: string }`
- **401:** `{ error: string, code: 'not_authenticated' }`
- **403:** `{ error: string, code: 'membership_revoked' }` — membership no longer "Joined", session deleted.
- **Behavior:** Re-checks Google Sheet membership (with 5-min per-email cache). Deletes session if revoked.

### `POST /api/auth/logout` (requires auth)
- **200:** `{ code: 'logged_out' }` — deletes session row, clears cookie.

### `GET /api/members/directory` (requires auth)
- **200:** `{ members: Record<string, string>[] }` — each member is a key-value object from sheet headers.
- **500:** `{ error: string, code: 'service_error' }` — Sheets API unavailable.

### `GET /health`
- **200:** `{ status: 'ok' }`

---

## Database Schema

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

---

## Environment Variables

```env
DATABASE_URL=postgresql://agency:PASSWORD@dpg-d5bgrrp5pdvs73bjmq20-a/truth_tone
RESEND_API_KEY=re_xxxxx
RESEND_FROM_EMAIL=noreply@theaiexpert.ai
GOOGLE_SHEETS_CREDENTIALS=BASE64_ENCODED_SERVICE_ACCOUNT_JSON
GOOGLE_SHEET_ID=14DZ6Zp1UHg688FPTFdbJLCY6AXhgnAcnodV5KjVCAMs
SESSION_SECRET=random-secret-at-least-32-chars
MAGIC_LINK_BASE_URL=https://cpoconnect.com/api/auth/verify
```

---

## Tasks

### Task 1: Server foundation

**Files:** `package.json`, `server.ts`, `server/db.ts`, `server/migrations/001-init.sql`, `vite.config.ts`, `.env.example`, `.gitignore`

- [ ] Install deps: `npm install express cookie-parser cookie-signature pg resend googleapis tsx` and `npm install -D @types/express @types/cookie-parser @types/cookie-signature @types/pg concurrently`
- [ ] Add scripts: `"dev:server": "tsx --watch server.ts"`, `"dev:all": "concurrently \"npm run dev\" \"npm run dev:server\""`, `"start": "tsx server.ts"`
- [ ] Create `server/db.ts` — Pool with `ssl: { rejectUnauthorized: false }`, strip `sslmode=` from URL. Export `runMigrations()` that reads `001-init.sql` via `fs.readFileSync` and executes via `pool.query()`.
- [ ] Create `server/migrations/001-init.sql` — schema DDL above (all `IF NOT EXISTS`).
- [ ] Create `server.ts` — Express app: `cookie-parser()` middleware, mount API routes, `GET /health` returning `{ status: 'ok' }`, serve `dist/` static files, SPA catch-all. Call `runMigrations()` before `app.listen()`.
- [ ] Add Vite proxy: `server.proxy: { '/api': { target: 'http://localhost:3001', changeOrigin: true } }` in `vite.config.ts`.
- [ ] Create `.env.example` with all env vars documented.
- [ ] Append `.env` and `.env.local` to `.gitignore`.
- [ ] Verify: `npm run build && tsx server.ts` serves the landing page on port 3001.
- [ ] Commit.

---

### Task 2: Google Sheets service

**Files:** `server/services/sheets.ts`

- [ ] Initialize Google Sheets API v4 auth from base64-decoded `GOOGLE_SHEETS_CREDENTIALS` env var.
- [ ] `lookupMember(email: string)` — reads Sheet1, finds row by "Email" header (case-insensitive match), returns `{ email, name, status, jobRole }` or `null`. Cache the full Sheet1 data for 5 minutes to avoid repeated API calls.
- [ ] `getDirectory()` — reads "PublicMemberDirectoryMVP" tab, returns array of key-value objects from headers. 5-minute cache (separate from member cache). Returns empty array if Sheets API fails.
- [ ] Header-based column lookup (by name, not position) — resilient to column reordering.
- [ ] Commit.

---

### Task 3: Email and rate limit services

**Files:** `server/services/email.ts`, `server/services/rate-limit.ts`

- [ ] `sendMagicLink({ email, token, name })` — sends magic link email via Resend with branded HTML template. Link format: `${MAGIC_LINK_BASE_URL}?token=${token}`.
- [ ] `createRateLimiter({ windowMs, max })` — returns `{ check(key): { allowed, remaining }, reset(key) }`. The `check` function only increments the counter when the request is allowed (check before increment). Add periodic cleanup interval (2x window) with `.unref()`. Add a comment: `// NOTE: In-memory rate limits are lost on server restart. For multi-instance deployments, use Redis or PostgreSQL-backed rate limiting.`
- [ ] Write tests for the rate limiter in `src/test/rate-limit.test.ts`: allows under limit, blocks over limit, independent keys, resets after window, returns remaining count.
- [ ] Commit.

---

### Task 4: Auth middleware and routes

**Files:** `server/middleware/auth.ts`, `server/routes/auth.ts`, `server/routes/members.ts`

- [ ] `requireAuth` middleware — reads `req.cookies.cpo_session`, strips `s:` prefix, unsigns with `cookie-signature`, queries `cpo_connect.sessions` for valid unexpired session, attaches `req.user` (`{ id, email, name }`). Returns 401 on failure.
- [ ] `POST /api/auth/request` — rate limit by IP (10/min) and email (3/hr). Normalize email. Lookup via sheets service. Generate token with `crypto.randomBytes(32).toString('hex')`, 15-min expiry. Store in PostgreSQL. Send email. Always return 200 with `memberStatus`. Best-effort cleanup of expired tokens/sessions (non-blocking).
- [ ] `GET /api/auth/verify` — validate token (unused + unexpired). Mark used. Create session (7-day expiry). Set signed httpOnly cookie. Redirect 302 to `/members`. On any error: redirect to `/?verify=expired|error`.
- [ ] `GET /api/auth/me` — re-validate membership against Google Sheet (5-min per-email cache). Return `{ name, email, jobRole }`. If membership revoked: delete session, return 403.
- [ ] `POST /api/auth/logout` — delete session row, clear cookie.
- [ ] `GET /api/members/directory` — requireAuth, return `{ members }` from sheets service.
- [ ] Mount routes in `server.ts`: `/api/auth` and `/api/members`.
- [ ] Commit.

---

### Task 5: Frontend auth (context, modal, navbar, routes)

**Files:** `src/contexts/AuthContext.tsx`, `src/components/LoginModal.tsx`, `src/components/ProtectedRoute.tsx`, `src/components/Navbar.tsx`, `src/components/members/MembersLayout.tsx`, `src/App.tsx`

- [ ] `AuthContext` — provides `{ user, isLoading, isAuthenticated, login(email), logout(), checkAuth() }`. On mount, calls `/api/auth/me`. `login()` calls `POST /api/auth/request`, throws on 429.
- [ ] `LoginModal` — dialog with 4 states: `email` (input form), `sent` (check your inbox), `not-member` (apply to join CTA + try different email), `error` (error message + try again). Accepts optional `verifyError` prop for magic link failures.
- [ ] `ProtectedRoute` — if loading: spinner. If unauthenticated: `<Navigate to="/" state={{ showLogin: true }} />`. Otherwise: render children.
- [ ] `MembersLayout` — `<Navbar />` + `<main className="container py-8">{children}</main>`.
- [ ] Update `Navbar.tsx`:
  - **Unauthenticated on landing:** public hash links (Manifesto, Channels, Events, Founders) + Login button (ghost, opens modal) + Apply to Join button (primary, external link to Fillout form).
  - **Authenticated on landing:** same public links + "Members Area" button linking to `/members/chat-insights`.
  - **Authenticated on members pages:** member links (Chat Insights, Directory) with active state + avatar dropdown with sign out.
  - Read `location.state?.showLogin` → auto-open modal. Read `searchParams.get('verify')` → set verify error and open modal, then clean URL.
  - Mobile hamburger menu with same states.
- [ ] Update `App.tsx` — wrap with `AuthProvider`. Add routes: `/members` → redirect to `/members/chat-insights`, `/members/chat-insights`, `/members/directory` (all wrapped in `ProtectedRoute` + `MembersLayout`).
- [ ] Verify build succeeds.
- [ ] Commit.

---

### Task 6: Chat insights components and data

**Files:** `src/components/members/insights/*.tsx`, `src/data/insights/config.ts`, `src/data/insights/feb-2026.tsx`, `src/data/insights/jan-2026.tsx`

- [ ] Create shared chart components using recharts (already installed):
  - `StatCard` — number with gradient text + label. Props: `{ label, value, gradient }`.
  - `TrendItem` — rank number + title + description + colored tags. Tag variants: hot (red), green, gold, amber, pink, blue.
  - `DailyVolumeChart` — recharts `BarChart` with dark-themed tooltips.
  - `ContributorsChart` — recharts `PieChart` (donut style) with legend.
  - `SentimentChart` — recharts `RadarChart` with polar grid.
  - `ChannelSection` — groups DailyVolume + Contributors side-by-side, then Sentiment, then Trends list.
- [ ] Define `ChannelData` interface: `{ name, dailyVolume[], contributors[], sentiment[], trends[], chartColor, sentimentColor }`.
- [ ] Create `config.ts` with `MonthConfig` type and `months` array (newest first). Each entry: `{ id: '2026-02', label: 'February 2026', component: lazy(() => import('./feb-2026')) }`.
- [ ] Create `feb-2026.tsx` — extract data from `chat-analysis-feb2026.html` Chart.js datasets (labels + data arrays for all 3 channels: AI, General, Leadership). Use shadcn `Tabs` for channel switching. Stats row at top.
- [ ] Create `jan-2026.tsx` — extract data from `chat-analysis-jan2026.html` (AI channel only). No tabs needed.
- [ ] Commit.

---

### Task 7: Chat insights page

**Files:** `src/pages/members/ChatInsights.tsx`

- [ ] Month selector with left/right chevron buttons. `months` array is newest-first: "previous" goes to older (higher index), "next" goes to newer (lower index). Disable buttons at boundaries.
- [ ] Render current month's component via `React.Suspense` with loading spinner fallback.
- [ ] Commit.

---

### Task 8: Member directory page

**Files:** `src/components/members/directory/MemberCard.tsx`, `src/pages/members/Directory.tsx`

- [ ] `MemberCard` — shadcn Card with Avatar (initials fallback), name, role, focus area tags (Badge), LinkedIn icon link. Hover effect on card border.
- [ ] `Directory` page — fetch from `/api/members/directory` via TanStack Query. Search input (filters by name, role, industry — client-side). Role filter dropdown (populated from unique roles in data). Responsive grid: `xl:grid-cols-3 md:grid-cols-2 grid-cols-1`. Skeleton loading state. Error state. Empty state.
- [ ] Column header names from the Google Sheet are dynamic — use fallback keys (e.g., `m['Full Name'] || m['Name']`). Confirm actual headers during implementation.
- [ ] Commit.

---

### Task 9: Dark mode and final verification

**Files:** `index.html`

- [ ] Add `class="dark"` to `<html>` tag in `index.html`.
- [ ] Run `npm run build` — verify no errors.
- [ ] Run `npm test` — verify rate limiter tests pass.
- [ ] Run `npm run dev:all` — verify landing page, login modal, and Vite proxy all work.
- [ ] Commit.

---

## Dependency Order

```
Task 1 (foundation) — must be first
Task 2 (sheets) — after Task 1
Task 3 (email + rate limit) — after Task 1, parallel with Task 2
Task 4 (auth routes) — after Tasks 2 + 3
Task 5 (frontend auth) — after Task 1, parallel with Tasks 2-4
Task 6 (insights components) — independent, parallel with Tasks 2-5
Task 7 (insights page) — after Task 6
Task 8 (directory page) — after Task 5
Task 9 (final) — after all
```

**Parallel groups for subagent execution:**
- **Group A** (sequential): Task 1 → Task 2, Task 3 (parallel) → Task 4
- **Group B** (sequential): Task 5 (can start after Task 1)
- **Group C** (sequential): Task 6 → Task 7 (fully independent)
- **Group D**: Task 8 (after Task 5)
- **Final**: Task 9 (after all)

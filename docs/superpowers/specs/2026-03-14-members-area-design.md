# CPO Connect — Members Area & Render Deployment

**Date:** 2026-03-14
**Status:** Approved

## Overview

Add a members-only area to the CPO Connect Hub (React + Vite + Tailwind + shadcn/ui) with magic link authentication backed by a Google Spreadsheet as the membership source of truth. Deploy as a Render Web Service ($7/month) in Frankfurt, using the shared `truth_tone` PostgreSQL instance with a dedicated `cpo_connect` schema.

## Architecture

### Deployment

- **Platform:** Render Web Service (Frankfurt region — must match PostgreSQL)
- **Runtime:** Node.js + Express
- **Build:** `npm run build` (Vite builds React to `dist/`)
- **Start:** `node server.js` (Express serves `dist/` + API routes)
- **Database:** Shared `truth_tone` PostgreSQL instance → `cpo_connect` schema (isolated from other projects)

### System Components

1. **React SPA** — Vite + Tailwind + shadcn/ui, served as static build from Express
2. **Express API** — auth endpoints, Google Sheets proxy, member directory
3. **PostgreSQL** — `cpo_connect` schema for magic link tokens and sessions
4. **Resend** — sends magic link emails from `theaiexpert.ai`
5. **Google Sheets API v4** — reads membership data (source of truth) and directory data

## Authentication — Magic Link Flow

### Flow

1. User clicks **Login** button in navbar (was "Join")
2. Login modal opens — user enters email
3. `POST /api/auth/request` — server normalizes email to lowercase, looks up in Google Sheet by header name ("Email"), checks "Status" column = "Joined"
4. **If Joined:** generate cryptographically random token (`crypto.randomBytes(32)`), store in PostgreSQL, send magic link email via Resend
5. **If NOT Joined:** return error, frontend shows "Apply to Join" with link to `https://cpoconnect.fillout.com/application`
6. User clicks magic link → `GET /api/auth/verify?token=xxx`
7. Token validated → session created in PostgreSQL → signed httpOnly session cookie set → `302` redirect to `/members` (replaces verify URL in browser history)
8. **If token expired/invalid:** show error with option to request a new link

### Token & Session Rules

- Magic link tokens: cryptographically random (256 bits), expire in **15 minutes**, single-use
- Requesting a new magic link does NOT invalidate prior unexpired tokens (both remain valid until used or expired)
- Sessions expire in **30 days**
- Sessions stored server-side in PostgreSQL, referenced by signed httpOnly cookie (using `cookie-signature` with `SESSION_SECRET`)
- Logout clears cookie and deletes session row
- **Membership re-validation:** `/api/auth/me` re-checks the Google Sheet on each call (with 5-min cache) to detect revoked members. If status is no longer "Joined", session is deleted and 401 returned.
- **Cleanup:** expired tokens and sessions are deleted on a best-effort basis — a cleanup query runs on each `/api/auth/request` call to prune rows where `expires_at < NOW()`

### Rate Limiting

- `POST /api/auth/request`: max **3 requests per email per hour**, max **10 requests per IP per minute**
- Rate limits enforced in-memory (acceptable for single-instance deployment)
- Returns `429 Too Many Requests` when exceeded

### Email Handling

- All email comparisons are **case-insensitive** (both sides lowercased before matching)
- Email is normalized to lowercase before storage in tokens and sessions

## API Endpoints

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/api/auth/request` | No | Takes email, checks Sheet, sends magic link |
| GET | `/api/auth/verify` | No | Validates token, creates session, 302 redirects to `/members` |
| GET | `/api/auth/me` | Yes | Returns current user `{ name, email, jobRole }` |
| POST | `/api/auth/logout` | Yes | Clears session |
| GET | `/api/members/directory` | Yes | Returns member directory data |

### Error Response Contract

All API errors return JSON: `{ error: string, code: string }`

| Endpoint | Status | Code | Meaning |
|----------|--------|------|---------|
| `POST /api/auth/request` | 200 | `magic_link_sent` | Email sent (always 200 even if not found, to prevent email enumeration) |
| `POST /api/auth/request` | 429 | `rate_limited` | Too many requests |
| `GET /api/auth/verify` | 302 | — | Valid token, redirect to `/members` |
| `GET /api/auth/verify` | 400 | `token_invalid` | Token not found, expired, or already used |
| `GET /api/auth/me` | 200 | — | Returns user object |
| `GET /api/auth/me` | 401 | `not_authenticated` | No valid session |
| `GET /api/auth/me` | 403 | `membership_revoked` | Session existed but membership no longer "Joined" |
| `POST /api/auth/logout` | 200 | `logged_out` | Session cleared |
| `GET /api/members/directory` | 401 | `not_authenticated` | No valid session |

**Note:** `POST /api/auth/request` always returns 200 regardless of whether the email was found. This prevents attackers from enumerating which emails are members. The frontend shows different UI states, but the distinction is communicated securely.

## Database Schema — `cpo_connect`

```sql
CREATE SCHEMA IF NOT EXISTS cpo_connect;

CREATE TABLE cpo_connect.magic_link_tokens (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email TEXT NOT NULL,
  token TEXT UNIQUE NOT NULL,
  expires_at TIMESTAMPTZ NOT NULL,
  used BOOLEAN DEFAULT FALSE,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE cpo_connect.sessions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email TEXT NOT NULL,
  name TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  expires_at TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_tokens_token ON cpo_connect.magic_link_tokens(token);
CREATE INDEX idx_tokens_email ON cpo_connect.magic_link_tokens(email);
CREATE INDEX idx_sessions_id ON cpo_connect.sessions(id);
```

## Google Sheets Integration

- **Sheet ID:** `14DZ6Zp1UHg688FPTFdbJLCY6AXhgnAcnodV5KjVCAMs`
- **Auth lookup:** Sheet1 tab — match by header "Email" (col D), check header "Status" (col Q) = "Joined", read header "Full Name" (col C). Column lookup uses **header names, not positional letters** — if columns are reordered or inserted, the system still works.
- **Directory data:** PublicMemberDirectoryMVP tab — read all rows, cache in memory for 5 minutes. Cache is cleared on restart; first request after deploy will be slower.
- **Sheets API unavailability:** If the Google Sheets API is down, auth requests fail gracefully with a "Service temporarily unavailable" message. Directory page shows cached data if available, otherwise shows an error state.
- **Authentication:** Google Sheets API v4 with service account credentials

## Frontend — UX Design

### Navbar Changes

- **Unauthenticated:** "Login" button (ghost style) + "Apply to Join" button (primary gradient)
- **Authenticated:** Members nav items (Home, Chat Insights, Directory) + user avatar with initials + logout dropdown

### Login Modal

- Overlay modal on the landing page (not a separate route)
- Email input → "Send Magic Link" button
- Success state: "Check your inbox" with email shown, 15-min expiry note
- Not-a-member state: "We don't recognise that email" with "Apply to Join" CTA and "Try a different email" link

### Protected Routes

- `/members` — client-side redirect (`Navigate`) to `/members/chat-insights`
- `/members/chat-insights` — monthly chat analysis
- `/members/directory` — searchable member directory
- Unauthenticated access to `/members/*` redirects to login modal
- Express catch-all: `app.get('*', (req, res) => res.sendFile('dist/index.html'))` ensures SPA routing works on direct navigation / page refresh

### Chat Insights Page (`/members/chat-insights`)

- Month selector — arrows to browse past months, defaults to latest (Feb 2026)
- Existing HTML chat analysis content ported to React components
- Preserves: channel tabs (AI, General, Leadership), stats cards, daily volume charts, contributor doughnuts, trend lists, sentiment polar charts
- Each month is a static React component with data baked in
- Config file maps months to components — adding a new month = one file + one config line

### Member Directory Page (`/members/directory`)

- Data sourced from PublicMemberDirectoryMVP Google Sheet tab
- Searchable by name, role, or industry (client-side filtering)
- Filter dropdown by role
- Card grid layout: avatar (initials), name, role, focus area tags, LinkedIn link
- Responsive: 3 columns desktop, 2 tablet, 1 mobile

## Visual Design

- **Matches existing site:** dark gradient background, Space Grotesk + Inter fonts, purple/blue/green palette, glassmorphic cards
- **Dark mode only** for launch — light/dark toggle deferred to fast follow
- Members area feels like an "unlocked" deeper layer of the same site

## Environment Variables (Render)

| Variable | Description |
|----------|-------------|
| `DATABASE_URL` | Internal PostgreSQL connection string (Frankfurt) |
| `RESEND_API_KEY` | Resend API key for sending emails |
| `RESEND_FROM_EMAIL` | e.g., `noreply@theaiexpert.ai` |
| `GOOGLE_SHEETS_CREDENTIALS` | Service account JSON (base64 encoded) |
| `GOOGLE_SHEET_ID` | `14DZ6Zp1UHg688FPTFdbJLCY6AXhgnAcnodV5KjVCAMs` |
| `SESSION_SECRET` | Secret for signing session cookies |
| `MAGIC_LINK_BASE_URL` | e.g., `https://cpoconnect.com/api/auth/verify` |

## Render Configuration

- **Region:** Frankfurt (must match PostgreSQL instance)
- **Instance:** Web Service
- **Build command:** `npm run build`
- **Start command:** `node server.js`
- **Internal DB hostname:** `dpg-d5bgrrp5pdvs73bjmq20-a`
- **SSL:** Strip `sslmode=` from URL, use `ssl: { rejectUnauthorized: false }` on pg.Pool. This disables cert verification but is safe because Render's internal network is trusted and the connection never leaves their infrastructure.

## Local Development

- Vite dev server runs on port 8080 (existing config)
- Express API server runs on port 3001
- Add Vite `server.proxy` config to forward `/api/*` requests to `http://localhost:3001`
- `server.js` uses ES modules (matching existing `"type": "module"` in package.json)
- Add a `dev:server` script to run Express in watch mode alongside Vite

## Schema Migration

- SQL DDL is applied as a **one-time manual migration** (run via psql or a seed script)
- Not run on every server boot — `CREATE SCHEMA IF NOT EXISTS` is safe but the table creation should be idempotent
- Migration file stored at `server/migrations/001-init.sql` for reference

## Future Extensibility

The following are explicitly deferred but the architecture supports them:
- **Light/dark mode toggle** — deferred, will need a theme provider setup
- **Chat with history AI** — add a `/api/chat` endpoint + new members page
- **Event recordings** — add a "Recordings" tab to members nav
- **Event photos** — add a "Gallery" tab
- **Substack links** — add a "Resources" tab
- **Member profile CRUD** — add write endpoints to the API for individual member updates
- **Full database migration** — if the spreadsheet is outgrown, swap the Google Sheets reads for PostgreSQL queries; auth layer stays the same

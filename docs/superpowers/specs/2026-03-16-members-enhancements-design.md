# CPO Connect — Members Area Enhancements

**Date:** 2026-03-16
**Status:** Draft
**Depends on:** [Members Area & Render Deployment](2026-03-14-members-area-design.md) (implemented), [Profile Enrichment Plan](../plans/2026-03-15-profile-enrichment.md) (implemented)
**Design reference:** [Original Lovable site](https://id-preview--8ae9007e-19de-4279-9826-1e09f5355ed1.lovable.app/)

## Overview

Five enhancements to the deployed CPO Connect members area:

1. **Persistent auth** — auto-detect existing sessions so returning users see "Members Area" in the navbar without re-logging in
2. **Member directory improvements** — add contact info (email, phone), expandable card details, and better search UX
3. **Richer LinkedIn enrichment** — extract role and current organisation in addition to bio and skills
4. **Member headshots** — use existing founder photos from `public/founders/` in directory cards, add Gravatar/UI Avatars fallback for other members
5. **Light/dark mode toggle** — add a theme toggle persisted via localStorage

All changes target the active repo at `/Users/eschwaa/Projects/cpo-connect-hub`, deployed to Render (Frankfurt).

---

## 1. Persistent Auth

### Problem

The current `AuthContext` does not check session validity on app mount. `checkAuth()` is only called when the user navigates to a `/members/*` route via `ProtectedRoute`. This means:

- A returning user with a valid 7-day session cookie lands on the homepage and sees "Login" + "Apply to Join" instead of "Members Area"
- They must navigate to `/members/*` or click Login (which triggers `ProtectedRoute` → `checkAuth()`) to restore their session

### Solution

Use a **session hint cookie** to avoid unnecessary `/api/auth/me` calls for users who have never logged in.

**How it works:**
- When a session is created (in `GET /api/auth/verify`), set an additional non-httpOnly cookie `cpo_has_session=1` alongside the httpOnly `cpo_session` cookie. Same expiry (7 days), same path.
- When a session is destroyed (logout or membership revoked), clear both cookies.
- On app mount, the `AuthContext` checks `document.cookie` for `cpo_has_session`. If absent, skip the `/api/auth/me` call entirely — the user has never logged in (or their session expired).
- If `cpo_has_session` is present, call `/api/auth/me` to validate the session and populate user state.

This avoids:
- Wasted network requests for first-time/unauthenticated visitors
- 401 errors in the browser DevTools console (the current code intentionally avoided these — see `AuthContext.tsx` comment at line 51-53)
- Navbar flash on the landing page for unauthenticated users

### Changes

#### Backend — Auth Verify (`server/routes/auth.ts`)

In the `GET /api/auth/verify` handler, after setting the httpOnly session cookie, also set:

```typescript
res.cookie('cpo_has_session', '1', {
  httpOnly: false,   // readable by JS
  secure: true,
  sameSite: 'lax',
  maxAge: 7 * 24 * 60 * 60 * 1000,  // 7 days, matching session expiry
  path: '/',
})
```

In `POST /api/auth/logout` and in the `/api/auth/me` membership-revoked path, clear it:

```typescript
res.clearCookie('cpo_has_session', { path: '/' })
```

#### AuthContext (`src/contexts/AuthContext.tsx`)

- Add a `useEffect` that runs once on mount:
  - Check `document.cookie.includes('cpo_has_session')` — if absent, set `hasChecked = true` immediately (no fetch)
  - If present, call `checkAuth()` to validate session
- Set `isLoading = true` initially (currently `false`) only when the hint cookie is detected, so the navbar doesn't flash between states
- Keep `hasChecked` flag — once the mount check completes (success or 401), set `hasChecked = true` and `isLoading = false`
- On 401/403: silently set `user = null`, no error UI — this is the normal unauthenticated state. Also clear the hint cookie client-side to prevent retries on next load.

#### Navbar (`src/components/Navbar.tsx`)

- While `isLoading && !hasChecked`: render a minimal navbar without Login/Members buttons (prevents flash)
- Once `hasChecked`:
  - **Authenticated + landing page:** Show "Members Area" button (link to `/members`)
  - **Authenticated + members area:** Show member nav links + avatar dropdown (existing behaviour)
  - **Unauthenticated:** Show "Login" + "Apply to Join" (existing behaviour)

#### Performance

- For unauthenticated visitors: zero network overhead (cookie check is synchronous)
- For returning members: one lightweight `/api/auth/me` call (session cookie → one PostgreSQL lookup → user JSON)
- The 5-minute membership cache already prevents excessive Google Sheets calls
- On Render cold starts, the auth check adds ~200ms — acceptable since the navbar skeleton prevents layout shift

---

## 2. Member Directory Improvements

### 2a. Contact Information on Cards

#### Problem

Directory cards show name, role, org, sector, focus areas, and LinkedIn link — but no email or phone. Members need to contact each other directly.

#### Data Source

The `PublicMemberDirectoryMVP` Google Sheet tab needs two columns added (or confirmed present):
- **"Email"** — member's email address
- **"Phone"** — member's phone number (optional)

These columns should be **opt-in**: only members who want their contact info visible in the directory should have values in these columns. The sheet is the source of truth — if a column is empty, the UI simply doesn't show a contact icon.

#### Frontend Changes — `MemberCard.tsx`

Add contact icons below the LinkedIn icon:
- **Email:** `Mail` icon from lucide-react. On tap: `mailto:` link. Show only if `email` field is present.
- **Phone:** `Phone` icon from lucide-react. On tap: `tel:` link. Show only if `phone` field is present.

Icon row layout: LinkedIn | Email | Phone (horizontally, with consistent spacing).

#### Data Flow

1. Google Sheet → `GET /api/members/directory` already returns all columns from `PublicMemberDirectoryMVP` as key-value pairs
2. `normalizeMember()` in `Directory.tsx` maps `m["Email"]` → `email` and `m["Phone"]` → `phone`
3. `MemberCard` receives and renders the new fields

### 2b. Expandable Card Detail View

#### Problem

Cards show a summary. Members want to see the full profile (bio, skills, areas of interest, all contact info) without leaving the directory page.

#### Design

**Tap/click a card → card expands inline** (not a modal, not a new route).

Expanded state shows:
- All summary fields (name, role, org, sector, focus areas) — already visible
- **Bio** — full text, up to ~500 chars
- **Skills** — comma-separated tags (same style as focus area tags but different colour)
- **Areas of Interest** — tag list
- **Location**
- **Email** (if present) — displayed as text, clickable
- **Phone** (if present) — displayed as text, clickable
- **LinkedIn** — displayed as full URL, clickable

Collapse: tap the card again, or tap a "Close" (X) button in the expanded header.

#### Animation

- Use `framer-motion` `AnimatePresence` for smooth expand/collapse (`framer-motion` v12 is already installed and used by `FoundersSection.tsx`)

#### State

- Single `expandedMemberId: string | null` state in `Directory.tsx`
- Only one card expanded at a time
- Expanding a new card collapses the previous one

#### Responsive

- On mobile (1-column layout), expanded card takes full width — natural
- On desktop (3-column grid), expanded card stays in its grid cell — the row height grows to accommodate. No spanning across columns (keeps layout stable).

### 2c. Search Improvements

The existing search already covers name, role, org, sector, location, bio, skills, focusAreas, areasOfInterest (per the profile enrichment plan). Additional improvements:

- **Add email to search** — members should be findable by email
- **Search result count** — enhance the existing count display (currently shows `N members`) to show "Showing X of Y members" when a filter/search is active
- **Clear search button** — add an X icon inside the search input when non-empty

---

## 3. Richer LinkedIn Enrichment

### Problem

The current enrichment service (`server/services/enrichment.ts`) extracts only `bio` and `skills` from LinkedIn. The Claude prompt does not ask for role or current organisation. Members who enrich their profile still have stale/empty role and org fields.

### Solution

Update the Claude Haiku prompt to also extract:
- **role** — current job title
- **current_org** — current employer/organisation

Update the DB write in the enrichment endpoint to set these fields alongside bio and skills.

### Changes

#### Enrichment Service (`server/services/enrichment.ts`)

Update the `EnrichmentResult` interface:

```typescript
interface EnrichmentResult {
  bio: string
  skills: string
  role: string
  currentOrg: string
}
```

Update the Claude prompt to request:

```
1. **bio**: ... (existing)
2. **skills**: ... (existing)
3. **role**: Extract their current job title. If not clearly stated, return an empty string.
4. **currentOrg**: Extract their current employer or organisation. If not clearly stated, return an empty string.

Respond in JSON format only:
{"bio": "...", "skills": "...", "role": "...", "currentOrg": "..."}
```

#### Enrichment Endpoint (`server/routes/members.ts`)

**Overwrite policy:** Enrichment always writes `bio` and `skills`. It writes `role` and `current_org` only if the extracted values are non-empty — this prevents enrichment from clearing manually-entered data.

Build the SET clause dynamically (same pattern as the existing PUT /profile endpoint):

```typescript
// Always write bio + skills; conditionally write role + current_org
const updates: Record<string, string> = {
  bio: enrichment.bio,
  skills: enrichment.skills,
}
if (enrichment.role) updates.role = enrichment.role
if (enrichment.currentOrg) updates.current_org = enrichment.currentOrg
// Note: TypeScript interface uses camelCase (currentOrg) but DB column is snake_case (current_org)
```

Also set `profile_enriched = TRUE`, `enrichment_source = 'linkedin'`, and `updated_at = NOW()` in the same UPDATE.

### No Migration Required

The `role` and `current_org` columns already exist in `cpo_connect.member_profiles` (created in migration 002).

### Data Source Note

Enrichment writes `role` and `current_org` to **PostgreSQL** (`member_profiles`), but the directory page reads from the **Google Sheet** (`PublicMemberDirectoryMVP` tab). These are separate data sources. Enriched role/org will appear on the **Profile page** immediately, but will **not** appear in the **directory** until the data sources are unified (out of scope for this spec — see "Out of Scope" section). This is acceptable for MVP: the Profile page is the authoritative view of enriched data.

---

## 4. Member Headshots

### Problem

All avatars are initials-only (generated from name). The community wants real photos, especially for the founding members visible on the landing page and in the directory.

### Existing Assets

The active repo already has all 9 founder headshots at `public/founders/` (lowercase-hyphenated filenames):

| File | Founder |
|------|---------|
| `erik-schwartz.jpeg` | Erik Schwartz |
| `glynn-williams.jpeg` | Glynn Williams |
| `gokul-raju.png` | Gokul Raju |
| `gregor-young.jpeg` | Gregor Young |
| `james-engelbert.jpeg` | James Engelbert |
| `jessie-rushton.jpeg` | Jessie Rushton |
| `sarah-baker-white.jpeg` | Sarah Baker-White |
| `scott-weiss.jpeg` | Scott Weiss |
| `shiv-khuti.jpeg` | Shiv Khuti |

`FoundersSection.tsx` already uses these images via public URL paths (e.g., `"/founders/erik-schwartz.jpeg"`) with `rounded-full object-cover` styling and `framer-motion` animations. **No changes needed to the founders section or to copy any assets.**

### Strategy — Two Tiers

#### Tier 1: Founders (already done for landing page; extend to directory)

The landing page `FoundersSection.tsx` already displays real headshots. The remaining work is extending this to the **directory cards** — when a directory member matches a founder name, show their headshot instead of initials.

#### Tier 2: General Members (layered fallback)

For non-founder members, photos are resolved in priority order:

1. **Uploaded photo URL** (future) — not in this spec, but the schema supports it via `photo_url` column
2. **Gravatar** — based on member email hash (many professionals have Gravatar)
3. **UI Avatars** — deterministic, colourful generated avatar from name (external service, no API key)
4. **Initials** — current fallback (stays as final fallback if external services are unavailable)

**Note:** LinkedIn photo scraping (`og:image` extraction) is deliberately excluded. LinkedIn aggressively blocks unauthenticated scraping and frequently returns placeholder images or login walls. The effort-to-reliability ratio is poor. If members want their LinkedIn photo, they can upload it manually in a future profile-upload feature.

### 4a. Gravatar Fallback (Server-Side)

Generate Gravatar URLs server-side in the `/api/members/directory` response. Compute the URLs **once when the directory cache is populated** (not on every request):

```typescript
import { createHash } from 'node:crypto'

function gravatarUrl(email: string, size = 80): string {
  const hash = createHash('md5').update(email.trim().toLowerCase()).digest('hex')
  // d=404 means return 404 if no Gravatar exists (client falls through to next level)
  return `https://www.gravatar.com/avatar/${hash}?s=${size}&d=404`
}
```

Add a `gravatarUrl` field to each member object in the directory response. No new client-side dependency needed.

**Privacy note:** Only generate `gravatarUrl` for members who have opted in to showing their email in the directory (i.e., the "Email" column in the Google Sheet is non-empty). For members who chose not to share their email, do not generate a Gravatar URL — the MD5 hash is derived from the email and would leak email-derived data. These members fall through to UI Avatars → initials.

### 4b. UI Avatars Fallback

If Gravatar returns 404, use UI Avatars (client-side, generated from name):

```
https://ui-avatars.com/api/?name=John+Doe&background=7c3aed&color=fff&size=80&bold=true
```

Uses CPO Connect brand purple (`#7c3aed`) as background. Free external service, no API key, no dependencies.

**Timeout handling:** `ui-avatars.com` has no SLA. If it's slow (not down, just slow), the `<img>` tag won't fire `onError` until the request times out, leaving a broken avatar. Mitigation: the `MemberAvatar` component should always render initials as a **CSS background layer** behind the `<img>`, so there's always visual content regardless of load state. When the image loads, it covers the initials. If it fails, initials are already visible.

### 4c. MemberAvatar Component

Build `MemberAvatar` as a wrapper around the existing `Avatar`, `AvatarImage`, and `AvatarFallback` Radix primitives from `src/components/ui/avatar.tsx` (already used by `Navbar.tsx` and `MemberCard.tsx`):

```tsx
interface MemberAvatarProps {
  name: string
  founderPhoto?: string   // public URL for founders (e.g., "/founders/erik-schwartz.jpeg")
  gravatarUrl?: string    // pre-computed on server
  size?: number           // px, default 40
  className?: string
}
```

Fallback chain: `founderPhoto` → `gravatarUrl` → UI Avatars URL → initials (via `AvatarFallback`).

Use `AvatarImage` with its built-in `onLoadingStatusChange` callback to detect load failures and cascade through the chain. `AvatarFallback` renders initials as the always-present background (visible while images load or if all fail).

**Prerequisite:** Extract `getInitials()` into a shared utility (e.g., `src/lib/utils.ts`). It's currently duplicated in `Navbar.tsx` and `MemberCard.tsx` with slightly different implementations.

#### Founder Photo Matching

In the directory, match member names to founder photo URLs using a static map:

```typescript
// Normalise name for matching: lowercase, collapse whitespace, trim
function normaliseForMatch(name: string): string {
  return name.trim().replace(/\s+/g, ' ').toLowerCase()
}

const FOUNDER_PHOTOS: Record<string, string> = {
  'erik schwartz': '/founders/erik-schwartz.jpeg',
  'glynn williams': '/founders/glynn-williams.jpeg',
  // ...
}

// Usage: FOUNDER_PHOTOS[normaliseForMatch(member.name)]
```

This normalised matching avoids fragile exact-string comparisons (handles extra whitespace, casing differences in the Google Sheet).

### 4d. Where Avatars Appear

| Location | Current | After |
|----------|---------|-------|
| Navbar user dropdown | Initials | `MemberAvatar` (small, 32px) |
| Directory cards | Initials | `MemberAvatar` (medium, 48px) |
| Expanded card detail | N/A | `MemberAvatar` (large, 80px) |
| Landing page founders section | Real headshots (already done) | No change |

### 4e. Migration for Future Photo Uploads

```sql
-- 004-photo-url.sql
ALTER TABLE cpo_connect.member_profiles
  ADD COLUMN IF NOT EXISTS photo_url TEXT DEFAULT '';
```

This column is not used by the current spec but prepares for a future profile photo upload feature. The `MemberAvatar` component can be extended to check `photoUrl` (from DB) before falling through to Gravatar.

---

## 5. Light/Dark Mode Toggle

### Current State

The site defaults to dark mode via `class="dark"` on the `<html>` element. There is no toggle — dark mode is hardcoded. The original spec explicitly deferred this feature.

### Solution

Add a theme toggle that switches between light and dark mode, persisted via `localStorage`.

### Changes

#### Theme Provider (`src/contexts/ThemeContext.tsx` — new file)

Create a minimal theme context:

```typescript
type Theme = 'light' | 'dark'

interface ThemeContextValue {
  theme: Theme
  toggleTheme: () => void
}
```

On mount:
1. Read `localStorage.getItem('cpo-theme')`
2. If no stored preference, default to `'dark'` (preserves current behaviour)
3. Apply by setting/removing the `dark` class on `document.documentElement`

On toggle:
1. Flip the theme
2. Update `document.documentElement.classList`
3. Persist to `localStorage.setItem('cpo-theme', newTheme)`

Wrap the app with `<ThemeProvider>` in `App.tsx` (or `main.tsx`).

#### Toggle Button — Navbar

Add a `Sun`/`Moon` icon button (from lucide-react) to the navbar:
- **Dark mode active:** show `Sun` icon (click to switch to light)
- **Light mode active:** show `Moon` icon (click to switch to dark)
- Position: right side of navbar, before the user avatar dropdown (authenticated) or before the Login button (unauthenticated)
- Style: ghost button, consistent with existing navbar button styles

#### CSS / Tailwind Considerations

The site already uses Tailwind's `dark:` variant classes throughout. Since `class="dark"` is on `<html>`, Tailwind's `darkMode: 'class'` strategy is already in effect. Switching to light mode simply removes the `dark` class — all `dark:` prefixed styles stop applying and the default (light) styles take over.

**Verify:** all existing components must have acceptable light mode styles. The dark gradient backgrounds, glassmorphic cards, and text colours may need light-mode counterparts. Key areas to audit:
- `index.css` — CSS custom properties (hsl values) for light vs dark themes
- Glassmorphic cards — `bg-card/50`, `border-border/50` need light equivalents
- Gradient backgrounds — dark gradients may need light alternatives
- Text contrast — ensure readability in both modes

#### No Backend Changes Required

Theme preference is entirely client-side (localStorage + CSS class toggle).

---

## Database Changes Summary

### New Migration: `004-photo-url.sql`

```sql
ALTER TABLE cpo_connect.member_profiles
  ADD COLUMN IF NOT EXISTS photo_url TEXT DEFAULT '';
```

This column is for future photo upload support. Not written by enrichment in this spec.

### Updated Columns Used by Enrichment

| Column | Current enrichment | After this spec |
|--------|-------------------|-----------------|
| `bio` | Written | Written |
| `skills` | Written | Written |
| `role` | Not touched | Written (if non-empty) |
| `current_org` | Not touched | Written (if non-empty) |

---

## API Changes Summary

| Endpoint | Change |
|----------|--------|
| `GET /api/auth/verify` | Set `cpo_has_session=1` non-httpOnly cookie alongside session cookie |
| `POST /api/auth/logout` | Clear `cpo_has_session` cookie |
| `GET /api/auth/me` | Clear `cpo_has_session` cookie on membership-revoked (403) path |
| `GET /api/members/directory` | Add `gravatarUrl` field to each member in response (only for members with opt-in email) |
| `POST /api/members/profile/enrich` | Extract + write `role`, `current_org` in addition to `bio`, `skills` |

No new endpoints required.

---

## Google Sheets Changes

The `PublicMemberDirectoryMVP` tab needs these columns (added by the sheet admin, not code):
- **"Email"** — opt-in, for directory contact info
- **"Phone"** — opt-in, for directory contact info

If these columns don't exist in the sheet, the directory simply won't show contact icons for those members. No code error — `m["Email"]` returns `undefined`, which is handled.

---

## Implementation Order

```
1. Persistent auth (AuthContext + Navbar + hint cookie)  — frontend + small backend change
2. Richer enrichment (service + endpoint)                — standalone backend change
3. Directory contact info + expandable cards             — standalone frontend
4. MemberAvatar component + Gravatar integration         — standalone frontend + small backend change
5. Light/dark mode toggle                                — standalone frontend
6. Photo URL migration (004)                             — standalone, future-proofing
```

Parallelisable: #1, #2, #3, and #5 are all independent of each other. #4 is independent but benefits from #3 being done first (shared directory card changes).

---

## Out of Scope

- **Photo upload UI** — future feature; `photo_url` column supports it but no upload endpoint in this spec
- **LinkedIn photo scraping** — unreliable due to aggressive blocking; excluded deliberately
- **Phone number validation** — the sheet is the source of truth; no server-side format validation
- **Email obfuscation** — emails are shown to authenticated members only (directory is behind auth). No additional obfuscation needed for MVP
- **Directory data source unification** — the directory reads from Google Sheets while enrichment writes to PostgreSQL. Merging these data sources so enriched role/org appears in the directory is deferred to a future spec.

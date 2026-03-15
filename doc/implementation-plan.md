# Kindling — Implementation Plan

## Permission Allowlist

Request these patterns upfront so every tool call auto-approves:

```
Bash(npm *)
Bash(npx *)
Bash(git *)
Bash(cd *)
Bash(ls *)
Bash(mkdir *)
Bash(cp *)
Bash(mv *)
Bash(rm *)
Bash(cat *)
Bash(firebase *)
Bash(node *)
Write(*)
Edit(*)
Read(*)
```

**Headless rules reminder:**
- One simple command per `Bash` call — no `&&`, `||`, `;`, or pipes
- No inline quoted arguments in Bash — use HEREDOCs or `Write` tool
- Prefer `Write` over `echo >` or `cat <<`
- Prefer `Edit` over `sed`/`awk`

---

## Checkpoints

Progress is saved via git commits at defined checkpoints. If a session runs out of tokens, the next session can pick up from the last checkpoint.

| Checkpoint | After Package(s) | Commit Message |
|------------|-------------------|----------------|
| CP-1 | 1 (scaffold) | `checkpoint: project scaffold with all deps and config` |
| CP-2 | 2, 3 (types + firebase) | `checkpoint: types and firebase client config` |
| CP-3 | 4, 5 (auth + firestore services) | `checkpoint: auth context and firestore service layer` |
| CP-4 | 6 (algorithm) | `checkpoint: contact selection algorithm with tests` |
| CP-5 | 7, 8, 11, 12 (routing + simple pages) | `checkpoint: routing, landing, settings, 404 pages` |
| CP-6 | 9, 10 (core pages) | `checkpoint: home and contacts pages` |
| CP-7 | 13, 14, 15, 16 (config packages) | `checkpoint: PWA, firebase config, CI/CD, lint/prettier` |
| CP-8 | 17 (integration) | `checkpoint: integration tests and final verification` |

---

## Work Packages

### Package 1: Project Scaffold

**Agent:** main (opus)
**Dependencies:** none
**Files created:** `package.json`, `tsconfig.json`, `tsconfig.node.json`, `vite.config.ts`, `index.html`, `.gitignore`, `src/main.tsx`, `src/App.tsx`, `src/vite-env.d.ts`, `src/theme.ts`

**Steps:**
1. `npx create-vite kindling-scaffold --template react-ts` (in a temp dir, then move files — or just Write files directly since the structure is well-defined in TDD)
2. Actually — since we already have a repo with docs, we'll scaffold in-place:
   - Write `package.json` with all dependencies
   - Write `tsconfig.json`, `tsconfig.node.json`
   - Write `vite.config.ts` (with PWA plugin placeholder — actual config in pkg 13, and `define: { __APP_VERSION__: JSON.stringify(process.env.APP_VERSION || 'dev') }`)
   - Write `index.html`
   - Write `src/main.tsx` (minimal — just ReactDOM.render of `<App />`)
   - Write `src/App.tsx` (placeholder with "Hello Kindling")
   - Write `src/vite-env.d.ts` (include `declare const __APP_VERSION__: string;`)
   - Write `src/theme.ts` (MUI theme with `#6C63FF` primary)
   - Update `.gitignore`
3. `npm install`
4. `npm run build` — verify clean build
5. **→ CP-1: git commit**

**Dependencies (npm):**
```
react react-dom
@mui/material @mui/icons-material @emotion/react @emotion/styled
react-router-dom
react-hook-form
firebase
seedrandom
```

**Dev dependencies:**
```
typescript @types/react @types/react-dom @types/seedrandom
@vitejs/plugin-react
vite
vite-plugin-pwa
vitest @testing-library/react @testing-library/jest-dom jsdom
eslint @typescript-eslint/eslint-plugin @typescript-eslint/parser eslint-plugin-react-hooks eslint-plugin-react-refresh
prettier eslint-config-prettier
```

---

### Package 2: TypeScript Types & Interfaces

**Agent:** sub-agent (haiku, worktree)
**Dependencies:** Package 1
**Files created:** `src/types/index.ts`

Write all interfaces per TDD section 3: `Contact`, `ContactFormData`, `UserSettings`, `AuthContextType`.

---

### Package 3: Firebase Client Config

**Agent:** sub-agent (haiku, worktree)
**Dependencies:** Package 1
**Files created:** `src/firebase.ts`

Write Firebase initialization per TDD section 4. Reads env vars via `import.meta.env`.

---

**→ After packages 2 & 3 merge: CP-2: git commit**

---

### Package 4: Auth Context + Provider + Hook

**Agent:** sub-agent (sonnet, worktree)
**Dependencies:** Packages 2, 3
**Files created:** `src/contexts/AuthContext.tsx`

**Details:**
- Create `AuthContext` with `React.createContext`
- `AuthProvider` component wraps children, uses `onAuthStateChanged` listener
- `signInWithGoogle` uses `GoogleAuthProvider` + `signInWithPopup`
- `signOut` calls Firebase `signOut`
- Export `useAuth()` hook with context validation
- The sub-agent should read `src/types/index.ts` and `src/firebase.ts` for imports

---

### Package 5: Firestore Service Layer

**Agent:** sub-agent (sonnet, worktree)
**Dependencies:** Packages 2, 3
**Files created:** `src/services/contacts.ts`

**Functions:**
- `getContactsRef(userId)` — returns collection reference `/users/{userId}/contacts`
- `subscribeToContacts(userId, callback)` — `onSnapshot` listener, filters `archived !== true`, returns unsubscribe function
- `addContact(userId, data: ContactFormData)` — generates `offset`, sets timestamps, adds to Firestore
- `updateContact(userId, contactId, data: ContactFormData)` — updates with new timestamps
- `archiveContact(userId, contactId)` — sets `archived: true`
- `markContacted(userId, contactId)` — updates `last_contacted` to `Date.now()` with server timestamp
- All functions use modular Firebase SDK imports
- The sub-agent should read `src/types/index.ts` and `src/firebase.ts` for imports

---

**→ After packages 4 & 5 merge: CP-3: git commit**

---

### Package 6: Contact Selection Algorithm

**Agent:** sub-agent (sonnet, worktree)
**Dependencies:** Package 2
**Files created:** `src/services/algorithm.ts`, `src/services/__tests__/algorithm.test.ts`

**Implementation:**
- `getTodaysContacts(contacts: Contact[], settings: UserSettings): Contact[]`
- Uses `seedrandom` seeded by today's date string (`YYYY-MM-DD`)
- Implements the formula, cooldown check, archived filter, already-contacted-today filter per TDD section 5
- Helper: `formatDateSeed(date: Date): string`
- Helper: `isToday(timestamp: number): boolean`

**Tests (Vitest):**
- Determinism: same inputs → same outputs across calls
- Filters out archived contacts
- Filters out contacts contacted today
- Cooldown period prevents too-recent contacts from appearing
- Formula selects correct contacts (use known seed values)
- Empty input → empty output
- All contacts filtered → empty output

---

**→ After package 6 merges: CP-4: git commit**

---

### Package 7: Navigation + Routing + PrivateRoute

**Agent:** sub-agent (sonnet, worktree)
**Dependencies:** Packages 4
**Files created:** `src/components/Navbar.tsx`, `src/components/PrivateRoute.tsx`
**Files modified:** `src/App.tsx`

**Navbar:**
- MUI `AppBar` with app title "Kindling"
- Hamburger menu (MUI `Drawer`) for authed users: Home, Contacts, Settings links
- Uses `useAuth()` to conditionally show menu items
- Login/Logout button in the AppBar

**PrivateRoute:**
- Checks `useAuth()` for user state
- Shows `CircularProgress` while loading
- Redirects to `/` if not authenticated

**App.tsx update:**
- Wrap with `ThemeProvider`, `AuthProvider`, `BrowserRouter`
- Set up all routes per TDD section 7
- Import placeholder page components (will be replaced by later packages)
- Create minimal placeholder components inline or as stubs for pages not yet built

---

### Package 8: Landing Page

**Agent:** sub-agent (haiku, worktree)
**Dependencies:** Package 4 (needs `useAuth`)
**Files created:** `src/pages/LandingPage.tsx`

- Welcome headline and tagline
- Three-step quick-start guide (numbered list with MUI `Typography`)
- Google sign-in button using `useAuth().signInWithGoogle`
- Redirect to `/home` if already authenticated
- Use MUI `Container`, `Box`, `Button`, `Typography`

---

### Package 11: Settings Page

**Agent:** sub-agent (haiku, worktree)
**Dependencies:** Package 4 (needs `useAuth`)
**Files created:** `src/pages/Settings.tsx`

- Page title "Settings"
- Logout button using `useAuth().signOut`
- App version display at bottom: `Typography variant="caption"` showing `__APP_VERSION__` (global const injected by Vite `define`)
- Minimal layout with MUI `Container`, `Button`

---

### Package 12: 404 Page

**Agent:** sub-agent (haiku, worktree)
**Dependencies:** Package 1 (no auth needed)
**Files created:** `src/pages/NotFound.tsx`

- "Page not found" message
- Link back to home
- MUI `Container`, `Typography`, `Button`

---

**→ After packages 7, 8, 11, 12 merge: CP-5: git commit + `npm run build` verification**

---

### Package 9: Home Page (Today's Contacts)

**Agent:** sub-agent (sonnet, worktree)
**Dependencies:** Packages 4, 5, 6
**Files created:** `src/pages/Home.tsx`, `src/components/ContactCard.tsx`

**Home.tsx:**
- Uses `useAuth()` for user ID
- Sets up `subscribeToContacts` listener in `useEffect` (cleanup on unmount)
- Passes contacts to `getTodaysContacts()` with default `UserSettings`
- Renders list of `ContactCard` components
- Empty states:
  - No contacts at all → "Add your first contact" with link to `/contacts`
  - All done for today → congratulatory message

**ContactCard.tsx:**
- Displays contact name
- Three action buttons: Text (`sms:`), Call (`tel:`), Email (`mailto:`)
- Buttons only render if the corresponding field (phone/email) exists
- Tapping any action calls `markContacted` and opens the URL scheme
- MUI `Card`, `CardContent`, `CardActions`, `IconButton`

---

### Package 10: Contacts Page + Add/Edit Modal

**Agent:** sub-agent (sonnet, worktree)
**Dependencies:** Packages 4, 5
**Files created:** `src/pages/Contacts.tsx`, `src/components/ContactFormModal.tsx`

**Contacts.tsx:**
- Uses `subscribeToContacts` listener
- Renders list of contacts (name + edit button)
- FAB or button for "Add Contact" → opens modal
- Edit button → opens modal with pre-filled data

**ContactFormModal.tsx:**
- MUI `Dialog` with React Hook Form
- Fields: Name (required), Phone, Email, Days Between Contacts (required, default 14)
- Submit → calls `addContact` or `updateContact`
- Delete button (edit mode only) → calls `archiveContact`
- Validation: name required, days_between_contacts must be positive integer
- The sub-agent should read `src/types/index.ts` and `src/services/contacts.ts` for imports

---

**→ After packages 9 & 10 merge: CP-6: git commit + `npm run build` verification**

---

### Package 13: PWA Config

**Agent:** sub-agent (haiku, worktree)
**Dependencies:** Package 1
**Files created:** `public/kindling-192.png`, `public/kindling-512.png`
**Files modified:** `vite.config.ts`

- Update `vite.config.ts` with full `VitePWA` config per TDD section 8
- Generate simple placeholder PNG icons (solid color square with "K" — can use a canvas script, or just write minimal valid PNGs)
- Write `public/robots.txt`

---

### Package 14: Firebase Config Files

**Agent:** sub-agent (haiku, worktree)
**Dependencies:** Package 1
**Files created:** `firestore.rules`, `firestore.indexes.json`, `firebase.json`

- Write security rules per TDD/PRD
- Write empty indexes file
- Write `firebase.json` with hosting config (public: `dist`, SPA rewrite)

---

### Package 15: GitHub Actions CI/CD

**Agent:** sub-agent (haiku, worktree)
**Dependencies:** Package 1
**Files created:** `.github/workflows/deploy-preview.yml`, `.github/workflows/deploy-production.yml`

**Two separate workflows per TDD section 10:**

1. **`deploy-preview.yml`** — triggered on push to `main`
   - Build with `APP_VERSION=v{pkg.version}-dev.{short_sha}`
   - Deploy to Firebase preview channel (7-day expiry)
   - Runs lint + tests before deploying

2. **`deploy-production.yml`** — triggered on GitHub Release published
   - Build with `APP_VERSION={release_tag}` (e.g., `v1.0.0`)
   - Deploy to Firebase live channel
   - Runs lint + tests before deploying

Both workflows need Firebase env vars as GitHub secrets (see TDD section 10).

---

### Package 16: ESLint + Prettier Config

**Agent:** sub-agent (haiku, worktree)
**Dependencies:** Package 1
**Files created:** `.eslintrc.cjs`, `.prettierrc`

- ESLint config for React + TypeScript per TDD
- Prettier config (standard defaults: semi, singleQuote, 2-space indent)

---

**→ After packages 13, 14, 15, 16 merge: CP-7: git commit**

---

### Package 17: Integration & Verification

**Agent:** main (opus)
**Dependencies:** ALL previous packages
**Files created/modified:** potentially any (fixes)

**Steps:**
1. `npm run build` — must succeed with zero errors
2. `npm run lint` — must pass (fix any lint errors found)
3. `npm run test` — must pass (algorithm tests)
4. **Review agent (sonnet):** Spawn a sub-agent to review the full codebase for:
   - Import errors / missing exports
   - Type mismatches
   - Incorrect Firestore paths
   - Missing MUI theme usage
   - Broken routing
   - Security issues (no secrets in code, proper auth guards)
5. Fix any issues found by the review agent
6. `npm run build` — final clean build verification
7. **→ CP-8: git commit**
8. Print final status and remaining human actions

---

## Execution Graph

```
Package 1 (scaffold, main/opus)
    │
    ├──► CP-1 commit
    │
    ├──► Package 2 (types, haiku) ──┐
    ├──► Package 3 (firebase, haiku)┤
    │                               ├──► CP-2 commit
    │                               │
    │    ┌──────────────────────────┘
    │    │
    │    ├──► Package 4 (auth, sonnet) ──────┐
    │    ├──► Package 5 (firestore, sonnet) ─┤
    │    │                                    ├──► CP-3 commit
    │    │                                    │
    │    ├──► Package 6 (algorithm, sonnet) ──┼──► CP-4 commit
    │    │                                    │
    │    │    ┌───────────────────────────────┘
    │    │    │
    │    │    ├──► Package 7 (routing, sonnet) ──┐
    │    │    ├──► Package 8 (landing, haiku) ───┤
    │    │    ├──► Package 11 (settings, haiku) ─┤
    │    │    ├──► Package 12 (404, haiku) ──────┤
    │    │    │                                   ├──► CP-5 commit
    │    │    │                                   │
    │    │    ├──► Package 9 (home, sonnet) ─────┤
    │    │    ├──► Package 10 (contacts, sonnet) ┤
    │    │    │                                   ├──► CP-6 commit
    │    │    │                                   │
    │    │    ├──► Package 13 (PWA, haiku) ──────┤
    │    │    ├──► Package 14 (firebase cfg, haiku)┤
    │    │    ├──► Package 15 (CI/CD, haiku) ────┤
    │    │    ├──► Package 16 (lint cfg, haiku) ─┤
    │    │    │                                   ├──► CP-7 commit
    │    │    │                                   │
    │    │    └──► Package 17 (verify, main/opus) ──► CP-8 commit
```

## Sub-Agent Prompt Template

Each sub-agent receives a prompt containing:

1. **Role:** "You are building package N of the Kindling app."
2. **Context files to read:** List of files the agent must `Read` before writing (types, firebase config, etc.)
3. **Files to create/modify:** Exact paths and detailed specs from this plan + TDD
4. **Conventions:**
   - Use MUI `sx` prop, no CSS files
   - Firebase modular SDK imports
   - `VITE_` prefix for env vars
   - Soft-delete with `archived: true`
   - Unix ms for dates, `serverTimestamp()` for server timestamps
5. **Validation:** "After writing all files, read them back to verify correctness."
6. **No chaining:** "Use one simple command per Bash call. No `&&`, `||`, `;`, or pipes."

## Review Agent Protocol

After all packages merge, a **review agent (sonnet)** is spawned to audit:

1. Read every `src/` file
2. Check all imports resolve to real exports
3. Check TypeScript types are used consistently
4. Check Firestore collection paths match across services and components
5. Check auth guards are on all protected routes
6. Check no hardcoded secrets or Firebase config values
7. Report issues as a structured list

Main agent then fixes any reported issues before final build.

---

## Post-Build Human Actions

After the build completes, the human needs to:

1. Complete `doc/human-setup.md` if not already done
2. Run `npm run dev` and verify app loads
3. Test Google sign-in flow
4. Add a contact and verify it appears on the Home page
5. Optionally: `firebase deploy` to push to hosting

# Kindling — Technical Design Document

## 1. Architecture Overview

### Component Tree

```
<App>
  <ThemeProvider>
    <AuthProvider>
      <BrowserRouter>
        <Navbar />
        <Routes>
          <Route path="/" element={<LandingPage />} />
          <Route path="/home" element={<PrivateRoute><Home /></PrivateRoute>} />
          <Route path="/contacts" element={<PrivateRoute><Contacts /></PrivateRoute>} />
          <Route path="/settings" element={<PrivateRoute><Settings /></PrivateRoute>} />
          <Route path="*" element={<NotFound />} />
        </Routes>
      </BrowserRouter>
    </AuthProvider>
  </ThemeProvider>
</App>
```

### Data Flow

```
Firebase Auth ──► AuthContext (React Context)
                      │
                      ▼
               PrivateRoute guard
                      │
                      ▼
              Page Components (Home, Contacts, Settings)
                      │
                      ▼
              Firestore Service Layer (CRUD + listeners)
                      │
                      ▼
              Cloud Firestore (/users/{uid}/contacts/*)
```

- **Auth flow:** Firebase Auth SDK handles Google OAuth → AuthContext stores user state → PrivateRoute checks context before rendering protected pages.
- **Data flow:** Components call service functions → service functions use Firestore SDK → real-time listeners (`onSnapshot`) push updates back to components via state.
- **Action flow:** User taps Call/Text/Email → opens native URL scheme → marks contact as contacted via Firestore update.

---

## 2. File/Folder Structure

```
kindling/
├── public/
│   ├── favicon.ico
│   ├── kindling-192.png
│   ├── kindling-512.png
│   └── robots.txt
├── src/
│   ├── components/
│   │   ├── ContactFormModal.tsx    # Add/Edit contact modal (React Hook Form)
│   │   ├── ContactCard.tsx         # Single contact display with action buttons
│   │   ├── Navbar.tsx              # Top nav with hamburger menu
│   │   └── PrivateRoute.tsx        # Auth guard wrapper
│   ├── contexts/
│   │   └── AuthContext.tsx         # Auth state provider + hook
│   ├── pages/
│   │   ├── LandingPage.tsx         # Public welcome page
│   │   ├── Home.tsx                # Today's contacts
│   │   ├── Contacts.tsx            # All contacts list
│   │   ├── Settings.tsx            # Settings + logout
│   │   └── NotFound.tsx            # 404 page
│   ├── services/
│   │   ├── contacts.ts             # Firestore CRUD for contacts
│   │   └── algorithm.ts            # Contact selection algorithm
│   ├── types/
│   │   └── index.ts                # TypeScript interfaces
│   ├── App.tsx                     # Root component with routing
│   ├── firebase.ts                 # Firebase app initialization
│   ├── theme.ts                    # MUI theme configuration
│   ├── main.tsx                    # Entry point
│   └── vite-env.d.ts              # Vite type declarations
├── doc/
│   ├── prd.md
│   ├── tdd.md
│   ├── human-setup.md
│   └── implementation-plan.md
├── .github/
│   └── workflows/
│       └── deploy.yml              # GitHub Actions CI/CD
├── .env.local                      # Firebase config (not committed)
├── .gitignore
├── .eslintrc.cjs                   # ESLint config
├── .prettierrc                     # Prettier config
├── firestore.rules                 # Firestore security rules
├── firestore.indexes.json          # Firestore indexes
├── firebase.json                   # Firebase hosting config
├── index.html                      # Vite entry HTML
├── package.json
├── tsconfig.json
├── tsconfig.node.json
└── vite.config.ts
```

---

## 3. TypeScript Interfaces

```typescript
// src/types/index.ts

export interface Contact {
  id: string;                       // Firestore document ID
  name: string;
  email: string;
  phone: string;
  days_between_contacts: number;    // e.g., 14 = every two weeks
  last_contacted: number;           // Unix ms
  last_updated: number;             // Unix ms (client)
  last_updated_server: any;         // Firestore ServerTimestamp
  offset: number;                   // Random int [0, days_between_contacts)
  archived: boolean;
}

// Form data (subset — no id, offset, timestamps, or archived)
export interface ContactFormData {
  name: string;
  email: string;
  phone: string;
  days_between_contacts: number;
}

export interface UserSettings {
  minimum_contact_threshold: number; // Default: 0.25
}

export interface AuthContextType {
  user: import('firebase/auth').User | null;
  loading: boolean;
  signInWithGoogle: () => Promise<void>;
  signOut: () => Promise<void>;
}
```

---

## 4. Firebase Setup

### Configuration Shape

```typescript
// src/firebase.ts
import { initializeApp } from 'firebase/app';
import { getAuth } from 'firebase/auth';
import { getFirestore } from 'firebase/firestore';

const firebaseConfig = {
  apiKey: import.meta.env.VITE_FIREBASE_API_KEY,
  authDomain: import.meta.env.VITE_FIREBASE_AUTH_DOMAIN,
  projectId: import.meta.env.VITE_FIREBASE_PROJECT_ID,
  storageBucket: import.meta.env.VITE_FIREBASE_STORAGE_BUCKET,
  messagingSenderId: import.meta.env.VITE_FIREBASE_MESSAGING_SENDER_ID,
  appId: import.meta.env.VITE_FIREBASE_APP_ID,
};

const app = initializeApp(firebaseConfig);
export const auth = getAuth(app);
export const db = getFirestore(app);
```

### Security Rules

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /users/{userId}/{document=**} {
      allow read, write: if request.auth != null && request.auth.uid == userId;
    }
  }
}
```

### Indexes

No composite indexes required for v1. All queries are on a single subcollection filtered by `archived == false`, which Firestore handles with automatic single-field indexes.

```json
{
  "indexes": [],
  "fieldOverrides": []
}
```

---

## 5. Contact Selection Algorithm

### Pseudocode

```
function getTodaysContacts(contacts: Contact[], userSettings: UserSettings): Contact[]
  today = formatDate(now)             // "2026-03-15"
  rng = seedrandom(today)            // Deterministic RNG seeded by date string
  todayRandomInt = floor(rng() * 1000)

  results = []

  for each contact in contacts:
    if contact.archived:
      continue

    // Check if already contacted today
    if isToday(contact.last_contacted):
      continue

    // Cooldown check: don't suggest if contacted too recently
    cooldownDays = contact.days_between_contacts * userSettings.minimum_contact_threshold
    daysSinceLastContact = (now - contact.last_contacted) / MS_PER_DAY
    if daysSinceLastContact < cooldownDays:
      continue

    // Deterministic selection
    if (contact.offset + todayRandomInt) % contact.days_between_contacts === 0:
      results.push(contact)

  return results
```

### Worked Example

Given:
- Today: "2026-03-15"
- `seedrandom("2026-03-15")` → first call returns 0.7342 → `todayRandomInt = 734`
- `minimum_contact_threshold = 0.25`

| Contact | offset | days_between | last_contacted | Archived | Calculation | Selected? |
|---------|--------|-------------|----------------|----------|-------------|-----------|
| Alice   | 3      | 7           | 2026-03-10     | false    | (3+734) % 7 = 737 % 7 = 2 ≠ 0 | No |
| Bob     | 12     | 14          | 2026-03-01     | false    | (12+734) % 14 = 746 % 14 = 4 ≠ 0 | No |
| Carol   | 5      | 7           | 2026-03-08     | false    | (5+734) % 7 = 739 % 7 = 4 ≠ 0 | No |
| Dave    | 2      | 7           | 2026-03-12     | false    | cooldown = 7*0.25 = 1.75 days; elapsed = 3 days → passes. (2+734) % 7 = 736 % 7 = 0 ✓ | **Yes** |
| Eve     | 8      | 30          | 2026-03-15     | false    | Already contacted today | No |
| Frank   | 1      | 7           | 2026-02-01     | true     | Archived → skip | No |

Result: `[Dave]`

### Key Properties

- **Deterministic:** Same contacts appear all day for a given user (seed = date string).
- **Fair distribution:** The `offset` is randomly assigned per contact on creation, spreading selections evenly across days.
- **Cooldown protection:** Even if the formula selects a contact, if they were recently contacted (within threshold), they're skipped.
- **Self-correcting:** If a user misses a day, the algorithm naturally cycles to different contacts the next day.

---

## 6. State Management

### AuthContext (React Context)

- Wraps the entire app
- Listens to `onAuthStateChanged` from Firebase Auth
- Provides `user`, `loading`, `signInWithGoogle()`, `signOut()`
- Components consume via `useAuth()` custom hook

### Data State (Component-Level with Firestore Listeners)

No global state management library (Redux, Zustand, etc.) — not needed at this scale.

- **Home page:** `useEffect` sets up `onSnapshot` listener for active contacts → feeds into algorithm → local state for today's contacts
- **Contacts page:** `onSnapshot` listener for all active contacts → local state
- **Settings page:** Static — just a logout button in v1

### Why No Global Store

- Only 3 pages need data, and they need different views of it
- Firestore `onSnapshot` already acts as a "live subscription" — the source of truth is Firestore, not local state
- Adding Redux/Zustand would add complexity without benefit at this scale

---

## 7. Routing

### Route Table

| Path | Component | Guard | Behavior |
|------|-----------|-------|----------|
| `/` | `LandingPage` | None (redirect if authed) | If user is logged in, redirect to `/home` |
| `/home` | `Home` | `PrivateRoute` | Today's contacts |
| `/contacts` | `Contacts` | `PrivateRoute` | Contact list + add/edit |
| `/settings` | `Settings` | `PrivateRoute` | Logout |
| `*` | `NotFound` | None | 404 |

### PrivateRoute Pattern

```typescript
// src/components/PrivateRoute.tsx
function PrivateRoute({ children }: { children: React.ReactNode }) {
  const { user, loading } = useAuth();

  if (loading) return <CircularProgress />;
  if (!user) return <Navigate to="/" replace />;
  return <>{children}</>;
}
```

### LandingPage Redirect

```typescript
// Inside LandingPage component
const { user, loading } = useAuth();
if (!loading && user) return <Navigate to="/home" replace />;
```

---

## 8. PWA Configuration

### Manifest (via vite-plugin-pwa)

```typescript
// In vite.config.ts → VitePWA plugin options
{
  registerType: 'autoUpdate',
  manifest: {
    name: 'Kindling',
    short_name: 'Kindling',
    description: 'Keep the spark alive — maintain your relationships',
    theme_color: '#6C63FF',
    background_color: '#ffffff',
    display: 'standalone',
    start_url: '/',
    icons: [
      { src: '/kindling-192.png', sizes: '192x192', type: 'image/png' },
      { src: '/kindling-512.png', sizes: '512x512', type: 'image/png' },
    ],
  },
  workbox: {
    globPatterns: ['**/*.{js,css,html,ico,png,svg}'],
  },
}
```

### Service Worker Strategy

- **Precache** the app shell (JS, CSS, HTML, icons) for offline loading
- **Network-first** for Firestore data (handled by Firebase SDK, not service worker)
- `registerType: 'autoUpdate'` — silently updates the service worker when a new version is deployed

---

## 9. Testing Strategy

### What to Test

| Layer | Tool | What |
|-------|------|------|
| Algorithm | Vitest | `getTodaysContacts` with various inputs — determinism, cooldown, archived filtering |
| Services | Vitest + mocks | Firestore service functions call correct SDK methods with correct paths |
| Components | Vitest + React Testing Library | Key user flows — login redirect, contact list rendering, form submission |

### What Not to Test

- Firebase SDK internals (trust the SDK)
- MUI component behavior (trust the library)
- Visual styling / pixel-perfect layout
- E2E flows requiring a live Firebase project (save for manual QA)

### Test Files

```
src/
├── services/
│   └── __tests__/
│       └── algorithm.test.ts
```

Focus testing on the algorithm — it's the core business logic and most likely to have edge cases.

---

## 10. CI/CD & Versioning

### Version Strategy

The app version is derived from `package.json` `version` field plus build-time context:

- **Production** (GitHub Release tag): `v1.0.0` — clean semver from the git tag
- **Preview/Dev** (push to main): `v1.0.0-dev.abc1234` — base version + short git SHA

Version is injected at build time via Vite's `define` option:

```typescript
// vite.config.ts
export default defineConfig({
  define: {
    __APP_VERSION__: JSON.stringify(process.env.APP_VERSION || 'dev'),
  },
  // ...
});
```

CI sets `APP_VERSION` before building:
- Push to main: `APP_VERSION=v{pkg.version}-dev.{git_sha_short}`
- Release: `APP_VERSION=v{tag_version}` (from the release tag)

The version is displayed in the Settings page as a small subtitle.

A global type declaration in `src/vite-env.d.ts`:
```typescript
declare const __APP_VERSION__: string;
```

### Deployment Strategy

| Trigger | Channel | URL | Version Label |
|---------|---------|-----|---------------|
| Push to `main` | Firebase preview channel | Auto-generated preview URL | `v1.0.0-dev.abc1234` |
| GitHub Release published | Firebase live channel | `kindling-app.web.app` | `v1.0.0` |

This means:
- Every merge to main is instantly testable at a preview URL
- Promoting to production = creating a GitHub Release (tag `v1.0.0`)
- Shared Firestore database across both environments (no separate projects needed)

### GitHub Actions Workflows

**Preview deploy (push to main):**

```yaml
# .github/workflows/deploy-preview.yml
name: Deploy Preview

on:
  push:
    branches: [main]

jobs:
  deploy-preview:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm

      - run: npm ci
      - run: npm run lint
      - run: npm run test -- --run

      - name: Set version
        run: |
          VERSION="v$(node -p "require('./package.json').version")-dev.$(git rev-parse --short HEAD)"
          echo "APP_VERSION=$VERSION" >> $GITHUB_ENV

      - run: npm run build
        env:
          APP_VERSION: ${{ env.APP_VERSION }}

      - name: Deploy to preview channel
        uses: FirebaseExtended/action-hosting-deploy@v0
        with:
          repoToken: ${{ secrets.GITHUB_TOKEN }}
          firebaseServiceAccount: ${{ secrets.FIREBASE_SERVICE_ACCOUNT }}
          channelId: preview
          expires: 7d
```

**Production deploy (GitHub Release):**

```yaml
# .github/workflows/deploy-production.yml
name: Deploy Production

on:
  release:
    types: [published]

jobs:
  deploy-production:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm

      - run: npm ci
      - run: npm run lint
      - run: npm run test -- --run

      - name: Set version from release tag
        run: echo "APP_VERSION=${{ github.event.release.tag_name }}" >> $GITHUB_ENV

      - run: npm run build
        env:
          APP_VERSION: ${{ env.APP_VERSION }}

      - name: Deploy to live channel
        uses: FirebaseExtended/action-hosting-deploy@v0
        with:
          repoToken: ${{ secrets.GITHUB_TOKEN }}
          firebaseServiceAccount: ${{ secrets.FIREBASE_SERVICE_ACCOUNT }}
          channelId: live
```

### Environment Variables in CI

Firebase config values are baked into the build via `import.meta.env`. For CI, add these as GitHub Actions repository secrets:
- `VITE_FIREBASE_API_KEY`
- `VITE_FIREBASE_AUTH_DOMAIN`
- `VITE_FIREBASE_PROJECT_ID`
- `VITE_FIREBASE_STORAGE_BUCKET`
- `VITE_FIREBASE_MESSAGING_SENDER_ID`
- `VITE_FIREBASE_APP_ID`
- `FIREBASE_SERVICE_ACCOUNT` (for deploy action)

### Build Commands

| Command | Purpose |
|---------|---------|
| `npm run dev` | Start Vite dev server |
| `npm run build` | Production build to `dist/` |
| `npm run preview` | Preview production build locally |
| `npm run lint` | Run ESLint |
| `npm run test` | Run Vitest |

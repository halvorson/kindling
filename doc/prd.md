# Kindling - Product Requirements Document

## Overview

**Kindling** is a progressive web app that helps users maintain personal relationships by intelligently suggesting which friends and contacts to reach out to each day. The core premise: people lose touch not because they don't care, but because they forget. Kindling removes that friction by showing a short, actionable list of people to contact today, with one-tap access to call, text, or email them.

**Tagline:** "Keep the spark alive."

---

## Problem Statement

Maintaining a broad network of personal relationships is cognitively expensive. People intend to keep in touch with friends, family, and colleagues but lack a system to remind them who to contact and when. Existing tools (calendars, recurring reminders) are too rigid and create obligation rather than gentle nudges.

## Target Users

- Anyone who wants to stay in touch with friends, family, or professional contacts
- People who feel guilty about losing touch but don't have a system
- Professionals maintaining a broad network

---

## Platform Choices & Rationale

### Why React + Vite (not Create React App)

The original app used Create React App, which is now deprecated and unmaintained. **Vite** is the modern standard for React tooling — faster dev server (instant HMR), faster builds, and actively maintained. We'll use Vite with React 18 and TypeScript for type safety from the start.

### Why TypeScript

The original was plain JavaScript. TypeScript catches bugs at compile time, provides better IDE support, and makes the codebase more maintainable as it grows. The data model (contacts, user settings) benefits especially from typed interfaces.

### Why Material UI v5 (MUI)

Continues the original design direction. MUI v5 offers a mature component library with built-in accessibility, a robust theming system, and the `sx` prop for inline styling that replaces the older `makeStyles` pattern. Reduces custom CSS to near-zero.

### Why Firebase (continued)

Firebase remains the right choice for this app:
- **Zero backend code** — auth, database, hosting, and security rules are all managed services
- **Real-time sync** — Firestore listeners update the UI instantly when data changes
- **Free tier** is generous enough for personal/small-scale use (50K reads/day, 20K writes/day)
- **Google Auth** is a first-class citizen
- **Firebase Hosting** with GitHub Actions CI/CD provides automatic preview deploys on PRs

### Why React Hook Form (continued)

Lightweight, minimal re-renders, excellent TypeScript support. The contact form is simple enough that heavier solutions (Formik, etc.) aren't warranted.

### Why seedrandom (continued)

The deterministic daily algorithm is a core differentiator. `seedrandom` provides reproducible random numbers seeded by date string, ensuring the same contacts appear all day but vary day to day. No suitable alternative exists in the standard library.

---

## Features

### 1. Authentication

- **Google OAuth sign-in** via Firebase Authentication
- No email/password accounts — Google-only for simplicity
- Unauthenticated users see a landing page with app description and login prompt
- Authenticated users are redirected to the Home page

### 2. Landing Page

- Public-facing welcome screen
- App description and value proposition
- Three-step quick-start guide:
  1. Sign in with Google
  2. Add your contacts with a desired frequency
  3. Open Kindling each day to see who to reach out to
- Prominent Google sign-in button
- Illustrations to communicate the concept visually

### 3. Home Page (Today's Contacts)

The core experience. Displays a curated list of contacts the user should reach out to today.

**Contact Selection Algorithm:**
- Uses a deterministic daily randomization (seeded by today's date) combined with each contact's frequency preference and a per-contact random offset
- Formula: `(contact.offset + todayRandomInt) % contact.days_between_contacts === 0`
- Filters out contacts already contacted today
- Applies a cooldown period: won't suggest a contact if they were reached out to within `days_between_contacts * minimum_contact_threshold` (default threshold: 0.25)
- Excludes archived contacts

**Per-Contact Actions:**
- **Text** button — opens `sms://` with the contact's phone number
- **Call** button — opens `tel://` with the contact's phone number
- **Email** button — opens `mailto:` with the contact's email address
- Tapping any action records the contact as "contacted today" (updates `last_contacted` timestamp)

**Empty States:**
- No contacts added yet → illustration + prompt to add contacts
- All contacts for today done → congratulatory message

### 4. Contacts Page

Full list of all active (non-archived) contacts.

- Each contact displays name and edit button
- "Add Contact" button opens the contact form modal
- Edit button opens the same modal pre-filled with contact data

### 5. Contact Management (Add/Edit Modal)

Modal form for creating and editing contacts.

**Fields:**
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| Name | Text | Yes | Contact's display name |
| Phone | Tel | No | Phone number for calls and texts |
| Email | Email | No | Email address |
| Days Between Contacts | Number | Yes (default: 14) | How often to suggest this contact (in days) |

**Actions:**
- Save — creates or updates the contact in Firestore
- Delete (edit mode only) — soft-deletes by setting `archived: true`

**On Create**, the system also generates:
- `offset` — random integer (0 to `days_between_contacts`) for algorithm distribution
- `last_contacted` — initialized to current timestamp
- `last_updated` / `last_updated_server` — audit timestamps

### 6. Settings Page

- Logout button
- (Minimal — placeholder for future settings like notification preferences, contact frequency defaults, etc.)

### 7. Navigation

- Top navbar present on all pages
- Hamburger menu for authenticated users with links: Home, Contacts, Settings
- Login/Logout button in navbar
- Route protection: unauthenticated users cannot access Home, Contacts, or Settings

### 8. 404 Page

- Friendly "page not found" illustration and message for unmatched routes

---

## Data Model

### Firestore Structure

```
/users/{userId}/
  └── contacts/{contactId}
        ├── name: string
        ├── email: string
        ├── phone: string
        ├── days_between_contacts: number
        ├── last_contacted: number (Unix ms)
        ├── last_updated: number (Unix ms)
        ├── last_updated_server: Timestamp (server)
        ├── offset: number
        └── archived: boolean
```

### User-Level Settings

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `minimum_contact_threshold` | number | 0.25 | Multiplier for cooldown period — prevents re-suggesting a contact too soon |

---

## Tech Stack

| Layer | Technology | Version |
|-------|-----------|---------|
| Frontend Framework | React | 18 |
| Language | TypeScript | 5.x |
| Build Tool | Vite | 6.x |
| Routing | React Router | v6 |
| UI Library | Material UI (MUI) | v5 |
| Form Handling | React Hook Form | v7 |
| Authentication | Firebase Authentication | v10 (modular SDK) |
| Database | Cloud Firestore | v10 (modular SDK) |
| Hosting | Firebase Hosting | — |
| CI/CD | GitHub Actions | — |
| PWA | Vite PWA Plugin (vite-plugin-pwa) | — |
| Algorithm RNG | seedrandom | v3 |
| Linting | ESLint + Prettier | — |

### Firebase Services Used

1. **Firebase Authentication** — Google sign-in provider
2. **Cloud Firestore** — user and contact data storage with real-time sync
3. **Firebase Hosting** — static site hosting with SPA rewrite rules
4. **Firebase Security Rules** — per-user data isolation (users can only read/write their own data)

---

## Deployment

### Firebase Project

- **Project ID:** TBD (new project for Kindling)
- **Hosting URL:** `kindling-app.web.app` (or similar, pending availability)

### CI/CD Pipeline (GitHub Actions)

| Trigger | Action |
|---------|--------|
| Push to `main` | Build and deploy to Firebase Hosting (live) |
| Pull request | Build and deploy to Firebase Hosting preview channel |

### Build & Deploy

```bash
npm install
npm run build
firebase deploy
```

### Firebase Configuration Files

- `firebase.json` — hosting config (serve from `dist/`, SPA rewrite to `index.html`)
- `.firebaserc` — project alias mapping
- `firestore.rules` — security rules for Firestore
- `firestore.indexes.json` — composite index definitions

---

## Firestore Security Rules

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

Users can only access their own data. No unauthenticated access.

---

## PWA Requirements

- App is installable as a standalone PWA
- Service worker via `vite-plugin-pwa` with Workbox under the hood
- Manifest includes app name ("Kindling"), theme color (`#6C63FF`), icons
- Works on mobile and desktop

---

## Pages & Routes

| Route | Component | Auth Required | Description |
|-------|-----------|---------------|-------------|
| `/` | LandingPage | No (redirects to /home if logged in) | Welcome + sign-in |
| `/home` | Home | Yes | Today's suggested contacts |
| `/contacts` | Contacts | Yes | All contacts list + add/edit |
| `/settings` | Settings | Yes | User settings + logout |
| `*` | NotFound | No | 404 page |

---

## Non-Functional Requirements

- **Performance:** Initial load under 3 seconds on 3G
- **Responsiveness:** Mobile-first, works on all screen sizes
- **Accessibility:** Follows MUI accessibility defaults; all interactive elements keyboard-navigable
- **Data Privacy:** User data isolated per Firebase Auth UID; no cross-user data access
- **Offline:** PWA shell loads offline; data operations require connectivity

---

## Future Considerations (Out of Scope for v1)

- Push notifications ("Time to catch up with Alice!")
- Contact import from phone/Google Contacts
- Contact groups/tags
- Notes per contact (e.g., "Ask about the new job")
- Contact history log (when did I last reach out and via what channel?)
- Customizable minimum contact threshold in Settings UI
- Dark mode
- Multiple auth providers (Apple, email/password)

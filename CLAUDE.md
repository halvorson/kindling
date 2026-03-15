# Kindling

Relationship maintenance PWA. See `doc/prd.md` for full product requirements.

## Stack

React 18 + TypeScript + Vite 6 + MUI v5 + Firebase v10 (Auth, Firestore, Hosting) + React Router v6 + React Hook Form v7 + seedrandom + vite-plugin-pwa

## Agent Orchestration Prompt

When told to **"build kindling"**, execute the following plan. Do NOT start implementation until a human confirms. Produce documents first, get approval, then build.

### Phase 0: Pre-flight (you, immediately)

Request these permissions upfront so the human isn't interrupted mid-flow:
- `Bash(npm *)` — package installs, vite scaffold, build commands
- `Bash(npx *)` — create-vite, firebase tools
- `Bash(git *)` — commits, branches
- `Write(*)` — creating all project files
- `Edit(*)` — modifying files

**IMPORTANT — Headless execution rules:**
- This session is intended to run headless with minimal human intervention.
- **Always** use the permission patterns above. Do NOT chain commands with `&&`, `||`, or `;` — each command must match one of the approved patterns exactly so it auto-approves.
- **Never** pass quoted strings as inline arguments in Bash commands (e.g., don't do `git commit -m "message"`). Instead, use HEREDOCs for commit messages, write config to files with the `Write` tool, etc.
- Prefer `Write` over `echo >` or `cat <<`. Prefer `Edit` over `sed`/`awk`.
- One simple command per `Bash` call. If a command would require quotes, pipes, or chaining to work, find another way.
- The goal: every tool call should auto-approve without human interaction.

### Phase 1: Technical Design Document (TDD)

Write `doc/tdd.md` covering:

1. **Architecture overview** — component tree, data flow diagram (text-based)
2. **File/folder structure** — exact paths for every file to be created
3. **TypeScript interfaces** — `Contact`, `UserSettings`, `AuthContext` types
4. **Firebase setup** — config shape, security rules, indexes
5. **Contact selection algorithm** — pseudocode with worked example
6. **State management** — React Context for auth, Firestore real-time listeners for data
7. **Routing** — route table, PrivateRoute guard pattern
8. **PWA config** — manifest values, service worker strategy
9. **Testing strategy** — what to test, what not to, which tools (Vitest + Testing Library)
10. **CI/CD** — GitHub Actions workflow files

Write `doc/human-setup.md` **in parallel** (use a background sub-agent) — a checklist the human must complete before code runs:

1. **Create Firebase project** at https://console.firebase.google.com
   - Project name: `kindling` (or similar)
   - Note the project ID
2. **Enable Authentication** → Sign-in method → Google → Enable
3. **Create Firestore database** → Start in test mode (rules will be deployed later)
4. **Register a web app** in Firebase console → copy the config object
5. **Install Firebase CLI** (`npm install -g firebase-tools`) and run `firebase login`
6. **Create `.env.local`** in project root with:
   ```
   VITE_FIREBASE_API_KEY=...
   VITE_FIREBASE_AUTH_DOMAIN=...
   VITE_FIREBASE_PROJECT_ID=...
   VITE_FIREBASE_STORAGE_BUCKET=...
   VITE_FIREBASE_MESSAGING_SENDER_ID=...
   VITE_FIREBASE_APP_ID=...
   ```
7. **Run `firebase init`** — select Hosting (dist/, SPA rewrite) + Firestore (rules + indexes)
8. **(Optional) Create GitHub repo** and add `FIREBASE_SERVICE_ACCOUNT` secret for CI/CD

**STOP after both TDD and human-setup.md are written. Print: "TDD and human setup guide complete — review `doc/tdd.md` and `doc/human-setup.md`. Do the human setup steps, then say 'go' to continue."**

### Phase 2: Implementation Plan

Write `doc/implementation-plan.md` with ordered work packages. Each package specifies:
- What to build
- Which files to create/modify
- Agent assignment (see below)
- Dependencies (which packages must complete first)

#### Work packages and agent assignments:

| # | Package | Agent | Model | Why |
|---|---------|-------|-------|-----|
| 1 | Project scaffold (Vite + deps + config files) | main | opus | Foundational, needs precision |
| 2 | TypeScript types & interfaces | sub-agent | haiku | Mechanical, well-defined by TDD |
| 3 | Firebase client config (`src/firebase.ts`) | sub-agent | haiku | Small, templated |
| 4 | Auth context + provider + hook | sub-agent | sonnet | Moderate complexity, needs Firebase interop |
| 5 | Firestore service layer (CRUD + real-time listeners) | sub-agent | sonnet | Core data layer, needs careful typing |
| 6 | Contact selection algorithm | sub-agent | sonnet | Algorithm logic, needs to match PRD spec exactly |
| 7 | Navigation + routing + PrivateRoute | sub-agent | sonnet | Integrates auth context, routing logic |
| 8 | Landing page | sub-agent | haiku | Simple presentational component |
| 9 | Home page (today's contacts) | sub-agent | sonnet | Core UX, integrates algorithm + Firestore + actions |
| 10 | Contacts page + Add/Edit modal | sub-agent | sonnet | Form logic with React Hook Form + Firestore |
| 11 | Settings page | sub-agent | haiku | Minimal — just logout button |
| 12 | 404 page | sub-agent | haiku | Simple presentational |
| 13 | PWA config (manifest, icons, service worker) | sub-agent | haiku | Config-only |
| 14 | Firebase config files (rules, indexes, firebase.json) | sub-agent | haiku | Config-only, defined in PRD |
| 15 | GitHub Actions CI/CD | sub-agent | haiku | Templated workflow |
| 16 | ESLint + Prettier config | sub-agent | haiku | Config-only |
| 17 | Integration testing & smoke test | main | opus | Needs full context to verify everything works |

**Parallelization:** After package 1 completes, packages 2-3 can run in parallel. After 2-5 complete, packages 6-16 can mostly run in parallel (8, 11, 12, 13, 14, 15, 16 are independent; 9 depends on 6; 10 depends on 5). Package 17 runs last.

**STOP after writing implementation plan. Print "Implementation plan complete — review `doc/implementation-plan.md` and say 'go' to build."**

### Phase 3: Build

Execute the implementation plan. Use `Agent` tool with `model` parameter to dispatch sub-agents per the table above. Use `isolation: "worktree"` for sub-agents writing independent packages so they don't conflict. Merge results back in order.

After all packages complete:
1. Run `npm run build` to verify clean build
2. Run `npm run lint` to verify no lint errors
3. Print final status and list any remaining human actions

### Phase 4: Verify

- `npm run dev` — confirm app loads
- Walk through each route manually (print what to check)
- Note anything that requires the human's Firebase project to be live

---

## Conventions

- Use MUI `sx` prop for styling, no separate CSS files
- Use Firebase modular SDK (tree-shakeable imports)
- All Firebase env vars prefixed with `VITE_`
- Soft-delete pattern: `archived: true` (never hard-delete contacts)
- Dates stored as Unix ms (`Date.now()`), server timestamps via `serverTimestamp()`

# Human Setup Checklist

Complete these steps before code generation begins. The build process depends on Firebase credentials being available in `.env.local`.

---

## 1. Create Firebase Project

1. Go to [Firebase Console](https://console.firebase.google.com)
2. Click **Add project**
3. Project name: `kindling` (or similar — note the **project ID**)
4. Disable Google Analytics (not needed for v1)
5. Click **Create project**

## 2. Enable Google Authentication

1. In Firebase Console → **Authentication** → **Get started**
2. Click **Sign-in method** tab
3. Enable **Google** provider
4. Set a support email address
5. Click **Save**

## 3. Create Firestore Database

1. In Firebase Console → **Firestore Database** → **Create database**
2. Select **Start in test mode** (security rules will be deployed later)
3. Choose a Cloud Firestore location closest to your users (e.g., `us-central1`)
4. Click **Enable**

## 4. Register a Web App

1. In Firebase Console → **Project settings** (gear icon) → **General** tab
2. Scroll to **Your apps** → click the web icon (`</>`)
3. App nickname: `kindling-web`
4. Check **Also set up Firebase Hosting**
5. Click **Register app**
6. **Copy the `firebaseConfig` object** — you'll need it for the next step

## 5. Install Firebase CLI

```bash
npm install -g firebase-tools
firebase login
```

## 6. Create `.env.local`

In the project root (`/Users/michael/Coding/kindling/`), create a file named `.env.local` with the values from step 4:

```
VITE_FIREBASE_API_KEY=your-api-key
VITE_FIREBASE_AUTH_DOMAIN=your-project.firebaseapp.com
VITE_FIREBASE_PROJECT_ID=your-project-id
VITE_FIREBASE_STORAGE_BUCKET=your-project.firebasestorage.app
VITE_FIREBASE_MESSAGING_SENDER_ID=123456789
VITE_FIREBASE_APP_ID=1:123456789:web:abcdef
```

## 7. Initialize Firebase in the Project

From the project root, run:

```bash
firebase init
```

Select the following:
- **Firestore** — Yes
  - Rules file: `firestore.rules` (default)
  - Indexes file: `firestore.indexes.json` (default)
- **Hosting** — Yes
  - Public directory: `dist`
  - Single-page app: **Yes**
  - Automatic builds with GitHub: **No** (we'll set up CI/CD separately)

This creates `firebase.json`, `.firebaserc`, and starter rules/indexes files (which will be overwritten during build).

## 8. (Optional) GitHub Repository for CI/CD

1. Create a GitHub repo for the project
2. In Firebase Console → **Project settings** → **Service accounts** → **Generate new private key**
3. In GitHub repo → **Settings** → **Secrets and variables** → **Actions** → add a secret named `FIREBASE_SERVICE_ACCOUNT` with the contents of the downloaded JSON key file

---

## Verification

After completing all steps, confirm:

- [ ] `.env.local` exists in project root with all 6 `VITE_FIREBASE_*` values
- [ ] `firebase login` shows you're authenticated
- [ ] Firebase Console shows Authentication (Google) enabled
- [ ] Firebase Console shows Firestore database created
- [ ] Running `firebase projects:list` shows your project

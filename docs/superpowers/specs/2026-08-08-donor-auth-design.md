# Donor Sign Up / Log In — Design Spec

Date: 2026-08-08
Status: Approved by user, pending implementation plan

## Purpose

Add a real, working donor account system to the Rudhira pitch site
(`website/index.html`), so the "donor portal" concept in the pitch is backed
by an actual signup/login flow rather than being purely descriptive. Accounts
are real (persisted), and Google Sign-In is a first-class option alongside
email/password.

## Non-goals

- No donation-recording or blood-bank verification pipeline. This spec only
  covers account creation/login and storing the donor's declared profile
  (name, blood group, email, age). Tier progression logic described in the
  site's existing Rewards section (`#loyalty`) is not implemented here.
- No password reset flow, no email verification requirement, no admin
  console/dashboard for viewing signed-up donors.
- No Hindi (i18n) support on the new page — the rest of the site is fully
  bilingual via the existing `data-i18n` dictionary, but replicating that
  system for a functional form (with dynamic auth error states) is
  out of scope for this pass. English only.
- No custom backend/server. Firebase's client SDK is the entire backend;
  the site remains static and stays on GitHub Pages.

## Architecture

- **New page**: `website/join.html`, a standalone page (not a modal),
  linked from a new "Join as Donor" button in `index.html`'s nav.
- **Auth + storage**: Firebase Auth (Email/Password + Google providers) and
  Cloud Firestore, loaded via the Firebase **compat** CDN `<script>` tags
  (`firebase-app-compat.js`, `firebase-auth-compat.js`,
  `firebase-firestore-compat.js`). Compat is chosen over the modular v9+
  SDK to match the existing site's plain `<script>` (non-module) convention
  and avoid introducing a build step or `type="module"` for the first time
  in this codebase.
- **Hosting**: unchanged — GitHub Pages, static files only. Firebase's
  client SDK talks directly to Google's servers from the browser; nothing
  runs server-side.
- **Styling**: `join.html` reuses the same CSS custom properties block from
  `index.html` (`--bg`, `--surface`, `--ink`, `--accent`, `--font-sans`,
  etc., including the dark/light `prefers-color-scheme` + `data-theme`
  override pattern) so it is visually indistinguishable in quality from the
  rest of the site. The existing `rudhira-theme` localStorage key is read
  on load so the theme choice carries over from the main page.

## Data model

Firestore collection `donors`, one document per authenticated user, keyed
by Firebase Auth `uid`:

```
donors/{uid} = {
  name: string,
  email: string,
  age: number,
  bloodGroup: string,   // one of: A+ A- B+ B- O+ O- AB+ AB-
  createdAt: Timestamp  // server timestamp, set once at signup, never overwritten
}
```

### Firestore security rules

A user may only read/write their own document. This is the actual security
boundary — the Firebase config object (including its API key) is visible in
the page source, which is normal and expected for Firebase web apps; it is
not a secret.

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /donors/{uid} {
      allow read, write: if request.auth != null && request.auth.uid == uid;
    }
  }
}
```

## Page structure: `join.html`

Single page, two tabs sharing one form container: **Sign Up** and **Log In**.
A tab click swaps which fields are visible/required; both tabs show the same
"Continue with Google" button.

### Sign Up tab (email/password path)

Fields, all required:
- Name (text)
- Email (email input)
- Age (number, min 1, max 120)
- Blood Group (select: A+, A-, B+, B-, O+, O-, AB+, AB-)
- Password (min 6 chars — Firebase's own minimum)
- Confirm Password (must match Password, checked client-side before submit)

On submit: `createUserWithEmailAndPassword`, then `setDoc` on
`donors/{uid}` with the form fields plus `createdAt: serverTimestamp()`,
then show the Welcome screen.

### Log In tab (email/password path)

Fields:
- Email
- Password

On submit: `signInWithEmailAndPassword`, then read `donors/{uid}` and show
the Welcome screen.

### Google Sign-In (shared by both tabs)

Single "Continue with Google" button, `signInWithPopup(GoogleAuthProvider)`.
After a successful popup:

1. Check whether `donors/{uid}` already exists.
2. **Exists** (returning user, either from a prior Google sign-in or —
   theoretically — if they'd signed up by email with the same address,
   though Firebase treats these as distinct accounts) → go straight to
   Welcome screen.
3. **Does not exist** (first-time Google sign-in) → show a short
   **"Complete your profile"** step: Blood Group + Age only (Name and
   Email are pre-filled read-only from the Google account, `user.displayName`
   / `user.email`). Submitting this creates the `donors/{uid}` doc with
   `createdAt: serverTimestamp()`, then shows the Welcome screen.

### Welcome screen

Shown after any successful signup or login (email/password or Google),
and immediately on page load if `onAuthStateChanged` reports an existing
signed-in session (so a reload doesn't force re-login).

Displays:
- "Welcome, {name}"
- Blood Group, Age
- Donation status line: **"0 verified donations — donate once to unlock
  Bronze."** This deliberately does not claim a Bronze tier for a fresh
  signup, since the site's own Rewards section (`#loyalty`) defines Bronze
  as requiring a *verified* first donation, and no donation-recording
  pipeline exists yet (see Non-goals). If a Bronze default is preferred for
  demo purposes instead, that's a one-line change at implementation time.
- A "Log out" link (`signOut()`, returns to the Sign Up/Log In tabs) —
  included so the flow is testable and so a shared/demo device isn't stuck
  signed in.
- A "Back to Rudhira" link to `index.html`.

## Error handling

Firebase Auth error codes are mapped to short inline messages near the
relevant field/button, not raw Firebase error strings. Minimum set to
handle: `auth/email-already-in-use`, `auth/weak-password`,
`auth/invalid-email`, `auth/user-not-found`, `auth/wrong-password`,
`auth/popup-closed-by-user` (Google popup dismissed — not an error, just
silently reset the button state).

## Integration with `index.html`

Add one nav button, styled like the existing `.navcta` ("The ask") element,
labeled **"Join as Donor"**, linking to `join.html`. Placed in the
`nav-right` div alongside the existing theme/lang toggles and the ask CTA.

## Setup required from the user (not part of implementation)

Before this can be implemented and tested end-to-end, the user needs to:

1. Create a free project at console.firebase.google.com.
2. Enable Authentication → Sign-in method → Email/Password and Google.
3. Enable Firestore (production mode, then apply the rules above).
4. Add `soaked1.github.io` to Authentication → Settings → Authorized
   domains (needed for Google Sign-In to work once deployed; `localhost`
   is included by default for local testing).
5. Copy the Firebase config object (from Project Settings → General → Your
   apps → Web app) and provide it for `join.html`.

## Testing plan

Verified locally via the gstack `browse` skill, matching how the rest of the
site has been checked in this project:

- Sign up with email/password → Firestore doc created with correct fields →
  Welcome screen shows correct data.
- Log out, log back in with the same email/password → Welcome screen shows
  the same data.
- Sign up with Google (first time) → profile-completion step appears,
  pre-filled name/email correct → submitting creates the doc → Welcome
  screen correct.
- Log out, sign in with Google again (returning user) → skips
  profile-completion, goes straight to Welcome screen.
- Reload `join.html` while signed in → lands on Welcome screen without
  re-prompting for login.
- Invalid inputs: duplicate email signup, wrong password, mismatched
  confirm-password, age out of 1-120 range → correct inline errors, no
  console errors.
- Dark/light theme carries over correctly from `index.html`'s stored
  preference.
- Nav "Join as Donor" button visible and correctly linked from `index.html`
  in both themes.

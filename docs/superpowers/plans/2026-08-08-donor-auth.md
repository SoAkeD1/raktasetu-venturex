# Donor Sign Up / Log In Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a real, working donor account system (`website/join.html`) to the Rudhira pitch site, with email/password signup+login and Google Sign-In, backed by Firebase Auth + Firestore.

**Architecture:** A new standalone static page, `website/join.html`, using the Firebase **compat** SDK loaded via CDN `<script>` tags (no build step, matches the site's existing plain-script convention). Firebase Auth handles identity; a Firestore `donors/{uid}` document holds each donor's declared profile (name, email, age, blood group). The existing `website/index.html` gets one new nav button linking to the new page.

**Tech Stack:** Vanilla HTML/CSS/JS, Firebase JS SDK v10 (compat build), Cloud Firestore. Verified manually via the gstack `browse` skill (no JS test framework exists in this repo).

## Global Constraints

- No build step, no bundler, no `type="module"` — plain `<script>` tags only, matching `index.html`.
- Firebase **compat** SDK, not the modular v9+ API.
- No Hindi/i18n on `join.html` — English only.
- No donation-recording/verification pipeline, no password reset flow, no admin console. (See spec Non-goals.)
- Firestore security rule (must be pasted into the Firebase console, this repo cannot deploy it):
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
- Welcome screen must show **"0 verified donations — donate once to unlock Bronze."**, never a default Bronze tier, per the spec's reasoning (Bronze requires a verified donation, and no verification pipeline exists).
- Spec reference: `docs/superpowers/specs/2026-08-08-donor-auth-design.md`.

---

### Task 1: Firebase project setup (user action, not code)

This has no automated deliverable — it produces the config values every later task needs. It cannot be done by a subagent; it needs the human's Google account.

**Interfaces:**
- Produces: a `firebaseConfig` object (apiKey, authDomain, projectId, storageBucket, messagingSenderId, appId) that Task 3 hardcodes into `join.html`.

- [ ] **Step 1:** Go to https://console.firebase.google.com, click "Add project", name it (e.g. "rudhira-venturex"), finish creation.
- [ ] **Step 2:** In the project, go to Build → Authentication → Get started. Under Sign-in method, enable **Email/Password** and **Google**.
- [ ] **Step 3:** Go to Build → Firestore Database → Create database. Choose **production mode**, pick any region.
- [ ] **Step 4:** In Firestore → Rules tab, paste the rule from the Global Constraints section above and click Publish.
- [ ] **Step 5:** In Authentication → Settings → Authorized domains, click Add domain, add `soaked1.github.io`. (`localhost` is already present by default, needed for local testing.)
- [ ] **Step 6:** Go to Project settings (gear icon) → General → scroll to "Your apps" → click the Web icon (`</>`) → register an app (any nickname) → copy the `firebaseConfig` object shown.
- [ ] **Step 7:** Hand the copied `firebaseConfig` object to whoever/whatever implements Task 3.

---

### Task 2: `join.html` static shell — layout, theme toggle, tabs

**Files:**
- Create: `website/join.html`

**Interfaces:**
- Produces: DOM element IDs used by every later task — `screen-auth`, `screen-complete`, `screen-welcome` (screens), `tab-signup`, `tab-login`, `form-signup`, `form-login`, `form-complete` (forms/tabs), `su-name`, `su-email`, `su-age`, `su-bloodgroup`, `su-password`, `su-confirm`, `su-error`, `su-submit`, `li-email`, `li-password`, `li-error`, `li-submit`, `pc-age`, `pc-bloodgroup`, `pc-error`, `pc-submit`, `google-btn`, `google-error`, `welcome-heading`, `w-bloodgroup`, `w-age`, `w-email`, `w-status`, `logout-link`. Also produces the global helper `showScreen(id)`.

- [ ] **Step 1: Write the full file**

```html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Join as a Donor | Rudhira</title>
<link rel="icon" href="data:image/svg+xml,<svg xmlns=%22http://www.w3.org/2000/svg%22 viewBox=%220 0 100 100%22><circle cx=%2250%22 cy=%2250%22 r=%2248%22 fill=%22%23170404%22/><circle cx=%2250%22 cy=%2250%22 r=%2232%22 stroke=%22%23c81e3a%22 stroke-width=%226%22 fill=%22none%22 opacity=%220.5%22/><circle cx=%2250%22 cy=%2250%22 r=%2220%22 stroke=%22%23c81e3a%22 stroke-width=%226%22 fill=%22none%22 opacity=%220.75%22/><circle cx=%2250%22 cy=%2250%22 r=%229%22 fill=%22%23e02020%22/></svg>">
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
<link href="https://fonts.googleapis.com/css2?family=IBM+Plex+Sans:wght@400;500;600;700&family=IBM+Plex+Mono:wght@400;500;600&display=swap" rel="stylesheet">
<style>
  :root{
    --bg:#faf9f8; --surface:#ffffff; --surface-2:#f1efec; --ink:#1c1b1a;
    --ink-soft:#54514d; --ink-faint:#8a8681; --line:#e2ded8; --accent:#c81e3a;
    --accent-ink:#ffffff; --good:#1f7a4d; --bad:#b5321f; --good-bg:#eaf4ee; --bad-bg:#fbeceA;
    --radius:14px;
    --font-sans:'IBM Plex Sans', -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
    --font-mono:'IBM Plex Mono', ui-monospace, 'SFMono-Regular', Menlo, monospace;
  }
  :root:not([data-theme="light"]){
    @media (prefers-color-scheme: dark){
      --bg:#131211; --surface:#1b1a18; --surface-2:#221f1d; --ink:#f2f0ed;
      --ink-soft:#c2beb8; --ink-faint:#8a8681; --line:#332f2b; --accent:#ff5c72;
      --accent-ink:#1c1b1a; --good:#5fd996; --bad:#ff7a63; --good-bg:#16281f; --bad-bg:#2c1a16;
    }
  }
  :root[data-theme="dark"]{
    --bg:#131211; --surface:#1b1a18; --surface-2:#221f1d; --ink:#f2f0ed;
    --ink-soft:#c2beb8; --ink-faint:#8a8681; --line:#332f2b; --accent:#ff5c72;
    --accent-ink:#1c1b1a; --good:#5fd996; --bad:#ff7a63; --good-bg:#16281f; --bad-bg:#2c1a16;
  }
  *{box-sizing:border-box;}
  body{
    margin:0; min-height:100vh; background:var(--bg); color:var(--ink);
    font-family:var(--font-sans); line-height:1.5; -webkit-font-smoothing:antialiased;
    display:flex; flex-direction:column; align-items:center;
  }
  .topbar{width:100%; display:flex; align-items:center; justify-content:space-between; padding:20px 24px; box-sizing:border-box;}
  .brand{display:flex; align-items:center; gap:10px; font-weight:600; font-size:16px; text-decoration:none; color:var(--ink);}
  .theme-toggle{
    width:36px; height:36px; padding:0; flex:none; border-radius:50%;
    border:1px solid var(--line); background:transparent; cursor:pointer;
    display:inline-flex; align-items:center; justify-content:center;
    color:var(--ink-soft); transition:border-color .2s ease, color .2s ease;
  }
  .theme-toggle:hover{color:var(--ink); border-color:var(--ink-faint);}
  .theme-toggle svg{width:17px; height:17px;}
  .theme-toggle .icon-sun{display:none;}
  .theme-toggle.is-dark .icon-sun{display:block;}
  .theme-toggle.is-dark .icon-moon{display:none;}

  .card-wrap{flex:1; width:100%; display:flex; align-items:center; justify-content:center; padding:24px;}
  .card{
    width:100%; max-width:440px; background:var(--surface); border:1px solid var(--line);
    border-radius:var(--radius); padding:32px; box-shadow:0 1px 2px rgba(0,0,0,.04);
  }
  .card h1{font-size:22px; margin:0 0 4px;}
  .card .lede{color:var(--ink-soft); font-size:14px; margin:0 0 24px;}

  .tabs{display:flex; border:1px solid var(--line); border-radius:10px; padding:3px; margin-bottom:24px;}
  .tab-btn{
    flex:1; background:none; border:none; padding:9px 0; border-radius:8px;
    font-family:var(--font-sans); font-size:14px; font-weight:600; color:var(--ink-soft); cursor:pointer;
  }
  .tab-btn.active{background:var(--accent); color:var(--accent-ink);}

  .field{margin-bottom:16px;}
  .field label{display:block; font-size:13px; font-weight:600; color:var(--ink-soft); margin-bottom:6px;}
  .field input, .field select{
    width:100%; padding:10px 12px; border:1px solid var(--line); border-radius:8px;
    background:var(--surface); color:var(--ink); font-family:var(--font-sans); font-size:14px;
  }
  .field input:focus, .field select:focus{outline:2px solid var(--accent); outline-offset:1px;}

  .btn-primary{
    width:100%; padding:11px 0; border:none; border-radius:8px; background:var(--accent);
    color:var(--accent-ink); font-family:var(--font-sans); font-size:14px; font-weight:600; cursor:pointer;
  }
  .btn-primary:disabled{opacity:.6; cursor:default;}
  .btn-google{
    width:100%; padding:10px 0; border:1px solid var(--line); border-radius:8px; background:var(--surface);
    color:var(--ink); font-family:var(--font-sans); font-size:14px; font-weight:600; cursor:pointer;
    display:flex; align-items:center; justify-content:center; gap:10px; margin-top:10px;
  }
  .divider{display:flex; align-items:center; gap:12px; color:var(--ink-faint); font-size:12px; margin:20px 0;}
  .divider::before, .divider::after{content:''; flex:1; height:1px; background:var(--line);}

  .error-text{color:var(--bad); font-size:13px; margin:-8px 0 16px; display:none;}
  .error-text.show{display:block;}

  .welcome dl{display:grid; grid-template-columns:auto 1fr; gap:8px 16px; margin:20px 0; font-size:14px;}
  .welcome dt{color:var(--ink-soft);}
  .welcome dd{margin:0; font-weight:600;}
  .status-line{background:var(--surface-2); border-radius:10px; padding:14px 16px; font-size:13px; color:var(--ink-soft); margin-bottom:20px;}
  .link-row{display:flex; justify-content:space-between; margin-top:20px; font-size:13px;}
  .link-row a{color:var(--accent); text-decoration:none; font-weight:600;}

  .screen{display:none;}
  .screen.active{display:block;}
</style>
</head>
<body>

<div class="topbar">
  <a class="brand" href="index.html">
    <svg width="26" height="26" viewBox="0 0 26 26" fill="none" aria-hidden="true"><circle cx="13" cy="13" r="11" stroke="var(--accent)" stroke-width="1.3" opacity="0.35"/><circle cx="13" cy="13" r="8" stroke="var(--accent)" stroke-width="1.3" opacity="0.55"/><circle cx="13" cy="13" r="5.2" stroke="var(--accent)" stroke-width="1.3" opacity="0.8"/><circle cx="13" cy="13" r="2.1" fill="var(--accent)"/></svg>
    Rudhira
  </a>
  <button class="theme-toggle" id="themeToggle" type="button" aria-label="Switch to dark mode">
    <svg class="icon-sun" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round"><circle cx="12" cy="12" r="4"/><path d="M12 2v2M12 20v2M4.9 4.9l1.4 1.4M17.7 17.7l1.4 1.4M2 12h2M20 12h2M4.9 19.1l1.4-1.4M17.7 6.3l1.4-1.4"/></svg>
    <svg class="icon-moon" viewBox="0 0 24 24" fill="currentColor"><path d="M20 14.5A8.5 8.5 0 1 1 9.5 4a7 7 0 0 0 10.5 10.5z"/></svg>
  </button>
</div>

<div class="card-wrap">
  <div class="card">

    <div class="screen active" id="screen-auth">
      <h1>Join Rudhira</h1>
      <p class="lede">Create a donor account, or log back in.</p>

      <div class="tabs" role="tablist">
        <button class="tab-btn active" id="tab-signup" type="button" role="tab" aria-selected="true">Sign Up</button>
        <button class="tab-btn" id="tab-login" type="button" role="tab" aria-selected="false">Log In</button>
      </div>

      <form id="form-signup" novalidate>
        <div class="field"><label for="su-name">Name</label><input id="su-name" name="name" type="text" required></div>
        <div class="field"><label for="su-email">Email</label><input id="su-email" name="email" type="email" required></div>
        <div class="field"><label for="su-age">Age</label><input id="su-age" name="age" type="number" min="1" max="120" required></div>
        <div class="field">
          <label for="su-bloodgroup">Blood Group</label>
          <select id="su-bloodgroup" name="bloodGroup" required>
            <option value="" disabled selected>Select your blood group</option>
            <option>A+</option><option>A-</option><option>B+</option><option>B-</option>
            <option>O+</option><option>O-</option><option>AB+</option><option>AB-</option>
          </select>
        </div>
        <div class="field"><label for="su-password">Password</label><input id="su-password" name="password" type="password" minlength="6" required></div>
        <div class="field"><label for="su-confirm">Confirm Password</label><input id="su-confirm" name="confirm" type="password" minlength="6" required></div>
        <p class="error-text" id="su-error"></p>
        <button class="btn-primary" id="su-submit" type="submit">Create account</button>
      </form>

      <form id="form-login" novalidate style="display:none;">
        <div class="field"><label for="li-email">Email</label><input id="li-email" name="email" type="email" required></div>
        <div class="field"><label for="li-password">Password</label><input id="li-password" name="password" type="password" required></div>
        <p class="error-text" id="li-error"></p>
        <button class="btn-primary" id="li-submit" type="submit">Log in</button>
      </form>

      <div class="divider">or</div>
      <button class="btn-google" id="google-btn" type="button">
        <svg width="18" height="18" viewBox="0 0 18 18"><path fill="#4285F4" d="M17.64 9.2c0-.64-.06-1.25-.16-1.84H9v3.48h4.84c-.21 1.13-.85 2.09-1.81 2.73v2.27h2.92c1.71-1.57 2.69-3.88 2.69-6.64z"/><path fill="#34A853" d="M9 18c2.43 0 4.47-.8 5.96-2.17l-2.92-2.27c-.81.54-1.84.86-3.04.86-2.34 0-4.32-1.58-5.03-3.71H.96v2.34C2.44 15.98 5.48 18 9 18z"/><path fill="#FBBC05" d="M3.97 10.71c-.18-.54-.28-1.11-.28-1.71s.1-1.17.28-1.71V4.95H.96A8.996 8.996 0 0 0 0 9c0 1.45.35 2.83.96 4.05l3.01-2.34z"/><path fill="#EA4335" d="M9 3.58c1.32 0 2.51.45 3.44 1.35l2.59-2.59C13.46.89 11.43 0 9 0 5.48 0 2.44 2.02.96 4.95l3.01 2.34C4.68 5.16 6.66 3.58 9 3.58z"/></svg>
        Continue with Google
      </button>
      <p class="error-text" id="google-error"></p>
    </div>

    <div class="screen" id="screen-complete">
      <h1>One more step</h1>
      <p class="lede">We just need your blood group and age to finish setting up your donor account.</p>
      <form id="form-complete" novalidate>
        <div class="field"><label for="pc-age">Age</label><input id="pc-age" name="age" type="number" min="1" max="120" required></div>
        <div class="field">
          <label for="pc-bloodgroup">Blood Group</label>
          <select id="pc-bloodgroup" name="bloodGroup" required>
            <option value="" disabled selected>Select your blood group</option>
            <option>A+</option><option>A-</option><option>B+</option><option>B-</option>
            <option>O+</option><option>O-</option><option>AB+</option><option>AB-</option>
          </select>
        </div>
        <p class="error-text" id="pc-error"></p>
        <button class="btn-primary" id="pc-submit" type="submit">Finish</button>
      </form>
    </div>

    <div class="screen welcome" id="screen-welcome">
      <h1 id="welcome-heading">Welcome</h1>
      <dl>
        <dt>Blood Group</dt><dd id="w-bloodgroup"></dd>
        <dt>Age</dt><dd id="w-age"></dd>
        <dt>Email</dt><dd id="w-email"></dd>
      </dl>
      <p class="status-line" id="w-status">0 verified donations &mdash; donate once to unlock Bronze.</p>
      <div class="link-row">
        <a href="index.html">&larr; Back to Rudhira</a>
        <a href="#" id="logout-link">Log out</a>
      </div>
    </div>

  </div>
</div>

<script>
  (function initTheme(){
    const root = document.documentElement;
    const btn = document.getElementById('themeToggle');
    const stored = localStorage.getItem('rudhira-theme');
    if(stored === 'light' || stored === 'dark') root.setAttribute('data-theme', stored);
    function effectiveDark(){
      const explicit = root.getAttribute('data-theme');
      if(explicit === 'dark') return true;
      if(explicit === 'light') return false;
      return window.matchMedia('(prefers-color-scheme: dark)').matches;
    }
    function syncButton(){
      const dark = effectiveDark();
      btn.classList.toggle('is-dark', dark);
      btn.setAttribute('aria-label', dark ? 'Switch to light mode' : 'Switch to dark mode');
    }
    btn.addEventListener('click', function(){
      const next = effectiveDark() ? 'light' : 'dark';
      root.setAttribute('data-theme', next);
      localStorage.setItem('rudhira-theme', next);
      syncButton();
    });
    window.matchMedia('(prefers-color-scheme: dark)').addEventListener('change', function(){
      if(!localStorage.getItem('rudhira-theme')) syncButton();
    });
    syncButton();
  })();

  (function initTabs(){
    var tabSignup = document.getElementById('tab-signup');
    var tabLogin = document.getElementById('tab-login');
    var formSignup = document.getElementById('form-signup');
    var formLogin = document.getElementById('form-login');
    tabSignup.addEventListener('click', function(){
      tabSignup.classList.add('active'); tabSignup.setAttribute('aria-selected','true');
      tabLogin.classList.remove('active'); tabLogin.setAttribute('aria-selected','false');
      formSignup.style.display = ''; formLogin.style.display = 'none';
    });
    tabLogin.addEventListener('click', function(){
      tabLogin.classList.add('active'); tabLogin.setAttribute('aria-selected','true');
      tabSignup.classList.remove('active'); tabSignup.setAttribute('aria-selected','false');
      formLogin.style.display = ''; formSignup.style.display = 'none';
    });
  })();

  function showScreen(id){
    document.querySelectorAll('.screen').forEach(function(el){ el.classList.remove('active'); });
    document.getElementById(id).classList.add('active');
  }

  document.getElementById('form-signup').addEventListener('submit', function(e){ e.preventDefault(); });
  document.getElementById('form-login').addEventListener('submit', function(e){ e.preventDefault(); });
  document.getElementById('form-complete').addEventListener('submit', function(e){ e.preventDefault(); });
</script>

</body>
</html>
```

- [ ] **Step 2: Verify visually with the gstack browse skill**

Open `website/join.html` locally (e.g. `file:///C:/Users/shrim/OneDrive/Desktop/ventureX/website/join.html`) via `/browse`. Confirm: page loads with no console errors; clicking "Log In" tab swaps the form and highlights the tab; clicking the theme toggle switches the whole page dark/light and persists across a reload; clicking "Sign Up" / "Log In" / "Finish" buttons does nothing (expected — no handlers yet) and does not navigate away or throw errors.

- [ ] **Step 3: Commit**

```bash
git add website/join.html
git commit -m "Add static shell for donor sign up / log in page"
```

---

### Task 3: Firebase init, shared helpers, auth-state listener, logout

**Files:**
- Modify: `website/join.html` (append before `</body>`, after the existing `<script>` block from Task 2)

**Interfaces:**
- Consumes: `showScreen(id)` from Task 2; element IDs from Task 2.
- Produces: `auth` (firebase.auth.Auth instance), `db` (firebase.firestore.Firestore instance), `mapAuthError(err) -> string|null`, `showError(elId, message)`, `renderWelcome(user, profile)` — used by Tasks 4, 5, 6.

- [ ] **Step 1: Add the Firebase SDK script tags and init block**

Insert this new `<script>` block right after the closing `</script>` tag added in Task 2 (still before `</body>`), replacing `REPLACE_WITH_*` with the real values from Task 1:

```html
<script src="https://www.gstatic.com/firebasejs/10.12.2/firebase-app-compat.js"></script>
<script src="https://www.gstatic.com/firebasejs/10.12.2/firebase-auth-compat.js"></script>
<script src="https://www.gstatic.com/firebasejs/10.12.2/firebase-firestore-compat.js"></script>
<script>
  var firebaseConfig = {
    apiKey: "REPLACE_WITH_API_KEY",
    authDomain: "REPLACE_WITH_PROJECT_ID.firebaseapp.com",
    projectId: "REPLACE_WITH_PROJECT_ID",
    storageBucket: "REPLACE_WITH_PROJECT_ID.appspot.com",
    messagingSenderId: "REPLACE_WITH_SENDER_ID",
    appId: "REPLACE_WITH_APP_ID"
  };
  firebase.initializeApp(firebaseConfig);
  var auth = firebase.auth();
  var db = firebase.firestore();

  function mapAuthError(err){
    var map = {
      'auth/email-already-in-use': 'An account with this email already exists. Try logging in instead.',
      'auth/weak-password': 'Password must be at least 6 characters.',
      'auth/invalid-email': "That email address doesn't look right.",
      'auth/user-not-found': 'No account found with that email.',
      'auth/wrong-password': 'Incorrect password.',
      'auth/popup-closed-by-user': null
    };
    return map.hasOwnProperty(err.code) ? map[err.code] : 'Something went wrong. Please try again.';
  }

  function showError(elId, message){
    var el = document.getElementById(elId);
    if(!message){ el.classList.remove('show'); return; }
    el.textContent = message;
    el.classList.add('show');
  }

  function renderWelcome(user, profile){
    document.getElementById('welcome-heading').textContent = 'Welcome, ' + profile.name;
    document.getElementById('w-bloodgroup').textContent = profile.bloodGroup;
    document.getElementById('w-age').textContent = profile.age;
    document.getElementById('w-email').textContent = profile.email;
    showScreen('screen-welcome');
  }

  auth.onAuthStateChanged(function(user){
    if(!user){ showScreen('screen-auth'); return; }
    db.collection('donors').doc(user.uid).get().then(function(doc){
      if(doc.exists){
        renderWelcome(user, doc.data());
      } else {
        showScreen('screen-complete');
      }
    });
  });

  document.getElementById('logout-link').addEventListener('click', function(e){
    e.preventDefault();
    auth.signOut();
  });
</script>
```

- [ ] **Step 2: Verify with the gstack browse skill**

Reload `join.html`. Confirm no console errors (Firebase SDK loaded and initialized correctly). The page should still show the Sign Up form (no user signed in yet, so `onAuthStateChanged` fires with `user = null` and calls `showScreen('screen-auth')`).

- [ ] **Step 3: Commit**

```bash
git add website/join.html
git commit -m "Wire Firebase init, auth-state listener, and shared helpers into join.html"
```

---

### Task 4: Email/password Sign Up

**Files:**
- Modify: `website/join.html:` the line `document.getElementById('form-signup').addEventListener('submit', function(e){ e.preventDefault(); });` from Task 2

**Interfaces:**
- Consumes: `auth`, `db`, `mapAuthError`, `showError`, `renderWelcome` from Task 3.

- [ ] **Step 1: Replace the guard listener with the real handler**

Find this line (from Task 2):
```js
document.getElementById('form-signup').addEventListener('submit', function(e){ e.preventDefault(); });
```

Replace it with:
```js
document.getElementById('form-signup').addEventListener('submit', function(e){
  e.preventDefault();
  showError('su-error', null);
  var name = document.getElementById('su-name').value.trim();
  var email = document.getElementById('su-email').value.trim();
  var age = parseInt(document.getElementById('su-age').value, 10);
  var bloodGroup = document.getElementById('su-bloodgroup').value;
  var password = document.getElementById('su-password').value;
  var confirm = document.getElementById('su-confirm').value;

  if(!bloodGroup){ showError('su-error', 'Please select a blood group.'); return; }
  if(!(age >= 1 && age <= 120)){ showError('su-error', 'Please enter a valid age.'); return; }
  if(password !== confirm){ showError('su-error', 'Passwords do not match.'); return; }

  var submitBtn = document.getElementById('su-submit');
  submitBtn.disabled = true;

  auth.createUserWithEmailAndPassword(email, password).then(function(cred){
    var profile = {
      name: name, email: email, age: age, bloodGroup: bloodGroup,
      createdAt: firebase.firestore.FieldValue.serverTimestamp()
    };
    return db.collection('donors').doc(cred.user.uid).set(profile).then(function(){
      renderWelcome(cred.user, profile);
    });
  }).catch(function(err){
    var msg = mapAuthError(err);
    if(msg) showError('su-error', msg);
    submitBtn.disabled = false;
  });
});
```

- [ ] **Step 2: Verify with the gstack browse skill**

Fill in the Sign Up form with a fresh test email (e.g. `test1@example.com`), a valid age (e.g. 24), a blood group, and matching passwords ("test123" / "test123"). Submit. Confirm: the page transitions to the Welcome screen showing "Welcome, {name}", the correct blood group, age, and email, and the status line "0 verified donations — donate once to unlock Bronze." Confirm in the Firebase console (Firestore Database → Data) that a `donors/{uid}` document now exists with the matching fields.

- [ ] **Step 3: Verify error cases**

Reload, sign out if needed, try submitting the same email again → expect inline error "An account with this email already exists. Try logging in instead." Try mismatched passwords → expect "Passwords do not match." Try age `200` → expect "Please enter a valid age."

- [ ] **Step 4: Commit**

```bash
git add website/join.html
git commit -m "Implement email/password sign up flow"
```

---

### Task 5: Email/password Log In

**Files:**
- Modify: `website/join.html:` the line `document.getElementById('form-login').addEventListener('submit', function(e){ e.preventDefault(); });` from Task 2

**Interfaces:**
- Consumes: `auth`, `db`, `mapAuthError`, `showError`, `renderWelcome` from Task 3.

- [ ] **Step 1: Replace the guard listener with the real handler**

Find:
```js
document.getElementById('form-login').addEventListener('submit', function(e){ e.preventDefault(); });
```

Replace with:
```js
document.getElementById('form-login').addEventListener('submit', function(e){
  e.preventDefault();
  showError('li-error', null);
  var email = document.getElementById('li-email').value.trim();
  var password = document.getElementById('li-password').value;
  var submitBtn = document.getElementById('li-submit');
  submitBtn.disabled = true;

  auth.signInWithEmailAndPassword(email, password).then(function(cred){
    return db.collection('donors').doc(cred.user.uid).get().then(function(doc){
      renderWelcome(cred.user, doc.data());
    });
  }).catch(function(err){
    var msg = mapAuthError(err);
    if(msg) showError('li-error', msg);
    submitBtn.disabled = false;
  });
});
```

- [ ] **Step 2: Verify with the gstack browse skill**

Click "Log out" on the Welcome screen from Task 4's test account, land back on the auth screen, switch to the "Log In" tab, sign in with `test1@example.com` / `test123`. Confirm the Welcome screen shows the same name/blood group/age/email as before.

- [ ] **Step 3: Verify error cases**

Try a nonexistent email → expect "No account found with that email." Try the correct email with a wrong password → expect "Incorrect password."

- [ ] **Step 4: Commit**

```bash
git add website/join.html
git commit -m "Implement email/password log in flow"
```

---

### Task 6: Google Sign-In + first-time profile completion

**Files:**
- Modify: `website/join.html` (add a new listener for `google-btn`, replace the guard listener for `form-complete`)

**Interfaces:**
- Consumes: `auth`, `db`, `mapAuthError`, `showError`, `renderWelcome`, `showScreen` from Tasks 2–3.

- [ ] **Step 1: Add the Google Sign-In handler**

Add this new block right after the `form-login` submit handler from Task 5 (still inside the same `<script>`):

```js
document.getElementById('google-btn').addEventListener('click', function(){
  showError('google-error', null);
  var provider = new firebase.auth.GoogleAuthProvider();
  auth.signInWithPopup(provider).then(function(cred){
    return db.collection('donors').doc(cred.user.uid).get().then(function(doc){
      if(doc.exists){
        renderWelcome(cred.user, doc.data());
      } else {
        showScreen('screen-complete');
      }
    });
  }).catch(function(err){
    var msg = mapAuthError(err);
    if(msg) showError('google-error', msg);
  });
});
```

- [ ] **Step 2: Replace the `form-complete` guard listener**

Find (from Task 2):
```js
document.getElementById('form-complete').addEventListener('submit', function(e){ e.preventDefault(); });
```

Replace with:
```js
document.getElementById('form-complete').addEventListener('submit', function(e){
  e.preventDefault();
  showError('pc-error', null);
  var user = auth.currentUser;
  var age = parseInt(document.getElementById('pc-age').value, 10);
  var bloodGroup = document.getElementById('pc-bloodgroup').value;

  if(!bloodGroup){ showError('pc-error', 'Please select a blood group.'); return; }
  if(!(age >= 1 && age <= 120)){ showError('pc-error', 'Please enter a valid age.'); return; }

  var submitBtn = document.getElementById('pc-submit');
  submitBtn.disabled = true;

  var profile = {
    name: user.displayName || 'Donor', email: user.email, age: age, bloodGroup: bloodGroup,
    createdAt: firebase.firestore.FieldValue.serverTimestamp()
  };
  db.collection('donors').doc(user.uid).set(profile).then(function(){
    renderWelcome(user, profile);
  }).catch(function(err){
    showError('pc-error', mapAuthError(err));
    submitBtn.disabled = false;
  });
});
```

- [ ] **Step 3: Verify with the gstack browse skill**

Click "Continue with Google", complete the popup with a real Google account not used before on this Firebase project. Confirm the "One more step" screen appears with no name/email fields (they're implicit). Fill in age + blood group, submit. Confirm the Welcome screen shows the Google account's name/email plus the entered age/blood group, and that Firestore now has a `donors/{uid}` doc for that account.

- [ ] **Step 4: Verify the returning-user path**

Log out, click "Continue with Google" again with the same account. Confirm it goes straight to the Welcome screen, skipping "One more step".

- [ ] **Step 5: Commit**

```bash
git add website/join.html
git commit -m "Implement Google sign-in and first-time profile completion"
```

---

### Task 7: Firestore security rules reference file

**Files:**
- Create: `website/firestore.rules`

**Interfaces:** none (documentation artifact; the live rule is applied manually in the Firebase console per Task 1 Step 4).

- [ ] **Step 1: Write the reference file**

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

- [ ] **Step 2: Verify the rule is actually live**

In the Firebase console, go to Firestore → Rules and confirm the published rule matches this file exactly (this checks Task 1 Step 4 was done correctly, not just that this reference file exists).

- [ ] **Step 3: Commit**

```bash
git add website/firestore.rules
git commit -m "Add Firestore security rules reference file"
```

---

### Task 8: Nav integration in `index.html`

**Files:**
- Modify: `website/index.html:610-617` (the `.nav-right` div)

**Interfaces:** none — this is a leaf UI change, nothing later depends on it.

- [ ] **Step 1: Add the "Join as Donor" button**

Find (in `website/index.html`):
```html
    <div class="nav-right">
      <button class="theme-toggle" id="themeToggle" type="button" aria-label="Switch to dark mode">
        <svg class="icon-sun" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round"><circle cx="12" cy="12" r="4"/><path d="M12 2v2M12 20v2M4.9 4.9l1.4 1.4M17.7 17.7l1.4 1.4M2 12h2M20 12h2M4.9 19.1l1.4-1.4M17.7 6.3l1.4-1.4"/></svg>
        <svg class="icon-moon" viewBox="0 0 24 24" fill="currentColor"><path d="M20 14.5A8.5 8.5 0 1 1 9.5 4a7 7 0 0 0 10.5 10.5z"/></svg>
      </button>
      <button class="lang-toggle" id="langToggle" type="button">हिं</button>
      <a class="navcta" href="#ask" data-i18n="nav.ask">The ask</a>
    </div>
```

Replace with (adds one `<a class="navcta">` before the existing "The ask" link):
```html
    <div class="nav-right">
      <button class="theme-toggle" id="themeToggle" type="button" aria-label="Switch to dark mode">
        <svg class="icon-sun" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round"><circle cx="12" cy="12" r="4"/><path d="M12 2v2M12 20v2M4.9 4.9l1.4 1.4M17.7 17.7l1.4 1.4M2 12h2M20 12h2M4.9 19.1l1.4-1.4M17.7 6.3l1.4-1.4"/></svg>
        <svg class="icon-moon" viewBox="0 0 24 24" fill="currentColor"><path d="M20 14.5A8.5 8.5 0 1 1 9.5 4a7 7 0 0 0 10.5 10.5z"/></svg>
      </button>
      <button class="lang-toggle" id="langToggle" type="button">हिं</button>
      <a class="navcta" href="join.html" style="background:var(--surface-2); color:var(--ink); border:1px solid var(--line);">Join as Donor</a>
      <a class="navcta" href="#ask" data-i18n="nav.ask">The ask</a>
    </div>
```

(The inline style keeps "Join as Donor" visually secondary — an outlined button — so the solid red `.navcta` styling stays reserved for "The ask", the investor-facing CTA. Two solid red buttons side by side would compete for attention.)

- [ ] **Step 2: Verify with the gstack browse skill**

Reload `index.html` in both light and dark theme. Confirm "Join as Donor" appears before "The ask", is visually distinct (outlined vs. solid), and clicking it navigates to `join.html`. Confirm no layout overflow/wrapping issues in the nav at both desktop and the `max-width:1100px` breakpoint.

- [ ] **Step 3: Commit**

```bash
git add website/index.html
git commit -m "Add Join as Donor nav link to index.html"
```

---

### Task 9: Full end-to-end QA pass

**Files:** none created/modified — verification only, against the spec's Testing Plan section.

- [ ] **Step 1:** Using the gstack `browse` skill, run through the full spec testing checklist in one pass:
  - Sign up with email/password → Firestore doc created with correct fields → Welcome screen shows correct data.
  - Log out, log back in with the same email/password → Welcome screen shows the same data.
  - Sign up with Google (first time) → profile-completion step appears, pre-filled name/email correct → submitting creates the doc → Welcome screen correct.
  - Log out, sign in with Google again (returning user) → skips profile-completion, goes straight to Welcome screen.
  - Reload `join.html` while signed in → lands on Welcome screen without re-prompting for login.
  - Invalid inputs: duplicate email signup, wrong password, mismatched confirm-password, age out of 1–120 range → correct inline errors, no console errors.
  - Dark/light theme carries over correctly from `index.html`'s stored preference (`rudhira-theme` key).
  - Nav "Join as Donor" button visible and correctly linked from `index.html` in both themes.
- [ ] **Step 2:** Fix anything that fails, re-verify, and commit the fix with a description of what broke.
- [ ] **Step 3:** Report back with a summary of what was verified — do not claim the feature "works" without having actually run every item above through the browser.

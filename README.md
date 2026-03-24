# QR Attendance System — Final v4

One HTML file, no build step, no dependencies to install.

---

## What was fixed

| Bug | Root cause | Fix |
|-----|-----------|-----|
| "QR code invalid" | Session data only existed in the lecturer's `localStorage`; students' phones had no access | Session data is now **encoded directly in the QR URL** as URL-safe base64. Any phone that scans the code decodes everything it needs instantly — zero network required |
| Tab switching broken | Tabs matched by text content (brittle) | Tabs now use `data-tab` attributes — clean and reliable |
| Code generation blocking UI | 100,000 codes generated synchronously on load | Generated in async chunks; count reduced to **10,000 access codes** (more than enough) |
| Lecturer IDs | Sequential, predictable | Randomly sampled from pool of 100,000 (LEC-000001…LEC-100000) |
| Duplicate check-in race conditions | Reads and writes not serialised | All checks happen in sequence before any write |
| XSS vectors | User strings injected into innerHTML | All user content escaped through `esc()` helper |

---

## Files — upload all 4 to GitHub

```
index.html     ← Complete app (all HTML + CSS + JS inline)
sw.js          ← Service worker (offline caching)
manifest.json  ← PWA manifest (installable on phone)
README.md      ← This file
```

---

## Deploy to GitHub Pages (2 minutes)

1. Create a new GitHub repository
2. Upload all 4 files to the root
3. Go to **Settings → Pages → Source → Deploy from branch → main / root → Save**
4. Live at: `https://<username>.github.io/<repo>/`

**Share with lecturers:** `https://<username>.github.io/<repo>/`  
Students only arrive through the QR code — they cannot navigate to any other view.

---

## Firebase Setup (10 minutes) — enables real cross-device sync

Without Firebase, the app runs in **demo mode**: everything is stored in the browser's localStorage, so the lecturer and student must be on the same device. For real classroom use, set up Firebase.

### Step 1 — Create project
1. Go to https://console.firebase.google.com
2. **Add project** → name it (e.g. `qr-attendance`) → Continue → Create project

### Step 2 — Enable Email/Password Authentication
Build → Authentication → Get started → Email/Password → Enable → Save

### Step 3 — Enable Realtime Database
Build → Realtime Database → Create database → choose region → **Start in test mode** → Enable

### Step 4 — Get config values
Project Settings (gear icon) → Your apps → Web `</>` → Register app → Copy the config object

### Step 5 — Paste into index.html
Find the `FB_CONFIG` block near the top of the `<script>` and replace the placeholder values:

```javascript
const FB_CONFIG = {
  apiKey:            "AIzaSy...",
  authDomain:        "your-project.firebaseapp.com",
  databaseURL:       "https://your-project-default-rtdb.firebaseio.com",
  projectId:         "your-project",
  storageBucket:     "your-project.appspot.com",
  messagingSenderId: "123456789",
  appId:             "1:123:web:abc..."
};
```

### Step 6 — Set database security rules
Firebase Console → Realtime Database → Rules:

```json
{
  "rules": {
    "sessions": {
      ".read": true,
      "$sessionId": {
        ".write": true
      }
    },
    "lecturers": {
      ".read": "auth != null",
      ".write": "auth != null"
    },
    "backup": {
      ".read": "auth != null",
      ".write": "auth != null"
    }
  }
}
```

---

## Custom domain + Google Search (Point 11)

### Custom domain via GitHub Pages
1. Buy a domain (Namecheap, GoDaddy, etc.)
2. GitHub repo → Settings → Pages → Custom domain → enter domain → Save
3. At your DNS provider, add:
   ```
   Type:  CNAME
   Name:  www
   Value: <username>.github.io
   ```
4. Wait up to 48 hours for DNS propagation
5. GitHub provisions HTTPS automatically

### Google Search Console indexing
1. https://search.google.com/search-console → Add property
2. Verify ownership via **DNS TXT record** (your DNS provider's control panel)
3. Once verified, go to **URL Inspection → Request Indexing**
4. The `<meta name="description">` and `<meta name="keywords">` tags are already in `index.html`

---

## First-time setup checklist

- [ ] Deploy files to GitHub Pages (or your hosting)
- [ ] Open the app → Admin → default password is `admin1234`
- [ ] Go to Settings → **Change the admin password immediately**
- [ ] Access Codes → **Export unused CSV** → email one code per lecturer
- [ ] Lecturers visit the site, choose Lecturer → Register, enter their code
- [ ] Students only ever need to scan the QR code — no login, no app

---

## All 11 features

| # | Feature | Implementation |
|---|---------|----------------|
| — | **QR bug fixed** | Session encoded in URL as URL-safe base64 |
| 1 | Multi-device login | Firebase Auth; any device with the email + password signs in |
| 2 | Auto-assigned Lecturer ID | Random from pool LEC-000001…LEC-100000 at signup |
| 3 | Downloadable QR | "Download QR" button saves PNG directly |
| 4 | 100,000 unique IDs | LEC-XXXXXX pool, randomly sampled |
| 5 | Works offline | Service worker caches app; Firebase offline persistence; check-in queue in localStorage |
| 6 | Biometric login | WebAuthn (Face ID / Fingerprint) registered at signup, available on login |
| 7 | Admin backup | Auto-saved on every session end; separated by lecturer ID; preserved even if lecturer deletes their records |
| 8 | Combined attendance file | "Combined Reports" tab per lecturer; "Master CSV" per course |
| 9 | Co-admins with approval | Super admin adds co-admins; destructive actions (delete/revoke) queued as pending, require super-admin approval |
| 10 | 1,000+ concurrent users | Firebase Realtime DB scales to millions; localStorage fallback for single-device demo |
| 11 | Custom URL + Google Search | GitHub Pages custom domain guide + Search Console indexing steps above |

---

## Security model

| Check | How it works |
|-------|-------------|
| Lecturer registration | Valid, unused access code required |
| Login | Firebase Auth email + password; WebAuthn biometrics optional |
| One device per session | Browser fingerprint (UA, screen, timezone, platform) stored per session |
| Unique student ID | Stored per session; duplicates rejected with name of original registrant shown |
| Location fence | Haversine GPS distance compared to classroom anchor |
| QR expiry | Expiry timestamp embedded in QR URL; validated client-side |
| Co-admin actions | Queued as pending approvals; only super admin can execute |
| Admin backup | Written to Firebase on every session end; never deleted by lecturer actions |

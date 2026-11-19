# Theingabar Invoice — Firebase variant (Firestore + Hosting)

The same PWA as the Render backend variant, but the server is Firebase
instead of a Node Web Service + PostgreSQL:

- **Store:** Cloud Firestore, collection `invoices`. Documents:
  `{ uid, date, customer, items: [{desc, amount}], total, createdAt, updatedAt }`
- **Hosting:** Firebase Hosting serves `public/` (the PWA).
- **Sync:** on open, the PWA loads Firestore invoices (newest first) as
  the source of truth and rewrites the phone's IndexedDB to match.
  Saving writes to Firestore, then locally; deleting deletes the
  Firestore document, then locally. Offline, everything still works on
  the phone (IndexedDB) and syncs on the next open.
- **Validation:** client-side before every Firestore write (date/customer
  required, max 20 items, integer amounts ≥ 0, total must equal the sum
  of items) **and** in `firestore.rules` (same checks, enforced
  server-side — the total==sum check is written out explicitly because
  security rules have no loops/reduce).
- **PIN lock:** unchanged, local only. **QR, layout, fonts, CSV export:**
  unchanged.

## Why Firebase instead of Render?

- No 90-day expiry (Render's free PostgreSQL deletes its data after
  90 days). Firestore on the free Spark plan keeps data.
- No sleeping server: Render's free Web Service spins down after
  inactivity (first open takes 30–60 s to wake). Firebase Hosting +
  Firestore answer immediately.
- Free-tier limits (Spark plan, per project): 1 GiB Firestore storage,
  50,000 document reads / 20,000 writes / 20,000 deletes per day,
  10 GiB/month Hosting transfer. A shop invoice log is far below this.

## Setup (one time)

1. **Create the Firebase project** at
   https://console.firebase.google.com — Add project, name it e.g.
   `theingabar-invoice`. Google Analytics: off (not needed).
2. **Enable Firestore:** Build → Firestore Database → Create database →
   **Production mode** → location `asia-southeast1` (Singapore, closest)
   if offered.
3. **Register the web app:** Project settings (gear) → Your apps →
   `</>` Web app → nickname `theingabar-invoice` → copy the
   `firebaseConfig` object.
4. **Paste the config** into `public/index.html`: find
   `FIREBASE_CONFIG` near the end of the file (in the
   `<script type="module">` block) and replace every
   `"PASTE-YOURS-HERE"` value with the real one from step 3.
   Until then the app shows "Firebase: Not configured" and works
   fully on local IndexedDB only.
5. **Install the Firebase CLI** (needs Node 18+ on the machine that
   deploys — a phone cannot run it; use a PC, or deploy via the
   browser/CI):
   ```sh
   npm install -g firebase-tools
   firebase login
   ```
6. **Point the CLI at your project** — edit `.firebaserc` and replace
   `"theingabar-invoice"` with your real Firebase project id
   (the one from step 1).
7. **Deploy rules + hosting:**
   ```sh
   firebase deploy --only firestore:rules,firestore:indexes,hosting
   ```
   Expected URL: `https://<your-project-id>.web.app`
   (also `https://<your-project-id>.firebaseapp.com`).

## First open — seeding

On the first successful sync, if Firestore is empty and the phone's
local log is not empty, the app uploads each local invoice to
Firestore — so the Nissan Safari invoice (19.11.2026, 120,000 +
50,000 + 130,000 = 300,000 MMK) uploads itself. No manual seed step.
The status line in the Invoice Log should read **Firebase: Synced**.

## Security notes (read this)

- **v1 has no Firebase Auth.** The web API key in `FIREBASE_CONFIG`
  is public by design (it identifies the project, it is not a secret),
  and `firestore.rules` allows anyone who knows the URL to *read* the
  log. Writes are validated (see `firestore.rules`), but anyone could
  write a *valid-looking* invoice or delete one. For a single-shop
  log this is the same exposure as the Render API-key variant
  (the key lived in the app), but be aware of it.
- **v2 (recommended):** enable Firebase Authentication (e.g. Google
  sign-in for the shop's Google account) and change
  `firestore.rules` to `allow read, write: if request.auth != null`
  (plus the `isValidInvoice(...)` check). Then only a signed-in
  shop phone can read or write.

## Local preview (before Firebase is configured)

No build step — the PWA is plain files:

```sh
cd public && python3 -m http.server 8000
# open http://localhost:8000/ — shows "Firebase: Not configured",
# seed Nissan Safari invoice in the log, PIN lock works.
```

## Files

| File | What it is |
|---|---|
| `public/index.html` | The PWA (v3, Firebase sync layer) |
| `public/manifest.webmanifest`, `public/sw.js`, `public/icon-*.png`, `public/apple-touch-icon.png` | PWA manifest, service worker (never caches Firebase/Google hosts), icons |
| `firebase.json` | Hosting (serves `public/`, SPA rewrites) + Firestore rules/indexes |
| `firestore.rules` | Write validation (incl. total == sum of items) |
| `firestore.indexes.json` | `invoices` ordered by `createdAt` desc |
| `.firebaserc` | Default Firebase project id (placeholder — replace) |
| `.gitignore` | `node_modules/`, `.firebase/`, `firebase-debug.log` |

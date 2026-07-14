# Us

A private, single-page web app for two people. It's a shared digital notebook for a
couple — a home feed, notes to each other, a shared to-do/rewards system, and a plans
calendar — wrapped in a hand-drawn, sticky-note/polaroid visual style.

Live at: **https://gyamdenmanil.github.io/Kiman/**

## What's in this repo

This is intentionally a *no-build* project — everything lives in one file.

```
.
├── index.html     # the entire app: markup, CSS, and React source (in a <script type="text/plain"> tag)
├── icon-180.png   # apple-touch-icon used when the app is added to an iOS home screen
└── .gitignore
```

There is no `package.json`, bundler, or build step. `index.html` loads React, ReactDOM,
Babel Standalone, and the Firebase Auth compat SDK from CDNs (`unpkg.com` /
`gstatic.com`), then Babel transpiles the app's JSX **in the browser** from the inline
`<script type="text/plain" id="app-src">` block. Opening `index.html` directly (or
serving the folder with any static file server) runs the full app — no `npm install`
needed.

## How it works

- **UI**: React 18 (via CDN, `production.min.js` builds), written as JSX and transpiled
  client-side by Babel Standalone. No JSX precompilation happens at commit time.
- **Auth**: Firebase Authentication (Google Sign-In popup). Access is meant to be
  limited to two specific Google accounts.
- **Data**: Firestore, accessed directly via the **REST API** (`fetch`) rather than the
  Firestore JS SDK — every request is a plain HTTPS call to
  `firestore.googleapis.com`, authenticated with the signed-in user's Firebase ID
  token. App state is stored as a single JSON blob (`appState/main`), with photos and
  photobooth sessions in their own collections/documents.
- **Offline fallback**: every write also goes to `localStorage`; if Firestore is
  unreachable or returns 401/403, the app silently falls back to local-only storage.
- **Image uploads**: photos are uploaded to Cloudinary via an unsigned upload preset
  (`CLOUDINARY_CLOUD_NAME` / `CLOUDINARY_UPLOAD_PRESET` near the top of the script),
  then the returned URL is what actually gets stored in Firestore.
- **Photobooth**: a turn-based, two-person photo session synced through Firestore
  polling (one document, `photobooth/currentSession`), with the strip composited
  client-side onto a `<canvas>`.

## App sections (tabs)

| Tab | Purpose |
|---|---|
| Home | Shared status ("home" / "out" / "at work" / custom), daily quote, days-together counter, photo feed |
| Notes | Freeform notes left for each other |
| Together | Shared task/checklist list with a spin-the-wheel reward system |
| Plans | Shift/date calendar |

## Local development

No install step — just serve the folder statically so the CDN scripts and relative
`icon-180.png` path resolve correctly:

```bash
python3 -m http.server 8000
# then open http://localhost:8000
```

(Opening `index.html` via `file://` mostly works too, but auth popups and some fetch
calls behave more reliably over `http://`.)

To point the app at your own Firebase/Cloudinary backend instead of the existing one,
edit the top of the inline script in `index.html`:

- `firebaseConfig` / `FIREBASE_PROJECT_ID` — your Firebase project
- `ALLOWED_EMAILS` — the two Google accounts allowed to sign in (client-side only; see
  Security note below)
- `CLOUDINARY_CLOUD_NAME` / `CLOUDINARY_UPLOAD_PRESET` — your Cloudinary account

You'll also need matching **Firestore Security Rules** on your Firebase project that
restrict `appState`, `photos`, and `photobooth` to the same two accounts — the
`ALLOWED_EMAILS` check in the app is just a friendly "not authorized" screen, not real
access control.

## Deployment (GitHub Pages)

This repo is deployed with GitHub Pages configured to build straight from the `main`
branch (no `.github/workflows` file, no Actions build step). That means:

- **Yes — pushing to `main` deploys automatically.** GitHub re-publishes the Pages
  site on every push to the branch it's configured to serve from; there's nothing
  extra to trigger.
- There's no build step to fail, since `index.html` is served as-is and Babel
  transpiles in the visitor's browser at load time.
- Deploys are usually live within a minute or two of pushing. If a change doesn't show
  up, check the **Pages** tab under repo Settings for the last build status, and hard
  refresh (CDN/browser caching is the usual culprit).

## Known limitations / things to be aware of

- **No LICENSE file** — the repo has no license, so by default all rights are
  reserved (others can view but not legally reuse the code).
- **This repo is public**, and both partners' personal Gmail addresses are hardcoded
  in plaintext in `index.html` (`ALLOWED_EMAILS`) — see security note below.
- **No bundler/minifier** — the whole app (currently ~130KB of markup/CSS/JS) is
  parsed and transpiled by Babel in every visitor's browser on every load. Fine for a
  two-person app; would need a real build step if this ever grows or needs faster
  load times.
- **No app manifest / Android or desktop icon** — `icon-180.png` only covers iOS
  "Add to Home Screen"; there's no `manifest.json` or favicon for other platforms.

## Security & privacy notes

A couple of things worth deliberately deciding on, not necessarily "bugs":

1. **Personal emails are public.** Since this repo is public, `ALLOWED_EMAILS` in
   `index.html` exposes both partners' real Gmail addresses to anyone who views the
   source on GitHub. If that's not desired, either make the repo private, or move the
   allow-list out of the client entirely (it's only used for a friendly message —
   Firestore Security Rules are what actually gate access).
2. **Firebase `apiKey` in the source is expected**, not a leak — Firebase web API keys
   identify the project, they don't grant access by themselves. Actual protection
   comes entirely from Firestore Security Rules, so it's worth double-checking (in the
   Firebase console, not in this repo) that those rules restrict `appState`, `photos`,
   and `photobooth` reads/writes to the two allowed accounts.
3. **The Cloudinary upload preset is unsigned by design** (required for a pure
   client-side upload). That also means anyone who reads the public source can extract
   `CLOUDINARY_CLOUD_NAME`/`CLOUDINARY_UPLOAD_PRESET` and upload arbitrary files to
   that Cloudinary account directly, outside the app. Worth checking the preset's
   settings in the Cloudinary dashboard for restrictions (allowed formats, max file
   size, folder scoping) if abuse/storage cost is a concern.

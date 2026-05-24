# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

A single-file web app (`index.html`, ~1470 lines) for managing portal credentials and software repository metadata. All logic, styles, and markup live in one file with no build step. Firebase Firestore is the primary datastore; localStorage is the offline fallback. Deployed on GitHub Pages at `https://capulongfarms.github.io/repository/`.

## Running the App

Open `index.html` directly in a browser — `file://` protocol works. No server required. Firebase sync starts automatically on load.

## Architecture

**Single global state object `S`** holds all runtime state: current tab, loaded records, auth session, crypto key, filter/sort state, etc.

**Four Firestore collections:**
- `portal_records` — credential entries (portalDescription, portalAddress, loginAccount, password, remarks). The `password` field is AES-GCM encrypted before every write.
- `repo_records` — repository metadata (17 fields: local paths, VS Code workspace, GitHub, Firebase project ID, Cloudflare, Railway, PayMongo, etc.). Each record also carries `createdAt` and `lastModified` timestamps (auto-set; not in `REPO_FIELDS`, excluded from export/import).
- `backups` — snapshot documents for disaster recovery
- `app_settings` — admin password hash (`passHash`) and app config

**Auth layer:** SHA-256 password hashing (via SubtleCrypto), 5-attempt lockout, session stored in memory only (`S.adminSession`).

**Encryption layer:** AES-256-GCM client-side encryption for portal passwords.
- Key derived from admin password via PBKDF2 (100,000 iterations, salt `repo-portal-v1`) on login — stored in `S.cryptoKey`, never persisted.
- Encrypted values stored as `enc:<ivBase64>:<ciphertextBase64>`.
- `encryptField(val)` / `decryptField(val)` handle encryption/decryption transparently.
- `loadPortal()` decrypts all passwords after fetching from Firestore — in-memory records are always plaintext.
- On sign-out, `S.cryptoKey` is cleared.

**PIN protection:** `pinProtect(cb)` always re-prompts for the admin password regardless of session state. Used for Backup, Restore, Export, Import, Clear All, and Clear Backups. `requireAdmin(cb)` only prompts when `S.adminSession` is false — used on Edit and Delete buttons for both Portal and Repository records. Add (Portal and Repo) requires no auth — no gate on button press or save. Auth for Edit is gated on the Edit button press only; `savePortalRecord`, `saveRepoEdit`, and `newRepoRecord` carry no session guards.

**Encryption migration:** `migrateEncryption()` re-encrypts all portal passwords when the admin password changes. Decrypts with the old key, re-encrypts with the new key, updates the password hash, and refreshes `S.cryptoKey` — all in one operation. Handles collections of any size via 500-doc batch chunks.

**UI structure:** Tab-based navigation — Overview (dashboard stats), Portal DB, Repository DB, Settings. Repository DB uses a split-panel layout (list on left, detail/edit on right). Modals handle add/edit for Portal DB. Settings tab order: Change Admin Password → Migrate Encryption Key → Portal Database → Repository Database → App Settings → App Information.

**Offline handling:** Firebase offline persistence is enabled. Falls back to localStorage cache when Firestore is unreachable. A status indicator shows connectivity state.

**Dependencies (all CDN, no npm):**
- Firebase SDK v10.12.2 (compat)
- XLSX.js v0.18.5 (Excel import/export)
- Google Fonts (IBM Plex Sans/Mono)

## Key Patterns

- UI interactions use direct `onclick` handlers and imperative DOM manipulation — no virtual DOM or reactive framework.
- Toast notifications for feedback; all async operations are `async/await`.
- Excel import maps column headers to field names; export serializes the active collection to `.xlsx`.
- Backup/restore snapshots the entire collection as a single Firestore document (`portal_backup`, `repo_backup`, `settings_backup` docs in the `backups` collection).
- `clearBackups()` deletes all three snapshot docs; PIN-protected.
- Batch deletes/updates are chunked at 500 docs to stay within Firestore limits.
- `lastModified` is stamped on every repo record save (`saveRepoEdit`) and create (`newRepoRecord`); displayed read-only at the bottom of the detail panel via `fmtDate()`. Not part of `REPO_FIELDS` so it is never exported or imported.
- There are no seed functions — `seedPortal()` and `seedRepo()` have been fully deleted.

## Firebase Config

Firebase project credentials are embedded directly in `index.html` (search for `firebaseConfig`). To switch Firebase projects, update that config object and ensure Firestore rules allow reads/writes.

**Firestore rules** are permanently open (`allow read, write: if true`) — security relies on the admin password gate and client-side encryption rather than Firestore-level auth.

## Deployment

- GitHub repository: `capulongfarms/repository`
- GitHub Pages URL: `https://capulongfarms.github.io/repository/`
- Push to `main` branch to deploy — GitHub Pages rebuilds automatically (~1–2 min).
- Local git remote: `https://github.com/capulongfarms/repository.git`

## Password Change Sequence

When changing the admin password, use one of these two approaches:

**Using the Migration Tool (Settings → Migrate Encryption Key):**
1. Export — save a local Excel backup as a safety net
2. Migrate Encryption Key — re-encrypts all records and changes the password in one step

**Manual approach (if migration tool unavailable):**
1. Export — save local Excel backup
2. Clear All — wipe Firebase
3. Change Admin Password — update the hash
4. Re-import — data comes back encrypted with the new key

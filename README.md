# Repository Portal

A single-file web app for managing portal credentials and software repository metadata. Built for personal use — all data stays in Firebase Firestore with client-side AES-256 encryption.

**Live app:** https://capulongfarms.github.io/repository/

---

## Features

- **Portal & Email Database** — store and manage credentials (site, login, password, remarks)
- **Repository Database** — track 17 metadata fields per project (GitHub, Firebase, Cloudflare, Railway, etc.)
- **AES-256-GCM encryption** — passwords are encrypted in the browser before reaching Firebase; plaintext never stored in the cloud
- **Admin password gate** — SHA-256 hashed, 5-attempt lockout, session in memory only
- **PIN re-prompt** — Backup, Restore, Export, Import, and Clear All always ask for the password even within an active session
- **Excel import / export** — bulk data in and out via `.xlsx`
- **Firebase backup / restore** — snapshot entire collections to Firestore
- **Offline support** — localStorage fallback when Firebase is unreachable
- **Encryption migration** — change admin password and re-encrypt all records in one step

---

## Security Model

| Layer | What it does |
|---|---|
| Login screen | SHA-256 hashed password, 5-attempt lockout |
| AES-256-GCM encryption | Portal passwords encrypted before every Firestore write |
| PBKDF2 key derivation | Encryption key derived from admin password (100,000 iterations); never stored |
| PIN re-prompt | Sensitive bulk operations require password re-entry mid-session |
| Key wiped on sign-out | `S.cryptoKey` cleared from memory on sign-out |

---

## Changing Your Admin Password

> **Important:** Changing the admin password changes the encryption key. Follow one of these sequences to avoid losing access to your data.

### Using the Migration Tool (recommended)
1. **Export** — save a local Excel backup as a safety net
2. **Settings → Migrate Encryption Key** — re-encrypts all records and changes the password in one step

### Manual approach
1. **Export** — save local Excel backup
2. **Clear All** — wipe Firebase
3. **Change Admin Password** — update the hash
4. **Re-import** — data comes back encrypted with the new key

---

## Tech Stack

- No build step — open `index.html` directly in a browser (`file://` works)
- Firebase Firestore v10.12.2 (CDN, compat mode)
- XLSX.js v0.18.5 for Excel import/export
- Web Crypto API (SubtleCrypto) for SHA-256 and AES-GCM
- Google Fonts — IBM Plex Sans / Mono

---

## Firebase Project

`repository-15ce3` — owned by `polcapulong@yahoo.com`

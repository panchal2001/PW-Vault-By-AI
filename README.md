# PW-Vault-By-AI
Offline password manager in a single HTML file. AES-256-GCM + PBKDF2. Runs from a USB drive. No server, no cloud, no install.

# Vault — Offline Password Manager

> A single-file, fully offline, encrypted password manager that runs in any browser directly from a USB pendrive. No server. No cloud. No installation. Your `.vault` file is the only storage.

---

## Table of contents

- [Overview](#overview)
- [How to use](#how-to-use)
- [Security model](#security-model)
- [Features](#features)
- [Architecture](#architecture)
- [File format](#file-format)
- [Browser compatibility](#browser-compatibility)
- [Vulnerability fixes applied](#vulnerability-fixes-applied)
- [Known limitations](#known-limitations)
- [Credits](#credits)

---

## Overview

Vault is a **single HTML file** (`vault.html`) you carry on a USB pendrive. Open it in any browser, pick your `.vault` backup file, enter your master password, and your passwords are available — with zero data sent anywhere.

Everything is encrypted with **AES-256-GCM** and a key derived via **PBKDF2 (400,000 iterations, SHA-256)**. The `.vault` file is the sole source of truth. Nothing is stored in `localStorage` except non-sensitive state (brute-force counters, UI snooze preferences).

```
vault.html          ← the entire app — one file, ~150KB
vault.vault         ← your encrypted data — keep this safe
```

---

## How to use

### First time — create a new vault

1. Open `vault.html` in Chrome, Firefox, or Edge
2. Click **New Vault** tab
3. Choose a strong master password (minimum Fair strength)
4. Save the generated `.vault` file to your pendrive

### Every subsequent use

1. Open `vault.html`
2. Click **Browse** and select your `.vault` file
3. Enter your master password → **Unlock Vault**
4. Use the vault normally — changes save back to the file automatically

### Incognito / private mode (recommended on shared devices)

Same flow — pick your `.vault` file, enter password, unlock. When you close the incognito window, no data is left on the machine. The app detects incognito automatically and warns you to export before closing.

### Chrome / Edge — auto-save mode

On Chrome and Edge, use the **"Pick file & auto-save back"** button. This uses the File System Access API to grant write permission once — every save (add, edit, delete, password change) writes silently back to the same file. No download prompts.

### Firefox / Safari — download mode

Every save triggers a download of the updated `.vault` file. Replace the old file on your pendrive with the new download.

---

## Security model

| Property | Implementation |
|----------|---------------|
| Encryption | AES-256-GCM |
| Key derivation | PBKDF2, 400,000 iterations, SHA-256 |
| Integrity check | HMAC-SHA-256 on ciphertext (separate key derivation) |
| Master password | Held only in JS closure — never written to disk or DOM |
| Network | Zero outgoing connections — CSP blocks all |
| Storage | `.vault` file only — no localStorage for vault data |
| Auto-lock | 5 minutes idle + immediate lock on window blur (3s grace) |
| Clipboard | Cleared automatically after 30 seconds |
| Brute force | 5 attempts → 30s lockout, exponential backoff, survives page reload |
| Screen capture | Optional CSS-based protection blocks most screenshot tools |
| Keylogger defence | On-screen randomised keyboard, shuffled after every keypress |

### What the `.vault` file contains

The encrypted payload is a JSON envelope:

```json
{
  "entries": [ ...your password entries... ],
  "meta": {
    "pwChangedAt": "2025-03-23T09:41:00.000Z",
    "createdAt":   "2025-01-01T00:00:00.000Z"
  }
}
```

All metadata — including when you last changed your master password — travels inside the encrypted file. Opening the vault on any device instantly shows the correct password age without relying on any local state.

### What is never stored

- Your master password (wiped from memory on lock)
- Plaintext passwords (only in JS memory while unlocked)
- Any data in `localStorage` beyond brute-force counters and UI snooze dates
- Anything on a server (there is no server)

---

## Features

### Core

- **Unlimited entries** — name, username, password, URL, notes, expiry date
- **Password generator** — configurable length (8–64), uppercase, lowercase, numbers, symbols
- **Password strength meter** — live scoring on every keystroke
- **Search** — instant filter across name, username, URL, notes
- **Filter tabs** — All, Weak, Recent, Health dashboard

### Health dashboard

A dedicated tab showing:
- **Overall health score** (0–100) weighted by weak, reused, and outdated passwords
- **Weak passwords** — strength score ≤ Fair
- **Reused passwords** — same password used on two or more accounts
- **Expired passwords** — entries past their set expiry date
- **Outdated passwords** — not updated in 30+ days
- Each issue has a **Fix** button that opens the edit modal directly

### Password expiry

Set a per-entry expiry date with quick **+30d** / **+90d** buttons. Expiry status appears as a coloured pill on every card (green / amber / red). Expired entries surface in the Health dashboard.

### Backup and restore

- **Export** — encrypted `.vault` download, password re-confirmation required
- **Import** — pick a `.vault` file + enter the password it was encrypted with
- **Merge** — import without overwriting existing entries (skips duplicates)
- **QR code backup** — generates scannable QR of the encrypted vault for phone transfer
- **Email reminder** — opens mail client with pre-filled reminder to attach the file
- **Auto-export reminder** — status bar turns amber after 7 days, red after 30

### Master password management

- **Change password** — re-encrypts entire vault under new key, updates `pwChangedAt` in meta
- **Password age tracking** — shows days since last change, warns at 7 days, critical at 30
- **Reminder banner** — dismissible, snooze for 3 days
- Age data stored inside encrypted vault — survives incognito and cross-device restore

### Security features

- **Decoy vault** — set a second password that shows convincing fake entries. Real entries untouched.
- **On-screen keyboard** — randomised layout, shuffled after every keypress, blocks keyloggers
- **Screen capture protection** — CSS-based, disables text selection and print-to-PDF
- **Brute-force lockout** — persistent across page reloads via localStorage
- **Window blur lock** — vault locks 3 seconds after you switch away from the browser
- **HMAC integrity** — detects file corruption before decryption

### Incognito detection

Detects private/incognito mode via FileSystem API quota, localStorage quota errors, and IndexedDB availability. When detected:
- Blue banner with step-by-step safe workflow
- Screen capture protection enabled automatically
- Warning toast on close if vault has unsaved entries

### Reset App

A **Reset** button clears all localStorage data (lockout state, decoy vault, snooze timers) without touching the `.vault` file. Requires checkbox confirmation. Useful when handing a device back after use.

---

## Architecture

```
vault.html (single file)
│
├── CSS — design tokens, layout, components, virtual keyboard
│
├── Crypto module
│   ├── deriveKey()       — PBKDF2 → AES-256-GCM key
│   ├── deriveHmacKey()   — PBKDF2 → HMAC-SHA-256 key
│   ├── encrypt()         — AES-GCM + HMAC sign
│   └── decrypt()         — HMAC verify + AES-GCM decrypt
│
├── Store module (file-based, no localStorage)
│   ├── unlockFromHandle() — File System Access API (Chrome/Edge)
│   ├── unlockFromFile()   — FileReader fallback (Firefox/Safari)
│   ├── createNew()        — new vault with FS picker or download
│   ├── _save()            — write-back to file handle or download
│   └── CRUD + changePw + importData + mergeData
│
├── BruteGuard            — attempt counter, lockout timer (localStorage)
├── PasswordAge           — reads pwChangedAt from Store meta
├── BackupManager         — last export date tracking
├── HealthAnalyser        — weak/reused/expired/old analysis
├── DecoyVault            — separate encrypted decoy (localStorage)
├── ScreenCapture         — CSS protection toggle
├── VirtualKeyboard       — randomised on-screen keyboard
├── IncognitoDetector     — three-method detection
│
└── Render / UI
    ├── render()          — entry cards, filter, search
    ├── renderHealth()    — health dashboard
    └── Modals            — entry, import, export, QR, decoy, reset, change pw
```

---

## File format

The `.vault` file is a Base64-encoded JSON envelope:

```
base64( JSON({ s, i, c, h }) )
```

| Field | Content |
|-------|---------|
| `s` | 16-byte random salt (Base64) |
| `i` | 12-byte random IV (Base64) |
| `c` | AES-256-GCM ciphertext (Base64) |
| `h` | HMAC-SHA-256 signature of `c` (Base64) |

The plaintext (before encryption) is:

```json
{
  "entries": [
    {
      "id":          "uuid-v4",
      "name":        "GitHub",
      "username":    "you@example.com",
      "password":    "correct-horse-battery-staple",
      "url":         "https://github.com",
      "notes":       "",
      "expiryDate":  "2025-06-23",
      "createdAt":   "2025-01-01T00:00:00.000Z",
      "updatedAt":   "2025-03-23T09:41:00.000Z"
    }
  ],
  "meta": {
    "pwChangedAt": "2025-03-23T09:41:00.000Z",
    "createdAt":   "2025-01-01T00:00:00.000Z"
  }
}
```

---

## Browser compatibility

| Browser | Open vault | Auto-save to file | QR code |
|---------|-----------|-------------------|---------|
| Chrome 86+ | ✅ | ✅ File System Access API | ✅ |
| Edge 86+ | ✅ | ✅ File System Access API | ✅ |
| Firefox | ✅ | ⬇️ Download on each save | ✅ |
| Safari 15+ | ✅ | ⬇️ Download on each save | ✅ |

The app is fully functional on all browsers. Chrome/Edge offer the best experience because the File System Access API allows silent auto-save back to the same file.

---

## Vulnerability fixes applied

The following security issues were identified during development and resolved:

| # | Issue | Fix |
|---|-------|-----|
| 1 | Firebase removed | 100% offline, no network calls |
| 2 | Raw password as AES key | PBKDF2 (400k iterations) key derivation |
| 3 | Vault path reversible | SHA-256 one-way hash (now unused — file-based) |
| 4 | Password in DOM attributes | Passwords held in JS closure only |
| 5 | mousemove floods lock timer | Throttled to one reset per 10 seconds |
| 6 | Predictable localStorage key | Removed — vault not in localStorage |
| 7 | Clipboard not cleared | Always cleared after 30s unconditionally |
| 8 | No CSP | Content-Security-Policy meta tag blocks all external scripts |
| 9 | Import blindly trusts JSON | Schema validation: types, lengths, URL sanitisation |
| 10 | javascript: URLs in entries | Only https/http/ftp schemes accepted |
| 11 | Export without confirmation | Password re-entry required before download |
| 12 | No external integrity check | HMAC-SHA-256 on ciphertext |
| 13 | Brute-force resets on reload | Attempt counter persisted in localStorage |
| 14 | Clipboard readText permission | Removed — always clear, no read required |
| 15 | Window blur stays unlocked | Locks after 3s on window/tab switch |
| 16 | Weak password accepted at creation | Minimum score 3 (Fair) enforced everywhere |
| 17 | innerHTML for user data | All user-controlled data uses textContent |
| 18 | localStorage as vault storage | Removed entirely — .vault file is sole storage |

---

## Known limitations

- **No mobile browser file picker write-back** — mobile browsers do not support the File System Access API, so every save triggers a download
- **On-screen keyboard cannot stop hardware keyloggers** — a USB keylogger between keyboard and machine intercepts before any software defence
- **Screen capture protection is CSS-based** — a phone camera pointed at the screen bypasses it
- **QR code for large vaults** — very large vaults (500+ entries) may produce multiple QR codes that require sequential scanning
- **Decoy vault cannot prove it is real** — a determined attacker who knows about the feature may notice fake entries

---

## Credits

**Designed, architected, and built** in collaboration with [Claude](https://claude.ai) by Anthropic — an AI assistant used for the full development lifecycle: initial code, security analysis, iterative vulnerability fixes, feature additions, and this documentation.

All 18 security vulnerabilities were identified and fixed through structured code review sessions. Features added across the development conversation include:

- Core encryption (AES-256-GCM + PBKDF2)
- Virtual on-screen keyboard with per-keypress shuffle
- Decoy vault with separate encryption
- Password health dashboard (weak, reused, expired, outdated)
- Per-entry expiry dates
- HMAC integrity verification
- File System Access API integration for silent auto-save
- QR code backup
- Incognito detection with step-by-step workflow
- Master password age tracking inside encrypted vault meta
- Window blur auto-lock
- Reset App with confirmation guard

> Built with Claude Sonnet 4.6 — [https://claude.ai](https://claude.ai)

---

## Licence

MIT — do whatever you want with it. If you improve it, consider opening a pull request.

```
THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND.
```

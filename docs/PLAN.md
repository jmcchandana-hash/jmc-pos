# JMC STUDIO POS: audit, target architecture and plan

Status: written without Firebase project access. Nothing in `firebase/` is deployed. The live app still uses the existing offline file/RTDB sync described below.

## 1. Audit of the current app (v4)

**Storage.** localStorage keys = the keys of `DEFAULTS()`: `companySettings, products, categories, sales, invoices, invoiceCounter, reports, theme, cart, orders, orderCounter, deleted, device` (plus `lang`, `theme`, `cloudPhotos`). Photos live in IndexedDB. localStorage is per browser, about 5 MB, and is lost if the user clears site data. It is not a backup.

**Models.** Product (id, sku, name, category, price, cost, stock settings, image, updatedAt), sale (id, invoice no, items, payments[], customer, totals, createdAt), order (id, no, title, customer, due date, status, payments, photo ids, remindedFor), deleted tombstones `{products, sales, orders}`.

**Stock** is derived from the sales ledger, not stored per device, so two devices cannot overwrite each other's stock number.

**Invoice numbers** use prefix + optional device code + counter. A duplicate across devices is renamed with a suffix at merge time. Sale IDs are random, so a number clash never merges two sales.

**Sync today (three modes).** (1) sync file, (2) shared folder, (3) Firebase Realtime Database at `<url>/shops/<shop code>/devices/<deviceId>`. Each device uploads its full snapshot, other devices fetch and merge per record (newest `updatedAt` wins, tombstones for deletes, payments merged as a union by id).

**Backups.** Manual email backup, and automatic email via the owner's Google Apps Script. Both depend on the app being opened (or the script's own trigger). No retention policy, checksums, verification or restore preview besides "Restore backup".

**Authentication.** None. The "shop code" and the link code (`JMC1-` + Base64 of URL, code and secret) are the only gate.

## 2. Weaknesses (by severity)
1. **Shop code is the only access control for cloud sync.** Anyone who has it, or guesses the database URL pattern with open rules, can read or change every record. Link codes are plain Base64 and may contain a database secret.
2. **Open database rules are required for it to work** unless the legacy database secret is embedded in the link code.
3. **Edits to the same record**: newest timestamp wins silently. Device clocks that are wrong can overwrite newer data. There is no conflict screen.
4. **Whole snapshot upload per device**: large data means slow sync, and deletes depend on tombstones.
5. **Photos stored as Base64 data URLs** in the database.
6. **No real backup**: email depends on the user; no retention, no integrity check.
7. **The BotFather token, CallMeBot key and database secret live in browser storage** of each device.
8. **No roles**: any device can edit or delete anything.

## 3. Target architecture (needs a Firebase project)
Firebase Auth (email/password) + Cloud Firestore (offline persistence) + Cloud Storage for photos + Cloud Functions (scheduled backups, invitations, invoice number allocation, owner alerts) + App Check. Hosting stays on GitHub Pages. The frontend must move to a bundled project (Firebase SDK). Do it on a feature branch and keep the single-file app working until the new one is verified.

**Collections** (under `shops/{shopId}`): members/{uid}, devices/{deviceId}, products, sales, payments, orders, customers, stockMovements, settings, auditLogs, backups, syncOperations.

**Every record**: id, shopId, createdAt, updatedAt, createdBy, updatedBy, deviceId, version, deletedAt.

**Financial rules.** Sales, payments and stock movements are append-only (see `firebase/firestore.rules`). Corrections are refund/adjustment documents. Stock = sum of movements. Operations carry an `operationId` so retries are idempotent (document ID = operationId).

**Invoice numbers.** Immutable `saleId` plus a human number from a per-device sequence (`INV-<deviceCode>-000123`) so offline devices never clash; an optional Cloud Function can allocate a global sequence when online.

**Sync operation record**: operationId, userId, deviceId, shopId, entityType, entityId, operationType, payload, baseVersion, clientCreatedAt, serverCreatedAt, retryCount, status. Order edits compare `baseVersion`; on mismatch both versions are kept and shown on a conflict screen, and the conflict is written to auditLogs.

**Backups.** Daily scheduled function exports each shop to Cloud Storage as `backups/{shopId}/{date}.json.gz`, stores sha256 + counts + schemaVersion, re-downloads to verify, retries on failure, emails the owner on failure, keeps 30 daily + 12 monthly copies. Restore: load into a staging collection, show added/updated/skipped/conflicts, owner confirms, pre-restore backup, rollback available.

**Migration from localStorage.** Emergency export first, validate and normalise, generate missing IDs, detect duplicates, preview, confirm, write in resumable batches, keep original localStorage until success, produce a report, rollback by deleting the batch by `migrationId`.

## 4. What is implemented now
- Honest sync status for the existing Online sync: Synced only when nothing is waiting; Syncing, Waiting to upload (with approximate count), Offline, Failed with Retry, last successful and last failed time. Fixed a race where changes made during an upload could be marked as synced.
- Reminder channels: WhatsApp and Telegram (separate feature).
- Draft rules and config files: `firebase/database.rules.json`, `firebase/firestore.rules`, `.gitignore`, `.env.example`.

## 4b. Security hardening done in the app (v12)
- **Content Security Policy** (meta tag): no external scripts, connections only to Telegram, CallMeBot, Google Apps Script and Firebase; no plugins, no base tag changes, no form posts.
- **Untrusted data is cleaned** before use (sync files, cloud, backups, imports): prototype-pollution keys removed, records with unsafe ids dropped, images must be real image data URLs, oversized strings cut.
- **Backups and exports no longer contain secrets**: this device's connection settings (sync secret, script token, PIN) and the bot/WhatsApp keys are left out; restoring a backup keeps the device's own values.
- **Optional PIN lock** per device: PBKDF2-SHA256 (150,000 rounds, random salt), 4 to 12 digits, weak PINs refused, 30 s lockout doubling up to 15 min after 5 wrong tries, auto-lock after inactivity, app inert while locked. It is a screen lock, not disk encryption.
- Referrer is not sent; external links open with `noopener`; debug handle `window.__pos` only when `localStorage.debug = 1`.
- Still true: Online sync has no accounts. Anyone with the link code can read the shop data. The Telegram token and WhatsApp key are in the shared sync data by design (set once, every device gets it). Real fixes (accounts, per-user rules) need the Firebase project in section 3.

## 5. What is NOT implemented (and what it needs)
Everything in section 3. Needs from the owner: a Firebase project (Blaze plan for Functions and scheduled backups), Auth enabled, a Storage bucket, a deploy token or service account stored as a GitHub Actions secret, and an agreement to move to a bundled frontend.

## 6. Rollback and disaster recovery
- Code: every release is a commit on `main`. Roll back with `git revert <commit>` and push; bump `VERSION` in `sw.js` so phones update.
- Data: before any migration take Data & Sync > Export a full backup and keep the file outside the browser.
- If a device loses data: reinstall, Data & Sync > Restore backup, or reconnect Online sync and let it merge from another device.

## 7. Interim safety advice for the current cloud sync
Use a long random shop code (the app generates one), do not share the link code outside staff, and set the Realtime Database rules so only the shop path is readable with the secret. Treat the link code as a password.

## 8. Test checklist (status)
| Test | Status |
|---|---|
| Online/offline sale and product, stock from ledger | Covered earlier by Playwright runs on the app (offline file) |
| Two simulated devices merge, duplicate invoice rename | Tested earlier with two simulated devices |
| Sync status transitions (synced / failed / retry) | Tested now against a mock database |
| Telegram and WhatsApp reminders, Google script | Tested against mocks, not live services |
| Mobile layout at phone width, English and Sinhala | Checked now |
| Security rules, cross-shop, revoked users, backup/restore, migration, deploy | Not tested: not built, no Firebase project |

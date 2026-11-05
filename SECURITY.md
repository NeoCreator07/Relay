# Security Policy — Relay

This document describes the security posture of the Relay chat app (single-file HTML client on Firebase), what is protected today, what is *not*, and how to harden it further. It is written to be honest about the trust model rather than to overstate the protections.

| | |
|---|---|
| **Current release** | `relay-v147.html` |
| **Firebase project** | `relay-87573` |
| **Repository** | [github.com/NeoCreator07/Relay-Messaging](https://github.com/NeoCreator07/Relay-Messaging) |
| **Owner** | NeoCreator07 (TerminalVelocity) |
| **Policy last updated** | 2026-11-05 |

---

## Supported versions

| Version | Status |
|---|---|
| `relay-v147.html` | ✅ **Current release — supported** |
| `relay-v146.html` | ⚠️ Legacy, kept for rollback only |
| `relay-v145.html` | ⚠️ Legacy, kept for rollback only |
| `relay-v144.html` | ⚠️ Legacy, kept for rollback only |
| `relay-v143.html` | ⚠️ Legacy, kept for rollback only |
| Older "old relay" builds | ❌ Do not use — contained a dead Firebase project config, no auth sign-in before DB calls, two conflicting AES keys, and a hardcoded master admin password, all since removed |

## Reporting a vulnerability

**Please do not open a public issue for security problems.** Relay is developed openly on GitHub, and public reports give every attacker the same head start you had.

**How to report privately:**

1. **Preferred:** use GitHub's private vulnerability reporting on the [Relay-Messaging repository](https://github.com/NeoCreator07/Relay-Messaging) — *Security → Report a vulnerability*. This keeps the report, discussion and fix coordination private to the maintainer.
2. **Fallback:** if private reporting is not enabled on the repository, contact **NeoCreator07** directly via [their GitHub profile](https://github.com/NeoCreator07). Do not post details in public issues, discussions, or pull requests.

**Include in your report:**

- The exact file and version you tested (e.g. `relay-v145.html`) — the app is a single file, so version is everything.
- Step-by-step reproduction (console snippets, rule expressions, or a minimal PoC HTML file are ideal).
- The impact in concrete terms (what an attacker gains: account access, data exposure, bill run-up, XSS execution).
- Whether the issue is in the app client, the delivered Firebase rules, or the interaction between them.

**Disclosure:** please allow **14–90 days** for a fix before public disclosure. Coordinated disclosure is happy to credit reporters by name (or anonymously, your choice) in release notes.

**In scope / out of scope:**

- ✅ In scope: XSS or injection in the app client, bypass or weakness of the delivered Firebase rules, unintended data exposure, account-takeover paths, crypto flaws beyond the documented shared-key design.
- ❌ Out of scope: the shared-key message "encryption" (a documented design decision, see below), volumetric DoS / quota exhaustion, self-XSS you can only trigger on your own browser, social engineering, and anything requiring the Firebase project owner's console credentials.

---

## Trust model — read this first

Relay is a **client-only application**: there is no server-side code of our own, no API layer and no session backend. The browser talks **directly** to Firebase using an embedded config, and **Firebase security rules are the only real enforcement boundary**. Keep three consequences in mind:

1. **Everything in the HTML file is public.** The Firebase project id, the app's AES key constant, the hashing scheme, the Giphy API key fallback, and every rule of logic can be read by anyone who opens the file. Client-side checks are UX, not security.
2. **Every visitor signs in anonymously.** Under the database rules, `auth != null` therefore effectively means *the whole internet*. Anyone who loads the page (or writes their own Firebase client with the same config) satisfies the read/write conditions of the current rules.
3. **The rules decide everything.** A malicious client can read and write any node the rules allow — the app's polite behavior (validation, cooldowns, uniqueness checks) does not bind it.

Within that model the app is a **walled garden, not a fortress**: it is well-suited to a semi-open community chat, and it is *not* suited to storing secrets that must be kept from other users of the same system.

### How Relay is tested — and why the test harness can write to the database

Every release is verified against the **live** database before delivery: an automated harness registers throwaway accounts (`zz…` usernames), drives the real UI (register → login → clicks), exercises the feature under test, and then **deletes exactly the records it created** — the test accounts, their presence, friend/request nodes, servers and test DM channels. Watching test data appear and vanish is *not* evidence of secret admin access: the harness signs in anonymously like any other visitor and holds exactly the read/write power the rules grant *every* visitor (see the trust model above). It cannot change the security rules, touch the Firebase console, or delete the project itself — only the data nodes the rules leave open.

If that power concerns you, it should — the attack scenarios below describe what any anonymous visitor can do with the same access. The harness scripts are kept alongside the project so every write and the exact cleanup list are auditable line by line.

## Attack surface inventory

Every service the shipped file talks to, and why:

| Surface | Used for | Trust note |
|---|---|---|
| Firebase Auth (anonymous) | Sign-in before any DB call | Authenticated ≠ trusted — anyone can be anonymous |
| Firebase Realtime Database | Users, servers, messages, DMs, friends, presence, config | Governed by `firebase-database-rules.json` |
| Firebase Cloud Storage *(optional — only for attachments > 4 MB)* | Large file attachments | Governed by `firebase-storage-rules.rules`; the app degrades gracefully — smaller files route through the Realtime Database instead |
| Giphy API (`api.giphy.com`) | GIF picker | Key shipped in the file (see *Attack scenarios* #6) |
| noembed.com (JSONP) | Link embeds | Script injection by design — fixed reputable endpoint |
| microlink.io | Embed fallback when noembed fails | Best-effort, cached per URL |
| `youtube-nocookie.com` iframes | Inline video playback | Google's privacy-hardened embed domain |
| Google Fonts, three.js r121 CDN, p5.js 1.4.2 CDN | Styling + Vanta backgrounds | Lazy-loaded only when a background is chosen |
| `localStorage` | Session token, theme, colors, drafts | Same-origin only; readable by any script in the page |

## Controls in place

### Authentication & boot

- **Auth-first boot:** `signInAnonymously()` runs before any database call. This fixed the old fatal hang (dead project + rules requiring `auth != null`) and guarantees no DB traffic precedes authentication.
- **Account passwords** are hashed client-side with **SHA-256 + a per-user random salt** before ever touching the database. Plaintext passwords are never stored or transmitted. (But see *Attack scenarios* #3 — the hash is fast, not key-stretched.)
- **Auto-login tokens:** login stores a `sessToken` in `users/{username}` and in `localStorage` (`relay_session`). It is a bearer token — see *Attack scenarios* #2.

### Realtime Database rules (`firebase-database-rules.json`)

- **Default deny:** top-level `.read` / `.write` are `false`; every node must be explicitly allowed.
- **Every node requires `auth != null`** (`users`, `servers`, `serverInvites`, `messages`, `presence`, `typing`, `friends`, `friendRequests`).
- **`config` is read-only to clients** (`.write: false` at the node and at `config/banned` / `config/giphyKey`) — admin lists, the ban list and the Giphy key cannot be tampered with by app users.
- **Per-node `.validate` schema (v148, hardened rules):** every writable field is now type- and size-checked server-side — message shape (`u` ≤ 40 chars, numeric `t` in a 2020–2100 window, `type` allow-list, `text` ≤ 4 000 chars), `url` must be an `https://`/`http://`/`data:` string of ≤ 5.7 M chars (bounds the base64 attachment route at ~4.3 MB — just above the client's own 4 MB cap), avatars ≤ 400 k chars, `fileSize` ≤ 25 MB, boolean-only flags (`muted`, `online`, `enc`, `edited`), `true`-only map leaves (`members`, `ips`, reactions), hex-64 `pw`/`codeHash`, hex `color`, timestamp windows on `createdAt`/`lastLogin`/`lastSeen`/`ts`, and friend/request edges must be timestamps. A machine cross-check (scripts/v148_rules_check.js) verified every read path and all 40 write call-sites in `relay-v148.html` against these rules with the app's real payloads, plus 12 adversarial payloads that must be rejected. The pre-hardening rules are kept as `firebase-database-rules-v1-permissive.json` for rollback.

### Cloud Storage rules (`firebase-storage-rules.rules`)

- Reads and writes require `auth != null`.
- **Uploads capped at 25 MB** (`request.resource.size < 25 * 1024 * 1024`) — the previous 1 GB allowance was a bill-run-up vector for anonymous visitors.
- **Content-type allow-list:** only images, video, audio, text, PDF, JSON, and common archive/office/Apple document types are accepted.
- The app mirrors the cap client-side (`MAX_FILE = 25 MB`), cancels stalled uploads after ~35 s (no Storage access no longer hangs the send button), and routes files ≤ 4 MB through the Realtime Database as base64 instead — so the app works fully on the free Spark plan.

### XSS surface

- All user-controlled strings interpolated into HTML go through `esc()` (escapes `& < > "`), including room names, message text fragments and embed URLs.
- Message links are rendered with `textContent` (not HTML) and trailing punctuation is stripped; mentions/bold formatting run *after* URL detection so they cannot corrupt URLs.
- YouTube embed ids pass through `encodeURIComponent`; embed iframes use `youtube-nocookie.com`.
- GIFs are Giphy **URLs**, not uploaded media; avatars are compressed base64 images in the Realtime Database (not Storage), sized client-side before upload.

### Message "encryption" — what it actually is

Messages are encrypted with **AES-256-GCM** (PBKDF2, 100k iterations). However, the key material is a **constant embedded in the single-file source** (`relay-e2e-k3y-…`) that every user can read, and the same key is shared by all clients. This is **obfuscation against casual database snooping (e.g. a leaked DB export), not end-to-end encryption** — a determined user of the app can decrypt anything. Do not treat the DB as unreadable to insiders. True E2EE would require per-conversation key exchange.

### Moderation & abuse guards (client-side)

- 2-second send/edit cooldowns; 30-second repeated-message window.
- Server owners can mute their servers; hidden/private servers require one-time invites.
- `config/banned` is readable by the client so a ban list can be enforced in the UI (and the rules keep it tamper-proof).
- Server deletion is guarded against resurrecting empty nodes from still-open tabs.

### Identity uniqueness (v145)

- Usernames, display names (case- and whitespace-insensitive) and name colors are unique, checked against a **fresh live snapshot** of `users` at register/edit time, with live availability hints and last-guard revert-on-conflict in the profile editor.
- This is enforced **client-side only** — see *Attack scenarios* #5 for the race window and bypass.

---

## Attack scenarios the current rules permit

These are verified against the shipped clients (`relay-v145.html` through `relay-v148.html`) and the delivered rules files. They are **accepted risks of the current design**, not unknown bugs — documented so nobody mistakes politeness of the client for a guarantee. All of them require nothing more than anonymous auth, i.e. *any visitor*. (The v148 hardened rules bound the *size and shape* of what can be written — scenarios 1–6 below still need identity binding, which is roadmap item 3.)

1. **Full account takeover by overwrite.** `users/.write = auth != null` means any visitor can simply overwrite `users/{victim}` — setting their own `pw` + `salt` (then logging in with a chosen password), or replacing the `sessToken`. The client only ever writes the logged-in user's own record, but the rules do not enforce that.
2. **Account takeover via `sessToken` theft.** `users/.read = auth != null` exposes every account's `sessToken`; auto-login only compares the `localStorage` token with the DB value. An attacker who reads a victim's token and sets `relay_session = {u: victim, t: token}` in their own browser is logged in as the victim — no password needed.
3. **Offline password cracking.** The stored `pw` hash is a *single round* of SHA-256 (the PBKDF2 stretching is only used for the AES key). Any visitor can download all password hashes and brute-force weak passwords offline at GPU speed. Strong passwords remain safe; weak ones do not.
4. **Message forgery and destructive writes.** `messages/.write` and `servers/.write` are `auth != null`, so a custom client can post messages under any username (`u` is client-chosen), edit or delete *anyone's* messages, and mute or delete *any* server. The in-app edit/delete guards apply only to the real client.
5. **Uniqueness race + bypass.** Two clients registering the same name in the same moment can both pass the live-check; and a custom client skips the checks entirely. Uniqueness is UX-level, not enforced by the database.
6. **API-key exposure.** The Giphy key is readable two ways: hardcoded in the file (as a fallback) and via `config/giphyKey` (readable by any authenticated visitor). A third party could extract and use it for their own requests against Giphy's rate limits.
7. **DB-routed attachment quota abuse (v146, bounded in v148).** Files ≤ 4 MB travel as base64 data URLs inside `messages`. The v148 hardened rules now cap `url` at 5.7 M characters server-side and validate message shape, so a custom client can no longer push arbitrarily large strings or garbage shapes per message — the worst case per message is bounded at ~4.3 MB. What remains: **volume** abuse (many *valid* ~4 MB messages would still exhaust the Spark quotas, 1 GB stored / ~10 GB downloaded per month, and slow listeners in the affected room) — there is no server-side rate limiting; see the roadmap (App Check, item 5).

## Privacy considerations

- **IP addresses are stored and exposed.** `users/{username}` keeps `ip` / `ips` and `presence/{username}` broadcasts `ip` (plus `lastSeen`, `name`, `color`, `pfp`, `status`) — and both nodes are readable by every authenticated visitor. Every user can see every other user's IP.
- **Message content is decryptable by insiders** (shared AES key, see above).
- **Avatars** are compressed base64 images stored in the Realtime Database (readable by all).
- **Presence tracking:** `lastSeen` updates every 20 seconds and survives as a timestamp after disconnect — a detailed activity log of when any user was online.
- **`localStorage` on shared machines:** the session token, message drafts, and avatar persist after logout only partially (logout removes `relay_session`; other keys remain) — use the browser's site-data clearing if you hand over a machine.

## Known limitations (summary)

1. `auth != null` = the whole internet — see the seven concrete scenarios above.
2. Client-side guards (cooldowns, spam windows, mute UX, validation) are bypassable.
3. `sessToken` is a non-expiring bearer credential with no revocation flow.
4. Link previews use JSONP and third-party iframes (noembed, YouTube) — convenience, not isolation.
5. No App Check and no server-side rate limiting — per-field `.validate` rules on message shape/size **shipped in v148** (see *Controls in place*), which bounds scenario #7 per-message but not per-volume.
6. Anonymous Firebase accounts are free and unlimited, so the auth boundary itself cannot slow down a determined abuser.
7. Two benign debug hooks remain in the shipped file: `__NOAUTO` (skips auto-opening the first server — test aid) and `__lastVantaCfg` (exposes the last Vanta config object). Neither writes to Firebase nor grants privileges.

## Billing-abuse vector (if Storage is enabled)

Because anonymous auth means "anyone", the storage rules are deliberately strict: **25 MB cap + content-type allow-list + auth required**. Remaining exposure is *volume*: many valid 25 MB uploads could still consume the Blaze free quota (~5 GB stored, ~1 GB/day download). Mitigations to do on your side:

- Set a **budget alert** in the Google Cloud console.
- Consider lowering the size cap in `firebase-storage-rules.rules` if your community mostly shares images.
- Do not deploy the storage rules / do not enable Storage at all if you don't need file attachments — the app runs fine without it.

## Hardening roadmap (recommended next steps)

Ordered roughly by impact-to-effort:

1. **Per-node `.validate` rules** — ✅ **shipped in v148** (`firebase-database-rules.json`, see *Controls in place*): message shape, string lengths (including the 5.7 M-char `url` cap that bounds base64 attachments), timestamp windows and boolean/map-leaf constraints are enforced server-side. Authorship bindings (`newData.child('u').val()` tied to the writer) are NOT included — they require the uid registry of item 3, since usernames are client-chosen strings today.
2. **Atomic uniqueness registries** — add `usernames/{name} → uid`, `displayNames/{name} → uid`, `colors/{hex} → uid` nodes with `.validate: !data.exists()` (or multi-path transactions) so name/color collisions become impossible server-side.
3. **Bind writes to the authenticated identity** — once records carry the Firebase `uid`, restrict `users/{name}` writes to `auth.uid === data.child('uid').val()`; this closes the overwrite-takeover (#1) and most forgery paths (#4).
4. **Move `sessToken` (and the password hash) out of the world-readable `users` node** — e.g. a private `creds/{uid}` node with read denied; rotate/expire tokens and add revocation.
5. **Firebase App Check** (reCAPTCHA v3 / Enterprise) — blocks custom scripted clients from hitting the DB/Storage at scale.
6. **Cloud Functions moderation** — move ban enforcement, rate limiting and message validation server-side (requires the Blaze plan; functions have their own free tier).
7. **True E2EE for DMs** — per-conversation keys via public-key exchange, never stored in the shared DB.
8. **Storage path rules** — restrict uploads to per-user paths (`uploads/{uid}/…`) matching `request.auth.uid` so files can be attributed and cleaned up.
9. **Stop writing IPs to world-readable nodes** (or drop them entirely) — the least-effort privacy win.

## Deploying the rules

```bash
# Realtime Database (required — free Spark plan)
firebase deploy --only database

# Cloud Storage (optional — requires Blaze)
firebase deploy --only storage
```

Or paste the file contents into *Firebase Console → Realtime Database / Storage → Rules* and publish. **Always deploy the database rules when you ship a new version of the HTML file** — a rules/client mismatch (e.g. missing `serverInvites`/`typing` allowances) silently breaks features.

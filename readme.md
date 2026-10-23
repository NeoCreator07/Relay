# Relay

**Relay** is a real-time chat app that ships as a **single HTML file**. There is no build step, no bundler and no install — open the file in a browser and chat. Everything lives in one document: the UI, the styles, the crypto, the animated backgrounds and the Firebase wiring. It runs on **Firebase** (anonymous auth + Realtime Database, with optional Storage for file attachments) and works on desktop and mobile.

- **Current release:** `relay-v145.html`
- **Firebase project:** `relay-87573`
- **Repository:** [github.com/NeoCreator07/Relay-Messaging](https://github.com/NeoCreator07/Relay-Messaging)
- **Owner:** NeoCreator07 (TerminalVelocity)
- **Security policy:** see [`SECURITY.md`](./SECURITY.md)

---

## Files in this folder

| File | What it is |
|---|---|
| `relay-v145.html` | **Current release** — the entire app in one file |
| `firebase-database-rules.json` | Security rules for Realtime Database — **required, deploy this** |
| `firebase-storage-rules.rules` | Security rules for Cloud Storage — optional (Blaze plan, see *Billing*) |
| `relay-texting-starter-kit.xlsx` | Reference kit: community rules, starter server/room tables, message schema, rules map |
| `relay-v144.html` | Previous release (sidebar rework) — kept for rollback |
| `relay-v143.html` | Original rewrite — kept for rollback |

---

## Features

### Identity & uniqueness (v145)

- **Auth-first boot.** Every visitor signs in to Firebase *anonymously before any database call* — this was the fatal bug in the old build (dead project + missing sign-in made every DB promise hang forever) and it is fixed for good.
- **Accounts.** Register with a username and password. Passwords are hashed client-side with SHA-256 + a per-user salt; a `sessToken` in `localStorage` gives you auto-login on return visits.
- **Unique usernames.** A username belongs to exactly one person. The register form shows a live **✓ available / ✕ taken** hint as you type.
- **Unique display names.** Display names are also unique — the check is case- and whitespace-insensitive, and a display name cannot impersonate someone else's *username* either. The profile editor warns live while you type and blocks saves on conflict.
- **Unique name colors.** Every user gets one color nobody else has. New accounts are auto-assigned the first free color from a 10-color palette (hue-wheel fallback when the palette runs out). Taken colors appear struck-through in the profile picker with a tooltip naming the current owner, and a last-guard reverts any save that would collide. Legacy accounts without a stored color claim a unique one automatically on their first login.

### Servers, rooms & chat

- **Servers with rooms inside them** (`#general`, `#random`, …), including hidden and private servers with one-time invite links.
- **Real-time messaging** via `child_added/changed/removed` listeners with batched `requestAnimationFrame` rendering (plus a `setTimeout` fallback so hidden tabs don't swallow messages).
- **Edit, delete, replies and emoji reactions** — all live-synced across users.
- **Typing indicators, presence / online list, per-room unread dots.**
- **Friends & DMs.** Send friend requests, accept, and open a direct-message channel keyed by the user pair (`_dm/<a>--<b>`).
- **Moderation.** Server owners can mute their server; server deletion is guarded against resurrecting nodes from open tabs.
- **Spam guards.** 2-second send/edit cooldowns and a 30-second repeated-message window.
- **GIF picker** powered by Giphy, and **file attachments up to 25 MB** via Firebase Storage (see *Billing* — degrades gracefully if Storage is not enabled).

### Links & media

- **Clickable links** in message text, with trailing punctuation handled and XSS-safe escaping.
- **Rich link embeds.** An embed chain of noembed.com (JSONP, works from any origin) → microlink fallback, cached per URL. YouTube links get an instant thumbnail and **click-to-play inline video** via `youtube-nocookie.com`.
- **In-app link popup.** Clicking any link (or embed card) opens a modal viewer with the provider, a preview (playable YouTube iframe, provider embed, image, or title+description), and *Open in new tab* / *Copy link* buttons. `Esc`, `✕` or the backdrop closes it; Ctrl/Cmd/Shift-click still does the browser default.

### Backgrounds & themes (v145)

- **All 13 free Vanta.js effects:** birds, clouds, fog, globe, halo, net, waves, cells, topology, dots, rings, trunk and ripple. Each effect's script is lazy-loaded only when selected (three.js r121 for most, p5.js 1.4.2 for topology and trunk), so the app stays light.
- **Persistent color changer.** Settings → Chat background lists every effect; picking one reveals that effect's *real* color options as pickers. Edits re-initialize the effect live, each effect's overrides are stored per-effect in `localStorage`, there's a Reset colors button, and **your chosen effect and colors survive refresh**.
- **Dark and light themes**, with extra text shadows over animated backgrounds for readability.

### Interface

- **Fully expanded sidebar.** One unified scroll column — servers, search, DMs and the online list flow at natural height with the header and user footer pinned. No nested box-inside-a-box scrolling; all servers are visible at once on desktop and ~9–10 rows on a 390px phone.
- **Auto-growing message input.** A `<textarea>` that grows with your content up to ~6 lines, then scrolls internally. `Enter` sends, `Shift+Enter` adds a newline.
- **Mobile support** with a slide-in sidebar.

---

## Quick start

1. **Deploy the database rules** — paste `firebase-database-rules.json` into *Firebase Console → Realtime Database → Rules* (or `firebase deploy --only database`). Required for invites and typing indicators to work.
2. **Open `relay-v145.html`** in any modern browser. Internet access is required (Firebase, fonts and CDN scripts load from the web).
3. *(Optional)* Deploy `firebase-storage-rules.rules` if you want file attachments — see *Billing* below.

> The embedded Firebase config points at the live `relay-87573` project. To run against your own project, replace the `firebaseConfig` block in the file's `<script>` and deploy both rules files to it.

## Firebase & billing

| Service | Plan | Notes |
|---|---|---|
| Realtime Database | **Free (Spark)** | Chat, users, servers, DMs, friends, presence, reactions all run here. Deploy `firebase-database-rules.json`. |
| Cloud Storage | **Blaze required** (free quota under it) | Since Feb 2026 Storage requires Blaze on *all* buckets. Blaze keeps a free quota (~5 GB stored, ~1 GB/day downloads, 20K/50K ops/day) — $0/month under it, ~$0.02/GB stored + ~$0.12/GB downloaded above. **If you don't upgrade, everything works except the ＋ attach button** (it shows a friendly "Upload failed" message). |
| Anonymous Auth | Free | Must be enabled in *Console → Authentication → Sign-in method*. |

**Cost-abuse protection is built into the storage rules:** uploads are capped at 25 MB and restricted to an allow-list of media/document content types, so a stranger can't run up your bill with junk. Set a budget alert in the Google Cloud console anyway if you enable Storage.

## Data layout (Realtime Database)

| Node | Shape |
|---|---|
| `users/{username}` | `pw` (SHA-256), `salt`, `sessToken`, `display`, `color`, `avatar` (base64), `ip`, `ips`, `createdAt`, `lastLogin` |
| `servers/{sid}` | `name`, `owner`, `createdAt`, `muted`, hidden/private flags, `rooms/{rid} → {name}` |
| `messages/{sid}/{rid}` | `t`, `text`, `type`, `u`, `url`, `w`, `h`, `edited`, `reactions/{emoji}/{username}` |
| `messages/_dm/{a--b}` | Same message shape, keyed by the alphabetical user pair |
| `friends/{a}/{b}`, `friendRequests/{to}/{from}` | Friend graph |
| `presence/{username}` | `ip`, `lastSeen`, `online` |
| `typing` | Per-channel typing state |
| `serverInvites` | One-time invite links |
| `config` | `admins`, `banned`, `giphyKey` — **read-only to clients** (rules deny all writes) |

Uniqueness of usernames / display names / colors is enforced client-side against a live snapshot of `users` — no schema or rules changes were needed for it.

## Local persistence (localStorage)

| Key | Purpose |
|---|---|
| `relay_session` | `{u, t}` — username + sessToken for auto-login (removed on logout) |
| `lc_theme` | `dark` / `light` |
| `lc_vanta` | Active background effect id (`none` if off) |
| `lc_vanta_colors` | JSON of per-effect color overrides — why your colors survive refresh |
| `lc_color` | Your unique name color |
| `lc_pfp` | Avatar (compressed base64) |
| `lc_sound`, `lc_notif` | `1` / `0` toggles |
| `unread_*`, `seen_*`, `draft_*` | Per-channel unread counts, last-seen timestamps, message drafts |

## Version history

| Version | What changed |
|---|---|
| **v145** | Unique usernames + display names (live hints, profile guards), unique per-person name colors (auto-assign, taken-guard, legacy claim), complete 13-effect Vanta suite with a persistent per-effect color changer that survives refresh. |
| **v144** | Sidebar rebuilt into one unified scroll column — all servers fully visible, no nested scroll boxes; header/footer pinned. |
| **v143** | Ground-up rewrite preserving the original look: fixed the fatal anonymous-auth bug, rooms inside servers, friends/DMs/invites, reactions/replies/edit/delete, spam guards, batched rendering, lazy three.js. Storage rules hardened to 25 MB + type allow-list. |
| *starter kit* | `relay-texting-starter-kit.xlsx` — community rules, starter server/room tables, message schema and a rules map for building a companion texting client. Reference material only, nothing deployed. |

## Testing

Every release was verified before delivery: syntax checks on all script blocks, offline UI harnesses run against the real file's CSS/DOM/JS (32 tests for v143's link/textarea work, 22 for v144's sidebar, 42 for v145's uniqueness + Vanta persistence), live boot smoke tests against the real Firebase project with zero page errors, mobile 390px checks, and visual audits of the screenshots. The harnesses live in `/home/z/my-project/scripts/` for future regression runs.

## Security

See **[`SECURITY.md`](./SECURITY.md)** for the full policy: how to report vulnerabilities privately (GitHub private reporting on the repo), the trust model, the attack-surface inventory, six concrete attack scenarios the current rules permit, privacy considerations, known limitations, and a hardening roadmap.

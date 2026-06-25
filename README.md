# ⚡ Relay

A real-time group chat web app — single HTML file, no backend server, no build step. Just open it, sign up, and start chatting.

**Current version:** 83

---

## Features

- **Accounts** — sign up with a username and password (stored as plain text in Firebase — see [Security Notes](#security-notes))
- **Auto sign-in** — your session is saved locally, so you stay logged in across visits
- **Rooms** — join the default `General` room, search for others, or create your own
- **Private rooms** — optionally protect a room with a security code at creation
- **Real-time messaging** — powered by Firebase Realtime Database; messages sync instantly across everyone in a room
- **GIFs** — built-in Giphy search and picker
- **Image uploads** — attach photos directly from your device (auto-compressed before sending)
- **Reactions** — hover any message to react with 👍 ❤️ 😂 😮 😢 🔥
- **Replies** — quote and reply to a specific message
- **Profiles** — custom display name, name color (color picker), and profile photo
- **Click-to-view profiles** — click anyone's name or avatar to see their profile card
- **Animated backgrounds** — switchable Vanta.js backgrounds (Birds, Waves, Net, Fog, Dots, or None) via the settings gear
- **Collapsible sidebar** — toggle with the `‹ ›` caret; resizes content for more chat space
- **Admin panel** — hidden tool for moderating rooms and users (see below)
- **Spam protection** — per-message cooldown, rate limiting, and duplicate-message blocking
- **Profanity filter** — censors common profanity, including leetspeak variants, before a message is sent
- **Shift+Enter** for a new line, **Enter** to send

---

## Tech Stack

| Layer | Tech |
|---|---|
| Frontend | Vanilla HTML/CSS/JS — single file, no framework |
| Realtime data | [Firebase Realtime Database](https://firebase.google.com/docs/database) (SDK bundled inline) |
| GIFs | [Giphy API](https://developers.giphy.com/) |
| Backgrounds | [Vanta.js](https://www.vantajs.com/) (loaded on demand, per selected background) |
| Hosting | Any static host — built and tested for [Cloudflare Pages](https://pages.dev) via GitHub |

The Firebase SDK is **bundled directly into the HTML file** (not loaded from a CDN), so the app has no external script dependencies at runtime except Vanta.js backgrounds and Giphy's API, both loaded only when needed.

---

## Setup

### 1. Create a Firebase project

1. Go to [console.firebase.google.com](https://console.firebase.google.com) → **Add project**
2. In your project: **Build → Realtime Database → Create Database** → choose a region → start in **test mode** (or use the rules below)
3. **Project settings (gear icon) → General → Your apps → Add app → Web**, then copy the `firebaseConfig` object

### 2. Add your config to `index.html`

Open `index.html`, find the `firebaseConfig` block near the bottom (inside the last `<script>` tag), and replace it with your own values:

```js
const firebaseConfig = {
  apiKey: "...",
  authDomain: "...",
  databaseURL: "...",      // make sure this is included!
  projectId: "...",
  storageBucket: "...",
  messagingSenderId: "...",
  appId: "..."
};
```

> ⚠️ `databaseURL` is required for Realtime Database and is sometimes left out of the config Firebase shows you by default. Double check it's there — it usually looks like `https://YOUR_PROJECT-default-rtdb.firebaseio.com` (or a region-specific variant).

### 3. Set your database rules

In Firebase Console → **Realtime Database → Rules**, paste:

```json
{
  "rules": {
    "users":         { ".read": true, ".write": true },
    "messages":       { ".read": true, ".write": true },
    "rooms":          { ".read": true, ".write": true },
    "userRooms":      { ".read": true, ".write": true },
    "roomPasswords":  { ".read": true, ".write": true }
  }
}
```

Click **Publish**.

### 4. (Optional) Add your own Giphy API key

Get a free key at [developers.giphy.com](https://developers.giphy.com), then replace the value of `GIPHY_KEY` near the top of the script section.

### 5. Deploy

**Option A — Cloudflare Pages (recommended):**
1. Push `index.html` to a GitHub repo
2. Cloudflare Dashboard → **Workers & Pages → Create → Pages** → connect your repo
3. Framework preset: **None**, leave build command and output directory blank
4. Deploy — you'll get a live `*.pages.dev` URL

**Option B — any static host:**
Just upload `index.html` anywhere that serves static files (Netlify, Vercel, GitHub Pages, S3, etc.) — no build step required.

---

## Using the App

### Sign up / sign in
Pick a username (3+ characters, letters/numbers/underscores) and a password (4+ characters). New accounts are created instantly — no email or verification step.

### Rooms
- The sidebar lists rooms you've joined
- Use the search bar to find public rooms by name
- Click **+** to create a new room — give it a name and, optionally, a security code
- Rooms without a code are open to anyone who finds them; rooms with a code prompt for it before joining

### Profile
Click your name/avatar at the bottom of the sidebar to:
- Upload a profile photo (auto-resized/compressed)
- Pick a name color with the color wheel

### Messages
- **Hover** any message to reveal a `•••` button — react or reply
- **Shift+Enter** inserts a newline; **Enter** sends
- **Click** a sender's name or avatar to view their profile

### Settings
Click the **⚙ gear** icon (top of sidebar) to change the animated background or sign out.

### Admin panel
**Triple-click/tap the logo** (top-left, next to "Relay") within ~0.7 seconds to open the admin panel. From here you can:
- Delete all messages in a given room (by room ID)
- Delete a user account
- List all users / all rooms currently in the database
- View quick links and the current app version

> The admin panel has no authentication — it's gated only by the triple-tap gesture. Don't rely on it for real moderation security; anyone with access to the deployed site can use it.

---

## Security Notes

This app prioritizes simplicity and "it just works" over hardened security. Before using it for anything beyond a casual project, be aware:

- **Passwords are stored in plain text** in the Firebase database, not hashed. Anyone with read access to your database (including via the open rules above) can see every password.
- **Database rules are fully open** (`read: true, write: true` on most paths) by default, meaning anyone with your Firebase config — which is visible in the page source — can read or write any data, including other users' messages and accounts.
- **No rate limiting at the database level** — the in-app spam protection is client-side only and can be bypassed by anyone calling the Firebase API directly.
- **The admin panel has no password** — it's a UI convenience, not a security boundary.

This setup is appropriate for a private group chat among people you trust (friends, a small community, a personal project). It is **not** suitable for handling sensitive data or for public-facing deployments without significant hardening (proper auth, hashed passwords, scoped database rules, server-side rate limiting, etc.).

---

## File Structure

This is a single-file app — everything lives in `index.html`:

```
index.html
├── <style>           All CSS (no external stylesheets)
├── HTML               Auth screen, sidebar, chat UI, modals
└── <script> × 4
    ├── Firebase App SDK (bundled)
    ├── Firebase Database SDK (bundled)
    ├── Three.js (loaded from CDN, required by Vanta.js)
    └── App logic — auth, rooms, messaging, profiles, reactions, admin tools
```

No `npm install`, no build step, no `node_modules` — just open the file in a browser or deploy it as-is.

---

## Customization Ideas

- Swap the logo image (`#logo-img` `src`) for your own branding
- Add more Vanta.js background presets (`VOPTS` / `VCDNS` objects)
- Extend the emoji reaction set (`EMOJIS` / `EKEYS` objects)
- Adjust spam thresholds (`COOLDOWN`, `WLIMIT`, `WINDOW_MS`, `DLIMIT`)
- Edit the profanity word list (`BWORDS` array)

---

## Version History

Every change to this app increments the version number shown on the login screen and in the admin panel. Check `VERSION` near the top of the script to see the current build.

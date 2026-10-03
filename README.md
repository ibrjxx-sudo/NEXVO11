# NEXVO

NEXVO is a mobile-first Arabic messaging interface with a same-origin Node.js API and a SQLite database.

## Requirements

- Node.js 22.13 or newer (the built-in SQLite module is used).
- Copy `.env.example` to `.env` and replace both secret placeholders before starting the app:
  - Set a unique administrator password of at least 14 characters.
  - Set `SESSION_SECRET` to at least 32 cryptographically random characters.
- The password previously shared in chat must not be reused. It was exposed; choose a new unique password.

Generate a session secret with:

```powershell
node -e "console.log(require('node:crypto').randomBytes(48).toString('base64url'))"
```

The `.env.example` values are configured for Render deployment: they bind to `0.0.0.0`, use Render's default port, enable production cookies, and store persistent data under `/var/data`. Replace the secret placeholders with new secrets; do not put real passwords in `.env.example` or commit them.

### Deploying to Render

The repository includes a `render.yaml` Blueprint. In Render, choose **New + → Blueprint**, connect the Git repository containing this project, and provide the requested `ADMIN_EMAIL` and a new `ADMIN_PASSWORD` when prompted. The Blueprint generates `SESSION_SECRET`, builds with `npm ci`, starts with `npm start`, checks `/healthz`, binds on `0.0.0.0`, and mounts a persistent disk at `/var/data` for SQLite and uploaded status media.

If an existing manually created service crashes with `Missing or unsafe SESSION_SECRET`, open **Render Dashboard → your web service → Environment** and add:

- `ADMIN_EMAIL`: your administrator email.
- `ADMIN_PASSWORD`: a new unique password of at least 14 characters; do not reuse a password exposed in chat.
- `SESSION_SECRET`: a randomly generated value of at least 32 characters. You can generate one with `node -e "console.log(require('node:crypto').randomBytes(48).toString('base64url'))"` and paste it directly into Render.
- `NODE_ENV=production`, `HOST=0.0.0.0`, and `SESSION_COOKIE_SECURE=true`.
- `NEXVO_DATA_DIR=/var/data` if the service has a persistent disk mounted at `/var/data`.

Save the environment changes and redeploy. Do not upload `1.env`, `.env`, or `.env.example` as Render environment settings; Render does not read those files as service secrets. For an existing service, also explicitly set **Settings → Build & Deploy → Build Command** to `npm ci`; the deployment logs show it is currently running Yarn, so this service has not synced the Blueprint's npm build command. Commit and push this repository's latest changes before deploying them.

This configuration uses a paid Starter web service because Render persistent disks are not available on the Free plan. Keep a single service instance: SQLite and the attached disk are local to one instance. Push the project and `render.yaml` to your Git provider before creating the Blueprint; Render deploys from that repository.

The project uses **npm** (`packageManager` is declared in `package.json`) and `package-lock.json`; keep that lockfile and use `npm ci` as the Render Build Command. If Render logs warn that Yarn found `package-lock.json`, that deploy is still running Yarn: in the existing service open **Settings → Build & Deploy**, set **Build Command** to `npm ci`, save, then trigger a deploy with **Clear build cache & deploy**. Alternatively, sync/create the service from `render.yaml`. Updating repository files alone does not change a manually configured existing service. Do not delete `package-lock.json`; `npm ci` requires it.

For local development, change `HOST` to `127.0.0.1`, `PORT` to `3000`, `NODE_ENV` to `development`, `SESSION_COOKIE_SECURE` to `false`, and `NEXVO_DATA_DIR` to `.data` in `.env`, then start with `npm start` and open <http://127.0.0.1:3000>. The administrator email and password are provisioned from environment variables and are never embedded in browser code.

For deployment, terminate HTTPS at a trusted reverse proxy that forwards WebSocket upgrades and keeps realtime connections open. Keep `NODE_ENV=production` and `SESSION_COOKIE_SECURE=true`, store real secrets in the hosting provider's secret manager, and back up the database and uploaded media together.

## Current server-backed features

- Profile registration with a phone number, username, password, and optional profile image.
- Passwords hashed with Node's scrypt; opaque, HttpOnly, SameSite session cookies backed by SQLite.
- Environment-provisioned administrator sign-in, user directory and account status/verification controls.
- Persistent brand settings and image assets for the app logo, app icon, favicon, default avatar, and Open Graph image.
- Administrative audit log.
- Private one-to-one conversations backed by SQLite, with messages delivered in real time over authenticated WebSockets.
- User lookup by exact username or phone number, without exposing phone numbers in search results.
- Text and supported image/video statuses shared with conversation peers, served from the server, and removed after 24 hours.

Video status media supports byte-range requests for browser playback. The status upload limit is 50 MB per item; supported formats are PNG, JPEG, WEBP, GIF, MP4, and WebM.

Real phone verification, media transcoding/resumable uploads, WebRTC, groups, and end-to-end encryption are not implemented. Messages are stored as text in SQLite and are not end-to-end encrypted. Place the app behind HTTPS in production; administrators do not have an API or dashboard feature to read private conversation content.

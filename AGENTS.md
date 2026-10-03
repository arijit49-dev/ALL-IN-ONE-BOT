# AGENTS.md

## What this is
"ALL IN ONE BOT" — a Discord bot (discord.js v14) with a MongoDB datastore and a Lavalink music node. It also runs a tiny Express web server on **port 3000** that serves `index.html` (a static "GlaceYT" splash/status page). That page is the only web-visible surface; the bot itself runs as a background process.

## Running here (Base44)
`docker-compose.base44.yml` runs three services:
- `mongodb` — plain `mongo:7` with local credentials (`botuser` / `botpass123`, db `discord-bot`).
- `lavalink` — `ghcr.io/lavalink-devs/lavalink:4`, config mounted from `lavalink/application.yml`. Replaces the dead public node that `config.json` points at.
- `app` — `node:22`, repo bind-mounted, runs `npm install && node index.js`, maps host port **3000**.

`app` waits for both `mongodb` and `lavalink` to be healthy before starting.

### Env overrides
The bot reads its Lavalink node from `config.json`, but `events/music.js` now prefers env vars so the repo config stays untouched:
`LAVALINK_HOST` (→ `lavalink`), `LAVALINK_PORT` (→ `2333`), `LAVALINK_PASSWORD` (→ `glaceyt`). The Mongo URI comes from `MONGODB_URI` (code reads `config.mongodbUri || process.env.MONGODB_URI`).

`npm install` runs on every container start (no lockfile is committed). Dependencies live in the `app-node-modules` volume.

## Required credentials
- `TOKEN` — Discord bot token, delivered from `/run/base44/app.env`. Required for the bot to connect. A login failure is non-fatal (caught in `main.js`), so the Express page stays online.

## Discord Developer Portal requirements (non-obvious)
`main.js` constructs the client with **every** `GatewayIntentBits`, including the privileged ones. The bot fails to connect with `Used disallowed intents` until these are enabled:
1. Discord Developer Portal → your application → **Bot**.
2. Under **Privileged Gateway Intents**, enable **Presence Intent**, **Server Members Intent**, **Message Content Intent**.
3. Save, then restart the `app` service.

## Lavalink notes (non-obvious)
- Plugin **dependencies** must live under `lavalink.plugins:` — a top-level `plugins:` list is silently ignored. Top-level `plugins:` is only for plugin *settings* (e.g. `plugins.youtube`).
- The YouTube plugin is **not** on Maven Central; it needs `repository: "https://maven.lavalink.dev/releases"`.
- `/version` and all REST endpoints require the password: `curl -H "Authorization: glaceyt" http://localhost:2333/version`.
- Verify YouTube search: `curl -H "Authorization: glaceyt" 'http://localhost:2333/v4/loadtracks?identifier=ytmsearch%3Atest'`.
- Lavalink plugin jars are downloaded at container start into `/opt/Lavalink/plugins` (ephemeral; re-downloaded on recreate).

## Notes
- `shiva.js` pings an external backend (`API_BASE_URL`, default `http://0.0.0.0:10000/api`) that is not part of this repo; a connection failure there is expected and harmless.
- `main.js` also contacts `https://server-backend-tdpa.onrender.com` for command verification; unreachable is fine.
- `config.json` ships with empty `token`/`mongodbUri` and a public Lavalink/Spotify config. Env vars take precedence for the token, Mongo URI, and Lavalink node.

## Verifying it works
- `curl -s -o /dev/null -w "%{http_code}" http://localhost:3000/` → `200`.
- `docker compose -f docker-compose.base44.yml ps` → all three services healthy.
- App logs should show `Connected to MongoDB ✅`, `Bot Name: …`, `[ LAVALINK CONNECTION ] Node connected: lavalink`, and `Successfully Loaded Slash Commands ✅`.
- Lavalink logs should show `Loaded 'youtube-plugin-1.18.2.jar'`.

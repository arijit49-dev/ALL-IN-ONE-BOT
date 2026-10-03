# AGENTS.md

## What this is
"ALL IN ONE BOT" — a Discord bot (discord.js v14) with a MongoDB datastore. It also runs a tiny Express web server on **port 3000** that serves `index.html` (a static "GlaceYT" splash/status page). That page is the only web-visible surface; the bot itself runs as a background process.

## Running here (Base44)
- `docker-compose.base44.yml` runs two services:
  - `mongodb` — plain `mongo:7` with local credentials (`botuser` / `botpass123`, db `discord-bot`).
  - `app` — `node:22`, repo bind-mounted, runs `npm install && node index.js`, maps host port **3000**.
- The Mongo connection string is injected via the `MONGODB_URI` env var (the code reads `config.mongodbUri || process.env.MONGODB_URI`).
- `npm install` runs on every container start (no lockfile is committed). Dependencies live in the `app-node-modules` volume.

## Required credentials
- `TOKEN` — Discord bot token, delivered from `/run/base44/app.env`. Required for the bot to connect. Without a valid token the bot does not connect (the Express page still runs — `client.login()` has a `.catch()` so a login failure is non-fatal).

## Discord Developer Portal requirements (non-obvious)
`main.js` constructs the client with **every** `GatewayIntentBits`. This includes the privileged intents, so the bot will fail to connect with `Used disallowed intents` until they are enabled:
1. Open the Discord Developer Portal → your application → **Bot**.
2. Under **Privileged Gateway Intents**, enable **Presence Intent**, **Server Members Intent**, and **Message Content Intent**.
3. Save, then restart the `app` service.

## Notes
- `shiva.js` pings an external backend (`API_BASE_URL`, defaults to `http://0.0.0.0:10000/api`) that is not part of this repo; a connection failure there is expected and harmless.
- `main.js` also contacts `https://server-backend-tdpa.onrender.com` for command verification; unreachable is fine.
- `config.json` ships with empty `token`/`mongodbUri` and a public Lavalink/Spotify config. Env vars take precedence over `config.json` for the token and Mongo URI.

## Verifying it works
- `curl -s -o /dev/null -w "%{http_code}" http://localhost:3000/` → `200`.
- `docker compose -f docker-compose.base44.yml ps` → both services healthy.
- App logs should show `Connected to MongoDB ✅` and `Listening to GlaceYT : http://localhost:3000`. A connected bot additionally logs the bot name/client id; `Used disallowed intents` means the portal intents above are not enabled.

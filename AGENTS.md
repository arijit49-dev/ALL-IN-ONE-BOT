# Base44 Development Environment

## Project Overview
Discord "All in One Bot" (aio) — a Node.js Discord bot that also runs an Express web server on port 3000 serving `index.html`.

## Architecture
- **Entry point**: `index.js` → loads `main.js` (Discord client + Express server), `bot.js` (empty), `shiva.js` (API status check)
- **Express server**: listens on port 3000, serves `index.html` at `/`
- **Discord client**: logs in with `process.env.TOKEN || config.token`
- **MongoDB**: connects via `config.mongodbUri || process.env.MONGODB_URI`; local MongoDB runs as a compose service
- **Commands**: loaded from `commands/` directories filtered by `config.json` `categories`
- **Events**: loaded from `events/` directory

## Setup
- `docker compose -f docker-compose.base44.yml up -d --build`
- MongoDB runs as a compose service (mongo:7), app connects via `MONGODB_URI=mongodb://mongo:27017`
- Dependencies installed on container startup via `npm install`

## Secrets
- `TOKEN` — Discord bot token (required for bot login; web server runs without it). Development placeholder generated; replace with real token from Discord Developer Portal for bot functionality.

## Key Fix Applied
- `main.js`: `client.login()` rejection was unhandled, crashing the process (and the Express server) when the token is invalid. Added `.catch()` to log the error and keep the web server running.

## Verification
- `curl http://localhost:3000/` returns 200 (serves index.html)
- `docker compose -f docker-compose.base44.yml ps` shows both services healthy
- Bot login error is logged but does not crash the process

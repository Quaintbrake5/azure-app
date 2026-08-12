# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

Minimal Express.js application deployed to Azure App Service. Single entry point, no build step, no test framework configured.

## Commands

```bash
# Start the server (development or production)
node server.js          # or: npm start

# Install dependencies
npm install
```

## Deployment

Deployed to Azure App Service (`azure-node-web-app`) via GitHub Actions using OIDC (no secrets stored in the repo). The CI workflow is at `.github/workflows/deploy.yml`. Azure provides the `PORT` env var at runtime — the server must read `process.env.PORT` with a local fallback.

**Startup command** in Azure App Service must be `npm start` (not the default `node index.js`), since the entry point is `server.js`.

## Commands

```bash
# Start the server (development or production)
node server.js          # or: npm start

# Install dependencies
npm install

# CI (runs in GitHub Actions)
npm ci
```

## Architecture

- **Entry point**: `server.js` — single file, CommonJS modules (`type: "commonjs"` in package.json)
- **Framework**: Express v5 — two routes only: `GET /` (HTML) and `GET /students` (JSON)
- **Port**: reads `process.env.PORT`, defaults to `3000` — Azure overrides this at runtime
- **State**: entirely in-memory; no database, no caching, no external services
- **Module format**: CommonJS (`require`/`module.exports`) — package.json has `"type": "commonjs"`

## Conventions

- Routes are defined directly in `server.js` with inline handlers — no route modules or middleware layers yet
- Hardcoded mock data in routes (e.g. `/students`); no ORM or data access layer
- No request validation, logging, or error handling middleware at this stage
- No build step — Express serves raw JS directly; the GitHub Actions `build` step is a no-op (`npm run build --if-present`)

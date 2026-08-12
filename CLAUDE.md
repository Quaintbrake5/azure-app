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

## Architecture

- **Entry point**: `server.js` — single file, CommonJS modules (`type: "commonjs"` in package.json)
- **Framework**: Express v5 — two routes only: `GET /` (HTML) and `GET /students` (JSON)
- **Port**: reads `process.env.PORT`, defaults to `3000`
- **State**: entirely in-memory; no database, no caching, no external services
- **Deployment target**: Azure App Service — this is the app's primary purpose

## Conventions

- Routes are defined directly in `server.js` with inline handlers — no route modules or middleware layers yet
- Hardcoded mock data in routes (e.g. `/students`); no ORM or data access layer
- No request validation, logging, or error handling middleware at this stage

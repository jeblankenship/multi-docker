# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

A training exercise demonstrating a multi-container Docker application deployed to AWS Elastic Beanstalk. The app is a Fibonacci calculator that accepts an index, computes the value asynchronously, and stores results in both Redis (cache) and Postgres (history).

## Development

Start the entire stack for local development:

```bash
docker compose up --build
```

The app is accessible at `http://localhost:3050`.

Install dependencies for local (non-Docker) work:

```bash
cd client && npm install
cd server && npm install
cd worker && npm install   # no package-lock; no scripts defined
```

Run the client dev server standalone (port 3000):

```bash
cd client && npm start
```

Run the API server standalone (port 5000, requires Redis and Postgres env vars):

```bash
cd server && npm run dev   # nodemon
```

Run security audits (mirrors CI):

```bash
npm audit --audit-level=high   # run from client/ and server/ separately
```

## Architecture

Six services, all defined in `docker-compose.yml`:

| Service | Port (internal) | Role |
|---------|----------------|------|
| `nginx` | 80 → host 3050 | Reverse proxy routing `/api/*` to the Express server and `/` to the React client |
| `client` | 3000 | Vite + React SPA |
| `api` | 5000 | Express REST API |
| `worker` | — | Redis subscriber; computes Fibonacci values |
| `postgres` | 5432 | Persists all submitted indexes |
| `redis` | 6379 | Pub/sub channel + hash cache of computed values |

**Data flow:** User submits an index → `POST /api/values` → API stores `"Nothing yet!"` in Redis hash and publishes to `insert` channel → Worker picks up the message, computes `fib(n)` recursively, and writes the result back to the Redis hash. The API also inserts the raw index into Postgres. The front-end polls both `/api/values/current` (Redis) and `/api/values/all` (Postgres) on mount.

**Index cap:** Indexes above 40 are rejected by the server to prevent runaway recursion in the worker.

**nginx routing:** `nginx/default.conf` proxies `/api/*` (stripping the `/api` prefix via `rewrite`) to the `api` upstream and everything else to the `client` upstream. `client/nginx/default.conf` is used only in the **production** client image to serve the static Vite build.

## Build & CI

Each service has two Dockerfiles:
- `Dockerfile.dev` — used by `docker compose up` for local development (mounts source as a volume)
- `Dockerfile` — multi-stage production build; used by CI

CI (`.github/workflows/docker-image.yml`) runs on pushes/PRs to `main`:
1. `npm audit --audit-level=high` for `client` and `server`
2. Parallel production image builds for all four services (client, nginx, worker, server)
3. A gated final job that requires all of the above to pass

## AWS Deployment

`Dockerrun.aws.json` defines the Elastic Beanstalk multi-container configuration. Images are pulled from Docker Hub under the `docscode/` namespace. `nginx` is marked `essential: true`; the other containers are non-essential.

# multi-docker

[![Docker Image CI](https://github.com/jeblankenship/multi-docker/actions/workflows/docker-image.yml/badge.svg)](https://github.com/jeblankenship/multi-docker/actions/workflows/docker-image.yml)

A multi-container Docker application that calculates Fibonacci numbers. Used as a training exercise for orchestrating multiple services with Docker Compose and deploying to AWS Elastic Beanstalk.

## Architecture

Six containers work together:

| Service | Role |
|---------|------|
| **nginx** | Reverse proxy — routes `/api/*` to the Express API and everything else to the React client |
| **client** | React SPA (Vite) — form to submit an index, displays computed results |
| **api** | Express REST API — accepts submissions, writes to Redis and Postgres |
| **worker** | Redis subscriber — computes `fib(n)` and stores the result back in Redis |
| **postgres** | Persists all submitted indexes |
| **redis** | Pub/sub channel and hash cache for computed values |

**Data flow:** User submits an index → API stores `"Nothing yet!"` in Redis and publishes to the `insert` channel → Worker computes `fib(n)` and writes the result back to Redis. The UI polls both Redis (current computed values) and Postgres (all seen indexes). Indexes above 40 are rejected to prevent runaway recursion.

## Running locally

```bash
docker compose up --build
```

Open `http://localhost:3050`.

## CI

GitHub Actions runs on every push/PR to `main`:

1. `npm audit --audit-level=high` for `client` and `server`
2. Production Docker image builds for all four services (parallel)
3. A gated job that requires all of the above to pass

## Deployment

`Dockerrun.aws.json` defines the AWS Elastic Beanstalk multi-container configuration. Images are pulled from Docker Hub under the `docscode/` namespace.

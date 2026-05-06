---
title: "Docker Compose for Local Development"
date: 2025-02-01
tags: ["docker", "devops", "tooling"]
categories: ["DevOps"]
description: "How to use Docker Compose to set up a reproducible local dev environment."
showToc: true
draft: false
---

"Works on my machine" is a joke that stopped being funny the first time it cost you a production incident. Docker Compose eliminates the problem by defining your entire local environment as code — databases, caches, message queues, and your app itself — in a single file that every team member can use identically.

## Why Docker Compose Over Plain Docker

You could run each service as a separate `docker run` command, but managing networks, volumes, and environment variables by hand quickly becomes unmaintainable. Compose gives you:

- **Declarative config** — one `compose.yaml` describes the entire stack.
- **Named networks** — services discover each other by name, not IP.
- **Managed volumes** — persistent data survives container restarts.
- **Ordered startup** — `depends_on` with `condition: service_healthy` prevents race conditions.

## A Real-World compose.yaml

Here's a compose file for a Go API backed by PostgreSQL and Redis:

```yaml
services:
  api:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "8080:8080"
    environment:
      DATABASE_URL: "postgres://dev:dev@postgres:5432/mydb?sslmode=disable"
      REDIS_URL: "redis://redis:6379"
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_started
    volumes:
      - .:/app   # mount source for live reload

  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: dev
      POSTGRES_PASSWORD: dev
      POSTGRES_DB: mydb
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U dev -d mydb"]
      interval: 5s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

volumes:
  postgres_data:
```

## A Minimal Dockerfile for Go

```dockerfile
FROM golang:1.22-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN go build -o /server ./cmd/server

FROM alpine:3.19
COPY --from=builder /server /server
EXPOSE 8080
ENTRYPOINT ["/server"]
```

The multi-stage build keeps the final image lean — only the compiled binary ships.

## Useful Compose Commands

```bash
# Start all services in detached mode
docker compose up -d

# Stream logs from the api service
docker compose logs -f api

# Open a psql shell inside the postgres container
docker compose exec postgres psql -U dev -d mydb

# Rebuild the api image after code changes
docker compose up -d --build api

# Tear down everything, including volumes
docker compose down -v
```

## Live Reload with Air

For Go development, mount the source directory and run [Air](https://github.com/air-verse/air) as the container entrypoint for hot-reloading:

```yaml
  api:
    image: cosmtrek/air:latest
    working_dir: /app
    volumes:
      - .:/app
```

Air watches for file changes and recompiles automatically, giving you a fast feedback loop without rebuilding the Docker image on every change.

## Health Checks Are Non-Negotiable

Always define a `healthcheck` on stateful services like PostgreSQL. Without it, your app container may start and attempt to connect before the database is ready to accept connections, resulting in a crash-loop that's hard to diagnose. The `condition: service_healthy` in `depends_on` ensures Compose waits for the health check to pass before starting dependent services.

## Wrapping Up

A well-crafted `compose.yaml` is one of the highest-leverage investments you can make in a project. It onboards new developers in minutes, eliminates environment-specific bugs, and documents service dependencies in a machine-readable format. Add it to your repo on day one.

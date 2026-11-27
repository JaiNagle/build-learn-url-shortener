# URL Shortener

A small URL shortening service built with **Flask**, **PostgreSQL** and **Redis**, packaged with Docker Compose. Built as a learning project following the [NextWork URL shortener tutorial](https://nextwork.ai/projects/02a35a5b-1d8b-43ee-9982-6ceef44145e5).

## How it works

1. `POST /shorten` stores the long URL in PostgreSQL. The auto-incrementing row ID is encoded in **base62** (`0-9a-zA-Z`) to produce the short code.
2. The code-to-URL mapping is written to Redis so repeat lookups skip the database.
3. `GET /<short_code>` checks Redis first (cache hit), falls back to PostgreSQL on a miss, re-populates the cache, then issues a `302` redirect.
4. `POST /shorten` is rate limited per client IP (10 requests per 60 seconds) using a Redis counter with an expiry.

## Features

- Base62 short codes derived from database IDs, so no collision checks are needed
- Redis read-through cache for fast redirects
- Per-IP rate limiting on link creation (returns `429` when exceeded)
- Health check endpoint
- One-command local setup with Docker Compose, including health checks so the app waits for Postgres and Redis

## Tech stack

Python 3.12, Flask, psycopg2, redis-py, PostgreSQL 16, Redis 7, Docker Compose.

## Getting started

### Prerequisites

Docker and Docker Compose.

### Configure

`docker-compose.yml` leaves `POSTGRES_PASSWORD` empty so no secret is committed. Set a password for both the `web` and `db` services before starting (the Postgres image refuses to initialise without one):

```yaml
- POSTGRES_PASSWORD=choose-a-password
```

### Run

```bash
docker compose up --build
```

The service listens on `http://localhost:5000`. The `urls` table is created automatically from `app/init.sql`.

## API

| Method | Path | Description |
| --- | --- | --- |
| `POST` | `/shorten` | Body: `{"url": "https://example.com/very/long"}`. Returns `201` with `short_url` and `short_code`. |
| `GET` | `/<short_code>` | Redirects (`302`) to the original URL, or `404` if unknown. |
| `GET` | `/health` | Returns `{"status": "healthy"}`. |

### Example

```bash
curl -X POST http://localhost:5000/shorten \
  -H "Content-Type: application/json" \
  -d '{"url": "https://github.com/JaiNagle"}'

# {"short_code":"1","short_url":"http://localhost:5000/1"}

curl -i http://localhost:5000/1
```

Watch the `web` container logs for `CACHE HIT` / `CACHE MISS` lines to see the cache working.

## Project structure

```
.
├── docker-compose.yml   # web + Postgres + Redis
└── app/
    ├── app.py           # Flask app: shorten, redirect, rate limit, health
    ├── init.sql         # urls table schema
    ├── requirements.txt
    └── Dockerfile
```

## Possible improvements

- Validate submitted URLs and restrict schemes
- Expire cache entries (TTL) and add click analytics
- Use a connection pool instead of opening a connection per request
- Make the generated short URL host configurable instead of hardcoding `localhost:5000`

# Refined_RAG_Project

> Built and maintained by Claude Code via OpenClaw (Moltbot).

## Description
A company called Refined have lots of data on SAP HANA

## Quick Start

```bash
# 1. Copy environment file
cp .env.example .env
# Edit .env with your values

# 2. Build and start
docker compose up -d --build

# 3. Run tests
docker compose run --rm app python -m pytest tests/ -v

# 4. Check health
docker compose ps
```

## Project Structure

```
├── CLAUDE.md              ← Claude Code instructions for this project
├── README.md              ← This file
├── docker-compose.yml     ← Service orchestration
├── Dockerfile             ← Application container
├── requirements.txt       ← Python dependencies
├── .env.example           ← Environment template
├── src/                   ← Application source code
│   ├── __init__.py
│   └── main.py
├── tests/                 ← Test suite
│   └── __init__.py
├── database/              ← Database files
│   └── init/              ← Init SQL scripts (01-*.sql, 02-*.sql, ...)
└── docs/                  ← Documentation
```

## Development

All code runs inside Docker. Never run directly on the host.

```bash
# Rebuild after code changes
docker compose up -d --build

# Run a specific test
docker compose run --rm app python -m pytest tests/test_something.py -v

# Shell into the container
docker compose exec app bash

# View database
docker compose exec db psql -U appuser -d appdb
```

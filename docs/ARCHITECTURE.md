# Field and Fen - Technical Architecture

## Overview

Automated pipeline for wildlife/nature art e-commerce. Transforms raw artwork into print-ready products published across multiple e-commerce and social platforms.

## Stack

| Layer | Technology |
|-------|------------|
| Backend API | Python, FastAPI |
| Message Queue | RabbitMQ |
| Database | SQLite (persistent state) |
| Frontend | React, Vite, Tailwind CSS |
| Containerization | Docker, Docker Compose |
| Hosting | Cloud (Heroku/Render/Fly.io) or self-hosted with Tailscale |
| Cloud Compute | Replicate (image upscaling) |

## Architecture Diagram

```
External Triggers
(Zapier webhooks, Google Drive, manual)
              |
              | JWT-signed webhook
              v
+--------------------------------------------------+
|  FastAPI Application                              |
|  +--------------------------------------------+  |
|  | /graphql         - Internal Graph API      |  |
|  | /webhooks/ingest - File notifications      |  |
|  | /health          - Liveness check          |  |
|  +--------------------------------------------+  |
+--------------------------------------------------+
              |
              v
+--------------------------------------------------+
|  RabbitMQ                                         |
|  +--------------------------------------------+  |
|  | Queues:                                    |  |
|  |   - file.ingest                            |  |
|  |   - image.upscale                          |  |
|  |   - image.update                           |  |
|  |   - metadata.generate                      |  |
|  |   - listing.create                         |  |
|  |   - social.publish                         |  |
|  |   - dead-letter (failed jobs)              |  |
|  +--------------------------------------------+  |
+--------------------------------------------------+
              |
              v
+--------------------------------------------------+
|  Worker Process                                   |
|  +--------------------------------------------+  |
|  | - Consumes from RabbitMQ queues            |  |
|  | - Executes commands (CRUD)                 |  |
|  | - Publishes events                         |  |
|  | - Retry with exponential backoff           |  |
|  +--------------------------------------------+  |
+--------------------------------------------------+
              |
    +---------+---------+---------+
    |         |         |         |
    v         v         v         v
+-------+ +-------+ +-------+ +-------+
| Cloud | | E-com | | Social| | SQLite|
| APIs  | | APIs  | | APIs  | | State |
+-------+ +-------+ +-------+ +-------+
Replicate  Shopify   Instagram  Files
           Etsy      TikTok     Jobs
           eBay      Pinterest  Hashes
           Amazon    Facebook   API Usage

+--------------------------------------------------+
|  React + Vite Frontend                            |
|  +--------------------------------------------+  |
|  | - Dashboard (job status, API usage, errors)|  |
|  | - Review queue (approve/edit metadata)     |  |
|  | - Settings (retry config, API keys)        |  |
|  +--------------------------------------------+  |
+--------------------------------------------------+
```

## Commands

CRUD operations following the command pattern:

| Command | Description |
|---------|-------------|
| CreateFileRecord | Log incoming file to database |
| UpscaleImage | Send to Replicate, store result |
| UpdateImage | Re-process/regenerate image |
| GenerateMetadata | AI analysis, write JSON sidecar |
| CreateListing | Push product to e-commerce platform |
| UpdateListing | Modify existing listing |
| PublishSocial | Post to social media platform |
| DeleteFile | Cleanup/archive file record |

## Events (Pub/Sub)

| Event | Trigger |
|-------|---------|
| file.received | New file ingested |
| file.processed | Upscaling complete |
| file.updated | Image regenerated |
| file.approved | Human review passed |
| listing.created | Product live on platform |
| listing.updated | Listing modified |
| social.posted | Social media post live |
| job.failed | Command execution failed |

## Database Schema (SQLite)

```sql
-- File records
CREATE TABLE files (
    id TEXT PRIMARY KEY,
    original_filename TEXT NOT NULL,
    processed_filename TEXT,
    species TEXT,
    status TEXT NOT NULL,
    metadata_json TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Job tracking
CREATE TABLE jobs (
    id TEXT PRIMARY KEY,
    file_id TEXT REFERENCES files(id),
    command_type TEXT NOT NULL,
    status TEXT NOT NULL,
    attempts INTEGER DEFAULT 0,
    last_error TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    completed_at TIMESTAMP
);

-- Idempotency tracking
CREATE TABLE processed_hashes (
    hash TEXT PRIMARY KEY,
    file_id TEXT REFERENCES files(id),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- API usage tracking
CREATE TABLE api_usage (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    service TEXT NOT NULL,
    endpoint TEXT,
    request_count INTEGER DEFAULT 1,
    recorded_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

## Security

- Webhooks authenticated via JWT signature validation
- Idempotency via SHA256 hash of payload
- API keys stored in environment variables (not in code)
- Tailscale for secure self-hosted networking

## Retry Strategy

Exponential backoff with jitter:

```
delay = base_delay * (2 ^ attempt) + random_jitter
```

Configuration:
- base_delay: 1 second (configurable)
- max_retries: 5 (configurable)
- jitter: 0-500ms random

Failed jobs after max retries move to dead-letter queue for manual review.

## External Services

| Service | Purpose | Auth Type |
|---------|---------|-----------|
| Google Drive | File storage | Service Account |
| Replicate | Image upscaling | API Token |
| Shopify | Primary storefront | Admin API Key |
| Printful | Print fulfillment | API Key |
| Etsy | Marketplace | OAuth 2.0 (PKCE) |
| eBay | Marketplace | OAuth + App Tokens |
| Meta Business Suite | Instagram/Facebook | Long-lived Token |
| TikTok | Video posts | OAuth 2.0 |
| Pinterest | Pins | OAuth 2.0 |

## Docker Compose Structure

```yaml
services:
  api:
    build: ./backend
    ports:
      - "8000:8000"
    depends_on:
      - rabbitmq
      - db
    environment:
      - DATABASE_URL=sqlite:///data/fieldfen.db
      - RABBITMQ_URL=amqp://rabbitmq:5672

  worker:
    build: ./backend
    command: python worker.py
    depends_on:
      - rabbitmq
      - db

  rabbitmq:
    image: rabbitmq:3-management
    ports:
      - "5672:5672"
      - "15672:15672"

  frontend:
    build: ./frontend
    ports:
      - "3000:3000"
```

## Directory Structure

```
field-and-fen/
├── backend/
│   ├── app/
│   │   ├── api/
│   │   │   ├── graphql/
│   │   │   └── webhooks/
│   │   ├── commands/
│   │   ├── events/
│   │   ├── models/
│   │   ├── repositories/
│   │   ├── services/
│   │   └── workers/
│   ├── tests/
│   ├── Dockerfile
│   └── requirements.txt
├── frontend/
│   ├── src/
│   │   ├── components/
│   │   ├── pages/
│   │   ├── hooks/
│   │   └── utils/
│   ├── Dockerfile
│   └── package.json
├── docker-compose.yml
├── .env.example
└── README.md
```

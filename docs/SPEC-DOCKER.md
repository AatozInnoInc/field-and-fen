# Docker & Infrastructure Specification

## Overview

Containerized application stack for Field and Fen automation platform. Designed for local development and cloud deployment (Fly.io).

---

## Stack Components

| Service | Image | Purpose |
|---------|-------|---------|
| api | Custom (Python FastAPI) | REST + GraphQL API |
| worker | Custom (Python) | Background job processor |
| rabbitmq | rabbitmq:3-management | Message queue |
| frontend | Custom (React + Vite) | Admin dashboard |

**Note:** SQLite database is file-based, no separate container needed. Shared via volume mount.

---

## Directory Structure

```
field-and-fen/
├── backend/
│   ├── app/
│   ├── tests/
│   ├── Dockerfile
│   ├── requirements.txt
│   └── pyproject.toml
├── frontend/
│   ├── src/
│   ├── public/
│   ├── Dockerfile
│   ├── package.json
│   └── vite.config.ts
├── docker-compose.yml
├── docker-compose.prod.yml
├── .env.example
├── .dockerignore
└── README.md
```

---

## Backend Dockerfile

**File:** `backend/Dockerfile`

```dockerfile
# Multi-stage build for smaller image size

# Stage 1: Build dependencies
FROM python:3.11-slim AS builder

WORKDIR /build

# Install build dependencies
RUN apt-get update && apt-get install -y --no-install-recommends \
    gcc \
    g++ \
    && rm -rf /var/lib/apt/lists/*

# Copy requirements and install
COPY requirements.txt .
RUN pip install --user --no-cache-dir -r requirements.txt

# Stage 2: Runtime
FROM python:3.11-slim

WORKDIR /app

# Copy installed packages from builder
COPY --from=builder /root/.local /root/.local

# Add local bin to PATH
ENV PATH=/root/.local/bin:$PATH

# Copy application code
COPY app/ ./app/
COPY alembic/ ./alembic/
COPY alembic.ini .

# Create data directory
RUN mkdir -p /app/data

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=40s --retries=3 \
    CMD python -c "import requests; requests.get('http://localhost:8000/health')"

# Default command (can be overridden in docker-compose)
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

**Why Multi-Stage?**
- Reduces final image size (~200MB vs ~600MB)
- Removes build tools from production image
- Faster deployment and scaling

---

## Frontend Dockerfile

**File:** `frontend/Dockerfile`

```dockerfile
# Multi-stage build

# Stage 1: Build
FROM node:20-alpine AS builder

WORKDIR /build

# Copy package files
COPY package.json package-lock.json ./
RUN npm ci

# Copy source
COPY . .

# Build production bundle
RUN npm run build

# Stage 2: Serve with nginx
FROM nginx:alpine

# Copy built files
COPY --from=builder /build/dist /usr/share/nginx/html

# Copy nginx config (if custom needed)
COPY nginx.conf /etc/nginx/nginx.conf

# Expose port
EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

**nginx.conf (optional, for SPA routing):**
```nginx
server {
    listen 80;
    root /usr/share/nginx/html;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }

    # Cache static assets
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }
}
```

---

## docker-compose.yml (Development)

```yaml
version: '3.8'

services:
  api:
    build:
      context: ./backend
      dockerfile: Dockerfile
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URL=sqlite:////app/data/fieldfen.db
      - RABBITMQ_URL=amqp://guest:guest@rabbitmq:5672/
      - DEBUG=true
    env_file:
      - .env
    volumes:
      - ./backend/app:/app/app  # Hot reload
      - ./data:/app/data        # Persistent DB
    depends_on:
      rabbitmq:
        condition: service_healthy
    command: uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload
    networks:
      - fieldfen

  worker:
    build:
      context: ./backend
      dockerfile: Dockerfile
    environment:
      - DATABASE_URL=sqlite:////app/data/fieldfen.db
      - RABBITMQ_URL=amqp://guest:guest@rabbitmq:5672/
    env_file:
      - .env
    volumes:
      - ./backend/app:/app/app
      - ./data:/app/data
    depends_on:
      rabbitmq:
        condition: service_healthy
      api:
        condition: service_started
    command: python -m app.worker
    restart: unless-stopped
    networks:
      - fieldfen

  rabbitmq:
    image: rabbitmq:3.13-management-alpine
    ports:
      - "5672:5672"   # AMQP
      - "15672:15672" # Management UI
    environment:
      - RABBITMQ_DEFAULT_USER=guest
      - RABBITMQ_DEFAULT_PASS=guest
    volumes:
      - rabbitmq_data:/var/lib/rabbitmq
    healthcheck:
      test: rabbitmq-diagnostics -q ping
      interval: 10s
      timeout: 10s
      retries: 5
    networks:
      - fieldfen

  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile
    ports:
      - "3000:80"
    depends_on:
      - api
    networks:
      - fieldfen

volumes:
  rabbitmq_data:
    driver: local

networks:
  fieldfen:
    driver: bridge
```

**Key Points:**
- `depends_on` with health checks ensures services start in order
- Hot reload in development (volume mount + --reload flag)
- Shared network for inter-service communication
- Persistent volumes for RabbitMQ and SQLite

---

## docker-compose.prod.yml (Production Overrides)

Use this for production deployments (Fly.io, Render, etc.).

```yaml
version: '3.8'

services:
  api:
    environment:
      - DEBUG=false
    command: uvicorn app.main:app --host 0.0.0.0 --port 8000 --workers 4
    volumes:
      - ./data:/app/data  # Remove hot reload mount
    restart: unless-stopped

  worker:
    deploy:
      replicas: 2  # Scale workers
    restart: unless-stopped

  rabbitmq:
    environment:
      - RABBITMQ_DEFAULT_USER=${RABBITMQ_USER}
      - RABBITMQ_DEFAULT_PASS=${RABBITMQ_PASS}
    restart: unless-stopped

  frontend:
    restart: unless-stopped
```

**Usage:**
```bash
docker-compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

---

## .env.example

```bash
# Application
DEBUG=false

# Database
DATABASE_URL=sqlite:////app/data/fieldfen.db

# RabbitMQ
RABBITMQ_URL=amqp://guest:guest@rabbitmq:5672/
RABBITMQ_USER=guest
RABBITMQ_PASS=guest

# JWT
JWT_SECRET=your-secret-key-here-change-in-production

# External APIs
REPLICATE_API_TOKEN=r8_...
SHOPIFY_API_KEY=...
SHOPIFY_API_SECRET=...
SHOPIFY_STORE_URL=fieldandfen.myshopify.com
GOOGLE_DRIVE_CREDENTIALS_PATH=/app/credentials/google-oauth.json

# Retry Config
RETRY_MAX_ATTEMPTS=5
RETRY_BASE_DELAY_SECONDS=1
```

**Security Notes:**
- Never commit `.env` to Git
- Use secrets management in production (Fly.io secrets, Render env vars)
- Rotate JWT_SECRET regularly

---

## .dockerignore

```
# Python
__pycache__/
*.py[cod]
*$py.class
*.so
.Python
env/
venv/
.venv
pip-log.txt
pip-delete-this-directory.txt

# Node
node_modules/
npm-debug.log
yarn-error.log

# IDE
.vscode/
.idea/
*.swp
*.swo

# Testing
.pytest_cache/
.coverage
htmlcov/

# Git
.git/
.gitignore

# Docs
docs/
README.md

# Data
data/
*.db
*.sqlite

# Env
.env
.env.local
.env.*.local
```

---

## Development Workflow

### First-Time Setup

```bash
# Clone repo
git clone git@github.com:AatozInnoInc/field-and-fen.git
cd field-and-fen

# Copy environment template
cp .env.example .env
# Edit .env with your credentials

# Build and start services
docker-compose up --build

# Run database migrations (in another terminal)
docker-compose exec api alembic upgrade head

# Access services:
# - API: http://localhost:8000
# - GraphQL Playground: http://localhost:8000/graphql
# - RabbitMQ Management: http://localhost:15672 (guest/guest)
# - Frontend: http://localhost:3000
```

### Daily Development

```bash
# Start services
docker-compose up

# View logs
docker-compose logs -f api
docker-compose logs -f worker

# Run tests
docker-compose exec api pytest

# Stop services
docker-compose down

# Reset everything (including volumes)
docker-compose down -v
```

### Database Migrations

```bash
# Create migration
docker-compose exec api alembic revision --autogenerate -m "Add new column"

# Apply migrations
docker-compose exec api alembic upgrade head

# Rollback one migration
docker-compose exec api alembic downgrade -1
```

---

## Production Deployment: Fly.io

### fly.toml (API Service)

```toml
app = "fieldfen-api"
primary_region = "iad"  # US East

[build]
  dockerfile = "backend/Dockerfile"

[env]
  PORT = "8000"
  DATABASE_URL = "/data/fieldfen.db"

[[mounts]]
  source = "fieldfen_data"
  destination = "/app/data"

[[services]]
  internal_port = 8000
  protocol = "tcp"

  [[services.ports]]
    handlers = ["http"]
    port = 80

  [[services.ports]]
    handlers = ["tls", "http"]
    port = 443

  [[services.http_checks]]
    interval = 10000
    timeout = 2000
    grace_period = "5s"
    method = "get"
    path = "/health"

[processes]
  api = "uvicorn app.main:app --host 0.0.0.0 --port 8000 --workers 2"
  worker = "python -m app.worker"
```

### fly.toml (RabbitMQ Service)

```toml
app = "fieldfen-rabbitmq"
primary_region = "iad"

[build]
  image = "rabbitmq:3.13-management-alpine"

[[mounts]]
  source = "rabbitmq_data"
  destination = "/var/lib/rabbitmq"

[[services]]
  internal_port = 5672
  protocol = "tcp"

[[services]]
  internal_port = 15672
  protocol = "tcp"

  [[services.ports]]
    handlers = ["http"]
    port = 80
```

### Deployment Commands

```bash
# Install flyctl
curl -L https://fly.io/install.sh | sh

# Login
flyctl auth login

# Create apps
flyctl apps create fieldfen-api
flyctl apps create fieldfen-rabbitmq

# Create volumes
flyctl volumes create fieldfen_data --region iad --size 10
flyctl volumes create rabbitmq_data --region iad --size 5

# Set secrets
flyctl secrets set JWT_SECRET=your-secret -a fieldfen-api
flyctl secrets set REPLICATE_API_TOKEN=... -a fieldfen-api
flyctl secrets set SHOPIFY_API_KEY=... -a fieldfen-api

# Deploy
flyctl deploy -a fieldfen-api
flyctl deploy -a fieldfen-rabbitmq

# Check status
flyctl status -a fieldfen-api
flyctl logs -a fieldfen-api
```

**Estimated Cost (Fly.io):**
- 1x shared-cpu-1x VM (API + Worker): ~$3/month
- 1x shared-cpu-1x VM (RabbitMQ): ~$2/month
- 15GB storage: ~$1.50/month
- **Total: ~$6.50/month**

---

## Production Deployment: Render

### render.yaml (Blueprint)

```yaml
services:
  - type: web
    name: fieldfen-api
    env: docker
    dockerfilePath: ./backend/Dockerfile
    envVars:
      - key: DATABASE_URL
        value: sqlite:////data/fieldfen.db
      - key: RABBITMQ_URL
        fromService:
          type: pserv
          name: rabbitmq
          property: connectionString
      - key: JWT_SECRET
        generateValue: true
      - key: REPLICATE_API_TOKEN
        sync: false  # Set manually
    healthCheckPath: /health
    disk:
      name: fieldfen-data
      mountPath: /data
      sizeGB: 10

  - type: worker
    name: fieldfen-worker
    env: docker
    dockerfilePath: ./backend/Dockerfile
    dockerCommand: python -m app.worker
    envVars:
      - key: DATABASE_URL
        value: sqlite:////data/fieldfen.db
      - key: RABBITMQ_URL
        fromService:
          type: pserv
          name: rabbitmq
          property: connectionString
    disk:
      name: fieldfen-data
      mountPath: /data
      sizeGB: 10

  - type: pserv
    name: rabbitmq
    env: docker
    dockerImage: rabbitmq:3.13-management-alpine
    disk:
      name: rabbitmq-data
      mountPath: /var/lib/rabbitmq
      sizeGB: 5
```

**Estimated Cost (Render):**
- Web service: $7/month (starter)
- Worker: $7/month (starter)
- Private service (RabbitMQ): $7/month
- **Total: ~$21/month**

**Recommendation:** Use Fly.io for better cost/performance ratio.

---

## Monitoring & Debugging

### View Logs

```bash
# Docker Compose
docker-compose logs -f api
docker-compose logs -f worker

# Fly.io
flyctl logs -a fieldfen-api

# Render
render logs -s fieldfen-api
```

### RabbitMQ Management UI

Access at `http://localhost:15672` (dev) or `http://your-rabbitmq-app.fly.dev` (prod)

**Useful Metrics:**
- Queue depths (should be near zero under normal load)
- Message rates (messages/second)
- Consumer count (should match worker replicas)
- DLQ message count (alert if >10)

### Database Shell

```bash
# Docker Compose
docker-compose exec api sqlite3 /app/data/fieldfen.db

# Fly.io
flyctl ssh console -a fieldfen-api
sqlite3 /app/data/fieldfen.db
```

**Useful Queries:**
```sql
-- Job status summary
SELECT status, COUNT(*) FROM jobs GROUP BY status;

-- Recent failures
SELECT id, command_type, last_error, created_at 
FROM jobs 
WHERE status = 'failed' 
ORDER BY created_at DESC 
LIMIT 10;

-- API usage by service
SELECT service, SUM(request_count), DATE(recorded_at) 
FROM api_usage 
GROUP BY service, DATE(recorded_at);
```

---

## Backup & Recovery

### Automated Backups

**Cron Job (on host or in container):**
```bash
#!/bin/bash
# backup.sh

DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR=/backups
DB_PATH=/app/data/fieldfen.db

# Create backup
cp $DB_PATH $BACKUP_DIR/fieldfen_$DATE.db

# Compress
gzip $BACKUP_DIR/fieldfen_$DATE.db

# Clean up old backups (keep 30 days)
find $BACKUP_DIR -name "fieldfen_*.db.gz" -mtime +30 -delete

echo "Backup completed: fieldfen_$DATE.db.gz"
```

**Add to crontab:**
```
0 2 * * * /path/to/backup.sh
```

### Restore from Backup

```bash
# Stop services
docker-compose down

# Restore database
gunzip -c /backups/fieldfen_20260207_020000.db.gz > ./data/fieldfen.db

# Start services
docker-compose up -d
```

---

## Performance Tuning

### Worker Scaling

Increase replicas in production:
```yaml
  worker:
    deploy:
      replicas: 5
```

**How many workers?**
- Rule of thumb: 1 worker per CPU core
- Monitor queue depth: if >100 messages, add workers
- Monitor CPU/memory: don't exceed 80% utilization

### Database Optimization

SQLite is fast for reads, slower for writes. If job throughput exceeds 1000/minute, consider PostgreSQL.

**Migration to PostgreSQL (future):**
1. Export SQLite data: `sqlite3 fieldfen.db .dump > dump.sql`
2. Import to Postgres: `psql < dump.sql`
3. Update `DATABASE_URL` in .env
4. No code changes needed (SQLAlchemy abstracts DB)

---

## Troubleshooting

### Service Won't Start

```bash
# Check logs
docker-compose logs api

# Common issues:
# - Port already in use: Change in docker-compose.yml
# - Missing .env file: Copy from .env.example
# - Database locked: Stop all services, remove data/fieldfen.db-wal
```

### Worker Not Processing Jobs

```bash
# Check worker logs
docker-compose logs worker

# Check RabbitMQ
open http://localhost:15672

# Verify queues exist and have consumers
# Common issues:
# - Worker crashed: Check logs for Python errors
# - RabbitMQ connection lost: Restart worker
# - Commands not registered: Check COMMAND_REGISTRY in worker.py
```

### API Timeouts

```bash
# Increase workers in production
# In fly.toml:
[processes]
  api = "uvicorn app.main:app --workers 4"
```

---

## Next Steps

After implementing this spec:
1. Create Dockerfiles for backend and frontend
2. Set up docker-compose for local development
3. Test end-to-end: API → Queue → Worker
4. Deploy to Fly.io or Render
5. Set up monitoring and alerts

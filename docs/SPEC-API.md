# API Specification

## Overview

FastAPI application with two interfaces:
1. **REST API** - Webhooks, health checks, simple CRUD
2. **GraphQL** - Internal queries, complex data fetching

**Why both?**
- REST for external integrations (Zapier, simple webhooks)
- GraphQL for frontend (flexible queries, reduce over-fetching)

---

## Tech Stack

- **Framework:** FastAPI 0.109+
- **GraphQL:** Strawberry GraphQL
- **ORM:** SQLAlchemy 2.0+
- **Validation:** Pydantic v2
- **Auth:** JWT (HS256, env secret)
- **CORS:** Allow frontend origin (configurable)

---

## Project Structure

```
backend/
├── app/
│   ├── __init__.py
│   ├── main.py                 # FastAPI app + startup
│   ├── config.py               # Environment config (Pydantic Settings)
│   ├── database.py             # SQLAlchemy setup
│   ├── dependencies.py         # DI (db sessions, auth)
│   ├── models/                 # SQLAlchemy models
│   │   ├── __init__.py
│   │   ├── file.py
│   │   ├── job.py
│   │   ├── listing.py
│   │   └── ...
│   ├── schemas/                # Pydantic schemas (request/response)
│   │   ├── __init__.py
│   │   ├── file.py
│   │   ├── job.py
│   │   └── ...
│   ├── api/                    # REST endpoints
│   │   ├── __init__.py
│   │   ├── webhooks.py
│   │   ├── health.py
│   │   └── ...
│   ├── graphql/                # GraphQL schema + resolvers
│   │   ├── __init__.py
│   │   ├── schema.py           # Root schema
│   │   ├── queries.py
│   │   ├── mutations.py
│   │   └── types.py
│   ├── commands/               # Command pattern implementations
│   │   ├── __init__.py
│   │   ├── base.py             # BaseCommand abstract class
│   │   ├── create_file_record.py
│   │   ├── upscale_image.py
│   │   └── ...
│   ├── services/               # External service clients
│   │   ├── __init__.py
│   │   ├── replicate.py
│   │   ├── shopify.py
│   │   ├── google_drive.py
│   │   └── ...
│   └── utils/                  # Helpers
│       ├── __init__.py
│       ├── retry.py            # Exponential backoff logic
│       ├── hash.py             # SHA256 hashing
│       └── ...
├── tests/
│   ├── __init__.py
│   ├── conftest.py             # Pytest fixtures
│   ├── test_api/
│   ├── test_commands/
│   └── test_services/
├── Dockerfile
├── requirements.txt
├── pyproject.toml              # Optional: use Poetry for deps
└── alembic/                    # Database migrations
    └── versions/
```

---

## Configuration (config.py)

Use Pydantic Settings for environment-based config.

```python
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    # App
    app_name: str = "Field and Fen API"
    debug: bool = False
    
    # Database
    database_url: str = "sqlite:///./data/fieldfen.db"
    
    # RabbitMQ
    rabbitmq_url: str = "amqp://guest:guest@rabbitmq:5672/"
    
    # JWT
    jwt_secret: str  # Required, no default (security)
    jwt_algorithm: str = "HS256"
    jwt_expiration_hours: int = 24
    
    # External APIs
    replicate_api_token: str = ""
    shopify_api_key: str = ""
    shopify_api_secret: str = ""
    shopify_store_url: str = ""
    google_drive_credentials_path: str = "./credentials/google-oauth.json"
    
    # Retry Config
    retry_max_attempts: int = 5
    retry_base_delay_seconds: int = 1
    
    class Config:
        env_file = ".env"

settings = Settings()
```

**Environment Variables (.env):**
```bash
DATABASE_URL=sqlite:///./data/fieldfen.db
RABBITMQ_URL=amqp://guest:guest@rabbitmq:5672/
JWT_SECRET=your-secret-key-here
REPLICATE_API_TOKEN=r8_...
SHOPIFY_API_KEY=...
SHOPIFY_API_SECRET=...
SHOPIFY_STORE_URL=fieldandfen.myshopify.com
```

---

## REST Endpoints

### Health Check

**GET /health**

**Purpose:** Liveness check for orchestrators (Docker, Fly.io)

**Response:**
```json
{
  "status": "healthy",
  "timestamp": "2026-02-07T21:00:00Z",
  "database": "connected",
  "rabbitmq": "connected"
}
```

**Status Codes:**
- 200: All systems operational
- 503: Service degraded (DB or RabbitMQ down)

**Implementation:**
```python
from fastapi import APIRouter, status
from sqlalchemy import text
from app.database import SessionLocal
from app.dependencies import get_rabbitmq_connection

router = APIRouter()

@router.get("/health")
async def health_check():
    db_ok = False
    rabbitmq_ok = False
    
    # Check database
    try:
        db = SessionLocal()
        db.execute(text("SELECT 1"))
        db.close()
        db_ok = True
    except Exception:
        pass
    
    # Check RabbitMQ
    try:
        conn = get_rabbitmq_connection()
        conn.close()
        rabbitmq_ok = True
    except Exception:
        pass
    
    if db_ok and rabbitmq_ok:
        return {
            "status": "healthy",
            "timestamp": datetime.utcnow().isoformat() + "Z",
            "database": "connected",
            "rabbitmq": "connected"
        }
    else:
        raise HTTPException(
            status_code=status.HTTP_503_SERVICE_UNAVAILABLE,
            detail="Service degraded"
        )
```

---

### Webhook: File Ingested

**POST /webhooks/file-ingested**

**Purpose:** Zapier calls this when a new file is added to Google Drive.

**Authentication:** JWT in `Authorization: Bearer <token>` header.

**Request Body:**
```json
{
  "file_id": "1abc123xyz",
  "filename": "whitetail-buck-morning.jpg",
  "species": "whitetail-deer",
  "drive_url": "https://drive.google.com/file/d/1abc123xyz/view"
}
```

**Response:**
```json
{
  "job_id": "uuid-v4",
  "status": "queued"
}
```

**Status Codes:**
- 201: Job created
- 400: Invalid payload
- 401: Unauthorized (bad/missing JWT)
- 409: Duplicate (hash already processed)

**Implementation Notes:**
1. Hash the incoming payload (SHA256)
2. Check `processed_hashes` table
3. If duplicate, return 409
4. Create a `CreateFileRecord` command
5. Publish to RabbitMQ `file.ingest` queue
6. Insert job record in `pending` status
7. Insert hash to `processed_hashes`
8. Return job ID

**Code Sketch:**
```python
from fastapi import APIRouter, Depends, HTTPException, status
from app.schemas.webhook import FileIngestedPayload
from app.dependencies import get_db, verify_jwt
from app.utils.hash import sha256_hash
from app.commands.create_file_record import CreateFileRecordCommand
import uuid

router = APIRouter()

@router.post("/webhooks/file-ingested", status_code=status.HTTP_201_CREATED)
async def file_ingested_webhook(
    payload: FileIngestedPayload,
    db: Session = Depends(get_db),
    user: dict = Depends(verify_jwt)
):
    # Hash payload for idempotency
    payload_hash = sha256_hash(payload.json())
    
    # Check if already processed
    existing = db.query(ProcessedHash).filter_by(hash=payload_hash).first()
    if existing:
        raise HTTPException(
            status_code=status.HTTP_409_CONFLICT,
            detail="Duplicate payload"
        )
    
    # Create job
    job_id = str(uuid.uuid4())
    command = CreateFileRecordCommand(
        file_id=payload.file_id,
        filename=payload.filename,
        species=payload.species
    )
    
    # Publish to queue (implementation in worker section)
    publish_command(command, queue="file.ingest")
    
    # Log job
    job = Job(id=job_id, command_type="CreateFileRecord", status="pending")
    db.add(job)
    
    # Log hash
    db.add(ProcessedHash(hash=payload_hash))
    db.commit()
    
    return {"job_id": job_id, "status": "queued"}
```

---

### Manual Trigger: Reprocess File

**POST /api/files/{file_id}/reprocess**

**Purpose:** Manual trigger to re-run AI processing on a file.

**Authentication:** JWT required

**Response:**
```json
{
  "job_id": "uuid-v4",
  "status": "queued"
}
```

**Implementation:**
Create an `UpdateImage` command, publish to queue.

---

## GraphQL Schema

**Endpoint:** `/graphql`

**Playground:** Enabled in debug mode (`/graphql` in browser)

### Types

**File:**
```graphql
type File {
  id: ID!
  originalFilename: String!
  processedFilename: String
  species: String
  status: FileStatus!
  driveFileId: String
  metadata: FileMetadata
  width: Int
  height: Int
  aspectRatio: String
  createdAt: DateTime!
  updatedAt: DateTime!
  listings: [Listing!]!
  socialPosts: [SocialPost!]!
}

enum FileStatus {
  INGESTED
  PROCESSING
  PROCESSED
  APPROVED
  PUBLISHED
  FAILED
}

type FileMetadata {
  title: String
  description: String
  hashtags: [String!]
  captions: Captions
}

type Captions {
  instagram: String
  tiktok: String
  pinterest: String
}
```

**Job:**
```graphql
type Job {
  id: ID!
  fileId: ID
  commandType: String!
  status: JobStatus!
  attempts: Int!
  lastError: String
  createdAt: DateTime!
  completedAt: DateTime
}

enum JobStatus {
  PENDING
  RUNNING
  COMPLETED
  FAILED
}
```

**Listing:**
```graphql
type Listing {
  id: ID!
  fileId: ID!
  platform: String!
  platformListingId: String!
  listingUrl: String
  status: String!
  createdAt: DateTime!
  updatedAt: DateTime!
}
```

### Queries

**Get Files (with filters):**
```graphql
type Query {
  files(
    status: FileStatus
    species: String
    limit: Int = 50
    offset: Int = 0
  ): [File!]!
  
  file(id: ID!): File
  
  jobs(
    status: JobStatus
    commandType: String
    limit: Int = 50
    offset: Int = 0
  ): [Job!]!
  
  job(id: ID!): Job
  
  apiUsageStats(
    service: String
    startDate: DateTime
    endDate: DateTime
  ): [ApiUsageStat!]!
}

type ApiUsageStat {
  service: String!
  endpoint: String
  totalRequests: Int!
  date: DateTime!
}
```

### Mutations

**Approve File:**
```graphql
type Mutation {
  approveFile(id: ID!): File!
  
  reprocessFile(id: ID!): Job!
  
  deleteFile(id: ID!): Boolean!
}
```

**Implementation Notes:**
- Use Strawberry's decorators (`@strawberry.type`, `@strawberry.field`)
- Resolvers query SQLAlchemy models
- Return Pydantic schemas converted to Strawberry types

---

## Authentication

**JWT Flow:**

1. External service (Zapier) has a pre-shared JWT secret
2. Generate token with claims: `{"sub": "zapier", "exp": <timestamp>}`
3. Include in `Authorization: Bearer <token>` header
4. API validates signature, expiration

**Token Generation (for Zapier):**
```python
import jwt
from datetime import datetime, timedelta
from app.config import settings

def generate_jwt(subject: str, expiration_hours: int = 24) -> str:
    payload = {
        "sub": subject,
        "exp": datetime.utcnow() + timedelta(hours=expiration_hours),
        "iat": datetime.utcnow()
    }
    return jwt.encode(payload, settings.jwt_secret, algorithm=settings.jwt_algorithm)
```

**Token Validation (dependency):**
```python
from fastapi import Depends, HTTPException, status
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
import jwt
from app.config import settings

security = HTTPBearer()

def verify_jwt(credentials: HTTPAuthorizationCredentials = Depends(security)) -> dict:
    try:
        payload = jwt.decode(
            credentials.credentials,
            settings.jwt_secret,
            algorithms=[settings.jwt_algorithm]
        )
        return payload
    except jwt.ExpiredSignatureError:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Token expired"
        )
    except jwt.InvalidTokenError:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Invalid token"
        )
```

---

## Error Handling

**Standard Error Response:**
```json
{
  "detail": "Human-readable error message",
  "error_code": "DUPLICATE_FILE",
  "timestamp": "2026-02-07T21:00:00Z"
}
```

**Custom Exception Handler:**
```python
from fastapi import Request
from fastapi.responses import JSONResponse
from datetime import datetime

@app.exception_handler(Exception)
async def global_exception_handler(request: Request, exc: Exception):
    return JSONResponse(
        status_code=500,
        content={
            "detail": "Internal server error",
            "error_code": "INTERNAL_ERROR",
            "timestamp": datetime.utcnow().isoformat() + "Z"
        }
    )
```

---

## Testing

**Unit Tests (pytest):**

```python
from fastapi.testclient import TestClient
from app.main import app

client = TestClient(app)

def test_health_check():
    response = client.get("/health")
    assert response.status_code == 200
    assert response.json()["status"] == "healthy"

def test_webhook_requires_auth():
    response = client.post("/webhooks/file-ingested", json={})
    assert response.status_code == 401
```

**Integration Tests:**
- Use TestClient with in-memory SQLite
- Mock external services (Replicate, Shopify) with responses library
- Test full webhook → command → queue flow

---

## Deployment

**Docker:**
```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app/ ./app/

CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

**docker-compose.yml:**
```yaml
services:
  api:
    build: ./backend
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URL=sqlite:///./data/fieldfen.db
      - RABBITMQ_URL=amqp://rabbitmq:5672
    env_file:
      - .env
    volumes:
      - ./data:/app/data
    depends_on:
      - rabbitmq
```

---

## Next Steps

After implementing this spec:
1. Write worker process spec (queue consumer)
2. Implement commands (CreateFileRecord, UpscaleImage, etc.)
3. Add external service clients (Replicate, Shopify)
4. Build frontend to consume GraphQL API

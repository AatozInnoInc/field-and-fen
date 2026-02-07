# Database Specification

## Overview

SQLite database for persistent state tracking. Single file, no external dependencies, sufficient for this scale (expected <10k products, <1k jobs/day).

## Schema

### Table: files

Tracks all files in the pipeline.

```sql
CREATE TABLE files (
    id TEXT PRIMARY KEY,                    -- UUID v4
    original_filename TEXT NOT NULL,        -- As uploaded/ingested
    processed_filename TEXT,                -- After processing (optional)
    species TEXT,                           -- Category (whitetail-deer, fox, etc)
    status TEXT NOT NULL,                   -- Enum: ingested, processing, processed, approved, published, failed
    drive_file_id TEXT,                     -- Google Drive file ID (for re-download)
    metadata_json TEXT,                     -- JSON blob: {title, description, hashtags, etc}
    width INTEGER,                          -- Image width in pixels
    height INTEGER,                         -- Image height in pixels
    aspect_ratio TEXT,                      -- '3:2', '2:3', '4:5', etc
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_files_status ON files(status);
CREATE INDEX idx_files_species ON files(species);
CREATE INDEX idx_files_drive_file_id ON files(drive_file_id);
```

**Field Notes:**
- `id`: Use Python `uuid.uuid4()` for generation
- `status`: Transitions: `ingested → processing → processed → approved → published` (or `failed` at any point)
- `metadata_json`: Store as JSON string. Example:
  ```json
  {
    "title": "Whitetail Buck in Morning Mist",
    "description": "A majestic whitetail buck emerges...",
    "hashtags": ["wildlifeart", "whitetaildeer", "naturephotography"],
    "captions": {
      "instagram": "...",
      "tiktok": "...",
      "pinterest": "..."
    }
  }
  ```
- `aspect_ratio`: Computed from width/height. Used for variant selection.

---

### Table: jobs

Tracks asynchronous work.

```sql
CREATE TABLE jobs (
    id TEXT PRIMARY KEY,                    -- UUID v4
    file_id TEXT,                           -- Foreign key to files(id), nullable for non-file jobs
    command_type TEXT NOT NULL,             -- Enum: CreateFileRecord, UpscaleImage, GenerateMetadata, etc
    status TEXT NOT NULL,                   -- Enum: pending, running, completed, failed
    attempts INTEGER DEFAULT 0,             -- Retry counter
    last_error TEXT,                        -- Error message from last failure
    payload_json TEXT,                      -- Command-specific data (JSON)
    result_json TEXT,                       -- Command result data (JSON)
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    started_at TIMESTAMP,
    completed_at TIMESTAMP,
    FOREIGN KEY (file_id) REFERENCES files(id) ON DELETE CASCADE
);

CREATE INDEX idx_jobs_status ON jobs(status);
CREATE INDEX idx_jobs_file_id ON jobs(file_id);
CREATE INDEX idx_jobs_command_type ON jobs(command_type);
```

**Field Notes:**
- `command_type`: One of: `CreateFileRecord`, `UpscaleImage`, `UpdateImage`, `GenerateMetadata`, `CreateListing`, `UpdateListing`, `PublishSocial`, `DeleteFile`
- `status`: `pending → running → completed` (or `failed`)
- `attempts`: Increment on each retry. Max retries: 5 (configurable)
- `payload_json`: Command-specific. Example for `UpscaleImage`:
  ```json
  {
    "file_id": "abc123",
    "scale_factor": 4,
    "target_dpi": 300
  }
  ```

---

### Table: processed_hashes

Idempotency tracking for webhooks.

```sql
CREATE TABLE processed_hashes (
    hash TEXT PRIMARY KEY,                  -- SHA256 of webhook payload
    file_id TEXT,                           -- Associated file (if applicable)
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (file_id) REFERENCES files(id) ON DELETE SET NULL
);

CREATE INDEX idx_processed_hashes_created_at ON processed_hashes(created_at);
```

**Purpose:**
Prevent duplicate processing when a webhook fires multiple times with the same payload. Hash the incoming JSON payload (SHA256), check if it exists. If yes, skip. If no, process and insert.

**Cleanup:**
Delete hashes older than 24 hours (configurable). Use a periodic cleanup job.

---

### Table: api_usage

Track external API calls for rate limit monitoring.

```sql
CREATE TABLE api_usage (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    service TEXT NOT NULL,                  -- 'shopify', 'etsy', 'replicate', etc
    endpoint TEXT,                          -- API endpoint or action name
    request_count INTEGER DEFAULT 1,        -- Number of requests (for batch tracking)
    rate_limit_remaining INTEGER,           -- From response headers (if available)
    rate_limit_reset_at TIMESTAMP,          -- From response headers (if available)
    recorded_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_api_usage_service ON api_usage(service);
CREATE INDEX idx_api_usage_recorded_at ON api_usage(recorded_at);
```

**Purpose:**
Log every external API call. Use for:
1. Monitoring rate limits
2. Cost tracking (Replicate charges per call)
3. Debugging failed integrations

---

### Table: listings

Track published products across platforms.

```sql
CREATE TABLE listings (
    id TEXT PRIMARY KEY,                    -- UUID v4
    file_id TEXT NOT NULL,                  -- Foreign key to files(id)
    platform TEXT NOT NULL,                 -- 'shopify', 'etsy', 'ebay', 'amazon'
    platform_listing_id TEXT NOT NULL,      -- External ID from platform
    listing_url TEXT,                       -- Direct link to product page
    status TEXT NOT NULL,                   -- 'draft', 'active', 'sold_out', 'delisted'
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (file_id) REFERENCES files(id) ON DELETE CASCADE,
    UNIQUE(platform, platform_listing_id)
);

CREATE INDEX idx_listings_file_id ON listings(file_id);
CREATE INDEX idx_listings_platform ON listings(platform);
```

**Purpose:**
Track where each file is published. Enables:
- Cross-platform listing management
- Deduplication (don't re-publish to same platform)
- Sales tracking (future)

---

### Table: social_posts

Track social media posts.

```sql
CREATE TABLE social_posts (
    id TEXT PRIMARY KEY,                    -- UUID v4
    file_id TEXT NOT NULL,                  -- Foreign key to files(id)
    platform TEXT NOT NULL,                 -- 'instagram', 'tiktok', 'facebook', 'pinterest'
    post_type TEXT,                         -- 'post', 'story', 'reel', 'pin'
    platform_post_id TEXT,                  -- External ID from platform
    post_url TEXT,                          -- Direct link to post
    posted_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (file_id) REFERENCES files(id) ON DELETE CASCADE
);

CREATE INDEX idx_social_posts_file_id ON social_posts(file_id);
CREATE INDEX idx_social_posts_platform ON social_posts(platform);
```

---

## Migrations

Use Alembic (SQLAlchemy migration tool) for versioned schema changes.

**Initial Migration (001_initial_schema.sql):**

Create all tables as defined above. This is the baseline.

**Future Migrations:**

Any schema changes must:
1. Be backward-compatible (add columns with defaults, don't drop)
2. Include rollback logic
3. Be tested on a copy of production data

---

## Connection Details

**Python (SQLAlchemy):**

```python
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

DATABASE_URL = "sqlite:///./data/fieldfen.db"
engine = create_engine(DATABASE_URL, connect_args={"check_same_thread": False})
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)
```

**File Location:**
`./data/fieldfen.db` (relative to app root)

Ensure `./data/` directory exists and is writable. Add to `.gitignore`.

---

## Testing

**Unit Tests:**
- Create in-memory SQLite (`:memory:`) for tests
- Reset schema before each test
- Use fixtures for common data setup

**Example:**
```python
import pytest
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

@pytest.fixture
def db_session():
    engine = create_engine("sqlite:///:memory:")
    # Run schema creation here
    SessionLocal = sessionmaker(bind=engine)
    session = SessionLocal()
    yield session
    session.close()
```

---

## Backups

SQLite is a single file. Backup strategy:
1. Daily cron job: `cp data/fieldfen.db backups/fieldfen-$(date +%Y%m%d).db`
2. Retain last 30 days
3. Weekly backups to cloud storage (S3/GCS)

---

## Performance Notes

SQLite is sufficient for this workload. Expected scale:
- <10k products
- <1k jobs/day
- <100 concurrent reads

If we exceed:
- 10k products → Consider PostgreSQL
- 1k jobs/day → Consider dedicated queue (Redis)
- 100 concurrent reads → Consider read replicas (PostgreSQL)

For now: SQLite is the right tool.

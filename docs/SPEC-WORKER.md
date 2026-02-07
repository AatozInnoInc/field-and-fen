# Worker Process Specification

## Overview

Background worker that consumes jobs from RabbitMQ, executes commands, publishes events, and handles retries.

**Key Responsibilities:**
1. Consume messages from RabbitMQ queues
2. Execute command objects (CRUD operations)
3. Publish events to RabbitMQ for downstream processing
4. Handle failures with exponential backoff retry
5. Log all actions to database and stdout

---

## Architecture

```
RabbitMQ Queue → Worker Process → Command Execution → Event Publishing → RabbitMQ
                      ↓
                  Database (job status, results)
```

**Concurrency Model:**
- Multi-threaded consumer (configurable thread pool size)
- Each thread handles one job at a time
- Safe to run multiple worker instances for horizontal scaling

---

## Queue Structure

### Queues

| Queue Name | Purpose | Dead Letter |
|------------|---------|-------------|
| file.ingest | New file ingestion | file.ingest.dlq |
| image.process | Upscaling + metadata generation | image.process.dlq |
| listing.create | E-commerce publishing | listing.create.dlq |
| social.publish | Social media posting | social.publish.dlq |
| events | Event notifications (logging, webhooks) | events.dlq |

**Dead Letter Queues (DLQ):**
Messages that fail after max retries are moved to DLQ for manual inspection.

---

## Message Format

**Queue Message:**
```json
{
  "job_id": "uuid-v4",
  "command_type": "UpscaleImage",
  "payload": {
    "file_id": "abc123",
    "scale_factor": 4,
    "target_dpi": 300
  },
  "attempt": 1,
  "max_attempts": 5
}
```

**Fields:**
- `job_id`: UUID from jobs table
- `command_type`: Name of command class to execute
- `payload`: Command-specific data (JSON object)
- `attempt`: Current attempt number (1-indexed)
- `max_attempts`: Retry limit (default: 5)

---

## Command Pattern

All commands inherit from `BaseCommand` and implement `execute()`.

### BaseCommand (Abstract)

```python
from abc import ABC, abstractmethod
from sqlalchemy.orm import Session
from typing import Any, Dict

class BaseCommand(ABC):
    def __init__(self, db: Session, payload: Dict[str, Any]):
        self.db = db
        self.payload = payload
    
    @abstractmethod
    def execute(self) -> Dict[str, Any]:
        """
        Execute the command.
        
        Returns:
            Result dictionary (stored in jobs.result_json)
        
        Raises:
            Exception: On failure (triggers retry)
        """
        pass
    
    def on_success(self, result: Dict[str, Any]):
        """Hook called after successful execution."""
        pass
    
    def on_failure(self, error: Exception):
        """Hook called after failure (before retry)."""
        pass
```

### Example: CreateFileRecord

```python
from app.commands.base import BaseCommand
from app.models.file import File
from typing import Dict, Any
import uuid

class CreateFileRecordCommand(BaseCommand):
    def execute(self) -> Dict[str, Any]:
        file_id = self.payload.get("file_id")
        filename = self.payload.get("filename")
        species = self.payload.get("species")
        drive_file_id = self.payload.get("drive_file_id")
        
        # Validate required fields
        if not all([file_id, filename, species]):
            raise ValueError("Missing required fields")
        
        # Check if file already exists
        existing = self.db.query(File).filter_by(id=file_id).first()
        if existing:
            return {"status": "exists", "file_id": file_id}
        
        # Create file record
        file = File(
            id=file_id,
            original_filename=filename,
            species=species,
            drive_file_id=drive_file_id,
            status="ingested"
        )
        self.db.add(file)
        self.db.commit()
        
        return {"status": "created", "file_id": file_id}
    
    def on_success(self, result: Dict[str, Any]):
        # Publish event: file.received
        from app.worker import publish_event
        publish_event("file.received", {
            "file_id": result["file_id"],
            "timestamp": datetime.utcnow().isoformat() + "Z"
        })
```

### Example: UpscaleImage

```python
from app.commands.base import BaseCommand
from app.models.file import File
from app.services.replicate import ReplicateClient
from app.services.google_drive import GoogleDriveClient
from typing import Dict, Any
import tempfile
import os

class UpscaleImageCommand(BaseCommand):
    def execute(self) -> Dict[str, Any]:
        file_id = self.payload.get("file_id")
        scale_factor = self.payload.get("scale_factor", 4)
        
        # Fetch file record
        file = self.db.query(File).filter_by(id=file_id).first()
        if not file:
            raise ValueError(f"File {file_id} not found")
        
        # Update status
        file.status = "processing"
        self.db.commit()
        
        # Download from Google Drive
        drive_client = GoogleDriveClient()
        with tempfile.NamedTemporaryFile(delete=False, suffix=".jpg") as tmp:
            drive_client.download_file(file.drive_file_id, tmp.name)
            input_path = tmp.name
        
        try:
            # Upscale with Replicate
            replicate_client = ReplicateClient()
            output_url = replicate_client.upscale_image(
                input_path=input_path,
                scale=scale_factor
            )
            
            # Download upscaled image
            upscaled_path = input_path.replace(".jpg", "_upscaled.jpg")
            replicate_client.download_result(output_url, upscaled_path)
            
            # Upload back to Google Drive (in "02-Ready-to-Process" folder)
            processed_filename = f"processed_{file.original_filename}"
            drive_file_id = drive_client.upload_file(
                upscaled_path,
                processed_filename,
                folder_id="READY_TO_PROCESS_FOLDER_ID"
            )
            
            # Update file record
            file.processed_filename = processed_filename
            file.status = "processed"
            self.db.commit()
            
            return {
                "status": "success",
                "file_id": file_id,
                "processed_filename": processed_filename,
                "drive_file_id": drive_file_id
            }
        
        finally:
            # Cleanup temp files
            if os.path.exists(input_path):
                os.remove(input_path)
            upscaled_path = input_path.replace(".jpg", "_upscaled.jpg")
            if os.path.exists(upscaled_path):
                os.remove(upscaled_path)
    
    def on_success(self, result: Dict[str, Any]):
        from app.worker import publish_event
        publish_event("file.processed", {
            "file_id": result["file_id"],
            "processed_filename": result["processed_filename"]
        })
```

---

## Worker Implementation

### worker.py

```python
import pika
import json
import logging
from sqlalchemy.orm import Session
from app.database import SessionLocal
from app.config import settings
from app.commands.base import BaseCommand
from app.models.job import Job
from typing import Dict, Any
import time
import random

# Import all command classes
from app.commands.create_file_record import CreateFileRecordCommand
from app.commands.upscale_image import UpscaleImageCommand
from app.commands.generate_metadata import GenerateMetadataCommand
# ... import others

COMMAND_REGISTRY = {
    "CreateFileRecord": CreateFileRecordCommand,
    "UpscaleImage": UpscaleImageCommand,
    "GenerateMetadata": GenerateMetadataCommand,
    # ... register others
}

logger = logging.getLogger(__name__)

def get_command_class(command_type: str) -> type[BaseCommand]:
    """Resolve command type string to class."""
    if command_type not in COMMAND_REGISTRY:
        raise ValueError(f"Unknown command type: {command_type}")
    return COMMAND_REGISTRY[command_type]

def execute_job(job_id: str, command_type: str, payload: Dict[str, Any], attempt: int):
    """Execute a single job."""
    db = SessionLocal()
    
    try:
        # Update job status
        job = db.query(Job).filter_by(id=job_id).first()
        if not job:
            logger.error(f"Job {job_id} not found")
            return
        
        job.status = "running"
        job.attempts = attempt
        job.started_at = datetime.utcnow()
        db.commit()
        
        # Execute command
        command_class = get_command_class(command_type)
        command = command_class(db=db, payload=payload)
        
        result = command.execute()
        
        # Success: update job
        job.status = "completed"
        job.result_json = json.dumps(result)
        job.completed_at = datetime.utcnow()
        db.commit()
        
        # Call success hook
        command.on_success(result)
        
        logger.info(f"Job {job_id} completed successfully")
    
    except Exception as e:
        logger.error(f"Job {job_id} failed: {str(e)}", exc_info=True)
        
        # Update job with error
        job = db.query(Job).filter_by(id=job_id).first()
        if job:
            job.status = "failed"
            job.last_error = str(e)
            db.commit()
        
        # Call failure hook
        try:
            command_class = get_command_class(command_type)
            command = command_class(db=db, payload=payload)
            command.on_failure(e)
        except:
            pass
        
        # Raise to trigger retry
        raise
    
    finally:
        db.close()

def callback(ch, method, properties, body):
    """RabbitMQ message callback."""
    try:
        message = json.loads(body)
        job_id = message["job_id"]
        command_type = message["command_type"]
        payload = message["payload"]
        attempt = message.get("attempt", 1)
        max_attempts = message.get("max_attempts", settings.retry_max_attempts)
        
        logger.info(f"Processing job {job_id} (attempt {attempt}/{max_attempts})")
        
        try:
            execute_job(job_id, command_type, payload, attempt)
            ch.basic_ack(delivery_tag=method.delivery_tag)
        
        except Exception as e:
            # Retry logic
            if attempt < max_attempts:
                # Calculate backoff delay
                delay = settings.retry_base_delay_seconds * (2 ** (attempt - 1))
                jitter = random.uniform(0, 0.5)
                delay += jitter
                
                logger.info(f"Retrying job {job_id} in {delay:.2f} seconds")
                
                # Re-queue with incremented attempt
                message["attempt"] = attempt + 1
                
                time.sleep(delay)  # Simple delay (for better: use delayed exchange)
                ch.basic_publish(
                    exchange='',
                    routing_key=method.routing_key,
                    body=json.dumps(message)
                )
                ch.basic_ack(delivery_tag=method.delivery_tag)
            
            else:
                # Max retries exceeded: move to DLQ
                logger.error(f"Job {job_id} exhausted retries, moving to DLQ")
                ch.basic_publish(
                    exchange='',
                    routing_key=f"{method.routing_key}.dlq",
                    body=body
                )
                ch.basic_ack(delivery_tag=method.delivery_tag)
    
    except Exception as e:
        logger.error(f"Error processing message: {str(e)}", exc_info=True)
        ch.basic_nack(delivery_tag=method.delivery_tag, requeue=False)

def start_worker():
    """Start the worker process."""
    connection = pika.BlockingConnection(
        pika.URLParameters(settings.rabbitmq_url)
    )
    channel = connection.channel()
    
    # Declare queues
    queues = [
        "file.ingest",
        "image.process",
        "listing.create",
        "social.publish",
        "events"
    ]
    
    for queue in queues:
        channel.queue_declare(queue=queue, durable=True)
        channel.queue_declare(queue=f"{queue}.dlq", durable=True)
    
    # Set prefetch count (process one message at a time per thread)
    channel.basic_qos(prefetch_count=1)
    
    # Start consuming
    for queue in queues:
        channel.basic_consume(queue=queue, on_message_callback=callback)
    
    logger.info("Worker started, waiting for messages...")
    channel.start_consuming()

if __name__ == "__main__":
    logging.basicConfig(level=logging.INFO)
    start_worker()
```

---

## Event Publishing

Events are published to the `events` queue for downstream processing (logging, webhooks, analytics).

```python
import pika
import json
from app.config import settings

def publish_event(event_type: str, data: Dict[str, Any]):
    """Publish an event to the events queue."""
    connection = pika.BlockingConnection(
        pika.URLParameters(settings.rabbitmq_url)
    )
    channel = connection.channel()
    
    channel.queue_declare(queue="events", durable=True)
    
    message = {
        "event_type": event_type,
        "data": data,
        "timestamp": datetime.utcnow().isoformat() + "Z"
    }
    
    channel.basic_publish(
        exchange='',
        routing_key='events',
        body=json.dumps(message),
        properties=pika.BasicProperties(delivery_mode=2)  # Persistent
    )
    
    connection.close()
```

**Event Types:**
- `file.received`
- `file.processed`
- `file.updated`
- `file.approved`
- `listing.created`
- `listing.updated`
- `social.posted`
- `job.failed`

---

## Retry Strategy

**Exponential Backoff with Jitter:**

```
delay = base_delay * (2 ^ (attempt - 1)) + jitter
```

**Example (base_delay = 1s):**
- Attempt 1: 1s + jitter
- Attempt 2: 2s + jitter
- Attempt 3: 4s + jitter
- Attempt 4: 8s + jitter
- Attempt 5: 16s + jitter

**Jitter:** Random value between 0 and 0.5 seconds (prevents thundering herd)

**Configuration:**
- `retry_max_attempts`: 5 (default)
- `retry_base_delay_seconds`: 1 (default)

---

## Logging

**Structured Logging (JSON):**

```python
import logging
import json
from datetime import datetime

class JSONFormatter(logging.Formatter):
    def format(self, record):
        log_data = {
            "timestamp": datetime.utcnow().isoformat() + "Z",
            "level": record.levelname,
            "message": record.getMessage(),
            "module": record.module,
            "function": record.funcName,
            "line": record.lineno
        }
        if record.exc_info:
            log_data["exception"] = self.formatException(record.exc_info)
        return json.dumps(log_data)

# Configure
handler = logging.StreamHandler()
handler.setFormatter(JSONFormatter())
logger = logging.getLogger()
logger.addHandler(handler)
logger.setLevel(logging.INFO)
```

**What to Log:**
- Job start/completion (INFO)
- Errors (ERROR with stack trace)
- Retry attempts (INFO)
- External API calls (INFO with service name, endpoint, duration)

---

## Testing

**Unit Tests:**

```python
import pytest
from app.commands.create_file_record import CreateFileRecordCommand
from app.models.file import File

def test_create_file_record_command(db_session):
    payload = {
        "file_id": "test123",
        "filename": "test.jpg",
        "species": "whitetail-deer"
    }
    
    command = CreateFileRecordCommand(db=db_session, payload=payload)
    result = command.execute()
    
    assert result["status"] == "created"
    assert result["file_id"] == "test123"
    
    # Verify DB record
    file = db_session.query(File).filter_by(id="test123").first()
    assert file is not None
    assert file.original_filename == "test.jpg"
```

**Integration Tests:**

Mock external services (Replicate, Google Drive) with responses library or pytest-mock.

```python
from unittest.mock import patch

def test_upscale_image_command(db_session):
    # Setup: create file record
    file = File(id="test123", original_filename="test.jpg", status="ingested")
    db_session.add(file)
    db_session.commit()
    
    with patch("app.services.replicate.ReplicateClient.upscale_image") as mock_upscale:
        mock_upscale.return_value = "https://replicate.com/output.jpg"
        
        payload = {"file_id": "test123", "scale_factor": 4}
        command = UpscaleImageCommand(db=db_session, payload=payload)
        result = command.execute()
        
        assert result["status"] == "success"
        assert file.status == "processed"
```

---

## Deployment

**Dockerfile:**
```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app/ ./app/

CMD ["python", "-m", "app.worker"]
```

**docker-compose.yml (add to existing):**
```yaml
  worker:
    build: ./backend
    command: python -m app.worker
    environment:
      - DATABASE_URL=sqlite:///./data/fieldfen.db
      - RABBITMQ_URL=amqp://rabbitmq:5672
    env_file:
      - .env
    volumes:
      - ./data:/app/data
    depends_on:
      - rabbitmq
      - api
    restart: unless-stopped
```

**Scaling:**

Run multiple worker instances:
```yaml
  worker:
    # ... same config
    deploy:
      replicas: 3
```

Each worker consumes from the same queues. RabbitMQ distributes messages across workers (round-robin).

---

## Monitoring

**Health Metrics to Track:**
- Queue depth (messages pending)
- Job success/failure rate
- Average job duration by command type
- Retry rate
- DLQ message count

**Tools:**
- RabbitMQ Management UI (port 15672)
- Prometheus + Grafana (future)
- CloudWatch/DataDog (if cloud-hosted)

**Alerts:**
- DLQ message count > 10
- Job failure rate > 20%
- Queue depth > 1000

---

## Next Steps

After implementing this spec:
1. Write command implementations (all 8 commands)
2. Implement external service clients (Replicate, Shopify, Google Drive, etc.)
3. Set up RabbitMQ in Docker
4. Test end-to-end: webhook → queue → command → event

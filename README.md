# Bulk Certificate Generator Backend Service

A high-performance, resilient asynchronous backend service for bulk certificate generation built with **FastAPI**, **SQLAlchemy 2.0 (async)**, **MySQL** (via `aiomysql`), **Pillow**, and **Alembic**.

---

## Architecture Overview

- **`app/api/`**: Clean FastAPI route controllers handling HTTP status codes, responses, and BackgroundTasks dispatching.
- **`app/schemas/`**: Pydantic v2 schemas for strict input validation, response serialization, and decoupling ORM entities.
- **`app/models/`**: SQLAlchemy 2.0 Declarative ORM models (`GenerationJob`, `CertificateTask`).
- **`app/services/`**: Business logic isolating per-recipient validation and Pillow image rendering.
- **`app/core/`**: Centralized configuration (`pydantic-settings`), async engine, and database session dependency.
- **`app/assets/`**: Predefined high-resolution certificate template (`template.png`).
- **`app/generated_certs/`**: Unique output storage (`{job_id}_{task_id}.png`).
- **`tests/`**: Comprehensive pytest test suite with isolated SQLite async test database.

---

## Design Decisions

### 1. Why `FastAPI BackgroundTasks` Over Celery / RQ for This Scope
- **Simplicity & Zero Infrastructure Burden**: Celery requires an external message broker (RabbitMQ/Redis), worker processes, monitoring dashboards (Flower), and complex concurrency management. For a focused technical assessment or low-to-medium volume, `BackgroundTasks` executes in-process asynchronously without requiring external infrastructure.
- **Synchronous Fast Ingestion**: The HTTP POST creates the `GenerationJob` and its `CertificateTask` rows in MySQL and returns `202 Accepted` within milliseconds, decoupling client response times from image generation.

### 2. What We Would Change at Scale (Higher Volume)
- **Distributed Queue & Broker**: Replace `BackgroundTasks` with **Celery** or **ARQ** backed by Redis/RabbitMQ or AWS SQS.
- **Horizontal Worker Auto-scaling**: Run dedicated worker containers scaled horizontally via Kubernetes (HPA based on queue depth) so image generation load never competes with the API web server for CPU/memory.
- **Object Storage**: Store generated certificates in Amazon S3 or Google Cloud Storage and save signed/pre-signed URLs in the database rather than local disks.
- **Automatic Retries & Exponential Backoff**: Implement dead-letter queues and retry policies for transient I/O or network failures.
- **Idempotency & Rate Limiting**: Support idempotency keys on `POST /jobs` to prevent accidental duplicate bulk submissions and enforce per-tenant rate limits.

### 3. Per-Recipient Task Rows vs. Single JSON Blob
- **Granular Partial Failure Tracking**: Storing each recipient as an individual `CertificateTask` row makes individual failures auditable and queryable (`WHERE status = 'FAILED'`).
- **Batch Resilience**: An invalid email format or corrupted name on record #5 does not invalidate or abort the other 999 records in the batch.
- **Atomic Retries**: Allows retrying individual failed tasks without reprocessing the entire job.

---

## Step-by-Step Commands to Run

### Option A: Run Locally with Python & Uvicorn (Fastest)

#### 1. Setup Virtual Environment & Install Dependencies
```bash
# In the project root (d:\Aereo)
python -m venv venv

# Activate virtual environment
# Windows (PowerShell):
.\venv\Scripts\Activate.ps1
# Linux/macOS:
source venv/bin/activate

# Install requirements
pip install -r requirements.txt
```

#### 2. Run Database Migrations
By default, the application runs on a local SQLite async database for zero-config startup (`sqlite+aiosqlite:///./bulk_certs.db`). If using MySQL, set `DATABASE_URL=mysql+aiomysql://<user>:<password>@localhost:3306/<dbname>` in `.env`.

```bash
# Run Alembic migrations to create tables
alembic upgrade head
```

#### 3. Start the FastAPI Server
```bash
uvicorn app.main:app --reload --host 0.0.0.0 --port 8000
```
- Interactive Swagger UI: [http://localhost:8000/docs](http://localhost:8000/docs)
- Health Check: [http://localhost:8000/health](http://localhost:8000/health)

---

### Option B: Run via Docker Compose (Includes MySQL 8.0)

```bash
# Build images and start MySQL + FastAPI services
docker-compose up --build

# Run in background (detached mode)
docker-compose up -d
```

---

## Running Tests

The test suite runs against an isolated async test database with complete mocks and fault injections:

```bash
pytest -v
```

Tests include:
- `test_create_job`: Valid bulk payload -> 202 Accepted + DB tasks created.
- `test_create_job_invalid_payload`: Empty or malformed payload -> 422 Unprocessable Entity.
- `test_per_recipient_validation_failure`: Partial validation failure isolation.
- `test_certificate_generation`: PIL image rendering and output verification.
- `test_generation_failure_isolation`: Pillow fault injection isolation test.
- `test_job_status_endpoint`: Querying status and task tracking.
- `test_certificate_retrieval`: Downloading generated certificate files.

---

## API Usage Examples

### 1. Create a Bulk Certificate Job
```bash
curl -X POST "http://localhost:8000/jobs" \
  -H "Content-Type: application/json" \
  -d '{
    "recipients": [
      {
        "name": "Jane Doe",
        "email": "jane.doe@example.com",
        "course_name": "Full Stack Mastery",
        "completion_date": "2026-10-15"
      },
      {
        "name": "John Smith",
        "email": "john.smith@example.com",
        "course_name": "Cloud Architecture",
        "completion_date": "2026-10-18"
      },
      {
        "name": "Broken Email Person",
        "email": "invalid-email-string",
        "course_name": "Python Fundamentals",
        "completion_date": "2026-10-18"
      }
    ]
  }'
```
**Response (202 Accepted)**:
```json
{
  "job_id": "848a60a7-bf94-4632-9cb7-7f9e54d318e8",
  "status": "PENDING",
  "total_count": 3,
  "message": "Job accepted for processing"
}
```

### 2. Check Job Status & Progress
```bash
curl -X GET "http://localhost:8000/jobs/848a60a7-bf94-4632-9cb7-7f9e54d318e8"
```
**Response (200 OK)**:
```json
{
  "id": "848a60a7-bf94-4632-9cb7-7f9e54d318e8",
  "status": "COMPLETED_WITH_ERRORS",
  "total_count": 3,
  "success_count": 2,
  "failure_count": 1,
  "tasks": [
    {
      "id": "e935bbff-1a67-4d94-a16f-d128d5d4d38c",
      "recipient_name": "Jane Doe",
      "recipient_email": "jane.doe@example.com",
      "course_name": "Full Stack Mastery",
      "completion_date": "2026-10-15",
      "status": "SUCCESS",
      "failure_reason": null,
      "certificate_file_path": ".../848a60a7..._e935bbff....png"
    },
    {
      "id": "33b2fcbc-fba3-4c90-8438-e67c858ba5c4",
      "recipient_name": "Broken Email Person",
      "recipient_email": "invalid-email-string",
      "course_name": "Python Fundamentals",
      "completion_date": "2026-10-18",
      "status": "FAILED",
      "failure_reason": "Invalid email format: 'invalid-email-string'"
    }
  ]
}
```

### 3. Download Generated Certificate
```bash
curl -X GET "http://localhost:8000/jobs/848a60a7-bf94-4632-9cb7-7f9e54d318e8/certificates/e935bbff-1a67-4d94-a16f-d128d5d4d38c" \
  --output certificate.png
```

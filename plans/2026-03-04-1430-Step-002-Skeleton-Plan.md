# STEP-002: Skeleton MVP Plan

**Created:** 2026-03-04
**Status:** Draft
**Author:** Claude

---

## A) Scope

### In Scope
- Docker Compose setup with 5 services: postgres, redis, api, worker, web
- FastAPI backend with health endpoint and CRUD for endpoints/tasks/datasets
- Redis + RQ task queue for async scan jobs
- PostgreSQL database with minimal schema
- Next.js minimal admin UI with 3 pages (endpoints, tasks, datasets)
- Local filesystem scanning (type=local, roots=/data)
- Dataset detection logic (raw/result/hybrid/unknown + has_gcp flag)
- Read-only file access (no modifications to scanned files)

### Out of Scope (Future Steps)
- SMB/CIFS network scanning (Step-003)
- Authentication/authorization
- Multi-tenancy
- Real-time WebSocket updates
- Pagination/search/filtering
- Error recovery and retry logic
- Backup/restore
- Production hardening

---

## B) File Tree to Create

```
dms-handoff-public/
├── docker-compose.yml
├── .env.example
├── .dockerignore
├── .gitignore (update)
│
├── backend/
│   ├── Dockerfile
│   ├── requirements.txt
│   ├── pyproject.toml
│   ├── app/
│   │   ├── __init__.py
│   │   ├── main.py                    # FastAPI app entry point
│   │   ├── config.py                  # Settings via pydantic-settings
│   │   ├── database.py                # SQLAlchemy async engine
│   │   ├── models/
│   │   │   ├── __init__.py
│   │   │   ├── endpoint.py
│   │   │   ├── scan_task.py
│   │   │   └── dataset.py
│   │   ├── schemas/
│   │   │   ├── __init__.py
│   │   │   ├── endpoint.py
│   │   │   ├── scan_task.py
│   │   │   └── dataset.py
│   │   ├── routers/
│   │   │   ├── __init__.py
│   │   │   ├── health.py
│   │   │   ├── endpoints.py
│   │   │   ├── tasks.py
│   │   │   └── datasets.py
│   │   ├── services/
│   │   │   ├── __init__.py
│   │   │   ├── scanner.py             # Dataset detection logic
│   │   │   └── task_runner.py         # RQ task implementation
│   │   └── utils/
│   │       ├── __init__.py
│   │       └── file_utils.py          # Read-only file operations
│   └── alembic/
│       ├── alembic.ini
│       ├── env.py
│       └── versions/
│           └── 001_initial.py
│
├── worker/
│   ├── Dockerfile
│   ├── requirements.txt               # Same as backend + rq
│   └── worker.py                      # RQ worker entry point
│
├── web/
│   ├── Dockerfile
│   ├── package.json
│   ├── next.config.js
│   ├── tsconfig.json
│   ├── app/
│   │   ├── layout.tsx
│   │   ├── page.tsx                   # Redirect to /endpoints
│   │   ├── endpoints/
│   │   │   └── page.tsx
│   │   ├── tasks/
│   │   │   └── page.tsx
│   │   └── datasets/
│   │       └── page.tsx
│   ├── components/
│   │   ├── Layout.tsx
│   │   ├── EndpointForm.tsx
│   │   ├── TaskForm.tsx
│   │   └── DatasetList.tsx
│   └── lib/
│       └── api.ts                     # API client
│
├── sample_data/
│   ├── raw_dataset/
│   │   ├── info/
│   │   │   └── device_info.json
│   │   └── metadata.yaml
│   ├── result_dataset/
│   │   ├── transforms.json
│   │   └── undistort/
│   │       └── .gitkeep
│   ├── hybrid_dataset/
│   │   ├── info/
│   │   │   └── device_info.json
│   │   ├── metadata.yaml
│   │   └── transforms.json
│   └── with_gcp_dataset/
│       ├── info/
│       │   └── device_info.json
│       ├── metadata.yaml
│       └── ctrl_points.csv
│
└── docs/
    └── todo/
        └── 2026-03-04-1430-Step-002-Skeleton-Plan.md (this file)
```

---

## C) DB Schema + Migration Approach

### Tables

#### `endpoints`
| Column       | Type      | Constraints                |
|--------------|-----------|----------------------------|
| id           | UUID      | PK, default uuid_generate_v4() |
| name         | VARCHAR   | NOT NULL                   |
| type         | VARCHAR   | NOT NULL, CHECK IN ('local') |
| roots        | JSONB     | NOT NULL (array of paths)  |
| is_active    | BOOLEAN   | DEFAULT true               |
| created_at   | TIMESTAMP | DEFAULT NOW()              |
| updated_at   | TIMESTAMP | DEFAULT NOW()              |

#### `scan_tasks`
| Column       | Type      | Constraints                |
|--------------|-----------|----------------------------|
| id           | UUID      | PK                         |
| endpoint_id  | UUID      | FK -> endpoints.id         |
| status       | VARCHAR   | CHECK IN ('pending', 'running', 'succeeded', 'failed') |
| progress     | INTEGER   | DEFAULT 0 (0-100)          |
| message      | TEXT      | NULL (current status msg)  |
| error        | TEXT      | NULL (error details)       |
| started_at   | TIMESTAMP | NULL                       |
| completed_at | TIMESTAMP | NULL                       |
| created_at   | TIMESTAMP | DEFAULT NOW()              |

#### `datasets`
| Column        | Type      | Constraints                |
|---------------|-----------|----------------------------|
| id            | UUID      | PK                         |
| scan_task_id  | UUID      | FK -> scan_tasks.id        |
| path          | TEXT      | NOT NULL                   |
| name          | VARCHAR   | NOT NULL (folder name)     |
| type          | VARCHAR   | CHECK IN ('raw', 'result', 'hybrid', 'unknown') |
| has_gcp       | BOOLEAN   | DEFAULT false              |
| file_count    | INTEGER   | DEFAULT 0                  |
| first_seen    | TIMESTAMP | DEFAULT NOW()              |
| last_seen     | TIMESTAMP | DEFAULT NOW()              |

### Migration Approach
- Use **Alembic** for migrations
- Single initial migration `001_initial.py` creates all tables
- On container start: `alembic upgrade head`
- No auto-migrate in production; explicit migrations only

---

## D) API Endpoints

### Health
```
GET /health
Response 200: { "status": "ok", "version": "0.1.0" }
```

### Endpoints
```
GET    /api/endpoints              # List all
POST   /api/endpoints              # Create
Request:  { "name": str, "type": "local", "roots": ["/data"] }
Response: { "id": uuid, "name": str, "type": str, "roots": [...], "is_active": bool, ... }

GET    /api/endpoints/{id}         # Get one
DELETE /api/endpoints/{id}         # Delete (soft: set is_active=false)
```

### Tasks
```
GET    /api/tasks                  # List all (optional ?endpoint_id=...&status=...)
POST   /api/tasks                  # Create & enqueue scan task
Request:  { "endpoint_id": uuid }
Response: { "id": uuid, "endpoint_id": uuid, "status": "pending", ... }

GET    /api/tasks/{id}             # Get one (poll for status)
Response: { "id": uuid, "status": str, "progress": int, "message": str, ... }
```

### Datasets
```
GET    /api/datasets               # List all (optional ?scan_task_id=...&type=...)
Response: [{ "id": uuid, "path": str, "name": str, "type": str, "has_gcp": bool, ... }]

GET    /api/datasets/{id}          # Get one
```

---

## E) Worker Task Payload + Progress/Logging

### RQ Task: `scan_endpoint`
```python
# Payload
{
    "task_id": "uuid",
    "endpoint_id": "uuid",
    "roots": ["/data", "/data/more"]
}

# Progress updates via Redis key: task:{task_id}:progress
{
    "progress": 45,
    "message": "Scanning /data/hybrid_dataset..."
}

# On completion:
# - Insert datasets into DB
# - Update task status to 'succeeded' or 'failed'
```

### Logging Approach
- Worker logs to stdout (Docker captures)
- Structured JSON logs: `{"level": "INFO", "task_id": "...", "message": "..."}`
- Progress stored in DB (`scan_tasks.progress`, `scan_tasks.message`) for UI polling

---

## F) Web Pages (Minimal UI)

### `/endpoints`
- Table: Name | Type | Roots | Status | Actions
- "Create Endpoint" button opens modal/form
- Form: Name (text), Type (dropdown: local), Roots (comma-separated paths)
- "Submit Scan" button per endpoint -> creates task, redirects to /tasks

### `/tasks`
- Table: ID | Endpoint | Status | Progress | Created | Actions
- Status badge color: pending(gray), running(blue), succeeded(green), failed(red)
- Polling: Refresh every 2s if any task is 'running'
- Click task row -> filter datasets by scan_task_id

### `/datasets`
- Table: Name | Type | Path | Has GCP | First Seen
- Filter by: type dropdown, scan_task dropdown
- Type badge color: raw(blue), result(green), hybrid(purple), unknown(gray)

---

## G) Sample Test Data Structure

```
sample_data/
├── raw_dataset/                    # Type: raw
│   ├── info/
│   │   └── device_info.json       # {"device": "camera-001", ...}
│   └── metadata.yaml              # "timestamp: 2024-01-01"
│
├── result_dataset/                 # Type: result
│   ├── transforms.json             # {"version": "1.0", ...}
│   └── undistort/
│       └── image_001.jpg
│
├── hybrid_dataset/                 # Type: hybrid
│   ├── info/
│   │   └── device_info.json
│   ├── metadata.yaml
│   └── transforms.json
│
├── with_gcp_dataset/               # Type: raw, has_gcp=true
│   ├── info/
│   │   └── device_info.json
│   ├── metadata.yaml
│   └── ctrl_points.csv            # GCP control points
│
└── unknown_folder/                 # Type: unknown (no signatures)
    └── random_file.txt
```

### Volume Mount in docker-compose.yml
```yaml
services:
  api:
    volumes:
      - ./sample_data:/data:ro  # Read-only mount
  worker:
    volumes:
      - ./sample_data:/data:ro  # Read-only mount
```

---

## H) Verification Commands

### 1. Start Stack
```bash
docker compose up -d
docker compose ps  # All 5 services should be "running"
```

### 2. API Health
```bash
curl http://localhost:8090/health
# Expected: {"status": "ok", "version": "0.1.0"}
```

### 3. Create Endpoint
```bash
curl -X POST http://localhost:8090/api/endpoints \
  -H "Content-Type: application/json" \
  -d '{"name": "Local Test", "type": "local", "roots": ["/data"]}'
# Expected: {"id": "...", "name": "Local Test", ...}
```

### 4. Submit Scan Task
```bash
curl -X POST http://localhost:8090/api/tasks \
  -H "Content-Type: application/json" \
  -d '{"endpoint_id": "<ENDPOINT_ID>"}'
# Expected: {"id": "...", "status": "pending", ...}
```

### 5. Poll Task Status
```bash
curl http://localhost:8090/api/tasks/<TASK_ID>
# Wait until status = "succeeded"
```

### 6. List Datasets
```bash
curl http://localhost:8090/api/datasets
# Expected: Array with raw_dataset, result_dataset, hybrid_dataset, with_gcp_dataset, unknown_folder
```

### 7. Web UI Verification
1. Open http://localhost:3000
2. Navigate to /endpoints
3. Create endpoint (name="Test", type="local", roots="/data")
4. Click "Submit Scan"
5. Navigate to /tasks, watch progress
6. Navigate to /datasets, verify 5 datasets listed with correct types

---

## I) Risks + Mitigations

| Risk | Mitigation |
|------|------------|
| Docker networking issues | Use explicit service names, healthchecks |
| DB migration race on first start | Add `depends_on` + healthcheck for postgres |
| Worker fails silently | Set RQ failure handlers, log to stdout |
| File permission denied | Mount volumes as `:ro`, document required permissions |
| Large directory scan timeout | Add progress updates, reasonable timeouts |
| Next.js SSR hydration mismatch | Use client-only rendering for lists |
| Alembic version mismatch | Pin alembic version in requirements.txt |

### Rollback Plan
1. `docker compose down -v` (removes containers + volumes)
2. `git checkout .` (reverts any local changes)
3. Re-run `docker compose up -d`
4. If DB schema broken: `docker volume rm dms-handoff-public_postgres_data`

---

## J) Implementation Order

1. **Docker infrastructure** (docker-compose.yml, Dockerfiles, .env)
2. **Database** (models, Alembic migration)
3. **API core** (main.py, config, health endpoint)
4. **API routers** (endpoints, tasks, datasets)
5. **Scanner service** (detection logic)
6. **Worker** (RQ setup, task implementation)
7. **Web UI** (Next.js skeleton, 3 pages)
8. **Sample data** (create test datasets)
9. **Verification** (run through acceptance criteria)

---

## Acceptance Criteria Checklist

- [ ] `docker compose up -d` starts postgres, redis, api, worker, web
- [ ] `GET /health` returns 200
- [ ] Web UI loads at http://localhost:3000
- [ ] Can create endpoint (type=local, roots=/data)
- [ ] Can submit scan task
- [ ] Task transitions: pending -> running -> succeeded
- [ ] Datasets list shows detected items with correct types
- [ ] All file operations are read-only (no modifications)

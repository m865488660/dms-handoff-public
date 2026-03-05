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
datama/                              # Private repo at C:\datama
├── docker-compose.yml
├── .env.example
├── .dockerignore
├── .gitignore (update)
├── README.md
│
├── api/                             # FastAPI backend
│   ├── Dockerfile
│   ├── requirements.txt
│   ├── init_db.sql                  # Simple init SQL (no Alembic for v0.1)
│   ├── main.py                      # FastAPI app entry point
│   ├── database.py                  # asyncpg connection pool
│   ├── models.py                    # Pydantic schemas
│   └── routes/
│       ├── __init__.py
│       ├── endpoints.py
│       ├── tasks.py
│       └── datasets.py
│
├── worker/                          # RQ worker
│   ├── Dockerfile
│   ├── requirements.txt
│   ├── worker.py                    # RQ worker entry point
│   └── scan.py                      # Scan logic + dataset classification
│
├── web/                             # Next.js frontend
│   ├── Dockerfile
│   ├── package.json
│   ├── next.config.js
│   ├── app/
│   │   ├── layout.js
│   │   ├── page.js                  # Home page
│   │   ├── globals.css
│   │   ├── endpoints/
│   │   │   └── page.js
│   │   ├── tasks/
│   │   │   └── page.js
│   │   └── datasets/
│   │       └── page.js
│   └── lib/
│       └── api.js                   # API client
│
├── sample_data/                     # Test fixtures (mounted to /data)
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
│   └── gcp_dataset/
│       ├── info/
│       │   └── device_info.json
│       ├── metadata.yaml
│       └── ctrl_points.csv
│
├── configs/
│   └── settings.yaml
│
└── docs/
    └── todo/
        └── 2026-03-04-1430-Step-002-Skeleton-Plan.md (this file)
```

---

## C) DB Schema (Simplified for v0.1)

### UUID Strategy
- Use `gen_random_uuid()` from `pgcrypto` extension for all UUID primary keys
- init_db.sql must include: `CREATE EXTENSION IF NOT EXISTS pgcrypto;`
- All PK columns have `DEFAULT gen_random_uuid()`

### Tables (via /docker-entrypoint-initdb.d/init_db.sql)

#### `endpoints`
| Column       | Type      | Constraints                |
|--------------|-----------|----------------------------|
| endpoint_id  | UUID      | PK, DEFAULT gen_random_uuid() |
| name         | TEXT      | NOT NULL                   |
| type         | TEXT      | NOT NULL, CHECK IN ('local','smb','s3') |
| config       | JSONB     | NOT NULL DEFAULT '{}'      |
| status       | TEXT      | NOT NULL DEFAULT 'active'  |
| created_at   | TIMESTAMPTZ | DEFAULT NOW()            |

#### `tasks`
| Column       | Type      | Constraints                |
|--------------|-----------|----------------------------|
| task_id      | UUID      | PK, DEFAULT gen_random_uuid() |
| endpoint_id  | UUID      | NOT NULL, FK -> endpoints.endpoint_id |
| type         | TEXT      | NOT NULL DEFAULT 'scan'    |
| status       | TEXT      | CHECK IN ('pending', 'running', 'succeeded', 'failed') |
| payload      | JSONB     | NOT NULL DEFAULT '{}'      |
| progress     | REAL      | DEFAULT 0                  |
| error        | TEXT      | NULL                       |
| result       | JSONB     | DEFAULT '{}'                |
| created_at   | TIMESTAMPTZ | DEFAULT NOW()            |
| updated_at   | TIMESTAMPTZ | DEFAULT NOW()            |

#### `datasets`
| Column        | Type      | Constraints                |
|---------------|-----------|----------------------------|
| dataset_id    | UUID      | PK, DEFAULT gen_random_uuid() |
| dataset_type  | TEXT      | CHECK IN ('raw', 'result', 'hybrid', 'unknown') |
| fingerprint   | TEXT      | NULL                       |
| display_name  | TEXT      | NULL                       |
| has_gcp       | BOOLEAN   | DEFAULT false              |
| notes         | TEXT      | NULL                       |
| first_seen    | TIMESTAMPTZ | DEFAULT NOW()            |
| last_seen     | TIMESTAMPTZ | DEFAULT NOW()            |
| status        | TEXT      | DEFAULT 'active'           |
| meta          | JSONB     | DEFAULT '{}'               |

#### `dataset_locations`
| Column        | Type      | Constraints                |
|---------------|-----------|----------------------------|
| id            | UUID      | PK, DEFAULT gen_random_uuid() |
| dataset_id    | UUID      | FK -> datasets.dataset_id  |
| endpoint_id   | UUID      | FK -> endpoints.endpoint_id |
| path          | TEXT      | NOT NULL                   |
| is_primary    | BOOLEAN   | DEFAULT TRUE               |
| bytes         | BIGINT    | NULL                       |
| last_verified | TIMESTAMPTZ | DEFAULT NOW()            |

### Migration Approach
- **v0.1**: Use Postgres initdb via `/docker-entrypoint-initdb.d/init_db.sql`
  - Mount `postgres/init_db.sql` to `/docker-entrypoint-initdb.d/` in docker-compose.yml
  - Executed automatically on first container start
  - Idempotent: use `IF NOT EXISTS` for all objects
- Future: Add Alembic for production migrations

---

## D) API Endpoints

### Health
```
GET /health
Response 200: { "status": "ok", "version": "0.1.0" }
```

### Endpoints
```
GET    /endpoints              # List all
POST   /endpoints              # Create
Request:  { "name": str, "type": "local", "config": {"roots": ["/data"]} }
Response: { "endpoint_id": uuid, "name": str, "type": str, "config": {...}, "status": str, ... }
```

### Tasks
```
POST   /tasks/scan             # Create & enqueue scan task
Request:  { "endpoint_id": uuid }
Response: { "task_id": uuid, "endpoint_id": uuid, "status": "pending", "progress": 0, ... }

GET    /tasks                  # List all
GET    /tasks/{id}             # Get one (poll for status)
```

### Datasets
```
GET    /datasets               # List all (optional ?dataset_type=...&has_gcp=...)
Response: [{ "dataset_id": uuid, "dataset_type": str, "display_name": str, "has_gcp": bool, ... }]

GET    /datasets/{id}          # Get one (includes locations)
```

---

## E) Worker Task: Scan

### Signature Rules
- **Raw**: `metadata.yaml` AND `info/device_info.json` present
- **Result**: `transforms.json` OR `undistort/` directory present
- **Hybrid**: Both Raw + Result signatures present
- **GCP**: `ctrl_points.csv` exists → set `has_gcp=true`

### RQ Task: `scan.run_scan`
```python
# Payload from task.payload
{
    "endpoint_id": "uuid",
    "roots": ["/data"],
    "max_depth": 4,
    "exclude_patterns": ["$RECYCLE.BIN", "@eaDir", ".Trash"]
}

# Progress updates via DB task.progress
# On completion: upsert datasets + locations, update task.status
```

---

## F) Web Pages (Minimal UI)

### `/endpoints`
- Table: Name | Type | Roots | Status | Created
- "Add Endpoint" button opens modal
- Form: Name, Type (local), Root Paths (textarea, one per line)

### `/tasks`
- Table: Task ID | Type | Status | Progress | Created
- Status badge colors: pending(gray), running(blue), succeeded(green), failed(red)
- Auto-refresh every 3s if any running task

### `/datasets`
- Table: Name | Type | GCP | First Seen | Last Seen
- Filter by: dataset_type, has_gcp
- Click row → show detail modal with locations

---

## G) Sample Test Data Structure

```
sample_data/
├── raw_dataset/                    # Type: raw
│   ├── info/
│   │   └── device_info.json
│   └── metadata.yaml
│
├── result_dataset/                 # Type: result
│   ├── transforms.json
│   └── undistort/
│       └── .gitkeep
│
├── hybrid_dataset/                 # Type: hybrid
│   ├── info/
│   │   └── device_info.json
│   ├── metadata.yaml
│   └── transforms.json
│
└── gcp_dataset/                    # Type: raw, has_gcp=true
    ├── info/
    │   └── device_info.json
    ├── metadata.yaml
    └── ctrl_points.csv
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

### 1. Start Stack (from C:\datama)
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
curl -X POST http://localhost:8090/endpoints \
  -H "Content-Type: application/json" \
  -d '{"name": "Local Test", "type": "local", "config": {"roots": ["/data"]}}'
# Expected: {"endpoint_id": "...", "name": "Local Test", ...}
```

### 4. Submit Scan Task
```bash
curl -X POST http://localhost:8090/tasks/scan \
  -H "Content-Type: application/json" \
  -d '{"endpoint_id": "<ENDPOINT_ID>"}'
# Expected: {"task_id": "...", "status": "pending", ...}
```

### 5. Poll Task Status
```bash
curl http://localhost:8090/tasks/<TASK_ID>
# Wait until status = "succeeded"
```

### 6. List Datasets
```bash
curl http://localhost:8090/datasets
# Expected: 4 datasets with types: raw, result, hybrid, raw (gcp)
```

### 7. Web UI Verification
1. Open http://localhost:3000
2. Navigate to /endpoints
3. Create endpoint (name="Test", roots="/data")
4. Navigate to /tasks, submit scan
5. Watch task progress, wait for succeeded
6. Navigate to /datasets, verify 4 datasets listed

---

## I) Risks + Mitigations

| Risk | Mitigation |
|------|------------|
| Docker networking issues | Use explicit service names, healthchecks |
| DB init race on first start | Add `depends_on` + healthcheck for postgres |
| Worker fails silently | Set RQ failure handlers, log to stdout |
| File permission denied | Mount volumes as `:ro`, document required permissions |
| Large directory scan timeout | Add progress updates, reasonable timeouts |

### Rollback Plan
1. `docker compose down -v` (removes containers + volumes)
2. `git checkout .` (reverts any local changes)
3. Re-run `docker compose up -d`

---

## J) Implementation Order

1. **Docker infrastructure** (docker-compose.yml, Dockerfiles, .env)
2. **Database** (init_db.sql)
3. **API core** (main.py, database.py, health endpoint)
4. **API routes** (endpoints, tasks, datasets)
5. **Scanner logic** (worker/scan.py)
6. **Worker** (worker.py, RQ setup)
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

---

## Publication Note

After this plan is approved, publish to public repo:
```powershell
.\scripts\publish_handoff.ps1 -PublicRepoDir C:\dms-handoff-public -StepId "Step-002" -Message "plan"
```
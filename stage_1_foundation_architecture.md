# Stage 1 — Foundation Architecture and Project Skeleton

**Project:** AI Sourcing Tool  
**Stage:** 1 of 5  
**Purpose:** Establish the base architecture required for all future stages.  
**Primary outcome:** A working full-stack skeleton where a user can create a sourcing job, store it in the database, trigger a backend process, and see process events in the frontend.

---

## 1. Stage Objective

Build the foundational application structure for the AI sourcing tool.

This stage should not yet perform real sourcing. It should create the base system that later stages will extend.

By the end of Stage 1, the app should support:

1. Minimal frontend dashboard.
2. Job creation form.
3. Backend API.
4. PostgreSQL/Supabase schema.
5. Background worker setup.
6. Process event tracking.
7. Basic job lifecycle states.
8. Environment configuration.
9. Local development workflow.
10. Clean project structure for Cursor/Codex/Plod to continue building.

---

## 2. High-Level System

Use the following architecture:

```text
Frontend: Next.js
   ↓
Backend API: FastAPI
   ↓
Database: PostgreSQL / Supabase
   ↓
Queue: Redis + Celery
   ↓
Worker: Python Celery workers
   ↓
Process Events: Stored in DB and displayed in frontend
```

---

## 3. Selected Stack

### Frontend

Use:

```text
Next.js
TypeScript
Tailwind CSS
shadcn/ui or simple reusable components
React Query or native fetch
```

### Backend

Use:

```text
FastAPI
Pydantic
SQLAlchemy or SQLModel
Alembic migrations
Python 3.11+
```

### Database

Use:

```text
PostgreSQL
Supabase-compatible schema
JSONB columns where useful
```

### Queue

Use:

```text
Redis
Celery
```

### Local Development

Use:

```text
Docker Compose
```

Services:

```text
frontend
backend
worker
postgres
redis
```

---

## 4. Repository Structure

Create a monorepo.

```text
ai-sourcing-tool/
  README.md
  docker-compose.yml
  .env.example

  apps/
    web/
      package.json
      next.config.ts
      tsconfig.json
      src/
        app/
          page.tsx
          jobs/
            page.tsx
            new/
              page.tsx
            [jobId]/
              page.tsx
              process/
                page.tsx
              candidates/
                page.tsx
        components/
          layout/
          jobs/
          process/
          ui/
        lib/
          api.ts
          types.ts

    api/
      pyproject.toml
      app/
        main.py
        core/
          config.py
          database.py
          celery_app.py
        models/
          job.py
          process_event.py
        schemas/
          job.py
          process_event.py
        api/
          routes/
            jobs.py
            process_events.py
            health.py
        services/
          jobs_service.py
          process_event_service.py
        workers/
          tasks.py
        migrations/
```

---

## 5. Database Schema for Stage 1

Implement only the base tables required for job creation and process tracking.

### 5.1 `jobs`

```sql
CREATE TABLE jobs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  title TEXT NOT NULL,
  raw_jd TEXT NOT NULL,
  role_category TEXT,
  status TEXT NOT NULL DEFAULT 'created',
  parsed_criteria_json JSONB,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### 5.2 `process_events`

```sql
CREATE TABLE process_events (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  job_id UUID NOT NULL REFERENCES jobs(id) ON DELETE CASCADE,
  stage TEXT NOT NULL,
  status TEXT NOT NULL,
  message TEXT,
  records_count INTEGER DEFAULT 0,
  metadata_json JSONB,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### 5.3 Recommended Job Status Values

```text
created
queued
running
completed
failed
archived
```

### 5.4 Recommended Process Status Values

```text
pending
running
completed
failed
warning
manual_action_required
```

---

## 6. Backend Requirements

### 6.1 FastAPI App

Create a FastAPI app with:

```text
GET /health
POST /jobs
GET /jobs
GET /jobs/{job_id}
GET /jobs/{job_id}/process-events
POST /jobs/{job_id}/start
```

---

### 6.2 API Contracts

#### POST `/jobs`

Request:

```json
{
  "title": "Backend Engineer",
  "raw_jd": "We need a Python backend developer..."
}
```

Response:

```json
{
  "id": "uuid",
  "title": "Backend Engineer",
  "status": "created",
  "created_at": "timestamp"
}
```

#### GET `/jobs`

Response:

```json
[
  {
    "id": "uuid",
    "title": "Backend Engineer",
    "role_category": null,
    "status": "created",
    "created_at": "timestamp",
    "updated_at": "timestamp"
  }
]
```

#### GET `/jobs/{job_id}`

Response:

```json
{
  "id": "uuid",
  "title": "Backend Engineer",
  "raw_jd": "...",
  "role_category": null,
  "status": "created",
  "parsed_criteria_json": null,
  "created_at": "timestamp",
  "updated_at": "timestamp"
}
```

#### POST `/jobs/{job_id}/start`

Purpose:

- Update job status to `queued`.
- Create process event `JOB_QUEUED`.
- Enqueue a Celery task.
- Return immediately.

Response:

```json
{
  "job_id": "uuid",
  "status": "queued",
  "message": "Job processing started"
}
```

#### GET `/jobs/{job_id}/process-events`

Response:

```json
[
  {
    "id": "uuid",
    "job_id": "uuid",
    "stage": "JOB_QUEUED",
    "status": "completed",
    "message": "Job has been queued",
    "records_count": 0,
    "metadata_json": {},
    "created_at": "timestamp"
  }
]
```

---

## 7. Worker Requirements

Create one placeholder Celery task:

```text
process_job_stage_1(job_id)
```

This task should simulate the future backend process.

It should create process events:

```text
JOB_PROCESS_STARTED
BASE_VALIDATION_STARTED
BASE_VALIDATION_COMPLETED
STAGE_1_COMPLETED
```

It should update the job status:

```text
queued → running → completed
```

If an exception occurs:

```text
status = failed
process event = STAGE_1_FAILED
```

---

## 8. Frontend Requirements

### 8.1 Main Pages

Implement:

```text
/
 /jobs
 /jobs/new
 /jobs/[jobId]
 /jobs/[jobId]/process
```

---

### 8.2 `/jobs`

Show a table:

| Column | Description |
|---|---|
| Title | Job title |
| Role Category | Empty for Stage 1 |
| Status | Current job status |
| Created | Created date |
| Actions | View / Process |

Buttons:

```text
Create Job
View
Process
```

---

### 8.3 `/jobs/new`

Form fields:

```text
Job Title
Raw Job Description
```

Button:

```text
Create Job
```

After creation, redirect to:

```text
/jobs/[jobId]
```

---

### 8.4 `/jobs/[jobId]`

Show:

```text
Job title
Status
Raw JD
Created date
Start Processing button
Link to Process View
```

---

### 8.5 `/jobs/[jobId]/process`

Show process timeline.

For Stage 1, use polling every 2–5 seconds.

Display:

```text
Stage
Status
Message
Records Count
Timestamp
```

Example:

```text
✓ JOB_PROCESS_STARTED — completed — Worker started job
✓ BASE_VALIDATION_STARTED — completed — Validating input
✓ BASE_VALIDATION_COMPLETED — completed — Input validated
✓ STAGE_1_COMPLETED — completed — Foundation process complete
```

---

## 9. UI Style Requirements

Keep UI minimal.

Use:

```text
White/neutral background
Simple card layout
Tables
Status badges
No heavy animations
No complex visual design
```

Status colors:

```text
pending: gray
running: blue
completed: green
failed: red
warning: yellow
manual_action_required: purple
```

---

## 10. Environment Variables

Create `.env.example`.

```env
DATABASE_URL=postgresql://postgres:postgres@localhost:5432/ai_sourcing
REDIS_URL=redis://localhost:6379/0
BACKEND_URL=http://localhost:8000
NEXT_PUBLIC_API_URL=http://localhost:8000
APP_ENV=development
```

Later stages will add:

```env
OPENAI_API_KEY=
GITHUB_TOKEN=
SEARCH_API_KEY=
HUGGINGFACE_TOKEN=
KAGGLE_USERNAME=
KAGGLE_KEY=
```

Do not add them yet unless needed.

---

## 11. Docker Compose

Create `docker-compose.yml` with:

```text
postgres
redis
api
worker
web
```

For Stage 1, local running should be possible with:

```bash
docker compose up
```

---

## 12. Logging

Backend logs should include:

```text
job_id
stage
status
message
timestamp
```

Do not log full private job descriptions unless necessary.

---

## 13. Error Handling

Implement simple error handling:

- Missing job title → 400.
- Missing JD → 400.
- Invalid job ID → 404.
- Worker exception → job status `failed`.
- API exception → JSON error response.

Example error response:

```json
{
  "error": "Job not found",
  "code": "JOB_NOT_FOUND"
}
```

---

## 14. Stage 1 Acceptance Criteria

Stage 1 is complete when:

1. The app runs locally.
2. User can create a job.
3. Job is stored in PostgreSQL.
4. User can view job list.
5. User can view job details.
6. User can start backend processing.
7. Worker creates process events.
8. Process events appear in the frontend.
9. Job status changes correctly.
10. Project structure supports future stages.

---

## 15. Important Non-Goals for Stage 1

Do not implement:

```text
LLM parsing
Candidate sourcing
GitHub API
LinkedIn query generation
Candidate scoring
Outreach
Search API
Deduplication
Authentication
```

These come later.

---

## 16. Instructions for Coding Agent

When implementing this stage:

1. Prioritize clean architecture.
2. Keep interfaces extensible.
3. Use typed schemas.
4. Keep frontend simple.
5. Do not overbuild.
6. Make process events reliable.
7. Make database migrations repeatable.
8. Keep each module small.
9. Use placeholder worker logic only.
10. Ensure Stage 2 can add parsing and query generation without refactoring the whole project.

---

## 17. Expected Final State

At the end of Stage 1, the system should feel like a working shell:

```text
Create JD → Start process → Watch backend stages → Completed
```

This stage creates the base on which every future sourcing feature will be built.

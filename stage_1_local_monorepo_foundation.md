# Stage 1 — Local-First Monorepo Foundation and Research Workstation Architecture

**Project:** AI Sourcing Research Workstation  
**Stage:** 1 of 5  
**Primary goal:** Build the local-first foundation for a personal sourcing tool that can run long research jobs whenever a new JD arrives.  
**Deployment assumption:** No cloud deployment. Everything runs locally on the user's machine.  
**Data assumption:** Local SQLite database, local filesystem storage, local Redis/Celery workers.  
**AI assumption:** Gemini API will be used in later stages, but Stage 1 should work without AI.

---

## 1. Stage Objective

Stage 1 creates the technical foundation for the full system.

This stage should establish:

1. A clean monorepo.
2. A local web UI.
3. A local FastAPI backend.
4. A local SQLite database.
5. SQLite WAL mode for better local reads/writes.
6. Alembic migrations.
7. Local filesystem storage under `/data`.
8. Redis + Celery for long-running local background jobs.
9. Process event tracking.
10. Research task tracking.
11. Local-only Docker Compose setup.
12. A basic job creation and processing skeleton.
13. A structure that Cursor/Codex/other coding agents can extend stage by stage.

This stage should **not** implement actual sourcing yet.

---

## 2. Final Local-First Architecture

```text
Local browser
   ↓
Next.js frontend
   ↓
FastAPI local backend
   ↓
SQLite database in WAL mode
   ↓
Redis + Celery local workers
   ↓
Local filesystem storage
   ↓
Future online connectors:
GitHub, Gemini, Search API, Hugging Face, Kaggle, Dribbble, LinkedIn manual-assisted
```

The system should behave like a **local sourcing research workstation**, not a cloud SaaS.

---

## 3. Monorepo Requirement

Use a monorepo from the start.

Recommended structure:

```text
ai-sourcing-workstation/
  README.md
  .env.example
  docker-compose.yml
  Makefile
  package.json
  pnpm-workspace.yaml

  apps/
    web/
      package.json
      next.config.ts
      tsconfig.json
      tailwind.config.ts
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
          formatters.ts

    api/
      pyproject.toml
      alembic.ini
      app/
        main.py
        core/
          config.py
          database.py
          sqlite.py
          celery_app.py
          logging.py
          paths.py
        models/
          job.py
          process_event.py
          research_task.py
          source_artifact.py
        schemas/
          job.py
          process_event.py
          research_task.py
        api/
          routes/
            health.py
            jobs.py
            process_events.py
            research_tasks.py
        services/
          job_service.py
          process_event_service.py
          research_task_service.py
          local_storage_service.py
        workers/
          tasks.py
        migrations/
          versions/

  packages/
    shared/
      README.md
      types/
        api.ts

  data/
    ai_sourcing.db
    storage/
      jobs/
      exports/
      logs/
      ai_cache/
```

### Why monorepo?

Use a monorepo because:

- Frontend and backend evolve together.
- Shared types can be maintained in one place.
- A coding agent can understand the whole project in one workspace.
- Local Docker Compose can run all services together.
- Future stages can add connectors without restructuring.

---

## 4. Selected Local Stack

| Layer | Selected Tool |
|---|---|
| Frontend | Next.js + TypeScript |
| UI | Tailwind CSS + simple reusable components |
| Backend | FastAPI |
| Backend language | Python |
| Database | SQLite |
| ORM | SQLAlchemy or SQLModel |
| Migrations | Alembic |
| Queue | Redis + Celery |
| Local storage | Local filesystem under `/data/storage` |
| AI provider later | Gemini API |
| Process tracking | SQLite tables |
| Background jobs | Celery worker |
| Runtime | Docker Compose local |
| Browser automation later | Playwright, only where allowed |

---

## 5. Why SQLite Is Selected

SQLite is selected because this is a personal local project.

It is suitable because:

```text
One user
One local machine
1–2 JDs per week
Long-running but controlled research jobs
Thousands of candidates, not millions
No cloud deployment
Easy backup by copying a DB file
Low setup overhead
```

Important design rule:

> SQLite stores structured metadata. Large raw pages, README snapshots, crawled HTML, screenshots, and exports go into local files.

---

## 6. SQLite Configuration

Enable WAL mode at app startup.

Recommended pragmas:

```sql
PRAGMA journal_mode=WAL;
PRAGMA synchronous=NORMAL;
PRAGMA foreign_keys=ON;
PRAGMA temp_store=MEMORY;
```

Reason:

- WAL allows frontend reads while workers write.
- Foreign keys preserve relational integrity.
- Local long-running research is more stable.

---

## 7. Local Storage Design

Create a local `/data` folder.

```text
data/
  ai_sourcing.db
  storage/
    jobs/
      {job_id}/
        raw_sources/
        github/
        web/
        linkedin_manual/
        huggingface/
        kaggle/
        dribbble/
        exports/
        logs/
    ai_cache/
    global_exports/
```

### Storage rule

SQLite should store:

```text
metadata
candidate records
scores
statuses
source URLs
local file paths
small JSON summaries
```

Filesystem should store:

```text
large raw API responses
HTML pages
README files
portfolio pages
screenshots
CSV exports
long logs
AI input/output cache files if large
```

---

## 8. Environment Variables

Create `.env.example`.

```env
APP_ENV=development

# Local database
DATABASE_URL=sqlite:///./data/ai_sourcing.db
SQLITE_DB_PATH=./data/ai_sourcing.db

# Local storage
LOCAL_STORAGE_PATH=./data/storage

# Backend/frontend
API_HOST=0.0.0.0
API_PORT=8000
NEXT_PUBLIC_API_URL=http://localhost:8000

# Queue
USE_REDIS=true
REDIS_URL=redis://localhost:6379/0
CELERY_CONCURRENCY=2

# AI provider, used in later stages
AI_PROVIDER=gemini
GEMINI_API_KEY=
GEMINI_MODEL_FAST=
GEMINI_MODEL_REASONING=
GEMINI_EMBEDDING_MODEL=

# API keys for later stages
GITHUB_TOKEN=
SEARCH_PROVIDER=
SEARCH_API_KEY=
HUGGINGFACE_TOKEN=
KAGGLE_USERNAME=
KAGGLE_KEY=
DRIBBBLE_ACCESS_TOKEN=

# Local research limits
MAX_CANDIDATES_PER_JOB=1000
MAX_SEARCH_QUERIES_PER_JOB=100
MAX_RESULTS_PER_QUERY=20
MAX_URLS_TO_CRAWL_PER_JOB=300
MAX_GITHUB_USERS_PER_JOB=300
MAX_REPOS_PER_GITHUB_USER=8
MAX_README_CHARS=8000
MAX_PORTFOLIO_PAGES_PER_DOMAIN=5
MAX_PARALLEL_WORKERS=2
REQUEST_DELAY_SECONDS=2
```

Model names should remain configurable instead of being hardcoded because AI provider models change over time.

---

## 9. Docker Compose Local Only

Create `docker-compose.yml`.

Services:

```text
web
api
worker
redis
```

Do **not** include cloud database services.

SQLite file should be mounted from local disk:

```text
./data:/app/data
```

Suggested service responsibilities:

### `web`

- Next.js frontend.
- Runs on `localhost:3000`.

### `api`

- FastAPI backend.
- Runs on `localhost:8000`.
- Reads and writes SQLite.
- Creates research tasks.
- Serves process events.

### `worker`

- Celery worker.
- Runs long local jobs.
- Writes process events and research task status.
- Uses same mounted `/data`.

### `redis`

- Local queue broker.
- No persistent data required initially.

---

## 10. Database Tables for Stage 1

Use SQLite-compatible schemas.

### 10.1 `jobs`

```sql
CREATE TABLE jobs (
  id TEXT PRIMARY KEY,
  title TEXT NOT NULL,
  raw_jd TEXT NOT NULL,
  role_category TEXT,
  status TEXT NOT NULL DEFAULT 'created',
  parsed_criteria_json TEXT,
  created_at TEXT NOT NULL,
  updated_at TEXT NOT NULL
);
```

### 10.2 `process_events`

```sql
CREATE TABLE process_events (
  id TEXT PRIMARY KEY,
  job_id TEXT NOT NULL,
  stage TEXT NOT NULL,
  status TEXT NOT NULL,
  message TEXT,
  records_count INTEGER DEFAULT 0,
  metadata_json TEXT,
  created_at TEXT NOT NULL,
  FOREIGN KEY(job_id) REFERENCES jobs(id) ON DELETE CASCADE
);
```

### 10.3 `research_tasks`

```sql
CREATE TABLE research_tasks (
  id TEXT PRIMARY KEY,
  job_id TEXT NOT NULL,
  task_type TEXT NOT NULL,
  source TEXT NOT NULL,
  status TEXT NOT NULL DEFAULT 'pending',
  input_json TEXT,
  output_json TEXT,
  attempts INTEGER NOT NULL DEFAULT 0,
  max_attempts INTEGER NOT NULL DEFAULT 3,
  error_message TEXT,
  started_at TEXT,
  completed_at TEXT,
  created_at TEXT NOT NULL,
  updated_at TEXT NOT NULL,
  FOREIGN KEY(job_id) REFERENCES jobs(id) ON DELETE CASCADE
);
```

### 10.4 `source_artifacts`

```sql
CREATE TABLE source_artifacts (
  id TEXT PRIMARY KEY,
  job_id TEXT NOT NULL,
  candidate_id TEXT,
  source TEXT NOT NULL,
  artifact_type TEXT NOT NULL,
  artifact_url TEXT,
  local_file_path TEXT,
  title TEXT,
  description TEXT,
  metadata_json TEXT,
  content_hash TEXT,
  created_at TEXT NOT NULL,
  FOREIGN KEY(job_id) REFERENCES jobs(id) ON DELETE CASCADE
);
```

### 10.5 `rate_limit_events`

```sql
CREATE TABLE rate_limit_events (
  id TEXT PRIMARY KEY,
  source TEXT NOT NULL,
  event_type TEXT NOT NULL,
  message TEXT,
  retry_after_seconds INTEGER,
  created_at TEXT NOT NULL
);
```

---

## 11. Required Indexes

Add indexes early.

```sql
CREATE INDEX idx_jobs_status ON jobs(status);
CREATE INDEX idx_process_events_job_id ON process_events(job_id);
CREATE INDEX idx_process_events_created_at ON process_events(created_at);
CREATE INDEX idx_research_tasks_job_id ON research_tasks(job_id);
CREATE INDEX idx_research_tasks_status ON research_tasks(status);
CREATE INDEX idx_research_tasks_source ON research_tasks(source);
CREATE INDEX idx_source_artifacts_job_id ON source_artifacts(job_id);
CREATE INDEX idx_source_artifacts_source ON source_artifacts(source);
CREATE INDEX idx_source_artifacts_content_hash ON source_artifacts(content_hash);
```

---

## 12. Backend API Requirements

Implement:

```text
GET /health
POST /jobs
GET /jobs
GET /jobs/{job_id}
POST /jobs/{job_id}/start
GET /jobs/{job_id}/process-events
GET /jobs/{job_id}/research-tasks
GET /jobs/{job_id}/artifacts
```

### POST `/jobs`

Request:

```json
{
  "title": "Python Backend Engineer",
  "raw_jd": "We need..."
}
```

Response:

```json
{
  "id": "job_uuid",
  "title": "Python Backend Engineer",
  "status": "created"
}
```

### POST `/jobs/{job_id}/start`

At Stage 1, this only runs a placeholder local research workflow.

It should:

1. Set job status to `queued`.
2. Create `JOB_QUEUED` event.
3. Create one `research_tasks` row.
4. Enqueue Celery task.
5. Return immediately.

---

## 13. Worker Requirements

Create placeholder task:

```text
run_stage_1_foundation_task(job_id)
```

It should create process events:

```text
JOB_PROCESS_STARTED
LOCAL_STORAGE_CHECK_STARTED
LOCAL_STORAGE_CHECK_COMPLETED
SQLITE_CHECK_STARTED
SQLITE_CHECK_COMPLETED
RESEARCH_TASK_CREATED
STAGE_1_COMPLETED
```

It should update job status:

```text
created → queued → running → completed
```

If error:

```text
failed
```

---

## 14. Frontend Requirements

Implement pages:

```text
/
 /jobs
 /jobs/new
 /jobs/[jobId]
 /jobs/[jobId]/process
```

### Jobs page

Show table:

```text
Title
Status
Role Category
Created At
Actions
```

### New job page

Fields:

```text
Job Title
Raw Job Description
```

### Job detail page

Show:

```text
Title
Status
Raw JD
Start Local Research button
Links:
- Process
- Research Tasks
- Artifacts
```

### Process page

Show process timeline:

```text
Stage
Status
Message
Records Count
Timestamp
```

Poll every 2–5 seconds.

---

## 15. Local Research Process Visibility

Process statuses:

```text
pending
running
completed
failed
warning
manual_action_required
paused
retrying
```

The frontend should clearly show if a task is:

```text
queued
running
failed
retrying
completed
waiting for manual action
```

This is critical because later stages will run long research jobs.

---

## 16. Error Handling

Implement:

```text
404 for missing jobs
400 for invalid input
500 for unexpected backend errors
failed process event for worker errors
research task error_message
```

Do not crash the whole app if one worker task fails.

---

## 17. Stage 1 Acceptance Criteria

Stage 1 is complete when:

1. Monorepo exists.
2. Next.js frontend runs locally.
3. FastAPI backend runs locally.
4. SQLite database is created locally.
5. WAL mode is enabled.
6. Alembic migrations work.
7. Redis and Celery run locally.
8. `/data/storage` is created automatically.
9. User can create a job.
10. User can start placeholder processing.
11. Worker writes process events.
12. Frontend shows process events.
13. Research task table works.
14. Local-only Docker Compose works.
15. No cloud database or deployment dependency exists.

---

## 18. Non-Goals for Stage 1

Do not implement:

```text
Gemini calls
JD parsing
query generation
candidate sourcing
GitHub integration
search API integration
candidate scoring
outreach
LinkedIn import
```

---

## 19. Instructions for Coding Agent

When implementing:

1. Keep the monorepo clean.
2. Use SQLite-compatible types.
3. Generate UUIDs in Python, not the DB.
4. Store JSON as text with serialization helpers.
5. Use repository/service pattern.
6. Keep all file paths relative to `LOCAL_STORAGE_PATH`.
7. Ensure worker and API use the same local DB file.
8. Add process events for every meaningful action.
9. Keep the frontend minimal.
10. Prioritize future extensibility for long-running research.

---

## 20. Expected Final State

At the end of Stage 1:

```text
Local app starts
→ User creates JD
→ Backend queues placeholder local research job
→ Worker runs task
→ SQLite stores events
→ Frontend shows progress
```

This is the base local research workstation.

# Stage 2 — Gemini JD Intelligence, Research Plan Builder, Query Generation, and LinkedIn-Assisted Manual Lane

**Project:** AI Sourcing Research Workstation  
**Stage:** 2 of 5  
**Depends on:** Stage 1  
**Primary goal:** Add AI-powered JD parsing, structured criteria extraction, platform-specific query generation, research task planning, and LinkedIn-assisted manual sourcing.

---

## 1. Stage Objective

Stage 2 converts a raw job description into an actionable research plan.

By the end of this stage, the app should:

1. Use Gemini API for JD parsing.
2. Extract structured job criteria.
3. Classify the role as developer, ML/data, designer, or other.
4. Generate source-specific search queries.
5. Generate LinkedIn native and LinkedIn X-Ray queries.
6. Build a local research plan.
7. Create source-specific `research_tasks`.
8. Display generated queries to the user.
9. Support manual candidate import.
10. Support LinkedIn-assisted import.
11. Store AI runs with input hashing for caching.

---

## 2. AI Provider Design

Do not hardcode Gemini directly inside business logic.

Create:

```text
app/services/ai/
  ai_service.py
  providers/
    gemini_provider.py
    base.py
  prompts/
    jd_parser_prompt.py
    query_generator_prompt.py
    candidate_extraction_prompt.py
```

### AI service interface

```python
class AIService:
    def parse_jd(self, raw_jd: str) -> dict: ...
    def generate_queries(self, parsed_criteria: dict) -> dict: ...
    def extract_candidate_profile(self, raw_text: str, context: dict) -> dict: ...
    def generate_outreach(self, context: dict) -> dict: ...
```

Stage 2 only needs:

```text
parse_jd
generate_queries
```

### Provider abstraction

```text
AIService
  ↓
GeminiProvider
```

Later this can support:

```text
OpenAIProvider
AnthropicProvider
LocalModelProvider
```

---

## 3. Gemini Usage in Stage 2

Gemini is used for:

```text
JD parsing
role classification
criteria extraction
query generation
LinkedIn query generation
```

Gemini is **not** used for:

```text
database logic
task orchestration
deterministic scoring
queue handling
source crawling
```

---

## 4. AI Caching

Add `ai_runs`.

```sql
CREATE TABLE ai_runs (
  id TEXT PRIMARY KEY,
  task_type TEXT NOT NULL,
  input_hash TEXT NOT NULL,
  provider TEXT NOT NULL,
  model TEXT NOT NULL,
  output_json TEXT,
  token_usage_json TEXT,
  created_at TEXT NOT NULL
);
```

Index:

```sql
CREATE INDEX idx_ai_runs_task_hash ON ai_runs(task_type, input_hash);
```

Before calling Gemini:

```text
1. Hash normalized input.
2. Check ai_runs for same task_type + input_hash.
3. Reuse cached output if available.
4. Otherwise call Gemini.
5. Store output.
```

This prevents repeated costs during local development.

---

## 5. Database Additions

### 5.1 `job_criteria`

```sql
CREATE TABLE job_criteria (
  id TEXT PRIMARY KEY,
  job_id TEXT NOT NULL,
  criterion_type TEXT NOT NULL,
  name TEXT NOT NULL,
  importance TEXT NOT NULL DEFAULT 'medium',
  is_hard_filter INTEGER NOT NULL DEFAULT 0,
  confidence REAL DEFAULT 0.0,
  created_at TEXT NOT NULL,
  FOREIGN KEY(job_id) REFERENCES jobs(id) ON DELETE CASCADE
);
```

Criterion types:

```text
must_have_skill
nice_to_have_skill
role_title
seniority
experience
location
industry
tool
domain
certification
language
work_mode
```

### 5.2 `search_queries`

```sql
CREATE TABLE search_queries (
  id TEXT PRIMARY KEY,
  job_id TEXT NOT NULL,
  source TEXT NOT NULL,
  query_type TEXT NOT NULL,
  query_text TEXT NOT NULL,
  priority INTEGER DEFAULT 1,
  created_at TEXT NOT NULL,
  FOREIGN KEY(job_id) REFERENCES jobs(id) ON DELETE CASCADE
);
```

Sources:

```text
github
web_search
linkedin_native
linkedin_xray
huggingface
kaggle
dribbble
behance
portfolio
manual
```

### 5.3 `candidates`

```sql
CREATE TABLE candidates (
  id TEXT PRIMARY KEY,
  canonical_name TEXT,
  headline TEXT,
  role_category TEXT,
  location TEXT,
  email TEXT,
  website TEXT,
  profile_json TEXT,
  created_at TEXT NOT NULL,
  updated_at TEXT NOT NULL
);
```

### 5.4 `candidate_sources`

```sql
CREATE TABLE candidate_sources (
  id TEXT PRIMARY KEY,
  candidate_id TEXT NOT NULL,
  source TEXT NOT NULL,
  source_id TEXT,
  source_url TEXT,
  raw_text TEXT,
  raw_json TEXT,
  local_file_path TEXT,
  permission_type TEXT DEFAULT 'manual_user_import',
  collected_at TEXT NOT NULL,
  FOREIGN KEY(candidate_id) REFERENCES candidates(id) ON DELETE CASCADE
);
```

### 5.5 `lead_status`

```sql
CREATE TABLE lead_status (
  id TEXT PRIMARY KEY,
  job_id TEXT NOT NULL,
  candidate_id TEXT NOT NULL,
  status TEXT NOT NULL DEFAULT 'new',
  notes TEXT,
  last_contacted_at TEXT,
  contact_count INTEGER NOT NULL DEFAULT 0,
  do_not_contact INTEGER NOT NULL DEFAULT 0,
  updated_at TEXT NOT NULL,
  FOREIGN KEY(job_id) REFERENCES jobs(id) ON DELETE CASCADE,
  FOREIGN KEY(candidate_id) REFERENCES candidates(id) ON DELETE CASCADE
);
```

Indexes:

```sql
CREATE INDEX idx_job_criteria_job_id ON job_criteria(job_id);
CREATE INDEX idx_search_queries_job_source ON search_queries(job_id, source);
CREATE INDEX idx_candidates_role_category ON candidates(role_category);
CREATE INDEX idx_candidate_sources_source ON candidate_sources(source);
CREATE INDEX idx_candidate_sources_url ON candidate_sources(source_url);
CREATE INDEX idx_lead_status_job_candidate ON lead_status(job_id, candidate_id);
```

---

## 6. JD Parser

Create:

```text
app/services/jd_parser_service.py
```

Responsibilities:

```text
Accept raw JD
Call AIService.parse_jd
Validate parsed JSON
Save parsed criteria to jobs.parsed_criteria_json
Save rows to job_criteria
Create process events
Add warnings if JD is weak
```

### Expected parsed output

```json
{
  "role_category": "developer",
  "role_titles": ["Backend Engineer", "Python Developer"],
  "seniority": "mid",
  "min_experience_years": 3,
  "max_experience_years": null,
  "must_have_skills": ["Python", "FastAPI", "PostgreSQL"],
  "nice_to_have_skills": ["AWS", "Docker"],
  "locations": ["India"],
  "work_mode": "remote",
  "industries": [],
  "tools": [],
  "domains": [],
  "hard_filters": ["India", "3+ years"],
  "warnings": []
}
```

### Prompt rules

The prompt must say:

```text
Return valid JSON only.
Do not invent requirements.
Separate must-have from nice-to-have.
Classify as developer, ml_data, designer, or other.
Do not extract protected traits.
Flag unclear or contradictory requirements.
```

Protected traits to exclude:

```text
age
gender
religion
caste
race
ethnicity
health
family status
political views
sexuality
```

---

## 7. Query Generator

Create:

```text
app/services/query_generator_service.py
```

It should generate:

```text
GitHub queries
Public web search queries
LinkedIn native queries
LinkedIn Google X-Ray queries
Hugging Face queries
Kaggle queries
Dribbble queries
Behance queries
Portfolio queries
```

### Example output

```json
{
  "github": [
    "language:Python FastAPI PostgreSQL location:India",
    "FastAPI PostgreSQL Docker in:readme"
  ],
  "web_search": [
    "\"Python Backend Engineer\" \"FastAPI\" \"India\" \"GitHub\"",
    "\"FastAPI\" \"PostgreSQL\" \"portfolio\" \"India\""
  ],
  "linkedin_native": [
    "(\"Backend Engineer\" OR \"Python Developer\") AND Python AND FastAPI AND India"
  ],
  "linkedin_xray": [
    "site:linkedin.com/in \"Backend Engineer\" \"Python\" \"FastAPI\" \"India\""
  ],
  "huggingface": [],
  "kaggle": [],
  "dribbble": []
}
```

---

## 8. Research Plan Builder

Create:

```text
app/services/research_plan_service.py
```

Purpose:

Convert parsed criteria and queries into local `research_tasks`.

### Task types

```text
github_search
web_search
linkedin_manual
huggingface_search
kaggle_search
dribbble_search
manual_import_wait
```

### Logic

For `developer` jobs:

```text
Create GitHub tasks
Create web search tasks
Create LinkedIn manual task
Create manual import task
```

For `ml_data` jobs:

```text
Create GitHub tasks
Create Hugging Face tasks
Create Kaggle tasks
Create web search tasks
Create LinkedIn manual task
```

For `designer` jobs:

```text
Create Dribbble tasks
Create web/portfolio tasks
Create Behance query tasks
Create LinkedIn manual task
```

For `other` jobs:

```text
Create web search tasks
Create LinkedIn manual task
Create manual import task
```

### Important

Stage 2 should create tasks but not execute automated source tasks yet, except placeholder/manual tasks.

---

## 9. LinkedIn-Assisted Lane

LinkedIn remains included.

But Stage 2 should not automate LinkedIn scraping.

### LinkedIn process

```text
1. System generates LinkedIn queries.
2. User searches LinkedIn manually.
3. User opens promising profiles manually.
4. User copies profile URL and visible relevant text.
5. User imports candidate into the app.
6. App stores source = linkedin_manual.
```

### LinkedIn query types

```text
linkedin_native
linkedin_xray
title_based
skill_based
location_based
alternative_title
company_based
```

### UI helper text

Show:

```text
Use these queries manually on LinkedIn or Google.
Paste only profiles you intentionally select.
The system will analyze and score them in later stages.
```

---

## 10. Manual Candidate Import

Create:

```text
POST /jobs/{job_id}/manual-candidates
```

Request:

```json
{
  "name": "Amit Kumar",
  "headline": "Backend Engineer",
  "location": "India",
  "email": null,
  "website": "https://amit.dev",
  "source": "linkedin_manual",
  "source_url": "https://linkedin.com/in/example",
  "raw_text": "Pasted profile text...",
  "notes": "Looks relevant"
}
```

It should:

```text
Create candidate
Create candidate_source
Create lead_status
Create source_artifact if raw text is large
Create process event
```

If `raw_text` is long, store it in local file and save path.

---

## 11. Backend API Additions

```text
POST /jobs/{job_id}/parse
GET /jobs/{job_id}/criteria
PUT /jobs/{job_id}/criteria
POST /jobs/{job_id}/generate-queries
GET /jobs/{job_id}/search-queries
POST /jobs/{job_id}/build-research-plan
GET /jobs/{job_id}/research-plan
POST /jobs/{job_id}/manual-candidates
GET /jobs/{job_id}/candidates
```

---

## 12. Worker Process

Update main job processing to run:

```text
JD_PARSE_STARTED
JD_PARSE_COMPLETED
CRITERIA_SAVED
QUERY_GENERATION_STARTED
QUERY_GENERATION_COMPLETED
RESEARCH_PLAN_STARTED
RESEARCH_PLAN_COMPLETED
LINKEDIN_MANUAL_TASK_READY
MANUAL_IMPORT_READY
STAGE_2_COMPLETED
```

Job status:

```text
created → queued → parsing → planning → ready_for_research
```

---

## 13. Frontend Requirements

Add tabs to job detail:

```text
Overview
Criteria
Queries
Research Plan
LinkedIn
Manual Import
Candidates
Process
```

### Criteria tab

Show editable:

```text
Role category
Seniority
Experience
Location
Work mode
Must-have skills
Nice-to-have skills
Tools
Domains
Warnings
```

### Queries tab

Group by:

```text
GitHub
Web Search
LinkedIn Native
LinkedIn X-Ray
Hugging Face
Kaggle
Dribbble
Behance
```

Each query has:

```text
Copy button
Source
Priority
Query type
```

### Research Plan tab

Show created research tasks:

```text
Task Type
Source
Status
Input
Attempts
Created At
```

### LinkedIn tab

Show:

```text
LinkedIn manual instructions
Native LinkedIn queries
Google X-Ray LinkedIn queries
Manual import CTA
```

### Manual Import tab

Fields:

```text
Candidate Name
Headline
Location
Email
Website
Source
Source URL
Raw Profile Text
Notes
```

---

## 14. Stage 2 Acceptance Criteria

Stage 2 is complete when:

1. Gemini integration works through AIService.
2. JD parsing creates structured criteria.
3. Criteria are saved in SQLite.
4. User can view and edit criteria.
5. Queries are generated and saved.
6. LinkedIn queries are visible and copyable.
7. Research plan tasks are created locally.
8. Manual candidate import works.
9. LinkedIn manual import works.
10. AI runs are cached by input hash.
11. Process events show all steps.
12. No cloud deployment or cloud DB is required.

---

## 15. Non-Goals

Do not implement:

```text
GitHub API execution
web search execution
Hugging Face execution
Kaggle execution
Dribbble execution
advanced scoring
deduplication
outreach
browser automation
```

---

## 16. Instructions for Coding Agent

1. Keep Gemini behind AIService.
2. Do not hardcode model names.
3. Cache AI runs.
4. Validate AI JSON output.
5. Store JSON as text in SQLite.
6. Create research tasks but do not execute them yet.
7. Keep LinkedIn manual-assisted only.
8. Make queries easy to copy.
9. Keep local file storage for large text.
10. Preserve Stage 1 architecture.

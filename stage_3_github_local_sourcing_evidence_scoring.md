# Stage 3 — Automated GitHub Developer Sourcing, Local Artifacts, Evidence Extraction, and Scoring

**Project:** AI Sourcing Research Workstation  
**Stage:** 3 of 5  
**Depends on:** Stage 1 and Stage 2  
**Primary goal:** Implement the first fully automated sourcing lane: GitHub developer sourcing.

---

## 1. Stage Objective

Stage 3 turns the local research workstation into a real candidate lead generator for developers.

By the end of this stage:

1. GitHub search tasks are executed locally.
2. GitHub users are discovered from generated queries.
3. Candidate profiles are enriched using GitHub API data.
4. Repositories are analyzed.
5. README files and metadata are stored locally.
6. Candidates are normalized into SQLite.
7. Evidence is extracted against JD criteria.
8. Developer scoring is computed.
9. Results are shown in ranked candidate dashboard.
10. Research tasks support retry, checkpointing, and resume.

---

## 2. Scope

### Included

```text
GitHub connector
GitHub task execution
GitHub profile enrichment
Repository analysis
README extraction
Local artifact storage
Evidence extraction
Developer scoring
Candidate detail page
Candidate review status update
Checkpointing/retry
```

### Excluded

```text
Public web search
Hugging Face
Kaggle
Dribbble
Designer sourcing
Outreach drafts
Advanced cross-source deduplication
LinkedIn automation
```

---

## 3. GitHub Source Lane Flow

```text
Stored GitHub research_tasks
   ↓
Celery worker executes github_search
   ↓
GitHub API returns users/repos
   ↓
Create candidate records
   ↓
Fetch user profile
   ↓
Fetch top repositories
   ↓
Fetch README and repo languages/topics
   ↓
Store raw artifacts locally
   ↓
Extract evidence
   ↓
Score candidate
   ↓
Show ranked results
```

---

## 4. GitHub Connector

Create:

```text
app/connectors/github_connector.py
```

### Responsibilities

```text
Run GitHub search queries
Fetch user profile
Fetch user repositories
Fetch repository languages
Fetch repository topics
Fetch README files
Detect rate limits
Return structured data
```

### Functions

```python
class GitHubConnector:
    def search_users(self, query: str, limit: int) -> list[dict]: ...
    def search_repositories(self, query: str, limit: int) -> list[dict]: ...
    def get_user(self, username: str) -> dict: ...
    def get_user_repos(self, username: str, limit: int) -> list[dict]: ...
    def get_repo_languages(self, owner: str, repo: str) -> dict: ...
    def get_repo_readme(self, owner: str, repo: str) -> str | None: ...
```

---

## 5. GitHub Data Collection Limits

Use environment values:

```env
MAX_GITHUB_USERS_PER_JOB=300
MAX_REPOS_PER_GITHUB_USER=8
MAX_README_CHARS=8000
REQUEST_DELAY_SECONDS=2
```

Default per job:

```text
Max GitHub queries: 20
Max users per query: 30
Max enriched candidates: 300
Max repos per user: 8
Max README chars per repo: 8000
```

---

## 6. GitHub API Authentication

Use:

```env
GITHUB_TOKEN=
```

If missing:

```text
Allow unauthenticated mode but warn user.
Reduce limits automatically.
Create process warning event.
```

Process event:

```text
GITHUB_TOKEN_MISSING
```

---

## 7. Local Artifact Storage for GitHub

Store raw GitHub data under:

```text
/data/storage/jobs/{job_id}/github/
```

Suggested files:

```text
github_search_{query_hash}.json
users/{username}/profile.json
users/{username}/repos.json
users/{username}/repos/{repo_name}/readme.md
users/{username}/repos/{repo_name}/languages.json
```

SQLite `source_artifacts` rows should point to these files.

Example `source_artifacts`:

```json
{
  "source": "github",
  "artifact_type": "readme",
  "artifact_url": "https://github.com/user/repo",
  "local_file_path": "data/storage/jobs/job_123/github/users/user/repos/repo/readme.md",
  "content_hash": "sha256..."
}
```

---

## 8. Database Additions

### 8.1 `candidate_evidence`

```sql
CREATE TABLE candidate_evidence (
  id TEXT PRIMARY KEY,
  candidate_id TEXT NOT NULL,
  job_id TEXT NOT NULL,
  requirement TEXT NOT NULL,
  evidence_type TEXT NOT NULL,
  source TEXT NOT NULL,
  source_url TEXT,
  local_file_path TEXT,
  text_snippet TEXT,
  confidence REAL DEFAULT 0.0,
  created_at TEXT NOT NULL,
  FOREIGN KEY(candidate_id) REFERENCES candidates(id) ON DELETE CASCADE,
  FOREIGN KEY(job_id) REFERENCES jobs(id) ON DELETE CASCADE
);
```

### 8.2 `candidate_scores`

```sql
CREATE TABLE candidate_scores (
  id TEXT PRIMARY KEY,
  job_id TEXT NOT NULL,
  candidate_id TEXT NOT NULL,
  match_score REAL NOT NULL DEFAULT 0,
  confidence_score REAL NOT NULL DEFAULT 0,
  contactability_score REAL NOT NULL DEFAULT 0,
  source_quality_score REAL NOT NULL DEFAULT 0,
  recency_score REAL NOT NULL DEFAULT 0,
  final_action_score REAL NOT NULL DEFAULT 0,
  explanation TEXT,
  missing_requirements_json TEXT,
  created_at TEXT NOT NULL,
  FOREIGN KEY(job_id) REFERENCES jobs(id) ON DELETE CASCADE,
  FOREIGN KEY(candidate_id) REFERENCES candidates(id) ON DELETE CASCADE
);
```

Indexes:

```sql
CREATE INDEX idx_candidate_evidence_job_candidate ON candidate_evidence(job_id, candidate_id);
CREATE INDEX idx_candidate_scores_job_id ON candidate_scores(job_id);
CREATE INDEX idx_candidate_scores_candidate_id ON candidate_scores(candidate_id);
CREATE INDEX idx_candidate_scores_final_action_score ON candidate_scores(final_action_score);
```

---

## 9. GitHub Sourcing Service

Create:

```text
app/services/sourcing/github_sourcing_service.py
```

Responsibilities:

```text
Load pending github_search research_tasks
Execute search
Create search run events
Create raw artifacts
Create or update candidate records
Create candidate_sources
Enqueue enrichment tasks
Update research_task checkpoint
```

---

## 10. GitHub Enrichment Service

Create:

```text
app/services/enrichment/github_enrichment_service.py
```

Responsibilities:

```text
Fetch user profile
Fetch repositories
Select relevant repositories
Fetch README/languages/topics
Store artifacts locally
Update candidate profile_json
Create source_artifacts
```

### Repo selection logic

Prefer repos that:

```text
match must-have skills
match nice-to-have skills
are recently pushed
are not forks
have meaningful README
have relevant language
have relevant topics
```

---

## 11. Candidate Normalization

Normalize GitHub data into candidate format:

```json
{
  "canonical_name": "Amit Kumar",
  "headline": "GitHub Developer Profile",
  "role_category": "developer",
  "location": "India",
  "email": "public@example.com",
  "website": "https://amit.dev",
  "profile_json": {
    "github": {
      "login": "amit",
      "url": "https://github.com/amit",
      "bio": "Backend developer",
      "company": null,
      "public_repos": 42,
      "followers": 120
    },
    "skills_detected": ["Python", "FastAPI", "PostgreSQL"],
    "repos_analyzed": []
  }
}
```

---

## 12. Lightweight Deduplication in Stage 3

Auto-dedupe on:

```text
same GitHub username
same GitHub profile URL
same public email
same website
```

If existing candidate found:

```text
update candidate profile
add source data
do not create duplicate
```

Do not implement fuzzy dedupe until Stage 4.

---

## 13. Evidence Extraction

Create:

```text
app/services/evidence/evidence_service.py
```

Evidence should be extracted using code first.

Check:

```text
repo language
repo topics
repo description
README text
GitHub bio
portfolio URL
location field
```

Evidence types:

```text
repo_language
repo_topic
repo_description
readme_match
profile_bio
location_match
contact_evidence
recent_activity
```

Each evidence row should include:

```text
requirement
evidence_type
source
source_url
local_file_path
text_snippet
confidence
```

---

## 14. Developer Scoring

Create:

```text
app/services/scoring/developer_scoring_service.py
```

### Score components

| Component | Points |
|---|---:|
| Must-have skill evidence | 35 |
| Relevant project evidence | 20 |
| Role/title similarity | 10 |
| Recent activity | 10 |
| Location match | 10 |
| Nice-to-have skills | 10 |
| Contactability | 5 |

### Final action score

Use:

```text
final_action_score =
  0.65 * match_score
+ 0.20 * confidence_score
+ 0.10 * contactability_score
+ 0.05 * recency_score
```

---

## 15. Research Task Checkpointing

Each task should update:

```text
status
attempts
started_at
completed_at
output_json
error_message
updated_at
```

Task statuses:

```text
pending
running
completed
failed
retrying
paused
```

If rate-limited:

```text
status = retrying
create rate_limit_event
respect retry_after if available
```

---

## 16. Worker Tasks

Add Celery tasks:

```text
run_github_search_task(task_id)
run_github_enrichment_task(job_id, candidate_id)
run_github_evidence_task(job_id, candidate_id)
run_developer_scoring_task(job_id, candidate_id)
run_github_sourcing_pipeline(job_id)
```

Process events:

```text
GITHUB_SOURCING_STARTED
GITHUB_QUERY_STARTED
GITHUB_QUERY_COMPLETED
GITHUB_PROFILE_ENRICHMENT_STARTED
GITHUB_PROFILE_ENRICHMENT_COMPLETED
GITHUB_ARTIFACTS_SAVED
EVIDENCE_EXTRACTION_STARTED
EVIDENCE_EXTRACTION_COMPLETED
DEVELOPER_SCORING_STARTED
DEVELOPER_SCORING_COMPLETED
STAGE_3_COMPLETED
```

---

## 17. Backend API Additions

```text
POST /jobs/{job_id}/run-github-sourcing
GET /jobs/{job_id}/candidates
GET /jobs/{job_id}/candidates/{candidate_id}
GET /jobs/{job_id}/candidates/{candidate_id}/evidence
GET /jobs/{job_id}/candidates/{candidate_id}/score
PATCH /jobs/{job_id}/candidates/{candidate_id}/status
POST /research-tasks/{task_id}/retry
```

---

## 18. Frontend Changes

### Job overview

Add:

```text
Run GitHub Sourcing
Resume Failed GitHub Tasks
View GitHub Artifacts
```

### Candidate table

Columns:

```text
Name
Headline
Location
Sources
Match Score
Confidence
Contactability
Evidence Count
Status
Actions
```

Sort by:

```text
final_action_score descending
```

### Candidate detail

Sections:

```text
Summary
GitHub Profile
Repositories Analyzed
Matched Requirements
Missing Requirements
Evidence Table
Score Breakdown
Artifacts
Review Actions
Notes
```

---

## 19. Stage 3 Acceptance Criteria

Stage 3 is complete when:

1. GitHub research tasks run locally.
2. GitHub profiles are discovered.
3. Repos and READMEs are fetched within configured limits.
4. Raw artifacts are stored locally.
5. Candidate records are created.
6. Candidate sources are stored.
7. Evidence rows are created.
8. Developer scoring works.
9. Candidate results are ranked.
10. Candidate detail page shows evidence and artifacts.
11. Failed/rate-limited tasks are tracked.
12. Tasks can be retried.
13. SQLite remains stable under local worker usage.

---

## 20. Non-Goals

Do not implement:

```text
search API public web lane
Hugging Face
Kaggle
Dribbble
designer scoring
outreach
email sending
LinkedIn scraping
```

---

## 21. Instructions for Coding Agent

1. Do not over-fetch GitHub data.
2. Respect configured limits.
3. Store large raw data in files, not SQLite.
4. Create artifacts with content hashes.
5. Update research task status frequently.
6. Keep scoring deterministic.
7. Use evidence-only explanations.
8. Do not call Gemini for every repo unless needed.
9. Keep local runs resumable.
10. Preserve manual LinkedIn candidates from Stage 2.

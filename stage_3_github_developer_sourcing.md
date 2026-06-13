# Stage 3 — Automated Developer Sourcing with GitHub, Candidate Enrichment, Evidence, and Scoring

**Project:** AI Sourcing Tool  
**Stage:** 3 of 5  
**Depends on:** Stage 1 and Stage 2  
**Purpose:** Add the first real automated sourcing lane: developers from GitHub.  
**Primary outcome:** The system can automatically search GitHub for developer candidates, enrich their profiles, extract evidence, score candidates against the JD, and show ranked results.

---

## 1. Stage Objective

Stage 3 turns the tool into a working automated sourcing system for developers.

By the end of this stage:

1. The system runs generated GitHub queries.
2. It collects GitHub users and repositories.
3. It enriches candidates using profile, repo, language, topic, and README data.
4. It normalizes GitHub profiles into candidate records.
5. It extracts evidence for job requirements.
6. It creates developer-specific scores.
7. It displays ranked candidates.
8. It supports review actions such as shortlist/reject.
9. It stores source data and evidence for auditability.

---

## 2. Scope

### Included

```text
GitHub API connector
GitHub search execution
Profile enrichment
Repository analysis
README extraction
Developer candidate normalization
Evidence extraction
Developer scoring
Candidate score table
Candidate detail page
Basic candidate review workflow
```

### Excluded

```text
Hugging Face
Kaggle
Dribbble
Full web crawling
Email enrichment
Gmail/outreach integration
Advanced deduplication across all sources
```

---

## 3. High-Level Flow

```text
Job criteria from Stage 2
   ↓
GitHub queries from Stage 2
   ↓
GitHub Sourcing Agent
   ↓
Raw GitHub users/repos
   ↓
Profile Enrichment Agent
   ↓
Candidate Normalization
   ↓
Evidence Extraction
   ↓
Developer Scoring
   ↓
Ranked Candidate Dashboard
```

---

## 4. GitHub Connector Design

Create:

```text
app/connectors/github_connector.py
```

Responsibilities:

```text
Run GitHub search queries
Fetch user profile
Fetch user repositories
Fetch repository languages
Fetch repository topics
Fetch README files
Fetch recent activity if feasible
Return raw structured data
```

---

## 5. GitHub API Inputs

Use generated `search_queries` from Stage 2 where:

```text
source = github
```

Example queries:

```text
language:Python FastAPI PostgreSQL location:India
FastAPI PostgreSQL AWS in:readme
topic:fastapi location:India
```

Important: GitHub search syntax is limited. The query generator may create imperfect queries. The connector should sanitize or adapt queries before execution.

---

## 6. GitHub Data to Collect

### 6.1 User Profile

Collect:

```text
login
id
html_url
name
company
blog
location
email
bio
twitter_username
public_repos
followers
following
created_at
updated_at
```

### 6.2 Repositories

For each candidate, collect top relevant repos.

Fields:

```text
name
full_name
html_url
description
language
topics
stargazers_count
forks_count
created_at
updated_at
pushed_at
readme_text
languages
```

Limit:

```text
Max repos per user: 10 initially
Prefer recently updated repos
Prefer repos matching job skills
Ignore forks unless highly relevant
```

---

## 7. GitHub Rate Limit Strategy

Add:

```text
GITHUB_TOKEN
```

Use authenticated requests when available.

Design considerations:

```text
Cache responses where possible
Avoid fetching too many repos per user
Limit raw candidate collection per job
Add retry handling
Add rate limit handling
Store API error events
```

Recommended first limits:

```text
Max GitHub queries per job: 10
Max raw users per query: 20
Max users enriched per job: 80
Max repos analyzed per user: 10
Max README characters per repo: 8000
```

---

## 8. Database Changes

### 8.1 `search_runs`

If not already created, ensure it exists.

```sql
CREATE TABLE search_runs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  job_id UUID NOT NULL REFERENCES jobs(id) ON DELETE CASCADE,
  source TEXT NOT NULL,
  query TEXT NOT NULL,
  status TEXT NOT NULL DEFAULT 'pending',
  records_found INTEGER DEFAULT 0,
  records_processed INTEGER DEFAULT 0,
  error_message TEXT,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  completed_at TIMESTAMPTZ
);
```

---

### 8.2 `candidate_evidence`

```sql
CREATE TABLE candidate_evidence (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  candidate_id UUID NOT NULL REFERENCES candidates(id) ON DELETE CASCADE,
  job_id UUID NOT NULL REFERENCES jobs(id) ON DELETE CASCADE,
  requirement TEXT NOT NULL,
  evidence_type TEXT NOT NULL,
  source TEXT NOT NULL,
  source_url TEXT,
  text_snippet TEXT,
  confidence NUMERIC DEFAULT 0.0,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

### 8.3 `candidate_scores`

```sql
CREATE TABLE candidate_scores (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  job_id UUID NOT NULL REFERENCES jobs(id) ON DELETE CASCADE,
  candidate_id UUID NOT NULL REFERENCES candidates(id) ON DELETE CASCADE,
  match_score NUMERIC NOT NULL DEFAULT 0,
  confidence_score NUMERIC NOT NULL DEFAULT 0,
  contactability_score NUMERIC NOT NULL DEFAULT 0,
  source_quality_score NUMERIC NOT NULL DEFAULT 0,
  recency_score NUMERIC NOT NULL DEFAULT 0,
  final_action_score NUMERIC NOT NULL DEFAULT 0,
  explanation TEXT,
  missing_requirements_json JSONB,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## 9. Services to Add

### 9.1 GitHub Sourcing Service

Create:

```text
app/services/github_sourcing_service.py
```

Responsibilities:

```text
Load GitHub queries for job
Run each query
Create search_runs
Collect raw GitHub users
Create/update candidate records
Create candidate_sources
Trigger enrichment
```

---

### 9.2 GitHub Enrichment Service

Create:

```text
app/services/github_enrichment_service.py
```

Responsibilities:

```text
Fetch user profile
Fetch repositories
Fetch README content
Extract languages/topics
Build source profile JSON
Update candidate profile_json
```

---

### 9.3 Evidence Extraction Service

Create:

```text
app/services/evidence_service.py
```

Responsibilities:

```text
Compare job criteria against candidate source data
Find skill evidence in repo languages, topics, descriptions, README
Create candidate_evidence rows
Identify missing requirements
```

Evidence sources:

```text
repo_language
repo_topic
repo_description
readme_text
profile_bio
profile_location
profile_website
```

---

### 9.4 Developer Scoring Service

Create:

```text
app/services/scoring/developer_scoring_service.py
```

Responsibilities:

```text
Compute developer match score
Compute confidence score
Compute contactability score
Compute source quality score
Compute recency score
Compute final action score
Generate score explanation
```

---

## 10. Developer Scoring Formula

Use deterministic scoring.

### 10.1 Match Score

Total: 100

| Component | Points |
|---|---:|
| Must-have skill evidence | 35 |
| Relevant project evidence | 20 |
| Role/title similarity | 10 |
| Recent activity | 10 |
| Location match | 10 |
| Nice-to-have skills | 10 |
| Contactability | 5 |

---

### 10.2 Must-Have Skill Score

```text
matched_must_have_skills / total_must_have_skills * 35
```

Evidence can come from:

```text
repo language
repo topic
README
repo description
profile bio
```

---

### 10.3 Relevant Project Evidence Score

Award points for:

```text
Repo directly related to role
README contains relevant architecture/framework words
Repo not forked
Repo has recent commits
Repo has meaningful description
Repo has multiple matching technologies
```

Suggested:

```text
0–5 weak evidence
6–12 moderate evidence
13–20 strong evidence
```

---

### 10.4 Recent Activity Score

Use `pushed_at` and `updated_at`.

```text
10 = activity in last 90 days
7 = activity in last 180 days
4 = activity in last 365 days
1 = older activity
0 = no usable activity
```

---

### 10.5 Contactability Score

```text
5 = public email available
4 = website/portfolio available
3 = GitHub profile only
1 = weak contact path
0 = no contact path
```

---

### 10.6 Confidence Score

Confidence should depend on evidence count and quality.

```text
High confidence:
- Multiple evidence rows
- Source URLs available
- Recent activity
- Clear location/contact

Low confidence:
- Only keyword match
- No README evidence
- Sparse profile
```

---

## 11. Candidate Normalization

For GitHub candidates, normalize into:

```json
{
  "name": "Amit Kumar",
  "headline": "GitHub developer profile",
  "role_category": "developer",
  "location": "India",
  "email": "public@example.com",
  "website": "https://amit.dev",
  "profile_json": {
    "github": {
      "login": "amit",
      "url": "https://github.com/amit",
      "bio": "Backend engineer...",
      "repos_analyzed": []
    },
    "skills_detected": ["Python", "FastAPI", "PostgreSQL"],
    "activity_summary": {
      "recent_pushes": 4,
      "last_activity_at": "timestamp"
    }
  }
}
```

---

## 12. Deduplication for Stage 3

Implement light deduplication only.

Auto-match if:

```text
same GitHub source_id
same GitHub URL
same email
```

Do not implement fuzzy cross-source deduplication yet.

If candidate already exists with same GitHub URL:

```text
Update candidate_sources
Update profile_json
Do not create duplicate candidate
```

---

## 13. Worker Process

Add a new Celery task:

```text
run_github_sourcing(job_id)
```

Process events:

```text
GITHUB_SOURCING_STARTED
GITHUB_QUERY_STARTED
GITHUB_QUERY_COMPLETED
GITHUB_PROFILE_ENRICHMENT_STARTED
GITHUB_PROFILE_ENRICHMENT_COMPLETED
EVIDENCE_EXTRACTION_STARTED
EVIDENCE_EXTRACTION_COMPLETED
DEVELOPER_SCORING_STARTED
DEVELOPER_SCORING_COMPLETED
STAGE_3_COMPLETED
```

If failed:

```text
GITHUB_SOURCING_FAILED
```

---

## 14. Backend API Additions

### 14.1 POST `/jobs/{job_id}/run-github-sourcing`

Starts GitHub sourcing worker.

Response:

```json
{
  "job_id": "uuid",
  "status": "github_sourcing_queued"
}
```

---

### 14.2 GET `/jobs/{job_id}/candidates`

Update response to include scores.

```json
[
  {
    "candidate_id": "uuid",
    "name": "Amit Kumar",
    "headline": "Backend Developer",
    "location": "India",
    "sources": ["github"],
    "match_score": 82,
    "confidence_score": 76,
    "contactability_score": 4,
    "status": "new"
  }
]
```

---

### 14.3 GET `/jobs/{job_id}/candidates/{candidate_id}`

Return candidate detail.

Include:

```text
candidate profile
sources
scores
evidence
missing requirements
lead status
```

---

### 14.4 PATCH `/jobs/{job_id}/candidates/{candidate_id}/status`

Request:

```json
{
  "status": "shortlisted",
  "notes": "Strong FastAPI evidence"
}
```

---

## 15. Frontend Changes

### 15.1 Job Overview

Add button:

```text
Run GitHub Sourcing
```

Show warning if:

```text
No GitHub queries exist
No GitHub token configured
Role category is not developer/ml_data
```

---

### 15.2 Process View

Show GitHub sourcing stages.

Example:

```text
GitHub Sourcing
Status: Running
Queries: 8
Raw profiles found: 93
Profiles enriched: 41
Candidates scored: 36
```

---

### 15.3 Candidate Results Table

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

Filters:

```text
Score >= 70
Has email
Has website
Location match
Missing must-have
Source = GitHub
```

---

### 15.4 Candidate Detail Page

Sections:

```text
Summary
Score Breakdown
Matched Requirements
Missing Requirements
Evidence
GitHub Profile
Repositories Analyzed
Contact Info
Review Actions
Notes
```

Evidence table:

| Requirement | Evidence Type | Source | Snippet | Confidence |
|---|---|---|---|---|

Repository table:

| Repo | Language | Topics | Last Push | Relevance |
|---|---|---|---|---|

---

## 16. Explanation Generation

At Stage 3, explanations can be deterministic.

Example:

```text
Strong developer match. Candidate has evidence for Python, FastAPI, and PostgreSQL across GitHub repositories. Recent activity found within 90 days. Location appears to match India. AWS evidence was not found.
```

Do not require an LLM for explanation unless already available.

---

## 17. Stage 3 Acceptance Criteria

Stage 3 is complete when:

1. User can run GitHub sourcing for a job.
2. System executes stored GitHub queries.
3. GitHub users are collected.
4. GitHub profiles and repos are enriched.
5. Candidate records are created.
6. Candidate sources are stored.
7. Evidence rows are created.
8. Developer scores are created.
9. Ranked candidate table is visible.
10. Candidate detail page shows evidence and score breakdown.
11. User can shortlist or reject candidates.
12. Process events show sourcing progress.

---

## 18. Non-Goals for Stage 3

Do not implement:

```text
Hugging Face
Kaggle
Dribbble
Email enrichment
Bulk outreach
Advanced cross-source deduplication
Chrome extension
LinkedIn scraping
```

---

## 19. Instructions for Coding Agent

When implementing Stage 3:

1. Keep GitHub connector isolated.
2. Respect API rate limits.
3. Store raw source data separately from normalized candidates.
4. Do not over-fetch repositories.
5. Use deterministic scoring first.
6. Always create evidence rows for matched requirements.
7. Show missing requirements clearly.
8. Avoid hallucinated explanations.
9. Preserve manual candidates from Stage 2.
10. Make Stage 4 easy to add new source connectors.

---

## 20. Expected Final State

At the end of Stage 3:

```text
JD → Criteria → Queries → GitHub sourcing → Candidate enrichment → Evidence → Scores → Ranked developer leads
```

This is the first stage where the tool generates automated candidate leads.

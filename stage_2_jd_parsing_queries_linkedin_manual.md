# Stage 2 — JD Parsing, Criteria Extraction, Query Generation, and LinkedIn-Assisted Workflow

**Project:** AI Sourcing Tool  
**Stage:** 2 of 5  
**Depends on:** Stage 1  
**Purpose:** Add the intelligence layer for understanding job descriptions and generating source-specific sourcing strategies.  
**Primary outcome:** A user can upload/paste a JD, extract structured criteria, edit criteria, generate queries for GitHub, public web, and LinkedIn-assisted search, and manually import candidates.

---

## 1. Stage Objective

Stage 2 turns the app from a simple job tracker into a sourcing preparation tool.

By the end of this stage, the system should:

1. Parse job descriptions into structured criteria.
2. Classify the role category.
3. Extract must-have and nice-to-have skills.
4. Generate search queries for different platforms.
5. Generate LinkedIn native and Google X-Ray queries.
6. Display editable criteria to the user.
7. Add a manual candidate import flow.
8. Add a LinkedIn-assisted import flow.
9. Store imported candidates in the database.
10. Prepare data structures needed for automated sourcing in Stage 3.

---

## 2. Scope

### Included

```text
JD parsing
Role classification
Criteria extraction
Search query generation
LinkedIn query generation
Manual candidate import
LinkedIn manual import
Candidate base schema
Basic process stages
Editable criteria UI
```

### Excluded

```text
GitHub automated sourcing
Search API connector
Hugging Face connector
Kaggle connector
Dribbble connector
Advanced scoring
Deduplication
Outreach
```

---

## 3. New System Flow

```text
Create Job
   ↓
Start Processing
   ↓
Parse JD
   ↓
Extract Criteria
   ↓
Classify Role
   ↓
Generate Queries
   ↓
Show Editable Criteria
   ↓
Show LinkedIn Manual Queries
   ↓
Allow Manual Candidate Import
```

---

## 4. Role Categories

The system should classify jobs into one of:

```text
developer
ml_data
designer
other
```

### 4.1 Developer Indicators

Examples:

```text
backend
frontend
full-stack
software engineer
devops
platform
mobile
react
node
python
java
go
kubernetes
aws
api
database
```

### 4.2 ML/Data Indicators

Examples:

```text
machine learning
data scientist
data engineer
mlops
pytorch
tensorflow
transformers
llm
nlp
computer vision
analytics
sql
spark
airflow
```

### 4.3 Designer Indicators

Examples:

```text
product designer
ux
ui
figma
design system
wireframe
prototype
framer
webflow
case study
visual design
user research
```

---

## 5. Database Changes

Add the following tables.

### 5.1 `job_criteria`

```sql
CREATE TABLE job_criteria (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  job_id UUID NOT NULL REFERENCES jobs(id) ON DELETE CASCADE,
  criterion_type TEXT NOT NULL,
  name TEXT NOT NULL,
  importance TEXT NOT NULL DEFAULT 'medium',
  is_hard_filter BOOLEAN NOT NULL DEFAULT false,
  confidence NUMERIC DEFAULT 0.0,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

Recommended `criterion_type` values:

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
```

Recommended `importance` values:

```text
high
medium
low
```

---

### 5.2 `search_queries`

```sql
CREATE TABLE search_queries (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  job_id UUID NOT NULL REFERENCES jobs(id) ON DELETE CASCADE,
  source TEXT NOT NULL,
  query_type TEXT NOT NULL,
  query_text TEXT NOT NULL,
  priority INTEGER DEFAULT 1,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

Recommended `source` values:

```text
github
web_search
linkedin_native
linkedin_xray
huggingface
kaggle
dribbble
manual
```

Recommended `query_type` values:

```text
primary
skill_based
title_based
location_based
alternative_title
company_based
xray
```

---

### 5.3 `candidates`

```sql
CREATE TABLE candidates (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  canonical_name TEXT,
  headline TEXT,
  role_category TEXT,
  location TEXT,
  email TEXT,
  website TEXT,
  profile_json JSONB,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

### 5.4 `candidate_sources`

```sql
CREATE TABLE candidate_sources (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  candidate_id UUID NOT NULL REFERENCES candidates(id) ON DELETE CASCADE,
  source TEXT NOT NULL,
  source_id TEXT,
  source_url TEXT,
  raw_text TEXT,
  raw_json JSONB,
  permission_type TEXT DEFAULT 'manual_user_import',
  collected_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

### 5.5 `lead_status`

```sql
CREATE TABLE lead_status (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  job_id UUID NOT NULL REFERENCES jobs(id) ON DELETE CASCADE,
  candidate_id UUID NOT NULL REFERENCES candidates(id) ON DELETE CASCADE,
  status TEXT NOT NULL DEFAULT 'new',
  notes TEXT,
  last_contacted_at TIMESTAMPTZ,
  contact_count INTEGER NOT NULL DEFAULT 0,
  do_not_contact BOOLEAN NOT NULL DEFAULT false,
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## 6. Backend Services to Add

### 6.1 JD Parser Service

Create:

```text
app/services/jd_parser_service.py
```

Responsibilities:

```text
Accept raw JD
Extract structured criteria
Classify role category
Return JSON
```

Use an LLM if available through environment variable.

If no LLM key exists, use fallback rule-based extraction.

---

### 6.2 Query Generator Service

Create:

```text
app/services/query_generator_service.py
```

Responsibilities:

```text
Generate GitHub queries
Generate public web search queries
Generate LinkedIn native queries
Generate LinkedIn X-Ray queries
Generate role-specific queries
Store queries in search_queries table
```

---

### 6.3 Candidate Import Service

Create:

```text
app/services/candidate_import_service.py
```

Responsibilities:

```text
Create candidate from manual input
Create candidate_source record
Create lead_status for job
Extract basic fields from pasted text where possible
```

---

## 7. LLM Prompt Design

### 7.1 JD Parser Prompt

Use this structure.

```text
You are a sourcing system that extracts structured recruiting criteria from a job description.

Rules:
- Return valid JSON only.
- Do not invent requirements.
- Separate must-have skills from nice-to-have skills.
- Classify role as developer, ml_data, designer, or other.
- Identify hard filters.
- Do not extract protected attributes such as age, gender, religion, caste, health, family status, or political identity.
```

Expected JSON:

```json
{
  "role_category": "developer",
  "role_titles": ["Backend Engineer", "Python Developer"],
  "seniority": "mid",
  "min_experience_years": 3,
  "max_experience_years": null,
  "must_have_skills": ["Python", "FastAPI", "PostgreSQL"],
  "nice_to_have_skills": ["AWS", "Docker", "Kubernetes"],
  "locations": ["India"],
  "work_mode": "remote",
  "industries": [],
  "tools": [],
  "domains": [],
  "hard_filters": ["India", "3+ years experience"],
  "warnings": []
}
```

---

### 7.2 Query Generator Prompt

```text
You are a sourcing query generator.

Given structured job criteria, generate source-specific search queries.

Sources:
- GitHub
- Public web search
- LinkedIn native search
- LinkedIn Google X-Ray

Rules:
- Generate practical queries.
- Include alternative titles.
- Include skill combinations.
- Include location variants.
- Avoid protected attributes.
- Return JSON only.
```

Expected JSON:

```json
{
  "github": [
    "language:Python FastAPI PostgreSQL location:India",
    "FastAPI PostgreSQL AWS in:readme"
  ],
  "web_search": [
    "\"Backend Engineer\" \"FastAPI\" \"India\" \"GitHub\"",
    "\"Python Developer\" \"PostgreSQL\" \"portfolio\" \"India\""
  ],
  "linkedin_native": [
    "(\"Backend Engineer\" OR \"Python Developer\") AND Python AND FastAPI AND India"
  ],
  "linkedin_xray": [
    "site:linkedin.com/in \"Backend Engineer\" \"Python\" \"FastAPI\" \"India\""
  ]
}
```

---

## 8. Backend API Additions

### 8.1 POST `/jobs/{job_id}/parse`

Runs JD parsing only.

Response:

```json
{
  "job_id": "uuid",
  "parsed_criteria": {},
  "criteria": []
}
```

---

### 8.2 GET `/jobs/{job_id}/criteria`

Returns criteria rows.

---

### 8.3 PUT `/jobs/{job_id}/criteria`

Allows replacing or editing criteria.

Request:

```json
{
  "criteria": [
    {
      "criterion_type": "must_have_skill",
      "name": "Python",
      "importance": "high",
      "is_hard_filter": true
    }
  ]
}
```

---

### 8.4 POST `/jobs/{job_id}/generate-queries`

Generates and stores search queries.

---

### 8.5 GET `/jobs/{job_id}/search-queries`

Returns generated queries grouped by source.

---

### 8.6 POST `/jobs/{job_id}/manual-candidates`

Manual import.

Request:

```json
{
  "name": "Amit Kumar",
  "headline": "Backend Engineer",
  "location": "India",
  "email": null,
  "website": "https://amit.dev",
  "source": "linkedin",
  "source_url": "https://linkedin.com/in/example",
  "raw_text": "Pasted visible profile text...",
  "notes": "Looks relevant"
}
```

Response:

```json
{
  "candidate_id": "uuid",
  "lead_status": "new"
}
```

---

### 8.7 GET `/jobs/{job_id}/candidates`

Returns candidates linked to the job.

At Stage 2, candidates may not yet have scores.

---

## 9. Worker Process Changes

Update the existing Stage 1 worker to run:

```text
JD_PARSE_STARTED
JD_PARSE_COMPLETED
CRITERIA_SAVED
QUERY_GENERATION_STARTED
QUERY_GENERATION_COMPLETED
LINKEDIN_QUERIES_READY
STAGE_2_COMPLETED
```

Job status should become:

```text
created → queued → running → ready_for_manual_import
```

Recommended `ready_for_manual_import` status:

```text
ready_for_manual_import
```

This indicates:

- Criteria extracted.
- Queries generated.
- User can manually research LinkedIn or add profiles.

---

## 10. Frontend Changes

### 10.1 `/jobs/[jobId]`

Add tabs:

```text
Overview
Criteria
Queries
Candidates
Process
```

---

### 10.2 Criteria Tab

Show editable extracted criteria.

Sections:

```text
Role Category
Seniority
Experience
Location
Must-have Skills
Nice-to-have Skills
Tools
Domains
Warnings
```

Actions:

```text
Edit
Save
Regenerate Queries
```

---

### 10.3 Queries Tab

Group queries by source.

Sections:

```text
GitHub Queries
Web Search Queries
LinkedIn Native Queries
LinkedIn X-Ray Queries
```

Each query should have:

```text
Copy button
Source label
Priority label
```

For LinkedIn, add helper text:

```text
Use these queries manually in LinkedIn or Google. Paste selected profiles into Manual Import.
```

---

### 10.4 Manual Import Page

Create:

```text
/import
```

Also allow job-specific import:

```text
/jobs/[jobId]/candidates/import
```

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

Sources dropdown:

```text
linkedin
github
portfolio
dribbble
behance
kaggle
huggingface
other
```

---

### 10.5 Candidates Tab

Show imported candidates.

Columns:

```text
Name
Headline
Location
Source
Status
Imported At
Actions
```

No advanced scoring yet.

---

## 11. Manual LinkedIn Workflow

The app should guide the user.

Display instructions:

```text
1. Copy a generated LinkedIn query.
2. Search LinkedIn manually.
3. Open promising profiles.
4. Copy profile URL and visible relevant profile text.
5. Paste into Manual Candidate Import.
6. The system will analyze and score candidates in later stages.
```

Do not implement automated LinkedIn scraping.

---

## 12. Validation Rules

### JD Parsing

If JD is too short:

```text
Warning: JD may not contain enough detail.
```

If role cannot be classified:

```text
role_category = other
```

If no must-have skills found:

```text
Add warning: No clear must-have skills found.
```

---

## 13. Stage 2 Acceptance Criteria

Stage 2 is complete when:

1. User can create a job and run parsing.
2. System extracts structured job criteria.
3. User can view and edit criteria.
4. System generates GitHub, web, LinkedIn native, and LinkedIn X-Ray queries.
5. Queries are stored and visible in frontend.
6. User can copy LinkedIn queries.
7. User can manually import a candidate.
8. Candidate is stored and linked to job.
9. Process events show all Stage 2 steps.
10. Stage 3 can use stored criteria and queries for automated sourcing.

---

## 14. Non-Goals for Stage 2

Do not implement:

```text
GitHub API execution
Profile enrichment
Advanced candidate scoring
Deduplication
Outreach drafts
Email enrichment
Search API crawling
```

---

## 15. Instructions for Coding Agent

When implementing Stage 2:

1. Preserve Stage 1 architecture.
2. Add database migrations, do not modify tables manually.
3. Keep parser service separate from query generator service.
4. Use LLM only behind a service abstraction.
5. Provide fallback parsing so the app works without an API key.
6. Keep LinkedIn manual-assisted only.
7. Store all generated queries.
8. Make criteria editable before future sourcing.
9. Add process events for every meaningful backend stage.
10. Keep UI simple and operational.

---

## 16. Expected Final State

At the end of Stage 2, the user should be able to:

```text
Upload JD
   ↓
Extract criteria
   ↓
Review/edit criteria
   ↓
Get platform-specific queries
   ↓
Use LinkedIn manually
   ↓
Import candidates manually
```

This stage prepares the system for real automated sourcing in Stage 3.

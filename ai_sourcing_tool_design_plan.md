# AI Developer, ML/Data, and Designer Sourcing Tool — Detailed Design Plan

**Version:** 1.0  
**Primary user:** Personal use / solo sourcing workflow  
**Primary goal:** Generate qualified candidate leads from job descriptions using a semi-automated, agentic sourcing system.  
**Focus roles:** Developers, Data/ML candidates, and Designers.  
**Core constraint:** Keep cost low while automating as much sourcing, extraction, scoring, and shortlisting as possible.  
**Important note:** LinkedIn remains part of the workflow, but it should be treated as a manual or semi-manual assisted lane, not as the core automated scraping source.

---

## 1. Executive Summary

This tool will take a job description as input and convert it into a structured sourcing process. The system will extract job requirements, generate source-specific search strategies, run automated searches on API-friendly sources, collect and enrich candidate profiles, score candidates against the job, and present a ranked lead list.

The system will use **multiple sourcing lanes**:

1. **Automated Developer Lane**
   - GitHub
   - GitLab where available
   - Stack Exchange
   - Public blogs
   - Personal websites
   - Search API results

2. **Automated ML/Data Lane**
   - GitHub
   - Hugging Face
   - Kaggle
   - Papers / research pages
   - Personal websites
   - Search API results

3. **Automated Designer Lane**
   - Dribbble
   - Behance via search/manual-assisted flow
   - Webflow showcase
   - Framer/community portfolios
   - Personal portfolio websites
   - Search API results

4. **LinkedIn-Assisted Lane**
   - LinkedIn manual research
   - LinkedIn Boolean query generation
   - LinkedIn profile URL collection
   - Optional manual paste/import
   - Optional browser helper later
   - No dependency on unauthorized LinkedIn scraping as the main system foundation

5. **Manual Import Lane**
   - CSV import
   - Resume/profile paste
   - Candidate URL paste
   - Existing personal database

The product should not behave like a mass-scraping bot. The strongest design is a **sourcing command center** that automates discovery and ranking where allowed, while keeping human review for LinkedIn, final selection, and outreach.

---

## 2. Product Vision

### 2.1 Product Statement

> A personal AI sourcing dashboard that turns a job description into a ranked, evidence-backed candidate lead list by searching developer, ML/data, designer, public web, and LinkedIn-assisted sources.

### 2.2 Core Outcome

For every job description, the user should be able to:

1. Upload or paste a JD.
2. Watch backend sourcing stages run.
3. See which sources are being searched.
4. Get generated search queries for LinkedIn/manual sources.
5. Get automatically collected leads from API-friendly/public sources.
6. Paste or import LinkedIn profiles manually when needed.
7. See ranked candidates with evidence.
8. Shortlist the best candidates.
9. Generate careful outreach drafts.
10. Track contact status and outcomes.

### 2.3 What This Tool Is

This is:

- A sourcing assistant.
- A lead generation dashboard.
- A candidate scoring system.
- A multi-source profile aggregator.
- A semi-agentic research workflow.
- A personal talent database.

### 2.4 What This Tool Is Not

This is not:

- A LinkedIn scraping product.
- A fully automated mass-message tool.
- A replacement for human judgment.
- A hiring decision system.
- A black-box AI ranking tool.
- A tool for storing sensitive personal traits.

---

## 3. Selected Scope

### 3.1 Included Role Categories

The system will focus on three categories.

#### A. Developers

Examples:

- Backend Engineer
- Frontend Engineer
- Full-stack Engineer
- DevOps Engineer
- Platform Engineer
- Mobile Developer
- AI Engineer with coding-heavy profile

Primary evidence sources:

- GitHub repositories
- Programming language usage
- README files
- Recent activity
- Project topics
- Public email or portfolio links
- Personal blogs
- Stack Exchange reputation/activity

#### B. Data / ML Candidates

Examples:

- Data Scientist
- ML Engineer
- MLOps Engineer
- AI Engineer
- NLP Engineer
- Computer Vision Engineer
- Data Engineer
- Analytics Engineer

Primary evidence sources:

- GitHub ML repos
- Hugging Face models/datasets/spaces
- Kaggle notebooks/competitions/datasets
- Research pages
- Personal websites
- Technical blogs

#### C. Designers

Examples:

- Product Designer
- UI Designer
- UX Designer
- Web Designer
- Brand Designer
- Motion Designer
- Framer/Webflow Designer

Primary evidence sources:

- Portfolio websites
- Dribbble
- Behance
- Webflow showcase
- Framer community
- Case studies
- Public design posts

---

## 4. Data Source Strategy

The sourcing system should use different sources based on role type and automation quality.

---

### 4.1 Source Priority Table

| Source | Developer | ML/Data | Designer | Automation Level | Priority |
|---|---:|---:|---:|---:|---:|
| GitHub | High | High | Low | High | P0 |
| Hugging Face | Low | High | None | High | P1 |
| Kaggle | Low | High | None | Medium | P1 |
| Stack Exchange | Medium | Medium | None | Medium | P2 |
| Dribbble | None | None | High | Medium | P1 |
| Behance | None | None | High | Medium/Manual-assisted | P2 |
| Personal websites | High | High | High | Medium | P0 |
| Search API | High | High | High | Medium | P0 |
| LinkedIn | High | High | High | Manual-assisted | P0 |
| CSV/manual import | High | High | High | Manual | P0 |
| Email enrichment API | Medium | Medium | Medium | Paid/Selective | P3 |

---

## 5. LinkedIn Strategy

LinkedIn cannot be ignored because it has strong professional identity data: current title, company, location, career history, and social proof. However, LinkedIn should not be the main automated scraping engine.

### 5.1 LinkedIn’s Role in This System

LinkedIn should be used as a **manual-assisted sourcing lane**:

1. The system generates LinkedIn Boolean queries.
2. The user runs searches manually on LinkedIn or Google.
3. The user saves profile URLs into the app.
4. The app analyzes manually pasted profile text or imported profile notes.
5. The app scores those profiles against the job description.
6. The app deduplicates LinkedIn profiles against GitHub, portfolio, Kaggle, Hugging Face, and other sources.

### 5.2 Why LinkedIn Should Be Manual-Assisted

LinkedIn’s official API documentation is organized by business lines such as Consumer, Marketing, Sales, Compliance, Learning, and Talent Solutions. Broad candidate search is generally not a freely available open API for arbitrary third-party apps. Official integrations depend on approved use cases and access models.

Source: LinkedIn API Documentation, Microsoft Learn  
https://learn.microsoft.com/en-us/linkedin/

### 5.3 LinkedIn Workflow

```text
JD uploaded
   ↓
System extracts criteria
   ↓
System generates LinkedIn Boolean queries
   ↓
User manually searches LinkedIn
   ↓
User saves profile URLs / profile text
   ↓
System analyzes and scores LinkedIn-sourced leads
   ↓
System merges with candidates from other sources
```

### 5.4 LinkedIn Query Generator

For each job, the system should generate:

- Basic LinkedIn search query
- Google X-Ray LinkedIn query
- Title-focused query
- Skill-focused query
- Location-focused query
- Company-focused query
- Alternative title query

Example for a backend role:

```text
LinkedIn native search:
("Backend Engineer" OR "Software Engineer" OR "Python Developer") AND Python AND AWS AND PostgreSQL AND India

Google X-Ray search:
site:linkedin.com/in ("Backend Engineer" OR "Software Engineer") "Python" "AWS" "India"

Alternative title query:
site:linkedin.com/in ("Platform Engineer" OR "API Engineer" OR "Backend Developer") "FastAPI" "PostgreSQL"
```

### 5.5 LinkedIn Import Methods

#### Version 1

Manual input:

- Candidate name
- LinkedIn URL
- Current title
- Company
- Location
- Pasted profile summary
- Notes

#### Version 2

Semi-manual browser helper:

- User opens LinkedIn profile manually.
- Browser extension captures visible text only after user action.
- User clicks “Save to Sourcing Tool.”
- The app parses and scores the saved information.

#### Version 3

Approved integrations only:

- If the user later gets access to permitted LinkedIn or ATS integrations, the system can connect through official APIs.

### 5.6 LinkedIn Risk Controls

The app should include:

- Manual confirmation before importing LinkedIn data.
- No automatic mass profile visits.
- No bypassing rate limits.
- No proxy/captcha circumvention.
- No fake account automation.
- No auto-messaging from LinkedIn.
- A do-not-contact flag.
- Source and timestamp tracking.

---

## 6. Automation Design: Multiple Lanes

The system should be designed as a **multi-lane sourcing engine**. Each lane is specialized for a different type of source.

---

### 6.1 Lane Overview

```text
Job Description
   ↓
JD Parser Layer
   ↓
Criteria + Search Intent
   ↓
┌──────────────────────────────┐
│ Multi-Lane Sourcing Engine   │
├──────────────────────────────┤
│ Lane 1: GitHub Developer     │
│ Lane 2: ML/Data Sources      │
│ Lane 3: Designer Sources     │
│ Lane 4: Search API/Public Web│
│ Lane 5: LinkedIn-Assisted    │
│ Lane 6: Manual Import        │
└──────────────────────────────┘
   ↓
Profile Normalization Layer
   ↓
Deduplication Layer
   ↓
Evidence Extraction Layer
   ↓
Scoring Layer
   ↓
Candidate Review Dashboard
```

---

## 7. System Layers

The system should use layered design so every stage is clear, observable, and replaceable.

---

### 7.1 Layer 1: Frontend Layer

Purpose:

- Minimal interface.
- Upload/paste JD.
- Show backend process status.
- Display generated queries.
- Display candidate results.
- Allow manual LinkedIn/profile imports.
- Show candidate scoring and evidence.

Selected framework:

- **Next.js**
- **React**
- **Tailwind CSS**
- **shadcn/ui** or simple component library

Reason:

- Fast development.
- Easy deployment.
- Good form/dashboard support.
- Works well with API backend.
- Minimal frontend can still look clean.

Frontend pages:

```text
/jobs
/jobs/new
/jobs/:jobId
/jobs/:jobId/process
/jobs/:jobId/candidates
/candidates/:candidateId
/import
/settings
```

---

### 7.2 Layer 2: API Gateway / Backend API Layer

Purpose:

- Accept JD uploads.
- Trigger sourcing jobs.
- Serve process status.
- Save candidates.
- Serve candidate results.
- Manage scoring and review actions.

Selected framework:

- **FastAPI**

Reason:

- Python works well for AI, scraping-safe extraction, data processing, and ML workflows.
- Easy async support.
- Clean API contracts.
- Works well with Celery/Redis.

Alternative:

- Node.js/NestJS if the entire app needs TypeScript consistency.

Selected for this design:

> **FastAPI backend + Next.js frontend**

---

### 7.3 Layer 3: Job Processing / Queue Layer

Purpose:

- Run long backend tasks without blocking the frontend.
- Show process status to the user.
- Process multiple source lanes independently.
- Retry failed jobs.
- Track logs per stage.

Selected tools:

- **Redis**
- **Celery**

Reason:

- Simple and reliable.
- Works naturally with Python/FastAPI.
- Good enough for a personal project.
- Can later be replaced by Temporal or Cloud Tasks if needed.

Process stages:

```text
JD_RECEIVED
JD_PARSED
QUERIES_GENERATED
SOURCE_SEARCH_STARTED
SOURCE_SEARCH_COMPLETED
PROFILES_COLLECTED
PROFILES_ENRICHED
DEDUPLICATION_COMPLETED
EVIDENCE_EXTRACTED
SCORING_COMPLETED
RESULTS_READY
```

---

### 7.4 Layer 4: Source Connector Layer

Purpose:

- Each source has its own connector.
- Avoid mixing source-specific code with scoring logic.
- Make sources replaceable.

Selected initial connectors:

```text
GitHubConnector
SearchAPIConnector
LinkedInAssistConnector
ManualImportConnector
PortfolioCrawlerConnector
```

Next connectors:

```text
HuggingFaceConnector
KaggleConnector
DribbbleConnector
StackExchangeConnector
```

Later connectors:

```text
EmailEnrichmentConnector
ATSImportConnector
CRMImportConnector
```

---

### 7.5 Layer 5: Extraction and Normalization Layer

Purpose:

- Convert raw source data into a common candidate format.
- Extract skills, projects, locations, role titles, contact details, and evidence.
- Separate “claimed data” from “evidence-backed data.”

Selected tools:

- Python extraction code
- LLM API for structured extraction
- Regex/rules for emails, URLs, locations, years
- Optional embeddings for semantic matching

Candidate normalization output:

```json
{
  "candidate_name": "string",
  "headline": "string",
  "location": "string",
  "role_category": "developer | ml_data | designer",
  "skills": [],
  "projects": [],
  "profiles": {},
  "contact": {},
  "evidence": [],
  "source_confidence": 0.0
}
```

---

### 7.6 Layer 6: Storage Layer

Purpose:

- Store jobs, source runs, candidates, raw source data, normalized profiles, evidence, scores, and statuses.

Selected database:

- **PostgreSQL via Supabase**

Reason:

- Low-cost/free tier friendly.
- Good relational model.
- Works well with JSONB.
- Can support vector search through pgvector.
- Easy dashboard/admin support.

Optional vector capability:

- **pgvector**

Used for:

- Similar candidate search.
- Matching job descriptions to previous candidates.
- Semantic skill similarity.
- Search over profile/project text.

---

### 7.7 Layer 7: AI / LLM Layer

Purpose:

- Extract structured JD criteria.
- Generate search queries.
- Normalize profile text.
- Summarize candidate evidence.
- Explain candidate fit.
- Generate outreach drafts.

Selected options:

- OpenAI API
- Gemini API
- Anthropic API
- Local model later for cost reduction

Recommended initial choice:

> Use one paid API for quality and speed. Keep prompts small and use deterministic logic wherever possible.

AI should not be responsible for everything. Use AI for language-heavy tasks and rules/code for scoring.

---

### 7.8 Layer 8: Scoring Layer

Purpose:

- Score candidates using deterministic scoring.
- Keep scoring explainable.
- Avoid black-box ranking.

The score should be built from multiple sub-scores:

```text
final_score =
  must_have_skill_score
+ role_relevance_score
+ evidence_quality_score
+ experience/seniority_score
+ source_quality_score
+ location_score
+ contactability_score
+ recency_score
```

Important distinction:

- **Match score:** How well they fit the JD.
- **Confidence score:** How strong the evidence is.
- **Contactability score:** Whether you can reach them.
- **Source score:** How trustworthy/fresh the source is.
- **Action score:** Whether they are worth contacting now.

---

### 7.9 Layer 9: Review and Workflow Layer

Purpose:

- Human review before outreach.
- Status tracking.
- Notes.
- Shortlisting.
- Do-not-contact handling.

Candidate statuses:

```text
NEW
AUTO_SCORED
NEEDS_REVIEW
SHORTLISTED
REJECTED
CONTACT_READY
CONTACTED
REPLIED
FOLLOW_UP
DO_NOT_CONTACT
ARCHIVED
```

---

### 7.10 Layer 10: Outreach Layer

Purpose:

- Generate short, personalized outreach drafts.
- Avoid spammy outreach.
- Keep sending manual or semi-manual initially.

Initial design:

- Generate draft only.
- User reviews and sends manually.

Later:

- Gmail draft creation.
- No automatic send until the user explicitly approves.

---

## 8. Selected Tech Stack

### 8.1 Final Recommended Stack

| Area | Selected Tool |
|---|---|
| Frontend | Next.js |
| UI | Tailwind CSS + shadcn/ui |
| Backend | FastAPI |
| Language | Python + TypeScript |
| Database | Supabase PostgreSQL |
| Vector Search | pgvector |
| Queue | Redis + Celery |
| Storage | Supabase Storage or S3-compatible storage |
| AI | OpenAI/Gemini/Anthropic API |
| Search API | Brave Search / SerpAPI / Tavily / Exa / DataForSEO |
| Developer Source | GitHub REST/GraphQL API |
| ML Source | Hugging Face Hub API |
| ML/Data Source | Kaggle API/CLI |
| Designer Source | Dribbble API + portfolio crawling |
| Monitoring | Sentry |
| Product Analytics | PostHog optional |
| Deployment | Vercel + Render/Fly.io/Railway |
| Auth | Supabase Auth |
| Browser Helper Later | Chrome Extension |

---

## 9. Why These Tools Are Selected

### 9.1 Next.js

Selected because:

- Great for dashboards.
- Easy routing.
- Fast frontend development.
- Simple deployment on Vercel.
- Works well with API routes if needed.

### 9.2 FastAPI

Selected because:

- Python ecosystem is strong for AI and data extraction.
- Easy to build structured APIs.
- Good async support.
- Good fit for background workers.

### 9.3 PostgreSQL / Supabase

Selected because:

- Low cost.
- Good for structured candidate data.
- Supports JSONB for flexible source payloads.
- Can support pgvector for embeddings.
- Good enough for MVP and personal use.

### 9.4 Redis + Celery

Selected because:

- Sourcing workflows are background-heavy.
- Each source connector may take time.
- Frontend needs process visibility.
- Jobs need retry/failure tracking.

### 9.5 GitHub API

Selected because:

- Strong developer evidence.
- Official API support.
- High value for developer sourcing.
- Can find real technical work.

GitHub officially documents REST API rate limits and notes that some endpoints, including search endpoints, have more restrictive limits than general API endpoints.

Source: GitHub REST API Rate Limits  
https://docs.github.com/en/rest/using-the-rest-api/rate-limits-for-the-rest-api

### 9.6 Hugging Face Hub API

Selected because:

- Strong source for ML and AI talent.
- Public models, datasets, and Spaces provide evidence.
- Programmatic Hub API access exists.

Source: Hugging Face Hub API Endpoints  
https://huggingface.co/docs/hub/api

### 9.7 Kaggle API

Selected because:

- Useful for data science and ML profiles.
- Programmatic access exists through Kaggle CLI and kagglehub.

Source: Kaggle API Docs  
https://www.kaggle.com/docs/api

### 9.8 LinkedIn-Assisted Lane

Selected because:

- LinkedIn is valuable for professional identity.
- It is not ideal as a fully automated data source without approved access.
- Manual-assisted use is practical and useful.

Source: LinkedIn API Documentation  
https://learn.microsoft.com/en-us/linkedin/

---

## 10. Frontend Design

The frontend should be minimal but process-oriented. The goal is not visual complexity. The user should clearly see what the backend is doing.

---

### 10.1 Design Principles

- Minimal UI.
- Clear process visibility.
- Strong table views.
- Easy manual import.
- Evidence-first candidate cards.
- No unnecessary animations.
- Functional over beautiful.

---

### 10.2 Main Navigation

```text
Jobs
Candidates
Search Runs
Manual Import
Settings
```

---

### 10.3 Page: Jobs

Purpose:

- List all sourcing jobs.

Columns:

```text
Job Title
Role Type
Location
Status
Candidates Found
Strong Matches
Created At
Actions
```

Actions:

```text
View
Run Again
Export
Archive
```

---

### 10.4 Page: New Job

Fields:

```text
Job Title
Raw Job Description
Role Category
Location Preference
Remote/Hybrid/On-site
Target Candidate Count
Sources to Use
LinkedIn Included: Yes/No
```

Source checkboxes:

```text
GitHub
Search API/Public Web
LinkedIn-Assisted
Hugging Face
Kaggle
Dribbble
Manual Import
```

Button:

```text
Start Sourcing
```

---

### 10.5 Page: Job Process View

This is important. The user wants to see backend processes.

Display timeline:

```text
✓ JD received
✓ JD parsed
✓ Criteria extracted
✓ Search queries generated
⟳ GitHub search running
⟳ Search API running
○ Hugging Face pending
○ LinkedIn manual queries ready
○ Deduplication pending
○ Scoring pending
○ Results pending
```

For each stage, show:

```text
Stage name
Status
Started at
Completed at
Records processed
Errors/warnings
View logs
```

---

### 10.6 Page: Extracted Criteria

Show editable criteria:

```text
Role Category: Backend Developer
Seniority: Mid-level
Must-have skills:
- Python
- FastAPI
- PostgreSQL
- AWS

Nice-to-have:
- Docker
- Kubernetes
- Fintech

Location:
- India
- Remote acceptable

Experience:
- 3+ years
```

User can edit before running sources.

---

### 10.7 Page: Generated Search Queries

Separate query sections:

#### GitHub Queries

```text
language:Python FastAPI PostgreSQL location:India
topic:fastapi AWS India
"FastAPI" "PostgreSQL" "Docker" in:readme
```

#### Web Search Queries

```text
"Python Backend Engineer" "FastAPI" "India" "GitHub"
"Backend Developer" "AWS" "PostgreSQL" "portfolio"
```

#### LinkedIn Queries

```text
("Backend Engineer" OR "Python Developer") AND FastAPI AND PostgreSQL AND India

site:linkedin.com/in "Backend Engineer" "Python" "AWS" "India"
```

The LinkedIn section should include a copy button.

---

### 10.8 Page: Candidate Results

Columns:

```text
Candidate
Role/Headline
Source
Location
Match Score
Confidence
Contactability
Evidence Count
Status
Actions
```

Actions:

```text
View
Shortlist
Reject
Generate Outreach
Mark Contacted
Do Not Contact
```

Filters:

```text
Source
Score Range
Role Category
Location
Contact Available
Status
Must-have Missing
```

---

### 10.9 Candidate Detail Page

Sections:

1. Summary
2. Source profiles
3. Matched requirements
4. Missing requirements
5. Evidence table
6. Score breakdown
7. Contact details
8. Outreach draft
9. Notes
10. Status history

Evidence table:

| Requirement | Evidence | Source | Confidence |
|---|---|---|---|
| Python | Repo language and README mention | GitHub | High |
| FastAPI | README mentions FastAPI API server | GitHub | High |
| AWS | No confirmed evidence | N/A | Low |
| India | Profile location says India | GitHub | Medium |

---

## 11. Backend Process Design

---

### 11.1 High-Level Process

```text
1. Create job
2. Parse job description
3. Generate structured criteria
4. Generate source queries
5. Run selected source lanes
6. Collect raw profiles
7. Enrich profiles
8. Normalize candidates
9. Deduplicate candidates
10. Extract evidence
11. Score candidates
12. Generate explanations
13. Present ranked results
14. Enable manual LinkedIn import
15. Shortlist and outreach
```

---

### 11.2 Process Visibility

The backend should write process events.

Example:

```json
{
  "job_id": "job_123",
  "stage": "GITHUB_SEARCH",
  "status": "running",
  "message": "Running 8 GitHub queries",
  "records_found": 42,
  "created_at": "timestamp"
}
```

Frontend subscribes through:

- Polling every 2–5 seconds initially.
- Later WebSockets or Server-Sent Events.

---

## 12. Agentic Design

The system should use specialized agents. Each agent has one responsibility.

---

### 12.1 Agent List

| Agent | Responsibility |
|---|---|
| JD Parser Agent | Extract role requirements |
| Search Strategy Agent | Generate platform-specific queries |
| GitHub Sourcing Agent | Search and collect GitHub candidates |
| ML Sourcing Agent | Search Hugging Face/Kaggle |
| Designer Sourcing Agent | Search Dribbble/portfolio sources |
| Web Discovery Agent | Use search API to find portfolios |
| LinkedIn Assist Agent | Generate LinkedIn queries and process manual imports |
| Profile Enrichment Agent | Expand candidate data |
| Normalization Agent | Convert data into canonical profile |
| Deduplication Agent | Merge duplicate identities |
| Evidence Agent | Extract requirement-level evidence |
| Scoring Agent | Score candidates |
| Explanation Agent | Explain candidate fit |
| Outreach Draft Agent | Draft contact messages |

---

### 12.2 Agent Boundaries

Important design rule:

> Agents should not directly make final decisions. They should produce structured outputs, confidence scores, and evidence.

For example, the Scoring Agent should not say:

```text
Hire this person.
```

It should say:

```text
Match score: 84
Confidence score: 78
Recommendation: Shortlist for human review
```

---

## 13. Source Lanes in Detail

---

### 13.1 Developer Lane

#### Sources

- GitHub API
- Search API
- Stack Exchange
- Personal websites
- LinkedIn-assisted import

#### Data Collected

```text
GitHub username
Name
Bio
Location
Email if public
Website
Repositories
Languages
Topics
README text
Recent activity
Stars/forks
Profile links
```

#### Scoring Signals

```text
Relevant language usage
Relevant framework evidence
Recent project activity
Project complexity
README quality
Deployment evidence
Portfolio link
Location match
Contact availability
```

#### Candidate Examples

Strong developer candidate:

```text
Has multiple Python repos
Uses FastAPI
Mentions PostgreSQL
Recent commits
Profile location matches
Portfolio/email available
```

Weak candidate:

```text
Only forked repos
No recent activity
No relevant frameworks
No location/contact data
```

---

### 13.2 ML/Data Lane

#### Sources

- GitHub
- Hugging Face
- Kaggle
- Search API
- Personal websites
- LinkedIn-assisted import

#### Data Collected

```text
Models
Datasets
Spaces
Notebooks
ML repos
Frameworks
Papers
Blog posts
Competition activity
MLOps evidence
```

#### Scoring Signals

```text
PyTorch/TensorFlow/JAX
Transformers
Model deployment
Dataset work
Notebook quality
MLOps tools
Recent activity
Domain match
Portfolio/research page
```

#### Strong ML Candidate

```text
Has public models on Hugging Face
Has ML repos on GitHub
Has notebooks/projects
Mentions deployment/MLOps
Recent activity
```

---

### 13.3 Designer Lane

#### Sources

- Dribbble
- Behance/search
- Webflow showcase
- Framer community
- Personal portfolio
- LinkedIn-assisted import

#### Data Collected

```text
Portfolio URL
Project titles
Case studies
Design tools
Visual categories
Industry/domain
Contact info
Social links
```

#### Scoring Signals

```text
Portfolio availability
Case study depth
Product design relevance
SaaS/web/mobile experience
Figma/Webflow/Framer evidence
Visual consistency
Contactability
Recency
```

#### Strong Designer Candidate

```text
Has portfolio
Shows case studies
Works on product/SaaS/mobile interfaces
Explains process
Has contact link
```

---

### 13.4 Public Web Lane

#### Sources

- Search API
- Personal websites
- Blogs
- Portfolio pages
- Conference speaker pages
- Open-source maintainer pages

#### Search API Options

Potential choices:

- Brave Search API
- SerpAPI
- Tavily
- Exa
- DataForSEO
- Bing Web Search API

#### Design Role

This lane is useful for discovering:

```text
portfolio websites
GitHub pages
blogs
LinkedIn public pages
Dribbble/Behance pages
Hugging Face profiles
Kaggle profiles
conference profiles
```

---

### 13.5 LinkedIn-Assisted Lane

#### Purpose

The LinkedIn lane is for professional identity enrichment and manual sourcing.

#### Process

```text
1. System generates LinkedIn queries.
2. User searches LinkedIn manually.
3. User imports selected profile URLs/text.
4. System analyzes imported profile text.
5. System links profile to existing candidates.
6. System updates candidate score and confidence.
```

#### LinkedIn Output

For every job, the app should provide:

```text
LinkedIn Native Search Queries
Google X-Ray LinkedIn Queries
Alternative Job Title Queries
Target Company Queries
Location Queries
Boolean Skill Queries
```

#### Example Output

```text
Native:
("ML Engineer" OR "Machine Learning Engineer" OR "AI Engineer") AND PyTorch AND MLOps AND India

X-Ray:
site:linkedin.com/in ("ML Engineer" OR "Machine Learning Engineer") "PyTorch" "India"

Designer:
site:linkedin.com/in ("Product Designer" OR "UX Designer") "Figma" "SaaS" "India"
```

---

## 14. Candidate Scoring Design

The scoring system should be role-sensitive.

---

### 14.1 Universal Scores

Every candidate gets:

```text
match_score
confidence_score
contactability_score
source_quality_score
recency_score
final_action_score
```

#### Match Score

How well the candidate fits the job.

#### Confidence Score

How reliable the evidence is.

#### Contactability Score

How easy it is to contact the candidate.

#### Source Quality Score

How trustworthy/useful the source is.

#### Recency Score

How current the information/activity is.

#### Final Action Score

Whether the candidate should be reviewed/contacted.

---

### 14.2 Developer Score

| Component | Weight |
|---|---:|
| Must-have skill evidence | 35 |
| Relevant project evidence | 20 |
| Role/title similarity | 10 |
| Recent activity | 10 |
| Location match | 10 |
| Nice-to-have skills | 10 |
| Contactability | 5 |

---

### 14.3 ML/Data Score

| Component | Weight |
|---|---:|
| ML framework evidence | 20 |
| Project/model/notebook evidence | 25 |
| Domain relevance | 10 |
| MLOps/deployment evidence | 15 |
| Recent activity | 10 |
| Location match | 10 |
| Contactability | 10 |

---

### 14.4 Designer Score

| Component | Weight |
|---|---:|
| Portfolio availability | 20 |
| Relevant case studies | 25 |
| Visual/product relevance | 15 |
| Tool/domain match | 10 |
| Portfolio recency | 10 |
| Location match | 10 |
| Contactability | 10 |

---

### 14.5 Scoring Bands

```text
85–100: Strong candidate
70–84: Good candidate
55–69: Review manually
40–54: Weak match
Below 40: Ignore/archive
```

---

## 15. Candidate Evidence Design

The system should not only rank candidates. It should show proof.

### 15.1 Evidence Types

```text
skill_evidence
project_evidence
role_evidence
location_evidence
contact_evidence
activity_evidence
portfolio_evidence
ml_artifact_evidence
design_case_study_evidence
```

### 15.2 Evidence Object

```json
{
  "candidate_id": "candidate_123",
  "requirement": "FastAPI",
  "evidence_type": "skill_evidence",
  "source": "github",
  "source_url": "https://github.com/user/repo",
  "text_snippet": "Built with FastAPI and PostgreSQL",
  "confidence": 0.92
}
```

### 15.3 Evidence Rules

- Never invent evidence.
- Always attach a source.
- Prefer recent evidence.
- Distinguish confirmed evidence from inferred evidence.
- Missing evidence should be shown clearly.

---

## 16. Data Model

---

### 16.1 Core Tables

```sql
jobs (
  id,
  title,
  raw_jd,
  role_category,
  parsed_criteria_json,
  status,
  created_at,
  updated_at
);

job_criteria (
  id,
  job_id,
  criterion_type,
  name,
  importance,
  is_hard_filter,
  confidence
);

search_runs (
  id,
  job_id,
  source,
  query,
  status,
  records_found,
  records_processed,
  error_message,
  created_at,
  completed_at
);

candidates (
  id,
  canonical_name,
  headline,
  role_category,
  location,
  email,
  website,
  profile_json,
  created_at,
  updated_at
);

candidate_sources (
  id,
  candidate_id,
  source,
  source_id,
  source_url,
  raw_json,
  permission_type,
  collected_at
);

candidate_evidence (
  id,
  candidate_id,
  job_id,
  requirement,
  evidence_type,
  source,
  source_url,
  text_snippet,
  confidence,
  created_at
);

candidate_scores (
  id,
  job_id,
  candidate_id,
  match_score,
  confidence_score,
  contactability_score,
  source_quality_score,
  recency_score,
  final_action_score,
  explanation,
  missing_requirements_json,
  created_at
);

lead_status (
  id,
  job_id,
  candidate_id,
  status,
  notes,
  last_contacted_at,
  contact_count,
  do_not_contact,
  updated_at
);

process_events (
  id,
  job_id,
  stage,
  status,
  message,
  records_count,
  metadata_json,
  created_at
);
```

---

## 17. Backend Status and Logs

The user should be able to see what the backend is doing.

### 17.1 Process Event Examples

```text
JD_PARSE_STARTED
JD_PARSE_COMPLETED
QUERY_GENERATION_STARTED
QUERY_GENERATION_COMPLETED
GITHUB_SEARCH_STARTED
GITHUB_SEARCH_COMPLETED
WEB_SEARCH_STARTED
WEB_SEARCH_COMPLETED
LINKEDIN_QUERIES_READY
PROFILE_ENRICHMENT_STARTED
PROFILE_ENRICHMENT_COMPLETED
DEDUPLICATION_STARTED
DEDUPLICATION_COMPLETED
SCORING_STARTED
SCORING_COMPLETED
RESULTS_READY
```

### 17.2 UI Display

Example:

```text
GitHub Search
Status: Completed
Queries run: 12
Profiles found: 138
Candidates accepted: 47
Time: 2m 11s

LinkedIn-Assisted Search
Status: Manual action required
Generated queries: 8
Imported profiles: 5
```

---

## 18. Manual Workflow Design

Even with automation, some manual work remains useful.

### 18.1 Manual LinkedIn Flow

```text
1. Open job page.
2. Copy generated LinkedIn query.
3. Search LinkedIn manually.
4. Open likely profiles.
5. Copy profile URL and visible relevant text.
6. Paste into app.
7. App analyzes and scores profile.
8. App deduplicates against existing candidates.
```

### 18.2 Manual Candidate Import

Inputs:

```text
Candidate name
Profile URL
Source
Current title
Company
Location
Pasted profile text
Notes
```

### 18.3 Manual Review Queue

Candidate statuses:

```text
Needs Review
Shortlist
Reject
Contact Ready
Contacted
Replied
Do Not Contact
```

---

## 19. Outreach Design

Outreach should be semi-automated.

### 19.1 Draft Generation

The system generates:

```text
Short message
Reason for contact
Role relevance
Personalized evidence
Call to action
```

Example:

```text
Hi Ankit, I found your FastAPI/PostgreSQL project on GitHub while sourcing for a Python backend role. Your backend API work looked relevant, especially the database and deployment parts. Would you be open to hearing more?
```

### 19.2 Sending

Initial version:

- Copy draft.
- User sends manually.

Later version:

- Gmail draft creation.
- User approves before sending.

Avoid:

- Auto-send.
- Bulk spam.
- Generic messages.
- Multiple follow-ups without response.

---

## 20. Compliance and Ethics Design

Even for personal use, the system should be responsible.

### 20.1 Data Rules

- Store source URL.
- Store collected date.
- Store whether contact is public or manually provided.
- Allow delete/archive.
- Respect do-not-contact.
- Do not store sensitive personal traits.
- Do not infer age, caste, religion, gender, health, politics, family status, etc.

### 20.2 Outreach Rules

```text
Only contact relevant people.
Use personalized reason.
Do not over-message.
Stop after opt-out.
Keep follow-ups limited.
Do not misrepresent the role.
```

### 20.3 LinkedIn Rules

```text
Do not automate mass scraping.
Do not bypass restrictions.
Do not use fake accounts.
Do not auto-visit profiles at scale.
Do not auto-message on LinkedIn.
```

---

## 21. Roadmap

---

### Phase 0: No-Code Validation

Goal:

- Validate workflow before building full app.

Tools:

```text
Google Sheets
ChatGPT
Manual LinkedIn
GitHub search
Google search
```

Deliverable:

- 100 candidate records across 2–3 jobs.
- Manual scoring process.
- Evidence format.

---

### Phase 1: Minimal App

Build:

```text
Next.js frontend
FastAPI backend
Supabase database
JD parser
Query generator
Manual candidate import
Candidate scoring
Results table
CSV export
```

Sources:

```text
Manual LinkedIn
Manual import
GitHub basic API
Search query generation
```

---

### Phase 2: Automated Developer Sourcing

Build:

```text
GitHub search connector
GitHub profile fetcher
Repo analyzer
README extractor
Developer scoring model
Process status timeline
```

Output:

- Automatic developer lead discovery.

---

### Phase 3: Public Web Discovery

Build:

```text
Search API connector
URL classifier
Portfolio crawler
Profile page extractor
Candidate normalization
Deduplication
```

Output:

- Automatic discovery of portfolios, blogs, profile pages.

---

### Phase 4: ML/Data Sources

Build:

```text
Hugging Face connector
Kaggle connector
ML scoring model
ML evidence extractor
```

Output:

- ML/data candidate discovery and ranking.

---

### Phase 5: Designer Sources

Build:

```text
Dribbble connector
Designer portfolio crawler
Portfolio evidence extractor
Designer scoring model
Behance/manual-assisted import
```

Output:

- Designer candidate discovery and ranking.

---

### Phase 6: LinkedIn Workflow Enhancement

Build:

```text
LinkedIn query generator
LinkedIn import form
LinkedIn profile text analyzer
LinkedIn URL deduplication
Optional browser helper
```

Output:

- LinkedIn included without making scraping the core system.

---

### Phase 7: Outreach and Tracking

Build:

```text
Outreach draft generator
Contact status tracking
Gmail draft option
Reply tracking manually
Do-not-contact list
Follow-up reminders
```

---

## 22. Cost Strategy

### 22.1 Free/Low-Cost First

Start with:

```text
GitHub API
Manual LinkedIn
Manual imports
Supabase free tier
Vercel free tier
Render/Fly/Railway low tier
One LLM API
Search API only when needed
```

### 22.2 Avoid Early Costs

Avoid paying early for:

```text
Large enrichment APIs
LinkedIn automation tools
Expensive recruiter platforms
Heavy CRM integrations
Large-scale scraping infrastructure
```

### 22.3 Cost Control Funnel

Do not enrich every candidate.

Use:

```text
Raw candidates: 300
Potential candidates: 100
Scored candidates: 60
Shortlisted candidates: 20
Enriched/contacted candidates: 10
```

Only spend money on shortlisted leads.

---

## 23. What the First Build Should Include

### 23.1 Must-Have Features

```text
JD upload/paste
AI JD parsing
Editable criteria
Source selection
Query generation
Process timeline
GitHub connector
Manual LinkedIn import
Manual candidate import
Candidate scoring
Candidate results table
Candidate detail page
Evidence display
CSV export
```

### 23.2 Should-Have Features

```text
Search API connector
Portfolio crawler
Deduplication
Outreach draft generation
Status tracking
Do-not-contact flag
```

### 23.3 Later Features

```text
Hugging Face connector
Kaggle connector
Dribbble connector
Browser helper
Gmail draft creation
Analytics dashboard
Vector search
Saved talent pools
```

---

## 24. Final Selected Design

### 24.1 Selected Product Design

A minimal web dashboard with:

```text
JD upload
Backend process tracker
Generated search queries
Automated source runs
LinkedIn manual-assist section
Candidate ranking
Evidence-backed candidate details
Manual shortlist/outreach workflow
```

### 24.2 Selected Technical Design

```text
Frontend:
Next.js + Tailwind + shadcn/ui

Backend:
FastAPI

Database:
Supabase PostgreSQL

Queue:
Redis + Celery

AI:
OpenAI/Gemini/Anthropic API

Primary Automated Source:
GitHub

Secondary Automated Sources:
Search API, Hugging Face, Kaggle, Dribbble

Manual-Assisted Source:
LinkedIn

Storage:
Supabase Storage or S3-compatible

Deployment:
Vercel + Render/Fly/Railway
```

### 24.3 Selected Architecture Pattern

```text
Layered architecture
+ multi-lane sourcing engine
+ specialized agents
+ event-based process tracking
+ evidence-based scoring
+ human-in-the-loop review
```

---

## 25. Key Design Decisions

### Decision 1: LinkedIn is included but not automated as a scraper

Reason:

- LinkedIn is valuable.
- Full automation is risky.
- Manual-assisted LinkedIn still gives value.
- The system can analyze and rank imported LinkedIn profiles.

### Decision 2: GitHub is the first automated source

Reason:

- Best source for developers.
- API-friendly.
- Evidence-rich.
- Low cost.

### Decision 3: Hugging Face and Kaggle are added for ML/Data

Reason:

- Strong evidence for ML/data candidates.
- Public artifacts are useful.
- Programmatic access exists.

### Decision 4: Designer sourcing is portfolio-first

Reason:

- Designer quality is better judged from work samples.
- LinkedIn title alone is not enough.
- Portfolio evidence is stronger.

### Decision 5: Scoring is evidence-based

Reason:

- Prevents hallucinated matches.
- Makes shortlisting trustworthy.
- Helps the user understand why someone is recommended.

### Decision 6: Frontend stays minimal

Reason:

- Main value is backend sourcing and ranking.
- User wants to upload JD, see backend process, and get candidates.
- Avoid wasting time on visual complexity.

---

## 26. Risks and Mitigations

| Risk | Mitigation |
|---|---|
| LinkedIn automation risk | Manual-assisted LinkedIn lane |
| Low GitHub signal for private-code developers | Add LinkedIn/manual import and search API |
| Stale profile data | Store last-seen date and recency score |
| AI hallucination | Evidence-only explanations |
| Bad candidate matches | Human review and feedback |
| Duplicate profiles | Deduplication layer |
| High API cost | Enrich only shortlisted candidates |
| Designer quality hard to judge | Use portfolio/case-study evidence |
| ML candidates hard to classify | Use Hugging Face/Kaggle/GitHub evidence |
| Too much noise from web search | URL classifier + source filters |

---

## 27. Success Metrics

### 27.1 Sourcing Metrics

```text
Raw candidates found per job
Candidates accepted after filtering
Strong matches per job
Duplicates removed
Source success rate
Average match score
```

### 27.2 Quality Metrics

```text
Shortlist rate
False positive rate
Evidence coverage
Missing must-have rate
Manual review approval rate
```

### 27.3 Outreach Metrics

```text
Contacted candidates
Reply rate
Positive reply rate
Follow-up rate
Do-not-contact count
```

### 27.4 Personal Productivity Metrics

```text
Time spent per job
Leads generated per hour
Good leads per source
Best role categories
Best search queries
```

---

## 28. Recommended First Milestone

The first meaningful milestone should be:

> For one developer job description, automatically collect 50+ candidates from GitHub/public web, manually add 10–20 LinkedIn candidates, score all candidates, and produce a shortlist of 10 strong leads with evidence.

Do this before adding more complexity.

---

## 29. Recommended Build Order

```text
1. Create database schema.
2. Build minimal frontend.
3. Build JD parser.
4. Build query generator.
5. Build process timeline.
6. Build manual candidate import.
7. Build GitHub connector.
8. Build candidate scoring.
9. Build candidate detail/evidence page.
10. Add LinkedIn-assisted workflow.
11. Add search API connector.
12. Add Hugging Face/Kaggle.
13. Add designer portfolio lane.
14. Add outreach drafts.
```

---

## 30. Final Conclusion

The strongest version of this tool is not a LinkedIn scraper. It is a **multi-lane AI sourcing command center**.

It should combine:

```text
Automated developer sourcing from GitHub
Automated ML/data sourcing from Hugging Face and Kaggle
Designer sourcing from portfolios and Dribbble/Behance-style sources
Public web discovery through search APIs
LinkedIn-assisted manual research
Manual imports
Evidence-based scoring
Human-reviewed outreach
```

This design keeps costs low, automates the repetitive work, includes LinkedIn where it matters, and creates a system that can grow from a personal tool into a serious sourcing product later.

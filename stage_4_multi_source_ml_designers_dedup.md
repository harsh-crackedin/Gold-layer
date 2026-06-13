# Stage 4 — Multi-Source Expansion: Public Web, ML/Data Sources, Designer Sources, and Cross-Source Deduplication

**Project:** AI Sourcing Tool  
**Stage:** 4 of 5  
**Depends on:** Stage 1, Stage 2, and Stage 3  
**Purpose:** Expand beyond GitHub into public web discovery, ML/data sources, designer sources, and stronger deduplication.  
**Primary outcome:** The tool can source developers, ML/data candidates, and designers from multiple lanes, merge identities, and produce role-specific ranked results.

---

## 1. Stage Objective

Stage 4 turns the tool from a GitHub-focused developer sourcing app into a multi-source sourcing system.

By the end of this stage, the app should support:

1. Public web discovery through a search API.
2. Portfolio URL classification and extraction.
3. Hugging Face sourcing for ML/AI candidates.
4. Kaggle sourcing for data/ML candidates where feasible.
5. Dribbble and portfolio-first designer sourcing.
6. Behance/manual-assisted designer import.
7. Cross-source candidate deduplication.
8. Role-specific scoring for developer, ML/data, and designer roles.
9. Better evidence extraction across multiple source types.
10. Combined candidate ranking across all sources.

---

## 2. Scope

### Included

```text
Search API connector
URL classifier
Portfolio crawler
Hugging Face connector
Kaggle connector or import connector
Dribbble connector
Designer portfolio extraction
Cross-source deduplication
ML/data scoring
Designer scoring
Multi-source candidate detail
```

### Excluded

```text
Outreach automation
Gmail draft creation
Advanced analytics
Feedback-based learning
Full browser extension
Auto LinkedIn scraping
```

---

## 3. High-Level Flow

```text
Job criteria
   ↓
Source-specific queries
   ↓
Multi-source collectors:
  - GitHub
  - Public web search
  - Hugging Face
  - Kaggle
  - Dribbble
  - Manual LinkedIn
  - Manual import
   ↓
Raw profile/source data
   ↓
URL classification
   ↓
Profile extraction
   ↓
Normalization
   ↓
Cross-source deduplication
   ↓
Evidence extraction
   ↓
Role-specific scoring
   ↓
Unified ranked candidate list
```

---

## 4. New Source Connectors

Add connectors inside:

```text
app/connectors/
```

New files:

```text
search_api_connector.py
portfolio_crawler.py
huggingface_connector.py
kaggle_connector.py
dribbble_connector.py
```

---

## 5. Public Web Discovery Lane

### 5.1 Purpose

The public web lane finds:

```text
personal portfolios
GitHub Pages
developer blogs
designer portfolios
Hugging Face profiles
Kaggle profiles
Dribbble pages
Behance pages
conference/speaker profiles
personal websites
```

### 5.2 Search API Options

Support one provider initially through an abstraction.

Possible providers:

```text
Brave Search API
SerpAPI
Tavily
Exa
DataForSEO
Bing Web Search API
```

Implement:

```text
SEARCH_PROVIDER
SEARCH_API_KEY
```

The code should allow replacing providers later.

---

### 5.3 Search API Connector

Create:

```text
app/connectors/search_api_connector.py
```

Interface:

```python
class SearchAPIConnector:
    def search(self, query: str, limit: int = 10) -> list[SearchResult]:
        ...
```

SearchResult:

```json
{
  "title": "string",
  "url": "string",
  "snippet": "string",
  "source": "web_search"
}
```

---

### 5.4 URL Classifier

Create:

```text
app/services/url_classifier_service.py
```

Classify URLs into:

```text
github_profile
github_repo
linkedin_profile
portfolio_site
huggingface_profile
kaggle_profile
dribbble_profile
behance_profile
blog
company_page
irrelevant
job_post
unknown
```

Rules:

```text
Reject job boards when looking for candidates
Reject company career pages
Reject obvious articles unless author page is useful
Prefer profile pages and portfolio pages
```

---

### 5.5 Portfolio Crawler

Create:

```text
app/connectors/portfolio_crawler.py
```

Purpose:

```text
Fetch public portfolio page
Extract title
Extract visible text
Extract links
Extract email if public
Extract social links
Extract project sections
Extract technologies/tools where possible
```

Constraints:

```text
Respect robots.txt where feasible
Limit pages per domain
No login-required pages
No bypassing anti-bot protections
No aggressive crawling
```

Initial limits:

```text
Max pages per portfolio: 5
Max HTML text per page: 30,000 characters
Max crawl depth: 1
Timeout: 10 seconds
```

---

## 6. Hugging Face Lane

### 6.1 Purpose

Find ML/AI candidates based on:

```text
models
datasets
Spaces
model cards
profile metadata
activity
tags
```

### 6.2 Connector

Create:

```text
app/connectors/huggingface_connector.py
```

Functions:

```text
search_models(query)
search_datasets(query)
search_spaces(query)
fetch_user_profile(username)
fetch_user_artifacts(username)
```

### 6.3 Candidate Evidence

Evidence types:

```text
hf_model
hf_dataset
hf_space
model_card
tag_match
```

### 6.4 ML Signals

Extract:

```text
transformers
llm
nlp
computer vision
diffusion
pytorch
tensorflow
jax
gradio
spaces
dataset work
model deployment
```

---

## 7. Kaggle Lane

### 7.1 Purpose

Find data/ML candidates based on:

```text
notebooks
datasets
competition activity
models
profile pages
```

### 7.2 Connector Options

Kaggle can be implemented in two modes:

#### Mode A: API/CLI Mode

Use Kaggle API/CLI where credentials are configured.

Environment:

```env
KAGGLE_USERNAME=
KAGGLE_KEY=
```

#### Mode B: Manual/Search-Assisted Mode

Use search API to find Kaggle profile URLs.

Example:

```text
site:kaggle.com "machine learning" "India" "notebooks"
site:kaggle.com "data scientist" "Bangalore"
```

### 7.3 Candidate Evidence

Evidence types:

```text
kaggle_notebook
kaggle_dataset
kaggle_competition
kaggle_profile
```

---

## 8. Designer Lane

### 8.1 Sources

Use:

```text
Dribbble
Behance search/manual-assisted
Webflow showcase
Framer community
Personal portfolios
LinkedIn-assisted import
```

### 8.2 Dribbble Connector

Create:

```text
app/connectors/dribbble_connector.py
```

Collect:

```text
profile name
username
profile URL
bio
location
website
shots/projects
tags
descriptions
social links where public
```

### 8.3 Behance Strategy

Because Behance API access may vary, implement Behance as:

```text
Search API discovery + manual import + portfolio crawler
```

The app should generate Behance queries:

```text
site:behance.net "UX case study" "India"
site:behance.net "Product Designer" "SaaS"
```

### 8.4 Designer Evidence

Evidence types:

```text
portfolio_available
case_study
figma
webflow
framer
ui_design
ux_research
design_system
product_design
visual_design
```

---

## 9. Cross-Source Deduplication

Stage 4 should add stronger deduplication.

Create:

```text
app/services/deduplication_service.py
```

### 9.1 Deduplication Signals

| Signal | Confidence |
|---|---:|
| Same email | 100 |
| Same source URL | 100 |
| Same website | 90 |
| GitHub linked from portfolio | 90 |
| Same name + same website | 90 |
| Same name + same location + same role | 70 |
| Same LinkedIn URL | 100 |
| Same portfolio URL | 100 |
| Same username across platforms | 60 |
| Same name only | 20 |

### 9.2 Merge Rules

```text
95–100: auto-merge
70–94: mark as possible duplicate
below 70: keep separate
```

Add field to candidate table if needed:

```sql
canonical_candidate_id UUID NULL REFERENCES candidates(id)
```

Alternative:

Create `candidate_merge_suggestions`.

```sql
CREATE TABLE candidate_merge_suggestions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  candidate_a_id UUID NOT NULL REFERENCES candidates(id),
  candidate_b_id UUID NOT NULL REFERENCES candidates(id),
  confidence NUMERIC NOT NULL,
  signals_json JSONB,
  status TEXT NOT NULL DEFAULT 'pending',
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## 10. Multi-Source Evidence Extraction

Update evidence service to support:

```text
github
huggingface
kaggle
dribbble
portfolio
web_search
linkedin_manual
manual
```

Each evidence row should include:

```text
requirement
evidence_type
source
source_url
text_snippet
confidence
```

---

## 11. Role-Specific Scoring

### 11.1 Developer Score

Keep Stage 3 scoring but allow web/portfolio evidence to improve score.

| Component | Points |
|---|---:|
| Must-have skill evidence | 35 |
| Project evidence | 20 |
| Role/title similarity | 10 |
| Recent activity | 10 |
| Location match | 10 |
| Nice-to-have skills | 10 |
| Contactability | 5 |

---

### 11.2 ML/Data Score

| Component | Points |
|---|---:|
| ML framework evidence | 20 |
| Project/model/notebook evidence | 25 |
| Domain relevance | 10 |
| MLOps/deployment evidence | 15 |
| Recent activity | 10 |
| Location match | 10 |
| Contactability | 10 |

Signals:

```text
PyTorch
TensorFlow
JAX
Transformers
LLM
NLP
Computer Vision
MLOps
Docker
Kubernetes
FastAPI
Model deployment
Hugging Face models
Kaggle notebooks
```

---

### 11.3 Designer Score

| Component | Points |
|---|---:|
| Portfolio availability | 20 |
| Relevant case studies | 25 |
| Visual/product relevance | 15 |
| Tool/domain match | 10 |
| Portfolio recency | 10 |
| Location match | 10 |
| Contactability | 10 |

Signals:

```text
Figma
Framer
Webflow
UX research
case study
design system
product design
SaaS dashboard
mobile app
branding
UI kit
prototype
```

---

## 12. Database Additions

### 12.1 `candidate_merge_suggestions`

```sql
CREATE TABLE candidate_merge_suggestions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  candidate_a_id UUID NOT NULL REFERENCES candidates(id) ON DELETE CASCADE,
  candidate_b_id UUID NOT NULL REFERENCES candidates(id) ON DELETE CASCADE,
  confidence NUMERIC NOT NULL,
  signals_json JSONB,
  status TEXT NOT NULL DEFAULT 'pending',
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### 12.2 Optional `source_artifacts`

Use this if source-specific data becomes too large for `candidate_sources`.

```sql
CREATE TABLE source_artifacts (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  candidate_source_id UUID NOT NULL REFERENCES candidate_sources(id) ON DELETE CASCADE,
  artifact_type TEXT NOT NULL,
  artifact_url TEXT,
  title TEXT,
  description TEXT,
  raw_text TEXT,
  metadata_json JSONB,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

Examples:

```text
github_repo
readme
hf_model
hf_dataset
kaggle_notebook
dribbble_shot
portfolio_page
```

---

## 13. Worker Process

Create separate tasks:

```text
run_web_discovery(job_id)
run_huggingface_sourcing(job_id)
run_kaggle_sourcing(job_id)
run_dribbble_sourcing(job_id)
run_cross_source_deduplication(job_id)
run_role_specific_scoring(job_id)
```

Process events:

```text
WEB_DISCOVERY_STARTED
WEB_DISCOVERY_COMPLETED
PORTFOLIO_CRAWL_STARTED
PORTFOLIO_CRAWL_COMPLETED
HUGGINGFACE_SOURCING_STARTED
HUGGINGFACE_SOURCING_COMPLETED
KAGGLE_SOURCING_STARTED
KAGGLE_SOURCING_COMPLETED
DRIBBBLE_SOURCING_STARTED
DRIBBBLE_SOURCING_COMPLETED
DEDUPLICATION_STARTED
DEDUPLICATION_COMPLETED
ROLE_SCORING_STARTED
ROLE_SCORING_COMPLETED
STAGE_4_COMPLETED
```

---

## 14. Backend API Additions

### 14.1 POST `/jobs/{job_id}/run-web-discovery`

Starts public web discovery.

### 14.2 POST `/jobs/{job_id}/run-ml-sourcing`

Starts Hugging Face and Kaggle sourcing.

### 14.3 POST `/jobs/{job_id}/run-designer-sourcing`

Starts Dribbble/portfolio designer sourcing.

### 14.4 POST `/jobs/{job_id}/run-all-sources`

Runs all relevant source lanes for the job role category.

Logic:

```text
developer:
  GitHub + Web + LinkedIn-assisted

ml_data:
  GitHub + Hugging Face + Kaggle + Web + LinkedIn-assisted

designer:
  Dribbble + Web/Portfolio + Behance queries + LinkedIn-assisted

other:
  Web + LinkedIn-assisted + manual import
```

### 14.5 GET `/jobs/{job_id}/merge-suggestions`

Returns possible duplicates.

### 14.6 POST `/merge-suggestions/{id}/accept`

Merges candidates.

### 14.7 POST `/merge-suggestions/{id}/reject`

Rejects merge.

---

## 15. Frontend Changes

### 15.1 Source Control Panel

On job page, add source controls:

```text
Run GitHub
Run Public Web
Run ML Sources
Run Designer Sources
Run All Relevant Sources
```

Show source readiness:

```text
GitHub token configured: yes/no
Search API configured: yes/no
Hugging Face token configured: optional
Kaggle configured: yes/no
Dribbble configured: yes/no
```

---

### 15.2 Candidate Results

Add source badges:

```text
GitHub
Portfolio
LinkedIn
Hugging Face
Kaggle
Dribbble
Behance
Manual
```

Add role filter:

```text
Developer
ML/Data
Designer
Other
```

Add evidence filter:

```text
Has portfolio
Has GitHub
Has ML artifact
Has design case study
Has contact
```

---

### 15.3 Candidate Detail

Add multi-source sections:

```text
Source Profiles
GitHub Repos
Hugging Face Models
Kaggle Artifacts
Design Portfolio
LinkedIn Manual Data
Web/Portfolio Pages
Evidence
Score Breakdown
Merge/Duplicate Info
```

---

### 15.4 Merge Suggestions Page

Create:

```text
/jobs/[jobId]/merge-suggestions
```

Show:

| Candidate A | Candidate B | Confidence | Signals | Action |
|---|---|---:|---|---|

Actions:

```text
Accept Merge
Reject
View Both
```

---

## 16. Source Limits

Use configurable limits.

```env
MAX_WEB_RESULTS_PER_QUERY=10
MAX_PORTFOLIO_PAGES=5
MAX_HF_RESULTS_PER_QUERY=20
MAX_KAGGLE_RESULTS_PER_QUERY=20
MAX_DRIBBBLE_RESULTS_PER_QUERY=20
MAX_CANDIDATES_PER_SOURCE=100
```

Default conservative limits are important for cost control.

---

## 17. Stage 4 Acceptance Criteria

Stage 4 is complete when:

1. User can run public web discovery.
2. Web search results are classified.
3. Portfolio pages can be extracted.
4. Hugging Face candidates can be collected or imported.
5. Kaggle candidates can be collected or search-assisted.
6. Dribbble/designer candidates can be collected or imported.
7. Candidates from multiple sources are normalized.
8. Cross-source deduplication creates suggestions.
9. Role-specific scoring works for developer, ML/data, and designer jobs.
10. Candidate detail page shows multi-source evidence.
11. Candidate ranking includes all source lanes.
12. LinkedIn-assisted flow still works.

---

## 18. Non-Goals for Stage 4

Do not implement:

```text
Outreach automation
Email sending
Gmail integration
Feedback learning
Advanced analytics
Chrome extension
Mass LinkedIn automation
```

---

## 19. Instructions for Coding Agent

When implementing Stage 4:

1. Keep each connector isolated.
2. Use a common connector response format.
3. Do not let one source failure fail the whole job.
4. Store raw source artifacts.
5. Normalize all profiles into the same candidate model.
6. Keep role-specific scoring modular.
7. Add rate limits and max result limits.
8. Use deduplication confidence, not blind merging.
9. Preserve evidence-first design.
10. Keep LinkedIn as manual-assisted.

---

## 20. Expected Final State

At the end of Stage 4:

```text
JD → Criteria → Multi-source search → Profile enrichment → Deduplication → Role-specific scoring → Unified ranked lead list
```

The tool now works for developers, ML/data candidates, and designers.

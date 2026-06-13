# Stage 4 — Multi-Source Local Research: Search API, Portfolio Crawler, ML/Data Sources, Designer Sources, and Deduplication

**Project:** AI Sourcing Research Workstation  
**Stage:** 4 of 5  
**Depends on:** Stages 1, 2, and 3  
**Primary goal:** Expand the local workstation beyond GitHub into public web search, portfolio crawling, ML/data sources, designer sources, and cross-source deduplication.

---

## 1. Stage Objective

Stage 4 turns the tool into a multi-source local research system.

By the end of this stage:

1. The app can run long public web discovery jobs.
2. It can call a configurable search provider.
3. It can classify URLs.
4. It can crawl allowed public portfolio pages.
5. It can source ML/data candidates from Hugging Face and Kaggle/search-assisted flows.
6. It can source designers from Dribbble, Behance search-assisted flow, and portfolio pages.
7. It can deduplicate candidates across sources.
8. It can score developer, ML/data, and designer candidates differently.
9. It can store large source artifacts locally.
10. It can resume failed source tasks.

---

## 2. Scope

### Included

```text
Search provider abstraction
Search API connector
URL classifier
Portfolio crawler
Hugging Face connector
Kaggle connector or search-assisted mode
Dribbble connector
Behance search/manual-assisted mode
Cross-source deduplication
ML/data scoring
Designer scoring
Multi-source candidate detail page
Large-scale local research controls
Rate-limit tracking
```

### Excluded

```text
Automatic LinkedIn scraping
Automatic outreach sending
Gmail integration
CRM/ATS integrations
Cloud deployment
Enterprise permissions
```

---

## 3. Multi-Source Flow

```text
Research plan tasks
   ↓
Source-specific workers:
  - GitHub
  - Web Search
  - Portfolio Crawler
  - Hugging Face
  - Kaggle
  - Dribbble
  - LinkedIn Manual
  - Manual Import
   ↓
Raw results and artifacts
   ↓
URL classification
   ↓
Candidate extraction
   ↓
Normalization
   ↓
Deduplication
   ↓
Evidence extraction
   ↓
Role-specific scoring
   ↓
Unified ranked lead list
```

---

## 4. Search Provider Abstraction

Create:

```text
app/services/search/
  search_service.py
  providers/
    base.py
    brave_provider.py
    tavily_provider.py
    exa_provider.py
    serpapi_provider.py
    searxng_provider.py
```

Use environment variable:

```env
SEARCH_PROVIDER=brave
SEARCH_API_KEY=
```

### Search service interface

```python
class SearchProvider:
    def search(self, query: str, limit: int = 10) -> list[SearchResult]: ...
```

SearchResult:

```json
{
  "title": "string",
  "url": "string",
  "snippet": "string",
  "provider": "brave"
}
```

---

## 5. Search Strategy

Use search API to find:

```text
developer portfolios
GitHub profiles
GitHub Pages
Hugging Face profiles
Kaggle profiles
Dribbble profiles
Behance profiles
designer portfolios
personal websites
technical blogs
conference speaker pages
open-source maintainer pages
```

Use limits:

```env
MAX_SEARCH_QUERIES_PER_JOB=100
MAX_RESULTS_PER_QUERY=20
MAX_URLS_TO_CRAWL_PER_JOB=300
```

---

## 6. URL Classifier

Create:

```text
app/services/url_classifier_service.py
```

Classifications:

```text
github_profile
github_repo
linkedin_profile
portfolio_site
personal_blog
huggingface_profile
kaggle_profile
dribbble_profile
behance_profile
webflow_site
framer_site
conference_profile
company_page
job_post
irrelevant
unknown
```

Reject:

```text
job posts
career pages
generic company pages
spam pages
login-required pages
irrelevant articles
```

Keep:

```text
profile pages
portfolio pages
technical blogs
project pages
public professional pages
```

---

## 7. Portfolio Crawler

Create:

```text
app/connectors/portfolio_crawler.py
```

Use simple HTTP fetching first.

Add Playwright only when needed for JS-rendered pages.

### Rules

```text
No login-required crawling
No CAPTCHA bypassing
No aggressive crawling
Limit pages per domain
Respect configured delays
Store raw HTML locally
Extract visible text
Extract emails if public
Extract social links
Extract project links
```

Limits:

```env
MAX_PORTFOLIO_PAGES_PER_DOMAIN=5
REQUEST_DELAY_SECONDS=2
```

Storage path:

```text
/data/storage/jobs/{job_id}/web/{domain}/
```

---

## 8. Playwright Usage

Playwright can be used only for:

```text
public portfolios that require JavaScript rendering
designer portfolio screenshots
manual-assisted browser workflows
```

Do not use Playwright for:

```text
mass LinkedIn scraping
bypassing restrictions
fake accounts
CAPTCHA bypass
automated LinkedIn messaging
```

Create optional service:

```text
app/services/browser_render_service.py
```

---

## 9. Hugging Face Lane

Create:

```text
app/connectors/huggingface_connector.py
```

Collect:

```text
user/profile URL
models
datasets
spaces
model cards
tags
README/model card text
activity metadata
```

Evidence types:

```text
hf_model
hf_dataset
hf_space
model_card
tag_match
ml_framework_match
```

Store artifacts:

```text
/data/storage/jobs/{job_id}/huggingface/{username}/
```

---

## 10. Kaggle Lane

Implement two modes.

### Mode A: API/CLI Mode

Use:

```env
KAGGLE_USERNAME=
KAGGLE_KEY=
```

Collect where feasible:

```text
profile URL
notebooks
datasets
competitions
models
metadata
```

### Mode B: Search-Assisted Mode

Use search API queries:

```text
site:kaggle.com "machine learning" "India" "notebook"
site:kaggle.com "data scientist" "Bangalore"
```

Then classify and import profile-like pages.

---

## 11. Designer Lane

### Sources

```text
Dribbble
Behance via search/manual-assisted
Webflow showcases
Framer sites
Personal portfolios
LinkedIn manual-assisted
```

### Dribbble connector

Create:

```text
app/connectors/dribbble_connector.py
```

Collect:

```text
profile
shots/projects
tags
descriptions
website
location
bio
```

### Behance

Implement as search/manual-assisted initially:

```text
Generate queries
Search web
Classify Behance profile/project pages
Allow manual import
Extract visible public content where allowed
```

---

## 12. Candidate Extraction from Web/Portfolio Text

Create:

```text
app/services/candidate_extraction_service.py
```

Use:

```text
rules first
Gemini only for promising/complex pages
AI cache via ai_runs
```

Extract:

```text
name
headline
role category
skills/tools
location
email
website
social links
project summaries
evidence snippets
```

---

## 13. Cross-Source Deduplication

Create:

```text
app/services/deduplication_service.py
```

Add table:

```sql
CREATE TABLE candidate_merge_suggestions (
  id TEXT PRIMARY KEY,
  candidate_a_id TEXT NOT NULL,
  candidate_b_id TEXT NOT NULL,
  confidence REAL NOT NULL,
  signals_json TEXT,
  status TEXT NOT NULL DEFAULT 'pending',
  created_at TEXT NOT NULL,
  FOREIGN KEY(candidate_a_id) REFERENCES candidates(id) ON DELETE CASCADE,
  FOREIGN KEY(candidate_b_id) REFERENCES candidates(id) ON DELETE CASCADE
);
```

Signals:

| Signal | Confidence |
|---|---:|
| Same email | 100 |
| Same LinkedIn URL | 100 |
| Same GitHub URL | 100 |
| Same website | 90 |
| GitHub linked from portfolio | 90 |
| Same name + same website | 90 |
| Same name + same location + same role | 70 |
| Same username across platforms | 60 |
| Same name only | 20 |

Merge rules:

```text
95–100: auto-merge
70–94: create merge suggestion
below 70: keep separate
```

---

## 14. Multi-Role Scoring

Create scoring services:

```text
app/services/scoring/
  developer_scoring_service.py
  ml_data_scoring_service.py
  designer_scoring_service.py
  scoring_router.py
```

### Developer score

| Component | Points |
|---|---:|
| Must-have skill evidence | 35 |
| Project evidence | 20 |
| Role/title similarity | 10 |
| Recent activity | 10 |
| Location match | 10 |
| Nice-to-have skills | 10 |
| Contactability | 5 |

### ML/Data score

| Component | Points |
|---|---:|
| ML framework evidence | 20 |
| Project/model/notebook evidence | 25 |
| Domain relevance | 10 |
| MLOps/deployment evidence | 15 |
| Recent activity | 10 |
| Location match | 10 |
| Contactability | 10 |

### Designer score

| Component | Points |
|---|---:|
| Portfolio availability | 20 |
| Relevant case studies | 25 |
| Product/visual relevance | 15 |
| Tool/domain match | 10 |
| Portfolio recency | 10 |
| Location match | 10 |
| Contactability | 10 |

---

## 15. Worker Tasks

Add:

```text
run_web_search_task(task_id)
run_url_classification_task(job_id)
run_portfolio_crawl_task(task_id)
run_huggingface_task(task_id)
run_kaggle_task(task_id)
run_dribbble_task(task_id)
run_candidate_extraction_task(job_id, artifact_id)
run_deduplication_task(job_id)
run_role_specific_scoring_task(job_id)
run_all_relevant_sources(job_id)
```

Process events:

```text
WEB_SEARCH_STARTED
WEB_SEARCH_COMPLETED
URL_CLASSIFICATION_STARTED
URL_CLASSIFICATION_COMPLETED
PORTFOLIO_CRAWL_STARTED
PORTFOLIO_CRAWL_COMPLETED
HUGGINGFACE_STARTED
HUGGINGFACE_COMPLETED
KAGGLE_STARTED
KAGGLE_COMPLETED
DRIBBBLE_STARTED
DRIBBBLE_COMPLETED
DEDUPLICATION_STARTED
DEDUPLICATION_COMPLETED
ROLE_SCORING_STARTED
ROLE_SCORING_COMPLETED
STAGE_4_COMPLETED
```

---

## 16. Backend API Additions

```text
POST /jobs/{job_id}/run-web-discovery
POST /jobs/{job_id}/run-ml-sources
POST /jobs/{job_id}/run-designer-sources
POST /jobs/{job_id}/run-all-sources
GET /jobs/{job_id}/merge-suggestions
POST /merge-suggestions/{id}/accept
POST /merge-suggestions/{id}/reject
GET /jobs/{job_id}/artifacts
GET /jobs/{job_id}/source-summary
```

---

## 17. Frontend Changes

### Source control panel

Add buttons:

```text
Run GitHub
Run Web Discovery
Run ML Sources
Run Designer Sources
Run All Relevant Sources
Resume Failed Tasks
Pause Research
```

### Candidate table

Add source badges:

```text
GitHub
Portfolio
LinkedIn Manual
Hugging Face
Kaggle
Dribbble
Behance
Manual
```

### Candidate detail page

Add sections:

```text
Source Profiles
GitHub Repos
Portfolio Pages
Hugging Face Artifacts
Kaggle Artifacts
Design Portfolio
LinkedIn Manual Data
Evidence
Score Breakdown
Merge Suggestions
Local Artifacts
```

### Merge suggestions page

Show:

```text
Candidate A
Candidate B
Confidence
Signals
Accept
Reject
View Both
```

---

## 18. Large-Scale Local Controls

Add settings page:

```text
Max candidates per job
Max search results per query
Max URLs to crawl
Max GitHub users
Max repos per user
Request delay
Worker concurrency
AI usage limit
```

These settings should read/write local config.

---

## 19. Stage 4 Acceptance Criteria

Stage 4 is complete when:

1. Search provider abstraction works.
2. Public web search tasks run locally.
3. URLs are classified.
4. Portfolio pages are crawled within limits.
5. Hugging Face lane works or supports search-assisted import.
6. Kaggle lane works or supports search-assisted import.
7. Dribbble/designer lane works.
8. Large artifacts are stored locally.
9. Candidates are normalized from multiple sources.
10. Deduplication suggestions are created.
11. Developer, ML/data, and designer scoring work.
12. Candidate ranking combines all sources.
13. Failed/rate-limited tasks can be retried.
14. LinkedIn remains manual-assisted, not scraped.

---

## 20. Non-Goals

Do not implement:

```text
automatic outreach
Gmail drafts
Chrome extension
LinkedIn scraping
cloud deployment
billing
multi-user permissions
```

---

## 21. Instructions for Coding Agent

1. Keep each source connector isolated.
2. Use research_tasks for every long operation.
3. Store raw content in local files.
4. Do not overload SQLite with large HTML/text.
5. Use AI sparingly and cache results.
6. Respect limits and delays.
7. Do not let one source failure kill the whole job.
8. Keep LinkedIn manual-assisted.
9. Make all tasks resumable.
10. Preserve evidence-first candidate ranking.

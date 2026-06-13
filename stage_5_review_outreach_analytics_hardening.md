# Stage 5 — Review Workflow, Outreach Drafts, Feedback Loop, Analytics, Export, and Production Hardening

**Project:** AI Sourcing Tool  
**Stage:** 5 of 5  
**Depends on:** Stages 1–4  
**Purpose:** Turn the sourcing engine into a usable personal lead-generation workflow with review, outreach drafts, feedback, analytics, export, and deployment hardening.  
**Primary outcome:** The user can run sourcing, review ranked candidates, generate personalized outreach drafts, track contact status, export lists, and learn which sources/queries produce the best leads.

---

## 1. Stage Objective

Stage 5 completes the practical workflow.

By the end of this stage, the app should support:

1. Candidate review pipeline.
2. Shortlist/reject/contact-ready workflow.
3. Outreach draft generation.
4. Contact tracking.
5. Do-not-contact handling.
6. CSV export.
7. Source performance analytics.
8. Query performance analytics.
9. Feedback capture.
10. Production-ready configuration.
11. Better error handling and observability.
12. Deployment documentation.

---

## 2. Scope

### Included

```text
Review workflow
Outreach draft generation
Manual send support
Optional Gmail draft design hook
Feedback events
Analytics dashboard
CSV export
Saved talent pools
Do-not-contact handling
Production hardening
Deployment docs
```

### Excluded

```text
Automatic bulk email sending
Automatic LinkedIn messaging
Paid enrichment by default
Full ATS/CRM product
Multi-user team permissions
```

---

## 3. Final Product Flow

```text
Upload JD
   ↓
Parse criteria
   ↓
Generate queries
   ↓
Run source lanes
   ↓
Collect candidates
   ↓
Normalize + dedupe
   ↓
Score + explain
   ↓
Review candidates
   ↓
Shortlist
   ↓
Generate outreach draft
   ↓
Manually contact
   ↓
Track reply/status
   ↓
Analyze source performance
```

---

## 4. Review Workflow

### 4.1 Candidate Statuses

Use the following statuses:

```text
new
auto_scored
needs_review
shortlisted
rejected
contact_ready
contacted
replied
follow_up
do_not_contact
archived
```

### 4.2 Recommended Status Transitions

```text
new → auto_scored
auto_scored → needs_review
needs_review → shortlisted
needs_review → rejected
shortlisted → contact_ready
contact_ready → contacted
contacted → replied
contacted → follow_up
any → do_not_contact
any → archived
```

### 4.3 Review Actions

On candidate table and detail page, add:

```text
Shortlist
Reject
Mark Contact Ready
Generate Outreach
Mark Contacted
Mark Replied
Mark Follow-Up
Do Not Contact
Archive
Add Note
```

---

## 5. Feedback Events

Add `feedback_events`.

```sql
CREATE TABLE feedback_events (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  job_id UUID NOT NULL REFERENCES jobs(id) ON DELETE CASCADE,
  candidate_id UUID REFERENCES candidates(id) ON DELETE CASCADE,
  event_type TEXT NOT NULL,
  reason TEXT,
  metadata_json JSONB,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

Recommended `event_type` values:

```text
shortlisted
rejected
bad_match
good_match
contact_ready
contacted
replied
positive_reply
negative_reply
do_not_contact
duplicate
wrong_location
missing_skill
too_junior
too_senior
wrong_role
weak_evidence
```

Feedback is critical because it helps the user improve sourcing quality manually and can later train ranking adjustments.

---

## 6. Outreach Draft Generation

### 6.1 Purpose

The system should create short, specific, non-spammy outreach drafts.

It should not auto-send messages.

### 6.2 Outreach Draft Table

Create:

```sql
CREATE TABLE outreach_drafts (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  job_id UUID NOT NULL REFERENCES jobs(id) ON DELETE CASCADE,
  candidate_id UUID NOT NULL REFERENCES candidates(id) ON DELETE CASCADE,
  channel TEXT NOT NULL DEFAULT 'email',
  subject TEXT,
  body TEXT NOT NULL,
  status TEXT NOT NULL DEFAULT 'draft',
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

Recommended `channel` values:

```text
email
linkedin
twitter
portfolio_contact_form
other
```

Recommended `status` values:

```text
draft
approved
used
archived
```

---

## 7. Outreach Draft Prompt

Use evidence-only personalization.

Prompt:

```text
You are writing a short sourcing outreach draft.

Rules:
- Be concise.
- Use only the provided candidate evidence.
- Do not exaggerate.
- Do not claim the candidate is perfect.
- Mention one specific reason for reaching out.
- Do not mention sensitive traits.
- Do not sound automated.
- End with a low-pressure question.
```

Input:

```json
{
  "job_title": "Python Backend Engineer",
  "candidate_name": "Amit",
  "matched_evidence": [
    "GitHub repo uses FastAPI",
    "README mentions PostgreSQL",
    "Profile location says India"
  ],
  "missing_requirements": ["AWS not confirmed"]
}
```

Output:

```text
Hi Amit, I found your FastAPI/PostgreSQL project while sourcing for a Python backend role. Your backend API work looked relevant to what this role needs. Would you be open to hearing a few details?
```

---

## 8. Outreach Rules

Hard rules:

```text
Do not auto-send.
Do not generate outreach for do_not_contact candidates.
Do not generate outreach for candidates below score threshold unless user forces it.
Do not include unverified claims.
Do not mention protected attributes.
Do not use manipulative language.
```

Default thresholds:

```text
match_score >= 70
confidence_score >= 50
status in shortlisted/contact_ready
do_not_contact = false
```

---

## 9. Contact Tracking

Extend `lead_status` if needed.

Add:

```sql
ALTER TABLE lead_status
ADD COLUMN contact_channel TEXT,
ADD COLUMN last_outreach_draft_id UUID REFERENCES outreach_drafts(id),
ADD COLUMN reply_status TEXT,
ADD COLUMN next_follow_up_at TIMESTAMPTZ;
```

Recommended `reply_status` values:

```text
none
positive
negative
neutral
no_response
```

---

## 10. Export

### 10.1 CSV Export

Add endpoint:

```text
GET /jobs/{job_id}/export/candidates.csv
```

CSV columns:

```text
candidate_id
name
headline
location
email
website
sources
source_urls
match_score
confidence_score
contactability_score
final_action_score
status
matched_requirements
missing_requirements
evidence_summary
notes
last_contacted_at
do_not_contact
```

### 10.2 Export Filters

Support query params:

```text
status=shortlisted
min_score=70
source=github
has_email=true
```

Example:

```text
/jobs/{job_id}/export/candidates.csv?status=shortlisted&min_score=70
```

---

## 11. Analytics Dashboard

Create:

```text
/jobs/[jobId]/analytics
```

### 11.1 Job-Level Metrics

Show:

```text
Total raw candidates
Total normalized candidates
Total duplicates/merge suggestions
Total scored candidates
Strong matches
Shortlisted candidates
Contact-ready candidates
Contacted candidates
Replies
```

### 11.2 Source Metrics

For each source:

```text
Raw candidates found
Candidates scored
Average match score
Shortlist rate
Contactability rate
Rejected rate
Strong match count
```

Example table:

| Source | Found | Scored | Avg Score | Shortlisted | Contactable |
|---|---:|---:|---:|---:|---:|
| GitHub | 80 | 52 | 71 | 14 | 8 |
| LinkedIn Manual | 15 | 15 | 78 | 7 | 2 |
| Hugging Face | 30 | 18 | 69 | 5 | 3 |

### 11.3 Query Metrics

Track:

```text
Query text
Source
Candidates found
Candidates accepted
Average score
Best candidate score
```

This helps the user improve future sourcing.

---

## 12. Saved Talent Pools

Add the ability to save candidates independent of one job.

Create:

```sql
CREATE TABLE talent_pools (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name TEXT NOT NULL,
  description TEXT,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE talent_pool_candidates (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  talent_pool_id UUID NOT NULL REFERENCES talent_pools(id) ON DELETE CASCADE,
  candidate_id UUID NOT NULL REFERENCES candidates(id) ON DELETE CASCADE,
  notes TEXT,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

Example pools:

```text
Python Backend India
ML Engineers India
Product Designers SaaS
High-Quality Future Leads
```

---

## 13. Backend API Additions

### 13.1 Candidate Review

```text
PATCH /jobs/{job_id}/candidates/{candidate_id}/status
POST /jobs/{job_id}/candidates/{candidate_id}/feedback
```

### 13.2 Outreach

```text
POST /jobs/{job_id}/candidates/{candidate_id}/outreach-drafts
GET /jobs/{job_id}/candidates/{candidate_id}/outreach-drafts
PATCH /outreach-drafts/{draft_id}
```

### 13.3 Export

```text
GET /jobs/{job_id}/export/candidates.csv
```

### 13.4 Analytics

```text
GET /jobs/{job_id}/analytics
GET /jobs/{job_id}/analytics/sources
GET /jobs/{job_id}/analytics/queries
```

### 13.5 Talent Pools

```text
POST /talent-pools
GET /talent-pools
POST /talent-pools/{pool_id}/candidates
GET /talent-pools/{pool_id}/candidates
DELETE /talent-pools/{pool_id}/candidates/{candidate_id}
```

---

## 14. Frontend Changes

### 14.1 Candidate Review Queue

Create:

```text
/jobs/[jobId]/review
```

Show candidates one by one or in table view.

Candidate review card:

```text
Name
Headline
Source badges
Match score
Confidence score
Top evidence
Missing requirements
Actions
```

Actions:

```text
Shortlist
Reject
Need More Info
Generate Outreach
Do Not Contact
```

---

### 14.2 Outreach Draft UI

On candidate detail:

```text
Generate Outreach Draft
Edit Draft
Copy Draft
Mark Used
```

Draft display:

```text
Channel
Subject
Body
Status
Created At
```

---

### 14.3 Analytics Page

Add charts or simple tables.

For minimal implementation, tables are enough.

Sections:

```text
Summary Metrics
Source Performance
Query Performance
Status Funnel
```

---

### 14.4 Export Button

Add export button to:

```text
Job candidates page
Shortlisted filter view
Analytics page
```

---

### 14.5 Do-Not-Contact Handling

If `do_not_contact = true`:

- Show red badge.
- Disable outreach generation.
- Disable mark contact-ready.
- Allow archive only.

---

## 15. Production Hardening

### 15.1 Configuration

Add `.env.example` with all final variables.

```env
DATABASE_URL=
REDIS_URL=
NEXT_PUBLIC_API_URL=
OPENAI_API_KEY=
GEMINI_API_KEY=
ANTHROPIC_API_KEY=
GITHUB_TOKEN=
SEARCH_PROVIDER=
SEARCH_API_KEY=
HUGGINGFACE_TOKEN=
KAGGLE_USERNAME=
KAGGLE_KEY=
DRIBBBLE_CLIENT_ID=
DRIBBBLE_CLIENT_SECRET=
MAX_CANDIDATES_PER_JOB=300
MAX_GITHUB_USERS_PER_JOB=80
MAX_WEB_RESULTS_PER_QUERY=10
```

### 15.2 Error Handling

Every source connector should:

```text
catch API errors
create process event
continue other sources
store partial results
avoid crashing whole job
```

### 15.3 Observability

Add:

```text
Sentry
structured logs
process_events
source error events
API latency logs
```

### 15.4 Security

For personal use:

```text
Basic auth or Supabase Auth
Do not expose API keys to frontend
Use server-side environment variables
Sanitize profile text
Avoid storing unnecessary sensitive data
```

### 15.5 Data Hygiene

Add:

```text
Delete candidate
Archive candidate
Delete job
Export job data
Do-not-contact flag
```

---

## 16. Deployment Plan

### 16.1 Recommended Deployment

Frontend:

```text
Vercel
```

Backend:

```text
Render / Fly.io / Railway
```

Database:

```text
Supabase Postgres
```

Redis:

```text
Upstash Redis / Render Redis / Railway Redis
```

Worker:

```text
Render worker service / Railway worker / Fly machine
```

### 16.2 Deployment Checklist

```text
Set environment variables
Run database migrations
Start backend API
Start worker
Start frontend
Verify health endpoint
Create test job
Run Stage 2 parser
Run GitHub sourcing
Verify process events
Export candidates CSV
```

---

## 17. Quality Checks

Before marking Stage 5 complete:

1. Create a developer JD.
2. Run full source pipeline.
3. Review candidate results.
4. Shortlist candidates.
5. Generate outreach drafts.
6. Export shortlisted candidates.
7. Check analytics.
8. Mark one candidate as do-not-contact.
9. Confirm outreach is disabled for that candidate.
10. Confirm logs and process events show failures clearly.

---

## 18. Stage 5 Acceptance Criteria

Stage 5 is complete when:

1. User can review candidates in a workflow.
2. User can shortlist/reject candidates.
3. User can generate outreach drafts.
4. Drafts use candidate evidence.
5. User can mark candidates contacted/replied.
6. Do-not-contact prevents outreach.
7. User can export candidates to CSV.
8. Analytics show source and query performance.
9. Talent pools can save candidates.
10. Production environment variables are documented.
11. App can be deployed using provided deployment plan.
12. Errors are visible without crashing the entire job.

---

## 19. Non-Goals for Stage 5

Do not implement:

```text
Automatic bulk sending
LinkedIn auto-messaging
Proxy-based scraping
CAPTCHA bypassing
Enterprise multi-user RBAC
Billing
ATS marketplace integration
```

---

## 20. Instructions for Coding Agent

When implementing Stage 5:

1. Preserve evidence-first design.
2. Do not auto-send outreach.
3. Never generate outreach for do-not-contact leads.
4. Keep analytics simple but useful.
5. Make export reliable.
6. Add clear UI for candidate statuses.
7. Keep feedback events structured.
8. Make deployment documentation practical.
9. Ensure all prior stages still work.
10. Favor simple, maintainable code over complex automation.

---

## 21. Final Expected Product

At the end of Stage 5, the user has a usable personal AI sourcing tool:

```text
Upload JD
→ Watch backend process
→ Get automated leads
→ Add LinkedIn/manual leads
→ Review evidence-backed candidates
→ Shortlist
→ Generate outreach
→ Track status
→ Export
→ Learn which sources work
```

This completes the first full version of the AI sourcing system.

# Stage 5 — Review Workflow, Outreach Drafts, Analytics, Export, Backup, and Local Production Hardening

**Project:** AI Sourcing Research Workstation  
**Stage:** 5 of 5  
**Depends on:** Stages 1–4  
**Primary goal:** Complete the local sourcing workflow so the user can review candidates, generate outreach drafts, track recommendations, export results, analyze source quality, and maintain the local database.

---

## 1. Stage Objective

Stage 5 turns the local research engine into a complete personal sourcing workflow.

By the end of this stage:

1. Candidate review workflow is complete.
2. Shortlist/reject/contact-ready actions work.
3. Gemini generates evidence-based outreach drafts.
4. Do-not-contact protections exist.
5. Contact tracking exists.
6. CSV/Markdown exports work.
7. Analytics show source/query performance.
8. Talent pools can be created.
9. Local backup and restore exist.
10. Job archive and cleanup exist.
11. The local app is stable for repeated weekly sourcing.

---

## 2. Final Product Flow

```text
New JD arrives
   ↓
Create local job
   ↓
Gemini parses JD
   ↓
System builds research plan
   ↓
Run automated sources
   ↓
Use LinkedIn manual-assisted lane
   ↓
Normalize and dedupe candidates
   ↓
Score and explain matches
   ↓
Review top candidates
   ↓
Generate outreach drafts
   ↓
Manually contact candidates
   ↓
Track outcome
   ↓
Export recommendations
   ↓
Analyze source performance
```

---

## 3. Review Workflow

Candidate statuses:

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

Recommended transitions:

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

Actions:

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

## 4. Feedback Events

Add:

```sql
CREATE TABLE feedback_events (
  id TEXT PRIMARY KEY,
  job_id TEXT NOT NULL,
  candidate_id TEXT,
  event_type TEXT NOT NULL,
  reason TEXT,
  metadata_json TEXT,
  created_at TEXT NOT NULL,
  FOREIGN KEY(job_id) REFERENCES jobs(id) ON DELETE CASCADE,
  FOREIGN KEY(candidate_id) REFERENCES candidates(id) ON DELETE CASCADE
);
```

Event types:

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

Purpose:

```text
Improve future sourcing
Understand why candidates were rejected
Track what produces good leads
Support manual tuning later
```

---

## 5. Outreach Drafts

Add:

```sql
CREATE TABLE outreach_drafts (
  id TEXT PRIMARY KEY,
  job_id TEXT NOT NULL,
  candidate_id TEXT NOT NULL,
  channel TEXT NOT NULL DEFAULT 'email',
  subject TEXT,
  body TEXT NOT NULL,
  status TEXT NOT NULL DEFAULT 'draft',
  created_at TEXT NOT NULL,
  updated_at TEXT NOT NULL,
  FOREIGN KEY(job_id) REFERENCES jobs(id) ON DELETE CASCADE,
  FOREIGN KEY(candidate_id) REFERENCES candidates(id) ON DELETE CASCADE
);
```

Channels:

```text
email
linkedin
twitter
portfolio_contact_form
other
```

Statuses:

```text
draft
approved
used
archived
```

---

## 6. Gemini Outreach Generation

Use Gemini through AIService.

Rules:

```text
Use only candidate evidence.
Do not exaggerate.
Do not claim they are perfect.
Mention one specific reason for contact.
Keep it short.
Do not mention protected traits.
End with a low-pressure question.
Do not generate for do-not-contact candidates.
```

Input context:

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

```json
{
  "subject": "Python backend role",
  "body": "Hi Amit, I found your FastAPI/PostgreSQL project while sourcing for a Python backend role. Your backend API work looked relevant to what this role needs. Would you be open to hearing a few details?"
}
```

---

## 7. Outreach Safeguards

Hard rules:

```text
No automatic bulk sending.
No LinkedIn auto-messaging.
No outreach for do_not_contact candidates.
No outreach for candidates below threshold unless user explicitly overrides.
No unverified claims.
No protected attribute references.
No manipulative language.
```

Default generation threshold:

```text
match_score >= 70
confidence_score >= 50
do_not_contact = false
status in shortlisted/contact_ready
```

---

## 8. Contact Tracking

Extend `lead_status`.

```sql
ALTER TABLE lead_status ADD COLUMN contact_channel TEXT;
ALTER TABLE lead_status ADD COLUMN last_outreach_draft_id TEXT;
ALTER TABLE lead_status ADD COLUMN reply_status TEXT;
ALTER TABLE lead_status ADD COLUMN next_follow_up_at TEXT;
```

Reply statuses:

```text
none
positive
negative
neutral
no_response
```

---

## 9. Export System

### 9.1 CSV Export

Endpoint:

```text
GET /jobs/{job_id}/export/candidates.csv
```

Columns:

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
local_artifact_paths
```

### 9.2 Markdown Report Export

Endpoint:

```text
GET /jobs/{job_id}/export/report.md
```

Report sections:

```text
Job summary
Extracted criteria
Sources used
Top candidates
Candidate evidence
Missing requirements
Recommended shortlist
Manual LinkedIn notes
Source performance
```

Store exports locally:

```text
/data/storage/jobs/{job_id}/exports/
```

---

## 10. Analytics Dashboard

Create page:

```text
/jobs/[jobId]/analytics
```

### Job metrics

```text
Total research tasks
Completed tasks
Failed tasks
Raw artifacts collected
Candidates created
Duplicates found
Candidates scored
Strong matches
Shortlisted candidates
Contact-ready candidates
Contacted candidates
Replies
```

### Source metrics

For each source:

```text
Raw results
Candidate records
Average match score
Strong candidates
Shortlisted count
Contactability rate
Failure count
Cost estimate if available
```

### Query metrics

For each query:

```text
Query text
Source
Results found
Candidates accepted
Average score
Best candidate score
```

### Funnel

```text
Raw results
→ Relevant URLs
→ Candidate records
→ Scored candidates
→ Strong matches
→ Shortlisted
→ Contacted
→ Replies
```

---

## 11. Talent Pools

Add:

```sql
CREATE TABLE talent_pools (
  id TEXT PRIMARY KEY,
  name TEXT NOT NULL,
  description TEXT,
  created_at TEXT NOT NULL
);

CREATE TABLE talent_pool_candidates (
  id TEXT PRIMARY KEY,
  talent_pool_id TEXT NOT NULL,
  candidate_id TEXT NOT NULL,
  notes TEXT,
  created_at TEXT NOT NULL,
  FOREIGN KEY(talent_pool_id) REFERENCES talent_pools(id) ON DELETE CASCADE,
  FOREIGN KEY(candidate_id) REFERENCES candidates(id) ON DELETE CASCADE
);
```

Example pools:

```text
Python Backend India
ML Engineers India
Product Designers SaaS
Strong Future Leads
```

---

## 12. Local Backup and Restore

Because this is local-only, backup matters.

### Backup command

Create endpoint or CLI command:

```text
POST /maintenance/backup
```

It should create:

```text
/data/backups/backup_{timestamp}.zip
```

Include:

```text
SQLite DB file
WAL/SHM files if needed
storage artifacts
exports
config snapshot without API secrets
```

### Restore command

Optional endpoint/CLI:

```text
POST /maintenance/restore
```

For safety, implement restore as CLI first, not frontend.

---

## 13. Cleanup and Archive

Add:

```text
Archive job
Delete job
Delete candidate
Delete source artifacts
Clear failed tasks
Vacuum database
```

SQLite maintenance:

```sql
VACUUM;
PRAGMA wal_checkpoint(TRUNCATE);
```

Add CLI command:

```text
make db-maintain
```

---

## 14. Backend API Additions

### Review

```text
PATCH /jobs/{job_id}/candidates/{candidate_id}/status
POST /jobs/{job_id}/candidates/{candidate_id}/feedback
```

### Outreach

```text
POST /jobs/{job_id}/candidates/{candidate_id}/outreach-drafts
GET /jobs/{job_id}/candidates/{candidate_id}/outreach-drafts
PATCH /outreach-drafts/{draft_id}
```

### Export

```text
GET /jobs/{job_id}/export/candidates.csv
GET /jobs/{job_id}/export/report.md
```

### Analytics

```text
GET /jobs/{job_id}/analytics
GET /jobs/{job_id}/analytics/sources
GET /jobs/{job_id}/analytics/queries
```

### Talent pools

```text
POST /talent-pools
GET /talent-pools
POST /talent-pools/{pool_id}/candidates
GET /talent-pools/{pool_id}/candidates
DELETE /talent-pools/{pool_id}/candidates/{candidate_id}
```

### Maintenance

```text
POST /maintenance/backup
POST /jobs/{job_id}/archive
DELETE /jobs/{job_id}
POST /maintenance/db-checkpoint
```

---

## 15. Frontend Requirements

### Candidate review queue

Create:

```text
/jobs/[jobId]/review
```

Candidate card:

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

### Outreach UI

On candidate detail:

```text
Generate Outreach Draft
Edit Draft
Copy Draft
Mark Used
Mark Contacted
```

### Analytics UI

Sections:

```text
Summary
Source Performance
Query Performance
Research Task Health
Candidate Funnel
```

### Export buttons

Add:

```text
Export CSV
Export Markdown Report
Open Export Folder Path
```

### Maintenance UI

Optional simple page:

```text
/settings/maintenance
```

Show:

```text
Database path
Storage path
Backup now
Last backup
DB size
Storage size
```

---

## 16. Local Production Hardening

Although there is no cloud deployment, the local app should be stable.

### Requirements

```text
Graceful worker shutdown
Task retry
Task resume
Clear error messages
No API keys exposed in frontend
No raw private data in logs unless needed
Configurable limits
Backup/restore
Local storage path validation
```

### Logging

Store logs:

```text
/data/storage/logs/
```

Log:

```text
job_id
task_id
source
stage
status
error
timestamp
```

---

## 17. Security and Privacy

Even for personal use:

```text
Do not store protected traits.
Do not infer age/gender/religion/caste/health/politics.
Do not auto-message.
Respect do-not-contact.
Track source URLs.
Track collected_at timestamps.
Allow deleting candidates.
```

---

## 18. Stage 5 Acceptance Criteria

Stage 5 is complete when:

1. Candidate review workflow works.
2. Candidate statuses update correctly.
3. Feedback events are stored.
4. Gemini outreach drafts work.
5. Outreach is blocked for do-not-contact candidates.
6. CSV export works.
7. Markdown report export works.
8. Analytics dashboard shows source/query performance.
9. Talent pools work.
10. Local backup works.
11. Local cleanup/archive works.
12. The app can be used weekly for new JDs without resetting state.
13. All services run locally through Docker Compose or local scripts.
14. No cloud deployment is required.

---

## 19. Non-Goals

Do not implement:

```text
automatic bulk email sending
automatic LinkedIn messaging
proxy-based scraping
CAPTCHA bypassing
cloud deployment
billing
multi-user RBAC
ATS marketplace integration
```

---

## 20. Instructions for Coding Agent

1. Keep everything local-first.
2. Do not add cloud dependencies.
3. Do not auto-send outreach.
4. Use evidence-only outreach.
5. Make export reliable.
6. Make backup simple.
7. Keep analytics table-first, not chart-heavy.
8. Use Gemini through AIService only.
9. Preserve all source artifacts.
10. Keep weekly repeated use in mind.

---

## 21. Final Expected Product

The final local workstation supports:

```text
JD upload
Gemini parsing
Research plan generation
Long-running local sourcing jobs
GitHub automation
Public web discovery
ML/data sources
Designer sources
LinkedIn manual-assisted workflow
Candidate normalization
Evidence extraction
Scoring
Deduplication
Review
Outreach drafts
Export
Analytics
Backup
```

This completes the first full local version of the sourcing tool.

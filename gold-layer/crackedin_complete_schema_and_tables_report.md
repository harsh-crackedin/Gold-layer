# CrackedIn Gold Layer — Complete Schema and Table Report

**Document type:** Schema / table report  
**Scope:** Existing Postgres architecture + proposed full gold layer  
**Project:** CrackedIn / InterviewPrep.ai  
**Purpose:** Explain every important existing table and every proposed derived gold/product table, including each table's role, data grain, relationship, and example usage.

---

## 1. Executive Summary

CrackedIn's data architecture should be understood as a medallion-style pipeline:

| Layer | Purpose | Main Schemas / Tables | Source of Truth? | Product-facing? |
|---|---|---|---|---|
| Bronze | Raw ingest tracking and lifecycle state | `ingest.ingest_item`, `ingest.source_cursor`, S3 raw/extracted payloads | Yes for raw pipeline state | No |
| Silver | Normalized, queryable interview corpus storing reported facts verbatim | `silver.interview`, `silver.round`, `silver.assessment_item_occurrence`, `silver.signal`, `silver.chunk`, `silver.evidence_span` | Yes for extracted interview facts | Partially, but not ideal for direct product APIs |
| Existing App/User Data | User identity, chats, memory, LeetCode sync, prep plans, progress | `public`, `catalog`, `user_data`, `code_blobs`, `system`, `app` | Yes for app/user state | Yes for user features |
| Gold | Derived, canonicalized, aggregated, product-optimized intelligence | proposed `gold.*` tables | No for raw facts; yes for derived product intelligence | Yes |
| Personalized Product Layer | User-specific recommendations and readiness derived from gold + user activity | proposed/extended `app.*` tables | Yes for user-facing state | Yes |

The gold layer should not replace silver. Silver remains the canonical extracted corpus. Gold converts silver into product-ready intelligence: canonical questions, mapped platform references, role/level cohorts, frequency/trending, topic trends, interview loop profiles, evidence cards, failure intelligence, search documents, comparison views, and refresh metadata.

---

## 2. Key Design Rules

1. **Silver stores reported occurrences verbatim.** It should not own canonical question IDs, final LeetCode/Codeforces mappings, company-role trend scores, or product-facing frequency.
2. **Gold owns canonicalization and aggregation.** It resolves raw prompts into canonical questions, groups variants into families, maps platform references, computes trends, and prepares fast product queries.
3. **Gold tables should have normal names.** Use `gold.questions`, not `gold_layer_questions`; the schema name already tells us the layer.
4. **Trend means recent, not lifetime.** Old questions can contribute to lifetime support, but product-facing trends should use explicit windows such as 90, 180, 365, and 540 days.
5. **Frequency counts distinct interviews, not repeated mentions.** One interview mentioning the same question multiple times should not inflate public frequency.
6. **Every product insight should be explainable.** Store enough occurrence, evidence, sample prompt, confidence, and fallback data to explain why a question or topic is recommended.
7. **Personalization should consume gold, not pollute it.** User-specific readiness and recommendation state can live in `app.*`, while global company/role intelligence lives in `gold.*`.

---

# Part A — Existing Database Tables

This section covers the important tables already present or represented in the current architecture. The project is transitional, but the target direction is Postgres-only. The older production migration contains app/user tables, while the newer ingest/silver schema contains the bronze/silver interview corpus pipeline.

---

## 3. Existing Core Identity and User Tables

### 3.1 `public.users`

| Property | Details |
|---|---|
| Layer | Existing app/user state |
| Grain | One row per CrackedIn user |
| Main purpose | Stores user identity, email, display name, auth provider, tier, and migration bridge ID |
| Used by | Chat, memory, prep plans, LeetCode sync, progress, recommendation, user-specific readiness |

**What it stores**

- User UUID
- Email
- Display name
- Auth provider
- Tier such as free/premium
- Legacy `sqlite_id` for migration compatibility
- Creation/update timestamps

**Example**

A user signs up with email `harsh@example.com`. A row is created in `public.users`. All user-owned records such as LeetCode submissions, prep plans, chat sessions, and future readiness snapshots reference this user ID.

---

## 4. Existing Coding Catalog and User Coding Tables

### 4.1 `catalog.coding_problems`

| Property | Details |
|---|---|
| Layer | Existing catalog |
| Grain | One coding problem per platform and slug |
| Main purpose | Shared catalog of coding problems from platforms such as LeetCode |
| Used by | User coding submissions, recommendation engine, gold question references |

**What it stores**

- Platform name, such as `leetcode`
- Problem slug
- Display ID
- Title
- Difficulty
- Topic tags
- Acceptance rate
- Statement, constraints, examples, hints, similar slugs
- Raw metadata

**Example**

For LeetCode 200, the table stores platform `leetcode`, slug `number-of-islands`, title `Number of Islands`, difficulty `Medium`, and topics such as `graph`, `dfs`, `bfs`, `matrix`.

**Gold relationship**

`gold.question_references` should point to this kind of platform data when a canonical gold question maps to LeetCode, Codeforces, HackerRank, or another source.

---

### 4.2 `user_data.user_coding_profiles`

| Property | Details |
|---|---|
| Layer | Existing user coding data |
| Grain | One row per user per coding platform |
| Main purpose | Tracks a connected LeetCode/Codeforces/etc. profile |
| Used by | Extension sync, user readiness, personalized recommendations |

**What it stores**

- User ID
- Platform
- External user ID / handle
- Profile URL
- Premium status
- Sync status
- Last full sync, delta sync, and live capture timestamps
- Raw profile metadata

**Example**

A user connects their LeetCode account. This table stores the external handle, profile URL, connection status, and last sync timestamps.

---

### 4.3 `user_data.user_coding_problems`

| Property | Details |
|---|---|
| Layer | Existing user coding data |
| Grain | One row per user + platform + problem |
| Main purpose | Aggregated user performance per coding problem |
| Used by | Weakness detection, readiness scoring, personalized prep |

**What it stores**

- User ID
- Platform and problem slug
- Status such as attempted/accepted
- First attempt timestamp
- First accepted timestamp
- Latest attempt timestamp
- Attempt count
- Accepted count
- Wrong answer, TLE, MLE, runtime error, compile error counts
- Best runtime, memory, and language

**Example**

The user attempted `number-of-islands` five times and solved it twice. This table keeps the aggregate problem-level state.

**Gold relationship**

Gold says `Number of Islands` is trending for Google Mid Backend. This table says whether the user has solved it recently and how well. Combining both powers personalized recommendations.

---

### 4.4 `user_data.user_coding_submissions`

| Property | Details |
|---|---|
| Layer | Existing user coding data |
| Grain | One row per captured coding submission |
| Main purpose | Stores detailed submission history from platforms like LeetCode |
| Used by | User progress, error analysis, weakness detection, recommendation events |

**What it stores**

- Platform and submission ID
- User ID and profile ID
- Problem slug
- Verdict
- Language
- Runtime and memory stats
- Testcase counts
- Error messages
- Timestamp
- Capture source
- Whether code was captured
- Raw metadata

**Example**

The user submits `number-of-islands` in JavaScript and gets TLE. The table stores the verdict, timestamp, runtime, and language.

---

### 4.5 `code_blobs.user_coding_submission_code`

| Property | Details |
|---|---|
| Layer | Existing code storage |
| Grain | One row per captured code blob |
| Main purpose | Stores or references the code submitted by a user |
| Used by | Code review, weakness analysis, future AI feedback |

**What it stores**

- Platform and submission ID
- Code text
- Code language
- Code hash
- Storage tier such as hot/archive
- Optional S3 key
- Fetch timestamp

**Example**

For a failed LeetCode submission, the actual code is stored here. Later, the tutor can analyze why it failed.

---

## 5. Existing Sync and Operational Tables

### 5.1 `system.sync_state`

| Property | Details |
|---|---|
| Layer | Existing sync infrastructure |
| Grain | One row per user and platform |
| Main purpose | Tracks sync cursor and error state for platform integrations |
| Used by | Browser extension, historical sync, delta sync |

**What it stores**

- User ID
- Platform
- Current sync phase
- Cursor JSON
- Last progress timestamp
- Retry count
- Last error

**Example**

A user has synced 600 of 900 LeetCode submissions. This table stores the cursor so syncing can resume after a tab/browser/service-worker interruption.

---

## 6. Existing Chat, Memory, and Prep Tables

### 6.1 `app.chat_sessions`

| Property | Details |
|---|---|
| Layer | Existing app state |
| Grain | One row per chat session |
| Main purpose | Stores user chat sessions and optional attached prep plan |
| Used by | AI assistant, tutor, prep mode |

**Example**

A user starts a chat called “Google SDE2 prep.” The session may be attached to a prep plan.

---

### 6.2 `app.chat_messages`

| Property | Details |
|---|---|
| Layer | Existing app state |
| Grain | One row per chat message |
| Main purpose | Stores user and assistant messages |
| Used by | Chat history, memory extraction, feedback, future retrieval |

**Example**

The user asks: “What are the top Google graph questions?” The message is stored here, possibly with trace IDs and token usage.

---

### 6.3 `app.chat_feedback`

| Property | Details |
|---|---|
| Layer | Existing app feedback |
| Grain | One row per user rating on a message |
| Main purpose | Stores thumbs up/down style feedback |
| Used by | Quality improvement, personalization, evaluation |

---

### 6.4 `app.user_context_events`

| Property | Details |
|---|---|
| Layer | Existing user memory/event state |
| Grain | One extracted memory event |
| Main purpose | Stores extracted user context events from chat/import/sync |
| Used by | Personalization, memory rebuilding, profile summary |

**Example**

From a conversation, the system extracts: “User is targeting Google SDE2 in two months.” That becomes a structured context event.

---

### 6.5 `app.user_memory`

| Property | Details |
|---|---|
| Layer | Existing memory state |
| Grain | One summarized memory |
| Main purpose | Stores durable user memory summaries with tags and importance |
| Used by | Personalized AI responses and tutoring |

---

### 6.6 `app.user_profile_summary`

| Property | Details |
|---|---|
| Layer | Existing user profile projection |
| Grain | One profile summary per user |
| Main purpose | Stores aggregated user profile state |
| Used by | Personalization, target tracking, readiness calculation |

**Example**

The summary says the user is a fresher, targeting backend roles, weak in graphs, and currently focused on system design.

---

### 6.7 `app.pt_user_targets`

| Property | Details |
|---|---|
| Layer | Existing prep target state |
| Grain | One target company/level/role entry per user |
| Main purpose | Tracks target company, level, and role family |
| Used by | Prep planner, future gold-personalized recommendations |

**Example**

A user sets target `Amazon`, level `l4`, role family `backend`.

---

### 6.8 `app.pt_user_topic_progress`

| Property | Details |
|---|---|
| Layer | Existing prep progress |
| Grain | One user-topic progress row |
| Main purpose | Tracks user readiness and coverage per topic |
| Used by | Tutor orchestrator, revision scheduling, readiness scoring |

**Example**

The user has `coverage_score=0.4` for graphs and `readiness_score=0.25`. If gold says graphs are heavily trending for the user's target company, this becomes a high-priority gap.

---

### 6.9 `app.pt_checkpoint_attempts`

| Property | Details |
|---|---|
| Layer | Existing checkpoint/evaluation state |
| Grain | One checkpoint attempt |
| Main purpose | Stores quiz/checkpoint attempts and evaluator feedback |
| Used by | Tutor readiness and topic mastery |

---

### 6.10 `app.pt_learning_events`

| Property | Details |
|---|---|
| Layer | Existing learning event log |
| Grain | One user learning/progress event |
| Main purpose | Stores learning progress events such as taught, covered, passed, failed, revised, struggled |
| Used by | Progress projections and readiness |

---

### 6.11 `app.user_prep_plans`

| Property | Details |
|---|---|
| Layer | Existing prep plan state |
| Grain | One prep plan assigned to a user |
| Main purpose | Tracks a user-specific prep plan from a template |
| Used by | Planner, task progress, readiness snapshots |

---

### 6.12 `app.user_prep_plan_feedback`

| Property | Details |
|---|---|
| Layer | Existing feedback state |
| Grain | One feedback row per user prep plan |
| Main purpose | Stores feedback on prep plan quality |

---

### 6.13 `app.user_prep_task_progress`

| Property | Details |
|---|---|
| Layer | Existing prep task progress |
| Grain | One task progress row per user prep plan day/task |
| Main purpose | Tracks completion of plan tasks |

---

### 6.14 `app.user_readiness_snapshots`

| Property | Details |
|---|---|
| Layer | Existing readiness projection |
| Grain | One readiness snapshot per user prep plan and timestamp |
| Main purpose | Stores overall readiness score and component breakdown |
| Future relationship | Should eventually consume gold trend demand signals |

---

### 6.15 `app.user_contest_history`

| Property | Details |
|---|---|
| Layer | Existing coding profile enrichment |
| Grain | One contest record per user/platform/contest |
| Main purpose | Stores coding contest history |

---

### 6.16 `app.user_read_posts`

| Property | Details |
|---|---|
| Layer | Existing user reading state |
| Grain | One row per user-read interview post |
| Main purpose | Tracks which interview posts a user has already read |

---

### 6.17 `app.uc_forget_audit`

| Property | Details |
|---|---|
| Layer | Existing privacy/audit state |
| Grain | One forget/delete memory command |
| Main purpose | Audits user context deletion commands |

---

### 6.18 `app.api_logs`

| Property | Details |
|---|---|
| Layer | Existing operational logging |
| Grain | One API event/log row |
| Main purpose | Stores API event metadata, duration, and related user |

---

## 7. Existing Bronze Ingest Tables

### 7.1 `ingest.ingest_item`

| Property | Details |
|---|---|
| Layer | Bronze |
| Grain | One raw item discovered from a source |
| Main purpose | Tracks lifecycle of raw interview-source items across discover, hydrate, extract, reject, fail stages |
| Used by | Ingest orchestration, retries, idempotent pipeline, bronze-to-silver flow |

**What it stores**

- Source and source-specific ID
- Source URL
- Current lifecycle stage
- S3 raw and extracted artifact keys
- Content hash
- Adapter and extractor versions
- Query key
- Retry and lock state
- Pre-gate and extraction status

**Example**

A LeetCode discussion post is discovered. The row starts as `discovered`, moves to `raw_stored` after raw content is saved to S3, then becomes `extracted` if the LLM accepts it as an interview experience.

**Why it matters**

This table makes ingestion resumable and safe. It does not store interview content; it stores pipeline state and S3 pointers.

---

### 7.2 `ingest.source_cursor`

| Property | Details |
|---|---|
| Layer | Bronze |
| Grain | One source/query/version/mode cursor |
| Main purpose | Tracks crawl resume position |
| Used by | Incremental crawls, backfills, idempotent discovery |

**Example**

For a Glassdoor crawl query, this table stores the last page or cursor so the next job resumes from the correct point.

---

## 8. Existing Silver Reference and Dimension Tables

### 8.1 `silver.company`

| Property | Details |
|---|---|
| Layer | Silver dimension |
| Grain | One canonical company |
| Main purpose | Canonical company dimension with aliases |
| Used by | Silver interviews, gold company-role cohorts, company pages |

**What it stores**

- Canonical company name
- Glassdoor company ID
- Aliases
- Country and sector

**Example**

`Amazon`, `AWS`, and regional variants can map to a canonical Amazon company entry.

---

### 8.2 `silver.canonical_ladder`

| Property | Details |
|---|---|
| Layer | Silver dimension |
| Grain | One canonical seniority rank |
| Main purpose | Neutral cross-company seniority ladder |
| Used by | Role/level resolution and gold cohorting |

**Example ranks**

| Rank | Label | Typical YOE |
|---|---|---|
| 3 | Entry | 0-2 |
| 4 | Mid | 2-5 |
| 5 | Senior | 5-9 |
| 6 | Staff | 9-13 |
| 7 | Senior Staff | 12-16 |
| 8 | Principal | 15+ |
| 9 | Director+ | 18+ |

---

### 8.3 `silver.company_level`

| Property | Details |
|---|---|
| Layer | Silver/transitional dimension |
| Grain | One native company level mapping |
| Main purpose | Maps native company labels such as `SDE-2`, `L5`, `E4`, `IC4` to canonical ranks |
| Future | Gold should own the authoritative product mapping via `gold.role_levels` |

**Example**

`Amazon SDE-2` maps to a canonical rank such as Mid/Senior depending on the verified company-level mapping.

---

### 8.4 `silver.level_resolution`

| Property | Details |
|---|---|
| Layer | Silver/transitional cache |
| Grain | One resolved raw company/role/level tuple |
| Main purpose | Caches level resolution decisions so silver loading remains deterministic |
| Future | Gold should own final product mapping and review workflow |

**Example**

Raw level string `SDE II` for Amazon backend is resolved once, cached, and reused during silver loading.

---

## 9. Existing Silver Corpus Tables

### 9.1 `silver.interview`

| Property | Details |
|---|---|
| Layer | Silver corpus |
| Grain | One interview experience |
| Main purpose | Stores normalized interview-level facts extracted from raw posts |
| Used by | Gold role resolution, cohorting, profile aggregation, trend windows |

**What it stores**

- Source, source ID, source URL
- S3 raw/extracted keys
- Content hash and duplicate hooks
- Posted date and interview date
- Company raw and company ID
- Role raw, role family, role specialties, level raw
- Canonical level candidate and company level candidate
- Employment type
- Candidate YOE and profile JSON
- Location and country
- Outcome, difficulty, sentiment
- Number of rounds and process duration
- Application source
- Compensation JSON
- Takeaways
- Metadata
- Quality fields such as `llm_rejected`, `richness_score`, `extract_confidence`

**Example**

A candidate reports a Google backend interview from June 2026. The table stores company, role family, level, date, outcome, difficulty, number of rounds, and high-level takeaways.

**Gold relationship**

`gold.interview_roles` derives one clean cohort row from this table. `gold.interview_profiles` aggregates many interviews into expected loop profiles.

---

### 9.2 `silver.round`

| Property | Details |
|---|---|
| Layer | Silver corpus |
| Grain | One round/stage inside an interview |
| Main purpose | Stores interview loop structure |
| Used by | Interview loop prediction, round-specific question ranking, round mix analysis |

**What it stores**

- Interview ID
- Round index
- Round type
- Raw round name
- Round outcome
- Difficulty reported
- Feedback
- Occurrence date
- Metadata such as duration, platform, interviewer role, sections, tested areas, format, proctoring

**Example**

An interview has round 1 `online_assessment`, round 2 `problem_solving_dsa`, round 3 `system_design_hld`, and round 4 `behavioral`.

**Gold relationship**

`gold.interview_profiles` uses this to estimate likely interview loops. `gold.question_occurrences` stores round context for each mapped question.

---

### 9.3 `silver.assessment_item_occurrence`

| Property | Details |
|---|---|
| Layer | Silver corpus |
| Grain | One reported assessment item/question occurrence |
| Main purpose | Stores what was asked, verbatim |
| Used by | Question canonicalization, DSA mapping, design clustering, trend scoring |

**What it stores**

- Interview ID and round ID
- Item index
- Item type such as DSA, system design, LLD, SQL, behavioral, debugging, ML, product sense
- Raw prompt
- Prompt summary
- Prompt hash
- Attributes JSON containing external refs, topics, patterns, follow-ups, design specs, candidate answer, item outcome, difficulty, constraints, time given, evaluation focus

**Example**

The prompt says: “Asked Number of Islands with a follow-up to handle updates dynamically.” Silver stores the raw prompt and attributes. Gold resolves it to a canonical question family like `grid traversal / islands`, with reference to LeetCode 200 if appropriate.

---

### 9.4 `silver.signal`

| Property | Details |
|---|---|
| Layer | Silver corpus |
| Grain | One atomic countable insight |
| Main purpose | Stores extracted insights that can be counted across interviews |
| Used by | Failure intelligence, prep advice trends, red flag trends, process insights |

**Signal examples**

- `failure_reason`: candidate did not finish code
- `prep_advice`: practice edge cases
- `process_event`: recruiter ghosted after final round
- `interviewer_behavior`: interviewer gave hints
- `candidate_mistake`: weak complexity explanation
- `red_flag`: unclear hiring process

**Gold relationship**

`gold.signal_trends` aggregates these by company, role, level, round type, and time window.

---

### 9.5 `silver.chunk`

| Property | Details |
|---|---|
| Layer | Silver derived search index |
| Grain | One searchable text chunk |
| Main purpose | Stores prose chunks for semantic search |
| Used by | Hybrid retrieval, evidence search, similar interview search |

**What it stores**

- Interview, round, item, or signal linkage
- Chunk type
- Text
- Source path
- Metadata
- Embedding metadata placeholders

**Gold relationship**

`gold.search_documents` can rebuild product-ready search documents from silver chunks, questions, occurrences, and signals.

---

### 9.6 `silver.evidence_span`

| Property | Details |
|---|---|
| Layer | Silver trust/audit layer |
| Grain | One verified evidence snippet |
| Main purpose | Stores exact text evidence supporting classified fields |
| Used by | Evidence-backed question cards, trust labels, auditability |

**Example**

A raw post says “The second round was system design, they asked me to design a rate limiter.” An evidence span can support `round_type=system_design_hld` or item classification.

**Gold relationship**

Gold product cards should use evidence spans and sample prompts to explain why a question, topic, or failure reason is shown.

---

# Part B — Proposed Gold Layer Tables

The following tables create the full gold layer. They are not just a V1; they represent a complete product-facing design that supports trends, evidence, topic intelligence, interview loop prediction, personalization, readiness, semantic retrieval, and change detection.

---

## 10. Gold Schema Overview

| Table | Category | Purpose |
|---|---|---|
| `gold.role_levels` | Dimension / mapping | Product-owned role and level mapping |
| `gold.interview_roles` | Derived cohort fact | Clean company-role-level assignment per interview |
| `gold.question_families` | Canonical grouping | Groups related questions and variants |
| `gold.questions` | Canonical entity | One canonical interview question/problem concept |
| `gold.question_references` | External mapping | Maps canonical questions to LeetCode, Codeforces, HackerRank, URLs, etc. |
| `gold.question_occurrences` | Fact table | Maps each silver item occurrence to a canonical question |
| `gold.question_resolution_reviews` | Review workflow | Stores human/LLM review state for ambiguous mappings |
| `gold.question_trends` | Product aggregate | Company/role/level/window-specific question trends |
| `gold.topic_trends` | Product aggregate | Topic-level trend intelligence |
| `gold.interview_profiles` | Product aggregate | Interview loop profile by company/role/level/window |
| `gold.signal_trends` | Product aggregate | Failure reasons, prep advice, process signals, red flags |
| `gold.search_documents` | Retrieval index | Product-ready SQL + FTS + vector search documents |
| `gold.trend_changes` | Change detection | Newly emerging, rising, declining, resurfacing trends |
| `gold.company_role_comparisons` | Product aggregate | Company-vs-company or role-vs-role comparison snapshots |
| `gold.refresh_runs` | Operations | Tracks gold refresh, batch, and embedding jobs |

---

## 11. `gold.role_levels`

| Property | Details |
|---|---|
| Category | Dimension / mapping |
| Grain | One role-level mapping per company + role family + native label |
| Main purpose | Gold-owned role and level taxonomy |
| Source tables | `silver.company`, `silver.canonical_ladder`, raw `silver.interview.level_raw`, `silver.interview.role_raw` |
| Used by | `gold.interview_roles`, `gold.question_trends`, `gold.interview_profiles`, readiness scoring |

**What it stores**

- Company ID
- Role family
- Native company level label
- Native role name
- Canonical ladder rank
- IC/manager track
- Mapping source
- Confidence
- Evidence
- Review status
- Active/inactive flag

**Example**

Amazon `SDE II` for software/backend maps to canonical rank `4 Mid` or `5 Senior`, depending on the verified mapping rules. The mapping is stored once and reused for every Amazon interview.

**Why it exists**

Silver may contain raw level strings and transitional resolution cache. Gold should own the product-facing mapping because the user wants company/role/level intelligence.

---

## 12. `gold.interview_roles`

| Property | Details |
|---|---|
| Category | Derived fact / cohort assignment |
| Grain | One row per silver interview |
| Main purpose | Clean cohort row used by all trend/profile queries |
| Source tables | `silver.interview`, `silver.company`, `gold.role_levels` |
| Used by | Almost every gold aggregation |

**What it stores**

- Interview ID
- Company ID and company name
- Role family
- Role specialties
- Native raw level
- Gold role level ID
- Canonical rank
- Employment type
- Country
- Interview date and estimated-date flag
- Public eligibility
- Quality weight
- Mapping method and confidence

**Example**

A raw interview has company `AWS`, role `SDE-2 backend`, and level raw `L5`. Gold resolves it into company `Amazon`, role family `software_general` or `backend`, canonical rank `Mid/Senior`, public eligible `true`, quality weight `0.92`.

**Why it exists**

It prevents every gold query from repeatedly resolving company, level, and quality filters.

---

## 13. `gold.question_families`

| Property | Details |
|---|---|
| Category | Canonical grouping |
| Grain | One row per question family or pattern family |
| Main purpose | Groups variants and related canonical questions |
| Source tables | `gold.questions`, `silver.assessment_item_occurrence`, embeddings, review decisions |
| Used by | Variant intelligence, topic trends, prep plans, company pages |

**What it stores**

- Family key
- Family title
- Family type such as DSA pattern, system design archetype, LLD pattern, behavioral theme
- Description
- Topics
- Patterns
- Primary item type
- Support count
- Confidence

**Example**

Family: `grid traversal / islands`  
Canonical questions inside it: `Number of Islands`, `Max Area of Island`, `Shortest Bridge`, custom dynamic-island variant.

Family: `URL shortener`  
Canonical questions inside it: basic URL shortener, URL shortener with analytics, URL shortener with abuse detection, global short-link service.

**Why it exists**

Users need to understand patterns, not just memorize exact questions. It also prevents similar variants from fragmenting trend scores.

---

## 14. `gold.questions`

| Property | Details |
|---|---|
| Category | Canonical entity |
| Grain | One canonical interview question/problem concept |
| Main purpose | Product-facing normalized question catalog |
| Source tables | `silver.assessment_item_occurrence`, `catalog.coding_problems`, external refs, resolver output |
| Used by | Question pages, question trends, topic trends, readiness, search, prep plans |

**What it stores**

- Question key such as `lc:200`, `cf:1791A`, `sd:url-shortener`, `lld:parking-lot`
- Item type
- Canonical title and slug
- Summary
- Family ID
- Topics
- Patterns
- Difficulty
- Custom/internal flag
- Canonicalization version
- Lifetime support count
- Created-from occurrence

**Example**

A silver prompt says “Asked LC 200 Number of Islands with BFS.” Gold creates or links to `gold.questions` row `lc:200`, title `Number of Islands`, item type `dsa_coding`, family `grid traversal / islands`, topics `graph`, `bfs`, `dfs`, `matrix`.

**Why it exists**

This is the central canonical object that all product surfaces use.

---

## 15. `gold.question_references`

| Property | Details |
|---|---|
| Category | External mapping |
| Grain | One external reference per canonical question |
| Main purpose | Maps gold questions to LeetCode, Codeforces, HackerRank, company-named problems, URLs, etc. |
| Source tables | `silver.assessment_item_occurrence.attributes.external_refs`, `catalog.coding_problems`, platform crawlers |
| Used by | Question cards, practice links, DSA mapping, source transparency |

**What it stores**

- Question ID
- Platform
- External ID
- External slug
- External URL
- Reference type such as same-as, variant-of, mentioned, named-problem
- Confidence

**Example**

`gold.questions` row `Number of Islands` has a reference to LeetCode external ID `200`, slug `number-of-islands`, platform `leetcode`.

**Why it exists**

One canonical question can map to multiple external references or variants. Keeping references separate avoids overloading `gold.questions`.

---

## 16. `gold.question_occurrences`

| Property | Details |
|---|---|
| Category | Fact table |
| Grain | One row per silver assessment item occurrence |
| Main purpose | Auditable map from reported silver item to canonical gold question |
| Source tables | `silver.assessment_item_occurrence`, `silver.interview`, `silver.round`, `gold.questions`, `gold.interview_roles` |
| Used by | Trend computation, evidence cards, debugging, resolver review |

**What it stores**

- Silver occurrence ID
- Question ID
- Interview ID and round ID
- Item type
- Prompt hash
- Raw prompt and summary snapshot
- Company, role family, canonical rank
- Interview date
- Round type and round index
- Mapping method
- Mapping score
- Routing status: auto, review, defer
- Variant flag
- Public eligibility
- Resolved timestamp

**Example**

A raw item says: “Design a parking lot with multiple vehicle types.” Gold maps it to question `lld:parking-lot`, occurrence method `embedding+rerank`, routing `auto`, round type `machine_coding_lld`, company `Flipkart`, role family `backend`.

**Why it exists**

This is the auditable source for frequency. If a user asks why a question is trending, the product can trace back to supporting occurrences.

---

## 17. `gold.question_resolution_reviews`

| Property | Details |
|---|---|
| Category | Human-in-the-loop workflow |
| Grain | One unresolved or ambiguous mapping decision |
| Main purpose | Stores review tasks for uncertain canonicalization |
| Source tables | `gold.question_occurrences`, `silver.assessment_item_occurrence`, embeddings, reranker/LLM results |
| Used by | Quality control, active learning, canonicalization improvement |

**What it stores**

- Occurrence ID
- Candidate question IDs
- Candidate scores
- Suggested route
- Review status
- Reviewer decision
- Reviewer notes
- Final question ID
- Timestamps

**Example**

A prompt says “similar to Word Ladder but with weighted transformations.” The system is unsure whether it should map to `Word Ladder`, `Shortest Path`, or a custom variant. It creates a review row.

**Why it exists**

Gold should not silently overmerge or misclassify questions. Ambiguous matches need a safe workflow.

---

## 18. `gold.question_trends`

| Property | Details |
|---|---|
| Category | Product aggregate |
| Grain | One question trend per scope + item type + question + time window |
| Main purpose | Fast product-facing trending/relevance table |
| Source tables | `gold.question_occurrences`, `gold.questions`, `gold.interview_roles`, `silver.round` |
| Used by | Company pages, role pages, question rankings, prep recommendations |

**What it stores**

- Scope type such as exact company-role-level, company-role-any-level, company-any-role, global-role, global-family, global-all
- Company, role family, canonical rank
- Item type
- Question ID
- Window days
- Distinct interview count
- Raw occurrence count
- Weighted frequency
- Interview relevance
- Trend score
- Confidence label
- Fallback used flag
- First/last seen date
- Round type counts
- Topic counts
- Common follow-ups
- Common failure modes
- Observed difficulty
- Sample interview IDs and occurrence IDs
- Sample prompt summaries
- Refreshed timestamp

**Example**

For `Google + Backend + Mid + dsa_coding + 180 days`, the table ranks `Number of Islands` with distinct interviews `8`, weighted frequency `6.42`, trend score `9.81`, last seen `2026-06-19`, round types `phone screen` and `online assessment`.

**Why it exists**

This is the core product table for “what is being asked recently.”

---

## 19. `gold.topic_trends`

| Property | Details |
|---|---|
| Category | Product aggregate |
| Grain | One topic trend per scope + topic + time window |
| Main purpose | Shows which topics are rising, stable, or declining by company/role/level |
| Source tables | `gold.question_occurrences`, `gold.questions`, `silver.assessment_item_occurrence.attributes`, `gold.interview_roles` |
| Used by | Prep prioritization, readiness gap analysis, topic pages, planner |

**What it stores**

- Scope type
- Company, role family, canonical rank
- Item type
- Topic slug/name
- Window days
- Distinct interviews
- Question count
- Weighted frequency
- Trend score
- Trend direction
- Confidence label
- Top supporting questions
- Last seen date

**Example**

For `Amazon SDE2 Backend`, topics show `Graphs` rising, `HashMap` stable, `Dynamic Programming` moderate, `Leadership Principles` highly frequent for behavioral.

**Why it exists**

Users often need to know what to study at topic level before choosing exact questions.

---

## 20. `gold.interview_profiles`

| Property | Details |
|---|---|
| Category | Product aggregate |
| Grain | One interview profile per scope + time window |
| Main purpose | Predicts interview loop structure and evaluation mix |
| Source tables | `gold.interview_roles`, `silver.round`, `silver.assessment_item_occurrence`, `silver.signal` |
| Used by | Company guides, prep plan generation, mock interview setup |

**What it stores**

- Scope type
- Company, role family, canonical rank
- Window days
- Total interviews
- Total rounds
- Total questions
- Average round count
- Median process duration
- Round mix
- Ordered round pattern
- Item type mix
- Signal mix
- Top topics
- Top round warnings
- First/last seen date
- Confidence label

**Example**

For `Amazon Mid Backend`, the profile says: DSA appears in 82% of interviews, behavioral appears in 71%, system design appears in 46%, and common loop is recruiter → OA → DSA → LLD/system design → behavioral.

**Why it exists**

Users do not only prepare questions; they prepare for a full interview loop.

---

## 21. `gold.signal_trends`

| Property | Details |
|---|---|
| Category | Product aggregate |
| Grain | One signal trend per scope + signal type/subtype + time window |
| Main purpose | Aggregates failure reasons, prep advice, red flags, process events, interviewer behavior |
| Source tables | `silver.signal`, `gold.interview_roles`, `silver.round`, `silver.assessment_item_occurrence` |
| Used by | Failure intelligence, prep advice, company process insights |

**What it stores**

- Scope type
- Company, role family, canonical rank
- Round type
- Signal type and subtype
- Window days
- Distinct interviews
- Weighted frequency
- Trend score
- Sample signal texts
- Confidence label

**Example**

For `Meta phone screen`, common failure reasons include “did not finish code,” “poor time management,” and “weak edge-case handling.”

**Why it exists**

This answers why candidates fail or succeed, which is often more useful than a question list.

---

## 22. `gold.search_documents`

| Property | Details |
|---|---|
| Category | Retrieval index |
| Grain | One product-searchable document chunk |
| Main purpose | Supports SQL + full-text + vector retrieval |
| Source tables | `gold.questions`, `gold.question_occurrences`, `silver.chunk`, `silver.signal`, `silver.evidence_span`, `silver.round` |
| Used by | AI chat, similar experience search, semantic evidence retrieval, hybrid search |

**What it stores**

- Source type such as question, occurrence, round, signal, chunk, evidence
- Source IDs
- Question, interview, round, occurrence, signal references
- Company, role family, canonical rank, item type
- Search text
- Full-text search vector
- Embedding model/version
- Embedding vector
- Metadata

**Example**

A search document might contain: “Candidate failed Meta phone screen because they explained Longest Duplicate Substring but did not finish coding.” It links to the interview, signal, round, and company-role cohort.

**Why it exists**

Not every user query can be answered by SQL. This table supports semantic retrieval while still preserving structured filters.

---

## 23. `gold.trend_changes`

| Property | Details |
|---|---|
| Category | Change detection aggregate |
| Grain | One detected change per scope + entity + window comparison |
| Main purpose | Tracks newly emerging, rising, declining, and resurfacing questions/topics/signals |
| Source tables | `gold.question_trends`, `gold.topic_trends`, `gold.signal_trends` |
| Used by | Watchlists, alerts, “what changed recently,” marketing pages |

**What it stores**

- Entity type such as question, topic, signal
- Entity ID or key
- Scope
- Current window and previous window
- Current score
- Previous score
- Delta
- Change type: new, rising, stable, declining, resurfacing
- Confidence label
- Supporting examples

**Example**

`Agentic coding debugging` becomes a rising topic for startup fullstack interviews in the last 90 days compared with the previous 90 days.

**Why it exists**

This is the freshness moat. Static prep sheets cannot tell users what changed this month.

---

## 24. `gold.company_role_comparisons`

| Property | Details |
|---|---|
| Category | Product aggregate |
| Grain | One comparison snapshot between two companies/roles/levels |
| Main purpose | Powers comparison pages and recommendations |
| Source tables | `gold.question_trends`, `gold.topic_trends`, `gold.interview_profiles`, `gold.signal_trends` |
| Used by | Company comparison pages, user targeting, market fit analysis |

**What it stores**

- Left company/role/level scope
- Right company/role/level scope
- Window days
- Differences in topic mix
- Differences in round mix
- Differences in question trends
- Differences in failure reasons
- Summary text
- Confidence label

**Example**

Comparison: Google Mid Backend vs Amazon Mid Backend. Google is more graph/system-design heavy; Amazon is more behavioral/LP-heavy and OA-heavy.

**Why it exists**

Users often choose where to apply or how to shift preparation between companies.

---

## 25. `gold.refresh_runs`

| Property | Details |
|---|---|
| Category | Operations |
| Grain | One gold refresh or processing job run |
| Main purpose | Tracks gold jobs, batch refreshes, failures, and metrics |
| Source tables | All gold jobs |
| Used by | Observability, debugging, audit, pipeline health |

**What it stores**

- Job name
- Start/end timestamps
- Status
- Source date range
- Source max updated timestamp
- Rows inserted/updated/skipped
- Error
- Metadata

**Example**

A nightly job refreshes question trends for all company-role cohorts. The run logs 1,200 updated rows and 8 skipped ambiguous mappings.

---

# Part C — Proposed Downstream Product Tables That Consume Gold

These are not pure gold tables because they are user-specific. They should live in `app.*` or another user-owned schema. They are included because a foolproof gold design should show how gold becomes actual user value.

---

## 26. `app.user_target_readiness_snapshots`

| Property | Details |
|---|---|
| Category | User-specific product projection |
| Grain | One readiness snapshot per user + target + timestamp |
| Main purpose | Measures readiness against a target company/role/level using gold demand signals and user progress |
| Source tables | `gold.question_trends`, `gold.topic_trends`, `gold.interview_profiles`, `user_data.user_coding_problems`, `app.pt_user_topic_progress` |

**Example**

A user targeting Google Mid Backend gets readiness score 64%. Breakdown: DSA 72%, System Design 38%, Behavioral 55%, SQL/DB 80%.

---

## 27. `app.user_gold_recommendations`

| Property | Details |
|---|---|
| Category | User-specific recommendations |
| Grain | One personalized recommendation per user |
| Main purpose | Converts gold market demand + user weakness into actionable tasks |
| Source tables | Gold trends, user coding history, prep progress, readiness snapshots |

**Example**

Recommendation: “Practice grid BFS/DFS because Google Mid Backend has high recent graph frequency and your recent graph attempt success is low.”

---

## 28. `app.company_watchlists`

| Property | Details |
|---|---|
| Category | User-specific retention feature |
| Grain | One watchlist target per user |
| Main purpose | Lets users follow companies/roles for trend changes |
| Source tables | `gold.trend_changes`, `gold.question_trends`, `gold.topic_trends` |

**Example**

The user watches Meta E4 Backend. When new DSA/system-design patterns emerge, the app can show an alert.

---

## 29. `app.watchlist_notifications`

| Property | Details |
|---|---|
| Category | User notification state |
| Grain | One notification generated from a watchlist signal |
| Main purpose | Stores notification delivery state |
| Source tables | `app.company_watchlists`, `gold.trend_changes` |

**Example**

“Newly emerging at Meta E4: graph string hybrid problem variants increased in the last 90 days.”

---

# Part D — Table Relationship Map

## 30. Source-to-Gold Flow

| Step | Input Tables | Output Tables | Purpose |
|---|---|---|---|
| Raw discovery | External sources | `ingest.ingest_item`, S3 raw | Track source item lifecycle |
| Extraction | S3 raw, ingest state | `silver.interview`, `silver.round`, `silver.assessment_item_occurrence`, `silver.signal`, `silver.chunk`, `silver.evidence_span` | Normalize interview facts verbatim |
| Role resolution | `silver.interview`, `silver.company`, ladder data | `gold.role_levels`, `gold.interview_roles` | Build clean product cohorts |
| Question canonicalization | `silver.assessment_item_occurrence`, references, catalog, embeddings | `gold.questions`, `gold.question_families`, `gold.question_references`, `gold.question_occurrences` | Resolve raw prompts into canonical product questions |
| Aggregation | Gold facts + silver rounds/signals | `gold.question_trends`, `gold.topic_trends`, `gold.interview_profiles`, `gold.signal_trends` | Serve fast product intelligence |
| Search | Gold + silver chunks/signals/evidence | `gold.search_documents` | Enable hybrid semantic retrieval |
| Change detection | Trend tables | `gold.trend_changes` | Identify new/rising/declining trends |
| Personalization | Gold + user progress | `app.user_target_readiness_snapshots`, `app.user_gold_recommendations` | Turn market intelligence into user-specific action |

---

## 31. Core Query Examples

### Example 1 — Trending DSA questions for Google Mid Backend

1. User asks for Google Mid Backend DSA questions.
2. API normalizes Google to `company_id` and Mid to canonical rank.
3. Query `gold.question_trends` with exact scope, item type `dsa_coding`, window `180 days`.
4. Join `gold.questions` and `gold.question_references` for title and LeetCode/Codeforces links.
5. Return confidence, trend score, last seen date, round mix, and sample prompts.

### Example 2 — Expected interview loop for Amazon SDE2

1. Query `gold.interview_profiles` for Amazon + backend/software + canonical rank.
2. Return round mix, ordered round pattern, DSA/system design/behavioral probability, top warnings, and confidence.

### Example 3 — Why this question is recommended

1. Start with a row in `gold.question_trends`.
2. Use sample occurrence IDs from `gold.question_occurrences`.
3. Fetch prompt summaries, round context, and evidence snippets.
4. Explain: frequency, recency, round type, company/role relevance, and user readiness gap.

### Example 4 — Personalized preparation gap

1. Gold says graph/grid questions are trending for Google Mid Backend.
2. `user_data.user_coding_problems` says the user has low recent success on graph problems.
3. `app.pt_user_topic_progress` says graph readiness is weak.
4. `app.user_gold_recommendations` recommends a graph practice path.

---

# Part E — Final Recommended Table Set

## 32. Required Gold Tables

These should be part of the complete gold design:

1. `gold.role_levels`
2. `gold.interview_roles`
3. `gold.question_families`
4. `gold.questions`
5. `gold.question_references`
6. `gold.question_occurrences`
7. `gold.question_resolution_reviews`
8. `gold.question_trends`
9. `gold.topic_trends`
10. `gold.interview_profiles`
11. `gold.signal_trends`
12. `gold.search_documents`
13. `gold.trend_changes`
14. `gold.company_role_comparisons`
15. `gold.refresh_runs`

## 33. Recommended Product Tables Consuming Gold

1. `app.user_target_readiness_snapshots`
2. `app.user_gold_recommendations`
3. `app.company_watchlists`
4. `app.watchlist_notifications`

---

## 34. Final Schema Principle

The gold layer should be designed around one core rule:

**Silver tells us what was reported. Gold tells the product what it means.**

That means gold should answer:

- What questions are being asked?
- Which questions are trending recently?
- Which topics are rising or declining?
- Which companies ask which kinds of rounds?
- Which variants and follow-ups matter?
- Why should a user trust this recommendation?
- What should this specific user prepare next?
- What changed since the last refresh?

If the gold schema supports those questions, it is not just a data layer. It becomes the intelligence engine for CrackedIn.

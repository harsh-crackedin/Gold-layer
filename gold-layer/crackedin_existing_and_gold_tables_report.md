# CrackedIn Postgres Tables + Proposed Gold Layer Table Report

**Purpose:** This report lists the tables currently visible in the repository's Postgres-oriented schema files and the new product-facing derived tables recommended for the Gold layer.

**How to read this:**

- **Existing tables** are source, operational, user-state, or silver-corpus tables already represented in the repo.
- **Gold tables** are proposed derived/product-facing tables. They should live in the `gold` schema, but table names should stay normal and readable.
- Gold should be rebuildable from Silver. It should not become the source of truth for raw interview content.

**Primary repo sources reviewed:**

- `ingest/ledger.sql` — Bronze ingest ledger.
- `ingest/silver.sql` — current Postgres Silver corpus.
- `docs/SILVER-DB-SCHEMA.md` — readable Silver schema reference.
- `migrations/prod_full_schema.sql` — existing production/transitional app/user/catalog schema.
- `docs/GOLD-LAYER-HANDOFF.md` — prior Gold handoff and resolver direction.

---

## 1. Layer overview

| Layer / schema | Role | Data style | Product usage |
|---|---|---|---|
| `ingest` | Bronze ingest ledger | Pipeline state + S3 pointers, no full interview content | Crawling, hydration, extraction lifecycle |
| `silver` | Normalized interview corpus | Verbatim reported interview facts and occurrences | Source of truth for interview intelligence |
| `catalog` | Coding problem catalog | Normalized platform problems | User coding history, DSA mapping seed |
| `user_data` | User coding activity | User-owned LeetCode / coding-platform sync | Personalized recommendations and progress |
| `code_blobs` | Submission code storage | User submission code | Deep analysis, debugging, future code review |
| `system` | Sync cursors | Operational state | Extension sync durability |
| `app` | App/user state | Chat, memory, prep, feedback, API logs | User-facing app workflows |
| `gold` | Derived product intelligence | Canonical questions, trends, profiles, search docs | Fast company/role-wise questions and interview insights |

---

# Part A — Existing tables

## 2. `ingest` schema: Bronze ingest ledger

The `ingest` schema tracks raw items across the ingestion pipeline. It intentionally does **not** store full interview content; full raw/extracted payloads live in object storage, while queryable facts are projected into `silver`.

### 2.1 `ingest.ingest_item`

| Aspect | Details |
|---|---|
| Role | One row per raw discovered post/review/comment item. Tracks the item through discovery, hydration, extraction, rejection, or failure. |
| Grain | One raw source item, identified by `(source, source_id)`. |
| Important fields | `source`, `source_id`, `source_url`, `stage`, `s3_raw_key`, `s3_extracted_key`, `content_hash`, `raw_adapter_version`, `extract_version`, `query_key`, retry/lock fields. |
| Used by | Ingest workers, Dagster/orchestration, reprocessing, retry, reclaim, extraction pipeline. |
| Not used for | Product search, company question trends, raw interview UI. |

**Example row**

| Field | Example |
|---|---|
| `source` | `leetcode` |
| `source_id` | `new_123456789` |
| `source_url` | `https://leetcode.com/discuss/interview-experience/...` |
| `stage` | `extracted` |
| `s3_raw_key` | `raw/leetcode/new_123456789.json` |
| `s3_extracted_key` | `extracted/v5/leetcode/new_123456789.json` |
| `content_hash` | SHA-256 hash of raw payload |

**Gold impact:** Gold never reads raw content from this table directly. Gold uses the `silver` projection after extraction has succeeded.

---

### 2.2 `ingest.source_cursor`

| Aspect | Details |
|---|---|
| Role | Tracks crawl position per source/query/mode so crawlers can resume safely. |
| Grain | One logical crawl cursor per `(source, query_key, query_version, mode)`. |
| Important fields | `last_page`, `last_cursor`, `oldest_seen_date`, `is_complete`, incremental watermark fields. |
| Used by | Discovery/backfill/incremental crawling jobs. |
| Not used for | Gold rollups, user-facing APIs. |

**Example row**

| Field | Example |
|---|---|
| `source` | `glassdoor` |
| `query_key` | `google_sde` |
| `mode` | `backfill` |
| `last_page` | `12` |
| `is_complete` | `false` |

---

## 3. `silver` schema: normalized interview corpus

Silver is the source of truth for interview intelligence. It stores reported occurrences verbatim. Gold is responsible for canonicalization, role/level finalization, question mapping, aggregation, and trend scoring.

### 3.1 `silver.company`

| Aspect | Details |
|---|---|
| Role | Canonical company dimension used by Silver and Gold. |
| Grain | One canonical company. |
| Important fields | `canonical_name`, `glassdoor_company_id`, `aliases`, `country`, `sector`. |
| Used by | Company filters, company normalization, company-wise question trends. |

**Example row**

| Field | Example |
|---|---|
| `canonical_name` | `Amazon` |
| `aliases` | `{Amazon, AWS, Amazon Web Services}` |
| `country` | `US` |
| `sector` | `Big Tech / Cloud` |

**Gold usage:** `gold.interview_roles` and `gold.question_trends` should reference `silver.company.id` rather than storing only free-text company names.

---

### 3.2 `silver.canonical_ladder`

| Aspect | Details |
|---|---|
| Role | Neutral cross-company seniority ladder. |
| Grain | One canonical seniority rank. |
| Existing seeded ranks | `3 Entry`, `4 Mid`, `5 Senior`, `6 Staff`, `7 Senior Staff`, `8 Principal`, `9 Director+`. |
| Used by | Role/level mapping, cross-company pooling, fallback logic. |

**Example row**

| Field | Example |
|---|---|
| `rank` | `4` |
| `label` | `Mid` |
| `typical_yoe` | `2-5` |

**Gold usage:** Gold should store final role mapping to this ladder through `gold.role_levels` and `gold.interview_roles`.

---

### 3.3 `silver.company_level`

| Aspect | Details |
|---|---|
| Role | Native company level labels mapped to canonical ranks. |
| Grain | One native level label per company. |
| Important fields | `company_id`, `native_label`, `native_name`, `canonical_rank`, `cross_company_approx`, `note`. |
| Used by | Current level resolution, deterministic lookup for known company levels. |

**Example row**

| Field | Example |
|---|---|
| `company_id` | Amazon company id |
| `native_label` | `SDE-2` |
| `native_name` | `Software Development Engineer II` |
| `canonical_rank` | `4` |

**Gold decision:** This table can remain a seed/input. Gold should own the product-facing mapping in `gold.role_levels`, where confidence, review state, and active status can be managed cleanly.

---

### 3.4 `silver.level_resolution`

| Aspect | Details |
|---|---|
| Role | Cache for resolving raw `(company, role_family, level_raw)` tuples into native company levels and canonical ranks. |
| Grain | One normalized raw level tuple. |
| Important fields | `company_norm`, `role_family`, `level_norm`, `resolved_company`, `native_code`, `company_level_id`, `canonical_rank`, confidence/review fields. |
| Used by | Silver loader and level resolver. |

**Example row**

| Field | Example |
|---|---|
| `company_norm` | `amazon` |
| `role_family` | `software_general` |
| `level_norm` | `sde2` |
| `native_code` | `SDE-2` |
| `canonical_rank` | `4` |
| `confidence` | `high` |

**Gold usage:** Gold should use this as a signal, not blindly as final truth. `gold.interview_roles` should store the final resolved role/level used for product queries.

---

### 3.5 `silver.interview`

| Aspect | Details |
|---|---|
| Role | One normalized interview experience. This is the core source table for company/role/outcome/date/process metadata. |
| Grain | One interview experience, not necessarily one raw post. A raw post may contain multiple experiences. |
| Important fields | `company_raw`, `company_id`, `role_raw`, `role_family`, `level_raw`, `canonical_level_id`, `company_level_id`, `candidate_yoe`, `interview_date`, `interview_date_estimated`, `outcome`, `num_rounds`, `process_duration_days`, `takeaways`, `llm_rejected`, `richness_score`, `extract_confidence`. |
| Used by | Gold cohort assignment, trends, company/role filtering, interview profiles, quality filtering. |

**Example row**

| Field | Example |
|---|---|
| `source` | `leetcode` |
| `company_raw` | `Google` |
| `company_id` | Google company id |
| `role_raw` | `SDE2 Backend` |
| `role_family` | `software_general` |
| `level_raw` | `SDE2` |
| `canonical_level_id` | `4` |
| `interview_date` | `2026-06-10` |
| `outcome` | `reject` |
| `num_rounds` | `4` |
| `llm_rejected` | `false` |

**Gold usage:** `gold.interview_roles` should derive product-ready company/role/cohort assignments from this table. Gold trend rollups should filter out rejected/non-experience rows and generally avoid estimated dates for public trending.

---

### 3.6 `silver.round`

| Aspect | Details |
|---|---|
| Role | Stores the interview loop stages. |
| Grain | One round/stage inside one interview. |
| Important fields | `interview_id`, `round_index`, `round_type`, `round_name_raw`, `occurred_on`, `round_outcome`, `difficulty_reported`, `feedback_raw`, `meta`. |
| Used by | Interview loop profiles, round mix, ordered round patterns, round-specific question trends. |

**Example row**

| Field | Example |
|---|---|
| `round_index` | `2` |
| `round_type` | `problem_solving_dsa` |
| `round_name_raw` | `Technical Phone Screen` |
| `round_outcome` | `fail` |
| `meta.duration_min` | `45` |

**Gold usage:** `gold.interview_profiles` should aggregate round counts, round order, round mix, and common process patterns.

---

### 3.7 `silver.assessment_item_occurrence`

| Aspect | Details |
|---|---|
| Role | Stores what was asked in the interview. This is the main input for canonical questions. |
| Grain | One reported item/prompt/question occurrence. |
| Important fields | `interview_id`, `round_id`, `item_index`, `item_type`, `prompt_raw`, `prompt_summary`, `prompt_hash`, `attributes`, `extract_confidence`. |
| Used by | Question canonicalization, DSA/system design/LLD/machine coding/behavioral trends. |

**Example row**

| Field | Example |
|---|---|
| `item_type` | `dsa_coding` |
| `prompt_raw` | `LC 200 Number of Islands` |
| `prompt_summary` | `Number of Islands graph traversal problem` |
| `prompt_hash` | Stable SHA-256 hash |
| `attributes.external_refs` | `[{source: leetcode, ref_raw: LC 200, ref_type: same_as}]` |
| `attributes.topics_reported` | `{graph, dfs, bfs}` |

**Gold usage:** `gold.question_occurrences` should map each row to `gold.questions`. If no match exists, the occurrence should be routed to review or deferred clustering.

---

### 3.8 `silver.signal`

| Aspect | Details |
|---|---|
| Role | Stores atomic, countable insights captured during extraction. |
| Grain | One signal/insight within an interview, round, or item. |
| Important fields | `interview_id`, `round_id`, `item_id`, `signal_type`, `signal_subtype`, `text_value`, `value`, `extract_confidence`. |
| Used by | Aggregating prep advice, red flags, failure reasons, success reasons, process events, interviewer behavior. |

**Example row**

| Field | Example |
|---|---|
| `signal_type` | `failure_reason` |
| `signal_subtype` | `time_management` |
| `text_value` | `Ran out of time on the second problem and did not finish coding.` |
| `item_id` | Linked DSA item id |

**Gold usage:** Signals should feed `gold.interview_profiles` and optionally `gold.search_documents`. They should not be collapsed into interview columns.

---

### 3.9 `silver.chunk`

| Aspect | Details |
|---|---|
| Role | Rebuildable text search index over narrative prose. |
| Grain | One searchable text chunk from an interview, round, item, or signal. |
| Important fields | `interview_id`, `round_id`, `item_id`, `signal_id`, `chunk_index`, `chunk_type`, `text`, `source_path`, `meta`, embedding metadata placeholders. |
| Used by | Semantic search, evidence retrieval, future pgvector backfill. |

**Example row**

| Field | Example |
|---|---|
| `chunk_type` | `prep_advice` |
| `text` | `Focus on graph traversal and time management for Meta phone screens.` |
| `source_path` | `signals[3]` |

**Gold usage:** Gold should not treat chunks as source of truth, but can transform them into `gold.search_documents` for hybrid SQL + vector retrieval.

---

### 3.10 `silver.evidence_span`

| Aspect | Details |
|---|---|
| Role | Stores verified exact snippets justifying classified fields. |
| Grain | One evidence snippet linked to an interview, round, item, or signal. |
| Important fields | `interview_id`, `round_id`, `item_id`, `signal_id`, `entity_type`, `field_name`, `evidence_text`, `char_start`, `char_end`, `extract_confidence`. |
| Used by | Trust, audit, explainability, admin review. |

**Example row**

| Field | Example |
|---|---|
| `entity_type` | `item` |
| `field_name` | `item_type` |
| `evidence_text` | `The interviewer asked me Number of Islands from LeetCode.` |

**Gold usage:** Can be attached to product explanations and sample evidence, especially when showing why a question is trending.

---

## 4. `catalog` schema: coding problem catalog

### 4.1 `catalog.coding_problems`

| Aspect | Details |
|---|---|
| Role | Platform problem catalog for LeetCode and future coding platforms. |
| Grain | One problem per `(platform, slug)`. |
| Important fields | `platform`, `slug`, `display_id`, `title`, `difficulty`, `topic_tags`, `ac_rate`, `statement_md`, `constraints_md`, `examples_json`, `hints`, `similar_slugs`, `sync_status`, `raw_meta`. |
| Used by | User coding sync, DSA mapping, practice recommendations. |

**Example row**

| Field | Example |
|---|---|
| `platform` | `leetcode` |
| `slug` | `number-of-islands` |
| `display_id` | `200` |
| `title` | `Number of Islands` |
| `difficulty` | `Medium` |
| `topic_tags` | `{graph, dfs, bfs, matrix}` |

**Gold usage:** Used as a seed for `gold.questions` and `gold.question_references`. It should not be the only source of truth because many interview questions are custom, design-based, or platform-independent.

---

## 5. `user_data` schema: user coding history

### 5.1 `user_data.user_coding_profiles`

| Aspect | Details |
|---|---|
| Role | Stores a connected coding-platform profile for a user. |
| Grain | One user-profile connection per `(user_id, platform)`. |
| Important fields | `user_id`, `platform`, `external_handle`, `profile_url`, sync timestamps, connection flags, `raw_profile`. |
| Used by | Extension sync, personalization, coding activity summary. |

**Example row**

| Field | Example |
|---|---|
| `user_id` | User UUID |
| `platform` | `leetcode` |
| `external_handle` | `harsh_dev` |
| `is_connected` | `true` |

---

### 5.2 `user_data.user_coding_problems`

| Aspect | Details |
|---|---|
| Role | Aggregated per-user problem status. |
| Grain | One row per user/platform/problem. |
| Important fields | `status`, attempt counts, accepted counts, error counts, best runtime/memory/language, latest attempt timestamps. |
| Used by | Personalization, weak-topic analysis, avoiding already-solved recommendations. |

**Example row**

| Field | Example |
|---|---|
| `platform` | `leetcode` |
| `slug` | `number-of-islands` |
| `status` | `solved` |
| `attempt_count` | `4` |
| `ac_count` | `1` |
| `latest_ac_at` | `2026-06-25` |

**Gold usage:** Not part of global public trends. Can be joined later for personalized ordering of `gold.question_trends`.

---

### 5.3 `user_data.user_coding_submissions`

| Aspect | Details |
|---|---|
| Role | Stores each captured coding submission. |
| Grain | One coding-platform submission. |
| Important fields | `platform`, `submission_id`, `user_id`, `slug`, `verdict`, language/runtime/memory/error fields, timestamp, capture source, code flag. |
| Used by | Submission history, personal analytics, future code review. |

**Example row**

| Field | Example |
|---|---|
| `platform` | `leetcode` |
| `submission_id` | `1234567890` |
| `slug` | `number-of-islands` |
| `verdict` | `Accepted` |
| `lang` | `python3` |
| `ts` | `2026-06-25 15:10:00+00` |

---

## 6. `code_blobs` schema

### 6.1 `code_blobs.user_coding_submission_code`

| Aspect | Details |
|---|---|
| Role | Stores hot submission code or external object-storage pointers. |
| Grain | One code blob per platform submission. |
| Important fields | `platform`, `submission_id`, `code`, `code_lang`, `code_hash`, `storage_tier`, `s3_key`, `fetched_at`. |
| Used by | Code review, debugging, future personalized learning. |

**Example row**

| Field | Example |
|---|---|
| `platform` | `leetcode` |
| `submission_id` | `1234567890` |
| `code_lang` | `python3` |
| `storage_tier` | `hot` |

---

## 7. `system` schema

### 7.1 `system.sync_state`

| Aspect | Details |
|---|---|
| Role | Cursor/checkpoint table for platform sync. |
| Grain | One sync state per user/platform. |
| Important fields | `user_id`, `platform`, `phase`, `cursor`, `last_progress_at`, `last_full_sync_at`, retry/error fields. |
| Used by | Extension sync durability and resumability. |

**Example row**

| Field | Example |
|---|---|
| `user_id` | User UUID |
| `platform` | `leetcode` |
| `phase` | `historical_backfill` |
| `cursor` | `{last_submission_id: 1234567890}` |

---

## 8. `public` schema

### 8.1 `public.users`

| Aspect | Details |
|---|---|
| Role | User profile mirror / application user record. |
| Grain | One user. |
| Important fields | `id`, `email`, `display_name`, `auth_provider`, `tier`, timestamps. |
| Used by | Auth-linked user identity, user-owned data FKs. |

**Example row**

| Field | Example |
|---|---|
| `id` | User UUID |
| `email` | `harsh@example.com` |
| `tier` | `free` |

---

## 9. `app` schema: application/user state

The `app` schema includes current and transitional application tables. Some are user-facing, some are future/deferred, and some may be pruned later depending on the final pure-Postgres architecture.

### 9.1 `app.chat_sessions`

| Role | Grain | Example |
|---|---|---|
| Stores chat sessions owned by users. | One chat session. | User opens “Google Interview Prep” chat. |

Key usage: grouping messages, attaching future plan context, session-level metadata.

---

### 9.2 `app.chat_messages`

| Role | Grain | Example |
|---|---|---|
| Stores user and assistant messages. | One message. | User asks “What should I prepare for Amazon SDE2?” |

Key usage: chat history, traceability, feedback, future memory extraction.

---

### 9.3 `app.chat_feedback`

| Role | Grain | Example |
|---|---|---|
| Stores thumbs up/down per message. | One user rating per message. | User downvotes an answer. |

Key usage: quality analytics and model/prompt iteration.

---

### 9.4 `app.user_context_events`

| Role | Grain | Example |
|---|---|---|
| Stores extracted or manual user-context events. | One context event. | “User is targeting Google L4 in 30 days.” |

Key usage: long-term personalization, memory rebuild, invalidation/audit.

---

### 9.5 `app.user_memory`

| Role | Grain | Example |
|---|---|---|
| Stores summarized user memories. | One memory. | “User prefers concise system-design explanations.” |

Key usage: personalized chat and future tutor context.

---

### 9.6 `app.user_profile_summary`

| Role | Grain | Example |
|---|---|---|
| Stores a rebuilt summary of user profile/context. | One row per user. | Current role, YOE, target company, weak domains. |

Key usage: fast personalization without scanning all events.

---

### 9.7 `app.pt_user_targets`

| Role | Grain | Example |
|---|---|---|
| Stores progress-tracker target settings. | One user target. | User targets Google L5 backend. |

Key usage: progress tracker / prep targeting.

---

### 9.8 `app.pt_user_topic_progress`

| Role | Grain | Example |
|---|---|---|
| Stores progress/readiness per user/topic. | One user-topic progress row. | User is 60% ready for Graphs. |

Key usage: learning progress, revision scheduling, readiness scoring.

---

### 9.9 `app.pt_checkpoint_attempts`

| Role | Grain | Example |
|---|---|---|
| Stores checkpoint/quiz attempts. | One checkpoint attempt. | User takes a DSA graph checkpoint and scores 70%. |

Key usage: readiness validation and weak subtopic detection.

---

### 9.10 `app.pt_learning_events`

| Role | Grain | Example |
|---|---|---|
| Event log for learning/progress actions. | One learning event. | User completed a plan task or struggled on DP. |

Key usage: rebuildable projections for progress tracker.

---

### 9.11 `app.user_prep_plans`

| Role | Grain | Example |
|---|---|---|
| Stores a user's adopted prep plan. | One user plan instance. | User starts a 30-day Google plan. |

Key usage: active prep dashboard and progress tracking.

---

### 9.12 `app.user_prep_plan_feedback`

| Role | Grain | Example |
|---|---|---|
| Stores feedback on prep plans. | One plan rating/comment. | User says plan is too DSA-heavy. |

Key usage: plan quality improvement.

---

### 9.13 `app.user_prep_task_progress`

| Role | Grain | Example |
|---|---|---|
| Stores completion status for prep plan tasks. | One task inside one user plan. | `lc:200` marked done on Day 4. |

Key usage: daily tasks, revision, streak/readiness.

---

### 9.14 `app.user_readiness_snapshots`

| Role | Grain | Example |
|---|---|---|
| Stores computed readiness scores over time. | One readiness snapshot. | User is 47% ready after week 2. |

Key usage: readiness chart and progress trend.

---

### 9.15 `app.user_contest_history`

| Role | Grain | Example |
|---|---|---|
| Stores coding contest history. | One contest result per user/platform. | User attended LeetCode Weekly Contest 450. |

Key usage: coding profile enrichment.

---

### 9.16 `app.user_read_posts`

| Role | Grain | Example |
|---|---|---|
| Tracks interview posts already read by a user. | One user-post read event. | User read a Google SDE2 experience. |

Key usage: avoid duplicate recommendations and support reading history.

---

### 9.17 `app.uc_forget_audit`

| Role | Grain | Example |
|---|---|---|
| Audit log for memory forget/delete commands. | One forget action. | User says “forget my target company.” |

Key usage: privacy, auditability, user control.

---

### 9.18 `app.api_logs`

| Role | Grain | Example |
|---|---|---|
| Stores application API events/logs. | One API event. | `gold_trends_query` completed in 120ms. |

Key usage: debugging, metrics, monitoring.

---

### 9.19 `pgmq.events` queue

| Aspect | Details |
|---|---|
| Role | Queue created through PGMQ extension for async/event processing. |
| Grain | One queued event/message. |
| Used by | Background workers, event-driven jobs, future incremental projections. |

**Gold usage:** Can support immediate/lightweight gold updates after batch design is stable.

---

# Part B — Proposed Gold layer tables

## 10. Gold design principles

Gold should be:

- **Derived:** rebuildable from Silver and catalog data.
- **Product-facing:** optimized for API/UI queries, not raw storage.
- **Canonical:** resolves raw prompts and role labels into stable product concepts.
- **Auditable:** every trend should trace back to supporting Silver interviews/items.
- **Batch-first:** public trends are refreshed in controlled batches.
- **Immediate-capable:** deterministic incremental updates can be added later without corrupting canonicalization.

---

## 11. `gold.role_levels`

| Aspect | Details |
|---|---|
| Role | Product-facing company/role/level mapping table. Gold owns this mapping. |
| Grain | One native role-level mapping per company and role family. |
| Key data stored | Company, role family, native level label, native title, canonical rank, track, confidence, review status, source/evidence. |
| Source tables | `silver.interview`, `silver.company`, `silver.canonical_ladder`, `silver.company_level`, `silver.level_resolution`. |
| Product usage | “Google L4”, “Amazon SDE2”, “Meta E5” all map into common canonical seniority buckets. |

**Example row**

| Field | Example |
|---|---|
| `company` | Amazon |
| `role_family` | `software_general` |
| `native_label` | `SDE-2` |
| `native_name` | `Software Development Engineer II` |
| `canonical_rank` | `4 / Mid` |
| `mapping_source` | `manual_seed + resolver` |
| `needs_review` | `false` |

**Why this table exists:**

Silver keeps raw role labels and may contain unresolved or approximate mappings. Product features need stable mappings. This table lets us evolve mappings without rewriting raw interview records.

---

## 12. `gold.interview_roles`

| Aspect | Details |
|---|---|
| Role | One derived cohort assignment per interview. |
| Grain | One row per `silver.interview`. |
| Key data stored | Final company, role family, canonical rank, native level, employment type, country, interview date, quality/public eligibility. |
| Source tables | `silver.interview`, `silver.company`, `gold.role_levels`. |
| Product usage | Fast filtering for “Google SDE2 backend interviews in last 6 months.” |

**Example row**

| Field | Example |
|---|---|
| `interview_id` | UUID of a Silver interview |
| `company_name` | Google |
| `role_family` | `software_general` |
| `native_level_raw` | `SDE2` |
| `canonical_rank` | `4 / Mid` |
| `interview_date` | `2026-06-10` |
| `is_public_eligible` | `true` |

**Why this table exists:**

It prevents every trend query from recalculating company/role resolution. It also centralizes quality decisions before public aggregation.

---

## 13. `gold.questions`

| Aspect | Details |
|---|---|
| Role | Canonical question/concept table. |
| Grain | One canonical question concept, platform-agnostic. |
| Key data stored | Question key, item type, canonical title, canonical slug, summary, topics, patterns, difficulty, custom flag, lifetime support count. |
| Source tables | `silver.assessment_item_occurrence`, `catalog.coding_problems`, resolver outputs. |
| Product usage | The stable entity shown to users as a practice question. |

**Example rows**

| `question_key` | `item_type` | `canonical_title` | `topics` | `is_custom` |
|---|---|---|---|---|
| `lc:200` | `dsa_coding` | Number of Islands | `{graph, dfs, bfs}` | `false` |
| `sd:url-shortener` | `system_design` | Design a URL Shortener | `{system-design, scalability}` | `false` |
| `custom:amazon-s3-prefix-partitioning` | `domain_tech` | S3 Prefix Partitioning Deep Dive | `{storage, distributed-systems}` | `true` |

**Why this table exists:**

Raw interview prompts vary heavily. The same concept may appear as “LC 200”, “Number of Islands”, “island counting”, or a URL. Gold needs one stable row for counting and ranking.

---

## 14. `gold.question_references`

| Aspect | Details |
|---|---|
| Role | Connects canonical questions to external platform references. |
| Grain | One external reference per canonical question. |
| Key data stored | Question id, platform, external id/slug/url, reference type, confidence. |
| Source tables | `catalog.coding_problems`, `silver.assessment_item_occurrence.attributes.external_refs`, resolver. |
| Product usage | Allows DSA questions to link to LeetCode, Codeforces, HackerRank, or other platforms. |

**Example rows**

| Question | Platform | External id | External slug | Ref type |
|---|---|---|---|---|
| Number of Islands | `leetcode` | `200` | `number-of-islands` | `same_as` |
| Longest Duplicate Substring | `leetcode` | `1044` | `longest-duplicate-substring` | `same_as` |
| Parking Lot LLD | `company_named` | null | `parking-lot` | `named_problem` |

**Why this table exists:**

One canonical question may have multiple references or variants. Keeping references separate prevents `gold.questions` from becoming platform-specific.

---

## 15. `gold.question_occurrences`

| Aspect | Details |
|---|---|
| Role | Fact table mapping each Silver item occurrence to a canonical Gold question. |
| Grain | One row per `silver.assessment_item_occurrence`. |
| Key data stored | Occurrence id, question id, interview id, round id, item type, prompt text/hash, company, role family, canonical rank, interview date, mapping method, mapping score, routing state, public eligibility. |
| Source tables | `silver.assessment_item_occurrence`, `silver.interview`, `silver.round`, `gold.questions`, `gold.interview_roles`. |
| Product usage | Auditable base for all question trends. |

**Example row**

| Field | Example |
|---|---|
| `occurrence_id` | UUID from Silver assessment item |
| `question` | Number of Islands |
| `interview` | Google SDE2 interview UUID |
| `round_type` | `problem_solving_dsa` |
| `mapping_method` | `external_ref` |
| `mapping_score` | `0.99` |
| `routing` | `auto` |
| `public_eligible` | `true` |

**Why this table exists:**

It is the audit trail. If a user asks why a question is trending, we can trace from trend → occurrence → silver item → silver interview → evidence.

---

## 16. `gold.question_trends`

| Aspect | Details |
|---|---|
| Role | Product-serving ranking table for recently asked and trending questions. |
| Grain | One aggregate row per scope, question, item type, and time window. |
| Key data stored | Scope type, company, role family, canonical rank, item type, question id, window days, distinct interviews, raw occurrences, weighted frequency, interview relevance, trend score, first/last seen, round type counts, topic counts, sample ids/prompts. |
| Source tables | `gold.question_occurrences`, `gold.questions`, `gold.interview_roles`. |
| Product usage | Main API source for “top/trending/recently asked questions by company and role.” |

**Example row**

| Field | Example |
|---|---|
| `scope_type` | `exact_company_role` |
| `company` | Google |
| `role_family` | `software_general` |
| `canonical_rank` | `4 / Mid` |
| `item_type` | `dsa_coding` |
| `question` | Number of Islands |
| `window_days` | `180` |
| `distinct_interviews` | `8` |
| `weighted_frequency` | `6.42` |
| `trend_score` | `9.81` |
| `last_seen_at` | `2026-06-19` |

**Why this table exists:**

This is the fast product table. APIs should hit this table instead of recomputing trend scores from Silver on every request.

**Scope examples**

| Scope type | Meaning |
|---|---|
| `exact_company_role` | Company + role family + level. Example: Google + software_general + Mid. |
| `company_role_any_level` | Company + role family, any level. |
| `company_any_role` | Company-wide trends. |
| `global_role` | Same role family + level across all companies. |
| `global_family` | Same role family across all companies. |
| `global_all` | Global trends. |

---

## 17. `gold.interview_profiles`

| Aspect | Details |
|---|---|
| Role | Aggregated interview-loop intelligence by company/role/window. |
| Grain | One profile per scope and time window. |
| Key data stored | Total interviews, total rounds, total questions, average round count, median process days, round mix, ordered round pattern, item type mix, signal mix, top topics, common warnings. |
| Source tables | `silver.interview`, `silver.round`, `silver.signal`, `gold.question_occurrences`, `gold.interview_roles`. |
| Product usage | “What does the Google SDE2 interview loop usually look like?” |

**Example row**

| Field | Example |
|---|---|
| `scope_type` | `exact_company_role` |
| `company` | Amazon |
| `role_family` | `software_general` |
| `canonical_rank` | `4 / Mid` |
| `window_days` | `365` |
| `total_interviews` | `52` |
| `round_mix` | `{online_assessment: 21, problem_solving_dsa: 34, system_design_hld: 12, behavioral: 18}` |
| `ordered_round_pattern` | Recruiter → OA → DSA → System Design → HM |

**Why this table exists:**

Question lists alone do not explain the interview process. This table gives users role-specific interview-loop expectations.

---

## 18. `gold.search_documents`

| Aspect | Details |
|---|---|
| Role | Search and retrieval table for hybrid SQL + full-text + vector search. |
| Grain | One searchable document derived from a question, occurrence, round, signal, chunk, or evidence span. |
| Key data stored | Source type/id, question id, interview id, company/role filters, item type, text, text-search vector, embedding metadata, embedding vector, metadata. |
| Source tables | `gold.questions`, `gold.question_occurrences`, `silver.chunk`, `silver.signal`, `silver.evidence_span`, `silver.round`. |
| Product usage | Semantic search, evidence retrieval, hybrid RAG answers. |

**Example rows**

| Source type | Text | Usage |
|---|---|---|
| `question` | `Design a URL shortener with rate limits and analytics.` | Find canonical design prompt. |
| `signal` | `Candidates failed because they did not finish coding under time pressure.` | Find failure patterns. |
| `evidence` | `The interviewer asked Number of Islands and then a follow-up on BFS.` | Explain source evidence. |

**Why this table exists:**

SQL handles structured queries like company/role/frequency. Vector/full-text search handles vague natural-language questions and evidence retrieval. Keeping both in Postgres avoids a separate vector database early.

---

## 19. `gold.refresh_runs`

| Aspect | Details |
|---|---|
| Role | Operational metadata for Gold refresh jobs. |
| Grain | One refresh job run. |
| Key data stored | Job name, status, start/end time, source time range, rows inserted/updated/skipped, errors, metadata. |
| Source tables | Written by Gold jobs. |
| Product usage | Not user-facing. Used for debugging stale trends and failed refreshes. |

**Example row**

| Field | Example |
|---|---|
| `job_name` | `refresh_question_trends` |
| `status` | `success` |
| `rows_updated` | `1482` |
| `source_max_updated_at` | `2026-07-05 01:00:00+00` |

---

# Part C — Data flow example

## 20. Example: Google SDE2 interview asks Number of Islands

### Step 1 — Bronze

A LeetCode interview experience is discovered.

| Table | Stored data |
|---|---|
| `ingest.ingest_item` | Source id, URL, stage, S3 raw/extracted pointers. |

### Step 2 — Silver

The extractor loads structured interview data.

| Table | Stored data |
|---|---|
| `silver.interview` | Company = Google, role = SDE2 Backend, interview date = 2026-06-10, outcome = reject. |
| `silver.round` | Round 2 = DSA technical screen. |
| `silver.assessment_item_occurrence` | Prompt = `LC 200 Number of Islands`, item type = `dsa_coding`, external ref = LeetCode 200. |
| `silver.signal` | Failure reason = time management, if mentioned. |
| `silver.evidence_span` | Verified snippet from the raw post. |

### Step 3 — Gold role mapping

| Table | Stored data |
|---|---|
| `gold.role_levels` | Google/SDE2 maps to Mid rank if verified. |
| `gold.interview_roles` | This interview is assigned to Google + software_general + Mid. |

### Step 4 — Gold question mapping

| Table | Stored data |
|---|---|
| `gold.questions` | Canonical question = `lc:200`, Number of Islands. |
| `gold.question_references` | LeetCode 200 / number-of-islands. |
| `gold.question_occurrences` | This Silver occurrence maps to Number of Islands with method = external_ref. |

### Step 5 — Gold trend aggregation

| Table | Stored data |
|---|---|
| `gold.question_trends` | Google + Mid + DSA + 180-day trend count/score for Number of Islands increments. |
| `gold.interview_profiles` | Google Mid interview-loop round mix updates. |
| `gold.search_documents` | Search docs created for the question, evidence, and related signal text. |

---

# Part D — Table ownership summary

| Product question | Primary table |
|---|---|
| “What raw interview did this come from?” | `silver.interview` |
| “What exact prompt was reported?” | `silver.assessment_item_occurrence` |
| “What canonical question is this?” | `gold.questions` |
| “Where can I practice it?” | `gold.question_references` |
| “How often was it asked?” | `gold.question_trends` |
| “Which interviews support this trend?” | `gold.question_occurrences` |
| “What does this interview loop look like?” | `gold.interview_profiles` |
| “Search semantically across experiences.” | `gold.search_documents` |
| “How was role SDE2 mapped?” | `gold.role_levels` + `gold.interview_roles` |
| “Did the Gold refresh run successfully?” | `gold.refresh_runs` |

---

# Part E — Final recommendation

Use the following Gold table set:

1. `gold.role_levels`
2. `gold.interview_roles`
3. `gold.questions`
4. `gold.question_references`
5. `gold.question_occurrences`
6. `gold.question_trends`
7. `gold.interview_profiles`
8. `gold.search_documents`
9. `gold.refresh_runs`

This is enough to support:

- company-wise recently asked questions,
- role-wise trends,
- DSA mapping to LeetCode / Codeforces / other platforms,
- system design, LLD, machine coding, behavioral, SQL, domain, and debugging items,
- interview-loop intelligence,
- SQL-only fast paths,
- hybrid SQL + vector retrieval,
- auditable evidence-backed product output.

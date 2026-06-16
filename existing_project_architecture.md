# Existing Project Architecture and Data Model

**Project:** crackedinlabs/interview-prep-ai  
**Purpose of this document:** Capture what the project already has today: current data layers, important tables, how they are used, and what should be reused before adding any new learner-domain tables.

This document is intentionally architectural. It does not list every index in full DDL form, but it explains what each table/domain does, whether the data is global/shared or user-specific, and how it participates in the current AI tutor / interview-prep system.

---

## 1. Current architecture in one view

The project already has four major layers:

```text
1. Global content layer
   - coding problem catalog
   - interview corpus / round questions
   - knowledge RAG sources and chunks
   - topic/domain taxonomy

2. User activity layer
   - chat sessions and messages
   - LeetCode / coding submissions
   - progress-tracker events
   - context-memory events
   - checkpoint attempts

3. User state / projection layer
   - user topic progress
   - user profile summary
   - coding problem aggregates
   - prep plan progress
   - readiness snapshots

4. AI runtime layer
   - chat endpoint
   - extraction worker
   - progress aggregator
   - user-context reconciler
   - RAG retrieval
   - next-action engine
```

The most important design choice already present in the system is **event stream + projection**:

```text
raw activity / chat / sync event
        ↓
append-only event tables
        ↓
materialized user state tables
        ↓
fast reads for chat, dashboard, reports, and tutor behavior
```

That is the correct foundation. New learner-domain work should extend this pattern, not replace it.

---

## 2. Database schemas / namespaces

Current architecture uses multiple logical schemas.

| Schema | Role | Data type | Notes |
|---|---|---|---|
| `public` | Core auth/account mirror | User-specific identity | Supabase user-facing account data. |
| `catalog` | Coding problem catalog | Global/shared | Source-of-truth for platform problems such as LeetCode. |
| `user_data` | User coding sync data | User-specific | Synced coding profiles, problem status, submissions. |
| `code_blobs` | Submitted code bodies | User-specific / sensitive | Code text and code hashes. Should have stricter retention/privacy handling. |
| `system` | Sync control state | User-specific operational | Tracks extension sync phase/cursor/errors. |
| `app` | Product app data | Mixed | Chat, progress tracker, memory, prep plans, logs. |
| Legacy SQLite / corpus tables | Interview corpus and older app tables | Mixed | Still important because the interview corpus is valuable and should be bridged into the new learner domain. |

---

## 3. Core identity and account tables

### `public.users`

**Type:** User-specific identity table  
**Status:** Current Postgres/Supabase table  
**Purpose:** Stores the application user row that other user-owned tables reference.

**Important fields:**

- `id` — UUID primary key.
- `email` — unique account email.
- `display_name` — user-facing name.
- `auth_provider` — email, Google, anonymous, etc.
- `tier` — free/premium plan status.
- `sqlite_id` — legacy migration linkage.
- `created_at`, `updated_at`.

**Use in architecture:**

Everything user-specific should ultimately reference `public.users(id)`. The current Postgres code treats the Supabase `sub` UUID as the identity boundary. This is good and should continue.

**Notes:**

- This table is not the learner model.
- Do not add learning-state columns here.
- Keep it clean: identity, subscription, migration linkage.

---

## 4. Coding problem catalog and LeetCode sync domain

This domain is already strong. It tracks global coding problems and user-specific submissions/problem status. It should be reused as the primary evidence source for DSA progress.

### `catalog.coding_problems`

**Type:** Global/shared content table  
**Status:** Current Postgres table  
**Purpose:** Stores global problem metadata for coding platforms.

**Important fields:**

- `platform`, `slug` — composite primary key.
- `display_id`, `title`, `difficulty`.
- `topic_tags` — platform topic tags.
- `statement_md`, `constraints_md`, `examples_json`, `hints`.
- `similar_slugs`.
- `sync_status` — pending/complete state for catalog hydration.
- `raw_meta`.

**Use in architecture:**

This is the global problem catalog. For DSA, this should become one source for `learning_items`, not be replaced.

**Notes:**

- This table answers: “What problem exists?”
- It does not answer: “Has this user solved it?” That is handled by user-specific tables.

---

### `user_data.user_coding_profiles`

**Type:** User-specific sync profile  
**Status:** Current Postgres table  
**Purpose:** Stores the connection between a Crackedin user and an external coding platform account.

**Important fields:**

- `user_id`, `platform` — unique together.
- `external_user_id`, `external_handle`, `profile_url`.
- `is_premium`, `is_connected`, `sync_enabled`.
- `connected_at`, `last_full_sync_at`, `last_delta_sync_at`, `last_live_capture_at`.
- `last_sync_error`, `raw_profile`.

**Use in architecture:**

The extension sync layer uses this to identify which external account belongs to the user.

**Notes:**

- Keep this platform/account-specific.
- Do not add generic learner-state fields here.

---

### `user_data.user_coding_problems`

**Type:** User-specific projection table  
**Status:** Current Postgres table  
**Purpose:** Stores aggregate per-user, per-problem coding status.

**Important fields:**

- `user_id`, `platform`, `slug` — primary key.
- `status` — current aggregate status.
- `first_attempt_at`, `first_ac_at`, `latest_ac_at`, `latest_attempt_at`.
- `attempt_count`, `ac_count`, `wa_count`, `tle_count`, `mle_count`, `re_count`, `ce_count`, `other_count`.
- `best_runtime_ms`, `best_memory_kb`, `best_lang`.
- `primary_submission_id`.

**Use in architecture:**

This already answers a DSA-specific version of “what has the user solved?” It should feed the new generic learner item state for coding problems.

**Notes:**

- This is a coding-specific projection.
- The new learner-domain table should not duplicate all these stats. It should only normalize the item state into a cross-domain learner view.

---

### `user_data.user_coding_submissions`

**Type:** User-specific raw/event-like table  
**Status:** Current Postgres table  
**Purpose:** Stores every captured coding submission.

**Important fields:**

- `platform`, `submission_id` — primary key.
- `user_id`, `profile_id`, `slug`.
- `verdict`, `raw_status`, `status_code`.
- `lang`, `runtime_ms`, `runtime_percentile`, `memory_kb`, `memory_percentile`.
- `total_correct`, `total_testcases`.
- `last_testcase`, `expected_output`, `actual_output`.
- `runtime_error`, `compile_error`.
- `ts`, `captured_at`, `capture_source`, `code_capture_method`.
- `has_code`, `raw_meta`.

**Use in architecture:**

This is high-quality, deterministic learning evidence. Accepted submissions, failed submissions, compile errors, runtime errors, language choice, and failed testcases are all useful signals for the tutor.

**Notes:**

- Prefer this over LLM guessing for DSA evidence.
- Use accepted submissions to mark coding learning items as solved/verified.
- Use failed submissions to update `pt_learning_events`, weak topics, or future debugging guidance.

---

### `code_blobs.user_coding_submission_code`

**Type:** User-specific sensitive data  
**Status:** Current Postgres table  
**Purpose:** Stores actual submitted code or references to stored code.

**Important fields:**

- `platform`, `submission_id` — primary key and FK to submissions.
- `code`, `code_lang`, `code_hash`.
- `storage_tier`, `s3_key`, `fetched_at`.

**Use in architecture:**

This supports code review, debugging, style analysis, and deeper tutor feedback.

**Notes:**

- Treat as sensitive.
- Avoid injecting full code into prompts unless needed.
- For learner state, store derived signals elsewhere; do not make this a learner-state table.

---

### `system.sync_state`

**Type:** User-specific operational table  
**Status:** Current Postgres table  
**Purpose:** Tracks sync progress for external platform ingestion.

**Important fields:**

- `user_id`, `platform` — primary key.
- `phase`, `cursor`, `last_progress_at`.
- `last_full_sync_at`, `retry_count`, `last_error`.

**Use in architecture:**

The extension sync service uses this to resume, pause, or recover from sync operations.

**Notes:**

- Operational state only.
- Do not use this for product-facing learning state.

---

## 5. Chat and conversation domain

### `app.chat_sessions`

**Type:** User-specific session table  
**Status:** Current Postgres table  
**Purpose:** Groups chat messages into conversations.

**Important fields:**

- `id`, `user_id`, `title`.
- `attached_plan_id` — optional prep plan linkage.
- `pt_tracking_enabled` — whether progress extraction is enabled.
- `uc_pause_until` — temporary user-context memory pause.
- `created_at`, `updated_at`.

**Use in architecture:**

Chat sessions are the main user interface container for tutor interaction.

**Notes:**

- Keep it lightweight.
- Do not store learner state here.

---

### `app.chat_messages`

**Type:** User-specific conversation log  
**Status:** Current Postgres table  
**Purpose:** Stores individual user/assistant messages.

**Important fields:**

- `session_id`, `role`, `content`.
- `tool_calls`.
- `tokens_used`.
- `trace_id`.
- `created_at`.

**Use in architecture:**

Raw chat data used by the LLM, the extraction pipeline, audits, and search.

**Notes:**

- This is not a durable learner model by itself.
- The extractor should convert meaningful educational activity into structured events.

---

### `app.chat_feedback`

**Type:** User-specific feedback table  
**Status:** Current Postgres table  
**Purpose:** Captures thumbs-up/down feedback on assistant messages.

**Important fields:**

- `message_id`, `user_id`, `rating`.
- Unique per message/user.

**Use in architecture:**

The progress-tracker V2 design uses feedback as a quality signal to retroactively adjust event weights.

**Notes:**

- This is valuable because it closes the loop on assistant quality.
- Should influence prompt variants and future tutor policies.

---

## 6. User context and memory domain

The context-memory design already separates structured context from semantic memory.

### `app.user_context_events`

**Type:** User-specific append-only event table  
**Status:** Current Postgres table  
**Purpose:** Stores structured facts/events about the user extracted from chat or manual actions.

**Important fields:**

- `user_id`, `session_id`, `message_id`.
- `event_type`.
- `subject` — `self` or `not_self`.
- `confidence`.
- `source` — extraction/manual/chat_feedback/leetcode_sync/import.
- `extracted_json`.
- `response_quality`.
- `importance_score`, `ttl_days`.
- invalidation fields.
- `idempotency_key`, `shadow`, `created_at`.

**Use in architecture:**

This is the structured personal-context stream. It answers: “What did the user say about themselves, their goals, preferences, constraints, or state?”

**Notes:**

- Use for goals, target companies, preferences, emotional/situational state, constraints.
- Do not use as the main “what has the student solved?” table.

---

### `app.user_memory`

**Type:** User-specific semantic memory table  
**Status:** Current Postgres table  
**Purpose:** Stores open-ended memories about the user.

**Important fields:**

- `raw_text` — original remembered text.
- `summary` — concise model-written memory.
- `tags_json`.
- `importance_score`, `ttl_days`.
- invalidation fields.
- `source`, `shadow`, `created_at`.

**Use in architecture:**

This is for human-like continuity: projects, preferences, deadlines, external influences, personal constraints.

**Notes:**

- Excellent for mentor personality and continuity.
- Bad fit for deterministic mastery tracking.

---

### `app.user_profile_summary`

**Type:** User-specific projection table  
**Status:** Current Postgres table  
**Purpose:** Materialized compact profile used for every-turn tutor context.

**Important fields:**

- identity/profile: `current_company`, `years_experience`, `job_role`.
- target: `primary_target_id`.
- strengths/weaknesses JSON.
- preferences: learning style, depth, response length.
- state: current focus domain/topic, last active.
- emotional state with expiry.
- recent interview events.
- known company signals.
- provenance and rebuild metadata.

**Use in architecture:**

This is the “who is this user?” prompt block.

**Notes:**

- Keep this compact.
- Rebuild from events; do not manually mutate as a primary source of truth.
- It should include derived summaries, not detailed item-level solved history.

---

### `app.uc_forget_audit`

**Type:** User-specific audit table  
**Status:** Current Postgres table  
**Purpose:** Records user forget/delete memory commands.

**Important fields:**

- `user_id`, `command_text`, `resolved_intent`.
- `invalidated_event_ids`.
- `created_at`.

**Use in architecture:**

Supports reversible memory deletion and user trust.

**Notes:**

- Important for privacy and user control.
- Future learner-domain forget/correction flows should follow this pattern.

---

## 7. Progress tracker domain

This is the current backbone for topic-level preparation tracking.

### `app.pt_user_targets`

**Type:** User-specific goal table  
**Status:** Current Postgres table  
**Purpose:** Stores one or more target companies/levels/role families for the user.

**Important fields:**

- `user_id`, `company`, `level`, `role_family`.
- `is_active`, `is_primary`, `priority`.
- timestamps.

**Use in architecture:**

Targets determine prep plan, domain weighting, and tutor context.

**Notes:**

- Already supports multi-target prep.
- Do not duplicate target state in learner-domain tables.

---

### `app.pt_user_topic_progress`

**Type:** User-specific projection table  
**Status:** Current Postgres table  
**Purpose:** Tracks a user’s progress/readiness per topic.

**Important fields:**

- `user_id`, `topic_id`.
- `status` — not_started, covered, attempted, verified.
- `coverage_score`, `readiness_score`, `confidence_score`.
- `covered_depth`, `verified_depth`.
- `quiz_best_score`, `attempt_count`.
- timestamps: first seen, last seen, last practiced, last revised, next revision.
- `covered_subtopics_json`, `missing_subtopics_json`, `incomplete_checkpoints_json`.

**Use in architecture:**

Topic-level learner state. This should remain the main topic mastery/readiness table.

**Notes:**

- Do not create a new topic mastery table yet.
- Use this for “weak topics,” “covered topics,” “needs revision,” and dashboard readiness.

---

### `app.pt_checkpoint_attempts`

**Type:** User-specific assessment table  
**Status:** Current Postgres table  
**Purpose:** Stores checkpoint/quiz attempts.

**Important fields:**

- `user_id`, `topic_id`, `checkpoint_topic_id`.
- `level`, `score`, `passed`.
- `questions_json`, `answers_json`.
- `weak_subtopic_ids_json`.
- `evaluator_feedback`, `duration_ms`.

**Use in architecture:**

Use this for quizzes, validation, checkpoint assessment, and future course/module checks.

**Notes:**

- Strong fit for both DSA and non-DSA knowledge checks.
- Do not create a new quiz table unless checkpoint behavior becomes too different.

---

### `app.pt_learning_events`

**Type:** User-specific append-only event table  
**Status:** Current Postgres table  
**Purpose:** Stores learning progress events extracted from chat, checkpoints, plan tasks, calibration, and manual actions.

**Important fields:**

- `user_id`, `session_id`, `message_id`.
- `event_type`: taught, covered, checkpoint_started, checkpoint_passed, checkpoint_failed, self_reported_known, revised, struggled, skipped, plan_task_completed, etc.
- `area_slug`, `topic_id`, `subtopic_ids_json`.
- `depth_inferred`.
- `coverage_delta`, `readiness_delta`, `confidence`.
- `response_quality`, `source`.
- `metadata_json`.
- `quality_weight`, idempotency, shadow, invalidation.

**Use in architecture:**

This is the existing learning event ledger. It should be the event source for any simplified learner-domain projection.

**Notes:**

- Use `metadata_json` to attach `learning_item_id`, help level, rubric score, source ID, etc.
- Avoid creating new event tables until necessary.

---

### `app.user_prep_plans`

**Type:** User-specific plan table  
**Status:** Current Postgres table  
**Purpose:** Tracks a user’s active/paused/completed prep plan.

**Important fields:**

- `user_id`, `template_id`, `target_date`.
- `status`.
- `template_version_seen`, `hours_per_day_override`, `paused_at`.

**Use in architecture:**

Supports scheduled prep and dashboard progress.

**Notes:**

- Use plan tasks as one source of learning events.

---

### `app.user_prep_plan_feedback`

**Type:** User-specific feedback table  
**Status:** Current Postgres table  
**Purpose:** Captures feedback about prep plans.

**Important fields:**

- `user_prep_plan_id`, `rating`, `text`.

**Use in architecture:**

Product feedback for plan quality.

---

### `app.user_prep_task_progress`

**Type:** User-specific task projection table  
**Status:** Current Postgres table  
**Purpose:** Tracks progress through individual prep-plan tasks.

**Important fields:**

- `user_prep_plan_id`, `day_number`, `task_ref`.
- `status`: pending, done, hinted, skipped.
- `time_spent_minutes`, `note`, `revision_due_date`, `completed_at`.

**Use in architecture:**

Prep-plan task completion source. Completed tasks can emit `pt_learning_events`.

---

### `app.user_readiness_snapshots`

**Type:** User-specific historical snapshot table  
**Status:** Current Postgres table  
**Purpose:** Stores readiness score over time for a prep plan.

**Important fields:**

- `user_prep_plan_id`, `computed_at`, `readiness_score`.
- `component_breakdown`, days elapsed, tasks done/total.

**Use in architecture:**

Historical readiness trend. Useful for reports.

---

### `app.user_contest_history`

**Type:** User-specific external-history table  
**Status:** Current Postgres table  
**Purpose:** Stores coding contest performance.

**Important fields:**

- `user_id`, `platform`, `contest_title`.
- `ranking`, `rating`, `problems_solved`, `contest_date`.

**Use in architecture:**

Good optional signal for DSA ability and trajectory.

---

### `app.user_read_posts`

**Type:** User-specific content interaction table  
**Status:** Current Postgres table  
**Purpose:** Tracks interview/corpus posts a user has read.

**Important fields:**

- `user_id`, `post_id`, `read_at`.

**Use in architecture:**

This can indicate exposure to interview experiences, but reading should not equal mastery.

**Notes:**

- Treat as “seen,” not “learned” or “solved.”

---

## 8. Observability / logs

### `app.api_logs`

**Type:** Mixed operational/product log  
**Status:** Current Postgres table  
**Purpose:** Best-effort event log for API/product events.

**Important fields:**

- `event_type`, `event_data`, `user_id`, `duration_ms`, `ip_address`, `created_at`.

**Use in architecture:**

Operational logging, analytics, debugging.

**Notes:**

- Do not use as authoritative learner state.

---

## 9. Legacy / interview corpus architecture

The project also has an older but highly valuable interview corpus schema. Even if some of this is not in the current Postgres full schema, it represents product data that should be reused, especially for `learning_items`.

### `problems`

**Type:** Global/shared legacy coding catalog  
**Purpose:** Older LeetCode-style problem table.

**Important fields:**

- LeetCode number, name, slug, difficulty.
- topics, acceptance rate, description, hints, URL.

**Use:**

Can be mapped into modern `catalog.coding_problems` or directly into `learning_items` during migration/backfill.

---

### `interview_posts`

**Type:** Global/shared interview corpus metadata  
**Purpose:** Stores metadata and extracted dimensions for interview-experience posts.

**Important fields:**

- source URL/platform, topic ID, title, post date/year.
- company, company tier, industry.
- role, role family, role level, seniority, specialization.
- location, work arrangement, visa.
- candidate experience, education, previous company tier.
- process channel, outcome, total rounds, negotiation data.
- pipeline stage, quality, dedup status, content hash.

**Use:**

This is market/interview intelligence. It should feed company-specific and role-specific prep context.

---

### `interview_rounds`

**Type:** Global/shared corpus table  
**Purpose:** Stores each interview round inside a post.

**Important fields:**

- `interview_post_id`, round order, label/type.
- duration, programming language, environment/platform.
- interviewer role/helpfulness, hints, round outcome, notes.

**Use:**

Helps model actual interview process by company/role/level.

---

### `round_questions`

**Type:** Global/shared corpus table  
**Purpose:** Stores questions asked in interview rounds.

**Important fields:**

- `round_id`, `question_type`.
- optional `problem_id` link.
- title, description, topics, difficulty.
- expected approach, candidate result.

**Use:**

This is one of the most important sources for the new `learning_items` table. It already supports DSA, HLD, LLD, API design, data modeling, debugging, code review, concurrency, OS, networking, behavioral, product sense, and more.

---

### `compensations`

**Type:** Global/shared corpus table  
**Purpose:** Stores compensation data extracted from interview posts.

**Use:**

Not a learner-state table. Useful for company intelligence and product surfaces around offers/negotiation.

---

### `post_quotes`

**Type:** Global/shared corpus evidence table  
**Purpose:** Stores real quotes from posts/comments.

**Use:**

Good for grounded, human-sounding RAG responses and company/process context.

---

### `post_insights`

**Type:** Global/shared corpus insight table  
**Purpose:** Stores tips, strategies, warnings, and general insights extracted from posts.

**Use:**

Useful for RAG and interview prep advice.

---

### `prep_profiles`

**Type:** Global/shared extracted profile table  
**Purpose:** Stores how a candidate prepared: months preparing, hours/day, DSA problems solved, mocks, resources, study group, etc.

**Use:**

Useful for benchmarking users against historical candidates.

---

### `post_timeline_events`

**Type:** Global/shared corpus timeline table  
**Purpose:** Stores application/interview timeline events.

**Use:**

Useful for process expectations and company-specific timelines.

---

### `technologies` and `post_technologies`

**Type:** Global/shared normalized vocabulary + join table  
**Purpose:** Tracks technologies mentioned in interview posts.

**Use:**

Can map technologies to topics/learning items in future domains like tools, backend, data engineering, DevOps, frontend, ML infra.

---

### `resources` and `post_resources`

**Type:** Global/shared resource catalog + join table  
**Purpose:** Tracks books, courses, platforms, and resources mentioned by candidates.

**Use:**

Useful for content recommendations and prep context, but not the core recommendation engine.

---

### `hiring_signals`

**Type:** Global/shared company intelligence table  
**Purpose:** Tracks temporal hiring signals by company.

**Use:**

Useful for company-aware prep and user target context.

---

### `company_profiles`

**Type:** Global/shared derived table  
**Purpose:** Stores derived company profiles such as typical process length, rounds, H1B sponsorship, remote friendliness, offer rate, and total posts.

**Use:**

Useful for target-aware coaching.

---

### `interview_posts_raw`, `interview_posts_comments`, `interview_posts_analysis`, `interview_posts_fts`

**Type:** Global/shared corpus storage/search tables  
**Purpose:** Store raw post content, comments, AI analysis, and full-text search index.

**Use:**

Backbone for interview corpus retrieval, extraction, and citations.

---

## 10. Knowledge RAG architecture

The project has a v11 knowledge-RAG migration/design that adds a general-purpose knowledge corpus.

### `knowledge_sources`

**Type:** Global/shared content source table  
**Purpose:** Stores metadata about high-quality knowledge sources.

**Important fields:**

- `area`, `source_type`, publisher/authors/title.
- URL, published date, last checked date.
- deprecation flag.
- reputation score.
- citability flag.
- content hash, raw object key, metadata JSON.

**Use:**

Represents canonical papers, engineering blogs, books, docs, articles, videos, etc.

---

### `knowledge_chunks`

**Type:** Global/shared RAG chunk table  
**Purpose:** Stores heading-aware chunks extracted from knowledge sources.

**Important fields:**

- source ID, area, chunk type, topic.
- content, raw chunk text, keywords.
- section title, heading path, page range.
- previous/next chunk linkage.
- figure/table/code flags.
- content hash.

**Use:**

Core source for grounded tutor explanations.

---

### `knowledge_chunks_vec`

**Type:** Global/shared vector index  
**Purpose:** Stores embeddings for semantic retrieval.

**Use:**

Semantic search over the knowledge corpus.

---

### `knowledge_chunks_fts`

**Type:** Global/shared full-text index  
**Purpose:** Stores FTS over chunk content/topic/keywords.

**Use:**

Hybrid retrieval with vector search.

---

## 11. Current core data flows

### Chat flow

```text
User sends message
  ↓
app.chat_messages stores user message
  ↓
LLM responds
  ↓
app.chat_messages stores assistant message
  ↓
Extraction worker analyzes turn
  ↓
Writes pt_learning_events and/or user_context_events
  ↓
Progress aggregator updates pt_user_topic_progress
  ↓
Context reconciler updates user_profile_summary
```

### Coding sync flow

```text
Extension connects external profile
  ↓
user_coding_profiles created/updated
  ↓
metadata/code batches ingested
  ↓
user_coding_submissions stores raw submissions
  ↓
code_blobs stores code when available
  ↓
user_coding_problems recomputes aggregate per problem
  ↓
future learner-domain projector can mark learning item solved/attempted
```

### Progress tracker flow

```text
Learning event
  ↓
pt_learning_events append-only row
  ↓
aggregator applies coverage/readiness deltas
  ↓
pt_user_topic_progress updated
  ↓
next-action engine selects revision/incomplete/plan/reorientation actions
```

### Context-memory flow

```text
Context event
  ↓
user_context_events append-only row
  ↓
reconciler rebuilds user_profile_summary
  ↓
chat prompt receives compact user profile block
```

---

## 12. What the project already does well

1. **Good separation of global and user-specific data.**
   - Catalog/corpus data is global.
   - User activity/progress is user-specific.

2. **Event + projection architecture.**
   - `pt_learning_events` is the event ledger.
   - `pt_user_topic_progress` is the materialized topic view.
   - `user_context_events` is the personal-context event stream.
   - `user_profile_summary` is the materialized user-profile view.

3. **Strong DSA sync foundation.**
   - Submissions and per-problem aggregates are already available.

4. **Rich interview corpus.**
   - The system already has a broad schema for interviews across DSA, design, behavioral, domain, SQL, debugging, and more.

5. **Good forward compatibility.**
   - Free-text domains/question types.
   - JSON metadata fields.
   - Provisional topics.
   - Shadow mode.
   - Invalidation instead of hard delete.

---

## 13. Current gaps before adding learner domain

1. **No universal learnable item abstraction.**
   - DSA problems, interview questions, design cases, knowledge chunks, and future course tasks are not unified.

2. **No cross-domain item state.**
   - The project can track coding-specific problem status and topic progress, but not a generic “this user solved/practiced/completed this item” state across all domains.

3. **Topic progress is not item progress.**
   - `pt_user_topic_progress` is useful, but it cannot say exactly which design case, question, lab, or coding problem a user solved.

4. **Reading/exposure can be confused with mastery if not careful.**
   - `user_read_posts` means seen/read, not solved/understood.

5. **RAG knowledge is global; learner state is separate.**
   - The tutor needs a bridge between retrieved content and user-specific solved/attempted state.

---

## 14. Architecture principle for future changes

Before creating a new table, ask:

1. Can this be stored as a `pt_learning_events.metadata_json` field?
2. Can this be derived into `pt_user_topic_progress`?
3. Is this actually a global content object?
4. Is this actually a user-specific current-state projection?
5. Will this table be useful across multiple future domains?

Only create new tables when the answer is clearly yes for cross-domain reuse.

For the learner domain, the minimal necessary new tables are:

1. `app.learning_items` — global/shared universal learnable item catalog.
2. `app.user_learning_item_state` — user-specific current state per learning item.

Everything else should reuse the existing architecture.

# Tutor Architecture PostgreSQL Tables Reference

This document is a working reference for the Crackedin Labs adaptive tutor architecture.

It includes:

1. Tables already present in the current PostgreSQL database.
2. The 7 recommended new PostgreSQL tables needed for the tutor architecture.
3. The intended role of each table in the Domain Layer, Student Model Layer, Tutor Orchestrator flow, event logging, and progress tracking.

## Core Decision

PostgreSQL is the production runtime source of truth.

SQLite local tables should not be used by the production tutor runtime. SQLite can be used only as a seed or migration source for curriculum, topics, concept relationships, checkpoints, system design cases, and knowledge content.

The tutor architecture should stay minimal. Do not recreate every SQLite table in PostgreSQL. Use a compact set of domain, event, projection, and universal learning-item tables instead.

---

# 1. Existing PostgreSQL Tables

These tables already exist in the current PostgreSQL database and should be reused.

## 1.1 Identity Tables

| Schema | Table | Existing/New | Tutor Architecture Role |
|---|---|---:|---|
| `public` | `users` | Existing | Canonical user identity table. All user-specific tutor state should reference this table. |

### Notes

`public.users` should remain the identity anchor for:

- chat sessions,
- prep plans,
- coding data,
- learning events,
- topic state,
- learning-item state.

Do not create a separate tutor-specific users table.

---

## 1.2 Chat Tables

| Schema | Table | Existing/New | Tutor Architecture Role |
|---|---|---:|---|
| `app` | `chat_sessions` | Existing | Stores chat session containers. Used to group tutor conversations. |
| `app` | `chat_messages` | Existing | Stores user and assistant messages. Used as raw conversation history and evidence references. |
| `app` | `chat_feedback` | Existing | Stores message-level feedback. Can help weight response quality and extraction confidence. |

### Notes

These tables should continue to store the raw chat experience.

The tutor should reference `chat_sessions` and `chat_messages` from learning events where useful, but learning progress should not be stored only inside chat messages.

Recommended usage:

- `chat_messages` stores the raw conversation.
- `learning_events` stores structured learning evidence extracted from the conversation.
- `user_topic_state` and `user_learning_item_state` store current projected progress.

Do not create another generic chat-log table.

---

## 1.3 Application Logs and Audit Tables

| Schema | Table | Existing/New | Tutor Architecture Role |
|---|---|---:|---|
| `app` | `api_logs` | Existing | Product/API logging and operational analytics. Can temporarily store lightweight tutor traces if needed. |
| `app` | `uc_forget_audit` | Existing | Forget/delete audit trail. Relevant for privacy and user-data lifecycle. |

### Notes

`api_logs` may be used for temporary debugging or rollout traces, but should not become the source of truth for learner progress.

For tutor progress, use:

- `learning_events`,
- `user_topic_state`,
- `user_learning_item_state`.

Do not create `app.tutor_plan_logs` in V1 unless debugging or analytics requirements become strong enough.

---

## 1.4 Prep Plan Tables

| Schema | Table | Existing/New | Tutor Architecture Role |
|---|---|---:|---|
| `app` | `user_prep_plans` | Existing | Durable multi-day or multi-week prep plans. Use only for actual roadmap/plan flows. |
| `app` | `user_prep_task_progress` | Existing | Tracks user progress against durable prep-plan tasks. |
| `app` | `user_prep_plan_feedback` | Existing | Stores feedback on generated or active prep plans. |

### Notes

A `TutorPlan` and a `Prep Plan` are different.

| Concept | Scope | Stored? | Example |
|---|---|---:|---|
| TutorPlan | One chat turn or short interaction | Usually not as a first-class table in V1 | "For this answer, explain deadlock with a prerequisite bridge." |
| Prep Plan | Multi-day or multi-week roadmap | Yes, existing prep-plan tables | "Prepare for backend interviews over 45 days." |

Do not create or update a durable prep plan every time the user asks a concept question.

Create or update `user_prep_plans` only when:

- the user explicitly asks for a plan,
- onboarding/calibration creates one,
- the user enters formal prep mode,
- the user changes target role/company/date,
- the product intentionally reorients their roadmap.

---

## 1.5 Coding Catalog and User Coding Data

| Schema | Table | Existing/New | Tutor Architecture Role |
|---|---|---:|---|
| `catalog` | `coding_problems` | Existing | Global coding problem catalog. Source for DSA/coding `learning_items`. |
| `user_data` | `user_coding_profiles` | Existing | User's connected coding-platform profile. Useful for platform-level context. |
| `user_data` | `user_coding_problems` | Existing | Per-user aggregate coding problem state. Useful for topic/item evidence. |
| `user_data` | `user_coding_submissions` | Existing | Per-user coding submissions and verdicts. Strong evidence for DSA progress. |
| `code_blobs` | `user_coding_submission_code` | Existing | Stores submitted code blobs. Useful for code review/debugging with privacy controls. |

### Notes

Coding data should be treated as strong evidence for DSA/coding learning state.

Example mappings:

| Coding Evidence | Tutor Interpretation |
|---|---|
| Accepted submission without significant help | `solved_independently` or `verified` item state. |
| Accepted submission after hints/full solution | `solved_with_help`, not fully verified. |
| Wrong answer | `attempted` or `struggled`. |
| TLE | Possible complexity weakness. |
| Runtime error | Possible implementation/debugging weakness. |
| Repeated failed submissions | Strong weakness signal through `learning_events`. |

`catalog.coding_problems` should be backfilled into `app.learning_items` with `item_type = 'coding_problem'`.

---

# 2. Recommended New PostgreSQL Tables

These are the 7 recommended new tables for the adaptive tutor architecture.

## Summary

| # | Schema | Table | Layer | Required for V1? |
|---:|---|---|---|---:|
| 1 | `app` | `pt_areas` | Domain Layer | Yes |
| 2 | `app` | `pt_topics` | Domain Layer | Yes |
| 3 | `app` | `pt_concept_relationships` | Domain Layer | Yes |
| 4 | `app` | `learning_events` | Student Model / Event Ledger | Yes |
| 5 | `app` | `user_topic_state` | Student Model / Projection | Yes |
| 6 | `app` | `learning_items` | Domain + Student Bridge | Yes |
| 7 | `app` | `user_learning_item_state` | Student Model / Projection | Yes |

---

## 2.1 `app.pt_areas`

### Purpose

Stores high-level learning domains or curriculum areas.

Examples:

- DSA,
- system design,
- operating systems,
- computer networks,
- databases,
- SQL,
- data engineering,
- backend engineering,
- distributed systems,
- cloud,
- debugging,
- tools.

### Why Needed

The tutor needs a canonical domain layer before it can reason about topics, prerequisites, and progress.

Without this table, domains would be hardcoded or inferred inconsistently.

### Example Uses

| User Query | Area |
|---|---|
| "What is deadlock?" | `operating_systems` |
| "Explain TCP vs UDP." | `computer_networks` |
| "Design a rate limiter." | `system_design` |
| "Give me Kafka questions." | `data_engineering` |
| "Review my SQL query." | `sql` |

### Conceptual Fields

Do not treat this as final DDL. This is a design reference.

```text
id
slug
name
description
is_active
metadata_json
created_at
updated_at
```

---

## 2.2 `app.pt_topics`

### Purpose

Stores canonical topics, subtopics, modules, skills, concepts, and curriculum nodes.

This table should represent both topics and subtopics.

Do not create a separate `pt_subtopics` table in V1.

### Why Needed

The tutor needs stable topic IDs for:

- prerequisite mapping,
- topic resolution,
- user progress,
- event logging,
- learning items,
- recommendations,
- review scheduling,
- analytics.

### Example Topic Hierarchy

```text
Operating Systems
  -> Synchronization
      -> Locks
      -> Deadlock
          -> Coffman conditions
          -> Deadlock prevention
          -> Deadlock detection

Databases
  -> Transactions
      -> Isolation levels
      -> Locks
      -> Deadlocks

System Design
  -> Rate Limiting
      -> Token bucket
      -> Leaky bucket
      -> Distributed rate limiting
```

### Conceptual Fields

```text
id
area_slug
slug
name
description
parent_topic_id
topic_type
difficulty
is_active
metadata_json
created_at
updated_at
```

### Recommended `topic_type` Values

```text
area
module
concept
subtopic
skill
case
checkpoint
```

Use only what is needed early. Avoid over-modeling.

---

## 2.3 `app.pt_concept_relationships`

### Purpose

Stores prerequisite and concept relationships between topics.

### Why Needed

This table lets the tutor understand learning dependency structure.

It answers:

- What does the user need to know before this topic?
- What topics are related?
- What does this topic unlock?
- Which prerequisite should be repaired first?
- Which concepts are often confused?

### Example Relationships

| From Topic | Relationship | To Topic |
|---|---|---|
| `deadlock` | `requires` | `process` |
| `deadlock` | `requires` | `thread` |
| `deadlock` | `requires` | `lock` |
| `deadlock` | `requires` | `resource_allocation` |
| `consistent_hashing` | `requires` | `hashing` |
| `kafka_consumer_groups` | `requires` | `partitioning` |
| `transaction_deadlocks` | `related_to` | `os_deadlocks` |

### Conceptual Fields

```text
id
from_topic_id
to_topic_id
relationship_type
strength
metadata_json
created_at
updated_at
```

### Recommended Relationship Types

Start with:

```text
requires
related_to
```

Add later only when needed:

```text
unlocks
part_of
often_confused_with
similar_to
contrasts_with
```

---

## 2.4 `app.learning_events`

### Purpose

Append-only ledger of structured learning evidence.

This table records what happened pedagogically.

It should replace the need for separate early tables such as:

- `user_topic_log`,
- `user_attempt_log`,
- `user_weakness_signal`,
- `user_revisit_queue`.

### Why Needed

The tutor needs auditable learning evidence, not only current state.

Examples:

- tutor taught a topic,
- user saw a concept,
- user attempted a problem,
- user passed a diagnostic,
- user failed a diagnostic,
- user self-reported knowing something,
- user used a hint,
- user solved independently,
- user solved with full solution help,
- user struggled repeatedly.

### Event vs Projection

| Layer | Table | Role |
|---|---|---|
| Raw evidence | `learning_events` | Append-only source of learning truth. |
| Current state | `user_topic_state` | Fast topic-level progress projection. |
| Current item state | `user_learning_item_state` | Fast item-level progress projection. |

### Conceptual Fields

```text
id
user_id
session_id
message_id
area_slug
topic_id
learning_item_id
event_type
source
confidence
coverage_delta
readiness_delta
help_level
score
metadata_json
idempotency_key
created_at
```

### Recommended Event Types

```text
seen
taught
covered
attempted
struggled
self_reported_known
diagnostic_passed
diagnostic_failed
reviewed
verified
checkpoint_passed
checkpoint_failed
plan_task_completed
coding_submission_accepted
coding_submission_failed
```

### Important Rules

- This table should be append-only.
- Do not silently mutate learning history.
- If an event is wrong, add correction/invalidation metadata or a compensating event.
- Self-report is low-confidence evidence.
- Exposure is not mastery.
- Assisted solving is not the same as independent solving.

---

## 2.5 `app.user_topic_state`

### Purpose

Stores the current projected user state for each topic.

This is the fast-read table the tutor uses during chat.

### Why Needed

The tutor should not scan and aggregate every learning event on every user message.

This table answers quickly:

- Has the user seen this topic?
- Has it been taught?
- Did the user struggle?
- Is the topic verified?
- Are prerequisites missing?
- Is the topic due for review?
- What is the confidence in the state?

### Conceptual Fields

```text
user_id
topic_id
status
coverage_score
readiness_score
confidence_score
attempt_count
first_seen_at
last_seen_at
last_practiced_at
last_verified_at
next_review_at
missing_prerequisites_json
misconception_signals_json
evidence_json
created_at
updated_at
```

### Recommended Statuses

```text
not_started
seen
taught
attempted
struggled
covered
needs_review
verified
```

### Important Rules

- `user_topic_state` is a projection, not the raw source of truth.
- It should be updated by projectors consuming `learning_events` and other trusted evidence.
- The tutor should read this table before deciding whether to explain directly, probe prerequisites, repair prerequisites, or assess.

---

## 2.6 `app.learning_items`

### Purpose

Universal catalog of things a user can learn, practice, solve, review, or be assessed on.

This is the key table that prevents domain-specific table explosion.

### What Counts as a Learning Item?

A learning item can be:

- coding problem,
- system design case,
- SQL exercise,
- debugging task,
- OS diagnostic,
- networking scenario,
- data engineering lab,
- course lesson,
- interview question,
- checkpoint,
- reading item,
- project/lab.

### Why Needed

Without this abstraction, we would end up creating many specialized tables:

```text
user_solved_system_design_cases
user_completed_sql_exercises
user_debugging_task_progress
user_checkpoint_attempts
user_data_engineering_lab_progress
```

Instead, all of these can be represented through:

```text
learning_items
user_learning_item_state
learning_events
```

### Conceptual Fields

```text
id
area_slug
primary_topic_id
topic_ids_json
item_type
title
difficulty
verification_type
source_type
source_ref_json
estimated_minutes
is_active
metadata_json
created_at
updated_at
```

### Recommended Item Types

```text
concept
coding_problem
system_design_case
sql_exercise
debugging_task
quiz
checkpoint
reading
project_lab
interview_question
diagnostic_question
```

### Recommended Verification Types

```text
none
read
self_explain
quiz
code_acceptance
rubric_eval
mock_interview
```

### Example Source Mappings

| Source | Learning Item Representation |
|---|---|
| `catalog.coding_problems` | `item_type = 'coding_problem'` |
| SQLite `pt_system_design_cases` | `item_type = 'system_design_case'` |
| SQLite `pt_checkpoints` | `item_type = 'checkpoint'` |
| SQLite `pt_checkpoint_questions` | `item_type = 'diagnostic_question'` or `quiz` |
| Future SQL exercise bank | `item_type = 'sql_exercise'` |
| Future debugging cases | `item_type = 'debugging_task'` |

---

## 2.7 `app.user_learning_item_state`

### Purpose

Stores the current projected user state for each learning item.

Where `user_topic_state` answers topic-level readiness, this table answers item-level progress.

### Why Needed

Topic-level state is not enough.

Example:

```text
Topic state:
  User is improving in graph traversal.

Item state:
  User solved Number of Islands with hints.
  User solved BFS traversal independently.
  User failed Word Ladder twice.
  User needs to review shortest-path modeling.
```

### Conceptual Fields

```text
user_id
learning_item_id
state
best_outcome
best_score
attempt_count
independent_success_count
assisted_success_count
failure_count
first_seen_at
first_attempted_at
first_solved_at
last_attempted_at
last_solved_at
last_verified_at
next_review_at
last_help_level
evidence_json
created_at
updated_at
```

### Recommended States

```text
not_started
seen
attempted
struggled
solved_with_help
solved_independently
completed
verified
needs_review
stale
skipped
```

### Help Level Model

Use help level to distinguish independent work from assisted work.

Recommended scale:

```text
0 = no help; independent
1 = small nudge
2 = hint
3 = scaffolded approach
4 = partial solution
5 = full solution / answer-level help
```

Suggested interpretation:

| Help Level | Success Result | Suggested State |
|---:|---|---|
| 0 | success | `solved_independently` |
| 1 | success | `solved_independently` or high-confidence assisted success |
| 2-4 | success | `solved_with_help` |
| 5 | success | assisted completion, not verified |

Important rule:

A user who solved after receiving the full answer should not be marked as independently verified.

---

# 3. Full Table List for Tutor Architecture Reference

## 3.1 Existing Tables to Reuse

| # | Schema | Table | Role |
|---:|---|---|---|
| 1 | `public` | `users` | User identity. |
| 2 | `app` | `api_logs` | API/product logs. |
| 3 | `app` | `chat_feedback` | Message feedback. |
| 4 | `app` | `chat_messages` | Raw conversation messages. |
| 5 | `app` | `chat_sessions` | Chat session containers. |
| 6 | `app` | `uc_forget_audit` | Forget/delete audit trail. |
| 7 | `app` | `user_prep_plan_feedback` | Feedback on durable prep plans. |
| 8 | `app` | `user_prep_plans` | Durable roadmap/prep plans. |
| 9 | `app` | `user_prep_task_progress` | Task progress inside prep plans. |
| 10 | `catalog` | `coding_problems` | Global coding problem catalog. |
| 11 | `code_blobs` | `user_coding_submission_code` | Submitted code blobs. |
| 12 | `user_data` | `user_coding_problems` | Per-user coding problem aggregate state. |
| 13 | `user_data` | `user_coding_profiles` | User coding-platform profiles. |
| 14 | `user_data` | `user_coding_submissions` | User coding submissions and verdicts. |

## 3.2 New Tables to Add

| # | Schema | Table | Role |
|---:|---|---|---|
| 15 | `app` | `pt_areas` | High-level domains. |
| 16 | `app` | `pt_topics` | Canonical topics/subtopics/modules. |
| 17 | `app` | `pt_concept_relationships` | Prerequisites and concept graph. |
| 18 | `app` | `learning_events` | Append-only learning evidence ledger. |
| 19 | `app` | `user_topic_state` | Current user-topic progress/readiness projection. |
| 20 | `app` | `learning_items` | Universal catalog of learnable/practiceable/verifiable items. |
| 21 | `app` | `user_learning_item_state` | Current per-user learning-item state. |

## 3.3 Total V1 Table Surface

```text
Existing PostgreSQL tables reused: 14
New tutor tables recommended:      7
Total tutor architecture surface:  21 tables
```

---

# 4. Tables Not Recommended for V1

Do not create these now unless a specific product requirement forces it.

| Deferred Table | Why Not Now |
|---|---|
| `app.pt_subtopics` | Use `pt_topics.parent_topic_id` and `topic_type`. |
| `app.pt_learning_paths` | Use concept relationships plus existing prep-plan tables. |
| `app.pt_checkpoints` | Represent checkpoints as `learning_items`. |
| `app.pt_checkpoint_questions` | Represent questions as `learning_items` with item type `diagnostic_question` or `quiz`. |
| `app.pt_system_design_cases` | Represent system design cases as `learning_items`. |
| `app.user_weakness_signal` | Store weakness evidence in `learning_events` and projection JSON. |
| `app.user_revisit_queue` | Use `next_review_at` on topic/item state. |
| `app.user_attempt_log` | Use `learning_events` and item-state attempt counters. |
| `app.user_misconceptions` | Store in event metadata and state JSON first. |
| `app.tutor_plan_logs` | Keep TutorPlan ephemeral in V1; use existing logs/traces if needed. |
| `app.assessment_rubrics` | Store small rubrics in `learning_items.metadata_json` first. |
| `app.learning_item_topic_map` | Use `topic_ids_json` first; create join table later only if querying demands it. |

---

# 5. Optional Later Tables

These are useful but not required in the first implementation.

## 5.1 `app.user_context_events`

### Purpose

Separate durable user context from learning mastery.

Examples:

- target role,
- target company,
- preferred explanation style,
- available preparation time,
- user says they want more system design,
- user says they prefer concise answers.

### Why Later

For V1, some of this can be inferred from:

- `chat_messages`,
- `chat_feedback`,
- `user_prep_plans`,
- metadata fields.

Create this when personalization becomes more important.

## 5.2 `app.user_profile_summary`

### Purpose

Compact materialized user profile for fast tutor prompts.

Example fields:

```text
target_role
target_level
preferred_style
known_strengths
known_weaknesses
recent_focus
active_goal
updated_at
```

### Why Later

Useful once there are enough context events and learning events to summarize.

Do not create prematurely if the first implementation does not use it.

---

# 6. Runtime Architecture Mapping

## 6.1 DomainService Reads

DomainService should primarily read:

```text
app.pt_areas
app.pt_topics
app.pt_concept_relationships
app.learning_items
catalog.coding_problems
```

Later, if RAG is moved to Postgres, it may also read:

```text
app.knowledge_sources
app.knowledge_chunks
```

Not required in the 7-table V1 foundation.

## 6.2 StudentModelService Reads

StudentModelService should primarily read:

```text
app.learning_events
app.user_topic_state
app.user_learning_item_state
app.chat_messages
app.chat_feedback
app.user_prep_plans
app.user_prep_task_progress
user_data.user_coding_problems
user_data.user_coding_submissions
```

## 6.3 TutorOrchestrator Uses

TutorOrchestrator should consume:

```text
DomainContext
StudentSnapshot
chat history
product/session context
```

Then produce an ephemeral:

```text
TutorPlan
```

TutorPlan should not be stored as a dedicated first-class table in V1.

## 6.4 Extractor Writes

Extractor should write structured evidence into:

```text
app.learning_events
```

Later, if needed, it may also write:

```text
app.user_context_events
```

## 6.5 Projectors Write

Projectors should update:

```text
app.user_topic_state
app.user_learning_item_state
```

Potential later projector target:

```text
app.user_profile_summary
```

---

# 7. Recommended Implementation Order

```text
Phase 1:
  Create/port app.pt_areas, app.pt_topics, app.pt_concept_relationships.

Phase 2:
  Create app.learning_events and app.user_topic_state.

Phase 3:
  Create app.learning_items and app.user_learning_item_state.

Phase 4:
  Backfill app.learning_items from catalog.coding_problems first.

Phase 5:
  Build DomainService read APIs.

Phase 6:
  Build StudentModelService read APIs.

Phase 7:
  Build TutorOrchestratorService and TutorPlan generation.

Phase 8:
  Update chat pipeline:
    user message
      -> DomainContext
      -> StudentSnapshot
      -> TutorPlan
      -> LLM response

Phase 9:
  Add extractor and projectors.

Phase 10:
  Run in shadow mode before changing live tutor behavior.
```

---

# 8. Final Recommendation

Use a 7-table tutor foundation on top of the 14 existing PostgreSQL tables.

The core architecture should be:

```text
Domain graph:
  app.pt_areas
  app.pt_topics
  app.pt_concept_relationships

Evidence ledger:
  app.learning_events

Current student state:
  app.user_topic_state
  app.user_learning_item_state

Universal practice/content bridge:
  app.learning_items
```

This gives enough structure for:

- multi-domain tutoring,
- prerequisite reasoning,
- user progress tracking,
- learning event logs,
- DSA/coding evidence integration,
- system design cases,
- SQL exercises,
- diagnostics,
- spaced review,
- assisted vs independent solving,
- future domain expansion,

without overengineering the database into 10-20 specialized tables.

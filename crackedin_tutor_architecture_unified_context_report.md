# Crackedin Labs Tutor Architecture Unified Context Report

**Project:** `crackedinlabs/interview-prep-ai`  
**Date:** 2026-06-20  
**Purpose:** Single consolidated source-of-truth report for the adaptive tutor architecture, current PostgreSQL runtime tables, recommended new tutor tables, service boundaries, runtime flow, and implementation direction.

---

## 0. Executive Summary

We are upgrading the current Crackedin Labs chat experience into a durable, adaptive, multi-domain AI tutor.

The tutor should not behave like a generic chatbot that only answers the current prompt. It should answer using:

1. the user's current message,
2. the domain and topic involved,
3. the prerequisites required for that topic,
4. the user's known learning state,
5. the user's solved/attempted/reviewed/struggled/verified history,
6. global curriculum and content context,
7. coding-platform evidence such as LeetCode history and submissions,
8. future evidence from chat, extensions, labs, checkpoints, and interview practice.

The core architecture is:

```text
Domain Layer
  +
Student Model Layer
  +
Tutor Orchestrator / Tutor Administrator Layer
  -> LLM response generation
  -> Extractor writes events
  -> Projectors update durable state
```

The key system shift is:

```text
Current: User message -> LLM decides tools/direct answer -> LLM answers

Target:  User message -> DomainContext + StudentSnapshot -> TutorPlan -> LLM answers -> events -> projections
```

PostgreSQL is the production runtime source of truth. SQLite local tables should be discarded as runtime dependencies and used only as seed/migration input where useful.

The current PostgreSQL database already contains identity, chat, prep-plan shell, coding catalog/sync, code blob, logs, and audit tables. It does not yet contain the adaptive tutor's durable domain graph, learning events, topic-state projections, universal learning items, or user learning-item state.

The recommended V1 addition is exactly **7 new PostgreSQL tables**:

```text
app.pt_areas
app.pt_topics
app.pt_concept_relationships
app.learning_events
app.user_topic_state
app.learning_items
app.user_learning_item_state
```

Do not create 10-20 specialized tables in V1. Use a compact model built around domain graph tables, append-only events, projection tables, and universal learning items.

---

## 1. Source-of-Truth Database Decision

### 1.1 PostgreSQL Is Runtime Source of Truth

The production tutor should read and write PostgreSQL only.

PostgreSQL should own:

- user identity,
- chat history,
- prep plans,
- coding evidence,
- domain graph,
- learning events,
- user topic state,
- universal learning items,
- user learning-item state.

### 1.2 SQLite Is Not Runtime

SQLite currently contains richer curriculum/knowledge tables, but it should not remain a runtime dependency.

SQLite can be used for:

- seeding areas,
- seeding topics,
- seeding concept relationships,
- migrating system design cases into learning items,
- migrating checkpoints/questions into learning items,
- migrating knowledge sources/chunks later.

SQLite should not be queried by the live tutor pipeline.

### 1.3 Do Not Mirror SQLite Table-for-Table

SQLite contains many tables such as:

```text
pt_subtopics
pt_learning_paths
pt_checkpoints
pt_checkpoint_questions
pt_system_design_cases
knowledge_sources
knowledge_chunks
knowledge_chunks_vec
knowledge_chunks_fts
user_memory
user_topic_log
user_attempt_log
user_weakness_signal
user_revisit_queue
```

Do not recreate all of these in PostgreSQL for V1.

Most of their behavior can be covered by:

```text
app.pt_topics
app.pt_concept_relationships
app.learning_events
app.user_topic_state
app.learning_items
app.user_learning_item_state
metadata_json fields
existing app.user_prep_plans
existing app.user_prep_task_progress
```

---

## 2. Latest PostgreSQL Table Inventory

This section combines:

1. tables already present in the current PostgreSQL database,
2. the 7 new recommended tutor architecture tables.

Together, these form the working table reference for the tutor architecture.

---

# 2A. Existing PostgreSQL Tables

## 2A.1 Identity

| Schema | Table | Status | Role in Tutor Architecture |
|---|---|---:|---|
| `public` | `users` | Existing | Canonical user identity. All user-specific tutor state should reference this table. |

### Notes

Do not create a separate tutor-specific user table.

`public.users` should anchor:

- chat sessions,
- prep plans,
- learning events,
- topic state,
- item state,
- coding data,
- future user profile summaries.

---

## 2A.2 Chat Tables

| Schema | Table | Status | Role in Tutor Architecture |
|---|---|---:|---|
| `app` | `chat_sessions` | Existing | Groups conversation sessions. Used for tutor session context. |
| `app` | `chat_messages` | Existing | Stores user/assistant messages. Raw conversation record and evidence reference. |
| `app` | `chat_feedback` | Existing | Stores message feedback. Can affect response quality signals and extraction confidence. |

### Notes

Chat tables remain the raw conversation source.

They should not be overloaded as the only learning progress store.

Recommended separation:

```text
chat_messages = raw conversation
learning_events = structured learning evidence
user_topic_state = current topic-level projection
user_learning_item_state = current item-level projection
```

Do not create another generic chat-log table.

---

## 2A.3 Application Logs and Audit

| Schema | Table | Status | Role in Tutor Architecture |
|---|---|---:|---|
| `app` | `api_logs` | Existing | Product/API logs and operational analytics. Can temporarily store lightweight tutor traces. |
| `app` | `uc_forget_audit` | Existing | Forget/delete audit trail. Important for user-data lifecycle and privacy. |

### Notes

`api_logs` can help during rollout/debugging but should not become the source of truth for learner state.

Do not create `app.tutor_plan_logs` in V1 unless debugging/analytics requirements become strong enough.

---

## 2A.4 Prep Plan Tables

| Schema | Table | Status | Role in Tutor Architecture |
|---|---|---:|---|
| `app` | `user_prep_plans` | Existing | Durable roadmap/prep-plan shell. Used only for explicit plan flows. |
| `app` | `user_prep_task_progress` | Existing | Tracks progress against durable prep-plan tasks. |
| `app` | `user_prep_plan_feedback` | Existing | Stores feedback on prep plans. |

### TutorPlan vs Prep Plan

| Concept | Scope | Stored as First-Class Table? | Example |
|---|---|---:|---|
| TutorPlan | One turn or short interaction | No, not in V1 | “For this answer, use scaffolded explanation and ask one diagnostic.” |
| Prep Plan | Multi-day or multi-week roadmap | Yes, existing prep-plan tables | “Prepare for backend interviews over 45 days.” |

### When to Create or Update `user_prep_plans`

Create/update durable prep plans only when:

- the user explicitly asks for a plan,
- onboarding/calibration creates one,
- the user enters formal prep mode,
- the user changes target role/company/date,
- the product intentionally reorients their roadmap.

Do not create durable prep plans for ordinary concept questions like “What is deadlock?”

---

## 2A.5 Coding Catalog and User Coding Data

| Schema | Table | Status | Role in Tutor Architecture |
|---|---|---:|---|
| `catalog` | `coding_problems` | Existing | Global coding problem catalog. Source for DSA/coding `learning_items`. |
| `user_data` | `user_coding_profiles` | Existing | User's connected coding-platform profile. Useful platform-level context. |
| `user_data` | `user_coding_problems` | Existing | Per-user problem aggregate status. Useful as coding progress evidence. |
| `user_data` | `user_coding_submissions` | Existing | Per-user coding submissions/verdicts. Strong evidence for DSA progress. |
| `code_blobs` | `user_coding_submission_code` | Existing | Stores submitted code. Useful for code review/debugging with privacy controls. |

### Coding Evidence Mapping

| Evidence | Tutor Interpretation |
|---|---|
| Accepted submission with no/low help | `solved_independently` or `verified` item state. |
| Accepted submission after hints/full solution | `solved_with_help`, not fully verified. |
| Wrong answer | `attempted` or `struggled`. |
| TLE | Possible complexity-analysis weakness. |
| Runtime error | Possible implementation/debugging weakness. |
| Repeated failures | Strong weakness signal via `learning_events`. |

`catalog.coding_problems` should be backfilled into `app.learning_items` with `item_type = 'coding_problem'`.

---

# 2B. Recommended New PostgreSQL Tables

These are the 7 required new tables for the adaptive tutor foundation.

## 2B.0 Summary

| # | Schema | Table | Layer | Required for V1? | Short Why |
|---:|---|---|---|---:|---|
| 1 | `app` | `pt_areas` | Domain Layer | Yes | Canonical high-level domains. |
| 2 | `app` | `pt_topics` | Domain Layer | Yes | Canonical topics/subtopics/modules. |
| 3 | `app` | `pt_concept_relationships` | Domain Layer | Yes | Prerequisites, related concepts, unlocks. |
| 4 | `app` | `learning_events` | Student/Event Layer | Yes | Append-only learning evidence ledger. |
| 5 | `app` | `user_topic_state` | Student Projection | Yes | Fast current user-topic readiness/mastery state. |
| 6 | `app` | `learning_items` | Domain + Student Bridge | Yes | Universal catalog of things to learn/practice/verify. |
| 7 | `app` | `user_learning_item_state` | Student Projection | Yes | Per-user state for each learning/practice item. |

---

## 2B.1 `app.pt_areas`

### Purpose

Stores high-level learning domains.

Examples:

```text
dsa
system_design
operating_systems
computer_networks
databases
sql
data_engineering
backend
cloud
distributed_systems
debugging
tools
```

### Why Needed

The tutor needs stable domain identifiers before it can reason about topics, prerequisites, content, and progress.

Without this table, domains would be hardcoded or inferred inconsistently.

### Example Uses

| Query | Area |
|---|---|
| “What is deadlock?” | `operating_systems` |
| “Explain TCP vs UDP.” | `computer_networks` |
| “Design a rate limiter.” | `system_design` |
| “Give me Kafka questions.” | `data_engineering` |
| “Review my SQL query.” | `sql` |

### Conceptual Fields

Not final DDL.

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

## 2B.2 `app.pt_topics`

### Purpose

Stores canonical topics, subtopics, modules, concepts, cases, and skills.

Examples:

```text
operating_systems.deadlock
operating_systems.process
operating_systems.thread
system_design.rate_limiter
databases.indexes
sql.window_functions
data_engineering.kafka_consumer_groups
```

### Why Needed

Free-text topic inference is not enough. The tutor needs stable topic IDs for:

- progress tracking,
- prerequisite checks,
- learning events,
- learning items,
- review scheduling,
- analytics,
- TutorPlan generation.

### Important Design Decision

Do not create `app.pt_subtopics` in V1.

Represent subtopics using the same table:

```text
parent_topic_id
topic_type
area_slug
```

Example hierarchy:

```text
Operating Systems
  -> Synchronization
      -> Deadlock
          -> Four Coffman Conditions
          -> Deadlock Prevention
          -> Deadlock Detection
```

### Suggested `topic_type` Values

```text
area_module
module
concept
subtopic
case
checkpoint
skill
```

### Conceptual Fields

```text
id
area_slug
slug
title
description
parent_topic_id
topic_type
difficulty
is_active
metadata_json
created_at
updated_at
```

---

## 2B.3 `app.pt_concept_relationships`

### Purpose

Stores prerequisite and concept graph relationships.

Examples:

```text
deadlock requires process
deadlock requires thread
deadlock requires lock
deadlock requires resource allocation
consistent_hashing requires hashing
kafka_consumer_groups requires partitioning
transactions requires isolation_levels
```

### Why Needed

This table lets DomainService and TutorOrchestrator answer:

- What prerequisites does this topic require?
- What topics does this unlock?
- What related concepts should be included?
- What missing foundation should be repaired first?
- What concepts are often confused?

### Minimal Relationship Types

Start with:

```text
requires
related_to
```

Later add:

```text
unlocks
part_of
often_confused_with
contrasts_with
```

### Conceptual Fields

```text
id
source_topic_id
target_topic_id
relationship_type
weight
metadata_json
created_at
updated_at
```

---

## 2B.4 `app.learning_events`

### Purpose

Append-only ledger of structured learning evidence.

This is the production replacement for several local/legacy SQLite-style logs:

```text
user_topic_log
user_attempt_log
user_weakness_signal
user_revisit_queue
```

### Why Needed

The tutor needs historical evidence, not just current state.

Bad design:

```text
user_topic_state.status = verified
```

with no evidence trail.

Better design:

```text
event: taught deadlock
event: user self-reported knowing locks
event: diagnostic_failed on circular wait
event: diagnostic_passed on deadlock example
projection: user_topic_state updated from those events
```

### Event Types

Recommended V1 event types:

```text
seen
taught
covered
self_reported_known
attempted
struggled
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

### Evidence Strength Rule

Not all events have equal weight.

| Event | Evidence Strength |
|---|---|
| `seen` | Very weak. Exposure only. |
| `taught` | Weak/medium. Tutor covered the topic, but user did not prove mastery. |
| `self_reported_known` | Low confidence. Never mark verified from self-report alone. |
| `diagnostic_passed` | Medium/high confidence depending on question quality. |
| `checkpoint_passed` | High confidence if rubric is strong. |
| `coding_submission_accepted` | Strong for coding item state, subject to help level. |

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

### Important Rule

This table should be append-only. If extraction/projector logic changes, replay events or add correction/invalidation events instead of silently mutating history.

---

## 2B.5 `app.user_topic_state`

### Purpose

Current user-specific topic-level projection.

It answers quickly:

```text
What is this user's current state for this topic?
```

### Why Needed

The tutor cannot recompute readiness from all events on every chat turn.

Runtime chat needs low-latency reads like:

- Has this user seen deadlock?
- Are prerequisites verified or unknown?
- Is this topic due for review?
- Has the user struggled with recursion?
- What misconception signals exist?

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

### Projection Rule

`user_topic_state` is derived from:

```text
learning_events
coding evidence
checkpoint evidence
manual/admin corrections if needed
```

It should not be manually treated as the only source of truth.

---

## 2B.6 `app.learning_items`

### Purpose

Universal catalog of things a user can learn, practice, solve, read, review, or be assessed on.

A learning item can be:

```text
coding problem
system design case
SQL exercise
debugging task
OS diagnostic
networking scenario
data engineering lab
course lesson
interview question
checkpoint
reading item
```

### Why Needed

This table prevents domain-specific table explosion.

Do not create separate V1 tables like:

```text
user_solved_system_design_cases
user_completed_sql_exercises
user_completed_debugging_tasks
user_checkpoint_question_state
user_data_engineering_lab_state
```

Instead:

```text
learning_items.item_type = 'coding_problem'
learning_items.item_type = 'system_design_case'
learning_items.item_type = 'sql_exercise'
learning_items.item_type = 'checkpoint'
learning_items.item_type = 'debugging_task'
```

### Example Items

| Item | Item Type |
|---|---|
| LeetCode Two Sum | `coding_problem` |
| Design URL Shortener | `system_design_case` |
| SQL Window Functions Exercise | `sql_exercise` |
| Deadlock Diagnostic Question | `diagnostic_question` |
| Kafka Partitioning Lab | `project_lab` |
| Debug Docker Container Networking | `debugging_task` |

### Source Mapping

| Source | Mapping |
|---|---|
| `catalog.coding_problems` | `learning_items` with `item_type = 'coding_problem'` |
| SQLite `pt_system_design_cases` | `learning_items` with `item_type = 'system_design_case'` |
| SQLite `pt_checkpoints` | `learning_items` with `item_type = 'checkpoint'` |
| SQLite `pt_checkpoint_questions` | `learning_items` with `item_type = 'diagnostic_question'` or `quiz_question` |
| Future interview corpus | `learning_items` with `item_type = 'interview_question'` |
| Manual curriculum | `learning_items` with suitable item type |

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

### Verification Types

```text
none
read
self_explain
quiz
code_acceptance
rubric_eval
mock_interview
```

---

## 2B.7 `app.user_learning_item_state`

### Purpose

Current user-specific state for each learning item.

It answers:

```text
Has this user seen, attempted, solved, verified, or forgotten this specific item?
```

### Why Needed

Topic-level state is not enough.

`user_topic_state` says:

```text
User is weak in dynamic programming.
```

`user_learning_item_state` says:

```text
User attempted Coin Change 3 times.
User solved Two Sum independently.
User solved Number of Islands after hints.
User failed Deadlock Diagnostic once.
User completed Rate Limiter design case.
```

The tutor must distinguish independent success from assisted success.

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

### Help-Level Model

Recommended help scale:

```text
0 = no help; independent
1 = small nudge
2 = hint
3 = scaffolded approach
4 = partial solution
5 = full solution / answer-level help
```

State implications:

| Outcome | State Implication |
|---|---|
| Success with help level 0-1 | `solved_independently` or `verified` depending on item type. |
| Success with help level 2-4 | `solved_with_help`; not necessarily verified. |
| Success with help level 5 | Assisted completion; not verified. |
| Repeated failures | `struggled`, schedule review/prerequisite repair. |

### Conceptual Fields

```text
user_id
learning_item_id
state
best_outcome
best_score
last_score
attempt_count
independent_success_count
assisted_success_count
failure_count
hint_count
first_seen_at
first_attempted_at
first_solved_at
last_attempted_at
last_solved_at
last_verified_at
next_review_at
last_help_level
misconception_signals_json
evidence_json
created_at
updated_at
```

---

## 3. Optional Later Tables

These are useful, but not required for the first migration.

## 3.1 `app.user_context_events`

### Create Now?

Not required for V1.

### Why Useful Later?

Learning mastery and user context are different.

Learning evidence:

```text
User struggled with deadlock.
User solved Two Sum independently.
```

User context:

```text
User is preparing for backend interviews.
User prefers concise explanations.
User targets L5.
User wants system design emphasis.
```

Use `user_context_events` later to separate durable user facts/preferences/goals from learning evidence.

## 3.2 `app.user_profile_summary`

### Create Now?

Not required for V1, but likely useful after context events exist.

### Why Useful Later?

The tutor should not scan all chat history every request.

A profile summary can store:

```text
target_role
target_level
preferred_style
known_domains
weak_domains
recent_focus
active_prep_goal
updated_at
```

---

## 4. Tables Not Recommended for V1

Do not create these now unless there is a concrete product requirement.

| Deferred Table | Why Deferred / V1 Alternative |
|---|---|
| `app.pt_subtopics` | Use `app.pt_topics.parent_topic_id` and `topic_type`. |
| `app.pt_learning_paths` | Use concept relationships + existing prep-plan tables. |
| `app.pt_checkpoints` | Represent checkpoints as `learning_items.item_type = 'checkpoint'`. |
| `app.pt_checkpoint_questions` | Represent questions as `learning_items.item_type = 'diagnostic_question'` or `quiz_question`. |
| `app.pt_system_design_cases` | Represent cases as `learning_items.item_type = 'system_design_case'`. |
| `app.user_weakness_signal` | Use `learning_events` + `user_topic_state.misconception_signals_json`. |
| `app.user_revisit_queue` | Use `next_review_at` on topic/item state. |
| `app.user_attempt_log` | Use `learning_events` + item/topic attempt counters. |
| `app.tutor_plan_logs` | Keep TutorPlan ephemeral; use chat/API trace fields if needed. |
| `app.user_misconceptions` | Store in event/state JSON first. Promote later if analytics require it. |
| `app.assessment_rubrics` | Store rubric metadata in `learning_items.metadata_json` first. |
| `app.learning_item_topic_map` | Use `topic_ids_json` first. Add join table later only if query complexity demands it. |

---

## 5. Architecture Overview

## 5.1 Domain Layer

### Purpose

The Domain Layer is the tutor's structured understanding of what can be learned.

It answers:

- What domain does this question belong to?
- What topic is the user asking about?
- What prerequisites does this topic require?
- What subtopics are inside this topic?
- What examples, questions, cases, or content support this topic?
- What should be taught before or after this topic?
- What learning items can be used to teach, practice, or verify the topic?

### Tables Used

```text
app.pt_areas
app.pt_topics
app.pt_concept_relationships
app.learning_items
catalog.coding_problems
future app.knowledge_sources / app.knowledge_chunks if migrated
```

### Supported Domains

The design is not DSA-specific. It should support:

```text
DSA
System Design
Low-Level Design
Operating Systems
Computer Networks
Databases
SQL
Data Engineering
Backend Engineering
Distributed Systems
Cloud / Infrastructure
Debugging
DevOps / Tools
Future courses and labs
```

---

## 5.2 Student Model Layer

### Purpose

The Student Model Layer is the tutor's user-specific learning state and evidence layer.

It answers:

- Has this user seen this topic before?
- Has the user practiced it?
- Has the user demonstrated mastery?
- Has the user struggled with prerequisites?
- Is this topic stale or due for revision?
- What has the user solved independently vs with help?
- What does the tutor know with confidence vs low confidence?

### Tables Used

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
code_blobs.user_coding_submission_code
```

### Key Rule

Do not treat exposure as mastery.

```text
seen != understood
taught != verified
self_reported_known != verified
solved_with_full_solution != solved_independently
```

---

## 5.3 Tutor Orchestrator / Tutor Administrator Layer

### Purpose

The Tutor Orchestrator is the decision-making layer.

It answers:

- What is the user trying to do?
- What topic/domain is involved?
- Is the user ready for the requested topic?
- Should the answer be direct, scaffolded, diagnostic, Socratic, or practice-based?
- What context should the LLM receive?
- What should be extracted and updated after the response?

### Key Principle

The LLM should not be the only decision-maker.

The LLM can help with:

- intent classification,
- topic mapping,
- natural-language explanation,
- diagnostic question generation,
- summarizing retrieved context.

The platform should own:

- durable mastery updates,
- readiness scores,
- prerequisite truth,
- verified/unverified status,
- long-term learner state,
- event projection logic.

---

## 6. Runtime Chat Pipeline

## 6.1 Current Pipeline

The current pipeline is LLM-led:

```text
User message
  -> prompt + history + tools
  -> LLM decides whether to call tools or answer directly
  -> optional tool calls
  -> LLM final response
  -> chat persistence
```

This can answer questions, but it does not reliably know:

- whether prerequisites are known,
- whether the user has struggled before,
- whether the topic is due for review,
- whether the answer should be direct or scaffolded,
- whether a diagnostic should be asked,
- whether mastery should be updated.

## 6.2 Target Pipeline

The target pipeline is orchestrator-led:

```text
User message
  -> Intent/topic/domain resolution
  -> DomainService fetches DomainContext
  -> StudentModelService fetches StudentSnapshot
  -> TutorOrchestrator builds TutorPlan
  -> LLM writes final response using TutorPlan
  -> Extractor writes learning/context events
  -> Projectors update durable state
  -> Next turn uses updated state
```

### Detailed Flow

1. User sends message.
2. System identifies intent, domain, topic, and ambiguity.
3. DomainService retrieves topic graph, prerequisites, related topics, learning items, and content.
4. StudentModelService retrieves user topic state, item state, recent events, coding evidence, and prep-plan context.
5. TutorOrchestrator builds a validated TutorPlan.
6. LLM receives TutorPlan, DomainContext, StudentSnapshot, selected context, and chat history.
7. LLM answers according to the plan.
8. Extractor emits structured events.
9. Projectors update topic/item state.
10. Next turn becomes more personalized.

---

## 7. DomainService

## 7.1 Purpose

DomainService handles global curriculum and content intelligence.

It answers:

- What topic is this?
- What domain does it belong to?
- What prerequisites does it require?
- What content can explain it?
- What learning items can practice or verify it?
- What interview/corpus evidence is relevant?

## 7.2 Inputs

```text
user_message
candidate_domain
candidate_topic
target_role_or_level optional
retrieval_mode optional
```

## 7.3 Outputs: DomainContext

```text
domain
target_topic
target_topic_id
resolved_confidence
ambiguity_candidates
prerequisites
related_topics
subtopics
learning_items
knowledge_context optional
interview_context optional
```

## 7.4 Core Responsibilities

```text
Resolve domain
Resolve topic
Detect ambiguity
Fetch prerequisites
Fetch related topics
Fetch topic hierarchy
Fetch candidate learning items
Fetch content/RAG context if available
Fetch interview evidence if available
Recommend adjacent/next topics
```

## 7.5 Example: “What is deadlock?”

```text
domain = operating_systems
target_topic = deadlock
prerequisites = process, thread, lock, resource allocation, blocking wait, synchronization
subtopics = Coffman conditions, prevention, avoidance, detection, recovery
learning_items = deadlock concept, diagnostic question, checkpoint, lock-ordering debugging task
```

---

## 8. StudentModelService

## 8.1 Purpose

StudentModelService handles user-specific learner state.

It answers:

- What does this user know?
- What has this user attempted?
- What did they solve independently?
- What did they solve with help?
- What is weak or stale?
- What prerequisites are unknown or weak?
- What is due for review?

## 8.2 Inputs

```text
user_id
target_topic
prerequisite_topics
candidate_learning_items
session_id
```

## 8.3 Outputs: StudentSnapshot

```text
user_id
target_topic_state
prerequisite_states
relevant_item_states
recent_learning_events
coding_evidence_summary
prep_plan_context
known_strengths
known_weaknesses
misconception_signals
preferred_style optional
confidence
```

## 8.4 Core Responsibilities

```text
Fetch topic state for target and prerequisites
Fetch item states for candidate learning items
Fetch recent events
Fetch coding evidence
Estimate prerequisite readiness
Build compact StudentSnapshot
Record learning events after extraction
Trigger or run projections
```

## 8.5 Cold Start Rule

If no state exists:

```text
target_topic_state = unknown
prerequisite_state = unknown
evidence_count = 0
confidence = low
```

The tutor should not assume knowledge, but also should not block the user with a long quiz.

Good cold-start strategies:

- scaffolded explanation,
- one lightweight prerequisite probe,
- ask whether they want prerequisite review,
- continue with a short bridge explanation.

---

## 9. TutorOrchestratorService / Tutor Administrator

## 9.1 Purpose

The orchestrator makes the pedagogical decision before the LLM writes the answer.

It combines:

```text
user message
intent/domain/topic classification
DomainContext
StudentSnapshot
chat context
product/session flags
prep-plan context if relevant
```

It outputs:

```text
TutorPlan
response contract
retrieval contract
extraction contract
```

## 9.2 TutorPlan Is Ephemeral

TutorPlan is per-turn or short-session.

It is not a durable prep plan.

Do not create a first-class `tutor_plan_logs` table in V1.

Store lightweight traces in existing chat/API logs if needed during rollout.

## 9.3 TutorPlan Fields

```text
intent
strategy
domain
target_topic
target_topic_id
required_prerequisites
student_state_summary
domain_context_summary
learning_items_to_reference
content_context_to_reference
response_contract
state_update_expectations
extractor_hints
```

---

## 10. Intent and Strategy Model

## 10.1 Recommended V1 Intent Enum

Intent answers: “What is the user trying to do?”

```text
concept_explanation
prerequisite_check
diagnostic_answer
practice_request
problem_solving
code_review_or_debugging
system_design_case
comparison_or_tradeoff
roadmap_or_prep_plan
progress_or_readiness_check
revision_request
resource_recommendation
interview_intelligence
off_topic_or_meta
```

Examples:

| User Message | Intent | Domain | Topic |
|---|---|---|---|
| “What is deadlock?” | `concept_explanation` | `operating_systems` | `deadlock` |
| “TCP vs UDP?” | `comparison_or_tradeoff` | `computer_networks` | `tcp_udp` |
| “Design a rate limiter.” | `system_design_case` | `system_design` | `rate_limiter` |
| “Give me Spark questions.” | `practice_request` | `data_engineering` | `spark` |
| “Make me a 30-day plan.” | `roadmap_or_prep_plan` | varies | varies |

Intent should not encode domain. Domain and topic are separate fields.

## 10.2 Recommended V1 Strategy Enum

Strategy answers: “How should the tutor respond?”

```text
direct_explanation
scaffolded_explanation
prerequisite_probe
prerequisite_repair
socratic_guidance
guided_practice
misconception_correction
retrieval_grounded_answer
assessment_checkpoint
revision_or_spaced_repetition
prep_plan_generation
```

Examples:

| Situation | Strategy |
|---|---|
| Prerequisites verified | `direct_explanation` |
| Unknown user state | `scaffolded_explanation` or `prerequisite_probe` |
| Missing prerequisite | `prerequisite_repair` |
| User solving a problem | `socratic_guidance` |
| User asks for practice | `guided_practice` |
| User has wrong mental model | `misconception_correction` |
| Topic is stale | `revision_or_spaced_repetition` |
| User asks for roadmap | `prep_plan_generation` |

---

## 11. Extractor and Projectors

## 11.1 Extractor

The extractor reads:

```text
user message
assistant response
TutorPlan
DomainContext
StudentSnapshot
possibly user follow-up
```

It writes structured events to `app.learning_events`.

Potential extracted events:

```text
taught
seen
covered
self_reported_known
diagnostic_passed
diagnostic_failed
attempted
struggled
reviewed
verified
checkpoint_passed
checkpoint_failed
```

The extractor should distinguish:

- what the tutor taught,
- what the user demonstrated,
- what the user merely claimed,
- what remains unknown.

## 11.2 Projectors

Projectors consume events and update projections.

| Projector | Reads | Writes |
|---|---|---|
| Topic Progress Projector | `learning_events`, coding evidence, checkpoint events | `user_topic_state` |
| Item State Projector | `learning_events`, coding submissions, item interactions | `user_learning_item_state` |
| Prep Plan Projector | learning/item events | `user_prep_task_progress` |
| Future Profile Projector | `user_context_events`, feedback, chat signals | `user_profile_summary` |

## 11.3 Event + Projection Rule

Use both logs/events and hard projection tables.

```text
learning_events = what happened
user_topic_state = current topic state
user_learning_item_state = current item state
```

Do not store everything only as hard state. That loses evidence.

Do not store everything only as logs. That hurts chat latency and complicates runtime.

Best architecture:

```text
append-only events -> projectors -> current state tables
```

---

## 12. End-to-End Example: “What is Deadlock?”

## 12.1 User Message

```text
What is deadlock?
```

## 12.2 Intent/Topic Resolution

```text
intent = concept_explanation
domain = operating_systems
topic = deadlock
confidence = high
```

## 12.3 DomainService Output

```text
target_topic = deadlock
prerequisites = process, thread, resource, lock, blocking wait, synchronization
subtopics = four Coffman conditions, prevention, avoidance, detection, recovery
learning_items = concept item, quick diagnostic, checkpoint, lock-ordering debugging task
```

## 12.4 StudentModelService Output

If no state exists:

```text
target_topic_state = unknown
prerequisite_state = unknown
evidence_count = 0
confidence = low
```

## 12.5 TutorPlan

```text
intent = concept_explanation
strategy = scaffolded_explanation or prerequisite_probe
response_contract = ask at most one lightweight check or include a short prerequisite bridge before explaining
state_update_expectations = capture taught/seen/self-report/diagnostic evidence
create_durable_prep_plan = false
```

## 12.6 LLM Response

A good answer might:

1. briefly bridge process/resource/wait concepts,
2. explain deadlock simply,
3. give a concrete lock example,
4. ask one diagnostic question.

## 12.7 State Update

If user says:

```text
Yes, I know processes and locks.
```

Write low-confidence events:

```text
event_type = self_reported_known
topics = process, lock
confidence = low
```

Do not mark those topics verified.

If user answers diagnostic correctly:

```text
event_type = diagnostic_passed
topic = deadlock
confidence = medium/high
```

Then projector updates `user_topic_state`.

---

## 13. Implementation Order

## Phase 0: Confirm Source of Truth

Use actual PostgreSQL table list as runtime source of truth.

Treat SQLite only as seed/migration input.

## Phase 1: Port/Create Minimal Domain Graph

Create/port:

```text
app.pt_areas
app.pt_topics
app.pt_concept_relationships
```

Seed from SQLite where useful.

Do not create `pt_subtopics` separately.

## Phase 2: Create Student Event/State Tables

Create:

```text
app.learning_events
app.user_topic_state
```

## Phase 3: Add Universal Learning Tables

Create:

```text
app.learning_items
app.user_learning_item_state
```

Backfill from:

```text
catalog.coding_problems
SQLite pt_system_design_cases if useful
SQLite pt_checkpoints/questions if useful
future interview/corpus sources
manual curriculum
```

## Phase 4: Build DomainService

Implement:

```text
resolve topic
fetch prerequisites
fetch related topics
fetch learning items
fetch content/context if available
```

## Phase 5: Build StudentModelService

Implement:

```text
fetch topic state
fetch prerequisite readiness
fetch item state
fetch recent events
fetch coding evidence
build compact StudentSnapshot
```

## Phase 6: Build TutorOrchestratorService

Implement:

```text
classify intent/domain/topic
build TutorPlan
choose strategy
validate response contract
prepare LLM input
```

## Phase 7: Update Chat Pipeline

Replace:

```text
LLM decides tools/direct answer first
```

with:

```text
orchestrator builds plan first
LLM writes answer second
```

## Phase 8: Add Extractor and Projectors

Add:

```text
learning event extractor
topic-state projector
item-state projector
prep-plan task projector if needed
```

## Phase 9: Shadow Rollout

Use rollout mode:

```text
off
shadow
on
```

In shadow mode:

- generate TutorPlans,
- write learning events,
- update projections,
- compare behavior,
- do not yet alter user-facing response strategy unless enabled.

---

## 14. Minimal Build Checklist

| Work Item | Required for MVP? |
|---|---:|
| Port/create `app.pt_areas` | Yes |
| Port/create `app.pt_topics` | Yes |
| Port/create `app.pt_concept_relationships` | Yes |
| Add `app.learning_events` | Yes |
| Add `app.user_topic_state` | Yes |
| Add `app.learning_items` | Yes |
| Add `app.user_learning_item_state` | Yes |
| Backfill coding problems into learning items | Yes |
| Build DomainService | Yes |
| Build StudentModelService | Yes |
| Build TutorOrchestratorService | Yes |
| Update chat pipeline to use TutorPlan | Yes |
| Add extractor | Yes |
| Add projectors | Yes |
| Shadow rollout | Yes |
| Add full checkpoint product | Later |
| Add misconception analytics table | Later |
| Add tutor-plan analytics table | Later |
| Add profile summary/context events | Later |
| Add full RAG migration | Later unless required for V1 |

---

## 15. Naming Decision

The latest recommended names are cleaner production names:

```text
app.learning_events
app.user_topic_state
```

instead of:

```text
app.pt_learning_events
app.pt_user_topic_progress
```

Reason:

- shorter,
- more general,
- less tied to old `pt_*` naming,
- clearer as production service tables.

However, if repo migrations or backward compatibility already depend on `pt_*` names, either:

1. use `pt_*` names consistently, or
2. create aliases/views during migration.

Do not mix both names casually.

---

## 16. Product and Architecture Principles

1. Avoid overengineering.
2. Do not create 10-20 new tables if 7 can cover the need.
3. Do not make the model DSA-specific.
4. Treat SQLite as seed/source, not production runtime.
5. Use PostgreSQL as production source of truth.
6. Use append-only events plus projection tables.
7. Keep TutorPlan ephemeral.
8. Keep durable prep plans separate.
9. Treat self-report as low-confidence evidence.
10. Do not treat exposure as mastery.
11. Distinguish independent solving from assisted solving.
12. Let orchestrator decide teaching strategy before LLM writes.
13. Do not let the LLM alone update mastery/readiness.
14. Add specialized tables later only when product requirements force them.
15. Prefer JSON metadata for early flexibility, but promote to normalized tables when querying/reporting demands it.

---

## 17. Compact Next-Chat Handoff

Use this summary to continue in a new chat:

```text
We are designing a multi-domain adaptive AI tutor for Crackedin Labs.

Current PostgreSQL already has:
- public.users
- app.chat_sessions
- app.chat_messages
- app.chat_feedback
- app.api_logs
- app.uc_forget_audit
- app.user_prep_plans
- app.user_prep_task_progress
- app.user_prep_plan_feedback
- catalog.coding_problems
- user_data.user_coding_profiles
- user_data.user_coding_problems
- user_data.user_coding_submissions
- code_blobs.user_coding_submission_code

SQLite has richer local curriculum/knowledge tables, but SQLite is not runtime source of truth.
It should be discarded for production runtime and used only as seed/migration input.

The recommended new PostgreSQL tables are exactly 7:
1. app.pt_areas
2. app.pt_topics
3. app.pt_concept_relationships
4. app.learning_events
5. app.user_topic_state
6. app.learning_items
7. app.user_learning_item_state

The architecture is:
User message -> intent/topic/domain resolution -> DomainService -> StudentModelService -> TutorOrchestrator builds TutorPlan -> LLM writes response -> Extractor writes learning_events -> Projectors update user_topic_state and user_learning_item_state.

TutorPlan is ephemeral and should not be stored as a first-class table in V1.
Prep plans are durable and should reuse existing app.user_prep_plans and app.user_prep_task_progress.

The system should support DSA, system design, OS, networks, DBs, SQL, data engineering, backend, distributed systems, cloud, debugging, tools, and future domains.

Do not create specialized V1 tables such as pt_subtopics, pt_learning_paths, pt_checkpoints, pt_checkpoint_questions, pt_system_design_cases, user_weakness_signal, user_revisit_queue, user_attempt_log, tutor_plan_logs, or user_misconceptions.
Represent these initially through pt_topics, concept relationships, learning_items, learning_events, projection tables, and metadata_json.

Self-report is low-confidence evidence.
Exposure is not mastery.
Independent solving and assisted solving must be distinguished using help_level.
```

---

## 18. Recommended Immediate Next Step

Create rough schema designs for the 7 new tables, but still avoid final DDL until these questions are answered:

1. Should table names use clean names (`learning_events`, `user_topic_state`) or repo-compatible names (`pt_learning_events`, `pt_user_topic_progress`)?
2. What are the exact user ID types and FK constraints in current PostgreSQL?
3. What columns already exist in current chat/tool-call logs that can hold TutorPlan traces?
4. Which SQLite tables should be used only as seed input for Phase 1?
5. What is the minimum first domain seed: DSA only, or DSA + OS + DB + System Design?
6. Should `learning_items.topic_ids_json` be enough for V1, or do we expect complex topic-item querying soon?
7. What is the rollout mode implementation: feature flag, tenant flag, user flag, or environment flag?


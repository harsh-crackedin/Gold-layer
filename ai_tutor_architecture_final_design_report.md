# AI Tutor Architecture Final Design Report

**Project:** Crackedin Labs AI Tutor / Creator Platform  
**Scope:** Skill Graph, Learner Skill Graph, and Tutor Orchestrator Layer  
**Runtime database decision:** PostgreSQL only  
**Repository status:** historical/outdated implementation; not authoritative for this design  
**Excluded for this phase:** production-support tables, data-population workflows, content-ingestion workflows, compliance, moderation, and safety layers

---

## 1. Executive Decision

The final architecture should be an **orchestrator-led adaptive tutoring system**, not a generic chatbot with tools. The core runtime path is:

```text
User request
  -> Intent + skill + domain resolution
  -> DomainService builds DomainContext
  -> StudentModelService builds StudentSnapshot
  -> UserMemoryService builds UserMemorySnapshot when personalization is enabled
  -> TutorOrchestrator builds ephemeral TutorPlan
  -> Retrieval layer fetches approved domain/content/practice context
  -> LLM produces response under TutorPlan contract
  -> Extractor emits structured learning/context events
  -> Projectors update current student state
```

The source documents already agree on the key shift: the old chat flow is LLM-led, while the target flow is `DomainContext + StudentSnapshot -> TutorPlan -> LLM response -> events -> projections`.
The unified report explicitly frames the tutor as using the user message, domain/skill, prerequisites, known learning state, solved/attempted/verified history, curriculum context, coding evidence, and future evidence from chat/labs/checkpoints/interview practice. fileciteturn22file2

The final design keeps the strongest source-doc ideas:

- PostgreSQL is the only production runtime database.
- Chat messages remain raw conversation history, not the learner model.
- Tutor progress is modeled as append-only learning evidence plus projected current state.
- TutorPlan is ephemeral; durable prep plans remain separate.
- Mem0/MemZero is optional semantic memory for personalization, not canonical learning truth.
- Exposure is not mastery; self-report is low-confidence; independent solving must be separated from assisted solving.

The final design **replaces the older 7-table-only recommendation with a production-ready 13-table tutor core**. The original 7 tables are still the minimum kernel, but production-quality skill resolution, item retrieval, grounded response generation, and Mem0 personalization require six additional tables that are directly part of the domain or student model, not production-support infrastructure.

Source set reconciled: PostgreSQL table reference fileciteturn19file0, unified tutor report fileciteturn19file1, and MemZero/Mem0 integration report fileciteturn19file2.

### 1.1 Naming Update: Skill Graph and Learner Skill Graph

This version replaces the legacy `pt_*` naming family with a clearer production vocabulary:

| Old naming | New naming | Meaning |
|---|---|---|
| `app.pt_areas` | `app.skill_areas` | Global high-level skill domains. |
| `app.pt_topics` | `app.skills` | Canonical skill/concept nodes in the Skill Graph. |
| `app.pt_topic_aliases` | `app.skill_aliases` | Aliases, synonyms, abbreviations, and common phrases for skill resolution. |
| `app.pt_concept_relationships` | `app.skill_relationships` | Prerequisite, related, unlock, contrast, and confusion edges between skills. |
| `app.learning_item_topics` | `app.learning_item_skills` | Many-to-many mapping between learning items and skills. |
| `app.user_topic_state` | `app.learner_skill_state` | User-specific projected state for each skill. |

Terminology rule:

```text
Skill Graph = global/shared curriculum graph.
Learner Skill Graph = user-specific evidence and projections over that graph.
```

The product UI may still use the word “topic” where it is natural for users, but the backend schema and architecture should use **skill** as the canonical unit. A skill can represent a concept, subtopic, module, case, checkpoint, pattern, technique, or practical competency.

---

## 2. Source Reconciliation

### 2.1 Areas of agreement across the sources

The source documents converge on five durable architecture principles.

First, PostgreSQL is the production runtime source of truth. The table-reference document says PostgreSQL is the runtime source of truth and SQLite should not be used by the production tutor runtime. fileciteturn22file4 The unified architecture report says the production tutor should read and write PostgreSQL only. fileciteturn22file2

Second, the tutor must move from an LLM-led flow to an orchestrator-led flow. The unified report identifies the target pipeline as skill/domain resolution, DomainService, StudentModelService, TutorOrchestrator, LLM response, extractor, and projectors. fileciteturn21file13

Third, chat tables remain raw conversation storage. The table-reference report states that chat messages should store the raw conversation while learning events and projection tables store structured progress. fileciteturn22file10

Fourth, TutorPlan and Prep Plan are different. TutorPlan is a one-turn instructional plan and should not become a first-class table in V1, while `user_prep_plans` represents durable multi-day or multi-week roadmap flows. fileciteturn22file10

Fifth, Mem0/MemZero must not become the tutor brain. The MemZero report says PostgreSQL remains the source of truth and Mem0/MemZero is a semantic personalization and user-memory retrieval layer, not an authority for mastery, readiness, verified knowledge, independent solving, attempt counts, review schedules, or curriculum truth. fileciteturn22file5

### 2.2 Contradictions and final resolution

| Issue | Source position | Final decision |
|---|---|---|
| Runtime database | Source docs already say PostgreSQL-only and no SQLite runtime. | Keep PostgreSQL-only. Do not include SQLite in runtime diagrams or implementation plans. |
| Repository | Earlier chat inspected repo, but user corrected that it is outdated. | Repository is historical only. It should not drive table names, service shape, or runtime architecture. |
| 7 tables vs production completeness | Source docs recommend 7 new tutor tables. | Keep the 7 as the minimum kernel, but add 6 direct tutor-model tables: skill aliases, item-skill mapping, knowledge sources, knowledge chunks, user context events, user profile summary. |
| Mem0 table timing | MemZero report says `user_context_events` and `user_profile_summary` are later-stage. | Promote both into the production-ready Learner Skill Graph because personalization is an explicit product requirement. |
| `learning_items.skill_ids_json` vs join table | Source docs defer `learning_item_skill_map`. | Replace JSON-only mapping with normalized `learning_item_skills`. Multi-skill item retrieval directly affects tutor quality and performance. |
| Knowledge/RAG tables | Source docs mention knowledge chunks only as future migration. | Add a minimal `knowledge_sources` + `knowledge_chunks` pair now because grounded retrieval is directly tied to response quality. Data ingestion remains out of scope. |

---

## 3. Final Architecture Principles

1. **The platform owns pedagogy; the LLM owns language.** The LLM can classify, explain, generate questions, and summarize, but the platform owns prerequisites, current state, verified status, help-level interpretation, and event projection.
2. **Domain truth is stable and canonical.** Skills, aliases, prerequisites, content chunks, and learning items live in PostgreSQL under a stable domain model.
3. **Student truth is evidence-based.** `learning_events` records what happened; `learner_skill_state` and `user_learning_item_state` summarize current state for fast reads.
4. **TutorPlan is ephemeral.** It is created per turn and used to control response strategy; it should not be persisted as a primary product table.
5. **Mem0 is a personalization accelerator, not a truth system.** Mem0 can retrieve preferences, goals, interests, and prior context. It cannot certify mastery.
6. **Verification requires performance evidence.** `seen`, `taught`, and `self_reported_known` are not enough for `verified`.
7. **Independent work and assisted work are different states.** The help-level model must be included in item-state and learning-event logic. The table reference explicitly defines help levels 0 through 5 and states that solving after the full answer should not count as independent verification. fileciteturn21file2
8. **Retrieval is a first-class tutor function.** Domain retrieval is not just “search docs.” It must retrieve skill graph, prerequisites, examples, learning items, prior student evidence, and grounded content.
9. **Scores must be interpretable before they become predictive.** V1 should use transparent readiness/coverage/confidence rules inspired by knowledge tracing, not opaque ML-only mastery predictions.

---

## 4. Final Three-Layer Architecture

```text
┌─────────────────────────────────────────────────────────────┐
│  Layer 1: Skill Graph                                      │
│  Skill areas, skills, aliases, relationships, learning      │
│  items, item-skill mapping, knowledge sources/chunks        │
└─────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│  Layer 2: Learner Skill Graph                            │
│  Learning events, learner skill state, item state, coding   │
│  evidence, prep-plan context, user context, profile summary │
└─────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│  Layer 3: Tutor Orchestrator                                 │
│  Intent/skill resolution, DomainContext, StudentSnapshot,    │
│  UserMemorySnapshot, TutorPlan, retrieval contract,          │
│  response contract, extraction contract                      │
└─────────────────────────────────────────────────────────────┘
                           │
                           ▼
                  LLM response generation
                           │
                           ▼
             events -> projectors -> next-turn state
```

The **Creator** aspect should not be a separate architecture. “Creator” is a set of intents and response contracts inside the same orchestrator: create a diagnostic question, create a practice task, create a short lesson, create a drill, create a rubric-based mock question, or create a prep-plan artifact. It uses the same DomainContext and StudentSnapshot.

---

## 5. Existing PostgreSQL Tables That Remain Unchanged

The following existing tables remain unchanged and should be reused only for their current responsibilities.

| Table | Final role in tutor architecture |
|---|---|
| `public.users` | Canonical user identity anchor. |
| `app.chat_sessions` | Conversation/session container. |
| `app.chat_messages` | Raw user/assistant message history and evidence references. |
| `app.chat_feedback` | Message-level response feedback; can inform extraction confidence and response-quality tuning. |
| `app.user_prep_plans` | Durable multi-day or multi-week roadmap/prep plans. Not TutorPlan. |
| `app.user_prep_task_progress` | Durable progress against prep-plan tasks. |
| `app.user_prep_plan_feedback` | Feedback on durable prep plans. |
| `catalog.coding_problems` | Existing coding problem catalog; referenced by `learning_items` and coding evidence interpretation. |
| `user_data.user_coding_profiles` | User coding-platform profile context. |
| `user_data.user_coding_problems` | Per-user coding problem aggregate evidence. |
| `user_data.user_coding_submissions` | Strong DSA/coding evidence. |
| `code_blobs.user_coding_submission_code` | Code-level evidence for review/debugging flows. |
| `app.api_logs` | Existing operational logs only. Not learner state. |
| `app.uc_forget_audit` | Existing audit table; unchanged and outside core tutor modeling for this phase. |

The sources explicitly distinguish raw chat from structured learning progress: `chat_messages` stores the raw conversation, `learning_events` stores structured evidence, and state projections store current progress. fileciteturn22file12

---

## 6. New Tables That Must Be Added

### 6.1 Summary

The older source recommendation was exactly 7 new tables. fileciteturn22file2 This final production design adds 13 new tables. The increase is intentional and limited: every added table directly supports domain modeling, student modeling, retrieval quality, personalization quality, or orchestrator effectiveness.

| # | Table | Layer | Why it is required |
|---:|---|---|---|
| 1 | `app.skill_areas` | Domain | Canonical high-level domains. |
| 2 | `app.skills` | Domain | Canonical concepts, skills, modules, cases, and subskills/subtopics. |
| 3 | `app.skill_aliases` | Domain | Production-grade skill resolution, synonym handling, ambiguity handling. |
| 4 | `app.skill_relationships` | Domain | Prerequisites, related concepts, unlocks, contrasts. |
| 5 | `app.learning_items` | Domain/Student bridge | Universal learn/practice/verify item catalog. |
| 6 | `app.learning_item_skills` | Domain/Student bridge | Normalized multi-skill item mapping and relevance weighting. |
| 7 | `app.knowledge_sources` | Domain/Retrieval | Canonical source metadata for grounded tutor content. |
| 8 | `app.knowledge_chunks` | Domain/Retrieval | Retrieval units for grounded explanations and examples. |
| 9 | `app.learning_events` | Student | Append-only learning evidence. |
| 10 | `app.learner_skill_state` | Student | Fast skill-level learner state projection. |
| 11 | `app.user_learning_item_state` | Student | Fast item-level learner state projection. |
| 12 | `app.user_context_events` | Student personalization | Durable preference/goal/context evidence. |
| 13 | `app.user_profile_summary` | Student personalization | Fast current personalization snapshot for orchestrator prompts. |

### 6.2 Why the final design exceeds the 7-table foundation

The 7-table Skill Graph foundation is an excellent prototype foundation, but production creates four additional needs:

1. **Skill resolution needs aliases.** Without `skill_aliases`, the system must rely too heavily on LLM inference or loose string matching for phrases like “locks,” “mutex,” “deadlock,” “transaction deadlock,” “rate limiter,” “token bucket,” and company/framework abbreviations.
2. **Learning items are many-to-many.** A coding problem, diagnostic, or system-design case often maps to multiple skills. A JSON list can work briefly, but a normalized `learning_item_skills` table improves retrieval precision, weighting, referential integrity, and maintainability.
3. **Grounded response generation needs content retrieval.** `knowledge_sources` and `knowledge_chunks` are the minimum PostgreSQL-only replacement for runtime content retrieval. This does not imply any content-ingestion workflow in scope.
4. **Mem0 personalization needs Postgres truth.** The MemZero report already says Mem0 should not be the canonical brain and recommends `user_context_events` plus `user_profile_summary`; this final design promotes them because personalization is core to tutor quality. fileciteturn22file18

---

## 7. Skill Graph

### 7.1 Responsibilities

The Skill Graph answers:

```text
What domain is this request in?
What skill is being discussed?
What aliases or ambiguous meanings apply?
What prerequisites does this skill require?
What related or often-confused skills matter?
What learning items can teach, practice, or verify this skill?
What grounded content should support the response?
```

The table-reference source already identifies `skill_relationships` as the mechanism for prerequisites, related concepts, unlocks, and prerequisite repair. fileciteturn21file12 The final design keeps that, but strengthens skill resolution and retrieval.

### 7.2 Core domain objects

#### `SkillArea`

High-level domain: DSA, system design, operating systems, databases, SQL, networking, distributed systems, backend, data engineering, cloud, debugging, tools, and future domains.

Required semantics:

```text
id
slug
name
description
is_active
metadata_json
```

#### `Skill`

Canonical Skill Graph node. A skill can be a module, concept, subskill/subtopic, competency, case, checkpoint, or assessment anchor. Use `parent_skill_id` for hierarchy. Do not create a separate subskills/subtopics table.

Required semantics:

```text
id
area_id / area_slug
slug
name
description
parent_skill_id
skill_type
difficulty
is_active
metadata_json
```

Recommended `skill_type` values:

```text
module
concept
subskill
competency
case
checkpoint
pattern
technique
```

#### `SkillAlias`

Production skill resolution requires aliases. This table maps user language to canonical skills.

Required semantics:

```text
id
skill_id
alias
alias_type        -- synonym, abbreviation, common_phrase, company_phrase, misspelling
language
confidence
is_active
metadata_json
```

Examples:

| Alias | Canonical skill | Notes |
|---|---|---|
| `mutex` | OS locks / mutual exclusion | Could be OS or language-specific; allow ambiguity. |
| `db deadlock` | database transaction deadlocks | Distinguish from OS deadlock. |
| `token bucket` | rate limiting/token bucket | Subskill/subtopic under rate limiting. |
| `lc graph bfs` | BFS graph traversal | DSA alias. |

#### `SkillRelationship`

Relationships between skills. Start with `requires`, `related_to`, `part_of`, `contrasts_with`, and `often_confused_with`.

Required semantics:

```text
id
source_skill_id
target_skill_id
relationship_type
weight
metadata_json
```

Recommended interpretation:

| Relationship | Meaning for orchestrator |
|---|---|
| `requires` | Must be known before target skill is taught deeply. |
| `part_of` | Target skill is inside a larger skill or module. |
| `related_to` | Useful context, not a prerequisite. |
| `contrasts_with` | Useful for comparison questions. |
| `often_confused_with` | Useful for misconception correction. |
| `unlocks` | Useful for next-skill recommendations. |

#### `LearningItem`

Universal object for anything the user can learn, practice, solve, read, review, or be assessed on.

Required semantics:

```text
id
area_id / area_slug
primary_skill_id
item_type
title
difficulty
verification_type
source_type
source_ref_json
estimated_minutes
is_active
metadata_json
```

Recommended `item_type` values:

```text
concept
coding_problem
system_design_case
sql_exercise
debugging_task
quiz_question
diagnostic_question
checkpoint
interview_question
project_lab
reading
```

Recommended `verification_type` values:

```text
none
read
self_explain
quiz
code_acceptance
rubric_eval
mock_interview
```

#### `LearningItemSkill`

Normalized many-to-many skill mapping.

Required semantics:

```text
learning_item_id
skill_id
role              -- primary, secondary, prerequisite, misconception_target, extension
relevance_weight
metadata_json
```

This replaces `skill_ids_json` as the primary retrieval mechanism. `learning_items.primary_skill_id` can remain for fast default routing.

#### `KnowledgeSource` and `KnowledgeChunk`

These provide PostgreSQL-only grounded retrieval. Do not recreate every old content table. Use one source table and one chunk table.

`knowledge_sources` semantics:

```text
id
source_type       -- lesson, article, official_doc, internal_note, interview_corpus, problem_explanation
publisher
title
url
area_id / area_slug
primary_skill_id
metadata_json
is_active
```

`knowledge_chunks` semantics:

```text
id
source_id
skill_id nullable
area_id / area_slug
chunk_type        -- explanation, example, tradeoff, pitfall, rubric, question, answer
content
embedding         -- pgvector if enabled
keywords_tsv      -- PostgreSQL FTS
metadata_json
is_active
```

The key decision is not ingestion; it is runtime shape. The tutor needs a compact, skill-addressable, PostgreSQL-native retrieval surface.

---

## 8. Learner Skill Graph

### 8.1 Responsibilities

The Learner Skill Graph answers:

```text
What does this user currently know?
What have they seen, practiced, solved, or failed?
What was independent vs assisted?
What prerequisites are missing or weak?
What skills/items are due for review?
What confidence do we have in the state?
What personal context should influence response style or strategy?
```

The unified report emphasizes that the Learner Skill Graph must know whether the user has seen, practiced, demonstrated mastery, struggled, become stale, solved independently, or solved with help. It also states the key rule: `seen != understood`, `taught != verified`, `self_reported_known != verified`, and solving after a full solution is not independent. fileciteturn21file10

### 8.2 Evidence-first model

The Learner Skill Graph is not a single mutable score. It has three layers:

```text
Raw evidence:      learning_events, coding submissions, chat evidence references
Current state:     learner_skill_state, user_learning_item_state
Personal context:  user_context_events, user_profile_summary, optional Mem0 recall
```

The table-reference source already defines this event/projection split: `learning_events` is raw evidence, while `learner_skill_state` and `user_learning_item_state` are fast current projections. fileciteturn21file14

### 8.3 `learning_events`

`learning_events` is the append-only ledger of pedagogical evidence.

Recommended fields:

```text
id
user_id
session_id
message_id
area_id / area_slug
skill_id
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

Recommended event types:

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
coding_submission_accepted
coding_submission_failed
solution_revealed
hint_used
misconception_observed
```

Event interpretation:

| Event | Coverage effect | Readiness effect | Verification effect |
|---|---:|---:|---|
| `seen` | Low positive | None | Never verifies. |
| `taught` | Medium positive | Low/none | Never verifies by itself. |
| `covered` | Higher coverage | Low/medium | Needs assessment to verify. |
| `self_reported_known` | Low | Low | Never verifies alone. |
| `attempted` | Low | Depends on result | Not verified. |
| `struggled` | None/low | Negative or weak | May trigger repair/review. |
| `diagnostic_passed` | Medium | Medium/high | Can verify if item quality and help level allow. |
| `checkpoint_passed` | Medium/high | High | Can verify. |
| `coding_submission_accepted` | Medium/high | High for coding item | Can verify with help level 0-1. |
| `solution_revealed` | None | Blocks independent verification for that attempt | No verification. |

### 8.4 `learner_skill_state`

`learner_skill_state` is the fast-read skill projection.

Recommended fields:

```text
user_id
skill_id
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

Recommended statuses:

```text
not_started
seen
taught
attempted
struggled
covered
needs_review
verified
stale
```

The table reference says `learner_skill_state` is a projection, should be updated from learning events and other trusted evidence, and should be read before deciding whether to explain, probe prerequisites, repair prerequisites, or assess. fileciteturn21file16

### 8.5 `user_learning_item_state`

`user_learning_item_state` is the fast-read item projection.

Recommended fields:

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

Recommended states:

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

Help-level model:

```text
0 = independent / no help
1 = small nudge
2 = hint
3 = scaffolded approach
4 = partial solution
5 = full solution / answer-level help
```

Production rule:

```text
verified requires performance evidence and acceptable help level.
help_level >= 5 prevents independent verification for that attempt.
```

### 8.6 Learner-state scoring model

Use three separate scores:

| Score | Meaning | Updated by |
|---|---|---|
| `coverage_score` | How much material the tutor has exposed or taught. | `seen`, `taught`, `covered`, review events. |
| `readiness_score` | How likely the user can apply the skill independently. | diagnostics, checkpoints, coding submissions, item performance. |
| `confidence_score` | How confident the system is in its estimate. | number, recency, source, and quality of evidence. |

Do not collapse these into one “mastery score.” A user can have high coverage and low readiness if they have read many explanations but failed diagnostic questions.

### 8.7 Knowledge tracing stance

The production V1 should use an **interpretable knowledge-tracing-inspired heuristic**, not a black-box deep knowledge tracing model. The reason is practical: the product needs predictable state transitions before it has enough high-quality interaction data to train or calibrate a neural learner model.

Knowledge tracing is the right research family: Corbett and Anderson’s classic knowledge-tracing work modeled changing student knowledge states during skill acquisition. citeturn534038search6 Deep Knowledge Tracing later showed that recurrent neural networks can improve prediction performance and infer structure from student-task sequences, but that is a later-stage modeling path after sufficient interaction data exists. citeturn460086academia49 DAS3H is especially relevant for future versions because it models both learning and forgetting across multi-skill-tagged items, which fits coding problems and multi-skill system-design tasks. citeturn460086academia48

V1 formula direction:

```text
readiness_delta = base_event_weight
                * evidence_confidence
                * item_quality_weight
                * help_level_multiplier
                * recency_multiplier
                * prerequisite_multiplier
```

Example multipliers:

```text
help_level 0: 1.00
help_level 1: 0.85
help_level 2: 0.60
help_level 3: 0.40
help_level 4: 0.20
help_level 5: 0.00 for independent verification
```

### 8.8 Review and forgetting

Keep `next_review_at` in both `learner_skill_state` and `user_learning_item_state`. Retrieval practice and distributed practice should influence review scheduling. Karpicke and Roediger’s retrieval-practice work shows that spacing and retrieval affect retention, and DAS3H specifically models learning and forgetting for optimally scheduled distributed practice across skills. citeturn254500search4turn460086academia48

Production implication:

```text
If readiness is high but last_verified_at is old -> status can become stale.
If readiness is medium and next_review_at <= now -> strategy = revision_or_spaced_repetition.
If diagnostic fails after prior verification -> lower confidence and schedule repair.
```

---

## 9. Personalization and Mem0 Integration

### 9.1 Final decision

Mem0/MemZero should be integrated behind `UserMemoryService`, not wired directly into prompts. The MemZero report already recommends that Mem0 run before TutorOrchestrator so the orchestrator can decide whether memory is relevant, whether it affects the answer, and whether it is used silently or explicitly. fileciteturn22file6

External research supports using long-term memory for multi-session coherence and efficiency. The Mem0 paper describes dynamically extracting, consolidating, and retrieving salient conversational information; it reports improved long-term memory performance and large latency/token reductions compared with full-context approaches. citeturn870351academia38 Official Mem0 docs describe adding conversation facts and interactions for later retrieval, and searching memories with semantic search and filters such as `user_id`. citeturn870351search1turn870351search0

### 9.2 What belongs in personalization memory

Allowed personalization categories:

```text
preferred explanation style
preferred programming language
target role / target level
target company or interview context
study availability
preferred interaction style
self-reported strengths and weaknesses
recent focus areas
interests that affect practice selection
```

The MemZero report lists exactly these kinds of memories: style preference, examples-first preference, programming language preference, domain interest, target role/company, study constraint, self-reported weakness/strength, and preference against full solutions. fileciteturn22file7

### 9.3 What must not be owned by Mem0

Mem0 cannot certify:

```text
verified skill mastery
readiness score
solved independently
attempt count
review schedule
curriculum prerequisite truth
prep-plan progress
```

The MemZero report explicitly maps those facts to `learner_skill_state`, `user_learning_item_state`, `learning_events`, coding submissions, prep progress, and skill/domain tables, not Mem0. fileciteturn22file7

### 9.4 Required Postgres memory tables

#### `app.user_context_events`

Stores durable user-context evidence that is not learning mastery.

Recommended fields:

```text
id
user_id
session_id
message_id
event_type
context_category
memory_text
normalized_key
normalized_value_json
domain
 skill_id
confidence
source
actionability
valid_from
valid_until
is_current
supersedes_event_id
contradicts_event_id
memzero_memory_id
metadata_json
idempotency_key
created_at
```

Recommended event types:

```text
preference_stated
preference_updated
goal_stated
goal_updated
target_company_stated
style_preference_stated
availability_stated
interest_stated
weakness_self_reported
strength_self_reported
context_observed
context_invalidated
```

The MemZero design already defines `user_context_events` as the append-only event ledger for non-mastery user context and lists fields such as context category, memory text, normalized key/value, domain, skill, confidence, actionability, validity, and MemZero memory ID. fileciteturn22file11

#### `app.user_profile_summary`

Fast-read deterministic personalization snapshot.

Recommended fields:

```text
user_id
target_role
target_level
target_companies_json
active_prep_goal
active_prep_plan_id
preferred_explanation_style
preferred_language
preferred_domains_json
known_strengths_json
known_weaknesses_json
recent_focus_json
learning_preferences_json
schedule_constraints_json
memory_confidence_score
last_context_event_id
last_memzero_sync_at
summary_text
metadata_json
created_at
updated_at
```

The MemZero report states that `user_profile_summary` gives the tutor a stable, compact current view of the user because MemZero retrieval is probabilistic. fileciteturn22file11

### 9.5 UserMemorySnapshot

`UserMemoryService` returns a compact object:

```text
user_id
memory_enabled
stable_preferences
active_goals
style_preferences
relevant_interests
recent_focus
silent_context
explicit_context
memory_confidence
personalization_strategy
```

Usage modes:

| Mode | Meaning | Example |
|---|---|---|
| `none` | No memory used. | No useful context. |
| `silent` | Memory affects style or examples but is not mentioned. | User prefers concise answers. |
| `explicit` | Memory is useful to mention. | User asks for Amazon prep and target company is Amazon. |
| `mixed` | Some memory silent, some explicit. | Concise style silent; target role explicit. |

The MemZero report defines these usage modes and says memory should be used only when it changes answer, strategy, examples, recommendation, tone, or next step. fileciteturn22file8

---

## 10. Tutor Orchestrator Layer

### 10.1 Responsibility

The TutorOrchestrator is the decision-making layer. It answers:

```text
What is the user trying to do?
What domain/skill is involved?
Is the user ready for this skill?
Should the tutor explain, probe, repair, guide, quiz, review, or create practice?
What domain context, student context, memory context, and retrieval context should the LLM receive?
What learning events should the extractor look for after the turn?
```

The unified report states that the LLM can help with intent classification, skill mapping, natural-language explanation, diagnostic generation, and summarization, but the platform must own mastery updates, readiness, prerequisites, verified status, long-term learner state, and projection logic. fileciteturn21file13

### 10.2 Runtime objects

#### `RequestContext`

```text
user_id
session_id
message_id
user_message
chat_history_window
attached_prep_plan_id optional
product_mode
```

#### `DomainContext`

```text
area
target_skill
resolved_skill_id
resolution_confidence
ambiguity_candidates
parent_skills
prerequisite_skills
related_skills
often_confused_skills
learning_items
knowledge_chunks
retrieval_notes
```

#### `StudentSnapshot`

```text
user_id
target_skill_state
prerequisite_states
relevant_item_states
recent_learning_events
coding_evidence_summary
prep_plan_context
known_strengths
known_weaknesses
misconception_signals
review_due_items
confidence
```

#### `UserMemorySnapshot`

```text
stable_preferences
active_goals
style_preferences
relevant_interests
recent_focus
silent_context
explicit_context
personalization_strategy
memory_confidence
```

#### `TutorPlan`

```text
intent
strategy
area
target_skill_id
student_level_assumption
prerequisite_action
retrieval_contract
response_contract
practice_or_assessment_contract
memory_usage
state_update_expectations
extractor_hints
```

`TutorPlan` is not stored as a first-class table. This follows the source design distinction between one-turn TutorPlan and durable Prep Plan. fileciteturn22file10

### 10.3 Intent enum

Recommended V1 intents:

```text
concept_explanation
comparison_or_tradeoff
prerequisite_check
practice_request
problem_solving
code_review_or_debugging
system_design_case
sql_or_query_review
roadmap_or_prep_plan
progress_or_readiness_check
revision_request
resource_recommendation
interview_intelligence
creator_request
meta_or_product_question
```

`creator_request` covers generation of diagnostics, drills, lesson snippets, mock questions, rubric prompts, and custom practice items.

### 10.4 Strategy enum

Recommended V1 strategies:

```text
direct_explanation
scaffolded_explanation
prerequisite_probe
prerequisite_repair
socratic_guidance
guided_practice
worked_example_then_fade
misconception_correction
retrieval_grounded_answer
assessment_checkpoint
revision_or_spaced_repetition
prep_plan_generation
creator_generation
```

### 10.5 Strategy selection matrix

| Situation | Strategy |
|---|---|
| Skill clear, prerequisites verified, no active struggle | `direct_explanation` or `retrieval_grounded_answer` |
| Skill clear, prerequisites unknown | `scaffolded_explanation` with one light probe |
| Required prerequisite weak or missing | `prerequisite_repair` |
| User solving a problem | `socratic_guidance`; avoid full answer unless requested |
| User asks for practice | `guided_practice` with appropriate item selection |
| User is stale on a skill | `revision_or_spaced_repetition` |
| Misconception detected | `misconception_correction` |
| User asks for a full system design case | `assessment_checkpoint` or `guided_practice` depending on state |
| User asks to generate a diagnostic, quiz, or drill | `creator_generation` |
| User asks for durable roadmap | `prep_plan_generation` and existing prep-plan tables |

### 10.6 Response contract

Every TutorPlan should produce a response contract with:

```text
answer_depth
format
allowed_directness
hint_policy
prerequisite_bridge
examples_to_use
questions_to_ask
practice_item_to_offer
citation_or_grounding_requirement
max_diagnostic_count
```

Example for “What is deadlock?” in cold start:

```text
intent = concept_explanation
strategy = scaffolded_explanation
prerequisite_action = bridge process/thread/lock/resource briefly
response_contract = simple explanation + concrete lock example + one diagnostic question
state_update_expectations = taught deadlock, seen prerequisites, maybe diagnostic result next turn
```

---

## 11. Retrieval Strategy

### 11.1 Retrieval is layered

The tutor should not send a vague query to a generic retriever and hope for the best. Retrieval must be constrained by DomainContext and StudentSnapshot.

```text
1. Resolve area/skill using skills + aliases.
2. Expand with prerequisites and related skills.
3. Fetch user state for target/prerequisites/items.
4. Fetch learning items through learning_item_skills.
5. Fetch grounded knowledge chunks through skill filters + search.
6. Rerank by skill match, intent match, item relevance, student state, and source fit.
7. Pack only the minimum useful context into the prompt.
```

### 11.2 PostgreSQL-only retrieval implementation

Use PostgreSQL-native capabilities:

```text
Skill resolution:
  - exact slug match
  - alias match
  - trigram similarity over aliases/titles
  - optional embedding similarity over skill descriptions

Content retrieval:
  - skill_id / area filters first
  - full-text search over chunk content
  - optional pgvector similarity over chunk embeddings
  - hybrid lexical + semantic ranking
  - rerank by intent, skill specificity, difficulty, and chunk_type

Learning item retrieval:
  - primary_skill_id exact match
  - learning_item_skills many-to-many match
  - item difficulty appropriate to StudentSnapshot
  - exclude stale/inactive items
  - prioritize verification_type based on strategy
```

RAG research supports evaluating retrieval on more than final-answer quality. RAGAS frames RAG evaluation around context relevance, whether the LLM faithfully uses context, and generation quality; ARES similarly evaluates context relevance, answer faithfulness, and answer relevance. citeturn254500academia48turn254500academia50

### 11.3 Retrieval result object

```text
RetrievedContext
  skill_matches
  prerequisite_context
  related_context
  learning_items
  knowledge_chunks
  user_state_evidence
  memory_context
  omitted_context_summary
```

Prompt packing order:

1. TutorPlan instructions.
2. Target skill and prerequisite state.
3. Most relevant student evidence.
4. Minimal approved memory context.
5. Grounded content chunks.
6. Selected learning items.
7. Chat history window.

### 11.4 Grounding rule

Grounded technical explanations should prefer:

```text
skill-specific chunks > area-level chunks > generic examples > model prior knowledge
```

For cold-start or missing content, the TutorPlan should say whether the LLM may answer from general knowledge or should produce a fallback response with a practice-oriented explanation. This is a response-quality decision, not a data-ingestion concern.

---

## 12. Extractor and Projectors

### 12.1 Extractor role

The extractor reads:

```text
user message
assistant response
TutorPlan
DomainContext
StudentSnapshot
UserMemorySnapshot
```

It emits:

```text
learning_events
user_context_events
```

It should be conservative. A turn that only teaches a concept can emit `taught` or `seen`, but not `verified`. A turn where the user claims knowledge can emit `self_reported_known` or `weakness_self_reported`, but not high-confidence mastery.

### 12.2 Learning event extraction examples

| Interaction | Event emitted |
|---|---|
| Tutor explains deadlock | `taught` for `deadlock`; maybe `seen` for prerequisites mentioned. |
| User answers diagnostic correctly without help | `diagnostic_passed` with help level 0. |
| User asks for hint on BFS | `hint_used`; maybe `attempted`. |
| User accepts full solution and then says “got it” | `solution_revealed`; no verification. |
| LeetCode accepted submission with no tutor help | `coding_submission_accepted`; item may become `solved_independently`. |
| User says “I struggle with recursion” | `user_context_events.weakness_self_reported`; optional low-confidence learning event. |

### 12.3 Projector role

Projectors update:

```text
learner_skill_state
user_learning_item_state
user_profile_summary
user_prep_task_progress only for durable prep-plan tasks
```

Projector logic should be deterministic and replayable from events, but this phase does not require separate infrastructure tables.

### 12.4 State transition examples

Skill state:

```text
not_started -> seen        after first exposure
seen -> taught             after scaffolded explanation
taught -> attempted        after practice attempt
attempted -> struggled     after repeated failure or misconception
covered -> verified        after independent diagnostic/checkpoint success
verified -> stale          after time decay or failed review
stale -> verified          after successful review
```

Item state:

```text
not_started -> seen
seen -> attempted
attempted -> solved_with_help      if success with help_level 2-5
attempted -> solved_independently  if success with help_level 0-1
solved_independently -> verified   if item verification_type supports it
verified -> needs_review/stale     after review interval
```

---

## 13. Model Interaction Design

The platform should use model calls in bounded roles.

### 13.1 Intent/skill resolution

Use deterministic lookup first, model adjudication second.

```text
Input: user message + short chat context
Deterministic: aliases, slugs, FTS, trigram
Model: disambiguate among candidates, classify intent
Output: intent, area, skill candidates, confidence
```

### 13.2 TutorPlan generation

This may be a model-assisted structured output, but must be validated by platform rules.

```text
Input: RequestContext + DomainContext + StudentSnapshot + UserMemorySnapshot
Output: TutorPlan JSON
Validation: required fields, valid strategy, allowed state transitions, prompt budget
```

### 13.3 Response generation

The response model receives:

```text
TutorPlan
DomainContext summary
StudentSnapshot summary
selected memory context
selected retrieval context
recent chat history
```

It does not receive all chat history, all memories, or all events.

### 13.4 Event extraction

The extractor should produce structured evidence with confidence and source references. It should not directly update projection tables.

### 13.5 Diagnostic grading

For diagnostics, use structured rubrics where possible:

```text
expected_concepts
wrong_concept_patterns
score_range
pass_threshold
help_level
```

A diagnostic answer should update readiness only if it maps to a known skill/item and has sufficient confidence.

---

## 14. Scalability and Performance Considerations

These are implementation constraints, not new infrastructure tables.

### 14.1 Per-turn read budget

A normal tutor turn should require bounded reads:

```text
1. Session/chat context read
2. DomainContext read
3. StudentSnapshot read
4. UserMemorySnapshot read if personalization enabled
5. RetrievalContext read if needed by strategy
```

### 14.2 Indexing requirements

Core indexes:

```text
skills(area_slug, slug)
skills(parent_skill_id)
skill_aliases(alias)
skill_aliases(skill_id)
skill_relationships(source_skill_id, relationship_type)
skill_relationships(target_skill_id, relationship_type)
learning_items(primary_skill_id, item_type, difficulty)
learning_item_skills(skill_id, role, relevance_weight)
knowledge_chunks(skill_id, chunk_type)
learning_events(user_id, skill_id, created_at desc)
learning_events(user_id, learning_item_id, created_at desc)
learner_skill_state(user_id, skill_id)
user_learning_item_state(user_id, learning_item_id)
user_context_events(user_id, normalized_key, is_current)
user_profile_summary(user_id)
```

Optional PostgreSQL extensions:

```text
pg_trgm for fuzzy skill/alias matching
PostgreSQL full-text search for content chunks
pgvector for embedding retrieval if vector search is required
```

### 14.3 Prompt budget control

The orchestrator should cap prompt context:

```text
DomainContext: compact skill/prerequisite graph only
StudentSnapshot: current target/prerequisite/item state, not full event history
MemorySnapshot: only relevant preferences/goals/context
Knowledge chunks: top N focused chunks, not broad corpus dumps
Chat history: recent window plus summarized relevant prior context
```

### 14.4 Service boundaries

Recommended internal services:

| Service | Responsibility |
|---|---|
| `DomainService` | Skill resolution, graph traversal, item retrieval, content retrieval. |
| `StudentModelService` | State reads, evidence summaries, coding evidence interpretation. |
| `UserMemoryService` | Profile summary, context events, optional Mem0 recall. |
| `TutorOrchestratorService` | Strategy selection and TutorPlan generation. |
| `ResponseGenerator` | LLM response under TutorPlan contract. |
| `LearningEventExtractor` | Structured learning/context event extraction. |
| `StateProjector` | Skill/item/profile projection updates. |

---

## 15. Trade-Offs and Final Choices

### 15.1 7-table kernel vs 13-table production core

| Option | Pros | Cons | Final decision |
|---|---|---|---|
| Keep exactly 7 tables | Minimal, fast to implement. | Weak skill resolution, JSON-heavy item mapping, no grounded content tables, no durable personalization truth. | Use only for prototype. |
| 13-table production core | Still compact; directly supports skill resolution, retrieval, personalization, and production query patterns. | More schema upfront. | Recommended. |

### 15.2 JSON metadata vs normalized domain tables

| Use JSON for | Normalize now |
|---|---|
| Rubric details, UI hints, source-specific metadata, temporary item details. | Skill aliases, item-skill mapping, concept relationships, user skill/item states. |

Final rule: JSON is acceptable for content-specific metadata, but not for relationships the orchestrator must query on every turn.

### 15.3 Mem0-only memory vs Postgres profile + Mem0 recall

| Option | Pros | Cons | Final decision |
|---|---|---|---|
| Mem0-only personalization | Fast to integrate. | Vendor memory becomes truth; hard to reason deterministically; no compact prompt-time profile. | Reject. |
| Postgres profile/events only | Deterministic and simple. | Weaker semantic recall across long conversations. | Valid MVP fallback. |
| Postgres truth + optional Mem0 recall | Deterministic current profile plus semantic retrieval. | More service logic. | Recommended. |

### 15.4 Heuristic mastery vs ML knowledge tracing

| Option | Pros | Cons | Final decision |
|---|---|---|---|
| Simple heuristic | Explainable, easy to debug. | Less predictive. | Use in V1. |
| Bayesian/IRT calibration | More statistically grounded. | Needs item data and tuning. | Add after enough evidence. |
| Deep Knowledge Tracing | Powerful sequence modeling. | Data-hungry and harder to explain. | Future only. |

### 15.5 LLM-led vs orchestrator-led

| Option | Pros | Cons | Final decision |
|---|---|---|---|
| LLM-led tool calling | Easy to build. | Inconsistent pedagogy and state updates. | Reject for new tutor. |
| Orchestrator-led TutorPlan | Predictable, testable, state-aware. | More backend logic. | Required. |

---

## 16. Final Runtime Flow

### 16.1 Normal concept question

```text
User: “What is deadlock?”

1. IntentResolver:
   intent = concept_explanation
   skill = operating_systems.deadlock

2. DomainService:
   prerequisites = process, thread, lock, resource allocation, blocking wait
   related = mutual exclusion, circular wait, prevention, detection
   content = top chunks for deadlock explanation
   learning_items = diagnostic question + checkpoint + debugging task

3. StudentModelService:
   target_skill_state = unknown or current
   prerequisite_states = known/unknown/weak
   relevant_item_states = none or prior attempts

4. UserMemoryService:
   style = concise/examples-first if known
   relevant goals = only if relevant

5. TutorOrchestrator:
   strategy = scaffolded_explanation if state unknown
   response_contract = short prerequisite bridge + explanation + example + one diagnostic

6. LLM:
   produces guided explanation

7. Extractor:
   emits taught/seen events
   waits for diagnostic answer before verification

8. Projectors:
   update coverage, not verified mastery
```

### 16.2 Problem-solving request

```text
User: “I’m stuck on Word Ladder.”

Strategy:
  Socratic guidance first.
  Retrieve graph/BFS prerequisite state.
  Check prior attempts and help level.
  Offer nudge -> hint -> scaffold -> partial solution -> full solution only if requested.

State:
  hint_used events accumulate.
  solved_with_help is distinct from solved_independently.
```

### 16.3 Creator request

```text
User: “Create a quick diagnostic for deadlocks.”

Strategy:
  creator_generation

DomainContext:
  target skill + prerequisites + common misconceptions

StudentSnapshot:
  difficulty and known weak prerequisites

Output:
  one diagnostic question with expected concepts, scoring rubric, and follow-up action.

Persistence:
  Do not create a durable learning item unless product flow explicitly saves it.
```

---

## 17. Implementation Sequence

This sequence avoids data-ingestion details and focuses only on core architecture.

### Phase 1: Schema and service contracts

Create the 13 new tutor-domain/student-model tables. Define TypeScript/Python contracts for:

```text
DomainContext
StudentSnapshot
UserMemorySnapshot
TutorPlan
RetrievedContext
LearningEvent
```

### Phase 2: DomainService

Implement:

```text
resolve_domain_skill()
get_skill_graph()
get_prerequisites()
get_related_skills()
get_learning_items_for_skill()
get_knowledge_context()
```

### Phase 3: StudentModelService

Implement:

```text
get_skill_state()
get_prerequisite_states()
get_item_states()
get_recent_learning_evidence()
get_coding_evidence_summary()
build_student_snapshot()
```

### Phase 4: UserMemoryService

Implement:

```text
get_profile_summary()
get_relevant_context_events()
optional_mem0_recall()
build_user_memory_snapshot()
```

Do not let Mem0 update mastery. The MemZero report states that Mem0 can influence personalization but cannot certify learning state. fileciteturn22file7

### Phase 5: TutorOrchestratorService

Implement:

```text
classify_intent()
build_tutor_plan()
validate_tutor_plan()
build_retrieval_contract()
build_response_contract()
build_extractor_hints()
```

### Phase 6: Response generation

Implement prompt assembly from:

```text
TutorPlan
DomainContext
StudentSnapshot
UserMemorySnapshot
RetrievedContext
recent chat history
```

### Phase 7: Extractor and projectors

Implement:

```text
extract_learning_events()
extract_user_context_events()
project_skill_state()
project_item_state()
project_profile_summary()
```

### Phase 8: Enable adaptive behavior by strategy

Enable strategies gradually:

```text
scaffolded_explanation
prerequisite_probe
socratic_guidance
guided_practice
revision_or_spaced_repetition
creator_generation
assessment_checkpoint
```

---

## 18. Production Readiness Criteria

The core tutor architecture is ready when these behaviors work consistently:

1. A skill question resolves to a canonical skill, prerequisites, related skills, learning items, and knowledge chunks.
2. A user with missing prerequisites receives prerequisite repair rather than a shallow generic answer.
3. A user with verified prerequisites receives a direct, deeper answer.
4. A user solving a problem receives progressive help, not an immediate full solution unless requested.
5. A solved-with-help item does not become solved independently.
6. A taught skill updates coverage but not verified readiness.
7. A diagnostic answer can update readiness only with sufficient confidence.
8. A stale skill triggers review instead of being treated as permanently mastered.
9. User preferences influence style and examples only when relevant.
10. Mem0 recall can improve personalization but cannot override Postgres learning state.
11. The LLM never independently mutates mastery; it only produces response text and structured extractor candidates.
12. TutorPlan remains ephemeral while durable prep plans remain in existing prep-plan tables.

---

## 19. Final Authoritative Design

The final implementation should use this system boundary:

```text
PostgreSQL-only runtime

Skill Graph:
  app.skill_areas
  app.skills
  app.skill_aliases
  app.skill_relationships
  app.learning_items
  app.learning_item_skills
  app.knowledge_sources
  app.knowledge_chunks

Learner Skill Graph:
  app.learning_events
  app.learner_skill_state
  app.user_learning_item_state
  app.user_context_events
  app.user_profile_summary
  existing coding evidence tables
  existing chat tables as raw references
  existing prep-plan tables for durable roadmaps only

Orchestrator Layer:
  DomainService
  StudentModelService
  UserMemoryService
  TutorOrchestratorService
  ResponseGenerator
  LearningEventExtractor
  StateProjector

Mem0/MemZero:
  optional semantic recall for personalization
  hidden behind UserMemoryService
  never canonical for mastery, readiness, attempts, prerequisites, or item state

LLM:
  language generation, structured classification, diagnostic generation, summarization
  never final authority for durable learner state
```

The older 7-table design is a correct kernel, but not the final production architecture. The production-ready tutor needs skill aliases for reliable skill resolution, normalized item-skill mapping for accurate practice/retrieval, knowledge source/chunk tables for grounded response quality, and user context/profile tables for Mem0-backed personalization. These additions directly improve tutor quality, orchestration effectiveness, scalability, maintainability, response quality, and learning outcomes while still avoiding specialized table sprawl.

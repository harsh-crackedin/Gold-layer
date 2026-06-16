# Proposed Minimal Learner Domain Architecture

**Project:** crackedinlabs/interview-prep-ai  
**Purpose of this document:** Define the simplest useful learner-domain extension that can sit on top of the current architecture without overcomplicating the schema.

The goal is to build an AI tutor that knows what a student has solved, attempted, reviewed, struggled with, and should revisit across DSA, system design, data engineering, tools, courses, and future domains.

This proposal intentionally avoids creating many new tables. It adds only what is needed.

---

## 1. Executive summary

Create **two new tables**:

1. `app.learning_items` — global/shared catalog of things that can be learned, solved, practiced, reviewed, or assessed.
2. `app.user_learning_item_state` — user-specific current state for each learning item.

Everything else should reuse existing tables:

- `pt_learning_events` for raw learning activity/history.
- `pt_user_topic_progress` for topic-level progress/readiness.
- `pt_checkpoint_attempts` for quizzes/checkpoints.
- `user_profile_summary` for compact mentor context.
- `user_memory` for personal/open-ended context.
- `user_coding_submissions` and `user_coding_problems` for DSA evidence.
- `round_questions`, `knowledge_chunks`, and coding catalog as global content sources.

The selected architecture is:

```text
Global content sources
  coding problems
  round_questions
  knowledge chunks
  course modules/labs
        ↓
NEW: app.learning_items
        ↓
User activity creates pt_learning_events
        ↓
Projector updates
NEW: app.user_learning_item_state
        ↓
Tutor fetches compact learner state
        ↓
Personalized tutor response
```

---

## 2. Design goals

### Goals

- Track what a user has solved or completed across multiple engineering domains.
- Avoid domain-specific solved tables like `user_solved_system_design_cases` or `user_solved_data_labs`.
- Reuse the current progress tracker and context-memory architecture.
- Keep implementation small enough to ship quickly.
- Support DSA today and system design/data/tools/courses later.
- Make the tutor feel like a mentor without building a complex knowledge-tracing ML system yet.

### Non-goals for the first version

- No deep knowledge-tracing model.
- No separate skill-state table.
- No separate attempt table.
- No separate misconception table.
- No report table unless reports need to be persisted.
- No rewrite of progress tracker.
- No migration away from existing coding sync tables.

---

## 3. Why only two tables?

The current system already has most of the building blocks:

- It has a learning event stream: `pt_learning_events`.
- It has topic progress: `pt_user_topic_progress`.
- It has checkpoints: `pt_checkpoint_attempts`.
- It has personal context: `user_context_events`, `user_memory`, `user_profile_summary`.
- It has coding evidence: `user_coding_submissions`, `user_coding_problems`.
- It has global content: coding catalog, interview corpus, knowledge chunks.

The missing part is a **bridge**:

```text
Different content types → one common learnable item ID
Different user actions → one common solved/attempted/completed state
```

That requires only:

1. A global item table.
2. A user-item state table.

Everything else can be derived from existing events and tables.

---

## 4. New table 1: `app.learning_items`

## 4.1 Table summary

| Property | Value |
|---|---|
| Table name | `app.learning_items` |
| Type | Global/shared |
| Required now? | Yes |
| Purpose | Universal catalog of things that can be solved, practiced, learned, completed, or assessed. |
| Replaces existing tables? | No |
| Sources | Coding catalog, interview `round_questions`, knowledge chunks, future courses/labs/tools. |

---

## 4.2 Why this table exists

The project has many content sources:

- DSA/coding problems.
- Interview corpus questions.
- System design cases.
- LLD cases.
- Data engineering labs.
- DevOps/tooling tasks.
- Course lessons.
- Quizzes.
- Knowledge chunks.

Each source has its own table and format. The tutor needs one common object to reference when it says:

- “The user solved this.”
- “The user attempted this.”
- “The user saw this but did not complete it.”
- “The user should review this.”
- “The user solved this with help, not independently.”

`learning_items` gives every learnable/practiceable thing one universal ID.

---

## 4.3 What this table should not do

It should **not** duplicate full content.

For example:

- Do not copy the full LeetCode problem statement if it already exists in `catalog.coding_problems`.
- Do not copy the full interview post if it already exists in the interview corpus.
- Do not copy full RAG chunk content if it already exists in `knowledge_chunks`.

Instead, store a reference in `source_ref_json`.

---

## 4.4 Proposed DDL

```sql
CREATE TABLE app.learning_items (
  id BIGSERIAL PRIMARY KEY,

  -- Domain / taxonomy
  area_slug TEXT NOT NULL,
  topic_id BIGINT,
  subtopics_json JSONB,

  -- Item classification
  item_type TEXT NOT NULL,
  source_type TEXT NOT NULL,
  source_ref_json JSONB NOT NULL,

  -- Display / search metadata
  title TEXT NOT NULL,
  slug TEXT,
  difficulty TEXT,

  -- Behavior flags
  is_practiceable BOOLEAN NOT NULL DEFAULT TRUE,
  is_assessable BOOLEAN NOT NULL DEFAULT TRUE,
  is_active BOOLEAN NOT NULL DEFAULT TRUE,

  -- Flexible future extension
  metadata_json JSONB NOT NULL DEFAULT '{}'::jsonb,

  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_learning_items_area ON app.learning_items(area_slug);
CREATE INDEX idx_learning_items_topic ON app.learning_items(topic_id);
CREATE INDEX idx_learning_items_type ON app.learning_items(item_type);
CREATE INDEX idx_learning_items_active ON app.learning_items(is_active) WHERE is_active = TRUE;
CREATE INDEX idx_learning_items_source_type ON app.learning_items(source_type);
```

Optional uniqueness can be added once source references are normalized:

```sql
-- Optional after source_ref_json format stabilizes
-- CREATE UNIQUE INDEX idx_learning_items_source_unique
-- ON app.learning_items(source_type, md5(source_ref_json::text));
```

---

## 4.5 Field explanations

| Field | Why it exists |
|---|---|
| `area_slug` | Domain: `dsa`, `system_design`, `data_engineering`, `tools`, `backend`, etc. Reuse existing `pt_areas` semantics. |
| `topic_id` | Optional link to existing topic taxonomy. This lets item progress roll up into `pt_user_topic_progress`. |
| `subtopics_json` | Lightweight list of subtopics without creating a new skill table. |
| `item_type` | What kind of item this is: coding problem, design case, lab, lesson, quiz, interview question. |
| `source_type` | Where the item came from: LeetCode, interview corpus, knowledge source, course, generated, manual. |
| `source_ref_json` | Pointer to original source row. Keeps this table from duplicating content. |
| `title` | User-facing display name. |
| `slug` | Stable URL/API-friendly identifier where applicable. |
| `difficulty` | Difficulty/level. Flexible text to support Easy/Medium/Hard, L4/L5/Staff, beginner/intermediate/advanced. |
| `is_practiceable` | Whether user can actively practice/attempt this item. |
| `is_assessable` | Whether the system can evaluate user output. |
| `is_active` | Allows deprecation without delete. |
| `metadata_json` | Extension point for domain-specific fields. |

---

## 4.6 Example rows

### DSA / coding problem

```json
{
  "area_slug": "dsa",
  "topic_id": 101,
  "subtopics_json": ["array", "hash-map"],
  "item_type": "coding_problem",
  "source_type": "catalog.coding_problems",
  "source_ref_json": {
    "platform": "leetcode",
    "slug": "two-sum"
  },
  "title": "Two Sum",
  "slug": "leetcode:two-sum",
  "difficulty": "Easy",
  "is_practiceable": true,
  "is_assessable": true
}
```

### System design case from interview corpus

```json
{
  "area_slug": "system_design",
  "topic_id": 220,
  "subtopics_json": ["requirements", "capacity-estimation", "api-design", "caching"],
  "item_type": "design_case",
  "source_type": "round_questions",
  "source_ref_json": {
    "round_question_id": 8821
  },
  "title": "Design URL Shortener",
  "slug": "design-url-shortener",
  "difficulty": "L5",
  "is_practiceable": true,
  "is_assessable": true
}
```

### Knowledge lesson / concept chunk

```json
{
  "area_slug": "data_engineering",
  "topic_id": 330,
  "subtopics_json": ["partitioning", "consumer-groups", "ordering"],
  "item_type": "lesson",
  "source_type": "knowledge_chunks",
  "source_ref_json": {
    "knowledge_chunk_id": 441
  },
  "title": "Kafka Partitioning and Ordering",
  "difficulty": "Intermediate",
  "is_practiceable": false,
  "is_assessable": false
}
```

### Tooling lab

```json
{
  "area_slug": "tools",
  "topic_id": 410,
  "subtopics_json": ["docker", "networking", "debugging"],
  "item_type": "lab",
  "source_type": "course",
  "source_ref_json": {
    "course_slug": "docker-debugging",
    "module_slug": "container-networking-lab"
  },
  "title": "Debug Docker Container Networking",
  "difficulty": "Medium",
  "is_practiceable": true,
  "is_assessable": true
}
```

---

## 4.7 Population/backfill strategy

Start small.

### Phase A: DSA

Backfill from `catalog.coding_problems`.

```text
catalog.coding_problems(platform, slug)
  → app.learning_items
```

Mapping:

| Source | `learning_items` |
|---|---|
| `platform` + `slug` | `source_ref_json` |
| `title` | `title` |
| `difficulty` | `difficulty` |
| `topic_tags` | `subtopics_json` initially, later mapped to topics |
| `statement_md` | stay in catalog table, do not copy |

### Phase B: Interview corpus

Backfill high-quality `round_questions`.

```text
round_questions
  → app.learning_items
```

Mapping:

| Source | `learning_items` |
|---|---|
| `round_questions.id` | `source_ref_json.round_question_id` |
| `question_type` | `item_type` mapping |
| `title` | `title` |
| `difficulty` | `difficulty` |
| `topics` | `subtopics_json` / topic mapping |

### Phase C: Course/lab items

When courses or report weeks arrive, insert course modules/labs as `learning_items`.

No new table required for generic learner state.

---

## 5. New table 2: `app.user_learning_item_state`

## 5.1 Table summary

| Property | Value |
|---|---|
| Table name | `app.user_learning_item_state` |
| Type | User-specific projection/current-state table |
| Required now? | Yes |
| Purpose | Tracks each user’s current state for each learning item. |
| Replaces existing tables? | No |
| Inputs | `pt_learning_events`, coding sync, checkpoints, chat extraction, plan tasks. |
| Outputs | Tutor context, dashboard, reports. |

---

## 5.2 Why this table exists

The tutor needs fast answers to questions like:

- What has this user solved?
- Did they solve it independently or with help?
- How many times did they attempt it?
- Did they fail before solving?
- Is the item stale and due for review?
- What system design cases has the user practiced?
- What labs/courses has the user completed?

You could compute this from `pt_learning_events`, but doing so on every chat turn or dashboard load would be slow and messy.

This table is a projection, similar to `pt_user_topic_progress`, but at the item level.

---

## 5.3 Proposed DDL

```sql
CREATE TABLE app.user_learning_item_state (
  user_id UUID NOT NULL REFERENCES public.users(id) ON DELETE CASCADE,
  learning_item_id BIGINT NOT NULL REFERENCES app.learning_items(id) ON DELETE CASCADE,

  -- Current state
  state TEXT NOT NULL,
  best_outcome TEXT,
  best_score REAL,

  -- Counts
  attempt_count INTEGER NOT NULL DEFAULT 0,
  independent_success_count INTEGER NOT NULL DEFAULT 0,
  assisted_success_count INTEGER NOT NULL DEFAULT 0,
  failure_count INTEGER NOT NULL DEFAULT 0,

  -- Time markers
  first_seen_at TIMESTAMPTZ,
  first_attempted_at TIMESTAMPTZ,
  first_solved_at TIMESTAMPTZ,
  last_attempted_at TIMESTAMPTZ,
  last_solved_at TIMESTAMPTZ,
  last_verified_at TIMESTAMPTZ,
  next_review_at TIMESTAMPTZ,

  -- Tutor-specific metadata
  last_help_level INTEGER,
  evidence_json JSONB NOT NULL DEFAULT '{}'::jsonb,

  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),

  PRIMARY KEY (user_id, learning_item_id)
);

CREATE INDEX idx_ulis_user_state ON app.user_learning_item_state(user_id, state);
CREATE INDEX idx_ulis_user_updated ON app.user_learning_item_state(user_id, updated_at DESC);
CREATE INDEX idx_ulis_next_review ON app.user_learning_item_state(user_id, next_review_at) WHERE next_review_at IS NOT NULL;
```

---

## 5.4 State model

Keep states simple.

| State | Meaning |
|---|---|
| `seen` | User has been exposed to the item but has not attempted it. |
| `attempted` | User tried it but did not complete/solve/verify it. |
| `solved_with_help` | User completed it but needed hints/scaffolding/solution help. |
| `solved_independently` | User completed it without meaningful help. |
| `verified` | User passed an assessment/checkpoint or deterministic verification. |
| `needs_review` | User previously completed/covered it but should revisit soon. |
| `stale` | User has not reviewed/practiced it for long enough that confidence should decay. |

Do not add more states until the product needs them.

---

## 5.5 Field explanations

| Field | Why it exists |
|---|---|
| `user_id` | Owner of the state row. |
| `learning_item_id` | The item being tracked. |
| `state` | Current simplified learner state. |
| `best_outcome` | Domain-specific best result: AC, passed, completed, good rubric score, etc. |
| `best_score` | Numeric score if available. Useful for checkpoints/design rubrics. |
| `attempt_count` | How many attempts/interactions. |
| `independent_success_count` | Counts evidence where user solved without tutor giving the answer. |
| `assisted_success_count` | Counts evidence where user needed help. |
| `failure_count` | Counts failed attempts. |
| timestamp fields | Let the tutor know recency, review timing, and history. |
| `last_help_level` | Tracks how much help was needed recently. |
| `evidence_json` | Flexible latest evidence/provenance. |

---

## 5.6 Example rows

### DSA problem solved independently

```json
{
  "state": "solved_independently",
  "best_outcome": "accepted",
  "best_score": 1.0,
  "attempt_count": 3,
  "independent_success_count": 1,
  "assisted_success_count": 0,
  "failure_count": 2,
  "last_help_level": 0,
  "evidence_json": {
    "source": "leetcode_sync",
    "platform": "leetcode",
    "submission_id": 992212,
    "language": "python"
  }
}
```

### System design case solved with help

```json
{
  "state": "solved_with_help",
  "best_outcome": "completed",
  "best_score": 0.68,
  "attempt_count": 1,
  "independent_success_count": 0,
  "assisted_success_count": 1,
  "failure_count": 0,
  "last_help_level": 3,
  "evidence_json": {
    "source": "chat_checkpoint",
    "rubric": {
      "requirements": 0.8,
      "capacity_estimation": 0.4,
      "tradeoffs": 0.6,
      "failure_modes": 0.5
    },
    "weak_subtopics": ["capacity-estimation", "failure-modes"]
  }
}
```

### Course/lab completed

```json
{
  "state": "verified",
  "best_outcome": "passed_tests",
  "best_score": 0.92,
  "attempt_count": 2,
  "independent_success_count": 1,
  "assisted_success_count": 0,
  "failure_count": 1,
  "evidence_json": {
    "source": "course_lab",
    "test_suite": "docker-networking-lab",
    "passed": true
  }
}
```

---

## 6. How to reuse `pt_learning_events`

Do not create `user_learning_attempts` yet.

Use `pt_learning_events` as the history/event log. Add item details inside `metadata_json`.

### Metadata contract

Every event that relates to a learning item should include:

```json
{
  "learning_item_id": 123,
  "item_event": "attempted",
  "outcome": "failed",
  "help_level": 2,
  "verification_method": "leetcode_verdict",
  "score": 0.0,
  "source_ref": {
    "platform": "leetcode",
    "submission_id": 992212
  }
}
```

### Suggested `item_event` values

| `item_event` | Meaning |
|---|---|
| `seen` | User was shown/exposed to the item. |
| `attempted` | User tried the item. |
| `failed` | Attempt failed. |
| `solved` | User completed/solved item. |
| `verified` | Deterministic or tutor-evaluated verification passed. |
| `reviewed` | User revised/reviewed the item. |
| `needs_review` | Tutor/projector marks it due for review. |

### Help level scale

| Help level | Meaning |
|---:|---|
| 0 | No help; independent. |
| 1 | Small nudge. |
| 2 | Hint. |
| 3 | Scaffolded approach. |
| 4 | Partial solution. |
| 5 | Full solution / answer-level help. |

This is important because a tutor should not treat “solved after full answer” the same as “solved independently.”

---

## 7. Projector design

Create a small service:

```text
api/services/learner_state_projector.py
```

### Responsibility

Read new or changed `pt_learning_events`, then update `user_learning_item_state`.

### Input

`pt_learning_events` rows where:

```text
metadata_json.learning_item_id exists
```

### Output

Upsert into `app.user_learning_item_state`.

### Basic logic

```text
if item_event = seen:
    state = seen unless already stronger
    first_seen_at ||= event.created_at

if item_event = attempted:
    attempt_count += 1
    state = attempted unless already solved/verified
    first_attempted_at ||= event.created_at
    last_attempted_at = event.created_at

if item_event = failed:
    attempt_count += 1
    failure_count += 1
    state = attempted unless already solved/verified

if item_event = solved:
    attempt_count += 1 if this event represents a new attempt
    if help_level <= 1:
        independent_success_count += 1
        state = solved_independently
    else:
        assisted_success_count += 1
        state = solved_with_help unless already solved_independently/verified
    first_solved_at ||= event.created_at
    last_solved_at = event.created_at

if item_event = verified:
    state = verified
    last_verified_at = event.created_at
    best_score = max(best_score, score)

if item_event = reviewed:
    next_review_at = compute_next_review(...)
```

### State precedence

Use simple precedence:

```text
verified
solved_independently
solved_with_help
attempted
seen
```

`needs_review` and `stale` are overlays based on time and review rules.

---

## 8. How this updates existing progress tables

When an item event comes in, update both:

1. `user_learning_item_state` — item-level current state.
2. `pt_user_topic_progress` — topic-level aggregate.

Example:

```text
User solves Two Sum independently
  ↓
pt_learning_events row:
  event_type = checkpoint_passed or covered
  topic_id = arrays/hashmap topic
  metadata_json.learning_item_id = 123
  metadata_json.item_event = solved
  metadata_json.help_level = 0
  ↓
user_learning_item_state:
  state = solved_independently
  independent_success_count += 1
  ↓
pt_user_topic_progress:
  coverage_score += delta
  readiness_score += delta
  attempt_count += 1
```

This means item progress and topic progress stay aligned.

---

## 9. How each domain uses the same two tables

## 9.1 DSA

### Global item

`learning_items` row points to `catalog.coding_problems`.

### User state

`user_learning_item_state` tracks:

- seen;
- attempted;
- solved with help;
- solved independently;
- verified;
- stale/needs review.

### Evidence source

- `user_coding_submissions`.
- `user_coding_problems`.
- `pt_checkpoint_attempts`.
- chat extraction.

### Example

```text
LeetCode Accepted submission
  → pt_learning_events metadata.learning_item_id
  → user_learning_item_state = solved_independently or verified
```

---

## 9.2 System design

### Global item

`learning_items` row points to:

- `round_questions`; or
- manually curated canonical design case; or
- future system-design case table if created.

### User state

Tracks whether the user attempted/completed/verified a case.

### Evidence source

- chat mock/design session;
- checkpoint rubric stored in `pt_checkpoint_attempts.questions_json/answers_json`;
- `pt_learning_events.metadata_json.rubric`.

### Example

```text
User practices URL Shortener
  → chat extractor creates pt_learning_events
  → metadata_json.learning_item_id = URL Shortener item
  → rubric score stored in metadata_json
  → user_learning_item_state = solved_with_help or verified
  → weak subtopics update pt_user_topic_progress.missing_subtopics_json
```

No new system-design-specific solved table is needed.

---

## 9.3 Data engineering

### Global item

`learning_items` row points to:

- course/lab module;
- SQL exercise;
- Kafka/Spark/Airflow lab;
- knowledge chunk lesson;
- interview corpus data-engineering question.

### User state

Tracks lesson/lab completion and assessment.

### Evidence source

- course/lab completion event;
- checkpoint attempt;
- tutor-evaluated answer;
- future code/test runner result.

### Example

```text
User completes Kafka partitioning lab
  → pt_learning_events item_event = verified
  → user_learning_item_state = verified
  → pt_user_topic_progress for Kafka updates readiness_score
```

---

## 9.4 Tools / DevOps

### Global item

`learning_items` row points to tool task/lab.

Examples:

- Debug Docker networking.
- Write a working Dockerfile.
- Diagnose Kubernetes CrashLoopBackOff.
- Fix Git rebase conflict.

### User state

Tracks completion and help level.

### Evidence source

- tutor chat;
- future terminal/lab runner;
- self-report;
- checkpoint.

---

## 9.5 Courses

### Global item

A course lesson, exercise, project, or module becomes a `learning_item`.

### User state

The user can be seen/attempted/verified for that item.

### Evidence source

- course completion event;
- embedded quiz;
- code test result;
- tutor interaction.

---

## 10. Tutor runtime with learner domain

For each chat turn:

```text
1. User sends message
2. System classifies domain and intent
3. Fetch user profile summary
4. Fetch topic progress for relevant topics
5. Fetch item state for relevant/recent items
6. Retrieve knowledge/interview/coding content through RAG
7. Apply tutor policy
8. Generate response
9. Write pt_learning_events
10. Project into user_learning_item_state and pt_user_topic_progress
```

---

## 11. Tutor prompt block

Inject a compact learner-state block, not the full database.

Example:

```text
Learner state:
- Target: Meta L5 Backend
- Current focus: DSA graphs + system design basics
- Recently solved independently: Two Sum, Best Time to Buy/Sell Stock
- Solved with help: Number of Islands
- Recently attempted: Design URL Shortener, score 0.68, weak on capacity estimation
- Weak topics: BFS traversal, capacity estimation, cache consistency tradeoffs
- Due for review: HashMap patterns, URL shortener API design
- Preference: concise, examples first
```

This gives mentor-like behavior without passing huge context.

---

## 12. API endpoints

Start with read APIs only.

### `GET /api/learner/items/recent`

Returns recent item states.

### `GET /api/learner/items?area=dsa&state=solved_independently`

Returns user item state filtered by area/state.

### `GET /api/learner/summary`

Returns compact learner summary for dashboard/tutor.

Suggested response:

```json
{
  "recent_solved": [],
  "recent_attempted": [],
  "needs_review": [],
  "weak_topics": [],
  "active_area": "dsa"
}
```

### `POST /api/learner/items/{id}/mark`

Optional manual mark endpoint for user/admin actions.

Example body:

```json
{
  "item_event": "reviewed",
  "source": "manual",
  "help_level": 0
}
```

This should write `pt_learning_events`; the projector updates state.

---

## 13. Weekly reports without a report table

Do not create `user_weekly_learning_reports` immediately.

Generate weekly reports from:

- `pt_learning_events`.
- `user_learning_item_state`.
- `pt_user_topic_progress`.
- `pt_checkpoint_attempts`.
- `user_coding_submissions`.
- `chat_messages` summaries if needed.

Report output:

```text
This week you:
- solved 5 DSA items independently
- solved 2 with hints
- attempted 1 system design case
- improved arrays/hashmaps
- still need review on BFS and capacity estimation
- should do one graph checkpoint and retry URL shortener next week
```

Create a persisted report table only when you need:

- email history;
- immutable weekly archive;
- report comparison over time;
- shareable report links.

Optional future table:

```sql
CREATE TABLE app.user_weekly_learning_reports (
  id BIGSERIAL PRIMARY KEY,
  user_id UUID NOT NULL REFERENCES public.users(id) ON DELETE CASCADE,
  week_start DATE NOT NULL,
  week_end DATE NOT NULL,
  summary TEXT NOT NULL,
  report_json JSONB NOT NULL,
  generated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE(user_id, week_start)
);
```

---

## 14. Implementation phases

## Phase 1: Add `learning_items`

### Work

- Create migration for `app.learning_items`.
- Backfill from `catalog.coding_problems`.
- Backfill a small subset of `round_questions` for system design/LLD/SQL/etc.

### Outcome

The system has a universal catalog of learnable/practiceable items.

---

## Phase 2: Add `user_learning_item_state`

### Work

- Create migration for `app.user_learning_item_state`.
- Write initial backfill from `user_data.user_coding_problems`.
- Mark accepted coding problems as solved/verified.

### Outcome

You can answer: “What has this user solved?” for DSA.

---

## Phase 3: Write `learner_state_projector.py`

### Work

- Read new `pt_learning_events` where `metadata_json.learning_item_id` exists.
- Upsert `user_learning_item_state`.
- Keep state precedence simple.

### Outcome

Future chat/checkpoint/course activity updates item state automatically.

---

## Phase 4: Tutor integration

### Work

- Add learner-state fetch function.
- Inject compact learner-state block into tutor prompt.
- Use item state to choose tutor policy.

### Outcome

Tutor can adapt:

- no full answer if user has not attempted;
- quiz if user has seen but not verified;
- review if stale;
- harder follow-up if independently solved;
- scaffold if solved only with help.

---

## Phase 5: Reports and dashboard

### Work

- Add dashboard cards:
  - Solved independently.
  - Solved with help.
  - Attempted but unsolved.
  - Needs review.
- Generate weekly report from state/events.

### Outcome

User sees visible progress across domains.

---

## 15. Migration details

## 15.1 Migration file

Recommended file:

```text
migrations/XXXX_learner_domain_minimal.sql
```

or if following project naming:

```text
db/schema_v14_learner_domain.sql
scripts/migrate_v14_learner_domain.py
```

## 15.2 Rollout flags

Add env flag:

```text
LEARNER_DOMAIN_MODE=off|shadow|on
```

Modes:

| Mode | Behavior |
|---|---|
| `off` | No writes/read usage. |
| `shadow` | Populate tables but do not affect tutor response. |
| `on` | Use learner state in tutor prompt and dashboard. |

This mirrors the existing progress/user-context rollout style.

---

## 16. Product behavior enabled by this domain

## 16.1 Mentor memory

The tutor can say:

> “You solved Two Sum independently, but Number of Islands required hints. Let’s test graph traversal without giving the full approach.”

## 16.2 Cross-domain prep map

The dashboard can show:

```text
DSA
- 24 solved independently
- 8 solved with help
- 6 need review

System Design
- 2 cases attempted
- 1 verified
- weak: capacity estimation

Data Engineering
- Kafka lesson seen
- partitioning lab not attempted
```

## 16.3 Better weekly reports

Reports become evidence-based instead of generic:

```text
You improved in arrays/hashmaps this week.
You attempted one design case but needed scaffolding.
Next week: retry URL shortener and do BFS checkpoint.
```

## 16.4 Tutor policy decisions

The tutor can use item state to decide:

| User state | Tutor behavior |
|---|---|
| `seen` | Ask a diagnostic question. |
| `attempted` | Ask what failed, then hint. |
| `solved_with_help` | Re-test with less help. |
| `solved_independently` | Give harder variant. |
| `verified` | Move forward or schedule review. |
| `needs_review` | Short recall quiz. |
| `stale` | Reorientation/refresher. |

---

## 17. Risks and controls

## Risk 1: Treating exposure as mastery

`seen` must not increase readiness much. Reading an interview post or receiving an explanation is not the same as solving.

### Control

Only `solved_independently` or `verified` should strongly affect readiness.

---

## Risk 2: Overusing LLM judgment

For DSA, deterministic signals are better than LLM extraction.

### Control

Prefer:

- LeetCode accepted verdict;
- unit test result;
- checkpoint score;
- rubric score;
- explicit manual mark.

Use LLM judgment for soft domains only when deterministic signals are unavailable.

---

## Risk 3: Item catalog becomes messy

If `learning_items` is populated from many sources without normalization, duplicates will appear.

### Control

- Use stable `source_type` and `source_ref_json`.
- Add dedupe scripts.
- Add unique index after source formats stabilize.
- Keep `is_active` instead of deleting bad/duplicate rows.

---

## Risk 4: Too much prompt context

Do not inject all item states.

### Control

Inject only:

- recent solved items;
- recent attempted items;
- active weak items;
- due-for-review items;
- current topic items.

---

## Risk 5: Privacy/security

This table is user-specific and should respect RLS/user ownership boundaries.

### Control

- `user_learning_item_state` must be scoped by `user_id`.
- Backend queries must filter by authenticated user ID.
- Avoid exposing another user’s item state.
- Follow the same deletion/forget semantics used elsewhere.

---

## 18. What not to build yet

Do not build these in V1:

- `user_learning_attempts`.
- `user_skill_state`.
- `user_misconceptions`.
- `tutor_interventions`.
- `rubrics`.
- `weekly_reports`.

These may be useful later, but they are not required to ship the first learner-domain version.

Use JSON metadata and existing tables first.

---

## 19. Final selected architecture

```text
GLOBAL SHARED LAYER
───────────────────
public/corpus/catalog content:
- catalog.coding_problems
- interview corpus: interview_posts / interview_rounds / round_questions
- knowledge_sources / knowledge_chunks
- future course/lab/task sources

NEW:
- app.learning_items

USER ACTIVITY / EVENT LAYER
───────────────────────────
- app.chat_messages
- app.pt_learning_events
- app.user_context_events
- app.pt_checkpoint_attempts
- user_data.user_coding_submissions

USER STATE / PROJECTION LAYER
─────────────────────────────
- app.pt_user_topic_progress
- app.user_profile_summary
- app.user_memory
- user_data.user_coding_problems

NEW:
- app.user_learning_item_state

TUTOR RUNTIME
─────────────
1. classify domain/intent
2. fetch profile summary
3. fetch topic progress
4. fetch relevant item state
5. retrieve global content via RAG
6. choose tutor policy
7. respond
8. write learning/context events
9. projector updates item/topic state
```

---

## 20. Final recommendation

Build the learner domain with only two new required tables:

1. **`app.learning_items`** — global/shared, universal catalog of learnable/practiceable items.
2. **`app.user_learning_item_state`** — user-specific, current solved/attempted/verified/review state for each item.

Reuse everything else.

This gives the project the minimum needed structure to build a real tutor:

- It knows what content exists.
- It knows what the user has solved.
- It distinguishes independent solving from assisted solving.
- It works across DSA, system design, data engineering, tools, courses, and future domains.
- It avoids overengineering.
- It preserves the existing architecture.

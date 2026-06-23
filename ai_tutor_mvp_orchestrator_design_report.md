# AI Tutor Orchestrator MVP Design Report

**Project:** Crackedin Labs AI Tutor / Creator Platform  
**Focus:** MVP AI Tutor Orchestrator, Domain Snapshot, Student Snapshot, Resume Context, MemZero Integration  
**Runtime database decision:** PostgreSQL only  
**SQLite decision:** SQLite is fully discarded for runtime, retrieval, progress tracking, and user state.  
**Status:** MVP architecture proposal based on agreed decisions. This document is not the final migration source of truth.

---

## 1. Executive Summary

The MVP should ship a simple but real AI Tutor Orchestrator instead of waiting for the full production student model.

The agreed MVP architecture is:

```text
User request
  -> resolve initial DSA topic/domain
  -> build Domain Snapshot
  -> build Student Snapshot from existing coding evidence + tutor events
  -> build User Profile Context from resume/profile/MemZero
  -> Tutor Orchestrator creates ephemeral TutorPlan
  -> LLM generates response under TutorPlan
  -> extractor writes tutor learning events and user context events
```

The MVP adds only three new tables:

```text
app.tutor_topics
user_data.user_resume
app.tutor_learning_events
```

The MVP reuses existing tables for user identity, chat history, coding evidence, profile summary, user context, and coding catalog.

The core principle is:

```text
Performance evidence determines learner readiness.
Resume and MemZero personalize the tutor, but do not certify mastery.
```

---

## 2. Final MVP Decisions

### 2.1 Keep the MVP narrow

The first tutor domain is **DSA** only.

The first topic graph should cover common DSA topics such as:

```text
arrays
strings
hash_maps
two_pointers
sliding_window
binary_search
stack
queue
linked_list
trees
graphs_bfs_dfs
heaps
intervals
dynamic_programming
greedy
backtracking
```

System design, SQL, LLD, behavioral, and broader interview domains can be added later after the MVP orchestration loop works.

### 2.2 Use existing coding tables as the first Student Model

The existing coding data is already strong student evidence:

```text
user_data.user_coding_profiles
user_data.user_coding_problems
user_data.user_coding_submissions
code_blobs.user_coding_submission_code
catalog.coding_problems
```

These tables can answer:

```text
What platforms are connected?
Which problems has the user attempted?
Which problems were accepted?
Which languages are used?
Which topics appear strong or weak?
How recent is the evidence?
What failed attempts or repeated struggles exist?
```

For MVP, do not add `user_topic_state`, `user_learning_item_state`, `learning_items`, or `learning_item_topics` yet. Those are future projection and normalization tables.

### 2.3 Add tutor learning events

Existing coding tables do not capture tutor interactions, so add:

```text
app.tutor_learning_events
```

This table records evidence created by tutor interactions:

```text
user was taught a topic
user used a hint
user struggled with a topic
user passed/failed a tutor diagnostic
assistant revealed a full solution
user self-reported weakness
```

This is the minimal bridge between chat behavior and future student modeling.

### 2.4 Use a single resume table

For MVP, use one table:

```text
user_data.user_resume
```

Do not normalize resume skills into many tables yet. Resume parsing can store extracted frameworks, languages, databases, tools, role family, years of experience, projects, and raw parser output in JSON fields.

### 2.5 Keep resume context separate from learner mastery

Resume context is not hard evidence of skill mastery.

Resume should influence:

```text
cold-start assumptions
examples used by the tutor
role-specific framing
preferred technology examples
roadmap personalization
```

Resume should not directly mark:

```text
topic verified
readiness strong
problem solved
mastery achieved
interview-ready
```

The correct hierarchy is:

```text
coding/diagnostic/performance evidence > tutor interaction evidence > resume/self-report prior
```

### 2.6 Add MemZero behind UserModelService

MemZero should be integrated behind the user model layer, not directly wired into prompts.

MemZero can recall:

```text
preferred explanation style
target role/company
preferred language
known goals
self-reported weaknesses
recent focus areas
```

MemZero must not own:

```text
accepted submissions
attempt counts
verified mastery
readiness score
resume truth
badges
streaks
```

PostgreSQL remains canonical truth. MemZero is semantic personalization recall.

---

## 3. Tables Reused in MVP

### 3.1 User identity and chat

```text
public.users
app.chat_sessions
app.chat_messages
app.chat_feedback
```

Usage:

```text
public.users: canonical user identity anchor
app.chat_sessions: active tutor session container
app.chat_messages: raw conversation history
app.chat_feedback: response-quality signal
```

### 3.2 Existing coding evidence

```text
catalog.coding_problems
user_data.user_coding_profiles
user_data.user_coding_problems
user_data.user_coding_submissions
code_blobs.user_coding_submission_code
```

Usage:

```text
catalog.coding_problems: problem metadata, difficulty, topic tags
user_coding_profiles: connected platform profile
user_coding_problems: aggregate per-problem user state
user_coding_submissions: strongest DSA performance evidence
user_coding_submission_code: code-level evidence for review/debugging later
```

### 3.3 Existing personalization/context tables

```text
app.user_context_events
app.user_profile_summary
app.user_memory   -- legacy/optional; do not make it canonical
```

Usage:

```text
user_context_events: structured user context and preferences
user_profile_summary: compact current profile summary
user_memory: legacy memory table; optional compatibility only
```

### 3.4 Existing prep/engagement tables not used by MVP orchestrator

```text
app.user_prep_plans
app.user_prep_task_progress
app.user_prep_plan_feedback
app.user_readiness_snapshots
```

These remain durable prep-plan tables. They are not TutorPlan tables.

---

## 4. New MVP Tables

The following table definitions are initial design shapes only. They are included to guide implementation, not to serve as the final migration source of truth.

---

### 4.1 `app.tutor_topics`

#### Why add it

The orchestrator needs a simple canonical topic surface for DSA. Without this table, the tutor has to infer topics from free text or raw LeetCode tags every turn.

This table gives the MVP:

```text
domain/topic resolution
alias matching
basic prerequisites
related topic lookup
mapping from coding problem tags to tutor topics
```

#### Initial table shape

```sql
CREATE TABLE app.tutor_topics (
  id BIGSERIAL PRIMARY KEY,
  domain_slug TEXT NOT NULL DEFAULT 'dsa',
  slug TEXT NOT NULL,
  name TEXT NOT NULL,
  parent_slug TEXT,
  aliases_json JSONB NOT NULL DEFAULT '[]',
  prerequisite_slugs_json JSONB NOT NULL DEFAULT '[]',
  related_slugs_json JSONB NOT NULL DEFAULT '[]',
  coding_problem_tags_json JSONB NOT NULL DEFAULT '[]',
  difficulty TEXT,
  is_active BOOLEAN NOT NULL DEFAULT TRUE,
  metadata_json JSONB NOT NULL DEFAULT '{}',
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE(domain_slug, slug)
);

CREATE INDEX idx_tutor_topics_domain_slug
ON app.tutor_topics(domain_slug, slug);
```

#### Example row

```json
{
  "domain_slug": "dsa",
  "slug": "graphs_bfs_dfs",
  "name": "Graph Traversal: BFS and DFS",
  "aliases_json": ["graph", "graphs", "bfs", "dfs", "graph traversal"],
  "prerequisite_slugs_json": ["queue", "stack"],
  "related_slugs_json": ["trees", "backtracking"],
  "coding_problem_tags_json": ["Graph", "Breadth-First Search", "Depth-First Search"],
  "difficulty": "intermediate"
}
```

#### Future evolution

Later split into normalized tables:

```text
pt_areas
pt_topics
pt_topic_aliases
pt_concept_relationships
```

Do not do this normalization until topic graph complexity requires it.

---

### 4.2 `user_data.user_resume`

#### Why add it

The tutor needs structured resume context for personalization and cold start. A single table is enough for MVP because resume parsing is probabilistic and can store extracted data in JSON.

This table gives the MVP:

```text
role family
experience level
years of experience
languages/frameworks/databases/cloud tools
project/work context
raw parser output
current resume version
```

#### Initial table shape

```sql
CREATE TABLE user_data.user_resume (
  id BIGSERIAL PRIMARY KEY,
  user_id UUID NOT NULL REFERENCES public.users(id) ON DELETE CASCADE,

  resume_file_key TEXT,
  parser_version TEXT NOT NULL,
  raw_text_hash TEXT,

  role_title TEXT,
  role_family TEXT,
  experience_level TEXT,
  years_experience REAL,

  primary_languages_json JSONB NOT NULL DEFAULT '[]',
  frameworks_json JSONB NOT NULL DEFAULT '[]',
  databases_json JSONB NOT NULL DEFAULT '[]',
  cloud_tools_json JSONB NOT NULL DEFAULT '[]',
  tools_json JSONB NOT NULL DEFAULT '[]',

  work_experience_json JSONB NOT NULL DEFAULT '[]',
  projects_json JSONB NOT NULL DEFAULT '[]',
  education_json JSONB NOT NULL DEFAULT '[]',

  raw_parse_json JSONB NOT NULL DEFAULT '{}',
  confidence REAL NOT NULL DEFAULT 0.5,

  is_current BOOLEAN NOT NULL DEFAULT TRUE,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE UNIQUE INDEX idx_user_resume_current
ON user_data.user_resume(user_id)
WHERE is_current = TRUE;

CREATE INDEX idx_user_resume_user_created
ON user_data.user_resume(user_id, created_at DESC);
```

#### How it should be used

Resume should be read into `UserProfileContext`, not hard-coded into mastery.

Example:

```text
Resume says: mid-level backend engineer, Java, Spring Boot, PostgreSQL.
Coding evidence says: weak on graph problems.
Tutor behavior: use backend/service examples, but do not assume graph mastery.
```

#### Future evolution

Later, if needed, split into:

```text
user_resume_profiles
user_resume_skill_mentions
user_work_experience
```

Do not normalize before the parser output stabilizes.

---

### 4.3 `app.tutor_learning_events`

#### Why add it

Existing coding submissions show coding performance, but not tutor interaction evidence. This table records tutor-created learning evidence without requiring full `user_topic_state` or `user_learning_item_state` projection tables.

This table gives the MVP:

```text
append-only tutor evidence
hint/full-solution tracking
diagnostic pass/fail tracking
self-reported weakness tracking
topic exposure tracking
future replay path for projections
```

#### Initial table shape

```sql
CREATE TABLE app.tutor_learning_events (
  id BIGSERIAL PRIMARY KEY,
  user_id UUID NOT NULL REFERENCES public.users(id) ON DELETE CASCADE,
  session_id BIGINT REFERENCES app.chat_sessions(id) ON DELETE SET NULL,
  message_id BIGINT REFERENCES app.chat_messages(id) ON DELETE SET NULL,

  domain_slug TEXT NOT NULL DEFAULT 'dsa',
  topic_slug TEXT,
  platform TEXT,
  problem_slug TEXT,

  event_type TEXT NOT NULL,
  source TEXT NOT NULL DEFAULT 'tutor',
  confidence REAL NOT NULL DEFAULT 0.5,
  help_level SMALLINT,

  metadata_json JSONB NOT NULL DEFAULT '{}',
  idempotency_key TEXT UNIQUE,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_tle_user_created
ON app.tutor_learning_events(user_id, created_at DESC);

CREATE INDEX idx_tle_user_topic_created
ON app.tutor_learning_events(user_id, domain_slug, topic_slug, created_at DESC);

CREATE INDEX idx_tle_user_problem_created
ON app.tutor_learning_events(user_id, platform, problem_slug, created_at DESC)
WHERE problem_slug IS NOT NULL;
```

#### Initial event types

```text
seen
taught
hint_used
solution_revealed
attempted
struggled
diagnostic_passed
diagnostic_failed
self_reported_known
weakness_self_reported
```

#### Help-level model

```text
0 = independent / no help
1 = small nudge
2 = hint
3 = scaffolded approach
4 = partial solution
5 = full solution / answer-level help
```

#### Important rule

```text
help_level >= 5 prevents independent verification for that attempt.
```

For MVP, this rule can be used inside `StudentModelService` at runtime instead of writing projection rows.

---

## 5. Domain Snapshot

### 5.1 Purpose

The Domain Snapshot tells the orchestrator what the user is asking about.

It should answer:

```text
What domain is this request in?
What topic is being discussed?
What aliases matched?
What prerequisites matter?
What related topics matter?
Which coding problem tags map to this topic?
```

### 5.2 MVP data sources

```text
app.tutor_topics
catalog.coding_problems
```

### 5.3 Domain Snapshot shape

```ts
type DomainSnapshot = {
  domainSlug: 'dsa';
  targetTopic?: {
    slug: string;
    name: string;
    difficulty?: string;
    aliases: string[];
  };
  resolutionConfidence: number;
  matchedAliases: string[];
  prerequisiteSlugs: string[];
  relatedSlugs: string[];
  codingProblemTags: string[];
  ambiguityCandidates: Array<{
    slug: string;
    name: string;
    confidence: number;
  }>;
};
```

### 5.4 Topic resolution strategy

Use deterministic lookup first:

```text
exact topic slug
exact alias match
coding problem tag match
simple fuzzy match over aliases/name
```

Use the LLM only to disambiguate when deterministic matching returns multiple plausible candidates.

---

## 6. Student Snapshot

### 6.1 Purpose

The Student Snapshot tells the orchestrator what the user can likely do right now.

It should answer:

```text
What coding evidence exists?
What problems were solved, failed, or recently attempted?
Which DSA topics look strong or weak?
What tutor events exist for the current topic?
What is the confidence of the estimate?
```

### 6.2 MVP data sources

```text
user_data.user_coding_profiles
user_data.user_coding_problems
user_data.user_coding_submissions
code_blobs.user_coding_submission_code
catalog.coding_problems
app.tutor_learning_events
app.chat_sessions
app.chat_messages
```

### 6.3 What belongs in Student Snapshot

```text
accepted submissions
failed attempts
problem difficulty
problem topic tags
recency of attempts
languages used
recent tutor events
hint/full-solution history
diagnostic pass/fail history
```

### 6.4 What does not belong in core Student Snapshot

Resume and MemZero should not be part of hard learner mastery. They should be passed separately as `UserProfileContext` and `MemoryContext`.

### 6.5 Student Snapshot shape

```ts
type StudentSnapshot = {
  userId: string;
  domainSlug: 'dsa';

  codingEvidence: {
    connectedPlatforms: string[];
    solvedCount: number;
    attemptedCount: number;
    recentLanguages: string[];
    strongTopics: string[];
    weakTopics: string[];
    recentProblems: Array<{
      platform: string;
      slug: string;
      title?: string;
      status: 'attempted' | 'accepted' | 'failed' | 'unknown';
      difficulty?: string;
      topicTags: string[];
      attemptCount?: number;
      lastAttemptAt?: string;
      latestAcceptedAt?: string;
    }>;
  };

  tutorEvidence: {
    recentEvents: Array<{
      eventType: string;
      topicSlug?: string;
      problemSlug?: string;
      helpLevel?: number;
      confidence: number;
      createdAt: string;
    }>;
    hintCountForCurrentTopic: number;
    fullSolutionSeenForCurrentProblem: boolean;
    diagnosticSignals: Array<{
      topicSlug: string;
      passed: boolean;
      confidence: number;
    }>;
  };

  topicReadiness: {
    targetTopicSlug?: string;
    inferredReadiness: 'unknown' | 'weak' | 'medium' | 'strong';
    evidenceConfidence: number;
    missingPrerequisites: string[];
  };
};
```

### 6.6 MVP readiness heuristic

Use transparent rules, not ML.

Example:

```text
strong:
  multiple accepted problems in topic or related tags, recent enough, limited full-solution help

medium:
  some accepted problems or successful tutor diagnostics, but limited evidence

weak:
  repeated failed attempts, many hints, diagnostic failures, or no accepted evidence

unknown:
  no useful coding/tutor evidence
```

Resume may provide a weak cold-start prior only when evidence is missing.

---

## 7. User Profile Context

### 7.1 Purpose

User Profile Context helps the tutor personalize teaching without changing verified mastery.

It should answer:

```text
What role is the user targeting?
What level are they preparing for?
What technologies do they know from resume?
What explanation style or language do they prefer?
What goals or constraints are known?
```

### 7.2 MVP data sources

```text
user_data.user_resume
app.user_profile_summary
app.user_context_events
MemZero recall
```

### 7.3 User Profile Context shape

```ts
type UserProfileContext = {
  resume?: {
    roleTitle?: string;
    roleFamily?: string;
    experienceLevel?: string;
    yearsExperience?: number;
    primaryLanguages: string[];
    frameworks: string[];
    databases: string[];
    cloudTools: string[];
    tools: string[];
    confidence: number;
  };

  profileSummary?: {
    targetRole?: string;
    targetLevel?: string;
    preferredLanguage?: string;
    preferredLearningStyle?: string;
    currentFocusDomain?: string;
    currentFocusTopicSlug?: string;
  };

  memoryContext?: {
    stablePreferences: string[];
    activeGoals: string[];
    relevantMemories: string[];
    memoryConfidence: number;
  };
};
```

### 7.4 How resume affects tutor behavior

Resume can affect:

```text
example selection
tone and depth
role-specific framing
cold-start difficulty assumption
technology examples
prep relevance
```

Resume cannot directly affect:

```text
verified mastery
readiness score
solved status
attempt count
independent success
```

---

## 8. MemZero Integration

### 8.1 Placement

MemZero should sit behind `UserModelService`.

```text
UserModelService
  -> reads Postgres canonical data
  -> reads user resume
  -> reads coding evidence
  -> reads tutor learning events
  -> reads profile/context events
  -> calls MemZero for semantic recall
  -> returns StudentSnapshot + UserProfileContext
```

### 8.2 Canonical ownership rule

```text
PostgreSQL owns truth.
MemZero owns recall.
```

### 8.3 What MemZero can store or retrieve

```text
preferred explanation style
preferred programming language
target role/company
study constraints
self-reported weaknesses
recent focus areas
interests that affect examples
```

### 8.4 What MemZero must not own

```text
accepted submissions
attempt count
verified topic mastery
readiness score
resume parse truth
badges
streaks
prep task completion
```

### 8.5 MVP usage modes

```text
none: no relevant memory found
silent: use memory to adjust style/examples without mentioning it
explicit: mention memory when relevant to the user request
mixed: style silently, goal explicitly
```

---

## 9. Tutor Orchestrator

### 9.1 Responsibility

The Tutor Orchestrator decides how to respond. It should not simply pass the user message to the LLM.

It receives:

```text
RequestContext
DomainSnapshot
StudentSnapshot
UserProfileContext
MemoryContext
recent chat history
```

It outputs an ephemeral `TutorPlan`.

### 9.2 TutorPlan shape

```ts
type TutorPlan = {
  intent:
    | 'concept_explanation'
    | 'practice_request'
    | 'problem_solving'
    | 'code_review_or_debugging'
    | 'progress_or_readiness_check'
    | 'revision_request'
    | 'resource_recommendation'
    | 'creator_request';

  strategy:
    | 'direct_explanation'
    | 'scaffolded_explanation'
    | 'prerequisite_repair'
    | 'socratic_guidance'
    | 'guided_practice'
    | 'misconception_correction'
    | 'revision_or_spaced_repetition'
    | 'creator_generation';

  targetTopicSlug?: string;
  targetProblemSlug?: string;
  prerequisiteAction?: 'none' | 'light_bridge' | 'repair_first';
  responseDepth: 'short' | 'medium' | 'deep';
  hintPolicy: 'nudge_first' | 'hints_allowed' | 'direct_answer_allowed';
  personalizationNotes: string[];
  extractorHints: string[];
};
```

### 9.3 MVP strategies

Start with these strategies:

```text
direct_explanation
scaffolded_explanation
prerequisite_repair
socratic_guidance
guided_practice
```

Add later:

```text
misconception_correction
revision_or_spaced_repetition
creator_generation
assessment_checkpoint
```

### 9.4 Strategy examples

```text
Clear topic + strong evidence:
  direct_explanation

Clear topic + unknown evidence:
  scaffolded_explanation

Missing prerequisite:
  prerequisite_repair

User is stuck on a problem:
  socratic_guidance, with progressive help

User asks for practice:
  guided_practice
```

---

## 10. Runtime Flow

### 10.1 Normal concept question

```text
User: "Explain sliding window."

1. RequestContext created from user/session/message.
2. DomainService resolves dsa/sliding_window from tutor_topics aliases.
3. UserModelService reads coding evidence for Sliding Window tags.
4. UserModelService reads tutor_learning_events for recent hints/diagnostics.
5. UserModelService reads resume/profile/context and MemZero recall.
6. TutorOrchestrator creates TutorPlan.
7. LLM responds with explanation under the TutorPlan.
8. Extractor writes tutor_learning_events: taught/seen.
```

### 10.2 Problem-solving request

```text
User: "I'm stuck on Longest Substring Without Repeating Characters."

1. Resolve problem/topic: sliding_window + hash_maps.
2. Read user_coding_submissions for that problem.
3. Read tutor_learning_events to check prior hints/full solutions.
4. If no full solution requested, use socratic_guidance.
5. If user asks for hint, write hint_used with help_level.
6. If full answer is revealed, write solution_revealed with help_level=5.
```

### 10.3 Resume-personalized explanation

```text
Resume: backend engineer, Java/Spring Boot.
Topic: graphs_bfs_dfs.
Evidence: weak graph attempts.

Tutor behavior:
  Use backend/service examples for graph traversal.
  Do not assume graph mastery.
  Keep readiness based on submissions and tutor events.
```

---

## 11. Services

### 11.1 `DomainService`

Responsibilities:

```text
resolve_domain_topic(user_message)
get_topic_snapshot(topic_slug)
map_problem_tags_to_topics(topic_tags)
get_prerequisites(topic_slug)
get_related_topics(topic_slug)
```

Reads:

```text
app.tutor_topics
catalog.coding_problems
```

### 11.2 `UserModelService`

Responsibilities:

```text
build_student_snapshot(user_id, domain/topic/problem)
build_user_profile_context(user_id, domain/topic/problem)
get_coding_evidence_summary(user_id)
get_tutor_event_summary(user_id)
get_resume_context(user_id)
get_memzero_context(user_id)
```

Reads:

```text
user_data.user_resume
user_data.user_coding_profiles
user_data.user_coding_problems
user_data.user_coding_submissions
catalog.coding_problems
app.tutor_learning_events
app.user_context_events
app.user_profile_summary
MemZero
```

### 11.3 `TutorOrchestratorService`

Responsibilities:

```text
classify intent
select strategy
create TutorPlan
build response contract
build extractor hints
```

### 11.4 `ResponseGenerator`

Responsibilities:

```text
assemble prompt from TutorPlan + snapshots
call LLM
return response text
```

### 11.5 `TutorEventExtractor`

Responsibilities:

```text
extract tutor_learning_events
extract user_context_events
avoid over-claiming mastery
write only conservative evidence
```

---

## 12. Event Extraction Rules

### 12.1 Conservative extraction

The extractor can emit:

```text
taught
seen
hint_used
solution_revealed
attempted
struggled
diagnostic_passed
diagnostic_failed
weakness_self_reported
```

It must not emit verified mastery in MVP.

### 12.2 Examples

```text
Tutor explains binary search:
  event = taught, topic_slug = binary_search

User asks for a hint:
  event = hint_used, help_level = 2

Assistant gives full solution:
  event = solution_revealed, help_level = 5

User answers diagnostic correctly without help:
  event = diagnostic_passed, help_level = 0

User says "I struggle with DP":
  event = weakness_self_reported, topic_slug = dynamic_programming
```

---

## 13. Future Tables Deferred

Do not add these in MVP:

```text
app.user_topic_state
app.user_learning_item_state
app.learning_items
app.learning_item_topics
app.knowledge_sources
app.knowledge_chunks
app.user_engagement_state
app.user_badges
app.user_streak_events
```

### Why defer them

```text
user_topic_state / user_learning_item_state:
  useful after runtime projection performance becomes a problem

learning_items / learning_item_topics:
  useful when we move beyond LeetCode/catalog-driven DSA

knowledge_sources / knowledge_chunks:
  useful when Postgres RAG becomes necessary

badges / streaks / engagement:
  useful after tutor loop has retention surfaces worth measuring
```

---

## 14. Production Guardrails for MVP

1. PostgreSQL is the only runtime database.
2. SQLite must not be used for tutor runtime, retrieval, progress, or user state.
3. Coding submissions are primary DSA performance evidence.
4. Tutor learning events capture chat-created learning evidence.
5. Resume is profile context and cold-start prior, not mastery truth.
6. MemZero is semantic recall, not canonical truth.
7. TutorPlan is ephemeral and should not be stored as a table.
8. Full-solution help blocks independent verification for that attempt.
9. The extractor should be conservative and avoid over-claiming mastery.
10. Retention tables such as badges/streaks are intentionally deferred.

---

## 15. MVP Acceptance Criteria

The MVP orchestrator is ready when it can:

```text
resolve a DSA topic from a user question
read existing LeetCode/coding evidence for that user
read resume context without treating it as mastery
retrieve relevant MemZero memories behind UserModelService
create a TutorPlan before response generation
respond differently for weak/strong/unknown evidence
use progressive help for problem-solving requests
write tutor_learning_events after the turn
run without SQLite anywhere in runtime
```

---

## 16. Final Selected MVP Architecture

```text
PostgreSQL only

New tables:
  app.tutor_topics
  user_data.user_resume
  app.tutor_learning_events

Reused evidence tables:
  public.users
  app.chat_sessions
  app.chat_messages
  app.user_context_events
  app.user_profile_summary
  catalog.coding_problems
  user_data.user_coding_profiles
  user_data.user_coding_problems
  user_data.user_coding_submissions
  code_blobs.user_coding_submission_code

Runtime services:
  DomainService
  UserModelService
  TutorOrchestratorService
  ResponseGenerator
  TutorEventExtractor

Memory:
  MemZero behind UserModelService
  PostgreSQL remains canonical truth

Core runtime object split:
  DomainSnapshot: what topic/domain is involved
  StudentSnapshot: what the user can likely do, based on evidence
  UserProfileContext: resume/profile/MemZero personalization
  TutorPlan: one-turn strategy, ephemeral
```

This keeps the MVP small, shippable, and compatible with the final production architecture without forcing premature table sprawl.

# MVP AI Tutor Orchestrator Design Report

**Project:** Crackedin Labs AI Tutor / Creator Platform  
**Document type:** Production MVP design report  
**Phase:** Design phase only; no implementation PR created  
**Scope:** DSA-first AI Tutor Orchestrator, existing PostgreSQL coding evidence, MVP topic model, PR #11 recommendation logic reuse  
**Runtime database decision:** PostgreSQL only  
**Status:** Recommended production MVP architecture with explicit launch guardrails

---

## 1. Executive Verdict

The proposed architecture is good enough for a production MVP launch if we keep the scope disciplined:

```text
DSA-only tutor
PostgreSQL-only tutor runtime
Existing coding evidence as the first student model
MVP topic table as the first domain model
PR #11 reused only as a practice recommendation engine
TutorPlan remains ephemeral
Tutor learning events capture chat-created evidence
Final Skill Graph / Learner Skill Graph deferred until after launch
```

The main architectural bet is sound: do not wait for the full production learner model before shipping a useful tutor. The MVP can deliver real adaptive behavior by combining:

```text
what the user is asking about
what coding evidence already exists
what tutor interactions have happened
which practice problems are appropriate next
how the tutor should respond this turn
```

The architecture should not be treated as the final platform architecture. It is a production MVP bridge. It should be built so that the final Skill Graph / Learner Skill Graph migration is straightforward later.

### Hard condition for launch

The MVP is launchable only if this boundary is enforced:

```text
PR #11 recommends practice problems.
The Tutor Orchestrator decides whether to use those recommendations.
StudentSnapshot remains the source of readiness and learner-state interpretation.
PostgreSQL remains the source of truth.
```

If PR #11 starts owning mastery, readiness, tutor strategy, or durable learner state, the architecture will drift into a recommendation-led tutor instead of an orchestrator-led tutor.

---

## 2. Architecture Challenge Review

### 2.1 What is strong

The architecture has several strong MVP characteristics.

First, it reuses existing high-signal data. Coding submissions, accepted problems, failed attempts, recency, difficulty, languages, and topic tags are already better evidence than chat-only memory or resume claims.

Second, it keeps the tutor orchestrator in control. The LLM should not decide pedagogy directly. The platform should decide whether the response should be direct explanation, scaffolded explanation, Socratic guidance, prerequisite repair, guided practice, or revision.

Third, it reuses PR #11 pragmatically. PR #11 already has useful recommendation primitives: candidate generation, weak-topic practice, failed-submission retry, revision, similar problem, difficulty progression, guardrails, scoring, and diversity. Reusing this logic avoids rebuilding a parallel problem recommendation engine for the tutor.

Fourth, the MVP does not prematurely implement the full final schema. The final architecture is better long-term, but adding every Skill Graph and Learner Skill Graph table before validating tutor behavior would slow launch and increase migration risk.

### 2.2 What is risky

The architecture has real risks that must be controlled.

#### Risk 1: PR #11 may leak into student-model authority

PR #11 produces recommendation scores. Those scores are useful for ranking practice problems, but they are not mastery scores. A high recommendation score means a problem is a good next candidate, not that the user is ready or verified.

Mitigation:

```text
Expose PR #11 output as practiceCandidates only.
Never write readiness or mastery from recommendation score.
Never let recommendation_type become learner status.
```

#### Risk 2: SQLite bridge from PR #11 conflicts with PostgreSQL-only runtime

PR #11 includes route-level legacy behavior that maps a SQLite user id to a Postgres UUID. The tutor runtime must not depend on SQLite.

Mitigation:

```text
Tutor auth must resolve directly to public.users.id UUID.
ProblemRecommendationService receives Postgres UUID only.
No tutor path may call get_db() for SQLite user identity resolution.
```

#### Risk 3: DSA sheet topics are not the final domain model

`catalog.dsa_sheet_problems.topic_slug` is useful for MVP practice ordering, but it is not the final Skill Graph. It has weaker semantics than `app.skills`, `app.skill_aliases`, `app.skill_relationships`, and `app.learning_item_skills`.

Mitigation:

```text
Use app.tutor_topics as the MVP canonical tutor topic surface.
Map dsa_sheet topic_slug and coding problem tags to app.tutor_topics.
Keep the mapping explicit and replaceable.
```

#### Risk 4: Tutor interactions may not update state cleanly

Existing coding tables do not capture tutor-created learning evidence. Without tutor learning events, the system cannot know whether the tutor gave hints, revealed solutions, taught a topic, or observed struggle.

Mitigation:

```text
Add app.tutor_learning_events.
Extractor writes conservative evidence.
Full-solution help blocks independent verification for that attempt.
```

#### Risk 5: Recommendations can distract from tutoring

A tutor should not always recommend problems. Sometimes it should explain, debug, ask a diagnostic question, or repair prerequisites.

Mitigation:

```text
TutorPlan contains practiceRecommendationPolicy.
ProblemRecommendationService is called only when the policy says practice is useful.
```

### 2.3 Final challenge conclusion

The architecture is strong enough for production MVP if we define production MVP narrowly:

```text
A DSA tutor that can read real coding evidence, adapt response strategy, and recommend appropriate practice problems.
```

It is not yet a full production adaptive learning platform:

```text
No final Skill Graph yet.
No normalized learning item model yet.
No learner skill projection tables yet.
No grounded knowledge chunk retrieval yet.
No ML-based knowledge tracing yet.
```

That is acceptable for MVP. The design should explicitly call this a launch bridge, not the final architecture.

---

## 3. MVP Goals and Non-Goals

### 3.1 Goals

The MVP must support these behaviors:

1. Resolve a DSA topic or coding problem from a user message.
2. Build a Domain Snapshot using existing domain tables and `app.tutor_topics`.
3. Build a Student Snapshot from coding evidence and tutor learning events.
4. Use resume/profile/MemZero only for personalization, not mastery.
5. Create an ephemeral TutorPlan before response generation.
6. Reuse PR #11 logic to recommend practice questions when TutorPlan asks for them.
7. Respond differently for weak, medium, strong, and unknown evidence.
8. Use progressive help for problem-solving and debugging requests.
9. Write tutor-created evidence into `app.tutor_learning_events`.
10. Run tutor runtime without SQLite.

### 3.2 Non-goals

The MVP should not attempt to solve these yet:

1. Full final Skill Graph schema.
2. Full Learner Skill Graph projection model.
3. Production RAG content ingestion.
4. Knowledge chunks and embeddings.
5. ML-based mastery prediction.
6. Badges, streaks, gamification, or retention surfaces.
7. System design, SQL, LLD, behavioral, or non-DSA domains.
8. Durable TutorPlan persistence.
9. Treating resume, memory, or recommendation score as verified mastery.

---

## 4. Final MVP Runtime Architecture

```text
User request
  -> RequestContext
  -> DomainService
       resolves DSA topic/problem
       reads app.tutor_topics
       reads catalog.coding_problems
       reads catalog.dsa_sheet_problems when practice context is needed
  -> UserModelService
       builds StudentSnapshot
       reads user coding profiles/problems/submissions/code
       reads app.tutor_learning_events
  -> UserProfileService
       builds UserProfileContext
       reads resume/profile/context/MemZero
  -> TutorOrchestratorService
       classifies intent
       selects strategy
       decides whether practice recommendation is useful
  -> ProblemRecommendationService
       only called if TutorPlan needs practice candidates
       reuses PR #11 candidate/scoring/guardrail logic
       returns ephemeral ranked practice candidates
  -> ResponseGenerator
       assembles prompt under TutorPlan contract
       generates tutor response
  -> TutorEventExtractor
       writes app.tutor_learning_events
       writes user_context_events when appropriate
```

Important design point:

```text
ProblemRecommendationService sits behind TutorOrchestratorService.
It does not sit before the orchestrator as the main driver of the response.
```

---

## 5. Data Model for MVP

## 5.1 Reused identity and chat tables

```text
public.users
app.chat_sessions
app.chat_messages
app.chat_feedback
```

Responsibilities:

| Table | MVP role |
|---|---|
| `public.users` | Canonical authenticated user identity. |
| `app.chat_sessions` | Tutor session container. |
| `app.chat_messages` | Raw conversation history. |
| `app.chat_feedback` | Response quality feedback signal. |

Chat messages are not the learner model. They are raw source evidence and context.

---

## 5.2 Reused coding evidence tables

```text
user_data.user_coding_profiles
user_data.user_coding_problems
user_data.user_coding_submissions
code_blobs.user_coding_submission_code
```

Responsibilities:

| Table | MVP role |
|---|---|
| `user_data.user_coding_profiles` | Connected coding platform state. |
| `user_data.user_coding_problems` | Per-user aggregate problem evidence. |
| `user_data.user_coding_submissions` | Strongest coding performance evidence. |
| `code_blobs.user_coding_submission_code` | Code-level evidence for review/debugging. |

These tables answer:

```text
Which platforms are connected?
Which problems did the user attempt?
Which problems were accepted?
Which problems have repeated failed attempts?
How recent is the evidence?
Which languages are used?
Which topics look strong or weak?
```

---

## 5.3 Reused domain and practice catalog tables

```text
catalog.coding_problems
catalog.dsa_sheet_problems
```

Responsibilities:

| Table | MVP role |
|---|---|
| `catalog.coding_problems` | Platform problem metadata: title, slug, difficulty, topic tags, similar slugs. |
| `catalog.dsa_sheet_problems` | Curated DSA sheet order, topic slug/name, importance score, interview frequency, foundation score, revision value. |

`catalog.dsa_sheet_problems` comes from the PR #11 recommendation model and is useful for MVP because the tutor needs an ordered, curated DSA practice pool.

This table is not the final domain model. It is a launch-time practice catalog.

---

## 5.4 New MVP tutor tables

The MVP should add only these tutor-specific tables:

```text
app.tutor_topics
user_data.user_resume
app.tutor_learning_events
```

### `app.tutor_topics`

Purpose:

```text
Canonical MVP DSA topic surface.
Used for topic resolution, aliases, prerequisites, related topics, and mapping coding problem tags to tutor topics.
```

Key fields:

```text
domain_slug
slug
name
parent_slug
aliases_json
prerequisite_slugs_json
related_slugs_json
coding_problem_tags_json
difficulty
metadata_json
```

### `user_data.user_resume`

Purpose:

```text
Structured resume context for personalization and cold start.
```

Resume may affect examples, tone, role framing, and technology references. It must not directly mark mastery.

### `app.tutor_learning_events`

Purpose:

```text
Append-only tutor-created learning evidence.
```

Initial event types:

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

Help levels:

```text
0 = independent / no help
1 = small nudge
2 = hint
3 = scaffolded approach
4 = partial solution
5 = full solution / answer-level help
```

Rule:

```text
help_level >= 5 prevents independent verification for that attempt.
```

---

## 5.5 Existing personalization/context tables

```text
app.user_context_events
app.user_profile_summary
app.user_memory optional legacy compatibility
```

Responsibilities:

| Table | MVP role |
|---|---|
| `app.user_context_events` | Durable preferences, goals, self-reported strengths/weaknesses. |
| `app.user_profile_summary` | Compact current profile summary. |
| `app.user_memory` | Legacy compatibility only; not canonical. |

MemZero can be used behind the user profile/memory layer. It must not own mastery or readiness.

---

## 5.6 PR #11 optional recommendation product tables

PR #11 includes:

```text
app.user_problem_recommendations
app.recommendation_events
app.projector_cursors
```

For the tutor MVP, these are optional product-surface tables, not required for TutorPlan.

Use them for:

```text
dashboard recommendation pool
notifications
recommendation lifecycle tracking
background recommendation refresh
```

Do not require them for:

```text
TutorPlan practice candidates
tutor mastery
topic readiness
independent verification
```

Tutor practice recommendations should be ephemeral unless the product explicitly needs a durable dashboard/notification experience.

---

## 5.7 Deferred final architecture tables

Do not add these in the MVP launch:

```text
app.skill_areas
app.skills
app.skill_aliases
app.skill_relationships
app.learning_items
app.learning_item_skills
app.knowledge_sources
app.knowledge_chunks
app.learning_events
app.learner_skill_state
app.user_learning_item_state
```

These belong to the post-MVP Skill Graph / Learner Skill Graph architecture.

---

## 6. Domain Snapshot

### 6.1 Purpose

Domain Snapshot tells the orchestrator what the request is about.

It should answer:

```text
What domain is this request in?
What topic is being discussed?
Which problem, if any, is referenced?
Which aliases matched?
What prerequisites matter?
What related topics matter?
Which coding problem tags map to this topic?
Which DSA sheet topic maps to this tutor topic?
```

### 6.2 Data sources

```text
app.tutor_topics
catalog.coding_problems
catalog.dsa_sheet_problems
```

### 6.3 Shape

```ts
type DomainSnapshot = {
  domainSlug: 'dsa';
  targetTopic?: {
    slug: string;
    name: string;
    difficulty?: string;
    aliases: string[];
  };
  targetProblem?: {
    platform: 'leetcode';
    slug: string;
    title?: string;
    difficulty?: string;
    topicTags: string[];
    similarSlugs: string[];
  };
  dsaSheetContext?: {
    sheetName: string;
    sheetVersion: string;
    topicSlug?: string;
    topicName?: string;
    problemOrder?: number;
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

### 6.4 Topic resolution strategy

Use deterministic matching first:

```text
exact tutor topic slug
exact alias match
coding problem slug/title match
coding problem tag match
DSA sheet topic_slug/topic_name match
simple fuzzy match over aliases/name
```

Use the LLM only to disambiguate multiple plausible matches.

---

## 7. Student Snapshot

### 7.1 Purpose

Student Snapshot tells the orchestrator what the user can likely do right now.

It should answer:

```text
What coding evidence exists?
What problems were solved, failed, or recently attempted?
Which topics look strong or weak?
What tutor events exist for the current topic/problem?
Has the user seen hints or full solutions?
Are prerequisites missing?
How confident is the estimate?
```

### 7.2 Data sources

```text
user_data.user_coding_profiles
user_data.user_coding_problems
user_data.user_coding_submissions
code_blobs.user_coding_submission_code
catalog.coding_problems
catalog.dsa_sheet_problems
app.tutor_learning_events
app.chat_sessions
app.chat_messages
```

### 7.3 Shape

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
      dsaSheetTopicSlug?: string;
      attemptCount?: number;
      failedAttemptCount?: number;
      lastAttemptAt?: string;
      latestAcceptedAt?: string;
      nearSolveScore?: number;
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

### 7.4 MVP readiness heuristic

Use transparent rules.

```text
strong:
  multiple accepted problems in topic or related tags,
  recent enough,
  limited full-solution help,
  no repeated recent diagnostic failure

medium:
  some accepted problems or successful diagnostics,
  but limited evidence or older evidence

weak:
  repeated failed attempts,
  high hint/full-solution usage,
  failed diagnostics,
  no accepted evidence in topic

unknown:
  no useful coding/tutor evidence
```

Resume and MemZero can provide cold-start personalization, but not readiness truth.

---

## 8. PR #11 Recommendation Logic Reuse

### 8.1 Role in MVP

PR #11 should be reused as the implementation foundation for:

```text
practice problem ranking
weak-topic practice suggestions
failed-submission retry suggestions
revision suggestions
similar-problem suggestions
next-in-topic suggestions
difficulty progression suggestions
```

It should not be reused as:

```text
student mastery model
tutor strategy selector
durable learner state projector
final domain graph
```

### 8.2 Recommendation types

The MVP should preserve these PR #11 recommendation types:

| Recommendation type | Tutor use |
|---|---|
| `standard_practice` | General unsolved curated practice. |
| `weak_topic_practice` | Repair weak topic or low solve coverage. |
| `failed_submission_retry` | Encourage retry when user has failed but may be close. |
| `revision` | Revisit solved problems after staleness. |
| `next_in_topic` | Continue curated topic sequence. |
| `similar_problem` | Reinforce recently solved or submitted pattern. |
| `next_difficulty` | Controlled step-up when user is ready. |

### 8.3 PR #11 signal reuse

The MVP should reuse these signals:

```text
curated importance
interview frequency
topic weakness
difficulty fit
failed attempts
near-solve score
revision due date
progression in topic
freshness / recency
recently shown penalty
sync stale penalty
```

### 8.4 PR #11 guardrail reuse

Reuse these guardrails:

```text
remove inactive sheet rows
remove missing catalog rows
remove duplicate active rows when using active pool
remove solved non-revision problems
respect cooldown when using persisted recommendation lifecycle
block impossible difficulty jumps
limit topic overrepresentation
limit type overrepresentation
remove duplicate same-problem candidates across recommendation types
```

### 8.5 Live similar recommendation reuse

When the user just submitted or solved a problem, reuse live-similar logic:

```text
catalog similar slugs
shared curated patterns
shared catalog topic tags
same DSA sheet topic
```

Tutor usage:

```text
After the user completes, submits, or asks what to solve next,
suggest one to three similar or next-step problems.
```

---

## 9. ProblemRecommendationService

### 9.1 Purpose

`ProblemRecommendationService` is the adapter between Tutor Orchestrator and PR #11 recommendation logic.

It answers:

```text
Which practice problems should the tutor suggest right now, if the TutorPlan wants practice suggestions?
```

It does not answer:

```text
What strategy should the tutor use?
What has the user mastered?
Should the tutor reveal a solution?
Is the user verified?
```

### 9.2 Inputs

```ts
type ProblemRecommendationRequest = {
  userId: string;                 // public.users.id UUID
  platform?: 'leetcode';
  sheetName?: 'default_dsa_sheet';
  sheetVersion?: 'v1';

  targetTopicSlug?: string;
  currentProblemSlug?: string;
  currentSubmissionId?: string | number;

  goal:
    | 'practice'
    | 'weak_topic_repair'
    | 'failed_retry'
    | 'revision'
    | 'similar_problem'
    | 'next_in_topic'
    | 'difficulty_step_up';

  limit: number;
  includeDebug?: boolean;
};
```

### 9.3 Outputs

```ts
type ProblemRecommendationResult = {
  source: 'pr11_candidate_preview' | 'pr11_live_similar_preview';
  maturity: string;
  activityStates: string[];
  learningStates: string[];
  recommendations: Array<{
    rank: number;
    platform: string;
    problemSlug: string;
    title: string;
    difficulty?: string;
    topicSlug?: string;
    topicName?: string;
    recommendationType: string;
    score: number;
    reasonCode: string;
    reason: string;
    evidence: Record<string, unknown>;
    scoreBreakdown?: Record<string, unknown>;
  }>;
  metadata: {
    persisted: false;
    mvpSchemaMode: 'pr11_topic_sheet_compat';
    targetTopicSlug?: string;
    topicFilterApplied: boolean;
  };
};
```

### 9.4 Internal flow

```text
ProblemRecommendationService.recommend_for_tutor(request):

1. Validate userId is a Postgres UUID.
2. Use PR #11 preview_recommendation_candidates for general practice.
3. Use PR #11 preview_live_similar_recommendation_candidates when currentProblemSlug + submission id exist.
4. Apply targetTopicSlug filter when useful.
5. Apply TutorPlan maxProblems limit.
6. Return ephemeral practice candidates.
7. Do not persist recommendation rows by default.
```

### 9.5 PostgreSQL-only rule

The service must not resolve a SQLite user ID.

Correct:

```text
RequestContext.userId = public.users.id UUID
ProblemRecommendationService receives UUID
```

Incorrect:

```text
SQLite integer user id
ensure_pg_user()
recommendation engine
```

---

## 10. Tutor Orchestrator

### 10.1 Responsibility

Tutor Orchestrator decides how to respond. It should not simply pass the user message to the LLM.

It receives:

```text
RequestContext
DomainSnapshot
StudentSnapshot
UserProfileContext
recent chat history
optional practice recommendations
```

It outputs an ephemeral TutorPlan.

### 10.2 TutorPlan shape

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
    | 'revision_or_spaced_repetition'
    | 'misconception_correction';

  targetTopicSlug?: string;
  targetProblemSlug?: string;

  prerequisiteAction: 'none' | 'light_bridge' | 'repair_first';
  responseDepth: 'short' | 'medium' | 'deep';
  hintPolicy: 'nudge_first' | 'hints_allowed' | 'direct_answer_allowed';

  practiceRecommendationPolicy: {
    shouldRecommendProblems: boolean;
    goal?:
      | 'practice'
      | 'weak_topic_repair'
      | 'failed_retry'
      | 'revision'
      | 'similar_problem'
      | 'next_in_topic'
      | 'difficulty_step_up';
    maxProblems: number;
    showInline: boolean;
    reason: string;
  };

  practiceCandidates: Array<{
    rank: number;
    platform: string;
    problemSlug: string;
    title: string;
    difficulty?: string;
    recommendationType: string;
    reason: string;
  }>;

  personalizationNotes: string[];
  extractorHints: string[];
};
```

### 10.3 Strategy selection matrix

| Situation | Strategy | Practice recommendation policy |
|---|---|---|
| Clear topic, strong evidence | `direct_explanation` | Usually no; optional advanced problem. |
| Clear topic, unknown evidence | `scaffolded_explanation` | Optional one beginner-friendly/next-in-topic problem. |
| Missing prerequisite | `prerequisite_repair` | No initially; repair first. |
| User stuck on problem | `socratic_guidance` | No initially; recommend after progress or explicit ask. |
| User asks for practice | `guided_practice` | Yes. |
| Weak topic detected | `guided_practice` or `prerequisite_repair` | Yes, weak-topic repair. |
| Recent failed submission | `socratic_guidance` or `guided_practice` | Yes, failed retry or simpler related problem. |
| Solved topic stale | `revision_or_spaced_repetition` | Yes, revision. |
| User asks what next | `guided_practice` | Yes, next-in-topic or similar problem. |
| Code review/debugging | `socratic_guidance` | No until debug flow reaches next-step phase. |

---

## 11. When to Recommend Problems

### 11.1 Recommend problems when

```text
User explicitly asks for practice.
User asks what to solve next.
User asks for a roadmap within DSA topic.
StudentSnapshot shows repeated failure in a topic.
StudentSnapshot shows solved-but-stale topic.
User just completed or submitted a problem and asks for next step.
Tutor has just finished explaining a concept and one practice problem would help.
```

### 11.2 Do not recommend problems when

```text
User asks for direct debugging help.
User is missing a prerequisite and needs repair first.
User asks for full solution.
User is in the middle of a Socratic problem-solving exchange.
The answer would become cluttered.
The user is asking about product/account/meta issues.
```

### 11.3 Recommendation display limits

```text
Concept explanation: 0-1 problem
Practice request: 2-3 problems
Weak-topic repair: 1-2 problems
Revision request: 2-3 problems
Problem-solving follow-up: 1-2 problems
```

The tutor should not dump a long ranked list by default.

---

## 12. Response Generation Contract

The LLM should receive a compact prompt package:

```text
TutorPlan
DomainSnapshot summary
StudentSnapshot summary
UserProfileContext summary
Practice candidates, only if policy says to use them
Recent chat history window
```

The LLM should be instructed:

```text
Use practice candidates only if the TutorPlan says to recommend practice.
Do not describe recommendation score as mastery or readiness.
Do not imply solved/verified status unless StudentSnapshot supports it.
Prefer progressive help for problem-solving requests.
Avoid full solution unless user asks or hintPolicy allows it.
```

Recommended response style:

```text
Start with the answer/help the user asked for.
Add practice recommendation only when it improves the next step.
Explain briefly why the problem was suggested.
Avoid exposing internal scoring unless user asks.
```

Example output:

```text
To practice this, solve "Longest Substring Without Repeating Characters" next.
It targets sliding window + hash map, and it is a good next step because your recent attempts suggest this pattern needs reinforcement.
```

---

## 13. Tutor Event Extraction

### 13.1 Extractor role

After each turn, the extractor emits conservative evidence.

It can write:

```text
seen
taught
hint_used
solution_revealed
attempted
struggled
diagnostic_passed
diagnostic_failed
weakness_self_reported
self_reported_known
```

It must not write:

```text
verified mastery from explanation only
strong readiness from recommendation score
solved independently after full solution reveal
mastery from resume claims
mastery from MemZero memory
```

### 13.2 Examples

| Interaction | Event |
|---|---|
| Tutor explains binary search | `taught`, topic_slug = `binary_search` |
| User asks for a hint | `hint_used`, help_level = 2 |
| Assistant gives full answer | `solution_revealed`, help_level = 5 |
| User answers diagnostic correctly without help | `diagnostic_passed`, help_level = 0 |
| User says "I struggle with DP" | `weakness_self_reported`, topic_slug = `dynamic_programming` |
| Tutor suggests a practice problem | Optional `seen` for problem/topic; no mastery update |

---

## 14. Runtime Flows

### 14.1 Normal concept question

```text
User: "Explain sliding window."

1. DomainService resolves dsa/sliding_window.
2. UserModelService reads coding evidence for Sliding Window tags.
3. UserModelService reads tutor_learning_events.
4. UserProfileService reads resume/profile/MemZero context.
5. TutorOrchestrator creates TutorPlan:
   strategy = scaffolded_explanation if evidence unknown
   practiceRecommendationPolicy = optional one next-in-topic problem
6. ProblemRecommendationService runs only if policy says yes.
7. ResponseGenerator explains concept and maybe suggests one problem.
8. Extractor writes taught/seen events.
```

### 14.2 Practice request

```text
User: "Give me graph BFS practice."

1. DomainService resolves graphs_bfs_dfs.
2. StudentSnapshot finds graph topic state.
3. TutorPlan:
   strategy = guided_practice
   recommendation goal = weak_topic_repair or next_in_topic
   maxProblems = 3
4. ProblemRecommendationService reuses PR #11 candidate scoring.
5. Tutor returns 2-3 ranked problems with brief reasons.
```

### 14.3 Stuck on problem

```text
User: "I'm stuck on Word Ladder."

1. DomainService resolves problem = word-ladder, topic = graphs_bfs_dfs.
2. StudentSnapshot reads prior Word Ladder submissions and tutor events.
3. TutorPlan:
   strategy = socratic_guidance
   hintPolicy = nudge_first
   shouldRecommendProblems = false
4. Tutor gives a nudge or targeted diagnostic.
5. If user later asks what to solve next, call live-similar recommendation logic.
```

### 14.4 Failed submission follow-up

```text
User has repeated failed attempts on a curated problem.

1. StudentSnapshot marks topic weak or problem struggling.
2. TutorPlan:
   strategy = guided_practice or prerequisite_repair
   recommendation goal = failed_retry
3. ProblemRecommendationService selects retry or nearby practice problem.
4. Tutor suggests retry only if near-solve score or failure pattern supports it.
```

### 14.5 Revision request

```text
User: "What should I revise today?"

1. StudentSnapshot reads solved problems and latest accepted dates.
2. TutorPlan:
   strategy = revision_or_spaced_repetition
   recommendation goal = revision
3. PR #11 revision_due_score helps rank solved-but-stale problems.
4. Tutor suggests a short revision set.
```

---

## 15. Service Boundaries

### 15.1 DomainService

Responsibilities:

```text
resolve_domain_topic(user_message)
resolve_problem(user_message)
get_topic_snapshot(topic_slug)
map_problem_tags_to_topics(topic_tags)
map_dsa_sheet_topic_to_tutor_topic(topic_slug)
get_prerequisites(topic_slug)
get_related_topics(topic_slug)
```

Reads:

```text
app.tutor_topics
catalog.coding_problems
catalog.dsa_sheet_problems
```

### 15.2 UserModelService

Responsibilities:

```text
build_student_snapshot(user_id, domain/topic/problem)
get_coding_evidence_summary(user_id)
get_problem_attempt_history(user_id, problem_slug)
get_topic_strengths_and_weaknesses(user_id)
get_tutor_event_summary(user_id)
```

Reads:

```text
user_data.user_coding_profiles
user_data.user_coding_problems
user_data.user_coding_submissions
code_blobs.user_coding_submission_code
catalog.coding_problems
app.tutor_learning_events
```

### 15.3 UserProfileService

Responsibilities:

```text
build_user_profile_context(user_id)
get_resume_context(user_id)
get_profile_summary(user_id)
get_context_events(user_id)
get_memzero_context(user_id)
```

Reads:

```text
user_data.user_resume
app.user_profile_summary
app.user_context_events
MemZero optional
```

### 15.4 ProblemRecommendationService

Responsibilities:

```text
recommend_problems_for_tutor(request)
recommend_similar_after_submission(request)
recommend_revision_items(request)
recommend_weak_topic_practice(request)
```

Reads through PR #11 repository logic:

```text
catalog.dsa_sheet_problems
catalog.coding_problems
user_data.user_coding_profiles
user_data.user_coding_problems
user_data.user_coding_submissions
optional app.user_problem_recommendations
optional app.recommendation_events
```

Reuses PR #11:

```text
build_user_recommendation_state
decide_eligible_types
generate_candidates
apply_guardrails
score_candidates
apply_diversity
preview_recommendation_candidates
preview_live_similar_recommendation_candidates
```

### 15.5 TutorOrchestratorService

Responsibilities:

```text
classify_intent
select_strategy
decide_practice_recommendation_policy
create TutorPlan
validate TutorPlan
build extractor hints
```

### 15.6 ResponseGenerator

Responsibilities:

```text
assemble prompt from TutorPlan and snapshots
call LLM
return response text
```

### 15.7 TutorEventExtractor

Responsibilities:

```text
extract tutor_learning_events
extract user_context_events
avoid over-claiming mastery
write conservative evidence only
```

---

## 16. PostgreSQL-Only Runtime Rule

The tutor MVP runtime must use PostgreSQL only.

Allowed:

```text
public.users.id UUID
app.chat_sessions
app.chat_messages
catalog.coding_problems
catalog.dsa_sheet_problems
user_data.user_coding_profiles
user_data.user_coding_problems
user_data.user_coding_submissions
app.tutor_topics
app.tutor_learning_events
user_data.user_resume
```

Not allowed in tutor path:

```text
SQLite users
SQLite chat tables
SQLite progress tables
SQLite recommendation state
SQLite-to-Postgres identity bridge
```

If legacy routes still use SQLite elsewhere in the product, they must remain outside the tutor orchestrator runtime.

---

## 17. Production MVP Launch Criteria

The MVP is launch-ready when these pass:

### 17.1 Functional criteria

1. User can ask a DSA concept question and get a topic-aware answer.
2. User can ask for practice and receive ranked problems from PR #11 logic.
3. User stuck on a problem receives Socratic guidance before a full solution.
4. User with weak topic evidence receives repair or easier practice.
5. User with strong evidence receives a more direct/deeper answer.
6. User with stale solved problems can receive revision suggestions.
7. User profile/resume context affects examples but not mastery.
8. Tutor events are written after turns.

### 17.2 Data criteria

1. Topic resolution uses `app.tutor_topics` and catalog data.
2. Coding evidence uses `user_data.user_coding_problems` and `user_data.user_coding_submissions`.
3. Practice recommendations use `catalog.dsa_sheet_problems` and `catalog.coding_problems`.
4. No tutor runtime read/write depends on SQLite.
5. Recommendation rows are not required for TutorPlan.

### 17.3 Safety and pedagogy criteria

1. Full solution reveal is tracked with help_level = 5.
2. Full solution reveal does not count as independent verification.
3. Explanation does not create verified mastery.
4. Self-report does not create verified mastery.
5. Recommendation score is not shown as readiness score.
6. LLM cannot directly mutate student state.

### 17.4 Observability criteria

Track at least:

```text
intent classification distribution
strategy distribution
recommendation policy decisions
recommendation candidate count
recommendation shown count
accepted/suppressed recommendation types
tutor event extraction count
help-level distribution
full solution reveal count
user feedback rating
```

---

## 18. Main Trade-Offs

### 18.1 Why not implement final Skill Graph now?

Because MVP needs to validate tutor behavior first:

```text
Can the tutor adapt?
Can it use coding evidence?
Can it suggest useful practice?
Can it avoid over-helping?
Can it collect meaningful events?
```

The final Skill Graph is the correct destination, but it adds schema and migration complexity before behavior is proven.

### 18.2 Why use PR #11 now?

Because it already solves a real MVP problem:

```text
Given a user and coding history, which DSA problems are reasonable next?
```

Reusing it gives immediate leverage, as long as its boundary is limited to practice recommendation.

### 18.3 Why keep TutorPlan ephemeral?

TutorPlan is a one-turn instructional control object. Persisting it as a table too early would confuse it with durable prep plans and learner state.

The durable records are:

```text
chat_messages for raw conversation
tutor_learning_events for learning evidence
optional recommendation lifecycle tables for dashboard/notification product surfaces
```

### 18.4 Why not use recommendation tables as learner state?

Because recommendation lifecycle is not learning progress.

A shown recommendation means:

```text
The platform suggested a problem.
```

It does not mean:

```text
The user practiced it.
The user solved it.
The user mastered its topic.
The user is ready for the next skill.
```

---

## 19. Post-MVP Migration Path

### Phase 1: Launch MVP

```text
app.tutor_topics
catalog.coding_problems
catalog.dsa_sheet_problems
user coding tables
app.tutor_learning_events
PR #11 recommendation logic
```

### Phase 2: Normalize domain model

```text
app.tutor_topics -> app.skills
catalog.dsa_sheet_problems -> app.learning_items
topic_slug -> skill_id / skill_slug
catalog topic tags -> app.learning_item_skills
```

### Phase 3: Normalize learner state

```text
app.tutor_learning_events -> app.learning_events
runtime StudentSnapshot heuristic -> app.learner_skill_state
problem-level state -> app.user_learning_item_state
```

### Phase 4: Add grounded retrieval

```text
app.knowledge_sources
app.knowledge_chunks
PostgreSQL full-text search
optional pgvector
```

### Phase 5: Improve student modeling

```text
interpretable readiness scoring
item quality weighting
forgetting/review scheduling
later Bayesian or ML-based knowledge tracing if data volume supports it
```

---

## 20. Final Recommended MVP Boundary

Build the MVP around this exact service boundary:

```text
DomainService:
  topic/problem resolution using MVP domain tables

UserModelService:
  student evidence and readiness from coding + tutor events

UserProfileService:
  resume/profile/MemZero personalization only

ProblemRecommendationService:
  PR #11-based practice recommendation adapter

TutorOrchestratorService:
  strategy and TutorPlan ownership

ResponseGenerator:
  language generation under TutorPlan contract

TutorEventExtractor:
  conservative learning/context event extraction
```

The design is suitable for production MVP launch if:

```text
DSA scope is enforced.
PostgreSQL-only runtime is enforced.
PR #11 is kept as recommendation logic only.
TutorPlan remains ephemeral.
Tutor learning events are written conservatively.
Final Skill Graph migration remains planned but deferred.
```

This architecture is pragmatic, shippable, and aligned with the final direction without forcing premature table sprawl.

# MemZero / Mem0 Integration Design Report for Tutor Orchestration

**Project:** Crackedin Labs Adaptive Tutor Architecture  
**Purpose:** Evaluate feasibility and define a detailed design for adding MemZero / Mem0-style long-term user memory to the already-decided tutor architecture.  
**Status:** Design report, not implementation report.  
**Implementation code:** Not included.  

---

## 1. Feasibility Verdict

The proposed MemZero / Mem0 integration is feasible with the current decided tutor architecture.

However, the feasible and safe design is not to make MemZero the canonical brain of the tutor. The correct architecture is:

> PostgreSQL remains the source of truth. MemZero / Mem0 becomes a semantic personalization and user-memory retrieval layer.

The tutor architecture should continue to rely on PostgreSQL for:

- canonical user identity,
- chat history,
- prep plans,
- coding evidence,
- domain graph,
- learning events,
- topic state,
- learning-item state,
- future user-context events,
- future user-profile summary.

MemZero / Mem0 should be used for:

- remembering user preferences,
- recalling durable personal/user context,
- retrieving relevant prior facts during chat,
- improving personalization,
- helping select examples, practice items, tone, and strategy.

It should not be used as the source of truth for:

- mastery,
- readiness,
- verified topic knowledge,
- solved independently vs solved with help,
- attempt counts,
- review schedules,
- curriculum graph truth,
- durable prep-plan progress.

### Final feasibility rating

**Feasibility: 8/10**

This is feasible if the system follows these constraints:

1. PostgreSQL remains canonical.
2. MemZero is hidden behind a `UserMemoryService` abstraction.
3. Memory retrieval is filtered before entering the LLM prompt.
4. The orchestrator decides whether memory is used.
5. Learning mastery is never updated directly from MemZero memories.
6. User context is separated from learning evidence.
7. Privacy, deletion, and memory-scoping rules are built from day one.
8. MemZero rollout starts in shadow mode before affecting live responses.

---

## 2. Current Decided Tutor Architecture

The already-decided tutor architecture is orchestrator-led rather than LLM-led.

The current target flow is:

```text
User message
 -> Intent / domain / topic resolution
 -> DomainService fetches DomainContext
 -> StudentModelService fetches StudentSnapshot
 -> TutorOrchestrator builds TutorPlan
 -> LLM writes response
 -> Extractor writes events
 -> Projectors update durable state
```

The core principle is:

> The LLM writes the final natural-language response, but the platform owns durable learning state, progression, evidence, and orchestration decisions.

### 2.1 Existing PostgreSQL tables reused

| Schema | Table | Role |
|---|---|---|
| `public` | `users` | Canonical user identity. |
| `app` | `chat_sessions` | Chat session containers. |
| `app` | `chat_messages` | Raw user/assistant messages. |
| `app` | `chat_feedback` | Feedback on chat messages. |
| `app` | `api_logs` | Operational/product logs. |
| `app` | `uc_forget_audit` | Forget/delete audit trail. |
| `app` | `user_prep_plans` | Durable multi-day/multi-week prep plans. |
| `app` | `user_prep_task_progress` | Progress against prep-plan tasks. |
| `app` | `user_prep_plan_feedback` | Feedback on generated/active prep plans. |
| `catalog` | `coding_problems` | Global coding problem catalog. |
| `user_data` | `user_coding_profiles` | User coding-platform profile. |
| `user_data` | `user_coding_problems` | Per-user coding problem aggregate state. |
| `user_data` | `user_coding_submissions` | Per-user coding submission evidence. |
| `code_blobs` | `user_coding_submission_code` | Submitted code blobs. |

### 2.2 Recommended 7 new tutor tables

| # | Schema | Table | Layer | Purpose |
|---:|---|---|---|---|
| 1 | `app` | `pt_areas` | Domain Layer | Canonical high-level domains. |
| 2 | `app` | `pt_topics` | Domain Layer | Canonical topics, subtopics, concepts, modules, and skills. |
| 3 | `app` | `pt_concept_relationships` | Domain Layer | Prerequisites, related concepts, unlocks, and concept graph relationships. |
| 4 | `app` | `learning_events` | Student/Event Layer | Append-only structured learning evidence ledger. |
| 5 | `app` | `user_topic_state` | Student Projection | Fast current user-topic readiness/mastery state. |
| 6 | `app` | `learning_items` | Domain + Student Bridge | Universal catalog of learnable/practiceable/verifiable items. |
| 7 | `app` | `user_learning_item_state` | Student Projection | Per-user state for each learning/practice item. |

These 7 tables remain required even after adding MemZero.

MemZero does not remove the need for the event/projection model. It only adds a personalization memory layer.

---

## 3. Where MemZero Fits

MemZero should be integrated as a new layer:

> User Memory Layer

This layer should be exposed internally through:

> `UserMemoryService`

The updated runtime flow becomes:

```text
User message
 -> Intent / domain / topic resolution
 -> DomainService fetches DomainContext
 -> StudentModelService fetches StudentSnapshot
 -> UserMemoryService fetches UserMemorySnapshot
 -> TutorOrchestrator builds TutorPlan
 -> LLM writes response
 -> Extractor writes learning/context events
 -> Projectors update durable state
 -> Optional MemZero sync/update
```

### 3.1 Why this location is correct

MemZero should run before `TutorOrchestrator` creates the `TutorPlan` because the orchestrator needs to decide:

- whether the memory is relevant,
- whether the memory should affect the answer,
- whether the memory should be mentioned explicitly,
- whether the memory should only be used silently,
- whether the memory should be blocked,
- whether the memory is stale or low confidence.

Do not let the LLM directly receive all user memories.

### 3.2 High-level responsibility split

| Component | Responsibility |
|---|---|
| PostgreSQL | Source of truth for users, learning state, events, plans, curriculum, and profile summaries. |
| MemZero / Mem0 | Semantic recall of user preferences, goals, durable context, and personalization facts. |
| UserMemoryService | Retrieves, filters, ranks, and packages memory for the orchestrator. |
| TutorOrchestrator | Decides whether memory affects the TutorPlan. |
| LLM | Generates the final response using the approved TutorPlan context. |
| Extractor | Extracts learning evidence and user-context facts from interactions. |
| Projectors | Update topic state, item state, prep-plan progress, and profile summary. |

---

## 4. What MemZero Should Store

MemZero should store or retrieve user-context memories, not hard learning truth.

### 4.1 Good MemZero memory categories

| Category | Examples | Usage |
|---|---|---|
| Style preference | User prefers concise answers. | Adjust answer length and structure. |
| Explanation preference | User likes examples before theory. | Change teaching format. |
| Programming language preference | User prefers Python or Java. | Use preferred language in examples. |
| Domain interest | User likes dynamic programming. | Recommend more relevant practice. |
| Target role | User is preparing for backend SDE-2. | Adjust depth and roadmap. |
| Target company | User is targeting Amazon. | Use company-specific prep when relevant. |
| Study constraint | User can study 1 hour per day. | Create realistic plans. |
| Motivation context | User's friend got selected at Amazon. | Use only when directly relevant. |
| Self-reported weakness | User struggles with recursion. | Use as low-confidence strategy hint. |
| Self-reported strength | User says they are good at SQL. | Treat as low-confidence until verified. |
| Interaction preference | User does not want full solutions first. | Prefer hints and Socratic guidance. |

### 4.2 Examples

| User statement | Store? | How to use later |
|---|---:|---|
| "I like DP." | Yes | Use for DSA practice selection, not unrelated topics. |
| "My friend got selected at Amazon." | Maybe | Use only when user asks about Amazon/interviews. |
| "Keep answers short." | Yes | Use silently across most responses. |
| "I am targeting backend SDE-2." | Yes | Use for depth, roadmap, and interview framing. |
| "I struggle with recursion." | Yes | Use for DP, trees, graphs, and backtracking scaffolding. |

---

## 5. What MemZero Should Not Store as Authority

MemZero may contain references to some learning-related facts, but it should not be treated as the authoritative store for them.

| Data | Correct authority |
|---|---|
| User has verified BFS mastery. | `app.user_topic_state` |
| User solved Two Sum independently. | `app.user_learning_item_state` + `app.learning_events` |
| User failed deadlock diagnostic. | `app.learning_events` |
| User attempted a coding problem five times. | `user_data.user_coding_submissions` + `app.user_learning_item_state` |
| User is due for review. | `app.user_topic_state.next_review_at` or `app.user_learning_item_state.next_review_at` |
| User used full solution help. | `app.learning_events.help_level` |
| User completed prep-plan task. | `app.user_prep_task_progress` |
| Topic prerequisites. | `app.pt_concept_relationships` |
| Curriculum structure. | `app.pt_topics` and `app.pt_areas` |

### Core rule

> MemZero can influence personalization, but it cannot certify learning state.

---

## 6. New Service: `UserMemoryService`

### 6.1 Purpose

`UserMemoryService` is the internal boundary between the tutor architecture and MemZero.

It should:

- retrieve candidate memories,
- combine them with Postgres profile data,
- rank and filter them,
- block unsafe or irrelevant memories,
- return a compact `UserMemorySnapshot`,
- write approved memories to MemZero,
- keep MemZero optional and replaceable.

### 6.2 Inputs

| Field | Description |
|---|---|
| `user_id` | Canonical user ID from `public.users`. |
| `session_id` | Current chat session. |
| `message_id` | Current user message. |
| `user_message` | Raw text of the current message. |
| `intent` | Resolved user intent. |
| `domain` | Resolved domain, if known. |
| `target_topic_id` | Canonical topic if resolved. |
| `candidate_topic_ids` | Ambiguous or related topics. |
| `domain_context` | Output from DomainService. |
| `student_snapshot` | Output from StudentModelService. |
| `product_context` | Feature flags, prep mode, interview mode, tenant mode, rollout mode. |

### 6.3 Outputs

| Field | Description |
|---|---|
| `stable_preferences` | Durable preferences useful across sessions. |
| `active_goals` | Current prep/career/interview goals. |
| `relevant_interests` | Interests relevant to the current request. |
| `relevant_past_context` | Past user context useful for the current answer. |
| `company_context` | Target companies or interview-related context. |
| `style_context` | Preferred tone, length, depth, examples, and format. |
| `avoid_mentioning_directly` | Memories that may inform strategy but should not be surfaced. |
| `memory_confidence` | Overall confidence in the memory set. |
| `memory_usage_policy` | Whether memory should be ignored, used silently, or mentioned. |
| `memory_trace` | Lightweight debug trace for internal observability. |

---

## 7. New Runtime Object: `UserMemorySnapshot`

`UserMemorySnapshot` should be a compact, policy-filtered object passed to `TutorOrchestrator`.

### 7.1 Required fields

| Field | Purpose |
|---|---|
| `user_id` | User identity. |
| `memory_enabled` | Whether memory is enabled for this user/session. |
| `retrieval_attempted` | Whether MemZero retrieval was attempted. |
| `retrieval_source` | Postgres only, MemZero only, or hybrid. |
| `stable_preferences` | Preferences that are likely to remain valid. |
| `active_goals` | Current target role/company/prep goals. |
| `style_preferences` | Response style and explanation preferences. |
| `relevant_interests` | Relevant interests for this turn. |
| `relevant_past_context` | Useful past context. |
| `recent_focus` | Recently studied domains/topics. |
| `silent_context` | Memories used internally but not mentioned. |
| `explicit_context` | Memories safe and useful to mention. |
| `blocked_context` | Count or references to blocked memories. |
| `memory_confidence` | Low, medium, or high. |
| `personalization_strategy` | None, silent, explicit, or mixed. |
| `expiration_warnings` | Memories that may be stale. |
| `contradiction_warnings` | Potential conflict with newer user statements. |

### 7.2 Usage modes

| Mode | Meaning | Example |
|---|---|---|
| `none` | No memory used. | Cold start or irrelevant memory. |
| `silent` | Memory affects style/strategy but is not mentioned. | User prefers concise answers. |
| `explicit` | Memory may be referenced directly. | User asks for Amazon prep and has Amazon as target. |
| `mixed` | Some memory silent, some explicit. | Concise style silently; target company explicitly. |

---

## 8. Memory Decision Policy

The system should never inject memories randomly. Every memory should pass a policy gate.

### 8.1 Core decision question

> Should this memory change the answer, strategy, examples, recommendation, tone, or next step?

If the answer is no, the memory should not be used.

### 8.2 Ranking criteria

| Criterion | Description |
|---|---|
| Semantic relevance | Does the memory relate to the current user message? |
| Domain overlap | Does it match the current domain or topic? |
| Intent match | Does it help the current intent? |
| Actionability | Can the tutor use it to improve the answer? |
| Confidence | Was the memory explicitly stated or weakly inferred? |
| Freshness | Is it likely still true? |
| Sensitivity | Could using it feel invasive? |
| User benefit | Does it improve learning, prep, or answer quality? |
| Contradiction risk | Does newer information conflict with it? |
| Explicitness safety | Is it safe to mention directly? |

### 8.3 Suggested usage thresholds

| Score band | Action |
|---|---|
| High | Use explicitly if it improves the response. |
| Medium | Use silently for style, depth, strategy, or examples. |
| Low | Ignore. |
| Blocked | Do not pass to the LLM. |

### 8.4 Memory examples

| Memory | Current user message | Correct action |
|---|---|---|
| User likes DP. | "Give me DSA practice." | Use. |
| User likes DP. | "Explain TCP handshake." | Ignore. |
| User prefers concise answers. | Any normal technical question. | Use silently. |
| User's friend got selected at Amazon. | "How do I prepare for Amazon?" | Use lightly. |
| User's friend got selected at Amazon. | "What is mutex?" | Ignore. |

---

## 9. Required PostgreSQL Additions for MemZero Integration

The 7 tutor tables remain unchanged. For clean MemZero integration, add two later-stage tables:

1. `app.user_context_events`
2. `app.user_profile_summary`

These are required if you want controlled, auditable, non-vendor-locked memory.

---

## 10. Table: `app.user_context_events`

### 10.1 Purpose

Append-only event ledger for durable user context that is not learning mastery.

This table stores facts like:

- user prefers concise explanations,
- user targets Amazon,
- user likes DP,
- user wants backend interview prep,
- user has 1 hour/day to study,
- user prefers Python examples.

This table should not replace `app.learning_events`.

### 10.2 Required fields

| Field | Purpose |
|---|---|
| `id` | Primary identifier. |
| `user_id` | FK to `public.users`. |
| `session_id` | Source chat session. |
| `message_id` | Source chat message. |
| `event_type` | Type of user-context event. |
| `context_category` | Preference, goal, target, style, company, availability, interest, etc. |
| `memory_text` | Human-readable extracted fact. |
| `normalized_key` | Stable key like `preferred_style`, `target_company`, `likes_topic`. |
| `normalized_value_json` | Structured representation of extracted value. |
| `domain` | Optional domain such as DSA, system design, OS, DB. |
| `topic_id` | Optional canonical topic ID. |
| `confidence` | Confidence score for extraction. |
| `source` | Chat, onboarding, prep plan, feedback, imported data, admin. |
| `sensitivity_level` | Low, medium, high, restricted. |
| `actionability` | How useful this context is for future responses. |
| `valid_from` | When this context became valid. |
| `valid_until` | Optional expiry timestamp. |
| `is_current` | Whether this fact is currently active. |
| `supersedes_event_id` | Older event replaced by this one. |
| `contradicts_event_id` | Older event contradicted by this one. |
| `memzero_memory_id` | Optional external MemZero memory reference. |
| `metadata_json` | Flexible metadata. |
| `idempotency_key` | Duplicate prevention key. |
| `created_at` | Insert timestamp. |

### 10.3 Recommended event types

| Event type | Example |
|---|---|
| `preference_stated` | User says, "Keep answers short." |
| `preference_updated` | User says, "Actually explain deeply." |
| `goal_stated` | User says, "I want backend interviews." |
| `goal_updated` | User changes from backend to data engineering. |
| `target_company_stated` | User says they are preparing for Amazon. |
| `style_preference_stated` | User asks for examples-first explanations. |
| `availability_stated` | User says they can study 1 hour per day. |
| `interest_stated` | User says they like DP. |
| `weakness_self_reported` | User says they struggle with recursion. |
| `strength_self_reported` | User says they are good at SQL. |
| `context_observed` | Pattern inferred from repeated behavior. |
| `context_invalidated` | User says previous fact no longer applies. |
| `memory_deleted` | User asks to forget something. |

---

## 11. Table: `app.user_profile_summary`

### 11.1 Purpose

Fast-read deterministic user profile for prompting and personalization.

MemZero retrieval is probabilistic. `user_profile_summary` gives the tutor a stable, compact current view of the user.

### 11.2 Required fields

| Field | Purpose |
|---|---|
| `user_id` | FK to `public.users`; one row per user. |
| `target_role` | Backend, full-stack, data engineer, etc. |
| `target_level` | Intern, SDE-1, SDE-2, L5, etc. |
| `target_companies_json` | Target companies. |
| `active_prep_goal` | Current primary prep goal. |
| `active_prep_plan_id` | Optional FK to `app.user_prep_plans`. |
| `preferred_explanation_style` | Concise, detailed, Socratic, examples-first, etc. |
| `preferred_language` | Java, Python, JavaScript, C++, SQL, etc. |
| `preferred_domains_json` | Domains the user prefers. |
| `known_strengths_json` | User-level strengths; not the same as verified mastery. |
| `known_weaknesses_json` | Self-reported or observed weaknesses. |
| `recent_focus_json` | Recent topics/domains. |
| `learning_preferences_json` | Hints, no full solution, more quizzes, etc. |
| `schedule_constraints_json` | Study availability and cadence. |
| `personalization_policy_json` | Explicit/silent personalization preferences. |
| `memory_confidence_score` | Overall confidence in summary. |
| `last_context_event_id` | Last context event included in projection. |
| `last_memzero_sync_at` | Last successful MemZero sync timestamp. |
| `summary_text` | Compact natural-language profile for prompts. |
| `metadata_json` | Flexible metadata. |
| `created_at` | Created timestamp. |
| `updated_at` | Updated timestamp. |

### 11.3 Why this table is still needed with MemZero

| Need | Best source |
|---|---|
| Current target role | `user_profile_summary` |
| Current preferred answer style | `user_profile_summary` |
| Semantic recall of old context | MemZero |
| Full audit trail of context changes | `user_context_events` |
| User-facing memory management | `user_profile_summary` + `user_context_events` |
| Deletion sync | `user_context_events.memzero_memory_id` |

---

## 12. Optional Later Table: `app.memory_retrieval_logs`

### 12.1 Purpose

Internal observability table for memory retrieval and usage.

This is optional and should not be created in V1 unless debugging or compliance requirements demand it.

### 12.2 Possible fields

| Field | Purpose |
|---|---|
| `id` | Primary ID. |
| `user_id` | User. |
| `session_id` | Session. |
| `message_id` | Message. |
| `retrieval_query` | Query sent to memory system. |
| `candidate_count` | Number of memories returned. |
| `used_count` | Number actually used. |
| `blocked_count` | Number blocked by policy. |
| `memzero_memory_ids_json` | Retrieved memory IDs. |
| `used_memories_json` | Memories used in final TutorPlan. |
| `blocked_reasons_json` | Reasons for blocked memories. |
| `latency_ms` | Retrieval latency. |
| `created_at` | Timestamp. |

### 12.3 Recommendation

Start with lightweight traces in existing app/API logs. Add this table only if you need deeper explainability or compliance auditing.

---

## 13. Updated TutorPlan With Memory Context

The existing `TutorPlan` is ephemeral and should remain ephemeral.

Do not create a durable `tutor_plan_logs` table in V1.

Add a `memory_context` section to the ephemeral `TutorPlan`.

### 13.1 Required memory context fields

| Field | Purpose |
|---|---|
| `memory_enabled` | Whether memory is enabled. |
| `memory_retrieved` | Whether memory retrieval was attempted. |
| `memory_used` | Whether memory affected the final plan. |
| `explicit_memories` | Memories safe to mention directly. |
| `silent_memories` | Memories used only internally. |
| `blocked_memories_count` | Count of memories blocked by policy. |
| `style_preferences` | Current style preferences. |
| `goal_context` | Active role/company/prep goals. |
| `interest_context` | Relevant learning interests. |
| `sensitivity_constraints` | What not to mention. |
| `memory_confidence` | Low, medium, or high. |
| `personalization_strategy` | None, silent, explicit, or mixed. |
| `extractor_memory_hints` | What context to watch for after the turn. |

### 13.2 Example behavior

| Situation | TutorPlan memory behavior |
|---|---|
| User prefers concise answers. | Use silently to shorten answer. |
| User likes DP and asks for practice. | Use explicitly or semi-explicitly to include DP. |
| User targets Amazon and asks for Amazon prep. | Use explicitly. |
| User mentioned friend got selected at Amazon and asks about mutex. | Do not use. |
| User has stale target company memory. | Ask lightweight confirmation before using heavily. |

---

## 14. Extractor and Projector Changes

### 14.1 Existing learning extractor

The existing extractor should continue to produce `app.learning_events`.

Examples:

- `seen`,
- `taught`,
- `attempted`,
- `struggled`,
- `diagnostic_passed`,
- `diagnostic_failed`,
- `verified`,
- `coding_submission_accepted`,
- `coding_submission_failed`.

### 14.2 New memory/context extractor

Add a memory-context extraction path.

It should produce `app.user_context_events` and optionally sync approved memories to MemZero.

Examples:

| User statement | Extracted context event |
|---|---|
| "Keep it short." | `preference_stated`, key `preferred_explanation_style`. |
| "I like DP." | `interest_stated`, key `likes_topic`. |
| "I am preparing for Amazon." | `target_company_stated`, key `target_company`. |
| "I have one hour daily." | `availability_stated`, key `study_time_daily`. |
| "I struggle with recursion." | `weakness_self_reported`, key `weak_topic`. |

### 14.3 Projector changes

Add a `UserProfileProjector`.

| Projector | Reads | Writes |
|---|---|---|
| Topic Progress Projector | `learning_events`, coding evidence | `user_topic_state` |
| Item State Projector | `learning_events`, coding submissions | `user_learning_item_state` |
| Prep Plan Projector | learning/item events | `user_prep_task_progress` |
| User Profile Projector | `user_context_events`, feedback, selected chat signals | `user_profile_summary` |

---

## 15. Privacy and Deletion Requirements

Because MemZero stores user-personal memory, privacy cannot be an afterthought.

### 15.1 Required privacy rules

| Rule | Design requirement |
|---|---|
| User isolation | Every memory query must filter by `user_id`. |
| App isolation | Every memory query should filter by app/product scope. |
| No cross-user leakage | Never retrieve memory without user scope. |
| Deletion support | Store `memzero_memory_id` in `user_context_events`. |
| Audit support | Use `app.uc_forget_audit`. |
| Memory invalidation | Use `context_invalidated` and `memory_deleted` events. |
| Sensitive data control | Avoid storing high-risk personal facts unless explicitly needed. |
| User visibility | Later expose remembered facts to the user. |
| Expiry | Some memories should expire or require confirmation. |

### 15.2 Sensitivity levels

| Level | Examples | Default behavior |
|---|---|---|
| Low | Likes DP, prefers concise answers, prefers Python. | Store and use when relevant. |
| Medium | Target company, availability, friend got selected. | Store carefully and use only when relevant. |
| High | Health, religion, politics, identity, family issues. | Avoid unless explicitly requested and product requires it. |
| Restricted | Credentials, secrets, deeply private data. | Do not store. |

---

## 16. Rollout Plan

### Phase 0: Keep tutor foundation unchanged

Build the 7 tutor tables first:

1. `app.pt_areas`
2. `app.pt_topics`
3. `app.pt_concept_relationships`
4. `app.learning_events`
5. `app.user_topic_state`
6. `app.learning_items`
7. `app.user_learning_item_state`

### Phase 1: Add Postgres user-memory truth

Add:

1. `app.user_context_events`
2. `app.user_profile_summary`

At this stage, do not depend on MemZero yet.

### Phase 2: Add `UserMemoryService`

Initially, it can read from:

- `user_profile_summary`,
- recent `user_context_events`,
- selected chat context if needed.

### Phase 3: Add MemZero in shadow mode

In shadow mode:

- write approved context memories to MemZero,
- retrieve memories,
- score and filter them,
- log what would have been used,
- do not change live responses yet.

### Phase 4: Silent personalization

Allow memory to affect:

- answer length,
- explanation format,
- example selection,
- problem recommendation,
- roadmap generation,
- prerequisite strategy.

### Phase 5: Explicit personalization

Allow direct references only when relevant and natural.

Examples:

- "Since you are targeting Amazon..."
- "Since you prefer concise explanations..."
- "Since you have been focusing on DP..."

### Phase 6: User-facing memory controls

Add controls for:

- view remembered facts,
- edit memory,
- delete memory,
- disable personalization,
- reset profile.

---

## 17. Guardrails

### 17.1 Never personalize randomly

Do not use memory just to prove the system remembers.

Bad:

> Since your friend got selected at Amazon, here is what a mutex is.

Good:

> Since Amazon is on your radar, here is how to prepare for Amazon SDE interviews.

### 17.2 Never let MemZero update mastery

Bad:

> User likes DP, therefore mark DP as strong.

Good:

> User likes DP, therefore include some DP practice. Mastery must still come from diagnostics, submissions, and learning events.

### 17.3 Never pass all memories to the LLM

Only pass memories that are:

- relevant,
- useful,
- policy-approved,
- low-risk,
- current,
- scoped to the correct user.

### 17.4 Keep source-of-truth boundaries clear

| Concern | Correct system |
|---|---|
| Curriculum truth | Domain tables. |
| Learning evidence | `learning_events`. |
| Current topic state | `user_topic_state`. |
| Current item state | `user_learning_item_state`. |
| Durable user context | `user_context_events`. |
| Current user profile | `user_profile_summary`. |
| Semantic recall | MemZero. |
| Final response strategy | TutorOrchestrator. |

---

## 18. Final Recommended Architecture

The final architecture should look like this:

```text
PostgreSQL
  - users
  - chat sessions/messages/feedback
  - prep plans
  - coding evidence
  - domain graph
  - learning events
  - topic state
  - learning items
  - item state
  - user context events
  - user profile summary

MemZero / Mem0
  - semantic memory retrieval
  - user preference recall
  - goal/context recall
  - relevant past context recall

UserMemoryService
  - fetches candidate memories
  - filters and ranks memory
  - blocks irrelevant or risky memories
  - returns UserMemorySnapshot

TutorOrchestrator
  - combines DomainContext, StudentSnapshot, and UserMemorySnapshot
  - builds TutorPlan
  - decides explicit vs silent personalization

LLM
  - writes the final response from approved context
  - does not own truth
```

---

## 19. CTO-Level Recommendation

Proceed with the MemZero integration, but do it in the correct order.

Do not wire MemZero directly into chat prompts.

Recommended sequence:

1. Build the 7-table tutor foundation.
2. Add `app.user_context_events`.
3. Add `app.user_profile_summary`.
4. Add `UserMemoryService`.
5. Add memory scoring and policy filtering.
6. Add MemZero in shadow mode.
7. Enable silent personalization.
8. Enable explicit personalization only where relevant.
9. Add user-facing memory controls.

Final decision:

> MemZero is worth using, but only as a governed semantic memory layer. PostgreSQL must remain the canonical source of truth for tutor state, learning evidence, and progress.

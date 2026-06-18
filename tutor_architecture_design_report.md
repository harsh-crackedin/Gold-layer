# Adaptive AI Tutor Architecture Design Report

## 1. Purpose of This Design

This report defines the design for upgrading the current Crackedin chat experience into a durable, adaptive AI tutor. The goal is not to build another generic chatbot. The goal is to build a system that understands:

- what the user is asking now,
- what topic or skill the question belongs to,
- what prerequisite knowledge is required,
- what the user already knows or has attempted,
- what the user is probably ready to learn next,
- and how the assistant should respond in the most useful teaching style.

The target architecture has three main layers:

1. **Domain Layer**: the structured curriculum and content intelligence layer.
2. **Student Model Layer**: the user-specific learning state and evidence layer.
3. **Tutor Administrator / Orchestrator Layer**: the decision-making layer that chooses how to teach before the LLM generates the final response.

This design is intentionally **not DSA-specific**. It must work for DSA, system design, operating systems, networks, databases, SQL, backend engineering, data engineering, cloud, debugging, and future domains.

---

## 2. Where We Stand Today

### 2.1 Current Production Postgres State

The current Postgres database contains the core application tables, chat tables, coding-sync tables, and prep-plan shell tables. Based on the currently observed Postgres tables, the app has:

| Schema | Table | Current Role |
|---|---|---|
| `public` | `users` | Core user identity. |
| `app` | `chat_sessions` | Chat session container. |
| `app` | `chat_messages` | User and assistant messages. |
| `app` | `chat_feedback` | Message-level thumbs up/down feedback. |
| `app` | `api_logs` | Product/API analytics. |
| `app` | `uc_forget_audit` | Forget/delete audit trail. |
| `app` | `user_prep_plans` | Durable prep-plan shell. |
| `app` | `user_prep_task_progress` | User progress against plan tasks. |
| `app` | `user_prep_plan_feedback` | Feedback on prep plans. |
| `catalog` | `coding_problems` | Coding problem catalog. |
| `user_data` | `user_coding_profiles` | Connected coding-platform profile. |
| `user_data` | `user_coding_problems` | Per-user problem status aggregate. |
| `user_data` | `user_coding_submissions` | Per-user coding submissions and verdicts. |
| `code_blobs` | `user_coding_submission_code` | Sensitive submitted code storage. |

This means the current production DB can support chat, user identity, coding-sync evidence, and prep-plan task tracking. It does **not yet contain the full adaptive tutor state** in Postgres.

The major missing Postgres pieces are:

- domain graph tables,
- topic prerequisite relationships,
- durable learning event ledger,
- topic-level user mastery/readiness projection,
- universal learning item abstraction,
- per-user learning item state,
- persistent student profile summary for tutoring,
- context/learning extraction events.

### 2.2 Current SQLite State

The SQLite database has the richer curriculum and knowledge foundation. It contains tables such as:

| SQLite Table | Role |
|---|---|
| `pt_areas` | High-level domains or learning areas. |
| `pt_topics` | Topic taxonomy. |
| `pt_subtopics` | Fine-grained topic breakdown. |
| `pt_concept_relationships` | Prerequisite and concept relationships. |
| `pt_learning_paths` | Predefined learning paths. |
| `pt_checkpoints` | Milestone assessments. |
| `pt_checkpoint_questions` | Checkpoint questions and rubrics. |
| `pt_system_design_cases` | System design practice cases. |
| `knowledge_sources` | RAG source metadata. |
| `knowledge_chunks` | RAG-ready chunks. |
| `knowledge_chunks_vec` | Vector index for knowledge chunks. |
| `knowledge_chunks_fts` | Full-text search index for knowledge chunks. |
| `user_memory` | Legacy/transitional memory. |
| `user_topic_log` | Legacy/transitional topic interaction log. |
| `user_attempt_log` | Legacy/transitional attempt log. |
| `user_weakness_signal` | Legacy/transitional weakness aggregate. |
| `user_revisit_queue` | Legacy/transitional review queue. |

The conclusion is that **SQLite already contains many curriculum concepts**, but production Postgres currently does not expose the minimum tutor state and domain graph needed for runtime adaptive tutoring.

### 2.3 Important Design Decision

Postgres should become the production runtime source of truth for the adaptive tutor.

SQLite can remain a seed/migration source for curriculum and knowledge content, but the live chat pipeline should not depend on SQLite as the long-term runtime tutor brain.

---

## 3. Current Chat Pipeline Overview

The current chat pipeline is primarily **LLM-led**.

Today, the high-level flow is:

1. User sends a message.
2. Chat pipeline builds the prompt and available tools.
3. LLM decides whether to call tools or answer directly.
4. If the LLM calls tools, tool results are returned.
5. LLM writes the final response.
6. Chat messages and tool calls are stored.

This is useful for a general interview assistant, but it has a major limitation: the model is making teaching decisions without first being forced to consult a structured domain graph and student model.

The current model prompt is effectively a coaching prompt. It tells the LLM to behave like a DSA and system design interview coach, use tools for interview/coding data, and answer directly for general concepts. That means the model can answer questions, but it is not yet operating as a controlled adaptive tutor.

### 3.1 Current Pipeline Shape

| Stage | Current Behavior |
|---|---|
| Message intake | User message enters chat endpoint. |
| Prompt construction | Chat history, system prompt, and tool definitions are prepared. |
| Tool decision | LLM decides if it needs tools. |
| Retrieval/data use | Tools may be called for interview questions or coding problems. |
| Response generation | LLM answers directly. |
| Persistence | Chat session/message state is saved. |

### 3.2 Current Limitation

The current system can answer:

- “What is deadlock?”
- “How do I solve Two Sum?”
- “What does Google ask?”
- “Explain consistent hashing.”

But it does not reliably know:

- whether the user understands required prerequisites,
- whether the user has struggled with this topic before,
- whether the topic is due for revision,
- whether the answer should be direct or scaffolded,
- whether the user needs a diagnostic question first,
- whether the answer should update topic readiness,
- whether the user has repeatedly failed related practice.

That is the core reason for introducing the three-layer tutor architecture.

---

## 4. Target Chat Pipeline Overview

The updated pipeline should become **orchestrator-led**.

The LLM should still write the final natural-language answer, but it should no longer be the first component deciding how to teach. The Tutor Administrator / Orchestrator should first generate a structured TutorPlan.

### 4.1 Target Pipeline Shape

| Stage | New Behavior |
|---|---|
| Message intake | User message enters chat endpoint. |
| Intent/topic resolution | System classifies intent, domain, topic, and possible ambiguity. |
| Domain fetch | DomainService retrieves topic, prerequisites, related concepts, learning items, and content context. |
| Student fetch | StudentModelService retrieves user topic state, item state, history, coding evidence, and profile summary. |
| TutorPlan generation | Tutor Administrator decides the teaching strategy and response contract. |
| LLM response generation | LLM receives TutorPlan, DomainContext, StudentSnapshot, and chat history, then writes the response. |
| Extraction | Extractor reads the turn and emits learning/context events. |
| Projection | Projectors update topic state, item state, profile summary, and review queues. |
| Persistence | Chat messages, events, and projections are saved. |

### 4.2 Main Architectural Change

Current pipeline:

User message → LLM decides tools/direct answer → LLM answers

Target pipeline:

User message → Domain + Student fetch → TutorPlan → LLM answers according to plan → extractor writes events → projectors update state

The key shift is that the LLM becomes the **response generator**, not the sole tutor decision-maker.

---

## 5. Three-Layer Architecture

## 5.1 Layer 1: Domain Layer

### 5.1.1 Purpose

The Domain Layer is the tutor’s structured understanding of what can be learned.

It answers:

- What domain does this question belong to?
- What topic is the user asking about?
- What prerequisites does this topic require?
- What subtopics are inside this topic?
- What examples, questions, cases, or content support this topic?
- What should be taught before or after this topic?
- What learning items can be used to teach or verify the topic?

### 5.1.2 Supported Domains

The design must support at least:

| Domain | Examples |
|---|---|
| DSA | arrays, graphs, DP, recursion, trees, heaps. |
| System Design | URL shortener, rate limiter, cache, feed, notification system. |
| Operating Systems | process, thread, deadlock, memory, scheduling, synchronization. |
| Networks | TCP, UDP, DNS, HTTP, TLS, load balancing. |
| Databases | indexes, transactions, isolation levels, replication, sharding. |
| SQL | joins, windows, grouping, CTEs, query optimization. |
| Data Engineering | Airflow, Spark, Kafka, batch pipelines, idempotency, data quality. |
| Backend Engineering | APIs, concurrency, auth, queues, observability. |
| Distributed Systems | consensus, replication, consistency, leader election. |
| Cloud / Infrastructure | storage, compute, networking, deployment, scaling. |
| Debugging | runtime errors, logs, tracing, performance bottlenecks. |

### 5.1.3 Domain Layer Tables

The minimum Postgres runtime domain layer should include:

| Table | Required? | Purpose |
|---|---:|---|
| `app.pt_areas` | Yes | High-level domains such as OS, DB, DSA, system design, data engineering. |
| `app.pt_topics` | Yes | Canonical topics and subtopics. Should support parent-child hierarchy. |
| `app.pt_concept_relationships` | Yes | Prerequisite, related, unlocks, and dependency relationships. |
| `app.learning_items` | Yes | Universal abstraction for things the user can learn, practice, read, solve, or be assessed on. |
| `catalog.coding_problems` | Existing | Coding problem catalog. Used as one source for learning items. |
| `knowledge_sources` | Port/Reuse | Source metadata for high-quality RAG content. |
| `knowledge_chunks` | Port/Reuse | RAG-ready text chunks mapped to areas/topics. |
| `round_questions` or migrated equivalent | Later/Optional | Interview-question evidence across coding, design, SQL, debugging, OS, networks, etc. |

### 5.1.4 Why `app.pt_topics` Should Also Represent Subtopics

SQLite currently has both `pt_topics` and `pt_subtopics`. For Postgres MVP, we should avoid duplicating complexity.

Recommended Postgres design:

- Use `app.pt_topics` for both topics and subtopics.
- Add or preserve fields such as `parent_topic_id` and `topic_type`.
- Use `topic_type` values such as `module`, `concept`, `subtopic`, `case`, `checkpoint`, `skill`.

This lets us represent:

- Operating Systems → Synchronization → Deadlock
- Databases → Indexes → B-Tree Indexes
- Data Engineering → Airflow → Idempotent DAGs
- System Design → Caching → Cache Invalidation

without requiring separate tables for every curriculum level.

### 5.1.5 DomainService Responsibilities

DomainService is the API over the domain graph and content layer.

| Responsibility | Description |
|---|---|
| Resolve domain | Determine whether the message belongs to OS, DSA, SQL, system design, etc. |
| Resolve topic | Map user phrasing to canonical topic. Example: “process deadlock” → `operating_systems.deadlock`. |
| Resolve ambiguity | Detect whether a term can mean multiple things. Example: “locks” in OS vs DB vs distributed systems. |
| Fetch prerequisites | Retrieve required topics from `pt_concept_relationships`. |
| Fetch topic structure | Retrieve parent, children, subtopics, and neighboring concepts. |
| Fetch learning items | Retrieve practice, explanation, checkpoint, case, or reading items. |
| Fetch RAG context | Retrieve knowledge chunks relevant to the topic and user query. |
| Fetch interview evidence | Retrieve interview questions or company patterns when needed. |
| Recommend next topics | Suggest follow-up topics based on graph and student readiness. |

### 5.1.6 DomainContext Output

DomainService should return a structured DomainContext object to the orchestrator.

Conceptual fields:

| Field | Meaning |
|---|---|
| `domain` | High-level area, such as `operating_systems`. |
| `target_topic` | Canonical topic being taught. |
| `resolved_confidence` | Confidence that the topic mapping is correct. |
| `prerequisites` | Required prerequisite topics. |
| `related_topics` | Neighboring concepts. |
| `subtopics` | Internal breakdown of the target topic. |
| `learning_items` | Candidate items to teach, practice, or assess. |
| `knowledge_context` | RAG snippets or summaries. |
| `interview_context` | Relevant interview signals, if any. |

---

## 5.2 Layer 2: Student Model Layer

### 5.2.1 Purpose

The Student Model Layer is the tutor’s understanding of the individual user.

It answers:

- Has this user seen this topic before?
- Has the user practiced it?
- Has the user demonstrated mastery?
- Has the user struggled with related prerequisites?
- Is this topic due for revision?
- What learning style or response depth does the user prefer?
- What should not be assumed about the user?

### 5.2.2 Current Production Gap

Current Postgres does not contain a durable topic-progress projection such as `app.pt_user_topic_progress`. It also does not contain an append-only learning event table such as `app.pt_learning_events`.

Because of that, the current chat pipeline cannot reliably adapt based on learning history. It can use chat history and coding submissions, but it does not have a clean topic-level student model.

### 5.2.3 Student Layer Tables

The minimum Postgres student model should include:

| Table | Required? | Purpose |
|---|---:|---|
| `app.pt_learning_events` | Yes | Append-only event ledger for learning evidence. |
| `app.pt_user_topic_progress` | Yes | Materialized user-topic state projection. |
| `app.user_learning_item_state` | Yes | Per-user state for each learning item. |
| `app.user_context_events` | Recommended | Append-only user context/prefs/events separate from learning mastery. |
| `app.user_profile_summary` | Recommended | Compact materialized profile used during tutoring. |
| `user_data.user_coding_problems` | Existing | Coding problem aggregate evidence. |
| `user_data.user_coding_submissions` | Existing | Submission-level coding evidence. |
| `app.chat_messages` | Existing | Conversational evidence and context. |
| `app.chat_feedback` | Existing | Quality feedback that can weight extracted learning events. |

### 5.2.4 Learning Events

`app.pt_learning_events` should be the append-only ledger of what happened pedagogically.

Examples:

| Event Type | Meaning |
|---|---|
| `taught` | Tutor explained a topic. |
| `covered` | Topic was covered enough to count as exposure. |
| `self_reported_known` | User claimed they already know something. Low-confidence evidence. |
| `diagnostic_passed` | User answered a diagnostic correctly. Stronger evidence. |
| `diagnostic_failed` | User failed a diagnostic. |
| `struggled` | User showed confusion or repeated failure. |
| `checkpoint_passed` | User passed a formal checkpoint. |
| `checkpoint_failed` | User failed a formal checkpoint. |
| `revised` | User reviewed a previously covered topic. |
| `plan_task_completed` | User completed a prep-plan task. |
| `coding_submission_accepted` | Coding submission passed. |
| `coding_submission_failed` | Coding attempt failed. |

The event table should be append-only. If a mistake is made, the event should be invalidated or corrected by a new event rather than mutated silently.

### 5.2.5 User Topic Progress

`app.pt_user_topic_progress` should be a projection over learning events and other evidence.

It should answer quickly:

- user’s status for topic,
- coverage score,
- readiness score,
- confidence score,
- attempt count,
- last seen time,
- last practiced time,
- next revision time,
- missing prerequisites,
- misconception signals.

This table should not store raw conversation transcripts. It should store compact state.

Recommended statuses:

| Status | Meaning |
|---|---|
| `not_started` | No meaningful evidence. |
| `seen` | User has encountered the topic. |
| `taught` | Tutor has explained it. |
| `attempted` | User has tried to answer/solve/apply it. |
| `struggled` | User has shown difficulty. |
| `covered` | Topic has been taught sufficiently for exposure. |
| `needs_review` | Topic should be revisited. |
| `verified` | User demonstrated understanding through assessment or successful application. |

### 5.2.6 StudentModelService Responsibilities

| Responsibility | Description |
|---|---|
| Fetch topic state | Retrieve `pt_user_topic_progress` for target and prerequisite topics. |
| Fetch item state | Retrieve `user_learning_item_state` for relevant learning items. |
| Fetch recent events | Retrieve recent learning events for the user/topic/domain. |
| Fetch coding evidence | Use `user_coding_problems` and `user_coding_submissions` for DSA/coding topics. |
| Fetch profile summary | Retrieve role, experience, goals, preferences, and recent focus. |
| Estimate prerequisite readiness | Decide which prerequisites are known, unknown, weak, or verified. |
| Build StudentSnapshot | Produce a compact snapshot for the Tutor Administrator and final LLM prompt. |
| Record learning events | Persist extracted learning evidence. |
| Update projections | Trigger or perform topic/item state updates through projectors. |

### 5.2.7 StudentSnapshot Output

StudentModelService should return a StudentSnapshot.

Conceptual fields:

| Field | Meaning |
|---|---|
| `user_id` | User identity. |
| `target_topic_state` | Current state for the requested topic. |
| `prerequisite_states` | Readiness for prerequisite topics. |
| `relevant_item_states` | User status on candidate learning items. |
| `recent_learning_events` | Recent topic/domain evidence. |
| `known_strengths` | Strong domains/topics. |
| `known_weaknesses` | Weak domains/topics. |
| `misconception_signals` | Known misconceptions or failure modes. |
| `preferred_style` | Concise, detailed, Socratic, example-heavy, etc. |
| `confidence` | How reliable this snapshot is. |

### 5.2.8 Handling Cold Start

If no state exists for a topic, the StudentSnapshot should explicitly say:

- target topic state: unknown,
- prerequisite state: unknown,
- evidence count: zero,
- confidence: low.

The system should not assume the user knows prerequisites. It should also not block the user with a long quiz. It should either:

- ask one lightweight diagnostic question,
- provide a scaffolded explanation,
- or ask the user whether they want prerequisite review.

---

## 5.3 Layer 3: Tutor Administrator / Orchestrator Service

### 5.3.1 Purpose

The Tutor Administrator, also called the Tutor Orchestrator, is the decision-making layer.

It answers:

- What is the user trying to do?
- What topic/domain is involved?
- Is the user ready for the requested topic?
- Should the answer be direct, scaffolded, diagnostic, Socratic, or practice-based?
- What context should the LLM receive?
- What should be extracted and updated after the response?

### 5.3.2 Why This Layer Is Needed

Without this layer, the LLM decides everything from its prompt and chat history. That creates inconsistent tutoring behavior.

The orchestrator gives the system control over pedagogy.

It turns the LLM from:

- “general coach that decides everything”

into:

- “response generator operating inside a structured tutoring plan.”

### 5.3.3 TutorPlan Sheet

The main output of the Tutor Administrator is a TutorPlan Sheet.

The TutorPlan is not a long-term prep plan. It is a per-turn or short-session instruction sheet for how to respond right now.

It includes:

| Field | Purpose |
|---|---|
| `intent` | What the user is trying to do. |
| `domain` | Domain such as OS, DB, DSA, data engineering. |
| `target_topic` | Canonical topic being addressed. |
| `prerequisites` | Required prerequisite topics. |
| `student_state_summary` | Compact readiness summary. |
| `strategy` | Teaching strategy for the response. |
| `response_contract` | What the LLM must do or avoid. |
| `context_to_use` | Which domain/student/RAG context to include. |
| `state_update_expectations` | What the extractor should watch for after the turn. |

### 5.3.4 Intent Types

Intent identifies what the user wants.

Recommended initial intent set:

| Intent | Meaning |
|---|---|
| `concept_explanation` | User wants to understand a concept. |
| `prerequisite_check` | User asks if they need to know something first. |
| `diagnostic_answer` | User answers or attempts a concept check. |
| `practice_request` | User asks for practice questions or exercises. |
| `problem_solving` | User wants help solving a specific problem. |
| `code_review_or_debugging` | User provides code or asks for debugging help. |
| `system_design_case` | User wants to design or evaluate a system. |
| `comparison_or_tradeoff` | User asks for differences, tradeoffs, or when-to-use guidance. |
| `roadmap_or_prep_plan` | User asks for a plan or roadmap. |
| `progress_or_readiness_check` | User asks if they are ready or how they are doing. |
| `revision_request` | User wants to review something learned before. |
| `resource_recommendation` | User asks for resources. |
| `interview_intelligence` | User asks what companies ask or how interviews look. |
| `off_topic_or_meta` | User asks account/app/meta questions. |

Intent should not encode the domain. Domain should be a separate field.

For example:

| User Message | Intent | Domain | Topic |
|---|---|---|---|
| “What is deadlock?” | `concept_explanation` | `operating_systems` | `deadlock` |
| “TCP vs UDP?” | `comparison_or_tradeoff` | `networks` | `tcp_udp` |
| “Design a rate limiter” | `system_design_case` | `system_design` | `rate_limiter` |
| “Give me Spark questions” | `practice_request` | `data_engineering` | `spark` |
| “Review this SQL query” | `code_review_or_debugging` | `sql` | `query_optimization` |

### 5.3.5 Strategy Types

Strategy determines how the tutor should respond.

Recommended initial strategy set:

| Strategy | When to Use |
|---|---|
| `direct_explanation` | User is likely ready and asked for explanation. |
| `scaffolded_explanation` | User state is unknown; explain with minimal prerequisites included. |
| `prerequisite_probe` | Required prerequisites are unknown and important. |
| `prerequisite_repair` | User is missing prerequisite knowledge. |
| `socratic_guidance` | User is solving and should be guided rather than given the answer. |
| `guided_practice` | User needs practice with hints and feedback. |
| `misconception_correction` | User shows a wrong mental model. |
| `retrieval_grounded_answer` | Answer should use trusted corpus/RAG/interview evidence. |
| `assessment_checkpoint` | Need to verify mastery. |
| `revision_or_spaced_repetition` | Topic was learned before and is due for review. |
| `prep_plan_generation` | User explicitly asks for a durable plan. |

### 5.3.6 TutorPlan Generation: Rules + LLM, Not LLM Alone

TutorPlan generation should be hybrid.

The LLM can help classify intent, map fuzzy user wording to topics, and propose diagnostic questions. But the final TutorPlan should be validated by deterministic service logic.

The orchestrator should not allow the LLM to invent:

- mastery scores,
- verified prerequisites,
- database facts,
- nonexistent learning history,
- nonexistent curriculum relationships.

The correct flow is:

1. Use an LLM or classifier to classify intent and candidate topic.
2. Use DomainService to fetch the actual domain graph and content.
3. Use StudentModelService to fetch actual user state.
4. Use rules to select readiness and strategy.
5. Optionally use LLM to draft a structured TutorPlan proposal.
6. Validate the plan against allowed enums and retrieved facts.
7. Pass the validated TutorPlan to the final response-generation LLM.

---

## 6. Universal Learning Tables

The tutor needs a universal abstraction for “things the user can learn, practice, or be assessed on.” This cannot be only LeetCode problems.

A learning item can be:

- coding problem,
- system design case,
- SQL exercise,
- debugging task,
- OS concept check,
- network concept check,
- data engineering scenario,
- reading item,
- checkpoint,
- interview question,
- project/lab.

## 6.1 `app.learning_items`

### Purpose

`app.learning_items` is the universal catalog of teachable/practicable/verifiable units.

It bridges many sources into one tutor-facing table:

| Source | Example Learning Item |
|---|---|
| `catalog.coding_problems` | “Coin Change” or “Two Sum.” |
| `pt_system_design_cases` | “Design a rate limiter.” |
| `pt_checkpoints` | “Deadlock diagnosis checkpoint.” |
| `knowledge_chunks` | “Read Raft leader election explanation.” |
| `round_questions` | “Uber SQL window function question.” |
| Manual curriculum | “Explain OS deadlock conditions.” |

### Conceptual Fields

| Field | Purpose |
|---|---|
| `id` | Primary identifier. |
| `area_slug` | Domain such as OS, DB, DSA, system design. |
| `primary_topic_id` | Main topic this item belongs to. |
| `topic_ids_json` | Additional related topics/subtopics. |
| `item_type` | Concept, coding problem, system design case, SQL exercise, checkpoint, reading, etc. |
| `title` | Human-readable item title. |
| `difficulty` | Beginner, intermediate, advanced, L4, L5, staff, etc. |
| `verification_type` | How mastery is verified. |
| `source_type` | Where the item came from. |
| `source_ref` | Reference to source row or external object. |
| `estimated_minutes` | Estimated time to complete. |
| `is_active` | Whether item is active. |
| `metadata_json` | Rubrics, tags, constraints, company metadata, etc. |

### Recommended Item Types

| Item Type | Example |
|---|---|
| `concept` | “Deadlock conditions.” |
| `coding_problem` | “LeetCode Coin Change.” |
| `system_design_case` | “Design a notification system.” |
| `sql_exercise` | “Find second highest salary with window functions.” |
| `debugging_task` | “Find deadlock in lock-ordering code.” |
| `quiz` | “OS synchronization quick quiz.” |
| `checkpoint` | “Verify deadlock understanding.” |
| `reading` | “Read TCP congestion control notes.” |
| `project_lab` | “Build idempotent ingestion pipeline.” |
| `interview_question` | “Past Google OS/threading question.” |

### Recommended Verification Types

| Verification Type | Meaning |
|---|---|
| `none` | Exposure only. |
| `read` | User read the item. |
| `self_explain` | User must explain the concept back. |
| `quiz` | User answers one or more questions. |
| `code_acceptance` | Coding platform accepts solution. |
| `rubric_eval` | LLM/human rubric evaluation. |
| `mock_interview` | Verified through mock interview interaction. |

## 6.2 `app.user_learning_item_state`

### Purpose

`app.user_learning_item_state` stores the user’s state for each learning item.

`app.pt_user_topic_progress` says:

- user is weak in deadlocks,
- user has covered indexes,
- user is verified in recursion.

`app.user_learning_item_state` says:

- user attempted the deadlock diagnostic twice,
- user solved Coin Change after three attempts,
- user struggled with lock ordering,
- user completed the rate limiter design case,
- user needs to review SQL window functions tomorrow.

### Conceptual Fields

| Field | Purpose |
|---|---|
| `user_id` | User identity. |
| `learning_item_id` | Learning item identity. |
| `status` | User’s current state for the item. |
| `mastery_signal` | Numeric signal for mastery. |
| `confidence` | Confidence in the signal. |
| `attempt_count` | Number of attempts. |
| `success_count` | Number of successful attempts. |
| `hint_count` | Number of hints used. |
| `best_score` | Best rubric/quiz/checkpoint score. |
| `last_score` | Most recent score. |
| `first_seen_at` | When item was first shown. |
| `first_attempted_at` | When user first attempted item. |
| `first_verified_at` | When item was first verified. |
| `last_interaction_at` | Most recent interaction. |
| `next_review_at` | Scheduled review time. |
| `misconception_signals_json` | Item-level misconceptions. |
| `evidence_refs_json` | References to events, submissions, messages, attempts. |
| `metadata_json` | Additional flexible state. |

### Recommended Statuses

| Status | Meaning |
|---|---|
| `not_started` | User has not interacted with the item. |
| `seen` | Item was shown or mentioned. |
| `attempted` | User attempted it. |
| `struggled` | User had difficulty. |
| `completed` | User completed it. |
| `verified` | User demonstrated mastery. |
| `skipped` | User skipped it. |
| `needs_review` | Item should be revisited. |

---

## 7. Updating the Current Chat Pipeline

## 7.1 New Runtime Components

The existing chat endpoint should be updated to call these new services before the final LLM response:

| Component | Responsibility |
|---|---|
| Message Intake | Accept user message and session context. |
| Intent/Topic Resolver | Classify intent, domain, and candidate topic. |
| DomainService | Fetch domain graph, prerequisites, learning items, RAG context. |
| StudentModelService | Fetch user state, events, item state, profile summary. |
| Tutor Administrator / Orchestrator | Build TutorPlan and response contract. |
| Response LLM | Generate final answer using TutorPlan, context, and snapshot. |
| Extractor | Extract learning/context events from the completed turn. |
| Projector | Update topic progress, item state, profile summary, and review scheduling. |

## 7.2 New Chat Pipeline: Detailed Flow

### Step 1: User sends message

The user sends a message such as:

“What is deadlock?”

The chat system stores or stages the incoming message and attaches the current user/session identifiers.

### Step 2: Intent and topic resolution

The system identifies:

| Field | Example |
|---|---|
| Intent | `concept_explanation` |
| Domain | `operating_systems` |
| Target topic | `deadlock` |
| Ambiguity | Low |

If the message is ambiguous, the resolver should return candidates. For example, “locks” may mean OS locks, database locks, distributed locks, or language-level mutexes.

### Step 3: DomainService fetches DomainContext

For “deadlock,” DomainService retrieves:

- canonical topic: deadlock,
- domain: operating systems,
- prerequisites: process, thread, resource, lock, blocking wait, synchronization,
- subtopics: circular wait, hold-and-wait, mutual exclusion, no preemption, prevention, avoidance, detection,
- learning items: concept explanation, diagnostic question, checkpoint, debugging scenario,
- RAG context: trusted OS/system content if available.

### Step 4: StudentModelService fetches StudentSnapshot

StudentModelService checks Postgres state:

- topic state for deadlock,
- topic states for prerequisites,
- user’s recent OS-related learning events,
- item state for related learning items,
- coding or debugging history if relevant,
- user profile summary and current learning preferences.

If there is no data, the snapshot explicitly marks the state as unknown rather than assuming mastery or weakness.

### Step 5: Tutor Administrator builds TutorPlan

Tutor Administrator combines:

- user message,
- intent,
- DomainContext,
- StudentSnapshot,
- chat session context.

For unknown prerequisite state, it may choose:

| Field | Value |
|---|---|
| Intent | `concept_explanation` |
| Strategy | `prerequisite_probe` or `scaffolded_explanation` |
| Response contract | Ask one lightweight prerequisite check, then explain. |
| State update expectation | Watch for self-reported prerequisite knowledge and diagnostic answer. |

### Step 6: LLM receives TutorPlan, context, and student snapshot

The final response LLM receives:

| Input | Purpose |
|---|---|
| User message | The direct request. |
| Chat history | Local conversation continuity. |
| TutorPlan | Binding teaching strategy and response contract. |
| DomainContext | Topic, prerequisites, content, examples, learning items. |
| StudentSnapshot | User readiness, weaknesses, history, preferences. |
| Retrieved content | Knowledge snippets or interview evidence if needed. |

The LLM should follow the TutorPlan. It should not override the strategy unless the plan is invalid or unsafe.

### Step 7: LLM answers

For “What is deadlock?” with no prior state, a good response might:

- briefly check prerequisite understanding,
- explain process/resource/wait relationship,
- define deadlock,
- give a simple example,
- ask one diagnostic follow-up.

The answer should not create a full durable prep plan unless the user asked for one.

### Step 8: Extractor writes learning/context events

After the answer, an extractor reads the completed turn and emits structured events.

Potential extracted events:

| Event | Example |
|---|---|
| Learning event | Tutor taught deadlock. |
| Context event | User prefers concise OS explanations. |
| Self-report event | User says prerequisites are clear. |
| Diagnostic event | User answers the follow-up correctly/incorrectly. |
| Misconception event | User confuses deadlock with starvation. |

The extractor should distinguish between:

- what the tutor taught,
- what the user demonstrated,
- what the user merely claimed,
- what remains unknown.

### Step 9: Projector updates state

Projectors consume extracted events and update materialized state tables.

| Event | Projection Update |
|---|---|
| `taught` | Increase coverage slightly. Mark topic as seen/taught. |
| `self_reported_known` | Low-confidence readiness increase for prerequisite. |
| `diagnostic_passed` | Medium/high-confidence readiness increase. |
| `diagnostic_failed` | Mark missing prerequisite or misconception. |
| `struggled` | Lower readiness, add weakness signal, schedule review. |
| `checkpoint_passed` | Mark topic verified if score is strong enough. |
| `coding_submission_accepted` | Mark coding item verified and update related topic progress. |
| `coding_submission_failed` | Mark item attempted/struggled and capture failure mode. |

### Step 10: State is available for the next turn

The next user message is now more personalized.

If the user later asks:

“Explain deadlock prevention.”

The system can see:

- deadlock was taught recently,
- prerequisites were self-reported but not verified,
- circular wait may not be verified,
- the previous diagnostic answer may have shown confusion.

Then the tutor can choose a better strategy.

---

## 8. Example End-to-End Flow: “What is Deadlock?”

### 8.1 Initial User Message

User asks:

“What is deadlock?”

### 8.2 Intent/Topic Resolution

| Field | Value |
|---|---|
| Intent | `concept_explanation` |
| Domain | `operating_systems` |
| Topic | `deadlock` |
| Confidence | High |

### 8.3 DomainService Output

DomainService returns:

| Field | Value |
|---|---|
| Target topic | Deadlock |
| Prerequisites | Process, thread, resource, lock, blocking wait, synchronization |
| Subtopics | Four Coffman conditions, prevention, avoidance, detection, recovery |
| Learning items | Deadlock concept item, quick diagnostic, checkpoint, lock-ordering debugging task |
| Content | OS knowledge chunks, examples, diagrams if available |

### 8.4 StudentModelService Output

If no state exists:

| Field | Value |
|---|---|
| Target topic state | Unknown |
| Prerequisite state | Unknown |
| Evidence count | 0 |
| Confidence | Low |

### 8.5 TutorPlan

| Field | Value |
|---|---|
| Intent | `concept_explanation` |
| Strategy | `prerequisite_probe` or `scaffolded_explanation` |
| Response style | Brief check, then explanation. |
| Max diagnostic questions | 1 |
| Do not create durable prep plan | True |
| State update expectation | Capture self-report and diagnostic result. |

### 8.6 User Confirms Prerequisites

If user says:

“Yes, I know processes and locks.”

The system should not mark prerequisites as verified. It should write a low-confidence event:

| Field | Value |
|---|---|
| Event type | `self_reported_known` |
| Topics | Process, lock, resource basics |
| Confidence | Low to medium |
| Source | User self-report |

Then the tutor proceeds to explain deadlock.

### 8.7 Verification

After explanation, the tutor asks one check:

“Suppose A holds Lock 1 and waits for Lock 2, while B holds Lock 2 and waits for Lock 1. What happens?”

If the user answers correctly, the system writes stronger evidence.

If the user answers incorrectly, the system records a misconception or missing prerequisite.

### 8.8 Prep Plan Decision

A durable prep plan should **not** be created for this ordinary concept question.

Create or update `app.user_prep_plans` only when:

- the user explicitly asks for a plan,
- onboarding/calibration generates one,
- the user changes target role/company/date,
- the user asks for a multi-week roadmap,
- or the product triggers a deliberate reorientation flow.

---

## 9. Required Postgres Table Changes

The goal is to add the smallest set of tables that enables the tutor architecture.

## 9.1 Minimum Required Tables to Add or Port

| Table | Layer | Why Needed |
|---|---|---|
| `app.pt_areas` | Domain | Defines high-level domains. |
| `app.pt_topics` | Domain | Defines topics/subtopics/modules. |
| `app.pt_concept_relationships` | Domain | Defines prerequisites and related concepts. |
| `app.pt_learning_events` | Student | Append-only learning evidence. |
| `app.pt_user_topic_progress` | Student | Fast user-topic mastery/readiness projection. |
| `app.learning_items` | Domain + Student bridge | Universal learnable/practicable/verifiable item catalog. |
| `app.user_learning_item_state` | Student | Per-user state for each learning item. |

This is the minimum practical foundation.

## 9.2 Recommended Tables to Add Soon After

| Table | Layer | Why Needed |
|---|---|---|
| `app.user_context_events` | Student/context | Separate durable user facts/preferences from learning mastery. |
| `app.user_profile_summary` | Student/context | Fast profile snapshot for chat personalization. |
| `app.knowledge_sources` | Domain/content | Move trusted RAG source metadata to Postgres runtime. |
| `app.knowledge_chunks` | Domain/content | Move trusted RAG chunks to Postgres runtime. |

These are strongly recommended, but the absolute MVP can start without fully migrating all RAG tables if the first version focuses on structured tutoring and basic content.

## 9.3 Existing Tables to Reuse

| Existing Table | How to Use |
|---|---|
| `public.users` | Identity anchor for all user-specific state. |
| `app.chat_sessions` | Session container. |
| `app.chat_messages` | Raw conversation record and trace reference. |
| `app.chat_feedback` | Quality signal for response and extractor weighting. |
| `app.user_prep_plans` | Durable multi-day or multi-week plans. |
| `app.user_prep_task_progress` | Task-level progress inside a durable plan. |
| `catalog.coding_problems` | Source for coding learning items. |
| `user_data.user_coding_problems` | Aggregate evidence for coding progress. |
| `user_data.user_coding_submissions` | Strong evidence from actual submissions. |
| `code_blobs.user_coding_submission_code` | Optional deep code review/debugging evidence with privacy controls. |

---

## 10. Data Flow: Events and Projectors

## 10.1 Why Event + Projection

The tutor should separate raw evidence from current state.

Raw evidence:

- user asked a question,
- tutor explained a topic,
- user self-reported knowledge,
- user answered a diagnostic,
- user solved a problem,
- user failed a checkpoint,
- user used a hint.

Current state:

- user is probably ready for deadlock prevention,
- user is weak in lock ordering,
- user has verified recursion,
- user should review SQL window functions tomorrow.

Raw evidence belongs in event tables. Current state belongs in projection tables.

## 10.2 Event Tables

| Table | Event Type |
|---|---|
| `app.pt_learning_events` | Learning/mastery evidence. |
| `app.user_context_events` | User facts, preferences, goals, context. |
| `app.chat_messages` | Raw conversational record. |
| `user_data.user_coding_submissions` | Coding submission evidence. |

## 10.3 Projection Tables

| Table | Projection |
|---|---|
| `app.pt_user_topic_progress` | Topic-level readiness/mastery. |
| `app.user_learning_item_state` | Item-level attempts/completion/verification. |
| `app.user_profile_summary` | Compact user profile and preferences. |
| `app.user_prep_task_progress` | Durable prep-plan task progress. |

## 10.4 Projector Responsibilities

| Projector | Reads | Writes |
|---|---|---|
| Topic Progress Projector | `pt_learning_events`, coding evidence, checkpoint events | `pt_user_topic_progress` |
| Item State Projector | `pt_learning_events`, coding submissions, item interactions | `user_learning_item_state` |
| Profile Summary Projector | `user_context_events`, chat feedback, profile signals | `user_profile_summary` |
| Prep Plan Projector | learning events, item completions | `user_prep_task_progress` |
| Review Scheduler | topic/item state, event recency | `next_revision_at`, `next_review_at` |

---

## 11. Detailed Service Design

## 11.1 DomainService

### Purpose

DomainService turns a raw user topic into structured curriculum context.

### Inputs

| Input | Description |
|---|---|
| User message | Raw text from user. |
| Candidate domain/topic | Output from resolver. |
| Target role/level | Optional user goal such as backend L5, data engineer, staff. |
| Retrieval mode | Whether to fetch concepts, practice, RAG, interview evidence. |

### Outputs

| Output | Description |
|---|---|
| DomainContext | Canonical topic, prerequisites, related concepts, learning items, content. |
| Topic confidence | Confidence in topic mapping. |
| Ambiguity list | Alternative interpretations if needed. |

### Core Methods Conceptually

| Method | Purpose |
|---|---|
| Resolve topic | Map user phrase to canonical topic. |
| Get prerequisites | Fetch required topics. |
| Get related topics | Fetch adjacent/unlocked/related topics. |
| Get learning items | Retrieve relevant learning items. |
| Get knowledge context | Retrieve trusted content chunks. |
| Get interview evidence | Retrieve relevant interview questions or company signals. |

## 11.2 StudentModelService

### Purpose

StudentModelService creates a user-specific view of readiness and history.

### Inputs

| Input | Description |
|---|---|
| User ID | User identity. |
| Target topic | Topic being addressed. |
| Prerequisite topics | Required topics from DomainService. |
| Candidate learning items | Items from DomainService. |
| Session ID | Current chat session. |

### Outputs

| Output | Description |
|---|---|
| StudentSnapshot | Compact learner-state object. |
| Readiness report | Target and prerequisite readiness. |
| Item-state report | Status of relevant items. |
| Evidence summary | Events/submissions/messages supporting the state. |

### Core Methods Conceptually

| Method | Purpose |
|---|---|
| Get topic progress | Fetch topic state for target and prerequisites. |
| Get item states | Fetch per-user state for candidate items. |
| Get recent events | Retrieve recent learning evidence. |
| Get coding evidence | Use coding submissions/problem status. |
| Estimate readiness | Produce readiness classification and confidence. |
| Record learning event | Persist extracted learning event. |
| Trigger projections | Update materialized state after events. |

## 11.3 Tutor Administrator / Orchestrator Service

### Purpose

The Tutor Administrator makes the tutoring decision.

### Inputs

| Input | Description |
|---|---|
| User message | Raw user request. |
| Intent/topic classification | What the user is asking. |
| DomainContext | Domain graph and content. |
| StudentSnapshot | User-specific readiness. |
| Chat context | Recent conversation. |
| Product context | Plan/session settings, target role, feature flags. |

### Outputs

| Output | Description |
|---|---|
| TutorPlan | Structured plan for this response. |
| Response contract | Constraints for the LLM. |
| Retrieval contract | Which context to include. |
| Extraction contract | What to watch for after the response. |

### Decision Rules

| Situation | Strategy |
|---|---|
| Topic known and prerequisites verified | `direct_explanation` |
| Topic unknown and prerequisites unknown | `scaffolded_explanation` or `prerequisite_probe` |
| Prerequisites weak | `prerequisite_repair` |
| User is solving a problem | `socratic_guidance` or `guided_practice` |
| User asks for design | `system_design_case` intent + `guided_practice` or `assessment_checkpoint` |
| User shows misconception | `misconception_correction` |
| Topic due for review | `revision_or_spaced_repetition` |
| User asks for roadmap | `prep_plan_generation` |

---

## 12. How Durable Prep Plans Fit In

A TutorPlan and a Prep Plan are different.

| Concept | Scope | Stored? | Example |
|---|---|---:|---|
| TutorPlan | One turn or short session | Usually not as first-class table initially | “For this answer, probe prerequisites before explaining deadlock.” |
| Prep Plan | Multi-day/multi-week roadmap | Yes, in `app.user_prep_plans` and `app.user_prep_task_progress` | “Prepare OS + DB + system design for backend interviews over 30 days.” |

A normal concept question should not create a durable prep plan.

Create a durable prep plan only when:

- user explicitly asks for a plan,
- onboarding creates a plan,
- user changes goals,
- user asks for roadmap/schedule,
- product triggers a reorientation flow.

---

## 13. Migration Plan

## 13.1 Phase 1: Establish Postgres Tutor Foundation

Add or port:

- `app.pt_areas`,
- `app.pt_topics`,
- `app.pt_concept_relationships`,
- `app.pt_learning_events`,
- `app.pt_user_topic_progress`,
- `app.learning_items`,
- `app.user_learning_item_state`.

Seed areas and topics from SQLite.

Start with a small but representative domain set:

- operating systems,
- system design,
- databases/SQL,
- data engineering,
- DSA.

## 13.2 Phase 2: Backfill Universal Learning Items

Backfill learning items from:

- `catalog.coding_problems`,
- SQLite `pt_checkpoints`,
- SQLite `pt_system_design_cases`,
- SQLite `knowledge_chunks`,
- curated manual curriculum seeds.

Do not try to backfill every possible item on day one. Start with enough items to support real tutor behavior.

## 13.3 Phase 3: Build DomainService and StudentModelService

Implement read services first:

- resolve topic,
- fetch prerequisites,
- fetch learning items,
- fetch user topic state,
- fetch item state,
- build StudentSnapshot.

## 13.4 Phase 4: Build Tutor Administrator

Introduce TutorPlan generation behind a feature flag.

Start with a limited set of intents and strategies:

- `concept_explanation`,
- `practice_request`,
- `problem_solving`,
- `system_design_case`,
- `roadmap_or_prep_plan`,
- `direct_explanation`,
- `scaffolded_explanation`,
- `prerequisite_probe`,
- `guided_practice`,
- `prep_plan_generation`.

## 13.5 Phase 5: Update Chat Pipeline

Modify chat pipeline so that the final LLM receives:

- TutorPlan,
- DomainContext,
- StudentSnapshot,
- selected retrieved content,
- normal chat history.

The LLM should be instructed to follow the TutorPlan and not invent missing state.

## 13.6 Phase 6: Add Extractor and Projectors

After each assistant turn and user response:

- extractor emits learning/context events,
- topic projector updates `pt_user_topic_progress`,
- item projector updates `user_learning_item_state`,
- profile projector updates `user_profile_summary`, if present.

## 13.7 Phase 7: Expand Domains and Quality

Expand coverage to:

- more OS topics,
- data engineering scenarios,
- system design cases,
- SQL exercises,
- network fundamentals,
- distributed systems,
- cloud/backend topics.

Add dashboards/debugging views to inspect TutorPlans, event extraction, and projection changes.

---

## 14. What We Are Not Building Now

To keep the system focused, do not create these tables in the first implementation unless a specific product need appears:

| Deferred Table/Layer | Why Deferred |
|---|---|
| `app.user_misconceptions` | Store misconceptions in learning events and state JSON first. Promote later if needed. |
| `app.tutor_plan_logs` | Store TutorPlan in `chat_messages.tool_calls` or API logs initially. Add table only for analytics/debugging. |
| `app.assessment_rubrics` | Use `learning_items.metadata_json` first. Add table when rubrics become large/reusable. |
| `app.learning_item_topic_map` | Use `topic_ids_json` first. Add join table only if querying becomes complex. |
| `app.pt_learning_paths` | Use concept relationships plus existing prep-plan tables first. |
| `app.pt_checkpoint_questions` | Represent checkpoint questions as learning items first. Split later if assessment product matures. |
| Separate subtopic table | Use `pt_topics.parent_topic_id` and `topic_type` first. |

---

## 15. Final Target Architecture Summary

The final system should work like this:

1. User asks a question.
2. System resolves the intent, domain, and topic.
3. DomainService fetches topic graph, prerequisites, learning items, and content.
4. StudentModelService fetches user readiness, history, item state, and profile context.
5. Tutor Administrator creates a validated TutorPlan.
6. LLM receives TutorPlan, DomainContext, StudentSnapshot, and selected context.
7. LLM answers according to the teaching strategy.
8. Extractor writes learning/context events.
9. Projectors update topic progress, item state, profile summary, and review scheduling.
10. The next user turn becomes more personalized.

The core principle is:

**Keep the LLM excellent at language and explanation, but make the platform responsible for tutoring decisions, state, evidence, and progression.**

---

## 16. Minimal Build Checklist

| Work Item | Required for MVP? |
|---|---:|
| Port `pt_areas` to Postgres | Yes |
| Port `pt_topics` to Postgres | Yes |
| Port `pt_concept_relationships` to Postgres | Yes |
| Add `pt_learning_events` | Yes |
| Add `pt_user_topic_progress` | Yes |
| Add `learning_items` | Yes |
| Add `user_learning_item_state` | Yes |
| Build DomainService | Yes |
| Build StudentModelService | Yes |
| Build Tutor Administrator | Yes |
| Update chat pipeline to use TutorPlan | Yes |
| Add extractor | Yes |
| Add projectors | Yes |
| Add durable prep-plan generation | Partially, reuse existing tables |
| Add full checkpoint product | Later |
| Add misconception analytics table | Later |
| Add tutor-plan analytics table | Later |


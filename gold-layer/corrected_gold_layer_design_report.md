# CrackedIn Corrected Gold Layer Design Report

**Project:** CrackedIn / InterviewPrep.ai  
**Document type:** Corrected product + architecture design report  
**Purpose:** Reconcile the proposed Gold Layer with the existing AI Tutor architecture and remove duplicate personalization/readiness logic.  
**Date:** 2026-07-06  

---

## 1. Executive Summary

The previous Gold Layer design is directionally right, but it overreaches in one critical area: it makes Gold responsible for personalized readiness, personalized recommendations, prep-path generation, and user-specific dashboard state.

That responsibility already belongs to the AI Tutor architecture.

The corrected boundary is:

> **Gold tells CrackedIn what the market is asking. The AI Tutor tells a specific user what to do next.**

Gold should remain the product-facing market intelligence layer built from Bronze/Silver interview data. It should canonicalize, aggregate, score, and expose interview-market demand signals:

- Recently asked company-wise questions
- Role-level and level-wise question trends
- DSA, system design, LLD, SQL, behavioral, debugging, ML, and domain-technical trends
- Canonical question and question-family intelligence
- Platform mappings such as LeetCode, Codeforces, HackerRank, and custom company variants
- Topic/skill demand trends
- Interview loop profiles
- Failure-reason and candidate-mistake trends
- Evidence-backed cards
- Confidence and fallback labels
- Change detection
- Structured + semantic interview evidence search

The AI Tutor should own user-specific interpretation:

- User readiness
- User weaknesses
- User learning state
- User memory
- User target context
- Personalized gap analysis
- Personalized prep paths
- Revision priority
- Next-task recommendations
- Mock/practice generation strategy
- User-facing recommendation persistence

This avoids building two personalization engines.

---

## 2. Core Correction

### 2.1 Original Problem in the Gold Report

The original Gold report says Gold should answer:

- What should this user study next?
- What is the gap between the user's readiness and the target company's demand?
- Which questions are trending but not yet practiced by the user?
- Which topics can the user safely deprioritize?
- Which company is the user most ready for?

Those are valid product questions, but they should not be answered directly by Gold.

They require user-specific state:

- User solved/attempted history
- Independent vs assisted solving
- Skill readiness
- Learning coverage
- Staleness/revision state
- User targets
- User memory and preferences
- Prep-plan context
- Checkpoint performance
- Code-submission evidence

The AI Tutor architecture already models this through the Learner Skill Graph, StudentSnapshot, UserMemorySnapshot, TutorPlan, learning events, projections, and projectors.

### 2.2 Corrected Responsibility Split

| Concern | Correct Owner | Reason |
|---|---|---|
| Raw ingest lifecycle | Bronze | Operational crawl/hydration/extraction state. |
| Extracted interview facts | Silver | Source of truth for reported interview facts. |
| Canonical market intelligence | Gold | Product-optimized derived intelligence from Silver. |
| Skill taxonomy and prerequisites | AI Tutor Skill Graph | Already owns canonical learning skills, aliases, relationships, and learning items. |
| User learning state | AI Tutor Learner Skill Graph | Already owns readiness, coverage, attempts, verified/stale status. |
| User memory/preferences | AI Tutor UserMemoryService / app memory tables | Already owns target, style, goals, preferences, recent focus. |
| Personalized readiness | AI Tutor / app product layer | Requires Gold demand + StudentSnapshot. |
| Personalized recommendations | AI Tutor / app recommendation layer | Requires Gold demand + user learning state + prep-plan context. |
| Watchlist subscriptions and notification state | App/product layer | User-specific retention state; Gold only supplies trend-change signals. |
| Mock generation | AI Tutor Creator / Orchestrator | Gold supplies market pattern; Tutor generates practice artifact. |

---

## 3. Corrected Product Promise

The Gold Layer should allow CrackedIn to say:

> “For a target company, role, level, and time window, we can show what is actually being asked recently, what topics and question families are rising, what the interview loop looks like, what candidates fail on, how confident the signal is, and what evidence supports it.”

The combined product should allow CrackedIn to say:

> “Given your target and current learning state, the AI Tutor will use Gold market demand to decide what you personally should prepare next.”

The second statement is not Gold alone. It is the result of:

```text
Gold Market Demand Snapshot
+ AI Tutor StudentSnapshot
+ UserMemorySnapshot
+ Prep-plan context
+ Coding evidence
= Personalized action
```

---

## 4. Corrected Architecture Boundary

### 4.1 Full System Flow

```text
External interview sources
  -> Bronze ingest tracking
  -> Silver normalized interview facts
  -> Gold market intelligence
  -> Product APIs / AI Tutor tools

User activity and learning evidence
  -> App/user tables
  -> AI Tutor Learner Skill Graph
  -> StudentSnapshot / UserMemorySnapshot

AI Tutor runtime
  -> Retrieves Gold market demand when intent requires interview intelligence
  -> Combines it with StudentSnapshot
  -> Produces response, prep guidance, practice, or recommendation
  -> Emits learning/context events
  -> Projectors update learner state
```

### 4.2 Gold Does Not Replace the Tutor

Gold should not become a second orchestrator. It should not maintain user mastery, user weakness, user memory, user profile, or a user-specific learning graph.

Gold should expose stable, queryable, explainable market signals. The Tutor should consume those signals as one input to its per-turn or per-plan decision.

---

## 5. What Gold Should Still Answer

Gold should answer market-facing and corpus-facing questions.

### 5.1 Question Intelligence

- What questions are being asked recently at a company?
- What DSA questions are trending for a role and level?
- Which system design prompts are common for senior backend roles?
- Which questions map to exact platform problems?
- Which questions are custom company variants?
- Which follow-ups commonly appear with a question?
- Which question families are rising?

### 5.2 Topic / Skill Demand Intelligence

Gold should show what the market demands, not what the user knows.

Examples:

- Google Mid Backend has high graph/grid demand.
- Amazon SDE2 has high behavioral ownership demand.
- Senior backend roles have rising system design demand.
- Startup fullstack roles are showing more debugging and agentic coding tasks.

Important correction:

> Gold should reference the AI Tutor Skill Graph for canonical skill IDs when possible, instead of creating a separate competing topic taxonomy.

### 5.3 Interview Loop Intelligence

- How many rounds are common?
- Is there usually an online assessment?
- Does this role/level include system design?
- How common are behavioral rounds?
- What round order appears most often?
- Which item types appear in each round?

### 5.4 Failure and Success Intelligence

- Why do candidates fail this company/round?
- What candidate mistakes appear repeatedly?
- What prep advice appears repeatedly?
- What communication or system-design weaknesses are reported?
- Which failure reasons are rising recently?

### 5.5 Evidence and Trust

- Why is this question shown?
- How recent is the evidence?
- How many distinct interviews support it?
- Was exact scope used or fallback scope used?
- What sample prompts or evidence spans support this card?
- Is the canonicalization reviewed, automatic, or pending review?

### 5.6 Change Detection

- What newly appeared in the last 90 days?
- Which questions are rising?
- Which topics are declining?
- Which old patterns resurfaced?
- What changed since the previous refresh?

---

## 6. What Gold Should Not Answer Directly

Gold should not directly answer user-specific learning questions.

| User-facing question | Should Gold answer it directly? | Correct owner |
|---|---:|---|
| What should this user study next? | No | AI Tutor / recommendation service |
| Is this user weak in graphs? | No | Learner Skill Graph |
| Has this user solved Number of Islands independently? | No | user_learning_item_state / coding evidence |
| What is this user's readiness score? | No | AI Tutor / app readiness projection |
| What revision is due today? | No | Learner Skill Graph / prep-plan logic |
| What prep path should this user follow? | No | TutorOrchestrator / durable prep-plan service |
| Should this user deprioritize DP? | No | Tutor combines Gold demand with StudentSnapshot |
| Which company is this user most ready for? | No | App-level comparison using Gold + StudentSnapshot |
| Which notification should this user receive? | No | App notification service using Gold trend_changes |

Gold can provide inputs for these answers, but should not own the final user-specific decision.

---

## 7. Corrected Gold Capabilities

## 7.1 Evidence-Backed Question Cards

This remains a core Gold capability.

A question card should expose:

- Canonical title
- Item type
- Platform references
- Company/role/level scope
- Time window
- Recent support count
- Lifetime support count
- Last seen date
- Trend score
- Round types where it appears
- Common follow-ups
- Related question family
- Skill/topic mappings
- Confidence label
- Fallback scope used
- Sample anonymized prompts
- Evidence snippets when available

Gold owns these cards because they are market-intelligence artifacts, not user-specific artifacts.

## 7.2 Question Families and Variants

Gold should group related interview prompts into canonical families:

| Family | Example variants |
|---|---|
| Grid traversal / islands | Number of Islands, Max Area of Island, dynamic islands, shortest bridge |
| URL shortener | TinyURL, Bitly, analytics variant, abuse-detection variant |
| Parking lot LLD | Vehicle hierarchy, pricing strategy, gates, ticketing, payments |
| Rate limiter | Token bucket, sliding window, distributed API gateway limiter |
| Behavioral ownership | Conflict, ambiguity, missed deadline, project rescue |

This is not the same as the Tutor Skill Graph. Gold families describe market-observed question clusters. Tutor skills describe learning concepts and competencies. They should be linked, not merged.

## 7.3 Topic / Skill Demand Trends

Gold should compute topic/skill demand trends using canonical Skill Graph IDs where possible.

Correct design:

- `app.skills` remains the canonical learning taxonomy.
- Gold stores references to `skill_id` for trend aggregation.
- Gold may keep raw topic labels when mapping confidence is low.
- Gold should not create a separate `gold.skills` or `gold.topics` taxonomy that competes with the Tutor Skill Graph.

Example output:

| Scope | Demand insight |
|---|---|
| Google Mid Backend | Graph/grid traversal rising; DP moderate; hashmap stable |
| Amazon SDE2 | Behavioral ownership highly frequent; arrays/hashmap common |
| Frontend roles | JavaScript internals, async, component design, browser APIs rising |
| Data roles | SQL, statistics, ML theory, data pipelines frequent |

## 7.4 Interview Loop Profiles

Gold should aggregate round structure and item mix.

Example:

```text
Amazon Mid Backend commonly includes OA, DSA, LLD/system design, and behavioral rounds. Behavioral signals are stronger than many peer companies at the same level.
```

Gold owns this because it is extracted from interview experiences, not from user progress.

## 7.5 Failure Reason Intelligence

Gold should aggregate failure reasons, prep advice, and candidate mistakes from Silver signals.

Examples:

| Scope | Common failure reasons |
|---|---|
| Meta phone screen | Did not finish code, poor edge cases, weak time management |
| Amazon behavioral | Weak ownership stories, vague impact, poor conflict examples |
| System design rounds | Shallow tradeoffs, weak bottleneck analysis, ignored scaling limits |
| OA rounds | Hidden edge cases, TLE, time pressure |

Gold owns aggregate failure intelligence. The Tutor decides whether a specific user needs repair or practice for that failure mode.

## 7.6 Hybrid Interview Evidence Search

Gold should support corpus search over interview evidence.

Use cases:

- “Show recent Google system design interviews where caching was discussed.”
- “Find Meta interviews where candidates failed because of time management.”
- “Show prompts similar to this DSA question.”
- “What do candidates say about Amazon behavioral rounds?”

Correct retrieval rule:

```text
SQL filters first: company, role, level, item type, date window
Then lexical/vector search: wording, themes, failure narratives, similar prompts
Then synthesis: AI Tutor or research mode summarizes evidence
```

Gold owns the searchable market evidence. The AI Tutor owns the response strategy and final user-facing explanation.

## 7.7 Change Detection

Gold should compare current and previous windows.

It should detect:

- Newly emerging questions
- Rising topics/skills
- Declining questions
- Resurfacing old patterns
- New failure reasons
- Company process changes

Gold should publish trend-change signals. The app notification layer decides which users should be notified.

## 7.8 Company and Role Comparison

Gold can support company/role comparison, but this should be an aggregate market feature, not a user readiness feature.

Correct examples:

- Google Mid Backend vs Amazon Mid Backend market pattern
- Meta E4 vs Google L4 interview loop differences
- Backend vs fullstack topic mix
- Senior vs mid system design frequency

Incorrect Gold responsibility:

- “Which of these companies is Harsh personally most ready for?”

That requires user readiness and belongs to the AI Tutor/app layer.

---

## 8. Corrected Data Model

## 8.1 Required Gold Tables

These are the corrected Gold tables.

| Table | Keep? | Responsibility |
|---|---:|---|
| `gold.refresh_runs` | Yes | Tracks batch/incremental refreshes, inputs, status, row counts, errors. |
| `gold.role_levels` | Yes | Product-owned company role/level mapping for interview-market cohorts. |
| `gold.interview_roles` | Yes | Clean company/role/level/date/quality cohort row per Silver interview. |
| `gold.question_families` | Yes | Market-observed question/pattern families. |
| `gold.questions` | Yes | Canonical interview question/problem concept. |
| `gold.question_references` | Yes | External platform references and variant links. |
| `gold.question_skills` | Add | Bridge Gold questions/families to AI Tutor `app.skills`; not a separate skill taxonomy. |
| `gold.question_occurrences` | Yes | Auditable mapping from Silver assessment item occurrence to Gold question. |
| `gold.question_resolution_reviews` | Yes | Human/LLM review workflow for ambiguous canonicalization. |
| `gold.question_trends` | Yes | Question trend aggregate by scope/window. |
| `gold.topic_trends` | Yes, renamed conceptually to skill demand trends | Demand by `skill_id`/topic, scope, item type, and window. |
| `gold.interview_profiles` | Yes | Interview loop and round-mix aggregate. |
| `gold.signal_trends` | Yes | Failure reasons, prep advice, process events, red flags. |
| `gold.search_documents` | Yes, but scoped | Interview evidence retrieval docs; should not duplicate Tutor educational `knowledge_chunks`. |
| `gold.trend_changes` | Yes | Rising/new/declining/resurfacing trend signals. |
| `gold.company_role_comparisons` | Optional | Precomputed comparison snapshots; can start as query/materialized view. |

### Why add `gold.question_skills`?

The original report used topic fields inside Gold questions/trends. That works for a prototype, but it risks creating a second topic taxonomy.

The corrected design should link Gold market questions to the AI Tutor Skill Graph:

```text
gold.questions / gold.question_families
  -> gold.question_skills
  -> app.skills
```

This lets Gold say “Graph BFS/DFS is rising for Google Mid Backend” using the same skill ID that the Tutor uses for readiness, prerequisites, learning items, and practice selection.

## 8.2 Tables to Remove or Not Create

The following should not be introduced as new Gold-driven personalization tables.

| Previous proposal | Corrected decision | Why |
|---|---|---|
| `app.user_target_readiness_snapshots` | Do not create as a separate Gold-owned table. Extend/reuse existing readiness snapshots or Tutor readiness projections. | Readiness is already a learner-state/app concern. |
| `app.user_gold_recommendations` | Do not create as a Gold-owned concept. Use Tutor/app recommendation tables if needed. | Recommendations require StudentSnapshot, memory, help level, schedule, and prep-plan context. |
| `app.company_watchlists` | Keep only as app/product table if needed. | User subscription state is not Gold. |
| `app.watchlist_notifications` | Keep only as app/product notification table if needed. | Delivery state is not Gold. |
| Gold-owned prep path table | Do not build. | Durable prep plans already exist in app/tutor layer. |
| Gold-owned revision priority table | Do not build. | Revision due dates belong to learner/item state. |
| Gold-owned user weakness table | Do not build. | Weakness is derived from learning events, coding evidence, and learner projections. |

## 8.3 Existing App/Tutor Tables Gold Must Reuse

Gold should not duplicate these AI Tutor/app responsibilities.

| Existing / Tutor table | Gold relationship |
|---|---|
| `app.skills` | Gold references this for skill/topic demand mapping. |
| `app.skill_aliases` | Tutor uses this for user-language resolution; Gold may use it offline for mapping raw topics but should not own it. |
| `app.skill_relationships` | Tutor uses prerequisites/related concepts; Gold may expose demand by skill but does not own learning graph edges. |
| `app.learning_items` | Tutor practice/assessment catalog. Gold questions may be converted into learning items, but Gold does not own practice state. |
| `app.learning_item_skills` | Tutor item-to-skill bridge. Gold can use matching skill IDs. |
| `app.learning_events` | User learning evidence. Gold never writes user mastery events. |
| `app.learner_skill_state` | User skill readiness/coverage/confidence. Gold only supplies demand signals. |
| `app.user_learning_item_state` | User item state and independent/assisted solving. Gold does not own this. |
| `app.user_context_events` | User memory/context events. Gold does not own this. |
| `app.user_profile_summary` | Current user profile/target/preferences. Gold reads only through app services if needed. |
| `user_data.user_coding_problems` | User coding performance evidence. Gold should not write to this. |
| `user_data.user_coding_submissions` | Strong coding evidence. Tutor/app uses this for readiness. |
| `app.user_readiness_snapshots` | Existing readiness projection. Should consume Gold demand if extended. |
| `app.user_prep_plans` | Durable prep plans. Gold supplies market demand inputs only. |

---

## 9. Corrected Table Details

## 9.1 `gold.refresh_runs`

**Grain:** One Gold job run.  
**Purpose:** Observability, replay, debugging, and audit.

Stores:

- Job name
- Refresh type: full, incremental, overlay
- Source date range
- Source max updated timestamp
- Started/finished timestamps
- Status
- Rows inserted/updated/skipped
- Ambiguous rows deferred
- Error metadata
- Model/embed/reranker versions where relevant

## 9.2 `gold.role_levels`

**Grain:** One company + role family + native level mapping.

Purpose:

- Product-owned company role/level normalization
- Native level to canonical rank mapping
- IC/manager track normalization
- Reviewable confidence and evidence

Gold should own this because company-role-level market intelligence depends on consistent cohorting.

## 9.3 `gold.interview_roles`

**Grain:** One clean cohort assignment per Silver interview.

Purpose:

- Avoid repeated joins/resolution across Silver tables
- Store public eligibility and quality weight
- Store interview date and estimated-date flag
- Provide a reusable base for all trend/profile aggregates

## 9.4 `gold.question_families`

**Grain:** One market-observed question/pattern family.

Purpose:

- Group exact questions and variants
- Prevent fragmented trend rankings
- Support concept-level preparation intelligence
- Represent system design, LLD, behavioral, and custom question families

Important boundary:

`question_families` is not the same as `app.skills`. A family is a market cluster; a skill is a learning competency. They should be linked through `gold.question_skills`.

## 9.5 `gold.questions`

**Grain:** One canonical interview question/problem concept.

Purpose:

- Normalize raw prompts into product-facing question objects
- Support question pages and cards
- Preserve item type, title, summary, family, difficulty, custom/internal flag, and support count

Examples:

- `lc:200` — Number of Islands
- `sd:url-shortener` — Design URL Shortener
- `lld:parking-lot` — Parking Lot LLD
- `behavioral:ownership-conflict` — Ownership conflict story

## 9.6 `gold.question_references`

**Grain:** One external reference per Gold question.

Purpose:

- Map Gold questions to platform/source references
- Support practice links and source transparency
- Represent same-as, variant-of, mentioned, or named-problem relationships

## 9.7 `gold.question_skills`

**Grain:** One Gold question/family to Tutor skill mapping.

Purpose:

- Connect market demand to user readiness without duplicating the Skill Graph
- Support topic/skill trends
- Let Tutor compare Gold demand with `learner_skill_state`

Recommended semantics:

| Field | Meaning |
|---|---|
| `question_id` | Gold question. |
| `family_id` | Optional Gold family. |
| `skill_id` | Canonical `app.skills` ID. |
| `mapping_role` | primary, secondary, prerequisite, follow_up, misconception_target. |
| `relevance_weight` | Weight for demand/readiness comparison. |
| `mapping_confidence` | Confidence of skill mapping. |
| `mapping_method` | exact, platform_tags, resolver, embedding, LLM, review. |
| `review_status` | auto, review_pending, reviewed, rejected. |

## 9.8 `gold.question_occurrences`

**Grain:** One Silver assessment item occurrence mapped to one Gold question.

Purpose:

- Auditable source for frequency and trend computation
- Store mapping method and confidence
- Preserve raw prompt summary and round context
- Support evidence-backed question cards

Frequency must count distinct interviews, not repeated mentions inside one interview.

## 9.9 `gold.question_resolution_reviews`

**Grain:** One ambiguous or reviewable canonicalization decision.

Purpose:

- Prevent silent overmerging
- Support human-in-the-loop correction
- Track candidate question IDs, scores, suggested route, final decision, reviewer notes

## 9.10 `gold.question_trends`

**Grain:** One question trend per scope + item type + question + time window.

Purpose:

- Fast product query for “what is being asked recently?”
- Store support count, trend score, confidence, fallback scope, last seen, round mix, common follow-ups, sample occurrence IDs

Recommended windows:

| Window | Meaning |
|---:|---|
| 90 days | Hot / very recent |
| 180 days | Current season |
| 365 days | Stable current market |
| 540 days | Sparse fallback only |

## 9.11 `gold.topic_trends` / Skill Demand Trends

**Grain:** One skill/topic demand trend per scope + skill/topic + item type + time window.

Purpose:

- Show market demand at concept/skill level
- Power company guide topic sections
- Feed Tutor demand snapshots

Corrected fields should include:

- `skill_id` when mapped to `app.skills`
- `raw_topic_label` when not confidently mapped
- `scope_type`
- `company_id`
- `role_family`
- `canonical_rank`
- `item_type`
- `window_days`
- `distinct_interviews`
- `question_count`
- `weighted_frequency`
- `trend_score`
- `trend_direction`
- `confidence_label`
- `top_supporting_question_ids`
- `last_seen_date`

## 9.12 `gold.interview_profiles`

**Grain:** One interview profile per scope + time window.

Purpose:

- Interview loop prediction
- Round mix and ordered pattern aggregation
- Item type mix
- Top warnings
- Confidence and fallback metadata

Example output:

```text
Amazon Mid Backend: OA appears frequently; DSA and behavioral are common; LLD/system design appears more often at higher levels.
```

## 9.13 `gold.signal_trends`

**Grain:** One signal trend per scope + signal type/subtype + time window.

Purpose:

- Aggregate failure reasons, prep advice, candidate mistakes, process events, red flags, interviewer behavior

Gold owns aggregate signal frequency. Tutor owns whether the signal applies to a user.

## 9.14 `gold.search_documents`

**Grain:** One market-evidence search document.

Purpose:

- Support SQL + FTS + vector retrieval over interview evidence
- Serve AI chat/research mode with scoped evidence

Important boundary:

`gold.search_documents` should not duplicate Tutor `app.knowledge_chunks`.

| Table | Purpose |
|---|---|
| `app.knowledge_chunks` | Educational/tutorial content used for explanations and practice. |
| `gold.search_documents` | Interview-market evidence used for trend/evidence/search intelligence. |

## 9.15 `gold.trend_changes`

**Grain:** One detected change per scope + entity + window comparison.

Purpose:

- Identify new, rising, declining, stable, and resurfacing trends
- Feed app watchlists, notifications, marketing pages, and “what changed recently” pages

Gold should not decide which user receives which notification.

## 9.16 `gold.company_role_comparisons`

**Grain:** One comparison snapshot between two scopes.

Status:

- Optional table after core trends are stable
- Can start as a materialized view or computed API query

Purpose:

- Compare market demand between companies, roles, levels, and windows

---

## 10. Corrected Data Flow

## 10.1 Batch-First Flow

Recommended source-of-truth flow:

```text
1. Bronze discovers and tracks raw source items.
2. Extractor produces structured Silver data.
3. Silver stores interviews, rounds, assessment items, signals, chunks, and evidence spans.
4. Gold role resolver builds clean company-role-level cohorts.
5. Gold question resolver maps raw prompts to canonical questions and families.
6. Gold maps questions/families to `app.skills` through `gold.question_skills`.
7. Gold aggregation jobs compute question trends, skill demand trends, interview profiles, and signal trends.
8. Gold search jobs build market-evidence search documents and embeddings.
9. Gold change jobs compare current and previous windows.
10. Product APIs and AI Tutor tools read Gold demand signals.
11. AI Tutor combines Gold signals with StudentSnapshot/UserMemorySnapshot for user-specific guidance.
```

## 10.2 Immediate Overlay Option

If near-real-time freshness is needed, use deterministic immediate overlay only.

When a new Silver interview is committed:

- Add it to a pending Gold queue.
- Run exact deterministic mapping only:
  - prompt hash
  - platform URL/slug/ID
  - known external reference
  - already reviewed canonical match
- Write safe occurrences or pending overlay rows.
- Do not publish ambiguous semantic/LLM matches as final public trends.
- Defer embedding/rerank/LLM matching to batch.

Recommended surfaces:

| Surface | Freshness source |
|---|---|
| Public company pages | Latest completed reviewed batch |
| Internal/admin preview | Batch + pending overlay |
| Premium fresh feed | Batch + deterministic overlay with confidence labels |
| Final canonical trends | Batch-reviewed Gold |

---

## 11. Retrieval and Routing Boundary

## 11.1 Deterministic Planner First

Gold queries should be planned deterministically whenever structured filters are present:

- Company
- Role family
- Level
- Item type
- Round type
- Date window
- Skill/topic

Do not make LLM routing the default for Gold queries.

## 11.2 SQL-Only Gold Queries

Use SQL-only retrieval for:

- Trending questions
- Skill/topic demand trends
- Interview loop profiles
- Round mix
- Question frequency
- Confidence/fallback metadata
- Company/role comparisons
- Trend changes

## 11.3 Hybrid Gold Evidence Queries

Use SQL + FTS/vector for:

- Similar interview experiences
- Similar prompts
- Narrative failure reasons
- Candidate mistakes
- Prep advice examples
- Evidence search
- Conceptual queries without exact filters

Correct hybrid flow:

```text
1. SQL narrows eligible corpus.
2. Full-text/vector search runs within the filtered set.
3. Results are ranked by structured relevance + semantic similarity.
4. AI Tutor or research mode summarizes with citations/evidence.
```

## 11.4 Tutor Integration

AI Tutor should expose an `interview_intelligence` intent that can call Gold tools.

Gold tool outputs should look like compact snapshots, not raw table dumps:

```text
GoldDemandSnapshot
  target_scope
  demand_window
  top_questions
  top_skill_demands
  interview_profile
  failure_signals
  trend_changes
  confidence_summary
  fallback_scope_used
  evidence_refs
```

Then Tutor combines:

```text
GoldDemandSnapshot
+ StudentSnapshot
+ UserMemorySnapshot
+ PrepPlanContext
= TutorPlan / response / recommendation
```

---

## 12. Corrected Personalization Design

## 12.1 Gold Demand Signal

Gold represents demand.

Examples:

- Google Mid Backend has high graph/grid frequency.
- Amazon SDE2 has high behavioral ownership frequency.
- Senior backend has rising system design frequency.

## 12.2 Tutor User Readiness Signal

Tutor/app represents supply.

Examples:

- User has solved 20 array problems but only 2 graph problems.
- User has multiple failed graph submissions.
- User has no verified system design practice.
- User has strong SQL progress.
- User solved a problem with full-solution help, so it should not count as independent verification.

## 12.3 Gap Calculation

The gap is calculated outside Gold.

Correct formula boundary:

```text
Gap = Gold demand weight - Tutor readiness/coverage/confidence state
```

Gold supplies:

- Demand score
- Trend score
- Recency
- Confidence
- Scope/fallback
- Evidence

Tutor supplies:

- Readiness score
- Coverage score
- Confidence score
- Help-level-adjusted evidence
- Staleness/review state
- Recent performance
- Prep-plan constraints
- User target and timeline

## 12.4 Recommendation Generation

Recommendations should be generated by Tutor/app logic, not Gold.

Recommendation ranking can use:

- Gold target relevance
- Gold trend score
- Gold confidence
- User weakness
- User readiness gap
- User recency/staleness
- Difficulty fit
- Time until target interview
- Revision value
- Learning item availability
- User preferred language/style

Gold should not persist the final personalized recommendation unless the product intentionally uses a generic app recommendation table that is clearly outside `gold.*`.

## 12.5 Explainability

A personalized recommendation should explain both sides:

| Explanation component | Source |
|---|---|
| Why this matters for target | Gold question/skill trends |
| Why now | Gold recency + trend change + user timeline |
| Why for this user | Learner Skill Graph / coding evidence |
| What to do next | Tutor learning item/practice selection |
| Evidence | Gold occurrences/evidence snippets |
| Confidence | Gold signal confidence + Tutor learner-state confidence |

---

## 13. Product Surfaces Powered by Corrected Gold

## 13.1 Company Interview Guide Pages

Gold powers:

- Expected interview loop
- Most asked DSA questions
- Trending system design prompts
- Common topics/skills
- Role-level differences
- Recently emerging questions
- Common failure reasons
- Evidence-backed examples

Tutor/app adds:

- “Your readiness for this company”
- “Your next tasks”
- “Your weak areas for this target”

## 13.2 Role-Level Pages

Gold powers:

- Common DSA patterns by role/level
- System design frequency
- LLD/machine-coding frequency
- Common failure reasons
- Company comparison by role

Tutor/app adds:

- User-specific role readiness
- Target-fit ranking

## 13.3 Question Pages

Gold powers:

- Companies where the question appears
- Roles/levels where it appears
- Recent trend score
- Common follow-ups
- Related variants
- Evidence snippets
- Platform references

Tutor/app adds:

- User completion status
- User last attempt
- User readiness for linked skills
- Suggested practice/revision action

## 13.4 Personalized Dashboard

This surface is not owned by Gold.

Gold supplies widgets/inputs:

- Recently emerging target-company questions
- Target-company skill demand
- Interview loop profile
- Failure signals

Tutor/app owns dashboard state:

- Your target readiness
- Trending questions you have not solved
- Weak skills that matter for your target
- Upcoming revision tasks
- Mock/practice suggestions

## 13.5 AI Chat / Research Mode

Gold should improve answers for interview-intelligence queries:

- “What should I prepare for Google L4 backend?”
- “What changed recently for Amazon SDE2?”
- “Show recent system design prompts around caching.”
- “Why am I weak for Meta E4?”

Correct handling:

| Query | Gold role | Tutor role |
|---|---|---|
| What changed recently for Amazon SDE2? | Return trend changes and evidence. | Summarize clearly. |
| Show recent caching design prompts. | Return scoped evidence search. | Synthesize examples. |
| What should I prepare for Google L4? | Return market demand. | Compare with StudentSnapshot and recommend. |
| Why am I weak for Meta E4? | Return Meta E4 demand. | Compare with user readiness and explain weakness. |

---

## 14. Public Eligibility and Confidence

Gold should keep the original report's quality posture.

### 14.1 Public Eligibility Gates

An occurrence should contribute to public trends only if:

- Interview is not rejected
- Interview is not known duplicate
- Authenticity is not suspect
- Company/role/date signal is usable
- Canonicalization is high-confidence or reviewed
- Minimum support threshold is met for the displayed claim

### 14.2 Minimum Support Thresholds

Suggested starting thresholds:

| Scope | Minimum support for public trend |
|---|---:|
| Exact company + role + level | 3 distinct interviews |
| Company + role any level | 3 distinct interviews |
| Company broad | 5 distinct interviews |
| Global role/family | 5+ distinct interviews |
| Custom question promotion | 3 distinct interviews or strong repeated exact-prompt evidence |

### 14.3 Confidence Labels

| Label | Meaning |
|---|---|
| Strong signal | Enough exact-scope recent evidence. |
| Moderate signal | Useful evidence, smaller support or broader fallback. |
| Weak signal | Sparse support, old data, or fallback used. |
| Sparse data | Too little evidence for a strong conclusion. |
| Review pending | Canonicalization/scope mapping is not fully trusted. |

Confidence must be visible in the product.

---

## 15. Corrected Feature Prioritization

## Phase 1 — Gold Foundation

Build first:

- `gold.refresh_runs`
- `gold.role_levels`
- `gold.interview_roles`
- `gold.questions`
- `gold.question_families`
- `gold.question_references`
- `gold.question_skills`
- `gold.question_occurrences`
- `gold.question_resolution_reviews`

Goal:

- Resolve market facts into canonical, auditable entities.

## Phase 2 — Core Market Aggregates

Build next:

- `gold.question_trends`
- `gold.topic_trends` / skill demand trends
- `gold.interview_profiles`
- `gold.signal_trends`

Goal:

- Power company pages, role pages, question pages, and AI Tutor demand snapshots.

## Phase 3 — Evidence and Trust Layer

Build:

- Evidence-backed cards
- Sample prompt selection
- Evidence snippet linking
- Confidence/fallback display
- Round-specific rankings

Goal:

- Make Gold trustworthy, not just ranked.

## Phase 4 — Search and Research Mode

Build:

- `gold.search_documents`
- SQL-first + FTS/vector retrieval
- Scoped evidence search APIs

Goal:

- Support AI chat/research mode over the interview corpus.

## Phase 5 — Change Detection

Build:

- `gold.trend_changes`
- Rising/new/declining/resurfacing detection

Goal:

- Power freshness moat, watchlist inputs, and “what changed recently” pages.

## Phase 6 — Tutor/App Personalization Integration

Do not build personalization inside Gold.

Instead:

- Expose GoldDemandSnapshot APIs.
- Let Tutor consume Gold demand in `interview_intelligence`, `roadmap_or_prep_plan`, `resource_recommendation`, and `creator_request` intents.
- Extend existing app readiness/recommendation surfaces if needed.
- Persist user-specific results in app/tutor tables, not `gold.*`.

---

## 16. Risks and Mitigations

## Risk 1 — Gold duplicates Tutor personalization

**Problem:** Two systems compute readiness/recommendations differently.

**Mitigation:** Gold exports demand only. Tutor/app computes personalized action.

## Risk 2 — Skill taxonomy splits into Gold topics vs Tutor skills

**Problem:** Gold says “graph traversal,” Tutor says “BFS/DFS,” and mapping becomes inconsistent.

**Mitigation:** Use `gold.question_skills` to reference `app.skills`. Keep raw topic labels only as fallback/debug metadata.

## Risk 3 — Gold search duplicates Tutor knowledge retrieval

**Problem:** `gold.search_documents` and `app.knowledge_chunks` become overlapping RAG stores.

**Mitigation:** Strict boundary:

- Gold search = interview evidence and market intelligence.
- Tutor knowledge chunks = educational explanations and practice content.

## Risk 4 — Old questions dominate trends

**Problem:** Lifetime frequency hides current interview reality.

**Mitigation:** Keep hard time windows and recency decay. Separate lifetime support from recent trend score.

## Risk 5 — Sparse data creates overconfident product claims

**Problem:** Low-support claims look authoritative.

**Mitigation:** Expose confidence, fallback scope, support count, and sparse-data labels.

## Risk 6 — Canonicalization overmerges different questions

**Problem:** Trust collapses if variants are incorrectly merged.

**Mitigation:** Use resolver ladder: exact hash, external references, lexical match, embedding candidates, reranking, LLM scoring, human review, deferred clustering.

## Risk 7 — Personalized recommendations feel fake

**Problem:** User does not trust “you are weak in X.”

**Mitigation:** Tutor recommendation explanations must include both Gold evidence and user evidence.

---

## 17. Corrected “What Not To Build”

Do not build:

- Hardcoded company-specific tables like `google_questions` or `amazon_questions`
- Per-role hardcoded trend tables
- Gold-owned user readiness tables
- Gold-owned user recommendation tables
- Gold-owned memory/profile tables
- Gold-owned revision schedule tables
- A separate Gold skill taxonomy competing with `app.skills`
- Vector-only retrieval for product queries
- LLM-first routing for structured Gold queries
- Public “trending” labels for one-off sightings
- Mock interview persistence inside Gold

Use flexible scoped aggregate tables and app/tutor-owned personalization.

---

## 18. Final Corrected Architecture

```text
Bronze
  ingest.ingest_item
  ingest.source_cursor
  raw/extracted object storage

Silver
  silver.interview
  silver.round
  silver.assessment_item_occurrence
  silver.signal
  silver.chunk
  silver.evidence_span

Gold: Market Intelligence
  gold.refresh_runs
  gold.role_levels
  gold.interview_roles
  gold.question_families
  gold.questions
  gold.question_references
  gold.question_skills
  gold.question_occurrences
  gold.question_resolution_reviews
  gold.question_trends
  gold.topic_trends
  gold.interview_profiles
  gold.signal_trends
  gold.search_documents
  gold.trend_changes
  optional gold.company_role_comparisons

AI Tutor / App: User Intelligence
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
  app.user_context_events
  app.user_profile_summary
  app.user_readiness_snapshots
  app.user_prep_plans
  app.user_prep_task_progress
  user_data.user_coding_problems
  user_data.user_coding_submissions
```

---

## 19. Final Decision

The corrected Gold Layer should be implemented as a **market intelligence layer**, not a personalization engine.

Keep Gold responsible for:

- Canonical interview questions
- Question families and variants
- Platform references
- Role/level cohorting
- Question occurrence mapping
- Question trends
- Skill/topic demand trends
- Interview loop profiles
- Failure and signal trends
- Evidence-backed cards
- Hybrid market evidence search
- Change detection
- Confidence, fallback, and auditability

Move or keep outside Gold:

- User readiness
- Personalized gap analysis
- Personalized recommendations
- Prep path generation
- Revision priority
- User dashboard state
- Watchlist subscriptions
- Notification delivery
- Mock interview generation
- User memory and profile summaries

The correct final product boundary is:

> **Gold says what matters in the interview market. Tutor says what matters for this user right now.**

This preserves the strongest parts of the Gold design while preventing duplicated user-personalization logic, conflicting readiness scores, and unnecessary app tables.

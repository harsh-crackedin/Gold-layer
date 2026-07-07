# CrackedIn Updated Gold Layer Schema Report — Gold V1

**Document type:** Updated schema/table/view report  
**Project:** CrackedIn / InterviewPrep.ai  
**Runtime database:** PostgreSQL only  
**Design update:** 6 physical Gold tables + views/materialized views  
**Scope:** Non-personalized interview-market intelligence across all interview question types, not only DSA  
**Gold V1 boundary:** No user-specific readiness, personalized recommendations, revision state, prep-path state, watchlist state, notification state, learner memory, or user progress dependency

---

## 1. Executive Summary

The updated Gold Layer should be a small durable canonicalization layer plus product-facing read models.

The final rule is:

```text
Gold physical tables store durable decisions.
Gold views/materialized views expose derived intelligence.
```

This reduces the physical Gold schema from the earlier 15-table proposal to **6 physical tables**:

1. `gold.question_families`
2. `gold.questions`
3. `gold.question_references`
4. `gold.question_occurrences`
5. `gold.question_skills`
6. `gold.refresh_runs`

Derived intelligence should be exposed through **views or materialized views**:

1. `gold.v_interview_scope`
2. `gold.v_review_queue`
3. `gold.v_question_trends`
4. `gold.v_skill_trends`
5. `gold.v_signal_trends`
6. `gold.v_interview_profiles`
7. `gold.v_trend_changes`
8. `gold.v_company_role_comparisons`

This schema is designed for **all interview assessment item types**, including:

- DSA / coding
- system design / HLD
- low-level design / LLD
- machine coding
- SQL and database design
- networking
- operating systems
- distributed systems
- concurrency
- debugging
- code review
- API design
- domain-specific technical questions
- ML/data questions
- behavioral questions
- resume deep dives
- product sense / estimation / case-style prompts

Gold should not create a second DSA-only taxonomy. It should map questions and families to the shared Skill Graph through `gold.question_skills`.

Gold V1 is intentionally **non-personalized**. It does not read or write user progress, learner state, readiness scores, target plans, watchlists, notifications, or recommendation rows. Those should be deferred to a later Tutor/App version that consumes Gold market-demand views as input.

---

## 2. Source Table Usage Summary

The following existing tables are referenced by the new Gold V1 schema. They are not recreated in this report.

### 2.1 Existing source tables used by Gold V1

| Existing table | How Gold V1 uses it |
|---|---|
| `silver.interview` | Source of company, role family, level, dates, outcome, quality, public eligibility, and trend windows. |
| `silver.round` | Source of round type, round order, stage structure, round-level difficulty, and interview loop profile data. |
| `silver.assessment_item_occurrence` | Main source of reported questions/items. Gold maps each occurrence to a canonical question or family. |
| `silver.signal` | Source of failure reasons, prep advice, red flags, process signals, interviewer behavior, and candidate mistakes. |
| `silver.chunk` | Source for semantic/evidence retrieval. Gold V1 should reference it but not duplicate chunk text or embeddings. |
| `silver.evidence_span` | Source of verified snippets used to explain why a trend or question is trusted. |
| `silver.company` | Canonical company dimension used by Gold scope filters and company pages. |
| `silver.canonical_ladder` | Cross-company level/rank dimension used for role-level aggregation. |
| `silver.company_level` | Native company level mapping used when exact company-level filtering is needed. |
| `catalog.coding_problems` | Optional platform-backed coding catalog used by `gold.question_references` for LeetCode/Codeforces/etc. mappings. |
| `app.skills` | Shared Skill Graph node table. Gold maps questions/families to these skills through `gold.question_skills`. |
| `app.skill_areas` | Skill domain grouping used for display and filtering through `app.skills`. |
| `app.skill_aliases` | Optional enrichment input for resolving extracted tags/names to shared skills. |

### 2.2 Existing tables explicitly not used by Gold V1

Gold V1 should not depend on user-owned or learner-state tables. These may consume Gold later, but they are not part of the Gold V1 schema or refresh jobs.

| Existing/future table family | V1 decision |
|---|---|
| `user_data.user_coding_problems`, `user_data.user_coding_submissions`, `code_blobs.*` | Not used by Gold V1. Later Tutor/App personalization can compare user practice history against Gold market demand. |
| `app.learner_skill_state`, `app.user_learning_item_state`, `app.learning_events` | Not used by Gold V1. These belong to the Tutor Learner Skill Graph. |
| `app.user_profile_summary`, `app.user_context_events`, external memory systems | Not used by Gold V1. These belong to user memory/personalization. |
| `app.user_target_readiness_snapshots`, `app.user_gold_recommendations` | Not part of Gold V1. Defer to V2 Tutor/App product layer. |
| `app.company_watchlists`, `app.watchlist_notifications` | Not part of Gold V1. Defer to V2 retention/notification layer that consumes `gold.v_trend_changes`. |
| `app.learning_items` | Not required by Gold V1. Later Tutor/App can map Gold questions to practice/learning inventory. |

---

## 3. Object Inventory

### 3.1 Physical Gold tables

| Object | Type | Grain | Main purpose |
|---|---|---|---|
| `gold.question_families` | Table | One canonical family/pattern/theme | Group variants and related questions. |
| `gold.questions` | Table | One canonical interview question/concept | Stable product-facing question object. |
| `gold.question_references` | Table | One external reference per question | Map questions to platforms, URLs, named refs, or custom refs. |
| `gold.question_occurrences` | Table | One Silver item occurrence mapping | Auditable mapping from raw reported item to Gold question/family. |
| `gold.question_skills` | Table | One question/family-to-skill mapping | Bridge Gold market demand to shared Skill Graph. |
| `gold.refresh_runs` | Table | One Gold pipeline job run | Track refresh, resolver, aggregation, and materialized view jobs. |

### 3.2 Gold views / materialized views

| Object | Type | Grain | Main purpose |
|---|---|---|---|
| `gold.v_interview_scope` | View or materialized view | One eligible interview scope row | Standard company/role/level/date/quality surface. |
| `gold.v_review_queue` | View | One occurrence needing review | Review UI/read surface over `question_occurrences`. |
| `gold.v_question_trends` | View or materialized view | One question trend per scope/window | Trending questions across all item types. |
| `gold.v_skill_trends` | View or materialized view | One skill trend per scope/window | Topic/skill demand trends using shared Skill Graph. |
| `gold.v_signal_trends` | View or materialized view | One signal trend per scope/window | Failure/prep/process/red-flag intelligence. |
| `gold.v_interview_profiles` | View or materialized view | One loop profile per scope/window | Expected interview loop and round mix. |
| `gold.v_trend_changes` | View or materialized view | One detected change per entity/scope | Rising, declining, new, resurfacing market signals. |
| `gold.v_company_role_comparisons` | View or materialized view | One comparison between two scopes | Compare companies, roles, and levels. |

---

# Part A - Physical Gold Tables

---

## 4. `gold.question_families`

### Purpose

Stores canonical interview question families, patterns, archetypes, themes, or conceptual clusters.

A family is broader than one exact question. It can represent:

- a DSA pattern such as graph traversal / islands,
- a system design archetype such as URL shortener,
- an LLD archetype such as parking lot,
- a networking concept cluster such as TCP congestion control,
- an OS/concurrency cluster such as deadlock / locks,
- a behavioral theme such as ownership or conflict resolution.

### Grain

One row per canonical family.

### Source tables used

| Source | Usage |
|---|---|
| `silver.assessment_item_occurrence` | Recurring raw prompts are clustered into families. |
| `gold.questions` | Questions attach to one family. |
| `app.skills` | Optional canonical skill anchor through `canonical_skill_id` or `gold.question_skills`. |

### Logical schema

| Column | Suggested type | Required | Description |
|---|---:|---:|---|
| `id` | `bigserial` | Yes | Primary key. |
| `family_key` | `text` | Yes | Stable slug/key, e.g. `dsa:grid-islands`, `sd:url-shortener`. |
| `family_title` | `text` | Yes | Display title. |
| `family_type` | `text` | Yes | `dsa_pattern`, `system_design_archetype`, `lld_archetype`, `behavioral_theme`, `cs_concept`, `domain_theme`, etc. |
| `primary_item_type` | `text` | Yes | Dominant item type, e.g. `dsa_coding`, `system_design_hld`, `behavioral`. |
| `summary` | `text` | No | Human-readable description. |
| `parent_family_id` | `bigint` | No | Optional hierarchy for subfamilies. |
| `canonical_skill_id` | `bigint` | No | Optional direct link to `app.skills.id`; detailed mapping still lives in `question_skills`. |
| `topics_snapshot` | `jsonb` | No | Transitional topic strings before full skill mapping. |
| `patterns_snapshot` | `jsonb` | No | Transitional pattern labels. |
| `support_count_lifetime` | `integer` | Yes | Denormalized lifetime support count from distinct interviews. |
| `confidence_label` | `text` | Yes | `strong`, `moderate`, `weak`, `review_pending`. |
| `created_from_occurrence_id` | `uuid` | No | First Silver occurrence that caused this family to be created. |
| `metadata_json` | `jsonb` | Yes | Sparse metadata: aliases, resolver notes, archetype details. |
| `created_at` | `timestamptz` | Yes | Creation time. |
| `updated_at` | `timestamptz` | Yes | Update time. |

### Suggested constraints / indexes

| Constraint or index | Purpose |
|---|---|
| `unique(family_key)` | Prevent duplicate families. |
| index on `(primary_item_type, family_type)` | Fast filtering by question type. |
| index on `canonical_skill_id` | Skill-linked browsing. |
| GIN on `metadata_json` | Sparse metadata search. |

### Use case

Power concept-level preparation such as:

> "Google backend interviews are focusing heavily on graph traversal / islands, not just one exact LeetCode problem."

### Example row

| Field | Example |
|---|---|
| `family_key` | `sd:url-shortener` |
| `family_title` | `URL Shortener` |
| `family_type` | `system_design_archetype` |
| `primary_item_type` | `system_design_hld` |
| `summary` | `Design short-link services such as TinyURL/Bitly, often with analytics, abuse prevention, or global scale follow-ups.` |
| `support_count_lifetime` | `37` |
| `confidence_label` | `strong` |

---

## 5. `gold.questions`

### Purpose

Stores one canonical interview question/problem concept across all item types.

This is not DSA-only. Examples include:

- `lc:200:number-of-islands`
- `sd:url-shortener`
- `lld:parking-lot`
- `networking:tcp-three-way-handshake`
- `os:deadlock-prevention`
- `sql:window-functions-ranking`
- `behavioral:ownership-story`
- `ml:feature-store-design`
- `debugging:memory-leak-investigation`

### Grain

One row per canonical question/concept.

### Source tables used

| Source | Usage |
|---|---|
| `silver.assessment_item_occurrence` | Raw prompts become canonical questions. |
| `catalog.coding_problems` | Optional external catalog reference for coding problems. |
| `gold.question_families` | Each question can belong to a broader family. |
| `gold.question_references` | External platform links attach to this table. |

### Logical schema

| Column | Suggested type | Required | Description |
|---|---:|---:|---|
| `id` | `bigserial` | Yes | Primary key. |
| `question_key` | `text` | Yes | Stable canonical key, e.g. `lc:200`, `sd:url-shortener`. |
| `canonical_title` | `text` | Yes | Display title. |
| `canonical_slug` | `text` | Yes | URL/product slug. |
| `item_type` | `text` | Yes | Dominant interview item type. |
| `family_id` | `bigint` | No | FK to `gold.question_families.id`. |
| `summary` | `text` | No | Short product-facing summary. |
| `difficulty_label` | `text` | No | `easy`, `medium`, `hard`, `mixed`, `unknown`, or design-specific label. |
| `is_custom` | `boolean` | Yes | True for company-authored/custom question with no platform equivalent. |
| `is_platform_backed` | `boolean` | Yes | True when backed by known external platform/catalog. |
| `canonicalization_version` | `text` | Yes | Resolver/canonicalization version. |
| `support_count_lifetime` | `integer` | Yes | Distinct-interview lifetime support count. |
| `first_seen_date` | `date` | No | First supported interview date. |
| `last_seen_date` | `date` | No | Last supported interview date. |
| `created_from_occurrence_id` | `uuid` | No | Silver occurrence that created/promoted this canonical. |
| `metadata_json` | `jsonb` | Yes | Design requirements, aliases, variant metadata, rubric hints, etc. |
| `created_at` | `timestamptz` | Yes | Creation time. |
| `updated_at` | `timestamptz` | Yes | Update time. |

### Suggested constraints / indexes

| Constraint or index | Purpose |
|---|---|
| `unique(question_key)` | Stable identity. |
| `unique(canonical_slug)` | Product URL safety. |
| index on `(item_type, family_id)` | Type/family browsing. |
| index on `(last_seen_date desc)` | Recent question pages. |
| GIN on `metadata_json` | Sparse metadata search. |

### Use case

Answer:

> "What exact system design prompts are being asked recently for senior backend interviews?"

### Example row

| Field | Example |
|---|---|
| `question_key` | `networking:tcp-congestion-control` |
| `canonical_title` | `Explain TCP congestion control` |
| `item_type` | `cs_fundamentals` |
| `family_id` | `networking:tcp-internals` family |
| `difficulty_label` | `medium` |
| `is_custom` | `false` |
| `is_platform_backed` | `false` |
| `summary` | `Conceptual networking question about slow start, congestion avoidance, packet loss, and throughput behavior.` |

---

## 6. `gold.question_references`

### Purpose

Maps a canonical question to external platforms, catalogs, URLs, source mentions, or named references.

This table is optional for custom questions. A canonical question can exist without any external reference.

### Grain

One external reference per canonical question.

### Source tables used

| Source | Usage |
|---|---|
| `silver.assessment_item_occurrence.attributes.external_refs` | Extracted references such as URLs, LC IDs, or named refs. |
| `catalog.coding_problems` | Platform problem lookup for coding references. |
| future catalogs | Codeforces, HackerRank, GFG, InterviewBit, internal curated catalogs. |

### Logical schema

| Column | Suggested type | Required | Description |
|---|---:|---:|---|
| `id` | `bigserial` | Yes | Primary key. |
| `question_id` | `bigint` | Yes | FK to `gold.questions.id`. |
| `platform` | `text` | Yes | `leetcode`, `codeforces`, `hackerrank`, `gfg`, `interviewbit`, `system_design_catalog`, `raw_url`, `company_named`, `custom`. |
| `external_id` | `text` | No | External numeric/string ID, e.g. `200`. |
| `external_slug` | `text` | No | External problem slug. |
| `external_title` | `text` | No | External title snapshot. |
| `external_url` | `text` | No | Source/practice URL. |
| `reference_type` | `text` | Yes | `same_as`, `variant_of`, `mentioned`, `named_problem`, `source_url`, `practice_link`. |
| `confidence_score` | `numeric` | Yes | 0-1 confidence. |
| `source_occurrence_id` | `uuid` | No | Silver occurrence that supplied the reference. |
| `metadata_json` | `jsonb` | Yes | Source-specific details. |
| `created_at` | `timestamptz` | Yes | Creation time. |
| `updated_at` | `timestamptz` | Yes | Update time. |

### Suggested constraints / indexes

| Constraint or index | Purpose |
|---|---|
| `unique(question_id, platform, external_id, external_slug, reference_type)` | Prevent duplicate references. |
| index on `(platform, external_slug)` | Fast lookup from parsed refs. |
| index on `question_id` | Join from question cards. |

### Use case

Show practice links on a question card, while keeping the canonical question independent from any one platform.

### Example row

| Field | Example |
|---|---|
| `question_id` | `Number of Islands canonical ID` |
| `platform` | `leetcode` |
| `external_id` | `200` |
| `external_slug` | `number-of-islands` |
| `reference_type` | `same_as` |
| `confidence_score` | `0.99` |

---

## 7. `gold.question_occurrences`

### Purpose

Stores the durable mapping from each reported Silver assessment item occurrence to a canonical Gold question/family.

This is the most important Gold fact table. It is where canonicalization decisions are persisted.

### Grain

One row per `silver.assessment_item_occurrence.id`.

### Source tables used

| Source | Usage |
|---|---|
| `silver.assessment_item_occurrence` | One source item/question occurrence per Gold occurrence row. |
| `silver.interview` | Scope fields such as company, role, level, date, quality. |
| `silver.round` | Round type and round index context. |
| `gold.questions` | Canonical question mapping. |
| `gold.question_families` | Family mapping, especially for variants. |
| `gold.refresh_runs` | Resolver run provenance. |

### Logical schema

| Column | Suggested type | Required | Description |
|---|---:|---:|---|
| `occurrence_id` | `uuid` | Yes | PK and FK to `silver.assessment_item_occurrence.id`. |
| `interview_id` | `uuid` | Yes | FK to `silver.interview.id`. |
| `round_id` | `uuid` | No | FK to `silver.round.id`. |
| `question_id` | `bigint` | No | FK to `gold.questions.id`; nullable for deferred/unresolved. |
| `family_id` | `bigint` | No | FK to `gold.question_families.id`. |
| `item_type` | `text` | Yes | Snapshot from Silver item type. |
| `prompt_hash` | `text` | No | Snapshot from Silver prompt hash. |
| `mapping_method` | `text` | Yes | `exact_hash`, `external_ref`, `catalog_match`, `lexical_match`, `embedding_rerank`, `llm_match`, `manual`, `deferred_cluster`. |
| `mapping_score` | `numeric` | No | Resolver score. |
| `routing_status` | `text` | Yes | `auto`, `review`, `defer`. |
| `is_variant` | `boolean` | Yes | Whether this occurrence is a variant. |
| `variant_of_question_id` | `bigint` | No | Base question if occurrence is a variant. |
| `candidate_matches_json` | `jsonb` | Yes | Candidate canonical matches and scores for review/audit. |
| `review_status` | `text` | Yes | `not_needed`, `pending`, `approved`, `rejected`, `needs_second_pass`. |
| `reviewer_decision` | `jsonb` | No | Final review decision payload. |
| `reviewer_notes` | `text` | No | Human review notes. |
| `reviewed_by` | `text` | No | Reviewer identifier. |
| `reviewed_at` | `timestamptz` | No | Review timestamp. |
| `public_eligible` | `boolean` | Yes | Whether this occurrence can contribute to public/product aggregate views. |
| `quality_weight` | `numeric` | Yes | Weight from interview quality, mapping confidence, date precision, duplicate status. |
| `resolver_version` | `text` | Yes | Version of resolver logic. |
| `refresh_run_id` | `bigint` | No | FK to `gold.refresh_runs.id`. |
| `resolved_at` | `timestamptz` | Yes | First resolution timestamp. |
| `updated_at` | `timestamptz` | Yes | Last update timestamp. |

### Suggested constraints / indexes

| Constraint or index | Purpose |
|---|---|
| primary key on `occurrence_id` | One mapping row per Silver item occurrence. |
| index on `(question_id)` | Trend aggregation. |
| index on `(family_id)` | Family trend aggregation. |
| index on `(interview_id)` | Distinct-interview counting. |
| index on `(round_id)` | Round-specific trends. |
| index on `(item_type)` | Type-specific trend filtering. |
| index on `(routing_status, review_status)` | Review queue. |
| index on `(public_eligible)` | Aggregate filter. |
| GIN on `candidate_matches_json` | Resolver audit/debugging. |

### Use case

Answer:

> "Why is Parking Lot LLD shown as trending for Flipkart backend?"

The system can trace the trend to `question_occurrences`, then back to Silver item prompts, rounds, interviews, and evidence.

### Example row

| Field | Example |
|---|---|
| `occurrence_id` | Silver item ID for "Design parking lot with multiple vehicle types" |
| `question_id` | `lld:parking-lot` canonical question ID |
| `family_id` | `lld:parking-lot-family` |
| `item_type` | `low_level_design` |
| `mapping_method` | `embedding_rerank` |
| `routing_status` | `auto` |
| `is_variant` | `true` |
| `public_eligible` | `true` |
| `quality_weight` | `0.87` |

---

## 8. `gold.question_skills`

### Purpose

Maps Gold questions and families to the shared Skill Graph.

This table prevents Gold from creating a duplicate topic taxonomy. Product UI can still say "topics", but backend should use `app.skills` as the canonical shared skill/concept layer.

### Grain

One mapping between either:

- one Gold question and one shared skill, or
- one Gold family and one shared skill.

At least one of `question_id` or `family_id` should be present.

### Source tables used

| Source | Usage |
|---|---|
| `gold.questions` | Exact question-to-skill mapping. |
| `gold.question_families` | Family/pattern-to-skill mapping. |
| `app.skills` | Canonical shared Skill Graph node. |
| `app.skill_aliases` | Helps resolve extracted tags/names to skills. |
| `silver.assessment_item_occurrence.attributes.topics_reported` | Transitional signal for initial mapping. |
| `catalog.coding_problems.topic_tags` | Useful for coding problem skill mapping. |

### Logical schema

| Column | Suggested type | Required | Description |
|---|---:|---:|---|
| `id` | `bigserial` | Yes | Primary key. |
| `question_id` | `bigint` | No | FK to `gold.questions.id`. |
| `family_id` | `bigint` | No | FK to `gold.question_families.id`. |
| `skill_id` | `bigint` | No | FK to `app.skills.id`; nullable during transition. |
| `skill_slug` | `text` | No | Transitional stable slug. |
| `skill_name_snapshot` | `text` | No | Display name snapshot for transition/debugging. |
| `relation_type` | `text` | Yes | `primary`, `secondary`, `prerequisite`, `followup`, `evaluation_focus`, `behavioral_theme`, `variant_dimension`. |
| `relevance_weight` | `numeric` | Yes | How strongly this question/family maps to the skill. |
| `confidence_score` | `numeric` | Yes | Mapping confidence. |
| `mapping_source` | `text` | Yes | `attributes`, `catalog`, `resolver`, `manual`, `llm`, `family_inference`. |
| `review_status` | `text` | Yes | `not_needed`, `pending`, `approved`, `rejected`. |
| `metadata_json` | `jsonb` | Yes | Evidence, source tags, alias match, notes. |
| `created_at` | `timestamptz` | Yes | Creation time. |
| `updated_at` | `timestamptz` | Yes | Update time. |

### Suggested constraints / indexes

| Constraint or index | Purpose |
|---|---|
| check: one of `question_id`, `family_id` is not null | Valid mapping target. |
| index on `(skill_id)` | Skill trend aggregation. |
| index on `(question_id)` | Question page skill list. |
| index on `(family_id)` | Family page skill list. |
| index on `(skill_slug)` | Transitional skill lookup. |
| unique logical mapping on `(question_id, family_id, skill_id, skill_slug, relation_type)` | Prevent duplicate mappings. |

### Use case

Answer:

> "What is Google mainly focusing on right now?"

`v_skill_trends` aggregates interview demand through `question_skills`, so it can say Graph BFS, API rate limiting, database indexes, TCP internals, or ownership stories are rising.

### Example row

| Field | Example |
|---|---|
| `family_id` | `sd:rate-limiter` family ID |
| `skill_id` | shared skill ID for `distributed-systems.rate-limiting` |
| `skill_slug` | `distributed-systems/rate-limiting` |
| `relation_type` | `primary` |
| `relevance_weight` | `1.0` |
| `mapping_source` | `manual` |

---

## 9. `gold.refresh_runs`

### Purpose

Tracks Gold pipeline jobs, resolver runs, aggregation refreshes, materialized view refreshes, and failures.

Gold is derived from Silver, so every product-facing result should be traceable to a refresh run.

### Grain

One row per Gold job run.

### Logical schema

| Column | Suggested type | Required | Description |
|---|---:|---:|---|
| `id` | `bigserial` | Yes | Primary key. |
| `job_name` | `text` | Yes | e.g. `question_resolver`, `refresh_question_trends`. |
| `job_type` | `text` | Yes | `resolver`, `aggregate`, `trend_change`, `materialized_view_refresh`, `full_refresh`, `backfill`. |
| `status` | `text` | Yes | `running`, `success`, `failed`, `partial`, `cancelled`. |
| `source_min_interview_date` | `date` | No | Lower bound of processed source interviews. |
| `source_max_interview_date` | `date` | No | Upper bound of processed source interviews. |
| `source_max_updated_at` | `timestamptz` | No | Latest source update included. |
| `resolver_version` | `text` | No | Resolver version. |
| `embedding_model_version` | `text` | No | Embedding model version if used. |
| `reranker_version` | `text` | No | Reranker/cross-encoder version if used. |
| `llm_model_version` | `text` | No | LLM version if used in ambiguous matching. |
| `rows_inserted` | `integer` | Yes | Insert count. |
| `rows_updated` | `integer` | Yes | Update count. |
| `rows_skipped` | `integer` | Yes | Skipped count. |
| `rows_deferred` | `integer` | Yes | Deferred/unresolved count. |
| `views_refreshed_json` | `jsonb` | Yes | List/status of refreshed views/materialized views. |
| `error_summary` | `text` | No | Failure summary. |
| `metadata_json` | `jsonb` | Yes | Arbitrary job metadata. |
| `started_at` | `timestamptz` | Yes | Start timestamp. |
| `finished_at` | `timestamptz` | No | Finish timestamp. |

### Suggested constraints / indexes

| Constraint or index | Purpose |
|---|---|
| index on `(job_name, started_at desc)` | Job history. |
| index on `(status)` | Monitor failures/running jobs. |
| index on `(job_type, started_at desc)` | Pipeline observability. |

### Use case

Debug why a trend page changed:

> "The last successful `question_resolver` run used resolver version `v3.1`, processed interviews through 2026-07-06, inserted 72 mappings, and deferred 18 ambiguous prompts."

### Example row

| Field | Example |
|---|---|
| `job_name` | `gold_question_resolver_daily` |
| `job_type` | `resolver` |
| `status` | `success` |
| `rows_inserted` | `214` |
| `rows_updated` | `89` |
| `rows_deferred` | `27` |
| `views_refreshed_json` | `["gold.v_question_trends", "gold.v_skill_trends"]` |

---

# Part B - Views and Materialized Views

---

## 10. `gold.v_interview_scope`

### Type

Start as a normal view. Convert to materialized view if aggregation queries become slow.

### Purpose

Creates one standardized interview scope row with company, role, level, date, and quality eligibility.

All trend/profile views should use this instead of repeating quality filters.

### Grain

One row per `silver.interview.id`.

### Source tables used

| Source | Usage |
|---|---|
| `silver.interview` | Main source of interview scope and quality fields. |
| `silver.company` | Company names/aliases if needed for product display. |
| `silver.canonical_ladder` | Level label and rank display. |
| `silver.company_level` | Native level display when exact company level is needed. |

### Logical schema

| Column | Description |
|---|---|
| `interview_id` | Silver interview ID. |
| `company_id` | Canonical company ID. |
| `company_name` | Optional joined display name. |
| `role_family` | Normalized role family from Silver. |
| `role_specialties` | Role specialties from Silver. |
| `canonical_level_id` | Cross-company rank from Silver. |
| `company_level_id` | Exact native company level mapping if available. |
| `country` | Role/candidate country context. |
| `employment_type` | Fulltime/internship/new-grad/etc. |
| `interview_date` | Date used for trend windows. |
| `posted_at` | Source post date. |
| `is_date_estimated` | Whether interview date is estimated. |
| `llm_rejected` | Relevance gate from Silver. |
| `duplicate_state` | Derived duplicate status. |
| `authenticity_state` | Derived authenticity/trust status. |
| `extract_confidence` | Silver extraction confidence. |
| `richness_score` | Silver richness score. |
| `is_public_eligible` | Derived Boolean for public/product aggregation. |
| `quality_weight` | Derived numeric weight for trend scoring. |

### Use case

Every trend view can filter:

```text
where is_public_eligible = true
```

instead of reimplementing rejection, duplicate, confidence, and date-quality logic.

### Example row

| Field | Example |
|---|---|
| `company_name` | `Amazon` |
| `role_family` | `software_general` |
| `canonical_level_id` | `4` |
| `interview_date` | `2026-06-14` |
| `is_public_eligible` | `true` |
| `quality_weight` | `0.91` |

---

## 11. `gold.v_review_queue`

### Type

Normal view.

### Purpose

Provides a review workflow surface without creating a separate physical review table.

### Grain

One row per `gold.question_occurrences` row that needs review.

### Source tables used

| Source | Usage |
|---|---|
| `gold.question_occurrences` | Review/routing fields. |
| `silver.assessment_item_occurrence` | Raw prompt and attributes for reviewer context. |
| `gold.questions` | Candidate/final question display. |
| `gold.question_families` | Candidate/final family display. |

### Logical schema

| Column | Description |
|---|---|
| `occurrence_id` | Silver occurrence ID. |
| `interview_id` | Silver interview ID. |
| `round_id` | Silver round ID. |
| `item_type` | Assessment item type. |
| `prompt_summary` | From Silver item occurrence. |
| `prompt_raw` | Optional reviewer-only field from Silver. |
| `current_question_id` | Current mapped question, if any. |
| `current_family_id` | Current mapped family, if any. |
| `candidate_matches_json` | Candidate matches and scores. |
| `routing_status` | `review`, `defer`, etc. |
| `review_status` | Pending/approved/rejected. |
| `mapping_score` | Resolver score. |
| `resolver_version` | Resolver version. |
| `updated_at` | Last mapping update. |

### Use case

Human reviewer opens ambiguous mappings such as:

> "Is this weighted Word Ladder variant the same exact problem, a variant, or only the same shortest-path family?"

### Example row

| Field | Example |
|---|---|
| `item_type` | `dsa_coding` |
| `prompt_summary` | `Word Ladder with weighted transformations` |
| `routing_status` | `review` |
| `candidate_matches_json` | `Word Ladder: 0.72, Shortest Path family: 0.66` |

---

## 12. `gold.v_question_trends`

### Type

Normal view in development. Materialized view for launch/company pages.

### Purpose

Answers question-level market-demand queries for all item types.

Examples:

- "What DSA questions are trending at Google?"
- "What system design prompts are common for senior backend?"
- "What behavioral questions are asked at Amazon?"
- "What networking questions are showing up in infra interviews?"

### Grain

One row per scope + item type + question/family + time window.

### Source tables used

| Source | Usage |
|---|---|
| `gold.question_occurrences` | Canonical occurrence support. |
| `gold.questions` | Question metadata and title. |
| `gold.question_families` | Family grouping. |
| `gold.v_interview_scope` | Scope and eligibility. |
| `silver.round` | Round type distribution. |
| `silver.evidence_span` | Sample evidence IDs, if available. |

### Logical schema

| Column | Description |
|---|---|
| `scope_type` | `company_role_level`, `company_role`, `company_any_role`, `global_role_level`, `global_role`, `global_all`. |
| `company_id` | Nullable for global scopes. |
| `role_family` | Nullable for company/global broad scopes. |
| `canonical_level_id` | Nullable for mixed-level scopes. |
| `company_level_id` | Optional native level scope. |
| `item_type` | `dsa_coding`, `system_design_hld`, `behavioral`, etc. |
| `question_id` | Canonical question ID. |
| `family_id` | Family ID. |
| `window_days` | 90, 180, 365, 540. |
| `distinct_interview_count` | Count distinct interviews, not raw mentions. |
| `raw_occurrence_count` | Raw occurrence count. |
| `weighted_frequency` | Quality-weighted support. |
| `recency_score` | Recentness component. |
| `trend_score` | Final score. |
| `confidence_label` | Product-visible confidence. |
| `fallback_scope_used` | Whether exact scope was unavailable and fallback was used. |
| `first_seen_date` | First seen in window/scope. |
| `last_seen_date` | Last seen in window/scope. |
| `round_type_counts_json` | Distribution by round type. |
| `common_followups_json` | Frequent follow-ups from occurrence attributes. |
| `observed_difficulty_json` | Reported difficulty distribution. |
| `sample_occurrence_ids_json` | IDs for explanation/evidence. |
| `sample_interview_ids_json` | Interview samples. |
| `sample_evidence_span_ids_json` | Evidence snippets when available. |
| `refreshed_at` | Required if materialized. |

### Use case

Power company pages:

> "For Google mid-level backend interviews in the last 180 days, graph/grid traversal and system design caching questions are strong signals."

### Example row

| Field | Example |
|---|---|
| `scope_type` | `company_role_level` |
| `company_id` | `Google ID` |
| `role_family` | `software_general` |
| `canonical_level_id` | `4` |
| `item_type` | `system_design_hld` |
| `question_id` | `sd:rate-limiter` |
| `window_days` | `180` |
| `distinct_interview_count` | `7` |
| `trend_score` | `8.7` |
| `confidence_label` | `strong_signal` |

---

## 13. `gold.v_skill_trends`

### Type

Normal view in development. Materialized view for product pages.

### Purpose

Answers topic/skill-level market-demand queries using the shared Skill Graph.

Examples:

- "What is this company mainly focusing on?"
- "Are networking fundamentals rising for infra roles?"
- "Is system design more important than DSA for senior roles?"
- "Which behavioral themes are common at Amazon?"

### Grain

One row per scope + skill + optional item type + time window.

### Source tables used

| Source | Usage |
|---|---|
| `gold.question_skills` | Maps questions/families to shared skills. |
| `gold.question_occurrences` | Interview support. |
| `gold.questions` | Question metadata. |
| `gold.question_families` | Family metadata. |
| `gold.v_interview_scope` | Scope and eligibility. |
| `app.skills` | Canonical skill display and hierarchy. |

### Logical schema

| Column | Description |
|---|---|
| `scope_type` | Same scope model as question trends. |
| `company_id` | Nullable for global scopes. |
| `role_family` | Role family filter. |
| `canonical_level_id` | Level/rank filter. |
| `company_level_id` | Native company level filter. |
| `item_type` | Nullable; allows cross-item skill demand. |
| `skill_id` | FK to `app.skills.id`, nullable during transition. |
| `skill_slug` | Skill slug snapshot. |
| `skill_name_snapshot` | Skill name snapshot. |
| `window_days` | 90, 180, 365, 540. |
| `distinct_interview_count` | Distinct supporting interviews. |
| `distinct_question_count` | Distinct questions/families supporting skill. |
| `weighted_frequency` | Quality-weighted support. |
| `trend_score` | Final trend score. |
| `trend_direction` | `new`, `rising`, `stable`, `declining`, `resurfacing`. |
| `confidence_label` | Product-visible confidence. |
| `top_question_ids_json` | Top supporting questions. |
| `top_family_ids_json` | Top supporting families. |
| `sample_occurrence_ids_json` | Evidence samples. |
| `last_seen_date` | Last observed date. |
| `refreshed_at` | Required if materialized. |

### Use case

Answer:

> "Amazon SDE2 is mainly focusing on behavioral ownership, arrays/hashmaps, graph traversal, and LLD/machine-coding style design."

### Example row

| Field | Example |
|---|---|
| `company_id` | `Amazon ID` |
| `role_family` | `software_general` |
| `canonical_level_id` | `4` |
| `skill_slug` | `behavioral/ownership` |
| `item_type` | `behavioral` |
| `window_days` | `180` |
| `distinct_interview_count` | `19` |
| `trend_direction` | `stable` |
| `confidence_label` | `strong_signal` |

---

## 14. `gold.v_signal_trends`

### Type

Normal view first. Materialized view when failure/process intelligence becomes product-facing.

### Purpose

Aggregates Silver signals into product intelligence.

Examples:

- "Why do candidates fail Meta phone screens?"
- "What prep advice appears repeatedly for Google system design?"
- "What red flags are common in startup hiring loops?"
- "Which behavioral mistakes are common at Amazon?"

### Grain

One row per scope + signal type/subtype + optional round/item type + time window.

### Source tables used

| Source | Usage |
|---|---|
| `silver.signal` | Atomic signal facts. |
| `gold.v_interview_scope` | Scope and eligibility. |
| `silver.round` | Round type context. |
| `silver.assessment_item_occurrence` | Optional item type context. |
| `silver.evidence_span` | Supporting evidence snippets when available. |

### Logical schema

| Column | Description |
|---|---|
| `scope_type` | Scope bucket. |
| `company_id` | Nullable for global scopes. |
| `role_family` | Role family. |
| `canonical_level_id` | Cross-company level. |
| `company_level_id` | Native company level. |
| `round_type` | Nullable round filter. |
| `item_type` | Nullable item type filter. |
| `signal_type` | `failure_reason`, `prep_advice`, `red_flag`, `process_event`, `candidate_mistake`, etc. |
| `signal_subtype` | More specific signal category. |
| `window_days` | 90, 180, 365, 540. |
| `distinct_interview_count` | Distinct supporting interviews. |
| `raw_signal_count` | Raw signal count. |
| `weighted_frequency` | Quality-weighted support. |
| `trend_score` | Final score. |
| `confidence_label` | Product-visible confidence. |
| `sample_signal_ids_json` | Sample signal IDs. |
| `sample_interview_ids_json` | Sample interview IDs. |
| `sample_evidence_span_ids_json` | Sample evidence spans. |
| `last_seen_date` | Latest supporting interview date. |
| `refreshed_at` | Required if materialized. |

### Use case

Answer:

> "Candidates often fail Meta phone screens due to not finishing code, weak edge-case handling, and poor time management."

### Example row

| Field | Example |
|---|---|
| `company_id` | `Meta ID` |
| `round_type` | `problem_solving_dsa` |
| `signal_type` | `failure_reason` |
| `signal_subtype` | `time_management` |
| `window_days` | `180` |
| `distinct_interview_count` | `8` |
| `confidence_label` | `moderate_signal` |

---

## 15. `gold.v_interview_profiles`

### Type

Materialized view for company/role guide pages.

### Purpose

Describes expected interview loop structure and round mix.

Examples:

- "Does Google L4 ask system design?"
- "How many rounds does Amazon SDE2 usually have?"
- "Is machine coding common for this role?"
- "Are networking questions asked in infra interviews?"

### Grain

One row per scope + time window.

### Source tables used

| Source | Usage |
|---|---|
| `gold.v_interview_scope` | Eligible interviews. |
| `silver.round` | Round count, round order, round mix. |
| `silver.assessment_item_occurrence` | Item type mix. |
| `silver.signal` | Process/failure/prep signals. |
| `gold.question_occurrences` | Canonical question support. |
| `gold.question_skills` | Skill mix. |

### Logical schema

| Column | Description |
|---|---|
| `scope_type` | Scope bucket. |
| `company_id` | Nullable for global scopes. |
| `role_family` | Role family. |
| `canonical_level_id` | Cross-company level. |
| `company_level_id` | Native company level. |
| `window_days` | 90, 180, 365, 540. |
| `total_interviews` | Supporting interview count. |
| `total_rounds` | Supporting round count. |
| `average_round_count` | Avg rounds/interview. |
| `median_round_count` | Median rounds/interview. |
| `median_process_duration_days` | Median process duration. |
| `round_type_mix_json` | Distribution of OA, DSA, HLD, LLD, behavioral, etc. |
| `item_type_mix_json` | Distribution of assessment item types. |
| `ordered_round_patterns_json` | Common loop sequences. |
| `top_skill_ids_json` | Most common skills. |
| `top_question_ids_json` | Most common questions. |
| `top_signal_keys_json` | Most common signals. |
| `confidence_label` | Product-visible confidence. |
| `first_seen_date` | First supporting interview date. |
| `last_seen_date` | Latest supporting interview date. |
| `sample_interview_ids_json` | Sample interviews. |
| `refreshed_at` | Required if materialized. |

### Use case

Answer:

> "Amazon mid-level software roles commonly include OA, DSA/problem-solving, behavioral/LP-heavy evaluation, and sometimes LLD or system design depending on team/level."

### Example row

| Field | Example |
|---|---|
| `company_id` | `Amazon ID` |
| `role_family` | `software_general` |
| `canonical_level_id` | `4` |
| `average_round_count` | `4.2` |
| `round_type_mix_json` | `{online_assessment: 0.72, problem_solving_dsa: 0.81, behavioral: 0.68, system_design_hld: 0.31}` |
| `confidence_label` | `moderate_signal` |

---

## 16. `gold.v_trend_changes`

### Type

Materialized view after trend windows are stable.

### Purpose

Detects what changed between current and previous windows.

Examples:

- new questions,
- rising skills,
- declining questions,
- resurfacing system design prompts,
- newly common behavioral/failure signals,
- round mix changes.

### Grain

One row per changed entity + scope + comparison window.

### Source views used

| Source | Usage |
|---|---|
| `gold.v_question_trends` | Question/family change detection. |
| `gold.v_skill_trends` | Skill/topic change detection. |
| `gold.v_signal_trends` | Failure/process signal change detection. |
| `gold.v_interview_profiles` | Round/profile change detection. |

### Logical schema

| Column | Description |
|---|---|
| `entity_type` | `question`, `family`, `skill`, `signal`, `round_profile`, `item_type_mix`. |
| `entity_id` | Nullable for profile-level changes. |
| `entity_key` | Stable display/debug key. |
| `scope_type` | Scope bucket. |
| `company_id` | Nullable for global scopes. |
| `role_family` | Role family. |
| `canonical_level_id` | Level. |
| `item_type` | Nullable item type. |
| `current_window_days` | Current window, e.g. 90. |
| `previous_window_days` | Previous comparison window. |
| `current_score` | Current trend score. |
| `previous_score` | Previous trend score. |
| `delta_score` | Difference. |
| `change_type` | `new`, `rising`, `stable`, `declining`, `resurfacing`, `disappeared`. |
| `confidence_label` | Product-visible confidence. |
| `support_count_current` | Current support count. |
| `support_count_previous` | Previous support count. |
| `sample_ids_json` | Supporting question/skill/signal/interview IDs. |
| `detected_at` | Detection timestamp. |

### Use case

Answer:

> "In the last 90 days, rate-limiter design moved from weak to moderate signal for senior backend roles, while classic URL shortener remained stable."

### Example row

| Field | Example |
|---|---|
| `entity_type` | `skill` |
| `entity_key` | `distributed-systems/rate-limiting` |
| `role_family` | `software_general` |
| `canonical_level_id` | `5` |
| `change_type` | `rising` |
| `delta_score` | `+3.1` |
| `confidence_label` | `moderate_signal` |

---

## 17. `gold.v_company_role_comparisons`

### Type

Normal view first. Materialized view only for high-traffic comparison pages.

### Purpose

Compares two company/role/level scopes.

Examples:

- "Google SDE2 vs Amazon SDE2"
- "Backend vs frontend interviews"
- "Entry vs senior system design expectations"
- "Data engineering vs backend SQL focus"

### Grain

One comparison row per left scope + right scope + window.

### Source views used

| Source | Usage |
|---|---|
| `gold.v_question_trends` | Compare question mix. |
| `gold.v_skill_trends` | Compare skill/topic mix. |
| `gold.v_signal_trends` | Compare failure/prep/process signals. |
| `gold.v_interview_profiles` | Compare round/interview loop patterns. |

### Logical schema

| Column | Description |
|---|---|
| `left_scope_json` | Company/role/level/item filters for left side. |
| `right_scope_json` | Company/role/level/item filters for right side. |
| `window_days` | Time window. |
| `left_support_count` | Supporting interview count. |
| `right_support_count` | Supporting interview count. |
| `skill_delta_json` | Differences in skill mix. |
| `question_delta_json` | Differences in top questions/families. |
| `round_mix_delta_json` | Differences in loop/round mix. |
| `signal_delta_json` | Differences in failure/process/prep signals. |
| `summary_text` | Optional generated/cached summary if materialized. |
| `confidence_label` | Product-visible confidence. |
| `refreshed_at` | Required if materialized. |

### Use case

Answer:

> "Google backend mid-level is more graph/design-depth heavy, while Amazon mid-level has stronger behavioral ownership signal and more OA-heavy reports."

### Example row

| Field | Example |
|---|---|
| `left_scope_json` | `{company: Google, role_family: software_general, level: 4}` |
| `right_scope_json` | `{company: Amazon, role_family: software_general, level: 4}` |
| `skill_delta_json` | `{google_higher: [graph, system-design], amazon_higher: [ownership, arrays]}` |
| `confidence_label` | `moderate_signal` |

---

# Part C - Item Type Coverage

## 18. Recommended `item_type` Coverage

Gold should preserve Silver's item type but should not assume every question is coding.

Recommended item types:

| Item type | Examples |
|---|---|
| `dsa_coding` | Number of Islands, Word Ladder, sliding window, DP. |
| `system_design_hld` | URL shortener, rate limiter, news feed, distributed cache. |
| `low_level_design` | Parking lot, elevator, splitwise, chess, logger. |
| `machine_coding` | Build an in-memory cache, file parser, UI widget, REST mini-app. |
| `sql` | Window functions, joins, query optimization, analytics queries. |
| `database_design` | Schema design, indexing, transactions, sharding. |
| `networking` | TCP handshake, DNS, HTTP, TLS, load balancing. |
| `operating_systems` | Processes, threads, deadlocks, memory management. |
| `distributed_systems` | Consensus, replication, queues, rate limiting. |
| `concurrency` | Locks, semaphores, race conditions, async execution. |
| `cs_fundamentals` | OOP, complexity, memory, compiler/runtime concepts. |
| `debugging` | Find bug, production incident, memory leak, latency issue. |
| `code_review` | Review PR/code snippet for correctness/design. |
| `api_design` | REST/GraphQL endpoint design, idempotency, pagination. |
| `behavioral` | Ownership, conflict, leadership, ambiguity, failure. |
| `resume_deepdive` | Project architecture, tradeoffs, impact, decisions. |
| `ml_theory` | Bias/variance, model metrics, training concepts. |
| `ml_system_design` | Feature store, model serving, recommender system. |
| `data_engineering` | ETL, pipelines, Kafka, Spark, warehouse modeling. |
| `product_sense` | Metrics, experimentation, product tradeoffs. |
| `estimation` | Capacity planning, Fermi estimates, scale calculations. |
| `unknown` | Kept for low-confidence extraction; should not dominate trends. |

---

# Part D - Query and Product Examples

## 19. Query: "What are trending questions?"

Use:

- `gold.v_question_trends`
- join `gold.questions`
- join `gold.question_references` when practice links are needed
- sample IDs from `gold.question_occurrences`
- sample snippets from `silver.evidence_span` / `silver.chunk`

Example answer:

> "For Google mid-level backend in the last 180 days, graph traversal, rate limiter design, and behavioral ambiguity questions are strong or moderate signals."

---

## 20. Query: "What is this company focusing on right now?"

Use:

- `gold.v_skill_trends`
- `gold.v_question_trends`
- `gold.v_signal_trends`

Example answer:

> "Amazon SDE2 is currently heavy on ownership/behavioral, arrays/hashmaps, graph traversal, and some LLD/machine-coding variants."

---

## 21. Query: "What was asked in behavioral rounds at this company?"

Use:

- `gold.v_question_trends` filtered by `item_type = behavioral`
- `gold.v_signal_trends` filtered by behavioral round type/signals
- `silver.round` for `round_type = behavioral`
- `silver.chunk` for semantic examples

Example answer:

> "Recent Amazon behavioral rounds repeatedly mention ownership, conflict resolution, handling ambiguity, and failure reflection. The strongest recurring signal is ownership-style storytelling."

---

## 22. Query: "What networking questions are being asked?"

Use:

- `gold.v_question_trends` filtered by `item_type in ('networking', 'cs_fundamentals')`
- `gold.v_skill_trends` where skill area is networking
- Silver chunks for semantic examples

Example answer:

> "Infra/backend interviews show recurring networking questions around TCP handshake, DNS resolution, HTTP vs HTTPS, load balancing, and latency debugging."

---

## 23. Query: "What changed recently?"

Use:

- `gold.v_trend_changes`
- back references to `v_question_trends`, `v_skill_trends`, and `v_signal_trends`

Example answer:

> "In the last 90-day window, rate-limiter design and debugging-style system questions are rising for senior backend roles, while classic URL-shortener remains stable."

---

# Part E - Build Order

## 24. Phase 1 - Physical tables

Create:

1. `gold.question_families`
2. `gold.questions`
3. `gold.question_references`
4. `gold.question_occurrences`
5. `gold.question_skills`
6. `gold.refresh_runs`

## 25. Phase 2 - Core views

Create:

1. `gold.v_interview_scope`
2. `gold.v_review_queue`
3. `gold.v_question_trends`
4. `gold.v_skill_trends`

This is enough for:

- trending questions,
- company focus,
- skill/topic trends,
- review queue,
- non-personalized market-demand read models.

## 26. Phase 3 - Product intelligence views

Create:

1. `gold.v_signal_trends`
2. `gold.v_interview_profiles`
3. `gold.v_trend_changes`
4. `gold.v_company_role_comparisons`

This unlocks:

- failure reason intelligence,
- interview loop prediction,
- process/round insights,
- change detection,
- comparison pages.

## 27. Phase 4 - Materialization and performance

Materialize high-traffic or expensive views:

- `gold.v_question_trends`
- `gold.v_skill_trends`
- `gold.v_signal_trends`
- `gold.v_interview_profiles`
- `gold.v_trend_changes`

Keep the same names in application/service code. Internally decide whether each object is a view or materialized view.

---

# Part F - Gold V1 Non-Personalization Boundary

Gold V1 should remain a pure market-intelligence layer.

## 28. What Gold V1 must not store

Do not add Gold V1 tables or columns for:

- user readiness scores,
- user-specific recommendations,
- user weak/strong skill projections,
- user target plans,
- user revision priority,
- user watchlists,
- notification delivery state,
- learner memory,
- personalized dashboard state.

## 29. What can consume Gold later

A later Tutor/App V2 can consume Gold views such as:

- `gold.v_question_trends`,
- `gold.v_skill_trends`,
- `gold.v_signal_trends`,
- `gold.v_interview_profiles`,
- `gold.v_trend_changes`.

That V2 layer can compare non-personalized Gold market demand with user-owned state from Tutor/App tables, but that comparison result should live outside `gold.*`.

Recommended V2-owned objects, if/when needed:

| Future object | Owner | Why not Gold V1 |
|---|---|---|
| `app.user_target_readiness_snapshots` | Tutor/App | User-specific readiness projection. |
| `app.user_gold_recommendations` | Tutor/App | Personalized action ranking. |
| `app.company_watchlists` | App/Product | User-specific retained preference. |
| `app.watchlist_notifications` | App/Product | User-specific delivery state. |

The boundary is:

```text
Gold V1 = what the interview market is asking.
Tutor/App V2 = what this specific user should do about it.
```

---

# Part G - Final Recommendation

The updated Gold Layer should use:

```text
6 physical tables
+ 8 views/materialized views
```

Physical tables store:

- canonical families,
- canonical questions,
- external references,
- occurrence mappings,
- skill mappings,
- refresh lineage.

Views/materialized views expose:

- interview scope,
- review queue,
- question trends,
- skill trends,
- signal trends,
- interview loop profiles,
- change detection,
- company/role comparisons.

This keeps the schema smaller, easier to rebuild, and more aligned with the separation of responsibilities:

```text
Silver tells us what was reported.
Gold tells us what it means in the interview market.
Tutor/App V2 may later decide what it means for a specific user, outside `gold.*`.
```


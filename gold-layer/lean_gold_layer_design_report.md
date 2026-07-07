# CrackedIn Lean Gold Layer Design Report

**Document type:** Fresh gold-layer architecture and table/view design  
**Project:** CrackedIn / Interview Prep AI  
**Runtime database:** PostgreSQL only  
**Primary decision:** Gold is a market-demand intelligence layer, not a personalization layer  
**Updated decision:** Physical Gold tables store durable decisions; views/materialized views expose derived intelligence  
**Recommended complete Gold footprint:** 6 physical tables + read views/materialized views  
**Recommended V1 footprint:** 5 to 6 physical tables + 3 to 5 views/materialized views  

---

## 1. Executive Decision

You do **not** need 15 to 20 new Gold tables.

The earlier large design was directionally right about the intelligence CrackedIn needs, but it over-expanded the schema because it treated every product answer as a physical table. That is not necessary.

The corrected rule is:

```text
Gold physical tables store durable decisions.
Gold views/materialized views expose derived intelligence.
```

That means:

- Canonical questions should be tables.
- Silver occurrence to canonical question mappings should be tables.
- External platform references should be tables.
- Skill mappings should be tables.
- Batch/refresh metadata should be a table.
- Trend outputs should mostly be views or materialized views.
- Interview profiles should mostly be views or materialized views.
- Signal/failure intelligence should mostly be views or materialized views.
- Company comparisons should be views, not tables.
- Review queue should be a view over occurrence mappings, not a separate V1 table.

The lean final boundary is:

```text
Bronze: ingestion lifecycle and raw/extracted artifact pointers.
Silver: normalized reported interview facts, stored verbatim and structurally.
Gold: canonicalization decisions, question/family registry, skill mappings, and product read models.
Tutor/App: personalization, readiness, recommendations, prep paths, memory, and learning state.
Search: use silver chunks and embeddings; gold supplies structured filters and canonical IDs.
```

Gold should answer:

- What questions are being asked recently?
- Which question families and variants are recurring?
- Which skills/topics are rising for a company, role, and level?
- What round types and interview loop patterns should a candidate expect?
- What failure reasons, prep advice, red flags, or evaluation themes are recurring?
- What changed recently?
- How strong or weak is the evidence?
- Which silver interviews, rounds, items, chunks, signals, and evidence spans support the answer?

Gold should **not** answer by itself:

- What should this specific user study next?
- What is this user's readiness score?
- What has this user solved?
- What is this user's revision priority?
- What is this user's prep path?

Those questions belong to the AI Tutor architecture, which consumes Gold demand signals and combines them with user state.

---

## 2. Final Table/View Decision

### 2.1 Final physical Gold tables

Use these physical tables:

1. `gold.question_families`
2. `gold.questions`
3. `gold.question_references`
4. `gold.question_occurrences`
5. `gold.question_skills`
6. `gold.refresh_runs`

That is the recommended complete physical Gold schema.

### 2.2 Final Gold views/materialized views

Use these read models:

1. `gold.v_interview_scope`
2. `gold.v_review_queue`
3. `gold.v_question_trends`
4. `gold.v_skill_trends`
5. `gold.v_signal_trends`
6. `gold.v_interview_profiles`
7. `gold.v_trend_changes`
8. `gold.v_company_role_comparisons`

These can start as normal views. Convert the expensive/high-traffic ones to materialized views after query volume or latency demands it.

### 2.3 Final operating principle

```text
Tables = canonicalization decisions, mappings, lineage, stable entities.
Views = aggregations, rankings, profiles, comparisons, change summaries.
Materialized views = same as views, but cached for performance and batch consistency.
```

---

## 3. What Changed From the Previous 15-Table Design

The previous proposed Gold design listed 15 Gold tables:

1. `gold.role_levels`
2. `gold.interview_roles`
3. `gold.question_families`
4. `gold.questions`
5. `gold.question_references`
6. `gold.question_occurrences`
7. `gold.question_resolution_reviews`
8. `gold.question_trends`
9. `gold.topic_trends`
10. `gold.interview_profiles`
11. `gold.signal_trends`
12. `gold.search_documents`
13. `gold.trend_changes`
14. `gold.company_role_comparisons`
15. `gold.refresh_runs`

The corrected design removes or reframes most of these.

## 3.1 Remove `gold.role_levels`

Do **not** create this table now.

The current Postgres Silver layer already has:

- `silver.company`
- `silver.canonical_ladder`
- `silver.company_level`
- `silver.level_resolution`
- `silver.interview.company_id`
- `silver.interview.role_family`
- `silver.interview.canonical_level_id`
- `silver.interview.company_level_id`

The company/role/level matching logic already exists. Therefore, Gold should consume those resolved fields instead of creating another authoritative role-level taxonomy.

If level resolution needs review or improvement, improve the existing company-level resolver path. Do not duplicate it as `gold.role_levels`.

## 3.2 Replace `gold.interview_roles` with `gold.v_interview_scope`

Do **not** create a physical `gold.interview_roles` table in V1.

Use a view/materialized view called `gold.v_interview_scope` that selects from `silver.interview` and computes:

- `interview_id`
- `company_id`
- `role_family`
- `canonical_level_id`
- `company_level_id`
- `country`
- `employment_type`
- `interview_date`
- `posted_at`
- date-window eligibility
- public eligibility
- quality weight
- duplicate/authenticity/date filters

Because Silver already stores the interview-level cohort dimensions, a physical Gold table is not necessary at the start.

Materialize this later only if trend jobs or product queries become slow.

## 3.3 Fold `gold.question_resolution_reviews` into `gold.question_occurrences`

Do **not** create a separate review table for V1.

Store review state directly on the occurrence mapping row:

- `routing_status`: `auto`, `review`, `defer`
- `candidate_matches_json`
- `review_status`
- `reviewer_decision`
- `reviewer_notes`
- `reviewed_by`
- `reviewed_at`

A review UI can query:

```text
select * from gold.v_review_queue
```

where `gold.v_review_queue` is a view over `gold.question_occurrences` with `routing_status = 'review'` or `review_status in (...)`.

Split to a separate review table only when you need reviewer assignment, SLA, threaded comments, audit history, or multiple review decisions per occurrence.

## 3.4 Replace `gold.topic_trends` with `gold.v_skill_trends`

The product can still say "topics", but the backend should avoid creating a second topic taxonomy.

The AI Tutor architecture owns a Skill Graph. Gold questions and families should map to Tutor skills through `gold.question_skills`. Gold trend aggregation should then expose `gold.v_skill_trends`, not an unrelated topic table.

If `app.skills` is not fully implemented yet, support transitional `skill_slug` and `skill_name_snapshot` on `gold.question_skills`, then backfill `skill_id` later.

## 3.5 Replace `gold.question_trends` with `gold.v_question_trends`

Do **not** create a physical `gold.question_trends` table initially.

Question trends are derived from:

- `gold.question_occurrences`
- `gold.questions`
- `gold.question_families`
- `silver.interview`
- `silver.round`
- `gold.v_interview_scope`

This makes them a read model. Start as a view or materialized view.

Use a materialized view when:

- product pages need fast reads,
- the query becomes expensive,
- you want stable "last completed refresh" behavior,
- you run Gold as batch-first.

## 3.6 Replace `gold.interview_profiles` with `gold.v_interview_profiles`

Interview loop profiles are aggregations over Silver interviews and rounds.

They should be exposed as a view/materialized view, not a physical V1 table.

## 3.7 Replace `gold.signal_trends` with `gold.v_signal_trends`

Signal trends are aggregations over `silver.signal` joined to `gold.v_interview_scope`.

They should be exposed as a view/materialized view.

## 3.8 Replace `gold.trend_changes` with `gold.v_trend_changes`

Trend changes compare current and previous windows from question, skill, signal, and profile views.

They should be a view/materialized view unless you need durable alert history. Durable user notifications belong in app/user tables, not Gold.

## 3.9 Remove `gold.search_documents`

Do **not** create a Gold search document table in V1.

The Silver layer already has `silver.chunk`, linked to interview, round, item, and signal, with embedding metadata placeholders. Embeddings should live in Silver or a dedicated Silver embedding table keyed by `silver.chunk.id`.

Gold should not duplicate chunk text or embeddings. Gold should only provide structured filters and canonical IDs.

Semantic retrieval should work like this:

```text
SQL filters from silver/gold:
  company_id, role_family, canonical_level_id, round_type, item_type, date window

Vector/full-text search from silver:
  silver.chunk.text, silver.chunk.embedding, chunk_type

Synthesis:
  LLM summarizes retrieved silver evidence and gold trend metadata
```

## 3.10 Replace `gold.company_role_comparisons` with `gold.v_company_role_comparisons`

Do not store company-role comparison snapshots as a table.

Company comparisons can be computed from:

- `gold.v_question_trends`
- `gold.v_skill_trends`
- `gold.v_interview_profiles`
- `gold.v_signal_trends`

Only materialize comparisons if a high-traffic SEO/product page needs precomputed deltas.

## 3.11 Keep personalization out of Gold

Do not create these as part of Gold:

- `app.user_target_readiness_snapshots`
- `app.user_gold_recommendations`
- `app.company_watchlists`
- `app.watchlist_notifications`

Gold can power them, but Tutor/App should own them.

---

# 4. Final Physical Gold Tables

## 4.1 `gold.question_families`

**Grain:** One canonical question/pattern family.

**Purpose:** Group related variants so the product can reason at pattern level, not just exact problem level.

Examples:

- Grid traversal / islands
- Word ladder / shortest transformation path
- URL shortener
- Rate limiter
- Parking lot LLD
- Amazon leadership principle ownership
- Behavioral conflict story

**Core fields:**

- `id`
- `family_key`
- `family_title`
- `family_type`
- `primary_item_type`
- `summary`
- `parent_family_id`, optional
- `canonical_skill_id`, optional transition to Tutor Skill Graph
- `topics_snapshot`, transitional only
- `patterns_snapshot`, transitional only
- `support_count_lifetime`
- `confidence_label`
- `created_from_occurrence_id`
- `created_at`
- `updated_at`

**Why it is necessary:**

Without families, the same market signal fragments across variants. For example, "Number of Islands", "dynamic islands", "max area of island", and "shortest bridge" may each be too sparse alone, but together they show a clear grid traversal trend.

**What not to store:**

Do not store user progress, readiness, recommended lessons, or solved status.

---

## 4.2 `gold.questions`

**Grain:** One canonical interview question/problem concept.

**Purpose:** Product-facing normalized question catalog.

Examples:

- LeetCode 200 Number of Islands
- Design a URL shortener
- Design a rate limiter
- Parking lot LLD
- Behavioral ownership story
- SQL window function debugging task
- Custom company-authored DSA variant

**Core fields:**

- `id`
- `question_key`
- `canonical_title`
- `canonical_slug`
- `item_type`
- `family_id`
- `summary`
- `difficulty_label`
- `is_custom`
- `is_platform_backed`
- `canonicalization_version`
- `support_count_lifetime`
- `first_seen_date`
- `last_seen_date`
- `created_from_occurrence_id`
- `created_at`
- `updated_at`

**Why it is necessary:**

This is the core entity that turns repeated raw prompts into one product object.

**Important design note:**

Do not depend on LeetCode as the source of truth. LeetCode, Codeforces, HackerRank, etc. are references. Gold questions can exist with zero external refs when they are custom company-authored questions.

---

## 4.3 `gold.question_references`

**Grain:** One external reference for one canonical question.

**Purpose:** Map canonical Gold questions to platforms or named references.

Supported references:

- LeetCode
- Codeforces
- HackerRank
- InterviewBit
- GeeksforGeeks
- company-named internal problem
- raw URL
- named design archetype
- custom problem reference

**Core fields:**

- `id`
- `question_id`
- `platform`
- `external_id`
- `external_slug`
- `external_title`
- `external_url`
- `reference_type`: `same_as`, `variant_of`, `mentioned`, `named_problem`, `source_url`
- `confidence_score`
- `source_occurrence_id`
- `created_at`
- `updated_at`

**Why it is necessary:**

One canonical Gold question can map to multiple practice references. Also, some interview prompts only mention `LC 726`, some mention a title, and some mention a URL. This table keeps those references clean and auditable.

---

## 4.4 `gold.question_occurrences`

**Grain:** One row per `silver.assessment_item_occurrence` mapped to a canonical Gold question or deferred cluster.

**Purpose:** Auditable mapping from reported Silver item to canonical Gold question.

This is the most important Gold fact table.

**Core fields:**

- `occurrence_id`
- `interview_id`
- `round_id`
- `question_id`, nullable when unresolved/deferred
- `family_id`, nullable
- `item_type`
- `prompt_hash`
- `mapping_method`: `exact_hash`, `external_ref`, `catalog_match`, `embedding_rerank`, `llm_match`, `manual`, `deferred_cluster`
- `mapping_score`
- `routing_status`: `auto`, `review`, `defer`
- `is_variant`
- `variant_of_question_id`, nullable
- `candidate_matches_json`
- `review_status`
- `reviewer_decision`
- `reviewer_notes`
- `reviewed_by`
- `reviewed_at`
- `public_eligible`
- `quality_weight`
- `resolver_version`
- `refresh_run_id`
- `resolved_at`
- `updated_at`

**Why it is necessary:**

Public frequency must count distinct interviews through this mapping, not repeated mentions. It also gives every trend an audit path back to Silver.

**What not to store:**

Do not duplicate full raw prompts unless performance proves it necessary. Store IDs and hashes. Fetch full text from Silver when needed.

**Key invariant:**

Every occurrence should land in one of three states:

```text
auto mapped -> safe to aggregate
review      -> visible in review queue, excluded or downweighted
defer       -> unresolved/emerging custom entity pool
```

---

## 4.5 `gold.question_skills`

**Grain:** One mapping between a Gold question/family and a Tutor Skill Graph skill.

**Purpose:** Avoid a separate Gold-only topic taxonomy. Gold market demand should map onto the same skill graph that the Tutor uses.

**Core fields:**

- `id`
- `question_id`, nullable
- `family_id`, nullable
- `skill_id`, nullable until Tutor Skill Graph is fully migrated
- `skill_slug`, transitional
- `skill_name_snapshot`, transitional
- `relation_type`: `primary`, `secondary`, `prerequisite`, `followup`, `evaluation_focus`, `behavioral_theme`
- `relevance_weight`
- `confidence_score`
- `mapping_source`: `attributes`, `catalog`, `resolver`, `manual`, `llm`
- `review_status`
- `created_at`
- `updated_at`

**Why it is necessary:**

The user may ask, "what topics are rising?" while the Tutor needs, "what skills matter for this user's target?" This bridge makes both use the same conceptual layer.

**Transitional rule:**

If `app.skills` is not ready, store `skill_slug` and `skill_name_snapshot`; later backfill `skill_id`.

---

## 4.6 `gold.refresh_runs`

**Grain:** One Gold job run.

**Purpose:** Observability, replay, and audit.

**Core fields:**

- `id`
- `job_name`
- `job_type`: `resolver`, `aggregate`, `trend_change`, `materialized_view_refresh`, `embedding_dependency`, `full_refresh`
- `status`
- `source_min_interview_date`
- `source_max_interview_date`
- `source_max_updated_at`
- `repository_commit_sha`
- `resolver_version`
- `embedding_model_version`, nullable
- `llm_model_version`, nullable
- `rows_inserted`
- `rows_updated`
- `rows_skipped`
- `rows_deferred`
- `views_refreshed_json`
- `error_summary`
- `metadata_json`
- `started_at`
- `finished_at`

**Why it is necessary:**

Gold is derived. Every derived product answer should know which refresh produced it.

---

# 5. Gold Views and Materialized Views

## 5.1 View strategy

Use views where the object only duplicates existing facts or aggregations.

Use materialized views where:

- product traffic is high,
- queries become expensive,
- stable batch snapshots matter,
- company pages need low latency,
- refresh consistency matters more than always-live data.

Recommended default:

```text
V1 development: normal views
V1 launch/product pages: materialized views for high-traffic read models
Later: incrementally refreshed materialized views if needed
```

---

## 5.2 `gold.v_interview_scope`

**Type:** View first; materialized view if performance requires.

**Source:** `silver.interview`

**Purpose:** Standardize scope and quality filters for all Gold read models.

Should expose:

- `interview_id`
- `company_id`
- `role_family`
- `canonical_level_id`
- `company_level_id`
- `country`
- `employment_type`
- `interview_date`
- `posted_at`
- `is_date_estimated`
- `is_public_eligible`
- `quality_weight`
- `duplicate_state`
- `authenticity_state`

Suggested public eligibility logic:

```text
not llm_rejected
and company_id is not null
and interview_date is not null
and interview_date_estimated = false for trend charts
and possible_duplicate_of is null, or weighted down
and authenticity_suspect is not true, or excluded from public trends
and extract_confidence is above threshold, or weighted down
```

**Why view, not table:**

The source fields already live in Silver. This is a standardized read surface, not a new durable fact.

---

## 5.3 `gold.v_review_queue`

**Type:** View.

**Source:** `gold.question_occurrences`

**Purpose:** Provide the review UI/workflow with ambiguous mappings.

Should expose rows where:

- `routing_status = 'review'`, or
- `review_status in ('pending', 'needs_second_pass')`, or
- `question_id is null and routing_status != 'defer'`, depending on final workflow.

**Why view, not table:**

Review state already lives on the occurrence mapping. A separate table is premature until review workflow becomes more complex.

---

## 5.4 `gold.v_question_trends`

**Type:** Materialized view for product pages; normal view acceptable during development.

**Source:**

- `gold.question_occurrences`
- `gold.questions`
- `gold.question_families`
- `gold.v_interview_scope`
- `silver.round`

**Purpose:** Fast answer for questions like:

- What are trending questions at Google for backend mid-level?
- What DSA questions are repeatedly asked at Amazon SDE2?
- Which system-design prompts are rising for senior backend?
- What questions appeared in behavioral rounds?

Should expose:

- `scope_type`
- `company_id`
- `role_family`
- `canonical_level_id`
- `company_level_id`
- `item_type`
- `question_id`
- `family_id`
- `window_days`: `90`, `180`, `365`, `540`
- `distinct_interview_count`
- `raw_occurrence_count`
- `weighted_frequency`
- `recency_score`
- `trend_score`
- `confidence_label`
- `fallback_scope_used`
- `first_seen_date`
- `last_seen_date`
- `round_type_counts_json`
- `common_followups_json`
- `observed_difficulty_json`
- `sample_occurrence_ids_json`
- `sample_interview_ids_json`
- `sample_evidence_span_ids_json`
- `refreshed_at` for materialized version

**Ranking rule:**

Trend score should never be pure lifetime frequency. It should combine recent distinct-interview support, recency decay, quality weight, and scope closeness.

**Why view/materialized view, not table:**

This is a deterministic aggregation over persisted canonicalization decisions.

---

## 5.5 `gold.v_skill_trends`

**Type:** Materialized view for product pages; normal view acceptable during development.

**Source:**

- `gold.question_occurrences`
- `gold.question_skills`
- `gold.questions`
- `gold.question_families`
- `gold.v_interview_scope`

**Purpose:** Fast answer for:

- What topics is this company focusing on right now?
- Are graph questions rising for this role?
- Is behavioral ownership high for Amazon?
- Are SQL and data modeling frequent for data roles?

Should expose:

- `scope_type`
- `company_id`
- `role_family`
- `canonical_level_id`
- `company_level_id`
- `item_type`, nullable for cross-item skill trends
- `skill_id`, nullable during transition
- `skill_slug`
- `skill_name_snapshot`
- `window_days`
- `distinct_interview_count`
- `distinct_question_count`
- `weighted_frequency`
- `trend_score`
- `trend_direction`: `rising`, `stable`, `declining`, `new`, `resurfacing`
- `confidence_label`
- `top_question_ids_json`
- `top_family_ids_json`
- `sample_occurrence_ids_json`
- `last_seen_date`
- `refreshed_at` for materialized version

**Why view/materialized view, not table:**

Skill trends are derived from question-skill mappings and occurrence support. The durable decision is the mapping in `gold.question_skills`, not the aggregate row.

---

## 5.6 `gold.v_signal_trends`

**Type:** Materialized view when failure/prep/process intelligence becomes product-facing.

**Source:**

- `silver.signal`
- `gold.v_interview_scope`
- `silver.round`
- optionally `silver.assessment_item_occurrence`

**Purpose:** Aggregate Silver signals into product intelligence.

Answers:

- Why do candidates fail this company or round?
- What prep advice recurs?
- What red flags or process issues recur?
- What evaluation behaviors are common?

Should expose:

- `scope_type`
- `company_id`
- `role_family`
- `canonical_level_id`
- `company_level_id`
- `round_type`, nullable
- `item_type`, nullable
- `signal_type`
- `signal_subtype`
- `window_days`
- `distinct_interview_count`
- `raw_signal_count`
- `weighted_frequency`
- `trend_score`
- `confidence_label`
- `sample_signal_ids_json`
- `sample_interview_ids_json`
- `sample_evidence_span_ids_json`
- `last_seen_date`
- `refreshed_at` for materialized version

**Why view/materialized view, not table:**

Signals already live in Silver. Gold should expose product-ready aggregations without duplicating the signal facts.

---

## 5.7 `gold.v_interview_profiles`

**Type:** Materialized view for company/role guide pages.

**Source:**

- `gold.v_interview_scope`
- `silver.round`
- `silver.assessment_item_occurrence`
- `silver.signal`
- `gold.question_occurrences`

**Purpose:** Describe expected interview process and round mix.

Answers:

- Does this company usually have an OA?
- Does this role ask system design?
- How common is behavioral?
- How many rounds should candidates expect?
- Which item types dominate the loop?

Should expose:

- `scope_type`
- `company_id`
- `role_family`
- `canonical_level_id`
- `company_level_id`
- `window_days`
- `total_interviews`
- `total_rounds`
- `average_round_count`
- `median_round_count`
- `median_process_duration_days`
- `round_type_mix_json`
- `item_type_mix_json`
- `ordered_round_patterns_json`
- `top_skill_ids_json`
- `top_question_ids_json`
- `top_signal_keys_json`
- `confidence_label`
- `first_seen_date`
- `last_seen_date`
- `sample_interview_ids_json`
- `refreshed_at` for materialized version

**Why view/materialized view, not table:**

The profile is computed from Silver facts and Gold mappings. It has no independent durable decision.

---

## 5.8 `gold.v_trend_changes`

**Type:** Materialized view after trend windows are stable.

**Source:**

- `gold.v_question_trends`
- `gold.v_skill_trends`
- `gold.v_signal_trends`
- `gold.v_interview_profiles`

**Purpose:** Power "what changed recently?", rising/new/declining trend summaries, and watchlist-like app features.

Should expose:

- `entity_type`: `question`, `family`, `skill`, `signal`, `round_profile`
- `entity_id`, nullable for profile-level changes
- `entity_key`
- `scope_type`
- `company_id`, nullable
- `role_family`, nullable
- `canonical_level_id`, nullable
- `item_type`, nullable
- `current_window_days`
- `previous_window_days`
- `current_score`
- `previous_score`
- `delta_score`
- `change_type`: `new`, `rising`, `stable`, `declining`, `resurfacing`, `disappeared`
- `confidence_label`
- `support_count_current`
- `support_count_previous`
- `sample_ids_json`
- `detected_at`

**Why view/materialized view, not table:**

Trend changes are computed by comparing current and previous aggregate windows. Durable notification delivery history belongs in app/user tables, not Gold.

---

## 5.9 `gold.v_company_role_comparisons`

**Type:** View first; materialized view only for high-traffic pages.

**Source:**

- `gold.v_question_trends`
- `gold.v_skill_trends`
- `gold.v_interview_profiles`
- `gold.v_signal_trends`

**Purpose:** Compare companies/roles/levels without storing static snapshots.

Answers:

- How does Google SDE2 differ from Amazon SDE2?
- Which company is more graph-heavy?
- Which company asks more behavioral?
- Which company has more system design?
- Which roles are more OA-heavy?

**Why view/materialized view, not table:**

Comparison is a query pattern, not a source-of-truth fact.

---

# 6. Embedding and Semantic Search Design

Gold should not store embeddings for rounds or duplicate Silver chunks.

Store embeddings in Silver, specifically on `silver.chunk` or a follow-up Silver embedding table keyed by chunk ID.

The Silver chunk design already links each chunk to:

- `interview_id`
- `round_id`
- `item_id`
- `signal_id`
- `chunk_type`
- `text`
- `source_path`
- `embedding_model`
- `embed_version`

The follow-up migration should add:

- `embedding vector`
- HNSW or IVFFlat index, depending on pgvector choice
- full-text search vector if not already present

## 6.1 How to answer: "What was asked in behavioral round in this company?"

Use hybrid retrieval:

1. SQL filter:
   - company_id = target company
   - round_type = behavioral
   - optional role_family/canonical_level_id
   - interview_date within window
   - not rejected / public eligible

2. Search Silver chunks:
   - `silver.chunk.round_id`
   - `silver.chunk.chunk_type in ('round_narrative', 'candidate_answer', 'feedback', 'prep_advice')`
   - vector/full-text query for the user's wording

3. Join Gold where needed:
   - map behavioral items to canonical behavioral questions/families through `gold.question_occurrences`
   - include `gold.v_signal_trends` for repeated behavioral themes and failure signals

4. Synthesize:
   - return top behavioral themes/questions
   - include sample snippets from Silver
   - include trend confidence and support count

This avoids a duplicate Gold search corpus while still supporting semantic Q&A.

---

# 7. Canonicalization Pipeline

Gold should canonicalize raw Silver assessment items into stable question entities.

## 7.1 Resolver ladder

For each `silver.assessment_item_occurrence`:

1. **Exact prompt hash**
   - If `prompt_hash` already mapped, reuse the same canonical question.

2. **External reference parsing**
   - Parse LeetCode IDs, URLs, slugs, Codeforces IDs, HackerRank names, and named design archetypes.

3. **Catalog match**
   - Match known platform-backed problems against `catalog.coding_problems` or other future catalogs.

4. **Lexical/title matching**
   - Normalize title-like prompts and match obvious canonical question names.

5. **Embedding candidate retrieval**
   - Retrieve candidate Gold questions/families using embedding similarity.

6. **Rerank / cross-encoder step**
   - Avoid false positives among similar topics.

7. **LLM match scoring only for ambiguous middle cases**
   - Ask whether candidate and occurrence describe the same question, variant, or only same family.

8. **Review or defer**
   - High confidence -> auto map.
   - Medium confidence -> review.
   - Low confidence -> defer/emerging cluster.

9. **Recurring unresolved promotion**
   - If deferred occurrences cluster across enough distinct interviews, promote to a custom Gold question or family.

## 7.2 Do not force-map everything

A company-authored question may not be a known LeetCode problem. If you force it into the closest known question, you destroy the moat.

Correct behavior:

```text
known exact problem -> map to known canonical question
known variant -> map as variant
same broad pattern only -> map to family, not exact question
novel repeated prompt -> promote to custom question
one-off unclear prompt -> keep deferred / low confidence
```

## 7.3 Distinct interview counting rule

Public frequency must use:

```text
count(distinct interview_id)
```

not raw occurrence count.

Reason: one interview report can mention the same question multiple times. It should not inflate public support.

---

# 8. Scoring Model

## 8.1 Core metrics

Every trend view should expose:

- distinct interview count
- raw occurrence count
- weighted frequency
- recency score
- trend score
- confidence label
- first seen date
- last seen date
- sample occurrence IDs
- fallback scope used

## 8.2 Trend score

Trend score should combine:

```text
recent distinct-interview support
+ recency decay
+ quality weight
+ scope closeness
+ confidence of canonical mapping
- duplicate/authenticity/date penalties
```

## 8.3 Scope fallback

Use strict scope fallback:

1. exact company + role family + canonical level
2. same company + role family
3. same company, any role
4. same role family + level globally
5. same role family globally
6. global all

Product responses must show the scope used.

Example:

```text
Strong signal: 8 recent Google backend mid-level interviews.
Moderate signal: 11 Google backend interviews, level mixed.
Weak signal: sparse Google data, using global backend fallback.
```

## 8.4 Confidence labels

Use product-visible labels:

- strong signal
- moderate signal
- weak signal
- sparse data
- review pending

Confidence should consider:

- support count
- freshness
- exactness of scope
- mapping confidence
- quality weight
- duplicate/authenticity flags
- date precision

---

# 9. Query Examples

## 9.1 Trending DSA questions for Google Mid Backend

Flow:

1. Resolve company -> `silver.company.id`.
2. Resolve role/level -> `role_family`, `canonical_level_id`.
3. Query `gold.v_question_trends`.
4. Filter `item_type = 'dsa_coding'`.
5. Prefer exact scope and 180-day window.
6. Join `gold.questions` and `gold.question_references`.
7. Return score, confidence, last seen, support, and sample prompts.

## 9.2 What is this company mainly focusing on right now?

Flow:

1. Query `gold.v_skill_trends` for the company/role/level.
2. Use 90/180-day windows.
3. Group by skill and item type.
4. Show rising/stable/declining skills.
5. Use `gold.v_question_trends` for top supporting questions.
6. Use `gold.v_signal_trends` for failure/prep themes.

## 9.3 What was asked in behavioral rounds at Amazon?

Flow:

1. Query `gold.v_question_trends` with behavioral item types.
2. Query `gold.v_signal_trends` for behavioral signal types.
3. SQL-filter Silver rounds where `round_type = 'behavioral'`.
4. Use Silver chunks/evidence for semantic examples.
5. Return themes, sample prompts, support count, and confidence.

## 9.4 What changed recently for Meta E4?

Flow:

1. Query `gold.v_trend_changes`.
2. Filter company and canonical level.
3. Show new/rising/declining questions, skills, and signals.
4. Link back to `gold.v_question_trends` and Silver examples.

## 9.5 Compare Google SDE2 vs Amazon SDE2

Flow:

1. Query `gold.v_company_role_comparisons`, or compute from individual trend/profile views.
2. Compare skill mix, question mix, round mix, signal/failure mix.
3. Show confidence and support counts for each side.

---

# 10. Refresh Strategy

Gold should be batch-first.

Recommended sequence:

1. Silver extraction completes.
2. `gold.refresh_runs` row starts.
3. Resolver maps new/changed Silver assessment items into `gold.question_occurrences`.
4. New canonical `gold.questions` / `gold.question_families` / `gold.question_references` are created only when needed.
5. `gold.question_skills` is updated or reviewed.
6. Materialized views are refreshed in dependency order:
   - `gold.v_interview_scope`
   - `gold.v_question_trends`
   - `gold.v_skill_trends`
   - `gold.v_signal_trends`
   - `gold.v_interview_profiles`
   - `gold.v_trend_changes`
   - `gold.v_company_role_comparisons`
7. `gold.refresh_runs` row finishes with counts and status.

## 10.1 Immediate overlay

If near-real-time freshness is needed, only use deterministic immediate mapping:

- exact prompt hash
- known external reference
- known catalog slug

Do **not** run ambiguous embedding/LLM canonicalization inside the ingest transaction.

Product behavior:

```text
Public pages: latest completed materialized refresh.
Admin preview: latest completed refresh + deterministic pending overlay.
AI chat: can mention fresh low-confidence signals if clearly labeled.
```

---

# 11. What To Build First

## Phase 1: Physical table foundation

Create:

1. `gold.question_families`
2. `gold.questions`
3. `gold.question_references`
4. `gold.question_occurrences`
5. `gold.question_skills`
6. `gold.refresh_runs`

## Phase 2: Core views

Create:

1. `gold.v_interview_scope`
2. `gold.v_review_queue`
3. `gold.v_question_trends`
4. `gold.v_skill_trends`

This is enough to answer:

- What questions are trending?
- What topics/skills are rising?
- What is this company mainly asking?
- What should the AI Tutor consider as market demand?

## Phase 3: Product intelligence views

Add:

1. `gold.v_interview_profiles`
2. `gold.v_signal_trends`
3. `gold.v_trend_changes`
4. `gold.v_company_role_comparisons`

This unlocks:

- interview loop predictions
- failure reason intelligence
- change detection
- company comparisons

## Phase 4: Materialization and performance

Convert high-traffic views into materialized views:

- `gold.v_question_trends`
- `gold.v_skill_trends`
- `gold.v_interview_profiles`
- `gold.v_signal_trends`
- `gold.v_trend_changes`

Keep the names stable. Internally, decide whether each object is a normal view or materialized view.

---

# 12. Final Recommended Gold Footprint

## 12.1 Physical tables

```text
1. gold.question_families
2. gold.questions
3. gold.question_references
4. gold.question_occurrences
5. gold.question_skills
6. gold.refresh_runs
```

## 12.2 Views/materialized views

```text
1. gold.v_interview_scope
2. gold.v_review_queue
3. gold.v_question_trends
4. gold.v_skill_trends
5. gold.v_signal_trends
6. gold.v_interview_profiles
7. gold.v_trend_changes
8. gold.v_company_role_comparisons
```

## 12.3 Not part of Gold V1

```text
gold.role_levels
gold.interview_roles
gold.question_resolution_reviews
gold.question_trends as physical table
gold.skill_trends as physical table
gold.interview_profiles as physical table
gold.signal_trends as physical table
gold.trend_changes as physical table
gold.company_role_comparisons as physical table
gold.search_documents
app user readiness/recommendation/watchlist tables under Gold scope
```

---

# 13. Final CTO Recommendation

Build Gold as a small, durable canonicalization layer plus cached read models.

The right design is not:

```text
15-20 physical Gold tables
```

The right design is:

```text
6 physical Gold tables
+ views/materialized views for trends, profiles, comparisons, and change detection
```

This gives you:

- fewer migrations,
- less duplicated data,
- easier rebuilds,
- stable canonicalization auditability,
- faster product reads when materialized,
- cleaner separation from AI Tutor personalization,
- no duplicate search corpus,
- no duplicate topic taxonomy,
- no duplicate role-level taxonomy.

The core principle to preserve is:

```text
Silver tells us what was reported.
Gold tells us what it means in the interview market.
Tutor tells us what it means for this specific user.
```

And at the storage level:

```text
Gold tables store decisions.
Gold views expose intelligence.
```

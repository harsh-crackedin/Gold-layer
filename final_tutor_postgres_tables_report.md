# Final Tutor PostgreSQL Tables Report

**Project:** Crackedin Labs AI Tutor / Creator Platform  
**Status:** Final schema proposal for new tutor source-of-truth tables  
**Runtime database:** PostgreSQL  
**Primary focus:** DSA-first Skill Graph and Learner Skill Graph, future-proofed for system design, LLD, OS, networks, databases, SQL, distributed systems, cloud, debugging, and company-specific interview intelligence.

---

## 1. Executive decision

The tutor database should be modeled around two source-of-truth layers:

1. **Skill Graph** - the global shared curriculum graph.
2. **Learner Skill Graph** - user-specific evidence and projections over the global graph.

The system should not create DSA-specific tables such as `dsa_topics`, `dsa_subtopics`, or `dsa_problem_topics`. DSA is the first seeded domain, not a special schema. Future domains should use the same tables.

The final new tutor core is:

| Layer | Table |
|---|---|
| Skill Graph | `app.skill_areas` |
| Skill Graph | `app.skills` |
| Skill Graph | `app.skill_aliases` |
| Skill Graph | `app.skill_relationships` |
| Skill Graph / Learner bridge | `app.learning_items` |
| Skill Graph / Learner bridge | `app.learning_item_skills` |
| Retrieval | `app.knowledge_sources` |
| Retrieval | `app.knowledge_chunks` |
| Learner Skill Graph | `app.learning_events` |
| Learner Skill Graph | `app.learner_skill_state` |
| Learner Skill Graph | `app.user_learning_item_state` |
| Personalization | `app.user_context_events` |
| Personalization | `app.user_profile_summary` |

Future/deferred tables:

| Layer | Table | Status |
|---|---|---|
| Company intelligence | `app.companies` | Deferred until company-specific prep is needed |
| Company intelligence | `app.company_skill_demand` | Deferred |
| Company intelligence | `app.company_learning_item_demand` | Deferred |
| Observability | `app.memory_retrieval_logs` | Optional, not V1 |

---

## 1.1 Source basis

This report reconciles the prior architecture sources:

- `AI Tutor Architecture Final Design Report` as the primary design source.
- `Crackedin Labs Tutor Architecture Unified Context Report` for runtime flow and service boundaries.
- `Tutor Architecture PostgreSQL Tables Reference` for the original 7-table kernel and existing table roles.
- `MemZero / Mem0 Integration Design Report` for personalization boundaries.

The final schema below intentionally focuses on the new tutor source-of-truth tables. Existing tables are referenced only as integration anchors.

---

## 2. Existing tables overview

These tables already exist and should be reused. They are not the main focus of this report.

| Existing table | Role in tutor architecture |
|---|---|
| `public.users` | Canonical identity anchor. All user-owned tutor data references this table. |
| `app.chat_sessions` | Conversation session container. |
| `app.chat_messages` | Raw conversation history and evidence references. Not learner state. |
| `app.chat_feedback` | Message-level feedback. Can influence extraction confidence and quality signals. |
| `app.api_logs` | Operational logs only. Not source of truth for learner progress. |
| `app.uc_forget_audit` | Privacy/delete audit. |
| `app.user_prep_plans` | Durable multi-day or multi-week roadmap/prep plans. Not TutorPlan. |
| `app.user_prep_task_progress` | Progress against durable prep-plan tasks. |
| `app.user_prep_plan_feedback` | Feedback on prep plans. |
| `catalog.coding_problems` | Existing global coding catalog. Backfilled into `app.learning_items`. |
| `user_data.user_coding_profiles` | User's connected coding-platform profile. |
| `user_data.user_coding_problems` | Per-user coding problem aggregate state. |
| `user_data.user_coding_submissions` | Strong DSA evidence from submissions/verdicts. |
| `code_blobs.user_coding_submission_code` | Submitted code blobs for review/debugging flows. |

---

## 3. Global schema conventions

### 3.1 PostgreSQL extensions

```sql
CREATE EXTENSION IF NOT EXISTS pgcrypto;
CREATE EXTENSION IF NOT EXISTS pg_trgm;
-- Optional later if semantic retrieval is required:
-- CREATE EXTENSION IF NOT EXISTS vector;
```

### 3.2 Type conventions

| Use | Type |
|---|---|
| Primary keys | `uuid` with `gen_random_uuid()` |
| Foreign keys | `uuid` |
| Slugs and enum-like values | `text` with `CHECK` constraints |
| Flexible metadata | `jsonb` |
| Scores | `numeric(5,4)` or `numeric(6,4)` |
| Time | `timestamptz` |
| Full-text search | `tsvector` |
| Embeddings | Optional `vector(1536)` or model-specific dimension |

### 3.3 Enum strategy

Use `text` with `CHECK` constraints for V1. Avoid PostgreSQL enum types initially because taxonomy values will evolve while the product learns.

### 3.4 Update timestamp policy

Every mutable table has `updated_at`. Use a shared trigger or application-level write helper to refresh it.

### 3.5 Append-only policy

`app.learning_events` and `app.user_context_events` should be treated as append-only ledgers. If a previous event is wrong, add a correction or invalidation event instead of silently editing history.

---

## 4. Design hardening decisions

The schemas below include several corrections and hardening choices:

1. **One canonical skill row per competency.** Similar terms become aliases; genuinely different competencies become separate skills connected by graph edges.
2. **Hierarchy is not dependency.** `skills.parent_skill_id` is for primary display hierarchy. `skill_relationships` owns prerequisites, variants, overlaps, confusion, and unlocks.
3. **No subtopic JSON.** Subtopics are rows in `app.skills`, not arrays inside a topic row.
4. **Learning items are many-to-many.** Every coding problem, diagnostic, or system design case can map to multiple skills through `learning_item_skills`.
5. **User-generated saved items are supported.** `learning_items` includes `visibility` and `created_by_user_id` so creator flows can save private or global items later.
6. **Events separate `occurred_at` from `created_at`.** Imported coding submissions may have historical occurrence times.
7. **Demand tables use normalized non-null scope fields.** Avoid invalid composite primary keys with `coalesce()` expressions.
8. **Mem0 is not source of truth.** User memory can personalize responses, but mastery/readiness remain in Postgres learner tables.

---

# 5. New core tables

---

## 5.1 `app.skill_areas`

### Purpose

Stores high-level domains such as DSA, operating systems, computer networks, databases, SQL, system design, low-level design, distributed systems, cloud, and debugging.

DSA is the first seeded area.

### Use cases

- Resolve a user query to a domain.
- Filter skills, learning items, and knowledge chunks by domain.
- Enable or disable future domains without schema changes.
- Drive UI navigation and prep-plan domain grouping.

### Schema

```sql
CREATE TABLE app.skill_areas (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),

  slug text NOT NULL UNIQUE,
  name text NOT NULL,
  description text,

  display_order integer NOT NULL DEFAULT 0,
  is_active boolean NOT NULL DEFAULT true,

  metadata_json jsonb NOT NULL DEFAULT '{}'::jsonb,

  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now(),

  CONSTRAINT skill_areas_slug_format
    CHECK (slug ~ '^[a-z0-9][a-z0-9_]*$')
);
```

### Important indexes

```sql
CREATE INDEX idx_skill_areas_active_order
  ON app.skill_areas(is_active, display_order);
```

### Example row

```json
{
  "id": "11111111-1111-1111-1111-111111111111",
  "slug": "dsa",
  "name": "Data Structures and Algorithms",
  "description": "Interview-focused DSA skills, patterns, and problem-solving techniques.",
  "display_order": 1,
  "is_active": true,
  "metadata_json": {
    "domain_type": "interview_prep",
    "supports_coding_evidence": true,
    "default_verification_type": "code_acceptance"
  }
}
```

---

## 5.2 `app.skills`

### Purpose

Stores canonical skill nodes. This table replaces the idea of separate topic and subtopic tables.

A skill can be:

- module
- concept
- subskill
- pattern
- technique
- competency
- case
- checkpoint

Examples:

- `arrays_hashing`
- `sliding_window`
- `variable_size_window`
- `bfs`
- `grid_bfs`
- `dynamic_programming`
- `knapsack_dp`
- `rate_limiter`
- `deadlock`
- `tcp_handshake`

### Use cases

- Resolve user messages to canonical skills.
- Build the Skill Graph.
- Track per-user skill state.
- Retrieve related learning items and knowledge chunks.
- Create future OS/network/system-design domains without new tables.

### Schema

```sql
CREATE TABLE app.skills (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),

  area_id uuid NOT NULL REFERENCES app.skill_areas(id),
  slug text NOT NULL,
  name text NOT NULL,
  description text,

  parent_skill_id uuid,

  skill_type text NOT NULL,
  difficulty text,
  depth_level integer NOT NULL DEFAULT 0,
  display_order integer NOT NULL DEFAULT 0,

  is_foundational boolean NOT NULL DEFAULT false,
  is_interview_relevant boolean NOT NULL DEFAULT true,
  is_active boolean NOT NULL DEFAULT true,

  metadata_json jsonb NOT NULL DEFAULT '{}'::jsonb,

  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now(),

  CONSTRAINT skills_unique_area_slug UNIQUE (area_id, slug),
  CONSTRAINT skills_unique_id_area UNIQUE (id, area_id),

  CONSTRAINT skills_parent_same_area_fk
    FOREIGN KEY (parent_skill_id, area_id)
    REFERENCES app.skills(id, area_id),

  CONSTRAINT skills_slug_format
    CHECK (slug ~ '^[a-z0-9][a-z0-9_]*$'),

  CONSTRAINT skills_skill_type_check CHECK (
    skill_type IN (
      'module',
      'concept',
      'subskill',
      'pattern',
      'technique',
      'competency',
      'case',
      'checkpoint'
    )
  ),

  CONSTRAINT skills_difficulty_check CHECK (
    difficulty IS NULL OR difficulty IN (
      'beginner',
      'easy',
      'medium',
      'hard',
      'advanced',
      'expert'
    )
  )
);
```

### Important indexes

```sql
CREATE INDEX idx_skills_area_parent
  ON app.skills(area_id, parent_skill_id);

CREATE INDEX idx_skills_parent
  ON app.skills(parent_skill_id);

CREATE INDEX idx_skills_area_type_active
  ON app.skills(area_id, skill_type, is_active);

CREATE INDEX idx_skills_name_trgm
  ON app.skills USING gin (name gin_trgm_ops);
```

### Example row

```json
{
  "id": "22222222-2222-2222-2222-222222222222",
  "area_id": "11111111-1111-1111-1111-111111111111",
  "slug": "sliding_window",
  "name": "Sliding Window",
  "description": "Array/string pattern for maintaining a moving range under constraints.",
  "parent_skill_id": "aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa",
  "skill_type": "pattern",
  "difficulty": "medium",
  "depth_level": 2,
  "display_order": 30,
  "is_foundational": false,
  "is_interview_relevant": true,
  "metadata_json": {
    "common_misconceptions": [
      "Confuses fixed-size and variable-size windows",
      "Forgets to shrink the left pointer after constraint violation"
    ],
    "default_verification_type": "coding_problem",
    "mvp_seed": true
  }
}
```

### Modeling note

`parent_skill_id` should define the primary UI/curriculum hierarchy only. It should not be used to infer prerequisites. Use `app.skill_relationships` for prerequisites and overlaps.

---

## 5.3 `app.skill_aliases`

### Purpose

Maps real user language to canonical skills.

Examples:

| Alias | Canonical skill |
|---|---|
| `two pointer` | `two_pointers` |
| `fast slow` | `fast_slow_pointers` |
| `mono stack` | `monotonic_stack` |
| `dsu` | `union_find` |
| `uf` | `union_find` |
| `topo sort` | `topological_sort` |
| `kadane` | `maximum_subarray_kadane` |
| `hld` | `system_design` or `high_level_design`, depending on product taxonomy |

### Use cases

- Resolve user queries reliably without relying only on LLM inference.
- Support abbreviations, synonyms, misspellings, and company/platform phrases.
- Represent ambiguity by allowing the same alias to point to multiple candidate skills.

### Schema

```sql
CREATE TABLE app.skill_aliases (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),

  skill_id uuid NOT NULL REFERENCES app.skills(id) ON DELETE CASCADE,

  alias text NOT NULL,
  normalized_alias text NOT NULL,
  alias_type text NOT NULL DEFAULT 'synonym',
  language text NOT NULL DEFAULT 'en',

  confidence numeric(5,4) NOT NULL DEFAULT 1.0000,
  is_active boolean NOT NULL DEFAULT true,

  metadata_json jsonb NOT NULL DEFAULT '{}'::jsonb,

  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now(),

  CONSTRAINT skill_aliases_alias_type_check CHECK (
    alias_type IN (
      'synonym',
      'abbreviation',
      'common_phrase',
      'company_phrase',
      'misspelling',
      'platform_tag'
    )
  ),

  CONSTRAINT skill_aliases_confidence_check
    CHECK (confidence BETWEEN 0 AND 1)
);
```

### Important indexes

```sql
CREATE UNIQUE INDEX uq_skill_aliases_skill_normalized
  ON app.skill_aliases(skill_id, normalized_alias, language);

CREATE INDEX idx_skill_aliases_normalized_trgm
  ON app.skill_aliases USING gin (normalized_alias gin_trgm_ops);

CREATE INDEX idx_skill_aliases_skill
  ON app.skill_aliases(skill_id);
```

### Example row

```json
{
  "id": "33333333-3333-3333-3333-333333333333",
  "skill_id": "22222222-2222-2222-2222-222222222222",
  "alias": "variable window",
  "normalized_alias": "variable window",
  "alias_type": "common_phrase",
  "language": "en",
  "confidence": 0.9200,
  "is_active": true,
  "metadata_json": {
    "notes": "Usually maps to sliding window in DSA questions."
  }
}
```

---

## 5.4 `app.skill_relationships`

### Purpose

Stores graph edges between skills.

This is the table that handles:

- prerequisites
- part-of relationships
- variants
- related concepts
- commonly confused concepts
- unlock paths
- cross-domain relationships

### Use cases

- Determine missing prerequisites before answering.
- Repair weak foundations.
- Recommend next topics.
- Explain contrast questions.
- Detect overlap between DSA and future domains.

### Schema

```sql
CREATE TABLE app.skill_relationships (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),

  source_skill_id uuid NOT NULL REFERENCES app.skills(id) ON DELETE CASCADE,
  target_skill_id uuid NOT NULL REFERENCES app.skills(id) ON DELETE CASCADE,

  relationship_type text NOT NULL,
  weight numeric(5,4) NOT NULL DEFAULT 1.0000,

  min_readiness_required numeric(5,4),
  is_blocking_prerequisite boolean NOT NULL DEFAULT false,

  is_active boolean NOT NULL DEFAULT true,
  metadata_json jsonb NOT NULL DEFAULT '{}'::jsonb,

  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now(),

  CONSTRAINT skill_relationships_no_self_edge
    CHECK (source_skill_id <> target_skill_id),

  CONSTRAINT skill_relationships_relationship_type_check CHECK (
    relationship_type IN (
      'requires',
      'part_of',
      'related_to',
      'contrasts_with',
      'often_confused_with',
      'unlocks',
      'variant_of',
      'uses_technique'
    )
  ),

  CONSTRAINT skill_relationships_weight_check
    CHECK (weight BETWEEN 0 AND 1),

  CONSTRAINT skill_relationships_min_readiness_check
    CHECK (min_readiness_required IS NULL OR min_readiness_required BETWEEN 0 AND 1)
);
```

### Important indexes

```sql
CREATE UNIQUE INDEX uq_skill_relationships_edge
  ON app.skill_relationships(source_skill_id, target_skill_id, relationship_type);

CREATE INDEX idx_skill_relationships_source_type
  ON app.skill_relationships(source_skill_id, relationship_type, is_active);

CREATE INDEX idx_skill_relationships_target_type
  ON app.skill_relationships(target_skill_id, relationship_type, is_active);
```

### Example row

```json
{
  "id": "44444444-4444-4444-4444-444444444444",
  "source_skill_id": "77777777-7777-7777-7777-777777777777",
  "target_skill_id": "88888888-8888-8888-8888-888888888888",
  "relationship_type": "requires",
  "weight": 0.9500,
  "min_readiness_required": 0.6000,
  "is_blocking_prerequisite": true,
  "is_active": true,
  "metadata_json": {
    "reason": "Dijkstra requires priority queue understanding."
  }
}
```

### Relationship direction rule

Use this convention:

```text
source_skill requires target_skill
```

Example:

```text
dijkstra requires priority_queue
backtracking requires recursion
grid_bfs variant_of bfs
sliding_window often_confused_with two_pointers
```

For symmetric relationships such as `related_to` and `often_confused_with`, either insert both directions or make the query layer treat them as bidirectional. Pick one convention and keep it consistent.

---

## 5.5 `app.learning_items`

### Purpose

Universal catalog of things a user can learn, solve, practice, read, review, or be assessed on.

Examples:

- coding problem
- diagnostic question
- quiz question
- checkpoint
- reading
- system design case
- LLD case
- SQL exercise
- debugging task
- mock interview prompt
- project/lab

### Use cases

- Backfill `catalog.coding_problems` into tutor-native items.
- Represent DSA problems, future system design cases, SQL exercises, OS diagnostics, and interview questions uniformly.
- Track item-level user state in `user_learning_item_state`.
- Select practice or verification items for the TutorOrchestrator.
- Store saved creator-generated items if product explicitly supports saving them.

### Schema

```sql
CREATE TABLE app.learning_items (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),

  area_id uuid NOT NULL REFERENCES app.skill_areas(id),
  primary_skill_id uuid,

  item_type text NOT NULL,
  title text NOT NULL,
  slug text,

  difficulty text,
  difficulty_score numeric(5,4),

  verification_type text NOT NULL DEFAULT 'none',

  source_type text NOT NULL DEFAULT 'internal',
  source_ref_json jsonb NOT NULL DEFAULT '{}'::jsonb,

  estimated_minutes integer,

  visibility text NOT NULL DEFAULT 'global',
  created_by_user_id uuid REFERENCES public.users(id),

  is_active boolean NOT NULL DEFAULT true,

  metadata_json jsonb NOT NULL DEFAULT '{}'::jsonb,

  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now(),

  CONSTRAINT learning_items_primary_skill_same_area_fk
    FOREIGN KEY (primary_skill_id, area_id)
    REFERENCES app.skills(id, area_id),

  CONSTRAINT learning_items_item_type_check CHECK (
    item_type IN (
      'concept',
      'coding_problem',
      'system_design_case',
      'low_level_design_case',
      'sql_exercise',
      'debugging_task',
      'quiz_question',
      'diagnostic_question',
      'checkpoint',
      'interview_question',
      'project_lab',
      'reading'
    )
  ),

  CONSTRAINT learning_items_verification_type_check CHECK (
    verification_type IN (
      'none',
      'read',
      'self_explain',
      'quiz',
      'code_acceptance',
      'rubric_eval',
      'mock_interview'
    )
  ),

  CONSTRAINT learning_items_difficulty_check CHECK (
    difficulty IS NULL OR difficulty IN (
      'beginner',
      'easy',
      'medium',
      'hard',
      'advanced',
      'expert'
    )
  ),

  CONSTRAINT learning_items_visibility_check CHECK (
    visibility IN ('global', 'private_user', 'tenant')
  ),

  CONSTRAINT learning_items_difficulty_score_check
    CHECK (difficulty_score IS NULL OR difficulty_score BETWEEN 0 AND 1),

  CONSTRAINT learning_items_estimated_minutes_check
    CHECK (estimated_minutes IS NULL OR estimated_minutes > 0)
);
```

### Important indexes

```sql
CREATE INDEX idx_learning_items_area_type
  ON app.learning_items(area_id, item_type, is_active);

CREATE INDEX idx_learning_items_primary_skill
  ON app.learning_items(primary_skill_id);

CREATE INDEX idx_learning_items_source
  ON app.learning_items(source_type);

CREATE INDEX idx_learning_items_visibility_owner
  ON app.learning_items(visibility, created_by_user_id);

CREATE UNIQUE INDEX uq_learning_items_source_external_id
  ON app.learning_items(source_type, ((source_ref_json->>'external_id')))
  WHERE source_ref_json ? 'external_id';
```

### Example row

```json
{
  "id": "55555555-5555-5555-5555-555555555555",
  "area_id": "11111111-1111-1111-1111-111111111111",
  "primary_skill_id": "99999999-9999-9999-9999-999999999999",
  "item_type": "coding_problem",
  "title": "Number of Islands",
  "slug": "number_of_islands",
  "difficulty": "medium",
  "difficulty_score": 0.6200,
  "verification_type": "code_acceptance",
  "source_type": "leetcode",
  "source_ref_json": {
    "external_id": "200",
    "url": "https://leetcode.com/problems/number-of-islands/"
  },
  "estimated_minutes": 35,
  "visibility": "global",
  "created_by_user_id": null,
  "is_active": true,
  "metadata_json": {
    "platform_tags": ["array", "dfs", "bfs", "matrix"],
    "interview_priority": "high"
  }
}
```

---

## 5.6 `app.learning_item_skills`

### Purpose

Maps learning items to one or more skills.

A single DSA problem often tests multiple skills. A system design case can also test caching, consistency, sharding, rate limiting, and observability. This table makes that explicit.

### Use cases

- Retrieve practice items for a target skill.
- Infer skill evidence from item attempts.
- Identify secondary weaknesses from failed attempts.
- Support cross-domain items later.

### Schema

```sql
CREATE TABLE app.learning_item_skills (
  learning_item_id uuid NOT NULL REFERENCES app.learning_items(id) ON DELETE CASCADE,
  skill_id uuid NOT NULL REFERENCES app.skills(id) ON DELETE CASCADE,

  role text NOT NULL DEFAULT 'secondary',
  relevance_weight numeric(5,4) NOT NULL DEFAULT 1.0000,

  metadata_json jsonb NOT NULL DEFAULT '{}'::jsonb,

  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now(),

  PRIMARY KEY (learning_item_id, skill_id, role),

  CONSTRAINT learning_item_skills_role_check CHECK (
    role IN (
      'primary',
      'secondary',
      'prerequisite',
      'misconception_target',
      'extension'
    )
  ),

  CONSTRAINT learning_item_skills_weight_check
    CHECK (relevance_weight BETWEEN 0 AND 1)
);
```

### Important indexes

```sql
CREATE INDEX idx_learning_item_skills_skill_role
  ON app.learning_item_skills(skill_id, role, relevance_weight DESC);

CREATE INDEX idx_learning_item_skills_item
  ON app.learning_item_skills(learning_item_id);
```

### Example row

```json
{
  "learning_item_id": "55555555-5555-5555-5555-555555555555",
  "skill_id": "99999999-9999-9999-9999-999999999999",
  "role": "primary",
  "relevance_weight": 1.0000,
  "metadata_json": {
    "mapping_reason": "Number of Islands primarily verifies grid traversal."
  }
}
```

Additional mappings for the same item might include:

```json
[
  {
    "skill": "dfs",
    "role": "secondary",
    "relevance_weight": 0.8500
  },
  {
    "skill": "bfs",
    "role": "secondary",
    "relevance_weight": 0.7500
  },
  {
    "skill": "visited_set",
    "role": "prerequisite",
    "relevance_weight": 0.7000
  }
]
```

---

## 5.7 `app.knowledge_sources`

### Purpose

Stores source-level metadata for grounded tutor content.

This does not define the ingestion pipeline. It defines the runtime shape that the tutor can retrieve from.

### Use cases

- Ground explanations in approved internal or external content.
- Organize chunks by source, domain, and primary skill.
- Track source metadata, version, and license.

### Schema

```sql
CREATE TABLE app.knowledge_sources (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),

  source_type text NOT NULL,
  publisher text,
  title text NOT NULL,
  url text,

  area_id uuid REFERENCES app.skill_areas(id),
  primary_skill_id uuid,

  version text,
  license text,

  is_active boolean NOT NULL DEFAULT true,

  metadata_json jsonb NOT NULL DEFAULT '{}'::jsonb,

  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now(),

  CONSTRAINT knowledge_sources_primary_skill_same_area_fk
    FOREIGN KEY (primary_skill_id, area_id)
    REFERENCES app.skills(id, area_id),

  CONSTRAINT knowledge_sources_skill_area_present CHECK (
    primary_skill_id IS NULL OR area_id IS NOT NULL
  ),

  CONSTRAINT knowledge_sources_source_type_check CHECK (
    source_type IN (
      'lesson',
      'article',
      'official_doc',
      'internal_note',
      'interview_corpus',
      'problem_explanation',
      'rubric',
      'transcript'
    )
  )
);
```

### Important indexes

```sql
CREATE INDEX idx_knowledge_sources_area
  ON app.knowledge_sources(area_id, is_active);

CREATE INDEX idx_knowledge_sources_primary_skill
  ON app.knowledge_sources(primary_skill_id, is_active);
```

### Example row

```json
{
  "id": "66666666-6666-6666-6666-666666666666",
  "source_type": "lesson",
  "publisher": "Crackedin Labs",
  "title": "Sliding Window Fundamentals",
  "url": null,
  "area_id": "11111111-1111-1111-1111-111111111111",
  "primary_skill_id": "22222222-2222-2222-2222-222222222222",
  "version": "v1",
  "license": "internal",
  "is_active": true,
  "metadata_json": {
    "audience": "interview_prep",
    "difficulty": "medium"
  }
}
```

---

## 5.8 `app.knowledge_chunks`

### Purpose

Stores retrieval units for explanations, examples, pitfalls, rubrics, questions, answers, and hints.

### Use cases

- Retrieve grounded context for the LLM.
- Fetch skill-specific examples and pitfalls.
- Support hybrid lexical and semantic retrieval.
- Reuse the same retrieval model for DSA and future domains.

### Schema

```sql
CREATE TABLE app.knowledge_chunks (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),

  source_id uuid NOT NULL REFERENCES app.knowledge_sources(id) ON DELETE CASCADE,

  area_id uuid REFERENCES app.skill_areas(id),
  skill_id uuid,

  chunk_type text NOT NULL,
  title text,
  content text NOT NULL,

  token_count integer,

  keywords_tsv tsvector GENERATED ALWAYS AS (
    to_tsvector('english', coalesce(title, '') || ' ' || coalesce(content, ''))
  ) STORED,

  -- Optional later if pgvector is enabled:
  -- embedding vector(1536),

  is_active boolean NOT NULL DEFAULT true,

  metadata_json jsonb NOT NULL DEFAULT '{}'::jsonb,

  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now(),

  CONSTRAINT knowledge_chunks_skill_same_area_fk
    FOREIGN KEY (skill_id, area_id)
    REFERENCES app.skills(id, area_id),

  CONSTRAINT knowledge_chunks_skill_area_present CHECK (
    skill_id IS NULL OR area_id IS NOT NULL
  ),

  CONSTRAINT knowledge_chunks_chunk_type_check CHECK (
    chunk_type IN (
      'explanation',
      'example',
      'tradeoff',
      'pitfall',
      'rubric',
      'question',
      'answer',
      'hint',
      'worked_example'
    )
  ),

  CONSTRAINT knowledge_chunks_token_count_check
    CHECK (token_count IS NULL OR token_count > 0)
);
```

### Important indexes

```sql
CREATE INDEX idx_knowledge_chunks_skill_type
  ON app.knowledge_chunks(skill_id, chunk_type, is_active);

CREATE INDEX idx_knowledge_chunks_area
  ON app.knowledge_chunks(area_id, is_active);

CREATE INDEX idx_knowledge_chunks_fts
  ON app.knowledge_chunks USING gin (keywords_tsv);
```

### Example row

```json
{
  "id": "77777777-7777-7777-7777-777777777777",
  "source_id": "66666666-6666-6666-6666-666666666666",
  "area_id": "11111111-1111-1111-1111-111111111111",
  "skill_id": "22222222-2222-2222-2222-222222222222",
  "chunk_type": "pitfall",
  "title": "Variable window pitfall",
  "content": "A common mistake in variable sliding window problems is expanding the right pointer but forgetting to shrink the left pointer when the constraint is violated.",
  "token_count": 32,
  "is_active": true,
  "metadata_json": {
    "use_when": ["misconception_correction", "guided_practice"]
  }
}
```

---

## 5.9 `app.learning_events`

### Purpose

Append-only ledger of structured learning evidence.

This table records what happened pedagogically. It is the raw source of truth for learning evidence.

### Use cases

- Record that a skill was seen, taught, attempted, struggled with, reviewed, or verified.
- Record hint usage and solution reveals.
- Ingest coding submission evidence.
- Rebuild learner projections by replaying events.
- Audit why a user was marked as verified, struggling, or stale.

### Schema

```sql
CREATE TABLE app.learning_events (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),

  user_id uuid NOT NULL REFERENCES public.users(id),

  session_id uuid REFERENCES app.chat_sessions(id),
  message_id uuid REFERENCES app.chat_messages(id),

  area_id uuid REFERENCES app.skill_areas(id),
  skill_id uuid REFERENCES app.skills(id),
  learning_item_id uuid REFERENCES app.learning_items(id),

  event_type text NOT NULL,
  source text NOT NULL,

  confidence numeric(5,4) NOT NULL DEFAULT 1.0000,
  coverage_delta numeric(6,4) NOT NULL DEFAULT 0,
  readiness_delta numeric(6,4) NOT NULL DEFAULT 0,

  help_level smallint,
  score numeric(6,4),

  metadata_json jsonb NOT NULL DEFAULT '{}'::jsonb,

  idempotency_key text,

  occurred_at timestamptz NOT NULL DEFAULT now(),
  created_at timestamptz NOT NULL DEFAULT now(),

  CONSTRAINT learning_events_has_target CHECK (
    area_id IS NOT NULL OR skill_id IS NOT NULL OR learning_item_id IS NOT NULL
  ),

  CONSTRAINT learning_events_event_type_check CHECK (
    event_type IN (
      'seen',
      'taught',
      'covered',
      'self_reported_known',
      'attempted',
      'struggled',
      'diagnostic_passed',
      'diagnostic_failed',
      'reviewed',
      'verified',
      'checkpoint_passed',
      'checkpoint_failed',
      'coding_submission_accepted',
      'coding_submission_failed',
      'solution_revealed',
      'hint_used',
      'misconception_observed',
      'plan_task_completed',
      'event_corrected',
      'event_invalidated'
    )
  ),

  CONSTRAINT learning_events_source_check CHECK (
    source IN (
      'chat',
      'extractor',
      'coding_submission',
      'diagnostic',
      'checkpoint',
      'prep_plan',
      'manual_admin',
      'import'
    )
  ),

  CONSTRAINT learning_events_confidence_check
    CHECK (confidence BETWEEN 0 AND 1),

  CONSTRAINT learning_events_delta_check CHECK (
    coverage_delta BETWEEN -1 AND 1
    AND readiness_delta BETWEEN -1 AND 1
  ),

  CONSTRAINT learning_events_help_level_check
    CHECK (help_level IS NULL OR help_level BETWEEN 0 AND 5),

  CONSTRAINT learning_events_score_check
    CHECK (score IS NULL OR score BETWEEN 0 AND 1)
);
```

### Important indexes

```sql
CREATE UNIQUE INDEX uq_learning_events_idempotency
  ON app.learning_events(idempotency_key)
  WHERE idempotency_key IS NOT NULL;

CREATE INDEX idx_learning_events_user_skill_time
  ON app.learning_events(user_id, skill_id, occurred_at DESC);

CREATE INDEX idx_learning_events_user_item_time
  ON app.learning_events(user_id, learning_item_id, occurred_at DESC);

CREATE INDEX idx_learning_events_user_time
  ON app.learning_events(user_id, occurred_at DESC);
```

### Example row

```json
{
  "id": "88888888-8888-8888-8888-888888888888",
  "user_id": "aaaaaaaa-0000-0000-0000-000000000000",
  "session_id": "bbbbbbbb-0000-0000-0000-000000000000",
  "message_id": "cccccccc-0000-0000-0000-000000000000",
  "area_id": "11111111-1111-1111-1111-111111111111",
  "skill_id": "22222222-2222-2222-2222-222222222222",
  "learning_item_id": null,
  "event_type": "taught",
  "source": "extractor",
  "confidence": 0.8800,
  "coverage_delta": 0.2500,
  "readiness_delta": 0.0500,
  "help_level": null,
  "score": null,
  "metadata_json": {
    "assistant_explained": "Sliding window with fixed and variable window examples",
    "verification": "not_verified"
  },
  "idempotency_key": "chatmsg-cccccccc-taught-sliding-window",
  "occurred_at": "2026-06-24T17:00:00+05:30"
}
```

### Production rule

`seen`, `taught`, and `self_reported_known` should never directly verify a skill. Verification requires performance evidence such as diagnostics, checkpoints, accepted submissions, or rubric-evaluated practice with acceptable help level.

---

## 5.10 `app.learner_skill_state`

### Purpose

Current user-specific projection for each skill.

This is the fast-read table used by the tutor at runtime.

### Use cases

- Determine whether the user has seen, practiced, struggled with, or verified a skill.
- Check prerequisite readiness before answering.
- Schedule review.
- Detect weak or stale skills.
- Generate StudentSnapshot.

### Schema

```sql
CREATE TABLE app.learner_skill_state (
  user_id uuid NOT NULL REFERENCES public.users(id),
  skill_id uuid NOT NULL REFERENCES app.skills(id),

  status text NOT NULL DEFAULT 'not_started',

  coverage_score numeric(5,4) NOT NULL DEFAULT 0.0000,
  readiness_score numeric(5,4) NOT NULL DEFAULT 0.0000,
  confidence_score numeric(5,4) NOT NULL DEFAULT 0.0000,

  attempt_count integer NOT NULL DEFAULT 0,

  first_seen_at timestamptz,
  last_seen_at timestamptz,
  last_practiced_at timestamptz,
  last_verified_at timestamptz,
  next_review_at timestamptz,

  missing_prerequisites_json jsonb NOT NULL DEFAULT '[]'::jsonb,
  misconception_signals_json jsonb NOT NULL DEFAULT '[]'::jsonb,
  evidence_json jsonb NOT NULL DEFAULT '{}'::jsonb,

  last_event_id uuid REFERENCES app.learning_events(id),
  state_version integer NOT NULL DEFAULT 1,
  projected_at timestamptz NOT NULL DEFAULT now(),

  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now(),

  PRIMARY KEY (user_id, skill_id),

  CONSTRAINT learner_skill_state_status_check CHECK (
    status IN (
      'not_started',
      'seen',
      'taught',
      'attempted',
      'struggled',
      'covered',
      'needs_review',
      'verified',
      'stale'
    )
  ),

  CONSTRAINT learner_skill_state_scores_check CHECK (
    coverage_score BETWEEN 0 AND 1
    AND readiness_score BETWEEN 0 AND 1
    AND confidence_score BETWEEN 0 AND 1
  ),

  CONSTRAINT learner_skill_state_attempt_count_check
    CHECK (attempt_count >= 0)
);
```

### Important indexes

```sql
CREATE INDEX idx_learner_skill_state_user_status
  ON app.learner_skill_state(user_id, status);

CREATE INDEX idx_learner_skill_state_review
  ON app.learner_skill_state(user_id, next_review_at)
  WHERE next_review_at IS NOT NULL;

CREATE INDEX idx_learner_skill_state_skill
  ON app.learner_skill_state(skill_id);
```

### Example row

```json
{
  "user_id": "aaaaaaaa-0000-0000-0000-000000000000",
  "skill_id": "22222222-2222-2222-2222-222222222222",
  "status": "taught",
  "coverage_score": 0.6500,
  "readiness_score": 0.3500,
  "confidence_score": 0.5200,
  "attempt_count": 2,
  "first_seen_at": "2026-06-20T10:00:00+05:30",
  "last_seen_at": "2026-06-24T17:00:00+05:30",
  "last_practiced_at": "2026-06-24T17:10:00+05:30",
  "last_verified_at": null,
  "next_review_at": "2026-06-27T09:00:00+05:30",
  "missing_prerequisites_json": [
    {
      "skill_slug": "two_pointers",
      "readiness_score": 0.25
    }
  ],
  "misconception_signals_json": [
    {
      "type": "window_shrink_missing",
      "confidence": 0.70
    }
  ],
  "evidence_json": {
    "event_count": 5,
    "last_event_type": "taught"
  },
  "state_version": 3,
  "projected_at": "2026-06-24T17:15:00+05:30"
}
```

### Production rule

Do not materialize every skill for every user upfront. Create rows lazily when the user sees, practices, attempts, struggles with, or verifies a skill. If no row exists, treat state as `not_started` with low confidence.

---

## 5.11 `app.user_learning_item_state`

### Purpose

Current user-specific projection for each learning item.

This table separates item-level progress from skill-level readiness.

### Use cases

- Track whether a user has seen, attempted, failed, solved with help, solved independently, or verified an item.
- Distinguish assisted success from independent success.
- Schedule item review.
- Prevent full-solution-assisted attempts from becoming independent verification.

### Schema

```sql
CREATE TABLE app.user_learning_item_state (
  user_id uuid NOT NULL REFERENCES public.users(id),
  learning_item_id uuid NOT NULL REFERENCES app.learning_items(id),

  state text NOT NULL DEFAULT 'not_started',

  best_outcome text,
  best_score numeric(6,4),
  last_score numeric(6,4),

  attempt_count integer NOT NULL DEFAULT 0,
  independent_success_count integer NOT NULL DEFAULT 0,
  assisted_success_count integer NOT NULL DEFAULT 0,
  failure_count integer NOT NULL DEFAULT 0,
  hint_count integer NOT NULL DEFAULT 0,

  first_seen_at timestamptz,
  first_attempted_at timestamptz,
  first_solved_at timestamptz,

  last_attempted_at timestamptz,
  last_solved_at timestamptz,
  last_verified_at timestamptz,
  next_review_at timestamptz,

  last_help_level smallint,

  misconception_signals_json jsonb NOT NULL DEFAULT '[]'::jsonb,
  evidence_json jsonb NOT NULL DEFAULT '{}'::jsonb,

  last_event_id uuid REFERENCES app.learning_events(id),
  state_version integer NOT NULL DEFAULT 1,
  projected_at timestamptz NOT NULL DEFAULT now(),

  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now(),

  PRIMARY KEY (user_id, learning_item_id),

  CONSTRAINT user_learning_item_state_state_check CHECK (
    state IN (
      'not_started',
      'seen',
      'attempted',
      'struggled',
      'solved_with_help',
      'solved_independently',
      'completed',
      'verified',
      'needs_review',
      'stale',
      'skipped'
    )
  ),

  CONSTRAINT user_learning_item_state_help_level_check
    CHECK (last_help_level IS NULL OR last_help_level BETWEEN 0 AND 5),

  CONSTRAINT user_learning_item_state_counts_check CHECK (
    attempt_count >= 0
    AND independent_success_count >= 0
    AND assisted_success_count >= 0
    AND failure_count >= 0
    AND hint_count >= 0
  ),

  CONSTRAINT user_learning_item_state_score_check CHECK (
    (best_score IS NULL OR best_score BETWEEN 0 AND 1)
    AND (last_score IS NULL OR last_score BETWEEN 0 AND 1)
  )
);
```

### Important indexes

```sql
CREATE INDEX idx_user_learning_item_state_user_state
  ON app.user_learning_item_state(user_id, state);

CREATE INDEX idx_user_learning_item_state_review
  ON app.user_learning_item_state(user_id, next_review_at)
  WHERE next_review_at IS NOT NULL;

CREATE INDEX idx_user_learning_item_state_item
  ON app.user_learning_item_state(learning_item_id);
```

### Example row

```json
{
  "user_id": "aaaaaaaa-0000-0000-0000-000000000000",
  "learning_item_id": "55555555-5555-5555-5555-555555555555",
  "state": "solved_with_help",
  "best_outcome": "accepted_after_hint",
  "best_score": 1.0000,
  "last_score": 1.0000,
  "attempt_count": 3,
  "independent_success_count": 0,
  "assisted_success_count": 1,
  "failure_count": 2,
  "hint_count": 2,
  "first_seen_at": "2026-06-21T12:00:00+05:30",
  "first_attempted_at": "2026-06-21T12:10:00+05:30",
  "first_solved_at": "2026-06-24T16:30:00+05:30",
  "last_attempted_at": "2026-06-24T16:30:00+05:30",
  "last_solved_at": "2026-06-24T16:30:00+05:30",
  "last_verified_at": null,
  "next_review_at": "2026-06-29T09:00:00+05:30",
  "last_help_level": 3,
  "misconception_signals_json": [
    {
      "skill_slug": "grid_traversal",
      "signal": "missed_visited_marking"
    }
  ],
  "evidence_json": {
    "latest_submission_verdict": "accepted",
    "verification_blocked_reason": "help_level_3"
  },
  "state_version": 4,
  "projected_at": "2026-06-24T16:35:00+05:30"
}
```

### Production rule

Help level interpretation:

| Help level | Meaning |
|---:|---|
| 0 | Independent, no help |
| 1 | Small nudge |
| 2 | Hint |
| 3 | Scaffolded approach |
| 4 | Partial solution |
| 5 | Full solution or answer-level help |

A success with help level 2-5 should not be counted as fully independent. A success with help level 5 should never become independent verification for that attempt.

---

## 5.12 `app.user_context_events`

### Purpose

Append-only event ledger for durable user context that is not learning mastery.

Examples:

- User prefers concise answers.
- User wants Python examples.
- User is targeting Amazon.
- User can study one hour per day.
- User self-reports weakness in recursion.
- User prefers hints before full solutions.

### Use cases

- Build personalization safely.
- Keep Postgres as the source of truth for durable context.
- Sync approved facts to Mem0/MemZero without making Mem0 canonical.
- Allow future memory deletion and audit.

### Schema

```sql
CREATE TABLE app.user_context_events (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),

  user_id uuid NOT NULL REFERENCES public.users(id),

  session_id uuid REFERENCES app.chat_sessions(id),
  message_id uuid REFERENCES app.chat_messages(id),

  event_type text NOT NULL,
  context_category text NOT NULL,

  memory_text text NOT NULL,
  normalized_key text NOT NULL,
  normalized_value_json jsonb NOT NULL DEFAULT '{}'::jsonb,

  area_id uuid REFERENCES app.skill_areas(id),
  skill_id uuid REFERENCES app.skills(id),

  confidence numeric(5,4) NOT NULL DEFAULT 1.0000,
  source text NOT NULL DEFAULT 'chat',

  sensitivity_level text NOT NULL DEFAULT 'low',
  actionability numeric(5,4) NOT NULL DEFAULT 0.5000,

  valid_from timestamptz NOT NULL DEFAULT now(),
  valid_until timestamptz,
  is_current boolean NOT NULL DEFAULT true,

  supersedes_event_id uuid REFERENCES app.user_context_events(id),
  contradicts_event_id uuid REFERENCES app.user_context_events(id),

  memzero_memory_id text,

  metadata_json jsonb NOT NULL DEFAULT '{}'::jsonb,
  idempotency_key text,

  occurred_at timestamptz NOT NULL DEFAULT now(),
  created_at timestamptz NOT NULL DEFAULT now(),

  CONSTRAINT user_context_events_event_type_check CHECK (
    event_type IN (
      'preference_stated',
      'preference_updated',
      'goal_stated',
      'goal_updated',
      'target_company_stated',
      'style_preference_stated',
      'availability_stated',
      'interest_stated',
      'weakness_self_reported',
      'strength_self_reported',
      'context_observed',
      'context_invalidated',
      'memory_deleted'
    )
  ),

  CONSTRAINT user_context_events_category_check CHECK (
    context_category IN (
      'preference',
      'goal',
      'target',
      'style',
      'company',
      'availability',
      'interest',
      'self_reported_strength',
      'self_reported_weakness',
      'recent_focus',
      'policy'
    )
  ),

  CONSTRAINT user_context_events_source_check CHECK (
    source IN (
      'chat',
      'onboarding',
      'prep_plan',
      'feedback',
      'imported_data',
      'admin',
      'extractor'
    )
  ),

  CONSTRAINT user_context_events_sensitivity_check CHECK (
    sensitivity_level IN ('low', 'medium', 'high', 'restricted')
  ),

  CONSTRAINT user_context_events_scores_check CHECK (
    confidence BETWEEN 0 AND 1
    AND actionability BETWEEN 0 AND 1
  )
);
```

### Important indexes

```sql
CREATE UNIQUE INDEX uq_user_context_events_idempotency
  ON app.user_context_events(idempotency_key)
  WHERE idempotency_key IS NOT NULL;

CREATE INDEX idx_user_context_events_user_key_current
  ON app.user_context_events(user_id, normalized_key, is_current);

CREATE INDEX idx_user_context_events_user_time
  ON app.user_context_events(user_id, occurred_at DESC);

CREATE INDEX idx_user_context_events_memzero
  ON app.user_context_events(memzero_memory_id)
  WHERE memzero_memory_id IS NOT NULL;
```

### Example row

```json
{
  "id": "99999999-9999-9999-9999-999999999999",
  "user_id": "aaaaaaaa-0000-0000-0000-000000000000",
  "session_id": "bbbbbbbb-0000-0000-0000-000000000000",
  "message_id": "cccccccc-0000-0000-0000-000000000000",
  "event_type": "style_preference_stated",
  "context_category": "style",
  "memory_text": "User prefers concise, practical explanations.",
  "normalized_key": "preferred_explanation_style",
  "normalized_value_json": {
    "style": "concise",
    "format": "practical",
    "avoid": ["unnecessary motivation"]
  },
  "area_id": null,
  "skill_id": null,
  "confidence": 0.9500,
  "source": "chat",
  "sensitivity_level": "low",
  "actionability": 0.9000,
  "valid_from": "2026-06-24T17:30:00+05:30",
  "valid_until": null,
  "is_current": true,
  "memzero_memory_id": "mem_123",
  "metadata_json": {
    "usage_mode": "silent"
  }
}
```

---

## 5.13 `app.user_profile_summary`

### Purpose

One fast-read current profile row per user.

This table is a projection from `user_context_events`, selected chat signals, feedback, prep plans, and optional Mem0 sync metadata.

### Use cases

- Provide compact personalization context to TutorOrchestrator.
- Avoid scanning all memories and chat messages every turn.
- Store current target role, level, companies, preferred style, language, recent focus, and learning preferences.
- Keep Mem0 optional and replaceable.

### Schema

```sql
CREATE TABLE app.user_profile_summary (
  user_id uuid PRIMARY KEY REFERENCES public.users(id),

  target_role text,
  target_level text,
  target_companies_json jsonb NOT NULL DEFAULT '[]'::jsonb,

  active_prep_goal text,
  active_prep_plan_id uuid REFERENCES app.user_prep_plans(id),

  preferred_explanation_style text,
  preferred_language text,

  preferred_domains_json jsonb NOT NULL DEFAULT '[]'::jsonb,

  known_strengths_json jsonb NOT NULL DEFAULT '[]'::jsonb,
  known_weaknesses_json jsonb NOT NULL DEFAULT '[]'::jsonb,
  recent_focus_json jsonb NOT NULL DEFAULT '[]'::jsonb,

  learning_preferences_json jsonb NOT NULL DEFAULT '{}'::jsonb,
  schedule_constraints_json jsonb NOT NULL DEFAULT '{}'::jsonb,
  personalization_policy_json jsonb NOT NULL DEFAULT '{}'::jsonb,

  memory_confidence_score numeric(5,4) NOT NULL DEFAULT 0.0000,

  last_context_event_id uuid REFERENCES app.user_context_events(id),
  last_memzero_sync_at timestamptz,

  summary_text text,

  profile_version integer NOT NULL DEFAULT 1,
  projected_at timestamptz NOT NULL DEFAULT now(),

  metadata_json jsonb NOT NULL DEFAULT '{}'::jsonb,

  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now(),

  CONSTRAINT user_profile_summary_memory_confidence_check
    CHECK (memory_confidence_score BETWEEN 0 AND 1)
);
```

### Important indexes

```sql
CREATE INDEX idx_user_profile_summary_target_role
  ON app.user_profile_summary(target_role);

CREATE INDEX idx_user_profile_summary_active_prep_plan
  ON app.user_profile_summary(active_prep_plan_id);
```

### Example row

```json
{
  "user_id": "aaaaaaaa-0000-0000-0000-000000000000",
  "target_role": "backend_engineer",
  "target_level": "sde2",
  "target_companies_json": ["amazon", "google"],
  "active_prep_goal": "Prepare for backend SDE-2 interviews",
  "active_prep_plan_id": null,
  "preferred_explanation_style": "concise_practical",
  "preferred_language": "python",
  "preferred_domains_json": ["dsa", "system_design"],
  "known_strengths_json": [
    {
      "source": "self_report",
      "skill": "sql",
      "confidence": 0.40
    }
  ],
  "known_weaknesses_json": [
    {
      "source": "self_report",
      "skill": "recursion",
      "confidence": 0.50
    }
  ],
  "recent_focus_json": [
    {
      "area": "dsa",
      "skill": "sliding_window",
      "last_seen_at": "2026-06-24T17:00:00+05:30"
    }
  ],
  "learning_preferences_json": {
    "hint_policy": "do_not_reveal_full_solution_first"
  },
  "schedule_constraints_json": {
    "daily_minutes": 60
  },
  "personalization_policy_json": {
    "allow_silent_personalization": true,
    "allow_explicit_personalization": true
  },
  "memory_confidence_score": 0.7200,
  "summary_text": "Backend SDE-2 candidate targeting Amazon/Google, prefers concise Python examples and hint-first DSA guidance.",
  "profile_version": 2,
  "projected_at": "2026-06-24T17:35:00+05:30"
}
```

---

# 6. Deferred company intelligence tables

These should not be part of the initial DSA Skill Graph MVP unless company-specific prep is actively being built. The core schema is already ready for them.

---

## 6.1 `app.companies`

### Purpose

Canonical company list.

### Use cases

- Normalize target companies.
- Attach company-specific frequency signals to skills/items.
- Support aliases such as `AWS` for Amazon.

### Schema

```sql
CREATE TABLE app.companies (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),

  slug text NOT NULL UNIQUE,
  name text NOT NULL,

  aliases_json jsonb NOT NULL DEFAULT '[]'::jsonb,

  is_active boolean NOT NULL DEFAULT true,
  metadata_json jsonb NOT NULL DEFAULT '{}'::jsonb,

  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now(),

  CONSTRAINT companies_slug_format
    CHECK (slug ~ '^[a-z0-9][a-z0-9_]*$')
);
```

### Example row

```json
{
  "id": "11111111-2222-3333-4444-555555555555",
  "slug": "amazon",
  "name": "Amazon",
  "aliases_json": ["AWS"],
  "is_active": true,
  "metadata_json": {
    "interview_style": ["leadership_principles", "dsa", "system_design"]
  }
}
```

---

## 6.2 `app.company_skill_demand`

### Purpose

Stores company-specific demand/frequency signal for skills.

This answers:

```text
Which skills are usually asked by a company for a role, level, stage, and region?
```

### Use cases

- Recommend high-frequency weak topics for a target company.
- Rank DSA topics by company relevance.
- Support future interview intelligence reports.

### Schema

```sql
CREATE TABLE app.company_skill_demand (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),

  company_id uuid NOT NULL REFERENCES app.companies(id) ON DELETE CASCADE,
  skill_id uuid NOT NULL REFERENCES app.skills(id) ON DELETE CASCADE,
  area_id uuid NOT NULL REFERENCES app.skill_areas(id),

  role_family text NOT NULL DEFAULT 'any',
  level text NOT NULL DEFAULT 'any',
  interview_stage text NOT NULL DEFAULT 'any',
  region text NOT NULL DEFAULT 'global',

  frequency_score numeric(5,4) NOT NULL DEFAULT 0.0000,
  recency_score numeric(5,4) NOT NULL DEFAULT 0.0000,
  confidence_score numeric(5,4) NOT NULL DEFAULT 0.0000,

  sample_count integer NOT NULL DEFAULT 0,
  last_observed_at timestamptz,

  source_summary_json jsonb NOT NULL DEFAULT '{}'::jsonb,
  metadata_json jsonb NOT NULL DEFAULT '{}'::jsonb,

  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now(),

  CONSTRAINT company_skill_demand_unique_scope UNIQUE (
    company_id,
    skill_id,
    role_family,
    level,
    interview_stage,
    region
  ),

  CONSTRAINT company_skill_demand_scores_check CHECK (
    frequency_score BETWEEN 0 AND 1
    AND recency_score BETWEEN 0 AND 1
    AND confidence_score BETWEEN 0 AND 1
  ),

  CONSTRAINT company_skill_demand_sample_count_check
    CHECK (sample_count >= 0)
);
```

### Important indexes

```sql
CREATE INDEX idx_company_skill_demand_company_scope
  ON app.company_skill_demand(company_id, role_family, level, interview_stage, region);

CREATE INDEX idx_company_skill_demand_skill
  ON app.company_skill_demand(skill_id);
```

### Example row

```json
{
  "id": "22222222-3333-4444-5555-666666666666",
  "company_id": "11111111-2222-3333-4444-555555555555",
  "skill_id": "22222222-2222-2222-2222-222222222222",
  "area_id": "11111111-1111-1111-1111-111111111111",
  "role_family": "software_engineer",
  "level": "sde1",
  "interview_stage": "online_assessment",
  "region": "global",
  "frequency_score": 0.7800,
  "recency_score": 0.7000,
  "confidence_score": 0.6400,
  "sample_count": 42,
  "last_observed_at": "2026-06-01T00:00:00+05:30",
  "source_summary_json": {
    "sources": ["internal_corpus", "user_reports"]
  },
  "metadata_json": {
    "notes": "Sliding window appears frequently in OA-style array/string problems."
  }
}
```

---

## 6.3 `app.company_learning_item_demand`

### Purpose

Stores company-specific demand/frequency signal for specific learning items.

This answers:

```text
Which specific problems, questions, or cases are usually asked by a company?
```

### Use cases

- Rank practice items by target company.
- Generate company-specific drill sets.
- Identify repeated question variants.

### Schema

```sql
CREATE TABLE app.company_learning_item_demand (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),

  company_id uuid NOT NULL REFERENCES app.companies(id) ON DELETE CASCADE,
  learning_item_id uuid NOT NULL REFERENCES app.learning_items(id) ON DELETE CASCADE,
  area_id uuid NOT NULL REFERENCES app.skill_areas(id),

  role_family text NOT NULL DEFAULT 'any',
  level text NOT NULL DEFAULT 'any',
  interview_stage text NOT NULL DEFAULT 'any',
  region text NOT NULL DEFAULT 'global',

  frequency_score numeric(5,4) NOT NULL DEFAULT 0.0000,
  recency_score numeric(5,4) NOT NULL DEFAULT 0.0000,
  confidence_score numeric(5,4) NOT NULL DEFAULT 0.0000,

  sample_count integer NOT NULL DEFAULT 0,
  last_observed_at timestamptz,

  source_summary_json jsonb NOT NULL DEFAULT '{}'::jsonb,
  metadata_json jsonb NOT NULL DEFAULT '{}'::jsonb,

  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now(),

  CONSTRAINT company_item_demand_unique_scope UNIQUE (
    company_id,
    learning_item_id,
    role_family,
    level,
    interview_stage,
    region
  ),

  CONSTRAINT company_item_demand_scores_check CHECK (
    frequency_score BETWEEN 0 AND 1
    AND recency_score BETWEEN 0 AND 1
    AND confidence_score BETWEEN 0 AND 1
  ),

  CONSTRAINT company_item_demand_sample_count_check
    CHECK (sample_count >= 0)
);
```

### Important indexes

```sql
CREATE INDEX idx_company_item_demand_company_scope
  ON app.company_learning_item_demand(company_id, role_family, level, interview_stage, region);

CREATE INDEX idx_company_item_demand_item
  ON app.company_learning_item_demand(learning_item_id);
```

### Example row

```json
{
  "id": "33333333-4444-5555-6666-777777777777",
  "company_id": "11111111-2222-3333-4444-555555555555",
  "learning_item_id": "55555555-5555-5555-5555-555555555555",
  "area_id": "11111111-1111-1111-1111-111111111111",
  "role_family": "software_engineer",
  "level": "sde1",
  "interview_stage": "online_assessment",
  "region": "global",
  "frequency_score": 0.6200,
  "recency_score": 0.7400,
  "confidence_score": 0.5800,
  "sample_count": 19,
  "last_observed_at": "2026-06-01T00:00:00+05:30",
  "metadata_json": {
    "variant_notes": "Often appears as grid traversal or island-counting variant."
  }
}
```

---

# 7. Optional observability table

---

## 7.1 `app.memory_retrieval_logs`

### Purpose

Optional internal observability table for memory retrieval and filtering.

Do not add this in V1 unless debugging, compliance, or explainability requirements justify it.

### Use cases

- Debug why memory was or was not used.
- Measure Mem0/MemZero retrieval quality.
- Track blocked memories and policy decisions.

### Schema

```sql
CREATE TABLE app.memory_retrieval_logs (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),

  user_id uuid NOT NULL REFERENCES public.users(id),
  session_id uuid REFERENCES app.chat_sessions(id),
  message_id uuid REFERENCES app.chat_messages(id),

  retrieval_query text,
  retrieval_source text NOT NULL,

  candidate_count integer NOT NULL DEFAULT 0,
  used_count integer NOT NULL DEFAULT 0,
  blocked_count integer NOT NULL DEFAULT 0,

  memzero_memory_ids_json jsonb NOT NULL DEFAULT '[]'::jsonb,
  used_memories_json jsonb NOT NULL DEFAULT '[]'::jsonb,
  blocked_reasons_json jsonb NOT NULL DEFAULT '[]'::jsonb,

  latency_ms integer,

  metadata_json jsonb NOT NULL DEFAULT '{}'::jsonb,

  created_at timestamptz NOT NULL DEFAULT now(),

  CONSTRAINT memory_retrieval_logs_source_check CHECK (
    retrieval_source IN ('postgres', 'memzero', 'hybrid')
  ),

  CONSTRAINT memory_retrieval_logs_counts_check CHECK (
    candidate_count >= 0
    AND used_count >= 0
    AND blocked_count >= 0
  ),

  CONSTRAINT memory_retrieval_logs_latency_check
    CHECK (latency_ms IS NULL OR latency_ms >= 0)
);
```

### Important indexes

```sql
CREATE INDEX idx_memory_retrieval_logs_user_time
  ON app.memory_retrieval_logs(user_id, created_at DESC);

CREATE INDEX idx_memory_retrieval_logs_session
  ON app.memory_retrieval_logs(session_id);
```

### Example row

```json
{
  "id": "44444444-5555-6666-7777-888888888888",
  "user_id": "aaaaaaaa-0000-0000-0000-000000000000",
  "session_id": "bbbbbbbb-0000-0000-0000-000000000000",
  "message_id": "cccccccc-0000-0000-0000-000000000000",
  "retrieval_query": "DSA practice recommendation for sliding window",
  "retrieval_source": "hybrid",
  "candidate_count": 8,
  "used_count": 2,
  "blocked_count": 1,
  "memzero_memory_ids_json": ["mem_123", "mem_456"],
  "used_memories_json": [
    {
      "normalized_key": "preferred_language",
      "value": "python"
    }
  ],
  "blocked_reasons_json": [
    {
      "reason": "irrelevant_to_current_domain"
    }
  ],
  "latency_ms": 92
}
```

---

# 8. Source-of-truth boundaries

| Concern | Source of truth |
|---|---|
| Canonical domains | `app.skill_areas` |
| Canonical skills/topics/subtopics | `app.skills` |
| User phrase resolution | `app.skill_aliases` |
| Prerequisites, variants, overlaps | `app.skill_relationships` |
| Problems/questions/cases/readings | `app.learning_items` |
| Item-to-skill mapping | `app.learning_item_skills` |
| Grounded source metadata | `app.knowledge_sources` |
| Grounded retrieval chunks | `app.knowledge_chunks` |
| Raw learning evidence | `app.learning_events` |
| Current user skill state | `app.learner_skill_state` |
| Current user item state | `app.user_learning_item_state` |
| Durable user context/preferences/goals | `app.user_context_events` |
| Fast personalization profile | `app.user_profile_summary` |
| Raw conversation | existing `app.chat_messages` |
| Durable prep roadmap | existing `app.user_prep_plans` |
| Coding submissions | existing `user_data.user_coding_submissions` |
| Semantic memory recall | Mem0/MemZero behind `UserMemoryService`, not canonical |

---

# 9. How this supports DSA first

Initial DSA seed should create:

| Object | Approximate V1 size |
|---|---:|
| `skill_areas` | 1 row: `dsa` |
| Top-level DSA modules in `skills` | 22 to 24 rows |
| DSA canonical subskills | 180 to 220 rows |
| Skill aliases | 400 to 800 rows over time |
| Learning items | Backfilled from `catalog.coding_problems` |
| Item-skill mappings | Usually 2 to 5 per coding problem |

Examples of top-level DSA modules:

```text
problem_solving_fundamentals
complexity_analysis
arrays_hashing
strings
two_pointers
sliding_window
prefix_suffix_range
sorting_ordering
binary_search
linked_lists
stacks_queues_deques
trees_binary_trees
binary_search_trees
tries
heaps_priority_queues
graph_fundamentals
advanced_graphs
recursion_backtracking
dynamic_programming
greedy
intervals_sweep_line
bit_manipulation
math_geometry_combinatorics
design_data_stream_custom_structures
```

The same schema later supports:

```text
operating_systems.processes
operating_systems.threads
operating_systems.deadlocks
computer_networks.tcp_udp
computer_networks.dns
system_design.rate_limiter
system_design.consistent_hashing
low_level_design.factory_pattern
databases.indexes
sql.window_functions
```

No schema change is required for those future areas.

---

# 10. Implementation order

Recommended migration/build order:

1. `app.skill_areas`
2. `app.skills`
3. `app.skill_aliases`
4. `app.skill_relationships`
5. `app.learning_items`
6. `app.learning_item_skills`
7. `app.knowledge_sources`
8. `app.knowledge_chunks`
9. `app.learning_events`
10. `app.learner_skill_state`
11. `app.user_learning_item_state`
12. `app.user_context_events`
13. `app.user_profile_summary`

Then backfill:

```text
catalog.coding_problems
  -> app.learning_items
  -> app.learning_item_skills
```

Then project existing evidence:

```text
user_data.user_coding_submissions
  -> app.learning_events
  -> app.user_learning_item_state
  -> app.learner_skill_state
```

---

# 11. Final recommendation

Use the 13-table production core as the new tutor source-of-truth model.

Do not reduce back to the 7-table prototype kernel. The 7-table kernel is useful for a prototype, but production tutoring needs:

- `skill_aliases` for reliable skill resolution,
- `learning_item_skills` for multi-skill problem mapping,
- `knowledge_sources` and `knowledge_chunks` for grounded retrieval,
- `user_context_events` and `user_profile_summary` for governed personalization.

Do not create domain-specific tables for DSA. DSA should be represented as rows inside the global Skill Graph.

The most important invariant is:

```text
Skill Graph = global curriculum truth.
Learning Events = raw user learning evidence.
Learner State tables = projected current user state.
Mem0/MemZero = optional personalization recall, never mastery truth.
```

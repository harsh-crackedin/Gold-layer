# CrackedIn Gold Layer Design and Implementation Report

**Purpose:** This report explains how to implement the Gold layer as a product-facing derived layer over the current Postgres Silver corpus.

**Important constraint:** This report intentionally avoids SQL schema blocks. It focuses on design, implementation flow, scoring, retrieval, batching, and product behavior.

---

## 1. Executive summary

The Gold layer should turn raw, verbatim Silver interview data into fast, canonical, product-facing intelligence.

Silver answers:

- What did the candidate report?
- Which company and role were mentioned?
- Which round happened?
- What exact prompt was asked?
- What signals, warnings, or advice were captured?

Gold answers:

- What canonical question does this raw prompt map to?
- How often is this question being asked by company and role?
- Which questions are currently trending?
- What does the interview loop look like for this company and role?
- Which DSA prompts map to LeetCode, Codeforces, or other platforms?
- Which custom company-authored prompts are recurring enough to show?
- What supporting evidence can we show to users?

The core product table should be `gold.question_trends`. It should serve the UI/API for recently asked and trending questions by company, role, item type, and time window.

---

## 2. Current source-of-truth architecture

The latest master-readable architecture is Postgres-oriented and medallion-style:

- Bronze lives in `ingest`.
- Silver lives in `silver`.
- Gold is planned but not yet implemented.

Bronze tracks ingestion lifecycle and object-storage pointers. It does not hold full interview content.

Silver stores normalized interview corpus data:

- one interview experience,
- many rounds,
- many assessment item occurrences,
- many atomic signals,
- searchable text chunks,
- verified evidence spans.

Silver intentionally avoids canonicalization. That is the key boundary. It stores reported occurrences verbatim. Gold owns canonical question resolution, final role mapping, frequency aggregation, trend scoring, and product serving.

---

## 3. Design goals

The Gold layer should satisfy six goals.

### 3.1 Product-facing

Gold should be optimized for application queries, not for raw storage. Product APIs should avoid scanning Silver repeatedly for common pages like:

- Google SDE2 DSA trends,
- Amazon SDE3 system design questions,
- Meta E5 interview loop,
- recently asked frontend machine-coding prompts.

### 3.2 Rebuildable

Gold should be rebuildable from Silver. If scoring changes, canonicalization thresholds change, or role-level mapping improves, the system should be able to re-run Gold jobs without re-crawling or re-running the extraction LLM.

### 3.3 Auditable

Every visible trend should trace back to supporting Silver occurrences and interviews. This is important because the product moat is trust: “these were actually reported in interview experiences.”

### 3.4 Batch-first

Public aggregate data should update in batches. This gives better quality control, makes duplicate detection safer, and avoids showing unstable canonicalization results immediately.

### 3.5 Immediate-capable

Even though public aggregation should be batch-first, the design should support near-real-time deterministic updates later. Immediate updates should be limited to safe, deterministic mappings such as exact prompt hash and explicit platform references.

### 3.6 Hybrid retrieval ready

Gold should support both structured SQL queries and semantic retrieval. SQL should handle company/role/frequency questions. Vector/full-text retrieval should handle vague, evidence-seeking, or semantically phrased questions.

---

## 4. Recommended Gold tables

The Gold schema should use normal, readable table names. Avoid names like `gold_layer_*`. The schema name already communicates that this is the Gold layer.

Recommended tables:

- `gold.role_levels`
- `gold.interview_roles`
- `gold.questions`
- `gold.question_references`
- `gold.question_occurrences`
- `gold.question_trends`
- `gold.interview_profiles`
- `gold.search_documents`
- `gold.refresh_runs`

### 4.1 `gold.role_levels`

Owns product-facing company/role/level mapping.

This table maps native labels like Google L4, Amazon SDE2, Meta E5, Microsoft L61, or Uber 5a into a neutral canonical seniority ladder. It should include confidence and review state because level equivalence is opinionated.

### 4.2 `gold.interview_roles`

Stores the final cohort assignment for each interview.

This table makes company/role queries fast and consistent. Instead of recalculating role mapping from raw Silver fields on every trend query, the Gold refresh process resolves each interview once and stores the result.

### 4.3 `gold.questions`

Stores canonical question concepts.

Examples:

- Number of Islands,
- Design a URL Shortener,
- Parking Lot LLD,
- Amazon Leadership Principle ownership story,
- Top-N SQL query,
- Custom company-specific distributed systems prompt.

This table is platform-agnostic. A canonical question may have LeetCode references, Codeforces references, or no external platform reference at all.

### 4.4 `gold.question_references`

Stores external practice references.

This table maps canonical questions to LeetCode, Codeforces, HackerRank, company-named prompts, URLs, or other platforms. This separation is important because one canonical question can have many references or variants.

### 4.5 `gold.question_occurrences`

Maps each Silver assessment occurrence to a canonical Gold question.

This is the audit fact table. It tells us exactly which raw prompt from which interview supports a canonical question.

### 4.6 `gold.question_trends`

Serves product-facing trending question lists.

This table should store counts, weighted frequency, relevance, trend score, last seen date, sample interviews, and scope information. It should power most of the user-facing “recently asked” and “trending questions” APIs.

### 4.7 `gold.interview_profiles`

Serves interview-loop intelligence.

This table summarizes common round patterns, round mix, item type mix, process duration, signals, top topics, and role-specific interview expectations.

### 4.8 `gold.search_documents`

Serves hybrid SQL, full-text, and vector retrieval.

This table stores searchable text derived from questions, occurrences, chunks, signals, and evidence. It should include company/role filters so vector search can be constrained by structured product dimensions.

### 4.9 `gold.refresh_runs`

Tracks Gold refresh jobs.

This is an operational/debug table. It helps answer: Did the latest batch run? How many rows changed? Did trend scores fail to update?

---

## 5. Canonicalization strategy

Canonicalization is the hardest part of Gold. It should use a laddered resolver instead of relying on a single model call.

### 5.1 Tier 1 — deterministic matching

Use deterministic matching first because it is cheap, explainable, and safe.

Examples:

- same prompt hash already mapped,
- explicit LeetCode number,
- explicit Codeforces reference,
- known platform URL,
- exact normalized title match,
- known company prompt alias.

If Tier 1 succeeds with high confidence, map automatically.

### 5.2 Tier 2 — embedding candidate generation

For unresolved prompts, generate candidate matches using embeddings.

This step should retrieve possible canonical questions. It should not make the final decision by itself. Embedding similarity is useful for paraphrases but can over-merge similar-looking prompts.

Example:

- “Design Bitly” and “Design a URL shortener” should be candidates.
- “Design Twitter feed” and “Design news feed ranking” should be candidates but need careful scoring.

### 5.3 Tier 2.5 — reranking and hybrid scoring

After embedding retrieval, rerank candidates using a hybrid score:

- semantic similarity,
- lexical/title overlap,
- topic overlap,
- pattern overlap,
- external reference strength,
- item type match.

This protects against common mistakes like mapping two graph problems together just because both mention BFS/DFS.

### 5.4 Tier 3 — LLM decision for ambiguous cases

Use the LLM only in the ambiguous middle band. Do not use it for every occurrence. The LLM should answer a narrow question: are these two prompts the same canonical problem/concept, a variant, or different?

### 5.5 Routing

Every occurrence should land in one of three states:

- auto-linked,
- needs review,
- deferred.

Deferred does not mean discarded. It means the prompt is unresolved for now and may later become a recurring custom company-authored question.

### 5.6 Custom question promotion

When several deferred prompts cluster together across distinct interviews, promote the cluster to a custom canonical question.

This is important for the product moat. Company-authored or custom interview prompts may not exist on LeetCode, but if they recur across real interview experiences, they are highly valuable.

---

## 6. Role and level mapping strategy

Gold should own the final role-level mapping.

Silver already stores raw role fields and some resolved level signals, but product-facing cohort logic needs a clean derived layer.

Recommended flow:

1. Normalize company through `silver.company`.
2. Normalize role family from Silver role fields.
3. Resolve native level labels into `gold.role_levels`.
4. Assign each interview to a final company, role family, and canonical rank in `gold.interview_roles`.
5. Mark low-confidence or ambiguous mappings for review.

Examples:

- Amazon SDE2 maps to Mid.
- Google L4 maps to Mid.
- Meta E5 maps to Senior.
- Ambiguous role strings like “Software Engineer” without level may map to unknown or approximate, depending on evidence.

Do not silently over-map vague levels. Wrong cohort assignment will poison trends.

---

## 7. Trend scoring design

Trending should not be lifetime frequency. Older questions should decay or be excluded depending on the product mode.

Use three concepts:

- frequency,
- recency,
- relevance.

### 7.1 Frequency

Frequency should count distinct interviews, not raw repeated occurrences. If the same interview mentions the same question three times, it should not triple the public count.

### 7.2 Recency

Trending should use hard time windows:

- 90 days for hot/current signals,
- 180 days for current interview season,
- 365 days for stable market signal,
- 540 days only as sparse fallback.

Questions outside the selected window should not contribute to that trend row.

### 7.3 Relevance

Exact company-role matches should get the highest relevance. Broader fallback scopes should contribute lower relevance.

Recommended fallback order:

1. exact company + role family + canonical level,
2. same company + role family, any level,
3. same company, any role,
4. same role family + canonical level, all companies,
5. same role family, all companies,
6. global.

This keeps company-specific signal as long as possible before falling back globally.

### 7.4 Quality weighting

Quality should affect trend scoring. Gold should downweight or exclude:

- LLM-rejected interviews,
- duplicate or suspected duplicate interviews,
- authenticity-suspect posts,
- estimated interview dates for public trending,
- low extraction-confidence rows,
- unresolved or review-pending question mappings.

For public product pages, prefer precision over recall.

---

## 8. Batch aggregation design

The public Gold layer should update in batches.

Batch flow:

1. Silver extraction completes.
2. Silver level resolution completes.
3. Gold role mapping runs.
4. Gold question resolver runs.
5. Gold occurrence mapping runs.
6. Gold trend aggregation runs.
7. Gold interview profile aggregation runs.
8. Gold search documents and embeddings update.
9. Refresh metadata is written.

This aligns with the existing repository style: idempotent stages, retryable jobs, and orchestration around stage transitions.

### Why batch-first is better

Batch-first aggregation gives:

- stable public results,
- easier duplicate detection,
- lower risk of publishing bad canonical mappings,
- better control over expensive embedding/LLM calls,
- cleaner operational debugging.

### Suggested cadence

For early stage:

- run batch refresh every 6 to 12 hours,
- run full refresh initially,
- switch to incremental refresh after data volume or runtime demands it.

For mature stage:

- incremental Gold refresh after each Silver batch,
- daily full reconciliation job,
- weekly canonicalization audit.

---

## 9. Immediate aggregation option

Immediate aggregation can be added, but it should be conservative.

Do not run full semantic canonicalization synchronously after a user submits an experience. Immediate updates should only use deterministic logic:

- exact prompt hash match,
- explicit platform reference,
- known external URL,
- already-known canonical alias.

If a prompt is ambiguous, mark it as deferred or review-pending and wait for the batch resolver.

Recommended model:

- public pages use latest completed batch,
- internal/admin views can show batch plus pending overlay,
- deterministic immediate mappings can update low-risk trend rows,
- ambiguous/LLM/embedding-based mappings wait for batch.

This gives freshness without corrupting public trend quality.

---

## 10. SQL, vector, and hybrid retrieval design

Do not start with an LLM router for every query. Use deterministic routing first.

### SQL-only route

Use SQL when the user asks a structured question:

- trending questions by company,
- recently asked DSA questions,
- company/role interview loop,
- most common system design prompts,
- top topics by role.

These should hit `gold.question_trends` and `gold.interview_profiles` directly.

### Vector-only route

Use vector search when the user provides a vague semantic query without clear structured filters.

Examples:

- “Find experiences similar to this prompt.”
- “Show cases where candidates failed due to ambiguity.”
- “Find system design prompts involving queues and consistency.”

### Hybrid route

Use hybrid retrieval when both structure and semantics matter.

Examples:

- “For Google SDE2, what graph variants were asked recently?”
- “Show evidence for why URL shortener is trending at backend roles.”
- “Find Meta phone screen failures related to time management.”

Hybrid flow:

1. SQL prefilters by company, role, level, item type, and time window.
2. Full-text and vector search run only inside that candidate set.
3. Final ranking blends structured trend score, text match, and semantic similarity.
4. The answer includes provenance from occurrences, interviews, and evidence spans.

### Why pgvector first

Use pgvector inside Postgres first. The product needs joins across company, role, question, date, and quality metadata. Keeping vectors in Postgres keeps retrieval simpler and avoids early operational overhead.

Consider a separate vector database only when vector volume or latency becomes a clear bottleneck.

---

## 11. Product API behavior

The Gold API should expose product-ready results rather than raw database rows.

Recommended endpoints:

- trending questions,
- company-specific questions,
- question detail,
- interview profile,
- hybrid search.

### Trending questions response should include

- requested scope,
- actual scope used,
- fallback reason if any,
- time window,
- canonical question title,
- item type,
- platform references,
- distinct interview count,
- weighted frequency,
- trend score,
- last seen date,
- sample evidence availability.

### Interview profile response should include

- company and role scope,
- total supporting interviews,
- round mix,
- common round order,
- common item types,
- top questions by category,
- common signals and warnings,
- confidence/fallback notes.

---

## 12. Public eligibility rules

Not every Silver row should contribute to public Gold trends.

Public trend rows should include only interviews and occurrences that pass quality gates.

Recommended rules:

- exclude LLM-rejected interviews,
- exclude unresolved question mappings,
- exclude review-pending mappings from public trend scores,
- exclude suspected duplicate/authenticity-suspect rows,
- exclude estimated interview dates for public “trending” views,
- require minimum distinct interview support,
- show fallback scope when exact cohort is sparse.

Suggested thresholds:

- exact company-role trends: show internally at 2 supporting interviews, public at 3,
- company-wide trends: public at 3,
- global trends: public at 5,
- custom question promotion: 3 distinct interviews, or 2 distinct interviews when prompt hash/external evidence is very strong.

The main risk is false precision. If a cohort has only one interview, the product should say so clearly or fall back.

---

## 13. Implementation phases

### Phase 1 — Gold migration

Create the Gold schema and proposed tables.

Expected result:

- tables exist,
- indexes are planned,
- refresh metadata table is ready,
- no product API yet.

### Phase 2 — Role mapping

Build `gold.role_levels` and `gold.interview_roles`.

Expected result:

- each public-eligible interview has a resolved company/role/level cohort,
- ambiguous mappings are reviewable,
- product queries have a consistent cohort layer.

### Phase 3 — Deterministic question resolver

Resolve easy cases first:

- prompt hash reuse,
- platform URLs,
- LeetCode numbers,
- known Codeforces references,
- exact known titles.

Expected result:

- large portion of DSA and known prompts mapped with no model cost,
- unresolved cases routed to defer/review.

### Phase 4 — Semantic resolver

Add embeddings, hybrid scoring, reranking, and LLM match decisions for ambiguous cases.

Expected result:

- paraphrased system design, LLD, machine coding, and custom prompts resolve better,
- recurring novel prompts can become custom canonical questions.

### Phase 5 — Trend aggregation

Build trend windows and fallback scopes.

Expected result:

- `gold.question_trends` serves company/role-wise trends quickly,
- public UI can show recently asked and trending questions.

### Phase 6 — Interview profiles

Aggregate round mix, item type mix, ordered process patterns, and signals.

Expected result:

- product can answer interview-loop questions, not just question-list questions.

### Phase 7 — Search documents and retrieval

Populate searchable Gold documents and embeddings.

Expected result:

- SQL-only, vector-only, and hybrid retrieval routes are available.

### Phase 8 — API and UI integration

Expose product endpoints and connect UI surfaces.

Expected result:

- company pages,
- role pages,
- question pages,
- search/RAG-style intelligence responses.

---

## 14. Operational considerations

### 14.1 Idempotency

Gold jobs should be safe to re-run. A failed job should not leave partially corrupted product data.

Use refresh metadata to track successful runs and source watermarks.

### 14.2 Rebuild strategy

Gold should support:

- full rebuild from Silver,
- incremental refresh for new/updated Silver rows,
- selective recompute for one company or time window,
- re-resolution when canonicalization thresholds change.

### 14.3 Review queues

Ambiguous canonicalization should not block the whole system. It should produce review tasks.

Review priority should be based on:

- high-frequency unresolved prompts,
- top companies,
- recent interviews,
- user-facing traffic demand,
- high score but below auto-link threshold.

### 14.4 Monitoring

Track:

- number of unresolved occurrences,
- number of review-pending mappings,
- number of custom questions promoted,
- trend refresh duration,
- embedding backlog,
- public trend rows by window,
- sparse cohort fallback rate.

A high fallback rate means either the data is sparse or role/company mapping is too strict.

---

## 15. Risks and mitigations

### Risk: over-merging different questions

Mitigation: Use deterministic references first, use hybrid scoring, require item type match, and route ambiguous cases to review.

### Risk: under-merging same question variants

Mitigation: Use embeddings, reranking, synonym dictionaries, and periodic deferred-cluster promotion.

### Risk: old popular questions dominate trends

Mitigation: Use hard time windows for trending and do not use lifetime count for trend ranking.

### Risk: sparse company-role cohorts

Mitigation: Use transparent fallback order and show `scope_used` in API responses.

### Risk: wrong role mapping poisons trends

Mitigation: Put role mapping in Gold with confidence, review state, and explicit evidence. Do not silently map vague levels.

### Risk: vector retrieval ignores structured filters

Mitigation: SQL prefilter first, vector search second. Do not run global vector search when company/role/date filters are known.

### Risk: immediate aggregation publishes bad data

Mitigation: Keep public pages batch-first. Immediate mode should only update deterministic matches.

---

## 16. Final recommendation

Build Gold as a compact product-facing schema with nine tables:

- role mapping,
- interview cohort assignment,
- canonical questions,
- external references,
- occurrence mapping,
- trend rankings,
- interview profiles,
- search documents,
- refresh metadata.

Make `gold.question_trends` the primary serving table and `gold.question_occurrences` the audit base.

Use batch aggregation as the public source of truth. Add immediate deterministic updates only after the batch path is reliable.

Use Postgres + pgvector first. Add a separate vector database only if scale forces it.

The strongest product outcome is not just “top questions.” It is evidence-backed, company-role-specific interview intelligence:

- what gets asked,
- how recently,
- how often,
- at which role level,
- in which round,
- with which supporting interviews,
- and what the user should practice next.

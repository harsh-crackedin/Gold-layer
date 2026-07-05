# CrackedIn Gold Layer — Complete Design and Decision Report

**Document type:** Product + architecture design report  
**Scope:** Complete future-ready gold layer design  
**Constraint:** No implementation code or schema code blocks  
**Project:** CrackedIn / InterviewPrep.ai  

---

## 1. Executive Summary

The gold layer should be the product-facing intelligence layer of CrackedIn. It should convert raw and normalized interview data into trusted, recent, explainable, and actionable interview-preparation intelligence.

The current design is directionally correct, but the complete version should go beyond a simple “top company questions” table. The stronger vision is:

**Gold should answer what to prepare, why to prepare it, how confident we are, what changed recently, and how it applies to a specific user.**

That means the gold layer should power:

- Company-wise recently asked questions
- Role-wise and level-wise trending questions
- DSA questions mapped to LeetCode, Codeforces, HackerRank, or custom company variants
- System design, low-level design, machine coding, SQL, behavioral, and domain-technical trends
- Question families and variants
- Topic trend intelligence
- Interview loop prediction
- Evidence-backed question cards
- Failure reason intelligence
- Confidence and fallback labels
- Hybrid SQL/vector retrieval
- Personalized readiness gap analysis
- Company watchlists and trend-change notifications
- Mock interview generation based on real market patterns

The core design should remain simple in principle: silver stores reported facts; gold resolves and aggregates those facts into product intelligence.

---

## 2. Problem We Are Solving

Interview preparation is usually built around stale and generic material:

- Static LeetCode sheets
- Old company-tagged lists
- Unverified interview experiences
- Generic system design lists
- One-size-fits-all preparation plans
- Weak personalization
- No evidence trail
- No freshness or trend detection

CrackedIn has a stronger opportunity because it collects and normalizes real interview experiences. The gold layer should turn that corpus into a preparation engine.

The real user problem is not only:

“What questions are asked at Google?”

The deeper user problem is:

“I am targeting Google Mid Backend in two months. What should I study first, what is actually being asked recently, what round types should I expect, what are my weak areas, and why should I trust these recommendations?”

The gold layer exists to answer that deeper question.

---

## 3. Current Architecture Understanding

CrackedIn has a layered architecture.

### Bronze Layer

The bronze layer tracks ingestion state. It discovers raw items from sources such as LeetCode, Glassdoor, Reddit, curated markdown, or future platforms. It does not store normalized interview facts directly. It tracks lifecycle state, retries, S3 raw payload keys, extracted payload keys, content hashes, and crawl cursors.

Bronze answers operational questions such as:

- Was this raw item discovered?
- Was it hydrated?
- Was it extracted?
- Was it rejected as non-interview content?
- Did it fail after retries?
- Where is the raw/extracted artifact stored?

Bronze is not product-facing.

### Silver Layer

Silver is the normalized, queryable corpus. It stores extracted interview facts verbatim and structurally.

It stores:

- One interview experience per row
- One or more rounds per interview
- One or more assessment items/questions per interview
- Atomic signals such as failure reasons, prep advice, process events, red flags, interviewer behavior, and candidate mistakes
- Searchable chunks
- Verified evidence spans
- Company and level dimensions

Silver is the source of truth for extracted interview facts, but it is not the ideal product API layer. It stores raw reported occurrences, not final canonical question intelligence.

### Existing App/User Layer

The app/user layer stores:

- Users
- Chat sessions and messages
- User memory
- Profile summaries
- Prep targets
- Learning progress
- Checkpoint attempts
- Readiness snapshots
- LeetCode/coding profiles
- Coding submissions
- Code blobs
- Sync state

This layer tells us what a specific user has done and what they are targeting.

### Gold Layer

Gold should sit above silver and existing user data. It resolves, canonicalizes, aggregates, scores, and prepares data for product use.

Gold should not store raw source truth. Instead, it should store derived intelligence.

---

## 4. Core Design Philosophy

### Principle 1 — Silver reports, gold interprets

Silver says: “This interview report mentioned this raw prompt.”

Gold says: “This raw prompt maps to the canonical question Number of Islands, belongs to the grid traversal family, was recently asked at Google Mid Backend, appears mostly in phone screens, has common follow-ups, and is relevant for users weak in graph BFS/DFS.”

### Principle 2 — Product queries should not depend on raw silver joins

A product page should not repeatedly join raw interviews, rounds, items, signals, evidence, and user data just to show top questions. Those joins should happen in refresh jobs. Product APIs should read from precomputed gold tables.

### Principle 3 — Trending means recent

A question asked heavily in 2020 but not recently should not rank high as a current trend. Gold should separate lifetime support from recent trend score.

### Principle 4 — Confidence must be visible

Interview data is noisy. The product should clearly distinguish strong signals from weak signals, exact scopes from fallback scopes, and verified mappings from unresolved/deferred mappings.

### Principle 5 — Personalization should be derived, not baked into global gold

Global company-role intelligence belongs in gold. User-specific readiness and recommendations should consume gold and live in user-facing app tables.

### Principle 6 — SQL first, vector second, LLM last

Most product questions are structured. Company, role, level, item type, and time window are SQL filters. Vector search should help with semantic matching and evidence search. LLMs should synthesize and resolve ambiguity, not act as the default database router for every request.

---

## 5. What the Gold Layer Should Answer

A complete gold layer should answer multiple classes of product questions.

### Question Intelligence

- What questions are being asked recently at a company?
- What DSA questions are trending for a role and level?
- Which system design prompts are common for senior backend roles?
- Which questions are exact LeetCode problems and which are custom variants?
- Which follow-ups are commonly attached to a question?

### Topic Intelligence

- Which topics are rising for Google Mid Backend?
- Is graph more important than DP for this target?
- Are SQL and database questions appearing more frequently for data roles?
- Which behavioral themes are common for Amazon?

### Interview Loop Intelligence

- How many rounds should the user expect?
- Is there usually an OA?
- Does this target include system design?
- How common is behavioral?
- Which round types usually come first?
- Does the role include machine coding or low-level design?

### Failure and Success Intelligence

- Why do candidates fail this company/round?
- What mistakes are commonly reported?
- What prep advice appears repeatedly?
- Are candidates failing because of code completion, edge cases, communication, or system design depth?

### Evidence and Trust

- Why is this question recommended?
- How recent is the evidence?
- How many distinct interviews support it?
- Was this from exact company-role data or fallback data?
- Which sample prompts support this insight?

### Personalization

- What should this user study next?
- What is the gap between the user’s readiness and the target company’s demand?
- Which questions are trending but not yet practiced by the user?
- Which topics can the user safely deprioritize?
- Which company is the user most ready for?

### Change Detection

- What newly appeared in the last 90 days?
- Which questions are rising?
- Which topics are declining?
- What changed since the last batch refresh?
- Should a user watching a company be notified?

---

## 6. Recommended Gold Capabilities

## 6.1 Evidence-Backed Question Cards

This should be one of the first major product capabilities.

A question card should not only say:

“Number of Islands — asked 8 times.”

It should explain:

- The canonical question title
- Platform references such as LeetCode or Codeforces
- Company and role scope
- Recent support count
- Last seen date
- Trend score
- Round types where it appears
- Common follow-ups
- Related question family
- Confidence label
- Whether fallback was used
- Sample anonymized prompts or evidence snippets
- Why it is recommended

This turns a list into a trusted intelligence product.

### Value

Evidence-backed cards differentiate CrackedIn from static question lists. They make the user trust that a recommendation is grounded in real interview reports.

### Necessity

Strongly recommended for the first complete product design. Without this, gold becomes a ranking engine but not a trust engine.

---

## 6.2 Question Families and Variant Intelligence

Raw interview prompts often refer to the same concept in different ways.

Examples:

| Family | Variants |
|---|---|
| Grid traversal / islands | Number of Islands, Max Area of Island, dynamic islands, shortest bridge |
| URL shortener | TinyURL, Bitly, short-link service with analytics, global shortener |
| Parking lot LLD | Vehicle classes, pricing strategy, multiple gates, ticketing, payment |
| Rate limiter | Token bucket, sliding window, distributed rate limiter, API gateway limiter |

Gold should group exact questions into families. This prevents fragmented rankings and helps users study concepts rather than memorize one title.

### Value

Question families answer:

- What pattern is this question testing?
- What variants should I prepare?
- Is this exact or modified?
- Which variants are rising recently?

### Necessity

Necessary for high-quality DSA, system design, LLD, and machine-coding intelligence.

---

## 6.3 Topic Trend Intelligence

Question-level trends are useful, but topic-level trends are often more actionable.

A user with limited time may not need 100 questions. They need topic priority.

Example output:

| Target | Topic Trend Insight |
|---|---|
| Google Mid Backend | Graph/grid traversal rising, hashmap stable, DP moderate |
| Amazon SDE2 | Behavioral ownership and conflict very frequent, arrays/hashmap common |
| Frontend roles | JavaScript internals, async, component design, browser APIs rising |
| Data roles | SQL, statistics, ML theory, data pipelines frequent |

### Value

Topic trends help generate study plans, readiness gaps, and company pages.

### Necessity

Should be part of the complete gold design, even if launched after question trends.

---

## 6.4 Interview Loop Predictor

Users need to know the expected structure of the interview, not just questions.

An interview loop profile should answer:

- How many rounds are common?
- Is there an online assessment?
- Does system design appear at this level?
- Are behavioral rounds frequent?
- Is machine coding common?
- What round order is typical?
- What evaluation areas are common?

Example product summary:

Amazon Mid Backend commonly includes an online assessment, at least one DSA round, one design or machine-coding round, and a behavioral/leadership-principle-heavy round. System design appears more often as the level increases.

### Value

This is highly valuable because candidates prepare differently for OA, phone screen, onsite, system design, and behavioral rounds.

### Necessity

Should be part of the complete product-facing gold layer.

---

## 6.5 Failure Reason Intelligence

Gold should aggregate failure reasons and candidate mistakes from silver signals.

Examples:

| Company/Round | Common Failure Reasons |
|---|---|
| Meta phone screen | Did not finish code, poor time management, weak edge cases |
| Amazon behavioral | Weak ownership examples, vague impact, poor conflict story |
| System design rounds | Shallow scaling discussion, weak tradeoffs, ignored bottlenecks |
| OA rounds | Time pressure, hidden edge cases, TLE |

### Value

This helps users avoid mistakes, not just solve questions.

### Necessity

Very high premium value. It can be added after question and topic trends, but the schema should support it from the start.

---

## 6.6 Personalized Preparation Gap Analysis

The gold layer represents market demand. User progress represents user supply.

The most valuable personalization comes from comparing the two.

Example:

Gold says Google Mid Backend frequently asks graph/grid traversal, system design basics, and behavioral ambiguity handling. User data says the user has weak graph performance, no system design practice, and strong array/hashmap history. The product recommends graph BFS/DFS, Number of Islands family, and a rate-limiter design before more array practice.

### Value

This makes the product feel personally useful instead of generic.

### Necessity

Not required for the first gold aggregation job, but essential for the long-term product moat.

---

## 6.7 Readiness Score by Target

A target readiness score should combine gold demand with user preparation state.

Example:

| Component | Score |
|---|---|
| DSA | 72% |
| System Design | 38% |
| Behavioral | 55% |
| SQL/DB | 80% |
| Overall | 64% |

The score must be explainable. A user should see why their score is low and what would improve it.

### Value

This creates retention and a clear feedback loop.

### Necessity

Not required for gold infrastructure itself, but strongly recommended for product differentiation.

---

## 6.8 Change Detection and Emerging Trends

Gold should compare current and previous windows.

It should detect:

- Newly emerging questions
- Rising topics
- Declining questions
- Resurfacing old patterns
- New failure reasons
- Company process changes

Example:

In the last 90 days, agentic coding and debugging tasks have started appearing more often for startup fullstack roles.

### Value

This creates a freshness moat. Static prep sheets cannot compete with change detection.

### Necessity

Important for premium positioning, watchlists, and marketing pages.

---

## 6.9 Company and Role Comparison

Users often compare companies or decide where to apply first.

Gold should answer:

- How does Google SDE2 differ from Amazon SDE2?
- Which company is more graph-heavy?
- Which company asks more behavioral?
- Which company has more system design?
- Which roles are more OA-heavy?

### Value

This supports company pages, SEO, and application strategy.

### Necessity

Useful but can come after base trends and profiles.

---

## 6.10 Hybrid Interview Evidence Search

Some user questions cannot be answered by aggregated tables alone.

Examples:

- Show recent Google system design interviews where caching was discussed.
- Find Meta interviews where candidates failed because of time management.
- Show examples similar to this DSA prompt.
- What do candidates say about Amazon behavioral rounds?

These require hybrid retrieval.

The correct approach is:

| Query component | Retrieval method |
|---|---|
| Company, role, level, date, item type | SQL filters |
| Similar wording, themes, narratives, failure examples | Full-text and vector search |
| Final explanation | LLM synthesis |

### Value

This turns the corpus into a searchable intelligence engine.

### Necessity

Important for AI chat and research mode, but not required for the first question ranking UI.

---

## 6.11 Mock Interview Generation from Gold

Gold can generate realistic mock interviews based on target company, role, level, and recent trends.

Example:

For Google Mid Backend, generate a mock with one behavioral opener, one graph/grid DSA question, one optimization follow-up, and one system design mini prompt. Evaluation rubrics should reflect recent signals and common failure modes.

### Value

Generic mocks become company-specific and trend-aware.

### Necessity

Later-stage premium feature.

---

## 6.12 Target Company Watchlists

Users should be able to watch companies and roles.

The system can notify them when:

- A new question emerges
- A topic rises sharply
- A system design prompt resurfaces
- A trend matches their weak area
- A watched role changes its round mix

### Value

Creates retention and recurring usage.

### Necessity

Later-stage feature after trend changes are reliable.

---

# 7. Data Flow Design

## 7.1 Batch-First Flow

The recommended source-of-truth flow is batch-first.

1. Bronze discovers and stores raw items.
2. Extractor produces structured silver data.
3. Silver stores interviews, rounds, assessment items, signals, chunks, and evidence.
4. Gold role resolver builds clean company-role-level cohorts.
5. Gold question resolver maps raw prompts to canonical questions and families.
6. Gold aggregation jobs compute question trends, topic trends, interview profiles, and signal trends.
7. Gold search jobs build search documents and embeddings.
8. Product APIs read from gold tables.
9. User-specific services combine gold with user progress and coding history.

Batch-first is safer because canonicalization, embeddings, reranking, and human review do not need to run inside a user-facing write path.

---

## 7.2 Immediate Aggregation Option

If near-real-time freshness is needed, use a deterministic immediate overlay.

When a new silver interview is committed:

- Add the interview to a pending gold queue.
- Run only deterministic mapping first: exact prompt hash, known external reference, known platform slug, existing canonical match.
- Write safe question occurrences.
- Increment affected trend rows or store pending overlay rows.
- Defer ambiguous matches for batch resolver or review.

Do not run expensive embedding, cross-encoder, or LLM matching inside the silver write transaction.

Recommended product behavior:

| Product surface | Data freshness source |
|---|---|
| Public pages | Latest completed batch |
| Internal/admin preview | Batch + pending overlay |
| Premium fresh feed | Batch + deterministic immediate overlay |
| Final canonical trends | Batch-reviewed gold |

---

# 8. Canonicalization Design

Gold canonicalization should follow a resolver ladder.

## 8.1 Company and Role Resolution

Company and role resolution should produce clean cohorts.

Company resolution should handle:

- Exact company IDs
- Aliases
- Group-brand mappings
- Acquired or renamed companies
- Raw company strings
- Unknown/new companies

Role/level resolution should handle:

- Role family
- Role specialty
- Native company level
- Cross-company canonical rank
- IC vs manager track
- Confidence and review status

Output should land in `gold.role_levels` and `gold.interview_roles`.

---

## 8.2 Question Resolution

Question resolution should convert raw assessment item occurrences into canonical gold questions.

Recommended ladder:

| Tier | Method | Purpose |
|---|---|---|
| 1 | Exact prompt hash | Reuse known canonical for identical prompts |
| 2 | External reference parsing | Map LC/CF/HackerRank URLs, IDs, slugs, or named references |
| 3 | Lexical/title matching | Match obvious normalized titles |
| 4 | Embedding candidate retrieval | Find semantically similar canonical questions |
| 5 | Reranking | Avoid false matches among similar topics |
| 6 | LLM match scoring | Resolve ambiguous middle cases |
| 7 | Human review | Finalize uncertain mappings |
| 8 | Deferred clustering | Promote recurring unresolved prompts as custom canonical questions |

The resolver should never force every prompt into an existing question. Some company-authored prompts are genuinely custom and should become gold questions only after enough evidence.

---

## 8.3 Variant and Family Resolution

A prompt can be:

- Exact same question
- Named platform variant
- Same family but different question
- Custom company variant
- Hybrid item
- Follow-up attached to a base question

Gold should represent this distinction.

Examples:

| Raw Prompt | Gold Interpretation |
|---|---|
| “LC 200 Number of Islands” | Exact LeetCode question |
| “Number of Islands but dynamic updates” | Variant in Islands family |
| “Design TinyURL with analytics” | URL Shortener family, analytics variant |
| “Parking lot with pricing and multiple gates” | Parking Lot LLD family, pricing/gates variant |
| “Behavior question on ownership” | Behavioral theme question/family |

---

# 9. Scoring Design

## 9.1 Frequency

Frequency should count distinct interviews, not repeated occurrences.

Why: if one candidate mentions the same question twice in one report, it should not double public support.

Important metrics:

- Distinct interviews
- Raw occurrence count
- Distinct companies if global
- Distinct roles if global
- Last seen date
- First seen date

---

## 9.2 Recency

Trend scores should use hard windows and recency decay.

Recommended windows:

| Window | Meaning |
|---|---|
| 90 days | Very recent / hot |
| 180 days | Current season |
| 365 days | Stable current market |
| 540 days | Sparse fallback only |

Older questions should not rank as trending unless they resurface.

---

## 9.3 Relevance

Relevance depends on scope closeness.

Recommended scope priority:

| Priority | Scope | Meaning |
|---|---|---|
| 1 | Exact company + role family + level | Best match |
| 2 | Same company + role family | Good company-specific fallback |
| 3 | Same company, any role | Company-specific broad fallback |
| 4 | Same role family + level globally | Role/level fallback |
| 5 | Same role family globally | Broad role fallback |
| 6 | Global all | Last resort |

Product responses must show the scope used.

---

## 9.4 Quality Weight

Quality should reduce noisy rows.

Factors that should reduce or block contribution:

- LLM rejected interview
- Estimated interview date
- Suspected duplicate
- Low extract confidence
- Authenticity suspect metadata
- Weak richness score
- Unreviewed ambiguous canonicalization

Factors that may increase trust:

- Strong evidence spans
- High richness score
- Clear company/role/level
- Recent interview date
- Multiple independent supporting interviews

---

## 9.5 Confidence Labels

Every trend should expose a confidence label.

Recommended labels:

| Label | Meaning |
|---|---|
| Strong signal | Enough exact-scope recent evidence |
| Moderate signal | Useful evidence, but smaller support or broader fallback |
| Weak signal | Sparse support, old data, or fallback used |
| Sparse data | Too little evidence for a strong conclusion |
| Review pending | Canonicalization or scope resolution is not fully trusted |

Confidence is not just internal. It should be visible in the product.

---

# 10. Retrieval Design

## 10.1 Deterministic Planner First

The system should not use an LLM router for every query.

Most queries contain structured filters:

- Company
- Role family
- Level
- Item type
- Date window
- Round type
- Topic

A deterministic planner can route these to SQL quickly.

---

## 10.2 SQL-Only Queries

Use SQL-only retrieval for:

- Trending questions
- Topic trends
- Interview loop profiles
- Round mix
- Company/role comparisons
- Readiness calculations
- Question frequency
- Confidence/fallback metadata

SQL-only should power most product pages.

---

## 10.3 Vector / Semantic Queries

Use semantic retrieval for:

- Similar interview experiences
- Similar prompts
- Narrative failure reasons
- Candidate mistakes
- Prep advice examples
- Evidence search
- Conceptual queries without exact filters

---

## 10.4 Hybrid Retrieval

Hybrid retrieval should work like this:

1. SQL narrows the eligible corpus by company, role, level, item type, and time window.
2. Full-text and vector search run inside or after that filtered set.
3. Results are reranked with both structured trend score and semantic similarity.
4. The LLM summarizes the final retrieved evidence.

This avoids the main failure mode of vector-only search: semantically similar but product-irrelevant results.

---

# 11. Product Surfaces Powered by Gold

## 11.1 Company Interview Guide Pages

Example page:

Google Software Engineer Interview Guide

Sections:

- Expected interview loop
- Most asked DSA questions
- Trending system design prompts
- Common topics
- Role-level differences
- Recently emerging questions
- Common failure reasons
- Evidence-backed examples
- Suggested preparation path

Value:

This becomes a major acquisition and SEO surface.

---

## 11.2 Role-Level Pages

Example:

Backend Engineer Mid-Level Interview Trends

Sections:

- Top companies asking this role
- Common DSA patterns
- System design frequency
- LLD/machine-coding frequency
- Common failure reasons
- Suggested study order

Value:

Useful for users who are not targeting one company yet.

---

## 11.3 Question Pages

Example:

Number of Islands Interview Intelligence

Sections:

- Platform references
- Companies where it appears
- Roles/levels where it appears
- Recent trend score
- Common follow-ups
- Related variants
- Similar questions
- Evidence snippets
- User completion status

Value:

Turns a coding problem into an interview intelligence object.

---

## 11.4 Personalized Dashboard

Example dashboard sections:

- Your target readiness
- Trending questions you have not solved
- Weak topics that matter for your target
- Upcoming revision tasks
- Recently emerging target-company questions
- Mock interview suggestions

Value:

This is the retention loop.

---

## 11.5 AI Chat / Research Mode

Gold should improve AI chat answers.

Examples:

- “What should I prepare for Google L4 backend?”
- “What changed recently for Amazon SDE2?”
- “Show me recent system design prompts around caching.”
- “Why am I weak for Meta E4?”

The chat should use SQL when possible and semantic retrieval only when needed.

---

# 12. Personalization Design

## 12.1 Gold Demand Signal

Gold represents what the market/company is asking.

Examples:

- Google Mid Backend has high graph/grid frequency.
- Amazon SDE2 has high behavioral ownership frequency.
- Senior backend roles increasingly include system design.

## 12.2 User Readiness Signal

User state represents what the user has prepared.

Examples:

- User solved 20 array problems but only 2 graph problems.
- User has multiple failed graph submissions.
- User has no system design practice history.
- User has strong SQL progress.

## 12.3 Gap Calculation

The gap is the difference between gold demand and user readiness.

Example:

Gold says graph is high priority for Google Mid Backend. User graph readiness is low. Therefore graph BFS/DFS becomes a high-priority recommendation.

## 12.4 Recommendation Generation

Recommendations should be ranked by:

- Target relevance
- Trend score
- User weakness
- Recency
- Difficulty fit
- Time until target interview
- Revision value
- Confidence

## 12.5 Explainability

Every recommendation should answer:

- Why this now?
- What target does it support?
- What evidence supports it?
- What user weakness does it address?
- What should the user do next?

---

# 13. Batch, Freshness, and Refresh Strategy

## 13.1 Recommended Source of Truth

The source-of-truth product data should come from completed batch refreshes.

Why:

- Safer canonicalization
- Easier debugging
- Lower cost
- Better review workflow
- Stable public pages
- Easier scoring consistency

## 13.2 Refresh Cadence

Recommended cadence:

| Job | Suggested cadence |
|---|---|
| Ingest raw sources | Continuous or scheduled by source |
| Silver extraction | Batch/drain jobs |
| Role resolution | After extraction batch |
| Question deterministic resolution | After role resolution |
| Embedding/rerank/LLM resolution | Batch, rate-limited |
| Trend aggregation | After mapping batch |
| Search document embedding | Batch or incremental |
| Change detection | After trend aggregation |
| User readiness refresh | On target change, progress event, or scheduled daily |

## 13.3 Immediate Overlay

Immediate overlay should only include deterministic safe mappings.

Do not immediately publish ambiguous semantic/LLM matches as public trends. Let batch and review finalize them.

---

# 14. Governance and Quality

## 14.1 Public Eligibility

An interview or occurrence should become public-aggregate eligible only after quality gates.

Recommended gates:

- Not LLM rejected
- Not known duplicate
- Not authenticity suspect
- Has usable company/role/date signal
- Canonicalization is auto-high-confidence or manually reviewed
- Meets minimum support threshold for public display

## 14.2 Minimum Support Thresholds

Suggested starting thresholds:

| Scope | Minimum support for public trend |
|---|---|
| Exact company + role + level | 3 distinct interviews |
| Company + role any level | 3 distinct interviews |
| Company broad | 5 distinct interviews |
| Global role/family | 5+ distinct interviews |
| Custom question promotion | 3 distinct interviews or strong repeated exact prompt evidence |

## 14.3 Human Review

Human review is needed for:

- Ambiguous question mapping
- Potential overmerged families
- New custom question promotion
- Company/level mapping uncertainty
- High-traffic public pages with low confidence

## 14.4 Auditability

Every product insight should trace back to:

- Supporting question occurrences
- Supporting interviews
- Round context
- Sample prompts
- Evidence snippets when available
- Refresh run metadata

---

# 15. Feature Prioritization

This is a complete design, but implementation should still be staged.

## Critical Foundation

These are foundational and should be implemented before high-level product features:

- Role-level mapping
- Interview role cohorts
- Canonical questions
- Question references
- Question occurrences
- Question families
- Question trends
- Refresh runs

## High-Value Product Layer

These create visible product value:

- Evidence-backed question cards
- Topic trends
- Interview profiles
- Signal/failure trends
- Confidence labels
- Fallback transparency
- Round-specific ranking
- Trend changes

## Personalization Layer

These create retention and moat:

- User target readiness
- Personalized gap analysis
- User gold recommendations
- Prep path generation
- Revision priority

## Advanced Growth Layer

These are valuable after the core is stable:

- Company comparison pages
- Watchlists and notifications
- Mock interview generation
- Market fit ranking
- Hiring-process anomaly detection

---

# 16. Risks and Mitigations

## Risk 1 — Overcomplicated schema

Adding too many tables too early can slow implementation.

Mitigation:

Keep the table set conceptually complete, but implement in phases. Avoid per-company/per-role hardcoded tables.

## Risk 2 — Bad canonicalization

Incorrectly merging different questions can destroy trust.

Mitigation:

Use exact matching first, semantic candidate generation second, reranking third, LLM only for ambiguity, and review for uncertain matches.

## Risk 3 — Stale trends

Old questions may dominate if lifetime frequency is used.

Mitigation:

Use hard windows and recency weighting. Separate lifetime support from trend score.

## Risk 4 — Low-support claims

Sparse data can make the product overconfident.

Mitigation:

Expose confidence labels, fallback scope, support counts, and sparse-data warnings.

## Risk 5 — Vector search noise

Vector-only retrieval can return semantically similar but structurally irrelevant examples.

Mitigation:

Use SQL filters first, then vector/full-text search inside the eligible set.

## Risk 6 — Personalization feels fake

Readiness scores can feel arbitrary.

Mitigation:

Every score must include a breakdown and explanation tied to gold trends and user history.

---

# 17. What Not To Build

Do not build separate hardcoded tables like:

- Google questions table
- Amazon questions table
- Backend questions table
- System design questions table
- DSA trending questions table per company

Use flexible aggregate tables with scope fields.

Do not make LLM routing the default for all queries. Use deterministic routing for structured product requests.

Do not publish one-off sightings as “trending.” Use labels like “recently reported” or “low confidence” when support is weak.

Do not hide uncertainty. Fallbacks, confidence, and evidence should be part of the product experience.

---

# 18. Recommended Final Product Promise

The gold layer should allow CrackedIn to say:

“Tell us your target company, role, level, and timeline. We will show what is actually being asked recently, what topics are rising, what the interview loop looks like, what candidates fail on, which questions map to known platforms, what custom variants appear, what you personally need to prepare, and why every recommendation is supported by real interview evidence.”

That is the full value of the gold layer.

---

# 19. Final Decision

The existing gold-layer direction is correct, but the complete design should include the expanded capabilities discussed in the CTO review.

The final design should not be treated as a minimal V1. It should be treated as a future-proof architecture with phased implementation.

The core gold layer should include:

- Product-owned role and level mapping
- Clean interview cohorts
- Canonical question catalog
- Question families and variants
- Platform references
- Occurrence mapping
- Review workflow
- Question trends
- Topic trends
- Interview profiles
- Signal/failure trends
- Search documents
- Change detection
- Company/role comparisons
- Refresh observability

The downstream product layer should include:

- User target readiness
- Personalized gold-based recommendations
- Watchlists
- Notifications

This design makes the gold layer more than an analytics layer. It becomes the intelligence engine that powers company guides, interview prep plans, AI chat, user readiness, mock interviews, and retention loops.

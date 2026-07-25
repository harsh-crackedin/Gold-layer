# CrackedIn Adaptive Preparation Planner — Product and Architecture Handoff

## Purpose of this document

This document explains:

- What preparation-planning system CrackedIn is building
- What has already been observed in the current repository and PR #22
- The agreed product behaviour
- The recommended architecture
- How company-specific, round-specific, and general subject plans should work
- How plans should be generated, stored, edited, adapted, and displayed
- Important edge cases
- A suggested implementation roadmap

This document is intended to be pasted into another ChatGPT or Codex conversation so the discussion can continue without losing context.

---

# 1. Product goal

CrackedIn is building an **adaptive interview preparation planner**.

A user should be able to ask natural-language requests such as:

- “Prepare me for Google L4 in one month.”
- “Create a plan for Google, Meta, Amazon, and Microsoft.”
- “Prepare me only for Amazon’s machine-coding round.”
- “Give me a system design plan.”
- “Help me master dynamic programming.”
- “I have 12 days and can study 90 minutes per day.”
- “Make tomorrow lighter.”
- “I completed more than today’s work. Give me something harder.”

The system should convert the request into a structured, evidence-backed preparation plan.

The planner should use:

- CrackedIn’s interview-experience database
- Canonical interview questions
- Topic and skill mappings
- Company, role, level, round, and location information
- The user’s existing progress
- LeetCode or practice history when available
- User availability, deadline, experience level, and preferences

The planner should generate a complete long-term curriculum, while the product normally shows the user only a small actionable window of the plan.

---

# 2. Core product decision

The planner should use a **hybrid model**:

> Topics and skills are the planning backbone.  
> Canonical questions are the execution units.

The system should not choose between:

1. Storing only important topics, or
2. Generating and storing an entire month of fixed questions.

Instead, it should do both at different levels.

## What should be stored

For the entire requested duration, store:

- Skill priorities
- Topic priorities
- Round allocation
- Phase structure
- Learning objectives
- Estimated effort
- Revision schedule
- Mock interview schedule
- Company-specific focus areas
- Evidence snapshot and confidence
- Planner and model versions

For the immediate execution window, store exact tasks:

- Canonical question IDs
- System design prompts
- LLD or machine-coding exercises
- Behavioural tasks
- Reading or concept tasks
- Revision tasks
- Estimated duration
- Difficulty
- Selection reason
- Evidence

## Rolling execution window

The full plan may be one month, three months, or longer, but the UI should normally materialise and show only the next **three days** of exact questions.

Recommended schedule zones:

| Zone | Behaviour |
|---|---|
| Past days | Immutable execution history |
| Current day and next 2 days | Exact, stable task window |
| Days 4–7 | Soft-planned and editable |
| Beyond day 7 | Skill/topic curriculum, not permanently fixed questions |

This provides consistency for the user while preserving adaptability.

---

# 3. Plan types

The planner should classify the user request before retrieving evidence or generating tasks.

## 3.1 Company all-round plan

Triggered when a user mentions one or more companies but does not restrict the request to one round.

Examples:

- “Prepare me for Google.”
- “Create a plan for Google and Meta.”
- “I have an Amazon SDE2 interview in 30 days.”
- “Prepare me for Google, Meta, Amazon, and Microsoft.”

The planner should prepare for **all relevant interview rounds**, based on available evidence.

Possible rounds include:

- Online assessment
- Recruiter screening
- Phone screen
- DSA or coding
- System design
- LLD
- Machine coding
- SQL
- Debugging
- Code review
- Domain knowledge
- Hiring-manager round
- Behavioural or bar-raiser round

The round distribution must be derived from the company, role, level, location, and available evidence.

Example:

- Google L4 may be DSA-heavy.
- Salesforce PMTS may be system-design-heavy.
- Amazon SDE2 may need DSA, LLD, system design, and Leadership Principles.
- Frontend roles may need browser, JavaScript, React, machine coding, and UI architecture.

The system must not use one fixed round distribution for every company.

---

## 3.2 Company-specific round plan

Triggered when the user mentions both a company and a specific round.

Examples:

- “Prepare me for Google’s system design round.”
- “Give me an Amazon machine-coding plan.”
- “Prepare me for Meta behavioural interviews.”
- “Create a DSA plan for Microsoft and Amazon.”
- “I only want LLD preparation for Flipkart.”

This is not merely an all-round plan with a filter.

A round-specific plan should go deeper into that round and include:

- Typical round structure
- Frequently observed topics
- Canonical questions
- Follow-up patterns
- Expected evaluation signals
- Common mistakes
- Timed practice
- Mock rounds
- Relevant prerequisites
- Revision

The planner may include prerequisites from related areas, but it should not silently add unrelated rounds.

For example, a system design plan may include databases, caching, networking, APIs, and distributed systems concepts, but it should not automatically add generic LeetCode grinding.

---

## 3.3 General subject or topic plan

Triggered when the user does not require a company-specific plan.

Examples:

- “Teach me system design.”
- “Create a DSA plan.”
- “Help me master graphs.”
- “Give me a one-month operating systems plan.”
- “Prepare me for backend fundamentals.”
- “I want to learn dynamic programming.”

This plan should use:

- Pedagogical ordering
- Prerequisite relationships
- General interview frequency
- Global corpus trends
- User skill level
- Existing progress and solved questions
- Time constraints

It must not claim company-specific evidence unless a company was actually part of the request.

Example graph curriculum:

```text
Graph representation
→ BFS and DFS
→ Connected components
→ Cycle detection
→ Topological sorting
→ Shortest paths
→ Union-Find
→ Advanced mixed problems
→ Timed sets
→ Revision
```

---

## 3.4 Mixed-focus company plan

Triggered when the user wants all rounds but asks for emphasis on one or more areas.

Examples:

- “Prepare me for Google, but focus mostly on system design.”
- “Create a complete Amazon plan, but I am weak in behavioural.”
- “Prepare me for Meta E5 with extra DSA.”

This should remain an all-round plan, but with explicit focus weights.

Example:

```json
{
  "scope": "company_all_rounds",
  "include_remaining_rounds": true,
  "round_focus": {
    "system_design": 0.60
  }
}
```

The phrase “focus on” should not automatically mean “exclude every other round.”

---

# 4. Deterministic routing rules

Use rules such as:

```text
Company mentioned + specific round mentioned
    → company_specific_round

Company mentioned + no round mentioned
    → company_all_rounds

No company + subject/topic mentioned
    → general_subject

Multiple companies + specific round
    → that round across all target companies

Multiple companies + no specific round
    → merged all-round company plan

Company plan + explicit focus area
    → mixed-focus company plan
```

Normalised scope values may be:

```text
company_all_rounds
company_specific_round
general_subject
mixed_focus
```

User-facing names can be:

- Complete interview plan
- Round-focused plan
- Subject mastery plan
- Custom focused plan

---

# 5. Normalised plan request

Natural-language input should first be converted into a structured request.

Example:

```json
{
  "scope": "company_all_rounds",
  "targets": [
    {
      "type": "company",
      "company": "google",
      "role_family": "backend",
      "level": "L4",
      "location": null,
      "weight": 0.40
    },
    {
      "type": "company",
      "company": "meta",
      "role_family": "backend",
      "level": "E4",
      "location": null,
      "weight": 0.30
    }
  ],
  "rounds": [],
  "subjects": [],
  "duration_days": 30,
  "daily_budget_minutes": 120,
  "available_days": ["mon", "tue", "wed", "thu", "fri", "sat"],
  "experience_level": "intermediate",
  "preferences": {
    "include_mocks": true,
    "include_revision": true,
    "unseen_questions_only": false,
    "allow_extra_practice": false
  }
}
```

The rest of the planner should operate on this normalised request instead of repeatedly interpreting the original chat message.

---

# 6. Evidence retrieval strategy

The initial idea was to retrieve around 20–30 interview experiences per company and ask an LLM to scan them.

That can be useful for representative evidence, but it should not be the primary statistical method.

For four companies, sending 80–120 raw experiences into an LLM repeatedly would be:

- Slow
- Expensive
- Inconsistent
- Difficult to validate
- Vulnerable to duplicates
- Larger than necessary

The recommended approach is:

> Aggregate broadly; sample narrowly.

## 6.1 Aggregate over the complete eligible cohort

Use SQL, projections, or materialised views to calculate:

- Number of interviews
- Round distribution
- Topic frequency
- Canonical question frequency
- Role and level distribution
- Location-specific differences
- Outcome-linked patterns
- Recency-weighted frequency
- Trend direction
- Distinct source count
- Confidence score

## 6.2 Retrieve representative experiences

Select approximately 20–30 representative experiences per company for:

- Evidence citations
- Real question wording
- Follow-up patterns
- Round ordering
- Qualitative details
- Interviewer signals
- Candidate feedback

Sampling should be stratified across:

- Recent and older evergreen experiences
- Different levels
- Different roles
- Locations
- Outcomes
- Round types
- Sources

Duplicate or reposted experiences must be deduplicated.

---

# 7. Precomputed company interview profiles

To avoid rescanning the full corpus for every user, create an aggregate projection resembling:

```text
company_interview_signal
------------------------
company_id
role_family
level_canonical
location
round_type
topic_id
question_id
time_bucket
interview_count
distinct_source_count
weighted_frequency
offer_frequency
rejection_frequency
trend_score
last_seen_at
confidence_score
data_snapshot_version
```

This projection should support fast queries for:

- Company all-round plans
- Company round-specific plans
- Multi-company merging
- Trend summaries
- Evidence confidence

Raw interview retrieval should be reserved for evidence hydration, low-confidence cohorts, or detailed explanations.

---

# 8. Canonical questions and skill mapping

Question canonicalisation is mandatory.

The same interview question may appear as:

- “Design TinyURL”
- “Build a URL shortener”
- “Design bit.ly”
- “How would you shorten long links?”

These should resolve to one canonical question:

```text
question_id = sd:url-shortener
```

A canonical question should include:

```json
{
  "question_id": "sd:url-shortener",
  "title": "Design a URL shortener",
  "modality": "system_design",
  "skills": [
    "id-generation",
    "caching",
    "database-sharding",
    "availability"
  ],
  "difficulty": "medium",
  "prerequisites": [
    "http",
    "databases"
  ],
  "estimated_minutes": 60
}
```

Each task must store both:

- `question_id`
- `skill_ids`

Questions are the practice units. Skills are the optimisation and adaptation units.

Without canonicalisation:

- Frequency counts will be inaccurate.
- Multi-company plans will repeat equivalent questions.
- Progress cannot be reliably mapped to skill mastery.
- Evidence becomes fragmented.

---

# 9. Multi-company plans

A multi-company plan must not be four separate company plans concatenated together.

## 9.1 Build each company profile independently

For each company, derive:

- Round mix
- Topic weights
- Skill weights
- Question weights
- Confidence
- Evidence volume
- Recency
- Role and level adjustments

## 9.2 Separate common and company-specific requirements

Example:

```text
Common high-value preparation
├── Graphs
├── Dynamic programming
├── Trees
├── System design fundamentals
└── Behavioural stories

Google-specific
└── Additional algorithmic depth

Amazon-specific
├── Leadership Principles
└── LLD

Meta-specific
├── Speed-focused coding
└── Product-oriented behavioural

Microsoft-specific
└── Implementation and design discussion
```

## 9.3 Merge using weighted coverage

A task or skill priority may use a score such as:

```text
priority =
    target_company_weight
  × interview_frequency
  × recency_weight
  × round_importance
  × trend_weight
  × user_gap
  × confidence
  × prerequisite_readiness
  - repetition_penalty
  - already_solved_penalty
  - workload_penalty
```

Skills common across multiple companies should receive a coverage bonus.

## 9.4 Preserve company-specific capacity

The plan should reserve capacity for target-specific requirements.

A starting model may be:

- Common high-leverage preparation
- Company-specific preparation
- Revision and mocks

The exact ratio should come from actual overlap, not remain permanently hard-coded.

---

# 10. Planner architecture

Recommended high-level flow:

```text
User query
    ↓
Plan intent detection
    ↓
Request normalisation
    ↓
Entity, role, level, round, and constraint resolution
    ↓
Evidence profile retrieval
    ↓
User progress and skill-state retrieval
    ↓
Skill and round priority calculation
    ↓
Deterministic curriculum blueprint
    ↓
Candidate question ranking
    ↓
Three-day task materialisation
    ↓
LLM sequencing and explanation
    ↓
Deterministic validation
    ↓
Persist version, tasks, and evidence
    ↓
Render plan in right sidebar and My Prep surface
```

---

# 11. Three logical planner layers

## Layer A: Evidence profile

Represents what the source data says.

Example fields:

- Target companies
- Role and level
- Typical rounds
- Round distribution
- Top skills
- Top questions
- Trends
- Confidence
- Evidence volume
- Fallback level

## Layer B: Curriculum blueprint

Represents what the user should learn during the entire plan.

Example fields:

- Phases
- Skill objectives
- Target mastery
- Topic sequencing
- Allocated minutes
- Revision cadence
- Mock cadence
- Company-specific focus
- Round allocation

## Layer C: Execution window

Represents the exact tasks currently shown to the user.

Example fields:

- Scheduled date
- Canonical question
- Task modality
- Skill IDs
- Difficulty
- Estimated minutes
- Evidence
- Selection reason
- Progress state
- User override state

---

# 12. Deterministic code versus LLM responsibilities

This separation is critical.

## Deterministic code should control

- Dates
- Duration
- Daily time budget
- Available days
- Rest days
- Round allocation constraints
- Topic prerequisites
- Question eligibility
- Deduplication
- Candidate ranking
- Difficulty progression
- Revision scheduling
- Already-solved filtering
- Evidence references
- User locks and overrides
- Plan validation
- Regeneration boundaries

## The LLM may control

- Natural-language summaries
- Day names and themes
- User-facing explanations
- Choosing between similarly ranked eligible candidates
- Pedagogical sequencing within safe constraints
- Converting ambiguous natural language into a structured request
- Explaining why a task was selected

The LLM must not:

- Invent questions
- Invent database IDs
- Claim unsupported company evidence
- Ignore daily time constraints
- Rewrite user-locked tasks
- generate arbitrary tasks outside supplied candidate pools

Every generated question reference must come from an approved candidate pool or a controlled synthesised task type such as:

```text
read:<topic>
concept:<topic>
mock:<label>
revision:<question_id>
```

---

# 13. Current repository findings

## 13.1 PR #22 agent pipeline

PR #22 introduces a staged agent pipeline under:

```text
api/services/chat_agent_v2/
```

Its main sequence is conceptually:

```text
prechecks
→ selector
→ tool execution
→ answerer
→ citation verification
→ emit
```

It currently supports read-only tools such as:

- Interview search
- Round-level evidence retrieval
- Interview detail retrieval
- Company insights
- Corpus statistics
- General concept explanation

The current V2 registry explicitly permits read-only tools only.

Therefore, PR #22 can research and explain preparation needs, but it cannot yet safely create, edit, or regenerate a persisted plan.

A separate mutation-capable planner lane is required.

## 13.2 Existing Prep Mode backend

The repository already contains an older Prep Mode implementation with useful ideas:

- Deterministic skeleton generation
- Candidate pools
- Company, level, and role profiles
- LLM fill-in
- Strict JSON output
- Validation and retry
- Plan templates
- Per-user plan progress
- Evidence metadata
- Revision scheduling

Relevant areas include:

```text
api/services/prep_generator.py
api/services/prep_service.py
api/services/prep_skeleton.py
api/services/prep_validator.py
api/services/prep_data.py
api/services/prep_priors.py
api/services/prep_tools.py
api/routes/prep.py
docs/PREP-MODE-DESIGN.md
```

These concepts should be reused where appropriate.

However, the older Prep Mode still depends on the earlier database and service structure, while PR #22 is designed around the newer Postgres-backed chat pipeline.

The planner should be aligned with the current Postgres architecture instead of connecting the frontend directly to the old SQLite-oriented service.

## 13.3 Current frontend

The desktop `/chat` right sidebar renders a shared preparation panel.

Relevant components include:

```text
web/components/chat/streak-sidebar.tsx
web/components/prep/my-prep-panel.tsx
web/components/chat/today-tasks.tsx
```

The same `MyPrepPanel` concept is reused for desktop and mobile.

However, the current “Today’s Focus” task list is mock data stored in local React state.

The right sidebar is therefore visually prepared for a planner, but it is not yet connected to real planner APIs or persisted progress.

---

# 14. Agent integration

Do not run full long-duration plan generation inside the normal PR #22 synchronous selector loop.

Plan generation may involve:

- Multiple aggregate queries
- Evidence analysis
- Candidate ranking
- LLM generation
- Validation
- Retry
- Persistence

This may exceed the normal tool execution budget.

## 14.1 Read-only planner tools

These can be safely exposed to the normal agent pipeline:

```text
analyse_plan_request
preview_plan_profile
get_plan_window
get_plan_progress
get_plan_evidence
list_user_plans
```

## 14.2 Mutation planner tools

These require a separate secured registry or execution path:

```text
create_plan
update_plan_constraints
edit_plan_day
replace_plan_task
move_plan_task
record_task_result
pause_plan
resume_plan
regenerate_future_window
archive_plan
```

Mutation tools require:

- Authentication
- Ownership checks
- Explicit intent
- Input validation
- Idempotency keys
- Rate limits
- Audit events
- Optimistic concurrency

## 14.3 Asynchronous creation pattern

The creation endpoint should return quickly:

```json
{
  "plan_id": "plan_123",
  "status": "generating"
}
```

A worker can generate the plan and update status to:

```text
ready
failed
needs_input
```

The frontend can poll, subscribe, or consume a job event.

Use an idempotency key based on:

```text
user_id
+ normalised_request_hash
+ client_request_id
```

This prevents duplicate plans from retries or double clicks.

---

# 15. Plan states

Recommended states:

```text
draft
analysing
generating
ready
needs_input
failed
paused
completed
archived
```

These states should be surfaced explicitly in the frontend.

---

# 16. Suggested storage model

A compact versioned model can be built with approximately five core tables.

## 16.1 `prep_plans`

User-owned plan identity.

```text
id
user_id
plan_kind
title
status
timezone
start_date
target_date
daily_budget_minutes
active_version_id
created_at
updated_at
```

## 16.2 `prep_plan_versions`

Immutable generation snapshots.

```text
id
plan_id
version
normalised_request_jsonb
evidence_profile_jsonb
curriculum_blueprint_jsonb
data_snapshot_version
planner_version
model_version
generation_reason
created_at
```

## 16.3 `prep_plan_tasks`

Exact materialised tasks.

```text
id
plan_id
version_id
scheduled_date
position
question_id
task_type
skill_ids
estimated_minutes
difficulty
status
locked
selection_reason
evidence_jsonb
user_override
created_at
updated_at
```

## 16.4 `prep_plan_events`

Append-only activity and adaptation history.

```text
id
plan_id
user_id
event_type
task_id
payload_jsonb
created_at
```

Possible events:

```text
plan_created
task_completed
task_failed
hint_used
task_skipped
task_replaced
task_moved
day_edited
constraints_changed
window_regenerated
plan_paused
plan_resumed
plan_completed
```

## 16.5 `user_skill_state`

Current skill projection.

```text
user_id
skill_id
mastery_score
confidence
attempt_count
success_count
hint_count
failure_count
last_practised_at
next_revision_at
updated_at
```

Optional aggregate or projection tables may support company evidence and analytics.

---

# 17. Editing behaviour

Users should be able to edit upcoming days.

Supported actions:

- Move a task
- Replace a task
- Reduce daily workload
- Add a rest day
- Change availability
- Lock a task
- Remove a topic
- Prioritise a company
- Mark “already know this”
- Add a custom task
- Change target date
- Increase or reduce daily minutes

User edits should become explicit constraints or overrides.

Future regeneration must preserve:

- Completed tasks
- Past days
- User-locked tasks
- User-added tasks
- User-disabled topics
- Explicit time constraints

The planner must never silently overwrite manual edits.

---

# 18. Adaptation behaviour

## 18.1 User completes more than expected

Do not automatically overload the current day.

Possible adaptations:

- Pull one task forward
- Increase future difficulty
- Add an optional task
- Add a mock
- Add company-specific breadth
- Reduce introductory tasks
- Schedule revision earlier

Additional work should be optional unless the user has enabled an extra-practice mode.

## 18.2 User falls behind

Do not carry an unlimited backlog.

Offer strategies such as:

- Reduce daily load
- Extend the target date
- Drop low-priority topics
- Compress the plan
- Preserve the deadline and prioritise high-yield skills
- Pause and resume

## 18.3 User repeatedly struggles

Update skill state rather than only marking individual question status.

Example:

```text
failed without progress
    → reduce skill mastery

completed with hint
    → small mastery increase

completed independently
    → stronger mastery increase

repeated fast success
    → increase difficulty or reduce repetition
```

## 18.4 Regeneration boundary

Normally do not regenerate:

- Past days
- Completed tasks
- Locked tasks
- Manually added tasks

Normally regenerate:

- Days after the stable three-day window
- Future tasks affected by changed availability
- Future tasks affected by new mastery signals
- Future tasks affected by a changed target date

---

# 19. Frontend recommendation

Keep the shared `MyPrepPanel`, but replace mock tasks with API-backed state.

## Right sidebar

Recommended structure:

```text
Active plan selector
Progress or streak

Today
  task
  task
  task

Tomorrow
  collapsed preview

Day 3
  collapsed preview

Adjust plan
```

The right rail should remain execution-focused.

It should not become a dense full-month calendar.

## Expanded plan surface

Use a dedicated route or drawer for:

- Full curriculum view
- Topic and round distribution
- Company coverage
- Evidence confidence
- Plan editing
- Progress analytics
- Readiness
- Past activity
- Upcoming phases

## Task actions

Each task may support:

- Complete
- Completed with hint
- Struggled
- Skip
- Replace
- Move
- Open question
- Why this task?
- Add time spent
- Lock

“Why this task?” should show real reasons such as:

```text
Asked in 18 Meta interviews
Covers graph traversal
Shared across 3 target companies
High-priority skill for your target role
You recently struggled with this skill
Due for revision
```

---

# 20. Edge cases

## Evidence and cohort edge cases

- One company has thousands of experiences while another has very few.
- The requested level is missing.
- The requested role is missing.
- The location-specific process differs.
- The company changed its interview process recently.
- Experiences disagree on the number of rounds.
- A round appears in only one duplicated source.
- Old and recent evidence conflict.
- The requested company has insufficient evidence.
- The requested round has no canonical questions.
- Some interview experiences are gated or unavailable.
- Several companies use different names for equivalent rounds.

The system should expose confidence labels such as:

```text
high confidence
moderate confidence
limited evidence
fallback profile
```

It must not pretend generic priors are strong company-specific evidence.

## Request edge cases

- No duration supplied
- No daily time supplied
- Interview date in the past
- Unrealistic workload
- Weekends-only preparation
- Very short one-day sprint
- Very long one-year plan
- Multiple active plans
- Same plan requested twice
- User changes target company midway
- User changes role or level
- User requests a plan for a non-interview subject
- User asks for only theory
- User asks for only questions
- User asks for unseen questions only

## Task edge cases

- Question already solved
- Question completed outside CrackedIn
- Equivalent question already scheduled
- Broken question link
- Empty candidate pool
- Task too difficult because prerequisites are missing
- User repeatedly skips one modality
- Several target companies ask the same question
- User completes future tasks early

## Concurrency and reliability edge cases

- LLM timeout
- Invalid JSON
- Validation failure
- Partial evidence query failure
- Worker retry
- User edits while generation is running
- Two regeneration jobs run at once
- Frontend repeats a mutation
- Planner version changes
- Evidence snapshot changes
- Plan generation succeeds but notification fails

Use optimistic concurrency with:

```text
expected_plan_version
```

Reject or merge mutations when the active version has changed.

---

# 21. Validation rules

A generated plan should be rejected if it violates important invariants.

Examples:

- Unknown question reference
- Duplicate task in one day
- Equivalent canonical question repeated too soon
- Daily time budget exceeded
- Missing required round coverage
- Prerequisite violation
- Invalid date
- Task assigned outside available days
- User-locked task overwritten
- Unsupported company claim
- Evidence missing for company-specific task
- Wrong modality in a round-specific plan
- Too many difficult tasks in sequence
- Revision scheduled before first exposure

The validator should return machine-readable errors and allow targeted retry.

---

# 22. Observability and evaluation

## Planner quality metrics

- Unsupported-task rate
- Evidence coverage per task
- Duplicate-question rate
- Prerequisite violation rate
- Daily-budget violation rate
- Company coverage balance
- Round coverage balance
- Topic diversity
- Difficulty progression
- Regeneration stability
- Number of validation retries
- Low-confidence task percentage

## Product metrics

- Plan creation completion
- Time to plan readiness
- Day-one task start
- Three-day completion
- Seven-day retention
- Task replacement rate
- Too-easy and too-hard feedback
- Manual edit rate
- Percentage of users falling behind
- Plan completion rate
- Mock participation
- Replan acceptance rate

## Adaptation metrics

- Whether difficulty changes improve completion
- Whether reduced-load plans improve adherence
- Whether revision improves independent success
- Whether users accept regenerated windows
- Whether skill mastery predicts future task performance

Persist for every plan version:

- Planner version
- Model version
- Evidence snapshot version
- Request hash
- Validation errors
- Regeneration reason
- Changed tasks
- Generation duration
- Token and provider usage where relevant

---

# 23. Recommended implementation sequence

## Phase 1 — Align architecture

- Move planner storage and retrieval to Postgres.
- Define the normalised `PlanRequest`.
- Define canonical scope values.
- Reuse the good ideas from the existing Prep Mode.
- Create canonical question and skill relationships.
- Separate read-only planner tools from mutation tools.
- Define planner versioning and idempotency.

## Phase 2 — Evidence profile service

- Build aggregate company, role, level, location, and round projections.
- Add confidence and trend calculation.
- Add canonical question frequency.
- Add stratified representative sampling.
- Support multi-company evidence merging.

## Phase 3 — Hybrid curriculum planner

- Build deterministic curriculum blueprint generation.
- Rank candidate questions.
- Materialise only the next three days.
- Generate user-facing summaries and explanations.
- Validate every result.
- Persist immutable versions.

## Phase 4 — Frontend integration

- Replace mock `TodayTasks`.
- Add API-backed three-day execution window.
- Add generation and failure states.
- Add task actions.
- Add plan selector.
- Add expanded plan-management route or drawer.

## Phase 5 — Adaptation engine

- Build `user_skill_state`.
- Process completion, hint, failure, skip, and timing events.
- Add revision scheduling.
- Add rolling-window regeneration.
- Respect user locks and overrides.
- Add ahead-of-plan and behind-plan strategies.

## Phase 6 — Agent integration

- Add planner read tools to the V2 agent.
- Add a secured mutation lane.
- Support requests such as:
  - “Build me a Google plan.”
  - “Make tomorrow lighter.”
  - “Replace this question.”
  - “I completed everything.”
  - “Pause my plan.”
- Attach chat sessions to the relevant plan.
- Add job-status handling.

## Phase 7 — Evaluation and rollout

- Create planner golden cases.
- Test sparse and conflicting evidence.
- Test multi-company merging.
- Test round-specific isolation.
- Test user edits and concurrent regeneration.
- Launch to a small beta group.
- Measure adherence and plan usefulness before increasing complexity.

---

# 24. Final architecture decision

The agreed architecture is:

```text
Interview experiences and knowledge data
        ↓
Canonical company, round, topic, skill, and question signals
        ↓
Evidence-backed target profile
        ↓
User-specific skill priorities
        ↓
Full-duration curriculum blueprint
        ↓
Rolling three-day exact task window
        ↓
User progress, edits, and external activity
        ↓
Skill-state update
        ↓
Future-window recalculation
```

The system should:

- Prepare for all relevant rounds when the user requests company preparation.
- Prepare only for the requested round when the user explicitly asks for a round-specific plan.
- Generate a general pedagogical plan when the user requests a subject or topic without a company.
- Support multiple companies by merging common preparation and preserving company-specific requirements.
- Store skills, topics, and curriculum for the full duration.
- Materialise exact questions only for the near-term rolling window.
- Preserve every task actually shown or completed.
- Use deterministic ranking, scheduling, constraints, and validation.
- Use the LLM mainly for interpretation, sequencing among approved candidates, summaries, and explanations.
- Adapt future work based on user progress without rewriting completed history or manual edits.

---

# 25. Important design principle

The product advantage is not simply that CrackedIn can generate a list of interview questions.

Static company-tagged lists already exist.

The advantage should be:

- Evidence-backed company and round targeting
- Multi-company overlap optimisation
- Role and level awareness
- Skill-based progression
- Adaptive question selection
- Revision and forgetting management
- Personal progress
- Transparent selection reasons
- A living plan that changes as the user improves

The planner should therefore behave like a **stateful preparation operating system**, not a one-time roadmap generator.

---

# 26. Suggested next discussion topics

The next conversation can continue with one of these areas:

1. Final Postgres schema and table relationships
2. Planner APIs and request/response contracts
3. Company evidence aggregation queries
4. Canonical question and skill schema
5. Multi-company scoring algorithm
6. Curriculum and daily scheduling algorithm
7. Skill mastery model
8. Plan regeneration rules
9. PR #22 mutation-tool architecture
10. Frontend right-sidebar data contract
11. Worker and job architecture
12. Planner validation rules and tests

---

# 27. Continuation prompt for another chat

Use the following prompt after attaching or pasting this document:

> Read this entire planner handoff before answering. Treat the product decisions and architecture described here as the current agreed direction. We are building the CrackedIn adaptive preparation planner on top of the current repository, including PR #22’s agent pipeline and the existing but older Prep Mode implementation. Do not restart the product discussion from zero. Continue from this architecture, point out contradictions when necessary, and think like a CTO or senior system architect. The next task is: [INSERT TASK HERE].


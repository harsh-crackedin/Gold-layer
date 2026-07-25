# CrackedIn Adaptive Preparation Planner — System Design (v1)

Status: proposed design, grounded in a full audit of the repository at `4e6491c`
(master), PR #22 (`Feat/router-pipeline`, head `6177fe8`), and PR #18
(`feat/mvp-ai-tutor-orchestrator`, head `0813e46` — source of the
recommendation engine integrated in §7.6). Revision 3 reworks priority
scoring (§7.2), the storage model's versioning semantics (§8), worker
topology (§9), adds a security section (§14), scopes the PR #18 reuse to the
coding track (§7.6), and cuts a walking-skeleton M0 into the roadmap (§15)
— in response to an external architecture review.
Companion to the product handoff document ("CrackedIn Adaptive Preparation
Planner — Product and Architecture Handoff"), which remains the product source
of truth. This document turns that handoff into an implementable architecture
for *this* codebase, corrects it where it conflicts with repo reality, and
resolves the edge cases around under-specified user requests (no company, only
companies, no role, topic-only, etc.).

---

## 0. Decisions taken up front (read this first)

Five decisions were ambiguous or contradicted by the chat discussion / the
repository. Each was resolved to the option below; all five are overridable,
but the rest of this document assumes them.

| # | Decision | Resolution | Why |
|---|---|---|---|
| D1 | Sync vs async plan generation | **Async job, triggered by a fast tool call.** `create_plan` normalises the request, inserts a plan row with `status=generating`, enqueues a job, and returns `{plan_id, status}` well inside the pipeline's 5-second tool budget. A worker generates; the sidebar polls/receives the ready state. | The chat framing ("run the tool call and create a plan") collides with two hard facts: handoff §14 forbids generation inside the selector loop, and PR #22 enforces a 5 s per-tool timeout (`chat_agent_v2/config.py:152`) plus a constructor-level read-only registry invariant (`chat_agent_v2/tools/registry.py:17-24`). Full generation (aggregates + candidate ranking + LLM fill + validation retries) is a 20–60 s job. |
| D2 | Plan UI surface | **Execution window in the rail + a curriculum overview surface. No day-by-day multi-week calendar.** | Handoff §19 asks for an "expanded plan surface", but `web/components/prep/my-prep-panel.tsx` carries a deliberate product comment that a static multi-week plan is a *banned concept*, and the old `/prep/plan` day timeline was retired in favour of the Days/Topics toggle. The curriculum overview (phases, topics, coverage, confidence, editing) satisfies the handoff's intent without resurrecting the banned artifact — because what we store beyond the window is skills/topics, not a fixed question calendar (§2 of the handoff already says exactly this). |
| D3 | Evidence corpus | **Postgres `silver.*` + a "gold-lite" canonical layer, published as snapshot tables into the app database.** Not the legacy SQLite corpus; not the full gold clustering pipeline. | The live SQLite corpus has no question identity, no level resolution, and is slated for retirement. `silver.*` is built, rich (rounds, `assessment_item_occurrence` with `prompt_hash`, LLM-resolved levels, evidence spans) and currently has zero API readers — the planner is the reason to finally wire it. The full gold pipeline (ANN → cross-encoder → LLM adjudication, `docs/GOLD-LAYER-HANDOFF.md`) is weeks of work and not a prerequisite: gold-lite (below, §5.3) gives real canonicalisation where it matters most. |
| D4 | V1 scope | **Create + rolling window + light adaptation.** Skill state is *stored* from day one but only lightly used; the full mastery model, spaced-revision ladder, and ahead/behind replan strategies are v2. | Ships a genuinely "living" plan (regeneration on edits/completions, already-solved filtering, locks) without gating launch on a research-grade adaptation engine. The storage model (§8) is designed so v2 needs no schema rework. |
| D5 | User-side scoring | **Reuse the PR #18 recommendation engine's pure modules as the planner's pedagogical scoring half** (§7.6), instead of building `user_gap`, already-solved handling, and difficulty calibration from scratch. | PR #18 (`api/services/recommendations/`) is a deterministic 8-signal heuristic scorer — no ML — whose user-side half (topic weakness, failed-attempt/near-solve, revision staleness, maturity×difficulty matching, anti-frustration guardrails, diversity) implements exactly the user-side terms of the planner's priority formula. 12 of its 14 modules are pure (stdlib dataclasses, explicit `now`, no I/O), have zero tutor coupling, and already run on the same Postgres tables and UUID user ids this design targets. It knows nothing about companies or interview evidence, so it complements rather than overlaps the evidence layer. |

### Contradiction log (handoff vs chat vs repo)

1. **"Run the tool call and create a plan" vs async lane** — resolved by D1.
   The tool call *is* the trigger; it is just not the place the work happens.
2. **"Expanded plan surface" vs "banned concept"** — resolved by D2.
3. **"Fetch all the interview experiences … and plan based on it"** (chat) vs
   handoff §6 "aggregate broadly, sample narrowly" — the handoff wins.
   Raw-experience scans do not fit any latency or cost budget; the planner
   reads precomputed signal tables and hydrates a small stratified sample for
   citations only.
4. **Handoff §13 describes PR #22 as the newest layer** — no longer true. The
   frontend at master is *newer* than PR #22's `web/` (My Prep panel, mock
   `TodayTasks`, the API-backed `/prep` surface and its retirement of the day
   timeline all post-date the PR). PR #22's `mergeable_state` is `dirty`.
   The design below targets: backend = `chat_agent_v2` (PR #22, rebased),
   frontend = master's components.
5. **Handoff §8 calls canonicalisation mandatory; the repo has none.**
   `round_questions` has no dedup key at all; `silver` has `prompt_hash`
   (exact-match only, explicitly documented as "NOT canonicalization");
   `gold.*` is designed but zero lines exist. Resolved by D3 (gold-lite).
6. **Handoff §16 storage vs existing tables** — Postgres already has legacy
   `app.user_prep_plans` / `app.user_prep_task_progress` /
   `app.user_readiness_snapshots` (deferred at cutover, orphan `template_id`,
   router unmounted). The new model uses the handoff's table names
   (`prep_plans`, `prep_plan_versions`, …) which do **not** collide; legacy
   tables are left untouched and deprecated.

---

## 1. What the repository actually provides today

Condensed from the audit; file paths are the proof points.

**Chat pipeline (PR #22, `api/services/chat_agent_v2/`).**
Synchronous SSE generator: prechecks → selector loop (≤3 tool rounds, ≤4
parallel calls, 5 s per call, `temperature=0`, all registered tools offered
every round) → optional answerer → citation verification → emit.
Tools are single-source declared (`tools/builtin.py::LAUNCH_TOOLS`, six
read-only tools) and dispatched generically (`tools/executor.py`); adding a
read-only tool requires touching only `handlers.py` + `builtin.py` (proven by
`tests/test_chat_agent_v2_extensibility.py`). Hard constraints that shape the
planner integration:

- `ToolRegistry.__init__` **raises** on any non-read-only tool.
- `ToolContext` carries **no `user_id` / `session_id`** — identity stops at
  the pipeline layer (`pipeline.py:83-89` builds the executor without it).
- Selector direct replies are capped at 400 tokens / "under 120 words"
  (`config.py:157`, `prompt/rounds.py`); the answerer path hardcodes
  `citation_required=True` (`pipeline.py:132`).
- Real per-tool status events exist (`stages/selector.py::_TOOL_VERB`), and
  the route already maps `AgentEvent("status"|"text"|"sources"|"done")` onto
  SSE frames the current frontend understands.

**Legacy Prep Mode (paused, unmounted — `api/main.py:167-172`).**
A complete v1 planner exists in git: deterministic skeleton
(`prep_skeleton.py`), 4-tier cohort fallback with prior blending
(`prep_data.py`, `prep_priors.py`, α = 0.70/0.45/0.25/0.10 by fallback tier),
recency-decay weighting (`prep_weighting.py`), Gemini fill-in with
schema-constrained JSON + validate/retry (`prep_generator.py`,
`prep_validator.py`), template caching with versioning and snapshot hashes
(`prep_templates.py`), and per-user progress/readiness/replan/pause
(`prep_service.py`). It is SQLite-bound and integer-user-keyed, so the
*service* is dead code, but five modules are pure and directly portable:
`prep_priors`, `prep_skeleton`, `prep_validator`, `prep_weighting`,
`prep_topics` — ~65 passing unit tests among them.

**Evidence data.**
- Live-but-legacy: SQLite `interview_posts` / `interview_rounds` /
  `round_questions` (flat, no question identity).
- Built-but-unwired: Postgres `silver.interview` / `silver.round` /
  `silver.assessment_item_occurrence` (+ `signal`, `evidence_span`,
  `company`, `company_level`, `level_resolution`) — rich, deduped at post
  level, `prompt_hash` at item level, LLM-resolved canonical levels.
  **Zero readers in `api/`.**
- Not started: `gold.*` canonical items (`docs/GOLD-LAYER-HANDOFF.md`).
- No vector index over interview text (silver embedding column deferred).

**User history.**
LeetCode sync is fully live: `user_data.user_coding_submissions`
(user_id UUID, slug, verdict, ts — indexed exactly the way a planner needs),
`user_data.user_coding_problems` per-problem rollups, and
`catalog.coding_problems` (platform+slug, `topic_tags`, `display_id`).
**Blocking gap:** nothing joins `catalog.coding_problems` (slug-keyed) to
interview-frequency data (`problems.leetcode_number` in SQLite;
`external_refs` inside silver `attributes`). Without a bridge, "you've
already solved 40% of what Amazon asks" is impossible. The bridge is a
prerequisite task in the roadmap (§15, P1).

**Recommendation engine (PR #18, `api/services/recommendations/`, unmerged).**
A deterministic, fully heuristic "what should I solve next" engine built for
the AI-tutor sidebar: fixed-weight linear scoring over 8 signals
(`SCORING_WEIGHTS` sums to exactly 1.0 — importance 0.16,
interview_frequency 0.14, topic_weakness 0.18, difficulty_match 0.14,
failed_attempt 0.16, revision_due 0.10, progression 0.08, freshness 0.04),
user-state classifiers (maturity ladder by solve count; activity states;
learning states), per-topic weakness/need analysis with evidence tiers, seven
candidate generators, rule guardrails, and a diversity re-rank. **No ML
anywhere** — no embeddings, no trained model, no collaborative filtering.
Properties that matter here: 12 of 14 modules are pure and DB-free (frozen
`now` passed in; proven by `tests/test_recommendation_topic_analysis.py`,
which runs the full pipeline with zero database); dependency arrow is strictly
tutor → recommendations, never the reverse; the impure remainder
(`repository.py`, `candidate_service.py`) reads the same
`user_data.user_coding_problems` / `user_coding_submissions` /
`catalog.coding_problems` tables with UUID user ids this design already
targets. Limits: single-shot top-3 output (no calendar, no curriculum),
hard-locked to one curated sheet (`catalog.dsa_sheet_problems`,
`('leetcode','default_dsa_sheet','v1')`), and zero company/interview-evidence
awareness — its `interview_frequency_score` is a static curated column, not
report data. §7.6 defines exactly what is reused and what is not.

**Worker precedent.** `api/main.py:61` starts a daemon thread running
`extension_sync/catalog_worker.py` over a pgmq queue (`events`) with retry +
archive semantics. The plan-generation worker copies this pattern.

**Frontend.** `MyPrepPanel` = `StreakWeekRow` (live) + `TodayTasks` (mock
local state) + `TrendingQuestions` (hardcoded, blurred). A complete
API-backed plan surface exists at `/prep` (readiness ring, phase ribbon,
Days/Topics toggle, evidence chips, cohort disclosure) pointed at the
unmounted legacy API. `web/lib/api.ts` still carries the full legacy prep
client (`PrepTask`, `TodayResponse`, `startPrepPlan`, …).

---

## 2. Architecture overview

```text
                       ┌────────────────────────────────────────────┐
 user message ────────▶│ chat_agent_v2 pipeline (PR #22)            │
                       │  selector offers planner tools:            │
                       │   read:  get_plan_window, list_user_plans, │
                       │          preview_plan_profile              │
                       │   write: create_plan  (mutation lane, §4)  │
                       └───────────────┬────────────────────────────┘
                                       │ create_plan: normalise → insert
                                       │ prep_plans(status=generating)
                                       │ → enqueue pgmq 'plan_jobs' → return ack
                                       ▼
                       ┌────────────────────────────────────────────┐
                       │ Plan worker (daemon thread, pgmq consumer) │
                       │  1. evidence profile   (signal tables)     │
                       │  2. user signal load   (LC history, skill; │
                       │     PR #18 topic/user-state analysis)      │
                       │  3. curriculum blueprint (deterministic)   │
                       │  4. candidate ranking  (evidence score ×   │
                       │     PR #18 pedagogical score, §7.6)        │
                       │  5. 3-day materialisation + LLM theming    │
                       │  6. validation (retry ≤2) → persist version│
                       │  7. status → ready | failed | needs_input  │
                       └───────────────┬────────────────────────────┘
                                       ▼
   ┌───────────────────────────┐   ┌──────────────────────────────────┐
   │ Snapshot publisher (cron/ │   │ Postgres (app DB)                │
   │ Dagster): silver → app DB │──▶│ prep_plans / prep_plan_versions  │
   │ company_interview_signal, │   │ prep_plan_tasks / prep_plan_events│
   │ canonical_items, samples  │   │ user_skill_state                 │
   └───────────────────────────┘   └───────────────┬──────────────────┘
                                                   ▼
                       ┌────────────────────────────────────────────┐
                       │ REST /api/plans/* (window, actions, edits) │
                       │  → MyPrepPanel rail + curriculum overview  │
                       └────────────────────────────────────────────┘
```

Three planes, matching handoff §11:

- **Layer A — evidence profile**: computed by the worker from snapshot tables
  (§5); persisted per version in `evidence_profile_jsonb`.
- **Layer B — curriculum blueprint**: full-duration skills/topics/phases plan;
  persisted in `curriculum_blueprint_jsonb`. Never a day-by-day question list.
- **Layer C — execution window**: exact tasks for today + 2 days in
  `prep_plan_tasks`; regenerated forward, never backward.

---

## 3. Normalised plan request, routing, and the under-specified-request problem

### 3.1 `PlanRequest`

The handoff §5 schema is adopted as-is, with one structural addition: every
resolvable field carries a **resolution provenance** so downstream layers and
the UI can distinguish what the user said from what we guessed.

```json
{
  "scope": "company_all_rounds | company_specific_round | general_subject | mixed_focus",
  "targets": [{ "type": "company", "company": "google", "role_family": "backend",
                "level": "L4", "location": null, "weight": 0.4 }],
  "rounds": [], "subjects": [],
  "duration_days": 30, "daily_budget_minutes": 120,
  "available_days": ["mon","tue","wed","thu","fri","sat"],
  "experience_level": "intermediate",
  "preferences": { "include_mocks": true, "include_revision": true,
                   "unseen_questions_only": false, "allow_extra_practice": false,
                   "theory_only": false, "questions_only": false },
  "resolution": {
    "role_family": "assumed:user_profile", 
    "level": "assumed:leetcode_calibration",
    "duration_days": "default",
    "daily_budget_minutes": "stated"
  }
}
```

Provenance values: `stated` (explicit in the message), `inherited`
(carried from conversation context / active plan), `assumed:<source>`
(profile, LeetCode calibration, company-median default), `default`.
Assumed fields render as editable chips in the plan header ("Assumed: L4 —
change"), which converts silent guessing into a one-tap correction loop
instead of a `needs_input` interrogation.

Normalisation is LLM-assisted but rule-checked: the `create_plan` tool's
LLM-facing parameter schema is intentionally loose (free-text companies,
subjects, durations), and the handler runs deterministic normalisation on top
— company resolution against `silver.company`/aliases (already the pattern in
`chat_agent_v2/entities.py`), level canonicalisation via the ported
`prep_priors.canonicalize_level`, date/duration resolution reusing the
server-authoritative date logic style of `chat_agent_v2/arguments.py`. The
LLM never gets to invent canonical values; it only extracts.

### 3.2 Deterministic routing

Handoff §4 rules adopted verbatim:

```text
company + round        → company_specific_round
company, no round      → company_all_rounds
no company, subject    → general_subject
companies + round      → that round across all targets
companies, no round    → merged company_all_rounds
company + focus area   → mixed_focus (all rounds, weighted)
```

One addition: **"focus on X" never silently drops other rounds** (handoff
§3.4), and conversely a round-specific plan **never silently adds other
rounds** (§3.2). The router emits `include_remaining_rounds` and
`round_focus` weights explicitly so the blueprint step cannot reinterpret.

### 3.3 The under-specified-request matrix

This is the core of the "think around the corners" requirement. The principle:
**never block plan creation on a question we can answer with a labelled
assumption; ask at most one consolidated question, and only when the answer
changes the plan's shape fundamentally.**

| User gave | Missing | Resolution |
|---|---|---|
| Company only ("prepare me for Google") | role, level, duration, budget | Role: `app.user_profile_summary.job_role` → role_family map; else `backend` (`assumed`). Level: profile → else LeetCode calibration (§6.2) → else company median (`prep_priors` map: google→L4, meta→E5, amazon→SDE2…). Duration: 30 d default. Budget: 90 min default (120 if `experience_level=advanced`). Plan generates immediately; header shows the assumption chips. |
| 5–6 company names, nothing else ("Google, Meta, Amazon, Microsoft, Uber") | everything else | Same as above per company, merged (§7). Company weights default to equal; if the user's profile has a `primary_target_id`, that company gets 1.5×. Duration default rises to 60 d (multi-company). If more than 6 companies: keep the 6 with the most evidence, tell the user which were parked and why ("limited evidence for X; I focused on…"). |
| Topic only ("master dynamic programming", "teach me system design") | company | `general_subject`. Pedagogical ordering from the curriculum graph (§7.2), global corpus frequency as the ranking signal, **zero company-specific claims** (validator rule V-10 rejects any company-evidence field on a general_subject task). Companies later mentioned in the same session do NOT retroactively attach — a new request is required. |
| Round only, no company ("prepare me for system design interviews") | company | `general_subject` with `subjects=[system_design]` — treated as subject mastery, not a company round plan. Round-structure content (timing, evaluation signals) uses global patterns and is labelled as such. |
| Company + topic that is not a round ("Google + dynamic programming") | — | `company_specific_round` is wrong (DP is not a round). Route to `mixed_focus`: company_all_rounds with `round_focus={dsa: high}` and a topic emphasis `subjects=[dynamic_programming]` inside the DSA allocation. |
| No duration, but a date ("my Amazon interview is on March 3") | duration | `duration_days = target_date − today`, clamped (§3.4). Past date → treat as needs_input with one question ("that date has passed — new date, or general practice?"). |
| Duration but absurd ("prepare me for Google in 2 days") | — | Generate a **sprint plan**: no foundations phase, highest-frequency + revision + one mock; header states plainly this is triage, not preparation. Never refuse. |
| Very long ("for one year") | — | Cap materialisation exactly as any plan (3-day window); blueprint stores quarter-level phases; duration capped at 365 d. |
| Weekends only / "90 min on weekdays" | — | `available_days` + per-day budget honoured by the scheduler (already a solved problem in `prep_skeleton`'s hour-budget model — extend it with a per-day map). |
| "Only theory" / "only questions" / "unseen only" | — | Preference flags; unseen-only cross-checks LeetCode history via the bridge (§6). |
| Company with almost no evidence ("prepare me for a seed-stage startup") | evidence | Fallback ladder (§5.4) bottoms out at `fallback_profile` — a role/level-generic plan, labelled honestly ("No CrackedIn evidence for X; this is a strong general plan for backend mid-level"). Never fabricate company claims (validator V-10). |
| Non-interview subject ("plan my wedding") | — | The selector simply doesn't call the planner tool (its description is scoped to interview/tech preparation); if called anyway, the normaliser returns a structured `unsupported_subject` and the selector explains. |
| Request identical to an active plan | — | Idempotency (§4.1, §8): `create_plan` returns the existing plan reference instead of a duplicate; the chat reply says the plan already exists and offers the window. |
| User already has N active plans | — | Multi-plan is supported (legacy model already was). At 3+ active plans, the ack nudges consolidation ("want me to merge these into one multi-company plan?") but does not block. |

`needs_input` (handoff §15) is reserved for exactly two cases: past interview
date, and a normalisation that produced zero resolvable targets *and* zero
subjects. Everything else resolves with labelled assumptions.

---

## 4. Chat integration: exposing the planner as tool calls

### 4.1 Two lanes, matching PR #22's actual invariants

**Read lane (registered in the normal v2 registry — legal today):**

| Tool | Purpose | Notes |
|---|---|---|
| `get_plan_window` | Active plan's today + next 2 days, with progress. Powers "what should I do today?", "how am I doing?" | read-only, `grounding=False` |
| `list_user_plans` | Plan switcher / "what plans do I have?" | read-only |
| `preview_plan_profile` | Fast evidence preview from the signal tables (round mix, top topics, confidence) for a company/level — lets the selector answer "what does a Google L4 loop look like?" and *tees up* plan creation | read-only; this is also the internal first step of generation, so it is cheap by construction |

**Mutation lane (v1: `create_plan` only):**

`ToolRegistry` hard-raises on `read_only=False`. Do **not** weaken that
invariant globally. Instead:

1. `ToolDefinition` keeps `read_only`; a new factory
   `build_launch_registry(user: AuthedUser | None)` composes
   `LAUNCH_TOOLS + PLANNER_READ_TOOLS`, and appends `PLANNER_MUTATION_TOOLS`
   **only when** the caller is a fully authenticated (non-anonymous) user.
   The registry class gains an explicit
   `ToolRegistry(defs, allow_mutations=False)` constructor flag; the guard
   stays the default and the planner factory is the single site that opts in.
   The existing test pinning the read-only invariant is updated to pin the
   *default*, plus a new test pinning that anonymous users can never see a
   mutation tool.
2. **Identity threading (required change):** add `user_id: str | None` and
   `session_id: int | None` to `ToolContext`
   (`chat_agent_v2/tools/contracts.py:21`), populate them in the
   `load_executor()` closure in `pipeline.py:83-89` (both values are already
   lexically in scope there), and have every planner handler refuse
   (`{"status": "auth_required"}`) when `user_id` is missing. This is the one
   genuinely invasive edit to PR #22, and it is small.
3. Mutation handlers enforce, in order: auth present → ownership → input
   validation → **idempotency** (`request_fingerprint =
   sha256(normalised_request)`, unique per user across *live* plans only —
   §8) → rate limit (≤3 plan creations/hour/user) → enqueue → audit event
   (`prep_plan_events`).

`create_plan` itself does no generation: normalise (§3), insert
`prep_plans(status='generating')` + `prep_plan_events(plan_created)`, enqueue
`plan_jobs`, return:

```json
{ "status": "ok", "plan_id": 123, "plan_status": "generating",
  "title": "Google + Meta · backend · 30 days",
  "scope": "company_all_rounds",
  "targets": [{"company": "google", "level": "L4", "confidence": "high"},
              {"company": "meta", "level": "E5", "confidence": "moderate"}],
  "assumptions": [{"field": "level", "value": "L4", "source": "leetcode_calibration"}],
  "eta_seconds": 30 }
```

Chat-driven *editing* tools (`edit_plan_day`, `replace_plan_task`,
`pause_plan`, "make tomorrow lighter", …) are **v2 of the chat surface**
(handoff Phase 6). In v1 those actions live on REST endpoints driven by the
UI (§10), which keeps the v1 mutation lane to a single, easily-audited tool.

### 4.2 Fitting the pipeline's answer constraints (hazards found in the audit)

- **Selector reply cap.** A planner turn typically ends as a selector direct
  reply (400-token cap, "under 120 words"). That is *fine* — by design the
  chat reply is a short ack ("Building your 30-day Google+Meta plan — DSA-heavy
  for L4, system-design for E5. It'll appear in your prep panel in ~30s.
  I assumed L4 from your LeetCode history — say the word to change it.").
  The plan itself is never prose in chat; it lives in the panel. This turns
  the constraint into the correct product behaviour.
- **Citation gate.** Planner tools set `grounding=False` and return no
  `interview_id` fields in v1, so `citation_required` answerer turns and the
  `[source:N]` machinery are never triggered by planner results. (Later, when
  "why this task?" surfaces sampled interviews in chat, evidence rows will
  carry `interview_id` and ride the existing `citation_ref` mechanism as
  designed — that is additive.)
- **Status verbs.** Add entries to `stages/selector.py::_TOOL_VERB`
  (`create_plan → "Building your plan"`, `get_plan_window → "Checking your
  plan"`, …) — otherwise completions render as "Found N reports".
- **`_result_count`** (`tools/executor.py:118`): add planner names so
  telemetry counts are meaningful.
- **Argument inheritance:** add `preview_plan_profile` to the follow-up
  inheritance set in `arguments.py:233` (company/role/level carry across
  "what about Meta?" turns); **exclude** `create_plan` from inheritance —
  plan creation must always be explicit, never inherited from a stale filter.
- **Goldens:** regenerate `launch_tools.json` and the selector-system golden;
  update the six-name registry assertion in `tests/test_chat_agent_v2.py:151`.
- **Selector prompt:** one added rule block via the supported
  `PromptAssembler(extra_blocks=…)` hook: create_plan requires explicit
  planning intent ("prepare me", "make me a plan", "I have an interview on…"),
  never fires on informational questions (those go to `preview_plan_profile`
  or the existing evidence tools), and must pass through user-stated
  constraints verbatim.

### 4.3 Plan-ready notification

The chat turn ends before generation finishes. V1 closes the loop without new
socket infrastructure:

- `GET /api/plans/{id}` is polled by the prep panel while
  `status=generating` (3 s interval, capped 2 min); the panel swaps a
  shimmer for the window when `ready` and shows a retry affordance on
  `failed` / a question card on `needs_input`.
- The existing `window CustomEvent("chat:turn-complete")` (already emitted by
  `chat-area.tsx`) triggers an immediate panel refresh so the generating state
  appears the moment the ack lands.
- V2 option: a `plan_status` SSE frame pushed into an open chat stream, or a
  one-shot notification row — not needed for launch.

---

## 5. Evidence layer

### 5.1 Snapshot publication (silver → app DB)

The API process keeps a **single database dependency**. A scheduled publisher
(Dagster job in `orchestration/`, where ingestion already runs; cron
fallback) computes aggregates in the ingest Postgres and upserts three
compact snapshot tables into the app Postgres, stamped with a
`data_snapshot_version`:

**`evidence.company_interview_signal`** — handoff §7's projection, verbatim
columns (`company_id, role_family, level_canonical, location, round_type,
topic_id, item_id, time_bucket, interview_count, distinct_source_count,
weighted_frequency, offer_frequency, rejection_frequency, trend_score,
last_seen_at, confidence_score, data_snapshot_version`). Row count is
bounded (companies × roles × levels × rounds × top-K items), thousands not
millions.

**`evidence.canonical_items`** — the gold-lite catalogue (§5.3): one row per
canonical question/prompt with `item_id, modality, title, skills[],
difficulty, prerequisites[], estimated_minutes, external_slug,
leetcode_number`.

**`evidence.interview_samples`** — 20–30 stratified representative
experiences per company cohort (handoff §6.2: stratified across recency,
level, role, outcome, round type, source; deduped via
`cross_source_fingerprint` / `possible_duplicate_of`), stored as compact
JSON (round ordering, real question wording, follow-ups, interview_id for
citation hydration).

Recency weighting reuses the ported `prep_weighting.post_weight_from_date`
half-life math (SQL regenerated for Postgres — the module was built for
exactly this). Confidence scoring per cohort cell:
`confidence = f(interview_count, distinct_source_count, recency_mass,
agreement)` mapped to the handoff's labels
`high | moderate | limited | fallback`.

### 5.2 Why snapshot tables (and not live silver queries)

- Generation reads become single-store, indexable, and cheap — the worker
  never blocks on the ingest DB being up.
- `data_snapshot_version` lands naturally on `prep_plan_versions`, giving the
  handoff's reproducibility requirement (§22) for free.
- The publisher is the one place that pays canonicalisation cost.

### 5.3 Gold-lite canonicalisation

Full gold (ANN → cross-encoder → LLM → HDBSCAN) is deferred; v1 ships the
20 % that yields 80 %:

1. **LeetCode-referenced items** (the majority of DSA evidence):
   `assessment_item_occurrence.attributes.external_refs` and prompt slugs
   resolve to `lc:<slug>` canonical IDs — exact, free, and they double as the
   **user-history bridge** (`lc:<slug>` ⇄ `catalog.coding_problems(platform,
   slug)` ⇄ legacy `problems.leetcode_number` via `display_id`). The bridge
   table `evidence.problem_bridge(slug, leetcode_number, display_id)` is
   prerequisite P1-1 in the roadmap.
2. **Exact-hash grouping** for everything else: `prompt_hash` already exists;
   occurrences sharing a hash collapse into one item.
3. **Curated seed list for the head of the distribution**: the ~200 highest
   frequency system-design/LLD/behavioural prompts get hand-curated canonical
   IDs (`sd:url-shortener`, `lld:parking-lot`, `bhvr:conflict`) with alias
   lists — a one-time, high-leverage editorial task; the publisher matches by
   normalised alias before falling back to raw hash groups.
4. Every canonical item carries `skills[]` (§7.1 taxonomy) — for `lc:` items,
   seeded from `catalog.coding_problems.topic_tags` mapped through the ported
   `DSA_TOPICS` bucketing; for curated items, hand-assigned.

The interface (`item_id`, `skills[]`) is exactly what full gold will provide,
so swapping the publisher's matcher later is invisible to the planner.

### 5.4 Cohort fallback ladder

Ported from `prep_data.resolve_cohort`, re-expressed over the signal table:

```text
company+level+role → company+level → company → tier/role/level generic → fallback_profile
```

Each step lowers the prior-blend α (legacy values 0.70/0.45/0.25/0.10 are the
starting point) and the surfaced confidence label. The plan header and every
company-specific task carry the label; validator rule V-10 blocks
company-specific claims on fallback-profile cohorts.

---

## 6. User signal layer

### 6.1 Inputs

- `user_data.user_coding_problems` (per-slug rollups: status, attempt/AC/WA
  counts, timestamps) and `user_data.user_coding_submissions` for recency.
- Prior CrackedIn progress: `prep_plan_events` + `user_skill_state` from any
  earlier plan.
- `app.user_profile_summary` (role, YoE, strongest/weakest domains) when
  present.

### 6.2 Cold-start skill state (an improvement over the handoff)

At plan creation the worker **seeds `user_skill_state` from LeetCode history**
via the bridge. This is no longer greenfield: the seeding computation *is*
PR #18's per-topic analysis (§7.6), run over the planner's skill taxonomy
instead of sheet topics. Per skill:

- **weakness** from `topic_analysis.py`'s formula —
  `0.35·failure_ratio + 0.25·repeated_failure + 0.25·unsolved_attempted +
  0.15·accuracy_gap` — with its **evidence tiers** (high/medium/low/none by
  attempted- and submission-count floors) clamping the score when data is
  thin, so a single bad submission can never brand a skill "weak";
- **mastery** as the complement of weakness, blended with solved-problem
  coverage weighted by difficulty and recency-decayed (a problem solved 14
  months ago contributes little);
- `user_skill_state.confidence` recording the evidence tier, and
  `source='seeded_leetcode'`.

This replaces the handoff's implicit cold start (everyone begins unknown) and
directly powers:

- **already-solved policy** — three-way, not boolean, implemented with PR #18
  scoring components: recent clean AC (≤90 d, low attempts) → excluded from
  fresh scheduling, eligible for revision slots via `revision_due_score`
  (step-decay on days since last AC: 0 → 0.25 → 0.5 → 0.75 → 1.0 at
  7/21/45/90 d); old AC (>90 d) → schedulable as a timed re-solve labelled
  "refresher"; attempted-but-failed → scheduled *earlier* via
  `failed_attempt_score`, whose `near_solve_score` input (share of testcases
  passed on the latest submission) distinguishes "almost had it — retry soon"
  from "fundamentally stuck — on-ramp with an easier problem first".
- **level calibration** when the user stated no level (§3.3): PR #18's
  maturity ladder (`new_user`/`starter`/`beginner`/`regular`/`advanced` by
  solve count) is the deterministic input to the level guess, and its
  `DIFFICULTY_MATCH` table (§7.6) bounds what difficulty the first window may
  contain regardless of the stated level.
- **day-0 diagnostics**: the first day of any plan ≥14 days includes 2–3
  short probe tasks targeting the highest-uncertainty skills (evidence tier
  low/none), so the first regeneration corrects the assumed level — cheap
  adaptive value on day one, no mastery model required.

Seeding covers **coding-track skills only** (§7.6 scope limit): system-design
and behavioural skills have no external history to seed from, so they start
at the neutral prior with evidence tier `none` — which automatically routes
day-0 diagnostic probes toward them.

If no LeetCode account is synced, everything above degrades to priors and the
ack nudges: "Connect LeetCode and I'll skip what you've already solved."

---

## 7. Planning core

### 7.1 Skill taxonomy

One table, `evidence.skills` (`skill_id, track, label, prerequisites[]`),
seeded from the ported `DSA_TOPICS` (26 DSA buckets, already the shared
vocabulary with the live `/api/practice` surface) plus curated
system-design/LLD/behavioural skill lists. Prerequisite edges encode the
pedagogical orderings that today exist only as prose in the legacy LLM prompt
(graphs: representation → BFS/DFS → components → topo-sort → shortest paths →
union-find; DP after recursion; sharding after databases…). Prerequisites are
**enforced by the scheduler and validator**, not requested from the LLM —
this is the single biggest quality upgrade over the legacy generator.

### 7.2 Curriculum blueprint (deterministic)

Port of `prep_skeleton`'s budget model, generalised:

- Round allocation from the evidence profile (per company round mix ×
  request weights), replacing the legacy hardcoded phase split with
  evidence-derived phases (`foundations → patterns → depth → mocks/revision`
  with boundaries computed from duration and gap sizes).
- Skill objectives per phase with target mastery and allocated minutes;
  per-day budget honours `available_days` and per-day minute overrides.
- **Interleaving**: within a phase, two active skills alternate rather than
  serial blocks (better retention, and it de-risks single-topic fatigue —
  the legacy skeleton's greedy "highest remaining hours" produced monotone
  stretches).
- Revision cadence and mock placement per duration (legacy constants are the
  starting point: cadence none/10/7 by 30/90/180 d).
- Multi-company merge per handoff §9: per-company profiles → common-core vs
  company-specific split (skills shared by ≥2 targets get the coverage
  bonus) → capacity reservation derived from actual overlap.

Priority scoring is deterministic and unit-testable (handoff §9.3 signals
with concrete sources). It is deliberately **three separate stages — gates,
scores, ordering — not one formula**: a single expression that mixes hard
constraints, unbounded evidence sums, multiplicative factors, and absolute
subtraction is impossible to reason about (any zero factor silently
annihilates an item; unbounded corpus mass drowns bounded pedagogy; a
subtraction can push scores negative).

**Stage 1 — hard gates (boolean, evaluated first, each gate named in the
task breakdown when it fires):** the §7.3 guardrails (difficulty
categorically outside the maturity range, anti-frustration, cooldown,
solved-only-as-revision); the prerequisite gate (an item whose prerequisite
skill sits below threshold mastery is ineligible except as part of that
skill's on-ramp); preference gates (`theory_only`, `questions_only`,
`unseen_questions_only`). Gates are never expressed as score factors —
an excluded item is excluded for a stated reason, not scored to zero.

**Stage 2 — two independent scores, each normalised to [0,1]:**

```text
evidence_score(item) =
    weighted_mean over targets [
        target_weight ×
        (weighted_frequency / max weighted_frequency in that target cohort)
        × round_importance × trend        # bounded multipliers, 0.5–1.2
    ]
    × coverage_bonus                       # 1.0–1.25, shared by ≥2 targets
    × confidence_damping                   # 1.0 high / 0.9 moderate /
                                           # 0.8 limited — applied ONLY when
                                           # cohorts of different confidence
                                           # compete in one candidate pool
```

Per-cohort max-normalisation makes the top-asked item score 1.0 whether the
cohort has 40 reports or 4,000, so evidence and pedagogy live on the same
scale. Confidence is otherwise carried as a *label*, not a multiplier — a
uniformly low-confidence cohort should change what the UI says, not
flatten the ordering.

```text
pedagogical_score(item) =                  # ported PR #18 scorer, §7.6,
    0.26 × topic_weakness                  # re-weighted after REMOVING its
  + 0.24 × failed_attempt (incl. near_solve)  # importance + interview_frequency
  + 0.20 × difficulty_match                # signals — both now live in the
  + 0.14 × revision_due                    # evidence half; keeping them would
  + 0.10 × progression                     # double-count frequency. Remaining
  + 0.06 × freshness                       # six re-normalised to sum 1.0.
```

The weights are starting values, pinned by golden tests (§7.6 F2) and tuned
only through those tests.

**Stage 3 — combination and total order:**

```text
priority(item) = evidence_score^0.6 × max(pedagogical_score, 0.15)^0.4
                 × repetition_multipliers   # recently-shown ×0.85,
                                            # same-skill-too-recently ×0.9 —
                                            # bounded multipliers, never
                                            # subtraction
```

- The geometric blend encodes "must matter to the target AND serve the
  user"; 0.6/0.4 puts evidence in the lead for company plans.
  `general_subject` plans invert the exponents (0.4/0.6) and read
  `evidence_score` from the global corpus instead of a company cohort.
- The **0.15 pedagogical floor** prevents mastery-annihilation: a fully
  mastered, company-critical item never vanishes — it survives at reduced
  priority and enters as a revision task (§6.2), which is the correct
  product behaviour for "you'll still be asked Two Sum".
- Total order is `(−priority, confidence_label, canonical item_id)` —
  fully deterministic; no tie is ever resolved by iteration order or
  randomness.

Every stage writes its intermediates into the per-task breakdown
(`prep_plan_tasks.evidence_jsonb`): fired gates by name, both halves' raw
signals, the blend — this is what "why this task?" renders and what golden
tests assert against.

### 7.3 Rolling execution window

Handoff §2 zones adopted verbatim: past days immutable; today + 2 days exact
and stable; days 4–7 soft-planned; beyond day 7 curriculum only.
Materialisation selects top-priority eligible candidates per day slot under
the minute budget, difficulty ramp (≤2 hard tasks consecutive), and modality
mix from the blueprint.

Eligibility and day composition adopt PR #18's hard-filter + soft-rerank
split (§7.6):

- **Guardrails (hard, pre-scoring):** categorical difficulty exclusion via
  the maturity table (a `hard` problem is impossible for a `new_user`, not
  merely down-ranked); anti-frustration (drop a retry candidate once
  `failed_attempt_count > 5` with `near_solve_score < 0.5` — stop
  re-scheduling what keeps hurting); solved problems eligible only as
  revision tasks; repeat-memory (an item scheduled-and-skipped recently is
  excluded for a cooldown period — the planner analogue of PR #18's 90-day
  `previously_recommended` window, but keyed on *outcome*, see §7.6 fix F3);
  per-day caps per task type (at most 1 retry, 1 revision per day by
  default).
- **Diversity (soft, post-scoring):** within a day, cap same-skill tasks at
  2 and avoid a single-difficulty day — direct ports of PR #18's
  `TOP_5_TOPIC_CAP` and `_avoid_same_difficulty_top5` passes, applied to the
  day slate instead of a top-5 list.

### 7.4 LLM responsibilities (narrow, per handoff §12)

The LLM: (a) request extraction (§3.1), (b) choosing among similarly-ranked
candidates within a day, (c) day themes/intros and per-task notes, (d)
user-facing explanations. Structured output via response-schema JSON with the
legacy validate/retry loop (`MAX_RETRIES=2`, errors formatted back via the
ported `format_errors_for_retry`). The LLM never invents refs — the validator
holds the candidate pool. Provider: the plan worker is **not** bound to the
chat slots; it gets its own model slot config (`PLANNER_MODEL` /
`PLANNER_FALLBACK_MODEL`) reusing the `llm/` adapter layer from PR #22, since
generation runs off-turn where latency tolerance is high.

### 7.5 Validation

Port `prep_validator` (error-code vocabulary, budget tolerance 15 %, day
bounds) and extend with the handoff §21 rules: prerequisite violation,
equivalent-canonical-repeat, available-days violation, locked-task overwrite,
company-claim-without-evidence (V-10), wrong modality in round-specific
plans, revision-before-exposure. Machine-readable errors drive targeted
retry; two failures → `status=failed` + event, never a partial plan.

### 7.6 Pedagogical scoring: reusing the PR #18 recommendation engine

The planner's evidence layer answers *what the target companies ask*; PR
#18's engine answers *what this user pedagogically needs next*. They are
complementary by construction — the engine has zero company awareness, and
the evidence layer has zero per-user mastery modelling — so the integration
is a composition, not a merge.

**What is ported (as-is, into `api/services/planner/pedagogy/`):** the pure
module slice — `scoring_components.py`, `scorer.py`, `classifiers.py`,
`type_decider.py`, `topic_analysis.py`, `guardrails.py`, `diversity.py`,
`user_state.py`, the dataclass contracts from `models.py`, and `constants.py`
— all stdlib-only, I/O-free, taking `now` as a parameter, with no tutor
imports (verified: the dependency arrow is strictly tutor → recommendations).
PR #18's own `docs/recommendation_import_inventory.md` independently
identifies the same slice as the minimal reusable core.

**What is not ported:** `repository.py` and `candidate_service.py` (the
planner has its own data access, and the service binds to the curated-sheet
pool), `sheet_validator.py` (sheet-specific), and the seven candidate
generators as an *entry point* — the planner's candidate pool is the
evidence-derived `evidence.canonical_items`, not `catalog.dsa_sheet_problems`.
The generators' *conditions* (retry-ready, revision-stale, next-in-topic
progression, similar-reinforcement) survive as task-type selection rules
inside window materialisation.

**Scope limit — the engine is coding-track only.** Its signals presuppose
things only coding problems have: submission verdicts and testcase counts
(`near_solve_score`), an easy/medium/hard triple (`DIFFICULTY_MATCH`),
LeetCode topic tags, and a synced attempt history. None of that exists for
system-design, LLD, or behavioural tasks, so the ported components score
**only the coding portion** of a plan. Non-coding tracks use a reduced
pedagogical score built from what those tracks do have:

```text
pedagogical_score_noncoding(item) =
    0.40 × curriculum_position      # §7.1 prerequisite-graph progression
  + 0.35 × observed_skill_gap       # user_skill_state from plan-task events
                                    # (hinted/struggled/skipped — §12); starts
                                    # at the 0.5 neutral prior on a cold start
  + 0.25 × revision_due             # PR #18's staleness step-decay IS
                                    # reusable here — it is a pure function of
                                    # days, fed from prep_plan_events
                                    # completion dates instead of latest_ac_at
```

This means non-coding tracks are evidence-led and become user-adaptive only
as the user works through the plan — an honest statement of the available
signal, not a gap to paper over. The claim in earlier revisions that the
engine powers "the user-side half" generally is hereby narrowed: it powers
the user-side half *of the coding track*, which is where most per-user
signal exists anyway.

**How the two halves compose.** The worker builds one
`UserRecommendationState`-shaped object per generation from the same tables
PR #18 reads (`user_data.user_coding_problems`, `user_coding_submissions`)
plus `user_skill_state`, then evaluates the §7.2 formula: evidence half from
the signal tables, `pedagogical_need` from the ported components with the
curated-sheet static columns replaced by planner-side equivalents —
`importance_score` → normalised evidence `weighted_frequency`,
`interview_frequency_score` → the target-cohort frequency itself (real report
data instead of a hand-set constant), `progression_score`'s sheet-ordinal →
position in the §7.1 prerequisite graph. The per-signal weights are re-tuned
for the planner context and pinned by unit tests (see F2). Because both
halves are pure functions over dataclasses, the composed scorer keeps the
full per-task breakdown (`raw_scores`, `penalties`, `final_score`) that PR
#18 persists for explainability — this lands in
`prep_plan_tasks.evidence_jsonb` and powers "why this task?".

**What the engine contributes beyond scoring:**

- *Classifiers as adaptation vocabulary* (§12): maturity ladder, activity
  states (`inactive_user`, `high_activity_user`, `sync_stale_user`), and
  per-topic `need_type ∈ {weak, revision, neglected, progression}` become
  the planner's regeneration signals.
- *Sync-staleness honesty*: PR #18's `sync_stale_user` handling (penalise
  history-derived recommendation types when LeetCode sync is stale or
  failing) carries over — a window generated from stale history says so
  instead of confidently scheduling around a phantom state.
- *Determinism*: the engine has no randomness and total-orders every
  tie-break; combined with the planner's own determinism this keeps
  generation reproducible per `data_snapshot_version`.

**Fixes required when porting (found in audit, cheap):**

- **F1 — recency-count bug:** `_count_recent_attempts` sums a problem's
  *lifetime* `attempt_count` when only its latest attempt falls in the
  window, so one re-touched old problem can single-handedly trip
  `high_activity_user`. The port counts submissions in-window from
  `user_coding_submissions` instead.
- **F2 — unpinned weights:** PR #18 has no unit test on `scorer.py` /
  `scoring_components.py`; the exact weights and thresholds are unpinned.
  The port adds golden tests for every constant table (weights,
  `DIFFICULTY_MATCH`, staleness steps, weakness thresholds) before any
  re-tuning.
- **F3 — status-blind repeat memory:** PR #18's 90-day
  `previously_recommended` exclusion ignores row status, so merely *proposed*
  problems are blocked for 90 days. The planner's analogue keys cooldown on
  outcome events (skipped/replaced), never on having been scheduled.
- **F4 — packaging:** the package ships without `__init__.py`; the ported
  copy gets one.
- **F5 — shared constant:** the 0.65 "weak" threshold is duplicated in two
  modules; the port hoists it into the constants module.

**Coordination note.** PR #18 is unmerged (`mergeable_state: dirty`) and the
tutor will keep using the same engine. To avoid divergence, the port should
land as a shared package (`api/services/recommendations/` stays canonical;
the planner imports the pure slice rather than copying it) once PR #18's
rebase lands — until then the planner vendors the slice under
`planner/pedagogy/` with a provenance header naming the source commit
(`0813e46`). Either way, tutor and planner then share one definition of
"what this user is weak at". This needs agreement with PR #18's author
(open question §16.6).

---

## 8. Storage model

Handoff §16 adopted with concrete types (all `app` schema, UUID user keys,
`TIMESTAMPTZ`). New tables — no collision with, and no dependency on, the
legacy deferred `app.user_prep_*` tables:

```sql
prep_plans        (id BIGSERIAL PK, user_id UUID → public.users,
                   plan_kind TEXT, title TEXT, status TEXT
                     CHECK (status IN ('draft','analysing','generating','ready',
                                       'needs_input','failed','paused',
                                       'completed','archived')),
                   timezone TEXT, start_date DATE, target_date DATE,
                   daily_budget_minutes INT, available_days JSONB,
                   active_version_id BIGINT,   -- no FK: see versioning notes
                   request_fingerprint TEXT NOT NULL,
                   created_at, updated_at)
-- idempotency is scoped to LIVE plans only (an archived plan must not
-- swallow a fresh identical request forever):
CREATE UNIQUE INDEX prep_plans_live_fingerprint
  ON prep_plans (user_id, request_fingerprint)
  WHERE status NOT IN ('archived','completed','failed');

prep_plan_versions(id, plan_id FK, version INT,
                   normalised_request_jsonb JSONB,
                   evidence_profile_jsonb JSONB,
                   curriculum_blueprint_jsonb JSONB,
                   data_snapshot_version TEXT, planner_version TEXT,
                   model_version TEXT, generation_reason TEXT,
                   validation_report_jsonb JSONB, created_at,
                   UNIQUE (plan_id, version))          -- immutable rows

prep_plan_tasks   (id, plan_id FK, version_id FK,
                   lineage_id UUID NOT NULL,  -- STABLE across versions:
                                              -- copy-forward preserves it,
                                              -- fresh tasks mint a new one
                   scheduled_date DATE, position INT,
                   item_id TEXT,              -- canonical or read:/concept:/mock:/revision:
                   task_type TEXT, skill_ids TEXT[],
                   estimated_minutes INT, difficulty TEXT,
                   status TEXT CHECK (status IN ('pending','done','skipped')),
                   outcome_flags TEXT[] DEFAULT '{}',  -- 'hinted','struggled',
                                              -- 'completed_external' — signals
                                              -- about HOW, not lifecycle states
                   locked BOOL DEFAULT FALSE, user_added BOOL DEFAULT FALSE,
                   selection_reason TEXT, evidence_jsonb JSONB,
                   user_override JSONB, created_at, updated_at,
                   UNIQUE (version_id, scheduled_date, item_id),
                   UNIQUE (version_id, lineage_id))

prep_plan_events  (id, plan_id FK, user_id UUID, event_type TEXT,
                   task_lineage_id UUID NULL,  -- survives regeneration; a row
                                               -- id would dangle as soon as a
                                               -- new version copies the task
                   payload_jsonb JSONB, created_at)
                   -- append-only; event vocabulary per handoff §16.4

user_skill_state  (user_id UUID, skill_id TEXT,
                   mastery_score REAL,        -- current blended value
                   confidence REAL,           -- evidence tier, §6.2
                   seed_mastery REAL, seed_confidence REAL, seeded_at,
                                              -- the LeetCode-derived seed is
                                              -- kept separately so observed
                                              -- learning never erases — and is
                                              -- always distinguishable from —
                                              -- the cold-start estimate
                   attempt_count INT, success_count INT,
                   hint_count INT, failure_count INT,
                   last_practised_at, next_revision_at, updated_at,
                   PRIMARY KEY (user_id, skill_id))
```

Versioning semantics — the invariants that make the schema self-consistent:

- **Task uniqueness is per version, not per plan.** Regeneration writes a
  complete new task set under the new `version_id` (future dates recomputed,
  past/completed/locked/user-added rows copied forward byte-identical with
  their `lineage_id` preserved), then flips `active_version_id` in the same
  transaction. A plan-wide unique key would make every regeneration violate
  itself; per-version keys make old versions immutable history and the
  active version the single serving surface.
- **Identity across versions is `lineage_id`.** Progress updates, events,
  and API task references address `(plan_id, lineage_id)` resolved through
  the active version — row ids are storage, lineage is identity. This is
  what lets "done" survive regeneration and lets events written last week
  still join to today's task rows.
- **`active_version_id` carries no FK** (it would be circular with
  `prep_plan_versions.plan_id`); integrity is enforced by the single
  transaction that inserts the version and flips the pointer, and a nightly
  invariant check (§13) asserts every pointer resolves.
- **Constraint authority is split explicitly:** the `prep_plans` row holds
  the *current* constraints (mutable via `POST /constraints`) and is what the
  next generation reads; `normalised_request_jsonb` on each version is the
  *frozen record* of what that generation actually used. They are expected
  to diverge between a constraints edit and the regeneration it enqueues.
- **Status vs signals:** `status` is lifecycle only (`pending → done |
  skipped`); `hinted`/`struggled` are `outcome_flags` — they describe how a
  task went, can coexist with `done`, and feed skill-state updates without
  corrupting completion semantics.

Progress lives **on the task rows** (denormalised current state) *and* in the
event log (authoritative history) — the legacy lesson that keying progress by
`(plan, day, ref)` blocks cross-plan knowledge is fixed by `user_skill_state`
keyed on the user, plus canonical `item_id`s that are stable across plans.

Concurrency: mutations carry `expected_plan_version`; a version bump between
read and write → 409, the client refetches.

---

## 9. Worker and job architecture

The **queue** copies the proven catalog-worker pattern (pgmq, visibility
timeout, retry-then-archive — `extension_sync/catalog_worker.py`). The
**topology does not**: the catalog worker is a daemon thread inside the API
process (`api/main.py:61`), which is fine for its lightweight event syncing
but wrong for plan generation — a 20–60 s LLM-bound job inside gunicorn
competes with request serving for CPU and memory, dies on every deploy and
worker recycle, and under multiple API replicas every process would start a
duplicate consumer thread it was never designed to be.

- **Dedicated worker process:** `python -m api.planner_worker`, same codebase
  and virtualenv, no HTTP surface, its own systemd unit
  (`interview-prep-planner-worker.service` alongside
  `deploy/interview-prep.service`). One instance is sufficient for MVP;
  scaling is "run more instances" because every job is safe under
  at-least-once delivery (below). API replicas never consume the queue.
- pgmq queue `plan_jobs`; messages `{plan_id, reason, requested_version}`.
  **Visibility timeout is set above p99 generation time** (5 min to start)
  so a live worker is never raced by redelivery; a crashed worker's job
  redelivers after the timeout.
- **Every job is idempotent and lock-guarded:** the job takes a Postgres
  advisory lock on `plan_id` (concurrent jobs for one plan cannot
  interleave), re-checks that its `requested_version` has not already been
  superseded (no-op if so — this also collapses duplicate enqueues), and
  writes its output — version row, task rows, `active_version_id` flip — in
  **one transaction**. A redelivered job after a crash therefore either
  finds the version already written and no-ops, or re-runs cleanly from
  nothing. ≥3 failed attempts → archive + `status=failed` +
  `generation_failed` event.
- **Graceful shutdown:** SIGTERM stops dequeuing and lets the in-flight job
  finish (bounded by the visibility timeout); a hard kill just means
  redelivery. Deploys restart the worker unit after the API unit.
- **Scheduled work has one owner, not one per replica:** the nightly
  per-timezone window roll is enqueued by a single scheduler — pg_cron in
  the app database (preferred: no new infrastructure) or the existing
  Dagster schedule — never by API-process cron. The roll job is idempotent
  via a `window_rolled_through DATE` column on `prep_plans`: re-enqueueing
  the same day no-ops.
- **Enqueue discipline on the API side:** plan creation enqueues exactly one
  `initial` job inside the same transaction that inserts the plan row
  (status-transition guarded); edit/constraint regenerations may enqueue
  freely because superseded-version no-ops make duplicates harmless.
- Job steps are §2's numbered list; each stamps progress into
  `prep_plan_events` so the UI can show "analysing evidence → building
  curriculum → selecting tasks". User edits during generation queue as
  events and are applied by an immediate follow-up regeneration rather than
  mid-job merging.
- Regeneration reasons: `initial`, `window_roll`, `user_edit`,
  `constraints_changed`, `completion_adaptation`, `manual_retry`.

---

## 10. REST API (UI-facing)

`api/routes/plans.py`, mounted (unlike its predecessor), Supabase-auth'd:

```text
POST /api/plans                       create (same normaliser as the tool; idempotency key honoured)
GET  /api/plans                       list (id, title, kind, status, targets, progress)
GET  /api/plans/{id}                  status + header + assumptions + confidence
GET  /api/plans/{id}/window           today + 2 days, tasks with evidence + selection_reason
GET  /api/plans/{id}/curriculum       blueprint view (phases, skills, coverage, confidence)
POST /api/plans/{id}/tasks/{tid}      {action: done|hinted|struggled|skipped|lock|unlock,
                                       time_spent_minutes?, note?}
POST /api/plans/{id}/tasks/{tid}/replace   {reason?} → same-skill alternative from pool
POST /api/plans/{id}/tasks/{tid}/move      {to_date}   (within window/soft zone)
POST /api/plans/{id}/constraints      {daily_budget_minutes?, available_days?, target_date?}
                                       → enqueue regeneration
POST /api/plans/{id}/pause | resume | archive
POST /api/plans/{id}/regenerate       {expected_plan_version} → enqueue window regen
```

Task completion also upserts `app.user_streak_windows` with a real
`completion_source='plan_task'` — replacing the `'placeholder'` source and
making the streak row in the same panel truthful.

---

## 11. Frontend integration

- **`TodayTasks` → API.** Replace `INITIAL_TASKS` local state with
  `GET /api/plans/{active}/window`. The mock `Task` shape maps cleanly:
  `label ← title`, `modality ← task_type`, `minutes ← estimated_minutes`,
  `done ← status`. Add the action menu (done / hinted / struggled / skip /
  replace / lock / "why this task?") — `selection_reason` + `evidence_jsonb`
  power "why this task?" with real counts ("asked in 18 Meta interviews ·
  covers graph traversal · shared across 3 targets").
- **Plan states in the rail.** `generating` → shimmer + stage line from
  events; `needs_input` → one question card; `failed` → retry;
  `ready` → window. Active-plan selection reuses the `useActivePlan` hook
  pattern (multi-plan aware, localStorage-persisted).
- **Curriculum overview.** Evolve `/prep` (whose Topics view, readiness ring,
  phase ribbon, evidence chips and cohort disclosure are exactly the right
  components) to consume `/api/plans/*`; the Days toggle stays retired.
  The rail keeps its deliberate no-full-plan-link stance; the overview is
  reachable from plan headers and chat, not as a default destination.
- **Chat ack rendering.** V1: plain text ack (selector direct reply).
  V1.5 option: a `plan` rich-block in `rich-blocks.tsx` rendering the ack as
  a card with an "open panel" affordance; the removed PR-22 `sources`-frame
  pattern is the precedent if a structured frame is preferred later.
- **Refresh loop.** Panel refreshes on `chat:turn-complete` (already
  emitted) + 3 s polling while `generating`.
- **`MyPrepPanel` parameterisation.** It currently takes no props; give it an
  optional `variant: "rail" | "page"` so mobile `/my-prep` and the desktop
  rail can diverge in density without forking.

---

## 12. Adaptation (v1-light) — what ships now vs later

**V1 (shipping):**

- Event capture for everything (`task_completed/hinted/struggled/skipped`,
  `task_replaced/moved`, `day_edited`, `constraints_changed`, timing) — the
  event log is the training data for v2, so it must be complete from day one.
- `user_skill_state` updates on task events with the handoff §18.3 rules
  (fail → down, hint → small up, independent → up, repeated fast success →
  ramp difficulty) — simple additive scoring, no forgetting-curve model yet.
  One planner-only enrichment over PR #18's inputs: plan task events carry
  `hinted`/`struggled`, which LeetCode history cannot see, so observed skill
  state becomes richer than seeded state from the first completed task.
- Window regeneration reads the PR #18 classifiers recomputed over fresh
  state (§7.6): a skill flipping to `need_type=weak` pulls a repair task
  into the next window; `revision`/`neglected` schedules a refresher;
  `progression` unlocks the next difficulty tier (gated by its
  `difficulty_progression_ready` failure-ratio checks); `sync_stale_user`
  down-weights history-derived task types until sync recovers.
- Window regeneration triggers: nightly roll, edits, constraint changes,
  "completed everything early" (pull one optional task forward — never
  auto-overload, per §18.1).
- Behind-plan v1: after 3 consecutive under-50 % days, surface the §18.2
  strategy chooser (reduce load / extend date / drop low-priority / keep
  deadline & prioritise) as a card; applying one = constraints change +
  regeneration. No silent compression (the legacy `compress`/`powerthrough`
  no-ops taught this lesson).
- LeetCode live sync: an AC on a scheduled item's slug arriving via
  extension sync marks the task done automatically (event
  `completed_external`).

**V2 (deliberately deferred):** mastery model with forgetting curves, the
3/7/21/60 revision ladder, difficulty auto-ramp, ahead-of-plan enrichment,
chat-driven plan editing tools, plan-ready push notifications, full gold
canonicalisation, semantic evidence retrieval (pending pgvector on silver).

---

## 13. Observability

Persist per version (handoff §22): planner/model/data-snapshot versions,
request hash, validation errors + retry count, generation duration, token
usage. Emit through the existing `obs` package. Launch-gate quality metrics:
unsupported-task rate (must be 0 by construction — validator), duplicate
canonical within window, budget violations, evidence coverage per
company-task, low-confidence-task %, generation p50/p95, plan-ready success
rate. Product metrics per handoff §22 land in `prep_plan_events` and are
queryable without new infrastructure.

---

## 14. Security and privacy

Three trust boundaries, each with its own threat model.

### 14.1 The mutation lane (LLM-decided writes)

The selector LLM chooses when `create_plan` fires and what arguments it
carries — which makes conversation content an input to a write path.
Anything the user (or pasted content) says can try to steer the tool call.
Defences, all deterministic and server-side:

- **Identity is never an argument.** No planner tool accepts a user id,
  plan owner, or session id from the LLM; identity comes exclusively from
  the authenticated `ToolContext` (§4.1). There is no argument the model
  could produce that operates on another user's data.
- **The normaliser is the trust boundary.** The LLM *extracts*; it never
  *authorises*. Every argument is re-validated by the §3.1 normaliser:
  companies resolve against the known-alias table or are dropped, durations
  clamp to [1, 365], daily budget to [15, 480] minutes, targets cap at 6,
  free-text never passes through to storage or prompts unchecked. Plan
  titles are composed server-side from normalised fields — never
  LLM-authored strings.
- **Cost ceilings as security controls:** ≤3 plan creations/hour and ≤10
  generation jobs/day per user (regenerations included) cap the LLM spend
  any single account — or any prompt-injection attempt — can trigger. The
  same limits apply to the REST endpoints (§10), which share the handler.
- **Full audit trail:** every mutation writes a `prep_plan_events` row with
  actor, origin (`chat` | `rest`), the normalised request, and the
  idempotency outcome — replayable per user.
- Anonymous-JWT users never see mutation tools (registry factory, §4.1);
  `create_plan` is excluded from argument inheritance (§4.2) so a stale
  conversational filter can never become an implicit write.

### 14.2 User-generated evidence (interview reports)

Corpus text is untrusted **twice**: as *input* to the generation LLM
(scraped or user-submitted reports can contain adversarial instructions)
and as *output* surfaced to other users (reports can contain PII).

- **Injection containment is structural, not prompt-polite.** Sampled
  evidence enters generation prompts inside delimited data blocks, but the
  real defence is §7.5: the validator confines LLM output to the
  deterministic candidate pool, so injected text cannot add tasks, links,
  or refs — at worst it wastes a retry. Day themes and task notes (the only
  free-text the LLM writes) are validated for length and rendered as plain
  text, never markdown-with-links, in the panel.
- **PII redaction happens at the boundary:** the snapshot publisher (§5.1)
  runs a redaction pass (names, emails, phone numbers, employee ids) before
  `evidence.interview_samples` leaves the ingest database. The app database
  never holds unredacted report text; aggregate signal tables hold no report
  text at all.
- **UI exposure is counts-first:** `evidence_jsonb` shown on tasks carries
  counts and canonical titles ("asked in 18 Meta interviews"), not verbatim
  quotes; quotes appear only through the citation path, which serves
  curated, redacted sample rows.

### 14.3 Storage and access

- Every REST route resolves plan/task ownership before acting and returns
  **404 (not 403)** on foreign ids — no existence oracle.
- All five planner tables get RLS policies (`user_id = auth.uid()`),
  matching the pattern PR #18's migrations already use — defence-in-depth
  even though the API connects with a service role, and a prerequisite for
  any future direct-from-frontend Supabase read.
- Plan, task, event, and skill-state rows **cascade-delete with the user
  account**. Skill state and plan events are per-user learning records:
  excluded from analytics exports unless aggregated across ≥ k users.
  Snapshot tables (`evidence.*`) contain no user data by construction.
- Observability (§13) logs evidence *ids* and score breakdowns, never raw
  report text or prompt bodies.

---

## 15. Implementation roadmap (repo-mapped)

The full design is intentionally larger than the first shippable increment.
Two scope controls make delivery tractable:

**M0 — walking skeleton (ships first, before any P-phase completes).**
One end-to-end vertical slice, deliberately narrow: a **single-company,
coding-round-only, 14-day plan**; evidence profile from a one-off aggregate
script over `silver` (the full publisher comes in P1 — M0 only needs one
cohort's numbers in a table); candidate pool from `evidence.problem_bridge`
+ `catalog.coding_problems`; the ported skeleton + validator; the dedicated
worker with a single `initial` job; three REST endpoints (create / window /
task-done); and **`TodayTasks` wired to the real window — the first thing
that replaces the prep panel's mock state with live data**. No chat, no
multi-company, no LLM theming (deterministic day titles), no adaptation
beyond marking done. M0 exists to force every risky seam — silver read,
bridge join, worker transaction, versioned tasks, UI contract — through real
data in week one, so the review's "static, not runtime-verified" caveat
stops being true at the earliest possible moment.

**Launch gate (v1) and explicit deferrals.** V1 launch =
M0 + P1 + P2 + P3-core + P4: single- and multi-company plans across coding +
system-design rounds, REST + rail UI, chat-driven creation. Explicitly
**deferred to v1.1** (designed, not built): the curriculum overview page
(§11 — the rail window alone is the launch surface), behind-plan strategy
cards (§12), mock-interview and behavioural task depth, the `plan`
rich-block, and `preview_plan_profile` for anonymous users. Everything in
the deferral list has its storage and events shipped in v1 so v1.1 is
additive.

**P1 — Foundations (unblocks everything)**
1. `evidence.problem_bridge` (slug ⇄ leetcode_number ⇄ display_id) + backfill.
2. Snapshot publisher: `company_interview_signal`, `canonical_items`
   (gold-lite matcher), `interview_samples`, `skills` — Dagster job in
   `orchestration/`.
3. Port pure modules (`prep_priors`, `prep_weighting`, `prep_skeleton`,
   `prep_validator`, `prep_topics`) into `api/services/planner/` with their
   tests; regenerate weighting SQL for Postgres.
4. Port the PR #18 pure slice into `planner/pedagogy/` with fixes F1–F5
   (§7.6): golden tests pinning every constant table, in-window submission
   counting, outcome-keyed repeat memory, `__init__.py`, hoisted weak
   threshold. Agree the shared-package plan with the PR #18 author.
5. Migrations for §8 tables.

**P2 — Generation core**
`PlanRequest` normaliser + routing (+ edge-case table §3.3 as parametrised
tests); evidence-profile reader; blueprint builder; candidate ranking;
window materialiser; LLM fill + validator; worker + pgmq queue; golden
cases for sparse/conflicting evidence and multi-company merge (handoff
Phase 7 list).

**P3 — API + frontend**
`routes/plans.py`; wire `TodayTasks`; plan states in the rail; task actions;
`/prep` → curriculum overview; streak source swap.

**P4 — Chat integration (PR #22)**
Identity threading into `ToolContext`; registry factory with the
authenticated mutation lane; the four tools; prompt block, status verbs,
goldens, tests. (Deliberately after P3 so the planner is testable via REST
before it is reachable via chat.)

**P5 — Adaptation v1-light + beta**
Event-driven skill updates, behind/ahead flows, LeetCode auto-complete,
metrics dashboards, small-beta rollout.

---

## 16. Open questions for the founder

1. Confirm D1–D5 (§0) — especially D2, since it reinterprets the handoff's
   "expanded plan surface" in favour of the repo's banned-concept stance.
2. Multi-plan posture: keep unlimited active plans with a nudge at 3+, or
   cap at N? (Design assumes nudge-not-block.)
3. Planner model slot: reuse the DeepSeek/Gemini adapter chain from PR #22
   config, or the legacy Gemini generator path? (Design assumes the PR #22
   adapter layer with dedicated `PLANNER_MODEL` env.)
4. Should `preview_plan_profile` be surfaced to anonymous users as a teaser
   (evidence profile visible, plan creation gated on login), or fully
   auth-gated? (Growth question, not an architecture one — both fit.)
5. The ~200-item curated canonical seed list (§5.3) needs an owner — it is
   editorial work, not engineering.
6. PR #18 sharing (§7.6): vendor the pure recommendation slice into the
   planner now and converge on a shared package after PR #18's rebase lands,
   or block P1-4 on landing PR #18 first? (Design assumes vendor-now,
   converge-later; needs sign-off from PR #18's author since both branches
   are unmerged and `dirty`.)

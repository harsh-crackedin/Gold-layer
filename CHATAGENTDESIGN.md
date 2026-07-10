# CrackedIn Chat Agent — Design Document

**Audience:** an engineer joining the project cold. Read this top to bottom and you will
understand what we are building, why the major decisions were made, and what to pick up first.
This document is the *why* and the *what*. The exact contracts, prompts, SQL, and per-task
acceptance criteria live in `CHAT-AGENT-IMPLEMENTATION-SPEC.md` (referenced throughout as
"the spec"); test fixtures, threat models, and live-provider evidence live in
`docs/chat-agent-hardening/`.

---

## 1. Summary

The CrackedIn chat agent is a **grounded interview-intelligence chat** over our silver corpus
of ~4,700 real, dated interview experiences (24k+ retrieval chunks). A user asks about a
company's interview process, questions, levels, or outcomes, and the agent answers from actual
sourced experiences — with receipts — rather than from the model's parametric memory.

Three product promises define the system, and each is enforced by server code, never by prompt:

1. **Every corpus claim is grounded and cited with receipts.** Any statement about our data is
   backed by a tool lookup, and every cited interview carries a verifiable receipt (link, date,
   source). Ungrounded corpus claims are structurally prevented, not merely discouraged.
2. **Zero canned replies — every reply is generated.** There is no template pool, no canned
   greeting, no canned acknowledgment. The only static string in the entire system is one
   infrastructure-error message shown when a model call hard-fails after retry.
3. **It feels fast.** Answers stream token-by-token; a status indicator fires within ~1.4s of a
   grounded turn; the trivial-turn fast path skips two model hops entirely.

**Launch scope:** a single chat surface. No UI modes, no mode buttons, no mock-interview flow,
no write-tools. Those are later tracks (§2).

**Scale target:** designed for **1,000 DAU at launch**, reaching **10,000 DAU with
configuration only** (uvicorn workers, connection-pool sizing, provider tier) — no
re-architecture. The turn engine is stateless per request: session state is read once at turn
start and written once at turn end, so horizontal scaling is a knob, not a rewrite (§12).

---

## 2. Goals, non-goals, constraints

**Goals.** Ground every corpus answer in real experiences; cite them with legal receipts;
generate every reply; stream fast; run cheaply enough to be viable at 1k DAU on a flash-tier
model; keep the architecture extensible (a new tool = a registry entry + goldens, nothing else).

**Non-goals at launch** (each is a specified later track, not scope creep):

- **Mentor write-tools** — proposing/committing study-plan changes with a confirm gate and taint
  rule. The ToolSpec annotations and a hard `NotImplementedError` guard ship now so no write tool
  can be added without meeting the gate; the gate itself is the mentor track.
- **Mock-interview FSM** — a stateful interview-simulation mode. Later track.
- **Weak-areas / prep-progress aggregation** — a nightly job feeding mentor context. Later track.

**Constraints.**

- **Model budget.** The primary model is flash-tier (`deepseek-v4-flash`) on both the selector
  and answerer slots. Frontier-everywhere is ~40× the cost at 1k DAU and is not on the table for
  launch; a Haiku 4.5 lane exists only as an escalation fallback (§6).
- **Receipts legality.** For paywalled / gated third-party content we may serve
  **link + date + paraphrase** only, and **never verbatim body text**. This is a legal
  boundary, enforced by data flow (`receipts_legal`) upstream of the model, not by prompt (§7, §8).
- **No UI buttons at launch.** Everything is conversational. Modes (when they arrive) enter via
  tools and exit via deterministic text in the gate; there are no clickable mode switches.

---

## 3. High-level design

### 3.1 Component diagram (reproduced from the spec §0.1)

```
┌────────────────────────────── CLIENT (web/) ───────────────────────────────┐
│  chat UI ── streamChatMessage: named-event SSE parser (§10) ── sources UI  │
│  (chips render ONLY by lookup against the `sources` event — §10)           │
└───────────────────────────────────┬─────────────────────────────────────────┘
                                    │ POST /api/chat/stream  (SSE)
┌───────────────────────────────────▼─────────────────────────────────────────┐
│ API  api/routes/chat.py — endpoints/CRUD kept, stream body = async for      │
│      over run_turn(ctx) → SSEEvent frames (§1 Entry-point contract)         │
│      chat_repository.begin_turn() ─── loads SessionState ──┐                │
│      chat_repository.finish_turn() ── ONE txn: msg+state+log┘ (§5)          │
└───────────────────────────────────┬─────────────────────────────────────────┘
┌───────────────────────────────────▼─────────────────────────────────────────┐
│ AGENT CORE  api/services/agent/                                              │
│                                                                              │
│  gate.py [0]      whitelist literals + normalize + ack offer-check          │
│     │             (>8k chars → condense-first: one no-tools answerer-slot    │
│     │              call → structured digest, wrap_ugc'd, persisted — §2[0]) │
│     │ trivial ──────────────► smalltalk.py  SmallTalkPlan                    │
│     │                          (1 no-tool LLM call, persona prefix,          │
│     ▼                           gated user-context, anti-repeat, streams)    │
│  dispatch() [1]   Plan factory — mode allowlists live here; ForcedToolCall/  │
│     │             ReducedToolLoop stubs = the router seam (§2[1] promotion)  │
│     ▼                                                                        │
│  loop.py [2–5]    ≤3 rounds; tool_choice required→auto→none                  │
│     │  ├ lexicon.py   is_corpus_referent (company terms + intent words)      │
│     │  ├ runtime guard: escape-hatch veto → re-run w/ hatches stripped       │
│     │  ├ §2[5] grounding invariant (pre-answer, deterministic)               │
│     │  └ repair.py    4-rung ladder → Haiku escalation                       │
│     ▼                                                                        │
│  executor.py +    validate args → resolve entities (company/level/          │
│  tools/*.py [3]    outcome/topics) → FIXED recipe → wrap_ugc() → result     │
│     ▼                                                                        │
│  citations.py [6] verifier (allowed-set membership) → sources hydration      │
│  history.py       cross-request reconstruction, elision stubs, digest subst. │
└──────────┬──────────────────────────────────┬────────────────────────────────┘
           │                                  │
┌──────────▼──────────────┐      ┌────────────▼──────────────────────────────┐
│ llm_client.py            │      │ DATA — one Supabase PG (via get_pg_pool)  │
│ slots (env): SELECTOR /  │      │  silver.company + level_resolution        │
│ ANSWERER = v4-flash,     │      │   (entity resolution + lexicon source)    │
│ ESCALATION = Haiku 4.5   │      │  silver.interview (pure-filter branch)    │
│ strict-beta, streaming,  │      │  search.interview_chunk (halfvec+HNSW     │
│ thinking:disabled ALWAYS │      │   +FTS, receipts_legal, interview_id)     │
└──────────────────────────┘      │  app.chat_sessions/messages (state+log)   │
                                  └───────────────────────────────────────────┘
Observability: Langfuse traces + §6 turn-log jsonb (= dashboards, golden growth,
and the future router's training set). Evals: §11 goldens + trivial suite +
bake-off + nightly canary, CI-blocking on agent/** and prompts.
```

### 3.2 The turn flow (reproduced from the spec §0.2)

```
prompt
  ↓
[0] WHITELIST GATE (exact literals, ~0ms, NO LLM)  → greeting/thanks/farewell/ack → SmallTalkPlan
  ↓ (everything else; bare ack after an offer falls through)
[1] dispatch(turn, session) → Plan           ← deterministic; modes/gate/future-router live here
  ↓
[2] SELECTOR LLM call — picks tool(s) + fills args (tool_choice="required" on 1st call; toolset incl. respond_directly)
  ↓
[3] SERVER: validate args → resolve entities → run the tool's FIXED recipe (SQL/vector/RRF)
  ↓
[4] loop to [2] for dependent calls (tool_choice="auto"), HARD CAP 3 rounds
  ↓
[5] ANSWERER LLM call — streams grounded answer over tool results
  ↓
[6] citation verifier → SSE sources → finish_turn() → log jsonb
```

### 3.3 The mental model

**The LLM chooses WHAT (which tool, which arguments); code decides HOW (which backend, which SQL,
which ranking). Every guarantee that matters is enforced by server code, never by the prompt.**
Tools split by user intent, never by backend. Entity resolution is server-side, never
model-side. Grounding is a code decision (lexicon + escape-hatch guard + a deterministic
pre-answer invariant + a citation verifier), not a model whim. The prompt is belt-and-suspenders;
the force is `tool_choice` and the executor.

---

## 4. The turn lifecycle, narrated

Consider a real prompt arriving mid-conversation:

> "thanks! now show me recent Google L5 system design interviews, and open the most detailed one"

**Gate [0].** `gate.py` normalizes and checks the message against exact-literal whitelists
(greeting/thanks/farewell/ack). The leading "thanks!" is *not* a whole-string match — the message
is a compound, and the whitelist matches whole strings only — so the gate falls through. This is
by design: a real question can never be swallowed by a stray pleasantry. The message is under the
8,000-char condense threshold, so it passes straight to dispatch.

**Dispatch [1].** `dispatch()` is deterministic. No gate hit, launch mode, so it returns an
`LLMToolLoop` plan carrying the launch toolset — the four core tools plus `respond_directly` —
with `first_call_tool_choice="required"` and `max_rounds=3`.

**Selector round 1 [2].** The first model call of the turn runs with `tool_choice="required"`
and `thinking:{"type":"disabled"}` (both mandatory — see §6). "Required" forces the model to pick
a tool; because every path into the answerer must first pass through a tool, an ungrounded corpus
answer can never stream out ahead of the server. The selector emits a `search_interviews` call
with `company:"Google"`, `level:"L5"`, `semantic_query:"system design"`, `sort:"recency"`. It fills
surface forms only — never ids, never level rungs.

**Validate / resolve / recipe [3].** `executor.py` validates the args against the strict JSON
schema, then resolves entities server-side: `"Google"` folds to the canonical company;
`"L5"` is looked up in the level-resolution cache keyed on the *resolved company's* canonical
fold (Google L5 → rank 5; the same code means a different rung at Amazon, which is why the model
never emits rungs). It then runs the tool's **fixed recipe** — a hybrid retrieval over
`search.interview_chunk`: filtered HNSW vector search + FTS, fused with RRF, collapsed to
interview grain, relevance-gated recency sort. Every UGC-derived field in the result is passed
through `wrap_ugc()` (datamarking) before it re-enters the model's context.

**Dependent round 2 [4].** The user also asked to "open the most detailed one." Under
`tool_choice="auto"` (mandatory reset after round 1), the selector issues a `get_interview` call
on one of the ids the search just returned. The server carries ids between calls — the model
never does. This second round counts against the hard cap of 3.

**Terminal completion = streamed answer [5].** With results in hand, the answerer call streams
the grounded reply. Because the selector and answerer default to the same model+provider, this is
simply the loop's terminal completion — **no extra model call** — plus a steering block reminding
it to cite facts and to be honest about empty results.

**Citation verifier + finish [6].** As the answer streams, it is scanned for `[iv_<uuid>]` tags.
The allowed set is exactly the ids this turn's tools returned, unioned with ids seen earlier this
session. Tags outside that set are stripped and logged (a fabricated or injected id fetches
nothing); malformed tags are stripped; valid tags become the `sources` SSE event, with receipts
hydrated server-side. Finally `finish_turn()` persists the assistant message, the session-state
side effects (seen-set, last-result ids), and the turn-log jsonb in **one transaction**.

### The trivial-turn path

A bare "hi" never reaches the selector. The gate matches a greeting literal and dispatch returns a
`SmallTalkPlan`, which is **one no-tool streamed model call** on the answerer slot with a stable
persona prefix. Cost: exactly one model call, no tool hops. Acknowledgments ("ok", "done",
"thanks") are handled with care: if the prior assistant message ended in an offer ("want me
to…?"), the ack is consent-to-action and *falls through to the selector*; otherwise it is a
`SmallTalkPlan`. And when an ack arrives after substantive work this session, the reply becomes a
**mentor-moment recap** — grounded strictly in the conversation tail, it briefly recaps what was
accomplished and names the next step in a warm mentor voice (a product requirement, not a canned
nudge). There is no canned text on this path — even the session-open greeting is generated.

### The condense path

A paste longer than 8,000 chars triggers **condense-first**: one no-tools call on the answerer
slot extracts a structured digest (≤1,400 chars) using a fixed template. The verbatim original is
persisted as the user message; only the digest — datamarked and marked `[user pasted N chars;
condensed]` — enters the model window. Embedded instructions in the paste are stripped and logged.
The gate and corpus-referent detection always run on the *original* text. Inputs above 32k chars
are sandwiched (first 24k + last 8k) before the condense call; anything above
`CHAT_MAX_RAW_INPUT_CHARS` (100,000) is hard-rejected. Full contract:
IMPLEMENTATION-SPEC §2[0] and `CONDENSE-PATH-TESTS.md`.

---

## 5. Component reference

Each module lives under `api/services/agent/`. For every one: what it does, its key invariants,
and where its tests live. Full contracts are in the spec section named.

- **`gate.py` — deterministic whitelist gate [step 0].** Matches EXACT literals (never patterns)
  across four classes (greeting/thanks/farewell/ack) after normalization; runs the ack offer-check;
  routes long pastes to condense-first. Invariants: whitelist-only (compounds structurally cannot
  match, so a real question is never swallowed); a trailing `?` always defeats the gate; the
  denylist lives only as CI assertions, never as match logic; ~10 non-English literals per class
  (Chinese-speaking users are a core segment). Tests: gate pytest + `TRIVIAL-TURN-HINGLISH.md`.
  Full contract: spec §2[0].

- **`dispatch.py` — Plan factory [step 1].** Deterministic. Returns `SmallTalkPlan` on a gate hit,
  else `LLMToolLoop` with the mode's tool allowlist. Defines `TurnContext`, the `Plan` dataclasses,
  and the `ForcedToolCall`/`ReducedToolLoop` stubs that are the **router seam** (§12). Invariant:
  `CannedReplyPlan` does not exist — trivial turns always hit the LLM. Full contract: spec §2[1].

- **`lexicon.py` — the single shared grounding mechanism.** Loads company canonical names + aliases
  from `silver.company` (drop terms ≤2 chars), plus EN/Hinglish/ZH intent words. Exposes
  `is_corpus_referent(text)`. This one module backs the gate's CI denylist assertions, the runtime
  escape-hatch guard, and the pre-answer grounding invariant. Invariant: CI asserts
  whitelist ∩ lexicon = ∅. Tuned broad, for recall. Tests: `GROUNDING-POLICY.md` boundary cases.
  Full contract: spec §2[2].

- **`loop.py` — the tool loop [steps 2–5], plus the runtime guard and grounding invariant.** Drives
  ≤3 rounds; `tool_choice` policy is `required` (round 1) → `auto` (subsequent) → `none` (terminal).
  The **runtime guard** vetoes an escape-hatch (`respond_directly`/`explain_concept`) pick when the
  message is a corpus referent, is question-shaped, or is consent-after-offer, and re-runs the
  selector once with the hatches stripped. The **grounding invariant** (deterministic, pre-answerer)
  is the single choke point: if the message is a corpus referent and no grounding tool succeeded, it
  forces a grounding round or appends a "the lookup failed; state no numbers" steering line. Tests:
  scripted transcript + grounding units. Full contract: spec §2[2], §2[5].

- **`executor.py` + `tools/*.py` — validation, entity resolution, fixed recipes [step 3].** Per
  tool call: validate args (pydantic) → resolve entities server-side → run the tool's FIXED recipe
  (SQL / vector / RRF) under a 5s timeout → post-process (seen-set exclusion, receipts shape,
  `basis`, one `wrap_ugc()` pass over declared UGC fields). Invariants: entity resolution is
  server-side and abstains rather than guesses (unmapped level → filter dropped + noted; unresolved
  company → no verbatim echo of the request string); recipes are code, never model-planned. Tests:
  executor pytest units + `ENTITY-RESOLUTION-SUITE.md`. Full contract: spec §2[3], §4.

- **`smalltalk.py` — the only trivial-turn terminal.** Both entrances (gate fast-path and the
  `respond_directly` tool) dispatch the same plan: one no-tool streamed answerer-slot call with a
  stable cached persona prefix, a gated user-context block, an anti-repeat block, and the
  conversation tail. Invariants: no canned text (session opener included); never promises actions;
  mentor-moment acks recap only tail-grounded work. Tests: trivial-turn suite (route + reply
  quality). Full contract: spec §2 SmallTalkPlan, §8.

- **`repair.py` — the fallback ladder.** Four rungs for selector failures: schema-validate →
  `json_repair` → one retry with the parse error quoted → escalate the whole turn to Haiku 4.5.
  Invariant: tool-call-shaped text in `content` is a *telemetry detector only* — never
  parse-and-executed (that would be an injection-to-execution laundering path). Logs `fallback_rung`
  0–4 on every turn (the model-quality dashboard). Tests: fault-injection units. Full contract:
  spec §3.

- **`history.py` — cross-request reconstruction + condense substitution.** Rebuilds the message list
  from persisted text rows plus the turn-log jsonb: synthesizes prior `tool_use` blocks from logged
  args (deterministic synthetic ids) and elides prior `tool_result` bodies to id-only stubs;
  substitutes the digest for any condensed prior turn. Invariants: prior-turn result bodies are
  always elided across requests; a verbatim paste never re-enters the model window; a 32k-token
  budget guard drops whole oldest turns first. Tests: `CONDENSE-PATH-TESTS.md` units. Full contract:
  spec §9.

- **`citations.py` — verifier + finish [step 6].** Scans the answer for `[iv_<uuid>]` tags; the
  allowed set = this turn's returned ids ∪ ids seen before this turn. Strips out-of-set tags
  (`fake_citation_count`), keeps but flags earlier-session tags (`stale_citation_count`), strips
  malformed tags. Emits the `sources` event with receipts hydrated via a server-side PK lookup that
  selects **only** the receipt columns (id, company, date, source, url) — never body text — so
  gated ids are safe to cite by construction. Tests: `CITATION-VERIFIER-TESTS.md` (22 cases). Full
  contract: spec §2[6].

- **`llm_client.py` — provider-agnostic model slots.** `complete(model_slot, messages, tools,
  tool_choice, stream)`. Three env slots: `CHAT_SELECTOR_MODEL`, `CHAT_ANSWERER_MODEL` (both
  `deepseek-v4-flash`), `CHAT_ESCALATION_MODEL` (`claude-haiku-4-5`). DeepSeek via the strict-beta
  base URL with `strict:true` schemas. Invariant: `thinking:{"type":"disabled"}` on **every**
  request (§6). Tests: replays the live smoke harness. Full contract: spec §2[2], §12.

---

## 6. Model strategy

**Two config slots, merged by default.** The selector (picks tools + fills args) and the answerer
(streams the grounded reply) are two environment slots, both `deepseek-v4-flash` at launch. Because
they default to the same model+provider, the answerer call is simply the loop's terminal completion
— **no extra model call**. The split is config, not code, so either slot can be upgraded
independently later.

**Escalation ladder.** On selector failure, the repair ladder ends by re-running the whole turn on
`claude-haiku-4-5` (Haiku 4.5 — the strongest small tool-caller measured, on a different provider
for outage independence). Haiku is a fallback lane, not a primary.

**Two live-verified DeepSeek criticals** (evidence: `DEEPSEEK-SMOKE-REPORT.md`, ~120 live calls):

1. **`thinking:{"type":"disabled"}` is a mandatory per-request parameter**, not a model property.
   `deepseek-v4-flash` defaults to thinking mode on both base URLs, and in thinking mode
   `tool_choice="required"` is rejected with **HTTP 400** (`Thinking mode does not support this
   tool_choice`). Omitting the param 400s the first selector call of every turn and silently routes
   all traffic to Haiku. `enable_thinking:false` is *not* the knob. Every call — selector, answerer,
   SmallTalk, condense — must include the disabled-thinking param. A nightly canary includes one
   intentionally-thinking `required` call as a contract-drift tripwire.
2. **No `format:"date"` in strict schemas.** DeepSeek's beta strict validator supports only
   email/hostname/ipv4/ipv6/uuid; `format:"date"` 400s the whole request. Date args are plain
   strings validated server-side by pydantic.

**Measured latency (live smoke, spec §12).** Selector single call 1.0–1.7s total (~0.9s streamed
TTFT); a 4-call parallel round 2.2–3.5s; answerer streamed TTFT 0.68–0.83s, a ~2.5k-char answer
6–10s; prefix cache hit confirmed. Budget for a 1-round grounded turn: first status ~1.4s, first
answer token ~3.5–5s, done ~7–9s. Because the selector deep-dives `get_interview` after a search,
typical grounded turns run 2 rounds (done ≈ 8.5–11s) unless the `get_interview`-damping tool
description holds. The models named `deepseek-chat`/`deepseek-reasoner` retire 2026-07-24 and must
never be referenced.

---

## 7. Data layer

**One Supabase Postgres** holds everything; the agent reuses `get_pg_pool()` / `SUPABASE_DB_URL`
with `statement_cache_size=0`. Tables:

- **`silver.company`** — canonical names + aliases; the source for entity resolution and the
  lexicon.
- **`silver.level_resolution`** — the level cache keyed `(company_norm, role_family, level_norm) →
  canonical_rank / native_code`. `company_norm` is a **code-side fold**, not a DB column. Abstain
  when no row exists or when `canonical_rank IS NULL` (~50% of rows), and always key on the
  *resolved* company's canonical fold, never the raw user surface.
- **`silver.interview`** — the interview grain; the pure-filter (no-semantic-query) branch queries
  it directly. `iv_<id>` = `silver.interview.id` (a UUID) everywhere: results, seen-set,
  `within_last_results`, citations.
- **`search.interview_chunk`** — the retrieval substrate: `embedding halfvec(1024)` (Voyage-3-large
  sized) with an HNSW index, FTS in the `fts` column, filter columns (`company_id`, `level_rung`,
  `outcome`, `round_type`, `posted_at`), and a `receipts_legal` flag. 24k+ rows.
- **`app.chat_sessions` / `chat_messages` / `chat_feedback`** — session state and the turn log.

**Phase-0 rebuild rationale.** A live audit found `search.interview_chunk` is a **stale snapshot**
(it lags live silver), and at launch it has **zero rows embedded**. Phase 0 therefore *regenerates*
the chunk table from live silver with the new columns (`interview_id` FK, denormalized
`interview_date`, `level_rung`, `receipts_legal`) and runs the embedding backfill. The rebuild and
backfills live with `build_search_chunks.py` in the ingest-pipeline repo; they are listed here
because retrieval depends on them.

**`receipts_legal` gating by data flow.** The flag is `false` for gated interviews (~5.8% of
chunks, computed from the parent's `content_gated` jsonb boolean — never key-presence, since 604
interviews carry `content_gated:false` and must stay legal). Gated chunk **text** is never embedded,
never ranked, never snippeted. Gated interviews still surface on the pure-filter path with
`snippet:null` — their *existence* plus structured metadata + link + date is the legal receipt
shape; only their text is untouchable.

**Session state columns** (migration `db/2026-07-09_chat_agent_state.sql`): `mode text`,
`fsm_state text`, `seen_interview_ids uuid[]`, `last_result_ids uuid[]`. All are read once by
`begin_turn` into the in-memory `TurnContext` and written once by `finish_turn` in a single
transaction. Concurrency (two tabs) uses atomic array-append, not full-array overwrite. Full
contract: spec §5.

---

## 8. Security and safety

**Prompt injection.** The corpus is user-generated forum content, so every UGC-derived field in a
tool result is datamarked by `wrap_ugc()` (Microsoft spotlighting: strip/escape `⟦ ⟧ ^`, then mark
whitespace) before it re-enters the model context. The launch toolset is **read-only** — with zero
side-effect tools, the worst outcome of a successful injection is a bad paragraph, never an action.

**Deterministic grounding enforcement.** Grounding is guaranteed by four code layers, none of them
a prompt: the **lexicon** (`is_corpus_referent`), the **runtime escape-hatch guard**, the
**pre-answer grounding invariant** (spec §2[5]), and the **citation verifier**. Each has a §6
telemetry field and is pytest-able.

**Gated-content legal rules.** Gated bodies never reach model context on any retrieval leg;
`get_interview` returns a strict field allowlist for gated interviews (structured fields +
paraphrase only, never `prompt_raw` or any of the five raw-text locations that hold verbatim text).
Full threat model: `GATED-CONTENT-THREATMODEL.md`.

**Two launch-blocking security items** (owner: pre-cutover checklist; do these before any public
exposure). RLS is off on all silver/search/app/public tables, but `anon`/`authenticated` have no
schema USAGE on silver/search/app, so the corpus is behind schema isolation. The two genuinely
open holes:

1. **`public.users` anon grants** — `anon` holds full SELECT/INSERT/UPDATE/DELETE on PII rows,
   live the instant PostgREST's schema cache recovers. Revoke them.
2. **A default-privilege landmine** auto-grants `anon` full CRUD on every *future* `public` table.
   Fix the default privilege.

Then, as defense-in-depth, **enable RLS** on silver/search/app with no anon policies (invisible to
the API/ingest, which connect as the `BYPASSRLS` owner) and confirm the exposed-schemas config
excludes them. The Supabase security advisor does **not** flag either blocker — do the manual grant
review. Full ordered checklist: `RLS-PRELAUNCH-CHECKLIST.md`.

---

## 9. Design decisions and rationale

The evidence register below (spec §16) records why each locked decision won; do not re-litigate any
of them without new evidence of comparable grade. Decisions are backed by multi-track research, an
adversarial design review, live production-data audits, and a ~120-call live provider smoke test —
artifacts in `docs/chat-agent-hardening/`.

| # | Decision (locked) | Rejected alternative | Why it won |
|---|---|---|---|
| 1 | No intent-classifier/router at launch; `dispatch()` seam + log-trained promotion path | LLM router; hand-written keyword lanes | Pre-traffic taxonomies overstate accuracy ~13pts (Rasa); classifiers re-test globally per change vs additive tool evals; forced-lane false positives are user-visible, guard re-runs invisible; lanes don't remove the LLM call |
| 2 | `tool_choice="required"` first call + escape hatches + deterministic veto | let the model decide if grounding is needed | Under `auto`, an ungrounded corpus answer streams before the server can intercept; under `required`, skipping retrieval is a named, loggable, vetoable pick. Live smoke: `required` works 30/30 |
| 3 | Selector/answerer = two config slots, merged terminal completion by default | one merged loop writing cited answers; always-separate answerer | Loop-written citations measurably rot (3–13% fabricated sources); our default costs zero extra calls while keeping the upgrade seam |
| 4 | One fat `search_interviews` (filters+semantic+sort+scope in one schema) | thin per-backend tools the model composes | Model-composed chains cap ~60% end-to-end even on frontier; fat tool converts composition → slot-filling, which small models do well |
| 5 | Entity resolution 100% server-side (model emits surface forms only) | model emits company_id/level_rung | 43% of identifier values fabricated when unconstrained; shifted ladders (Google L5=rank5 vs Amazon L5=rank4, live) make word-matching wrong; server lookup + abstain is testable |
| 6 | Fixed recipes inside executors | model plans SQL-vs-vector per query | Deletes the two biggest measured chain-failure classes (~30% wrong value propagation, ~20% early stopping); recipe behavior is pytest-able |
| 7 | Whitelist gate = exact literals only | regex/NLU smalltalk intents; no gate | Compounds structurally cannot match, so a real question can't be swallowed; the router backlash came from misrouted real queries; gate saves a full selector round on ~20% of turns (share to be measured) |
| 8 | Zero canned text (SmallTalkPlan generates everything; acks = mentor moments) | canned ack pool / templated greetings | Market unanimous (QnA Maker retired, Rasa added an LLM rephraser); product rule is varied, context-aware, tail-grounded recaps |
| 9 | `deepseek-v4-flash` both slots + Haiku 4.5 escalation | frontier everywhere; single-provider | Cost ≈ $900/mo @1k DAU vs ~$37k frontier (~40×); Haiku = best small tool-caller on a different provider (outage independence); smoke-verified with `thinking:disabled` + strict schemas |
| 10 | Hard 3-round cap + `none` terminal + `cap_violation` logging | unbounded agentic loop | Agents are fragile in repetition; cap turns tail-risk into a bounded, honest "couldn't finish" |
| 11 | Datamarking + read-only launch toolset | trust the model to ignore injected instructions | Corpus is UGC; spotlighting is the measured mitigation; with zero side-effect tools the worst injection is a bad paragraph, not an action |
| 12 | Session state server-managed, never model args | model remembers/carries IDs | Models fabricate IDs and windows get pruned; server referent set makes "more"/"those" deterministic across requests |
| 13 | Direct API calls, no LangGraph/LangChain | framework-hosted loop | The loop is ~300 easy lines; frameworks abstract exactly the raw params that bit us (`thinking:disabled` found only by seeing raw requests) |
| 14 | Grounding enforced by code, never by prompt | prompt instructions + goldens only | Prompts are requests, code is a guarantee; every enforcement point is pytest-able and has a telemetry field |
| 15 | Gated content excluded by data flow (`receipts_legal` on every leg) | prompt the model not to quote paywalled text | Legal, not stylistic; enforcement upstream of the model means there is nothing to leak |
| 16 | Modes = explicit UI state, never inferred (later tracks) | auto-detected mode switching | Silent mode inference breaks trust; mode = contract change, tool = capability |

---

## 10. Delivery plan

**Total: 20.5 person-days solo (~4 weeks; ~2.5–3 weeks with Phase 3/4 parallelized or a second
pair of hands on frontend + evals), plus 1.0d of pre-launch blocker tasks tracked separately
(different owners/tracks).**

| # | Phase | Deliverable | Accept when |
|---|---|---|---|
| 0 | Embeddings | backfill script + nightly sweep | all non-gated rows embedded (predicate-based count); the 50-query ranking eval passes its gate (recall@3 ≥ 0.75 overall, ≥ 0.6 per category, all single-target cases top-3, median distinct_interviews@40 ≥ 15); CHAT_SIM_FLOOR calibrated |
| 1 | Executors | `agent/tools/*` + entity resolution + seen-set + recipes + `agent/lexicon.py` | pytest recipes correct on fixtures; ranked results for "google L5 system design" look right by hand |
| 2 | Agent loop | `llm_client` (strict-beta + Haiku slot) + loop + ladder + history | scripted convo incl. follow-up + dependent call + cap; ladder fault-injection tested |
| 3 | Transport | SSE events into the existing route; citation verifier; FE re-spec | end-to-end streamed convo in the web UI; status <2s perceived |
| 4 | Prompts + gate | system core c1.0; whitelist gate + `SmallTalkPlan` + runtime guard; datamarking | gate pytest green; injection fixtures not followed |
| 5 | Evals | 50 goldens + trivial-turn suite (100–150) + CI + selector bake-off + canary | CI red/green demo; trivial-turn suite passes route+reply grading; bake-off table recorded |
| 6 | Cutover | log writer; scrap list deleted; `CHAT_PIPELINE=agent` everywhere; staged rollout | prod+dev identical pipeline; old tests removed; Langfuse traces flowing |

### 10.1 Task board

IDs are stable — reference them in commit messages. Effort = solo person-days incl. that task's
tests. "Load" = the spec sections + companion suites to read before starting. Do tasks in
dependency order; same-dependency tasks can run in parallel.

**Pre-launch blockers (not chat code — do first; they gate other tracks):**

| ID | Task | Effort | Deps | Done when |
|---|---|---|---|---|
| P-1 | Deploy the silver date/company hardening to production (ingest-pipeline repo): merge + deploy the pipeline changes, apply `db/2026-07-08_company_provisional_status.sql`, run `backfill_companies`, run the date-precision `refan_silver` rescue | 0.5d | — | PROD `silver.company.status` exists; the §2[3]/§2[2] interim mitigations are deleted |
| P-2 | Security blockers: RLS-checklist items 1–2 — revoke anon CRUD on `public.users`, fix the default-privilege landmine | 0.5d | — | grants query shows zero anon privileges on `public.users`; a new test table gets no auto-grant. Load: RLS-PRELAUNCH-CHECKLIST.md |

**Phase 0 — Data readiness (3.0d):**

| ID | Task | Effort | Deps | Done when |
|---|---|---|---|---|
| T0.1 | Rebuild `search.interview_chunk` from live silver with the new columns (`interview_id` composite backfill, denormalized `interview_date` + index, `receipts_legal` exact jsonb predicate, `level_rung` via silver join). Supersedes the "backfill 24,173 rows" framing — the live table is a stale snapshot, so Phase 0 regenerates it | 1.0d | P-1 | chunk interview count == live silver count; `receipts_legal=false` share ≈ gated share; 20-row hand check. Load: §5, GATED-CONTENT-THREATMODEL.md §E1 |
| T0.2 | Embedding backfill (Voyage-3-large → `halfvec(1024)`) + nightly sweep `WHERE embedding IS NULL AND receipts_legal` + the embedded-gated-rows=0 CI/nightly invariant | 1.0d | T0.1 | predicate-based full coverage; invariant query = 0 rows |
| T0.3 | Run the 50-query ranking eval + tune `iterative_scan`, RRF k, calibrate `CHAT_SIM_FLOOR`, record thin-evidence thresholds | 1.0d | T0.2 | Phase 0 acceptance row. Load: RANKING-EVAL.md |

**Phase 1 — Executors (4.5d):**

| ID | Task | Effort | Deps | Done when |
|---|---|---|---|---|
| T1.1 | ToolSpec registry: 5 strict schemas (no `format:"date"`), annotations `side_effect` (required)/`grounding`/`ugc_fields` | 0.5d | — | schemas round-trip the strict-beta validator |
| T1.2 | `agent/lexicon.py`: company-term loader (canonical+aliases, ≤2-char drop), EN/Hinglish/ZH intent words, `is_corpus_referent`, CI assertions (whitelist∩lexicon=∅) | 0.5d | P-1 | pytest green incl. GROUNDING-POLICY.md boundary cases |
| T1.3 | Entity resolution: company fold+alias (+status rule), level canonical-fold lookup + NULL-rank abstain, outcome mapping + nightly drift guard, no-echo miss messages | 1.0d | T1.2 | the 6 adversarial cases pass. Load: §2[3], ENTITY-RESOLUTION-SUITE.md |
| T1.4 | `search_interviews` recipe: hybrid (SQL base → HNSW top-40 + FTS top-40 → RRF k=60 → interview collapse), pure-filter branch, scope/seen-set invariants 1–6, `basis` + thin-evidence, relevance-gated recency | 1.5d | T0.1, T0.2, T1.3 | pytest units (a)–(d) green; hand-check "google L5 system design" |
| T1.5 | `get_interview` (id membership check, accumulator join, gated field allowlist) + `corpus_stats` + `explain_concept`/`respond_directly` executors + `wrap_ugc()` | 1.0d | T1.3 | gated payload contains no body-text field; injection fixtures inert |

**Phase 2 — Agent loop (4.5d):**

| ID | Task | Effort | Deps | Done when |
|---|---|---|---|---|
| T2.1 | `llm_client.py`: provider-agnostic slots, strict-beta base URL, `thinking:{"type":"disabled"}` on every request, streaming, Haiku lane | 1.0d | T1.1 | replays the smoke harness green (required 10/10, none honored, parallel emission) |
| T2.2 | `loop.py` + `dispatch.py` (TurnContext/Plan dataclasses, Plan factory, mode allowlists, router stubs): required→auto→none policy, parallel-batch=one-round, 3-trigger runtime guard, grounding invariant, cap handling + `cap_violation` | 1.5d | T2.1, T1.2, T1.4, T1.5 | scripted convo: follow-up + dependent call + cap + guard-override matches §2 |
| T2.3 | `repair.py`: 4-rung ladder, `fc_text_emission` detection-only, rung-4 re-run with a fresh pre-turn session-state copy, fault-injection tests | 0.5d | T2.1 | ladder unit tests with injected malformed/text-shaped outputs |
| T2.4 | `history.py` (synthetic ids, elision stubs, digest substitution, 32k budget) + condense path (CONDENSER_SYSTEM, no-tools invariant, 100k cap + 24k/8k sandwich, digest persisted) | 1.5d | T2.1 | CONDENSE-PATH-TESTS.md §G units green; ordinal follow-up resolves from fixture turn-logs |

**Phase 3 — Transport (3.0d):**

| ID | Task | Effort | Deps | Done when |
|---|---|---|---|---|
| T3.1 | `run_turn` AsyncIterator + route stream-body rewrite + `begin_turn`/`finish_turn` extensions (atomic array-append) + `db/2026-07-09_chat_agent_state.sql` + SSE keepalive / persist-partial on disconnect | 1.0d | T2.2 | end-to-end streamed turn via curl; state columns round-trip; single-txn finish verified; client kill mid-stream → partial persisted |
| T3.2 | `citations.py`: verifier (allowed-set = accumulator ∪ seen_pre, strict UUID grammar, STRIP/KEEP/MARK), sources hydration (column whitelist), uncited-count scan (telemetry-only) | 1.0d | T3.1 | CITATION-VERIFIER-TESTS.md 22 cases green |
| T3.3 | Frontend: `streamChatMessage` named-event parser (done/full_text back-compat), sources UI with stale separator, chips render-from-sources, `rich-blocks.tsx` re-spec | 1.0d | T3.1 | full convo in the web UI; fake tag renders as nothing |

**Phase 4 — Gate + prompts (1.5d, parallel with Phase 3):**

| ID | Task | Effort | Deps | Done when |
|---|---|---|---|---|
| T4.1 | `gate.py`: whitelist literal sets (EN+ZH+Hinglish), normalize (trailing-run strip, danda, emoji w/ ❓ exception), ack offer-check, end-anchored IDENTITY/MODE_EXIT, CI denylist assertions | 0.5d | — | gate pytest green incl. TRIVIAL-TURN-HINGLISH.md cases; "cool, now Google L5?" falls through |
| T4.2 | `smalltalk.py` SmallTalkPlan (persona prefix, gated user-context, anti-repeat, tail; session opener; hard-failure string) + system core c1.0 + steering blocks; run the 50-sample distinct-n temperature test | 1.0d | T2.1 | mentor-moment ack demo (grounded recap, nothing invented); byte-stable prefixes verified |

**Phase 5 — Evals (2.5d):**

| ID | Task | Effort | Deps | Done when |
|---|---|---|---|---|
| T5.1 | promptfoo+agentevals wiring, ~50 core goldens, merge companion suites (grounding 18, gated 12, entity 6, condense 16, multitool subset), pytest batteries in CI | 1.5d | T3.x, T4.x | CI red/green demo on an intentional break |
| T5.2 | Trivial-turn suite (100–150 EN+ZH+Hinglish) + selector bake-off (v4-flash vs Haiku vs 4o-mini ×3, + trivial routing half) + nightly canary incl. the thinking-mode tripwire | 1.0d | T5.1 | bake-off table recorded; ship-gate pass on the trivial path |

**Phase 6 — Cutover (1.5d):**

| ID | Task | Effort | Deps | Done when |
|---|---|---|---|---|
| T6.1 | Turn-log writer (full field set), Langfuse spans, online monitors + alert thresholds | 0.5d | T3.1 | dashboards show a scripted convo correctly |
| T6.2 | Scrap-list deletion (~4.5k lines + old promptfoo + dead FE fns), `CHAT_PIPELINE=agent` everywhere, RLS-checklist items 3–4, staged rollout | 1.0d | ALL | prod+dev identical pipeline; old tests removed |

**Later tracks (specified in the v2 design doc, NOT launch):** write-gate + confirm UX + taint
rule (+4–6d) → mentor tools; mock-interview FSM (+5–8d); weak-areas nightly job (+3–4d);
`get_stats` gold tool (+1–2d after gold-A); suggestion-chip/classifier (only once the router
promotion criteria fire).

---

## 11. Testing and evals

Everything lives under `evals/chat/`; promptfoo-action is **merge-blocking** on changes to
`agent/**` or prompts.

- **Golden set** (~50 cases): asserts valid tool call, tool name, hard args exact (`company`,
  `level`, `sort`, `within_last_results`), semantic_query loose. Mix: single-intent filtered (incl.
  a long-paste condense case), follow-up state-carry, compare-two-companies, empty-result honesty,
  chitchat (asserting `route=gate_smalltalk`), injection, entity-resolution adversarial,
  out-of-corpus refusals. Plus a grounding subset (18 cases) and a gated-content suite (12 cases).
- **Trivial-turn suite** (100–150 cases, separate from the golden set): the **ship gate for the
  trivial-turn path specifically** — grades both route choice and reply quality (rubric-graded,
  in-language for ZH, in-register for Hinglish). This is the primary evidence on an unbenchmarked
  model: `tool_choice="required"` safety on `deepseek-v4-flash` is unproven, and at 25 cases a
  5–10% misroute rate is statistically invisible. Includes a mentor-moment-ack case and the 44
  Hinglish/Indian-English cases.
- **Selector bake-off** (run first, in Phase 5): the 50 goldens plus the trivial routing half, run
  against v4-flash vs Haiku 4.5 vs 4o-mini in the selector slot ×3 repeats; pick on arg-accuracy +
  flip-rate + trivial-route accuracy. Answers "which model is the consistent classifier"
  empirically.
- **Nightly canary** (20 cases vs live provider): alert on delta; **includes the thinking-mode
  tripwire** — one intentionally-thinking `required` call that must 400, catching the day the
  provider changes the default.
- **Gate/executor unit tests** (pure pytest, no LLM): gate regexes; executor recipes (two-turn
  `within_last_results`, empty-result-doesn't-clobber, seen-set FIFO cap, referent short-circuit);
  the 22-case citation verifier; the gated no-body-text invariant; the grounding-invariant fire on
  a forced tool-timeout; the uncited-count scan.
- **Online monitors:** per-segment `respond_directly` rate, `guard_override` rate,
  `fc_text_emission` rate, gate-hit rate by language — all from the turn log. Trivial-turn share is
  measured from `route` labels, never assumed.

### 11.1 Companion suite index (`docs/chat-agent-hardening/`)

An implementation session picking up any phase MUST read the matching suite — they carry the exact
prompts, fixtures, pinned PROD UUIDs, and assertions this document only summarizes:
`GROUNDING-POLICY.md`, `GOLDENS-MULTITOOL.md` (+ `goldens_multitool.yaml`), `RANKING-EVAL.md`,
`GATED-CONTENT-THREATMODEL.md`, `ENTITY-RESOLUTION-SUITE.md`, `TRIVIAL-TURN-HINGLISH.md`,
`CONDENSE-PATH-TESTS.md`, `CITATION-VERIFIER-TESTS.md`, the DeepSeek smoke report + harness +
raw runs, and `RLS-PRELAUNCH-CHECKLIST.md`.

---

## 12. Scale roadmap

**Stateless scaling knobs.** The turn engine holds no cross-request in-flight state: `begin_turn`
reads session state once into the in-memory `TurnContext`, `finish_turn` writes it once. Scaling
from 1k to 10k DAU is therefore configuration only — uvicorn workers, asyncpg pool (10–20),
provider tier. The 2M-MAU path adds replicas + pgbouncer + multi-provider (already built), and only
*then* the log-trained classifier at the dispatch seam.

**The router promotion path.** The end state at high DAU is `dispatch()` → deterministic lane
router → `ForcedToolCall`/`ReducedToolLoop` with a restricted toolset, the LLM only slot-filling
args. This is a **cost/consistency optimization on top of** the grounding guarantee (which is
already enforced by code), so it must be *grown from production data, not written from intuition* —
pre-traffic keyword taxonomies overstate real accuracy (~13pts, Rasa), and a hand rule has no
calibrated confidence. The stubs (`ForcedToolCall`, `ReducedToolLoop`) exist as real types today so
the seam is code, not a comment. Promotion procedure:

1. Mine the turn log for (lexicon-signal pattern → round-1 tool choice) pairs.
2. A pattern with **≥99% consistency over ≥5,000 logged turns** becomes a candidate lane.
3. Run it in **shadow mode** (log `router_would_force` beside the live selector's actual choice,
   compare ≥2 weeks).
4. Only then promote to `ForcedToolCall` (selector still fills args) or `ReducedToolLoop`.

**Why lanes are NOT hand-written at launch.** Forced-lane false positives are *user-visible* wrong
behavior, whereas the runtime guard's re-run is invisible — that asymmetry is why lanes need
measured precision before they get authority. And lanes do not remove the selector call anyway
(routers can't slot-fill), so the savings are consistency + schema tokens, not the LLM call.

---

## 13. Prerequisites and open items

**Launch-blocking prerequisites:**

- **P-1 — silver company-status hardening must reach production.** The verified/provisional company
  status hardening is DEV-only today; the entity-resolution suggestion rules and the verified-only
  lexicon filter depend on it. Interim mitigations are specified in the spec §2[3] and must be
  removed once it ships. Owner: silver/ingest track.
- **P-2 — `public.users` lockdown.** RLS-checklist items 1–2 (revoke anon CRUD on `public.users`,
  fix the default-privilege landmine) before any public exposure; items 3–4 (enable RLS as
  defense-in-depth, confirm exposed-schemas config) pre-cutover. Owner: pre-cutover checklist.

**Deferred items (parked, not lost — clean voice from the spec §15):**

- **`position_closed` outcome mapping** (340 rows) is currently folded into `reject` by the
  server-side outcome mapping — this conflates "req closed" with "candidate rejected." Revisit when
  outcome filters see real usage.
- **Entity-table hygiene debt.** Qualifier-bearing aliases (`amazon sde-2`, `amazon london`) should
  be rejected at mint time; one inverted canonicalization (`d. e. shaw india`) needs fixing; an
  orphan `level_resolution` row (`tiktok`, unreachable once lookups key on the canonical fold)
  needs a nightly assertion and re-keying. Must land with the canonical-fold lookup rule. Owner:
  silver/ingest track.
- **Condense-path digest-only answerer limitation.** Critique-type pastes (resume, own essay >8k
  chars) reach the answerer only as a ≤1,400-char digest, so detailed critique is degraded at
  launch. A split view (selector sees digest, answerer sees the original when small enough) is a
  post-launch change — it forks the shared message list and cache assumptions. Size demand from
  `input_condensed.content_type` telemetry first.

---

## 14. Document map

- **This document (`CHAT-AGENT-DESIGN.md`)** — the *why* and the *what*. Read it first, cold, to
  understand the system and pick up your first task.
- **`CHAT-AGENT-IMPLEMENTATION-SPEC.md`** — the *how*: exact tool schemas, prompts, SQL recipes,
  the repository and SSE contracts, per-task acceptance criteria. The contract for implementers;
  where this doc summarizes, the spec is authoritative.
- **`docs/chat-agent-hardening/*`** — test fixtures, threat models, golden suites, entity-resolution
  and citation-verifier cases, and the live-provider smoke evidence. An implementer must read the
  suite matching their phase (see §11.1) before starting.

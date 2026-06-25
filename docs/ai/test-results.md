# Progressive Disclosure — Test Results

> Test run for `recipe-agent-events` progressive disclosure docs.
> Date: 2026-06-25 · Standard: AgoraIO-Community/ai-devkit progressive-disclosure.

## Step 1 — Structural checks

| Check                                              | Result |
| -------------------------------------------------- | ------ |
| `L0_repo_card.md` ≤ 50 lines                       | Pass (36) |
| All 8 L1 files present                             | Pass |
| Each L1 has purpose blockquote + Related Deep Dives| Pass (8/8) |
| L1 line counts in 80–200 target                    | **Below target** (37–104) — see note |
| L2 `_index.md` present                             | Pass |
| Each L2 opens with "When to Read This" callout     | Pass (2/2) |
| Relative links resolve (`docs/ai/` + AGENTS.md)    | Pass (39/39, 0 broken) |
| AGENTS.md has How to Load / Git Conventions / Doc Commands | Pass |

**Note on L1 line counts:** files are table-dense and information-complete but
run 37–104 lines, mostly under the 80–200 soft target. The standard favors tables
over prose and warns against bloat, so they were left concise rather than padded.
`06_interfaces.md` hits 104 lines due to the event taxonomy table. Accepted
deviation; revisit if a section needs more depth.

## Step 2/3 — Question runs

Questions span the five standard categories. Each answer was checked against the
repo source before being marked Pass. "Level" is the lowest disclosure level
that fully answers the question.

### Setup & Build

| # | Question | Expected answer | Source of truth | Level | Status |
|---|----------|-----------------|-----------------|-------|--------|
| 1 | How do I install and run it locally? | `bun run setup` then `bun run dev` (backend :8000 + web :3000). | `L1/01_setup.md` ↔ `package.json` scripts | L1 | Pass |
| 2 | Which env vars are required? | `AGORA_APP_ID`, `AGORA_APP_CERTIFICATE`. `OPENAI_API_KEY` is optional. | `L1/01_setup.md` ↔ `server/src/agent.py` `__init__` | L1 | Pass |
| 3 | Is this zero-key? | Yes — OpenAI is Agora-managed; no `OPENAI_API_KEY` required by default. | `L1/01_setup.md`, `07_gotchas.md` ↔ `README.md`, `agent.py` | L1 | Pass |

### Test & Run

| # | Question | Expected answer | Source of truth | Level | Status |
|---|----------|-----------------|-----------------|-------|--------|
| 4 | How do I run backend tests without cloud creds? | `cd server && pytest tests -v`; `conftest.py` fakes env + SDK session. | `L1/04_conventions.md`, `01_setup.md` ↔ `tests/conftest.py` | L1 | Pass (ran: 2 passed) |
| 5 | What's the narrowest gate for a web-only change? | `bun run verify:web`. | `L1/05_workflows.md` ↔ `package.json` | L1 | Pass |
| 6 | What does `verify:local:fastapi` do? | Spawns real FastAPI with `FakeAgent` and proxies routes through the rewrite map. | `L1/03_code_map.md`, `05_workflows.md` ↔ `web/scripts/verify-local-fastapi.ts` | L1 | Pass |

### Conventions

| # | Question | Expected answer | Source of truth | Level | Status |
|---|----------|-----------------|-----------------|-------|--------|
| 7 | What response shape do backend routes use? | `{ code, msg, data }`; `data` only when there's a payload. | `L1/04_conventions.md`, `06_interfaces.md` ↔ `server/src/server.py` | L1 | Pass |
| 8 | How are errors mapped to HTTP codes? | `ValueError→400`, `RuntimeError→500`, else 500 via `_to_http_error`. | `L1/04_conventions.md` ↔ `server/src/server.py` | L1 | Pass |
| 9 | What are the commit/branch conventions? | Conventional commits `type: description`; branches `type/short-description`; no AI tool names. | `AGENTS.md` Git Conventions | L0 | Pass |

### Development

| # | Question | Expected answer | Source of truth | Level | Status |
|---|----------|-----------------|-----------------|-------|--------|
| 10 | How do I add a new browser-facing route? | Add FastAPI handler → add rewrite in `next.config.ts` → add client helper → extend `verify-api-contracts.ts`. | `L1/05_workflows.md` ↔ source | L1 | Pass |
| 11 | Where are event flags set and what are they? | All four flags in `Agent.start()` in `server/src/agent.py`: `data_channel`, `enable_metrics`, `enable_error_message` (in `parameters`), `enable_rtm` (in `advanced_features`). | `L1/06_interfaces.md`, `AGENTS.md` ↔ `agent.py` | L1 | Pass |
| 12 | Where does token generation live? | `server/` (`generate_convo_ai_token` in `server.py`); App Certificate stays server-side. | `L1/02_architecture.md`, `08_security.md` ↔ `server/src/server.py` | L1 | Pass |

### Deep Dive

| # | Question | Expected answer | Source of truth | Level | Status |
|---|----------|-----------------|-----------------|-------|--------|
| 13 | How does an RTM event become a `TimelineEvent`? | SDK callback (`AgoraVoiceAI`) in `ConversationComponent.tsx` calls `addEvent({kind, label, detail})`; buffer capped at 50 via `.slice(-50)`. | `L2/event_taxonomy.md` ↔ `ConversationComponent.tsx`, `EventTimeline.tsx` | L2 | Pass |
| 14 | What payload does `AGENT_METRICS` carry and how is it shown? | `{ type, name, value (ms) }` → `TimelineEvent` label `"{type}: {name}"`, detail `"{Math.round(value)}ms"`; also drives `QuickstartPipelineMetrics` (last 8). | `L2/event_taxonomy.md` ↔ `ConversationComponent.tsx` | L2 | Pass |
| 15 | How does stop survive a backend restart? | `_sessions` is in-memory; missing session falls back to `client.stop_agent(agent_id)`. | `L2/session_lifecycle.md` ↔ `server/src/agent.py` | L2 | Pass |

## Step 4 — Analysis

- All 15 questions answered at the expected disclosure level (12 at L1/L0, 3 at L2).
  No "correct but needed L2 unnecessarily" or "wrong/missing L2" cases.
- One known gap: `server/.env.example` ships stale translator vars (`SOURCE_LANG`, `TARGET_LANG`, `TTS_VOICE`) from a prior copy-paste. These are unused and documented as a gotcha in `L1/07_gotchas.md`. No source code reads them.
- One soft deviation: L1 line counts below the 80–200 target (accepted; concise/table-dense).

## Step 5 — Summary

| Category       | Questions | Pass | Notes |
| -------------- | :-------: | :--: | ----- |
| Setup & Build  | 3 | 3 | — |
| Test & Run     | 3 | 3 | backend tests executed: 2 passed |
| Conventions    | 3 | 3 | — |
| Development    | 3 | 3 | — |
| Deep Dive      | 3 | 3 | resolved at L2 as designed |
| **Total**      | **15** | **15** | — |

## Step 6 — Fixes / Retest

No failing questions; no fixes required. Evidence executed during this run:

- `pytest tests -v` (throwaway venv /tmp/v_events, Python 3.14.4) → `2 passed in 0.60s`.
- Relative link check → `39 checked, 0 broken`.

**Known gap (not a doc defect):** `server/.env.example` contains stale translator vars. Documented in `L1/07_gotchas.md`. Fixing the `.env.example` is a separate code task, not a documentation task.

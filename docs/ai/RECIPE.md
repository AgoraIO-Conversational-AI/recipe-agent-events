---
recipe_version: 1.0.0
recipe_status: experimental
extension_points:
  - id: api.routes
    name: Browser-facing API routes
  - id: agent.vendor-config
    name: STT/LLM/TTS vendor selection, model, system prompt, greeting, VAD, and session parameters
  - id: agent.event-flags
    name: RTM event flags (enable_metrics, enable_error_message, enable_rtm, data_channel)
  - id: web.event-timeline
    name: EventTimeline kinds, styles, and TimelineEvent buffer
  - id: web.conversation-ui
    name: Conversation UI panels and controls
  - id: verification.contracts
    name: Contract, proxy, and local FastAPI smoke verification
invariants:
  - id: api.rewrite-boundary
    summary: Browser calls stay on /api/* and Next rewrites to FastAPI; no Route Handlers for agent/token logic.
  - id: secrets.server-only
    summary: Agora App Certificate stays in the Python backend; OPENAI_API_KEY (if provided) is also server-only.
  - id: pipeline.cascade
    summary: Agent uses the cascading STT/LLM/TTS pipeline (DeepgramSTT → OpenAI → MiniMaxTTS); no MLLM.
  - id: events.rtm-only
    summary: All built-in events (state, metrics, errors, transcript) are delivered over RTM; the web client subscribes via AgoraVoiceAI.
  - id: token.uid-concrete
    summary: Backend resolves missing, zero, or negative UIDs before issuing an RTC+RTM token.
stable_contracts:
  - id: env.required
    summary: AGORA_APP_ID and AGORA_APP_CERTIFICATE are required; AGENT_BACKEND_URL is required by deployed web rewrites.
  - id: api.core-routes
    summary: GET /api/get_config, POST /api/startAgent, and POST /api/stopAgent remain the browser-facing contract.
  - id: response.envelope
    summary: Successful backend responses use { code, msg, data }.
  - id: events.built-in-only
    summary: Only the four built-in RTM event types are available (AGENT_STATE_CHANGED, AGENT_METRICS, AGENT_ERROR/MESSAGE_ERROR, TRANSCRIPT_UPDATED); no server-side custom-event API.
---

# Recipe Contract

This base recipe defines the reusable surface for a Python-backed Agora Conversational AI **events observability** quickstart: a cascading STT/LLM/TTS agent whose built-in RTM event surface is surfaced in a live EventTimeline in the browser.

## Recipe Role

- Role: `base` recipe (self-contained, clone-and-run; no `Extends` pin).
- Target audience: developers who want to observe Agora ConvoAI agent lifecycle events (state, per-stage latency, errors, transcript turns) without any LLM API key.
- Reuse model: clone, bind project, run, then customize agent pipeline, event handling, or browser UI.

## Recipe Scope

- Python FastAPI token generation and managed agent lifecycle.
- A cascading `DeepgramSTT → OpenAI (Agora-managed) → MiniMaxTTS` pipeline with RTM event flags enabled.
- Next.js browser UI with live `EventTimeline`, annotated transcript, RTC audio, RTM event subscription, connection status.
- Rewrite-only `/api/*` browser facade hiding backend placement.
- Contract, proxy, and local FastAPI smoke verification that need no live Agora calls.

## Baseline Implementation Guidance

Use this repo's source and progressive disclosure docs as the starting point, then customize. Do not recreate the Agora ConvoAI integration from memory — vendor schemas, SDK builder fields, token behavior, event flags, and RTM details drift. Copy verified patterns from this repo.

## Extension Points

| ID | Surface | How to extend | Required follow-up |
| -- | ------- | ------------- | ------------------ |
| `api.routes` | `server/src/server.py`, `web/next.config.ts`, `web/src/services/api.ts` | Add FastAPI route, add rewrite, add browser fetch helper. | Extend `web/scripts/verify-api-contracts.ts`; add proxy/fastapi coverage. |
| `agent.vendor-config` | `server/src/agent.py` `Agent.start()` | Change `OPENAI_MODEL`, system prompt, greeting, VAD `turn_detection`, TTS voice/model, or `parameters`. | Run `verify:backend` + `pytest tests`; document new env in `server/.env.example` (never add `PORT`). |
| `agent.event-flags` | `server/src/agent.py` `Agent.start()` | Change `enable_metrics`, `enable_error_message`, `data_channel`, or `advanced_features.enable_rtm`. | Update matching `AgoraVoiceAI` listeners in `ConversationComponent.tsx`. |
| `web.event-timeline` | `web/src/components/EventTimeline.tsx`, `ConversationComponent.tsx` | Add new kind to `TimelineEvent`, add `KIND_STYLES` entry, add SDK listener. | Keep `EventTimeline.tsx` as the sole source for the `TimelineEvent` type. |
| `web.conversation-ui` | `web/src/components/*`, `web/src/lib/conversation.ts` | Customize pre-call, transcript, metrics, timeline, mic, or visualizer UI. | Preserve RTC/RTM lifecycle ownership and transcript UID normalization. |
| `verification.contracts` | `web/scripts/*.ts`, root `package.json` | Add checks for new browser/backend boundaries. | Keep checks runnable without live Agora credentials. |

## Invariants

- Browser code calls only `/api/get_config`, `/api/startAgent`, and `/api/stopAgent` for the default flow.
- Next.js owns `/api/*` through rewrites only; no `web/app/api/**/route.ts` for agent/token logic.
- FastAPI owns token generation, `AGORA_APP_CERTIFICATE`, and agent lifecycle.
- Event flags (`data_channel`, `enable_metrics`, `enable_error_message`, `advanced_features.enable_rtm`) are set once in `Agent.start()`.
- Only built-in RTM event types are available; no server-side custom-event API exists in this recipe.
- The backend issues one RTC+RTM-capable token for a concrete non-zero UID.

## Stable Contracts

| Contract | Stable shape |
| -------- | ------------ |
| Required backend env | `AGORA_APP_ID`, `AGORA_APP_CERTIFICATE` |
| Optional backend env | `OPENAI_MODEL`, `OPENAI_API_KEY`, `AGENT_GREETING`, `PORT` (env only) |
| Required web deploy env | `AGENT_BACKEND_URL` |
| `GET /api/get_config` | Query `channel?`, `uid?`; returns `data.app_id`, `data.token`, `data.uid`, `data.channel_name`, `data.agent_uid`. |
| `POST /api/startAgent` | Body `{ channelName, rtcUid, userUid, parameters? }`; returns `data.agent_id`, `data.channel_name`, `data.status`. |
| `POST /api/stopAgent` | Body `{ agentId }`; returns `{ code: 0, msg: "success" }`. |
| Success envelope | `{ "code": 0, "msg": "success", "data": ... }` where the route has data. |
| Verification entry points | `bun run verify:web`, `bun run verify:backend`, `bun run verify:web:proxy`, `bun run verify:local:fastapi`, `bun run verify:local`. |

## Internal / Subject to Change

- Visual layout, component composition, Tailwind classes, and assets under `web/src/components/`.
- Exact STT model, TTS model/voice, VAD timing, and greeting text, as long as they stay documented extension points.
- In-memory `Agent._sessions` details; the stable behavior is start by channel/user and stop by returned `agent_id`.
- Verification internals under `web/scripts/`; the stable surface is the root script names and what they assert.
- `agora-agents` SDK minor-version behavior; this recipe lower-bounds `>=2.3.0` but does not freeze every field.

## Related Progressive Disclosure Docs

- `L1/01_setup.md` — setup, env, and commands.
- `L1/02_architecture.md` — request flow, event flag wiring, and topology.
- `L1/05_workflows.md` — common modification workflows.
- `L1/06_interfaces.md` — route, rewrite, env, event taxonomy, and `TimelineEvent` contracts.
- `L1/L2/event_taxonomy.md` — full RTM event payloads and `AgoraVoiceAI` subscription detail.
- `L1/L2/session_lifecycle.md` — RTC/RTM/session orchestration.

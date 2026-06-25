# 02 · Architecture

> Two co-located processes. The browser talks only to Next.js `/api/*`, which rewrites to the FastAPI agent backend. The backend owns Agora tokens and the agent session, enabling three built-in event flags so that RTM streams live observability events to the browser.

## Topology

```
Browser (localhost:3000)
  │  fetch /api/*
  ▼
Next.js (web/)  ──rewrite──▶  Agent backend (server/, :8000)
                                 │  starts agent with event flags:
                                 │    enable_rtm=true, enable_metrics=true,
                                 │    enable_error_message=true, data_channel="rtm"
                                 ▼
                              Agora ConvoAI Cloud
                                 │  DeepgramSTT (nova-3, en) → OpenAI (Agora-managed, keyless)
                                 │  → MiniMaxTTS → user's channel
                                 │  RTM events → browser:
                                 │    AGENT_STATE_CHANGED, AGENT_METRICS,
                                 │    AGENT_ERROR, MESSAGE_ERROR, TRANSCRIPT_UPDATED
                                 ▼
                              EventTimeline + annotated transcript in the web UI
```

- **`web/`** — Next.js 16 / React 19 / TypeScript. Owns UI plus the RTC/RTM client lifecycle, EventTimeline, and annotated transcript. Calls only `/api/*`.
- **`server/`** — Python FastAPI (:8000). Owns Agora token generation and agent session lifecycle. SDK: `agora-agents>=2.3.0` (`import agora_agent`).
- No `llm/` service — OpenAI is Agora-managed (no BYO key required by default).

## Request lifecycle

1. Browser `GET /api/get_config` → Next rewrites to backend `/get_config`; backend mints a Token007 from `AGORA_APP_ID` + `AGORA_APP_CERTIFICATE` and returns channel + UIDs.
2. Browser joins the RTC channel, logs in to RTM, then `POST /api/startAgent`; backend builds the STT/LLM/TTS cascade with the three event flags and starts an async agent session.
3. Agora routes user audio through `DeepgramSTT → OpenAI → MiniMaxTTS`; the agent's voice is delivered back into the channel.
4. RTM delivers `AGENT_STATE_CHANGED`, `AGENT_METRICS`, `AGENT_ERROR`, `MESSAGE_ERROR`, and `TRANSCRIPT_UPDATED` events to the web UI. The browser's `AgoraVoiceAI` SDK converts each into a `TimelineEvent` (capped at 50).
5. `EventTimeline` renders events reverse-chronologically with a colored badge per kind.
6. `POST /api/stopAgent { agentId }` ends the session.

## Why no `llm/` service

The events recipe uses the **managed OpenAI vendor** (`agora_agent.agentkit.vendors.OpenAI`). Agora holds the OpenAI API key on its cloud; the recipe is zero-key by default. An optional `OPENAI_API_KEY` env var lets you bring your own account. No separate LLM service to expose publicly — no tunnel required.

## Key abstractions

- **`Agent`** (`server/src/agent.py`) — async wrapper around `AgoraAgent`; owns the `AsyncAgora` client, env, and the in-memory `_sessions` map keyed by `agent_id`. Sets all event flags and VAD config.
- **`EventTimeline`** (`web/src/components/EventTimeline.tsx`) — React component rendering a `TimelineEvent[]` buffer (max 50); exports the `TimelineEvent` type.
- **`AgoraVoiceAI`** (from `agora-agent-client-toolkit`) — client SDK that translates RTM messages into typed event callbacks (`AGENT_STATE_CHANGED`, `AGENT_METRICS`, etc.).
- **Rewrite proxy** (`web/next.config.ts`) — the only browser→backend boundary; no Next Route Handlers exist for agent/token logic.

## Tech decisions

- **Rewrites, not Route Handlers** — hides backend placement behind `/api/*` so the same client works locally and deployed (set `AGENT_BACKEND_URL`).
- **Event flags on the backend** — `data_channel="rtm"`, `enable_metrics=True`, `enable_error_message=True`, and `advanced_features={"enable_rtm": True}` are set once in `Agent.start()`; the web client does not control them.
- **Net-new work is web-only** — the backend has no special event logic beyond the three flags; all observable behavior is in `EventTimeline` and the `AgoraVoiceAI` subscription.

## Related Deep Dives

- [event_taxonomy](L2/event_taxonomy.md) — full RTM event payloads, `TimelineEvent` mapping, and `AgoraVoiceAI` subscription wiring.
- [session_lifecycle](L2/session_lifecycle.md) — browser orchestration of config + start/stop, RTC/RTM, transcript mapping.

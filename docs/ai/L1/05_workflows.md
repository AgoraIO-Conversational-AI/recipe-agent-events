# 05 · Workflows

> Step-by-step guides for the common changes in this recipe. Each ends with the narrowest verify command to run.

## Add or change a browser-facing route

1. Add the FastAPI handler in `server/src/server.py` (return the `{ code, msg, data }` envelope).
2. Add the `/api/<name>` → `/<name>` mapping in `web/next.config.ts` `rewrites()`.
3. Add a client helper in `web/src/services/api.ts`.
4. Extend `web/scripts/verify-api-contracts.ts` with the new path + envelope assertions.
5. Verify: `bun run verify:web` (and `bun run verify:local:fastapi` if it should go through the real backend).

## Change the agent prompt / greeting / model

1. Greeting: set `AGENT_GREETING` (env) or edit the default in `server/src/agent.py`.
2. Model: set `OPENAI_MODEL` (default `gpt-4o-mini`).
3. Other LLM options (system message, temperature): edit the `OpenAI(...)` constructor in `Agent.start()`.
4. Verify: `bun run verify:backend` (compile) + `cd server && pytest tests -v`.

## Change or add event flags

1. Edit the `parameters` dict or `advanced_features` in `Agent.start()` in `server/src/agent.py`.
   - `data_channel`, `enable_metrics`, `enable_error_message` are in `parameters`.
   - `enable_rtm` is in `advanced_features`.
2. If adding a new event kind, add the `AgoraVoiceAIEvents.*` listener in `ConversationComponent.tsx` and map it to a `TimelineEvent` kind.
3. Verify: `bun run verify:local:fastapi`.

## Add a new `TimelineEvent` kind to the UI

1. Add the kind to `TimelineEvent["kind"]` union in `EventTimeline.tsx`.
2. Add a matching entry to `KIND_STYLES` in `EventTimeline.tsx`.
3. Add the SDK listener in `ConversationComponent.tsx` that calls `addEvent({ kind: ..., ... })`.
4. Verify: `bun run verify:web`.

## Adjust session parameters (codec, VAD)

1. Edit the `parameters` dict (e.g. `output_audio_codec`) or `turn_detection` config in `Agent.start()`.
2. Verify: `bun run verify:local:fastapi`.

## Run / debug locally

```bash
bun run dev              # both processes
bun run doctor:local     # check creds + .env.local before a live call
```

## Verify before finishing

| Change touches…              | Run                                                                 |
| ---------------------------- | ------------------------------------------------------------------- |
| Web only                     | `bun run verify:web`                                                |
| Backend logic / agent config | `bun run verify:backend` + `cd server && pytest tests -v`           |
| Route/proxy boundary         | `bun run verify:web:proxy` and/or `bun run verify:local:fastapi`    |
| Anything end-to-end (local)  | `bun run verify:local`                                              |

## Deploy

1. Deploy `web/` as a Next.js app.
2. Deploy `server/` (or any reachable FastAPI host); the published backend-only image is `ghcr.io/AgoraIO-Conversational-AI/recipe-agent-events` on `v*` tags.
3. Set `AGENT_BACKEND_URL` in the web deployment so rewrites reach the backend.

## Related Deep Dives

- [event_taxonomy](L2/event_taxonomy.md) — full RTM event payloads and `TimelineEvent` mapping.
- [session_lifecycle](L2/session_lifecycle.md) — client-side join/renewal/teardown.

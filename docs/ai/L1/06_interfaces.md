# 06 · Interfaces

> Boundary contracts: backend routes, the `/api/*` rewrite map, env vars, the response envelope, event taxonomy, and the `TimelineEvent` type.

## Backend routes (port 8000)

The browser calls these as `/api/<name>`; Next rewrites to the backend `/<name>`.

### `GET /get_config`

- Query (optional): `channel?: string`, `uid?: int` (≤ 0 or missing → backend generates one).
- Returns `data`: `{ app_id, token, uid (string), channel_name, agent_uid (string) }`.
- Token is a Token007 RTC+RTM token, expiry 3600s, for a concrete non-zero UID.

### `POST /startAgent`

- Body: `{ channelName: string, rtcUid: int, userUid: int, parameters?: object }`.
  - `parameters.output_audio_codec?: string` is the only honored parameter field.
- Returns `data`: `{ agent_id, channel_name, status: "started" }`.
- 400 if `channelName`/`rtcUid`/`userUid` invalid, or if `Agent` failed to initialize.

### `POST /stopAgent`

- Body: `{ agentId: string }`.
- Returns `{ code: 0, msg: "success" }` (no `data`).

## Response envelope

```json
{ "code": 0, "msg": "success", "data": { } }
```

`data` omitted when the route has no payload. Non-zero `code` or missing `data` = error on the client side.

## Rewrite map (`web/next.config.ts`)

| Browser path        | Backend destination |
| ------------------- | ------------------- |
| `/api/get_config`   | `/get_config`       |
| `/api/startAgent`   | `/startAgent`       |
| `/api/stopAgent`    | `/stopAgent`        |

`rewrites()` returns `[]` when `AGENT_BACKEND_URL` is unset. The contract is asserted by `verify-api-contracts.ts` and exercised by `verify-local-proxy.ts`.

## Browser API client (`web/src/services/api.ts`)

- `getConfig({ channel?, uid? }) → GetConfigResponse`
- `startAgent(channelName, rtcUid, userUid) → agent_id`
- `stopAgent(agentId) → void`

## Environment variables

| Variable                | Scope          | Required | Default         |
| ----------------------- | -------------- | :------: | --------------- |
| `AGORA_APP_ID`          | backend        |    ✅    | —               |
| `AGORA_APP_CERTIFICATE` | backend        |    ✅    | —               |
| `OPENAI_MODEL`          | backend        |          | `gpt-4o-mini`   |
| `OPENAI_API_KEY`        | backend        |          | — (optional BYO key) |
| `AGENT_GREETING`        | backend        |          | built-in line   |
| `AGENT_BACKEND_URL`     | web (deploy)   |  ✅\*   | `http://localhost:8000` (dev) |
| `PORT`                  | backend (env only) |      | `8000` — do **not** put in `.env.example` |

\* Required wherever the web app is deployed; rewrites are empty without it.

## RTM event taxonomy

All events arrive over RTM and are converted to `TimelineEvent` by `ConversationComponent`. Delivered via `AgoraVoiceAI` from `agora-agent-client-toolkit`.

| SDK event (`AgoraVoiceAIEvents.*`) | `TimelineEvent.kind` | Payload fields used         |
| ---------------------------------- | -------------------- | --------------------------- |
| `AGENT_STATE_CHANGED`              | `"state"`            | `event.state` (listening / thinking / speaking / idle) |
| `AGENT_METRICS`                    | `"metric"`           | `metrics.type`, `metrics.name`, `metrics.value` (ms)   |
| `AGENT_ERROR`                      | `"error"`            | `error.type`, `error.message`, `error.code`             |
| `MESSAGE_ERROR`                    | `"error"`            | `error.code`, `error.message`                           |
| `TRANSCRIPT_UPDATED`               | `"turn"`             | most recent item with text; role inferred from uid      |

## `TimelineEvent` type (`web/src/components/EventTimeline.tsx`)

```typescript
export type TimelineEvent = {
  id: string;        // crypto.randomUUID()
  ts: number;        // Date.now() at append time
  kind: "state" | "metric" | "error" | "turn";
  label: string;
  detail?: string;
};
```

Buffer is capped at 50 events (`.slice(-50)`). Import `TimelineEvent` from `EventTimeline.tsx`, not from a separate types file.

## Agent session parameters (set in `Agent.start()`)

| Key                    | Value                    | Why                                               |
| ---------------------- | ------------------------ | ------------------------------------------------- |
| `audio_scenario`       | `"chorus"`               | Ultra-low-latency profile for web clients.        |
| `data_channel`         | `"rtm"`                  | Routes all events over RTM to the browser.        |
| `enable_error_message` | `True`                   | Surfaces agent-side errors over RTM.              |
| `enable_metrics`       | `True`                   | Emits per-stage latency metrics over RTM.         |
| `advanced_features`    | `{"enable_rtm": True}`   | Enables the RTM event channel.                    |
| `output_audio_codec`   | optional string          | Forwarded from `POST /startAgent` `parameters`.   |

## Related Deep Dives

- [event_taxonomy](L2/event_taxonomy.md) — full event payloads and subscription wiring detail.

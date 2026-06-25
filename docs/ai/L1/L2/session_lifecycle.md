# Deep Dive — Session Lifecycle

**When to Read This:** You are touching client-side join, token renewal, RTC/RTM wiring, transcript handling, or mid-call control. For the contracts these calls hit, see [06_interfaces](../06_interfaces.md).

The browser owns the full RTC/RTM client lifecycle; the backend owns tokens and the agent session. The two meet only at `/api/*`.

## End-to-end flow

1. **Config** — `LandingPage.tsx` calls `getConfig()` (`web/src/services/api.ts`) → `GET /api/get_config`. Backend mints a Token007 (RTC+RTM, 3600s) for a concrete non-zero UID and returns `{ app_id, token, uid, channel_name, agent_uid }`.
2. **Parallel start** — `LandingPage.tsx` runs `startAgent(...)` and RTM login concurrently via `Promise.all`. RTM login uses `new AgoraRTM.RTM(appId, uid)` → `login` → `subscribe(channel)`.
3. **Join** — `ConversationComponent.tsx` joins the RTC channel with the returned token/UID, publishes the microphone. `AgoraVoiceAI.init(...)` is called after `isReady && joinSuccess` and `ai.subscribeMessage(channel)` starts the event stream.
4. **Converse** — user audio flows through `DeepgramSTT → OpenAI → MiniMaxTTS`; agent voice returns into the channel. RTM events feed the `EventTimeline` and annotated transcript.
5. **Stop** — `handleEndConversation` in `LandingPage.tsx`: calls `stopAgent(agentId)`, logs out RTM, tears down AgoraVoiceAI (`ai.unsubscribe(); ai.destroy()`), releases mic track.

## Backend session bookkeeping

`Agent` (`server/src/agent.py`) keeps an in-memory map `self._sessions[agent_id] = session`.

- `stop(agent_id)` pops the session and calls `session.stop()`.
- If the session is missing (e.g. process restarted), it falls back to `self.client.stop_agent(agent_id)` — the stateless cloud path. This is why stop is robust across restarts but `_sessions` itself is **not** a durable store.

## `AgoraVoiceAI` lifecycle

`AgoraVoiceAI` is initialized inside an async `useEffect` guarded by `isReady && joinSuccess`. The cleanup function calls `ai.unsubscribe(); ai.destroy()`. A `cancelled` flag prevents race conditions on strict mode double-mount:

```typescript
let cancelled = false;
const ai = await AgoraVoiceAI.init({ rtcEngine: client, rtmConfig: { rtmEngine: rtmClient }, ... });
if (cancelled) { ai.unsubscribe(); ai.destroy(); return; }
// register listeners ...
ai.subscribeMessage(agoraData.channel);
return () => { cancelled = true; /* cleanup */ };
```

## Transcript handling (`web/src/lib/conversation.ts`)

- `normalizeTranscript(transcript, localUid)` — maps `uid === '0'` to the local UID and runs `normalizeTranscriptSpacing` on text.
- `normalizeTimestampMs(ts)` — promotes second-precision timestamps to ms (multiplies if `ts < 1e12`).
- `getMessageList` — filters out `TurnStatus.IN_PROGRESS` items and maps to `IMessageListItem`.
- `getCurrentInProgressMessage` — returns the single in-progress item (or `null`) for live transcript rendering.
- `mapAgentVisualizerState(agentState, isConnected, connectionState)` — maps SDK state → UIKit visualizer state (`joining`, `listening`, `analyzing`, `talking`, `ambient`, `disconnected`).

## Token renewal

Tokens expire at 3600s. The client renews via `onTokenWillExpire` in `LandingPage.tsx`: re-fetches `get_config` with the same channel + uid, then calls `client.renewToken(rtcToken)` and `rtmClient.renewToken(rtmToken)`. Keep renewal client-side — the backend stays stateless about who is connected.

## Agent connection detection

`ConversationComponent` tracks agent presence via two mechanisms:
- `useClientEvent(client, "user-joined"/"user-left")` — updates `isAgentConnected` as users join/leave the RTC channel.
- `useEffect` on `remoteUsers` — syncs `isAgentConnected` from the current remote users snapshot.

`agentUID` is sourced from `agoraData.agentUid` (returned by `get_config`), falling back to `process.env.NEXT_PUBLIC_AGENT_UID`, then `DEFAULT_AGENT_UID` (123456 from `src/lib/agora.ts`).

## What stays where

- **Client owns:** RTC join, mic publish, RTM login, `AgoraVoiceAI` subscription, event/transcript listeners, token renewal, explicit end-call media release.
- **Backend owns:** token minting, agent session start/stop, all event flag configuration.
- Do not move token logic into the web app or add Route Handlers for it (see [07_gotchas](../07_gotchas.md)).

## Related L1

- [02_architecture](../02_architecture.md) · [03_code_map](../03_code_map.md) · [06_interfaces](../06_interfaces.md)

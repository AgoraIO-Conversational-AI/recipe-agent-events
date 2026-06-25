# Deep Dive — Event Taxonomy and Payloads

**When to Read This:** You are changing event flags, adding a new `TimelineEvent` kind, debugging missing or malformed events, or tracing how an RTM message becomes a rendered timeline entry. For the high-level picture, start at [02_architecture](../02_architecture.md).

## How events reach the browser

1. The backend sets `data_channel="rtm"`, `enable_metrics=True`, `enable_error_message=True`, and `advanced_features={"enable_rtm": True}` in `Agent.start()`.
2. Agora ConvoAI cloud emits RTM messages for every agent lifecycle event.
3. In `ConversationComponent.tsx`, after RTC join + RTM login, `AgoraVoiceAI.init({ rtcEngine, rtmConfig, ... })` is called.
4. `ai.subscribeMessage(channelName)` starts receiving RTM messages.
5. Each SDK event callback calls `addEvent(...)` to append a `TimelineEvent` to the 50-slot buffer.
6. `EventTimeline` re-renders with the updated buffer (displayed in reverse-chronological order).

## SDK event → `TimelineEvent` mapping

### `AGENT_STATE_CHANGED`

Callback signature: `(agentUserId: string, event: { state: AgentState }) => void`

States: `"listening"`, `"thinking"`, `"speaking"`, `"idle"`, `"silent"`

```typescript
ai.on(AgoraVoiceAIEvents.AGENT_STATE_CHANGED, (_, event) => {
  setAgentState(event.state);           // drives QuickstartTranscriptPanel header
  addEvent({ kind: "state", label: String(event.state) });
});
```

Also drives `mapAgentVisualizerState` → `AgentVisualizer` UIKit state.

### `AGENT_METRICS`

Callback signature: `(agentUserId: string, metrics: { type: string; name: string; value: number }) => void`

`value` is in milliseconds. `type` identifies the pipeline stage (e.g., `"stt"`, `"llm"`, `"tts"`); `name` is the metric name within that stage.

```typescript
ai.on(AgoraVoiceAIEvents.AGENT_METRICS, (_, metrics) => {
  setAgentMetrics((prev) => [...prev, metrics].slice(-8));  // drives QuickstartPipelineMetrics
  addEvent({
    kind: "metric",
    label: `${metrics.type}: ${metrics.name}`,
    detail: `${Math.round(metrics.value)}ms`,
  });
});
```

### `AGENT_ERROR`

Callback signature: `(agentUserId: string, error: { type: string; code: unknown; message: string; timestamp: number }) => void`

```typescript
ai.on(AgoraVoiceAIEvents.AGENT_ERROR, (agentUserId, error) => {
  addConnectionIssue({ source: "agent", ... });
  addEvent({ kind: "error", label: "agent error", detail: `${error.type}: ${error.message}` });
});
```

### `MESSAGE_ERROR`

Callback signature: `(agentUserId: string, error: { code: unknown; message: string; timestamp: number }) => void`

```typescript
ai.on(AgoraVoiceAIEvents.MESSAGE_ERROR, (agentUserId, error) => {
  addConnectionIssue({ source: "rtm", ... });
  addEvent({ kind: "error", label: "message error", detail: String(error.message ?? error.code) });
});
```

### `TRANSCRIPT_UPDATED`

Callback signature: `(transcript: TranscriptHelperItem[]) => void`

Only the most recent item with non-empty text is appended to the timeline. Role is inferred from uid: `uid !== "0"` → `"agent"`, else `"user"`.

```typescript
ai.on(AgoraVoiceAIEvents.TRANSCRIPT_UPDATED, (t) => {
  setRawTranscript([...t]);
  const latest = [...t].reverse().find((item) => item.text);
  if (latest?.text) {
    const role = latest.uid && latest.uid !== "0" ? "agent" : "user";
    addEvent({ kind: "turn", label: role, detail: String(latest.text).slice(0, 80) });
  }
});
```

## `TimelineEvent` type

Defined and exported from `web/src/components/EventTimeline.tsx`:

```typescript
export type TimelineEvent = {
  id: string;        // crypto.randomUUID()
  ts: number;        // Date.now() at append
  kind: "state" | "metric" | "error" | "turn";
  label: string;     // short human-readable name
  detail?: string;   // optional value or truncated text
};
```

Buffer cap: `setEvents((prev) => [...prev, evt].slice(-50))`.

## Raw RTM signaling fallback

In addition to `AgoraVoiceAI` callbacks, `ConversationComponent` also listens to `rtmClient.addEventListener("message", ...)` directly to catch `message.error` and `message.sal_status` payloads that the SDK may not surface. These are dispatched to `addConnectionIssue` only — not to the `EventTimeline`.

## Backend event flag reference

All flags are set in `server/src/agent.py` `Agent.start()`:

| Flag | Location | Value |
| ---- | -------- | ----- |
| `data_channel` | `parameters` dict | `"rtm"` |
| `enable_metrics` | `parameters` dict | `True` |
| `enable_error_message` | `parameters` dict | `True` |
| `enable_rtm` | `advanced_features` dict | `True` |

No event logic exists in `server.py` — all flag setting is in `agent.py`.

## Adding a new event kind

1. Add the kind to the `TimelineEvent["kind"]` union in `EventTimeline.tsx`.
2. Add a matching entry to `KIND_STYLES` (badge + dot colors) in `EventTimeline.tsx`.
3. Add the `AgoraVoiceAIEvents.*` listener in `ConversationComponent.tsx` that calls `addEvent(...)`.
4. If the event requires a new backend flag, add it to `parameters` or `advanced_features` in `Agent.start()`.

## Related L1

- [02_architecture](../02_architecture.md) · [06_interfaces](../06_interfaces.md) · [07_gotchas](../07_gotchas.md)

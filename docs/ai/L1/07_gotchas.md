# 07 · Gotchas

> Non-obvious pitfalls specific to the events recipe. Read before changing the agent, env, or verify scripts.

## `server/.env.example` has stale translator vars

The file ships `SOURCE_LANG`, `TARGET_LANG`, `TTS_VOICE`, and an `AGENT_GREETING` value that says "I'll translate." Those lines are **unused** by the events recipe's `agent.py`. They are artifacts from a copy-paste and should be ignored. The vars the backend actually reads are `AGORA_APP_ID`, `AGORA_APP_CERTIFICATE`, `OPENAI_MODEL`, `OPENAI_API_KEY` (optional), and `AGENT_GREETING`.

## Do not put `PORT` in `server/.env.example`

`verify:local:fastapi` injects a random `PORT` and loads env with `load_dotenv(override=True)`. A `PORT` line in `.env.example` (copied to `.env.local`) would clobber the injected port and break the smoke test.

## Keep `/api/*` ownership in rewrites

Adding `web/app/api/**/route.ts` for agent/token logic breaks the boundary — `verify-api-contracts.ts` explicitly fails if a `route.ts` exists under `app/api`. Token logic belongs in `server/`.

## `TimelineEvent` type is in `EventTimeline.tsx`, not a types file

`TimelineEvent` is exported from `web/src/components/EventTimeline.tsx`. Do not extract it to a separate types file — `ConversationComponent.tsx` imports it from there.

## Event flags belong in `Agent.start()`, not in the route

`data_channel`, `enable_metrics`, `enable_error_message`, and `advanced_features.enable_rtm` are set once in `server/src/agent.py`. Do not move them into `server.py` or the web client.

## `OPENAI_API_KEY` is truly optional

Unlike BYO-key recipes, this recipe runs without `OPENAI_API_KEY`. The server boots and agents start with only `AGORA_APP_ID` and `AGORA_APP_CERTIFICATE`. Raising a `ValueError` at boot for a missing `OPENAI_API_KEY` would break zero-key deployments.

## camelCase request fields

`StartAgentRequest` uses `channelName`, `rtcUid`, `userUid` (camelCase) to match the browser client. Renaming one side without the other breaks the contract tests.

## UID normalization in transcripts

`normalizeTranscript` maps `uid === '0'` to the local UID. Token issuance also rejects zero/negative UIDs and generates a concrete one. Preserve both — speaker mapping and tokens depend on concrete UIDs.

## Local calls under a global proxy

Global proxies (Clash, etc.) can break `localhost`/RFC-1918 traffic. Configure the proxy to send `127.0.0.1`, `localhost`, and private ranges DIRECT, or `socksio` (in `requirements.txt`) plus `all_proxy` to route the backend through SOCKS.

## Related Deep Dives

- [event_taxonomy](L2/event_taxonomy.md) — correct event flag wiring and subscription setup.

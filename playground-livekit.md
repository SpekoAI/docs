# How LiveKit is wired into the Playground

End-to-end: three processes meet in a LiveKit room. The dashboard browser tab, our API server, and a voice agent worker. LiveKit is the audio transport — we don't proxy audio.

## Actors

1. **Browser** (`apps/dashboard` /playground → `@spekoai/client` → `livekit-client` SDK)
2. **API server** (`apps/server`, `livekit-server-sdk`) — mints tokens, dispatches agents, never touches audio
3. **Agent worker** (`apps/worker-ts`, `@livekit/agents`) — a Node process registered with LiveKit Cloud; joins rooms on demand and runs STT→LLM→TTS

## Session create flow

User hits **Start** in the playground. Browser sends `POST /v1/sessions` to our API with the pipeline config (`intent`, `constraints`, `voice`, `systemPrompt`, …).

Server does four things, in order (`apps/server/src/routes/sessions.ts`):

1. **Generate IDs.** `sessionId = uuid`, `roomName = speko_${sessionId}`, `identity = user_${uuid}`.
2. **Mint a LiveKit JWT** via `LiveKitTokenService` (`services/livekit-token.ts`). Uses `AccessToken` from `livekit-server-sdk` with grants `roomJoin / canPublish / canSubscribe / canPublishData`, TTL 900s by default. Token metadata embeds `{ sessionId, organizationId }`. Minted *before* the DB insert so a mint failure never leaves an orphan row.
3. **Persist** a `voiceSession` row with the full pipeline config (status: `created`).
4. **Dispatch the agent.** `LiveKitDispatchService` (`services/livekit-dispatch.ts`) calls `AgentDispatchClient.createDispatch(roomName, agentName, { metadata })`. Metadata is the full pipeline config as JSON — this is how config flows from the API request to the worker without any extra network hop. If dispatch fails, the row is marked `failed` and we return 500.

Response to the browser: `{ conversationToken, livekitUrl, roomName, identity, expiresAt }`.

## Browser joins the room

Dashboard calls `VoiceConversation.create({ conversationToken, livekitUrl, ...callbacks })` (`packages/client/src/voice-conversation.ts`). Internally, `WebRTCConnection` uses `livekit-client` to connect straight to LiveKit Cloud. **Our server is out of the audio path.** Mic track is published, remote agent track is subscribed. Transcripts + status events arrive over LiveKit's data channel and fan out to the `onMessage` / `onStatusChange` / `onModeChange` callbacks the playground passed in.

## Agent worker picks up the dispatch

`apps/worker-ts/src/agent.ts` boots with `@livekit/agents` `cli.runApp(new ServerOptions({ agent, agentName: LIVEKIT_AGENT_NAME }))`. The `agentName` is what links the dispatch → this worker: server calls `createDispatch(..., agentName, ...)`, LiveKit routes a job to a worker registered under that name.

Per job (`defineAgent({ entry })`):

1. Pulls `ctx.job.metadata` — the JSON the server dispatched.
2. Zod-parses it. If missing/invalid → `ctx.shutdown()`. No silent defaults: playground requires explicit config, so bad metadata kills the job rather than spinning up a wrong-pipeline session.
3. Calls `createSpekoComponents({ speko, vad, intent, constraints, voice, llm, ttsOptions })` from `@spekoai/adapter-livekit`. This returns LiveKit-compatible `stt`, `llm`, `tts` objects that internally hit our Speko SDK (`POST /v1/transcribe`, `/complete`, `/synthesize`) — which is where the benchmark-driven provider routing actually happens.
4. Builds `new voice.AgentSession({ vad, stt, llm, tts })` + `new voice.Agent({ instructions: systemPrompt })`, then `session.start({ room: ctx.room })` and `ctx.connect()`. Silero VAD is prewarmed once per process.
5. Fires `session.generateReply({ instructions: 'Greet the user…' })` so the agent speaks first.

## How config travels

Browser form → `POST /v1/sessions` body → stored in `voiceSession.pipelineConfig` + serialized into dispatch metadata → `ctx.job.metadata` on the worker → parsed → `createSpekoComponents`. **LiveKit dispatch metadata is the channel** — no extra API call from worker back to server for config.

## What LiveKit gives us vs what we own

| Concern | Owner |
|---|---|
| Audio transport, WebRTC, room lifecycle, SFU | LiveKit |
| Worker dispatch + agent framework | LiveKit (`@livekit/agents`) |
| VAD, turn detection, interruption handling | `@livekit/agents` + Silero plugin |
| Auth (JWT grants, TTL) | Us — `LiveKitTokenService` |
| Pipeline config + provider selection | Us — server + Speko SDK routing |
| STT/LLM/TTS adapters bridging LiveKit ↔ Speko | Us — `@spekoai/adapter-livekit` |

## Teardown

Browser calls `conv.endSession()` → `livekit-client` disconnects. Agent `AgentSession` sees the room empty, the job ends. The `VoiceSession` row stays for audit/usage. Playground's unmount effect also runs `endSession()` so navigating away doesn't leak a room.

## Files to point them at

- `apps/server/src/routes/sessions.ts` — the orchestrator
- `apps/server/src/services/livekit-token.ts` — JWT minting
- `apps/server/src/services/livekit-dispatch.ts` — agent dispatch
- `apps/worker-ts/src/agent.ts` — what actually runs in the room
- `packages/adapter-livekit/src/{stt,llm,tts}.ts` — the LiveKit ↔ Speko bridge
- `packages/client/src/{voice-conversation,webrtc-connection}.ts` — browser side

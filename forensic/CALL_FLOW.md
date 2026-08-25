# Call Flow — End-to-End Telephony Reference

Companion to [`CONTRIBUTION_DOSSIER.md`](./CONTRIBUTION_DOSSIER.md). This traces how a call moves
through the system with file:line citations, for interview deep-dives on the call state machine.
Repo: `lawyer-appointment/src/features/call-center`. All facts are `[VERIFIED]` from code unless noted.

---

## Actors & correlation keys
- **`exotel_call_sid`** — the provider's call id; unique index on `call_logs`; primary idempotency key.
- **`custom_field`** — set on outbound calls to route downstream branching:
  `consultation_<id>`, `call_center_outbound`, `call_center_lawyer_confirmation`
  (`exotel-integration.service.ts:127`).
- **`call_metadata`** (JSON) — carries `dtmf_priority`, `internal_recording_url` (S3 key),
  `api_verified_at`, claim markers.

---

## A. Outbound call (agent → client, or lawyer confirmation)
1. `CallController.connectAgentWithClient` (`call/call.controller.ts:324`) →
   `CallService.connectAgentWithClient` (`call/call.service.ts:1046`).
2. Transport chosen by agent capability: SIP/WebRTC if `agent.voip_enabled && agent.sip_id`, else
   PSTN (`call.service.ts:1085-1092`).
3. `ExotelIntegrationService.initiateOutboundCall` (`exotel/exotel-integration.service.ts:38`):
   - SIP path sends `from.user_contact_uri = agentSipId` (`:120`).
   - PSTN path first `validateOrCreateContact` (`:94`), then `from.contact_uri`.
   - **v3 with pre-connect `playback[]`** when `EXOTEL_PLAYBACK_ENABLED` + PSTN
     (`:168-186`, `exotel-api.service.ts:567 makeCallV3`); otherwise **v2** `makeCall`
     (`:189`, `exotel-api.service.ts:449`).
4. Call log created with the returned `callSid` via `CallService.initiateCall`
   (`call.service.ts:37`); duplicate SID → `ConflictError` (`:63-69`); agent flipped to `on_call`
   with `current_call_id` if available (`:87`).
5. Status callbacks registered → `/exotel/webhook/call-details` for
   terminal/answered/completed/cancelled (`exotel-integration.service.ts:135-152`).
6. DND/NDNC (TRAI code 10717) → typed `DndError` → HTTP 403 `DND_BLOCKED`
   (`exotel-api.service.ts:550-556`, controller `call.controller.ts:380`).

---

## B. Inbound call (Exotel CCM Programmable Connect — poll-driven router)
1. Exotel polls `ExotelController.handleCCMProgrammableConnect` (`exotel/exotel.controller.ts:307`)
   repeatedly while responses carry `fetch_after_attempt: true`. It parses
   `CallStatus/CallType/DialCallStatus/numbers/digits`, strips quoted DTMF (`:332`), logs one
   greppable line/poll (`:337`), then routes. On error, returns a safe
   `fetch_after_attempt:false` fallback (`:366`).
2. `ExotelWebhookService.handleIncomingCallRouting` (`exotel/exotel-webhook.service.ts:475`) →
   single-flight wrapper → `routeCall` (`:527`) → `routeCallAtomic` (`:543`) or
   `routeCallLegacy` (`:1039`) depending on `CCM_ATOMIC_CLAIM`.

### B.1 Single-flight (per-CallSid) — `CcmRoutingLock`
- Redis `SET NX`, 15s TTL (`ccm-routing-lock.ts:19-26`). Winner routes and caches its response;
  losers poll `getCachedResponse` (from Redis **master**, `:90`) up to 4×200ms and replay the
  identical destination (`exotel-webhook.service.ts:499-511`).
- **Degraded mode:** Redis down → `{acquired:true, degraded:true}` → proceed without single-flight;
  the atomic DB claim (B.2) still prevents double-assignment (`:487-494`).

### B.2 Atomic agent claim + guarded assignment
- Candidate agents ranked, then claimed with a conditional
  `UPDATE call_center_agents SET status='on_call' WHERE id=? AND status='available'`
  (`CallService.claimAgentForCall`, `call.service.ts:111`). Lost claim = normal contention → try
  next candidate (`exotel-webhook.service.ts:720-734`). Agent is claimed **before** its destination
  is returned (`:637-640`).
- Post-claim `call_logs` update re-checks `call_status NOT IN (ANSWERED,RINGING)` AND
  `agent_id IN (null, claimedAgent)` (`:748-770`); `affected!==1` → release claim, re-read winner,
  replay routing (`:772-783`). DB-level guard also in `createOrUpdateIncomingCall`
  (`call.service.ts:331-357`), with `UniqueConstraintError` fallthrough for concurrent CREATE
  (`:300-318`).

### B.3 DTMF priority routing
- Exotel sends `digits` only on the first poll, so the pressed priority is persisted to
  `call_metadata.dtmf_priority` and re-read every subsequent poll
  (`exotel-webhook.service.ts:627-655`). Priority narrowing with **spillover**: keep the caller in
  that priority for `PRIORITY_SPILLOVER_ATTEMPTS=2` polls, then spill to any agent (`:687-701`).

### B.4 Routing response
- `buildAgentRoutingResponse` (`:924`) returns CCM JSON: `destination`, `sticky_agent`,
  `record:true`, `recording_channels:"dual"`, `max_ringing_duration:30`,
  `max_conversation_duration:1800`, and `dial_passthru_event_url` → `/webhook/call-events` (`:944`).

---

## C. Live state transitions (webhook ingestion)
Two ingestion families:

### C.1 Passthru events → `/call-events`
`ExotelController.handleWebhookEvents` (`exotel.controller.ts:224`) → `processCallEvent`
(`exotel-webhook.service.ts:1511`), dispatched by `EventType`:
`Dial` (`:1606`), `Ringing` (`:1643`), `Answered` (`:1660`), `Terminal/Completed/Cancelled`
→ `handleTerminalEvent` (`:1690`). Completion detection is deliberately broad: any payload without
EventType carrying `DialCallStatus||RecordingUrl||DialCallDuration` is treated as final (`:1561`).

### C.2 Enhanced status callbacks → `/call-details`
`handleWebhookCallDetails` (`exotel.controller.ts:273`) → `processEnhancedCallEvent` (`:2390`),
reading `call_details.call_sid` (v2) or `.sid` (v3) (`:2403`) and dispatching to
`handleEnhancedAnswered/Terminal/Ringing/Dial`.

### C.3 The status resolver (single source of truth)
All vocabularies collapse into **`resolveCallStatus`** (`:3399`), direction- and duration-aware:
- unanswered from-leg → `MISSED` (inbound) vs `NO_ANSWER` (outbound) (`:3434`);
- duration ≤30s → `SHORT_CALL` (`:3412`);
- unknown status → `FAILED` + alert (`:3508`) — naive default had been `COMPLETED`.
IST timestamp fix: `parseExotelTimestamp` (`:3142`) appends `+05:30` to naive Exotel strings.
Writes go through `CallService.updateCallStatus` (`call.service.ts:390`), which flips the agent back
to `available` on terminal via `updateAgentStatusOnCallChange` (guarded on
`current_call_id === callId`, `:1019`).

### C.4 Agent-hop correction at terminal
`DialWhomNumber` in a Terminal event is authoritative; if the DB `agent_id` differs (a hop bypassed
the guards), it corrects both the call log and the auto-disposition and fires `alert.high`
(`:1703-1778`).

### C.5 v3 API re-verification (dual source of truth)
After each terminal write, `setImmediate(verifyCallStatusViaApi)` (`:2003,:2582,:1826`) fetches
authoritative details from the v3 API and overwrites status/duration/recording if the API "knows
better" (`:3523-3596`), guarded once by `call_metadata.api_verified_at`. Webhooks are treated as
possibly stale.

---

## D. Recording → S3 migration (`recording-migration.service.ts`)
- Triggered from the recording webhook (`exotel-webhook.service.ts:259-261`) and from any
  `updateCallStatus` that writes a `recording_url` (`call.service.ts:427-432`).
- `schedule()` runs via `setImmediate` **after** the HTTP response flushes — zero hot-path latency
  (`:99-108`).
- Download: axios `arraybuffer`, HTTP Basic auth, `rejectUnauthorized:false` (Exotel self-signed
  cert), 60s timeout, in-memory (no disk) (`:26-41`).
- Idempotency: check `internal_recording_url` before download, then **re-read the DB after download**
  and bail if another process finished first (`:138-151`). S3 key
  `<env>/recordings/<callSid>.mp3` in bucket `vs-call` (`:14-17,:129`).
- Failure bookkeeping: `recordFailure` increments attempts; after `MAX_MIGRATION_ATTEMPTS=3` sets
  `recording_migration_failed=true` (`:59-88`).
- Read path: `CallController.withPresignedRecordingUrl` swaps the stored key for a short-lived
  presigned S3 URL in the response only (DB untouched), falling back to the Exotel URL on S3 error
  (`call.controller.ts:29-42`).
- Bulk: `migrateRecordings` (`call.controller.ts:498`), bounded concurrency
  `MIGRATE_ALL_CONCURRENCY=5`, max 500/run, 6-month age cutoff.

---

## E. AI QA transcription (`ai/gemini-transcription.service.ts`)
1. `processRecording` (`:288`) guards on config + API key (returns silently, non-blocking).
2. Reads the S3 key from `call_metadata.internal_recording_url` (`:313`) — requires D to have run.
3. Concurrency guard: bail if a `pending|processing` row exists (`:324`).
4. Context resolution (`resolveCallContextType`, `:52`): `agent_lawyer` / `client_lawyer` /
   `agent_client`.
5. Version = `MAX(version)+1` (`call-transcription.model.ts:119-128`); insert a **pending row first**
   (`:341`) snapshotting model + per-1k pricing; flip to `processing` (`:359`).
6. `s3Service.downloadBuffer` → base64 → Gemini `@google/genai` with
   `responseMimeType:'application/json'`, `temperature:0`, audio as `inlineData` (`:385-410`).
7. Token/cost from `usageMetadata` (`:413-417`); robust JSON extraction surfacing
   `finishReason`/`blockReason` on failure (`:427-432`).
8. Persist `completed` with `transcription[]`, `analysis{summary, agent_performance_score 1-5,
   feedback, areas_for_improvement[], sentiment_analysis}`, tokens, `estimated_cost_usd` (`:441-451`).
9. Failure → flip the pre-created row to `failed` with `error_message` (`:457-467`), never re-thrown.
Recovery: `transcription.service.ts reprocessPending` (`:166`) revives rows stuck past `staleMinutes`
with a per-recording failed-attempt ceiling and bounded concurrency.

---

## F. Consultation conversion & lawyer transfer (`consultation/consultation-integration.service.ts`)
- **Call → consultation:** `convertCallToConsultation` (`:162`) creates a consultation (`stages:
  CREATED`, provenance metadata) then links back via `updateCallStatus`. Two writes, **no
  transaction** (partial-failure risk, see dossier §10).
- **Initiate from consultation:** `initiateCallFromConsultation` (`:49`) sets `ONGOING`, calls
  `exotelService.initiateOutboundCall`, emits `emitConsultationCallStarted`. (Availability check is
  commented out — §10.)
- **Transfer to lawyer:** `transferConsultationCall` (`:389`) finds the active call, resolves the
  target lawyer→agent, `exotelService.transferCall`, appends a `transferHistory[]` entry, emits
  `emitCallTransferred`.
- **Completion:** `completeConsultationCall` (`:317`) ends the call, sets `COMPLETED`, sets the agent
  back to `AVAILABLE` (frees the claim). Post-consult order/billing runs elsewhere against the
  consultation's `post_consult_*` columns.

---

## G. AI-call claim (`ai-call/ai-call.service.ts`)
- Ingest `handleCompletedAiCall` (`:18`): two-tier idempotency on `exotel_call_sid` (app pre-check +
  `UniqueConstraintError` catch); creates an unclaimed (`agent_id: NULL`, `agent_type:'ai'`) log,
  broadcasts to available agents.
- Claim `claimAiCall` (`:70`): friendly pre-checks, then the **atomic guard**
  `UPDATE … SET agent_id=? WHERE id=? AND agent_id IS NULL AND agent_type='ai'`; `affectedRows===0`
  → `ConflictError` (409). Same-agent re-claim = idempotent success. Recovery list
  `getUnclaimedAiCalls` (`:137`) for agents who missed the socket broadcast.

---

## H. Realtime push (socket.io)
Emitted from the webhook service through `callCenterSocketServiceCore`:
- `emitIncomingCall(agentId, …)` — agent popup on routing (`exotel-webhook.service.ts:807,1436`).
- `emitCallUpdate({callId, status, …})` — `assigned/queued/missed/ringing/dialing/answered/
  completed/disposition_required`; outbound-answered carries `autoOpenModal:true` (`:2496`).
- `emitAgentStatusChange` on every status write (`:2186`).
- Queued/unassigned calls broadcast with sentinel `agentId:'unassigned'` (`:1019-1025`).

---

## I. Background reconciliation (Jenkins/endpoint-triggered)
- `cleanupStaleCalls` (`exotel-webhook.service.ts:3158`): QUEUED/INITIATED/RINGING/ANSWERED older
  than 20min (bounded ≤24h) → fetch v3 API for truth, else QUEUED→MISSED / else→FAILED, releasing
  agent claims on terminal resolution (`:3227`).
- `backfillStuckCallDispositions` (`:3303`): creates missing dispositions without mutating
  `call_status`.
- Daily assignment-counter reset via node-cron at 06:00 IST (`:73`).

---

## State reference

**`call_logs.call_status`** (`call-log.model.ts:17`):
`initiated → queued → ringing → answered → completed | short_call | missed | failed | busy |
no_answer | cancelled`. Active = `{initiated, queued, ringing, answered}` (`:66-68`).

**`call_center_agents.status`** (`call-center-agent.model.ts:12`):
`available → on_call → after_call_work → break → offline`.

**`consultations.stages`** (`consultation.model.ts:10-20`):
`created → scheduled → ongoing → completed | networkDisconnected | notConnected | cancelled |
lawyerNotFound | clientNotFound`.

**`call_transcriptions.status`** (`call-transcription.model.ts:34`):
`pending → processing → completed | failed`, versioned per `(call_log_id, version)`.

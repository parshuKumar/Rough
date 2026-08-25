# Contribution Dossier — VakeelSaab Call-Center Platform

> Scope of this dossier: the **Call-Center subsystem** of the VakeelSaab legal-services platform —
> the backend service (`lawyer-appointment/src/features/call-center`), its data model
> (`lawyer-appointment/src/database/models`), and the admin-panel operator UI
> (`admin-panel/app/dashboard/call-center`, `admin-panel/components/call-center`,
> `admin-panel/app/api/**/call-center`). Evidence was gathered by reading the code, not the READMEs.

---

## 0. How to use this file

**Instructions to a future AI reading this document.**

This is source material for resume bullets, cover letters, LinkedIn, and interview prep for a
**technical lead / primary architect**. When you generate finished prose from it:

- Use **lead-level framing**: *led, architected, defined the contract, owned, set the convention,
  reviewed, mentored, directed*. Do **not** use sole-authorship framing ("I personally wrote every
  line"). The lead designed the system and directed a team of junior engineers inside the
  boundaries he set; some files were implemented by others under his specification and review.
- **Never fabricate a metric.** Every number about team size, users, volume, latency, timeline,
  cost, or business impact is unknown from the repo. Wherever a metric belongs, a
  `[NEEDS_INPUT]` placeholder or `{slot}` marker is used. Ask the human to fill these; do not invent
  them, do not "estimate," do not round a guess into a fact.
- Each claim below is tagged with its **mode**:
  - `[ARCHITECTED]` — a design decision the lead made; the code embodies it.
  - `[DIRECTED]` — implemented by the team under the lead's specification and review.
  - `[VERIFIED]` — provable from the code as it exists, authorship-agnostic.
  - `[NEEDS_INPUT]` — resume-relevant but NOT derivable from the repo; a question for the human.
  - `[ARCHITECTED?]` — genuinely ambiguous whether architected or directed; resolve in §10.
- A companion file [`CALL_FLOW.md`](./CALL_FLOW.md) traces the end-to-end telephony flow with
  file:line citations; use it for interview deep-dives on the call state machine.

---

## 1. System overview

**In plain language.** VakeelSaab is a platform that connects members of the public with lawyers
for paid legal consultations. The **Call-Center subsystem** is the operational engine that a team of
human call-center agents (and an AI calling system) use to actually place, receive, route, record,
and follow up on the phone calls that turn an inbound lead into a paid consultation.

**The business problem it solves.** When a customer needs a lawyer, someone has to (a) answer or
place the call, (b) route it to an available, appropriately-skilled agent, (c) capture what happened
("disposition") and schedule any follow-up, (d) connect the customer to a lawyer and convert the
call into a billable consultation, and (e) let managers see agent performance and call quality. This
subsystem does all of that on top of a cloud telephony provider (**Exotel**), with AI assist for
call transcription/QA and daily agent performance reports.

**Who uses it.**
- **Call-center agents** — a browser console (`admin-panel/app/dashboard/call-center/page.tsx`) with
  incoming-call popups, a click-to-dial VoIP softphone, a personal work queue, and disposition entry.
- **Call-center managers / supervisors** — dashboards for call logs, live operations, agent
  performance, and a monitoring "command center."
- **The AI calling system** — an automated caller that creates call records agents can *claim*.
- **Customers and lawyers** — indirectly, as the two parties a call connects.

**Where it sits in the stack.** It is a feature module inside the `lawyer-appointment` Node/Express
service, mounted at `/api/v1/call-center` (`lawyer-appointment/src/app.ts:96`) and exposed to the
browser through an API gateway prefix `/g/cc/…` and a Next.js server-side proxy layer in the
admin-panel. It shares the platform's MySQL database (Sequelize), Redis, and socket.io server with
the rest of the product (consultations, users, documents, cases). `[VERIFIED]`

---

## 2. Technical profile

### Stack `[VERIFIED]`
- **Backend:** Node.js, **TypeScript**, **Express** (mounted under Express 5-aware router tagging),
  **Sequelize ORM** over **MySQL**. Path aliases (`@core`, `@database`, `@features`).
- **Realtime:** **socket.io** (server) / **socket.io-client** (browser), three namespaces
  (agent, manager, disposition).
- **Cache / distributed coordination:** **Redis** (`ioredis`), reader/writer split, used as a
  distributed lock and distributed counter.
- **Telephony:** **Exotel** — Voice API **v2 and v3**, **CCM Programmable Connect** (poll-driven
  inbound routing), and the **Exotel WebRTC IP-calling web SDK** for in-browser softphone.
- **AI:** **Google Gemini** (`@google/genai`) for audio transcription + QA analysis and daily
  reports; **Anthropic Claude** (`@anthropic-ai/sdk`) and **OpenAI** (`openai`, Whisper + GPT) as
  pluggable alternate providers.
- **Storage:** **AWS S3** (`@aws-sdk/client-s3` + `s3-request-presigner`) for durable call
  recordings.
- **Frontend:** **Next.js App Router**, **React 18**, **Zustand** (agent console state), **SWR**
  (manager/analytics data), **OpenReplay** (session telemetry).
- **Ops:** **New Relic** APM (guarded so local startup survives its absence, `app.ts:7-19`),
  **node-cron** + `setInterval`/`setTimeout` schedulers.

### Real counts (from the repo, this subsystem only) `[VERIFIED]`
| Metric | Count | Source |
|---|---|---|
| Backend call-center TypeScript | **~28,200 LOC** across **131 files** | `src/features/call-center` |
| Data model (call-center) | **9 Sequelize models, ~2,430 LOC** | `src/database/models/*.model.ts` |
| Distinct DB tables owned/used | **~10** (see §3) | model `tableName` |
| DTOs (request validation objects) | **37** | `src/features/call-center/**/dto` |
| Route files / HTTP endpoints | **13 files / 134 endpoints** | `*.routes.ts` verb count |
| Admin-panel call-center UI | **~35,000 LOC .tsx**, **96 components** | `app/dashboard/call-center` + `components/call-center` |
| Admin-panel BFF/proxy routes | **53 route handlers, ~4,360 LOC** | `app/api/**/call-center` |
| External integrations | **8+** (Exotel, Gemini, Claude, OpenAI, S3, Redis, socket.io, New Relic, OpenReplay) | `package.json` + code |
| Automated tests (this subsystem) | **1 real concurrency harness** + 1 test spec doc | `exotel/__tests__/ccm-race-condition.test.ts` |

> Note on LOC: line counts are a size signal, not a productivity claim, and include code
> implemented by the team under direction. See §0 on framing.

### Architectural pattern `[VERIFIED]`
Feature-module monolith with **strict per-feature layering**. Every sub-feature under
`src/features/call-center/<domain>/` repeats the same shape: `*.routes.ts` → `*.controller.ts` →
`*.service.ts`, with `dto/`, and where relevant `services/`, `config/`, `events/`, `providers/`,
`prompts/`, `__tests__/`. The module publishes a single barrel (`index.ts`) that re-exports services,
controllers, DTOs, and one aggregated `callCenterRoutes` router. The frontend mirrors this with a
**BFF/proxy layer** (`app/api/**/call-center`) between browser and gateway. This uniformity across
13 domains is itself the primary leadership fingerprint (§4).

---

## 3. Architecture and the decisions behind it

This is the highest-value section. Each decision: the decision, the constraint that forced it, the
naive alternative rejected, and where it lives.

### 3.1 Data model — a call-centric star with multi-channel dispositions `[ARCHITECTED]`
**Decision.** A normalized relational model centered on `call_logs`, with 9 tables:
`call_center_agents`, `call_logs`, `call_dispositions`, `follow_up_tasks`, `agent_activity_logs`,
`call_notes`, `call_transcriptions`, `agent_daily_reports`, plus the shared `consultations` bridge.
Associations are declared centrally in `src/database/models/index.ts:105-266`.

**Non-obvious design choices, each with the naive alternative it rejects:**

- **`call_dispositions` is deliberately multi-channel, not call-bound.** `call_log_id` is
  **nullable** and the table carries `source` (call/manual/…), `direction`, `user_id`,
  `customer_number` (`call-disposition.model.ts:19-70,334-341`). Naive alternative: force every
  disposition to belong to a call row. Rejected because agents must be able to log an outcome for a
  contact that never produced a telephony call (manual entry, cross-channel). The model even
  encodes this: `isManualEntry` returns true when `call_log_id === null` (`:180-182`). `[ARCHITECTED]`

- **Self-referential reschedule chains.** `call_dispositions.parent_disposition_id` and
  `follow_up_tasks.parent_task_id` are self-FKs (`call-disposition.model.ts:170-178`,
  `follow-up-task.model.ts:330-331`). A reschedule doesn't mutate history — it **creates a child**
  and marks the parent complete (`CallDisposition.reschedule()`, `:257-291`;
  `FollowUpTaskService.complete()` spawns a child in-transaction, `follow-up-task.service.ts:232-320`).
  Naive alternative: overwrite `follow_up_date` in place. Rejected because it destroys the audit
  trail of *how many times* a customer was rescheduled — which the model then exposes as first-class
  business logic: `isChronicRescheduler = reschedule_count >= 3` (`:188-190`), used to auto-deprioritize.

- **Append-only audit metadata as structured JSON.** `call_dispositions.metadata` carries
  `categoryChanges[]` and `priorityChanges[]` arrays, each entry `{timestamp, agentId, old, new,
  reason}` (`call-disposition.model.ts:54-65`; written in `disposition.service.ts:223-255,406-450`).
  Naive alternative: a single mutable category column. Rejected in favor of an append-only trail so a
  manager can reconstruct *who* reclassified a call and *why*. `[ARCHITECTED]`

- **Versioned transcriptions, never overwritten.** `call_transcriptions` has a unique index on
  `(call_log_id, version)` and a `getNextVersion() = MAX(version)+1` helper
  (`call-transcription.model.ts:119-128,260`). Re-running QA analysis writes v2, v3 … and GET returns
  the latest. Naive alternative: one transcription row per call, updated in place. Rejected so prior
  AI analyses (and their cost accounting) remain auditable. `[ARCHITECTED]`

- **Rich indexing chosen for the actual query shapes.** `call_dispositions` carries 15 indexes,
  including the composite `(agent_id, resolution_status, priority, created_at)`
  (`call-disposition.model.ts:485-502`) that exactly serves the Work Queue V2 ORDER BY. `call_logs`
  carries composites `(call_type, call_status)` and `(agent_id, initiated_at)`
  (`call-log.model.ts:258-284`). This is index design driven by read patterns, not defaults. `[ARCHITECTED]`

- **Timestamps stored UTC; "a day" is an IST business day.** Business logic converts in SQL with
  `DATE(CONVERT_TZ(col,'+00:00','+05:30'))` (`work-queue-v2.helpers.ts:265-267`) and reconstructs
  IST day-edges as absolute UTC (`istDayBoundaries`). Naive alternative: host-local `setHours(0,0,0)`.
  Rejected because it drifts ~5.5h and breaks "due today." (Legacy paths still use the naive form —
  a known divergence flagged in §10.) `[ARCHITECTED]`

### 3.2 The CCM double-assignment race — the signature engineering problem `[ARCHITECTED]`
**Constraint.** Exotel's CCM Programmable Connect endpoint is **poll-driven**: while a response
carries `fetch_after_attempt: true`, Exotel calls the same connect URL *concurrently* for the same
`CallSid`. A naive "search agents → pick one → write `agent_id`" flow lets two concurrent polls
assign the same call to two different agents ("call hopping") or double-book one agent.

**Decision — three layered defenses**, all feature-flag gated for staged rollout
(`CCM_ATOMIC_CLAIM`, `CCM_SINGLE_FLIGHT`), living in `exotel/exotel-webhook.service.ts`,
`call/call.service.ts`, and `exotel/ccm-routing-lock.ts`:

1. **Atomic DB claim.** `CallService.claimAgentForCall` issues a conditional
   `UPDATE call_center_agents SET status='on_call' WHERE id=? AND status='available'` and treats
   `affectedRows===1` as the sole winner (`call.service.ts:111`). MySQL row locking guarantees
   exactly one concurrent winner. The agent is claimed **before** the destination is returned, so a
   crash mid-flow leaves a claimable-but-unassigned call, never a split brain.
2. **Guarded assignment write.** Even post-claim, the `call_logs` update re-checks
   `call_status NOT IN (ANSWERED,RINGING) AND agent_id IN (null, claimedAgent)`
   (`exotel-webhook.service.ts:748-770`); a lost guard releases the claim and replays the true
   winner's routing.
3. **Per-CallSid single-flight distributed lock.** `CcmRoutingLock` uses Redis `SET NX` with a 15s
   TTL (`ccm-routing-lock.ts:19-26`); the winner routes and caches the response, losers poll the
   cache and **replay the identical destination**. Reads come from the Redis *master*
   (`getFromMaster`) because replica lag would defeat a sub-second race window.

**Degraded-mode design.** If Redis is unavailable, `acquire()` returns
`{acquired:true, degraded:true}` and the flow proceeds *without* single-flight — Defense 1 still
prevents double-assignment (`ccm-routing-lock.ts:63`, `exotel-webhook.service.ts:487-494`). The
system degrades safely rather than failing closed.

**Naive alternative rejected:** check-then-act agent selection (explicitly removed, see the
"replaced" comments at `call.service.ts:280-291`) and/or wrapping everything in a single SQL
transaction. Instead the atomicity is pushed into **single-statement conditional UPDATEs**, which is
why the codebase deliberately uses almost no multi-row transactions.

**Proven, not asserted.** `exotel/__tests__/ccm-race-condition.test.ts` is a standalone ts-node
harness that runs against a **real MySQL** (mocks would prove nothing about row locking): 10
concurrent claims → exactly one winner; 5 concurrent inbound creates → one row; Guard-3 blocks
overwrite on ANSWERED; the Redis single-flight replay. It refuses to run under `NODE_ENV=production`.
`[VERIFIED]`

### 3.3 API contract style — layered REST behind a gateway and a BFF `[ARCHITECTED]`
**Decision.** 134 REST endpoints across 13 domain routers, aggregated in
`call-center/index.ts` and mounted once at `/api/v1/call-center`. The browser never calls the
backend directly: it goes through a Next.js **BFF** (`app/api/**/call-center`) that forwards the
user's Bearer token to the gateway prefix `/g/cc/…`. Webhooks from Exotel are a *separate* public
router (`/webhooks`, `/exotel`). Naive alternative: expose the service directly to the browser and
sprinkle telephony webhooks among authenticated routes. Rejected to (a) hide the internal base URL,
(b) allow-list params before they become cache keys, (c) keep unauthenticated webhook ingress
isolated. `[ARCHITECTED]` (frontend BFF layering — see §10 for the ARCHITECTED/DIRECTED split).

### 3.4 State management — call status as a single resolver over 3 vendor vocabularies `[ARCHITECTED]`
**Decision.** All of Exotel's inconsistent status vocabularies (legacy passthru, v2, v3) collapse
into one `resolveCallStatus()` (`exotel-webhook.service.ts:3399`) that is **direction- and
duration-aware**: e.g. an unanswered from-leg → `MISSED` inbound vs `NO_ANSWER` outbound; ≤30s →
`SHORT_CALL`; unknown → `FAILED` + alert (naive default had been `COMPLETED`). Frontend keeps its own
status vocabulary and maps both ways (`mapServerStatusToClient`). Naive alternative: store Exotel's
raw strings. Rejected because downstream billing/QA logic needs one coherent state machine. `[ARCHITECTED]`

### 3.5 Auth model `[VERIFIED]` / `[ARCHITECTED?]`
**Two layers.** Core JWT authentication (`core/middleware/auth.middleware.ts`) verifies the Bearer
token, loads the user, and attaches `req.permissions` **carried inside the JWT**. A call-center
authorization middleware (`call-center/middleware/call-center-auth.middleware.ts`) then resolves the
user to a `CallCenterAgent` row by `user_id` and attaches `req.agent`, with escalating strictness:
`authorizeCallCenter` → `requireAgent` → `requireActiveAgent` (rejects `offline`). The frontend adds
route-level `PermissionGuard` + feature-level `useHasPermission` gating (manager vs agent), even
reflowing table columns by permission. Inbound telephony webhooks are guarded at the router by an
`INTERNAL_API_SECRET` (git commit `7e72902`), *not* by JWT. See §10 for the auth gaps to own honestly.

### 3.6 Error-handling & resilience strategy `[ARCHITECTED]`
**Decision — "never let an external dependency take down the hot path."** Consistent patterns:
- Every webhook handler **responds 200 even on internal failure** so Exotel stops retrying
  (`exotel.controller.ts:260,294`).
- Heavy side-effects (recording download/S3 migration, auto-disposition, transcription) run in
  `setImmediate`/fire-and-forget IIFEs **after the HTTP response flushes** — zero hot-path latency
  (`recording-migration.service.ts:99-108`; `exotel-webhook.service.ts:2013,2778`).
- Both AI pipelines are **non-blocking**: errors are caught, the row is flipped to `failed` with an
  `error_message`, and nothing is re-thrown (`gemini-transcription.service.ts:457-467`).
- Aggregations use `Promise.allSettled` so one failed sub-query degrades to 0 instead of 500-ing a
  whole dashboard (`work-queue-v2.service.ts:283`).

Naive alternative: synchronous processing inside the webhook. Rejected because Exotel would retry on
timeout and the agent-facing latency would spike.

### 3.7 Async / background work `[VERIFIED]`
Registered schedulers: a **node-cron** daily reset of assignment counters at 06:00 IST
(`exotel-webhook.service.ts:73`); an **hourly `setInterval`** overdue-task sweep (`app.ts:239`); a
**self-rescheduling `setTimeout`** daily-report generator at ~02:00 IST (`app.ts:190-225`); a
Jenkins/endpoint-triggered **stale-call reconciliation** that reconciles calls stuck >20min against
the Exotel v3 API. The deliberate mix (cron vs interval vs timeout) is discussed in §10 (no
distributed lock → double-fire risk on horizontal scale). `[ARCHITECTED?]`

### 3.8 Consistency / transactional guarantees `[VERIFIED]`
The system's stance is explicit: **atomicity via single-statement conditional UPDATEs**, not
multi-row transactions. The one place a real `sequelize.transaction()` is used is task
complete-and-spawn-child, with **events emitted only after commit** so they can't fire on
rolled-back work (`follow-up-task.service.ts:238-289`). This is a coherent, defensible choice for the
concurrency profile — and its limits (cross-row invariants unprotected) are named in §10.

---

## 4. Technical leadership evidence

Concrete, cited fingerprints of leadership. Consistency across many files written by different hands
is treated, per instruction, as a decision the lead imposed.

### 4.1 Contracts and abstractions others had to build against
- **The `AiProvider` strategy interface.** A single-method contract
  `generateContent(options): Promise<AiProviderResponse>` (`daily-report/providers/ai-provider.interface.ts:26-28`)
  normalizes `{systemPrompt, userMessage, mimeType, maxOutputTokens}` in and `{text, tokens}` out.
  Three vendor adapters (Gemini/Claude/OpenAI) each hide their SDK's quirks behind that identical
  signature; a `createAiProvider(...)` factory (`providers/index.ts:13-26`) selects one **per pipeline
  slot** (analyst/reporter/validator) by env. This is a textbook Strategy+Factory the team's report
  pipeline was required to consume. `[ARCHITECTED]`
- **The prompt registry + versioning contract.** Prompts self-register into a keyed `Map`
  (`role:versionId`) via import side-effects, with a pre-flight `validateCompatibility()` that diffs
  the actual data object's field paths against each prompt's declared `requires_data_fields`
  (`prompts/prompt-registry.ts`). Every report stamps its active prompt versions into metadata and
  into an HTML comment — per-artifact provenance. `[ARCHITECTED]`
- **The atomic-claim primitive as a reusable pattern.** The conditional-UPDATE claim appears in
  three independent places — agent claim (`call.service.ts:111`), callback claim
  (`claimCallbackCall`, `:162-230`, via `JSON_SET`/`JSON_EXTRACT ... IS NULL`), and AI-call claim
  (`ai-call.service.ts:100-116`). The same concurrency contract is imposed wherever "exactly one
  winner" is required. `[ARCHITECTED]`
- **The `CcmRoutingLock` distributed-lock abstraction** (`exotel/ccm-routing-lock.ts`) — a small,
  documented lock/cache/invalidate API with an explicit degraded-mode contract that callers rely on.
  `[ARCHITECTED]`

### 4.2 Enforced conventions (the repeated pattern = the imposed decision)
- **`routes → controller → service → dto` layering repeated across all 13 domains.** No controller
  talks to the DB directly; no route contains logic. This uniformity across ~131 files is the
  clearest evidence of an imposed standard. `[ARCHITECTED]`
- **DTO-per-operation validation.** 37 DTOs; request shape is validated at the edge, not in
  services. `[ARCHITECTED]`
- **Model conventions:** every model declares typed `Attributes`/`CreationAttributes` interfaces,
  computed getters (`isActive`, `isOverdue`, `overallScore`…), instance methods for state
  transitions, `underscored` snake_case columns, explicit index blocks, and paranoid soft-delete
  where retention matters (`call_center_agents`). The shape is identical model to model. `[ARCHITECTED]`
- **The BFF 5-step block** (read auth header → `buildApiUrl` → forward Bearer → relay status →
  `OPTIONS` handler) repeated near-verbatim across ~33 route files — a convention so consistent it
  was later refactored into a catch-all proxy with an allow-list (`app/api/proxy/call-center`). The
  repetition *is* the standard. `[ARCHITECTED?]` (frontend authorship — see §10)
- **Timezone discipline:** UTC storage + explicit IST conversion in the V2 helpers, applied
  consistently in the new code paths. `[ARCHITECTED]`

### 4.3 How the work was structured to be delegated
The module boundaries are **feature folders with a single public barrel**
(`call-center/index.ts`). Each of the 13 domains (agent, call, disposition, exotel, consultation,
call-notes, activity, transcription, daily-report, consolidated, voip, ai-call, socket) is a
self-contained unit with its own routes/controller/service/dto — a clean seam along which parallel
work could be assigned to different engineers without merge collisions. The daily-report subsystem
goes further, isolating `providers/`, `prompts/`, `services/` (collector/analyst/generator/validator/
renderer) as independently ownable slots. `[ARCHITECTED]`

### 4.4 Quality mechanisms
- **A real concurrency test against real MySQL** (`ccm-race-condition.test.ts`) — the hardest part
  of the system (the claim race) is the part that has an executable proof, with a production guard.
  `[VERIFIED]`
- **Defense-in-depth validation:** availability checks re-done in both controller and service;
  disposition uniqueness enforced by both a pre-check *and* a `UniqueConstraintError` catch. `[VERIFIED]`
- **Prompt/schema compatibility pre-flight** as an observability guard in the AI pipeline (§4.1).
- **Staged rollout via feature flags** (`CCM_ATOMIC_CLAIM`, `CCM_SINGLE_FLIGHT`, `CALL_STATUS_V2`,
  `AI_NOTES_ENABLED`, `EXOTEL_PLAYBACK_ENABLED`) with legacy paths kept side-by-side and marked for
  deletion after the new path soaks — a deliberate migration discipline. `[ARCHITECTED]`

### 4.5 Developer-experience & onboarding investment
Per-feature `README.md` files (call-notes, consultation, exotel), an `EXOTEL_CONFIGURATION.md`, an
`exotel-api.example.ts`, an agent-connection test spec, and a completion summary doc
(`VS-331-aiCallLogClaim-COMPLETE-SUMMARY.md`). Design decisions are referenced by path in code
comments (e.g. `docs/race-condition/IMPLEMENTATION_PLAN.md`, `docs/work-queue-backend-driven/PLAN.md`)
— evidence that architecture was written down before implementation and pointed to from the code. `[VERIFIED]`

---

## 5. Contributions by domain

Only domains actually touched. Each: problem, what was built (with paths), mode, complexity signals.

### Architecture `[ARCHITECTED]`
- **Problem:** turn a telephony provider + a shared DB into a coherent, delegatable call-center.
- **Built:** the feature-module layering, the 9-table data model, the barrel/router aggregation, the
  BFF boundary, the flag-gated migration strategy. See §3, §4.
- **Complexity signals:** distributed locking, conditional-UPDATE concurrency, state-machine design,
  multi-vendor abstraction, staged rollout.

### Backend (telephony core) `[ARCHITECTED]` / `[DIRECTED]` (mixed)
- **Problem:** drive call state off an unreliable, poll-based, multi-version webhook stream.
- **Built:** `exotel-webhook.service.ts` (~3,600 LOC) — CCM routing (`routeCallAtomic` /
  `routeCallLegacy`), passthru + enhanced v2/v3 event ingestion, `resolveCallStatus`, DTMF priority
  routing persisted to `call_metadata.dtmf_priority` with spillover, v3 API re-verification of
  webhook truth, stale-call reconciliation. `call.service.ts` (~1,700 LOC) — claim primitives,
  status updates, agent status coupling. `exotel-api.service.ts` — v2/v3 clients with typed error
  mapping (DND/NDNC → 403). See [`CALL_FLOW.md`](./CALL_FLOW.md).
- **Complexity signals:** idempotency/replay on retries, agent-hop correction at terminal, race
  handling, external-API failure isolation, dual webhook formats.

### Frontend `[DIRECTED]` / `[ARCHITECTED?]` (mixed — lead did "some")
- **Problem:** give agents a real softphone console and managers real dashboards.
- **Built:** agent console with Zustand "focused-workflow-is-the-screen" state machine
  (`app/dashboard/call-center/page.tsx`), incoming-call notification with 30s countdown + background-
  tab escalation + dedup (`components/call-center/IncomingCallNotification.tsx`), Exotel WebRTC
  softphone (`components/call-center/VoIPCallController.tsx` + `lib/voip/exotel-webrtc.service.ts`),
  the agent **My Work** queue (V2), manager **Call Logs**, **Live-Ops**, **Command Center**,
  **Agent Performance**, and the AI-transcription modal state machine.
- **Complexity signals:** websocket + polling fallback (visibility-aware), optimistic claim updates
  with 409 handling, three pagination strategies (numbered / IntersectionObserver infinite /
  SWR-infinite), debounced dependent filters, permission-driven UI reflow, single-tab session lock.
- **Mode note:** the user states he "did some things on the frontend." Two clearly-built surfaces per
  his direction are **My Work** and **Manager → Call Logs**; the exact per-file split needs his input
  (§10).

### Data / DB `[ARCHITECTED]`
- **Problem:** model calls, dispositions, follow-ups, agents, QA, and reports without losing history.
- **Built:** the 9 models, self-referential chains, append-only JSON audit trails, versioned
  transcriptions, query-shaped composite indexes, IST-aware date logic. See §3.1.
- **Complexity signals:** audit trails, versioning, soft delete, backwards-compatible nullable-FK
  evolution (`call_log_id` "Changed from false", `call-disposition.model.ts:336`).

### API design `[ARCHITECTED]`
- **Built:** 134 endpoints, DTO validation, the `/g/cc` gateway + BFF proxy, isolated public webhook
  routers, catch-all proxy with param/endpoint allow-listing.
- **Complexity signals:** pagination contracts, allow-listing before cache-key formation, CORS
  handling, per-endpoint cache policy.

### Auth / Security `[VERIFIED]` / `[ARCHITECTED?]`
- **Built:** JWT-in-permissions model, agent-row resolution middleware with 3 strictness levels,
  frontend route + feature RBAC, `INTERNAL_API_SECRET` on inbound webhooks, S3 presigned-URL reads so
  recording keys aren't exposed, HTML sanitization in the report renderer/validator (strip
  script/iframe/on*=/external src). See §10 for the honest gaps (webhook signature verification wired
  only to stub handlers; a couple of non-atomic assign paths).
- **Complexity signals:** RBAC, PII masking (phone masking in reports; `[REDACTED]` prompt
  instruction), XSS defense in generated HTML.

### Performance `[ARCHITECTED]`
- **Built:** the Work Queue V2 **stats-in-2-queries** design (all 5 tab counts + ~17 cards via
  `SUM(CASE WHEN <bucketSql>)` conditional aggregation, reusing identical bucket fragments so a badge
  can never disagree with its list, `work-queue-v2.service.ts:176-344`); `findAndCountAll({distinct,
  subQuery:false})` to avoid JOIN row-multiplication; in-memory `Map`/`Set` batch lookups in the
  report collector to avoid N+1; bounded-concurrency (`Promise.allSettled`, batch=5) recording
  migration and transcription reprocessing.
- **Complexity signals:** N+1 elimination, conditional aggregation, bounded concurrency, native
  TINYINT-rank sorting vs `CASE` weight for enum priority.

### DevOps / CI-CD `[VERIFIED]`
- **Built:** New Relic guarded init, Jenkins-triggered reconciliation/reset endpoints, feature-flag
  gating, env-driven provider/model selection. (Full CI config not in the read scope — §10.)

### Testing `[VERIFIED]`
- **Built:** the real-MySQL concurrency harness (§4.4). Coverage is otherwise thin — named honestly
  in §10.

### Integrations `[ARCHITECTED]` / `[DIRECTED]` (mixed)
- **Built:** Exotel (v2/v3/CCM/WebRTC), Gemini, Claude, OpenAI, S3, Redis. Each behind a service
  boundary with typed errors and failure isolation.
- **Complexity signals:** self-signed-cert recording downloads (`rejectUnauthorized:false`, a
  deliberate, flagged trade-off), token/cost accounting per AI call, retry/backoff delegated to
  Exotel's own polling.

### Developer Experience / Documentation `[ARCHITECTED]` / `[VERIFIED]`
- **Built:** per-feature READMEs, Exotel config/example docs, path-referenced design docs, completion
  summaries. See §4.5.

---

## 6. Scale and complexity indicators

**What the repo can prove (authorship-agnostic):** `[VERIFIED]`
- ~28,200 LOC backend + ~39,000 LOC frontend (UI + BFF) for this subsystem alone.
- 9 tables, 134 endpoints, 37 DTOs, 96 UI components, 53 BFF handlers, 8+ external integrations.
- Concurrency-critical paths with an executable race test.
- 4 AI pipeline stages, 3 interchangeable AI providers, prompt versioning.
- 3 background scheduler mechanisms; multi-channel disposition + reschedule-chain domain model.

---

### `[NEEDS_INPUT]` — metrics only the human can supply

> These are resume-critical and **cannot** be derived from the repo. Do not guess them.

1. **Team:** How many engineers did you lead on this, and their seniority mix? `{team of N}`
2. **Composition:** Which sub-systems did juniors own under your spec (e.g. dashboards, DTO layers)?
3. **Duration:** Start and end dates / total elapsed time for the call-center build. `{X months}`
4. **Users:** How many call-center agents use it in production? How many managers? `{N agents}`
5. **Volume:** Calls handled per day/month in production? `{N calls/day}`
6. **Manual process replaced:** What did agents do before this (spreadsheets? a third-party dialer?
   manual dialing?) and what did that cost in time/money? `{X hours saved}` / `{₹ saved}`
7. **Reliability:** Production uptime, or the incident rate before/after the CCM race fix?
8. **AI impact:** Did automated QA/transcription replace manual call auditing? By how much? `{N hours}`
9. **Adoption:** Was this adopted beyond the original call-center team (other regions/BUs)?
10. **Cost:** Monthly AI (Gemini/Claude/OpenAI) and telephony spend, if you track it via the
    `estimated_cost_usd` columns you built.

---

## 7. Hard skills evidence table

| Skill | Proof (file / config / pattern) | Depth |
|---|---|---|
| Distributed systems / concurrency | Redis `SET NX` single-flight `ccm-routing-lock.ts`; conditional-UPDATE claims `call.service.ts:111`, `ai-call.service.ts:100-116` | core |
| Race-condition analysis & testing | `exotel/__tests__/ccm-race-condition.test.ts` (real-MySQL harness) | core |
| Relational data modeling | 9 models, self-FK chains, composite indexes, audit JSON `src/database/models/*` | core |
| Telephony / CTI integration | Exotel v2/v3 + CCM + WebRTC `exotel/*`, `lib/voip/exotel-webrtc.service.ts` | core |
| State-machine design | `resolveCallStatus` `exotel-webhook.service.ts:3399`; consultation/disposition/task lifecycles | core |
| API design (REST) | 134 endpoints, DTO validation, gateway + BFF `call-center/index.ts`, `app/api/**` | core |
| Software architecture / abstraction | `AiProvider` strategy+factory `daily-report/providers/*`; prompt registry | core |
| Applied AI / LLM engineering | Gemini audio→structured QA `gemini-transcription.service.ts`; multi-provider pipeline; cost tracking | core |
| Performance engineering | stats-in-2-queries `work-queue-v2.service.ts:176-344`; N+1 elimination; bounded concurrency | core |
| Caching & invalidation | CCM response cache/invalidate `ccm-routing-lock.ts`; per-endpoint BFF cache policy | core |
| Cloud storage | S3 recording migration + presigned reads `recording-migration.service.ts`, `call.controller.ts:29-42` | core |
| Realtime systems | socket.io 3-namespace design; websocket+polling fallback `socket/*`, `lib/socket/*` | core |
| Auth / RBAC | JWT-in-permissions + agent-row middleware `call-center-auth.middleware.ts`; frontend PermissionGuard | core |
| React / Next.js frontend | Zustand workflow state machine, SWR-infinite, WebRTC softphone `app/dashboard/call-center/*` | core |
| Idempotency & resilience | webhook 200-on-failure, fire-and-forget side-effects, `Promise.allSettled` degradation | core |
| TypeScript (typed domain) | typed model Attributes/CreationAttributes, typed provider contracts throughout | core |
| Timezone-correct engineering | `CONVERT_TZ` IST-day SQL, absolute-UTC edge reconstruction `work-queue-v2.helpers.ts` | core |
| Background job scheduling | node-cron + interval + self-rescheduling timeout `app.ts:190-246`, `exotel-webhook.service.ts:73` | peripheral |
| Technical documentation (leadership) | per-feature READMEs, EXOTEL_CONFIGURATION.md, path-referenced design docs | peripheral |
| Code-review / convention enforcement | uniform routes→controller→service→dto across 13 domains; repeated BFF template | core |

---

## 8. Raw resume-bullet feedstock

Atomic facts, one per line, tagged by mode. Placeholders mark every missing metric. These are inputs
for bullet-writing, not finished bullets.

1. Architected the call-center subsystem of a legal-consultation platform: ~28k LOC backend, 9 DB tables, 134 REST endpoints. [ARCHITECTED]
2. Designed a call-centric relational data model with multi-channel dispositions decoupled from calls (nullable call_log_id). [ARCHITECTED]
3. Solved a telephony double-assignment race with a 3-layer defense: atomic DB claim, guarded write, Redis single-flight lock. [ARCHITECTED]
4. Wrote conditional-UPDATE optimistic-concurrency claims guaranteeing exactly-one-winner across agent, callback, and AI-call flows. [ARCHITECTED]
5. Built a Redis SET-NX distributed lock with master-read cache replay and a safe degraded mode when Redis is down. [ARCHITECTED]
6. Authored a real-MySQL concurrency test harness proving single-winner claims under 10 concurrent racers. [VERIFIED]
7. Designed a single call-status resolver unifying 3 Exotel status vocabularies, direction- and duration-aware. [ARCHITECTED]
8. Built poll-driven CCM inbound routing with DTMF-priority persistence and spillover after N attempts. [ARCHITECTED]
9. Added v3-API re-verification that overrides stale webhook status/duration/recording after each terminal event. [ARCHITECTED]
10. Implemented stale-call reconciliation reconciling calls stuck >20min against the provider API. [DIRECTED]
11. Designed an `AiProvider` strategy interface with Gemini/Claude/OpenAI adapters selectable per pipeline slot by env. [ARCHITECTED]
12. Built a 4-stage AI daily-report pipeline (collect → analyst → deterministic renderer → validator) with token/cost accounting. [ARCHITECTED]
13. Replaced an LLM HTML generator with a deterministic TypeScript renderer to eliminate output drift and token cost. [ARCHITECTED]
14. Built a tiered report validator that applies free programmatic fixes first and only spends tokens on residual issues. [ARCHITECTED]
15. Implemented a prompt registry with version pinning and pre-flight data-schema compatibility checks. [ARCHITECTED]
16. Built a Gemini audio→structured-QA transcription pipeline with versioned rows and per-row cost snapshots. [ARCHITECTED]
17. Enforced transcription idempotency via a unique (call_log_id, version) index plus pending/processing guards. [ARCHITECTED]
18. Built a stuck-transcription reprocessor with a per-recording failed-attempt ceiling and bounded concurrency. [DIRECTED]
19. Designed self-referential reschedule chains for dispositions and follow-up tasks preserving full history. [ARCHITECTED]
20. Auto-deprioritized chronic reschedulers (>=3) to low priority at task creation. [ARCHITECTED]
21. Built append-only JSON audit trails (categoryChanges, priorityChanges) with per-change agent + reason. [ARCHITECTED]
22. Designed Work Queue V2 as backend-driven paginated/filtered/bucketed reads with a stable id tiebreaker. [ARCHITECTED]
23. Computed 5 tab counts + ~17 summary cards in 2 DB queries via conditional aggregation reusing shared bucket SQL. [ARCHITECTED]
24. Eliminated JOIN row-multiplication and N+1s with distinct/subQuery:false and in-memory batch lookups. [ARCHITECTED]
25. Migrated call recordings from Exotel to S3 with idempotency guards, presigned-URL reads, and retry ceilings. [ARCHITECTED]
26. Built a browser softphone on the Exotel WebRTC SDK, dynamically imported to avoid SSR document access. [DIRECTED]
27. Built incoming-call notifications with 30s countdown, dedup, and background-tab title-flash + Notification escalation. [DIRECTED]
28. Implemented optimistic claim UI with first-class 409 handling and socket-push invalidation of stale modals. [DIRECTED]
29. Designed a Next.js BFF proxy forwarding user Bearer tokens to a gateway, later collapsed into an allow-listed catch-all. [ARCHITECTED?]
30. Built a two-layer auth model: JWT-carried permissions + agent-row resolution middleware with 3 strictness levels. [ARCHITECTED?]
31. Added permission-driven UI: route guards, feature gates, and permission-based table-column reflow. [DIRECTED]
32. Enforced UTC-storage / IST-business-day correctness with CONVERT_TZ SQL and absolute-UTC edge reconstruction. [ARCHITECTED]
33. Isolated all external-dependency failures from the webhook hot path via 200-on-failure and post-response side-effects. [ARCHITECTED]
34. Standardized routes→controller→service→dto layering across 13 domains and 131 files. [ARCHITECTED]
35. Sanitized AI-generated report HTML (strip script/iframe/on*=/external src) and masked phone PII. [ARCHITECTED]
36. Shipped features behind flags (CCM_ATOMIC_CLAIM, CCM_SINGLE_FLIGHT, CALL_STATUS_V2, AI_NOTES_ENABLED) with legacy fallbacks. [ARCHITECTED]
37. Led a team of {team of N} engineers over {X months} building the above. [NEEDS_INPUT]
38. Supported {N agents} agents handling {N calls/day} in production. [NEEDS_INPUT]
39. Replaced {manual process} that previously cost {X hours}/{₹}. [NEEDS_INPUT]
40. Reduced misrouted/double-assigned calls from {baseline} to {after} after the CCM race fix. [NEEDS_INPUT]

---

## 9. Interview defense notes

The 10 most impressive items, each with the senior-interviewer follow-up and a code-grounded answer.

1. **The CCM double-assignment race.**
   *Q: Walk me through exactly how two Exotel polls can't double-book an agent.*
   A: The claim is a single conditional `UPDATE … WHERE status='available'`; MySQL row-locks it so
   only one poll sees `affectedRows=1`. On top, a Redis `SET NX` single-flight makes losers replay
   the winner's cached destination, and a guarded `call_logs` write re-checks status+agent before
   assigning. If Redis is down we degrade to just the DB claim, which is still correct. It's proven
   in `ccm-race-condition.test.ts` against real MySQL. `[ARCHITECTED]`

2. **Why almost no SQL transactions?**
   *Q: Isn't that reckless?*
   A: The critical operations are single-row state flips; a conditional UPDATE is atomic by itself
   and cheaper than a transaction under a high-concurrency poll storm. I used a real transaction only
   where I mutate two rows at once — completing a task and spawning its child — and I emit events
   post-commit so nothing fires on a rollback (`follow-up-task.service.ts:238-289`). Cross-row
   invariants beyond that are a known limit (§10). `[ARCHITECTED]`

3. **Multi-provider AI abstraction.**
   *Q: Why not just call Gemini directly?*
   A: I defined a one-method `AiProvider` contract so the report pipeline is vendor-agnostic; each of
   analyst/reporter/validator picks a provider by env. That let me later move the HTML "reporter" off
   the LLM entirely to a deterministic renderer — zero tokens, no drift — without touching the
   pipeline. The specification I gave the team was: honor mimeType, return raw text + token counts.
   `[ARCHITECTED]`

4. **De-AI-ing the report renderer.**
   *Q: You removed AI from an AI feature — why?*
   A: The LLM produced inconsistent CSS/structure per call. Presentation is deterministic; judgment
   isn't. So I kept the LLM for the analyst/validator stages and made rendering a fixed TypeScript
   function with HTML-escaping to kill XSS from agent names/notes. `[ARCHITECTED]`

5. **Work Queue V2 stats in two queries.**
   *Q: How do the tab badges stay consistent with the lists?*
   A: Both the list `WHERE` and the badge `SUM(CASE WHEN …)` are generated from the *same* bucket SQL
   fragment (`bucketSql`/`bucketWhere`), so a badge can't disagree with the list behind it, and I get
   all five counts + the summary cards in one aggregate query per table instead of 15 counts. `[ARCHITECTED]`

6. **Transcription versioning + idempotency.**
   *Q: Two agents trigger QA on the same call at once — what happens?*
   A: A pending/processing guard short-circuits the second, and the real backstop is a unique
   `(call_log_id, version)` index — the loser's insert throws into a catch. Re-analysis creates v2,
   never overwrites v1, so cost history is auditable. `[ARCHITECTED]`

7. **DTMF priority routing under polling.**
   *Q: Exotel only sends the pressed digit on the first poll — how do you keep routing to that
   priority?*
   A: I persist the pressed priority to `call_metadata.dtmf_priority` and re-read it on every
   subsequent poll, with a spillover to any agent after N attempts so a caller isn't stuck forever.
   `[ARCHITECTED]`

8. **Recording durability + security.**
   *Q: Recordings live on Exotel — why move them?*
   A: Exotel URLs expire and purge after ~6 months and use a self-signed cert. I migrate them to S3
   after the response flushes (zero hot-path latency), guard against duplicate downloads by
   re-reading the DB post-download, and serve reads as short-lived presigned URLs so the S3 key never
   leaks. `[ARCHITECTED]`

9. **Timezone correctness.**
   *Q: "Due today" — today where?*
   A: Everything is stored UTC; "a day" is an IST business day, computed in SQL with `CONVERT_TZ`
   and absolute-UTC edge reconstruction, not host-local `setHours`. The older code still used the
   naive form — I flagged that as debt to converge. `[ARCHITECTED]`

10. **The atomic AI-call claim (directed work).**
    *Q: A junior built the claim endpoint — how did you make sure it was correct?*
    A: I gave the spec: no read-then-write; the claim must be a conditional
    `UPDATE … WHERE agent_id IS NULL` and a 0-row result must surface as a 409. Review standard was:
    same-agent re-claim is idempotent success, different-agent is a conflict, and the friendly
    pre-checks never substitute for the atomic guard (`ai-call.service.ts:70-127`). It matches the
    pattern I set for every claim in the system. `[DIRECTED]`

---

## 10. Gaps, weaknesses, and open questions

Where the code is weak or would expose the lead under questioning — own these before an interviewer
finds them.

### Code weaknesses to be ready to discuss `[VERIFIED]`
- **Webhook signature verification is implemented but not wired to live endpoints.**
  `validateWebhookSignature` (HMAC-SHA1) exists and is called only in stub handlers; the *live* CCM /
  passthru / call-details endpoints do no signature check and rely on the router-level
  `INTERNAL_API_SECRET` instead. Defensible, but name it as a hardening TODO.
- **Two non-atomic assign paths remain.** `AgentService.assignCallToAgent` /
  `connectAgentWithClient` are read-check-then-write (TOCTOU), inconsistent with the atomic AI-claim
  path. Same operation, two concurrency postures.
- **Background schedulers have no distributed lock** (`setInterval` overdue sweep, `setTimeout`
  report generator) — they will double-fire on multi-instance deploys.
- **`consolidated.service.ts` queries a `disposition_type` column** while the model uses
  `disposition_category` — likely a stale field silently grouping everything as `unknown`;
  `organizationId` multi-tenant filtering is a TODO throughout.
- **A couple of auth/id inconsistencies:** `optionalAuth` sets `req.userId = user.user_id` vs
  `authenticateToken`'s `user.id`; `disposition.controller.update` reads `req.user?.agentId` where
  the rest reads `req.agent?.id` (so some audit `agentId`s may land undefined).
- **`mock-fallback` in `app/api/call-center/call-notes/route.ts`** returns a fabricated success on
  upstream 404/network error — the UI believes a note saved when the backend never got it. Real
  correctness/observability risk.
- **Cost accuracy gap:** non-Gemini AI models resolve to a null pricing preset and are tracked as $0.
- **Test coverage is thin** beyond the concurrency harness; most services are untested.
- **`rejectUnauthorized:false`** on recording downloads (deliberate, self-signed cert) is a TLS hole.
- **Dead code retained:** the 1,100-line legacy work-queue page and `routeCallLegacy` are kept behind
  flags — intentional during migration, but maintenance surface.

### `[ARCHITECTED?]` ambiguities to resolve (questions for the human)
1. **Frontend authorship split.** You said you "did some things on the frontend." Which surfaces did
   you personally architect vs direct? Confirm: My Work (V2), Manager → Call Logs — you named these.
   What about the agent console workflow state machine, VoIP softphone, Command Center, Live-Ops?
   (Currently marked `[DIRECTED]`/`[ARCHITECTED?]`.)
2. **BFF/proxy convention.** Did you design the BFF pattern and the later catch-all allow-list
   refactor, or did a frontend engineer? (Bullet 29, §4.2.)
3. **Auth model.** Did you design the JWT-in-permissions + agent-row middleware, or inherit the core
   auth from a platform team and build only the call-center authorization layer on top?
4. **The AI daily-report pipeline.** Did you architect the provider abstraction and the
   de-AI-the-renderer decision, or direct a junior who proposed it? (Interview item 3-4 hinges on
   this.)
5. **Scheduler strategy.** Was the cron-vs-interval-vs-timeout mix a deliberate call you made, or
   incidental?
6. **The stale-call reconciliation + Jenkins triggers.** Yours to architect, or ops-team-owned?

Resolve 1–6 and every `[NEEDS_INPUT]` in §6 before this becomes finished resume copy.

---

*Constraint compliance: no credentials, secrets, hostnames, customer data, or proprietary business
logic are reproduced here. All claims are tied to file paths, class/function names, or config lines;
unverifiable numbers are marked `[NEEDS_INPUT]`, not invented.*

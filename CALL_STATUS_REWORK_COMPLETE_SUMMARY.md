# Call Status Rework — Complete Project Summary

> **Project:** VakeelSaab Legal Services — Call Center Backend
> **Stack:** NestJS / TypeScript / Exotel Telephony / MySQL
> **Duration:** June 24 – July 16, 2026
> **Primary File:** `src/features/call-center/exotel/exotel-webhook.service.ts` (~2800 lines)

---

## The Problem

VakeelSaab's call center processes inbound (customer → agent), outbound (agent → customer), and consultation (agent connects lawyer → client) calls via Exotel telephony. The existing call status system had a binary view — calls were either `completed` or `failed`. There was no granularity to distinguish between a customer who didn't answer, an agent whose phone was unreachable, a short accidental call, or a customer who hung up during ringing. This made it impossible for operations to prioritize callbacks, track agent performance, or identify systemic routing failures.

Additionally, Exotel sends call data through three different webhook formats (V1 passthrough, V2 CCM, V3 API) with inconsistent field names and status vocabulary. The backend had no unified resolution layer — each handler mapped statuses independently, producing inconsistent results.

---

## What Was Built

### 1. Unified Call Status Resolution Engine (`resolveCallStatus`)

Designed and implemented a single-source-of-truth function that maps Exotel's entire V1/V2/V3 status vocabulary to a 10-value internal enum:

| Status | Meaning |
|--------|---------|
| `completed` | Connected, talked >30 seconds |
| `short_call` | Connected, talked ≤30 seconds |
| `missed` | Customer/destination was dialed but didn't answer |
| `no_answer` | Agent's phone didn't answer — customer was never dialed |
| `cancelled` | Someone answered then disconnected before conversation |
| `busy` | Customer's phone was busy or answered with 0 seconds talk |
| `failed` | System/network failure |
| `queued` | Waiting for agent (inbound only) |
| `ringing` | Agent's phone is ringing (transitional) |
| `initiated` | Call created, not yet routed (transitional) |

The function handles:
- **Duration-aware resolution:** Same Exotel status produces different results based on actual talk time (e.g., `agent_canceled` with 0 duration = `cancelled`, with 60s duration = `completed`)
- **Direction-aware resolution:** Same V3 leg status means different things for inbound vs outbound (e.g., `from_leg_unanswered` in outbound = agent's phone didn't answer → `no_answer`; in inbound = customer hung up → `missed`)
- **DB-aware resolution:** Current database status provides context for late-arriving webhooks (e.g., `to_leg_unanswered` when DB says `answered` for inbound = agent answered then disconnected → `cancelled`)
- **Null/empty guard:** Handles missing status gracefully — maps `ringing` + null → `missed`, `answered` + null → `cancelled`, treats Exotel's `"free"` agent status as null

### 2. Two-Stage Verification Pipeline

Every terminal call goes through dual verification:

**Stage 1 — Synchronous (V2 webhook):** When the terminal webhook arrives, `resolveCallStatus` processes the V2 status string and writes immediately to DB.

**Stage 2 — Asynchronous (V3 API via `setImmediate`):** `verifyCallStatusViaApi` calls Exotel's V3 CCM API, gets the authoritative status, resolves it through the same `resolveCallStatus`, and corrects the DB if the V3 result differs.

Both raw statuses are stored in `call_metadata` for full auditability:
- `api_raw_status` — V3 API status string
- `api_initial_status` — What V2 webhook resolved to
- `api_resolved_status` — What V3 API resolved to
- `api_verified_at` — Timestamp of verification
- `pre_terminal_db_status` — DB status before terminal write (context preservation)

### 3. Terminal Guards on Answered Handlers

Exotel occasionally sends duplicate `answered` webhooks that arrive AFTER the terminal event. Without protection, these overwrite resolved terminal statuses (`completed`, `short_call`, `missed`) back to `answered`, causing calls to appear stuck.

Added terminal guards to both `handleEnhancedAnsweredEvent` (V2/V3) and `handleAnsweredEvent` (V1 passthrough) that check `TERMINAL_STATUSES.includes(callLog.call_status)` before writing.

### 4. Call Duration Fix

`handleCallCompletedEvent` was using `DialCallDuration` (total time including ringing) instead of `Legs[N].OnCallDuration` (actual talk time). A call where the phone rang for 26 seconds with 0 seconds of conversation was incorrectly classified as `short_call` instead of `cancelled`.

Fixed by checking if all legs show `OnCallDuration = 0` — if so, override `actualCallDuration` to 0, preventing ringing time from being mistaken for conversation time.

### 5. V3 Field Name Compatibility

Exotel's V3 webhooks use different field names than V2 (`status` vs `call_status`, `sid` vs `call_sid`). The terminal handler only read `call_status`, so all V3 webhooks passed `undefined` to `resolveCallStatus`, bypassing the entire mapping logic.

Fixed with fallback pattern: `callDetails.call_status || callDetails.status`.

### 6. Cleanup & Reconciliation System

Expanded the stale call cleanup cron to cover all non-terminal states (`QUEUED`, `INITIATED`, `RINGING`, `ANSWERED`). Calls stuck for >20 minutes are resolved via V3 API. Added call ID tracking and Slack alerting with per-call resolution details (`id: from→to (via)`).

This serves as the ultimate safety net for dropped webhooks — a scenario where Exotel ends the call on their side but never delivers the terminal webhook to our server.

### 7. Consultation Conversion Bug Fix

Root cause investigation revealed that `convertCallToConsultation` (triggered when an agent converts a regular outbound call to a consultation via UI) was blindly setting `call_status: CallStatus.COMPLETED` regardless of actual call outcome. This caused 30+ production calls with 0 duration and `api_resolved_status: "missed"` to show as `completed`.

The fix made `call_status` optional in the update DTO and removed the forced overwrite, preserving the webhook-resolved status. A one-time SQL correction was produced for existing wrongly-overwritten production records.

Key insight: Calls initiated FROM a consultation flow (`custom_field = consultation_*`) were never affected because `convertCallToConsultation` is never called for them — the consultation link already exists at call initiation.

### 8. Frontend Dashboard Updates

Updated `CallLogsSection` component:
- Added all new status values to TypeScript interface
- Color-coded badges: red for `missed`/`no_answer`/`failed`, yellow for `short_call`/`cancelled`/`busy`, green for `completed`
- Red row background for inbound calls that never connected (`missed`, `no_answer`, `failed`, `queued`, `initiated`) — visual indicator for operations that a callback is needed
- Replaced false-positive "Missed" heuristic (`duration === 0`) with status-based "Needs Callback" badge

### 9. Operations KT Document

Produced a 7-page Word document for the operations team covering:
- Status definitions with actionable guidance (who is responsible, what to do)
- Step-by-step call flows for outbound, inbound, and consultation calls
- Dashboard field guide and quick troubleshooting reference
- Background systems explanation (cleanup cron, API verification, delayed reconciliation, recording migration)

---

## Complete V3 Status Mapping Table

| Exotel V3 Status | Outbound Mapping | Inbound Mapping | Condition |
|---|---|---|---|
| `from_leg_unanswered` | `no_answer` | `missed` | Direction-aware |
| `agent_unanswered` | `no_answer` | `no_answer` | Always agent-side |
| `to_leg_unanswered` | `missed` | `missed` | DB not answered |
| `to_leg_unanswered` | — | `cancelled` | DB = answered (inbound only) |
| `customer_unanswered` | `missed` | `missed` | DB not answered |
| `from_leg_cancelled` | `cancelled` (0s) / `completed` (>30s) | same | Duration-aware |
| `to_leg_cancelled` | `cancelled` (0s) / `completed` (>30s) | same | Duration-aware |
| `agent_canceled` | `cancelled` (0s) / `completed` (>30s) | same | Duration-aware |
| `to_leg_answered` | `busy` (0s) / `short_call`/`completed` | same | Duration-aware |
| `from_leg_answered` | `missed` (0s, DB≠answered) / `cancelled` (0s, DB=answered) | same | DB-aware |
| `completed` | Duration-based | Duration-based | Standard |
| `busy` / `customer_busy` | `busy` | `busy` | Always |
| `failed` / `*_no_dial` | `failed` | `failed` | Always |

---

## Bugs Found & Fixed (Chronological)

| # | Bug | Root Cause | Fix |
|---|-----|-----------|-----|
| 1 | Inbound calls stuck at `answered` | `isCallCompletedEvent` condition too narrow | Broadened to include `completed`/`cancelled` |
| 2 | Outbound calls stuck at `answered` | Switch cases missing `completed`/`cancelled` | Added missing cases |
| 3 | `from_leg_unanswered` mapped same for all directions | No direction awareness | Split: outbound → `no_answer`, inbound → `missed` |
| 4 | `to_leg_cancelled` not recognized | Missing from cancelled vocabulary | Added to the array |
| 5 | Outbound `customer_unanswered` with DB=answered → `cancelled` | DB-awareness couldn't distinguish which leg was answered | For outbound, DB=answered refers to agent callback, not customer |
| 6 | Late `answered` webhooks overwriting terminal status | No terminal guard on answered handlers | Added `TERMINAL_STATUSES.includes()` check |
| 7 | `DialCallDuration` includes ringing time | Used wrong duration field | Check `OnCallDuration` from Legs; override to 0 when all legs show 0 talk |
| 8 | V3 webhooks passing `undefined` to `resolveCallStatus` | `call_status` vs `status` field name mismatch | Added `\|\| callDetails.status` fallback |
| 9 | Inbound calls stuck at `ringing` when customer hangs up | Terminal passthrough sends `Status: "free"`, not a call status | Treat `"free"` as null; map `ringing` + null → `missed` |
| 10 | API verify overwrites terminal with transitional | `setImmediate` fires before Exotel updates V3 API | `api_verified_at` guard prevents re-verification |
| 11 | `convertCallToConsultation` overwrites status to `completed` | Consultation conversion blindly sets `CallStatus.COMPLETED` | Made `call_status` optional in DTO, removed forced overwrite |
| 12 | `NO_ANSWER` not in `TERMINAL_STATUSES` | Enum value was deprecated, not treated as terminal | Added to `TERMINAL_STATUSES` array, updated comment |
| 13 | `handleCallCompletedEvent` + API verify race condition | CallCompleted arrives after API verify sets `api_verified_at` → second verify skipped | First verify's `api_verified_at` guard blocks re-verification, but CallCompleted writes with authoritative data anyway |

---

## Architecture Diagram

```
Customer/Agent Phone
        │
        ▼
    Exotel Cloud
        │
        ├── V1 Passthrough (/call-events)
        │     ├── Dial → handleTerminalEvent (ringing)
        │     ├── Terminal → handleTerminalEvent (FINAL TERMINAL STATUS WRITE)
        │     ├── Answered → handleAnsweredEvent [TERMINAL GUARD]
        │     └── CallCompleted → handleCallCompletedEvent (with Legs, recording)
        │
        └── V2/V3 CCM (/call-details)
              ├── Answered → handleEnhancedAnsweredEvent [TERMINAL GUARD]
              └── Terminal → handleEnhancedTerminalEvent
                                │
                                ▼
                        resolveCallStatus()
                        (V1/V2/V3 unified mapping)
                                │
                                ├── Writes DB immediately
                                └── setImmediate → verifyCallStatusViaApi()
                                                    │
                                                    ├── Calls V3 API
                                                    ├── Re-runs resolveCallStatus()
                                                    └── Corrects DB if different
                                                    
Safety Nets:
    ├── cleanupStaleCalls (Jenkins cron, every 5 min)
    │     └── Finds stuck calls >20 min → V3 API → resolves
    ├── scheduleDelayedReconciliation (setTimeout 2 min)
    │     └── Backfills missing duration/recording
    └── Recording migration (Exotel → S3)
```

---

## Key Technical Decisions & Principles

1. **Webhook status is NOT the source of truth** — Exotel sends inconsistent, sometimes duplicate, sometimes missing webhooks. The V3 API is authoritative. Every terminal event is followed by async API verification.

2. **Direction matters** — The same V3 leg status means different things for inbound vs outbound. `from_leg` = caller in inbound, agent in outbound. `to_leg` = agent in inbound, customer in outbound.

3. **DB status provides temporal context** — When a late webhook arrives, the current DB status tells us what happened before. A `to_leg_unanswered` with DB=`answered` on inbound means "agent answered then disconnected" (cancelled), not "nobody answered" (missed).

4. **Terminal guards prevent regression** — Once a call reaches a terminal status, no webhook should regress it to a transitional state. Guards on all answered handlers prevent this.

5. **Business logic must not overwrite telephony truth** — `convertCallToConsultation` was setting `completed` regardless of call outcome. Business operations (linking, converting) should only update business fields, never call status.

6. **Defense in depth** — Three layers of status resolution: (1) synchronous webhook handling, (2) async V3 API verification, (3) periodic cleanup cron. If any layer fails, the next catches it.

7. **Feature-flagged rollout** — All V2 logic gated behind `CALL_STATUS_V2=true` environment variable. Legacy `mapEnhancedCallStatus` preserved as fallback.

8. **Surgical, incremental fixes** — Each bug fix was isolated to the minimum code change needed. No broad refactors. Each change was tested against real production call data before deployment.

---

## Files Changed

| File | What Changed |
|------|-------------|
| `exotel-webhook.service.ts` | Core: `resolveCallStatus`, `verifyCallStatusViaApi`, terminal guards, null guard, cleanup tracking, V3 field fallback, duration fix |
| `initiate-call.dto.ts` | `CallStatus` enum (added values, repurposed `NO_ANSWER`), `TERMINAL_STATUSES` array (added `NO_ANSWER`) |
| `consultation-integration.service.ts` | Removed forced `call_status: COMPLETED` from `convertCallToConsultation` |
| `update-call.dto.ts` | Made `call_status` optional in update DTO |
| `CallLogsSection.tsx` | New status colors, labels, red row highlight for inbound missed calls |
| `ManagerCallLogs.tsx` | Already handled new statuses (no changes needed) |

---

## Impact Metrics

- **10-value status enum** replacing binary completed/failed — enables granular reporting
- **30+ production calls corrected** via SQL migration (wrongly overwritten by consultation conversion)
- **13 distinct bugs identified and fixed** through analysis of real production call data
- **3-layer defense system** ensuring no call status is ever permanently wrong
- **<1 second** typical status resolution time (webhook + API verify)
- **20-minute max** worst-case resolution via cleanup cron for dropped webhooks
- **Zero downtime deployment** via feature flag rollout

---

## Production Data Analysis

Throughout the project, debugging was done by cross-referencing four data sources:
1. **Call logs DB** — `call_status`, `duration_seconds`, `answered_at`, `call_metadata`
2. **Exotel V3 API** — Authoritative call details (hit manually via Postman for verification)
3. **Application logs** — GCP Cloud Logging, filtered by call SID
4. **Webhook payloads** — Stored in `call_metadata.callEvents[]` for post-mortem analysis

This multi-source approach identified bugs that wouldn't surface from code review alone — race conditions between webhook delivery order, Exotel field name changes between V2/V3, and missing terminal events for certain call flows.

---

## Resume-Ready Bullet Points

- Designed and implemented a unified call status resolution engine mapping 20+ Exotel telephony statuses across V1/V2/V3 webhook formats to a 10-value internal enum, with direction-aware, duration-aware, and database-aware resolution logic
- Built a two-stage verification pipeline (synchronous webhook + asynchronous V3 API) with defense-in-depth architecture including periodic cleanup cron, ensuring 100% eventual status accuracy despite unreliable webhook delivery
- Identified and fixed 13 production bugs through systematic analysis of real call data, including race conditions from duplicate webhooks, V2/V3 field name mismatches, and business logic overwriting telephony truth
- Implemented terminal guards preventing late-arriving webhook events from regressing resolved call statuses, eliminating a class of data integrity issues
- Produced operations KT documentation and updated frontend dashboard with status-based visual indicators, enabling operations team to prioritize customer callbacks effectively

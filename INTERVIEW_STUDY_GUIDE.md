# Interview Study Guide — Parshuram Kumar

Every claim on your resume, decoded, with what to study and what you will be asked.

---

## 0. How to use this

Your resume makes ~40 technical claims. An interviewer will pick 3–5 and dig until you
stop being able to answer. The goal is not to memorize — it is to be able to explain
**the decision, the alternative you rejected, and the trade-off you accepted** for each one.

Rule of thumb: for every bullet, you should be able to talk for **2 minutes unprompted**
and then survive **3 follow-up questions**. If you can't, that bullet is a liability.

Topics are ranked in three tiers. Do not study them evenly.

- **Tier 1** — will almost certainly be asked, and asked deeply. Know cold.
- **Tier 2** — likely to come up. Know well enough to hold a conversation.
- **Tier 3** — terminology and vocabulary. Know enough not to fumble.

---

## 1. The numbers on your resume — what each one literally means

Interviewers test whether *you* know what your own numbers mean. This trips up more
candidates than any hard technical question.

| Claim | What it literally means | If asked, say |
|---|---|---|
| **28K+ LOC backend** | 28,200 lines of TypeScript across 131 files in the call-center feature folder. LOC = Lines Of Code. | "Size of the subsystem I owned. Includes team code written to my spec. Better proxies for complexity are the 9 tables and 134 endpoints." |
| **9 MySQL tables** | 9 Sequelize models: agents, call logs, dispositions, follow-up tasks, activity logs, call notes, transcriptions, daily reports, + the consultations bridge. | Name them. This is the fastest way to prove you actually designed it. |
| **134 REST endpoints** | Total HTTP verb handlers across 13 route files. | "Across 13 domain modules, each following the same routes→controller→service→DTO layering." |
| **96 React components** | Component files under the call-center admin panel. | Be careful — you did *some* frontend. Say "the module contained 96 components; I owned the My Work queue and the Manager Call Logs surfaces." |
| **13 domain modules** | agent, call, disposition, exotel, consultation, call-notes, activity, transcription, daily-report, consolidated, voip, ai-call, socket. | Memorize this list. It demonstrates the module boundary design instantly. |
| **20+ Exotel statuses → 10-value enum** | The provider emits different status vocabularies across 3 API versions; you collapsed them into one internal state machine. | Your internal enum: initiated, queued, ringing, answered, completed, short_call, missed, failed, busy, no_answer, cancelled. |
| **13 production bugs** | Bugs you found by analyzing real call data. | Have 2–3 specific ones ready: duplicate-webhook races, V2/V3 field name mismatches (`call_sid` vs `sid`), business logic overwriting telephony status. |
| **15+ queries → 2** | The work-queue dashboard used to run a separate COUNT per tab + per card. You collapsed them into 2 aggregate queries. | See §2.4 — this is a strong bullet, learn the SQL. |
| **4x gain / ~50% latency cut** | Your own prior measurements. | Know **how you measured it**. "Compared p50/p95 before and after" is the answer. If you never measured properly, say "measured on our APM (New Relic)". |
| **1–5 agent scoring** | The Gemini QA output includes `agent_performance_score` on a 1–5 scale. | Fine as-is. |

**Practice drill:** say each number out loud with its meaning, no notes. Ten minutes, once.

---

# TIER 1 — Know these cold

These four areas are where your resume is strongest, which means they are where the
interview will go.

---

## 1.1 Concurrency and race conditions

This is your signature bullet. Expect the deepest questioning here. It is also the
highest-value thing on your resume — very few 3-YOE candidates can discuss this credibly.

### The core concept: TOCTOU

**Time-Of-Check to Time-Of-Use.** A bug class where you check a condition, then act on it,
and something changes in between.

```js
// BROKEN — this is what you removed
const agent = await Agent.findOne({ where: { status: 'available' } });  // CHECK
// ...another request runs right here and claims the same agent...
await agent.update({ status: 'on_call' });                              // USE
```

Two concurrent requests both read `available`, both write `on_call`. Both think they won.
Result: one agent assigned two calls, or one call routed to two agents.

### The fix: atomic conditional UPDATE (optimistic concurrency)

```sql
UPDATE call_center_agents
SET status = 'on_call'
WHERE id = ? AND status = 'available';
```

Check and write are now **one statement**. The database decides the winner.

**Why this works — the mechanism you must be able to explain:**

1. InnoDB takes an **exclusive row lock (X lock)** on the matching row.
2. Request B blocks waiting for that lock.
3. When A commits, B acquires the lock and — because `UPDATE` performs a
   **current read** (reads the latest committed version, not the MVCC snapshot) —
   re-evaluates its `WHERE` clause.
4. B now sees `status='on_call'`, the predicate fails, and B gets **0 affected rows**.

`affectedRows === 1` means you won. `affectedRows === 0` means you lost. Exactly one winner,
guaranteed by the storage engine.

**Gotcha you should know:** MySQL distinguishes `affectedRows` (rows changed) from
`foundRows` (rows matched). If you `SET status='on_call'` on a row *already* `on_call`,
MySQL reports 0 changed rows by default even though it matched. Understand this before
you claim it, because a sharp interviewer will probe it.

### Study list

- **MVCC** (Multi-Version Concurrency Control) — how InnoDB gives each transaction a
  consistent snapshot.
- **Isolation levels**: READ UNCOMMITTED / READ COMMITTED / REPEATABLE READ (MySQL default)
  / SERIALIZABLE. Know **dirty read, non-repeatable read, phantom read** and which level
  prevents which.
- **Locking reads**: `SELECT ... FOR UPDATE` vs `LOCK IN SHARE MODE`, and why you chose a
  conditional UPDATE *instead* of `SELECT FOR UPDATE`.
- **Optimistic vs pessimistic concurrency control.** Yours is optimistic. Know the
  difference and when each wins (optimistic = low contention, cheap; pessimistic = high
  contention, avoids retry storms).
- **Gap locks and next-key locks** in REPEATABLE READ (why phantoms are prevented).
- **Deadlocks** — how InnoDB detects them, why lock ordering matters, `SHOW ENGINE INNODB STATUS`.
- **Idempotency** — an operation safe to repeat. Your unique constraints and pending-status
  guards are idempotency mechanisms.

### Questions you will be asked

- *"Walk me through exactly how two concurrent requests can't claim the same agent."*
- *"Why not just use a transaction?"* → Your answer: the critical ops are single-row state
  flips; a conditional UPDATE is atomic on its own and cheaper under a poll storm. I used a
  real transaction only where I mutate two rows (complete task + spawn child), and I emit
  events post-commit so nothing fires on rollback.
- *"What if the process crashes right after the claim?"* → The agent is claimed but the call
  isn't assigned. Recoverable state, not a split brain — the reconciliation job releases it.
- *"What isolation level are you on and does it matter here?"* → REPEATABLE READ (MySQL
  default). It matters for the read paths, less for the UPDATE because that's a current read.
- *"Is `affectedRows` reliable?"* → See gotcha above.

---

## 1.2 Redis and distributed locking

### The concept

The conditional UPDATE prevents *incorrectness*. The Redis lock prevents *waste* — it stops
N concurrent polls from all doing the routing work. That distinction matters; state it clearly.

```
SET lock:ccm:<CallSid> <token> NX PX 15000
```

- `NX` — set only if the key does not exist. Atomic test-and-set.
- `PX 15000` — expire after 15 seconds. **Critical**: without a TTL, a crashed holder
  deadlocks the key forever.

Winner routes and caches its response. Losers poll the cache and **replay the identical
destination** — so all concurrent polls get the same answer instead of racing.

### The senior-level critique you must know

Redis locks are **not** safe under failover. This is a famous debate — Martin Kleppmann's
critique of Redlock vs. antirez's response. Read a summary of both. The core problem:

1. Client A acquires the lock on the master.
2. The master dies before replicating the lock to a replica.
3. A replica is promoted. The lock key doesn't exist there.
4. Client B acquires "the same" lock. Two holders.

**Your answer when challenged:** "That's exactly why the Redis lock isn't my correctness
guarantee. It's an optimization to prevent duplicate work. Correctness comes from the
database's atomic claim, which is why the system stays correct in degraded mode when Redis
is completely unavailable."

That answer is genuinely strong. Practice it.

### Also study

- **Replica lag** — why you read the lock cache from the **master**, not a replica. A
  sub-second race window is shorter than replication lag, so a replica read would miss the
  cached response and defeat the whole lock.
- **Fencing tokens** — the proper fix Kleppmann proposes (monotonically increasing token
  checked by the resource). You don't have this. Say so honestly.
- **Lock release safety** — you must only delete the lock if you still own it (compare the
  token). Deleting blindly can release someone else's lock after your TTL expired. Standard
  solution: a small Lua script that does compare-and-delete atomically.
- **Redis single-threaded model** — why individual Redis commands are atomic.
- **Redis persistence**: RDB snapshots vs AOF (append-only file), and why neither makes locks
  safe under failover.
- **Redis data types** you'd reach for: strings, hashes, sorted sets, TTL/expiry semantics.
- **Cache patterns**: cache-aside (lazy loading), write-through, write-behind. **Cache
  invalidation** strategies and why it's hard.
- **Thundering herd / cache stampede** — many clients missing the cache at once. Single-flight
  is literally the fix for this. Name the pattern; it scores points.

---

## 1.3 Database design, indexing and query optimization

### Indexing

- **B+ tree** structure — why range queries and `ORDER BY` work on indexes, why hashes don't.
- **Clustered index** — in InnoDB the primary key *is* the table's physical order. Secondary
  indexes store the PK, so a secondary lookup costs an extra hop ("bookmark lookup").
- **Leftmost prefix rule** — an index on `(a, b, c)` serves `WHERE a`, `WHERE a AND b`,
  `WHERE a AND b AND c` — but **not** `WHERE b` alone. This is the single most-asked indexing
  question. Your composite `(agent_id, resolution_status, priority, created_at)` is a perfect
  example: know why the column order is what it is.
- **Covering index** — when the index contains every column the query needs, InnoDB never
  touches the table. `Using index` in EXPLAIN.
- **Cardinality and selectivity** — why indexing a boolean is usually useless.
- **Why indexes cost you** — every INSERT/UPDATE/DELETE must maintain them. Write amplification.
- **`EXPLAIN` / `EXPLAIN ANALYZE`** — learn to read the output: `type` (const, eq_ref, ref,
  range, index, ALL — ALL is a full scan), `key`, `rows`, `filtered`, `Extra` (Using index,
  Using filesort, Using temporary).
- **Function on a column kills the index.** `WHERE DATE(created_at) = '2026-08-25'` cannot use
  an index on `created_at`. This is why you reconstruct absolute UTC boundaries instead:
  `WHERE created_at >= ? AND created_at < ?`. **This is a genuinely good interview answer —
  know it.**

### N+1 queries

Fetch 100 calls, then loop and fetch each call's agent → 101 queries. Fixes:

- **Eager loading** (`include` / JOIN)
- **Batch loading** — collect all IDs, one `WHERE id IN (...)`, build an in-memory `Map`.
  This is what your report collector does.
- **DataLoader** pattern (worth knowing by name).

### Conditional aggregation (your 15→2 bullet)

```sql
SELECT
  SUM(CASE WHEN resolution_status = 'pending'  THEN 1 ELSE 0 END) AS pending,
  SUM(CASE WHEN resolution_status = 'overdue'  THEN 1 ELSE 0 END) AS overdue,
  SUM(CASE WHEN priority = 'high'              THEN 1 ELSE 0 END) AS high_priority
FROM call_dispositions
WHERE agent_id = ?;
```

One table scan produces every count. The key insight to articulate: **the same SQL fragment
generates both the list's WHERE clause and the badge's CASE condition**, so a tab count can
never disagree with the list behind it. That's a correctness argument, not just a perf one —
lead with it.

### Sequelize specifics

- `findAndCountAll({ distinct: true })` → `COUNT(DISTINCT parent.id)`, needed because a
  hasMany JOIN multiplies parent rows.
- `subQuery: false` → stops Sequelize wrapping the query in a subquery to apply LIMIT to the
  parent. Faster, but **changes semantics**: LIMIT now applies to joined rows. Know the trade-off.
- **Why an ORM at all** — and when you'd drop to raw SQL. You did drop to raw SQL for the
  aggregation. Good answer.

### Transactions

- **ACID** — Atomicity, Consistency, Isolation, Durability. Be able to define each in one line.
- Your stance: *atomicity via single-statement conditional UPDATEs, not multi-row transactions.*
  Defend it as above.
- **Emit events after commit, never inside the transaction** — otherwise a rollback fires
  events for work that never happened. You do this in the task-complete flow. Great detail.
- **Two-phase commit** and why distributed transactions are avoided — conceptual only.

### Schema design patterns on your resume

- **Self-referential foreign key / adjacency list** — `parent_disposition_id` pointing at the
  same table. Used for reschedule chains. Know the alternatives for tree storage: adjacency
  list, nested set, materialized path, closure table.
- **Append-only audit trail** — never overwrite; write a new row or append to a JSON array
  with `{timestamp, actor, old, new, reason}`. Related concept: **event sourcing** (know the
  term and that yours is a lightweight version, not the full pattern).
- **Soft delete** (`deleted_at`, Sequelize `paranoid: true`) — why retention/compliance needs
  it, and the cost (every query must filter it).
- **Versioning with a unique index** — `UNIQUE (call_log_id, version)`, `version = MAX+1`.
  Preserves history and doubles as the concurrency backstop: the loser's INSERT throws a
  `UniqueConstraintError`.
- **Nullable FK as a design decision** — `call_log_id` is nullable so a disposition can exist
  without a call (manual entry). Be ready to defend allowing NULL.
- **Normalization** — 1NF/2NF/3NF, and when you'd denormalize for read performance.

### Timezone correctness

- Store **UTC always**. Convert at the boundary.
- `CONVERT_TZ(col, '+00:00', '+05:30')` — **numeric offsets always work**; named zones like
  `'Asia/Kolkata'` require the MySQL timezone tables to be loaded (`mysql_tzinfo_to_sql`).
  Know this gotcha.
- IST is **UTC+5:30** — a half-hour offset, which breaks naive assumptions.
- Why `new Date().setHours(0,0,0,0)` is wrong: it uses the *server's* local timezone, which
  drifts 5.5 hours from your business day.
- The right pattern: compute the IST day boundary, convert it to an absolute UTC instant,
  then filter with `>=` / `<` so the index still works.

---

## 1.4 Webhooks, eventual consistency and distributed system failure

### Idempotency

An operation you can safely repeat. Providers retry. Networks duplicate. Assume **at-least-once
delivery**, never exactly-once.

Your mechanisms, know all three:
1. **Unique constraint** on `exotel_call_sid` — the database refuses the duplicate.
2. **Application pre-check** — a fast path that avoids the exception in the common case.
3. **Catch the `UniqueConstraintError`** — the actual guarantee, because the pre-check is a
   TOCTOU window.

The layering (fast path + real guarantee) is the point. Say it that way.

### Delivery semantics

- **At-most-once** — may lose messages.
- **At-least-once** — may duplicate. What webhooks give you. Requires idempotent consumers.
- **Exactly-once** — effectively unachievable end-to-end; approximated by at-least-once
  delivery + idempotent processing. Know why the "exactly-once" claim is usually marketing.

### Returning 200 on internal failure

You return HTTP 200 to Exotel even when your own processing failed, to stop the provider
retrying. This is a real trade-off and interviewers will push:

*"Isn't that swallowing errors?"*

Your answer: "It stops a retry storm from amplifying an outage, and provider retries were
causing duplicate processing. I gave up the provider's retry as a safety net, so I built my
own: a reconciliation job that sweeps calls stuck in a non-terminal state past 20 minutes and
reconciles them against the provider's API. The retry moved from them to me, deliberately."

That is a strong, senior-sounding answer. Rehearse it.

### Eventual consistency

Webhooks can be **stale, out of order, or lost**. Your two-stage design:
1. Webhook arrives → write the state you believe.
2. Asynchronously re-verify against the provider's v3 API → overwrite if the API knows better.
3. Reconciliation cron → catch anything both stages missed.

Concepts to name: **source of truth**, **read-repair**, **reconciliation**,
**anti-entropy**, **convergence**.

### Resilience patterns (learn the vocabulary — you built these, name them properly)

- **Graceful degradation** — reduced function instead of failure. Your Redis-down path.
- **Fail open vs fail closed** — you fail *open* on Redis (proceed without the lock) because
  the DB still guarantees correctness. Know when failing closed is right instead.
- **Bulkhead** — isolate failures so one dependency can't sink the whole service.
- **Circuit breaker** — stop calling a failing dependency; states: closed → open → half-open.
  You don't have one. Say so; propose it as an improvement.
- **Retry with exponential backoff + jitter** — and why jitter matters (prevents synchronized
  retry storms).
- **Timeouts** — every network call needs one. A missing timeout is how one slow dependency
  takes down a whole service.
- **Dead letter queue** — where permanently-failed messages go.
- **Backpressure** — what to do when you're receiving faster than you can process.

### Signature verification

Your dossier flags this: HMAC-SHA1 verification exists but is only wired to stub handlers;
live endpoints rely on a shared internal secret at the router instead. **Volunteer this as a
known hardening gap.** Study: HMAC, why you compare with a **constant-time** comparison
(`crypto.timingSafeEqual`) to avoid timing attacks, and replay protection via timestamp +
nonce.

---

# TIER 2 — Know well

---

## 2.1 Node.js runtime

You claim "post-response side effects for zero hot-path latency." That requires understanding
the event loop.

- **Event loop phases** in order: timers → pending callbacks → idle/prepare → **poll** →
  **check** → close callbacks.
- `setImmediate()` runs in the **check** phase — i.e. after the current poll phase completes.
  This is how you defer work until after the HTTP response flushes.
- `process.nextTick()` runs **before** the event loop continues — it is not a phase, it drains
  after the current operation. Overusing it starves the loop.
- `setTimeout(fn, 0)` — timers phase; ordering vs `setImmediate` at top level is
  non-deterministic. Know why.
- **Why blocking the event loop is fatal** in a single-threaded runtime.
- **libuv thread pool** (default 4) — handles fs, dns, crypto, zlib. Not your network I/O.
- **Worker threads** vs **cluster** vs **child_process** — when each applies.
- **Streams and backpressure** — relevant to your S3 downloads.
- `Promise.all` vs `Promise.allSettled` — `all` rejects on the first failure;
  `allSettled` always resolves with per-promise status. You use `allSettled` so one failed
  sub-query degrades to zero instead of 500-ing the dashboard. Good detail.
- **Bounded concurrency** — why you batch at 5 instead of firing 500 promises. Know how to
  hand-roll a promise pool; know `p-limit` exists.
- **Unhandled promise rejections** and why fire-and-forget needs an explicit `.catch()`.

## 2.2 API and system design

- **REST** — resource modeling, correct verbs, status codes. Know **409 Conflict** specifically
  (your claim endpoint returns it when a call is already claimed) and why 409 not 400 or 403.
- **Idempotent HTTP methods** — GET/PUT/DELETE idempotent, POST not. How idempotency keys fix POST.
- **Pagination**: offset/limit vs **cursor/keyset**. Offset degrades on deep pages and can skip
  or duplicate rows when data shifts. Know why a **stable tiebreaker** (sorting by
  `created_at, id`) is required — you have this; it's a good detail.
- **API gateway** — why the browser talks to a gateway, not your service directly.
- **BFF (Backend For Frontend)** — the Next.js proxy layer. Why: hides the internal base URL,
  allow-lists params before they become cache keys, keeps token handling server-side.
- **Authentication vs authorization.** JWT: header/payload/signature, stateless verification,
  expiry, refresh tokens, and **why you cannot revoke a JWT** without a blocklist. Your model
  carries permissions inside the token — know the trade-off (fast, but stale until expiry).
- **RBAC** vs ABAC.
- **Microservices** — service boundaries, why you'd split, distributed transaction problems,
  the **saga** pattern (conceptual).
- **Rate limiting** algorithms: fixed window, sliding window, token bucket, leaky bucket.
- **Feature flags** — staged rollout, kill switches, keeping legacy paths side by side, and
  the cost (branching complexity, dead code). You do this; know both sides.

## 2.3 Realtime — Socket.IO / WebSockets

- WebSocket handshake (HTTP Upgrade), full-duplex, vs HTTP polling vs SSE.
- Socket.IO's fallback to long-polling, and its **rooms** and **namespaces** (you used three).
- **Scaling websockets horizontally** — sticky sessions, and the **Redis adapter / pub-sub**
  to broadcast across instances. Almost certain follow-up question.
- Reconnection, heartbeats, and why you keep a polling fallback.

## 2.4 AI / GenAI engineering

- **Strategy + Factory pattern** — your `AiProvider` interface with Gemini/Claude/OpenAI
  adapters. Be able to explain both patterns generically and why you used them
  (vendor swap without touching the pipeline).
- **Structured output** — `responseMimeType: 'application/json'`, `temperature: 0` for
  determinism, and why you still need defensive JSON parsing.
- **Tokens** — what they are, why cost is per-1K/1M tokens, input vs output pricing.
- **Why you removed the LLM from HTML rendering** — presentation is deterministic, judgment
  isn't. This is one of your best answers; it shows engineering taste, not just AI usage.
- **Prompt versioning** — why prompts are artifacts that need version pinning and
  schema-compatibility checks.
- **RAG** (for the project bullet): chunking strategy and why 800 tokens, overlap, embeddings,
  **vector similarity** (cosine vs dot product vs euclidean), **pgvector**, and index types
  (**HNSW** vs **IVFFlat**) — expect "how does approximate nearest neighbour search work?"
- **Streaming responses** — SSE, why streaming improves perceived latency.
- **Hallucination**, **context window**, **temperature/top-p** — basic vocabulary.

## 2.5 AWS S3

- Buckets, keys, why the S3 key looks like a path but isn't a directory.
- **Presigned URLs** — time-limited signed access so you never expose the key or make the
  bucket public. Know how the signature works conceptually.
- Storage classes and lifecycle policies (why you'd expire old recordings).
- `rejectUnauthorized: false` on the recording download — you disabled TLS certificate
  verification because the provider uses a self-signed cert. **This is a security hole and you
  should volunteer it as a known, accepted trade-off**, along with the better fix: pin the
  provider's certificate rather than disabling verification wholesale.

---

# TIER 3 — Vocabulary and terminology

You should not stumble on these, but you won't be grilled.

## 3.1 Telephony / CTI

- **CTI** — Computer Telephony Integration. Software controlling phone calls.
- **PSTN** — the traditional phone network. **VoIP** — voice over IP.
- **SIP** — Session Initiation Protocol; the signaling protocol for VoIP.
- **WebRTC** — real-time audio/video in the browser. Know **signaling**, **STUN**, **TURN**,
  **ICE** at a one-sentence level. TURN = relay server used when peer-to-peer fails.
- **DTMF** — the tones from keypad presses. Your priority routing reads these.
- **IVR** — the "press 1 for sales" menu.
- **CallSid** — the provider's unique call identifier. Your primary idempotency key.
- **Inbound vs outbound**, **from-leg vs to-leg** — a call has two legs; the from-leg is the
  originating party. Your status resolver is direction-aware because "unanswered" means
  different things on each.
- **Sticky agent** — route a repeat caller back to the same agent.
- **Disposition** — the outcome an agent records after a call.
- **Poll-driven routing** — the provider repeatedly calls your endpoint asking "who should I
  connect to?" rather than you pushing a decision. This is the root cause of your race condition.

## 3.2 Design patterns you have used (name them correctly)

Strategy, Factory, Adapter, Repository, Registry, Singleton, Observer/pub-sub,
Circuit breaker, Bulkhead, Retry, Single-flight, Optimistic locking, Idempotency key.

## 3.3 DSA — for the coding round

Your resume claims 450+ LeetCode. You will get a coding round regardless of the systems work.

Priorities: arrays/two pointers, hashmaps, sliding window, binary search (including
"binary search on answer"), stacks, linked lists, trees/BFS/DFS, graphs (BFS, DFS, topological
sort, Dijkstra), heaps/priority queues, intervals, basic DP (knapsack, LIS, grid paths).

Be able to state **time and space complexity** for everything you write, unprompted.

---

# 4. Skills-list exposure audit

Blunt assessment. Your skills section lists things the dossier shows no evidence of. If an
interviewer picks one of these, you have a problem.

| Listed skill | Evidence in the dossier | Risk |
|---|---|---|
| Express.js, Sequelize, MySQL, Redis, Socket.IO, S3, TypeScript | Extensive | None — these are your core |
| Next.js, React, Zustand, SWR | Present in the module | Low–medium; you owned some surfaces |
| Gemini / Claude / OpenAI, RAG, pgvector | Extensive | None |
| Python / FastAPI | The iCal service | Low, but review FastAPI basics |
| **NestJS** | None visible | **Medium** — be ready to say where you used it, or drop it |
| **Fastify** | None visible | **Medium** — same |
| **TypeORM, Prisma** | Prisma in the RAG project only | Low–medium; know Prisma, be honest about TypeORM |
| **MongoDB** | The CampusMatch project (now cut from the resume) | **Medium** — you list it but the supporting project is gone |
| **Kubernetes** | None visible | **High** — "Kubernetes" invites deep questions (pods, deployments, services, ingress, HPA). If you've only ever read about it, remove it. |
| **Docker** | Not shown, but plausible | Low if you've written a Dockerfile; know images vs containers, layers, volumes, networks |
| **C/C++** | None | Low — usually read as academic |

**Recommendation:** either spend a weekend getting genuinely conversational on Kubernetes and
NestJS, or remove them. An interviewer catching you on a listed skill damages credibility on
everything else you said — including the parts that are completely true. The cost of removal
is near zero; the cost of being caught is not.

---

# 5. Weaknesses to volunteer before you're caught

From dossier §10. Naming your own system's flaws is the single strongest signal of seniority
you can give. Prepare a one-line version of each:

1. **Webhook signature verification isn't wired to live endpoints** — relies on a shared
   internal secret instead. Hardening TODO.
2. **Two non-atomic assign paths remain** (`assignCallToAgent`, `connectAgentWithClient`) —
   still read-check-then-write, inconsistent with the atomic pattern used elsewhere.
3. **Background schedulers have no distributed lock** — they will double-fire on a multi-instance
   deploy. Fix: a Redis lock or a leader-election mechanism.
4. **A `mock-fallback` in one BFF route returns fake success** on upstream failure — the UI
   believes a note saved when the backend never received it. Real correctness bug.
5. **Test coverage is thin** outside the concurrency harness.
6. **`rejectUnauthorized: false`** on recording downloads — accepted TLS trade-off, better fix
   is certificate pinning.
7. **Cost tracking is wrong for non-Gemini models** — they resolve to a null pricing preset and
   record as $0.

Have the *fix* ready for each, not just the admission. "I know it's broken" is weak;
"I know it's broken and here's the two-hour fix" is strong.

---

# 6. Study plan

Assumes ~2 hours a day. Adjust to your timeline.

**Week 1 — Your own system.** Re-read the dossier and CALL_FLOW.md with the code open. For
each of the 10 resume bullets, write a 5-sentence answer in your own words. This is the
highest-return week by far: you cannot be caught out on your own code, but you *can* be caught
having forgotten it.

**Week 2 — Databases.** MySQL isolation levels, locking, MVCC, indexing, `EXPLAIN`. Actually
run `EXPLAIN` on your own slow queries and read the output. Rebuild your 15→2 aggregation
query from scratch on paper.

**Week 3 — Distributed systems and Node.** Redis locking + the Redlock debate, idempotency,
delivery semantics, resilience patterns, the Node event loop.

**Week 4 — System design.** Practice designing: a call-center router, a URL shortener, a rate
limiter, a notification service, a job scheduler. Your real experience maps directly onto the
first one — lead with it when given the choice.

**Ongoing — DSA.** 2–3 problems daily, timed, always stating complexity out loud.

---

# 7. Rapid-fire self-test

Answer out loud. Any hesitation marks a gap.

1. What does 28K LOC mean and why is it a weak metric?
2. Name your 9 tables.
3. Name your 13 domain modules.
4. Explain TOCTOU in one sentence.
5. Why does `affectedRows === 1` prove you won the race?
6. What's MySQL's default isolation level?
7. Difference between a dirty read and a non-repeatable read?
8. What does `NX` do in `SET NX PX`?
9. Why must a distributed lock have a TTL?
10. Why is a Redis lock unsafe under failover, and why doesn't that break your system?
11. Why read the lock cache from the master and not a replica?
12. Leftmost prefix rule — give an example that fails.
13. Why does `WHERE DATE(col) = ?` kill index usage?
14. What's an N+1 query and three ways to fix it?
15. Why `distinct: true` in `findAndCountAll`?
16. What does `subQuery: false` change semantically?
17. What does ACID stand for? Define each in one line.
18. Why emit domain events *after* commit?
19. At-least-once vs exactly-once delivery?
20. Name three idempotency mechanisms in your system.
21. Why return 200 to a webhook on internal failure, and what did you build to compensate?
22. What's the difference between failing open and failing closed? Which did you choose and why?
23. What is a circuit breaker and what are its three states?
24. Which event loop phase does `setImmediate` run in?
25. `Promise.all` vs `Promise.allSettled` — why did you pick one?
26. Why 409 and not 400 for a claim conflict?
27. Offset vs cursor pagination — when does offset break?
28. Why can't you revoke a JWT?
29. How do you scale Socket.IO across multiple server instances?
30. Explain the Strategy pattern using your AI provider abstraction.
31. Why did you remove the LLM from HTML rendering?
32. What is a presigned S3 URL and what problem does it solve?
33. Cosine similarity vs euclidean distance for embeddings — why cosine?
34. What is HNSW?
35. What's the biggest weakness in the system you built?

If you can answer all 35 without notes, you are ready.

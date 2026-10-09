# Java Backend Developer Interview Preparation Handbook

> **Target:** Java Backend Developer, 2–3 years of experience  
> **Scope:** Backend engineering, Core Java through testing and deployment. **DSA and System Design intentionally excluded.**  
> **Estimated effort:** ~300–400 hours including study, coding, and revision. Times are planning estimates, not guaranteed completion times.

## How to use this handbook

1. Start in order through Java, SQL, Spring Boot, JPA, Security and Testing.
2. For every subtopic: learn the concept, code an example, answer the interview questions aloud, then explain a practical debugging scenario.
3. Mark a section as completed only after independently implementing relevant examples.
4. Practice testing in parallel with Spring Boot instead of delaying it until the end.
5. Use a modern LTS JDK such as Java 21 for the practice application.

## Preparation hour allocations

| Area | Suggested hours |
|---|---:|
| Core Java Fundamentals | 18–22 hours |
| OOP and Design Principles | 12–15 hours |
| Collections and Generics | 20–25 hours |
| Exceptions and Modern Java | 18–22 hours |
| Multithreading and Concurrency | 22–28 hours |
| JVM and Diagnostics | 12–16 hours |
| SQL and PostgreSQL | 25–30 hours |
| Spring Core and Spring Boot | 35–45 hours |
| REST, HTTP and Networking | 15–20 hours |
| Spring Data JPA and Hibernate | 25–30 hours |
| Spring Security | 18–24 hours |
| Microservices and Reliability | 22–28 hours |
| Kafka, Redis and Messaging | 20–25 hours |
| JUnit, Mockito and Integration Testing | 20–25 hours |
| Git, Maven, Docker, CI/CD, Linux and Cloud | 20–25 hours |

---

## 1. Core Java Fundamentals
**Estimated time:** 18–22 hours

### Subtopics and common interview questions

#### 1.1 Java execution model
**What to cover:** JDK vs JRE vs JVM; javac, bytecode, class loading, JIT, platform independence.

**Common interview questions:**
- What happens when you run a Java program?
- Why is Java platform independent?
- JDK vs JVM?

**Practice task:** Build a small isolated example and be ready to describe its behavior, failure cases, and tradeoffs.

#### 1.2 Types and variables
**What to cover:** Primitives vs references; numeric promotions, casting, scope, literals, default values.

**Common interview questions:**
- Why is Java pass-by-value?
- == vs equals()?
- What is auto-boxing?

**Practice task:** Build a small isolated example and be ready to describe its behavior, failure cases, and tradeoffs.

#### 1.3 Control flow and methods
**What to cover:** conditions, loops, switch, overloading, varargs, recursion basics.

**Common interview questions:**
- Can main be overloaded?
- What happens with overloaded method resolution?

**Practice task:** Build a small isolated example and be ready to describe its behavior, failure cases, and tradeoffs.

#### 1.4 Strings and wrappers
**What to cover:** String pool, intern(), immutability, StringBuilder vs StringBuffer, wrapper caching.

**Common interview questions:**
- Why is String immutable?
- String vs StringBuilder vs StringBuffer?
- == comparisons with Integer?

**Practice task:** Build a small isolated example and be ready to describe its behavior, failure cases, and tradeoffs.

#### 1.5 Modifiers and keywords
**What to cover:** static, final, this, super, access modifiers, packages, enum, records basics.

**Common interview questions:**
- final vs finally vs finalize?
- static block execution order?
- Can static methods be overridden?

**Practice task:** Build a small isolated example and be ready to describe its behavior, failure cases, and tradeoffs.

#### 1.6 Object basics
**What to cover:** Object methods, toString, equals, hashCode, clone, immutable values.

**Common interview questions:**
- Explain equals/hashCode contract
- shallow vs deep copy?

**Practice task:** Build a small isolated example and be ready to describe its behavior, failure cases, and tradeoffs.

**Section completion test:** Explain all the questions in this section without notes and demonstrate one or more working examples.

---

## 2. OOP and Design Principles
**Estimated time:** 12–15 hours

### Subtopics and common interview questions

#### 2.1 Class design
**What to cover:** constructors, chaining, encapsulation, inheritance, composition.

**Common interview questions:**
- Inheritance vs composition?
- Constructor execution order?

**Practice task:** Build a small isolated example and be ready to describe its behavior, failure cases, and tradeoffs.

#### 2.2 Polymorphism
**What to cover:** overload vs override; runtime dispatch; casting; covariant return types.

**Common interview questions:**
- Overloading vs overriding?
- Can private methods be overridden?

**Practice task:** Build a small isolated example and be ready to describe its behavior, failure cases, and tradeoffs.

#### 2.3 Abstraction
**What to cover:** abstract classes vs interfaces; default and static interface methods.

**Common interview questions:**
- When choose interface instead of abstract class?
- Multiple inheritance in Java?

**Practice task:** Build a small isolated example and be ready to describe its behavior, failure cases, and tradeoffs.

#### 2.4 Maintainability
**What to cover:** SOLID with Java examples; coupling, cohesion, dependency inversion.

**Common interview questions:**
- Explain SRP, OCP and DIP with code
- What makes a class hard to test?

**Practice task:** Build a small isolated example and be ready to describe its behavior, failure cases, and tradeoffs.

#### 2.5 Immutability
**What to cover:** defensive copying, final fields, immutable collections, records.

**Common interview questions:**
- How would you design a truly immutable Java class?

**Practice task:** Build a small isolated example and be ready to describe its behavior, failure cases, and tradeoffs.

**Section completion test:** Explain all the questions in this section without notes and demonstrate one or more working examples.

---

## 3. Collections and Generics
**Estimated time:** 20–25 hours

### Subtopics and common interview questions

#### 3.1 Hierarchy and choice
**What to cover:** Collection vs Map; List, Set, Queue, Deque; iteration and ordering.

**Common interview questions:**
- List vs Set vs Map?
- When use ArrayDeque?

**Practice task:** Build a small isolated example and be ready to describe its behavior, failure cases, and tradeoffs.

#### 3.2 List and Set
**What to cover:** ArrayList capacity and resizing; LinkedList; HashSet, LinkedHashSet, TreeSet.

**Common interview questions:**
- ArrayList vs LinkedList?
- HashSet vs TreeSet?

**Practice task:** Build a small isolated example and be ready to describe its behavior, failure cases, and tradeoffs.

#### 3.3 Map internals
**What to cover:** HashMap hash distribution, bucket, collision, resize, tree bins; load factor.

**Common interview questions:**
- How does HashMap work internally?
- What happens on collision?
- Why immutable keys?

**Practice task:** Build a small isolated example and be ready to describe its behavior, failure cases, and tradeoffs.

#### 3.4 Comparison and iteration
**What to cover:** Comparable vs Comparator; Iterator; fail-fast vs weakly consistent.

**Common interview questions:**
- How to sort custom objects?
- ConcurrentModificationException?

**Practice task:** Build a small isolated example and be ready to describe its behavior, failure cases, and tradeoffs.

#### 3.5 Concurrent collections
**What to cover:** ConcurrentHashMap, CopyOnWriteArrayList, blocking queues.

**Common interview questions:**
- HashMap vs ConcurrentHashMap?
- When CopyOnWriteArrayList?

**Practice task:** Build a small isolated example and be ready to describe its behavior, failure cases, and tradeoffs.

#### 3.6 Generics
**What to cover:** generic classes/methods, invariance, extends/super wildcards, PECS, erasure.

**Common interview questions:**
- What is type erasure?
- List<?> vs List<Object>?
- Explain PECS.

**Practice task:** Build a small isolated example and be ready to describe its behavior, failure cases, and tradeoffs.

#### 3.7 Complexity and tradeoffs
**What to cover:** common operations cost, ordering, memory overhead.

**Common interview questions:**
- Which collection for frequent lookup vs ordered iteration?

**Practice task:** Build a small isolated example and be ready to describe its behavior, failure cases, and tradeoffs.

**Section completion test:** Explain all the questions in this section without notes and demonstrate one or more working examples.

---

## 4. Exceptions and Modern Java
**Estimated time:** 18–22 hours

### Subtopics and common interview questions

#### 4.1 Exception handling
**What to cover:** Throwable hierarchy, checked/unchecked, try/catch/finally, propagation.

**Common interview questions:**
- Checked vs unchecked?
- throw vs throws?
- Does finally always execute?

**Practice task:** Build a small isolated example and be ready to describe its behavior, failure cases, and tradeoffs.

#### 4.2 Resource handling
**What to cover:** try-with-resources, AutoCloseable, suppressed exceptions.

**Common interview questions:**
- How does try-with-resources work?

**Practice task:** Build a small isolated example and be ready to describe its behavior, failure cases, and tradeoffs.

#### 4.3 Functional programming
**What to cover:** functional interfaces, lambda capture, method references.

**Common interview questions:**
- Lambda vs anonymous class?
- Effectively final variables?

**Practice task:** Build a small isolated example and be ready to describe its behavior, failure cases, and tradeoffs.

#### 4.4 Streams
**What to cover:** map, filter, flatMap, sorted, distinct, reduce, collect, laziness.

**Common interview questions:**
- map vs flatMap?
- Intermediate vs terminal operations?
- Why streams are lazy?

**Practice task:** Build a small isolated example and be ready to describe its behavior, failure cases, and tradeoffs.

#### 4.5 Collectors and Optional
**What to cover:** groupingBy, partitioningBy, toMap merge, Optional use.

**Common interview questions:**
- How to handle duplicate keys in toMap?
- Optional.orElse vs orElseGet?

**Practice task:** Build a small isolated example and be ready to describe its behavior, failure cases, and tradeoffs.

#### 4.6 Modern language features
**What to cover:** Java 9+ collections, records, sealed types, switch expressions, pattern matching; Date/Time API.

**Common interview questions:**
- Record vs class?
- Why prefer java.time?

**Practice task:** Build a small isolated example and be ready to describe its behavior, failure cases, and tradeoffs.

#### 4.7 Parallelism pitfalls
**What to cover:** parallel stream thread safety, mutable state, ordering.

**Common interview questions:**
- When can parallel streams make performance worse?

**Practice task:** Build a small isolated example and be ready to describe its behavior, failure cases, and tradeoffs.

**Section completion test:** Explain all the questions in this section without notes and demonstrate one or more working examples.

---

## 5. Multithreading and Concurrency
**Estimated time:** 22–28 hours

### Subtopics and common interview questions

#### 5.1 Thread concepts
**What to cover:** thread lifecycle; Runnable, Callable; interrupts; daemon threads.

**Common interview questions:**
- Runnable vs Callable?
- What is interrupt?
- Thread vs process?

**Practice task:** Build a small isolated example and be ready to describe its behavior, failure cases, and tradeoffs.

#### 5.2 Executors
**What to cover:** fixed/cached pools, queueing, ExecutorService, shutdown, rejection policies.

**Common interview questions:**
- Why use executor instead of new Thread?
- How size a pool?

**Practice task:** Build a small isolated example and be ready to describe its behavior, failure cases, and tradeoffs.

#### 5.3 Futures
**What to cover:** Future, CompletableFuture composition, exception handling, timeout.

**Common interview questions:**
- thenApply vs thenCompose?
- How handle async failures?

**Practice task:** Build a small isolated example and be ready to describe its behavior, failure cases, and tradeoffs.

#### 5.4 Synchronization
**What to cover:** synchronized, volatile, atomics, lock visibility, race condition.

**Common interview questions:**
- volatile vs synchronized?
- What makes ++ non-atomic?

**Practice task:** Build a small isolated example and be ready to describe its behavior, failure cases, and tradeoffs.

#### 5.5 Locks and coordination
**What to cover:** ReentrantLock, ReadWriteLock, Semaphore, CountDownLatch, BlockingQueue.

**Common interview questions:**
- How avoid deadlock?
- Semaphore vs CountDownLatch?

**Practice task:** Build a small isolated example and be ready to describe its behavior, failure cases, and tradeoffs.

#### 5.6 Memory model
**What to cover:** happens-before, visibility, publication, concurrent data structures.

**Common interview questions:**
- What is happens-before?
- How safely publish an object?

**Practice task:** Build a small isolated example and be ready to describe its behavior, failure cases, and tradeoffs.

#### 5.7 Modern threads
**What to cover:** virtual threads use cases, thread-local considerations, blocking I/O.

**Common interview questions:**
- What problem do virtual threads solve?
- When not helpful?

**Practice task:** Build a small isolated example and be ready to describe its behavior, failure cases, and tradeoffs.

**Section completion test:** Explain all the questions in this section without notes and demonstrate one or more working examples.

---

## 6. JVM and Diagnostics
**Estimated time:** 12–16 hours

### Subtopics and common interview questions

#### 6.1 Runtime architecture
**What to cover:** classloaders, heap, stack frames, metaspace, bytecode/JIT.

**Common interview questions:**
- Heap vs stack?
- How does class loading work?

**Practice task:** Build a small isolated example and be ready to describe its behavior, failure cases, and tradeoffs.

#### 6.2 Garbage collection
**What to cover:** GC roots, reachability, generations, G1 concepts, pause times.

**Common interview questions:**
- How can Java have memory leaks?
- Explain GC roots.

**Practice task:** Build a small isolated example and be ready to describe its behavior, failure cases, and tradeoffs.

#### 6.3 Troubleshooting
**What to cover:** OOM types, stack overflow, heap dump, thread dump, JFR, GC logs.

**Common interview questions:**
- How diagnose high heap usage?
- How find thread deadlock?

**Practice task:** Build a small isolated example and be ready to describe its behavior, failure cases, and tradeoffs.

#### 6.4 Performance awareness
**What to cover:** allocation rates, boxing overhead, CPU vs I/O bottlenecks.

**Common interview questions:**
- What would you inspect when API latency increases?

**Practice task:** Build a small isolated example and be ready to describe its behavior, failure cases, and tradeoffs.

**Section completion test:** Explain all the questions in this section without notes and demonstrate one or more working examples.

---

## 7. SQL and PostgreSQL
**Estimated time:** 25–30 hours

### Subtopics and common interview questions

#### 7.1 Modeling
**What to cover:** tables, keys, constraints, relationships, normalization, denormalization.

**Common interview questions:**
- Primary vs unique key?
- Why normalize?

**Practice task:** Build a small isolated example and be ready to describe its behavior, failure cases, and tradeoffs.

#### 7.2 Query fundamentals
**What to cover:** SELECT, filtering, aggregation, JOIN types, NULL handling, CASE.

**Common interview questions:**
- WHERE vs HAVING?
- INNER vs LEFT JOIN?
- How does NULL affect comparisons?

**Practice task:** Build a small isolated example and be ready to describe its behavior, failure cases, and tradeoffs.

#### 7.3 Advanced queries
**What to cover:** subqueries, CTEs, window functions, UNION/UNION ALL.

**Common interview questions:**
- ROW_NUMBER vs RANK?
- Query top N per group?

**Practice task:** Build a small isolated example and be ready to describe its behavior, failure cases, and tradeoffs.

#### 7.4 Indexes
**What to cover:** B-tree, composite, index selectivity, EXPLAIN ANALYZE, covering index.

**Common interview questions:**
- Why might a query ignore an index?
- Composite index order?

**Practice task:** Build a small isolated example and be ready to describe its behavior, failure cases, and tradeoffs.

#### 7.5 Transactions
**What to cover:** ACID, isolation levels, lost update, dirty/nonrepeatable/phantom reads.

**Common interview questions:**
- Explain isolation levels
- How avoid lost updates?

**Practice task:** Build a small isolated example and be ready to describe its behavior, failure cases, and tradeoffs.

#### 7.6 Locks and concurrency
**What to cover:** row locks, deadlocks, optimistic/pessimistic choices.

**Common interview questions:**
- How does SELECT FOR UPDATE work?
- What causes database deadlocks?

**Practice task:** Build a small isolated example and be ready to describe its behavior, failure cases, and tradeoffs.

#### 7.7 Production SQL
**What to cover:** pagination, keyset vs offset, migrations, connection pooling, N+1.

**Common interview questions:**
- How optimize a slow SQL query?
- Why can offset pagination be slow?

**Practice task:** Build a small isolated example and be ready to describe its behavior, failure cases, and tradeoffs.

**Section completion test:** Explain all the questions in this section without notes and demonstrate one or more working examples.

---

## 8. Spring Core and Spring Boot
**Estimated time:** 35–45 hours

### Subtopics and common interview questions

#### 8.1 IoC and DI
**What to cover:** ApplicationContext, bean registration, scopes, lifecycle, constructor injection.

**Common interview questions:**
- How does DI work?
- @Bean vs @Component?
- Singleton bean thread safety?

**Practice task:** Build a small isolated example and be ready to describe its behavior, failure cases, and tradeoffs.

#### 8.2 Boot internals
**What to cover:** starters, auto-configuration, conditional annotations, application startup.

**Common interview questions:**
- What happens on SpringApplication.run()?
- How does auto-configuration work?

**Practice task:** Build a small isolated example and be ready to describe its behavior, failure cases, and tradeoffs.

#### 8.3 Configuration
**What to cover:** profiles, property sources, @ConfigurationProperties, external secrets.

**Common interview questions:**
- @Value vs @ConfigurationProperties?
- Profile precedence?

**Practice task:** Build a small isolated example and be ready to describe its behavior, failure cases, and tradeoffs.

#### 8.4 Web MVC
**What to cover:** DispatcherServlet, controller, service, repository, mapping, DTOs.

**Common interview questions:**
- Trace an HTTP request through Spring MVC
- @Controller vs @RestController?

**Practice task:** Build a small isolated example and be ready to describe its behavior, failure cases, and tradeoffs.

#### 8.5 Validation and errors
**What to cover:** Bean Validation, @Valid, exception advice, meaningful error contracts.

**Common interview questions:**
- How implement global exception handling?
- 400 vs 422?

**Practice task:** Build a small isolated example and be ready to describe its behavior, failure cases, and tradeoffs.

#### 8.6 Cross-cutting
**What to cover:** filters, interceptors, AOP proxies, @Async, scheduler.

**Common interview questions:**
- Filter vs Interceptor vs AOP?
- Why does self-invocation bypass advice?

**Practice task:** Build a small isolated example and be ready to describe its behavior, failure cases, and tradeoffs.

#### 8.7 Transactions
**What to cover:** @Transactional, propagation, rollback, proxying, transaction boundaries.

**Common interview questions:**
- REQUIRED vs REQUIRES_NEW?
- Why checked exceptions may not trigger rollback?

**Practice task:** Build a small isolated example and be ready to describe its behavior, failure cases, and tradeoffs.

#### 8.8 Operations
**What to cover:** Actuator, health endpoints, logging, graceful shutdown.

**Common interview questions:**
- How expose a secure health check?

**Practice task:** Build a small isolated example and be ready to describe its behavior, failure cases, and tradeoffs.

**Section completion test:** Explain all the questions in this section without notes and demonstrate one or more working examples.

---

## 9. REST, HTTP and Networking
**Estimated time:** 15–20 hours

### Subtopics and common interview questions

#### 9.1 HTTP semantics
**What to cover:** GET/POST/PUT/PATCH/DELETE, codes, safe/idempotent methods.

**Common interview questions:**
- PUT vs PATCH?
- 401 vs 403?
- 200 vs 201 vs 204?

**Practice task:** Build a small isolated example and be ready to describe its behavior, failure cases, and tradeoffs.

#### 9.2 API contracts
**What to cover:** resource names, DTO validation, versioning, pagination and filtering.

**Common interview questions:**
- How design consistent error responses?
- How evolve APIs safely?

**Practice task:** Build a small isolated example and be ready to describe its behavior, failure cases, and tradeoffs.

#### 9.3 Web mechanics
**What to cover:** headers, cookies, sessions, HTTP caching, CORS, TLS, DNS.

**Common interview questions:**
- CORS vs CSRF?
- What happens when client calls HTTPS URL?

**Practice task:** Build a small isolated example and be ready to describe its behavior, failure cases, and tradeoffs.

#### 9.4 Reliability
**What to cover:** timeouts, retries, idempotency keys, rate limiting, webhooks.

**Common interview questions:**
- How prevent duplicate payment on retry?
- When should you not retry?

**Practice task:** Build a small isolated example and be ready to describe its behavior, failure cases, and tradeoffs.

#### 9.5 Client communication
**What to cover:** RestClient/WebClient basics, connection pools, synchronous vs asynchronous.

**Common interview questions:**
- WebClient vs RestClient?
- What is a connection timeout?

**Practice task:** Build a small isolated example and be ready to describe its behavior, failure cases, and tradeoffs.

**Section completion test:** Explain all the questions in this section without notes and demonstrate one or more working examples.

---

## 10. Spring Data JPA and Hibernate
**Estimated time:** 25–30 hours

### Subtopics and common interview questions

#### 10.1 ORM foundation
**What to cover:** JPA specification vs Hibernate implementation; Spring Data repositories.

**Common interview questions:**
- JPA vs Hibernate vs Spring Data JPA?

**Practice task:** Build a small isolated example and be ready to describe its behavior, failure cases, and tradeoffs.

#### 10.2 Entity mapping
**What to cover:** @Entity, IDs, generation, relationships, mappedBy, join columns.

**Common interview questions:**
- Owning side of bidirectional relation?
- Cascade vs orphanRemoval?

**Practice task:** Build a small isolated example and be ready to describe its behavior, failure cases, and tradeoffs.

#### 10.3 Persistence lifecycle
**What to cover:** transient/managed/detached/removed, dirty checking, flush.

**Common interview questions:**
- save vs saveAndFlush?
- What triggers an SQL UPDATE?

**Practice task:** Build a small isolated example and be ready to describe its behavior, failure cases, and tradeoffs.

#### 10.4 Fetching
**What to cover:** LAZY/EAGER, LazyInitializationException, N+1, fetch join/entity graphs.

**Common interview questions:**
- What is N+1 and how fix it?
- Why avoid eager loading everywhere?

**Practice task:** Build a small isolated example and be ready to describe its behavior, failure cases, and tradeoffs.

#### 10.5 Querying
**What to cover:** JPQL, native SQL, derived queries, projections, specification/pagination.

**Common interview questions:**
- JPQL vs SQL?
- How return DTO without loading full entity?

**Practice task:** Build a small isolated example and be ready to describe its behavior, failure cases, and tradeoffs.

#### 10.6 Transactional consistency
**What to cover:** @Transactional, optimistic @Version, pessimistic locking.

**Common interview questions:**
- How handle concurrent updates?
- What is OptimisticLockException?

**Practice task:** Build a small isolated example and be ready to describe its behavior, failure cases, and tradeoffs.

#### 10.7 Performance
**What to cover:** batch writes, JDBC batching, first/second-level cache, connection pool.

**Common interview questions:**
- Why can Hibernate create many queries?
- L1 vs L2 cache?

**Practice task:** Build a small isolated example and be ready to describe its behavior, failure cases, and tradeoffs.

**Section completion test:** Explain all the questions in this section without notes and demonstrate one or more working examples.

---

## 11. Spring Security
**Estimated time:** 18–24 hours

### Subtopics and common interview questions

#### 11.1 Core concepts
**What to cover:** authentication vs authorization, filter chain, SecurityFilterChain.

**Common interview questions:**
- How does Spring Security filter chain work?
- 401 vs 403?

**Practice task:** Build a small isolated example and be ready to describe its behavior, failure cases, and tradeoffs.

#### 11.2 Credentials
**What to cover:** password storage, BCrypt/Argon2, user details, brute force protection.

**Common interview questions:**
- Why not encrypt passwords?
- How protect login?

**Practice task:** Build a small isolated example and be ready to describe its behavior, failure cases, and tradeoffs.

#### 11.3 JWT
**What to cover:** header/payload/signature, expiry, signing, access/refresh tokens, rotation.

**Common interview questions:**
- Is a JWT encrypted?
- How revoke JWT?
- Where validate claims?

**Practice task:** Build a small isolated example and be ready to describe its behavior, failure cases, and tradeoffs.

#### 11.4 OAuth2 and OIDC
**What to cover:** authorization vs identity, authorization code with PKCE, bearer tokens.

**Common interview questions:**
- OAuth2 vs OIDC?
- Why PKCE?

**Practice task:** Build a small isolated example and be ready to describe its behavior, failure cases, and tradeoffs.

#### 11.5 Application security
**What to cover:** CSRF, CORS, XSS, SQL injection, BOLA/IDOR, least privilege.

**Common interview questions:**
- How prevent users accessing other users records?
- Does JWT prevent CSRF?

**Practice task:** Build a small isolated example and be ready to describe its behavior, failure cases, and tradeoffs.

#### 11.6 Production practices
**What to cover:** secure headers, secrets, audit logs, TLS and rate limits.

**Common interview questions:**
- How store secrets securely?
- Which data should never be logged?

**Practice task:** Build a small isolated example and be ready to describe its behavior, failure cases, and tradeoffs.

**Section completion test:** Explain all the questions in this section without notes and demonstrate one or more working examples.

---

## 12. Microservices and Reliability
**Estimated time:** 22–28 hours

### Subtopics and common interview questions

#### 12.1 Service decomposition
**What to cover:** monolith vs microservices; bounded ownership; deployment tradeoffs.

**Common interview questions:**
- When are microservices a bad choice?
- Why separate databases?

**Practice task:** Build a small isolated example and be ready to describe its behavior, failure cases, and tradeoffs.

#### 12.2 Communication
**What to cover:** REST calls, asynchronous events, API Gateway, service discovery.

**Common interview questions:**
- Sync vs async communication?
- API Gateway role?

**Practice task:** Build a small isolated example and be ready to describe its behavior, failure cases, and tradeoffs.

#### 12.3 Resilience
**What to cover:** timeouts, retry/backoff+jitter, circuit breaker, bulkhead, fallback.

**Common interview questions:**
- How stop cascading failures?
- What errors are safe to retry?

**Practice task:** Build a small isolated example and be ready to describe its behavior, failure cases, and tradeoffs.

#### 12.4 Consistency
**What to cover:** eventual consistency, saga, outbox, duplicate processing.

**Common interview questions:**
- How handle failure mid-workflow?
- Why transactional outbox?

**Practice task:** Build a small isolated example and be ready to describe its behavior, failure cases, and tradeoffs.

#### 12.5 Observability
**What to cover:** correlation IDs, distributed tracing, health checks.

**Common interview questions:**
- How trace one request through services?

**Practice task:** Build a small isolated example and be ready to describe its behavior, failure cases, and tradeoffs.

**Section completion test:** Explain all the questions in this section without notes and demonstrate one or more working examples.

---

## 13. Kafka, Redis and Messaging
**Estimated time:** 20–25 hours

### Subtopics and common interview questions

#### 13.1 Messaging models
**What to cover:** queues vs topics, durable vs ephemeral messages, pub/sub.

**Common interview questions:**
- Kafka vs RabbitMQ?
- When use async messaging?

**Practice task:** Build a small isolated example and be ready to describe its behavior, failure cases, and tradeoffs.

#### 13.2 Kafka fundamentals
**What to cover:** brokers, topics, partitions, keys, offsets, consumer groups.

**Common interview questions:**
- How Kafka maintains order?
- Why partition keys matter?

**Practice task:** Build a small isolated example and be ready to describe its behavior, failure cases, and tradeoffs.

#### 13.3 Delivery and failures
**What to cover:** at-least-once, retries, DLT, rebalancing, idempotent consumers.

**Common interview questions:**
- How handle duplicate Kafka events?
- What happens on consumer failure?

**Practice task:** Build a small isolated example and be ready to describe its behavior, failure cases, and tradeoffs.

#### 13.4 Serialization
**What to cover:** JSON vs Avro, schema evolution, compatibility.

**Common interview questions:**
- How safely evolve event schemas?

**Practice task:** Build a small isolated example and be ready to describe its behavior, failure cases, and tradeoffs.

#### 13.5 Redis
**What to cover:** strings, hashes, sets, TTL, eviction, cache-aside, invalidation.

**Common interview questions:**
- How prevent stale cache?
- Cache-aside vs write-through?

**Practice task:** Build a small isolated example and be ready to describe its behavior, failure cases, and tradeoffs.

#### 13.6 Caching risk
**What to cover:** cache stampede, hot keys, distributed lock limitations.

**Common interview questions:**
- How mitigate cache stampede?
- Is a Redis lock sufficient for payments?

**Practice task:** Build a small isolated example and be ready to describe its behavior, failure cases, and tradeoffs.

**Section completion test:** Explain all the questions in this section without notes and demonstrate one or more working examples.

---

## 14. JUnit, Mockito and Integration Testing
**Estimated time:** 20–25 hours

### Subtopics and common interview questions

#### 14.1 Testing strategy
**What to cover:** unit/integration/e2e, test pyramid, deterministic tests.

**Common interview questions:**
- Unit vs integration test?
- What should you mock?

**Practice task:** Build a small isolated example and be ready to describe its behavior, failure cases, and tradeoffs.

#### 14.2 JUnit 5
**What to cover:** assertions, lifecycle, parameterized tests, nested tests.

**Common interview questions:**
- @BeforeEach vs @BeforeAll?
- Why parameterize tests?

**Practice task:** Build a small isolated example and be ready to describe its behavior, failure cases, and tradeoffs.

#### 14.3 Mockito
**What to cover:** mock/stub/spy, when/thenReturn, verify, argument captors.

**Common interview questions:**
- @Mock vs @InjectMocks?
- Spy vs mock?

**Practice task:** Build a small isolated example and be ready to describe its behavior, failure cases, and tradeoffs.

#### 14.4 Spring testing
**What to cover:** @WebMvcTest, MockMvc, @DataJpaTest, @SpringBootTest.

**Common interview questions:**
- WebMvcTest vs SpringBootTest?
- Why tests fail due to missing context?

**Practice task:** Build a small isolated example and be ready to describe its behavior, failure cases, and tradeoffs.

#### 14.5 Integration testing
**What to cover:** Testcontainers for PostgreSQL/Kafka, realistic fixtures, transaction checks.

**Common interview questions:**
- Why use Testcontainers instead of H2?
- How test a rollback?

**Practice task:** Build a small isolated example and be ready to describe its behavior, failure cases, and tradeoffs.

#### 14.6 Quality
**What to cover:** JaCoCo, static analysis, flaky tests, test naming and maintainability.

**Common interview questions:**
- Is 100% code coverage enough?
- How debug flaky integration tests?

**Practice task:** Build a small isolated example and be ready to describe its behavior, failure cases, and tradeoffs.

**Section completion test:** Explain all the questions in this section without notes and demonstrate one or more working examples.

---

## 15. Git, Maven, Docker, CI/CD, Linux and Cloud
**Estimated time:** 20–25 hours

### Subtopics and common interview questions

#### 15.1 Build
**What to cover:** Maven lifecycle, pom.xml, dependency scopes, transitive conflicts, BOM.

**Common interview questions:**
- mvn test vs verify?
- How resolve version conflicts?

**Practice task:** Build a small isolated example and be ready to describe its behavior, failure cases, and tradeoffs.

#### 15.2 Version control
**What to cover:** Git branching, merge/rebase, conflicts, cherry-pick, revert.

**Common interview questions:**
- merge vs rebase?
- How revert a bad production commit?

**Practice task:** Build a small isolated example and be ready to describe its behavior, failure cases, and tradeoffs.

#### 15.3 Linux
**What to cover:** permissions, file/process tools, journal/log reading, curl, ps, top, grep.

**Common interview questions:**
- How find which process occupies a port?
- How follow service logs?

**Practice task:** Build a small isolated example and be ready to describe its behavior, failure cases, and tradeoffs.

#### 15.4 Docker
**What to cover:** images, containers, layers, Dockerfile, multi-stage builds, compose.

**Common interview questions:**
- Image vs container?
- How reduce image size?

**Practice task:** Build a small isolated example and be ready to describe its behavior, failure cases, and tradeoffs.

#### 15.5 CI/CD
**What to cover:** pipeline stages, automated tests, artifact/image registry, deployment approvals.

**Common interview questions:**
- How would you create a CI pipeline for Spring Boot?

**Practice task:** Build a small isolated example and be ready to describe its behavior, failure cases, and tradeoffs.

#### 15.6 Cloud and orchestration basics
**What to cover:** AWS EC2/RDS/IAM/CloudWatch; Kubernetes Pod/Deployment/Service/probes.

**Common interview questions:**
- What is a readiness probe?
- What is least-privilege IAM?

**Practice task:** Build a small isolated example and be ready to describe its behavior, failure cases, and tradeoffs.

#### 15.7 Runtime monitoring
**What to cover:** metrics, structured logs, traces, alerts, troubleshooting.

**Common interview questions:**
- How diagnose a running container repeatedly restarting?

**Practice task:** Build a small isolated example and be ready to describe its behavior, failure cases, and tradeoffs.

**Section completion test:** Explain all the questions in this section without notes and demonstrate one or more working examples.

---

## Suggested capstone: Appointment Booking Backend

Build a Spring Boot application with user accounts, role-based permissions, appointment availability, booking confirmation, mock payments, and notifications. Use PostgreSQL, Spring Data JPA, JWT, Redis caching, Kafka events, JUnit/Mockito/Testcontainers, Docker Compose, and a CI workflow. This is a learning project, not a description of any existing system.

### Practical interview questions from your project
- How did you prevent two clients from booking the same slot?
- What happens if the payment succeeds but your service crashes before saving the result?
- Why did you create a service layer rather than querying repositories from controllers?
- Which tables and indexes did you create, and why?
- How did you protect endpoints from cross-account data access?
- How did you test simultaneous booking attempts?
- How did you prevent duplicate event or webhook processing?
- How would you diagnose slow database queries and failing Kafka consumers?

---

## Verified reference starting points and YouTube discovery links

Official documentation is the source of truth for version-dependent behavior. YouTube search links are deliberately labeled as searches, so they do not pretend to be fixed course videos that may later disappear.

| Area | Documentation / reading | YouTube video search |
|---|---|---|
| Java language + collections + concurrency | [dev.java](https://dev.java/learn/) · [Oracle Java API](https://docs.oracle.com/en/java/javase/21/docs/api/) | [Telusko Core Java](https://www.youtube.com/results?search_query=Telusko+Core+Java+full+course) · [Engineering Digest collections](https://www.youtube.com/results?search_query=Engineering+Digest+Java+collections+framework) · [Engineering Digest multithreading](https://www.youtube.com/results?search_query=Engineering+Digest+Java+multithreading) |
| JVM diagnostics | [Java Flight Recorder](https://docs.oracle.com/en/java/javase/21/jfapi/) | [Java JVM garbage collection explained](https://www.youtube.com/results?search_query=Java+JVM+garbage+collection+heap+thread+dumps) |
| SQL + PostgreSQL | [PostgreSQL documentation](https://www.postgresql.org/docs/current/) · [PostgreSQL tutorial](https://www.postgresql.org/docs/current/tutorial.html) | [PostgreSQL joins indexes transactions](https://www.youtube.com/results?search_query=PostgreSQL+joins+indexes+transactions+tutorial) |
| Spring Core + Boot + REST | [Spring Boot Reference](https://docs.spring.io/spring-boot/reference/) · [Spring Guides](https://spring.io/guides) | [in28Minutes Spring Boot](https://www.youtube.com/results?search_query=in28minutes+Spring+Boot+REST+API+course) · [Telusko Spring Boot](https://www.youtube.com/results?search_query=Telusko+Spring+Boot+complete+course) |
| Hibernate / JPA | [Hibernate ORM docs](https://hibernate.org/orm/documentation/) · [Spring Data JPA docs](https://docs.spring.io/spring-data/jpa/reference/) | [Java Techie Spring Data JPA Hibernate](https://www.youtube.com/results?search_query=Java+Techie+Spring+Data+JPA+Hibernate) |
| Security | [Spring Security Reference](https://docs.spring.io/spring-security/reference/) · [OWASP API Security Top 10](https://owasp.org/API-Security/editions/2023/en/0x11-t10/) | [Spring Boot Security JWT OAuth2](https://www.youtube.com/results?search_query=Spring+Security+6+JWT+OAuth2+Spring+Boot+3+Java+Techie) |
| Microservices | [Spring Cloud](https://spring.io/projects/spring-cloud) · [Resilience4j](https://resilience4j.readme.io/) | [Java Techie microservices](https://www.youtube.com/results?search_query=Java+Techie+Spring+Boot+microservices+resilience4j) |
| Kafka + Redis | [Apache Kafka docs](https://kafka.apache.org/documentation/) · [Redis docs](https://redis.io/docs/latest/) | [Java Techie Kafka](https://www.youtube.com/results?search_query=Java+Techie+Kafka+Spring+Boot) · [Spring Redis cache](https://www.youtube.com/results?search_query=Spring+Boot+Redis+cache+Java+Techie) |
| Testing | [JUnit 5 guide](https://junit.org/junit5/docs/current/user-guide/) · [Mockito](https://site.mockito.org/) · [Testcontainers](https://testcontainers.com/guides/) | [Java Techie JUnit Mockito Spring Boot](https://www.youtube.com/results?search_query=Java+Techie+JUnit+Mockito+Spring+Boot+testing) |
| DevOps | [Maven docs](https://maven.apache.org/guides/) · [Docker docs](https://docs.docker.com/) · [Git docs](https://git-scm.com/doc) · [GitHub Actions](https://docs.github.com/en/actions) | [Spring Boot Docker GitHub Actions](https://www.youtube.com/results?search_query=Spring+Boot+Docker+GitHub+Actions+CI+CD+tutorial) |

**Interview-question banks:** [Baeldung](https://www.baeldung.com/) · [Java Guides](https://www.javaguides.net/) · [Spring Guides](https://spring.io/guides). Cross-check answers against the official manuals, especially for version-specific APIs.


## 20-week study outline (3 hours/day, 6 days/week)

| Weeks | Study focus | Output |
|---|---|---|
| 1–2 | Core Java, OOP | Java examples and interview notes |
| 3–4 | Collections, Generics, modern Java | Streams and Map internals practice |
| 5–6 | Concurrency, JVM | Thread-safe executor example |
| 7–8 | SQL and Spring Core | Relational schema + bootstrapped REST API |
| 9–10 | Spring Boot, REST | Validated CRUD APIs and structured errors |
| 11–12 | JPA / Hibernate | Mapping, transactions, query optimization |
| 13 | Spring Security | Authn/authz with secured endpoints |
| 14–15 | Microservices, Kafka, Redis | Event + caching proof of concept |
| 16 | JUnit, Mockito, integration tests | Controller/service/repository test suite |
| 17–18 | Maven, Git, Docker, CI/CD, deployment | Build-and-test CI pipeline |
| 19–20 | Backend interview revision | Question bank review, project walkthroughs |

## Revision checklist

- [ ] I can answer **why** as well as **what** for each concept.
- [ ] I can implement small Java examples without following a tutorial.
- [ ] I can explain Spring bean lifecycles, transaction proxies and the request flow.
- [ ] I can diagnose an N+1 query, SQL deadlock and concurrent update.
- [ ] I can show JWT authorization and resource ownership checks.
- [ ] I can write meaningful unit tests and Testcontainers integration tests.
- [ ] I can explain my own Dockerfile, build pipeline and logs.
- [ ] I can discuss tradeoffs and real failure modes from my capstone.

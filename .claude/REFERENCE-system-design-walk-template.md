# System Design Walk Template

> **Purpose**: a procedural sequence for designing a specific system (a new app, a new major component of the harness, a new skill with significant architectural surface). Where `REFERENCE-system-design.md` is the *vocabulary* of system design concepts, this is the *procedure* for applying them.
> **How to use**: walk a new system design through these steps in order. Each step has a clear output. Resist the temptation to skip ahead — the sequence is the discipline. If a later step surfaces a flaw in an earlier step, back up and revise rather than working around it.
> **Scope**: applies to both harness-layer (orchestrator core, knowledge graph, event log substrate, etc.) and app-layer (customer-facing apps each org ships) systems. Each step notes where the walk diverges between layers.
> **Revisit when**: starting a new system design; auditing an existing system design for completeness; coaching someone else through a design; structuring a design review.

---

## The five phases

| Phase | Steps | Purpose |
|---|---|---|
| **1. Frame the problem** | Overview, functional requirements, non-functional requirements | What are we building and what does success look like |
| **2. Design the components** | Data model, API, storage, algorithms | The pieces |
| **3. Synthesize** | High-level design | Compose the pieces into a working system |
| **4. Production readiness** | Failure modes, observability, security, deployment | Make it survive the real world |
| **5. Scale** | Bottleneck identification, scaling strategy | Handle growth deliberately |

A **decisions log** runs throughout — capture decisions as you make them, review at the end.

---

## Phase 1: Frame the problem

### Step 1: Overview

**Goal**: Name what you're building and why, in 2-5 sentences. Force the reader (including future-you) to understand the problem before reading the solution.

**Output**:
- One paragraph naming the system
- The problem it solves
- Who experiences the problem
- Why solving it now matters

**Draws from**: nothing yet — this is the entry point.

**Anti-patterns**:
- Starting with "we should build a [solution]" instead of "we have [problem]"
- Skipping this step because "everyone knows what we're building"
- Writing too much (more than 5 sentences means the problem isn't clear yet)

**Layer note**: Same shape at both layers. App-layer overviews tend to be customer-facing ("contractors lose 20% of their day to scheduling friction"); harness-layer overviews tend to be capability-facing ("the navigator can't find skills by capability without a query layer").

---

### Step 2: Functional requirements

**Goal**: List what the system must DO for its users (or callers). Bullet points, not prose.

**Output**: A bulleted list of "the system must..." statements. Each item is observable behavior, not implementation. Aim for 5-15 items at first; can grow during design.

**Draws from**: nothing — this is independent of any system design concept.

**Anti-patterns**:
- Conflating functional with non-functional ("must be fast" is non-functional, not functional)
- Writing implementation ("must use Redis" is not a requirement)
- Missing edge cases that are actually requirements ("must handle delivery failures gracefully")

**Layer note**:
- **App layer**: requirements are user-facing — "customer can book an appointment," "technician can mark a job complete"
- **Harness layer**: requirements are navigator-facing or system-facing — "the harness can spawn a clone with a specific skill loadout," "the event log can replay from any offset"

---

### Step 3: Non-functional requirements

**Goal**: List how WELL the system must do those things. This is where most beginner designs fail — non-functional requirements determine every architectural choice downstream.

**Output**: Specific, measurable targets for each of:
- **Scale**: peak QPS / events per second, total users / items / events, growth rate
- **Latency**: p50/p95/p99 targets for key operations
- **Availability**: uptime target (99%, 99.9%, 99.99%)
- **Consistency**: per-resource (strong, read-your-writes, eventual)
- **Durability**: RPO (acceptable data loss) and RTO (acceptable downtime)
- **Cost**: budget per request, per user, per month
- **Sovereignty / privacy**: data residency, who owns what
- **Security / threat model**: trust boundaries (full security thinking is Step 11; this names the level)

**Draws from**:
- CAP theorem and consistency tradeoffs entry — consistency choice
- Latency and throughput entry — latency targets, percentile thinking
- Replication entry — RPO/RTO framing
- System models entry — timing model, failure model assumptions

**Anti-patterns**:
- Vague targets ("must be fast" — fast for whom, what percentile?)
- Picking a number without justifying it ("99.99% because that sounds good")
- Skipping cost in commercial systems
- Forgetting sovereignty in harness-layer designs

**Layer note**:
- **App layer**: standard production targets (99.9% uptime is typical; latency under a few hundred ms; costs in dollars per user per month)
- **Harness layer**: sleep test is a non-functional requirement — "must survive the human walking away" is a hard liveness target. Sovereignty is non-negotiable, not a target.

---

## Phase 2: Design the components

### Step 4: Data model

**Goal**: Identify the entities, their attributes, and the relationships between them. This is upstream of database choice — it's about what concepts exist, not how they're stored.

**Output**:
- List of entities (User, Job, Skill, Event, etc.)
- Attributes for each entity
- Relationships between entities (1:1, 1:many, many:many)
- Lifecycle for each entity (created when, updated when, deleted when, never deleted)
- Ownership (who/what creates and owns this entity)

**Draws from**:
- Database selection §1 — data shape (graph-of-typed-nodes vs relational vs document)
- Indexes §3 — canonical access path per entity (name the access path now, not at storage time)
- Sharding §2 — shard key candidates (usually emerge from entity ownership)
- Service discovery §1 — naming (every entity gets a fully-qualified hierarchical name)

**Anti-patterns**:
- Jumping to database schema before modeling concepts
- Creating entities that exist only because the database needs them (ID-only join tables that don't represent a real concept)
- Missing the lifecycle (entities that "live forever" usually shouldn't)
- Missing the ownership (entities owned by no one usually shouldn't exist)

**Layer note**: Same shape at both layers. The harness layer often discovers that an entity should be a graph node rather than a row; the app layer often discovers that an entity should be event-sourced rather than CRUD.

---

### Step 5: API design

**Goal**: Specify the contract between the system and its callers. The API encodes what the callers actually need; everything downstream is in service of the API.

**Output**:
- List of operations (endpoints, MCP tools, function calls, events emitted)
- Input shape for each operation
- Output shape for each operation
- Error semantics
- Rate limits, idempotency keys, authentication if relevant

**Draws from**:
- API design entry — entire entry (style, transport, schema, versioning, pagination, error semantics)
- Service discovery §6 — multiple record types per name (same name, multiple kinds of answer)
- Resilience patterns §1 — idempotency (every operation should answer "what happens if called twice?")

**Anti-patterns**:
- Designing API as a thin wrapper over the database schema (the API should reflect what callers want, not what storage looks like)
- Skipping error semantics (errors are part of the API contract)
- Forgetting pagination on any operation that returns a list
- Over-versioning (one version per breaking change is enough)

**Layer note**:
- **App layer**: HTTP/REST or GraphQL for human-facing apps; the API surface is the most user-visible artifact
- **Harness layer**: MCP tools, in-process functions, event types — the API surface is mostly internal but the same discipline applies

---

### Step 6: Database / storage design

**Goal**: Given the data model and non-functional requirements, pick the storage substrate and design the schema (or projection layout, if event-sourced).

**Output**:
- Substrate choice (SQLite, Postgres, event log, file system, knowledge graph, vector store, etc.) with justification
- Schema or projection layout
- Index strategy (which queries are hot, which need indexes)
- Partitioning strategy (if any)
- Retention policy

**Draws from**:
- Database selection entry — the substrate decision
- Indexes entry — index data structure, partial indexes, canonical access path
- Transactions entry — ACID requirements, isolation level
- Replication entry — durability, sync vs async
- Sharding entry — partition strategy, hot keys
- Open table formats entry — if considering Iceberg / object storage
- TSDBs entry — if the workload is time-series-shaped
- Event streaming entry — if event-sourced
- WAL entry — durability primitive underneath

**Anti-patterns**:
- Picking a database before knowing the access patterns
- Defaulting to "Postgres" without justifying it
- Building indexes for queries that don't exist yet (premature indexing)
- Skipping retention (data that's "kept forever by default" usually shouldn't be)

**Layer note**:
- **Harness layer**: SQLite or Postgres for now, with the event log on top; future migration to Iceberg+DuckDB possible. Sovereignty matters.
- **App layer**: managed Postgres (Neon, Supabase) is the default for most apps; specialized substrates (Redis, vector DB) for specific use cases. Sovereignty is not a constraint.

---

### Step 7: Algorithms

**Goal**: Identify and design the load-bearing computational core. Some systems have a heavy algorithmic component (search, matching, ranking, routing, scheduling, recommendation); others are mostly CRUD with little algorithmic surface. This step exists to prevent the algorithmic core from being treated as an implementation detail.

**Output**:
- List of algorithms the system needs
- For each: the input, the output, the algorithm choice, the complexity
- Tradeoffs (accuracy vs speed, batch vs online, exact vs approximate)
- Whether to build, borrow, or use an existing library

**Draws from**:
- Query engines entry — if any algorithm is query-shaped
- TSDBs entry — if any algorithm is over time-series
- Latency and throughput entry — algorithm latency, percentile thinking, three-way tradeoff
- Resilience patterns §6 — rate limiting algorithms (token bucket, leaky bucket)
- Indexes §5 — graph traversal as a distinct algorithmic primitive
- `REFERENCE-scaling-laws.md` — if any algorithm is ML-shaped

**Anti-patterns**:
- Skipping this step because "we'll figure out the math when we get there"
- Building bespoke algorithms when good libraries exist
- Over-engineering for accuracy when approximate is fine
- Under-engineering for accuracy when exact matters (financial, gates, safety)

**Layer note**:
- **Harness layer**: leverage scoring, calibration, trace coreset selection, knowledge graph traversal, drift detection are the orchestrator's algorithmic core
- **App layer**: matching, ranking, routing, fraud detection, recommendation are typical app-level algorithmic cores

---

## Phase 3: Synthesize

### Step 8: High-level design (HLD)

**Goal**: Compose the components into a working system. Show how data flows through it, which components depend on which, and where the boundaries are.

**Output**:
- A diagram (text or visual) showing components and their connections
- A description of the request/event flow for each major use case
- A list of components (services, modules, processes, agents) with their responsibilities
- Dependency graph (which component depends on which)

**Draws from**:
- All previous steps — the HLD is the synthesis
- Sharding entry — component / service boundaries
- Event streaming entry — component coupling via events vs direct calls
- Service discovery entry — how components find each other
- Saga orchestration entry — multi-step workflows across components

**Anti-patterns**:
- Drawing the diagram first and reverse-engineering the components (the HLD should fall out of the prior steps)
- Hiding too much complexity in one box ("magic happens here")
- Showing too much detail (the HLD is the high level, not the implementation)
- Missing the failure paths (what does the diagram look like when each component fails?)

**Layer note**: Same shape at both layers. The harness HLD usually has fewer components but more event flow; the app HLD usually has more components (frontend, backend, database, cache, queue) with more standard topology.

---

## Phase 4: Production readiness

### Step 9: Failure modes

**Goal**: Enumerate what can fail and what happens when it does. This is the step most beginner designs skip, and it's the step that determines whether the system survives contact with reality.

**Output**:
- For each component: what failure modes it has (crash, hang, slow, partial, Byzantine)
- For each failure: what the system does (retry, fail over, degrade, alert, escalate)
- Recovery semantics (RTO, RPO from non-functional requirements)
- Compensating actions (what undoes a partial failure)

**Draws from**:
- Resilience patterns entry — idempotency, retry, timeouts, circuit breakers, bulkheads, backpressure, graceful degradation, compensation
- Saga orchestration entry — compensation, forward recovery, pivot recovery, backward recovery
- System models §2 — failure model (crash, timing, Byzantine)
- CAP theorem entry — per-operation CP vs AP choice when something fails
- Replication §6 — failover and recovery
- WAL entry — recovery via replay
- Disk I/O §5 — the underlying durability failures

**Anti-patterns**:
- Assuming components don't fail
- Bundling all failure handling into "we'll catch exceptions" (each failure mode needs a specific response)
- Forgetting the compensating action for non-idempotent operations
- Designing for crash failures only (slow / hung is the harder failure mode)

**Layer note**:
- **Harness layer**: sleep test is the dominant failure-handling concern — every failure mode must answer "what happens if this fails while the human sleeps?"
- **App layer**: standard production failure handling — alerts go to humans during business hours, on-call rotation handles after-hours

---

### Step 10: Observability

**Goal**: Design how you'll know the system is working, how you'll debug it when it isn't, and how you'll be alerted to problems.

**Output**:
- Logs: what gets logged, at what level, with what context
- Metrics: what's measured, what the SLOs are, what gets dashboarded
- Traces: what request flows are traced, what spans they have
- Alerts: what fires, when, to whom
- Dashboards: what views exist for what audiences

**Draws from**:
- Observability entry — entire entry (logs, metrics, traces, dashboards, alerts, OpenTelemetry, sampling, cardinality)
- Event streaming entry — logs and metrics as projections from the event log
- Resilience patterns §3 — heartbeat-based timeouts surface as observability primitive
- TSDBs entry — if metrics volume is high enough to need TSDB-style infrastructure

**Anti-patterns**:
- Adding observability after the system is built (instrument from day one)
- Logging everything at INFO level (overload destroys the signal)
- Alerting on noise (every alert should require human action)
- Cardinality explosion (per-user labels in metrics is the classic killer)

**Layer note**:
- **Harness layer**: the event log IS the observability infrastructure — logs, metrics, traces are projections from the event log
- **App layer**: standard observability stack — Sentry for errors, Datadog/Honeycomb/Vercel Analytics for metrics, structured logs

---

### Step 11: Security and threat model

**Goal**: Identify trust boundaries, threats, and defenses. Even systems with no adversarial users have threat models — the threats are bugs, compromised dependencies, and accidental misuse.

**Output**:
- Trust boundaries (where untrusted input enters the system)
- Authentication strategy (who's calling)
- Authorization strategy (what they can do)
- Input validation (what gets validated, where)
- Rate limiting and abuse defense (per-user, per-IP, global)
- Secrets management (where credentials live, how they rotate)
- Threat model (top 3-5 threats and how the system defends against each)

**Draws from**:
- System models §4 — compromised dependency model (LLM provider, MCP servers, web fetch results are all potentially malicious)
- API design §7 — idempotency, rate limiting, auth as API concerns
- Resilience patterns §6 — rate limiting as flow control
- The OWASP top 10 — universally applicable

**Anti-patterns**:
- "Internal system, no security needed" (compromised dependencies are still a threat)
- Storing secrets in code or config files (use a secrets manager or env vars, never commit)
- Trusting LLM output without verification (always verify; agents are soft-Byzantine per System models §2)
- Skipping rate limiting on expensive operations

**Layer note**:
- **Harness layer**: trust within an org, untrust across orgs, untrust against external dependencies (LLM, MCP, web). Verification gates ARE the security model.
- **App layer**: standard production auth (Clerk, Auth0, Supabase Auth), real authorization, real input validation, real rate limiting

---

### Step 12: Deployment and operations

**Goal**: Design how the system gets to production, how it gets updated, and how it gets operated.

**Output**:
- Deployment target (local, Vercel, Railway, custom)
- CI/CD pipeline (test, build, deploy)
- Release strategy (rolling, blue-green, canary, feature flag)
- Rollback strategy (how to revert a bad deploy)
- Backup and restore strategy
- Runbook entries (common operational tasks)

**Draws from**:
- Replication §6 — backup and restore framing (RTO dominated by setup time)
- Disk I/O §5 — backup integrity verification
- WAL entry — recovery via replay
- The orchestrator's `distribute` skill — for app-layer deployment

**Anti-patterns**:
- "We'll figure out deployment later" — production needs are part of the design, not an afterthought
- Manual deployment (anything that runs more than once should be automated)
- No rollback plan (every deploy should have an undo)
- No backup verification (untested backups are imaginary)

**Layer note**:
- **Harness layer**: `npm start` locally; backup is to the user's filesystem + maybe S3/R2. Sovereignty: no managed deployment platform.
- **App layer**: GitHub Actions → Vercel/Railway via the distribute skill. Standard production deployment.

---

## Phase 5: Scale

### Step 13: Scaling strategy

**Goal**: Identify the bottleneck and design how to relieve it. This is the LAST step because premature scale optimization is the most common system design failure.

**Output**:
- The current binding constraint (what saturates first as load grows)
- Headroom analysis (how much load can the current design handle)
- Scaling strategy (vertical, horizontal, sharding, caching, queue absorption)
- The next binding constraint (what becomes the bottleneck after the first scaling move)
- Cost projection at different scales

**Draws from**:
- Latency and throughput entry — Theory of Constraints, percentile thinking, three-way tradeoff
- Sharding entry — horizontal partitioning
- Caching entry — as a scaling strategy, not just a perf optimization
- Replication entry — read replicas for scale
- Load balancing entry — distributing across instances
- Resilience patterns §6 — backpressure, rate limiting as scale defenses
- TSDBs entry — if the workload is time-series-shaped at scale
- Open table formats entry — if the substrate needs to migrate at scale

**Anti-patterns**:
- Scaling the visible component instead of the binding constraint
- Adding caching as the first response (caching is downstream of correctness, not a substitute for it)
- Designing for hypothetical scale (10x current, not 1000x)
- Skipping cost analysis (scale has dollar consequences)

**Layer note**:
- **Harness layer**: scale is mostly about multi-clone parallelism, not multi-user. The binding constraint is usually the LLM provider rate limit.
- **App layer**: standard production scaling — horizontal app instances behind a load balancer, read replicas, caching tier, queue for spiky workloads

---

## Throughout: Decisions log

**Goal**: Capture decisions as you make them, so future-you (or future-collaborators) understand why the system is the way it is. This isn't a step at the end — it runs throughout the walk.

**Output**: A running log with one entry per decision:
- The decision (what was decided)
- The alternatives (what else was considered)
- Why this option won (the load-bearing reason)
- When to revisit (the condition under which the decision should be re-examined)

**Anti-patterns**:
- Not capturing decisions (future-you will not remember why)
- Capturing only the chosen option (the alternatives matter for future context)
- No revisit condition (decisions become permanent by accident)

**Layer note**: Same shape at both layers. The decisions log is sometimes called an ADR (Architecture Decision Record) and there are templates for it; the orchestrator has its own decision-recording infrastructure via the knowledge graph and `record_decision` MCP tool.

---

## Anti-patterns of the walk itself

A few meta-mistakes that show up across multiple steps:

1. **Skipping ahead** — jumping to Step 6 (storage) before doing Step 4 (data model) is the most common one. The sequence is the discipline.
2. **Treating later steps as optional** — failure modes, observability, security, deployment are not "we'll get to it" steps. They're load-bearing.
3. **Designing for the easy case** — every step should consider the failure case, the edge case, the scale case.
4. **No iteration** — if Step 9 surfaces a flaw in Step 4, back up and fix Step 4 rather than working around it.
5. **No decisions log** — the walk produces decisions; if they're not captured, the design becomes opaque to future readers.
6. **Premature HLD** — drawing the diagram first and reverse-engineering the components inverts the walk and produces designs that don't match what the requirements actually call for.

---

## When to apply this walk

**Required**:
- New apps each org ships (servicegrid customer agent, pilgrimage game, dreamtime service)
- New major harness components (a new query layer, a new substrate migration, a new MCP server)
- Significant architectural changes to existing components

**Recommended**:
- Skills with significant architectural surface (more than ~3 files, persistent state, external dependencies)
- Loops with significant new behavior (new phases, new gate types, new state transitions)

**Optional**:
- Small skills, bug fixes, content changes, refactors that don't change architecture

---

## Cross-references

- **`REFERENCE-system-design.md`** — the system design vocabulary (concepts, options, tradeoffs). Each step in this walk template draws from specific entries in that doc.
- **`REFERENCE-navigators-cosmos.md`** — the cosmos framing that determines harness-layer architectural choices.
- **`REFERENCE-scaling-laws.md`** — relevant for any system with an ML/AI component.
- **`DREAM-STATE-ooda.md`** — the architectural commitments the design must respect.

---

## Possible follow-up: encode as a skill

This walk template could be encoded as an orchestrator skill (`system-design-walk`) where each step is a sub-skill and the gates between steps prevent skipping ahead. The skill would:

- Take a system name as input
- Walk the user through each step in order
- Produce a system design doc as output
- Capture the decisions log as a side effect into the knowledge graph
- Refuse to advance to the next step until the current step's output is captured

This would make the walk template enforceable rather than aspirational. Worth doing if and only if the walk template proves useful in practice first.

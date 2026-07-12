# Dakar — Implementation Backlog

**Status:** Living document v0.1 — updated in the same PR as the work it tracks
**Authorities:** [system_architecture.md](system_architecture.md) (system), [dakar_internals.md](dakar_internals.md) (domain module), [model.md](model.md) (formal net: definitions, proofs, refinement), [agentic.md](agentic.md) (agent layer: episodes, tools, autonomy). A task that contradicts any of them is a defect in one of the documents; fix the disagreement first.

> **Standing constraint (since 2026-07-11, amended 2026-07-12):** development is
> **local-only** — no AWS deployment. The `cdk` module may be **modified but never
> executed** (no `synth`, no `deploy`, no deploy scripts) pending explicit
> authorization. Tasks marked ⛔ are blocked on that authorization; local
> equivalents are noted inline. Local database: `ops/docker-compose.yml`
> (`cockroachdb/cockroach:v23.2.31`, user-pinned).

---

## 0. Conventions

- **Task ID** `P<phase>-<n>`. One task ≈ one PR. If a task won't fit in a reviewable PR, split it and amend this file.
- **Size** S (≤ half day), M (≈ a day), L (multiple days — should usually be split).
- **State** ☐ open · ◐ in progress · ☑ done · ✕ dropped (struck through, with a note).
- **Global Definition of Done** (applies to every task, per `CLAUDE.md`): complete implementation (no TODOs/placeholders), tests included and green in `mvn verify`, typed error handling (no swallowed exceptions), no magic values, docs (`system_architecture.md` / `dakar_internals.md` / this file) amended if the task changed a contract.
- **Module names:** `dakar` = pure domain library · `dakar-quarkus` = product API + shell adapters · `dakar-agents` = agent Lambda(s), created in Phase 5 · `cdk` = infrastructure.

### Sequencing rationale

Two tracks run in parallel after Phase 0: the **domain track** (P1 → P3, pure Java, no infrastructure) and the **infra track** (P2 CDK/CockroachDB plumbing). They meet in the **walking skeleton** (end of P2): one thin, deployed, end-to-end slice — request a delivery, claim a truck inside a `SERIALIZABLE` firing, read the status. Everything after broadens a working system instead of assembling a big bang at the end.

```
P0 foundations
├── P1 domain primitives ──→ P3 full net ──→ P4 episodic memory ─┐
└── P2 walking skeleton  ─────────────────────────────────────────┼→ P5 agents → P6 hardening → P7 demo
     (needs P1 + a 2-transition net)                              ┘
```

---

## Phase 0 — Foundations & build guardrails

Goal: the laws of `dakar_internals.md` are mechanically enforced before the first domain class exists.

| ID | State | Task | Size | Depends |
|---|---|---|---|---|
| P0-1 | ☑ | **Toolchain & bytecode decision.** 2026-07-12 resolution: JDK 25 stays the build toolchain, but the whole reactor emits **`--release 21` bytecode** (root `maven.compiler.release=21`; dakar's module-local 25 pin removed). Forcing facts: Quarkus 3.20.1's bytecode tooling rejects class-file 69 ("Unsupported class file major version 69" in `mvn verify`), and `java21` is the newest managed Lambda runtime. The domain's language surface (records, sealed interfaces, switch patterns) is fully within 21. Revisit when the platform reads class-file 69; internals §2 amended. | S | — |
| P0-2 | ☑ | **`dakar` POM: dependency policy.** Empty compile scope (JDK + optionally `org.jspecify`); test scope JUnit 5, jqwik, ArchUnit, AssertJ. Maven Enforcer `<bannedDependencies>` failing the build on any compile-scope dependency (internals §2.1, §12). | S | — |
| P0-3 | ☑ | **ArchUnit law suite.** Purity (no infra packages), determinism (no `currentTimeMillis`/`nanoTime`/`randomUUID`/`Random`/`java.time now()`), closed package set, intra-module layering per internals §4 (future packages guarded via `allowEmptyShould`). Runs in `mvn test`. | M | P0-2 |
| P0-4 | ☑ | **Quality gate script.** 2026-07-12: `verify.sh` at repo root runs `./mvnw verify` across the reactor (`cdk` builds standalone and is not in the reactor; its deploys stay blocked). This is the merge gate until real CI exists; add a follow-up task when a CI system is chosen. | S | P0-1 |

---

## Phase 1 — Domain primitives (leaf packages)

Goal: the leaves of internals §4 — `money`, `geo`, `freight` — complete, tested, frozen.

| ID | State | Task | Size | Depends |
|---|---|---|---|---|
| P1-1 | ☑ | **Typed IDs + quantity types.** `TruckId`, `DeliveryId`, `ClientId`, `JournalEntryId` (Comparable, validated); `Tons` (scale 3, `freight/`), `Kilometers` (scale 3, lives in `geo/` — it is produced by distance math). | S | P0-3 |
| P1-2 | ☑ | **`Money`.** Fixed scale 2, `HALF_EVEN`, domain arithmetic (`plus`, `times`, `isAtLeast`), currency-mismatch rejection, non-negative by construction. Golden tests pin exact rounding behavior (internals §9). | S | P0-3 |
| P1-3 | ☑ | **`GeoPoint` + haversine.** Pure `distanceTo` over `StrictMath` (bit-identical across JVMs); golden tests with independently computed expected values; −0.0 normalized. | S | P1-1 |
| P1-4 | ☑ | **Color sets.** `Truck`, `Delivery` records with validating canonical constructors (floor > 0, version ≥ 1, agreed ≥ floor, payment ⇒ agreed); `TruckStatus`, `DeliveryStatus`, `CargoClass` enums — in `freight/`, not `desk/` (internals §4/§5.2 amended: keeps `freight` a leaf). | M | P1-1, P1-2, P1-3 |
| P1-5 | ☑ | **`DispatchBoard`.** Defensive `TreeMap` copies, unmodifiable views, key↔record-identity check, duplicate rejection. jqwik properties: iteration order independent of insertion order; every record retrievable by id. | S | P1-4 |
| P1-6 | ☑ | **Vocabulary firewall applied (2026-07-11, internals Law 4).** Shipped P1 code renamed to operational vocabulary: packages `token`→`freight`, `net`→`desk`; `TruckToken`→`Truck`, `DeliveryToken`→`Delivery`, `TruckPlace`→`TruckStatus`, `DeliveryPlace`→`DeliveryStatus`, `Marking`→`DispatchBoard`, `FiringId`→`JournalEntryId`; record component `place`→`status`; spec words purged from messages/comments/test names. ArchUnit gains the Law 4 naming rule (bans Token/Marking/Place/Firing/Transition/Guard/Binding/Petri in simple names). Spec↔code map: model.md §12. 56/56 tests green. | S | P1-5 |

---

## Phase 2 — Walking skeleton (thin vertical slice, deployed)

Goal: the smallest honest end-to-end system: `POST /v1/deliveries` → `Requested` token in CockroachDB → Dispatcher-less `ClaimTruck` fired by a staff endpoint → `GET /v1/deliveries/{id}` shows projected status. No LLM, no vectors, two transitions. This slice proves the plan/execute split, the `SERIALIZABLE` firing, and the abstraction boundary before we invest in breadth.

**Local-only re-scope (2026-07-11):** the skeleton runs entirely locally — Quarkus dev mode against a single-node CockroachDB (Docker/Testcontainers). The cloud legs (P2-2 ccloud, P2-7 CDK, deployed variant of P2-8) are ⛔ until authorized.

| ID | State | Task | Size | Depends |
|---|---|---|---|---|
| P2-1 | ☑ | **Minimal net.** 2026-07-12: `Action` sealed (`ClaimTruck` + `Cancel`); `RuleVerdict`/`RefusalReason` (reachable subset); `ActionOutcome` (`InvalidRequest` variant deferred to P3-1); `ActionPlan`; `StateWrite` shipped as a **typed sealed** pair (`TruckWrite`/`DeliveryWrite` — internals §7.4 amended away from the stringly `recordId/Object` shape); `FreightDesk.plan(action, snapshot, **actor**, ids)` (actor added so the journal entry is fully formed — internals §7.4/App. A amended); `requiredRoutes` + `JournalEntry`/`Evidence`/`Actor` live in `desk/` (a `journal/`-resident entry type would cycle with plans; internals §4 amended). `StatusProjection` pulled forward minimal-but-total. Rule tests table-driven; artifact contracts in dakar-javadoc.md. | M | P1-5 |
| P2-2 | ◐ | **Provision dev cluster via `ccloud`.** Cloud leg still ⛔ (local-only constraint). **Local stand-in shipped 2026-07-12:** `ops/docker-compose.yml` — single-node `cockroachdb/cockroach:v23.2.31` (pinned per direction), insecure/localhost-only, healthcheck + idempotent init job creating database `dakar` and user `dakar_app`; DB console on 8090. Tests use the same image via Testcontainers. ⚠ v23.2 predates `VECTOR`: the ≥ v25.2 check applies to the cloud cluster and to a local image bump before P4. *(Challenge tool #1: ccloud CLI — cloud leg.)* | M | authorization |
| P2-3 | ☑ | **Schema migrations.** 2026-07-12: Flyway `V1__operational_baseline.sql` in `dakar-quarkus`: `truck` (+ `truck_single_assignment` partial unique index — INV-5 storage layer, integration-tested), `delivery` (incl. `quote_expires_at`/`payment_due_at`, unmapped until P3-1), `operations_journal` (event-supplied `occurred_at` + `recorded_at`), `decision_trace`, **`api_request`** (product-API idempotency ledger — arch §6.1 amended). Tested via Testcontainers on the pinned image; DDL comments are Law 4-clean. | M | P2-1 |
| P2-4 | ☑ | **Snapshot loader.** 2026-07-12: `SnapshotLoader` reads both tables in one read-only transaction (no torn view), bounded by `dakar.board.max-records` (loud `BoardSizeExceededException`, never a partial board). Two task-text defects fixed per the fix-the-disagreement rule: (a) an *empty* `RouteTable` would refuse every claim with MISSING_ROUTE — routes are resolved via `requiredRoutes` + a dev `HaversineRouteEstimator` (`dakar.routing.average-speed-kmh`); (b) `Snapshot` ships as `(board, routes, now)` — the `PolicySet` member joins in P3-7 (internals §6 amended). | M | P2-1, P2-3 |
| P2-5 | ☑ | **ActionPlan executor.** 2026-07-12: one `SERIALIZABLE` transaction per plan (version-guarded `UPDATE`s + journal insert; truck updates never touch telemetry columns); `40001`/version-mismatch → fresh snapshot → re-enter the pure planner, bounded by `dakar.actions.max-attempts` with capped full-jitter backoff; typed `ExecutionResult` (`Completed`/`Refused`/`UnknownRecord`/`ContentionExhausted`). **The heart test is green**: two concurrent claims on one truck — exactly one commits, the loser is refused `TRUCK_ALREADY_ASSIGNED`, exactly one journal row. Also covered: stale-plan re-plan path, refusal passthrough (no journal row), storage-level double-assignment rejection. | L | P2-4 |
| P2-6 | ☑ | **Product API v0.** 2026-07-12: `POST /v1/deliveries` (field-typed 400s; required `Idempotency-Key`; raced/repeated keys return the original — `api_request` ledger; floor priced internally via dev cost model, never client-visible), `GET /v1/deliveries/{id}` via `StatusProjection`, staff-only `POST /v1/deliveries/{id}/assign` (constant-time `X-Staff-Access-Key` check — "token" avoided per Law 4 word list) executing `ClaimTruck` as `Actor.STAFF`; 200/409/404/503 mapped from `ExecutionResult`. Product vocabulary only. **Deferral:** delivery creation is not yet journaled — the `RequestDelivery` admin action joins with P3-4 replay wiring, until then the journal covers `ClaimTruck`/`Cancel` only. Old `/v1/info` scaffold resource removed. | M | P2-5 |
| P2-7 | ◐ | **CDK: real stack v0.** Constraint relaxed 2026-07-12 to *modify but do not execute*. Stack written and compiled (never synthed/deployed): Lambda `java21` from `FUNCTION_ASSET` (fail-fast when unset), HTTP API default integration, Secrets Manager `dakar/database-connection` (value set by an operator, never in code; Lambda receives only the ARN), `grantRead` as the sole IAM statement, one-month log retention, stack outputs. Enablers: `constructs` bumped 3.4.344 → 10.4.2 (CDK v2 requires 10.x) and stale local `dakar-bom` reinstalled. **Remaining ⛔: synth/deploy + the `function.zip` packaging (needs the Quarkus Lambda extension) await authorization.** | L | authorization |
| P2-8 | ☑ | **Skeleton acceptance test.** 2026-07-12: `ProductApiTest` runs the slice end-to-end against Testcontainers CockroachDB in `mvn verify` (repeatable, no manual steps): request → RECEIVED → staff assign → PREPARING_QUOTE, journal row with actor `staff`; plus idempotent-replay, busy-truck 409, staff-key 401s, and typed validation 400s. Re-run against AWS becomes a P2-7 follow-up once unblocked. | S | P2-6 |

---

## Phase 3 — The full net (domain complete)

Goal: every place, transition, and guard from arch §4.3–4.4; invariants executable; replay proven. Pure-Java work; parallelizable with Phase 4 infra tasks.

| ID | State | Task | Size | Depends |
|---|---|---|---|---|
| P3-1 | ☑ | **Remaining actions + rules (full net, model.md §4).** 2026-07-12: all 22 actions shipped — operational (`Quote`…`PaymentOverdue`), dispute/rescue (`ReleaseTruck`, `SettleDispute`, `WriteOff`, `TransferCargo`, `AbortLoaded`), `RecordDisruption`, and the administrative sub-net (`RegisterTruck`/`Decommission`/`Start`/`CompleteMaintenance` → new `StateWrite.TruckInsert`/`TruckDelete` variants; executor gained INSERT/DELETE + maps `23505` to lost-race). P1-4 catch-up landed (`DISPUTED`, `quoteExpiresAt`/`paymentDueAt` + status-coherence validation). Decisions (internals §7 v0.3.1): policy windows travel as action arguments until P3-7; staff-only enforcement at the desk (`STAFF_AUTHORIZATION_REQUIRED`); `RecordDisruption` bumps the delivery version (stale-plan invalidation); `ArriveOrigin` position guard discharged by the arrival signal, position journaled as witness (model.md §4.2 note); `RefusalReason` per-case names collapsed into `WRONG_STATUS`. Adversarial tests green: payment-before-completion on both arcs, floor price, stranded truck (overdue → release → write-off, plus late-payment settle), breakdown (`TransferCargo` atomic hand-over / `AbortLoaded`), quote expiry releasing the truck, staff-auth refusals, full request→completion fold. | L | P2-1 |
| P3-2 | ☐ | **`RouteTable` + `requiredRoutes`.** Routes-as-data per internals §6; `MISSING_ROUTE` guard reason; deterministic `RouteRequests` enumeration. | M | P3-1 |
| P3-3 | ☐ | **`BoardAudit` + property-based invariant suite.** jqwik: random legal firing sequences from random markings; INV-1…INV-5 checked after every step (internals §13) — INV-5 (assignment injectivity) is the no-double-booking property and gets targeted generators around `ClaimTruck` contention. This suite is the shipped verification of the CPN claims (arch §5 v0.2) — treat a counterexample as severity-1. | L | P3-1 |
| P3-4 | ☐ | **Events + replay.** `JournalEntry` incl. `Evidence` (**R-BINDING**, model.md §4.5: events carry route estimates, policy windows, and signal/position predicates so replay is closed over `(M₀, log)`); `Journal.replay` (total — throws on not-enabled entries); equality over the net-state projection, telemetry excluded (internals §10 v0.2). Properties: fold ≡ stepwise; drop/reorder/duplicate mutations fail loudly; reordering commuting (token-disjoint) events does **not** change the fold (model.md Theorem 4). | M | P3-1 |
| P3-5 | ◐ | **`StatusProjection` (full).** Exhaustive over `DeliveryStatus × TruckStatus`; test asserts no internal vocabulary in any output string (internals §11). *(2026-07-12: `DISPUTED → PAYMENT_OVERDUE` landed with P3-1; the truck-status-sensitive wording and the vocabulary assertion remain.)* | S | P3-1 |
| P3-6 | ☐ | **Enumeration.** `claimCandidates(Snapshot)` with annotations (repositioning distance, drive time, slack), `offerBand(...)`; determinism test: same snapshot twice → identical lists (internals §7.3). | M | P3-2 |
| P3-7 | ☐ | **`PolicySet` + floor-price formula evaluation.** Procedural memory as data (internals §15); versioned; `policy` table migration + loader in `dakar-quarkus`. | M | P3-1 |
| P3-8 | ☐ | **Wire full net into the shell.** Extend Flyway (any new columns), snapshot loader, and API: quote accept endpoint, payment-confirmation hook (stubbed signal), saga transitions fireable. Skeleton acceptance test extended to the full happy path through `Completed`. | M | P3-4, P3-6, P2-8 |

---

## Phase 4 — Episodic memory & vector retrieval

Goal: the negotiation-memory layer of arch §6.2–6.3 — real, co-transactional, retrievable. *(Challenge tool #2: distributed vector indexing.)*

| ID | State | Task | Size | Depends |
|---|---|---|---|---|
| P4-1 | ☐ | **Migrations: `negotiation_episode` (+ `VECTOR(1024)` + vector index), `client_profile`, `lane_stat`.** Dimension pinned to the chosen embedding model (see P4-2) — record the choice here. | M | P2-3 |
| P4-2 | ☐ | **Embedding adapter (Bedrock).** Titan Text Embeddings V2 (1024-dim) via `dakar-quarkus` shell; typed errors, timeout, retry with backoff; **vector never enters the `dakar` module** (internals §8.4). Decision note: if a different model is picked, update P4-1 dimension in the same PR. | M | P4-1 |
| P4-3 | ☐ | **Episode writer.** Domain emits `NegotiationEpisodeDraft` (transcript + structured fields); shell embeds and persists episode row **in the same transaction as the `Quote` firing** (arch MI-2: memory can never disagree with the ledger). Integration test kills the writer between embed and commit — no orphan state. | M | P4-2, P3-8 |
| P4-4 | ☐ | **Retrieval query.** k-NN over the vector index filtered by lane/cargo (cosine), joined with `client_profile` + `lane_stat` into the `NegotiationContext` shape (internals §8.2). Latency budget asserted in an integration test against seeded data (≥ 10k episodes — no toy-scale pass). | M | P4-3 |
| P4-5 | ☐ | **Seed + distillation job.** Synthetic-but-plausible episode corpus for dev/demo; scheduled distillation of `client_profile`/`lane_stat` from episodes (simple SQL aggregation, runs in shell). | M | P4-4 |

---

## Phase 5 — Agents

Goal: the three MVP agents of arch §8 — policies over enabled transitions, crash-only, human-approved where client-facing.

| ID | State | Task | Size | Depends |
|---|---|---|---|---|
| P5-1 | ☐ | **Decision: agent packaging.** Recommendation: new `dakar-agents` module (Quarkus, separate Lambda) for EventBridge-triggered work; `dakar-quarkus` keeps the HTTP surface. Alternatives (single function, one module per agent) considered and the choice + reasoning recorded **here** before P5-2 starts. | S | — |
| P5-2 | ☐ | **Bedrock tool-use adapters.** `BedrockDispatchPolicy` and `BedrockNegotiationPolicy` implement the domain ports as **bounded tool-use loops** (arch §8.2, internals §8.3a): transitions exposed as typed action tools, perception tools (episode search, client profile, lane stats, routes) served `AS OF SYSTEM TIME` of the episode snapshot; loop bounds from `policy` data; unknown tools / malformed arguments / disabled transitions rejected with typed tool errors and bounded re-deliberation — **never a guessed repair**; decision traces persisted per arch §6.2. Prompts/model IDs live here only. §8.3 validation exercised by hostile-fake tests plus one real-model contract test. | L | P5-1, P3-6, P4-4 |
| P5-3 | ☐ | **Dispatcher agent.** EventBridge rule on new `Requested` deliveries → enumerate `claimCandidates` → policy chooses → execute via executor; `40001` → re-plan loop; policy-violation and no-candidate paths emit typed events. Chaos test: kill mid-decision, re-drive, no duplicate claim (INV-5 holds by construction — test proves the plumbing). | L | P5-2 |
| P5-4 | ☐ | **Negotiator agent (human-in-the-loop).** Drafts offer within `offerBand`, with retrieval driven by the agent's **own perception tools** (arch §8.2/§8.4), not a pre-assembled context blob; writes draft + decision trace for staff; staff approval endpoint fires `Quote`. Below-floor proposals surface as rule refusals in metrics, not silent clamps (arch §8.4, internals §8.3). | L | P5-2, P4-3 |
| P5-5 | ☐ | **Saga driver.** EventBridge signals (`arrived`, `loaded`, `payment-settled`, injected `disruption`) → load snapshot → fire physical transition. Includes the ⏲ timeout scheduler firing `QuoteExpired`/`PaymentOverdue` (LIVE-1 fairness F1, arch §5). Disruption path is a **policy decision** — reassign (`Cancel` + re-dispatch) / wait / escalate per `saga_escalation_threshold` (arch §8.5) — recorded via `RecordDisruption` + decision trace, not a hardcoded response. Simulated-signal generator for dev/demo. | L | P5-1, P3-8 |
| P5-6 | ☐ | **Degradation test.** Bedrock adapter forced to fail: physical transitions still fire on signals; only optimization stalls (arch §11 "dumber but never unsafe and never stopped"). Automated, not a slide claim. | M | P5-3, P5-4, P5-5 |

---

## Phase 6 — Production hardening & operator surface

Goal: arch §11 made real; remaining challenge tooling wired. *(Challenge tool #3: MCP Server.)*

| ID | State | Task | Size | Depends |
|---|---|---|---|---|
| P6-1 | ☐ | **Per-agent service accounts + RBAC.** `ccloud`-scripted accounts per arch §11 grants table (Dispatcher, Negotiator, Saga, Product API); pipeline provisions idempotently; an integration test proves the Negotiator account *cannot* write `truck`. | M | P5-3, P2-2 |
| P6-2 | ☐ | **MCP Server (read-only) for operators.** Cloud Console config; audited; a documented operator runbook query set (live marking, stuck sagas, rule-refusal triage). Never exposed to product users (arch §9). | S | P2-2 |
| P6-3 | ☐ | **Observability.** Metrics: firing rate per transition, `40001` retry rate, rule-refusal rate per agent, policy-violation rate, tool-call rate + deliberation-loop length per agent, time-in-place for `Disputed` (F2 fairness made measurable — arch §11); structured logs at boundary crossings with correlation IDs (no PII); alarm thresholds documented. | M | P5-6 |
| P6-4 | ☐ | **Follower reads for projections.** `AS OF SYSTEM TIME follower_read_timestamp()` on status/dashboard reads; consistency caveat documented in the API contract (arch §7). | S | P3-8 |
| P6-5 | ☐ | **Replay audit job.** Scheduled: `Journal.replay(initial, journal)` vs live board (net-state projection); divergence pages as severity-1 (internals §10). | M | P3-4, P3-8 |
| P6-6 | ☐ | **Failure-mode drill script.** Automates the arch §11 table: mid-firing kill, double-claim race, below-floor proposal, Bedrock outage, bad policy version rollback. Output is the production-readiness evidence for judging. | M | P6-3 |
| P6-7 | ☐ | **S3 artifact store.** PODs / weighbridge tickets / contracts bucket, pre-signed upload from product API, keys referenced from `operations_journal` payloads. | M | P2-7 |

---

## Phase 7 — Challenge deliverables & demo

| ID | State | Task | Size | Depends |
|---|---|---|---|---|
| P7-1 | ☐ | **Demo scenario script.** One delivery end-to-end with an injected disruption; **centerpiece: kill the Negotiator Lambda mid-negotiation, show resume from CockroachDB with full context** (arch §13). Rehearsed, timed, reproducible from seed data. | M | P5-6, P6-6 |
| P7-2 | ☐ | **Submission README + diagrams.** Judging-criteria map (arch §12) with links to *evidence* (tests, drill outputs, metrics screenshots), tool usage summary (ccloud, vector index, MCP + Lambda/Bedrock/EventBridge/S3), setup instructions verified on a clean checkout. | M | P7-1 |
| P7-3 | ☐ | **Docs reconciliation pass.** system_architecture.md, dakar_internals.md, and this backlog agree with the shipped code; stale sections fixed or explicitly marked roadmap. | S | P7-2 |

---

## Post-MVP (parked, deliberately)

Not scheduled; listed so scope pressure has somewhere to point.

- **Backhaul Hunter** agent (deadhead reduction — the headline economic number; arch §8.4).
- **Disruption agent** with episodic disruption→resolution recall.
- **Autonomous negotiation** (removing the human-approval gate) — requires rule-refusal and policy-violation rates from P6-3 to be boring first.
- **Min-cost assignment solver** behind the Dispatcher; LLM ranks solver output.
- **TLA+ mini-spec of the contended `ClaimTruck`/`Offload` protocol**, model-checked in CI (arch §5 v0.2 — the jqwik suite P3-3 is the shipped verification; the spec is a complementary check of the concurrency protocol, and the architecture no longer claims it exists before it does).
- **Client-facing actuators** (send quote, payment link, driver notification — arch §8.6): extends the action boundary beyond the marking.
- **Multi-region survivability configuration** + region-failure drill.
- **Real CI system** replacing `verify.sh` gate (P0-4 follow-up).
- **Real routing adapter** (P3-2 ships with a static/dev `RouteTable` source).

---

## Risk register (watch while executing)

| Risk | Watched at | Mitigation |
|---|---|---|
| Java 25 Lambda runtime unavailable | P0-1 | Container-image Lambda or pin deployables to 21; domain stays 25-compatible either way |
| CockroachDB cluster version lacks vector indexing | P2-2 | Verify ≥ v25.2 *before* provisioning; it's cheap to check and expensive to re-provision |
| `40001` retry storms under agent fan-out | P2-5, P6-3 | Bounded retries + jittered backoff in the executor; retry-rate metric with alarm |
| LLM structured-output flakiness | P5-2 | Reject-don't-repair tool-call handling; §8.3 validation; violation-rate metric as model-health alarm |
| Tool-use loop runaway (cost/latency) | P5-2, P6-3 | Hard bounds from `policy` data (`max_perception_calls`, `max_action_attempts`); loop-length metric with alarm |
| Vector retrieval latency at scale | P4-4 | Latency budget asserted against ≥ 10k-row seed in CI, not discovered in the demo |
| Scope creep toward Post-MVP items | every phase | This file is the cut line; additions require a backlog PR first |

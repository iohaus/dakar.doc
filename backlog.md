# Dakar — Implementation Backlog

**Status:** Living document v0.1 — updated in the same PR as the work it tracks
**Authorities:** [system_architecture.md](system_architecture.md) (system), [dakar_internals.md](dakar_internals.md) (domain module), [model.md](model.md) (formal net: definitions, proofs, refinement), [agentic.md](agentic.md) (agent layer: episodes, tools, autonomy). A task that contradicts any of them is a defect in one of the documents; fix the disagreement first.

> **Standing constraint (since 2026-07-11, amended 2026-07-12):** development is
> **local-only** — no AWS deployment. The `cdk` module may be **modified but never
> executed** (no `synth`, no `deploy`, no deploy scripts) pending explicit
> authorization. Tasks marked ⛔ are blocked on that authorization; local
> equivalents are noted inline. Local database: `ops/docker-compose.yml`
> (`cockroachdb/cockroach:v25.2.20` since P4 — the pre-P4 vector-capability bump
> recorded at P2-2; the v23.2.31 pin served Phases 2–3. Dev volume recreated:
> 23.2 data cannot cross majors).

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
| P0-1 | ☑ | **Toolchain & bytecode decision.** 2026-07-12 resolution: JDK 25 toolchain with `--release 21` bytecode (Quarkus 3.20.1 rejected class-file 69; `java21` Lambda cap). **Superseded 2026-07-13:** the platform was upgraded to **Quarkus 3.37.2** (reads class-file 69) and the reactor now targets **Java 25** end to end (root `release=25`); Testcontainers moved to 2.x (`testcontainers-cockroachdb`), `quarkus-junit5`→`quarkus-junit`. The Lambda-runtime consequence returns to the P2-7 deploy decision (container image or 21-target deployable) — deploys remain blocked anyway. | S | — |
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
| P3-2 | ☑ | **`RouteTable` + `requiredRoutes`.** Landed across P2-4/P3-1: routes-as-data with `MISSING_ROUTE` (eligibility-gated), deterministic `RouteRequests` (sorted `RouteKey`s covering claim + rescue legs), dev haversine source in the shell. Real routing adapter stays Post-MVP. | M | P3-1 |
| P3-3 | ☑ | **`BoardAudit` + property-based consistency suite.** 2026-07-12: `check/` ships `BoardAudit` → `ConsistencyViolation(rule, detail)` covering assignment injectivity + target-activity + coverage, truck↔delivery phase coherence, holder capacity, and payment-before-completion (the executable INV-1…INV-5 + PHASE). jqwik `DeskConsistencyPropertyTest` drives seeded legal runs (`testing/LegalRuns`: random fleets/work, 40 steps drawn from all 22 actions incl. contention on shared trucks) and audits **every** intermediate board. Targeted corrupt-board tests pin each rule. A live-board violation is severity-1. | L | P3-1 |
| P3-4 | ☑ | **Events + replay.** 2026-07-12: `journal/` ships `Journal.replay` (total — `ReplayException` on any entry not allowed at its point; staff authority re-checked) over the operational-state projection; measurement rules are witnessed per R-BINDING (policy windows re-read from `appliedQuoteValidity`/`appliedPaymentTerms` evidence). Properties on seeded legal runs (`testing/LegalRuns`): replay ≡ desk fold; delivery-targeting drop/duplicate/order-sensitive-swap mutations are always detectable (throw or diverge). Swapping *commuting* entries is an equivalent history (model.md Theorem 4), deliberately not flagged. `RequestDelivery` creation-journaling remains with the P6-5 replay-audit wiring. | M | P3-1 |
| P3-5 | ☑ | **`StatusProjection` (full).** 2026-07-12: truck-aware wording — `AGREED` projects `DRIVER_EN_ROUTE` only while the truck is actually moving (EN_ROUTE_EMPTY/LOADING); otherwise the new product status `SCHEDULED` (internals §11 + arch §9 amended — "driver en route" was a lie for a garage-parked truck). Exhaustive test over `DeliveryStatus × (no truck + every TruckStatus)` plus the internal-vocabulary assertion on all product names. `DISPUTED → PAYMENT_OVERDUE` had landed with P3-1. | S | P3-1 |
| P3-6 | ☑ | **Enumeration.** 2026-07-12: `claimCandidates(Snapshot)` — only enabled pairs, ordered by (deliveryId, truckId), annotated with repositioning distance, total drive time, and driver-hours slack; `offerBand(deliveryId, snapshot)` — `[floor, ceiling]` from the pricing policy for MATCHED deliveries, `OfferBand.admits` refuses out-of-band values (never clamps). Determinism test: same snapshot twice → identical lists. | M | P3-2 |
| P3-7 | ☑ | **`PolicySet` + pricing formula.** 2026-07-12: domain `policy/` ships `PolicySet(pricing, quoteValidity, paymentTerms)` and `PricingPolicy` (floor = base + per-ton-km × tonnage × distance; ceiling = floor × multiplier); `Snapshot` gained its `policies` member (internals §6 v0.3.2). V2 migration creates the versioned `policy` table (partial unique index: one active version per key) and seeds v1 values from the former dev config; shell `PolicyStore` parses active rows (typed `PolicyConfigurationException` on gaps) and is read **inside the snapshot transaction** so board and policies are one view. `Quote`/`ArriveDest` dropped their window arguments; the applied windows are journaled as evidence witnesses. The shell's `DevFloorPriceModel` is deleted — the product API prices floors through the policy. | M | P3-1 |
| P3-8 | ☑ | **Wire full net into the shell.** 2026-07-12: schema was already v0.2.1-complete; API gained staff `POST /v1/deliveries/{id}/quote` (offer, floor-checked by the desk), client `POST /v1/quotes/{id}/accept` (`Actor.PRODUCT`), and staff `POST /v1/deliveries/{id}/signals` (DEPARTED/ARRIVED_AT_PICKUP/LOADED/ARRIVED_AT_DESTINATION/PAYMENT_CONFIRMED/OFFLOADED — the stubbed signal surface until the P5-5 coordinator); `ExecutionResponses` centralizes result→HTTP mapping. Acceptance test runs the full happy path request → assign → quote → accept → 5 signals → **DELIVERED**, asserting 8 journal entries and a freed truck; below-floor quote refused end-to-end (409 QUOTE_REFUSED). | M | P3-4, P3-6, P2-8 |

---

## Phase 4 — Episodic memory & vector retrieval

Goal: the negotiation-memory layer of arch §6.2–6.3 — real, co-transactional, retrievable. *(Challenge tool #2: distributed vector indexing.)*

| ID | State | Task | Size | Depends |
|---|---|---|---|---|
| P4-1 | ☑ | **Migrations: episodic + semantic memory.** 2026-07-13: V3 ships `negotiation_episode` (`VECTOR(1024)` + vector index + lane/cargo index; `outcome` open→won/lost), `client_profile`, `lane_stat`. Dimension pinned to **Titan Text Embeddings V2 (1024)**. Requires the vector-capable image: local stack bumped to `cockroachdb/cockroach:v25.2.20` with `feature.vector_index.enabled` (compose init + test resource); dev volume recreated (23.2 data cannot cross majors). Same day the platform moved to Quarkus 3.37.2 / Java 25 / Testcontainers 2.x (P0-1 note). | M | P2-3 |
| P4-2 | ◐ | **Embedding adapter.** 2026-07-13: shell `TranscriptEmbedder` (contract: 1024 dims) with two providers selected by `dakar.embedding.provider`: `LocalHashEmbedder` (deterministic hashed bag-of-words, unit-normalized — dev/test default) and `BedrockTranscriptEmbedder` (Titan V2 `InvokeModel`, typed `EmbeddingException` on transport/shape failures, client built only when selected; unit-tested against a fake client incl. request-contract assertions). Embeddings are computed **before** the action transaction — an outage 503s the request, never stores an unrecallable episode. Vectors never enter `dakar` (internals §8.4). **Remaining ⛔: the one real-model contract test needs AWS credentials (cloud leg).** | M | P4-1 |
| P4-3 | ☑ | **Episode writer.** 2026-07-13: executor gained `CompanionWrite` — extra rows that exist iff the action commits. Staff `quote` records the episode (outcome `open`) in the same transaction as the Quote journal entry; client accept marks it `won` and copies the agreed price in the same transaction as AcceptQuote; `markLost` ships and wires to cancellation/expiry surfaces with P5-5. Atomicity test: a failing companion rolls back journal entry and assignment alike. (The draft/transcript is assembled by the API for now — the staff quote is the whole negotiation until the P5 Negotiator; transcripts carry no personal data.) | M | P4-2, P3-8 |
| P4-4 | ☑ | **Retrieval query.** 2026-07-13: `EpisodeStore.retrieveContext` — kNN over the vector index (`<=>`, unit-normalized vectors so cosine≡L2 ranking), prefiltered by lane + cargo, won/lost only, joined with `client_profile`/`lane_stat` into the domain `NegotiationContext` (`SimilarEpisode` + `ClientFacts` + `LaneFacts`, package `agent/`). Latency budget asserted against a **10k-episode** seeded corpus: warm retrieval < 2s (green; the corpus load itself dominates the test's runtime and retries chunk-wise on serialization conflicts from index backfill). | M | P4-3 |
| P4-5 | ☑ | **Seed + distillation.** 2026-07-13: `SemanticDistillation` runs `EpisodeStore.distill()` on `dakar.distillation.every` (idempotent full re-aggregation; win-rate/avg-final-price per client, `percentile_cont`-median price-per-ton per lane — FLOAT8-cast, CockroachDB rejects DECIMAL there). Synthetic corpora: the 10k latency corpus and wording-controlled retrieval fixtures live in the test layer; the polished demo-seed CLI belongs to P7-1. | M | P4-4 |

---

## Phase 5 — Agents

Goal: the three MVP agents of arch §8 — policies over enabled transitions, crash-only, human-approved where client-facing.

| ID | State | Task | Size | Depends |
|---|---|---|---|---|
| P5-1 | ☑ | **Decision: agent packaging.** 2026-07-14: agents are hosted **inside `dakar-quarkus`** behind the Quarkus scheduler while cloud triggers are blocked — EventBridge cannot fire locally, and a separate Lambda module would be untestable dead weight until deploys are authorized. The `dakar-agents` Lambda module becomes the cloud-leg refactor (same domain ports, EventBridge rules replacing `@Scheduled`). Recorded here per the task's own requirement. | S | — |
| P5-2 | ☑ | **Policy adapters.** 2026-07-14: domain ports `DispatchPolicy`/`NegotiationPolicy` (+`DispatchContext`) with two providers behind `dakar.agents.provider`: deterministic `local` baselines (least repositioning; band midpoint — dev/test default) and **Bedrock Converse tool-use** adapters (`choose_truck`/`propose_offer` as forced tool calls; lazy client so missing credentials degrade at call time as `PolicyUnavailableException`, never at boot; un-offered trucks and unreadable amounts are discarded with warnings — reject, never repair). Fake-client unit tests pin the request contract and failure taxonomy; **live-model contract tests are ⛔ cloud.** | L | P5-1, P3-6, P4-4 |
| P5-3 | ☑ | **Dispatcher automation.** 2026-07-14: `DispatchAutomation.tick()` (`@Scheduled`, `dakar.agents.dispatch-every`; disabled toggle) — snapshot → `claimCandidates` per requested delivery → policy → **validated against the offered set** (hostile-fake test: outside choices are discarded, journaled nothing, traced `policy-violation`) → `ClaimTruck` as `DISPATCHER` with the `decision_trace` row co-committed with the claim (contention → typed refusal traces; `PolicyUnavailable` → tick skipped, degraded-not-broken test). EventBridge trigger remains the cloud leg. | L | P5-2 |
| P5-4 | ☑ | **Negotiator (human-in-the-loop).** 2026-07-14: staff-only `GET /v1/deliveries/{id}/quote-advice` — retrieval context (own embedding query per arch §8.2 spirit), `offerBand`, policy draft **rejected (never clamped)** when out of band (502 `ADVICE_REJECTED`, traced as a model-health signal); drafted/abstained outcomes traced under actor `negotiator`. The draft is advice: staff still fire the quote through the existing endpoint (acceptance test does exactly that round-trip). | L | P5-2, P4-3 |
| P5-5 | ☑ | **Lifecycle coordination (local shape).** 2026-07-14: `LifecycleCoordinator.tick()` (`@Scheduled`, `dakar.agents.lifecycle-every`) fires the ⏲ timeouts — `QuoteExpired` (episode marked lost in the same transaction) and `PaymentOverdue` — discharging fairness F1 without any client action; tested end-to-end (expired quote → CANCELLED + truck freed + episode lost; overdue payment → DISPUTED; journal actor `coordinator`). Signal-driven physical transitions remain on the staff `/signals` surface until EventBridge (cloud leg); disruption re-planning policy joins with the cloud-leg coordinator. | L | P5-1, P3-8 |
| P5-6 | ☑ | **Degradation test.** 2026-07-14: automated, not a slide claim — a throwing policy makes `DispatchAutomation` skip its tick with zero side effects (no journal entries, board untouched); the Bedrock adapters map every transport failure to `PolicyUnavailableException`; advice degrades to 503 `ADVICE_UNAVAILABLE`; staff assignment and signals are policy-independent by construction (separate code path). Dumber, never unsafe, never stopped. | M | P5-3, P5-4, P5-5 |

---

## Phase 6 — Production hardening & operator surface

Goal: arch §11 made real; remaining challenge tooling wired. *(Challenge tool #3: MCP Server.)*

| ID | State | Task | Size | Depends |
|---|---|---|---|---|
| P6-1 | ◐ | **Per-agent service accounts + RBAC.** 2026-07-18 local leg: `ops/service-accounts.sql` provisions `dispatcher_agent`/`negotiator_agent`/`coordinator_agent`/`product_api` with the arch §11 grants (idempotent; run after migrations); `ServiceAccountsTest` proves the negotiator principal **cannot** `UPDATE truck` while its pricing surface works. The `ccloud` pipeline provisioning of the same grants is the cloud leg (⛔). | M | P5-3, P2-2 |
| P6-2 | ◐ | **MCP Server (read-only) for operators.** 2026-07-18: the operator query set shipped as **operator-runbook.md** (live board, stuck work, refusal/decision triage, memory health, alarm sheet, divergence procedure). The Cloud Console MCP configuration itself is the cloud leg (⛔) — locally operators use `cockroach sql`/the DB console per the runbook. | S | P2-2 |
| P6-3 | ☑ | **Observability.** 2026-07-18: Micrometer/Prometheus at `/q/metrics` — `dakar.actions.executed{action,actor}`, `.refused`, `.lost.race`, `dakar.replay.divergence`, dispute-queue gauges (`dakar.disputes.open`, `.oldest.seconds`, refreshed on `dakar.metrics.refresh-every`) — F2 fairness made measurable. `X-Request-Id` correlation filter: honored inbound, generated otherwise, echoed on every response, on every log line via MDC (no PII). Alarm thresholds documented in the runbook §5 (replay divergence > 0 pages, severity-1). Agent tool-loop metrics arrive with the cloud-leg Bedrock loops. | M | P5-6 |
| P6-4 | ☑ | **Follower reads for projections.** 2026-07-18: staff `GET /v1/deliveries` (fleet dashboard, `dakar.staff.list-limit`) reads `AS OF SYSTEM TIME follower_read_timestamp()` — off the write hot path, may lag seconds (caveat in code, runbook, and test, which polls for visibility instead of pretending freshness). Point reads (`GET /{id}`, post-action reloads) stay fresh deliberately: read-your-writes is part of the product contract. | S | P3-8 |
| P6-5 | ☑ | **Replay audit job.** 2026-07-18: prerequisite first — **creations are now journaled**: new `RequestDelivery` action (+`StateWrite.DeliveryInsert`), `POST /v1/deliveries` runs through the executor with the `api_request` idempotency row as a companion write (raced keys resolve to the winner), and staff `POST /v1/trucks` fires `RegisterTruck`. `JournalPayloadCodec.fromJson` (total inverse, loud on unknown shapes) + `JournalStore` + scheduled `ReplayAudit` (`dakar.audit.every`): replays from the empty board and compares operational state (telemetry excluded per model.md §5.3); divergence → ERROR log + `dakar.replay.divergence` metric, severity-1. Tests: fully-journaled flow audits clean; tampered status and journal-unknown records are reported. | M | P3-4, P3-8 |
| P6-6 | ☑ | **Failure-mode drill script.** 2026-07-18: `ops/drills.sh` runs the named evidence suite — double-claim race, stale-plan re-plan, below-floor refusal, companion rollback, policy-outage degradation, journal tamper, and pricing-policy roll-forward/rollback (`OperationsSurfaceTest`). Output is the production-readiness evidence record. | M | P6-3 |
| P6-7 | ◐ | **S3 artifact store.** 2026-07-18 CDK leg (modify-not-run): `DeliveryArtifacts` bucket — block-all public access, SSE-S3, versioned, SSL-enforced, RETAIN — plus `dakar/staff-access-key` secret and CloudWatch error/throttle alarms; function gets bucket/secret grants and env wiring; stack compiles, never synthed. The pre-signed-upload API endpoint is the cloud leg (needs credentials to sign). | M | P2-7 |

---

## Phase U — User interface (operations console & client surface)

| ID | State | Task | Size | Depends |
|---|---|---|---|---|
| U-1 | ☑ | **Design authority.** 2026-07-18: `ui-design.md` shipped — activity-graph board (topological, not geographic), Nominatim geographic model (region-agnostic), full flow/screen/component/state specs, validated stage ramps + reserved health palette, repo-safe vocabulary (§16). High-fidelity previews in `dakar-ui/design/` (`palette.css`, styleguide/console/negotiation/client) — firewall-swept. | M | P6 |
| U-2 | ☐ | **Console read surface.** `GET /v1/trucks` (fleet), board view of deliveries, `GET /v1/activity` (journal + decision traces merged) — staff-only, follower reads, strict caps (ui-design §14.2) + RestAssured tests. | M | U-1 |
| U-3 | ☐ | **Serving & toolchain.** `dakar-ui/public` served same-origin from Quarkus (`META-INF/resources` wiring); vendor pixi.min.js + Source Sans 3 woff2; tsc-only browser ESM (`.js`-suffixed imports); npm for dev typings only (no `any`). | S | U-1 |
| U-4 | ☐ | **Console shell + board.** Frame/nav/access-key flow (sessionStorage), typed fetch + snapshot poll loop with backoff, board canvas per ui-design §10 (layers, DOM mirror, focus & context, reduced motion). | L | U-2, U-3 |
| U-5 | ☐ | **Operator flows.** Register/assign/quote-with-advice/signals forms, Negotiation queue, Activity screen, refusal callouts wired to real error codes (ui-design §6, §14.3). | M | U-4 |
| U-6 | ☐ | **Client surface + tests.** Request/track/accept with Nominatim geocoding (debounce, cache, attribution); `node:test` logic tests (projection, api client); a11y pass per ui-design §13. | M | U-4 |

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

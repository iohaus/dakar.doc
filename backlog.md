# Dakar — Implementation Backlog

**Status:** Living document v0.1 — updated in the same PR as the work it tracks
**Authorities:** [system_architecture.md](system_architecture.md) (system), [dakar_internals.md](dakar_internals.md) (domain module), [model.md](model.md) (formal net: definitions, proofs, refinement), [agentic.md](agentic.md) (agent layer: episodes, tools, autonomy). A task that contradicts any of them is a defect in one of the documents; fix the disagreement first.

> **Standing constraint, superseded 2026-08-12.** The original "local-only, no
> AWS deployment" constraint (2026-07-11, amended 2026-07-12) no longer
> reflects reality: **Dakar is deployed on AWS.** The no-execution CDK
> constraint was lifted and acted on; this banner previously went stale
> without being corrected, which is itself a lesson — update it the same day
> a constraint changes, not retroactively. Local database:
> `ops/docker-compose.yml` (`cockroachdb/cockroach:latest-v26.2`; this
> banner's `v25.2.20` pin had also drifted from the actual compose file).
>
> **CockroachDB Cloud remains ⛔**, separately from the AWS point above (a
> cloud cluster and AWS deployment are different things). The team's
> CockroachDB Cloud cluster hit its monthly Request Unit limit and is
> disabled; restoring/replacing it depends on the db team, no timeline. The
> live AWS deployment's DB connection is presumably still pointed at that
> disabled cluster (i.e. likely non-functional against the DB right now).
> Plan: **redeploy once the local compose cluster is hardened and
> port-forwarded**, pointing AWS at it instead (tracked below, P6-2).

> **Reconciliation pass, 2026-08-09.** Session-based staff auth (`AuthResource`,
> `SessionStore`, cookie login/logout) was removed — staff auth is the
> `X-Staff-Access-Key` header only (`StaffAccessFilter`), which is now correctly
> re-registered as an active JAX-RS provider (it had been left disabled
> mid-edit, silently leaving every `@StaffOnly` endpoint unprotected — fixed as
> part of this pass, not a design change). `SsmConfigSource` (AWS SSM Parameter
> Store resolution) was removed as dead code — never wired to any real secret
> flow; CDK injects secrets as plain Lambda env vars, not SSM. See Phase U's
> supersession note for the `dakar-ui`→`dakar-console` + curl-only-clients
> pivot. `aws.md` and `demo-commentary.md` corrected to match: Java 21 (not 25),
> Claude 3 Haiku (not 3.5 Sonnet, which was never the configured model), no
> `/login`/`/logout` routes, no web visualizer.

> **MCP integration, 2026-08-12 (P6-2).** `ops/docker-compose.yml` converted
> to CockroachDB secure mode (real CA/node/client cert hierarchy, generated
> and permission-fixed idempotently by `cockroach-cert-init`) — closes a
> latent bug found in the process: `cockroach-init` created user `eze` only,
> while Flyway/Quarkus both default to `dakar_app`, which was never created;
> this only "worked" under `--insecure` because password enforcement was
> off. `dakar_app` is now created for real; `eze` kept as a cert-authenticated
> convenience identity. New read-only `mcp_operator` principal added to
> `ops/service-accounts.sql`, scoped to exactly the 7 tables
> `operator-runbook.md` queries (`truck, delivery, operations_journal,
> decision_trace, negotiation_episode, client_profile, lane_stat`) —
> verified directly against the secured cluster: allowed reads succeed,
> excluded tables (`policy`, `api_request`) and all writes are rejected by
> CockroachDB itself. The self-hosted **CockroachDB MCP Server**
> (`github.com/cockroachdb/cockroachdb-mcp-server`, installed via `go
> install`, confirmed genuinely self-hostable against any cluster — not
> exclusive to CockroachDB Cloud) is wired via `.mcp.json`/`.mcp.json.example`
> (stdio transport, cert-based auth as `mcp_operator`) — sanity-checked
> against the live secured cluster (`"msg":"starting server"`); the live
> Claude-Code-side tool call itself needs a session restart to confirm, not
> yet done as of this note. Full `mvn verify` shows zero regressions, and a
> live `quarkus:dev` + `curl` smoke test against the secured cluster passed
> end to end (create, read, staff-auth enforcement). **Found but not fixed**
> (out of scope for this pass, flagged for the team): `.env`'s
> `DAKAR_DISPATCHER_DB_PASSWORD`/`DAKAR_NEGOTIATOR_DB_PASSWORD`/
> `DAKAR_COORDINATOR_DB_PASSWORD`/`DAKAR_DB_PASSWORD` are all currently the
> *same* literal value, contradicting `service-accounts.sql`'s own stated
> rationale for per-principal passwords; rotating them touches the live AWS
> deployment's env vars too, so it wasn't done unilaterally here.
> CockroachDB Cloud (MCP's "Cloud Managed" branded variant, and `ccloud` CLI)
> both remain ⛔ per the constraint banner above — the self-hosted path
> doesn't depend on either.

> **CockroachDB Cloud restored, 2026-08-13.** The cluster
> (`dakar-spindle-29010.j77.aws-us-east-1.cockroachlabs.cloud`) is back — the
> "CockroachDB Cloud remains ⛔" line above is stale as of this note,
> superseded here per this file's own stated practice (update the same day,
> not retroactively). Confirmed via a real `mvn quarkus:dev` connection, not
> `psql` — `sslfactory=org.postgresql.ssl.DefaultJavaSSLFactory` in
> `DAKAR_DB_URL` is a pgjdbc-specific parameter; only the actual Java client
> exercises that trust path correctly, and `psql`/libpq-based checks against
> the same URL fail for reasons that have nothing to do with the cluster
> (confirmed both ways this session — worth remembering before reading a
> `psql` failure against this cluster as a real outage). Getting to a clean
> start required one fix already precedented in this repo: `ALTER ROLE eze
> SET allow_unsafe_internals = true` was missing on the cloud side (Flyway's
> version-detection query needs it; `ops/docker-compose.yml`'s
> `cockroach-init` already carries the same grant locally, for the same
> reason) — applied directly against the cloud cluster via a one-off `psql`
> connection (`sslmode=require`, metadata/role-level change only).
>
> **Found, explicitly not fixed — pending the user's go-ahead (P6-9):** the
> cloud `public` schema already has all 9 tables through V3
> (`api_request`, `client_profile`, `decision_trace`, `delivery`,
> `lane_stat`, `negotiation_episode`, `operations_journal`, `policy`,
> `truck` — confirmed by a read-only `information_schema.tables` query) but
> has no `flyway_schema_history` row, so Flyway refuses to start against it
> ("Found non-empty schema... but no schema history table"). The fix is a
> one-time, metadata-only `flyway baseline -baselineVersion=3` — it does not
> touch table data — but it's a write against a shared cloud resource the
> user hasn't explicitly named, so an auto-mode safety check correctly
> blocked it mid-session and it's tracked as its own task (P6-9) instead of
> done unilaterally.

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
| P0-1 | ☑ | **Toolchain & bytecode decision.** 2026-07-12 resolution: JDK 25 toolchain with `--release 21` bytecode (Quarkus 3.20.1 rejected class-file 69; `java21` Lambda cap). **Superseded 2026-07-13:** the platform was upgraded to **Quarkus 3.37.2** (reads class-file 69) and the reactor targeted **Java 25** end to end (root `release=25`); Testcontainers moved to 2.x (`testcontainers-cockroachdb`), `quarkus-junit5`→`quarkus-junit`. **Superseded again 2026-08-09:** reverted to **Java 21** end to end (root `release=21`) — the CDK stack (built out under the modify-not-execute leg, see P2-7) targets `Runtime.JAVA_21` for all four Lambdas, so the reactor now matches its actual deploy target instead of carrying a mismatch. The maven wrapper was also dropped in favor of a global `mvn`; `verify.sh` is unchanged as the merge gate. | S | — |
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
| P6-1 | ◐ | **Per-agent service accounts + RBAC.** 2026-07-18 local leg: `ops/service-accounts.sql` provisions `dispatcher_agent`/`negotiator_agent`/`coordinator_agent`/`product_api` with the arch §11 grants (idempotent; run after migrations); `ServiceAccountsTest` proves the negotiator principal **cannot** `UPDATE truck` while its pricing surface works. **Unblocked 2026-08-13**: CockroachDB Cloud is restored, so `ccloud`-CLI provisioning of the same grants is now reachable — not yet done; optional/polish (the SQL-script leg already proves the RBAC design correctly; `ccloud` strengthens the Technical Implementation story but isn't gate-critical the way P6-8 is). | M | P5-3, P2-2 |
| P6-2 | ☑ | **MCP Server (read-only) for operators.** 2026-07-18: the operator query set shipped as **operator-runbook.md** (live board, stuck work, refusal/decision triage, memory health, alarm sheet, divergence procedure). 2026-08-12: the self-hosted CockroachDB MCP Server leg shipped — secure-mode local cluster, cert-authenticated `mcp_operator` principal (read-only, 7-table scope), `.mcp.json`/`.mcp.json.example` wiring (stdio, no Docker needed — the binary is `go install`-able and genuinely self-hostable against any cluster, not Cloud-exclusive). **2026-08-14: confirmed running** by the user — the earlier "needs a Claude Code session restart to confirm" caveat is resolved. The Cloud Console-managed variant is tracked separately as **P6-8** (not yet confirmed done — don't assume it from this row). | M | P2-2 |
| P6-3 | ☑ | **Observability.** 2026-07-18: Micrometer/Prometheus at `/q/metrics` — `dakar.actions.executed{action,actor}`, `.refused`, `.lost.race`, `dakar.replay.divergence`, dispute-queue gauges (`dakar.disputes.open`, `.oldest.seconds`, refreshed on `dakar.metrics.refresh-every`) — F2 fairness made measurable. `X-Request-Id` correlation filter: honored inbound, generated otherwise, echoed on every response, on every log line via MDC (no PII). Alarm thresholds documented in the runbook §5 (replay divergence > 0 pages, severity-1). Agent tool-loop metrics arrive with the cloud-leg Bedrock loops. | M | P5-6 |
| P6-4 | ☑ | **Follower reads for projections.** 2026-07-18: staff `GET /v1/deliveries` (fleet dashboard, `dakar.staff.list-limit`) reads `AS OF SYSTEM TIME follower_read_timestamp()` — off the write hot path, may lag seconds (caveat in code, runbook, and test, which polls for visibility instead of pretending freshness). Point reads (`GET /{id}`, post-action reloads) stay fresh deliberately: read-your-writes is part of the product contract. | S | P3-8 |
| P6-5 | ☑ | **Replay audit job.** 2026-07-18: prerequisite first — **creations are now journaled**: new `RequestDelivery` action (+`StateWrite.DeliveryInsert`), `POST /v1/deliveries` runs through the executor with the `api_request` idempotency row as a companion write (raced keys resolve to the winner), and staff `POST /v1/trucks` fires `RegisterTruck`. `JournalPayloadCodec.fromJson` (total inverse, loud on unknown shapes) + `JournalStore` + scheduled `ReplayAudit` (`dakar.audit.every`): replays from the empty board and compares operational state (telemetry excluded per model.md §5.3); divergence → ERROR log + `dakar.replay.divergence` metric, severity-1. Tests: fully-journaled flow audits clean; tampered status and journal-unknown records are reported. | M | P3-4, P3-8 |
| P6-6 | ☑ | **Failure-mode drill script.** 2026-07-18: `ops/drills.sh` runs the named evidence suite — double-claim race, stale-plan re-plan, below-floor refusal, companion rollback, policy-outage degradation, journal tamper, and pricing-policy roll-forward/rollback (`OperationsSurfaceTest`). Output is the production-readiness evidence record. | M | P6-3 |
| P6-7 | ◐ | **S3 artifact store.** 2026-07-18 CDK leg (modify-not-run): `DeliveryArtifacts` bucket — block-all public access, SSE-S3, versioned, SSL-enforced, RETAIN — plus `dakar/staff-access-key` secret and CloudWatch error/throttle alarms; function gets bucket/secret grants and env wiring; stack compiles, never synthed. The pre-signed-upload API endpoint is the cloud leg (needs credentials to sign). | M | P2-7 |
| P6-8 | ☐ | **CockroachDB Cloud Console-managed MCP Server.** Get the MCP config snippet from the Cloud Console for the restored cluster; add it as a second entry in `.mcp.json`/`.mcp.json.example` alongside the self-hosted one (both are legitimate — no reason to choose only one); verify one real tool call against it. Directly closes the residual "does self-hosting count as the named tool" risk flagged when the self-hosted leg shipped. | S | P6-2, cloud restored |
| P6-9 | ◐ | **Cloud cluster Flyway baseline.** 2026-08-14: the user applied the V1–V3 migrations and `service-accounts.sql` **manually** (raw SQL, not through Flyway) — confirmed via a fresh read-only check: same 9 tables as before, still **no `flyway_schema_history` row**. Manually running the DDL doesn't create Flyway's own bookkeeping table; only `flyway baseline` (or a real Flyway-executed migration) does. **Reframed, not resolved**: this only blocks Flyway's own migration check, which only runs in the `%dev` profile (`quarkus.flyway.migrate-at-start` defaults `false` at root, and none of the four Lambda env-var maps in `CDKStack.java` override it) — so it blocks local `mvn quarkus:dev` against the cloud cluster, **not** the deployed AWS Lambdas, which is consistent with the user's separate report that the AWS deployment is live. Still worth the one-time baseline for local-dev convenience; still ⛔ pending explicit go-ahead per the risk-register entry below. | S | user go-ahead |
| P6-10 | ✕ | ~~Per-principal password rotation~~ **Dropped 2026-08-14, by the user's explicit direction**: the same password was used deliberately when manually applying the scripts to the cloud cluster. Not a bug to fix — `service-accounts.sql`'s per-principal-password *comment* is now slightly aspirational versus actual cloud-cluster practice; leaving the comment as-is (it still correctly describes the mechanism's intent for anyone rotating later) rather than editing it to match a decision that may change again. | — | — |
| P6-11 | ☐ | **`ServiceAccountsTest` / Testcontainers regression.** Found in this session's baseline check, unrelated to the MCP work: `cockroachdb/cockroach:latest-v26.2` (a floating tag) now rejects `CREATE`/`ALTER USER ... WITH PASSWORD` under `--insecure` ("setting or updating a password is not supported in insecure mode"), breaking the Testcontainers-backed `ServiceAccountsTest`. Pre-existing, not caused or fixed by the docker-compose secure-mode work (`CockroachTestResource` runs its own ephemeral `--insecure` container, fully decoupled — see the 2026-08-12 note). Fix: either pin the Testcontainers image to a version that still permits this, or secure the Testcontainers path the same way `ops/docker-compose.yml` was secured. | S | — |
| P6-12 | ◐ | **AWS redeploy pointed at a working DB.** 2026-08-14: the user reports the AWS deployment is live. I have no AWS CLI access in this environment to independently confirm (`aws` isn't installed here), so this is recorded on the user's word, not verified — worth a real check (e.g. `curl $API_ENDPOINT/v1/deliveries` from wherever the user has AWS access) before `demo_script.md`'s AWS-live version is recorded on. Supersedes the local-compose-cluster-port-forward plan explored earlier this session — kept as a documented fallback (P2-2/local secure mode). | S | user action |
| P6-13 | ☐ | **Remove the stale `ConsoleReadSurfaceTest.theConsoleIsSameOriginSpa` test.** Fails 404 in every `mvn verify` run this session — it asserts same-origin static serving of the web console that Phase U's supersession note (2026-08-09) confirms no longer exists (`dakar-ui` was removed; `dakar-console` is a separate native app that doesn't get served by Quarkus at all). This is a stale assertion about a removed feature, not a regression to fix. | S | — |
| P6-14 | ☐ | **Diagnose `OperationsSurfaceTest.policyVersionsRollForwardAndBack`.** Failing across multiple sessions (`expected: 50000.00, was: 90.00` — the default `dakar.pricing.floor-base`, suggesting a policy-version rollback isn't taking effect or test data/ordering has drifted). Root cause not yet investigated; flagged repeatedly, not yet diagnosed. | S | — |
| P6-16 | ☑ | **`dakar-console` BoardCanvas trail-leg bug.** 2026-08-14: confirmed and fixed a real bug the user caught — edges were keyed statically per delivery (origin→destination) for its whole life, with no line at all for the garage→pickup repositioning leg (the vehicle glyph moved along an invisible path) and no fading tied to leg transitions, only to delivery completion. Rewrote `_reconcile`/`_applyRoute` → `_applyLeg`: exactly one live leg edge per assigned vehicle at a time (reposition: garage→pickup, transit: pickup→destination), the previous leg cloned into a fading "trail" ghost the instant a new leg begins — across both leg transitions within one delivery and across a truck's consecutive deliveries. Caught and fixed one regression while rewriting: `OFFLOADING` shares the "transit" leg with `EN_ROUTE_LOADED` (no leg-change event), so the original code's unconditional `progress = 1.0` snap-to-destination on arrival needed to stay unconditional too, not gated behind the new leg-change check. **Not yet visually verified in a running Qt session** — QML has no static type checker here; needs an actual `dakar-console` run against live board/fleet data before this is fully trusted. | S | — |
| P6-17 | ☐ | **Resolve the Negotiator autonomy discrepancy.** Found 2026-08-15 while writing `demo_flow.md` — a line-by-line trace of every trigger in the system, read from source, not recalled. `NegotiationAutomation`'s own class-level Javadoc states the `Quote` transition is "NOT fired here — the draft is staff-gated"; its actual code calls `desk.plan(new Quote(...), fresh, Actor.NEGOTIATOR, null)` through `executor.execute(...)` — the exact same commit pattern every other real, database-committing transition in the codebase uses (identical in shape to `DispatchAutomation`'s genuine `ClaimTruck` firing). A `Completed` result here **is** a fired `Quote`: the delivery moves to `QUOTED`, `agreed_price` set, with no staff action anywhere in the path — logged as "negotiation draft created," which is not what it is. **This directly contradicts `agentic.md` §5/§7's autonomy ladder and the root `README.md`'s explicit claim** ("hands that draft to a staff member for a final yes. It never sends a price on its own"). `QuoteAdviceResource` (the `GET .../quote-advice` path) is the actually-correct draft-only reference implementation sitting right next to it in the same package. Needs a team decision, not a silent fix either direction: (a) if the human gate is intended, `NegotiationAutomation` should stop calling `desk.plan(Quote...)` and only persist the draft + trace like `QuoteAdviceResource` does, leaving staff to fire via the existing `/v1/deliveries/{id}/quote` endpoint; (b) if autonomous negotiation is now actually intended (a deliberate move up the autonomy ladder, post-MVP per the original design), the Javadoc, `agentic.md`, and the README all need correcting to match, and this stops being a bug and becomes a documented capability. Either way, leaving the contradiction as-is going into a public submission is the one outcome that's clearly wrong. | S | — |
| P6-15 | ☐ | **Agent Skills citation (optional, cheap).** Not required — vector indexing (real, confirmed on both local and CockroachDB Cloud) + MCP (P6-2, real, confirmed running) already clear the hackathon's "≥2 CockroachDB tools" gate — but strengthens the submission. Cite a real skill from `cockroachlabs/cockroachdb-skills` actually consulted during this work (e.g. a `SERIALIZABLE`/`40001` retry-pattern skill, directly relevant to `ActionPlanExecutor`'s existing retry logic) by name in **P7-2** (the submission writeup — `demo-commentary.md`, this row's original target, was deleted 2026-08-14). | S | — |
| P6-18 | ☑ | **Dispatcher → Negotiator EventBridge gap.** 2026-08-15: found while writing `demo_flow.md`'s trigger-by-trigger trace — `DispatchAutomation`'s autonomous claim path (the `Completed` case in `dispatch()`) never published `delivery.matched`; only `DeliveriesResource.assign()` (a manual staff assign) did. In prod, where EventBridge — not the dev/test `@Scheduled` tick — is the only thing that wakes the Negotiator, an autonomously-dispatched delivery would sit at `MATCHED` forever unless a staff member also happened to assign it by hand. Fixed by injecting `EventPublisher` into `DispatchAutomation` and publishing `delivery.matched` from the `Completed` case, mirroring `DeliveriesResource.assign()` exactly. `DispatchAutomationTest`'s two direct constructor calls updated (`new EventPublisher("")` — an empty bus name is a genuine no-op by `EventPublisher`'s own contract, not a workaround). Verified: `mvn test` 3/3, and a real publish confirmed in the application logs against the live bus. **Not done, deliberately deferred**: a `%dev`/`%test` override to make the *default* bus name resolve to a true no-op was attempted and reverted — Quarkus's String-typed `@ConfigProperty` null-converts an explicit empty property value and then rejects the null even with `defaultValue=""` set. The correct fix (`Optional<String>` on `EventPublisher`'s constructor param) is documented inline in `application.properties` but out of scope for this fix — dev/test keep attempting real (harmless, caught) EventBridge calls as before. | S | — |

---

## Phase U — User interface (operations console & client surface)

> **Superseded 2026-08-09.** The web console this phase describes (`dakar-ui`,
> served same-origin from Quarkus, Pixi board) was replaced by **`dakar-console`**
> — a native PyQt6/QML desktop app for staff — and the planned client web
> surface / Telegram bot were dropped: **shippers call the API directly with
> `curl`**, no client app. `dakar-ui` no longer exists in the tree. Rows below
> are kept as a record of what was built and later replaced, not current state.
> Current staff/client surfaces: [dakar-console/README.md](../../dakar/dakar-console/README.md),
> [dakar-quarkus/README.md](../../dakar/dakar-quarkus/README.md).

| ID | State | Task | Size | Depends |
|---|---|---|---|---|
| U-1 | ☑ | **Design authority.** 2026-07-18: `ui-design.md` shipped — activity-graph board (topological, not geographic), Nominatim geographic model (region-agnostic), full flow/screen/component/state specs, validated stage ramps + reserved health palette, repo-safe vocabulary (§16). High-fidelity previews in `dakar-ui/design/` (`palette.css`, styleguide/console/negotiation/client) — firewall-swept. | M | P6 |
| U-2 | ☑ | **Console read surface.** 2026-07-19: `GET /v1/trucks` (`FleetStore`), `GET /v1/deliveries?view=board` (`DeliveryStore.listBoard`, assigned-truck join, thin list preserved), `GET /v1/activity` (`ActivityStore`, journal+traces merged, actor/delivery filters) — staff-only, follower reads, `ListLimits.clamp` caps. `ConsoleReadSurfaceTest` 5/5 incl. static serving. | M | U-1 |
| U-3 | ☑ | **Serving & toolchain.** 2026-07-19: `dakar-ui/public` → `META-INF/resources` via pom resource (console `/`, client `/client/`, same-origin, no CORS); pixi 7.3.2 + Source Sans 3 woff2 vendored; tsc-only browser ESM (`.js` imports); npm dev-typings only (`pixi.js`, `@types/node`), zero runtime deps. | S | U-1 |
| U-4 | ☑ | **Console shell + board.** 2026-07-19: hash router, sessionStorage access-key flow with probe, typed `Outcome<T>` fetch, 3s snapshot poller (visibility pause, 3→30s backoff, abort), Pixi board (edges/nodes/vehicles/labels, focus & context dimming, eased motion off under reduced motion, zoom/pan/reset), DOM mirror listing every entity. | L | U-2, U-3 |
| U-5 | ☑ | **Operator flows.** 2026-07-19: register truck, assign (candidate filter; dispatcher race rendered as supervision), quote with advice card (floor/ceiling range, evidence count, degraded path), per-stage signal groups, Negotiation queue, Activity screen with filters, refusal-vs-error callouts wired to the real code vocabulary. | M | U-4 |
| U-6 | ☑ | **Client surface + tests.** 2026-07-19: request (Nominatim geocode fields — debounce ≥1.1s, session cache, importance decisiveness, OSM attribution, manual coordinates; per-form idempotency key), track timeline in client vocabulary, accept-quote with expiry honesty; `node:test` suites 8/8 (layout, journey legs, pressure, geocode parsing). | M | U-4 |

---

## Phase 7 — Challenge deliverables & demo

**Deadline: Aug 18, 2026, 5:00pm EDT.** Submission-critical (Stage One pass/fail
gate, or a hard checklist requirement) vs. polish, so time pressure hits the
right rows first:

- **Submission-critical:** P7-4 (root README), P7-5 (repo public + LICENSE),
  P7-2 (submission writeup must not overclaim tools judges are told to
  verify — this is now the only place that claim can be made, see P7-6),
  P7-7 (demo video — explicitly required, must show CockroachDB
  memory in action), P6-8 (Cloud MCP — closes the ≥2-tools gate with zero
  residual risk).
- **Strengthens scoring, not gate-critical:** P7-1, P7-2, P6-10/11/12 (once
  the user unblocks P6-9), P6-1/P6-15 (ccloud, Agent Skills).
- **Quality hygiene, do if time remains:** P6-13, P6-14, P7-3.

| ID | State | Task | Size | Depends |
|---|---|---|---|---|
| P7-1 | ◐ | **Demo scenario script.** 2026-08-14: written as **[demo_script.md](demo_script.md)** — full shot-by-shot script (visual cues, on-screen text, exact commands, spoken lines), timed to the 3:00 hard cap, built around the organizers' explicit video guidance (live demo within 20–30s, name AWS/CockroachDB features on screen, show memory in action visibly rather than narrated). **Rewritten same day, on request, to assume every integration is live**: `dakar-console` moved from bookend shots to the main event (real-world-user view, jumped to via its command-palette lookup — it has no direct fetch-by-ID API call, confirmed against `main.py`); the live demo runs against the real deployed API Gateway URL with EventBridge auto-triggering the Dispatcher/Negotiator Lambdas (no manual `/internal/trigger/*` calls on camera). Centerpiece reworked to match how Lambda actually works: not a staged "kill," but proof that all four functions (`dakar-http`/`dispatcher-agent`/`negotiator-agent`/`coordinator-agent`) share one `function.zip` (`aws lambda get-function --query Configuration.CodeSha256`, same hash) and that every invocation is already a fresh, memoryless execution by design — CockroachDB is the only thing connecting two separately cold-started invocations. **This version's assumptions are real dependencies, not decoration**: it needs P6-9 (cloud baseline) and P6-12 (AWS redeploy pointed at a working DB) actually done before it's true on camera — the file's own "Fallback notes" section says explicitly not to record this version if that isn't confirmed first. **Still remaining regardless of which version records: not yet rehearsed/timed with a stopwatch, not yet recorded.** | M | P5-6, P6-6, P6-9, P6-12 |
| P7-2 | ☐ | **Submission README + diagrams.** Judging-criteria map (arch §12) with links to *evidence* (tests, drill outputs, metrics screenshots), tool usage summary (vector index confirmed real locally and on CockroachDB Cloud — `SHOW INDEXES` verified 2026-08-14; MCP confirmed running; ccloud/Agent Skills only if P6-1/P6-15 land — list only what's actually true). **This is now the sole home for the submission's tool-usage claims** — `demo-commentary.md` (which used to carry this) was deleted 2026-08-14, see P7-6. This is a *different* document from the root README (P7-4): this one is the submission form's writeup, P7-4 is what a judge sees first opening the repo. | M | P7-1 |
| P7-3 | ☐ | **Docs reconciliation pass.** system_architecture.md, dakar_internals.md, and this backlog agree with the shipped code; stale sections fixed or explicitly marked roadmap. This session is itself an example of the failure mode this task exists to catch: the standing-constraint banner and (the now-deleted) demo-commentary.md both went stale for days after real state changed (AWS deployed, cloud cluster disabled then restored) without being corrected same-day — check specifically for *that* pattern, not just obvious errors. | S | P7-2 |
| P7-4 | ☑ | **Root README.md.** 2026-08-14: non-technical overview — the domain (freight matching/pricing/tracking), the three agents described by what they decide rather than how, the memory-centric design explained in plain terms (crash-and-resume, no-double-booking as a proven guarantee not just tested code), a short repo-layout map, links out to `dakar-quarkus/README.md`/`dakar-console/README.md` for setup instead of duplicating them, license. | S | — |
| P7-5 | ☐ | **Repo public + LICENSE confirmation.** LICENSE added (user, this session — confirm MIT/Apache 2.0 and that it's visible in GitHub's "About" section, per the hackathon's exact wording). Repo visibility unconfirmed: an unauthenticated fetch of `github.com/iohaus/dakar` returned 404 earlier this session, consistent with it currently being private. The rules require "public and open source" as a hard submission gate — confirm/flip visibility explicitly, don't assume. | S | — |
| P7-6 | ✕ | ~~`demo-commentary.md` fact-correction pass~~ **Moot 2026-08-14: the user deleted `demo-commentary.md` outright** ("seems it is misguiding you a bit"), rather than have it corrected — it had accumulated overclaims (Cloud MCP, `ccloud`, Agent Skills all marked `[x]` when they weren't real) across multiple sessions faster than they were being caught. The underlying job this task existed for — an accurate tool-usage claim somewhere in the submission — now lives solely in **P7-2**, written once, after the tools it describes are actually in their final state, not accumulated incrementally like the deleted file was. | — | — |
| P7-7 | ☐ | **Demo video.** Required: under 3 minutes, uploaded publicly to YouTube or Vimeo, "must include footage showing the CockroachDB memory layer at work." Not started. Natural to record once P7-1's script is rehearsed. | M | P7-1 |
| P7-8 | ☑ | **Pre-seed demo data with `tooling/dakar-tools/demand_sim`** (separate repo now — moved out of `dakar/tools/` since this session's earlier work; not a `dakar` git submodule, tracked independently). 2026-08-14: reviewed — genuinely good, not a toy: Poisson-arrival clients across 7 hub cities with a weighted lane graph, realistic per-archetype budget-sensitivity accept/reject/cancel decisions, a feedback controller holding a target active-delivery count. **Correction, same day**: the fix belongs in the script, not the tool — Dakar is a company name, the fleet's actual operating geography is US-based, and `demand_sim`'s Chicago/Memphis/Dallas/... hubs are correct as they are (an earlier attempt to reskin them to Senegal cities was reverted in full — that file is gitignored, so `git diff` won't show this history, note it here instead). Fixed `demo_script.md` instead: Scene 2's scripted request is now Chicago→Memphis (demand_sim's own highest-weighted lane, 0.40) with plain, reliably-geocodable city names rather than demand_sim's fictional hub facility labels (which ship with exact lat/lon and so never actually hit live geocoding — using them for the one request that *should* exercise geocoding would risk a resolution failure on camera). Run it against whichever DB the demo targets for a while *before* recording, so `negotiation_episode` has real Chicago–Memphis history and Scene 5's vector search returns something genuine; stop it before hitting record — running it live during the take is still the wrong call, just not for a geography-mismatch reason anymore (it'd still be uncontrolled background noise during a precisely-timed take). **2026-08-15**: added a sequential/paced mode (`--sequential`, `--pause-seconds`, `--max-wait-seconds`) so one delivery can be created, watched settle through `dakar-console` and the AWS agent logs, and only then followed by the next — the "slow controlled manner" the user asked for. First design assumed the pacing had to wait for a human to drive physical signals (departure/arrival/loading/payment/offload), reasoning from `demo_flow.md` §10's finding that those are always operator-triggered; the user corrected this directly — the tool must not wait on a person, only on its own automation (create → autonomous dispatch → autonomous/triggered negotiate → the same accept/reject `QuotePolicy` call the concurrent mode already makes). `run_sequential()`/`_observe_until_settled()` (`main.py`) stop waiting on a delivery once it reaches `AGREED`, its quote is declined by policy, it goes terminal, or `--max-wait-seconds` elapses as a safety valve against a stalled or untriggered agent. README documented; `--help` wiring and `py_compile` verified — not yet run live end-to-end against a running stack. | S | — |
| P7-9 | ☐ | **Resolve the staff-access-key gap before recording.** The user reports "I really don't know about the staff access key, I'm not using it for now." `StaffAccessFilter` hard-requires `dakar.staff.access-key`/`DAKAR_STAFF_ACCESS_KEY` to be set (the bean's constructor throws if blank), and `dakar-console` needs a matching key to authenticate at all — every staff-only call the rewritten `demo_script.md` leans on (Scene 3's console jump, Scene 5's quote-advice card) depends on this being correctly configured on whichever backend (local, cloud-connected local, or the live Lambdas) the demo actually targets. Needs a concrete decision before recording, not discovered live on camera. | S | — |

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

**Verification pass, 2026-08-15.** Every row below re-checked against actual
code/config/live-cluster state (not memory) on request. Verdicts:
✅ confirmed with fresh evidence this pass · ⚠️ partially true, wording or
scope corrected · ⏳ mitigation exists but hasn't been exercised against the
specific environment the risk is about.

| Risk | Watched at | Mitigation | Verified |
|---|---|---|---|
| ~~Java 25 Lambda runtime unavailable~~ | P0-1 | **Resolved 2026-08-09** by reverting the whole reactor to Java 21. | ✅ Root `pom.xml` (`maven.compiler.release=21`), `dakar-quarkus/pom.xml` (`21`), and `dakar/pom.xml` (no override — the module-local `25` pin from an earlier session is gone, inherits `21`) all confirmed today. `CDKStack.java` targets `Runtime.JAVA_21` for all four functions (confirmed earlier this session). No `25` anywhere in the build. |
| CockroachDB cluster version lacks vector indexing | P2-2 | Verify ≥ v25.2 *before* provisioning; it's cheap to check and expensive to re-provision | ✅ Local: `cockroachdb/cockroach:latest-v26.2`, vector index live (this session's docker-compose work). **Cloud, newly confirmed today**: `SELECT version()` → `CockroachDB CCL v26.2.5`; `SHOW INDEXES FROM negotiation_episode` shows a real `negotiation_episode_embedding` index on the `embedding` column. Both environments genuinely clear the bar, not just the local one. |
| `40001` retry storms under agent fan-out | P2-5, P6-3 | Bounded retries + jittered backoff in the executor; retry-rate metric with alarm | ✅ `ActionPlanExecutor.java`: `SERIALIZATION_CONFLICT_STATE = "40001"` caught, `CommitAttempt.LOST_RACE` outcome, `backOff(attempt)` (capped jitter), and a real `dakar.actions.lost.race` Micrometer `Counter` — all re-confirmed by direct grep today, not recalled. |
| LLM structured-output flakiness | P5-2 | Reject-don't-repair tool-call handling; §8.3 validation; violation-rate metric as model-health alarm | ✅ `DispatchAutomation.java`, re-confirmed today: explicit comment "a choice outside the offered set is discarded, recorded, and never executed," trace `outcome: policy-violation`. Matches the reject-not-repair claim exactly. |
| Tool-use loop runaway (cost/latency) | P5-2, P6-3 | ~~Hard bounds from `policy` data (`max_perception_calls`, `max_action_attempts`)~~ | ⚠️ **Corrected today — the original wording overclaimed.** `max_action_attempts` is real (`dakar.actions.max-attempts`, read via `@ConfigProperty` in `ActionPlanExecutor.java`). `max_perception_calls` **does not exist anywhere in the codebase** (grepped, zero matches) — it describes `agentic.md`'s spec-level multi-turn perception-tool-budget loop, which was never built. The actual implementation (confirmed earlier this session) is a single forced-tool-use Bedrock Converse call per tick, not a looping agent — so there's no multi-call perception loop *to* runaway in the first place, which is arguably a stronger mitigation than the one originally claimed, just a different one. Risk register now says what's true. |
| Vector retrieval latency at scale | P4-4 | Latency budget asserted against ≥ 10k-row seed in CI, not discovered in the demo | ⏳ True and re-confirmed for **local** (P4-4 ☑, 10k-episode corpus, warm retrieval < 2s). **Not exercised against CockroachDB Cloud specifically** — a real 10k-row load test against the cloud cluster hasn't been run, and given that cluster already hit its Request Unit limit once, deliberately running one now is a call for the user, not something to do unilaterally. If the demo or submission leans on cloud vector-search performance as a claim, this gap should close first. |
| Scope creep toward Post-MVP items | every phase | This file is the cut line; additions require a backlog PR first | — procedural, not a code-verifiable claim; holds as long as this file is actually consulted. |
| Unauthorized writes against shared cloud resources (CockroachDB Cloud, live AWS deploy) | P6-9, P6-10, P6-12 | Read-only checks first, then ask explicitly before any write against a resource the user didn't specifically name — even a "safe," metadata-only, easily-reversible one | ✅ Followed consistently this session: the Flyway baseline (P6-9) was proposed with full read-only evidence and left for explicit go-ahead rather than run; a Senegal-hub edit to `tools/demand_sim` was reverted in full the moment the user clarified it wasn't asked for, rather than defended or partially kept. Behavioral, not a one-time fix — stays a standing practice, not a closed item. |

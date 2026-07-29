# Dakar — Artifact Reference (`dakar-javadoc.md`)

**Status:** Living document v1.0 — the nontrivial documentation for code artifacts
**Rule (2026-07-12):** code carries **no Javadoc blocks and no multi-line comments**; the only in-code comments are one-liners flagging something important and non-obvious. Everything else about an artifact — contract, invariants, complexity, failure modes — lives here, doc-side, under the vocabulary firewall (internals Law 4). Amend this file in the same PR as the artifact it documents.
**Companions:** [dakar_internals.md](dakar_internals.md) (module laws), [model.md](model.md) (formal meaning; §12 maps spec ↔ code), [system_architecture.md](system_architecture.md).

---

## 1. Module `dakar` (pure core)

### 1.1 `freight` — operational entities (P1, amended P2)

| Artifact | Contract |
|---|---|
| `TruckId`, `DeliveryId`, `ClientId`, `JournalEntryId` | Typed UUID wrappers; `Comparable` by UUID; reject null. Never pass bare UUIDs across the core. |
| `Tons` | Quantity at scale 3 (kilogram resolution), `HALF_EVEN`, strictly positive after normalization — `0.0004` rounds to zero and is rejected. |
| `CargoClass` | Closed enum of cargo families; `name()` is the persisted string. |
| `TruckStatus` / `DeliveryStatus` | The operational status sets; `name()` is exactly the `status` column value (adapter adds nothing). `DeliveryStatus.isTerminal()` ⇔ COMPLETED or CANCELLED. `DISPUTED` and the quote/payment stamps arrive with the full desk (backlog P3-1). |
| `Truck` | Immutable record. Validation: capacity > 0 (via `Tons`), driver hours non-negative, version ≥ 1, optionals never null. `position`/`driverHoursRemaining` are **measurement fields**: owned by telemetry, read by rules, never written by actions (model.md §5.3). `assignedDelivery` is the reservation; `isAssigned()` is its test. |
| `Delivery` | Immutable record. Validation: floor price positive; agreed price, when present, ≥ floor (a below-floor agreement is unrepresentable, independent of any rule check); payment confirmed ⇒ agreed price present; version ≥ 1. |

### 1.2 `geo`, `money` — primitives (P1)

| Artifact | Contract |
|---|---|
| `GeoPoint` | Degrees, validated ranges, −0.0 normalized. `distanceTo` is pure great-circle over `StrictMath` (bit-identical across JVMs; golden-tested). It is an *estimator*: road distances come from `RouteTable`. |
| `Kilometers` | Scale 3, `HALF_EVEN`, non-negative. |
| `Money` | Scale 2, `HALF_EVEN`, non-negative by construction, currency-checked arithmetic (`plus`, `times`, `isAtLeast`); mismatched currencies throw at the operation, never propagate. Raw `BigDecimal` arithmetic outside `money` is forbidden by review convention. |

### 1.3 `desk` — the freight desk (P2-1)

**`DispatchBoard`** — a *value*: the point-in-time picture of fleet and work as
two deterministic sorted maps. Constructor takes defensive unmodifiable
copies, enforces key ↔ record identity and duplicate rejection. The desk never
observes change; it sees one board, emits one plan.

**`Snapshot(board, routes, now)`** — everything one evaluation may consult.
`now` is data (Law 2): comparing against it twice in one evaluation yields the
same instant by construction. *Phasing:* the procedural-policy member joins
this record with policy storage (backlog P3-7); until then the record is
deliberately three fields.

**`RouteKey` / `RouteEstimate` / `RouteTable` / `RouteRequests`** — routing as
data. `RouteKey` orders by (from.lat, from.lon, to.lat, to.lon) so tables and
request sets iterate deterministically. `RouteTable.estimate` returns empty
for unknown legs; the desk converts that to a `MISSING_ROUTE` refusal — no
component ever falls back to calling a routing service mid-evaluation.
`RouteRequests` is the desk's shopping list: the shell resolves it *before*
the evaluation starts.

**`Action`** (sealed) — one record per operational action. P2 ships
`ClaimTruck(truck, delivery)` and `Cancel(delivery)`; the sealed hierarchy
grows with the lifecycle (internals §7.1). Exhaustive switches over `Action`
are the mechanism that forces every consumer to decide behavior for a new
action at compile time.

**`Actor`** — who asked: `DISPATCHER`, `NEGOTIATOR`, `COORDINATOR`, `STAFF`,
`PRODUCT`. Journaled lowercase in the `actor` column.

**`RefusalReason`** — machine-readable causes; callers act on the enum, never
on message text. The set grows with the actions.

**`RuleVerdict`** — `Satisfied` or `Refused(reasons)`; a refusal always carries
*every* applicable reason (not first-failure), because the observability
pipeline counts causes.

**`Evidence`** — sorted fact-name → rendered-value map of everything a rule
consulted that is not recoverable from the records themselves (drive
estimates, computed hour requirements, released resources). Journaled with the
entry so replay/audit can re-check the decision from the journal alone
(model.md R-BINDING). Values use the deterministic `toString` of the typed
values (ISO-8601 durations, `<km>km`, ids).

**`JournalEntry(id, action, evidence, actor, occurredAt)`** — fully formed by
the desk (id drawn from `IdSource`, `occurredAt = snapshot.now()`); the shell
persists it verbatim and adds nothing. Lives in `desk` (not `journal`) because
plans embed entries; the `journal` package (replay) arrives with P3-4 and
depends on `desk`, never the reverse.

**`StateWrite`** (sealed: `TruckWrite`, `DeliveryWrite`) — one record to
persist plus the version the plan was computed against. Constructor enforces
`newValue.version() == expectedVersion + 1` and `expectedVersion ≥ 1`; the
executor applies a write only while the stored version still equals
`expectedVersion`.

**`ActionPlan(action, writes, entry)`** — the desk's only "write": non-empty
write list (copied), and the entry's action must equal the planned action.
Write order is deterministic (delivery first, then truck).

**`ActionOutcome`** (sealed) — `Planned`, `Refused(reasons)`,
`UnknownRecord(id)`. `InvalidRequest` joins with the full desk (P3-1) when
actions gain arguments that can be structurally invalid.

**`FreightDesk`** — stateless, pure, deterministic.
- `requiredRoutes(board)`: for every claimable truck (GARAGE or
  EN_ROUTE_EMPTY, unassigned) × requested delivery: the truck→pickup leg and
  the pickup→drop-off leg. Time O(trucks × deliveries); output deduplicated
  and sorted.
- `plan(action, snapshot, actor, ids)`: re-checks all rules even when the
  caller enumerated earlier (enumeration is advice; planning is the check).
  - `ClaimTruck` rules, in reported order: WRONG_STATUS (truck not claimable
    or delivery not REQUESTED), TRUCK_ALREADY_ASSIGNED, CAPACITY_EXCEEDED,
    MISSING_ROUTE (either leg unresolved — and then driver hours are *not*
    judged: no guessing), INSUFFICIENT_DRIVER_HOURS (required = pickup leg +
    loaded leg vs `driverHoursRemaining`). **MISSING_ROUTE is reported only
    when the pair is otherwise eligible** (no WRONG_STATUS /
    TRUCK_ALREADY_ASSIGNED): `requiredRoutes` deliberately resolves no legs
    for ineligible pairs, and blaming an absent estimate for a claim that was
    already dead would poison the refusal metrics (e.g. every lost
    concurrent race would spuriously count a MISSING_ROUTE). Effects: truck reserved (status
    unchanged), delivery → MATCHED, both versions +1. Evidence: both
    distances, both drive times, required and available hours.
  - `Cancel` rules: delivery must be REQUESTED/MATCHED/QUOTED/AGREED
    (structurally "no cargo on board"), else WRONG_STATUS. Effects: delivery
    → CANCELLED; the (unique, by assignment injectivity) reserving truck, if
    any, is released — LOADING becomes EN_ROUTE_EMPTY (it heads back out),
    other statuses unchanged; reservation cleared; versions +1. Evidence:
    `releasedTruck` when applicable. Finding the holder scans trucks once,
    O(trucks); **two** holders means the board violates assignment
    injectivity and the desk throws `IllegalStateException` — corrupt state
    fails loudly, severity-1.

### 1.3a The full desk (P3-1, 2026-07-12)

**Freight amendments.** `DeliveryStatus` gained `DISPUTED` (maps to client
status `PAYMENT_OVERDUE`). `Delivery` gained `quoteExpiresAt` (stamped by
`Quote` = now + validity) and `paymentDueAt` (stamped by `ArriveDest` = now +
terms), with status coherence enforced in the constructor: quoted-onward
statuses require an agreed price and a quote expiry; `REQUESTED`/`MATCHED`
forbid them; payment-phase statuses require a payment due time; `CANCELLED`
is exempt (cancellation can happen at any stage).

**Action inventory (22).** Operational: `ClaimTruck`, `Quote(delivery, offer,
validity)`, `AcceptQuote`, `QuoteExpired` ⏲, `BeginReposition`,
`ArriveOrigin`, `Load(truck, loadedTons)`, `ArriveDest(truck, paymentTerms)`,
`ConfirmPayment`, `Offload(truck, nextTruckStatus ∈ {GARAGE,
EN_ROUTE_EMPTY})`, `PaymentOverdue` ⏲, `RecordDisruption(delivery, note)`,
`Cancel`. Staff-only (desk-enforced `STAFF_AUTHORIZATION_REQUIRED` for any
other actor): `ReleaseTruck(truck, next)`, `SettleDispute`, `WriteOff`,
`TransferCargo(from, to)`, `AbortLoaded(truck, next ∈ {GARAGE,
MAINTENANCE})`, and the fleet administration set `RegisterTruck`,
`Decommission`, `StartMaintenance`, `CompleteMaintenance`.

Rule highlights (each refusal cause tested):
- **Quote**: delivery MATCHED; offer ≥ floor (`OFFER_BELOW_FLOOR`). Stamps
  price + expiry.
- **AcceptQuote / QuoteExpired**: QUOTED, then strictly before / at-or-after
  the stamped expiry (`QUOTE_EXPIRED` / `QUOTE_NOT_EXPIRED`). Expiry cancels
  and releases the reserved truck (LOADING → EN_ROUTE_EMPTY, else in place).
- **BeginReposition**: GARAGE + reserved (`TRUCK_NOT_ASSIGNED`); refuses
  `ALREADY_AT_PICKUP` when the telemetry position equals the pickup.
- **ArriveOrigin/Load/ArriveDest**: signal-driven leg; statuses must pair
  (truck ↔ carried delivery); `Load` re-checks capacity against the actually
  loaded tonnage. The arrival signal discharges the position predicate; the
  position is journaled as an Evidence witness.
- **ConfirmPayment**: AWAITING_PAYMENT or DISPUTED; payment is monotone
  (`PAYMENT_ALREADY_CONFIRMED` guards double signals).
- **Offload / SettleDispute**: the only two paths to COMPLETED, both requiring
  confirmed payment (`PAYMENT_NOT_CONFIRMED`) — completion without payment is
  unrepresentable end to end.
- **PaymentOverdue** ⏲: AWAITING_PAYMENT, unpaid, at-or-after `paymentDueAt`
  (`PAYMENT_NOT_YET_DUE`) → DISPUTED.
- **ReleaseTruck**: frees an OFFLOADING truck from a DISPUTED delivery
  (cargo to storage); bumps the delivery version — custody changed, stale
  plans must die.
- **SettleDispute/WriteOff**: require the truck already detached
  (`TRUCK_STILL_ATTACHED`).
- **TransferCargo**: rescue for a broken loaded carrier — moves the
  assignment atomically (from → MAINTENANCE unassigned, to → EN_ROUTE_LOADED
  assigned) in one plan, re-checking capacity and driver hours over the
  rescue legs (to→cargo, cargo→drop-off); route gating mirrors ClaimTruck.
- **RecordDisruption**: journal-only in effect, but bumps the delivery
  version deliberately so concurrent stale plans are invalidated by the
  version guard.
- **Administration**: `RegisterTruck` → `StateWrite.TruckInsert` (version 1;
  duplicate id refuses `ALREADY_REGISTERED`, and a raced duplicate insert is
  turned into that refusal by the executor's 23505→re-plan path);
  `Decommission` → `StateWrite.TruckDelete` (GARAGE/MAINTENANCE, unassigned).

**StateWrite** grew two variants: `TruckInsert(newValue)` (version must be 1)
and `TruckDelete(id, expectedVersion)`. The executor maps them to
INSERT/DELETE, still version-guarded where applicable, and now treats
SQLSTATE 23505 like 40001: lost race → fresh snapshot → re-plan.

**Policy phasing.** Quote validity and payment terms are action arguments
supplied by the shell (config) until `PolicySet` lands (P3-7); they are
journaled inside the action payload, so replay stays closed.

### 1.3b Policies, enumeration, audit, replay (P3-3/4/6/7, 2026-07-12)

**`policy.PricingPolicy` / `policy.PolicySet`** — procedural memory as typed
data. Floor = `floorBase + floorPerTonKm × tonnage × distance`; ceiling =
`floor × ceilingMultiplier` (validated ≥ 1). `PolicySet` carries the pricing
policy plus the quote-validity and payment-terms windows (validated
positive). `Snapshot` is now `(board, policies, routes, now)`. `Quote` and
`ArriveDest` take no window arguments: the desk reads the snapshot's policies
and journals the applied window as evidence (`appliedQuoteValidity`,
`appliedPaymentTerms`) so replay never consults the policy table.

**Enumeration (`FreightDesk`)** — `claimCandidates(snapshot)`: only enabled
truck–delivery pairs (status, assignment, capacity, resolvable legs, hours),
ordered by (deliveryId, truckId), annotated with repositioning distance,
total drive time, and hours slack. `offerBand(deliveryId, snapshot)`: present
only for MATCHED deliveries; `OfferBand.admits` is the negotiator's bound —
out-of-band proposals are refused, never clamped.

**`check.BoardAudit`** — the executable consistency audit. Rules:
`ASSIGNMENT_INJECTIVE` (≤ 1 holder per delivery), `ASSIGNMENT_TARGET` (a held
delivery exists and is active), `ASSIGNMENT_COVERAGE` (MATCHED…
AWAITING_PAYMENT must have a holder), `PHASE_COHERENCE` (holder truck status
must pair with the delivery status), `CAPACITY` (holder can carry its load),
`PAYMENT_BEFORE_COMPLETION`. Any violation on a live board is severity-1.

**`journal.Journal.replay`** — total fold over journal entries; throws
`ReplayException` when an entry was not allowed at its point (statuses,
assignment, payment monotonicity, stamped time predicates against
`occurredAt`, staff authority). Measurement-dependent rules are treated as
witnessed by the entry's evidence rather than recomputed — the journal does
not carry telemetry (model.md §5.3). Property-pinned: replay equals the
desk's fold on seeded legal runs; delivery-targeting mutations are always
detectable. Swaps of *commuting* entries (a disruption note beside a payment)
replay to the same board by design — an equivalent history, not corruption.

**`testing.LegalRuns` (test util)** — seeded generator of legal action
sequences: random fleet/work, up to 40 steps drawn from all 22 actions with
random actors (staff where required), desk-planned and folded; produces the
entry list plus every intermediate board. Shared by the consistency and
replay property suites; determinism comes from the seed.

**Shell (P3-7/P3-8)** — `V2__pricing_and_window_policies.sql` creates the
versioned `policy` table (one active version per key via partial unique
index) and seeds v1 from the former dev config. `PolicyStore` parses active
rows into a `PolicySet` (typed `PolicyConfigurationException` on missing or
unreadable definitions) and is read inside the snapshot's read-only
transaction — board and policies are one consistent view. The product API
prices floors through the policy (`DevFloorPriceModel` deleted) and gained:
staff `POST /{id}/quote` (offer in operation currency), client
`POST /v1/quotes/{id}/accept`, and staff `POST /{id}/signals`
(DEPARTED / ARRIVED_AT_PICKUP / LOADED / ARRIVED_AT_DESTINATION /
PAYMENT_CONFIRMED / OFFLOADED — the stubbed signal surface until the Phase-5
coordinator). `ExecutionResponses` centralizes the ExecutionResult → HTTP
mapping.

### 1.3c Episodic memory (P4, 2026-07-13)

**`freight.LaneKey`** — origin→destination bucketed to half-degree cells
(`"29,-35>29,-34"`); nearby routes share a lane, giving retrieval and lane
aggregates a stable grouping key. Deterministic (`StrictMath.floor`).

**`agent.NegotiationEpisodeDraft`** — what the desk-side caller hands to the
shell for recording: delivery/client ids, lane, cargo, tonnage, opening
offer, transcript. The transcript carries no client identifiers or personal
data — it is embedded and retrieved as advisory context.

**`agent.EpisodeOutcome`** — OPEN → WON (quote accepted) / LOST; `dbValue()`
is the persisted lowercase string.

**`agent.SimilarEpisode` / `agent.NegotiationContext`** — the retrieval
shape the Phase-5 Negotiator consumes: k most-similar past episodes
(with cosine similarity), plus distilled `ClientFacts` (win rate, average
final price, episode count) and `LaneFacts` (median price per ton) when the
aggregates exist.

**Shell `embedding` package** — `TranscriptEmbedder` (contract: 1024
dimensions, pinned to the `VECTOR(1024)` column), `LocalHashEmbedder`
(deterministic hashed bag-of-words, L2-normalized — the development/test
provider; shared wording ⇒ high cosine similarity), and
`BedrockTranscriptEmbedder` (Titan Text Embeddings V2 via `InvokeModel`;
typed `EmbeddingException` on transport/shape failures; unit-tested against
a fake client — the real-model contract test is a cloud leg). Provider
selected by `dakar.embedding.provider` (`local` default; the Bedrock client
is only constructed when selected). Embeddings are computed **before** the
action transaction: an embedding outage fails the request (503
`EMBEDDING_UNAVAILABLE`), never stores an episode the ledger cannot recall.

**Shell `EpisodeStore`** — companion writes for the executor
(`record`/`markWon`/`markLost`) so the episode row commits **in the same
transaction** as the action's journal entry (quote inserts `open`; accept
marks `won` and copies the agreed price; lost-marking is wired when
cancellation/expiry surfaces arrive with P5-5). `retrieveContext` runs the
kNN (`embedding <=> query`, lane+cargo prefilter, won/lost only) and joins
the aggregates; vectors are unit-normalized so cosine and L2 rankings agree.
`distill()` is an idempotent full re-aggregation into `client_profile` and
`lane_stat` (median via `percentile_cont`), scheduled by
`SemanticDistillation` every `dakar.distillation.every`.

**Executor `CompanionWrite`** — the general mechanism: extra rows that must
exist iff the action commits; a companion failure rolls back the whole
action (property held by test).

**Schema V3** — `negotiation_episode` (`VECTOR(1024)` + vector index,
lane/cargo index), `client_profile`, `lane_stat`. Requires CockroachDB ≥
v25.2 (`feature.vector_index.enabled`); the local stack runs v25.2.20 since
this phase.

### 1.3d Agents (P5, 2026-07-14)

**Domain ports (`agent/`)** — `DispatchPolicy.choose(candidates, DispatchContext)`
and `NegotiationPolicy.proposeOffer(band, NegotiationContext)`. Ports are
inputs, not authorities: callers discard non-candidate choices and reject
(never clamp) out-of-band offers; the desk re-checks every rule at plan time
regardless. `DispatchContext(delivery, laneFacts)`.

**Shell `agents` package** — hosted in `dakar-quarkus` behind the Quarkus
scheduler while cloud triggers are blocked (the P5-1 decision; the
EventBridge-driven Lambda module is the cloud leg).
- `LocalDispatchPolicy` / `LocalNegotiationPolicy` — deterministic baselines
  (least repositioning; band midpoint). Default provider.
- `BedrockConverse` — one **forced tool call** per decision over the Converse
  API; lazy client (missing credentials degrade at call time as
  `PolicyUnavailableException`, never at boot). `BedrockDispatchPolicy`
  (`choose_truck{truckId, rationale}` over the offered candidates — an
  un-offered truck is discarded with a warning) and
  `BedrockNegotiationPolicy` (`propose_offer{amount, rationale}`; unreadable
  amounts → empty). Fake-client unit tests pin the request contract and the
  failure taxonomy; live-model contract tests are the cloud leg.
- `DispatchAutomation.tick()` — snapshot → candidates per requested delivery
  → policy → validate-against-offered-set → execute as `DISPATCHER` with the
  `decision_trace` row **co-committed** with the claim; abstentions,
  violations, and refusals leave standalone traces; `PolicyUnavailable`
  skips the tick (degraded, never unsafe).
- `LifecycleCoordinator.tick()` — the timekeeper: `QuoteExpired` (with the
  episode marked lost in the same transaction) and `PaymentOverdue` fire on
  time alone. This discharges fairness assumption F1.
- `QuoteAdviceResource` (`GET /v1/deliveries/{id}/quote-advice`, staff-only) —
  human-in-the-loop negotiation: retrieval context + offer band + drafted
  offer with floor/ceiling for staff judgment; drafts are traced
  (`drafted`/`abstained`/`policy-violation`); out-of-band drafts return 502
  `ADVICE_REJECTED`. Staff still quote through the existing endpoint.

**`DecisionTraceStore`** — the `decision_trace` writer: companion form
(trace commits with the action it explains, `journal_entry_id` set) and
standalone form (no-action episodes). Traces carry candidates/choice/outcome
as JSONB.

### 1.3d Hardening (P6, 2026-07-18)

**`desk.RequestDelivery` / `StateWrite.DeliveryInsert`** — delivery creation
is now a journaled action carrying the *full initial record* (validated:
REQUESTED, version 1, no negotiation/payment state), so replay is closed
over the journal from the empty board. The product API executes it as
`Actor.PRODUCT` with the `api_request` idempotency row as a companion write;
a raced duplicate key exhausts as contention and resolves to the winner's
delivery. Staff register trucks through `POST /v1/trucks` (`RegisterTruck`)
— production fleets never enter by SQL.

**`persistence.JournalPayloadCodec.fromJson`** — the total inverse of the
payload writer: every action type reconstructed, unknown types and missing
fields fail loudly. **`JournalStore.readAll`** orders by
`(recorded_at, entry_id)` — local commit-order approximation, documented.

**`persistence.ReplayAudit`** — scheduled (`dakar.audit.every`): replays the
whole journal from empty and diffs against the live board. Deliveries
compare fully; trucks compare on operational fields only (position and
driver hours are telemetry, per the model's ownership split). Findings →
ERROR log + `dakar.replay.divergence` counter; **any finding is
severity-1**. Dev-only `Db.seedTruck` bypasses legitimately trip it — the
audit tests use only journaled surfaces.

**Observability** — Micrometer/Prometheus (`/q/metrics`):
`dakar.actions.executed{action,actor}` / `.refused` / `.lost.race`,
`dakar.replay.divergence`, dispute gauges (`dakar.disputes.open`,
`.oldest.seconds`, refreshed by `DisputeMetrics` on
`dakar.metrics.refresh-every`). `RequestCorrelationFilter` puts
`X-Request-Id` on every response and `{requestId}` on every log line (MDC);
inbound ids ≤ 64 chars are honored. Alarm sheet: runbook §5.

**Follower reads** — staff `GET /v1/deliveries` (limit
`dakar.staff.list-limit`) reads `AS OF SYSTEM TIME
follower_read_timestamp()`: dashboard semantics, may lag seconds. Point
reads stay fresh — read-your-writes is part of the product contract.

**Access control (local leg)** — `ops/service-accounts.sql`: four
least-privilege principals mirroring the cloud RBAC; the negotiator
deliberately holds no `UPDATE` on `truck` (proved by `ServiceAccountsTest`).

**Drills** — `ops/drills.sh` is the runnable failure-mode evidence:
double-claim race, stale plan, below-floor refusal, companion rollback,
policy outage, journal tamper, policy roll-forward/rollback.

**CDK (modify-not-run)** — stack gains the `DeliveryArtifacts` bucket
(block-public, SSE-S3, versioned, SSL-enforced, RETAIN), the
`dakar/staff-access-key` secret, error/throttle alarms, and least-privilege
grants + env wiring for the function. Compiled, never synthed.

### 1.3e Console & client surface (Phase U, 2026-07-19)

**Read surface (dakar-quarkus)** — three staff-only follower reads (all
clamped by `dakar.staff.list-limit` via `ListLimits.clamp`, `?limit=`
never trusted):
- `GET /v1/trucks` → `TruckView[]` (`FleetStore.listRecent`; position,
  driver hours, assignment, `updated_at`).
- `GET /v1/deliveries?view=board` → `BoardDeliveryView[]`
  (`DeliveryStore.listBoard`; LEFT JOIN resolves the assigned truck in the
  same follower-read pass; alias-qualified columns —
  `FreightRows.DELIVERY_COLUMNS_ALIASED`). The un-parameterized list keeps
  its thin shape for existing callers. Tons render
  `stripTrailingZeros` (DECIMAL scale is storage detail, not contract).
- `GET /v1/activity` → `ActivityView[]` (`ActivityStore`): journal actions
  and decision traces merged newest-first; filters `actor` (validated
  against the actor enum, 400 `INVALID_ACTOR`) and `deliveryId` (traces
  join through `journal_entry_id`). Trace JSON rides in `detail`.
Proven by `ConsoleReadSurfaceTest` (401 gates, row shapes, filters, caps,
static serving).

**Serving** — `dakar-quarkus/pom.xml` maps `../dakar-ui/public` into
`META-INF/resources`: console at `/`, client surface at `/client/`, all
same-origin (no CORS anywhere). The jar ships whatever `public/` holds —
run `npm run build` in `dakar-ui` before packaging.

**dakar-ui toolchain** — no bundler, no runtime npm packages: `tsc` emits
browser-native ES modules (`.js`-suffixed relative imports) into
`public/js`; npm is dev-only (`typescript`, `pixi.js` for typings,
`@types/node` for tests). PixiJS 7.3.2 and the Source Sans 3 variable
woff2 are vendored under `public/vendor/`. `npm test` compiles with
`tsconfig.tests.json` and runs `node --test` over `dist/tests` —
pure-logic suites only (layout clustering/stability, journey legs,
pressure cues, geocode parsing/decisiveness).

**Console app (src/)** — `main.ts` (hash router, 3s snapshot poller via
`poll.ts` with visibility pause + 3→30s backoff, keyboard shortcuts,
selection in the hash); `api.ts` (typed fetch → discriminated
`Outcome<T>`: ok / apiError with code+requestId / network); `access.ts`
(staff key in `sessionStorage` only); board: `layout.ts` (facility
derivation by merge radius + stable radial-by-bearing arrangement),
`progress.ts` (signal sequence → journey leg fractions; route pressure),
`render.ts` (Pixi layers: edges/nodes/vehicles/labels, focus-and-context
dimming, eased vehicle motion disabled under reduced motion),
`mirror.ts` (visually-hidden focusable listing of every board entity —
the accessible truth); `ui/` (panel with per-stage action groups,
refusal-vs-error callouts per the no-optimistic-writes stance, feed with
expandable traces, forms, six screens). Client app: `client/main.ts`
(request/track/accept, per-form idempotency key reused across retries)
and `client/geocode.ts` (Nominatim jsonv2, ≥1.1s spacing, session cache,
importance-based decisiveness, OSM attribution, endpoint override via
`<meta name="dakar-geocoder">`).

**Known scope cuts** (deliberate, tracked in backlog): no cancel action
(no HTTP endpoint exists), facility labels are coordinates (reverse
geocoding later), advice range bar reads exact values as text beside it.

### 1.4 `port` — `IdSource`

`UUID next()`. The only sanctioned impurity reachable from the core:
production wires `UUID::randomUUID` (shell), tests wire seeded sequences so
plans (including journal entry ids) are reproducible values.

### 1.5 `projection` — `ClientStatus`, `StatusProjection`

Total, pure mapping from internal statuses to the only vocabulary a client
ever sees. The switch is exhaustive over `DeliveryStatus` *without* a default
branch — adding a status refuses to compile until its client wording is
decided. Current mapping: REQUESTED→RECEIVED, MATCHED→PREPARING_QUOTE,
QUOTED→QUOTE_READY, AGREED→DRIVER_EN_ROUTE (a truck is committed from
MATCHED on — assignment injectivity), IN_TRANSIT→IN_TRANSIT,
AWAITING_PAYMENT→AWAITING_PAYMENT, COMPLETED→DELIVERED,
CANCELLED→CANCELLED. The `assignedTruck` parameter is part of the contract
for wording that depends on truck state (used from P3-5).

---

## 2. Module `dakar-quarkus` (shell)

### 2.1 `DakarConfiguration`

CDI producers for the pure core: `FreightDesk`, `IdSource`
(`UUID::randomUUID`), `Clock` (system UTC). The core carries no container
annotations; all wiring happens here.

### 2.2 `persistence`

**`FreightRows`** — the row ↔ record mapping and the canonical column lists.
Positions are selected as lat/lon (`ST_Y`/`ST_X` over the stored
`GEOGRAPHY(POINT)`); inserts write `ST_MakePoint(lon, lat)::GEOGRAPHY`
(**longitude first**). Driver hours persist as fractional hours (DECIMAL) and
convert to second-precision `Duration` (`HALF_EVEN`). Money columns carry
amounts only; the operation currency comes from `dakar.pricing.currency`
(single-currency operation; a currency column is a deliberate future change).
The delivery mapper ignores `quote_expires_at`/`payment_due_at` until the
record gains those fields (P3-1).

**`SnapshotLoader`** — assembles one `Snapshot`: both freight tables read in a
single read-only transaction (one point in time — never a torn view across
tables), then `requiredRoutes` resolved via the estimator, then `now` stamped
once from the injected clock. Bounded: each table is read with `LIMIT max+1`
and more than `dakar.board.max-records` rows raises
`BoardSizeExceededException` — a partial board must never silently drive
decisions. Failures wrap in `DataAccessException` (retry-later semantics).

**`ActionPlanExecutor`** — the execution contract (internals §7.5) made real:
one SERIALIZABLE transaction per plan; every `StateWrite` applied as a
version-guarded `UPDATE` (0 rows = lost race), the journal entry inserted in
the same transaction; SQLSTATE `40001` = lost race. On a lost race: fresh
snapshot, re-enter the planner (which must be a pure function of the
snapshot), bounded by `dakar.actions.max-attempts` with capped full-jitter
backoff; exhaustion returns `ContentionExhausted`, never an exception. Truck
updates deliberately never write `position`/`driver_hours_rem` (measurement
columns). Journal columns: `actor` lowercase, `occurred_at` from the entry
(replay determinism), `truck_id` derived from the plan's truck write when
present.

**`ExecutionResult`** (sealed) — `Completed(entryId)`, `Refused(reasons)`,
`UnknownRecord(id)`, `ContentionExhausted(attempts)`. The API maps these to
200/409/404/503 respectively.

**`JournalPayloadCodec`** — versioned payload
`{v:1, action:{type,…ids}, evidence:{facts}}`. Shapes are append-only:
released shapes never change; a changed action gets a new type name (replay
compatibility, internals §10). Serialization failure is a programming error
(`IllegalStateException`), not a data-access condition.

**`DeliveryStore`** — product-API persistence. `createRequested(key,
delivery)`: `api_request` row + `delivery` row in one transaction; SQLSTATE
`23505` on the request key means an earlier (or concurrently racing) request
already created the delivery — returns `AlreadyExists(existingId)` after
rollback. Reads: by id, and the reserving truck via
`truck.assigned_delivery = ?` (unique partial index guarantees ≤ 1 row).

**`DataAccessException` / `BoardSizeExceededException`** — retry-later vs
deliberate-config-change. Neither is ever swallowed; API maps the former to
503 and treats the latter as fatal.

### 2.3 `routing.HaversineRouteEstimator`

Development estimator: straight-line distance at
`dakar.routing.average-speed-kmh` (validated positive). Deterministic; time
O(legs). Replaced by a road-network adapter without touching the core (routes
are snapshot data).

### 2.4 `pricing.DevFloorPriceModel`

Internal floor = `floor-base + floor-per-ton-km × tonnage × straight-line
km`, all `Money` arithmetic (no raw `BigDecimal` math). Base validated
positive so every created delivery satisfies the record's floor > 0 rule. The
floor is internal and never appears in any client response. Superseded by the
versioned pricing policy (P3-7).

### 2.5 `api`

**`DeliveriesResource`** (`/v1/deliveries`) — the product surface; product
vocabulary only (client statuses, delivery ids). Endpoints:
- `POST /v1/deliveries` — requires an `Idempotency-Key` header (≤ 128 chars);
  field-by-field validation with typed 400s; floor priced internally;
  duplicate/raced keys return 200 with the original delivery, first
  acceptance returns 201 + Location.
- `GET /v1/deliveries/{id}` — 404 when unknown; otherwise
  `{deliveryId, status}` via `StatusProjection`.
- `POST /v1/deliveries/{id}/assign` — staff-only; executes `ClaimTruck` as
  `Actor.STAFF` through the executor. Mapping: Completed→200 (fresh view),
  Refused→409 with operational reason codes (staff surface), UnknownRecord→
  404, ContentionExhausted→503 + Retry-After.

**`StaffOnly` / `StaffAccessFilter`** — name-bound request filter comparing
`X-Staff-Access-Key` against `dakar.staff.access-key` in constant time
(`MessageDigest.isEqual`); the key is required non-blank at startup (fail
fast), has no production default, and dev/test values are local-only.

**`ApiProblem` / `ApiError` / mappers** — one typed client-facing failure
shape `{code, message}`. `ApiProblemMapper` renders thrown problems;
`DataAccessExceptionMapper` maps persistence unavailability to 503;
`UnexpectedFailureMapper` logs and sanitizes anything else to a 500 with no
internals in the body.

---

## 3. Test fixtures worth knowing

| Fixture | Purpose |
|---|---|
| `SeededIdSource` (dakar, test) | Deterministic id sequence (UUID low bits 1,2,3…) so planned entries are assertable values. |
| `CockroachTestResource` (dakar-quarkus, test) | One CockroachDB Testcontainer (`cockroachdb/cockroach:v23.2.31` — same tag as ops/docker-compose.yml) for the whole suite; injects datasource config; Flyway migrates on app start. Two build pins make it work on modern engines: Testcontainers overridden to 1.21.x (the Quarkus-BOM 1.20.6 handshakes Docker API 1.32, refused by engines ≥ 25) and surefire sets system property `api.version=1.44` for docker-java. Tests bind a random HTTP port (`quarkus.http.test-port=0`) so local services (e.g. the dev DB console on 8090) never collide. |
| Table truncation helpers | Each persistence/API test truncates the five tables before running; tests share the container but never share state. |

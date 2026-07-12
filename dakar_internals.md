# Dakar Internals — The Core Library (`dakar`)

**Status:** Normative v0.3 — this document *defines* the module; code conforms to it, not vice versa. *(v0.3: the **vocabulary firewall** (Law 4) — the CPN specification language of [model.md](model.md) never appears in code, schema, or code comments; all prescriptions below use the freight-operations vocabulary. The spec↔code mapping lives in model.md §12. v0.2 alignment history retained where noted.)*
**Audience:** Dakar engineering
**Companion:** [system_architecture.md](system_architecture.md) is the system-level authority; [model.md](model.md) is the formal authority; this document is the module-level authority for the domain library. Where documents disagree, fix the disagreement — neither silently wins.

The key words **MUST**, **MUST NOT**, **SHOULD**, and **MAY** are used as in RFC 2119.

---

## 1. Purpose and authority

`dakar` is the Java module containing **all core domain logic** for Dakar: the
freight entities (trucks, deliveries), the dispatch board, business-rule
evaluation, action planning, the operations journal and its replay, consistency
audits, and product-status projections. It is the executable form of the
formal net defined in model.md — expressed entirely in operational vocabulary
(Law 4).

It is the *functional core* of the hexagonal architecture (system_architecture.md §2).
Everything else in the system — the Quarkus product API, the agent Lambdas, the
CockroachDB adapter, the Bedrock adapter — is an *imperative shell* around this
module.

Two sentences govern every decision in this document:

> **The domain library computes decisions as data; the shell executes them.**
> **Given the same inputs, the domain library always produces the same outputs.**

---

## 2. Module identity

| Property | Value |
|---|---|
| Maven coordinates | `com.iohaus:dakar:1.0.0` |
| Maven parent | `dakar-parent` (inherits toolchain; imports `dakar-bom`) |
| Reactor | added to `<modules>` in the root `dakar-build-parent` aggregator |
| Java | JDK 25 toolchain; bytecode `--release 21` (Quarkus 3.20 rejects class-file 69, and `java21` is the newest managed Lambda runtime — backlog P0-1, 2026-07-12). The language surface used (records, sealed interfaces, switch patterns) is fully within 21; no unnamed variables (`_`). Revisit when the platform reads class-file 69. |
| Packaging | plain `jar` — **not** a Quarkus module, no `quarkus-maven-plugin` |
| Root package | `com.iohaus.dakar` |

**Consumers:** the `dakar-quarkus` Quarkus module (product API) and the agent Lambda
modules depend on `dakar`. The `cdk` module MUST NOT depend on it.

### 2.1 Dependency policy

The module's `compile` scope is **empty of frameworks by construction**:

- **Allowed (compile):** the JDK. Optionally `org.jspecify` annotations
  (annotation-only, no runtime behavior).
- **Forbidden (compile):** Quarkus, `jakarta.*` / `javax.*` (including
  `jakarta.inject`), AWS SDKs, JDBC, Jackson or any serialization library,
  HTTP clients, logging frameworks (JBoss LogManager, SLF4J), Guava or any
  general-purpose utility library.
- **Allowed (test):** JUnit 5, jqwik (property-based testing), ArchUnit,
  AssertJ.

No CDI or DI-framework annotations appear in this module. "Dependency
injection" here means **constructor injection of interfaces**, wired by the
host application (Quarkus CDI producers in `dakar-quarkus`, plain construction in
Lambdas). The domain defines the ports; it never sees the container.

Enforcement is mechanical, not aspirational — see §12.

---

## 3. The four design laws

### Law 1 — Purity (no I/O, no infrastructure)

Domain code MUST NOT: open sockets or files, call HTTP endpoints, touch a
database, log, read environment variables or system properties, spawn threads,
or block. There is no SQL, no JSON, no prompt text, and no retry loop anywhere
in this module. A class in `com.iohaus.dakar..` compiles and runs with
nothing but the JDK on the classpath.

### Law 2 — Determinism

Given the same inputs, every public function returns the same output, on every
JVM, every time. Consequences:

| Source of nondeterminism | Rule |
|---|---|
| Wall-clock time | `Instant.now()`, `System.currentTimeMillis()`, `System.nanoTime()` MUST NOT be called. Time enters as data: every evaluation receives `now` inside the `Snapshot` (§6). |
| Randomness | `java.util.Random`, `SecureRandom`, `Math.random()` MUST NOT be used. The domain has no legitimate need for randomness. |
| ID generation | `UUID.randomUUID()` MUST NOT be called. New IDs are supplied by the injected `IdSource` port (§8), making the impurity explicit and replayable. |
| Iteration order | All collections exposed or iterated by domain logic are deterministically ordered (`SortedMap`/`SortedSet` keyed on comparable IDs, or `List` with documented order). `HashMap`/`HashSet` iteration MUST NOT influence any output, including candidate enumeration order. |
| Floating point | Money is `BigDecimal` with fixed scale and rounding (§9). `double` is permitted only in `geo` distance math, which is pure and covered by golden tests. |
| Locale / charset / timezone | Nothing locale-sensitive. All timestamps are `Instant` (UTC). String comparisons are `Comparator.naturalOrder()` on ASCII identifiers. |

Determinism is what makes the rest of the system cheap: serialization-failure
retries are trivially safe (re-plan from a fresh snapshot is a pure call),
journal replay reproduces the board exactly (§10), and the offline model and
the runtime provably evaluate the same rules.

### Law 3 — Ports own the boundary

Every interaction with the outside world — routing estimates, LLM policy
choices, ID generation — is expressed as a **domain-owned interface** (§8),
implemented in the shell and passed in through constructors. The domain never
names an implementation technology. Corollary: the domain **re-validates
everything a port returns** (§8.3); ports are inputs, not authorities.

### Law 4 — The vocabulary firewall

The specification documents (this directory, `dakar.doc`) are accessible only
to the highest-privilege contributors. The `dakar` repository is not. Two
consequences, both mechanical:

1. **Spec vocabulary never appears in the repository.** The modeling
   formalism's words — *token, place, marking, net, transition, firing, guard,
   binding, arc, color, Petri, enabled, saga* — MUST NOT appear in package
   names, class/method/field names, schema identifiers, string constants,
   Javadoc, or code comments. Code speaks freight operations: trucks,
   deliveries, statuses, the dispatch board, actions, the journal, rules,
   refusals. (ArchUnit-enforced for symbols — §12; review-enforced for
   comments and strings.)
2. **Code never references `dakar.doc`.** No Javadoc citation, no link, no
   path. Traceability is one-way: the private docs cite code symbols
   (model.md §12 is the spec↔code map); the code is self-contained and fully
   understandable in operational terms without any privileged document.

The naming standard this yields is exactly CLAUDE.md's domain-driven-clarity
rule applied ruthlessly: names describe the business thing, never the
formalism or pattern behind it.

**The vocabulary map** (normative; the prose of *this* document may use either
column — it is a privileged document — but code uses only the right column):

| Specification (model.md) | Code / schema |
|---|---|
| truck token | `Truck` |
| delivery token | `Delivery` |
| truck place | `TruckStatus` (column `status`) |
| delivery place | `DeliveryStatus` (column `status`) |
| marking | `DispatchBoard` |
| the net / `DakarNet` | package `desk`, class `FreightDesk` |
| transition | `Action` (sealed interface; concrete records were always domain-named) |
| firing | executing an action; its record is a `JournalEntry` |
| firing id | `JournalEntryId` |
| event log | `operations_journal` table; package `journal` |
| `fired_at` | column `occurred_at` |
| binding | action arguments (event `payload`) |
| guard | (business) rule: `RuleVerdict`, `RefusalReason` |
| guard witness (R-BINDING) | `Evidence` |
| enabled binding of ClaimTruck | `ClaimCandidate`; `claimCandidates(...)` |
| firing plan / outcome / token write | `ActionPlan` / `ActionOutcome` / `StateWrite` |
| replay fold | `Journal.replay` |
| marking validator, invariant violation | `BoardAudit`, `ConsistencyViolation` |
| saga (actor) | `COORDINATOR` (DB string `coordinator`) |

---

## 4. Package layout

```
com.iohaus.dakar
├── freight/      Operational entities: Truck, Delivery, TruckStatus, DeliveryStatus,
│                 CargoClass; typed IDs (TruckId, DeliveryId, ClientId, JournalEntryId); Tons
├── desk/         The freight desk: Action hierarchy, rules, DispatchBoard, FreightDesk,
│                 ActionPlan, StateWrite, JournalEntry, Evidence, Actor, Snapshot, Route*
├── journal/      Journal.replay (P3-4). JournalEntry lives in desk/ — plans embed
│                 entries, so a journal-resident entry type would cycle with desk (v0.3.1)
├── policy/       Procedural memory: PolicySet, floor-price formula evaluation
├── agent/        Policy ports (DispatchPolicy, NegotiationPolicy) + validation
├── port/         Non-policy ports: IdSource (and nothing else — see §8.4)
├── projection/   DispatchBoard → product status (pure mapping, arch §9)
├── money/        Money, fixed scale/rounding
├── geo/          GeoPoint, pure great-circle distance
└── check/        Consistency audits (BoardAudit — the executable form of model.md INV-1…INV-5)
```

Dependency direction *within* the module: `freight`, `money`, `geo` are leaves;
`desk` depends on them; `journal`, `projection`, `check`, `agent` depend on
`desk`. No cycles (ArchUnit-enforced, §12).

---

## 5. Core types

All domain types are immutable `record`s with validating canonical
constructors. Identifiers are typed wrappers, never bare `UUID`/`String`:

```java
public record TruckId(UUID value) implements Comparable<TruckId> { ... }
public record DeliveryId(UUID value) implements Comparable<DeliveryId> { ... }
public record ClientId(UUID value) implements Comparable<ClientId> { ... }
public record JournalEntryId(UUID value) implements Comparable<JournalEntryId> { ... }
```

### 5.1 Operational entities (`freight`)

```java
public record Truck(
    TruckId id,
    TruckStatus status,                    // GARAGE | EN_ROUTE_EMPTY | LOADING | ...
    Tons capacity,
    GeoPoint position,                     // telemetry-owned (model.md §5.3)
    Duration driverHoursRemaining,         // telemetry-owned
    Optional<Instant> maintenanceDueAt,
    Optional<DeliveryId> assignedDelivery, // empty unless claimed
    long version)                          // optimistic-concurrency token, opaque here
{ /* canonical ctor validates capacity > 0, version >= 1, ... */ }

public record Delivery(
    DeliveryId id,
    DeliveryStatus status,                 // REQUESTED | MATCHED | QUOTED | ...
    ClientId client,
    Tons tonnage,
    GeoPoint origin,
    GeoPoint destination,
    CargoClass cargoClass,
    Money floorPrice,                      // procedural memory, resolved at creation
    Optional<Money> agreedPrice,
    boolean paymentConfirmed,
    Optional<Instant> quoteExpiresAt,      // set by Quote: now + policy validity (v0.2)
    Optional<Instant> paymentDueAt,        // set by ArriveDest: now + policy terms (v0.2)
    Instant deadlineAt,
    long version) { ... }
```

`version` is carried *through* the domain but interpreted only by the shell
(it becomes the `WHERE version = ?` predicate in the CockroachDB adapter). The
domain treats it as opaque and copies it into `ActionPlan` writes unchanged
except for the `+1` on modified entities.

### 5.2 Statuses (`freight`)

```java
public enum TruckStatus    { GARAGE, EN_ROUTE_EMPTY, LOADING, EN_ROUTE_LOADED, OFFLOADING, MAINTENANCE }
public enum DeliveryStatus { REQUESTED, MATCHED, QUOTED, AGREED, IN_TRANSIT,
                             AWAITING_PAYMENT, DISPUTED, COMPLETED, CANCELLED }
```

These enums are the code form of the formal state sets (model.md §2.1). The
strings persisted in CockroachDB's `status` columns are exactly `name()` of
these constants; the adapter maps mechanically, adding nothing.

The status enums live in `freight`, not `desk`: entities carry their status,
and `freight` is a leaf package — placing the enums in `desk` would create a
`freight → desk` cycle and break the §4 layering. *(Amended v0.2: `DISPUTED`
added; shipped P1-4 predates it — code catch-up in backlog P3-1. v0.3: renamed
from the spec's vocabulary per Law 4.)*

### 5.3 DispatchBoard (`desk`)

```java
public record DispatchBoard(
    SortedMap<TruckId, Truck> trucks,
    SortedMap<DeliveryId, Delivery> deliveries) {

  public DispatchBoard {                   // defensive, deterministic copies
    trucks = Collections.unmodifiableSortedMap(new TreeMap<>(trucks));
    deliveries = Collections.unmodifiableSortedMap(new TreeMap<>(deliveries));
  }
}
```

A `DispatchBoard` is a *value* — a point-in-time picture of (the relevant
slice of) the fleet and its work, loaded by the shell. The domain never
observes state change; it observes one board and emits one plan.

---

## 6. The Snapshot: data in, decisions out

The domain's unit of input is a `Snapshot` — everything an evaluation may
consult, assembled by the shell *before* the domain is entered:

```java
public record Snapshot(
    DispatchBoard board,
    RouteTable routes,      // pre-resolved distance/duration estimates
    Instant now) {}
```

*(v0.3.1: the shipped P2 shape. A `PolicySet policies` member — procedural
memory: active guardrail versions — joins this record with policy storage in
P3-7; no P2 rule consults a policy, so carrying an empty placeholder would
have violated the no-placeholder rule.)*

**Routing is data, not a call.** External routing APIs are impure and slow, so
route estimates do not enter the domain as a port that is called
mid-computation. Instead:

1. The shell asks the domain what it needs:
   `RouteRequests FreightDesk.requiredRoutes(DispatchBoard b)` — a pure
   function returning the (origin, destination) pairs any rule or annotation
   might consult for this board (e.g., every feasible truck-position → pickup
   pair).
2. The shell resolves them against its routing adapter (HTTP, cached, whatever
   — not this module's concern) into an immutable `RouteTable`.
3. The domain evaluates against the table. A missing entry is a domain error
   (`MISSING_ROUTE` refusal), never a trigger for I/O.

This keeps the core 100 % pure with zero mid-computation calls, makes every
evaluation replayable from its `Snapshot`, and moves all caching/timeout/retry
policy for routing into the shell where it belongs.

```java
public record RouteTable(SortedMap<RouteKey, RouteEstimate> entries) { ... }
public record RouteEstimate(Kilometers distance, Duration duration) {}
```

---

## 7. The desk and the execution protocol

### 7.1 Actions (`desk`)

Actions are a sealed hierarchy; each carries its arguments:

```java
public sealed interface Action permits
    ClaimTruck, Quote, AcceptQuote, QuoteExpired,
    BeginReposition, ArriveOrigin, Load, ArriveDest,
    ConfirmPayment, Offload, PaymentOverdue,
    ReleaseTruck, SettleDispute, WriteOff, TransferCargo, AbortLoaded,
    RecordDisruption, Cancel,
    RegisterTruck, Decommission, StartMaintenance, CompleteMaintenance {}

public record ClaimTruck(TruckId truck, DeliveryId delivery)          implements Action {}
public record Quote(DeliveryId delivery, Money offer, Duration validity) implements Action {}
public record Offload(TruckId truck, TruckStatus nextTruckStatus)     implements Action {}
// ... one record per action in model.md §4; the administrative sub-net
// (RegisterTruck/Decommission/Start-/CompleteMaintenance) shipped with P3-1.
```

*(v0.3.1 decisions: policy-derived windows — quote validity, payment terms —
travel as action arguments until `PolicySet` lands in P3-7; `Offload`,
`ReleaseTruck`, `AbortLoaded` carry the staff/coordinator choice of the
truck's next status; journal-only `RecordDisruption` bumps the delivery
version so concurrent stale plans are invalidated; staff-only actions refuse
non-STAFF actors at the desk with `STAFF_AUTHORIZATION_REQUIRED`.)*

### 7.2 Rules

Every action has preconditions — the business rules of model.md §4, expressed
as pure predicates over `(Action, Snapshot)` producing a verdict with
machine-readable reasons — never a bare boolean, because the shell and the
observability pipeline need *why*:

```java
public sealed interface RuleVerdict {
  record Satisfied() implements RuleVerdict {}
  record Refused(List<RefusalReason> reasons) implements RuleVerdict {}
}

public enum RefusalReason {                  // shipped set (P3-1, v0.3.1)
  WRONG_STATUS,                              // any status precondition, uniformly
  TRUCK_ALREADY_ASSIGNED, TRUCK_NOT_ASSIGNED,
  CAPACITY_EXCEEDED, INSUFFICIENT_DRIVER_HOURS, MISSING_ROUTE,
  OFFER_BELOW_FLOOR,
  QUOTE_EXPIRED, QUOTE_NOT_EXPIRED,
  PAYMENT_NOT_CONFIRMED, PAYMENT_ALREADY_CONFIRMED, PAYMENT_NOT_YET_DUE,
  TRUCK_STILL_ATTACHED,
  STAFF_AUTHORIZATION_REQUIRED, ALREADY_REGISTERED, ALREADY_AT_PICKUP }
// v0.3.1: draft per-case status names (DELIVERY_NOT_DISPUTED, ALREADY_LOADED,
// WRONG_PLACE-style) collapsed into WRONG_STATUS; time-based causes keep
// their own names because they are not status facts.
```

The rule table in model.md §4 is normative. **Traceability is one-way (Law
4): model.md §12 maps each formal rule to its implementing symbol; the code's
Javadoc explains the rule in self-contained operational language and MUST NOT
cite any `dakar.doc` document.** A reviewer with docs access checks the map;
a contributor without it reads Javadoc that stands on its own.

### 7.3 Enumeration (what the agents choose from)

```java
public final class FreightDesk {

  /** All ClaimTruck candidates available in this snapshot, deterministically
      ordered (by (deliveryId, truckId)), each annotated with the pure
      decision-support data the Dispatcher policy will see. */
  public List<ClaimCandidate> claimCandidates(Snapshot s) { ... }

  /** The admissible offer interval for a Quote on this delivery:
      [floorPrice, ceiling(policies, lane)] — empty if none. */
  public Optional<OfferBand> offerBand(DeliveryId d, Snapshot s) { ... }
}

public record ClaimCandidate(
    ClaimTruck claim,
    Kilometers repositioningDistance,
    Duration estimatedDriveTime,
    Duration driverHoursSlack) {}
```

Enumeration output order MUST be deterministic (Law 2): candidate lists are
sorted on typed IDs, so a replayed evaluation presents candidates to a policy
in the identical order.

### 7.4 Planning — the domain's only "write"

The domain never mutates anything. Its terminal operation converts an intended
action into a **plan**, or refuses it:

```java
public sealed interface ActionOutcome {
  record Planned(ActionPlan plan)             implements ActionOutcome {}
  record Refused(List<RefusalReason> reasons) implements ActionOutcome {}
  record UnknownRecord(String id)             implements ActionOutcome {}
  // InvalidRequest(detail) joins in P3-1, when actions gain arguments that
  // can be structurally invalid.
}

public record ActionPlan(
    Action action,
    List<StateWrite> writes,          // new entity values + expected old versions
    JournalEntry entry) {}            // the journal row, fully formed

public sealed interface StateWrite {  // v0.3.1: typed, not stringly —
  long expectedVersion();             // the executor pattern-matches to a table
  record TruckWrite(Truck newValue, long expectedVersion)       implements StateWrite {}
  record DeliveryWrite(Delivery newValue, long expectedVersion) implements StateWrite {}
  // ctor enforces newValue.version() == expectedVersion + 1
}
```

```java
ActionOutcome outcome = desk.plan(action, snapshot, actor, idSource);
```

*(v0.3.1: `plan` takes the requesting `Actor` — the journal entry is fully
formed by the desk, and the actor is part of the entry.)*

`plan(...)` re-evaluates the rules against the snapshot even if the caller
already enumerated — **enumeration is advice; planning is the check**. The
`entry` inside the plan is complete (including its `JournalEntryId` drawn from
`IdSource`), so the shell has nothing to invent.

### 7.5 The shell execution contract (informative, but binding on adapters)

The CockroachDB adapter in the shell MUST execute an `ActionPlan` as exactly
one `SERIALIZABLE` transaction: apply every `StateWrite` guarded by its
`expectedVersion`, insert the `entry`, commit (arch §7). On a serialization
failure (`40001`) or a version mismatch, the adapter MUST NOT patch the plan;
it re-loads a fresh `Snapshot` and re-enters the domain. Because planning is
pure (Laws 1–2), re-entry is always safe and always cheap.

This is the whole point of the plan/execute split: **the domain decides, once,
purely; the shell executes, atomically, with retries.** Neither does the
other's job.

---

## 8. Ports and dependency injection

### 8.1 What a port is here

A port is a plain Java interface **defined in this module**, implemented in the
shell, and passed to domain services via constructor. No annotations, no
container awareness, no defaults that do I/O.

### 8.2 The port inventory (closed list)

The port list is deliberately tiny and closed — a new port requires amending
this document first:

| Port | Package | Contract |
|---|---|---|
| `IdSource` | `port` | Supplies fresh `JournalEntryId`/`DeliveryId`… values. The only sanctioned impurity. Test/replay implementations are seeded and sequential. |
| `DispatchPolicy` | `agent` | Chooses among enumerated `ClaimCandidate`s. LLM-backed in production. |
| `NegotiationPolicy` | `agent` | Chooses an offer within an `OfferBand`, given retrieved episodic context. LLM-backed in production. |

```java
public interface IdSource {
  UUID next();                       // shell: UUID.randomUUID(); tests: seeded
}

public interface DispatchPolicy {
  /** MUST return an element of {@code candidates} or empty. Anything else is
      a policy violation and is rejected by the caller (§8.3). */
  Optional<ClaimCandidate> choose(List<ClaimCandidate> candidates, DispatchContext ctx);
}

public interface NegotiationPolicy {
  /** MUST return an offer within {@code band}. Out-of-band values are
      REJECTED, not clamped — see §8.3. */
  Optional<Money> proposeOffer(OfferBand band, NegotiationContext ctx);
}
```

`DispatchContext` / `NegotiationContext` are immutable records assembled by the
shell (retrieved episodes, client profile, lane stats — arch §6.2–6.3). The
domain defines their *shape*; the shell fills them. Note what is absent:
prompts, model names, token budgets. Those are Bedrock-adapter concerns and
MUST NOT leak into these types.

### 8.3 The validation rule: ports are inputs, not authorities

The domain treats every port return value as untrusted input:

- A `DispatchPolicy` choice that is not (by `equals`) an element of the
  candidate list → discarded; outcome `PolicyViolation`, logged by the shell,
  no execution.
- A `NegotiationPolicy` offer outside the `OfferBand` → **rejected, not
  clamped**. Clamping would silently launder an out-of-policy decision into a
  legal-looking one; rejection preserves the signal (a spike in policy
  violations is a model-health alarm — arch §11).
- And regardless, `plan(...)` re-checks the rules (§7.4). Defense in depth:
  even a buggy caller cannot execute a refused action.

This is "the LLM proposes, the formalism disposes" (arch §4.5) expressed at
the type level.

### 8.3a Adapter-side deliberation loops (v0.2)

A policy adapter in the shell MAY implement its port as a **bounded tool-use
conversation** with the model (arch §8.2): perception tool calls (episode
search, client profile, lane stats, route estimates — all shell reads, served
`AS OF SYSTEM TIME` of the decision episode's snapshot) followed by an action
proposal. This changes nothing in this module:

- The port signatures above are unchanged; the loop is invisible to the domain.
- The domain is entered exactly once per intended execution (`plan(...)`),
  which re-checks the rules regardless of what the conversation concluded
  (§7.4).
- Loop bounds (`max_perception_calls`, `max_action_attempts`) are procedural
  memory (`PolicySet` data), not adapter constants.
- §8.3 validation applies to the loop's final proposal exactly as it applies
  to a single-shot one. A rule refusal is returned to the model as a typed
  tool error (a percept), never repaired by the adapter.

Laws 1–2 are untouched: all impurity — the conversation, the retrievals, the
retries — lives in the shell.

### 8.4 What is deliberately *not* a port

| Not a port | Why |
|---|---|
| Routing | Data in the `Snapshot` (§6). No mid-computation calls. |
| Clock | Data in the `Snapshot` (`now`). A clock port invites drift between two reads in one evaluation. |
| Persistence / repository | The domain never loads or saves. The shell loads `Snapshot`s and executes `ActionPlan`s. A repository interface in the domain would put I/O sequencing decisions inside it. |
| Embeddings | The domain emits transcript text inside `NegotiationEpisodeDraft`; embedding is a shell concern (Bedrock) and the vector never enters the domain. |
| Logging / metrics | The domain returns rich outcome types (`RefusalReason`, `ActionOutcome`); the shell logs them. |

---

## 9. Primitives: money, geo, time

**Money** (`money`): `record Money(BigDecimal amount, Currency currency)`.
Scale is fixed at 2, rounding `HALF_EVEN`, enforced in the canonical
constructor. Arithmetic lives on `Money` (`plus`, `times(BigDecimal)`,
`isAtLeast`) — raw `BigDecimal` maths in domain logic is forbidden by review
convention. Currency mismatches throw `IllegalArgumentException` at
construction, never propagate.

**Geo** (`geo`): `record GeoPoint(double latDeg, double lonDeg)` with a pure
haversine `Kilometers distanceTo(GeoPoint other)`. This is an *estimator* used
in rules and annotations (driver-hours slack, repositioning cost hints); road
distances come from the `RouteTable`. Golden tests pin exact outputs so JVM
uniformity is verified, not assumed.

**Time**: `Instant` and `Duration` only, always UTC. `now` comes from the
`Snapshot`. Any comparison against deadlines or maintenance windows uses
`snapshot.now()` — twice in one evaluation yields the same value by
construction.

---

## 10. The journal and replay (`journal`)

Every executed action produces a `JournalEntry` — the domain-typed form of the
`operations_journal` row (arch §6.2):

```java
public record JournalEntry(
    JournalEntryId id,
    Action action,                   // the full arguments, typed
    Evidence evidence,               // route estimates, policy windows, and
                                     // signal/position facts the rules consulted —
                                     // replay must be closed over the journal alone
                                     // (model.md R-BINDING, §4.5)
    Actor actor,                     // DISPATCHER | NEGOTIATOR | COORDINATOR | STAFF | PRODUCT
    Instant occurredAt) {}           // = snapshot.now(), NOT a wall-clock read
```

The module ships replay:

```java
public final class Journal {
  /** Applies the journal to an initial board. Total: throws on any entry whose
      rules would not have been satisfied — a corrupted or reordered journal
      fails loudly. */
  public static DispatchBoard replay(DispatchBoard initial, List<JournalEntry> entries) { ... }
}
```

Replay is the module's deepest self-test and the system's audit spine
(arch MI-2): replay the production journal from the initial board and the
result MUST equal the live board's **operational-state projection** — statuses,
assignments, prices, flags, stamped instants — exactly. Telemetry attributes
(`position`, `driverHoursRemaining`) are environment-owned measurements,
excluded from the comparison by definition, not by tolerance (model.md §5.3);
rules that consulted them are re-checked against the entry's `Evidence`.
Determinism (Law 2) is what makes this equality achievable; any divergence is
a severity-1 defect in either the domain or the adapter.

Journal evolution rule: `JournalEntry` fields are append-only; an action
record, once released, never changes shape — it gains a versioned successor
(`ClaimTruckV2`) and `Journal.replay` handles both forever.

---

## 11. Projections (`projection`)

The product-facing status vocabulary (arch §9) is a pure function in this
module, so the API layer cannot invent its own mapping:

```java
public enum ClientStatus { RECEIVED, PREPARING_QUOTE, QUOTE_READY, DRIVER_EN_ROUTE,
                           IN_TRANSIT, AWAITING_PAYMENT, PAYMENT_OVERDUE,
                           DELIVERED, CANCELLED }

public final class StatusProjection {
  public static ClientStatus of(Delivery d, Optional<Truck> assigned) { ... }
}
```

The projection consumes internal vocabulary and emits product vocabulary
(arch §9's abstraction rules, enforced at the source). Law 4 already keeps
spec words out of the internal vocabulary; this projection additionally keeps
*internal operational* words (board, journal, action) out of anything a client
sees.

---

## 12. Mechanical enforcement of the laws

Aspirations rot; builds don't. Three layers:

**1. Maven Enforcer** (in `dakar/pom.xml`): `<bannedDependencies>`
excluding everything (compile scope) except the JDK and `org.jspecify`. A
transitive framework dependency fails the build, not a review.

**2. ArchUnit test suite** (`src/test/java`, runs in `mvn test`):

```java
// No infrastructure packages, ever.
noClasses().that().resideInAPackage("com.iohaus.dakar..")
    .should().dependOnClassesThat().resideInAnyPackage(
        "java.sql..", "java.net..", "java.nio.file..", "java.io..",
        "jakarta..", "javax..", "io.quarkus..", "software.amazon..",
        "com.fasterxml..", "org.slf4j..", "org.jboss..");

// No hidden nondeterminism.
noClasses().that().resideInAPackage("com.iohaus.dakar..")
    .should().callMethod(System.class, "currentTimeMillis")
    .orShould().callMethod(System.class, "nanoTime")
    .orShould().callMethod(UUID.class, "randomUUID")
    .orShould().dependOnClassesThat().belongToAnyOf(
        java.util.Random.class, java.security.SecureRandom.class);

// Law 4: the vocabulary firewall on code symbols.
noClasses()
    .should().haveSimpleNameContaining("Token")
    .orShould().haveSimpleNameContaining("Marking")
    .orShould().haveSimpleNameContaining("Place")
    .orShould().haveSimpleNameContaining("Firing")
    .orShould().haveSimpleNameContaining("Transition")
    .orShould().haveSimpleNameContaining("Guard")
    .orShould().haveSimpleNameContaining("Binding")
    .orShould().haveSimpleNameContaining("Petri");

// Intra-module layering (§4): desk may not depend on agent, etc.
// Closed package set: freight, desk, journal, policy, agent, port,
// projection, money, geo, check — and nothing else.
```

(`java.io` is banned as a *dependency target* with a narrow allowance list for
`Serializable`-free record plumbing if ever needed; the default is a hard no.)

**3. Review conventions** not expressible in ArchUnit — raw `BigDecimal`
arithmetic outside `money`, `HashMap` iteration feeding an output, a new port
without a §8.2 amendment, **spec vocabulary in comments/strings, or any
reference to `dakar.doc` from code (Law 4)** — are called out in this document
so reviewers cite a section, not a taste.

---

## 13. Testing strategy

The purity dividend: **domain tests use no mocks of infrastructure, no
Testcontainers, no Quarkus test harness.** Construct values, call functions,
assert. The full suite runs in seconds.

| Layer | Technique | What it pins |
|---|---|---|
| Rules | Example-based JUnit, table-driven from model.md §4 (docs-side mapping) | Each rule, satisfied and each refusal reason |
| Consistency (model.md INV-1…INV-5) | **Property-based (jqwik)**: generate random legal action sequences from random initial boards; assert `BoardAudit` after every replay step | No reachable board violates a consistency property — including assignment injectivity (no double-booking). The executable verification of the formal claims |
| Replay | Property-based: `replay(initial, journal)` equals step-by-step application; journal mutations (drop/reorder/duplicate) fail loudly | Law 2, MI-2 |
| Enumeration determinism | Same snapshot twice → identical candidate lists (order included) | Law 2 |
| Policy validation | Hostile fake policies (returns non-candidate, out-of-band offer, null) | §8.3 rejection paths |
| Projection | Exhaustive over `DeliveryStatus × TruckStatus` | Total mapping, no internal vocabulary leaks |
| Geo/Money | Golden tests with exact expected values | Cross-JVM determinism |
| Architecture | ArchUnit (§12) | Laws 1–4 |

The property-based consistency suite is the load-bearing one: it is the shipped
verification of the formal claims (model.md §11), executed on every build
against the *actual* Java implementation. The offline TLA+ spec of the
contended-claim protocol is roadmap (backlog, Post-MVP), not a present-tense
dependency.

---

## 14. Anti-scope: what must never live here

For avoidance of doubt, the following belong to the shell and MUST NOT enter
this module, however convenient it looks in the moment:

- SQL, transaction management, `40001` retry loops (CockroachDB adapter)
- JSON/serialization annotations or mappers (API + persistence adapters)
- Prompts, model IDs, token budgets, tool definitions (Bedrock adapter)
- Embedding vectors and vector-search queries (memory adapter)
- HTTP anything; routing-API clients and their caching (routing adapter)
- Logging, metrics, tracing (shell observes domain *outcomes*)
- Configuration reading of any kind
- Scheduling, EventBridge shapes, Lambda handlers (agent shells)
- CDI/`@ApplicationScoped`/`@Inject` (wiring lives in `dakar-quarkus`'s producers)
- **Specification vocabulary or references to `dakar.doc` (Law 4)**

The test for a proposed addition: *could this line ever behave differently on
two machines, two runs, or two regions?* If yes, it is shell.

---

## 15. Evolution and versioning

- **Structural model changes** (new status/action, changed preconditions) are
  a domain version bump, a model.md + architecture-doc update, and a
  replay-compatibility review (§10) — in that order.
- **Rule-threshold changes** (a new floor-price formula, revised driver-hours
  margin) are *not* code changes; they are `PolicySet` data (procedural
  memory, arch §6.3), versioned and rolled back in the database. The domain
  evaluates whatever active policy version the `Snapshot` carries.
- **New agent policies** implement existing port interfaces in the shell; the
  domain is untouched.
- **Renames touching the vocabulary map** (Law 4) amend §3's table in the same
  PR.
- This document is amended in the same PR as any change it governs.

---

## Appendix A — Public API quick reference

```java
// Entry points (all pure unless noted)
FreightDesk.requiredRoutes(DispatchBoard)               → RouteRequests
FreightDesk.claimCandidates(Snapshot)                   → List<ClaimCandidate>
FreightDesk.offerBand(DeliveryId, Snapshot)             → Optional<OfferBand>
FreightDesk.plan(Action, Snapshot, Actor, IdSource)     → ActionOutcome  // IdSource = sanctioned impurity
Journal.replay(DispatchBoard, List<JournalEntry>)       → DispatchBoard
StatusProjection.of(Delivery, Optional<Truck>)          → ClientStatus
BoardAudit.check(DispatchBoard)                         → List<ConsistencyViolation>

// Ports (implemented by the shell, constructor-injected)
IdSource.next()                                                → UUID
DispatchPolicy.choose(List<ClaimCandidate>, DispatchContext)   → Optional<ClaimCandidate>
NegotiationPolicy.proposeOffer(OfferBand, NegotiationContext)  → Optional<Money>
```

## Appendix B — Wiring example (informative, lives in the `dakar-quarkus` module, not here)

```java
// Quarkus shell: CDI producer wires the pure core with impure adapters.
@ApplicationScoped
public class DakarConfiguration {
  @Produces FreightDesk desk() { return new FreightDesk(); }
  @Produces IdSource ids() { return UUID::randomUUID; }
  @Produces DispatchPolicy dispatchPolicy(BedrockClient bedrock) {
    return new BedrockDispatchPolicy(bedrock);   // adapter: shell module
  }
}
```

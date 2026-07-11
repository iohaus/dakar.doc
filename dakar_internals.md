# Dakar Internals — The Domain Library (`dakar`)

**Status:** Normative v0.2 — this document *defines* the module; code conforms to it, not vice versa. *(v0.2 aligns with system_architecture.md v0.2: closed net with timeout/dispute transitions, INV-5, adapter-side tool-use loops. Code catch-up for the amended token/place shapes is scheduled in backlog P3-1.)*
**Audience:** Dakar engineering
**Companion:** [system_architecture.md](system_architecture.md) is the system-level authority; this document is the module-level authority for the domain library. Where the two disagree, fix the disagreement — neither silently wins.

The key words **MUST**, **MUST NOT**, **SHOULD**, and **MAY** are used as in RFC 2119.

---

## 1. Purpose and authority

`dakar` is the Java module containing **all core domain logic** for Dakar:
the Colored Petri Net (places, transitions, guards, marking), the color-set
types (trucks, deliveries, assignments), guard evaluation, firing planning,
event types, replay, invariant checks, and product-status projections.

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
| Java | 25 (records, sealed interfaces, pattern matching) |
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

## 3. The three design laws

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
event-log replay reproduces the marking bit-for-bit (§10), and the CPN model
checker and the runtime provably evaluate the same guards.

### Law 3 — Ports own the boundary

Every interaction with the outside world — routing estimates, LLM policy
choices, ID generation — is expressed as a **domain-owned interface** (§8),
implemented in the shell and passed in through constructors. The domain never
names an implementation technology. Corollary: the domain **re-validates
everything a port returns** (§8.3); ports are inputs, not authorities.

---

## 4. Package layout

```
com.iohaus.dakar
├── token/        Color sets + places: TruckToken, DeliveryToken, TruckPlace, DeliveryPlace; typed IDs (assignment is truck color, not a token — arch §4.2 v0.2)
├── net/          Transitions, Guards, Marking, DakarNet, FiringPlan
├── event/        TransitionFiringEvent, replay (fold) function
├── policy/       Procedural memory: PolicySet, floor-price formula evaluation
├── agent/        Policy ports (DispatchPolicy, NegotiationPolicy) + validation
├── port/         Non-policy ports: IdSource (and nothing else — see §8.4)
├── projection/   Marking → product status (pure mapping, arch §9)
├── money/        Money, fixed scale/rounding
├── geo/          GeoPoint, pure great-circle distance
└── check/        Executable invariants INV-1…INV-5 (MarkingValidator)
```

Dependency direction *within* the module: `token`, `money`, `geo` are leaves;
`net` depends on them; `event`, `projection`, `check`, `agent` depend on `net`.
No cycles (ArchUnit-enforced, §12).

---

## 5. Core types

All domain types are immutable `record`s with validating canonical
constructors. Identifiers are typed wrappers, never bare `UUID`/`String`:

```java
public record TruckId(UUID value) implements Comparable<TruckId> { ... }
public record DeliveryId(UUID value) implements Comparable<DeliveryId> { ... }
public record ClientId(UUID value) implements Comparable<ClientId> { ... }
```

### 5.1 Color sets (`token`)

```java
public record TruckToken(
    TruckId id,
    TruckPlace place,                      // Garage | EnRouteEmpty | Loading | ...
    Tons capacity,
    GeoPoint position,
    Duration driverHoursRemaining,
    Optional<Instant> maintenanceDueAt,
    Optional<DeliveryId> assignedDelivery, // empty unless claimed
    long version)                          // optimistic-concurrency token, opaque here
{ /* canonical ctor validates capacity > 0, version >= 1, ... */ }

public record DeliveryToken(
    DeliveryId id,
    DeliveryPlace place,                   // Requested | Matched | Quoted | ...
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
domain treats it as opaque and copies it into `FiringPlan` writes unchanged
except for the `+1` on modified tokens.

### 5.2 Places (`token`)

```java
public enum TruckPlace    { GARAGE, EN_ROUTE_EMPTY, LOADING, EN_ROUTE_LOADED, OFFLOADING, MAINTENANCE }
public enum DeliveryPlace { REQUESTED, MATCHED, QUOTED, AGREED, IN_TRANSIT,
                            AWAITING_PAYMENT, DISPUTED, COMPLETED, CANCELLED }
```

These enums are the single source of truth for the CPN places. The strings
persisted in CockroachDB's `place` columns are exactly `name()` of these
constants; the adapter maps mechanically, adding nothing.

The place enums live in `token`, not `net`: tokens carry their place, and
`token` is a leaf package — placing the enums in `net` would create a
`token → net` cycle and break the §4 layering. *(Amended v0.1: originally
listed under `net`. Amended v0.2: `DISPUTED` added per arch §4.3; the shipped
P1-4 enum and `DeliveryToken` predate this — code catch-up is part of backlog
P3-1.)*

### 5.3 Marking (`net`)

```java
public record Marking(
    SortedMap<TruckId, TruckToken> trucks,
    SortedMap<DeliveryId, DeliveryToken> deliveries) {

  public Marking {                         // defensive, deterministic copies
    trucks = Collections.unmodifiableSortedMap(new TreeMap<>(trucks));
    deliveries = Collections.unmodifiableSortedMap(new TreeMap<>(deliveries));
  }
}
```

A `Marking` is a *value* — a point-in-time snapshot of (the relevant slice of)
the net's state, loaded by the shell. The domain never observes state change;
it observes one marking and emits one plan.

---

## 6. The Snapshot: data in, decisions out

The domain's unit of input is a `Snapshot` — everything an evaluation may
consult, assembled by the shell *before* the domain is entered:

```java
public record Snapshot(
    Marking marking,
    PolicySet policies,     // procedural memory: active guardrail versions
    RouteTable routes,      // pre-resolved distance/duration estimates
    Instant now) {}
```

**Routing is data, not a call.** External routing APIs are impure and slow, so
route estimates do not enter the domain as a port that is called
mid-computation. Instead:

1. The shell asks the domain what it needs:
   `RouteRequests DakarNet.requiredRoutes(Marking m)` — a pure function
   returning the (origin, destination) pairs any guard or annotation might
   consult for this marking (e.g., every feasible truck-position → pickup pair).
2. The shell resolves them against its routing adapter (HTTP, cached, whatever
   — not this module's concern) into an immutable `RouteTable`.
3. The domain evaluates against the table. A missing entry is a domain error
   (`MissingRoute` outcome), never a trigger for I/O.

This keeps the core 100 % pure with zero mid-computation calls, makes every
evaluation replayable from its `Snapshot`, and moves all caching/timeout/retry
policy for routing into the shell where it belongs.

```java
public record RouteTable(SortedMap<RouteKey, RouteEstimate> entries) { ... }
public record RouteEstimate(Kilometers distance, Duration duration) {}
```

---

## 7. The net and the firing protocol

### 7.1 Transitions (`net`)

Transitions are a sealed hierarchy; each carries its binding variables:

```java
public sealed interface Transition
    permits ClaimTruck, Quote, AcceptQuote, QuoteExpired, BeginReposition,
            ArriveOrigin, Load, ArriveDest, ConfirmPayment, PaymentOverdue,
            Offload, ReleaseTruck, SettleDispute, WriteOff, TransferCargo,
            AbortLoaded, RecordDisruption, Cancel {}

public record ClaimTruck(TruckId truck, DeliveryId delivery) implements Transition {}
public record Quote(DeliveryId delivery, Money offer)         implements Transition {}
public record Offload(TruckId truck, DeliveryId delivery)     implements Transition {}
// ... one record per transition in arch §4.4
```

### 7.2 Guards

Guards are pure predicates over `(Transition, Snapshot)` producing a verdict
with machine-readable reasons — never a bare boolean, because the shell and
the observability pipeline need *why*:

```java
public sealed interface GuardVerdict {
  record Satisfied() implements GuardVerdict {}
  record Violated(List<GuardReason> reasons) implements GuardVerdict {}
}

public enum GuardReason {
  CAPACITY_EXCEEDED, INSUFFICIENT_DRIVER_HOURS, TRUCK_ALREADY_ASSIGNED,
  OFFER_BELOW_FLOOR, PAYMENT_NOT_CONFIRMED, WRONG_PLACE, ALREADY_LOADED,
  MISSING_ROUTE, DEADLINE_UNREACHABLE,
  QUOTE_EXPIRED, QUOTE_NOT_EXPIRED,          // AcceptQuote / QuoteExpired (v0.2)
  PAYMENT_NOT_YET_DUE, PAYMENT_ALREADY_CONFIRMED, // PaymentOverdue / ReleaseTruck (v0.2)
  DELIVERY_NOT_DISPUTED, TRUCK_STILL_ATTACHED,    // dispute-resolution guards (v0.2)
  STAFF_AUTHORIZATION_REQUIRED /* ... */ }
```

The guard table in system_architecture.md §4.4 is normative; each row is one
guard implementation here, and each MUST cite its architecture row in Javadoc.

### 7.3 Enumeration (what the agents choose from)

```java
public final class DakarNet {

  /** All bindings of ClaimTruck enabled in this snapshot, deterministically
      ordered (by (deliveryId, truckId)), each annotated with the pure
      decision-support data the Dispatcher policy will see. */
  public List<EnabledClaim> enabledClaims(Snapshot s) { ... }

  /** The admissible offer interval for a Quote on this delivery:
      [floorPrice, ceiling(policies, lane)] — empty if none. */
  public Optional<OfferBand> offerBand(DeliveryId d, Snapshot s) { ... }
}

public record EnabledClaim(
    ClaimTruck binding,
    Kilometers repositioningDistance,
    Duration estimatedDriveTime,
    Duration driverHoursSlack) {}
```

Enumeration output order MUST be deterministic (Law 2): candidate lists are
sorted on typed IDs, so a replayed evaluation presents candidates to a policy
in the identical order.

### 7.4 Planning — the domain's only "write"

The domain never mutates anything. Its terminal operation converts an intended
transition into a **plan**, or rejects it:

```java
public sealed interface FiringOutcome {
  record Planned(FiringPlan plan)             implements FiringOutcome {}
  record NotEnabled(List<GuardReason> why)    implements FiringOutcome {}
  record UnknownToken(String tokenId)         implements FiringOutcome {}
  record InvalidBinding(String detail)        implements FiringOutcome {}
}

public record FiringPlan(
    Transition transition,
    List<TokenWrite> writes,          // new token values + expected old versions
    TransitionFiringEvent event) {}   // the episodic-log row, fully formed

public record TokenWrite(
    String tokenId, long expectedVersion, Object newTokenValue) {}
```

```java
FiringOutcome plan = net.plan(transition, snapshot, idSource);
```

`plan(...)` re-evaluates the guard against the snapshot even if the caller
already enumerated — **enumeration is advice; planning is the check**. The
`event` inside the plan is complete (including its `FiringId` drawn from
`IdSource`), so the shell has nothing to invent.

### 7.5 The shell execution contract (informative, but binding on adapters)

The CockroachDB adapter in the shell MUST execute a `FiringPlan` as exactly one
`SERIALIZABLE` transaction: apply every `TokenWrite` guarded by its
`expectedVersion`, insert the `event`, commit (arch §7). On a serialization
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
| `IdSource` | `port` | Supplies fresh `FiringId`/`DeliveryId`… values. The only sanctioned impurity. Test/replay implementations are seeded and sequential. |
| `DispatchPolicy` | `agent` | Chooses among enumerated `EnabledClaim` candidates. LLM-backed in production. |
| `NegotiationPolicy` | `agent` | Chooses an offer within an `OfferBand`, given retrieved episodic context. LLM-backed in production. |

```java
public interface IdSource {
  UUID next();                       // shell: UUID.randomUUID(); tests: seeded
}

public interface DispatchPolicy {
  /** MUST return an element of {@code candidates} or empty. Anything else is
      a policy violation and is rejected by the caller (§8.3). */
  Optional<EnabledClaim> choose(List<EnabledClaim> candidates, DispatchContext ctx);
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
  no firing.
- A `NegotiationPolicy` offer outside the `OfferBand` → **rejected, not
  clamped**. Clamping would silently launder an out-of-policy decision into a
  legal-looking one; rejection preserves the signal (a spike in policy
  violations is a model-health alarm — arch §11).
- And regardless, `plan(...)` re-checks the guard (§7.4). Defense in depth:
  even a buggy caller cannot fire a disabled transition.

This is "the LLM proposes, the formalism disposes" (arch §4.5) expressed at
the type level.

### 8.3a Adapter-side deliberation loops (v0.2)

A policy adapter in the shell MAY implement its port as a **bounded tool-use
conversation** with the model (arch §8.2): perception tool calls (episode
search, client profile, lane stats, route estimates — all shell reads, served
`AS OF SYSTEM TIME` of the decision episode's snapshot) followed by an action
proposal. This changes nothing in this module:

- The port signatures above are unchanged; the loop is invisible to the domain.
- The domain is entered exactly once per intended firing (`plan(...)`), which
  re-checks the guard regardless of what the conversation concluded (§7.4).
- Loop bounds (`max_perception_calls`, `max_action_attempts`) are procedural
  memory (`PolicySet` data), not adapter constants.
- §8.3 validation applies to the loop's final proposal exactly as it applies
  to a single-shot one. A guard rejection is returned to the model as a typed
  tool error (a percept), never repaired by the adapter.

Laws 1–2 are untouched: all impurity — the conversation, the retrievals, the
retries — lives in the shell.

### 8.4 What is deliberately *not* a port

| Not a port | Why |
|---|---|
| Routing | Data in the `Snapshot` (§6). No mid-computation calls. |
| Clock | Data in the `Snapshot` (`now`). A clock port invites drift between two reads in one evaluation. |
| Persistence / repository | The domain never loads or saves. The shell loads `Snapshot`s and executes `FiringPlan`s. A repository interface in the domain would put I/O sequencing decisions inside it. |
| Embeddings | The domain emits transcript text inside `NegotiationEpisodeDraft`; embedding is a shell concern (Bedrock) and the vector never enters the domain. |
| Logging / metrics | The domain returns rich outcome types (`GuardReason`, `FiringOutcome`); the shell logs them. |

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
in guards and annotations (driver-hours slack, repositioning cost hints); road
distances come from the `RouteTable`. Golden tests pin exact outputs so JVM
uniformity is verified, not assumed.

**Time**: `Instant` and `Duration` only, always UTC. `now` comes from the
`Snapshot`. Any comparison against deadlines or maintenance windows uses
`snapshot.now()` — twice in one evaluation yields the same value by
construction.

---

## 10. Events and replay (`event`)

Every firing produces a `TransitionFiringEvent` — the domain-typed form of the
`transition_firing` row (arch §6.2):

```java
public record TransitionFiringEvent(
    FiringId id,
    Transition transition,           // the full binding, typed
    GuardWitness witness,            // route estimates, policy windows, and
                                     // signal/position predicates the guard
                                     // consulted — model.md R-BINDING (§4.5):
                                     // replay must be closed over (M₀, log)
    Actor actor,                     // DISPATCHER | NEGOTIATOR | SAGA | STAFF
    Instant firedAt) {}              // = snapshot.now(), NOT a wall-clock read
```

The module ships the **fold**:

```java
public final class Replay {
  /** Applies the log to an initial marking. Total: throws on any event that
      would not have been enabled — a corrupted or reordered log fails loudly. */
  public static Marking fold(Marking initial, List<TransitionFiringEvent> log) { ... }
}
```

Replay is the module's deepest self-test and the system's audit spine
(arch MI-2): fold the production event log from the initial marking and the
result MUST equal the live marking's **net-state projection** — places,
assignments, prices, flags, stamped instants — exactly. Telemetry attributes
(`position`, `driverHoursRemaining`) are environment-owned measurements,
excluded from the comparison by definition, not by tolerance (model.md §5.3);
guards that consulted them are re-checked against the event's `GuardWitness`.
Determinism (Law 2) is what makes this equality achievable; any divergence is
a severity-1 defect in either the domain or the adapter.

Event evolution rule: `TransitionFiringEvent` fields are append-only; a
transition record, once released, never changes shape — it gains a versioned
successor (`ClaimTruckV2`) and `Replay.fold` handles both forever.

---

## 11. Projections (`projection`)

The product-facing status vocabulary (arch §9) is a pure function in this
module, so the API layer cannot invent its own mapping:

```java
public enum ClientStatus { RECEIVED, PREPARING_QUOTE, QUOTE_READY, DRIVER_EN_ROUTE,
                           IN_TRANSIT, AWAITING_PAYMENT, PAYMENT_OVERDUE,
                           DELIVERED, CANCELLED }

public final class StatusProjection {
  public static ClientStatus of(DeliveryToken d, Optional<TruckToken> assigned) { ... }
}
```

The projection consumes internal vocabulary and emits product vocabulary. The
words *token*, *place*, *transition* MUST NOT appear in `ClientStatus` names or
any string this package produces (arch §9's abstraction rules, enforced at the
source).

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

// Intra-module layering (§4): net may not depend on agent, etc.
```

(`java.io` is banned as a *dependency target* with a narrow allowance list for
`Serializable`-free record plumbing if ever needed; the default is a hard no.)

**3. Review conventions** not expressible in ArchUnit — raw `BigDecimal`
arithmetic outside `money`, `HashMap` iteration feeding an output, a new port
without a §8.2 amendment — are called out in this document so reviewers cite a
section, not a taste.

---

## 13. Testing strategy

The purity dividend: **domain tests use no mocks of infrastructure, no
Testcontainers, no Quarkus test harness.** Construct values, call functions,
assert. The full suite runs in seconds.

| Layer | Technique | What it pins |
|---|---|---|
| Guards | Example-based JUnit, table-driven from arch §4.4 | Each guard row, satisfied and each violation reason |
| Invariants INV-1…INV-5 | **Property-based (jqwik)**: generate random legal firing sequences from random initial markings; assert `MarkingValidator` after every fold step | No reachable marking violates an invariant — including INV-5 assignment injectivity (no double-booking). The executable verification of the CPN claims (arch §5) |
| Replay | Property-based: `fold(initial, log)` equals step-by-step application; log mutations (drop/reorder/duplicate) fail loudly | Law 2, MI-2 |
| Enumeration determinism | Same snapshot twice → identical candidate lists (order included) | Law 2 |
| Policy validation | Hostile fake policies (returns non-candidate, out-of-band offer, null) | §8.3 rejection paths |
| Projection | Exhaustive over `DeliveryPlace × TruckPlace` | Total mapping, no internal vocabulary leaks |
| Geo/Money | Golden tests with exact expected values | Cross-JVM determinism |
| Architecture | ArchUnit (§12) | Laws 1–3 |

The property-based invariant suite is the load-bearing one: it is the shipped
verification of the CPN claims (arch §5), executed on every build against the
*actual* Java implementation. The offline TLA+ spec of the contended-claim
protocol is roadmap (backlog, Post-MVP), not a present-tense dependency.

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
- CDI/`@ApplicationScoped`/`@Inject` (wiring lives in `dakar`'s producers)

The test for a proposed addition: *could this line ever behave differently on
two machines, two runs, or two regions?* If yes, it is shell.

---

## 15. Evolution and versioning

- **Structural net changes** (new place/transition, changed arcs) are a domain
  version bump, an architecture-doc update, and a replay-compatibility review
  (§10) — in that order.
- **Guard-threshold changes** (a new floor-price formula, revised driver-hours
  margin) are *not* code changes; they are `PolicySet` data (procedural
  memory, arch §6.3), versioned and rolled back in the database. The domain
  evaluates whatever active policy version the `Snapshot` carries.
- **New agent policies** implement existing port interfaces in the shell; the
  domain is untouched.
- This document is amended in the same PR as any change it governs.

---

## Appendix A — Public API quick reference

```java
// Entry points (all pure unless noted)
DakarNet.requiredRoutes(Marking)                      → RouteRequests
DakarNet.enabledClaims(Snapshot)                      → List<EnabledClaim>
DakarNet.offerBand(DeliveryId, Snapshot)              → Optional<OfferBand>
DakarNet.plan(Transition, Snapshot, IdSource)         → FiringOutcome   // IdSource = sanctioned impurity
Replay.fold(Marking, List<TransitionFiringEvent>)     → Marking
StatusProjection.of(DeliveryToken, Optional<TruckToken>) → ClientStatus
MarkingValidator.check(Marking)                       → List<InvariantViolation>

// Ports (implemented by the shell, constructor-injected)
IdSource.next()                                       → UUID
DispatchPolicy.choose(List<EnabledClaim>, DispatchContext)   → Optional<EnabledClaim>
NegotiationPolicy.proposeOffer(OfferBand, NegotiationContext) → Optional<Money>
```

## Appendix B — Wiring example (informative, lives in the `dakar-quarkus` module, not here)

```java
// Quarkus shell: CDI producer wires the pure core with impure adapters.
@ApplicationScoped
public class DakarConfgiuration {
  @Produces DakarNet net() { return new DakarNet(); }
  @Produces IdSource ids() { return UUID::randomUUID; }
  @Produces DispatchPolicy dispatchPolicy(BedrockClient bedrock) {
    return new BedrockDispatchPolicy(bedrock);   // adapter: shell module
  }
}
```

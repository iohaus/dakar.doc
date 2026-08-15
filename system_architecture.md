# Dakar — System Architecture

**Status:** Draft v0.2.1 (MVP design — full revision, see revision note)
**Audience:** Dakar engineering, solutions architecture, challenge reviewers
**Scope:** The agentic freight-orchestration platform and its memory layer

---

## Revision note (v0.1 → v0.2)

Revised in full while the codebase is still at Phase 1 (value objects only), so
the paper could be fixed before the code exists to contradict it. The changes:

1. **Agency is now a governed constraint, not an assumption.** §8 audits the
   system against the five defining characteristics of an AI agent (perception,
   reasoning, action, autonomy, goal-direction), states each agent's objective
   explicitly, and defines an autonomy ladder. Agents act through **tool use**:
   the net's transitions are the action tools — typed, guarded, executed as
   serializable firings — and perception tools let an agent direct its own
   retrieval (§8.2). "Where are the actions and what tool is it calling" now
   has a literal answer in the API traces, not just in prose.
2. **The net is closed and deadlock-free.** The free-floating `Assignment`
   token is replaced by truck color (`assignedDelivery`), matching the schema
   and the Java core; exception tokens are replaced by episodic disruption
   records; and timeout/dispute transitions (`QuoteExpired`, `PaymentOverdue`,
   `ReleaseTruck`, `SettleDispute`, `WriteOff`) close the post-`Load`
   compensation gap that could previously strand a truck in `Offloading`
   forever (§4.3–4.4).
3. **Safety claims now match their theorems.** New **INV-5 (assignment
   injectivity)** carries the no-double-booking claim that v0.1 mis-attributed
   to INV-1; **LIVE-1** is restated with its fairness assumptions explicit
   (§5).
4. **Verification claims match reality.** The executable property-based
   invariant suite is the MVP verification instrument; a TLA+ companion spec is
   roadmap, no longer described in the present tense (§5).
5. **Event-log honesty.** The design is *event-logged with a replay audit*,
   co-committed in one transaction — not event-sourced in the
   rebuild-from-log sense (§3, §7).
6. **Schema:** `quote_expires_at`, `payment_due_at`, event-supplied `occurred_at`
   (replay determinism), assignment-uniqueness index (INV-5 in the storage
   layer), `decision_trace` (§6).
7. **Rubinstein bargaining** downgraded from "formal frame" to vocabulary; the
   enforceable guarantee was always the guard, and now the doc says so (§8.4).
8. *(v0.2.1)* Two deep-dive companions added: **[model.md](model.md)** (the
   complete formal net, all proofs, the refinement argument to CockroachDB)
   and **[agentic.md](agentic.md)** (the full agent-layer contract). Writing
   the proofs surfaced one more liveness hole — a loaded truck that breaks
   down permanently had no exit from `EnRouteLoaded` — closed by back-porting
   `TransferCargo` and `AbortLoaded` (👤) from model.md §4.3 into §4.4 here.
9. *(v0.2.1)* **The vocabulary firewall** (internals Law 4). This document
   set is privileged; the `dakar` repository is not. Specification vocabulary
   (token, place, marking, net, transition, firing, guard, binding, saga…)
   never appears in code symbols, schema identifiers, code comments, or
   anything else in the repository, and code never references `dakar.doc`.
   Code and schema speak freight operations only: `Truck`, `Delivery`,
   `TruckStatus`/`DeliveryStatus`, `DispatchBoard`, `FreightDesk`, `Action`,
   `operations_journal`, `RefusalReason`. Traceability is one-way: the docs
   cite code (model.md §12); ArchUnit enforces the symbol ban. All schema and
   API prescriptions in this document now use the operational identifiers;
   the formal prose (this document's own use of CPN language) is unaffected —
   it never leaves this directory.

---

## 0. Reading guide

This document is organized so that each of the four governing constraints is
satisfied by a specific, load-bearing part of the architecture rather than by a
paragraph of intent:

| Constraint | Where it is discharged |
|---|---|
| Agentic memory is not an afterthought | §3 (memory-first invariant), §6 (schema), §7 (transition = transaction) |
| Grounded in a rigorous formal system | §4 (Colored Petri Net), §5 (invariants & verification) |
| It is an agent system under the standard definition | §8.1 (five-characteristics audit, objectives, autonomy ladder), §8.2 (tool architecture) |
| Product abstraction hides internal complexity | §2 (hexagonal boundary), §9 (marking-projection API) |

The central claim of the design, stated once here and defended throughout:

> **The Colored Petri Net marking, persisted in CockroachDB, *is* the system.**
> Compute is disposable. Agents are stateless, tool-using policies that choose
> among provably-legal moves. Memory is not a component the agents *use*; it is
> the substrate the agents *are*.

---

## 1. Domain in one page

Dakar operates a fleet of heavy trucks (smallest unit: 35-ton capacity) moving
freight between warehouses, manufacturers, and suppliers. The operational
lifecycle of a single delivery is a long-running saga measured in **days**, with
an expensive physical asset attached to it the entire time.

The canonical flow:

```
client requests delivery
  → find a suitable idle/repositionable truck (capacity ≥ tonnage)
  → negotiate price
  → navigate to loading location
  → load
  → navigate to destination
  → payment
  → offload
  → done
```

Two properties of this domain drive every architectural decision:

1. **The saga outlives its compute.** A truck is physically moving for hours or
   days while the software that manages it is ephemeral (Lambda invocations,
   redeploys, cold starts, region events). The unit of identity therefore
   *cannot* be a process. It must be a durable record.

2. **Autonomous writers contend over scarce physical resources.** When several
   delivery requests arrive at once, each spawns matching logic, and two will
   want the same truck. Double-booking a 35-tonner is a multi-thousand-dollar
   physical error. Concurrency correctness is a *safety* property here, not a
   performance nicety.

These two properties are exactly what the "agentic memory that never goes down"
thesis is about, made physical.

---

## 2. Architectural style: hexagonal, memory-centric

We use a **ports-and-adapters (hexagonal)** structure with an explicit,
formally-defined domain core. The organization sees a freight desk; it never
sees agents, sagas, embeddings, or transitions.

```
┌─────────────────────────────────────────────────────────────────────┐
│  PRODUCT SURFACE  (Quarkus / JAX-RS, module: dakar-quarkus, path: /v1) │
│  POST /v1/deliveries            "request a delivery"                  │
│  GET  /v1/deliveries/{id}       "quoted / loading / in transit / …"   │
│  POST /v1/quotes/{id}/accept    "accept the quote"                    │
│  staff console (recommendations, approve / override)                  │
├─────────────────────────────────────────────────────────────────────┤
│  DOMAIN CORE  (pure Java, module: dakar — no AWS, no AI, no SQL)      │
│    Colored Petri Net: Places, Transitions, Guards, Marking            │
│    — this layer is provable in isolation and has no I/O                │
├──────────────────────────────┬──────────────────────────────────────┤
│  AGENTS  (tool-using policies)│  MEMORY  (CockroachDB, system of record)│
│  Dispatcher / Negotiator /   │   marking tables (working memory)      │
│  Saga driver                 │   event log + decision traces (episodic)│
│  Lambda + Bedrock tool use,  │   embeddings / vector index (episodic) │
│  stateless, crash-only       │   client & lane stats (semantic)       │
│                              │   policies & guardrails (procedural)   │
├──────────────────────────────┴──────────────────────────────────────┤
│  OPERATOR SURFACE (internal only)                                    │
│   CockroachDB MCP Server (self-hosted, read-only, cert-authenticated) │
│   ccloud CLI (roadmap — needs a CockroachDB Cloud cluster)           │
└─────────────────────────────────────────────────────────────────────┘
```

**Dependency rule.** Arrows point inward. The domain core depends on nothing.
Agents depend on the core and on memory ports. The product surface depends on
the core and on a query port. Nothing in the core knows CockroachDB, Bedrock,
or Lambda exist. This is what makes the formal model (§4) testable without any
infrastructure and what keeps the product abstraction (§9) honest.

The agents' deliberation loops (§8.2) live entirely in the adapter layer — the
domain core stays pure and single-shot; the shell may converse with a model as
many (bounded) turns as it likes before entering the core exactly once per
intended firing.

---

## 3. Memory-first: the three invariants that make it non-optional

To guarantee memory is not an afterthought, the design makes it *impossible for
the system to function without its memory being authoritative*. Three
invariants enforce this:

**MI-1 — No agent owns state.** An agent is a stateless policy
`decide(context) → intended_transition`. All context is read from CockroachDB at
the start of a decision episode; all effects are committed to CockroachDB at the
end. If any agent process retains information across invocations that is not in
the database, that is a defect, and it is detectable by chaos testing (kill the
process mid-decision; the system must be indistinguishable from one where the
process was never scheduled).

**MI-2 — Every state change is event-logged, atomically, with a replay
audit.** Every *net-state* change — places, assignments, prices, flags — is a
firing that writes an immutable `operations_journal` row *in the same
serializable transaction* as the marking change (§7), so the log can never
disagree with the marking. (Telemetry attributes — truck position, driver
hours — are environment-owned *measurements*, updated by the telemetry
adapter and excluded from event sourcing and replay by definition; see
model.md §5.3 for the formal ownership split.) A scheduled replay audit folds the log from the
initial marking and compares it byte-for-byte with the live marking; divergence
is a severity-1 defect. (Precisely: this is *co-committed event logging with
replay verification*, not event sourcing — the marking is written directly, not
rebuilt from the log. The v0.1 draft used the stronger term loosely; the
guarantee that matters — the episodic memory the AI relies on is the same
history the business runs on — holds either way.)

**MI-3 — Memory has a typed taxonomy, mapped to schema up front.** We commit to
four memory kinds, each with named tables (§6), before writing agent logic:

| Memory kind | Contents | Backing |
|---|---|---|
| **Working** | live marking: truck & delivery token states | `truck`, `delivery` |
| **Episodic** | firings, negotiation transcripts, disruption records, agent decision traces | `operations_journal`, `negotiation_episode`, `decision_trace` + vector index |
| **Semantic** | distilled client profiles, lane statistics | `client_profile`, `lane_stat` |
| **Procedural** | versioned guardrails: price floors, capacity rules, autonomy thresholds, loop bounds | `policy` (data, not code) |

Storing procedural knowledge (guardrails) as *versioned data* rather than code
is deliberate: it lets a guard change be audited, rolled back, and reasoned
about as a memory update, and it keeps the guard enforcement in the transaction
layer where it cannot be bypassed by a confused agent.

---

## 4. Formal grounding: a Colored Petri Net (CPN)

The fleet-and-delivery lifecycle is modeled as a **Colored Petri Net**. Petri
nets are the standard formalism for resource flow and concurrency in logistics;
the coloring carries the business data. This is the rigorous computational
framework the system is grounded in.

> This section is the working summary. **[model.md](model.md)** is the
> complete formal authority: full color/marking definitions, the
> administrative sub-net, environment semantics (telemetry vs. net state),
> all invariant proofs, the liveness proof, the policy-independence theorem,
> and the refinement argument down to CockroachDB.

### 4.1 Definition

A CPN is a tuple `N = (P, T, A, Σ, C, G, E, M₀)` where:

- **P** — finite set of *places* (states).
- **T** — finite set of *transitions* (events), `P ∩ T = ∅`.
- **A ⊆ (P×T) ∪ (T×P)** — directed arcs.
- **Σ** — set of *color sets* (data types carried by tokens).
- **C: P → Σ** — assigns each place the color set of tokens it may hold.
- **G: T → Bool** — *guard* expression per transition (must hold to fire).
- **E: A → expression** — arc inscriptions (how tokens are consumed/produced).
- **M₀** — the initial *marking* (multiset of colored tokens per place).

A **marking** M assigns to each place a multiset of tokens. A transition `t` is
**enabled** in M for a binding `b` (an assignment of values to the variables on
`t`'s input arcs) iff every input place holds the required tokens **and** `G(t)`
evaluates true under `b`. Firing consumes input tokens and produces output
tokens, yielding a new marking M′. We write `M —(t,b)→ M′`.

### 4.2 Color sets (token data)

The net is closed over exactly two token kinds. Assignment is a *color field*
on the truck token, not a separate token — this matches the relational encoding
(§6.1) and the Java core exactly, so the formal object and the persisted object
are the same shape. *(v0.2: the v0.1 `Assignment` token had no place in the
place set; the net as drawn could not be loaded by a model checker.)*

```
Truck    = record { id, capacityTons, position:GeoPoint, driverHoursRemaining,
                    maintenanceDueAt, assignedDelivery:DeliveryId? }
Delivery = record { id, clientId, tonnage, origin:GeoPoint, dest:GeoPoint,
                    cargoClass, floorPrice:Money, agreedPrice:Money?,
                    paymentConfirmed:Bool, quoteExpiresAt:Instant?,
                    paymentDueAt:Instant?, deadlineAt }
```

`quoteExpiresAt` is stamped by `Quote` (now + policy validity window) and
`paymentDueAt` by `ArriveDest` (now + policy payment terms); both feed the
timeout transitions that make LIVE-1 provable (§4.4, §5).

### 4.3 Places

**Truck places** (a truck token is in exactly one at all times — see INV-1):

| Place | Meaning |
|---|---|
| `Garage` | idle at depot |
| `EnRouteEmpty` | repositioning without cargo (deadhead) |
| `Loading` | at origin, loading |
| `EnRouteLoaded` | carrying cargo to destination |
| `Offloading` | at destination, offloading |
| `Maintenance` | out of service |

**Delivery places:**

| Place | Meaning |
|---|---|
| `Requested` | client submitted, unmatched |
| `Matched` | a truck is claimed for it |
| `Quoted` | a price offer is on the table |
| `Agreed` | client accepted the quote |
| `InTransit` | cargo loaded and moving |
| `AwaitingPayment` | at destination, payment pending |
| `Disputed` | payment overdue; staff resolution path *(v0.2)* |
| `Completed` | offloaded and settled, terminal |
| `Cancelled` | terminal failure/abort/write-off |

### 4.4 Transitions and guards

The guards are where the business rules live. **An agent can never fire a
disabled transition; a guard that is false makes an intended move simply not
exist.** This is the soundness layer.

Legend: ⏲ = *timeout transition* — enabled by the passage of time alone, fired
by the scheduler, no external party required. 👤 = *staff-authorized* — fired
only from the staff console, never by an agent.

| Transition | Consumes | Produces | Guard `G` |
|---|---|---|---|
| `ClaimTruck` | Truck∈{Garage,EnRouteEmpty}, Delivery∈Requested | Truck (same place, `assignedDelivery := d`), Delivery→Matched | `capacity(truck) ≥ tonnage(d)` ∧ `driverHoursRemaining ≥ estDriveHours` ∧ `assignedDelivery(truck) = ∅` |
| `Quote` | Delivery∈Matched | Delivery→Quoted (`agreedPrice := offer`, `quoteExpiresAt := now + validity(policy)`) | `offer ≥ floorPrice(d)` |
| `AcceptQuote` | Delivery∈Quoted | Delivery→Agreed | client acceptance signal ∧ `now < quoteExpiresAt` |
| `QuoteExpired` ⏲ | Delivery∈Quoted | Delivery→Cancelled; truck released (`assignedDelivery := ∅`) | `now ≥ quoteExpiresAt` |
| `BeginReposition` | Truck∈Garage | Truck→EnRouteEmpty | `assignedDelivery(truck) = d` ∧ `position(truck) ≠ origin(d)` |
| `ArriveOrigin` | Truck∈{EnRouteEmpty,Garage}, Delivery∈Agreed | Truck→Loading | truck at origin |
| `Load` | Truck∈Loading, Delivery∈Agreed | Truck→EnRouteLoaded, Delivery→InTransit | `loadedTons ≤ capacity(truck)` |
| `ArriveDest` | Truck∈EnRouteLoaded, Delivery∈InTransit | Truck→Offloading, Delivery→AwaitingPayment (`paymentDueAt := now + terms(policy)`) | truck at destination |
| `ConfirmPayment` | Delivery∈{AwaitingPayment, Disputed} | same place, `paymentConfirmed := true` | payment settled (external signal) |
| `Offload` | Truck∈Offloading, Delivery∈{AwaitingPayment, Disputed} | Truck→Garage/EnRouteEmpty (`assignedDelivery := ∅`), Delivery→Completed | **`paymentConfirmed(d)`** |
| `PaymentOverdue` ⏲ | Delivery∈AwaitingPayment | Delivery→Disputed | `now ≥ paymentDueAt` ∧ `¬paymentConfirmed(d)` |
| `ReleaseTruck` 👤 | Truck∈Offloading, Delivery∈Disputed | Truck→Garage/EnRouteEmpty (`assignedDelivery := ∅`); cargo to bonded storage; Delivery stays Disputed | `¬paymentConfirmed(d)` ∧ staff authorization |
| `TransferCargo` 👤 | Truck k₁∈EnRouteLoaded (assigned to d), Truck k₂∈{Garage,EnRouteEmpty}, Delivery∈InTransit | k₁→Maintenance (`assignedDelivery := ∅`); k₂→EnRouteLoaded (`assignedDelivery := d`); Delivery stays InTransit | `assignedDelivery(k₂) = ∅` ∧ `capacity(k₂) ≥ tonnage(d)` ∧ `driverHoursRemaining(k₂) ≥ estDriveHours` ∧ staff authorization *(rescue — v0.2.1)* |
| `AbortLoaded` 👤 | Truck∈EnRouteLoaded (assigned to d), Delivery∈InTransit | Truck→Garage/Maintenance (`assignedDelivery := ∅`); Delivery→Cancelled | staff authorization (total loss / post-load abort; insurance settles outside the net) *(v0.2.1)* |
| `SettleDispute` 👤 | Delivery∈Disputed | Delivery→Completed | **`paymentConfirmed(d)`** ∧ no truck attached (cargo released from storage) |
| `WriteOff` 👤 | Delivery∈Disputed | Delivery→Cancelled | no truck attached ∧ staff authorization (cargo disposition recorded) |
| `RecordDisruption` | any non-terminal Delivery | same place; disruption appended to episodic log | — |
| `Cancel` | Delivery∈{Requested,Matched,Quoted,Agreed} | Delivery→Cancelled; truck released | not yet loaded |

Three guards that encode real Dakar policy directly into the formalism:

- `Offload` and `SettleDispute` — the **only two arcs into `Completed`** — both
  require `paymentConfirmed`. "Payment is made, *then* offload" is not a
  convention enforced by discipline; it is a structural property of the net,
  and §5 (INV-2) shows no reachable marking violates it.
- `Quote` requires `offer ≥ floorPrice`. The Negotiator agent may *propose* any
  number, but a below-floor offer yields a disabled transition. The floor is
  procedural memory (`policy` table), enforced in the firing transaction.
- The two ⏲ timeout transitions guarantee that **no client behavior is required
  for the system to terminate**: a client who never accepts routes to
  `Cancelled` via `QuoteExpired`; a client who never pays routes to `Disputed`
  via `PaymentOverdue`, from which staff can always free the truck
  (`ReleaseTruck`) and close the delivery (`SettleDispute`/`WriteOff`).
  Likewise **no physics is required**: a loaded truck that fails permanently
  is rescued (`TransferCargo`) or aborted (`AbortLoaded`) by staff.
  *(v0.2: in v0.1, a delivery stuck in `AwaitingPayment` stranded its truck in
  `Offloading` with no enabled exit — a liveness hole the formalism existed to
  catch and the revision closes. v0.2.1 closed the analogous `EnRouteLoaded`
  breakdown hole found while writing model.md's proofs.)*

### 4.5 The role of agents in the net

LLM-backed agents are **tool-using policies over enabled transitions** (fully
specified in §8): the transitions an agent is authorized to fire are exposed to
the model as *action tools*; guarded execution makes an illegal tool call
physically unable to commit. This is the crisp answer to *what makes agentic
systems different from traditional apps*: **the LLM supplies deliberation and
choice among provably-legal moves; the formal net supplies soundness.**
Non-determinism is confined to policy; safety is a property of the structure.

---

## 5. What the formalism buys us: invariants and liveness

Because the model is a Petri net, correctness properties are **claims about
reachable markings**, checkable by place-invariant argument and — against the
actual Java implementation — by the property-based invariant suite that runs on
every build. These are the properties we assert and verify:

- **INV-1 (single-location).** For every truck token `k`, across all reachable
  markings, `k` appears in exactly one truck place. *Argument:* every
  truck-moving transition consumes `k` from exactly one place and produces
  exactly one `k` — the truck arcs form a place-invariant summing to 1.
  *(Note: INV-1 alone does not forbid double-booking — a truck in exactly one
  place could still be referenced by two deliveries if assignment were
  unconstrained. That property is INV-5. The v0.1 draft conflated the two.)*

- **INV-2 (payment-before-completion).** No reachable marking has a delivery in
  `Completed` without `paymentConfirmed`. *Argument:* the only arcs into
  `Completed` are `Offload` and `SettleDispute`, and both guards require
  `paymentConfirmed`.

- **INV-3 (truck conservation).** The number of truck tokens is invariant; the
  system cannot create or lose a truck. Trucks enter/leave only via explicit
  `RegisterTruck` / `Decommission` administrative transitions.

- **INV-4 (capacity soundness).** No `Load` fires with `loadedTons >
  capacity(truck)`; combined with the `ClaimTruck` guard, no delivery is ever
  matched to an under-capacity truck.

- **INV-5 (assignment injectivity — no double-booking).** *(v0.2)* In every
  reachable marking, (a) each truck's `assignedDelivery` references at most one
  delivery, and (b) no delivery is referenced by more than one truck.
  *Argument:* `ClaimTruck` is the only transition that sets `assignedDelivery`,
  its guard requires the truck unassigned and the delivery in `Requested`
  (which it leaves atomically), and the only transitions that clear assignment
  (`Offload`, `ReleaseTruck`, `Cancel`, `QuoteExpired`) move the delivery out
  of active places. Enforced three ways, deliberately redundant: the guard
  (domain), `SERIALIZABLE` isolation (transaction), and a partial `UNIQUE`
  index on `truck.assigned_delivery` (storage — §6.1), so even a buggy
  adapter cannot persist a violation. **No double-booking is this invariant**,
  not INV-1.

- **LIVE-1 (progress, with explicit fairness assumptions).** *(v0.2 restated)*
  Under two assumptions — **F1:** time advances and the scheduler eventually
  fires enabled ⏲ timeout transitions; **F2:** staff eventually act on
  stalled deliveries — every delivery token reaches `Completed` or
  `Cancelled`, and every truck token returns to
  `Garage`/`EnRouteEmpty`/`Maintenance` unassigned, from any reachable marking
  (full proof: model.md §7). **No client action — and no physics — is
  required for termination**:
  client signals (`AcceptQuote`, `ConfirmPayment`) can only *advance* a
  delivery toward `Completed`; their absence routes it to a terminal place via
  `QuoteExpired` or `PaymentOverdue` → staff resolution. A liveness claim
  without stated fairness assumptions is not a claim; these are ours, and both
  are internal to Dakar (a scheduler and a staffed desk), not hopes about
  client behavior.

These are exactly the guarantees a heuristic workflow cannot make. They are also
the guarantees a physical logistics business most needs.

**Verification tooling (what exists vs. what is planned).** The net is
specified once (the place/transition/guard tables above; normative) and
verified by the **property-based invariant suite** (jqwik) in the Java domain
core: random legal firing sequences from random initial markings, INV-1…INV-5
checked after every step, on every build (backlog P3-3). This is the shipped,
executable verification — it runs against the *actual implementation*, which an
external spec cannot. A small **TLA+ specification of the contended
`ClaimTruck`/`Offload` protocol**, model-checked in CI, is on the roadmap
(backlog, Post-MVP) as a complementary check of the concurrent-firing protocol
specifically; it does not exist yet, and this document will not claim it does
until it merges.

---

## 6. CockroachDB as the memory layer (schema)

CockroachDB is the system of record for the marking, the event log, and all four
memory kinds (§3). It is chosen for the three properties this domain demands:
**serializable isolation** (transition firing is atomic under contention),
**survivability** (multi-region, no maintenance windows — the "memory never goes
down" requirement), and **native vector indexing** (episodic recall in the same
transactional domain as the operational data, so there is no consistency gap
between what the agent remembers and what the business committed).

### 6.1 Working memory — the marking

```sql
-- One row per physical truck; `status` is its operational status.
-- (Identifiers are operational vocabulary only — the migrations live in the
--  public repo; the spec↔schema mapping is model.md §12. Law 4, internals §3.)
CREATE TABLE truck (
  truck_id            UUID PRIMARY KEY,
  status              STRING NOT NULL,          -- GARAGE | EN_ROUTE_EMPTY | ...
  capacity_tons       DECIMAL NOT NULL,
  position            GEOGRAPHY(POINT) NOT NULL,
  driver_hours_rem    DECIMAL NOT NULL,
  maintenance_due_at  TIMESTAMPTZ,
  assigned_delivery   UUID,                     -- null unless claimed
  version             INT8 NOT NULL DEFAULT 1,  -- optimistic concurrency
  updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- INV-5 in the storage layer: a delivery can be assigned to at most one truck,
-- even if a (buggy) adapter bypasses the domain rule check.
CREATE UNIQUE INDEX truck_single_assignment
  ON truck (assigned_delivery)
  WHERE assigned_delivery IS NOT NULL;

CREATE TABLE delivery (
  delivery_id         UUID PRIMARY KEY,
  status              STRING NOT NULL,          -- REQUESTED | MATCHED | ...
  client_id           UUID NOT NULL,
  tonnage             DECIMAL NOT NULL,
  origin              GEOGRAPHY(POINT) NOT NULL,
  dest                GEOGRAPHY(POINT) NOT NULL,
  cargo_class         STRING NOT NULL,
  floor_price         DECIMAL NOT NULL,         -- from cost model (procedural)
  agreed_price        DECIMAL,                  -- set by Quote
  payment_confirmed   BOOL NOT NULL DEFAULT false,
  quote_expires_at    TIMESTAMPTZ,              -- set by Quote      (v0.2)
  payment_due_at      TIMESTAMPTZ,              -- set by ArriveDest (v0.2)
  deadline_at         TIMESTAMPTZ NOT NULL,
  version             INT8 NOT NULL DEFAULT 1,
  updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Product-API idempotency ledger (v0.2.2, shipped with the P2 skeleton): one
-- row per accepted create request, so a retried or raced request key returns
-- the delivery the first request created instead of creating twice.
CREATE TABLE api_request (
  request_key   STRING PRIMARY KEY,
  delivery_id   UUID NOT NULL,
  created_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### 6.2 Episodic memory — event log, decision traces, negotiation transcripts

```sql
-- Append-only journal of every executed action. This IS the episodic memory.
CREATE TABLE operations_journal (
  entry_id      UUID PRIMARY KEY,               -- supplied by the domain event (IdSource)
  action        STRING NOT NULL,                -- ClaimTruck | Quote | ...
  payload       JSONB NOT NULL,                 -- versioned serialization of the typed
                                                -- Action arguments + Evidence (the facts
                                                -- the rules consulted — replay closure)
  actor         STRING NOT NULL,                -- dispatcher | negotiator | coordinator | staff
  delivery_id   UUID,
  truck_id      UUID,
  occurred_at   TIMESTAMPTZ NOT NULL,           -- = snapshot.now() from the event: replay-deterministic (v0.2)
  recorded_at   TIMESTAMPTZ NOT NULL DEFAULT now(),  -- ingest time, observability only
  INDEX (delivery_id, occurred_at),
  INDEX (truck_id, occurred_at)
);
```

*(v0.2: `occurred_at` is supplied by the domain event — internals §10 — not
defaulted to `now()`. A database-clock default would make `Journal.replay`
equality unachievable; the ingest timestamp lives separately in `recorded_at`.)*

```sql
-- Agent deliberation, persisted: the ordered tool calls, their results, and the
-- model's rationale for one decision episode. Episodic memory of *how* the
-- agent decided, not just what it did; feeds audit and future retrieval.
CREATE TABLE decision_trace (
  trace_id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  actor            STRING NOT NULL,             -- dispatcher | negotiator | coordinator
  journal_entry_id UUID,                        -- null if the episode ended without an action
  snapshot_ts      TIMESTAMPTZ NOT NULL,        -- the AS OF SYSTEM TIME all perception used (§8.2)
  trace            JSONB NOT NULL,              -- ordered tool calls + results + rationale
  created_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
  INDEX (journal_entry_id)
);

-- Negotiation episodes with embeddings for semantic recall of "deals like this".
-- (v0.2.3: shipped as V3 — outcome gains 'open' for quote-time recording,
--  won/lost settle on acceptance/loss; embeddings are unit-normalized, so
--  cosine and L2 rankings agree and the plain vector index serves both.)
CREATE TABLE negotiation_episode (
  episode_id    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  delivery_id   UUID NOT NULL,
  client_id     UUID NOT NULL,
  lane_key      STRING NOT NULL,                -- origin→dest half-degree bucket
  cargo_class   STRING NOT NULL,
  tonnage       DECIMAL NOT NULL,
  opening_offer DECIMAL,
  final_price   DECIMAL,
  outcome       STRING NOT NULL DEFAULT 'open', -- open | won | lost
  transcript    STRING NOT NULL,
  embedding     VECTOR(1024) NOT NULL,          -- Titan Text Embeddings V2 contract
  created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
  INDEX (delivery_id),
  INDEX (lane_key, cargo_class)
);

-- Distributed vector index for episodic retrieval at scale
-- (requires CockroachDB >= v25.2 with feature.vector_index.enabled).
CREATE VECTOR INDEX negotiation_episode_embedding
  ON negotiation_episode (embedding);
```

The vector index is used for **episodic memory over business episodes**, not
document RAG: before making an offer, the Negotiator retrieves the *k* most
similar past negotiations (same client tendencies, similar lane/cargo/season) —
via its own perception tool (§8.2) — as context. The embedding and the committed
relational fact are written in the same transaction (MI-2), so there is never a
moment where the agent "remembers" a deal the ledger doesn't have.

### 6.3 Semantic and procedural memory

```sql
-- (v0.2.3: shipped V3 shapes — distilled by the scheduled aggregation job.
--  Richer facets from the v0.1 sketch (price sensitivity, demand index,
--  seasonality, profile embeddings) join when an agent consumes them.)
CREATE TABLE client_profile (              -- semantic: distilled from episodes
  client_id        UUID PRIMARY KEY,
  win_rate         DECIMAL,
  avg_final_price  DECIMAL,
  episode_count    INT8 NOT NULL DEFAULT 0,
  updated_at       TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE lane_stat (                   -- semantic: pricing by lane
  lane_key             STRING PRIMARY KEY,
  median_price_per_ton DECIMAL,
  episode_count        INT8 NOT NULL DEFAULT 0,
  updated_at           TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE policy (                      -- procedural: versioned guardrails
  policy_key    STRING NOT NULL,           -- e.g. 'floor_price_formula',
  version       INT8 NOT NULL,             --      'quote_validity', 'payment_terms',
  definition    JSONB NOT NULL,            --      'max_perception_calls', 'saga_escalation_threshold'
  active        BOOL NOT NULL DEFAULT true,
  created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
  PRIMARY KEY (policy_key, version)
);
```

The agents' operating limits — deliberation-loop bounds, action-attempt caps,
the Saga driver's escalation threshold (§8) — are `policy` rows, not constants
in code: procedural memory governs the agents' *own* behavior the same way it
governs prices.

---

## 7. Transition firing = one serializable transaction

This is the mechanical heart of the design and the point where the formalism,
the memory-first invariant, and the concurrency-safety requirement all coincide.
**Firing a CPN transition is exactly one CockroachDB `SERIALIZABLE`
transaction.**

Shape of a firing (illustrated with `ClaimTruck`):

```
BEGIN;  -- SERIALIZABLE (CockroachDB default)

  -- 1. Read the records the action touches (working memory).
  SELECT * FROM truck    WHERE truck_id    = $truck    FOR UPDATE;
  SELECT * FROM delivery WHERE delivery_id = $delivery FOR UPDATE;

  -- 2. Re-check the ClaimTruck business rules in the domain core:
  --    capacity ≥ tonnage ∧ driverHoursRem ≥ estDriveHours ∧ unassigned.
  --    If refused → ROLLBACK (nothing to execute).

  -- 3. Apply the state writes. The expectedVersion values come from the
  --    ActionPlan (computed from an earlier snapshot), which is why the
  --    version predicate is load-bearing, not decorative.
  UPDATE delivery
     SET status='MATCHED', version=version+1, updated_at=now()
   WHERE delivery_id=$delivery AND version=$expectedVersion;
  UPDATE truck
     SET assigned_delivery=$delivery, version=version+1
   WHERE truck_id=$truck AND version=$expectedVersion;

  -- 4. Append the journal entry (MI-2). occurred_at comes from the event,
  --    not now().
  INSERT INTO operations_journal(entry_id, action, payload, actor,
                                 delivery_id, truck_id, occurred_at)
       VALUES ($entryId, 'ClaimTruck', $payload, 'dispatcher',
               $delivery, $truck, $occurredAt);

COMMIT;
```

Why this is correct and production-grade:

- **Contention is resolved by the database, per net semantics.** Two Dispatcher
  invocations racing to `ClaimTruck` the same truck token cannot both commit
  under `SERIALIZABLE`; one gets a retryable `40001` (serialization failure).
  There is no separate lock service to build or to fail.
- **Retry is a first-class agent behavior.** On `40001`, the losing invocation
  re-reads the marking and re-plans (the truck it wanted is now taken → choose
  another enabled binding). This retry loop is standard CockroachDB client
  practice and is implemented once in the memory port, not per agent.
- **Atomic memory update.** The token change *and* its episodic-log entry commit
  together, so the log can never disagree with the marking (MI-2).
- **Guard enforcement is server-side of the agent.** The floor-price and
  capacity guards are evaluated inside the firing transaction against `policy`
  and token data; an agent that "hallucinates" a below-floor quote produces a
  transaction that rolls back, not a bad commitment.

Read paths that don't need the freshest marking (dashboards, the staff console's
fleet map, analytics) use CockroachDB **follower reads**
(`AS OF SYSTEM TIME follower_read_timestamp()`) to stay fast and off the hot
path. Agent perception reads use `AS OF SYSTEM TIME` pinned to the decision
episode's snapshot timestamp (§8.2) — a different mechanism for a different
guarantee: intra-decision consistency rather than latency.

---

## 8. The agent portfolio

Agents are stateless AWS Lambda functions driving Amazon Bedrock models through
**tool use**. None holds state (MI-1). This section first establishes *that*
they are agents — against the standard definition, honestly — and then
specifies the machinery.

> This section is the working summary. **[agentic.md](agentic.md)** is the
> agent-layer authority: decision-episode anatomy with crash-point analysis,
> the complete tool catalogs and error taxonomy, per-agent specifications
> with episode walkthroughs, autonomy governance and gate-removal criteria,
> the coordination model, the prompt-injection threat model, and the agent
> testing strategy.

### 8.1 Why these are agents: the five-characteristics audit

An AI agent perceives its environment, reasons over what it perceives,
autonomously takes actions toward explicit goals, and maintains state. The
audit, with the honest boundary of each claim stated next to it:

| Characteristic | Realization in Dakar | Honest boundary |
|---|---|---|
| **Perception** | Marking snapshot + EventBridge signals as push percepts; **perception tools** (§8.2) as pull percepts the agent invokes itself | The percept space is bounded by the tool inventory — deliberate, so perception is auditable |
| **Internal state** | Externalized entirely to CockroachDB: marking (working), firings + decision traces (episodic), profiles/stats (semantic), guardrails (procedural) | No in-process state, *by design* (MI-1) — statelessness of the process is what makes the state durable |
| **Reasoning** | Bounded multi-turn deliberation: the model retrieves via perception tools, then proposes an action tool call; the trace is persisted (`decision_trace`) | Loop bounds (policy data) trade deliberation depth for cost and latency |
| **Action** | Action tools = net transitions, executed as serializable firings. The marking is the operational system of record staff and drivers work from, so a committed firing changes the world Dakar operates in | Physical actuation (driving, loading) is mediated by humans reading the same marking; direct telematics integration is roadmap |
| **Autonomy** | Ladder below — Dispatcher fully autonomous; Saga driver autonomous within a policy threshold; Negotiator human-gated | The Negotiator gate is a deliberate governor on client-facing financial commitments, not a limitation of the machinery |
| **Goal-direction** | Explicit objectives, stated below, one per agent | Objectives are optimized heuristically by the LLM policy; the *constraints* are what is provably enforced |

**Explicit objectives** (v0.2 — previously implicit):

- **Dispatcher:** minimize the expected cost of serving a delivery —
  repositioning kilometers + driver-hours consumption + forecast deadhead on
  the return lane — over the feasible (guard-enabled) truck–delivery pairs;
  secondary objective: maximize backhaul potential at the destination
  (`lane_stat.demand_index`).
- **Negotiator:** maximize expected margin, `P(win | offer, client, lane) ×
  (offer − floorPrice)`, with `offer ∈ [floorPrice, ceiling]`; the win
  probability is informed by retrieved episodes and the client profile.
- **Saga driver:** keep every active delivery progressing toward a terminal
  place; on disruption, minimize recovery cost and delay across the options it
  is authorized to take.

**Autonomy ladder:**

| Agent | Autonomy | Gate and rationale |
|---|---|---|
| Dispatcher | **Fully autonomous** — fires `ClaimTruck` without human approval | Pre-load claims are reversible (`Cancel`); the guard + INV-5 bound the blast radius |
| Saga driver | **Autonomous within policy** — fires physical transitions on signals; chooses disruption responses itself | Escalates to staff when a response's estimated cost exceeds `saga_escalation_threshold` (procedural memory) |
| Negotiator | **Draft + human approval** | Client-facing financial commitment; the gate is removed post-MVP only when rule-refusal and policy-violation rates are demonstrably boring (backlog, Post-MVP) |

At least one agent is fully autonomous end-to-end from the first release —
otherwise this would be a recommender system with an agent-shaped vocabulary.

### 8.2 The tool architecture: transitions are the tools

*(v0.2 — this section replaces v0.1's single-shot structured-output design.)*

The question "what tool is the agent calling" has a precise answer: **the tool
inventory of each agent is (a) the set of net transitions its role is
authorized to fire — the action tools — and (b) a small set of read-only
perception tools.** Both are exposed through the Bedrock tool-use (Converse)
API; the deliberation loop lives entirely in the agent's adapter (shell), so
the domain core's purity laws are untouched.

**Action tools.** One tool per authorized transition, its JSON schema derived
from the typed transition record (`claim_truck(truck_id, delivery_id)`,
`quote(delivery_id, offer)`, …). Executing an action tool call means: the shell
converts the call into a typed `Transition`, asks the domain to plan it
(`FreightDesk.plan` — which re-evaluates the guard), and executes the resulting
`ActionPlan` as the §7 transaction. Consequences:

- A call to a **disabled transition** returns a typed tool error carrying the
  `RefusalReason`s to the model, which may re-deliberate; after
  `max_action_attempts` (procedural memory) the episode ends with a typed
  no-action outcome. The error path is a *first-class percept*, not an
  exception.
- A call to an **unknown tool or malformed arguments** is rejected outright —
  never repaired by guessing (internals §8.3).
- A `40001` serialization loss re-enters the loop with a fresh snapshot: the
  world changed, re-perceive, re-decide.

The soundness story is now mechanical end-to-end: *the agent's tools are Petri
net transitions; the database is the tool executor; an illegal tool call cannot
commit.* This is "the LLM proposes, the formalism disposes" visible in the API
traces, not just in this document.

**Perception tools.** Read-only, per-role: `list_claim_candidates()`,
`search_similar_negotiations(query)` (vector index), `get_client_profile(id)`,
`get_lane_stat(lane)`, `get_route_estimate(from, to)`. Two rules make them
safe and reproducible:

1. **Snapshot consistency.** All perception reads within one decision episode
   are served `AS OF SYSTEM TIME` pinned to the episode's snapshot timestamp —
   one decision observes one consistent world, a guarantee CockroachDB gives us
   for free and most agent frameworks cannot make at all.
2. **Bounded and audited.** At most `max_perception_calls` (procedural memory)
   per episode; every call and result is recorded in the `decision_trace` row
   (§6.2), committed with the firing when there is one.

**Symmetry with access control.** Each agent's tool inventory is exactly its
database principal's grant set (§11): the Negotiator is not *offered* a
`claim_truck` tool, and its service account could not execute one anyway.
Capability and permission are the same list read twice.

**Where the loop lives.** The domain port (`DispatchPolicy`,
`NegotiationPolicy` — internals §8.2) is still a pure-looking function from
context to choice; the Bedrock adapter *implements* that port as a bounded
tool-use conversation. The domain is entered exactly once per intended firing;
the deliberation that precedes it is a shell concern. Purity Laws 1–2 hold
untouched.

### 8.3 Dispatcher (fleet matching)

Replaces the heuristic "find an idle truck with enough capacity." Reasons over
the fleet's **near-future** state, not just its current one: a truck that is
`EnRouteEmpty` 50 km from origin can beat an idle `Garage` truck 200 km away.

- **Perception:** `list_claim_candidates()` — every guard-enabled truck→delivery
  pair, annotated with repositioning distance, driver-hours slack, maintenance
  window; `get_lane_stat` for backhaul potential at the destination.
- **Problem frame:** min-cost assignment over feasible pairs (objective in
  §8.1). The LLM chooses from the enumerated candidate set; a candidate outside
  the set is rejected by the port validation (internals §8.3). Post-MVP, a
  solver computes the assignment and the LLM ranks/annotates it.
- **Action:** `claim_truck` — the §7 serializable claim; on `40001`,
  re-perceive and re-plan.

### 8.4 Negotiator (pricing)

Chooses the `offer` on `Quote`, grounded in episodic + semantic memory and
bounded by a guard.

- **Perception:** `search_similar_negotiations` over the vector index for the
  *k* most similar past deals — retrieval the agent *directs itself*, not a
  context blob pre-assembled for it; `get_client_profile`, `get_lane_stat` for
  structured priors.
- **Pricing frame.** The `floor_price` (from the cost model: fuel, distance,
  driver hours, repositioning) is the reservation price and defines the lower
  edge of the zone of possible agreement; a market/`lane_stat` estimate defines
  the ceiling. The concession schedule is the LLM policy. *(v0.2: v0.1 dressed
  this in Rubinstein alternating-offers language; nothing was computed from
  that model, so the claim is withdrawn. The enforceable guarantee is and
  always was the guard: `offer ≥ floorPrice`, evaluated inside the firing
  transaction. Bargaining vocabulary — reservation price, ZOPA — is retained
  as vocabulary.)*
- **Action & safety:** `quote(delivery_id, offer)`. Below-floor offers are
  disabled transitions (§4.4). The MVP runs **agent-proposed, human-approved**:
  the agent drafts an offer *with its decision trace attached* — retrieved
  episodes, priors, rationale — and a staff member confirms. This is more
  production-credible than autonomous client-facing negotiation and keeps a
  human in the loop for the first release.

### 8.5 Saga driver (lifecycle)

One logical driver per delivery, but **not** a long-lived process — it is woken
by external signals (Amazon EventBridge → Lambda) such as "arrived at origin,"
"loaded," "payment settled," or an injected disruption. On each wake it reads
the delivery's full context from CockroachDB and fires the appropriate physical
transition.

The disruption path is a genuine decision, not a hardcoded reaction *(v0.2)*:
on a breakdown or delay, the driver deliberates over its options — **reassign**
(fire `Cancel` + re-invoke the Dispatcher), **wait** (tolerable delay, with
rationale), or **escalate** to staff — using route estimates and the delivery's
deadline slack as percepts. Responses whose estimated cost exceeds
`saga_escalation_threshold` (procedural memory) are always escalated. Every
choice lands in `decision_trace`; `RecordDisruption` puts the episode in the
same episodic memory future disruptions will be resolved against.

### 8.6 Roadmap agents (not in MVP)

- **Backhaul Hunter** — before/after offload, searches predicted return-lane
  demand (`lane_stat` + vector recall of past return loads) to cut deadhead
  kilometers. High economic impact; reuses the exact memory and tool machinery.
  "We cut empty kilometers by X%" is the one number that lands with both
  clients and reviewers.
- **Disruption agent** — embeds past disruption→resolution episodes for
  semantic recall when a new exception hits. Same vector layer, different
  corpus; the Saga driver's `decision_trace` corpus is its training ground.
- **Client-facing actuators** — send quote, request payment link, notify
  driver: real outward actions (SES/SNS adapters) that extend the action
  boundary beyond the marking. Named here so "action" has a growth path past
  the database row.

All are deliberately out of MVP scope; their presence in the schema (the
`operations_journal`, `decision_trace`, and vector layers already carry the
needed data) is evidence the memory design generalizes.

---

## 9. Product abstraction: the org sees a freight desk

The external vocabulary is **business states only**. Client- and staff-facing
statuses are a *projection of the net's marking* — which the formalism gives us
for free, because the observable states are precisely abstractions of markings.

| Internal (marking) | Product-facing status |
|---|---|
| Delivery ∈ Requested | "Received — finding a truck" |
| Delivery ∈ Matched/Quoted | "Preparing your quote" |
| Delivery ∈ Agreed, Truck ∈ EnRouteEmpty/Loading | "Driver on the way to pick up" |
| Delivery ∈ Agreed, Truck not yet moving | "Scheduled — assigning your driver" *(v0.2.2)* |
| Delivery ∈ InTransit | "In transit" |
| Delivery ∈ AwaitingPayment | "Awaiting payment" |
| Delivery ∈ Disputed | "Payment overdue — on hold" |
| Delivery ∈ Completed | "Delivered" |

Rules that keep the abstraction honest:

- The words *agent*, *saga*, *transition*, *token*, *embedding*, *marking*,
  *tool* never appear in any API response, UI string, or client communication.
- One layer down, the same discipline applies to the repository itself: the
  formal vocabulary never appears in code, schema, or code comments, and code
  never references this document set (the vocabulary firewall — internals
  Law 4, revision note item 9). There are therefore **two** firewalls: spec →
  code (Law 4) and code-internal → product (this section).
- Staff see **recommendations, not machinery**: "Suggested truck: DK-114 (saves
  62 km repositioning)" with approve/override — operations language, not
  dispatch-algorithm language. The decision trace backs the recommendation for
  staff who ask *why*, rendered as an explanation, not as tool-call JSON.
- MCP and ccloud are **internal engineering surfaces only** (§10), never
  product surfaces.

The product API (module `dakar-quarkus`, path `/v1`) is a thin adapter that (a)
translates a client request into a `Requested` delivery token and (b) reads the
marking projection for status. All intelligence is behind that boundary.

---

## 10. Deployment topology (grounded in the current stack)

The system extends the existing repository layout: multi-module Maven
(`dakar-bom`, `dakar-parent`, `dakar`, `dakar-quarkus`, standalone `cdk`),
Quarkus 3.20.x, AWS CDK deploying Quarkus apps as Lambdas via `FUNCTION_ASSET`.

```
                         ┌──────────────────────────────┐
   client / staff ──────▶│  API Gateway → dakar-quarkus λ │  (product surface, /v1)
                         └───────────────┬──────────────┘
                                         │ writes Requested token / reads projection
                                         ▼
                         ┌──────────────────────────────┐
                         │        CockroachDB Cloud       │◀── system of record (multi-region)
                         │ marking · event log · traces · │
                         │           vectors              │
                         └───────▲───────────────▲───────┘
                                 │               │ perception tools (AS OF SYSTEM TIME)
        EventBridge ──▶ ┌────────┴───────┐ ┌─────┴────────┐
        (signals)       │ Dispatcher λ   │ │ Negotiator λ │ ──▶ Amazon Bedrock
        (disruptions)   │ Saga driver λ  │ │              │     (Converse tool use + embeddings)
                        └────────┬───────┘ └──────────────┘
                                 │ artifacts (PODs, weighbridge tickets, contracts)
                                 ▼
                          Amazon S3
```

- **AWS services used:** Lambda (agent + product execution — matches the current
  `FUNCTION_ASSET` CDK scaffolding), Amazon Bedrock (Converse API with tool
  use, plus Titan/Cohere embeddings), Amazon EventBridge (saga wake-ups /
  disruption signals), Amazon S3 (proof-of-delivery, weighbridge tickets,
  contracts). This satisfies the "at least one AWS service" requirement several
  times over.
- **CockroachDB tools used (≥2 required, satisfied):** (1) **Distributed vector
  indexing** — load-bearing episodic memory, queried by the agents' own
  perception tools; (2) **MCP Server** — self-hosted
  (`cockroachdb/cockroachdb-mcp-server`), cert-authenticated read-only
  operator introspection of live marking, decision traces, and lane/client
  memory (§11's `mcp_operator` principal), and the dev-loop surface.
  **ccloud CLI** (provisioning the cluster and per-agent RBAC from CDK/CI) is
  roadmap — it needs a CockroachDB Cloud cluster, currently disabled
  (Request Unit limit, no timeline); not required once two tools are already
  satisfied.

---

## 11. Security, resilience, observability, and failure modes (production readiness)

**Access control (least privilege per agent).** Each agent gets its own
CockroachDB service account, provisioned via `ccloud` in the deploy pipeline,
with the *minimum* grants its transitions require. The grants table **is** the
tool inventory (§8.2) — capability and permission are the same list:

| Principal | Grants (= action + perception tools) |
|---|---|
| Dispatcher | `SELECT` on trucks/deliveries/lane_stat; `UPDATE` for `ClaimTruck`; `INSERT` on `operations_journal`, `decision_trace` |
| Negotiator | `SELECT` on episodes/profiles/lanes/policy; `UPDATE` for `Quote`; `INSERT` on episodes/firings/traces |
| Saga driver | `UPDATE` for physical transitions; `INSERT` on firings/traces |
| Product API | `INSERT` Requested tokens; `SELECT` on marking projection (follower reads) |
| Staff console | `UPDATE` for 👤 transitions (`ReleaseTruck`, `SettleDispute`, `WriteOff`, `TransferCargo`, `AbortLoaded`) + approvals |
| Operators (MCP) | `SELECT`-only, cert-authenticated as `mcp_operator` (no password), scoped to exactly `truck, delivery, operations_journal, decision_trace, negotiation_episode, client_profile, lane_stat` — self-hosted CockroachDB MCP Server, `ops/service-accounts.sql` |

**Resilience.** CockroachDB multi-region with survivability configured so the
memory tolerates node and zone failure with zero data loss and no maintenance
windows — the "memory that never goes down" requirement. Because agents are
crash-only (MI-1) and firing is atomic (§7), a Lambda dying mid-decision leaves
no partial state; the work is simply re-driven.

**Observability.** Every state change is already an `operations_journal` row and
every agent decision a `decision_trace` row, so the audit trail and the
operational log are the same artifact. MCP audit logging covers all operator
introspection. Metrics: firing rate per transition, `40001` retry rate
(contention pressure), rule-refusal rate per agent (a spike in Negotiator
floor rejections is an early signal of a misbehaving model or a bad cost
model), tool-call rate and deliberation-loop length per agent (cost and
model-health signal), time-in-place for `Disputed` (staff-queue health, the F2
fairness assumption made measurable).

**Failure modes and responses.**

| Failure | Response |
|---|---|
| Two agents claim one truck | `SERIALIZABLE` → one `40001` → loser re-perceives and re-plans (§7, §8.2). No double-booking (INV-5, three layers deep). |
| Agent proposes below-floor price | `Quote` guard false → typed tool error back to the model; nothing committed; bounded re-deliberation. |
| Agent calls an unknown tool / malformed arguments | Rejected, never repaired by guessing; policy-violation metric increments. |
| Deliberation loop runs away | Hard bounds from procedural memory (`max_perception_calls`, `max_action_attempts`); episode ends with a typed no-action outcome. |
| Lambda dies mid-firing | Transaction never committed; marking unchanged; re-driven. |
| Client never accepts / never pays | ⏲ `QuoteExpired` / `PaymentOverdue` route to terminal or staff resolution — LIVE-1 needs no client fairness (§5). |
| Loaded truck fails permanently | 👤 `TransferCargo` (rescue truck continues the delivery) or `AbortLoaded` — no delivery or truck is stranded in `InTransit`/`EnRouteLoaded` (model.md §7). |
| Model/LLM unavailable | Physical transitions (Saga driver) still fire on external signals; only *optimization* (Dispatcher ranking, Negotiator drafting) degrades — the business keeps moving, it just gets less clever. Graceful degradation, not stop. |
| Region event | Multi-region survivability; memory stays authoritative. |
| Bad guardrail deployed | `policy` is versioned data (MI-3); roll back to prior version, audited. |

Note the degradation story: because *safety* lives in the net's guards and the
database, and only *heuristics* live in the LLM, an AI outage makes Dakar dumber
but never unsafe and never stopped.

---

## 12. How this maps to the challenge judging criteria

| Criterion | Where satisfied |
|---|---|
| **Agentic Memory Design** | CockroachDB is the system of record for state (marking), embeddings (episodic negotiation/disruption memory), decision traces, and transactional data (claims, payments) — all four memory kinds (§3, §6). Memory is the availability model, not a feature (§3, §11). |
| **Technical Implementation** | Transition-firing-as-serializable-transaction with correct `40001` retry (§7); tool-use loop with snapshot-consistent perception via `AS OF SYSTEM TIME` (§8.2); native vector index co-located with operational data and queried by the agent itself (§6.2, §8.4); per-agent RBAC ≡ tool inventory via ccloud; read-only audited MCP (§10, §11). |
| **Real-World Impact** | No-double-booking on multi-thousand-dollar assets enforced at three layers (§5 INV-5); guaranteed lifecycle termination without client cooperation (§5 LIVE-1); backhaul/deadhead reduction on the roadmap with a headline economic number (§8.6); replaces a heuristic dispatcher with an auditable one. |
| **Production Readiness** | Multi-region survivability, least-privilege service accounts, versioned guardrails governing the agents' own limits, crash-only agents, explicit failure-mode table including agent-specific failures, graceful AI-outage degradation (§11). |
| **Creativity & Originality** | Three framings: (1) **transitions as the agent's tools** — the tool executor is a serializable transaction and an illegal tool call cannot commit (§8.2); (2) vector search as *episodic memory over business episodes*, not document RAG (§6.2); (3) crash-only agents whose identity is a marking row, not a process (§3, §7). |

---

## 13. MVP cut line

Build the **Saga driver + Dispatcher + Negotiator** trio, one end-to-end
delivery:

```
request → agent-matched truck (ClaimTruck via action tool, serializable firing)
        → agent-drafted, human-approved quote (self-directed vector recall
          + floor guard; decision trace attached to the draft)
        → simulated transit with one injected disruption (EventBridge;
          Saga driver chooses reassign / wait / escalate)
        → payment → offload (payment-before-offload guard) → Completed
```

The full net — including the timeout/dispute transitions — ships in the domain
core (they are cheap and make LIVE-1 true); the demo exercises the happy path
plus one disruption.

**Demo centerpiece:** kill the agent's Lambda mid-negotiation and show the saga
resume from CockroachDB with full context — memory *is* the system, made
visible. Then show the `decision_trace` of the resumed episode: perception
calls, rationale, the tool call that fired — agency, made auditable.

Out of MVP, in the roadmap slide: Backhaul Hunter, Disruption agent, autonomous
(not human-approved) negotiation, min-cost-assignment solver behind the
Dispatcher, TLA+ spec of the contended-claim protocol, client-facing actuators.

---

## Appendix A — Glossary (internal terms; never surfaced to the product)

- **Marking** — the multiset of tokens across all places; the live system state.
- **Token** — a colored record representing one truck or one delivery.
  Assignment is a color field on the truck token, not a token of its own.
- **Place** — a discrete state a token can occupy.
- **Transition** — an event that moves tokens; fires iff enabled and its guard holds.
- **Guard** — a boolean precondition on a transition; where business rules and
  safety live.
- **Firing** — one atomic execution of a transition = one serializable CRDB txn.
- **Action tool** — a transition exposed to the model via the Bedrock tool-use
  API; executing it means planning and firing that transition.
- **Perception tool** — a read-only, snapshot-consistent query tool an agent
  invokes during deliberation.
- **Decision episode** — one agent invocation: snapshot → bounded tool-use
  deliberation → at most one firing; persisted as a `decision_trace` row.
- **Policy (agent)** — an LLM-backed function choosing among enabled transitions.
- **Policy (procedural memory)** — a versioned guardrail row in the `policy`
  table, governing prices, timeouts, and the agents' own loop bounds.

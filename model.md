# Dakar — The Formal Model

**Status:** Normative v1.0 — the formal authority for the Dakar Colored Petri Net
**Audience:** Dakar engineering, formal-methods reviewers, challenge judges who ask "show me"
**Companions:** [system_architecture.md](system_architecture.md) (system authority; its §4–§5 summarize this document), [dakar_internals.md](dakar_internals.md) (module authority; the Java core implements this net), [agentic.md](agentic.md) (the agent layer that chooses among the moves this net permits). Where documents disagree, fix the disagreement — neither silently wins.

---

## 0. Purpose and scope

This document defines the Dakar net **exactly once, completely**: every color
set, every place, every transition (operational, exceptional, and
administrative), every guard, and the environment semantics. It then states and
proves the safety invariants, states and proves liveness under explicit
fairness assumptions, proves that safety holds **for every possible agent
policy** (including a malicious or hallucinating one), and establishes the
refinement claim: that the CockroachDB implementation described in
system_architecture.md §6–§7 is a faithful executor of this net.

What this document is *not*: a mechanized proof. The proofs here are rigorous
pen-and-paper inductions, checkable by a careful reader. Their executable
counterpart is the property-based invariant suite (backlog P3-3) which checks
every invariant against the *actual Java implementation* on every build. A
TLA+ mechanization of §9's concurrency argument is roadmap (backlog,
Post-MVP). We claim exactly what exists and no more.

---

## 1. Mathematical preliminaries

**Notation.**

- `⊥` denotes "absent/none". For a set `S`, write `S⊥ = S ∪ {⊥}`.
- A **finite partial map** `f : A ⇀ B` has finite domain `dom(f)`.
- `𝔹 = {true, false}`; `ℚ≥0` the non-negative rationals.
- **Time** is an abstract totally ordered domain `(𝕋, ≤)` with no other
  structure (implemented as UTC `Instant`). `τ` ranges over 𝕋.
- **Identifiers**: `KID` (truck ids), `DID` (delivery ids), `CID` (client
  ids) are countable sets of opaque atoms (implemented as UUIDs). Identity is
  equality of atoms; nothing else about an id is meaningful.
- For a record value `x`, field access is written `field(x)`, e.g. `cap(k)`,
  `asg(k)`, `pay(d)`.

**Multisets** are not needed in their full generality here: Theorem 1 (§3.3)
shows every reachable marking assigns each token id at most one token, so
reachable markings are isomorphic to finite partial maps. We therefore work
with the map representation throughout, and note where the general CPN
multiset semantics would differ (nowhere reachable).

---

## 2. The color universe

### 2.1 Scalar colors

| Color set | Definition | Well-formedness |
|---|---|---|
| `TONS` | ℚ>0, scale 3 | strictly positive |
| `MONEY` | ℚ≥0, scale 2, single currency per value | non-negative by construction |
| `GEO` | [−90, 90] × [−180, 180] ⊂ ℝ² | −0.0 normalized to 0.0 |
| `DUR` | ℚ≥0 (hours) | non-negative |
| `CARGO` | finite enum | — |
| `TPLACE` | `{Garage, EnRouteEmpty, Loading, EnRouteLoaded, Offloading, Maintenance}` | — |
| `DPLACE` | `{Requested, Matched, Quoted, Agreed, InTransit, AwaitingPayment, Disputed, Completed, Cancelled}` | — |

Terminal delivery places: `TERM = {Completed, Cancelled}`.
Active delivery places: `ACT = {Matched, Quoted, Agreed, InTransit, AwaitingPayment, Disputed}`.
Assigned-coupled places: `COUP = ACT \ {Disputed}` (places where a truck is
*necessarily* attached — Lemma PHASE, §6.7).

### 2.2 Token colors

```
Truck    ≜ ⟨ id : KID,
             place : TPLACE,
             cap : TONS,
             pos : GEO,                    -- telemetry-owned (§5.3)
             hrs : DUR,                    -- telemetry-owned (§5.3)
             maint : 𝕋⊥,
             asg : DID⊥ ⟩

Delivery ≜ ⟨ id : DID,
             place : DPLACE,
             client : CID,
             ton : TONS,
             orig : GEO,
             dest : GEO,
             cargo : CARGO,
             floor : MONEY,                -- resolved from procedural memory at creation
             agreed : MONEY⊥,
             pay : 𝔹,
             qExp : 𝕋⊥,                    -- stamped by Quote
             pDue : 𝕋⊥,                    -- stamped by ArriveDest
             ddl : 𝕋 ⟩
```

**Well-formedness of a token** (checked by the record constructors — internals
§5, backlog P1-4):

```
WF-K(k) ≜ cap(k) > 0
WF-D(d) ≜ floor(d) > 0
        ∧ (agreed(d) ≠ ⊥ ⇒ agreed(d) ≥ floor(d))
        ∧ (pay(d) ⇒ agreed(d) ≠ ⊥)
        ∧ (place(d) ∈ {Quoted, Agreed, InTransit, AwaitingPayment, Disputed, Completed}
             ⇒ agreed(d) ≠ ⊥)                        -- price exists from Quote onward
        ∧ (qExp(d) ≠ ⊥ ⇔ place(d) ∈ {Quoted, …})     -- stamped exactly when quoted
        ∧ (pDue(d) ≠ ⊥ ⇐ place(d) ∈ {AwaitingPayment, Disputed})
```

The color constraints are *not* decorative: `agreed ≥ floor` at the type level
means even a corrupted write path cannot represent a below-floor agreement,
independent of the `Quote` guard. This is the same defense-in-depth pattern as
INV-5's unique index (§9.4).

---

## 3. Markings

### 3.1 Definition

A **marking** is a pair of finite partial maps

```
M = ( K : KID ⇀ Truck,  D : DID ⇀ Delivery )
```

subject to the **key–identity condition**: `K(κ).id = κ` and `D(δ).id = δ`
for all keys. We write `k ∈ M` for `k ∈ ran(K)`, `place_M(k)` for the place of
`k` in `M`, etc. This is exactly the shipped Java `DispatchBoard` record (internals
§5.3), which enforces the key–identity condition and duplicate rejection in
its constructor.

### 3.2 Correspondence to CPN multiset semantics

In orthodox CPN terms, each place `p` holds a multiset of tokens of color
`C(p)`. The map representation above is justified by:

> **Theorem 1 (token uniqueness / functional representation).**
> In every reachable marking (§5), each truck id and each delivery id labels
> **exactly one** token, located in exactly one place. Hence the multiset
> marking is isomorphic to the pair of maps of §3.1, with `place` as a field.

*Proof.* Immediate from INV-1 and INV-3 (§6), whose proofs do not presuppose
this theorem: the initial marking is well-formed by assumption (§5.1), and
inspection of every transition in §4 shows each consumes and produces tokens
with preserved, never-duplicated ids. ∎

The practical weight of Theorem 1: **the relational schema (one row per token,
the place as the `status` column — system_architecture.md §6.1) is not an
approximation of the net; it is an exact representation of its reachable
fragment.**

### 3.3 Well-formed markings

```
WF(M) ≜ (∀k ∈ M. WF-K(k)) ∧ (∀d ∈ M. WF-D(d)) ∧ key–identity
```

All invariants in §6 implicitly conjoin WF; every transition preserves it
(each produced token passes its constructor).

---

## 4. Transitions

### 4.1 Schema notation

Each transition is given as:

```
Name (binding variables) [class]
  pre:    place requirements on consumed tokens
  guard:  boolean formula over the binding, token colors, τ (now), and signals
  effect: produced token values (only changed fields shown)
```

Classes: **[op]** operational (agent- or shell-fired), **[⏲]** timeout —
enabled by the passage of time alone, fired by the scheduler, **[👤]**
staff-authorized — fired only from the staff console, **[adm]** administrative
(fleet management/ops tooling). Every class is formally identical (a guarded
transition); the class only determines *which principal* may fire it (§9.4,
agentic.md §8).

`τ` is the evaluation time supplied by the snapshot (never a wall-clock read —
internals Law 2). `sig(x)` denotes an environment signal (§5.2).

### 4.2 Operational transitions

```
ClaimTruck (k, d, r : RouteEstimate)                                  [op]
  pre:    place(k) ∈ {Garage, EnRouteEmpty},  place(d) = Requested
  guard:  cap(k) ≥ ton(d)  ∧  hrs(k) ≥ estHours(r)  ∧  asg(k) = ⊥
          -- estHours(r) ≜ drive(pos(k) → orig(d)) + drive(orig(d) → dest(d)):
          -- the repositioning leg plus the loaded leg (implementation-fixed
          -- v1.0.1; both estimates are recorded as R-BINDING witnesses)
  effect: asg(k) := id(d);  place(d) := Matched

Quote (d, offer : MONEY, v : DUR)         -- v = quote validity, from policy   [op]
  pre:    place(d) = Matched
  guard:  offer ≥ floor(d)
  effect: agreed(d) := offer;  qExp(d) := τ + v;  place(d) := Quoted

AcceptQuote (d)                                                       [op]
  pre:    place(d) = Quoted
  guard:  sig(clientAccepted(d))  ∧  τ < qExp(d)
  effect: place(d) := Agreed

QuoteExpired (d)                                                      [⏲]
  pre:    place(d) = Quoted
  guard:  τ ≥ qExp(d)
  effect: place(d) := Cancelled;  for the k with asg(k)=id(d): asg(k) := ⊥

BeginReposition (k, d)                                                [op]
  pre:    place(k) = Garage
  guard:  asg(k) = id(d)  ∧  pos(k) ≠ orig(d)
  effect: place(k) := EnRouteEmpty

ArriveOrigin (k, d)                                                   [op]
  pre:    place(k) ∈ {EnRouteEmpty, Garage},  place(d) = Agreed
  guard:  asg(k) = id(d)  ∧  pos(k) = orig(d)        -- position: telemetry
  effect: place(k) := Loading

Load (k, d, loadedTons : TONS)                                        [op]
  pre:    place(k) = Loading,  place(d) = Agreed
  guard:  asg(k) = id(d)  ∧  loadedTons ≤ cap(k)
  effect: place(k) := EnRouteLoaded;  place(d) := InTransit

ArriveDest (k, d, t : DUR)                -- t = payment terms, from policy    [op]
  pre:    place(k) = EnRouteLoaded,  place(d) = InTransit
  guard:  asg(k) = id(d)  ∧  pos(k) = dest(d)
  effect: place(k) := Offloading;  place(d) := AwaitingPayment;  pDue(d) := τ + t

ConfirmPayment (d)                                                    [op]
  pre:    place(d) ∈ {AwaitingPayment, Disputed}
  guard:  sig(paymentSettled(d))
  effect: pay(d) := true                              -- place unchanged

Offload (k, d, c : {Garage, EnRouteEmpty})                            [op]
  pre:    place(k) = Offloading,  place(d) ∈ {AwaitingPayment, Disputed}
  guard:  asg(k) = id(d)  ∧  pay(d)
  effect: place(k) := c;  asg(k) := ⊥;  place(d) := Completed

PaymentOverdue (d)                                                    [⏲]
  pre:    place(d) = AwaitingPayment
  guard:  τ ≥ pDue(d)  ∧  ¬pay(d)
  effect: place(d) := Disputed

RecordDisruption (d, note)                                            [op]
  pre:    place(d) ∉ TERM
  guard:  true
  effect: marking unchanged; the firing itself is the episodic record

Cancel (k⊥, d)                                                        [op/👤]
  pre:    place(d) ∈ {Requested, Matched, Quoted, Agreed}
  guard:  true                                        -- "not yet loaded" is structural:
                                                      -- Load leaves these places
  effect: place(d) := Cancelled;
          if ∃k. asg(k) = id(d):
              asg(k) := ⊥;
              place(k) := EnRouteEmpty if place(k) ∈ {Loading}   -- heads home
                          else place(k)                          -- unchanged
```

### 4.3 Exceptional (staff) transitions — the dispute & rescue sub-net

These transitions exist so that **no liveness claim depends on clients or
physics** (§7). Each is staff-authorized.

```
ReleaseTruck (k, d)                                                   [👤]
  pre:    place(k) = Offloading,  place(d) = Disputed
  guard:  asg(k) = id(d)  ∧  ¬pay(d)                  -- cargo moves to bonded storage
  effect: place(k) := Garage or EnRouteEmpty (staff choice);  asg(k) := ⊥
          -- d stays Disputed; custody of cargo transfers to storage

SettleDispute (d)                                                     [👤]
  pre:    place(d) = Disputed
  guard:  pay(d)  ∧  ¬∃k. asg(k) = id(d)              -- settled late; cargo released
  effect: place(d) := Completed

WriteOff (d)                                                          [👤]
  pre:    place(d) = Disputed
  guard:  ¬∃k. asg(k) = id(d)                         -- cargo disposition recorded
  effect: place(d) := Cancelled

TransferCargo (k₁, k₂, d, r)              -- rescue: loaded truck fails        [👤]
  pre:    place(k₁) = EnRouteLoaded,  place(k₂) ∈ {Garage, EnRouteEmpty},
          place(d) = InTransit
  guard:  asg(k₁) = id(d)  ∧  asg(k₂) = ⊥
          ∧  cap(k₂) ≥ ton(d)  ∧  hrs(k₂) ≥ estHours(r)
  effect: place(k₁) := Maintenance;  asg(k₁) := ⊥;
          place(k₂) := EnRouteLoaded;  asg(k₂) := id(d)
          -- d stays InTransit: the delivery continues on the rescue truck

AbortLoaded (k, d)                        -- total loss / post-load abort      [👤]
  pre:    place(k) = EnRouteLoaded,  place(d) = InTransit
  guard:  asg(k) = id(d)                              -- insurance/credit handled
  effect: place(k) := Garage or Maintenance (staff choice);  asg(k) := ⊥;      -- outside the net
          place(d) := Cancelled
```

*(v1.0: `TransferCargo` and `AbortLoaded` are introduced here and back-ported
to system_architecture.md §4.4. Without them, a loaded truck that breaks down
permanently strands its delivery in `InTransit` — the same class of liveness
hole the v0.2 dispute sub-net closed for payment.)*

### 4.4 Administrative transitions — fleet boundary and maintenance

```
RegisterTruck (k_new)                                                 [adm]
  pre:    id(k_new) ∉ dom(K)
  guard:  WF-K(k_new)  ∧  asg(k_new) = ⊥  ∧  place(k_new) = Garage
  effect: K := K ∪ {k_new}

Decommission (k)                                                      [adm]
  pre:    place(k) ∈ {Garage, Maintenance}
  guard:  asg(k) = ⊥
  effect: K := K \ {k}

StartMaintenance (k)                                                  [adm]
  pre:    place(k) = Garage
  guard:  asg(k) = ⊥
  effect: place(k) := Maintenance

CompleteMaintenance (k)                                               [adm]
  pre:    place(k) = Maintenance
  guard:  true
  effect: place(k) := Garage
```

Deliveries enter the net only via the product API creating a `Requested`
token (`RequestDelivery [adm]`: fresh id, WF-D, place `Requested`) and never
leave it — terminal tokens are retained (they are the ledger). INV-3 (§6.3)
counts trucks *modulo* `RegisterTruck`/`Decommission` by construction.

### 4.5 The binding-witness requirement (R-BINDING)

> **R-BINDING.** The journal entry (`JournalEntry`) for a firing records the **full
> guard valuation**: every binding variable *including* values drawn from
> outside the token colors — the route estimate `r` in `ClaimTruck` and
> `TransferCargo`, the policy-derived `v` in `Quote` and `t` in `ArriveDest`,
> and the position/signal predicates that held.

Rationale: replay (§10) re-checks enabledness of each event. Token colors are
in the marking, but route estimates come from the `RouteTable` (snapshot data,
internals §6) and policy windows from `PolicySet` — neither is recoverable
from the log unless the event carries it. With R-BINDING, **replay is closed
over (M₀, log)**: no external system needs to be consulted, ever, to audit
history. This is normative for backlog P3-4.

---

## 5. Semantics

### 5.1 Runs

The initial marking `M₀` is any well-formed marking whose deliveries are all
in `TERM ∪ {Requested}` and whose trucks are unassigned (fresh install: empty;
migration: current fleet in `Garage`).

An **environment state** is `E = (τ, S)` where `τ ∈ 𝕋` and `S` is the set of
raised signals (client acceptances, payment settlements — §5.2).

A **run** is a finite or infinite sequence

```
(M₀, E₀) →x₁ (M₁, E₁) →x₂ …
```

where each step `xᵢ` is either:

- a **firing** `(t, b, τᵢ)`: transition `t` under binding `b` with
  `pre`/`guard` satisfied in `(Mᵢ₋₁, Eᵢ₋₁)`; `Mᵢ` is `Mᵢ₋₁` with the effect
  applied; `Eᵢ = Eᵢ₋₁`; or
- an **environment step** `ε`: `Mᵢ` differs from `Mᵢ₋₁` only in
  telemetry-owned fields (§5.3); `Eᵢ` may advance `τ` (never backwards) and
  add/consume signals.

`R(N, M₀)` is the set of markings reachable by any run. All safety claims in
§6 quantify over `R`.

### 5.2 Signals (the open-net boundary)

`AcceptQuote` and `ConfirmPayment` are guarded on external signals: the net is
**reactive/open**. Signals are *inputs*; the net never blocks the environment
and the environment never changes places or assignments. Crucially, §7 shows
liveness does **not** assume any signal ever arrives — signals only accelerate
runs toward `Completed`; their absence routes to terminals via ⏲/👤
transitions.

### 5.3 Telemetry ownership (net state vs. measurements)

The truck fields `pos` and `hrs` are **telemetry-owned**: they are
measurements of the physical world, updated by environment steps (a telemetry
adapter in the shell), not by firings.

This induces the formal split that the implementation and the replay audit
must respect:

| | Owned by | Changed by | Event-logged | Replay-audited |
|---|---|---|---|---|
| **Net state**: `place`, `asg`, `agreed`, `pay`, `qExp`, `pDue`, token existence | the net | firings only | yes (MI-2) | yes, exactly |
| **Telemetry**: `pos`, `hrs` | environment | environment steps | no | no (witnessed in guards via R-BINDING) |

Guards may *read* telemetry (e.g. `ArriveOrigin`); the value read is recorded
in the firing's witness (R-BINDING), so replay re-checks the guard against the
recorded witness, not against unreproducible sensor history. Every invariant
in §6 constrains **net state only** and is therefore unaffected by environment
steps — checked case-by-case trivially (environment steps change no
invariant-relevant field).

---

## 6. Safety: invariants and proofs

All proofs are by induction over runs: the property holds in `(M₀, E₀)` and is
preserved by every environment step (trivial — §5.3) and every firing (case
analysis over §4). Cases that do not touch the relevant fields are omitted as
trivially preserving.

### 6.1 INV-1 — single location

> For every truck id κ ∈ dom(K), the truck occupies exactly one place.

*Proof.* By the map representation (§3.1) a truck has exactly one `place`
field; what requires proof in multiset semantics is that no transition
duplicates or drops a truck token. Inspection: every transition of §4 that
consumes a truck produces exactly that truck (same id) exactly once, except
`RegisterTruck` (adds a fresh id, guard `id ∉ dom(K)`) and `Decommission`
(removes). No transition produces a truck with an id equal to an existing
different token (all effects preserve `id`). ∎

**Scope note (unchanged from arch v0.2):** INV-1 does *not* forbid
double-booking; that is INV-5. INV-1 forbids a truck being "in two states at
once" — e.g. simultaneously `Loading` and `Maintenance` — which is what makes
the `place` column encoding sound.

### 6.2 INV-2 — payment before completion

> ∀d: place(d) = Completed ⇒ pay(d) = true.

*Proof.* Base: M₀ has no `Completed` deliveries with ¬pay (WF plus §5.1; a
migrated ledger must satisfy this to be admissible). Step: the only
transitions producing `place(d) := Completed` are `Offload` (guard `pay(d)`)
and `SettleDispute` (guard `pay(d)`). No transition ever sets `pay(d) :=
false` (`pay` is monotone: only `ConfirmPayment` writes it, to `true`). Hence
preserved. ∎

Monotonicity of `pay` is itself worth stating: **once paid, always paid** —
there is deliberately no refund transition in v1.0; refunds are a
business-ledger concern outside the net, so the net cannot "unhappen" a
payment that gated an offload.

### 6.3 INV-3 — truck conservation

> |dom(K)| changes only at `RegisterTruck` (+1) and `Decommission` (−1); all
> other steps preserve it.

*Proof.* Inspection of §4: only those two transitions change `dom(K)`.
Environment steps never change token existence (§5.3). ∎

### 6.4 INV-4 — capacity soundness

> (a) No `Load` fires with `loadedTons > cap(k)`.
> (b) ∀k, d: asg(k) = id(d) ⇒ cap(k) ≥ ton(d).

*Proof.* (a) is the `Load` guard verbatim. (b): assignments are created only
by `ClaimTruck` (guard `cap ≥ ton`) and `TransferCargo` (guard `cap(k₂) ≥
ton(d)`); `cap` and `ton` are immutable colors (no transition writes them);
all other transitions only clear assignments. ∎

### 6.5 INV-5 — assignment injectivity (no double-booking)

> In every reachable marking:
> **(a) Injectivity.** asg(k₁) = asg(k₂) ≠ ⊥ ⇒ k₁ = k₂.
> **(b) Domain soundness.** asg(k) = δ ⇒ δ ∈ dom(D) ∧ place(D(δ)) ∈ ACT.
> **(c) Coverage.** place(d) ∈ COUP ⇒ ∃! k. asg(k) = id(d);
> place(d) = Disputed ⇒ at most one such k.

*Proof.* Base: M₀ has no assignments (§5.1). Step — the complete list of
transitions that read or write `asg` or move deliveries in/out of ACT:

| Transition | Effect on assignment state | Preserves (a) | (b) | (c) |
|---|---|---|---|---|
| `ClaimTruck` | sets `asg(k) := δ`; guard `asg(k) = ⊥`; `d`: Requested→Matched | ✓ new target δ was in `Requested`, so by (b)+(c) inductively **no** truck had δ; injectivity extends | ✓ δ enters ACT in the same firing | ✓ Matched gains exactly the one new truck |
| `Quote`, `AcceptQuote` | move `d` within COUP | ✓ untouched | ✓ | ✓ truck unchanged |
| `QuoteExpired` | `d`: Quoted→Cancelled; clears the unique `asg(k)=δ` | ✓ | ✓ δ leaves ACT and its assignment is cleared **in the same firing** | ✓ |
| `Cancel` | same shape as `QuoteExpired` over {Requested..Agreed} | ✓ | ✓ | ✓ (Requested had no truck; others had exactly one, cleared) |
| `Load`, `ArriveDest`, `BeginReposition`, `ArriveOrigin` | move tokens, `asg` untouched, `d` stays in COUP | ✓ | ✓ | ✓ |
| `ConfirmPayment` | no place/asg change | ✓ | ✓ | ✓ |
| `Offload` | `d`: →Completed (leaves ACT); `asg(k) := ⊥` same firing | ✓ | ✓ | ✓ |
| `PaymentOverdue` | `d`: AwaitingPayment→Disputed; truck untouched | ✓ | ✓ | ✓ Disputed permits one truck |
| `ReleaseTruck` | clears `asg(k)`; `d` stays Disputed | ✓ | ✓ | ✓ Disputed permits zero |
| `SettleDispute`, `WriteOff` | guard `¬∃k. asg(k)=δ`; `d` leaves ACT | ✓ | ✓ nothing dangles | ✓ |
| `TransferCargo` | `asg(k₁) := ⊥` and `asg(k₂) := δ` **atomically**; guard `asg(k₂) = ⊥` | ✓ count for δ stays exactly 1 through the firing | ✓ | ✓ |
| `AbortLoaded` | `d`→Cancelled; `asg(k) := ⊥` same firing | ✓ | ✓ | ✓ |
| `Decommission` | guard `asg(k) = ⊥` | ✓ removal of unassigned truck | ✓ | ✓ |

Every row preserves all three clauses; no other transition mentions `asg` or
ACT membership. ∎

**Corollary (no double-booking).** No reachable marking has one truck serving
two active deliveries (a: injectivity read in reverse — `asg` is a partial
*function* of the truck, so one truck has at most one delivery by
construction; and one delivery has at most one truck by (a)). This — not
INV-1 — is the multi-thousand-dollar theorem. Its three-layer runtime
enforcement is §9.4.

### 6.6 Monotone flags

`pay` is monotone (§6.2). `agreed` is write-once in practice but formally
overwritable only by `Quote`, which can only fire in `Matched`; once the
delivery leaves `Matched` the price is frozen. `qExp`/`pDue` are each stamped
by exactly one transition. These micro-invariants are what make WF-D's
implications stable.

### 6.7 Lemma PHASE — truck–delivery phase coherence

> For every d with place(d) ∈ ACT and the (by INV-5 unique, if any) k with
> asg(k) = id(d):
>
> | place(d) | place(k) |
> |---|---|
> | Matched, Quoted | Garage or EnRouteEmpty |
> | Agreed | Garage, EnRouteEmpty, or Loading |
> | InTransit | EnRouteLoaded |
> | AwaitingPayment | Offloading |
> | Disputed | Offloading, or no truck |

*Proof.* Induction; the truck-moving transitions each require the coupled
delivery place on the left of their row and produce the pair on the right
(`ArriveOrigin` requires `Agreed`; `Load` produces (EnRouteLoaded, InTransit);
`ArriveDest` produces (Offloading, AwaitingPayment); `TransferCargo` keeps
(EnRouteLoaded, InTransit); `ReleaseTruck` produces (—, Disputed);
`PaymentOverdue` moves AwaitingPayment→Disputed leaving the truck in
Offloading — permitted by the last row). `BeginReposition` moves
Garage→EnRouteEmpty, legal in rows 1–2. No other transition moves an assigned
truck. ∎

PHASE is not exposed as a numbered invariant (to avoid renumbering across the
doc set) but is checked by the property suite alongside INV-1…5 and is the
backbone of the liveness proof.

---

## 7. Liveness

### 7.1 Fairness assumptions — stated, minimal, internal

- **F1 (scheduler).** τ grows without bound along every infinite run, and any
  ⏲ transition continuously enabled is eventually fired. *(Discharged by: the
  timeout scheduler, backlog P5-5; measurable via the `Disputed`/`Quoted`
  time-in-place metrics.)*
- **F2 (staffed desk).** Any 👤 transition that is continuously enabled for a
  delivery making no other progress is eventually fired. *(Discharged by: the
  staff console and its queue; measurable the same way.)*

**No assumption is made about clients** (they may never accept, never pay) **or
about physics** (trucks may break down and never arrive). This is deliberate:
F1 and F2 are the only two things Dakar itself controls.

### 7.2 LIVE-1 — every delivery terminates

> Under F1–F2, along every run, every delivery token eventually reaches
> `Completed` or `Cancelled`.

*Proof.* Define the stage order
`Requested < Matched < Quoted < Agreed < InTransit < AwaitingPayment <
Disputed < TERM` and note every transition moves each delivery weakly upward
(inspection: no transition moves a delivery to an earlier stage; the only
same-stage transitions are `ConfirmPayment` — which fires at most once per
delivery since `pay` is monotone — and `RecordDisruption`, which changes no
state and hence cannot sustain progress-free change forever; environment steps
don't move deliveries). So each delivery's stage stabilizes unless it reaches
TERM. Suppose stage stabilizes at a non-terminal place; we exhibit a
continuously enabled ⏲/👤 exit, contradicting F1/F2:

| Stuck at | Continuously enabled exit | Needs |
|---|---|---|
| Requested | `Cancel` 👤 | F2 |
| Matched | `Cancel` 👤 | F2 |
| Quoted | `QuoteExpired` ⏲ (τ ≥ qExp eventually, F1: τ unbounded) | F1 |
| Agreed | `Cancel` 👤 | F2 |
| InTransit | `AbortLoaded` 👤 (truck is in `EnRouteLoaded` by PHASE) | F2 |
| AwaitingPayment | if ¬pay: `PaymentOverdue` ⏲; if pay: `Offload` is enabled (truck in `Offloading` by PHASE) — an [op] transition; treat via F2's staff console `assign`/drive or fold into F2 as "stalled" | F1/F2 |
| Disputed | truck attached: `ReleaseTruck` 👤 then; no truck: `SettleDispute` 👤 (if pay) or `WriteOff` 👤 | F2 |

Each exit either reaches TERM directly or strictly increases the stage
(AwaitingPayment→Disputed, Disputed-with-truck→Disputed-without-truck, a
finite refinement); the stage order is finite, so TERM is reached. ∎

### 7.3 LIVE-2 — every truck is eventually recoverable

> Under F1–F2, every truck eventually satisfies
> `asg = ⊥ ∧ place ∈ {Garage, EnRouteEmpty, Maintenance}`.

*Proof.* If `asg(k) = δ`, LIVE-1 terminates δ, and every transition that moves
a delivery to TERM clears the assignment of its truck in the same firing
(`Offload`, `QuoteExpired`, `Cancel`, `AbortLoaded`, `SettleDispute`/`WriteOff`
— the last two require it already cleared via `ReleaseTruck`). By PHASE the
truck is then in `Garage`, `EnRouteEmpty`, `Maintenance` (rescue donor), or
was moved there by the clearing transition (`Offload`/`ReleaseTruck`/
`AbortLoaded` all place the truck in a free place; `Cancel` from `Loading`
sends it `EnRouteEmpty`). ∎

### 7.4 What liveness does *not* claim

LIVE-1 is termination, not service: it guarantees no delivery and no truck is
stranded *in the state machine*; it does not guarantee deliveries complete
successfully (a write-off is a terminal, too), nor bound time-to-terminal
beyond the policy windows. Service-level properties are business metrics
(§ arch 11), not theorems.

---

## 8. Policy-independent safety (the agent theorem)

An **agent policy** is any function

```
π : Snapshot → (T × Binding)⊥
```

— given a snapshot it proposes a transition and binding, or abstains. The
composed system (arch §8, agentic.md) only ever executes a proposal through
`plan(...)` + the firing transaction, both of which re-check `pre` and
`guard`.

> **Theorem 2 (safety for all policies).** For every family of policies —
> including policies that are adversarial, hallucinating, prompt-injected, or
> simply wrong — every reachable marking of the composed system satisfies
> WF, INV-1…INV-5, and PHASE.

*Proof.* The composed system's runs are runs of `N` (§5): a proposal that is
not an enabled transition is rejected before firing (internals §7.4, §8.3) and
changes nothing; a proposal that is enabled fires as a legal step. The
invariants were proved over *all* runs of `N`, with no reference to how
bindings are chosen. ∎

> **Corollary (graceful degradation).** Policy quality affects only *which*
> legal run occurs — never whether the run is legal. An LLM outage, therefore,
> removes optimization, not safety, and (by §7, whose fairness assumptions
> mention no agent) not liveness either.

This theorem is the formal content of the sentence "the LLM proposes, the
formalism disposes." It is also the precise security claim against prompt
injection: an injected Negotiator can, at absolute worst, propose a legal
`Quote` within `[floor, ceiling]` — see agentic.md §8.

---

## 9. Refinement: the CockroachDB implementation executes this net

### 9.1 The encoding

Define `ρ(M)` = the database state where each truck token is one `truck`
row and each delivery token one `delivery` row, with `status` =
`name()` of the place enum and colors mapped per arch §6.1. Theorem 1
guarantees `ρ` is a bijection between well-formed markings and well-formed
table states. Net state maps to columns owned by firings; telemetry columns
(`position`, `driver_hours_rem`) are updated by the telemetry adapter (§5.3).

### 9.2 Firing = one `SERIALIZABLE` transaction

The executor contract (internals §7.5, arch §7): read tokens `FOR UPDATE`,
re-evaluate `pre`+`guard` via the domain, apply effects as version-guarded
`UPDATE`s, insert the `operations_journal` entry, commit.

> **Theorem 3 (soundness of committed transactions).** Every committed firing
> transaction corresponds to a legal firing `M —(t,b,τ)→ M′` on the encoded
> marking, and the concurrent execution of any number of such transactions is
> equivalent to some sequential interleaving of legal firings.

*Proof sketch.* (i) Within one transaction, the guard is evaluated against
`FOR UPDATE` reads — the same versions the writes are predicated on; a
concurrent conflicting commit forces a `40001` serialization failure or a
zero-row version-guarded update, both aborting the transaction. So a commit
implies the guard held on exactly the token values the effect was applied to.
(ii) CockroachDB `SERIALIZABLE` guarantees the committed history is equivalent
to some serial order of the transactions; by (i) each is a legal firing on the
marking left by its predecessor. Hence every reachable database state is
`ρ(M)` for some `M ∈ R(N, M₀)`, and **all §6 invariants hold in the database**.
∎

The **plan/execute split** (internals §7.4–7.5) does not weaken this: the plan
is advice computed from an earlier snapshot; the transaction re-derives
enabledness on current data. A stale plan produces an aborted transaction and
a re-plan, never an illegal commit.

### 9.3 What `40001` means in net terms

A serialization failure is the database detecting that two proposed firings
are **not independent** (their token sets intersect). The loser's retry —
fresh snapshot, re-plan — is exactly "re-evaluate enabledness in the new
marking." Contention handling is therefore not an implementation wart; it is
the net's interleaving semantics made operational.

### 9.4 Defense in depth for INV-5

INV-5 is enforced three times, independently:

1. **Domain:** the `ClaimTruck`/`TransferCargo` guards (§4) — proven in §6.5.
2. **Transaction:** serializability (Theorem 3) — two concurrent claims of one
   truck cannot both commit.
3. **Storage:** `CREATE UNIQUE INDEX truck_single_assignment ON truck
   (assigned_delivery) WHERE assigned_delivery IS NOT NULL` — even a bypassing
   adapter bug cannot persist a violation of clause (a).

Layers 1–2 are proved; layer 3 is a *witness*: its violation alarm firing in
production would falsify an assumption of this document and is treated as
severity-1.

### 9.5 Follower reads and perception staleness

Product status reads and agent perception use timestamp-pinned reads (arch §7,
§8.2). Formally these observe `ρ(Mᵢ)` for some *earlier* reachable `Mᵢ` — a
legal marking, never a torn state (snapshot isolation of reads). Stale
perception is safe because **no read is load-bearing for safety**: Theorem 3
places the entire safety burden on the firing transaction.

---

## 10. Replay

### 10.1 The fold

`Journal.replay(M₀, log)` (internals §10) applies events in order, re-checking
enabledness (using each event's R-BINDING witness for guard inputs outside net
state) and applying effects; it throws on any event not enabled — a corrupted,
reordered, or fabricated log fails loudly.

### 10.2 Ordering

> **Lemma (commutation).** Two firings whose token sets are disjoint commute:
> applying them in either order yields the same marking.

*Proof.* Effects write only fields of their own tokens; guards of the second
firing read only its own tokens' net state plus recorded witnesses. ∎

> **Theorem 4 (replay well-definedness).** Any linear extension of the
> per-token order (given by each token's `version` chain: the event writing
> version v+1 follows the event that produced version v) folds to the same
> final marking, equal to the live marking produced by the committed history.

*Proof sketch.* Theorem 3 gives a serial order `σ` of all firings whose fold
is the live marking. Any linear extension `σ′` of the per-token order agrees
with `σ` on the relative order of non-commuting (token-sharing) pairs, because
version chains totally order all firings on a shared token. Repeatedly swapping
adjacent commuting pairs transforms `σ` into `σ′` without changing the fold
(Lemma). ∎

Practical consequence: `operations_journal` needs **no global sequence
number**; `(token, expectedVersion)` pairs in the event suffice, and
`occurred_at` (event-supplied, §arch 6.2) is for humans, not for correctness.

### 10.3 The audit

The scheduled replay audit (backlog P6-5) computes
`fold(M₀, log) = live net-state projection?` — equality is over **net state
only** (§5.3); telemetry fields are excluded by definition, not by tolerance.
Divergence falsifies one of: the executor contract, R-BINDING completeness, or
this model — all severity-1.

---

## 11. Verification strategy: three layers, honestly labeled

| Layer | Artifact | What it establishes | Status |
|---|---|---|---|
| Pen-and-paper | this document, §6–§10 | invariants/liveness over *the model*; refinement argument | **exists (v1.0)** |
| Executable | jqwik property suite: random legal firing sequences from random WF markings, `BoardAudit` (INV-1…5 + PHASE) after every step; replay fold ≡ stepwise; log-mutation kills | the *Java implementation* satisfies the model's invariants; Law-2 determinism | **backlog P3-3 / P3-4** — runs on every build once merged |
| Mechanized | TLA+ spec of §9.2–§9.3 (two concurrent claimants, bounded fleet, retry loop) model-checked in CI | no interleaving of the *executor protocol* violates INV-5/atomicity | **roadmap (Post-MVP)** — does not exist; nothing in this document depends on it |

The three layers answer different failure modes: the proofs catch design
errors, the property suite catches implementation drift from the design, the
model checker (when it exists) catches protocol interleavings humans reason
about badly. None substitutes for another; only the first two are claimed
today.

---

## 12. Model–code–schema correspondence

This table is the **only bridge across the vocabulary firewall** (internals
Law 4): the repository contains no specification vocabulary and no reference
to this document set, so the mapping below — maintained here, on the
privileged side — is where spec and code meet. Traceability is one-way: this
document cites code symbols; code never cites this document. Amend this table
in the same PR as any rename on either side.

| Model element (this doc) | Java (`dakar` module) | CockroachDB |
|---|---|---|
| Truck / Delivery colors §2.2 | `Truck` / `Delivery` records (package `freight`) | `truck` / `delivery` rows |
| Place §2.1 | `TruckStatus` / `DeliveryStatus` enums | `status` columns (= `name()`) |
| WF constraints §2.2 | validating canonical constructors | `CHECK`-equivalent enforced at write path + types |
| Marking §3.1 | `DispatchBoard` (SortedMaps, key–identity; package `desk`) | the two tables |
| Transition schemas §4 | sealed `Action` records + rule functions | — (actions are code, not rows) |
| Guard verdicts | `RuleVerdict` / `RefusalReason` | `40001` / zero-row update on race |
| Firing `M —(t,b,τ)→ M′` §5 | `FreightDesk.plan` → `ActionPlan` (writes: `StateWrite`) | one `SERIALIZABLE` txn (§9.2) |
| Enabled ClaimTruck bindings | `FreightDesk.claimCandidates` → `ClaimCandidate` | — |
| Event + R-BINDING §4.5 | `JournalEntry` (id: `JournalEntryId`, action, `Evidence`, actor, `occurredAt = snapshot.now()`) | `operations_journal` row (`payload` = action + evidence) |
| Environment step §5.3 | (not in domain — shell adapters) | telemetry column updates |
| INV-1…5, PHASE §6 | `BoardAudit` (`ConsistencyViolation`) | + `truck_single_assignment` index (INV-5(a)) |
| Replay §10 | `Journal.replay` | audit job vs. live projection |
| Saga driver (actor) | `Actor.COORDINATOR` | actor string `coordinator` |
| Fairness F1/F2 §7 | — (operational, not code) | time-in-place metrics + alarms |

---

## 13. Evolution discipline

- A change to places, transitions, arcs, or guards is a **model version
  bump**: amend this document first, re-run the §6 case analyses for every
  invariant whose relevant-field set intersects the change, then amend
  system_architecture.md §4–5 and internals §7, then code (backlog ordering
  rule).
- New invariants get the next INV number; internal lemmas (PHASE) are named,
  not numbered, to keep cross-document references stable.
- Event compatibility: transitions are append-only versioned
  (`ClaimTruckV2`), and `Journal.replay` handles all released versions forever
  (internals §10). A model change that cannot replay history is inadmissible.

---

## Appendix A — symbol table

| Symbol | Meaning |
|---|---|
| `𝕋, τ` | time domain; current evaluation time (from snapshot) |
| `⊥`, `S⊥` | absent; `S ∪ {⊥}` |
| `K, D` | truck / delivery token maps of a marking |
| `asg, cap, ton, pay, qExp, pDue` | token color accessors |
| `ACT / COUP / TERM` | active / truck-coupled / terminal delivery places |
| `WF, WF-K, WF-D` | well-formedness predicates |
| `sig(·)` | environment signal predicate |
| `R(N, M₀)` | reachable markings |
| `π` | an agent policy (§8) |
| `ρ` | marking → database encoding (§9.1) |
| `F1, F2` | fairness: scheduler; staffed desk |

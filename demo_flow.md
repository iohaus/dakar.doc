# Dakar — Complete System Flow (Launch to Every Trigger)

**Purpose:** a ground-truth trace of the entire system, read directly from source
(not recalled, not summarized from docs) — every request, what it triggers,
what fires in response, and what the operator sees. Written to verify the
system actually behaves the way `demo_script.md` and the README claim it does.

**How to read this:** each flow names its exact trigger, the exact HTTP
call(s), the exact Java classes/methods involved, the exact DB effect, and
what happens on `dakar-console` as a result. Where the code disagrees with
what the docs say, that's called out explicitly, not smoothed over — see
**§13** for the one significant discrepancy found while writing this.

---

## 1. Launch: `python3 main.py`

`dakar-console/main.py:main()`, in order:

1. `QApplication` created, org/app name set (`dakar-console`), qasync event
   loop installed and set as the asyncio loop (`qasync.QEventLoop(app)`) —
   this is what lets the rest of the app use plain `async`/`await` (aiohttp
   calls) without blocking the Qt UI thread.
2. Fonts loaded from `dakar-console/fonts/Titillium_Web/*.ttf`.
3. A `Backend` instance is constructed — reads `DAKAR_API_BASE_URL`
   (default `http://localhost:8080`), initializes empty `_board`/`_fleet`/
   `_feed` lists, and creates a `QTimer` (`_poll_timer`) wired to
   `_schedule_poll`, interval `POLL_INTERVAL_MS_DEFAULT = 3000` ms — **but
   not started yet**.
4. `DkTheme.qml` registered as a singleton QML type; `QQmlApplicationEngine`
   loads `main.qml` with `backend` injected into the root context as
   `context.backend` — this is the only bridge between Python and QML; every
   QML view reads `backend.board`/`backend.fleet`/`backend.feed` and calls
   `backend.someSlot(...)` for actions.
5. **`backend.start_polling()`** — starts the `QTimer` *and* immediately
   fires one `_poll_snapshot()` off-cycle, so the UI has real data before the
   first 3-second tick even elapses.
6. `loop.run_forever()` — from here on, everything is event-driven: the poll
   timer, and whatever the operator clicks.

Nothing else happens at launch. No staff-key handshake, no board pre-fetch
before the engine loads — the QML views render empty/loading state for one
tick, then `_poll_snapshot()` lands and `snapshotChanged` fires.

---

## 2. The poll loop — the heartbeat everything else rides on

Every `pollIntervalMs` (3000 ms default, clamped 1000–30000, adjustable live
via `backend.pollIntervalMs`), `Backend._poll_snapshot()` runs:

```python
fleet_result, board_result, feed_result = await asyncio.gather(
    self._get("v1/trucks"),
    self._get("v1/deliveries?view=board"),
    self._get(f"v1/activity?limit=20"),
    return_exceptions=True,
)
```

Three **parallel** GETs, all staff-only (`@StaffOnly` on `TrucksResource.list`,
`DeliveriesResource.list`, `ActivityResource` — the console sends whatever
staff-key mechanism is configured on every request via the session headers).

- **`GET /v1/trucks`** → `FleetStore.listRecent()` — a **follower read**
  (`AS OF SYSTEM TIME follower_read_timestamp()`), may lag the true state by
  a few seconds by design (off the write hot path).
- **`GET /v1/deliveries?view=board`** → `DeliveryStore.listBoard()` — also a
  follower read; joins each delivery to its assigned truck.
- **`GET /v1/activity?limit=20`** → merged `operations_journal` +
  `decision_trace` rows, most recent 20 — this is the one place both
  *actions* (journal) and *agent reasoning* (traces) show up interleaved.

If any of the three fails or returns non-200, the whole poll is discarded and
`isStale` flips true (a UI staleness indicator) — **partial snapshots are
never shown**; the board only ever reflects a fully-consistent poll. On
success: `_board`/`_fleet`/`_feed` are replaced wholesale, a rolling 60-tick
`_board_history` gets the new board appended (for the Analytics view), and
`snapshotChanged` fires — every QML binding to `backend.board`/`.fleet`/
`.feed` re-renders, including `BoardCanvas`'s `_reconcile()`.

**This is the only mechanism by which the console ever learns anything
changed.** There is no push/websocket path. Every flow below that "the
console shows X" really means: *the next poll, up to 3 seconds later, picks
up X*. Staff actions (assign, quote, signal, accept, create) each trigger one
extra out-of-cycle `_poll_snapshot()` immediately after a successful response,
so the operator's own actions feel instant even though background agent
activity is on the 3-second clock.

---

## 3. Flow A — a delivery is requested

**Trigger:** `POST /v1/deliveries` — either a shipper's own `curl` (the
documented client path), or the console's `Backend.createDelivery(...)` slot
(the console *can* create deliveries too — the endpoint has no `@StaffOnly`,
it's the same client-facing surface either way, just reachable from either
UI).

`DeliveriesResource.create()`:

1. Requires an `Idempotency-Key` header (400 `MISSING_IDEMPOTENCY_KEY` if
   absent). Looked up first — a repeated key returns the **original** result
   (`store.findByRequestKey`), no duplicate ever created.
2. Builds a `Delivery` in `REQUESTED` status: resolves `origin`/`destination`
   either from `originLat`/`originLon` directly, or by geocoding
   `pickupAddress`/`dropoffAddress` (Nominatim) if coordinates are absent —
   this is the one moment in the whole system that calls out to a live
   external geocoding service.
3. Computes the **floor price** server-side, immediately, from
   `PolicyStore.load().pricing()` — base + per-ton-km × tonnage × distance.
   The client never sees or sets this.
4. `desk.plan(new RequestDelivery(delivery), snapshot, Actor.PRODUCT, ids)` →
   `ActionPlanExecutor.execute(...)`, with `store.recordRequestKey(key, id)`
   as a **companion write** — the idempotency-key row and the delivery insert
   commit in the *same* `SERIALIZABLE` transaction (§12), so a crash between
   them is impossible, not just unlikely.
5. On success: `events.publish(DETAIL_TYPE_DELIVERY_REQUESTED, {"deliveryId":...})`
   — fire-and-forget onto the EventBridge custom bus. **This is the trigger
   for Flow B.**

**Response:** `201`, body = the delivery (status `RECEIVED` in product
vocabulary). **Console reaction:** next poll (≤3s), the delivery appears on
the board and in the Deliveries list at `REQUESTED`/`RECEIVED`; the Activity
feed gets one `RequestDelivery` journal entry, actor `product`.

---

## 4. Flow B — the Dispatcher fires (autonomous)

**Trigger — three possible paths, all converging on the same method:**

| Path | Mechanism | Where |
|---|---|---|
| **Production** | `events.publish(delivery.requested)` (end of Flow A) → EventBridge rule → `dakar-dispatcher-agent` Lambda invoked with a synthesized `POST /internal/trigger/dispatcher` | Real AWS |
| **Dev/local** | `@Scheduled(every = "{dakar.agents.dispatch-every}")` — `15s`, gated by `dakar.agents.dispatch-enabled` (`true` only in `%dev`/`%test`) | `DispatchAutomation.tick()` |
| **Manual/demo** | `curl -X POST .../internal/trigger/dispatcher -H "X-Agent-Trigger-Key: ..."` | `AgentTriggerResource` |

All three call **`DispatchAutomation.runForAllPendingDeliveries()`** — there
is exactly one code path for "the dispatcher runs," regardless of what woke
it. It is idempotent by construction: it reads a fresh snapshot every
invocation, so a duplicate EventBridge delivery or an overlapping schedule
tick is harmless.

For **every** `REQUESTED` delivery in the snapshot (not just the one that
triggered it — a full sweep each time):

1. `desk.claimCandidates(snapshot)` — the domain enumerates every
   *guard-legal* truck→delivery pairing (capacity, driver hours, etc.) —
   grouped by delivery.
2. `policy.choose(candidates, DispatchContext(delivery, ...))` — either:
   - **`LocalDispatchPolicy`** (`dakar.agents.provider=local`, the default):
     picks the candidate with the least repositioning distance. Pure,
     deterministic, no network call.
   - **`BedrockDispatchPolicy`** (`provider=bedrock`): a single **Amazon
     Bedrock Converse** call to Claude 3 Haiku, with a **forced tool call**
     (`choose_truck`) — the model cannot respond with prose, only a
     structured choice from the candidates it was shown.
3. **Validation, regardless of which policy answered:** `if
   (!candidates.contains(choice))` → the choice is discarded, traced
   `outcome: policy-violation`, and **nothing is executed**. This is the
   reject-don't-repair guarantee made concrete — a hallucinated or malformed
   choice cannot reach the database no matter what the model said.
4. If valid: `executor.execute(fresh -> desk.plan(claim, fresh,
   Actor.DISPATCHER, ids), traces.recordWith(...))` — the `ClaimTruck`
   transition and its `decision_trace` row (candidates offered, choice made)
   commit in **one transaction**.
5. **Contention:** if a concurrent claim beat this one to the same truck, the
   `SERIALIZABLE` commit fails `40001`, `ActionPlanExecutor` re-plans from a
   fresh snapshot up to `dakar.actions.max-attempts` (4) times with capped
   jittered backoff. If every attempt loses, the tick simply ends — the
   delivery stays `REQUESTED` and the *next* dispatcher invocation (15s local
   tick, or the next `delivery.requested`/schedule event in prod) tries
   again. Nothing is lost, nothing is stuck.

**On `PolicyUnavailableException`** (Bedrock down): the whole tick aborts
immediately, no deliveries processed this round, logged as a warning — degraded,
not broken; the console's manual **Assign** button (Flow C) still works.

**Console reaction:** next poll, the delivery moves to `MATCHED` (product
status roughly "driver assigned" depending on truck movement), the truck
shows `assigned_delivery` set, the Activity feed gets a `ClaimTruck` journal
entry (actor `dispatcher`) plus a `decision_trace` row showing exactly which
candidates were offered and which was chosen. On the board canvas, nothing
visually moves yet — the truck is still sitting at its garage node; the
reposition leg starts once a `DEPARTED` signal arrives (Flow H).

**Also on success:** `assign()`'s caller path (see Flow C) publishes
`delivery.matched` — **but note:** `DispatchAutomation.dispatch()` itself does
**not** publish this event; only the *staff-manual* `assign()` endpoint does
(see Flow C, step 3). In production, that means the Negotiator's
`delivery.matched` EventBridge trigger only fires reliably when a human (or
the console) uses the assign endpoint — the autonomous dispatcher path relies
on the Negotiator's own scheduled tick / next sweep to notice the `MATCHED`
delivery instead. Worth confirming this is the intended shape before treating
"fully autonomous, zero clicks" as accurate for the *whole* chain in the demo
script — the Dispatcher→Negotiator handoff specifically depends on which
trigger fired the claim.

---

## 5. Flow C — staff manually assigns a truck

**Trigger:** console's Assign action → `Backend.assignTruck(delivery_id,
truck_id)` → `POST /v1/deliveries/{id}/assign` (`@StaffOnly`).

`DeliveriesResource.assign()`:
1. Fires `ClaimTruck` exactly like the Dispatcher does, but as
   `Actor.STAFF` — same guard, same transaction, same contention handling.
   No candidate-offering/validation step because a human chose directly.
2. On success (`200`/`201`): **`events.publish(delivery.matched, ...)`** —
   this is the one path confirmed to always trigger the Negotiator.

This is the manual override path: if the Dispatcher abstains (no legal
candidates) or is degraded (Bedrock down), staff assign directly from the
console's candidate-filtered picker, and the rest of the pipeline (Negotiator
onward) proceeds identically regardless of who made the claim.

---

## 6. Flow D — the Negotiator fires (autonomous)

**Trigger:** same three-path pattern as Flow B — EventBridge on
`delivery.matched` (prod, published by Flow C), `dakar.agents.lifecycle-every`
— correction, **no scheduled tick exists for the Negotiator** (`grep` confirms
`NegotiationAutomation` has no `@Scheduled` method at all, unlike Dispatcher
and Coordinator) — so locally/in dev, the **only** way `runForAllMatchedDeliveries()`
runs is the manual `POST /internal/trigger/negotiator` call. There is no
dev-mode automatic negotiation sweep; the demo script's automatic-EventBridge
framing is prod-only, and a local/dev run needs that curl explicitly.

For every `MATCHED` delivery with no open draft:

1. `desk.offerBand(deliveryId, snapshot)` → `[floor, ceiling]`. Empty →
   skip (already quoted, etc.).
2. Builds a query string from tonnage/cargo/lane, embeds it
   (`TranscriptEmbedder` — Bedrock Titan or local deterministic hash
   depending on `dakar.embedding.provider`).
3. `EpisodeStore.retrieveContext(...)` — the **real vector search**:
   `ORDER BY embedding <=> ?::VECTOR LIMIT 5`, filtered to the same lane +
   cargo class, `outcome IN ('won','lost')` only — this is genuinely the
   negotiator "remembering" past deals, not decoration.
4. `policy.proposeOffer(band, context)` — Bedrock Converse or local
   band-midpoint, same provider switch as Dispatch.
5. **Validated against the band** (`band.admits(...)`) — out-of-band is
   `policy-violation`, discarded, never reaches the database.

### ⚠️ The discrepancy

Step 6, if the offer is valid, is where the class's own Javadoc and its code
disagree. The Javadoc states:

> "The transition itself (`Quote`) is **NOT fired here** — the draft is
> staff-gated per the autonomy ladder. Staff approve via the
> `/v1/deliveries/{id}/quote` endpoint."

The code:

```java
ExecutionResult result = executor.execute(
        fresh -> desk.plan(
                new com.iohaus.dakar.desk.Quote(delivery.id(), proposal.get()),
                fresh, Actor.NEGOTIATOR, null),
        companion);
```

This is the *exact same* `desk.plan(...)` → `executor.execute(...)` pattern
every other real, committing transition in the codebase uses (compare
`DispatchAutomation.dispatch()`'s identical shape for `ClaimTruck`). A
`Completed` result here means the `Quote` transition **fired for real** —
the delivery moves to `QUOTED`, `agreed_price` is set, actor is `NEGOTIATOR`
— with no staff action anywhere in the path. The log line even calls it
"negotiation draft created," but nothing about the mechanism is draft-only;
`/v1/deliveries/{id}/quote` (Flow F, actor `STAFF`) is a separate, independent
way to fire the *same* transition, not an approval gate on top of this one.

**This directly contradicts:**
- `agentic.md` §5/§7's autonomy ladder ("Negotiator: draft + human approval").
- The README (`dakar/README.md`, written this session): *"then hands that
  draft to a staff member for a final yes. It never sends a price on its
  own."*

I have not changed either the code or the README — this could be an
intentional, undocumented move up the autonomy ladder, or a regression from
an earlier draft-only design. It's a real decision for the team, not
something to silently paper over in a flow document whose entire purpose is
ground truth. If the human-gate is the intended design, `NegotiationAutomation`
needs to stop calling `desk.plan(Quote...)` and instead only persist the
draft + trace (the way `QuoteAdviceResource`, Flow E, already correctly
does) — leaving the actual `Quote` firing solely to Flow F.

**Console reaction (as the code actually behaves today):** next poll after
an autonomous negotiator run, a `MATCHED` delivery can jump straight to
`QUOTED` with a price already set, with no staff click in between.

---

## 7. Flow E — staff requests quote advice (read-only, always draft-only)

**Trigger:** console's "Get Advice" action → `Backend.fetchAdvice(delivery_id)`
→ `GET /v1/deliveries/{id}/quote-advice` (`@StaffOnly`).

`QuoteAdviceResource.advise()` — same steps as Flow D (offer band, embed,
retrieve context, propose, validate) but **fires nothing**. It writes one
`decision_trace` row (`outcome: drafted|abstained|policy-violation`) and
returns the proposal as plain JSON (`suggestedOffer`, `floor`, `ceiling`,
`similarEpisodes`) for the console's advice card. This is genuinely
draft-only, exactly matching what `NegotiationAutomation`'s Javadoc claims for
itself. **This is the correct reference implementation of the human-in-the-loop
pattern** — Flow D should arguably look like this plus a separate staff-fire
step, not fire directly.

---

## 8. Flow F — staff fires the quote

**Trigger:** console's Quote submission (after reviewing advice) →
`Backend.submitQuote(delivery_id, offer)` → `POST /v1/deliveries/{id}/quote`
(`@StaffOnly`).

`DeliveriesResource.quote()`:
1. Parses the offer, builds a `NegotiationEpisodeDraft`, **embeds the
   transcript before the transaction begins** — an embedding-service outage
   fails the request outright rather than committing a `Quote` whose episode
   can never be recalled later.
2. `desk.plan(new Quote(deliveryId, offer), snapshot, Actor.STAFF, ids)` +
   `episodeStore.record(draft, embedding)` as companion — the price
   transition and its episode row commit together.

Guard-enforced: an offer below the policy floor is refused (`409
QUOTE_REFUSED`) at the domain layer regardless of who — human or agent —
proposed it; the guard doesn't know or care which actor is asking.

---

## 9. Flow G — the client accepts

**Trigger:** `POST /v1/quotes/{id}/accept` — client-facing, no staff key.

`QuotesResource.accept()`: `desk.plan(new AcceptQuote(deliveryId), snapshot,
Actor.PRODUCT, ids)` + `episodeStore.markWon(deliveryId)` companion — the
episode that was `open` since Flow D/F is now `won`, `final_price` copied
from `agreed_price`. **This is also the only place an episode is marked
`won`** — a delivery that never gets accepted keeps its episode `open`
forever unless it expires (Flow I marks those `lost`).

---

## 10. Flow H — physical signals (staff-driven, one delivery leg at a time)

**Trigger:** console's per-stage signal buttons → `Backend.sendSignal(id,
signal, loadedTons?)` → `POST /v1/deliveries/{id}/signals` (`@StaffOnly`).
There is **no automated signal source anywhere in the code** — no telematics,
no simulated GPS — every physical-world signal is a human clicking a button
in the console (or curl). This is the one part of the system with zero
autonomy by design; acknowledging reality needs no deliberation.

| Console signal | `DeliveriesResource.signal()` maps to | Truck status becomes | `BoardCanvas` leg (§ backlog P6-16) |
|---|---|---|---|
| `DEPARTED` | `BeginReposition(truck)` | `EN_ROUTE_EMPTY` | **reposition** leg starts: garage→pickup line appears |
| `ARRIVED_AT_PICKUP` | `ArriveOrigin(truck)` | `LOADING` | leg becomes **transit** (pickup→destination line appears immediately; reposition leg ghosts and fades) |
| `LOADED` | `Load(truck, loadedTons)` | `EN_ROUTE_LOADED` | same transit leg continues; vehicle glyph begins animating along it |
| `ARRIVED_AT_DESTINATION` | `ArriveDest(truck)` | `OFFLOADING` | same transit leg; vehicle glyph snaps to the destination pin (progress forced to 1.0) |
| `PAYMENT_CONFIRMED` | `ConfirmPayment(deliveryId)` | *(delivery-only; truck status unaffected)* | — |
| `OFFLOADED` | `Offload(truck, TruckStatus.GARAGE)` | **`GARAGE`**, hardcoded | transit leg ghosts and fades; vehicle returns to its garage node |

Two things worth being precise about for the demo:
- **`OFFLOADED` always parks the truck at `GARAGE`** — this specific
  resource hard-codes that choice (the formal net supports releasing straight
  to `EN_ROUTE_EMPTY` for an immediate next job, but nothing in the current
  API surface exercises that branch). A truck's very next assignment will
  always show a fresh reposition leg from garage, never a direct
  job-to-job line.
- **`ConfirmPayment` requires `paymentConfirmed` before `Offload`'s guard
  will admit it** — the "payment, then offload" rule from the README is a
  guard on `Offload`, not a UI-enforced ordering; clicking `OFFLOADED` before
  `PAYMENT_CONFIRMED` is refused by the domain layer itself (`409`), not by
  the console disabling a button.

---

## 11. Flow I — the Coordinator sweep (background, no trigger needed)

**Trigger:** `@Scheduled(every = "{dakar.agents.lifecycle-every}")` — `30s`
locally, **always on** regardless of `dakar.agents.*-enabled` flags (per its
own comment: "no LLM path... mechanically safe to re-fire"); in prod, an
EventBridge Scheduler rate rule invokes `dakar-coordinator-agent` instead
(no `@Scheduled` needed there — Quarkus dev-mode scheduling and the prod
Lambda schedule are two separate mechanisms hitting the same `runCycle()`).

Every tick, for every delivery in the snapshot:
- `QUOTED` past `quoteExpiresAt` → fires `QuoteExpired` + `episodeStore.markLost(...)`
  companion — the episode that's been sitting `open` since Flow D/F is now
  `lost`, and the truck is freed.
- `AWAITING_PAYMENT`, unconfirmed, past `paymentDueAt` → fires
  `PaymentOverdue` (delivery moves to `DISPUTED`).

No LLM anywhere in this path — pure time comparison and domain-guard-checked
firing. This is what keeps deliveries moving even if no human and no other
agent ever looks at them again.

---

## 12. Cross-cutting: what "firing a transition" always means

Every flow above that "fires" an action goes through exactly one mechanism,
`ActionPlanExecutor.execute(planner, companion)`:

1. Load a fresh `Snapshot` (`SnapshotLoader` — a consistent read of the
   board + fleet + active policy version, one point in time).
2. Run the pure planner (`desk.plan(...)`) against it → `Refused` /
   `UnknownRecord` / `Planned`.
3. If `Planned`: open a JDBC connection, `TRANSACTION_SERIALIZABLE`, apply
   every `StateWrite` (`UPDATE ... WHERE version = ?` — optimistic
   concurrency; zero rows affected means someone else won the race), insert
   the `operations_journal` row, run the companion write (episode row,
   idempotency row, decision trace — whatever that flow needs), commit.
4. `40001`/unique-violation on commit, or a version-guarded `UPDATE`
   affecting zero rows → `LOST_RACE` → rollback → re-plan from a **fresh**
   snapshot, up to `dakar.actions.max-attempts` (4) times, jittered capped
   backoff between attempts (`dakar.actions.backoff-{base,max}-millis`,
   25ms–250ms).
5. Every commit increments `dakar.actions.executed{action,actor}`; every
   lost race increments `dakar.actions.lost.race`; every guard refusal
   increments `dakar.actions.refused`.

This one function is why the double-claim-race drill (`ops/drills.sh`) always
produces exactly one winner: two concurrent callers of `execute()` racing to
claim the same truck will have one succeed and one hit `40001`/a zero-row
`UPDATE`, full stop — there's no other code path that writes truck/delivery
state.

---

## 13. Summary: what's genuinely autonomous vs. what needs a click

| Step | Autonomous today? | Notes |
|---|---|---|
| Delivery request | No — always an explicit `POST` (client or console) | |
| Dispatcher claims a truck | **Yes**, if a trigger reaches it (EventBridge in prod; `15s` schedule in dev; manual curl always works) | Reject-don't-repair validated |
| Negotiator drafts/fires a quote | **Yes in practice, but contradicts the documented design** — see §6. No dev-mode schedule at all; only fires on `delivery.matched` (prod) or manual curl | ⚠️ Needs a team decision, not a silent fix |
| Quote advice (no firing) | Always manual (`GET .../quote-advice`) | The actually-draft-only path |
| Client accepts | No — always an explicit `POST` | |
| Physical signals | **Never** autonomous — always a human click | No telematics exists |
| Quote expiry / payment overdue | **Yes**, always-on `30s` sweep regardless of provider/schedule flags | No LLM in this path |
| Replay audit | **Yes**, hourly (`dakar.audit.every`), `@Startup` so the metric exists from boot | Severity-1 alarm on any finding |

**One console session, left running, will show delivery status changing on
its own** for the Dispatcher and Coordinator paths without anyone touching
the keyboard — genuinely, not simulated. The Negotiator's autonomy is real
today but not where the docs say it should be; worth resolving (§6) before
it's described either way in a submission.

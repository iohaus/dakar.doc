# Dakar — Demo Video Shooting Script

**Target runtime:** 2:50–3:00 (hard cap 3:00 per submission rules)
**Assumption for this version:** every integration is live for real — AWS deployment reachable, CockroachDB Cloud connected, both MCP variants wired. If any one of these isn't actually true on recording day, see the fallback notes at the bottom before you record, not after.
Practice against this file with a stopwatch before recording — the organizers' own advice.

---

## One-sentence pitch (memorize this, it opens the video)

> "For a freight company, one wrong dispatch decision means a double-booked
> truck and a furious customer — Dakar hands that decision to AI agents, and
> makes it provably safe, because CockroachDB is the agents' entire memory,
> not just their database."

---

## Before you hit record — checklist

- [ ] AWS deployment confirmed live: `curl $API_ENDPOINT/v1/deliveries -X POST ...` succeeds end to end (delivery lands `RECEIVED`) against the real API Gateway URL, not `localhost`.
- [ ] Confirm the EventBridge chain fires **automatically** with no manual trigger call: create a delivery, watch `dakar-dispatcher-agent`'s CloudWatch log group for a fresh invocation within a few seconds, unprompted.
- [ ] `dakar-console` configured with `DAKAR_API_BASE_URL` pointed at the real API Gateway endpoint (not `localhost:8080`), logged in with the staff key, sitting on the Operations board.
- [ ] AWS CLI configured and authenticated in a visible terminal profile — you'll run `aws lambda get-function` and tail CloudWatch Logs live.
- [ ] Seed data: enough trucks registered, and enough prior rows in `negotiation_episode` on the demo lane/cargo that the vector search returns something real, not empty.
- [ ] `dakar.agents.provider=bedrock` / `dakar.embedding.provider=bedrock` confirmed on the deployed Lambdas (they are, per `CDKStack.java` — but confirm no override snuck in).
- [ ] Terminal font size bumped up, readable at 1080p, `jq` installed. Two terminal panes: **Pane A** = `aws logs tail --follow` on the agent log groups, **Pane B** = your command pane.
- [ ] A stopwatch or the recording software's own timer visible while you rehearse.

---

## Scene-by-scene script

Format per scene: **VISUAL** · **OVERLAY** (on-screen text) · **SCRIPT** (verbatim narration) · **ACTION** (exact commands/clicks).

### Scene 1 — Hook (0:00–0:15)

**VISUAL:** dakar-console Operations board, fleet of trucks sitting idle, calm.
**SCRIPT:**
> "For a freight company, one wrong dispatch decision means a double-booked
> truck and a furious customer. Dakar hands that decision to AI agents — and
> it's provably safe, because CockroachDB is where those agents keep their
> entire memory."

**ACTION:** none — sit on the board for these 15 seconds.

---

### Scene 2 — Straight into the live demo (0:15–0:30)

*The organizers explicitly grade "within 20–30 seconds." Cut straight to Pane B, no extra intro sentence.*

**VISUAL:** Pane B, a real `curl` command against the live API Gateway URL.
**OVERLAY:** `LIVE — deployed on AWS`
**SCRIPT:**
> "Watch. A shipper requests a delivery — twenty tons, Chicago to Memphis —
> straight against our live API."

**ACTION:**
```bash
curl -X POST $API_ENDPOINT/v1/deliveries \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: $(uuidgen)" \
  -d '{
    "clientId": "f4420db3-673a-45e1-badb-c9a617335dde",
    "tonnage": "20",
    "pickupAddress": "Chicago, IL",
    "dropoffAddress": "Memphis, TN",
    "cargoClass": "GENERAL",
    "deadlineAt": "2027-01-15T12:00:00Z"
  }' | jq
```
*(Plain city names, not `tools/demand_sim`'s fictional hub facility labels — those ship with exact lat/lon too, so the real service never has to geocode them; this request has no coordinates, so it's the one moment that genuinely exercises live geocoding, and it needs a name Nominatim can actually resolve.)*
Let the response (`"status": "RECEIVED"`, a real `deliveryId`) sit for a beat.

---

### Scene 3 — dakar-console: the real-world view (0:30–1:05)

*This is the main event, not a bookend. This is what a Dakar operator's actual day looks like.*

**VISUAL:** Cut to dakar-console. Use the command palette (`Ctrl+K`) to jump straight to the delivery just created — type a few characters of its ID, hit enter, land on its detail view. Nothing else touched by hand from here — just watch it update live as the backend works.
**OVERLAY:** `dakar-console — the operator's actual interface`
**SCRIPT:**
> "This is what our staff see — I didn't trigger anything else. Somewhere out
> there, EventBridge just noticed this new delivery and woke up an agent."

**ACTION:** `Ctrl+K` → paste/type the delivery ID → Enter. Then just wait and watch — within a few seconds the delivery's stage advances on screen as the Dispatcher claims a truck.

---

### Scene 4 — The Dispatcher wakes up, for real (1:05–1:35)

**VISUAL:** Split attention: Pane A (CloudWatch tail) shows a brand-new Lambda invocation appearing — cold start line, fresh `RequestId` — the instant dakar-console's board updates.
**OVERLAY:** `Amazon EventBridge → AWS Lambda (dakar-dispatcher-agent) → Amazon Bedrock, Claude 3 Haiku`
**SCRIPT:**
> "Nobody clicked a button to make that happen. EventBridge triggered a
> Lambda function that didn't exist a second ago, it asked Bedrock to choose
> a truck from exactly the ones it was offered, and the claim is already a
> transaction in CockroachDB."

**ACTION:**
```bash
aws logs tail /aws/lambda/dakar-staging-dispatcher-agent --since 1m --follow
```
Point at the cold-start / `REPORT RequestId:` lines as they scroll.

---

### Scene 5 — The Negotiator remembers the past (1:35–2:00)

**VISUAL:** dakar-console's quote-advice card populating live (retrieved episodes, offer band, rationale). Quick cut to a `cockroach sql` pane proving the real vector query underneath.
**OVERLAY:** `CockroachDB — Distributed Vector Search (VECTOR(1024), cosine index)`
**SCRIPT:**
> "Before it prices this load, the Negotiator searches its own memory for
> similar past negotiations — same database, same transaction boundary as
> everything else it just did. No separate vector store to keep in sync."

**ACTION:**
```bash
cockroach sql --url "$DAKAR_CLOUD_URL" -e "
  SELECT lane_key, opening_offer, final_price, outcome,
         1 - (embedding <=> (SELECT embedding FROM negotiation_episode ORDER BY random() LIMIT 1)) AS similarity
  FROM negotiation_episode
  WHERE lane_key = '<REAL CHICAGO-MEMPHIS LANE KEY>'
  ORDER BY similarity DESC LIMIT 5;"
```
*(the exact `lane_key` the backend generates for a Chicago→Memphis pair isn't confirmed — check it against real rows before recording, e.g. `SELECT DISTINCT lane_key FROM negotiation_episode LIMIT 20;`, rather than guessing a value live. The point on screen is the `<=>` operator and real similarity scores, seeded by P7-8's `demand_sim` run beforehand — same US geography throughout, nothing to reconcile.)*

---

### Scene 6 — Centerpiece: same code, zero shared memory, ever (2:00–2:35)

*Give this the most room. This is the direct answer to "show memory in action visibly," and it's the truest thing about the architecture — not a staged kill, the actual mechanics.*

**VISUAL:** Pane B: prove all four Lambdas are the identical artifact. Then Pane A: two genuinely separate CloudWatch log entries, different `RequestId`s, side by side.
**OVERLAY:** `Four Lambda functions. One function.zip. Zero memory between invocations.`
**SCRIPT:**
> "Here's the part that matters most. Dispatcher, Negotiator, Coordinator,
> the API itself — four different Lambda functions, but they're built from
> the exact same code."
> *(show the hash match)*
> "What makes each one behave differently is nothing more than which
> credentials and which role it's told to play. And every single invocation
> of any of them starts from absolute zero — no memory of the one before it,
> on purpose. That's not a limitation we work around. It's the whole point.
> The only reason this negotiation holds together at all is that every
> decision, the instant it's made, is already sitting in CockroachDB —
> not in a process that could disappear."

**ACTION:**
```bash
aws lambda get-function --function-name dakar-staging-dispatcher-agent --query 'Configuration.CodeSha256'
aws lambda get-function --function-name dakar-staging-negotiator-agent --query 'Configuration.CodeSha256'
```
Same hash, both outputs on screen together. Then cut to Pane A: two `REPORT RequestId: ...` lines from two different invocations — visibly different IDs, both correctly continuing the same delivery's story.

---

### Scene 7 — No double-booking, proven not promised (2:35–2:50)

**VISUAL:** Pane B, two concurrent claim attempts race for the same truck. Two outcomes land: one success, one refusal.
**OVERLAY:** `CockroachDB SERIALIZABLE transactions — enforced by the database, not the code`
**SCRIPT:**
> "And the one mistake this business can't afford — one truck, two
> deliveries — isn't prevented by careful code. It's structurally
> impossible."

**ACTION:**
```bash
./ops/drills.sh --only double-claim-race
```
Let both outcomes print clearly — one committed, one refused.

---

### Scene 8 — Name everything, close (2:50–3:00)

**VISUAL:** Fast text card, then back to dakar-console's board.
**OVERLAY:**
```
CockroachDB — SERIALIZABLE transactions · Vector Search · MCP Server (read-only, audited)
AWS — Lambda · API Gateway · EventBridge · Bedrock (Claude 3 Haiku + Titan Embeddings V2)
```
**SCRIPT:**
> "That's Dakar — real AWS infrastructure, real agents, and memory that
> survives anything. Thanks for watching."

**ACTION:** none.

---

## If you're running long: cut in this order

1. **Scene 7** (double-booking race) first — Scenes 4–6 already prove memory is real.
2. **Scene 6**'s CLI hash-check — keep the spoken line, cut the two `aws lambda get-function` calls to one quick screen if time is tight.
3. Scene 5: don't read the JSON/SQL output aloud, let it sit on screen while you talk over it.

**Never cut:** Scene 1 (required pitch), Scene 2 (20–30s live-demo rule), Scene 6 (the actual thesis of the project).

---

## Fallback notes — read before recording, not after

- **If the AWS↔CockroachDB connection isn't confirmed live yet**, do not record this version. Use the local-stack version of this script instead (git history / ask me to regenerate it) — a demo that stalls on camera is worse than an honest, fully-local one. Confirm Scene 2's `curl` against `$API_ENDPOINT` succeeds *before* you start recording anything else.
- **If EventBridge auto-triggering is flaky or slow in practice**, don't let dead air run in Scene 3 — have the manual trigger commands (`POST /internal/trigger/dispatcher` / `.../negotiator` with `X-Agent-Trigger-Key`) ready as a silent fallback, cut to them without announcing you're doing so.
- **The CockroachDB MCP Server didn't get its own dedicated scene on purpose** — it's operator/dev tooling, architecturally separate from how the agents themselves read and write memory (`agentic.md`'s explicit boundary). It's named correctly in Scene 8 as what it actually is, not dramatized as something it isn't.
- **Rehearse Scene 6 exactly as written**, including landing on the matching hash before you speak the punchline — the reveal only lands if the proof is already on screen when you say it, not after.

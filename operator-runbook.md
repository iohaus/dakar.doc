# Dakar — Operator Runbook

**Status:** v1.0 (P6-2) — the query set and alarm sheet for operating Dakar
**Audience:** operators with database read access; privileged doc (Law 4 applies — none of this vocabulary reaches the repo or clients)
**Surfaces:** locally, `cockroach sql` against `ops/docker-compose.yml` (console on `:8090`); in the cloud, the **read-only, audited MCP Server** configured from the Cloud Console (⛔ until cloud work is authorized). Operators never write; every mutation goes through the desk.

---

## 1. The live board

```sql
-- Fleet at a glance
SELECT truck_id, status, capacity_tons, assigned_delivery, version, updated_at
FROM truck ORDER BY status, updated_at DESC;

-- Work in flight
SELECT delivery_id, status, tonnage, agreed_price, payment_confirmed,
       quote_expires_at, payment_due_at, updated_at
FROM delivery WHERE status NOT IN ('COMPLETED', 'CANCELLED')
ORDER BY updated_at DESC;
```

## 2. Stuck work (the F1/F2 promises, checked by hand)

```sql
-- Deliveries that have not moved in 6 hours (tune to taste)
SELECT delivery_id, status, updated_at FROM delivery
WHERE status NOT IN ('COMPLETED', 'CANCELLED')
  AND updated_at < now() - INTERVAL '6 hours'
ORDER BY updated_at;

-- The dispute queue, oldest first (staff must act on these)
SELECT delivery_id, payment_due_at, updated_at FROM delivery
WHERE status = 'DISPUTED' ORDER BY updated_at;

-- Quotes past their validity that the coordinator has not yet expired
SELECT delivery_id, quote_expires_at FROM delivery
WHERE status = 'QUOTED' AND quote_expires_at < now();
```

## 3. Refusal & decision triage

```sql
-- What ran, by whom, lately
SELECT action, actor, count(*) FROM operations_journal
WHERE recorded_at > now() - INTERVAL '24 hours'
GROUP BY action, actor ORDER BY count(*) DESC;

-- One delivery's full history
SELECT recorded_at, action, actor, payload FROM operations_journal
WHERE delivery_id = $1 ORDER BY recorded_at;

-- Automation decisions and their reasoning
SELECT created_at, actor, trace FROM decision_trace
ORDER BY created_at DESC LIMIT 50;
```

## 4. Episodic & semantic memory health

```sql
SELECT outcome, count(*) FROM negotiation_episode GROUP BY outcome;
SELECT count(*) AS profiles FROM client_profile;
SELECT lane_key, median_price_per_ton, episode_count FROM lane_stat
ORDER BY episode_count DESC LIMIT 20;
```

## 5. Metrics & alarm sheet (`/q/metrics`, Prometheus)

| Metric | Meaning | Alarm threshold |
|---|---|---|
| `dakar_actions_executed_total{action,actor}` | committed actions | drop to 0 during business hours → investigate |
| `dakar_actions_refused_total` | rule refusals | sustained spike → misbehaving caller/policy or bad pricing data |
| `dakar_actions_lost_race_total` | serialization retries | sustained growth → contention hot spot |
| `dakar_replay_divergence_total` | journal cannot account for live state | **> 0 pages immediately (severity-1)** |
| `dakar_disputes_open` | dispute queue depth | above staffing capacity → escalate |
| `dakar_disputes_oldest_seconds` | oldest unresolved dispute | > 2× payment terms → escalate |

Every HTTP response and log line carries `X-Request-Id` / `{requestId}` — quote it in incident notes.

## 6. Drills

`ops/drills.sh` re-runs the failure-mode evidence suite (double-claim race, stale plan, below-floor offer, companion rollback, policy outage, journal tamper, policy rollback). Run it after any change to the executor, the desk rules, or the policy table.

## 7. Replay audit findings

The audit runs every `dakar.audit.every` (default 1h) and logs `replay divergence:` lines at ERROR. A finding means one of: a write bypassed the desk (check for manual SQL), a journal row is corrupt, or — worst — an executor bug. Freeze writes, snapshot the database, and treat as severity-1. Trucks seeded outside the journal (dev-only `Db.seedTruck`) will legitimately appear as findings; production fleets must be registered through `POST /v1/trucks`.

<!-- day-nav -->
[← Day 60 — Nearby](60-nearby.md) · [Day 62 — Metrics ingestion →](62-metrics-ingestion.md)

# Day 61 — Ticket booking

**Do now**

1. Set a timer.
2. Attempt the problem. Stop at the attempt line. Do not scroll.
3. Then read.

## Time box

40 minutes.

| Minutes | Do this |
|---|---|
| 0–12 | Attempt. Assigned seats. Hold with TTL. No venue-wide lock. No oversell. |
| 12–32 | Read. Day 49 was fungible units. Today each seat is unique. Day 70 is the flash queue — not required if arrival fits. |
| 32–40 | Say how a seat is held, what the map read sees, and how payment boundaries work. |

## Intent

Facing assigned seats, leave able to hold specific seats with a TTL so you neither oversell nor lock the whole venue.

## Problem

> Design ticket booking for assigned seats.
>
> A venue has a seating map. A buyer picks specific seats and holds them while checking out. Two buyers must not get the same seat. Holds expire.

## Attempt before reading

12 minutes. Do not scroll.

Write:

1. Seat states: available, held, sold.
2. Hold API: multi-seat atomicity in one order.
3. Map read: can it be slightly stale?
4. Hot seat (front row) concurrency — conditional updates vs venue lock.
5. Relation to day 49 inventory and day 70 flash sale.

---

**Stop. Per-seat conditional hold. Order groups seats. Map is cacheable with care.**

---

## Requirements

**In.** Events with seat maps (**1k–50k** seats). Hold **1–8** seats for **5–10 minutes**. Commit on paid (payment outside — day 65). Release on cancel/expiry. Idempotent holds.

**Assumptions.** **100** events on sale; peak **2,000** hold attempts/s on a hot event; one section hotter. One region.

**Out.** Dynamic pricing engine, resale marketplace, ADA complexity beyond a seat attribute, flash waiting room unless numbers demand (day 70).

## Estimates

50k seats × 100 events = 5M seat rows — small. Correctness is hot-row contention on popular seats, not storage. Hold attempts 2k/s; each touches 1–8 rows in one transaction on the event's seat shard.

## API and data

`POST /v1/events/{id}/holds` `{seat_ids:[], idempotency_key}` → 201 hold_id, expires_at or 409 with conflicts.

`POST /v1/holds/{id}/commit` / `cancel`.

`GET /v1/events/{id}/map` → seat statuses (may lag **1–2 s**).

Seat row: `(event_id, seat_id)`, state, hold_id, expires_at. Partition by `event_id`.

## Design

### Hold transaction

On event leader/shard: for all seat_ids, conditional update `available → held` with hold_id and expiry **only if available**. If any fails, roll back all (transaction) and 409 listing conflicts. Insert hold row + idempotency.

**Refuse** locking the whole venue row. **Refuse** checking seats in app then writing — race.

### Map reads

Cache map snapshots **1 s** or stream invalidations per section. Stale "available" that 409s on hold is OK. Stale "held" that blocks another buyer briefly is false scarcity.

### Sweeper

Same as day 49: expire holds → available. State guard vs commit race.

### Payment

Reserve/hold → pay → commit. Hold TTL > payment budget. Crash window: paid but uncommitted → recovery commits (day 49 payment paragraph).

### vs day 49 / 70

Fungible qty vs unique seats. Flash queue when hold QPS or fairness needs it — not default at 2k/s.

## Diagrams

```mermaid
sequenceDiagram
  participant B as Buyer
  participant L as Event shard
  B->>L: hold seats A1,A2 key=K
  Note over L: txn: both available or neither
  alt conflict
    L-->>B: 409
  else
    L-->>B: 201 hold expires
  end
```

Caption: "All-or-nothing on the named seats. No venue lock."

## Failure the user sees

**Two buyers one seat.** Second 409. Map may still show available briefly.

**Leader down.** 503 holds, not 409 empty venue.

**Sweeper vs pay.** Same false scarcity / commit rules as day 49.

## Trade-offs

**Atomic multi-seat vs best-effort partial.** Atomic — partial holds anger users mid-checkout.

**Exact map vs cached.** Cached with seconds of lag.

## Talking points

**If they lock the venue.** "Hot seat must not stop the balcony."

**If they salt seats.** "Seat identity is the key; salting breaks uniqueness."

## Say this in the room

Assigned seats are conditional updates on `(event, seat)` rows inside one transaction so a hold of A1 and A2 is all-or-nothing — not a venue-wide lock and not an app-level check-then-set. Holds expire with a sweeper and a state guard against commit, same shape as warehouse reservation but the SKU is a unique seat. The map can lag a second and show false availability that becomes 409 on hold. Payment stays outside; the hold must outlive the payment budget so we never release seats we already charged for.

### Staff depth: unique seats vs fungible qty

Day 49 increments a counter; day 61 CAS-updates named seat rows in one transaction (all-or-nothing for the selection). Venue-wide lock is forbidden. Map lag → false availability → 409 on hold is OK; false "sold" briefly is false scarcity.

**Payment boundary.** Hold TTL > pay budget; recovery commits if paid (day 49/65). Flash queue only if arrivals exceed ~hundreds/s on the hot event (day 70).

**What staff sounds like.** Conditional multi-seat txn, sweeper vs commit guard, 503 not 409 on leader loss.

## Kit artifact

| Follow-up | You answer with |
|---|---|
| "Same seat twice?" | Conditional multi-seat txn; 409; no venue lock. |

## Design log

One line: how multi-seat atomicity worked in your attempt, and how it differed from day 49.

Next: [Day 62 — Metrics ingestion](62-metrics-ingestion.md). Firehose with cardinality limits.

---

<!-- day-nav -->
[← Day 60 — Nearby](60-nearby.md) · [Day 62 — Metrics ingestion →](62-metrics-ingestion.md)

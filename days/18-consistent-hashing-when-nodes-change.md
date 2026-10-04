<!-- day-nav -->
[← Day 17 — Rate limits and abuse](17-rate-limits-and-abuse.md) · [Day 19 — Backpressure and load shedding →](19-backpressure-and-load-shedding.md)

# Day 18 — Consistent hashing when nodes change

**Do now**

1. Set a timer.
2. Attempt the problem. Stop at the attempt line. Do not scroll.
3. Then read.


## Time box

35 minutes.

| Minutes | Do this |
|---|---|
| 0–10 | Attempt. A node joins or leaves. What fraction of keys move, under modulo and under a ring. |
| 10–28 | Read. If you hashed the app tier, undo it. The app tier is not the ring. |
| 28–35 | Say when a fixed slot map is the simpler tool, in one sentence you could defend. |

## Intent

Facing cache or storage nodes that come and go, leave able to use consistent hashing and say when a fixed slot map is simpler. The ring is a fix for a membership change that would otherwise dump the keyspace onto the primary. It is not a default for every tier.

## Problem

> Design a pastebin. People paste text and share a link.

The interviewer says: "You have one cache node. You need several, and you will replace them. What happens to the keys when the count changes?"

## Attempt before reading

10 minutes. Do not scroll. The cache holds metadata by paste id, TTL jittered around a minute, tombstones on delete. The primary's read ceiling is about 15,000/s. Four app processes sit behind least connections. They are not keyed by paste id.

Write:

1. What `hash(id) % N` does when N goes from 4 to 5. A fraction is enough; derive it if you can.
2. What a ring does to that fraction when one node joins.
3. Whether a lost cache node loses pastes, or only hit rate.
4. The tier you would **not** put on a ring (the app processes, the single primary).

---

**Stop. The ring is for the cache, below. Durability does not live on it.**

---

## Diagrams

### What moves

```mermaid
flowchart LR
  keys[Paste ids on a ring]
  keys --> n1[Node A about 1/N]
  keys --> n2[Node B about 1/N]
  keys --> n3[Node C about 1/N]
  dead[Node B leaves] -.->|only B's arc misses| n3
```

### What is not on the ring

```mermaid
flowchart TB
  ring[Ring: cache and limiter keys]
  lb[Least connections: app processes]
  one[One primary: not hashed]
  bucket[Object store: not your ring]
  ring --- lb
  ring --- one
  ring --- bucket
```

## Requirements

The cache is not durable. A missing entry is a miss, and the miss path is the primary, which is correct and slow. Any hashing scheme you pick is allowed to drop entries on the floor. It is not allowed to turn a small membership change into a **full** miss storm. That storm is day 11's stampede, multiplied by the fraction of keys that moved, aimed at a primary you already said melts around 15,000 point reads/s.

Today's origin read rate, once the CDN is up, depends on a hit rate you still must not treat as measured. The worst case you design the ring for is the one day 10 named: cache cold or cache dying, and the primary sees the read load that reaches origin. If that load is the full 17,400, moving **all** keys is a melt. Moving a quarter of them might not be. The scheme is how you keep "a node died" inside the quarter, not the whole set.

Apps stay on least connections. A paste id must not pick an app. You settled that on day 9, because there is no local cache on the app and a hot id would pin one process. Do not reopen it now that you know the word "hash."

The bucket is not your ring either. You do not operate object-storage nodes. The key `pastes/{id}` is an identifier inside a service that places bytes itself. Consistent-hashing the bucket is role-play.

## Design

**Cache nodes own id ranges on a ring.** Hash the paste id onto a circle. Each node is a point on the circle (in practice, many virtual points).

With equal shares, one node out of N owns about **1/N** of the keys. Add a fifth node to four: about **1/5** of the keys move, not 4/5.

**Virtual nodes** so one physical box is many points on the ring. A planning number: on the order of **100 virtual nodes** per physical node, not 2 and not a million.

Replication factor: **none for correctness.** A second cache copy of the metadata does not make it more true; the primary is the truth. A death costs you about 1/N misses, once, until the TTL would have expired anyway.

### How big is 1/N, in this product

You do not need many cache nodes for RAM. Even a generous hot set, a few million keys, is a handful of gigabytes (you will pin this assumption on day 21). **One node can hold it.** The ring is not here because you are out of memory at 1×.

It is here because:

- You will run more than one node so a single cache process is not a total cold start (day 10: cache down throws reads at the primary).
- Replacing a node, or autoscaling one more, must not remap the world.
- The limiter keys from day 17 live here too. Remapping them resets budgets. Moving 1/N of the limiter keys is a brief extra burst from those IPs, which you can live with. Moving all of them is a global burst. Another reason modulo is the wrong default.

Start with **4** cache nodes as a number you can divide. Losing one moves about **25%** of the keys. If origin is seeing, say, a few thousand reads a second in a bad hour, a quarter of that as extra misses is uncomfortable but inside a 15,000 ceiling.

### When a fixed slot map is simpler

A fixed map (a table of slot → node, slots pre-declared, classically many thousands) moves the same kind of fraction if you reassign only the slots of the dead node. It is not modulo. It is "explicit ranges."

Use the fixed map when **people** change membership: a runbook says "move slots 0–999 to the new box," the move is visible, and you can pause it. That is the better tool for a primary you are sharding, where the data is durable and a surprise remap is data loss or a double writer. You are not sharding the primary today.

Use the ring when membership changes without a ceremony: a cache node's replacement, a scale event, an instance that disappears at 3 a.m. The app recomputes the owner from the current member list.

Modulo is the tool you use when N is fixed forever. N is not fixed forever. So you do not use modulo.


## Trade-offs

**Choice.** Consistent hashing with virtual nodes for the cache. Apps compute the owner from a versioned member list.

**What you give up.** A member list that can be briefly wrong, and a spread that is only as even as your virtual nodes. You take a cold arc of about 1/N on every loss, which the primary must absorb as misses.

**10× break.** The hot set grows, and 1/N of a much larger origin QPS may exceed 15,000 misses when a node dies. Then you add nodes (smaller arcs) before you add replication, because replication of a cache brings the tombstone dual-write.

## Say this in the room

Modulo 4 to modulo 5 moves almost every key. A ring moves about one over N. I only need that for the cache, which is allowed to miss. Four nodes, one dies, about a quarter of the entries miss once and refill from the primary. I do not replicate cache entries.

## Kit artifact

One checklist row.

| Row | Lock |
|---|---|
| Membership | Ring or fixed slots, with the fraction that moves. Modulo only if N never changes. Say whether a miss loses data or only a hit. |

## Design log

One line: the fraction of keys your attempt moved when one node left, and whether that store was allowed to miss. If the fraction was "all of them," the gap is modulo.

Next: [Day 19 — Backpressure and load shedding](19-backpressure-and-load-shedding.md).

---

<!-- day-nav -->
[← Day 17 — Rate limits and abuse](17-rate-limits-and-abuse.md) · [Day 19 — Backpressure and load shedding →](19-backpressure-and-load-shedding.md)

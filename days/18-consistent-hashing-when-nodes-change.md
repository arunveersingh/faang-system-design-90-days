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

## Requirements

The cache is not durable. A missing entry is a miss, and the miss path is the primary, which is correct and slow. Any hashing scheme you pick is allowed to drop entries on the floor. It is not allowed to turn a small membership change into a **full** miss storm. That storm is day 11's stampede, multiplied by the fraction of keys that moved, aimed at a primary you already said melts around 15,000 point reads/s.

Today's origin read rate, once the CDN is up, depends on a hit rate you still must not treat as measured. The worst case you design the ring for is the one day 10 named: cache cold or cache dying, and the primary sees the read load that reaches origin. If that load is the full 17,400, moving **all** keys is a melt. Moving a quarter of them might not be. The scheme is how you keep "a node died" inside the quarter, not the whole set.

Apps stay on least connections. A paste id must not pick an app. You settled that on day 9, because there is no local cache on the app and a hot id would pin one process. Do not reopen it now that you know the word "hash."

The bucket is not your ring either. You do not operate object-storage nodes. The key `pastes/{id}` is an identifier inside a service that places bytes itself. Consistent-hashing the bucket is role-play.

## Design

**Cache nodes own id ranges on a ring.** Hash the paste id onto a circle. Each node is a point on the circle (in practice, many virtual points). A key is stored on the first node clockwise from its hash. When a node leaves, only the keys that mapped to it move, to the next node clockwise. When a node joins, it takes the keys that now fall between it and its predecessor. Everyone else stays.

With equal shares, one node out of N owns about **1/N** of the keys. Add a fifth node to four: about **1/5** of the keys move, not 4/5. That is the whole trick. You do not need to derive the hash function. You need the fraction, and you need to say the old keys are misses, not migrations you must complete. Because this is a cache, "move" means "the next read misses and fills the new owner." You do not copy RAM from the old node. If the old node is dead, there is nothing to copy.

**Virtual nodes** so one physical box is many points on the ring. Without them, unlucky hashes pile half the keyspace on one machine. With them, the share is close to even. A planning number: on the order of **100 virtual nodes** per physical node, not 2 and not a million. The map of virtual nodes has to fit in memory on every app, because the app computes the owner. It is a small table. Do not put a network call on the read path just to ask who owns a key.

Replication factor: **none for correctness.** A second cache copy of the metadata does not make it more true; the primary is the truth. A replica on the clockwise neighbor would save hit rate when a node dies (the neighbor might already have the entry). It also doubles memory and makes invalidation write to both, and a tombstone must reach every copy or the stale one resurrects the paste. At this size you skip cache replication. A death costs you about 1/N misses, once, until the TTL would have expired anyway. Say the cost. Do not buy a second consistency problem to avoid a cold quarter.

### How big is 1/N, in this product

You do not need many cache nodes for RAM. A metadata entry is a few hundred bytes. Even a generous hot set, a few million keys, is a handful of gigabytes (you will pin this assumption on day 21). **One node can hold it.** The ring is not here because you are out of memory at 1×.

It is here because:

- You will run more than one node so a single cache process is not a total cold start (day 10: cache down throws reads at the primary).
- Replacing a node, or autoscaling one more, must not remap the world.
- The limiter keys from day 17 live here too. Remapping them resets budgets. Moving 1/N of the limiter keys is a brief extra burst from those IPs, which you can live with. Moving all of them is a global burst. Another reason modulo is the wrong default.

Start with **4** cache nodes as a number you can divide. Losing one moves about **25%** of the keys. If origin is seeing, say, a few thousand reads a second in a bad hour, a quarter of that as extra misses is uncomfortable but inside a 15,000 ceiling. `hash % 4` becoming `hash % 3` when a node dies moves on the order of **most** keys (any key whose hash mod 4 differs from hash mod 3 — about three quarters). Three quarters of a cold-ish origin, or of the full 17,400 if the CDN is also cold, is the melt. That is the break this box fixes. Not the steady-state QPS.

### When a fixed slot map is simpler

A fixed map (a table of slot → node, slots pre-declared, classically many thousands) moves the same kind of fraction if you reassign only the slots of the dead node. It is not modulo. It is "explicit ranges."

Use the fixed map when **people** change membership: a runbook says "move slots 0–999 to the new box," the move is visible, and you can pause it. That is the better tool for a primary you are sharding, where the data is durable and a surprise remap is data loss or a double writer. You are not sharding the primary today.

Use the ring when membership changes without a ceremony: a cache node's replacement, a scale event, an instance that disappears at 3 a.m. The app recomputes the owner from the current member list. There is no reshard to finish. The cost is imbalance if you skip virtual nodes, and a member-list you must agree on. Agreement can be a small configuration the apps poll, or a gossip story you should not invent. **Configuration with a version is enough.** If two apps briefly disagree on membership, a key is read and written on two nodes for a moment: a miss or a stale tombstone. The TTL bounds it. For a cache, that is acceptable. For the primary, it would be split brain, which is why the primary is not on this ring.

Modulo is the tool you use when N is fixed forever. N is not fixed forever. So you do not use modulo.

### The fractions, derived, and what they cost the primary

**Modulo, derived in one line.** A key stays put only if `h mod 4` equals `h mod 5`. Over any run of 20 consecutive hashes that happens for 4 of them. So 4 → 5 keeps 20% and moves **80%**. Going down, 4 → 3, it holds for 3 of every 12: **75%** move. In general, N to N+1 under modulo keeps only 1/(N+1). The more nodes you have, the worse a single change is. That is the sentence that ends the modulo discussion.

**The miss storm, in this product's numbers.** Take the worst case you designed for: 17,400 reads a second reaching the cache tier, and the 57% hit rate day 10 said you wanted. The primary sees about 7,500. Lose one of four nodes on a ring: a quarter of the hits become misses, about 17,400 × 0.57 × 0.25 ≈ 2,500 extra a second. The primary goes to about 10,000, under 15,000, and decays back as the arc refills, hot keys in the first second, the tail within a TTL. Lose one under modulo: three quarters of the hits miss, about 7,400 extra, and the primary sits near 14,900 for the better part of a minute. One is a bump. The other is a page, at a moment you did nothing wrong except replace a box.

**Virtual nodes, as a spread.** With one point per node, arc sizes are random and one of four nodes can easily own twice its share. With V points per node, the spread of shares shrinks roughly like 1/√V: at 100 virtual nodes, shares land within about ±10% of even. That is why "about 100" is the planning number. Ten leaves a node at plus or minus a third. A thousand buys almost nothing more and makes the member table bigger on every app.

**A ring is not the only answer.** Rendezvous hashing scores every node for each key and picks the highest. It moves the same 1/N on a change, needs no virtual nodes, and at four to a few dozen nodes the per-lookup cost is nothing. If the interviewer prefers it, agree; the graded idea is "a change moves about 1/N, and only on a tier allowed to miss," not the shape of the circle.

**Adding a node is a miss event too.** A new node joins empty and takes about 1/(N+1) of the keys, which all miss on their next read. Scaling the cache out during a primary incident adds misses at the worst moment. Add cache capacity when the primary has headroom, not as the first reaction to the primary being hot.

What staff sounds like here is pricing the membership change on the primary, not on the cache. The cache is allowed to lose entries; the question is how many extra reads a lost node sends to the one store that cannot be scaled by adding a box. "A quarter of the hits, about 2,500 a second, decaying within a TTL" is the answer the interviewer can push on. "Consistent hashing minimizes movement" is a definition.

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

Caption it: "One node out of four leaves: a quarter of the hits miss once, about 2,500 a second extra at the primary." Write the number on the dotted arrow. The drawing exists to show the size of the cold arc, so put its size on it.

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

Caption: "Only the tier that is allowed to miss sits on the ring." Say the reason for each off-ring box out loud: apps are unpinned, the primary has one writer, the bucket is not yours to place.

## Failure the user sees, when cache membership changes

**A cache node dies.** Readers of ids on that arc get a slower read once, a primary round trip, then hits again. The primary takes about 2,500 extra reads a second at the worst-case load, decaying within a minute. Users notice nothing unless the primary was already near its ceiling. Limiter buckets on that arc reset, so a few IPs get a fresh burst.

**Two apps disagree on membership for a few seconds.** A delete writes its tombstone to the new owner; an app still on the old list reads the old owner, which holds the live entry, and serves the deleted paste. Bounded by the time it takes the new version to reach every app, plus at worst the entry's TTL. Say it as the ring's version of the tombstone race: rare, short, bounded.

**Autoscaling flaps.** Nodes join and leave every few minutes. Every change cold-starts an arc. The primary sees a sawtooth of extra misses all afternoon and nothing looks broken in any single graph. Users see intermittent slow reads. Page on member-list version changes per hour, not only on miss rate.

**Someone deploys modulo by accident.** One node replacement turns into a three-quarter miss storm at the primary, near its ceiling for a minute. Reads slow everywhere, creates slow with them. The postmortem says "cache incident"; the cause was arithmetic.

## Trade-offs

**Choice.** Consistent hashing with virtual nodes for the cache. Apps compute the owner from a versioned member list. No cache-to-cache replication. No modulo. Primary and app tier stay off the ring.

**Alternative.** `hash(id) % N`, or a single cache node so the question goes away, or a fixed slot map you reshard by hand.

**What you give up.** A member list that can be briefly wrong, and a spread that is only as even as your virtual nodes. You give up the simplicity of one cache process. You take a cold arc of about 1/N on every loss, which the primary must absorb as misses. Singleflight still collapses a hot key; it does not collapse a random quarter of the keyspace. A random quarter is many distinct ids, so it is many primary reads. That is the cost you sized above.

**When the fixed map wins.** The moment the data is the primary and a moved key has to land exactly once. Then you want slots, a visible move, and one writer per slot. The ring's "miss and refill" story is false there: a miss is not a refill, a miss is a lost row. Do not "consistent-hash the database" with the cache's failure semantics.

**Name the refusal inside each alternative.** Against modulo: you refuse to turn every replacement into a 75 to 80% miss storm on a primary with no slack. Against one cache node: you refuse to make every restart a full cold start. Against the fixed slot map for the cache: you refuse a human ceremony for a tier that changes membership without one. Against cache replication: you refuse a second tombstone write to save a cold quarter that the primary can absorb. Each refusal names the miss fraction or the consistency problem it would cost.

**10× break.** The hot set grows, and 1/N of a much larger origin QPS may exceed 15,000 misses when a node dies. Then you add nodes (smaller arcs) before you add replication, because replication of a cache brings the tombstone dual-write. If the member list itself flaps — autoscaling oscillating — you will cold-start arcs all afternoon. The fix is damping scale events, not a better hash. Flapping is an operations break that looks like a hashing break.

## Talking points

**Say.** "Modulo 4 to modulo 5 moves almost every key. A ring moves about one over N. I only need that for the cache, which is allowed to miss. Four nodes, one dies, about a quarter of the entries miss once and refill from the primary. I do not replicate cache entries. I do not hash the app tier; least connections stays. I do not hash the primary."

**Say.** "A fixed slot map is what I'd want if this were durable data and a human was moving a range. The cache changes membership without a human, so the ring is the smaller tool. Virtual nodes, on the order of a hundred per box, so one owner doesn't take half the ids by luck."

**Hand-waving.** "We'll consistent-hash everything." Everything includes tiers where a miss is a lie or a loss. Name the tier.

**Hand-waving.** "The hash ring gives us durability." It gives you a stable miss fraction. Durability is still the primary's replica and the bucket.

**Hand-waving.** "Virtual nodes solve hot keys." They balance **ids**, not **traffic**. One viral id is still one id, on one node, at 8,700 reads/s. The ring does not split a single key. Day 11's singleflight and day 15's CDN are what split that load. If you hash a suffix onto the hot key to spread it, you have salting, which is a later phase, and you have broken "the id is the cache key" unless every read knows the suffix scheme. Do not salt today.

**If they ask how many keys modulo moves.** "A key stays only if the two remainders match. Four to five, that's 4 of every 20: 80% move. Four to three, 75%. On a ring, about a quarter for a loss and a fifth for a join."

**If they ask what a dead cache node does to the primary.** "At the worst-case load and a 57% hit rate, about 2,500 extra reads a second, so roughly 7,500 becomes 10,000 against a 15,000 ceiling, decaying within a TTL. Under modulo it's about 7,400 extra, which puts the primary near the ceiling."

**If they suggest rendezvous hashing instead.** "Fine with me. Same 1/N movement, no virtual nodes, trivial cost at this node count. What I'm defending is 'a change moves about 1/N on a tier allowed to miss,' not the circle."

**If they ask where the member list lives.** A versioned config the apps refresh. Not a consensus protocol you build in the interview. Disagreement is a short double-place of a cache entry, bounded by TTL.

**If they ask what you page on.** Miss rate jumping by about 1/N when no node was supposed to leave. Member-list version flapping. One cache node at a much higher CPU than the others, which means virtual nodes are too few or a single key is hot. The second case is not fixed by adding nodes.

## Say this in the room

Modulo four to five keeps a key only when the remainders match, 4 of every 20, so 80% move; four to three, 75%. A ring with about a hundred virtual nodes per box moves about one over N and keeps shares within roughly ten percent. I only need that for the cache, which is allowed to miss. Four nodes, one dies: a quarter of the hits miss once. At the worst case I designed for, that's about 2,500 extra reads a second, 7,500 to 10,000 on a 15,000 primary, decaying within a TTL. Modulo would put it near the ceiling. Apps compute the owner from a versioned member list; a few seconds of disagreement can serve a deleted paste, bounded by the TTL. No cache replication: the primary is the truth and a second copy doubles the tombstone write. Apps stay on least connections, the primary has one writer, and the bucket places its own bytes. Virtual nodes balance ids, not traffic; the viral id is still one key on one node.

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

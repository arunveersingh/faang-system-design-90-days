<!-- day-nav -->
[← Day 10 — Cache-aside for the melted read](10-cache-aside-for-the-melted-read.md) · [Day 12 — SQL or NoSQL from the broken query →](12-sql-or-nosql-from-the-broken-query.md)

# Day 11 — Hot-key stampede

**Do now**

1. Set a timer.
2. Attempt the problem. Stop at the attempt line. Do not scroll.
3. Then read.

## Time box

35 minutes.

| Minutes | Do this |
|---|---|
| 0–10 | Attempt. One hot id, the moment the cache entry dies, what hits the primary. |
| 10–28 | Read. If your fix is "lower the TTL," you made the stampede more frequent. |
| 28–35 | Say, from memory, the difference between a TTL and an invalidation, and one stampede control that is not a TTL. |

## Intent

Facing a hot pastebin key, leave able to stop a stampede without treating a TTL as invalidation. Popularity is still a point read. The failure is many misses at the same instant, not a hard key range.

## Problem

> Design a pastebin. People paste text and share a link.

The interviewer says: "One paste just got a large fraction of your reads. The cache entry expires. What happens to the primary in that second?"

## Attempt before reading

10 minutes. Do not scroll. Cache-aside of metadata is in place. TTL is 60 seconds. Peak reads are about 17,400/s. The primary's planning ceiling is about 15,000 point reads/s.

Write:

1. A concrete hot key. Pick a fraction of peak, one id, and the QPS that implies.
2. What the primary sees in the second the entry expires, if every app misses independently.
3. A control that collapses those misses. Say whether it is per process or per cluster.
4. How delete actually removes the entry. If the answer is "the TTL," start over.

---

**Stop. Stampede control below. TTL stays a bound, not a delete.**

---

## Requirements

Same contract. One id is still the only lookup. A viral paste is the easy disk shape from day 5: one row, one file, page cache on the NVMe. It is the hard **cache** shape, because a single key's expiry is a synchronization point for every reader.

You do not add a ranking page, a counter table, or a "trending" product to handle this. A view counter would be a new write hot spot. Non-goal, unless they force it, and then it is async and lossy and not on the read you are serving.

Assumption for this hour, spoken: **one id takes half of peak reads.** Half of 17,400 is **8,700 reads/s on one key.** The rest of the keyspace behaves as before. You can change the fraction; the shape of the answer should not change.

## Design

### What a stampede is

The hot entry is present. Those 8,700 reads/s hit the cache and then the NVMe (the body is not in the cache). The primary does not see them. Good.

At TTL expiry, the next requests miss. You have four app processes. Each will, without help, send its own lookup to the primary and its own body read. That is only four misses if they are perfectly coalesced by accident. They are not. In the same few milliseconds you can have hundreds of misses in flight, because 8,700/s means a new request about every 0.1 ms, and a primary round trip is a millisecond or more. Hundreds of identical point reads land on one row at once. The row is cached in Postgres's buffer pool, so you might survive **this** one key. The failure mode the interviewer is allowed to push: the hot key expires **and** a deploy just restarted the cache, **or** you have many keys expiring together because you set them all with the same TTL aligned to a clock. Then the primary sees a cliff, not one row.

Aligned expiry is self-inflicted. If you fill every entry with `TTL = 60` from a cron-shaped warmup, you built the stampede. Jitter is the first control, and it is not sufficient by itself.

### TTL is not invalidation

Say these as different mechanisms.

| Mechanism | What it is for | What it is not |
|---|---|---|
| TTL plus jitter | Bounds memory. Bounds how long a bug or a missed delete can serve stale metadata. Spreads expiry so keys do not fall off together. | Not a delete. Not how expiry of the **paste** works. The paste expires because the read checks `expires_at`. |
| Tombstone on delete | Makes the origin stop serving a removed paste, and blocks the late-fill race from day 10. | Not a memory policy. A tombstone has its own short TTL so the key does not live for 45 days. |
| Paste `expires_at` | The product deadline. Checked on every hit and every miss. | Not a cache setting. |

If you "invalidate" by setting a short TTL and waiting, a delete is invisible for that whole TTL, and a viral paste multiplies the leak. The hot key is exactly the paste you most need to be able to pull.

### Controls that actually collapse a miss

You want **one** fill in flight for a given id, and everyone else waits for it or serves the previous value.

**Per-process singleflight.** Inside one app, concurrent misses for the same id share one primary lookup and one body read. The others wait on that result, with a timeout. Four processes mean up to four fills, not 8,700. At this ceiling, four point reads are nothing. This is the control you take first, because it does not add a cluster lock.

**Jittered TTL.** Set the entry TTL uniformly between **45 and 75 seconds**, not a flat 60. Keys, and rebuilds of the same hot key, do not expire on the minute. Jitter does not collapse a single key's thundering herd by itself; singleflight does. Jitter stops a whole slab of keys from expiring together after a restart that refilled them in a tight loop.

**Stale-while-revalidate, as a sentence.** If you still have the previous entry, serve it for a bounded extra window (the jitter band is enough) while one singleflight refreshes. A hot key then never has a zero. The refresh still checks the primary. You do not serve stale past a tombstone. You do not serve stale past `expires_at`. The extra staleness is "the row might have been deleted in the last few hundred milliseconds and the tombstone has not landed," which you already admitted, not "the paste lives for another minute."

What you do not add:

- A distributed lock in the cache for every id, as the first design. Across four processes, singleflight already cut the miss fan-out from thousands to four. A lock service is a new dependency on the read path of your hottest key. Take it only if they show you dozens of app processes and a primary that still falls over at that fan-out. Then the lock is "one filler cluster-wide," with a short lease, and waiters that give up and 503 rather than all hitting the primary. Day 19 is the give-up.
- A counter of hits in the primary. That write path is hotter than the read you were protecting.
- Pre-warming every live paste. 450 million entries will not fit, and most are never read.

### The body of the hot key

8,700 reads/s times a 10 KB mean is about **87 MB/s** off the NVMe for that one file, or about **8.7 GB/s** if the viral paste is the 1 MB cap and you were wrong to use the mean. One file in the page cache can do the mean case. The cap case is an origin-bandwidth break. Do not "fix" it by putting the 1 MB value into the metadata cache under the same stampede logic and calling it done. Note it. The CDN day is where public bytes leave the origin. Today, singleflight also collapses the **body** read: one process reads the file once per miss wave and can hold the bytes for the waiters. You are allowed to keep that one body in process memory for the singleflight. You are not allowed to pin every hot body forever on every app. The process is stateless again a moment later.

### Delete of the hot key

Tombstone, as on day 10. Singleflight waiters who are in flight with a pre-delete primary read must not overwrite the tombstone when they finish. The filler checks the tombstone before `SET`. Waiters who already received bytes may still be streaming. You do not recall a TCP stream. The user-visible leftover is in-flight responses, not a fresh hour of cache.

## Diagrams

### One filler, many waiters

```mermaid
sequenceDiagram
  participant R1 as Readers
  participant A as One app
  participant K as Cache
  participant P as Primary
  R1->>A: GET hot id
  A->>K: get
  K-->>A: miss, entry just expired
  Note over A: singleflight, one lookup
  A->>P: point read
  P-->>A: live row
  A->>K: set jittered TTL if no tombstone
  A-->>R1: all waiters get the one result
```

The other three app processes may do this once each. Draw a second arrow only if you are answering "is singleflight global?" The answer is no.

### Two clocks that are not the same

```mermaid
flowchart LR
  ttl[Cache TTL 45 to 75s]
  inv[Tombstone on delete]
  exp[expires_at on the row]
  ttl -->|bounds a mistake| mem[Memory and stale window]
  inv -->|pulls the paste| origin[Origin stops serving]
  exp -->|product deadline| origin
```

If a design merges these three into one timer, it will either serve deleted pastes or forget them too slowly to bound RAM. Keep the arrows apart.

## Trade-offs

**Choice.** Per-process singleflight, jittered TTL, tombstone invalidation, stale-while-revalidate only inside the jitter band and never past `expires_at` or a tombstone.

**Alternative.** A flat 5 second TTL "so deletes feel instant," or a cluster-wide lock on every miss, or no cache on hot keys ("just hit the primary, one row is cheap").

**What you give up.** Singleflight adds wait time on a miss: the waiters take the latency of one primary read, and a slow filler slows them all. You need a timeout so a stuck filler does not pin 8,700 requests. Those timed-out waiters get a 503 or a single extra try, not a private trip to the primary. You also give up perfect global collapse. Three fills can still happen at once.

**Why not the alternatives.** A 5 second TTL makes the stampede twelve times more often and does not replace a tombstone. A cluster lock on every key puts your hottest read behind a lock service for a problem three fills already solved. "Just hit the primary" is fine for one row at 8,700 **if** the only problem is one row. It is not fine when the cache restarts and the whole working set misses. The stampede control is for the cliff, not for the steady state of one popular file.

**10× break.** Half of ~174,000 is about **87,000 reads/s on one key.** Four singleflights still protect the primary. The NVMe and the one app process streaming 87,000 responses do not. At a 10 KB body that is ~870 MB/s, near a full 10 Gbit NIC, from one key, and the process ceiling was 8,000 reads/s. The hot key breaks the **app and the NIC** at 10× long before it breaks a primary you protected. The next fix is to stop serving those bytes from the app (day 15), not a smarter lock. Say that so you do not spend the hour deepening singleflight while the bytes set the building on fire.

## Talking points

**Say.** "Half of peak on one id is about 8,700 reads a second. Steady state is a cache hit. The cliff is expiry or a cold cache. Per process I singleflight the miss, so four apps mean about four primary reads, not thousands. TTL is jittered between 45 and 75 seconds so I don't line the cliffs up. Delete is a tombstone. I will not shorten the TTL and call it invalidation."

**Say.** "One row in Postgres can take this. The dangerous case is many keys missing at once. Singleflight plus jitter is aimed at that. I am not adding a trending table."

**Hand-waving.** "Hot keys go in a special cache." It is the same cache. The specialness is the collapse of in-flight misses, which every key should get. You do not need a second product for famous pastes.

**Hand-waving.** "We'll just use a mutex." Where does it live, what is the lease, and what do the waiters do when it is slow? If you cannot answer, you have a pile-up with a new name.

**If they ask about the dogpile on the body.** Singleflight returns the same bytes to the waiters. You do not re-read the file per waiter inside one process. You still do not pin the body in the metadata cache as the general policy.

**If they ask what you page on.** Primary read QPS spiking while the cache is up, which means singleflight is not collapsing or the cache is not being used. Cache evictions aligned in time. Not the existence of a popular paste. Popular is the product.

## Kit artifact

One checklist row.

| Row | Lock |
|---|---|
| Hot key | Stampede control is singleflight, not a shorter TTL. TTL bounds memory and mistakes. Invalidation is the tombstone. Paste expiry is `expires_at`. |

## Design log

One line: how many primary reads your attempt produced when the hot entry expired. If you could not bound it, that number is the gap.

Next: [Day 12 — SQL or NoSQL from the broken query](12-sql-or-nosql-from-the-broken-query.md).

---

<!-- day-nav -->
[← Day 10 — Cache-aside for the melted read](10-cache-aside-for-the-melted-read.md) · [Day 12 — SQL or NoSQL from the broken query →](12-sql-or-nosql-from-the-broken-query.md)

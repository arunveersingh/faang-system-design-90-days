<!-- day-nav -->
[← Day 11 — Hot-key stampede](11-hot-key-stampede.md) · [Day 13 — Replication and the lagging read →](13-replication-and-the-lagging-read.md)

# Day 12 — SQL or NoSQL from the broken query

**Do now**

1. Set a timer.
2. Attempt the problem. Stop at the attempt line. Do not scroll.
3. Then read.


## Time box

35 minutes.

| Minutes | Do this |
|---|---|
| 0–10 | Attempt. List the queries you actually run. Mark the one a key-value store cannot answer. |
| 10–28 | Read. If you switched databases without a broken query, switch back. |
| 28–35 | Say the choice, what you gave up, and the query that would force the other store. |

## Intent

Facing a query the current store cannot serve, leave able to choose SQL or NoSQL from the access pattern and name what you give up. The store is a fix for a query shape. "NoSQL scales" is not a query.

## Problem

> Design a pastebin. People paste text and share a link.

The interviewer says: "Why is this still Postgres? Wouldn't a key-value store be a better fit?"

## Attempt before reading

10 minutes. Do not scroll. Metadata is a row in Postgres. Bodies are files on one NVMe. The cache holds a copy of the read decision, not the source of truth. About 350 writes/s peak, about 450 million live rows, one secondary index on `expires_at`.

Write:

1. Every query the system really runs. If you cannot point at a caller, it is not a query.
2. Which of those a primary-key store answers, and which it does not.
3. Your choice, SQL or a key-value store, and the concrete thing you give up.
4. The query someone in the room will propose that you refuse because it is a different product (a directory, a search).

Do not name a vendor as the answer. Name the access pattern first.

---

**Stop. The choice below stays SQL, for a reason you can point at.**

---

## Diagrams

### The only queries

```mermaid
flowchart LR
  get[Get by id]
  ins[Insert by id]
  range[Range on expires_at]
  get --> sql[Postgres primary key]
  ins --> sql
  range --> idx[Secondary index]
  range -.->|full scan if you drop the index| bad[450 million rows]
```

### What a pure key-value store breaks

```mermaid
flowchart TB
  kv[Key-value id to row]
  kv -->|serves| point[Create read delete]
  kv -->|does not serve| sweep[Sweeper batch]
  sweep --> choice{Your move}
  choice -->|keep SQL| stay[Index you already have]
  choice -->|leave SQL| bucket[Second structure and a dual write]
```

The right branch is real work. Take it when the left branch's primary cannot take the write rate. Not when the logo feels dated.

## Requirements

The queries the contract actually demands:

| Caller | Query | Rate, order of magnitude |
|---|---|---|
| Read and delete | Point get by `id` | Peak reads were 17,400 before the cache. After cache-aside, the primary sees misses, not the full peak. Writes and deletes ~350/s peak, ~116/s average. |
| Create | Insert by `id`, unique | Same write rate. |
| Sweeper | The ids whose `expires_at` is before now, in batches | About 116 deletes/s once retention is full. A batch is a range, not a point. |

There is still no "list my pastes," no search by syntax, no "most viewed." Those would be new queries. You do not get to invent them to justify a database, and you do not get to invent them to justify leaving one.

Bodies are not in this decision. A 10 KB to 1 MB value was already refused as a column, on day 5, because of WAL, vacuum, and backup. That refusal is about **value size**, not about SQL versus NoSQL. Putting the body in a document store does not make 4.5 TB of dead bytes a good idea. The body stays a file until day 14 moves it for the inode reason.

## Design

**Stay on Postgres for the row.** The access pattern is a point key plus one time range for garbage collection. You already sized that index inside the metadata budget, on the order of the 135 GB, not as a second cluster.

A key-value or wide-column store, used as "id → document," serves the point get and the insert. It does **not** serve "rows expiring now" unless you add a second structure. That second structure is the broken query, made visible:

```text
SELECT id FROM pastes
WHERE expires_at < now()
ORDER BY expires_at
LIMIT 1000;
```

If the store cannot answer that without a full scan of 450 million rows, the sweeper either scans the world or you redesign expiry. Scanning 450 million rows to delete 116 a second is the break. You do not "fix" a scan by calling the product scalable.

### What you would have to build to leave SQL

If they insist on a key-value primary for the row, expiry becomes a **bucket you write at create time**, not a predicate you discover later. Day 39's outbox is the grown-up version of that. You do not need it at 116 deletes a second against an index you already have.

So the "broken query" in this interview is not a query Postgres cannot run. It is the query a fashionable replacement cannot run.

### What would actually force a move

Be ready for the push. You leave this primary when one of these is the break, not before:

- **Write rate.** Thousands of commits a second that one primary cannot group-commit. You are at ~350 peak. Not this break. At 10×, ~3,500 peak commits, you will look again (day 24). The next step then is a partition key, which can still be SQL. Partitioning is not a synonym for NoSQL.
- **A query you agreed to serve** that is not a key and not a time range. Full-text search of bodies would be an index product. You refused search in week 1. Do not accept it now as a reason to change databases.
- **Value size.** Already handled by keeping bytes out of the row. Object storage later is not a NoSQL metadata decision.

A document store that still has a secondary index is SQL with a different license. Do not spend the hour on that distinction. Spend it on the range query.

### What you give up by staying

You give up horizontal write scaling as a default. Your schema is eight fields and has been stable since day 4; flexibility you will not use is not a feature. At 350/s that cost is in the noise next to body durability.

You do not run a second metadata database "for the reads." The cache is the read scaling tool. Two metadata stores means two sources of truth and the tombstone problem doubled.


## Trade-offs

**Choice.** Postgres for metadata. Point key plus an `expires_at` index.

**What you give up by refusing the alternative.** The ability to say "we can add nodes" as a write-scaling story. You also give up a slightly simpler mental model on the read path (one key, one value, no planner). You keep a sweeper that is a query, not an application-maintained bucket, and you keep a single commit for "row exists and will be found by expiry."

**10× break.** ~3,500 commits/s peak and ~4.5 billion live rows. The index still answers the same query; the primary may not keep up with the commits.

## Say this in the room

The user-facing calls are point lookups by id. The sweeper is a range on `expires_at`, about a hundred deletes a second once we're full. Postgres does both. A key-value mapping does the point and full-scans 450 million rows to expire, or I maintain a second bucket and a dual write. I'm not taking that trade at 350 writes a second.

## Kit artifact

One trade-off card row.

| Choice | You take a key-value primary when | You keep SQL when |
|---|---|---|
| Metadata store | The write rate has broken one primary, and you have a plan for the expiry range that is not a full scan | The queries are a point key plus a range the index already serves |

## Design log

One line: the query your attempt could not answer after the store you picked. If you had no such query and still switched, the gap is the switch.

Next: [Day 13 — Replication and the lagging read](13-replication-and-the-lagging-read.md).

---

<!-- day-nav -->
[← Day 11 — Hot-key stampede](11-hot-key-stampede.md) · [Day 13 — Replication and the lagging read →](13-replication-and-the-lagging-read.md)

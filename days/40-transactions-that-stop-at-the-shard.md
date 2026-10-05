<!-- day-nav -->
[← Day 39 — Outbox, not dual write](39-outbox-not-dual-write.md) · [Day 41 — Sagas and compensation →](41-sagas-and-compensation.md)

# Day 40 — Transactions that stop at the shard

**Do now**

1. Set a timer.
2. Attempt the problem. Stop at the attempt line. Do not scroll.
3. Then read.

## Time box

40 minutes.

| Minutes | Do this |
|---|---|
| 0–12 | Attempt. Draw the rows that commit together. Name the isolation level. Name one change that would cross a shard. |
| 12–32 | Read. If your transaction spans two slot maps, you have a distributed transaction you did not price. |
| 32–40 | Say the isolation level, what anomaly you refuse, and the boundary the transaction does not cross. |

## Intent

Facing a multi-row change, leave able to draw the transaction at one shard and pick an isolation level on purpose. A transaction that "covers everything" is a wish. A transaction that covers the rows that must not diverge is a design.

## Problem

> Design a pastebin. People paste text and share a link.

The interviewer says: "You have the paste row, the idempotency row, and the outbox row. What is in one transaction, what isolation do you run, and what happens if I ask you to move a paste between shards?"

## Attempt before reading

12 minutes. Do not scroll. Day 37 colocated the idempotency key with the paste. Day 39 put the outbox on the same primary. Day 31's slots are the partition. Isolation levels you may name: read committed, repeatable read, serializable. Pick one. Do not invent a fourth.

Write:

1. The create transaction's rows. The delete transaction's rows. What is deliberately outside both (the object PUT, the purge).
2. The isolation level, and one anomaly it stops that matters for this product.
3. What a "move this paste to another shard" request would require. Two-phase commit, a saga, or a refusal.
4. Whether two concurrent creates with different keys can serialize without blocking each other. If your level makes them wait, say so.

---

**Stop. One shard, one level, a hard edge, below.**

---

## Requirements

A transaction ends at the shard boundary. That is not a Postgres preference. It is the day-31 decision: one slot's rows live on one primary, and that primary's log is the order. Crossing two primaries means a different protocol (two-phase commit, or a saga on day 41). You do not smuggle a cross-shard transaction into "BEGIN" by hoping both hosts are up.

What must be atomic for create:

- the paste row exists iff the idempotency key exists for that request,
- the 201 body you will replay is the one that describes that paste.

What must be atomic for delete:

- the paste is gone iff the outbox row that will purge it exists.

What is allowed to be outside:

- the object bytes (PUT before commit, orphan on crash),
- the origin cache tombstone (day 33),
- the broker message (day 39's poller).

If those outside pieces were inside the database transaction, you would be waiting on the bucket inside a held lock. You already refused that. The transaction is the metadata claim. The bytes are a precondition you enforce with order and a reaper, not with a distributed commit.

## Design

### Isolation, chosen

**Read committed** is the default on many systems, and it allows a second transaction to insert a row you already checked for. Check-then-insert of an idempotency key under read committed is a race: two creates with the same key both see absence, both insert, one unique-constraint conflict. The unique constraint is what saves you, not the isolation level. You still want the unique constraint. You do not want to rely on "I SELECTed and it was empty."

**Repeatable read** (Postgres's, which is snapshot isolation) gives each transaction a stable view of committed data as of its start. Two concurrent creates of **different** keys do not block each other. A concurrent delete of a paste you are not touching does not rewrite your snapshot mid-flight. A write-write conflict on the **same** row aborts one of you. That is the level you run: **repeatable read**, and unique constraints on `paste.id` and on `idempotency.key`.

What anomaly you care about stopping: a delete that commits while a create's transaction is open does not matter for different ids. For the **same** id you never create twice, because ids are unique and never reused. The anomaly that would hurt is two retries with the same key both inserting. The unique key turns that into one commit and one conflict. On conflict of the idempotency key, you abort, re-read, and return the stored 201. You do not invent a second paste. That path is the same under read committed **if** the unique constraint is present. Repeatable read buys you a stable read of related rows if you ever SELECT more than one, and it is the level you can say without pretending you need serializable.

**Serializable** would refuse concurrent transactions that could produce a cycle. At 350 creates/s you do not want a serializable abort storm because two unrelated pastes touched an index page the planner decided to fight over. You do not need it. The product has no "total of all pastes" row that two transactions increment. If they invent that counter, day 32's salt applies, and serializable still would not fix a hot key. Refuse serializable until a constraint requires it. Say which constraint that would be: a check that spans rows without a unique index to enforce it. You do not have one.

### What is in the BEGIN / COMMIT

Create:

```
BEGIN
  SELECT idempotency WHERE key = K
  if found and hash matches: return stored (no write)
  if found and hash differs: 409
  INSERT paste
  INSERT idempotency (key, hash, response, expires)
COMMIT
```

The PUT was before BEGIN, or between a short "reserve" and BEGIN, as day 37 said. Holding BEGIN across the PUT is how you pin the primary for a 1-second object write. Do not. Open the transaction for the metadata, not for the network.

Delete:

```
BEGIN
  SELECT paste WHERE id = I FOR UPDATE
  if missing or token mismatch: 404 path
  DELETE paste
  INSERT outbox
COMMIT
```

`FOR UPDATE` takes the row lock so a concurrent delete of the same id does not both think they won. The second gets no row and 404s. That is correct. You do not need serializable for two deletes of one id. You need the row lock.

### The edge of the world

"Move paste P from shard A to shard B" is a cross-shard transaction if you try to do it in one BEGIN. Options:

1. **Refuse.** Ids encode the slot. Moving would change the id or break GET routing. You never reuse ids, so you cannot renumber. Refusal is the product answer.
2. **Saga.** Copy to B, flip a pointer, delete from A, with compensations if a step fails. Day 41. You do not need it for this pastebin.
3. **Two-phase commit.** A coordinator prepares on A and B, then commits. You take a coordinator, a blocking prepare, and a recovery story. At this QPS and this product, it is a paper you cite to refuse, not a box you draw. Spanner-shaped external consistency is the appendix. You are not inventing TrueTime to move a paste.

Any SELECT that joins two shards is also outside a single transaction. The sweeper fans out and deletes with a per-shard predicate. It does not open a transaction that holds locks on four primaries. That would be a distributed deadlock with a sweeper as the victim or the villain.

### Long transactions

A transaction that stays open while the app waits on the CDN purge is a lock you gave to a network. Keep BEGIN short. The outbox exists so the long work is not in the transaction. If your attempt has "BEGIN, delete row, purge CDN, COMMIT," open it again. The purge is the worker. The transaction ends before 204.

## Diagrams

### The boundary

```mermaid
flowchart LR
  subgraph shard [One shard, one transaction]
    paste[Paste row]
    idem[Idempotency row]
    outbox[Outbox row]
  end
  put[Object PUT] -.->|before commit| shard
  purge[CDN purge] -.->|after, via outbox| shard
  other[Other shard] ===|no BEGIN across| ban[Not in this transaction]
```

Solid is atomic. Dotted is ordered but not atomic with the rows. The double bar is the refusal.

## Trade-offs

**Choice.** Repeatable read. Unique constraints. Create and delete transactions stop at the shard. PUT and purge stay outside. Cross-shard moves refused by the id encoding.

**Alternative.** Serializable everywhere, or a distributed transaction for anything that looks related.

**What you give up.** The ability to move a paste without a saga. The ability to hold a lock across a network call. You keep create and delete free of dual-write holes inside the shard, and you keep unrelated creates from serializing behind each other.

**10×.** 3,500 creates/s across four shards is still ~875/s per shard. Repeatable read holds. Serializable abort rates get worse with concurrency. That is another reason not to "upgrade" isolation because traffic grew. The key and the shard count grow. The isolation level does not.

## Talking points

**Hand-waving.** "We're ACID." Which A, which rows, which isolation? ACID without a boundary is a brand.

**Hand-waving.** "We'll use a distributed transaction." Name the coordinator, the prepare timeout, and what the user sees when one branch prepared and the other did not. If you cannot, you wanted a saga or a refusal.

**If they ask about the object store in the transaction.** You cannot. The object store does not join your BEGIN. The order PUT-then-commit, and the reaper, is the substitute for atomicity across that boundary. Say the orphan failure. Do not call it ACID.

## Say this in the room

A create commits the paste row and the idempotency row in one transaction on one shard, under repeatable read, with a unique key so two retries cannot both insert. A delete commits the removal and the outbox row the same way, with a row lock so two deletes of one id do not both succeed. The object PUT and the CDN purge stay outside that transaction on purpose. I will not BEGIN across two shards, and the id encodes the slot, so I refuse a move rather than invent two-phase commit. If something must cross shards later, that is a saga, not a longer BEGIN.

## Kit artifact

One checklist row.

| Follow-up | You answer with |
|---|---|
| "Is this transactional?" | The rows inside one shard, the isolation level, and the boundary you will not cross. |

## Design log

One line: the row you had left outside the transaction that can diverge, or the isolation level you named without an anomaly.

Next: [Day 41 — Sagas and compensation](41-sagas-and-compensation.md).

---

<!-- day-nav -->
[← Day 39 — Outbox, not dual write](39-outbox-not-dual-write.md) · [Day 41 — Sagas and compensation →](41-sagas-and-compensation.md)

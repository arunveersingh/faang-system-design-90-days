<!-- day-nav -->
[← Day 13 — Replication and the lagging read](13-replication-and-the-lagging-read.md) · [Day 15 — CDN in front of public bytes →](15-cdn-in-front-of-public-bytes.md)

# Day 14 — Object storage for the body

## Time box

40 minutes.

| Minutes | Do this |
|---|---|
| 0–12 | Attempt. Why the NVMe layout breaks, the new write order, and the orphan. |
| 12–32 | Read. If you delete the object before the row, walk it back. |
| 32–40 | Say the orphan case and the live-row-missing-object case as two different bugs. |

## Intent

Facing large bodies in the database — or, in this pastebin, hundreds of millions of small files that are just as stuck — leave able to move bytes to object storage and name the orphan-object failure. Object storage is the durability split day 5 deferred. It is not a new product, and it is not where the row moves.

## Problem

> Design a pastebin. People paste text and share a link.

The interviewer says: "Stop describing a directory of 450 million files. Put the bytes somewhere you can restore. Tell me what a crash in the middle leaves behind."

## Attempt before reading

12 minutes. Do not scroll. You still have `/data/{id[0:3]}/{id}` on one NVMe. Postgres, with one async replica, holds the row. Cache-aside holds metadata only. About 4.5 TB resident, about 450 million live objects, mean body 10 KB, cap 1 MB. 201 only after bytes are durable and the row commits.

Write:

1. The operation that fails on the directory even though GET latency is fine (backup, restore, or a dead disk).
2. The write order with an object store: what is durable before the insert, and when you 201.
3. The orphan: a byte object with no row. How it can happen. How you reap it without deleting a live paste.
4. What you refuse to store in the object: the delete token, the index, a public bucket with no expiry check.

---

**Stop. Bytes move. The row stays in Postgres.**

---

## Requirements

Same contract. The body is opaque UTF-8, at most 1,000,000 bytes. The path is not part of the API. Clients never learn a bucket name.

Durability you can now say: an acknowledged paste's bytes have a store that is not one NVMe you administer, and the row has the replica from day 13. You still do not say the paste survives every region. One region. The object store's internal replication is a dependency with a durability claim you are buying, not a protocol you are designing. One sentence: you treat a successful PUT as the durability ack the fsync used to be. If they ask "what if the vendor loses an object," the user sees `body_missing` and a page, same as a missing file. You do not invent a second copy in another cloud unless they make that the deep dive.

The mean object is 10 KB. Object storage is often described for large blobs. You are using it anyway, because **the inode and restore problem is the break**, not the size of one paste. Say that so you do not get talked into packing volumes as the only grown-up answer. Packing is the alternative below. Both beat one flat directory.

## Design

Object key: `pastes/{id}`. The id is 12-character base62 and never reused, so the key is never reused. You do not store the key in the row. There is nothing to diverge. If a future packing scheme needs an offset, the row grows then. Not today.

The bucket is private. The app holds credentials. A reader on the internet cannot GET the bucket. If they could, expiry and delete would be optional, which breaks the contract. The app streams the body after the metadata check, or it issues a very short signed URL. Prefer streaming from the app at today's 174 MB/s peak, which one origin can do, so the expiry check and the byte read stay in one place. A signed URL that lives for an hour is a capability you cannot revoke on delete. If you sign, the signature lifetime is short enough to say out loud, on the order of a minute, and you still have a hole until it dies. Day 15 puts a CDN in front of public bytes; do not build that CDN today. Today the app is the only reader of the bucket.

Max body 1 MB. One PUT, not a multipart upload. Multipart is how you resume a 5 GB video. You do not have one. Refusing multipart keeps the crash window to a single request.

### Write order

1. Validate size, UTF-8, and the TTL allow-list. Reject before any PUT. A 413 writes nothing.
2. Mint the id. Generate the delete token. Hash it.
3. PUT `pastes/{id}` with the raw bytes. This ack replaces the group-commit fsync. Many PUTs are in flight; you are no longer under the 200/s serial fsync ceiling. Peak 350 PUTs of 10 KB is about 3.5 MB/s. That is not a storage-bandwidth problem.
4. Insert the row on the primary. On id conflict, mint again and PUT the new key. The old object is an orphan.
5. Commit. Then 201 with the plaintext token.
6. Do not warm the cache.

If the PUT fails, you do not insert, and you do not 201. No orphan from a failed PUT. If you time out and the PUT actually landed, you do not know; a retry of the **client request** mints a new id (create is still not idempotent) and the timed-out object may be an orphan. Same class of mess, reaped the same way.

If the PUT succeeds and the insert fails, the client gets an error, not a link. The object exists and nothing points at it. **That is the orphan.** It is not readable, because every read starts at the row, then the cache. An orphan wastes money. It does not leak text, as long as the bucket stays private.

### The reaper

A loop, same status as the sweeper: not a queue, not a second platform. The queue day will move work that should not live in the request process. This loop can be a single worker beside the sweeper until then. Do not block 201 on reaping.

Rules so the reaper does not eat a live paste:

- It only deletes an object whose id has **no row**, and whose object is older than a margin, **one hour**. The margin is larger than any in-flight create you are willing to allow, including a stuck request and clock skew between the bucket and the primary. A younger object might be a PUT whose insert has not committed. Leave it.
- It never deletes by prefix listing as the way you expire pastes. Expiry is still `expires_at` on the read, and the sweeper deletes the **row first**, then deletes the object.
- User delete: commit the row removal and the cache tombstone, return 204, then delete the object. Opposite order is the outage: a live row whose object is already gone is a 500 for every reader. An object that outlives the row is an orphan, which is the recoverable direction.
- If the object delete fails, the row is already gone, so readers 404. Retry the object delete. A leftover object is waste, not a read.

How the reaper finds orphans without listing 450 million keys on a timer: keep a small side list of ids whose PUT succeeded and whose insert did not, written only in the failure path, and also accept that a crash between PUT and that side list is why the age rule exists. You can periodically sample. You cannot `LIST` the whole bucket as your correctness path; at 450 million keys, listing is the same operational cliff as the directory walk you just escaped. Say that. The hour-age rule plus a failure log is the design. A full bucket inventory is a repair tool, not the loop.

### Read order

Unchanged in shape. Cache, then primary on miss, check `expires_at`, then GET the object by id.

- No row or expired: 404. Do not GET the object.
- Live row, object missing: **500** and `body_missing`. Do not 404. Do not let the reaper be a suspect if you followed the age rule; this is loss or a bug that deleted early.
- Live row, object present: stream `text/plain` with `nosniff`.

The data host loses the NVMe. It is now Postgres only, plus the replica on another host. The app tier is stateless in a way day 8 wanted and could not quite have: any app can PUT and GET. There is no exported directory.

### What stays out of the object

The row, the token hash, the secondary index, the cache. Analytics. A public HTML rendering. The object is the bytes and nothing else. Content-type on the object, if you set one, is `text/plain`. You still override on the way out and set `nosniff`, so a stored header cannot turn the paste into a page.

## Diagrams

### Write order and the orphan

```mermaid
sequenceDiagram
  participant C as Client
  participant A as App
  participant O as Object storage
  participant P as Primary
  C->>A: POST body
  A->>O: PUT pastes/id
  O-->>A: durable ack
  A->>P: insert row
  alt insert commits
    A-->>C: 201
  else insert fails
    A-->>C: error, no link
    Note over O: orphan object, not readable
  end
```

### Two failures that are not the same

```mermaid
flowchart LR
  orphan[Object and no row]
  missing[Row and no object]
  orphan -->|reap after one hour| waste[Wasted bytes]
  missing -->|do not hide| err[500 and page]
```

If a design treats both as 404, it will reap the wrong one and hide the other.

## Trade-offs

**Choice.** Private bucket, key equals id, PUT then insert, 201 after both, reaper with a one-hour age floor, row deleted before the object on the way out.

**Alternative.** Append-only volume files on disk you still operate. The row gains an offset and a length. Backup is large files, which was the point. Compaction is the new job: reclaiming expired pastes inside a volume without disturbing live offsets.

**What you give up by choosing the bucket.** A single-machine crash story you could draw with one disk. PUT latency is now on the create path, a network hop with a planning round trip you should state (tens of milliseconds, not the 5 ms local fsync). You take a dependency on a store you do not fsync yourself. You take orphans as a normal failure, not as a bug you are surprised by. You take the bill for 4.5 TB plus PUT/GET requests, which at this size is not the thing that changes the design, and you say so rather than pretending it is free.

**Why not the volumes, as the first move.** Volumes fix restore and keep you on a disk you run. They add compaction, a crash window inside a file, and a row format change. The bucket is the smaller interview design at 10 KB mean and 1 MB cap, and it is the one that lets every app process see every byte without a shared filesystem. Take volumes if they forbid a managed object store or if the mean size is so small that per-object overhead dominates and you have measured it. You have not measured it. Do not invent a packing layer to sound careful.

**10× break.** Ingest becomes ~1 TB/day and ~45 TB resident, ~4.5 billion objects. PUT bandwidth is still only tens of MB/s. The break is **listing, inventory, and a reaper that scans**. Key-directed reaping from a failure log still works. A design that lists the bucket to expire will not finish. The metadata primary's commit rate breaks earlier than the bucket's byte rate (day 24). Do not "shard the object store" in the abstract. You are not operating its internals.

## Talking points

**Say.** "The break is restore of 450 million files, and a dead NVMe taking the only copy. Key is the id, bucket is private, PUT then commit the row, 201 only after both. A failed insert leaves an orphan I reap only if it is older than an hour and has no row. A live row with no object is a 500, not a 404, and not a reap."

**Say.** "I am not doing this because 10 KB is a large object. I am doing it because the directory is the outage. The serial fsync ceiling goes away because the PUT is the ack and many are in flight. Peak body bandwidth is still about 3.5 MB/s."

**Hand-waving.** "We'll just use S3." That is this design only if the order, the private bucket, and the orphan are in the next two sentences. The product name is not the order.

**Hand-waving.** "Eventual consistency, so the read might not see the PUT." You waited for the PUT ack before the insert, and you do not hand out the id before the commit. A store that can ack a PUT and then hide the object from a GET is a `body_missing` bug you page on. You do not paper over it by retrying until it appears and calling that the architecture.

**Hand-waving.** "Public bucket, the URL is the object URL." Then you cannot expire or delete, and you cannot stop serving the bytes as whatever content-type the bucket guessed. The paste URL stays on your origin.

**If they ask about encryption.** TLS in transit. Server-side encryption on the bucket as a default, not a box you draw. The delete token is still a hash in the row, not an object. End-to-end encryption remains a non-goal.

**If they ask what you page on.** PUT error rate. Reaper deleting zero objects while the orphan log grows. `body_missing` above zero. Bucket size versus the 4.5 TB mix, and the 365-day share, same alarm as the disk headroom on day 5. Not 404s.

## Kit artifact

One checklist row.

| Row | Lock |
|---|---|
| Orphan object | PUT succeeded, row did not. Unreadable. Reap only after an age floor and a missing row. Never the reverse order on delete. |

## Design log

One line: the crash window your attempt left, and whether the leftover was an orphan or a 500. If you could not tell those apart, that confusion is the gap.

Next: [Day 15 — CDN in front of public bytes](15-cdn-in-front-of-public-bytes.md).

---

<!-- day-nav -->
[← Day 13 — Replication and the lagging read](13-replication-and-the-lagging-read.md) · [Day 15 — CDN in front of public bytes →](15-cdn-in-front-of-public-bytes.md)

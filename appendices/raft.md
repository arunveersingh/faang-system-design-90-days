<!-- appendix-nav -->
[← Spanner](spanner.md) · [Classic papers](classic-papers.md)

# Raft (appendix)

**Not a day. Not a mock.** Under 40 minutes. Day 36 already owns the interview move: a leader is a dependency with an unavailable window. This note makes **leader**, **log**, and **election** concrete so you do not derive votes and terms on the whiteboard.

## Time box

| Minutes | Do this |
|---|---|
| 0–8 | Three words only: leader, log, election window. |
| 8–28 | What commits, what the old leader must not do, what the client sees. |
| 28–35 | Delete any protocol steps from your practice page. Keep the window and the 503. |
| 35–40 | One sentence for the room. |

## Intent

Leave able to treat consensus as a **bought dependency**: one writer per shard, a replicated log, and a bounded time with no writer. Leave unable to spend the hour on vote messaging, term numbers, or log matching proofs.

## What the paper is (one breath)

Raft is a consensus algorithm for managing a replicated log. The point of the paper, for interview altitude, is clarity: there is a **leader**, followers replicate its **log**, and when the leader dies an **election** chooses a new one. Safety rules keep two leaders from committing different entries in the same slot.

You buy this from etcd, ZooKeeper (different protocol, same dependency shape), your database's quorum, or a managed consensus service. You do not implement Raft in the design hour.

## Ideas you can spend

### Leader

One node accepts client writes for the shard and appends them to the log. Followers accept appends from the leader. Reads that must see the latest commit go to the leader (or to a follower only after you accept lag — day 13 / 34).

**Say this:**

> "One writer per partition. While it is alive, that partition has an order. I am not drawing the election."

### Log

Committed entries are durable on a majority and will survive leader change. A new leader's log is at least as complete as any prior commit — that is the safety property you depend on without proving it.

**User-visible meaning:** a **201** you returned after commit will still be there after failover. A write that only lived in the old leader's memory, never committed, is gone — so you do not tell the client success before commit.

### Election window

When the leader is unreachable, followers hold an election. During that window **there is no leader**. Writes that need the log **fail or wait**. Day 36's planning number is about **30 seconds** notice-plus-elect-plus-repoint — say your number; do not say "a few."

**User-visible failure (pastebin altitude):**

- POST → **503** (not 201 queued on the app).
- Authoritative GET on miss → **503** (not 404 — you do not know if the row exists).
- Cache / edge hits may still serve inside the stale window you already accepted.
- At peak, a 30-second write hole on one shard is on the order of **peak_create_rate × 30** failed creates — use the peak you already stated in the hour (day 36 used ~10,500 at one planning peak).

**Old leader must not keep committing.** If it is partitioned and still thinks it is leader, it must not accept writes that a majority will never see. That is split-brain. The database's fencing / term / lease story is the dependency. Your app refuses to invent a second primary.

**Say this:**

> "If the old primary is partitioned, it must not keep taking creates. I rely on the store's fencing. I will not accept a write path that bypasses it."

### What you will not draw

Cross these out if they appear on your page:

- Vote request / vote response sequences
- Term numbers and who voted for whom
- Log matching property proofs
- Millisecond election timeouts and randomized backoff diagrams
- "I'll implement Raft for metadata"

Replace with: leader, log, window duration, client status codes, fencing.

## What you refuse in the room

| Ask | Refusal |
|---|---|
| "How does the election work?" | "Dependency with a timeout. I will not implement it. Paper talk is after this interview if you want it." |
| "Can we write to a follower while electing?" | "Not for commits that need the order. 503 until a leader exists." |
| "Queue creates on the app during the window" | Refuse. A 201 needs a commit. An app queue is a second log. |
| "Two writers for availability" | Refuse without a conflict story (day 42) or a real multi-primary product cost. |

## Tie-back to this course

| Day | Already owns | This note adds |
|---|---|---|
| 13 | Promote replica; lagging read | Election as the named window behind promotion |
| 35 | Quorum integers | Raft majority is how the log commits — still do not derive it |
| 36 | Leader as dependency; 503; fencing | Paper vocabulary without protocol theater |
| 80 | Losing a region | Election window is local; region loss is a different altitude |

## Say this in the room (full breath)

> "I treat consensus as a dependency: one leader per shard, a replicated log, and about thirty seconds with no leader if the primary dies — during which creates and cold authoritative reads return 503. Cache hits can continue inside the stale window we already named. The old leader must be fenced so it cannot commit after a new one exists. I am not implementing Raft in this interview."

## Close

Stop at 40. If you still have protocol steps on your practice page, delete them. The appendix worked only if the hour got shorter, not longer.

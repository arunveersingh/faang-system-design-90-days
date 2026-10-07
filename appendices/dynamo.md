<!-- appendix-nav -->
[← Classic papers](classic-papers.md) · [Spanner →](spanner.md)

# Dynamo (appendix)

**Not a day. Not a mock.** Under 40 minutes. Day 18 (consistent hashing) and Day 35 (quorums) already own the interview moves. This note is vocabulary and failure altitude — not a license to redesign the pastebin as leaderless storage.

## Time box

| Minutes | Do this |
|---|---|
| 0–8 | What Dynamo is for, and what you will not claim. |
| 8–28 | Hashing, N/R/W, sloppy quorum, hinted handoff, conflicts. |
| 28–35 | Say the user-visible failure. Refuse the paper as a consistency skip. |
| 35–40 | One sentence you would actually say in the room. |

## Intent

Leave able to use **consistent hashing**, **sloppy quorum**, **hinted handoff**, and **application-level conflict handling** when the interviewer forbids a leader or hands you a leaderless store. Leave unable to hide behind "Dynamo-style" without integers and a failure.

## What the paper is (one breath)

A highly available key-value store. Nodes come and go. Writes should often succeed even when some preferred replicas are down. Conflict resolution is pushed to the application. That last sentence is the trap: availability of the write is not the same as a single truth the next read can trust.

You are not building Dynamo. You are borrowing four ideas.

## Ideas you can spend

### Consistent hashing

Keys and nodes live on a ring. Each key maps to a preference list of N nodes. When a node joins or leaves, only nearby keys move — not the whole map.

**In the room (day 18 already):** say what fraction of keys remaps on add/remove, and whether you need virtual nodes for balance. A fixed slot map is simpler when the node set is small and changes are rare. Dynamo's ring is the answer when membership churn is normal.

**Failure:** a bad membership view means two clients write the same key to different preference lists. You get siblings. Membership gossip is a dependency; treat a split brain in membership like a partition, not like "eventual consistency will fix it."

### N, R, W — overlap, not magic

Same integers as day 35. **N** copies intended. **W** must ack a write. **R** must answer a read. If **W + R > N**, read and write sets overlap.

**Say this:**

> "N=3, W=2, R=2. The write waits for two acks. The read takes the higher version from two answers. One node down, both paths still work. Two nodes down, writes fail closed."

**Refuse:** "we use quorum" without the three integers. **Refuse:** calling W=2, R=2 "strongly consistent" without versions and without ruling out forks from concurrent writers.

### Sloppy quorum

If a preferred node in the list is unreachable, Dynamo may write to the next reachable node outside the preference list so the write can still get W acks. That is **sloppy**: the write set is no longer drawn from the same N the inequality assumed.

**User-visible failure:**

- Create returns **201** from a sloppy set.
- A later **strict** read of the preferred three misses the row → **404** after a success. That is a broken ack, not "eventual."
- A **delete** that lands only on a stand-in, while preferred copies still serve the body → **resurrection**. Refuse sloppy deletes.

**Say this:**

> "I refuse sloppy quorum for deletes and for any write I already called durable to the client. If I cannot explain the repair, I 503 instead of taking a stand-in ack."

### Hinted handoff

The stand-in keeps a **hint**: "this write belonged to node X." When X returns, the hint is forwarded. Until then, strict readers of X's slot can miss.

**Operability:** page on "hints outstanding older than T" and on "201 then miss," not only on raw 5xx. A quiet hint backlog is a durability lie with a delay.

**If you cannot operate hints:** do not take sloppy writes. Fail closed.

### Conflicts are your problem

Two writers, overlapping but not identical replica sets, both succeed → **siblings** (vector clocks in the paper; versions in your mouth). The application merges, or picks a winner, or surfaces a conflict to a human.

**Say this:**

> "Leaderless writes can fork. I need a version on every write and a merge rule, or I keep a single writer per key. The paper does not pick the merge rule for me."

Last-writer-wins with wall clocks is a choice you must name as lossy. Do not call it "Dynamo consistency."

## What you refuse in the room

| Claim | Refusal |
|---|---|
| "Dynamo-style, so we're good" | Name N, R, W, version rule, and sloppy-or-not. |
| "Availability means the read is fine" | Availability of the write ≠ read sees it. |
| "I'll summarize the paper" | Spend the minute on the user-visible miss instead. |
| "Sloppy quorum for everything" | Refuse for delete and for durable create acks. |

## Tie-back to this course

| Day | Already owns | This note adds |
|---|---|---|
| 18 | Hash ring, remap cost, when fixed slots win | Preference list language; membership as a dependency |
| 35 | N/R/W, overlap, stale vs unavailable | Sloppy break of the inequality; hinted repair |
| 42 | Conflict rules | Why leaderless stores force that day into the critical path |
| 46 | Tombstones | Why sloppy delete resurrects |

The pastebin in Phases 1–3 keeps a **primary**. You do not migrate it to Dynamo in an appendix sitting. You use these words when they forbid the leader or hand you an already-leaderless store.

## Say this in the room (full breath)

> "If we cannot have a leader: N=3, W=2, R=2, version on every write, highest version wins on read. I refuse sloppy quorum for deletes. If a preferred replica is down and you still want the write, that is a hinted handoff I have to repair — until repair, a strict read can miss. Concurrent updates can sibling; the app merges or I keep single-writer per key. I am not implementing Dynamo; I am picking integers and the failure I accept."

## Close

Stop at 40. Do not open Spanner or Raft in the same sitting unless the interview tomorrow explicitly needs both vocabularies — it almost never does.

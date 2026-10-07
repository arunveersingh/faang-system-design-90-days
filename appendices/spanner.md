<!-- appendix-nav -->
[← Dynamo](dynamo.md) · [Classic papers](classic-papers.md) · [Raft →](raft.md)

# Spanner (appendix)

**Not a day. Not a mock.** Under 40 minutes. Day 30 (linearizability) and Day 40 (transactions that stop at the shard) already own the interview moves. This note is why a **global** transaction is expensive — and when you refuse it.

## Time box

| Minutes | Do this |
|---|---|
| 0–8 | External consistency in one sentence; what TrueTime is not. |
| 8–28 | Global vs shard-local transaction; commit wait; cost and failure. |
| 28–35 | Say when you refuse a cross-region transaction. |
| 35–40 | One sentence for the room. |

## Intent

Leave able to say **external consistency** and **why a multi-region transaction costs latency and availability**, without inventing a clock protocol on the whiteboard. Leave able to refuse a cross-region transaction when a shard-local commit plus async replication is enough.

## What the paper is (one breath)

A globally distributed database that offers externally consistent transactions: the system behaves as if every transaction committed in a single global order that matches real time. That property is rare and expensive. Spanner pays for it with synchronized time (TrueTime), consensus per shard (Paxos in the paper), and careful commit protocols across shards.

You will not build Spanner. You will borrow the **cost argument** and the **refusal**.

## Ideas you can spend

### External consistency (what to say)

Not "strong consistency" as an adjective. External consistency means: if transaction T1 commits before T2 starts in real time, then T1's effects are visible to T2. Readers do not see a reorder that real time forbids.

**Say this:**

> "If the product needs 'this debit happened before that credit in the real world, and every reader agrees,' that is external consistency. Most pastebin reads do not need it. A multi-region money move might."

Tie to day 30: linearizability on one row at one primary is already what you buy locally. Spanner's claim is that property **across** machines and regions, for transactions that may touch many rows.

### TrueTime is a dependency

TrueTime exposes an interval of uncertainty for "now." Commit protocols use that interval (including a **commit wait** while uncertainty passes) so commit timestamps line up with real time.

**In the room:**

> "TrueTime is a dependency I do not invent on a whiteboard. If we need Spanner-like external consistency, we buy a database that already runs synchronized time and consensus. I will not draw GPS clocks or atomic clocks in this hour."

**Failure altitude:** if time uncertainty spikes, commit wait grows — **write latency** rises, or commits stall. That is a user-visible create/update slowdown or timeout, not a theoretical footnote. Page on commit latency and time-uncertainty, not only on CPU.

### Global transactions are expensive

A transaction that touches rows in multiple shards (and especially multiple regions) must coordinate. Consensus rounds, locks or optimistic validation, and commit wait stack up.

**Numbers you can invent out loud (label them assumptions):**

- Same-zone shard-local commit: on the order of **single-digit milliseconds** once consensus is healthy.
- Cross-region coordinated commit: often **tens to low hundreds of milliseconds**, dominated by WAN RTTs and wait — say your assumed RTT (e.g. 60 ms coast-to-coast) and that a multi-region commit pays **multiple** of that, plus uncertainty wait.
- Under partition: a minority region **cannot** complete a commit that needs a quorum across the partitioned set — writes **fail closed** or wait until the partition heals, depending on what you bought.

**Say the trade-off:**

> "I will keep the transaction inside one shard when I can. Cross-region two-phase commit buys a global order and costs latency and a larger unavailable set when a region is sick."

### When you refuse

| Ask | Refusal |
|---|---|
| "Make every read globally linearizable" | Refuse for public CDN-cached reads; name the stale window instead. |
| "One transaction across two regions on every click" | Refuse; shard-local commit + async replicate; accept lag on the secondary. |
| "Explain TrueTime internals" | "Dependency I buy, not one I design in forty minutes." |
| "We'll invent our own global clock" | Refuse. That is a research project, not an interview design. |

## User-visible failures

- **Commit wait / uncertainty spike:** updates hang past the client's patience → timeouts; retries need idempotency (day 37).
- **Region partition during multi-region commit:** transaction aborts or stalls → user sees failed checkout / failed transfer, not a silent half-commit if the database is honest.
- **Read your writes across regions without the global property:** user posts in region A, reads in region B, misses their own write — day 34's problem; Spanner is one way to buy out of it; async replica + sticky primary is another.

## Tie-back to this course

| Day | Already owns | This note adds |
|---|---|---|
| 30 | Linearizability per operation; what you refuse | Global / external framing |
| 40 | Transactions stop at the shard | Why crossing shards/regions is a different product cost |
| 43 | Active-passive vs active-active | Spanner-like global write as the expensive alternative |
| 65–66 | Payments, reconciliation | When money might justify the cost — still prefer idempotent ledger + careful scope |

## Say this in the room (full breath)

> "External consistency means commits line up with real time across the deployment. I get that from a database that already runs synchronized time and per-shard consensus — I do not invent TrueTime here. Most operations in this design are shard-local: one partition, one commit. If you require a cross-region multi-row transaction on the click path, I will say the latency assumption and the unavailable set when a region is dark, and I will try to shrink the transaction to one shard first. I refuse a global transaction as the default for reads that can be stale inside a named TTL."

## Close

Stop at 40. Raft is the appendix for **leader / log / election window** vocabulary. Spanner is not Raft. Do not blend them into one protocol lecture.

# Curriculum

> **This is the map, not today's lesson.** Days 1–49 are written. Start at [Day 1](days/01-what-the-interview-is-grading.md). Days 50–90 are titles until a page exists.

Interview-depth system design for senior and staff loops. One day, one sitting. Pure distributed systems: no AI or ML lessons.

This file is the 90-day assignment list, adapted from the course topic list for this repo. **Phase 1 (days 1–7), Phase 2 (days 8–28), and Phase 3 (days 29–49) are written** under `days/`. Days 50–90 are assigned below and are not in the repo yet. Do not treat a title as a lesson until a page exists.

Problem pages under [`prompts/`](prompts/README.md) are the problem bank. The folder is still named `prompts/`. If a problem file is updated, **the problem file wins** over a title here. Read the problem page on this site before a mock.

Appendices at the bottom are **open**: optional, not part of the 90 days, and not a substitute for a mock. The notes here are the whole appendix until a loop is actually on the calendar. Pointers: [`appendices/README.md`](appendices/README.md).

How to run a day: [`README.md`](README.md). Diagram blanks: [`stencils/README.md`](stencils/README.md). Log: [`design-log/TEMPLATE.md`](design-log/TEMPLATE.md).

---


Interview-depth system design for senior and staff loops. One day, one sitting, one artifact trail in a design log. Pure distributed systems: no AI or ML lessons.

- Audience is engineers with 10+ years of experience, targeting senior and staff loops up to about 20 years of depth, not new grads.
- Self-paced on GitHub, 30–40 minutes a day, text and diagrams in v1. Video may come later and is not required. No live cohort.
- One shared method for every company loop. Depth stays something you can say on a whiteboard. Company quirks and classic papers are optional appendices after day 90, not extra days.
- No certificate. The design log is the artifact. Problem pages are kept current by Arunveer and team. The kit is a separate product and is not required to start.

Days 1–6 teach the method on a pastebin. That pastebin is the closed-book problem on day 7 only, because the lesson topics that week are the method steps, not a second product. From day 28 on, a mock problem is never that week's lesson topic.

## Phase 1 — Hearing the problem (Days 1–7)

Install the hour: requirements and numbers before boxes, then a diagram you can redraw. Pastebin is the worked example.

| Day | Title | Type | Intent |
|---|---|---|---|
| 1 | [What the interviewer is grading](days/01-what-the-interview-is-grading.md) | Lesson | Facing a senior or staff interview, leave able to separate graded signal (scope, trade-offs, operability) from a technology tour. |
| 2 | [Vague problem to requirements](days/02-vague-problem-to-requirements.md) | Lesson | Facing a one-line problem, leave able to lock behavior, constraints, and non-goals before any box is drawn. |
| 3 | [Capacity math out loud](days/03-capacity-math-out-loud.md) | Lesson | Facing a blank estimate, leave able to compute QPS, storage, and bandwidth out loud with every assumption visible. |
| 4 | [API and data model first](days/04-api-and-data-model-first.md) | Lesson | Facing pressure to draw servers, leave able to put endpoints, retention, and the real lookup key down first. |
| 5 | [One box, and why it breaks](days/05-one-box-and-why-it-breaks.md) | Lesson | Facing pressure to distribute immediately, leave able to show a correct single box and the first limit that forces a split. |
| 6 | [A diagram that survives](days/06-a-diagram-that-survives.md) | Lesson | Facing a sketch that will not take questions, leave able to redraw context, whiteboard, read path, and write path from memory. |
| 7 | [Timed dry run (pastebin)](days/07-timed-dry-run-pastebin.md) | Mock | Facing a closed-book timer, leave able to run the pastebin in about 35 minutes with no notes, score it, and file the first design-log entry. |

## Phase 2 — First distributed design (Days 8–28)

Each new box is a fix for a break in the pastebin, not a catalog of tools. Day 28 is a different problem. Do not treat this map as the boxes that mock requires.

| Day | Title | Type | Intent |
|---|---|---|---|
| 8 | [The second box](days/08-the-second-box.md) | Lesson | Facing a saturated process, leave able to split a stateless app tier and say where session and file state still sit. |
| 9 | [Load balancing and draining](days/09-load-balancing-and-draining.md) | Lesson | Facing uneven load, leave able to choose balancing, health checks, and drain behavior and what each does to in-flight work. |
| 10 | [Cache-aside for the melted read](days/10-cache-aside-for-the-melted-read.md) | Lesson | Facing a database melted by reads, leave able to add cache-aside and name the miss path and what must not be cached. |
| 11 | [Hot-key stampede](days/11-hot-key-stampede.md) | Lesson | Facing a hot pastebin key, leave able to stop a stampede without treating a TTL as invalidation. |
| 12 | [SQL or NoSQL from the broken query](days/12-sql-or-nosql-from-the-broken-query.md) | Lesson | Facing a query the current store cannot serve, leave able to choose SQL or NoSQL from the access pattern and name what you give up. |
| 13 | [Replication and the lagging read](days/13-replication-and-the-lagging-read.md) | Lesson | Facing a durability or read-scale wall, leave able to add replicas and refuse a lagging replica for reads that cannot lie. |
| 14 | [Object storage for the body](days/14-object-storage-for-the-body.md) | Lesson | Facing large bodies in the database, leave able to move bytes to object storage and name the orphan-object failure. |
| 15 | [CDN in front of public bytes](days/15-cdn-in-front-of-public-bytes.md) | Lesson | Facing origin bandwidth on public reads, leave able to place a CDN and say what stays origin-authoritative. |
| 16 | [Queue for work the user does not wait on](days/16-queue-for-work-the-user-does-not-wait-on.md) | Lesson | Facing work that should not block the response, leave able to add a queue and describe the backlog the user can see. |
| 17 | [Rate limits and abuse](days/17-rate-limits-and-abuse.md) | Lesson | Facing a public write API, leave able to put rate limits and abuse control on the path before the expensive store. |
| 18 | [Consistent hashing when nodes change](days/18-consistent-hashing-when-nodes-change.md) | Lesson | Facing cache or storage nodes that come and go, leave able to use consistent hashing and say when a fixed slot map is simpler. |
| 19 | [Backpressure and load shedding](days/19-backpressure-and-load-shedding.md) | Lesson | Facing a dependency slower than arrivals, leave able to apply backpressure or shed load and say who feels it. |
| 20 | [Timeouts and retry storms](days/20-timeouts-and-retry-storms.md) | Lesson | Facing a slow dependency, leave able to set timeouts and retries so a blip cannot multiply into a storm. |
| 21 | [Capacity redo with visible assumptions](days/21-capacity-redo-with-visible-assumptions.md) | Lesson | Facing a design that only works at the average, leave able to redo the estimate with fan-out and a stated cache-hit assumption. |
| 22 | [Noisy neighbor and fairness](days/22-noisy-neighbor-and-fairness.md) | Lesson | Facing tenants on one cluster, leave able to cap a noisy neighbor so one key cannot spend the whole budget. |
| 23 | [Read path and write path](days/23-read-path-and-write-path.md) | Lesson | Facing one tangled picture, leave able to narrate the read path and the write path as separate sequences. |
| 24 | [The order-of-magnitude break](days/24-the-order-of-magnitude-break.md) | Lesson | Facing a large jump in traffic, leave able to name the first component that breaks and the fix you would reach for next. |
| 25 | [Failure overlay on the pastebin](days/25-failure-overlay-on-the-pastebin.md) | Lesson | Facing "what if this dies," leave able to overlay one dependency failure and the user-visible result. |
| 26 | [End-to-end distributed pastebin](days/26-end-to-end-distributed-pastebin.md) | Lesson | Facing a full loop on the spine, leave able to assemble the distributed pastebin in one interview-shaped pass. |
| 27 | [Red-team before they do](days/27-red-team-before-they-do.md) | Lesson | Facing your own finished design, leave able to find the holes a staff interviewer would open and patch the reasoning. |
| 28 | [Mock: image upload and thumbnails](days/28-mock-image-upload-and-thumbnails.md) | Mock | Facing an interview problem that is not the pastebin, leave able to design image upload and thumbnails from the product behavior, then log a self-score. |

## Phase 3 — Data and consistency (Days 29–49)

Guarantees, keys, and the failure you accept. The pastebin is still the worked example. Consensus stays a dependency; Raft is appendix reading, not this phase. Day 49 is a different problem. Do not treat this map as the design that mock requires.

| Day | Title | Type | Intent |
|---|---|---|---|
| 29 | [The guarantee per operation](days/29-the-guarantee-per-operation.md) | Lesson | Facing the word "consistent," leave able to state the guarantee per operation instead of one label for the whole system. |
| 30 | [Linearizability and what you refuse](days/30-linearizability-and-what-you-refuse.md) | Lesson | Facing a demand for linearizability, leave able to say which operations need it and which you will refuse. |
| 31 | [Partition key, hash, and range](days/31-partition-key-hash-and-range.md) | Lesson | Facing one hot database, leave able to pick a partition key and say what hash versus range gives up. |
| 32 | [Hot keys and salting](days/32-hot-keys-and-salting.md) | Lesson | Facing a celebrity key, leave able to salt or split it without destroying the lookup you still need. |
| 33 | [Invalidation and TTL](days/33-invalidation-and-ttl.md) | Lesson | Facing stale or thundering cache entries, leave able to choose invalidation or TTL and the stale window you accept. |
| 34 | [Lag, monotonic reads, read-your-writes](days/34-lag-monotonic-reads-read-your-writes.md) | Lesson | Facing replica lag, leave able to give the right operations monotonic reads or read-your-writes. |
| 35 | [Quorums in plain language](days/35-quorums-in-plain-language.md) | Lesson | Facing a quorum design, leave able to explain what N, R, and W buy and the unavailable or stale outcome you took. |
| 36 | [A leader is a dependency](days/36-a-leader-is-a-dependency.md) | Lesson | Facing a single-writer partition, leave able to treat leader election as a dependency with an unavailable window, not as a protocol lecture. |
| 37 | [Idempotency on an at-least-once path](days/37-idempotency-on-an-at-least-once-path.md) | Lesson | Facing retries, leave able to make an at-least-once write safe with idempotency keys and name what can still arrive twice. |
| 38 | [Dedupe and the inbox](days/38-dedupe-and-the-inbox.md) | Lesson | Facing a consumer that can see a message twice, leave able to dedupe with an inbox and a defined retention for those keys. |
| 39 | [Outbox, not dual write](days/39-outbox-not-dual-write.md) | Lesson | Facing a write that must also become a message, leave able to replace a dual write with an outbox. |
| 40 | [Transactions that stop at the shard](days/40-transactions-that-stop-at-the-shard.md) | Lesson | Facing a multi-row change, leave able to draw the transaction at one shard and pick an isolation level on purpose. |
| 41 | [Sagas and compensation](days/41-sagas-and-compensation.md) | Lesson | Facing a workflow that crosses partitions, leave able to use a saga and name the compensation and the crash window. |
| 42 | [Conflicts you can explain](days/42-conflicts-you-can-explain.md) | Lesson | Facing two writers, leave able to choose versions, last write, or a merge, and say when a CRDT is worth naming. |
| 43 | [Active-passive or active-active](days/43-active-passive-or-active-active.md) | Lesson | Facing users in more than one region, leave able to choose a write topology and the conflict or failover cost that comes with it. |
| 44 | [Clocks, ids, and order](days/44-clocks-ids-and-order.md) | Lesson | Facing order across machines, leave able to pick ids and a clock story and name the reorder bug you still have. |
| 45 | [Schema change and backfill](days/45-schema-change-and-backfill.md) | Lesson | Facing a schema that must change, leave able to plan expand, backfill, and cutover without stopping writes, plus a rollback. |
| 46 | [Delete, tombstone, retention](days/46-delete-tombstone-retention.md) | Lesson | Facing a delete, leave able to use tombstones and retention so replicas and indexes do not resurrect the row. |
| 47 | [Secondary indexes and data-path cost](days/47-secondary-indexes-and-data-path-cost.md) | Lesson | Facing an extra index or replica, leave able to state the write amplification, storage class, or egress you just bought. |
| 48 | [Close the data chapter](days/48-close-the-data-chapter.md) | Lesson | Facing the end of the data section, leave able to close on guarantee, partition plan, and the failure you accept. |
| 49 | [Mock: warehouse inventory reservation](days/49-mock-warehouse-inventory-reservation.md) | Mock | Facing a problem that is not this week's schema, index, or retention lesson, leave able to reserve inventory so two buyers cannot take the last unit, then log a self-score. |

## Phase 4 — Product-shaped systems (Days 50–77)

**Coming. Not written.** These rows are titles only. They are not links, so there is no empty page to open.


One product a day, or a variant whose failure mode is new. Inventory on day 49, assigned seats on day 61, and a flash-sale queue on day 70 are three different problems. Ranking models stay out.

| Day | Title | Type | Intent |
|---|---|---|---|
| 50 | URL shortener | Lesson | Facing a redirect path, leave able to design create and redirect so a hot link cannot pin one row or one cache key. |
| 51 | News feed | Lesson | Facing a home feed, leave able to choose fan-out-on-write or fan-out-on-read from follower skew, with ranking models left out. |
| 52 | Celebrity fan-out | Lesson | Facing one author with a huge audience, leave able to hybrid-fan-out so a single post cannot stampede storage or the online cluster. |
| 53 | One-to-one chat | Lesson | Facing two people messaging, leave able to keep per-conversation order, receipts, and offline catch-up without a global total order. |
| 54 | Large-room chat | Lesson | Facing a huge room, leave able to serve history and fan-out without copying every message to every member on the send path. |
| 55 | Notifications | Lesson | Facing a user who is not in the app, leave able to design push plus an inbox with preferences, dedupe, and a dead provider. |
| 56 | Typeahead | Lesson | Facing a prefix box, leave able to serve suggestions from a popularity-biased index and bound how stale a suggestion may be. |
| 57 | Product search | Lesson | Facing catalog search rather than a prefix, leave able to query an inverted index with facets and explain lag after a write. |
| 58 | Video on demand | Lesson | Facing a long video, leave able to take upload through transcode to CDN playback without bytes sitting on the app tier. |
| 59 | Live video | Lesson | Facing a live stream, leave able to ingest with adaptive bitrate and survive the stampede when the stream starts. |
| 60 | Nearby | Lesson | Facing "what is near me," leave able to index points, bound staleness, and avoid leaking a precise location. |
| 61 | Ticket booking | Lesson | Facing assigned seats, leave able to hold specific seats with a TTL so you neither oversell nor lock the whole venue. |
| 62 | Metrics ingestion | Lesson | Facing a firehose of measurements, leave able to ingest points with a cardinality limit so one tenant cannot blow the store. |
| 63 | Mock: job scheduler | Mock | Facing an unseen problem, leave able to design a job scheduler with leases, missed fires, and duplicate runs, not this week's search, video, location, ticket, or metrics lessons. |
| 64 | Metrics query and alerts | Lesson | Facing graphs and pages rather than ingest, leave able to query downsampled series and stop one alert rule from paging on every blip. |
| 65 | Payments | Lesson | Facing money movement, leave able to authorize and capture against an idempotent ledger so a retry cannot move money twice. |
| 66 | Payment reconciliation | Lesson | Facing a provider and a ledger that disagree, leave able to find the mismatch and repair it without a silent rewrite of history. |
| 67 | Collaborative document | Lesson | Facing two editors in one document, leave able to persist concurrent edits and presence and say what you do when they diverge. |
| 68 | File sync | Lesson | Facing folders that must match across devices, leave able to sync chunks, resolve conflicts, and survive a client clock that lies. |
| 69 | Cart and checkout | Lesson | Facing a purchase, leave able to snapshot price and hold stock so a retry cannot double-charge or double-commit inventory. |
| 70 | Flash sale | Lesson | Facing an arrival spike on scarce stock, leave able to queue buyers with a fairness rule instead of letting them pile onto the database. |
| 71 | Social graph | Lesson | Facing follow and block, leave able to store the graph so a privacy check stays correct on the read path. |
| 72 | Ephemeral stories | Lesson | Facing posts that must disappear, leave able to fan them out with a TTL that is real in storage and caches, not only in the UI. |
| 73 | Web crawler | Lesson | Facing the public web as input, leave able to run a polite frontier with dedupe and freshness so one host is not melted. |
| 74 | Order book | Lesson | Facing buy and sell orders, leave able to match one instrument on a single sequence and rebuild the book after the matcher crashes. |
| 75 | Feature flags | Lesson | Facing a risky rollout, leave able to serve a flag or kill switch at low latency and bound how stale a client may be. |
| 76 | Audit log | Lesson | Facing "who did that," leave able to append a queryable activity log with retention and a tamper claim you can actually keep. |
| 77 | Mock: email inbox | Mock | Facing an unseen product, leave able to design a large email inbox (ingest, folders, search, attachments), not this week's graph, stories, crawler, book, flags, or audit lessons. |

## Phase 5 — Failure and senior signal (Days 78–84)

**Coming. Not written.** These rows are titles only. They are not links, so there is no empty page to open.


Staff credit is the failure you can operate, pay for, and migrate. Day 84 is a product mock with a scripted outage, not a repeat of these lectures.

| Day | Title | Type | Intent |
|---|---|---|---|
| 78 | SLOs and the miss a user sees | Lesson | Facing a vague reliability ask, leave able to state an SLO and the user-visible miss when it is breached. |
| 79 | Degradation under partial failure | Lesson | Facing a sick dependency rather than a total outage, leave able to pick a degraded mode and what you refuse to drop. |
| 80 | Losing a region | Lesson | Facing a region that is already dark, leave able to state RPO, RTO, and which side is authoritative for new writes. |
| 81 | Observability that pages a human | Lesson | Facing an on-call question, leave able to name the dashboard, the page, and the first runbook step for a failure you designed. |
| 82 | Cost as a spoken trade-off | Lesson | Facing a design that is correct and expensive, leave able to cut replicas, egress, or retention and say the user-visible risk. |
| 83 | Migration while live | Lesson | Facing a system that must change shape, leave able to migrate with a dual path and a rollback while writes continue. |
| 84 | Mock: delivery dispatch with a late failure | Mock | Facing a dispatch problem that is not an SLO or migration lecture, leave able to design matching and, near minute 25, lose a region or the location store. |

## Phase 6 — Mocks and the kit (Days 85–90)

**Coming. Not written.** These rows are titles only. They are not links, so there is no empty page to open.


Three full loops: read-heavy, write-heavy, realtime. Debriefs are their own days. Day 90 is kit-run: no lesson beside the timer.

| Day | Title | Type | Intent |
|---|---|---|---|
| 85 | Kit hookup and the design log | Kit | Facing the last mocks without a shared ritual, leave able to run a timer, the senior rubric, and a design log, and see where the separate kit plugs in. |
| 86 | Mock: article serving (read-heavy) | Mock | Facing a read-heavy problem that is not the kit-setup lesson, leave able to design article serving with hot pages and invalidation on publish, inside the time box. |
| 87 | Debrief: read-heavy mock | Debrief | Facing the day-86 writeup, leave able to score it on the senior rubric and rewrite only the weakest section into the design log. |
| 88 | Mock: event intake (write-heavy) | Mock | Facing a write-heavy problem, leave able to design event intake with dedupe, late events, and a durable sink, and leave a complete design-log entry. |
| 89 | Debrief: write-heavy mock | Debrief | Facing the day-88 writeup, leave able to score it, compare it with day 86, and log the one staff-level gap to close before the final mock. |
| 90 | Mock: realtime lobby (kit run) | Mock | Facing a realtime loop with no lesson beside it, leave able to run the kit's current problem (fallback: multiplayer lobby) and leave the design log as the only artifact. |

## Optional appendices

Not part of the 90 days. Do not swap one in for a mock. Each is a single sitting under 40 minutes, or skip it until a loop is actually on the calendar.

### Company quirk appendix

One method everywhere: requirements, estimates, API and data, design, deep dive, failure. Use this only to bias the hour.

| Loop | Thin bias |
|---|---|
| Google | More time on the estimate, the data model, and naming the consistency guarantee. Non-goals are expected to be explicit. |
| Meta | The problem stays ambiguous longer. Fan-out, what the user waits for, and what can be async show up early. |
| Amazon | Customer-visible behavior and operational ownership before boxes. "What pages you" is a normal deep dive, not a personality test bolted on the end. |
| Apple | Privacy and data minimization are a first-class constraint: what you refuse to store, and what can stay on device. Scale still has to be answered. |
| Netflix | Expect a region or a dependency to die in the conversation, and expect cost of a heavy media path to be spoken, not appended. |
| Microsoft | Tenancy, identity boundaries, and an enterprise isolation story sit beside scale. Do not switch into a compliance lecture. |

If the problem bank adds a company card later, that card wins over this table.

### Classic papers appendix

Recommend, not required. Take only what you can use in an answer. Do not summarize the paper in the room.

- **Dynamo.** Useful for consistent hashing, hinted handoff, sloppy quorum, and a failure model where nodes are wrong or gone. Conflict handling stays application-specific. Do not cite the paper as permission to skip a consistency choice.
- **Spanner.** Useful for the idea of external consistency and for why a global transaction is expensive. TrueTime is a dependency you do not invent on a whiteboard. Say when you would refuse a cross-region transaction instead.
- **Raft.** Useful so leader, log, and an election window are concrete when you call consensus a dependency. You are not there to derive the protocol or to propose building your own.

## How to use this on GitHub

The course lives in the GitHub repo as text and diagrams. Work one numbered day at a time, in order. Stop at 40 minutes even if you are mid-sentence: the cap is the point. There is no cohort and no certificate.

Suggested layout, which the repo can mirror:

- `days/NN-title.md` for the lesson or mock page
- `prompts/` for the problem bank (the folder name stays `prompts/`)
- `design-log/` for your entries
- `appendices/` for the quirk note and the paper note

On a lesson day, attempt the problem before you read the design, then read, then redraw the six diagram types from Week 1 (context, whiteboard, read path, write path, data model, failure or scale overlay). On a mock day, use a timer, a blank page, and no notes from that week. Read the reference only after you stop.

Design-log entry, every mock and any day you want to keep: problem, assumptions, a small rubric score, one gap. That log is the artifact.

Mock rule: the problem is never that week's lesson topic. Day 7 is the exception that installs the rubric: the week's lessons are the method, and the closed-book problem is the pastebin you already touched. Later mocks are different products on purpose.

| Day | Problem | Not this week |
|---|---|---|
| 7 | Pastebin, closed book | Method lessons only; this mock installs the rubric |
| 28 | Image upload and thumbnails | Not the pastebin spine. Closed book. Score behavior, not a required box list |
| 49 | Warehouse inventory reservation | Not schema change, indexes, or retention. Seats are day 61 |
| 63 | Job scheduler | Not search, video, nearby, tickets, or metrics ingest |
| 77 | Email inbox | Not graph, stories, crawler, order book, flags, or audit |
| 84 | Delivery dispatch | Not the SLO, failover, or migration lectures. A region or the location store dies near minute 25 |
| 86 | Article serving | Read-heavy. Not the day-85 setup lesson |
| 88 | Event intake | Write-heavy. Not a replay of the metrics days |
| 90 | Kit realtime card, else multiplayer lobby | No lesson. Realtime |

Titles in this list are the initial assignment. If a page under `prompts/` is updated, the updated problem wins. Pull before a mock. Do not keep a private copy as the source of truth.

The kit (script, stencils, trade-off card, follow-up bank, grader notes, extra problem cards) is a separate product sold on its own. Day 85 shows how it attaches. Days 1–84 and the debriefs do not require it. Day 90 is written to be run from the kit; until you own it, use the fallback lobby problem and the same timer and rubric. Do not block the other 89 days on the kit.

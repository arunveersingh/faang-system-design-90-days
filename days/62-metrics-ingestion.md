<!-- day-nav -->
[← Day 61 — Ticket booking](61-ticket-booking.md) · [Day 63 — Mock: job scheduler →](63-mock-job-scheduler.md)

# Day 62 — Metrics ingestion

**Do now**

1. Set a timer.
2. Attempt the problem. Stop at the attempt line. Do not scroll.
3. Then read.

## Time box

35 minutes.

| Minutes | Do this |
|---|---|
| 0–10 | Attempt. Firehose of measurements. Cardinality limit so one tenant cannot blow the store. |
| 10–28 | Read. If unbounded label combos are allowed, the store dies. |
| 28–35 | Say the write path, aggregation, and how you enforce cardinality. |

## Intent

Facing a firehose of measurements, leave able to ingest points with a cardinality limit so one tenant cannot blow the store.

## Problem

> Design metrics ingestion.
>
> Services send time-series metrics with labels. You store them for later query and alerting. One noisy tenant must not be able to create infinite series and take the system down.

## Attempt before reading

10 minutes. Do not scroll. Query/alerts are day 64.

Write:

1. Wire format and auth (tenant).
2. Write path: buffer, shard, durable store.
3. What a series key is; cardinality definition.
4. Enforcement when tenant exceeds series budget.
5. Out-of-order / duplicate points.

---

**Stop. Tenant-scoped cardinality. Append-optimized TSDB path. Shed new series hard.**

---

## Requirements

**In.** Multi-tenant push of gauges/counters/histograms. Retention raw **15 days**, downsampled longer (query day). Per-tenant series limit (e.g. **100k–1M** active series). At-least-once delivery from agents.

**Assumptions.** **10 billion** points/day ≈ **116k**/s average, **~500k**/s peak. **5,000** tenants; one whale.

**Out.** Full PromQL surface today, log ingestion, tracing backend, day-64 alert rules deep dive.

## Estimates

500k points/s × 20 bytes compressed ≈ **10 MB/s** — fine; **cardinality** and index of series keys is the cliff. 1e9 series × metadata kills memory.

## API and data

`POST /v1/metrics/write` protobuf/HTTP, tenant key.

Series identity: `tenant + metric_name + ordered labels`.

Store: time-partitioned chunks per series; separate series catalog with counts per tenant.

## Design

### Ingest path

Gateway auth → validate → buffer (queue) → distributors hash by series → ingesters → object/block storage. Ack after durability bar you state (WAL quorum).

### Cardinality control

Track active series per tenant (approx OK with HF counts). On new series insert: if over limit → **reject point** or drop labels into `overflow` bucket — prefer **429/400 with metric** so the customer fixes. Existing series continue. Page tenant owners; do not let one tenant create 50M series overnight.

Relabel / drop rules at gateway for known bad patterns (`user_id` on metric labels).

### Duplicates / order

Same timestamp+series: last write wins or first — say it. Out-of-order window bounded (e.g. 1 h); older rejected or sent to repair path.

## Diagrams

```mermaid
flowchart LR
  agent[Agent] --> gw[Gateway + limits]
  gw --> q[Buffer]
  q --> dist[Distributor]
  dist --> ing[Ingesters]
  ing --> blocks[(Blocks)]
  gw --> card[Cardinality tracker]
```

Caption: "Limits before fan-out. New series are the scarce resource."

## Failure the user sees

**Tenant over card.** New series fail; dashboards miss new pods until they drop labels. Old series keep writing.

**Ingester down.** Quorum write may 503; agents retry. Gaps possible — say scrape retry.

## Trade-offs

**Exact vs approx card counts.** Approx can admit slightly over; exact is hotter. Prefer protect-the-store.

**Reject vs silent drop.** Reject — silent drop hides cardinality bugs.

## Talking points

**If they index every label forever.** "That is how tenants melt you. Budgets and gateway drop rules."

## Say this in the room

Ingest is a buffered, hash-sharded write path into a time-series store, but the scarce resource is active series — not raw points. Each tenant has a hard cardinality budget enforced at the gateway on new series creation so one customer cannot invent infinite label combos overnight. Existing series still accept points; over-budget new series get a clear error. Duplicates and limited out-of-order windows have an explicit rule. Query and alerts are tomorrow; today I keep the firehose from becoming an unbounded index.

### Staff depth: series are the scarce resource

500k points/s is easy compressed; **1e9 series** of metadata is not. Per-tenant active-series budget at the gateway on **new** series; existing series keep writing. Reject with a clear error — silent drop hides the bug.

**Drop rules.** Ban `user_id` as a metric label at ingest.

**What staff sounds like.** Cardinality fuse before naming TSDB brands; out-of-order window stated.

## Kit artifact

| Follow-up | You answer with |
|---|---|
| "Tenant explodes labels." | Gateway card limit; reject new series; drop-rule examples. |

## Design log

One line: your per-tenant series limit and whether you reject or silently drop.

Next: [Day 63 — Mock: job scheduler](63-mock-job-scheduler.md). Closed book — not this week's search/video/geo/tickets/metrics.

---

<!-- day-nav -->
[← Day 61 — Ticket booking](61-ticket-booking.md) · [Day 63 — Mock: job scheduler →](63-mock-job-scheduler.md)

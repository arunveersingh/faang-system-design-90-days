<!-- day-nav -->
[← Day 63 — Mock: job scheduler](63-mock-job-scheduler.md) · [Day 65 — Payments →](65-payments.md)

# Day 64 — Metrics query and alerts

**Do now**

1. Set a timer.
2. Attempt the problem. Stop at the attempt line. Do not scroll.
3. Then read.

## Time box

40 minutes.

| Minutes | Do this |
|---|---|
| 0–12 | Attempt. Graphs and alerts on stored series. Stop one rule from paging on every blip. |
| 12–32 | Read. Ingest was day 62. Today is read path, downsample, and alert fatigue. |
| 32–40 | Say how a dashboard query is served, and how alerts debounce/flap. |

## Intent

Facing graphs and pages rather than ingest, leave able to query downsampled series and stop one alert rule from paging on every blip.

## Problem

> Design metrics query and alerts.
>
> People open dashboards of time-series graphs. Alert rules page a human when a condition holds. A noisy rule must not page on every blip.

## Attempt before reading

12 minutes. Do not scroll.

Write:

1. Query API: range, step, aggregations.
2. Where raw vs downsampled data lives; which the UI hits for 7-day vs 6-hour views.
3. Alert evaluation loop; pending → firing → resolved.
4. Flapping: hysteresis, for-duration, silence, grouping.
5. One bad rule × 10,000 series — cardinality of alerts.

---

**Stop. Downsampled read path. Alert state machine with for-duration. Group pages.**

---

## Requirements

**In.** Range queries with aggregations (sum/avg/p99 over labels). Dashboards at **1–5 s** interactive budget for typical panels. Alerts: threshold/ Prom-like expressions, `for: 5m`, notify Slack/Pager. Silences. Retention: raw 15d, 1m/5m rolls longer (day 62).

**Assumptions.** **50k** dashboard QPS peak? No — **5k** panel queries/s peak more realistic. **20,000** alert rules; eval every **15–30 s**. One region.

**Out.** Building the ingest path again, log search, full anomaly ML.

## Estimates

Eval: 20k rules × scrape of matched series — must index rules carefully; naive Cartesian explodes. Alert managers handle **thousands** of active alerts, not millions of pages/minute.

## API and data

`GET /v1/query_range?query=&start=&end=&step=`

Alert rule store; alert instance state `(rule_id, fingerprint) → pending|firing`.

Notify outbox.

## Design

### Query

Query frontend fans to queriers; choose resolution by range (6h → raw/1s; 30d → 5m). Compactor downsamples in background. Cache recent dashboard queries briefly.

### Alerts

Evaluator loads rules, computes expression over window. **Pending** until `for` duration continuous breach → **firing** → notify. Resolve when clear for a duration. **Hysteresis**: fire at 90%, resolve at 70% optional.

**Grouping / aggregation:** page once per `{alertname, service}` not per replica. Repeat notifications spaced (e.g. 4h).

**Bad rule protection:** max series matched per rule; max alerts created per eval; circuit-break the rule; page the rule owner.

### Failure

Querier down: dashboards 503; alerts may use separate path. Alert manager down: buffer eval or risk delayed pages — page on alert-manager lag. Never let eval stall ingest (already separate).

### Staff arithmetic: raw vs downsampled, alert fan-out

A 30-day graph at 1 s resolution for one series is ~2.6e6 points — fine alone; times a dashboard of 50 charts times 5k viewers is how query melts. Downsample to 1 min / 5 min / 1 h blocks by age; query planner picks the step from the time range. Alert rules: **for-duration** (e.g. 5 minutes below SLO) is the anti-flap; without it a blip pages at 3 a.m. Grouping by service so one bad deploy pages once, not once per replica. A rule that expands to a million series gets a hard cap and is **disabled**, not "best effort."

## Diagrams

```mermaid
flowchart TB
  ui[Dashboard] --> q[Query frontend]
  q --> raw[Raw blocks]
  q --> ds[Downsampled]
  ev[Evaluator] --> state[Alert state]
  state --> am[Alert manager]
  am --> page[Pager / Slack]
```

Caption: "Reads choose resolution. Alerts are a state machine, not a raw threshold on every sample."

## Failure the user sees

**Dashboard stampede.** Query path saturates; graphs spin. Must not stop alert evaluation — isolate pools. User-visible: slow dashboards, alerts still fire.

**Alert flap.** No for-duration → page every scrape blip. On-call hatred is a product failure of the rule design.

**Rule cardinality bomb.** One PromQL-shaped regex expands to 1e6 series; notifier and eval melt. Cap and disable; page the rule owner.

**Ingest healthy, eval lagging.** Alerts fire late; SLO burn looks fine until it is not. Page on eval lag vs wall time, not only on ingest QPS.

**Wrong step on long range.** UI asks 30 days at 1 s; query scans forever. Planner must coerce step or 400 the request.

## Trade-offs

**Shared query+eval pool vs split.** Split. Dashboard chaos must not delay pages.

**Exact raw forever vs downsample.** Downsample older data; say retention per resolution.

**Name the refusal inside each alternative.** Against one pool for dashboards and alerts: you refuse on-call latency tied to a viral dashboard. Against no for-duration: you refuse flap. Against unbounded rule fan-out: you refuse eval death. Against serving 30-day 1 s raw as the default: you refuse scan storms. Against paging per replica without grouping: you refuse 50 pages for one deploy.

**10× query QPS.** More query replicas; downsample stays. Alert eval capacity is sized on rule count × series, not on dashboard fashion.

## Talking points

**If they only talk ingest.** "Day 62. Today is read path and pages."

**If they ask how alerts avoid flap.** "For-duration pending→firing; hysteresis on resolve if needed; group by service."

**If they ask what happens when a rule is huge.** "Hard series cap; disable and notify the owner. Do not 'try'."

**If they ask about recording rules.** "Precompute expensive expressions into new series — capacity trade for eval CPU. Optional if time remains."

**If they ask what pages.** "Eval lag, notifier error rate, and disabled-rule count — not dashboard p99 alone."

## Say this in the room

Dashboards hit a query path that picks raw or downsampled blocks based on the time range so a thirty-day graph does not scan fifteen days of one-second samples. Alerts are evaluated on a loop into a pending/firing state machine with a for-duration, hysteresis if needed, and grouping so one bad service pages once, not once per replica. A rule that expands to a million series gets a hard cap and is disabled rather than melting the notifier. Ingest stays isolated — a dashboard stampede must not stop alert eval, and vice versa.

### Staff depth: for-duration is the anti-flap

Pending → firing only after the condition holds for N minutes. Resolve may use hysteresis. Without this, scrape noise pages humans.

**Pool split.** Alert eval and dashboard query do not share a fate.

**Cardinality of rules.** Cap expansion; disable offenders.

**What staff sounds like.** Downsample planner, for-duration, group pages, isolate eval from dashboards.

### More probes, with the answer

**"What pages?"** Name the user-visible lag or error metric from Failure — not only CPU. **"What do you refuse?"** Pick one refusal from Trade-offs and say the lie it prevents. **"What is the sensitive assumption?"** The estimate that flips the design if wrong by 10×.

## Kit artifact

| Follow-up | You answer with |
|---|---|
| "Too many pages." | for-duration, group, silence, rule series cap. |

## Design log

One line: your for-duration and how you stop a Cartesian rule explosion.

Next: [Day 65 — Payments](65-payments.md). Authorize, capture, idempotent ledger.

---

<!-- day-nav -->
[← Day 63 — Mock: job scheduler](63-mock-job-scheduler.md) · [Day 65 — Payments →](65-payments.md)

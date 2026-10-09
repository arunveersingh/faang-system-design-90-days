<!-- appendix-nav -->
[Appendices](README.md) · [Classic papers →](classic-papers.md)

# Company quirks

**Not a day. Not a mock.** Open this only when a real interview is on the calendar at one of these companies. Do not swap it in for Day 28, 49, 63, 77, 84, 86, 88, or 90.

## Time box

35 minutes for the whole table, or 10 minutes on the one company you are walking into.

| Minutes | Do this |
|---|---|
| 0–5 | Read the shared method. Do not invent a second method. |
| 5–25 | Read the row for the company you have. Say the bias out loud once. |
| 25–35 | Pick one practice problem and bias the hour the way the row says. Score yourself on whether the bias showed, not on whether you mentioned the company. |

## Intent

One method everywhere: requirements, estimates, API and data, design, deep dive, failure. This note only **biases where you spend the hour**. It does not change the pastebin spine, the mock rule, or the senior rubric.

If a company card is ever added under `prompts/`, that card wins over this page.

## Shared method (do not abandon)

Every interview still runs:

1. Lock behavior, constraints, and non-goals.
2. Numbers with visible assumptions.
3. API and data model before boxes.
4. Design that survives a redraw.
5. Deep dive on the hard path.
6. Failure the user sees, and what you refuse.

Company bias sits **on top** of that sequence. It does not replace step 1 with a technology tour, and it does not excuse skipping failure.

## Bias table

| Company | Thin bias | Spend more here | Do not |
|---|---|---|---|
| Google | Estimate, data model, named consistency. Non-goals explicit. | Capacity out loud; which operations are linearizable vs stale; what you refuse to store. | Skip the estimate because the product "feels small." Leave "consistent" as an adjective. |
| Meta | Problem stays ambiguous longer. Fan-out and async early. | What the user waits for vs what can be eventual; celebrity / skew; feed or social graph fan-out cost. | Force-close requirements in minute 2 and never reopen them. Hide fan-out behind a single "notification service" box. |
| Amazon | Customer-visible behavior and operational ownership before boxes. | What pages you; runbook first step; ownership of the failure; idempotent money or order paths if relevant. | Treat "what pages you" as a personality add-on at minute 40. Design a perfect box diagram with no on-call story. |
| Apple | Privacy and data minimization as a first-class constraint. | What you refuse to store; what can stay on device; retention; who can read the row. Scale still answered. | Deliver only a privacy lecture with no capacity. Store everything "because analytics" without a refusal. |
| Netflix | Region or dependency dies in the conversation; media cost spoken. | Multi-region RPO/RTO; degraded mode; CDN and origin cost; what you shed under partial failure. | Append cost as a slide after the design. Assume the region never dies. |
| Microsoft | Tenancy, identity boundaries, enterprise isolation beside scale. | Tenant isolation; noisy-neighbor caps; authn/z boundary on the data path; blast radius of one tenant. | Switch into a compliance or GDPR lecture. Ignore scale because "enterprise QPS is low." |

## Say this in the room (per company)

### Google

> "Before boxes: peak QPS, working set, and retention. Non-goals: no full-text search in v1, no cross-region active-active. Creates are linearizable on the primary; public reads may be stale inside the CDN TTL I am about to name."

If they push on consistency without integers: refuse the adjective. Name the operation and the failure (unavailable vs stale).

### Meta

> "The product is still ambiguous — I am locking what the viewer waits for on first paint, and what can fan out async. Celebrity authors get hybrid fan-out so one post cannot stampede the online cluster."

If they keep the problem vague past minute 10: that is the test. Re-state assumptions as bets, not as facts, and keep designing.

### Amazon

> "Customer-visible: create returns 201 only after the row is durable on the primary. If the primary is down, the customer sees 503, not a lying 201. The page that fires is create error rate and 'ack without row,' and the first runbook step is promote-or-fail closed — I own that path."

If they ask "what pages you" early: answer with a metric and a human action, then return to the design. Do not save operability for the last five minutes.

### Apple

> "Constraint: we do not store precise location or raw message bodies longer than the retention the product needs. On-device can hold the draft; the server holds the minimum to sync. Capacity still: peak create rate and the working set of metadata."

If they only want privacy: give the refusal list, then force the scale sentence so the design is still interview-shaped.

### Netflix

> "I am designing for a region loss: RPO for new writes is zero on the active region, RTO is the failover we can operate. Heavy media path: origin egress and encode cost are spoken trade-offs, not a footnote. Under partial failure I shed personalized ranking before I shed playback of the bytes already at the edge."

If cost never comes up: introduce it yourself on the media or multi-region path. Staff credit is the spoken trade-off.

### Microsoft

> "Tenancy: data for tenant A never shares a query path that can return tenant B. Noisy neighbor: per-tenant rate and storage caps so one customer cannot spend the cluster. Identity boundary sits on the data path, not only on the load balancer."

If they drift into policy theater: name the isolation mechanism (key prefix, separate cluster, row-level predicate) and the failure if it breaks, then move on.

## Failure you are biasing toward

| Company | Failure they often open | Your one-line answer shape |
|---|---|---|
| Google | "Is this consistent?" | Per-operation guarantee + unavailable-or-stale outcome. |
| Meta | "What about a celebrity?" | Hybrid fan-out; online path protected; async backlog visible. |
| Amazon | "What pages you at 3am?" | Metric, threshold, first runbook step, customer-visible symptom. |
| Apple | "Why store that?" | Refusal list; retention; on-device vs server split. |
| Netflix | "Us-east is gone." | RPO, RTO, who takes new writes, what degrades. |
| Microsoft | "Tenant X is noisy." | Cap, isolation boundary, blast radius. |

## What this appendix is not

- Not a second curriculum.
- Not permission to skip Days 78–84 (SLOs, degradation, region loss, observability, cost, migration).
- Not a script you recite instead of designing the product in front of you.
- Not company-specific box catalogs. The pastebin method still wins.

## After this sitting

Close the appendix. Tomorrow's work is a numbered day or a closed-book mock, not another company row. If the interview is tomorrow, bias one practice problem once, log the gap, stop.

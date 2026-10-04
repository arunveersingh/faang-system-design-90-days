<!-- day-nav -->
[← Day 19 — Backpressure and load shedding](19-backpressure-and-load-shedding.md) · [Day 21 — Capacity redo with visible assumptions →](21-capacity-redo-with-visible-assumptions.md)

# Day 20 — Timeouts and retry storms

## Time box

35 minutes.

| Minutes | Do this |
|---|---|
| 0–10 | Attempt. A budget for the read, and what a retry does to a dependency that is already late. |
| 10–28 | Read. If you retry creates, or you retry timeouts, fix that before you add a circuit breaker. |
| 28–35 | Say the retry rule in one sentence: which errors, how many, which calls never. |

## Intent

Facing a slow dependency, leave able to set timeouts and retries so a blip cannot multiply into a storm. The timeout is a fix for a call that would hold a slot until the process is useless. The retry rule is a fix for the amplification you add while "being resilient."

## Problem

> Design a pastebin. People paste text and share a link.

The interviewer says: "The primary blips. Your apps retry. What does the primary see, and what does a create do if you retry it?"

## Attempt before reading

10 minutes. Do not scroll. Peak origin reads can be as high as 17,400/s if the edge is cold. Creates peak at about 350/s and are not idempotent. Day 19 capped in-flight work. A slot that never returns is the same as no cap.

Write:

1. A timeout for: metadata lookup, object GET, object PUT. Units. You may be wrong; you may not be silent.
2. What you retry, with a count. What you do not retry even once.
3. The multiplication: if 20% of 17,400 reads time out and you try each of those three more times, what arrives at the dependency?
4. Whether a create retry from **your** app, not from the client, can mint a second paste or a second object.

---

**Stop. Budgets and a narrow retry rule, below.**

---

## Requirements

A user-facing read has an end-to-end budget. You do not promise a global 100 ms, and you do not leave the budget implicit. Planning budget for an origin read in-region: **300 ms** until you have a histogram. Inside that, each hop gets a slice. If the slices add up to more than the budget, you have already decided to blow it. Add them on the whiteboard.

A create is slower because it waits for durability. Its budget is longer and still finite: a PUT that runs for minutes is a stuck slot. Planning create budget: **2 seconds**, then 503, not an infinite spinner.

The contract facts that constrain retries:

- GET of metadata and GET of an object are idempotent.
- POST create is not. The app must not replay it.
- DELETE is idempotent if the row delete and the detach job are safe twice, which they are. A client retry of DELETE after a lost 204 is fine. An automatic storm of DELETEs is still load.
- A timeout does not mean the call failed. It means you stopped waiting. The primary or the bucket may still be doing the work. A retry then **adds** a second copy of that work on top of the first.

That last point is the storm.

## Design

### Budgets

| Call | Timeout | Retry |
|---|---|---|
| Cache get | 5 ms, then treat as miss | No. A slow cache is a miss path, not three more cache calls. |
| Primary point read | 20 ms | No retry on timeout. One retry only if the connection failed **before** the request was sent, and only if the 300 ms budget still has the 20 ms. |
| Object GET | 200 ms | Same rule: no retry on timeout. One immediate retry only on a fast connect failure, with jitter, inside the budget. |
| Object PUT | 1 s inside the 2 s create budget | **No retry from the app.** You do not know whether the bytes landed. |
| Primary insert | 50 ms | **No retry of the whole create.** A retry of the insert alone, with the same id, is safe if the first might have committed: the unique key makes the second a conflict, and you must treat "unique violation" after a timeout as "the row may exist" and then decide. Simplest rule you can say: **do not retry the insert either; return 500; leave an orphan if the PUT landed; the reaper has the age floor.** A clever retry-on-unique-violation is easy to get wrong in the room. Refuse it unless this is the deep dive. |
| CDN purge, from the worker | seconds, worker-side | Retry with backoff. This is not on the user path. At-least-once is the design. |

Jitter means the retry waits a random slice of a small window, tens of milliseconds, not a synchronized sleep that makes every app hit the primary in the same millisecond. One retry plus jitter is the whole client policy. Exponential backoff across five attempts belongs in the **worker**, where nobody is holding an HTTP request, not in the read path.

### The multiplication

Do it out loud.

Origin reads, cold edge, 17,400/s. Suppose **20%** time out. That is 3,480 timed-out reads a second.

- If each retries **three** more times, you add about **10,400** attempts on top of the original 17,400. The dependency sees roughly 17,400 − 3,480 + 4 × 3,480. The timed-out fifth... keep it simple and say it the way you would in the room: the original traffic, plus three extra copies of the failed fifth. About **10,000 extra reads/s**, and if those also time out you have not even counted the tree. Three retries on a timeout is up to **four times** the load of the sick subset, launched because it was sick.
- The primary's ceiling was 15,000. You can push it over the edge with retries alone, while "only" 20% of calls are slow.

The fix is not a smarter backoff on the read path. The fix is: **a timeout is not retried.** The slot is released. The user gets 503 if you have no answer. The dependency gets to finish or fail the attempt you already sent, without its twin.

What you may retry: a connect error where you are sure the request never left the process. That attempt did not create load on the far side. One retry, jittered, budget permitting. If you are not sure the request was unsent, you treat it as a timeout: no retry.

### Creates

The app does not retry a create. The client might. A client retry is a second paste, which the API has always allowed. Automatic app retries would mint that second paste **without the user asking**, or PUT two objects for one 500. Neither is resilience.

If the PUT timed out, return 503, do not insert, enqueue a reap for that id if you minted one and might have stored bytes. If you have not minted yet, there is nothing to reap. Mint **after** you are committed to trying the PUT, and reap on uncertainty. Do not mint, PUT, time out, mint a new id, and retry inside the same user request. That is two orphans and a slower failure.

### Circuit breaker, only as a cap on attempts

After the retry rule, a breaker is the remaining tool, not the first one. If the primary is failing fast (connection refused, not slow), every app can open a breaker and **stop sending** for a short window, planning **5 seconds**, then let one probe through. That protects a primary that is down from 17,400 connect attempts a second.

If the primary is **slow**, a breaker that waits for error rate may never open, because timeouts are your errors and you already chose not to retry them. The in-flight cap from day 19 is what protects you there. Do not expect a breaker to solve latency. Say the difference:

- Down, errors, breaker opens, shed quickly.
- Slow, slots fill, 503s from the cap, breaker optional.

A breaker that opens on the cache will send every read to the primary. That is the day-10 failure mode. Prefer "cache timeout = miss" for a single call, and only open a cache breaker if the cache is failing so hard that the 5 ms timeouts themselves are the load. Then you shed origin reads rather than melting the primary. One sentence. Do not draw a breaker in front of every box.

### Hedging

A hedge is a second request you send before the first finishes, to cut tail latency. It is a deliberate extra load on the healthy path. You refuse it. This pastebin's read budget is 300 ms and the dependency is one primary. A hedge of 17,400 reads doubles the primary. You do not need the tail that badly.

## Diagrams

Mermaid stands in for the whiteboard. SVG figures come later; do not wait on them.

### What may be retried

```mermaid
flowchart TB
  timeout[Timeout or unknown]
  connect[Connect failed before send]
  create[Create PUT or insert]
  timeout -->|no retry| one[One attempt, then 503]
  connect -->|one jittered retry inside budget| twice[At most two]
  create -->|never from the app| client[Client may send a new paste]
```

### How a retry becomes a storm

```mermaid
flowchart LR
  reads[17400 origin reads]
  reads --> slow[20 percent time out]
  slow --> bad[3 retries each]
  bad --> extra[about 10000 extra attempts]
  extra --> primary[Primary past its ceiling]
```

Erase the "3 retries" box and the extra load goes away. That is the design, not a tuning exercise.

## Trade-offs

**Choice.** Explicit budgets. No retry on timeout. At most one retry on a pre-send failure, jittered, inside the budget. No app retry of create. Breaker only for fast failures. No hedging. Workers may retry purge with backoff.

**Alternative.** Retry three times with exponential backoff on every error, including creates and timeouts, because "the network is unreliable."

**What you give up.** Some blips that a retry would have hidden become user-visible 503s. A single lost UDP-shaped packet... you are on TCP. A connect failure still gets one retry. You are giving up the **second and third** try of a call that may already be executing. You accept a slightly worse success rate on a healthy day to avoid a much worse day when the dependency is ill.

**Why the alternative fails.** It multiplies the sick subset, it holds slots longer (backoff inside an HTTP request is a self-inflicted timeout), and on creates it double-writes. Backoff belongs off the request path.

**10× break.** 174,000 cold origin reads, 20% timing out, three retries: on the order of 100,000 extra attempts. The storm scales with the traffic. The rule "do not retry a timeout" scales too: it stays one attempt, and the 10× problem remains the primary's real capacity, not your amplifier. A breaker at 10× must be per process or shared carefully; 22 app processes each probing every 5 seconds is fine, 22 processes each retrying the world is not. The rule matters more as you get bigger, which is the opposite of "we'll add retries when we're serious."

## Talking points

**Say.** "Origin read budget 300 ms. Primary lookup 20 ms, object GET 200 ms, cache 5 ms then it's a miss. I do not retry a timeout, because the work may still be running and a retry doubles it. I retry once, with jitter, only if the request never left. I never retry a create from the app. Three retries on 20% of 17,400 is about ten thousand extra reads a second, which is enough to tip the primary. I won't do that to look resilient."

**Say.** "Workers retry purges. Users don't wait on those. A breaker is for when the primary is refusing connections, not for when it's slow. Slow is the in-flight cap."

**Hand-waving.** "We have retries and timeouts." The matrix above, or it is not a design. Which call, which error, how many.

**Hand-waving.** "Exponential backoff" on the GET path. You are holding a person and a slot. Backoff in the worker. Fail the request.

**Hand-waving.** "Exactly-once create by retrying until the primary says the id exists." You are designing a protocol under pressure. Return the error. Let the client send a new paste. Idempotency keys were a non-goal on day 4 because anonymous clients will not keep them. Do not add them in the retry section.

**If they ask what the user sees.** A timed-out read is a 503, not a 404 and not a truncated paste. A timed-out create is a 503 with no link. They may try again and get a second paste. You have said that since week 1.

**If they ask what you page on.** Timeout rate per dependency, retry rate (which should be nearly zero on the read path), breaker open, create 503s. A rising retry rate is the storm starting. Alert on it before you alert on "high QPS," because the QPS may be yours.

## Kit artifact

One checklist row.

| Row | Lock |
|---|---|
| Retry | Timeouts named per call. No retry on timeout. No automatic retry of a non-idempotent create. Extra attempts have a multiplication you can say. |

## Design log

One line: the retry rule you actually wrote down, and the extra QPS it would have created at a 20% failure rate. If you did not compute it, that computation is the gap.

Next: [Day 21 — Capacity redo with visible assumptions](21-capacity-redo-with-visible-assumptions.md).

---

<!-- day-nav -->
[← Day 19 — Backpressure and load shedding](19-backpressure-and-load-shedding.md) · [Day 21 — Capacity redo with visible assumptions →](21-capacity-redo-with-visible-assumptions.md)

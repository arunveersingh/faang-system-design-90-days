<!-- day-nav -->
[← Day 16 — Queue for work the user does not wait on](16-queue-for-work-the-user-does-not-wait-on.md) · [Day 18 — Consistent hashing when nodes change →](18-consistent-hashing-when-nodes-change.md)

# Day 17 — Rate limits and abuse

**Do now**

1. Set a timer.
2. Attempt the problem. Stop at the attempt line. Do not scroll.
3. Then read.


## Time box

35 minutes.

| Minutes | Do this |
|---|---|
| 0–10 | Attempt. Where the limit sits, what it counts, and what it refuses to count. |
| 10–28 | Read. If the limit is after the PUT, it is a bill you already paid. |
| 28–35 | Say the 429 path and the one abuse case a per-IP limit will not see. |

## Intent

Facing a public write API, leave able to put rate limits and abuse control on the path before the expensive store. The limiter is a fix for a client who can spend your PUT budget. It is not a classifier, and it is not a moral stance about viral reads.

## Problem

> Design a pastebin. People paste text and share a link.

The interviewer says: "Creates are anonymous. What stops one caller from writing a terabyte before lunch, and where do you stop them?"

## Attempt before reading

10 minutes. Do not scroll. A create is a PUT of up to 1 MB plus a row insert. Average ingest is 100 GB/day because you assumed 10 million honest pastes, not because the API enforces that. The bucket is private. The CDN serves hot reads.

Write:

1. The expensive thing you must not do for a rejected create (name the call).
2. A limit with units: per who, how many, burst or not. Tie the global ceiling to the ~350 writes/s peak you already designed for, not to a new fantasy QPS.
3. Whether reads of a shared link are limited the same way. They should not be, or you need a reason.
4. What you will not build: a malware model, an account system, a reputation graph.

---

**Stop. A token bucket in front of the PUT, below.**

---

## Diagrams

### Reject before the expensive call

```mermaid
sequenceDiagram
  participant C as Client
  participant A as App
  participant L as Limiter in cache
  participant O as Bucket
  C->>A: POST Content-Length
  alt over cap or over budget
    A-->>C: 413 or 429
  else allowed
    A->>L: take token
    A->>O: PUT
    Note over O: only now
  end
```

### Who is limited

```mermaid
flowchart TB
  ip[One source address]
  ip --> creates[Creates: count and bytes]
  ip --> originGets[Origin GETs: loose]
  viral[Viral paste id] --> cdn[CDN reads: not limited]
  botnet[Many addresses] --> gap[Per-IP limit does not see this]
```

The bottom box is part of the answer. A limiter you describe as complete, while keyed only on IP, is hand-waving.

## Requirements

Abuse control here means: the storage and the primary cannot be filled by one source faster than the plan you stated, and a rejection does not look like a successful paste or like a missing one.

It does not mean: you detect hostile text. The body is untrusted UTF-8 you already refuse to render. Scanning it for malice is a different product, and it is not this course. You do not add a classifier on the write path "for safety." Safety you can defend in this hour is the cap, the content-type, the unguessable id, and the limit.

There are still no accounts. The tenant you can actually key on is the **source address**, and it is a bad tenant. Carrier-grade NAT puts a stadium behind one address. A botnet uses a million addresses. You say both limits of the idea before you sell it. A per-IP limit cuts the single noisy host and the script on one VM. It does not cut a distributed flood, and it will cut a university if you set it like a home broadband line. That is the trade, not a bug you discover after launch.

Reads are the product. A popular paste is supposed to be fetched many times, ideally from the CDN, without the author's fans presenting a token. You do not put the create budget on GET.

## Design

The check runs on the app, **after** you know the request is a create, **before** you buffer the full body if you can help it, and **definitely before** PUT and before the primary insert.

Order:

1. Read `Content-Length`. If it is missing, or greater than 1,000,000, reject. 413 for too large, 400 for a length you cannot bound. You do not accept a chunked body of unknown size into memory and "check later."
2. Ask the limiter. If it says no: **429**, with `Retry-After`. Nothing is written. No id is minted. No orphan, because no PUT.
3. Only then read the body, validate UTF-8 and the TTL, mint, PUT, insert, 201.

Why before the body is fully read: a 1 MB upload you intend to reject is still 1 MB of bandwidth. It is cheaper than 4.5 TB of junk and a primary full of rows you must sweep.

### Budgets you can say

These are assumptions, as visible as the 3× peak. They are not measurements of abusers.

| Bucket | Key | Rate | Burst | Why this number |
|---|---|---|---|---|
| Create count | source IP | **1 per second** sustained | **10** | A human pastebin session is not 10 creates a second for a minute. A script is. 1/s is 86,400 a day from one IP, which is already a lot of text and still under 1% of daily ingest if the mean holds. |
| Create bytes | source IP | **100 KB/s** sustained | **2 MB** | Stops the 1 MB cap from being used as a disk filler at the count limit. At 1 create/s of 1 MB, the count limit alone still allows 86 GB/day from one IP. The byte bucket is what makes "1 per second" not equal to "1 MB per second." |
| Global creates | whole service | **350 per second** | small | This is the peak you sized. Above it, more creates are not "success the limiter should allow." They are overload. Shedding them is day 19. The limiter is where the 429 happens. |

429 is not 404 and not 500. The client can retry later.

Shared state for the buckets lives in the **cache tier you already have**, not in the memory of one app process. A dead cache that allows the per-IP burst until the global 350/s cap still bounds the cluster.

### Reads

A separate, loose ceiling per IP on origin GETs, not on CDN hits. You cannot see CDN hits per IP in the app, and you should not pretend to.

Planning number: **50 origin GETs per second per IP.** A real reader is on the CDN after the first miss. Fifty is enough for a cold client and useless for scraping 450 million ids, which was already hopeless because the id space is 62^12.

You do **not** rate-limit a paste id's successful CDN reads. That is the viral paste.

### What else sits before the store

- Body cap, already there.
- TTL allow-list, already there.
- UTF-8 check, after you have the bytes, still before PUT. Invalid text does not get an object.
- A minimum size? No. Empty pastes can be 400 if you want; they are not the cost problem.

No account lockout, no captcha theater unless they ask. "A hook on create, before the durable write, same place as the 429" is the whole answer.


## Trade-offs

**Choice.** Per-IP token buckets for create count and create bytes, enforced before PUT, state in the shared cache, global ceiling at the 350/s peak you already planned, loose origin-GET ceiling, no limit on CDN hits of a hot id.

**What you give up.** Some NATed users will share a budget and get 429s they did not earn. Some attackers with many IPs will walk under the per-IP floor and you will only stop them at the global 350/s, which means they can consume the entire honest peak.

**10× break.** The global ceiling becomes ~3,500/s if honest traffic grew, and you must move the number with the plan, not leave 350 in the config and 429 the real users. The per-IP numbers do not automatically 10×. A single abusive IP at 10× the site is still one IP; you do not give them 10× the budget because the site grew.

## Say this in the room

429 before PUT. Per IP, about one create a second with a small burst, and a byte budget so they can't post the 1 MB cap at that rate all day. Global cap at the 350 a second I sized the write path for. State lives in the cache, otherwise four apps are four budgets. I do not rate-limit a viral read at the CDN.

## Kit artifact

One checklist row.

| Row | Lock |
|---|---|
| Admission | Limit sits before the durable write. Key and units are spoken. Viral reads are not the thing you throttle. No content classifier. |

## Design log

One line: where your limit sat relative to the PUT, and one bypass you already know (NAT or many IPs). If the limit was after the write, that placement is the gap.

Next: [Day 18 — Consistent hashing when nodes change](18-consistent-hashing-when-nodes-change.md).

---

<!-- day-nav -->
[← Day 16 — Queue for work the user does not wait on](16-queue-for-work-the-user-does-not-wait-on.md) · [Day 18 — Consistent hashing when nodes change →](18-consistent-hashing-when-nodes-change.md)

<!-- day-nav -->
[← Day 7 — Timed dry run (pastebin)](07-timed-dry-run-pastebin.md) · [Day 9 — Load balancing and draining →](09-load-balancing-and-draining.md)

# Day 8 — The second box

## Time box

35 minutes.

| Minutes | Do this |
|---|---|
| 0–10 | Attempt. One page. The break first, then the box that fixes it. |
| 10–28 | Read. If your second box does not name where the bytes still live, fix that before you add a third box. |
| 28–35 | Close the page. Say which state is not on the app tier, and why a second process does not make the disk durable. |

## Intent

Facing a saturated process, leave able to split a stateless app tier and say where session and file state still sit. The second box is a fix for a process that cannot take the peak, not a new architecture.

## Problem

> Design a pastebin. People paste text and share a link.

The interviewer says: "Your one box meets the NIC math. The process does not. Add the next machine, and tell me what is still single."

## Attempt before reading

10 minutes. Do not scroll. Use the locks you already have: peak about 17,400 reads/s and 350 writes/s, bodies on a local NVMe, Postgres for the row, 201 only after the body is durable and the row commits. No cache yet.

Write:

1. The limit of **one OS process**, as a number you could be wrong about. Not "it won't scale."
2. The second box. What it runs. What it must not store.
3. Where the body bytes sit after the split, and where a session would sit if you had one.
4. One sentence on what this split does **not** fix (disk loss, the 450 million files, metadata durability).

If the second box is a cache, a queue, or a CDN, erase it. Those fix different breaks. This break is the process.

---

**Stop. The second box below is the app tier, not a platform.**

---

## Requirements

Unchanged contract. Anonymous create, capability URL, same 404 for missing and expired, 201 only after durable body and committed row, ids never reused, no accounts, no listing.

The new assumption, same kind as the 5 ms fsync, said out loud so it can be thrown out:

**One paste process, one event loop, saturates at about 8,000 streaming reads a second.** Planning ceiling, not a benchmark. Peak read is 17,400. You are over by about 2×. A second hidden safety factor is not allowed; if the real ceiling is 30,000, you delete this box.

Why a process ceiling exists even though the NIC and the NVMe were fine on day 5: the host can move 174 MB/s, and a single thread copying those bytes onto TLS sockets cannot. Deploys make the same box mandatory. Restarting the only process is an outage for every in-flight create and every read, even when the disk is healthy.

There is still **no session**. Do not invent one so the second box has something to stick.

## Design

You do not clone the whole host. A second full copy with its own NVMe would fork the bytes. Reads would disagree. That is a split-brain pastebin, not a scale-out.

You narrow the original machine and add app machines.

```text
clients
   |
   v
[app host 1] [app host 2] [app host 3]     stateless paste process
        \         |         /
         \        |        /
          v       v       v
              [data host]
                |-- Postgres (the row)
                |-- NVMe /data/{id[0:3]}/{id}
```

Three app processes because 17,400 / 8,000 is 2.2, and you want one process to die or drain without falling back through the ceiling. Two would meet the average and fail the peak the moment one restarts. Say that. Do not draw thirty.

**Stateless means:** any app process can take any request. Kill one and the others still create, read, and delete. A deploy replaces one process at a time. You will define drain on day 9. Today the claim is only that the process is not the sole place a request can land.

**Session state sits nowhere.** The URL is the capability. The delete token lives with the creator, and only its hash sits in the row. There is no login cookie, no server-side session map, and nothing to pin to a process. If a load balancer later offers stickiness, you refuse it. Sticky sessions would put state back on one process, which is the box you just split, and you have no session to stick.

**File state still sits on the one data host.** The app tier does not have the bodies. Each app opens Postgres over the network and reads or writes `/data/...` on that host (a directory that host exports to the app tier, or a tiny file API on the data host — one sentence, not a new product). The path is still derived from the id. The write order does not change: body durable on that NVMe (group commit, same 200/s serial ceiling, so the batching from day 5 stays), then row commit, then 201.

**Database state still sits on that same host.** The second box did not replicate Postgres. You will do that when the break is a dead primary, not a hot event loop. Day 13.

What you refuse while drawing this:

- A cache. The primary is not the break you were given. The process was.
- A queue. The user is still waiting for the link. The app processes do the create inline.
- Putting the NVMe on every app host. That is N copies you do not reconcile.
- Calling the app tier highly available. Three processes hide a process crash. They do not hide the data host. Say the failure domain out loud: **the data host is still one disk and one Postgres.** Day 5's restore problem is exactly where you left it. 450 million files, one NVMe. You added machines in front of it.

Memory on an app process stays small. It holds the in-flight body only until the data host has it. It does not accumulate pastes. If it does, you have accidentally built a second store.

Create is still not idempotent. A client that retries after a killed process can mint a second paste. The second box does not fix that. Do not "solve" it today with a dedupe log you did not require.

### How many, said as arithmetic

Peak reads 17,400. Ceiling 8,000 per process. Minimum at peak is 3 processes so that one can be down and 16,000 of ceiling remains against 17,400. That is still tight on purpose: you are not buying a fleet to feel calm. If they change the ceiling to 20,000, you go back to one process and this page was wrong. Good. The ceiling was an assumption.

Peak writes, 350/s, are not why you have three processes. Group commit on the data host still absorbs them. The app tier spends almost nothing on a write compared with streaming a read. Do not size the tier from the write number.

## Diagrams

Mermaid stands in for the whiteboard. SVG figures come later; do not wait on them.

### Whiteboard

```mermaid
flowchart LR
  clients[Clients] --> app1[App process]
  clients --> app2[App process]
  clients --> app3[App process]
  app1 --> data[Data host]
  app2 --> data
  app3 --> data
  data --> pg[(Postgres rows)]
  data --> disk["NVMe /data/abc/id"]
```

Say while drawing: "Three numbers in the corner. Peak 17,400 reads. Planning ceiling 8,000 per process. Three processes. Bytes and rows stay on one data host."

### Where state sits

```mermaid
flowchart TB
  subgraph app [App tier, any process]
    none[No session and no body]
  end
  subgraph data [Data host, still one failure domain]
    row[Row including token hash]
    file[Body bytes]
  end
  creator[Creator] -->|delete token, once| clientstate[Client keeps the secret]
  app --> data
```

The picture is the lesson. If session or file state appears inside the app box, the tier is not stateless and a deploy will either drop creates or serve disagreement.

## Trade-offs

**Choice.** Three stateless app processes in front of one data host. No sticky sessions. Bodies stay on the one NVMe.

**Alternative.** A bigger single process, or sticky sessions to a copy of the cache you do not have yet, or a second data host with its own disk.

**What you give up.** A network hop to Postgres and to the bytes, on every create and every read. In-process calls became remote calls. A slow data host now stalls three app processes instead of one. You also give up the ability to reboot "the server" as one thing; you have a role split to operate.

**Why you still do it.** 17,400 does not fit an 8,000 ceiling, and a deploy of the only process is a total outage that the NIC math never mentioned. The hop is the cost of being able to restart an app without restarting the disk.

**10× break.** Peak reads go to about 174,000. At 8,000 per process that is about 22 processes, which you can still call an app tier. The data host does not come along for free: one NIC at ~14 Gbit/s still dies, and one Postgres still sees every metadata read. The second box fixed the event loop. It did not fix egress or the primary. Do not "add more app servers" as the answer to a full disk.

## Talking points

**Say.** "One process, I'm calling the ceiling 8,000 streaming reads a second, planning number, not a benchmark. Peak is 17,400, so I want three processes and I want any of them to take any request. There is no session. The delete token is on the client, the hash is in the row. The files are still on one NVMe. Three app boxes do not make that disk durable."

**Say.** "I size from peak reads, not from writes. Writes are 350 a second and they still group-commit on the data host. The 201 order is unchanged: body durable, then row, then the response."

**Hand-waving.** "We'll make the app stateless and put it on Kubernetes." The noun is not the split. Where do the bytes sit, and what happens when that one disk dies?

**Hand-waving.** "Sticky sessions for safety." Safety of what? You have no session. Stickiness turns a dead process into a pile of clients who cannot move.

**Hand-waving.** "The second box is Redis." Redis does not accept the TCP connection or run the expiry check. The process was the saturated thing. A cache fixes a melted primary, which you have not shown yet.

**If they ask what the user sees when one app dies.** In-flight requests on that process fail. The other two keep serving. Clients retry. A retry of a create that might have committed is a second paste. You already sold that in the API. Reads retry cleanly.

**If they ask whether the data host should keep running an app process too.** It can, and then it is a special replica with a local disk the others do not have. You refuse the special case. Apps are identical. The data host serves Postgres and the files. One role each.

## Kit artifact

One checklist row, for when a kit exists. Not the kit.

| Row | Lock |
|---|---|
| Second box | App process only. Session: none. Files and rows: still one data host. The limit written next to the box is the per-process read ceiling, not the word stateless. |

If your attempt cannot say where the file went, you are not done. The kit cannot place it for you.

## Design log

One line: the ceiling you used, and whether your second box accidentally stored bytes or a session. That miss is the gap.

Next: [Day 9 — Load balancing and draining](09-load-balancing-and-draining.md).

---

<!-- day-nav -->
[← Day 7 — Timed dry run (pastebin)](07-timed-dry-run-pastebin.md) · [Day 9 — Load balancing and draining →](09-load-balancing-and-draining.md)

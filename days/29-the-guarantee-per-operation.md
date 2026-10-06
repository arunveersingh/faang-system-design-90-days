<!-- day-nav -->
[← Day 28 — Mock: image upload and thumbnails](28-mock-image-upload-and-thumbnails.md) · [Day 30 — Linearizability and what you refuse →](30-linearizability-and-what-you-refuse.md)

# Day 29 — The guarantee per operation

**Do now**

1. Set a timer.
2. Attempt the problem. Stop at the attempt line. Do not scroll.
3. Then read.

## Time box

35 minutes.

| Minutes | Do this |
|---|---|
| 0–12 | Attempt. One sentence of guarantee for create, for origin GET, for edge GET, and for delete. Not one word for the whole system. |
| 12–28 | Read. If you wrote "strong" or "eventual" once and stopped, replace it with the four sentences. |
| 28–35 | Say the four guarantees and the failure each one accepts. Stop. |

## Intent

Facing the word "consistent," leave able to state the guarantee per operation instead of one label for the whole system. "The pastebin is consistent" is not a design. It is a word you hid behind.

## Problem

> Design a pastebin. People paste text and share a link.

The interviewer says: "What does consistent mean for this system? I want it per call, and I want the failure you are accepting on that call."

## Attempt before reading

12 minutes. Do not scroll. Use the assembled pastebin: 201 after the primary commit, async replica you do not read, metadata cache with a tombstone on delete, CDN max-age at most 60 seconds, creates not idempotent, ids never reused.

Write four lines, one each for POST, origin GET, edge GET, and DELETE. Each line names the guarantee and the lie or loss you still allow. If a line uses only the word "eventual," it is not finished.

---

**Stop. Worked per-call promises below. Leave the four lines you wrote. Do not revise them to match.**

---

## Requirements

The contract from day 26 does not change today. What changes is that you are no longer allowed to answer consistency questions with a single adjective.

A guarantee is something a caller can test:

- **Durability of a 201.** After the response, the row survives a process crash. It does not have to survive the primary disk until you said it does.
- **Visibility.** After a defined response, a defined reader sees a defined result.
- **Uniqueness.** One id, one row. A remint on conflict is the mechanism, not a vibe.
- **Idempotence, or its absence.** A retry either returns the same result or it is a second product action you have named.

"Eventual consistency" fails the test. Eventual *what*, for *which reader*, after *which bound*? If you cannot fill those in, you have not chosen.

You are not picking a CAP side for the whiteboard. CAP is a theorem about partitions, not a setting on your load balancer. If they say "are you CP or AP," you answer with the operation that stalls and the operation that serves a stale byte. That is the whole translation.

## Design

Say these four. The numbers are the ones you already own. Do not invent a tighter product to sound serious.

**The order you say them matters.** Lead with DELETE and edge GET if they opened with "is it consistent," because that is where the lie lives. Lead with POST if they opened with "durable." Leading with origin GET sounds safe and hides the edge. Staff picks the call that makes the next question expensive for the interviewer, not the call that sounds nicest.


### POST, the 201

**Guarantee.** The row is committed on the primary, and the object PUT returned success, before the client sees 201. A crash of the app process after that commit does not lose the row. The id will not be issued again.

**What you do not guarantee.** The async replica does not have the row yet. If the primary disk dies in that window, the 201 becomes a 404 after promotion. That window is the RPO you already accepted on day 13: the last seconds of commits, not zero. A client that times out and POSTs again gets a **second paste**. Create is not idempotent. Do not smuggle idempotency in while defining durability. That is day 37, and only if they add the requirement.

**Failure you accept.** Lost tail on primary death, and a duplicate paste on client retry. Both are named. Neither is "eventual."

### Origin GET

**Guarantee.** A miss reads the primary, not the replica. The row is readable only if it exists and `expires_at` is in the future, checked against the timestamp in the row. After a delete commits, a later origin read does not return the body: the tombstone blocks a late cache fill from putting the row back. Unknown, expired, and deleted are the same 404. A live row whose object is missing is a 500, not a 404.

**What you do not guarantee.** A cache hit filled before the delete can still be served until the tombstone write lands. That is a race of one in-flight read, not a 60-second policy. You do not promise that a GET which *started* before the delete returns 404. Linearizability is a later conversation. Today you promise the origin result **after** the delete has committed and the tombstone is in, for a request that starts then.

**Failure you accept.** Primary down: hits may still serve until their TTL; misses are 503, not 404. You will not promote a lagging replica into this path to "keep GETs consistent." A stale fill would make the lie sticky.

### Edge GET

**Guarantee.** You do not make one. The edge is allowed to serve the body it cached for up to **60 seconds** after a delete or an expiry that the origin already enforces. Purge is best-effort and is not part of the guarantee. You never say the edge is linearizable, strongly consistent, or "invalidated."

**Failure you accept.** A reader with the link sees a deleted paste until max-age. On a hot key at half of peak, half of 17,400 is **8,700 reads/s**. Across a full 60 seconds that is 8,700 × 60 = **522,000** serves of a paste the deleter already killed, if the entry was hot and purge did nothing. You accept that number because the alternative is origin bandwidth you already moved off the NIC. Say the number when they flinch. Do not shrink it in the sentence and leave it in the config.

### DELETE

**Guarantee.** 204 means the row commit succeeded on the primary and the tombstone was written. A later origin GET 404s. The delete is safe to retry: a second DELETE is the same 404 as a missing id, and the token does not get a second life.

**What you do not guarantee.** The object is gone. The edge is dark. Those are the queue from day 16. 204 is not "every copy on earth has forgotten the bytes."

**Failure you accept.** Orphan object until the worker runs, and the 522,000 edge serves above. A lost outbox row leaks the object; it does not resurrect the origin row. If you cannot keep those two failures apart, you will "fix" a leak by making deletes slower, or you will reap a live object to make the leak metric pretty.

**What "after" means in each sentence.** After the 201 bytes leave your process, not after the TCP ACK you did not see. After the 204, for a GET whose start timestamp is later than the commit. After max-age on the edge, measured from when that POP filled, not from when you deleted. Mixing those clocks is how a correct design gets described as broken in the room. Write the clock next to the promise when you rehearse.


### How you test each promise in the room

A guarantee you cannot fail a call against is still an adjective. Walk them through one probe per row.

**POST.** Commit, return 201, kill the app process, GET the id from a fresh process. The row is there. That is durability of the response. Then ask for the replica: it may 404 for up to the lag you already named, about **200 ms** p99 when healthy, and for longer when shipping is sick. At **350 creates/s**, that lag is about **70 pastes** the replica has not applied yet at 200 ms (350 × 0.2). At a 5-second alarm from day 13 it is about **1,750**. Those are the 201s whose RPO you accepted. Say the paste count, not "a little lag."

**Origin GET after DELETE.** Return 204, then GET from a second client that talks only to origin. 404. That is the linearizable half of the origin promise. A fill that started before the tombstone and lands after must lose to `gone`. If they ask how you prove the race, the answer is the compare on the cache write, not a longer TTL.

**Edge GET after DELETE.** Return 204, then GET through the POP that already held the body. 200 until max-age. That is the promise you **refuse**. At half of peak on one key, **8,700 × 60 = 522,000** serves. Write it under the probe so the flinch has a number.

**DELETE retry.** Second DELETE with the same token: 404, no second outbox row if you guard the state, or a harmless second job if the outbox is idempotent at the worker. Either way the origin does not resurrect. The probe is that the second call is not a 500 and not a new body.

What staff sounds like here is refusing one word for four probes, and pricing the edge lie in serves before anyone asks for CAP.

### One table, so you cannot hide

| Call | You promise | You accept |
|---|---|---|
| POST 201 | Primary commit and a durable PUT. Id never reused. | Replica may not have it. Client retry is a second paste. |
| Origin GET | Primary or a cache entry the tombstone still dominates. Expiry from the stored timestamp. | Misses 503 when the primary is down. Not a replica read. |
| Edge GET | Nothing tighter than max-age. | Up to 60 seconds, and up to ~522,000 serves of one hot deleted paste. |
| DELETE 204 | Origin will not serve it after this returns. | Object and edge lag. Retry is a 404, not a second effect. |

If they ask for one word anyway: "It depends on the call. The origin delete is a committed primary write. The public read is a 60-second cache. I will not average those into 'eventual.'"

**Cross-check against day 26's contract.** Creates still return 201 after primary commit and successful PUT. Reads still prefer the edge. Deletes still 204 before purge. Nothing in today's four sentences changes a box. What changes is that you can no longer hide behind "the database is consistent" when they point at the CDN. If your attempt added a new store to "fix" the edge, you failed the day: the work was naming the guarantee you already bought, not buying a different product.

## Diagrams

### Where each call is allowed to be wrong

```mermaid
flowchart LR
  post[POST 201]
  origin[Origin GET]
  edge[Edge GET]
  delete[DELETE 204]
  post --> rpo[Replica may be missing the tail]
  origin --> primary[Primary or tombstone, not the replica]
  edge --> sixty[Up to 60 seconds after delete]
  delete --> worker[Object and purge still in the queue]
```

The picture is four promises. A diagram with one "consistency" box on the side is the adjective you just refused.

Caption: "Four promises, four places to be wrong." Point at each arrow and say the failure, not the happy path. A single "consistency" box on the side of a bigger drawing is the adjective you just refused.

## Failure the user sees, per call

**Creator, primary disk dies three seconds after 201.** The link they shared 404s after promotion if the replica never applied it. At 350 creates/s that is on the order of a thousand pastes in a three-second RPO window (350 × 3 = 1,050). The user who saved sees a working link until failover; the stranger who opens it after promotion sees 404. You already accepted this on day 13. Today you stop calling the 201 "durable everywhere."

**Creator who retries a timed-out POST.** Two pastes, two links, two delete tokens. The first may or may not exist. You do not merge them. Day 37 is when that changes, and only if they add the key.

**Reader on origin while the primary is down.** Cache hits still return the body until their TTL. Misses are 503. A 404 here would teach them the paste is gone when you cannot know. They see an error page, not an empty paste.

**Reader on the edge after the owner deleted.** Body for up to 60 seconds. On the hot paste that is hundreds of thousands of successful 200s of a paste the owner believes is dead. The failure mode is success with the wrong bytes, which is why you must say the number out loud.

**Owner after 204.** Origin is dark. Edge may still show it. Object may still exist until the worker runs. The owner who refreshes through the CDN thinks delete failed. Tell them to hard-refresh origin or wait a minute. The product copy matters as much as the status code.

## Trade-offs

**Choice.** Four guarantees, four accepted failures, no product change.

**Alternative.** One label, usually "strong" for the room and "eventual" in the notes, or the reverse.

**What you give up.** The comfort of a single answer. You also give up the right to claim the edge is correct the moment DELETE returns. You already gave that up on day 15. Today you stop describing it as a temporary bug.

**Name the refusal inside each alternative.** Against one label for the system: you refuse to average a committed primary write with a 60-second cache into "eventual." Against "we're CP": you refuse a letter that does not say which call stalls and which call serves a stale byte. Against shrinking the 522,000 in the sentence while leaving max-age at 60: you refuse a honesty bug. Against promoting a lagging replica to "keep GETs consistent" when the primary is down: you refuse a sticky lie on the origin path. Each refusal names the call it would mis-describe.

**10×.** The promises do not get stricter because traffic grew. The edge number does: 10 × 522,000 = **about 5.2 million** serves of one deleted hot paste inside the same 60 seconds, if the hot fraction holds. If that is no longer acceptable, the fix is a shorter max-age on that class of object, paid in origin reads, not a new consistency product. 5.2 × 10^5 × 10 is 5.2 × 10^6. Say it as five million, not "a lot more."

## Talking points

**Hand-waving.** "We are strongly consistent at the database and eventually consistent at the edge." That sentence is almost this lesson, and it is still missing the bound and the operation. Add the 60 seconds and the 201's RPO or it is a slogan.

**Hand-waving.** "CAP means we chose AP." Which call fails closed, and which call returns a stale body? If you cannot point, you chose a letter.

**Hand-waving.** "Strongly consistent creates, eventually consistent reads." Name the bound on the read and the RPO on the create, or you have restated the slogan from the first hand-wave with two adjectives instead of one.

**If they ask whether a cache hit is part of the origin guarantee.** "Only while the tombstone still dominates that key. A hit filled before the delete, without a tombstone, is the race I close on the delete path. A hit after the tombstone is a 404. The hit is not a second consistency model; it is the primary's order pushed at one key."


**If they ask what "durable" means on the 201.** "The row survives an app crash after commit. It does not survive primary disk loss until the replica has applied it. At 350 creates a second and 200 ms of lag that is about 70 pastes the replica may still be missing; at the 5-second alarm it is about 1,750."

**If they ask how many stale edge reads you are accepting.** "On a hot paste at half of peak, 8,700 a second times 60 seconds is about 522,000. At 10× that is about 5.2 million inside the same window. The fix is a shorter max-age paid in origin reads, not a new consistency product."

**If they ask whether expiry is consistency.** Expiry is a predicate on a timestamp you stored. A lagging replica that has the row still expires it on time, because the timestamp is in the row. Lag does not un-expire. Lag resurrects a delete and hides a create. Those are different bugs. Do not file them under one word.

## Say this in the room

A 201 means the primary committed the row and the object PUT succeeded, and it does not mean the replica has that row yet: at 350 creates a second and 200 ms of lag that is about 70 pastes still missing on the replica, about 1,750 at the 5-second alarm. An origin GET reads the primary or a cache entry the delete tombstone still wins over, and a miss while the primary is down is a 503, not a 404. An edge GET has no tighter promise than the 60-second max-age, which on a hot deleted paste is about 8,700 reads a second times 60, about 522,000 serves I am explicitly accepting. A 204 means the origin will not serve the paste anymore, and it does not mean the object and the edge are already dark. I will not call that mix strong, eventual, or a CAP letter, and I will not promote a lagging replica onto the origin path to make the adjective prettier.


## Kit artifact

One checklist row.

| They say | You answer with |
|---|---|
| "Is it consistent?" | Four lines: 201, origin GET, edge GET, DELETE. Each line has a promise and a bound. |

## Design log

One line: which call you had collapsed into "eventual," and the bound you can now say. That is the gap if the bound is still missing.

Next: [Day 30 — Linearizability and what you refuse](30-linearizability-and-what-you-refuse.md).

---

<!-- day-nav -->
[← Day 28 — Mock: image upload and thumbnails](28-mock-image-upload-and-thumbnails.md) · [Day 30 — Linearizability and what you refuse →](30-linearizability-and-what-you-refuse.md)

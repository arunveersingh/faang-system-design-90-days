# Day 71 — Social graph: X thread draft

Status: draft for Arunveer's approval. Do not post without his OK.
Spoiler check: no full design, no answer to the attempt. Size math, block vs unfollow, and one framing line only.

---

**1/**
"Design a social graph" sounds like a big-data problem.

It isn't. 200M follow edges fit in about 26 GB.

The hard part is the block button 🧵

**2/**
Quick math: one edge is two ids, a type, a state, and a timestamp. Call it ~64 bytes with index overhead.

Store it both ways (who I follow, who follows me) and 200M follows is ~26 GB.

A few machines. You shard for hot keys, not size.

**3/**
Now the block button.

Block is not unfollow. It's two-way, and it has to hold on every surface: feed, profile, DMs, search, comments.

Miss one surface and that's where the leak shows up.

**4/**
And it can't lag.

"They unfollowed me and I saw one old post" is fine.

"I blocked them and still saw their posts for 10 minutes" is a privacy bug.

**5/**
Celebrity check: 50M followers × 8-byte ids = 400 MB.

If your API can return that in one response, it eventually will. Then it times out.

**6/**
In the room, say this early:

"The graph is small. The reads are the cost. And block gets a tighter promise than unfollow."

**7/**
Day 71 of my free 90-day system design book. Try the problem first, then read:
https://reasonfrom.com/faang-system-design-90-days/#/days/71-social-graph

# Day 71 — Social graph: YouTube short script (45–60 s)

Status: draft for Arunveer to record. Spoiler-light: size math and one framing line, no full design.

---

**[0–4 s] Hook, to camera**
"Design a social graph." Sounds huge. It's actually tiny. Here's the part that isn't.

**[4–18 s] The size math**
Say 200 million follows. Each one is two user ids, a type, and a timestamp. About 64 bytes with indexes. Store it both ways, who I follow and who follows me, and that's around 26 gigabytes. A few machines.

**[18–32 s] The real problem**
The hard part is the block button. Block isn't unfollow. It works both ways, and it has to hold everywhere: your feed, their profile, DMs, search, comments. Skip one and that's your leak.

**[32–44 s] The promise**
And it can't lag. Seeing one old post from someone who unfollowed you is fine. Seeing someone you blocked for ten more minutes is a privacy bug.

**[44–53 s] The line to say in the room**
So say this in your interview: "The graph is small, the reads are the cost, and block gets a tighter promise than unfollow."

**[53–59 s] Close**
That's Day 71 of my free 90-day system design book. Link's in the description. Try the problem before you read the answer.

---

On-screen text: `200M follows ≈ 26 GB` · `Block ≠ unfollow` · `Block lag: seconds, not minutes`
Description link: https://reasonfrom.com/faang-system-design-90-days/#/days/71-social-graph

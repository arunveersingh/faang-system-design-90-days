# Day 64 - Metrics query and alerts: YouTube short script (45-60 s)

Status: draft for Arunveer to record. Spoiler-light: "wait before you page" and "one page per problem". No storage layout, no full design.

---

**[0-4 s] Hook, to camera**
Your phone buzzed 50 times last night. Nothing was actually broken.

**[4-16 s] The setup**
Engineers on call get woken up when a website breaks. A computer watches numbers, like how busy each server is, and a rule says: above 90%? Wake someone up.

**[16-30 s] Trap one**
But numbers spike. A server can jump for 40 seconds and calm down on its own. Check once and page instantly, and you wake a human for nothing. Fix: page only if it stays high for 5 minutes straight.

**[30-46 s] Trap two**
Now a bad update breaks 2,000 servers at once. That's 2,000 calls for one mistake. So bundle them: one page per problem per team. In the interview, say the cost too. Waiting 5 minutes means a real outage pages about 6 minutes late.

**[46-58 s] Close**
That's Day 64 of my free 90-day system design book. Link's in the description. Try the problem before you read the answer.

---

On-screen text: `wait 5 minutes before you page` · `one page per problem, not per server`
Description link: https://reasonfrom.com/faang-system-design-90-days/#/days/64-metrics-query-and-alerts

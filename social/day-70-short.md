# Day 70 — Flash sale: YouTube short script (45–60 s)

Status: draft for Arunveer to record. Spoiler-light: load math, one trick, and an open fairness question. No full design.

---

**[0–4 s] Hook, to camera**
Ten thousand sneakers. A million shoppers. All at 10 o'clock. How do you keep the site up?

**[4–16 s] Why the obvious answer fails**
Every purchase subtracts one from the same stock count, kept in one place in the database. That single record handles a few hundred updates a second. A million shoppers buries it.

**[16–30 s] The line problem**
So you put people in a waiting line. But a million phones asking "where am I?" every 5 seconds is 200,000 questions a second.

**[30–42 s] The trick**
Instead, the site publishes one number once a second: "now letting in shopper number X." Your phone knows your number and does the subtraction. One answer serves a million screens.

**[42–52 s] The question to answer in the interview**
Then say who goes first. First come, first served? A random draw? Loyal customers? Each has a cost. Pick one and say the cost.

**[52–59 s] Close**
That's Day 70 of my free 90-day system design book. Link's in the description. Try the problem before you read the answer.

---

On-screen text: `10k units vs 1M shoppers` · `one stock record: a few hundred updates/s` · `1 number, 1M screens`
Description link: https://reasonfrom.com/faang-system-design-90-days/#/days/70-flash-sale

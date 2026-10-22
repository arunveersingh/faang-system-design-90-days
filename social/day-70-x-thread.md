# Day 70 — Flash sale: X thread draft

Status: draft for Arunveer's approval. Do not post without his OK.
Spoiler check: load math, the "one number for a million screens" trick, and an open fairness question. No full design, no answer to the attempt.

---

**1/**
A flash sale: 10,000 sneakers go on sale at 10:00.

A million people show up at 10:00.

It's a classic system design interview problem, and the obvious answer crashes the site. 🧵

**2/**
The obvious answer: let every shopper try to buy right away.

But every purchase has to subtract one from the same stock count, stored in one place in the database.

A single record like that can only take a few hundred updates a second.

**3/**
A million shoppers. A few hundred updates a second.

So the database gets buried, most people see error pages, and a few lucky requests slip through.

That isn't a sale. It's a lottery nobody agreed to.

**4/**
Now say you put shoppers in a waiting line. A million people still want to know "where am I?"

If each phone asks the server every 5 seconds, that's 200,000 questions a second.

**5/**
The fix: publish one number, "we're now letting in shopper #X," once a second.

Each phone already knows its own number and does the subtraction itself.

A million screens, one tiny shared answer.

**6/**
Then comes the part interviewers really care about: who goes first?

First come, first served? A random draw? Loyal customers?

Each one has a cost. Pick one and say the cost out loud in the interview.

**7/**
Day 70 of my free 90-day system design book. Try the problem first, then read:
https://reasonfrom.com/faang-system-design-90-days/#/days/70-flash-sale

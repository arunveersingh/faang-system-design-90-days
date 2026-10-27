# Day 66 — Payment reconciliation: X thread draft

Status: draft for Arunveer's approval. Do not post without his OK.
Spoiler check: the "never edit old records" rule and the timezone math. No full design, no answer to the attempt.

---

**1/**
A customer pays you ₹499. The card company says the money moved. Your own records say the payment never happened.

Finding and fixing gaps like that is called "payment reconciliation", and it's a real system design interview problem. 🧵

**2/**
Two sides keep score.

The payment provider (a company like Stripe that actually moves card money) sends you a daily report of every charge.

Your app keeps its own record of every rupee, usually called a ledger. Reconciliation means matching the two, line by line.

**3/**
The tempting fix when they disagree: open your record and change the number so it matches.

Don't. In the interview, say why: once you overwrite a record, nobody can ever answer "what did we believe on 3 March?" Auditors and support both need that answer.

**4/**
The honest fix: leave the old record alone and add a new one that says "correction: +₹50, because the provider's report says so, line 4,812."

History stays readable. Anyone can see what was wrong and who fixed it.

**5/**
Now a sneaky one. Your app counts days on India time. The provider counts days on UTC (London winter time).

India is 5.5 hours ahead, so 5.5 ÷ 24 ≈ 23% of payments get a different date on each side. At a big company that's about 20 million payments a day that look "missing" but aren't.

**6/**
So you don't match on date and amount. Two people can pay ₹499 in the same minute.

You match on the ID the provider gives every charge, and you allow a day of slack on dates.

**7/**
Day 66 of my free 90-day system design book has the full problem, the numbers, and what to do with money nobody can explain.

Try the problem before you read the answer:
https://reasonfrom.com/faang-system-design-90-days/#/days/66-payment-reconciliation

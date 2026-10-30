# Day 66 - Payment reconciliation: X thread draft

Status: draft for Arunveer's approval. Do not post without his OK.
Spoiler check: the "never edit old records" rule and the timezone mechanism. No full design, no answer to the attempt.

---

1/ A customer pays you Rs 499. The card company says the money moved. Your own records say the payment never happened.

Finding and fixing gaps like that is called "payment reconciliation", and it's a real system design interview problem.

2/ Two sides keep score.

The payment provider (a company like Stripe that actually moves card money) sends you a daily report of every charge.

Your app keeps its own record of every rupee, usually called a ledger. Reconciliation means matching the two, line by line.

3/ The tempting fix when they disagree: open your record and change the number so it matches.

Don't. In the interview, say why: once you overwrite a record, nobody can ever answer "what did we believe on 3 March?" Auditors and support both need that answer.

4/ The honest fix: leave the old record alone and add a new one that says "correction: +Rs 50, because the provider's report says so, line 4,812."

History stays readable. Anyone can see what was wrong and who fixed it.

5/ Now a sneaky one. Your app counts days on India time (IST). The provider counts days on UTC.

IST is 5.5 hours ahead, so payments made between midnight and 5:30 AM IST land on the previous UTC day. Same payment, two dates. Match on date and it looks missing when it isn't.

6/ So don't match on date and amount either. Two people can pay Rs 499 in the same minute.

Match on the ID the provider gives every charge, and allow a day of slack on dates.

7/ Check your reconciliation job today: does it match on the provider's charge ID? Does it ever overwrite a ledger row?

The full problem is in Day 66 of my free 90-day system design book. Try it first:
https://reasonfrom.com/faang-system-design-90-days/#/days/66-payment-reconciliation

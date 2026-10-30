# Day 65 - Payments: X thread draft

Status: draft for Arunveer's approval. Do not post without his OK.
Spoiler check: why "retry with the same request ID" matters and that "unknown" is an honest answer. No ledger layout, no state machine, no answer to the attempt.

---

1/ You tap "Pay ₹2,499" on a shopping app. The spinner spins. Then: "Something went wrong."

So you tap Pay again.

Later your bank app shows ₹4,998 on hold. You got charged twice.

How a payment system stops that is a classic system design interview problem.

2/ Quick setup. The shop's servers don't move money themselves. They ask a payment company (like Stripe or Razorpay) to charge your card.

Sometimes that request just takes too long, and the shop's server stops waiting. That's called a timeout.

3/ Here's the trap. A timeout doesn't mean the charge failed.

The payment company may have charged you fine. The "yes" just never made it back in time.

So the shop honestly doesn't know. It's not success, and it's not failure. It's "unknown."

4/ The fix: every payment gets a unique ID number, created once, before the first try.

When the shop retries, it sends the same ID. The payment company sees "I already did this one" and returns the first answer instead of charging again.

5/ The mistake that double-charges you: retrying with a brand new ID. To the payment company, that looks like a fresh purchase.

At 1,000 payments a second, if just 0.1% time out, that's 3,600 double charges an hour.

6/ And the app should say "Payment processing", not "Payment failed", while it checks. "Failed" is what made you tap Pay again in the first place.

In the interview, say that out loud: unknown is a real answer, and you keep the same ID until you know.

7/ Question to ask about any button that moves money: what happens if someone presses it twice?

The full problem is Day 65 of my free 90-day system design book. Try it first:
https://reasonfrom.com/faang-system-design-90-days/#/days/65-payments

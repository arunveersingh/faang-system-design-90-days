# Day 65 - Payments: YouTube short script (45-60 s)

Status: draft for Arunveer to record. Spoiler-light: "same request ID on retry" and "unknown is not failed". No ledger, no state machine, no full design.

---

**[0-4 s] Hook, to camera**
You paid once. Your bank shows two charges. Here's how that happens.

**[4-16 s] The setup**
When you tap Pay, the shop asks a payment company to charge your card. Sometimes that request is slow, and the shop's server stops waiting. That's a timeout.

**[16-30 s] The trap**
A timeout doesn't mean it failed. The charge may have gone through, and the "yes" just got lost. If the shop retries as a brand new payment, you get charged twice.

**[30-46 s] The fix**
Give every payment one ID, created before the first try. Every retry sends the same ID, so the payment company says "already done" and returns the first answer. And the app shows "Processing", not "Failed". In the interview, say it plainly: unknown is a real answer.

**[46-58 s] Close**
That's Day 65 of my free 90-day system design book. Link's in the description. Try the problem before you read the answer.

---

On-screen text: `timeout is not failure` · `same ID on every retry`
Description link: https://reasonfrom.com/faang-system-design-90-days/#/days/65-payments

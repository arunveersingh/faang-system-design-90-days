# Day 66 — Payment reconciliation: YouTube short script (45–60 s)

Status: draft for Arunveer to record. Spoiler-light: the "never edit old records" rule and the timezone math. No full design.

---

**[0–4 s] Hook, to camera**
The card company says you got paid. Your app says you didn't. Who's right?

**[4–18 s] The setup**
Every day, the payment provider, a company like Stripe that actually moves the card money, sends you a report of every charge. Your app keeps its own record of every rupee. Checking those two against each other is called reconciliation. Big companies run it every single day.

**[18–32 s] The trap**
When they disagree, the tempting fix is to edit your old record so it matches. In the interview, say no. Leave the old record alone and add a new one: "correction, plus fifty rupees, here's the proof." Now anyone can see what went wrong and when.

**[32–48 s] One number**
Here's a sneaky one. Your app counts days on India time. The provider counts on UTC. India is 5.5 hours ahead, so a payment made between midnight and 5:30 in the morning lands on the previous day for the provider. Same payment, two dates. Match on date and it looks missing when it isn't. So match on the provider's payment ID, and allow a day of slack on dates.

**[48–58 s] Close**
That's Day 66 of my free 90-day system design book. Link's in the description. Try the problem before you read the answer.

---

On-screen text: `never edit old money records` · `add a correction instead` · `midnight to 5:30 AM IST = previous UTC day`
Description link: https://reasonfrom.com/faang-system-design-90-days/#/days/66-payment-reconciliation

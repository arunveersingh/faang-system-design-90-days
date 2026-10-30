# Day 64 - Metrics query and alerts: X thread draft

Status: draft for Arunveer's approval. Do not post without his OK.
Spoiler check: the "wait before you page" idea and grouping pages. No storage layout, no full design, no answer to the attempt.

---

1/ An engineer is on call. That means their phone wakes them up when a company's website breaks.

Now imagine it buzzes 50 times a night for things that fixed themselves in 40 seconds.

Designing alerts that don't do that is a real system design interview problem.

2/ Quick setup. Companies record numbers about their servers every few seconds, like "how busy is this machine's processor."

An alert is a rule a computer checks over and over: "if this number goes above 90%, wake someone up."

3/ The naive rule checks once and pages instantly.

But numbers jump around. A server can spike for 40 seconds and calm down on its own. The naive rule wakes a human for that.

4/ The fix: make the rule wait. "Page only if it stays above 90% for 5 minutes straight."

A short spike never lasts that long, so it never pages. A real problem does, so it still gets through.

5/ Second trap. A bad software update hits 2,000 servers at once. Each server breaks the rule.

That's 2,000 phone calls for one mistake.

6/ So the system bundles them: one page per problem per team, not one per server.

In the interview, say both ideas out loud, plus the cost: waiting 5 minutes means a real outage also pages about 6 minutes late. Then say why that trade is worth it.

7/ Question to ask about any alert you build: would a human be glad they got woken up for this?

The full problem is Day 64 of my free 90-day system design book. Try it first:
https://reasonfrom.com/faang-system-design-90-days/#/days/64-metrics-query-and-alerts

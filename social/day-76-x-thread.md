# Day 76 — Audit log: X thread draft

Status: draft for Arunveer's approval. Do not post without his OK.
Spoiler check: why the obvious answer fails, the "say who you're defending against" idea, and one size estimate. No full design, no answer to the attempt.

---

**1/**
"Who deleted this customer's account?"

Someday a security team or support agent will ask your system that question.

Designing the record that answers it, called an audit log, is a common system design interview problem. 🧵

**2/**
An audit log is a list of who did what, to which thing, and when.

Sounds easy: every time someone changes something, also save a line saying so.

**3/**
Here's the catch. Say the app first deletes the account, then writes the "who did it" line.

If the server crashes between those two steps, the account is gone and no record says who did it.

That missing line is exactly what auditors go looking for.

**4/**
Then the interviewer asks: "Can anyone change this log?"

Most people answer "it's immutable" (meaning nobody can edit it).

But if it sits in a normal database, the person who runs that database can edit it.

**5/**
The stronger answer starts with a question: who are you defending against?

A buggy app? An employee with database access? Someone who controls your whole cloud account?

Each one needs a different protection. Name the person first, then the protection.

**6/**
And size it. 5,000 actions a second at 500 bytes each is about 216 GB a day.

Keep it for 7 years and that adds up to hundreds of terabytes. Storage cost becomes part of the design.

**7/**
Day 76 of my free 90-day system design book. Try the problem first, then read:
https://reasonfrom.com/faang-system-design-90-days/#/days/76-audit-log

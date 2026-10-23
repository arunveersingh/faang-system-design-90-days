# Day 76 — Audit log: YouTube short script (45–60 s)

Status: draft for Arunveer to record. Spoiler-light: why the obvious answer fails, the "who are you defending against" question, and one size estimate. No full design.

---

**[0–4 s] Hook, to camera**
"Who deleted this account?" Your system has to answer that. Can it?

**[4–18 s] What an audit log is, and the trap**
An audit log is a record of who did what, to which thing, and when. The easy version: delete the account, then write the record. But if the server crashes in between, the account is gone and nobody knows who did it.

**[18–32 s] The "can anyone change it" question**
In the interview, they'll ask if anyone can change this log. Don't just say "nobody." If it's in a normal database, whoever runs that database can edit it.

**[32–46 s] The better answer**
Start by asking who you're protecting against. A buggy app? An employee with database access? Someone who controls your whole cloud account? Each needs a different protection. Name the person, then the protection.

**[46–53 s] One number**
Five thousand actions a second is about 216 gigabytes a day. Keep it seven years and storage is part of the design.

**[53–59 s] Close**
That's Day 76 of my free 90-day system design book. Link's in the description. Try the problem before you read the answer.

---

On-screen text: `who did what, to which thing, when` · `who are you defending against?` · `5,000/s ≈ 216 GB/day`
Description link: https://reasonfrom.com/faang-system-design-90-days/#/days/76-audit-log

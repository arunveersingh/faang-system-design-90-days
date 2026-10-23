# Day 73 — Web crawler: YouTube short script (45–60 s)

Status: draft for Arunveer to record. Spoiler-light: numbers and one framing line, no full design.

---

**[0–4 s] Hook, to camera**
"Design a web crawler." Most people think it's a loop around HTTP GET. Here's why it's not.

**[4–15 s] The big number**
Say you want 100,000 pages a second. At 50 KB a page, that's 5 gigabytes a second. Over 400 terabytes a day. Big, but you can buy it.

**[15–30 s] The real limit**
Here's the catch. A polite crawler hits each website about once a second. So 100,000 fetches a second means 100,000 different websites ready at the same moment. Your speed depends on how many hosts are ready, not how many servers you have.

**[30–42 s] The surprise**
And one site with 10 million pages? At one request a second, that's about 116 days for a single pass. You won't finish. You choose which pages are worth it.

**[42–52 s] The line to say in the interview**
So say this in your interview: "Politeness is capacity. I size ready hosts before I size fetchers."

**[52–58 s] Close**
That's Day 73 of my free 90-day system design book. Link's in the description. Try the problem before you read the answer.

---

On-screen text: `100k/s ÷ 1 req/s per host = 100k ready hosts` · `10M pages ÷ 1/s ≈ 116 days`
Description link: https://reasonfrom.com/faang-system-design-90-days/#/days/73-web-crawler

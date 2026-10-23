# Day 73 — Web crawler: X thread draft

Status: draft for Arunveer's approval. Do not post without his OK.
Spoiler check: no frontier design, no answer to the attempt. Numbers and robots.txt facts only.

---

**1/**
"Design a web crawler" sounds like a for-loop around HTTP GET.

It isn't. The hard part is being polite at scale.

Here's the math most candidates never do 🧵

**2/**
Say you fetch 100k pages/s at 50 KB each.

That's 5 GB/s. About 432 TB a day before compression. 8.64B fetches a day.

Big, but you can buy that. The real limit is somewhere else.

**3/**
Good crawlers hit one host about once a second.

So 100k fetches/s needs 100k different hosts ready to fetch at every instant.

Your throughput is capped by ready hosts, not by how many machines you own.

**4/**
Now take a site with 10M pages.

At 1 req/s, one full pass takes 10M seconds. That's about 116 days.

You will never crawl it all. You pick which pages earn the budget.

**5/**
robots.txt has an actual standard now: RFC 9309 (2022).

• 404 on robots.txt → no restrictions
• 5xx → assume you may crawl nothing
• Don't cache it longer than 24 h
• Crawl-delay isn't in the RFC. Google ignores it.

**6/**
In the interview, say this early:

"Politeness is capacity. I size ready hosts before I size fetchers."

It changes every design choice after it.

**7/**
Day 73 of my free 90-day system design book. Try the problem first, then read:
https://reasonfrom.com/faang-system-design-90-days/#/days/73-web-crawler

# Day 74 — Order book: YouTube short script (45–60 s)

Status: draft for Arunveer to record. Spoiler-light: load math and one framing line, no full design.

---

**[0–4 s] Hook, to camera**
"Design an order book." Most people start adding threads and locks. Here's why that's the wrong move.

**[4–16 s] The big number**
A busy stock might get 50,000 order messages a second. Sounds like a lot. But LMAX, a trading venue in London, reported around 6 million orders a second on a single thread.

**[16–30 s] The real risk**
So one core already covers your busiest symbol about 100 times over. Add a second matcher and you don't get speed. You get two matchers filling the same order. That's a money bug, not a slow page.

**[30–44 s] The promise**
The hard part is what you tell the trader. When you say "ACK," it should mean the order is recorded and safe. Not that it's filled. Fills come later as separate events.

**[44–53 s] The line to say in the room**
So say this in your interview: "One core covers the load, so I spend my complexity on the ack, the fence, and the rebuild."

**[53–59 s] Close**
That's Day 74 of my free 90-day system design book. Link's in the description. Try the problem before you read the answer.

---

On-screen text: `50k msgs/s per hot symbol` · `~6M orders/s on one thread (LMAX, 2011)` · `ACK = durable, not filled`
Description link: https://reasonfrom.com/faang-system-design-90-days/#/days/74-order-book

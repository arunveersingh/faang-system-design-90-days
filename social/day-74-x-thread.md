# Day 74 — Order book: X thread draft

Status: draft for Arunveer's approval. Do not post without his OK.
Spoiler check: no full design, no answer to the attempt. Load math, what ACK means, and one framing line only.

---

**1/**
"Design an order book" makes most candidates reach for threads, locks, and sharding.

The best answer uses one core per symbol.

Here's why 🧵

**2/**
A hot symbol might see 50k order messages a second.

LMAX reported about 6M orders a second on a single thread back in 2011.

One core already covers your hottest symbol about 100 times over.

**3/**
So parallel matching on one symbol buys you nothing but risk.

Two matchers can both fill the same resting order. That's not a slow page. That's a money bug.

**4/**
The hard part is the promise you make to the trader.

When you send ACK, what does it mean?

It should mean "your order is sequenced and durable." Not "filled." Fills are separate events.

**5/**
Two small rules that interviewers love:

• Store prices as integer ticks, never floats
• Time priority comes from your sequence number, never the client's clock

Clocks drift. Sequence doesn't.

**6/**
In the room, say this early:

"One core covers the load, so I spend my complexity on the ack, the fence, and the rebuild."

**7/**
Day 74 of my free 90-day system design book. Try the problem first, then read:
https://reasonfrom.com/faang-system-design-90-days/#/days/74-order-book

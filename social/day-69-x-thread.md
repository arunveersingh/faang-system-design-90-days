# Day 69 - Cart and checkout: X thread draft

Status: draft for Arunveer's approval. Do not post without his OK.
Spoiler check: the double-tap trick and the "unknown isn't failure" rule. No step order, no full design, no answer to the attempt.

---

1/ You tap "Pay" on a shopping app. Nothing happens, so you tap again.

Did you just pay twice?

Building a checkout that says "no" every time is a real system design interview problem.

2/ Here's why it's hard. Your phone sends the order to the shop's servers twice. To the server, those look like two separate shoppers who want the same thing.

If it handles both, you get charged twice.

3/ The fix starts on your phone, not the server.

When you open the "review your order" screen, the app makes up one random ticket number for this purchase. Every tap on Pay sends that same number.

4/ The server keeps a list of ticket numbers it has seen. Second tap, same number? It doesn't start a new order. It just sends back the order it already made.

One ticket, one order, however many taps.

5/ Now a sneakier case. You pay by UPI and the approval takes 4 minutes. The server asked the bank and got no answer yet.

Tempting move: call it a failure and put your item back on sale.

6/ Don't. In the interview, say it out loud: "unknown is not the same as failed."

If the payment goes through later and the item was already sold to someone else, you've taken money for something you can't ship.

7/ Two questions to ask about any checkout you build: what stops a double tap from charging twice, and what happens while the bank hasn't answered yet?

The full problem is Day 69 of my free 90-day system design book. Try it first:
https://reasonfrom.com/faang-system-design-90-days/#/days/69-cart-and-checkout

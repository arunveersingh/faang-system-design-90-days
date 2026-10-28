# Day 69 - Cart and checkout: YouTube short script (45-60 s)

Status: draft for Arunveer to record. Spoiler-light: the double-tap trick and "unknown isn't failure". No step order, no full design.

---

**[0-4 s] Hook, to camera**
You tapped Pay twice. Did you just pay twice?

**[4-18 s] The setup**
When you tap Pay, your phone sends the order to the shop's servers. Tap twice and it sends two. To the server, that looks like two shoppers buying the same thing. Handle both and you get charged twice.

**[18-32 s] The fix**
So when you open the review screen, the app makes up one random ticket number for this purchase, and every tap sends that same number. The server remembers numbers it's seen. Same number again? It hands back the order it already made. One ticket, one order.

**[32-48 s] The sneaky part**
Now say you pay by UPI and the approval takes four minutes. The server has no answer yet. Tempting move: call it failed and put your item back on sale. In the interview, say no. Unknown is not failed. If the money lands later, you've charged someone for an item you already sold.

**[48-58 s] Close**
That's Day 69 of my free 90-day system design book. Link's in the description. Try the problem before you read the answer.

---

On-screen text: `same ticket number = same order` · `unknown is not failed`
Description link: https://reasonfrom.com/faang-system-design-90-days/#/days/69-cart-and-checkout

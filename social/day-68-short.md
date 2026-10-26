# Day 68 — File sync: YouTube short script (45–60 s)

Status: draft for Arunveer to record. Spoiler-light: the "clocks lie" trap and the big-file math. No full design.

---

**[0–4 s] Hook, to camera**
Your laptop just overwrote your newest file with an old one. Here's why.

**[4–18 s] The obvious answer, and the trap**
Apps like Dropbox keep your files the same on every device. The obvious way to pick a winner: newest save wins. But "newest" comes from each device's own clock, and those clocks are often wrong. A laptop that thinks it's last week can overwrite your real latest work.

**[18–32 s] The better answer**
In the interview, say this: the device clock never picks the winner. The server numbers every version in order. If your device edited version 5 and the server's already on 6, your device syncs first. The clock just decides what time to show on screen.

**[32–48 s] One number**
Now add 1 MB to the start of a 4 GB file. Sending the whole file again means 4 GB over home Wi-Fi. Splitting the file into pieces helps, but only if adding bytes doesn't shift every piece. Done right, you send a few megabytes, not four gigabytes.

**[48–58 s] Close**
That's Day 68 of my free 90-day system design book. Link's in the description. Try the problem before you read the answer.

---

On-screen text: `device clocks lie` · `server numbers every version` · `4 GB file, 1 MB edit ≈ a few MB sent`
Description link: https://reasonfrom.com/faang-system-design-90-days/#/days/68-file-sync

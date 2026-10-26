# Day 68 — File sync: X thread draft

Status: draft for Arunveer's approval. Do not post without his OK.
Spoiler check: the "clocks lie" trap and the big-file math. No full design, no answer to the attempt.

---

**1/**
You edit a file on your laptop. Later your phone shows the old version, or worse, the old version overwrites your work.

Designing the system that keeps files the same on every device ("file sync", like Dropbox or Google Drive) is a classic system design interview problem. 🧵

**2/**
The obvious way to decide which version wins: look at when each one was last saved. Newest save wins.

Every file carries that "last saved" time. It comes from the device's own clock.

**3/**
Here's the trap. Device clocks are often wrong.

A laptop that's been asleep, or one with a dead clock battery, can think it's last Tuesday. Its "newest" save is actually older, and it quietly overwrites the real latest version.

**4/**
In the interview, say it plainly: the device's clock never decides who wins.

The server hands out version numbers in order: 5, 6, 7. A device saying "I edited version 5" when the server is already on 6 has to sync first. Clocks only decide what time to show on screen.

**5/**
Second trap: big files.

Someone adds 1 MB to the start of a 4 GB video file. Upload the whole file again and they wait for 4 GB over home Wi-Fi.

**6/**
Good sync systems split files into small pieces and only send the pieces that changed.

But if you cut pieces at fixed spots, adding bytes at the start shifts every piece. Every one looks new, and you upload 4 GB anyway. How you cut matters as much as cutting at all.

**7/**
Day 68 of my free 90-day system design book has the full problem, the math, and how to handle two devices editing the same file offline.

Try the problem before you read the answer:
https://reasonfrom.com/faang-system-design-90-days/#/days/68-file-sync

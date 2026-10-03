# FAANG System Design, 90 Days

A self-paced interview course for senior and staff loops. One day, one sitting, text and diagrams. Arunveer Singh.

## The promise

You learn to run a 35–45 minute design loop out loud. Requirements come before boxes. Every number has an assumption you can recompute. The diagram is one you can redraw from memory. The trade-off names what you gave up, and what breaks at 10×. When a dependency dies, you say what the user sees.

Phase 1 installs that hour on one system: a pastebin. You do not leave the week with a catalog of caches, queues, and consensus. You leave able to take a one-line prompt and finish a design a senior interviewer can push on.

There is no certificate and no cohort. The design log is the artifact.

## Who it is for

Engineers with 10 or more years of experience, preparing for senior and staff loops at Google, Meta, Amazon, Apple, Netflix, Microsoft, and peers. The altitude is someone who has already shipped, and now has to make the reasoning visible on a whiteboard.

Not a new-grad course. Not an AI or ML course. The 90 days are distributed systems, scale, and reliability only.

## How to use it

Work one numbered day at a time, in order. Budget **30–40 minutes**. Stop at 40 even if you are mid-sentence. The cap is the practice.

On a lesson day:

1. Read the time box and the problem only.
2. Stop at **Attempt before reading**. Work on paper. Do not scroll.
3. Then read requirements, design, diagrams, trade-offs, and talking points.
4. Close the page and redraw the picture.
5. Answer the design-log prompt if you want to keep the day.

On a mock day: a timer, a blank page, and no notes from that week. Score yourself with the rubric. Write the log entry from *your* attempt. Only then read the reference under the attempt barrier.

Diagrams are mermaid or ASCII you could put on a whiteboard. If a figure needs a legend, it does not belong in the hour.

Days 1–6 teach the method. The pastebin is the worked example, not a second product. Day 7 is that same pastebin, closed book, because the week's lessons were the method steps. From day 28 on, a mock prompt is never that week's lesson topic. The full rule and the day list are in [CURRICULUM.md](CURRICULUM.md).

Pages under `prompts/` are the live bank. If a prompt file changes, it wins over a title in the curriculum. Pull before a mock. Do not keep a private copy as the source of truth.

## Phase 1 status

| Days | Status |
|---|---|
| 1–7 | Written. Start at [Day 1](days/01-what-the-interview-is-grading.md). |
| 8–90 | Assigned in the curriculum. Not written yet. |

Phase 1 is the spine. Later phases add a box only when this pastebin misses a number or a fault you already stated. Do not skip ahead to "look distributed."

The course brief is [PROPOSAL.md](PROPOSAL.md). Locked choices (audience, length, no cohort, kit sold separately, no certificate) live there and in this file. Do not reopen them inside a lesson.

## Repo layout

```
days/            one file per day; Phase 1 is days 01–07
prompts/         closed-book prompt bank (pastebin is the Week 1 card)
design-log/      template for your entries; keep real notes out of a public push if you want them private
stencils/        the six diagram types, blank
appendices/      open; company-quirk and paper pointers, not numbered days
CURRICULUM.md    the 90-day list
PROPOSAL.md      the brief this repo is built from
LICENSE          MIT
```

## The kit is sold separately

The interview kit (timer script, requirements checklist, estimation sheet, stencils, trade-off card, follow-up bank, grader prompts, extra prompt cards) is a **separate product**. It is not in this repo, and it is not required to start.

Each Phase 1 lesson leaves a single hookup note: one checklist row or one stencil callout, so the kit can attach later to the same places you already practice. Days 1–84 do not need it. When day 85 exists, it shows the attachment. Day 90 can be run without the kit, on the fallback prompt named in the curriculum.

Do not block the other days waiting for the kit.

## License

[MIT](LICENSE). Copyright (c) 2026 Arunveer Singh.

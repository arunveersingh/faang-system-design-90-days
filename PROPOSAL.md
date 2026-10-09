# 90-Day System Design Interview Course — Proposal

Course brief for this repo. The lesson pages follow it. Where this file and a lesson disagree, the lesson is the Phase 1 text students run; this file is why the course is shaped this way.

## Decisions locked for the repo

These close section J. Do not reopen them inside a lesson.

1. **Audience.** Engineers with 10+ years of experience, senior and staff interviews. Not a new-grad course.
2. **Daily length.** 30–40 minutes, self-paced. Stop at 40 even mid-sentence.
3. **Papers.** Optional appendix only (Dynamo, Spanner, Raft). Not a seminar and not a numbered day.
4. **Company color.** One method for every interview. A thin quirk appendix biases the hour. It does not add days.
5. **Platform.** This GitHub repo. Text and diagrams.
6. **Cohort.** None. Self-paced stands alone. No certificate.
7. **Packaging.** The course is this repo. The interview kit is a **separate product, sold later**. Lessons leave one hookup note each. They do not dump the kit.
8. **Video.** Not in v1. If it ever exists, it does not hide the design.
9. **Problem bank.** Arunveer and team keep `prompts/` (problem bank) current. The problem file wins over a curriculum title.
10. **Artifact.** The design log. Nothing else.

No AI or ML content anywhere in the 90 days.

---


For Arunveer Singh (reasonfrom.com). Engineers preparing for design interviews at Google, Meta, Amazon, Apple, Netflix, Microsoft, and peers. Distributed systems, scale, and reliability only. Not an AI/ML course.

## A. Positioning and promise

One topic a day for about 90 days. The day starts from a problem an interviewer would give. Requirements and constraints come before any diagram; the design and its trade-offs come second. A cache, shard, queue, replica, or consistency rule appears only when an earlier design misses a number or fails a stated fault.

A graduate can run a 35–45 minute interview out loud, then keep drilling new problems with an LLM, the kit, and a fixed rubric.

## B. Learning outcomes

- Run the hour in order: requirements, estimates, API, design, deep dive.
- Turn a vague problem into scope, numbers, and non-goals, and estimate QPS, storage, and bandwidth with assumptions visible.
- Draw the whiteboard, separate read and write paths, and a keyed data model.
- Defend storage, cache, queue, and partition choices, including what was given up.
- Treat hot keys, stampedes, retries, idempotency, lag, partial failure, and a lost region as user-visible behavior.
- Score a mock and pick the next problem from the miss.

## C. Structure (90 days)

Nine timed mocks (H) close phases; they are not all left to the last week.

| Phase | Days | Weeks | What it builds |
|---|---|---|---|
| 1. Hearing the problem | 1–7 | 1 | Scoring, requirements, estimates, API, data model, single-box failure. |
| 2. First distributed design | 8–28 | 2–4 | LB, cache, SQL vs NoSQL, replication, queues, object storage, CDN, rate limits, consistent hashing. Each as a fix. |
| 3. Data and consistency | 29–49 | 5–7 | Partitioning, hot keys, invalidation, lag, quorums, idempotency. At-least-once plus dedupe. |
| 4. Product-shaped systems | 50–77 | 8–11 | One problem a day: shortener, feed, chat, notifications, typeahead, video, location, tickets, metrics, payments, collab edit, variants. |
| 5. Failure and senior signal | 78–84 | 12 | SLOs, backpressure, degradation, multi-region, observability, migration, cost. |
| 6. Mocks and the kit | 85–90 | 13 | Full mocks, then one kit-only session. |

Phase 4 is longest because interviews reward a finished design. Phases 2–3 make sure a shard key is not new on day 50.

## D. Daily lesson template

Same skeleton every non-mock day. It should be automatic by day 10.

1. **Problem.** Problem card only. No hints.
2. **Requirements.** Functions, numbers, non-goals. Assumptions are locked before the write-up confirms them.
3. **Design.** One box, then the design that hits the numbers. API and data model before topology.
4. **Diagrams.** Section E. Read path and write path apart.
5. **Trade-offs.** Two alternatives, the choice, and the 10× break.
6. **Talking points.** What to say, what is hand-waving, short follow-up answers.
7. **Kit artifact.** One checklist row, stencil, problem card, or rubric line.

Mocks replace steps 3–6 with a timer, a blank sheet, and the rubric. The reference comes after the attempt.

## E. Diagram strategy

Six types, used the same way every day:

- **Context.** Actors and trust boundary.
- **Whiteboard.** Few boxes, labeled arrows. Stores named by access pattern, not vendor.
- **Read path and write path.** Separate sequences.
- **Data model.** Entities, keys, indexes, partition key. Before scaling.
- **Scale overlay.** Replica, shard, cache, or queue, and the bottleneck it removes.
- **Failure overlay.** One dependency down: retry, queue, or the user-visible result.

Week 1 ships the blanks; later days only mark them up, and the kit reuses them. If a figure needs a legend, it will not fit a whiteboard.

## F. Interview Prep Kit

What they keep after day 90.

- **Script.** 35- and 45-minute time boxes, and what to say before drawing.
- **Requirements checklist.** Function, scale, freshness, durability, consistency, privacy, non-goals.
- **Estimation sheet.** Formulas and latency orders of magnitude, assumption beside each number.
- **Stencils.** The six blanks plus the Week 1 example.
- **Trade-off card.** Consistency, latency, durability, cost, complexity, operability.
- **Follow-up bank.** By component, each item pointed at the lesson that answered it.
- **Three LLM roles (optional kit).** Interviewer (no hints, demand numbers, one deep dive, then stop). Grader (rubric only, quote the student). Next problem from the gaps. None may design the system.
- **Design log.** Problem, assumptions, score 1–4, one gap. Short glossary in the course’s words.

Lessons fill the kit. It is not dumped complete on day 1.

## G. Sample week 1

All seven days use a pastebin. It returns later for stampede, object storage, and abuse.

| Day | Title | Intent |
|---|---|---|
| 1 | What the interviewer is grading | The hour’s signals. Not a technology tour. |
| 2 | Vague problem to requirements | Scope, non-goals, and the two numbers that dominate. |
| 3 | Capacity math out loud | QPS, storage, bandwidth. Assumptions visible. |
| 4 | API and data model first | Endpoints, retention, and the real lookup key. |
| 5 | One box, and why it breaks | A correct small design, then the limit that forces the next step. |
| 6 | A diagram that survives | Context, whiteboard, read path, write path. Redraw from memory. |
| 7 | Timed dry run | 35 minutes, no notes. Score, then the reference. |

## H. Mocks

Nine timed days. The problem is never that week’s lesson.

- **Day 7.** Pastebin. Installs the rubric.
- **Day 28.** Add cache, durable storage, and a queue.
- **Day 49.** Booking or inventory: consistency and a partition key.
- **Days 63 and 77.** Unseen product problems.
- **Day 84.** A region or dependency dies near minute 25.
- **Days 85–90.** Kit setup on 85. Mocks on 86, 88, and 90: read-heavy, write-heavy, realtime. Day 90 is kit-only. Debriefs on 87 and 89.

## I. Delivery

Text plus diagrams is the course. That is what people reread before an interview, and what the kit is made of. Plan 30–40 minutes: try the problem, then read the design, then redraw.

Optional: an 8–12 minute video of the final picture and two trade-offs. Do not hide the design in video. Live, if any, is a weekly office hour for a cohort, not a daily class. Self-paced has to stand alone.

## J. Open decisions (resolved above)

Kept so the original questions stay visible. The locked list at the top of this file is the answer.


1. **Who is it for?** Backend engineers only, or strong new grads too?
2. **Daily length.** 30–40 minutes, or 60 with a second example?
3. **Papers.** Interview depth only, or a short appendix (Dynamo, Spanner, Raft) that is not a seminar?
4. **Company color.** One method for every interview above, or a thin quirk appendix? Prefer the appendix.
5. **Platform.** Your site, a cohort tool, or Notion/GitHub with the kit as a repo?
6. **Cohort or self-paced?** A cohort is worth it only if you run the Phase 6 mocks.
7. **Packaging.** Course and kit together, or kit also sold alone later?
8. **Video in v1, or text only?**
9. **Who refreshes problem cards** when interview fashion shifts? The problem bank goes stale first.
10. **Certificate.** Recommend none. The design log is the artifact.

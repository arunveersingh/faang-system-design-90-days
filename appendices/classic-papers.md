<!-- appendix-nav -->
[← Company quirks](company-quirks.md) · [Appendices](README.md)

# Classic papers

**Not a day. Not a mock.** Recommend, not required. Open a paper note only when you need vocabulary you can spend in an answer. Do not summarize a paper in the room.

## Time box

Under 40 minutes **per paper**, or skip until an interview is on the calendar. Do not read all three the night before a mock instead of sleeping.

| Paper | Open when | Full note |
|---|---|---|
| Dynamo | Consistent hashing, hinted handoff, sloppy quorum, nodes wrong or gone | [Dynamo](dynamo.md) |
| Spanner | External consistency, why a global transaction is expensive | [Spanner](spanner.md) |
| Raft | Leader, log, election window as a dependency | [Raft](raft.md) |

## Rules for the room

1. Take only what you can use in an answer.
2. Do not cite a paper as permission to skip a consistency choice.
3. Do not derive the protocol on the whiteboard.
4. Days 18, 30, 35, and 36 already own the interview moves. These notes deepen the words; they do not replace those days.

## One sentence each

- **Dynamo:** leaderless copies with overlap rules, and a failure model where a preferred node is gone — plus the repair story you owe if you took a sloppy write.
- **Spanner:** externally consistent transactions at global scale, paid for with synchronized time and cross-region coordination you do not invent on a whiteboard.
- **Raft:** a replicated log with one leader and a bounded window with no leader — buy it as a dependency, do not re-implement votes and terms in the hour.

Pick one note. Close the others.

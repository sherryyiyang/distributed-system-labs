# 2. Raft Consensus Algorithm
**Objective**: Implement the Raft protocol to achieve consensus across distributed systems.

- [MIT 6.5840 Lab 2: Raft](https://nil.csail.mit.edu/6.5840/2023/labs/lab-raft.html)
- [Journey to MIT 6.824 — Lab 2A Raft Leader Election](https://medium.com/codex/journey-to-mit-6-824-lab-2a-raft-leader-election-974087a55740)
- [Lab 2: Raft - GreyWind](https://greywind.hashnode.dev/lab-2-raft)
- [Introduction to Raft - consensus algorithm](https://www.linkedin.com/pulse/introduction-raft-consensus-algorithm-dk-cao-rrwtc)


**Implementation appoach**:
When implementing Raft, first need get the leader election mechanism. I created a Raft struct that tracks currentTerm, votedFor, and the replicated log, and I made sure those fields were persisted using Go’s labgob encoder so that a server could crash and restart safely. Each server starts as a follower, and when a randomized election timer expires, it becomes a candidate and starts a new election. I had to be careful with term comparisons in RequestVote RPCs to make sure older candidates didn’t overwrite newer leaders — debugging those edge cases took some work.

Once election worked, I moved on to log replication. The leader appends commands to its log and sends them to followers via AppendEntries RPCs. I had to maintain nextIndex and matchIndex for each peer so the leader could figure out what entries each follower was missing. Handling mismatched logs was tricky — I wrote logic to backtrack until follower and leader logs matched up. Only once a log entry was stored on a majority could the leader safely commit it.

One of the hardest parts was making sure committed entries were applied in order. I added a background goroutine to apply them to the state machine and used channels to coordinate with the test harness. There were lots of race conditions early on — I used a mutex to protect all shared state, but I also had to learn where locking hurt performance or caused deadlocks.
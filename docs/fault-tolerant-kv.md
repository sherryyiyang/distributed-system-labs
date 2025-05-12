# 3. Fault-Tolerant Key/Value Store
**Objective**: Build a replicated key/value store leveraging the Raft consensus algorithm.

- [MIT 6.5840 Lab 3: Fault-tolerant Key/Value Service](https://nil.csail.mit.edu/6.5840/2023/labs/lab-kvraft.html)
- [Lab 3: Fault-tolerant Key/Value Service · CS 475](https://tddg.github.io/cs475-fall21/lab3.html)
- [MIT 6.824 Lab 4: Fault-tolerant Key/Value Service](https://pdos.csail.mit.edu/6.824/labs/lab-kvraft1.html)

**Implementation appoach**:
When started building the fault-tolerant key/value store on top of it, the main challenge was figuring out how to integrate client requests into the Raft log and ensure they were applied exactly once — even if a request was retried or a leader failed mid-operation.

The first thing I did was define Put, Append, and Get operations as commands, and passed them through Raft’s Start() function. The tricky part was waiting for the command to actually commit before responding to the client. I used a notifyCh per command — basically, each client request gets a channel that’s triggered when Raft applies the log entry. I stored those channels in a map indexed by log index, and had the Raft apply loop send the result when the log entry was committed.

To make the store resilient to duplicate requests, I added a client ID and a sequence number to each command. On the server side, I kept a lastAppliedSeq map keyed by client ID. If a request came in with an old or duplicate sequence number, I skipped re-executing it and just returned the previous result. This made the system idempotent and prevented issues from network retries.

Another big step was snapshotting, because the Raft log can grow infinitely. When the log size passed a threshold, I saved a snapshot of the entire key/value map and the client sequence table. On restart or follower catch-up, Raft would load the snapshot to restore state. I used labgob for serialization and made sure to coordinate snapshotting with log compaction so everything stayed consistent.

This part of the project helped me really connect distributed consensus with actual service behavior — tying Raft's guarantees to real user-facing correctness was a valuable learning experience.

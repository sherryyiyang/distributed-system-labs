
##  5. Persistent Key/Value Service
**Objective**: Enhance the key/value store with persistence to recover from complete system failures.

- [6.824 Lab 5: Persistence](https://nil.csail.mit.edu/6.824/2015/labs/lab-5.html)
- [MIT 6.824 Distributed Systems 2022 - 0xFFFF](https://0xffff.one/d/1815)

**Implementation appoach**:
Adding persistence to a distributed key/value service ensures that the system can recover from full crashes — where all replicas restart and must resume from durable storage. The core of the solution is to persist Raft state and application-level snapshots, and to restore them correctly at startup.

The first thing to handle is Raft log and state persistence. This involves serializing currentTerm, votedFor, and the log entries using something like Go’s labgob encoder and writing them to disk after every update. This guarantees that after a crash, the server can reload Raft and resume consensus operations with no loss of data or election history.

However, persisting the full Raft log indefinitely is not scalable — which is where snapshotting comes in. Once the log grows beyond a certain size, the server should take a snapshot of the entire application state (i.e., the key/value map and client deduplication info). This snapshot replaces the need for the earlier log entries, allowing the log to be compacted.

Snapshotting needs to be tightly integrated with Raft:

When the application takes a snapshot, it must call Raft.Snapshot(index, snapshotData) to trim the log.

The snapshot data should be generated in a consistent state, typically just after applying a committed log entry.

Raft must persist this snapshot alongside its internal state so that the entire node can be rebuilt from disk.

On startup, the server first restores from any existing snapshot, then replays any remaining log entries on top. It's also important that log replication logic correctly skips over log entries that were compacted away.

Getting this right ensures that the system can handle complete crashes, scale better over time, and reduce startup times — which are all critical in production-quality distributed systems.

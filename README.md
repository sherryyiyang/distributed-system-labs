# Distributed Systems Projects in Go
This repository contains my implementations of core distributed systems components, focusing on scalability, fault tolerance, and consistency. Each project is developed in Go, emphasizing practical applications of distributed computing principles.

## 1. MapReduce Framework
**Objective**: Develop a simplified MapReduce framework to process large-scale data across multiple nodes.

- [MapReduce with RPC Pt. 1 – Victor Wu](https://medium.com/@wu.victor.95/building-mapreduce-with-rpc-pt-1-e596062233fd)  
- [Building a Distributed MapReduce System in Go – Better Programming](https://medium.com/better-programming/building-a-distributed-mapreduce-system-in-go-a22a205f5a0)  
- [Distributed MapReduce in Go – Yunus Kilic](https://yunuskilicdev.medium.com/distributed-mapreduce-algorithm-and-its-go-implementation-12273720ff2f)  
- [MapReduce from Scratch in Go – Param Codes](https://newsletter.param.codes/p/mapreduce-from-scratch-in-go)  
- [MapReduce Architecture – GeeksforGeeks](https://www.geeksforgeeks.org/mapreduce-architecture/)  
- [Go MapReduce GitHub Example – kumaab](https://github.com/kumaab/MapReduce)

**Implementation appoach**:
To implement MapReduce, I first built the coordinator, which is responsible for task scheduling and tracking progress. The coordinator maintains two job queues: one for Map tasks and one for Reduce tasks. Each task is represented as a struct containing metadata such as file name, task ID, and status (idle, in-progress, or completed). Workers periodically send RPCs to request a task. The coordinator responds by assigning a pending task, marking it as in-progress, and recording the start time to detect timeouts (e.g., 10 seconds). If a task isn't completed within the timeout window, it's re-queued for another worker.

The worker implementation handles both Map and Reduce phases. Upon receiving a Map task, the worker reads the input file, applies the user-defined mapf function to generate intermediate key/value pairs, and partitions these pairs into NReduce buckets based on a hash of the key. Each bucket is written to a temporary intermediate file named with a pattern like mr-X-Y, where X is the map task ID and Y is the reduce bucket index.

Once all Map tasks are complete, the coordinator begins assigning Reduce tasks. A Reduce worker reads all intermediate files with the same reduce index across different map outputs. It sorts the key/value pairs by key, then applies the user-defined reducef function on each group of values with the same key. The final output is written to a file named mr-out-Y.

To ensure fault tolerance, workers are stateless and can crash or retry safely. The coordinator periodically checks for stalled tasks and can reassign them. Temporary files are renamed atomically to avoid partial writes, ensuring correctness. All RPCs are blocking and use Go's net/rpc package, with careful handling of race conditions and concurrency using mutexes.

This design ensures parallelism, correctness, and fault-tolerance.


## 2. Raft Consensus Algorithm
**Objective**: Implement the Raft protocol to achieve consensus across distributed systems.

**Implementation appoach**:
When implementing Raft, first need get the leader election mechanism. I created a Raft struct that tracks currentTerm, votedFor, and the replicated log, and I made sure those fields were persisted using Go’s labgob encoder so that a server could crash and restart safely. Each server starts as a follower, and when a randomized election timer expires, it becomes a candidate and starts a new election. I had to be careful with term comparisons in RequestVote RPCs to make sure older candidates didn’t overwrite newer leaders — debugging those edge cases took some work.

Once election worked, I moved on to log replication. The leader appends commands to its log and sends them to followers via AppendEntries RPCs. I had to maintain nextIndex and matchIndex for each peer so the leader could figure out what entries each follower was missing. Handling mismatched logs was tricky — I wrote logic to backtrack until follower and leader logs matched up. Only once a log entry was stored on a majority could the leader safely commit it.

One of the hardest parts was making sure committed entries were applied in order. I added a background goroutine to apply them to the state machine and used channels to coordinate with the test harness. There were lots of race conditions early on — I used a mutex to protect all shared state, but I also had to learn where locking hurt performance or caused deadlocks.

## 3. Fault-Tolerant Key/Value Store
**Objective**: Build a replicated key/value store leveraging the Raft consensus algorithm.

**Implementation appoach**:
When started building the fault-tolerant key/value store on top of it, the main challenge was figuring out how to integrate client requests into the Raft log and ensure they were applied exactly once — even if a request was retried or a leader failed mid-operation.

The first thing I did was define Put, Append, and Get operations as commands, and passed them through Raft’s Start() function. The tricky part was waiting for the command to actually commit before responding to the client. I used a notifyCh per command — basically, each client request gets a channel that’s triggered when Raft applies the log entry. I stored those channels in a map indexed by log index, and had the Raft apply loop send the result when the log entry was committed.

To make the store resilient to duplicate requests, I added a client ID and a sequence number to each command. On the server side, I kept a lastAppliedSeq map keyed by client ID. If a request came in with an old or duplicate sequence number, I skipped re-executing it and just returned the previous result. This made the system idempotent and prevented issues from network retries.

Another big step was snapshotting, because the Raft log can grow infinitely. When the log size passed a threshold, I saved a snapshot of the entire key/value map and the client sequence table. On restart or follower catch-up, Raft would load the snapshot to restore state. I used labgob for serialization and made sure to coordinate snapshotting with log compaction so everything stayed consistent.

This part of the project helped me really connect distributed consensus with actual service behavior — tying Raft's guarantees to real user-facing correctness was a valuable learning experience.

##  4. Sharded Key/Value Service
**Objective**: Design a scalable key/value store by partitioning data into shards managed by different Raft groups.

**Implementation appoach**:
To implement a sharded key/value store, the key idea is to partition the key space across multiple Raft groups, allowing the system to scale horizontally. Each group is responsible for managing a subset of the shards, and shard ownership can change dynamically over time.

The first step is to build a Sharding Controller (often called the ShardMaster) that maintains the current mapping of shards to groups. It exposes an RPC interface that servers periodically query to get updated configurations. The controller itself is backed by a separate Raft group to ensure consistency across reconfigurations.

Each server group (i.e., Raft cluster) needs to keep track of:

* Its current configuration ID

* Which shards it owns

* What data belongs to which shard

* Metadata to handle deduplication for client requests

Whenever a configuration change occurs (e.g., shard movement), the server group must detect which shards it gained or lost. For shards it lost, it should stop serving them and discard local data. For shards it gained, it must fetch the data and dedup state from the previous owner, which requires implementing cross-group RPCs with retry logic.

It’s critical to coordinate shard transfers through Raft to ensure atomicity and avoid split-brain conditions. A common pattern is to write a special “install shard” log entry via Raft after fetching the shard data, so all replicas apply it consistently.

During reconfiguration, you also need to pause client requests for keys in "in-transit" shards to avoid serving stale or missing data. A shard should only be considered ready once:

* The group is the official owner in the latest config.

* The data has been transferred and committed via Raft.

This problem forces you to combine consensus, reconfiguration, and state transfer — making it a great exercise in real distributed system design.


##  5. Persistent Key/Value Service
**Objective**: Enhance the key/value store with persistence to recover from complete system failures.
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

## Technologies Used
Language: Go (Golang)

Concurrency: Goroutines and Channels

Communication: Remote Procedure Calls (RPC)

Testing: Comprehensive unit and integration tests

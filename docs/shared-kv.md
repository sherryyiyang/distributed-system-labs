#  4. Sharded Key/Value Service
**Objective**: Design a scalable key/value store by partitioning data into shards managed by different Raft groups.

- [MIT 6.5840 Lab 4: Sharded Key/Value Service](https://nil.csail.mit.edu/6.5840/2023/labs/lab-shard.html)
- [6.824 Lab 4: Sharded Key/Value Service](https://pdos.csail.mit.edu/archive/6.824-2013/labs/lab-4.html)
- [MIT 6.824 Lab 4 ShardKV 实现思路 (CSDN)](https://blog.csdn.net/qq_43460956/article/details/134885751)

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

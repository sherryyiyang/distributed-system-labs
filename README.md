# Distributed Systems Projects in Go
This repository contains my implementations of core distributed systems components, focusing on scalability, fault tolerance, and consistency. Each project is developed in Go, emphasizing practical applications of distributed computing principles.

🗂️ Projects Overview
## 1. MapReduce Framework
Objective: Develop a simplified MapReduce framework to process large-scale data across multiple nodes.

Implementation Highlights:

Designed a coordinator to manage task distribution and handle worker failures.

Implemented worker nodes capable of executing map and reduce tasks concurrently.

Ensured fault tolerance through task re-assignment and result verification.

## 2. Raft Consensus Algorithm
Objective: Implement the Raft protocol to achieve consensus across distributed systems.

Implementation Highlights:

Handled leader election, log replication, and safety guarantees.

Managed persistent state to recover from node failures.

Ensured consistency and availability during network partitions.

## 3. Fault-Tolerant Key/Value Store
Objective: Build a replicated key/value store leveraging the Raft consensus algorithm.

Implementation Highlights:

Supported Put, Append, and Get operations with linearizability.

Ensured data consistency across replicas and handled client retries gracefully.

Integrated snapshotting to optimize storage and recovery.

##  4. Sharded Key/Value Service
Objective: Design a scalable key/value store by partitioning data into shards managed by different Raft groups.

Implementation Highlights:

Implemented a shard controller to manage dynamic shard assignments.

Handled shard migrations seamlessly during configuration changes.

Maintained system consistency and availability during reconfigurations.


##  5. Persistent Key/Value Service
Objective: Enhance the key/value store with persistence to recover from complete system failures.

Implementation Highlights:

Integrated persistent storage mechanisms for Raft logs and snapshots.

Ensured data durability and quick recovery post-crash.

Validated system reliability through rigorous failure simulations.
CSDN Blog

# 🛠️ Technologies Used
Language: Go (Golang)

Concurrency: Goroutines and Channels

Communication: Remote Procedure Calls (RPC)

Testing: Comprehensive unit and integration tests

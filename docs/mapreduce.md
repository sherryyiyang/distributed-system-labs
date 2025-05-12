# 1. MapReduce Framework
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
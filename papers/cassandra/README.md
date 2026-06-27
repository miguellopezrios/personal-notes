# Cassandra - A Decentralized Structured Storage System

## Introduction

Cassandra was originally developed at Facebook to meet the scalability, availability, and reliability requirements of large-scale applications running on thousands of commodity servers, where failures are considered normal rather than exceptional. It was initially designed to support Inbox Search, a feature that allows users to search their Facebook Inbox. Since users were distributed across multiple geographical regions, Cassandra replicated data across data centers to reduce search latency while maintaining high availability. The system was also designed to handle extremely high write throughput, reaching billions of writes per day.

## System architecture

The typical behavior is:

- Any node in the Cassandra cluster can receive a read or write request. The node that receives the request acts as the **coordinator** for that operation and determines which replicas are responsible for the requested key.

- For writes, the coordinator forwards the request to the replicas responsible for the key and waits for a configurable number of acknowledgements (typically a quorum) before returning success to the client.

- For reads, Cassandra supports configurable consistency levels. Depending on the requested consistency, the coordinator can:
  1. Query the closest replica and return the result immediately.
  2. Query multiple or all replicas and wait for a quorum of responses.

  If replicas return different versions of the data, Cassandra uses timestamps to determine the latest version and performs **read repair** by updating stale replicas in the background. This mechanism ensures that replicas gradually converge toward consistency without requiring synchronous repair.

## Partitioning

Cassandra partitions data using **consistent hashing**. Each node is assigned a position (token) on the ring. A key is stored on the first node whose token is greater than or equal to the key's token. The node that receives the request acts as the coordinator for routing the operation. When a node joins or leaves the cluster, only a subset of keys must be reassigned, minimizing data movement.

Consistent hashing provides scalability because nodes can be added or removed without requiring a complete redistribution of data.

To address uneven data distribution and workload imbalances, Cassandra monitors node load and can move lightly loaded nodes to different positions in the ring, helping distribute load more evenly across the cluster.

## Replication

To achieve high availability and durability, Cassandra replicates data across multiple nodes.

A configurable replication factor (N) determines how many copies of data are stored. Each key is assigned to a coordinator node responsible for managing replication for that key range. The coordinator stores the data locally and replicates it to the next N-1 nodes on the ring.

ZooKeeper is used for certain coordination tasks such as leader election.

## Membership

Cassandra uses a gossip-based membership protocol.

Nodes periodically exchange state information with other nodes. Through repeated gossip, each node eventually learns about all other nodes in the cluster. Gossip is also used to propagate membership, status, and system state information.

This design avoids centralized membership management and scales efficiently.

## Failure detection

Failure detection is responsible for determining whether nodes in the cluster are reachable.

Cassandra uses a modified version of the Φ (Phi) Accrual Failure Detector.

Unlike traditional failure detectors that return a simple "alive" or "failed" result, the Φ Accrual Failure Detector produces a suspicion score (Φ) representing the likelihood that a node has failed.

The mechanism works as follows:

- Nodes periodically exchange gossip messages containing heartbeat information.

- Each node maintains a sliding window of heartbeat inter-arrival times for every monitored node.

- Based on the observed heartbeat history, Cassandra computes a Φ value.

- When Φ exceeds a configurable threshold, the node is considered unavailable.

The original Φ Accrual Failure Detector assumed a **Gaussian Distribution** of heartbeat intervals. Cassandra modifies this approach by using an **Exponential Distribution**, which was found to better match the behavior of real production systems.

## Local Persistence Architecture

Cassandra is optimized for **high write throughput** and **durable storage**.

Every write is first appended to a **commit log** stored on disk. This ensures that updates can be recovered in the event of a node failure. After being written to the commit log, updates are stored in an in-memory data structure. When this structure reaches a threshold size, its contents are flushed to disk. Flushed data is written to disk as immutable files.

Because these files are never modified:

- Writes are sequential.
- Reads require no locking.
- System scales well under heavy write loads.

**Bloom filters** are used to quickly determine whether a key may exist in a file. This reduces unnecessary disk reads.

Over time, multiple immutable files accumulate, so a background compaction process merges these files, removes obsolete versions of data, and improves read performance.

## Implementation Details

Cassandra process on a single machine consists of a:

- Partitioning module.
- Cluster membership module -> built on top of a network layer which uses non-blocking I/O.
- Failure detection module -> built on top of a network layer which uses non-blocking I/O.
- Storage engine module.

Both UDP (for system control messages) and TCP (for application messages) are used. A state machine is used to implement the request routing module.

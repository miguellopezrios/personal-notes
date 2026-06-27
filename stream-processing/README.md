# Transmitting Event Streams

The smallest unit in stream processing is an event (a small, immutable, and self-contained record representing something that happened at a specific point in time). These events are produced by a producer and consumed by a consumer. They can be stored and propagated through different mechanisms, such as files, databases, or messaging systems. However, in continuous processing systems, the overhead introduced by polling a datastore to detect changes can be significant. A better approach is to be notified whenever a new event arrives, which is where messaging systems come into play.

Some important considerations related to messaging systems are:

- Direct messaging from producers to consumers. Examples include UDP multicast, or consumers exposing services on a network and producers making HTTP or RPC requests to those services.

- Message brokers: a specialized intermediary that stores and distributes messages between producers and consumers. Some examples of message brokers are RabbitMQ and Google Cloud Pub/Sub.

- Multiple consumers: when multiple consumers read messages, the two most common patterns are load balancing (a specific message is delivered to only one consumer) and fan-out (the same message is delivered to multiple consumers).

- Acknowledgments and redelivery: brokers typically rely on acknowledgments (ACKs) to confirm that a message has been successfully processed. A common challenge appears when a consumer performs some side effect (for example, updating a database) but crashes before acknowledging the message. The broker may then redeliver the message, causing the operation to be executed twice. Atomic commit protocols are a common approach to address this problem.
  Another important consideration is that, when load balancing and redeliveries are involved, message ordering is not guaranteed.

## Partitioned logs

Traditional message brokers often remove messages once they have been successfully acknowledged by consumers. This contrasts with databases and filesystems, where historical data remains available and can be read by new clients at any time. So a natural question arises: why not combine both approaches? This idea is the foundation of log-based message brokers.

When implementing message brokers using partitioned logs, the producer appends to the end of the log and consumers read messages sequentially. We can now define a topic as a set of partitions, which allows higher throughput. Each partition has its own offset, so that messages within a partition are ordered. However, order is guaranteed only within a single partition, not across partitions. Apache Kafka or Amazon Kinesis Streams are examples of log-based message brokers.

This log-based approach naturally supports fan-out, while load balancing is achieved by assigning partitions to consumers. This has some limitations: the number of active consumers is limited by the number of partitions, and there can be a bottleneck if one partition receives more load or contains slower-to-process messages.
Therefore, JMS/AMQP-style message brokers are better when the objective is to process messages individually and ordering is not important. In contrast, log-based message brokers are preferable when each message is relatively fast to process and ordering (within a partition) matters.

Regarding disk space usage, the log can grow until storage is full or retention limits are reached. When this limit is reached, old messages are removed according to the retention policy. In practice, this is not usually a problem because logs are often configured to retain data for several days or even weeks.

# Databases and Streams

Some ideas from databases can be applied to streams, and also the opposite. As an example, a replication log is just a set of events (database operations) that are produced by the leader when processing transactions and then consumers process those events to apply the changes to their own copy of the database.

When multiple systems are being used together (databases, cache, data warehouses, etc) it is difficult to keep the systems in sync and some race conditions can occur if we use techniques like dual writes.

One interesting approach for data sync is Change Data Capture (CDC), which allows to detect changes in a database and apply the same changes to other systems. This approach effectively allows the database to act as a source of truth that downstream systems can follow. In practice, it is often implemented by consuming the database’s replication log, which is maintained using techniques such as periodic consistent snapshots and log compaction to prevent unbounded storage growth. Kafka Connect integrates CDC tools for database systems with Kafka.

A related concept is event sourcing. Instead of storing the final effects of operations, the system records user actions (commands) as an immutable sequence of events. These events become the primary source of truth, and the application state is derived by replaying them deterministically. This model makes it easier to derive additional side effects or build new read models from the same event stream. However, it also shifts complexity into the application, which must reliably persist events and ensure they can be transformed consistently into state. As a result, long-term storage concerns remain relevant, and techniques such as log compaction or snapshotting are still required to control growth.

There are some similarities between CDC and event sourcing. In CDC, the application uses the database in a mutable way (for example, extracting changes from the replication log to avoid race conditions), while in event sourcing the gist is immutability. Also, log compaction is handled differently.

In event sourcing there is a difference between "commands" and "events". An initial request from the user is a command. If the validations for a command are satisfied, it is transformed into an event (immutable and durable).

A key idea is that database records represent the system's current mutable state, while an append-only log captures the immutable sequence of events that produced that state. These two representations are not in conflict; in fact they are complementary views of the same underlying system.

A downside of CDC and event sourcing is that consumers of the event log are usually asynchronous, which may have some problems such as reading your own writes. This can be solved with techniques like updating the read view synchronously during write operations. A more robust alternative is to rely on linearizable storage built on a total-order broadcast mechanism. On the other hand, when both the event log and the application state share the same partitioning scheme, a single-threaded consumer can process log entries sequentially, eliminating the need for write-side concurrency controls.

Moreover, immutable systems can have degraded performance if we keep increasing the append-only log.

# Processing Streams

After discussing where streams come from and how are they transported, now the question is: what can we do with them?

There are essentially three options:

1. Take the streams in one system to keep other systems in sync.

2. Push the events to the users (sending emails or push notifications).

3. Processing streams to produce other streams (known as "derived streams").

Some of the use cases of stream processing are fraud detection systems, trading systems, manufacturing systems and military/intelligence systems.

Stream processing is similar to batch processing because it treats inputs as immutable and writes results in an append-only manner. The key difference is that streams are unbounded and continuous, so operations that require a complete dataset, such as global sorting, are impractical. Likewise, simply rerunning a failed job from the beginning is often not feasible.

Complex Event Processing (CEP) takes a different approach by storing queries or patterns for long periods and continuously comparing incoming events against them to detect matches.

Many analytical tasks summarize data over fixed time intervals known as windows. Some stream processing frameworks rely on the local system clock to assign events to windows, but this approach can fail when there is substantial processing delay or lag. Not having a clear distinction between event time and processing time leads to bad/inconsistent data.

Defining windows in terms of event time is tricky because you never know when there are no more events in a specific window, since you may think that the events have finished for that window, but network delays or other problems may occur. In this case, the options are to ignore these minimal cases, or publishing corrections to update the value in a particular window.

Other clock-related problems can occur. If we are processing events from a mobile app that is being used offline, the events can appear hours or even days after, producing delayed events. Using the mobile local clock is wrong since it can be set to the wrong time, and using the server's clock can be more accurate but it does not precisely reflect user interaction. A solution to this is to log three timestamps: the time at which the event occurred (based on the device clock), when it was sent to the server (based on the device clock) and when it was received by the server (based on the server clock).

Some common window types are: tumbling windows, hopping windows, sliding windows, and session windows.

Similar to batch processing, stream processing often requires join operations. These may involve joining one stream with another stream, a stream with a table, or two tables together. In all three cases, the stream processor must maintain state derived from one side of the join and consult that state when processing records from the other side. As a result, event ordering becomes a critical concern, since out of order updates can lead to inconsistent join results. A common solution is to associate records with version numbers or unique identifiers that allow the processor to determine which version of a joined record is current. However, that has downsides (for example, log compaction is not possible because all versions of the records in the table need to be maintained).

Related to fault tolerance, in batch processing frameworks this is solved by just running again a failed job, but in stream processing this is not as straighforward to handle, since streams are infinite. Some solutions are microbatching (used in Spark Streaming) and checkpointing (used in Apache Flink), which provides exactly-once semantics similar to batch processing. This holds only while the output lives in the stream processor, but once the output leaves the stream processor, these solutions are not enough.

In some cases, atomic commits are required to prevent side effects from being applied more than once. Fortunately, the overhead of transaction protocols can be amortized by batching multiple messages into a single transaction.

An alternative to transactions is to use idempotent write operations that can be executed multiple times while producing the same effect as a single execution. This can achieve exactly-once semantics with minimal overhead.

Finally, to rebuild state after a failure, there are tradeoffs between storing the state local vs remote, and there is no universal truth on when to use one or the other.

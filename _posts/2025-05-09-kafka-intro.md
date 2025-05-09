---
title: Apache Kafka - A Deep Dive into the Distributed Streaming Platform
date: 2025-05-09 12:00:00 +600
categories: [technical]
tags: [Apache Kafka]
---

Apache Kafka is an open-source distributed event streaming platform designed to handle high volumes of data in real-time. Originally developed by LinkedIn and later donated to the Apache Software Foundation, Kafka has become a cornerstone for building real-time data pipelines and streaming applications. It excels at reliably processing and moving large streams of data from one system to another, enabling applications to react to events as they happen. Kafka combines messaging, storage, and stream processing capabilities, allowing for both historical and real-time data analysis

At its core, Kafka provides three main functions: publishing and subscribing to streams of records, storing these streams durably and in the order they were generated, and processing these streams in real-time. This makes it suitable for a wide array of use cases, including tracking website activity, aggregating logs, processing financial transactions, and powering event-driven architectures.

## Core Components: Producers, Brokers, and Consumers

The Kafka ecosystem revolves around three key components that work in concert:

<strong>Producers:</strong> These are client applications responsible for creating and sending data (records or messages) to Kafka. Producers publish messages to specific categories called "topics." They can choose to send messages synchronously, waiting for confirmation of delivery, or asynchronously for higher throughput. Producers can also specify a key with each message, which influences how messages are distributed across a topic's partitions.

<strong>Brokers:</strong> A Kafka cluster consists of one or more servers called brokers. Brokers are the heart of Kafka, responsible for storing the data published by producers and making it available to consumers. Each broker manages a subset of the data and handles read and write requests. Brokers are designed to be stateless and can be scaled horizontally by adding more servers to the cluster, enhancing both capacity and fault tolerance.

<strong>Consumers:</strong> These are client applications that subscribe to topics and process the messages published to them. Consumers read messages from the partitions within a topic. Kafka enables messages to be consumed by multiple independent consumers or by groups of consumers working together.

## Interaction Flow:
Producers write messages to a specific topic within the Kafka cluster. The broker that is the leader for the target partition receives this message, writes it to its log, and replicates it to follower brokers. Once the message is successfully written and replicated (based on configuration), the broker can send an acknowledgment back to the producer. Consumers, on the other hand, subscribe to one or more topics and pull messages from the brokers. They maintain their own pace of consumption.

## Topics and Partitioning: Enabling Scalability and Parallelism
In Kafka, a topic is a category or feed name to which records are published. Think of it as a logical channel for a specific type of data. For instance, you might have a topic for "user_clicks" or "order_updates."

To enable scalability, fault tolerance, and parallelism, Kafka topics are divided into one or more partitions. Each partition is an ordered, immutable sequence of records, acting as a structured commit log. New messages are appended to the end of a partition.

## How Partitioning Works:

<strong>Distribution:</strong> Partitions for a topic can be distributed across multiple brokers in the Kafka cluster. This means a single topic can scale beyond the capacity of a single server.

<strong>Ordering:</strong> Kafka guarantees the order of messages within a single partition. However, it does not guarantee global order across all partitions in a topic by default.

<strong>Message Assignment:</strong> When a producer sends a message to a topic, it can determine which partition the message goes to:
Key-based: If a message includes a key, Kafka uses a hash of the key to assign the message to a specific partition. This ensures that all messages with the same key always go to the same partition, thus maintaining order for messages related to that key (e.g., all events for a specific customer ID).

<strong>Round-robin:</strong> If no key is provided, messages are typically distributed in a round-robin fashion across the available partitions, balancing the load.

<strong>Custom Partitioner:</strong> Producers can also implement custom logic to decide the partition assignment.
The number of partitions for a topic is configurable and impacts the degree of parallelism you can achieve during consumption.

The number of partitions for a topic is configurable and impacts the degree of parallelism you can achieve during consumption.


## Read and Write Operations: The Journey of a Message

**Writing Data (Production):**

1.  A producer application creates a record, specifying the target topic and optionally a key and partition.
2.  The producer sends the record to a Kafka broker. The producer can be configured to wait for various levels of acknowledgment (`acks`) from the broker(s) to ensure reliability:
    * `acks=0`: Producer doesn't wait for any acknowledgment (potential data loss).
    * `acks=1`: Producer waits for an acknowledgment from the leader broker of the partition (default).
    * `acks=all`: Producer waits for an acknowledgment from the leader and all in-sync replicas (ISR), providing the strongest durability guarantee.
3.  The leader broker for the target partition receives the message and appends it to the partition's log segment on disk.
4.  Follower brokers for that partition replicate the message from the leader.
5.  Once the configured acknowledgment conditions are met, the leader sends an acknowledgment back to the producer.

Producers can also batch messages together before sending them to a broker to improve efficiency and throughput.

**Reading Data (Consumption):**

1.  A consumer application subscribes to one or more topics.
2.  Consumers fetch messages from the brokers. They specify the topic, partition, and an offset from which to start reading.
3.  Kafka delivers messages to consumers in the order they are stored within each partition.
4.  Consumers process the messages.
5.  Crucially, consumers are responsible for tracking which messages they have successfully processed. This is done by committing offsets.

## Resilience: What Happens When a Node Goes Down?

Kafka is designed for fault tolerance. This is primarily achieved through **replication**.

* **Replication Factor:** When creating a topic, you can define a replication factor. This specifies how many copies (replicas) of each partition should exist across different brokers in the cluster. A typical replication factor is 2 or 3.
* **Leader and Followers:** For each partition, one broker acts as the **leader**, and the other brokers holding replicas act as **followers**. All read and write requests for a partition go through the leader. Followers passively replicate the data from the leader.
* **In-Sync Replicas (ISR):** Followers that are up-to-date with the leader are called in-sync replicas.
* **Node (Broker) Failure:**
    * **If a follower broker fails:** The leader continues to serve requests. The failed follower will try to catch up once it's back online. If it remains offline, it might be removed from the ISR list.
    * **If a leader broker fails:** Kafka's controller (a designated broker, or managed via ZooKeeper/KRaft in newer versions) detects the failure. It then elects one of the in-sync followers as the new leader for the affected partitions. This leader election process is designed to be automatic and quick, minimizing downtime.
* **Data Safety:** The `acks=all` producer setting, combined with a sufficient number of in-sync replicas (configured by `min.insync.replicas`), ensures that messages are not lost even if the leader fails shortly after acknowledging a write, as the message would have already been replicated to the ISRs.

This replication mechanism ensures that data remains available and the system continues to operate even if some brokers experience failures.

## Consumer Groups: Enabling Parallel Processing

To process large volumes of messages efficiently and in parallel, Kafka uses the concept of **consumer groups**.

* **Shared Workload:** A consumer group consists of one or more consumers that jointly consume messages from one or more topics. Each consumer in the group is assigned a unique set of partitions from the subscribed topics. This means that a single partition is consumed by only one consumer within the same consumer group at any given time.
* **Scalability:** If you have a topic with N partitions, you can have up to N consumers in a group, each processing messages from a distinct partition. This allows you to scale out your message processing by simply adding more consumers to the group (up to the number of partitions). If you have more consumers than partitions, some consumers will be idle.
* **Fault Tolerance:** If a consumer within a group fails or leaves the group, Kafka automatically triggers a **rebalance**. During a rebalance, the partitions previously assigned to the failed consumer are reassigned to the remaining active consumers in the group. Similarly, when a new consumer joins a group, a rebalance occurs to distribute partitions among the new set of consumers.
* **Independent Consumption:** Different consumer groups can consume from the same topic independently. Each group maintains its own offset tracking, allowing them to process the same data for different purposes without interfering with each other. For example, one consumer group might be responsible for real-time analytics, while another archives the data to a data lake.

Consumer groups are identified by a `group.id` configuration. All consumers sharing the same `group.id` belong to the same consumer group.

## Offset Tracking: Knowing Where You Left Off

An **offset** in Kafka is a unique, sequential integer that identifies each record within a partition. It represents the position of a message in that partition's log.

**How Offset Tracking Works:**

* **Consumer Responsibility:** Consumers are responsible for keeping track of the offsets of the messages they have processed. As a consumer reads messages from a partition, it "commits" the offset of the last successfully processed message.
* **`__consumer_offsets` Topic:** Kafka stores these committed offsets in a special internal topic called `__consumer_offsets`. Each consumer group, for each partition it consumes, stores its last committed offset here.
* **Resuming Consumption:** When a consumer starts (or restarts after a failure or rebalance), it reads the last committed offset for its assigned partitions from the `__consumer_offsets` topic. It then begins consuming messages from the next offset, ensuring that it picks up where it left off and avoids reprocessing already handled messages (or minimizes it, depending on commit strategy).
* **Commit Strategies:** Consumers can commit offsets in a few ways:
    * **Automatic Commit:** The Kafka consumer client can be configured to commit offsets automatically at regular intervals (`auto.commit.enable=true`, `auto.commit.interval.ms`). This is convenient but can lead to duplicate processing or lost messages if a consumer fails between processing a message and the next auto-commit.
    * **Manual Commit (Synchronous):** The consumer explicitly calls `commitSync()`. This blocks until the offset commit is successful. It offers "at-least-once" processing semantics (a message might be processed more than once if a failure occurs after processing but before committing).
    * **Manual Commit (Asynchronous):** The consumer calls `commitAsync()`. This sends the commit request and continues processing without waiting for a response. It offers better throughput but requires more careful error handling.
* **Seek Operations:** Consumers also have the ability to manually set their read position to a specific offset using `seek()` operations (e.g., `seekToBeginning()`, `seekToEnd()`, or `seek(partition, offset)`). This can be useful for replaying messages or skipping ahead.

Effective offset management is crucial for building reliable streaming applications that can handle failures and ensure data is processed correctly, typically aiming for "at-least-once" or "exactly-once" processing semantics, depending on the application's requirements and how commits are handled.

## So, what's the main idea?

Basically, Apache Kafka has different parts that work together:

* **Producers** send out messages.
* **Brokers** manage all these messages.
* **Consumers** read and use the messages.

Kafka also has some smart ways of working, like:

* Splitting data into smaller bits called **partitions**.
* Keeping copies of data for safety (this is called **replication**).
* Letting **consumer groups** work as a team to read data.
* Always remembering which messages have been read (using things called **offsets**).

Because of all these parts and smart ways of working, Kafka is a really great tool for building strong systems that can handle a lot of data as it comes in, in real-time.

It might look a bit complicated at first, with all its different pieces. But those pieces are exactly why Kafka is so powerful and can grow to handle much more data without any trouble. You can think of it as a very reliable helper that makes sure all your important data gets where it needs to go.
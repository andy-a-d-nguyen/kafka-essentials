# Kafka

- [Kafka](#kafka)
  - [Topics, partitions and offsets](#topics-partitions-and-offsets)
    - [Topic example: truck_gps](#topic-example-truck_gps)
  - [Brokers](#brokers)
  - [Topic replication factor](#topic-replication-factor)
  - [Leader for a Partition](#leader-for-a-partition)
  - [Producers](#producers)
    - [Producers: Message Keys](#producers-message-keys)
  - [Consumers](#consumers)
    - [Consumer Groups](#consumer-groups)
    - [Consumer Offsets](#consumer-offsets)
    - [Delivery Semantics for Consumers](#delivery-semantics-for-consumers)
  - [Kafka Broker Discovery](#kafka-broker-discovery)
  - [Zookeeper](#zookeeper)
  - [Kafka Guarantees](#kafka-guarantees)
  - [Client Bi-Directional Compatibility](#client-bi-directional-compatibility)
  - [Producer Acks Deep Dive](#producer-acks-deep-dive)
    - [acks = 0 (no acks)](#acks--0-no-acks)
    - [acks = 1 (leader acks)](#acks--1-leader-acks)
    - [acks = all (replicas acks)](#acks--all-replicas-acks)
  - [Producer Retries](#producer-retries)
  - [Idempotent Producer](#idempotent-producer)
  - [Safe Producer Summary & Demo](#safe-producer-summary--demo)
  - [Message Compression](#message-compression)
    - [Message Compression Recommendations](#message-compression-recommendations)
  - [linger.ms & batch.size](#lingerms--batchsize)
  - [Producer Default Partitioner and How Keys are Hashed](#producer-default-partitioner-and-how-keys-are-hashed)
  - [max.block.ms & buffer.memory](#maxblockms--buffermemory)
  - [Delivery Semantics](#delivery-semantics)
    - [At Most Once](#at-most-once)
    - [At Least Once](#at-least-once)
    - [Exactly Once](#exactly-once)
    - [Bottomline](#bottomline)
  - [Consumer Poll behavior](#consumer-poll-behavior)
  - [Consumer Offset Commit Strategies](#consumer-offset-commit-strategies)
  - [Consumer Offset Reset Behavior](#consumer-offset-reset-behavior)
    - [Replaying data for Consumers](#replaying-data-for-consumers)
    - [Bottomline](#bottomline-1)
  - [Controlling Consumer Liveliness](#controlling-consumer-liveliness)
    - [Consumer Heartbeat Thread](#consumer-heartbeat-thread)
    - [Consumer Poll Thread](#consumer-poll-thread)
  - [Kafka Connect Introduction](#kafka-connect-introduction)
    - [Why Kafka Connect and Streams](#why-kafka-connect-and-streams)
      - [Why Kafka Connect](#why-kafka-connect)
      - [Kafka Connect - High Level](#kafka-connect---high-level)
  - [What is Kafka Streams?](#what-is-kafka-streams)
  - [The need for a schema registry](#the-need-for-a-schema-registry)
    - [Confluent Schema Registry Purpose](#confluent-schema-registry-purpose)
    - [Schema Registry Gotchas](#schema-registry-gotchas)
  - [Partition Count and Replication Factor](#partition-count-and-replication-factor)
    - [Partition Count](#partition-count)
    - [Replication Factor](#replication-factor)
    - [Clusters](#clusters)
  - [Big Data Ingestion](#big-data-ingestion)
  - [Kafka Cluster Setup High Level Architecture](#kafka-cluster-setup-high-level-architecture)
    - [Kafka Cluster Setup Gotchas](#kafka-cluster-setup-gotchas)
    - [Kafka Monitoring and Operations](#kafka-monitoring-and-operations)
    - [The need for encryption, authentication & authorization in Kafka](#the-need-for-encryption-authentication--authorization-in-kafka)
      - [Encryption in Kafka](#encryption-in-kafka)
      - [Authentication in Kafka](#authentication-in-kafka)
      - [Authorization in Kafka](#authorization-in-kafka)
  - [Kafka Multi Cluster + Replication](#kafka-multi-cluster--replication)
  - [Advanced Kafka - Topic Configurations](#advanced-kafka---topic-configurations)
    - [Why should I care about topic config?](#why-should-i-care-about-topic-config)
  - [Partitions and Segments](#partitions-and-segments)
    - [Segments and Indexes](#segments-and-indexes)
    - [Segments: Why should I care?](#segments-why-should-i-care)
  - [Log Cleanup Policies](#log-cleanup-policies)
    - [Log Cleanup: Why and When?](#log-cleanup-why-and-when)
    - [Log Cleanup Policy: Delete](#log-cleanup-policy-delete)
  - [Log Compaction](#log-compaction)
    - [Log Cleanup Policy: Compact](#log-cleanup-policy-compact)
      - [Log Compaction Guarantees](#log-compaction-guarantees)
      - [Log Compaction Myth Busting](#log-compaction-myth-busting)
  - [min.insync.replicas](#mininsyncreplicas)
  - [unclean.leader.election](#uncleanleaderelection)
  - [Understanding Communication between Client and Kafka](#understanding-communication-between-client-and-kafka)

## Topics, partitions and offsets

- Topic: a particular stream of data
  - Similar to a table in a database (without all the constraints)
  - You can have as many topics as you want
  - A topic is identified by its name
- Topics are split into partitions
  - Each partition is ordered
  - Each message within a partition gets an incremental id, called offset
- Offsets only have a meaning for a specific partition
  - Ex: Offset 3 in partition 0 doesn't represent the same data as offset 3 in partition 1
- Order is guaranteed only within a partition (not across partitions)
- Data is kept only for a limited time (default is one week)
- Once the data is written to a partition, it can't be changed (immutability)
- Data is assigned randomly to a partition unless a key is provided

### Topic example: truck_gps

Ex:

- You have a fleet of trucks, each truck reports its GPS position to Kafka
- You can have a topic truck_gps that contains the position of all trucks
- Each truck will send a message to Kafka every 20 seconds, each message will contain the truck ID and the truck position (latitude and longitude)
- We choose to create that topic with 10 partitions (arbitrary number)

## Brokers

- A Kafka cluster is composed of multiple brokers (servers)
- Each broker is identified with its ID (integer)
- Each broker contains certain topic partitions
- After connecting to any broker (called a bootstrap broker), you will be connected to the entire cluster
- A good number to get started is 3 brokers, but some big clusters have over 100 brokers

## Topic replication factor

- Topics should have a replication factor of > 1 (usually between 2 and 3)
- This way if a broker is down, another broker can serve the data

## Leader for a Partition

- At any time only ONE partition can be a leader for a given topic across brokers
- Only that leader can receive and serve data for a partition
- The other brokers will synchronize the data
- Therefore each partition has one leader and multiple ISR (in-sync replica)

## Producers

- Producers write data to topics (which is made of partitions)
- Producers automatically know which broker and partition to write to
- In case of broker failures, producers will automatically recover
- Producers can chooses to receive acknowledgement of data writes
  - acks = 0: Producer won't wait for acknowledgement (possible data loss)
  - acks = 1 : Producer will wait for leader acknowledgement (limited data loss)
  - acks = all: Leader + replicas acknowledgement (no data loss)

### Producers: Message Keys

- Producers can choose to send a key with the message (string, number, etc...)
- If key = null, data is sent round robin (broker 101 then 102 then 103...)
- If a key is sent, then all messages for that key will always go to the same partition
- A key is sent if you need message ordering for a specific field (ex: truck_id)

## Consumers

- Consumers read data from a topic (identified by name)
- Consumers know which broker to read from
- In case of broker failures, consumers know how to recover
- Data is read in order within each partition

### Consumer Groups

- Consumers read data in consumer groups
- Each consumer within a group reads from exclusive partitions
- If you have more consumers than partitions, some consumers will be inactive

### Consumer Offsets

- Kafka stores the offsets at which a consumer group has been reading
- The offsets are committed live in a Kafka topic named __consumer_offsets
- When a consumer in a group has processed data received from Kafka, it should be committing the offsets
- If a consumer dies, it will be able to read back from where it left off thanks to the committed consumer offsets

### Delivery Semantics for Consumers

- Consumers choose when to commit offsets
- There are 3 delivery semantics:
  - At most once:
    - Offsets are committed as soon as the message is received
    - If the processing goes wrong, the message will be lost (it won't be read again)
  - At least once (usually preferred):
    - Offsets are committed after the message is processed
    - If the processing goes wrong, the message will be read again
    - This can result in duplicating processing of messages. Make sure your processing is idempotent (processing the messages again won't impact your systems)
  - Exactly once:
    - Can be achieved with Kafka => Kafka workflows using Kafka Streams API
    - For Kafka => external system workflow, use an idempotent consumer

## Kafka Broker Discovery

- Every Kafka broker is also called a "bootstrap server"
  - By connecting to one broker, you will be connected to the entire cluster
- Each broker knows about all brokers, topics and partitions (metadata)

## Zookeeper

- Zookeeper manages broker (keeps a list of them)
- Zookeeper helps in performing leader election for partitions
- Zookeeper sends notifications to Kafka in case of changes (e.g. new topics, broker dies, broker comes up, delete topics, etc)
- Kafka can't work without Zookeeper
- Zookeeper by design operates with an odd number of servers (3, 5, 7)
- Zookeeper has a leader (handle writes) while the rest of the servers are followers (handle reads)
- Zookeeper does not store consumer offsets with Kafka > v0.10

## Kafka Guarantees

- Messages are appended to a topic-partition in the order they are sent
- Consumers read messages in the order stored in a topic-partition
- With a replication factor of N, producers and consumers can tolerate up to N - 1 brokers being down
  - This is why a replication factor of 3 is a good idea:
    - Allows for one broker to be taken down for maintenance
    - Allows for another broker to be taken down unexpectedly
- As long as the number of partitions remains constant for a topic (no new partitions), the same key will always go to the same partition

## Client Bi-Directional Compatibility

- An older client (Ex: 1.1) can talk to a newer broker (2.0)
- A newer client (Ex: 2.0) can talk to an older broker (1.1)

## Producer Acks Deep Dive

### acks = 0 (no acks)

- No response is requested
- If the broker goes offline or an exception happens, we won't know and will lose data
- Useful for data where it's okay to lose messages such as:
  - Metrics collection
  - Log collection

### acks = 1 (leader acks)

- Leader response is requested, but replication is not a guarantee (happens in the background)
- If an ack is not received, the producer may retry
- If the leader broker goes offline, but replicas haven't replicated the data yet, we have a data loss

### acks = all (replicas acks)

- Leader + Replicas ack requested
- Added latency and safety
- No data loss if enough replicas
- acks = all must be used in conjunction withj min.insync.replicas
- min.insync.replicas can be set at the broker or topic level (override)
- min.insync.replicas = 2 implies that at least 2 brokers that are ISR (including leaders) must respond that they have the data
- That means if you use replication.factor = 3, min.insync.replicas = 2, acks = all, you can only tolerate 1 broker going down, otherwise the producer will receive an exception on send

## Producer Retries

- In case of transient failures, developers are expected to handle exceptions, otherwise the data will be lost
- Example of transient failures:
  - NotEnoughReplicasException
- There is a "retries" setting
  - Defaults to 0
  - You can increase to a high number, ex: Integer.MAX_VALUE
- In case of retries, by default there is a chance that messages will be sent out of order (if a batch has failed to be sent)
  - If you reply on key-based ordering, that can be an issue
  - For this, you can set the setting that controls how many produce requests can be made in parallel: min.in.flight.rquests.per.connection
    - Default: 5
    - Set it to 1 if you need to ensure order (may impact throughput)

## Idempotent Producer

- Problem: the Producer can introduce duplicate messages in Kafka due to network errors
- In Kafka >= 0.11, you can define an "idempotent producer" which won't introduce duplicates on network error
  - During producer retries, the producer sends a produce request ID to Kafka, which Kafka checks to see if the message has been committed and if so, just send an ack back
- Comes with:
  - retires = Integer.MAX_VALUE
  - max.in.flight.requests = 1 (Kafka >= 0.11 & < 1.1) or
  - max.in.flight.requests = 5 (Kafka >= 1.1 - higher performance)
  - acks = all
- To create an idempotent producer, set:
  - producerProp.put("enable.idempotence", true)

## Safe Producer Summary & Demo

- Kafka < 0.11
  - acks = all (producer level)
    - Ensures data is properly replicated before an ack is received
  - min.insync.replicas = 2 (broker/topic level)
    - Ensures two brokers in ISR at least have the data after an ack
  - retries = MAX_INT (producer level)
    - Ensures transient errors are retried indefinitely
  - max.in.flight.requests.per.connection = 1 (producer level)
- Kafka >= 0.11
  - enable.idempotence = true (producer level) + min.insync.replicas = 2 (broker/topic level)
    - Implied acks = all, retries = MAX_INT, max.in.flight.requests.per.connection = 5 (default)
    - While keeping order guarantees and improving performance
  - Running a "safe producer" might impact throughput and latency

## Message Compression

- Producer usually send data that is text-based (for example: JSON)
  - In this case, it is important to apply compression to the producer
- Compression is enabled at the Producer level and doesn't require any configuration change in the Brokers or in the Consumers
  - "compression.type" can be "none" (default), "gzip", "lz4", or "snappy"
- Compression is more effective the bigger the batch of message being sent to Kafka
- The compressed batch has the following advantage:
  - Much smaller producer request size (compression ratio up to 4x)
  - Faster to transfer data over the network => less latency
  - Better throughput
  - Better disk utilization in Kafka (stored messages on disk are smaller)
- Disadvantages:
  - Producers must commit some CPU cycles to compression
  - Consumers must commit some CPU cycles to decompression
- Overall:
  - Consider testing snappy or lz4 for optimal speed / compression ratio

### Message Compression Recommendations

- Find a compression algorithm that gives you the best performance for your specific data
- Always use compression in production and especially if you have high throughput
- Consider tweaking linger.ms and batch.size to have bigger batches, and therefore more compression and higher throughput

## linger.ms & batch.size

- By default, Kafka tries to send records as soon as possible
  - It will have up to 5 requests in flight, meaning up to 5 messages individually sent at the same time
  - After this, if more messages have to be sent while others are in flight, Kafka is smart and will start batching them while they wait to send them all at once
- This smart batching allows Kafka to increase throughput while maintaining very low latency
- Batches have higher compression ratio so better efficiency
- Controlling the batching mechanism:
  - linger.ms: Number of milliseconds a producer is willing to wait before sending a batch out (default 0)
    - By introducing some lag (for example: linger.ms = 5), we increase the chances of messages being sent together in a batch
    - At rhe expense of introducing a small delay, we can increase throughput, compression and efficiency of our producer
    - If a batch is full (batch.size) before the end of the linger.ms period, it will be sent to Kafka right away
  - batch.size: Maximum number of bytes that will be included in a batch. The default is 16KB.
    - Increasing a batch size to something like 32KB or 64KB can help increase the compression, throughput and efficiency of requests
    - Any message that is bigger than the batch size will not be batched
    - A batch is allocated per partition so make sure that you don't set it to a number that's too high. Otherwise, you'll run waste memory
    - You can monitor the average batch size metric using Kafka Producer Metrics

## Producer Default Partitioner and How Keys are Hashed

- By default, your keys are hashed using the "murmur2" algorithm
- It is most likely preferred to not override the behavior of the partitioner, but it is possible to do so (partitioner.class)
- The formula is: targetPartition = Utils.abs(Utils.mumur2(record.key())) % numPartitions
  - This means the same key will go to the same partition and adding partitions to a topic will completely alter the formula

## max.block.ms & buffer.memory

- If the producer produces faster than the broker can take, the records will be buffered in memory
- buffer.memory = 33554432 (32MB): the size of the send buffer
- That buffer will fill up over time and fill back down when the throughput to the broker increases
- If that buffer if full (32MB), the send() method will start to block (won't return right away)
- max.block.ms = 60000: the time the send() will block until throwing an exception
- Exceptions are thrown when:
  - The producer has filled up its buffer.memory
  - The broker is not accepting any new data
  - 60 seconds has elapsed
- If you hit an exception hit that usually means your brokers are down or overloaded as they can't respond to requests

## Delivery Semantics

### At Most Once

- Offsets are committed as soon as the message batch is received. If the processing goes wrong, the message will be lost (it won't be read again)

### At Least Once

- Offsets are committed after the message is processed. If the processing goes wrong, the message will be read again. This can result in duplicate processing of messages. Make sure your processing is idempotent (i.e. processing the messages again won't impact your systems)

### Exactly Once

- Can be achieved for Kafka -> Kafka workflows using Kafka Streams API. For Kafka -> Sink workflows, use an idempotent consumer.

### Bottomline

For most applications you should use at least once processing and ensure your transformations / processing are idempotent

## Consumer Poll behavior

- Kafka consumers have a "poll" model while many other messaging bus in enterprises have a "push" model
  - This allows consumers to control where in the log they want to consume, how fast, and gives them the ability to replay events
- fetch.min.bytes (default 1):
  - Controls how much data you want to pull at least on each request
  - Helps improving throughput and decreasing request number
  - At the cost of latency
- max.poll.records (default 500):
  - Controls how many records to receive per poll request
  - Increase if your messages are very small and have a lot of available RAM
  - Good to monitor how many records are polled per request
- max.partitions.fetch.bytes (default 1MB):
  - Maximum data returned by the broker per partition
  - If you read from 100 partitions, you'll need a lot of memory (RAM)
- fetch.max.bytes (default 50MB):
  - Maximum data returned for each fetch request (covers multiple partitions)
  - The consumer performs multiple fetches in parallel
- Change these settings only if your consumer maxes out on throughput already

## Consumer Offset Commit Strategies

There are two most common patterns for committing offsets in a consumer application:

- (easy) enable.auto.commit =  true & synchronous processing of batches
  - With auto-commit, offsets will be committed automatically for you at regular interval (auto.commit.interval.ms = 5000 by default) every time you call .poll()
  - If you don't use synchronous processing, you will be in "at-most-once" behavior because offsets will be committed before your data is processed

```java
  while (true) {
    List<Records> batch = consumer.poll(Duration.ofMillis(100));
    doSomethingSynchronous(batch)
  }
```

- enable.auto.commit =  false & synchronous processing of batches
  - You control when you commit offsets and what's the condition for committing them
  - Ex: accumulating records into a buffer and then flushing the buffer to a database + committing offsets then

```java
  while (true) {
    batch += consumer.poll(Duration.ofMillis(100))
    if (isReady(batch)) {
      doSomethingSynchronous(batch)
      consumer.commitSync();
    }
  }
```

- (medium) enable.auto.commit = false & manual commit of offsets

## Consumer Offset Reset Behavior

- A consumer is expected to read from a log continuously
- If Kafka has a retention of 7 days, and your consumer is down for more than 7 days, the offsets are "invalid"
- The behavior for the consumer is to then use:
  - auto.offset.reset = latest: read from the end of the log
  - auto.offset.reset = earliest: read from the start of the log
  - auto.offset.reset = none: throw exception if no offset is found
- Additionally, consumer offsets can be lost:
  - If a consumer hasn't read new data in 1 day (Kafka < 2.0)
  - If a consumer hasn't read new data in 7 days (Kafka >= 2.0)
- This can be controlled by the broker setting offset.retention.minutes

### Replaying data for Consumers

- To replay data for a consumer group:
  - Take all the consumers from a specific group down
  - Use "kafka-consumer-groups" command to set offset to what you want
  - Restart consumers

### Bottomline

- Set proper data retention period & offset retention period
- Ensure the auto offset reset behavior is the one you expect / want
- Use replay capability in case of unexpected behavior

## Controlling Consumer Liveliness

- Consumers in a Group talk to a Consumer Groups Coordinator
- To detect consumers that are "down", there is a "heartbeat" mechanism and a "poll" mechanism
- To avoid issues, consumers are encouraged to process data fast and poll often

### Consumer Heartbeat Thread

- session.timeout.ms (default 10 seconds):
  - Heartbeats are sent periodically to the broker
  - If not heartbeat is sent during that period, the consumer is considered dead
  - Set lower for faster consumer rebalances
- heartbeat.interval.ms (default 3 seconds):
  - How often to send heartbeats
  - Usually set to 1/3rd of sessions.timeout.ms
- Takeaway: this mechanism is used to detect a consumer application being down

### Consumer Poll Thread

- max.poll.interval.ms (default 5 minutes):
  - Maximum amount of time between two .poll() calls before declaring the consumer dead
- This is particularly relevant for Big Data frameworks like Spark in case the processing takes time

## Kafka Connect Introduction

### Why Kafka Connect and Streams

- Four Common Kafka Use Case:
  - Source => Kafka (Producer API) (Kafka Connect Source)
  - Kafka => Kafka (Consumer, Producer API) (Kafka Streams)
  - Kafka => Sink (Consumer API) (Kafka Connect Sink)
  - Kafka => App (Consumer API)
- Simplify and improve getting data in and out of Kafka
- Simplify transforming data within Kafka without relying on external libs

#### Why Kafka Connect

- Programmers always want to import data from the same sources:
  - Databases, JDBC, Couchbase, GoldenGate, SAP HANA, Blockchain, Cassandra, DynamoDB, FTP, IoT, MongoDB, MQTT, RethinkDB, Salesforce, Solr, SQS, Twitter, etc...
- Programmers always want to store data in the same sinks:
  - S3, ElasticSearch, HDFS, JDBC, SAP HANA, DocumentDB, Cassandra, DynamoDB, HBase, MongoDB, Redis, Solr, Splunk, Twitter
- It is tough to achieve Fault Tolerance, Idempotence, Distribution, Ordering

#### Kafka Connect - High Level

- Source Connectors get data from Common Data Sources
- Sink Connectors publish data in Common Data Sources
- Make it easy for inexperienced dev to quickly get their data reliably into Kafka
- Part of your ETL pipeline
- Scaling made easy from small pipelines to company-wide pipelines
- Reusable code

## What is Kafka Streams?

- Easy data processing and transformation library within Kafka
- Standard Java application
- No need to create a separate cluster
- Highly scalable, elastic and fault tolerant
- Exactly one capability:
  - One record at a time processing (no batching)
- Works for any application size

## The need for a schema registry

- Kafka takes bytes as an input and publishes them
- No data verification
- We need data to be self-describable
- We need to be able to evolve data without breaking downstream consumers
- What if Kafka brokers were verifying the messages they receive?
  - It would break what makes Kafka so good:
    - Kafka doesn't parse or even read your data (no CPU usage)
    - Kafka takes bytes as an input without even loading them into memory (that's called zero copy)
    - Kafka distributes bytes
    - As far as Kafka is concerned, it doesn't know if your data is an integer, a string, etc.
- The schema registry has to be a separate components
- Producers and Consumers need to be able to talk to it
- The schema registry must be able to reject bad data
- A common data format must be agreed upon
  - It needs to support schemas
  - It needs to support evolution
  - It needs to be lightweight

### Confluent Schema Registry Purpose

- Store and retrieve schemas for Producers/Consumers
- Enforce backward/forward/full compatibility on topics
- Decrease the size of the payload of data sent to Kafka

### Schema Registry Gotchas

- You need to:
  - Set it up well
  - Make sure it's highly available
  - Partially change the producer and consumer code

## Partition Count and Replication Factor

- The two most important parameters when creating a topic
- They impact performance and durability of the system overall
- If the partition count increases during a topic lifecycle, you will break your keys ordering guarantees
- If the replication factor increases during a topic lifecycle, you put more pressure on your cluster, which can lead to unexpected performance decrease

### Partition Count

- Each partition can handle a throughput of a few MB/s
- More partitions implies:
  - Better parallelism, better throughput
  - Ability to run more consumers in a group to scale
  - Ability to leverage more brokers if you have a large cluster
  - More elections to perform for Zookeeper (Con)
  - More files open on Kafka (Con)
- Guidelines:
  - Partitions per topic: MILLION DOLLAR QUESTION
    - Small cluster (< 6 brokers): 2 x number of brokers
    - Big cluster (> 12 brokers): 1 x number of brokers
    - Adjust for number of consumers you need to run in parallel at peak throughput
    - Adjust for producer throughput (increase if super-high throughput or projected increase in the next 2 years)
  - Test: Every Kafka cluster will have different performance
  - Don't create a topic with 1000 partitions

### Replication Factor

- Should be at least 2, usually 3, maximum 4
- The higher the replication factor (N):
  - Better resilience of your system (N - 1 brokers can fail)
  - More replication (higher latency if acks=all) (con)
  - More disk space on your system (50% more if replication factor is 3 instead of 4)
- Guidelines:
  - Set it to 3 to get started (you must have at least 3 brokers for that)
  - If replication performance is an issue, get a better broker instead of less replication factor
  - Never set it to 1 in production

### Clusters

- It is accepted that a broker should not hold more than 2000 to 4000 partitions (across all topics in that broker)
- Additionally, a Kafka cluster should have a maximum of 20,000 partitions across all brokers
- The reason is that in the case of brokers going down, Zookeeper needs to perform a lot of leader elections
- If you need more partitions in your cluster, add brokers instead
- If you need more than 20,000 partitions in your cluster, follow the Netflix model and create more Kafka clusters
- Overall, you don't need a topic with 1000 partitions to achieve high throughput. Start at a reasonable number and test the performance

## Big Data Ingestion

- It is common to have "generic" connectors or solutions to offload data from Kafka to HDFS, Amazon S3, and ElasticSearch
- It is also very common to have Kafka serve a "speed layer" for real time applications while having a "slow layer" which helps with data ingestion into stores for later analytics
- Kafka as a front to Big Data Ingestion is a common pattern in Big Data to provide an "ingestion buffer" in front of some stores

## Kafka Cluster Setup High Level Architecture

- You want multiple brokers in different data centers (racks) to distribute your load. You also want a cluster of at least 3 zookeeper

### Kafka Cluster Setup Gotchas

- It's not easy to set up a cluster
- You want to isolate each Zookeeper & Broker on separate servers
- Monitoring needs to be implemented
- You need a really good Kafka Admin
- Alternative: many different "Kafka as a Service" offerings on the web
  - No operational burdens (updates, monitoring, setup, etc...)

### Kafka Monitoring and Operations

- Kafka exposes metrics through JMX
- These metrics are highly important for monitoring Kafka and ensuring the systems are behaving correctly under load
- Common places to hose the Kafka metrics:
  - ELK (ElasticSearch + Kibana)
  - Datadog
  - NewRelic
  - Confluent Control Center
  - Prometheus
  - etc...
- Some of the most important metrics are:
  - Under Replicated Partitions: Number of partitions having problems with the ISR (in-sync replicas). May indicate a high load on the system
  - Request Handlers: Utilization of threads for IO, network, etc... overall utilization of an Apache Kafka broker
  - Request Timing: How long it takes to reply to requests. Lower is better as latency will be improved
- Kafka Operations team must be able to perform the following tasks:
  - Rolling Restart of Brokers
  - Updating Configurations
  - Rebalancing Partitions
  - Increasing Replication Factor
  - Adding a Broker
  - Removing a Broker
  - Upgrading a Kafka Cluster with zero downtime

### The need for encryption, authentication & authorization in Kafka

- Currently, any client can access your Kafka cluster (authentication)
- The clients can publish / consume any topic data (authorization)
- All the data being sent is fully visible on the network (encryption)
- Someone could intercept data being sent
- Someone could publish bad data / steal data
- Someone could delete topics

#### Encryption in Kafka

- Encryption in Kafka ensures that the data exchanged between clients and brokers are secret to routers on the way
  - This is similar in concept to an https website

#### Authentication in Kafka

- Authentication in Kafka ensures that only clients that can prove their identity can connect to our Kafka Cluster
  - This is similar in concept to a login (username / password)
- Authentication in Kafka can take a few forms
  - SSL Authentication: clients authenticate to Kafka using SSL certificates
  - SASL Authentication:
    - PLAIN: clients authenticate using username / password (weak - easy to set up)
    - Kerberos: e.g. Microsoft Active Directory (strong - hard to set up)
    - SCRAM: username / password (strong - medium to set up)

#### Authorization in Kafka

- Once a client is authenticated, Kafka can verify its identity
- It still needs to be combined with authorization so that Kafka knows that
  - "User Alice can view topic finance"
  - "User Bob cannot view topic trucks"
- ACL (Access Control Lists) have to be maintained by admin to onboard new users

## Kafka Multi Cluster + Replication

- Kafka can only operate well in a single region
- Therefore, it is very common for enterprises to have Kafka clusters across the world with some level of replication between them
- A replication application at its core is just a consumer + a producer
- There are different tools to perform it:
  - Mirror Maker (MM) - open source tool that ships with Kafka
  - Netflix uses Flink - they wrote their own application
  - Uber uses uReplicator - addresses performance and operation issues with MM
  - Comcast has their own open source Kafka Connect Source
  - Confluent has their own Kafka Connect Source (paid)
- There are two designs for cluster replication:
  - Active => Passive
    - You want to have an aggregation cluster (e.g. for analytics)
    - You want to create some form of disaster recovery strategy (hard)
    - Cloud Migration (from on-premise cluster to cloud cluster)
  - Active => Active
    - You have a global application
    - You have a global dataset
- Replicating doesn't preserve offsets, just data

## Advanced Kafka - Topic Configurations

### Why should I care about topic config?

- Brokers have defaults for all the topic configuration parameters
  - These parameters impact __performance__ and __topic behavior__
- Some topics may need different values from the defaults
  - Replication Factor
  - Number of Partitions
  - Message size
  - Compression level
  - Log Cleanup Policy
  - Min Insync Replicas
  - Other configurations

## Partitions and Segments

- Topics are made of partitions
- Partitions are made of segments (files)
- Only one segment is ACTIVE (the one data is being written to)
- Two segment settings:
  - log.segment.bytes: the max size of a single segment in bytes
  - log.segment.ms: the time Kafka will wait before committing the segment if not full

### Segments and Indexes

- Segments come with two indexes (files):
  - An offset to position index: allows Kafka to find where to read a message
  - A timestamp to offset index: allows Kafka to find messages with a timestamp
- Therefore, Kafka knows where to find data in constant time

### Segments: Why should I care?

- A smaller log.segment.bytes (size, default: 1GB) means:
  - More segments per partition
  - Log Compaction happens more often
  - Kafka has to keep more files opened (Con)
- Ask yourself: How fast will I have new segments based on throughput?
- A smaller log.segment.ms (time, default: 1 week) means:
  - You set a max frequency for log compaction (more frequent triggers)
- Ask yourself: How often do I need log compaction to happen?

## Log Cleanup Policies

- Many Kafka clusters make data expire according to a policy

Policy 1: log.cleanup.policy = delete (Kafka default for all user topics)

- Delete based on age of data (default is a week)
- Delete based on max size of log (default is -1 == infinite)

Policy 2: log.cleanup.policy = compact (Kafka default for topic __consumer_offsets)

- Delete based on keys of your messages
- Will delete old duplicate keys after the active segment is committed
- Infinite time and space retention

### Log Cleanup: Why and When?

- Deleting data from Kafka allows you to:
  - Control the size of the data on the disk and delete obsolete data
  - Overall: limit maintenance work on the Kafka cluster
- How often does log cleanup happen?
  - Log cleanup happens on your partition segments
  - Smaller / More segments means that log cleanup will happen more often
  - Log cleanup shouldn't happen too often => takes CPU and RAM resources
  - The cleaner checks for work every 15 seconds (log.cleaner.backoff.ms)

### Log Cleanup Policy: Delete

- log.retention.hours:
  - Number of hours to keep data for (default is 168 - one week)
  - Higher number means more disk space
  - Lower number means that less data is retained (if your consumers are down for too long, they can miss data)
- log.retention.bytes:
  - Max size in bytes for each partition (default is - 1 == infinite)
  - Useful to keep the size of a log under a threshold
- Use cases - two common pair of options:
  - One week of retention:
    - log.retention.hours = 168 and log.retention.bytes = -1
  - Infinite time retention bounded by 500MB:
    - log.retention.hours = 17520 and log.retention.bytes = 524288000

## Log Compaction

- Configured by (log.cleanup.policy = compact):
  - segment.ms (default: 7 days): Max amount of time to wait to close active segment
  - segment.bytes (default: 1GB): Max size of a segment
  - min.compaction.lag.ms (default: 0): How long to wait before a message can be compacted
  - delete.retention.ms (default: 24 hours): How long to wait before deleting data marked for compaction
  - min.cleanable.dirty.ratio (default 0.5): Higher => less, more efficient cleaning. Lower => opposite

### Log Cleanup Policy: Compact

- Log compaction ensures that your log contains at least the last known value for a specific key within a partition
- Very useful if we just require a snapshot instead of full history (such as for a data table in a database)
- The idea is we only keep the latest "update" for a key in our log

#### Log Compaction Guarantees

- Any consumer that is reading from the tail of a log (most current data) will still see all the messages sent to the topic
- Ordering of messages is kept and log compaction only removes some messages, but does not reorder them
- The offset of a message is immutable (it never changes). Offsets are skipped if a message is missing
- Deleted records can still be seen by consumers for a period of delete.retention.ms (default is 24 hours)

#### Log Compaction Myth Busting

- It doesn't prevent you from pushing duplicate data to Kafka
  - De-duplication is done after a segment is committed
  - Your consumers will still read from tail as soon as the data arrives
- It doesn't prevent you from reading duplicate data from Kafka
  - Same points as above
- Log Compaction can fail from time to time
  - It is an optimization and the compaction thread might crash
  - Make sure you assign enough memory to it and that it gets triggered
  - Restart Kafka if log compaction is broken (bug)
- You can't trigger Log Compaction using an API call

## min.insync.replicas

- acks = all must be used in conjunction with max.insync.replicas
- min.insync.replicas can be set at the broker or topic level (override)
- min.insync.replicas = 2 implies that at least 2 brokers that are ISR (including leader) must respond that they have the data
- That means if you use replication.factor = 3, min.insync = 2, acks = all, you can only tolerate 1 broker going down, otherwise the producer will receive an exception on send

## unclean.leader.election

- If all your In Sync Replicas die (but you still have out of sync replicas up), you have the following option:
  - Wait for an ISR to come back online (default)
  - Enable unclean.leader.election = true and start producing to non ISR partitions
- If you enable unclean.leader.election = true, you improve availability, but you will lose data because other messages on the ISR will be discarded
- Don't use before understanding it
- Use cases include: metrics collection, log collection, and other cases where data loss is somewhat acceptable at the tradeoff of availability

## Understanding Communication between Client and Kafka

- Advertised listeners is the most important setting in Kafka
- If your clients are on your private network, use either:
  - The internal private IP
  - The internal private DNS hostname
- If your clients are on a public network, set either:
  - The external public IP
  - The external public DNS hostname pointing to the public IP
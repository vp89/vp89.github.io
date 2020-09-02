---
layout: post
title:  "SQS FIFO Queues, not so fast"
date:   2020-08-23 00:00:00 -0500
categories: programming
mermaid: true
---

My TL;DR take on these is that they are pretty weak-sauce. They make it a bit easier for you to sort of do things properly without expending too much effort, but if you scratch at the surface a little you can see that the FIFO queue is not a complete solution both in terms of ordering and de-duplication. We will get to that later.

One more obvious issue is that it's pretty easy to hit the default rate limits on these. Amazon doesn't document what the limits can be bumped to, which makes it difficult for me to provide comprehensive commentary on the suitability of these for very high scale workloads. [This post](https://softwaremill.com/amazon-sqs-performance-latency/) shows that a presumably "stock" standard queue handles at least 9k message batches per second which is 30x more than the "stock" FIFO queue publish rate limits.

From an architectural perspective, a queue that can practically handle whatever you throw at it is a huge strategic advantage. The more likely the queue system needs to shed load, the more likely we have to deal with that in the thing that publishes to the queue. You could continue to shed load at the publisher level and drop things on the floor, but if you are already thinking about FIFO queue semantics, that is likely not an option. One option is to have a "spillover" standard SQS queue that you publish to when the FIFO queue is overloaded. Any time you are spilling over you would lose the ordering semantics that the FIFO queue supposedly provides but at least you wouldn't need to drop data.

Another option would be to have the publisher queue the messages to it's own internal memory queue as a form of back-pressure. This would allow you to preserve proper ordering (if the creation of data is partitioned by logical groups that ought to be ordered, that is) but it does have some risks that you could overload the memory of the publisher. You also risk data loss if the publisher process crashes. In that case, you would lose all those in-flight messages it had in it's internal memory queue.

#### Ordering

FIFO queues are more limited in their throughput because they provide as the name implies "first-in, first-out" queue semantics. On the surface, this sounds great. Who doesn't want "correct ordering", right? Unfortunately, it's not quite so simple. The order in which data is consumed off the queue doesn't actually provide any comprehensive ordering guarantees for a couple of reasons:

- The FIFO queue can't guarantee that data is published to the queue in-order (upstream)
- The FIFO queue can't guarantee that data consumed from the queue is *processed* in-order (downstream)

The upstream publisher is likely to be some HTTP service that runs on N hosts. Requests to it are load balanced in a round-robin fashion. Given a user's transactions T1, T2, T3 ... it's quite feasible that T2 could be published to the queue before T1, for a million different reasons.

<div class="mermaid">
sequenceDiagram
    participant P1
    participant P2
    participant FIFO Queue
    participant C1
    P1->>FIFO Queue: T1 (failed)
    P2->>FIFO Queue: T2
    P1->>FIFO Queue: T1 (retry)
    C1->>FIFO Queue: Consume
    FIFO Queue->>C1: T2
    C1->>FIFO Queue: Consume
    FIFO Queue->>C1: T1
</div>

In this example, the FIFO queue did it's job but the data was not processed in-order.

The downstream consumer is also likely to be some service that runs on N hosts. Again, given a user's transactions T1, T2, T3 and processing steps P1, P2, P3 ... it's quite feasible that T1 and T2 could be consumed at the same time, P1 processing for T1 is blocked or fails which leads to T2 being completely processed before T1.

<div class="mermaid">
sequenceDiagram
    participant FIFO Queue
    participant C1
    participant C2
    participant Downstream 1
    participant Downstream 2
    C1->>FIFO Queue: Consume
    FIFO Queue->>C1: T1
    C2->>FIFO Queue: Consume
    FIFO Queue->>C2: T2
    C1->>Downstream 1:Process T1 (fail)
    C2->>Downstream 1:Process T2
    Downstream 1->>Downstream 2:Process T2
    C1->>Downstream 1:Process T1 (retry)
    Downstream 1->>Downstream 2:Process T1
</div>

Both these examples demonstrate that the queue alone cannot give meaningful ordering guarantees. To have proper in-order processing, we must also architect the system to ensure that both publishing and consuming happens in-order based on the way the data can be logically partitioned. All the way up and all the way down the stack. If you are not doing that, then you are paying all the costs of the FIFO queue without getting any of it's benefits.

#### Exactly-once processing

There are three types of *processing* guarantees that a system could provide for the data that flows through it.

- At most once
- At least once
- Exactly-once

Individual components of a system can also have their own *delivery* guarantees:

- At most once
- At least once

We can't guarantee that anything is delivered exactly-once, because networks can fail. We can only guarantee that things are *processed* exactly-once. This is done by tracking state of what has been processed before, using some unique identifier for a given piece of data. The FIFO queue promises exactly-once semantics, but if you read the fine print this is only on a 5-minute interval.

<div class="mermaid">
sequenceDiagram
    participant P1
    participant SQS
    participant DedupeStore
    participant FIFO Queue
    P1->>SQS: Publish T1
    SQS->>DedupeStore: T1 exists?
    DedupeStore->>SQS: false
    SQS->>FIFO Queue: T1
    SQS->>P1: ACK (fails)
    P1->>SQS: Publish T1
    SQS->>DedupeStore: T1 exists?
    DedupeStore->>SQS: true
    SQS->>P1: ACK
</div>

This is a typical example, and in this case the FIFO queue is providing some "exactly-once" guarantees. Whether T1 is processed exactly-once still depends on what happens downstream, but at least the FIFO Queue bit worked as expected. What happens though, if we had a failure downstream and want to re-process data from P1 from a given period of time?

<div class="mermaid">
sequenceDiagram
    participant P1
    participant SQS
    participant DedupeStore
    participant FIFO Queue
    participant C1
    participant Downstream 1
    P1->>SQS: Publish T1,T2
    SQS->>DedupeStore: T1,T2 exist?
    DedupeStore->>SQS: false
    SQS->>FIFO Queue: T1,T2
    SQS->>P1: ACK
    C1->>FIFO Queue: Consume
    FIFO Queue->>C1: T1
    C1->>Downstream 1: Process T1
    Downstream 1->>C1: Success
    C1->>FIFO Queue: Consume
    FIFO Queue->>C1: T2
    C1->>Downstream 1: Process T2
    Downstream 1->>C1: Error
    Note over P1: > 5 minutes have passed
    P1->>SQS: Republish T1,T2
    SQS->>DedupeStore: T1,T2 exist?
    DedupeStore->>SQS: false
    SQS->>FIFO Queue: T1,T2
    SQS->>P1: ACK
    C1->>FIFO Queue: Consume
    FIFO Queue->>C1: T1
    C1->>Downstream 1: Process T1
    Downstream 1->>C1: Success
    C1->>FIFO Queue: Consume
    FIFO Queue->>C1: T2
    C1->>Downstream 1: Process T2
    Downstream 1->>C1: Success
</div>

In this more complex example, we had a couple of transactions that went through the FIFO queue. We later (> 5 minutes) discover that there was a Production incident that led to errors processing some of these transactions. We want to replay all the transactions that would have been published during a given period of time to ensure that our system doesn't have any missing data. The FIFO queue is exactly-once so no problem right? Well unfortunately not. The FIFO queue only provides exactly-once guarantees within a 5 minute interval. This makes sense from their perspective, as I mentioned earlier, to provide exactly-once processing guarantees you have to track state of everything that has passed through the system. It would not be scalable for SQS to do this for all of it's FIFO queues so they don't. The real solution here is that things downstream of the queue must track state of all the input that has been submitted to them so they can provide the exactly-once guarantees.

The SQS FAQ documentation is a little disingenuous on this topic because they mention that if you use Standard Queues you must design your system to be idempotent which implies that if you use FIFO Queues you don't. But that is not actually true. You will likely have failures where it would be very desirable to be able to "replay" input, in which case you must *always* build idempotency in at the system level because the FIFO Queue cannot provide it beyond a 5 minute interval.

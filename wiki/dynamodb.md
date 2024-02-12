---
title: DynamoDB
layout: wiki
wiki: true
status: in progress
create_date: 2024-01-27 14:17:00 -0500
update_date: 2024-01-27 14:17:00 -0500
---

- In 2019, I was [skeptical](https://vincepergolizzi.com/programming/2019/02/19/no-sql-transitory-technology.html) of DynamoDB but the features and integrations with other AWS services that have been added since have made it a very compelling option for OLTP workloads

- When load testing with DynamoDB, you need good up front understanding of how tables scale otherwise you will waste time and potentially draw the wrong conclusions
    - Testing against the same table continuously may result in the wrong conclusions. When you deploy to Prod for the first time, the table may not be split in the same way or support the same peaks, if using on demand capacity
    - Longer tests will provide better signal as the partition splitting tends to require sustained peaks
    - Testing shows that very short and spiky workloads won't trigger a partition split and thus constantly be throttled
    - The test should reproduce how load is spread across partitions as this also plays a big factor
    - Dynamo provides good features to help cope with unevenly distributed load, but those same features can also make it harder to reason up front about how the table will behave. Understandably, no telemetry is surfaced to the user about when partition splits happen, when adaptive or burst capacity kicks in, when items are moved etc ...
    - SDKs tend to have retry policies baked in so that internal server errors or occassional throttles aren't felt by the application

- Best practice is to start with on-demand until you have good data about your workload which you can use to move to provisioned capacity and increase utilization. On-demand mode will scale up to 2x the previous peak (with an upto ~30 min lag). If the workload is naturally asynchronous, you can throttle in your service code and gradually increase throughput as needed. New on-demand tables start with 4 partitions. If you need more capacity right away but still want on-demand mode, you can deploy the table in provisioned mode for a higher amount of WCU / RCU than the default (4k/12k) and then switch it to on-demand mode
- During a partition split you can expect to see ~1 second of "internal errors" as there must necessarily be a hard cutover of leader nodes

- The [streams](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/streamsmain.html) feature enables an elegant solution for the [dual write problem](https://www.confluent.io/blog/using-logs-to-build-a-solid-data-infrastructure-or-why-dual-writes-are-a-bad-idea/):
    - <img src="/images/dynamo-streams-outbox.drawio.png" />
    - Can retain testability and encapsulation by writing fully formed events to an `outboxJson` attribute and having a simple pipes transform that "renders" it's value
    - To preserve atomicity, this attribute can be appended to the item already being written to or you can use TransactWriteItems
    - DynamoDB streams supports up to 2 consumers, you can have 1 pipe for your "outbox" and for your data warehousing
    - If you have many distinct events and consumers, they can all be serviced with a single pipe by having it target an SNS topic with filtered queue subscriptions on a common `eventType` field. Another option is to leverage EventBridge event bus but the cost is higher
    - EventBridge pipes cost $0.4/MM *records* after filtering. A record is up to 64kb of data and it's able to consume batches of changes from the stream shards. It's unclear whether there is a charge for a "pipe" to read from the stream, there isn't when Lambda polls the stream so presumably the same would apply here

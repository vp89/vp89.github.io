---
title: DynamoDB
layout: wiki
wiki: true
status: in progress
create_date: 2024-01-27 14:17:00 -0500
update_date: 2024-01-27 14:17:00 -0500
---

- In 2019, I was [skeptical](https://vincepergolizzi.com/programming/2019/02/19/no-sql-transitory-technology.html) of DynamoDB but the features and integrations with other AWS services that have been added since then have made it a very compelling option for OLTP workloads

- The [streams](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/streamsmain.html) feature enables a very elegant solution for when you want your service to **atomically** affect state in it's own database *and* send 1 or more messages to other services. Also known as the [dual write problem](https://www.confluent.io/blog/using-logs-to-build-a-solid-data-infrastructure-or-why-dual-writes-are-a-bad-idea/):
    - <img src="/images/dynamo-streams-outbox.drawio.png" />
    - Different variants possible on the above solution depending on needs:
        - If the message can be entirely reconstituted from an item modification, then you can simply consume the stream. Otherwise you can do an outbox pattern using either BatchWriteItem (non-atomic) or TransactWriteItems. The outbox pattern can cost between 2-4x more than just a single item modification so consider enriching the modified item with fields needed for the event
        - If only simple transform is needed and you can forward the message to SQS/SNS/Kinesis, then you can use an EventBridge pipe. If more complex transform is needed or you need to forward the message to a RabbitMQ or Pulsar, can deploy a Lambda. In both cases can apply a filter to the stream to only consume changes that should result in a message downstream
        - EventBridge is $0.4/MM *records*, a record is up to 64kb of data and it's able to consume batches of changes from the stream shards
    - This pattern means your service only takes a hard dependency on DynamoDB, if the message broker is down then the pipeline will automatically send messages once it's back up. Avoids laborious remediation efforts when messaging infra goes down. This is a very common failure mode in industry

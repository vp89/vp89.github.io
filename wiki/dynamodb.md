---
title: DynamoDB
layout: wiki
wiki: true
status: in progress
create_date: 2024-01-27 14:17:00 -0500
update_date: 2024-01-27 14:17:00 -0500
---

- The [streams](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/streamsmain.html) feature enables a very elegant solution for when you want your service to **atomically** affect state in it's own database *and* send 1 or more messages to other services. Also known as the [dual write problem](https://www.confluent.io/blog/using-logs-to-build-a-solid-data-infrastructure-or-why-dual-writes-are-a-bad-idea/):
    - <img src="/images/dynamo-streams-outbox.drawio.png" />
    - Different variants possible on the above solution depending on needs:
        - If the message can be entirely reconstituted from an item modification, then you can simply consume the stream. Otherwise you can do an outbox pattern using either BatchWriteItem (non-atomic) or TransactWriteItems. The outbox pattern can cost between 2-4x more than just a single item modification so consider enriching the modified item with fields needed for the event
        - If only simple transform is needed and you can forward the message to SQS/SNS/Kinesis, then you can use an EventBridge pipe. If more complex transform is needed or you need to forward the message to a RabbitMQ or Pulsar, can deploy a Lambda. In both cases can apply a filter to the stream to only consume changes that should result in a message downstream
        - EventBridge is $1/MM events, Lambda can be cheaper or more expensive depending on what you're doing
    - This pattern means your service only takes a hard dependency on DynamoDB, if the message broker is down then the pipeline will automatically send messages once it's back up. Avoids laborious remediation efforts when messaging infra goes down. This is a very common failure mode in industry

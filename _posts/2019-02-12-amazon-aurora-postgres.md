---
layout: post
title:  "Amazon Aurora Postgres - First look"
date:   2019-02-12 00:00:00 -0000
categories: programming
---

### What's the big idea?

Aurora is a closed-source fork of Postgres embedded into the RDS infrastructure. Amazon took the existing engine and then rewrote certain components to better leverage the properties of their infrastructure.

#### Efficient durable writes

Traditional RDBMS such as Postgres are written to be self-sufficient in terms of providing durability guarantees. As such, for each write they perform multiple synchronous operations on disk. Typically they first write to a log, then they modify the pages. For crash recovery, entire copies of pages can be written to the log in the event of a crash mid-write.

Aurora reduces the amount of synchronous work to just 1 log write. It pushes that log write to 6 storage nodes and the transaction commits as soon as it's written to at least 4 of those nodes.

Logs are then applied to your pages using a background process. The storage nodes keep a hot-log of writes which have not yet been coalesced with the existing pages.

This reduction of synchronous work required allows them to claim much better and more consistent performance. I will say that I think their claims are a bit disingenuous because they say that Aurora is 3x faster than Postgres. But the caveat "with 6 node, 3 AZ replication and on RDS" is not explicitly stated and up to the consumer to figure out.

#### Better RAM utilization

Because the database process does not use it's local filesystem, they are able to allocate all the memory that would normally be used for the filesystem cache to the database cache. There is a lot of duplication that happens when you run both a database and a filesystem cache, so this allows you to fully utilize the RAM you are paying for. They also claim that the database cache is de-coupled from the database process entirely, so if the process dies or you need to restart it you don't lose your cached data.

#### Fast initialization of read replicas

Steady state read replicas can be spun up without needing to copy data. The new process just points to the storage node. Currently up to 15 replicas are supported and AWS provides a "database endpoint" so that you can access them as 1 logical instance.

#### On-demand instances

Yet another advantage of decoupling the storage layer is that they can provide you with on-demand instances, much like a lot of the MPP analytics systems currently do such as Snowflake or Redshift. This allows you to pay for just the instance time you need which could save you money if your system has large periods of the day with low activity.

### Pricing Comparison

#### CPU

I chose the `db.r4.16xlarge` instance for comparison which is the strongest instance you can rent that is commonly available across these 3 different types:

|Type|CPU per hour
|---|---
|RDS Aurora Postgres|$9.28
|RDS Postgres|$8.00
|RDS SQL Server|$24.32

As you can see, you get all the great features and improved performance for just 16% more. It is also still substantially cheaper than other closed source databases such as SQL Server.

I did not mention storage/IO because:
- For storage at rest the cost differences are marginal across the 3 types
- Aurora's IO cost structure is based on $ per 1M IOPS as opposed to provisioned IOPS per month for the other types

Charging per 1M IOPS implies that everyone is guaranteed a certain service level. I would argue that this is better "in theory" provided that the guarantee and the rate it's delivered are sufficient. It's better because I can affect my costs by just changing my IO pattern and don't need to do as much capacity planning as you would with a provisioned IOPS model.

### Drawbacks

There is potential for incompatibility with the open-source version. This could force you into an "accept the vendor lock-in or migrate" decision. You also have to wait for Amazon to merge in patches from upstream. I think Amazon is initially incentivized to maintain compatibility, but if they capture significant market share they may not feel as compelled to do that.

You are taking a risk on a new database product. Even though it's not written from scratch, you are still somewhat trusting Amazon to deliver on something which is not their core competency. They don't have a long track record in this space. Another risk is the lack of an expert community around this product. Historically DBAs have been the most prolific in producing educational content for database systems. Are there a lot of DBAs managing and learning about RDS Aurora? The product aims to make them redundant. This is a general long-term trend I am worried about with the growing prevalence of managed services and maybe I will do a separate post on this in the future.

Currently Aurora Postgres has quite limited instance selection. The cheapest machine is ~$200/month just for the CPU. They might benefit by making it more accessible for experimental/training use. Aurora MySQL does have an instance that is ~$30/month so this could be coming to Postgres soon.

### Verdict

Overall, based on initial investigation I would suggest that if you're going to do relational on AWS and your system demands high levels of availability then it makes a lot of sense to use it.

With something as complex as a database, cloud providers are always going to be able to provide a better product with a fork that is custom tailored to suit their environment. So this trade-off is a common one that we will have to make going forwards.

The concerns about vendor lock-in are valid and this would become something you would want to monitor over time. I imagine though that it would be difficult for a customer to know just how much the Aurora fork has drifted from upstream, especially with regards to bug fixes. One thing the Aurora team could do is post a regular release blog to reassure customers that they are constantly trying to stay in sync with upstream changes.

You could create a process where you periodically deploy and test your system on the open-source version. If you notice that they start becoming incompatible, or there is some killer new feature that hasn't made its way into Aurora then you could always migrate back to the open source RDS.

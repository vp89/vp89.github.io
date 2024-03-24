---
title: Microservices
layout: wiki
wiki: true
status: in progress
create_date: 2024-02-12 08:52:00 -0500
update_date: 2024-02-12 08:52:00 -0500
---

- A common mistake is splitting up different workloads within a given "bounded context" into multiple logical services. Typically this happens when there are both synchronous and asynchronous workloads involved. The intent behind this is to isolate these workloads into their own compute resources, which *is* often justified once you reach a certain level of scale. The downside however is that you have now split your codebase into separate projects that most probably still share a database, common data structures and business logic. This can add unnecessary complexity and friction around making changes to common elements and generally results in "architecture bloat" when this tactic is commonly used. Splitting workloads into their own resources is a Production specific implementation detail, so it should not need to leak into how the codebase is structured. The same workload isolation can be achieved without these downsides by deploying a single logical service to multiple physical deployments, each configured to only handle a subset of the workloads. This can be easily achieved with dedicated EC2 auto scaling groups, Kubernetes deployments etc ...

    - <img src="/images/disparate-service-workloads.drawio.png" />

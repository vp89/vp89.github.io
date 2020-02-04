---
layout: post
title:  "Local vs Distributed caching"
date:   2020-02-03 00:00:00 -0000
categories: programming
---

I've been thinking a lot about caching recently, after reading this [great write-up](https://nickcraver.com/blog/2019/08/06/stack-overflow-how-we-do-app-caching/) from Nick Craver on how they do it over at StackOverflow.

One thing that struck me is how much faster a lookup against local memory is compared to one against a distributed cache. Assuming you can do that Redis lookup inside of .5ms, that is upto 5000x faster. From the performance side, it's a no brainer. Local cache all the things! But then my pessimist personality kicks in and I started thinking about what the downsides are of doing so.

Distributed caching introduces *inconsistency* as a failure state between what data our source of truth holds (the database) and what data our code has access to. Local caching introduces a much more insidious version of this failure state, where the inconsistency can happen across hosts rather than across layers of our stack. This is IMO a much more significant and undesirable type of inconsistency. "Layer-level" inconsistency is actually fairly easy to debug and deal with. My code says "a", the database says "b". We purge the entry in the distributed cache and the problem is solved. Because the inconsistency is global, it is very easy to reason about which users/requests would have been affected. This makes potential remediation a lot easier, as it's just based on a window of time between when the database was updated and when you purge the stale cache entries.

Host-level inconsistency by definition only happens some % of the time, based on the number of hosts that fell out of sync vs the ones that didn't. This produces an inconsistent experience for your users, where they can hit your API/site 10 times and get an incorrect result 1 or 2 those times for example. Because it doesn't surface as easily, it may go undetected and unresolved for longer. When we do identify and try to fix it, it's quite difficult to know who would have been affected unless we are logging every single HTTP request and which host the load balancer routed it to.

I'm aware that there are mechanisms to prevent host-level inconsistency (Redis pub/sub, local cache pushing to message broker etc...), but as Werner Vogels says "Everything fails all the time". I hope I have convinced you that host-level inconsistency is a sufficiently undesirable failure state to make us rethink whether that 5000x speed-up is *worth it*.

Another fairly common failure state of local caching is that it would be fairly easy for 2 concurrently processing tasks to mutate the same data and potentially affect each other's flow. Thus local mutations against data read from a distributed cache are "thread-safe", which could be worth the price of admission alone.

Let's continue by identifying other downsides. Local caching is memory-inefficient. If each machine in the system needs to cache `item A` for the system to be consistent, then we need to store N copies of the data instead of 1. As the amount of data we cache and the number of machines running our code grows, so does the *total* amount of memory we are using to maintain this local cache. I predict that minimizing the memory footprint of our servers will become extremely desirable in a Kubernetes/containerized everything world. The smaller the memory footprint your server has, the smaller the "cracks" of under-utilization it can fill. This is why Lambda charges you by the amount of memory you require, by the way. I think that in the upcoming years, writing servers that require many GBs of local memory to run will begin to be considered a huge anti-pattern once containerization becomes fully ubiquitous.

Beyond the containerization angle, network IO latency has been decreasing much faster than memory latency has in recent years. [This post](https://richardimaoka.github.io/blog/network-latency-analysis-with-ping-aws/) shows an average 0.09ms round-trip inside the same AZ which is almost 2x faster than the 0.17ms round-trip quoted by Nick Craver in his post. According to [this post](https://cacm.acm.org/magazines/2017/4/215032-attack-of-the-killer-microseconds/fulltext), a typical 200-300m datacentre round trip takes 1 microsecond at the speed of light so it seems feasible that 0.09ms AWS round-trip could be shaved down by an order of magnitude. While it will always be slower to go over the wire, these advances in networking do make the trade-off discussion a lot more interesting.

In conclusion, I'm becoming more convinced that you should *not* locally cache all the things (you *should* async all the things, though). Your caching strategy should probably default to distributed-first and only use local for small subsets when the trade-offs make sense:

- Very hot, latency sensitive code path
- Infrequently changing data
- No local mutations needed
- Fewer often fetched items that are larger in size

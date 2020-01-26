---
layout: post
title:  "MemoryCache deep dive"
date:   2020-01-25 00:00:00 -0500
categories: c#
---

I was curious to know more about how the MemoryCache implementation works in .NET Core and the documentation (as usual) provides an OK but not wholly satisfying look into the internals of it, nor does it provide much color into good practices or things to watch out for when using this type.

One interesting thing that is fairly vaguely documented is that removal of expired items and compaction are both only triggered by additional subsequent operations on the cache (`TryGetValue`, `SetEntry` or `Remove`). This means theoretically you could write a bunch of stuff to cache that expires and it will just sit there in-memory even though you could never retrieve it.

The expiration cleanup by default runs every 1 minute (presuming there was a cache operation to trigger it) and runs by creating a fire-and-forget `Task`. The compaction also runs on a background thread, though interestingly that function uses `ThreadPool.QueueUserWorkItem` as opposed to `Task.Factory.StartNew`. Based on this, it probably makes sense to have a singleton `IMemoryCache` or at least look for opportunities to share fewer ones across your code to reduce  thread utilization.

The compaction only happens if a size limit is set on the cache, and if one is set then you will have to provide a size for each cache entry you set which it uses to know how much it needs to remove to reach it's compaction target.

Based on these findings we can derive the following good practices when using `MemoryCache`:

- Try to use it as a singleton to minimize thread pool usage, unless you specifically need to run multiple instances of it with vastly different options. Avoid running one for each domain repository that is cacheable.

- Always set a size limit, the documentation implies in places that "memory pressure" can trigger a compaction but that does not appear to be the case when looking through the code.

- Goes without saying, but always ensure all entries you add have an expiry set.

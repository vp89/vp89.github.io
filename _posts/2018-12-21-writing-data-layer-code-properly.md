---
layout: post
title:  "Writing data layer code for your API properly"
date:   2018-12-21 00:00:00 -0000
categories: programming
---

Earlier this week, this [story](https://www.theguardian.com/info/2018/nov/30/bye-bye-mongo-hello-postgres) was trending on HackerNews. It's a good read about how the software people at The Guardian undertook a lengthy migration project from MongoDB to PostgreSQL. If you have not read it I highly recommend giving it a read as it outlines a pretty good playbook for doing such a project.

One piece that stood out to me was their admission that their API code was so coupled with Mongo that it was easier to start from scratch than to attempt to refactor it:

`
This is one of the reasons why building on top of the old API wasn’t an option. There was very little separation of concern in the original API and MongoDB specifics could be found even at the controller level. As a result the task of adding another database type in the existing API was too risky.
`

I recently had to do a similar investigation into a code base at my workplace and I came away with a similar conclusion. The database specific code was too tightly woven in and refactoring it would be a significant project with some risk involved.

I've read about this a lot and experienced it plenty in my career so far. It seems wherever you go no one knows how to structure their data layer code in APIs and they are making these same mistakes. Here are some pretty basic tips you can follow to write more maintainable API code:

### Read up on dependency injection, composition and inversion of control

These are very powerful concepts that when used with good taste and judgement allow you to build simple, modular code. Forget about inheritance, one of the worst concepts in programming history. Inheritance is guaranteed spaghetti. Code is not naturally hierarchical so stop trying to shoe-horn it into such a model.

Think of your API as a power drill and your data layer code as a drill bit. You should be able to plug in different drill bits if you need to without buying a new drill each time. This doesn't require much more additional work once you know what you are doing, and the flexibility it provides you is huge. With everyone moving to managed services the ability to pivot to different data stores and services is becoming much more important.

### Tag your queries and do the database performance analysis at the database

For all mature databases there are good third-party products that pull the query history and can provide you with reports on duration, execution statistics, waits etc ...

If you tag your queries you will be able to group them together and when you see something catching on fire in production you can quickly grab the tag name and find the exact spot in your code where it's being called.

Do not push database metrics out from your app. It's more efficient to pull it in batches directly from the database.

### Forget about query builders, hand-write your queries

Your queries shouldn't be obfuscated behind some weird query builder incantation. You should be able to go from a tagged query name in a production log to the full query code ready to debug and optimize inside of 5 minutes. If it takes you time to figure out how the query builder works or if you have to replicate the call to generate the query locally you are just wasting time. Having the query code be obfuscated has a behavioral impact where it's "out of sight, out of mind". Performance improvement then becomes painful and people put it off until it's too late. You want to make it as easy as possible to fix performance issues so dump the query builders.

### Forget about fully-fledged ORMs, use micro-ORMs only

This goes hand in hand with #3. Dapper and hand-written queries is the way to go. If you are not of the .NET persuasion I'm sure your language has some lightweight micro-ORM where you can plug in hand-written queries. The micro-ORM can deal with the tedious mapping provided it does it with very minimal overhead, but it should definitely NOT generate queries or decide how many times a query is ran. That should always be explicitly done by your own hand. You can't properly reason about the performance of your app if such things are hidden from you.

If you absolutely have to use the ORM, you should examine carefully every single call using a profiler to see how it translates into real calls to the database. Often ORMs have unexpected behaviors when pagination is involved or getting the top N results, so make sure you understand fully and document in your code how your ORM behaves. When it's time to debug performance issues this in-depth understanding is critical.

### Repository classes receive primitives only, no objects

Don't bog your repository code down with untangling of complex objects. Each method in your repository class should have 1 parameter for each matching query parameter and anything else that might be needed for conditional query generation. You want to keep the business logic at the layer above that, so if you need to write another instance of the repository you are only rewriting the queries.

### Controllers shouldn't know anything about the database (or even the business logic)

Your structure should be Controller -> Service -> Repository.

The Repository implements an interface, the Service doesn't have to because it should just be in-memory business logic and any IO calls should be done using an injected dependency object. The controller code should only be responsible for sanitizing input, calling the service(s) and reacting to the responses of those calls with specific HTTP response codes.

If you start having business logic in your controller and/or your repositories you probably need to push it into the middle layer.

The service layer is then the one which receives the bulk of attention with unit tests as that's where all the business logic should be.

### Conclusion

I hope these tips are helpful to you. This was a quick brain dump as I have been thinking a lot about this stuff recently. This post is a quick distillation of many years experience I have working on this stuff so it's difficult for me to not be opinionated. If you follow these tips you could be someone who doesn't have to potentially rewrite their entire API after making a bad database choice! I will probably do a post on THAT soon as I have a lot of opinions there.

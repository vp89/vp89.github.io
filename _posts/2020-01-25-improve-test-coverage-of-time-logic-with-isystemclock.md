---
layout: post
title:  "Improve test coverage of time sensitive logic with ISystemClock"
date:   2020-01-25 00:00:00 -0500
categories: c#
---

A quick post to get back into writing for the new year. The 2nd half of 2019 was pretty hectic with me starting a new job, but things are beginning to stabilize and I'd like to start putting posts out on a semi-regular cadence again.

Knowledge about "clean code" and coding for testability is pretty ubiquitous now, but one thing you still don't see a lot of is abstracting out the system clock. The thinking being that if we just use `UtcNow` everywhere, then it will all always work in the same way regardless of what time it is. But the real-world is not quite so simple. Sometimes there are hairy, annoying business reasons for why we have to do weird stuff with times and timezones. We may need to always show days left in the month in a specific timezone, or we may need to write logic based on timestamps that may not always have the timezone included with them. In these trickier cases, it would sure be great if we could write tests that prove our code works correctly all of the time!

[ISystemClock](https://github.com/dotnet/extensions/blob/aabe8c34a62786c313e20125d70b36d3c5e72a75/src/Caching/Abstractions/src/Internal/ISystemClock.cs) allows us to do just that. It's a very minimal interface, abstracting out just calls for UtcNow (there is a SystemClock implementation) but it should be all you need as you can just derive any more complex needs from the result of UtcNow.

By making the system clock injectable, and removing any naked calls to `DateTime.UtcNow` you can pass it into your units under test as a mocked object setup to return different values for each test case. This allows you to massively increase your test coverage of hairy time sensitive logic without writing much extra code.

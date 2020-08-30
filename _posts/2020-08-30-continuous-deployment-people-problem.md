---
layout: post
title:  "Continuous Deployment is a people problem"
date:   2020-08-30 00:00:00 -0500
categories: programming
---

Continuous Integration (CI) and Continuous Deployment (CD) often get lumped together because they can be achieved in a lot of cases using the same tech, Jenkins et al. Both concepts have the word *continuous* in them and they are often marketed together, so people who have CI think they are doing CI/CD. But how many sufficiently large companies out there are truly doing CD?

Even the most technologically backwards companies have CI these days because it's very easy to drop it in and it likely doesn't require any major process changes provided you already use some form of "centralized" source control. Yes I know Git is distributed, but in the real world it's *effectively* centralized.

CI is thus, pretty much purely a technological problem. SREs provide some servers that hook up to the SCM and run builds any time changes are pushed. The extent to which the CI is useful is also a technological problem. The "coverage" and determinism of your test suite, the packaging of build artifacts and configuration etc ... these are all things that can easily be solved with the right CI architecture and lines of code.

CD is both a technological problem *and* a people problem. The tech and patterns for executing CD are fairly well trodden now, so the tech side of it is not too challenging. The people problems around CD, however, *are* extremely challenging. As I mentioned earlier, I'm mostly talking here about medium/large engineering orgs. The companies who are currently hiring the most engineers in the world were formed well before CI/CD was table stakes, so their engineering practices have had to evolve to absorb these concepts instead of them being "native" to the org upon inception.

CD is more difficult to absorb into an existing engineering org than CI. This is because CD and bureaucracy around releasing code are anathema. If the org is sufficiently mature and was not "CD native", then it likely has lots of bureaucracy around releasing code. Sadly, the most prevalent way of thinking about change management is that stability and "good outcomes" come from increasing friction towards change. If we add process, then more things are less likely to go wrong. The [DORA reports](https://cloud.google.com/blog/products/devops-sre/the-2019-accelerate-state-of-devops-elite-performance-productivity-and-scaling) have largely proven this way of thinking to be incorrect. The teams that operate with the least friction have the best stability. These surveys have been compiled for several years now so this is not flukey, cherry-picked data either.

Anecdotally, I have experienced both high friction and low friction environments. Low friction definitely leads to higher velocity. I have seen issues with stability in both types of environments. This lends me to believe that low-friction change management alone doesn't give you stability, but it *is* a pre-requisite for stability. Stability comes from building reliable systems. Building reliable systems is hard. It's easier to write code that doesn't handle intermittent failures properly, doesn't ensure a bad message always ends up in the DLQ. It's easier to write naive systems without idempotent processing, without the ability to replay previously failed input.

People have finite amounts of willpower and time. If the easy things are hard, then the hard things never get done. We must therefore create space for hard things to get done by making everything that is not inherently hard (such as deploying changes to Prod) as easy as possible.

The orgs that have low friction have better stability and better engineering outcomes because their engineers have more time and willpower to solve the hard problems that their systems face. All the easy stuff has been completely automated away.

How do you solve the people problems that stop CD from being adopted? The org must first be willing to admit that the high-friction culture that currently exists around change management is completely wrong and not condusive towards either stability or velocity. This either requires a lot of "campaigning" to win hearts and minds, or there must be a mandate from the very top that allows one to extract certain roles/functions in the org whose purpose it is to preserve the high-friction status quo.

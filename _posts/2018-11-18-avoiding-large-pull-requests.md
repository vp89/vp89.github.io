---
layout: post
title:  "Avoiding large pull requests"
date:   2018-11-18 00:00:00 -0500
categories: sample
---

Large pull requests (LPRs) are a bad practice that all developers are aware of but yet engage in frequently. While there are many - sometimes fair - reasons for the existence of LPRs, teams should strive towards them being the exception rather than the norm.

### Drawbacks of large pull requests

Everyone involved in a code review should aim to understand every single line being changed and the reason why it's being changed. While there is not a linear relationship between lines of code changed and time needed to review them, LPRs have high cognitive costs for conscientious reviewers. Reviewers are aware that LPRs typically require a full context switch from their own work, and thus may put off reviewing the code until they are finished with whatever they are currently working on.

Long-lived branches generally waste time, as developers may need to rebase multiple times to ensure their change is compatible with the head of the master branch. LPRs which stay open for a long time and touch many files will require more merges and those introduce a risk of mistakes. Reviewers often trust that merges are done properly and this can lead to bad changes slipping through the review process.

So far we have been discussing LPRs and how they affect conscientious reviewers. Another reality is that not all reviewers are conscientious reviewers. Some people are less thorough and detail oriented than others. LPRs make it harder to be a conscientious reviewer and you want all your developers to be thorough and detail oriented when reviewing changes being made to your code.

Stepping back to long-lived branches - these can sometimes take on a life of their own. Someone could be working on a large change to an internal tool. The code has yet to be merged but others on the team begin using that code because they need the functionality now. In the meanwhile, other important changes are being merged in to master. This creates lots of confusion and stress about the state of the repo.

### How can we get rid of them?

I hope I have convinced you that LPRs are #bad and you should avoid them at all costs. There are many valid technical and behavioral reasons for LPRs, some are unavoidable but fortunately most aren't.

Reducing friction towards merging code is critical in avoiding the dreaded LPR. One way to do that is to remove business sign-off as a prerequisite to merging code and to encourage a practice of using multiple PRs to complete a "story". Developers can then separate out refactoring and business logic changes into separate PRs and merge the business logic change in last.

Reviews should be assigned based on availability and knowledge of the code base, instead of arbitrarily. While it seems "fair" to assign code reviews using a random number generator, it can be disruptive and it's generally not as effective. Developers should be made to use their judgement to achieve favorable outcomes, brainless process should be a fallback option and not the modus operandi.

Not requiring multiple reviewer approval before merging a change is another good way to reduce friction. It does introduce risk if not all potential reviewers are conscientious and familiar with the code base. This risk can be mitigated by encouraging a culture of thorough code reviews and by minimizing developer churn on your projects.

Often pull requests are large because they are full of refactoring and/or formatting changes that the developer felt were crucial in being able to deliver the required business logic change. Reading code others have written is hard and thus developers love "refactoring" even when it's not necessary or when it doesn't deliver any significant benefits. Often the mechanical act of slicing and dicing a block of code someone else has written to suit our own preferred coding style can help us understand it.

Developers should evaluate whether the formatting or subjective style changes they are making are necessary and be more empathetic towards those who will be tasked with reviewing their change. Code repositories should have contributing guidelines around style and formatting to avoid ambiguities. Wherever possible let the machines do the formatting and save your brain cycles for the stuff that matters.

Refactoring should be deferred unless it is essential for delivering the business logic change. Developers won't feel comfortable doing this unless you actually dedicate time towards refactoring/code quality in each work cycle. Ideally there should be a "refactoring backlog" and for each work cycle a certain % of capacity is allocated towards it. Good software engineering cultures understand that dedicating time for work that doesn't provide direct business value can often improve your velocity and capability to deliver business value.

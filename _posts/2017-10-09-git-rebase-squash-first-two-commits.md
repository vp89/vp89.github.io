---
layout: post
title:  "Git Rebase - Squash your first two commits"
date:   2017-10-09 00:00:00 -0500
categories: programming
---

While playing with Git I learned that it originally did not allow you to rebase to the root commit, an option was added to enable this at some point and it
can be done with the following command:

```csharp
git rebase -i --root master
```

A reason you may want to do this is if you want to push a squashed "First commit" to your remote, but you have multiple messy local commits when starting your
project.

One way to avoid this is to always create a no-op first commit whenever you create a new repo, that way you'd never have to rebase against the root.
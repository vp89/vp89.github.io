---
layout: post
title:  "Migrating a Git repository to a new remote"
date:   2019-02-27 00:00:00 -0000
categories: programming
---

GitHub now offers free private repos, so I will be moving my projects from Bitbucket to them. Mostly to test out their CI/CD integration with Heroku.

Here is a quick guide on how I did that:

- ```git pull```
- Create your new Git repository
- ```git remote add new-origin <new repository URL>```
- ```git push --all new-origin```
- Verify that entire history has been pushed
- Each local needs to run the following:
    - ```git remote remove origin```
    - ```git remote rename new-origin origin```
    - ```git branch --set-upstream-to=origin/master```

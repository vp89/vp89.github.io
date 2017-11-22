---
layout: post
title:  "Setup p4merge as your Git diff tool on OSX"
date:   2017-11-22 00:00:00 -0500
categories: programming
---

While sites like Bitbucket and GitHub have nice UIs for viewing diffs
of changes, it's also helpful to view things locally so you know exactly
what you are commiting to your repo.

Git allows us to set up as many as we want, and then it provides us
with 2 pointers (diff.tool and diff.guitool) so we can specify
for a non-GUI or GUI workflow which of these installed tools we 
want to use.

In this guide I will show you how to setup p4merge as your GUI diff tool.

This guide assumes you use homebrew. The first step is to install p4merge, if you don't have it already:

```
brew cask install p4merge
```

Homebrew automatically copies p4merge to /Applications/p4merge.app

Now we want to register "p4mergetool" as our default **GUI** diff tool. 
This allows us preserve current behavior for non-GUI
workflows (on my machine, it's set to use vimdiff).

```
git config --global diff.guitool p4mergetool
```

We then need to tell Git what to do when we try to use "p4mergetool":

```
git config --global difftool.p4mergetool.cmd \
"/Applications/p4merge.app/Contents/Resources/launchp4merge \$LOCAL \$REMOTE"
```

If all went well, the below command should open p4merge and show 
you a nice diff of the current changes:

```
git difftool --gui db/dojomanage.sql
```

You can also make sure you didn't modify the existing non-GUI difftool functionality,
this command should run vimdiff or whatever your machine is set to:

```
git difftool db/dojomanage.sql
```


You may notice Git prompts you any time you run git difftool, this is
annoying. I'm going to solve it by creating an alias
called "gitdiff" which runs with both the gui and no-prompt flags on.

Using the alias allows me to have both a short-hand and a sensible
default for how I like to work, without touching the actual Git
defaults on my machine.

Add this to ~/.bashrc

```
alias gitdiff='git difftool --gui --no-prompt'
```

Then reload:

```
source ~/.bashrc
```

If you don't like using aliases, you could have the same shorthand effect by setting your gui tool as your default and having no-prompt set as default.

For further info the Git
documentation page is pretty useful:

[https://git-scm.com/docs/git-difftool](https://git-scm.com/docs/git-difftool)
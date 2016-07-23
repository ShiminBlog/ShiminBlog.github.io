---
layout: post
title:  git problems on hands
date: 2016-07-23 16:23:30
---

##  git problems on hands ##
I am not a git-use-expert. These problems often occur in my work. I write them here and give the ways how to deal with them.

### Please, commit your changes or stash them before you can merge

Sometimes, to pull the changes from git server and get the error message like this:
> Please, commit your changes or stash them before you can merge <br>
>     xxxx.cpp <br>
> Please, commit your changes or stash them before you can merge. <br>
> Aborting

It can be resolved in this way and have no effect on local files:
> git stash <br>
> git pull  <br>
> git stash pop <br>

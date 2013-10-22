---
layout: post
title: "Spotting Deployment Regressions Like A Boss"
date:   2013-10-22 01:13:10-0300
tags: git capistrano
---

A common work flow I've seen some Rails devs settle in seems to include implementing features, fixing application bugs, and whipping up rigorous tests all locally on their machines and only to later cross their fingers hoping it didn't break on deployment. Since the only good code is shipped code, deployment time eventually comes around and suddenly chaos ensues as neither god, stack traces, or Stack Overflow seem to indicate what exactly is causing the broken deployment (we've encountered many just such hard to track down problems with the Rails asset pipeline). DevOps to the rescue!

We've got a wonderdful tool in our hands with [Capistrano](https://github.com/capistrano/capistrano), at least when deployment fails there's still a nice working build running as Capistrano handles rollbacks to the last working copy gracefully. For debugging these broken builds we could spend hours pouring over documentation looking for some funcional change in all our gem dependencies, blindly altering code until it works again... or break out the power tools and nail this in a matter of minutes.

### git bisect
Times like these are when you truly see the value of a great version control system like `git`. You could go backwards checking each commit one by one, but that's too time consuming. We'll make use of binary search to drastically cut down our possible sample space of breaking commits using [`git bisect`](https://www.kernel.org/pub/software/scm/git/docs/git-bisect.html). The bisecting workflow is fairly simple.

```bash
git bisect start
```

For every commit we pass through we mark it as good or bad if it doesn't exhibit the bug we're trying to track down. Initially we establish the boundaries of our search, and the current commit where we're starting from already has a problem so we'll tag it as such:

```bash
git bisect bad
```

Next we have to establish a decent endpoint where we know all was fine. You may not remember it but you sort of have an idea that 20 commits ago everything worked for sure.

```bash
git bisect good HEAD~20
```

You need only have a vague idea of when it last worked.

### capistrano local copy
Since we'll be constantly swapping commits through `git bisect`, it's imperative to make sure you're deploying from the commit you checkout last from your local repository, not the tip of some remote branch. Fortunately though, we don't have to edit our deployment recipes each and every time we change commits. We'll override a few Capistrano variables in a separate file `Capfile_override` to ensure we're pushing the right code.

```ruby 
# Capfile_override
set :repository,    '.'
set :scm,           :none
set :deploy_via,    :copy
set(:copy_exclude)  { %x(git clean -ndX).split("\n").each { |file|
                       file.sub!("Would remove ", "") }}
```
On every invocation of Capistrano we've also got to make sure we're loading our recipes and variables in the right order, we manually load our files with the `-f` flag. 

It's not particularly wise to try this on a production server and risk unnecessary downtime or database corruption, so spin up a VM and use it as the deploy target. Capistrano also allows us to easily change our deployment machine using the `HOSTS` environment variable.

```bash
HOSTS=192.168.122.100 cap deploy -f Capfile -f Capfile_override
```
*Don't forget to adjust the deployment command as necessary in case you use things like bundler or multistage extensions that require additional arguments. For example update your gems with `bundle install` on each run.*

### git bisect run
The exit code from the `cap deploy` tells us exactly whether our deployment was successful or a dud, we're going to take advantage of that to automatically tag our commits and speed up our search. We can automate this process using `git bisect run`. It takes a script that tests for our bug keeps skipping until it tracks down our culprit.

Putting it all together.

```bash
HOSTS=192.168.122.100 git bisect run cap deploy -f Capfile -f Capfile_override
```
Voila!

```
5da18fab84e7cff4b39ceaeebcd2d7c8b88cbe40 is the first bad commit
commit 5da18fab84e7cff4b39ceaeebcd2d7c8b88cbe40
Author: Joao Sa <joao.sa@innvent.com.br>
Date:   Tue Oct 12 11:48:37 2013 -0300

    Updates matross gem

:100644 100644 a1494a...13809 22159...7d779 M    Gemfile
:100644 100644 8aece6...6462d 00899...332cf M    Gemfile.lock
bisect run success
```
Now that we've found our first broken commit a quick diff will point out what caused our breakage.

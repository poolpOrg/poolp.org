---
title: "OpenSMTPD: development, bug tracker & GitHub"
date: 2012-06-19 19:57:37
category: OpenSMTPD
author: Gilles Chehade
---

It's been a while since my last post about OpenSMTPD.

I don't know what Eric and Charles have been up to, I was quite busy with my daytime job and couldn't really spend much time on the hobby ;-)

Anyways, one of the most requested feature of OpenSMTPD is to provide users with a mean to report bugs and let them contribute more easily. It's been requested two times this week, it's been requested countless times in the last few weeks and months, and as a matter of fact it's also been a nuisance for us as until now we've kept track of bugs and feature requests in my mailbox. Bug reports have been lost, others have been forgotten, it's not productive.

I decided to take a shot at fixing that issue and proposed to my fellow coworkers that we create a Github project, just like what jasper@ did for ports. Charles was already a big fan of Github ("I <3 Github") and Eric approved and created an account. So here we are ;-)

Why Github ?

I initially thought about writing a small tool at poolp.org to keep track of bugs and features, and it wouldn't be too hard, but it requires time and that is a very scarce resource these days.

I've had very good feedback of Github, it has a nice interface, is easy to use and has the bug tracker, comments and annotations I want. I don't see the point in reinventing the wheel (at this point) so as far as I'm concerned it will be just fine.

We don't intend to use the repository there as a development repository but instead as a mean to let users know what's in the works by exposing upcoming features.

See, OpenSMTPD has several repositories:

The OpenBSD repository, which is our primary repository and where OpenBSD users should grab the latest version of OpenSMTPD. It is as close to production-ready as it can get and no other tree (that I know of :-) holds a "better" version.

The portable repository, which is maintained by Charles Longeau at Github, and which is basically a checkout of the OpenBSD tree with the portability goo required to build on other operating systems.

Then, at poolp.org, we have a private repository where Charles, Eric and I create branches to work on specific new features that can possibly break OpenSMTPD, bring their share of regressions, or that are simply not meant to hit the OpenBSD tree as-is (the so-called "experiments" :)

The repository at poolp.org contains at least 2 branches:

The master branch which is synchronized with the OpenBSD tree and from which we derive when we implement new features; the poolp branch which is an experimental branch we use to run code live and test that it is reliable before it is committed to OpenBSD; and eventually several minor branches that come and go.

The workflow goes more or less like this:

create a branch X from master for feature X;
work on feature X until reasonnably stable;
merge feature X in branch poolp;
test on mx1.poolp.org that branch poolp is stable;
commit feature on OpenBSD;
synchronize master branch with OpenBSD;
The repository at Github would clone our master branch and eventually other branches, making it visible to the community.

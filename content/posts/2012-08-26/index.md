---
title: "OpenSMTPD: I have no idea for that title, sorry"
date: 2012-08-26 22:53:03
category: OpenSMTPD
authors:
 - "gilles"
categories:
 - technology
---

Howdie,

Eric and I had planned a loooooong time ago to have a hackathon this week-end. However, some asshole piece of shit ran into my car while it was parked, made it unable to take the road and flee without leaving a note. Long story made short, I had to be at the repair shop this week-end to be able to get my car before end of September.

We did get our hackathon though and while it was probably much less productive than if we had met in the same room, it was still very very interesting with tons of improvements and many great features written :-p

Add support for types and improve increment/decrement in stat_backend I recently committed the stat_backend API which allows us to provide different backends for statistics storage. Until now these stats were necessarily integer counters that supported being incremented, decremented and set.

I added support for types allowing us to store more precise/accurate statistics in places where integers weren't the best way to represent a value. It is currently possible to store counters, unix timestamp, timeval, timespec, etc ... We can easily extend types as we ran into uses for them.

While at it, Eric asked me if I could allow increments/decrements by values to avoid having to call stat->increment() or stat->decrement() multiple times when a batch of increments/decrements are happening.

Done, committed.

Introduce TRACE_PROFILING and profile stats A while ago, Eric has written profiling support which we have used to pinpoint some bottlenecks and fix them pretty successfully. For some reason, it was never committed but the idea was to fetch clock value before and after an imsg callback, allowing us to know precisely the time spent in each events.

I merged the profiling support to our -current tree since it required some rework and committed it; then I took advantage of the new stats types support to also support pushing profiling information into the stats API.

With the ramstat backend it has limited value, but with a persistent storage backend for statistics you can easily graph your events profiling in real-time with about no impact on performances. This doesn't sound too sexy to users but believe me it is. If you run OpenSMTPD on a very busy machine and you think it is not working as efficiently as it should, you can get a LIVE insight at which operations are taking time and should be optimized.

Start removing user backend We currently have a user_backend API which is supposed to allow us to bring support for virtual users. That API needed to provide getbyname() and getbyuid() handlers to allow users lookups by name AND uid, preventing us from simply using the maps API.

Turns out that the only use for getbyuid() was in the offline enqueue code path which can safely use getpwuid() as it will necessarily come from a local user. With getbyuid() out of the way, we're almost ready to start removing user_backend API and start providing support for completely virtual users using maps and the supported backends.

Scheduler improvement Eric did quite a bit of work to improve the scheduler, fix some bugs in it and think about some of the improvements we want to bring to it.

We still have a disagreement on one algorithm to use internally to boost performances further but it only affects a small part of the scheduler and we will sort that out by experimenting both as it is really nitpicking at this point and only a live test with billions of envelopes in the queue will prove either theory to be right/wrong.

Improved logging Todd (toddf@) improved IPv6 addresses logging by also printing ports when needed. He also changed the enqueuing logging to display both server and clients exchanges, instead of the old method that only displayed server answers. While there Eric added logging for smtpd pauses and resumes as well as envelopes removals.

Events race condition at startup We discovered a race condition that could cause OpenSMTPD to crash at startup if some processes start sending imsg to others before they are ready to service them.

Eric fixed the following bugs by disabling some events and enabling them back when the process is ready to serve. This has been a long-standing issue fixed with a 6 liners diff :-p

FD exhaustion handling in smtpd and control process Until now, if OpenSMTPD had reached system-imposed limits on descriptors, it would simply fatal. I spent some time making it more reliable by defining a descriptor reserve under which it would stop accepting clients on the network or through smtpctl until the exhaustion situation is resorbed.

Experimenting showed that it works pretty fine, there is still probably some improvements to do in that area performances-wise but at least it is stable in such situations and sessions run smooth without performances degradation.

Queue compression Charles committed his queue compression feature which allows OpenSMTPD to deflate/inflate envelopes and messages transparently when they hit or are read from the queue.

It worked pretty fine but when I tried to inflate an envelope, I realized that the envelopes and messages didn't use the same compression algorithm. I reworked a bit the compression backend to have it use gzip algorithm consistently.

This is awesome, it works out of the box by adding to smtpd.conf:

queue compress

With this, the size required to store envelopes is halved and a huge gain can be made on messages at the cost of some CPU time ... and from an administration point of view, the queue can be inspected using gzcat instead of cat ... piece of cake !

Qwalk rewrite We received a bug report where envelopes would appear several times in a mailq output.

Eric started investigating the issue and pinpointed the qwalk() API as the culprit. He rewrote it in a saner way and got the issue fixed.

While he was doing it, another OpenBSD hacker sent a different bug report which turned out to be also fixed with the new qwalk(). That's what we call proactive bug fixing :-p

Queue encryption Finally, one of the features I've been waiting for a long time to implement: encrypted queue.

I built upon Charles' queue compression diff to provide queue encryption as well. It works in a transparent way and out of the box by simply adding to smtpd.conf:

queue encrypt "mysecretkey"

With this, envelopes and messages are encrypted using AES-128 in ECB mode with "mysecretkey" as a key. I have plans to make the algorithm and mode selectable but I wanted it to work first before I start improving it. This feature is compatible with the compression, so you can enable both and it will compress AND encrypt (in proper order ;-) transparently.

What's even nicer is that our handling of envelopes/messages is fully isolated OUT of queue backends so that if you write a queue backend you never deal with envelopes and message contents. The side effect is that you can write a queue backend to store on a remote untrusted storage ... and people with control of that storage can never inspect content if encryption is enabled :-p

I didn't commit it yet but it should hit the tree this week.

Stay tuned for more awesomeness !


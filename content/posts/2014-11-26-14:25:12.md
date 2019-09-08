---
title: "Some OpenSMTPD overview, part 1"
date: 2014-11-26 14:25:12
category: OpenSMTPD
author: Gilles Chehade
---

EHLO world !
------------

Yesterday I thought I'd write a first OpenSMTPD-related post to sum up the changes that have happened since this summer but it turned out to be painful as they amount to quite a lot. Instead, I think a better strategy is to split this into a serie of smaller posts focused on the specific changes ;-)

So, in June I have organized a mini hackathon at my brand new place and invited some of the French Connection at home. The goal was to work a bit and find an excuse to watch the French team take over the world cup before an eventual surrender once everyone had left. Since it was an OpenSMTPD hackathon, eric@ and chl@ were present, but we also invited miod@ who had been hosting me several times at his hackathons and jca@ who turned out to live close to my place.

Eric and I planned to finish work on the filter API which would allow plugging third-party filters to the daemon for session-time and data post-processing, Charles had a few things to cleanup in the portable branch, and we all had a few other things we wanted to deal with.

Today, I will talk about the filter API and pluggable backends.

The filter API
--------------

Filters have been long awaited in OpenSMTPD, it's been a constant feature-request for at least two years.

In fact, filters have existed in OpenSMTPD for a long time as an (unsatisfying) unexposed prototype, but we had strong ideas about where we wanted to go with this subsystem and this was just not it.

We decided to start from scratch and only retain one idea from the prototype, the fact that a filter is not a shared object. The idea behind that was that we wanted filters to run outside of the daemon memory space and be able to provide per-filter privilege separation and chrooting. Of course, we didn't want to execute the filter for every session as the overhead would be ridiculous, so a filter would effectively be a daemon communicating with the server through IPC.

Now, our other goal was to make filters dead simple for both users and developers and having to write a daemon with IPC was kind of raising the bar too high. So the biggest challenge was to overcome this and make sure that neither of them would need to actually see that.

We came up with an API which turned the filters into event machines, a filter writer would call a filter_init() function that would take care of all the daemon/imsg part, then call a set of functions to register callbacks on specific events and finally call a filter_loop() function to make the filter start processing requests. These ideas were already part of the first prototype, but had to be revamped to allow for privilege separation and chrooting to be doable.

At the same time, eric@ completely rethought filter chaining to allow filters to pass data one to another, it was a much more complex task than it looked as the tricky parts are not getting it to work, but getting it to fail gracefully if a filter suddenly craps itself.

By the end of the hackathon, we had the subsystem mostly done and functional to a point where I could write both a python bridge filter and a php bridge filter so filters could be written in these languages, and write a few dummy filters to alter content, turn it into l33tsp34k, reject mails containing viagra and what not.

Shortly after, we received our first filter contribution from a community member who wrote a DKIM signer using our API, another feature long awaited :-)

Our next major release should come with this API enabled, the only thing blocking us from doing so at the moment is the lack of feedback and confidence that it is rock solid. People have been requesting this API for ages yet it is very very hard to actually get people to use and comment on it.

Pluggable backends
------------------

As I already mentionned in previous posts, we had done a lot of work on abstracting the queue API to make it "pluggable" in the sense that one can write a queue in a standalone file, expose it through a structure and have a pointer point to that structure to start using it instead of our stock queue. This has been done a long time ago to make it easier for us to experiment, revert and experiment again with confidence we didn't break something else in the way.

From OpenSMTPD's perspective, a queue is basically an iterable key-value store and all it has to know is that messages and envelopes have identifiers that it can pass to the queue operations which will know what to do with them. It no longer has any knowledge of the internals and you can basically move a queue to a database, a cloud or your own filesystem-based storage and the server will not notice a difference. As a matter of fact, the first prototype we wrote to prove this worked was to store a queue on the Google cloud, and let either one of eric's or my laptop schedule the message without having it locally.

At some point, I convinced eric@ that we should have a similar interface for the scheduler if only to make sure that the daemon did not make any wrong assumption on scheduling. I came up with a prototype while at an OpenBSD hackathon in Budapest two years ago and once it started working, eric@ took over, fixed some API layer violations and improved it further. All over last year, we had to do many changes to the scheduler while running OpenSMTPD in a high-volume environment and this has proved to be a very useful change.

So... if we have to play games with pointers and rebuild, that's not really "pluggable" is it ?

When I first considered splitting the subsystems into their own isolated API, what I had in mind was not just to make it easier to test things. I wanted to be able, just like filters, to provide an API that would allow plugging custom queues and schedulers without having to modify something in the server. People have tons of different use-cases and we want to keep the OpenSMTPD code base as simple, clean and dependency-free as possible. So by providing the proper API we can ensure that people can deal with their use-cases without having us to cooperate, they're free to use whatever dependency and we will not have to even care.

During our little hackathon, I asked eric@ what was needed to avoid the pointer juggling and just make it possible to set it from our configuration file. He worked on it for a few hours and we ended up with a version that would allow me to build a queue as an external executable that could be plugged and communicate with the daemon like filters. Since I was in a pythonic mood, I wrote a queue-python which allowed me to write a queue backend in python and used it to write a prototype queue_mysql in python. This was not really intended to be a serious project, just a proof of concept that this kind of things can be achieved, but it only took a couple hours to complete these and the python implementation of a queue is so small that it paves the road to interesting experiments :-)

Eric also made the scheduler pluggable, I didn't write one as I got context-switched to something else, but you get the idea of what can be achieved now.

What's coming in next post ?
----------------------------
I'll be discussing the new OpenSMTPD-extras repository which holds all pluggable contents, as well as the smtp/mta processes merge, the mfa process removal and reyk@'s RSA privsep engine.

Eric will be talking about asr, the asynchronous resolver, and how we pushed him into making it a standalone library.

I'll convince Charles to talk a bit about portable :-)

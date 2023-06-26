---
title: "OpenSMTPD: plenty of news"
date: 2012-08-21 22:22:00
category: OpenSMTPD
authors:
 - Gilles Chehade
categories:
 - technology
---

It's been over a month since my last post but I have an excuse: I was super busy, then I was super on vacation ... but I'm back again ;-)

So my life was pretty intense these last few weeks, both professionnally and on the opensource front. I'll explain later, for now let's just focus on the OpenSMTPD goodies.

OpenSMTPD portable to a new system First of all, I'd like to stress out that Rune Lynge and Charles Longeau did an amazing job at porting OpenSMTPD to yet another system ... MacOSX. While I don't know anyone running a mail server on MacOSX, nor do I know anyone who knows anyone doing it, the fact that we can do it fills my heart with joy. We can officially be a free and very expensive mail server at once ;)

The compressed/encrypted queue experiment Charles also worked on a proof-of-concept to compress the message and envelopes in the queue. It is not committed yet but there is discussion around it in private as it will very likely hit the tree soon. This is a very important move forward as it also proves that compressing/encrypting the queue does not prevent OpenSMTPD from working which in turn proves that we will be able to store an encrypted queue in an untrusted storage (ie: public cloud provider) and still be able to deliver mail while retaining privacy. Believe me, we're not done on that front and you'll hear about it again in the future.

Scheduler/Queue layer separations On my end, not much done since the hackathon. While there, we had discussed a scheduler and mta refactor with Eric Faurot. The design was not optimal, there was a layer violation with the scheduler taking looks at the queue during initialization, etc ...

During the hackathon I moved the queue initialization to the queue process. I didn't commit it but Eric cloned my repository, improved it further and committed the result a few days ago. I then realized that since the scheduler no longer had to look at the queue, it could be chrooted to /var/empty instead of /var/spool/smtpd to further restrict it in case of a catastrophe.

Transactional scheduling At the end of the hackathon I also started working on a new scheduler logic, it started with just a few simplifications but then I realized that making the scheduler transactionnal would help us reach the design we had discussed.

The scheduler initially consisted of a set of trees and a list sorted by schedule time. An envelope would be part of the trees allowing fast lookups for internal purposes, and the scheduler would just have to look at the first envelope of the list to see if something could be scheduled.

This had a side effect that as soon as pushing an envelope in the scheduler, it could be scheduled. That is not always desireable as sometimes we have multiple recipients for a same message / destination and we start scheduling before we get a chance to check that other recipients can be part of the same SMTP transaction. This would lead to inefficient MTA sessions as we would potentially connect multiple times to the same host to deliver the same message for a handful of different recipients.

The transactional scheduler uses a set of tree and linear queue BUT when it receives envelopes it doesn't add them to these structures. Instead it adds them to a temporary set of structures, the changeset, which can then be atomically committed to be merged to the scheduling structures, or rollbacked depending on errors we encounter. That ensures that whenever we commit and make envelopes scheduleable, we are able to send as many envelopes as possible in one go to optimize MTA sessions.

When I left the hackathon this was partly done but still required lots of work and improvements which Eric completed by himself and finished just a few days ago. It works perfect, it is a HUGE step forward which will make OpenSMTPD far more efficient.

New MTA logic While he was at it, he also reworked entirely the MTA logic. I'm not familiar with the new code yet but what I can tell from it is that it's been simplified, cleaned and that it now supports some long awaited features like connection-caching to avoid the overhead of round-trips by reusing existing connections when different messages are heading to the same host.

Actually, it is a bit more complex than that because we no longer deal with hosts but rather with "routes" which allows reusing the same connection when talking to a MX that is shared by multiple domains (ie: poolp.org / opensmptd.org share mx1.poolp.org, so if we have a connection to mx1.poolp.org already opened, mails for both domains will take advantage of it).

There is still work to do in that area but this is also another HUGE step forward on a bit of code that we had planned to rework for months.

Stats backends Statistics used to be stored as size_t fields of fixed structures residing in a chunk of shared mmap-ed memory.

This was highly annoying as adding new counters would require changing structures, while the shared memory was in complete opposition with our separated processes design. To make things more annoying even, some of the statistics were used to alter logic when they were supposed to be informative only ... and the statistics were necessarily non-persistent as they were kept in RAM.

I came up with the stat_backend API to allow writing custom statistic backends and implement the ramstat backend to provide the same feature as before.

The main difference lies in the fact that the stats are pushed and never fetched so the logic can never use them; the keys are dynamic so we can generate very precise statistics by creating keys as dynamic strings containing msgid, evpid, etc ... and since they are only pushed, we can centralize them to a single process, have them pushed by imsg and remove the need for shared memory.

I also wrote a sqlitestat backend which is not committed but that does work and that allows providing persistent statistics that survive accross restarts so that admins can generate graphs and whatnot.

Ideas float to write stat backends to send to collecting daemons, snmpd, etc ... but the next plan for the stat API is to extend it to provide different type of statistics so we can have ratios, times, etc...

Bounces grouping OpenSMPTD tries to avoid bounces by rejecting early at session time. Sometimes this is not doable as we accept a recipient and the delivery fails later, after a ~/.forward for example, and we have to generate a bounce for a failed recipient.

Initially OpenSMTPD grouped all failed recipients coming from same sender as part of the same bounce message so that sender would only receive one report with all failed recipients ... but that feature got broken at some point in time leading to as many bounces as failed recipients.

It was conceptually simple to implement but it could not easily be done until queue/scheduler interaction logic was rewritten. Now the queue handles the bounce re-enqueueing. The grouping is simply achieved by delaying the actual bounce a bit in case another bounce for the same message arrives shortly after. Guess who designed and implemented it ? Yup, Eric !

Loop detection Loop detection was performed by the scheduler, which was a layer violation and which went in the way of Charles' attempt at the compression experiment.

We got rid of it altogether and reimplemented it at the proper place. I added a Delivered-To loop detection at the MDA level while Eric plugged the Received loop detection at the MTA level.

Backup MX Oh and since Eric was bored with all these "little" reworks, he implemented today the also long awaited "backup MX" features.

Until now, OpenSMTPD could not operate like a real backup MX, we could trick it into doing a similar job by using a set of "relay via" rules like I did on mx2.poolp.org:

accept for domain "poolp.org" relay via "mx1.poolp.org"

But this would be a pain to maintain when many backup MX with different priorities are declared in the zone and it would not honour the priorities.

It is now possible to do:

accept for domain "poolp.org" relay backup "mx2.poolp.org"

which will declare ourselves as the mx2.poolp.org MX backup for domain poolp.org and allow OpenSMTPD to perform a MX lookup and try to find a MX with a higher priority to hand the mail over. The diff is so small that it's a shame we didn't have this before :-p

Getting closer to release OpenSMTPD is now in an almost production-ready state, in a better shape than ever, and we're looking forward to receive as many tests and bug reports as possible to ensure that it's rock solid.

You can submit your bug reports on our github tracker, feel free to join us on IRC at #OpenSMTPD @ freenode and subscribe to our mailing list misc@opensmtpd.org by sending a mail with subject [misc] subscribe.

Cheers !

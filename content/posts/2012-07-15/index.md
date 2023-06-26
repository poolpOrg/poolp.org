---
title: "g2k12: OpenBSD hackathon - part II"
date: 2012-07-15 21:43:58
category: OpenBSD
authors:
 - Gilles Chehade
categories:
 - technology
---

Yesterday, I came back from Budapest and was too tired to write anything, but here's a quick summary of all the features that have been implemented with regard to OpenSMTPD by Eric, Charles and I.

First of all, Eric [eric@] has moved ASR, his asynchronous resolver, to OpenBSD's libc. This is a huge step forward that will allow OpenSMTPD to contain less code, while providing a better coverage of the resolver code. I really hope it catches on other systems which can easily get the resolver from OpenBSD and provide saner asynchronous resolving.

I cleaned up the scheduler code a bit by introducing a new file to separate the scheduler code from the scheduler API. This brings the scheduler code closer to the logic we have for other parts of OpenSMTPD where we support custom backends.

When OpenSMTPD receives a mail for a local address with a '+' in it, it ignores the second part so that for example 'gilles+foo@poolp.org' is really delivered to 'gilles'. Charles [chl@] added support for Maildir tagging, a very nice feature that had been requested by a user recently and which allows OpenSMTPD to automatically create directories within a Maildir to categorize incoming mails. So now, if you setup OpenSMTPD to deliver to a maildir AND OpenSMTPD receives a mail for a local address with a '+' in it, it will extract the second part and use it as a subdirectory of the Maildir.

I finally plugged the relay URLS in parse.y allowing simpler smtpd.conf syntax when using "relay via" rules, but also preparing the way for upcoming relay maps. I had discussed this on this blog a while ago, but the idea is that you can now express relays using URL like 'smtp://mx1.poolp.org', 'tls://mail.poolp.org', etc ... and soon you will be able to provide a map with multiple URL and let OpenSMTPD deliver from either.

OpenSMTPD initially supported multiple queues and as time passed by, we removed them and came up with a simpler design that was more effective. The code to differentiate between queues was still all over the place and both Eric and Charles worked on getting rid of it. Finally, Charles managed to remove the last pieces and kill the latest bits of it.

With Eric, we decided to reduce the number of buckets from 0xfff to 0xff as we observed performances degradations during enqueuing when the queue had thousands of buckets. It was not a matter of envelopes in the buckets but really a degradation caused by the number of entries in the first level directory.

I simplified the scheduler loop logic to make it much much simpler than before. It cannot be much simpler than now: check next envelope schedule time -> schedule / sleep until schedulable. This may sound too simple for real world, but it is actually because we moved the smartness outside of the scheduling loop and it will deserve a post by itself. I have only committed the scheduler loop simplification for now, the rest is coming soon ;-)

There was a problem in the scheduler logic loop which was caused by an optimization I wrote and the fact that our signals are handled asynchronously. This would not be visible most of the time, but if you ever had a very busy queue with tons of messages being schedulable, then the scheduler could not be interrupted with a signal. Sending CTRL-C to OpenSMTPD would kill all processes excepted the scheduler which would keep on as long it had a schedulable envelope before catching the signal. This has been fixed by forcing the scheduler to exit the scheduling loop after each envelope but setting a new scheduling loop call right away. It allows OpenSMTPD to handle the signal and resume its loop.

I added a TRACE_SCHEDULER trace so that we can easily log scheduler actions using log_trace() and enable/disable them at will without rebuilding. This was prompted by Eric asking me if I still had to keep the ugly output or if I was done with debugging the scheduler ;-)

Charles added support for literal inet addresses, allowing a user to send mail from or to a user at a specific internet address (gilles@[192.168.1.2], for example). This was requested a while ago, not to hard to accomplish but it was still sitting in our bug tracker.

He also restricted the character set allowed in an email address. We're supposed to handle a shitload of characters including [!#$&'*/=?^`{|}] as well as any other one inside double quotes. This is so completely fucked up that we decided we don't want to support them and deal with being scared that one could leverage a very smart attack exploiting a user part in a shell filter. If your email address contains one of the forbidden characters, either come up with a good rationale, or use another MTA. For other people, you will probably not notice because no one sane would use '#' or '{' in an email address.

Finally, the hackathon has been much more productive as Eric and I shared the same room and got to discuss some refactors during hours after our hacking sessions at the hackroom. We have designed the new scheduler and MTA refactors which are going to be very very impressive, I can't wait to write about it when it's done [yes I have started working on it, and Eric cloned my branch and worked on it in the plane] ;-)

Off to sleep, I have to recover now and wake up for my real work in a few hours !

---
title: "OpenSMTPD: gentlemen, fasten your seatbelt"
date: 2013-01-25 23:01:47
category: OpenSMTPD
authors:
 - Gilles Chehade
categories:
 - technology
---

OHAI,

Lately we have stabilized and decided that we're okay with the features we have for our first release, this time for good, so we focused on making sure that we didn't introduce regressions and that we were rock solid.

We stressed incoming and outgoing path and found areas of improvements. This will be a long-standing task, but we have already improved memory usage considerably under high-load, we're pretty happy with the result.

All of the following has been committed, we published snapshots and we sync-ed the OpenBSD tree with our own repository so that OpenBSD-current now has all of the features, improvements and bug fixes we came up with in the last few weeks (or months for some stuff we didn't backport).

Anyway, here goes a summary and I'm off to bed as I'm sick as shit.

fix a descriptor leak

We were told of a crash which appeared to be coming from a descriptor leak. On Linux, lsof would display that message files were removed while the transfer process still had a descriptor opened. We tracked the issue and spotted that it would happen because of a missing condition in the transfer process where the message file would not be closed if recipients were rejected and the session was reused to send another message. Two liner fix.

SSL code cleanup

An OpenBSD user reported a problem with ldapd's ssl.c where the prime used for DH parameters was 512 bits which is short enough by today's standards to be rejected by OpenLDAP's client. OpenSMTPD's ssl.c has been changed a long time ago to bump this prime to a 1024 bits prime, but ldapd's ssl.c was actually a copy of OpenSMTPD's ssl.c from two years ago.

The desynchronization of ssl.c accross OpenBSD daemons has annoyed me for a long time but it occured to me that maintaining this gap would be causing further divergences in the future as reyk@ moves OpenIKED and relayd forward. After a quick chat with him I started creating a daemon-agnostic version of ssl.c.

There is still work to do in that area, but OpenSMTPD now comes with a ssl_privsep.c that is equivalent to that of relayd; a ssl.c file that no longer knows of any smtpd specific structures and which can be shared with other daemons; a ssl_smtpd.c that contains the smtpd-specific bits.

Note that ssl.c doesn't contain new code, this was only a rework of the interfaces to allow it to be shareable with different daemons.

Runtime tracing and profiling

I have added a feature that I've been wanting for a long time: activating traces at runtime without a daemon restart.

Say a user suddenly observes that connections to a remote host fails, he can now simply type: smtpctl trace transfer

To obtain a real-time view of the sessions as they take place with remote hosts.

There are many trace subsystems, for incoming connections, outgoing connections, msg exchanges accross processes, etc, etc, ... read the man page ;-)

While at it, if you need to verify bottlenecks OpenSMTPD also supports real-time profiling of imsg and queue using: smtpctl profile imsg and smtpctl profile queue

Both tracing and profiling can be turned on and off at runtime.

Improve memory use

Eric cleaned up some code to avoid passing envelopes when we could pass evpid instead. This prevents OpenSMTPD from accumulating data in the inter-process buffers (an evpid is 64bits, an envelope structure is several kbytes).

For the places where the envelope really needs to be passed, he suggested that we send a compressed version. I proposed that we use the ASCII envelope conversion as we know it works given that it's already used for disk-based envelopes. The ascii envelopes allow compressing the envelope quite a lot as instead of passing a large datastructure with partially used large fields, we pass the ascii representation that can be as small as a hundred bytes.

We should now be able to cope better in very heavily loaded situations ;-)

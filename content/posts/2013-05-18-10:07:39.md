---
title: "OpenSMTPD improvements summary"
date: 2013-12-14 21:36:40
category: OpenSMTPD
author: Gilles Chehade
---

Hey,

No OpenSMTPD news since almost a month... time to break the trend !

As far as I'm concerned, I've been busy with other work but still a git log shows that between Eric and I we have accumulated quite a few things during this month. Below is a short non-exhaustive summary of changes and news.

All of it is already pushed to our Github mirror and part of the snapshots that were published yesterday. I'd like to stress out that if you can run bleeding edge stuff, you REALLY want to test the latest snapshot as it has some very invasive improvements which we want to make sure are "bug-free" before the next big release in a few months.

We have also fixed some minor bugs that were reported to us since 5.3.1 and discovered/smashed some other bugs on our own. The bugfixes are part of the snapshots and were backported to the 5.3.2 stable release that should take place next week.

MTA improvements

OpenSMTPD has various "logical" optimizations to try and optimize its transfers. It will create several, but not too many, connections to a destination; it will detect that two domains share a same set of MX; it will spread load across MX of same priority; it will group messages on connections and group envelopes for messages; proceed to quadratic increase of delay upon temporary failures, and much more ...

This all performs pretty well with regular use-cases but we have discovered a use-case that was sub-optimal: dealing with a large list of single recipient messages going to the same destination. For example, the misc@opensmtpd.org list uses DKIM-proxy and signs all outgoing mail for each single recipient. Then, our code will optimize that out by grouping these recipients so that their mail is delivered through the same connection. So far, so good.

Problem occurs if the other side uses grey-listing and rejects the recipients with 451 instead of dropping the connection with a 421 at first recipient: we keep reusing the connection to submit envelopes which we know are likely to fail. With a small list like ours, that's not a problem as we will fail a few dozens envelopes and the quadratic delay will cause things to sort themselves after a few bounces but ...

... now imagine your list is that of a company with hundreds of thousands of recipients and the same happens. By the time you're done submitting your list of recipients, it's already time to try out the first of the list. The delay between your last recipient and the retry of first one causes you to fail to pass the grey-listing again, and you enter another loop of failures... It eventually sorts itself out when the quadratic delay is so large that the delay between your last submitted recipient and the next retry of the first recipient respects the grey-listing imposed delay.

Most people will never hit this case, but those who hit it will see their mails delivered after much more time than they really should (ie: your customers could receive their mail 12 hours later when they could be delivered after 4 hours.

Eric and I came up with two distinct mechanisms to improve the situation:

First of all, instead of starting several concurrent connections, OpenSMTPD will first try to establish one and ensure that it is working before creating new ones progressively. If an issue prevents connections from being established and sessions from being started, the MTA layer will penalize the route and prevent it from being used for a small period. If new envelopes arrive for that destination, they are delayed until the route is considered usable again.

Then, another mechanism kicks in. If too many recipients are rejected in a row, instead of trying the entire list OpenSMTPD will consider that they are all temporarily failed and mark the route temporarily unusable. This will cause the deliveries to be reattempted later, just like when a grey-listing mechanism is in place, without hammering the host with tons of recipients that are most likely going to fail. The ones that were rejected are penalized and their delay is slightly increased compared to those that OpenSMTPD failed. With this, in most situations people will not hit the case; in case we really face the problem, we will avoid hammering the server that kindly asked us to try again later several times; and if this was just a coincidence, legitimate envelopes are only deferred for 400 seconds.

Finally, Eric spotted a bug that could cause a wrong error to be detected and imposing a long delay on some deliveries to hosts that advertise both IPv4 and IPv6 MX records.

smtpctl show routes

With all this routing logic added, we needed a way to be able to better understand what is happening at the MTA level.

Log files have been extended and they provide a precise history, however it's always nice to have a tool that provides real-time display of the state of something you're investigating. So Eric added a "smtpctl show routes" subcommand that display the routing informations that are currently in used by MTA. It displays active routes, routes that have been penalized and when their penalty is going to expire and the route be usable again. This is a shortened output of the command right when I sent the "new snapshot published" mail to our mailing list, it displays the route in use, the state of the route, the number of connections currently active to that route, the penalty level and the timeout before a route is usable again after we detected a failure:


smtpctl show routes
88.190.237.114 <-> 144.76.32.53 (arati.lostca.se) ---- nconn=1 penalty=0 timeout=- 88.190.237.114 <-> 213.41.253.7 (gw.zefyris.com) ---- nconn=1 penalty=0 timeout=- 88.190.237.114 <-> 213.165.67.115 (mx01.gmx.net) --Q- nconn=0 penalty=0 timeout=2s 88.190.237.114 <-> 213.165.67.97 (mx01.gmx.net) --Q- nconn=0 penalty=0 timeout=1s 88.190.237.114 <-> 213.165.67.99 (mx00.gmx.net) --Q- nconn=0 penalty=0 timeout=2s [...] about a hundred entries [...] 88.190.237.114 <-> 72.249.41.52 (lavabit.com) ---- nconn=1 penalty=0 timeout=- 88.190.237.114 <-> 87.98.163.156 (ns1.huynguyen.org) N--- nconn=1 penalty=0 timeout=- 88.190.237.114 <-> 95.130.12.24 (host1.trois-doubles.net) --Q- nconn=0 penalty=0 timeout=1s

Truly perfect for troubleshooting remote delivery issues !

SSL issues

Eric ran into an issue where the daemon would crash when accepting a SSL connection from a client that didn't do what we expected.

I tried to reproduce but for some reason, it would not happen on OpenBSD, only on Linux. I spent a while troubleshooting and eventually found a subtle issue with our handling of I/O events.

During a non-SSL session, OpenSMTPD enables and disables events as the session progresses since a SMTP session is a succession of command/responses where it is not expected to read a command when it has to send a response, and not expected to send a response when it is waiting for a client command. This works fine.

During a SSL session, however, renegociation can take place and expect the server to write while the server is in read mode, and the other way around. To cope with this, the code wrongfully enabled both read/write events, causing the daemon to eventually end up in a weird state leading to either a hang until the client disconnects or a fatal() being hit.

In addition, we never set the client socket non-blocking, even though the logic around expected it, because it was assumed that we didn't really need to do so due to the event-based approach. However, I found ways to cause SSL_accept() to be triggered BUT block with a misbehaving client that didn't complete the SSL "handshake". Other related issues could cause a SSL client to "block" other concurrent sessions until it timedout.

All these were fixed by making the client socket non-blocking and by being more strict with our event handling, none of the bugs could be reproduce on Linux after the fixes while no regressions were observed on OpenBSD.

Portable improvements

An OpenBSD hacker complained that on Linux authentication didn't work.

In fact, it would only work when built with PAM (which most Linux users have been doing so far) as the default authentication method in -portable is getpwnam(3) which doesn't return the passwd field on Linux even when called as root. He suggested we implement a getspnam(3) method and provided the code which we merged.

I spent a while tracking a crash on FreeBSD-STABLE that didn't trigger on FreeBSD-RELEASE and which was caused by their import of NetBSD's strnvis(3) which has ... swapped parameters compared to OpenBSD's strnvis(3). This caused an invalid read that lead to a segv. We marked strnvis(3) as broken on NetBSD and FreeBSD to ensure that -portable builds with the compat glue until we deal with it differently.

Charles has also worked on some fixes for OSX which we broke with all our new code :-)

smtpd -dvn improvements

The -d option was supposed to start it in foreground, while the -v option was supposed to start it verbose. In practice, this was not the case as -d started it in foreground AND verbose, while -v was essentially no-op. Oh and due to the log.[ch] code we inherited, it was not possible to start in foreground without verbose, and it was not possible for us to activate traces without being verbose too.

Some people have been willing to start it in foreground without the verbose mode because they want to run it with a process manager and let it handle god knows what. Since it's a common request and our behaviour was not consistent with the documentation, I decided to rework that and make it behave like expected.

While at it, I fixed the requirement for verbose mode to enable traces so that now it is possible to:

smtpd -d -> start smtpd in foreground but not verbose smtpd -v -> start smtpd in background but verbose smtpd -dv -> foreground and verbose

and no matter if -d and/or -v is specified, "smtpctl trace " will activate traces at runtime which will either appear at foreground or be logged to syslog as mail.debug.

I also received two complaints the same day that 'smtpd -n' checked the configuration file for validity but didn't detect SSL certs errors as they are loaded after the configuration file parsing (on purpose). I slightly reworked the code so that we still load them after the configuration is parsed but to detect errors in the 'smtpd -n' case.

Assorted bugfixes and improvements

Table lookups where not always folding key to lowercase which could lead to failed lookups for existing entries.

The "as" feature allows a rule to override the sender at SMTP-level (not in the message itself) and a missing check would lead it to override the sender when generating a bounce. Since the override can be partial (ie: @domain, or user without domain) and the sender for a bounce is an empty address, this could lead to invalid addresses (ie: MAIL FROM: , MAIL FROM: <@poolp.org>) causing a reject of the bounce by remote host.

Also, on the bounce side, the delivery_mbox backend didn't assume that a sender could be empty in the bounce case and it would call the mailer.local third-party utility with an empty sender. That utility would write to the mbox with the usual delimiter "From

" but with an empty address causing the following delimiter to be generated "From " (two spaces before date). Some MUA that do not check for "From " but for "From
" as a delimiter would fail to detect the bounces in the mbox.
As mentionned before, the table API and queue API now allow people to write standalone backends that are started by OpenSMTPD. They can have their own dependencies and be maintained out of the official tree, just as any other tool. To help with this, various APIs have been improved and changed slightly to ease this mechanism.

A user has improved local LMTP delivery by adding support for unix sockets so that OpenSMTPD can talk LMTP to a unix socket setup by another daemon (Dovecot seems to be the trend ;-)

Another user reported a hang after a while and Eric traced it back to a missing check that could lead to an EOF not being detected. The mda/mta sessions would be in a broken state for a session and fail to deliver the message, causing it to stay in queue indefinitely. It was rare enough that we never hit it as it was ... specific to the size of a message ...

A third user reported a "reject" rule not working as expected and we spotted a typo in the config file parser causing "reject" rules to fail when used with the "sender" parameter.

A poolp user mentionned the '+' delimiter not working for one of his domains while it worked with his @poolp.org mail account. I realized that the '+' delimiter handling was not implemented for virtual domains, only for primary domains... so I added support for it.

Finally, an OpenBSD hacker reported that he had issues with his ~/.forward file causing recipient to be rejected when the file is empty. This was indeed the case and anoter user provided a diff to fix it.

That's all folks !

For now ;-)

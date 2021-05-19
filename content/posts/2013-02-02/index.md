---
title: "OpenSMTPD: minor fixes + preparing release"
date: 2013-02-02 12:54:36
category: OpenSMTPD
authors:
 - Gilles Chehade
---

OHAI,

We had planned to slow down a bit to prepare for release: no more feature, no more invasive changes until we tag something out of which we can publish a release. I then discussed with Bret Lambert (blambert@) and he asked me if we used some kind of code scanner to detect defects in our code.

We do use the clang static analyzer every now and then, the last time being a couple week ago, and we spend a few minutes cleaning up the defects we believe to be real issues.

Bret then asked me if I had ever tried the scanner at Coverity and told me I should try to get our codebase scanned with it. I recalled seeing a summary of issues uncovered by their scanner on Apache a while ago and it was very impressive, so I asked to register OpenSMTPD and was happy when they confirmed I could use their scanner without having to sign any NDA, terms of services or random lawyer documents [rare and appreciable] :-)

So here's a summary of this week's work, all of it has been committed and merged in the OpenBSD tree and Github mirror. The latest snapshot is up to date.

TRACE_LOOKUP

A user reported on IRC that he had an issue with aliases. The issue turned out to be a user error but it prompted me into improving the aliases expansion logging.

Until now, to follow the aliases expansion logic, one had to run in debug mode to get to see the tons of log_debug() we have there. I introduced TRACE_LOOKUP and replaced the log_debug() with log_trace() which allow to turn the logging on or off at runtime with the command smtpctl trace lookup

MTA logging improvement

Eric and I thought that we didn't log enough information when relaying. The incoming path logs many details about sessions, but the outgoing path logged almost exclusively SSL-related details. So Eric spent some time adding proper logging to te MTA process, which should make it much easier for an admin to understand what happens by reading logs ;-)

Improved memory usage

Eric introduced the mta_envelope to represent an envelope within the mta process. This allows keeping in memory ONLY the information needed by the mta and shrinks a lot the memory usage when the daemon is under a lot of stress.

He also introduced the mda_envelope structure to achieve the same for the mda process and raised limits to allow more concurrent local deliveries to take place.

Bug fixes

Finally, most of the work this week has been spent on fixing bugs reported by the code scanner provided by Coverity.

It doesn't run on OpenBSD but since it runs on Linux and our portable version is very close to the native version, we could run the test on Linux and fix on OpenBSD.

We didn't find anything really scary, probably due to the fact we already cleaned and removed scary code detected by Clang, but we did find lots of little issues that we wouldn't spot just from reading, even several times.

Amongst the issues, we found a descriptor leak, several small memory leaks, possible double-free, possible double-close, possible double-fclose, use-after-free, various missing bzero, dead code and null-derefs. Many of these would lead to a crash on OpenBSD but the reason we didn't experience them is that they happened in error code paths for the most part. We have fixed about 40 issues, there are 40 still, with several part of external code (mail.local, aldap, etc) and some we believe are false positives but require further reading with a clear mind. The scanner manages to find errors which makes you wonder if they really are, until you realize that it does a far better job at unrolling the code ;-)

This teaches us two lessons: find a way to better test error paths, run the scanner from Coverity regularly !

Anyways, all related commits are available from Github in case you're interested, as usual I'll document funky bugs here so stay tuned.

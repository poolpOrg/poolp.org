---
title: "OpenSMTPD released and upcoming filters preview"
date: 2018-11-03 17:27:00
category: OpenSMTPD
authors:
 - "gilles"
categories:
 - technology
---

{{< tldr >}}
    Filters have been a (the most ?) long awaited feature in OpenSMTPD.
    I finally committed most of the filters code to OpenBSD.
    There is still a bit of work required but the trickiest parts are done.
    This article describes how filters are implemented and what to expect.
{{< /tldr >}}


![Good news, everyone!](https://brillianceinsight.files.wordpress.com/2014/02/card-2-24-2014-1_thumb.jpg)

OpenSMTPD 6.4.0 was released !
--
We have [released OpenSMTPD 6.4.0 last week](https://opensmtpd.org/announces/release-6.4.0.txt) **without** filters.

I won't expand on the features in the 6.4 release as I already wrote about the configuration file changes,
the issues that required it and the refactors involved,
this was the one true major feature of the release.

One notable aspect though is that we dropped our support for OpenSSL in favor of LibreSSL,
and THAT I should expand upon ;-)


LibreSSL FTW !
--

OpenSMTPD was started long before LibreSSL and from the very first portable version,
we had to include ifdefs to accomodate the differences between the different OpenSSL versions across Linux distros:
some had SNI, others don't, some had GCM, others don't.

When LibreSSL was forked from OpenSSL after a large cleanup of the code,
we thought we'd just depend on that because it was cleaner and also we knew that this version had all the features we relied on.
Also, there was this plan for a libtls wrapper which would be much simpler to use and less error-prone than libssl.

It turned out that it wasn't possible because OpenSSL and LibreSSL couldn't coexist on many systems.
The fact that they shared the same library names (libssl, libcrypto),
and shared object versionning,
caused issues with the confused runtime link editor.
We decided to accomodate both,
keeping checks in the configure layer to detect what was being used, etc...
It was horrible.
It kept breaking every now and then due to either one making changes.
I had to deal with these breakages.
It was horrible.

Before the 6.4 release,
I did the portability build passes and yet again it broke on new OpenSSL conflicts with our grammar.
Three different people using different distros had sent me three different diffs which fixed the issues for them,
but which I couldn't merge without creating a delta with the native branch which was not acceptable for us.
The more I tried fixing and the more I was irritated by this OpenSSL/LibreSSL hacks.

I did a bit of investigation and it turns out the issues that prevented them from coexisting were no longer relevant.
I verified by building, linking and running OpenSMTPD against LibreSSL on FreeBSD, Ubuntu, Debian, CentOS, ArchLinux and Fedora.
It is technically possible to make OpenSMTPD depend on LibreSSL without forcing distros to switch from OpenSSL to LibreSSL,
so there's no reason for us not to depend on LibreSSL anymore.

This allowed me to start removing ifdefs from our code,
removing useless tests from the configure.ac and assume that we have the features we need for all systems.

This doesn't mean that people can't link OpenSMTPD against OpenSSL,
it just means that they get to handle the diffs to their own version,
the extra work of maintaining OpenSSL goes out of our hands.


Filters
--
We started working on filters years ago,
I even [wrote an article about them in 2014](https://poolp.org/posts/2014-12-12/the-state-of-filters/).

A first implementation was written.
After a while, it became clear that the design was wrong and causing unfixable bugs in some filters.
I decided to pull the plug on that attempt and we removed all of the code.

A year ago,
[during EuroBSDCon 2017](https://youtu.be/wnhvn1rsXR8?t=3056),
I mentionned the filtering daemon we had worked on and which solved these limitations.
But as OpenBSD 6.4 release was getting closer,
I started to dislike that idea because I believed it was not addressing the proper problem:
we had came up with a working solution, not the best solution to the problem.
The more I spent thinking about this,
the more the problem looked different from what we initially modeled.

I discussed my concerns with Eric,
told him that I thought we had made a mistake in modeling filters,
that by thinking them differently we could come up with a much simpler solution,
and somehow managed to convince him I was right :-)

Given we took so much time already,
it made sense delaying for a release so we would be confident about our technical choice.

I worked on a proof of concept for the new model and a week later I had filters working on my laptop,
a convinced eric and...
many months to make it bright and shiny because it was obviously not going to make it into 6.4



Event reports and filter request
--
The new filters are modeled around the notions of event reports and filter requests.

During the lifetime of a session,
an SMTP engine will report various events such as the beginning of a new connection,
the negotiation of ciphers during the TLS handshake or even the beginning of an SMTP transaction with MAIL FROM.
These events help the SMTP engine build a state for a session and determine what is acceptable or not from a client at a given time.

Some filters may not need a state, they may operate on the parameter to a command and that's all.
Other filters however may need to build a state for a session,
not necessarily the same exact and complete state as the SMTP engine,
but a state that makes sense given the work the filter will do.

The event reports are _informative_ messages,
which a filter may decide to process or not,
and that encompass ALL of the SMTP events a session goes through.
It is possible to replay an entire SMTP session based on these reports.

The filter requests however are not informative.
A filter will be configured to handle a particular filtering phase and will receive filter requests for that phase,
containing the phase and the parameter to filter,
it will then be REQUIRED to answer the request with an action to take.

Again,
simple filters may only deal with filter requests,
more complex filters may process the event reports to learn about sessions before dealing with filter requests.


Event processors
--

Event processors are standalone executables which ... process event reports and filter requests in an infinite loop.

OpenSMTPD executes them at startup and sets up an environment so they behave like traditional Unix-filters:
they read input from stdin and write output to stdout.

Both event reports and filter requests will consist of single lines,
each consisting of multiple fields separated by a pipe symbol `|`.
All lines will hold a generic set of fields such as event type and session identifier,
as well as a set of event-specific fields such as an ip address for connection event,
an email address for mail-from event, etc...

Event reports will expect no response,
meaning that for a line read on stdin there will be no line written on stdout,
whereas filter requests will expect a response,
meaning that for a line read on stdin there will be one line written to stdout.

The simplest processor to write is one reading events to log them:
```sh
#! /bin/sh
#

while read line;
do
	echo $line >> /tmp/output.log
done
```

If you have ever parsed OpenSMTPD logs to inject them into an ELK or similar,
you should now be in love with the events reporting feature.

Filters are not much more complex,
the only difference is that `$line` needs to be split to extract the session identifier,
and the processor needs to write back its decision.

The simplest filtering processor to write is one that accepts every filter-request:
```sh
#! /bin/sh
#

while read line;
do
        if echo $line | grep '^filter-request|'; then
		SESSION=`echo $line | cut -d\| -f4`
		echo "filter-response|${SESSION}|proceed"
	fi
						
done
```

Note that while this example shows a synchronous protocol where a filter-request gets an immediate answer,
the implementation does not impose that limitation.
There should be a filter-response for each filter-request,
but they do not need to happen right away and they do not need to happen in the same sequence order.

Now that you're all sold on this solution, here are the bonus advantages:

OpenSMTPD only knows that it has to execute the processor and write to its stdin.
There are no dependencies and no limitations to what language a processor is written in,
a postmaster does not have to learn C to write a filter but can effectively hack something fast with a few lines of shell or python.

The events processors are forked:
they do not share the same memory space as OpenSMTPD,
they do not share the same memory space as each other,
and on systems that provide randomized memory layout they do not even share the same memory layout.
On OpenBSD, each processor could benefit from its own `pledge()` and its own `unveil()`.
A compromise of either one does not imply the compromission of the others or of the daemon.

In addition, they can be executed with different privileges and including chrooted() to a particular directory:
```
proc filter1 "/tmp/filter1.sh" user _filter1
proc filter2 "/tmp/filter2.sh" group _filter2
proc filter3 "/tmp/filter3.sh" user _filter3 group gilles
proc filter4 "/filter4" user _filter4 chroot "/tmp"
```


Event proc
--
Event processors can receive event reports for each events generated by the SMTP engine (for now).

This allows two things.
First,
it allows fine-grained reporting to specialized software that can extract metrics,
generate fancy dashboards and such.

Until now,
people had to try and extract this information from our log files which were meant to be consumed by humans primarily.
The event logs provides all of the useful informations in a format that can be easily parsed by scripts.
Converting these entries to the proper format for injecting in a timeseries database or similar becomes trivial.

Then,
some filters may be able to work without any context by looking solely at the current command,
but in many cases a decision at a particular phase may be the consequence of decisions at earlier phases,
and filters can consume these event reports to build their own view of the state of sessions.

Here is a sample of event reports:
```
report|smtp-in|link-connect|1541271219|3189ac6874354895|localhost|127.0.0.1:41564|127.0.0.1:25
report|smtp-in|protocol-server|1541271219|3189ac6874354895|220 poolp.org ESMTP OpenSMTPD
report|smtp-in|protocol-client|1541271222|3189ac6874354895|helo localhost
report|smtp-in|protocol-server|1541271222|3189ac6874354895|250 poolp.org Hello localhost [127.0.0.1], pleased to meet you
report|smtp-in|link-disconnect|1541271224|3189ac6874354895
```

And this is how you define an event proc in smtpd.conf:
```
proc my_event_proc "/usr/local/bin/script.sh"

report smtp on my_event_proc

```

This gets OpenSMTPD to run the script at startup and to pipe the event reports to its stdin.


Filter proc
--
Filter processors are very similar to event processors in how they work,
excepted that they are registered for specific filtering phases:

```
proc my_filter_proc "/usr/local/bin/script.sh"

# line below can be uncommented if filters needs event reports to build a state
# report smtp on my_filter_proc
filter smtp helo on my_filter_proc
filter smtp ehlo on my_filter_proc
filter smtp mail-from on my_filter_proc
filter smtp rcpt-to on my_filter_proc

```

The filters will receive phase-specific parameters:

```
filter-request|smtp-in|ehlo|e68808ff61812a6f|laptop.home|localhost
filter-request|smtp-in|mail-from|e68808ff61812a6f|<gilles@laptop.home>
filter-request|smtp-in|rcpt-to|e68808ff61812a6f|<gilles@laptop.home>
filter-request|smtp-in|ehlo|e688090368c275a3|laptop.home|localhost
filter-request|smtp-in|mail-from|e688090368c275a3|<gilles@laptop.home>
filter-request|smtp-in|rcpt-to|e688090368c275a3|<gilles@laptop.home>

```

To which they will have to reply with a decision:
```
filter-response|e68808ff61812a6f|proceed
filter-response|e68808ff61812a6f|rewrite|BLEH
filter-response|e68808ff61812a6f|reject|550 go away
filter-response|e68808ff61812a6f|disconnect|550 go away
```

The only possible decisions are `proceed`, `rewrite`, `reject` or `disconnect`.


Builtin filters
--
Filter proc are very useful when a decision relies on anything else than the parameter to the command being filtered,
when it depends on information gathered from previous events,
or even when it depends on complex logic involving multiple lookups.

In many cases though,
a decision to filter may rely solely on the current command and a simple table lookup.
For instance,
one may want to reject a HELO hostname that's not part of a table,
or reject a MAIL FROM that matches a regular expression.

For these cases,
OpenSMTPD provides a small set of builtin filters which are fairly generic to be applied to all phases.
```
table helo-reject { "foobar", "barbaz", "bazqux" }
table helo-reject-regex { "^f[oO]o[bB]ar$" }

filter smtp helo check-table <helo-reject> reject "550 go away"
filter smtp helo check-regex "^go\-away$" reject "550 go away"
filter smtp helo check-regex <helo-reject-regex> reject "550 go away"
```

We will also implement some very limited additional phase-specific builtin filters that cover common use-cases.
OpenSMTPD performs a reverse DNS lookup on connect,
both connect and helo/ehlo filter phases commonly check reverse DNS,
so we already provided a `check-rdns` for these phases:

```
filter smtp connect check-rdns reject "550 you need a reverse DNS"
filter smtp ehlo check-rdns reject "550 your HELO hostname and rDNS mismatch"
```

There shouldn't be many builtin filters,
we don't expect more than a handful of them,
but they do exist so there's that ;-)


What next ?
--
This code has been committed to OpenBSD and will be available in OpenSMTPD 6.5.0,
which should happen sometime around April 2019.

I haven't documented anything yet because both configuration grammar and protocol are still going through lots of changes,
I would strongly suggest against playing with event reports and filter procs before March,
unless you're a developer and happy with having to make changes to your code every few days.

The only part that I have not committed yet is the filtering of DATA,
which still requires a bit of work but will be working and committed sometime in November.

Once I have this part done,
I'll implement a few filters for the use-cases I have such as filter-spf and filter-rspamd,
then it will be ok as far as I'm concerned.

My main work for the release cycle to come is finishing and polishing filters,
fixing the portable layer which has grown a monster,
and ultimately doing a full release cycle of...
cosmethic cleanup so code is pleasing to read ;-)

--- 
Comments: [https://github.com/poolpOrg/poolp.org/discussions/95](https://github.com/poolpOrg/poolp.org/discussions/95)

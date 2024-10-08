---
title: "October 2019 report: OpenSMTPD 6.6.0 release mostly"
date: 2019-10-26 16:13:00 +0200
authors:
 - "gilles"
categories:
 - technology
---

{{< tldr >}}
    - yay, surprise emergency hand surgery...
    - OpenSMTPD 6.6.0 was tagged and released, including portable version
    - Merged contributions to fix filter-rspamd bug with DKIM
    - Work resumed on 6.7.0 feature
    - An OpenSMTPD book is in the works
{{< /tldr >}}

How I landed in the emergencies
--
I accidentally cut my hand real bad early October and rushed to the nearest hospital.
They had a look,
told me it was superficial and required to be sutured,
so I was out a few hours later with a few stitches and pain killers.

A few days later,
my hand was swell so I rushed to another hospital that was specialized in hand wounds,
expecting a second advice and maybe some additional drugs.

Turns out the cut was not superficial at all,
it had hit joints and needed more than just a few stitches.
The stitches also lacked propre protection so the wound had infected.
I was told that I had to undergo immediate surgery because the infection was looking real bad,
and that I was lucky I took this seriously and came right away.

<center>
    <img src="/images/2019-10-26-hand.jpg">
</center>


So this is how my month of October was ruined.
It took me two weeks to be able to type back on a keyboard,
it is still painful and I'm still unable to practice music or martial arts as I can't close my fingers entirely.

Luckily,
I should be able to recover fully, but... it will take some time.

My work this month was obviously impacted,
**BUT** I managed to pull the OpenSMTPD 6.6.0 release and do some thinking work around redesigns planned for development cycle that just started.
I will discuss a bit my plan in this article.


OpenSMTPD 6.6.0 was released
--
I have published and [announced the OpenSMTPD 6.6.0 release](https://www.mail-archive.com/misc@opensmtpd.org/msg04725.html) today.

It is a very important release for us,
**probably one of the most important that happened in the last few years**,
as it brings the filter API and makes it possible to compete with other MTA by interfacing with a wide variety of third party tools.

It is now possible to do dnsbl,
interface with antispam solutions such as Rspamd or SpamAssassin,
perform DKIM signing and verifying at session time,
rejecting sessions that do not behave as we expect,
...

The API was released as an undocumented experimental feature.
This doesn't mean it is unstable,
the code has been used on the OpenSMTPD mail exchanger for over a year,
used by developers for at least as long,
and tested by users in the wild for several months.
The only reason it is considered experimental is that we didn't want to "release" a stable version of the API with easy access to documentation,
without having a chance to obtain a first round of feedback over the course of a release with a limited set of filters and developers ;-)


In addition to filters,
this is the first release that is considering LibreSSL as its target TLS library _while_ providing support for OpenSSL.
A lot of distributions remained with ancient OpenSMTPD release because of TLS library issues.
This release is going to make every distro and system out there able to catch up and use the latest OpenSMTPD release.


DKIM signing and verifying was fixed in filter-rspamd
--
Thanks to Reio Remma [@whataboutpereira](https://github.com/whataboutpereira),
a bug was fixed in filter-rspamd which would cause some mails to be incorrectly DKIM signed or verified.

The issue was that filters receive the raw SMTP lines during the DATA phase,
and the filter didn't account for the escaping of lines starting with a dot,
leading to some mails being incorrectly signed or verified.

His fix was committed to github and prompted me to issue a new release,
before updating the OpenBSD port as well.

I have received an ok to commit the fix to stable packages and will do so this week-end.


OpenSMTPD 6.7.x work
--
With my hand out of service,
I spent time thinking about bits that needed a redesign.

The smtp-out reporting was almost completed and didn't make it to this release,
so I will make sure it is completed very soon so it doesn't miss the next release.

The switch to libtls requires a standalone OpenSSL-based libtls interface to be written,
I have started taking a look at it too.
I already have a libtls-based OpenSMTPD in a branch,
but I don't want to switch to that until we know for sure that we'll be able to provide a portable release,
so hopefully for 6.7.x in six months and more realistically for 6.8.x in a year.

The MTA layer needs to be reworked and was discussed at lengths with Eric Faurot,
this would be a change transparent to users but which would make the daemon MUCH simpler for developers.
I hope we can pull that for 6.7.x but work has to start very soon to make this happen,
so we'll see in the next few weeks how this project evolves.

Finally,
I'd like to replace the current table API with one that's similar to the filter API.
The current API is a binary protocol relying on the imsg framework,
requiring C plumbing and making the writing of table backends limited to skilled developers.
I already discussed with Eric Faurot my feeling that it should be converted to a line-based protocol,
similar to filters,
which would allow backends to be written in any language,
providing the same benefits as filters in terms of privilege separation and memory isolation.
This would make the OpenSMTPD-extras repository deprecated,
but given that I'm the author for pretty much every addon there,
I think I can easily replace them with new interface counterparts.


An OpenSMTPD book is in the works
--
I had started writing a book about OpenSMTPD **years ago** but,
because huge reworks were in progress in both configuration and filter areas,
I kept delaying as I didn't want to write configuration bits that would have to be rewritten from scratch,
and I didn't want the book to lack a chapter about spam filtering.

With all of the work that has gone through OpenSMTPD in the last two years,
I feel that there's no longer a gap to fill and a very complete and useful book can be written.
I have therefore resumed work and currently have a PDF that's **well over a hundred pages** when fully built,
describing the history and design of OpenSMTPD,
large chunks of how SMTP and DNS work,
how to setup your MX,
how to configure various setups,
and much more.

<center>
    <img src="/images/2019-10-26-book.png">
</center>

Needless to say,
this is a LOT of work.

It will take months to complete the book and publish something I'll be happy with,
so if you're interested let me know and if you want to support me and help me get this out the sooner,
then you know how to show your love and buy me time :-)


What next ?
--
I have started writing a new article related to SMTP and reputation,
I will try to complete it next week and provide early access to my patrons.

Development cycle for OpenSMTPD as resumed,
I have a few tickets to tackle on Github and a few ideas that I have to dig,
but the table API rework is a very high priority thing to do for me.

I will also resume working on cleaning up the portability layer.


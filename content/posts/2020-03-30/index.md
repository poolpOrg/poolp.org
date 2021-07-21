---
title: "Anxiety, OpenBSD break, COVID-19 and resuming work"
date: 2020-03-30 14:01:00 +0200
category: opensource
authors:
 - Gilles Chehade
---
    TL;DR:
	- I took a break to deal with personal stuff
	- I'm taking a long break from OpenBSD for personal reasons
	- I may or may not have experienced COVID-19, who knows
	- resumed work on OpenSMTPD and other projects


This is a weird report
--
This is a weird report.

I'll mix a bit of personal info to provide some context as to why I decided to leave the OpenBSD project,
explain why this doesn't mean I won't be an active contributor,
and give some small insight into my upcoming work.


A bit of personal context
--
As some may have noticed,
I didn't write a report in February:
**I took a small break**.


In 2013,
my body started failing me with constant headaches and nausea.
I had weekly visits at my doctor to adapt non-working treatments,
which then led to weekly exams to rule out a serious illness and eventually led me to being admitted in hospital for biopsy,
endoscopy and other unpleasant tortures.
All of this resulted in the observation that there was no organic reason to my illness,
it was purely psychological.
I couldn't wrap my head around this because everything was apparently fine in my life,
but I was treated accordingly for a few weeks and my physical health got much better in no time.
The whole episode became a thing of the past after a while.


Late 2018,
[I was diagnosed with generalised anxiety disorder and alexithymia](https://poolp.org/posts/2019-06-02/happy-new-year-2019-a-personal-post/).
The general idea is that I'm pretty much constantly anxious but I'm also **unaware** of it most of the time:
I have trouble decoding my own emotions and the related physical sensations.
It is not uncommon for my better half to ask me if I'm feeling ok as she realizes I'm having an anxiety attack before/if I do.
This puts a lot of perspective on the events of 2013:
I don't realise my anxiety is becoming unsustainable until my body starts to dysfunction somehow for extended periods of time.


I've grown used to it and came up with fairly effective coping mechanisms.
In regular times,
most people don't know about this disorder even after knowing me for years,
but these are no regular times for me.
I had major changes in my life with the
[arrival of a baby](https://poolp.org/posts/2019-09-21/september-2019-report-jules-opensmtpd-6.6.0-upcoming-release-and-related-things/),
the learning of a bad news regarding a close family member,
and the doubts about staying at my daytime job.


My mind hasn't been at rest in a while and keeping things under control has a cost,
sadly I'm unable to properly assess when that cost is becoming too high... which brings me to the next topic.


Taking a break from OpenBSD
--
I've been an OpenBSD user since 1999 and an OpenBSD developer since 2008.
OpenBSD has literally been around me for half of my life now.


I've attended multiple hackathons,
met plenty of skilled hackers from across the globe and been part of a project I love.
A project that drove me into studying computer science and whose philosophy I'm still perfectly aligned with after all these years.
I spent **wonderful** years and I'm a bit sad that I have to put an end to it.


My anxiety disorder causes me to obsess a lot about my work,
and while I can easily put distance between my daytime job work and myself,
it is much harder for an opensource project I'm emotionally attached to.
I'm unable to disconnect and take a break,
I feel a disproportionate duty towards hackers and users.
I must not let people down and so I keep obsessing about what needs to be done and how it should be done.
I can never find enough time to do all of it,
so it leads to more anxiety and more obsession,
in a vicious circle.

The almost two months break I just took without booting an OpenBSD laptop was **a first in +10 years**.
I can't recall a single vacation or week-end where I didn't boot my laptop at least an hour to work on something.


I decided to quit because I don't believe I can be part of the project and work a few hours here and there,
disconnecting to take a break every now and then.
It's just not in my nature.
As long as I'll be part of the team,
I'll feel a duty towards it and will be unable to put distance when needed.
I have to deal with a lot of personal stuff these days and not being able to assess emotions correctly makes it hard to know my limits.
The last thing I want is to push myself too far and back into hospital for being too obsessed... about a hobby.


I told the OpenBSD people I was leaving in a lengthy mail late January IIRC,
and I received nothing but friendly replies.
I may not be an OpenBSD hacker anymore but that doesn't matter much,
**I met the coolest people**.



Does it mean that I will no longer contribute to OpenBSD/OpenSMTPD ?
--
Not at all, **I don't intend to ghost OpenBSD/OpenSMTPD**.

What it means is that **I'll be contributing as any other user** through diffs sent to specific hackers or to mailing lists.
This will make it easier for me to contribute on my own schedule and will leave to others the decision of what gets in and when.
Other hackers have done that in the past for diverse reasons,
I wouldn't be the first former hacker to send diffs to tech@.

It'll also make it easier for me to work on **_other_** stuff.
I have many other interests than but could never convince myself that my time was better spent elsewhere.
You know, duty and all.

As far as OpenSMTPD is concerned,
I'm currently sitting on diffs to convert it to `libtls` as well as other related features that are **waiting for the 6.7 release**.
I will also continue my work to improve the portable version which is outside of OpenBSD's scope and doesn't really attract developers besides me.

**I'll still be a main contributor, just not the lead on that project**.


Suspected COVID-19: achievement unlocked.
--
My wife and I were suspected COVID-19.
Symptoms were mild but between the fever, the weakness, the coughing and the caring for a baby,
it was not a pleasant experience.

**I'll give it 1 star, do not recommend**.

<center>
	<img src="/images/2020-03-30-covid.jpg" />
</center>

Resuming work
--
As I'm back from my break I don't have much work to talk about in this report,
but **I'm working on my personal projects this week** so expect stuff in the next one.

I have already synchronized OpenSMTPD portable with the OpenBSD tree so it carries commits that happened by other developers during my break.

I'm catching up on my `libtls` work in OpenSMTPD portable as it got interrupted by the advisory in January.
I need to dive back into that code and test that it works as expected.
I will also continue my work of cleanup in `openbsd-compat` which is bringing OpenSMTPD portable closer to a clean compat layer.

I'm currently working on **client-side** mail stuff.
I did a major fuckup with my mailbox and reinjected years of Trash and Junk to my Inbox.
This was +30k mails to **classify** and it inspired me to work on a tool which I'll continue working on and disclose when it is useable.

Finally,
I'll catch up on the MANY notifications I received on github and the tons of mails that I accumulated in my mailbox during my break.
I know there are pull requests on some filters and questions regarding OpenSMTPD.
I need to catch up on all of this.

Take care,
next report will be in the usual format.


---- 
Comments: [https://github.com/poolpOrg/poolp.org/discussions/118](https://github.com/poolpOrg/poolp.org/discussions/118)

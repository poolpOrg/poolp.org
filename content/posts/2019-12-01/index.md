---
title: "SPF-aware greylisting and filter-greylist"
date: 2019-12-01 10:11:00 +0200
authors:
 - "gilles"
categories:
 - technology
---
    TL;DR:
    - greylisting is a sound idea
    - yet it is not really practical today
    - people tend to disable it or find work-arounds
    - SPF-aware greylisting makes greylisting usable again


# Shout out to my sponsors &#x2764;&#xfe0f;

A **HUGE thanks** goes to my sponsors on [github](https://github.com/sponsors/poolpOrg)
and [patreon](https://www.patreon.com/gilles):
your continuous support is very much appreciated !

I created a Discord where I hang out and discuss my projects or sometimes screencast as I work on them.
Feel free to hop in if you want,
and feel free to do just like me and share thoughts as you work on your own projects there:
**this is a virtual hack room for anyone to join**: [https://discord.gg/YC6j4rbvSk](https://discord.gg/YC6j4rbvSk)

This is a fairly open server where you can ask random questions about a lot of topics,
not necessarily related to my own work.

SMTP failures in a nutshell
--
SMTP is a fail-safe protocol which attempts very hard to ensure that messages do not get lost once they are in transit.
Among the various mechanisms and requirements in place is the use of "Temporary Failures".

Basically,
an SMTP node has **two** final states for a mail.
**Either it is delivered**,
and the recipient has the mail,
**or it's not**,
and the recipient doesn't have the mail.
For the latter,
the SMTP requires that a node notifies the sender that an error occured either by rejecting during the SMTP transaction or by sending a `deferred bounce` (also known as `MAILER-DAEMON`).

Then,
there's a **third state** that's not final: `temporarily failed`.
It covers any **transient** error that prevented delivery...
but which **might** result in a delivery (or a permanent failure) if the mail was retried after **something** was fixed by the destination node.

Unlike delivered mail,
which the recipient sees,
and failed mail,
which the sender sees,
the `temporarily failed` mails are *usually* not seen by users.
They are handled between **the mail exchangers (MX) retrying behind the scene**,
and only ever seen by users when the situation is either resolved or the mail time-to-live has expired.
In the first case,
the recipient will get the mail delivered a bit later than expected,
often without even knowing that a retry took place.
In the second case,
the sender will get a notification that,
despite retries,
the mail could not be delivered after an extended period of time.

The retry strategies are handled differently by different software,
but you can expect that you'll see retries happening seconds or minutes after a temporary failure.
A popular approach,
the quadratic delay increase,
causes the delay between retries to start short,
but then grow with the number of retries so that the longer a destination host is unable to handle a mail,
the longer between the retries from a sender.

Temporary failures happen **all the time**,
it is a normal thing in the SMTP world,
looking at log you will very often see lines such as this:

    4b3a6c195c1f6010 mta delivery evpid=8f26ea98359eccea from=<misc+bounces-4541-[redacted]=tin.it@opensmtpd.org> to=<[redacted]@tin.it> rcpt=<-> source="45.76.46.201" relay="62.211.72.32 (smtp.tin.it)" delay=0s result="TempFail" stat="421 <[redacted]@tin.it> Service not available - too busy"

This is not just for small destinations,
I know of two Big Mailer Corps that have temporary issues pretty much always,
but they also have so many mail exchangers that a retry is unlikely to hit the same MX twice in a row,
so mail usually gets delivered at first or second try.


A lot more could be said but lets keep this article simple and summarize:

There exists a mechanism in SMTP which allows a destination MX to tell a sender MX that it should retry a delivery.
The retries happen behind the scene,
without any action from the users,
and can cause a delay in delivery.



Greylisting explained
--
Spam is a **volume** industry.

Spammers are not interested in targeting **specific** recipients,
they are interested in targeting a **large amount** of recipients so that it **statistically** increases the number of recipients that will fall for it.

I won't dig into this too much because it is the topic of another article I'm writing,
however you have to keep in mind that there are different kinds of spammers,
and that while some use regular MX that respect the retry requests,
a lot more rely on stateless scripts or custom MX code that **DOES NOT** honor the retries because...
**it slows them down considerably for uncertain result**.
Waiting minutes to retry a recipient just to end up resolving to an "unknown user" permanent failure is not interesting,
it clutters the queue and it prevents sending other domains at full speed.

The idea behind greylisting is very simple:

Greylisting causes a temporary failure to be triggered for MX you don't trust and keeps track of these retry requests.
If the retry happens too soon, it is ignored, so spammers can't just perform immediate retries.
If the retry happens too late, well... it is too late and the implementation may do different things ranging from blacklisting, to degrading reputation, to forgetting the initial request and causing greylisting to start all over again, and again, and again, ...
The bottom line is that spammers are **forced to keep state** and behave like an RFC-compliant implementation,
which goes against the economics of sending *en masse*.

In the other hand,
RFC-compliant MX naturally try again multiple times so they eventually fall in the correct window for a retry and are whitelisted.


How greylisting usually works
--
There are different greylisting solutions and they don't all work the same.

The general idea is that you have a database which keeps track of the window of time between which a retry is considered as valid to pass greylisting for a given MX. However, how the MX is considered for a retry varies and while some solutions will just look at the source IP address,
others will also look at the MAIL FROM,
others will also look at the RCPT TO,
and based on my own studying of some failure patterns,
the unique Message-ID is _sometimes_ also tracked.

The OpenBSD `spamd(8)` for example tracks the source IP address,
the identification hostname provided at HELO/EHLO,
the envelope-from (MAIL FROM) and the envelope-to (RCPT TO).
So, to pass greylisting, an MX must retry during the proper window of time, coming from the same IP address, identifying with the same HELO/EHLO, from the same MAIL FROM and to the same RCPT TO.

<center>
<img src="feature.jpg">
</center>

The problem with greylisting and Big Mailer Corps
--
For small destinations,
greylisting works nicely as they tend to use the same MX for incoming and outgoing trafic.
The MX will contact you,
be told to retry later,
then be accepted when it comes back after a minute or so.

Then,
you have Gmail / Yahoo / Outlook / ...
which not only have different MX for incoming and outgoing trafic,
but also have dozens and dozens of outgoing mail exchangers.
This leads to a situation where an initial MX contacts you,
gets asked to retry,
but you never see a retry because it came from another MX which was asked to retry,
and so on...

Greylisting with such hosts is unusable and results in mails being delayed for hours,
days,
or even expire and never reach you.

But at the same time,
greylisting keep a lot of nuisance outside too as it kills most scripts and compromised computers acting as spamming bots,
so a lot of operators put in place various work-arounds to keep using greylisting for small senders and not for Big Mailer Corps.

Work-around strategies
--
I used the plural but quite frankly there's only one strategy:
**whitelisting**.

The work-around is to figure out IP addresses of Big Mailer Corps,
then give them a free pass and allow them to skip greylisting.

Does this defeat greylisting ?
Not really.
Greylisting kills non-RFC compliant MX,
but they operate RFC-compliant MX and if they retried from the same outgoing MX they would pass greylisting hands up.

Fair enough,
let's whitelist them...
but how ?

For a while,
I used a naive approach which consisted in whitelisting the `/24` of any Big Mailer Corps MX that would get greylisted.
After a while,
my list was big enough that new ones would rarely get greylisted.
It was tedious,
it didn't cover the case of new ones hitting me from new ranges,
and I ended up merging lists from various people with mine to make sure I add as many as possible.
This resulted in a **constantly growing whitelist**.

This was not really smart because all of the Big Mailer Corps have SPF.
They actually advertise in a DNS record which IP they will contact you with,
so that whenever you receive a connection claiming to be from them,
you can do a lookup and check if it is legit.
So, with that in mind,
I wrote `spfwalk` (now part of `smtpctl`) in January 2018,
a tool that would walk through the SPF record of domains and extract as many IP addresses as it could.
There is a [write up about spfwalk](https://poolp.org/posts/2018-01-08/spfwalk/) on this blog if you're interested.
This tool was intended to replace the dummy approach I used until then,
but it is in no way perfect,
it is best effort.

The way SPF records work means that it's easy to check if an IP matches a policy but not to extract a list of IP addresses for later check.
For example, some policies expect runtime reverse DNS resolution of the source IP address to check if they match a domain,
so there's literally NO IP address to extract from the record.
So... this works for most cases until you hit one where it doesn't.

A lot of people are happy with SPF walking,
it does improve the situation considerably,
but it is very hackish and I personally don't use that tool anymore as I feel it's not the right way to tackle the issue.

The filter-greylist proof of concept
--
Last month,
[I wrote a proof-of-concept filter](https://poolp.org/posts/2019-11-17/november-2019-report-opensmtpd-6.6.1p1-filter-greylist-and-tons-of-portable-cleanup/)
for OpenSMTPD which uses a different approach and that is available
[on Github](https://github.com/poolpOrg/filter-greylist).

Let me clarify first that **I don't claim to have invented this**,
it just occured to me that it was a good idea so I implemented it and it may already be available elsewhere.
Since I **haven't looked very hard**,
(that's a synonym for "at all"),
not only do I not know of another implementation in another opensource software,
but if such an implementation exists,
I'm also unaware if my implementation tackles the idea the same way.

The base idea is that we don't really want Big Mailer Corps to skip the greylisting, we resort to whitelisting SPF because what we _really_ want is for all IP addresses from a Big Mailer Corp to be considered as equivalent when it comes to greylisting retries.
And by whitelisting IP addresses from the SPF record,
what we really want is to consider the IP addresses from the SPF record as equivalent...
In other words,
what we want is **SPF-aware greylisting**.

The proof of concept considers that there are two kinds of senders:
those not publishing SPF records and those publishing SPF records.

MX that do not publish SPF are greylisted by source address,
like they would with a regular greylisting.
This doesn't affect a lot of legitimate senders because Big Mailer Corps require senders to provide SPF in order not to degrade their reputation,
and because Big Mailer Corps represent such a large e-mail address space,
legitimate senders tend to comply.

Then,
you have the MX that do publish SPF records,
which includes all Big Mailer Corps for obvious reasons.
And for these,
instead of greylisting by source address,
we can do an SPF-aware greylisting for the domain.

Assuming this is the first mail we receive from an MX,
when a new connection arrives,
the filter tracks the source IP address.
The SMTP session moves forward,
initiates an SMTP transaction,
and then the client provides the envelope-from (MAIL FROM):

```
S: 220 in.mailbrix.mx ESMTP OpenSMTPD
C: EHLO localhost
S: [...]
S: 250 in.mailbrix.mx Hello localhost [127.0.0.1], pleased to meet you
C: MAIL FROM:<gilles@poolp.org>
```

At this point,
the filter can do an SPF lookup for the envelope-from domain, `poolp.org`.
If it **doesn't find an SPF record**,
it can simply **keep track of the source address** and issue a retry request.

If it **finds an SPF record**,
it **checks if the source IP address is valid**.
If it is not,
then it supposedly doesn't come from the sender domain,
so the filter keeps track of the source address and issues a retry request.
A stricter approach could be to reject the sender but I don't think it's the goal of a greylisting to check SPF validity.

If,
however,
**the source IP address is valid for the SPF record**,
then instead of keeping track of the source address,
the filter **keeps track of the domain** and issues a retry request.


Upon a second connection,
when reaching the same point in an SMTP transaction,
if the source address is valid for the SPF record and because we track the domain and not the source IP address,
any SPF valid IP address will be considered equivalent to pass greylisting.
What this means, **in simpler words**, is that if Gmail comes from one IP address,
it is allowed to retry from another,
**as long as both IP addresses are declared in SPF**.

This runtime SPF resolution also ensures that there is no need to pre-whitelist IP addresses,
instead SPF-enabled domains are whitelisted as a whole,
there's no need to run periodic cron or anything to catch up new addresses.
This way of working is fully compatible with non-SPF domains which degrade to IP-greylisted.

It can't be implemented in a daemon like `spamd(8)`,
the decision to greylist or let the session move forward is taken at `MAIL FROM`,
whereas redirection to `spamd(8)` is decided at connect time through firewall rules.


Does this get any better ?
--
Dunno but I doubt so.

There may be some adaptations to slightly improve the idea,
but I personally doubt there's much more to do in that area:
an SPF-aware greylisting solves the issue that breaks greylisting for Big Mailer Corps, no more, no less.

Keeping in mind that spamming is a volume industry,
more interesting work can be done in raising the cost of sending volumes:
just like spammers don't like retrying with uncertain result,
they **REALLY** dislike hosts that are laggy,
because it has a **HUGE** impact on throughput and queue size.
In my filter-senderscore,
I have already implemented something along these lines by correlating a delay to a sender reputation,
achieving stateless tarpitting for low reputation senders.

If I'm spending more time making OpenSMTPD a hard target to spammers it will be through time-penalty for sure.

---- 
Comments: [https://github.com/poolpOrg/poolp.org/discussions/110](https://github.com/poolpOrg/poolp.org/discussions/110)

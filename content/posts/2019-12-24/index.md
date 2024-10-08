---
title: "December 2019: OpenSMTPD and filters work, articles and goodies"
date: 2019-12-24 09:31:00 +0200
authors:
 - "gilles"
categories:
 - technology
---
{{< tldr >}}
    - wrote, reworked and translated multiple articles this month
    - got some goodies ready for my patrons
    - lots of work in OpenSMTPD's grammar, documentation and filters protocol
{{< /tldr >}}

---
<font color="red"><b>WARNING:</b></font>

Examples of **code** and **configuration** that appear in this article are
here to help illustrate and explain **development stages** of my work.

They are subject to **changes** and **must not** be considered as user documentation.
By the time you're reading this, they will likely no longer work or reflect reality.

---

Loooots of minor stuff here and there
--
These last two months,
I could carry Jules in a baby wrap and write code while he was asleep.
He then unilateraly decided that **I'm his mule now** and I'm no longer allowed to stop walking when carrying him.
Even standing still while writing code for more than five minutes straight is strictly forbidden.

As you can imagine,
this causes lots of context switching with code sessions interrupted every few minutes,
therefore he's now known as **_the interrupt storm_**.

As a result I could not finish all of what I started working on,
stuff that won't make it into this report but which I intend to finish by the end of January for the next report.


Articles rework and translations
--
This month,
I have written an article regarding my work on "[SPF-aware greylisting and filter-greylist](/posts/2019-12-01/spf-aware-greylisting-and-filter-greylist/)".

In addition,
I have reworked my "[Setting up a mail server with OpenSMTPD, Dovecot and Rspamd](/posts/2019-09-14/setting-up-a-mail-server-with-opensmtpd-dovecot-and-rspamd/)" article and removed the political aspect of it to make it a standalone article,
"[Decentralised SMTP is for the greater good](/posts/2019-12-15/decentralised-smtp-is-for-the-greater-good/)".
Both of them were translated in French,
"[Mettre en place un serveur de mail avec OpenSMTPD, Dovecot et Rspamd](/posts/2019-12-23/mettre-en-place-un-serveur-de-mail-avec-opensmtpd-dovecot-et-rspamd/)" and
"[Decentralisons SMTP pour le bien commun](/posts/2019-12-15/decentralisons-smtp-pour-le-bien-commun/)".

These were my last articles for this year.


Goodies for my sponsors
--
I have set up a reward system for my sponsors,
nothing too impressive,
but I have received the first goodies,
stickers and mugs,
a few days ago.

<center>
    <img src="2019-12-24-goodies.png">
</center>

I don't intend to start dispatching anything before mid-January,
I don't trust shipping stuff around this period of the year.
I haven't received the shirts yet anyways ;-)


OpenSMTPD work
--
I thought I'd be slacking,
but **a lot** of work has been put into OpenSMTPD since my last report,
far more than I thought I actually did. Go figure.

Reworking `match` rules a bit
---
As you probably know,
OpenSMTPD uses a ruleset to match envelopes:

```
match from local for local action "in"
match from local for any action "out"
```

These rules may have a set of criterias to refine them further:

```
match from local mail-from "gilles@poolp.org" for local action "in-alternative"
match from local for any rcpt-to "eric@faurot.net" action "out-alternative"
match from local for local action "in"
match from local for any action "out"
```

Because it has an implicit "local" behavior,
rules may skip `from local` and `for local`,
which makes ruleset more concise:
```
# match from any for local action "in"
match from any action "in"

# match from local for any action "out"
match for any action "out"
```

The problem with implicit "local"
---
When I first started working on OpenSMTPD,
mail operators kept mentionning two main problems with mail servers:

- the configuration files were crazy difficult to understand and maintain
- it was way too easy to accidentally create an open relay for spammers

I made it a project goal to have the most concise configuration file,
providing sane defaults and removing anything unnecessary,
so that the configuration would be easy to understand and so that it would take explicitely typing `from any for any` to create an open relay.
One of the "cool" features was the use of implicit local,
so that as explained above,
you could make your configuration shorter due to all rules assuming local by default.

Back then,
other developers were also trying to get the configuration shorter,
so sometimes I would get complaints that their configuration was taking four lines instead of three,
or that the lines were taking ten keywords instead of eight,
and I would try to find a way to express the grammar differently.
I will **fully take the blame for this one**,
at some point if you're competing with other MTA that are using M4 or plaintext configuration files that have hundreds of keys,
**trying to remove one line out of four is a pissing contest**.

With advances in OpenSMTPD,
and a ruleset that became more and more flexible with many more matching criterias,
trying to be as concise as possible to save two keywords became unproductive.

In some cases,
writing rules like this can be confusing and result in errors like this one:

```
match auth for any action "out"
```

where a lot of users mistakenly assume that this will match all authenticated sessions for any destination and relay...
but since `auth` is only a criteria that specifies further the rule and since rules are local by default,
this really translates to:

```
match from local auth for any action "out"
```

a rule that really only matches authenticated sessions coming from a local interface.

This is not the case that confuses users,
another error I saw happen multiple times is the following one:

```
match from any rcpt-to "gilles@poolp.org" action "out"
```

which due to implicit "local" translates to:

```
match from any for local rcpt-to "gilles@poolp.org" action "out"
```

unless `poolp.org` is the machines' hostname,
this will cause mails for `gilles@poolp.org` to be rejected,
because the `for` criteria didn't match much.

The proper way to write the rule would be:
```
match from any for domain "poolp.org" rcpt-to "gilles@poolp.org" action "out"
```

which everyone keeps simplifying as:
```
match from any for any rcpt-to "gilles@poolp.org" action "out"
```

just because rcpt-to already acts as a whitelist of recipients,
so having to maintain a list of corresponding domains is overkill.

This highlights an issue which is that the grammar should describe the user intent,
and the intent is very clear with rules like these:
```
match auth for any action "out"
match from any rcpt-to "gilles@poolp.org" action "in"
```

The fact that it doesn't work like they think is a problem.
A problem that **is caused by the implicit local behavior**,
because if `from` and `for` always had to be specified then such errors would not be possible.


Oh noes, not another grammar change !
---
Nope, don't worry, the grammar is correct as it is.

What is incorrect is the allowing of implicit behaviors,
like skipping `from` or `for`.
These should be explicit and mandatory,
the shortcomings of saving two keywords are far more annoying than the benefits.

Furthermore,
making them mandatory actually allows for shorter rules which are not doable today,
because the implicit behaviors makes them confusing.

So we have decided to go full explicit from now on,
the default configuration file will now provide both `from` and `for` even for local uses,
using implicit behaviors will result in warnings at startup,
then in a future release we will make it mandatory to declare them.

This means that **ALL** matching rules will **ALWAYS** have both a `from` and `for`,
how does that make things shorter ?


`from` used to be for source and `for` for domain
---
Initially,
the `from` keyword was used to declare the source of a connection,
or `local` if it could originate from any locally bound address or the Unix socket.
The `for` keyword was used to declare a destination domain,
or `local` if it could be destined for any domain known locally,
usually **localhost** and domains obtained through `gethostname()` or the `/etc/mail/mailname` file.
The special keyword `all` could be used to encompass any address when used with `from` and any domain when used with `for`.

As time passed by,
we started adding support for other criterias,
and ruleset could be expressed with intents that were no longer considering a source address or a destination domain.
The `mail-from` and `rcpt-to` criterias are perfect example of this,
they are **often** used with the intent of providing a whitelist of e-mail addresses.
A lot of people use constructs such as:

```
match from any mail-from <senders> [...]
match from any for any rcpt-to <rcpts> [...]
```

What they really want is to match a sender or recipient e-mail address regardless of the source.
If we take a step back to get a larger picture,
they want to use `mail-from` as an origin and `rcpt-to` as a destination,
the source address and destination domain are set to `any` because they are...
just a criteria that makes no sense in these rules and that they want to discard.

When you see things this way,
it makes you reevaluate how `from` and `for` should be used,
they are not source address and destination domain related but origin and destination related in a wider sense.
Luckily for us the shift in paradigm is **retro-compatible with previous grammar**.
Rulesets I write today will still be valid and work the same way tomorrow,
however new constructs become available that better depicts the user intent,
with no more ambiguous cases and shorter syntax for most cases.

How so ?

Let's take the examples above,
what the user really expresses when writing:
```
match from any mail-from <senders> [...]
```

is the following:
```
match from mail-from <senders> [...]
```

And what the user really wants to express when writing:
```
match from any for any rcpt-to <rcpts> [...]
```

is the following:
```
match from any for rcpt-to <rcpts> [...]
```

Specifying `any` is not invalid,
it is just redundant in both of these cases,
but is still valid because using a source address criteria in conjunction to a sender or recipient e-mail address is still a valid use-case:
```
match from src 192.168.1.0/24 mail-from <senders> [...]
match from local mail-from <senders> [...]
```

With that in mind,
the following rules can now be expressed to match authenticated users regardless of their source address:
```
match from auth for any [...]
match from auth gilles@poolp.org for any [...]
match from auth <users> for any [...]
```

And just like in the previous example,
it is still possible to filter on specific addresses:
```
match from src 192.168.1.0/24 auth <users> for any [...]
match from src 192.168.2.0/24 auth <users> for any [...]
```

This change took more time convincing others than it took writing,
but it is really the right direction.
Because it makes the syntax simpler but also because ambiguous cases translate into people mailing me for help,
and all of the common mistakes people do just vanish when `from` and `for` become explicit and the new constructs are available.

What is fun is that once I got enough okays,
I committed the change and I'm quite sure that other OpenBSD hackers didn't even realize it as nothing changed for existing setups.

<center>
    <img src="2019-12-24-plan.gif">
</center>

This will be part of the **OpenSMTPD 6.7.0 release** happening sometime around April/May,
but it is already available in OpenBSD -current of in the master and portable branches on Github.

I have converted multiple setups to the next syntax and they are much nicer this way.


Improve documentation
--
Various improvements were done to the manual pages of OpenSMTPD.

I have committed bits of missing information which `jmc@` reworded and fixed to get nicer.
There's still work to do,
most notably documenting how to plug filters and providing examples for DKIM and spam filtering,
but the documentation had a lot of redundant bits reworked to make it easier to grasp.

I spent **hours** writing an initial version of [`smtpd-filters(7)`](https://github.com/OpenSMTPD/OpenSMTPD/blob/master/usr.sbin/smtpd/smtpd-filters.7),
a man page describing the filter API for people willing to write their own filters.
This is a work in progress and I have not linked it to the build yet,
but it explains how things work at the protocol level,
the various events that can happen,
etc...

From now on,
I'll point people to this page when asked how to get started with filters,
it will help me find the parts that need improvement.


Fix a couple protocol issue in filters
--
There were two issues in the filter API,
issues which people would not notice because of their nature.

The first one is related to the request/response aspect of the filtering protocol.
I had a working proof-of-concept fairly fast last year but it took months to refine the order of fields.
I didn't realize that with the reordering of fields,
we ended up have two fields that are common to requests and responses swapped.

What this means is that the request had the fields `SESSION|TOKEN` while responses had the fields `TOKEN|SESSION`,
and since the parsing code was correct the code worked correctly,
but from a developer perspective reading through the code or looking at raw protocol lines,
this was confusing.
The order of fields in responses was swapped to match the order of fields in requests,
leading to a protocol version bump.

The second issue was related to the `smtp-out reporting` feature which I was working on.
Until now a filter could only be attached to a `listen` line so it knew for sure it could register `smtp-in` events.
With `smtp-out` a filter can be attached to a `relay` action allowing it to receive reporting events for outgoing trafic,
but nothing in the protocol would let the filter know where it was attached,
so a filter could not know if it had to register for `smtp-in` or `smtp-out` events.

The protocol has a handshake which allows the server to provide a set of `key|value` to filters at startup before they register events,
so I have made sure the server would let the filters know to which subsystems they were attached so they can register the proper events:

```
config|subsystem|smtp-in
config|subsystem|smtp-out
```


The smtp-out reporting
--
With the subsystem attachement issue fixed I could complete my work on `smtp-out reporting`.

This resulted in reworking how the `smtp` and `mta` layer registered sessions in filters,
how reporting events were sent to filters,
and which event made some informations available to filters.

The `smtp-out` and `smtp-in` reporting events are the same,
only the direction changes,
so there's no protocol change but internally the concept of a session differs between the `smtp` and `mta` layer.
For instance,
a session begins when a client connects for `smtp`,
but a session begins before we connect to a remote server in `mta`,
so the availability of `rdns`, `fcrdns`, `src` and `dest` addresses doesn't take place at the same timing for both.

I won't expand much because it's not that interesting,
but it required quite a bit of rework and it took me months of work on and off to get to a state where the code was clean and stable.
Not all events are generated in `smtp-out` yet but this will be worked on and we'll be fine before the next release.

All of my mail exchangers are now producing an event log for both incoming and outgoing trafic,
I'm looking forward to exploit these into nice graphs.


Introduce a `bypass` action for builtin filters
--
The builtin filters are filters that operates within OpenSMTPD and provide a set of simple filtering capabilities.
They allow attaching to different phases, matching different criterias and taking various decisions.
I won't dive into this because it was already discussed in this blog and is documented in the [`smtpd.conf(5)`](https://man.openbsd.org/smtpd.conf) man page.
There was just one thing missing,
the **ability to bypass filters** in a specific phase.

For instance,
you could define multiple filters:
```
filter no_rdns phase mail-from match !rdns reject "550 go away"
filter no_fcrdns phase mail-from match !fcrdns  "550 go away"

listen on all filter { no_rdns, no_fcrdns }
```

but you could not have a way to bypass these filters for a set of trusted hosts for example.

The `bypass` action allows a builtin filter to take the decision to not go through the chain but accept the phase,
it would allow doing the following:
```
filter trusted phase mail-from match src <trusted_sources> bypass
filter no_rdns phase mail-from match !rdns reject "550 go away"
filter no_fcrdns phase mail-from match !fcrdns  "550 go away"

listen on all filter { trusted, no_rdns, no_fcrdns }
```

The lack of a bypass mechanism caused multiple people to ask me for work-arounds,
as many use-cases require being able to skip filters for a set of hosts,
and OpenSMTPD didn't provide any way to do that.

The bypass mechanism was not pushed to proc filters,
I'm not convinced at this point that it is necessary.
Builtin filters reside in the configuration file and are crafted with knowledge of how the filter chain looks like,
this is not the case for proc filters so I'm unconvinced if this is a useful feature there.
I may change my mind but no rush on this anyways.


Work on multiple filters
--
All of my filters were adapted to cope with the protocol change described above with the swapped fields.
I had to release new versions of all of them,
a version that would keep working with the previous fields order and that would work with the new fields order.

I didn't work on new features but `filter-rspamd` was contributed to by [@freswa](https://github.com/freswa),
[@whataboutpereira](https://github.com/whataboutpereira) and [@lfos](https://github.com/lfos) who also made **several** contributions to `filter-senderscore`.

These filters live their lives now which is very cool.


Improvement to filter-greylist
--
Following a discussion on our IRC channel (#OpenSMTPD @ irc.freenode.net),
I made an improvement to `filter-greylist` which consists in detecting that a session was initiated by a local or authenticated user,
and whitelisting the destination for messages.

Not whitelisting destination doesn't cause huge penalty with SPF-aware greylisting,
but having destinations of local users whitelisted means that there's **no penalty at all** for hosts from which we expect e-mails.



What next ?
--
No more writing until 2020.

I will take a few days off from computers then resume writing code maybe next week,
hopefully finishing some of my pending works in progress so I can disclose them :-)


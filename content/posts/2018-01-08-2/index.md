---
title: spfwalk 
date: 2018-01-08 17:19:00
category: OpenSMTPD
author: Gilles Chehade
categories:
 - technology
---

{{< tldr >}}
	deraadt@ thought it would be nice to have a spf fetch utility in base.
	Aaron Poffenberger wrote a shell-based `spf_fetch` utility.
	I wrote a C-based `spfwalk` utility that's `pledge()`-ed.
	The `spfwalk` utility got merged to `smtpctl`.
{{< /tldr >}}

What's SPF in a few words
-------------------------
SPF is the Sender Policy Framework, a standard to verify the domain name of an e-mail sender.

Long story short, the SMTP protocol does not come with a way to authenticate a domain and,
during an SMTP session,
nothing really prevents a sender from pretending to come from any domain:

```
$ nc localhost 25 
220 poolp.org ESMTP OpenSMTPD
HELO pussycat
250 poolp.org Hello pussycat [127.0.0.1], pleased to meet you
MAIL FROM:<gilles@systemd.lol>
250 2.0.0: Ok
^C
$ 
```

Note that this is a feature of the SMTP protocol, not a bug.


It turns out that the internet is a hostile place and people started abusing this so...
the solution,
as usual when it comes to SMTP,
was to shove data inside DNS records \o/


SPF allows the owner of a DNS zone to declare which machines are allowed to send mail on behalf of the domain in a TXT record.
With this feature,
whenever a spammer attempts to send mail on behalf of @google.com,
the receiving MX can simply check that the client is allowed to send mail by the google.com zone.


This seems lovely,
however it's not a spam killer because it actually requires senders to create the record,
destination nodes to actually check the TXT record and match the client address against it,
and it doesn't prevent spammers from adding SPF records to their own domains.


What it does,
when everything is setup correctly on both ends,
is protect the destination node from spammers trying to impersonnate a sender domain...
as long as the spammer doesn't control a machine declared in the TXT record [ it's easy to whitelist a /16 ;) ].



Greylisting
-----------
A technique that's been used widely to reduce spam is to rely on greylisting.


Basically,
when a MX you don't know connects to your node,
it gets bounced with a temporary failure requesting a retry.
All SMTP server know how to handle these retries,
however in a spamming economy based on sending volumes of messages it's often not worth it to retry on temporary failures.


Spammers will come from many source addresses,
keep being seen as new MX,
keep being bounced,
life is good.


Why am I talking about greylisting you ask ?



When BIG MAILERS made it easier for spammers to annoy us
--------------------------------------------------------
It's impolite to point fingers so I won't name them explicitely,
you know who they are.


Greylisting works lovely to waste spammers' time...
as long as you can assume legitimate hosts come from the same IP addresses.


At some point,
BIG MAILERS decided that:
nope, we'll send from this host,
then we'll retry from this one,
then another one,
and then since we have hundreds of IP addresses available,
well just fuck you, we'll make sure we never hit you twice with the same.


I don't know if these were the exact words,
but it essentially leads to that "fuck you" result so...


Since BIG MAILERS could not send from a small set of addresses and reuse the same ones upon retries,
they started advertising their MX in SPF records.
And by that I mean,
they started whitelisting full ranges in records requiring recursive and cross-domains lookups because WHY NOT ?


This way, greylisting became essentially unusable to many who turned it off because they can't easily whitelist BIG MAILERS.


This is how we get to `spfwalk`
-------------------------------
We don't want to NOT have greylisting,
so we want a tool to actually harvest records enabled in BIG MAILERS' SPF record.


deraadt@ ran into an issue with one of his MX which had BIG MAILERS unable to pass greylisting.
He asked me if I could write a utility to fetch SPF records and insert them into a PF table to whitelist MX for particular domains.


Coincidentally,
Aaron Poffenberger announced the next day that he had been working on a tool called `spf_fetch` [https://github.com/akpoff/spf_fetch](https://github.com/akpoff/spf_fetch) to do just that. The idea was nice however it couldn't be committed to OpenBSD as is because it is written as a set of shell scripts and we wanted a piece of code that could be `pledge()`-ed.


I contacted Aaron and told him I was going to be working on a C version based on the asr asynchronous resolver,
and told him I would appreciate if he contributed since he had already started a similar project.
A few weeks later,
we had a work in progress `spfwalk` utility commited to my repository [https://github.com/poolpOrg/spfwalk](https://github.com/poolpOrg/spfwalk).



smtpctl spf walk
----------------
Instead of having a new utility committed to base,
we decided to make it a subcommand of the `smtpctl` utility.


The main idea is that `smtpctl spf walk` will read a list of domains from stdin and output a list of IP addresses and ranges to stdout,
allowing it to be used in scripts,
to generate file lists,
or to be piped directly into `pfctl` to feed a table.


This is still a work in progress,
but it allows you to `cat /etc/mail/BIGMAILERS.txt | smtpctl spf walk | pfctl -t spf-white -T add -f -`,
which is quite an improvement over having nothing to do it to start with :-)


Final words
-----------
sunil@ merged my utility to `smtpctl`,
I made sure it ran unprivileged,
so now we have an spf fetching utility that runs unprivileged and `pledge()`-ed in the base system.


This was committed a few days ago to OpenBSD -current,
it requires testing because I'm pretty sure it has bugs still.

---
Want to comment ? [Open an issue on Github](https://github.com/poolpOrg/poolpOrg.github.io/issues/)

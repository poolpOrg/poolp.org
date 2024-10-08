---
title: "OpenSMTPD: src-address maps and spamhaus"
date: 2012-05-17 17:23:09
category: OpenSMTPD
authors:
 - "gilles"
categories:
 - technology
---

First of all, I'd like to "thank" The Spamhaus project which forced me into hacking this feature in a hurry.

See, I rent a server at online.net which I use for OpenSMTPD live testing and to run some experimental code before it hits the OpenBSD tree.

Yesterday I received a mail from a user complaining that he couldn't send mail to a friend because we were listed on spamhaus. That was very strange because ALL users at poolp.org are trusted and I have SMTP logs displayed at all times precisely because we run experimental code. There is no way spam would be sent from that box without us noticing.

After investigating, it turns out that this very smart project decided that blacklisting a full /23 range was acceptable to block 2 spammers in a netblock. I thought it was a mistake but it turns out that it's part of their "escalation" process to break mail systems and have users suffer mail loss when they are in a conflict with a provider.

After a quick exchange with a couple people defending their position, I got bored of dealing with idiots and decided to start working on the src-address map right away.

So here it is source address maps:

# use 192.168.1.1 as source address accept for relay src-address "192.168.1.1"

and

map "srcmap" source plain "/etc/mail/srcmap.txt"

# use an address from the "srcmap" mapping accept for relay src-address map "srcmap"

Of course, the srcmap mapping allows dynamic changes so it can be updated at runtime without a daemon restart.

This has not been committed to the tree yet, I'd like to make some changes to the map API first to support selection policies. Hopefully it can be done by the end of this week-end.

Meanwhile, I'd like to take a few seconds to discourage STRONGLY the use of blacklists managed by irresponsable people that do not provide a FREE method for unlisting (a paying method is called racket in my book). Also, I'd like to stress out that using Spamhaus will cause you to lose legitimate mails whenever they feel it is in their interest to pressure a provider.

If you are a Spamhaus user and you don't agree with me, I'd like to send you a mail to prove my point, but I can't, which proves my point.


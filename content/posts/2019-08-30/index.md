---
title: "You should not run your mail server because mail is hard"
date: 2019-08-30 12:00:00
category: opensource
authors:
 - Gilles Chehade
---

    TL;DR:
    - Mail is not hard: people keep repeating that because they read it, not because they tried it
    - Big Mailer Corps are quite happy with that myth, it keeps their userbase growing
    - Big Mailer Corps control a large percentage of the e-mail address space which is good for none of us
    - It's ok that people have their e-mails hosted at Big Mailer Corps as long as there's enough people outside too


EDIT (2019-12-15)
--
A practical guide to [set up a mail exchanger](/posts/2019-09-14/setting-up-a-mail-server-with-opensmtpd-dovecot-and-rspamd/) was published on this blog.


Disclaimer
--
**<font color="red">THIS IS FOR SYSADMINS WITH TECH KNOWLEDGE, WHO KNOW HOW TO HOST SERVICES</font>.**
**Self-hosting mail is not HARD but requires WORK, which are two different things.**
**Setting up a mail infrastructures requires a lot of initial work, then basic long term maintenance.**

I work on an opensource SMTP server. I build both opensource and proprietary solutions related to mail. I will likely open a commercial mail service next year.

In this article,
I will voluntarily use the term `mail` because it is vague enough to encompass protocols and software.
This is not a very technical article and I don't want to dive into protocols,
I want people who have never worked with mail to understand all of it.

I will also not explain how I achieve the tasks I describe as easy.
I want this article to be about the "mail is hard" myth,
disregarding what technical solution you use to implement it.
I want people who read this to go read about [Postfix](http://www.postfix.org),
[Notqmail](https://github.com/notqmail/notqmail),
[Exim](https://www.exim.org) and [OpenSMTPD](https://www.OpenSMTPD.org),
and not go directly to OpenSMTPD because I provided examples.

I will write a follow-up article,
this time focusing on how I do things with OpenSMTPD.
If people write similar articles for other solutions,
please forward them to me and I'll link some of them.
it will be updated as time passes by to reflect changes in the ecosystem,
come back and check again over time.

Finally,
the name Big Mailer Corps represents the major e-mail providers.
I'm not targeting a specific one,
you can basically replace Big Mailer Corps anywhere in this text with the name of any provider that holds several hundred of millions of recipient addresses.
Keep in mind that some Big Mailer Corps allow hosting under your own domain name,
so when I mention the e-mail address space,
if you own a domain but it is hosted by a Big Mailer Corp,
your domain and all e-mail addresses below your domain are part of their address space.


Once upon a time, the "mail is hard" myth
--
When you first look into becoming independant with your e-mails,
as soon as you ask in a public tech medium for help setting up a "mail" server,
people will invariably jump in the discussion to discourage you from attempting because "mail is hard".

Not only "mail is hard" but it also seems that "Big Mailer Corps have already won",
that "all mail you send will end up in your recipients' spam box",
and that "you will be flooded by spammers" who will "abuse your server to relay spam to the world".

Wow, that's overwhelming :-|

You just wanted to send and receive mail because it seemed like a good idea,
and now this turned into the worst life decision. But is it ?


Software is hard
---
Mail used to be hard.
A long long time ago.

It was hard because software were hard to setup correctly.
A mail server like Sendmail required a Ph.D in compilers theory to operate,
and the elitist culture of postmasters who could read Sendmail's so-called "configuration files" didn't help create a user-friendly environment.
Postmasters bragging about how hard Sendmail was,
while disclosing their unconditional love for m4 was a pissing contest.
I can easily imagine these people whipping themselves with fresh nettles as a hobby.

Postfix is an alternative to Sendmail that has been around since 1998.
While I wouldn't consider it as "simple" by any stretch of the word,
it is **orders of magnitude** simpler than Sendmail.
With the help of a search engine,
new comers will easily find tutorials to spot the **three or four** configuration options they need to tweak.

OpenSMTPD is another alternative which was first released in 2013.
It is also **orders of magnitude** simpler than Sendmail.
The configuration reads almost as plain english and a usable configuration file can actually fit ... **in a tweet**.

<img alt="OpenSMTPD config fitting in a tweet" src="/images/2019-08-30-tweet.png" width=100%>
[full screen](/images/2019-08-30-tweet.png)

I don't have experience in other contenders,
but most operating systems and distributions provide multiple alternatives,
pre-packaged so you can install and run with a simple command.

Mail software is NOT hard.
It was if you stopped looking in the 90's.


OK, software isn't hard but dealing with SPAM is
--
The myth goes on by saying that mail is hard because as soon as you run your mail server,
spammers are going to come in hordes and dealing with spam will become a daily nightmare.
This could not be further from the truth.

Now that we've established trust,
I'm not going to lie to you: there are hordes of spammers.

When you plug your server in to the internet,
you'll start seeing connections from random sources trying to get mail accross.
You'll see them coming from home connections,
from remote countries,
from IP addresses sharing the same range,
there will be no end to how much amazement this will procure.

Basically, they all fit in two categories:

- clients trying to abuse your server to use it as a relay to send spam to the world
- clients trying to send spam to you after having obtained your e-mail address somehow

The ones from the first category are easy to deal with:
**IGNORE THEM**.
They search for misconfigured servers and try doing things that will be rejected on a properly configured server,
or even try to authenticate with a dictionnary attack which is not going to succeed if you have good passwords.
They are like mosquitoes on a summer evening,
annoying but... meh.
```
c1a89cb774083905 smtp connected address=185.234.219.64 host=<unknown>
c1a89cb774083905 smtp failed-command command="AUTH LOGIN" result="503 5.5.1 Invalid command: Command not supported"
c1a89cb774083905 smtp disconnected reason=disconnect
c1a89cb8c5b84cbf smtp connected address=193.32.160.143 host=<unknown>
c1a89cb8c5b84cbf smtp bad-input result="500 5.5.1 Invalid command: Pipelining not supported"
c1a89cb8c5b84cbf smtp disconnected reason=quit
c1a89cb9441966e7 smtp connected address=185.234.219.193 host=<unknown>
c1a89cb9441966e7 smtp failed-command command="AUTH LOGIN" result="503 5.5.1 Invalid command: Command not supported"
c1a89cb9441966e7 smtp disconnected reason=disconnect
```
If they really, really bother you or you dislike logs full of such attempts,
write a script that detects such patterns in logs and add them to your firewall.
Out of pure laziness,
I have never ever used scripts to deal with these in the twenty years I operated mail servers and I'm still here to talk about it.
As long as you don't see them succeeding anything,
you can just disregard these.

The ones from the second category are slighly more annoying because if you ignore them,
your mailbox becomes full of spam.
Luckily,
they are not so hard to filter through several **simple** means and it is very easy to reduce spam to a few,
every now and then,
properly classified in a Spam folder (also known as "junked" mails).
I take absolutely no precaution hiding my e-mail address,
[gilles@poolp.org](mailto:gilles@poolp.org),
and I sometimes get one or two spam e-mails per day in the junk folder.
Not only is that not a daily nightmare,
but it's less than what I actually receive on my own Big Mailer Corps account,
which I do not share as easily and which has an average of three to five daily junked mails.

Note that some very simple filters you'd apply for the second category of spammers,
are HIGHLY effective to also kill the spammers from the first category:
```
a56dece24dcac3d2 smtp failed-command command="DATA" result="550 message rejected"
a56ded3dd6cda8c2 smtp failed-command command="" result="550 your IP reputation is too low for this MX"
a56ded3f7db5b96c smtp failed-command command="DATA" result="550 message rejected"
a56dec6ffdb2caef smtp failed-command command="" result="421 you must have rDNS to contact this MX"
a56dec895475b9bf smtp failed-command command="" result="421 you must have FCrDNS to contact this MX"
```

I'll just state it as it is:
**You will never reach "absolute 0 spam"**,
it was proven mathematically in the 2000s,
but the amount you'll receive while self-hosted can be as low or lower as what you receive at Big Mailer Corps.
Spam is not more of an issue self-hosted,
no matter how much the marketing tries to tell you otherwise.

To illustrate this I emptied the Spam folder on my poolp.org account and my Big Mailer Corps account this morning,
here's the screenshot I took of both accounts tonight,
poolp.org on the left and Big Mailer Corps on the right.
Neither of them have spam in Inbox.

<img alt="poolp.org vs Big Mailer Corps spambox" src="/images/2019-08-30-spambox.png" width=100%>
[full screen](/images/2019-08-30-spambox.png)


OK SPAM is not the issue but my mails will not reach my users at Big Mailer Corps
--
Another myth,
with slightly more substance this time,
is that sending e-mail to an address at a Big Mailer Corp will result in the e-mail being rejected or junked.

Let me tell you a secret:
Big Mailer Corps are not worried about you but are worried about big senders harassing their users.
They do not care about your personal server sending a few mails,
even if its in the thousands per months.
What they care about is the infected computers or compromised servers flooding their users.
What they care about are the marketing companies that are literally shitting over them,
sending individually millions of commercial mails per day,
trying to work-around spam filters,
and that sometimes manage to go for a while without being rejected.
Unless you are sending hundreds of thousands of mails to them on a daily basis,
quite frankly and without trying to hurt your feelings,
you fall waaaaaaaaaaaaaaay below the radars.

<img alt="inbox at various Big Mailer Corps" src="/images/2019-08-30-bigmailercorps.png" width=100%>
[full screen](/images/2019-08-30-bigmailercorps.png)

So why did I say this claim has more substance than the others ?

Big Mailer Corps have introduced proof-of-work into mail exchanges,
voluntarily or not.

Unlike legitimate senders who want to reach a specific user,
spammers want to reach a ton of users whoever they are,
because statistically some of them are going to fall for whatever it is that they're trying to sell.
Among these spammers,
we'll also include marketing companies that buy lists of users from partners,
sending indiscriminately to "activate" them in hope of reaching some percentage of openings.
They don't care who receives an e-mail as long as it opens,
because you know, statistics.
So the harder it is for them,
the higher the chance they'll switch to another target because it's the only way to not fall behind statistically.
Don't think I'm making this up,
you'd be amazed at what tricks and hacks are used to mechanically increase openings by a few percents,
including lowering the number of recipients at a particular destination and increasing at another.

In opposition to this are legitimate users who can't just switch to another target,
they want to reach a specific recipient and,
if work is needed to achieve that,
well... you got to do what you got to do.
They'll pour in the work needed to make it work.

Knowing this,
Big Mailer Corps came with sets of rules about what a Good Sender should do to be able to communicate with them.
These lists are basically a proof of work:
they do not guarantee that you'll be able to send to them,
they do not guarantee that you'll hit the inbox and will not be junked,
but they are the minimal set of things you should do to prove you actually care.
And considering some spammers do their best to look good,
if you don't do it yourself it basically means you're not even willing to do better than spammers.

These rules are not only here to annoy you,
they are also very effective:
some of the rules cannot be achieved for spammers who use compromised hosts, for example.
This proof of work paradigm is annoying because it raises (in time) the cost of entry,
but it can also be leveraged by others to kill spam.
Since Good Senders most definitely want to be able to send to Big Mailer Corps,
if you receive connections from clients that didn't do the minimum to deliver there,
you can shut them down yourself because they're already cut from a big portion of the e-mail address space.

This is why I wrote this has more substance than the other claims.
It's true that **IF** you don't even try to do the minimum work,
**THEN** you'll start with a penalty.

In practice, a notion of reputation is also into play.
Some people don't even try but they fall but fall sooooooo far below the radars...
that even without trying they'll manage to send e-mails without issues.
I often send mail to my Big Mailer Corp account from my development laptop,
far from being properly configured,
and they almost always reach inbox.
A good reputation allows you to make some mistakes and go through without respecting all rules,
while bad reputation increases the amount of work you need to do.
Considering that a good reputation is earned from doing things right,
you get the general idea: do things right.

The minimum set rules is **VERY FAR** from being hard.
To quote someone on twitter:

<img alt="rDNS + SPF/DKIM and you're golden" src="/images/2019-08-30-tweet_2.png" width=100%>
[full screen](/images/2019-08-30-tweet_2.png)

I would add a few things to that list but this highlights that some people are already "golden",
just by setting up proper rDNS, SPF and DKIM.
That last bit about handling incoming spam,
we've already discussed it above ;-)


OK, then why is everyone saying it's hard ?
--
The first reason,
is because no one claims otherwise and,
since Big Mailer Corps benefits from this situation,
they're not going to contradict it either.
Big Mailer Corps **BENEFIT** from the myth that mail is hard as this means more people rely on them,
they control more of the e-mail address space,
and this translates to more e-mails being analyzed for targeted advertisement.
The more people are discouraged,
the more people will eventually subscribe to their services,
and since they already control a large share,
they can make mail slighly more difficult by making their requirements higher (harder, not hard).
This is **not** something they do in some kind of conspiracy,
this is just the result of them obtaining more power because people stay away from self hosting.

Another reason is because it used to be hard a long time ago.
People got traumatized by how hard it was to not screw up and never reevaluated the situation.
Some people today genuinely discourage other people from running their mail server,
citing the very real difficulties they faced over a decade ago,
far before some of today's tools even existed.

And finally,
another reason is that people keep repeating it without actually trying themselves.
I know this for a fact because people have been telling me that mail is hard for the last ten years,
and for the last ten years I asked what they found hard in order to try improving the situation.
A **VAST** majority of these people confessed that they never actually tried:
they read or heard that mail was hard,
often from a source they trusted,
then accepted that claim and started telling others that mail was hard.
We can't really blame them when that myth has been around for so long,
if I had no previous knowledge of mail and did a quick search today to find out how to setup my server,
I might just decide not to do it given how difficult it seems from reading others.


What do we do from now ?
--
We need to reclaim mail.
I'm not saying people shouldn't be hosted at Big Mailer Corps,
but these should not become the Pavlovian reaction to "where do I get an e-mail address ?".

As long as there are enough mail hosts out there,
Big Mailer Corps **HAVE** to remain friendly because their users can complain that their legitimate mail is junked,
which results in a risk of them leaving for elsewhere,
less e-mails, less targeted advertisement, less $$$.
If the number of mail hosts shrinks to the point that trafic not coming from Big Mailer Corps is irrelevant,
then it becomes their protocols and they can start making up rules that are not sustainable by anyone but them,
because no one will notice when the insignificant number of e-mails not coming from there is junked.
Again, **not** a conspiracy but a side effect of being the few relevant actors of a system:
why bother abiding to standards that work for everyone and ensuring any sender can reach them...
if most of the trafic comes from a handful of actors.

This is already happening with one specifically requiring mails to be sent from another Big Mailer Corp to hit the inbox,
or requiring that senders be added to the contacts for others.
Any other sender will hit spambox unconditionnally for a while before being eventually upgraded to inbox.

We can't let that happen:
allowing e-mail to be fully controlled by a small set of cooperating multi-million users hosts is just accepting to be screwed.

It is therefore very important that we don't let the myth propagate further.
Our best interest is to have a WIDE variety of mail hosts and providers,
small and big,
commercial and not.
We must not allow the number of mail hosts to shrink,
they must increase so the e-mail address space out of the control of Big Mailer Corps remains significant.

And by all means,
**we must not push everyone to use Big Mailer Corps**,
particularly because a lot of people simply read their e-mail from a smartphone and don't see a difference in interface,
so they could be using pretty much any provider and be just as happy.

I hope my point gets accross,
feel free to share this wherever you want and point people to this article.

In a few days I'll publish a _practical_ description of how to setup a host similar to mine,
providing spam protection for incoming mail,
and basic proof of work to make most Big Mailer Corps happy.

--- 
If you like my work, [support me on patreon](https://patreon.com/gilles) !

Comments: [https://github.com/poolpOrg/poolp.org/discussions/105](https://github.com/poolpOrg/poolp.org/discussions/105)

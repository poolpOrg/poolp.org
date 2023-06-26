---
title: "News-from-the-front"
date: 2013-12-14 21:36:40
category: OpenSMTPD
authors:
 - Gilles Chehade
categories:
 - technology
---

EHLO readers,

This blog post is the first since a few months, I've been busy and struggling with some personal health and familial issues. I won't share them here as its not really something anyone can help with, so... let's focus on OpenSMTPD !

What happened since last post

When I wrote the last blog post, we had just released 5.3.2 which was a minor release that fixed a few non-critical bugs that were reported to us since the first major release a few months earlier.

A while later, we released another minor release, 5.3.3, that also fixed minor bugs and brought some new non-invasive features to deal with common use-cases reported by our increasing user base.

OpenSMTPD 5.3.3 was very stable, it's been running on busy servers at work and we did not experience any bug with it while accepting and routing millions of daily messages with remote hosts on several machines.

It was a nice release for what it's worth :-)

What now ?

Well, we didn't stop hacking on OpenSMTPD and since 5.3.3 we have gone through lots of simplifications and adding new features. There are actually so many changes that a blog post can't possibly go through all of it but I'll discuss some of the most important and visible ones.

We have released new major version 5.4.1 a few days ago, and the features that are described below are all part of it. It is a very good release IMO and you should definitely take time to switch your 5.3.x setups to this new one.

smtpd.conf improvements

First of all, despite being already very simple our configuration file has been simplified further and new features were introduced.

For example, certificate configuration had always been a bit tricky because we inherited an old behaviour were we would infer file names from a certificate name. This imposed certificates to be in a specific location /etc/mail/certs and to follow a naming conventions so that a certificate foobar would expect /etc/mail/certs/foobar.key and /etc/mail/certs/foobar.crt to be found.

I was heavily bothered with this because I store my certificates and keys in /etc/ssl and /etc/ssl/private respectively and didn't like having the /etc/mail/certs path imposed because that's the way we had it with Sendmail. For years I used symbolic links but that was a hack around fixing this broken behavior.

Also, some people were confused because not only they could provide the path to their certificates but the listen directive expected parameters to be provided in a specific order which meant that:

listen on all tls hostname example.org certificate example would fail while:

listen on all tls certificate example hostname example.org would work the way they expected. We therefore went into a grammar cleanup to ensure that parameters were swappable where it made sense and to come up with a much simpler way to deal with certificates. The pki directive was introduced which was no longer tied to a listen statement but declared a database of certificates and keys associated to a hostname:

pki example.org certificate "/etc/ssl/example.org.crt" pki example.org key "/etc/ssl/private/example.org.key" With this in place, it became possible to simplify the configuration by referencing the hostname in places where it made sense:

listen on all tls pki example.org accept for any relay pki example.org and with parameters swappable, it was now also possible to use either one of these:

listen on all tls pki example.org listen on all pki example.org tls with the same expected result.

While there, we decided to introduce a set of new features to our grammar so that use-cases that previously required several rules could be simplified. For example, we introduced the possibility to use negations:

accept from ! local [...] reject for ! domain We also added the possibily to filter senders and recipients at the ruleset level:

accept from any sender "@poolp.org" [...] accept from any sender "gilles@poolp.org" [...] accept for domain poolp.org recipient "@poolp.org" [...] accept for domain poolp.org recipient "gilles@poolp.org" [...] accept from any sender [...] And pushed it further by accepting wildcards in domain parts:

accept for domain poolp.org recipient "gilles@*.poolp.org" [...] Eric thought it would be a good idea to be able to accept mail for domains but require that the mail be forwarded elsewhere. We therefore introduced the forward-only keyword which allows you to:

accept for domain poolp.org foward-only Accepting mails for local recipients ONLY if they are redirected to an external address either through the use of a ~/.forward file or by an alias.

With these the configuration file became quite more powerful while keeping the intended simplicity. But ... we did more ;-)

Until now, a listener had to declare it's hostname:

listen on all hostname myhostname But this meant that if you had a single interface accepting mails for various domains, you had to list them all by their IP addresses:

listen on 192.168.1.1 hostname mx1.poolp.org listen on 192.168.1.2 hostname mx1.pool.ps Eric introduced the hostnames table to perform address to hostname lookups, allowing for a single listen line to use a hostnames table:

listen on all hostnames and reduce considerably the size of a multi-domain setup.

More features were added later but they are not part of the latest major release so I will discuss them in a post next week ;-)

TLS improvements

The new pki stuff in smtpd.conf allowed us to simplify a bit of code internally and it allowed us to implement new features. Some where not mature enough to make it to the major release (I'll talk about them next week), however there's been a couple improvements that did make it to the release.

We have introduced support for TLS Perfect Forward Secrecy. We already had support for ephemeral key exchange for years, but after a suggestion in the OpenBSD hackers list to add EC support to another project, we came up with a diff to bring support for ECDHE in OpenSMTPD. It was such a simple change that it was committed the same day and we saw the result almost immediately when reading headers from mails we received.

We then brough back a feature I had implemented a long time ago but disabled for some reason I can't recall. It required a bit of work but when declaring a PKI, a custom CA certificate may be provided allowing a listener to use a private CA certificate to authenticate users from an organization... as simple as:

pki mx1.poolp.org ca "/etc/ssl/myca.pem" For a long time, certificates were verified but would simply alter the headers, but with this feature came the need to be able to prevent sessions from establishing if verification failed. We introduced the "verify" keyword:

a user MUST provide a certificate that can be verified
with our system bundle if no "ca" is part of our pki
entry, or with our "ca" if we declared one.
listen on all tls-require verify pki mx1.poolp.org We pushed the feature a bit further, by allowing a relay rule to only work if it can establish a TLS session ... and eventually verify the remote certificate:

ONLY relay through TLS
accept [...] relay tls

ONLY relay through TLS ... if remote certificate is valid !
accept [...] relay tls verify

ONLY relay through our route if remote certificate is valid !
accept [...] relay via tls://mysmarthost verify To increase the security of the entire SMTP network, when relaying OpenSMTPD will always look for a pki entry matching its hostname and present a client certificate. This works transparently and takes into account hostnames table, source address and other tweaks.

mta improvements

Our MTA now always attempts TLS before falling back to plaintext. As stated above, it can also require remote hosts to present valid certificates and it will always attempt to present one itself.

We implemented support for AUTH LOGIN which was missing and made a lot of improvements to the logic used to optimize transfers. The mechanisms range from detecting hosts which we can't establish a valid session with, to keeping a connection alive a few seconds to avoid a round-trip if an envelope gets scheduled for that same host just a few seconds later.

To summarize, we tried to make our MTA smarter so it takes less resources locally while not beeing seen as too agressive from the other side. There are many tricky details about this, if there's demand we'll post a more in-depth view of the internals ;-)

smtp improvements

We added some features that were requested by users, such as the possibily to create a listener that only listens to inet4 or inet6 rather than both.

We made it possible for listeners to require valid certificates, but also to select the banner name from a table dynamically based on the local address the client has connected to.

A common feature was to allow the Received line to hide the From part which is not really a requirement for us to display and which allows hiding details from internal network. This is done by simply indicating on the listener that it should mask source:

listen on all mask-source Finally, we did a lot of internal rework since we are currently working on the filter API and while it is disabled, we wanted the code path to enter the filter loop to detect that there was no regressions when no filters are plugged.

queue improvements

An envelope cache was introduced to improve disk-IO pattern.

OpenSMTPD is very strict about commiting changes to disk and keeping a coherent disk-based queue that it can start from if we were to crash or suffer a power down at the "wrong" moment.

The entire queue API has been designed around that constraint and this security comes with a price with regard to performances. There's not so much we can do about disk-writes, however a common pattern is for OpenSMTPD to transmit an envelope to queue which writes it to disk and notifies the scheduler ... which requests the queue to load that envelope so that it can be delivered.

By introducing an envelope cache, the envelope is written to disk but retained in queue memory so that when the scheduler requests it back we avoid a disk read which we know will happen almost right away.

The queue cache will be improved with time but this first version already had a positive impact.

We also made many improvements that I won't describe here as they are very technical and require a knowledge of the interactions between queue, mta, mda and scheduler. If people are interested, I'll go into details in a more specific post.

scheduler improvement

Like for the queue, we made TONS of improvements and I won't go into too many details either. But two of them are interesting enough that they need to be mentionned ... also they kept Eric and I working hours on investigating and fixing so just for that reason they deserve a mention.

First improvement was the breaking of envelope batches into smaller batches. For optimization reasons, the scheduler tries to pack sets of envelopes to send to the mta/mda however when there are many many envelopes going to the same destination, the batches may grow very large.

Due to our design, an event handler should never take a long time to operate otherwise it blocks the process from handling other events. In a situation, we had about 10k bounces from a major ISP and this caused the scheduler to send to the queue these 10k bounces in a row. The scheduler became hogged, it would not schedule anything until done with that task, and the queue was hogged as it would be processing this huge amount of envelopes without the scheduler letting it handle any other event. We broke the batch into small pieces allowing other events to interlace and this caused us to have much more efficient scheduling.

The other problem was an algorithmic issue... We mainly work with trees but the scheduler had one sorted-list which it used for a single purpose. While this was no problem when working with a small set of envelopes (small being below 100K), as we grew trafic and were facing larger sets of envelopes, our O(n) inserts became so consuming that it caused the scheduler to eat all CPU for a few seconds ... every few minutes. In our very busy environement, this led to late that cumulated and cumulated until it was no longer possible to deliver in time. We ended up figuring the issue and replaced with a nice tree which allowed us to deal with a much larger volume and a very relaxed smtpd. OpenSMTPD has no problem working with queues that are several hundreds of thousands messages larges, but in theory we could go much higher we just don't deal with enough volume to prove it ;-)

smtpcl improvements

The smtpctl utility which is the admin interface to the daemon has been rewritten to be much simpler and many new features were added.

It can now be used to show routing information, state of remote hosts, it can pause and resume individual envelopes and messages, and it transparently supports showing envelopes and messages even if the queue is encrypted and/or compressed.

We're planning on many new improvements to it but it's quite sexy as is ;-)

documentation improvements

We introduced a table(5) man page to describe the format of tables, something that has bothered many users and someone contributed a sendmail(8) page to document our sendmail-like interface.

tables improvements

We introduced table_passwd to provide user info and credentials in a passwd like table that is compatible with software such as Dovecot. Users can now store their recipients in a shared file and avoid duplicating information.

We made sure table_sqlite worked for all kinds of lookups, it is used in one of my side-project where an OpenSMTPD is used for a completely virtual setup where there's not a single real user.

table_mysql has been improved and is pretty much production-ready, we use it in production ourselves and do not face issues with it.

portable

I reworked the entire autotools layout to reduce the delta between master and portable branches. Every component is being dealt as a module and while I'm not done, this is already much more clean and will allow us to reduce the inheritance of unrequired dependencies.

I'll keep working on it until it reaches perfection :-)

A few words on performances and volumes

I work at a company that handles a lot of mails, and by a lot I'm talking in matters of several millions per day both incoming and outgoing.

We initially relied on other opensource software deployed on many servers and as timed passed, progressively switched many servers to OpenSMTPD 5.3.3.

After the many improvements done to the queue and scheduler, the number of servers could shrink and it is now technically possible to handle the same amount of mails (we're still talking several millions a day) on a single instance of OpenSMTPD 5.4.1 that's nowhere near busy.

I know for a fact other companies have started using OpenSMTPD with high volumes too, so people who wonder if we will be efficient enough for their uses should simply ask themselves: do I need to send and receive several millions of mails each day. If that's not the case, then you can't possibly go wrong with us... and if that's the case, well, we do that and much more efficiently than with the older software, so you probably can rely on us.

I will ask my manager if I can put out some graphs and numbers of real-life trafic as I don't believe in oriented benchmarks ;-)

That's all folks !

That's all for this first post since a long time, I will make sure to post every weeks about updates since we're very active on development.

Oh and the RSS feed will be back shortly, no need to tell me ;-)

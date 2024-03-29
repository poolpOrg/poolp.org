---
title: "OpenSMTPD REST queue"
date: 2012-06-06 14:44:10
category: OpenSMTPD
authors:
 - "gilles"
categories:
 - technology
---

This is the first post of a series to illustrate and describe a "proof of concept" code by Charles, Eric and I. I will describe the features as they are implemented.

Since mid-November 2011, OpenSMTPD offers support for an extensible queue API. The queue_backend API allows a developer to write a custom storage driver by implementing a small set of functions that take care of storing, updating and removing envelopes and messages. The details behind these functions aren't exposed to the daemon which only manipulates message and envelope identifiers.

When the feature was committed, I said that "we should know be able to store the queue anywhere" to which an OpenBSD hacker joked about being webscale with a MongoDB backend. Little did he know that using a cloud was (one of) the real motivations to that change ;-)

OpenSMTPD ships with a single queue backend, namely the queue_fsqueue backend, that implements a file-system store using a hashed layout. Technically, we SHOULD be able to use ANY backend as long as it allows key/value storage and linear walk (thought linear walk of a subset is preferable ;-).

Now, what would we gain from storing the queue in a cloud ?

SMTP servers tend to store their queue locally. When my primary MX is down, another server, called a "backup MX" is going to have to accept the mail and store it locally, so that when my primary MX is up again that backup MX sends the mail so my primary MX can store it locally. After what it can finally decide to deliver it, either locally or by relaying to another MX.

We actually have a hierarchy of prioritized servers, each accepting mail for a domain and keeping it locally until a server with a higher priority is available again. That sounds safe, but it is actually rather fragile as it relies on EVERY MX doing the proper work to ensure nothing gets lost. Technically, it shouldn't receive much mail, but in practice my secondary MX never has an empty queue and if this is true for a site as small as poolp.org, I'd guess secondary MX at bigger sites should also have mails they don't want to lose on their secondary MX.

To be really safe, EACH MX server be it primary or secondary should ensure queue redundancy and replication to another machine so that a mail never gets lost if the machine goes down temporarily or permanently. Of course, I don't think I know anyone doing that. Of course, I don't even do that. In place, we rely on statistics that it is unlikely to happen, or that it happens so rarely that it's not worth the trouble.

Nowadays, there are plenty of solutions to store mails on distributed storages. This could be an opensource solution deployed on a set of servers you own, or a service provided by Google, Amazon or even that awesome company called Scality. By sharing a queue that provides high-availability and replication, we can ensure that the burden is no longer on the MX. As far as incoming mail is concerned, ALL MX are equivalent and can be cheap and low end machines with small disks, that don't provide replication and backups. If a mail is accepted by a specific MX, it enters THE SAME QUEUE as the primary MX in charge of delivering the content.

Technically this means we can remove the priorities and create a pool of many MX with identical configuration that all accept incoming mail.

If we face a peak and need to scale ... we just add clones to the pool !

Nice theory, how long before it is reality ?

I told Charles and Eric that it would be very nice if we had a REST backend. This would allow people to write small frontends to their storage backend, and let OpenSMTPD communicate with these frontends. With this, we could let OpenSMTPD use any cloud instead of showing a preference to one (even though SCALITY offers the best of all worlds, but that's just a totally biased opinion ;-)).

Since I couldn't provide access to a scality RING to Eric and I was already familiar with Google's AppEngine API, I installed the SDK and started writing the frontend to their datastore. The beauty of our API is that if it works for ONE, we know it will work for EITHER ONE. I thought it would take a few days to achieve the result I wanted but after about an hour I had a working REST server and had sent Eric a mail with the description of the API. Basically, the goal of the front-end is just to map an URL to a storage operation, so it was really about writing a few dozens lines of code.

The next day, Eric told me that he had cloned the API on his machine to ease development and that he had started writing the backend. Yes. He had rewritten a frontend just to ease development because it was much easier than using the remote frontend. That speaks a lot about how easy it is to write one for your own custom storage backends.

He asked me to make somes changes to the frontend API that would make things easier and more efficient for him, and after about another hour of doing things right I left him with the latest version of the REST server deployed on Google's cloud.

When I woke up the next day, I had a mail in my mailbox saying that "his queue is web-scale".

Eric's backend uses an asynchronous HTTP client he had written and crafts the queries that are expected by the frontend. The code is not in a committable state yet, but it does work and paves the road for many many more interesting features to come.

With his backend, OpenSMTPD no longer stores mails locally, it has its queue deported to Google AppEngine and fetches/commits there. I can start another instance of OpenSMTPD with the same configuration on another machine that sees the exact same queue.

In one word, it's ... AWESOME ;-)

Where do I get the code ?

You don't, the PoC is not finished yet, we have three steps missing... each as funky as this one ;-)

We will very likely make the REST backends part of the official distribution as they are independent of the storage solution, however we won't ship the frontends as they are tied to the storage solution (and commercial companies) and it's not acceptable with regard to OpenBSD rules.

The frontends will be distributed unofficially, we'll sort out how in time.

What's next ?

WAIT AND SEE ;-)

You can join us on #OpenSMTPD @ irc.freenode.net to discuss and help.

We have a mailing list available at misc@opensmtpd.org where we can discuss OpenSMTPD related stuff that is not OpenBSD-specific. To subscribe, just send a mail with subject: [misc] subscribe

OH, and just for the record, NO I have not been paid by scality to place their product here, I just like what we do there ;-)


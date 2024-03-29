---
title: "g2k12: OpenBSD hackathon"
date: 2012-07-09 23:52:31
category: OpenBSD
authors:
 - "gilles"
categories:
 - technology
---

Yesterday, after nearly missing my plane by 5 minutes, I finally managed to make it to Budapest for g2k12, the OpenBSD general hackathon.

While this is not my first hackathon, this is the first general one with many hackers working on all kinds of subsystems ranging from ports to network and kernel. It's fun to see many people focused on improving different areas which you use daily, sometimes without even noticing, and doing it with as much fun as you are having fixing your own area ;-)

<p>There are way too many changes happening to mention them all, and they will probably be part of an Undeadly series of article, so I'll just focus on what Eric, Charles and I did. I apologize in advance for the low redactional quality of this post, but it's late, I'm tired and it's about hotter here than on the sun so I will probably be too lazy to read myself. Anyway, this is just a summary of what happened since yesterday:</p>

<p>First of all, I scratched the Github repository for OpenSMTPD and reimported it using 'git cvsimport' to retain the 4 years of history. Worked like a charm, there are still a few glitches in the workflow but they should be sorted soon.</p>

<p>In the past, OpenSMTPD supported multiple queues but it turned out to be a useless features which made the queue API slightly more complex. Eric had started simplifying and removing the code that was superfluous but we still had this queue_kind thing that allowed the logic to determine which queue it was dealing with. Charles got rid of it to simplify our queue code further.</p>

<p>Meanwhile, Eric (finally) committed his asynchronous resolver in the libc. This allowed him to get immediate feedback from other hackers who confirmed that it was pretty impressive and improved drastically some ports. There are still some shortcomings but this is a very nice move forward and allows to remove the asynchronous resolver from OpenSMTPD to keep the code base small.</p>

<p>I then plugged the text_to_relayhost() code which allows expressing relays as an URL. This means that:
<i>
    accept for domain foobar.org relay via mx1.poolp.org \
        port 666 tls auth
</i>
becomes:
<i>
    accept for domain foobar.org relay via "tls+auth://mx1.poolp.org:666"
</i></p>

<p>It is nicer as when relay maps are finished, one rule will be able to refer to multiple relays that do not necessarily share the same protocol and options. Also, I think that it looks much better and is much easier to remember ;-)</p>

<p>Recently, Eric and I did a quick benchmark of OpenSMTPD to figure out what were the bottlenecks. The benchmark was quite interesting and I think it deserves a post by itself so I won't go too much into the details. Point is, after a quick discussion yesterday evening I removed the /envelopes/ subdirectory of each message to store envelopes at the same level as the message they are carrying. This saves an extra mkdir() call per message, not much but saves 3 seconds on the enqueuing of 1000 mails. Also, Eric noted that the use of 24 bits for our buckets caused performances to degrade when our queue is full because of the readdir() iterations on a dir entry with 4096 entries. We reduced to 16 bits, which allowed to save a few more seconds. There's still lots of room for improvements but the performances area I will really need to write a full post about, we have fancy graphs and stuff to show how much we understand the bottlenecks and how we can be confident we will be damn fast ;-)</p>

<p>Charles then adapted portable OpenSMTPD to the removal of ASR and got a build that would link against it externally. It was committed to Github, no snapshot generated yet but we will generate one tomorrow for sure.</p>

<p>I started simplifying the runner code. The runner is actually a scheduler, it was called "runner" because it used to iterate over runqueues which we no longer have. Since it is a scheduler, I renamed it and made sure everything that refered to it would refer to it with the appropraite name. Also, since the scheduler code is tricky we need to be able to log and Eric was annoyed by the very verbose logging. I added support for a "scheduler" trace which allows to toggle the very verbose logging using '-T scheduler'. While at it, since the scheduler logic was tricky, I started working on making it much simpler by heavily commenting it and reducing it to about 20 lines of very very simple logic.</p>

<p>There's still code to be written for the scheduling because we discussed of a new strategy for scheduling envelopes. It will (in theory) make our lives a lot simpler and allow some very nice feature we had in the pipe for long. I've started but at this point I got too drunk to keep writing that kind of invasive code.</p>

<p>Most important of all, we spent a lot of time talking, designing and thinking about what will be done this week and in the weeks and months that follow. These discussions are probably worth more in terms of productivity than all of the code we've written these last few hours...</p>

<p>Finally, as I'm writing this, Charles is sitting at a table with me in the hall of the hotel working on closing one of our "known issues"... but I'll leave that for tomorrow ;-)</p>



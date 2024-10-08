---
title: "Some OpenSMTPD overview, part 2"
date: 2014-12-01 09:04:11
category: OpenSMTPD
authors:
 - "gilles"
categories:
 - technology
---

Why we killed the MFA (filter) process
--------------------------------------

For as long as I can remember, a process called MFA was created by OpenSMTPD at start time.

MFA stood for "Mail Filter Agent" and the goal of that process was initially to take care of all filtering tasks ranging from filtering senders based on the ruleset matching to starting and controlling filters.

As time passed by, we figured the lookup process was better suited for ruleset matching and the MFA process became mostly idle, we renamed it to "filter" process since the only thing it had to do was take care of starting filters and making sure they had the proper environment.

At this point, we should have realized that this was unnecessary but this process had been with us for so long that it seemed just right. After all, we gave it a name, how do you kill something you named ...

Anyways, as we improved the filter API design it became clear that the filter process kept getting in the way of us doing things in a simple way. We had all informations available at a place, yet we had to make an indirection and pass data back and forth just so the process could get a chance to acknowledge it didn't care. If the filter process is no longer needed after starting the filters, why not have them started by another process which actually does something useful afterwards instead of staying idle ?

Eric offered to take the process down and while it was heart-breaking, I encouraged him to kill it with fire.

The pony process
----------------

There's been complaints that the daemon was starting too many processes and that probably some could be merged.

After a bit of brainstorming, we figured that the smtp process in charge of accepting incoming sessions, the mta process in charge of starting outgoing sessions and the delivery process in charge of delivering mail locally were a bit related: they were all unprivileged, running as _smtpd, chrooted to /var/empty and far from being busy even in scenarios where the server was busy itself.

In addition, they all shared a common design, they all had a notion of sessions following the same pattern and we figured that without too much efforts we could merge them into a single process.

I did the initial work to merge them three into a single process and eric@ did another pass to cleanup the names of the IMSG we were passing to improve further.

Pretty neat, hu ? Indeed. However we faced another challenge: this change proved to be controversial.

While pretty much everyone agrees that this was the way to go (actually I have no idea, I didn't run a poll), some people were not too receptive to the name we gave that process: "pony". It is apparently unprofessional and should we leave it that way, gosh so much $$$ are we going to lose.

Actually "pony" was not meant to stay, it had nothing to do with the pony trend on internet and the name was chosen because we couldn't agree on a proper name for that process which did smtp, mta and delivery. Instead of spending two hours trying to find a name, I suggested we name it "pony express" as an hommage to the courageous messengers of the far west. If someone came up with a better name, we could just switch, I just didn't want to be stuck on that meaningless task. Until now, the challenge remains unsolved.

Also, I promised blambert@ a pony during a hackathon in Budapest so ... one stone, two birds.

The klondike process
--------------------

As you may have seen, this year was a terrible shot for the OpenSSL project. The Heartbleed vulnerability has shaken the internet and it even came out with a logo.

A few months earlier, for unrelated reasons I had started working on reducing the amount of time a key is exposed in the memory-space of a network facing process. I didn't solve the issue completely but tried to improve it slightly. My improvements would have helped in some cases but not with Heartbleed.

In the wake of the catastrophe, reyk@ implemented a privilege-separated RSA engine which allowed the RSA private parts (huhu) to remain COMPLETELY isolated from the network facing processes. His solution wasn't a fix to Heartbleed but a proper design which would have made it inoperant.

Of course, I immediately asked him if he could help integrate in OpenSMTPD which he did... in a process named "klondike"... because we're all "pony express" and unprofesionnal :-p

The OpenSMTPD-extras
--------------------

For a long time, we shipped countless pieces of code that didn't really belong in our repository.

Not because they're not related to OpenSMTPD but because they are not really useful in the stock daemon, they came with some dependencies that we didn't want to have or because too few people use them that we want to maintain them as part of the official release.

So I lobbied and lobbied and lobbied to convince eric@ and chl@ that we needed an OpenSMTPD-extras repository where we could move table_ldap, table_mysql, table_postgres, table_redis and table_whateverwecomeupwith to avoid keeping too many things in our main repository.

Some work was done to split the code between these two repositories and make sure that the content of OpenSMTPD-extras would build by itself. This work was mainly done by chl@ and myself, and turned out not to be too painful, it only took us a couple hours to finish the split.

The idea behind this repository is that the OpenSMTPD repository is essentially read-only, only developers can commit and the commits must follow strict rules with regard to dependencies and what they bring to the server. We don't want to ship mysql or posgres support with the daemon bringing their dependencies, and that's even more relevant now that they can be standalone backends that can be packaged separately.

By providing a separate repository, we can remove these restrictions: the OpenSMTPD repository remains strict, the OpenSMTPD-extras repository is a set of unrequired plugins that can ship with whatever dependencies they need and we won't be strict about that. Of course that doesn't mean we will allow bad code to creep in, but we will be less strict as these will be optional user contributions, kind of what "ports" are for the BSD.

The OpenSMTPD-extras repository is now packaged in OpenBSD ports at least.

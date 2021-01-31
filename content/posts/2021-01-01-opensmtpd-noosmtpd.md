---
title: "January 2021: OpenSMTPD libtls conversion and UNIX-domain sockets support, nooSMTPD"
date: 2021-01-31 00:18:00 +0200
category: opensource
share_img: "/images/2021-01-01-nothing-can-make-you-awake.jpg"
author: Gilles Chehade
---

<blockquote>
<b>TL;DR:</b>
I do LoFi now,
eric@ revived some libtls conversion work I did a while back,
I worked on UNIX-domain sockets support in OpenSMTPD
</blockquote>


# Shout outs to my patrons !

As usual, a **huge** thanks goes to the people **sponsoring me** on **[patreon](https://www.patreon.com/gilles)**,
**[github](https://github.com/sponsors/poolpOrg)** or **[liberaPay](https://liberapay.com/poolpOrg)**,
the work in this post was made possible by my **[sponsorship](/sponsorship/)**.


# Let's start with some LoFi

Relax.

<center>
    <iframe width="560" height="315" src="https://www.youtube.com/embed/n18RH8XWqL4" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>
</center>

I have a [youtube channel](https://www.youtube.com/c/GillesChehade) (subscribe ! now !) with a playlist of LoFi tracks that I mix.

I'll try to insert a new one in every monthly report if time allows ;-)


# libtls-enabled OpenSMTPD

In [January 2020](https://poolp.org/posts/2020-01-22/january-2020-opensmtpd-work-libasr-and-libtls/),
I worked on converting OpenSMTPD to the libtls API so that I could simplify TLS support and bring new features that I didn't want to implement with the libssl API.

The diff was never merged into OpenBSD and **ended up rotting in a private branch**.
This was unfortunate because I spent time making this work for portable too,
building an OpenSSL-enabled libtls that would allow distributions not shipping LibreSSL to still benefit from this TLS simplification.

Eric Faurot (eric@) has recently picked up my work and pushed it even further,
**managing to make OpenSMTPD completely libssl-agnostic while I still needed bits for the privsep crypto engines**.
He [submitted a diff to tech@](https://marc.info/?l=openbsd-tech&m=161173678114145&w=2) which is pending review.
Once his diff gets committed in OpenBSD,
I'll rework the portable code to make it work again so this benefits users on other systems.

I've been suggested by eric@ and a tech@ reader to look at [libretls](https://git.causal.agency/libretls/about/).
The problem is that libretls is not yet available in the package repositories of all distributions supported by OpenSMTPD,
and eric@ also brought changes in LibreSSL that libretls will need to catch up with before I can rely on it.
I think the best approach is to rely on the standalone code I already wrote and which is shipped with OpenSMTPD,
then once libretls is widely available I can reassess the situation.


# libtls-related features

I have three features related to the libtls change that are already written,
rotting in their own branches,
and that I will submit once the TLS diff is committed.
They make it possible to **configure the ciphers, curves and TLS versions supported for each listeners**.

The rationale behind this is that some setups require stronger (or specific) crypto constraints for specific networks.
This change would allow declaring that a specific interface requires use of a different set of ciphers or TLS version,
and allow for example to have an interface with the default settings for Internet and one with extra-paranoid settings for a specific MX.

I also had started working on other features,
like **certificates fingerprinting or OCSP stapling**,
but I stopped as I didn't want to have too much code building upon an uncommitted diff.

I'll resume working on these once eric@ commits the libtls conversion.


# `smtpd(8)` now binds UNIX-domain sockets

At the exception of the local enqueuer,
OpenSMTPD only supports SMTP sessions over network connections.
This shows in the configuration file as it is only possible to use `listen` directives with a network interface parameter...
or with the `socket` keyword which is handled as a special case for the local enqueuer.

When the daemon starts,
it creates a UNIX-domain socket to use as its control socket for the `smtpctl(8)` utility.
This was initially meant to accept control requests from root to interact with `smtpd(8)` at runtime,
but later came the need for local enqueueing and the control socket was used for that purpose too.

A special "enqueue" control command was implemented in `smtpd(8)`.
That command allows `smtpctl(8)` to request from `smtpd(8)` that it establishes an internal SMTP session and returns the file descriptor.
The `smtpctl(8)` utility then uses that descriptor to pretend that the socket was actually connected directly to the SMTP server,
and it toggles into an enqueue mode where it operates as an SMTP client.


This has three side-effects:

Because everyone is supposed to be able to enqueue a local mail,
the control socket cannot have strict permissions and has to be world-writeable.
Since it is also used for privileged control commands,
the daemon needs to explicitly check that the client is only allowed to access the enqueue mode if the user is not privileged:

```sh
$ ls -l /var/run/smtpd.sock                                                
srw-rw-rw-  1 root  wheel  0 Dec 28 06:38 /var/run/smtpd.sock
$ smtpctl show queue
smtpctl: need root privileges
```

Then,
because there is some magic involved to request the file descriptor and toggle in enqueue mode,
the control socket is not _really_ connected directly to the SMTP server.
It is not possible to connect to it with `netcat` and talk SMTP as it expects a control command sent with the `imsg(3)` framework.
The server will just ignore the client as it doesn't speak its protocol,
and the connection must be established by `smtpctl(8)` requesting the enqueue mode (or a clone that knows how to do so):

```sh
$ nc -U /var/run/smtpd.sock  
HELO ??????
^C
$
```

Finally,
users expect to be able to enqueue mail even if the daemon is not available.
This requires the local enqueuer to support an offline queue,
which itself relies on having an executable setgid to the same group as the offline directory.
Because `smtpctl(8)` is used for both control and enqueuing,
it ends up having the setgid bit itself:

```sh
-r-xr-sr-x  1 root  _smtpq  217696 Dec 23 06:42 /usr/sbin/smtpctl
```


A year ago,
I told eric@ that I believed this was a poor decision made in the early days.
Both the control and the enqueuing code could be made simpler and stricter if they were split apart.

If local enqueuing had its own dedicated socket which established a connection to an SMTP session,
then any SMTP client that knew how to connect to a UNIX-domain socket could be used as the local enqueuer.
OpenSMTPD even ships with one... `smtp(1)`.

This would simplify `smtpctl(8)` as it would no longer need an enqueue mode and a builtin SMTP client.
It would also simplify `smtpd(8)` as it would no longer need to implement the internal SMTP session and fd passing logic,
but would also no longer need to check if a user can or can't run a command: if it's not root, it can't.
The control socket and `smtpctl(8)` could both be restricted to root as they would only be used for control requests.

If you don't see the benefit behind this,
many years ago there were two bugs that allowed local users to crash `smtpd(8)` from `smtpctl(8)`.
One affected the control command counter,
which a user could not have messed up with if control commands were restricted to root,
and the other affected the fd passing for the enqueue mode,
which would not even exist if enqueuing had its own dedicated socket instead of being a control command.


This sounds nice but converting local enqueuing to use its own dedicated UNIX-domain socket is trickier than it seems.
The technical aspect is simple,
you just bind a UNIX-domain socket instead of a TCP socket,
but it raises a lot of other questions regarding the behavior of the daemon.
I will discuss that in a future post as this is still being sorted out.

What could be done right now and that wouldn't raise questions was to teach `smtpd(8)` how to bind a UNIX-domain listener.
It makes it possible to declare listeners that have UNIX-domain sockets as their endpoints and which plain SMTP clients can connect to:

```
$ cat /etc/mail/smtpd.conf |grep listen
listen on socket "/tmp/foobar.sock"
listen on socket "/tmp/barbaz.sock"
$ nc -U /tmp/foobar.sock
220 debug.poolp.org ESMTP OpenSMTPD
^C
$ printf "subject: test\n\ntest" | smtp -s /tmp/barbaz.sock gilles@poolp.org 
gilles@poolp.org: EOM: 250 2.0.0 593c830c Message accepted for delivery
$ 
```

It doesn't solve the local enqueuer issues as it still uses the control socket,
but it allows regular SMTP clients to submit mail over a UNIX-domain socket without relying on control commands.
Moving forward in this direction,
a new local enqueuer can be written that doesn't rely on the enqueue control command,
which ultimately leads to enqueuing being removed from `smtpctl(8)` and more restrictive permissions on the control socket.


I already showed the diff to eric@ but I will send it to OpenBSD next week.


# `smtp(1)` now talks Unix-domain socket

Following the UNIX-domain sockets listeners idea,
I thought it would be nice to do the client side too.

A while back,
eric@ wrote `smtp(1)` which is a simple utility to submit mail to SMTP servers on the command line:

```sh
$ cat<<EOF | smtp -s localhost gilles@poolp.org
Subject: foo

bar
EOF
gilles@poolp.org: EOM: 250 2.0.0 5cc0b0d8 Message accepted for delivery
$
```

The `smtp(1)` client didn't know how to connect to a UNIX-domain socket so I fixed that:

```sh
$ cat<<EOF | smtp -s /tmp/smtpd.sock gilles@poolp.org
Subject: foo

bar
EOF
gilles@poolp.org: EOM: 250 2.0.0 d73542e2 Message accepted for delivery
$
```

This allowed me to test my diff on the server side,
but it also made me realize that it could be used as the base for a new local enqueuer outside of `smtpctl(8)`.

The current local enqueuer is based on the femail MUA,
which was kind of hacked here and there to fit in `smtpctl(8)` and work with its enqueue mode.
It reads the mail from its standard input,
does some sanitizing and crafting,
then submits it using the SMTP protocol on a file descriptor which points to an SMTP session.

In a world where local enqueuing doesn't require `smtpctl(8)` entering a special enqueue mode,
it would be fairly easy to use `smtp(1)` as a base to write a new enqueuer.
It already reads mail from standard input,
it already knows how to speak SMTP and with my diff it knows how to connect to a UNIX-domain socket.
All that would be missing is adding the sanitizing and crafting bits which we already have.


I also already showed the diff to eric@ and will send it to OpenBSD next week with the listeners one.


# nooSMTPD: Not OpenBSD's OpenSMTPD

I didn't want to talk about this yet but since people have spotted the repository and are making assumptions,
I need to explain what that is.

In December, I started working on an MTA.
It is based on OpenSMTPD,
but it takes a different direction and has different goals.

OpenSMTPD is a general purpose MTA,
written primarily for OpenBSD,
which needs to fit the base system and care about legacy and how changes affect the user base.

nooSMTPD doesn't have these constraints.
I'm writing it primarily for myself and will happily break legacy behaviors and change the configuration file every two months if it makes things easier for me.
I intend to support some advanced features found in commercial MTA and that are unlikely to be accepted in OpenBSD because...
OpenSMTPD is a general purpose MTA.

OpenSMTPD benefits from the work I do there as I share all diffs,
but in some cases they are unsuitable for OpenBSD and this is where things diverge between the two,
it will contain diffs that don't make it into OpenSMTPD.

I'll write about it more in details in the future,
I just wanted to clarify that this isn't the fork some people think it is.
I have exchanged multiple diffs with eric@, millert@, martijn@ and even tech@ since December,
and I continue to talk about OpenSMTPD changes with eric@ every week.

I just have ideas for a different project :-)



# breaking changes in nooSMTPD

I have killed the `dead.letter` feature which allows OpenSMTPD to save a copy of a mail when it fails to submit it to the enqueuer.
Modern MUA do not make use of it,
it is solely used by legacy MUA,
and I always thought this was not the job of the MTA to handle it.
It was also a vector of attack in the past.

I have killed delivery to the `root` user for all mail delivery agents,
the only way `root` can receive a mail is through an alias to an unprivileged account.

I have merged all of the diffs from [last month](https://poolp.org/posts/2020-12-24/december-2020-opensmtpd-6.8.0p1-released-fixed-several-bugs-proposed-several-diffs-book-is-on-github/),
including the safety net that detects injection of a custom MDA for dispatchers not allowing `exec`.


# assorted portability improvements and cleanups in nooSMTPD

I reworked some code patterns confused compilers and led to false positives in warnings.
For example,
the following construct was found in multiple places,
and led compilers not knowing that `fatal()` never returned into assuming that `p` might be uninitialized in the call to `barbaz()`:

```c
int
foobar(x)
{
    char *p;

    switch (x) {
        case 0:
            p = "a";
            break;
        case 1:
            p = "b";
            break;
        default:
            fatal("die.")
    }

    barbaz(p);
}
```

I also replaced some functions,
such as `ctime()` and `localtime()`,
with their reentrant versions `ctime_r()` and `localtime_r()`.
nooSMTPD is not threaded but this raises alerts from analysis tools and,
while doing something just so a tool would shut up is not a good rationale,
in this case there's no real downside and it saves me from having to keep flagging stuff as false positives.


# What's next ?

Moving forward with all of the above.


---- 
Comments: [https://github.com/poolpOrg/poolp.org/issues/84](https://github.com/poolpOrg/poolp.org/issues/84)

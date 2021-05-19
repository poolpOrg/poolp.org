---
title: "August 2019 report: Fion, Plakar and OpenSMTPD"
date: 2019-08-25 04:00:00
author: Gilles Chehade
authors:
 - Gilles Chehade
---

    Tl;DR:
    - small inprovements to the fion window manager
    - plakar is a backup utility I wrote a long time ago that I will share
    - tons of opensmtpd stuff, mostly filters and issues handling


Shout outs to my patrons !
--
As has become the habit,
this report begins with a big thank you to my patrons,
cited by contribution then alphabetical order.

This month has been sponsored by:

- J. Derrick
- Wesley Mouedine Assaby
- Mischa Peters
- Diego Meseguer
- Edgar Pettijohn
- Jdelic
- Sean
- Geoff Hill
- Thomas
- Bleader Raton
- Nick Ryan
- Vegar Linge Halaand
- Igor Zinovik
- Paul Kelly
- Jan J
- C

I have recently switched to a 75% part-time schedule at work so that I can spend
a "free" week each month working on my own stuff, mostly opensource, without any
kind of pressure: no one knows what I'll be working on and no one but me gets to
decide how I'll spend this time.
This comes at the cost of slashing a quarter of my wage, which is sustainable but
not ideal, so while most of my "free" weeks will be spent on opensource, I'll be
doing some sponsored development or short contracts to cover some of the loss if
needed.

I opened a patreon account so that people who care about my work can sponsor it,
allowing me to spend most (if not all) of these "free" weeks publishing code for
the community. If you want me to spend more time doing this then you know what
to do: [become my patron](https://www.patreon.com/gilles) !

So thanks again to my patrons who make this possible, I will list them in all of
my monthly reports, they are the sponsors of the work I describe below.


FLOSS Weekly #543
--
This week, hosts Randal Schwartz and Jonathan Bennett
[invited me to discuss SMTP and OpenSMTPD in their show](https://twit.tv/shows/floss-weekly/episodes/543),
FLOSS Weekly.

There's not much to say besides that it was a pleasant experience,
that I really enjoyed talking to them about OpenBSD and OpenSMTPD,
and that you should go watch that episode and subscribe to the show :-)


Fion
--
I didn't work much on the [fion window manager](https://github.com/poolpOrg/fion),
it's obviously less of a priority to me than upcoming's October OpenSMTPD release.
Yet, I still managed to get a couple things in.

I started by introducing a notion of "keyboard" mode to ease interactions with the window manager:

For debugging purposes,
I had hardcoded a few keys as triggers for window management actions:
`w` would create a workspace, `d` would destroy it, `n` and `p` would switch to next and previous ones.
Then I worked on tiles,
so I reassigned the `n` and `p` keys to switch between tiles.
You get the idea,
at some point you have to be able to use both workspaces and tiles at the same time :-)

Keyboard modes allow some combinations of keys `win`+X to enter a mode specific to a fion concept.
For instance, `win+w` enters workspace mode so the next key pressed applies to workspaces,
`win+t` enters tile mode so the next key pressed applies to tiles.
This makes it easier for me to remember the shortcuts because `n` becomes `next` in whatever mode you are,
`p` becomes `previous` in whatever mode you are,
etc...


Then,
I worked on fixing one of the issues on the TODO because if I fix at least one issue each month,
I might just be able to switch to fion by the end of the year :-p

Fion displays a status bar at the top of the screen which provides informations regarding the time,
the current workspace and active tile.
The status bar is updated by a call to a function called `layout_update()` which...
updates the layout on all screens.
However, because I was learning XCB, I used the function `xcb_wait_for_event()` in my event loop,
which caused fion to block between events and the `layout_update()` function to only be called when an event wakes up the loop.
This meant that seconds in the displayed time would hang, then jump by several, it made things look laggy.

I needed to switch to a multiplexed model and luckily,
for once,
it was not tricky to find how to do it as there's a `xcb_get_file_descriptor()` function providing a descriptor to the connection.
All it took was some rearranging of the event loop,
using `poll()` to wake up on events happening on the connection,
and a timeout to act as a 'tick' so the layout can be updated disregarding if events occured or not.


I committed these improvements,
my next task will be to handle the superposition of tabs inside tiles so we can have multiple X clients sharing the same tile,
then I'll deal with killing tiles and resizing siblings,
which is the main show stopper for switching to fion as far as i'm concerned.


Plakar
--
I wasn't sure if I was going to mention this yet but it's useful enough.

I've been self-hosted since ~1999 and along the years I've dealt with backups in various ways.
I went from tar, to dump/restore, to incremental dump/restore, to rsync, to incremental rsync,
and started writing my own tools for different use-cases I had.

As I'm doing some cleanup of my private repositories I found the following ones:

```
drwxr-xr-x   2 gilles  gilles   512 Aug  3  2012 backup
drwxr-xr-x   2 gilles  gilles   512 Jul 23  2012 backuptool
drwxr-xr-x   6 gilles  gilles   512 Apr  1  2015 plakar
```

Both `backup` and `backuptool` are going to bite the dust,
they were basically experiments to come up with a one-file backup format,
providing dedup and versionning of content.
I won't enter much details as they did work but were not that interesting and I didn't use them for long,
proof that they weren't itching my scratch.

On another hand, `plakar` is far more interesting.

It is a utility that created a repository in `~/.plakar`,
and would then let you snapshot a filesystem,
splitting each file into content-defined chunks and storing them into the repository.
The chunks would be compressed and encrypted,
hard-link games would ensure that the repository can maintain multiple snapshots of the same directories at virtually no cost,
encryption would allow the repository to be pushed to a remote server since I tend to keep backups at two sites and on my google drive.

At this point,
some of you will ask "Isn't that what [tarsnap](https://tarsnap.com) do ?",
to which I'll answer "It seems to but I don't really know as I haven't used it and tarsnap seems not to allow self-hosting".

Anyways,
that repository contains a version written in Python which supports various operations:
`push` to save a snapshot into the repository,
`pull` to restore a snapshot,
`remove` to kill a snapshot,
`list` to list snapshots,
`diff` to diff two snapshots and `search` to search for files and directories inside snapshots.
In addition,
operations can be done on portions of a snapshot,
not requiring the extraction of an entire snapshot when only a few files or directories are needed.
I had started an UI which allowed `plakar` to display a randomized password and launch a local webserver expecting that password,
a web interface would then allow visualizing the snapshots and their contents.

```
laptop$ plakar list
laptop$ plakar push ~/wip/OpenSMTPD
snapshot ee30bf0b4a4f44b5bfe71c632424c4d3: dirs=47, files=426, errors=0
laptop$ plakar list
2019-08-25-05:34:01 ee30bf0b4a4f44b5bfe71c632424c4d3 dcnt=47 fcnt=426 size=28.4MB
laptop$ plakar pull ee30
laptop$ ls ee30bf0b4a4f44b5bfe71c632424c4d3/home/gilles/wip/OpenSMTPD/
README     THANKS     regress    smtpd      smtpscript
laptop$ ls /home/gilles/.plakar
chunks    pending   purge     resources snapshots
laptop$ plakar ui

To use UI, go to http://127.0.0.1:8080, and use password: 2f1f1b61350fdfbb

Bottle v0.12.17 server starting up (using WSGIRefServer())...
Listening on http://127.0.0.1:8080/
Hit Ctrl-C to quit.
```

<img alt="FION on my laptop screen" src="/images/2019-08-25-plakar_1.png" width=100%>
[full screen](/images/2019-08-25-plakar_1.png)

<img alt="FION on my laptop screen" src="/images/2019-08-25-plakar_2.png" width=100%>
[full screen](/images/2019-08-25-plakar_2.png)

<img alt="FION on my laptop screen" src="/images/2019-08-25-plakar_3.png" width=100%>
[full screen](/images/2019-08-25-plakar_3.png)

<img alt="FION on my laptop screen" src="/images/2019-08-25-plakar_4.png" width=100%>
[full screen](/images/2019-08-25-plakar_4.png)



I don't intend to release the Python code as I will not maintain it,
however to learn Golang I rewrote `plakar` in Go this week-end and I have a version that works.
It's not yet finished but the push/pull/list/trash commands are working,
I'm going to start using it on my machines and see how it goes for the next few weeks before I publish the code on Github.

```
laptop$ plakar list
laptop$ plakar push ~/wip/OpenSMTPD
2b5bec66-1799-48a1-8418-42509347eac1
laptop$ plakar list
2b5bec66-1799-48a1-8418-42509347eac1 2019-08-25 05:38:58.912591955 +0200 CEST
laptop$ plakar pull 2b5
Restoring directories: 48/48
Restoring files: 426/426
laptop$ plakar trash 2b5
laptop$ plakar list
laptop$
laptop$ ls 2b5bec66-1799-48a1-8418-42509347eac1/home/gilles/wip/OpenSMTPD/
README     THANKS     regress    smtpd      smtpscript
laptop$ ls /home/gilles/.plakar
chunks       objects      purge        snapshots    transactions
laptop$
```


OpenSMTPD
--
On the OpenSMTPD front,
I've done quite a lot of work so I will only mention the big stuff.

Errata published
---
A user reported a bug in OpenSMTPD causing the daemon to hit one of its own sanity checks and call `fatal()`,
resulting in a denial of service.

The bug affected OpenSMTPD 6.5 and prior,
so I had to investigate and prepare patches for OpenBSD 6.5, OpenBSD 6.4 as well as OpenSMTPD-portable.


Proxy-v2 support
---
I committed experimental support for proxy-v2 protocol,
based on a diff contributed by Antoine Kaufmann and reworked a bit.

What this does is allow OpenSMTPD to work with SMTP sessions proxied by haproxy, for example.
The session begins with a header which contains informations regarding the source address,
the proxy layer in OpenSMTPD will use that information to rewrite a `struct sockaddr_storage` for the SMTP engine.

I have ran the code for my own testing but it is experimental because I have only done light testing,
we need feedback from power users.


Handled a ton of Github issues
---
We had been lagging a bit behind issues on Github,
many of them being either easily fixable,
already resolved or user mistakes.

I took the time to go through about twenty of them,
resolving half in a way or another and ensuring the other half is on the right track.

Several issues are actually dependant of some projects we have,
so they are stuck until we're done with a project and will be closed in batches once this is done.
This is for example the case for several tickets related to TLS which will not be handled until we complete the switch to libtls...
which provides out of the box support for these features or makes them trivial.


filter-rspamd
---
I have pre-released version 0.1.0 of my `filter-rspamd` on [Github](https://github.com/poolpOrg/filter-rspamd).

It has been running since last month on my machines without a single issue,
so I guess it is fairly usable.

I also committed a port to OpenBSD so that it'll be possible to `pkg_add filter-rspamd`.


filter-senderscore
---
I have also pre-released version 0.1.0 of my `filter-senderscore` on [Github](https://github.com/poolpOrg/filter-senderscore).

What it does is query the SenderScore DNS to obtain the reputation of a sender IP address,
then allow blocking, flagging as Spam and/or applying a reputation-dependant delay to sessions.

I committed a port to OpenBSD for that one too so it'll be possible to `pkg_add filter-senderscore`.


filter-checksenderdomain
---
This one I don't think I'll release, but it's available on [Github](https://github.com/poolpOrg/filter-checksenderdomain).

The name should be quite obvious,
it simply performs a DNS lookup on the MAIL FROM domain to check if it resolves.
I wrote it to prove that some of our issues don't belong in OpenSMTPD but can be solved as easily in a filter.


smtp-out reporting
---
I decided to postpone the smtp-out reporting because it requires a bit of rework on the IPC between SMTP engine and filtering layer,
something not visible to people writing filters,
but which I'm not comfortable doing at this point knowing that I may not be very available in September in case I break things.

I will continue working on smtp-out reporting so it can get committed when OpenBSD 6.6 is out,
until then I'll work on stuff that are risk-free :-)


What next ?
--
I will take next few weeks to stress test OpenSMTPD and ensure our next release is rock-solid,
then I will enter some kind of code freeze until 6.6 is out because the changeset is already significant.

When I write my next report in late September,
I may or may not yet have `fork()`-ed a little human in my house,
no guarantees that the September report will be on time !

That being said,
I'm curious if the format of these reports is readable,
if you have suggestions feel free to let me know.
Should I provide more/less details, be more/less technical ?

I intend to write some technical articles regarding the internals of Plakar,
since there may be bits of interest to other projects,
including a noSQL private repo I still have sleeping out there ;-)

Am I readable ?
Let me know.

If you like my work, [support me on patreon](https://patreon.com/gilles) !

--- 
Comments: [https://github.com/poolpOrg/poolp.org/issues/60](https://github.com/poolpOrg/poolp.org/issues/60)

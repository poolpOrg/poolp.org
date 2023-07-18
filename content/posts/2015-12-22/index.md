---
title: "Home, sweet home"
date: 2015-12-22 09:53:01
category: OpenSMTPD
authors:
 - "gilles"
categories:
 - technology
---

{{< tldr >}}
	OpenSMTPD on github no longer diverges from OpenBSD.
{{< /tldr >}}

Since last episode
------------------
Sometime this year, we released OpenSMTPD 5.7.1 which was the first version that
shipped with the long-awaited experimental filters support. The result of over a
year and a half of very very deep refactor in our IO layer and the addition of a
very elegant but also very tricky new API.

This refactor took place outside the OpenBSD tree because it required breaking a
lot of code for extended period of times, which spanned over 3 OpenBSD releases.
It was near impossible to work in tree as we could not break the base daemon and
leave it in a weird state for too long, and the changes were so invasive that it
was also impossible to backout parts of it when a regression was spotted.

We eventually reached a point where it was very stable but also diverged greatly
from the daemon in base. We made a "checkpoint" by releasing 5.7.1 separately of
OpenBSD and the goal was now to bring the daemon in tree up to date.

The release was quickly followed by 5.7.2 and 5.7.3, fixing security issues that
were uncovered by Qualys during a coordinated audit and an issue spotted by an
independant user.


Bringing back OpenSMTPD to the OpenBSD tree
-------------------------------------------
Working off-tree came with a lot of burden, first because it cut us off from the
other OpenBSD hackers but also because it came with extra work.

For instance, during an internal audit we spotted a way to trigger a denial with
our control socket. This applied to both the version in OpenBSD and Github. Both
versions diverged quite a lot so the fix to one would not apply to the other and
of course it took place at worst point in time, when OpenBSD was so close to its
new release that 3 stable versions were tagged forcing us to test the fix on all
stable branches + current branch + github's master + github's portable.

Not to mention that every time someone submitted a pull request on Github we had
to adapt it to apply on the older version, and every time an OpenBSD hacker made
a cleanup to the older version, we had to adapt and apply to Github. This was no
fun at all, many of the contributions not applying cleanly and increasing a gap.

On another hand, it was very difficult to bring the changes back to OpenBSD as a
HUGE part of it was a refactor that could not be split into reviewable minidiffs
for other to look at it. Many small features could be split, but given that they
were written on top of the refactor, this implied that even taken independantly,
they had to be adapted to apply to the old code. This was considerable work that
caused us to defer, defer, defer, ...

Finally, we reached a point where it was no longer possible to defer and I dived
back into this with help from sunil@, jung@ and millert@. I took a few days off,
spent full-time on this, and we managed to make huge progress that allowed us to
fully bring OpenBSD's smtpd up-to-date after a two-months effort of spliting the
delta into small reviewable chunks.

I owe all of them a very large amount of beer.


OpenBSD's `pledge()`
--------------------
One of the most notable change that has taken place in the last months is that
we fully integrated the new OpenBSD `pledge()` system-call.

But what is `pledge()` ?

`pledge()` is a new system call that will appear with OpenBSD 5.9.
What it does is allow a program to promise it will only use certain features and
allow the system to terminate the process if it doesn't stick to them.

For example, let's imagine the following program which creates a directory:

```c
#include <sys/stat.h>
#include <unistd.h>

int
main(int argc, char *argv[])
{
	mkdir("/tmp/test", 0700);
	return 0;
}
```

If you execute it, the process creates the directory as expected:

```sh
$ cc test.c
$ ./a.out
$ ls -l /tmp|grep test
drwx------  2 gilles    wheel    512 Dec 22 10:44 test
$
```

But now, let's imagine that the program actually makes a promise that it will
only use `stdio` features, essentially IO on previously allocated descriptors
and memory allocation so the libc basic stuff works:

```c
#include <sys/stat.h>
#include <unistd.h>

int
main(int argc, char *argv[])
{
	pledge("stdio", NULL);
	mkdir("/tmp/test", 0700);
	return 0;
}
```

We run the program again:

```sh
$ cc test.c
$ ./a.out
Abort trap (core dumped)
$
```

But this time, when the process calls `mkdir()` the kernel detects that it has
violated the promises made in the program and aborts execution.


So `pledge()` is a security feature ?
-------------------------------------
At first it looks like `pledge()` is just a security feature to mitigate risks
and it DOES bring a very effective mitigation mechanism: an attacker who wants
to run arbitrary code in a compromised process still needs to respect promises
made by the program... in that arbitrary code...

In practice, it's far from being JUST a security feature, it is also a quality
feature.

When a developer converts a program to `pledge()`, he needs to understand what
features are being used, why they are being used, when they are being used and
where they are being used. If there's a wrong assumption, the process dies. In
addition, the promises can be shortened at runtime to restrict further.

In a project like OpenSMTPD, this is particularly interesting because it comes
with a multi-process design where different processes do specific things, each
being able to declare its own list of very reduced promises.

The daemon starts with a huge promise list to be able to setup its sockets and
processes, chroot, change privileges, do some filesystem stuff, then processes
rapidly reduce their promises down to what they will really use at runtime: in
many cases `stdio`.

Intuitively, having worked on this code base for so long, I had a pretty clear
idea of what the processes were doing at runtime so I thought it would be very
straightforward and would "just work (c)".

In practice, `pledge()` pinpointed a few cases where my promises were not held
and had me investigate WHY a component needed a particular feature at runtime,
when it was not part of the promises I intuitively thought off.

Most cases were really simple to solve and a matter of doing a call just a bit
too late in the code path, but in some cases it really helped refactor code to
get rid of a feature that wasn't really needed.

`pledge()` helps you write better code by getting a very clear feedback that a
piece of code is doing more than you think it does.


What's next ?
-------------
The code in Github is now a clone of the OpenBSD CVS tree and the only delta
is a version check so that it can build on OpenBSD releases that do not have
the `pledge()` system call.

The portable branch has been sync-ed with the CVS tree but it currently does
not build as it requires a bit of plumbing. We are currently working on this
so we can resume publishing snapshots.

We're going to synchronize our releases with OpenBSD again so you can expect
OpenSMTPD 5.9.1 to be released when OpenBSD 5.9 is released, though there is
a chance we release an intermediate version so people can run a version that
benefits from all the cleanups, improvements and safer "enqueuer" before the
5.9.1 release in May.

---
title: "April 2021: OpenSMTPD, plakar, ipcmsg, privsep and a small hypnosis talk"
date: 2021-04-30 21:12:00 +0200
category: opensource
authors:
 - Gilles Chehade
categories:
 - technology
---

<blockquote>
<b>TL;DR:</b>
I worked on OpenSMTPD-portable, did a lot of plakar, a lot of Go and gave a technical talk on hypnosis.
</blockquote>


# Shout outs to my patrons !

A **huge** thanks goes to the people **sponsoring me** on **[patreon](https://www.patreon.com/gilles)**,
**[github](https://github.com/sponsors/poolpOrg)** or **[liberaPay](https://liberapay.com/poolpOrg)**,
the work in this post was made possible by my sponsors.


# Let's start with some LoFi

Relax.

<center>
<iframe width="560" height="315" src="https://www.youtube.com/embed/OUaa4Q8owaI" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>
</center>

I have a [youtube channel](https://www.youtube.com/c/GillesChehade) (subscribe ! now !)

This one caused me a **copyright strike** so I hope it was worth it :-)



# OpenSMTPD-portable

I did a bit of **review and test** for diffs sent to me,
helped test a diff for a **reproducible crash** in the **new libtls code** on my machines,
and shared some of my nooSMTPD diffs with `eric@` so he could decide to reuse them or not in OpenSMTPD.

With OpenBSD 6.9 coming out,
**OpenSMTPD 6.9.0 was tagged** and `eric@` asked me if I could **synchronize the OpenSMTPD portable repository** so it matches OpenBSD.
I spent a few hours bringing back **every individual commit**,
**fixing conflicts** and ended up with a **libtls-powered** OpenSMTPD-portable which ... didn't build anywhere because **the world still uses OpenSSL**.

Since I had already dealt with this in nooSMTPD,
I brought back my **libtls compat layer** so that on systems with OpenSSL the compat layer allows **using the libtls interface on top of OpenSSL**.
This fixed the CI for **Ubuntu and Fedora**,
unfortunately the CI for **Alpine and ArchLinux** uses LibreSSL and remain broken until the latest LibreSSL is packaged there.

Because the libtls conversion is such a major change,
there will be a **delay** between the time OpenSMTPD is released for **OpenBSD** users and the time it is released for **other systems**:
the change to the libtls interface has been heavily tested and I've been running it for a while now,
but the portable adaptation of it has been **virtually untested**.
I'll send a mail this week-end to call for testing before we can tag a release.


# Plakar

I [wrote about plakar last month](/posts/2021-03-26/march-2021-backups-with-plakar) and made **a lot of progress** since then.

I'm **not publishing** the code yet as there are many things I want to finish first before the first people start commenting and bikeshedding.


## restructured the project

I'm not too familiar with Go so I didn't structure the project very nicely at first.
I spent a few hours creating specific modules and reworking things so that they are properly split.

I'm not done but it is starting to look less shameful :-)


## removed namespaces

I initially thought namespaces within a plakar was a nice idea,
but the more I played with them the more I realized it wasn't and didn't bring any benefit over creating multiple plakars.
I decided to remove namespaces to simplify things.


## plakar-level encryption

I brought **support for encryption** and tried two different approaches:
**snapshot-level** encryption and **plakar-level** encryption.

With the first approach, a plakar repository doesn't care about encryption.
It is **the snapshots themselves** that are encrypted on an **individual basis** and the same plakar can host both encrypted and cleartext snapshots.

With the second approach, **a plakar repository is initialized as encrypted or cleartext**.
The snapshots are automatically encrypted if needed and the same plakar ony hosts all encrypted or all cleartext snapshots.

I played with both but decided to go with the **second approach** because the first one came with additional unnecessary complexity.


## encryption macro-details

A user generates a **`P384` keypair** as well as a **random master key**,
the bundle is protected by a **`pbkdf2`-derived passphrase**.

When pushing to an encrypted plakar,
**the chunks and objects are `aes256-gcm` encrypted** using a **subkey** encrypted itself by the master key.

The snapshot index containing all the checksums for all objects and chunks is encrypted and **signed**.

Upon restore,
the index signature is **verified** and the chunks are decrypted.


## keypair and master key generation

The `P384` keypair and master key bundle is generated using the `plakar keygen` command:

```sh
% plakar keygen 
passphrase: 
passphrase (confirm): 
keypair saved to local store
%
```

This results in the passphrase-encrypted bundle being saved to `~/.plakar/plakar.key`.


## plakar initialization

To be able to create snapshots,
an initialized plakar repository must be available.
A local cleartext plakar is **initialized by default** in `~/.plakar` so that the command will work out of the box for local snapshots.

However it can also be initialized elsewhere,
**defaulting to an encrypted plakar**:
```sh
% plakar init /tmp/plakar
passphrase: 
/tmp/plakar: store initialized
% 
```

or can be made cleartext by passing the `-cleartext` option:
```sh
% plakar init -cleartext /tmp/plakar.ct
/tmp/plakar.ct: store initialized
%
```

## Reworked the command line interface

Last month,
to use an alternative plakar instead of `~/.plakar`,
it was necessary to use the `-store` option which I disliked...
so I reworked the command line to introduce a notion of **direction**.

When using the default plakar,
no change is required:
```sh
% plakar push /private/etc
% plakar ls
2021-04-30T22:35:35Z 702b5b48-15dc-41cf-bfc1-1c8b94d1e985 3.1 MB (files: 242, dirs: 41)
% plakar pull 702b5b48
%
```

But when using an alternate plakar,
instead of providing the `-store` option it is now possible to **push `to`** a plakar:
```sh
% plakar push /private/etc to /tmp/plakar.ct
%
```

.. and **run commands `from`** a plakar:
```sh
% plakar ls from /tmp/plakar.ct            
2021-04-30T22:30:51Z d499fd92-01cb-4eb3-bb60-b147417b68a1 3.1 MB (files: 242, dirs: 41)
% plakar pull d499fd92 from /tmp/plakar.ct
%
```

**All commands** support the direction option.


## Generate a tarball

I thought it would be nice if I could restore a snapshot or part of it **into a tarball**,
as I often want to extract a bit of a snapshot on a machine to send elsewhere.

I introduced a `tarball` command which allows generating a tarball:

```sh
% plakar tarball d499fd92 > /tmp/d499df92.tar.gz
```

It also supports partial restore:

```sh
% plakar tarball d499fd92:/etc > /tmp/d499df92_etc.tar.gz
```


## plakar server and client

I implemented a **proof of concept** for a plakar server and client protocol,
allowing the use of plakar **over the network**.

On one end, I initialize a plakar repository and run a server from it:
```sh
nas% plakar init /tmp/plakar
passphrase:
/tmp/plakar: store initialized
nas% plakar server 192.168.0.2:2222 from /tmp/plakar
passphrase:
```

On the other end, I simply push to a remote plakar:
```sh
laptop% plakar push /etc to plakar://192.168.0.2:2222 
```

There is absolutely **no difference or limitation** from the client point of view,
any command that works locally will work remotely just as well.

It is even possible to **chain servers** in order to **proxify** a plakar server:

```sh
nas% plakar server 192.168.0.2:2222 from /tmp/plakar
passphrase:

gate% plakar server 192.168.0.1:2222 from plakar://192.168.0.2:2222

laptop% plakar push /etc to plakar://192.168.0.1:2222 
```

Does it serve a purpose ? nope, it just **works by accident**.


## plakar ui

Finally, I implemented a **basic web UI** so that I could browse the snapshots easily.

```sh
% plakar ui
Launched UI on port 40717
```

This opens a web browser which lets me browse the snapshots **as a filesystem**:
<center>
    <img src="/images/2021-04-01-plakar1.png">
</center>

<center>
    <img src="/images/2021-04-01-plakar2.png">
</center>

allows me to inspect **individual files**:
<center>
    <img src="/images/2021-04-01-plakar3.png">
</center>

get a **preview**:
<center>
    <img src="/images/2021-04-01-plakar4.png">
</center>

including of **images, pdf, videos or sound**:
<center>
    <img src="/images/2021-04-01-plakar5.png">
</center>

and even **search** for files matching a pattern in every snapshot:
<center>
    <img src="/images/2021-04-01-plakar6.png">
</center>


Because the web UI is a plakar client that doesn't do anything but plakar commands,
it can be launched from **any plakar** store,
cleartext or encrypted,
local or remote.


# Some work in Go

I first wrote Go code with [filter-rspamd](https://github.com/poolpOrg/filter-rspamd) and
[filter-senderscore](https://github.com/poolpOrg/filter-senderscore) two years ago,
but never really dived into the language more than these two small projects because I still have some **love-hate issues** with some aspects of it.

I decided this month to get more familiar with it and started looking at what it would take to write a tiny daemon,
not just a program that runs an endless loop and does all work in the same process,
but one that does things the OpenBSD style with privileges separation,
message passing and fd passing.

I was not **disappointed**:
there's not much out there to do that.


## go-ipcmsg

The first thing that is missing is a package that provides something like the `imsg(3)` framework.

For those not familiar with it,
the `imsg(3)` framework provides functions that allow two processes to exchange messages,
**including file descriptors**,
while guaranteeing that messages are always received whole.

Typically,
you will create a `socketpair(2)` before `fork(2)`-ing a process.
The parent and the child will both use one end of the pair to communicate with each other,
using the `imsg(3)` framework that will take care of buffering I/O and exposing full messages to receiving end.

There are a lot of gory details on how it achieves this and requires understanding iovec,
the semantic of `sendmsg(2)` and `recvmsg(2)`,
how control messages and SCM_RIGHTS works,
as well as some of the side effets of cmsgbuf alignments.
I did a bit of work related to resources exhaustion there a few years ago and,
while the interface is lovely,
I can't say I really missed diving in that code.

When I figured that there was nothing similar in Go,
I started reproducing the `imsg(3)` API but realized that while the API was fine in C,
it could be made **much simpler in Go** using the language-provided **channels** and **goroutines**.

So I wrote a `Channel` struct which creates two channels,
one for reads and one for writes,
and associates them to a `socketpair(2)` end:

```go
// parent process main routine, forks a child then sets up an ipcmsg
// Channel on the socketpair, returning read and write channels. The
// channels can be used to emit messages to the child process.
//
func parent() {
    pid, fd := fork_child()
    child_r, child_w := ipcmsg.Channel(pid, fd)

    // send a message to child
    child_w <- ipcmsg.Message(42, []byte("foobar"))

    // read a message from child
    msg := <- child_r
    log.Printf("Received message: %s\n", string(msg.Data))
}

// child process main routine, sets up an ipcmsg Channel on fd 3,
// the socketpair end inherited from parent,
// read and write channels can be used to communicate with parent
// process.
//
func child() {
    parent_r, parent_w := ipcmsg.Channel(os.Getppid(), 3)

    // receive a message from parent
    msg := <- parent_r
    log.Printf("Received message: %s\n", string(msg.Data))

    // write a message back
    parent_w <- ipcmsg.Message(42, []byte("barbaz"))
}
```

In practice,
you wouldn't just inline the calls but the parent and the child would call a **dispatch function** to handle messages and act upon them,
something similar to below:
```go
const (
	IPCMSG_PING = 1
	IPCMSG_PONG = 2
)

func dispatcher(r chan ipcmsg.IPCMessage, w chan ipcmsg.IPCMessage) {
    for msg := range r {
        switch msg.Hdr.Type {
        case IPCMSG_PING:
            log.Printf("data: %s\n", string(msg.Data))
            w <- ipcmsg.Message(IPCMSG_PONG, []byte("barbaz"))
        }
    }
}

func parent() {
    pid, fd := fork_child()
    child_r, child_w := ipcmsg.Channel(pid, fd)

    dispatcher(child_r, child_w)
}
```

The file descriptors passing is handled with the function `MessageWithFd()` which takes an additional parameter:

```go
    fd, err := syscall.Open(os.Args[0], 0700, 0)
    if err != nil {
        log.Fatal(err)
    }
    w <- ipcmsg.MessageWithFd(IPCMSG_PING, []byte("barbaz"), fd)
```

The sending end will have the descriptor closed upon sending,
the receiving end will be receive a message with a HasFd option set in its header and an open descriptor:

```go
    msg := <- r
    if msg.Hdr.HasFd == 1 {
        if msg.Fd == -1 {
            // expected a descriptor but got none, handle this
        } else {
            syscall.Close(msg.Fd)
        }
    }
```

The package is already commited on [Github](https://github.com/poolpOrg/ipcmsg),
however **it hasn't been heavily tested** and I **would not recommend using it** for anything serious before it has matured a bit.
I would **LOVE** to receive testing feedbacks though !


## go-privsep

The second thing missing is the ability to easily do privileges separation,
that is **creating multiple processes** that are **inherited from a parent process** with **different privileges**,
and have the ability to **communicate one with another**.

On OpenBSD,
daemons follow the **fork+reexec** pattern which boils down to the following:

The daemon is started,
it forks all of its child processes after setting up the plumbing for communicating with them,
and each child process re-executes itself so that it benefits from ASLR and doesn't retain the memory layout of the parent process.

This is an **improvement** over the more widely used fork pattern,
where each child inherits from parent,
because **the reexec causes all inherited data to be lost**.
It prevents inheriting sensitive information by accident at the cost of forcing the parent to send back the necessary information to each process.
And again, it makes all child processes benefit from ASLR too which is nice.

This pattern was popularized in OpenBSD **a few years ago**,
long after many of the daemons were already in place,
so each one did what it could to make it happen.
There was **no attempt at unifying this through a framework** like was done for IPC and `imsg(3)`.

Since I had a good understanding of how to do that and no framework to inspire myself,
I worked on a `privsep` package that would make it easier to write such daemons.
The idea is that a daemon will describe the different processes that compose it,
as well as the communication channels between them,
and the `privsep` package will then handle all the plumbing so that processes are created according to expectations.

Here is a simple example of how it works:

```go
package main

import (
	"log"

	"github.com/poolpOrg/ipcmsg"
	"github.com/poolpOrg/privsep"
)

const (
	IPCMSG_PING = 1
	IPCMSG_PONG = 2
)

func parent_main() {
	<-make(chan bool) // sleep forever
}

func foobar_main() {
	parent := privsep.GetParent()
	parent.Write(ipcmsg.Message(IPCMSG_PING, []byte("abcdef")))
	<-make(chan bool)
}

func parent_dispatcher(r chan ipcmsg.IPCMessage, w chan ipcmsg.IPCMessage) {
	for msg := range r {
		if msg.Hdr.Type == IPCMSG_PING {
			log.Printf("[parent] received PING, sending PONG\n")
			w <- ipcmsg.Message(IPCMSG_PONG, []byte("abcdef"))
		}
	}
}

func foobar_dispatcher(r chan ipcmsg.IPCMessage, w chan ipcmsg.IPCMessage) {
	for msg := range r {
		if msg.Hdr.Type == IPCMSG_PONG {
			log.Printf("[foobar] received PONG, sending PING\n")
			w <- ipcmsg.Message(IPCMSG_PING, []byte("abcdef"))
		}
	}
}

func main() {
	privsep.Init()

	parent := privsep.Parent(parent_main)
	foobar := privsep.Child("foobar", foobar_main)

	parent.Channel(foobar, parent_dispatcher)
	foobar.Channel(parent, foobar_dispatcher)

	privsep.Start()
}
```

As you can see,
it uses the `ipcmsg` package to handle IPC between the two processes,
allowing my `pingpong` daemon to play ping pong indefinitely.
Each process can have **more specific settings** set so that it runs under a **specific user**,
a **specific group** or from a **`chroot(2)` jail**.

I still have **a lot of work** to do on this,
but I commited the [current state on Github](https://github.com/poolpOrg/privsep) anyways.


# small talk on hypnosis

This is **not tech related** but since my other main activity is hypnosis and hypnotherapy,
I spent a bit of time this month working on a 3-hours talk I gave to a community of street hypnotists and hypnotherapists in Nantes.

I've been working these last three months on a model to **describe the organization of the psychic apparatus**,
based on what I learnt, experienced and observed these last six years from hundreds of hypnosis sessions,
tons of psychology readings and many experiments in lucid dreaming.

The model describes how the psychic apparatus organization **changes with the altered state of consciousness** of a subject and what it means in terms of **access to the subconscious and repressed psychic content**.

This is **not limited to hypnosis** as it can be used to evaluate if a particular kind of therapy even makes sense,
but applied to hypnosis it is particularly useful to **understand resistance**,
what is happening in a subject when a technique is used,
what is happening when a technique isn't working,
and how to **bring the subject to the proper state of consciousness**.

I don't think this post is the right place to dive into details and,
well,
the talk lasted 3 hours and it was very superficial so I will stop there.
If people are interested in this topic,
let me know and I'll think of a way to present this as the slides I presented are not really informative without me talking over them.


# What's next ?

Not much.

May will be a calm month for me here as the **sponsoring has decreased** and I need to **do some freelance** to buy myself **spare time** in June/July.
Furthermore,
I have had to deal with some personal issues in April and I'm not in the mood to do a lot of stuff at the moment.

I'll probably do a few things,
as usual,
but I'm not sure what at this point as it depends on how much time I'll have available and how my mood evolves.



---- 
Comments: [https://github.com/poolpOrg/poolp.org/discussions/129](https://github.com/poolpOrg/poolp.org/discussions/129)

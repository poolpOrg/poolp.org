---
title: "OpenSMTPD 6.0.0 is released !"
date: 2016-09-12 08:00:00
category: OpenSMTPD
authors:
 - "gilles"
categories:
 - technology
---

{{< tldr >}}
	We just released OpenSMTPD 6.0.0 and it's quite different from former releases.
	Turns out most of the changes are not visible.
{{< /tldr >}}

A featureless release
---------------------
I managed to wrap the 6.0.0 release yesterday.

Unlike most of our releases, it comes out with almost no new feature.

The changelog fits in less than 10 lines as follows:

    - new fork+reexec model so each process has its own randomized memory space
    - logging format has been reworked
    - a "multi-line response" bug in the LMTP delivery backend has been fixed
    - connections concurrency limits have been bumped
    - artificial delaying in remote sessions have been reduced
    - dhparams option has been removed
    - dhe option has been added, supporting auto and legacy modes
    - smtp engine has been simplified
    - various cosmethic changes, code cleanup and documentation improvement

Seems like a very productive slacking, however some of these changes turn out
to be very interesting in terms of code simplification and security.

In this article I'll focus on the fork+(re)exec model change.


Memory space randomization
--------------------------
OpenBSD provides each process with a randomized memory space where pretty much
everything ending up in memory is going to have different locations each time
you run a new instance. All memory allocations using either `malloc()` or
`mmap()` are randomized, the dynamic loader is randomized and recent work was
committed so that libc gets randomized at each and every system start, the
goal being that not only the place where the symbols are mapped in memory
changes at every run, but that the symbols themselves get shuffled so that
they are randomized one relative to another.

This basically means that if we both run the same program, there's about zero
chances that we will have a similar memory layout and that functions will be
sitting at the same addresses.

In terms of security this makes exploitation particularly hard as there aren't
so many things an attacker can guess regarding the memory layout. She can't
study her own system to determine where things are located on yours, and every
time the instance is restarted, former knowledge has gone away so that anything
learnt will no longer be true once the program restarts.

This coupled with the fact that OpenBSD daemons tend to exit rather than attempt
to rexecute a crashed process, level up the barrier quite a bit.

Here, a program will simply print "the same" addresses every time it is
executed:

     $ ./a.out
     0x7f7ffffd4514  address of a stack variable
     0x1061902ffe40  address of a malloc() allocation
     0x105eeb000a64  address of a function within the program
     0x105eeb000a6a  address of the main function
     0x1060fcf67a90  address of a libc function
     
     $ ./a.out
     0x7f7ffffbb714  address of a stack variable
     0xf7cbac277c0   address of a malloc() allocation
     0xf7a05d00a64   address of a function within the program
     0xf7a05d00a6a   address of the main function
     0xf7c34ba9a90   address of a libc function
     
     $ ./a.out
     0x7f7ffffeab74  address of a stack variable
     0x15b5c11cc5c0  address of a malloc() allocation
     0x15b354800a64  address of a function within the program
     0x15b354800a6a  address of the main function
     0x15b5ab116a90  address of a libc function
     $


The OpenSMTPD bootstrap
-----------------------
The idea for fork+exec in OpenSMTPD came from a discussion between `deraadt@`
and `eric@` during a hackathon I hosted in Nantes mid-May this year.

The OpenSMTPD bootstrap process was quite simple:

Upon executation, the parent process would read configuration, build a memory
representation of it and would then create a bunch of `socketpair()` before
`fork()`-ing all of its child processes.

By the virtue of `fork()`, all children would inherit the configuration as well
as all the sockets necessary for IPC. They would only have to zero and free the
bits of configuration they don't need and close sockets meant to be used by the
other children and be set.

This was nice and all, but could be improved.

When a process `fork()`s, the child process gets an exact copy of the parent
memory space. Any symbol in the parent address space is present at the same
address in the child and only new allocations will cause the parent and child
to diverge.

So everytime you run OpenSMTPD, the processes in your instance are different
from the last time you ran it, however within a single instance the processes
share a lot of addresses together. In the following example, note how all of
the symbols share the same address in parent and child, except for the
`malloc()` call that happened after `fork()`:

    $ ./a.out
    === in parent
    0x7f7ffffd3c44  address of a stack variable
    0x8e8a457c040   address of a malloc() allocation BEFORE fork
    0x8e8181631c0   address of a malloc() allocation AFTER fork
    0x8e5ce100b34   address of a function within the program
    0x8e5ce100c20   address of the main function
    0x8e88231aa90   address of a libc function
    === in child
    0x7f7ffffd3c44  address of a stack variable
    0x8e8a457c040   address of a malloc() allocation BEFORE fork
    0x8e8ad0311c0   address of a malloc() allocation AFTER fork
    0x8e5ce100b34   address of a function within the program
    0x8e5ce100c20   address of the main function
    0x8e88231aa90   address of a libc function
    
    $ ./a.out
    === in parent
    0x7f7ffffc0684  address of a stack variable
    0x1fc99fc64100  address of a malloc() allocation BEFORE fork
    0x1fc9443a6280  address of a malloc() allocation AFTER fork
    0x1fc704400b34  address of a function within the program
    0x1fc704400c20  address of the main function
    0x1fc97c85aa90  address of a libc function
    === in child
    0x7f7ffffc0684  address of a stack variable
    0x1fc99fc64100  address of a malloc() allocation BEFORE fork
    0x1fc90bb1e280  address of a malloc() allocation AFTER fork
    0x1fc704400b34  address of a function within the program
    0x1fc704400c20  address of the main function
    0x1fc97c85aa90  address of a libc function
    $

While all the OpenSMTPD processes end up diverging by a great lot when
it comes to locally allocated addresses, the problem is that if a process
is compromised then it gives immediate information regarding all of the
other processes.

Knowing the memory address of the configuration in a child process will
immediately tell an attacker where the configuration is located in all
of the other processes, and if the proper amount of bugs is found this
can prove useful.


The new `fork()`+(re)exec model
-------------------------------
The exec family function will transform an existing process into a new
process by basically reconstructing it from the new program.

This is what happens when your shell stops being a shell and becomes
a `ls` instance for example. It forgets (pretty much) everything it
knew about the former process and starts with its own brand new
randomized memory space.

So `deraadt@` suggested that if OpenSMTPD would not just `fork()` children
but instead `fork()` them and reexecute the `smtpd` binary, then each of
the children would have its own randomized memory space.

The idea itself is neat, however not so trivial to implement because when
we reexec the whole "inherit configuration and descriptors" part goes away.
It's not just fork and exec, it's fork and exec and figure a way for the
parent to pass back all the information and descriptors back to the new
post-fork instance so it is the new instance that allocates memory and
decides where the information goes.

    $ ./a.out
    === in parent
    0x7f7fffff6984  address of a stack variable
    0x10a78e9cdc0   address of a malloc() allocation
    0x10873500b84   address of a function within the program
    0x10873500c57   address of the main function
    0x10b6b748a90   address of a libc function
    === in child
    0x7f7fffff8bc4  address of a stack variable
    0x142a9fe228c0  address of a malloc() allocation
    0x14289e900b84  address of a function within the program
    0x14289e900c57  address of the main function
    0x142ad94e0a90  address of a libc function
    
    $ ./a.out
    === in parent
    0x7f7ffffbdee4  address of a stack variable
    0x1f673b59d700  address of a malloc() allocation
    0x1f6482500b84  address of a function within the program
    0x1f6482500c57  address of the main function
    0x1f66b1b71a90  address of a libc function
    === in child
    0x7f7ffffdcc74  address of a stack variable
    0xe10ddc66e40   address of a malloc() allocation
    0xe0ec9f00b84   address of a function within the program
    0xe0ec9f00c57   address of the main function
    0xe10d3a5fa90   address of a libc function
    $

Eric reworked the entire bootstrap process so the parent reads configuration,
creates all of the sockets, forks & execs all children in a special mode...

... then have the parent pass each children only the bit of configuration
they need and the descriptors they require for IPC.

This was really hard work and I'm amazed he managed to wrap it between the
hackathon and the release freeze, this makes OpenSMTPD far more resistant to
a whole lot new range of attacks.

I'll write about the smtp engine improvements in a further article, stay tuned.

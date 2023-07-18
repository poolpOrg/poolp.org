---
title: "Writing a custom Mail Delivery Agent"
date: 2020-12-29 01:01:00 +0200
authors:
 - "gilles"
categories:
 - technology
---

{{< tldr >}}
In this article, I explain what is an MDA and how to write a custom one from scratch using only shell scripting.
{{< /tldr >}}


# Shout outs to my patrons !
As usual, a **huge** thanks goes to the people **sponsoring me** on **[patreon](https://www.patreon.com/gilles)**, **[github](https://github.com/sponsors/poolpOrg)** or **[liberaPay](https://liberapay.com/poolpOrg)**, the work in this post was made possible by my **[sponsorship](/sponsorship/)**.


# What is a Mail Delivery Agent (MDA) ?
When a mail enters an SMTP server it is initially stored in the server queue before being moved to its next destination.

If the destination is a remote machine, the mail goes back to a Mail Transfer Agent (MTA) and is transferred elsewhere, but if the destination is a local user then it is passed to an MDA for delivery.

An MDA is a program that is in charge of delivering a mail to a local user and reporting to the SMTP server if the delivery was successful or not.

The MDA interface is _somewhat_ standard so that if you write an MDA that doesn't take advantage of features specific to a particular SMTP server, it will work with others fine. This is why tools such as `procmail` or `fdm` can be used with OpenSMTPD, Postfix, Sendmail, notQmail or Exim indistinctly.

An MDA can be written in any language but to keep the examples simple and understandable by all, I decided to illustrate with shell scripting which is the poorest choice of a "language" you could make to write an MDA, be warned.


# How does it work ?
An MDA works in a straightforward way.

It is a program that reads its input from `stdin` and reports success or failure through its `exit(3)` status.

Based on this description, the following shell script is a valid MDA as far as any SMTP server is concerned:

```sh
#! /bin/sh

cat
exit $?
```

What it does is call the `cat` command, which consumes its input from `stdin` and writes it back to `stdout`, then propagate its exit value as that's how `cat` reports its success or failure.

This MDA does a poor job at MDA-ing because all it does is consume the input without writing it anywhere useful. As a result, the MDA will report success when `cat` is done reading the mail and echo-ing it back to `stdout` but the mail content will be gone forever.

This MDA could be made more useful by redirecting `stdout` to a file so that the input does not get lost:

```sh
#! /bin/sh

cat >> /tmp/mail-archives
exit $?
```

In this version, the `/tmp/mail-archives` file is opened for append and `cat` has its `stdout` redirected to it. Every time the MDA gets called for a mail, the mail gets appended to that file.

This does a good job at explaining what is expected from an MDA, unfortunately it is a broken example. 

Firstly because an MDA is executed for _each_ mail being delivered, possibly simultaneously, and in the absence of a locking mechanism this might result in concurrent writes mangling its content.

Secondly because an MDA is executed with the privileges of the _recipient user_ which means that every possible recipient needs to have write access to that file, with all the downsides you can imagine.

Let's see how to write something better.


# The `stdin` stream
The first thing to understand is that the mail is received on the standard input, `stdin`, as a stream of lines which are interrupted by an `EOF`.

There is no back and forth protocol of any kind, the MDA is the recipient of a unidirectional stream of input and does not have a side-channel to let the server know of what happens during the processing of this stream.

It consumes `stdin`, does something which the server doesn't know the details of, and tells the server when its done by exiting with a value indicating success or failure.

It doesn't get much simpler.

# sysexits(3)
To report success or failure, the MDA operates like a standard unix program: it exits with `0` to indicate success and anything else to report failure.

Since that is the only way to report information to the server, MDA can make use of `sysexits(3)` status code to let the server interpret the reason behind a failure ... or at least goes the theory.

In practice these codes are not used much and they don't impact the SMTP server behaviour much. Using the wrong code will result in nothing bad happening except _maybe_ an inaccurate error message if the SMTP server does try to interpret them.


# MDA privileges
An MDA is not a long-running process that gets started once and runs forever. Every time a delivery happens for a recipient, a process is `fork(3)`-ed with the recipient privileges in order to execute an instance of the MDA.

If `gilles@poolp.org` receives a mail and `jules@poolp.org` receives a mail, and they use the same MDA, then there will be one process owned by `gilles` and one process owned by `jules`, both running concurrently (or at least they might).

Therefore, the MDA must not assume access to resources that require specific privileges. It must access resources that are available to the recipient user it was executed for. In other words, the MDA for `gilles` may access resources that `gilles` could access using a shell if he had one.

The example MDA above was bad because the `/tmp/mail-archives` must be shared between all users for it to work and this implies open permissions (`770` if users share the same group, `777` otherwise). A simple change to create a recipient-specific file would have been enough to avoid this issue and allow a more restrictive permission of `700`:

```sh
#! /bin/sh

cat >> /tmp/mail-archives.`whoami`
exit $?
```

Note that the use of `whoami` here is not a good idea, it is here to back my point, but we'll discuss that in the next section.

Alternatively, the file could be moved to the recipient user home directory to achieve the same result:

```sh
#! /bin/sh

cat >> ~/mail-archives
exit $?
```

In that case, we'd be ok permission-wise but not with regard to concurrency as lack of locking would mean concurrent deliveries to the same user might mangle the file.


# The MDA environment
An MDA inherits an environment from the SMTP server allowing it to determine who it is running for, what was the e-mail address of the sender and recipient, and other useful information.

First comes the shell environment variables, the ones you get when logging in to an account on Unix systems:

The `PATH` variable contains the default PATH allowing the MDA to execute standard programs on the host system. In my example MDA above, I didn't write `/bin/cat` because `/bin` was exposed in the MDA `PATH`.

The `SHELL` variable contains the shell that was executed to run the MDA.

The `HOME` variable contains, well, the path to the current recipient user home directory.

The `LOGNAME` and `USER` variables contain the recipient user. This is the owner of the current process, the one whose `HOME` is set for.


Then comes the MDA specific ones:

The `DOMAIN` variable contains the domain name of the recipient at the time of the SMTP session (that is before any aliasing).

The `LOCAL` variable contains the user-part of the e-mail address that was used during the SMTP session (that is before any aliasing). `${LOCAL}@${DOMAIN}` is the e-mail address that was used in the SMTP session.

The `RECIPIENT` variable contains the recipient e-mail address following all aliasing.

The `SENDER` variable is set to the e-mail address of the sender or to an empty string in case of a MAILER-DAEMON bounce.

The `EXTENSION` is set to the extension if there's any used with the recipient e-mail address. The extension is what follows `+` and precedes `@` in e-mail addresses such as `gilles+1@poolp.org`

As you can see, there's a few of them covering about all session informations needed to properly execute an MDA. If we were to take a silly broken example from above:

```sh
#! /bin/sh

cat >> /tmp/mail-archives.`whoami`
exit $?
```

We could replace it as follows to make use of the environment:

```sh
#! /bin/sh

cat >> /tmp/mail-archives.${USER}
exit $?
```

It would be better, but what if we wrote a better MDA, one that is usable without concurrency or permission issues ?


# Writing a `maildir`-like MDA
In this section, we will write an MDA that delivers mail to a `maildir`-like structure.

Contrarily to the previous examples, this one will ensure that there are no issues if multiple instances run concurrently for one or many users.

First, we'll write the base for our MDA, the `stdin` reading loop:

```sh
#! /bin/sh

cat
exit $?
```

Then, we need to figure where the `maildir` should be located and create the directory structure if needed, applying restrictive permissions. Because we know that the MDA is provided with a `HOME` environment variable, we can make use of that:

```sh
#! /bin/sh

MAILDIR=${HOME}/Maildir

umask 077 
test -d ${MAILDIR} || mkdir ${MAILDIR}
test -d ${MAILDIR}/cur || mkdir ${MAILDIR}/cur
test -d ${MAILDIR}/new || mkdir ${MAILDIR}/new
test -d ${MAILDIR}/tmp || mkdir ${MAILDIR}/tmp

cat
exit $?
```

At this point, we have dodged issues with two different users running the same MDA since we deliver to a recipients home directory but we have to worry about the same MDA being executed concurrently for the same user.

This could be handled with a lock, as is done for `mbox`-style mailboxes that use a single file per user, but `maildir` has a more elegant solution to solve this: generating unique filenames in a temporary directory, then atomically renaming to the destination directory.

By writing to a temporary directory, the file is not yet visible by the user until it is finished being written, at which point the atomic rename makes it visible. If any error happens in between, the file remains in the temporary directory and can be purged after a while, but there is no risk of exposing a partial file.

We'll adapt the MDA to save a copy of the mail from `stdin` to a temporary file with a unique name. To construct the name, I decided to go with a unix timestamp, followed by a 64-bits random value and the local hostname. This ensures that there can't be collisions if generated on two machines sharing the same network storage due to the hostname, and that collisions are unlikely on the same host as they would require a 64-bits collision of a random value happening within a second. Once the temporary file has been written and `cat` reported success, it is moved to the `~/Maildir/new` which makes it visible to clients:

```sh
#! /bin/sh

MAILDIR=${HOME}/Maildir

umask 077 
test -d ${MAILDIR} || mkdir ${MAILDIR}
test -d ${MAILDIR}/new || mkdir ${MAILDIR}/new
test -d ${MAILDIR}/tmp || mkdir ${MAILDIR}/tmp

# ie: 1609171573.deb626c88c2da87f.laptop
FILENAME=`date +%s`.`openssl rand -hex 8`.`hostname`

cat > ${MAILDIR}/tmp/${FILENAME} && \
	mv ${MAILDIR}/tmp/${FILENAME} ${MAILDIR}/new/${FILENAME}

exit $?
```

We can test that it works by running the command outside the SMTP server:

```sh
gilles@mba ~ % ls -l ~/Maildir  
ls: /Users/gilles/Maildir: No such file or directory
gilles@mba ~ % sh mda.sh             
HOLA !
gilles@mba ~ % ls -l ~/Maildir/new 
total 8
-rw-------  1 gilles  staff  7 Dec 29 00:52 1609199557.db7be737b9423c87.mba
gilles@mba ~ % cat ~/Maildir/new/1609199557.db7be737b9423c87.mba
HOLA !
gilles@mba ~ % 
```

This MDA does not perform any check on the mail content and copies the stream as is but nothing prevents your MDA from doing arbitrary work.

The MDA could parse the headers to extract information, collect statistics, store the content in a database, or do anything including apply stupid rules like not allowing a delivery on rainy days.

As much as the SMTP server knows, this is a blackbox that will acknowledge having delivered, but how it did is left to the MDA.


# Conclusion
I wrote this because many people have had questions regarding custom MDA, how to write one and what they do.

I might have missed stuff, if you have questions feel free to ask and I will expand this article.

---- 
Comments: [https://github.com/poolpOrg/poolp.org/discussions/125](https://github.com/poolpOrg/poolp.org/discussions/125)

---
title: "December 2020: OpenSMTPD 6.8.0p1 released, fixed several bugs, proposed several diffs, book is on Github"
date: 2020-12-24 02:38:00 +0200
authors:
 - "gilles"
categories:
 - technology
---

{{< tldr >}}
- Crafted the OpenSMTPD 6.8.0p1 release
- Fixed several bugs in the way
- Proposed a few OpenSMTPD improvements to OpenBSD
- Working on my OpenSMTPD book
{{< /tldr >}}


# Let's start with some LoFi
Relax.

<center>
  <iframe width=560 height=315 src=https://www.youtube.com/embed/EK2cYxNmUDE frameborder=0 allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>
</center>


# About sponsorship
First of all,
I must say how **grateful and happy** I am that people **value my work enough to trust and sponsor me**,
I appreciate it a lot and it is a **huge motivation booster**.

I initially thought that after a few months I might have about 10 sponsors,
but here we are after a year or so with **over 50 sponsors** !

Please let me know what you want me to work on,
I may or may not do it (based on my interest and skills) but at least I'll have an idea how to attract more sponsors as I'm clueless.

Thank you all and hopefully when December 2021 hits, I can give a three-digits number and spend more time on coding and writing for a hobby :-)


# Preparing the OpenSMTPD 6.8.0p1 release
I spent a lot of time preparing the **OpenSMTPD 6.8.0p1 release** that SHOULD have been released in November shortly after OpenBSD 6.8 was out,
but which **I couldn't find time to work** on as I was contracted for some private work.

As I prepared the release,
I had to fix several issues,
some specific to the portable version and some that have been fixed upstream at OpenBSD.

The **[6.8.0p1 release was published today](https://www.mail-archive.com/misc@opensmtpd.org/msg05188.html)** and **contains the fixes** for the issues mentioned below.


## Fixed macOS build
The build on macOS was broken due to a missing OpenSSL include.

A user submitted a pull request that fixed the build but another user reported that this was **not enough for older macOS** that lack the `clock_gettime()` call which we use in the profiling code.

I'm not familiar enough (at all) with macOS to figure out which versions are "old" but this seems to fall into the "legacy" bucket and as far as the **latest releases** are concerned there is no known problems.

I don't intend to support all legacy systems out there but since fixing the `clock_gettime()` issue might be beneficial to portability outside of macOS, **I started tackling the issue and am almost done**.


## Fixed get_progname() bug
A bug was introduced over a release ago which caused OpenSMTPD to **improperly log the process name in maillog on some systems**.

It was weird because the process names were correctly reported in the `ps` output, they just weren't in the logs.

I traced it back to a broken feature detection in `configure.ac` leading to the `setprotctitle()` workaround not working correctly.

Issue was **fixed and committed** to the portable repository.


## Implemented an ECDSA engine for OpenSSL (sponsored development)
When OpenSMTPD 6.6.0 was released a year ago,
it brought support for **server-side ECDSA certificates** but with the caveat that it only worked with LibreSSL.
OpenSMTPD would build with OpenSSL but disable server-side ECDSA support.

People assumed that this was done because I wanted to push LibreSSL as otherwise I could have just added a few `#ifdef` and handled both,
but this was of course not the reason.

[As I wrote in June 2019](https://poolp.org/posts/2019-06-02/may-2019-report/),
the crypto operations in OpenSMTPD are done in a **special way** compared to what's commonly seen.
As a result,
adding support to ECDSA was not just about slapping a few `#ifdef` and calling the proper function but required writing a custom crypto engine which,
unluckily for me,
uses an API that **diverged between LibreSSL and OpenSSL**.

I didn't have interest in doing that work because it's not very pleasant,
but people kept asking for it and a user contacted me in private to ask if I was willing to accept a sponsored development to do that work.

I accepted and worked on it for a few days until I could start OpenSMTPD on an Ubuntu with OpenSSL and use ECDSA certificates.
A few people tested the diff and reported success so **it was committed** to the portable repository.

I'd like to thank again that user who wishes to remain anonymous,
this sponsored development helped me replace my laptop with a new MacBook Air which is lovely,
and the result **benefits all users of OpenSMTPD portable**.

<center>
  <img src="feature.jpg" />
</center>


## Plugged multiple memory leaks
As I requested feedback on the release candidate for OpenSMTPD 6.8.0p1,
another user reported a **memory leak** which sent me on a big hunt.

Thanks to a `top` output I knew it was the `lookup process`...
but that process covers several scopes as it is in charge of **DNS lookups**,
**table lookups** and **passing information back and forth between an SMTP session and filters**.
I didn't have an immediate intuition where the leak could come from as is often the case with other processes.

I was suspicious of the **filters layer** since filters are still somewhat experimental and recent,
however there are **very few runtime allocations** there and I could not find any unbalanced allocation
**[except for a minor one](https://github.com/openbsd/src/commit/21522cbeb01f50e971ec01911918d719023484fd#diff-6cb46d1f1c7fb0972d7569a29ef930a534cade4034f526f1083e4dfb9f5a6599)** that could not explain a noticeable leak.
This was confirmed when the user applied the diff and saw no improvement whatsoever.

Then I started being suspicious of **DNS lookups** and ended up diving into the `resolver.c` interface to `asr`, the asynchronous resolver.
This led to [another small leak](https://github.com/openbsd/src/commit/ce5509e44fe2267548c237d9c134a82641183226#diff-6cb46d1f1c7fb0972d7569a29ef930a534cade4034f526f1083e4dfb9f5a6599) found in OpenSMTPD and [another one in asr itself](https://github.com/openbsd/src/commit/caf7b7a2ea9bf223de3b73d4b702f2249359245f#diff-01bed4da21c6a83cc09cf6951ac885c4196a3c119bcc5bd61ac0b1ebddeca96c) prompting a diff from Eric Faurot in OpenBSD's libc.
This was slightly more noticeable than the previous one but still not very significant, it was **barely visible** on my own MX that had been running for a year without a restart.

Finally, I had a eureka moment and convinced myself to look at the handling of **regex lookup** in tables.
It turns out that a `regfree()` call was missing which caused a `regex_t` to leak for each regex lookup.
**Depending on the OpenSMTPD configuration**, the use of regex, the size of tables used by regex and the filters in place,
the leak **could grow from fully inexistant to very significant**.

I submitted a diff to OpenBSD **[that was committed today by martijn@](https://github.com/openbsd/src/commit/79a034b4aed29e965f45a13409268290c9910043#diff-6cb46d1f1c7fb0972d7569a29ef930a534cade4034f526f1083e4dfb9f5a6599)**.


## Tracked and found a possible crash in the filters state machine
As I was tracking the memory leak above,
I came up by accident with a script that caused the OpenSMTPD instance on my laptop to **crash**... just once.
I tried to reproduce multiple times without success and decided to stop tracking the leak and focus on finding what caused the crash as it was more concerning.

I spent hours adapting my script to play different kinds of sessions with varying number of recipients, concurrency levels and amounts of data sent.
Eventually, I managed to get a **kill script** that would reliably crash my instance after a few seconds of running.
It highly suggested a race condition.

I tested the script on various instances with different configurations and realised that this did not affect all OpenSMTPD setups,
but required a **specific configuration** and a **specific client pattern to trigger**.
I could not crash a default install or my main MX which makes use of a lot of features, including filters, but I could easily crash my submission MX.

The bug was tracked for two days and it ended up being a **logic issue in the filters state machine** which could cause the io channel between the SMTP engine and the filters layer to be released while filters were still processing a request. Upon return from a filter, the channel would be expected to be writeable but it would in turn be NULL.

This led to **[a fix being committed to OpenBSD-current](https://github.com/openbsd/src/commit/6c3220444ed06b5796dedfd53a0f4becd903c0d1#diff-6cb46d1f1c7fb0972d7569a29ef930a534cade4034f526f1083e4dfb9f5a6599)** and an **[errata](https://www.openbsd.org/errata68.html)** published today.


# Work on OpenSMTPD post-release
With the 6.8.0p1 release out of the way, I worked on a few diffs for future releases.


## Suggested the rename of OpenSMTPD processes (diff sent upstream)
Many years ago,
OpenSMTPD had dedicated processes for the MDA and the MTA processes.
OpenBSD hackers requested that I factor things a bit and merge these processes because they were both unprivileged and chrooted to `/var/empty`,
making it **pointless to over-isolate them**.

We couldn't find a proper name for the new process in charge of both MDA and MTA,
but **since an OpenBSD hacker had repeatedly requested that I offer him a pony** I decided to temporarily name the process `pony express` (if you don't understand why it's funny, I can't help).
Later, the joke was improved by another developer when he named the new privsep crypto agent `klondike`.

As years passed,
I considered renaming multiple times but could not convince myself the alternative names were better,
and when I found better names they would not convince others.
When a user said that these names were useless and unprofessional,
I decided to let the joke run longer.

Years later,
I think the joke has been utilized to its fullest and is now bugging me for both technical (a space in the name of a process is annoying) and non-technical reasons.

When the configuration file changed a while back to split rules into `action` and `match` sets,
the concept of `dispatchers` was introduced in the code:
an envelope is matched and based on the action,
it is scheduled to a local or a remote dispatcher.

I [sent a diff](http://openbsd-archive.7691.n7.nabble.com/diff-src-usr-sbin-smtpd-change-process-names-td404136.html) to OpenBSD suggesting that the `pony express` process be renamed `dispatcher`,
and that the `klondike` process be renamed `crypto`.
I don't know if/when this will be committed but the feedback was positive.


## Suggested explicitly requesting forward files (diff sent upstream)
Whenever a local dispatcher is used (actions `mbox`, `maildir`, `mda` or `lmtp`),
OpenSMTPD will unconditionally look for a `~/.forward` file in the recipients home directory.
This is a mechanism that today can't be turned off by an admin.

I [sent a diff](https://marc.info/?l=openbsd-tech&m=160841747511126&w=2) to OpenBSD suggesting the **introduction of a `forward-file` option**.

The idea is to have OpenSMTPD consider it must NOT try to process `~/.forward` files unless the admin explicitly requests it by adding the `forward-file` keyword to a local action.
In the example below,
I configure two domains with maildir,
one not supporting forward files and the other supporting it:
```
action "local_users_1" maildir
action "local_users_2" maildir forward-file

match from any for domain "foobar.org" action "local_users_1"
match from any for domain "barbaz.org" action "local_users_2"
```

The benefit is that this **reduces the attack surface** on setups where `~/.forward` files are not desired.
Processing these files require **an indirection through the parent process to open the file on behalf of the daemon**,
but if we know we don't want to process them **the indirection can be bypassed**.

I don't know if/when this will be committed.


## Suggested explicitly requesting custom MDA execution in forward files (diff sent upstream)
Built on top of the previous diff,
I [suggested the introduction of the `allow-exec` option for `forward-file`](http://openbsd-archive.7691.n7.nabble.com/diff-src-usr-sbin-smtpd-add-allow-exec-to-explicitly-allow-custom-mda-td404142.html).

The `~/.forward` files allow users to redirect their mails to another e-mail address,
local or not,
but they **can also be used to plug a custom mail delivery agent** such as `fdm` or `procmail`.

This is a very nice and interesting feature but it is also a vector of attack as if any bug allowed an attacker to write that file,
including a bug unrelated to OpenSMTPD,
it would allow them to execute commands with the recipient privileges.

We can't really remove the exec feature of `~/.forward` files because being able to execute a custom MDA is a very important feature.
However, there are setups were you want to support users redirecting their mails elsewhere but not allow them to run their own custom MDA.

My proposal was to **disallow execution of a custom MDA in `~/.forward` files unless the admin explicitly allows it**.
In the example below,
I configure two domains that support forward files but one does not allow execution of a custom MDA while the other does:
```
action "local_users_1" maildir forward-file
action "local_users_2" maildir forward-file allow-exec

match from any for domain "foobar.org" action "local_users_1"
match from any for domain "barbaz.org" action "local_users_2"
```

With my diff,
if a custom MDA command is placed in a `~/.forward` file then the SMTP session temporarily rejects the session until the recipient fixes the file.

I don't know if/when this will be committed.


## Suggested explicitly requesting custom MDA execution in aliases and virtual mappings (diff sent upstream)
Built on top of the previous diff,
I [suggested the introduction of the `allow-exec` option for `alias` and `virtual`](http://openbsd-archive.7691.n7.nabble.com/diff-src-usr-sbin-smtpd-add-allow-exec-to-explicitly-allow-commands-from-aliases-td404143.html).

Historically,
plugging a mailing list or similar tools was done by creating an alias to a command:
```
lists:  |/usr/local/bin/mlmmj ...
```

This is however **a very bad practice that should be discouraged** because attaching a command to an alias means there's no recipient user to run the command as,
and they **end up executed as the daemon's user**.
In every single use-case I ever met,
creating dedicated users and using their `~/.forward` files to execute the commands achieves the same result while properly isolating the commands from the daemon.

Killing support for execution of a command in aliases is a battle that I wanted to fight but that will meet a lot of resistance,
so my proposal **was to do the same as above for forward files and disable execution by default unless an admin explicitly requests it**.

In the example below,
I configure two domains with different aliases mappings,
one not supporting execution of commands and the other supporting it:
```
action "local_users_1" maildir alias <aliases_1>
action "local_users_2" maildir alias <aliases_2> allow-exec

match from any for domain "foobar.org" action "local_users_1"
match from any for domain "barbaz.org" action "local_users_2"
```

Note that `virtual` uses the same expansion loop as `alias` so it works the same:
```
action "local_users_1" maildir virtual <aliases_1>
action "local_users_2" maildir virtual <aliases_2> allow-exec

match from any for domain "foobar.org" action "local_users_1"
match from any for domain "barbaz.org" action "local_users_2"
```

Similarly to the behavior of `allow-exec` with `~/.forward` files,
if a command is found on an alias that's not allowed to execute a custom command then the SMTP session temporarily rejects the recipient until the alias is fixed.

I don't know if/when this will be committed.


## Privileged-process safety net for allow-exec
I have a diff which is done but that I haven't submitted as it depends on the diffs above and is pointless to submit until (if ?) they are committed.

What it does is add a check in `forkmda()` so that before executing a custom MDA command from an envelope,
it checks **again** that the dispatcher was configured to allow custom MDA commands through `allow-exec`.

Except for configuration changes,
this should **NEVER** happen because a dispatcher that doesn't `allow-exec` would reject the recipient during the SMTP session when it expands to a command.
However, if an attacker managed to find a bug which allows injection of a custom MDA command into an envelope,
then this safety net would prevent execution of the command by spotting the fishy situation.

On setups without `allow-exec` **this would have prevented one of the security issues that Qualys disclosed earlier this year**.

I'm trying to find a similar way to provide a safety net for setups with `allow-exec` but it's much harder as they explicitly allow custom MDA commands,
and it's not possible for OpenSMTPD to know from a custom MDA command if it was legitimate or not.


# Released new version of `filter-rspamd`
I did a minor release of `filter-rspamd` to change the versioning format and make it easier to package on OpenBSD.

Nothing fancy.


# Started releasing chapters from the OpenSMTPD book
As I've mentioned in the past,
I work on an OpenSMTPD book which is 120ish pages as of today.

My plan to release it when done does not work because I keep rewriting the same chapters over and over until I'm happy... **and I never am**.
Meanwhile the project moves forward so things get outdated and I need to rewrite them again.

I decided to take a more radical approach and work on it publicly on Github.
This way, I don't have to worry about people seeing an imperfect phrasing or pages not rendering the way I want in PDF.

You'll [find the repository here](https://github.com/poolpOrg/OpenSMTPD-book).


# What's next ?

I have great plans for 2021 but for now I don't want to disclose them.

Have a merry Xmas, a happy new year, and see you in January !



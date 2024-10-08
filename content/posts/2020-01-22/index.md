---
title: "January 2020: OpenSMTPD work - libasr and libtls"
date: 2020-01-22 06:51:00 +0200
authors:
 - "gilles"
categories:
 - technology
---
{{< tldr >}}
    - brought back libasr to OpenSMTPD, it is no longer an external dependency
    - libtls-enabled OpenSMTPD is now a thing
    - documented filters and improved reporting
{{< /tldr >}}


No shiny feature this month, ungrateful work
--
OpenSMTPD has had quite a few features implemented since its latest major release.
As we get closer and closer to the next major release,
my work on new features will slow down to focus more on getting the release in shape.

I spent this month working on plumbing and stuff that's not necessarily very visible to users,
as well as multiple features that will **NOT** be part of the next release but will be committed shortly after release is tagged.

This month I focused on simplifying OpenSMTPD portable packaging by removing the need for `libasr` dependency.
I also worked on making `libtls`-enabled OpenSMTPD possible for portable,
a long awaited change that was not trivial and that will be skipped for the next release as it is very invasive.


libasr brought back to compat layer
--
For various purposes,
OpenSMTPD relies **a lot** on DNS lookups and requires them to be **non-blocking**.

In May 2009,
because we didn't want threading in the daemon,
a former developer implemented this using a `fork()`-based approach.
In November 2010,
I replaced the `fork()`-based non-blocking resolver with `asr`,
a then experimental asynchronous resolver written by `eric@`,
which uses an asynchronous-friendly interface that can be used with a multiplexer.
In other words,
`asr` allows DNS lookups to happen in parallel without multi-processing or multi-threading,
and results to be notified through `select()`, `poll()` or similar interfaces (`libevent` in the case of OpenSMTPD).

As `asr` is not a standard interface,
OpenSMTPD initially shipped it with its compat layer so it could be built on other BSD & Linux.
Back then,
the release process didn't include major / minor / portable-only releases,
and `asr` was still a moving target with bugs being spotted every few weeks.
This led to a very annoying situation as bugs in `asr` required a new OpenSMTPD release,
sometimes a few days apart.

We decided to split `asr` from OpenSMTPD and have it as a standalone library,
with its own releases.
This was a bit inconvenient because it meant package maintainers had to to maintain two packages,
one for OpenSMTPD and one for `asr`,
but it allowed us to decorellate both releases which was very helpful at that point.

Years passed,
OpenSMTPD improved its release management and `asr` stabilized so much that four years passed without a need for release,
but this split was not reconsidered...
until recently.

In 2019,
after filters were released,
I started working heavily on improving our portable layer which [suffered multiple shortcomings](https://poolp.org/posts/2019-11-17/november-2019-report-opensmtpd-6.6.1p1-filter-greylist-and-tons-of-portable-cleanup/).
While discussing with _eric@_,
we came up with the conclusion that `asr` should now be brought back to OpenSMTPD compat layer:
unlike years ago,
rolling a portable-specific release to fix a bug in the compat layer is something we do,
and it is now inconvenient for us to have the two split apart.

The compat layers had diverged a bit so I did the work to bring back `asr` to OpenSMTPD,
conditionally compiling it if not available on the system.
Starting with the next major version,
6.7.0 due sometime in April/May,
the portable release of OpenSMTPD will no longer require a `libasr` dependency and `--with-libasr` configure flag.
It will autodetect if it is available on the system and conditionally build the compat layer one if it isn't.


libtls ... for systems without LibreSSL
--
To quote the LibreSSL home page:

> LibreSSL is a version of the TLS/crypto stack forked from OpenSSL in 2014, with goals of modernizing the codebase, improving security, and applying best practice development processes.

Among the improvements that LibreSSL brought was a new `libtls` library,
described as follow:

> `libtls`: a new TLS library, designed to make it easier to write foolproof applications.

<center>
    <img src="/images/2020-01-22-libtls.jpg">
</center>

Indeed,
as someone who started using OpenSSL and `libssl` as a student during the 2000's,
I can attest that using the library is overly complex and writing foolproof applications is a challenge.
I won't expand more because this post is not about OpenSSL bashing.
Lets just say that **it's easy to write code that works** using `libssl`,
but **it's very hard to know if that code is correct or overlooking an option or a flag somewhere**.
Part of this is because crypto is not easy and you must know what you're doing,
but a bigger part of this is because the API doesn't make it any easier:
its attempt at being extremely flexible and allowing a developer to do pretty much anything provides multiple ways for a developer to fail and shoot their own feet.

The `libtls` interface is **a wrapper** around `libssl` which exposes a much simpler interface.
It provides sane defaults,
so that missing something somewhere doesn't result in a less secure setup,
but it also provides _less_ possibilities of tweaking everything which is often better for _most_ people.
This results in TLS code that's considerably smaller,
easier to understand and less error-prone.

I have had in my plans to convert OpenSMTPD to `libtls` since it was first released,
but couldn't switch to that interface because it **requires LibreSSL** and many systems still do not provide a package for it:
a switch to `libtls` would essentially kill OpenSMTPD portable.
Another option could be to build a standalone `libtls` that carries LibreSSL's `libssl`,
[which is just what _reyk@_ did](https://github.com/reyk/libressl-deb).
His approach is a very good trade-off and I hope it catches with his standalone `libtls` becoming widespread.

In the meantime,
OpenSMTPD cannot use `libtls` interface without breaking on systems that didn't package his work and...
experience shows it might take time.
A time during which we're still forced to use `libssl` interface,
writing complex code that no one is confident reviewing.
This is highly annoying because multiple features were put on hold until the `libtls` switch,
some have been deferred for so long that its a bit difficult to defer them some more.
At some point we need to either implement them using the `libssl` interface,
which will take time and efforts for code that we don't intend to keep,
or find a way to **use the `libtls` interface with OpenSSL**'s `libssl` as a "degraded" mode.

Since `libtls` is **a wrapper** around `libssl`,
you'd figure that this shouldn't be too hard BUT LibreSSL and OpenSSL diverged on a few things,
the `libssl` interface now has a few differences in terms of structures,
functions available and options to these functions.
You can't just grab `libtls` and build it with OpenSSL,
it won't for a variety of reasons.

I didn't want to spend time working on an OpenSSL version of a standalone `libtls`:
I believe there are good reasons why LibreSSL was forked and I'd rather see more people use LibreSSL than endorsing OpenSSL myself.
So... I found a middle-ground which seemed the most pramatic to me.

I have brought `libtls` from latest LibreSSL into the **compat layer** of OpenSMTPD,
then did the **_strict minimum_** to allow it to build and link with OpenSSL **to cover OpenSMTPD's needs and nothing more**.
This allowed me to convert OpenSMTPD to the `libtls` interface simplifying _an awesome lot_ of code,
while being able to build on any system wether it ships LibreSSL or not.

So how does this differ from a standalone `libtls` ?

OpenSMTPD's configure will detect if LibreSSL is installed and use its `libtls` if available.
If it's not available,
it will detect if a `libtls` is available (_reyk@_'s work, for example),
then if neither one is available it will build a dumbed down version that contains only what OpenSMTPD needs.
**Using that version may not work on another project**.
This has the benefit that **LibreSSL is always prioritized** if available,
while still allowing us to switch to `libtls` if it isn't.
This is not ideal as it adds complexity to the compat layer in OpenSMTPD,
but this complexity allows unlocking the `libtls` conversion which will considerably simplify OpenSMTPD itself,
so I think it is a fair choice.

The work to build that compat `libtls` is done and so is the conversion of OpenSMTPD to the `libtls` interface.
It is very invasive and, like any huge work, it has a potential for introducing regressions.
Given that there's only a few weeks before the next major release,
I won't merge this for the next major release but skip it so we have a full development cycle to spot shortcomings.

The next major release of OpenSMTPD,
version 6.7.0 due April/May,
will still use the `libssl` interface but later releases starting with version 6.8.0,
due October/November,
will use the `libtls` interface.


Implemented multiple feature requests
--
Building on top of my `libtls` work,
I have implemented several of the features that were requested and left pending waiting for the switch.
They are committed in separate branches and waiting for the `libtls` work to be merged:

* selectable ciphers per listener
* selectable curves per listener
* selectable TLS protocol version per listener
* selectable SNI per listener (they're currently global)

Other features are being worked on but not finished yet:

* OCSP check
* certificate fingerprint verifications


Started documenting the filter API
--
I have also committed a `smtpd-filters(7)` man page,
which doesn't get installed yet,
that documents the protocol and events.

The feedback so far has been good and resulted in a few changes,
but having more people read it and comment would be nice.


More `smtp-out` reporting improvements
--
I discussed `smtp-out` reporting [in my last report](/posts/2019-12-24/december-2019-opensmtpd-and-filters-work-articles-and-goodies/),
but it wasn't finished and didn't cover all events.

I spent time to get this done so that we have events parity between `smtp-in` and `smtp-out`.
If you enable reporting for `smtp-in` and `smtp-out` then accept a mail for relaying,
you will first see the events happening from `smtp-in` as you accept the mail (server),
then see the same events from `smtp-out` as you relay the mail (client).

The message id and transaction id are preserved so that it is possible to track the path of an envelope through its entire life.


What next ?
--
I'm contemplating `mda` reporting,
more on that in a future report.

I started learning Go by writing OpenSMTPD filters a few months ago and now I'm learning Rust for other purposes.
Half-way through a book,
I will probably work on a real project to get my hands dirty.
I'm unsure what that project will be yet as I have a few ideas in my sleeves,
but if I come up with anything valuable I'll share it.

I will write another article next week to discuss some work that could not make it into this report.


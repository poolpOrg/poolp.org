---
title: "June 2019 report: fion, bpg and smtpd"
date: 2019-06-30 07:42:00
category: opensource
authors:
 - Gilles Chehade
---

    TL;DR:
    - started working on FION, a static tile window manager
    - revived BPG, a PGP parser
    - converted OpenSMTPD to libtls
    - wrote a library to make writing of native C OpenSMTPD filters easy
    - started writing a filter-rspamd


Thanks to my patrons !
--
First of all,
a huge thanks to my first patrons:

- Bleader Raton
- Diego Meseguer
- Mischa Peters
- Vegar Linge Halaand


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



FION: a featureless ion
--
I'm not too much of a graphical interface person,
so when I was shown the ion window manager in 2003 I became addicted and,
ever since,
it's been very hard for me to work efficiently using anything else.
This is not an elitist thing,
it is really me being easily distracted by stuff like windows not being properly
aligned and similar stupid concerns that actually slow me considerably.
I call this the "oh-a-butterfy" effect.

Unfortunately,
the last ion release dates from 2009 and is unmaintained.
This leads to multiple problems,
one being the lack of modern features, like multi-screen, which is essential for
me considering the amount of time I spend reading code and specs side-to-side.
Another problem is the fact that it's been removed from most repositories, and I
keep having to build it from source here and there.
Recently it's been a bit more difficult because on some distro,
the dependencies for ion were no longer available and I had to start patching it
here and there so it could build.

I fear the day this happens on OpenBSD because it will mean that I have to spend
time maintaining a piece of code for which there's no upstream anymore ... or be
back to working partly in console, partly in a maximized tmux when I really need
a browser visible.

I know this will lead people to tell me "but there's notion, ion's fork" or "but
you can use awesome/dwm/i3/wmii/<insert yours here>" but honestly I have tried a
lot of different window managers along the years to try getting away from ion as
it was abandoned and they never quite fit for me. I can only work if I have a wm
that does tabs within static tiles that I can h-split/v-split at will and resize
however I want. Notion came short because at least on OpenBSD it was laggy.

I might as well spend a few hours / days working on my own, learn how they work,
and release something I'm willing to maintain on the long run as it scratches my
own itches.

FION's feature
---
I didn't care that much for many of ion's feature, never used the lua binding or
the floating workspaces.

FION will simply implement what I intend to use, so at the time being, this is:

* pledge()-ed
* multiple screens of different sizes
* multiple workspaces per screen
* workspaces may be split in multiple horizontal and vertical tiles
* each tile may hold multiple X clients in tabs
* tiles may be resized
* screens / workspaces / tiles / tabs may be controlled through keyboard

As of today, I'd say I'm halfway through, FION will detect screens and setup the
workspaces to match screen size. It can create as many workspaces as you want on
every single screen. It can h-split and v-split as many tiles as you want on the
current workspace. It can iterate through workspaces and tiles. It can start the
X clients and attach them, properly sized, to the current tile. It can give the
focus to a tile through keyboard-triggered iteration of tiles or by moving mouse
over a tile.

Can you try it yet ? nope.

FION currently does not have the logic to remove tiles, you can create and split
but not kill tiles, which is not hard to do but I just ran of time this week ;-)

Also, keyboard events are not properly handled yet so I had to map features to a
simple key like 'n' or 'p' to test them. Until I have more time for figuring out
how I can implement my keyboard shortcuts so that they use control sequences, it
will not be usable for daily use.

Code is already committed on Github but again ... not usable, so stay tuned, the
first usable version will be announced on [twitter](https://twitter.com/poolpOrg)
when it's available. 


Obligatory screenshots
---
I must have spent about six hours worth of work on it and the result is quite ok
so I have good hope (no promise !) to make a first beta release during my "free"
week in July.

Here are a couple screenshots:

FION ran on my laptop a few days ago before X clients were resized to match tiles:

<img alt="FION on my laptop screen" src="/images/2019-06-30-fion_1.png" width=100%>
[full screen](/images/2019-06-30-fion_1.png)

FION ran on a bigger screen this morning:

<img alt="FION on my 3440x1440 screen" src="/images/2019-06-30-fion_2.png" width=100%>
[full screen](/images/2019-06-30-fion_2.png)

And because a project doesn't exist until it has a logo:

<img alt="FION logo" src="/images/2019-06-30-fion_logo.jpg" width=100%>
[full screen](/images/2019-06-30-fion_logo.jpg)

BPG: BSD Privacy Guard
--
While looking at old private repositories to decide if I should axe them or not,
I ran into an interesting one: bpg.

Long story short,
while I was a student in 2002 or 2003,
I was doing night shifts at a security company to monitor firewalls and since it
was essentially a passive job because I had nothing to do until the screens went
all red, I started working on BPG a BSD-licensed PGP implementation.

Then, NetBSD came up with a similar project for a Google Summer of code and they
had someone already working on it, so I gave up as I had only a few weekly hours
available for this and my work would surely be less good than someone working on
it full-time for NetBSD.

Anyways,
I spent a while looking at the code to see what could be done with it, and it is
actually not bad at all.

I have a parser that can read all PGP packets in both old & new format, map them
to appropriate structures that can actually be used: it can essentially read the
keyrings and packets generated from gpg2, though it will not decrypt/verify.

The parser code was quite clean and I didn't make changes to it except remove an
allocator to replace it with `malloc_conceal(3)`. I did a bit of code cleanup on
other parts, like armoring which was ugly, but that's about it.

I don't know what to do about that code, I have no idea how usable is the NetBSD
implementation and how interesting it is for people to have another PGP utility.
I will release the parser for sure since the code is already here and maybe it's
going to be useful to someone but I'm unsure if I should move project forward at
this point. Oh and I need a new name anyways because bpg is already taken and it
is certainly not going to be OpenBPG to avoid confusion with OpenBGP :-)

I have quickly written code to generate RSA keypairs just for fun:

```sh
laptop$ bpg key gen rsa 2048 > /tmp/rsa2048.bpg
..........................................+++...................+++.
laptop$ bpg debug parse /tmp/rsa2048.bpg
## 05 : New format Secret-Key packet (920 bytes)
        version: 4
        algorithm: RSA (encrypt or sign)
        keyid: e60e44654c3409be
        fingerprint: 7bfd297855a9c0d22a492ba2e60e44654c3409be

## 0d : New format User ID packet (16 bytes)
        content: test@example.com

laptop$ cat /tmp/rsa2048.bpg
-----BEGIN PGP PRIVATE KEY BLOCK-----
Version: BPG v0.1 (OpenBSD)

xcLYBF0YccABCACzay1zxLQzD9WvOaF5DZXKIx5nmwP70+ff4XceepFnylwe7iKzTUF6W4kv
QxOF++z9SQ5mF0melyj0+TkJXUSs7MzNjuwb4OhDLiM9uVJ2J1wxA3KivY0pVjvJPtPTlMpZ
D7Ebb1CnagP75DEcgpdzWHDSzBbgHN89ct6M7fPZ756iDxHdnhO8syr9sB4GOJyj8hC2dC4y
qJvn7G7/BdQwDov23FUGcddWXIyPMs4GGbE+mpV1U6jqZbNUkGPl+Kh8CfRU15D2/60PsrgD
3OSDo3J/DwGd2/NiVRRWVSOOtpzm4sc4j1J76nwrUJvvWRlH+AMOfDxoDu03bY2Jb/NtABEB
AAEAB/9Pw8ZhQYIbcV6+mBCBkNiXFSXfSbtrqbncfpBGrJcYXY628YfbzuzdSPSkXl2/o1Cp
CmGsYY4JQ4qh3mrNDvoJJv2mJXQysLqRo2Fnf4x5muYRpEbCsyKezgemYJgr6GpNTfyfBc4F
n8xFoB11X1mVniwKi1FgMXXOC9OFNATFTlK2EjiBCBN1g+AbO31ykqDyhuVjAisQXlrpvRSr
Jlk3zwNDgyu8jd9NyqCj3wUbp5GTj9/N+JVqm4lxqRAXUJk804w9q4Vn1x5f+lxyOw70m6vV
9xKZqKB/g2CmqFNVn8nzparvZQ45FSwKFkCz8Z0XrCJG8HvxkS0fBewu2BLBBADvAzcqblpX
9ygSDsOF/JDt9nhduLcVDgFzOagfZvh9eAny0YGQZ7rue2Vd3DKbMRBkybpn3jKpRbJC8+96
6AVaaxtoaHt7jKxZMHdjlxRPe6f9rFqETgL4a0+xyGzGd72t2OZG9pXNbydrwJPo0PSybkvR
hkyd7vP3886X1EkdrwQAwCupzj3qSKf06ATAy+31rKoIUgsHD7OuyrNl7I1FZIMWNttaEl1C
Z0bYaKHaTZbtoNk9EsISxSIXf3HB7nisKmGDe+LfXD06q9Bk1Tpjoj+zMXhJAm/NOQfbWXgJ
3/1b6EbI63o1JcBlZDXbrGv5IOY+c7gZqbOc3hD8EThHA6MD/0fQC4TuqH9nkJVyiDp0G+48
nq2oFDiDEfJ58TiSFlws/K+nuy2uGGO91iJLA6go5B87b0ihvl6A1Z4XFXuopRMcVp87n9gH
qqRLhvoBqhMdVlfPnYr8JZYu0HkzmFtCnwuQOIA8EH8FQ65ImrP5PopazIHiUwhunrTzoAWp
/4JOulbNEHRlc3RAZXhhbXBsZS5jb20=
=0fuz
-----END PGP PRIVATE KEY BLOCK-----
laptop$
```

They are not valid, generating a valid key that can be imported to
gnupg requires adding a signature packet which means I have to write the
signing code to do that.

So there we go, either the project ends up as a PGP parsing library that others
may use freely in their PGP implementations or it can evolve into a PGP library
and utility, it depends on how needed and wanted this is by the community.

I will commit what I have during July.


OpenSMTPD: 
--
What should come as no surprise: I spent most my week working on OpenSMTPD.

I have tackled two topics:

* migration from the OpenSSL to the libtls API for TLS support
* filters


OpenSMTPD: OpenSSL -> libtls migration
---
I have started working on the migration from the OpenSSL API to the libtls API
for TLS support in OpenSMTPD.

And by "I have started", I really mean "I'm almost done".

The server side is mostly finished as I just need to rewrite the SNI support
in a slightly different way. The client side is also mostly finished but needs a
bit of cleanup.

This wasn't as straightforward as I thought it would be, mostly due to the fact
that OpenSMTPD uses a low-level "io" API for all its network io and that it was
tricky to work on a progressive migration. I eventually found a way that allowed
me to migrate the client-side first, then focus on the server-side, then use
solely libtls. When this will be committed to OpenBSD, it will have to be an
atomic switch, I won't be able to progressively bring server and client side
migrations.

I also ran into some issues that were not easy to tackle, like the fact that we
used ex data in RSA / EC_KEY objects to pass pki name to the privsep crypto
engine for key lookups ... but libtls already used ex data to pass a checksum.
I could have worked-around but it made sense leaving the checksum in ex data,
as it'll be useful in the future, so a bit of refactoring was require here and
there.

So libtls-enabled OpenSMTPD is now a thing,
when its finally commited I'll be able to close multiple feature-requests that
will be trivial to implement or already solved by sife-effect of the mgiration.

**Does this mean OpenSMTPD will become LibreSSL only ?**

NOPE.
This means that instead of having OpenSMTPD detect and deal with the differences
between OpenSSL and LibreSSL as well as different OpenSSL versions, it will just
depend on the libtls interface. We can then work on having a portable libtls for
use as a wrapper around OpenSSL. This will then solve the LibreSSL vs OpenSSL vs
different versions of OpenSSL issue for good.


OpenSMTPD: Filters
---
I have resumed working on filters and this resulted in a few interesting bits.


timestamps in filter events
----
Until now,
filter events reported the time at which an event was generated using a time_t
unix timestamp accounting seconds since epoch.

I had already converted reporting events to use a timeval for sub-second precision,
so I also converted filter events to use a timeval too.


filter strderr mapped to `log_warn()`
----
I okayed a diff from `martijn@` to map the `stderr` file-descriptor in filters to
an endpoint in smtpd which logs all input with `log_warn()`.

This means that a filter calling `warnx()` or a shell script doing an `echo bleh >&2` will be logged in `maillog`,
making it simpler to see output during development and in case of issues in production.


filter-eventlog
----
I have committed to github my first native filter, `filter-eventlog` which is a
filter written in C that reports ALL smtp-in events to an append-only file, and
creating a new file every day to hold the events of the day.

This is useful for filter developers as it allows them to check what events are
generated by the daemon and which ones they should listen to in their filters.

With the lines below in your config:

```
[...]

filter mylogs proc-exec "/usr/libexec/filter-eventlog /tmp/logs"

listen on socket filter mylogs

[...]
```

Sending a mail will result in the following being written to files within /tmp/logs:

```
report|1|1561888072.457091|smtp-in|link-connect|d812eceafa0ae39e|laptop.home|pass|local:0|local:0
report|1|1561888072.457781|smtp-in|filter-response|d812eceafa0ae39e|connected|proceed
report|1|1561888072.457800|smtp-in|protocol-server|d812eceafa0ae39e|220 laptop.home ESMTP OpenSMTPD
report|1|1561888072.458290|smtp-in|protocol-client|d812eceafa0ae39e|EHLO localhost
report|1|1561888072.458773|smtp-in|filter-response|d812eceafa0ae39e|ehlo|proceed
report|1|1561888072.458778|smtp-in|link-identify|d812eceafa0ae39e|localhost
report|1|1561888072.458784|smtp-in|protocol-server|d812eceafa0ae39e|250-laptop.home Hello localhost [local], pleased to meet you
report|1|1561888072.458787|smtp-in|protocol-server|d812eceafa0ae39e|250-8BITMIME
report|1|1561888072.458790|smtp-in|protocol-server|d812eceafa0ae39e|250-ENHANCEDSTATUSCODES
report|1|1561888072.458794|smtp-in|protocol-server|d812eceafa0ae39e|250-SIZE 36700160
report|1|1561888072.458797|smtp-in|protocol-server|d812eceafa0ae39e|250-DSN
report|1|1561888072.458800|smtp-in|protocol-server|d812eceafa0ae39e|250 HELP
report|1|1561888072.459268|smtp-in|protocol-client|d812eceafa0ae39e|MAIL FROM:<gilles@laptop.home>
report|1|1561888072.459718|smtp-in|filter-response|d812eceafa0ae39e|mail-from|proceed
report|1|1561888072.460764|smtp-in|tx-begin|d812eceafa0ae39e|754cbaa4
report|1|1561888072.460770|smtp-in|tx-mail|d812eceafa0ae39e|754cbaa4|<gilles@laptop.home>  |ok
report|1|1561888072.460774|smtp-in|protocol-server|d812eceafa0ae39e|250 2.0.0: Ok
report|1|1561888072.460995|smtp-in|protocol-client|d812eceafa0ae39e|RCPT TO:<gilles@laptop.home>
report|1|1561888072.461131|smtp-in|filter-response|d812eceafa0ae39e|rcpt-to|proceed
report|1|1561888072.462172|smtp-in|tx-envelope|d812eceafa0ae39e|754cbaa4|754cbaa41a1afada
report|1|1561888072.462180|smtp-in|tx-rcpt|d812eceafa0ae39e|754cbaa4|<gilles@laptop.home> |ok
report|1|1561888072.462184|smtp-in|protocol-server|d812eceafa0ae39e|250 2.1.5 Destination address valid: Recipient ok
report|1|1561888072.462496|smtp-in|protocol-client|d812eceafa0ae39e|DATA
report|1|1561888072.462638|smtp-in|filter-response|d812eceafa0ae39e|data|proceed
report|1|1561888072.463142|smtp-in|tx-data|d812eceafa0ae39e|754cbaa4|ok
report|1|1561888072.463147|smtp-in|protocol-server|d812eceafa0ae39e|354 Enter mail, end with "." on a line by itself
report|1|1561888072.463866|smtp-in|protocol-client|d812eceafa0ae39e|.
report|1|1561888072.463997|smtp-in|filter-response|d812eceafa0ae39e|commit|proceed
report|1|1561888072.464034|smtp-in|tx-commit|d812eceafa0ae39e|754cbaa4|477
report|1|1561888072.465786|smtp-in|protocol-server|d812eceafa0ae39e|250 2.0.0: 754cbaa4 Message accepted for delivery
report|1|1561888072.466065|smtp-in|protocol-client|d812eceafa0ae39e|QUIT
report|1|1561888072.466294|smtp-in|filter-response|d812eceafa0ae39e|quit|proceed
report|1|1561888072.466300|smtp-in|protocol-server|d812eceafa0ae39e|221 2.0.0: Bye
report|1|1561888072.466481|smtp-in|link-disconnect|d812eceafa0ae39e
```


libopensmtpd
----
The filters protocol is a very simple line-based protocol that I already explained in previous blog posts,
but while writing filters in various scripting languages is simple,
writing a filter in C was still a bit tedious.

I started writing a `libopensmtpd` to provide a straightforward method for writing native C filters.
It is not ready yet for publishing but it is already functional and works enough to trigger all hooks.

All it takes is:

```sh
$ cc -o /tmp/filter-debug debug.c -lopensmtpd
$
```

To make the following filter usable:

```c
/*
 * Copyright (c) 2019 Gilles Chehade <gilles@poolp.org>
 *
 * Permission to use, copy, modify, and distribute this software for any
 * purpose with or without fee is hereby granted, provided that the above
 * copyright notice and this permission notice appear in all copies.
 *
 * THE SOFTWARE IS PROVIDED "AS IS" AND THE AUTHOR DISCLAIMS ALL WARRANTIES
 * WITH REGARD TO THIS SOFTWARE INCLUDING ALL IMPLIED WARRANTIES OF
 * MERCHANTABILITY AND FITNESS. IN NO EVENT SHALL THE AUTHOR BE LIABLE FOR
 * ANY SPECIAL, DIRECT, INDIRECT, OR CONSEQUENTIAL DAMAGES OR ANY DAMAGES
 * WHATSOEVER RESULTING FROM LOSS OF USE, DATA OR PROFITS, WHETHER IN AN
 * ACTION OF CONTRACT, NEGLIGENCE OR OTHER TORTIOUS ACTION, ARISING OUT OF
 * OR IN CONNECTION WITH THE USE OR PERFORMANCE OF THIS SOFTWARE.
 */

#include <err.h>
#include <inttypes.h>
#include <stdlib.h>

#include "opensmtpd.h"

extern char *__progname;


/* reporting */

static void
report_link_connect(opensmtpd_ctx_t *ctx, const char *rdns, const char *fcrdns, const char *src, const char *dest)
{
	warnx("report_link_connect: rdns=%s, fcrdns=%s, src=%s, dest=%s",
	    rdns, fcrdns, src, dest);
}

static void
report_link_disconnect(opensmtpd_ctx_t *ctx)
{
	warnx("report_link_disconnect");
}

static void
report_link_identify(opensmtpd_ctx_t *ctx, const char *helo_name)
{
	warnx("report_link_identify: helo_name=%s",
		helo_name);
}

static void
report_link_tls(opensmtpd_ctx_t *ctx, const char *tls_line)
{
	warnx("report_link_tls: tls_line=%s",
	    tls_line);
}

static void
report_tx_begin(opensmtpd_ctx_t *ctx, uint32_t msgid)
{
	warnx("report_tx_begin: msgid=%08x",
	    msgid);
}

static void
report_tx_mail(opensmtpd_ctx_t *ctx, uint32_t msgid, const char *mail_from, const char *result)
{
	warnx("report_tx_mail: msgid=%08x, mail_from=%s, result=%s",
	    msgid, mail_from, result);
}

static void
report_tx_rcpt(opensmtpd_ctx_t *ctx, uint32_t msgid, const char *rcpt_to, const char *result)
{
	warnx("report_tx_rcpt: msgid=%08x, mail_from=%s, result=%s",
	    msgid, rcpt_to, result);
}

static void
report_tx_envelope(opensmtpd_ctx_t *ctx, uint32_t msgid, uint64_t evpid)
{
	warnx("report_tx_envelope: msgid=%08x, evpid=%016"PRIx64"",
	    msgid, evpid);
}

static void
report_tx_commit(opensmtpd_ctx_t *ctx, uint32_t msgid)
{
	warnx("report_tx_commit: msgid=%08x",
	    msgid);
}

static void
report_tx_data(opensmtpd_ctx_t *ctx, uint32_t msgid, size_t msg_size)
{
	warnx("report_tx_data: msgid=%08x, msg_size=%zd",
	    msgid, msg_size);
}

static void
report_tx_rollback(opensmtpd_ctx_t *ctx, uint32_t msgid)
{
	warnx("report_tx_rollback: msgid=%08x",
	    msgid);
}

static void
report_protocol_client(opensmtpd_ctx_t *ctx, const char *line)
{
	warnx("report_protocol_client: line=%s",
	    line);
}

static void
report_protocol_server(opensmtpd_ctx_t *ctx, const char *line)
{
	warnx("report_protocol_server: line=%s",
	    line);
}

static void
report_filter_response(opensmtpd_ctx_t *ctx, const char *phase, const char *response)
{
	warnx("report_filter_response: phase=%s, response=%s",
	    phase, response);
}

static void
report_timeout(opensmtpd_ctx_t *ctx)
{
	warnx("report_timeout");
}


/* filtering */

static void
filter_connect(opensmtpd_ctx_t *ctx, const char *rdns, const char *arg)
{
	warnx("filter_connect: rdns=%s, arg=%s", rdns, arg);
	proceed(ctx);
}

static void
filter_helo(opensmtpd_ctx_t *ctx, const char *arg)
{
	warnx("filter_helo: arg=%s", arg);
	proceed(ctx);
}

static void
filter_ehlo(opensmtpd_ctx_t *ctx, const char *arg)
{
	warnx("filter_ehlo: arg=%s", arg);
	proceed(ctx);
}

static void
filter_starttls(opensmtpd_ctx_t *ctx, const char *arg)
{
	warnx("filter_starttls: arg=%s", arg);
	proceed(ctx);
}

static void
filter_auth(opensmtpd_ctx_t *ctx, const char *arg)
{
	warnx("filter_auth: arg=%s", arg);
	proceed(ctx);
}

static void
filter_mail_from(opensmtpd_ctx_t *ctx, const char *arg)
{
	warnx("filter_mail_from: arg=%s", arg);
	proceed(ctx);
}

static void
filter_rcpt_to(opensmtpd_ctx_t *ctx, const char *arg)
{
	warnx("filter_rcpt_to: arg=%s", arg);
	proceed(ctx);
}

static void
filter_data(opensmtpd_ctx_t *ctx, const char *arg)
{
	warnx("filter_data: arg=%s", arg);
	proceed(ctx);
}

static void
filter_data_line(opensmtpd_ctx_t *ctx, const char *arg)
{
	warnx("filter_data_line: arg=%s", arg);
	dataline(ctx, arg);
}

static void
filter_rset(opensmtpd_ctx_t *ctx, const char *arg)
{
	warnx("filter_rset: arg=%s", arg);
	proceed(ctx);
}

static void
filter_quit(opensmtpd_ctx_t *ctx, const char *arg)
{
	warnx("filter_quit: arg=%s", arg);
	proceed(ctx);
}

static void
filter_noop(opensmtpd_ctx_t *ctx, const char *arg)
{
	warnx("filter_noop: arg=%s", arg);
	proceed(ctx);
}

static void
filter_help(opensmtpd_ctx_t *ctx, const char *arg)
{
	warnx("filter_help: arg=%s", arg);
	proceed(ctx);
}

static void
filter_wiz(opensmtpd_ctx_t *ctx, const char *arg)
{
	warnx("filter_wiz: arg=%s", arg);
	proceed(ctx);
}

static void
filter_commit(opensmtpd_ctx_t *ctx, const char *arg)
{
	warnx("filter_commit: arg=%s", arg);
	proceed(ctx);
}


int
main(int argc, char *argv[])
{
	opensmtpd_ctx_t *ctx;

	if (argc != 1)
		errx(1, "usage: %s", __progname);

	smtpd_init(&ctx);

	smtpd_on_report(ctx, "smtp-in", "link-connect", report_link_connect);
	smtpd_on_report(ctx, "smtp-in", "link-disconnect", report_link_disconnect);
	smtpd_on_report(ctx, "smtp-in", "link-identify", report_link_identify);
	smtpd_on_report(ctx, "smtp-in", "link-tls", report_link_tls);

	smtpd_on_report(ctx, "smtp-in", "tx-begin", report_tx_begin);
	smtpd_on_report(ctx, "smtp-in", "tx-mail", report_tx_mail);
	smtpd_on_report(ctx, "smtp-in", "tx-rcpt", report_tx_rcpt);
	smtpd_on_report(ctx, "smtp-in", "tx-envelope", report_tx_envelope);
	smtpd_on_report(ctx, "smtp-in", "tx-data", report_tx_data);
	smtpd_on_report(ctx, "smtp-in", "tx-commit", report_tx_commit);
	smtpd_on_report(ctx, "smtp-in", "tx-rollback", report_tx_rollback);

	smtpd_on_report(ctx, "smtp-in", "protocol-client", report_protocol_client);
	smtpd_on_report(ctx, "smtp-in", "protocol-server", report_protocol_server);

	smtpd_on_report(ctx, "smtp-in", "filter-response", report_filter_response);
	smtpd_on_report(ctx, "smtp-in", "timeout", report_timeout);

	smtpd_on_filter(ctx, "smtp-in", "connect", filter_connect);
	smtpd_on_filter(ctx, "smtp-in", "helo", filter_helo);
	smtpd_on_filter(ctx, "smtp-in", "ehlo", filter_ehlo);
	smtpd_on_filter(ctx, "smtp-in", "starttls", filter_starttls);
	smtpd_on_filter(ctx, "smtp-in", "auth", filter_auth);
	smtpd_on_filter(ctx, "smtp-in", "mail-from", filter_mail_from);
	smtpd_on_filter(ctx, "smtp-in", "rcpt-to", filter_rcpt_to);
	smtpd_on_filter(ctx, "smtp-in", "data", filter_data);
	smtpd_on_filter(ctx, "smtp-in", "rset", filter_rset);
	smtpd_on_filter(ctx, "smtp-in", "quit", filter_quit);
	smtpd_on_filter(ctx, "smtp-in", "noop", filter_noop);
	smtpd_on_filter(ctx, "smtp-in", "help", filter_help);
	smtpd_on_filter(ctx, "smtp-in", "wiz", filter_wiz);
	smtpd_on_filter(ctx, "smtp-in", "commit", filter_commit);

	smtpd_on_filter(ctx, "smtp-in", "data-line", filter_data_line);

	if (! smtpd_event_loop(ctx))
		return 1;

	return 0;
}
```

and register all hooks:

```sh
laptop$ /tmp/filter-debug
register|report|smtp-in|filter-response
register|report|smtp-in|link-connect
register|report|smtp-in|link-disconnect
register|report|smtp-in|link-identify
register|report|smtp-in|link-tls
register|report|smtp-in|protocol-client
register|report|smtp-in|protocol-server
register|report|smtp-in|timeout
register|report|smtp-in|tx-begin
register|report|smtp-in|tx-commit
register|report|smtp-in|tx-data
register|report|smtp-in|tx-envelope
register|report|smtp-in|tx-mail
register|report|smtp-in|tx-rcpt
register|report|smtp-in|tx-rollback
register|filter|smtp-in|auth
register|filter|smtp-in|commit
register|filter|smtp-in|connect
register|filter|smtp-in|data
register|filter|smtp-in|data-line
register|filter|smtp-in|ehlo
register|filter|smtp-in|helo
register|filter|smtp-in|help
register|filter|smtp-in|mail-from
register|filter|smtp-in|noop
register|filter|smtp-in|quit
register|filter|smtp-in|rcpt-to
register|filter|smtp-in|rset
register|filter|smtp-in|starttls
register|filter|smtp-in|wiz
register|ready
^C
laptop$
```

In OpenSMTPD, with the following config:
```
[...]
filter foobar proc-exec "/tmp/filter-debug"

listen on socket filter foobar
```

it will result in the following output:

```
[...]
86c0820a13ffee17 smtp connected address=local host=laptop.home
<dynproc:00000001>: filter-debug: report_link_connect: rdns=laptop.home, fcrdns=pass, src=local:0, dest=local:0
<dynproc:00000001>: filter-debug: filter_connect: rdns=laptop.home, arg=local
<dynproc:00000001>: filter-debug: report_filter_response: phase=connected, response=proceed
<dynproc:00000001>: filter-debug: report_protocol_server: line=220 laptop.home ESMTP OpenSMTPD
<dynproc:00000001>: filter-debug: report_protocol_client: line=EHLO localhost
<dynproc:00000001>: filter-debug: filter_ehlo: arg=localhost
<dynproc:00000001>: filter-debug: report_filter_response: phase=ehlo, response=proceed
<dynproc:00000001>: filter-debug: report_link_identify: helo_name=localhost
<dynproc:00000001>: filter-debug: report_protocol_server: line=250-laptop.home Hello localhost [local], pleased to meet you
<dynproc:00000001>: filter-debug: report_protocol_server: line=250-8BITMIME
<dynproc:00000001>: filter-debug: report_protocol_server: line=250-ENHANCEDSTATUSCODES
<dynproc:00000001>: filter-debug: report_protocol_server: line=250-SIZE 36700160
<dynproc:00000001>: filter-debug: report_protocol_server: line=250-DSN
<dynproc:00000001>: filter-debug: report_protocol_server: line=250 HELP
<dynproc:00000001>: filter-debug: report_protocol_client: line=MAIL FROM:<gilles@laptop.home>
<dynproc:00000001>: filter-debug: filter_mail_from: arg=gilles@laptop.home
<dynproc:00000001>: filter-debug: report_filter_response: phase=mail-from, response=proceed
<dynproc:00000001>: filter-debug: report_tx_begin: msgid=2b8bbe3b
<dynproc:00000001>: filter-debug: report_tx_mail: msgid=2b8bbe3b, mail_from=<gilles@laptop.home>, result=ok
<dynproc:00000001>: filter-debug: report_protocol_server: line=250 2.0.0: Ok
<dynproc:00000001>: filter-debug: report_protocol_client: line=RCPT TO:<gilles@laptop.home>
<dynproc:00000001>: filter-debug: filter_rcpt_to: arg=gilles@laptop.home
<dynproc:00000001>: filter-debug: report_filter_response: phase=rcpt-to, response=proceed
<dynproc:00000001>: filter-debug: report_tx_envelope: msgid=2b8bbe3b, evpid=2b8bbe3b60567a11
<dynproc:00000001>: filter-debug: report_tx_rcpt: msgid=2b8bbe3b, mail_from=<gilles@laptop.home>, result=ok
<dynproc:00000001>: filter-debug: report_protocol_server: line=250 2.1.5 Destination address valid: Recipient ok
<dynproc:00000001>: filter-debug: report_protocol_client: line=DATA
<dynproc:00000001>: filter-debug: filter_data: arg=
<dynproc:00000001>: filter-debug: report_filter_response: phase=data, response=proceed
<dynproc:00000001>: filter-debug: filter_data_line: arg=Received: from localhost (laptop.home [local])
<dynproc:00000001>: filter-debug: filter_data_line: arg=        by laptop.home (OpenSMTPD) with ESMTPA id 2b8bbe3b
<dynproc:00000001>: filter-debug: filter_data_line: arg=        for <gilles@laptop.home>;
<dynproc:00000001>: filter-debug: filter_data_line: arg=        Sun, 30 Jun 2019 12:01:55 +0200 (CEST)
<dynproc:00000001>: filter-debug: report_tx_data: msgid=2b8bbe3b, msg_size=10034375646789
<dynproc:00000001>: filter-debug: report_protocol_server: line=354 Enter mail, end with "." on a line by itself
<dynproc:00000001>: filter-debug: filter_data_line: arg=From:  <gilles@laptop.home>
<dynproc:00000001>: filter-debug: filter_data_line: arg=Date: Sun, 30 Jun 2019 12:01:55 +0200 (CEST)
<dynproc:00000001>: filter-debug: filter_data_line: arg=To: gilles
<dynproc:00000001>: filter-debug: filter_data_line: arg=
<dynproc:00000001>: filter-debug: filter_data_line: arg=test
<dynproc:00000001>: filter-debug: filter_data_line: arg=.
<dynproc:00000001>: filter-debug: report_protocol_client: line=.
<dynproc:00000001>: filter-debug: filter_commit: arg=
<dynproc:00000001>: filter-debug: report_filter_response: phase=commit, response=proceed
<dynproc:00000001>: filter-debug: report_tx_commit: msgid=2b8bbe3b
<dynproc:00000001>: filter-debug: report_protocol_server: line=250 2.0.0: 2b8bbe3b Message accepted for delivery
<dynproc:00000001>: filter-debug: report_protocol_client: line=QUIT
<dynproc:00000001>: filter-debug: filter_quit: arg=
[...]
```


filter-rspamd in progress
----
I had written a filter-rspamd for OpenSMTPD in python already,
however it was a proof-of-concept more than a real filter and I didn't make it really robust.

With `libopensmtpd` at hands,
I decided to write a native C filter-rspamd that I would officially maintain,
one that would be robust enough to be used in production.

I had most of the logic done in half an hour,
unfortunately I ran out of time for this "free" week and didn't complete the filter.

Basically as it is right now, the filter will build everything needed for the call to `rspamd`,
but I didn't write the http request.

I will surely finish that during my next "free" week.


What next ?
--
Well, see you in a month for the July report.

Hopefully, by then I will have finished my filter-rspamd, I will have released an initial version of FION,
and I will have released an initial version of libopensmtpd.

When I'm done with FION and have decided what to to with BPG,
I will have a look at some of my other private repositories to see what can be released :-)

If you like my work, [support me on patreon](https://patreon.com/gilles) !

--- 
Comments: [https://github.com/poolpOrg/poolp.org/discussions/102](https://github.com/poolpOrg/poolp.org/discussions/102)

---
title: "September 2019 report: Jules, OpenSMTPD 6.6.0 upcoming release and related things"
date: 2019-09-21 07:08:00 +0200
category: opensource
authors:
 - Gilles Chehade
---
    TL;DR:
    - I started writing this post a week ago but got interrupted by a baby, Jules
    - Spent MANY hours on writing OpenSMTPD-related articles
    - Enabled continuous integration in the OpenSMTPD portable repository
    - Managed to get rid of all the blocking issues for OpenSMTPD 6.6.0 release
    - Added some features and fixed a crash in filter-rspamd


Shout outs to my patrons !
--
As usual,
a **huge** thanks goes to the people sponsoring me on [patreon](https://www.patreon.com/gilles) or [github](https://github.com/sponsors/poolpOrg), the work in this post was made possible by my [sponsorship](/sponsorship/).

Welcome Jules, `fork()` completed
--
In [my last report from August](/posts/2019-08-25/august-2019-report-fion-plakar-and-opensmtpd/),
I concluded not knowing if this report would be published on time.

The expected date for bringing our little monkey home was the 2nd of October,
but while I started writing this article the 21st of September,
he decided not to stick to the plan.
<center>
    <img src="/images/2019-09-21-baby2.jpg">
</center>

This is why I'm publishing today an article dated from the 21st,
which is when I started writing it ;-)

So my biggest completed project for this September report was the little `Jules`,
born Sunday the 22nd of September 2019 at 08:03 AM in the city of Nantes.

Hopefully a future OpenBSD hacker:
<center>
    <img src="/images/2019-09-21-baby.jpeg">
</center>

Though he can be anything,
including a unicorn if he wants.
He cute af.
The cutest.


Wrote two articles for the community
--
The first one,
[You should not run your mail server because mail is hard](https://poolp.org/posts/2019-08-30/you-should-not-run-your-mail-server-because-mail-is-hard/) was intended to be an MTA-agnostic article,
debunking some of the common claims that mail is hard.

The second one,
[Setting up a mail server with OpenSMTPD, Dovecot and Rspamd](https://poolp.org/posts/2019-09-14/setting-up-a-mail-server-with-opensmtpd-dovecot-and-rspamd/) was intended both as a follow up to the previous article,
to back the claims with a technical setup,
but also as a mean to promote OpenSMTPD **AND** answer the "there's not many OpenSMTPD tutorials out there" request.

The first article took me a couple hours to write,
whereas the second one took me about ten hours because I wanted to explain everything I did and make it a reference tutorial for OpenSMTPD users.


Deployed CI on our Github mirror repository
--
OpenSMTPD is developed on OpenBSD,
so we're guaranteed that it builds because we benefit from the HYAYFBTB<sup>1</sup> continuous integration (CI) mechanism.

The portable branch is different.
It is mainly maintained by myself and... I only run OpenBSD and OSX daily.
Most of the changes are merges from the OpenBSD branch which aren't challenging to portability,
but occasionally I need to rely on community feedback or spin a server for a specific distribution.

Sometimes,
I mess up and the trivial merge from OpenBSD lacks an include or uses a function that I thought was portable,
until someone reports the build is broken.

Github enabled CI on our account so I immediately set it up on the OpenSMTPD repository.
It will still need improvements but at this point,
every time we commit to the portable branch,
a `bootstrap`, `configure`, `make` and `make install` are done,
and members of the team all get an e-mail if the build is broken.

While at it,
I also added CI to the `filter-rspamd` and `filter-senderscore` repositories.

<sup>1</sup>_Hackers Yell At You For Breaking The Build_


Introduced `junk` action for builtin and proc filters
--
Until now,
builtin and proc filters had only 4 possible actions:
`proceed` to let other filters handle the session,
`rewrite` to man-in-the-middle rewrite parameters to the filter hook,
`reject` to reject the phase with a custom SMTP message,
and `disconnect` to reject AND disconnect the phase with a custom SMTP message.

Most filters used `reject` or `disconnect` as a mean to kill spammers upfront,
with builtin filters such as:
```
filter check_rdns phase connect match !rdns disconnect "550 GO AWAY"
```

But I received a lot of feedback from people scared of rejecting legitimate mails by being too harsh,
and asking if there was a way to junk the messages rather than throw them away.

I introduced the `junk` action which flags a session or transaction so messages can proceed,
but will get an `X-Spam: yes` header prepended.
The following builtin filter would match the same session as above but,
instead of killing the session,
all messages from the session would be marked as spam:
```
filter check_rdns phase connect match !rdns junk
```

With this new action,
people can actually build confidence for a while before going stricter.

As a bonus,
this makes it easier to write proc-filters that junk messages.
Before this change,
filters would have to keep state of junked sessions and register a callback for data-line phase,
so they could craft the header and prepend it.
With this change,
they can simply junk the message right away and let OpenSMTPD prepend the header.


Implemented Sender Rewriting Scheme support
--
OpenSMTPD lacked Sender Rewriting Scheme (SRS) support.

Long story short,
the Sender Policy Framework (SPF) was introduced to let a domain control which hosts can send mail on its behalf.
This is very nice and all,
but it doesn't play well with mailing lists and forwarding hosts.

For example,
if `eric@faurot.net` sends mail to `misc@opensmtpd.org`,
the `opensmtpd.org` domain needs to send mail to all subscribers on behalf of `faurot.net`,
but... it is not allowed to do so,
resulting in an SPF check failure.

To work around this,
mailing list software encode the sender address in the protocol so it originates from the mailing list domain,
and allows them to decode back upon replies:

```
bb957901382a5d3f mta delivery evpid=25d3675b29721118 from=<misc+bounces-probe-eric=faurot.net@opensmtpd.org> to=<gilles@poolp.org> rcpt=<-> [...]
```

The `misc+bounces-probe-eric=faurot.net@opensmtpd.org` address resolves to `misc@opensmtpd.org` once you remove the `+` subaddressing and the mailing list software handling that address uses the subaddressing value `bounces-probe-eric=faurot.net` to retrieve the original sender.

So far so good,
but this only works for mailing lists.
The `openbsd.org` domain is essentially a mail forwarder:
sending mail to `eric@openbsd.org` really forwards to `eric@faurot.net` and sending mail to `gilles@openbsd.org`really forwards to `gilles@poolp.org`.
In this case,
no mailing list software encodes and decodes addresses,
and this creates problems.

If `eric@faurot.net` sends mail to `gilles@openbsd.org`,
the `openbsd.org` mail exchangers accepts the mail and forwards it to `poolp.org`.
But the mail exchangers at `poolp.org` sees a mail originating from `faurot.net` coming from a mail exchanger that's not the correct one... the SPF check fails again.

SRS comes into play and,
once enabled,
makes sure that any address forwarded is encoded in such a way that it passes SPF correctly and that replies can be traced back to the original sender.
I won't detail this much,
there's a specificiation that can be easily found on a search engine,
but **it's now supported natively in OpenSMTPD**.

If your mail exchanger is intended to forward mails on behalf of other domains,
you can enable SRS using the following configuration:
```
srs key secret_key_to_prevent_people_from_impersonating_me
[...]
action "outgoing_mails" relay srs
```

Resulting in:

<center>
    <img src="/images/2019-09-21-srs.png">
</center>

Plain and simple.

Lots of minor stuff here and there
--
Tons of minor stuff were committed here and there,
ranging from new reporting events for developers,
cleanups,
stricter checks,
documentation.

I won't list everything we did because you can [check the history](https://github.com/openbsd/src/commits/master/usr.sbin/smtpd).



OpenSMTPD 6.6.0
--
OpenSMTPD 6.6.0 is now "feature-ready" and the only things we'll commit until the release are bug fixes if we find any.
The upcoming weeks will be stabilization only,
though... the last weeks were also stabilization mainly ;-)

If you hit issues with the portable branch,
don't worry,
we tag the release when the OpenBSD version is stable and keep working on improving the portable version after,
giving it its own tag.
If you experience issues with portable at the time we tag 6.6.0,
it doesn't mean the issues will not be fixed before the portable version is released.


Various improvements to `filter-rspamd`
--
With the help of Reio Remma [@whataboutpereira](https://github.com/whataboutpereira),
`filter-rspamd` got a lot of new features.

First of all,
I fixed a concurrency-related crash experienced by two OpenBSD hackers.
That was quite a mandatory feature. not crashing.

Then,
I added an X-Spam-Symbols header as we needed it to provide the same level of informations as the `rspamc` client,
which is the alternate way of integrating [Rspamd](https://rspamd.com) with OpenSMTPD.

Reio did a bit of cleanup,
taught the filter how to notify Rspamd about the MX name,
how to reuse Rspamd-provided SMTP messages in the reply to SMTP sessions,
how to add / remove headers based on the Rspamd configuration.
He basically made it closer to the Rspamd milter that's available to other MTA.


What next ?
--
I haven't slept much this week so my mind is a bit blurry about what's on my roadmap,
but my goal is now to focus on the portable branch to improve it.

When my next report is due,
OpenSMTPD 6.6.0 may or may not have been released based on OpenBSD's release plan.

Depending on what happens,
either I'll spend the month working on the release or I'll start working on the next big works,
namely reworking the table layer to work similarly to filters,
allowing tables to be written in any language.

I may also be focusing on diapers.

---- 
Comments: [https://github.com/poolpOrg/poolp.org/discussions/107](https://github.com/poolpOrg/poolp.org/discussions/107)

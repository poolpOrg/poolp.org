---
title: "OpenSMTPD relay maps & new url syntax"
date: 2012-05-14 09:55:52
category: OpenSMTPD
authors:
 - Gilles Chehade
---

When we first started working on OpenSMTPD, we planned for future features but we did not necessarily start integrating them right away as the goal was to bootstrap the project first with basic features.

Amongst these features was the ability to use mappings for outgoing MX servers. Currently, OpenSMTPD knows two ways of going out:

Using the DNS system to find MX records: accept for [...] relay

Using the provided mail exchanger: accept for [...] relay via "my-mx.poolp.org" [...]

The first method is straightforward, nothing can really be tweaked about it without tweaking the DNS system itself.

The second method is much more flexible as it allows providing a port, a transport method (plaintext ? starttls ? smtps ?), a credentials map for authentication to remote relays, a certificate, a SMTP-level FROM overriding, and possibly more as time passes.

In some situations, you can end up with a rule like: accept for all relay via "my-mx.poolp.org" tls port 25 certificate "foobar" auth "mycredsmap" as "foo@bar.org"

Not too complicated, but not too nice either. It becomes annoying when we start considering implementation of mappings at the relay level:

map "relaymap" source plain "/etc/mail/relaymap"

accept for all relay via map "relaymap" tls port 25 certificate "foobar" auth "mycredsmap" as "foo@bar.org"

How do you specifiy different ports, different ssl options and enable/disable auth on a specific MX ?

Well you can't because tls, port and auth is tied to the rule and not the MX referenced by the rule...

So I came up with a prototype for a new syntax:

accept for all relay via "[schema://]host[:port]"

Where schema can be smtp://, smtps://, tls://, ssl://, smtps+auth://, tls+auth:// or ssl+auth://.

As usual we default to the sanest behaviour so if you specify (or use the default) smtp://, like:

accept for all relay via "mx.poolp.org"

OpenSMTPD will attempt to use STARTTLS if possible before falling back to a plaintext session; whereas:

accept for all relay via "tls://mx.poolp.org"

OpenSMTPD will make it mandatory to use a STARTTLS session, refusing to deliver the message otherwise.

Since the MX options are now tied to the MX itself, it becomes possible to store them in a map:

% cat /etc/mail/mxmap.txt ssl://mx1.poolp.org ssl://mx2.poolp.org ssl://mx3.poolp.org

% cat /etc/mail/smtpd.conf map "mxmap" source plain "/etc/mail/mxmap.txt"

accept for all relay via map "mxmap"

Of course, this will become more interesting when the implementation for relay maps is done and allows looking policies (round-robin, random, ratio, ...) and that you can use them from other backends like SQL or db, as this will allow changing exchangers at runtime, a feature that offers plennnnnnty of possibilities :-)

Anyways, I have it mostly working on my sandbox, it still needs a couple hours of work I think, I'll get to it by this week-end if time permits.

Stay tuned !

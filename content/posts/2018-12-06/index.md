---
title: "OpenSMTPD proc filters & fc-rDNS"
date: 2018-12-06 21:31:00
category: OpenSMTPD
authors:
 - "gilles"
categories:
 - technology
---

{{< tldr >}}
    I *FINALFUCKINGLY* commited proc filters support allowing full filtering in OpenSMTPD.
    eric@ implemented fc-rDNS lookups.
{{< /tldr >}}


fc-rDNS
--
fc-rDNS,
or forward-confirmed reverse DNS,
consists in performing a reverse DNS lookup to determine the hostname associated to an IP address...
then performing a DNS lookup on that hostname to check if it resolves back to the IP address.

On my request,
eric@ implemented fc-rDNS lookups in our SMTP engine,
causing OpenSMTPD to perform the double lookup upon clients connections.

Right now,
the fc-rDNS result is only passed to the reporting API but the idea is to allow using it in a builtin filter,
something along the lines of `filter smtp-in connect check-fcrdns disconnect "550 go away punk"`.

Note that this is _particularly_ efficient at cutting spam,
it essentially means that a spammer must control both the DNS and reverse DNS zone to pass the test,
killing the bulk of zombified spam proxies living on infected home computers.

There is still some refining to do and the plugging of a builtin-filter,
but the code is here and -current users will be able to test it soon


tx-mail and tx-rcpt events
--
I have added two new reporting events for smtp-in and smtp-out: `tx-mail` and `tx-rcpt`.

They are generated whenever `MAIL FROM` or `RCPT TO` have received a result from the server:

```
report|1|1544130229|smtp-in|tx-mail|0f3004c08c82d33e|fc08ce7d|<owner-hackers+M85937=gilles=poolp.org@openbsd.org>|ok
report|1|1544130229|smtp-in|tx-rcpt|0f3004c08c82d33e|fc08ce7d|<gilles@poolp.org>|ok
```

A filter may listen for these events to determine which sender and recipients have been accepted in a transaction,
something that would otherwise require keeping track of protocol-client and protocol-server events.


proc filtering finally in
--
I have committed full proc filtering support today,
allowing a standalone filter to perform all kind of filtering on every single phase of an SMTP session.

This means that with the code in -current,
it is now possible to write a filter that talks to rspamd or that computes dkim signatures,
something that was not doable until today without resorting to tricky setups.

The code is still a work in progress,
we have things to clean up, improve and there are some known minor issues that we are working on.
We will likely hit corner cases that will cause new issues...
but this is a fully working implementation, the real deal, not a PoC.
Consider it as "in a process of stabilization to be production ready by April".

When we have stabilized the API,
I'll take time to write about how it works internally,
for now let's just celebrate because I FINALLY FUCKING COMMITED FILTERS.


python bridge, proof of concept
--
As I wrote in a previous post,
proc filters are programs reading on stdin and writing to stdout,
making it possible write them in any language.

I put my code where my mouth is:

```python
import opensmtpd

def link_connect(timestamp, session_id, args):
    rdns, fcrdns, laddr, raddr = args

def link_disconnect(timestamp, session_id, args):
    _ = args

def tx_begin(timestamp, session_id, args):
    tx_id = args[0]

def tx_mail(timestamp, session_id, args):
    tx_id, address, status = args

def tx_rcpt(timestamp, session_id, args):
    tx_id, address, status = args

def tx_envelope(timestamp, session_id, args):
    tx_id, evp_id = args
    
def tx_rollback(timestamp, session_id, args):
    tx_id = args[0]

def tx_commit(timestamp, session_id, args):
    tx_id, nbytes = args

def protocol_client(timestamp, session_id, args):
    line = args[0]

if __name__ == "__main__"
    o = opensmtpd.SMTP_IN()

    o.on_report('link-connect', link_connect)
    o.on_report('link-disconnect', link_disconnect)

    o.on_report('tx-begin', tx_begin)
    o.on_report('tx-mail', tx_mail)
    o.on_report('tx-rcpt', tx_rcpt)
    o.on_report('tx-envelope', tx_envelope)
    o.on_report('tx-commit', tx_commit)
    o.on_report('tx-rollback', tx_rollback)

    o.on_report('protocol-client', protocol_client)
    o.on_report('protocol-server', protocol_server)

    o.run()
```

This only covers the callback API for "report" events,
but it does so thanks to an opensmtpd package that consists of less than 100 lines of code.

I will be implementing the `on_filter` callback API probably next week,
so I can start writing the filters I need in python.

If you want to discuss how to write an interface for the language of your choice,
feel free to jump to our IRC channel, #opensmtpd @ freenode ;-)



What next ?
--
Essentially code cleanup and simplification,
API stabilization and writing filters to spot improvements required in the API.

Stay tuned !


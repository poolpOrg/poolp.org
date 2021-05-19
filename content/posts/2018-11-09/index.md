---
title: "OpenSMTPD reporting update"
date: 2018-11-09 08:27:00
category: OpenSMTPD
authors:
 - Gilles Chehade
---

    TL;DR:
    The reporting mechanism has been described shortly in my previous article about both reporting and filters.
    Let's focus a bit more on the reporting bits this time.
    The format is improving further and has extended to outgoing trafic reporting.


Reporting
--
In [previous article](https://poolp.org/posts/2018-11-03/opensmtpd-released-and-upcoming-filters-preview/),
I described the events reporting mechanism that has been introduced in the development branch of OpenSMTPD.

To sum it up,
you could now write an event processor as simple as a shell script reading its stdin on a loop:

```
$ cat  /tmp/reporting.sh
#! /bin/sh
#

while read line; do
        echo $line >> /tmp/reporting.log
done
```

and configure your OpenSMTPD so it would report all incoming SMTP events:

```
$ grep report /etc/mail/smtpd.conf                                                                                                                                                                                   
proc reporting "/tmp/reporting.sh"
report smtp on reporting
```

which would then produce an events report log in /tmp/reporting.log containing entries similar to these:

```
report|smtp-in|link-connect|1541271219|3189ac6874354895|localhost|127.0.0.1:41564|127.0.0.1:25
report|smtp-in|protocol-server|1541271219|3189ac6874354895|220 poolp.org ESMTP OpenSMTPD
report|smtp-in|protocol-client|1541271222|3189ac6874354895|helo localhost
report|smtp-in|protocol-server|1541271222|3189ac6874354895|250 poolp.org Hello localhost [127.0.0.1], pleased to meet you
report|smtp-in|link-disconnect|1541271224|3189ac6874354895
```


Improvements on the reports format
--
I made several improvements to the format described in the previous article.

The first improvement is that there is now a version embedded in each report.
This allows event processors to be able to check if they know how to parse an event report,
it allows them to easily support backward-compatible versions should we make changes to the format of an event report,
but most importantly it allows you to just store these events somewhere then have tools post-process them months later without ambiguity with regard to the format of entries,
even if there were OpenSMTPD updates in between.

The second improvement is that the timestamp which was appearing after the event type was moved in front of it.
This doesn't seem like an improvement but it eases reading and allows me to simplify some of the code :-)

The third improvement comes from adding some new events and adding information to some existing events.
For instance,
OpenSMTPD reported these transaction events:

```
report|smtp-in|tx-begin|1541271225|3189ac6874354895
report|smtp-in|tx-commit|1541271225|3189ac6874354895
report|smtp-in|tx-rollback|1541271225|3189ac6874354895
```

But reading from these,
you could only obtain the session identifier,
there was no way to find out the transaction identifier,
how many envelopes were generated in the transaction or even the size of the message.

To solve these issues,
the format of the events above has been extended so it would contain the transation identifier (aka. msgid),
a `tx-envelope` event was introduced to report the generation of a new envelope in the transaction along with its envelope identifier (aka. evpid),
and finally the size of the DATA part is reported on a `tx-commit` event.

Last but not least,
the DATA part begins with a `DATA` command issued by the client and ends with a single  `.` on a line by itself.
Despite the single `.` being sent within the DATA phase,
it is not really part of the DATA itself and must be considered as a _commit request_.

This doesn't seem like much but the devil is in the details.
Not reporting that commit request as a `protocol-client` command means that we go straight from `DATA` command to a `tx-commit` event,
without allowing a filter to actually refuse the commit request.
If we generate this commit request event,
a filter may decide that it wants to reject it which will then produce a `tx-rollback` that was not possible before.

A pattern emerges that `tx-*` events should appear in between `protocol-*` events otherwise they cannot be filtered.

Here is a sample curated event report log from my own server as of today:
```
$ cat /tmp/reporting.log     
report|1|1541750432|smtp-in|link-connect|c73c0aff0dfb6250|poolp.org|local:0|local:0
report|1|1541750432|smtp-in|protocol-server|c73c0aff0dfb6250|220 poolp.org ESMTP OpenSMTPD
report|1|1541750432|smtp-in|protocol-client|c73c0aff0dfb6250|EHLO localhost
report|1|1541750432|smtp-in|protocol-server|c73c0aff0dfb6250|250-poolp.org Hello localhost [local], pleased to meet you
report|1|1541750432|smtp-in|protocol-server|c73c0aff0dfb6250|250-8BITMIME
report|1|1541750432|smtp-in|protocol-server|c73c0aff0dfb6250|250-ENHANCEDSTATUSCODES
report|1|1541750432|smtp-in|protocol-server|c73c0aff0dfb6250|250-SIZE 36700160
report|1|1541750432|smtp-in|protocol-server|c73c0aff0dfb6250|250 HELP
report|1|1541750432|smtp-in|protocol-client|c73c0aff0dfb6250|MAIL FROM:<gilles@poolp.org>
report|1|1541750432|smtp-in|tx-begin|c73c0aff0dfb6250|f84306b3
report|1|1541750432|smtp-in|protocol-server|c73c0aff0dfb6250|250 2.0.0: Ok
report|1|1541750432|smtp-in|protocol-client|c73c0aff0dfb6250|RCPT TO:<gilles@poolp.org>
report|1|1541750432|smtp-in|tx-envelope|c73c0aff0dfb6250|f84306b3|f84306b34f388082
report|1|1541750432|smtp-in|protocol-server|c73c0aff0dfb6250|250 2.1.5 Destination address valid: Recipient ok
report|1|1541750432|smtp-in|protocol-client|c73c0aff0dfb6250|DATA
report|1|1541750432|smtp-in|protocol-server|c73c0aff0dfb6250|354 Enter mail, end with "." on a line by itself
report|1|1541750432|smtp-in|protocol-client|c73c0aff0dfb6250|.
report|1|1541750432|smtp-in|tx-commit|c73c0aff0dfb6250|f84306b3|301
report|1|1541750432|smtp-in|protocol-server|c73c0aff0dfb6250|250 2.0.0: f84306b3 Message accepted for delivery
report|1|1541750432|smtp-in|protocol-client|c73c0aff0dfb6250|QUIT
report|1|1541750432|smtp-in|protocol-server|c73c0aff0dfb6250|221 2.0.0: Bye
report|1|1541750432|smtp-in|link-disconnect|c73c0aff0dfb6250
```


Introducing smtp-out
--
Obviously my plan is to be able to report and create dashboard for ALL trafic,
not just incoming.

I have worked on generating reports for smtp-out and I actually have something working in a branch,
which I intend to commit next week.

The format is _identical_ with the sole difference that `smtp-in` is replaced with `smtp-out`,
the event types are the same,
the parameters are the same,
you just need to get your head around the fact that `protocol-client` is your peer when in `smtp-in`,
whereas `protocol-server` is your peer when in `smtp-out`:

```
report|1|1541750707|smtp-out|link-connect|c73c0b206ad6c41c||:0|64.233.167.26:25
report|1|1541750707|smtp-out|protocol-server|c73c0b206ad6c41c|220 mx.google.com ESMTP k132-v6si807155wma.16 - gsmtp
report|1|1541750707|smtp-out|protocol-client|c73c0b206ad6c41c|EHLO out.mailbrix.mx
report|1|1541750707|smtp-out|protocol-server|c73c0b206ad6c41c|250-mx.google.com at your service, [212.83.129.132]
report|1|1541750707|smtp-out|protocol-server|c73c0b206ad6c41c|250-SIZE 157286400
report|1|1541750707|smtp-out|protocol-server|c73c0b206ad6c41c|250-8BITMIME
report|1|1541750707|smtp-out|protocol-server|c73c0b206ad6c41c|250-STARTTLS
report|1|1541750707|smtp-out|protocol-server|c73c0b206ad6c41c|250-ENHANCEDSTATUSCODES
report|1|1541750707|smtp-out|protocol-server|c73c0b206ad6c41c|250-PIPELINING
report|1|1541750707|smtp-out|protocol-server|c73c0b206ad6c41c|250-CHUNKING
report|1|1541750707|smtp-out|protocol-server|c73c0b206ad6c41c|250 SMTPUTF8
report|1|1541750707|smtp-out|protocol-client|c73c0b206ad6c41c|STARTTLS
report|1|1541750707|smtp-out|protocol-server|c73c0b206ad6c41c|220 2.0.0 Ready to start TLS
report|1|1541750707|smtp-out|link-tls|c73c0b206ad6c41c|version=TLSv1.2, cipher=ECDHE-RSA-CHACHA20-POLY1305, bits=256
report|1|1541750708|smtp-out|protocol-client|c73c0b206ad6c41c|EHLO out.mailbrix.mx
report|1|1541750708|smtp-out|protocol-server|c73c0b206ad6c41c|250-mx.google.com at your service, [212.83.129.132]
report|1|1541750708|smtp-out|protocol-server|c73c0b206ad6c41c|250-SIZE 157286400
report|1|1541750708|smtp-out|protocol-server|c73c0b206ad6c41c|250-8BITMIME
report|1|1541750708|smtp-out|protocol-server|c73c0b206ad6c41c|250-ENHANCEDSTATUSCODES
report|1|1541750708|smtp-out|protocol-server|c73c0b206ad6c41c|250-PIPELINING
report|1|1541750708|smtp-out|protocol-server|c73c0b206ad6c41c|250-CHUNKING
report|1|1541750708|smtp-out|protocol-server|c73c0b206ad6c41c|250 SMTPUTF8
report|1|1541750708|smtp-out|protocol-client|c73c0b206ad6c41c|MAIL FROM:<gilles@poolp.org>
report|1|1541750708|smtp-out|protocol-server|c73c0b206ad6c41c|250 2.1.0 OK k132-v6si807155wma.16 - gsmtp
report|1|1541750708|smtp-out|tx-begin|c73c0b206ad6c41c|8ea46ad1
report|1|1541750708|smtp-out|protocol-client|c73c0b206ad6c41c|RCPT TO:<gilles.chehade@gmail.com>
report|1|1541750708|smtp-out|protocol-server|c73c0b206ad6c41c|250 2.1.5 OK k132-v6si807155wma.16 - gsmtp
report|1|1541750708|smtp-out|tx-envelope|c73c0b206ad6c41c|8ea46ad1|8ea46ad1ccae4934
report|1|1541750708|smtp-out|protocol-client|c73c0b206ad6c41c|DATA
report|1|1541750708|smtp-out|protocol-server|c73c0b206ad6c41c|354 Go ahead k132-v6si807155wma.16 - gsmtp
report|1|1541750708|smtp-out|protocol-client|c73c0b206ad6c41c|.
report|1|1541750708|smtp-out|protocol-server|c73c0b206ad6c41c|250 2.0.0 OK 1541750708 k132-v6si807155wma.16 - gsmtp
report|1|1541750708|smtp-out|tx-commit|c73c0b206ad6c41c|8ea46ad1|818
report|1|1541750718|smtp-out|protocol-client|c73c0b206ad6c41c|QUIT
report|1|1541750718|smtp-out|protocol-server|c73c0b206ad6c41c|221 2.0.0 closing connection k132-v6si807155wma.16 - gsmtp
report|1|1541750718|smtp-out|link-disconnect|c73c0b206ad6c41c
```

Because not everyone needs reporting and not everyone needs _both_ incoming and outgoing reporting,
I have added the smtp-in and smtp-out keywords to the grammar so that you can:

```
$ grep report /etc/mail/smtpd.conf                                                                                                                                                                                   
proc reporting "/tmp/reporting.sh"
report smtp-in on reporting
report smtp-out on reporting
```

I'll probably make `report smtp on` a shortcut for both `smtp-in` and `smtp-out`.

There is still work to be done on the smtp-out path because the SMTP engine is more complex than for the smtp-in path.
For instance,
it is currently not possible to have any of the transaction events generated between the `protocol-client` and `protocol-server` events due to how the state machine is written.
Not really as much of a big deal as for smtp-in since smtp-out isn't filtered and the order issues are less annoying,
but to be really clean and consistent,
the smtp-in and smtp-out reports should be very parallel in terms of order events.
I should be able to look at the smtp-out reports from my laptop and the smtp-in reports from my server and see them appear in the same order.

Finally,
there is also an rDNS lookup that needs to be added so the report is identical to smtp-in,
and we should be fine.


What's so good about this ?
--
Reporting is not JUST about being able to write dashboards,
it is not just about being able to generate state for filters eithers.

Generating event reports logs that can be parsed by external tools open the way for many side applications ranging from tools to replay sessions when tracking issues,
tools to analyze behavior of peers and feedback into `pf` or OpenSMTPD tables,
and more interestingly for people who will developer filters...
it brings the ability to write and test a filter without a running OpenSMTPD instance,
piping the event log directly into the filter.

To be very honest,
I'm personally more excited by this new feature than the filters feature which might be more visible but would be far less powerful without the event logs.


What next ?
--
More changes should happen to the format of entries in the next few weeks and months,
this is a moving target as I wrote in previous article.

Builtin filters already require some of these lines to provide more informations and this is being worked on.

My next focus is the filtering of the DATA phase which is the requirement for us to provide support for dkim and antispam stuff without the need of proxies and reenqueuing.
Work has already started but I will probably not commit any code related to this before the end of November.


--- 
Comments: [https://github.com/poolpOrg/poolp.org/issues/52](https://github.com/poolpOrg/poolp.org/issues/52)

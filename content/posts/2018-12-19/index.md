---
title: "more on OpenSMTPD filters"
date: 2018-12-19 18:21:00
category: OpenSMTPD
author: Gilles Chehade
---

    TL;DR:
    Not this time, pal/gal, I took hours writing this post, you'll take a few minutes reading it all.
    Oh, and merry X-mas :-*


A bit of short-sighted history
--
The filtering feature has been introduced only recently in OpenSMTPD,
first presented on [this blog a month ago](https://poolp.org/posts/2018-11-03/opensmtpd-released-and-upcoming-filters-preview/).

I had a working proof-of-concept running on my laptop and my plan was to start bringing the code to the OpenBSD tree,
small chunks by small chunks,
through a serie of diffs.

I won't cover the serie in details because it's irrelevant,
but the point is that we went from scratch to a working filter implementation in a few days,
one that we knew would work but that had only ran on a one-interface laptop and was not polished.

In the last few weeks,
as various proof-of-concepts for useful filters were written,
the protocol and API improved and while there's many more improvements to come before the next major release,
it is safe to say that the API now allows full-featured filters.


How does filtering work in OpenSMTPD ?
--
During the lifetime of an SMTP session,
whenever a client sends a command,
the server checks if the command should be accepted or rejected then replies to the client.

First it checks if the command is valid in terms of syntax and context (for example,
`AMIL FORM` is an invalid command,
`MAIL FROM: ><` is a valid command with an invalid parameter,
and you can't send `RCPT TO` before `MAIL FROM`).
Then, it checks if the command is actually allowed by the ruleset.
Sometimes, it may reject a valid command for issues related to lack of ressources,
but let's keep this corner case out.

How does filtering work then ?

Well,
right between the two types of checks,
after the server has checked that the input was valid in terms of syntax and context,
it pauses processing and requests the `lookup` process to take a decision based on the input.
The `lookup` process passes that input to the filters sequentially and if any decides to take an action,
the server will enforce that action instead of resuming its usual check.

When it comes to the `DATA` part,
things are trickier because the server doesn't answer to the individual lines but to the whole message.
Also,
it is expected that the filters can actually make changes to the content of the message.
As such,
there are two kinds of filtering in OpenSMTPD: protocol filtering and data filtering.
The former provides a simple query/reply mechanism to alter the decision taking in the SMTP engine,
the latter provides a simple transformation mechanism consuming an input and producing an output.


Let's get a bit more technical
--
As mentionned in the blog post linked in my introduction,
there are two mechanisms involved: reporting and filtering.

Each of these have their own specific hooks,
triggering at specific phases of the SMTP session,
and allowing the SMTP engine to generate reporting events or filter queries at appropriate times.

The basic idea when I designed the filter API was that we don't want complexity in these hooks.
We do want filters to have as much flexibility as possible,
a filter should be able to reject a session after the `data` phase because the source address used during the `connect` phase didn't have a proper reverse DNS configured,
but this should not come at the cost of having hooks with tons of parameters or huge structures encompassing each and everything.

To achieve this, the reporting and filtering mechanisms were designed with the following philosophy in mind:

The reporting mechanism should generate enough events that it is possible to replicate the state of an SMTP session,
but these events should only contain the information related to the event itself.
A filter that wants to reject a session after the `data` phase because of the reverse DNS doesn't need to receive the reverse DNS during the `data` phase,
instead it needs to receive it during the `connect` phase and keep it in a local state to check it at the `data` phase.
This allowed making the reporting mechanism very simple and not limiting since a filter can essentially hook all reporting phases and have the exact same state as the SMTP session itself.

The filtering mechanism in turn should only take decisions based on the parameters of the phase itself ... and any local state.
A filter registering a hook for the `MAIL FROM` phase will only receive the address of the sender.
If it wants to take decisions based on anything more than that,
then it needs to rely on a local state gathered from the reporting mechanism.

The `data` filtering is trickier because we want to be able to add, change or suppress lines.
As a result,
we can't rely on the number of bytes we sent and the number of bytes we received,
just like we can't rely on a lines count.
Furthermore,
there can be multiple filters processing the data in sequence,
so the first filter may receive 1 line,
generate 4 headers,
the second filter remove some of these headers,
the third filter append a ton of data,
etc...

This is where `eric@` had a very clever idea which is to consider the `DATA` part as a stream.
The SMTP protocol ends `DATA` with a single `.` on a line by itself,
so the lines are streamed to the filters which read them up to the `.` and output a new stream terminate by a `.` itself.
Doing this allows the SMTP engine to stream to filters the DATA up to the client-submitted `.`,
then read back a stream from the filters up to the filter-submitted `.`,
not having to care if the stream was altered.

Sounds like a lot of details ?

You need not worry,
in practice writing a filter is trivial.


A case study: filter-rspamd
--
I wrote many filters to experiment and refine the API,
but here's an interesting one.

Let's have a look at how this whole reporting and filtering mechanisms come into play.
I wrote the filter in python,
it is fairly small but I will not copy paste the whole code because I wrote it as a PoC,
[and I'm ashamed of the quality](https://github.com/poolpOrg/py-opensmtpd-rspamd/blob/master/opensmtpd_rspamd/processor.py).

First of all,
let's define what we want to do with the filter.

We want a filter that will pass a `DATA` part to the [rspamd](https://rspamd.com) daemon,
with several session informations gathered from the connection up to the message itself so it can take a decision,
and then alter the message to insert headers and possibly reject temporarily or permanently the message.
Since I was in a good mood when I wrote the PoC,
I also added DKIM signing but it's out of scope ;-)

Remember that this is a Python example,
because I find it easier to understand for all,
but filters can be written with any language really.

```python
sessions = {}

class Rspamd():
    def __init__(self):
        self.stream = smtp_in()

        self.stream.on_report('link-connect', link_connect, None)
        self.stream.on_report('link-disconnect', link_disconnect, None)
        self.stream.on_report('link-identify', link_identify, None)
        self.stream.on_report('tx-begin', tx_begin, None)
        self.stream.on_report('tx-mail', tx_mail, None)
        self.stream.on_report('tx-rcpt', tx_rcpt, None)
        self.stream.on_report('tx-data', tx_data, None)
        self.stream.on_report('tx-commit', tx_cleanup, None)
        self.stream.on_report('tx-rollback', tx_cleanup, None)

        self.stream.on_filter('commit', filter_commit, None)
        self.stream.on_filter('data-line', filter_data_line, None)
        
    def run(self):
        self.stream.run()
```

Based on this excerpt alone,
without looking at the actual implementation of the callback functions,
here's what you can understand:

The only times when we actually want to mess with a session somehow is when making changes to the data,
which is handled by the `data-line` callback,
and when we want to possibly reject temporarily or permanently a message,
which is handled by the `commit` callback.

All of the `on_report()` calls are used solely to accumulate enough informations so the two filter hooks can work,
and if we look at the implementation of one of these,
you'll realize that all it does is really store a particular bit of information in the local state for a session.

For example,
the purpose of registering for the `link-identify` event is to store the `helo` name the client submitted in the local session state:

```python
def link_identify(ctx, timestamp, session_id, args):
    helo = args[0]

    session = sessions[session_id]
    session.control['Helo'] = helo
```

So that the information can be sent to the `rspamd` daemon when the filter begins sending the message in the `data-line` callback.


Per-listener filtering
--
In the first version of the filter feature I commited,
filters were declared globally in the configuration,
listeners would only enable/disable filtering,
the filters would be applied in sequence:

```
filter smtp-in connect check-fcrdns reject "550 go away you punk"
filter smtp-in connect check-rdns reject "550 go away you punk"

listen on all filter	# not filtered
listen on socket	# filtered
```

This was a first step at plugging filters on and off,
but real use cases rely on being able to plug different sets of filters on different interfaces,
so that you can for example reject senders without a reverse DNS on an interface,
while allowing them on the submission port where they authenticate.

This feature is now supported.


Filter grammar changes
--
The filter declaration grammar I used to begin was easy to start playing right away,
it couldn't cover some of the use-cases I had in mind and planned for,
but now that the plumbing has evolved enough we can move towards what's going to look like the final grammar.

First of all,
let's look how I would plug that `filter-rspamd` in my config:

```
filter rspamd proc-exec "/usr/local/bin/filter-rspamd"

listen on all filter rspamd
```

That's all.

I declared a filter named `rspamd`,
instructed OpenSMTPD that it has to execute a proc filter from `/usr/local/bin/filter-rspamd`,
and instructed the listener that all sessions handled through it should go through the `rspamd` filter.

If I didn't want to use rspamd but a builtin filter to reject sessions without a forward-confirmed rDNS,
I could have used:

```
filter fcrdns builtin connect check-fcrdns reject "550 go away you punk"

listen on all filter fcrdns
```

But what's more interesting is the chaining of filters,
which allows ...
well, chaining filters:

```
filter fcrdns builtin connect check-fcrdns reject "550 go away you punk"
filter rspamd proc-exec "/usr/local/bin/filter-rspamd"
filter nazi_mode chain { fcrdns, rspamd }

listen on all filter nazi_mode
```

This,
combined with the fact that the filters apply per-interface,
allow for very flexible setups that could never be expressed on OpenSMTPD before.


What next ?
--
The code and new grammar is working and committed in a branch,
I intend to do some additional cleanup and code simplification before comitting to the OpenBSD tree hopefully this week.

I will spend the remaining of the release cycle,
until April,
performing code cleanups,
refining the reporting events and protocol,
and maybe committing a few minor features (or maybe I have some other nice major features ready, who knows ;-)

You are _HIGHLY_ encouraged to start playing with filters next week after my commit.
There's currently NO filter available,
you can really be the first to do useful stuff for the community.

Stay tuned !

OH AND MERRY X-MAS BECAUSE I WONT BE POSTING BEFORE THEN ;-)

--- 
Comments: [https://github.com/poolpOrg/poolp.org/issues/54](https://github.com/poolpOrg/poolp.org/issues/54)

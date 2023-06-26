---
title: "OpenSMTPD new config"
date: 2018-04-30 12:00:00
category: OpenSMTPD
authors:
 - Gilles Chehade
categories:
 - technology
---

    TL;DR:
    OpenBSD #p2k18 hackathon took place at Epitech in Nantes.
    I was organizing the hackathon but managed to make progress on OpenSMTPD.
    As mentionned at EuroBSDCon the one-line per rule config format was a design error.
    A new configuration grammar is almost ready and the underlying structures are simplified.
    Refactor removes ~750 lines of code and solves _many_ issues that were side-effects of the design error.
    New features are going to be unlocked thanks to this.


Anatomy of a design error
--
OpenSMTPD started ten years ago out of dissatisfaction with other solutions,
mainly because I considered them way too complex for me not to get things wrong from time to time.

The initial configuration format was very different,
I was inspired by pyr@'s `hoststated`,
which eventually became `relayd`,
and designed my configuration format with blocks enclosed by brackets.

When I first showed OpenSMTPD to pyr@,
he convinced me that PF-like one-line rules would be awesome,
and it was awesome indeed.

It helped us maintain our goal of simple configuration files,
it helped fight feature creeping,
it helped us gain popularity and become a relevant MTA,
it helped us get where we are now 10 years later.

That being said,
I believe this was a design error.
A design error that could not have been predicted until we hit the wall to understand WHY this was an error.
One-line rules are semantically wrong,
they are SMTP wrong,
they are wrong.

One-line rules are making the entire daemon more complex,
preventing some features from being implemented,
making others more complex than they should be,
they no longer serve our goals.

To get to the point: we should move to two-line rules :-)


SMTP is transactional protocol
--
SMTP is a transactional protocol which considers each message to be part of a transaction with one or many recipients.

A message gets assigned a unique transaction identifier when committed to queue:

    250 2.0.0: 8a2e1208 Message accepted for delivery

That identifier (supposedly) guarantees that the message has been written to disk,
that the mail exchanger will take responsibility not to let it vanish,
and that all recipients from the same transaction will share the same identifier.

Similarly,
recipients may result in aliasing or may have ~/.forward files resulting in new recipients,
and since all recipients from the same transaction must share the same identifier,
this implies that all aliases and .forward expansions must be done before accepting a recipient.

If you must remember only one thing from this section,
when the message identifier has been replied to the client,
no new envelope will be generated for this transaction.


The problem with one-line rules
--
OpenSMTPD decides to accept or reject messages based on one-line rules such as:

    accept from any for domain poolp.org deliver to mbox

Which can essentially be split into three units:

- the decision: accept/reject
- the matching: from any for domain poolp.org
- the (default) action: deliver to mbox

To ensure that we meet the requirements of the transactions,
the matching must be performed during the SMTP transaction before we take a decision for the recipient.

Given that the rule is atomic,
that it doesn't have an identifier and that the action is part of it,
the two only ways to make sure we can remember the action to take later on at delivery time is to either:

- save the action in the envelope, which is what we do today
- evaluate the envelope _again_ at delivery

And this this where it gets tricky... both solutions are NOT ok.

The first solution,
which we've been using for a decade,
was to save the action within the envelope and kind of carve it in stone.
This works fine...
however it comes with the downsides that errors fixed in configuration files can't be caught up by envelopes,
that delivery action must be validated way ahead of time during the SMTP transaction which is much trickier,
that the parsing of delivery methods takes place as the \_smtpd user rather than the recipient user,
and that envelope structures that are passed all over OpenSMTPD carry delivery-time informations,
and more, and more, and more.
The code becomes more complex in general,
less safe in some particular places,
and some areas are nightmarish to deal with because they have to deal with completely unrelated code that can't be dealt with later in the code path.

The second solution can't be done.
An envelope may be the result of nested rules,
for example an external client, hitting an alias, hitting a user with a .forward file resolving to a user.
An envelope on disk may no longer match any rule or it may match a completely different rule
If we could ensure that it matched the same rule,
evaluating the ruleset may spawn new envelopes which would violate the transaction.
Trying to imagine how we could work around this leads to more and more and more RFC violations,
incoherent states,
duplicate mails,
etc...

There is simply no way to deal with this with atomic rules,
the matching and the action must be two separate units that are evaluated at two different times,
failure to do so will necessarily imply that you're either using our first solution and all its downsides,
or that you are currently in a world of pain trying to figure out why everything is burning around you.
The minute the action is written to an on-disk envelope,
you have failed.

A proper ruleset must define a set of matching patterns resolving to an action identifier that is carved in stone,
AND a set of named action set that is resolved dynamically at delivery time.

    # this is resolved at delivery time
    action my_action mbox
    
    # this is resolved at SMTP transaction time 
    match from any for local action my_action


Action and Match
--
By splitting these,
we get immediate benefits,
I will only list a few of the improvements this allowed and why.

First of all,
the on-disk envelopes now save the name of the action,
rather than the action,
and resolve the action right before delivery.

The obvious benefit is that if I was wrong and wanted to convert to maildir,
it would be enough to change the previous example to:

    action my_action maildir
    
    match from any for local action my_action

After a restart,
all envelopes would immediately catch up change without any need to rewrite anything.
The same would apply if I got my action wrong:

    action my_action maildir "~/Maildirr"

Which would be unfixable today without going to edit envelopes on-disk manually,
but would only require fixing smtpd.conf and restarting with the new configuration.

The other benefit is that code paths like aliases expansion required going down to the action to write the envelope,
this would involve looking up users,
expanding .forward variables,
making sure that the delivery data is fully ready so that the delivery process just has to pass it to the MDA.
The aliases expansion is already a complex code path by nature,
throwing a ton of additional delivery-time data to process and craft into structures made some parts really hard to work with.

The structures involved would not only contain the envelope data,
they would also contain the delivery data which were passed to all components including the SMTP server, the disk queue, the scheduler, the delivery and finally the mda, when the only processes really needing them were the deivery and mda processes.

Code parsing user-supplied variables,
which could be expanded by the user process right before executing the mda command needed to be parsed during the SMTP transaction within an OpenSMTPD owned process.

I don't know where to end,
the refactor to move to his new grammar essentially simplified every single process within OpenSMTPD,
removed over 750 lines of code from the daemon,
automagically unlocked features that were previously not even doable,
and makes it possible to move forward with other features which were tricky.


New features ?
--
Yes, there are new features which required the grammar change but they are not part of it and will come after.

The only features the new grammar bring is some matching patterns such as:

    match tls [...]             # to match a session using TLS
    match auth [...]            # to match an authenticated session
    match from socket [...]     # to match sessions issued by the local enqueuer
    match helo foobar [...]     # to match sessions that issues HELO foobar


What's the catch ?
--
The code is ready but I haven't finished the man page update which is a pre-requisite to me sending this to OpenBSD developers to get their feedbak on the new grammar.

In addition,
the change of behaviour and grammar is absolutely not backward-compatible and will make the older queue incompatible.

Besides that,
it has 100% benefits and no downside compared to previous grammar,
it is cleaner, safer, saner and allows for a brighter future.

It is currently powering poolp.org and the misc@opensmtpd.org mailing list and shows no sign of regression.

This should hit OpenBSD -current in a few days/weeks and be part of the next OpenBSD (6.4) and OpenSMTPD major release.

---
Comments: [https://github.com/poolpOrg/poolp.org/discussions/93](https://github.com/poolpOrg/poolp.org/discussions/93)

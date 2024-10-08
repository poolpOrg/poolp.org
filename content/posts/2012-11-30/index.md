---
title: "OpenSMTPD: more features, more cleanup, more more"
date: 2012-11-30 22:13:25
category: OpenSMTPD
authors:
 - "gilles"
categories:
 - technology
---

OHAI,

First of all, on a completely unrelated note, I'd like to emphasize that this blog post is being written from my text editor launched by a little shell and pushed to poolp using the Ocean API. I can be web2.0 from console and emacs, and that's definitely worth a mention ;-)

As has become a habit, a lot of work has been poured in OpenSMTPD this week, here is an incomplete summary of the most notable changes. For the complete changelog feel free to check the commit log on our Github mirror.

A snapshot should be published tomorrow but meanwhile enjoy the reading, some paragraphs have guest-starred eric ;-)

Simplify parse.y grammar

I have spent many hours working on a simplification of the parse.y grammar which worked fine for the correct configurations but which could allow very twisted configurations to parse valid.

While fixing an ambiguous case, I realized that the parsing had inherited some complex logic from a feature that was desired a long time ago but which we never used as the rules could become ambiguous. So I began shooting down various parts which were not worth implementing and ended up removing no longer required lists and structures, making the use of tables more coherent accross the daemon as sometimes they were refered by table pointer and sometimes by table id.

The end result is a much less bloated grammar, which is semantically more right, and which is much cleaner.

More virtual simplification

Building on the foundations from last week and the cleaned up parse.y, I have brought a new feature which was not possible before: using "for local" or "for any" as the destination of a virtual domain. Indeed, you could "accept for any" or "accept for local" but then OpenSMTPD would assume a primary domain and would perform a system user lookup for the user part. This was actually a side-effet of a parse.y ambiguity where ANY, LOCAL, DOMAIN and VIRTUAL were at the same level and VIRTUAL being considered as a special case.

Since last week, a virtual domain is simply a regular domain which has defined a virtual mapping. The same logic has been applied to ANY and LOCAL meaning that you can "accept for any" or "accept for any virtual [...]" and it will work as expected, instead of the syntax I demonstrated last week: accept for domain "*" virtual [...], which is still valid but will work in a different way internally.

A user had opened a ticket to ask if we could turn OpenSMTPD into a sink where it would accept mail for any destination and deliver to a single account. With the "accept for any virtual [...]" improvement it became simpler as it only required adding a global catch-all that isn't domain aware.

Virtual now support a global catch-all "@" which allows a static mapping to handle the catch-alls for multiple domains:

accept for any virtual { "@" => gilles } deliver to maildir

The above will have any user of any domain delivered to local account gilles.

General cleanup

We have spent a large amount of time working on a general cleanup of the code base. Amongst other things, we removed some fields from the global struct smtpd to statically isolate them to the specific files that were using them. This helped ensure that we didn't violate API layers.

Then we spent a great deal of time killing a monster structure, struct submit_status, which was used for all kind of inter-process exchanges. We came up with several lighter structures, carrying only the required information and being tied to a particular process to process exchange. This has required a bit of rework in various processes but the end result is less confusion, code that's easier to maintain and read for new comers.

MFA rework

The MFA process, in charge of filtering the different stages of a SMTP session has been considerably simplified. It no longer knows about SMTP states, which was an API layer violation, and has had a lot of code removed while providing the same service. It shrank by over 100 lines and has become a very small piece of code whose only purpose is to serve as the entry point to the filters evaluation.

This was a pre-requisite to the filter work which is already complex enough that we didn't want to add unneeded complexity upfront.

MTA rework

A lot of work has been done in the MTA engine which was almost completely rewritten. MTA now knows how to share MX, PTR and credentials for a route. This means that a single DNS request to the lookup daemon can be performed to deal with multiple messages heading to multiple domains.

It also deals much better with MX problems: When too many sessions fail to establish a link with a specific MX, the MX is marked as broken and not tried any further by new sessions.

It is also capable of dispatching connections on various MX of same priority: When sending many messages, the MTA will spawn sessions against the different MXs with the lowest priority within the default connection limit.

If all MXs at a given priority have errors, it moves to the next level to reach backup MXs. If no MX can be reached for a route, a temporary error is triggered.

I should mention that several kinds of error that used to trigger a permanent error on messages will now simply mark MX on which they occur as broken, giving the mails a chance to be routed through another MX.

The logs have been improved too, especially with SSL-related errors. All problems on MXs are now logged (as "smtp-out:") to help the administrator diagnose with relaying.

SMTP rework

The SMTP engine has been reworked as a pre-requisite for filters. It is now running on poolp.org and powers the OpenSMTPD mailing list.

The idea was to make the code generally simpler to follow and extend.

First, most of the smtp specific structures and defines have been isolated into smtp_session.c, which makes the smtpd.h file a bit less bloated. It was also the opportunity to finally get rid of the horrible submit_status structure.

The dispatching of user command is more straightforward, and the imsg dispatching code now makes use of very specific message structures.

The rewrite was painful, mostly because the former code was not easy to grasp, and because we slacked too much on the regression suite, which got improved in the process.

Furthermore, the interaction with the MFA got more complicated with the recent update. Basically we need to de-couple the forwarding of the message data to the MFA from the receveing of filtered data.

To sum up, the new code should be a much better ground to implement features like proper filtering and pipelining on the SMTP side.

imsgproc and filter work

OpenSMTPD has had filters for over a year now, but disabled as the API is not stable and there were more important stuff to deal with.

The design for filters has been discussed on this blog already but as a quick reminder, filters in OpenSMTPD are standalone programs that are linked against a lib we provide to demonize and provide an event-based callback mechanism.

The filters can be very very easy and an example of a working filter could be as simple as:


define SMTPD_FILTER

include "smtpd-api.h"

void mail_cb(uint64_t id, struct filter_mail p, void *arg) { / block idiots */ if (! strcmp(p->domain, "0pointer.net")) { filter_api_reject(id, 530, "You're not welcome, go away !"); return; }

filter_api_accept(id);
}

int main(int argc, char argv[]) { / init the lib and setup daemon and imsg framework */ filter_api_init();

/* register callbacks */
filter_api_register_mail_callback(mail_cb, NULL);

/* event loop */
filter_api_loop();

/* never reached */
return 0;
}

I have spent time in the glue code to change a bit how it worked and have it rely on a new api imsgproc which generalizes the setup imsg / fork / exec operation which we'll end up using for all filters but also possibly for other pluggable backends. It works fine and I've successfully compiled about a dozen different filters working at different hooks and combinations of hooks.

The API is broken at this point due to the late commit of the smtp rework which didn't leave me much time to fix before the week-end but surely it will get better next week.

Monkey Branch

Yesterday morning I was waiting for Eric to merge a blocking part for my current work so I decided to switch to something radically different.

So I tackled a new issue: how do we ensure that our error code path is correct for the errors that you cannot reproduce easily because they are so rare. A typical example of such an error is a getpwnam() failure because a descriptor was not available or because we received an EIO. In such cases, we want OpenSMTPD to correctly handle the error as transient and not reject it permanently which would lead to a lost mail.

I recalled this Chaos Monkey tool at Netflix that voluntarily produced errors at random to let them ensure their high-availability really works, and I came up with a set of MONKEY_* macros to randomly produce chaos at strategic places. The monkeys will provoke latency in imsg handling and will cause some places to report a temporary failure at random.

In the ten minutes that followed my initial testing, they helped spot two very subtle bugs which we would have probably not run into in years. We now know for sure that these code paths are correct and we need to spread more monkeys ;-)

This is done in a separate branch which I mirrored on github and which is synched with master. To test, simply checkout the monkey branch (lol no ?) and setup env CFLAGS=-DUSE_MONKEY before building. Then, when sending mail to yourself you should start seeing:

$ echo test | mail -s 'test' gilles $ echo test | mail -s 'test' gilles send-mail: command failed: 421 Temporary failure $ echo test | mail -s 'test' gilles send-mail: command failed: 421 Temporary failure $ echo test | mail -s 'test' gilles $

Ideally, OpenSMTPD should NEVER return a 5xx error while running in monkey mode that it doesn't return when not running in monkey mode.

Fun with Google Charts ;-)

That has nothing to do with OpenSMTPD itself but rather with me playing with a Google API to generate graphs based on the output of smtpctl show stats:


    <img src="https://www.poolp.org/~gilles/graph.png">
    <img src="https://www.poolp.org/~gilles/graph2.png">

  
I will try to spend some time making sure the smtpctl tool can be used to build other tools upon for administrators to take full advantage of the real-time statistics, profiling, scheduler-provided mail queue info, etc...

Stay tuned, more goodies next week !

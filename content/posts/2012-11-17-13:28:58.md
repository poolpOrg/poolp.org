---
title: "News from the front"
date: 2012-11-17 13:28:58
category: OpenSMTPD
author: Gilles Chehade
---

Almost a month since my last post... I know, I know.

So I left my position at Scality two weeks ago. I had been there for a year and a half working on a very interesting and tricky project. I will miss my colleagues and the discussions we had on all kinds of topic, I hope them the best future and that we will be able to share beers again ;-)

I should also emphasize that they spoiled me for my leave with a ton of gifts including most of the books from my amazon wishlist (Cryptography Engineering, Implementing SSL/TLS, Theory of music) as well as candies, chocolate, rubber ducks see rubber duck debugging, boxers (for when I'll become a homeworker) and, last but not least, a book about the inner details of former president Sarkozy occupation of the Elysee. I suspect the book to be a reminder that if even criminals with disgusting ass-faces manage to succeed in life, I shouldn't worry about new challenges. Thanks.

Obligatory OpenSMTPD section of my blog post ;-)

The last two weeks have been spent working very hard on improving OpenSMTPD and we're making progress at an alarming rate. I won't go into the details of each and every improvement we made but here's a quick summary:

The logging format has been improved, we have done a lot of rewording and changes to provide the most information using a concise format that can be easily understood by humans and that can easily be parsed or grepped.

Support for a monitor command has been added to the smtpctl utility. It allows an administrator to easily monitor a running instance of OpenSMTPD and what it is doing in real-time, displaying states every second.

First shot at a regress suite with a utility that allows the scripting of SMTP sessions scenarios. In a near future, we will be writing various scenarios which will allow us to verify that we don't introduce regressions with new features and bugfixes.

Improved scheduler API by removing the very annoying and tricky to implement Qwalk API and replacing it with a new queue operation Q_LEARN.

Mailq now supports an "online" mode which provides more information than the offline mode. The online mode will query the scheduler getting reliable real-time information. Offline mode is equivalent to what we had before.

Improved format for expansion strings in smtpd.conf and ~/.forward files. We used to support one char formats like %a, %u, and it was both confusing and hard to extend to support new formats while still making sense at first sight. We now support a clearer format: %{rcpt.domain}, %{user.directory}, etc... The are various supported formats documented in smtpd.conf(5) and each support partial expansion using optional begin and end positive and negative offsets: with user "gilles", %{user.username[1:5]} will expand "ille".

Added a RAM queue_backend, mostly useful for debugging at this point or if you don't care about losing mail when you shutdown the daemon ;-)

Improved the grammar of our configuration file by removing a lot of stuff that was no longer relevant and changing the syntax to remove all ambiguities. Sure, it breaks existing configuration files but given that none should exceed about 10 lines, it should be straightforward to fix: the poolp.org smtpd.conf which voluntarily exhibits a complex setup (aliases, primary domains, backup MX, virtual domains, relaying, ...) was switched in two minutes.

Replace the map API with a new table API that provides much simpler and saner semantics. Tables are simpler to declare than maps from a smtpd.conf point of view; they have types so that OpenSMTPD can detect tables used in an inappropriate context at smtpd.conf parse time; and they provide lookup services which allows backends to support only a few kinds of lookups and smtpd to spot that at smtpd.conf parse time too. The new table API simplifies a lot of things, the backends are simpler to write and the end-result is much more reliable.

With new table API I could get rid of the user_backend API and replace it with a table lookup service. It is now possible to write table backends to lookup system users instead of relying on getpwnam... but at the moment OpenSMTPD hardcodes the use of internal table so there's still work to do.

We no longer support "file" as a lookup backend. Whenever a smtpd.conf refers to a file for a lookup, it will internally convert it to a static table which will achieve the exact same result while removing duplicate code. The change is not visible to the users.

New dict* API, akin to the tree* API but with char* keys will allow us to simplify a lot of code with regard to how tables are handled and managed by OpenSMTPD.

TONS of KNF cleanup (several hundreds), removal of old defines, refactors to remove structures that are used as unnecessary indirections, simplifications of equivalent code, etc, etc ...

Various bug fixes including one causing some envelopes to possibly be skipped and triggering a start-time crash.

That's a SUMMARY, the complete updates can be checked on the commit logs of our github mirror if you're interested. Keep in mind that this is the changelog of the past two weeks only, we have the same kind of changes coming in the next few days and we're definitely not slowing down ;-)

Stay tuned !

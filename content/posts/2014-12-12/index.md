---
title: "The state of filters"
date: 2014-12-12 16:56:24
category: OpenSMTPD
authors:
 - "gilles"
categories:
 - technology
---

{{< tldr >}}
	yeeeees, filters are coming. don't believe us ? here's an example. be patient.
{{< /tldr >}}

On my death bed
---------------
On my death bed, when my life flashes before my eyes and I start recalling what people have told me during my (hopefully long) lifetime, these sentences will single out:

> "When will OpenSMTPD support filters ? I need it."

Not that it carries a philosophical meaning that will have taken me a lifetime of thinking, but because for the last three years I have been hearing this every time I met someone IRL and discussed OpenSMTPD, I have read it on our Github issues tracker, on GTalk, on IRC, on Twitter, on Facebook, on random forums, and just this week three times in my mailbox. I can honestly say that during these three years, OpenSMTPD filters have been mentioned to me hundreds of times without exaggerating. Yup. That's what I'll remember :-/

A short reminder
----------------
The SMTP server accepts connections from clients, it establishes sessions and processes SMTP transactions which then eventually result in a mail being accepted for delivery or rejected. This decision isn't a very smart process, the server looks at a ruleset that basically lists which domains are to be accepted or rejected, and it tries to locate a user under these domains. That's it, more or less.

So what are filters useful for ?
--------------------------------
 Filters are small pieces of code that can be used to modify the behavior of an SMTP session and decide to accept, reject or modify the client input based on decisions that are not related to that ruleset. For instance, a filter may decide that besides the ruleset, it also wants to check if a domain is not part of a public blacklist. It may want to decide that a particular client should not be able to send mail because according to some database, it has exceeded a quota. It may simply want to check at the content of the DATA phase and try to detect some keywords typical of spam or even alter it to remove (bad bad bad) or append some headers.

 Technically these behavior could be implemented in the daemon itself, but providing an API for filters allows this code to remain outside of the daemon code, keeping it simple, and allowing anyone to customize it for their needs without needing us to cooperate.

Filters goals
-------------
 Filters are a feature available in various MTA, this is not an OpenSMTPD thing and that's why so many people have requested it. They expect the software to be extensible through filters because there are many tools out there that they could plug somehow to make it fit their use case. These tools are usable with the other MTA, so they should be usable with ours.

 There are different ways of achieving a filter API, some relying on forked processes with a common convention to read on standard input and write on standard output, others providing a set of functions to do the work. Some as standalone filters, others as shared objects or libraries importing the features inside the daemon code base.

 The most commonly used is the milters API which was written for a competitor software and adapted to other competitors with varying degrees of success and completeness. It is mature, has a wide range of filters written for it, and allows doing pretty much anything. When we started thinking about how we'd integrate filters, this was the first API we looked at as we could have a reference implementation to compare ours to and we would benefit from the plethora of filters that were already available.

 Then, we changed our mind: it doesn't do what we want and it doesn't fit well in our asynchronous model, it's going to be a pain in the ass. We don't want to grab an existing API and modify our design to have it fit it, we want an API that just fits in our design.

 What we need is an API that's fully asynchronous, that's highly flexible while not bloated and that's dead easy. It needs to be so flexible that we can implement bindings to other languages than C, or even a milter interface to translate milter calls to ours, and it needs to be so easy that it doesn't require hours to get a basic filter working even if using the API for the first time.

Filters design
--------------
 So, I did a bit of research on how we would implement this API and after a while came up with these requirements:

 filters should not be libraries or shared objects, they should be standalone executables running in their own memory space;
 they shouldn't be fork()-ed at runtime because it would crush performances;
 they should be able to be individually chroot()-ed;
 they should be able to run with their very own privileges, isolated one from another if needed;
 whatever their dependencies, the daemon should not have to care, it should not know anything besides its filter protocol;
 Long story short, while it would be considerably easier for us, we don't want filters to be plugged as shared objects. The problem with that approach is that the filter would then share the daemon memory space and that's something we don't want to happen. The filters should be isolated in their own memory space and they should know nothing about the server except what the server hands them. That way, if a filter suffers from a bug allowing leak of random memory areas, it will not be able to leak information that is part of the server. This concern predates by far the Heartbleed catastrophe but since some people back then thought it was not a real problem, now they have a perfect example of why it was ;-)

 Anyways, to respect these requirements and given that the daemon itself drops privileges at startup, filters had to become daemons of their own started with the server and communicating with it during its lifetime. Fortunately this is something we're quite familiar with, it is basically OpenSMTPD's model: parent starts, sets up communication channels with imsg framework and starts other processes.

 We decided to build the filter API on that model, then we realized that while we were familiar with it, it would be far too complex for users to deal with this whole fork() / imsg thing. So we wrote a "filter glue" which takes care of setting everything up and running so that a user can write a filter by focusing only on its own functions.

```c
static int
on_connect(uint64_t id, struct filter_connect *conn)
{
	log_warnx("filter-stub: OHAI, SOMEONE CONNECTED !");
	return (1);
}

int
main(int argc, char **argv)
{
	int	ch;
	
	log_init(-1);

	while ((ch = getopt(argc, argv, "")) != -1) {
		switch (ch) {
		default:
			log_warnx("warn: filter-stub: bad option");
			return (1);
		}
		/* NOTREACHED */
	}

	argc -= optind;
	argv += optind;

	/* register a callback for "on_connect" event */
	filter_api_on_connect(on_connect);

	/* start doing stuff */
	filter_api_loop();

	log_warnx("warn: filter-stub: exiting");
	return (1);
}
```

As shown in this example, the main function only calls filter_api_on_connect() to let it know that it wants on_connect() to be called when there's a connection. Then it calls filter_api_loop() to actually start processing events.

The whole "setup a channel with the daemon, fork and maintain state" is completely hidden within the API layer and transparent to the user. The filter API exposes a unique session identifier and per-hook structures filled with the data available in that phase, the filter is free to maintain its own data structures using the identifier to pass data from a hook to another.

It doesn't get much simpler.

But that's C ?!
---------------
Of course, it's C, what else ?

The API is written and exposed in C allowing C programmers to write native filters. We want to make it possible to write them in other languages but our language of choice and the one we will use for any filter that may be shipped with the daemon remains C.

So from there, if we want to allow filters to be written in different languages, we have two ways:

either we provide a binding, which is basically a reimplementation of our API in that language;
or we provide bridge filters, which are basically proxies that will convert calls in the end-user language to C calls;

Both are slightly different, they achieve the same, but it seemed to me a better approach to write bridge filters that keeps a binding of only the few exposed API calls, than having to reimplement the entire API glue. Also, if we provide a "kind of official" bridge, it will benefit everyone ultimately.


So how hard is it to write a bridge ?
-------------------------------------
It really depends on what the target language is. For instance, I had a working filter-python bridge in less than an hour without having ever used the python API before. The filter-perl bridge took a few hours. The code to make the bridge itself is rather short, it is simply a matter of mapping a callback to a translation function. For example, if we look at our filter-python and how it implements on_connect(), we would see something like this (just focusing on the interesting parts):

```c
static int
on_connect(uint64_t id, struct filter_connect conn)
{
	PyObject	py_args;
	PyObject	py_ret;
	PyObject	py_id;
	PyObject	py_local;
	PyObject	py_remote;
	PyObject       *py_hostname;

	/* we translate our parameters to types that python can understand */
	py_args     = PyTuple_New(4);
	py_id       = PyLong_FromUnsignedLongLong(id);
	py_local    = PyString_FromString(filter_api_sockaddr_to_text((struct sockaddr *)&conn->local));
	py_remote   = PyString_FromString(filter_api_sockaddr_to_text((struct sockaddr *)&conn->remote));
	py_hostname = PyString_FromString(conn->hostname);

	/* we prepare for calling the on_connect function inside the python script */
	PyTuple_SetItem(py_args, 0, py_id);
	PyTuple_SetItem(py_args, 1, py_local);
	PyTuple_SetItem(py_args, 2, py_remote);
	PyTuple_SetItem(py_args, 3, py_hostname);

	/* we let python do the work and give us back a return value */
	py_ret = PyObject_CallObject(py_on_connect, py_args);
	Py_DECREF(py_args);

	if (py_ret == NULL) {
		PyErr_Print();
		log_warnx("warn: filter-python: call to on_connect handler failed");
		exit(1);
	}

	return (1);
}

int
main(int argc, char *argv[])
{
	/* we don't care what's above */

	/* we look if there's a symbol "on_connect" defined in the end-user filter */
	py_on_connect = PyObject_GetAttrString(module, "on_connect");

	/* if there's one and it's a function, we register our translating callback */
	if (py_on_connect && PyCallable_Check(py_on_connect))
		filter_api_on_connect(on_connect);

	/* we don't care what's below */
}

```


Just for the purpose of comparing with another bridge, here's the filter-perl bridge, doing the exact same thing:

```c
static int
on_connect(uint64_t id, struct filter_connect *conn)
{
	const char	*local;
	const char	*remote;

	local = filter_api_sockaddr_to_text((struct sockaddr *)&conn->local);
	remote = filter_api_sockaddr_to_text((struct sockaddr *)&conn->remote);

	/* looks simpler than python ? I hid the gross parts in call_sub_sv() ;-) */
	call_sub_sv((SV *)pl_on_connect, "%i%s%s%s", id, local, remote, conn->hostname);
	return filter_api_accept(id);
}

int
main(int argc, char *argv[])
{
	/* we don't care what's above */

	/* we look for a function "on_connect" defined in the end-user filter and register a callback */
	if ((pl_on_connect = perl_get_cv("on_connect", FALSE)))
		filter_api_on_connect(on_connect);

	/* we don't care what's below */
}

```


As you can see, adding support for writing filters in another language is not really a big challenge, I could write two bridges in an afternoon without having played with either one of the C/API for these languages. I'm sure someone else could have completed them faster than I did.

Now, anyone willing to write a filter in perl or python can simply install the bridges then focus on writing the filter script itself in their language of choice. Here's an example for Python:

```python
import filter

def on_connect(session, local_addr, remote_addr, hostname):
    print "on_connect:", hex(session), local_addr, remote_addr, hostname
    return filter.accept(id)
```

And here's one for Perl:

```perl
use strict;
use warnings;

sub on_connect {
    my ($id, $l, $r, $h) = @_;
    return smtpd::filter_api_accept $id;
}
```

The nice part is that this comes with a very very very low overhead, the interpreter is started only once and the performances should be near native no matter which language is used.


What about milters ?
--------------------
Well, it should be possible to implement a milter bridge that can translate milter API calls to our API using the same technique as for translating python and perl calls to our API. There may be some changes required in our API, but we're not opposed to that and if someone stepped up to write a milter bridge, then I would definitely help by adapting the API where needed... as long as it remains simple and clean.

Now, as far as I am concerned...

I have absolutely no interest in a milter interface myself as I will be using the native one, so clearly this is not even going to hit my todo list. In my opinion, having native filters is a much better goal than trying to make filters written for another MTA work with ours, I suspect there will always be corner cases that will turn this into a nightmare when the same effort could be put in writing quality native filters.

If someone wants to spend time trying to make milters run on OpenSMTPD, well, good luck and may you find entertainment in that journey !


So, what's the state of filters ?
---------------------------------
The filter API works. It is not done, it still has bugs, it still lacks features, but it's in a state where you can actually write a filter and have it process all kinds of events and tell the SMTP server to reject or accept based on filter decision. I wrote tiny test scripts to verify that I could write filters to limit the number of connections from a same user, reject upon some keywords, alter content, skip headers, etc... all of this works.

Just as an example, here's a filter I wrote this morning, all it does is declare callbacks for every hook and print the parameters it receives. I wrote it in Python because it's easier for everyone to figure out what it does, but you could do the same in C or Perl:

```python
import filter

def on_connect(id, local_ip, remote_ip, hostname):
    print "on_connect:", hex(id), local_ip, remote_ip, hostname
    return filter.accept(id)

def on_helo(id, helo):
    print "on_helo:", hex(id), helo
    return filter.accept(id)

def on_mail(id, mail):
    print "on_mail:", hex(id), mail
    return filter.accept(id)

def on_rcpt(id, rcpt):
    print "on_rcpt:", hex(id), rcpt
    return filter.accept(id)

def on_data(id):
    print "on_data:", hex(id)
    return filter.accept(id)

def on_dataline(id, line):
    print "on_dataline:", hex(id), line
    return filter.writeln(id, line)

def on_eom(id, size):
    print "on_eom:", hex(id), size
    return filter.accept(id)

def on_commit(id):
    print "on_commit:", hex(id)
    return filter.accept(id)

def on_rollback(id):
    print "on_rollback:", hex(id)
    return filter.accept(id)

def on_disconnect(id):
    print "on_disconnect:", hex(id)

```


With this filter plugged in, the following SMTP session:


     220 debug.poolp.org ESMTP OpenSMTPD
     EHLO myself
     250-debug.poolp.org Hello myself [127.0.0.1], pleased to meet you
     250-8BITMIME
     250-ENHANCEDSTATUSCODES
     250-SIZE 36700160
     250-DSN
     250 HELP
     MAIL FROM:<gilles>
     250 2.0.0: Ok
     RCPT TO:<gilles>
     250 2.1.5 Destination address valid: Recipient ok
     DATA
     354 Enter mail, end with "." on a line by itself
     Subject: ohai !
     
     test
     .
     250 2.0.0: 2fb4c3b7 Message accepted for delivery


Results in the following log on the server running in debug mode, with all filter callbacks triggered:

	debug: smtp: new client on listener: 0x1c9bbad31000
	smtp-in: New session a7bc78b6177f09b5 from host localhost [127.0.0.1]
	on_connect: 0xa7bc78b6177f09b5L 127.0.0.1 127.0.0.1 localhost
	on_helo: 0xa7bc78b6177f09b5L myself
	on_mail: 0xa7bc78b6177f09b5L gilles@debug.poolp.org
	on_rcpt: 0xa7bc78b6177f09b5L gilles@debug.poolp.org
	on_data: 0xa7bc78b6177f09b5L
	smtp: 0x1c9c37af3000: fd 5 from queue
	smtp: 0x1c9c37af3000: fd 7 from filter
	on_dataline: 0xa7bc78b6177f09b5L Received: from myself (localhost [127.0.0.1]);
	on_dataline: 0xa7bc78b6177f09b5L        by debug.poolp.org (OpenSMTPD) with ESMTP id 2fb4c3b7;
	on_dataline: 0xa7bc78b6177f09b5L        for <gilles@debug.poolp.org>;
	on_dataline: 0xa7bc78b6177f09b5L        Fri, 12 Dec 2014 15:48:42 +0100 (CET)
	on_dataline: 0xa7bc78b6177f09b5L Subject: ohai !
	on_dataline: 0xa7bc78b6177f09b5L
	on_dataline: 0xa7bc78b6177f09b5L test
	debug: smtp: 0x1c9c37af3000: data io done (195 bytes)
	on_eom: 0xa7bc78b6177f09b5L 195
	filter: eom not received yet
	debug: 0x1c9c37af3000: end of message, msgflags=0x0000
	debug: scheduler: evp:2fb4c3b7f8d5ee1d scheduled (mda)
	smtp-in: Accepted message 2fb4c3b7 on session a7bc78b6177f09b5: from=<gilles@debug.poolp.org>, to=<gilles@debug.poolp.org>, size=195, ndest=1, proto=ESMTP
	on_commit: 0xa7bc78b6177f09b5L



CAN I HAZ IT FOR REAL !!?
-------------------------
Filters are not stable yet, just this morning I have fixed a bug that lead to a crash and another user has found a corner case which causes the filter glue to crash when a filter forks. They're not just disabled, but also removed from our stable release. Unlike a while ago, there's no way to activate them in a stable release. Gone. Bye-bye. Ma3 salam.

Filters support is only available in our development master and portable branches which are mirrored on Github. It is also available in our snapshots, which we publish every now and then from these branches. However... they are not meant for general use.

People keep complaining that we didn't document the API, but what they fail to grasp is that:

we don't write documentation before actual code, we write code then document;
we don't document code that is subject to heavy changes, it's not meant to be used, it should be ignored, you should pretend it doesn't exist;
we have a working API but it's not done, it has issues, we don't want people to use it;

At the time being, we leave it undocumented because it makes it harder for people who can't read code to actually play with it, hit issues that aren't bugs and confuse us about what is a real issue and what is a user bug.

We want people to help us squash the bugs in this API, however we can't hold hands of everyone willing to give this API a try and write a tiny filter. So we decided to keep the API undocumented for now and only help those that can read code and figure things out on their own, we want to keep the noise ratio to the lowest. If you don't know C, I would discourage you to play with the API yet, even if you intend to write filters using a bridge. You will be on your own and unlikely to go much far.

I wanted to show code today because so many people keep requesting a filter API and we've been saying that "it's a work in progress" for so long that it looked like vaporware. I hope this post clarifies that we actually HAVE an API, a usable one on top of that, but it's just not ready and we're too close to having what we want that we're not going to ruin it all by rushing to deliver something half-baked.

Note that I didn't show how you could actually plug it, this was done on purpose. If you manage to find by yourself, feel free to help us move the API forward by starting to write your own filters. If you need help to activate, you've failed the easiest test, by all means stay away from this API until it has stabilized.


AWWWWW :-(
----------
Don't be sad :-(

We're working precisely on getting this API stable enough that we can ship it with our next major release. It will be considered experimental, but it will be shipped and you will be able to play with it.

Before you ask, no we don't know when that release would be. Unlike other releases which we can plan on a regular schedule, this one is so invasive and the feature is so big that we will delay for as long as it takes before planning the release. The only thing delaying the major release is the filter code.

And don't get confused, there will be an OpenSMTPD-5.4.4 release soon... but this is a minor release, not the one with filters.

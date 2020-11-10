---
title: "OpenSMTPD: table_proc, queue_proc, crypto queue and other stuff"
date: 2013-04-26 18:42:36
category: OpenSMTPD
author: Gilles Chehade
---

Yop,

This week has been very productive with several tickets closed and more about to be closed.

Eric has done an amazing work as you will soon realize ;-)

I will push to our Github mirror and publish a snapshot this week-end very likely, until then don't look for the features, only we have them !

Compressed and Encrypted queue

While working on bringing back encrypted queue support, I ran into a couple issues.

First, I could use a compressed queue or an encrypted queue, but activating both would fail. This was strange because each one worked fine and their output was correct but somehow the combination led to errors. I tracked it down and figured that the reuse of the same buffer led to an overlap which didn't happen before the switch to aes-256-gcm but would completely corrupt decryption now. Using two separate buffers for the compression and encryption was enough to solve this.

Then, I realized that while everything was working flawlessly at runtime, OpenSMTPD would fail to load envelope at startup. After some tracking I isolated the bug to the queue_fsqueue.c backend which was violating API layers by trying to validate the envelope content while it should be done by the upper layer after decryption/decompression. The fix seemed tricky at first but was really about 10 lines to end up with.

As of my sandbox, I currently have an OpenSMTPD that runs with the following configuration file: listen on lo0

queue compression

generate key with: openssl rand -hex 16
queue encryption key 95ca5f8053fc2baca7c390bea62bd9e2

accept from any for domain poolp.org deliver to maildir accept for any relay All it lacks is better smtpctl integration so that we can "smtpctl show envelope" on an encrypted envelope :-)

table_proc API

A feature I have wanted for many months is the ability to externalize table backends so that we can write very funky backends with dependencies that are never going to make it to the OpenBSD tree.

For example, people are going to want to store their aliases in mysql or pgsql and while the feature is easy to achieve thanks to the table API, we can't provide it out of the box because of the dependencies AND people can't contribute it easily because tables are part of the daemon and will require rebuilding after some patching.

So what I wanted was to have table backends work like our filters: they are standalone programs that are executed at startup by OpenSMTPD and communication takes place through the imsg(3) framework.

Eric took that idea, improved it and made a working implementation. So now, one can write a custom backend without having us do anything on our side, with whatever dependencies wanted, and OpenSMTPD can make use of it by providing the path in the configuration file:

table aliases "proc:/usr/libexec/smtpd/backend-table-sqlite -f /etc/mail/sqlite.conf /etc/mail/sqlite-aliases.db"

listen on lo0

accept from any for domain poolp.org alias deliver to maildir accept for any relay

Writing a custom backend implementation is very simple, fill in the blanks: /* * Copyright (c) 2013 Eric Faurot eric@openbsd.org * * Permission to use, copy, modify, and distribute this software for any * purpose with or without fee is hereby granted, provided that the above * copyright notice and this permission notice appear in all copies. * * THE SOFTWARE IS PROVIDED "AS IS" AND THE AUTHOR DISCLAIMS ALL WARRANTIES * WITH REGARD TO THIS SOFTWARE INCLUDING ALL IMPLIED WARRANTIES OF * MERCHANTABILITY AND FITNESS. IN NO EVENT SHALL THE AUTHOR BE LIABLE FOR * ANY SPECIAL, DIRECT, INDIRECT, OR CONSEQUENTIAL DAMAGES OR ANY DAMAGES * WHATSOEVER RESULTING FROM LOSS OF USE, DATA OR PROFITS, WHETHER IN AN * ACTION OF CONTRACT, NEGLIGENCE OR OTHER TORTIOUS ACTION, ARISING OUT OF * OR IN CONNECTION WITH THE USE OR PERFORMANCE OF THIS SOFTWARE. */

include
include
include
include "smtpd-defines.h"
include "smtpd-api.h"
static int table_stub_update(void); static int table_stub_check(int, const char *); static int table_stub_lookup(int, const char *, char *, size_t); static int table_stub_fetch(int, char *, size_t);

int main(int argc, char **argv) { int ch;

while ((ch = getopt(argc, argv, "f:")) != -1) {
            switch (ch) {
	                default:
			                    errx(1, "bad option");
					                        /* NOTREACHED */
								            }
									        }
										    argc -= optind;
										        argv += optind;

    table_api_on_update(table_stub_update);
        table_api_on_check(table_stub_check);
	    table_api_on_lookup(table_stub_lookup);
	        table_api_on_fetch(table_stub_fetch);
		    table_api_dispatch();

    return (0);
    }

static int table_stub_update(void) { return (-1); }

static int table_stub_check(int service, const char *key) { return (-1); }

static int table_stub_lookup(int service, const char *key, char *dst, size_t sz) { return (-1); }

static int table_stub_fetch(int service, char *dst, size_t sz) { return (-1); }

Sexy hu ? :-)

queue_proc API

Since he was already deep into the proc-ification of stuff, he also proc-ified the queue which was another feature I have wanted real bad for a very long time.

So it is now possible to write a queue backend with whatever dependency you want and have OpenSMTPD use it through a single configuration line.

And just like for tables, writing a custom queue is pretty easy, just fill in the blanks: /* * Copyright (c) 2013 Eric Faurot eric@openbsd.org * * Permission to use, copy, modify, and distribute this software for any * purpose with or without fee is hereby granted, provided that the above * copyright notice and this permission notice appear in all copies. * * THE SOFTWARE IS PROVIDED "AS IS" AND THE AUTHOR DISCLAIMS ALL WARRANTIES * WITH REGARD TO THIS SOFTWARE INCLUDING ALL IMPLIED WARRANTIES OF * MERCHANTABILITY AND FITNESS. IN NO EVENT SHALL THE AUTHOR BE LIABLE FOR * ANY SPECIAL, DIRECT, INDIRECT, OR CONSEQUENTIAL DAMAGES OR ANY DAMAGES * WHATSOEVER RESULTING FROM LOSS OF USE, DATA OR PROFITS, WHETHER IN AN * ACTION OF CONTRACT, NEGLIGENCE OR OTHER TORTIOUS ACTION, ARISING OUT OF * OR IN CONNECTION WITH THE USE OR PERFORMANCE OF THIS SOFTWARE. */

include
include
include
include "smtpd-defines.h"
include "smtpd-api.h"
include "log.h"
static int queue_stub_message_create(uint32_t *msgid) { return (0); }

static int queue_stub_message_commit(uint32_t msgid) { return (0); }

static int queue_stub_message_delete(uint32_t msgid) { return (0); }

static int queue_stub_message_fd_r(uint32_t msgid) { return (-1); }

static int queue_stub_message_fd_w(uint32_t msgid) { return (-1); }

static int queue_stub_message_corrupt(uint32_t msgid) { return (0); }

static int queue_stub_envelope_create(uint32_t msgid, const char *buf, size_t len, uint64_t *evpid) { return (0); }

static int queue_stub_envelope_delete(uint64_t evpid) { return (0); }

static int queue_stub_envelope_update(uint64_t evpid, const char *buf, size_t len) { return (0); }

static int queue_stub_envelope_load(uint64_t evpid, char *buf, size_t len) { return (0); }

static int queue_stub_envelope_walk(uint64_t *evpid) { return (0); }

int main(int argc, char **argv) { int ch;

while ((ch = getopt(argc, argv, "f:")) != -1) {
            switch (ch) {
	                default:
			                    log_warnx("warn: backend-queue-stub: bad option");
					                        exit(1);
								                    /* NOTREACHED */
										                }
												    }
												        argc -= optind;
													    argv += optind;

    queue_api_on_message_create(queue_stub_message_create);
        queue_api_on_message_commit(queue_stub_message_commit);
	    queue_api_on_message_delete(queue_stub_message_delete);
	        queue_api_on_message_fd_r(queue_stub_message_fd_r);
		    queue_api_on_message_fd_w(queue_stub_message_fd_w);
		        queue_api_on_message_corrupt(queue_stub_message_corrupt);
			    queue_api_on_envelope_create(queue_stub_envelope_create);
			        queue_api_on_envelope_delete(queue_stub_envelope_delete);
				    queue_api_on_envelope_update(queue_stub_envelope_update);
				        queue_api_on_envelope_load(queue_stub_envelope_load);
					    queue_api_on_envelope_walk(queue_stub_envelope_walk);

    queue_api_dispatch();

    return (0);
    }

The awesome part is that this layer considers the queue as a key-value store and unless it WANTS to inspect content, it can consider it as a blob... and since this layer is below the encryption and compression layer, a backend that doesn't need to inspect content will magically support the compression and encryption feature out of the box ;-)

If someone is interested in writing a queue_torrent/queue_git or queue_whateverdecentralizedtechnology, I will help as I have ideas around decentralized encrypted queues...

scheduler_proc

No, just kidding, he didn't do that much. scheduler_proc should be next ;-)

assorted improvements

There's been various improvements here and there on my part.

I have improved logging so that relaying displays the source address as well as the relay session.

I have a pending diff that is waiting for an OK and which ensures that OpenSMTPD copes with corrupted envelopes at runtime by moving them to the /corrupt queue. Currently, if an admin edits manually an envelope and fucks up, the daemon will abort.

I also found a missing check within the imsg API that can cause OpenSMTPD and possibly other daemons to fatal() under some situations. The diff is ready and waiting for okays from other OpenBSD hackers.

That's about all I think ;-)

Want to make us happy ?

I will take the opportunity to let you know, in case you missed it, that after a lot of hesitation and feeling bad about it, Eric has asked for a laptop or small donations to help him get one. So far he hasn't received much which is a shame considering how shitty his laptop is and how much time he spends working on it to write tricky DNS and SMTP code ;-)

Consider making him a small paypal donation (eric@openbsd.org) if you like his work, he really deserves not to work on that trashpile.

As for me, feel free to Flattr or head your spare bitcoins my way (13tTchonwhjvLRo2NcUwyqfKrTvKMLBWRi) as it automagically converts into much needed beers ;-)

The OpenSMTPD book

The OpenSMTPD book initially planned for late May will be delayed slightly as my health issues (nothing worrisome, just annoying stuff) caused me to stop writing for weeks. I will resume by next week but I don't want to rush just to make it to the deadline I imposed myself ... so it'll probably be released sometime this Summer.

#
I hope, you'll enjoy it because it's proved to be much more efforts than I assumed ;)

More next week !
